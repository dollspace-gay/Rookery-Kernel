# Tier-5 syscall: pidfd_getfd(2) — syscall 438

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/pid.c (SYSCALL_DEFINE3(pidfd_getfd))
  - kernel/pid.c (pidfd_getfd, __pidfd_fget)
  - fs/file.c (fget_task, get_unused_fd_flags)
  - include/uapi/linux/pidfd.h
  - arch/x86/entry/syscalls/syscall_64.tbl (438 common pidfd_getfd)
-->

## Summary

`pidfd_getfd(2)` lets a privileged caller **steal/copy a file descriptor** from another process by referencing it via that process's pidfd and the target-side fd number. It returns a NEW fd in the caller's fd table that refers to the same underlying `struct file` as the target's `targetfd`. The two fds independently increment the file's refcount; closing one does not affect the other. Added in Linux 5.6, it is the userspace-visible equivalent of what `ptrace(PTRACE_GET_RSEQ_CONFIGURATION)`-style facilities did informally — but as an LSM-gated, CAP_SYS_PTRACE-gated, race-free API.

Use cases: container debuggers attaching to existing processes' open fds (sockets, file descriptors held by stuck processes); container init pulling file descriptors out of payload processes; CRIU-style checkpoint/restore (gathering all open fds from a target); supervisor stealing a stuck child's listening socket for graceful restart; sandbox observers introspecting fd-holdings.

Critical for: gdb-pidfd, lldb pidfd attach, criu (Checkpoint/Restore In Userspace), systemd-coredumpctl debug helper, debugger-as-a-service infrastructure, K8s sidecar inspectors, any "snatch the fd from a frozen process" pattern.

## Signature

```c
int pidfd_getfd(int pidfd, int targetfd, unsigned int flags);
```

## Parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `pidfd` | `int` | in | pidfd to the source process. |
| `targetfd` | `int` | in | fd number in the source process's fd table. |
| `flags` | `unsigned int` | in | Reserved — MUST be `0` in current upstream. |

## Return value

| Value | Meaning |
|---|---|
| ≥ 0 | New fd in caller's table referring to same `struct file` as target's `targetfd`. Has `FD_CLOEXEC` set by default. |
| `-1` | Error; `errno` set. |

## Errors

| `errno` | Cause |
|---|---|
| `EBADF` | `pidfd` is not a valid open fd, OR `pidfd` is not a pidfd, OR `targetfd` is not open in the target process. |
| `EINVAL` | `flags != 0`. |
| `EPERM` | Caller lacks `PTRACE_MODE_ATTACH_REALCREDS` on target (typically requires CAP_SYS_PTRACE OR same-uid+same-cgroup with no setuid). |
| `ESRCH` | Target process has exited (pidfd refers to dead process). |
| `EMFILE` | Caller's per-process fd limit reached. |
| `ENFILE` | System-wide fd table exhausted. |

## ABI surface

```text
__NR_pidfd_getfd  (x86_64) = 438
__NR_pidfd_getfd  (i386)   = 438
__NR_pidfd_getfd  (arm64)  = 438
__NR_pidfd_getfd  (generic)= 438

/* Flags: */
/* Currently must be 0. Reserved for future expansion (e.g., O_CLOEXEC override). */

/* The returned fd: */
/* - shares the underlying struct file (and thus file offset, F_SETFL flags, etc.) */
/*   with the target's targetfd */
/* - is independently ref-counted (close on either does not affect the other) */
/* - has FD_CLOEXEC set unconditionally (5.10+; mandatory) */
/* - file mode (read/write/append) is preserved from the target's fd */

/* Permission model: PTRACE_MODE_ATTACH_REALCREDS */
/* See ptrace(2): requires CAP_SYS_PTRACE, OR */
/*   - same-uid (real == real, effective == effective, saved == saved) */
/*   - target not dumpable=0 (set by setuid/setgid exec) */
/*   - target's namespaces are ancestors of caller's */
```

## Compatibility contract

REQ-1: Syscall number is **438** on all architectures (uniformly assigned, added in 5.6).

REQ-2: `pidfd` MUST be a valid pidfd (created by `pidfd_open` or `clone3(CLONE_PIDFD)`). Any other fd type ⇒ `EBADF`.

