# Tier-3: net/netfilter/conntrack-bpf — BPF kfunc set for conntrack

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_conntrack_bpf.c
  - net/netfilter/nf_nat_bpf.c
  - net/netfilter/nf_flow_table_bpf.c
  - include/net/netfilter/nf_conntrack.h
-->

## Summary
Tier-3 design for the BPF kfunc set exposing conntrack + NAT + flowtable lookup/manipulation to BPF programs. The cornerstone of every modern eBPF-driven dataplane: Cilium, Calico Express dataplane, Katran, custom XDP firewalls. Lets BPF-XDP / BPF-TC / BPF-LSM / cgroup-BPF programs allocate, lookup, change-state, release conntrack entries; insert flowtable entries; query NAT mappings; all under verifier-checked refcount-paired safety.

Three kfunc-set modules:
- `nf_conntrack_bpf.c` — `bpf_*ct_alloc`, `bpf_ct_insert_entry`, `bpf_ct_set_status`, `bpf_*ct_lookup`, `bpf_ct_release`, `bpf_ct_set_timeout`, `bpf_ct_change_timeout`, `bpf_ct_set_nat_info`
- `nf_nat_bpf.c` — `bpf_ct_set_nat_info` (alias; bridge for NAT-aware kfuncs)
- `nf_flow_table_bpf.c` — `bpf_xdp_flow_lookup`, `bpf_skb_flow_lookup`, `bpf_flowtable_offload_release`

Each kfunc registered with the BPF verifier with appropriate `KF_TRUSTED_ARGS` / `KF_RET_NULL` / `KF_ACQUIRE` / `KF_RELEASE` flags so verifier enforces every `*_alloc`/`*_lookup` is paired with `*_release` on every program path.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `kernel/bpf/00-overview.md` (verifier + struct_ops infrastructure), `net/netfilter/conntrack-core.md` (consumed by lookup kfuncs), `net/netfilter/flowtable.md` (cross-ref).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Conntrack BPF kfunc set | `net/netfilter/nf_conntrack_bpf.c` |
| NAT BPF kfunc bridge | `net/netfilter/nf_nat_bpf.c` |
| Flowtable BPF kfunc set | `net/netfilter/nf_flow_table_bpf.c` |
| Public API | `include/net/netfilter/nf_conntrack.h` |

## Compatibility contract

### `bpf_ct_alloc` / `bpf_ct_insert_entry` / `bpf_ct_release`

```c
struct nf_conn *bpf_ct_alloc(u32 zone_id, u32 flags, struct bpf_ct_opts *opts, u32 opts__sz);
struct nf_conn *bpf_ct_insert_entry(struct nf_conn *nfct);
void bpf_ct_release(struct nf_conn *nfct);
```

`bpf_ct_alloc` allocates a new `nf_conn` (UNCONFIRMED state) within specified zone. Returns refcount-acquired pointer.
`bpf_ct_insert_entry` confirms the alloc'd entry into the conntrack hashtable (transitions UNCONFIRMED → CONFIRMED). Returns same nf_conn or error.
`bpf_ct_release` paired with `bpf_ct_alloc` — verifier enforces.

KF_ACQUIRE on alloc + KF_RELEASE on release. Identical signatures.

### `bpf_*ct_lookup` (per-AF)

```c
struct nf_conn *bpf_xdp_ct_lookup(struct xdp_md *ctx, struct bpf_sock_tuple *tuple, u32 tuple__sz, u32 opts, u32 opts__sz);
struct nf_conn *bpf_skb_ct_lookup(struct __sk_buff *ctx, struct bpf_sock_tuple *tuple, u32 tuple__sz, u32 opts, u32 opts__sz);
```

Lookup conntrack by user-provided 5-tuple in XDP or SKB context. Returns refcount-acquired nf_conn on hit; NULL with opts.error set on miss.

KF_ACQUIRE + KF_RET_NULL.

