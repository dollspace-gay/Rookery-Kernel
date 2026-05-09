---
title: "Tier-3: fs/vfs/fs-context — modern mount API"
tags: ["design-doc", "tier-3", "fs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the modern mount API: a userspace-visible filesystem-context machinery (`struct fs_context`) replacing the legacy single-call `mount(2)` syscall with a multi-step flow (`fsopen` → `fsconfig` → `fsmount` → `move_mount`). Owns the `fs_context` lifecycle, parameter-parsing helpers (fs_parser), the `fsopen(2)` family of syscalls, error reporting (`/fs/fsfd/N`), and the per-FS init_fs_context callback that filesystems implement to plug into the modern API.

Sub-tier-3 of `fs/vfs/00-overview.md`. Modernizes mount() with: per-source-of-truth string-typed config (no opaque mount-data blob); fsfd-based error reporting (vs. dmesg-only); a standardized parameter-parsing framework; multi-step staging that can be aborted cleanly.

### Requirements

- REQ-1: All modern-mount syscalls (`fsopen`, `fsconfig`, `fsmount`, `fspick`, `move_mount`, `open_tree`) byte-identical entry/exit ABI per upstream.
- REQ-2: `enum fsconfig_command` numeric values byte-identical; semantics per upstream's `vfs_fsconfig_locked` dispatch.
- REQ-3: `fsopen` flags + `fsmount` flags + `fspick` flags + `move_mount` flags + `open_tree` flags numeric values byte-identical.
- REQ-4: `fs_context` struct layout-equivalent for first-cache-line + commonly-accessed fields. Fields: `ops`, `lock` (mutex), `purpose` (FS_CONTEXT_FOR_*), `phase`, `need_free`, `global`, `oldapi`, `exclusive`, `fs_type`, `fs_private`, `sget_key`, `root` (dentry), `user_ns`, `net_ns`, `cred`, `log` (error-message log), `source`, `subtype`, `security`, `s_fs_info`, `sb_flags`, `sb_flags_mask`, `s_iflags`.
- REQ-5: `fs_context_operations` vtable byte-identical: `free`, `dup`, `parse_param`, `parse_monolithic`, `get_tree`, `reconfigure`.
- REQ-6: `fs_parser` framework: `fsparam_*` macros (FLAG, STRING, BINARY, PATH, U32, U32OCT, U32HEX, U64, S32, ENUM, BOOL, FD) preserved; `fs_parse(fc, params, param, result)` produces byte-identical errno + log messages on parse failures.
- REQ-7: Per-FS init_fs_context: every existing in-tree FS (ext4, btrfs, xfs, tmpfs, …) plugs in identically; out-of-tree FSes work without changes.
- REQ-8: Multi-step flow: fsopen → fsconfig (multiple times) → fsconfig(FSCONFIG_CMD_CREATE) → fsmount → move_mount. Each step validates state; aborting (close fsfd) cleans up cleanly.
- REQ-9: Reconfiguration: `fsconfig(FSCONFIG_CMD_RECONFIGURE)` modifies an existing mount (modern equivalent of `mount -o remount`); identical semantics.
- REQ-10: Error reporting: `read(fsfd, buf, len)` after a parse error returns the queued error messages from `fc->log`; format-identical to upstream.
- REQ-11: Idmapped-mount integration via `MOUNT_ATTR_IDMAP` in fsmount (cross-ref `fs/vfs/mount.md`).
- REQ-12: `move_mount` with `MOVE_MOUNT_SET_GROUP` (newer) preserves upstream semantics.
- REQ-13: LSM hooks: `security_fs_context_dup`, `security_fs_context_free`, `security_fs_context_parse_param`, `security_sb_mount` (when fc lands as a real mount), `security_move_mount`. Identical.
- REQ-14: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `strace -e trace=fsopen,fsconfig,fsmount,fspick,move_mount,open_tree` of a systemd boot byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: A test exercising every fsconfig command on a tmpfs context succeeds; subsequent fsmount + move_mount mounts the tmpfs at the expected path. (covers REQ-2, REQ-8)
- [ ] AC-3: A test exercising every fsopen/fsmount/fspick/move_mount flag in valid combinations succeeds; invalid combinations return EINVAL identically. (covers REQ-3)
- [ ] AC-4: `pahole struct fs_context` first-cache-line layout byte-identical vs. upstream. (covers REQ-4)
- [ ] AC-5: `pahole struct fs_context_operations` byte-identical. (covers REQ-5)
- [ ] AC-6: A parameter-parsing harness exercises every fsparam_* macro on a curated input set; output (parsed value or error) matches upstream byte-for-byte. (covers REQ-6)
- [ ] AC-7: A canonical FS (ext4) loaded via fsopen+fsconfig+fsmount succeeds; verifiable via mountinfo. (covers REQ-7)
- [ ] AC-8: A test that fsopen + fsconfig + closes the fsfd before fsmount cleans up the partial fs_context (verifiable via slabinfo). (covers REQ-8)
- [ ] AC-9: A test that uses fsconfig(FSCONFIG_CMD_RECONFIGURE) on an existing mount changes the mount's flags; observable via mountinfo. (covers REQ-9)
- [ ] AC-10: After a deliberate parse failure (`fsconfig(FSCONFIG_SET_STRING, "nonsense", "value")`), `read(fsfd)` returns "FS-name: Unknown option 'nonsense'" (format-identical to upstream). (covers REQ-10)
- [ ] AC-11: An idmapped-mount via `fsmount + MOUNT_ATTR_IDMAP` test (cross-ref `fs/vfs/mount.md`). (covers REQ-11)
- [ ] AC-12: `move_mount` with MOVE_MOUNT_SET_GROUP test passes. (covers REQ-12)
- [ ] AC-13: An LSM-policy denying `security_fs_context_parse_param` results in EACCES from fsconfig. (covers REQ-13)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-14)

