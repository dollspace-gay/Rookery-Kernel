---
title: "Tier-3: net/netlink/genetlink.c — Generic netlink (genetlink) family multiplexer"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Generic Netlink (genetlink) is a multiplexer over `NETLINK_GENERIC` (protocol 16) supporting per-family registration of "sub-protocols" (ethtool / nl80211 / devlink / taskstats / wireguard / nfsdv4 / etc.). Per-family ID + per-cmd policy + per-cmd doit/dumpit callback. Per-family multicast groups + per-policy per-attr validation. Replaces "allocate-a-NETLINK-protocol-number" pattern. Critical for: modern Linux device control (ethtool/devlink/wireless/wireguard/etc.).

This Tier-3 covers `genetlink.c` (~1996 lines).

### Acceptance Criteria

- [ ] AC-1: `genl_register_family(&taskstats_family)` succeeds; family-id ∈ [16..1023].
- [ ] AC-2: Userspace CTRL_CMD_GETFAMILY by name "TASKSTATS" → returns family-id.
- [ ] AC-3: Per-cmd doit invoked for non-DUMP msg.
- [ ] AC-4: Per-cmd NLM_F_DUMP → dumpit + start + done sequence.
- [ ] AC-5: Per-cmd GENL_ADMIN_PERM without CAP_NET_ADMIN → -EPERM.
- [ ] AC-6: Per-attr policy mismatch → -EINVAL with extack populated.
- [ ] AC-7: Per-mcast group: subscribe via NETLINK_ADD_MEMBERSHIP, broadcast received.
- [ ] AC-8: CTRL_CMD_NEWFAMILY broadcast on register; CTRL_CMD_DELFAMILY on unregister.
- [ ] AC-9: Per-family unregister: subsequent CTRL_CMD_GETFAMILY → -ENOENT.
- [ ] AC-10: Per-namespace: family.netnsok = 1; nsA + nsB dispatch independently.
- [ ] AC-11: Concurrent doit: parallel_ops=1 allows; parallel_ops=0 serializes.

### Architecture

Per-family state:

```
struct GenlFamily {
  id: u16,                                       // assigned at register
  hdrsize: u32,
  name: String,                                  // ≤ 16
  version: u32,
  maxattr: u32,
  netnsok: bool,
  parallel_ops: bool,
  policy: &'static [NlaPolicy],
  ops: &'static [GenlOps],
  n_ops: usize,
  small_ops: &'static [GenlSmallOps],            // smaller ops shape
  n_small_ops: usize,
  split_ops: &'static [GenlSplitOps],
  n_split_ops: usize,
  mcgrps: &'static [GenlMcastGroup],
  n_mcgrps: usize,
  mcgrp_offset: u32,                             // base group-id
  pre_doit: Option<fn(...)>,
  post_doit: Option<fn(...)>,
  mcast_bind: Option<fn(net, group)>,
  mcast_unbind: Option<fn(net, group)>,
  module: ModuleRef,
}
```

Per-cmd handler:

```
struct GenlOps {
  cmd: u8,
  internal_flags: u8,
  flags: GenlOpFlags,
  validate: u8,
  policy: Option<&'static [NlaPolicy]>,
  doit: Option<fn(&mut Skb, &GenlInfo) -> Result<()>>,
  start: Option<fn(&mut NetlinkCallback) -> Result<()>>,
  dumpit: Option<fn(&mut Skb, &mut NetlinkCallback) -> Result<usize>>,
  done: Option<fn(&mut NetlinkCallback) -> Result<()>>,
}
```

Per-call info:

```
struct GenlInfo {
  snd_seq: u32,
  snd_portid: u32,
  family: &GenlFamily,
  nlhdr: &Nlmsghdr,
  genlhdr: &Genlmsghdr,
  attrs: &[Option<&Nla>],
  extack: &mut NlExtAck,
}
```

`Genl::register_family`:
1. Validate name + maxattr + ops[].
2. Allocate family.id from genl_family_idr (≥ 16).
3. Register kernel-side input via netlink_register_notifier on first family.
4. Per-mcgrp: assign per-group-id from global mc_group_idr.
5. Broadcast CTRL_CMD_NEWFAMILY to subscribers.

