# Tier-3: drivers/net/ethernet/google/gve/{gve_main,gve_adminq,gve_tx,gve_rx,gve_tx_dqo,gve_rx_dqo,gve_buffer_mgmt_dqo,gve_flow_rule,gve_ethtool,gve_ptp}.c — Google gVNIC (every GCE instance with gVNIC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/google/gve/gve_main.c                  (~3030 lines: PCI probe/remove, netdev ops, MSI-X, NAPI, queue create/destroy)
  - drivers/net/ethernet/google/gve/gve.h
  - drivers/net/ethernet/google/gve/gve_adminq.c                (~1600 lines: admin queue protocol — describe_device, configure_device, queue create, RSS, flow rules)
  - drivers/net/ethernet/google/gve/gve_adminq.h
  - drivers/net/ethernet/google/gve/gve_tx.c                    (~1040 lines: GQI legacy TX — copy QPL + segmented descriptors)
  - drivers/net/ethernet/google/gve/gve_rx.c                    (~1095 lines: GQI legacy RX — copy QPL + per-queue page pool)
  - drivers/net/ethernet/google/gve/gve_tx_dqo.c                (~1615 lines: DQO TX — desc-queue-out, zero-copy, multi-segment LSO)
  - drivers/net/ethernet/google/gve/gve_rx_dqo.c                (~1145 lines: DQO RX — desc-queue-out, header-data split, RSC)
  - drivers/net/ethernet/google/gve/gve_buffer_mgmt_dqo.c       (~345 lines: DQO buffer pool — header pool + payload pool)
  - drivers/net/ethernet/google/gve/gve_flow_rule.c             (~300 lines: ethtool ntuple flow rule install / del / list)
  - drivers/net/ethernet/google/gve/gve_ethtool.c               (~1015 lines: ethtool ops — stats, ring, RSS, ntuple, link, coalesce)
  - drivers/net/ethernet/google/gve/gve_ptp.c                   (~160 lines: PHC integration)
  - drivers/net/ethernet/google/gve/gve_utils.c
  - drivers/net/ethernet/google/gve/gve_dqo.h
  - drivers/net/ethernet/google/gve/gve_desc.h                  (GQI descriptor ABI — copy QPL)
  - drivers/net/ethernet/google/gve/gve_desc_dqo.h              (DQO descriptor ABI — zero-copy ring)
  - drivers/net/ethernet/google/gve/gve_register.h              (MMIO register layout)
-->

## Summary

gVNIC (Google Virtual NIC) is the NIC every GCE (Google Compute Engine) instance with `gvnic` networking sees. Like AWS ENA, it's a custom paravirt NIC presented by Google's host-side network stack (Andromeda) — not virtio-net. Every modern GCE instance type — `n2`, `c3`, `c3d`, `h3`, `a2`/`a3`/`a4` GPU, `m3`/`m4`, `n4`, `t2d`, `t2a` — uses gVNIC; the older `virtio_net` path is legacy and reserved for first-gen `n1`-with-virtio choice.

gVNIC has gone through a generational shift in its descriptor format:

- **GQI** (Google Queueing Interface, legacy) — `gve_tx.c` + `gve_rx.c`. Uses Queue-Page-List (QPL) — each ring has a pre-registered set of host pages; TX/RX descriptors reference *slots* in the QPL rather than DMA addresses. The host stack does memcpy in/out of the QPL slot. Available on most instance types but soft-deprecated.
- **DQO** (Desc-Queue-Out, newer) — `gve_tx_dqo.c` + `gve_rx_dqo.c`. True DMA-based descriptors with header-data split on RX, multi-segment LSO on TX, zero-copy throughout. The default mode on c3+ generations and required for >50 Gbps line rates.

Both modes coexist; per-queue mode is negotiated at queue-create time via the admin queue. RX in DQO also splits header buffers and payload buffers into separate page pools (`gve_buffer_mgmt_dqo.c`), so the NIC can land headers in a small pool (better cache) and payloads in a large pool (page-aligned for zero-copy splice).

The driver also supports XDP_DRV, ethtool ntuple flow rules (`gve_flow_rule.c` — installable HW filter rules), PHC clock (`gve_ptp.c`), full ethtool stats.

This Tier-3 covers ~11.4K lines of upstream gve driver.

## Rust translation posture

gVNIC has a clean two-mode architecture (GQI vs DQO) that maps well onto Rust enums-with-data. The translation:

