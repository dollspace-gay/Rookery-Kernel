# Tier-5 syscall: clone3(2) — syscall 435

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/fork.c (sys_clone3, copy_clone_args_from_user, kernel_clone)
  - include/linux/syscalls.h (SYSCALL_DEFINE2(clone3, ...))
  - include/uapi/linux/sched.h (struct clone_args, CLONE_ARGS_SIZE_VER0/1/2, upper-32 CLONE_* flags)
  - arch/x86/entry/syscalls/syscall_64.tbl (435  common  clone3)
-->

## Summary

`clone3(2)` is the modern, extensible task-creation syscall. It supersedes
`clone(2)` for all new code, removing the per-arch argument-ordering quirks
and lifting the flag space from 32 bits to a full **u64** (the upper 32 bits
host `clone3`-only flags such as `CLONE_INTO_CGROUP`, `CLONE_CLEAR_SIGHAND`,
`CLONE_PIDFD_AUTOKILL`). The arguments are packed into a single
`struct clone_args` whose size is **versioned** — the syscall takes a
`size_t size` second argument and the kernel accepts any of the published
sizes `CLONE_ARGS_SIZE_VER0 = 64`, `_VER1 = 80`, `_VER2 = 88`, plus larger
values whose trailing bytes are entirely zero (forward-compatibility).

Container runtimes (`runc`, `youki`, `crun`, `systemd-nspawn` in modern
versions) prefer `clone3` because `CLONE_INTO_CGROUP` atomically places the
child in a target cgroup at creation time — eliminating the window where a
freshly forked child is briefly in the parent's cgroup. Process supervisors
(`systemd` since v245) prefer `CLONE_PIDFD_AUTOKILL` for sidecar
"watchdog-bound" semantics: when the supervising fd is dropped, the child
is killed. Sandbox builders use `CLONE_CLEAR_SIGHAND` for deterministic
signal-disposition reset and `CLONE_NNP` for inline `no_new_privs`.

This Tier-5 covers `kernel/fork.c::SYSCALL_DEFINE2(clone3, ...)` plus the
`copy_clone_args_from_user` helper (~120 lines of direct entry-point code).
The shared `kernel_clone` workhorse and `copy_process` task-duplication are
covered by the Tier-3 `kernel/fork.c` document.

## Signature

```c
long clone3(struct clone_args *uargs, size_t size);
```

Both arguments are positional and identical across all architectures — the
removal of per-arch arg ordering is one of `clone3`'s headline ABI fixes.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `uargs` | `struct clone_args *` | in/out | Pointer to a kernel-visible argument block, 8-byte aligned. `pidfd` and `child_tid`/`parent_tid` fields are out-direction (kernel writes into the pointed-to user memory). |
| `size`  | `size_t` | in | Size in bytes of the user-supplied `clone_args`. MUST be one of `64`, `80`, `88`, or a larger value whose trailing bytes are zero. |

## `struct clone_args` (VER0 / VER1 / VER2)

```c
struct clone_args {
    __aligned_u64 flags;          /* CLONE_* (low + upper-32 bits) */
    __aligned_u64 pidfd;          /* &int — output pidfd if CLONE_PIDFD */
    __aligned_u64 child_tid;      /* &pid_t — for CLONE_CHILD_SETTID */
    __aligned_u64 parent_tid;     /* &pid_t — for CLONE_PARENT_SETTID */
    __aligned_u64 exit_signal;    /* signal queued to parent on exit */
    __aligned_u64 stack;          /* lowest address of child stack */
    __aligned_u64 stack_size;     /* size of child stack */
    __aligned_u64 tls;            /* TLS register value (CLONE_SETTLS) */
    /* ---- VER0 boundary: 64 bytes ---- */
    __aligned_u64 set_tid;        /* &pid_t[] — preselected PIDs */
    __aligned_u64 set_tid_size;   /* count of set_tid array */
    /* ---- VER1 boundary: 80 bytes ---- */
    __aligned_u64 cgroup;         /* fd of target cgroup (CLONE_INTO_CGROUP) */
    /* ---- VER2 boundary: 88 bytes ---- */
};
```

