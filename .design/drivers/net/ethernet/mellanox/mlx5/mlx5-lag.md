# Tier-3: drivers/net/ethernet/mellanox/mlx5/core/lag/{lag,mp,mpesw,port_sel,debugfs}.c — Mellanox mlx5 multi-port LAG aggregation (active-backup, hash-based, RoCE-LAG, SR-IOV-LAG, multipath, MPESW shared-FDB)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
depends-on:
  - drivers/net/ethernet/mellanox/mlx5/mlx5-core.md
  - drivers/net/ethernet/mellanox/mlx5/eswitch.md
upstream-paths:
  - drivers/net/ethernet/mellanox/mlx5/core/lag/lag.c           (~2470 lines: core LAG state machine, modes, netdev tracker, bond_work, demux)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/lag.h           (mlx5_lag struct, mode enum, lag_func, lag_tracker, hash buckets)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/mp.c            (~410 lines: multipath mode — IPv4/IPv6 ECMP route HW-offload)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/mpesw.c         (~265 lines: Multi-Port eSwitch — shared-FDB across both PFs)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/port_sel.c      (~660 lines: HW port-selection table, hash bucket mapping, definers)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/debugfs.c       (~185 lines: state, mode, hash-buckets debugfs)
  - drivers/net/ethernet/mellanox/mlx5/core/lag/mp.h
  - drivers/net/ethernet/mellanox/mlx5/core/lag/mpesw.h
  - drivers/net/ethernet/mellanox/mlx5/core/lag/port_sel.h
  - include/linux/mlx5/{driver,fs,eswitch}.h
  - include/linux/netdev_features.h                                              (netdev_lag_tx_type, netdev_lag_hash)
  - net/core/bonding/                                                            (NETDEV_CHANGEUPPER tracker source)
-->

## Summary

mlx5 LAG (Link Aggregation Group) is the on-NIC bonding feature for dual-port ConnectX cards (and multi-port BlueField-3). When two ports of the same NIC are bonded by the Linux `bond0` or `team0` driver — or by an active-backup IPv4/IPv6 multipath route — mlx5_lag offloads the bond into firmware so the host CPU does not see per-packet RSS-spread or active-backup decisions. The benefit is enormous: a 200 Gb/s dual-100G LAG can sustain line rate with zero host softirq overhead because both the egress port selection and the RoCE QP port-affinity decisions are made in silicon.

Five distinct LAG modes coexist (chosen by the upper-driver shape, not by the user):

- **`NONE`** — bond inactive (single-port operation per device).
- **`ROCE`** — RoCE/RDMA traffic LAG-aware. RoCE QPs created on one PF can fail over to the other PF's port; needed when an RDMA-NIC-pair is bonded for HA.
- **`SRIOV`** — VFs on either PF share both ports; egress port selection HW-offloaded per VF; ingress representor consistent regardless of which port the packet arrived on.
- **`MULTIPATH`** — IPv4/IPv6 ECMP routes installed in the kernel route table are offloaded as hash-based port selection (`mp.c`); the next-hop nexthops live on different PFs but appear as one offload.
- **`MPESW`** (Multi-Port eSwitch) — single shared FDB across both PFs; OVS-offload rules installed once apply to both ports. This is the cloud-data-center mode that lets a single OVS bridge spread across a dual-port LAG without per-port rule duplication.

The mode is auto-selected based on what's bonding the two PFs: a Linux bond driver triggers ROCE or SRIOV (depending on offload caps); an IPv4 multipath route triggers MULTIPATH; a devlink request explicitly selects MPESW. Mode transitions are coordinated through the global `devcom` (device communication) mechanism so the two PFs negotiate in lock-step.

This Tier-3 covers ~4000 lines of `lag/*.c`.

## Rust translation posture

LAG is a state machine across two `MlxCoreDev` instances + an upper bonding/teaming driver event source + a per-mode active sub-controller (mp/mpesw/port_sel). Upstream uses `mlx5_lag` as a shared struct keyed by xarray, with mode-specific fields union'd in. Rookery translates:

- `Lag<Mode>` as a phantom-typed handle (`Lag<NoneMode>`, `Lag<RoceMode>`, `Lag<SriovMode>`, `Lag<MultipathMode>`, `Lag<MpeswMode>`); mode transitions go through `activate(self: Lag<NoneMode>, tracker: BondTracker, mode: Mode) -> Result<Lag<Mode>, ...>` and `deactivate(self: Lag<Mode>) -> Lag<NoneMode>`.
- The shared LAG instance per ConnectX card is `Arc<LagBundle>` where `LagBundle` owns both `Arc<MlxCoreDev>` (PF0 + PF1) plus the current mode state. Per-PF `priv.lag` is a `Weak<LagBundle>` to avoid the upstream reference cycle.
- The `lag_tracker` (egg-collection of netdev events) becomes a typed `BondTracker { tx_type: TxType, port_state: [PortState; 2], hash_type: HashType, bond_speed_mbps: u32 }`.
- The `v2p_map` array (virtual-port-to-physical-port hash mapping, 16 buckets per port) is a `[[PhysPort; 16]; MAX_PORTS]` plain array; no allocations during fast-path.
- The `bond_work` delayed-work pattern (debounce netdev events into a single mode transition) becomes a tokio `tokio::time::sleep` + cancellable task; the latest tracker state wins.
- The `devcom` cross-PF coordination becomes a typed `LagDevcom` mailbox with explicit `Negotiate { mode }` / `Acknowledge` messages.
- The HW port-selection rule (an FDB rule that hash-selects port based on header fields) is encoded as a typed `PortSelDefiner { match_fields: Vec<HeaderField>, hash_bucket_count: u8 }`.

The grsec/PaX section is mandatory: LAG creates a *shared* eSwitch surface across two PFs, doubling the tenant-isolation surface. Mode-flip bugs here can momentarily allow traffic from one tenant's VF to egress out the wrong port — observable on a connected switch as a MAC-mismatch frame.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlx5_lag` | per-card LAG state (mode, ports, buckets, tracker, demux table) | `LagBundle` |
| `struct lag_func` | per-PF role-in-LAG (idx, netdev, port_change_nb) | `LagFunc` |
| `struct lag_tracker` | latest netdev event snapshot | `BondTracker` |
| `enum mlx5_lag_mode` | NONE / ROCE / SRIOV / MULTIPATH / MPESW | `LagMode` enum |
| `mlx5_lag_is_supported(dev)` | cap-derived support check | `MlxCoreDev::lag_supported` |
| `mlx5_lag_dev(dev)` / `mlx5_lag_pf(ldev, i)` | LAG handle / per-PF handle accessor | `MlxCoreDev::lag()` / `LagBundle::pf(i)` |
| `mlx5_lag_dev_alloc(dev)` / `_free(ldev)` | LAG allocator on first PF probe | `LagBundle::new` / `Drop` |
| `mlx5_dev_list_add_one(dev)` / `_remove_one(dev)` | PF register/unregister with LAG bundle | `LagBundle::add_pf` / `remove_pf` |
| `mlx5_do_bond(ldev)` / `_do_bond_work` | debounced bond-event handler (entry to mode flip) | `LagBundle::on_bond_event` |
| `mlx5_lag_check_prereq(ldev)` | prereq check (caps, both PFs ready, vendor-id match) | `LagBundle::check_prereq` |
| `mlx5_activate_lag(ldev, tracker, mode, shared_fdb)` | install LAG in firmware | `LagBundle::activate` |
| `mlx5_deactivate_lag(ldev)` / `mlx5_disable_lag(ldev)` | tear down LAG | `Lag::deactivate` |
| `mlx5_modify_lag(ldev, tracker)` | re-program v2p_map on tracker change | `Lag::modify` |
| `mlx5_cmd_create_lag(...)` / `_destroy_lag(...)` / `_modify_lag(...)` | FW cmd wrappers | `FwCmd::create_lag` / etc |
| `mlx5_lag_set_port_sel_mode(ldev, tracker, mode_flags)` | choose between QUEUE_AFFINITY and PORT_SEL | `Lag::set_sel_mode` |
| `mlx5_lag_port_sel_create(ldev)` / `_destroy` | HW port-selection table create/destroy | `PortSel::create` / `Drop` |
| `mlx5_lag_port_sel_modify(ldev, ports)` | reprogram per-bucket port assignment | `PortSel::modify_buckets` |
| `mlx5_lag_mp_init(ldev)` / `_cleanup` | multipath mode init (FIB notifier) | `MultipathCtl::init` |
| `mlx5_lag_fib_event(nb, event, ptr)` | FIB add/del → port-sel hash update | `MultipathCtl::on_fib_event` |
| `mlx5_mpesw_init(ldev)` / `_cleanup` | MPESW shared-FDB mode init | `MpeswCtl::init` |
| `mlx5_mpesw_metadata_set(ldev, mode)` | source-port metadata for MPESW | `MpeswCtl::set_metadata` |
| `mlx5_lag_demux_init(dev, ft_attr)` / `_demux_cleanup` | per-PF demux table for slow-path | `LagDemux::init` |
| `mlx5_lag_demux_rule_add(dev, vport, idx)` | per-vport demux rule install | `LagDemux::add_rule` |
| `mlx5_handle_changeupper_event(...)` | NETDEV_CHANGEUPPER from bond/team | `LagBundle::handle_changeupper` |
| `mlx5_handle_changelowerstate_event(...)` | NETDEV_CHANGELOWERSTATE | `LagBundle::handle_changelower` |
| `mlx5_lag_shared_fdb_supported(ldev)` | check shared-FDB cap | `LagBundle::shared_fdb_supported` |
| `mlx5_lag_set_vports_agg_speed(ldev)` | sum speeds across PFs for representor speed report | `Lag::set_vports_agg_speed` |
| `mlx5_get_str_port_sel_mode(mode, flags)` | mode → string for debugfs | `LagMode::name` |

