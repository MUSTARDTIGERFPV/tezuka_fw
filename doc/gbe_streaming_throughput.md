# GbE IQ streaming throughput: analysis, changes, plan

Status: written 2026-09-05 from source reading only. Nothing in this
document has been built or run on hardware yet. Every change below is
behind a runtime knob so it can be switched off from the u-boot
environment without a rebuild. Verify with the procedure at the end
before trusting any of it.

## Symptom

Complex8 (cs8) streaming over Gigabit Ethernet tops out around 30 Msps.
The README claims 45. The AD9363 can deliver 61.44 Msps. The goal is a
stable ~50 Msps over GbE.

| Rate (cs8) | Payload | Wire (TCP, MTU 1500) |
|---|---|---|
| 30 Msps | 60 MB/s = 480 Mbit/s | 51% of what GbE TCP can carry |
| 50 Msps | 100 MB/s = 800 Mbit/s | 85% |
| 55 Msps | 110 MB/s | ~94%, practical ceiling |
| 61.44 Msps | 123 MB/s = 983 Mbit/s | does not fit; impossible over GbE |

So the wire is not the limit until ~55 Msps. The device CPU is.

## Where the bytes go today

```
AD9361 -> axi_ad9361 -> cs12_cs8 mux (Q disabled => I holds 8b I + 8b Q)
       -> cpack -> axi_ad9361_adc_dma -> S_AXI_HP2 -> DDR
                                                        |
   iiod (core 1, SCHED_FIFO 99)  <--- mmap of DMA block (UNCACHED) ---+
       send() ---> tcp_sendmsg copy_from_user (1448 B at a time)
       ---> GSO/csum offload, fq+BBR qdisc, macb ring ---> GEM ---> PHY
```

Three CPU costs, all on the one core iiod is pinned to:

1. **Reading uncached memory.** `rx_dma` in every board DTS has no
   `dma-coherent` property and the ADC DMA lands on `S_AXI_HP2`, which
   does not snoop the CPU caches. The ADI kernel's
   `drivers/iio/buffer/industrialio-buffer-dma.c` therefore allocates
   every block with `dma_alloc_coherent()` and mmaps it with
   `dma_mmap_coherent()`: a Normal non-cacheable mapping on ARMv7.
   Each CPU load from it is a serialized bus transaction with no line
   fill and no prefetch. Uncached reads on a 667 MHz Cortex-A9 realistically
   run at 100-200 MB/s while burning the whole core.
2. **Per-packet TCP cost with no TSO and no jumbo frames.** libiio 0.26
   (what Buildroot 2026.02 ships) sends each block with a plain `send()`
   from `iio_buffer_start()` in `iiod/ops.c`. The macb driver's
   `zynq_config` has hardware checksum and scatter-gather but not
   `MACB_CAPS_JUMBO`, and the Zynq-7000 GEM has no TSO. At 100 MB/s that
   is ~69k TX segments/s plus ~35k ACKs/s, each built and completed by
   software.
3. **Extra per-packet overhead.** `S96networkcong` puts `fq` + BBR on
   eth0 (one hrtimer pacing event per packet). Kernel is `HZ_1000` +
   `PREEMPT`. All shared interrupts default to CPU0 while iiod is on
   CPU1, together with maia-httpd, mosquitto, classifier and avahi.

Rough single-core budget:

| Rate | Uncached copy | TCP TX + ACK + qdisc | Total on core 1 |
|---|---|---|---|
| 30 Msps | ~40% | ~45% | ~85-100% (where we are) |
| 50 Msps | ~65% | ~70% | ~135%: impossible |
| 50 Msps, cached blocks | ~10% | ~70% | ~80% |
| 50 Msps, zero-copy | ~5% | ~65% | ~70% |

Not the bottleneck: HP2 port and the 64-bit `axi_dmac` (well over
1 GB/s), DDR bandwidth, the cs12_cs8 mux, socket buffer sizes, the wire.

Corroborating evidence already in this tree:

- iiod is started with `chrt -r 99` + `taskset -c 1` and
  `kernel.sched_rt_runtime_us=-1` in `sysctl.conf`. That is what you do
  to a process that is CPU-bound.
- `005-maiasdr.patch` exports `v7_dma_inv_range` and an outer-cache
  invalidate so Maia-SDR's recorder can use **cached** memory with explicit
  invalidation. The Maia path already got this fix; the IIO path did not.
- `-D` (server demux) costs nothing with one client: `send_data()` only
  demuxes when the client's sample size differs from the device's.

## Changes made in this tree

### 1. Kernel: cacheable backing for iiod's mmap blocks

File: `board/tezuka/common/patches/linux/01e1dcc848ad2b63ec32384c3846f686fae6652c/0008-iio-buffer-dma-cacheable-legacy-mmap-blocks.patch`