`Genl::dispatch` (input handler):
1. Per-skb parse nlmsghdr + genlmsghdr.
2. Lookup family by nlh.nlmsg_type.
3. Lookup ops by genlh.cmd.
4. CAP-check per-ops.flags.
5. Acquire family.mutex if !parallel_ops.
6. nlmsg_parse(nlh, family.hdrsize, attrs, family.maxattr, family.policy, extack).
7. Build GenlInfo.
8. Call family.pre_doit(ops, skb, info).
9. If NLM_F_DUMP: netlink_dump_start with ops.start/dumpit/done.
10. Else: ops.doit(skb, info).
11. Call family.post_doit(ops, skb, info).
12. Release family.mutex.

`Genl::msg_put`:
1. Reserve nlmsghdr + genlmsghdr in skb.
2. Set genlh.cmd, genlh.version, genlh.reserved=0.
3. Return data ptr for caller to fill.

`Genl::multicast_netns`:
1. Resolve global group-id = family.mcgrp_offset + group_idx.
2. netlink_broadcast(genl_kernel_sock, skb, portid, group, allocation) within net.

### Out of Scope

- AF_NETLINK core (covered in `af_netlink.md` Tier-3)
- Per-family contents (ethtool, devlink, nl80211, taskstats — covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct genl_family` | per-family registration | `GenlFamily` |
| `struct genl_ops` | per-cmd handler | `GenlOps` |
| `struct genl_split_ops` | per-cmd split (do/dump) | `GenlSplitOps` |
| `struct genl_multicast_group` | per-mcast-group | `GenlMcastGroup` |
| `struct genl_info` | per-call info | `GenlInfo` |
| `genl_register_family()` | per-family register | `Genl::register_family` |
| `genl_unregister_family()` | per-family unregister | `Genl::unregister_family` |
| `genlmsg_parse()` | per-msg parse | `Genl::parse` |
| `genlmsg_put()` | per-reply build | `Genl::msg_put` |
| `genlmsg_unicast()` | per-reply unicast | `Genl::unicast` |
| `genlmsg_multicast_netns()` | per-mcast in netns | `Genl::multicast_netns` |
| `genl_notify()` | per-event notify | `Genl::notify` |
| `genlmsg_data()` | per-msg data ptr | `Genl::msg_data` |
| `GENL_DONE` / `GENL_REPLY` | per-cmd reply flags | shared |
| `GENL_CMD_CAP_DO` / `GENL_CMD_CAP_DUMP` | per-cmd capability | shared |

### compatibility contract

REQ-1: Per-family registration:
- `genl_register_family(family)` registers per-family at runtime.
- Per-family fields: `name` (string ≤ 16 chars), `version`, `hdrsize`, `maxattr`, `policy`, `ops[]`, `n_ops`, `mcgrps[]`, `n_mcgrps`, `module`, `parallel_ops`, `netnsok`, `pre_doit`, `post_doit`, `mcast_bind`, `mcast_unbind`.
- Per-family allocated id ∈ [GENL_MIN_ID..GENL_MAX_ID] (16..1023).
- GENL_ID_CTRL=16 (controller, ctrl_family).

REQ-2: Per-cmd dispatch:
- `genl_ops`: `cmd` + `doit` + optional `dumpit` + `policy` + `start` + `done` + `flags`.
- Per-cmd flags: GENL_ADMIN_PERM (CAP_NET_ADMIN), GENL_UNS_ADMIN_PERM (CAP_NET_ADMIN in user-ns), GENL_CMD_CAP_DO, GENL_CMD_CAP_DUMP, GENL_CMD_CAP_HASPOL.
- Per-cmd: doit invoked for non-DUMP; dumpit + start + done for DUMP.

REQ-3: Per-msg dispatch flow:
- AF_NETLINK / NETLINK_GENERIC sock receives nlmsg with family-id (in nlmsg_type) + cmd (in genlmsghdr.cmd).
- Per-family lookup in genl_family_idr.
- Per-cmd lookup in family.ops[].
- Per-flag CAP-check via netlink_capable / netlink_ns_capable.
- Per-cmd policy validation via nlmsg_parse.
- Per-cmd doit invoked.

REQ-4: Per-multicast group:
- Per-family registers `mcgrps[]`.
- Per-mcgrp gets a kernel-wide group-id (after concatenation across families).
- `genlmsg_multicast_netns(family, skb, portid, group_idx, allocation)` broadcasts.
- Userspace subscribes via NETLINK_ADD_MEMBERSHIP sockopt with group-id.

REQ-5: Per-controller (CTRL) family:
- `GENL_ID_CTRL = 16`, family-name="nlctrl".
- Per-cmd CTRL_CMD_GETFAMILY: lookup family by name; return per-family info + per-mcast group ids.
- Per-cmd CTRL_CMD_NEWFAMILY / DELFAMILY broadcast on per-family register/unregister.
- Per-userspace must always GETFAMILY first.

REQ-6: Per-pre/post-doit hooks:
- `family.pre_doit(ops, skb, info)` invoked before per-cmd doit.
- `family.post_doit(ops, skb, info)` after.
- Per-pre-doit can lock per-family mutex (parallel_ops=false).

REQ-7: Per-policy validation:
- `family.policy[]` per-attribute.
- Per-attr: `NLA_U8 / U16 / U32 / U64 / STRING / FLAG / NESTED / BINARY / S8...`.
- Per-attr range/min-len/max-len.
- Per-msg validated before doit.

REQ-8: Per-attr put helpers:
- `nla_put_u32(skb, type, value)`.
- `nla_put_string(skb, type, str)`.
- Per-attr aligned to 4-byte.

REQ-9: Per-version + extack:
- Per-family advertises version.
- Per-cmd extack populates via NL_SET_ERR_MSG / NL_SET_BAD_ATTR.

REQ-10: Per-namespace:
- Per-family.netnsok = 1 → per-namespace dispatch.
- Per-mcast multicast scoped to per-net.

REQ-11: Per-cmd parallel_ops:
- family.parallel_ops = 1: no per-family mutex; concurrent doit.
- family.parallel_ops = 0: per-family mutex held over doit.

REQ-12: Per-cmd validation flags:
- `GENL_DONT_VALIDATE_STRICT` / `GENL_DONT_VALIDATE_DUMP` / `GENL_DONT_VALIDATE_DUMP_STRICT`.
- Default: strict validation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `family_id_in_range` | INVARIANT | per-registered family.id ∈ [GENL_MIN_ID, GENL_MAX_ID]. |
| `family_id_unique` | INVARIANT | per-IDR no two families share id. |
| `mcgrp_offset_disjoint` | INVARIANT | per-family.mcgrp_offset ranges disjoint across families. |
| `parallel_ops_implies_no_mutex_held` | INVARIANT | family.parallel_ops ⟹ doit invoked without family.mutex. |
| `policy_validated_before_doit` | INVARIANT | per-cmd doit invoked only after policy-validated. |

### Layer 2: TLA+

`net/netlink/genetlink.tla`:
- Per-family register/unregister + per-cmd dispatch + per-mcgrp broadcast.
- Properties:
  - `safety_no_dispatch_to_unregistered` — per-cmd dispatch ⟹ family registered.
  - `safety_capability_required_for_perm_cmd` — per-cmd-with-GENL_ADMIN_PERM dispatch ⟹ caller has CAP_NET_ADMIN.
  - `liveness_dump_completes` — per-NLM_F_DUMP eventually done.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Genl::register_family` post: family.id ≥ 16; family in idr | `Genl::register_family` |
| `Genl::dispatch` pre: family registered; ops.cmd == genlh.cmd | `Genl::dispatch` |
| `Genl::dispatch` post: per-cmd doit called iff CAP-check + policy-pass | `Genl::dispatch` |
| `Genl::multicast_netns` post: per-net subscribers received clone | `Genl::multicast_netns` |

### Layer 4: Verus/Creusot functional

`Per-CTRL_CMD_GETFAMILY by name → returns family-id + per-mcast group-ids` semantic equivalence: per-userspace request matches Linux genetlink ABI (ethtool/devlink/wireguard interop).

### hardening

(Inherits row-1 features from `net/netlink/af_netlink.md` § Hardening.)

Genetlink-specific reinforcement:

- **Per-cmd CAP-check enforced** — defense against unprivileged process invoking GENL_ADMIN_PERM cmd.
- **Per-policy validation pre-doit** — defense against malformed-attr crashing per-family handler.
- **Per-family ID-allocation atomic** — defense against concurrent register collision.
- **Per-mcgrp offset disjoint** — defense against cross-family multicast leak.
- **Per-extack populated on validation fail** — defense against silent error.
- **Per-family parallel_ops gating mutex** — defense against doit-rentry corrupting per-family state.
- **Per-pre-doit invoked before doit** — defense against per-cmd missing rate-limit / setup.
- **Per-namespace family.netnsok required** — defense against cross-ns leak.
- **Per-CTRL_CMD_NEWFAMILY broadcast scoped to per-net** — defense against subscription-scope bypass.
- **Per-cmd version checked** — defense against userspace mismatched-protocol.

