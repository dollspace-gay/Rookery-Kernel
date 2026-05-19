# Tier-5: syscall 319 — memfd_create(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`319  common  memfd_create  sys_memfd_create`)
  - mm/memfd.c (`SYSCALL_DEFINE2(memfd_create, ...)`, `memfd_file_create`, `shmem_file_setup`, `hugetlb_file_setup`)
  - include/uapi/linux/memfd.h (`MFD_CLOEXEC`, `MFD_ALLOW_SEALING`, `MFD_HUGETLB`, `MFD_NOEXEC_SEAL`, `MFD_EXEC`, `MFD_HUGE_SHIFT`, `MFD_HUGE_MASK`)
  - include/uapi/linux/fcntl.h (`F_SEAL_SEAL`, `F_SEAL_SHRINK`, `F_SEAL_GROW`, `F_SEAL_WRITE`, `F_SEAL_FUTURE_WRITE`, `F_SEAL_EXEC`)
  - fs/hugetlbfs/inode.c (`hugetlb_file_setup`)
-->

## Summary

`memfd_create(2)` is **x86_64 syscall 319** (added in Linux 3.17), the anonymous-file primitive. It returns a file descriptor referring to an unnamed in-memory file backed by tmpfs (or hugetlbfs when `MFD_HUGETLB` is set). Unlike `shm_open(3)`, the file has no filesystem entry — it lives entirely in the VFS in-memory namespace, anchored only by open fds. The fd supports the full file API (`read`, `write`, `mmap`, `ftruncate`, `pwrite`, etc.) and additionally — when `MFD_ALLOW_SEALING` is set — supports `fcntl(F_ADD_SEALS)` to impose immutability constraints on subsequent operations. Sealed memfds are the primary mechanism for safely sharing read-only memory between mutually-distrusting processes (e.g., systemd LandLock, Wayland buffers, libvirt qemu memory).

Linux 6.3+ adds `MFD_NOEXEC_SEAL` (which forces the seal `F_SEAL_EXEC` at creation, preventing `mmap(PROT_EXEC)` on the file) and `MFD_EXEC` (which explicitly permits exec mapping). These options exist because the historical default was permissive and led to JIT-spray-style code-execution attacks.

Critical for: secure IPC of fixed-size buffers (Wayland clients, qemu/kvm guest memory), language runtimes hot-loading binary blobs, gVisor / Firecracker microVM page sharing, sandbox-escape-resistant config-file passing between privileged and unprivileged processes.

## Signature

C (POSIX / man-pages):

```c
int memfd_create(const char *name, unsigned int flags);
```

glibc wrapper: `__memfd_create` → `INLINE_SYSCALL(memfd_create, 2, name, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(memfd_create, const char __user *, uname, unsigned int, flags);
```

Rookery dispatch:

```rust
pub fn sys_memfd_create(name: UserPtr<c_char>, flags: u32) -> SyscallResult<i32>;
```

## Parameters

| name  | type             | constraints                                                       | errno-on-bad |
|-------|------------------|-------------------------------------------------------------------|--------------|
| name  | `const char *`   | NUL-terminated; `<= MFD_NAME_MAX_LEN` (`249`) bytes; used only for `/proc/self/fd/N → /memfd:<name>` display | `EFAULT` / `EINVAL` |
| flags | `unsigned int`   | bitmask of allowed flags                                          | `EINVAL`     |

## Return value

- Success: a new file descriptor referring to the memfd.
- Failure: `< 0` — negated errno.

## Errors

| errno     | condition                                                                                  |
|-----------|--------------------------------------------------------------------------------------------|
| `EFAULT`  | `uname` not accessible.                                                                    |
| `EINVAL`  | `flags & ~MFD_ALL_FLAGS != 0`; `name` longer than `MFD_NAME_MAX_LEN`; `MFD_HUGETLB` set with non-zero huge-page-size selector that does not map to a configured hstate; mutually-exclusive flag combination (`MFD_NOEXEC_SEAL` with `MFD_EXEC`); `MFD_HUGETLB` without enough hugepages of the requested size. |
| `ENFILE`  | System-wide fd limit reached.                                                              |
| `EMFILE`  | Per-process `RLIMIT_NOFILE` limit reached.                                                 |
| `ENOMEM`  | Slab allocation failure for the tmpfs inode or hugetlbfs file.                             |
| `EACCES`  | LSM denial.                                                                                |
| `ENOSYS`  | Kernel built without `CONFIG_MEMFD_CREATE` (very rare).                                    |
| `EPERM`   | `MFD_HUGETLB` and `CAP_IPC_LOCK` required by sysctl (`vm.hugetlb_shm_group` not satisfied) on some configurations. |

