---
title: "Tier-3: arch/x86/kvm/emulate.c — x86 instruction emulator (real-mode + 32-bit + invalid-guest-state recovery)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The x86 instruction emulator — a software interpreter for the x86 ISA that KVM uses when hardware can't or shouldn't directly execute guest instructions. Three primary use cases:

- **Real-mode + 32-bit-no-paging guest emulation** on Intel CPUs without `unrestricted_guest=1` capability (older HW pre-VT-x flexpriority): real-mode is non-virtualizable; emulator walks instruction stream + applies effects to a virtual register file. Modern HW (Nehalem+) supports unrestricted-guest making this rarely-needed for boot, but remains as fallback.
- **Invalid-guest-state recovery**: when guest's vmexit happens because guest CR0/CR3/CR4/EFER state is partially-invalid (e.g., guest in transition between protected-mode + paging-enable), emulator can step through enough instructions to reach a valid state.
- **MMIO emulation + cross-page instruction emulation**: when guest does MMIO that traps into KVM, or when an instruction crosses a page boundary where one page was dropped from EPT, emulator handles the access.

This Tier-3 covers `arch/x86/kvm/emulate.c` (~5700 lines) — the largest single file in `arch/x86/kvm/`. Implements ~100s of x86 opcodes including REX-prefixed 64-bit forms, two-byte 0F-prefixed opcodes, three-byte 0F38/0F3A opcodes, prefix-encoded SIMD, segment-prefix overrides, lock-prefix, repe/repne string ops.

### Acceptance Criteria

- [ ] AC-1: kvm-unit-tests `realmode` test passes: emulator boots BIOS-equivalent code in real-mode on HW without unrestricted-guest.
- [ ] AC-2: kvm-unit-tests `emulator` test passes: per-mnemonic correctness verified for ~200 opcodes.
- [ ] AC-3: MMIO emulation: guest does MMIO read at LAPIC mmio @ 0xfee00020 (ID register); emulator dispatches via `read_emulated` → returns LAPIC ID.
- [ ] AC-4: Lock-prefix atomicity: guest does `LOCK CMPXCHG` on shared page between two vCPUs; emulator's cmpxchg loop preserves atomicity vs HW vCPU.
- [ ] AC-5: REP-prefixed-string-op interruption: guest does `REP MOVSB` with ECX=10000; concurrent IRQ delivery interrupts string at ECX boundary; per-iteration check observable.
- [ ] AC-6: Hypercall: guest issues VMCALL with eax=KVM_HC_VAPIC_POLL_IRQ; emulator dispatches.
- [ ] AC-7: Invalid-guest-state recovery: guest enables paging while in PROT_16 mode (transition); emulator steps until valid state reached.
- [ ] AC-8: kvm-unit-tests x86 full suite: passes with same pass/fail manifest as upstream baseline.

### Architecture

`EmuCtxt` lives in `kernel::kvm::x86::EmuCtxt`:

```
struct EmuCtxt {
  ops: &'static dyn EmuOps,              // per-vendor callback vtable
  vcpu: Arc<Vcpu>,
  eflags: u64,
  eip: u64,                               // current instruction pointer
  mode: EmulationMode,                    // REAL / PROT_16 / PROT_32 / LONG / VM86 / SMM
  cpl: u8,
  intr: bool,
  perm_ok: bool,
  ud: bool,                                // #UD pending
  tf: bool,
  have_exception: bool,
  exception: ExceptionInfo,
  seg_override: u8,
  ad_bytes: u8,                            // address-size: 2/4/8 bytes
  op_bytes: u8,                            // operand-size: 1/2/4/8
  rex_prefix: u8,                          // REX/REX.W bits
  lock_prefix: bool,
  rep_prefix: u8,
  src: Operand,
  src2: Operand,
  dst: Operand,
  memop: MemOp,
  fop: FpuOp,
  d: u64,                                  // opcode-flags from decode-table
  execute: Option<fn(&mut EmuCtxt) -> i32>,// per-opcode handler
  check_perm: Option<fn(&mut EmuCtxt) -> i32>,
  intercept: u8,
  fetch: FetchCache,                       // per-fetch cache for instruction bytes
  mem_read: MemReadCache,
  decoded_modrm: Option<ModRM>,
  registers: GprRegisters,                 // per-vCPU GPR snapshot
  segments: [SegDesc; 6],                  // CS/DS/ES/FS/GS/SS
  has_seg_override: bool,
  ...
}
```