Every field is `__aligned_u64` (an 8-byte-aligned `u64`). Userspace `int *`
and `pid_t *` are zero-extended into the u64 by the libc wrapper before
the syscall. New fields are added strictly at the end, gated by the
versioned `size` boundary.

## Return value

| Value | Meaning |
|---|---|
| `> 0` (in parent) | Child PID in caller's pid_ns. |
| `0`   (in child)  | Successful clone — child execution begins here. |
| `-1` + `errno`    | Failure (no child created); see Errors. |

## Errors

| errno | Trigger |
|---|---|
| `E2BIG`  | `size > kernel-known max AND any trailing byte != 0`. |
| `EINVAL` | `size < CLONE_ARGS_SIZE_VER0 (64)`. |
| `EINVAL` | `uargs` misaligned (< 8-byte alignment). |
| `EINVAL` | Flag combination invalid (see REQ-12..REQ-22). |
| `EINVAL` | `exit_signal > 0xff` or `exit_signal` names an invalid signal. |
| `EINVAL` | `CLONE_INTO_CGROUP && cgroup` is not a valid cgroup-v2 fd. |
| `EBADF`  | `CLONE_INTO_CGROUP && cgroup` fd does not refer to an open file. |
| `ENOMEM` | task allocation failed. |
| `EAGAIN` | `RLIMIT_NPROC` exceeded. |
| `EUSERS` | per-user-ns nesting limit exceeded. |
| `EFAULT` | `uargs` not readable / any output pointer not writable. |
| `EEXIST` | `set_tid[i]` already in use in target pid_ns. |
| `ENOSPC` | per-pid-ns task limit reached. |
| `EOPNOTSUPP` | `set_tid` used but target pid_ns lacks `INIT_USER_NS` privilege. |

## ABI surface

### Versioning

```text
CLONE_ARGS_SIZE_VER0 = 64    /* original publication: through `tls` */
CLONE_ARGS_SIZE_VER1 = 80    /* + set_tid, set_tid_size */
CLONE_ARGS_SIZE_VER2 = 88    /* + cgroup */
```

Behavior:
- `size < VER0` → `-EINVAL`.
- `size in {64, 80, 88}` → exact-version copy.
- `size > VER2` AND any byte > 88 is non-zero → `-E2BIG`.
- `size > VER2` AND all bytes > 88 are zero → accepted (forward-compat).

### Upper-32 `CLONE_*` flags (clone3-only)

```text
CLONE_CLEAR_SIGHAND  = 1ULL << 32   /* clear all sig handlers — reset to SIG_DFL */
CLONE_INTO_CGROUP    = 1ULL << 33   /* place child in cgroup specified by clone_args.cgroup */
CLONE_AUTOREAP       = 1ULL << 34   /* auto-reap child on exit (kernel-side SA_NOCLDWAIT) */
CLONE_NNP            = 1ULL << 35   /* set no_new_privs on child */
CLONE_PIDFD_AUTOKILL = 1ULL << 36   /* SIGKILL child when last pidfd ref drops */
CLONE_EMPTY_MNTNS    = 1ULL << 37   /* create an empty mount namespace */
```

| Flag | Effect | Constraint |
|---|---|---|
| `CLONE_CLEAR_SIGHAND` | All sig dispositions → `SIG_DFL` in child. | Mutually exclusive with `CLONE_SIGHAND` (sharing + clearing is incoherent). |
| `CLONE_INTO_CGROUP` | Atomically attach child to `clone_args.cgroup` (cgroup-v2 fd). | Requires write access to `cgroup.threads`/`cgroup.procs`; cgroup must be v2. |
| `CLONE_AUTOREAP` | Child auto-reaps on exit (no zombie). | Mutually exclusive with non-zero `exit_signal`. |
| `CLONE_NNP` | `task_struct.no_new_privs = 1` in child. | Survives `execve`; cannot be cleared. |
| `CLONE_PIDFD_AUTOKILL` | On last pidfd put, `SIGKILL` is queued to child. | Requires `CLONE_PIDFD`. |
| `CLONE_EMPTY_MNTNS` | Child mount-ns is empty (no mounts inherited). | Requires `CLONE_NEWNS` + `CAP_SYS_ADMIN`. |