### `bpf_ct_set_status` / `bpf_ct_change_status`

```c
int bpf_ct_set_status(const struct nf_conn *nfct, u32 status);
int bpf_ct_change_status(struct nf_conn *nfct, u32 status);
```

Set or change `nf_conn.status` (IPS_* flags). Used for Cilium-style "mark connection as service-NAT'd" patterns.

### `bpf_ct_set_timeout` / `bpf_ct_change_timeout`

```c
int bpf_ct_set_timeout(struct nf_conn *nfct, u32 timeout);
int bpf_ct_change_timeout(struct nf_conn *nfct, u32 timeout);
```

Set or change per-flow timeout. KF_TRUSTED_ARGS required on the nf_conn pointer (must be from a `*_alloc`/`*_lookup` chain).

### `bpf_ct_set_nat_info` (in `nf_nat_bpf.c`)

```c
int bpf_ct_set_nat_info(struct nf_conn *nfct, union nf_inet_addr *addr, int port, enum nf_nat_manip_type manip);
```

Apply NAT mapping (SNAT / DNAT) to a BPF-allocated conntrack. Combined with `bpf_ct_alloc + bpf_ct_insert_entry` lets BPF programs install fully-NAT'd flows without traversing nftables.

### Flowtable kfuncs (in `nf_flow_table_bpf.c`)

```c
struct flow_offload_tuple_rhash *bpf_xdp_flow_lookup(struct xdp_md *ctx, struct bpf_sock_tuple *tuple, u32 tuple__sz, u32 flags);
struct flow_offload_tuple_rhash *bpf_skb_flow_lookup(struct __sk_buff *ctx, struct bpf_sock_tuple *tuple, u32 tuple__sz, u32 flags);
void bpf_flowtable_offload_release(struct flow_offload_tuple_rhash *handle);
```

Cross-ref `net/netfilter/flowtable.md` REQ-7 — these are the kfuncs that BPF programs use to do per-tuple flowtable lookups in XDP/SKB context for fast-path forwarding.

### Verifier integration

Each kfunc registered via `register_btf_kfunc_id_set` with appropriate flags:
- `bpf_ct_alloc`, `bpf_*ct_lookup`, `bpf_xdp_flow_lookup`, `bpf_skb_flow_lookup`: `KF_ACQUIRE | KF_RET_NULL`
- `bpf_ct_release`, `bpf_flowtable_offload_release`: `KF_RELEASE`
- `bpf_ct_set_*`, `bpf_ct_change_*`, `bpf_ct_set_nat_info`: `KF_TRUSTED_ARGS`
- `bpf_ct_insert_entry`: `KF_TRUSTED_ARGS | KF_RET_NULL`

Verifier enforces that every `KF_ACQUIRE` kfunc's return value is consumed by a paired `KF_RELEASE` kfunc on every reachable program path. Programs failing this constraint fail load.

### Per-prog-type kfunc set registration

Three prog-type masks:
- `BPF_PROG_TYPE_SCHED_CLS` (TC) — full conntrack + NAT + flowtable kfunc access
- `BPF_PROG_TYPE_XDP` — XDP-context-specific kfuncs (xdp_ct_lookup + xdp_flow_lookup)
- `BPF_PROG_TYPE_TRACING` — read-only ct lookup (for telemetry)

Identical per-prog-type allow-list.

### Refcount contract

Per-`nf_conn` refcount inherited from conntrack-core (cross-ref `conntrack-core.md`). BPF acquire bumps refcount; release decrements. Misbalanced acquire/release detected at verifier load + at program exit.

### Read-only fields

Per-kfunc, certain `nf_conn` fields are read-only from BPF (e.g., `tuplehash`, `proto.tcp.state` — protected to maintain conntrack state-machine integrity). Mutable fields explicitly allowed via dedicated `bpf_ct_set_*` kfuncs.

## Requirements