Adds module parameter `industrialio_buffer_dma.cached_mmap` (kernel
default 0). When set, blocks allocated through the legacy mmap ABI
(`BLOCK_ALLOC` ioctl, the path libiio 0.x uses) are backed by ordinary
cacheable memory:

- Allocation: `dma_alloc_attrs(dev, size, &phys, GFP_KERNEL,
  DMA_ATTR_NO_KERNEL_MAPPING)`. On this kernel dma-direct handles that
  attribute first (`kernel/dma/direct.c`, `dma_direct_alloc_no_mapping`),
  keeps using CMA (highmem allowed), returns the first `struct page` as
  an opaque cookie and creates no kernel mapping. This keeps 64 MB
  blocks and 1 GB boards (where the 64 MB CMA area is likely in highmem)
  working. `dma_alloc_pages()` was rejected because it refuses highmem
  CMA and falls back to the buddy allocator, capping blocks at 4 MB.
- mmap: `remap_pfn_range()` with the untouched (cacheable)
  `vma->vm_page_prot` instead of `dma_mmap_coherent()`.
- Before submit to the DMA: `dma_sync_single_for_device()`, invalidate
  for input blocks (whole block, since the dmaengine backend resets
  bytes_used to the block size), clean for output blocks (bytes_used).
- On `DEQUEUE` ioctl: `dma_sync_single_for_cpu()` over bytes_used, done
  in process context on purpose. Invalidating a 4 MB block line by line
  (L1 by MVA, PL310 by PA) takes on the order of a millisecond or two
  and must not run in the DMA completion tasklet.
- New blocks are zeroed via `clear_highpage()` and the zeros written
  back, because the no-mapping allocation does not zero and the old
  path did.
- fileio (`read()`/`write()`) blocks and the non-legacy DMABUF path are
  untouched. Only blocks allocated after the parameter changes are
  affected, so flipping it at boot is safe.

Why this is correct on Zynq-7000: the Cortex-A9 L1 D-cache is PIPT, so
maintenance by any virtual alias hits the right lines, and MVA-based
maintenance is broadcast between the two cores (ACTLR.SMP), so it does
not matter which core does the sync. Set/way operations are not
broadcast, which is why the patch does not use a "flush everything"
shortcut even though it would be cheaper for big blocks.

Cost of the maintenance: roughly 3M cache lines/s at 100 MB/s, split
between the pre-DMA invalidate and the post-DMA invalidate, a few percent
of a core. Far cheaper than the uncached reads it replaces.

Known limitation: the user mapping is VM_PFNMAP, so `get_user_pages()`
fails on it, which means `MSG_ZEROCOPY` and `vmsplice()` cannot be used
yet. See "Next steps".

The TX path (DAC, used by pluto_stream for DVB-S2 and by any libiio TX
client) goes through the same block code and is therefore also switched
to cached memory with a clean before each transfer. It has to be tested
too, not just RX.

### 2. `S21misc`: turn it on at boot

Writes `iio_dma_cached` (u-boot env, default 1) to
`/sys/module/industrialio_buffer_dma/parameters/cached_mmap`, guarded
by the file existing so an unpatched kernel is unaffected.

Rollback without rebuild: `fw_setenv iio_dma_cached 0` and reboot, or
`echo 0 > /sys/module/industrialio_buffer_dma/parameters/cached_mmap`
before the client creates its buffer.

### 3. `S25irqaffinity` (new): pin streaming interrupts

GEM (`eth0`) IRQ -> CPU0, ADC/DAC `axi_dmac` IRQs (`7c400000`,
`7c420000`) -> CPU1. Keeps the network stack's interrupt work off the
core iiod is pinned to and makes the block-done wakeup of iiod
core-local instead of an IPI. Only acts on 2-core systems. Disable with
`fw_setenv irq_affinity off`.

Expected gain is small (a few percent). It is the easiest one to drop if
it shows any regression, and it is a plausible suspect if the DMA
completion path ever looks starved on CPU1 (iiod is SCHED_FIFO 99 on
that core with RT throttling disabled; ksoftirqd on CPU1 only runs while
iiod sleeps in poll).

### 4. `S96networkcong`: selectable TCP profile

New u-boot env `tcp_profile`:

- `wan` (default, unchanged behaviour): BBR + `fq` pacing. Right for
  DATV/pluto_stream over the internet.
- `lan`: cubic + `pfifo_fast`. No per-packet pacing timer, no rbtree
  work per segment. Use this when streaming IQ to a host on the same
  switch.

The default was deliberately left at `wan` so existing DATV users see no
change. Per-socket congestion control (`TCP_CONGESTION` setsockopt in
iiod) would let both coexist and is on the list below.

