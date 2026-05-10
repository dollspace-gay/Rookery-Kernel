# Tier-3: fs/vfs/path-resolution — namei.c, LOOKUP_*, RCU-walk + ref-walk, symlinks

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/namei.c
  - fs/d_path.c
  - include/linux/namei.h
  - include/linux/path.h
  - include/uapi/linux/openat2.h
-->

## Summary
Tier-3 design for path resolution: walking a `/`-separated path string into a `(vfsmount, dentry)` pair. Owns `namei.c`'s `__path_lookup` family, the LOOKUP_* flag set, the RCU-walk fast path (most lookups; lockless via seqcounts), the ref-walk fallback (when RCU walks must restart), symlink resolution + the deep-symlink + LOOKUP_FOLLOW interaction, mount-traversal (`__follow_mount` family), and path-formatting (`d_path` + variants in `d_path.c`).

Sub-tier-3 of `fs/vfs/00-overview.md`. Path resolution is the most-frequently-called code path in the kernel: every `open(2)`, `stat(2)`, `unlink(2)`, etc. walks a path. Performance + correctness here are critical.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Path resolution core (RCU + ref walk) | `fs/namei.c` |
| Path-formatting (d_path family) | `fs/d_path.c` |
| Public API | `include/linux/namei.h`, `include/linux/path.h` |
| Modern openat2 UAPI | `include/uapi/linux/openat2.h` |

## Compatibility contract

### Syscalls

`open`, `openat`, `openat2`, `creat`, `mkdir`, `mkdirat`, `rmdir`, `rmdirat` (no, just rmdir; mkdirat is the ONE), `unlink`, `unlinkat`, `link`, `linkat`, `symlink`, `symlinkat`, `rename`, `renameat`, `renameat2`, `mknod`, `mknodat`, `chmod`, `fchmodat`, `fchmodat2`, `chown`, `lchown`, `fchownat`, `chdir`, `chroot`, `fchdir`, `getcwd`, `readlink`, `readlinkat`, `stat`, `lstat`, `fstatat`, `statx`, `name_to_handle_at`, `open_by_handle_at`, `acct`. Each gets a Tier-5 doc.

### LOOKUP_* flags

`include/linux/namei.h`:
- `LOOKUP_FOLLOW=0x1` — dereference final symlink
- `LOOKUP_DIRECTORY=0x2` — caller wants a directory
- `LOOKUP_AUTOMOUNT=0x4` — trigger automount of parent
- `LOOKUP_PARENT=0x10` — caller wants parent + final-component name (for create/unlink ops)
- `LOOKUP_REVAL=0x20` — force revalidate dcache
- `LOOKUP_RCU=0x40` — RCU-walk mode (internal; auto)
- `LOOKUP_OPEN=0x100` — for open(2)
- `LOOKUP_CREATE=0x200` — for create
- `LOOKUP_EXCL=0x400` — for O_EXCL
- `LOOKUP_RENAME_TARGET=0x800` — rename(2)'s target component
- `LOOKUP_JUMPED=0x1000` — internal: walk jumped (e.g., absolute path with leading /)
- `LOOKUP_ROOT=0x2000` — internal: walk reached fs root
- `LOOKUP_EMPTY=0x4000` — allow empty name (for AT_EMPTY_PATH)
- `LOOKUP_DOWN=0x8000` — for path resolution under a specific dirfd
- `LOOKUP_MOUNTPOINT=0x10000` — return the mountpoint, not the mount root
- `LOOKUP_BENEATH=0x80000` — fail if walk exits the starting dir
- `LOOKUP_IN_ROOT=0x40000` — fail if walk crosses fs-root
- `LOOKUP_NO_SYMLINKS=0x100000` — fail on any symlink encountered
- `LOOKUP_NO_MAGICLINKS=0x200000` — fail on /proc/<pid>/fd/N magiclinks
- `LOOKUP_NO_XDEV=0x800000` — fail if walk crosses mountpoint
- `LOOKUP_CACHED=0x1000000` — only check the dcache; don't initiate fs lookup

Numeric values byte-identical.

### `openat2(2)` flags

`include/uapi/linux/openat2.h`:
- `RESOLVE_NO_XDEV=0x01`
- `RESOLVE_NO_MAGICLINKS=0x02`
- `RESOLVE_NO_SYMLINKS=0x04`
- `RESOLVE_BENEATH=0x08`
- `RESOLVE_IN_ROOT=0x10`
- `RESOLVE_CACHED=0x20`

Numeric values byte-identical.

### Path-format helpers (`d_path.c`)

- `d_path(path, buf, len)` — format absolute path
- `d_absolute_path(path, buf, len)` — variant
- `dynamic_dname(...)` — for special dentries (e.g., pipefs, sockfs)
- `simple_dname(...)`, `__d_path(...)`, `dentry_path_raw(...)`, `dentry_path(...)`, `__dentry_path(...)` — variants

Output byte-identical to upstream — `getcwd(2)`, `/proc/<pid>/exe`, `/proc/<pid>/cwd` etc. produce identical strings.

