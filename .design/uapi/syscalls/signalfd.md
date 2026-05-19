# Tier-5 syscall: signalfd(2) — syscall 282

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/signalfd.c (SYSCALL_DEFINE3(signalfd))
  - fs/signalfd.c (SYSCALL_DEFINE4(signalfd4))
  - include/linux/signalfd.h
  - include/uapi/linux/signalfd.h (struct signalfd_siginfo)
  - kernel/signal.c (dequeue_signal)
  - arch/x86/entry/syscalls/syscall_64.tbl (282 64 signalfd)
-->

## Summary

`signalfd(2)` is the **legacy** three-argument variant of the signalfd interface; it creates (or reconfigures) a file descriptor that delivers signals as readable events rather than via asynchronous signal handlers. The legacy variant lacks any `flags` argument, so the resulting fd has neither `O_NONBLOCK` nor `O_CLOEXEC` — a property exploited by `execve`-spanning sandbox escapes and a primary motivation for `signalfd4(2)` (syscall 289). All modern userspace SHOULD invoke `signalfd4` (often through the `signalfd(2)` glibc wrapper, which transparently forwards to syscall 289 with `flags = 0`). The kernel still exports the legacy entry for ABI stability.

Used to: integrate signal handling into `epoll`/`poll`/`select`-driven event loops; deterministically drain pending signals in a single thread; replace `sigwaitinfo(2)` with an fd-backed equivalent; deliver `SIGCHLD`/`SIGTERM`/`SIGINT` synchronously in supervisors and IDEs; participate in `pidfd`-based shutdown flows.

Critical for: pre-flags-aware userspace (very old glibc), legacy embedded systems, ABI-museum binary compatibility, kernel `signalfd_ctx` lifetime tests, the historical pre-`O_CLOEXEC` race demonstration. Modern code uses `signalfd4`.

## Signature

```c
int signalfd(int fd, const sigset_t *mask, size_t sizemask);
```

## Parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `fd` | `int` | in | `-1` to create a new signalfd; or an existing signalfd fd to update its mask. |
| `mask` | `const sigset_t __user *` | in | Set of signals to deliver via this fd. `SIGKILL`/`SIGSTOP` are silently excluded. |
| `sizemask` | `size_t` | in | `sizeof(sigset_t)`. MUST equal kernel `_NSIG / 8` (8 bytes on 64-bit; 16 on 32-bit-x86 with 128-bit sigset). |

## Return value

| Value | Meaning |
|---|---|
| ≥ 0 | New signalfd file descriptor (when `fd == -1`), OR the same fd echoed back (when reconfiguring existing). |
| `-1` | Error; `errno` set. |

## Errors

| `errno` | Cause |
|---|---|
| `EBADF` | `fd` is not `-1` and not a valid open file descriptor. |
| `EINVAL` | `fd` is a valid open fd but NOT a signalfd; OR `sizemask != sizeof(sigset_t)`. |
| `ENFILE` | System-wide fd limit reached. |
| `EMFILE` | Per-process fd limit reached. |
| `ENOMEM` | Kernel could not allocate `signalfd_ctx`. |
| `EFAULT` | `mask` is not in caller's readable address space. |

## ABI surface

```text
__NR_signalfd  (x86_64) = 282
__NR_signalfd  (i386)   = 321
__NR_signalfd  (arm64)  = NOT IMPLEMENTED   (arm64 only provides signalfd4)
__NR_signalfd  (generic)= NOT IMPLEMENTED

/* Returned fd is read with read(2): each event is sizeof(struct signalfd_siginfo). */
struct signalfd_siginfo {
    __u32 ssi_signo;       /* signal number */
    __s32 ssi_errno;       /* error from kernel (always 0) */
    __s32 ssi_code;        /* SI_USER / SI_KERNEL / SI_QUEUE / CLD_EXITED / ... */
    __u32 ssi_pid;
    __u32 ssi_uid;
    __s32 ssi_fd;
    __u32 ssi_tid;
    __u32 ssi_band;        /* SIGIO band */
    __u32 ssi_overrun;     /* POSIX timer */
    __u32 ssi_trapno;      /* hardware trap */
    __s32 ssi_status;      /* exit code / signal (SIGCHLD) */
    __s32 ssi_int;         /* sigqueue payload */
    __u64 ssi_ptr;
    __u64 ssi_utime;
    __u64 ssi_stime;
    __u64 ssi_addr;
    __u16 ssi_addr_lsb;
    __u8  __pad2[2];
    __s32 ssi_syscall;
    __u64 ssi_call_addr;
    __u32 ssi_arch;
    __u8  __pad[28];       /* total 128 bytes */
};
_Static_assert(sizeof(struct signalfd_siginfo) == 128, "");
```