## ABI surface (constants + flags)

`MFD_*` (from `uapi/linux/memfd.h`):

- `MFD_CLOEXEC`        `0x0001` — set `FD_CLOEXEC` on the resulting fd.
- `MFD_ALLOW_SEALING`  `0x0002` — permit `fcntl(F_ADD_SEALS, ...)` on the fd.
- `MFD_HUGETLB`        `0x0004` — back with hugetlbfs (not tmpfs); page-size selectable via `MFD_HUGE_*` bits.
- `MFD_NOEXEC_SEAL`    `0x0008` — set `F_SEAL_EXEC` at creation; `mmap(PROT_EXEC)` on the fd thereafter ⟹ `-EACCES`. (Linux 6.3+.)
- `MFD_EXEC`           `0x0010` — explicitly permit exec mapping (overrides the sysctl `vm.memfd_noexec` default). (Linux 6.3+.)
- `MFD_HUGE_SHIFT`     `26` — base bit for huge-page-size selector inside `flags`.
- `MFD_HUGE_MASK`      `0x3f`.

Composed `MFD_HUGE_*` page-size values (`= MFD_HUGETLB | (log2(size) << MFD_HUGE_SHIFT)`):

- `MFD_HUGE_64KB`  — `MFD_HUGETLB | (16 << 26)`.
- `MFD_HUGE_512KB` — `MFD_HUGETLB | (19 << 26)`.
- `MFD_HUGE_1MB`   — `MFD_HUGETLB | (20 << 26)`.
- `MFD_HUGE_2MB`   — `MFD_HUGETLB | (21 << 26)`.
- `MFD_HUGE_8MB`   — `MFD_HUGETLB | (23 << 26)`.
- `MFD_HUGE_16MB`  — `MFD_HUGETLB | (24 << 26)`.
- `MFD_HUGE_32MB`  — `MFD_HUGETLB | (25 << 26)`.
- `MFD_HUGE_256MB` — `MFD_HUGETLB | (28 << 26)`.
- `MFD_HUGE_512MB` — `MFD_HUGETLB | (29 << 26)`.
- `MFD_HUGE_1GB`   — `MFD_HUGETLB | (30 << 26)`.
- `MFD_HUGE_2GB`   — `MFD_HUGETLB | (31 << 26)`.
- `MFD_HUGE_16GB`  — `MFD_HUGETLB | (34 << 26)`.

Seals (`fcntl` constants, applicable to memfd-created fds when `MFD_ALLOW_SEALING` was set):

- `F_SEAL_SEAL`         `0x0001` — prevent further sealing.
- `F_SEAL_SHRINK`       `0x0002` — prevent shrinking via truncate / fallocate(PUNCH_HOLE).
- `F_SEAL_GROW`         `0x0004` — prevent growing.
- `F_SEAL_WRITE`        `0x0008` — prevent writes (existing writable mappings remain; future `mmap(PROT_WRITE|MAP_SHARED)` denied).
- `F_SEAL_FUTURE_WRITE` `0x0010` — prevent future writable mappings/writes; existing ones unaffected.
- `F_SEAL_EXEC`         `0x0020` — prevent `mmap(PROT_EXEC)` and `mprotect` upgrade to exec.

`MFD_ALL_FLAGS` is the OR of all valid flag bits; flags outside this mask ⟹ `-EINVAL`.

Sysctl: `vm.memfd_noexec` controls default behavior when neither `MFD_NOEXEC_SEAL` nor `MFD_EXEC` is set:

- `0` — default: exec allowed (legacy compatibility).
- `1` — emit warning; allow exec.
- `2` — force `MFD_NOEXEC_SEAL`; deny exec.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=uname`, `%rsi=flags`.
- REQ-2: `flags & ~MFD_ALL_FLAGS != 0` ⟹ `-EINVAL`.
- REQ-3: `flags & MFD_HUGETLB == 0` and `flags & (MFD_HUGE_MASK << MFD_HUGE_SHIFT) != 0` ⟹ `-EINVAL` (hugepage selector without `MFD_HUGETLB`).
- REQ-4: `flags & MFD_NOEXEC_SEAL && flags & MFD_EXEC` ⟹ `-EINVAL` (mutually exclusive).
- REQ-5: Resolve effective exec policy:
   - If `flags & MFD_NOEXEC_SEAL` ⟹ `noexec_seal = true; allow_sealing = true; initial_seals = F_SEAL_EXEC`.
   - Else if `flags & MFD_EXEC` ⟹ `noexec_seal = false; initial_seals = 0`.
   - Else (legacy): consult `sysctl_memfd_noexec`. If `2` ⟹ act like `MFD_NOEXEC_SEAL`; if `1` ⟹ act like `MFD_EXEC` after warning; if `0` ⟹ act like `MFD_EXEC`.
- REQ-6: Copy `name` from user space, bounded by `MFD_NAME_MAX_LEN + 1` (including NUL); `EFAULT` on copy failure; over-length ⟹ `-EINVAL`.
- REQ-7: Construct the in-memory `name_buf = "memfd:" + user_name` (used in `/proc/self/fd/N`'s `dname`).
- REQ-8: Allocate a new fd via `get_unused_fd_flags((flags & MFD_CLOEXEC) ? O_CLOEXEC : 0)` ⟹ `-EMFILE` / `-ENFILE`.
- REQ-9: Create the backing file:
   - If `flags & MFD_HUGETLB`:
     - Look up `hstate` by selector bits; missing ⟹ `-EINVAL`.
     - `hugetlb_file_setup(name_buf, 0, VM_NORESERVE, HUGETLB_ANONHUGE_INODE, page_size_log)`.
   - Else:
     - `shmem_file_setup(name_buf, 0, VM_NORESERVE)` — anonymous tmpfs inode.
- REQ-10: Set `file->f_mode |= FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE;`.
- REQ-11: If `flags & MFD_ALLOW_SEALING` or `noexec_seal` ⟹ `file->f_inode->i_flags |= S_PRIVATE; shmem_get_inode_info(inode)->seals = initial_seals;`.
- REQ-12: Else ⟹ inode is *not* seal-capable: `fcntl(F_ADD_SEALS)` ⟹ `-EPERM`.
- REQ-13: `fd_install(fd, file)` and return `fd`.
- REQ-14: For hugetlb path: the per-file page-size is fixed at creation; `ftruncate` must be huge-aligned.
- REQ-15: `MFD_NOEXEC_SEAL` implies `MFD_ALLOW_SEALING` (else the seal could not be set in REQ-11).
- REQ-16: Subsequent `mmap(memfd, ..., PROT_EXEC)` checks `seals & F_SEAL_EXEC`; if set ⟹ `-EACCES`.
- REQ-17: Subsequent `mmap(memfd, ..., PROT_WRITE | MAP_SHARED)` checks `seals & F_SEAL_WRITE`; if set ⟹ `-EPERM`.
- REQ-18: LSM hook (`security_memfd_create` analog via `security_file_alloc` + `security_inode_init_security`) consulted; denial ⟹ `-EACCES`.

## Acceptance Criteria

- [ ] AC-1: `fd = memfd_create("test", 0)` returns a valid fd; `read(fd, buf, n) == 0` (empty file).
- [ ] AC-2: `memfd_create("test", MFD_CLOEXEC)` sets `O_CLOEXEC` on the fd (`fcntl(fd, F_GETFD) & FD_CLOEXEC != 0`).
- [ ] AC-3: `memfd_create("test", MFD_ALLOW_SEALING)` allows `fcntl(fd, F_ADD_SEALS, F_SEAL_WRITE) == 0`.
- [ ] AC-4: `memfd_create("test", 0)` does NOT allow sealing: `fcntl(fd, F_ADD_SEALS, F_SEAL_WRITE) == -EPERM`.
- [ ] AC-5: `memfd_create("test", MFD_HUGETLB | MFD_HUGE_2MB)` succeeds when 2MB hugepages are available; `ftruncate(fd, 2MB)` aligns properly.
- [ ] AC-6: `memfd_create("test", MFD_NOEXEC_SEAL)` then `mmap(NULL, len, PROT_EXEC, MAP_SHARED, fd, 0) == -EACCES`.
- [ ] AC-7: `memfd_create("test", MFD_NOEXEC_SEAL | MFD_EXEC) == -EINVAL`.
- [ ] AC-8: `memfd_create("test", 0xDEADBEEF) == -EINVAL` (unknown bits).
- [ ] AC-9: `memfd_create("a" * 500, 0) == -EINVAL` (name too long).
- [ ] AC-10: `memfd_create(NULL, 0) == -EFAULT`.
- [ ] AC-11: After AC-3, writing then sealing with `F_SEAL_WRITE`, subsequent `write(fd, ...) == -EPERM`.
- [ ] AC-12: With `vm.memfd_noexec == 2`, `memfd_create("test", 0)` behaves as if `MFD_NOEXEC_SEAL` was set.
- [ ] AC-13: `/proc/self/fd/N` symlink target reads `/memfd:test (deleted)`.

## Architecture

```
struct MemfdCreateArgs { name: UserPtr<c_char>, flags: u32 }
```

`sys_memfd_create(args) -> i32`:

1. Validate `flags & ~MFD_ALL_FLAGS == 0` ⟹ else `-EINVAL`.
2. Reject `MFD_NOEXEC_SEAL | MFD_EXEC` ⟹ `-EINVAL`.
3. Reject hugepage selector without `MFD_HUGETLB` ⟹ `-EINVAL`.
4. Resolve exec policy ⟹ compute `(noexec_seal, allow_sealing, initial_seals)`.
5. Copy `name` from user; bound `MFD_NAME_MAX_LEN`.
6. Compose `name_buf = "memfd:" + name`.
7. `fd = get_unused_fd_flags(MFD_CLOEXEC ? O_CLOEXEC : 0)?;`
8. If `flags & MFD_HUGETLB`:
   - `file = hugetlb_file_setup(name_buf, 0, VM_NORESERVE, HUGETLB_ANONHUGE_INODE, log2_page_size);`
9. Else:
   - `file = shmem_file_setup(name_buf, 0, VM_NORESERVE);`
10. `file->f_mode |= FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE;`
11. If `allow_sealing` ⟹ set `inode->i_flags |= S_PRIVATE; shmem_inode_info(inode)->seals = initial_seals;`
12. `fd_install(fd, file);`
13. Return `fd`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | `flags & ~MFD_ALL_FLAGS == 0`. |
| `mutually_exclusive_flags` | INVARIANT | `MFD_NOEXEC_SEAL && MFD_EXEC ⟹ -EINVAL`. |
| `noexec_seal_sets_seal` | INVARIANT | `MFD_NOEXEC_SEAL` ⟹ inode's `seals & F_SEAL_EXEC` set at return. |
| `cloexec_propagates` | INVARIANT | `MFD_CLOEXEC` ⟹ `fd_flags & FD_CLOEXEC` set. |
| `name_bounded` | INVARIANT | Resolved `name_buf` length `<= MFD_NAME_MAX_LEN + 6`. |
| `fd_installed_atomically` | INVARIANT | On error path, `fd` released via `put_unused_fd`. |

### Layer 2: TLA+

`uapi/syscalls/memfd_create.tla`:
- States: `{fd_table, inodes, seals_map, sysctl_memfd_noexec}`.
- Actions: `Create`, `CreateWithSeal`, `CreateHuge`, `CreateInvalidFlags`, `SeticeAfterCreate`, `MmapExecOnSealed`.
- Properties:
  - `safety_noexec_seal_is_immutable_after_creation`,
  - `safety_seal_capability_iff_allow_sealing_or_noexec_seal`,
  - `liveness_create_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `hugetlb_file_setup` selects matching hstate by `log2_page_size` | `Mm::sys_memfd_create` |
