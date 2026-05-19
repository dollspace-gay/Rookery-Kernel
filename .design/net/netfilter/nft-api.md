# Tier-3: net/netfilter/nft-api — nftables NETLINK API (NFNL_SUBSYS_NFTABLES)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_tables_api.c
  - net/netfilter/nf_tables_offload.c
  - net/netfilter/nf_tables_trace.c
  - include/net/netfilter/nf_tables.h
  - include/uapi/linux/netfilter/nf_tables.h
  - include/uapi/linux/netfilter/nfnetlink.h
-->

## Summary
Tier-3 design for nftables' NETLINK_NETFILTER + NFNL_SUBSYS_NFTABLES wire interface — the modern packet-filtering ruleset control protocol that replaces iptables/ip6tables/ebtables/arptables. Covers the full message surface: `NFT_MSG_NEWTABLE/DELTABLE/GETTABLE`, `_NEWCHAIN/DELCHAIN/GETCHAIN`, `_NEWRULE/DELRULE/GETRULE`, `_NEWSET/DELSET/GETSET/NEWSETELEM/DELSETELEM/GETSETELEM`, `_NEWGEN/GETGEN`, `_TRACE`, `_NEWOBJ/DELOBJ/GETOBJ`, `_NEWFLOWTABLE/DELFLOWTABLE/GETFLOWTABLE`, transaction batching via `NFNL_BATCH_BEGIN/END`, and HW-offload integration (`nf_tables_offload.c`).

Used by: `nft` userspace tool (libnftnl); `iptables-nft` (legacy iptables CLI translated to nftables); `firewalld` (in nftables backend mode); Cilium (control-plane subset); systemd's `iptables-restore-translate` mode.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/nft-core.md` (VM evaluation engine that consumes the rules installed via this API), `net/netfilter/nft-expressions.md` (per-expression types), `net/netfilter/nft-sets.md` (set storage backends).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| nftables NETLINK API: per-message handlers, transaction commit, dump | `net/netfilter/nf_tables_api.c` |
| HW offload (per-rule offload to NIC) | `net/netfilter/nf_tables_offload.c` |
| Per-rule trace facility | `net/netfilter/nf_tables_trace.c` |
| Public API | `include/net/netfilter/nf_tables.h` |
| UAPI: NFT_MSG_*, NFTA_*, NFT_*_F | `include/uapi/linux/netfilter/nf_tables.h`, `include/uapi/linux/netfilter/nfnetlink.h` |

## Compatibility contract

### NFT_MSG_* message types

Per `include/uapi/linux/netfilter/nf_tables.h`:

| Message | Operation |
|---|---|
| `NFT_MSG_NEWTABLE` | Create/replace table |
| `NFT_MSG_GETTABLE` | Lookup or dump |
| `NFT_MSG_DELTABLE` | Delete |
| `NFT_MSG_NEWCHAIN` | Create/replace chain |
| `NFT_MSG_GETCHAIN` | Lookup |
| `NFT_MSG_DELCHAIN` | Delete |
| `NFT_MSG_NEWRULE` | Add rule |
| `NFT_MSG_GETRULE` | Lookup |
| `NFT_MSG_DELRULE` | Delete |
| `NFT_MSG_NEWSET` | Create set |
| `NFT_MSG_GETSET` | Lookup |
| `NFT_MSG_DELSET` | Delete |
| `NFT_MSG_NEWSETELEM` | Add element to set |
| `NFT_MSG_GETSETELEM` | Lookup element |
| `NFT_MSG_DELSETELEM` | Delete element |
| `NFT_MSG_NEWGEN` / `_GETGEN` | Generation counter |
| `NFT_MSG_TRACE` | Per-skb trace event (kernel→user) |
| `NFT_MSG_NEWOBJ` | Create stateful object (counter/quota/limit/ct-helper/synproxy/secmark/connlimit/tunnel) |
| `NFT_MSG_GETOBJ` / `_DELOBJ` | Manage objects |
| `NFT_MSG_NEWFLOWTABLE` / `_GETFLOWTABLE` / `_DELFLOWTABLE` | Manage flowtable |

Wire format byte-identical so `nft` works unchanged.

### NFTA_* NLA attributes

Per-message NLA attribute schemas (e.g., for NEWTABLE):
- `NFTA_TABLE_NAME` (string)
- `NFTA_TABLE_FLAGS` (NFT_TABLE_F_DORMANT, NFT_TABLE_F_OWNER)
- `NFTA_TABLE_USE` (read-only: rule count)
- `NFTA_TABLE_HANDLE` (per-table 64-bit handle)
- `NFTA_TABLE_USERDATA` (opaque user data)
- `NFTA_TABLE_OWNER` (per-process owner — when F_OWNER set)

