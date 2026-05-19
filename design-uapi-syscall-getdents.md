---
title: "Tier-5 syscall: getdents(2) — syscall 78"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getdents(2)` is the legacy 32-bit-inode directory-reading interface. It reads several `struct linux_dirent` entries from the directory referred to by `fd` into the caller-provided buffer `dirp`, advancing the directory's seek position. It pre-dates `getdents64(2)` (which uses 64-bit inode numbers and includes a `d_type` field at a fixed offset); modern code uses `getdents64` exclusively, but `getdents(2)` remains in the UAPI for very old binaries.

In the legacy layout, the trailing byte at `[offset_of(d_name) + namelen]` holds the file type (DT_*), accessed via the `dirent->d_name[strlen(d_name)+1]` "type at name+strlen+1" idiom — a notorious foot-gun fixed by `getdents64`. Critical for: `ls(1)` from 1990s glibc, `bash`'s wildcard expansion legacy paths, any statically-linked vintage binary, and the libc emulation surface for `readdir(3)` on i686.

### Acceptance Criteria

- [ ] AC-1: getdents on opendir'd /tmp returns > 0; entries are valid linux_dirent layout.
- [ ] AC-2: getdents called repeatedly until 0 enumerates the whole directory.
- [ ] AC-3: getdents on a non-directory fd returns -ENOTDIR.
- [ ] AC-4: count = 0 returns -EINVAL.
- [ ] AC-5: count = sizeof(linux_dirent) - 1 returns -EINVAL.
- [ ] AC-6: EFAULT on dirp: returns -EFAULT, file offset unchanged for the failed entry.
- [ ] AC-7: A directory with inode > 2^32 on 32-bit ABI: returns -EOVERFLOW for that entry.
- [ ] AC-8: lseek(fd, d_off, SEEK_SET) then getdents resumes at the right entry.
- [ ] AC-9: Concurrent unlink during getdents: results contain pre-unlink names (snapshot semantics OR linearized — fs-dependent).
- [ ] AC-10: Type byte at dirp + d_reclen - 1 matches inode's S_IFMT.

### Architecture

```rust
#[syscall(nr = 78, abi = "sysv")]
pub fn sys_getdents(fd: u32, dirp: UserPtr<LinuxDirent>, count: u32) -> isize {
    let filp = Files::fget(fd as i32).ok_or(EBADF)?;
    let ret = ReadDir::do_getdents(&filp, dirp, count);
    Files::fput(filp);
    ret
}
```

`ReadDir::do_getdents(filp, dirp, count) -> isize`:
1. if !filp.is_dir() { return Err(ENOTDIR); }
2. if (count as usize) < size_of::<LinuxDirent>() { return Err(EINVAL); }
3. let mut ctx = GetDentsCtx {
4.     pos: filp.f_pos,
5.     dirp: dirp,
6.     count: count,
7.     used: 0,
8.     error: 0,
9. };
10. let actor = &|name, namlen, offset, ino, d_type| {
11.     ReadDir::filldir_legacy(&mut ctx, name, namlen, offset, ino, d_type)
12. };
13. inode_read_lock(filp.inode());
14. let r = filp.f_op.iterate_shared(filp, &actor);
15. inode_read_unlock(filp.inode());
16. filp.f_pos = ctx.pos;
17. if ctx.error != 0 { return Err(ctx.error); }
18. if ctx.used == 0 && r == 0 { return Ok(0); }   // EOF or no entries fit
19. Ok(ctx.used as isize)

`ReadDir::filldir_legacy(ctx, name, namlen, offset, ino, d_type) -> i32`:
1. let reclen = round_up(offset_of!(LinuxDirent, d_name) + namlen + 2, ALIGNOF_ULONG);
2. if ctx.used + reclen > ctx.count { return 1; }   // signal stop
3. let ino_ul = match ULong::try_from(ino) {
4.     Ok(v) => v,
5.     Err(_) => { ctx.error = EOVERFLOW; return -EOVERFLOW; }
6. };
7. let mut buf = StackBuf::new(reclen);
8. buf.write_field(d_ino,  ino_ul);
9. buf.write_field(d_off,  offset as ULong);
10. buf.write_field(d_reclen, reclen as u16);
11. buf.write_name(name, namlen);
12. buf.write_byte(reclen - 1, d_type);            // type at name+strlen+1 padding
13. unsafe {
14.     match ctx.dirp.add_bytes(ctx.used).copy_out(&buf, reclen) {
15.         Ok(()) => {}
16.         Err(_) => { ctx.error = EFAULT; return -EFAULT; }
17.     }
18. }
19. ctx.used += reclen;
20. ctx.pos = offset;
21. 0

### Out of Scope

- getdents64(2) (covered in Tier-5 `getdents64.md`).
- Per-fs iterate_shared implementations (covered per-fs Tier-3 docs).
- readdir(3) libc wrapper (covered in glibc / musl docs).
- Implementation code.

### signature

```c
int getdents(unsigned int fd,
             struct linux_dirent *dirp,
             unsigned int count);

