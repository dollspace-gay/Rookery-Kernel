# Tier-3: drivers/net/ethernet/mellanox/mlx5/core/{eswitch,eswitch_offloads,esw/*}.c — Mellanox mlx5 eSwitch (SR-IOV port representors + switchdev + TC HW-offload)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
depends-on: drivers/net/ethernet/mellanox/mlx5/mlx5-core.md
sibling: drivers/net/ethernet/mellanox-mlx5-en.md
upstream-paths:
  - drivers/net/ethernet/mellanox/mlx5/core/eswitch.c              (~2600 lines: vport/UC/MC/VLAN mgmt + legacy SR-IOV mode + ACL)
  - drivers/net/ethernet/mellanox/mlx5/core/eswitch.h              (vport, mode, repr, mapping types)
  - drivers/net/ethernet/mellanox/mlx5/core/eswitch_offloads.c     (~5100 lines: switchdev offloads, slow-path repr, FDB chains, miss-rule)
  - drivers/net/ethernet/mellanox/mlx5/core/eswitch_offloads_termtbl.c
  - drivers/net/ethernet/mellanox/mlx5/core/esw/legacy.c           (legacy mode: per-vport ACL tables)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/qos.c              (per-vport QoS scheduling: max-rate, min-rate, groups)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/devlink_port.c     (devlink-port registration per representor)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/bridge.c           (HW-offloaded Linux bridge — VLAN/MDB/FDB-learn)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/bridge_mcast.c
  - drivers/net/ethernet/mellanox/mlx5/core/esw/bridge_debugfs.c
  - drivers/net/ethernet/mellanox/mlx5/core/esw/vporttbl.c         (per-vport meta table cache)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/indir_table.c      (indirect-table dispatch for chained rules)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/acl/{egress_lgcy,egress_ofld,ingress_lgcy,ingress_ofld,helper}.c
  - drivers/net/ethernet/mellanox/mlx5/core/esw/ipsec.c            (per-representor IPsec offload)
  - drivers/net/ethernet/mellanox/mlx5/core/esw/diag/
  - include/linux/mlx5/eswitch.h
  - include/uapi/linux/devlink.h                                   (devlink-rate, devlink-port ABI used by eSwitch)
-->

## Summary

mlx5 eSwitch is the on-chip embedded switch that connects PF + VFs + SFs (Sub-Functions) + uplink port on a Mellanox/NVIDIA ConnectX. It is the structural feature that makes the NIC a tenant-isolation device rather than a simple Ethernet adapter: every VF or SF gets a representor netdev on the PF host, eSwitch policy decides which traffic flows where, and TC/devlink rules push that policy into hardware FDB chains so the datapath stays in silicon. This driver layer is what enables OpenStack/OVN/k8s-CNI/Calico/Cilium SR-IOV offload, OVS HW-offload (`tc-offload` for OVS bridges), and BlueField DPU host-isolation (DPU runs the eSwitch; host sees only what the DPU lets through).

Two modes coexist:

- **`legacy` mode** (`esw/legacy.c`, `esw/acl/*_lgcy.c`) — per-vport ACL tables enforce MAC/VLAN spoof-check and ingress/egress filtering. No switchdev, no representors-as-netdev — VFs are visible to the host only as PCI BDFs you assign to guests.
- **`switchdev` mode** (`eswitch_offloads.c`, `esw/acl/*_ofld.c`, `esw/bridge.c`) — each VF/SF gets a representor netdev on the PF; FDB chains (chain/prio) accept TC rules whose actions are HW-offloaded; the slow-path representor receives any unmatched packet and bridges to its representee through the host stack. This is the production mode for OVS-offload + cloud SR-IOV.

This Tier-3 covers the ~7700 lines of `eswitch.c` + `eswitch_offloads.c` plus the supporting `esw/` subdirectory (~3500 lines of ACL, QoS, bridge, devlink-port, indir-table, ipsec, vporttbl modules).

## Rust translation posture

eSwitch is the highest-CVE-density mlx5 component. The translation strategy is structural: replace the union-of-modes-with-runtime-mode-byte pattern that upstream uses with two distinct types `EswLegacy` and `EswOffloads`, where each carries only the state valid for that mode. Mode transitions become explicit constructors that consume the old type:

- `Eswitch<Legacy>` ↔ `Eswitch<Offloads>` via `Eswitch::switch_to_offloads(self: Eswitch<Legacy>) -> Result<Eswitch<Offloads>, (Eswitch<Legacy>, EswError)>` — on failure the legacy state is restored, type-system-enforced.
- `Vport` carries a phase: `Vport<Disabled> → Vport<Enabled<Legacy>> → Vport<Repr<Offloads>>`. Methods that only make sense in offloads mode (`set_chain_prio`, `tc_offload_add`) only exist on `Vport<Repr<Offloads>>`.
- FDB chains modelled as `FdbChain<Chain, Prio>` with const generics where useful; chain-0 / prio-0 is the entry rule (NIC-side ingress from uplink) and is a distinct singleton type.
- Mapped objects (`mlx5_mapped_obj` tagged union of chain/sample/int_port_metadata/act_miss) → Rust enum with exhaustive matching; the `void *` mapping table becomes `BTreeMap<u32, MappedObj>` with typed retrieval.
- Per-vport UC/MC L2 address tables → `BTreeMap<MacAddr, AddrState>` per vport; the `vport_addr` walking + `add/del/promiscuous` action enum becomes a state-machine pass.
- Bridge offload (`esw/bridge.c`) — Linux-bridge-compatible FDB learning + VLAN filtering + multicast snooping, all offloaded; in Rust this is a separate `BridgeOffload` actor that listens for `switchdev_notifier` events.
- Locking — eSwitch's `state_lock` rwsem becomes a typed `parking_lot::RwLock<EswState>`; the legacy/offloads mode-flip is the only writer. `mode_lock` and `state_lock` ordering (mode_lock first) becomes a `LockGroup` enforced at compile time.

The grsec/PaX section below is mandatory: eSwitch is a **multi-tenant security boundary**. A bug here lets one VF read/inject another VF's traffic — the structural property that an SR-IOV NIC is supposed to provide.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlx5_eswitch` | per-PF eSwitch state | `Eswitch<Mode>` (phase-typed) |
| `struct mlx5_vport` | per-vport state (uplink + ECPF + PF + VFs + SFs) | `Vport<Phase>` |
| `enum devlink_eswitch_mode` | LEGACY vs SWITCHDEV | `EswMode` enum |
| `mlx5_eswitch_init(dev)` / `_cleanup` | per-PF init / teardown | `Eswitch::init` / `Drop` |
| `mlx5_eswitch_enable(esw, num_vfs, mode)` / `_disable` | bring vports up/down with mode | `Eswitch::enable` |
| `mlx5_devlink_eswitch_mode_set(devlink, mode)` / `_get` | devlink mode toggle | `Devlink::set_esw_mode` |
| `esw_offloads_start(esw)` / `_stop(esw)` | switch into/out of offloads | `Eswitch::switch_to_offloads` / `back_to_legacy` |
| `mlx5_eswitch_load_vport(esw, vport, events_mask)` / `_unload_vport` | per-vport bring-up | `Vport::load` |
| `esw_acl_egress_lgcy_setup(esw, vport)` / `_offloads_setup` | egress ACL — spoof-check + VLAN strip | `EgressAcl::install` |
| `esw_acl_ingress_lgcy_setup` / `_offloads_setup` | ingress ACL — MAC/VLAN trust | `IngressAcl::install` |
| `esw_legacy_vport_acl_setup(esw, vport)` | legacy-mode per-vport ACL | `LegacyAcl::install` |
| `mlx5_esw_offloads_pair(esw, peer_esw)` / `_unpair` | multi-PF (LAG/MultiHost) peer-mirror | `Eswitch::pair` |
| `mlx5_esw_offloads_devlink_port_register` / `_unregister` | devlink-port per repr | `DevlinkPort::register` |
| `esw_offloads_set_min_inline(esw, vport, mininline)` | min-inline header for non-encapsulated TX | `Vport::set_min_inline` |
| `__mlx5_eswitch_set_vport_vlan(esw, vport, vid, qos, set_flags)` | per-vport guest-VLAN config | `Vport::set_vlan` |
| `mlx5_eswitch_set_vport_trust(esw, vport, setting)` | trust toggle (uplink-PF only) | `Vport::set_trust` |
| `mlx5_eswitch_set_vport_spoofchk` | MAC spoof-check toggle | `Vport::set_spoofchk` |
| `mlx5_eswitch_get_vport_stats(esw, vport, stats)` | per-vport RX/TX counters | `Vport::stats` |
| `mlx5_esw_offloads_add_send_to_vport_rule` / `_meta_rule` | slow-path forwarding | `Eswitch::add_meta_rule` |
| `mlx5_eswitch_add_send_to_vport_rule` | offloads-mode send-to-vport | `Eswitch::send_to_vport` |
| `esw_get_chain_prio_meta` / `_set_chain_prio_mapping` | chain/prio metadata register C0..C7 | `ChainMap::lookup` / `insert` |
| `mlx5_esw_bridge_setup(esw)` / `_cleanup` | HW-bridge offload init | `BridgeOffload::init` |
| `mlx5_esw_qos_init(esw)` / `_create_vport_sched_elem` | per-vport scheduler + rate | `Qos::init` / `Vport::set_rate` |
| `mlx5_esw_vport_match_metadata_supported` | tenant-isolation metadata feature gate | `Eswitch::caps.match_metadata` |
| `mlx5_esw_indir_table_init` / `_handle_lookup` | indirect-table dispatch | `IndirTable::lookup` |
| `mlx5_esw_event_listeners[]` | async-event listeners (VHCA add/del, PORT_CHANGE, etc.) | `Eswitch::on_event` |