REQ-3: `targetfd` MUST be open in the target process's fd table at the time of the syscall. If not open ⇒ `EBADF`.

REQ-4: `flags` MUST be `0` in all upstream kernels through 7.0. Other values ⇒ `EINVAL`.

REQ-5: Permission: the caller MUST satisfy `PTRACE_MODE_ATTACH_REALCREDS` against the target:
- Same-uid (real, effective, saved all match between caller and target), OR
- `CAP_SYS_PTRACE` in the target's user namespace.
- The target MUST be `dumpable` (not previously elevated via setuid/setgid exec without explicit `prctl(PR_SET_DUMPABLE)`).
- Yama LSM (`/proc/sys/kernel/yama/ptrace_scope`) MAY add further restrictions (mode 1 = restricted; mode 2 = admin only; mode 3 = disabled entirely).

REQ-6: LSM hook `security_ptrace_access_check(target_task, PTRACE_MODE_ATTACH_REALCREDS)` fires before fd duplication; SELinux/AppArmor MAY deny.

REQ-7: Returned fd: NEW fd number in caller; refers to SAME `struct file` (refcount++). Sharing semantics:
- File position (offset): shared.
- File status flags (`O_NONBLOCK`, `O_APPEND`): shared.
- File access mode: shared (cannot be changed post-duplication).
- File-descriptor flags (FD_CLOEXEC): independent per fd.

REQ-8: Returned fd has `FD_CLOEXEC` set unconditionally. Userspace MAY clear it via `fcntl(F_SETFD)` after creation.

REQ-9: After return, closing the source process's `targetfd` does NOT close caller's new fd (independent refcount). Closing caller's new fd does NOT affect target.

REQ-10: If target has exited (zombie or beyond), pidfd remains valid but `pidfd_getfd` returns `ESRCH`. The target's fd table was torn down at exit.

REQ-11: PaX UDEREF: not applicable (no userspace pointer args).

REQ-12: `seccomp` filters see all 3 args; SHOULD be allowed for debugger sandboxes, MAY be blocked for tighter sandboxes.

REQ-13: PID-namespace: pidfd's underlying `struct pid` is ns-agnostic. The target's fd table is per-task; resolution does not require ns-mapping.

REQ-14: Atomicity: the underlying `fget_task` operation atomically increments the file refcount under the target's `files_struct` lock. The race "target closes targetfd while pidfd_getfd is in-flight" is resolved by lock-ordered fget: either the file ref is acquired (success) OR the close wins (EBADF).

REQ-15: Memory accounting: the new fd is charged to caller's per-process fd quota; the file itself is shared (no additional file allocation).

REQ-16: File-system / device / socket type is preserved. Stealing a UNIX socket fd from another process and then `recvmsg(2)`-ing on it is well-defined; the socket's queue state is shared because the file is shared.

## Acceptance Criteria

- [ ] AC-1: Caller with CAP_SYS_PTRACE: `pidfd_getfd(pidfd, target_fd, 0)` returns valid new fd.
- [ ] AC-2: Same-uid caller without CAP_SYS_PTRACE: succeeds if target is dumpable.
- [ ] AC-3: Cross-uid caller without CAP_SYS_PTRACE: returns `EPERM`.
- [ ] AC-4: `pidfd_getfd(non_pidfd, fd, 0)` returns `EBADF`.
- [ ] AC-5: `pidfd_getfd(pidfd, target_unopened_fd, 0)` returns `EBADF`.
- [ ] AC-6: `flags = 1` returns `EINVAL`.
- [ ] AC-7: After target exits, `pidfd_getfd` returns `ESRCH`.
- [ ] AC-8: Returned fd has `FD_CLOEXEC` set (verified via `fcntl(F_GETFD)`).
- [ ] AC-9: Closing the new fd does NOT close target's `targetfd`.
- [ ] AC-10: Target closing its `targetfd` does NOT close caller's new fd.
- [ ] AC-11: File offset is shared: `lseek` on either fd is observable on the other.
- [ ] AC-12: Yama mode 2 + non-CAP_SYS_PTRACE: returns `EPERM`.
- [ ] AC-13: Yama mode 3: ALL `pidfd_getfd` returns `EPERM` regardless of CAP.