struct linux_dirent {
    unsigned long  d_ino;     /* inode number (32-bit on 32-bit kernels) */
    unsigned long  d_off;     /* offset to next dirent */
    unsigned short d_reclen;  /* length of this dirent */
    char           d_name[];  /* filename + '\0', then [d_type, '\0' pad] */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `unsigned int` | in | Open file descriptor for a directory. |
| `dirp` | `struct linux_dirent *` | out | Buffer to receive entries. |
| `count` | `unsigned int` | in | Size of `dirp` in bytes; must be ≥ sizeof(struct linux_dirent). |

### return value

| Value | Meaning |
|---|---|
| `>0` | Number of bytes written to `dirp` (sum of `d_reclen` of all entries written). |
| `0` | End of directory. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not open. |
| `EFAULT` | `dirp` user buffer faults during copy_to_user. |
| `EINVAL` | `count` too small to fit even one entry; `fd` is not a directory; underlying inode type unsupported. |
| `ENOENT` | Directory was removed (rare). |
| `ENOTDIR` | `fd` refers to a non-directory. |
| `EOVERFLOW` | An inode number does not fit in `unsigned long` (32-bit ABI on >2³² inode FS); use getdents64 instead. |

### abi surface

```text
__NR_getdents   (x86_64)  = 78
__NR_getdents   (arm64)   = absent — arm64 / riscv expose only getdents64 (61).
__NR_getdents   (riscv)   = absent
__NR_getdents   (i386)    = 141

/* linux_dirent layout: variable-length d_name; d_reclen is the stride. */
/* Type byte is at dirp + d_reclen - 1; namelen via strlen(d_name).      */
/* getdents64 puts d_type at a fixed offset — strongly preferred.        */
```

### compatibility contract

REQ-1: Syscall number is **78** on x86_64. Absent on arm64 / riscv; those architectures expose only `getdents64(2)`.

REQ-2: `fd` resolved via `fget`; must refer to a directory (S_ISDIR), else `-ENOTDIR`.

REQ-3: `dirp` is a writable buffer of at least `count` bytes; SMAP-guarded copy_to_user for each entry.

REQ-4: Each entry has:
- `d_ino`: inode number, truncated to `unsigned long`. If the real ino doesn't fit, `-EOVERFLOW` is returned and the call aborts (no entry written for that record).
- `d_off`: offset of the next entry in the directory stream (used by `lseek(fd, d_off, SEEK_SET)` to resume).
- `d_reclen`: byte stride of this record; always rounded up to alignof(unsigned long) (typically 8).
- `d_name`: NUL-terminated filename.
- File type byte at `dirp + d_reclen - 1`: one of DT_UNKNOWN / DT_FIFO / DT_CHR / DT_DIR / DT_BLK / DT_REG / DT_LNK / DT_SOCK / DT_WHT.

REQ-5: At least one entry must fit in `count`; if not, `-EINVAL`.

REQ-6: The kernel iterates via `f_op->iterate_shared(filp, &ctx)` (preferred) or legacy `f_op->iterate`. The callback `filldir(ctx, name, namlen, offset, ino, d_type)` formats each dirent into the user buffer.

REQ-7: filldir behavior on partial-fill:
- If the next entry would not fit, stop (return positive bytes already written).
- If a write fault (EFAULT) occurs mid-entry, abort with `-EFAULT` and the file position is unchanged for that entry.
- d_off is set BEFORE writing, so on resume, the directory position is correct.

REQ-8: Per-shared-iterate: holds inode read-lock for the duration; concurrent rename/unlink may see partial state but never corrupt the buffer.

REQ-9: Directory hash collisions / telldir-seekdir: position cookie is `d_off` of the previous entry (kernel-defined opaque); userspace must treat it as opaque.

REQ-10: Per-CAP_DAC_READ_SEARCH override: usual permission rules apply at directory-open time; getdents itself never re-checks permissions.

REQ-11: Per-namespace: directory contents are namespace-filtered (e.g. `/proc` filters per-pidns); getdents inherits the per-fs iterate's namespace awareness.

REQ-12: Per-overlayfs / fuse / NFS: backend iterates upstream; copy-up and lookup ordering may produce duplicate entries (handled by libc).

REQ-13: `count` MUST be < `INT_MAX` (kernel internal bound on size_t→ssize_t).

REQ-14: getdents(2) does not consume entropy and does not require any capability.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_is_dir` | INVARIANT | non-dir fd ⟹ ENOTDIR. |
| `count_min_fits` | INVARIANT | count < sizeof(linux_dirent) ⟹ EINVAL. |
| `reclen_alignof_ulong` | INVARIANT | every d_reclen is a multiple of alignof(unsigned long). |
| `ino_overflow_eovrflw` | INVARIANT | ino > ULong::MAX ⟹ EOVERFLOW for that entry. |
| `efault_no_pos_advance` | INVARIANT | copy_out failure ⟹ f_pos not advanced past failed entry. |
| `type_byte_at_end` | INVARIANT | DT_* byte at dirp + d_reclen - 1. |

### Layer 2: TLA+

`fs/readdir-syscall.tla`:
- States: per-fget, per-validate, per-iterate, per-filldir, per-copy-out, per-resume.
- Properties:
  - `safety_no_partial_entry` — entry either fully written or not written.
  - `safety_pos_consistent` — f_pos advances to last fully-written entry's d_off.
  - `safety_inode_overflow_eovrflw` — never silent truncation of ino.
  - `liveness_progress_or_eof` — iterate terminates either with bytes>0 or EOF=0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getdents` post: ret >= 0 ⟹ bytes-written matches sum of d_reclen | `ReadDir::do_getdents` |
| `filldir_legacy` post: ino fits in ULong, else EOVERFLOW | `ReadDir::filldir_legacy` |
| `GetDentsCtx` invariant: used ≤ count | `ReadDir::filldir_legacy` |

### Layer 4: Verus / Creusot functional

Per-getdents(2) Linux semantic equivalence; LTP `getdents01..getdents03`, glibc `readdir(3)` selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getdents(2)` reinforcement:

- **Per-ENOTDIR strict** — defense against per-non-dir-fd misuse.
- **Per-count minimum** — defense against per-zero-buffer DoS.
- **Per-EOVERFLOW on ino > ULong** — defense against per-ino-truncation security bug.
- **Per-reclen alignment** — defense against per-misaligned-entry parsing exploit.
- **Per-d_off opaque cookie** — defense against per-userspace-fabricated-offset directory-state confusion.
- **Per-EFAULT no-pos-advance** — defense against per-resume-skipping after fault.
- **Per-inode_read_lock during iterate** — defense against per-rename-race buffer corruption.

### grsecurity / pax surface

- **PaX UDEREF on dirp copy_to_user** — defense against per-userptr kernel-deref; SMAP forced; bounded per-entry by d_reclen.
- **GRKERNSEC_PROC_GETPID** — getdents on `/proc` filters per-process visibility; non-root sees only own.
- **GRKERNSEC_CHROOT_FCHDIR cooperates** — getdents on a directory above chroot is impossible because the open(2) would have failed first; redundant defense.
- **PAX_USERCOPY_HARDEN on filldir copy-out** — bounded by d_reclen, whitelisted slab; defense against per-overread.
- **PAX_REFCOUNT on filp refcount during fget** — defense against per-fput-double UAF.
- **GRKERNSEC_HIDESYM on iterate_shared vtables** — symbol not in kallsyms.
- **getdents64 preferred** — grsec emits a one-shot dmesg warning when an unprivileged binary uses getdents(2) instead of getdents64(2); legacy 32-bit ino is an exploit-amplification path on >2^32-inode filesystems.
- **EOVERFLOW strict** — defense against per-ino-32-bit-truncation that could collide with another inode; grsec audits at GRKERNSEC_AUDIT_KERN.
- **Per-namespace directory filtering** — defense against per-pidns leakage via `/proc/<pid>`.
- **Per-inode-read-lock held across iterate** — defense against per-rename-race directory enumeration tampering.