## Compatibility contract

REQ-1: Syscall number is **282** on x86_64; **321** on i386; **NOT exposed** on arm64/generic (modern arches only provide `signalfd4` syscall 289). Kernel uses `__ARCH_WANT_SYS_SIGNALFD` to compile-in legacy entry.

REQ-2: `fd == -1` creates a NEW signalfd. Returned fd has `O_RDONLY` semantics for `read`, `O_RDWR` for `fcntl/poll`. Flags: NO `O_CLOEXEC`, NO `O_NONBLOCK`. (This is the legacy hazard.)

REQ-3: `fd >= 0` MUST be an existing signalfd; the syscall replaces its signal-mask with the new `mask`. Returns `fd` unchanged. Any other fd type → `EINVAL`.

REQ-4: `sizemask` MUST equal `sizeof(sigset_t)` = `_NSIG / 8` (8 on most arches, 16 on i386 with realtime sigset). Otherwise `EINVAL`.

REQ-5: `SIGKILL` and `SIGSTOP` are silently masked-out of the effective mask; they cannot be delivered via signalfd (per `sigaction(2)` general rule).

REQ-6: Signals delivered to signalfd are NOT delivered to handlers/`sigwait`/default action; they sit in the per-task/per-process pending queue. `read(2)` on the signalfd dequeues them. Multiple readers dequeue distinct events (no broadcast).

REQ-7: `read(2)` on a signalfd returns ≥1 `struct signalfd_siginfo` records (multiple if buffer is large). Short reads MUST fit at least one complete record (128 bytes); buffer < 128 → `EINVAL`. With no events pending and blocking mode (no `O_NONBLOCK`), `read` blocks; with `O_NONBLOCK`, `read` returns `EAGAIN`.

REQ-8: `poll(2)`/`epoll(2)`/`select(2)` on signalfd: `POLLIN`/`EPOLLIN` is asserted when any signal in the effective mask is pending; level-triggered.

REQ-9: Multiple signalfds in a process MAY have overlapping masks; the first reader to `read` consumes the signal. POSIX-style cooperation.

REQ-10: Closing the last fd referring to a signalfd_ctx releases the context. Pending signals remain in the task/process queue (default actions resume if no handler installed) UNLESS the mask was previously installed via `sigprocmask`.

REQ-11: `dup(2)`/`dup2(2)` works normally; shares the underlying `signalfd_ctx`. `fcntl(F_SETFL, O_NONBLOCK)` is permitted post-create.

REQ-12: `execve(2)` semantics: signalfd survives execve (no `O_CLOEXEC` by default — security hazard).

REQ-13: PaX UDEREF: pre-validate `mask` userspace pointer before `copy_from_user`. Reject kernel-half pointer ranges.

REQ-14: LSM hook: `security_file_alloc` fires on new signalfd creation (general file hook).

REQ-15: signalfd events MUST be format-stable: `struct signalfd_siginfo` is 128 bytes, exact, frozen ABI since 2.6.22. Adding fields requires `__pad` reuse only, never expansion.

REQ-16: Per-process / per-task pending signals: signalfd intercepts BOTH per-thread (`current->pending`) and shared (`current->signal->shared_pending`) signals matching the mask. Determinism: synchronous read-driven dequeue.

## Acceptance Criteria

