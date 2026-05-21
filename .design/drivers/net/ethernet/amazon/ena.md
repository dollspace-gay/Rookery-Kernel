# Tier-3: drivers/net/ethernet/amazon/ena/{ena_netdev,ena_com,ena_eth_com,ena_ethtool,ena_xdp,ena_devlink,ena_phc}.c — Amazon Elastic Network Adapter (every AWS EC2 instance)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/amazon/ena/ena_netdev.c              (~4380 lines: netdev open/close/start_xmit/poll, NAPI, RX/TX rings, RSS, MSI-X)
  - drivers/net/ethernet/amazon/ena/ena_netdev.h
  - drivers/net/ethernet/amazon/ena/ena_com.c                 (~3230 lines: device communication, admin queue, AENQ, MMIO regs, reset, CQ/SQ)
  - drivers/net/ethernet/amazon/ena/ena_com.h
  - drivers/net/ethernet/amazon/ena/ena_eth_com.c             (~655 lines: IO TX/RX descriptor fast-path, LLQ submission)
  - drivers/net/ethernet/amazon/ena/ena_eth_com.h
  - drivers/net/ethernet/amazon/ena/ena_ethtool.c             (~1165 lines: ethtool ops — link/coalesce/ring/stats/RSS/registers/eee/wol)
  - drivers/net/ethernet/amazon/ena/ena_xdp.c                 (~470 lines: XDP_DRV native + XDP_REDIRECT support)
  - drivers/net/ethernet/amazon/ena/ena_xdp.h
  - drivers/net/ethernet/amazon/ena/ena_devlink.c             (~215 lines: devlink params — LLQ size, large_llq_header, etc.)
  - drivers/net/ethernet/amazon/ena/ena_devlink.h
  - drivers/net/ethernet/amazon/ena/ena_phc.c                 (~235 lines: PTP hardware clock, PHC ops)
  - drivers/net/ethernet/amazon/ena/ena_phc.h
  - drivers/net/ethernet/amazon/ena/ena_debugfs.c
  - drivers/net/ethernet/amazon/ena/ena_admin_defs.h          (admin queue ABI — frozen against ENA spec)
  - drivers/net/ethernet/amazon/ena/ena_common_defs.h
  - drivers/net/ethernet/amazon/ena/ena_eth_io_defs.h         (IO descriptor ABI — bit-packed TX/RX descriptors)
  - drivers/net/ethernet/amazon/ena/ena_regs_defs.h           (MMIO register layout)
  - drivers/net/ethernet/amazon/ena/ena_pci_id_tbl.h
-->

## Summary

ENA (Elastic Network Adapter) is the NIC every AWS EC2 instance sees. On every `m5`/`c5`/`r5`/`p3`/`g4`/`i3`/`hpc6`/`bare-metal-XX` instance type and all newer generations, the virtio-net or Intel-82599 of older days has been replaced by a Nitro-hypervisor-presented ENA PCIe function. ENA is *not* virtio-net — it is a custom paravirtualized adapter designed by Amazon for the Nitro card architecture, with its own admin queue protocol, its own per-CPU IO queue layout (per-CPU SQ + CQ + MSI-X), Low-Latency Queue (LLQ) write-combined TX path that bypasses host RAM for descriptor writes, and host-RX-buffer-pool integration for zero-copy receive on the largest instances. ENA-Express (the newer flavor) adds 100Gbps+ per ENI plus SRD (Scalable Reliable Datagram) transport semantics for HPC.

The driver has two layers:

- **`ena_com`** (~3230 lines) — the device-communication layer: admin SQ/CQ, async event notification queue (AENQ), MMIO register access, device reset, queue (SQ/CQ) create/destroy commands. This is the "below netdev" layer and is intentionally bus-agnostic in upstream (although currently only the PCI path is used).
- **`ena_netdev`** (~4380 lines) — the Linux netdev wrapper: NAPI per IO queue, page-pool integration on RX, sk_buff/xdp_buff handling, RSS via firmware indirection table, MSI-X allocation, ethtool, devlink, PHC, XDP, error recovery, watchdog.