### Deep-symlink limit

`MAX_NESTED_LINKS=8` and `SYMLOOP_MAX=40`. Constants byte-identical.

### Magic links (`/proc/<pid>/fd/N` style)

`/proc/<pid>/fd/N`, `/proc/<pid>/exe`, `/proc/<pid>/cwd`, `/proc/<pid>/root`, `/proc/<pid>/ns/*` are "magic" symlinks that resolve via per-task data, not on-disk content. Resolution behavior preserved per upstream.

## Requirements

- REQ-1: Every path-resolution syscall byte-identical entry/exit ABI per upstream.
- REQ-2: LOOKUP_* numeric values + semantics byte-identical.
- REQ-3: openat2 RESOLVE_* numeric values + semantics byte-identical.
- REQ-4: RCU-walk fast path: traverses dcache via seqcounts; on inconsistency (rename, mount-cross, symlink), retries in ref-walk mode. Behavior + restart conditions match upstream.
- REQ-5: Ref-walk fallback: takes references on dentries + vfsmounts as it walks; releases on completion. Identical lock acquisition order.
- REQ-6: Symlink resolution: follows up to MAX_NESTED_LINKS levels of nesting; LOOKUP_NO_SYMLINKS rejects any symlink; `O_NOFOLLOW` + LOOKUP_FOLLOW interaction matches upstream.
- REQ-7: Mount traversal: `__follow_mount` walks across mountpoints during path resolution; LOOKUP_NO_XDEV / RESOLVE_NO_XDEV reject cross-mount.
- REQ-8: Path-format helpers (`d_path`, `dentry_path`, etc.) produce byte-identical output for matching inputs (chroot, bind-mount, deleted-file, …).
- REQ-9: `getcwd(2)` returns the path to the calling task's `fs->pwd` (cross-ref `fs/vfs/mount.md`); identical behavior + buffer-size handling.
- REQ-10: chroot integration: per-task `fs->root`; path resolution can't escape via `..` past root; RESOLVE_IN_ROOT enforces this even for non-chrooted tasks.
- REQ-11: Magic-link resolution: `/proc/<pid>/fd/N` resolves via the task's fd's underlying path; LOOKUP_NO_MAGICLINKS rejects.
- REQ-12: TLA+ model `models/fs/dcache_rcu_walk.tla` (cross-ref `fs/vfs/dcache.md`; co-owned here) — proves RCU-walk's "no torn read" invariant under racing rename.
- REQ-13: idmapped-mount integration: when traversing an idmapped mount, the path-component-vs-permission check uses translated uid/gid (cross-ref `fs/vfs/mount.md`).
- REQ-14: LSM hook dispatch: `security_inode_permission` invoked at every component during path walk.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `strace -e trace=file` of a kernel-build run byte-identical for path-resolution semantics. (covers REQ-1)
- [ ] AC-2: A LOOKUP_* + RESOLVE_* test exercising every flag combination produces byte-identical outcome. (covers REQ-2, REQ-3)
- [ ] AC-3: A 16-CPU concurrent path_lookup benchmark with concurrent rename produces zero RCU-walk torn reads. (covers REQ-4, REQ-12)
- [ ] AC-4: A symlink-loop test (`ln -s a b; ln -s b a`) returns ELOOP. (covers REQ-6)
- [ ] AC-5: An LOOKUP_NO_SYMLINKS-using openat2 on a path containing a symlink returns ELOOP. (covers REQ-6)
- [ ] AC-6: An LOOKUP_NO_XDEV-using openat2 on a path that crosses a mountpoint returns EXDEV. (covers REQ-7)
- [ ] AC-7: `d_path` output for various paths (chroot, bind-mount, deleted-file-still-open, deleted-file-then-renamed) byte-identical. (covers REQ-8)
- [ ] AC-8: `getcwd` after `chdir` to various paths returns identical strings. (covers REQ-9)
- [ ] AC-9: A chroot'd task can't escape via `..`-traversal past root; behavior matches upstream. (covers REQ-10)
- [ ] AC-10: openat2 with RESOLVE_NO_MAGICLINKS on `/proc/self/fd/N` returns ELOOP (or appropriate error). (covers REQ-11)
- [ ] AC-11: An idmapped-mount + chmod test (cross-ref `fs/vfs/inode.md`) verifies path-walk uses translated uid for permission check. (covers REQ-13)
- [ ] AC-12: An LSM (e.g., AppArmor) policy denying `inode_permission` translates to EACCES from the openat syscall. (covers REQ-14)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::fs::vfs::namei::Lookup` — top-level path-resolution entry
- `kernel::fs::vfs::namei::rcu_walk` — RCU-walk fast path
- `kernel::fs::vfs::namei::ref_walk` — ref-walk fallback
- `kernel::fs::vfs::namei::symlink` — symlink-following
- `kernel::fs::vfs::namei::mount_traverse` — `__follow_mount` family
- `kernel::fs::vfs::namei::component` — per-component lookup (DOT, DOTDOT, name)
- `kernel::fs::vfs::namei::nameidata` — internal walk state
- `kernel::fs::vfs::d_path::FormatPath` — path-formatting

### Locking and concurrency

- **RCU read-side**: held during RCU-walk; no other locks
- **`d_lock`** (per-dentry spinlock): taken during ref-walk for parent-child link verification
- **`d_seq`** (per-dentry seqcount): RCU-walk readers retry on writer
- **`namespace_sem`** (read-side; cross-ref `fs/vfs/mount.md`): held during mount traversal in some paths

TLA+ model `models/fs/dcache_rcu_walk.tla` (cross-ref `fs/vfs/dcache.md`) proves the seqcount-based RCU-walk under racing rename + invalidate.

### Error handling

- `Err(EACCES)` — permission denied during walk
- `Err(ENOENT)` — path component not found
- `Err(ENOTDIR)` — non-directory component before final
- `Err(ELOOP)` — too many nested symlinks
- `Err(ENAMETOOLONG)` — name too long
- `Err(EXDEV)` — crossed mountpoint with LOOKUP_NO_XDEV / RESOLVE_NO_XDEV
- `Err(EAGAIN)` — RCU-walk failed; switch to ref-walk
- `Err(ESTALE)` — dentry invalidated mid-walk
- `Err(EINVAL)` — bad LOOKUP_* / RESOLVE_* combination

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| RCU-walk seqcount read + retry | `kani::proofs::fs::vfs::namei::rcu_seq_safety` |
| Ref-walk dentry refcount take/drop | `kani::proofs::fs::vfs::namei::ref_walk_safety` |
| Mount-traversal cross-mount | `kani::proofs::fs::vfs::namei::mount_traverse_safety` |
| Symlink-following with depth limit | `kani::proofs::fs::vfs::namei::symlink_depth_safety` |
| `d_path` buffer-bound writes | `kani::proofs::fs::vfs::namei::d_path_buf_safety` |

### Layer 2: TLA+ models

- `models/fs/dcache_rcu_walk.tla` (cross-ref `fs/vfs/dcache.md`; co-owned with this Tier-3) — proves RCU-walk's no-torn-read invariant under racing rename.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Walk state (nameidata) | At every step, `nd->path` points to a valid (vfsmount, dentry) pair; `nd->depth` ≤ MAX_NESTED_LINKS | `kani::proofs::fs::vfs::namei::walk_state_invariants` |
| LOOKUP_* flag interaction | LOOKUP_NO_SYMLINKS + LOOKUP_FOLLOW are mutually exclusive at the final component | `kani::proofs::fs::vfs::namei::flag_consistency_invariants` |
| chroot bound check | walk cannot exit `nd->root` when LOOKUP_IN_ROOT is set | `kani::proofs::fs::vfs::namei::root_bound_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Path-resolution termination** via Verus — proves: under any input path of length N, the walk terminates in O(N + symlinks_encountered * MAX_NESTED_LINKS) steps. Defeats infinite-loop / DoS via crafted symlink chains.
- **`d_path` correctness** via Creusot — proves: for any (vfsmount, dentry) pair, the formatted path matches walking from dentry to fs root. Cross-ref `fs/vfs/dcache.md` for shared infrastructure.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **chroot bound enforcement** (path can't escape root via `..`) | Inherent in upstream's walk algorithm; preserved | (existing) |
| **MAX_NESTED_LINKS / SYMLOOP_MAX** (DoS bound on symlink chains) | Constants enforced; default at upstream values | (existing) |
| **LOOKUP_BENEATH / LOOKUP_IN_ROOT / LOOKUP_NO_SYMLINKS / LOOKUP_NO_MAGICLINKS / LOOKUP_NO_XDEV** | Modern openat2-based hardening; preserved per upstream | (existing) |

### Row-1 features consumed by this component

- **UDEREF**: path strings come through `getname` (cross-ref `lib/usercopy.md`) → kernel-side `qstr`
- **SIZE_OVERFLOW**: name-length + buffer-size arithmetic uses checked operators
- **CONSTIFY**: per-FS `dentry_operations` provided by FS modules are `static const`

### Row-2 / GR-RBAC integration

LSM hooks dispatched per component:
- `security_inode_permission(inode, mask)` — check at every component
- `security_inode_follow_link` — symlink-follow check
- `security_inode_readlink` — readlink permission

GR-RBAC's TPE (Trusted Path Execution) + chroot-hardening (cross-ref `references/grsec-pax-notes.md`) consume these hooks. The grsec GRKERNSEC_CHROOT_* path-restrictions become loadable settings of the GR-RBAC LSM (per the layered absorption model).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — path-resolution semantics are exhaustively specified by POSIX + Linux extensions)

## Out of Scope

- dcache hash + LRU (cross-ref `fs/vfs/dcache.md`)
- inode permission check (cross-ref `fs/vfs/inode.md`)
- mount tree (cross-ref `fs/vfs/mount.md`)
- 32-bit-only paths
- Implementation code