## Compatibility contract

REQ-1: Devlink eswitch mode ABI — `devlink dev eswitch set pci/0000:XX:XX.0 mode {legacy,switchdev}` works exactly as upstream; transition succeeds only when no traffic is in flight on any VF (returns `-EBUSY` otherwise, matching upstream).

REQ-2: Per-vport ABI parity — `ip link set <pf> vf <n> {mac,vlan,trust,spoofchk,rate,state,query_rss}` all map to the same firmware ops upstream uses (no new restrictions, no removed ops).

REQ-3: Switchdev representor netdev — for each VF/SF enabled, a representor netdev named `<pf>_<vport>` (e.g. `enp1s0f0_0`) appears on the host; ethtool, mtu, qdisc, tc all work on it; carrier follows the VF's linkstate.

REQ-4: TC HW-offload — `tc filter add dev <repr> ingress protocol ip flower ... action <act>` installs HW rules when `hw_tc_offload` is enabled. Supported actions match upstream: `mirred {redirect,egress redirect}`, `vlan {push,pop,modify}`, `tunnel_key {set,unset}`, `pedit`, `csum`, `gact {drop,pass,trap}`, `ct`, `sample`, `meter` (police), `mpls {push,pop}`.

REQ-5: FDB chains — chain 0 = root, prio 0..N — same chain/prio space upstream exposes. Chain >0 prio >0 are slow-path-callable via `goto chain X` action.

REQ-6: Multi-PF LAG — when two ConnectX PFs are bonded (active-backup, active-active, hash), `mlx5_esw_offloads_pair` mirrors representor state across both eSwitches; one tc rule applied to a repr installs on both PFs.

REQ-7: HW bridge offload — when a representor is added to a Linux bridge with hw_offload on, `esw/bridge.c` mirrors FDB learning, VLAN filter table, and multicast snooping into HW; `bridge fdb` / `bridge vlan` / `bridge mdb` all visible on the repr.

REQ-8: Per-vport QoS — `devlink-rate` (group + leaf nodes), per-vport max_rate (Mbps), per-vport min_rate (relative weight), TX scheduler hierarchy 4 levels deep (root → group → vport-group → leaf).

REQ-9: Sub-Functions (SF) — `mlx5_sf` creates a vport that acts like a VF but lives on the PF (no separate PCIe BDF); SFs get representor netdevs too; FDB rules apply to SFs identically.