The fast path is in **`ena_eth_com`** (~655 lines): direct descriptor submission with LLQ optimization (TX descriptors written via write-combined MMIO directly into the host-NIC SQ memory, bypassing a host-RAM SQ).

This Tier-3 covers ~10K lines of upstream ENA driver. ENA is the single most-deployed NIC driver in the public cloud as of 2026: every AWS-hosted Linux VM has one or more.

## Rust translation posture

ENA is a clean target — well-structured upstream, layered (com / netdev / eth-com), explicit FW protocol, no SR-IOV (ENA functions are presented per-VM by Nitro, so each VM sees ENA functions that are already isolated). Translation:

- `ena_com_dev` → `Arc<EnaDev>` with phase typing (`EnaDev<Probed> → EnaDev<AdminQReady> → EnaDev<Online> → EnaDev<Resetting>`).
- Admin command submission becomes typed: `admin.create_io_sq(...) -> Result<SqHandle, AdminError>` rather than `ena_com_create_io_sq(dev, ctx)` with a context struct.
- AENQ (Async Event Notification Queue) handler dispatcher uses `match event.group { Link, Notification, Keepalive, ... }` — replaces the `aenq_handlers[group]` indirect call table with kCFI-signed exhaustive matching.
- The LLQ (Low-Latency Queue) write-combined TX descriptor path becomes a typed `LlqWriter<'_>` wrapping the WC-mapped MMIO region; `push_descriptor(&desc)` is an inlined `write_relaxed_u64` sequence with explicit `wmb`/`io_mb`.
- TX/RX descriptors (bit-packed structures from `ena_eth_io_defs.h`) → `bilge` typed bitfields with bounds-checked accessors.
- Per-NAPI ring state → `IoRing { sq: Sq, cq: Cq, txq: Option<TxQ>, rxq: Option<RxQ>, page_pool: PagePool, napi: Napi }`.
- Reset state machine → `EnaDev<Resetting>` is a transient phase reached on watchdog / TX timeout / AENQ keepalive miss; the recovery path either reaches `EnaDev<Online>` again or terminates the device.
- Devlink param plumbing → typed param registry; `large_llq_header` becomes `LlqHeaderConfig::{ Default, Large }`.
- XDP integration: standard NAPI-poll-with-XDP hook; XDP program install/uninstall go through the per-ring state; XDP_REDIRECT uses the kernel's xdp_do_redirect.