### Low-32 `CLONE_*` flags

(Identical to `clone(2)`; see `uapi/syscalls/clone.md` REQ-table.)

Notable: `CLONE_PIDFD` and `CLONE_THREAD` MAY be combined via `clone3(2)`
(a per-thread pidfd works, returning a TID-scoped pidfd useful for
`pidfd_send_signal` from co-threads). This combination is FORBIDDEN via
`clone(2)`.

`CLONE_NEWTIME` is reachable via `clone3` because the `exit_signal` field is
separate from the flags u64 — no `CSIGNAL` overlap.

## Compatibility contract

REQ-1: The syscall number is **435** on x86_64 / generic. Architecture-specific
tables MUST register at this slot.

REQ-2: `size` MUST be `>= CLONE_ARGS_SIZE_VER0 (64)`. Smaller MUST return
`-EINVAL` before any user-pointer dereference.

REQ-3: `size` MAY be any value in `{64, 80, 88}` (published versions) — kernel
copies the exact byte count and zero-fills the rest of its internal
`clone_args`.

REQ-4: `size > 88`: kernel reads bytes 0..88 as the known fields, then scans
bytes 88..size; any non-zero byte returns `-E2BIG`. This enforces
forward-compat: a future kernel adding new fields cannot have a stale
userspace silently set garbage.

REQ-5: `uargs` MUST be 8-byte aligned. Misalignment MUST return `-EINVAL`
(NOT `-EFAULT` — alignment is an ABI contract, faults are about mappings).

REQ-6: `copy_clone_args_from_user` MUST use `copy_struct_from_user` semantics:
copy `min(size, sizeof(kernel_struct))` bytes, then zero-check trailing,
then for kernel-newer-than-user case zero the unsupplied fields.

REQ-7: `clone_args.exit_signal`: low 8 bits hold the signal; bits 8..63 MUST be
zero. Non-zero high bits MUST return `-EINVAL`.

REQ-8: `clone_args.exit_signal` value: `0` (legal only with `CLONE_THREAD` or
`CLONE_AUTOREAP`), or `1..(_NSIG-1)`. Out-of-range MUST return `-EINVAL`.

REQ-9: `clone_args.set_tid` (VER1+): pointer to array of `set_tid_size` pid_t
values. The new task acquires the listed pids in successive pid_namespaces
(innermost first). Requires `CAP_SYS_ADMIN` in the *target* pid_ns owner
user-ns. Conflict with an in-use pid returns `-EEXIST`.

REQ-10: `clone_args.set_tid_size` MUST be `<= MAX_PID_NS_LEVEL (32)`. Exceeding
returns `-EINVAL`. A value of 0 means "kernel allocates pids" (default).

REQ-11: `clone_args.cgroup` (VER2+): fd referencing a cgroup-v2 directory;
ignored unless `CLONE_INTO_CGROUP` is set. Caller MUST have write access to
the cgroup's `cgroup.procs` (or `cgroup.threads` if `CLONE_THREAD`).

REQ-12: All `clone(2)` flag combination rules (clone.md REQ-2..REQ-10) apply,
EXCEPT: `CLONE_PIDFD | CLONE_THREAD` IS PERMITTED via `clone3`.

REQ-13: `CLONE_CLEAR_SIGHAND | CLONE_SIGHAND` returns `-EINVAL` (clearing and
sharing dispositions is incoherent).

REQ-14: `CLONE_INTO_CGROUP` without `cgroup` fd (== 0 sentinel rejected by
fdget) returns `-EBADF`.

REQ-15: `CLONE_INTO_CGROUP` with a cgroup-v1 fd returns `-EINVAL`. Cgroup-v2
is mandatory because v1's controller-disjoint hierarchies prevent atomic
placement.

REQ-16: `CLONE_INTO_CGROUP` with a frozen cgroup returns `-EBUSY` — placing
into a frozen cgroup would immediately stop the child mid-task-setup.

REQ-17: `CLONE_AUTOREAP` with `exit_signal != 0` returns `-EINVAL` (parent
cannot receive SIGCHLD for an auto-reaped child).