- REQ-1: `bpf_ct_alloc` + `bpf_ct_insert_entry` + `bpf_ct_release` per upstream signatures; verifier-checked acquire/release pairing.
- REQ-2: `bpf_xdp_ct_lookup` + `bpf_skb_ct_lookup` per upstream signatures; XDP + SKB context support.
- REQ-3: `bpf_ct_set_status` / `_change_status` mutate `nf_conn.status` (IPS_* flags); KF_TRUSTED_ARGS.
- REQ-4: `bpf_ct_set_timeout` / `_change_timeout` mutate per-flow timeout; KF_TRUSTED_ARGS.
- REQ-5: `bpf_ct_set_nat_info` applies NAT mapping; bridges to `nf_nat_setup_info` (cross-ref `nat-core.md`).
- REQ-6: `bpf_xdp_flow_lookup` + `bpf_skb_flow_lookup` + `bpf_flowtable_offload_release` per upstream signatures (cross-ref `flowtable.md`).
- REQ-7: Per-prog-type kfunc set registration: SCHED_CLS / XDP / TRACING with appropriate per-mask allow-list.
- REQ-8: Verifier-enforced acquire/release pairing on every reachable program path; reject programs that leak refs.
- REQ-9: Read-only contract: per-kfunc allow-list of mutable nf_conn fields; other fields read-only.
- REQ-10: Per-skb / per-xdp context BTF struct exposure: `bpf_sock_tuple` for tuple input; `bpf_ct_opts` for per-call options.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: bpf_ct_alloc/insert/release round-trip test: BPF program calls alloc → set tuple via opts → insert_entry → release; verifier accepts; conntrack visible via `conntrack -L`. (covers REQ-1, REQ-8)
- [ ] AC-2: Verifier-rejection test: BPF program calls `bpf_ct_alloc` but doesn't `bpf_ct_release` on one path → verifier rejects with refcount-leak error. (covers REQ-8)
- [ ] AC-3: bpf_xdp_ct_lookup test: XDP program looks up established TCP flow by 5-tuple; returns refcount-acquired nf_conn; release after use. (covers REQ-2)
- [ ] AC-4: bpf_skb_ct_lookup test: TC program looks up flow; returns nf_conn; can read state via permitted fields. (covers REQ-2)
- [ ] AC-5: Set-status test: BPF calls `bpf_ct_set_status(nfct, IPS_HELPER_BIT)`; subsequent `conntrack -L` shows HELPER status set. (covers REQ-3)
- [ ] AC-6: Set-timeout test: BPF calls `bpf_ct_set_timeout(nfct, 86400)`; per-flow timeout updated to 86400s. (covers REQ-4)
- [ ] AC-7: NAT-info test: BPF calls `bpf_ct_set_nat_info(nfct, addr, port, NF_NAT_MANIP_SRC)`; conntrack has NAT mapping; subsequent traffic NAT'd correctly. (covers REQ-5)
- [ ] AC-8: Flowtable lookup test: BPF calls `bpf_xdp_flow_lookup`; on hit, performs cached forward; on miss, falls through. (covers REQ-6)
- [ ] AC-9: Per-prog-type test: kfunc allow-list rejects `bpf_ct_alloc` from `BPF_PROG_TYPE_TRACING` (read-only); accepts from SCHED_CLS / XDP. (covers REQ-7)
- [ ] AC-10: Read-only field test: BPF program attempts to write `nfct->tuplehash[0]` directly → verifier rejects (only `bpf_ct_set_*` kfuncs allowed). (covers REQ-9)
- [ ] AC-11: Cilium-equivalent test: full CT/NAT/flowtable BPF dataplane works at line rate using these kfuncs. (covers REQ-1 through REQ-9)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::netfilter::conntrack_bpf::Kfuncs` — kfunc set registration
- `kernel::net::netfilter::conntrack_bpf::Alloc` — `bpf_ct_alloc` + `_insert_entry` + `_release`
- `kernel::net::netfilter::conntrack_bpf::Lookup` — `bpf_xdp_ct_lookup` + `bpf_skb_ct_lookup`
- `kernel::net::netfilter::conntrack_bpf::Mutate` — `bpf_ct_set_*` + `_change_*`
- `kernel::net::netfilter::conntrack_bpf::Nat` — `bpf_ct_set_nat_info`
- `kernel::net::netfilter::conntrack_bpf::Flowtable` — flowtable lookup kfuncs
- `kernel::net::netfilter::conntrack_bpf::Verifier` — per-prog-type allow-list

### Locking and concurrency

- **Per-`nf_conn` refcount** (Refcount, cross-ref `conntrack-core.md`): acquired/released by kfuncs
- **No new global locks**: kfuncs are RCU-side or refcount-protected
- **BPF program execution context**: NMI-safe / RCU-side reads only (unless KF_TRUSTED_ARGS for mutation)

### Error handling

- `Err(EINVAL)` — bad tuple size / bad opts struct
- `Err(ENOENT)` — conntrack not found
- `Err(ENOMEM)` — alloc fail
- Set via `opts.error` field for KF_RET_NULL kfuncs (NULL signals failure)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `bpf_ct_alloc` refcount-acquire (sound; no double-acquire) | `kani::proofs::net::netfilter::conntrack_bpf::alloc_safety` |
| `bpf_ct_release` refcount-decrement (idempotent on already-zero is impossible per verifier) | `kani::proofs::net::netfilter::conntrack_bpf::release_safety` |
| `bpf_ct_set_nat_info` integration with `nf_nat_setup_info` (refcount + state-machine sound) | `kani::proofs::net::netfilter::conntrack_bpf::nat_safety` |
| Per-prog-type kfunc-set register (no overlap; per-mask correctness) | `kani::proofs::net::netfilter::conntrack_bpf::register_safety` |

### Layer 2: TLA+ models

(none new; relies on `models/net/conntrack_state_machine.tla` from `conntrack-proto.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-program kfunc-acquire/release count | per-program acquired refs = released refs at all program exit points | `kani::proofs::net::netfilter::conntrack_bpf::balance_invariants` |
| Per-prog-type kfunc allow-list | every entry has correct flag combination (ACQUIRE / RELEASE / TRUSTED_ARGS / RET_NULL) | `kani::proofs::net::netfilter::conntrack_bpf::flags_invariants` |