### 5. `S40network` / `update.sh`: config.txt plumbing

The three knobs (`iio_dma_cached`, `tcp_profile`, `irq_affinity`) are
written to the `[TEZUKA]` section of `config.txt` on the USB mass
storage and picked up by `update.sh` like every other key, so they can be
changed from a laptop without SSH.

### Considered and not changed

- **Conntrack/iptables NOTRACK**: no `iptables` package is built and no
  script in the tree loads a rule, so no netfilter hooks are registered
  and conntrack is not on the packet path. Nothing to gain unless
  someone enables NAT rules; then a `raw` table NOTRACK rule for port
  30431 would be the fix.
- **HZ_1000 / PREEMPT**: each is worth a few percent of throughput but
  changes latency behaviour for everything else (DATV timing, sweep).
  Left as an experiment, see below.
- **Socket buffer sizes**: `S96networkcong` lowers wmem_max from 16 MB
  (sysctl.conf) to 4 MB. On a LAN the bandwidth-delay product is a few
  hundred KB, so this is irrelevant and was left alone.

## Next steps, in order of payoff

### A. Route the ADC DMA through ACP (FPGA change, other repo)

This makes the hardware coherent and removes the need for cache
maintenance altogether. It is the cleanest fix and lands the data in L2
before iiod reads it. In the maia-hdl block design
(`maia-hdl/projects/<board>/system_bd.tcl` in the F5OEO maia-sdr fork;
the ADC DMA is currently on the HP2 interconnect together with the
recorder and the DAC DMA):

```tcl
# Enable the ACP slave and move the ADC DMA master onto it.
ad_ip_parameter sys_ps7 CONFIG.PCW_USE_S_AXI_ACP 1
ad_ip_parameter sys_ps7 CONFIG.PCW_USE_DEFAULT_ACP_USER_VAL 1
ad_ip_parameter sys_ps7 CONFIG.PCW_S_AXI_ACP_ARUSER_VAL 31
ad_ip_parameter sys_ps7 CONFIG.PCW_S_AXI_ACP_AWUSER_VAL 31
ad_connect sys_cpu_clk sys_ps7/S_AXI_ACP_ACLK
# remove axi_ad9361_adc_dma/m_dest_axi from ad_mem_hp2_interconnect and:
ad_connect axi_ad9361_adc_dma/m_dest_axi sys_ps7/S_AXI_ACP
ad_mem_hp2_interconnect ... # keep recorder + DAC DMA where they are
create_bd_addr_seg -range 0x40000000 -offset 0x00000000 \
    [get_bd_addr_spaces axi_ad9361_adc_dma/m_dest_axi] \
    [get_bd_addr_segs sys_ps7/S_AXI_ACP/ACP_DDR_LOWOCM] SEG_sys_ps7_ACP_DDR_LOWOCM
```

