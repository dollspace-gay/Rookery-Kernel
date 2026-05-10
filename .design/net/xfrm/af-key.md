# Tier-3: net/xfrm/af-key — PF_KEYv2 legacy IPSec control interface (RFC 2367)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/key/af_key.c
  - include/uapi/linux/pfkeyv2.h
  - include/uapi/linux/ipsec.h
-->

## Summary
Tier-3 design for PF_KEYv2 (RFC 2367) — the legacy POSIX-standard IPSec control socket family. Predates NETLINK_XFRM; still supported because IKEv1 daemons (racoon from ipsec-tools, isakmpd, ipsec-tools-derivatives in some embedded stacks) and the `setkey` command-line tool from ipsec-tools require it. Sets up the same kernel SADB / SPD that NETLINK_XFRM does — the two interfaces share the underlying `xfrm_state` / `xfrm_policy` storage; PF_KEYv2 messages are translated through `pfkey_*` adapter functions.

`socket(PF_KEY, SOCK_RAW, PF_KEY_V2)` opens a control socket; sendmsg/recvmsg carries SADB messages with TLV (extension) payloads.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/user.md` (modern alternative), `net/xfrm/state.md` (underlying SADB), `net/xfrm/policy.md` (underlying SPD), `net/xfrm/algo.md` (PF_KEYv2-supported algorithm subset).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| PF_KEYv2 socket family + per-message handler + SADB↔xfrm_state translation | `net/key/af_key.c` |
| UAPI: SADB messages, TLV ext types, sadb_alg, sadb_sa | `include/uapi/linux/pfkeyv2.h` |
| UAPI: IPsec policy types, levels | `include/uapi/linux/ipsec.h` |

## Compatibility contract

### `socket(PF_KEY, SOCK_RAW, PF_KEY_V2)`

Identical creation semantics. CAP_NET_ADMIN required. Per-netns scope.

### SADB message types (`sadb_msg.sadb_msg_type`)

| Name | Constant | Direction |
|---|---|---|
| `SADB_RESERVED` | 0 | — |
| `SADB_GETSPI` | 1 | user → kernel (allocate SPI) |
| `SADB_UPDATE` | 2 | user → kernel (complete SPI/install SA) |
| `SADB_ADD` | 3 | user → kernel (install fresh SA) |
| `SADB_DELETE` | 4 | user → kernel (delete SA) |
| `SADB_GET` | 5 | user → kernel (lookup SA) |
| `SADB_ACQUIRE` | 6 | kernel → user (policy needs SA) |
| `SADB_REGISTER` | 7 | user → kernel (subscribe to algo capabilities + ACQUIRE notifications) |
| `SADB_EXPIRE` | 8 | kernel → user (lifetime threshold) |
| `SADB_FLUSH` | 9 | user → kernel (flush all SAs) |
| `SADB_DUMP` | 10 | user → kernel (dump SADB) |
| `SADB_X_PROMISC` | 11 | user → kernel (promiscuous: receive all PF_KEY traffic) |
| `SADB_X_PCHANGE` | 12 | bidir (policy change notification) |
| `SADB_X_SPDUPDATE` | 13 | user → kernel (policy update) |
| `SADB_X_SPDADD` | 14 | user → kernel (policy add) |
| `SADB_X_SPDDELETE` | 15 | user → kernel (policy delete) |
| `SADB_X_SPDGET` | 16 | user → kernel (policy get) |
| `SADB_X_SPDACQUIRE` | 17 | bidir |
| `SADB_X_SPDDUMP` | 18 | user → kernel (dump SPD) |
| `SADB_X_SPDFLUSH` | 19 | user → kernel (flush SPD) |
| `SADB_X_SPDSETIDX` | 20 | user → kernel (policy set by index) |
| `SADB_X_SPDEXPIRE` | 21 | kernel → user |
| `SADB_X_SPDDELETE2` | 22 | user → kernel (policy delete by index) |
| `SADB_X_NAT_T_NEW_MAPPING` | 23 | kernel → user (NAT-T mapping change) |
| `SADB_X_MIGRATE` | 24 | bidir (Mobile IPv6 SA migrate) |

Wire format byte-identical so racoon + setkey + ipsec-tools work unchanged.

### SADB extension TLV types (`sadb_ext.sadb_ext_type`)

| Constant | Type |
|---|---|
| `SADB_EXT_RESERVED` | 0 |
| `SADB_EXT_SA` | 1 (struct sadb_sa) |
| `SADB_EXT_LIFETIME_CURRENT` | 2 |
| `SADB_EXT_LIFETIME_HARD` | 3 |
| `SADB_EXT_LIFETIME_SOFT` | 4 |
| `SADB_EXT_ADDRESS_SRC` | 5 |
| `SADB_EXT_ADDRESS_DST` | 6 |
| `SADB_EXT_ADDRESS_PROXY` | 7 |
| `SADB_EXT_KEY_AUTH` | 8 |
| `SADB_EXT_KEY_ENCRYPT` | 9 |
| `SADB_EXT_IDENTITY_SRC` | 10 |
| `SADB_EXT_IDENTITY_DST` | 11 |
| `SADB_EXT_SENSITIVITY` | 12 |
| `SADB_EXT_PROPOSAL` | 13 |
| `SADB_EXT_SUPPORTED_AUTH` | 14 |
| `SADB_EXT_SUPPORTED_ENCRYPT` | 15 |
| `SADB_EXT_SPIRANGE` | 16 |
| `SADB_X_EXT_KMPRIVATE` | 17 |
| `SADB_X_EXT_POLICY` | 18 |
| `SADB_X_EXT_SA2` | 19 (extra SA fields) |
| `SADB_X_EXT_NAT_T_TYPE/SPORT/DPORT/OA/OAI/OAR/FRAG` | 20–25 |
| `SADB_X_EXT_NAT_T_FRAG` | 26 |
| `SADB_X_EXT_SEC_CTX` | 27 (LSM context) |
| `SADB_X_EXT_KMADDRESS` | 28 |
| `SADB_X_EXT_FILTER` | 29 (dump filter) |

Each TLV byte-identical layout.

### `struct sadb_msg` layout

```c
struct sadb_msg {
    __u8  sadb_msg_version;     /* PF_KEY_V2 */
    __u8  sadb_msg_type;
    __u8  sadb_msg_errno;
    __u8  sadb_msg_satype;      /* SADB_SATYPE_AH/ESP/IPCOMP */
    __u16 sadb_msg_len;          /* in 64-bit words */
    __u16 sadb_msg_reserved;
    __u32 sadb_msg_seq;
    __u32 sadb_msg_pid;
};
```

Layout-byte-identical (16 bytes).

### Multicast notification: `pfkey_broadcast`

Bound PF_KEYv2 sockets receive ACQUIRE / EXPIRE / SPDEXPIRE / NAT_T_NEW_MAPPING / X_MIGRATE notifications. Broadcast filtering: per-socket subscribed-message-types via `SADB_REGISTER` for SATYPE filter.

### `SADB_REGISTER` algo capabilities response

On `SADB_REGISTER` with `sadb_msg_satype=SADB_SATYPE_AH/ESP`, kernel replies with a SADB_REGISTER message carrying `SADB_EXT_SUPPORTED_AUTH` + `SADB_EXT_SUPPORTED_ENCRYPT` TLVs enumerating the `pfkey_supported = 1` algorithms from `xfrm_algo_list` (cross-ref `net/xfrm/algo.md`).

### Per-netns isolation

PF_KEYv2 sockets are per-netns; SADB/SPD mutations through PF_KEYv2 are netns-scoped (same as NETLINK_XFRM). Identical isolation.

### Translation to `xfrm_state` / `xfrm_policy`

PF_KEYv2 messages translate to underlying xfrm primitives via `pfkey_xfrm_state_add` / `pfkey_msg2xfrm_state` etc. The PF_KEYv2 path enforces `pfkey_supported = 1` algorithm gate; unknown / non-PF_KEYv2 algorithms return `EINVAL`.

## Requirements

- REQ-1: `socket(PF_KEY, SOCK_RAW, PF_KEY_V2)` family register; CAP_NET_ADMIN gated; per-netns.
- REQ-2: All SADB message types (per the table) parsed + emitted per RFC 2367 + Linux X-extensions.
- REQ-3: All SADB extension TLV types (per the table) byte-identical layout + parser/builder.
- REQ-4: `struct sadb_msg` byte-identical layout.
- REQ-5: Multicast notification: `pfkey_broadcast` to all listening sockets per per-socket SATYPE filter.
- REQ-6: `SADB_REGISTER` response carries SUPPORTED_AUTH + SUPPORTED_ENCRYPT enumerating PF_KEYv2-supported algorithms (`pfkey_supported = 1` from `net/xfrm/algo.md`).
- REQ-7: SADB↔xfrm translation: PF_KEYv2 messages map to xfrm_state / xfrm_policy mutations identically to NETLINK_XFRM.
- REQ-8: ACQUIRE emission: when policy needs SA, kernel emits SADB_ACQUIRE on per-netns PF_KEYv2 broadcast (alongside XFRM_MSG_ACQUIRE on NETLINK_XFRM).
- REQ-9: EXPIRE emission: SADB_EXPIRE soft + hard variants emitted at lifetime thresholds.
- REQ-10: NAT-T support: `SADB_X_EXT_NAT_T_TYPE/SPORT/DPORT/OA/FRAG` TLVs parsed + applied to xfrm_state's `encap` member.
- REQ-11: LSM context: `SADB_X_EXT_SEC_CTX` carries sec-context string; kernel calls LSM hooks for resolution.
- REQ-12: Per-netns isolation: PF_KEYv2 socket created in netns N can only mutate / observe netns-N SADB/SPD.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct sadb_msg` byte-identical (16 bytes). (covers REQ-4)
- [ ] AC-2: `setkey -DP` test: install SA via `setkey -c "add..."`; setkey -D dumps SA byte-identical to upstream's. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-3: SADB_REGISTER test: subscribe with SATYPE=AH; reply carries SUPPORTED_AUTH TLV listing all `pfkey_supported = 1` aalg algorithms. (covers REQ-6)
- [ ] AC-4: SADB_GETSPI + SADB_UPDATE flow: GETSPI returns SPI; UPDATE installs full SA; subsequent SADB_GET returns the SA. (covers REQ-2, REQ-7)
- [ ] AC-5: SADB_ACQUIRE test: install policy without SA via `setkey -c spdadd ...`; send packet → racoon-equivalent test daemon receives SADB_ACQUIRE. (covers REQ-8)
- [ ] AC-6: SADB_EXPIRE test: SA with `lifetime soft byte 1024` → after 1024-byte traffic → daemon receives SADB_EXPIRE soft variant. (covers REQ-9)
- [ ] AC-7: NAT-T test: install SA with NAT_T_TYPE/SPORT/DPORT TLVs → tcpdump shows UDP-encap'd ESP. (covers REQ-10)
- [ ] AC-8: LSM context test (SELinux): SADB_ADD with SEC_CTX TLV → SA installed with LSM sid; cross-ref via `ip xfrm state list` shows context. (covers REQ-11)
- [ ] AC-9: Per-netns test: PF_KEYv2 socket in netns A doesn't see SAs added in netns B. (covers REQ-12)
- [ ] AC-10: Wire compat: PF_KEYv2 message capture from racoon byte-identical to upstream's. (covers REQ-2, REQ-3)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::pfkey::PfKeyV2` — socket family root
- `kernel::net::xfrm::pfkey::sock::PfKeySock`
- `kernel::net::xfrm::pfkey::msg::SadbMsg` — `struct sadb_msg` wrapper
- `kernel::net::xfrm::pfkey::ext::SadbExt` — TLV ext-type parser/builder
- `kernel::net::xfrm::pfkey::dispatch::Dispatch` — per-SADB_* handler
- `kernel::net::xfrm::pfkey::translate::ToXfrm` — SADB → xfrm_state/xfrm_policy
- `kernel::net::xfrm::pfkey::translate::FromXfrm` — xfrm_state/xfrm_policy → SADB
- `kernel::net::xfrm::pfkey::broadcast::Broadcast` — `pfkey_broadcast`
- `kernel::net::xfrm::pfkey::register::Register` — `SADB_REGISTER` algo capability listing
- `kernel::net::xfrm::pfkey::nat_t::NatT` — NAT-T TLV apply
- `kernel::net::xfrm::pfkey::lsm::SecCtx` — `SADB_X_EXT_SEC_CTX` handler

