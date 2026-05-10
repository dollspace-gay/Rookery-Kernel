# Tier-3: kernel/bpf/verifier.c — BPF static analyzer (insn safety + bounds + register tracking + path-pruning + helper signatures)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/verifier.c
  - kernel/bpf/check_btf.c
  - kernel/bpf/cfg.c
  - kernel/bpf/backtrack.c
  - kernel/bpf/const_fold.c
  - kernel/bpf/disasm.c
  - kernel/bpf/disasm.h
  - include/linux/bpf_verifier.h
-->

## Summary

The largest single file in `kernel/bpf/` (~20K lines). Every BPF program submitted via `bpf(BPF_PROG_LOAD, &attr)` must pass through this verifier before it can be JIT'd or interpreted. Performs:

- **Per-instruction safety check**: every load/store within bounds (per-pointer-type), every arithmetic operation tracks register min/max + signed/unsigned bounds, every conditional jump checked for fall-through reachability, every helper-call argument matches helper-signature, every map access typed by `bpf_map_type` schema.
- **Control-flow graph (CFG) construction**: insn-stream → CFG with explicit basic blocks; backward-edge detection (loops); back-edge bound (max 8 iterations of loop-bounded analysis).
- **Per-path register state tracking**: each register has type (NOT_INIT / SCALAR_VALUE / PTR_TO_CTX / PTR_TO_MAP_VALUE / PTR_TO_PACKET / PTR_TO_STACK / PTR_TO_BTF_ID / PTR_TO_FUNC / etc.) + per-type metadata (min/max bounds, ID for liveness tracking, off / range for pointer arithmetic).
- **Path pruning**: at convergence points, equivalent (or strictly-included) per-path states pruned to keep the verification state space tractable. Liveness analysis identifies unused registers.
- **BTF-based type checking**: with BTF, helper signatures (PTR_TO_BTF_ID arguments) cross-checked against actual struct types.
- **Helper-call validation**: every call to a kernel helper validated against per-helper `bpf_func_proto` (arg types, return type, side effects).
- **Tracepoint / kfunc dispatch validation**: per-attach-point + per-kfunc allowlist enforced.
- **Stack overflow check**: per-prog stack-depth bounded by MAX_BPF_STACK = 512 bytes.
- **Pointer arithmetic restrictions**: PTR_TO_MAP_VALUE + scalar ALU bounds-checked; PTR_TO_PACKET range-tracked across `pkt_end` comparisons.
- **Spectre-v1 hardening**: speculatively-executed paths bounds-clamped via `BPF_ALU64_REG_X(BPF_AND, REG, MASK)` insertions on conditional-branch boundaries.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_verifier_env` | per-verification environment | `kernel::bpf::verifier::Env` |
| `struct bpf_verifier_state` | per-path register + stack state | `verifier::State` |
| `struct bpf_reg_state` | per-register tracked state | `verifier::RegState` |
| `struct bpf_func_state` | per-function state (caller stack frame chain) | `verifier::FuncState` |
| `struct bpf_func_proto` | per-helper signature | `verifier::FuncProto` |
| `bpf_check(env, prog, ...)` | top-level entry | `Env::check` |
| `do_check(env)` | main verification loop | `Env::do_check` |
| `do_check_main(env)` | per-subprog driver | `Env::do_check_main` |
| `check_subprogs(env)` | enumerate sub-functions + verify each | `Env::check_subprogs` |
| `check_cfg(env)` | construct + validate CFG | `Env::check_cfg` |
| `check_helper_call(env, &insn, ...)` | per-helper-call validation | `Env::check_helper_call` |
| `check_kfunc_call(env, &insn, ...)` | per-kfunc-call validation | `Env::check_kfunc_call` |
| `check_func_call(env, &insn, ...)` | per-bpf-to-bpf call validation | `Env::check_func_call` |
| `check_mem_access(env, ...)` | per-load/store memory access check | `Env::check_mem_access` |
| `check_map_access(env, ...)` | per-map-value access | `Env::check_map_access` |
| `check_packet_access(env, ...)` | per-PTR_TO_PACKET access | `Env::check_packet_access` |
| `check_stack_access(env, ...)` | per-stack access | `Env::check_stack_access` |
| `check_ptr_alignment(env, ...)` | pointer alignment check | `Env::check_ptr_alignment` |
| `adjust_reg_min_max_vals(env, &insn)` | per-ALU bounds propagation | `Env::adjust_reg_min_max_vals` |
| `is_state_visited(env, insn_idx)` | path-pruning lookup | `Env::is_state_visited` |
| `push_stack(env, insn_idx, prev_insn_idx, speculative)` | push verification branch onto stack | `Env::push_stack` |
| `mark_reg_known(reg, val)` | mark register with concrete known value | `RegState::mark_known` |
| `mark_reg_unknown(env, regs, regno)` | mark register as scalar unknown | `RegState::mark_unknown` |
| `__check_ptr_off_reg(...)` | check pointer +/- offset bounds | `Env::check_ptr_off_reg` |
| `coerce_reg_to_size(reg, size)` | cast bounds to N-byte size | `RegState::coerce_to_size` |
| `is_branch_taken(reg, val, opcode, is_jmp32)` | conditional branch eval | `RegState::is_branch_taken` |
| `regsafe(env, rold, rcur, idmap, exact)` | per-register state inclusion check (for path pruning) | `RegState::is_subset_of` |
| `states_equal(env, old, cur, exact)` | per-state inclusion check | `State::is_subset_of` |
| `mark_chain_precision(env, regno)` | back-track to mark instructions affecting reg precision | `Env::mark_chain_precision` |
| `do_misc_fixups(env)` | post-verification fixups (insn rewrites for sandboxing) | `Env::do_misc_fixups` |

## Compatibility contract

REQ-1: Every BPF program upstream-accepted MUST be Rookery-accepted (no false rejections of valid programs); every program upstream-rejected MUST be Rookery-rejected (no security regressions accepting unsafe programs).

REQ-2: `BPF_PROG_LOAD` returns -EACCES with byte-identical error log strings (`bpf_log` output) for the same rejected program; userspace tools (libbpf, bpftool) parse error messages identically.

REQ-3: Per-helper signatures (`bpf_func_proto`) byte-identical for every in-tree helper (~200 helpers); arg-type compatibility table preserved.

REQ-4: BTF-based type checking: programs loaded with BTF blob get full PTR_TO_BTF_ID validation; allows access to kernel struct fields via verified field offsets.

REQ-5: Per-path register state tracking matches upstream's tnum (tristate-number) representation: per-bit known-0 / known-1 / unknown; min/max ranges propagated.

REQ-6: CFG complexity limit: max 1M insns analyzed (BPF_COMPLEXITY_LIMIT_INSNS); programs over limit rejected. Loop iterations bounded (BPF_COMPLEXITY_LIMIT_STATES = 64K); state-equivalence pruning keeps tractable.

REQ-7: Pointer arithmetic on PTR_TO_PACKET requires explicit `pkt_end` comparison before deref; pointer wrap-around blocked.

REQ-8: Helper-call attach-point allowlist: per-prog-type, only helpers in `bpf_<type>_func_proto` callback returning non-NULL allowed.

REQ-9: kfunc registration via `BTF_KFUNCS_START/END` — verifier consults per-prog-type kfunc allowlist; non-allowed kfuncs rejected.

REQ-10: `do_misc_fixups` performs post-verification insn rewrites (e.g., insert AND-MASK insns for Spectre-v1 hardening, replace `bpf_get_func_ip` with helper-id, inline `bpf_loop` for performance).

REQ-11: Stack-depth check: per-subprog stack-depth + chained call stack <= MAX_BPF_STACK (512 bytes per frame, max 8 nested = 4096 total).

REQ-12: Spec-v1 mitigation: insertion of bounds-clamping AND-MASK instructions after every bounded-pointer ALU + after conditional branches that the speculative-execution might mis-predict.

## Acceptance Criteria

- [ ] AC-1: `tools/testing/selftests/bpf/test_progs` runs the standard suite with same pass/fail manifest as upstream baseline.
- [ ] AC-2: `tools/testing/selftests/bpf/test_verifier` runs the verifier-test suite (~600 tests) with same accept/reject manifest as upstream.
- [ ] AC-3: `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm]=count() }'` loads + runs.
- [ ] AC-4: Complex prog test: large XDP program (multiple subprogs, helper calls, map lookups, packet parsing) loads + JIT'd.
- [ ] AC-5: BTF test: program using `bpf_d_path` with PTR_TO_BTF_ID arg loads + path-walk works.
- [ ] AC-6: Rejection test: program with out-of-bounds map access, stack overflow, unbounded loop all rejected with byte-identical error messages.
- [ ] AC-7: Spec-v1 mitigation observable: post-verification insn dump shows AND-MASK insns inserted at bounds-check sites.

