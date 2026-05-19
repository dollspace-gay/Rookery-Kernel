# Tier-5: syscall 430 — fsopen(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/fsopen.c (sys_fsopen, fscontext_alloc_log, fscontext_create_fd)
  - fs/fs_context.c (fs_context_for_mount)
  - include/uapi/linux/mount.h (FSOPEN_CLOEXEC)
  - Documentation/filesystems/mount_api.rst
-->

## Summary

`fsopen(2)` creates a **filesystem context** for a named filesystem type and returns an `fscontext` file descriptor. The fd represents a half-built mount: subsequent `fsconfig(2)` calls set source, options, and key=value parameters; `fsconfig(FSCONFIG_CMD_CREATE)` materializes the superblock; `fsmount(2)` then converts the context into a detached mount fd. Per-FSOPEN_CLOEXEC sets O_CLOEXEC on the fd. Per-context retains a structured log of warning/error messages readable via `read(fd)`. Critical for: programmatic mount option parsing with explicit error feedback, container runtime mount plumbing, mount-via-fd composition.

This Tier-5 covers `syscall 430 fsopen`.

## Signature

```c
int fsopen(const char *fsname, unsigned int flags);
```

Per-x86_64-syscall-table: `__NR_fsopen = 430`. Per-glibc: thin wrapper or direct `syscall(__NR_fsopen, ...)`. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `fsname` | `const char *` | in | Registered filesystem type name (e.g. "ext4", "tmpfs", "overlay"). |
| `flags` | `unsigned int` | in | 0 or FSOPEN_CLOEXEC (0x1). |

## Return

- ≥ 0: new fscontext fd on success.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_ADMIN in mnt_ns.user_ns. |
| EINVAL | `flags` has bits other than FSOPEN_CLOEXEC. |
| ENODEV | `fsname` not a registered filesystem. |
| EFAULT | `fsname` invalid pointer. |
| ENAMETOOLONG | `fsname` longer than PAGE_SIZE. |
| ENOMEM | Allocation failed. |
| EMFILE | Process fd table full. |
| ENFILE | System-wide file limit. |

## ABI surface

- Syscall number: `__NR_fsopen = 430` (since 5.2).
- Flags: FSOPEN_CLOEXEC = 0x1; other bits reserved.
- Returned fd: `fscontext` anon-inode, with `f_op = fscontext_fops` (poll/read for log; ioctl-free).
- Lifecycle: fsopen → fsconfig (multiple) → fsconfig(FSCONFIG_CMD_CREATE) → fsmount → move_mount → close(fscontext_fd).
- The fscontext fd is **distinct** from the future mount fd produced by fsmount.

## Compatibility contract

REQ-1: Flag validation:
- flags & ~FSOPEN_CLOEXEC ⟹ EINVAL.

REQ-2: Capability:
- ns_capable(current.nsproxy.mnt_ns.user_ns, CAP_SYS_ADMIN).

REQ-3: fsname copy:
- strndup_user(fsname, PAGE_SIZE).
- Bound by PAGE_SIZE - 1 chars.

REQ-4: fs_type lookup:
- type = get_fs_type(fsname).
- NULL ⟹ ENODEV.
- module-on-demand permitted (request_module).

