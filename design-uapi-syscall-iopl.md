---
title: "Tier-5 syscall: iopl(2) — syscall 172"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`iopl(2)` ("I/O privilege level") sets the calling process's CPL-equivalent for legacy `in`/`out` instructions and CLI/STI by adjusting the IOPL field in EFLAGS. With `level == 3` userspace can execute any I/O port instruction without trapping, equivalent to ring-0 from the I/O perspective. The syscall is a sledgehammer: it grants access to all 64K I/O ports at once, unlike `ioperm(2)` which is byte-granular. Historically required by XFree86, DOSEMU, and SVGAlib. Modern Linux strongly discourages its use — almost every legitimate need is met by `ioperm` (finer grain) or by an in-kernel driver.

The syscall is gated by `CAP_SYS_RAWIO` and is one of the most security-critical x86 entrypoints. PaX `GRKERNSEC_IO` removes it entirely. Without this protection, an `iopl(3)` followed by carefully timed `out`-port writes to chipset registers can corrupt arbitrary physical memory, disable IOMMU, or reset the CPU.

This Tier-5 covers `iopl(2)` on x86_64 and i386. There is no ARM/RISC-V analogue.

### Acceptance Criteria

- [ ] AC-1: As root (CAP_SYS_RAWIO), `iopl(3)` returns 0 and subsequent `in al, 0x80` does not trap.
- [ ] AC-2: As non-root, `iopl(3)` returns `-EPERM`.
- [ ] AC-3: As root, `iopl(4)` returns `-EINVAL`.
- [ ] AC-4: As root, `iopl(-1)` returns `-EINVAL`.
- [ ] AC-5: After `iopl(3)`, `iopl(0)` lowers IOPL without requiring fresh privilege check beyond entry-time cap.
- [ ] AC-6: After `iopl(3)` in thread A, thread B's `in al, 0x80` still traps.
- [ ] AC-7: After `fork`, child inherits parent's IOPL: child can do `in al, 0x80` without trap.
- [ ] AC-8: After `execve`, the new process's IOPL is 0; `in al, 0x80` traps.
- [ ] AC-9: On a `CONFIG_X86_IOPL_IOPERM=n` kernel, `iopl(3)` returns `-ENOSYS`.
- [ ] AC-10: On grsec `GRKERNSEC_IO=y`, `iopl(3)` returns `-ENOSYS` even for root.
- [ ] AC-11: After `iopl(3)`, `cli` at user level is honored (interrupts disabled) until next user-mode return clears the kernel-side workaround.
- [ ] AC-12: Audit log records every `iopl(level != 0)` with task cred and RIP.
- [ ] AC-13: Per-LTP `iopl01..02` pass.

### Architecture

```rust
#[syscall(nr = 172, abi = "sysv")]
pub fn sys_iopl(level: i32) -> isize {
    if grsec::io_blocked() { return -ENOSYS; }
    Iopl::dispatch(level)
}
```

`Iopl::dispatch(level) -> isize`:
1. if level < 0 || level > 3 { return Err(EINVAL); }
2. let cur = current.thread.iopl as i32;
3. if level > cur && !capable(CAP_SYS_RAWIO) { return Err(EPERM); }
4. /* Audit non-zero */
5. if level != 0 {
6.   audit::log_iopl(current, level);
7. }
8. /* Update */
9. current.thread.iopl = level as u8;
10. #[cfg(iopl_iopem_emul)]
11. {
12.   /* Emulated path: do not flip real EFLAGS.IOPL; instead set per-task I/O bitmap */
13.   IoBitmap::install_for_iopl(current, level);
14. }
15. #[cfg(not(iopl_iopem_emul))]
16. {
17.   /* Real path: schedule EFLAGS update on return-to-user */
18.   Pcpu::request_eflags_iopl_write(level);
19. }
20. Ok(0)

`switch_to(prev, next)`:
1. if prev.thread.iopl != next.thread.iopl {
2.   write_eflags_iopl(next.thread.iopl);
3. }

`fork_thread(parent, child)`:
1. child.thread.iopl = parent.thread.iopl;

`execve(task)`:
1. task.thread.iopl = 0;

### Out of Scope