## Compatibility contract

REQ-1: Auto-bond detection — when a Linux `bond` or `team` netdev is created over both `mlx5_core_en` netdevs of the same card, `mlx5_handle_changeupper_event` fires; mode is auto-selected (ROCE if `MLX5_CAP_ROCE.lag` set, otherwise SRIOV if eswitch active, otherwise classic NIC-LAG). No user explicit-enable needed for this case.

REQ-2: Tx-type mapping — `BOND_MODE_ACTIVEBACKUP` → active-backup (one port active, the other passive); `BOND_MODE_XOR` / `802_3AD` (LACP) → hash-based with per-bucket v2p_map; `BOND_MODE_BROADCAST` not supported (returns -EOPNOTSUPP).

REQ-3: Hash-buckets — 16 buckets per egress port (8 ports max ⇒ 128 buckets total upstream). Hash function selected by `bond.xmit_hash_policy`: L2 (dst MAC), L23 (dst MAC + L3 src/dst), L34 (5-tuple). Each bucket has an assigned physical port; bucket→port assignment recomputed on tracker change.

REQ-4: ROCE-LAG — when active, RoCE QPs created on the LAG-master PF inherit a "LAG QP-bind" that routes responses through whichever physical port the request arrived on. On port failure, the surviving port absorbs all QPs.

REQ-5: SRIOV-LAG — VFs from both PFs share a unified eswitch view; representor netdev is created on the LAG-master only. TX from a VF hashes through the v2p_map to choose physical port.

REQ-6: MULTIPATH mode — IPv4/IPv6 ECMP routes installed in the kernel route table fire `FIB_EVENT_ENTRY_REPLACE/_DEL`; the handler offloads them as per-bucket port assignments. The bond driver is *not* involved; this is for cases where the two PFs have different IPs and ECMP routing distributes traffic.

