# Tier-3: net/sched/sch-bpf — BPF programmable qdisc (struct_ops)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/bpf_qdisc.c
  - include/linux/bpf.h
  - include/net/sch_generic.h
  - include/uapi/linux/pkt_sched.h
-->

## Summary
Tier-3 design for `bpf_qdisc` — a programmable qdisc whose enqueue/dequeue/init/reset/destroy/change/dump methods are implemented by a BPF struct_ops program. Lets userspace write entirely custom queueing disciplines in BPF without recompiling the kernel: per-flow load shedding, application-aware AQM, ML-driven shaping, custom SLA enforcement. Built atop the BPF struct_ops infrastructure (cross-ref `kernel/bpf/00-overview.md`).

The kernel exposes a `Qdisc_ops` interface for BPF programs to fill in; verifier checks each method against a per-method type signature; loaded programs become a real `Qdisc_ops` registered via `register_qdisc(ops)`. Per-qdisc state is held in BPF-map-backed memory accessible from program calls.

Recently added (kernel 6.7+ in upstream); Cilium's qdisc-driven HTTP-aware scheduling and similar setups exercise this Tier-3.

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `kernel/bpf/00-overview.md` (struct_ops infrastructure, verifier, kfunc set), `net/sched/sch-api.md` (Qdisc_ops contract).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| BPF qdisc struct_ops registration, per-method dispatch, NLA parser, dump | `net/sched/bpf_qdisc.c` |
| BPF struct_ops infrastructure | `include/linux/bpf.h` (BPF_PROG_TYPE_STRUCT_OPS) |
| Public API | `include/net/sch_generic.h` |
| UAPI | `include/uapi/linux/pkt_sched.h` (TCA_BPF_QDISC_*) |

## Compatibility contract

### `struct bpf_qdisc_ops` (BPF struct_ops handle)

```c
struct bpf_qdisc_ops {
    int  (*enqueue)(struct sk_buff *, struct Qdisc *, struct sk_buff **);
    struct sk_buff *(*dequeue)(struct Qdisc *);
    void (*reset)(struct Qdisc *);
    int  (*init)(struct Qdisc *, struct nlattr *, struct netlink_ext_ack *);
    void (*destroy)(struct Qdisc *);
    int  (*change)(struct Qdisc *, struct nlattr *, struct netlink_ext_ack *);
    int  (*dump)(struct Qdisc *, struct sk_buff *);
    /* ...mirror of struct Qdisc_ops methods that BPF can implement */
};
```