The grsec/PaX section is mandatory: ENA is the perimeter NIC for every AWS Linux VM, so it is the first thing an attacker who breaks the hypervisor boundary touches; it is also the only thing an in-VM attacker has to attack to reach the Nitro hypervisor.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ena_adapter` | per-PCI-function netdev priv (rings, MSI-X, devlink, watchdog) | `EnaAdapter` |
| `struct ena_com_dev` | hardware abstraction (admin q, regs, caps, AENQ, reset) | `EnaDev<Phase>` |
| `struct ena_ring` | per-IO-queue state (NAPI, page-pool, descriptors) | `IoRing` |
| `struct ena_com_io_sq` / `_cq` | per-queue SQ/CQ ring + MMIO mapping | `Sq` / `Cq` |
| `struct ena_admin_aq_entry` / `_acq_entry` / `_aenq_entry` | admin SQ entry / CQ entry / AENQ entry | `AdminCmd` / `AdminResp` / `AenqEvent` |
| `ena_probe(pdev, ent)` / `ena_remove(pdev)` | PCI probe / remove | `EnaAdapter::probe` / `Drop` |
| `ena_open(dev)` / `ena_close(dev)` | netdev open / close | `EnaAdapter::open` / `close` |
| `ena_start_xmit(skb, dev)` | TX entry | `EnaAdapter::start_xmit` |
| `ena_clean_rx_irq(rx_ring, napi, budget)` / `_tx_irq` | NAPI poll | `IoRing::poll` |
| `ena_com_admin_init(...)` / `_destroy_admin_queue(...)` | admin queue init / destroy | `AdminQ::new` / `Drop` |
| `ena_com_execute_admin_command(dev, cmd, cmd_size, resp, resp_size)` | sync admin cmd | `AdminQ::exec_sync` |
| `ena_com_create_io_queue(...)` / `_destroy_io_queue` | per-IO-queue create / destroy via admin | `EnaDev::create_io_queue` |
| `ena_com_io_queue_init(...)` | LLQ + IO queue ring init | `IoQueue::init` |
| `ena_com_prepare_tx(io_sq, ena_tx_ctx, &nb_hw_desc)` | bind sk_buff fragments to descriptors | `Sq::prepare_tx` |
| `ena_com_tx_comp_req_id_get(...)` / `_rx_pkt(...)` | TX completion / RX packet receive | `Cq::next_tx_comp` / `Cq::next_rx_pkt` |
| `ena_com_aenq_intr_handler(dev, data)` | AENQ irq handler | `Aenq::on_irq` |
| `ena_com_admin_aenq_enable(dev)` | enable AENQ groups | `Aenq::enable` |
| `ena_com_mmio_reg_read_request(dev, offset)` / `_read` | MMIO read via admin queue (some regs not directly readable) | `Mmio::read_via_admin` |
| `ena_com_dev_reset(dev, reset_reason)` | reset device with reason | `EnaDev::reset` |
| `ena_destroy_device(adapter, graceful)` / `_restore_device(...)` | full down/up around reset | `EnaAdapter::recover` |
| `check_for_missing_keep_alive(adapter)` / `_for_admin_com_state` | watchdog | `EnaAdapter::watchdog_tick` |
| `ena_indirection_table_set(adapter, indir)` | RSS indirection set | `EnaAdapter::set_rss_indir` |
| `ena_com_set_hash_function(dev, func, key, init_val)` | RSS hash func + key | `EnaDev::set_rss_hash` |
| `ena_xdp_install_rxq_info(...)` / `_register_rxq_info` | XDP rxq info | `XdpCtl::install` |
| `ena_xdp_xmit(dev, n, frames, flags)` | XDP_REDIRECT TX | `EnaAdapter::xdp_xmit` |
| `ena_phc_init(adapter)` / `_destroy` | PTP HW-clock | `Phc::init` |
| `ena_devlink_alloc(...)` / `_register(...)` | devlink registration | `Devlink::register` |
| `ena_get_drvinfo(...)` / `_get_ringparam(...)` / `_get_coalesce(...)` | ethtool ops | `Ethtool::*` |
| `ena_dim_work(work)` | dynamic interrupt moderation update | `Dim::on_tick` |

## Compatibility contract

REQ-1: PCI ID table — matches `0x1d0f:0xec20`, `0x1d0f:0xec21` (ENA classic and ENA-Express); ENA-SRD adds `0x1d0f:0x0061` for SRD. Frozen against `ena_pci_tbl[]`.

REQ-2: Admin queue protocol — fixed 32-entry SQ + 32-entry CQ at BARs declared via init regs. Opcodes (`ENA_ADMIN_*`) and the `ena_admin_aq_entry`/`ena_admin_acq_entry` structures frozen against `ena_admin_defs.h`. The 16-bit `command_id` round-trips between SQ and CQ entries; CQ entries lacking a matching SQ are dropped.

REQ-3: AENQ — async event notification queue, separate MSI-X vector; groups `LINK_CHANGE`, `NOTIFICATION`, `KEEPALIVE`, `REFRESH_CAPABILITIES`, `CONF_NOTIFICATIONS`; each event group has a handler in the driver. Keepalive miss (>watchdog interval, 6s default) triggers reset.

REQ-4: IO queue layout — N TX queues + N RX queues per VM, where N = `MIN(host_cpus, max_io_queues_per_dir)`. Each queue gets one MSI-X vector + one NAPI instance. Per-queue SQ entries hold descriptors pointing into per-queue ring buffers; CQ entries hold completions.

REQ-5: LLQ (Low-Latency Queue) — when supported (newer Nitro generations), TX SQ entries are written directly to the device via write-combined MMIO; saves a host-RAM bounce. Headers can be inlined up to a configurable size (`large_llq_header` devlink param: default 96B header, large = 224B). Bypass-by-feature: `disable_llq` devlink param falls back to host-RAM SQ.

REQ-6: RSS — 128-entry indirection table + 40-byte hash key (Toeplitz default) + per-protocol hash mask. Configurable via ethtool.

REQ-7: Watchdog — three watchdogs: keep-alive miss (AENQ not heard), admin-q state stuck (admin cmd not completing), TX timeout (per-queue). Any trip → device reset with reason code.

REQ-8: Device reset — admin command `RESET_DEVICE` or via the MMIO reset trigger; full driver re-init follows; pre-reset stats preserved and reported through ethtool counters.

REQ-9: XDP_DRV — native XDP at the RX-NAPI poll path; XDP_REDIRECT supported; XDP_TX uses dedicated XDP TX rings (created at `bpf attach` time).

REQ-10: PTP — when supported (ENA-Express on certain Nitro generations), PHC clock exposed at `/dev/ptpN`; supports per-packet TX/RX timestamping.

REQ-11: Devlink params — `large_llq_header` (default/large), `enable_phc` (on/off), other ENA-specific knobs preserved.

REQ-12: Ethtool counters — full per-ring TX/RX byte/packet counters + ENA-specific error counters (`bad_csum`, `bad_req_id`, `bad_desc_num`, `rx_drops`, `tx_drops`, `tx_dropped_unmask_intr`, `rx_buf_alloc_failed`, etc.) — frozen against upstream stat names so existing monitoring works.

REQ-13: SRIOV — ENA does not implement SR-IOV in the traditional sense; per-VM functions are presented by Nitro. The driver therefore has no `pci_sriov_configure` callback.

REQ-14: AER — `pci_err_handlers` with `ena_pci_io_error_detected` / `_pci_io_slot_reset` / `_pci_io_resume`. Mandatory because Nitro card surfaces PCIe AER on internal failures.

## Acceptance Criteria

- [ ] AC-1: `lspci` on an `m5.large` instance shows `1d0f:ec20` ENA; Rookery probes; `dmesg | grep ena` matches upstream init log lines.
- [ ] AC-2: `ip link` shows `eth0` UP with MAC matching `/sys/class/net/eth0/address` set by ENA admin; `ping` to the VPC gateway works.
- [ ] AC-3: `iperf3 -P 8` from a `c5n.18xlarge` (100 Gbps) sustains ≥80 Gbps single-VM (matching upstream baseline).
- [ ] AC-4: LLQ enabled by default on `m5n.*`/`c5n.*` instances; `ethtool -i eth0` shows `large_llq_header` capability; `devlink dev param show` reports the current setting.
- [ ] AC-5: ethtool RSS: `ethtool -X eth0 weight 1 2 3 4 ...` updates the indirection table; per-CPU softirq distribution observable in `mpstat`.
- [ ] AC-6: XDP smoke: `xdp-loader load eth0 xdp_pass.o` succeeds; `xdp-loader status eth0` shows attached; XDP_REDIRECT between two ENAs in the same VM (multi-NIC) works.
- [ ] AC-7: Watchdog recovery: simulate keepalive-miss by blocking AENQ MSI-X handler in debugfs; watchdog trips at 6s; device resets; netdev comes back online.
- [ ] AC-8: Hot-add NIC (AWS in-VM ENA add): `echo 1 > /sys/bus/pci/rescan` after attaching a second ENI in the AWS console; new eth1 appears.
- [ ] AC-9: PTP: on a supported instance, `ptp4l -i eth0 -m` syncs to the Nitro card's PHC; offset stable within ±100ns.
- [ ] AC-10: PCIe AER injection: simulated `pci_io_error_detected` event → driver brings the device down, slot-reset hook brings it back up; traffic resumes.

## Architecture

**Two-layer split.** `ena_com` (HAL) is intentionally bus-agnostic — admin q, AENQ, MMIO regs, queue create/destroy. `ena_netdev` is the Linux integration: NAPI per queue, page-pool RX, sk_buff/xdp_buff handling, RSS, ethtool, devlink. Rookery preserves this split: `EnaDev` is the HAL, `EnaAdapter` is the netdev wrapper that owns an `Arc<EnaDev>` plus per-queue `IoRing` instances.

**Admin queue.** A 32-entry submission queue + 32-entry completion queue + a 16-bit `command_id` allocator. Each command in flight reserves a slot; the slot stores the issuer's `Notify` so the completion path can wake the right waiter. Slot-allocator is a simple `AtomicU32` bitmap. Commands serialize through a `Mutex` (the admin q is not concurrent-submit-safe in upstream). Polling vs interrupt-driven completion is a build-time choice; default is interrupt-driven (admin-q MSI-X vector shared with AENQ vector).

**AENQ.** A separate ring polled by an MSI-X vector. Event groups (`LINK_CHANGE`, `NOTIFICATION`, `KEEPALIVE`, etc.) each have a registered handler. Keepalive events update a per-adapter `last_keepalive_jiffies`; the watchdog ticks every 1s and resets the device if no keepalive in `keep_alive_timeout` (6s default).

**Per-queue IO.** Each (TX, RX) queue pair gets:
- An MSI-X vector with NAPI instance.
- A page-pool for RX buffer allocation (zero-alloc per-packet on the steady state).
- A `Sq` (SQ ring) + `Cq` (CQ ring) — for non-LLQ, both are in host RAM; for LLQ TX, the SQ is in device-mapped WC memory.
- A DIM (Dynamic Interrupt Moderation) instance — adapts coalesce timer/packet thresholds.

The NAPI poll cleans the CQ: for RX, builds an sk_buff (or xdp_buff if XDP attached) per completion, batches them through the stack/XDP; for TX, frees the sk_buff after the firmware confirms transmit. Budget cap respected; if drained, `napi_complete_done` and re-enable IRQ; else reschedule.

**LLQ fast path.** When LLQ supported, `ena_com_prepare_tx` writes the SQ entry directly through `__iowrite64_copy` (write-combined) into the device-mapped SQ region. No host RAM SQ; the doorbell is the act of writing the last descriptor word. This shaves several µs off the TX path — the design point for the latency-sensitive workloads (HPC, low-latency trading on `c5n.metal`).

**Reset state machine.** Triggered by: AENQ keepalive miss, TX timeout per ring, admin-q stuck (admin cmd not completing in ADMIN_CMD_TIMEOUT_MS), or explicit AER notification. Sequence: stop all NAPI → drain TX queues → issue `RESET_DEVICE` admin cmd (or MMIO trigger) → wait for device-ready bit → re-init admin q → re-create IO queues → restart NAPI. Pre-reset stats preserved in `ena_stats_dev` (sticky counters survive reset).

**Page-pool integration.** RX rings allocate buffers through `page_pool_alloc_pages`. On free, pages are returned to the pool; on completion, the page is either recycled (skb head reuse) or released to the page-pool's freelist. XDP_REDIRECT uses `xdp_return_frame_napi` to return to the pool.

## Hardening

- All admin commands validate the CQ entry's `command_id` against the in-flight bitmap; CQ entries with unknown command_id are dropped (no use of stale `ena_comp_ctx`).
- AENQ events validate `group` against `aenq_handlers` length; out-of-range groups are dropped.
- MMIO read-via-admin path has a 200ms timeout; stuck admin q triggers reset (rather than wedging the driver).
- Per-ring `req_id_buf` (TX/RX descriptor in-flight tracking) bounds-checked on every descriptor; a corrupted FW returning out-of-range req_id triggers reset rather than UAF.
- Reset reason recorded and reported through devlink; differentiated reasons help post-mortem.
- Watchdog work is rescheduled via a single per-adapter `delayed_work`; cancellation on driver removal ensures no stale watchdog after `pci_remove`.
- All sk_buff freed via `dev_kfree_skb_any` (handles both irq + thread contexts).
- RX checksum offload trusted only when `ena_admin_feature_offload_desc.rx_supported.l3_csum`/`l4_csum_ipv4`/`l4_csum_ipv6` advertised; otherwise CHECKSUM_NONE.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool ioctl ingress (RSS key, indirection table, coalesce params) goes through bounded `copy_from_user`; devlink param values nlattr-validated.
- **PAX_KERNEXEC** — `ena_pci_id_tbl[]`, `aenq_handlers[]`, ethtool_ops vtable, devlink_param ops, netdev_ops, dma_ops, pci_error_handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / devlink / netlink / sysfs entry paths are per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `EnaDev` refcount, per-`IoRing` refcount, per-page-pool refcount use saturating refcounts; underflow at `ena_remove` is a hard panic.
- **PAX_MEMORY_SANITIZE** — admin SQ/CQ DMA buffers are `kfree_sensitive`-equivalent (zeroed on free) — they carry MAC addresses, RSS keys, and per-flow hash material. Per-ring `req_id_buf` zeroed on ring destroy.
- **PAX_UDEREF** — SMAP/SMEP enforced on every user-facing entry (ethtool, devlink, ptp ioctl).
- **PAX_RAP / kCFI** — `netdev_ops`, `ethtool_ops`, `aenq_handlers`, `pci_error_handlers`, `xdp_metadata_ops` are kCFI-signed; AENQ dispatch is exhaustive-match on the event group (no indirect-call table beyond the kCFI-signed handler set).
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/ena/<pci-bdf>/*`) dumps include SQ/CQ ring pointers, admin-q context pointers; non-root reads scrub addresses.
- **GRKERNSEC_DMESG** — ENA AENQ "NOTIFICATION" events carry vendor-tagged messages from Nitro (e.g. "instance reboot scheduled at T"); restricted from non-root dmesg.
- **CAP_NET_ADMIN** required on ethtool ops that change MAC, RSS, coalesce, ring size; devlink param sets; PTP-clock-adjust ops.
- **Nitro-side trust boundary explicit** — Nitro hypervisor is presumed adversarial in the same way the QEMU/KVM host is presumed adversarial for confidential-computing guests. AENQ event payloads, admin CQ entries, and CQ-entries-with-attacker-controlled-data (e.g. RX completion fields) are validated against expected ranges before consumption. A malicious/compromised Nitro card cannot drive a kernel oops by sending malformed admin responses.
- **AENQ event payload allowlist** — only `LINK_CHANGE`, `NOTIFICATION`, `KEEPALIVE`, `REFRESH_CAPABILITIES`, `CONF_NOTIFICATIONS` are accepted; other group values are dropped with a rate-limited warn. Closes a path where Nitro could push an unknown group and trigger handler-table OOB.
- **`req_id` bounds-checked in fast path** — every TX/RX completion's req_id is validated against the per-ring max before dereferencing the corresponding tracking entry. A compromised Nitro card cannot use a forged req_id to corrupt driver state.
- **Reset rate-limit** — `ena_com_dev_reset` rate-limited to 3 resets per minute; further reset requests within the window result in permanent driver-down rather than infinite-loop. Closes "Nitro is gaslighting us into endless reset" class.
- **LLQ MMIO bound** — the LLQ MMIO window is mapped exactly once at probe and the size is validated against the cap-advertised `llq_mem_size`. No re-map at runtime; an out-of-band attempt to expand the window is refused.
- **RSS-key entropy enforced** — `ena_rss_init_default` uses kernel CSPRNG (`get_random_bytes`); never accepts an externally provided default key. Closes a path where a compromised firmware could hint a weak RSS key to enable easier flow inference.
- **Watchdog cancel ordering strict** — `ena_remove` cancels the watchdog work *before* tearing down the device; no path where a watchdog tick runs against a torn-down device. Closes UAF-class.
- **AER recovery rate-limit** — if AER fires more than 3x in 60s, give up and leave device offline; same rationale as mlx5.
- **XDP program install gated** — `bpf attach` requires `CAP_NET_ADMIN`; the in-VM bpf-program installer must be root, not an unprivileged container user.
- **Debugfs CAP_SYS_ADMIN** — ENA debugfs writes (reset-trigger, ring-state dump, AENQ-replay) all require CAP_SYS_ADMIN; reads scrub pointers.

