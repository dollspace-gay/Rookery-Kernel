# Tier-3: kernel/bpf/core.c — BPF runtime core (insn-array exec + JIT bridge + tail-calls + dispatcher + prog refcount)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/core.c
  - include/linux/bpf.h
  - include/linux/filter.h
-->

## Summary

The runtime layer underneath every loaded BPF program — what executes the verified instruction stream when a tracepoint fires, when an XDP packet arrives, when an iter walks, when a syscall hook triggers, when a struct_ops callback runs. Owns: per-prog `struct bpf_prog` allocation + refcount (`bpf_prog_get` / `bpf_prog_put`), the in-kernel BPF interpreter (`__bpf_prog_run` / `___bpf_prog_run` — the giant computed-goto switch executing the BPF ISA), the JIT bridge (`bpf_int_jit_compile` calling per-arch JIT to translate eBPF bytecode → native machine code), tail-call infrastructure (per-array indexing into `bpf_prog *`, fall-through if slot empty), the dispatcher (`bpf_dispatcher_*` — per-attach-point indirect-call thunk that chains N attached programs in a single text-poke-patched trampoline), program-array CO-RE (Compile-Once Run-Everywhere) link-time relocations, prog-fd-id mapping, prog-aux metadata storage, dead-code elimination via `bpf_jit_dump`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_prog` | per-loaded-program control block (insns, jited, refcount, aux) | `kernel::bpf::Prog` |
| `bpf_prog_alloc(size, gfp)` / `_alloc_no_stats` | allocate prog struct + insn buffer | `Prog::alloc` |
| `bpf_prog_realloc(prog, sz, gfp)` | grow insn buffer post-verifier | `Prog::realloc_insns` |
| `bpf_prog_free(prog)` | release prog (after RCU grace period) | `Prog::free` (Drop) |
| `bpf_prog_get(fd)` / `_inc[_not_zero]` / `_put` | refcount mgmt | `Arc<Prog>` get/put |
| `__bpf_prog_run(ctx, insn, stack)` | interpreter execution loop (computed goto) | `Prog::run_interpreter` |
| `bpf_int_jit_compile(prog)` | per-arch JIT bridge | `arch::bpf::jit_compile(prog)` |
| `bpf_jit_get_func_addr(prog, ...)` | resolve helper-call target | `Prog::resolve_helper` |
| `bpf_dispatcher_change_prog(d, from, to)` | text-poke patch dispatcher to chain progs | `Dispatcher::swap_prog` |
| `bpf_tail_call(ctx, &prog_array, index)` | tail-call dispatch | `Prog::tail_call` (handler) |
| `bpf_prog_array_alloc(n, gfp)` / `_free_sleepable` | alloc per-attach-point array of progs | `ProgArray::alloc` / `free` |
| `bpf_prog_array_copy(old, exclude, include, &new)` | mutate prog array | `ProgArray::with_modification` |
| `bpf_prog_array_run_array(array, ctx)` | run all progs in array, return last result | `ProgArray::run_all` |
| `bpf_prog_select_runtime(prog, &err)` | choose interpreter vs JIT path post-verify | `Prog::finalize_runtime` |
| `bpf_prog_kallsyms_add` / `_del` | register JIT'd code in /proc/kallsyms | `Prog::register_kallsyms` |
| `bpf_prog_pack_alloc(size, jit_fill_hole)` / `_free` | per-arch executable-page pack alloc | `arch::bpf::PackPool::alloc` |

## Compatibility contract

REQ-1: `__bpf_prog_run` interpreter executes the same eBPF ISA semantics as upstream — every verified instruction (32 ALU + 64 ALU + JMP + JMP32 + LD/ST/STX + atomic + helper-call + tail-call + ld_imm64 + bpf-to-bpf call) produces identical observable results.

REQ-2: JIT bridge compiles every verifier-accepted program to native code; JIT'd code result MUST equal interpreter result for identical input (verified per-program by interpreter-vs-JIT differential tests in selftests).

REQ-3: Per-prog refcount (`bpf_prog_get` family) byte-equivalent to upstream; close-during-execution safe (interpreter holds RCU grace + dispatcher uses synchronize_rcu_tasks before patching).

REQ-4: Tail-call: `bpf_tail_call` resolves `prog_array[index]` lazily; index out-of-bounds OR slot empty → fall-through (no crash). Tail-call-count cap (MAX_TAIL_CALL_CNT = 33) enforced; over-cap → return PROG_VOID.

REQ-5: Dispatcher: `bpf_dispatcher_change_prog` text-pokes the trampoline atomically; every observer either sees the OLD chain or the NEW chain (never partial). Defense against concurrent execution + patch.

REQ-6: prog_array semantics for cgroup-attached / kprobe-attached / tracepoint-attached / netdev-attached chains: every prog in the array runs in array order; per-attach-point return-value combinator (last-wins, or-zeroed, RET_PIPE) preserved.

REQ-7: `/proc/sys/net/core/bpf_jit_enable` (0=interpreter, 1=JIT, 2=JIT+disasm) byte-identical knob; `bpf_jit_kallsyms` + `bpf_jit_harden` knobs preserved.

REQ-8: BPF prog kallsyms entries appear in `/proc/kallsyms` for JIT'd progs (consumed by perf for stack symbolization).

## Acceptance Criteria

- [ ] AC-1: BPF interpreter test: load a small XDP_PASS-returning program, attach to veth, packet passes; interpreter mode (CONFIG_BPF_JIT_ALWAYS_ON=N + bpf_jit_enable=0) gives identical result to JIT mode.
- [ ] AC-2: tools/testing/selftests/bpf/test_progs runs the standard suite with same pass/fail manifest as upstream baseline.
- [ ] AC-3: tail-call test: 33-deep tail-call chain returns successfully; 34th depth correctly truncated (no crash, returns 0).
- [ ] AC-4: dispatcher stress: 100 BPF tracepoint attaches + detaches in tight loop while tracepoint fires from N CPUs; no segfault, no missed call, no double-call.
- [ ] AC-5: prog_array attach test: cgroup-skb prog array of 5 progs runs all 5 in attach order; reordering via prog_array_copy works.
- [ ] AC-6: `bpftool prog show` lists loaded progs with byte-identical output vs upstream.
- [ ] AC-7: perf stack-walk on JIT'd prog correctly resolves to BPF prog name via kallsyms.

## Architecture

`Prog` lives in `kernel::bpf::Prog`:

```
struct Prog {
  refcount: Refcount,
  len: u32,
  jited: AtomicBool,
  prog_type: BpfProgType,
  expected_attach_type: BpfAttachType,
  aux: KBox<ProgAux>,
  insns: VarLenArray<BpfInsn>,  // verifier-accepted insn stream
  jited_image: Option<NonNull<[u8]>>,  // JIT'd native code (executable page)
  bpf_func: AtomicPtr<unsafe fn(ctx: *const c_void) -> u64>,  // dispatch entry
}
```

Lifecycle:
1. Userspace calls `bpf(BPF_PROG_LOAD, &attr)` (cross-ref `kernel/bpf/syscall.md`).
2. syscall.c → verifier.c (cross-ref `kernel/bpf/verifier.md`) accepts.
3. `bpf_prog_select_runtime`:
   - If `bpf_jit_enable >= 1` AND arch supports JIT → `bpf_int_jit_compile(prog)` → fills `jited_image`, sets `bpf_func` to point at JIT'd entry, sets `jited=true`.
   - Else → `bpf_func` points at `__bpf_prog_run` interpreter.
4. Returns prog_fd to userspace.
5. On attach (cgroup / kprobe / tp / xdp / ...): `bpf_prog_array_run_array` chain installed via dispatcher.

Interpreter (`__bpf_prog_run`): the giant computed-goto switch over BPF opcodes. ~150 cases. State: 11 registers (R0..R10, R10=stack-frame), 512-byte stack, ctx pointer.

JIT bridge (`bpf_int_jit_compile`): per-arch implementation lives in `arch/x86/net/bpf_jit_comp.c`. Translates BPF insns to x86_64 native code in an executable page (allocated from per-arch BPF prog-pack pool). Returns image addr + len. Sets `prog->bpf_func = image_addr`.

Tail-call: `bpf_tail_call(ctx, &prog_array, index)`:
1. RCU read-side critical section.
2. `target = rcu_dereference(prog_array->ptrs[index])`.
3. If `target == NULL` OR `index >= prog_array->len` OR `tail_call_cnt >= MAX_TAIL_CALL_CNT` → fall-through (return).
4. Else `target->bpf_func(ctx)` (JIT or interpreter).
5. Increment tail_call_cnt (per-frame counter, NOT per-prog).

Dispatcher: per-attach-point trampoline that chains N attached programs. `bpf_dispatcher_change_prog(d, old, new)` patches the trampoline via `text_poke_bp_batch` (cross-ref `arch/x86/text-poke.md`). Observer sees either old chain or new — atomic single-instruction update.

prog_array (`struct bpf_prog_array`): array of `Arc<Prog>` plus per-prog `attach_flags`. Modified via copy-on-write (`bpf_prog_array_copy`); old array RCU-freed after grace.

Pack pool (per-arch executable-page allocator): grouping multiple JIT'd progs into shared 2MB-aligned executable pages improves TLB locality and reduces fragmentation. Per-arch fill-hole callback emits NOP/INT3 between progs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prog_no_uaf` | UAF | `Arc<Prog>` get-during-execution guarantees prog stays alive until call returns; close + concurrent execution never UAF. |
| `tailcall_no_overflow` | OVERFLOW | tail_call_cnt saturates at MAX_TAIL_CALL_CNT (33); cannot wrap. |
| `dispatcher_atomic_swap` | ATOMICITY | text-poke patch replaces trampoline atomically — observer sees old OR new, never partial. |
| `pack_pool_no_oob` | OOB | per-pack alloc bounds-checked; fill-hole emits valid INT3 padding. |

