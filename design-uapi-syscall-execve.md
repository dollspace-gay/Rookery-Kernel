---
title: "Tier-5 syscall: execve(2) — syscall 59"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`execve(2)` replaces the calling task's memory image with a new program
loaded from a filesystem path. It is the second half of the canonical
Unix "fork-then-exec" sequence: `fork(2)` + `execve(2)` = "create a new
process running a different program." On success, `execve` does NOT
return — control transfers to the new program's entry point. On failure
it returns `-1` with `errno`.

`execve` is the ONE syscall where the security model has to:
1. Open + read the binary, parse it (ELF, script, misc binfmt), select a
   loader.
2. Compute new credentials (setuid/setgid bit propagation, file
   capabilities, securebits, no_new_privs gating).
3. Decide whether the new program runs with elevated privilege (`AT_SECURE`
   auxv flag).
4. Tear down the calling task's mm, threads, files (close-on-exec),
   timers, robust-futex list, sighand-state, ptrace state.
5. Build a brand-new mm with argv, envp, auxv, stack, initial mmap of the
   binary's program headers.
6. Switch to the new credentials and PC.

Critical for: every program start on Linux, every container `ENTRYPOINT`,
every shell builtin `exec`, every `setuid` binary (`sudo`, `passwd`,
`mount`).

This Tier-5 covers the entry surface (`fs/exec.c::SYSCALL_DEFINE3(execve,
...)` and the `do_execveat_common` workhorse insofar as ABI-visible).
The complete binfmt + load_elf + creds-recompute machinery is Tier-3 in
`fs/exec.c` / `fs/binfmt_elf.c` / `kernel/cred.c`.

### Acceptance Criteria

- [ ] AC-1: `execve("/bin/true", argv={"true", NULL}, envp={NULL})` succeeds and `/bin/true` exits 0.
- [ ] AC-2: `execve("/nonexistent", ...)` → `-ENOENT`.
- [ ] AC-3: `execve("/tmp/nonregular_file", ...)` → `-EACCES`.
- [ ] AC-4: `execve` on a `noexec`-mounted file → `-EACCES`.
- [ ] AC-5: `execve` with an ELF file of wrong architecture → `-ENOEXEC` (or `-EINVAL` for malformed ELF).
- [ ] AC-6: `execve` with a single argv string of length `MAX_ARG_STRLEN+1` → `-E2BIG`.
- [ ] AC-7: `execve` with `MAX_ARG_STRINGS+1` total argv entries → `-E2BIG`.
- [ ] AC-8: `execve` with combined argv+envp size > `RLIMIT_STACK/4` → `-E2BIG`.
- [ ] AC-9: `execve(path, NULL, NULL)` succeeds with empty argv/envp.
- [ ] AC-10: setuid binary: new euid = file owner uid; `AT_SECURE = 1`.
- [ ] AC-11: setuid binary on `MNT_NOSUID` mount: euid unchanged; `AT_SECURE = 0`.
- [ ] AC-12: `prctl(PR_SET_NO_NEW_PRIVS, 1)` then `execve(setuid_binary)`: euid unchanged; `AT_SECURE = 0`.
- [ ] AC-13: ptrace tracer receives `PTRACE_EVENT_EXEC` at point-of-no-return.
- [ ] AC-14: Co-thread in caller's tgid: killed before new program runs.
- [ ] AC-15: O_CLOEXEC fds closed; non-CLOEXEC fds preserved.
- [ ] AC-16: sigactions for non-SIG_DFL/SIG_IGN handlers reset to SIG_DFL.
- [ ] AC-17: cgroup membership preserved across execve.
- [ ] AC-18: `#!` interpreter recursion depth > 5 → `-ELOOP`.
- [ ] AC-19: File-cap revision-3 with rootid != current rootuserns → `-EINVAL`.
- [ ] AC-20: Failed execve (e.g. `-ENOEXEC`): caller's mm, fds, creds unchanged.
- [ ] AC-21: AUDIT_EXECVE record contains resolved path and argv.

