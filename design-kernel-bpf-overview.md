---
title: "Tier-2: kernel/bpf — eBPF (verifier, JIT, helpers, kfuncs, maps, prog types)"
tags: ["design-doc", "tier-2", "kernel", "bpf"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for eBPF — Linux's universal in-kernel programmable virtual machine. Used by:
- **Cilium / Calico Express dataplane / Katran** — XDP/TC dataplane for container networking
- **Tetragon / Falco** — runtime security observability via tracepoints + LSM hooks
- **bpftrace / bpf_probe / bcc** — kernel tracing
- **iptables-nft + nftables `meta` lookup** — programmable packet classification
- **Cilium service-routing**, **netflowd**, **bperf** — observability dataplanes
- Per-socket filters (BPF socket filters since Berkeley days; eBPF since 2014)

This is the highest-impact subsystem for modern Linux: every container-networking dataplane + tracing tool depends on eBPF.

The core primitives:
- **Programs**: bytecode compiled by clang (eBPF is the target ISA); loaded via `bpf(BPF_PROG_LOAD, ...)`; verified by the in-kernel verifier; JIT-compiled to native code (or run via interpreter)
- **Maps**: typed key-value stores shared between programs + userspace; many types — HASH, ARRAY, LRU_HASH, PERCPU_HASH, SOCKMAP, SOCKHASH, DEVMAP, CPUMAP, RINGBUF, ARENA, ...
- **Helpers**: kernel functions exposed to BPF programs (per-prog-type allow-list)
- **Kfuncs**: typed kernel functions exposed to BPF; per-prog-type registration with TRUSTED_ARGS / RET_NULL / ACQUIRE / RELEASE flags (verifier-checked refcount pairing)
- **BTF**: BPF Type Format — kernel + program type info for verifier + per-arg type checking + CO-RE (Compile Once Run Everywhere)
- **Trampolines**: function-call dispatch wrappers used by BPF_PROG_TYPE_TRACING + BPF_PROG_TYPE_LSM + BPF_PROG_TYPE_STRUCT_OPS
- **struct_ops**: BPF programs implement vtable methods (used by net/sched/bpf_qdisc, congestion control, etc.)
- **Iterators**: BPF programs that iterate kernel data structures (per-pid, per-conntrack, per-cgroup, per-task) via `bpf_iter`
- **LSM hooks**: BPF programs attached to LSM hooks (KRSI / Tetragon foundation)

40+ program types (BPF_PROG_TYPE_*) covering: socket filters, kprobes, tracepoints, perf events, XDP, tc-cls, tc-act, cgroup-sock, cgroup-sk-msg, cgroup-skb, sk_skb, socket-ops, lirc, lwt, sk_lookup, tracing, lsm, struct_ops, sched, raw_tracepoint, perf_event, sched_act, and more.

30+ map types covering: HASH, ARRAY, PROG_ARRAY, PERF_EVENT_ARRAY, PERCPU_*, STACK_TRACE, CGROUP_ARRAY, LRU_HASH, LRU_PERCPU_HASH, LPM_TRIE, ARRAY_OF_MAPS, HASH_OF_MAPS, DEVMAP, SOCKMAP, CPUMAP, XSKMAP, SOCKHASH, CGROUP_STORAGE, REUSEPORT_SOCKARRAY, PERCPU_CGROUP_STORAGE, QUEUE, STACK, SK_STORAGE, DEVMAP_HASH, STRUCT_OPS, RINGBUF, INODE_STORAGE, TASK_STORAGE, BLOOM_FILTER, USER_RINGBUF, CGRP_STORAGE, ARENA.

Sub-tier-2 of `kernel/00-overview.md`. Cross-referenced extensively from net/sched/cls-bpf.md, net/sched/sch-bpf.md, net/xfrm/bpf.md, net/netfilter/conntrack-bpf.md, net/netfilter/flowtable.md, net/sched/act-bpf.md.

### Out of Scope

- Per-Tier-3 child docs (Phase D)
- 32-bit-only paths
- Implementation code

### scope

This Tier-2 governs all of `/home/doll/linux-src/kernel/bpf/` (~80 source files), `net/core/filter.c` (BPF socket filter consumer), `lib/bpf_*` helpers, plus the public API + UAPI.

### compatibility contract — outline

### `bpf(2)` syscall

Single syscall with command codes:
- `BPF_MAP_CREATE` / `_LOOKUP_ELEM` / `_UPDATE_ELEM` / `_DELETE_ELEM` / `_GET_NEXT_KEY` / `_LOOKUP_AND_DELETE_ELEM` / `_LOOKUP_BATCH` / `_UPDATE_BATCH` / `_DELETE_BATCH` / `_FREEZE`
- `BPF_PROG_LOAD` / `_TEST_RUN` / `_GET_NEXT_ID` / `_PROG_GET_FD_BY_ID` / `_GET_INFO_BY_FD` / `_BIND_MAP` / `_QUERY` / `_RAW_TRACEPOINT_OPEN`
- `BPF_LINK_CREATE` / `_UPDATE` / `_GET_FD_BY_ID` / `_GET_NEXT_ID` / `_DETACH`
- `BPF_OBJ_PIN` / `_GET` (pinning to bpffs)
- `BPF_BTF_LOAD` / `_GET_FD_BY_ID` / `_GET_NEXT_ID` / `_GET_INFO_BY_FD`
- `BPF_TASK_FD_QUERY`
- `BPF_PROG_ATTACH` / `_DETACH` / `_QUERY` (cgroup attach)
- `BPF_ITER_CREATE`
- `BPF_ENABLE_STATS`

Wire format byte-identical so libbpf works.

### Verifier (`verifier.c`)

The verifier is the kernel's static analyzer for BPF bytecode. Per program load:
1. Build CFG (control flow graph) from bytecode
2. Symbolic execution per all reachable paths
3. Per-instruction type tracking (register types, pointer ranges, scalar ranges)
4. Per-helper / per-kfunc signature checking
5. Per-acquire kfunc must be paired with release on every program path (refcount safety)
6. Loop detection: bounded loops only (BPF_MAY_GOTO + iter helpers)
7. Stack + map access bounds
8. Speculative execution + Spectre-class mitigations (per `KERNEXEC` / verifier-side branch-pruning)

Per-program-type verifier policies (e.g., XDP allows direct packet access; SOCK_FILTER doesn't).

### JIT (`arch/x86/net/bpf_jit_comp.c` + per-arch)

Per-arch BPF→native compiler. x86_64 produces `mov`/`add`/`sub`/`call`/etc.; ARM64 / RISC-V / ppc64 / s390 each have their own JIT.

Fallback: in-kernel interpreter (when `CONFIG_BPF_JIT=n` or per-arch JIT unavailable).

### BTF (BPF Type Format)

Kernel + per-program type information emitted as ELF section `.BTF`. Contains struct/union/enum/func/typedef definitions. Used for:
- Verifier per-arg type checking (kfuncs declare their arg-types via BTF)
- CO-RE (Compile Once Run Everywhere) — userspace BPF program references kernel struct fields by name; libbpf relocates at load using BTF
- BPF iterators (per-iter type info)

### Map types (subset)

| Map type | Purpose |
|---|---|
| `BPF_MAP_TYPE_HASH` | Generic hashtable |
| `_ARRAY` | Fixed-size array indexed by integer |
| `_PROG_ARRAY` | Tail-call dispatch table |
| `_PERF_EVENT_ARRAY` | Per-CPU perf-event ring |
| `_PERCPU_HASH` / `_PERCPU_ARRAY` | Per-CPU sharded variants |
| `_STACK_TRACE` | Pre-allocated stack-trace storage |
| `_CGROUP_ARRAY` | Per-cgroup id array |
| `_LRU_HASH` / `_LRU_PERCPU_HASH` | LRU-evicting hash |
| `_LPM_TRIE` | Longest-prefix-match trie (for routes) |
| `_DEVMAP` / `_DEVMAP_HASH` / `_CPUMAP` / `_XSKMAP` | XDP redirect maps |
| `_SOCKMAP` / `_SOCKHASH` | Socket maps for sk_skb/sk_msg programs |
| `_CGROUP_STORAGE` / `_PERCPU_CGROUP_STORAGE` | Per-cgroup-program storage |
| `_REUSEPORT_SOCKARRAY` | SO_REUSEPORT load balancing |
| `_QUEUE` / `_STACK` | FIFO / LIFO |
| `_SK_STORAGE` / `_INODE_STORAGE` / `_TASK_STORAGE` / `_CGRP_STORAGE` | Per-object storage |
| `_RINGBUF` / `_USER_RINGBUF` | Lockless multi-producer multi-consumer ring buffer |
| `_BLOOM_FILTER` | Approximate set membership |
| `_STRUCT_OPS` | BPF programs implementing kernel vtable |
| `_ARENA` | Mmappable user-managed BPF address space |

Per-map UAPI byte-identical.

### Program types (subset)

| Prog type | Attach point |
|---|---|
| `BPF_PROG_TYPE_SOCKET_FILTER` | setsockopt SO_ATTACH_FILTER |
| `_KPROBE` | kprobe + kretprobe |
| `_TRACEPOINT` | tracepoints |
| `_SCHED_CLS` | tc filter |
| `_SCHED_ACT` | tc action |
| `_XDP` | netdev XDP hook |
| `_PERF_EVENT` | perf event |
| `_CGROUP_SKB` / `_CGROUP_SOCK` / `_CGROUP_SK_MSG` / `_CGROUP_DEVICE` | per-cgroup hooks |
| `_LWT_*` | light-weight tunnel |
| `_SK_SKB` / `_SK_MSG` | sockmap hooks |
| `_RAW_TRACEPOINT` | raw tracepoint |
| `_LSM` | LSM hook |
| `_SK_LOOKUP` | socket-listener lookup |
| `_STRUCT_OPS` | kernel vtable methods |
| `_TRACING` | fentry/fexit/freplace |
| `_NETFILTER` | netfilter hook |

Per-prog-type per-context BTF type. Identical UAPI.

### Helpers + kfuncs

Per-prog-type allow-list of kernel functions. Helpers (legacy): per-numeric-id; signature in struct `bpf_func_proto`. Kfuncs (modern): per-named function; signature via BTF; per-flag `KF_ACQUIRE` / `_RELEASE` / `_TRUSTED_ARGS` / `_RET_NULL`.

### Trampolines + struct_ops

`bpf_trampoline.c`: dynamic call-site trampolines for fentry/fexit/lsm. Per-attach-point BPF program is wrapped in trampoline.

`bpf_struct_ops.c`: BPF programs implement kernel vtable methods (e.g., TCP congestion control, sched_qdisc). Per-struct_ops registration creates a kernel-side `Qdisc_ops` (or similar) backed by BPF.

### Pinning to bpffs

`mount -t bpf bpf /sys/fs/bpf`. `bpf(BPF_OBJ_PIN, ...)` pins program / map / link to a path. Allows daemon restart without losing programs.

### `bpf_link` lifecycle

Per-attach: kernel returns a `bpf_link` fd that owns the attachment. Closing fd detaches (refcount-drop). Used for cgroup-sock-attach, XDP-attach, etc.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `kernel/bpf/syscall.md` | bpf(2) syscall + map/prog/link CRUD |
| `kernel/bpf/verifier.md` | verifier core (CFG + symbolic execution + speculative-mitigation) |
| `kernel/bpf/btf.md` | BTF format + relocation |
| `kernel/bpf/helpers.md` | helper function registry |
| `kernel/bpf/kfuncs.md` | kfunc registry + verifier flags |
| `kernel/bpf/maps.md` | map type registry + per-type implementations |
| `kernel/bpf/trampoline.md` | dynamic call-site trampolines |
| `kernel/bpf/struct_ops.md` | struct_ops BPF-implements-vtable |
| `kernel/bpf/lsm.md` | BPF-LSM hooks (KRSI) |
| `kernel/bpf/iter.md` | bpf_iter for kernel-data iteration |
| `kernel/bpf/cgroup-attach.md` | per-cgroup BPF attach |
| `kernel/bpf/jit-x86.md` | x86_64 JIT compiler (cross-ref `arch/x86/`) |
| `kernel/bpf/inode.md` | bpffs (BPF filesystem) |
| `kernel/bpf/sockmap.md` | sockmap / sockhash for sk_skb/sk_msg |
| `kernel/bpf/devmap.md` | XDP redirect maps |
| `kernel/bpf/ringbuf.md` | ringbuf + user_ringbuf |
| `kernel/bpf/arena.md` | mmappable BPF arena |

### compatibility outline (top-level)

- REQ-O1: `bpf(2)` syscall + all BPF_* command codes parsed identically.
- REQ-O2: Verifier per-program-type semantics + per-helper/kfunc signature checking + refcount-acquire/release pairing identical.
- REQ-O3: All map types + program types per the tables above.
- REQ-O4: BTF format byte-identical (libbpf + bpftool depend on this).
- REQ-O5: Per-prog-type helper/kfunc allow-list identical.
- REQ-O6: Trampoline + struct_ops infrastructure identical.
- REQ-O7: bpffs pinning + bpf_link lifecycle identical.
- REQ-O8: Per-arch JIT compiler produces functionally-equivalent native code.
- REQ-O9: Hardening: row-1 features applied; verifier-driven refcount safety gives compile-time UAF/leak prevention.
- REQ-O10: TLA+ models declared at this Tier-2 (verifier soundness — see § Verification Layer 2).

### acceptance criteria (top-level)

- [ ] AC-O1: `bpftool prog list` + `bpftool map list` byte-identical content for equivalent programs/maps. (covers REQ-O1)
- [ ] AC-O2: Verifier rejects programs with use-after-free / unbounded loops / out-of-bounds access; accepts equivalent valid programs. (covers REQ-O2)
- [ ] AC-O3: Each map type creatable + usable; each program type loadable + attachable. (covers REQ-O3)
- [ ] AC-O4: bpftool btf dump + libbpf load round-trip works. (covers REQ-O4)
- [ ] AC-O5: Cilium / bpftrace / Tetragon work end-to-end. (covers REQ-O5, REQ-O7)
- [ ] AC-O6: struct_ops example (e.g., BPF qdisc) works. (covers REQ-O6)
- [ ] AC-O7: Per-Tier-3 ACs cumulatively cover BPF ABI. (meta-AC)
- [ ] AC-O8: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O9)

### architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::bpf::syscall::Syscall` — bpf(2) entry
- `kernel::bpf::Verifier` — verifier core
- `kernel::bpf::Btf` — BTF format
- `kernel::bpf::Helpers` — helper registry
- `kernel::bpf::Kfuncs` — kfunc registry
- `kernel::bpf::map::*` — per-map-type implementations
- `kernel::bpf::Trampoline` — call-site trampolines
- `kernel::bpf::StructOps`
- `kernel::bpf::Lsm`
- `kernel::bpf::Iter`
- `kernel::bpf::Bpffs` — BPF filesystem

### verification (top-level)

### Layer 1: Kani SAFETY proofs
Per Tier-3.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/bpf/verifier_soundness.tla` | `kernel/bpf/verifier.md` (proves: any program accepted by verifier doesn't violate kernel-memory access bounds; KF_ACQUIRE/RELEASE are correctly paired on every reachable program path) |
| `models/bpf/jit_correctness.tla` | `kernel/bpf/jit-x86.md` (per-arch JIT preserves bytecode semantics — JIT-compiled native code produces same effect as interpreter) |

### Layer 3: invariant harnesses
Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **Verifier soundness theorem** via TLA+ refinement — owned by `kernel/bpf/verifier.md`
- **JIT-vs-interpreter equivalence** via Verus — proves: per-bytecode-op, JIT-compiled native code's effect equals interpreter's

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** + verifier-enforced KF_ACQUIRE/RELEASE pairing | § Mandatory + verifier-checked |
| **AUTOSLAB** for per-prog + per-map slab | § Mandatory |
| **MEMORY_SANITIZE** for freed map values + per-prog-stack | § Default-on configurable off |
| **CONSTIFY** for `bpf_func_proto` + `btf_kfunc_id_set` registrations | § Mandatory |
| **KERNEXEC** — JIT-mapped W^X memory; verifier-checked | § Mandatory |
| **MPROTECT carve-out** — JIT-runtime exemption per `00-security-principles.md` Q3 (already locked-in) | § Default-on configurable off, per-process JIT exemption |

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_BPF)` for program load + map create.
- LSM hook `security_capable(CAP_NET_ADMIN)` for network-prog-type loads.
- Default useful GR-RBAC policy: deny BPF program load outside gradm-marked `bpf_admin` role; BPF is the most powerful kernel-extension primitive.