REQ-10: ECPF (Embedded CPU Physical Function) — on BlueField DPU host, the ECPF is the PF that runs the eSwitch; the host sees only its representor. `mlx5_devlink_eswitch_get` returns the ECPF eswitch when the host queries.

REQ-11: Async events — VHCA add/del events (when VF probed/removed from guest), PORT_CHANGE (uplink link transitions), eswitch-mode transitions, ipsec-status-change all delivered via `mlx5_eq_notifier_register`.

REQ-12: IPsec per-representor — when MLX5_CAP_IPSEC_OFFLOAD present, `esw/ipsec.c` installs per-rep IPsec encap/decap rules; XFRM state add via netdev's xdo_dev_state_add maps to FDB rules.

REQ-13: Match-metadata cap — when `match_metadata` cap present (ConnectX-6+), every packet through the FDB carries a 32-bit metadata register (`reg_c0`) tagged with the source vport-id; this is the structural enforcement of tenant isolation. Rookery must use match_metadata whenever the cap is present; the fallback (no metadata) is for ConnectX-5 only.

REQ-14: Stop-eswitch / start-eswitch coordination with `fw_reset.c` — eswitch is torn down before FW reset, re-initialized after.

## Acceptance Criteria

- [ ] AC-1: `devlink dev eswitch set pci/X mode switchdev` succeeds on a ConnectX-6 with no kernel splat; representor netdevs appear named `<pf>_<vportid>` matching upstream.
- [ ] AC-2: `tc filter add dev <repr> ingress flower ... action mirred redirect dev <other_repr>` installs a HW rule (verified via `tc filter show` `hw_offload=yes` flag); packet redirected through HW datapath without host CPU touch.
- [ ] AC-3: OVS-offload smoke: `ovs-vsctl set Open_vSwitch . other_config:hw-offload=true`, add a flow between two VF representors, push pps with pktgen, verify zero host-CPU softirq cost.
- [ ] AC-4: Spoof-check enforced: VF guest sends packet with non-assigned source MAC, ACL drops at egress, counter increments on `ip -s link show <pf> vf <n>`.
- [ ] AC-5: Per-vport rate: `devlink port function rate set ... tx_max 5gbit` on a representor → iperf3 from the VF guest tops at ~5 Gbps.
- [ ] AC-6: Multi-PF pair: two ConnectX-6 in LAG, switch mode to switchdev on the bond — both representor lists appear, one tc rule installed mirrors to both.
- [ ] AC-7: Bridge offload: add two representors to `br0`, `bridge link set hwmode vepa`, `bridge vlan add dev <repr> vid 100 pvid untagged` — VLAN 100 ingress from VF mapped through FDB, no host-CPU.
- [ ] AC-8: Mode-flip rollback: trigger esw_offloads_start failure mid-transition (force a FW cmd failure via debugfs), verify state returns to legacy with no leaked vports.
- [ ] AC-9: SR-IOV+VFIO: enable 8 VFs on PF, switch to switchdev mode, pass VFs 0..3 to KVM guests via vfio-pci, send traffic across — host eSwitch enforces ACLs.
- [ ] AC-10: BlueField host: with BlueField in ECPF mode, host sees only its representor; ECPF (DPU side) controls full eSwitch.

## Architecture

**Mode as a phantom type.** Upstream's `mode` is a `u16` field in `struct mlx5_eswitch` that branches into legacy or offloads code paths via `if (esw->mode == MLX5_ESWITCH_LEGACY)`. Rookery promotes this to a phantom type: `Eswitch<Legacy>` and `Eswitch<Offloads>` are distinct types. The mode switch is a consuming method:

```rust
impl Eswitch<Legacy> {
    pub fn switch_to_offloads(self) -> Result<Eswitch<Offloads>, (Eswitch<Legacy>, EswError)> {
        // (1) quiesce all vports
        // (2) tear down legacy ACL tables
        // (3) install offloads FDB chains
        // (4) register representor netdevs
        // on any failure: rebuild legacy state, return Err((self_restored, e))
    }
}
```

This makes it impossible to call `add_tc_rule` on a `Eswitch<Legacy>` — the method doesn't exist on that type. A bug class (calling switchdev-only ops in legacy mode) is closed at compile time.