REQ-18: `CLONE_NNP`: equivalent to `prctl(PR_SET_NO_NEW_PRIVS, 1)` performed
in the child immediately after `copy_process`. Cannot be undone after
`execve`.

REQ-19: `CLONE_PIDFD_AUTOKILL` requires `CLONE_PIDFD`. Standalone returns
`-EINVAL`. Combined with `CLONE_AUTOREAP` is permitted: the child is
auto-reaped *and* gets `SIGKILL` on last pidfd close — useful for sidecar.

REQ-20: `CLONE_EMPTY_MNTNS` requires `CLONE_NEWNS` and `CAP_SYS_ADMIN` in the
new user-ns (or initial user-ns). Standalone returns `-EINVAL`.

REQ-21: Upper-32 bits beyond bit 37 MUST be zero. Setting unknown high bits
returns `-EINVAL` (sealed ABI — no silent ignore).

REQ-22: Unknown LOW-32 bits also return `-EINVAL`. Same sealing rule.

REQ-23: User pointer validation order: `uargs` first, then output pointers
(`pidfd`, `parent_tid`, `child_tid`, `set_tid`, `cgroup` fd validity) before
any task allocation. Failure MUST return the error without side effects.

REQ-24: Audit: every successful `clone3` emits `AUDIT_SYSCALL` with the
full flags, child pid, set_tid contents, and any namespace IDs created.

REQ-25: `clone3(uargs, size)` with `current` in a chroot: subject to
GRKERNSEC_CHROOT denial of `CLONE_NEW{USER,PID,NS,NET}` (see Hardening).

## Acceptance Criteria

- [ ] AC-1: `clone3` syscall number is 435 on x86_64.
- [ ] AC-2: `clone3(uargs, 0)` → `-EINVAL`.
- [ ] AC-3: `clone3(uargs, 63)` → `-EINVAL`.
- [ ] AC-4: `clone3(uargs, 64)` succeeds (VER0).
- [ ] AC-5: `clone3(uargs, 80)` succeeds (VER1); `set_tid` honored.
- [ ] AC-6: `clone3(uargs, 88)` succeeds (VER2); `cgroup` honored.
- [ ] AC-7: `clone3(uargs, 96)` with trailing-zero bytes succeeds; with any non-zero byte → `-E2BIG`.
- [ ] AC-8: `uargs` 4-byte aligned (not 8) → `-EINVAL`.
- [ ] AC-9: `exit_signal = 0xff_0000_0000_0001` (high bit set) → `-EINVAL`.
- [ ] AC-10: `flags & CLONE_NEWTIME` set via `clone3`: succeeds.
- [ ] AC-11: `flags & (CLONE_PIDFD | CLONE_THREAD)` succeeds, returning per-thread pidfd.
- [ ] AC-12: `flags & (CLONE_CLEAR_SIGHAND | CLONE_SIGHAND)` → `-EINVAL`.
- [ ] AC-13: `CLONE_INTO_CGROUP` with v1 fd → `-EINVAL`.
- [ ] AC-14: `CLONE_INTO_CGROUP` with closed fd → `-EBADF`.
- [ ] AC-15: `CLONE_INTO_CGROUP` into frozen cgroup → `-EBUSY`.
- [ ] AC-16: `CLONE_AUTOREAP` with `exit_signal = SIGCHLD` → `-EINVAL`.
- [ ] AC-17: `CLONE_PIDFD_AUTOKILL` without `CLONE_PIDFD` → `-EINVAL`.
- [ ] AC-18: `CLONE_PIDFD_AUTOKILL`: closing last pidfd queues SIGKILL.
- [ ] AC-19: `CLONE_NNP`: child has `prctl(PR_GET_NO_NEW_PRIVS) == 1`.
- [ ] AC-20: `set_tid = [5000]` with pid 5000 free → child pid is 5000; with pid 5000 in use → `-EEXIST`.
- [ ] AC-21: `set_tid_size > 32` → `-EINVAL`.
- [ ] AC-22: Unknown bit 38 set → `-EINVAL`.
- [ ] AC-23: `CLONE_EMPTY_MNTNS` without `CLONE_NEWNS` → `-EINVAL`.
- [ ] AC-24: Auditd records flags, set_tid, ns-ids.