- `gve_priv` → `Arc<GvePriv>` with phase typing (`Probed → DescribedDevice → ConfiguredDevice → Online`).
- Per-queue mode → `enum QueueMode { Gqi(GqiQueue), Dqo(DqoQueue) }`. NAPI poll dispatches via `match` (exhaustive, kCFI-signed). The two modes share zero per-packet code paths.
- QPL (Queue Page List) handle → typed `Qpl { pages: Vec<DmaPage>, id: QplId }`. Slot index validated against `pages.len()` on every access.
- DQO descriptors (bit-packed structs from `gve_desc_dqo.h`) → `bilge` typed bitfields.
- Admin queue → `AdminQ` actor with typed methods: `admin.describe_device() -> DescribeDeviceResponse`, `admin.configure_device(opts) -> Result<...>`, `admin.create_tx_queue(opts) -> Result<TxQueueId, AdminError>`, etc.
- Flow-rule install via ntuple → typed `FlowRule { match_fields, action, location: FlowRuleId }` with bounded count enforced by the driver.
- DQO buffer management — separate `HeaderPool` and `PayloadPool` types; cannot mix.
- Reset state — like ENA, gVNIC has watchdog-triggered reset; `GvePriv<Resetting>` is the transient phase.

The grsec/PaX section is mandatory: gVNIC's trust boundary is **Andromeda-host-as-adversary**, parallel to AWS Nitro. The same principle applies — assume the hypervisor-side may push malformed admin responses, AENQ-equivalent events, or completion descriptors.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gve_priv` | per-PCI-function netdev priv (queues, MSI-X, admin, devlink-ish) | `GvePriv<Phase>` |
| `struct gve_tx_ring` / `gve_rx_ring` | per-queue ring state (mode-discriminated union) | `TxRing` / `RxRing` (enum-typed) |
| `struct gve_adminq_command` | admin queue command | `AdminCmd` (enum) |
| `struct gve_queue_page_list` | per-ring host-page registry (QPL) | `Qpl` |
| `gve_probe(pdev, ent)` / `gve_remove(pdev)` | PCI probe / remove | `GvePriv::probe` / `Drop` |
| `gve_open(dev)` / `gve_close(dev)` | netdev open / close | `GvePriv::open` / `close` |
| `gve_tx(skb, dev)` / `gve_tx_dqo(skb, dev)` | TX entry (mode-specific) | `TxRing::xmit` |
| `gve_napi_poll(napi, budget)` / `_dqo(...)` | NAPI poll | `Napi::poll` |
| `gve_adminq_alloc(dev, priv)` / `_free(...)` | admin queue alloc / free | `AdminQ::new` / `Drop` |
| `gve_adminq_execute_cmd(priv, cmd_orig)` | sync admin cmd | `AdminQ::exec` |
| `gve_adminq_describe_device(priv)` | DESCRIBE_DEVICE — cap/option enumeration | `AdminQ::describe_device` |
| `gve_adminq_configure_device_resources(...)` | CONFIGURE_DEVICE_RESOURCES — counter / report blocks | `AdminQ::configure_device` |
| `gve_adminq_create_tx_queue(...)` / `_create_rx_queue(...)` | CREATE_TX_QUEUE / CREATE_RX_QUEUE | `AdminQ::create_tx_queue` / `_rx` |
| `gve_adminq_destroy_tx_queue(...)` / `_destroy_rx_queue(...)` | inverse | `AdminQ::destroy_*` |
| `gve_adminq_register_page_list(...)` / `_unregister_page_list(...)` | REGISTER_PAGE_LIST (QPL) | `AdminQ::register_qpl` / `unregister_qpl` |
| `gve_adminq_configure_rss(priv, rxfh)` | RSS hash + indir table set | `AdminQ::set_rss` |
| `gve_adminq_get_ptype_map_dqo(...)` | PTYPE map (per-packet-type info) for DQO RX | `AdminQ::get_ptype_map` |
| `gve_adminq_set_flow_rule(...)` | install ntuple flow rule | `AdminQ::set_flow_rule` |
| `gve_adminq_delete_flow_rule(priv, loc)` | delete | `AdminQ::del_flow_rule` |
| `gve_adminq_query_flow_rules(...)` | dump | `AdminQ::query_flow_rules` |
| `gve_adminq_report_stats(priv, stats_report_len, stats_report_addr, interval)` | NIC-side stats DMA push | `AdminQ::report_stats` |
| `gve_adminq_verify_driver_compatibility(...)` | NIC↔driver version handshake | `AdminQ::verify_compat` |
| `gve_handle_reset(priv)` / `gve_reset(priv, attempt_teardown)` | reset state machine | `GvePriv::handle_reset` |
| `gve_service_task(work)` | timer-driven stats-report + reset poll | `Service::tick` |
| `gve_alloc_buffer(rx, buf_state)` / `_recycle_buffer(...)` | DQO RX buffer alloc/recycle | `BufferPool::alloc` / `recycle` |
| `gve_rx_post_buffers_dqo(rx)` | refill DQO RX desc queue | `RxRing::post_buffers` |
| `gve_tx_buf_alloc_dqo(tx, ...)` / `_buf_clean_dqo(...)` | TX buffer alloc / clean DQO | `TxBufPool::alloc` / `clean` |
| `gve_rx_skb_alloc_hsplit_dqo(...)` | RX skb with header-data-split | `RxRing::build_skb_hsplit` |
| `gve_flow_rules_set(...)` / `_get(...)` | ethtool ntuple wrappers | `Ethtool::flow_rules` |
| `gve_xdp_xmit(dev, n, frames, flags)` | XDP_REDIRECT TX | `GvePriv::xdp_xmit` |
| `gve_ptp_init(priv)` / `_release` | PHC init | `Phc::init` |