### Architecture

### Rust module organization

- `kernel::fs::vfs::fs_context::FsContext` — `struct fs_context` wrapper
- `kernel::fs::vfs::fs_context::ops::FsContextOps` — fs_context_operations vtable trait
- `kernel::fs::vfs::fs_context::syscalls::fsopen` / `fsconfig` / `fsmount` / `fspick` / `move_mount` / `open_tree`
- `kernel::fs::vfs::fs_parser::FsParser` — parameter-parsing helpers
- `kernel::fs::vfs::fs_context::log::ErrorLog` — `fc->log` error message log
- `kernel::fs::vfs::fs_context::lifecycle` — fsopen → fsconfig → fsmount lifecycle

### Locking and concurrency

- **`fc->lock`** (mutex): per-fs-context; held during state transitions
- **`fc->log->mutex`** (mutex): per-error-log; held during message append
- Cross-ref `fs/vfs/mount.md`'s `namespace_sem` and `mount_lock` for the move_mount step

### Error handling

- `Err(EINVAL)` — bad fsconfig command / bad parameter / phase mismatch
- `Err(ENOENT)` — fs_type not registered
- `Err(EBADF)` — fsfd is closed
- `Err(EBUSY)` — fs_context in active operation
- `Err(EACCES)` — LSM denied
- `Err(ENOMEM)` — fs_context alloc failed

### Out of Scope