### Architecture

```rust
#[syscall(nr = 59, abi = "sysv")]
pub fn sys_execve(
    path:  UserPtr<u8>,
    argv:  UserPtr<UserPtr<u8>>,
    envp:  UserPtr<UserPtr<u8>>,
) -> isize {
    Exec::do_execveat_common(AT_FDCWD, path, argv, envp, 0)
}
```

`Exec::do_execveat_common(dfd, path, argv, envp, flags) -> isize`:
1. /* Resolve and open binary */
2. let file = Vfs::do_open_execat(dfd, path, flags)?;
3. /* Allocate linux_binprm */
4. let mut bprm = LinuxBinprm::alloc(&file)?;
5. /* Read BINPRM_BUF_SIZE bytes for binfmt detection */
6. Exec::prepare_binprm(&mut bprm)?;
7. /* Count + copy argv, envp strings into bprm vma */
8. bprm.argc = Exec::count_strings(argv, MAX_ARG_STRINGS)?;
9. bprm.envc = Exec::count_strings(envp, MAX_ARG_STRINGS - bprm.argc)?;
10. Exec::copy_strings(bprm.argc, argv, &mut bprm)?;
11. Exec::copy_strings(bprm.envc, envp, &mut bprm)?;
12. /* Compute credentials */
13. Exec::prepare_creds(&mut bprm)?;
14. /* Dispatch binfmt — point of no return */
15. let res = Exec::search_binary_handler(&mut bprm);
16. /* res Err here ⟹ force_sigsegv if past commit; else return error */
17. ...

`Exec::prepare_creds(bprm)`:
1. let inode = bprm.file.inode();
2. /* setuid / setgid handling */
3. if inode.mode & S_ISUID && !bprm.mnt_nosuid && !task.no_new_privs {
4.   bprm.cred.euid = inode.uid;
5. }
6. if inode.mode & S_IXGRP && inode.mode & S_ISGID && !bprm.mnt_nosuid && !task.no_new_privs {
7.   bprm.cred.egid = inode.gid;
8. }
9. /* File caps */
10. Exec::get_file_caps(bprm)?;       // reads security.capability xattr
11. Exec::compute_capabilities(bprm)?; // capabilities(7) formula
12. /* AT_SECURE flag */
13. bprm.secureexec = is_secure_transition(&bprm.cred, &task.cred);

`Exec::search_binary_handler(bprm) -> isize`:
1. for handler in registered_binfmts {
2.   match handler.load_binary(bprm) {
3.     Ok(()) => return 0,                  // point-of-no-return passed
4.     Err(-ENOEXEC) => continue,           // try next handler
5.     Err(e)        => return e,
6.   }
7. }
8. return -ENOEXEC;

### Out of Scope

- `execveat(2)` syscall (separate Tier-5 doc).
- ELF binfmt loader (`fs/binfmt_elf.c` Tier-3).
- Script binfmt loader (`fs/binfmt_script.c` Tier-3).
- binfmt_misc handler (`fs/binfmt_misc.c` Tier-3).
- Credential-recomputation internals (`kernel/cred.c` Tier-3).
- ELF auxv full layout (`uapi/auxv.md` Tier-5 — separate).
- Implementation code.

### signature

