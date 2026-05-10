---
title: "Tier-3: arch/x86/net/bpf_jit_comp.c — x86_64 BPF JIT compiler"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **x86_64 BPF JIT** translates verifier-approved eBPF bytecode into native x86-64 machine code so BPF programs execute at near-native speed in lieu of the in-kernel interpreter. Per-prog: `bpf_int_jit_compile(env, prog)` is invoked from the verifier finalize path and walks `prog->insnsi` instruction-by-instruction over multiple passes until image-length convergence. Per-image: a two-mapping `struct bpf_binary_header` is allocated by `bpf_jit_binary_pack_alloc` — one writable mapping (`rw_image`) and one ROX-execable mapping (`image`) backed by the same module-memory pages, copied across via `bpf_arch_text_copy`/`text_poke_copy`. Per-instruction: a `switch (insn->code)` dispatches over BPF ALU/ALU64/JMP/JMP32/LD/LDX/ST/STX/CALL/EXIT classes and emits the equivalent x86-64 byte sequence via `EMIT*` macros. Per-bpf-register: `reg2hex[]` maps `BPF_REG_0..9`/`AUX_REG`/`X86_REG_R9`/`X86_REG_R12` to x86-64 register numbers (e.g. `BPF_REG_0` → RAX, `BPF_REG_6` → RBX callee-saved). Per-tail-call: the JIT emits inline tail-call dispatch — indirect (table-walk on `struct bpf_array`) and direct (poke-patched static jump) — bounded by `MAX_TAIL_CALL_CNT` via a stack slot. Per-prologue: pushes RBP, sets RBP=RSP, subtracts `round_up(stack_depth, 8)`, optionally pushes R12 + callee-saved regs and zeros tail_call_cnt. Per-call: emits a 5-byte `E8 rel32` (call) or `E9 rel32` (jump), with `emit_rsb_call` interposing per-`X86_FEATURE_CALL_DEPTH` accounting and `emit_indirect_jump` selecting between retpoline thunks, LFENCE-prefixed indirect jumps, ITS static thunks, or bare indirect jumps based on `X86_FEATURE_RETPOLINE*` cpu features. Per-atomic: emits `LOCK`-prefixed RMW (`BPF_ADD`/`BPF_AND`/`BPF_OR`/`BPF_XOR`) plus `XCHG`/`CMPXCHG`/`FETCH` and acquire/release loads/stores. Per-ftrace/kfunc: trampoline emission via `__arch_prepare_bpf_trampoline` builds fentry/fexit/modify-return wrappers; static-keys + `bpf_arch_text_poke` patch call-sites atomically via `text_poke`. Per-CFI: `emit_cfi`/`emit_kcfi`/`emit_fineibt` emit per-program hashes for kernel CFI + FineIBT. Per-feature-query: `bpf_jit_supports_*` advertise capabilities to the verifier (`kfunc_call`, `subprog_tailcalls`, `percpu_insn`, `exceptions`, `private_stack`, `arena`, `ptr_xchg`, `timed_may_goto`, `fsession`, `insn` per-arena filter). Critical for: production BPF performance, XDP packet-rate, tracing-overhead reduction, and dataplane competitiveness with kernel C code.

This Tier-3 covers `arch/x86/net/bpf_jit_comp.c` (~4083 lines).

### Acceptance Criteria

- [ ] AC-1: bpf_int_jit_compile on a 3-insn ALU prog produces image where bpf_func returns expected value.
- [ ] AC-2: Multi-pass loop converges within MAX_PASSES; final proglen stable.
- [ ] AC-3: BPF_TAIL_CALL direct + indirect respect MAX_TAIL_CALL_CNT (33) via tcc_ptr slot.
- [ ] AC-4: BPF_LDX|BPF_PROBE_MEM faulting load: extable entry triggers ex_handler_bpf — clears dst, advances RIP.
- [ ] AC-5: BPF_STX|BPF_ATOMIC ADD/AND/OR/XOR emitted with LOCK prefix; XCHG without.
- [ ] AC-6: CMPXCHG: 0xF0 0x0F 0xB1 emitted; ZF-failure retry path closes correctly.
- [ ] AC-7: BPF_CALL to kfunc emits 5-byte E8 rel32 within ±2GiB of bpf_func.
- [ ] AC-8: Indirect call/jump emits retpoline thunk when X86_FEATURE_RETPOLINE.
- [ ] AC-9: emit_return uses x86_return_thunk when cpu_wants_rethunk; else `C3` (+ INT3 if MITIGATION_SLS).
- [ ] AC-10: bpf_arch_text_poke: NOP→CALL→NOP cycle observable; module addresses rejected.
- [ ] AC-11: bpf_jit_supports_exceptions == CONFIG_UNWINDER_ORC.
- [ ] AC-12: arch_prepare_bpf_trampoline emits frame-saving stub + fentry call sequence; orig function invoked.
- [ ] AC-13: Arena prog (BPF_PROBE_MEM32) JIT'ed with R12 as base; arena fault recorded with offset.
- [ ] AC-14: bpf_jit_free returns image pages to module memory; priv_stack guards checked.
- [ ] AC-15: CFI preamble (kCFI/FineIBT) emitted on subprog + main with cfi_bpf_hash.

### Architecture

```
struct BpfBinaryHeader {
  size: u32,
  /* followed by image bytes (ROX) */
}

struct X64JitData {
  header: *BpfBinaryHeader,
  rw_header: *BpfBinaryHeader,
  ctx: JitContext,
  proglen: u32,
  image: *u8,
  addrs: *i32,                     // [insn_cnt+1]
}

struct JitContext {
  cleanup_addr: u32,
  tail_call_direct_label: i32,
  tail_call_indirect_label: i32,
  prog_offset: i32,
}
```

`Jit::int_jit_compile(env, prog) -> *BpfProg`:
1. /* JIT enabled? */
2. if !prog.jit_requested: return prog.
3. jit_data = prog.aux.jit_data ?: alloc(X64JitData).
4. /* Priv-stack alloc */
5. if !aux.priv_stack_ptr ∧ aux.jits_use_priv_stack:
   - priv_stack_alloc_sz = round_up(aux.stack_depth, 8) + 2 * PRIV_STACK_GUARD_SZ.
   - priv_stack_ptr = __alloc_percpu_gfp(priv_stack_alloc_sz, 8, GFP_KERNEL).
   - priv_stack_init_guard(...).