- `ioperm(2)` byte-granular variant (separate Tier-5 doc).
- x86 EFLAGS architecture (Tier-3 in `arch/x86/eflags.md`).
- I/O bitmap mechanism (Tier-3 in `arch/x86/ioperm.md`).
- IOMMU / chipset register protection (Tier-3 in `drivers/iommu/00-overview.md`).
- Audit subsystem (Tier-3 in `kernel/audit.md`).
- Implementation code.

### signature

```c
int iopl(int level);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `level` | `int` | in | Target IOPL value (0, 1, 2, or 3). Linux honors only 0 and 3 in modern kernels. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. EFLAGS.IOPL updated for current thread. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `level > 3` or `level < 0`. |
| `EPERM`   | Lacking `CAP_SYS_RAWIO` in the calling task's user namespace; or grsec `GRKERNSEC_IO` blocks; or trying to raise IOPL above current. |
| `ENOSYS`  | Compiled out (rare; default-off on hardened kernels). |

### abi surface

```text
__NR_iopl (x86_64) = 172
__NR_iopl (i386)   = 110

IOPL values:
  0 — default; in/out from CPL 3 traps with #GP
  1 — historical; Linux treats as 0
  2 — historical; Linux treats as 0
  3 — full access; in/out at CPL 3 succeed; CLI/STI also permitted

