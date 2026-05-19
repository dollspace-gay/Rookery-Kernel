---
title: "Tier-5 syscall: close_range(2) — syscall 436"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`close_range(2)` atomically closes a contiguous range of file descriptors `[first, last]` inclusive, replacing the post-fork "loop 3..1024, close(fd)" idiom that has been the cause of countless fd-leak bugs and pre-exec performance hot-spots. Flag `CLOSE_RANGE_UNSHARE` first unshares the calling thread's file-descriptor table (so a subsequent close-range affects only this thread, not its sharing siblings), and `CLOSE_RANGE_CLOEXEC` sets FD_CLOEXEC on the range instead of closing — useful when followed by an `execve` that should keep its current standard fds but drop everything else.

close_range is the kernel-side substrate for `posix_spawn`-style file-action minimization, container runtimes' "scrub inherited fds" prelude, and security-tightening pre-`execve` cleanup. Critical for: systemd's `mount_setattr`/`open_tree` plumbing, podman/runc launchers, sandboxes like firejail, and any code that wants race-free fd-table hygiene.

### Acceptance Criteria

- [ ] AC-1: close_range(3, ~0U, 0) closes all fds ≥ 3 in this process.
- [ ] AC-2: close_range(5, 10, 0) closes fds 5..10 (sparse closes silently skipped).
- [ ] AC-3: close_range(10, 5, 0) returns -EINVAL.
- [ ] AC-4: close_range(3, ~0U, CLOSE_RANGE_CLOEXEC): fds remain open, FD_CLOEXEC set.
- [ ] AC-5: After CLOSE_RANGE_UNSHARE then close in one thread of a CLONE_FILES pair: the sibling thread still has the fds open.
- [ ] AC-6: Unknown flag bit returns -EINVAL.
- [ ] AC-7: ENOMEM on UNSHARE when dup_fd allocation fails.
- [ ] AC-8: close_range across a 1M-fd table completes in a single syscall (no userspace loop).
- [ ] AC-9: close_range does not extend the fdtable past current capacity.
- [ ] AC-10: close_range from inside a chroot only affects own thread's fds.

### Architecture

```rust
#[syscall(nr = 436, abi = "sysv")]
pub fn sys_close_range(first: u32, last: u32, flags: u32) -> isize {
    CloseRange::do_close_range(first, last, flags)
}
```

`CloseRange::do_close_range(first, last, flags) -> isize`:
1. if first > last { return Err(EINVAL); }
2. if flags & !(CLOSE_RANGE_UNSHARE | CLOSE_RANGE_CLOEXEC) != 0 { return Err(EINVAL); }
3. let cur = current().files();
4. let files = if flags & CLOSE_RANGE_UNSHARE != 0 {
5.     CloseRange::unshare_fd(&cur, last)?
6. } else {
7.     cur
8. };
9. let cap = files.lock().max_fds();
10. let last_real = min(last, cap.saturating_sub(1));
11. if flags & CLOSE_RANGE_CLOEXEC != 0 {
12.     CloseRange::set_close_on_exec_range(&files, first, last_real);
13. } else {
14.     CloseRange::close_fd_range(&files, first, last_real);
15. }
16. Ok(0)

`CloseRange::unshare_fd(files, max_fd) -> Result<Arc<Files>>`:
1. if Arc::strong_count(files) == 1 { return Ok(files.clone()); }
2. let new = Files::dup_with_max(files, max_fd)?;       // ENOMEM
3. current().set_files(new.clone());
4. Ok(new)

`CloseRange::close_fd_range(files, first, last)`:
1. let mut fd = first;
2. while fd <= last {
3.     let filp = {
4.         let mut g = files.lock();
5.         let f = g.fdt.fd[fd as usize].take();
6.         if !f.is_null() { g.fdt.clear_close_on_exec(fd); g.fdt.clear_open(fd); }
7.         f
8.     };
9.     if !filp.is_null() { Files::filp_close(filp); }   // may sleep
10.    fd += 1;
11. }

`CloseRange::set_close_on_exec_range(files, first, last)`:
1. let mut g = files.lock();
2. for fd in first..=last {
3.     if g.fdt.is_open(fd) { g.fdt.set_close_on_exec(fd); }
4. }

### Out of Scope

