# Tier-5: syscall 431 — fsconfig(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/fsopen.c (sys_fsconfig, vfs_fsconfig_locked)
  - fs/fs_context.c (vfs_parse_fs_string, vfs_parse_fs_param, vfs_get_tree, vfs_reconfigure)
  - include/uapi/linux/mount.h (FSCONFIG_*)
  - Documentation/filesystems/mount_api.rst
-->

## Summary

`fsconfig(2)` programs an `fs_context` (held in an fscontext fd from `fsopen(2)` or `fspick(2)`) one parameter at a time. Per-FSCONFIG_SET_* sets a key=value (string, binary, path, path-empty, fd, or flag). Per-FSCONFIG_CMD_CREATE instantiates the superblock by invoking `vfs_get_tree`. Per-FSCONFIG_CMD_RECONFIGURE applies the parameter set to an existing superblock (remount). Per-error feedback is appended to the fc.log for the caller to read via the fscontext fd. Critical for: structured mount-option parsing with first-class error reporting, programmatic remount, atomic superblock instantiation.

This Tier-5 covers `syscall 431 fsconfig`.

## Signature

```c
int fsconfig(int fs_fd,
             unsigned int cmd,
             const char *key,
             const void *value,
             int aux);
```

Per-x86_64-syscall-table: `__NR_fsconfig = 431`. Per-glibc: thin wrapper. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `fs_fd` | `int` | in | fscontext fd from fsopen/fspick. |
| `cmd` | `unsigned int` | in | One of FSCONFIG_SET_FLAG, SET_STRING, SET_BINARY, SET_PATH, SET_PATH_EMPTY, SET_FD, CMD_CREATE, CMD_RECONFIGURE, CMD_CREATE_EXCL (since 6.6). |
| `key` | `const char *` | in | Parameter key name (e.g. "source", "size", "uid"); NULL for CMD_CREATE / CMD_RECONFIGURE. |
| `value` | `const void *` | in | Per-cmd: NULL (FLAG); C string (STRING); raw buffer (BINARY); path (PATH/PATH_EMPTY); NULL (FD, fd given in aux); NULL (CMDs). |
| `aux` | `int` | in | Per-cmd: 0 (FLAG/STRING/CMD); buffer length (BINARY); dirfd (PATH); fd value (FD). |

## Return

- 0 on success.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EBADF | fs_fd invalid; or aux fd invalid for FSCONFIG_SET_FD. |
| EINVAL | fs_fd not fscontext; cmd unknown; key NULL when required; key invalid; value type mismatch; aux not allowed for cmd; fc.phase incompatible with cmd. |
| EOPNOTSUPP | fs does not implement parameter parser; cmd not supported for purpose (e.g. CMD_RECONFIGURE on FS_CONTEXT_FOR_MOUNT). |
| EACCES | Per-fs ACL rejection. |
| EPERM | Lacks privileges for the operation. |
| ENOMEM | Allocation failed. |
| ENAMETOOLONG | key > 256 bytes; value > PAGE_SIZE for string; aux > PAGE_SIZE for binary. |
| EFAULT | key/value pointer invalid. |
| EBUSY | fc already finalized (e.g. CMD_CREATE on already-created). |
| ELOOP | Symlink loop on PATH lookup. |
| ENOENT | PATH not found. |

## ABI surface

- Syscall number: `__NR_fsconfig = 431` (since 5.2).
- cmd enum: FSCONFIG_SET_FLAG=0, FSCONFIG_SET_STRING=1, FSCONFIG_SET_BINARY=2, FSCONFIG_SET_PATH=3, FSCONFIG_SET_PATH_EMPTY=4, FSCONFIG_SET_FD=5, FSCONFIG_CMD_CREATE=6, FSCONFIG_CMD_RECONFIGURE=7, FSCONFIG_CMD_CREATE_EXCL=8.
- value semantics by cmd:
  - FLAG: value=NULL, aux=0.
  - STRING: value=C-string (NUL-terminated, ≤ PAGE_SIZE), aux=0.
  - BINARY: value=buffer, aux=length (≤ PAGE_SIZE).
  - PATH / PATH_EMPTY: value=path string, aux=dirfd (AT_FDCWD = -100).
  - FD: value=NULL, aux=fd.
  - CMDs: value=NULL, key=NULL, aux=0.