## Architecture

`Env` lives in `kernel::bpf::verifier::Env`:

```
struct Env {
  prog: Arc<Prog>,
  insn_aux_data: VarLenArray<InsnAux>,
  explored_states: KBox<HashTable<HashedState>>,
  state: KBox<State>,                 // current verification state
  free_list: AtomicPtr<State>,         // recycled state structs
  stack: KVec<KBox<State>>,            // verification stack of branches
  log: KBox<BpfVerifierLog>,
  insn_idx: u32,
  prev_insn_idx: u32,
  scratched_regs: u64,
  scratched_stack_slots: u64,
  passes: u32,
  total_states: u32,
  peak_states: u32,
  longest_mark_read_walk: u32,
  test_state_freq: bool,
}

struct State {
  parent: Option<Arc<State>>,
  curframe: u32,
  active_locks: HashTable<...>,
  active_rcu_lock: bool,
  used_btf: KVec<UsedBtf>,
  used_map_cnt: u32,
  branches: u32,                       // number of branches still to verify from this state
  insn_idx: u32,
  jmp_history: KVec<JmpHistoryEntry>,
  speculative: bool,
  frame: [Option<Arc<FuncState>>; MAX_CALL_FRAMES],
  callback_unroll_depth: u32,
}

struct RegState {
  type_: BpfRegType,                   // NOT_INIT / SCALAR_VALUE / PTR_TO_CTX / ...
  off: i32,                            // pointer offset
  range: u32,                          // PTR_TO_PACKET range to pkt_end
  map_ptr: Option<Arc<BpfMap>>,
  raw: KBox<RawRegState>,              // per-type metadata union
  id: u32,                             // for liveness tracking
  ref_obj_id: u32,                     // for refcounted objects
  smin_value: i64,
  smax_value: i64,
  umin_value: u64,
  umax_value: u64,
  s32_min_value: i32,
  s32_max_value: i32,
  u32_min_value: u32,
  u32_max_value: u32,
  var_off: Tnum,                       // tristate-number per-bit known
  parent: Option<Arc<RegState>>,
  frameno: u32,
  precise: bool,
  live: u8,                            // REG_LIVE_NONE / _READ / _WRITTEN / _DONE
}
```