6. /* Persistent state */
7. addrs = jit_data.addrs ?: kvmalloc(insn_cnt + 1).
8. /* Multi-pass */
9. for pass in 0..MAX_PASSES ∨ image:
   - proglen = Jit::do_jit(env, prog, addrs, image, rw_image, oldproglen, ctx, padding).
   - if image ∧ proglen != oldproglen: error-out.
   - if image: break.
   - if proglen == oldproglen: bpf_jit_binary_pack_alloc(round_up(proglen) + extable_size).
   - oldproglen = proglen.
   - cond_resched().
10. /* Finalize */
11. if !prog.is_func ∨ extra_pass:
    - bpf_jit_binary_pack_finalize(header, rw_header).
    - Jit::tail_call_direct_fixup(prog).
12. prog.bpf_func = image + cfi_get_offset.
13. prog.jited = 1; prog.jited_len = proglen - cfi_get_offset.
14. bpf_prog_update_insn_ptrs(prog, addrs, image).
15. return prog.

`Jit::do_jit(env, prog, addrs, image, rw_image, oldproglen, ctx, jmp_padding) -> i32`:
1. /* Setup */
2. detect_reg_usage(insn, insn_cnt, callee_regs_used).
3. arena_vm_start = bpf_arena_get_kern_vm_start(prog.aux.arena).
4. /* Prologue */
5. Jit::emit_prologue(prog, image, stack_depth, was_classic, tail_call_reachable, is_subprog, exception_cb).
6. if arena_vm_start: Jit::push_r12(prog).
7. Jit::push_callee_regs(prog, callee_regs_used).
8. if arena_vm_start: emit_mov_imm64(R12, arena_vm_start).
9. addrs[0] = proglen.
10. /* Per-insn dispatch */
11. for i in 1..=insn_cnt:
    - if priv_frame_ptr ∧ src/dst == BPF_REG_FP: rewrite to X86_REG_R9.
    - if bpf_insn_is_indirect_target(env, prog, i-1): EMIT_ENDBR.
    - ip = image + addrs[i-1] + (prog - temp).
    - match insn.code:
      - BPF_ALU{,64}|*|BPF_X/K → ALU emit (REQ-9).
      - BPF_LDX|BPF_MEM/MEMSX/PROBE_MEM/PROBE_MEMSX/PROBE_MEM32|* → Jit::emit_ldx*/ldsx*/extable.
      - BPF_ST|BPF_MEM/PROBE_MEM32|* → Jit::emit_st_index/emit_st_r12.
      - BPF_STX|BPF_MEM/PROBE_MEM32|* → Jit::emit_stx/emit_stx_r12.
      - BPF_STX|BPF_ATOMIC|* → Jit::emit_atomic_rmw / emit_atomic_ld_st.
      - BPF_JMP{,32}|*|BPF_X/K → cmp/test + Jit::emit_cond_near_jump.
      - BPF_JMP|BPF_CALL → Jit::emit_call (REQ-13).
      - BPF_JMP|BPF_TAIL_CALL → Jit::emit_bpf_tail_call_{direct,indirect}.
      - BPF_JMP|BPF_EXIT → pop_callee_regs, leave, Jit::emit_return.
    - ilen = prog - temp.
    - if image: memcpy(rw_image + proglen, temp, ilen).
    - proglen += ilen; addrs[i] = proglen.
12. ctx.cleanup_addr = proglen.
13. return proglen.

`Jit::emit_prologue(prog, ip, stack_depth, was_classic, tail_call_reachable, is_subprog, is_exception_cb)`:
1. /* CFI preamble */
2. emit_cfi(prog, ip, is_subprog ? cfi_bpf_subprog_hash : cfi_bpf_hash, arity).
3. /* Fentry patch slot */
4. emit_nops(prog, X86_PATCH_SIZE).
5. if !was_classic:
   - if tail_call_reachable ∧ !is_subprog: emit `xor rax,rax` (init tcc).
   - else: emit_nops(3).
6. /* Frame */
7. if is_exception_cb: rebuild frame from cb args (mov rsp,rsi; mov rbp,rdx; pop_callee_regs(all); pop_r12; mov rsp,rbp).
8. else: EMIT1(0x55) /* push rbp */ ; EMIT3(0x48,0x89,0xE5) /* mov rbp,rsp */.
9. /* IBT */
10. EMIT_ENDBR.
11. if stack_depth: EMIT3_off32(0x48,0x81,0xEC, round_up(stack_depth,8)) /* sub rsp */.
12. if tail_call_reachable: Jit::emit_prologue_tail_call(prog, is_subprog).

`Jit::emit_bpf_tail_call_indirect(bpf_prog, prog, callee_regs_used, stack_depth, ip, ctx)`:
1. /* rdi=ctx, rsi=array, rdx=index */
2. EMIT2(0x89, 0xD2) /* mov edx, edx (zero-extend) */.
3. EMIT3(0x39, 0x56, off_max_entries) /* cmp [rsi+max_entries], edx */.
4. EMIT2(X86_JBE, ctx.tail_call_indirect_label - cur).
5. /* tcc bump */
6. EMIT3_off32(0x48, 0x8B, 0x85, tcc_ptr_off) /* mov rax, [rbp - tcc_ptr_off] */.
7. EMIT4(0x48, 0x83, 0x38, MAX_TAIL_CALL_CNT) /* cmp [rax], MAX */.
8. EMIT2(X86_JAE, ctx.tail_call_indirect_label - cur).
9. /* prog = array->ptrs[index] */
10. EMIT4_off32(0x48,0x8B,0x8C,0xD6, off_ptrs) /* mov rcx, [rsi + rdx*8 + ptrs] */.
11. EMIT3(0x48, 0x85, 0xC9) /* test rcx, rcx */.
12. EMIT2(X86_JE, ctx.tail_call_indirect_label - cur).
13. EMIT4(0x48, 0x83, 0x00, 0x01) /* add [rax], 1 */.
14. /* Unwind callee regs, pop tcc_ptr, add rsp, sd */
15. pop_callee_regs(prog, callee_regs_used); if arena: pop_r12.
16. EMIT1(0x58) /* pop rax */ ; EMIT1(0x58) /* pop rax */.
17. if stack_depth: EMIT3_off32(0x48, 0x81, 0xC4, round_up(sd,8)).
18. /* Jump to prog->bpf_func + X86_TAIL_CALL_OFFSET */
19. EMIT4(0x48,0x8B,0x49, off_bpf_func) /* mov rcx, [rcx+bpf_func] */.
20. EMIT4(0x48,0x83,0xC1, X86_TAIL_CALL_OFFSET) /* add rcx, OFF */.
21. Jit::emit_indirect_jump(prog, BPF_REG_4 /* rcx */, ip).
22. /* out: */
23. ctx.tail_call_indirect_label = prog - start.

