---
title: "Tier-5 syscall: kcmp(2) — syscall 312"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`kcmp(2)` provides a kernel-mediated, address-randomization-safe way to compare whether two processes share a given kernel resource: file descriptions, address space, file-table, fs root, signal handler table, IO context, SysV semaphore set, or a specific epoll-registered fd. It returns an ordering (`-1`, `0`, `+1`, with `0` meaning "equal/shared") without ever exposing kernel pointers to userspace. Used by CRIU to determine whether two processes inherited the same `struct file` (and therefore must be checkpointed as a shared resource), by debuggers detecting CLONE_FILES/CLONE_VM sharing, and by container tooling validating namespace inheritance. Critical for: CRIU dump correctness, sharing-graph reconstruction, KASLR-safe comparison primitive.

### Acceptance Criteria

- [ ] AC-1: Two processes sharing FD via fork-inheritance: KCMP_FILE returns 0.
- [ ] AC-2: Two unrelated processes with same path FD: KCMP_FILE returns 1 or 2 (not 0).
- [ ] AC-3: CLONE_VM siblings: KCMP_VM returns 0.
- [ ] AC-4: CLONE_FILES siblings: KCMP_FILES returns 0.
- [ ] AC-5: KCMP_EPOLL_TFD with non-registered target: returns 3.
- [ ] AC-6: type = 99 returns `-EINVAL`.
- [ ] AC-7: pid1 nonexistent: `-ESRCH`.
- [ ] AC-8: pid1 owned by another user, no CAP_SYS_PTRACE: `-EPERM`.
- [ ] AC-9: KCMP_FILE with bad idx1: `-EBADF`.
- [ ] AC-10: Returned permuted ordering does not leak raw kernel pointer bits (statistical test over many runs).
- [ ] AC-11: Audit record per call.

### Architecture

```rust
#[syscall(nr = 312, abi = "sysv")]
pub fn sys_kcmp(pid1: i32, pid2: i32, ty: i32, idx1: u64, idx2: u64) -> isize {
    Kcmp::do_kcmp(pid1, pid2, ty, idx1, idx2)
}
```

`Kcmp::do_kcmp(pid1, pid2, ty, idx1, idx2) -> isize`:
1. let kind = KcmpType::try_from(ty).map_err(|_| -EINVAL)?;
2. let t1 = Pid::lookup(pid1).ok_or(-ESRCH)?;
3. let t2 = Pid::lookup(pid2).ok_or(-ESRCH)?;
4. Kcmp::check_perm(&t1)?;     // EPERM
5. Kcmp::check_perm(&t2)?;
6. let result = match kind {
7.   KcmpType::File    => Kcmp::cmp_file(&t1, idx1, &t2, idx2)?,
8.   KcmpType::Vm      => Kcmp::cmp_ptr(t1.mm, t2.mm, KcmpType::Vm),
9.   KcmpType::Files   => Kcmp::cmp_ptr(t1.files, t2.files, KcmpType::Files),
10.  KcmpType::Fs      => Kcmp::cmp_ptr(t1.fs, t2.fs, KcmpType::Fs),
11.  KcmpType::Sighand => Kcmp::cmp_ptr(t1.sighand, t2.sighand, KcmpType::Sighand),
12.  KcmpType::Io      => Kcmp::cmp_ptr(t1.io_context, t2.io_context, KcmpType::Io),
13.  KcmpType::Sysvsem => Kcmp::cmp_ptr(t1.sysvsem.undo, t2.sysvsem.undo, KcmpType::Sysvsem),
14.  KcmpType::EpollTfd => Kcmp::cmp_epoll(&t1, idx1, &t2, idx2)?,
15. };
16. AuditLog::record_kcmp(t1.pid, t2.pid, kind);
17. result as isize

`Kcmp::check_perm(target) -> Result<()>`:
1. if !ptrace_may_access(target, PTRACE_MODE_READ_REALCREDS) { return Err(EPERM); }
2. Ok(())

`Kcmp::cmp_ptr<T>(a: *const T, b: *const T, ty: KcmpType) -> i32`:
1. if a == b { return 0; }
2. let ka = permute_kptr(a as usize, ty);
3. let kb = permute_kptr(b as usize, ty);
4. if ka < kb { 1 } else { 2 }

`Kcmp::cmp_file(t1, fd1, t2, fd2) -> Result<i32>`:
1. let f1 = FdTable::fget(t1, fd1 as i32).ok_or(-EBADF)?;
2. let f2 = FdTable::fget(t2, fd2 as i32).ok_or(-EBADF)?;
3. Ok(Kcmp::cmp_ptr(f1.as_ptr(), f2.as_ptr(), KcmpType::File))

### Out of Scope