```c
long execve(const char *path,
            char *const argv[],
            char *const envp[]);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated pathname of the executable. May be a symlink (followed). |
| `argv` | `char *const []` | in | NULL-terminated array of pointers to NUL-terminated strings; `argv[0]` is conventionally the program name. |
| `envp` | `char *const []` | in | NULL-terminated array of `KEY=VALUE` strings for the new environment. |

`argv` and `envp` MAY be (but should not be) `NULL` — modern kernels accept
NULL and treat it as an empty array (since v5.18 explicitly; previously
returned `-EFAULT`).

### return value

| Value | Meaning |
|---|---|
| (does not return) | Success — calling task is now executing the new program. |
| `-1` + `errno`    | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES`  | `path` not regular file, not executable by caller, on a `noexec` mount, or interpreter for a `#!` script is not executable. |
| `EFAULT`  | `path` / `argv` / `envp` / any string pointer not readable. |
| `ELOOP`   | Symlink loop exceeded `MAXSYMLINKS`. |
| `ENAMETOOLONG` | `path` length exceeds `PATH_MAX`. |
| `ENOENT`  | `path` does not exist. |
| `ENOEXEC` | No binfmt handler accepts the file (not ELF, not script, etc.). |
| `ENOMEM`  | Out of memory while building new mm. |
| `ENOTDIR` | Non-directory component in `path`. |
| `EPERM`   | Calling task has `MNT_NOSUID` and the file has setuid/setgid bits; or seccomp denied. |
| `ETXTBSY` | File is open for writing by another process (`MAY_WRITE` deny). |
| `E2BIG`   | argv+envp+strings exceed `_STK_LIM/4` (typically `1/4 * RLIMIT_STACK`); or any single string > `MAX_ARG_STRLEN`; or total strings > `MAX_ARG_STRINGS`. |
| `EINVAL`  | ELF e_type unsupported, or interpreter not statically-PIE / dynamic-linker malformed. |
| `EIO`     | I/O error reading binary. |
| `EISDIR`  | `path` is a directory. |
| `EAGAIN`  | The calling task is single-threaded and was joined by a thread that has not yet been reaped (race). |

### abi surface

```text
__NR_execve (x86_64)    = 59
__NR_execve (i386)      = 11
__NR_execve (generic)   = 221    (arm64/riscv/loongarch via asm-generic/unistd.h)
__NR_execve (powerpc)   = 11
__NR_execve (s390x)     = 11
__NR_execve (sparc)     = 59
__NR_execve (mips O32)  = 4011
__NR_execve (mips N64)  = 5057
__NR_execve (alpha)     = 59
```

### Limits

```text
MAX_ARG_STRLEN  = PAGE_SIZE * 32      /* 128 KiB on 4K pages — max length of any single argv/envp string */
MAX_ARG_STRINGS = 0x7FFFFFFF          /* 2^31-1 — max number of argv+envp entries */
BINPRM_BUF_SIZE = 256                 /* bytes of binary prefix used by binfmt detectors (#!, ELF magic, etc.) */
_STK_LIM        = 8 * 1024 * 1024     /* default RLIMIT_STACK; argv+envp combined limited to 1/4 of this */
```

### Credentials transitions

```text
AT_SECURE auxv flag set in the new program iff:
  - effective uid != real uid, OR
  - effective gid != real gid, OR
  - new credentials gained any capability the calling task did not have, OR
  - file system caps grant capability not in caller's permitted set, OR
  - LSM (selinux/apparmor/smack) flagged the transition as secure.

When AT_SECURE = 1:
  - glibc disables LD_LIBRARY_PATH, LD_PRELOAD, LD_AUDIT, LOCALDOMAIN, etc.
  - dynamic loader uses /etc/ld.so.cache only paths.
```

### no_new_privs (NNP) interaction

```text
If task.no_new_privs == 1:
  - Setuid bit on file: IGNORED (file ruid kept, no euid upgrade).
  - File capabilities: IGNORED.
  - SELinux/AppArmor domain transition: blocked unless policy permits "nnp".
  - LSM xattr exec transitions: blocked.
```

`no_new_privs` is sticky — once set via `prctl(PR_SET_NO_NEW_PRIVS, 1)` or
inherited via `clone3(CLONE_NNP)`, it propagates through ALL future
`execve` and can NEVER be cleared.

### compatibility contract

REQ-1: The syscall number is **59** on x86_64; **221** on generic-syscall
archs. ABI-stable forever.