### Locking and concurrency

- **`pfkey_table_lock`** (per-netns rwlock): protects per-netns PF_KEYv2 socket list
- **Per-socket socket-lock**: serializes per-socket sendmsg/recvmsg (inherited from socket core)
- **Per-netns broadcast emission**: lockless multicast via per-bound-socket queue

### Error handling

- `Err(EINVAL)` — malformed SADB message / unknown ext type / unsupported algorithm
- `Err(EEXIST)` — SADB_ADD for duplicate SA
- `Err(ENOENT)` — SADB_DELETE/GET for non-existent
- `Err(EPERM)` — non-CAP_NET_ADMIN
- `Err(ENOMEM)` — alloc fail
- `Err(EAFNOSUPPORT)` — bad address family

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TLV ext-walk per-message (bounded by sadb_msg_len) | `kani::proofs::net::xfrm::pfkey::tlv_walk_safety` |
| Per-SADB_* dispatch table walk | `kani::proofs::net::xfrm::pfkey::dispatch_safety` |
| SADB→xfrm translation arithmetic (no overflow on lifetime fields) | `kani::proofs::net::xfrm::pfkey::translate_safety` |
| Broadcast emission (skb-clone per recipient) | `kani::proofs::net::xfrm::pfkey::broadcast_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/xfrm_state_machine.tla` from `state.md` for state-mutation atomicity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns PF_KEYv2 socket list | every socket appears in exactly one bucket | `kani::proofs::net::xfrm::pfkey::table_invariants` |
| Per-message TLV ext list | sum of ext lengths ≤ `sadb_msg_len * 8` | `kani::proofs::net::xfrm::pfkey::msg_invariants` |

