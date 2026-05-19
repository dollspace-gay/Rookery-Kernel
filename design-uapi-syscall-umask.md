---
title: "Tier-5 syscall: umask(2) — syscall 95"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`umask(2)` sets the calling process's "file mode creation mask" — a
bitmap of mode bits that the kernel will CLEAR from the mode passed
to file-creating syscalls (`open(O_CREAT)`, `creat`, `mkdir`, `mknod`,
`mkfifo`, `socket-bind`). The previous umask value is returned.

The umask is stored in `current->fs->umask` and (per the POSIX rule)
is **shared by all threads that share `fs_struct`** — i.e. POSIX
threads spawned with `CLONE_FS` see each other's umask changes,
unlike per-thread state. This is a long-standing source of bugs: a
worker thread calling `umask` mid-flight affects all peers.

The mask is applied as `final_mode = requested_mode & ~umask`. Only
the low 9 bits (0777) of the umask have effect on rwx; the setuid /
setgid / sticky bits are also maskable. The umask never affects
chmod / fchmod / fchmodat — those operate on existing files and
bypass the mask.

`umask` is trivially safe (no permission check, no failure mode
short of the syscall never being entered). It returns `int`, not
`mode_t` — but only the low 12 bits of the argument and return are
significant.

Critical for: shell pipelines (`umask 077; touch f`), daemon
hardening (set 027 / 077 before opening config), container init,
secure-file-creation primitives.

### Acceptance Criteria

