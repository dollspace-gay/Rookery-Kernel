# Tier-3: net/xfrm/user — NETLINK_XFRM userspace control interface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_user.c
  - net/xfrm/xfrm_compat.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for the NETLINK_XFRM userspace control interface — the modern (post-PF_KEYv2) wire ABI used by strongSwan, libreswan, iked, NetworkManager-strongswan, and `ip xfrm`. Implements per-message-type parsers + builders for the full XFRM_MSG_* set, the per-netns notification socket (asynchronous events: ACQUIRE / EXPIRE / NEWAE / MIGRATE / MAPPING), and the multi-part dump protocol for GETSA / GETPOLICY / GETSPDINFO walks. Cooperates with `xfrm_state.c`, `xfrm_policy.c`, and `xfrm_replay.c` to perform the actual state mutation; this Tier-3 covers only the wire-format + per-message dispatch.

The "front door" of the IPSec stack: every IKE-driven SA install and policy update lands here. Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md`, `net/xfrm/policy.md`, `net/xfrm/replay.md`, `net/xfrm/af-key.md` (PF_KEYv2 alternative).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| NETLINK_XFRM family register, per-XFRM_MSG_* parser + builder, dump protocol, per-netns notify socket | `net/xfrm/xfrm_user.c` |
| 32-bit-userspace compat shims | `net/xfrm/xfrm_compat.c` (DEFERRED for v0 per `net/xfrm/00-overview.md` REQ-O3) |
| Public API | `include/net/xfrm.h` |
| UAPI | `include/uapi/linux/xfrm.h` |

## Compatibility contract

### NETLINK_XFRM message types (per `include/uapi/linux/xfrm.h`)

| `XFRM_MSG_*` | Direction | Operation |
|---|---|---|
| `NEWSA` | user → kernel | Install new SA |
| `DELSA` | user → kernel | Delete SA |
| `GETSA` | user → kernel | Lookup or dump SA(s) |
| `UPDSA` | user → kernel | Update SA in place |
| `NEWPOLICY` | user → kernel | Install new policy |
| `DELPOLICY` | user → kernel | Delete policy |
| `GETPOLICY` | user → kernel | Lookup or dump policy/policies |
| `UPDPOLICY` | user → kernel | Update policy in place |
| `FLUSHSA` | user → kernel | Bulk delete SAs by filter |
| `FLUSHPOLICY` | user → kernel | Bulk delete policies by filter |
| `ALLOCSPI` | user → kernel | Allocate SPI for an in-progress SA install |
| `EXPIRE` | kernel → user | Soft/hard lifetime threshold reached |
| `ACQUIRE` | kernel → user | Policy needs SA; daemon should IKE up |
| `REPORT` | kernel → user | Policy denied a packet; report to daemon |
| `MIGRATE` | both | Mobile IPv6 SA migration |
| `MAPPING` | kernel → user | NAT mapping change detected |
| `NEWAE` | both | Anti-replay event (window threshold or timer) |
| `GETAE` | user → kernel | Query AE state |
| `SETSPDINFO` | user → kernel | Set per-netns SPD parameters |
| `GETSPDINFO` | user → kernel | Get per-netns SPD info (counts, hash sizes) |
| `SETSADINFO` | user → kernel | Set per-netns SAD parameters |
| `GETSADINFO` | user → kernel | Get per-netns SAD info |
| `NEWSPDHTHRESH` | user → kernel | Set SPD hashing threshold |
| `MAPPING` | kernel → user | NAT mapping change detected |

Wire format byte-identical so existing IKE daemons work unchanged.

### NLA attributes

Per-message NLA attributes per `XFRMA_*` enum:
- `XFRMA_ALG_AUTH`, `XFRMA_ALG_CRYPT`, `XFRMA_ALG_AEAD`, `XFRMA_ALG_COMP`
- `XFRMA_ALG_AUTH_TRUNC` (with truncation len)
- `XFRMA_ENCAP` (NAT-T)
- `XFRMA_TMPL` (policy templates)
- `XFRMA_SA` / `XFRMA_POLICY`
- `XFRMA_SEC_CTX` (LSM context)
- `XFRMA_LTIME_VAL` (lifetime current)
- `XFRMA_REPLAY_VAL`, `XFRMA_REPLAY_THRESH`, `XFRMA_REPLAY_ESN_VAL`, `XFRMA_REPLAY_THRESH_BYTES`
- `XFRMA_ETIMER_THRESH`
- `XFRMA_SRCADDR`, `XFRMA_COADDR` (for migrate)
- `XFRMA_LASTUSED`
- `XFRMA_POLICY_TYPE` (main vs sub)
- `XFRMA_MIGRATE`
- `XFRMA_KMADDRESS`
- `XFRMA_ADDRESS_FILTER`
- `XFRMA_PAD`
- `XFRMA_OFFLOAD_DEV` (HW offload)
- `XFRMA_SET_MARK`, `XFRMA_SET_MARK_MASK`
- `XFRMA_IF_ID` (XFRMi)
- `XFRMA_MTIMER_THRESH`
- `XFRMA_SA_DIR`
- `XFRMA_NAT_KEEPALIVE_INTERVAL`
- `XFRMA_SA_PCPU` (per-CPU SA)
- `XFRMA_IPTFS_*` (IPTFS configuration)

Each NLA: byte-identical layout per upstream's `xfrma_policy[]` validation table.

### Multicast notification groups

Per `include/uapi/linux/xfrm.h`:
- `XFRMNLGRP_ACQUIRE` (1)
- `XFRMNLGRP_EXPIRE` (2)
- `XFRMNLGRP_SA` (3)
- `XFRMNLGRP_POLICY` (4)
- `XFRMNLGRP_AEVENTS` (5)
- `XFRMNLGRP_REPORT` (6)
- `XFRMNLGRP_MIGRATE` (7)
- `XFRMNLGRP_MAPPING` (8)

Daemons subscribe via `setsockopt(SOL_NETLINK, NETLINK_ADD_MEMBERSHIP, ...)`. Identical multicast groups.

### Dump protocol (GETSA / GETPOLICY / GETSPDINFO)

Multi-part: kernel splits dump across multiple netlink replies with `NLM_F_MULTI` flag; final reply has `NLMSG_DONE`. Walker cursor tracks per-dump state (cross-ref `xfrm_state_walk` in `net/xfrm/state.md`). Identical multi-part protocol.

### LSM context attribute (`XFRMA_SEC_CTX`)

When LSM (SELinux/Smack/AppArmor) labels are in use, NEWSA/NEWPOLICY can carry `XFRMA_SEC_CTX` with sec-context string. Kernel resolves to LSM's internal sid via `security_xfrm_state_alloc_acquire` etc. Identical UAPI.

## Requirements

- REQ-1: NETLINK_XFRM family registered per upstream's `xfrm_user_proto`; identical max NLA + per-message NLA validation policies.
- REQ-2: All XFRM_MSG_* message types (per the table above) parsed + built per upstream's `xfrm_dispatch[]`; wire format byte-identical.
- REQ-3: Per-message NLA attribute schemas (per `XFRMA_*`) enforced via netlink validation policy (`xfrma_policy[]`); identical layout.
- REQ-4: Multicast notification groups (`XFRMNLGRP_*`) registered; daemons can subscribe; ACQUIRE/EXPIRE/NEWAE/MIGRATE/MAPPING events emitted to subscribers.
- REQ-5: Dump protocol: GETSA / GETPOLICY / GETSPDINFO multi-part with `NLM_F_MULTI` + `NLMSG_DONE`; walker-based.
- REQ-6: ACQUIRE emission: from `net/xfrm/policy.md` REQ-9 path; per-netns notify socket multicast.
- REQ-7: EXPIRE emission: from `net/xfrm/state.md` REQ-7/8; soft + hard variants; LSM-relevant attributes preserved.
- REQ-8: NEWAE emission: from `net/xfrm/replay.md` REQ-7; threshold-driven + time-driven.
- REQ-9: LSM context attribute (`XFRMA_SEC_CTX`): NEWSA/NEWPOLICY carry sec-context; kernel calls LSM hooks for resolution.
- REQ-10: HW-offload attribute (`XFRMA_OFFLOAD_DEV`): NEWSA can request offload to specified netdev; kernel calls `xfrm_dev_offload` (cross-ref future `net/xfrm/device.md`).
- REQ-11: XFRMi attribute (`XFRMA_IF_ID`): NEWSA/NEWPOLICY carry XFRMi if_id; kernel routes RX/TX through XFRMi virtual netdev (cross-ref future `net/xfrm/interface.md`).
- REQ-12: 32-bit userspace compat (`xfrm_compat.c`) DEFERRED for v0 per `net/xfrm/00-overview.md` REQ-O3.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `ip xfrm state add ...` test: NLA-encoded NEWSA byte-identical to upstream's; kernel parse succeeds; SA visible in dump. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: `ip xfrm policy add ...` test: NEWPOLICY byte-identical; policy visible in dump. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-3: `ip xfrm monitor` test: subscribe to all multicast groups; trigger ACQUIRE/EXPIRE/NEWAE/MIGRATE/MAPPING via test fixture; daemon receives events with byte-identical wire format. (covers REQ-4, REQ-6, REQ-7, REQ-8)
- [ ] AC-4: GETSA dump test: install 100 SAs; `XFRM_MSG_GETSA` dump returns all 100 across multiple netlink-multipart messages with `NLM_F_MULTI` + final `NLMSG_DONE`. (covers REQ-5)
- [ ] AC-5: ACQUIRE event test: install policy without matching SA; send packet → daemon receives `XFRM_MSG_ACQUIRE` with byte-identical attributes. (covers REQ-6)
- [ ] AC-6: EXPIRE event test: SA with `lifetime soft byte 1024` → after 1024-byte traffic → daemon receives `XFRM_MSG_EXPIRE` with `hard=0`. (covers REQ-7)
- [ ] AC-7: NEWAE event test: SA with `XFRMA_REPLAY_THRESH=10` → after 11 packets → daemon receives `XFRM_MSG_NEWAE` with current replay state. (covers REQ-8)
- [ ] AC-8: SELinux test: NEWSA with `XFRMA_SEC_CTX="system_u:object_r:ipsec_spd_t:s0"` → policy installed with LSM sid; subsequent flow-classify uses LSM hook. (covers REQ-9)
- [ ] AC-9: HW-offload test (on supporting NIC): NEWSA with `XFRMA_OFFLOAD_DEV=eth0` → SA installed with offload flag; HW path engaged. (covers REQ-10)
- [ ] AC-10: XFRMi test: `XFRMA_IF_ID=0x42` on NEWSA + matching policy → packets via XFRMi virtual netdev with that if_id are correctly classified. (covers REQ-11)
- [ ] AC-11: 32-bit userspace test on x86_64 kernel: explicitly skipped per REQ-12 (deferred for v0). (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::user::XfrmUserNetlink` — NETLINK_XFRM family root
- `kernel::net::xfrm::user::dispatch::Dispatch` — per-XFRM_MSG_* dispatch table
- `kernel::net::xfrm::user::nla::NlaParser` — per-attribute NLA parser (using bound-checked accessors)
- `kernel::net::xfrm::user::nla::NlaBuilder` — reply NLA builder
- `kernel::net::xfrm::user::msg::Newsa`, `Delsa`, `Getsa`, `Updsa` — per-MSG handlers
- `kernel::net::xfrm::user::msg::Newpolicy`, `Delpolicy`, `Getpolicy`, `Updpolicy`
- `kernel::net::xfrm::user::msg::Flushsa`, `Flushpolicy`
- `kernel::net::xfrm::user::msg::Allocspi`, `Migrate`, `Newae`, `Getae`
- `kernel::net::xfrm::user::dump::Dump` — multi-part dump protocol
- `kernel::net::xfrm::user::notify::Notify` — multicast event emitter
- `kernel::net::xfrm::user::lsm::LsmCtx` — XFRMA_SEC_CTX handler