REQ-2: `execve` is a thin wrapper around `do_execveat_common(AT_FDCWD, path,
argv, envp, 0)` — same semantics as `execveat(AT_FDCWD, path, argv, envp,
0)`.

REQ-3: On success, `execve` does NOT return. The calling task is now
executing the loaded program. Failure cases MUST leave the calling task's
mm, files, signals, creds **unchanged** (atomic-or-fail).

REQ-4: Atomicity boundary: the "point of no return" is the call to
`exec_mmap` in `begin_new_exec`. Before this, any error returns to caller
with creds/mm intact. After this, any error MUST kill the task via
`force_sigsegv(SIGSEGV)` — the task cannot continue.

REQ-5: Path resolution: `path` is resolved via `LOOKUP_FOLLOW` (follow
symlinks). The final resolved file MUST be regular, executable by caller,
and not on a `MNT_NOEXEC` mount.

REQ-6: `MAX_ARG_STRLEN` (= `PAGE_SIZE * 32`) limits each individual
argv/envp string. Per-string overflow returns `-E2BIG`.

REQ-7: `MAX_ARG_STRINGS` (= `0x7FFFFFFF`) limits the total count of argv +
envp entries. Overflow returns `-E2BIG`.

REQ-8: Total combined argv+envp space is bounded by `1/4 * RLIMIT_STACK`.
Overflow returns `-E2BIG`. Strings are copied into a fresh anonymous-vma
in the new mm at the top of the stack region.

REQ-9: `argv == NULL` and `envp == NULL` are accepted as "empty array"
(equivalent to passing `{NULL}`). This is a 5.18+ liberalization; prior
behavior was `-EFAULT`.

REQ-10: Setuid/setgid propagation:
- File has `S_ISUID` bit set AND filesystem mounted without `MNT_NOSUID`
  AND `!task.no_new_privs`: new euid = file owner uid.
- File has `S_ISGID` bit AND group-execute bit AND `!MNT_NOSUID` AND
  `!task.no_new_privs`: new egid = file owner gid.
- `MNT_NOSUID` on the file's mount: setuid/setgid silently ignored (NOT
  `-EPERM`).
- `task.no_new_privs == 1`: setuid/setgid silently ignored.

REQ-11: File capabilities (security.capability xattr) — VFS_CAP_REVISION_3
parses as `{ permitted, inheritable, effective_bit, rootid }`. Computation
of new capability sets follows `capabilities(7)`:
- `P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset)`.
- `P'(effective) = effective_bit ? P'(permitted) : empty`.
- `P'(inheritable) = P(inheritable)`.
- `P'(ambient) = (file is not privileged) ? P(ambient) & P'(permitted) & P'(inheritable) : empty`.

REQ-12: `AT_SECURE` auxv vector entry is set to `1` if the credential
transition is "secure" per the criteria in the ABI surface section.

REQ-13: `no_new_privs` is propagated to the new program and is sticky.

REQ-14: Close-on-exec processing: every fd with `O_CLOEXEC` set is closed
before the new program runs. Done BEFORE any binfmt loading commits
(closing happens in `begin_new_exec`).

REQ-15: Pending signals state:
- Sigactions reset to `SIG_DFL` for any handler not `SIG_IGN` (handlers
  point into the old mm, which is going away). `SIG_IGN` is preserved
  (per POSIX).
- Sigmask preserved.
- Pending signal queue preserved (signals follow the task, not the
  program).

REQ-16: ptrace state:
- `PTRACE_EVENT_EXEC` delivered to tracer at the point-of-no-return.
- If tracer attached: SIGTRAP stop at first instruction.

REQ-17: Thread group reduction: all OTHER threads in the calling task's
thread group are killed (SIGKILL) BEFORE the new program loads. The
calling task becomes the new sole thread (TID = TGID).

REQ-18: Robust futex list cleared. Glibc's tcache, malloc state, locale
mutex, etc. are all dropped by virtue of mm replacement.