REQ-7: MPESW (Multi-Port eSwitch) — single shared FDB across both PFs. OVS-offload rules installed on either PF apply to both. Requires `MLX5_CAP_GEN.shared_fdb` on both PFs. Activated explicitly via devlink (`devlink dev param set ... lag mpesw true`).

REQ-8: Mode transitions atomic — mode changes are coordinated via `devcom` cross-PF rendezvous; both PFs negotiate to the same mode or both stay at NONE.

REQ-9: Devcom integration — mlx5_lag is one component of mlx5_devcom (device communication between PFs of the same card); other components (esw, ipsec, fdb) coordinate through the same mechanism.

REQ-10: Port-change events — `MLX5_EVENT_TYPE_PORT_CHANGE` per-PF feeds into the bond_tracker; link-down on one port triggers v2p_map remap (active-backup) or bucket redistribution (hash).

REQ-11: Speed aggregation — when LAG active, `mlx5e` ethtool reports the aggregate speed (e.g. 200 Gbps for dual-100G) on the bond netdev, individual port speed on per-PF netdev.

REQ-12: SR-IOV interaction — LAG can be created with VFs already active (live bond enable); LAG must stay active across VF add/remove of either PF.

## Acceptance Criteria

- [ ] AC-1: Dual-port ConnectX-6: create `bond0` with both ports in `802_3AD`, switch connected to LACP-enabled aggregation group. iperf3 single-flow ~100 Gbps; multi-flow saturates 200 Gbps with zero host softirq overhead (verified via `mpstat`).
- [ ] AC-2: Failover: with bond active and traffic flowing, `ip link set <port1> down` → bucket redistribution; iperf3 throughput drops to single-port without disconnection; restoration adds the port back into rotation.
- [ ] AC-3: SRIOV-LAG: enable 4 VFs on each PF, switch to switchdev mode on both, bond the uplinks — VFs from both PFs visible in a unified representor list on the LAG-master PF.
- [ ] AC-4: RoCE-LAG: with `rdma-core` tooling, create QPs on one PF, send RDMA-write to peer over the LAG bond; pull one cable mid-flight; QPs migrate to surviving port; no `IBV_WC_GENERAL_ERR`.
- [ ] AC-5: Multipath: install IPv4 ECMP route via `ip route add 10.0.0.0/24 nexthop via 1.1.1.1 dev <pf1> nexthop via 2.2.2.2 dev <pf2>`; HW port-sel table reflects ECMP; per-flow distribution observable via per-port counters.
- [ ] AC-6: MPESW: with two PFs in MPESW mode + OVS-offload, install one tc rule on PF0's representor; rule appears HW-offloaded on PF1 too; traffic through PF1 follows the rule.
- [ ] AC-7: Mode transition rollback: trigger a mid-flight ACTIVATE_LAG cmd failure (debugfs); LAG returns to NONE cleanly on both PFs; no half-bonded state.
- [ ] AC-8: Hot tear-down: with bond active, `ip link delete bond0` → LAG deactivates cleanly; per-PF eswitch state restored; traffic resumes on individual PFs.
- [ ] AC-9: Devcom rendezvous: simulate one PF being slower to respond (debugfs delay knob); LAG activation waits up to the timeout, then aborts cleanly on both PFs if the slow one never responds.

## Architecture

**LAG is a per-card object owned by `LagBundle`.** It's allocated when the *first* PF of a multi-PF card probes; the second PF registers into the existing bundle. Per-PF `priv.lag` holds a `Weak<LagBundle>`. When the last PF unregisters, the bundle drops. This shared-by-Arc design replaces upstream's `kref` + manual `kref_get/put`.

**The mode FSM.** `Lag<NoneMode>` is the resting state. Mode transitions are triggered by exactly three sources: (1) a `NETDEV_CHANGEUPPER` event when a bond/team driver adds both PF netdevs as slaves; (2) a `FIB_EVENT_ENTRY_*` when ECMP routes are installed; (3) a devlink user request (MPESW). Each source feeds into `LagBundle::on_bond_event`, which debounces (via the bond_work delayed task — 100ms upstream) so back-to-back netdev events compose into a single mode flip. After the debounce, the latest `BondTracker` is checked against `check_prereq` and used to drive `activate`.