### Layer 4: Functional correctness (opt-in)

- **PF_KEYv2 ↔ xfrm equivalence theorem** via Verus — proves: ∀ pair (PF_KEYv2 message, equivalent NETLINK_XFRM message) producing equivalent SADB/SPD state.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-SADB_* dispatch table + per-ext-type policy table `static const` | § Mandatory |
| **SIZE_OVERFLOW** | sadb_msg_len + ext-length arithmetic uses checked operators (CVE class: CVE-2010-class TLV overflow on sadb_msg_len ≠ Σ ext lengths) | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-emit broadcast skb cleared (may carry SA key material in SADB_GET response) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-socket (cross-ref `net/sock.md`)
- **CONSTIFY**: see above
- **USERCOPY**: SADB-message parsing uses bound-checked accessors throughout (defense vs. malformed-userspace messages)
- **KERNEXEC**: dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for PF_KEYv2 sockets; GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_xfrm_*` for SA/policy alloc/delete (cross-ref upstream LSM contract).
- Useful default GR-RBAC policy: deny PF_KEYv2 socket creation outside gradm-marked `ipsec_admin` role; PF_KEYv2 is a legacy interface and disabling it for new deployments hardens the IPSec attack surface.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — PF_KEYv2 wire format is exhaustively specified by RFC 2367 + upstream Linux X-extensions)

## Out of Scope

- NETLINK_XFRM modern alternative (cross-ref `net/xfrm/user.md`)
- Underlying SA/policy storage (cross-ref `net/xfrm/state.md`, `net/xfrm/policy.md`)
- 32-bit-only paths
- Implementation code