**Vport lifecycle.** Each `Vport<Phase>` advances through `Disabled` → `Enabled<M>` → `Loaded<M>`, where `M` is the eSwitch mode. The events bitmask (`MLX5_VPORT_UC_ADDR_CHANGE | MLX5_VPORT_MC_ADDR_CHANGE | MLX5_VPORT_PROMISC_CHANGE | ...`) is enabled at `load` time. UC/MC address change events arrive on the async-EQ and feed into the per-vport `BTreeMap<MacAddr, AddrState>` walker that decides ADD/DEL/PROMISC actions.

**Slow-path representor.** In offloads mode, every representor netdev has a "slow-path" TX path that wraps the packet in a metadata-bearing send descriptor with `reg_c0 = source_vport_id` and submits it to the uplink. The receive side reads `reg_c0` on incoming packets and demultiplexes to the corresponding representor's RX queue. The `mlx5_esw_offloads_add_send_to_vport_rule` and the per-repr send-to-vport rule are the structural anchors of this path.

**FDB chains.** Switchdev mode installs a multi-table FDB:
- chain 0, prio 0: ingress rules from uplink + per-vport ingress
- chain 0, prio 1..: TC ingress rules
- chain >0: goto-target tables for chained rules (`tc filter ... action goto chain N`)
- per-rep egress ACL table at the end
- the "miss" path falls through to the slow-path representor on the host

Each rule installed via `tc` lowers to a `mlx5_flow_handle` with a match-mask + action-list; the action list compiles to a `modify_header` + `set_fte` firmware call. The `indir_table.c` machinery handles cases where the action set is too large for one FTE — it chains through an intermediate table.

**Bridge offload.** When a representor joins a Linux bridge, `esw/bridge.c` registers as a switchdev driver. FDB learn events (`SWITCHDEV_FDB_ADD_TO_DEVICE`) install per-bridge-port HW FDB entries; VLAN filter (`SWITCHDEV_OBJ_PORT_VLAN`) maps to per-vport ingress VLAN trust + push/pop rules; MDB add (`SWITCHDEV_OBJ_PORT_MDB`) installs per-multicast-group FDB entries. Bridge MAC/VLAN/MDB tables shadow into HW FDB chains.

**QoS hierarchy.** Per-vport scheduler element with `max_average_bw`, `bw_share`, parent-element pointer. The hierarchy: root → group → vport-group → vport-leaf, up to 4 levels. `devlink-rate` nodes map 1:1 to HW scheduler elements; `devlink port function rate set` updates the leaf for a specific representor.

## Hardening