`Jit::emit_bpf_tail_call_direct(bpf_prog, poke, prog, ip, callee_regs_used, stack_depth, ctx)`:
1. EMIT3_off32(0x48, 0x8B, 0x85, tcc_ptr_off) /* mov rax, [rbp - tcc_ptr_off] */.
2. EMIT4(0x48, 0x83, 0x38, MAX_TAIL_CALL_CNT).
3. EMIT2(X86_JAE, ctx.tail_call_direct_label - cur).
4. poke.tailcall_bypass = ip + cur; poke.tailcall_target = ip + ctx.tail_call_direct_label - X86_PATCH_SIZE; poke.bypass_addr = target + X86_PATCH_SIZE.
5. Jit::emit_jump(prog, target + X86_PATCH_SIZE, tailcall_bypass) /* NOP patched to E9 rel32 at fixup */.
6. EMIT4(0x48, 0x83, 0x00, 0x01) /* add [rax], 1 */.
7. /* Unwind + pop tcc */
8. pop_callee_regs(prog, callee_regs_used); if arena: pop_r12.
9. EMIT1(0x58); EMIT1(0x58).
10. if stack_depth: EMIT3_off32(0x48, 0x81, 0xC4, round_up(sd,8)).
11. emit_nops(prog, X86_PATCH_SIZE) /* the target slot, patched by tail_call_direct_fixup */.
12. ctx.tail_call_direct_label = prog - start.

`Jit::emit_atomic_rmw(prog, atomic_op, dst, src, off, size) -> i32`:
1. if atomic_op != BPF_XCHG: EMIT1(0xF0) /* lock prefix */.
2. maybe_emit_mod(prog, dst, src, size == BPF_DW).
3. /* opcode */
4. match atomic_op:
   - BPF_ADD|AND|OR|XOR: EMIT1(simple_alu_opcodes[op]).
   - BPF_ADD|BPF_FETCH: EMIT2(0x0F, 0xC1) /* xadd */.
   - BPF_XCHG: EMIT1(0x87).
   - BPF_CMPXCHG: EMIT2(0x0F, 0xB1).
   - default: return -EFAULT.
5. emit_insn_suffix(prog, dst, src, off).
6. return 0.

`Jit::emit_indirect_jump(prog, bpf_reg, ip)`:
1. reg = reg2hex[bpf_reg]; ereg = is_ereg(bpf_reg).
2. if X86_FEATURE_INDIRECT_THUNK_ITS: emit_jump(its_static_thunk(reg + 8*ereg), ip).
3. else if X86_FEATURE_RETPOLINE_LFENCE: EMIT_LFENCE; __emit_indirect_jump(reg, ereg).
4. else if X86_FEATURE_RETPOLINE:
   - if X86_FEATURE_CALL_DEPTH: emit_jump(__x86_indirect_jump_thunk_array[reg + 8*ereg], ip).
   - else: emit_jump(__x86_indirect_thunk_array[reg + 8*ereg], ip).
5. else: __emit_indirect_jump(reg, ereg); if MITIGATION_RETPOLINE ∨ MITIGATION_SLS: EMIT1(0xCC).

`Jit::arch_text_poke(ip, old_t, new_t, old_addr, new_addr) -> i32`:
1. if !is_kernel_text(ip) ∧ !is_bpf_text_address(ip): return -EINVAL.
2. if is_endbr(ip): ip += ENDBR_INSN_SIZE.
3. return Jit::arch_text_poke_inner(ip, old_t, new_t, old_addr, new_addr).

`Jit::arch_text_poke_inner(ip, old_t, new_t, old_addr, new_addr) -> i32`:
1. /* Build expected old_insn / new_insn */
2. nop_insn = x86_nops[5].
3. old_insn = old_t == NOP ? nop_insn : (old_t == CALL ? emit_call : emit_jump)(old_addr, ip).
4. new_insn = new_t == NOP ? nop_insn : (new_t == CALL ? emit_call : emit_jump)(new_addr, ip).
5. mutex_lock(text_mutex).
6. if memcmp(ip, old_insn, 5) != 0: ret = -EBUSY; goto out.
7. if memcmp(ip, new_insn, 5) != 0: smp_text_poke_single(ip, new_insn, 5, NULL); ret = 0.
8. else: ret = 1.
9. out: mutex_unlock(text_mutex).
10. return ret.

`Jit::ex_handler_bpf(x, regs) -> bool`:
1. reg = FIELD_GET(FIXUP_REG_MASK, x.fixup).
2. insn_len = FIELD_GET(FIXUP_INSN_LEN_MASK, x.fixup).
3. is_arena = !!(x.fixup & FIXUP_ARENA_ACCESS).
4. is_write = (reg == DONT_CLEAR).
5. if is_arena: arena_reg = FIELD_GET(FIXUP_ARENA_REG_MASK, x.fixup); off = FIELD_GET(DATA_ARENA_OFFSET_MASK, x.data); addr = *((regs + arena_reg)) + off; bpf_prog_report_arena_violation(is_write, addr, regs.ip).
6. if reg != DONT_CLEAR: *((regs + reg)) = 0 /* clear dst */.
7. regs.ip += insn_len.
8. return true.

`Jit::free(prog)`:
1. if prog.jited:
   - if jit_data: bpf_jit_binary_pack_finalize(header, rw_header); free(addrs); free(jit_data).
   - prog.bpf_func -= cfi_get_offset.
   - hdr = bpf_jit_binary_pack_hdr(prog).
   - bpf_jit_binary_pack_free(hdr, None).
   - if priv_stack_ptr: priv_stack_check_guard(...); free_percpu.
   - WARN_ON_ONCE(!bpf_prog_kallsyms_verify_off(prog)).