- [ ] AC-1: `umask(022)` returns previous mask; subsequent `open("f", O_CREAT, 0666)` creates file with mode `0644`.
- [ ] AC-2: `umask(0)` then `open("f", O_CREAT, 0666)`: file mode `0666`.
- [ ] AC-3: `umask(077)` then `mkdir("d", 0777)`: directory mode `0700`.
- [ ] AC-4: `umask(0xFFFFFFFF)` (mask all high bits): stored as `0777`; high bits silently dropped.
- [ ] AC-5: `umask` does NOT affect `chmod(f, 0666)` — file ends up at `0666` regardless of mask.
- [ ] AC-6: Threads with CLONE_FS: thread A calls `umask(077)`; thread B sees it.
- [ ] AC-7: Threads without CLONE_FS: independent umasks observable.
- [ ] AC-8: `/proc/self/status | grep Umask:` reflects current value.
- [ ] AC-9: After execve, umask preserved.
- [ ] AC-10: `bind` of UNIX socket with default umask 022: socket mode `0644`.
- [ ] AC-11: Return value matches the previous mask exactly.
- [ ] AC-12: `umask` invoked while another thread is in `open(O_CREAT)`: fs.lock provides linearization (mask change either fully precedes or fully follows the open's mask read).

### Architecture

```rust
#[syscall(nr = 95, abi = "sysv")]
pub fn sys_umask(mask: u32) -> isize {
    Fs::do_umask(mask & 0o777)
}
```

`Fs::do_umask(new_mask) -> isize`:
1. let fs = current.fs;
2. spin_lock(&fs.lock);
3. let old = fs.umask;
4. fs.umask = new_mask & 0o777;
5. spin_unlock(&fs.lock);
6. old as isize

Mask application elsewhere (file creation):

```rust
fn apply_umask(requested: u16) -> u16 {
    let fs = current.fs;
    let mask = atomic_or_locked_read(&fs.umask);
    requested & !mask
}
```

In practice the application happens inline in:
- `Fs::do_open(O_CREAT)` → `vfs_create`.
- `Fs::do_mkdir` → `vfs_mkdir`.
- `Fs::do_mknod` → `vfs_mknod`.
- UNIX socket `bind`.

### Out of Scope

- File-creation syscalls (`open`, `mkdir`, `mknod`, `mkfifo`, `bind`) — each separate Tier-5.
- `chmod(2)` / `chown(2)` (separate Tier-5).
- POSIX `prctl(PR_GET_FS_UMASK)` (separate Tier-5 if added).
- Implementation code.

### signature

```c
mode_t umask(mode_t mask);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `mask` | `mode_t` | in | New file mode creation mask. Only low 9 bits (rwxrwxrwx) honored on Linux. Upper bits silently ignored. |

### return value

| Value | Meaning |
|---|---|
| `mode_t` | Previous umask value. Never fails. |

### errors

`umask(2)` cannot fail — it has no error return.

### abi surface

```text
__NR_umask (x86_64)  = 95
__NR_umask (i386)    = 60
__NR_umask (arm64)   = 166
__NR_umask (riscv)   = 166
__NR_umask (generic) = 166
```

### compatibility contract

REQ-1: Syscall number is **95** on x86_64; **166** on generic. ABI-stable.

REQ-2: Per-mask: `current->fs->umask = mask & 0777` — only the low 9
bits used (Linux historical choice). Some BSDs allow `S_ISVTX/SUID/SGID`
in the mask; Linux narrowed to 0777.

REQ-3: Per-return: previous `current->fs->umask` returned as positive
integer in `r/a0`.

REQ-4: Per-no-permission-check: any process may set any umask. There
is no DAC, no capability, no LSM hook.

REQ-5: Per-no-failure: the syscall always succeeds; cannot return a
negative value.

REQ-6: Per-fs_struct-shared: tasks created via `CLONE_FS` share
`fs_struct` and observe each other's umask. Threads created without
`CLONE_FS` have independent umask.

REQ-7: Per-write-under-fs.lock: store via
`spin_lock(&fs.lock); fs.umask = new; spin_unlock(&fs.lock)` to ensure
concurrent readers (during open) see a consistent value.

REQ-8: Per-application: file-creating syscalls mask via:
`mode_for_inode = arg.mode & ~current.fs.umask`. Applied at the VFS
layer right before `inode_init_owner` / `setattr_should_drop_suidgid`.

REQ-9: Per-no-effect on chmod: chmod / fchmod / fchmodat / chown do
NOT consult umask.

REQ-10: Per-creat path: `open(O_CREAT)` includes umask masking only
when the file is being created (not when O_CREAT is set but file
already exists).

REQ-11: Per-mkdir: `mkdir(path, mode)` writes `(mode & ~umask) & 0777`.

REQ-12: Per-mknod / mkfifo: `mknod(path, mode, dev)` similarly masks.

REQ-13: Per-bind: `bind(unix_sock, ...)` creates a filesystem entry
whose mode is `0666 & ~umask` (Linux UNIX-domain socket convention).

REQ-14: Per-/proc/self/status: `Umask:` field reflects current value
(in octal).

REQ-15: Per-fork: child inherits parent's umask (via fs_struct copy or
CLONE_FS-share).

REQ-16: Per-exec: umask preserved across execve — not reset.

REQ-17: Per-no-audit-record: AUDIT_SYSCALL captures umask call with
old + new value; informational.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mask_low_9_bits` | INVARIANT | stored mask & ~0o777 == 0. |
| `no_failure_path` | INVARIANT | sys_umask never returns negative. |
| `atomic_swap` | INVARIANT | mask read/write under fs.lock. |
| `applied_at_creation_only` | INVARIANT | umask consulted at vfs_create/mkdir/mknod; never at chmod. |
| `shared_under_clone_fs` | INVARIANT | tasks sharing fs_struct observe identical umask. |

### Layer 2: TLA+

`fs/umask.tla`:
- States: ENTRY → READ_LOCK → STORE → READ_OLD → RETURN.
- Properties:
  - `safety_low_9_bits_only` — upper bits never persisted.
  - `safety_atomic_swap` — concurrent open vs umask sees coherent value.
  - `safety_share_under_clone_fs` — fs_struct sharing implies umask sharing.
  - `safety_no_chmod_effect` — chmod path never masks.
  - `liveness_returns` — every umask returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_umask` post: returns old; fs.umask == new & 0o777 | `sys_umask` |
| `do_umask` post: atomic swap under fs.lock | `Fs::do_umask` |
| `apply_umask` post: requested & ~current.fs.umask | `Fs::apply_umask` |

### Layer 4: Verus / Creusot functional

Per-`umask(2)` man-page, per-POSIX, per-Linux kernel/sys.c, LTP `umask01..03`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`umask(2)` reinforcement:

- **Per-mask-low-9-bits** — defense against per-stale-suid-mask
  smuggling via mask argument.
- **Per-fs.lock atomic** — defense against per-mask-tearing during
  concurrent creation.
- **Per-no-permission-required** — by design; documented; not a
  weakness because umask only RESTRICTS, never EXPANDS, mode bits.
- **Per-fs_struct shared-or-not** — defense against per-thread
  policy confusion (documented sharing under CLONE_FS).
- **Per-creation-time-only application** — defense against per-chmod
  bypass attempts.

### grsecurity / pax-style reinforcement

- **PaX UDEREF** — umask has no user pointer argument; no UDEREF
  exposure.
- **CAP_FOWNER / CHOWN / SETID strict (cross-reference)** — umask does
  not interact with capabilities; the capability-strict policy applies
  to the create syscalls that consume the mask (chmod, chown, mknod).
- **GRKERNSEC_SUIDDUMP (cross-reference)** — umask itself is not logged;
  but a subsequent file-creation that yields a setuid binary (mode &
  ~umask still has S_ISUID set) is logged with the effective mask in
  the audit record. Grsec annotates the SUIDDUMP log line with the
  active umask at create time so the cause-chain is reconstructable.
- **GRKERNSEC_CHROOT_CHMOD (cross-reference)** — chrooted process
  setting a permissive umask (e.g. 0) does NOT bypass chroot-chmod
  enforcement; the create-syscall layer still applies the chroot-chmod
  policy on the resulting mode bits.
- **GRKERNSEC_TPE on setuid-mode set (cross-reference)** — umask of 0
  combined with `open(O_CREAT, mode=04755)` is denied by TPE at the
  create syscall, not at umask. The umask call itself is unrestricted.
- **File-cap-strip on chown (cross-reference)** — N/A to umask;
  chown is the operative syscall.
- **RLIMIT_FSIZE on truncate-grow (cross-reference)** — N/A to umask.
- **GRKERNSEC_DMESG** — umask transitions are NOT logged to dmesg by
  default (high frequency, low security value); audit subsystem captures
  the call in normal AUDIT_SYSCALL records.
- **GRKERNSEC_HIDESYM** — N/A; umask logs no pointers.
- **PaX KERNEXEC** — `sys_umask` and the masked-create call sites
  (vfs_create, vfs_mkdir, vfs_mknod) in read-only-after-init text.
- **PaX RANDSTRUCT on `struct fs_struct`** — randomized layout; corruption
  cannot deterministically set `umask` field to 0 to enable suid-bit
  smuggling at subsequent creates.
- **gradm policy: minimum-umask enforcement** — RBAC may enforce a
  MINIMUM mask (e.g. `0022`) for non-trusted subjects: `fs.umask |=
  policy_min_umask` on every store. This closes the "umask(0) then
  open(_, _, 0666) world-writable config" pattern in policy-restricted
  subjects.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/status` Umask field
  hidden across uid boundaries unless caller has CAP_DAC_READ_SEARCH;
  blocks per-process umask reconnaissance.
- **PaX MEMORY_SANITIZE** — fs_struct allocations zero-initialized on
  fork-without-CLONE_FS; no umask leak from previous owner.