- All firmware mailbox sends in eSwitch paths go through the typed `FwCmd` from `mlx5-core.md`; no raw `mlx5_cmd_exec` from eSwitch code.
- Mode-switch failures restore prior state atomically (no half-switched eswitch with partial state).
- Per-vport ACL install validates that the requested MAC/VLAN is allowed by the trust setting; trust=off VF cannot set MAC.
- `match_metadata` always used when supported; the no-metadata fallback is gated behind an explicit feature flag and warns at probe.
- FDB chain/prio number validated against firmware caps before install; out-of-range returns `-EINVAL` without partial install.
- TC offload action-list size validated against `MLX5_CAP_ESW_FLOWTABLE_FDB.max_modify_header_actions`; oversize is split via indir_table, never silently truncated.
- Per-vport stats reads use `READ_ONCE`-equivalent semantics and the counters are 64-bit-atomic on the FW side.
- Representor netdev unregister is coordinated with TC rule teardown via `flush_workqueue(eswitch->work_queue)`; no use-after-free of representor by stale tc rule.
- LAG pairing: peer-eswitch refcount tracked; unpair only happens when refcount drops to zero.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — all devlink/netlink ingress (TC, devlink-port, devlink-rate, bridge netlink) goes through nlattr-validate bounded copy; no user buffer reaches FW cmd mailbox unsanitized.
- **PAX_KERNEXEC** — `mlx5_esw_event_listeners[]`, the per-mode op tables (`esw_legacy_ops`, `esw_offloads_ops`), switchdev_notifier callback tables, and the TC-offload dispatch table are all `const` after init and live in W^X memory.
- **PAX_RANDKSTACK** — TC/devlink/sysfs entry points wrap their per-syscall stacks with the rerandomize hook before invoking eSwitch ops.
- **PAX_REFCOUNT** — `mlx5_eswitch->refcount`, per-vport refcounts, per-representor netdev refcounts, peer-eswitch pair refcount all use saturating-overflow refcounts; underflow at teardown is a hard panic (mis-pairing or double-unload).
- **PAX_MEMORY_SANITIZE** — per-vport context cleared on teardown (carries MAC, VLAN, trust state — leak across VF reassignment is a tenant-isolation break). FDB rule action-bufs that carry encap keys (IPsec, MACsec) sanitize on free.
- **PAX_UDEREF** — SMAP/SMEP enforced on every user-facing entry (TC, devlink, sysfs, ethtool, bridge netlink).
- **PAX_RAP / kCFI** — per-mode op vtables, switchdev notifier callbacks, TC action handlers, and FDB-rule action interpreters are kCFI-signed; indirect calls signature-checked.
- **GRKERNSEC_HIDESYM** — eSwitch debugfs (`/sys/kernel/debug/mlx5/<pf>/esw/*`) dumps FDB chain pointers, vport context addresses, flow-handle pointers; non-root reads scrub addresses.
- **GRKERNSEC_DMESG** — eSwitch info-level logs (vport enable/disable, FDB rule install) restricted; they reveal tenant topology in multi-tenant clouds.
- **CAP_NET_ADMIN** required on **every** eSwitch state-changing op — mode set, vport MAC/VLAN/trust/spoofchk/rate, TC rule add/del, devlink-port-fn create/delete, bridge-fdb-add for offloaded bridges.
- **Tenant isolation is structural** — match_metadata required whenever cap present; a VF cannot inject a packet that lacks its `reg_c0` source-vport tag (HW rejects). The driver enforces this by refusing to switch to offloads mode if the cap is missing on a multi-tenant configuration.
- **Mode-switch atomicity** — the legacy↔offloads transition is gated by a single `state_lock` rwsem held exclusive throughout; partial-mode states never observable. A failed switch restores state and reports the firmware error; no "half-switched" state.
- **Per-vport ACL spoof-check enforcement** — egress ACL with spoofchk=on installs an FW rule that drops any packet whose source MAC is not the assigned MAC. The rule is installed *before* the vport is enabled to forward, never after — closing the race where a VF sees its first packet through before spoof-check is up.
- **TC-action allowlist per representor** — the set of TC actions allowed on a representor is gated by `MLX5_CAP_ESW_FLOWTABLE_FDB.actions_bitmask`; unsupported actions return `-EOPNOTSUPP` rather than passing to firmware (which would silently no-op them in some FW versions, an exfil class).
- **Indir-table depth cap** — chained indirect-table dispatch has a max depth (upstream is implicit at ~16); Rookery makes it an explicit constant and returns `-ELOOP` on overflow so a malicious TC ruleset can't OOM the FDB.
- **FDB rule install rate-limit** — TC rule add/del rate-limited per representor to prevent a hostile container orchestrator from overflowing the FW cmd queue.
- **Sub-function (SF) handle randomization** — when allocating SF function-id, the allocation order is randomized (LATENT_ENTROPY) so a tenant can't infer sibling-SF identities by enumerating IDs.

Per-doc rationale: eSwitch is the tenant-isolation boundary for SR-IOV NICs. If `vport A` can read `vport B`'s packets, the cloud guarantee is broken. Historical CVEs in this exact code (CVE-2023-25775 use-after-free in mlx5_ib via eswitch teardown; mailbox-bound bugs from the chain dispatcher) demonstrate the bug class. The reinforcement above operates at three layers: (1) compile-time mode-typing prevents a whole bug class (calling offloads-ops in legacy mode); (2) firmware-side match_metadata enforces tenant tagging in silicon; (3) the runtime/kCFI/refcount/sanitize layer catches the residue.

## Open Questions