2. bpf_prog_unlock_free(prog).

### Out of Scope

- BPF verifier (covered in `verifier.md` Tier-3)
- BPF core interpreter + bpf_prog lifecycle (covered in `bpf-core.md` Tier-3)
- BTF type-info (covered in `btf.md` Tier-3)
- BPF map storage layouts (covered in per-map Tier-3, e.g. `hashmap.md`)
- x86 CPU feature detection / X86_FEATURE_* discovery (covered in `kernel-platform.md` / `cpu-mitigations.md`)
- text_poke / smp_text_poke_single internals (covered in `kernel-platform.md`)
- Module memory allocator (covered separately if expanded)
- Other architectures' JITs (none in v0)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_binary_header` | per-image two-mapping pack (rw + rox) | `BpfBinaryHeader` (shared with kernel/bpf) |
| `struct jit_context` | per-pass JIT cursor + labels | `JitContext` |
| `struct x64_jit_data` | per-prog persistent JIT state across passes | `X64JitData` |
| `bpf_int_jit_compile()` | per-prog driver: multi-pass + emit + finalize | `Jit::int_jit_compile` |
| `do_jit()` | per-pass walk: prologue → per-insn switch → epilogue | `Jit::do_jit` |
| `detect_reg_usage()` | per-prog: which of R6..R9 are touched | `Jit::detect_reg_usage` |
| `emit_prologue()` | per-prog: push rbp, sub rsp, optional ENDBR/tail-call init | `Jit::emit_prologue` |
| `emit_prologue_tail_call()` | per-tail-call: tail_call_cnt slot init | `Jit::emit_prologue_tail_call` |
| `push_callee_regs()` / `pop_callee_regs()` | per-prog: save/restore RBX, R13, R14, R15 | `Jit::push_callee_regs` / `pop_callee_regs` |
| `push_r12()` / `pop_r12()` | per-arena: save R12 for arena base | `Jit::push_r12` / `pop_r12` |
| `push_r9()` / `pop_r9()` | per-priv-stack: save R9 across calls | `Jit::push_r9` / `pop_r9` |
| `emit_call()` / `emit_rsb_call()` / `emit_jump()` / `emit_patch()` | per-call 5-byte E8/E9 rel32 | `Jit::emit_call` / `emit_rsb_call` / `emit_jump` |
| `emit_indirect_jump()` / `__emit_indirect_jump()` | per-retpoline indirect jmp | `Jit::emit_indirect_jump` |
| `emit_return()` | per-epilogue: ret / `x86_return_thunk` | `Jit::emit_return` |
| `emit_nops()` | per-padding NOPs | `Jit::emit_nops` |
| `emit_cfi()` / `emit_kcfi()` / `emit_fineibt()` | per-prog CFI/FineIBT preamble | `Jit::emit_cfi` |
| `emit_mov_imm32()` / `emit_mov_imm64()` | per-imm load | `Jit::emit_mov_imm32` / `imm64` |
| `emit_mov_reg()` / `emit_movsx_reg()` | per-mov reg-reg | `Jit::emit_mov_reg` / `movsx_reg` |
| `emit_ldx()` / `emit_ldsx()` / `emit_ldx_index()` / `emit_ldx_r12()` | per-load | `Jit::emit_ldx*` |
| `emit_stx()` / `emit_stx_index()` / `emit_st_index()` / `emit_st_r12()` | per-store | `Jit::emit_stx*` / `emit_st*` |
| `emit_store_stack_imm64()` | per-stack imm64 store | `Jit::emit_store_stack_imm64` |
| `emit_atomic_rmw()` / `emit_atomic_rmw_index()` | per-LOCK-RMW atomic | `Jit::emit_atomic_rmw*` |
| `emit_atomic_ld_st()` / `emit_atomic_ld_st_index()` | per-acquire/release ld/st | `Jit::emit_atomic_ld_st*` |
| `emit_bpf_tail_call_indirect()` | per-indirect tail-call (array-walk) | `Jit::emit_bpf_tail_call_indirect` |
| `emit_bpf_tail_call_direct()` | per-direct tail-call (poke-patched) | `Jit::emit_bpf_tail_call_direct` |
| `bpf_tail_call_direct_fixup()` | per-finalize: rewrite poke-target sites | `Jit::tail_call_direct_fixup` |
| `emit_spectre_bhb_barrier()` | per-spectre-BHB IBHF barrier | `Jit::emit_spectre_bhb_barrier` |
| `emit_priv_frame_ptr()` | per-priv-stack: load R9 from %gs:[priv] | `Jit::emit_priv_frame_ptr` |
| `emit_shiftx()` / `emit_3vex()` | per-BMI2 shift | `Jit::emit_shiftx` |
| `emit_align()` | per-16B align | `Jit::emit_align` |
| `emit_cond_near_jump()` | per-near-jcc emit | `Jit::emit_cond_near_jump` |
| `priv_stack_init_guard()` / `priv_stack_check_guard()` | per-priv-stack OOB guard | `Jit::priv_stack_init_guard` / `check_guard` |
| `ex_handler_bpf()` | per-PROBE_MEM exception fixup | `Jit::ex_handler_bpf` |
| `__bpf_arch_text_poke()` | per-atomic text patch | `Jit::arch_text_poke_inner` |
| `bpf_arch_text_poke()` | per-public text patch | `Jit::arch_text_poke` |
| `bpf_arch_text_copy()` | per-text-copy via text_poke_copy | `Jit::arch_text_copy` |
| `bpf_arch_text_invalidate()` | per-INT3 fill | `Jit::arch_text_invalidate` |
| `bpf_arch_poke_desc_update()` | per-tailcall poke-desc rewrite | `Jit::arch_poke_desc_update` |
| `arch_prepare_bpf_trampoline()` / `__arch_prepare_bpf_trampoline()` | per-trampoline emit | `Jit::prepare_trampoline` |
| `arch_alloc_bpf_trampoline()` / `arch_free_bpf_trampoline()` / `arch_protect_bpf_trampoline()` | per-trampoline page alloc/protect | `Jit::alloc_trampoline` / `free` / `protect` |
| `arch_bpf_trampoline_size()` | per-trampoline size estimate | `Jit::trampoline_size` |
| `arch_prepare_bpf_dispatcher()` / `emit_bpf_dispatcher()` | per-dispatcher binary-search jmp tree | `Jit::prepare_dispatcher` / `emit_bpf_dispatcher` |
| `arch_bpf_stack_walk()` | per-ORC unwind through BPF frames | `Jit::stack_walk` |
| `invoke_bpf()` / `invoke_bpf_prog()` / `invoke_bpf_mod_ret()` | per-trampoline-invoke prog | `Jit::invoke_bpf*` |
| `save_args()` / `restore_regs()` / `get_nr_used_regs()` | per-trampoline argstack save | `Jit::save_args` / `restore_regs` |
| `clean_stack_garbage()` | per-trampoline padding zero | `Jit::clean_stack_garbage` |
| `bpf_jit_free()` | per-prog: unbind image, free header | `Jit::free` |
| `bpf_jit_supports_kfunc_call()` etc. | per-feature query | `Jit::supports_*` |
| `reg2hex[]` | per-BPF-reg → x86 reg map | shared constant |
| `simple_alu_opcodes[]` | per-ALU-op → x86 opcode | shared constant |
| `is_imm8` / `is_imm8_jmp_offset` / `is_simm32` / `is_uimm32` | per-imm-range check | shared helpers |

