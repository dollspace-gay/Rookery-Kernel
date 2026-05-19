# Tier-5: syscall 447 — memfd_secret(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`447  common  memfd_secret  sys_memfd_secret`)
  - mm/secretmem.c (`SYSCALL_DEFINE1(memfd_secret, ...)`, `secretmem_init_fs_context`, `secretmem_fault`, `secretmem_isolate_page`)
  - include/uapi/linux/magic.h (`SECRETMEM_MAGIC`)
  - include/linux/secretmem.h (`vma_is_secretmem`)
  - include/uapi/linux/fcntl.h (no new seals; secretmem is implicitly sealed)
-->

## Summary

`memfd_secret(2)` is **x86_64 syscall 447** (added in Linux 5.14), the **secret-memory** primitive. It returns a file descriptor referring to a special anonymous file whose pages, once mapped, are **removed from the kernel's direct mapping**. The mapped pages remain accessible only via the process's own page tables; they are invisible to `/dev/kmem`, kernel reads via `copy_to_kernel`-style routines, kernel debuggers (kgdb), and crucially to any in-kernel data leak (Spectre/Meltdown-style transient execution attacks cannot extract them through kernel space because there is no kernel-space view). Unlike `memfd_create(MFD_NOEXEC_SEAL)`, which prevents code execution but still exposes pages in the kernel direct map, `memfd_secret` provides physical-page isolation.

The fd supports `ftruncate` (to size) and `mmap`. It does **not** support `read`/`write` (the contents must be accessed via the user mapping). Multiple `mmap`s on the same secretmem fd within the same process share underlying pages; cross-process sharing requires explicit fd passing (e.g., via `SCM_RIGHTS`). The kernel disables this feature by default (`CONFIG_SECRETMEM=y` and `secretmem.enable=1` boot parameter required); the sysctl `vm.secretmem_enable` (when present) further gates runtime use.

Critical for: cryptographic key material that should not be readable via any kernel-space path (defense against Spectre v1/v2/v4/L1TF/RIDL kernel leak), in-process secret managers, secure enclaves that lack hardware (SGX/SEV) support.

## Signature

C (POSIX / man-pages):

```c
int memfd_secret(unsigned int flags);
```

glibc wrapper: none historically; use `syscall(SYS_memfd_secret, flags)`. Recent glibc versions expose `memfd_secret()` directly.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(memfd_secret, unsigned int, flags);
```

Rookery dispatch:

```rust
pub fn sys_memfd_secret(flags: u32) -> SyscallResult<i32>;
```

## Parameters

| name  | type           | constraints                                                       | errno-on-bad |
|-------|----------------|-------------------------------------------------------------------|--------------|
| flags | `unsigned int` | `0` or `FD_CLOEXEC` (`0x1`)                                       | `EINVAL`     |

## Return value

- Success: a new file descriptor referring to the secretmem file.
- Failure: `< 0` — negated errno.

## Errors

| errno    | condition                                                                                  |
|----------|--------------------------------------------------------------------------------------------|
| `EINVAL` | `flags & ~FD_CLOEXEC != 0` (unknown flag bits).                                            |
| `ENOSYS` | Kernel built without `CONFIG_SECRETMEM`, or `secretmem.enable=0` boot parameter, or runtime sysctl disabled. |
| `ENFILE` | System-wide fd limit reached.                                                              |
| `EMFILE` | Per-process `RLIMIT_NOFILE` limit reached.                                                 |
| `ENOMEM` | Slab allocation failure for the secretmem inode/file.                                      |
| `EACCES` | LSM denial.                                                                                |

## ABI surface (constants + flags)

`flags`:

- `FD_CLOEXEC` `0x0001` — set close-on-exec on the resulting fd. (Note: in `memfd_secret`, this is the **only** valid flag, and uses the `FD_CLOEXEC` constant from `fcntl.h` — not `MFD_CLOEXEC`. The kernel accepts the same bit value.)

Reserved future-flag bits: any other bit ⟹ `-EINVAL`.

Filesystem magic:

- `SECRETMEM_MAGIC` `0x5345434d` (`"SECM"`) — used to identify the in-memory secretmem filesystem.

Property bits / behavior:

- File implements `file_operations.mmap` only; `read` / `write` return `-EINVAL`.
- File is implicitly **sealed against shrinking and growing via writes**; only `ftruncate` can resize.
- Pages allocated for the file are **not in the kernel direct map** (kernel-virtual-to-physical map): `set_direct_map_invalid_noflush(page)` is called when each page is allocated.
- Pages are **non-migratable** (`secretmem_isolate_page` returns failure for migration callers); avoids physical-page movement that would create a transient kernel mapping.
- Pages are **not swappable** (`folio->mapping` flagged `AS_NO_SWAPCACHE`-equivalent via address_space hooks).
- Pages are **mlocked-on-fault implicitly** (folio added to unevictable LRU).
- Pages are **not dumpable** (VMA carries `VM_DONTDUMP` semantics; excluded from core dumps).
- Pages are **not forkable** (VMA carries `VM_DONTCOPY`-equivalent; `fork(2)` does not duplicate the mapping; child has the fd but mapping the same fd in child yields a new disjoint page set).

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=flags`.
- REQ-2: `flags & ~FD_CLOEXEC != 0` ⟹ `-EINVAL`.
- REQ-3: `secretmem_enable` (set via boot param `secretmem.enable=1` and gated by `CONFIG_SECRETMEM=y`) must be true; else `-ENOSYS`.
- REQ-4: Allocate fd: `fd = get_unused_fd_flags((flags & FD_CLOEXEC) ? O_CLOEXEC : 0)?;`
- REQ-5: Allocate the secretmem inode via the internal `secretmem_mnt` mount: `file = alloc_file_pseudo(inode, secretmem_mnt, "secretmem", O_RDWR, &secretmem_fops);`
- REQ-6: `file->f_flags |= O_LARGEFILE;`
- REQ-7: `file->f_inode->i_size = 0;`
- REQ-8: `fd_install(fd, file)` and return `fd`.
- REQ-9: Subsequent `ftruncate(fd, size)` sets the file size; size must be page-aligned (else `-EINVAL`).
- REQ-10: Subsequent `mmap(fd, ...)` calls `secretmem_mmap`:
   - VMA flags forced to `VM_LOCKED | VM_DONTEXPAND | VM_DONTDUMP`.
   - VMA `vm_ops` set to `secretmem_vm_ops` (provides `secretmem_fault`).