- [ ] Q1: Should Rookery preserve `legacy` mode at all? Upstream still ships it because old userspace assumes it (pre-switchdev OpenStack neutron, pre-OVS-offload setups). Cost of keeping it: doubles the code surface. Recommendation: keep but make `switchdev` the default and emit a deprecation warning when `legacy` is selected.
- [ ] Q2: Multi-host (LAG-on-MultiHost) pairing — upstream's pairing logic is one of the bug-heavy areas. Rookery could initially refuse pairing and force users to a single-PF eSwitch; lose perf for the LAG-offload customers. Or implement pairing with the typed-state machine.
- [ ] Q3: HW-bridge offload (`esw/bridge.c`) — this duplicates a lot of bridge-driver state. Rookery could either reimplement as a separate `BridgeOffload` actor (clean but big) or proxy through the existing bridge driver's switchdev hooks (less code but couples to bridge implementation).
- [ ] Q4: TC-flower → FDB-rule lowering — upstream's `parse_tc_actions` is ~3k lines of nested matching. A Rookery rewrite as a typed pipeline (parse → typed-IR → FW-rule lowering) is desirable but is a big rewrite; defer or do up front?
- [ ] Q5: Match-metadata fallback for ConnectX-5 — keep the fallback path (and accept the weaker isolation) or refuse offloads mode entirely on ConnectX-5? Cloud deployment context matters.
- [ ] Q6: BlueField ECPF/ECPF-VF asymmetry — the ECPF runs the eSwitch but is itself a PF that needs its own representation. Need to model this cleanly with the phase types.

## Verification

- **Kani SAFETY**: prove the mode-switch state machine has no path where both `legacy_acl_installed` and `offloads_fdb_installed` are true simultaneously (mutex-mode invariant). Prove `Vport::set_mac` refuses when `trust=off` and `mac != assigned_mac`.
- **TLA+**: model the LAG-paired mode-switch protocol across two PFs; check that any sequence of mode-flip requests on either PF cannot leave the pair in mixed (one-legacy / one-offloads) state — that would silently break the offload guarantee on the legacy side.
- **Verus**: functional spec of `parse_tc_actions` → `FdbRule` lowering: for each TC-flower spec, the produced FdbRule's match-mask equals the original spec's match-set and its action-list equals the original spec's action-list (modulo the documented HW-incompatibility rejections).
- **Kani+Verus**: invariant that every `mlx5_flow_handle` returned by `eswitch_add_rule` is recorded in the rule-tracker, and the rule-tracker is empty at `eswitch_disable`. (Leak detection at the design level.)
- **Integration**: dual-host TC-offload smoke (OVS-offload with HW-offload=yes, push 10M flows/sec install rate, verify zero CPU on the datapath). SR-IOV passthrough to 16 KVM guests with isolation test — VF[i] cannot read VF[j]'s traffic for i≠j. Mode-flip stress: 1000 legacy↔offloads transitions back-to-back with traffic running on one VF — survive without lost packets or kernel splat.
- **Fuzz**: TC-rule fuzzer (mutate flower match + action set) against the parse → FW-rule lowering path; should never produce an FW cmd that the FW rejects with status=BAD_PARAM (any such case is a bug in our validator).
- **Penetration**: with two VFs assigned to two unprivileged KVM guests, try every TC-rule shape from inside the guest (via vDPA or virtio passthrough) to escape tenant isolation. None should succeed.

## Out of Scope

- mlx5_core firmware-cmd interface, EQ/CQ, health — covered in `mlx5-core.md`
- mlx5e datapath (NAPI, page-pool, XDP) — covered in `mellanox-mlx5-en.md`
- mlx5_ib RDMA verbs — separate doc under `drivers/infiniband/mellanox-mlx5-ib.md`
- mlx5_vdpa — separate doc under `drivers/vdpa/mellanox-mlx5-vdpa.md`
- Detailed mlx5e_rep slow-path datapath internals (the per-skb fast path on representors) — covered briefly here, in depth in the en doc
- VDPA-mediated VF management — separate doc
- Sub-Function (`mlx5_sf/*`) bring-up details — referenced here, full doc is `drivers/net/ethernet/mellanox/mlx5/mlx5-sf.md` (future iteration)
- BlueField DPU host-driver (`lib/hv_vhca`, `sf/dev`) — future iteration