For NEWCHAIN: `NFTA_CHAIN_TABLE`, `NFTA_CHAIN_NAME`, `NFTA_CHAIN_HANDLE`, `NFTA_CHAIN_HOOK` (nested: HOOK_HOOKNUM + HOOK_PRIORITY + HOOK_DEV + HOOK_DEVS), `NFTA_CHAIN_TYPE` (filter/nat/route), `NFTA_CHAIN_POLICY`, `NFTA_CHAIN_FLAGS` (DORMANT/HW_OFFLOAD/BINDING), `NFTA_CHAIN_USE`, `NFTA_CHAIN_USERDATA`.

For NEWRULE: `NFTA_RULE_TABLE`, `_CHAIN`, `_HANDLE`, `_EXPRESSIONS` (nested array of NFTA_LIST_ELEM each with NFTA_EXPR_NAME + NFTA_EXPR_DATA), `_POSITION`, `_USERDATA`, `_ID`, `_POSITION_ID`.

For NEWSET: `NFTA_SET_TABLE`, `_NAME`, `_HANDLE`, `_FLAGS` (NFT_SET_*), `_KEY_TYPE`, `_KEY_LEN`, `_DATA_TYPE`, `_DATA_LEN`, `_POLICY` (NFT_SET_POL_PERFORMANCE / MEMORY), `_DESC` (desc-tuple-size etc.), `_TIMEOUT`, `_GC_INTERVAL`, `_USERDATA`, `_OBJ_TYPE`, `_EXPR`.

For NEWSETELEM: `NFTA_SET_ELEM_LIST_TABLE`, `_SET`, `_ELEMENTS` (nested array of NFTA_LIST_ELEM each with NFTA_SET_ELEM_KEY / KEY_END / DATA / FLAGS / TIMEOUT / EXPIRATION / USERDATA / EXPR / OBJREF / PAD).

Wire format byte-identical for all message types.

### Transaction batching

Multi-message transactions via `NFNL_MSG_BATCH_BEGIN` and `NFNL_MSG_BATCH_END` envelope. Within batch:
1. All messages parsed + validated atomically
2. On any validation failure → entire batch rolled back
3. On success → atomically committed (concurrent classify never sees mid-batch state)

Implements the classic `nft -f file.nft` atomic-load semantics.

Identical batch protocol.

### Generation counter (`NFT_MSG_NEWGEN`)

Per-netns 32-bit generation counter; incremented on every successful commit. Userspace listeners subscribe to NEWGEN events to detect ruleset changes (e.g., monitor for ruleset-replacement events in fwd / `nft monitor`).

### Per-rule HW offload (`NFTA_CHAIN_FLAGS = NFT_CHAIN_HW_OFFLOAD`)

When chain has HW_OFFLOAD flag: per-rule expression chain compiled to `flow_action` array via `nf_tables_offload.c`; passed to driver via `flow_setup_cb_call(TC_SETUP_FT_*)`. On compile failure → ENOTSUPP returned to caller.

Identical contract.

### Per-rule trace (`NFT_MSG_TRACE`)

When rule has `nftrace=1` (set via `meta nftrace set 1`): per-rule traversal events emitted to userspace listeners (subscribed to NFTNLGRP_TRACE) for debugging (`nft monitor trace`).

### `nft monitor` event emission

Multicast groups per `include/uapi/linux/netfilter/nf_tables.h`:
- `NFTNLGRP_NEW` / `NFTNLGRP_DEL` / `NFTNLGRP_GETRULE` / `NFTNLGRP_GETSET` / `NFTNLGRP_TRACE`

Each NFT_MSG_* mutation broadcasts to subscribed listeners.

### Per-table ownership (`NFT_TABLE_F_OWNER`)

Modern feature: a userspace process can claim a table as its owner; the kernel tracks ownership via process pid; on owner exit, kernel auto-deletes the owned table. Identical UAPI.

### Flowtable management (`NFT_MSG_NEWFLOWTABLE` etc.)

Flowtable lifecycle: per-table flowtable object with hooks `NF_INET_INGRESS` per-netdev. Flowtable entries populated by conntrack + `flow add` rule; consumed by fast-path bypass.

Cross-ref `net/netfilter/flowtable.md` for the runtime offload mechanism.

## Requirements