### Locking and concurrency

- **NETLINK family per-socket mutex**: serializes per-socket message processing (inherited from netlink core)
- **Per-netns `xfrm_state_lock` / `xfrm_policy_lock`**: held during state/policy mutations (cross-ref `state.md` / `policy.md`)
- **Per-netns notify socket**: lockless multicast emission; recipients subscribe via NETLINK_ADD_MEMBERSHIP

### Error handling

- `Err(EINVAL)` — bad NLA / unsupported XFRM_MSG_* / bad attribute
- `Err(EEXIST)` — NEWSA / NEWPOLICY for duplicate
- `Err(ENOENT)` — DEL / GET / UPD for non-existent
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ENOMEM)` — alloc fail
- `Err(EAFNOSUPPORT)` — unsupported address family

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-XFRM_MSG_* dispatch table walk (no NULL fn-ptr deref) | `kani::proofs::net::xfrm::user::dispatch_safety` |
| NLA parser per-attribute bounds (`nla_*` accessors) | `kani::proofs::net::xfrm::user::nla_parse_safety` |
| Multi-part dump cursor (no out-of-bounds walker advance) | `kani::proofs::net::xfrm::user::dump_safety` |
| Multicast event emit (skb alloc + nlmsg_multicast under recipient lock) | `kani::proofs::net::xfrm::user::notify_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/xfrm_state_machine.tla` from `state.md` for state-mutation atomicity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Dispatch table | every entry has non-NULL handler + correct max-NLA-policy reference | `kani::proofs::net::xfrm::user::dispatch_invariants` |
| Per-message NLA validation | NLA size + alignment match per upstream's xfrma_policy[] table | `kani::proofs::net::xfrm::user::nla_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Wire-format compatibility theorem** via Verus — proves: serialized output for any valid input matches the upstream `xfrm_user.c` byte-for-byte.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-XFRM_MSG_* dispatch table + per-attribute NLA validation policy `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-emit dump skbs cleared (may carry SA attribute material) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors throughout
- **SIZE_OVERFLOW**: NLA-length arithmetic uses checked operators (CVE class: CVE-2017-7184 NLA-overflow)
- **KERNEXEC**: dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all XFRM_MSG_* mutations (per upstream); GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_xfrm_*` for SA/policy alloc/delete (cross-ref upstream LSM contract).
- Useful default GR-RBAC policy: deny NETLINK_XFRM mutations outside gradm-marked `ipsec_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — NETLINK_XFRM wire format is exhaustively specified by upstream; v0 explicitly omits 32-bit compat per REQ-12)

## Out of Scope

- 32-bit userspace compat (`xfrm_compat.c`) — DEFERRED for v0 (covered as `net/xfrm/compat.md` Tier-3 stub, marked OUT-OF-SCOPE for v0)
- PF_KEYv2 alternative interface (cross-ref `net/xfrm/af-key.md`)
- Underlying SA/policy mutation logic (cross-ref `net/xfrm/state.md`, `net/xfrm/policy.md`, `net/xfrm/replay.md`)
- 32-bit-only paths
- Implementation code