Verification flow `Env::check`:
1. **Initial setup**: `add_subprog_and_kfunc(env)` enumerates subprog boundaries.
2. **CFG check** (`check_cfg`): traverse insn-stream, build CFG, validate no unreachable code, no jumps to mid-insn, all subprog calls match return targets.
3. **Per-subprog verification**: `do_check_main` iterates over insns:
   - For each insn at `env->insn_idx`:
     - `check_insn(env, insn)` → per-opcode handler.
     - Loads: `check_mem_access(env, regno, off, size, BPF_READ)` → bounds-check per-reg-type.
     - Stores: `check_mem_access(env, regno, off, size, BPF_WRITE)`.
     - ALU: `adjust_reg_min_max_vals(env, &insn)` → propagate bounds.
     - JMP: `is_branch_taken(reg, val, opcode, is_jmp32)`:
       - If always-taken / always-not-taken: continue with single path.
       - Else: `push_stack(env, target_idx, env->insn_idx, false)` → branch verification, fall-through on current path.
     - CALL helper: `check_helper_call(env, &insn, ...)` → validate args against `bpf_func_proto`.
     - CALL kfunc: `check_kfunc_call(env, &insn, ...)` → validate against per-prog-type allowlist + BTF type check.
     - CALL bpf: `check_func_call(env, &insn, ...)` → push new frame.
     - RET: pop frame; on outermost return, check return-value type matches prog-type.
     - At each insn entry: `is_state_visited(env, insn_idx)` → if equivalent state seen before, prune (don't re-verify).
   - After insn processing: pop next branch from `env->stack` and verify until empty.
4. **Misc fixups** (`do_misc_fixups`): insert AND-MASK insns for Spec-v1, inline bpf_loop, replace privileged helpers with constants.
5. Return success (or -EACCES with bpf_log error trail).

Path-pruning: each visited insn_idx caches the per-path State; on re-visit, `states_equal(env, old, cur, exact=false)` checks if current state is a subset of cached (i.e., if old verified without issue, cur also OK). If subset → skip verification for this path.

Pruning is critical for tractability: typical XDP program with N branches has 2^N paths but pruning collapses to O(N).

Tnum (tristate number): per-register `var_off` field tracks per-bit known-0 / known-1 / unknown. After `r1 = r2 & 0xff`: r1.var_off bits 8..63 known-0, bits 0..7 unknown. Used for precise bounds-arithmetic propagation.

Liveness tracking: each insn marked with which registers it READs or WRITEs; backwards walk identifies regs whose state matters at each point. `mark_chain_precision` tracks back through ALU chain when an exact-value comparison demands precise tracking.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `verifier_terminates` | TERMINATION | per-prog total state-count bounded by BPF_COMPLEXITY_LIMIT_STATES (64K); per-path bounded by BPF_COMPLEXITY_LIMIT_INSNS (1M); total processing time bounded. |
| `reg_bounds_no_overflow` | OVERFLOW | smin/smax/umin/umax arithmetic checked for overflow during per-ALU propagation; saturate to i64::MAX/MIN on overflow. |
| `state_no_uaf` | UAF | per-state Arc refcount + RCU-defer free for prune cache; no UAF on lookup-after-free. |
| `tnum_consistency` | INVARIANT | tnum (mask, value) maintains invariant: `mask & value == 0` (no bit can be both known-0 and known-1). |

### Layer 2: TLA+

(Not declared specifically — verifier termination + soundness covered by parent's verifier-soundness model declaration `models/bpf/verifier_soundness.tla`.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Env::check` post (success): every accepted insn has all reg-states well-typed; every load/store bounded; every helper-call args match proto | `Env::check` |
| `is_state_visited(insn_idx)` post: returned cached state IS a superset of every concrete execution that would reach insn_idx via current path | `Env::is_state_visited` |
| Per-helper-call: arg-types after call match proto's documented out-types | `Env::check_helper_call` |
| Stack-depth invariant: at any insn, sum of frame stack-depths ≤ MAX_BPF_STACK | `Env::check_stack_access` |

### Layer 4: Verus/Creusot functional

Verifier soundness: a program accepted by Rookery's verifier, when executed by interpreter or JIT, never produces UAF / OOB-read / OOB-write / unbounded loop / type confusion. Encoded as Verus model-check over per-insn semantics: `forall prog. accepts(prog) ==> safe_execution(prog)`. Decisive proof of the central security guarantee.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

verifier-specific reinforcement:

- **Spec-v1 mitigation default-on** — AND-MASK insertion at all bounds-clamp sites; defense against Spectre-v1 attacker using BPF as gadget-source.
- **`bpf_jit_harden` integration** — when sysctl set, additional const-blinding via XOR-mask in JIT; defense against attacker placing exact byte values in JIT code.
- **Per-prog complexity limits** — BPF_COMPLEXITY_LIMIT_INSNS = 1M, BPF_COMPLEXITY_LIMIT_STATES = 64K; prevents verifier-DoS from pathological program.
- **Helper allowlist per-prog-type** — strict; no cross-type helper smuggling.
- **kfunc allowlist per-prog-type** — same.
- **PTR_TO_BTF_ID nullable check** — per-arg `arg_type & PTR_MAYBE_NULL` requires explicit null-check before deref.
- **Speculative-path verification** — paths only reachable via mis-speculation also verified for safety; defense against attacker constructing program where speculative-only path leaks data.
- **Stack-overflow detection** — per-frame stack <= MAX_BPF_STACK; chained calls track sum; defense against deeply-nested call exhausting kernel stack.
- **Verifier log size cap** — per-prog log buffer max 4MB; defense against verbose-error-flood DoS.
- **Per-uid + per-program unprivileged limits** — when `kernel.unprivileged_bpf_disabled=0`, additional restrictions (no ptr-to-packet, no certain helpers).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- BPF runtime (covered in `kernel/bpf/bpf-core.md` parent Tier-3)
- BPF JIT (covered in `arch/x86/bpf-jit.md` future Tier-3)
- BPF maps (covered in `kernel/bpf/maps-*.md` future Tier-3s)
- BPF helpers (covered in `kernel/bpf/helpers.md` future Tier-3)
- BTF parser (covered in `kernel/bpf/btf.md` future Tier-3)
- 32-bit-only paths
- Implementation code