- Per-CRIU dump semantics (out of kernel scope).
- Per-epoll registration model (covered in Tier-3 `fs/eventpoll.md`).
- Per-ptrace permission model (covered in Tier-3 `kernel/ptrace.md`).
- Implementation code.

### signature

```c
int kcmp(pid_t pid1, pid_t pid2,
         int type,
         unsigned long idx1, unsigned long idx2);
```

```c
enum kcmp_type {
    KCMP_FILE       = 0,   /* idx1, idx2 are file descriptors */
    KCMP_VM         = 1,   /* compare mm */
    KCMP_FILES      = 2,   /* compare files_struct */
    KCMP_FS         = 3,   /* compare fs_struct */
    KCMP_SIGHAND    = 4,   /* compare sighand_struct */
    KCMP_IO         = 5,   /* compare io_context */
    KCMP_SYSVSEM    = 6,   /* compare sysvsem undo list */
    KCMP_EPOLL_TFD  = 7,   /* idx1 = epfd; idx2 = ptr to kcmp_epoll_slot */
    KCMP_TYPES,
};

struct kcmp_epoll_slot {
    __u32 efd;     /* target fd registered in epoll */
    __u32 tfd;
    __u64 toff;    /* event slot offset */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid1`, `pid2` | `pid_t` | in | Process IDs (in caller's PID namespace). |
| `type` | `int` | in | One of `KCMP_*`; selects per-resource comparator. |
| `idx1`, `idx2` | `unsigned long` | in | Per-type indices (FDs for KCMP_FILE/KCMP_EPOLL_TFD, ignored otherwise). |

### return value

| Value | Meaning |
|---|---|
| `0` | Resources are identical (same kernel object). |
| `1` | Resource of pid1 sorts before pid2 (obfuscated pointer hash). |
| `2` | Resource of pid1 sorts after pid2. |
| `3` | Resources are not comparable (e.g., epoll target not found). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `type >= KCMP_TYPES`, invalid combination. |
| `EFAULT` | KCMP_EPOLL_TFD: idx2 user-pointer faults during `copy_from_user`. |
| `ESRCH` | `pid1` or `pid2` does not exist. |
| `EBADF` | KCMP_FILE/KCMP_EPOLL_TFD: idx1/idx2 not valid FD in respective process. |
| `EPERM` | Caller cannot ptrace-attach to one or both targets. |
| `ENOENT` | KCMP_EPOLL_TFD: target tfd not registered in epoll set. |

### abi surface

```text
__NR_kcmp (x86_64) = 312
__NR_kcmp (arm64)  = 272
__NR_kcmp (riscv)  = 272
__NR_kcmp (i386)   = 349

/* Return values 0/1/2/3, not classic 0 + errno on success. */
/* Kernel obfuscates pointers via per-boot random key + permute_ptr(). */
```

### compatibility contract

REQ-1: Syscall number is **312** on x86_64. ABI-stable.

REQ-2: `type` MUST be in `[0, KCMP_TYPES)`; otherwise `-EINVAL`.

REQ-3: Permission check: caller MUST be able to `ptrace_may_access(target, PTRACE_MODE_READ_REALCREDS)` for BOTH `pid1` and `pid2`. If either denies → `-EPERM`.

REQ-4: Per-`KCMP_FILE` (type 0):
- idx1 is an FD in pid1's fdtable; idx2 is an FD in pid2's fdtable.
- Compare `struct file *` pointers obtained from each fdtable.
- Returns 0 if same file description (e.g., dup'd, inherited).

REQ-5: Per-`KCMP_VM` (type 1):
- Compare `task->mm` pointers; 0 iff same mm (CLONE_VM siblings).

REQ-6: Per-`KCMP_FILES` (type 2):
- Compare `task->files` pointers; 0 iff CLONE_FILES sharing.

REQ-7: Per-`KCMP_FS` (type 3):
- Compare `task->fs` pointers; 0 iff CLONE_FS sharing (root, cwd, umask).

REQ-8: Per-`KCMP_SIGHAND` (type 4):
- Compare `task->sighand` pointers; 0 iff CLONE_SIGHAND sharing.

REQ-9: Per-`KCMP_IO` (type 5):
- Compare `task->io_context` pointers; 0 iff CLONE_IO sharing.

REQ-10: Per-`KCMP_SYSVSEM` (type 6):
- Compare `task->sysvsem.undo_list` pointers; 0 iff sharing SysV-undo state.

REQ-11: Per-`KCMP_EPOLL_TFD` (type 7):
- idx1 is the epfd in pid1; idx2 is a userspace pointer to `struct kcmp_epoll_slot`.
- `copy_from_user(&slot, idx2, sizeof(slot))`; EFAULT on fault.
- Look up `epitem` matching (slot.efd, slot.tfd, slot.toff) in epoll set.
- If not found: returns 3 (not comparable).
- Else: compare `epitem->ffd.file` pointer with pid2's `slot.efd` resolved file.

REQ-12: Pointer-comparison obfuscation: `kcmp_ptr(v1, v2, type)` returns:
- 0 if v1 == v2.
- 1 if permute_kptr(v1, type) < permute_kptr(v2, type), else 2.
- `permute_kptr` is per-boot xor + bitwise scramble; NEVER returns raw pointer bits.

REQ-13: Per-PID-namespace: `pid1`/`pid2` resolved in caller's PID namespace; not-visible target → `-ESRCH`.

REQ-14: Per-audit: every call audit-logged with both target pids, type, caller real-uid.

REQ-15: Per-`CONFIG_CHECKPOINT_RESTORE`: kcmp is gated; compiled out only when CRIU support is disabled. ABI present unconditionally in default builds.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `type_in_range` | INVARIANT | type >= KCMP_TYPES ⟹ EINVAL. |
| `permission_required` | INVARIANT | ptrace_may_access denied ⟹ EPERM. |
| `cmp_ptr_no_raw_leak` | INVARIANT | output ∈ {0,1,2}; never raw pointer bits. |
| `cmp_ptr_total_order` | INVARIANT | (a==b) ⟹ 0; else (1 XOR 2). |
| `epoll_slot_copy_bounded` | INVARIANT | KCMP_EPOLL_TFD copy_from_user bounded sizeof(slot). |

### Layer 2: TLA+

`kernel/kcmp.tla`:
- States: per-type-dispatch, per-permission, per-resource-fetch, per-permute, per-return.
- Properties:
  - `safety_no_compare_without_permission` — never compute without ptrace_may_access.
  - `safety_no_pointer_leak` — output never reveals raw pointer.
  - `safety_total_order` — comparison is reflexive (same ptr ⟹ 0) and total (different ptr ⟹ 1 XOR 2).
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_kcmp` post: type valid ⟹ return ∈ {0,1,2,3} | `Kcmp::do_kcmp` |
| `cmp_ptr` post: a==b ⟺ 0; otherwise 1 or 2 | `Kcmp::cmp_ptr` |
| `check_perm` post: ok ⟹ ptrace_may_access true | `Kcmp::check_perm` |

### Layer 4: Verus / Creusot functional

Per-`kcmp(2)` man-page semantic equivalence. CRIU regression tests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`kcmp(2)` reinforcement:

- **Per-type bound check** — defense against per-table OOB dispatch.
- **Per-ptrace permission both pids** — defense against per-cross-process info disclosure.
- **Per-permute_kptr per-boot key** — defense against per-KASLR-defeat via pointer ordering.
- **Per-epoll-slot copy_from_user bounded** — defense against per-attr-smuggling.
- **Per-audit record per-call** — defense against per-silent sharing-graph reconnaissance.
- **Per-CAP_SYS_PTRACE strict in target userns** — defense against per-userns priv escape.

### grsecurity / pax surface

- **PaX UDEREF on KCMP_EPOLL_TFD `idx2` copy_from_user** — defense against per-slot-pointer kernel deref; SMAP forced.
- **GRKERNSEC_KCMP_ASLR_DEFEAT_MITIGATION** — `permute_kptr` uses a per-boot 64-bit random secret XOR'd with pointer, then SipHash-finalize, then returned modulo 2-element domain (1/2). This prevents an attacker from leaking real pointer bits via repeated kcmp calls over many resource pairs; the secret is regenerated on `setuid` boundary as defense-in-depth.
- **GRKERNSEC_HARDEN_PTRACE strict** — `ptrace_may_access` check required for BOTH targets in target userns; sibling-userns CAP_SYS_PTRACE refused; root-in-non-init-userns refused.
- **GRKERNSEC_PROC_GETPID consistency** — invisible target pid returns `-ESRCH` (not EPERM) to avoid existence leak.
- **CAP_SYS_PTRACE in target ns** strict — defense against per-cross-ns ptrace leak.
- **PaX KERNEXEC on per-resource fetch path** — defense against per-W^X violation in kernel object resolution.
- **PAX_REFCOUNT on per-resource refcount take/drop** — defense against per-refcount overflow UAF during long-running multi-pair kcmp loop.
- **GRKERNSEC_AUDIT_KCMP** — kernel.audit emits ANOM_KCMP record per call with caller real-uid, both target pids, type.
- **GRKERNSEC_DENY_KCMP toggle** — restricts kcmp to `CAP_CHECKPOINT_RESTORE` in init_userns (recommended on hardened hosts without CRIU).
- **PAX_USERCOPY_HARDEN on epoll slot copy** — bounded into whitelisted slab.
- **GRKERNSEC_LOG_RATE_LIMIT on EPERM** — defense against per-log-flood by repeated denied kcmp probing.