REQ-5: fs_context allocation:
- fc = fs_context_for_mount(type, sb_flags = 0).
- fc.purpose = FS_CONTEXT_FOR_MOUNT.
- fc.cred = get_current_cred().
- fc.user_ns = mnt_ns.user_ns (or filesystem's preferred owner).
- fc.fs_private set by fs->init_fs_context(fc).

REQ-6: fs_context log allocation:
- fc.log = alloc_fc_log() — bounded circular buffer (default 8KiB).
- Holds informational, warning, error messages from fsconfig validation.

REQ-7: fd allocation:
- get_unused_fd_flags(O_RDWR | (flags & FSOPEN_CLOEXEC ? O_CLOEXEC : 0)).

REQ-8: file allocation:
- file = anon_inode_getfile("fscontext", &fscontext_fops, fc, O_RDWR).

REQ-9: fd install:
- fd_install(fd, file) — fd valid from this point.

REQ-10: fs_type put on error:
- All error paths between get_fs_type and fd_install must call put_filesystem(type).
- After fd_install: fc owns the type ref and releases on its destruction.

REQ-11: Per-FSOPEN_CLOEXEC: sets FD_CLOEXEC on the descriptor.

REQ-12: Per-fscontext fops:
- read: copies log entries (size-prefixed) to user buffer.
- poll: POLLIN when log has unread entries.
- llseek: not supported (-ESPIPE).
- write: not supported (-EINVAL).

REQ-13: Per-fc lifetime:
- fc.ref starts at 1 (the file).
- Increment on fsconfig calls (per call duration), fsmount call.
- Decrement when file closed.

REQ-14: Per-permission inheritance:
- fc.cred snapshot at fsopen time; fsconfig/fsmount operate under that cred for ns_capable checks (so opening fscontext as root then handing fd to lesser process is still authorized by the original opener).

## Acceptance Criteria

- [ ] AC-1: fsopen("tmpfs", 0): returns valid fd; /proc/self/fdinfo shows "fscontext".
- [ ] AC-2: fsopen("tmpfs", FSOPEN_CLOEXEC): fd has FD_CLOEXEC.
- [ ] AC-3: fsopen("no-such-fs", 0): ENODEV.
- [ ] AC-4: fsopen("ext4", 0x2): EINVAL.
- [ ] AC-5: Non-CAP_SYS_ADMIN: EPERM.
- [ ] AC-6: fsname = NULL: EFAULT.
- [ ] AC-7: fsname > PAGE_SIZE: ENAMETOOLONG.
- [ ] AC-8: read(fd, buf, 4096) after a deliberate fsconfig error: returns one log line.
- [ ] AC-9: poll(fd, POLLIN): triggers after fsconfig pushes a log line.
- [ ] AC-10: write(fd, ...): EINVAL.
- [ ] AC-11: lseek(fd, 0, SEEK_SET): ESPIPE.
- [ ] AC-12: Process fd table full: EMFILE; fs_context not leaked.

## Architecture

```
Fsopen::sys_fsopen(fsname: UserPtr<u8>, flags: u32) -> Result<i32, Errno>
```

1. /* Validate flags */
2. if flags & !FSOPEN_CLOEXEC != 0: return Err(EINVAL).
3. /* Capability */
4. let mnt_ns = current().nsproxy.mnt_ns;
5. if !ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN): return Err(EPERM).
6. /* Copy name */
7. let name = strndup_user(fsname, PAGE_SIZE)?;
8. /* Resolve type (with module load) */
9. let fs_type = FsRegistry::get_fs_type(&name).ok_or(ENODEV)?;
10. /* Build fs_context */
11. let fc = match FsContext::for_mount(fs_type.clone(), 0) {
        Ok(fc) => fc,
        Err(e) => { FsRegistry::put_fs_type(fs_type); return Err(e); }
      };