- REQ-11: `mmap(fd, ..., PROT_EXEC)` is denied: VMA cannot acquire `VM_EXEC` (checked in `secretmem_mmap`); ⟹ `-EINVAL`.
- REQ-12: `mmap(fd, ..., MAP_PRIVATE)` is denied: secretmem is shared-only; ⟹ `-EINVAL`.
- REQ-13: On page fault (`secretmem_fault`):
   - Allocate a fresh page via `alloc_pages(GFP_HIGHUSER | __GFP_ZERO, 0)`.
   - `set_direct_map_invalid_noflush(page)` — remove from kernel direct map.
   - Insert into the file's address space; map into the user PTE.
- REQ-14: On page free (refcount drops to zero, e.g., munmap or fd close):
   - `set_direct_map_default_noflush(page)` — restore direct map.
   - Release page to the allocator.
- REQ-15: `migrate_pages` consults `secretmem_isolate_page` and returns `-EBUSY` ⟹ secretmem pages never migrate.
- REQ-16: `vmsplice(fd, ...)` is denied (secretmem fd does not implement `splice_read`/`splice_write`).
- REQ-17: `userfaultfd_register` on a secretmem VMA is denied (`secretmem_mmap` rejects setting up `vma->vm_userfaultfd_ctx`).
- REQ-18: `fork(2)`: secretmem VMA inherited as `VM_DONTCOPY` ⟹ child gets a VMA but with no PTEs (effectively unmapped); child can re-`mmap` the same fd to obtain its own copy.
- REQ-19: `RLIMIT_MEMLOCK` is consulted at mmap time (because `VM_LOCKED` is forced); over-budget ⟹ `mmap` fails with `-EAGAIN` / `-ENOMEM`.
- REQ-20: LSM hook (`security_file_alloc` analog) consulted at creation; denial ⟹ `-EACCES`.

## Acceptance Criteria