**Port-selection definer.** The HW port-sel feature (when `MLX5_CAP_PORT_SELECTION.port_select_flow_table` is set, ConnectX-6+) installs an FDB rule that hashes selected packet header fields and selects one of N buckets, each mapped to a physical port. The `port_sel.c` machinery encodes:
- A *definer* (which header fields hash) — chosen by `bond.xmit_hash_policy`.
- A *bucket-to-port* map (`v2p_map[port * 16 + bucket]`) — recomputed on every `modify` and pushed via `MODIFY_LAG` FW cmd.
- The rules cover ingress (steering RX to representor) + egress (steering TX out the chosen port).

For older HW without port-sel-flow-table, mlx5 falls back to QUEUE_AFFINITY (per-tx-queue port pinning).

**Multipath (`mp.c`).** Registers a FIB notifier; on ECMP route add/del, recomputes the v2p_map. Only IPv4 and IPv6 ECMP supported; MPLS not. Coexists with bond-driven LAG (mutually exclusive — only one mode active at a time).

**MPESW (`mpesw.c`).** The shared-FDB mode. Both PFs' eswitches merge into a single FDB; rule installs on either PF mirror to both. Source-port metadata (`reg_c0` bits 28..31) carries the originating-PF id so tenant isolation cannot be confused across PFs. Activation issues `MODIFY_LAG` with shared_fdb=1 + per-PF stop/start of eswitch ops.

**Demux table.** Per-PF FDB egress table that "demuxes" slow-path packets (those that hit the miss-rule) back to their source representor. Installed by `mlx5_lag_demux_init`; per-vport demux rules added by `mlx5_lag_demux_rule_add`.

**Lifecycle integration with eswitch and fw_reset.** When eswitch goes through a mode transition, the LAG state-lock is held to prevent a concurrent LAG mode-flip; conversely, a LAG activate that needs to modify the eswitch takes the eswitch state-lock. Lock order: lag.lock → eswitch.state_lock → eswitch.mode_lock. FW reset notifies LAG to deactivate before reset and reactivate after.

## Hardening

