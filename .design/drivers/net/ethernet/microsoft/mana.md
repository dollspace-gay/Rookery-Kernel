# Tier-3: drivers/net/ethernet/microsoft/mana/{gdma_main,hw_channel,shm_channel,mana_en,mana_bpf,mana_ethtool}.c — Microsoft Azure Network Adapter (MANA, every modern Azure VM with accelerated networking)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/microsoft/mana/gdma_main.c            (~2330 lines: GDMA layer — admin queue, EQ/CQ alloc, MSI-X, BAR0 regs, MANA-card recovery)
  - drivers/net/ethernet/microsoft/mana/hw_channel.c           (~930 lines: HW channel — GDMA messaging protocol with the MANA card)
  - drivers/net/ethernet/microsoft/mana/shm_channel.c          (~290 lines: shared-memory channel — bootstrap protocol before HW channel up)
  - drivers/net/ethernet/microsoft/mana/mana_en.c              (~3875 lines: netdev ops, NAPI, RX/TX, RSS, multi-vport switching, hwc msg handlers)
  - drivers/net/ethernet/microsoft/mana/mana_bpf.c             (~270 lines: XDP_DRV native + XDP_REDIRECT)
  - drivers/net/ethernet/microsoft/mana/mana_ethtool.c         (~595 lines: ethtool ops — stats, ring, RSS, link, coalesce, fec)
  - drivers/net/ethernet/microsoft/mana/Makefile
  - include/net/mana/mana.h
  - include/net/mana/hw_channel.h
  - include/net/mana/shm_channel.h
  - include/net/mana/gdma.h
  - include/net/mana/mana_auxiliary.h                         (auxiliary-bus interface for mana_ib RDMA driver)
  - drivers/infiniband/hw/mana/                                (separate RDMA driver that sits on top — out of scope for this Tier-3)
-->

## Summary

MANA (Microsoft Azure Network Adapter) is the NIC every Azure VM with accelerated networking — every modern HC/HB/Dadsv5/Fasv6/Easv6/M generation — sees. Like ENA (AWS) and gVNIC (GCP), it is a custom paravirt NIC presented by the Azure host-side stack (the SmartNIC + AccelNet host VFP datapath). It is *not* mlx5 (despite Azure using ConnectX hardware underneath), and it is *not* virtio-net. MANA is a clean-slate architecture introduced by Microsoft in 2022 to replace the legacy Hyper-V NetVSC + SR-IOV-VF combination — one driver, one ABI, scales to 200 Gbps per ENI on the largest instances.

MANA has a layered architecture distinct from ENA/gVNIC:

- **`shm_channel`** (~290 lines) — bootstrap shared-memory channel used **only at probe time** to negotiate the HW-channel parameters. After HW channel up, shm_channel is unused.
- **`gdma`** (Generic DMA, ~2330 lines) — the bus-agnostic device-communication layer. Owns admin queue, MSI-X, EQ (event queue) / CQ (completion queue) / SQ (submission queue) primitives. GDMA is shared across MANA sub-functions: networking (MANA-EN), RDMA (MANA-IB), and storage/other future sub-functions auxiliary-bus-attach to GDMA.
- **`hw_channel`** (~930 lines) — the messaging protocol over GDMA. Driver-side primitives for sending typed messages to the MANA card; used by MANA-EN for configuration commands.
- **`mana_en`** (~3875 lines) — the Linux netdev wrapper. Per-vport (Azure VFP virtual port) netdev, NAPI per queue, RSS, multi-vport switching (an Azure NIC can present multiple netdevs to the same VM for multi-NIC topologies), MTU mgmt.
- **`mana_bpf`** — XDP.
- **`mana_ethtool`** — ethtool ops.

A separate driver `drivers/infiniband/hw/mana/` (mana_ib) provides RDMA on top of GDMA; it auxiliary-bus-attaches to the same physical MANA function. This Tier-3 covers only the netdev path; mana_ib gets its own Tier-3.

This Tier-3 covers ~8.3K lines of upstream MANA driver.

## Rust translation posture

MANA's layering is a clean fit for Rust:

- `Gdma` is the device HAL — `Arc<Gdma>` shared between MANA-EN and (future) MANA-IB aux drivers.
- `Gdma<Phase>` typed: `Probed → ShmChannelUp → HwChannelUp → Online`.
- ShmChannel is a one-shot bootstrap that consumes itself into the HwChannel handle on success.
- HW channel messages → typed enum: `enum HwcMsg { CreateGdmaQueue { ... }, DestroyGdmaQueue { ... }, CreateMrr { ... }, ... }` with bilge-style bit-packed serialization.
- GDMA EQ/CQ/SQ → typed handles owning their DMA regions; `Eq`, `Cq`, `Sq` with phase markers for created-but-not-bound vs bound to-a-queue.
- MANA-EN per-vport state → `ManaVport` with owned RxQueue + TxQueue list, RSS indir, MTU; `Arc<ManaPriv>` per netdev.
- Auxiliary-bus per-sub-function → typed `AuxDev<{ Eth, Ib }>` distinguishable at compile time; `mana_en` and `mana_ib` are independent drivers that match different aux device names.
- Recovery (`mana_dev_recovery_work`) — typed work item dispatched from the EQ EQE-type `GDMA_EQE_HWC_FPGA_RECONFIG` event; sequence: quiesce → tear down → re-init shm channel → re-init hw channel → restore vports.

The grsec/PaX section is mandatory: MANA's trust boundary is **Azure-SmartNIC + AccelNet-host-as-adversary**, parallel to ENA's Nitro and gVNIC's Andromeda. Same posture.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gdma_context` | top-level GDMA state (regs, EQs, IRQs, devs, ...) | `Gdma<Phase>` |
| `struct gdma_dev` | per-sub-function device (gdma + mana + mana_ib) | `GdmaSubDev` |
| `struct shm_channel` | bootstrap shm channel state | `ShmChannel` (single-use) |
| `struct hw_channel_context` | post-bootstrap HW channel | `HwChannel` |
| `struct gdma_eq` / `_cq` / `_sq` | event/completion/submission queue | `Eq` / `Cq` / `Sq` |
| `struct mana_port_context` | per-vport netdev state | `ManaVport` |
| `mana_gd_probe(pdev, ent)` / `_remove(pdev)` | PCI probe / remove | `Gdma::probe` / `Drop` |
| `mana_gd_setup(pdev, gc)` | full bring-up after PCI map | `Gdma::setup` |
| `mana_gd_init_registers(pdev)` | BAR0 register init | `Gdma::init_regs` |
| `mana_gd_alloc_msix(gc)` / `_free_msix(...)` | MSI-X allocate/free | `Gdma::alloc_msix` |
| `mana_gd_create_eq(gc, queue_spec, &queue)` / `_destroy_eq(...)` | EQ create/destroy | `Eq::create` / `Drop` |
| `mana_gd_create_cq(...)` / `_destroy_cq(...)` | CQ create/destroy | `Cq::create` / `Drop` |
| `mana_gd_create_sq(...)` / `_destroy_sq(...)` | SQ create/destroy | `Sq::create` / `Drop` |
| `mana_gd_create_hwc_queues(gc)` | bootstrap HW-channel queues | `HwChannel::create_queues` |
| `mana_hwc_create_channel(gc)` / `_destroy_channel(...)` | HW channel init/destroy | `HwChannel::new` / `Drop` |
| `mana_hwc_send_request(hwc, req_buf, req_len, resp_buf, resp_len)` | sync HW msg send | `HwChannel::send_sync` |
| `mana_smc_setup(sc, ...)` / `mana_smc_handshake_send(...)` | shm-channel bootstrap | `ShmChannel::handshake` |
| `mana_create_eth_dev(gc, gd)` / `_destroy_eth_dev(...)` | MANA-EN sub-device create | `ManaEnSubDev::create` |
| `mana_probe_port(ac, port_idx)` / `_remove_port(...)` | per-vport probe | `ManaVport::probe` |
| `mana_open(net_dev)` / `mana_close(...)` | netdev open/close | `ManaVport::open` |
| `mana_start_xmit(skb, ndev)` | TX entry | `ManaVport::start_xmit` |
| `mana_poll(napi, budget)` / `mana_poll_rx_cq(cq)` / `_poll_tx_cq(...)` | NAPI poll | `Napi::poll` |
| `mana_query_device_cfg(...)` / `_query_vport_cfg(...)` | per-vport configuration query | `HwChannel::query_*` |
| `mana_cfg_vport(...)` | configure vport | `HwChannel::cfg_vport` |
| `mana_create_wq_obj(...)` / `_destroy_wq_obj(...)` | per-queue work-queue obj | `WqObj::create` |
| `mana_cfg_rss(...)` / `_cfg_rss_steer_entry(...)` | RSS config | `HwChannel::set_rss` |
| `mana_register_filter(...)` / `_deregister_filter(...)` | MAC filter | `HwChannel::set_mac_filter` |
| `mana_bpf_setup(...)` | XDP attach/detach | `ManaVport::xdp_setup` |
| `mana_xdp_xmit(ndev, n, frames, flags)` | XDP_REDIRECT TX | `ManaVport::xdp_xmit` |
| `mana_dev_recovery_workfn(work)` | recovery work handler | `Recovery::handle` |
| `mana_gd_install_handlers(gc)` | EQE handler table install | `Gdma::install_eqe_handlers` |
| `mana_gd_dispatch_eqe(eq, eqe)` | EQE dispatch | `Eq::dispatch` |