### compatibility contract

REQ-1: struct bpf_binary_header (per-image):
- size: total bytes (including header).
- image: ROX-mapped executable text (image = (u8*)header + sizeof(*header)).
- rw_image: paired RW mapping over same backing pages (`bpf_jit_binary_pack_alloc`).
- header initially filled with INT3 (0xCC) via `jit_fill_hole`.

REQ-2: struct x64_jit_data (per-prog persistence across passes / subprogs):
- header: `struct bpf_binary_header *`.
- rw_header: paired RW header.
- ctx: `struct jit_context` cursor.
- proglen: emitted image size.
- image: pointer into ROX text.
- addrs: int[insn_cnt+1] mapping BPF insn → emitted offset.

REQ-3: struct jit_context (per-pass cursor):
- cleanup_addr: epilogue offset.
- tail_call_direct_label / tail_call_indirect_label: branch targets for tail-call out path.
- prog_offset: CFI preamble byte-offset.

REQ-4: bpf_int_jit_compile(env, prog):
- /* short-circuit if JIT disabled */
- if !prog.jit_requested: return prog.
- /* per-prog jit_data alloc-or-reuse */
- jit_data = prog.aux.jit_data ?: alloc(...).
- /* per-prog priv_stack alloc */
- if !aux.priv_stack_ptr ∧ aux.jits_use_priv_stack:
  - priv_stack_alloc_sz = round_up(aux.stack_depth, 8) + 2 * PRIV_STACK_GUARD_SZ.
  - priv_stack_ptr = __alloc_percpu_gfp(priv_stack_alloc_sz, 8, GFP_KERNEL).
  - priv_stack_init_guard(priv_stack_ptr, priv_stack_alloc_sz).
- /* per-pass loop until convergence: image-len monotone */
- for pass in 0..MAX_PASSES ∨ image:
  - proglen = do_jit(env, prog, addrs, image, rw_image, oldproglen, ctx, padding).
  - if image ∧ proglen != oldproglen: report convergence failure, free, return.
  - if image: break.
  - if proglen == oldproglen: bpf_jit_binary_pack_alloc(roundup(proglen, align) + extable_size, &image, ...) ; prog.aux.extable = image + roundup(proglen, align).
  - oldproglen = proglen.
  - cond_resched().
- /* per-finalize */
- if !prog.is_func ∨ extra_pass:
  - bpf_jit_binary_pack_finalize(header, rw_header).
  - bpf_tail_call_direct_fixup(prog).
- /* per-prog state publish */
- prog.bpf_func = image + cfi_get_offset().
- prog.jited = 1; prog.jited_len = proglen - cfi_get_offset().
- bpf_prog_update_insn_ptrs(prog, addrs, image).
- return prog.

REQ-5: do_jit(env, prog, addrs, image, rw_image, oldproglen, ctx, jmp_padding):
- /* per-pass init */
- detect_reg_usage(insn, insn_cnt, callee_regs_used).
- arena_vm_start = bpf_arena_get_kern_vm_start(aux.arena).
- emit_prologue(prog, image, stack_depth, was_classic, tail_call_reachable, is_subprog, exception_cb).
- if exception_boundary: push_r12; push_callee_regs(all_callee_regs_used).
- else: if arena_vm_start: push_r12; push_callee_regs(callee_regs_used).
- if arena_vm_start: emit_mov_imm64(R12, arena_vm_start).
- if priv_frame_ptr: emit_priv_frame_ptr(priv_frame_ptr).
- addrs[0] = proglen.
- /* per-insn switch */
- for i in 1..=insn_cnt:
  - if priv_frame_ptr ∧ src_reg/dst_reg == BPF_REG_FP: rewrite to X86_REG_R9.
  - if bpf_insn_is_indirect_target(env, prog, i-1): EMIT_ENDBR.
  - switch (insn.code): per-BPF-class emit.
  - if (image): memcpy(rw_image + proglen, temp, ilen).
  - proglen += ilen.
  - addrs[i] = proglen.
- /* per-epilogue */
- ctx.cleanup_addr = proglen.
- pop_callee_regs, pop_r12 (if arena), leave, emit_return.
- return proglen.

REQ-6: emit_prologue(stack_depth, ebpf_from_cbpf, tail_call_reachable, is_subprog, is_exception_cb):
- emit_cfi(cfi_bpf_subprog_hash if is_subprog else cfi_bpf_hash, arity).
- emit_nops(X86_PATCH_SIZE) /* room for trampoline fentry patch */.
- if !ebpf_from_cbpf:
  - if tail_call_reachable ∧ !is_subprog: EMIT3(0x48,0x31,0xC0) /* xor rax,rax = init tcc */.
  - else: emit_nops(3).
- if is_exception_cb: /* rebuild frame from callback args */
  - mov rsp, rsi; mov rbp, rdx; pop_callee_regs(all); pop_r12; mov rsp, rbp.