REQ-19: Timers: per-process POSIX timers cleared; itimers (REAL/VIRTUAL/
PROF) cleared.

REQ-20: cgroup membership preserved. namespace memberships preserved
(execve does NOT cross namespaces).

REQ-21: `MNT_NOEXEC` mount: `-EACCES` on attempt.

REQ-22: ELF binary requirements: ELF magic `\x7fELF`, valid e_ident class
(ELFCLASS32 or ELFCLASS64 matching kernel), valid e_machine for arch,
e_type ET_EXEC or ET_DYN, valid program headers. PT_INTERP names the
dynamic loader; PT_LOAD segments are mmap'd at proper alignment.

REQ-23: PIE (ET_DYN with PT_INTERP) is randomly loaded at ASLR offset.
ET_EXEC with PT_INTERP is loaded at fixed address (deprecated for new
binaries but ABI-stable for existing).

REQ-24: `#!` interpreter scripts: first line begins `#!`, up to
`BINPRM_BUF_SIZE - 2` chars; first whitespace-separated token is the
interpreter path; remainder is a single argument prepended to argv. Recursion
into another `#!` is permitted up to `BINPRM_MAX_RECURSION = 5`.

REQ-25: `binfmt_misc` registered handlers can match by magic / extension;
those handlers may invoke arbitrary interpreters. `binfmt_misc` recursion
also bounded by `BINPRM_MAX_RECURSION = 5`.

REQ-26: Coredump credentials: `RLIMIT_CORE` honored; coredump suppressed
if the new program has `prctl(PR_SET_DUMPABLE, 0)`, was setuid-loaded, or
if `kernel.core_pattern` starts with `|` (piped — requires `CAP_SYS_ADMIN`).

REQ-27: Audit: AUDIT_EXECVE record emitted with the resolved path, argv,
envp (truncated by audit policy). AUDIT_PATH for the file.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `arg_str_len_bounded` | INVARIANT | Each copied string len ≤ MAX_ARG_STRLEN. |
| `arg_str_count_bounded` | INVARIANT | argc + envc ≤ MAX_ARG_STRINGS. |
| `total_args_bounded_by_rlimit` | INVARIANT | sum of arg/env bytes ≤ RLIMIT_STACK / 4. |
| `atomic_or_fail_pre_commit` | INVARIANT | Pre-commit error returns task unchanged. |
| `nnp_blocks_setuid` | INVARIANT | task.no_new_privs ⟹ euid unchanged across exec. |
| `at_secure_iff_privilege_gain` | INVARIANT | bprm.secureexec ⟺ effective creds gained privilege vs caller. |

### Layer 2: TLA+