## Compatibility contract

REQ-1: PCI ID table — matches `0x1414:0x00ba` (MS MANA). Frozen.

REQ-2: Two-BAR layout — BAR0 = registers, BAR2 = doorbells. Mapped at probe.

REQ-3: Shm-channel bootstrap — before HW-channel init, driver and MANA card exchange a small handshake through a 4KB shared memory page. After handshake, the shm-channel is unused for the lifetime of the device.

REQ-4: HW-channel protocol — bidirectional typed message queue layered on GDMA SQ/CQ. Each request has a 32-bit `request_id` allocated by driver; response carries matching id. Used by MANA-EN for all configuration (vport create, RSS set, MAC filter, etc.).

REQ-5: EQ/CQ/SQ resource — GDMA exposes generic Event Queue / Completion Queue / Submission Queue primitives. Each EQ has an MSI-X vector. CQs sit on EQs (multiple CQs per EQ). SQs feed CQs.

REQ-6: Per-vport netdev — one MANA function can present multiple vports (each becomes its own netdev). Multi-vport configurations used for Azure scenarios like Direct Server Return load-balancers.

REQ-7: MAC filter registration — driver issues `MANA_REGISTER_FILTER` HWC msg with up to N MAC filters per vport. Multicast/promisc through configuration.

REQ-8: RSS — 128-entry indir + 40B Toeplitz key (similar to ENA/mlx5). Configurable via ethtool.

REQ-9: TX/RX queues — per-queue NAPI; each queue pair has SQ (TX), CQ (TX completion + RX completion combined or separate per HW config), RQ buffer pool. RX uses page-pool for buffer allocation.

REQ-10: Recovery — host can trigger a card-side reconfiguration (e.g. host upgrade). Driver receives a `GDMA_EQE_HWC_FPGA_RECONFIG`-class EQE on the EQ; the recovery work handler quiesces all sub-functions, tears down, re-bootstraps via shm-channel, brings everything back up.

REQ-11: Aux-bus integration — MANA card may expose multiple sub-functions (Ethernet, RDMA, future); each is registered on the aux-bus with a distinct name (`mana.eth`, `mana_ib`). Sub-drivers match on name.

REQ-12: Auxiliary-bus interface (`mana_auxiliary.h`) — typed ABI between GDMA and sub-drivers (mana_en, mana_ib). Frozen.

REQ-13: XDP — full XDP_DRV including XDP_REDIRECT; dedicated XDP TX queues at attach.

REQ-14: PCIe AER — `pci_error_handlers` registered; on AER event, recovery work fires.

## Acceptance Criteria