- else: EMIT1(0x55) /* push rbp */ ; EMIT3(0x48,0x89,0xE5) /* mov rbp,rsp */.
- /* X86_TAIL_CALL_OFFSET is here */
- EMIT_ENDBR.
- if stack_depth: EMIT3_off32(0x48,0x81,0xEC, round_up(stack_depth,8)) /* sub rsp, sd */.
- if tail_call_reachable: emit_prologue_tail_call(is_subprog).

REQ-7: emit_prologue_tail_call:
- /* tail_call_cnt enters R0/rax */
- if !is_subprog:
  - cmp rax, MAX_TAIL_CALL_CNT (33); ja 6.
  - push rax (tcc) ; mov rax, rsp (tcc_ptr) ; jmp 1.
  - push rax (alt path: rax already is tcc_ptr) ; push rax.
- else: push rax; push rax.
- Net effect: stack contains [tcc_ptr][tcc] (or [tcc_ptr][tcc_ptr] in sub-tail callee).

REQ-8: Per-BPF-register x86 mapping `reg2hex[]`:
- BPF_REG_0 → 0 (RAX), BPF_REG_1 → 7 (RDI), BPF_REG_2 → 6 (RSI), BPF_REG_3 → 2 (RDX), BPF_REG_4 → 1 (RCX), BPF_REG_5 → 0 (R8).
- BPF_REG_6 → 3 (RBX callee), BPF_REG_7 → 5 (R13 callee), BPF_REG_8 → 6 (R14 callee), BPF_REG_9 → 7 (R15 callee).
- BPF_REG_FP → 5 (RBP read-only), BPF_REG_AX → 2 (R10 temp), AUX_REG → 3 (R11 temp).
- X86_REG_R9 → 1 (R9), X86_REG_R12 → 4 (R12 callee, arena base).

REQ-9: ALU/ALU64 per-insn emit (do_jit switch):
- ALU{,64} ADD/SUB/AND/OR/XOR BPF_X: `maybe_emit_mod` REX + opcode (`simple_alu_opcodes`) + ModR/M(0xC0,dst,src).
- ALU{,64} MOV BPF_X: emit_mov_reg or emit_movsx_reg if off ∈ {8,16,32}.
- BPF_ALU64|BPF_MOV|BPF_X with `insn_is_cast_user`: emit shift/or/rol/test/cmove sequence to insert arena user_vm_start tag.
- BPF_ALU64|BPF_MOV|BPF_X with `insn_is_mov_percpu_addr`: add gs:[this_cpu_off] to dst.
- ALU NEG: `F7 /3`.
- ALU{,64} ADD/SUB/AND/OR/XOR BPF_K: 0x83 imm8 or 0x81/0x05 imm32.
- ALU{,64} MOV BPF_K: emit_mov_imm32.
- LD|IMM|DW (BPF wide imm64): emit_mov_imm64; skip insn+1.
- ALU{,64} DIV/MOD BPF_K|BPF_X: push rax/rdx, idiv/div, mov result.
- ALU{,64} MUL: imul.
- ALU{,64} LSH/RSH/ARSH BPF_K: shift-by-imm; BPF_X: shrx/shlx (BMI2) via emit_shiftx OR cl-shift.
- ALU END/BSWAP: bswap or rol/and depending on size.

REQ-10: LDX/ST/STX per-insn emit:
- LDX|MEM|{B,H,W,DW}: emit_ldx (movzbq/movzwq/mov/mov-64).
- LDX|MEMSX|{B,H,W}: emit_ldsx (movsbq/movswq/movslq).
- LDX|PROBE_MEM|{B,H,W,DW}: emit_ldx + record extable entry (faulting → ex_handler_bpf clears reg).
- LDX|PROBE_MEMSX|{B,H,W}: emit_ldsx + extable.
- LDX|PROBE_MEM32|*: arena variant; uses R12 as base via emit_ldx_r12 / emit_ldsx_r12 with arena offset stored in extable data.
- ST|MEM|{B,H,W,DW}: emit_st_index (mov imm to mem).
- STX|MEM|{B,H,W,DW}: emit_stx (mov reg to mem).
- STX|PROBE_MEM32|*: emit_stx_r12 + extable record.

REQ-11: Atomic emit (BPF_STX|BPF_ATOMIC):
- emit_atomic_rmw(op, dst, src, off, size):
  - if op != BPF_XCHG: EMIT1(0xF0) /* lock prefix */.
  - per-op: ADD/AND/OR/XOR → `simple_alu_opcodes[op]`; ADD|FETCH → 0x0F 0xC1 (xadd); XCHG → 0x87; CMPXCHG → 0x0F 0xB1.
  - emit_insn_suffix(dst, src, off).
- emit_atomic_rmw_index: arena variant via R12 base + SIB.
- emit_atomic_ld_st: BPF_LOAD_ACQ → emit_ldx (load-acquire is plain mov on x86); BPF_STORE_REL → emit_stx (store-release is plain mov on x86); TSO ordering implicit.
- 1- and 2-byte RMW atomics not supported (return -EFAULT).

REQ-12: JMP/JMP32 per-insn emit:
- JEQ/JNE/JGT/JLT/JGE/JLE/JSGT/JSLT/JSGE/JSLE BPF_X: cmp dst, src; emit_cond_jmp.
- JSET BPF_X: test dst, src; emit_cond_jmp.
- JEQ/JNE/... BPF_K: cmp dst, imm (or test for JSET).
- JMP|JMP|BPF_K: emit_jump (5-byte E9 rel32) or short EB.
- JMP|JA|BPF_K (32-bit imm offset): always 5-byte E9.
- JCOND: emit_cond_near_jump (jcc with rel32) or 2-byte rel8 if `is_imm8_jmp_offset`.
- JMP|EXIT: pop callee regs, leave, emit_return.

REQ-13: BPF_JMP|BPF_CALL:
- func = __bpf_call_base + imm32.
- if src_reg == BPF_PSEUDO_CALL ∧ tail_call_reachable: LOAD_TAIL_CALL_CNT_PTR; ip += 7.
- if priv_frame_ptr: push_r9; ip += 2.
- ip += x86_call_depth_emit_accounting(prog, func, ip).
- emit_call(prog, func, ip) → emit_patch(0xE8 rel32).
- if priv_frame_ptr: pop_r9.