Each field's BPF program is verified against the corresponding `Qdisc_ops` member's signature. Unimplemented fields fall through to a kernel default.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_BPF_QDISC_NAME` (qdisc-instance name string for dumps)
- `TCA_BPF_QDISC_PROG_FD` (FD obtained from BPF_PROG_LOAD struct_ops attach)
- `TCA_BPF_QDISC_FLAGS`

Wire format byte-identical so iproute2's `tc qdisc add ... bpf_qdisc ...` works unchanged.

### Per-qdisc BPF state

BPF programs access per-qdisc state via:
- `Qdisc->priv` — per-qdisc private data (BPF-allocated; size determined at struct_ops attach)
- BPF-map-backed maps (HASH, ARRAY, PERCPU_HASH, etc. — declared by program)
- Per-skb context via `__sk_buff` (full read access; limited write per BPF_PROG_TYPE_QDISC permissions)

### Helpers / kfuncs available

BPF qdisc programs can call:
- `bpf_skb_*` skb-mutation helpers (bpf_skb_store_bytes, etc.)
- `bpf_qdisc_enqueue_skb` — enqueue skb to internal kernel skb queue
- `bpf_qdisc_dequeue_skb` — dequeue from internal kernel skb queue
- `bpf_qdisc_skb_drop` — drop skb with stat update
- Generic helpers: maps, time-related (bpf_ktime_get_ns), per-CPU stats, BPF-spinlock

Identical helper set so existing programs work unchanged.

### Verifier contract

Per `kernel/bpf/00-overview.md` verifier:
- Each struct_ops method gets context-specific type-checking
- `enqueue` must call exactly one of `bpf_qdisc_enqueue_skb` or `bpf_qdisc_skb_drop` per entry
- `dequeue` must return either NULL or a skb obtained via `bpf_qdisc_dequeue_skb`
- No unbounded loops; per-method execution time bounded

Identical verifier rules.

### `register_qdisc` integration

When struct_ops attach completes successfully, kernel calls `register_qdisc(bpf_ops_synthesized_qdisc_ops)` registering the BPF-backed qdisc as a regular `Qdisc_ops` under the user-supplied `TCA_BPF_QDISC_NAME`. Other tc operations (graft, change, replace) work identically against the BPF qdisc.

### Per-qdisc dump

`TCA_BPF_QDISC_*` NLA-encoded plus optional BPF-program-emitted custom NLA via `dump` callback. Format byte-identical so `tc -s qdisc show` works.

## Requirements

- REQ-1: `struct bpf_qdisc_ops` struct_ops handle with all `Qdisc_ops` methods implementable by BPF.
- REQ-2: Each BPF method verified against per-method type signature; reject programs failing verifier.
- REQ-3: `TCA_BPF_QDISC_NAME` + `TCA_BPF_QDISC_PROG_FD` + `TCA_BPF_QDISC_FLAGS` NLAs parsed identically.
- REQ-4: Per-qdisc BPF state via `Qdisc->priv` (BPF-allocated; size declared at struct_ops attach).
- REQ-5: BPF helpers/kfuncs: `bpf_qdisc_enqueue_skb` / `_dequeue_skb` / `_skb_drop` + generic helpers; identical signatures.
- REQ-6: Verifier rules: each `enqueue` call must exit via `bpf_qdisc_enqueue_skb` or `_skb_drop`; `dequeue` must return NULL or kernel-tracked skb; no unbounded loops.
- REQ-7: `register_qdisc` integration: successful struct_ops attach → BPF-backed qdisc registered under `TCA_BPF_QDISC_NAME`.
- REQ-8: Per-qdisc dump: standard NLAs + BPF-program-emitted custom NLA via `dump` callback.
- REQ-9: Action chain integration: BPF qdisc can install inner qdiscs / classifiers in the standard tc tree (the BPF program can choose to delegate or not).
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: BPF qdisc struct_ops attach test: load BPF program implementing enqueue + dequeue (simple FIFO); attach via BPF_PROG_LOAD with struct_ops; `tc qdisc add ... bpf_qdisc ... fd <FD>` → qdisc visible via `tc qdisc show`. (covers REQ-1, REQ-3)
- [ ] AC-2: Verifier-rejection test: BPF enqueue program that doesn't call enqueue_skb or skb_drop → verifier rejects at struct_ops attach. (covers REQ-2, REQ-6)
- [ ] AC-3: Helper test: BPF enqueue calls `bpf_qdisc_enqueue_skb`; subsequent `bpf_qdisc_dequeue_skb` returns the same skb. (covers REQ-5)
- [ ] AC-4: Per-qdisc state test: BPF program declares HASH map for per-flow state; classify+enqueue updates per-flow counters; dequeue reads them. (covers REQ-4)
- [ ] AC-5: Custom NLA dump test: BPF dump method emits `BPF_QDISC_CUSTOM_ATTR_*` NLA; `tc -s qdisc show` displays them. (covers REQ-8)
- [ ] AC-6: Drop-program test: BPF enqueue calls `bpf_qdisc_skb_drop` for skb matching some criterion; `tc -s qdisc show` shows drop counter incrementing. (covers REQ-5)
- [ ] AC-7: register_qdisc integration test: BPF qdisc registered with name "myfifo"; `tc qdisc add dev eth0 root myfifo` works (uses BPF program for ops). (covers REQ-7)
- [ ] AC-8: Inner-qdisc test: BPF program graft inner pfifo onto a band; standard `tc qdisc replace dev eth0 parent <X>:1 fq_codel` replaces inner. (covers REQ-9)
- [ ] AC-9: Cilium-equivalent test: BPF qdisc implementing app-aware QoS; full Cilium QoS dataplane works at line rate. (covers REQ-1, REQ-4, REQ-5)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::sched::bpf_qdisc::BpfQdisc` — qdisc instance
- `kernel::net::sched::bpf_qdisc::StructOps` — `struct bpf_qdisc_ops` BPF struct_ops handle
- `kernel::net::sched::bpf_qdisc::Verify` — per-method verifier integration
- `kernel::net::sched::bpf_qdisc::Helpers` — kfunc set (enqueue_skb / dequeue_skb / skb_drop)
- `kernel::net::sched::bpf_qdisc::Register` — `register_qdisc` synthesis from struct_ops
- `kernel::net::sched::bpf_qdisc::Dump` — standard + custom NLA dump