## Compatibility contract

REQ-1: PCI ID table — matches `0x1ae0:0x0042` (PCI_DEV_ID_GVNIC). Frozen.

REQ-2: Two-BAR layout — BAR0 = registers (admin q control, version, doorbell-bar-info, device-status), BAR2 = doorbells (one per queue). Mapped at probe; never re-mapped.

REQ-3: Admin queue protocol — fixed in `gve_adminq.h`. 64-entry SQ/CQ. Each cmd has an opcode + opcode-specific body. CMD_OK on success; non-zero status on failure. Frozen against `enum gve_adminq_opcodes`.

REQ-4: Device capability discovery — `DESCRIBE_DEVICE` returns device-options TLV list with options including: `GVE_DEV_OPT_RSS_CONFIG`, `GVE_DEV_OPT_DQO_RDA`, `GVE_DEV_OPT_DQO_QPL`, `GVE_DEV_OPT_JUMBO_FRAMES`, `GVE_DEV_OPT_BUFFER_SIZES`, `GVE_DEV_OPT_NIC_TIMESTAMPS`, `GVE_DEV_OPT_MODIFY_RING`, `GVE_DEV_OPT_FLOW_STEERING`. Driver enables features based on what options are present.

REQ-5: Per-queue mode — at queue create time, the queue is either GQI (legacy QPL) or DQO (descriptor-out). Driver default per upstream `GVE_DEFAULT_*` constants + capability negotiation.

REQ-6: QPL (Queue Page List) registration — for GQI queues, a list of host pages is DMA-mapped and registered with the NIC via `REGISTER_PAGE_LIST`. NIC writes RX data into these pages; driver reads. TX is the reverse. Pages are recycled — driver acknowledges per-slot.

REQ-7: DQO ring layout — TX has a primary desc ring + completion ring; RX has a desc ring + completion ring + dedicated buffer pools (header + payload). All in DMA-mapped contiguous regions.

REQ-8: RX header-data split (DQO only) — when `GVE_DEV_OPT_BUFFER_SIZES` advertises header pool size > 0, NIC splits header+payload into separate buffers; sk_buff is built with head from header pool and frag from payload pool.

REQ-9: MSI-X — one vector for management (admin q + AENQ-equivalent stats-report), one vector per TX queue, one per RX queue. Affinity hints set per CPU.

REQ-10: NAPI per queue — per-queue NAPI instance; budget respected; XDP attached per-RX-queue.

REQ-11: Reset state machine — watchdog detects: device-status reporting reset-required, admin q stuck, stats-report missed. Driver re-init via `gve_reset`. Recovery preserves netdev state where possible.

REQ-12: Stats-report — driver allocates a DMA-mapped block; NIC writes per-queue stats periodically (every 20s); driver reads through ethtool counters.

REQ-13: Flow rules — `ethtool -N eth0 flow-type ...` installs HW filter rules via `SET_FLOW_RULE` admin cmd. Per-device limit reported by NIC; driver enforces.