REQ-14: BPF_JMP|BPF_TAIL_CALL:
- if imm32 != 0: emit_bpf_tail_call_direct (uses poke_tab[imm32-1]) — patches static-call sites via bpf_tail_call_direct_fixup post-finalize.
- else: emit_bpf_tail_call_indirect — walks bpf_array via index, checks max_entries / MAX_TAIL_CALL_CNT / prog != NULL, increments *tcc_ptr, then `jmp prog->bpf_func + X86_TAIL_CALL_OFFSET` via emit_indirect_jump.

REQ-15: emit_indirect_jump(reg, ip) per X86_FEATURE_*:
- X86_FEATURE_INDIRECT_THUNK_ITS: emit_jump(its_static_thunk(reg + 8*ereg), ip).
- X86_FEATURE_RETPOLINE_LFENCE: LFENCE (0x0F,0xAE,0xE8) + bare indirect.
- X86_FEATURE_RETPOLINE:
  - if X86_FEATURE_CALL_DEPTH: emit_jump(__x86_indirect_jump_thunk_array[reg + 8*ereg]).
  - else: emit_jump(__x86_indirect_thunk_array[reg + 8*ereg]).
- else: __emit_indirect_jump(reg, ereg); if MITIGATION_RETPOLINE ∨ MITIGATION_SLS: EMIT1(0xCC) /* int3 */.

REQ-16: emit_return(ip):
- if cpu_wants_rethunk(): emit_jump(x86_return_thunk, ip).
- else: EMIT1(0xC3); if MITIGATION_SLS: EMIT1(0xCC).

REQ-17: ex_handler_bpf (PROBE_MEM fault fixup):
- fixup-bit layout: `INSN_LEN[7:0] | DST_REG[15:8] | ARENA_REG[23:16] | ARENA_ACC[31]`.
- data layout: `EX_TYPE_BPF[7:0] | ARENA_OFFSET[31:16]`.
- if arena: addr = *(regs + arena_reg) + off; bpf_prog_report_arena_violation(is_write, addr, regs.ip).
- if reg != DONT_CLEAR: *(regs + reg) = 0 /* clear dst */.
- regs.ip += insn_len /* skip faulting insn */.
- return true.

REQ-18: __bpf_arch_text_poke(ip, old_t, new_t, old_addr, new_addr):
- Build expected old_insn[5] and new_insn[5] from {nop5, E8 rel32, E9 rel32}.
- mutex_lock(text_mutex).
- if memcmp(ip, old_insn, 5): return -EBUSY.
- if memcmp(ip, new_insn, 5): smp_text_poke_single(ip, new_insn, 5, NULL).
- mutex_unlock(text_mutex).

REQ-19: bpf_arch_text_poke(ip, old_t, new_t, old_addr, new_addr):
- if !is_kernel_text(ip) ∧ !is_bpf_text_address(ip): return -EINVAL (modules not supported).
- if is_endbr(ip): ip += ENDBR_INSN_SIZE.
- return __bpf_arch_text_poke(ip, old_t, new_t, old_addr, new_addr).

REQ-20: bpf_arch_text_copy / bpf_arch_text_invalidate:
- text_copy: text_poke_copy(dst, src, len) → returns dst or ERR_PTR(-EINVAL).
- text_invalidate: per-len fill with INT3 (0xCC) via text_poke_set.

REQ-21: Trampoline (`__arch_prepare_bpf_trampoline`):
- Compute stack layout: regs_off, func_meta_off, ip_off, cookie_off, rbx_off, run_ctx_off, arg_stack_off.
- Emit fentry stub: push rbp, mov rbp,rsp, sub rsp, stack_size, optional push rax (tail_call_cnt_ctx).
- save_args(m, prog, regs_off, ...).
- if F_CALL_ORIG: mov rdi, im; emit_rsb_call(__bpf_tramp_enter, ...).
- if fentry.nr_links: invoke_bpf(m, prog, fentry, ...).
- if fmod_ret.nr_links: invoke_bpf_mod_ret → branches[] for fall-through.
- if F_CALL_ORIG: call orig + skip ENDBR/X86_PATCH_SIZE.
- if fexit.nr_links: invoke_bpf(m, prog, fexit, ...).
- if F_CALL_ORIG: emit_rsb_call(__bpf_tramp_exit, ...).
- mov rax, [rbp - 8] /* return value if save_ret */.
- mov rbx, [rbp - rbx_off]; leaveq; ret-or-jmp.

REQ-22: Dispatcher (`emit_bpf_dispatcher`):
- Binary-search jmp tree over s64 funcs[a..b]: emit cmp + jcc + recursive emit; leaf: emit_jump(funcs[a]).

REQ-23: bpf_jit_free(prog):
- if prog.jited:
  - finalize header if jit_data still present.
  - prog.bpf_func -= cfi_get_offset; bpf_jit_binary_pack_free(hdr).
  - if priv_stack_ptr: priv_stack_check_guard; free_percpu.
  - WARN_ON_ONCE(!bpf_prog_kallsyms_verify_off(prog)).
- bpf_prog_unlock_free(prog).

REQ-24: Feature-query queries (verifier consults):
- bpf_jit_supports_kfunc_call: true.
- bpf_jit_supports_subprog_tailcalls: true (mixed bpf2bpf + tailcall).
- bpf_jit_supports_percpu_insn: true.
- bpf_jit_supports_exceptions: IS_ENABLED(CONFIG_UNWINDER_ORC).
- bpf_jit_supports_private_stack: true.
- bpf_jit_supports_arena: true.
- bpf_jit_supports_insn(insn, in_arena): rejects {AND|FETCH, OR|FETCH, XOR|FETCH} in arena.
- bpf_jit_supports_ptr_xchg: true.
- bpf_arch_uaddress_limit: 0 (JIT emits its own user-addr check).
- bpf_jit_supports_timed_may_goto: true.
- bpf_jit_supports_fsession: true.

REQ-25: CFI / FineIBT / kCFI preamble:
- emit_cfi → emit_kcfi or emit_fineibt per CFI_KCFI / CFI_FINEIBT.
- emit_kcfi(hash): emits 4-byte hash before function entry (KCFI direct-call check).
- emit_fineibt(hash, arity): emits the FineIBT preamble before ENDBR using kCFI hash + arity check.
- ctx.prog_offset tracks bytes prepended.

REQ-26: Private-stack guard:
- priv_stack_init_guard: write 0xEB in PRIV_STACK_GUARD_SZ bytes before+after each percpu region.
- priv_stack_check_guard: WARN_ON if guard bytes modified at free time.