### Locking and concurrency

- **Per-qdisc `q.lock`** (spinlock, inherited): held during enqueue/dequeue
- **Per-BPF struct_ops attach**: refcount-protected; updated via RCU
- **BPF-spinlock**: BPF programs can acquire BPF-spinlock for serialization within program

### Error handling

- `Err(EINVAL)` — bad NLA / bad FD
- `Err(EBADF)` — invalid struct_ops FD
- `Err(EPERM)` — non-CAP_BPF
- `Err(EOPNOTSUPP)` — verifier rejection
- BPF program errors: counted via per-CPU stats; not returned to caller

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| BPF struct_ops attach (refcount sound; verifier-checked before attach) | `kani::proofs::net::sched::bpf_qdisc::attach_safety` |
| BPF method invocation (per-method skb-bounds correct) | `kani::proofs::net::sched::bpf_qdisc::invoke_safety` |
| Helper bridges (`bpf_qdisc_enqueue_skb` → kernel skb_queue manipulation) | `kani::proofs::net::sched::bpf_qdisc::helper_safety` |
| `register_qdisc` synthesis (no NULL fn-ptr in synthesized ops) | `kani::proofs::net::sched::bpf_qdisc::register_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `kernel/bpf/00-overview.md` Layer 4 verifier-soundness theorem)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-qdisc BPF struct_ops | when attach succeeds, all required methods are implemented; unrequired methods may be NULL (kernel default) | `kani::proofs::net::sched::bpf_qdisc::ops_invariants` |
| Per-skb tracking | every skb passing through enqueue is either delivered to internal queue or dropped (no leak) | `kani::proofs::net::sched::bpf_qdisc::skb_invariants` |

### Layer 4: Functional correctness (depends on `kernel/bpf/00-overview.md`)

- **Verifier-soundness theorem** (cross-ref `kernel/bpf/00-overview.md`) — proves: any program accepted by verifier doesn't violate kernel-memory bounds + adheres to per-method preconditions/postconditions. Owned by BPF-verifier Tier-3, consumed here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-BPF-program-attach refcount uses `Refcount` (saturating); held while BPF qdisc instance is registered | § Mandatory |
| **CONSTIFY** | post-attach `Qdisc_ops` (synthesized from struct_ops) `static` once attached (immutable) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT**: see above
- **AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-Qdisc (cross-ref `net/sched/sch-api.md`)
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-CPU stats aggregation arithmetic uses checked operators
- **KERNEXEC**: BPF dispatch via JIT-mapped W^X memory; verifier-checked

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_BPF)` already required for struct_ops program load.
- LSM hook `security_capable(CAP_NET_ADMIN)` for `tc qdisc add ... bpf_qdisc`.
- Useful default GR-RBAC policy: deny BPF qdisc installation outside gradm-marked `bpf_users` role; arbitrary BPF in qdisc TX path is a frequent capability-escape vector.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — bpf_qdisc semantics are exhaustively specified by upstream + BPF struct_ops contract + Cilium qdisc test coverage)

## Out of Scope

- BPF verifier internals (cross-ref `kernel/bpf/00-overview.md`)
- BPF struct_ops mechanism (cross-ref `kernel/bpf/00-overview.md` Tier-3)
- Other qdiscs (cross-ref `net/sched/sch-fq-codel.md`, `sch-htb.md`, etc.)
- 32-bit-only paths
- Implementation code