| Seal-capable inode is not marked dirty until first write | `Mm::shmem_file_setup` |
| Fd table entry installed iff file successfully constructed | `Mm::sys_memfd_create` |

### Layer 4: Verus/Creusot functional

`memfd_create(name, flags) ≡ Linux man-page semantics (no POSIX standard).`

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`memfd_create(2)` reinforcement:

- **Per `MFD_NOEXEC_SEAL`** — denial of exec mapping at creation; defense against memfd-based JIT-spray code-execution.
- **Per `MFD_ALLOW_SEALING` opt-in** — sealing is opt-in; non-sealable memfds cannot accidentally become tampered.
- **Per `MFD_CLOEXEC` opt-in** — modern code should always set this; defense against fd-leak via `execve`.
- **Per name-length bound** — `MFD_NAME_MAX_LEN` (`249`) prevents arbitrarily long `/proc/self/fd/N` symlink targets.
- **Per `vm.memfd_noexec` sysctl** — sysadmin can force-deny exec on all new memfds (`= 2`); defense against legacy code that omits `MFD_NOEXEC_SEAL`.
- **Per `S_PRIVATE` flag** — sealed inodes are not visible via normal fs introspection (no `mount`-walk reveals them).
- **Per LSM hook integration** — SELinux / AppArmor / Smack observe memfd creation.
- **Per hugetlb resource check** — `MFD_HUGETLB` consults configured hstate count and `vm.hugetlb_shm_group` membership.
- **Per `FD_CLOEXEC` default-not-set warning** — modern style guides treat absence of `MFD_CLOEXEC` as a bug.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — sealed `F_SEAL_EXEC` memfds cannot be `mmap(PROT_EXEC)`'d; `mprotect` upgrade from RW to RX denied. Defense against memfd JIT-spray bypass of W^X.
- **PAX_PAGEEXEC** — file-backed PROT_EXEC mappings against memfds enforce NX-bit on non-exec pages. Defense against page-confusion.
- **PAX_NOEXEC** — anonymous-like memfds (`MFD_NOEXEC_SEAL` default under PaX) cannot acquire `VM_EXEC`. Under grsec, the default of `vm.memfd_noexec` is forced to `2`.
- **PAX_RANDMMAP** — memfd-backed mappings still subject to ASLR via `get_unmapped_area`.
- **PAX_RANDEXEC** — file-backed PROT_EXEC mappings (when allowed) are randomized.
- **GRKERNSEC_RWXMAP_LOG** — any memfd `mmap(PROT_WRITE|PROT_EXEC)` audited.
- **PAX_REFCOUNT** — `file->f_count` and `inode->i_count` saturating refcounts.
- **PAX_UDEREF** — `uname` validated via SMAP; PaX user-deref check ensures bounded user-string copy.
- **PAX_MEMORY_SANITIZE** — memfd pages sanitized on free (tmpfs page recycle path).
- **GRKERNSEC_BRUTE** — repeated memfd_create failures (e.g., RLIMIT_NOFILE brute) detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at memfd_create syscall entry.
- **`MFD_NOEXEC_SEAL` mandatory under suid** — grsec enforces that any setuid binary calling `memfd_create` without `MFD_NOEXEC_SEAL` (or `MFD_EXEC` explicitly) is denied (`-EACCES`) or audited. Defense against suid-helper-loaded-shellcode via memfd JIT-spray.
- **`MFD_NOEXEC_SEAL` mandatory under container** — grsec optionally enforces `MFD_NOEXEC_SEAL` in containers (via cgroup or namespace policy).
- **System-wide memfd-count cap** — grsec optional ceiling on aggregate memfd inodes across the system; defense against memfd-inode-exhaustion DoS.
- **Per name-bounded inode-leak protection** — `name` is not stored in the inode's `i_name` field for sealed memfds, denying side-channel introspection of in-flight memfd usage.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `memfd_secret(2)` (separate Tier-5 — `memfd_secret.md`).
- `fcntl(F_ADD_SEALS)` / `fcntl(F_GET_SEALS)` (separate Tier-5 if expanded).
- tmpfs internals (`shmem_file_setup`).
- hugetlbfs internals (`hugetlb_file_setup`).
- `shm_open(3)` POSIX-shm comparison.
- Implementation code.