### Layer 2: TLA+

| Model | Owner |
|---|---|
| `models/bpf/dispatcher_swap.tla` | this doc (proves: text-poke patch atomicity vs concurrent execution from N CPUs; no observer sees torn instruction) |
| `models/bpf/tail_call_chain.tla` | this doc (proves: tail-call chain bounded by MAX_TAIL_CALL_CNT; out-of-bounds fall-through; concurrent prog_array_copy + tail_call serialized via RCU) |

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `prog->bpf_func` always points at either interpreter entry OR valid JIT image (never null after `bpf_prog_select_runtime`) | `Prog::finalize_runtime` |
| `prog_array->ptrs[i]` for `i < prog_array->len` is either NULL or valid `Arc<Prog>` (refcount-bumped) | `ProgArray` |
| JIT'd image satisfies W^X (RX-only after `set_memory_ro` + `set_memory_x`) | `arch::bpf::jit_compile` |

### Layer 4: Verus/Creusot functional

`__bpf_prog_run` ↔ JIT'd code semantic equivalence: for every verifier-accepted insn stream, interpreter result == JIT result for identical (ctx, prog) input. Encoded as Verus model: `forall ctx prog. interpret(prog, ctx) == jit(prog, ctx)`.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

bpf-core specific reinforcement:

- **W^X for JIT'd pages** — RW during JIT emit, then `set_memory_ro` + `set_memory_x` before publishing `prog->bpf_func`; permanent guarantee no JIT page is RW + X simultaneously (defense against Spectre v1 BPF-JIT-spray attacks).
- **bpf_jit_harden** (`/proc/sys/net/core/bpf_jit_harden=1`): blinds constants in JIT'd code via XOR-mask (defense against attacker placing exact-byte-value gadgets via JIT).
- **bpf_jit_limit** (`/proc/sys/net/core/bpf_jit_limit`): cap total JIT'd memory per-system + per-uid (default ~50% of memory; prevents BPF-JIT memory exhaustion).
- **Per-prog refcount saturating** — overflow saturates at u32::MAX; never wraps.
- **Tail-call cap = 33** unforgeable in JIT'd code (per-frame counter on stack, not in prog).
- **Dispatcher patch + RCU-tasks-trace synchronize** — patch waits for all in-flight prog executions to drain via `synchronize_rcu_tasks_trace` before freeing old prog (defense against UAF if old prog is still running on another CPU).
- **CO-RE relocations applied at load time only** — no runtime self-modifying code.
- **kallsyms entries** for JIT'd code labeled with prog name + tag; helps attribution in perf + crash dump.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- BPF verifier internals (covered in `kernel/bpf/verifier.md` future Tier-3)
- BPF syscall dispatch (covered in `kernel/bpf/syscall.md` future Tier-3)
- Per-arch JIT (`arch/x86/net/bpf_jit_comp.c` — separate Tier-3)
- BPF helper functions (covered in `kernel/bpf/helpers.md` future Tier-3)
- BPF map types (covered in `kernel/bpf/maps-*.md` future Tier-3s)
- 32-bit-only paths
- Implementation code
