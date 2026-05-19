---
title: "Tier-5 special entry: uretprobe — userspace-probe return-trampoline"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`uretprobe` is the **return-side counterpart** of a uprobe: when a registered consumer provides a `ret_handler`, the kernel arranges that the tripped function's return is also instrumented. At entry, the kernel records the original return address into a per-task `return_instance` stack and rewrites the in-userspace return slot to point at a kernel-installed **return trampoline** living in the per-mm XOL/uretprobe VMA. When the function eventually executes its return, control lands at the trampoline, which raises a second trap (or, on x86_64 hardened kernels, performs a fast syscall `__NR_uretprobe`) and re-enters the kernel; the kernel then runs each consumer's `ret_handler`, restores the original return address, and resumes userspace at that address.

This document covers the **return-trampoline contract** and (where present) the x86_64 fast-path `uretprobe` syscall. It is paired with `uprobe.md`.

Critical for: BPF function-return tracing (`bpftrace 'uretprobe:/lib/libc.so:malloc { @[retval] = count(); }'`), latency measurement (entry-to-return delta), profilers that need both entry and return events, security agents that audit returned values.

### Acceptance Criteria

- [ ] AC-1: Registering a uprobe with a `ret_handler` on `malloc`: entry handler + return handler both fire, return handler observes `regs.ax = malloc-return`.
- [ ] AC-2: Userspace calls `syscall(__NR_uretprobe)` directly: `-ENOSYS`.
- [ ] AC-3: Recursive instrumented function (depth 10): each call pairs entry/return exactly once.
- [ ] AC-4: `longjmp` past 3 instrumented frames: 3 stale `return_instances` discarded; original return reached.
- [ ] AC-5: Concurrent same-function calls on two CPUs: each CPU sees its own `return_instance`.
- [ ] AC-6: `MAX_URETPROBE_DEPTH+1` nested calls: deepest skipped; original return preserved.
- [ ] AC-7: Signal mid-call: sigreturn unwinds `return_instances`.
- [ ] AC-8: `fork` mid-instrumented-call: child observes pending uretprobes; child return triggers handlers.
- [ ] AC-9: `execve` mid-instrumented-call: list cleared.
- [ ] AC-10: BPF `ret_handler` reads return value via `bpf_get_func_ret`.
- [ ] AC-11: Trampoline at forged IP (manually written `int3` at non-trampoline address): SIGSEGV / SIGTRAP, no handler runs.
- [ ] AC-12: Removing the last consumer with `ret_handler`: trampoline VMA persists (idempotent) but is unused.

### Architecture

```rust
/* Fast-path syscall (x86_64): only callable via kernel-installed trampoline. */
#[syscall(nr = 335, abi = "sysv", gated_internal = true)]
pub fn sys_uretprobe() -> isize {
    Uretprobe::sys_uretprobe_entry()
}
```

`Uretprobe::sys_uretprobe_entry() -> isize`:
1. let regs = current_pt_regs_mut();
2. let mm = current_mm();
3. /* Reject direct userspace invocation: only the trampoline IP is permitted */
4. let tramp = mm.uprobes_state.tramp_vaddr;
5. if regs.ip() != tramp { return -ENOSYS; }
6. Uretprobe::handle_trampoline(regs);
7. /* Return value is irrelevant; the trampoline path resets regs.ip before iret */
8. 0

`Uretprobe::handle_trampoline(regs)`:
1. let utask = current().utask_mut();
2. /* Walk LIFO matching on stack pointer */
3. while let Some(ri) = utask.return_instances.head() {
4.   if Uretprobe::stack_matches(ri, regs) { break; }
5.   /* Stale: longjmp / unwind past this entry */
6.   utask.return_instances.pop();
7. }
8. let Some(ri) = utask.return_instances.pop() else {
9.   Signal::force_sig_info(SIGILL, regs);
10.  return;
11. };
12. /* Dispatch ret_handlers */
13. let _rcu = rcu_read_lock();
14. for cons in ri.uprobe.consumers.iter() {
15.   if let Some(rh) = cons.ret_handler {
16.     (rh)(cons, ri.func, regs);
17.   }
18. }
19. /* Restore the original return target */
20. regs.set_ip(ri.orig_ret_vaddr);