- [ ] AC-1: `lspci` on a `D32ds_v5` Azure VM shows `1414:00ba` MANA; Rookery probes; `dmesg | grep mana` matches upstream log lines.
- [ ] AC-2: `ip link` shows `eth0` UP; gateway reachable.
- [ ] AC-3: `iperf3` between two `Standard_HC44-32rs` (200 Gbps) Azure VMs sustains >150 Gbps multi-flow; single-flow gets line rate with RSS-spread.
- [ ] AC-4: Multi-vport: an Azure VM with multiple ENIs of the MANA flavor presents multiple netdevs; each gets its own MAC; routing works.
- [ ] AC-5: RSS: `ethtool -X eth0 ...` updates indirection table; per-CPU softirq spread observable.
- [ ] AC-6: XDP smoke: `xdp-loader load eth0 xdp_pass.o` succeeds.
- [ ] AC-7: Recovery: trigger a `GDMA_EQE_HWC_FPGA_RECONFIG` from debugfs; driver quiesces, re-bootstraps shm-channel, re-creates queues; traffic resumes on all vports.
- [ ] AC-8: aux-bus: with mana_ib also loaded, both `mana.eth` (netdev) and `mana_ib` (rdma device) bind to the same MANA function; both work concurrently.
- [ ] AC-9: Hot-add NIC (Azure secondary ENI): rescan → new ethN appears.
- [ ] AC-10: PCIe AER: simulated; recovery completes.

## Architecture

**Three-layer driver stack.** `shm_channel` → `gdma` → `mana_en`. The shm_channel is a one-shot bootstrap — a 4KB page exchanged at probe to negotiate the HW channel parameters. Once the HW channel is up, all subsequent driver↔card communication goes through GDMA's typed message queue. This is structurally cleaner than ENA's admin-queue-everything model: GDMA admin q is for queue resource management; HW channel is for per-vport configuration; the netdev fast path is its own SQ/CQ that doesn't go through admin q at all.

**GDMA as shared HAL.** GDMA is bus-agnostic and sub-driver-agnostic. mana_en, mana_ib, and any future MANA sub-driver attach via the aux-bus. They share the same `gdma_context`, the same MSI-X vector pool, the same EQ/CQ/SQ primitives. Sub-drivers issue typed HWC messages for their configuration; the message contents are sub-driver-specific.

**EQ/CQ/SQ generality.** EQs receive events from the card (queue-state changes, errors, recovery requests). CQs receive per-message completions for a bound SQ. SQs are the submission ring (TX, admin, HWC). All three are DMA-mapped, MMIO-doorbell-rung. The layering means MANA-IB and MANA-EN both create their own SQ/CQ/EQ instances through the same GDMA primitives — no sub-driver-specific resource primitives.

**Per-vport state.** Each vport gets its own netdev. The card's vport metadata (vport_idx, MAC, max queues, MTU) is queried at probe via `MANA_QUERY_VPORT_CFG` HWC msg. The driver creates queues for the vport (via `MANA_CREATE_WQ_OBJ` etc.), installs filters, sets RSS, registers the netdev. Multi-vport configurations: per-NIC two or more vports → two or more netdevs in the VM; common for load-balancer setups where one vport handles ingress and another egress.

**Recovery work.** Triggered by an EQE with type `GDMA_EQE_HWC_FPGA_RECONFIG` (or similar reconfig event). Recovery handler:
1. quiesce — stop NAPI, stop ifqueues
2. tear down — destroy SQ/CQ/EQ/vport state via HWC msgs
3. tear down GDMA — destroy admin q, free MSI-X
4. re-bootstrap — re-run shm-channel handshake, re-init GDMA, re-init HWC
5. restore — re-query vport caps, re-create queues, restore RSS, restore filters
6. resume — start NAPI, start ifqueues

Sticky stats counters survive recovery; netdev does not bounce (carrier may briefly drop).

**Aux-bus.** GDMA registers one aux-device per sub-function. `mana.eth` matches the MANA-EN driver; `mana_ib` matches the MANA-IB driver (separate kmod upstream). Rookery integrates both as part of the same binary but uses the aux-bus pattern for clean separation; mana_ib has its own Tier-3.