## Architecture

```rust
#[syscall(nr = 438, abi = "sysv")]
pub fn sys_pidfd_getfd(
    pidfd: i32,
    targetfd: i32,
    flags: u32,
) -> KResult<i32> {
    PidfdGetfd::do_pidfd_getfd(pidfd, targetfd, flags)
}
```

`PidfdGetfd::do_pidfd_getfd(pidfd, targetfd, flags) -> KResult<i32>`:
1. /* Validate flags */
2. if flags != 0 { return Err(EINVAL); }
3. /* Resolve pidfd */
4. let pidfd_file = fdget(pidfd)?;
5. let target_pid: &Arc<Pid> = PidfdOpen::pid_from_file(&pidfd_file).ok_or(EBADF)?;
6. let target_task = target_pid.task_in(PIDTYPE_TGID).ok_or(ESRCH)?;
7. /* Permission check */
8. ptrace_may_access(target_task, PTRACE_MODE_ATTACH_REALCREDS)?;
9. /* LSM */
10. security_ptrace_access_check(target_task, PTRACE_MODE_ATTACH_REALCREDS)?;
11. /* Acquire target's struct file at targetfd */
12. let file = PidfdGetfd::fget_task(target_task, targetfd)?;
13. /* Allocate new fd in caller, with FD_CLOEXEC */
14. let new_fd = match get_unused_fd_flags(O_CLOEXEC) {
    - Ok(fd) => fd,
    - Err(e) => { put_file(file); return Err(e); }
15. };
16. fd_install(new_fd, file);
17. Ok(new_fd)

`PidfdGetfd::fget_task(task, fd) -> KResult<Arc<File>>`:
1. let _files_lock = task.files.lock_shared();
2. let fdt = task.files.fdtable_rcu();
3. if fd < 0 || (fd as usize) >= fdt.max_fds {
   - return Err(EBADF);
4. }
5. let file_ptr = fdt.fd[fd as usize].load(Ordering::Acquire);
6. let file = NonNull::new(file_ptr).ok_or(EBADF)?;
7. /* Refcount++ atomic */
8. let arc = Arc::from_raw_inc(file);
9. Ok(arc)

`ptrace_may_access(task, mode) -> KResult<()>`:
1. /* Yama LSM check */
2. if yama_mode >= 3 { return Err(EPERM); }
3. if yama_mode == 2 && !capable(CAP_SYS_ADMIN) { return Err(EPERM); }
4. if yama_mode == 1 && !is_yama_attacher_authorized(task) { return Err(EPERM); }
5. /* Standard ptrace check */
6. if same_creds_real(current(), task) && task.is_dumpable() { return Ok(()); }
7. if ns_capable(task.user_ns, CAP_SYS_PTRACE) { return Ok(()); }
8. Err(EPERM)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_zero` | INVARIANT | per-getfd: flags == 0 mandatory. |
| `ptrace_attach_check` | INVARIANT | per-getfd: PTRACE_MODE_ATTACH_REALCREDS enforced. |
| `lsm_hook_fires` | INVARIANT | per-getfd: security_ptrace_access_check invoked. |
| `file_refcount_atomic` | INVARIANT | per-fget_task: atomic refcount++. |
| `cloexec_mandatory` | INVARIANT | per-return: FD_CLOEXEC set on new fd. |
| `independent_close` | INVARIANT | per-dup: caller's fd and target's fd independent. |
| `rollback_on_alloc_failure` | INVARIANT | per-alloc-fail: file ref put back. |

### Layer 2: TLA+

`kernel/pidfd-getfd.tla`:
- States: per-task files_struct fdtable, per-file refcount, per-pidfd target pid.
- Properties:
  - `safety_no_unauthorized_steal` — per-getfd: ptrace-mode check passes.
  - `safety_file_ref_balanced` — per-success: refcount++ matched by close.
  - `safety_target_exit_safe` — per-target-exit: ESRCH not UAF.
  - `liveness_eventually_returns` — per-call: terminates with fd or errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pidfd_getfd` post: new fd in caller's fdtable | `PidfdGetfd::do_pidfd_getfd` |