- Mount-tree manipulation (cross-ref `fs/vfs/mount.md`)
- Per-FS init_fs_context implementations (cross-ref individual fs/* docs)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| fs_context lifecycle + multi-step flow | `fs/fs_context.c` |
| fs_parser parameter-parsing helpers | `fs/fs_parser.c` |
| fsopen + fsconfig + fsmount + move_mount syscalls | `fs/fsopen.c`, plus `fs/namespace.c` cross-ref |
| Filesystem-type registration init_fs_context callback | `fs/filesystems.c` |
| Public types | `include/linux/fs_context.h`, `include/linux/fs_parser.h` |
| UAPI flags | `include/uapi/linux/mount.h` (FSCONFIG_*, FSMOUNT_*, FSOPEN_*, MOUNT_ATTR_*) |

### compatibility contract

### Syscalls

`fsopen(name, flags)`, `fsconfig(fd, cmd, key, value, aux)`, `fsmount(fsfd, flags, attr_flags)`, `fspick(dirfd, path, flags)`, `move_mount(from_dirfd, from_path, to_dirfd, to_path, flags)`, `open_tree(dfd, path, flags)`. Each gets a Tier-5 doc.

### `fsopen` flags (`include/uapi/linux/mount.h`)

`FSOPEN_CLOEXEC=0x1`. Numeric byte-identical.

### `fsconfig` commands (`enum fsconfig_command`)

`FSCONFIG_SET_FLAG=0`, `FSCONFIG_SET_STRING=1`, `FSCONFIG_SET_BINARY=2`, `FSCONFIG_SET_PATH=3`, `FSCONFIG_SET_PATH_EMPTY=4`, `FSCONFIG_SET_FD=5`, `FSCONFIG_CMD_CREATE=6`, `FSCONFIG_CMD_RECONFIGURE=7`, `FSCONFIG_CMD_CREATE_EXCL=8`. Numeric byte-identical.

### `fsmount` flags

`FSMOUNT_CLOEXEC=0x1`. Plus `MOUNT_ATTR_*` for the per-mount attributes (cross-ref `fs/vfs/mount.md`).

### `fspick` flags

`FSPICK_CLOEXEC=0x1`, `FSPICK_SYMLINK_NOFOLLOW=0x2`, `FSPICK_NO_AUTOMOUNT=0x4`, `FSPICK_EMPTY_PATH=0x8`. Numeric byte-identical.

### `move_mount` flags

`MOVE_MOUNT_F_SYMLINKS=0x1`, `MOVE_MOUNT_F_AUTOMOUNTS=0x2`, `MOVE_MOUNT_F_EMPTY_PATH=0x4`, `MOVE_MOUNT_T_SYMLINKS=0x10`, `MOVE_MOUNT_T_AUTOMOUNTS=0x20`, `MOVE_MOUNT_T_EMPTY_PATH=0x40`, `MOVE_MOUNT_SET_GROUP=0x100`, `MOVE_MOUNT_BENEATH=0x200`. Numeric byte-identical.

### `open_tree` flags

`OPEN_TREE_CLONE=0x1`, `OPEN_TREE_CLOEXEC=0x80000` (= O_CLOEXEC). Plus reuses many AT_* flags.

### Error reporting

After `fsconfig` failure, `read(fsfd, ...)` returns a human-readable error message describing what went wrong (e.g., "ext4: Unknown option 'foobar'"). Format-identical to upstream — userspace error-display tools (mount, systemd-mount) parse identically.

### Per-FS init_fs_context callback

Each filesystem registers via `struct file_system_type` (cross-ref `fs/vfs/super.md`). Modern FSes implement `init_fs_context` instead of legacy `mount`. The callback initializes `fc->ops` (the FS-specific fs_context_operations: parse_param, parse_monolithic, get_tree, reconfigure, free, dup) and FS-private state in `fc->fs_private`.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| fs_context alloc + initialization | `kani::proofs::fs::vfs::fs_context::alloc_safety` |
| fsconfig parameter-parsing dispatch | `kani::proofs::fs::vfs::fs_parser::parse_safety` |
| fsmount → vfsmount creation | `kani::proofs::fs::vfs::fs_context::fsmount_safety` |
| Error log append (queue parse errors) | `kani::proofs::fs::vfs::fs_context::log_safety` |
| move_mount transition | `kani::proofs::fs::vfs::fs_context::move_mount_safety` |

### Layer 2: TLA+ models

(none mandatory — fs_context lifecycle is per-fc; concurrency models live in mount.md)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| fs_context phase machine | Transitions follow CREATING → CREATED → DESTROYING; no skipped states | `kani::proofs::fs::vfs::fs_context::phase_invariants` |
| Error log | Messages preserved in order; reading drains; over-quota messages dropped | `kani::proofs::fs::vfs::fs_context::log_invariants` |
| Parameter-parsing | Result matches upstream `fs_parse` for identical input | `kani::proofs::fs::vfs::fs_parser::parse_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Parameter-parser correctness** via Creusot — proves: fsparam_* macros produce result + errno matching upstream's reference implementation.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | fs_context refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | fs_context allocated via per-FS-type slab cache | § Mandatory |

### Row-1 features consumed by this component

- **UDEREF**: parameter strings + paths from userspace go through `getname` (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: parameter-list length + buffer-size arithmetic uses checked operators
- **CONSTIFY**: per-FS fs_context_operations vtables are `static const`
- **MEMORY_SANITIZE**: freed fs_context objects zeroed (especially relevant for any temporarily-stored secrets in fc->fs_private)

### Row-2 / GR-RBAC integration

LSM hooks dispatched:
- `security_fs_context_dup` / `security_fs_context_free` — context lifecycle
- `security_fs_context_parse_param` — per-parameter check
- `security_sb_mount` — pre-mount permission check (when context lands as a real mount)
- `security_move_mount` — modern mount API hook

GR-RBAC's mount-restriction policies (cross-ref `fs/vfs/mount.md`) consume these hooks.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