- [ ] AC-1: `signalfd(-1, &mask, sizeof(sigset_t))` returns a valid fd; subsequent read returns `struct signalfd_siginfo` on signal delivery.
- [ ] AC-2: Reading from a signalfd without pending signals blocks (default) or returns `EAGAIN` (with `O_NONBLOCK`).
- [ ] AC-3: `epoll_wait` returns `EPOLLIN` on a signalfd with pending matching signal.
- [ ] AC-4: `signalfd(existing_fd, &new_mask, ...)` updates the mask; new mask takes effect for subsequent signals.
- [ ] AC-5: `signalfd(non_signalfd, ...)` returns `EINVAL`.
- [ ] AC-6: `signalfd(-1, mask, sizemask = 7)` returns `EINVAL`.
- [ ] AC-7: `signalfd(-1, NULL, sizeof(sigset_t))` returns `EFAULT`.
- [ ] AC-8: Returned fd has NO `O_CLOEXEC` (verified via `fcntl(F_GETFD)`).
- [ ] AC-9: `SIGKILL` in mask is silently dropped — `read` never returns SIGKILL via signalfd.
- [ ] AC-10: After fork, child inherits signalfd; parent and child share the underlying signal queue for shared signals.
- [ ] AC-11: After execve without `FD_CLOEXEC`, signalfd survives (legacy semantics).
- [ ] AC-12: Read with buffer < 128 returns `EINVAL`.

## Architecture

```rust
#[syscall(nr = 282, abi = "sysv")]
pub fn sys_signalfd(
    fd: i32,
    mask: UserPtr<SigSet>,
    sizemask: usize,
) -> KResult<i32> {
    /* Legacy: forward to signalfd4 with flags = 0 */
    Signalfd::do_signalfd(fd, mask, sizemask, 0)
}
```

`Signalfd::do_signalfd(fd, mask_uptr, sizemask, flags) -> KResult<i32>`:
1. /* sigset size check */
2. if sizemask != core::mem::size_of::<SigSet>() { return Err(EINVAL); }
3. /* UDEREF userspace validation */
4. access_ok_uderef(mask_uptr, sizemask)?;
5. let mut mask: SigSet = SigSet::zeroed();
6. copy_from_user(&mut mask, mask_uptr, sizemask)?;
7. /* Mask out SIGKILL/SIGSTOP */
8. mask.clear(SIGKILL); mask.clear(SIGSTOP);
9. if fd == -1 {
   - /* Create */
   - let ctx = SignalfdCtx::new(mask);
   - let file = anon_inode_getfile("[signalfd]", &SIGNALFD_FOPS, ctx, O_RDWR)?;
   - /* Legacy: NO O_CLOEXEC, NO O_NONBLOCK */
   - let new_fd = get_unused_fd_flags(0)?;
   - fd_install(new_fd, file);
   - Ok(new_fd)
10. } else {
   - let file = fdget(fd)?;
   - let ctx = SignalfdCtx::from_file(&file).ok_or(EINVAL)?;
   - /* Update mask atomically */
   - ctx.lock().mask = mask;
   - Ok(fd)
11. }

`SignalfdCtx::read(buf: &mut [u8]) -> KResult<usize>`:
1. if buf.len() < sizeof::<SignalfdSigInfo>() { return Err(EINVAL); }
2. loop:
   - /* Try dequeue first matching signal */
   - if let Some(info) = current().dequeue_signal_with_mask(self.mask) {
     - let ssi: SignalfdSigInfo = SignalfdSigInfo::from_kernel_siginfo(&info);
     - copy_to_user_slice(buf, &ssi)?;
     - return Ok(sizeof::<SignalfdSigInfo>());
   - }
   - if self.file.flags & O_NONBLOCK { return Err(EAGAIN); }
   - wait_interruptible_until_signal_in_mask(self.mask)?;
3. /* unreached */

`SignalfdSigInfo::from_kernel_siginfo(si: &SigInfo) -> SignalfdSigInfo`:
1. Zero all 128 bytes first (no stack leak).
2. self.ssi_signo = si.si_signo as u32.
3. self.ssi_errno = si.si_errno.
4. self.ssi_code = si.si_code.
5. self.ssi_pid = si.pid_in_caller_ns().
6. self.ssi_uid = si.si_uid.
7. /* Copy union fields per si_code */
8. match si.si_code: CLD_*: self.ssi_status = si.si_status; ...; SI_TIMER: self.ssi_overrun = si.overrun; ...
9. return self.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sizemask_exact` | INVARIANT | per-signalfd: sizemask == sizeof(sigset_t). |
| `sigkill_sigstop_excluded` | INVARIANT | per-mask: SIGKILL/SIGSTOP cleared. |
| `siginfo_zeroed_before_write` | INVARIANT | per-read: SignalfdSigInfo cleared before populate. |
| `uderef_pre_copy_from_user` | INVARIANT | per-mask: UDEREF check before copy_from_user. |
| `ctx_lifetime_balanced` | INVARIANT | per-create/close: refcount get/put symmetric. |
| `read_min_record_size` | INVARIANT | per-read: buf < 128 returns EINVAL. |
| `legacy_no_cloexec` | INVARIANT | per-legacy: returned fd has FD_CLOEXEC unset. |

### Layer 2: TLA+

`fs/signalfd.tla`:
- States: per-task signal-queue, per-fd signalfd_ctx with mask, per-fd read-pointer.
- Properties:
  - `safety_one_consumer_per_signal` — per-signal: dequeued by exactly one reader.
  - `safety_mask_immediate` — per-mask-update: new mask filters from next dequeue.
  - `liveness_eventual_delivery` — per-pending-in-mask signal: eventually readable.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_signalfd` post: errno mapping matches man-page | `Signalfd::do_signalfd` |