Per-doc rationale: ENA is the cloud-perimeter NIC. The trust boundary that matters is **Nitro-card-as-adversary**, not "remote network attacker through the wire" — packets arriving over the wire are already wrapped in VPC encapsulation that Nitro maintains; what matters is whether a Nitro card that the VM cannot trust (because Nitro can be compromised by AWS-side issues, supply-chain, or other tenants on the same host pre-Nitro-isolation) can drive the in-VM driver into UAF, OOB read, or RCE. The reinforcement above is uniformly oriented at this boundary: AENQ payload allowlist, req_id bounds-check, MMIO window pinned at probe, admin CQ validation, reset rate-limit. The Rust translation's typed-state machine closes additional bug classes (calling `create_io_queue` before `admin_init` — compile error). Combined, the design treats ENA as a hostile-firmware-controlled adapter — the right posture for cloud-deployment.

## Open Questions

- [ ] Q1: SRD (Scalable Reliable Datagram) support — ENA-Express adds SRD for HPC workloads. The SRD admin commands and per-flow state are not yet upstream-merged in baseline 7.1.0-rc2. Defer SRD to a follow-up doc or include as REQ-future?
- [ ] Q2: LLQ default — upstream defaults to LLQ on supported Nitro generations. Rookery should match. Question: should the `large_llq_header` (224B) default-on for `c5n.metal`/`m5n.metal` where workloads benefit, or stay at the conservative 96B default?
- [ ] Q3: NAPI-thread mode — Linux 5.x added per-NAPI thread mode. ENA supports it (`napi_set_threaded`). Rookery's NAPI design needs to decide whether per-queue threaded NAPI is opt-in (default off, like upstream) or opt-out.
- [ ] Q4: ENA + bonding — multiple ENI on one VM bonded via Linux bond driver. ENA driver has no offload for this (bond runs in software). Should Rookery offer a vendor-specific LAG-equivalent (like mlx5_lag)? Probably not — ENA's design point is one ENI per VM scaled vertically; the multi-ENI case is rare and the software bond is fine.
- [ ] Q5: Confidential computing — AWS Nitro Enclaves use the same ENA driver. Should the driver have a "confidential mode" flag that tightens the Nitro-as-adversary posture even further (e.g. refuse any admin command not on a strict opcode allowlist)?