- [ ] AC-1: `fd = memfd_secret(0)` returns a valid fd; `fstatfs(fd, &sb)` yields `sb.f_type == SECRETMEM_MAGIC`.
- [ ] AC-2: `memfd_secret(FD_CLOEXEC)` sets close-on-exec.
- [ ] AC-3: `memfd_secret(0xDEADBEEF) == -EINVAL`.
- [ ] AC-4: `read(fd, buf, n) == -EINVAL`; `write(fd, buf, n) == -EINVAL` — direct I/O denied.
- [ ] AC-5: `ftruncate(fd, 4096) == 0`; `mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)` returns a valid address.
- [ ] AC-6: AC-5 mapping is automatically locked (`/proc/PID/smaps` shows `Locked: 4 kB`).
- [ ] AC-7: `mmap(NULL, 4096, PROT_EXEC, MAP_SHARED, fd, 0) == -EINVAL`.
- [ ] AC-8: `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, fd, 0) == -EINVAL`.
- [ ] AC-9: After AC-5, writing to the mapping then `kmem_cache_alloc` from another driver cannot return a page mapped via the secret area (verified by direct-map sweep).
- [ ] AC-10: `vmsplice(fd, ...) == -EINVAL`.
- [ ] AC-11: `userfaultfd_register` on the secretmem VMA ⟹ `-EINVAL`.
- [ ] AC-12: `fork(2)` after AC-5: child cannot read the secret pages without re-mapping fd; child's VMA contains no resident pages.
- [ ] AC-13: With `secretmem.enable=0` at boot, `memfd_secret(0) == -ENOSYS`.
- [ ] AC-14: Core dump of process with secretmem VMA does NOT include the secret pages.

## Architecture

```
struct MemfdSecretArgs { flags: u32 }
```

`sys_memfd_secret(args) -> i32`:

1. `if !secretmem_enable { return -ENOSYS; }`
2. `if args.flags & !FD_CLOEXEC != 0 { return -EINVAL; }`
3. `fd = get_unused_fd_flags(if args.flags & FD_CLOEXEC { O_CLOEXEC } else { 0 })?;`
4. `inode = alloc_anon_inode(secretmem_mnt->mnt_sb);`
5. `file = alloc_file_pseudo(inode, secretmem_mnt, "secretmem", O_RDWR, &secretmem_fops);`
6. `file->f_flags |= O_LARGEFILE;`
7. `fd_install(fd, file);`
8. Return `fd`.

`secretmem_fops = { .mmap = secretmem_mmap, .release = secretmem_release }` (no `.read` / `.write`).

`secretmem_mmap(file, vma)`:

1. Validate `vma->vm_flags & VM_EXEC == 0` ⟹ else `-EINVAL`.
2. Validate `vma->vm_flags & VM_SHARED != 0` ⟹ else `-EINVAL` (MAP_PRIVATE denied).
3. Validate `(vma->vm_flags & VM_MAYEXEC) == 0`.
4. Set `vma->vm_flags |= VM_LOCKED | VM_DONTEXPAND | VM_DONTDUMP;`
5. `vma->vm_ops = &secretmem_vm_ops;`
6. Account `mm->locked_vm += vma_pages(vma);` (respecting `RLIMIT_MEMLOCK`).
7. Return `0`.

`secretmem_fault(vmf)`:

1. Allocate page: `page = alloc_pages(GFP_HIGHUSER | __GFP_ZERO, 0);` ⟹ `VM_FAULT_OOM` on failure.
2. `set_direct_map_invalid_noflush(page);`
3. `flush_tlb_kernel_range(page_addr, page_addr+PAGE_SIZE);` (lazy; batched).
4. Insert into address space: `add_to_page_cache_lru(folio, &inode->i_mapping, index, GFP_KERNEL);`
5. Map into PTE: `vmf->page = page;` and return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | `flags & ~FD_CLOEXEC == 0`. |
| `secretmem_enabled_at_call` | INVARIANT | `secretmem_enable == true` at syscall entry. |
| `no_kernel_direct_map_on_resident_pages` | INVARIANT | For every resident secret page, `kernel_text_addr(page)` returns false. |
| `mapping_is_shared_only` | INVARIANT | All secretmem VMAs have `VM_SHARED` and lack `VM_EXEC`. |
| `mapping_is_locked` | INVARIANT | All secretmem VMAs have `VM_LOCKED`. |
| `non_migratable` | INVARIANT | `migrate_pages` on secretmem pages returns `-EBUSY`. |
| `no_swap` | INVARIANT | Secret pages never reach the swap path. |

### Layer 2: TLA+