`fs/execve.tla`:
- States: OPEN → BPRM_READ → COUNT_ARGS → COPY_ARGS → PREPARE_CREDS → DISPATCH → COMMIT → USERSPACE.
- Properties:
  - `safety_pre_commit_atomic` — error before COMMIT ⟹ no observable change.
  - `safety_post_commit_no_return` — past COMMIT ⟹ caller cannot return with old mm.
  - `safety_nnp_sticky` — no_new_privs propagates.
  - `liveness_eventual_dispatch` — every valid binary eventually reaches USERSPACE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_execveat_common` post: on Err, task state == pre-call state | `Exec::do_execveat_common` |
| `prepare_creds` post: cred deltas match capabilities(7) formula | `Exec::prepare_creds` |
| `count_strings` post: returns count or `-E2BIG` | `Exec::count_strings` |
| `copy_strings` post: each string NUL-terminated, len ≤ MAX_ARG_STRLEN | `Exec::copy_strings` |
| `begin_new_exec` post: thread group reduced to current; other tids reaped | `Exec::begin_new_exec` |

### Layer 4: Verus / Creusot functional

`execve(2)` man page + `capabilities(7)` semantic equivalence. LTP
`execve01..execve06`, `kselftests/exec/*` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`execve(2)` reinforcement:

- **Per-`no_new_privs` sticky** — defense against per-setuid-binary privilege gain after sandbox setup.
- **Per-`MNT_NOSUID` honored** — defense against per-setuid-binary on attacker-controlled mount.
- **Per-`AT_SECURE` propagation** — defense against per-LD_PRELOAD privilege-escalation.
- **Per-MAX_ARG_STRLEN + MAX_ARG_STRINGS** — defense against per-argv-bomb / per-envp-bomb OOM.
- **Per-RLIMIT_STACK/4 combined cap** — defense against per-large-argv stack-VMA explosion.
- **Per-O_CLOEXEC closed pre-load** — defense against per-fd-leak across privilege transition.
- **Per-non-SIG_IGN/DFL handler reset** — defense against per-handler-pointer-into-old-mm UAF.
- **Per-co-thread killed pre-load** — defense against per-thread-races during privilege transition.
- **Per-file-cap revision-3 rootid scoped** — defense against per-userns cap leak.
- **Per-atomic-or-fail pre-commit** — defense against per-partial-exec state corruption.
- **Per-binfmt-recursion `BINPRM_MAX_RECURSION = 5`** — defense against per-interpreter-recursion DoS.
- **Per-audit AUDIT_EXECVE** — defense against per-execution-log elision.

### grsecurity / pax surface

- **PAX_RANDKSTACK at syscall entry** — `sys_execve` randomizes kernel
  stack offset per call. Followed by full kernel-stack reset on commit
  (the new task's kernel stack is fresh).
- **PaX UDEREF on user buffers** — `path`, `argv[]`, `envp[]`, each string,
  ELF program headers all accessed via UDEREF-toggled access / PAN; a
  kernel bug that bypasses copy_from_user cannot silently read user memory.
- **PaX SEGMEXEC / NOEXEC PT_GNU_STACK** — at PT_LOAD time, the new program's
  stack vma is W^X (no PROT_EXEC); PaX-flagged binaries have additional
  per-binary `chpax` enforcement.
- **PaX ASLR** — PIE binaries get full-entropy randomized load base;
  `kernel.randomize_va_space = 2` (default in Rookery, mandatory under
  PaX).
- **GRKERNSEC_HARDEN_PTRACE** — `PTRACE_EVENT_EXEC` honors yama+grsec
  policy; ptrace across credential transition is denied.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/exe`, `/proc/<pid>/cmdline`,
  `/proc/<pid>/environ` of setuid-just-execve'd binary are restricted from
  unprivileged readers (per-uid hide-filter).
- **GRKERNSEC_CHROOT** — inside chroot:
  - setuid binaries denied by `chroot_deny_suid` (cannot exec setuid).
  - `chroot_deny_chmod` on the binary inode.
  - `chroot_caps` clears `CAP_SETUID`, `CAP_SETGID`, etc. on chroot entry,
    so even if exec passes, no privilege.
- **GRKERNSEC_BRUTE** — every `execve` failure increments a per-uid
  failure counter; sustained failures (e.g. ROP probing) trigger per-uid
  fork/exec lockout for 30 seconds.
- **GRKERNSEC_SIGNALS spoof prevention** — `PTRACE_EVENT_EXEC` SIGTRAP and
  any synchronous fault during load are kernel-sourced with the correct
  `si_code` (`TRAP_EXEC`); cannot be spoofed.
- **Per-grsec `tpe` (trusted-path-execution)** — execve of a binary
  outside `/bin /usr/bin /sbin /usr/sbin` (the TPE-trusted prefixes) by a
  non-trusted-group uid returns `-EACCES`.
- **Per-grsec `gradm` learning** — every execve recorded against the
  RBAC policy; offline policy MUST permit the (subject, object, exec)
  triple.
- **Per-grsec `chroot_findtask`** — execve does NOT leak any reference to
  outside-chroot binaries via `/proc/<other_pid>/exe`.