## Compatibility contract

REQ-1: fs_fd validation:
- fget(fs_fd) returns file*.
- file.f_op == &fscontext_fops; otherwise EINVAL.
- fc = file.private_data.

REQ-2: cmd validation:
- cmd ∈ valid enum; otherwise EINVAL.

REQ-3: Per-cmd argument validation table:
| cmd | key | value | aux | fc.phase precondition |
|---|---|---|---|---|
| SET_FLAG | required (string) | NULL | 0 | Configuring |
| SET_STRING | required | C-string | 0 | Configuring |
| SET_BINARY | required | bytes | length | Configuring |
| SET_PATH | required | path | dirfd | Configuring |
| SET_PATH_EMPTY | required | path (may be "") | dirfd | Configuring |
| SET_FD | required | NULL | fd | Configuring |
| CMD_CREATE | NULL | NULL | 0 | Configuring → AwaitingMount |
| CMD_RECONFIGURE | NULL | NULL | 0 | Reconfiguring (for fspick'd fc) |
| CMD_CREATE_EXCL | NULL | NULL | 0 | Configuring → AwaitingMount (fail if sb exists shared) |

REQ-4: key copy:
- strndup_user(key, 256). Longer ⟹ ENAMETOOLONG.

REQ-5: value copy:
- STRING: strndup_user(value, PAGE_SIZE).
- BINARY: kmemdup_user(value, aux), aux bounded by PAGE_SIZE.
- PATH/PATH_EMPTY: strndup_user(value, PATH_MAX).

REQ-6: Per-FSCONFIG_SET_FD:
- aux ∈ open fd table; fget(aux); attach to fc.
- fc keeps reference; released when fc destroyed or replaced.

REQ-7: Per-FSCONFIG_SET_PATH:
- user_path_at(aux as dirfd, value, LOOKUP_FOLLOW)?;
- result stored as struct path in fc parameter slot; path_put on fc destruction.

REQ-8: Per-vfs_parse_fs_param dispatch:
- param = build_fs_parameter(cmd, key, value, aux).
- fc.ops->parse_param(fc, &param) — fs-specific parsing.
- Default fallback: vfs_parse_fs_param_source (recognizes "source" → fc.source).
- Unknown key ⟹ fs may log + EINVAL or accept-and-ignore depending on fs policy.

REQ-9: Per-FSCONFIG_CMD_CREATE:
- Lock fc.uapi_mutex.
- if fc.phase != Configuring: EBUSY.
- vfs_get_tree(fc) — invokes fc.ops->get_tree → calls fs->get_tree (or legacy mount).
- On success: fc.phase = AwaitingMount; fc.root populated (struct path).
- On failure: fc.phase unchanged; error logged.

REQ-10: Per-FSCONFIG_CMD_RECONFIGURE:
- Only valid if fc.purpose == Reconfigure (built via fspick).
- vfs_reconfigure(fc) — invokes fc.ops->reconfigure → fs->reconfigure_fs (handles remount).
- Applies fc.s_flags / mount parameters to existing super_block.
- Atomicity per-fs.

REQ-11: Per-FSCONFIG_CMD_CREATE_EXCL (since 6.6):
- Same as CMD_CREATE but if vfs_get_tree finds an existing shared super_block matching, fails with EBUSY.

REQ-12: Per-capability:
- ns_capable(fc.user_ns, CAP_SYS_ADMIN) — using fc.cred.
- Some fs (e.g. proc) may relax this for their own params.

REQ-13: Per-error logging:
- Errors append to fc.log (bounded, ring-style).
- Returned errno is the canonical one; userspace reads fc.log via fscontext fd for human-readable detail.

REQ-14: Per-mutual-exclusion:
- fc.uapi_mutex held during each fsconfig call to serialize parameter writes and prevent concurrent CMD_CREATE.

REQ-15: Per-aux ≤ PAGE_SIZE for BINARY: enforced regardless of fs policy.

REQ-16: Per-fc.cred used for path lookups in SET_PATH (not current's cred) — preserves opener authorization.

## Acceptance Criteria

- [ ] AC-1: fsopen → fsconfig(SET_STRING, "source", "tmpfs", 0) → fsconfig(CMD_CREATE): succeeds; fc.phase == AwaitingMount.
- [ ] AC-2: fsconfig(SET_FLAG, "ro", NULL, 0) on ext4 fc: sets SB_RDONLY.
- [ ] AC-3: fsconfig(SET_PATH, "lowerdir", "/a", AT_FDCWD) on overlay fc: resolves path and stores.
- [ ] AC-4: fsconfig(SET_BINARY, "blob", buf, len > PAGE_SIZE): ENAMETOOLONG.
- [ ] AC-5: fsconfig(SET_FD, "exfat-image", NULL, fd): attaches fd to fc.
- [ ] AC-6: fsconfig(CMD_CREATE) twice: second call EBUSY.
- [ ] AC-7: fsconfig with unknown cmd: EINVAL.
- [ ] AC-8: fsconfig(SET_STRING, "no-such-key", "v", 0) on strict fs: EINVAL + log entry.
- [ ] AC-9: Read on fscontext fd after AC-8: returns log line.
- [ ] AC-10: fs_fd not fscontext: EINVAL.
- [ ] AC-11: fsconfig(SET_PATH, ..., bad-dirfd, ...): EBADF.
- [ ] AC-12: fsconfig(CMD_RECONFIGURE) on FS_CONTEXT_FOR_MOUNT fc (from fsopen): EOPNOTSUPP.
- [ ] AC-13: Non-CAP_SYS_ADMIN: EPERM.
- [ ] AC-14: key > 256 bytes: ENAMETOOLONG.

## Architecture

```
Fsconfig::sys_fsconfig(fs_fd: i32, cmd: u32, key: UserPtr<u8>,
                      value: UserPtr<u8>, aux: i32) -> Result<i32, Errno>
```

1. let file = fget(fs_fd).ok_or(EBADF)?;
2. let fc  = fscontext_from_file(&file).ok_or(EINVAL)?;
3. /* Acquire fc lock */
4. let _g = fc.uapi_mutex.lock();
5. /* Capability via fc.cred */
6. if !ns_capable_cred(&fc.cred, fc.user_ns, CAP_SYS_ADMIN) { drop_g; fput(file); return Err(EPERM); }
7. /* Phase compatibility */
8. require_phase_for_cmd(&fc.phase, cmd)?;
9. /* Dispatch */
10. let result = match cmd {
      FSCONFIG_SET_FLAG       => do_set_flag(&fc, key, value, aux),
      FSCONFIG_SET_STRING     => do_set_string(&fc, key, value, aux),
      FSCONFIG_SET_BINARY     => do_set_binary(&fc, key, value, aux),
      FSCONFIG_SET_PATH       => do_set_path(&fc, key, value, aux, follow=true, allow_empty=false),
      FSCONFIG_SET_PATH_EMPTY => do_set_path(&fc, key, value, aux, follow=true, allow_empty=true),
      FSCONFIG_SET_FD         => do_set_fd(&fc, key, value, aux),
      FSCONFIG_CMD_CREATE     => do_cmd_create(&fc, /*excl=*/false),
      FSCONFIG_CMD_RECONFIGURE=> do_cmd_reconfigure(&fc),
      FSCONFIG_CMD_CREATE_EXCL=> do_cmd_create(&fc, /*excl=*/true),
      _ => Err(EINVAL),
    };
11. drop(_g); fput(file);
12. result.
```

`do_set_string(fc, key, value, aux)`:
1. if aux != 0: return Err(EINVAL).
2. if key.is_null() || value.is_null(): return Err(EINVAL).
3. let k = strndup_user(key, 256)?;
4. let v = strndup_user(value, PAGE_SIZE)?;
5. let param = FsParameter::String { key: k, val: v };
6. vfs_parse_fs_param(fc, &param).

`do_set_path(fc, key, value, aux, follow, allow_empty)`:
1. let k = strndup_user(key, 256)?;
2. let p = strndup_user(value, PATH_MAX)?;
3. let lookup_flags = if follow { LOOKUP_FOLLOW } else { 0 } | if allow_empty { LOOKUP_EMPTY } else { 0 };
4. let path = user_path_at_with_cred(&fc.cred, aux, &p, lookup_flags)?;
5. let param = FsParameter::Path { key: k, path };
6. let res = vfs_parse_fs_param(fc, &param);
7. if res.is_err() { path_put(&param.path); } else { /* param consumed */ }
8. res.

`do_cmd_create(fc, excl)`:
1. if fc.phase != FsContextPhase::Configuring: return Err(EBUSY).
2. fc.phase_transition_pending = true;
3. let r = vfs_get_tree(fc, excl);
4. if r.is_ok(): fc.phase = FsContextPhase::AwaitingMount.
5. else: fc.phase remains Configuring; error appended to fc.log.
6. r.

`do_cmd_reconfigure(fc)`:
1. if fc.purpose != FsContextPurpose::Reconfigure: return Err(EOPNOTSUPP).
2. vfs_reconfigure(fc).

`vfs_parse_fs_param(fc, param)`:
1. /* fs-specific */
2. if let Some(parse) = fc.ops.parse_param: parse(fc, param).
3. /* common fallback */
4. else: vfs_parse_fs_param_source(fc, param).
5. /* log error context */
6. on Err: fc.log.push_error("...", param.key).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fs_fd_validated` | INVARIANT | per-sys_fsconfig: file must be fscontext. |
| `cmd_valid` | INVARIANT | per-sys_fsconfig: cmd ∈ enum. |
| `arg_shape_per_cmd` | INVARIANT | per-sys_fsconfig: key/value/aux nullness matches cmd table. |
| `phase_precondition` | INVARIANT | per-sys_fsconfig: fc.phase compatible with cmd. |
| `caps_via_fc_cred` | INVARIANT | per-sys_fsconfig: ns_capable uses fc.cred. |
| `key_bounded` | INVARIANT | per-do_set_*: key ≤ 256. |
| `value_bounded` | INVARIANT | per-do_set_string/binary: value ≤ PAGE_SIZE. |
| `path_bounded` | INVARIANT | per-do_set_path: path ≤ PATH_MAX. |
| `cmd_create_oneshot` | INVARIANT | per-do_cmd_create: phase transition irreversible. |
| `uapi_mutex_held` | INVARIANT | per-sys_fsconfig: fc.uapi_mutex held across cmd dispatch. |

### Layer 2: TLA+

`uapi/fsconfig.tla`:
- States: fc.phase, fc.parameters map, fc.log buffer.
- Properties:
  - `safety_phase_progression` — per-fc: Configuring → AwaitingMount monotonic; Reconfigure terminal.
  - `safety_parameter_locked` — per-CMD_CREATE/RECONFIGURE: no SET_* possible thereafter.
  - `safety_cmd_arg_shape` — per-cmd: argument validity invariant holds.
  - `safety_log_bounded` — per-fc.log: ≤ FC_LOG_MAX bytes.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fsconfig::sys_fsconfig` post: success ⟹ either parameter recorded or phase advanced | `Fsconfig::sys_fsconfig` |
| `do_cmd_create` post: success ⟹ fc.phase == AwaitingMount ∧ fc.root populated | `do_cmd_create` |
| `do_cmd_reconfigure` post: success ⟹ underlying super_block reflects new flags | `do_cmd_reconfigure` |
| `do_set_path` post: success ⟹ path_put paired (consumed) or released on err | `do_set_path` |
| `do_set_fd` post: aux fd ref count balanced | `do_set_fd` |
| `vfs_parse_fs_param` post: Err ⟹ fc.log has matching error entry | `vfs_parse_fs_param` |

### Layer 4: Verus/Creusot functional

Per-fsconfig semantic equivalence with upstream `sys_fsconfig` + `vfs_parse_fs_param` + `vfs_get_tree`/`vfs_reconfigure`: per-Documentation/filesystems/mount_api.rst.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fsconfig-syscall reinforcement:

- **Per-fc.cred capability source** — defense against per-credential-stripping via fd-handoff.
- **Per-uapi_mutex serialization** — defense against per-concurrent-parameter race.
- **Per-key bound 256 / value bound PAGE_SIZE / path bound PATH_MAX** — defense against per-overlong DoS.
- **Per-cmd argument-shape table strict** — defense against per-confused-cmd parameter smuggling.
- **Per-phase-precondition enforcement** — defense against per-out-of-order programming.
- **Per-CMD_CREATE one-shot phase transition** — defense against per-double-instantiation race.
- **Per-CMD_RECONFIGURE only for fspick fc** — defense against per-wrong-purpose remount-as-mount.
- **Per-SET_PATH uses fc.cred for lookup** — defense against per-current-task-privilege escalation by fd-passer.
- **Per-fs.parse_param fallback to source-only** — defense against per-unknown-key bleed into kernel.
- **Per-fc.log bounded ring** — defense against per-log-flood DoS.
- **Per-PAX UDEREF on copy_from_user of key/value** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-fsconfig.
- **Per-aux ≤ PAGE_SIZE for BINARY** — defense against per-payload-bomb.
- **Per-EBUSY on phase mismatch (no silent fallthrough)** — defense against per-stale-state programming.
- **Per-GRKERNSEC_CHROOT_MOUNT applies (chroot blocked from fsconfig)** — defense against per-chroot-escape via mount-API plumbing.
- **Per-fd attached via SET_FD ref-counted** — defense against per-UAF on dup-and-drop sequences.

## Grsecurity/PaX-style Reinforcement

This syscall inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `fsconfig(2)` key/value/aux buffers copy_from_user bounded against fs-context parameter caps; STRING / BINARY / PATH / PATH_EMPTY / FD payload sizes validated per command before allocation.
- **PAX_KERNEXEC** — `do_fsconfig`, `vfs_parse_fs_string`, `vfs_parse_fs_param`, per-FS `parser` tables, and `fs_context_operations.parse_param` reside in RX `.text`; FS parser dispatch via kCFI-signed indirection.
- **PAX_RANDKSTACK** — kstack-offset randomization on syscall entry; defense against ROP-via-fsconfig (REQ-PAX_RANDKSTACK).
- **PAX_REFCOUNT** — saturating refcount on `struct fs_context` (`fc->users`, `fc->root`) and per-fd `SET_FD` attached files; defense against UAF on dup-and-drop sequences.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `fs_context` param scratch (key/value buffers may carry credentials for cifs/9p/nfs) and per-FSCONFIG-command staging.
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on every key/value copy_from_user; FSCONFIG_CMD_* enum strictly validated against canonical set.
- **PAX_RAP / kCFI** — `fs_context_operations` (`parse_param`, `parse_monolithic`, `get_tree`, `reconfigure`) and per-FS `init_fs_context` dispatched via kCFI-signed indirect calls.
- **GRKERNSEC_HIDESYM** — `fs_context`, `vfs_parse_fs_*` symbols masked from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — parameter-parse failure / context-state-mismatch diagnostics restricted to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** — `fsconfig(2)` mutation gated on `fs_context` owner's CAP_SYS_ADMIN in init_user_ns (or per-mountns equivalent); FSCONFIG_CMD_CREATE / CMD_RECONFIGURE never fall through on capability miss.
- **FSCONFIG_* enum strict-validation** — out-of-range `cmd` rejected with -EINVAL pre-dispatch; STRING/BINARY length-discrimination enforced; aux ≤ PAGE_SIZE for BINARY.
- **GRKERNSEC_CHROOT_MOUNT** — chrooted task blocked from fsconfig; defense against chroot-escape via mount-API plumbing.
- **`-EBUSY` on phase mismatch (no silent fallthrough)** — defense against stale-state programming where attacker re-orders CREATE/RECONFIGURE.

Per-syscall rationale: `fsconfig(2)` is the parameter-configuration arm of the new mount API; an attacker controlling the value blob of `cifs` / `nfs` / `9p` key=value can inject credentials or path-references into a kernel `fs_context` that survives across the userspace boundary. PAX_USERCOPY + capability gate + per-FS parser kCFI + MEMORY_SANITIZE of credential-bearing buffers are the layered defense.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fsopen(2)` context creation (covered in `fsopen.md`).
- `fsmount(2)` materialization (covered in `fsmount.md`).
- `fspick(2)` reconfigure-context creation (covered separately).
- Per-fs parse_param implementations (covered in fs/<name>/Tier-3).
- Implementation code.