## Verification

- **Kani SAFETY**: prove that `Sq::reserve_slot` cannot return a slot index already in use (bitmap-allocator invariant). Prove that the admin command-id allocator never wraps before all in-flight cmds complete.
- **TLA+**: model the reset state machine, including watchdog races with admin-q completion. Check that any path that begins a reset reaches either `Online` (recovered) or `Failed` (terminal) within bounded steps, with no leaked IO queues.
- **Verus**: functional spec of `ena_com_prepare_tx` — given a sk_buff with valid fragment list (≤max_descs, total length ≤MTU), produces a sequence of SQ descriptors whose concatenation matches the sk_buff's data, with the correct LSO/csum/header-inline flags set per the offload caps.
- **Kani+Verus**: invariant that every TX descriptor pushed into the SQ has a corresponding entry in `req_id_buf`; every TX completion's req_id is in `req_id_buf` (no completion for an un-submitted descriptor).
- **Integration**: full AWS deployment matrix smoke — boot Rookery on `t3.nano`, `m5.large`, `c5n.18xlarge`, `c5n.metal`, `hpc6id.32xlarge`; ensure ENA probes and reaches `iperf3` line-rate per instance type. Hot-plug ENI test on each. Watchdog test by blocking AENQ vector in debugfs and confirming recovery.
- **Fuzz**: AENQ-event fuzzer (mutate group/seq/timestamp/payload) — should never produce an oops. Admin-CQ fuzzer (return malformed completions) — should always trigger reset rather than UAF.
- **Penetration**: in-VM unprivileged container attempts to use ethtool/devlink/XDP-attach against ENA — all must be CAP_NET_ADMIN-gated.

## Out of Scope

- AWS Nitro hypervisor internals (ENA's host side) — outside Linux kernel scope
- AWS VPC encapsulation, ELB, NLB — network-layer concerns above the NIC
- SRD (Scalable Reliable Datagram) — future iteration once upstream merges full SRD support
- AWS EFA (Elastic Fabric Adapter, separate Mellanox-derived driver `efa/`) — separate Tier-3 (`drivers/infiniband/hw/efa/`)
- NICE DCV / Trainium / Inferentia accelerator integration — separate accelerator-driver docs
- Multi-NIC vrf/bond setups — software-only, covered in net/ trees