REQ-14: PTP — when `GVE_DEV_OPT_NIC_TIMESTAMPS` present, `/dev/ptpN` exposed; per-packet TX/RX timestamping via standard PHC API.

REQ-15: XDP — XDP_DRV native, XDP_REDIRECT supported; dedicated XDP TX rings created at attach time for XDP_TX.

## Acceptance Criteria

- [ ] AC-1: `lspci` on a `c3-standard-22` instance shows `1ae0:0042` gVNIC; Rookery probes; `dmesg | grep gve` matches upstream log lines.
- [ ] AC-2: `ip link` shows `ens4` UP with MAC; gateway reachable.
- [ ] AC-3: `iperf3` between two `c3-standard-22` VMs in the same zone sustains >50 Gbps (gVNIC c3 line rate); single-flow gets full bandwidth thanks to DQO + RSS.
- [ ] AC-4: `ethtool -i ens4` shows driver `gve`, firmware version from NIC; `ethtool -k` lists offloads matching upstream (LSO TSO GSO checksum scatter-gather).
- [ ] AC-5: RSS: `ethtool -X ens4 ...` updates indirection; per-CPU softirq spread observable.
- [ ] AC-6: ntuple flow rule: `ethtool -N ens4 flow-type tcp4 src-ip 1.2.3.4 dst-port 80 action 5` installed; `ethtool -n ens4` lists it; per-rule counter increments.
- [ ] AC-7: XDP smoke: `xdp-loader load ens4 xdp_pass.o` succeeds.
- [ ] AC-8: GQI fallback: on a `n2-standard-2` (where DQO not available), driver creates queues in GQI mode using QPL.
- [ ] AC-9: Watchdog recovery: simulate device-status RESET_REQUIRED via debugfs; `gve_reset` fires; netdev recovers.
- [ ] AC-10: PTP on supported instance: `ptp4l -i ens4` syncs to NIC PHC.

## Architecture

**Two-mode descriptor format.** GQI uses QPL — each ring has a registered set of host pages, descriptors reference slot indices. The host copies sk_buff data into the QPL slot at TX and out of the slot at RX. DQO uses true DMA — TX descs point to DMA addresses of sk_buff frags, RX descs point to NIC-supplied buffer pages from the buffer pool. Per-queue, the mode is decided at create time and immutable for the queue lifetime; mode mixing within a netdev is allowed (different queues different modes) but rare.

**QPL lifecycle.** At ring creation, driver allocates N pages (size determined by ring size + per-buffer size), DMA-maps them, and pushes via `REGISTER_PAGE_LIST` admin cmd. NIC binds the QPL to the queue. Per-slot ack: as the host consumes RX slots, it re-advertises them to the NIC via a doorbell write. Pages are owned by the driver throughout; QPL is a slot-rental rather than a buffer-handoff.

**DQO RX with header-data split.** The buffer manager (`gve_buffer_mgmt_dqo.c`) keeps two pools: `header_buf_pool` (typically 256B per buf) and `payload_buf_pool` (page-sized). When the NIC has a packet, it writes the header into a header buffer and the payload into a payload buffer; the RX completion descriptor includes a pointer to each. Driver builds sk_buff with `__build_skb` from the header buffer + adds the payload buffer as a frag. Header pool stays cache-warm (small, frequently reused); payload pool serves zero-copy splice/TCP-recvzc paths.

**Admin queue.** Like ENA's, a fixed-depth SQ + CQ at a DMA-mapped region. Commands serialized by `adminq_lock` mutex; in-flight cmd waits on a `completion`. Cmd timeout is 10s; on timeout, driver triggers a device reset.

**Reset state machine.** Triggered by: `device_status` reg reporting RESET_REQUESTED, admin-q cmd timeout, stats-report missed for N intervals, TX timeout per ring. Sequence: cancel watchdog → unregister netdev queues (NAPI off) → destroy queues via admin → destroy admin → tear down PCI mapping → re-map → re-init admin → DESCRIBE_DEVICE → CONFIGURE_DEVICE_RESOURCES → re-create queues → restore RSS → re-register netdev queues. Stats counters survive (sticky).

**Flow rules.** Ethtool ntuple → translates to admin-q `SET_FLOW_RULE` with match fields + action. Driver maintains a parallel cache of installed rules for `ethtool -n` listing. Per-NIC rule count cap enforced by admin q's `query_flow_rules` reporting.