## Architecture

```rust
#[syscall(nr = 435, abi = "generic")]
pub fn sys_clone3(uargs: UserPtr<CloneArgs>, size: usize) -> isize {
    // REQ-2/3/4/5/6: size + alignment
    if size < CLONE_ARGS_SIZE_VER0       { return -EINVAL; }
    if uargs.as_usize() % 8 != 0         { return -EINVAL; }
    let kargs = match Fork::copy_clone_args_from_user(uargs, size) {
        Ok(a)  => a,
        Err(e) => return e,
    };
    Fork::kernel_clone(kargs)
}
```

`Fork::copy_clone_args_from_user(uargs, size) -> Result<KernelCloneArgs>`:
1. /* Zero-init kernel struct */
2. let mut kargs = KernelCloneArgs::zeroed();
3. /* Copy min(size, sizeof(kernel)) bytes */
4. let n = min(size, size_of::<CloneArgsLayout>());
5. user_access::copy_from_user(&mut kargs.layout, uargs, n)?;
6. /* If user > kernel: verify trailing zero */
7. if size > size_of::<CloneArgsLayout>() {
8.   let trailing = size - size_of::<CloneArgsLayout>();
9.   if !user_access::is_zero_region(uargs.offset(n), trailing)? { return Err(-E2BIG); }
10. }
11. /* Validate flag combinations (REQ-12..REQ-22) */
12. Fork::validate_flags_clone3(&kargs)?;
13. /* Validate exit_signal (REQ-7/8) */
14. if kargs.exit_signal > 0xff           { return Err(-EINVAL); }
15. if kargs.exit_signal as u8 >= _NSIG   { return Err(-EINVAL); }
16. /* Validate set_tid */
17. if kargs.set_tid_size > MAX_PID_NS_LEVEL { return Err(-EINVAL); }
18. /* Validate cgroup fd (if INTO_CGROUP) */
19. if kargs.flags & CLONE_INTO_CGROUP != 0 {
20.   kargs.cgroup_file = fdget(kargs.cgroup as i32)?;
21.   if !is_cgroup_v2(&kargs.cgroup_file)  { return Err(-EINVAL); }
22. }
23. Ok(kargs)