| `read` post: returns multiple-of-128 bytes | `SignalfdCtx::read` |
| `from_kernel_siginfo` post: 128-byte stable layout | `SignalfdSigInfo::from_kernel_siginfo` |

### Layer 4: Verus / Creusot functional

Per-`signalfd(2)` man-page equivalence. LTP `signalfd01` pass. Glibc forward-compat: `signalfd(3)` wrapping syscall 289 produces equivalent fd to legacy 282 with `flags = 0`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`signalfd(2)` legacy reinforcement:

- **Per-sigset-size exact match** — defense against per-undersized-mask info-leak.
- **Per-SIGKILL/SIGSTOP exclusion** — defense against per-uninterruptible-signal hijack.
- **Per-siginfo full-zero before populate** — defense against per-stale-stack info-leak.
- **Per-ctx refcount balanced** — defense against per-UAF on close-after-read.
- **Per-mask atomic update** — defense against per-torn-mask race on concurrent update.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `sigset_t *mask`** — every copy_from_user(mask) MUST pass UDEREF SMAP/SMEP gate; non-userspace pointers rejected before kernel reads.
- **signalfd CLOEXEC mandatory in hardened mode** — grsec REWRITES legacy `signalfd(2)` to forcibly set `FD_CLOEXEC` after allocation, eliminating the post-execve signalfd-leak class of bug. Userspace that depends on the legacy "fd survives execve" semantics MUST use `signalfd4(SFD_CLOEXEC)` explicitly to opt-out (grsec adds a sysctl `kernel.signalfd_legacy_cloexec_default = 1`).
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — signalfd context cannot be obtained for inspection by ptracers; signal masks are opaque to peers.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fdinfo/<sfd>` mask field is filtered for non-owners; you cannot enumerate another process's signalfd mask without sufficient privilege.
- **PIDFD_SELF / pidfd interplay** — signalfd is per-task/per-process and does NOT use pidfd targeting; grsec asserts no cross-PID-namespace signal delivery via signalfd that would not also happen via direct queue.
- **PaX UDEREF on signalfd_siginfo write** — copy_to_user(&ssi) on read path goes through UDEREF; prevents partial-write leak of stack residue if siginfo populate fails midway.
- **GRKERNSEC_AUDIT_GROUP** — frequent signalfd(-1, ...) creations from a marked group are rate-limited / audited to detect signal-queue enumeration scans.
- **No_new_privs neutral** — NNP does not change signalfd semantics; signal handling is unprivileged.
- **Anti-fingerprint hardening** — `ssi_pid` rendered in caller's PID namespace; host PIDs never leak to containerized signalfd readers.
- **Capability gate for cross-task signalfd** — signalfd is always per-current; you cannot create a signalfd for another task. (Grsec enforces this even if future patches attempt to add `signalfd_open` style cross-task variants.)
- **Seccomp interaction** — seccomp filters may explicitly block syscall 282 in favor of 289; default-deny hardening recommends forbidding legacy signalfd entirely.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `signalfd4(2)` (Tier-5 separate doc — modern variant with flags).
- `sigwaitinfo(2)` / `sigtimedwait(2)` (Tier-5 separate docs).
- `sigaction(2)` / `sigprocmask(2)` (Tier-5 separate docs).
- `epoll_wait` integration semantics (covered in `epoll_wait.md`).
- Signal-delivery internals (Tier-3 in `kernel/signal.md`).
- Implementation code.