**XDP integration.** XDP attach allocates dedicated XDP TX queues (separate from netdev TX); XDP_REDIRECT TX goes through these. XDP RX runs in the NAPI poll path; XDP_DROP/PASS/TX/REDIRECT all supported. Per-RX-queue XDP program install.

**Service task (timer).** Single per-priv `delayed_work` that wakes every `service_timer_period` (default 1s). Checks device-status, dispatches reset if needed, refreshes stats-report DMA target.

## Hardening

- Admin cmd timeout (10s) is hard; never wait indefinitely.
- All admin CQ entry statuses validated; non-OK → driver-side error path, no fallthrough.
- `device_options` TLV walker has bounded loop (limited by `descriptor_length`); a NIC sending a malformed/oversized TLV cannot OOM the driver.
- QPL slot indices bounds-checked on every TX/RX descriptor.
- DQO buffer-pool counts derived from NIC caps; bounded.
- Flow-rule install validates user-supplied fields against the `flow_rule_max_match_fields` cap before admin cmd.
- Reset is rate-limited (3 resets / 60s); further requests permanent-down.
- DMA mappings of admin q / QPL / buffer-pool freed on `gve_remove`; no leaks.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool ioctl ingress (flow-rule fields, RSS key, indir table, coalesce) goes through bounded `copy_from_user`.
- **PAX_KERNEXEC** — `gve_netdev_ops`, `gve_ethtool_ops`, admin-cmd dispatch tables, XDP TX rings static layout, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / netlink / sysfs / ptp ioctl entry paths are per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `GvePriv` refcount, per-queue refcount, QPL refcount, buffer-pool refcount use saturating refcounts; underflow at `gve_remove` is a hard panic.
- **PAX_MEMORY_SANITIZE** — admin SQ/CQ DMA buffers `kfree_sensitive` (carry MAC, RSS keys, flow-rule keys). QPL pages zeroed on registration release. Buffer-pool pages zeroed on pool destroy.
- **PAX_UDEREF** — SMAP/SMEP enforced on every user-facing entry path.
- **PAX_RAP / kCFI** — `gve_netdev_ops`, `gve_ethtool_ops`, NAPI poll dispatch (`match` exhaustive on QueueMode), admin-cmd opcode dispatch, XDP-prog ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/gve/<bdf>/*`) dumps include QPL page lists, admin-q ring pointers; non-root reads scrub.
- **GRKERNSEC_DMESG** — gVNIC log lines that reveal queue counts / per-VM topology restricted from non-root.
- **CAP_NET_ADMIN** required on ethtool ops that change MAC/RSS/ntuple/coalesce; flow-rule install; XDP-prog install; ptp-clock adjust.
- **Andromeda-host-as-adversary explicit** — like ENA, gVNIC's trust boundary is the GCE hypervisor stack (Andromeda). Admin CQ entries, device_options TLV walks, stats-report DMA writes from the NIC, and completion descriptors are all validated against expected formats and ranges before consumption.
- **device_options TLV validation strict** — TLV walker has fixed allowlist of option opcodes; unknown opcodes are skipped (with rate-limited warn). Closes a path where Andromeda could push a malicious option that triggers driver-side parsing bug.
- **Admin opcode allowlist** — only the known set of admin opcodes (`GVE_ADMINQ_DESCRIBE_DEVICE` through `GVE_ADMINQ_QUERY_FLOW_RULES` etc.) are accepted on CQ; unknown opcode-status combos rejected.
- **QPL slot validation in fast path** — every TX/RX descriptor's QPL slot index validated against the QPL page count before page-pointer dereference. Closes "compromised Andromeda forges slot index" → UAF/OOB-write class.
- **DQO completion descriptor validation** — completion's `buf_id` validated against the per-ring `buf_state` array bounds; mismatches trigger reset rather than UAF.
- **Reset rate-limit** — 3 resets / 60s like ENA; closes infinite-reset-loop class.
- **Stats-report DMA bounded** — the stats-report buffer is allocated once at probe with a fixed size; the NIC cannot induce growth. Periodic re-init also re-zeroes the buffer.
- **Flow-rule install gated by cap** — driver refuses to install more rules than NIC advertised in `flow_rule_max_count`; closes a DoS path where an unprivileged container with NET_ADMIN scoped to the netdev (containers + cgroups) could OOM the admin q with rule installs.
- **MSI-X vector pinning** — IRQ affinity set strictly; an attempt to migrate the management vector off CPU 0 is rejected (management work depends on a stable CPU for the watchdog).
- **PTP clock-adjust CAP_SYS_TIME** — PHC adjust ops require CAP_SYS_TIME, not just CAP_NET_ADMIN.
- **Debugfs CAP_SYS_ADMIN** — reset-trigger, ring-state-dump, admin-q-replay debugfs entries all require CAP_SYS_ADMIN; reads scrub pointers.
- **XDP-prog install in root-userns only** — XDP attach via netlink requires the calling process to be in init_user_ns or have a CAP_NET_ADMIN over the parent. Closes a container-escape path.

Per-doc rationale: gVNIC is the cloud-perimeter NIC for GCE, mirror-image to ENA's role for AWS. The same Andromeda-as-adversary trust model applies: the in-VM driver cannot trust the hypervisor-side firmware, so all admin CQ entries, device options, completion descriptors, and stats-report writes must be validated against the bounds the driver itself established. The two-mode (GQI vs DQO) architecture is a Rust translation strength — an `enum QueueMode { Gqi(GqiQueue), Dqo(DqoQueue) }` per-queue with exhaustive NAPI-poll dispatch closes the bug class where a queue created in DQO mode is poll'd via the GQI handler at runtime (or vice versa). The QPL slot-bounds check in the fast path is the structural defense against the historical "compromised hypervisor forges descriptor" attack class.

## Open Questions

- [ ] Q1: GQI vs DQO default — upstream defaults to DQO when NIC advertises support. Rookery should match. Edge case: some early-c3 firmware versions had DQO bugs; should the driver have a per-firmware-version override?
- [ ] Q2: Header-data split pool sizing — upstream defaults to `GVE_DEFAULT_HEADER_BUFFER_SIZE = 256`. The right size depends on workload (TCP vs UDP vs encrypted). Should this be devlink-parametric?
- [ ] Q3: XDP TX-queue count — upstream allocates per-CPU XDP TX queues at attach time. For high-CPU instances (`c3-standard-176`), that's 176 extra queues. NIC may have a queue-count cap; need clear behavior when XDP attach requests exceed cap.
- [ ] Q4: GVE_DEV_OPT_RSC (Receive-Side Coalescing) — newer NIC firmware may advertise RSC for very-large-receive coalescing. Upstream support is partial. Plan?
- [ ] Q5: Multi-NIC bond — like ENA, gVNIC sits one-per-VM typically; multi-NIC bond is uncommon. Defer to software bond.

## Verification

- **Kani SAFETY**: prove QPL slot validation cannot be bypassed by any TX/RX descriptor produced by the parsing layer. Prove DQO completion `buf_id` lookup is in bounds for every accepted completion.
- **TLA+**: model the reset state machine with concurrent admin-q timeout, watchdog tick, and netdev ifdown. Check that the resulting state is always one of {Online, Resetting-then-Online, PermanentlyFailed}; no leaked queues or dangling admin q.
- **Verus**: functional spec of the `device_options` TLV parser — for any TLV stream with valid length fields, returns the parsed options list; for malformed input, returns Err with a specific error code (never panics).
- **Kani+Verus**: invariant that every TX descriptor in flight has a corresponding sk_buff tracking entry; every TX completion clears the tracking entry; no leaks.
- **Integration**: GCE matrix smoke — Rookery boots on `n2-standard-2`, `c3-standard-22`, `c3-standard-176`, `a3-highgpu-8g`; gVNIC reaches line rate per instance type. Hot-add NIC (via GCE add-secondary-NIC). Reset injection. Flow rule install/remove stress (1000 rules).
- **Fuzz**: admin CQ fuzzer (mutate status, length, payload); should never produce oops. Device-options TLV fuzzer (mutate opcode/length/value); parser robustness.
- **Penetration**: in-VM unprivileged container attempts ethtool/devlink/XDP-attach against gVNIC — all CAP_NET_ADMIN-gated; container with CAP_NET_ADMIN scoped to a different netdev cannot affect gVNIC.

## Out of Scope

- GCE Andromeda host-side stack — outside Linux kernel scope
- GCE-specific virtio_net legacy path — covered separately under `drivers/net/virtio_net.md`
- GCE TPU/GPU accelerator integration — separate accelerator-driver docs
- AWS ENA (sibling cloud NIC, separate vendor) — covered in `ena.md`
- Azure mana (Microsoft cloud NIC) — future iteration
- Multi-NIC bond setups — software, covered in `net/bonding/` Tier-3
