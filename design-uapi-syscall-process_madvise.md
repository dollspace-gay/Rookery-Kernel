---
title: "Tier-5 syscall: process_madvise(2) — syscall 440"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`process_madvise(2)` lets a privileged caller apply `madvise(2)` operations to **another process's** address space, via that process's pidfd and an `iovec` array of (base, len) address ranges. Added in Linux 5.10, it is the foundation of remote memory-pressure management: a userspace memory-pressure daemon (e.g. Android's LMK Daemon, systemd-oomd) can call `process_madvise(pidfd, iov, MADV_PAGEOUT, ...)` to reclaim cold pages from a target process WITHOUT having to ptrace-attach. The advice values mirror `madvise(2)`: `MADV_COLD` (deactivate), `MADV_PAGEOUT` (reclaim immediately), `MADV_WILLNEED` (prefetch), `MADV_DONTNEED` (drop), `MADV_REMOVE`, etc., though only a subset are permitted cross-process.

Critical for: Android LMKD memory-pressure daemon, systemd-oomd userspace OOM killer, container memory managers (cgroupv2 + process_madvise reclaim hints), JVM heap-management agents, ML-training memory-pressure response (PyTorch GPU OOM-prevention via host madvise), every "reclaim pages from THAT process" pattern.

### Acceptance Criteria

- [ ] AC-1: `process_madvise(pidfd, iov, 1, MADV_COLD, 0)` returns iov[0].iov_len on success.
- [ ] AC-2: `process_madvise(pidfd, iov, 0, MADV_COLD, 0)` returns 0 (no-op).
- [ ] AC-3: `vlen = 2000` returns `EINVAL`.
- [ ] AC-4: `advice = MADV_HWPOISON` returns `EINVAL` (forbidden cross-process).
- [ ] AC-5: `flags = 1` returns `EINVAL`.
- [ ] AC-6: Cross-uid caller without CAP_SYS_PTRACE: `EPERM` for MADV_DONTNEED; possibly `EPERM` for MADV_PAGEOUT.
- [ ] AC-7: After target exits, returns `ESRCH`.
- [ ] AC-8: Partial success: 2-iovec call where iov[1] hits a bad range returns iov[0].iov_len.
- [ ] AC-9: Yama mode 3: returns `EPERM` regardless of caller's CAP.
- [ ] AC-10: `MADV_PAGEOUT` measurably reduces target's RSS (via /proc/<pid>/status:VmRSS).
- [ ] AC-11: `iovec = bad-pointer` returns `EFAULT`.
- [ ] AC-12: `iov[0].iov_base = bad-target-addr` returns 0 (or partial bytes from prior iovs if any).
- [ ] AC-13: LSM denies: returns `EPERM`.

### Architecture

```rust
#[syscall(nr = 440, abi = "sysv")]
pub fn sys_process_madvise(
    pidfd: i32,
    iovec: UserPtr<Iovec>,
    vlen: usize,
    advice: i32,
    flags: u32,
) -> KResult<isize> {
    ProcessMadvise::do_process_madvise(pidfd, iovec, vlen, advice, flags)
}
```

`ProcessMadvise::do_process_madvise(pidfd, iovec_uptr, vlen, advice, flags) -> KResult<isize>`:
1. /* Validate flags */
2. if flags != 0 { return Err(EINVAL); }
3. /* Validate vlen */
4. if vlen > UIO_MAXIOV { return Err(EINVAL); }
5. if vlen == 0 { return Ok(0); }
6. /* Validate advice */
7. let mode = match advice {
   - MADV_COLD | MADV_PAGEOUT | MADV_WILLNEED => PTRACE_MODE_READ_REALCREDS,
   - MADV_DONTNEED | MADV_REMOVE | MADV_COLLAPSE => PTRACE_MODE_ATTACH_REALCREDS,
   - _ => return Err(EINVAL),
8. };
9. /* Resolve pidfd → target task */
10. let pidfd_file = fdget(pidfd)?;
11. let target_pid = PidfdOpen::pid_from_file(&pidfd_file).ok_or(EBADF)?;
12. let target_task = target_pid.task_in(PIDTYPE_TGID).ok_or(ESRCH)?;
13. /* Permission check */
14. ptrace_may_access(target_task, mode)?;
15. /* LSM */
16. security_ptrace_access_check(target_task, mode)?;
17. /* Import iovec (UDEREF inside import_iovec) */
18. let mut fast_iov = [Iovec::default(); UIO_FASTIOV];
19. let iov = import_iovec(WRITE, iovec_uptr, vlen, &mut fast_iov)?;
20. /* Acquire target mm */
21. let mm = target_task.mm.get().ok_or(ESRCH)?;
22. /* Walk each iovec entry */
23. let mut total: isize = 0;
24. for entry in iov.iter() {
    - if entry.iov_len == 0 { continue; }
    - let res = ProcessMadvise::do_madvise_on_mm(mm, entry.iov_base, entry.iov_len, advice);
    - match res {
      - Ok(n) => total += n as isize,
      - Err(e) if total > 0 => break,   /* partial success */
      - Err(e) => return Err(e),
    - }
25. }
26. drop(iov);
27. Ok(total)

`ProcessMadvise::do_madvise_on_mm(mm, start, len, advice) -> KResult<usize>`:
1. let lock_mode = if advice_is_destructive(advice) { LockMode::Write } else { LockMode::Read };
2. let _lock = mm.mmap_lock.acquire(lock_mode);
3. let mut applied: usize = 0;
4. madvise_walk_vmas(mm, start, len, advice, |vma, vstart, vend| {
   - let res = match advice {
     - MADV_COLD => madvise_cold(vma, vstart, vend),
     - MADV_PAGEOUT => madvise_pageout(vma, vstart, vend),
     - MADV_WILLNEED => madvise_willneed(vma, vstart, vend),
     - MADV_DONTNEED => madvise_dontneed(vma, vstart, vend),
     - MADV_REMOVE => madvise_remove(vma, vstart, vend),
     - MADV_COLLAPSE => madvise_collapse(vma, vstart, vend),
     - _ => return Err(EINVAL),
   - };
   - if let Ok(()) = res { applied += (vend - vstart); }
   - res
5. })?;
6. Ok(applied)

### Out of Scope

- `madvise(2)` (Tier-5 separate doc — self-target madvise).
- `pidfd_open(2)` (Tier-5 separate doc).
- VMA walk implementation (Tier-3 in `mm/madvise.md`).
- `MADV_PAGEOUT` reclaim path (Tier-3 in `mm/vmscan.md`).
- Yama LSM (Tier-3 in `security/yama.md`).
- Implementation code.

### signature

```c
ssize_t process_madvise(int pidfd, const struct iovec *iovec,
                        size_t vlen, int advice, unsigned int flags);
```

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `pidfd` | `int` | in | pidfd to the target process. |
| `iovec` | `const struct iovec __user *` | in | Array of `(base, len)` pairs in target's address space. |
| `vlen` | `size_t` | in | Number of iovec entries; ≤ `UIO_MAXIOV` (1024). |
| `advice` | `int` | in | One of `MADV_COLD`, `MADV_PAGEOUT`, `MADV_WILLNEED`, `MADV_DONTNEED`, `MADV_COLLAPSE`, etc. |
| `flags` | `unsigned int` | in | Reserved — MUST be `0`. |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | Total number of BYTES the kernel applied advice to (may be < sum(iov.len) on partial success). |
| `-1` | Error; `errno` set. |

### errors

| `errno` | Cause |
|---|---|
| `EBADF` | `pidfd` is not a valid pidfd. |
| `EINVAL` | `vlen > UIO_MAXIOV`; `advice` not supported cross-process; `flags != 0`; iovec entry with overflow. |
| `EPERM` | Caller lacks `PTRACE_MODE_READ_REALCREDS` on target (advice was reading-only) OR `PTRACE_MODE_ATTACH_REALCREDS` (advice was destructive). |
| `ESRCH` | Target has exited. |
| `EFAULT` | iovec buffer or entry's `base` is invalid in TARGET's address space (per-range; may partial-succeed). |
| `ENOMEM` | Kernel could not allocate temporary iovec buffer. |
| `EAGAIN` | Target's mmap_sem held contended; retry. |

### abi surface

```text
__NR_process_madvise  (x86_64) = 440
__NR_process_madvise  (i386)   = 440
__NR_process_madvise  (arm64)  = 440
__NR_process_madvise  (generic)= 440

#define UIO_MAXIOV           1024
#define UIO_FASTIOV          8         /* on-stack fast-path */

/* iovec — see uio.h */
struct iovec {
    void  *iov_base;     /* target address-space pointer */
    size_t iov_len;
};

/* Permitted advice values (subset of madvise's full set): */
MADV_COLD          20    /* deactivate to inactive LRU */
MADV_PAGEOUT       21    /* reclaim now */
MADV_WILLNEED      3     /* readahead/prefault */
MADV_DONTNEED      4     /* drop pages (destructive — requires ATTACH) */
MADV_REMOVE        9     /* punch hole (destructive — requires ATTACH) */
MADV_COLLAPSE      25    /* THP collapse (5.18+) */

/* Permission model: */
/* READ-ONLY advice (MADV_COLD, MADV_PAGEOUT, MADV_WILLNEED):  */
/*   require PTRACE_MODE_READ_REALCREDS (less strict — only read access). */
/* DESTRUCTIVE advice (MADV_DONTNEED, MADV_REMOVE):             */
/*   require PTRACE_MODE_ATTACH_REALCREDS (full ptrace access). */
```

### compatibility contract

REQ-1: Syscall number is **440** on all architectures (uniformly assigned, added in 5.10).

REQ-2: `pidfd` MUST be a valid pidfd (from `pidfd_open` or `clone3(CLONE_PIDFD)`). Any other fd ⇒ `EBADF`.

REQ-3: `iovec` array imported via `import_iovec` (handles compat-32 / native-64 transparently). Up to `UIO_FASTIOV` (8) entries on-stack; larger requires kmalloc.

REQ-4: `vlen` MUST be in `[0, UIO_MAXIOV]` (`1024`). `vlen == 0`: returns 0 (no-op). `vlen > UIO_MAXIOV` ⇒ `EINVAL`.

REQ-5: Each iovec entry's `[iov_base, iov_base + iov_len)` MUST be valid in TARGET's address space. `iov_len == 0`: skipped. Overflow `iov_base + iov_len > target_addr_space_max` ⇒ stop and return partial bytes-applied.

REQ-6: `advice` MUST be one of the cross-process-permitted values:
- READ-only: `MADV_COLD`, `MADV_PAGEOUT`, `MADV_WILLNEED` — non-destructive, only reclaim hints.
- DESTRUCTIVE: `MADV_DONTNEED`, `MADV_REMOVE`, `MADV_COLLAPSE` (5.18+).
- Forbidden cross-process: `MADV_DONTFORK`, `MADV_DOFORK`, `MADV_HWPOISON`, `MADV_MERGEABLE`, etc. ⇒ `EINVAL`.

REQ-7: `flags` MUST be `0`. Any other value ⇒ `EINVAL`.

REQ-8: Permission: per-advice gate:
- READ-only: `PTRACE_MODE_READ_REALCREDS` (caller can read target).
- DESTRUCTIVE: `PTRACE_MODE_ATTACH_REALCREDS` (caller can attach to target).
- Yama LSM mode further restricts.

REQ-9: LSM hook `security_ptrace_access_check(target, mode)` fires before per-iovec walk; SELinux/AppArmor MAY deny.

REQ-10: Per-iovec walk: kernel locks target's `mmap_sem`/`mmap_lock` in appropriate mode (read for COLD/PAGEOUT/WILLNEED; write for DONTNEED/REMOVE), walks VMAs intersecting the range, applies advice. Walks are best-effort — partial completion is allowed.

REQ-11: Return value: total BYTES successfully advised across all iovec entries. May be < sum(iov.len) if a range hit `EFAULT`, hit a VMA boundary disallowing the operation, or target's mm tore down mid-call. NEVER negative on partial success — `EFAULT` only if the FIRST entry fails before any progress.

REQ-12: PaX UDEREF on the iovec buffer pre-import; per-entry `iov_base` is a TARGET-space pointer (no UDEREF check against target's mm by this syscall — that's enforced by the VMA walk in target's mm).

REQ-13: PID-namespace: target's struct pid is ns-agnostic. Permission check uses caller's creds vs target's creds.

REQ-14: `seccomp` filters see all 5 args; SHOULD be allowed for OOMD daemons with appropriate capability.

REQ-15: Memory accounting: temporary iovec buffer charged to caller's memcg if `vlen > UIO_FASTIOV`. The advised pages remain in the TARGET's memcg.

REQ-16: Concurrent `process_madvise` from multiple callers: each acquires target's `mmap_lock` in serialized order; no corruption but may queue.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vlen_in_range` | INVARIANT | per-syscall: 0 ≤ vlen ≤ UIO_MAXIOV. |
| `flags_zero` | INVARIANT | per-syscall: flags == 0 mandatory. |
| `advice_in_whitelist` | INVARIANT | per-syscall: advice ∈ permitted cross-process set. |
| `ptrace_mode_per_advice` | INVARIANT | per-advice: READ vs ATTACH mode correctly chosen. |
| `lsm_hook_fires` | INVARIANT | per-syscall: security_ptrace_access_check invoked. |
| `iovec_uderef_pre_import` | INVARIANT | per-import: UDEREF before copy_from_user iov entries. |
| `mmap_lock_correct_mode` | INVARIANT | per-walk: read-lock for non-destructive, write-lock for destructive. |
| `partial_progress_returned` | INVARIANT | per-error-after-progress: total > 0 returned. |

### Layer 2: TLA+

`mm/process-madvise.tla`:
- States: per-target mm VMA tree, per-iovec range, per-call progress counter.
- Properties:
  - `safety_no_unauthorized_advise` — per-call: ptrace mode passes.
  - `safety_target_exit_safe` — per-target-exit: ESRCH not UAF.
  - `safety_partial_success_monotonic` — per-iter: total only grows.
  - `liveness_eventually_returns` — per-call: terminates with isize.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_process_madvise` post: ret ≥ 0 on partial, errno on first-iov fail | `ProcessMadvise::do_process_madvise` |
| `do_madvise_on_mm` post: lock acquired in correct mode | `ProcessMadvise::do_madvise_on_mm` |
| `import_iovec` post: at most UIO_MAXIOV entries imported | `import_iovec` |

### Layer 4: Verus / Creusot functional

Per-`process_madvise(2)` man-page equivalence. LTP `process_madvise01..04` pass. Round-trip: `process_madvise(self_pidfd, ...)` equivalent to `madvise(...)`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`process_madvise(2)` reinforcement:

- **Per-advice cross-process whitelist** — defense against per-host-state corruption (HWPOISON etc.).
- **Per-PTRACE_MODE gate (READ vs ATTACH)** — defense against per-cross-domain memory destroying via destructive advice.
- **Per-LSM ptrace_access_check** — defense against per-policy-bypass.
- **Per-Yama gate** — defense against per-broad-introspection.
- **Per-iovec partial-progress monotonic** — defense against per-mid-walk UAF reporting.
- **Per-mmap_lock-mode correctness** — defense against per-concurrent-mutation race.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `struct iovec *iovec`** — every copy_from_user of the iovec buffer MUST traverse UDEREF SMAP/SMEP gate; rejects kernel-half pointers before iovec import. UDEREF is checked BEFORE per-entry processing so that no mm walk occurs against an invalid iovec.
- **PaX UDEREF on iovec entries** — each iovec.iov_base is a TARGET-space pointer; not directly UDEREF-checked against caller, but the VMA walk inside target's mm enforces target-side validity (no kernel pointers can be passed as iov_base because target mm cannot resolve them).
- **PaX UDEREF on siginfo_t-adjacent** — not applicable (no siginfo here); the broader UDEREF infrastructure covers all syscall pointer args.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — `process_madvise` is treated as ptrace-equivalent under hardened mode; CAP_SYS_PTRACE is required for cross-uid access regardless of dumpable status. Upstream allows same-uid + dumpable; grsec tightens to same-uid + same-cgroup + dumpable.
- **PIDFD_SELF safe self-target** — `process_madvise(pidfd_open(PIDFD_SELF), iov, vlen, advice, 0)` is permitted unconditionally (equivalent to vectored `madvise`); grsec fast-paths self-target.
- **GRKERNSEC_PROC restrictions on pidfd_getfd** — analogous restriction here: if caller cannot see /proc/<pid>/maps for the target, `process_madvise` returns EPERM (consistent with /proc visibility). This prevents using process_madvise as a back-channel to enumerate target's address-space layout.
- **MADV_PAGEOUT rate-limit** — high-rate MADV_PAGEOUT against a single target is rate-limited per-uid in hardened mode (sysctl `kernel.process_madvise_pageout_rate`); prevents memory-pressure DoS attacks where an attacker repeatedly evicts another process's pages.
- **MADV_DONTNEED CAP_SYS_RAWIO** — destructive cross-process madvise (DONTNEED/REMOVE) requires CAP_SYS_RAWIO in grsec hardened mode (in addition to PTRACE_MODE_ATTACH). Upstream requires only ATTACH; grsec tightens because DONTNEED can drop dirty private pages, causing data loss in the target.
- **signalfd CLOEXEC mandatory** — not directly applicable; process_madvise does not allocate fds.
- **EPOLL fd-lifetime refcount strict** — pidfd refcount on the input pidfd is held for the duration of the call; concurrent close of pidfd does not race.
- **GRKERNSEC_AUDIT_GROUP** — process_madvise calls from a marked group are audited unconditionally; MADV_PAGEOUT/MADV_DONTNEED operations on cross-uid targets are flagged.
- **No_new_privs neutral** — NNP does not change process_madvise semantics; access is governed by creds + CAP.
- **Anti-fingerprint hardening** — return value is bytes-applied; no kernel-pointer or address-layout info leaks via return path.
- **Yama mode 2/3 strict** — Yama mode 3 makes all cross-process process_madvise return EPERM unconditionally; grsec recommends mode 3 for production servers and provides a sysctl override per-cgroup.
- **eventfd counter quota** — not applicable; process_madvise does not interact with eventfd.
- **Seccomp interaction** — seccomp filters MAY restrict `advice` to a whitelist; default-deny hardening recommends permitting only MADV_COLD/MADV_PAGEOUT for OOMD daemons.