`Fork::validate_flags_clone3(args)`:
1. All of clone.md REQ-2..REQ-10 EXCEPT REQ-6 (clone3 permits PIDFD|THREAD).
2. Reject if `flags & ~(KNOWN_LOW32 | KNOWN_UPPER32) != 0`.
3. Reject if `CLONE_CLEAR_SIGHAND & CLONE_SIGHAND`.
4. Reject if `CLONE_AUTOREAP & exit_signal != 0`.
5. Reject if `CLONE_PIDFD_AUTOKILL & !CLONE_PIDFD`.
6. Reject if `CLONE_EMPTY_MNTNS & !CLONE_NEWNS`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_check_total`           | TOTAL | All `size: usize`: <64 → EINVAL; >88 with non-zero trailing → E2BIG. |
| `copy_struct_bounded`        | INVARIANT | `copy_clone_args_from_user` copies exactly `min(size, sizeof(kernel))` bytes. |
| `flag_validation_total`      | TOTAL | All `(flags: u64, exit_signal: u64)`: validate returns Ok or `-EINVAL`. |
| `cgroup_fd_consumed_on_ok`   | INVARIANT | Success ⟹ cgroup fdget'd; failure ⟹ fdput'd. |
| `pidfd_install_iff_requested`| INVARIANT | pidfd installed ⟺ `CLONE_PIDFD ∈ flags`. |

### Layer 2: TLA+

`kernel/clone3.tla`:
- Per-size-check → per-copy-args → per-flag-validate → per-kernel_clone → per-vfork-wait.
- Properties:
  - `safety_versioning_monotone` — accepted size monotone-equivalent: VER_n ⊆ VER_{n+1}.
  - `safety_unknown_flags_rejected` — any unknown bit ⟹ -EINVAL.
  - `safety_into_cgroup_v2_only` — `CLONE_INTO_CGROUP` ⟹ cgroup-v2.
  - `liveness_eventual_completion` — every accepted call returns (no infinite loop).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `copy_clone_args` post: kargs zero-padded beyond `size` | `Fork::copy_clone_args_from_user` |
| `validate_flags_clone3` total | `Fork::validate_flags_clone3` |
| `CLONE_INTO_CGROUP` post: cgroup_file holds a v2 cgroup ref | flag handler |
| `set_tid` post: each pid acquired or `-EEXIST` | `PidNamespace::alloc_pid_with_set_tid` |

### Layer 4: Verus / Creusot functional

Equivalence to `clone3(2)` man page and `kselftests/clone3/*`. Every published
size version exercised; every flag combination covered by LTP `clone301..`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`clone3(2)` reinforcement:

- **Per-versioned `size` sealed** — defense against per-future-bit-leak.
- **Per-trailing-zero check** — defense against per-stale-userspace-garbage.
- **Per-cgroup-v2 mandatory for `INTO_CGROUP`** — defense against per-controller-split race.
- **Per-fdget/fdput strict balancing on `cgroup` fd** — defense against per-cgroup-fd-leak on error path.
- **Per-`CLONE_NNP` set at copy_process** — defense against per-window-before-exec privilege carry.
- **Per-`CLONE_PIDFD_AUTOKILL` requires `CLONE_PIDFD`** — defense against per-orphan-AUTOKILL semantic.
- **Per-`CLONE_AUTOREAP` mutually exclusive with `exit_signal`** — defense against per-double-notify.
- **Per-`set_tid` `CAP_SYS_ADMIN` gated** — defense against per-pid-spoof.
- **Per-unknown-upper-bit `-EINVAL`** — defense against per-mainline-divergence on new bits.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — per-clone3 entry randomizes kernel-stack
  offset; combined with the `copy_clone_args` allocation pattern, defeats
  per-syscall stack-layout-reuse for ROP.
- **PaX UDEREF on user buffers** — `uargs`, `pidfd`, `parent_tid`, `child_tid`,
  `set_tid` array, cgroup `fdget` are all accessed via UDEREF / PAN; a kernel
  bug that bypasses `copy_struct_from_user` cannot silently read user memory.
- **GRKERNSEC_HARDEN_PTRACE** — `CLONE_PTRACE` honors the tighter
  yama+grsec policy (deny across-credential ptrace inherit).
- **GRKERNSEC_PROC restrictions** — newly allocated pids published to
  `/proc` are subject to per-uid/per-group hide-filter before any
  `/proc/<pid>` entry becomes lookupable.
- **GRKERNSEC_CHROOT** — chroot caller denied `CLONE_NEWUSER`, `CLONE_NEWPID`,
  `CLONE_NEWNS`, `CLONE_NEWNET`, `CLONE_INTO_CGROUP` (cgroup escape).
- **GRKERNSEC_BRUTE** — `clone3` returning `-EAGAIN` (NPROC) counts toward
  per-uid forkbomb detector; sustained over threshold → lockout.
- **GRKERNSEC_SIGNALS spoof prevention** — `exit_signal` queue uses
  `si_code = CLD_EXITED` with kernel-credentialed source; never an
  attacker-spoofable `SI_USER`.
- **Per-grsec `chroot_caps`** — chroot caller has effective capset bounded;
  `CLONE_NNP` is auto-set on every clone3 from chroot (defense-in-depth).
- **Per-grsec `gradm` learning** — `clone3` with `set_tid` recorded as a
  privileged action; offline RBAC policy MUST grant it explicitly.

## Open Questions

- (none at this Tier-5 level; future `CLONE_ARGS_SIZE_VER3` will extend this doc)

## Out of Scope

- `clone(2)` syscall (separate Tier-5 doc).
- `kernel_clone` / `copy_process` task duplication (Tier-3 `kernel/fork.c`).
- Cgroup-v2 internals (Tier-3 `kernel/cgroup/cgroup.c`).
- Pid-namespace internals (Tier-3 `kernel/pid_namespace.c`).
- Implementation code.