| `fget_task` post: file refcount incremented under lock | `PidfdGetfd::fget_task` |
| `ptrace_may_access` post: returns Ok only if creds match or CAP_SYS_PTRACE | `ptrace_may_access` |

### Layer 4: Verus / Creusot functional

Per-`pidfd_getfd(2)` man-page equivalence. LTP `pidfd_getfd01..03` pass. Round-trip: pidfd_open → pidfd_getfd → operations on returned fd semantically equivalent to operations on target's original fd.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pidfd_getfd(2)` reinforcement:

- **Per-PTRACE_MODE_ATTACH_REALCREDS gate** — defense against per-cross-domain fd theft.
- **Per-Yama mode gate** — defense against per-broad-ptrace exposure.
- **Per-LSM ptrace_access_check hook** — defense against per-policy-bypass fd steal.
- **Per-file refcount atomic under task-files lock** — defense against per-race close-during-getfd.
- **Per-FD_CLOEXEC mandatory** — defense against per-stolen-fd-leak via exec.

## Grsecurity / PaX-style Reinforcement

- **GRKERNSEC_PROC restrictions on pidfd_getfd** — pidfd_getfd to a process the caller cannot read /proc/<pid>/fd/ is denied at the same restriction level. In hardened mode, the target's /proc visibility gates pidfd_getfd resolution: if caller cannot see /proc/<pid>/fd/<n>, then `pidfd_getfd(pidfd, n, 0)` returns EBADF (consistent with /proc visibility).
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — pidfd_getfd is treated as ptrace-equivalent under hardened mode; CAP_SYS_PTRACE is required for cross-uid access (upstream allows same-uid + dumpable). Same-uid still passes only if target's cgroup matches caller's cgroup ancestor.
- **PIDFD_SELF safe self-target** — `pidfd_getfd(pidfd_open(PIDFD_SELF), fd, 0)` is permitted unconditionally (self-dup, equivalent to `dup(fd) | F_DUPFD_CLOEXEC`); grsec fast-paths self-target.
- **PaX UDEREF on syscall args** — not directly applicable (no userspace pointer args).
- **signalfd CLOEXEC mandatory** — stolen signalfd retains CLOEXEC (the new fd's CLOEXEC bit is set unconditionally by pidfd_getfd).
- **EPOLL fd-lifetime refcount strict** — stealing an epoll fd via pidfd_getfd is supported but rare; grsec ensures the stolen epfd's epitems are not corruptible by the caller (epoll is per-file, so stolen epfd shares the RB-tree with the target).
- **eventfd counter quota** — stealing an eventfd does not bypass per-uid eventfd counter quota; the new fd is still counted against the caller's quota.
- **GRKERNSEC_AUDIT_GROUP** — pidfd_getfd calls from a marked group are audited unconditionally (high-value introspection capability).
- **No_new_privs neutral** — NNP does not change pidfd_getfd semantics; ptrace access is governed by creds + CAP.
- **Anti-fingerprint hardening** — caller's new fd number is independent of target's fd number; no information about target's fd table layout leaks beyond what /proc/<pid>/fd/ already exposes.
- **Yama mode 2/3 strict** — grsec recommends Yama mode 3 (no ptrace) for production servers; pidfd_getfd unconditionally returns EPERM in mode 3, eliminating fd-theft attack surface.
- **CAP_SYS_PTRACE in target's user namespace** — capability check is scoped to TARGET's user ns, not caller's. A root-in-container caller can pidfd_getfd a process in the SAME container but NOT a process in the host. Grsec enforces this even if caller has multiple ns-capabilities.
- **Seccomp interaction** — seccomp filters MAY block pidfd_getfd entirely for sandboxed services; default-deny hardening recommends forbidding it outside of explicit debugger/CRIU sandboxes.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `pidfd_open(2)` (Tier-5 separate doc).
- `ptrace(2)` (Tier-5 separate doc — ptrace API).
- Yama LSM (Tier-3 in `security/yama.md`).
- CRIU integration (Tier-3 in `kernel/checkpoint-restore.md`).
- `/proc/<pid>/fd/` resolution (Tier-3 in `fs/proc/fd.md`).
- Implementation code.
