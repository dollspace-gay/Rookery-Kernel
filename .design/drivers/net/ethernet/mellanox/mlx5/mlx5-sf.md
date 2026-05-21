# Tier-3: drivers/net/ethernet/mellanox/mlx5/core/sf/{devlink,hw_table,vhca_event,cmd}.c — Mellanox mlx5 Sub-Functions (vport-without-PCI-BDF, scalable representor for k8s/cloud-native)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
depends-on:
  - drivers/net/ethernet/mellanox/mlx5/mlx5-core.md
  - drivers/net/ethernet/mellanox/mlx5/eswitch.md
upstream-paths:
  - drivers/net/ethernet/mellanox/mlx5/core/sf/devlink.c       (~555 lines: devlink-port-new/del, port-fn-state, SF lifecycle)
  - drivers/net/ethernet/mellanox/mlx5/core/sf/hw_table.c      (~410 lines: per-controller SF id allocator, sw↔hw fn-id mapping)
  - drivers/net/ethernet/mellanox/mlx5/core/sf/vhca_event.c    (~235 lines: VHCA-state async event handler, IN_USE/TEARDOWN signaling)
  - drivers/net/ethernet/mellanox/mlx5/core/sf/cmd.c           (~50 lines: thin wrappers around ALLOC_SF / DEALLOC_SF / ENABLE_HCA(SF) FW cmds)
  - drivers/net/ethernet/mellanox/mlx5/core/sf/sf.h
  - drivers/net/ethernet/mellanox/mlx5/core/sf/priv.h
  - drivers/net/ethernet/mellanox/mlx5/core/sf/vhca_event.h
  - drivers/net/ethernet/mellanox/mlx5/core/sf/mlx5_ifc_vhca_event.h
  - drivers/net/ethernet/mellanox/mlx5/core/sf/dev/                            (per-SF auxiliary bus device)
  - drivers/net/ethernet/mellanox/mlx5/core/lib/sf.h
  - drivers/net/ethernet/mellanox/mlx5/core/eswitch.c                          (esw integration: SF-controlled vports)
  - include/net/devlink.h                                                       (devlink_port_new_attrs, port_fn_state, devlink_port_function ops)
-->

## Summary

mlx5 Sub-Functions (SFs) are a vport-without-a-PCI-BDF: a Mellanox/NVIDIA ConnectX function that behaves like an SR-IOV VF (own QPs, own SQs/RQs, own representor on the PF, own MAC, own VLAN, own QoS, own ACL) but does **not** consume a PCIe Bus/Device/Function slot. That distinction matters because PCIe BDF space is finite — a host can have 256 functions per bus — and modern cloud-native deployments (k8s pod-per-NIC, Cilium per-endpoint NIC, OVN per-tenant) want tens of thousands of isolated network functions on a single NIC. SR-IOV maxes out around 256 VFs per PF; SFs scale into the thousands per PF on ConnectX-6/7/8 (BlueField-3 supports 4096+).

Mechanically: an SF is a `mlx5_vport` allocated through `MLX5_CMD_OP_ALLOC_SF` against a parent PF/ECPF, identified by `(controller, sfnum)` from the user side and by `hw_fn_id` (the FW-assigned 16-bit vport id) on the device side. Each SF has a `mlx5_vhca_state` lifecycle (ALLOCATED → ACTIVE → IN_USE → TEARDOWN → INVALID) and an auxiliary-bus device that lets sub-drivers (mlx5e, mlx5_ib, mlx5_vdpa) bind onto it as if it were a real PCI function.

This Tier-3 covers `mlx5/core/sf/*.c` (~1250 lines core + the `sf/dev/` auxiliary-bus glue).

## Rust translation posture

SFs are an excellent fit for typed-state-machine translation: the `mlx5_vhca_state` byte that upstream uses (`MLX5_VHCA_STATE_ALLOCATED`, `_ACTIVE`, `_IN_USE`, `_TEARDOWN`, `_INVALID`) is a 5-state FSM with deterministic transitions driven by user commands and FW async events. Replace the byte with a phantom-typed `Sf<S: SfState>`:

- `Sf<Allocated>` — `MLX5_CMD_OP_ALLOC_SF` succeeded, `MLX5_CMD_OP_ENABLE_HCA(SF)` not yet issued; FW knows the SF exists but no resources committed.
- `Sf<Active>` — `ENABLE_HCA` done; SF can have an auxiliary device bound to it.
- `Sf<InUse>` — a sub-driver (mlx5e, mlx5_ib) has bound the aux device and is actively driving traffic; the FW will refuse a tear-down request in this state.
- `Sf<TearDown>` — `DEALLOC_SF` issued, waiting for FW to confirm all references released; transient.
- `Sf<Invalid>` — terminal pre-`Drop` state; ID returned to allocator.

Transitions use consuming methods:

```rust
impl Sf<Allocated> {
    pub fn enable_hca(self, hca_caps: HcaCaps) -> Result<Sf<Active>, (Sf<Allocated>, FwError)>;
}
impl Sf<Active> {
    pub fn bind_aux(self, drv: SfSubDriver) -> Sf<InUse>;
}
impl Sf<InUse> {
    pub fn request_teardown(self) -> Result<Sf<TearDown>, (Sf<InUse>, EswError)>;
}
```

The `(controller, sfnum) → hw_fn_id` mapping is owned by a typed allocator (`SfIdAllocator`) that wraps the per-controller `xarray`; the mapping never escapes to callers, who hold `SfHandle` references that resolve via the allocator.

VHCA async events become `tokio`-style messages dispatched on an `mpsc` channel; the per-event-type allowlist (only ALLOCATED→ACTIVE→IN_USE→TEARDOWN→INVALID transitions allowed; FW cannot push e.g. INVALID→ACTIVE) is enforced in the dispatcher.