**Locking.** GDMA admin q serialized by mutex (one cmd at a time). HWC SQ separately serialized (HWC messages can be longer than admin cmds and shouldn't block other config ops). Per-vport state-lock for vport-lifecycle ops. NAPI per-queue uses standard NAPI synchronization.

## Hardening

- All HWC messages have a 30s timeout (longer than admin because config msgs can be slow); past timeout, recovery work fires.
- shm-channel handshake validates the response's magic + version before consuming any fields; mismatched response aborts probe cleanly.
- HWC response `request_id` validated against in-flight set; unknown responses dropped.
- EQE handler table indexed by `eqe_type`; out-of-range types dropped with warn.
- Recovery rate-limited: 3 within 60s, then permanent down.
- All HWC msg payload lengths validated before consuming.
- Per-vport queue counts capped by card-advertised `max_num_queues_per_vport`.
- AER handlers issue recovery work rather than per-handler ad-hoc reset.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool / RSS / MAC-filter ingress via bounded `copy_from_user`.
- **PAX_KERNEXEC** — `mana_netdev_ops`, `mana_ethtool_ops`, GDMA EQE handler table, HWC msg dispatch table, aux-bus match table, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / netlink / sysfs entry paths are per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `gdma_context` refcount, per-vport refcount, per-EQ/CQ/SQ refcount, aux-device refcount use saturating refcounts; underflow at remove is a hard panic.
- **PAX_MEMORY_SANITIZE** — shm-channel page zeroed before free (carries handshake secrets). HWC msg DMA buffers `kfree_sensitive` (carry RSS keys, MAC filters). EQ/CQ rings zeroed on destroy.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — `mana_netdev_ops`, `mana_ethtool_ops`, GDMA EQE handlers, HWC msg dispatch, aux-bus probe/remove callbacks all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/mana/<bdf>/*`) scrubbed for non-root.
- **GRKERNSEC_DMESG** — multi-vport topology, per-vport queue count, recovery events restricted from non-root dmesg.
- **CAP_NET_ADMIN** required on ethtool / aux-bus / XDP-attach ops.
- **Azure SmartNIC + AccelNet-as-adversary explicit** — like ENA's Nitro model and gVNIC's Andromeda model, MANA's trust boundary is the host-side accelerated-networking stack. EQE payloads, HWC msg responses, shm-channel handshake values, and per-queue completion descriptors are validated against expected formats and ranges before consumption.
- **shm-channel response validation strict** — magic + version + length fields all validated; any deviation aborts probe rather than parsing further. The shm-channel is the only attacker-controlled surface before MSI-X is wired, so it's the trickiest validation path; bounded parser is critical.
- **HWC msg opcode allowlist** — only the known MANA-EN opcodes (`MANA_QUERY_DEV_CONFIG` through `MANA_DEREGISTER_FILTER` etc.) processed; unknown opcodes dropped with rate-limited warn.
- **EQE-type allowlist** — only known EQE types dispatched; out-of-range or unknown types dropped (compromised card cannot trigger handler-table OOB).
- **Recovery rate-limit + bounded** — 3 recoveries in 60s → permanent-down. Closes the "Azure host keeps requesting reconfig until we OOM" class.
- **Per-vport queue count cap** — driver refuses to create more queues per-vport than card-advertised `max_num_queues_per_vport`. Closes a host-induced resource-exhaustion path.
- **Aux-bus name allowlist** — only `mana.eth` and `mana_ib` aux-driver matches accepted; foreign aux-driver registration on the MANA aux-bus refused. Closes "third-party kmod shadows mana_en" path.
- **HWC request_id allocator** — request_ids drawn from a bounded namespace; allocator wraps with check that the wrapped id is not in flight. Prevents a slow HWC msg from getting confused with a newer one if request_id space exhausts (closes a UAF-class via completion mismatch).
- **MAC filter count cap** — per-vport MAC filter installs capped at the card-advertised limit; closes a "fill the MAC filter table to make every MAC promisc-or-dropped" exhaustion path.
- **Debugfs CAP_SYS_ADMIN** — debugfs writes (recovery-trigger, queue-state-dump, HWC-msg-replay) all require CAP_SYS_ADMIN; reads scrub pointers.
- **MSI-X management vector pinned** — like gVNIC; the GDMA management EQ vector is affinity-pinned and refuse-migration.
- **Multi-vport tenant isolation** — when multiple vports are on the same VM, they're typically managed by the same userspace; but if a container with CAP_NET_ADMIN scoped to one vport (via cgroup) attempts to modify another vport, refuse.

Per-doc rationale: MANA completes the cloud-perimeter NIC trio (ENA/AWS, gVNIC/GCP, MANA/Azure). The trust model is identical across the three: hypervisor-side NIC firmware is adversarial; the in-VM driver must defensively validate every byte that comes out of admin/HWC/EQ/shm channels. MANA's three-layer architecture (shm-channel → GDMA → HWC) requires careful sequencing — the shm-channel handshake is the only attacker-controlled surface before MSI-X is set up, so its parser must be bulletproof and bounded. The aux-bus structure for sub-drivers (mana_en, mana_ib sharing a GDMA instance) is a Rust-translation strength: typed `AuxDev<Eth>` vs `AuxDev<Ib>` distinguish the sub-driver namespaces and an unauthorized aux-driver attempting to bind is rejected by the type system.

## Open Questions

- [ ] Q1: Should Rookery integrate mana_en + mana_ib as one binary (sharing the GDMA instance directly) or preserve the aux-bus separation? Aux-bus is cleaner architecturally but adds runtime dispatch. Recommendation: preserve aux-bus pattern — the structural separation pays off for security and for future sub-driver additions.
- [ ] Q2: Shm-channel bootstrap — only used once at probe. Rookery could make `ShmChannel` a single-use type that consumes itself into `HwChannel` on success, eliminating any post-bootstrap surface entirely. Recommended.
- [ ] Q3: Multi-vport on a single MANA function — common in Azure but uncommon in ENA/gVNIC. Rookery's per-vport ManaVport design needs to handle the case where one vport bounces (carrier drop) without affecting siblings on the same function.
- [ ] Q4: GDMA EQ event-type expansion — newer MANA firmware may add new EQE types. The allowlist approach (drop unknown) is safe but means new event types need driver updates. Alternative: ignore-with-warn unknown types. Pick conservative.
- [ ] Q5: Hot mana_ib remove while mana_en is active — both share GDMA. The aux-bus framework handles this, but the typed Rust translation needs explicit ordering on the GDMA refcount.

## Verification

- **Kani SAFETY**: prove ShmChannel cannot be used after consumption (typestate: ShmChannel::handshake consumes self and returns HwChannel; any further use is a compile error).
- **TLA+**: model the recovery state machine. Concurrent recovery request + new HWC msg from the netdev path: messages either complete or fail with EBUSY, never dangle. After recovery, all vports return to pre-recovery state or transition to PermanentlyFailed cleanly.
- **Verus**: functional spec of the HWC msg sync send: given a serializable request, returns the matching response (request_id round-trip) or a defined error code; never blocks indefinitely (bounded by timeout).
- **Kani+Verus**: invariant that the in-flight HWC request set is always equal to the request_id allocator's busy bitmap; no leaked request_ids.
- **Integration**: Azure matrix smoke — Rookery on `D2ds_v5`, `D32ds_v5`, `HC44-32rs` (200G), `HBv4-176rs`; verify line rate per type. Multi-vport test on a VM with two MANA ENIs. mana_en + mana_ib concurrent traffic. Recovery injection. AER injection.
- **Fuzz**: HWC response fuzzer (mutate request_id/status/length/payload); shm-channel response fuzzer (mutate handshake bytes); EQE fuzzer (mutate type/sub_type/payload). All must not panic.
- **Penetration**: in-VM unprivileged container attempts ethtool/ XDP-attach against MANA — CAP_NET_ADMIN-gated; container scoped to vport0 cannot affect vport1.

## Out of Scope

- mana_ib (RDMA driver on same MANA function) — covered in `drivers/infiniband/microsoft-mana-ib.md` (future iteration)
- Azure SmartNIC + AccelNet host-side stack — outside Linux kernel scope
- Hyper-V NetVSC + legacy SR-IOV-VF combination — separate older Tier-3 (`drivers/net/hyperv/`)
- Azure HPC/SR (Scalable Reliable transport for HPC) — separate Tier-3 once support upstreams
- Azure FPGA-direct accelerator paths — separate accelerator-driver doc
- AWS ENA / GCP gVNIC — sibling cloud NICs, covered separately