Per-emulation flow `Vcpu::emulate_instruction(gpa, type_, insn, len)`:
1. `EmuCtxt::init(ctxt, vcpu, mode)` — populate from per-vCPU register state.
2. `EmuCtxt::decode_insn(ctxt, insn, len)`:
   - Parse prefixes (segment-override / lock / rep / address-size / operand-size / REX).
   - Read 1-byte opcode; if 0x0F, read 2-byte; if 0x38/0x3A, read 3-byte.
   - Look up in opcode-decode-table → returns flag-set + execute-fn ptr + memop + register-class for src/dst/modrm/sib.
   - Decode ModR/M + SIB + displacement + immediate per addressing-mode encoding.
3. `EmuCtxt::check_privilege` if applicable (CPL check).
4. `EmuCtxt::linearize(ctxt, src.addr, src.bytes, false, &linear_src)` — seg+offset → linear; check limit + access-rights.
5. `ctxt.execute(ctxt)` — per-opcode handler:
   - Performs operation per Intel SDM Vol 2 entry.
   - For mem ops: `ops.read_emulated` or `ops.write_emulated` callback.
   - For PIO: `ops.pio_in_emulated` / `_pio_out_emulated`.
   - For MSR: `ops.read_msr` / `_write_msr`.
   - Sets EFLAGS bits per operation result.
   - Increments `ctxt.eip` past instruction length.
6. If exception detected: `EmuCtxt::inject_emulated_exception(...)` → KVM injects via VM-Entry-Interrupt-Information.
7. Apply EFLAGS + register changes to vCPU.

REP-prefixed loop `EmuCtxt::execute_rep_string_op(ctxt)`:
1. Loop while `ECX > 0`:
   - Per-iteration: check pending IRQ (cross-ref `x86-lapic.md`); if delivery-window open + IRQ pending: break (return early; KVM injects IRQ).
   - Execute one iteration (e.g., MOVSB: copy one byte from [ESI] to [EDI], inc/dec by 1 per direction-flag).
   - Decrement ECX.
   - For REPE/REPNE: also check ZF.
2. If ECX > 0 on return: re-enter same instruction on next vmentry (re-emulate).

MMIO emulation: `ops.read_emulated(ctxt, addr, &val, bytes, exception)`:
1. Per-vendor (KVM-side) callback walks per-VM IO bus (cross-ref `kvm-eventfd.md` § ioeventfd) for addr.
2. If matched ioeventfd: triggers eventfd_signal; advance + return success.
3. Else if matched in-kernel device (LAPIC, IOAPIC, PIT, virtio coalesced-mmio-ring): dispatch.
4. Else: set `vcpu->run.exit_reason = KVM_EXIT_MMIO`; populate exit info; return error → emulator returns to userspace.

Hypercall `Vcpu::emulate_hypercall`:
1. Read EAX (hypercall number) + EBX/ECX/EDX/ESI/EDI (args).
2. Per-KVM_HC_* dispatch:
   - KVM_HC_VAPIC_POLL_IRQ
   - KVM_HC_KICK_CPU
   - KVM_HC_CLOCK_PAIRING
   - KVM_HC_SEND_IPI
   - etc.
3. Set EAX = result.

Per-vendor `EmuOps` impl: VMX-side and SVM-side each provide their own `EmuOps` instance (cross-ref `vmx-core.md` + `svm-core.md`); differs in `intercept` callback (per-vendor VM-exit interception table) + per-MSR list.

### Out of Scope

