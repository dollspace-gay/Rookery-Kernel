# Tier-5 syscall: ioperm(2) — syscall 173

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/kernel/ioport.c (ksys_ioperm)
  - arch/x86/kernel/process.c (io_bitmap_share)
  - arch/x86/include/asm/io_bitmap.h
  - arch/x86/entry/syscalls/syscall_64.tbl (173  common  ioperm)
  - arch/x86/entry/syscalls/syscall_32.tbl (101  i386    ioperm)
-->

## Summary

`ioperm(2)` grants or revokes byte-granular access to legacy x86 I/O ports for the calling thread by toggling bits in the per-task I/O permission bitmap (IOPB). It is the finer-grained alternative to `iopl(2)`: where `iopl(3)` enables all 65 536 ports at once, `ioperm` enables a contiguous range of `num` ports starting at `from`. Userspace programs (XFree86, lm-sensors, parport_serial helper) historically used `ioperm` to access specific port ranges (`0x3C0..0x3DF` for VGA, `0x378..0x37F` for parallel port, `0x70..0x71` for RTC) without the system-wide blast radius of `iopl(3)`.

`ioperm` is gated by `CAP_SYS_RAWIO`. Only the first 0x400 ports (1024) are addressable by `ioperm`; for higher ports `iopl(3)` is required. The IOPB is installed in the per-CPU TSS on context switch.

PaX `GRKERNSEC_IO` removes the syscall entirely. Without this protection, `ioperm` on chipset / ACPI / SMBus ports can be used to bypass IOMMU programming, reset peripherals, or extract platform secrets.

This Tier-5 covers `ioperm(2)` on x86_64 and i386. There is no ARM/RISC-V analogue.

## Signature

```c
int ioperm(unsigned long from, unsigned long num, int turn_on);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `from` | `unsigned long` | in | First I/O port (inclusive). Must be < 0x400 (1024). |
| `num` | `unsigned long` | in | Number of ports to grant/revoke. `from + num` must be ≤ 0x400. |
| `turn_on` | `int` | in | Non-zero = grant; zero = revoke. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `from + num > 0x400`; or `from > 0xFFFF`; or `num == 0` (Linux historical: accepted with no-op). |
| `EPERM`   | Lacking `CAP_SYS_RAWIO`; or grsec `GRKERNSEC_IO` blocks. |
| `ENOMEM`  | Per-IOPB allocation failed (first call only — 8 KiB bitmap). |
| `ENOSYS`  | Compiled out. |

## ABI surface

```text
__NR_ioperm (x86_64) = 173
__NR_ioperm (i386)   = 101

I/O port range covered by ioperm: 0x0000 .. 0x03FF (first 1024 ports)
I/O port range above 0x03FF      : requires iopl(3) — ioperm returns -EINVAL

IOPB size                        : 8 KiB (65536 bits, one per port)
IOPB max active bytes per task    : 128 bytes (1024 bits) when ioperm-only is used

Required capability               : CAP_SYS_RAWIO

Commonly-granted port ranges (informational):
  0x60-0x64  i8042 keyboard/mouse controller (denied in modern Linux)
  0x70-0x71  RTC / NMI mask
  0x80       diagnostic / POST checkpoint
  0x3C0-0x3DF VGA
  0x378-0x37F LPT1
  0x2F8/0x3F8 UART