The AXI master must issue cacheable transactions for ACP to snoop:
AxUSER[0]=1 (the *_USER_VAL above) and AxCACHE with the cacheable bit
set (ADI's axi_dmac drives 0b0011; if that turns out not to allocate in
L2, override with a constant on the interconnect's AxCACHE or set the
DMAC's `AXI_AXCACHE`/`AXI_AXPROT` parameters if the IP version has them).

Then in each board DTS:

```dts
rx_dma: dma@7c400000 {
    ...
    dma-coherent;
};
```

With `dma-coherent` the arch DMA ops return cacheable memory from
`dma_alloc_coherent()` on their own and all `dma_sync_*` calls become
no-ops, so kernel patch 0008 is harmless but no longer necessary. Do
**not** add `dma-coherent` while the DMA is still on HP2: the CPU would
read stale cache lines.

Cost: a bitstream rebuild per board, and the 7010 boards have ~1% LUT
margin left (see `fpga_usage.md`); the ACP interconnect is similar in
size to the HP2 one it partly replaces, but measure.

### B. Zero-copy send in iiod (`MSG_ZEROCOPY`)

Once blocks are cacheable pages, the GEM can DMA straight out of them
and the CPU never touches sample bytes. Requirements:

1. The mapping must be `get_user_pages()`-able. Replace
   `remap_pfn_range()` in patch 0008 with a `vm_insert_page()` loop.
   That requires every page to be individually refcounted: pages from
   CMA (`alloc_contig_range` -> `split_page`) are, pages from the buddy
   fallback (`alloc_pages(order)`, non-compound) are not. Either force
   CMA (fail the allocation if `dma_alloc_contiguous` did not satisfy
   it) or allocate with `__GFP_COMP` and let the folio refcount cover
   the tail pages.
2. libiio patch under `board/tezuka/common/patches/libiio/` (picked up
   via `BR2_GLOBAL_PATCH_DIR`): in `iiod/ops.c` set `SO_ZEROCOPY` on the
   client socket, use `send(..., MSG_ZEROCOPY)` in `writefd_io()` for
   large writes, and before the next `iio_buffer_refill()` (which
   re-enqueues the block just sent) drain the socket error queue
   (`MSG_ERRQUEUE`) until the completion for that block arrives. With
   the default 4 blocks in flight this is a wait for the ACK of the
   block, ~1 ms on a LAN. A retransmission stall becomes an RX overflow
   instead of a silent data corruption, which is the right failure.
3. Hardware checksum + SG are already on (`ethtool -K eth0 tx on sg on`),
   which zero-copy frags need.

### C. Cheaper cache maintenance

- Skip the pre-DMA invalidate for input blocks when no writable mapping
  exists (iiod never writes RX blocks). Halves the maintenance cost.
- Or, with (A) done, drop it entirely.

### D. Per-packet path experiments (measure each alone)

- `CONFIG_HZ_250` and `CONFIG_PREEMPT_VOLUNTARY` in
  `board/tezuka/common/kernel/zynq_pluto_linux_defconfig`.
- `TCP_CONGESTION`/`SO_MAX_PACING_RATE` per socket in iiod instead of a
  global profile.
- `ethtool -G eth0 tx 1024` (macb supports up to 4096) if TX-ring-full
  stalls show up in `ethtool -S`.
- Client side: bigger blocks (4-8 MB) and more kernel buffers reduce
  wakeups; `plutorx -b` / `-k` expose this.

### E. Beyond ~55 Msps

Not reachable with TCP over the PS GEM. Boards whose RGMII goes through
the fabric (plutoskyr2: `sys_rgmii`) could carry a PL-side UDP
packetizer that bypasses the CPU entirely. Different project.

## Verification procedure

Do these in order; each step isolates one layer.

1. **Boot sanity**: `dmesg | grep -i "dma\|iio"` clean;
   `cat /sys/module/industrialio_buffer_dma/parameters/cached_mmap`
   prints 1; `cat /proc/irq/*/smp_affinity` shows eth0 on 1 and the
   two dmac IRQs on 2 (`grep -E "eth0|7c4" /proc/interrupts`).
2. **Data integrity before speed**: enable the DDS test tone
   (`siggen/*` via MQTT or `cf-ad9361-dds-core-lpc` attrs) in loopback,
   capture with `iio_readdev -u local: -b 1048576 cf-ad9361-lpc`
   with `cached_mmap=0` and `=1`, compare spectra. Any cache bug shows
   up as blocks of stale/zero samples or a periodic glitch at the block
   rate. Repeat for TX with `pluto_stream` or `iio_writedev`.
3. **Local ceiling** (no network): `sh tools/iio_benchmark.sh cached`
   vs `... uncached` (toggle the module parameter between runs). The
   buffer-size sweep is `iio_readdev` to `/dev/null`, i.e. the pure
   read-from-DMA-memory cost. Expect a large jump with cached blocks.
4. **TCP ceiling** (no IIO): `iperf3 -c <host> -t 20` from the device
   with `tcp_profile=wan` and `=lan`. This is the upper bound iiod can
   ever reach.
5. **End to end**: from the host, `plutorx -u ip:<board> -o 8 -p`
   (steps the sample rate until the FPGA overflow flag at reg 0x80000088
   trips), or `iio_readdev -u ip:<board> -b 4194304 cf-ad9361-lpc` with
   Q disabled, watching `top` per core on the device (`top -d 1`, press
   1). Success is 50 Msps cs8 with core 1 below ~85% and no overflow for
   several minutes.
6. **Thermals**: `iio_benchmark.sh` prints the AD9361 die temperature;
   a hotter chip at the same rate after the change means the CPU is
   doing less, not more.

## Risks to keep in mind

- Untested kernel patch on the DMA path. The failure mode of a cache
  maintenance mistake is corrupted samples, not a crash, so step 2 above
  is mandatory.
- All boards share the patch and the init scripts. The knobs default on
  everywhere; the 7010 USB-only boards gain nothing from it but also
  stream through the same block code over USB (FunctionFS), where the
  uncached copy costs just as much, so they may benefit too.
- `dma_alloc_attrs(NO_KERNEL_MAPPING)` needs the block to be physically
  contiguous in CMA or the buddy allocator, same as before. Block sizes
  above the 64 MB CMA area still fail, same as before.
- SoapyPlutoSDR / SDR++ / SatDump run on the host and use the network
  backend; they are unaffected by the kernel change but will see the
  throughput difference. On-device consumers of libiio (pluto_stream,
  iio_ws_proxy, classifier via maia-httpd) use these blocks directly.