12. /* Allocate log */
13. fc.log = FcLog::alloc()?;
14. /* fd + file */
15. let cloexec = flags & FSOPEN_CLOEXEC != 0;
16. let fd = get_unused_fd_flags(O_RDWR | if cloexec { O_CLOEXEC } else { 0 })?;
17. let file = anon_inode_getfile("fscontext", &FSCONTEXT_FOPS, Arc::clone(&fc), O_RDWR)?;
18. fd_install(fd, file);
19. /* fc now owned by file */
20. Ok(fd as i32).
```

`FscontextFops`:
- read = fscontext_read       // drains log into user buffer
- poll = fscontext_poll       // POLLIN if log non-empty
- release = fscontext_release // drops fc; put_filesystem; clear log
- llseek = no_llseek
- write = NULL (returns EINVAL)

`FsContext::for_mount(fs_type, sb_flags)`:
1. fc = FsContext::alloc()?;
2. fc.fs_type = fs_type;
3. fc.purpose = FsContextPurpose::Mount;
4. fc.sb_flags = sb_flags;
5. fc.cred = get_current_cred();
6. fc.user_ns = current().nsproxy.mnt_ns.user_ns;
7. fc.ref = AtomicRefcount::new(1);
8. if let Some(init) = fs_type.init_fs_context: init(&mut fc)?;
9. Ok(fc).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | per-sys_fsopen: only FSOPEN_CLOEXEC accepted. |
| `caps_required` | INVARIANT | per-sys_fsopen: CAP_SYS_ADMIN before lookup. |
| `fs_type_ref_balanced` | INVARIANT | per-sys_fsopen: error paths put_filesystem; success transfers ref to fc. |
| `fd_install_after_file_alloc` | INVARIANT | per-sys_fsopen: file allocated before fd_install; fd reservation paired. |
| `cloexec_propagated` | INVARIANT | per-sys_fsopen: FSOPEN_CLOEXEC ⟹ O_CLOEXEC on fd. |
| `fc_log_bounded` | INVARIANT | per-FcLog::alloc: capacity ≤ 8KiB. |
| `name_bound_by_page` | INVARIANT | per-strndup_user: ≤ PAGE_SIZE-1. |

### Layer 2: TLA+

`uapi/fsopen.tla`:
- States: fs_type registry, fs_context lifecycle, fd table.
- Properties:
  - `safety_caps` — per-call: CAP_SYS_ADMIN enforced.
  - `safety_refcount_balance` — per-call: fs_type and fc refs balanced (init=1, drop on close).
  - `safety_fd_observability` — per-success: fd visible iff fd_install completed.
  - `safety_cloexec_fidelity` — per-FSOPEN_CLOEXEC: O_CLOEXEC set.
  - `liveness_terminates` — per-call: returns fd or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fsopen::sys_fsopen` post: success ⟹ current.files.fdt[fd] points to fscontext file | `Fsopen::sys_fsopen` |
| `FsContext::for_mount` post: fc.purpose == Mount; fc.cred == current.cred; fc.user_ns == mnt_ns.user_ns | `FsContext::for_mount` |
| `FsContext::for_mount` post: fc.ref == 1 | `FsContext::for_mount` |
| `FcLog::alloc` post: capacity ≤ FC_LOG_MAX | `FcLog::alloc` |

### Layer 4: Verus/Creusot functional

Per-fsopen semantic equivalence with upstream `sys_fsopen` + `fs_context_for_mount` + `fscontext_create_fd`: per-Documentation/filesystems/mount_api.rst.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fsopen-syscall reinforcement:

- **Per-CAP_SYS_ADMIN in mnt_ns.user_ns** — defense against per-unprivileged context creation.
- **Per-fsname bound to PAGE_SIZE** — defense against per-overlong-name OOM.
- **Per-fs_type get/put strict** — defense against per-module-unload race.
- **Per-fc_log bounded (8KiB)** — defense against per-log-flood DoS.
- **Per-anon_inode_getfile pairs with fd reservation** — defense against per-half-open file leak.
- **Per-FSOPEN_CLOEXEC honored** — defense against per-exec-leak of context fd.
- **Per-fscontext file ops restricted (no write, no lseek)** — defense against per-state-mutation by handle holder.
- **Per-fc.cred snapshot at fsopen** — defense against per-credential-swap race during config.
- **Per-PAX UDEREF on fsname copy_from_user** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-fsopen.
- **Per-fc ref counter atomic** — defense against per-UAF on close-vs-fsconfig race.
- **Per-GRKERNSEC_CHROOT_MOUNT: chrooted caller blocked** — defense against per-chroot-escape via filesystem creation.
- **Per-fc.fs_private isolated per-context** — defense against per-cross-context-state leak.
- **Per-init_fs_context-error path: fc fully torn down** — defense against per-half-initialized context exposure.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fsconfig(2)` parameter setting (covered in `fsconfig.md`).
- `fsmount(2)` materialization (covered in `fsmount.md`).
- `move_mount(2)` attach (covered separately).
- Per-fs init_fs_context implementations (covered in fs/<name>/Tier-3).
- Implementation code.