```

### Limits

```text
from + num    : ≤ 0x400 (1024)
num           : 0 .. 0x400 (effectively; 0 is no-op)
ports per task: max 1024 (range covered by IOPB low region exposed by ioperm)
IOPB lifetime : persists for life of task; freed on exit or execve
```

## Compatibility contract

REQ-1: Syscall number is **173** on x86_64, **101** on i386. ABI-stable.

REQ-2: `from + num > 0x400` returns `-EINVAL`. (For higher ports, the caller must use `iopl(3)`.)

REQ-3: `from + num` arithmetic overflow (e.g. `from = ULONG_MAX, num = 1`) returns `-EINVAL`.

REQ-4: Caller MUST possess `CAP_SYS_RAWIO` in the user namespace owning the task; else `-EPERM`. Check is at syscall entry.

REQ-5: First successful `ioperm` allocates the per-task IOPB (8 KiB, `struct io_bitmap`) and stores a pointer in `task.thread.io_bitmap`. Subsequent calls update the existing bitmap in place.

REQ-6: `turn_on != 0`: clears bits `from..from+num-1` in the IOPB (the bitmap encodes "0 = allowed", "1 = denied" per the x86 TSS spec).

REQ-7: `turn_on == 0`: sets bits `from..from+num-1` (revoke).

REQ-8: After update, the kernel marks `task.thread.io_bitmap` dirty so the next return-to-user reloads the TSS IOPB pointer for this task.

REQ-9: Per-context-switch: when scheduling a task with `io_bitmap != NULL`, the kernel copies the bitmap into the per-CPU TSS at offset `iomap_base` (or lazily switches via the TSS_IO_BITMAP scheme). Tasks without an IOPB get the "deny-all" sentinel (all-ones bitmap) effectively.

REQ-10: Per-fork: child shares the parent's IOPB via refcount + COW (the per-task `io_bitmap` is shared until one mutates).

REQ-11: Per-execve: IOPB is freed; new process starts with no IOPB (all ports denied to userspace).

REQ-12: Per-thread scope: each thread has its own `task.thread.io_bitmap` pointer. `ioperm` in thread A does NOT grant access to thread B.

REQ-13: Per-CRIU: dump records the IOPB bits via `/proc/<pid>/task/<tid>/io_bitmap` (debugfs path) or via `ptrace`-equivalent introspection; restore re-issues `ioperm` calls per range.

REQ-14: Per-CONFIG_X86_IOPL_IOPERM=n: returns `-ENOSYS`. Default-on for kernels ≥ 5.5.

REQ-15: Per-grsec `GRKERNSEC_IO=y`: returns `-ENOSYS`.

REQ-16: `num == 0`: historically accepted as no-op returning 0. Modern Linux still tolerates this.

REQ-17: Audit: every successful `ioperm` with `turn_on != 0` logged with `from`, `num`, task cred, RIP. Revocations not audited by default.

REQ-18: x32 ABI: same number (173) with `0x40000000` mask. Behavior identical.

## Acceptance Criteria

- [ ] AC-1: As root, `ioperm(0x80, 1, 1)` returns 0; subsequent `out 0x80, al` from user mode succeeds without trap.
- [ ] AC-2: As root, `ioperm(0x80, 1, 1)` then `ioperm(0x80, 1, 0)` revokes; subsequent `out 0x80, al` traps with SIGSEGV.
- [ ] AC-3: As non-root, `ioperm(0x80, 1, 1)` returns `-EPERM`.
- [ ] AC-4: As root, `ioperm(0x3FF, 2, 1)` returns `-EINVAL` (range crosses 0x400).
- [ ] AC-5: As root, `ioperm(0x400, 1, 1)` returns `-EINVAL`.
- [ ] AC-6: As root, `ioperm(ULONG_MAX, 1, 1)` returns `-EINVAL`.
- [ ] AC-7: As root, `ioperm(0x70, 2, 1)` allows reading RTC; thread B in the same `mm` still traps on `in al, 0x70`.
- [ ] AC-8: After `fork`, child can `out` to the parent's granted range; after child issues `ioperm(0x80, 1, 0)`, parent's access at 0x80 is unaffected (COW).
- [ ] AC-9: After `execve`, the new process traps on `out al, 0x80` even if the previous binary had it granted.
- [ ] AC-10: On `CONFIG_X86_IOPL_IOPERM=n`, `ioperm` returns `-ENOSYS`.
- [ ] AC-11: On grsec `GRKERNSEC_IO=y`, `ioperm` returns `-ENOSYS` for all callers.
- [ ] AC-12: Audit log records every `ioperm(from, num, 1)` with task cred and RIP.
- [ ] AC-13: Per-LTP `ioperm01..02` pass.

## Architecture

```rust
#[syscall(nr = 173, abi = "sysv")]
pub fn sys_ioperm(from: usize, num: usize, turn_on: i32) -> isize {
    if grsec::io_blocked() { return -ENOSYS; }
    Ioperm::dispatch(from, num, turn_on != 0)
}
```

`Ioperm::dispatch(from, num, grant) -> isize`:
1. /* Bounds */
2. if from.checked_add(num).map_or(true, |e| e > 0x400) {
3.   return Err(EINVAL);
4. }
5. /* Capability */
6. if !capable(CAP_SYS_RAWIO) { return Err(EPERM); }
7. /* Allocate or COW bitmap */
8. let task = current;
9. let bm = task.thread.io_bitmap_mut_cow()?;     // ENOMEM possible
10. /* Update bits */
11. if grant {
12.   bm.clear_range(from, num);                 // 0 = allowed
13.   audit::log_ioperm(task, from, num);
14. } else {
15.   bm.set_range(from, num);                   // 1 = denied
16. }
17. /* Mark dirty */
18. task.thread.io_bitmap_dirty = true;
19. Ok(0)

`switch_to(prev, next)`:
1. if next.thread.io_bitmap_dirty {
2.   tss::install_io_bitmap(&next.thread.io_bitmap);
3.   next.thread.io_bitmap_dirty = false;
4. } else if !prev.thread.io_bitmap.is_none() && next.thread.io_bitmap.is_none() {
5.   tss::install_deny_all();
6. }

`fork_thread(parent, child)`:
1. if let Some(bm) = &parent.thread.io_bitmap {
2.   child.thread.io_bitmap = Some(bm.clone_refcounted());
3. }

`execve(task)`:
1. drop(task.thread.io_bitmap.take());

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `range_bounded` | INVARIANT | per-dispatch: from + num ≤ 0x400 or EINVAL. |
| `overflow_safe` | INVARIANT | per-dispatch: from + num uses checked_add. |
| `cap_required` | INVARIANT | per-dispatch: CAP_SYS_RAWIO checked before mutation. |
| `cow_on_mutate` | INVARIANT | per-dispatch: shared bitmap copied before mutate. |
| `execve_frees_bitmap` | INVARIANT | per-execve: io_bitmap == None. |
| `audit_for_grant` | INVARIANT | per-dispatch: grant ⟹ audit emitted. |

### Layer 2: TLA+

`arch/x86/ioperm.tla`:
- States: per-task io_bitmap (option, refcounted); per-cpu TSS IOPB image.
- Properties:
  - `safety_range_low` — granted ports always < 0x400.
  - `safety_cap` — only CAP_SYS_RAWIO tasks can mutate.
  - `safety_per_thread` — bitmap is per-task, COW on fork.
  - `safety_execve_resets` — execve clears bitmap.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ok ⟹ bits in [from, from+num) updated | `Ioperm::dispatch` |
| `dispatch` post: err ⟹ bitmap unchanged | `Ioperm::dispatch` |
| `switch_to` post: TSS IOPB == next.thread.io_bitmap | `switch_to` |
| `fork_thread` post: child.io_bitmap aliases parent's via refcount | `fork_thread` |
| `execve` post: io_bitmap == None | `execve` |

### Layer 4: Verus/Creusot functional

Per-`ioperm(2)` man-page semantic equivalence. LTP `ioperm01..02` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ioperm(2)` reinforcement:

- **Per-CAP_SYS_RAWIO strict** — defense against per-non-root I/O escalation.
- **Per-`from + num ≤ 0x400`** — defense against per-IOPB-overflow into adjacent TSS fields.
- **Per-checked_add** — defense against per-integer-overflow ACL bypass.
- **Per-COW bitmap on fork** — defense against per-cross-task IOPB corruption.
- **Per-execve free** — defense against per-suid binary inheriting attacker I/O perms.
- **Per-thread isolation** — defense against per-thread-B inheriting thread-A's I/O privilege.
- **Per-audit on grant** — defense against per-undetected I/O privilege grant.
- **Per-TSS IOPB lazy install** — defense against per-stale-IOPB after task termination.

## Grsecurity / PaX surface

- **GRKERNSEC_IO** — removes `ioperm(2)` (and `iopl(2)`) from the syscall table entirely. Returns `-ENOSYS`. Recommended hardened default.
- **PaX UDEREF** — no user pointer in `ioperm`, but IOPB load in TSS is checked against per-task allowed range.
- **PAX_RANDKSTACK at ioperm entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — kernel never exposes IOPB via `/dev/kmem` or `/proc/<pid>/mem`.
- **PaX MPROTECT** — once `ioperm(..., 1)` granted, the task's `mm` is flagged so that `mmap(PROT_EXEC)` of fresh anonymous pages is denied (I/O privilege + JIT combination is forbidden).
- **GRKERNSEC_CHROOT_NO_IOPL** — inside a grsec chroot, `ioperm(..., 1)` returns `-EPERM` unconditionally.
- **Per-grsec RBAC** — `+W` flag (raw-I/O-allow) gates per-subject; default policy denies.
- **Per-grsec audit** — every `ioperm(from, num, 1)` logged; every failure also logged.
- **Per-grsec per-port allowlist** — even with CAP_SYS_RAWIO, only ports in `/sys/kernel/grsec/io_allow` may be granted; others return `-EPERM`.
- **PaX TSS-IOPB consistency check** — IOPB pointer in TSS validated against `task.thread.io_bitmap` on every context-switch; mismatch panics.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `iopl(2)` whole-range variant (separate Tier-5 doc).
- x86 TSS / IOPB hardware semantics (Tier-3 in `arch/x86/tss.md`).
- I/O port routing / ACPI mapping (Tier-3 in `arch/x86/io_apic.md`).
- IOMMU programming via I/O ports (Tier-3 in `drivers/iommu/00-overview.md`).
- Implementation code.