`Uretprobe::prepare_uretprobe(uprobe, regs)`:                /* called from uprobe entry path */
1. let utask = current().utask_mut();
2. if utask.return_instances.depth() >= MAX_URETPROBE_DEPTH { return; }
3. let user_sp = regs.sp();
4. let orig_ret = UserPtr::<usize>::from_addr(user_sp).read()?;
5. let ri = ReturnInstance {
6.   uprobe, func: regs.ip(), stack: user_sp,
7.   orig_ret_vaddr: orig_ret, chained: false, next: utask.return_instances.head_link(),
8. };
9. /* Rewrite user return slot to trampoline */
10. UserPtr::<usize>::from_addr(user_sp).write(mm.uprobes_state.tramp_vaddr)?;
11. utask.return_instances.push(ri);
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lifo_ordering` | INVARIANT | per-handle_trampoline: matched ri is top-of-stack (after stale discard). |
| `tramp_ip_validated` | INVARIANT | per-sys_uretprobe: regs.ip == tramp_vaddr, else ENOSYS. |
| `orig_ret_restored` | INVARIANT | per-handle_trampoline: post-call regs.ip == ri.orig_ret_vaddr. |
| `depth_bound` | INVARIANT | per-prepare_uretprobe: depth <= MAX_URETPROBE_DEPTH after push. |
| `user_slot_rewrite_safe` | INVARIANT | per-prepare_uretprobe: write at user_sp uses UDEREF-guarded UserPtr. |
| `stale_unwind_terminates` | INVARIANT | per-handle_trampoline: stale-discard loop terminates (depth monotone decreasing). |

### Layer 2: TLA+

`kernel/uretprobe.tla`:
- States: per-task return_instance list, per-mm trampoline vaddr, per-task IP.
- Properties:
  - `safety_lifo_pairing` — each successful return-handler invocation pops its matching entry.
  - `safety_no_handler_without_register` — ret_handler runs only when consumer registered with ret_handler.
  - `safety_direct_syscall_denied` — sys_uretprobe with non-trampoline IP returns ENOSYS, no handler runs.
  - `safety_depth_bound` — depth never exceeds MAX_URETPROBE_DEPTH.
  - `safety_orig_return_reached` — handled trampoline ⟹ task resumes at ri.orig_ret_vaddr.
  - `liveness_trap_resumes` — every trampoline trap returns control.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_uretprobe_entry` post: ip mismatch ⟹ -ENOSYS ∧ no state change | `Uretprobe::sys_uretprobe_entry` |
| `handle_trampoline` post: regs.ip == ri.orig_ret_vaddr ∨ SIGILL | `Uretprobe::handle_trampoline` |
| `prepare_uretprobe` post: depth incremented by 1 ∨ at-cap | `Uretprobe::prepare_uretprobe` |
| `stack_matches` pure: equality on (ri.stack, regs.sp) up to ABI alignment | `Uretprobe::stack_matches` |

### Layer 4: Verus / Creusot functional

Per-`Documentation/trace/uprobetracer.rst` "return probes" section; `tools/testing/selftests/bpf/uprobe_multi*` (return-side tests) pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`uretprobe` trampoline reinforcement:

- **Per-IP trampoline gate (sys_uretprobe)** — defense against per-userspace direct `__NR_uretprobe` abuse.
- **Per-LIFO matching** — defense against per-out-of-order handler invocation.
- **Per-MAX_URETPROBE_DEPTH cap** — defense against per-unbounded recursion DoS.
- **Per-stale-entry discard** — defense against per-longjmp-orphan handler firing on wrong frame.
- **Per-UDEREF user-slot rewrite** — defense against per-kernel-range orig-ret smuggle.
- **Per-RCU consumer iteration** — defense against per-consumer-list mutation race.
- **Per-trampoline-VMA RX-only** — defense against per-trampoline-payload write.

## Grsecurity / PaX surface