The grsec/PaX section is mandatory: SF identifiers (`hw_fn_id`, `sfnum`) are tenant-visible names and must be randomized. The aux-bus device-add path is a cross-driver boundary that is historically a UAF bug source.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlx5_sf` | per-SF state (id, hw_fn_id, hw_state, controller, devlink_port) | `Sf<S>` (state-typed) |
| `struct mlx5_sf_table` | per-PF SF registry + state-lock | `SfTable` |
| `struct mlx5_sf_hw_table` / `mlx5_sf_hw` | SF id allocator (per-controller) | `SfIdAllocator` |
| `mlx5_devlink_sf_port_new(dl, attrs, extack, &dl_port)` | devlink port-new for SF | `Devlink::sf_port_new` |
| `mlx5_devlink_sf_port_del(dl, dl_port, extack)` | devlink port-del | `Devlink::sf_port_del` |
| `mlx5_devlink_sf_port_fn_state_get/_set` | port-fn-state get/set | `SfHandle::state` / `transition` |
| `mlx5_sf_alloc(table, esw, ctrl, sfnum, extack)` | allocate SW SF object + HW id | `SfTable::alloc` |
| `mlx5_sf_free(table, sf)` / `mlx5_sf_dealloc(...)` | tear down SW + HW | `Sf::Drop` / `SfTable::dealloc` |
| `mlx5_sf_hw_table_sf_alloc(dev, ctrl, sfnum)` / `_free` | per-controller HW id alloc/free | `SfIdAllocator::alloc` / `free` |
| `mlx5_sf_sw_to_hw_id(dev, ctrl, sw_id)` / `_hw_to_sw_id` | id translation | `SfIdAllocator::translate` |
| `mlx5_sf_cmd_alloc_sf(dev, ctrl, *hw_fn_id)` / `_dealloc_sf` | FW ALLOC_SF / DEALLOC_SF cmds | `FwCmd::alloc_sf` / `dealloc_sf` |
| `mlx5_cmd_alloc_obj(dev, ENABLE_HCA, ...)` for SF | enable HCA on SF | `FwCmd::enable_hca_sf` |
| `mlx5_vhca_event_init(dev)` / `_arm` | VHCA-state async-event listener | `VhcaEvtHandler::init` / `arm` |
| `mlx5_vhca_state_data` (struct) | event payload: function_id, new_state | `VhcaStateEvt` |
| `mlx5_vhca_event_notifier_register(dev, nb)` | hook for sub-drivers (esw + sf_table) | `VhcaEvtHandler::on_change` |
| `mlx5_sf_dev_add(dev, sf)` / `_remove` | aux-bus per-SF device add/remove | `AuxBus::add_sf` / `remove_sf` |
| `mlx5_esw_offloads_sf_vport_enable(esw, vport, sfnum, ctrl)` / `_disable` | esw side of SF enable | `Eswitch::enable_sf_vport` |
| `mlx5_esw_offloads_controller_valid(esw, ctrl)` | controller-id range check | `Eswitch::controller_valid` |
| `mlx5_esw_vport_to_devlink_port_index(dev, hw_fn_id)` | devlink-port index from hw_fn_id | `Vport::to_devlink_port` |

## Compatibility contract

REQ-1: Devlink port-new ABI — `devlink port add pci/0000:XX:XX.0 flavour pcisf pfnum 0 sfnum N` creates an SF with software-assigned sfnum N, returns a `devlink_port` that userspace can configure further. Same UAPI as upstream including the `controller` attribute for multi-controller (BlueField host vs DPU) setups.

REQ-2: Port-fn-state ABI — `devlink port function set pci/.../<idx> state active` transitions an `MLX5_VHCA_STATE_ALLOCATED` SF to `_ACTIVE` (issues `ENABLE_HCA`); `state inactive` does the reverse. Matches upstream `enum devlink_port_fn_state`.

REQ-3: VHCA-state event delivery — async event `MLX5_EVENT_TYPE_VHCA_STATE_CHANGE` is the truth-source for state transitions. Driver state always tracks FW state; on divergence, FW wins.

REQ-4: Per-controller SF id space — `(controller, sfnum)` is the user-side identity; `hw_fn_id` is internal. Multiple controllers (BlueField: host-controller-0 + DPU-controller-1) have independent sfnum spaces.

REQ-5: SF maximum count — driven by `MLX5_CAP_GEN.max_num_sf` and per-controller `max_num_sf_per_controller`. Allocator must refuse beyond these limits.

REQ-6: Aux-bus device — each SF in `_ACTIVE` state gets an auxiliary-bus device named `mlx5_core.sf.<port_index>`. Sub-drivers (mlx5e, mlx5_ib, mlx5_vdpa) match by `auxiliary_device_id`. On SF teardown, all bound aux-drivers are detached *before* `DEALLOC_SF` is issued.

REQ-7: Devlink-port-fn ops parity — `mlx5_devlink_port_fn_ops` exposes hw_addr_get/set, state_get/set, max_io_eqs_get/set, max_msix_get/set, opstate_get with the same semantics as upstream.

REQ-8: ECPF interaction — on BlueField in ECPF mode, the SF controller is the DPU-side; host-side SF requests are forwarded through the ECPF eswitch. Host driver sees only its own representor of each SF it owns.

REQ-9: SF representor in eSwitch — every active SF has an offloads-mode representor netdev on its parent PF, named `<pf>_sf<sfnum>`. TC rules on the representor mirror to the SF, exactly as for VF representors.

REQ-10: Hot tear-down rejected — `DEALLOC_SF` on an `IN_USE` SF returns `-EBUSY`; user must `state inactive` first, which drains traffic and detaches sub-drivers, before `port del`.

## Acceptance Criteria

- [ ] AC-1: `devlink port add pci/0000:XX:XX.0 flavour pcisf pfnum 0 sfnum 42` succeeds on a ConnectX-6 Dx; new SF with `state inactive`, opstate detached appears in `devlink port show`.
- [ ] AC-2: `devlink port function set pci/.../<idx> hw_addr aa:bb:cc:dd:ee:01 state active` → SF enters ACTIVE; aux-bus device `mlx5_core.sf.<idx>` created; mlx5_core_en auto-binds; new netdev `<pf>sf<sfnum>` appears.
- [ ] AC-3: SF representor on PF: `ip link show` shows `<pf>_sf42`; tc rule `tc filter add dev <pf>_sf42 ingress ...` HW-offloaded.
- [ ] AC-4: Scale: 1024 SFs allocated on a single PF (ConnectX-6 Dx supports up to 4096 per BlueField-3); all transition to ACTIVE; total memory footprint linear in SF count (not quadratic).
- [ ] AC-5: SF tear-down ordering: `devlink port function set ... state inactive` triggers aux-driver detach → vport drain → `DEALLOC_SF`; no leaked queues or pages.
- [ ] AC-6: Hot tear-down rejection: with traffic on an SF, `devlink port del` returns `-EBUSY` cleanly (no kernel splat, no half-deleted state).
- [ ] AC-7: VHCA event integrity: simulated FW VHCA_STATE_CHANGE for a nonexistent function_id is ignored (no UAF in event handler).
- [ ] AC-8: BlueField ECPF: SF created on DPU-controller-1, host sees it via ECPF representor; host-controller SFs invisible to DPU side and vice versa.
- [ ] AC-9: SF + mlx5_vdpa: aux-driver mlx5_vdpa binds to an active SF, exposes `/dev/vhost-vdpa-N`, QEMU guest uses it as vhost-vdpa-net; traffic flows end-to-end.

## Architecture

**State machine at compile time.** The SF lifecycle is a 5-state FSM. Upstream represents the state as a `u16 hw_state` field and dispatches by `switch (hw_state)`; Rookery promotes the state to a phantom type, so the compiler enforces that `enable_hca` can only be called on an `Sf<Allocated>` and `dealloc` only on `Sf<TearDown>`. The transitions are:

```
Allocated ──ENABLE_HCA──▶ Active ──aux_bind──▶ InUse
                                             └─aux_unbind──▶ Active
   ▲                       │
   │                       └─DISABLE_HCA──▶ Allocated
   │
   └─DEALLOC_SF──◀ TearDown ◀── ALLOC_SF failure rollback
                              ◀── async VHCA_STATE_CHANGE(INVALID)