- close(2) (single-fd close).
- fork(2) / clone(2) CLONE_FILES (covered in Tier-5 fork/clone docs).
- execve(2) FD_CLOEXEC handling (covered in Tier-5 `execve.md`).
- Implementation code.

### signature

```c
int close_range(unsigned int first, unsigned int last, unsigned int flags);
```

```c
#define CLOSE_RANGE_UNSHARE  (1u << 1)  /* unshare files table first */
#define CLOSE_RANGE_CLOEXEC  (1u << 2)  /* set FD_CLOEXEC instead of close */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `first` | `unsigned int` | in | Lowest fd in the inclusive range. |
| `last`  | `unsigned int` | in | Highest fd in the inclusive range; `~0U` means "to the end of the table". |
| `flags` | `unsigned int` | in | Bitmask of CLOSE_RANGE_* flags. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; range processed (closed or marked cloexec). |
| `-1` + `errno` | Failure (no fd in range was touched, except on partial-failure of CLOEXEC path). |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `first > last`; unknown bits in `flags`; CLOSE_RANGE_UNSHARE | CLOSE_RANGE_CLOEXEC combined with conflicting state. |
| `ENOMEM` | CLOSE_RANGE_UNSHARE: unshare_fd allocation failed. |

Note: there is no `EBADF`. Closed and never-open fds inside the range are silently skipped — close_range is best-effort over the range.

### abi surface

```text
__NR_close_range  (x86_64)  = 436
__NR_close_range  (arm64)   = 436
__NR_close_range  (riscv)   = 436
__NR_close_range  (i386)    = 436

/* Range is INCLUSIVE.  last = ~0U means "to top of fd table". */
/* CLOSE_RANGE_CLOEXEC: mark FD_CLOEXEC, do NOT close.            */
/* CLOSE_RANGE_UNSHARE: clone-files first, then operate on clone. */
```

### compatibility contract

REQ-1: Syscall number is **436** on all architectures. Introduced in 5.9 (Sep 2020); ABI-stable since.

REQ-2: `first > last` → `-EINVAL`.

REQ-3: `flags` must be a subset of `CLOSE_RANGE_UNSHARE | CLOSE_RANGE_CLOEXEC`; unknown bits → `-EINVAL`.

REQ-4: `CLOSE_RANGE_UNSHARE` semantics:
- If `current.files->count > 1` (shared with sibling threads via CLONE_FILES): allocate a copy via `dup_fd(current.files, last)`, install as `current.files`. Subsequent operations affect only the copy.
- If not shared: no-op (already private).
- Implementation MUST avoid allocating an fd-table larger than necessary; `unshare_fd(last)` is told the high-water mark so the new table is sized to `min(NR_OPEN_DEFAULT, last+1)`.

REQ-5: `CLOSE_RANGE_CLOEXEC` semantics:
- For each fd in `[first, min(last, max_fd)]`:
  - If fd is open: set FD_CLOEXEC.
  - Else: skip.
- The file descriptors remain open and usable until `execve(2)`; after exec, kernel closes them.

REQ-6: Default semantics (no CLOSE_RANGE_CLOEXEC):
- For each fd in `[first, min(last, max_fd)]`:
  - If fd is open: call `filp_close(fdt->fd[fd], files)`, clear the slot.
  - Else: skip.
- Errors from per-fd `filp_close` (e.g. SIGPIPE on flush) are swallowed (POSIX precedent: `close(2)` errors are advisory).

REQ-7: Atomicity: under `current.files->file_lock` (spinlock); during the close-loop the lock is dropped per-iteration to allow `filp_close` to sleep (e.g. NFS close-on-last-fd). New fd allocations in this thread cannot land in already-processed slots because the loop walks forward.

REQ-8: `last == ~0U`: clamped to `current.files->next_fd_below_table_capacity()`. The walk does not extend the fdtable.

REQ-9: After CLOSE_RANGE_UNSHARE, the calling thread no longer shares its file table with sibling CLONE_FILES threads. This is the only way to safely "scrub fds in just my thread" inside a multi-threaded process.

REQ-10: Calling close_range with flags=0 and first=3, last=~0U is the canonical "post-fork pre-exec cleanup".

REQ-11: Per-namespace: no userns-specific behavior. Per-pidns: no behavior change. Per-fdtable: strictly per-thread (after CLOSE_RANGE_UNSHARE).

REQ-12: close_range(2) does not consume entropy and does not require any capability.

REQ-13: ENOENT-style "fd not open" is never reported; the range is silently sparse.

REQ-14: For chroot containers, close_range is observed by `chroot`-aware audit subsystems but is not itself a chroot-escape gadget — closing fds cannot recreate them.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `range_inclusive` | INVARIANT | first ≤ last enforced before any close. |
| `flags_subset` | INVARIANT | flags ⊆ {UNSHARE, CLOEXEC}. |
| `unshare_idempotent` | INVARIANT | strong_count==1 ⟹ no copy. |
| `no_table_extend` | INVARIANT | last clamped to current capacity. |
| `cloexec_no_close` | INVARIANT | CLOEXEC path never calls filp_close. |
| `forward_walk` | INVARIANT | iteration order is ascending; lock dropped per iter. |

### Layer 2: TLA+

`fs/close-range-syscall.tla`:
- States: per-validate, per-unshare, per-walk, per-close, per-cloexec.
- Properties:
  - `safety_unshare_visibility` — after UNSHARE, sibling threads still hold pre-call fds.
  - `safety_no_double_close` — per-fd close at most once per call.
  - `safety_cloexec_no_close` — CLOEXEC flag set without filp_close.
  - `safety_walks_within_capacity` — fd ≤ max_fds throughout.
  - `liveness_terminates` — walk always reaches last.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_close_range` post: success ⟹ all open fds in range processed | `CloseRange::do_close_range` |