- **Verifier-gated uprobe/uretprobe attach (CAP_SYS_ADMIN-attached only)** — uretprobe registration inherits uprobe's elevated cap requirement under grsec. In-container `ret_handler` attach is denied unless an explicit `bpf_token` delegated the cap from `init_user_ns`.
- **GRKERNSEC_HARDEN_TRACE** — disables uretprobe entirely (and all userspace tracing). Default in many production hardening profiles.
- **PAX_KERNEXEC for trampoline install** — the per-mm trampoline VMA is created RX-only, with the trampoline insn written before the page is flipped to RX; the page is never simultaneously writable + executable.
- **PaX UDEREF on user return-slot rewrite** — `prepare_uretprobe` writes the trampoline vaddr into user-stack via UDEREF-guarded `UserPtr::write`, so a forged `regs->sp` cannot redirect the write to kernel memory.
- **GRKERNSEC_HIDESYM on trampoline address** — `[uprobes]` (or `[uretprobes]`) VMA visible in `/proc/<pid>/maps` but the exact slot offsets are scrubbed; user JOP/ROP gadget hunting via stable trampoline IPs is denied.
- **PAX_REFCOUNT on uretprobe consumer refcounts** — saturating; over-deregister underflow panics.
- **GRKERNSEC_AUDIT_BPF audits ret_handler attach** — every uretprobe register/unregister logged with caller creds.
- **CAP split (CAP_BPF + CAP_PERFMON)** — even with grsec relaxed, bpf-link uretprobe attach requires both caps.
- **sys_uretprobe gated to trampoline IP** — grsec adds an extra check that the trampoline vaddr is inside the mm's uprobes_state vma range; arbitrary syscall entries with IP near (but not at) the trampoline are rejected. Defeats trampoline-spoof variants.
- **GRKERNSEC_RAND_THREADSTACK reinforcement** — when thread-stack randomization is on, `regs->sp` mismatch is the common stale-frame case; the unwind discard logic must be robust. PaX adds a sanity check that discarded stale entries don't exceed `MAX_URETPROBE_DEPTH` per call (DoS bound).
- **No_new_privs neutral on tracing target** — but NNP-set tasks deny new uretprobe attach from outside (per consumer filter).
- **Anti-fingerprint** — trampoline slot offsets within the XOL VMA are randomized per mm.
- **PAX_NOEXEC compatible** — trampoline VMA RX-only; never RWX.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `uprobe` entry trampoline (Tier-5 separate doc — `uprobe.md`).
- `bpf(BPF_LINK_CREATE, BPF_TRACE_UPROBE_MULTI, ...)` registration ABI (covered in `bpf.md`).
- `perf_event_open` uprobe PMU (covered in `perf_event_open.md`).
- arch-specific return-address-hijack semantics (Tier-3 in `arch/<a>/uprobes.md`).
- Implementation code.

### signature

```c
/* The x86_64 fast-path syscall (post 6.11): */
long uretprobe(void);
/* Userspace MUST NOT invoke this directly; it is reached only via the
 * kernel-installed trampoline. Direct invocation: ENOSYS. */
```

```c
struct return_instance {
    struct uprobe          *uprobe;
    unsigned long           func;         /* function entry address */
    unsigned long           stack;        /* user stack pointer at entry */
    unsigned long           orig_ret_vaddr;
    bool                    chained;
    struct return_instance *next;
};
```

### parameters

(no user parameters — the entry IS the trampoline.)

| Implicit input | Source | Meaning |
|---|---|---|
| `regs->sp` | trap frame | The user stack at the trampoline; used to locate the matching `return_instance`. |
| `current->utask->return_instances` | per-task | Linked list of pending uretprobes, LIFO. |
| `regs->ip` | trap frame | Address of the trampoline (validated against `current->mm->uprobes_state.xol_area.vaddr + slot_off`). |

### return value

(no return — control flows to the original return target after handlers run.)

### errors

| Condition | Effect |
|---|---|
| `current->utask->return_instances` empty | `SIGILL` (impossible-by-construction guard). |
| Trampoline IP mismatch (forged trap) | `SIGSEGV`. |
| Userspace invoked `uretprobe(2)` directly | `-ENOSYS`. |
| Stack-pointer mismatch (longjmp past instrumented frame) | `return_instance` stack unwound; orphaned entries discarded. |