- `mlx5_lag_check_prereq` validates: both PFs are the same vendor/device, both are `MLX5_CAP_GEN.lag_master`, both have matching cap shape, both are netns-eq (no cross-netns LAG). Refuses with -EINVAL otherwise.
- Mode-flip failures restore prior state via `disable_lag` rollback; no partial-mode states.
- `modify_lag` cmd issued with the full v2p_map every time (no partial updates); guarantees FW state matches driver state.
- All FIB events validated: route must be IPv4/IPv6, must have ≥2 nexthops, nexthops must resolve to both PFs of the LAG bundle. Other shapes ignored.
- `bond_tracker` updates serialized through `lag.lock`; no concurrent tracker overwrites.
- Devcom rendezvous has a timeout (5s upstream); after timeout the request is aborted on both PFs.
- Per-PF `port_change_nb` notifier is registered before LAG activation and unregistered after deactivation; no stale notifier handlers.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — devlink params for LAG (`lag_mode`, `lag_mpesw_active`) ingress through bounded nlattr-validate; bond/team driver events are kernel-internal but their netdev pointers are validated against the per-bundle registry before use.
- **PAX_KERNEXEC** — `mlx5_handle_changeupper_event` callback table, the per-mode op tables (`lag_mp_ops`, `lag_mpesw_ops`, `lag_port_sel_ops`), FIB notifier callbacks, and the devcom protocol handler tables are `const` after init in W^X memory.
- **PAX_RANDKSTACK** — netdev-event entry paths from bond/team driver are per-syscall stack-rerandomized; the FIB notifier and devlink entry path inherit the same protection.
- **PAX_REFCOUNT** — `LagBundle` strong count + per-PF weak-ref count + demux-rule refcount + port-sel definer refcount all use saturating refcounts; underflow on `Drop` is a hard panic.
- **PAX_MEMORY_SANITIZE** — `mlx5_lag` struct cleared on free; the `v2p_map` array carries port-selection state derived from in-flight traffic and is sanitized so a subsequent LAG (re-bonded on the same card) doesn't inherit prior state.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry (devlink, ethtool, sysfs); the netdev-event paths are kernel-internal but defensively re-validate netdev pointers.
- **PAX_RAP / kCFI** — per-mode op vtables (lag_mp, lag_mpesw, lag_port_sel), FIB-notifier `notifier_block`, NETDEV-event notifier_block, devcom message dispatcher all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/mlx5/<pf>/lag/*`) dumps the v2p_map and mode flags including internal pointers (lag_demux_ft, demux_rules xarray entries); non-root reads scrub addresses.
- **GRKERNSEC_DMESG** — LAG mode-change log lines restricted; in a multi-tenant cloud, knowing whether a customer's NIC is LAG-bonded reveals deployment topology.
- **CAP_NET_ADMIN required on devlink LAG ops** — MPESW activate/deactivate, lag_mode param set; per-PF speed-aggregation knob.
- **Cross-tenant port-mismatch closed** — when LAG is in SRIOV mode and a VF hostnames a MAC, the resulting egress packet must hash through the v2p_map to a port whose source-port metadata matches the VF's owning PF. A bug here was historically possible (CVE-class: VF on PF0 egresses out PF1 with PF0's source-port tag, observable by switch). MPESW with `reg_c0[28..31]` originating-PF tag closes this structurally.
- **FIB event allowlist** — multipath mode only processes FIB events that name *both* PFs of this LAG bundle in the nexthop list; foreign routes (those that name other devices) are dropped. Closes a path where a misconfigured route could trigger a v2p_map update that disables one port.
- **Devcom rendezvous timeout** — strict 5s timeout on cross-PF coordination; failures roll back to NONE on both PFs. Prevents a deadlocked PF from blocking the other indefinitely.
- **Mode-prereq strict** — refuse LAG activation if either PF has a sub-function or VF currently in `IN_USE` state and the requested mode would change its egress port unexpectedly. Closes "user pulled the rug out from under in-flight traffic" class.
- **Per-PF event-notifier registration ordered** — port_change_nb registered *after* prereq check passes and *before* `MODIFY_LAG` is issued. Avoids a window where a port-state event arrives but no LAG state exists to consume it (NULL-deref class).
- **Demux-rule install rate-limit** — `mlx5_lag_demux_rule_add` rate-limited per-PF; a hostile orchestrator scripting per-vport demux additions can't OOM the FDB.
- **Definer header-field allowlist** — port-sel definer only allows hashing on a fixed allowlist of header fields (eth.dst, ip.src, ip.dst, l4.sport, l4.dport, IPv6 equivalents). Arbitrary header-field selection rejected.
- **MPESW mode requires both PFs trusted** — refuse MPESW activation if either PF has a tenant-untrusted state (e.g. a guest-VFIO'd VF that could push hostile FDB rules). Activation requires both PFs to be in fully-trusted-host posture.

Per-doc rationale: LAG creates a shared eSwitch surface across two PFs. In switchdev/MPESW mode, an FDB rule installed via one PF affects traffic on both. This expands the tenant-isolation perimeter — a bug that previously could only affect traffic on PF0 can now affect PF1's tenants too. The reinforcement above is layered: compile-time mode typing (`Lag<Mode>`) prevents calling-wrong-mode-op bugs; HW source-port metadata in MPESW closes the cross-PF tenant-mix class structurally; FIB-event allowlist and devcom-timeout close orchestration-side attack surface; definer-field allowlist closes a future-feature-creep attack surface. LAG has had multiple bonds-related bugs over the years (lockdep splats from inverted lock order; UAF on rapid bond create/delete cycles); the typed state and strict lock-group enforcement here close the class.

## Open Questions

- [ ] Q1: Should Rookery support all five modes from day one, or start with NONE + ROCE (the most-deployed pair) and add SRIOV/MULTIPATH/MPESW iteratively? User-impact: dropping any of them blocks a deployment shape. Recommendation: support all five from day one — the modes are mutually exclusive at runtime, so the surface added per-mode is small.
- [ ] Q2: Devcom (cross-PF coordination) — upstream uses a homegrown mechanism. Rookery could use a simple typed `tokio` channel between the two `MlxCoreDev` actors. Trade-off is divergence from the cross-component devcom usage (esw + ipsec + fdb also use devcom).
- [ ] Q3: Multipath mode + bond mode interaction — they are mutually exclusive but the FIB notifier and the bond notifier can race at startup. Need explicit serialization design (lock or message-passing).
- [ ] Q4: MPESW + SR-IOV — both touch the eswitch. Combining them (MPESW shared FDB with VFs on both PFs) is the most complex configuration. Need to model the interaction explicitly.
- [ ] Q5: Hot-add a second PF (e.g. via hot-plug) into an existing single-PF deployment — should the LAG bundle auto-form, or require explicit user action? Upstream auto-forms when both PFs are netdev-bonded; same semantics planned for Rookery.

## Verification

- **Kani SAFETY**: prove `LagBundle::activate` cannot leave the bundle in a partially-activated state (either both PFs see the mode change, or neither does). Prove `Lag::deactivate` cannot underflow the bundle refcount.
- **TLA+**: model the cross-PF devcom rendezvous; check liveness (every activate request either succeeds on both PFs, fails on both PFs with NONE-state-restored, or times out cleanly on both); check safety (no path leaves PF0 in mode X while PF1 is in mode Y).
- **Verus**: functional spec of `PortSel::modify_buckets(map)` — for every input v2p_map satisfying the per-PF cap (bucket count ≤ MLX5_LAG_MAX_HASH_BUCKETS, port ids in {0, num_lag_ports}), produces a sequence of MODIFY_LAG FW cmds whose composition is equivalent to a single atomic write of `map`.
- **Kani+Verus**: invariant that for every demux rule in `lag_demux_rules` xarray, there exists a vport in the eswitch with that vport_index; and conversely no leaked demux rules at LAG deactivation.
- **Integration**: dual-100G LAG sustained at 200 Gb/s for 1h with no fail-over; bond fail-back test (down/up one port 1000 times in 60s, verify zero packet loss across the transitions); MPESW + SR-IOV stress (16 VFs spread across 2 PFs in MPESW mode, OVS-offload rules, iperf3 mesh test).
- **Fuzz**: bond_tracker fuzzer (mutate netdev_lag_tx_type, hash_type, port_state combinations) against `do_bond` — should always converge to a valid mode-flip or refuse cleanly.
- **Penetration**: with two unprivileged KVM guests holding VFs from PF0 and PF1 respectively, in SRIOV-LAG mode, attempt to read each other's traffic (via tcpdump on the VF, via TC rule installation via a compromised orchestrator); must not succeed.

## Out of Scope

- mlx5_core firmware-cmd interface, EQ/CQ, health — covered in `mlx5-core.md`
- mlx5 eSwitch (consumer of LAG-shared FDB in MPESW mode) — covered in `eswitch.md`
- mlx5 Sub-Functions (orthogonal to LAG; SFs on either PF participate normally) — covered in `mlx5-sf.md`
- mlx5e netdev (per-PF netdev that participates in the bond) — covered in `mellanox-mlx5-en.md`
- Linux bond/team driver internals (the event source) — covered in `drivers/net/bonding/` and `drivers/net/team/` Tier-3s
- FIB notifier framework — covered in `net/ipv4/fib_*` Tier-3
- BlueField-3 4-port LAG (when num_lag_ports = 4) — same code paths, validated in scale tests; no separate doc