REQ-27: Convergence: pass loop continues until proglen == oldproglen for two consecutive passes (PADDING_PASSES marks switch to padded mode after which is_imm8_jmp_offset uses 123 instead of 127 to avoid je↔jmp 2↔6 oscillation).

REQ-28: arch_bpf_stack_walk(consume_fn, cookie):
- if CONFIG_UNWINDER_ORC: unwind_start; loop unwind_next_frame; consume_fn(cookie, ip, sp, bp); break on return-false.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `image_within_bounds` | INVARIANT | per-do_jit: proglen ≤ BPF_MAX_INSN_SIZE * insn_cnt. |
| `addrs_monotone` | INVARIANT | per-do_jit: addrs[i] ≥ addrs[i-1] for all i. |
| `text_poke_atomic` | INVARIANT | per-__bpf_arch_text_poke: text_mutex held during smp_text_poke_single. |
| `extable_entries_count` | INVARIANT | per-do_jit: num_exentries == count(BPF_PROBE_MEM*) ops. |
| `tail_call_cnt_bounded` | INVARIANT | per-emit_bpf_tail_call_*: *tcc_ptr ≤ MAX_TAIL_CALL_CNT before jump. |
| `cfi_preamble_present` | INVARIANT | per-emit_prologue: emit_cfi emitted at function entry. |
| `priv_stack_guard_intact` | INVARIANT | per-bpf_jit_free: guard bytes unchanged. |
| `indirect_jmp_respects_retpoline` | INVARIANT | per-emit_indirect_jump: thunk-or-bare per X86_FEATURE_RETPOLINE. |

### Layer 2: TLA+

`arch/x86/bpf-jit.tla`:
- Per-pass + per-finalize + per-poke + per-tail-call.
- Properties:
  - `safety_image_converges` — per-bpf_int_jit_compile: pass loop terminates with proglen stable.
  - `safety_addrs_consistent` — per-pass: addrs[insn_cnt] == proglen at end of pass.
  - `safety_text_poke_atomicity` — per-__bpf_arch_text_poke: writers serialized via text_mutex.
  - `safety_tail_call_bound` — per-tail-call: count ≤ MAX_TAIL_CALL_CNT.
  - `liveness_pass_converges` — per-int_jit_compile: terminates within MAX_PASSES + 1.
  - `liveness_trampoline_call_returns` — per-arch_prepare_bpf_trampoline: fentry/fexit invariant order.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Jit::int_jit_compile` post: prog.jited == 1 ∧ prog.bpf_func ∈ image | `Jit::int_jit_compile` |
| `Jit::do_jit` post: ret ≥ 0 ⟹ addrs[i] valid for i in [0, insn_cnt] | `Jit::do_jit` |
| `Jit::emit_prologue` post: image bytes form valid prologue | `Jit::emit_prologue` |
| `Jit::emit_atomic_rmw` post: image bytes form valid LOCK-prefixed RMW | `Jit::emit_atomic_rmw` |
| `Jit::emit_indirect_jump` post: image bytes form retpoline-or-bare indirect | `Jit::emit_indirect_jump` |
| `Jit::arch_text_poke_inner` post: ip bytes == new_insn ∨ ret < 0 | `Jit::arch_text_poke_inner` |
| `Jit::ex_handler_bpf` post: regs.ip += insn_len; reg cleared (unless DONT_CLEAR) | `Jit::ex_handler_bpf` |
| `Jit::free` post: header freed; priv_stack guard checked | `Jit::free` |

### Layer 4: Verus/Creusot functional

`Per-prog: bpf_int_jit_compile(prog) → do_jit (multi-pass) → emit_prologue → per-insn switch → emit_return → bpf_jit_binary_pack_finalize → tail_call_direct_fixup → prog.jited=1; prog.bpf_func` semantic equivalence: per-`Documentation/bpf/bpf_design_QA.rst` (BPF semantics) + Intel SDM (x86-64 encoding tables). JIT-correctness — i.e. the emitted x86 bytes implement the BPF program's semantics insn-by-insn — is the *signature* correctness obligation; tracked but may slip past v0 per `arch/x86/00-overview.md` Verification table.

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

BPF-JIT reinforcement:

- **Per-ROX image** — defense against per-W^X violation: image is ROX-only; rw_image is the only writable handle, bound at allocation.
- **Per-extable PROBE_MEM fixup** — defense against per-bpf-prog crashing kernel on bad pointer deref.
- **Per-RETPOLINE indirect-jmp** — defense against per-Spectre v2 BTI: emit_indirect_jump consults X86_FEATURE_RETPOLINE / RETPOLINE_LFENCE / INDIRECT_THUNK_ITS / CALL_DEPTH.
- **Per-cpu_wants_rethunk return** — defense against per-Retbleed: emit_return jumps to x86_return_thunk when required.
- **Per-MITIGATION_SLS INT3 padding** — defense against per-Straight-Line-Speculation: bare ret/indirect-jmp followed by 0xCC.
- **Per-CFI/FineIBT preamble** — defense against per-kCFI mismatch + per-FineIBT enforcement.
- **Per-ENDBR at prologue + at indirect target** — defense against per-CET-IBT violation.
- **Per-text_mutex + smp_text_poke_single** — defense against per-text-patch racing concurrent execution.
- **Per-tail_call_cnt ≤ MAX_TAIL_CALL_CNT (33)** — defense against per-tailcall-infinite-loop DoS.
- **Per-priv_stack guard 0xEB pattern** — defense against per-priv-stack-overflow / underflow.
- **Per-bpf_arch_text_poke kernel-text-only** — defense against per-poke-module misuse (returns -EINVAL).
- **Per-extable BPF tracker** — defense against per-arena UAF: arena_violation logged via bpf_prog_report_arena_violation.
- **Per-spectre_bhb_barrier (IBHF)** — defense against per-Branch-History-Injection on tail-call jumps.
- **Per-NX-only via bpf_jit_binary_pack_finalize** — defense against per-image-stays-writable.
- **Per-cfi_get_offset symmetric on free** — defense against per-CFI hash drift on prog teardown.
- **Per-padding-pass is_imm8_jmp_offset cap 123** — defense against per-jmp-encoding-oscillation (non-converging JIT).