### abi surface

```text
__NR_uretprobe (x86_64) = 335   /* fast-path; gate: only via trampoline */

/* Per-arch return trampoline instructions in XOL VMA: */
x86_64: 0x0F 0x05                /* syscall (post-6.11 fast path);
                                    pre-6.11: int3 + return-handler */
arm64 : BRK #UPROBE_BRK_UPROBE_BP /* 0xD420C000 */
riscv : ebreak

/* The uretprobe syscall is registered with a gating helper:
 *   sys_uretprobe verifies regs->ip == per-mm uretprobe trampoline vaddr;
 *   on mismatch, the call is rejected with -ENOSYS to deny userspace abuse. */
```

### compatibility contract

REQ-1: `uretprobe(2)` (where present) is a **gated, kernel-trampoline-only** syscall. Userspace cannot legitimately issue it; the kernel rejects calls whose `regs->ip` is not the per-mm uretprobe trampoline address.

REQ-2: On uprobe-entry, if any registered consumer has a `ret_handler`:
  - `prepare_uretprobe` is invoked.
  - The current return address (the value at `*(user_sp)` on x86) is captured into a fresh `return_instance` pushed onto `current->utask->return_instances`.
  - The user return slot is rewritten to point at the trampoline.

REQ-3: On trampoline trap (or fast-syscall on x86_64 post-6.11), `handle_trampoline`:
  - Locates the matching `return_instance` (LIFO; matches by `stack` pointer / direction).
  - For every consumer with a `ret_handler`, invokes it with `(consumer, ri->func, regs)`.
  - Restores `regs->ip` to `ri->orig_ret_vaddr`.
  - Pops `return_instance` from the per-task list.

REQ-4: `longjmp` / exception-unwind past instrumented frames: subsequent trampoline hits may find that the captured `stack` no longer matches the current `regs->sp`. The kernel walks and discards stale `return_instance` entries until a match is found; if none, the unwind chain is fully cleared and a SIGILL is delivered only if invariants are violated (normally just clears silently).

REQ-5: Re-entrancy: if a uretprobe-instrumented function is called recursively, each call gets its own `return_instance`; the LIFO ordering matches the C-ABI call/return discipline.

REQ-6: `chained` flag: when a uretprobe-traced function returns into another uretprobe-traced function (mutual instrumentation), the second `return_instance` is marked `chained` so the kernel does not double-rewrite.

REQ-7: Registration: `ret_handler != NULL` in a `uprobe_consumer` at `uprobe_register` time. Capability requirements identical to `uprobe.md` REQ-6.

REQ-8: BPF return programs see `pt_regs` with the return value visible in the conventional register (`rax` x86_64, `x0` arm64). Helper `bpf_get_func_ret(ctx, &val)` is the portable accessor.

REQ-9: The trampoline VMA is part of (or adjacent to) the XOL VMA: RX, per-mm, lazily allocated. It is installed when the first uretprobe-bearing consumer registers in that mm.

REQ-10: Maximum nesting per task: `MAX_URETPROBE_DEPTH` (kernel-configurable, default 64). Exceeding it: extra entries silently dropped, original return preserved (no nesting).

REQ-11: The fast-syscall path (x86_64, recent kernels) is the only legitimate use of `__NR_uretprobe = 335`. The pre-fast-path int3 trampoline remains the fallback when SYSCALL_DEFINE0(uretprobe) is unavailable.

REQ-12: Signal handling: a signal that interrupts an instrumented function before its return triggers a sigreturn into the original frame; the matching `return_instance` is unwound at sigreturn time so the trampoline is not re-entered after the longjmp-class non-local exit.

REQ-13: Forking: `fork()` copies `return_instances` list; child observes the same pending uretprobes as parent at fork point.

REQ-14: `execve`: clears all `return_instances` (mm replacement invalidates the user-side state).

REQ-15: `__NR_uretprobe` is **not** a stable userspace ABI; it is documented as gated-internal; direct callers receive `-ENOSYS`.