- Per-vendor VMX entry/exit (covered in `vmx-core.md` future Tier-3)
- Per-vendor SVM entry/exit (covered in `svm-core.md` future Tier-3)
- LAPIC IRQ injection (covered in `x86-lapic.md` Tier-3)
- IOAPIC + PIC + PIT emulation (covered in `x86-ioapic.md` future Tier-3)
- Guest pgtable walk (`paging_tmpl.h` — covered in `x86-mmu-shadow.md` future Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct x86_emulate_ctxt` | per-emulation context (registers + state) | `kernel::kvm::x86::EmuCtxt` |
| `struct x86_emulate_ops` | per-vendor (KVM-side) callback ops | `kernel::kvm::x86::EmuOps` (trait) |
| `x86_emulate_instruction(vcpu, gpa, emulation_type, insn, insn_len)` | top-level entry from KVM | `Vcpu::emulate_instruction` |
| `x86_emulate_init_ctxt(ctxt, vcpu, mode)` | per-vCPU emulator-context init | `EmuCtxt::init` |
| `x86_emulate_insn(ctxt)` | execute one emulated instruction | `EmuCtxt::execute_insn` |
| `decode_insn(ctxt, insn, len)` | decode opcode + ModR/M + SIB + displacement + immediate | `EmuCtxt::decode_insn` |
| `register_address(ctxt, reg)` / `_address_increment(...)` | per-register access + increment | `EmuCtxt::register_*` |
| `read_segment_descriptor(ctxt, selector, seg, base, limit, access_rights)` | per-segment GDT/LDT walk | `EmuCtxt::read_segment_descriptor` |
| `linearize(ctxt, addr, size, write, &linear)` | seg+offset → linear addr (with limit check) | `EmuCtxt::linearize` |
| `seg_override(ctxt)` | resolve segment override prefix | `EmuCtxt::seg_override` |
| `assign_register(ctxt, &dest, value, byte_op)` | per-register write w/ size-correct masking | `EmuCtxt::assign_register` |
| `emulate_int(ctxt, vector)` / `_int_real(...)` / `_iret_real(...)` | INT / IRET emulation | `EmuCtxt::emulate_int` / `_int_real` / `_iret_real` |
| `task_switch(ctxt, tss_selector, idt_index, reason, &has_error_code, &error_code)` | TSS-based task-switch emulation (real-mode + 16-bit pmode) | `EmuCtxt::task_switch` |
| `em_call_far(ctxt)` / `em_jmp_far(...)` / `em_ret_far(...)` | far call/jmp/ret w/ segment validation | `EmuCtxt::em_call_far` / etc. |
| `em_grp1` / `em_grp3` / `em_grp45` / etc. | grouped-opcode dispatch | `EmuCtxt::em_grp*` |
| `em_xchg(ctxt)` / `_movs_*(...)` / `_cmps_*(...)` / `_scas_*(...)` / `_lods_*(...)` / `_stos_*(...)` | per-mnemonic emulation | `EmuCtxt::em_*` |
| `em_pop_sreg(ctxt, ...)` / `_pop_seg(...)` | seg-reg pop w/ validation | `EmuCtxt::em_pop_*` |
| `setup_syscalls_segments(...)` | SYSCALL/SYSRET seg-reg setup | `EmuCtxt::setup_syscalls_segments` |
| `kvm_emulate_hypercall(vcpu)` | KVM hypercall (vmcall/vmmcall) | `Vcpu::emulate_hypercall` |
| `x86_decode_emulated_instruction(vcpu, emul_type, insn_bytes, insn_len)` | partial decode for instruction-window tracking | `Vcpu::decode_emulated_instruction` |

### compatibility contract

REQ-1: x86 instruction set coverage per Intel SDM Vol 2 + AMD APM Vol 3:
- All 1-byte opcodes (0x00-0xFF).
- All 2-byte 0F-prefixed (0F 00-FF).
- 3-byte 0F38/0F3A (subset relevant for emulation; full SIMD not needed since SSE+ HW-executed).
- All ModR/M + SIB encoding variants.
- All addressing modes (immediate / register / indirect / base+index*scale+displacement).
- Segment-override prefixes (CS/DS/ES/FS/GS/SS).
- Lock-prefix + REP/REPE/REPNE.
- REX prefix (64-bit operand-size + extended registers + extended index).

REQ-2: Per-mode emulation: REAL_MODE (16-bit segments + IVT for INT) / PROT_16 / PROT_32 / LONG (64-bit) / VM86 / SMM. Mode determined from `ctxt->mode`.

REQ-3: Segment register emulation: per-segment cache (selector + base + limit + access-rights); `linearize` checks limit + access-rights; SS/SS_RPL/CPL/DPL semantics enforced.

REQ-4: Privilege checks: per-instruction CPL check (e.g., MSR access requires CPL=0); SS-write triggers DR6 single-step disable per spec; #GP / #UD / #PF generated on violations + delivered via `emulate_int_real` or `kvm_inject_emulated_page_fault`.

REQ-5: MMIO emulation: instructions touching MMIO regions trap into emulator via `read_emulated` / `write_emulated` callbacks; per-call returns to userspace via KVM_EXIT_MMIO if KVM-side dispatch fails.

REQ-6: Page-fault delivery: emulator-detected #PF emulated via `inject_emulated_exception` → kvm_arch_async_page_present integration.

REQ-7: Lock-prefix emulation: `LOCK XCHG`, `LOCK CMPXCHG`, `LOCK ADD`, etc. emulated atomically using cmpxchg loop on host memory.

REQ-8: REP-prefixed string ops: emulator iterates ECX times, with per-iteration interruption-window check (RFLAGS.IF + pending IRQ); allows guest to receive IRQ mid-REP-string-op.

REQ-9: Task-switch emulation: real-mode + 16-bit-pmode TSS-based task switch via `task_switch` — seg-reg load + EFLAGS.NT + task-busy bit.

REQ-10: Hypercall emulation: VMCALL (Intel) / VMMCALL (AMD) → `kvm_emulate_hypercall(vcpu)` → KVM_HC_* opcode dispatch.

REQ-11: Per-vendor `EmuOps` callback table (trait):
- `read_std`/`write_std`: read/write GVA via guest pgtable.
- `read_phys`/`write_phys`: read/write GPA via memslots.
- `read_emulated`/`write_emulated`: same with MMIO trap fallback.
- `cmpxchg_emulated`: atomic cmpxchg.
- `pio_in_emulated`/`pio_out_emulated`: PIO instructions.
- `intercept`: per-vendor VMX/SVM intercept-table consultation (some VMEXITs are intercepted instead of emulated).
- `read_pmc`/`fix_hypercall`: PMC reads + hypercall-instruction emulation.
- `read_msr`/`write_msr`: MSR access.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `decode_no_oob` | OOB | per-instruction byte-stream access bounded by `insn_len`; FetchCache prevents over-fetch. |
| `seg_descriptor_no_oob` | OOB | GDT/LDT walk bounded by `gdtr_limit` / `ldtr_limit`. |
| `register_access_no_oob` | OOB | GPR index 0..15 (REX-extended); access bounded. |
| `string_op_no_infinite_loop` | TERMINATION | REP-prefix loop bounded by ECX; per-iteration ECX decrement guaranteed. |
| `linearize_arithmetic_no_overflow` | OVERFLOW | seg.base + offset checked for u64 overflow; #GP on overflow. |

### Layer 2: TLA+

(No specific TLA+ — emulator semantics covered by Verus functional equivalence to Intel SDM Vol 2.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `EmuCtxt::execute_insn` post: returned result indicates success / exception / restart-needed (REP) / userspace-handle (MMIO); exception correctly populated if returning exception | `EmuCtxt::execute_insn` |
| Per-instruction effect on EFLAGS matches Intel SDM Vol 2 specification | per-opcode handler |
| Memory access via ops.read/write/cmpxchg_emulated invokes per-vendor callback exactly once per emulator-side access | `EmuCtxt::*` |
| `linearize` post: returned linear addr satisfies `seg.base ≤ linear < seg.base + seg.limit + 1` with type-check (read/write/exec) per access_rights | `EmuCtxt::linearize` |

### Layer 4: Verus/Creusot functional

`EmuCtxt::execute_insn` ↔ Intel SDM Vol 2 specification: for any opcode + valid input register state + valid memory state, emulator's resulting register state + EFLAGS + memory write-set matches what HW would compute. Encoded as Verus model: `forall (ctxt, opcode). emulator(ctxt, opcode) == hw_semantics(ctxt, opcode)`.

### hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

emulator-specific reinforcement:

- **Per-instruction byte-stream bounded** by `insn_len` (typically 15 bytes per Intel max-instruction-length); defense against malformed instruction stream causing OOB-read.
- **Per-GDT/LDT limit checked at every seg-load** — defense against malicious guest forging GDT pointing at bogus selectors.
- **REP-loop interruption-window check** — defense against unbounded REP-prefix string-op blocking IRQ delivery.
- **Lock-prefix atomicity preserved via cmpxchg loop on host memory** — defense against torn-write race vs HW vCPUs.
- **Per-instruction CPL check** — defense against ring-3 emulating ring-0 instruction.
- **Privileged-instruction execution intercepted via per-vendor EmuOps.intercept** — VMX/SVM-specific check before invoking emulator handler.
- **Emulator-side state never leaks to userspace** — emulator's local registers + scratch state freed before return; defense against UAF if emulation aborts mid-instruction.
- **Per-instruction emulation budget** — bounded by `EMULATE_INSN_LIMIT` (default 32 instructions per single emulation call) to prevent emulator-loop DoS by guest.