### Layer 4: Functional correctness (depends on `kernel/bpf/00-overview.md` Layer 4)

- **Verifier refcount-pairing soundness theorem** (cross-ref `kernel/bpf/00-overview.md`) — proves: BPF verifier rejects programs that fail to call paired RELEASE on any reachable program path. Owned by BPF-verifier Tier-3, consumed here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-nf_conn refcount acquire/release uses `Refcount` (saturating) — verifier guarantees pairing | § Mandatory |
| **CONSTIFY** | kfunc registration tables `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, CONSTIFY**: see above
- **MEMORY_SANITIZE**: nf_conn fields read by BPF programs are restricted via per-field allow-list; sensitive fields (helper key, secctx) not exposed
- **USERCOPY**: not applicable (BPF programs run in kernel context with verifier-checked memory access)

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_BPF)` already required for BPF program load.
- LSM hook `security_xfrm_state_pol_flow_match` indirectly invoked through SA-lookup paths if BPF program's CT-NAT touches xfrm.
- Useful default GR-RBAC policy: deny BPF program loading with conntrack kfunc usage outside gradm-marked `bpf_ct_users` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — conntrack BPF kfunc set exhaustively specified by upstream + Cilium dataplane test corpus)

## Out of Scope

- BPF verifier internals (cross-ref `kernel/bpf/00-overview.md`)
- Conntrack core (cross-ref `conntrack-core.md`)
- Flowtable detail (cross-ref `flowtable.md`)
- 32-bit-only paths
- Implementation code