```

`Allocated → TearDown` (without going through Active) handles rollback after a failed `ENABLE_HCA`; `Active → Allocated` is the user-issued `state inactive`; `InUse → TearDown` is rejected (must unbind first).

**ID allocator.** Per-controller `xarray` keyed on `hw_fn_id`. `mlx5_sf_hw_table_sf_alloc` is essentially `xa_alloc_range(...)`; freeing returns the id to the range. The sw_id↔hw_id translation is a simple linear formula (`hw_fn_id = base_fn_id(controller) + sw_id`); Rookery's `SfIdAllocator` keeps the formula but adds bounds checks on every lookup.

**VHCA event handler.** Registered on the async-EQ; receives `mlx5_vhca_state_data` events. The handler looks up the `Sf<S>` by `function_id` (which is `hw_fn_id`), validates the new_state against the allowed transitions from current S (rejecting impossible transitions with a `dev_warn`), and either advances the typed handle or signals a waiter. The reason for validation: a buggy or compromised firmware could in principle push e.g. `Allocated → InUse` skipping `Active`; Rookery refuses such events rather than corrupting state.

**Aux-bus integration.** When an SF transitions to ACTIVE, `mlx5_sf_dev_add` registers an auxiliary device with `id = port_index` and `name = "sf"`. Sub-drivers (mlx5_core_en.sf, mlx5_ib.sf, mlx5_vdpa.sf) match on this name. Detach is the inverse — `mlx5_sf_dev_remove` walks bound aux-drivers, calls their `.remove` callbacks, and only then transitions to `Allocated`. Rookery's aux-bus is a typed mailbox: the SF emits an `SfAuxEvent::{Add, Remove}` on a channel, the aux-bus actor consumes it and dispatches.

**Eswitch coupling.** SF vports are managed by the same eSwitch that manages VFs (see `eswitch.md`). When an SF transitions ACTIVE, `mlx5_esw_offloads_sf_vport_enable` installs the per-vport ingress/egress ACLs, registers the representor netdev, and adds the slow-path send-to-vport rule. Inactivate is the reverse. The state-lock here is the *eswitch* state-lock (`esw->state_lock`), held while the SF state-lock (`table->sf_state_lock`) is held — Rookery enforces this ordering as a `LockGroup` constraint.

**Allocation atomicity.** `mlx5_sf_alloc` has a 3-step setup: (1) alloc hw_id, (2) kzalloc the `mlx5_sf` struct, (3) insert into xarray. The error labels (`id_err`, `alloc_err`, `insert_err`) unwind correctly. Rookery uses RAII guards (`HwIdGuard`, `SfStructGuard`) that auto-release on drop unless explicitly `defuse()`d on the success path; this eliminates the goto-ladder bug class entirely.

## Hardening

- All devlink/devlink-port/devlink-port-function ops require `CAP_NET_ADMIN`; no unprivileged SF lifecycle.
- `controller` and `sfnum` validated against the per-PF capability (`max_num_sf_per_controller`) before any FW command.
- VHCA event handler validates `function_id` is in the allocator's range; out-of-range events logged + dropped.
- Aux-bus device add/remove serialized through the SF state-lock; concurrent transitions on the same SF impossible.
- `mlx5_sf_alloc` returns -EEXIST cleanly if `sfnum` collides; no double-allocation.
- All FW cmd outputs validated by syndrome before consuming; partial-success states do not advance the typed state.
- Per-SF resource counters (QPs, CQs, EQs, MR-entries) tracked; teardown asserts all zero before issuing DEALLOC_SF.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — devlink-attr ingress (controller, sfnum, hw_addr, state) goes through bounded nlattr_validate; aux-bus name/version strings copied through `strscpy_pad` equivalent with size cap.
- **PAX_KERNEXEC** — `mlx5_devlink_port_fn_ops` vtable, VHCA event-listener tables, aux-bus driver match tables are `const` after init, W^X memory.
- **PAX_RANDKSTACK** — devlink netlink entry points are per-syscall stack-rerandomized; the SF lifecycle ops inherit that protection.
- **PAX_REFCOUNT** — per-SF object refcount, per-aux-bus-device refcount, per-controller allocator-handle refcount use saturating refcounts; underflow on `dealloc` is a hard panic.
- **PAX_MEMORY_SANITIZE** — `mlx5_sf` struct cleared on `kfree`; per-SF context includes MAC address + assigned trust state which must not leak across sfnum reuse.
- **PAX_UDEREF** — SMAP/SMEP on devlink-netlink, aux-bus sysfs entry paths.
- **PAX_RAP / kCFI** — `mlx5_devlink_port_fn_ops`, VHCA event notifier chain, aux-bus driver `probe`/`remove` callbacks are kCFI-signed; an attacker substituting an aux-bus driver vtable trips the signature check.
- **GRKERNSEC_HIDESYM** — devlink-port-show output that surfaces internal pointers (devlink_port struct, mlx5_sf struct addresses in debugfs) scrubbed for non-root.
- **GRKERNSEC_DMESG** — SF allocation/state-change log lines restricted; in a multi-tenant cloud the existence and identity of sibling SFs is sensitive.
- **CAP_NET_ADMIN on every lifecycle op** — port-new/del, port-function-set (state, hw_addr, max_io_eqs, max_msix), VHCA event listener registration.
- **sfnum LATENT_ENTROPY randomization** — when user passes `sfnum auto`, allocate from a randomized starting offset (per-controller, reseeded periodically) instead of monotonic. Closes a tenant-fingerprinting channel (sibling tenants cannot infer their neighbor's sfnum by enumerating).
- **hw_fn_id non-enumerable** — `hw_fn_id` is internal; not exposed in devlink-port output (only port_index, sfnum, controller). An attacker who can read devlink-port output learns nothing about FW-side function-id layout.
- **Cap-derived count limit** — refuse to allocate the (N+1)th SF when N >= `MLX5_CAP_GEN.max_num_sf` per controller; closes an exhaustion-DoS class (orchestrator misbehaving and creating SFs until FW OOMs).
- **VHCA event payload validation** — async event payload's `function_id` MUST be a previously-allocated id known to the allocator; events for unknown ids are dropped with a rate-limited warn (could indicate FW bug, compromise, or stale event from a sibling driver instance).
- **State-transition allowlist enforced on event** — async event handler refuses to advance state on a transition that isn't in the allowed-edges set (e.g. Active→InUse only when aux-bind happened; if FW signals InUse without a bind, ignored).
- **Aux-driver allowlist** — only blessed aux-driver names (`mlx5_core.eth`, `mlx5_ib`, `mlx5_vdpa`) bind; foreign aux-driver registration on the SF bus refused. Closes a third-party-module-shadows-mlx5e attack.
- **Tear-down quiesce check** — before DEALLOC_SF, driver verifies vport TX/RX counters are stable for ≥1 polling interval (no in-flight traffic). Otherwise refuses with -EBUSY. Closes a use-after-free race where DEALLOC_SF races packet-in-flight.
- **Controller value bounded** — `controller` field validated against `MLX5_ESW.controller_id_valid` cap mask; arbitrary controller numbers refused (avoid a path where a host-side request targets a DPU-side controller it shouldn't).

Per-doc rationale: SFs are the modern scalable tenant-isolation mechanism on Mellanox NICs; deployments with thousands of SFs per PF are the design target. Bugs in the SF lifecycle (state machine, aux-bus, eswitch coupling) directly become tenant-isolation breaches at cloud scale. The state-typed Rust translation closes the largest historical bug class (calling-the-wrong-op-on-the-wrong-state); the grsec reinforcements close exhaustion-DoS, tenant-fingerprinting (sfnum randomization), and untrusted-FW-event (state-transition allowlist) classes. SF teardown is particularly delicate — DEALLOC_SF on an in-use SF was historically a UAF source (CVE in mlx5_ib teardown coupling), the quiesce check and aux-driver detach ordering are the mitigations.

## Open Questions

- [ ] Q1: SF migration — BlueField-3 supports live-migrating an SF (its full FW state) between hosts. Rookery's typed state machine needs an additional `Sf<Migrating>` state and a serialize-into-buffer path. Defer or design now?
- [ ] Q2: SF per-driver-instance vs per-controller — should the SF table be per-`MlxCoreDev` (current upstream) or per-controller (cleaner for ECPF)? Has implications for cross-controller SF ops.
- [ ] Q3: Aux-bus or direct sub-driver embedding — upstream uses the kernel's auxiliary_bus to let sub-drivers be separate modules. Rookery has one binary; could embed sub-drivers directly and skip aux-bus entirely, simplifying lifetimes. Trade-off is losing the "can be a separate kmod" property.
- [ ] Q4: SF + VFIO passthrough — vfio-pci-mlx5 can pass an SF to a guest. The SF representor stays on the host; the SF itself is the guest's NIC. Locking + state-machine interaction with VFIO needs careful design.
- [ ] Q5: SF-per-namespace — should an SF be assignable to a netns at creation time? Currently SFs are global; netns assignment happens via `ip link set netns`. Cleaner UX would be `devlink port add ... netns N`.

## Verification

- **Kani SAFETY**: prove the SF state machine is deadlock-free (every state has at least one outgoing transition unless terminal) and progress-preserving (no state can both accept and reject a given transition non-deterministically).
- **TLA+**: model the concurrent path where a user issues `state inactive` while a VHCA event for the same SF arrives. Both paths converge to `Allocated` or one wins cleanly; no state where the typed handle and FW disagree.
- **Verus**: functional spec of `SfIdAllocator::alloc` — given a valid (controller, sfnum), returns Ok(hw_fn_id) where hw_fn_id is in the per-controller range and is not currently allocated; or returns Err(-EEXIST) if sfnum is taken.
- **Kani+Verus**: invariant that the SfTable's count is always equal to `count(Sf in {Allocated, Active, InUse, TearDown})` — leak detection at design level.
- **Integration**: scale test — allocate, activate, and put traffic on 1024 SFs on a single PF; verify per-SF stats counters are independent and totals add up. Tear-down stress: 1000 SF create/activate/delete cycles back-to-back, verify no growth in FW resource counters or kernel slab usage.
- **Fuzz**: VHCA event payload fuzzer — mutate `function_id`, `new_state`, transition direction; verify the event handler never advances state out-of-spec.
- **Penetration**: with a tenant that owns SF[i], try every devlink/ethtool/tc/sysfs operation that names a different SF[j]; none should succeed without CAP_NET_ADMIN.

## Out of Scope

- mlx5_core firmware-cmd interface, EQ/CQ — covered in `mlx5-core.md`
- mlx5 eSwitch (the consumer of SF vports) — covered in `eswitch.md`
- mlx5e ethernet datapath (the typical aux-driver bound to an active SF) — covered in `mellanox-mlx5-en.md`
- mlx5_vdpa per-SF binding — covered in (future) `drivers/vdpa/mellanox-mlx5-vdpa.md`
- BlueField DPU host-driver specifics (ECPF + lib/hv_vhca) — future iteration
- VFIO-PCI-mlx5 SF passthrough internals — future iteration once Rookery's VFIO doc stack covers vfio-pci-vendor-driver pattern