Required capability: CAP_SYS_RAWIO (capability bit 17)
```

### Limits

```text
level         : 0 .. 3 (only 0 and 3 are functionally distinct)
ports granted : per-iopl(3): all 65536 I/O ports (0x0000..0xFFFF)
CLI/STI       : per-iopl(3): permitted at CPL 3 (DANGEROUS — disables interrupts)
```

### compatibility contract

REQ-1: Syscall number is **172** on x86_64, **110** on i386. ABI-stable since 1.0.

REQ-2: `level > 3` or `level < 0` returns `-EINVAL` without state mutation.

REQ-3: The caller MUST possess `CAP_SYS_RAWIO` in the user namespace owning the task (typically init_user_ns); else `-EPERM`. The capability check is at the BEGINNING of the syscall — before any other check.

REQ-4: `iopl(level)` writes `level` into the IOPL field (bits 12-13) of EFLAGS for the calling THREAD, not process: each thread has its own EFLAGS context, and the kernel restores it on every user-mode return. Threads in the same `mm` therefore have independent IOPLs.

REQ-5: Raising IOPL above the current value requires `CAP_SYS_RAWIO`. Lowering it is unrestricted (one cannot grant oneself more privilege without the cap).

REQ-6: Per-iopl(3): the task can execute `in`, `out`, `inb`, `outb`, etc. on any port AND can execute `cli` and `sti` at user level. This is equivalent to disabling per-system interrupts globally if abused; the kernel runs with interrupts disabled until the next user-mode return restores EFLAGS.

REQ-7: Per-fork: child inherits parent's IOPL value (clone semantics; `task.thread.iopl` is copied).

REQ-8: Per-execve: IOPL is reset to 0. A setuid binary cannot accidentally inherit an iopl(3) state.

REQ-9: Per-context-switch path: the kernel loads the next task's IOPL bits into EFLAGS before the user-mode return; on entry from user, the kernel re-saves them.

REQ-10: `task.thread.iopl_emul`: under `CONFIG_X86_IOPL_IOPERM`, the kernel may emulate IOPL via the I/O bitmap rather than the real EFLAGS.IOPL bits, to keep CLI/STI safely virtualized. The emulation is transparent to userland but limits actual `cli`/`sti` execution to the kernel.

REQ-11: Per-CONFIG_X86_IOPL_IOPERM=n: `iopl` returns `-ENOSYS`. Default-on for kernels ≥ 5.5 but can be disabled in hardened builds.

REQ-12: Per-grsec `GRKERNSEC_IO=y`: `iopl` is removed from the syscall table entirely; returns `-ENOSYS`.

REQ-13: Per-CRIU: dump records `task.thread.iopl`; restore re-issues `iopl(level)` for each task. CRIU requires `CAP_SYS_RAWIO` to restore.

REQ-14: `level == 3` raises a kernel WARN (rate-limited) if the task does not own a chardev that legitimately requires it (best-effort heuristic; not a hard policy).

REQ-15: Per-thread isolation: thread A's `iopl(3)` does NOT grant thread B (in the same `mm`) access. Each thread's EFLAGS is independent.

REQ-16: x32 ABI: same number (172) with `0x40000000` mask. Behavior identical.

REQ-17: Compat (32-bit on 64-bit kernel): `iopl(110)` traps into the same handler; behavior identical except return path is via the 32-bit compat exit code.

REQ-18: Audit: every successful `iopl(level)` where `level != 0` is logged to the audit subsystem with task creds, RIP, and the level requested. Failed attempts (EPERM) are also logged.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `level_in_range` | INVARIANT | per-dispatch: level ∈ {0,1,2,3} or EINVAL. |
| `cap_required_for_raise` | INVARIANT | per-dispatch: level > current ⟹ CAP_SYS_RAWIO. |
| `per_thread_state` | INVARIANT | per-dispatch: only `current.thread.iopl` mutated. |
| `execve_clears_iopl` | INVARIANT | per-execve: iopl == 0. |
| `audit_for_nonzero` | INVARIANT | per-dispatch: level != 0 ⟹ audit record emitted. |

### Layer 2: TLA+

`arch/x86/iopl.tla`:
- States: per-thread iopl ∈ {0,1,2,3}; CPU EFLAGS.IOPL.
- Properties:
  - `safety_cap_for_raise` — raising IOPL requires CAP_SYS_RAWIO.
  - `safety_per_thread` — iopl is per-thread, not per-process.
  - `safety_execve_resets` — execve sets iopl = 0.
  - `safety_audit_emitted` — non-zero level ⟹ audit record.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ok ⟹ iopl == level | `Iopl::dispatch` |
| `dispatch` post: err ⟹ iopl unchanged | `Iopl::dispatch` |
| `switch_to` post: EFLAGS.IOPL == next.thread.iopl | `switch_to` |
| `fork_thread` post: child.iopl == parent.iopl | `fork_thread` |
| `execve` post: task.iopl == 0 | `execve` |

### Layer 4: Verus/Creusot functional

Per-`iopl(2)` man-page semantic equivalence. LTP `iopl01..02` pass. Manual XFree86 / DOSEMU smoke optional.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`iopl(2)` reinforcement:

- **Per-CAP_SYS_RAWIO strict** — defense against per-non-root I/O escalation; the cap is the single gate.
- **Per-thread isolation** — defense against per-thread-B inheriting thread-A's I/O privilege.
- **Per-execve reset** — defense against per-suid binary inheriting attacker IOPL.
- **Per-iopl_iopem emulation** — defense against per-CLI-via-userspace (kernel never actually allows user `cli` to take effect even with EFLAGS.IOPL=3).
- **Per-audit-all** — defense against per-undetected privilege escalation; every non-zero call recorded.
- **Per-WARN on legitimate-driver-heuristic miss** — defense against per-stealth I/O privilege grant by suid helpers.
- **Per-CONFIG_X86_IOPL_IOPERM-required** — defense against legacy kernels lacking emulation.

### grsecurity / pax surface

- **GRKERNSEC_IO** — removes `iopl(2)` from the syscall table entirely. Returns `-ENOSYS` regardless of capabilities. This is the recommended hardened default.
- **PaX UDEREF** — no user pointer in `iopl`, but EFLAGS load path on syscall return is checked against per-task allowed range.
- **PAX_RANDKSTACK at iopl entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — kernel never exposes `task.thread.iopl` via `/dev/kmem` or `/proc/<pid>/mem`; reachable only via the syscall return path.
- **PaX MPROTECT** — once `iopl(3)` granted, the task's `mm` is flagged so that `mmap(PROT_EXEC)` of fresh anonymous pages is denied (the I/O privilege + JIT combination is forbidden).
- **GRKERNSEC_CHROOT_NO_IOPL** — inside a grsec chroot, `iopl(level != 0)` returns `-EPERM` unconditionally.
- **Per-grsec RBAC** — `+W` flag (raw-I/O-allow) gates per-subject; default policy denies.
- **Per-grsec audit** — every successful `iopl(level != 0)` logged with task cred + RIP + level; every failure also logged.
- **Per-grsec `iopl(3) ⟹ no_new_privs`** — after a successful `iopl(3)`, the task is forced into `PR_SET_NO_NEW_PRIVS=1` mode so it cannot `execve` a suid binary to launder further capabilities.

