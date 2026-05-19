---
title: "Tier-5 syscall: getcwd(2) — syscall 79"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getcwd(2)` writes the absolute pathname of the calling thread group's current working directory (cwd) into a user-provided buffer. The path is rendered relative to the calling task's `fs->root` (its chroot, if any) — meaning a chrooted process will see paths that begin at its own chroot root, not the global filesystem root.

Internally the kernel walks the dentry chain from `current->fs->pwd.dentry` back up to `current->fs->root.dentry`, prepending each component, and crosses vfsmount boundaries via `follow_up`. If the dentry chain becomes disconnected (the cwd was unlinked) the path is suffixed with " (deleted)". Critical for: shell PWD, /proc/self/cwd readlink target, container-relative paths.

### Acceptance Criteria

- [ ] AC-1: chdir("/tmp"); getcwd(buf, 4096) returns 5; buf == "/tmp\0".
- [ ] AC-2: getcwd(NULL, 0): libc wrapper allocates; raw syscall behavior kernel-defined.
- [ ] AC-3: getcwd(buf, 2): -ERANGE for any path longer than 1 char.
- [ ] AC-4: chroot("/jail"); chdir("/"); getcwd: "/" (chroot-relative).
- [ ] AC-5: chdir into a dir; unlink the dir; getcwd: "<path> (deleted)".
- [ ] AC-6: getcwd on a thread without CLONE_FS: per-thread value.
- [ ] AC-7: bind-mounted cwd: getcwd shows bind-mount path.
- [ ] AC-8: getcwd under concurrent rename: rename_lock seqlock retries until consistent.
- [ ] AC-9: buf == kernel-pointer-shaped userspace: -EFAULT.

### Architecture

```rust
#[syscall(nr = 79, abi = "sysv")]
pub fn sys_getcwd(buf: UserPtr<u8>, size: u64) -> isize {
    Fs::do_getcwd(buf, size)
}
```

`Fs::do_getcwd(buf, size) -> isize`:
1. /* Snapshot pwd and root under fs.lock */
2. spin_lock(&current.fs.lock);
3. let pwd = current.fs.pwd.clone();
4. let root = current.fs.root.clone();
5. spin_unlock(&current.fs.lock);
6. /* Allocate kernel page */
7. let page = alloc_page(GFP_USER).ok_or(ENOMEM)?;
8. let buf_end = page.virt().add(PAGE_SIZE);
9. let mut cur = buf_end;
10. /* Walk dentry chain */
11. seq_retry: loop {
12.   let seq = read_seqbegin(&rename_lock);
13.   rcu_read_lock();
14.   let r = prepend_path(&pwd, &root, &mut cur, page.virt());
15.   rcu_read_unlock();
16.   if read_seqretry(&rename_lock, seq) { cur = buf_end; continue 'seq_retry; }
17.   break r;
18. }?;
19. /* Compute length */
20. let len = buf_end as usize - cur as usize;
21. if len as u64 > size { free_page(page); return -ERANGE; }
22. /* Move to start */
23. memmove(page.virt(), cur, len);
24. /* Copy to user */
25. let r = copy_to_user(buf, page.virt(), len);
26. free_page(page);
27. if r != 0 { return -EFAULT; }
28. len as isize

`Fs::prepend_path(pwd, root, cur, start) -> Result<()>`:
1. let mut p = *pwd;
2. while p.dentry != root.dentry || p.mnt != root.mnt {
3.   if p.dentry == p.mnt.mnt_root {
4.     /* Cross vfsmount */
5.     if let Some(parent_mnt) = follow_up(&mut p) { continue; }
6.     /* Disconnected */
7.     break;
8.   }
9.   let name = p.dentry.d_name();
10.  cur -= name.len + 1;
11.  if cur < start { return Err(ENAMETOOLONG); }
12.  write_bytes(cur, "/");
13.  write_bytes(cur + 1, name);
14.  p.dentry = p.dentry.d_parent();
15. }
16. if cur == buf_end { cur -= 1; write_byte(cur, '/'); }
17. /* Append " (deleted)" if dentry disconnected */
18. Ok(())

### Out of Scope

- Per-chdir(2) (covered in Tier-5 chdir.md).
- Per-fchdir(2) (covered in Tier-5 fchdir.md).
- Per-/proc/self/cwd magic symlink (covered in Tier-3 fs/proc-self.md).
- Implementation code.

### signature

```c
char *getcwd(char *buf, size_t size);   /* libc wrapper */
long  __getcwd(char *buf, unsigned long size);  /* raw syscall */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `buf` | `char *` | out | User buffer to receive the NUL-terminated path. |
| `size` | `unsigned long` | in | Capacity of `buf` in bytes (including the trailing NUL). |

### return value

| Value | Meaning |
|---|---|
| `> 0` | Length of the path written (including the trailing NUL). |
| `-1` + `errno` | Failure. |

(libc wrapper returns `buf` on success, NULL with `errno` on failure.)

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `buf` outside the caller's address space. |
| `ERANGE` | `size` smaller than the rendered path length (including NUL). |
| `EINVAL` | `size == 0` and `buf != NULL` (POSIX-defined; some kernels accept). |
| `ENOENT` | The cwd has been unlinked AND the kernel is configured to refuse rendering "(deleted)" paths (rare; vanilla returns the path with the suffix). |
| `EACCES` | Search permission on a component along the chain denied (DAC walk during prepend_path). |

### abi surface

```text
__NR_getcwd  (x86_64) = 79
__NR_getcwd  (arm64)  = 17
__NR_getcwd  (riscv)  = 17
__NR_getcwd  (i386)   = 183
```