`uapi/syscalls/memfd_secret.tla`:
- States: `{secret_fds, secret_pages, direct_map, vmas, kernel_mappings}`.
- Actions: `Create`, `Mmap`, `Fault`, `Munmap`, `ForkInherit`, `KernelRead`, `KernelMigrate`.
- Properties:
  - `safety_secret_pages_invisible_to_kernel_direct_map`,
  - `safety_no_exec_mapping`,
  - `safety_no_migrate`,
  - `safety_no_swap`,
  - `liveness_secretmem_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `set_direct_map_invalid_noflush` precedes any user-space mapping | `secretmem_fault` |
| `set_direct_map_default_noflush` precedes page-allocator release | `secretmem_release` per-page |
| `vma->vm_flags & (VM_EXEC|VM_MAYEXEC) == 0` for secretmem VMAs | `secretmem_mmap` |
| `RLIMIT_MEMLOCK` consulted at mmap time | `secretmem_mmap` |

### Layer 4: Verus/Creusot functional

`memfd_secret(flags) ≡ Linux man-page semantics (no POSIX standard).`

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`memfd_secret(2)` reinforcement:

- **Per kernel-direct-map removal** — `set_direct_map_invalid_noflush` is the core invariant; defense against kernel-side reads (Spectre, Meltdown, kgdb, /dev/kmem, dump_kernel_pte_at).
- **Per `VM_LOCKED` forced** — secret pages never paged out; defense against swap-leak attacks (e.g., suspend-to-disk reading swap file).
- **Per `VM_DONTDUMP` forced** — secret pages excluded from core dumps; defense against post-mortem extraction.
- **Per non-migratable** — secret pages never relocated; defense against transient kernel mapping during compaction.
- **Per `MAP_SHARED`-only** — denies `MAP_PRIVATE` COW which would create a parallel page that may not have direct-map removal.
- **Per `PROT_EXEC` denial** — defense against using secretmem as a JIT-arena bypass of W^X.
- **Per `read`/`write` denial** — denies kernel-mediated I/O on the file; the data must transit only through user mappings.
- **Per `secretmem.enable` opt-in** — feature requires explicit boot parameter; defense against unintended secret-page-direct-map fragmentation (which degrades direct-map performance due to 4KiB page splitting).
- **Per `RLIMIT_MEMLOCK` enforcement** — at mmap time; defense against unprivileged DoS via secret page reservation.
- **Per `userfaultfd_register` denial** — defense against UFD-based secret extraction via fault injection.
- **Per `vmsplice` denial** — defense against splice-mediated kernel-page-pipe extraction.
- **Per LSM hook** — SELinux / AppArmor / Smack observe secretmem creation.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — secretmem VMAs are non-exec by construction; `mprotect` cannot grant `PROT_EXEC`; defense against memfd_secret-based code injection.
- **PAX_PAGEEXEC** — NX bit set on all secretmem PTEs.
- **PAX_NOEXEC** — secretmem is treated as anonymous-like for exec policy.
- **PAX_RANDMMAP** — secretmem mappings still subject to ASLR via `get_unmapped_area`.
- **PAX_RANDEXEC** — irrelevant (no exec).
- **GRKERNSEC_RWXMAP_LOG** — irrelevant (no exec ever possible).
- **PAX_REFCOUNT** — `file->f_count` and `inode->i_count` saturating refcounts.
- **PAX_UDEREF** — no user pointer dereferenced.
- **PAX_MEMORY_SANITIZE** — secret pages **must** be sanitized at free (zero-on-free); grsec extends to enforce immediate page-poisoning on `secretmem_release`. Defense against post-free residue extraction via subsequent page allocation.
- **GRKERNSEC_BRUTE** — repeated memfd_secret failures (e.g., RLIMIT_MEMLOCK brute) detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at memfd_secret syscall entry.
- **`secretmem.enable` mandatory under hardened kernel** — grsec forces `secretmem.enable=1` and treats `=0` as misconfiguration for hardened deployments (with audit).
- **Defense against direct-map TLB-flush-delay attack** — grsec ensures `set_direct_map_invalid_noflush` is followed by a synchronous TLB flush (`flush_tlb_all`) before the page is exposed to user space; defense against stale-TLB transient extraction.
- **Per `MFD_NOEXEC_SEAL`-equivalent strict enforcement** — secretmem is semantically `MFD_NOEXEC_SEAL` with additional protections; grsec audits any attempt to escalate.
- **Defense against L1TF / MDS via kernel-direct-map** — by removing the direct map entirely, secretmem closes the transient-execution side-channel of L1TF on the kernel side. Grsec extends with stronger flush_l1d() on context switch when secretmem is in use.
- **System-wide secret-page cap** — grsec optional ceiling on aggregate secretmem pages; defense against direct-map fragmentation DoS (each secretmem page forces a 4KiB-grained split of the kernel direct map, hurting kernel TLB density).

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `memfd_create(2)` (separate Tier-5 — `memfd_create.md`).
- `set_direct_map_*` page-table internals.
- Kernel direct-map TLB-fragmentation cost analysis.
- Spectre / Meltdown / L1TF / MDS mitigation overall (separate hardening doc).
- `/dev/kmem` and `dump_kernel_pte_at` (debug interfaces).
- SGX / SEV / TDX hardware enclaves comparison.
- Implementation code.