- REQ-1: All `NFT_MSG_*` message types (per the table) parsed + emitted identically.
- REQ-2: All per-message `NFTA_*` NLA attribute schemas parsed identically.
- REQ-3: Transaction batching via `NFNL_MSG_BATCH_BEGIN/END`: atomic commit-or-rollback semantics; concurrent classify never sees mid-batch state.
- REQ-4: Per-netns generation counter incremented on every successful commit; emitted via `NFT_MSG_NEWGEN`.
- REQ-5: HW offload via `NFT_CHAIN_HW_OFFLOAD` flag: per-rule expression chain compiled to `flow_action` array; driver-side ndo_setup_tc bridge.
- REQ-6: Per-rule trace via `NFT_MSG_TRACE`: per-skb trace events emitted to NFTNLGRP_TRACE listeners.
- REQ-7: Multicast event groups: `NFTNLGRP_NEW / DEL / GETRULE / GETSET / TRACE`; mutations broadcast to subscribers.
- REQ-8: Per-table ownership: `NFT_TABLE_F_OWNER` flag; per-pid tracking; auto-delete on owner exit.
- REQ-9: Flowtable management: per-table flowtable lifecycle; per-netdev hook integration.
- REQ-10: Per-rule UDATA + per-set UDATA + per-object UDATA: opaque user-data preserved across get/set.
- REQ-11: Per-table dormant flag (`NFT_TABLE_F_DORMANT`): when set, all chains in the table are unhooked (rules don't run).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `nft list ruleset` byte-identical content after equivalent setup. (covers REQ-1, REQ-2)
- [ ] AC-2: Atomic transaction test: `nft -f file.nft` with valid + invalid rules → both rejected; `nft list ruleset` unchanged. (covers REQ-3)
- [ ] AC-3: Generation-counter test: `nft monitor` running; mutations cause `NFT_MSG_NEWGEN` events with incrementing generation. (covers REQ-4, REQ-7)
- [ ] AC-4: HW offload test: `nft add chain ... { type filter hook ingress device eth0 priority 0; flags offload; }`; supported NIC accepts offload; per-rule traffic counted in HW. (covers REQ-5)
- [ ] AC-5: Trace test: `nft add rule ... meta nftrace set 1; nft monitor trace`; per-skb trace events delivered. (covers REQ-6)
- [ ] AC-6: Owner test: `nft add table inet myown { flags owner; }`; process B can list but not modify A's table; on A exit, table auto-deleted. (covers REQ-8)
- [ ] AC-7: Flowtable test: `nft add flowtable inet f1 { hook ingress priority filter; devices = { eth0 }; }`; `nft add rule ... ip protocol tcp flow add @f1`; flow inserted into flowtable on first packet. (covers REQ-9)
- [ ] AC-8: UDATA preservation: write rule with UDATA blob; subsequent GETRULE returns same blob byte-identical. (covers REQ-10)
- [ ] AC-9: Dormant flag test: set table flags=DORMANT; rules in chain don't fire (no traffic match); clear flag → rules fire again. (covers REQ-11)
- [ ] AC-10: Multi-message-batch atomic test: batch with 100 NEWRULEs; concurrent classify either sees all-old-state or all-new-state, never partial. (covers REQ-3)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::nft::api::Api` — top-level NETLINK_NETFILTER NFNL_SUBSYS_NFTABLES handler
- `kernel::net::netfilter::nft::api::Dispatch` — per-NFT_MSG_* dispatch table
- `kernel::net::netfilter::nft::api::nla::NlaParser` — per-attribute NLA parser
- `kernel::net::netfilter::nft::api::msg::Newtable`, `Newchain`, `Newrule`, `Newset`, `Newsetelem`, `Newobj`, `Newflowtable`, etc.
- `kernel::net::netfilter::nft::api::Transaction` — `NFNL_MSG_BATCH_BEGIN/END` atomic commit
- `kernel::net::netfilter::nft::api::Generation` — per-netns generation counter
- `kernel::net::netfilter::nft::api::Trace` — `NFT_MSG_TRACE` event emitter
- `kernel::net::netfilter::nft::api::Notify` — multicast event broadcast
- `kernel::net::netfilter::nft::api::Owner` — `NFT_TABLE_F_OWNER` per-pid tracking
- `kernel::net::netfilter::nft::api::offload::Offload` — HW offload bridge

### Locking and concurrency

- **Per-netns `nft_commit_mutex`** (mutex): held during transaction commit
- **Per-netns table list** (RCU-protected): readers RCU-side; mutators under commit_mutex
- **Per-netns generation counter** (atomic): incremented on commit
- **Per-table dormant-state** (bit in flags; RCU-readable): when set, hook-traversal returns ACCEPT immediately

### Error handling

- `Err(EEXIST)` — duplicate (table/chain/rule/set/element) at same name/handle
- `Err(ENOENT)` — non-existent
- `Err(EBUSY)` — table dormant + DELCHAIN attempt with use_count > 0
- `Err(EINVAL)` — bad NLA / bad NFTA_*
- `Err(EOPNOTSUPP)` — HW offload requested but not supported
- `Err(ENOMEM)` — alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-NFT_MSG_* dispatch table walk | `kani::proofs::net::netfilter::nft::api::dispatch_safety` |
| NLA parser per-attribute bounds | `kani::proofs::net::netfilter::nft::api::nla_parse_safety` |
| Transaction commit (RCU-pointer swap; no readers see partial swap) | `kani::proofs::net::netfilter::nft::api::commit_safety` |
| Per-table dormant-state read (RCU-side) | `kani::proofs::net::netfilter::nft::api::dormant_safety` |
| Owner tracking (per-pid hashtable; cleanup on exit_creds) | `kani::proofs::net::netfilter::nft::api::owner_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/nft_vm_eval.tla` from `nft-core.md` for transaction-atomicity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns table list | every table has unique (family, name); generation counter monotonically increases on commit | `kani::proofs::net::netfilter::nft::api::list_invariants` |
| Per-table chain/rule/set lists | every chain/rule/set has unique handle within table | `kani::proofs::net::netfilter::nft::api::handle_invariants` |
| Per-rule expression list | every expression's `ops` non-NULL after successful NEWRULE | `kani::proofs::net::netfilter::nft::api::expr_invariants` |
| Per-table owner | when F_OWNER set, exactly one pid owns the table; cleanup on owner exit | `kani::proofs::net::netfilter::nft::api::owner_invariants` |

### Layer 4: Functional correctness (declared in `net/netfilter/00-overview.md` Layer 4)

- **nftables transaction atomicity theorem** via TLA+ refinement — proves: NFT_MSG_BEGIN..COMMIT is atomic from concurrent-classify's perspective. Owned in `nft-core.md`.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-table + per-chain + per-rule + per-set + per-object refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-type slab caches | § Mandatory |
| **CONSTIFY** | per-NFT_MSG_* dispatch table + per-NFTA_* validation policy `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed nft state cleared (carries policy decisions + selectors) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors throughout (CVE-2022-class nftables-userdata-overflow defense)
- **SIZE_OVERFLOW**: per-rule expression-array length + per-set element-count arithmetic uses checked operators
- **KERNEXEC**: dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all NFT_MSG_* mutations.
- Default useful GR-RBAC policy: deny NFT_MSG_NEWTABLE / NEWRULE outside gradm-marked `firewall_admin` role; nftables is the highest-privilege firewall surface.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: nft API attribute copy_in/out (NFTA_TABLE_NAME, NFTA_CHAIN_NAME, NFTA_RULE_EXPRESSIONS) goes through whitelisted slabs with bounded sizes.
- PAX_KERNEXEC: nft API dispatch and netlink message handlers run from .text only; W^X enforced.
- PAX_RANDKSTACK: per-syscall stack-base randomization on `sendmsg()` netlink batches addressing NFNL_SUBSYS_NFTABLES.
- PAX_REFCOUNT: `nft_table->use`, `nft_chain->use`, and dependent object refcounts use saturating refcount_t across batch commits.
- PAX_MEMORY_SANITIZE: zero on free for transaction-log entries and aborted ruleset fragments to prevent slab disclosure of partial config.
- PAX_UDEREF: all `nla_data()` pointers from userland validated before deref under uderef.
- PAX_RAP / kCFI: `nfnl_callback->call`, `nfnl_callback->call_batch`, and `nft_expr_type->ops` indirect calls kCFI-typed.
- GRKERNSEC_HIDESYM: nft list/dump output redacts kernel addresses; `kptr_restrict=2` applied to handle exposure.
- GRKERNSEC_DMESG: API error paths (EBUSY, ELOOP, ERANGE) rate-limited and CAP_SYSLOG-gated.
- nftables CAP_NET_ADMIN: every `NFT_MSG_*` checked with `ns_capable(net->user_ns, CAP_NET_ADMIN)` so unprivileged userns cannot mutate parent-net ruleset.
- Transaction commit-abort: `nf_tables_commit()` runs under `nft_net->commit_mutex`; abort restores prior generation atomically; generation counter prevents stale lookup.
- nft_chain PAX_REFCOUNT: `nft_use_inc()`/`nft_use_dec()` are saturating; binding count cannot wrap.
- JIT W^X (PAX_KERNEXEC): set lookup JIT (where present) holds pages RO+X after emit.

Rationale: the nft API is the privileged control plane; saturating refcounts plus transactional commit semantics close the class of "rule references freed object" UAFs reported repeatedly in nf_tables_api.c.

## Open Questions

(none — nftables NETLINK API exhaustively specified by upstream + libnftnl test corpus + nft regression-test suite)

## Out of Scope

- VM evaluation engine (cross-ref `net/netfilter/nft-core.md`)
- Per-expression types (cross-ref `net/netfilter/nft-expressions.md`)
- Set storage backends (cross-ref `net/netfilter/nft-sets.md`)
- Flowtable runtime (cross-ref `net/netfilter/flowtable.md`)
- 32-bit-only paths
- Implementation code