### compatibility contract

REQ-1: Syscall number is **79** on x86_64. ABI-stable.

REQ-2: Path is rendered relative to `current->fs->root` (the caller's chroot). A chrooted task that has not chdir'd outside its chroot sees paths starting with "/".

REQ-3: Path is rendered from `current->fs->pwd` walking dentry parents and crossing vfsmount boundaries (`follow_up`) until `current->fs->root` is reached, or until a disconnected dentry / global root is hit.

REQ-4: Path is written into `buf` END-FIRST (prepend semantics); the kernel computes the length and writes from `buf + size - 1` backward, then memmoves to the start of the buffer.

REQ-5: ERANGE: if the rendered path would exceed `size` (including the trailing NUL), getcwd returns -ERANGE without writing a partial path.

REQ-6: Unlinked cwd: kernel appends " (deleted)" to the path. POSIX is silent on this; Linux behavior is documented.

REQ-7: Bind-mounted cwd: getcwd renders the bind-mount path (the `f_path.mnt` view), not the underlying directory's source path.

REQ-8: Mount-namespace scope: if the cwd is in a vfsmount not visible to the caller's mount namespace (rare — would require ns-change races), getcwd may return "/" or an unanchored path.

REQ-9: Per-CLONE_FS: paths reflect the shared fs_struct.

REQ-10: Per-O_PATH-like: getcwd does NOT take an fd; it always uses current->fs->pwd. For an arbitrary directory fd, use `readlink("/proc/self/fd/N", ...)` or `name_to_handle_at`.

REQ-11: Per-d_op->d_dname callback: some pseudo-filesystems (anon_inode, pipefs) provide custom name renderers — these are NOT used by getcwd (only by /proc/self/fd readlink).

REQ-12: Per-RCU walk: getcwd takes rcu_read_lock around the dentry walk, then `rename_lock` seqlock for retry on concurrent rename.

REQ-13: Per-disconnected dentry (cwd unlinked + parent unlinked all the way): getcwd returns "/" + " (deleted)" or ENOENT depending on kernel version (current Linux returns the path with suffix).

REQ-14: Per-thread cwd: per-thread fs_struct (CLONE_FS=0) yields per-thread getcwd answers.

REQ-15: getcwd is interruptible: not signal-interruptible (short kernel path).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `buffer_bounded` | INVARIANT | written length <= size; ERANGE otherwise. |
| `path_under_root` | INVARIANT | rendered path is rooted at fs.root. |
| `rcu_locked_walk` | INVARIANT | dentry walk holds rcu_read_lock + rename_lock seqlock. |
| `copy_to_user_safe` | INVARIANT | exactly len bytes copied to user. |

### Layer 2: TLA+

`fs/getcwd.tla`:
- States: per-snapshot-pwd-root, per-prepend-walk, per-mnt-cross, per-copy-out.
- Properties:
  - `safety_path_starts_with_slash` — every successful return path begins with '/'.
  - `safety_path_bounded_by_size` — ERANGE returned if overflow.
  - `safety_chroot_relative` — chrooted task sees chroot-relative paths.
  - `liveness_getcwd_terminates` — getcwd terminates under bounded rename activity.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getcwd` post: success ⟹ path is canonical pwd-to-root | `Fs::do_getcwd` |
| `prepend_path` post: dentry chain fully walked or disconnected | `Fs::prepend_path` |
| `seq_retry` post: terminates under bounded concurrent rename | `Fs::do_getcwd` |

### Layer 4: Verus / Creusot functional

Per-`getcwd(2)` and `getcwd(3)` man-pages, per-Linux fs/d_path.c, ltp syscalls/getcwd suite.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getcwd(2)` reinforcement:

- **Per-snapshot under fs.lock** — defense against per-pwd-root tearing.
- **Per-rename_lock seqlock retry** — defense against per-concurrent-rename inconsistent path.
- **Per-bounded prepend buffer** — defense against per-kernel-buffer overflow.
- **Per-ERANGE strict** — defense against per-truncated-path data leak.
- **Per-chroot-relative rendering** — defense against per-chroot path information leak.

### grsecurity / pax surface

- **PaX UDEREF on user buf copy_to_user** — defense against per-userspace-pointer kernel deref; SMAP forced.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — chrooted process invoking getcwd sees a path rooted strictly at its chroot; the kernel never leaks parent-of-chroot path components even if the cwd dentry chain physically reaches above.
- **GRKERNSEC_HIDESYM in getcwd error klog** — pointers in error path stripped.
- **PAX_REFCOUNT on path snapshot during getcwd** — defense against per-fs_struct UAF.
- **PaX KERNEXEC on follow_up dispatch** — indirect-call hardened.
- **Sync data-leak prevention** — getcwd buffer cleared before being copied out so trailing scratch bytes do not leak prior page contents.
- **CAP_DAC_OVERRIDE strict** — getcwd does not perform DAC bypass; it merely renders a path string. (No DAC check is normally applied; the prepend walks only requires the dentry chain.)
- **GRKERNSEC_DMESG** — getcwd error klog rate-limited and CAP_SYSLOG-gated.
- **PAX_USERCOPY_HARDEN on copy_to_user** — defense against per-buffer overflow.
- **Per-namespace scoping** — if the cwd resolves to a vfsmount outside the current mount namespace (impossible without race), getcwd returns -ENOENT rather than an unanchored leak.
- **Per-deleted suffix sanitized** — " (deleted)" suffix length-checked under buffer cap.