| `unshare_fd` post: ret has unique Arc reference (count == 1) | `CloseRange::unshare_fd` |
| `close_fd_range` post: every fd in range is unset in fdt | `CloseRange::close_fd_range` |
| `set_close_on_exec_range` post: every open fd in range has FD_CLOEXEC | `CloseRange::set_close_on_exec_range` |

### Layer 4: Verus / Creusot functional

Per-close_range(2) Linux 5.9 semantic equivalence; LTP `close_range01..close_range04`, posix_spawn selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`close_range(2)` reinforcement:

- **Per-range validation strict (first ≤ last)** — defense against per-reverse-range silent walk.
- **Per-flags subset strict** — defense against per-bit-smuggling.
- **Per-CLOSE_RANGE_UNSHARE atomic table-copy** — defense against per-sibling-thread fd-aliasing.
- **Per-last clamped to current capacity** — defense against per-table-extension exploitation.
- **Per-CLOEXEC no close** — defense against per-double-close ambiguity.
- **Per-forward walk** — defense against per-loop iterator backflow.
- **Per-error-swallowed filp_close** — defense against per-EBADF-style ambiguity.

### grsecurity / pax surface

- **CLOSE_RANGE_UNSHARE chroot-escape defense** — close_range with UNSHARE inside chroot cannot recreate fds; grsec audits that the unshared fdtable cannot reach files above the chroot via `/proc/self/fd` ancestor traversal. The unshared table is per-thread, so leaking the unshared copy to a sibling outside chroot requires explicit pidfd / SCM_RIGHTS — already gated.
- **PaX UDEREF NOT applicable** — close_range takes only scalar args, no userptr.
- **GRKERNSEC_CHROOT_FCHDIR** — coordinates with close_range: chrooted callers can still close their own pre-chroot fds (which would otherwise be the chroot-escape gadget); explicit close_range cleanup is encouraged.
- **PAX_REFCOUNT on files->count** — defense against per-Arc<Files> refcount-overflow during UNSHARE.
- **PAX_REFCOUNT on filp refcount during filp_close** — defense against per-fput-double UAF.
- **RLIMIT_NOFILE strict for any subsequent open after close** — defense against per-fd-amplification after large-range close.
- **GRKERNSEC_AUDIT_GROUP=1** — audits close_range calls inside chroot to detect pre-execve cleanup patterns that might preceed exploit attempts.
- **No-cap-required honored** — close_range callable by any uid; grsec does not gate by cap (it is strictly a hygiene primitive).
- **GRKERNSEC_HIDESYM on close_fd vtable** — symbol not in kallsyms.
- **PAX_USERCOPY_HARDEN N/A** — no usercopy in close_range path.
- **Per-thread fdtable isolation** — defense against per-CLONE_FILES race: UNSHARE flag is the only safe way to scrub fds in a single thread of a CLONE_FILES process.

