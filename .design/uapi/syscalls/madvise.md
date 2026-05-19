# Tier-5: syscall 28 — madvise(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`28  common  madvise  sys_madvise`)
  - mm/madvise.c (`SYSCALL_DEFINE3(madvise, ...)`, `do_madvise`, `madvise_vma_behavior`, `madvise_walk_vmas`)
  - include/uapi/asm-generic/mman-common.h (`MADV_NORMAL`, `MADV_RANDOM`, `MADV_SEQUENTIAL`, `MADV_WILLNEED`, `MADV_DONTNEED`, `MADV_FREE`, `MADV_REMOVE`, `MADV_DONTFORK`, `MADV_DOFORK`, `MADV_HWPOISON`, `MADV_MERGEABLE`, `MADV_UNMERGEABLE`, `MADV_HUGEPAGE`, `MADV_NOHUGEPAGE`, `MADV_DONTDUMP`, `MADV_DODUMP`, `MADV_WIPEONFORK`, `MADV_KEEPONFORK`, `MADV_COLD`, `MADV_PAGEOUT`, `MADV_POPULATE_READ`, `MADV_POPULATE_WRITE`, `MADV_DONTNEED_LOCKED`, `MADV_COLLAPSE`, `MADV_GUARD_INSTALL`, `MADV_GUARD_REMOVE`)
-->

## Summary

`madvise(2)` is **x86_64 syscall 28**, the generic memory-region advice / control multiplex. It tells the kernel how the process intends to use the pages in `[addr, addr+length)` (or — for `MADV_*` advice values that *act* rather than merely advise — performs a specific operation on the range). The advice ranges from soft hints that change page-cache prefetch behavior (`MADV_SEQUENTIAL`, `MADV_RANDOM`, `MADV_WILLNEED`) to destructive operations (`MADV_DONTNEED` zaps PTEs, `MADV_REMOVE` punches a hole in a tmpfs file, `MADV_HWPOISON` simulates a memory error). The kernel dispatches on `advice` first, then walks the VMA tree of the calling `mm`, applying per-VMA behavior overrides via `vma->vm_flags`. Some advice values require `CAP_SYS_ADMIN` (`MADV_HWPOISON`, `MADV_SOFT_OFFLINE`); others are unprivileged but can be denied by LSMs.

Critical for: glibc tcache cleanup (`MADV_DONTNEED`), jemalloc / mimalloc trim (`MADV_FREE`), every database with explicit page-prefetch (`MADV_WILLNEED`), every guard-page allocator (`MADV_GUARD_INSTALL`), every hugepage-aware workload (`MADV_HUGEPAGE`/`MADV_NOHUGEPAGE`/`MADV_COLLAPSE`).

## Signature

C (POSIX / man-pages):

```c
int madvise(void *addr, size_t length, int advice);
```

glibc wrapper: `__madvise` / `posix_madvise` → `INLINE_SYSCALL(madvise, 3, addr, length, advice)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(madvise, unsigned long, start, size_t, len_in, int, behavior);
```

Rookery dispatch:

```rust
pub fn sys_madvise(start: u64, len: usize, behavior: i32) -> SyscallResult<i32>;
```

## Parameters

| name     | type             | constraints                                                       | errno-on-bad |
|----------|------------------|-------------------------------------------------------------------|--------------|
| start    | `unsigned long`  | page-aligned (`start & ~PAGE_MASK == 0`)                          | `EINVAL`     |
| len      | `size_t`         | `>= 0`; rounded up via `PAGE_ALIGN(len)`; sum must not overflow   | `EINVAL`     |
| behavior | `int`            | one of the `MADV_*` advice constants                              | `EINVAL`     |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.
- For destructive operations (`MADV_REMOVE`, `MADV_DONTNEED`, `MADV_FREE`), partial application is **possible** — the man page documents that on partial failure the call may have already applied to a prefix.

## Errors

| errno     | condition                                                                                  |
|-----------|--------------------------------------------------------------------------------------------|
| `EINVAL`  | `start` not page-aligned; `behavior` not a recognized advice value; `behavior` incompatible with VMA (e.g., `MADV_REMOVE` on non-shared/non-tmpfs); `MADV_HWPOISON` on a hugetlb VMA without support; `addr + len` overflow. |
| `EACCES`  | `MADV_REMOVE` on a sealed memfd (`F_SEAL_WRITE` and similar); `MADV_DONTNEED_LOCKED` without locked VMA; LSM denial. |
| `EAGAIN`  | `MADV_WILLNEED` on a swap-backed page and `RLIMIT_MEMLOCK` exceeded after read-ahead. |
| `EBADF`   | `MADV_REMOVE` on a VMA whose backing file does not support hole-punch (`fallocate(FALLOC_FL_PUNCH_HOLE)` not supported). |
| `EIO`     | `MADV_WILLNEED` page-cache read returned I/O error.                                       |
| `ENOMEM`  | Some address in range is not mapped (`-ENOMEM` returned only for advice values that require fully-mapped range, e.g., `MADV_WILLNEED`, `MADV_DONTNEED`, `MADV_FREE`); allocation failure during `MADV_POPULATE_*`; `RLIMIT_MEMLOCK` exceeded. |
| `EPERM`   | `MADV_HWPOISON` / `MADV_SOFT_OFFLINE` without `CAP_SYS_ADMIN`; LSM denial.                 |
| `EHWPOISON` | `MADV_POPULATE_*` encountered a hardware-poisoned page.                                 |
| `EFAULT`  | `MADV_POPULATE_*` hit a permanent fault (e.g., `SIGBUS` on truncated file).               |
| `EBUSY`   | `MADV_COLLAPSE` failed to acquire a transparent hugepage right now (retryable).            |

## ABI surface — full `MADV_*` table

Values from `include/uapi/asm-generic/mman-common.h` and `include/uapi/asm-generic/mman.h`:

| value | name                  | semantics                                                                         |
|-------|-----------------------|-----------------------------------------------------------------------------------|
| `0`   | `MADV_NORMAL`         | Default behavior; cancels any prior hint.                                         |
| `1`   | `MADV_RANDOM`         | Expect random access; disable read-ahead.                                         |
| `2`   | `MADV_SEQUENTIAL`     | Expect sequential access; aggressive read-ahead; aggressive page-out behind.      |
| `3`   | `MADV_WILLNEED`       | Pre-fault / pre-read pages.                                                       |
| `4`   | `MADV_DONTNEED`       | Zap PTEs in range; next access re-faults from backing or zero-page. Destructive.  |
| `8`   | `MADV_FREE`           | Lazy free anonymous private pages; mark `PG_lazyfree`; kernel may reclaim later.  |
| `9`   | `MADV_REMOVE`         | Hole-punch in shared file mapping (`fallocate(PUNCH_HOLE|KEEP_SIZE)`).            |
| `10`  | `MADV_DONTFORK`       | Child does not inherit this mapping (set `VM_DONTCOPY`).                          |
| `11`  | `MADV_DOFORK`         | Cancel `MADV_DONTFORK`.                                                           |
| `12`  | `MADV_MERGEABLE`      | KSM may merge identical pages (sets `VM_MERGEABLE`).                              |
| `13`  | `MADV_UNMERGEABLE`    | KSM stop merging; un-merge.                                                       |
| `14`  | `MADV_HUGEPAGE`       | Prefer THP for this range (`VM_HUGEPAGE`).                                        |
| `15`  | `MADV_NOHUGEPAGE`     | Disable THP for this range (`VM_NOHUGEPAGE`).                                     |
| `16`  | `MADV_DONTDUMP`       | Exclude from core dump (set `VM_DONTDUMP`).                                       |
| `17`  | `MADV_DODUMP`         | Cancel `MADV_DONTDUMP`.                                                           |
| `18`  | `MADV_WIPEONFORK`     | Children see zeros for this range after `fork(2)` (set `VM_WIPEONFORK`).          |
| `19`  | `MADV_KEEPONFORK`     | Cancel `MADV_WIPEONFORK`.                                                         |
| `20`  | `MADV_COLD`           | Mark range cold; reclaim early (move to inactive LRU).                            |
| `21`  | `MADV_PAGEOUT`        | Force page-out / swap-out without zap.                                            |
| `22`  | `MADV_POPULATE_READ`  | Pre-fault for read (read-faults entire range; like `MAP_POPULATE` post-hoc).      |
| `23`  | `MADV_POPULATE_WRITE` | Pre-fault for write (write-faults entire range).                                  |
| `24`  | `MADV_DONTNEED_LOCKED`| Like `MADV_DONTNEED` but only on `VM_LOCKED` ranges; pre-faults zero pages keeping lock count. |
| `25`  | `MADV_COLLAPSE`       | Synchronous THP collapse for this range.                                          |
| `100` | `MADV_HWPOISON`       | Simulate hardware memory error on covered pages (root-only).                      |
| `101` | `MADV_SOFT_OFFLINE`   | Soft-offline pages: migrate then mark not-reusable (root-only).                   |
| `102` | `MADV_GUARD_INSTALL`  | Install guard PTEs (signal SIGSEGV on touch) without consuming swap/anonymous backing. |
| `103` | `MADV_GUARD_REMOVE`   | Remove guard PTEs installed via `MADV_GUARD_INSTALL`.                              |

Internal classification:

- **Behavior-only advice** (just sets/clears `vma->vm_flags` bits): `NORMAL`, `RANDOM`, `SEQUENTIAL`, `HUGEPAGE`, `NOHUGEPAGE`, `MERGEABLE`, `UNMERGEABLE`, `DONTFORK`, `DOFORK`, `DONTDUMP`, `DODUMP`, `WIPEONFORK`, `KEEPONFORK`.
- **Page-cache advice** (consults page cache): `WILLNEED`.
- **Destructive PTE-zap**: `DONTNEED`, `FREE`, `REMOVE`, `DONTNEED_LOCKED`.
- **Reclaim/swap actions**: `COLD`, `PAGEOUT`.
- **Pre-fault actions**: `POPULATE_READ`, `POPULATE_WRITE`.
- **Hugepage**: `COLLAPSE`.
- **Hardware error simulation** (root): `HWPOISON`, `SOFT_OFFLINE`.
- **Guard PTE**: `GUARD_INSTALL`, `GUARD_REMOVE` (Linux 6.13+).

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=start`, `%rsi=len_in`, `%rdx=behavior`.
- REQ-2: `start = untagged_addr(start);`.
- REQ-3: `start & ~PAGE_MASK ⟹ -EINVAL`.
- REQ-4: `len = PAGE_ALIGN(len_in);` if overflow ⟹ `-EINVAL`.
- REQ-5: `start + len < start` ⟹ `-EINVAL`.
- REQ-6: `start + len > TASK_SIZE` ⟹ `-EINVAL`.
- REQ-7: `behavior` must be a known `MADV_*` constant ⟹ else `-EINVAL`.
- REQ-8: Privilege gates: `MADV_HWPOISON` / `MADV_SOFT_OFFLINE` require `capable(CAP_SYS_ADMIN)` ⟹ else `-EPERM`.
- REQ-9: Lock policy: behavior-only and `WILLNEED` take `mmap_read_lock`; `DONTNEED`/`FREE`/`REMOVE`/`POPULATE_*`/`COLLAPSE`/`GUARD_*` take `mmap_write_lock` (or downgrade pattern).
- REQ-10: For each intersecting VMA, `madvise_vma_behavior(vma, prev, start, end, behavior)` is called; per-advice handler may split VMAs at endpoints to apply.
- REQ-11: `MADV_DONTNEED` on a `MAP_PRIVATE|MAP_ANONYMOUS` VMA zaps PTEs; next read returns zero page; next write COWs an anonymous page.
- REQ-12: `MADV_DONTNEED` on a file-backed VMA reverts to file-backed (next fault reads from page cache).
- REQ-13: `MADV_DONTNEED` is denied on `VM_LOCKED` VMA unless `behavior == MADV_DONTNEED_LOCKED` ⟹ `-EINVAL`.
- REQ-14: `MADV_FREE` only on `MAP_PRIVATE|MAP_ANONYMOUS`; on shared / file-backed ⟹ silently treated as no-op (or `-EINVAL`, kernel-version dependent).
- REQ-15: `MADV_REMOVE` requires `vma->vm_file && (vma->vm_flags & VM_SHARED)` and backing fs supports `FALLOC_FL_PUNCH_HOLE`; else `-EINVAL` / `-EOPNOTSUPP`.
- REQ-16: `MADV_HWPOISON` looks up each page via `vm_normal_page` and invokes `memory_failure(pfn, MF_COUNT_INCREASED)`; success returns `0`.
- REQ-17: `MADV_POPULATE_READ/WRITE` walks the range via `get_user_pages_remote` faulting each page; `SIGBUS`/`SIGSEGV`-equivalent conditions return `-EFAULT`/`-EHWPOISON`/`-EAGAIN`.
- REQ-18: `MADV_COLLAPSE` calls `madvise_collapse(vma, start, end)`; returns `-EAGAIN` if collapse cannot proceed *right now* (caller may retry).
- REQ-19: `MADV_GUARD_INSTALL` installs special PTE entries that trigger SIGSEGV on touch; no anonymous page is consumed; backing accounting unchanged.
- REQ-20: `MADV_GUARD_REMOVE` removes guard PTEs; ranges that have no guard PTE are skipped.
- REQ-21: Userfaultfd-registered VMAs: destructive advice may emit `UFFD_EVENT_REMOVE`.
- REQ-22: LSM hook `security_madvise` (where present) consulted; denial ⟹ `-EPERM`.
- REQ-23: On partial application failure (e.g., one VMA in the middle of the range rejects), return value indicates the error; ranges before the failing VMA may have been applied (destructive case).

## Acceptance Criteria

- [ ] AC-1: `madvise(p, len, MADV_NORMAL) == 0`.
- [ ] AC-2: `madvise(p+1, len, MADV_NORMAL) == -EINVAL` (unaligned).
- [ ] AC-3: `madvise(p, len, -1) == -EINVAL` (unknown advice).
- [ ] AC-4: `madvise(p, len, MADV_DONTNEED)` on anonymous VMA ⟹ subsequent read returns zero.
- [ ] AC-5: `madvise(p, len, MADV_DONTNEED)` on a `VM_LOCKED` VMA ⟹ `-EINVAL`.
- [ ] AC-6: `madvise(p, len, MADV_DONTNEED_LOCKED)` on a `VM_LOCKED` VMA ⟹ `0` and the lock count is preserved.
- [ ] AC-7: `madvise(p, len, MADV_FREE)` on a `MAP_SHARED` mapping ⟹ no effect (or `-EINVAL`).
- [ ] AC-8: `madvise(p, len, MADV_REMOVE)` on a tmpfs `MAP_SHARED` VMA ⟹ subsequent read returns zero; file is hole-punched.
- [ ] AC-9: `madvise(p, len, MADV_HWPOISON)` as unprivileged user ⟹ `-EPERM`.
- [ ] AC-10: `madvise(p, len, MADV_HUGEPAGE)` on a `MAP_PRIVATE|MAP_ANONYMOUS` VMA ⟹ `vma->vm_flags & VM_HUGEPAGE` set.
- [ ] AC-11: `madvise(p, len, MADV_DONTDUMP)` excludes range from `/proc/PID/coredump_filter` results.
- [ ] AC-12: `madvise(p, len, MADV_POPULATE_READ)` pre-faults entire range; subsequent first-access does not generate page fault.
- [ ] AC-13: `madvise(p, len, MADV_COLLAPSE)` succeeds on a fully-mapped THP-eligible region.
- [ ] AC-14: `madvise(p, len, MADV_GUARD_INSTALL)` followed by access to the range ⟹ `SIGSEGV`.
- [ ] AC-15: `madvise(p, len, MADV_GUARD_REMOVE)` after install restores access semantics.

## Architecture

```
struct MadviseArgs { start: u64, len: usize, behavior: i32 }
```

`sys_madvise(args) -> i32`:

1. Strip address tag.
2. Validate `start` page-aligned, length non-overflow.
3. Validate `behavior` is a known constant.
4. Privilege gate for `MADV_HWPOISON`/`MADV_SOFT_OFFLINE`.
5. Choose lock: `mmap_read_lock` or `mmap_write_lock` per advice.
6. `do_madvise(current.mm, start, len, behavior)` walks VMAs and applies behavior.
7. Release lock; return result.

`Mm::do_madvise(mm, start, len, behavior)`:

1. Compute `end = start + len`.
2. `madvise_walk_vmas(mm, start, end, behavior, madvise_vma_behavior)`:
   - For each VMA intersecting `[start, end)`:
     - Compute the sub-range `[vstart, vend)` intersected with the VMA.
     - Call `madvise_vma_behavior(prev, vma, vstart, vend, behavior)`.
     - On per-VMA error, abort (or record and continue, per advice).
3. Return aggregated result.

`madvise_vma_behavior(...)` dispatch:

- Behavior bits ⟹ split VMA at endpoints, update `vma->vm_flags`, possibly merge.
- `WILLNEED` ⟹ `force_page_cache_readahead(vma->vm_file, &vma->vm_file->f_ra, start, end)`.
- `DONTNEED` / `FREE` / `REMOVE` ⟹ `zap_page_range_single` / `madvise_free_single_vma` / `vfs_fallocate(PUNCH_HOLE)`.
- `COLD` / `PAGEOUT` ⟹ `madvise_cold_or_pageout_pte_range` (reclaim path).
- `POPULATE_*` ⟹ `madvise_populate(vma, start, end, write)`.
- `HWPOISON` ⟹ per-page `memory_failure`.
- `COLLAPSE` ⟹ `madvise_collapse(vma, start, end)`.
- `GUARD_INSTALL` / `GUARD_REMOVE` ⟹ install/remove guard PTE entries.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `behavior_known` | INVARIANT | `behavior ∈ {MADV_*}`. |
| `lock_appropriate` | INVARIANT | Destructive advice runs under `mmap_write_lock`; non-destructive under at least `mmap_read_lock`. |
| `vm_flags_set_only_under_lock` | INVARIANT | All `vma->vm_flags` mutations under write lock. |
| `privilege_check` | INVARIANT | HWPOISON / SOFT_OFFLINE only with `CAP_SYS_ADMIN`. |
| `range_alignment` | INVARIANT | Operations apply to whole pages only. |

### Layer 2: TLA+

`uapi/syscalls/madvise.tla`:
- Action set parameterized by advice value.
- Properties:
  - `safety_destructive_advice_under_write_lock`,
  - `safety_no_partial_state_on_per_vma_failure_for_behavior_advice`,
  - `liveness_madvise_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `madvise_vma_behavior` returns consistent error or success per VMA | `Mm::madvise_walk_vmas` |
| Guard PTE installation does not consume swap entries | `Mm::madvise_guard_install` |
| `MADV_COLLAPSE` does not partially collapse a single 2MiB region | `Mm::madvise_collapse` |

### Layer 4: Verus/Creusot functional

`madvise(addr, len, advice) ≡ POSIX.1-2024 posix_madvise` for the POSIX subset (`NORMAL`, `RANDOM`, `SEQUENTIAL`, `WILLNEED`, `DONTNEED`); Linux extensions per `man 2 madvise`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`madvise(2)` reinforcement:

- **Per privilege-gate on destructive advice** — `MADV_HWPOISON`, `MADV_SOFT_OFFLINE` require `CAP_SYS_ADMIN`; defense against unprivileged HWE injection.
- **Per `MADV_DONTNEED_LOCKED` separation** — only this advice can act on `VM_LOCKED` ranges; plain `MADV_DONTNEED` is denied to prevent accidental unlock side effects.
- **Per `MADV_REMOVE` shared-tmpfs-only** — defense against hole-punch attacks on private mappings or non-supporting filesystems.
- **Per `MADV_FREE` private-anon-only** — prevents bypassing file consistency via lazy-free of shared/file-backed pages.
- **Per `MADV_GUARD_INSTALL` integrity** — guard PTEs are kernel-managed special entries; user cannot fabricate them via raw PTE manipulation.
- **Per `MADV_COLLAPSE` per-call retryable** — `-EAGAIN` allows userspace to bound retries, defeating spin attacks on the THP allocator.
- **Per `userfaultfd_remove` event** — destructive advice on UFD-registered regions emits `UFFD_EVENT_REMOVE` so the UFD owner observes coherent state.
- **Per LSM hook** — every `madvise` call observed by SELinux/AppArmor/Smack/...
- **Per-tag-strip** — TBI / LAM tag bits stripped from `start` before validation.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — `MADV_HUGEPAGE` / `MADV_COLLAPSE` cannot grant `VM_EXEC`; THP pages carry parent VMA's protections unchanged. Defense against using THP collapse to bypass W^X (no permission elevation possible).
- **PAX_NOEXEC** — `MADV_NORMAL` etc. cannot re-enable `VM_EXEC`; PaX policy permissions tracked at VMA creation and not mutable via madvise.
- **PAX_RANDMMAP** — irrelevant for advice (no address selection).
- **PAX_RANDEXEC** — irrelevant.
- **GRKERNSEC_HWPOISON** — additional gating on `MADV_HWPOISON` to require dmesg-audit log entry; defense against silent HWE injection on production hosts.
- **PAX_REFCOUNT** — `vma->vm_users`-equivalent saturating refcount during long walks (no UAF).
- **PAX_UDEREF** — `start` is numeric; no kernel-side user dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — pages freed by `MADV_DONTNEED` / `MADV_FREE` / `MADV_REMOVE` sanitized via `__free_pages` (zero-on-free). Defense against post-free residue infoleak.
- **GRKERNSEC_BRUTE** — repeated madvise-driven crashes (e.g., GUARD_INSTALL on the stack triggering SIGSEGV in a loop) detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers in VMA dumps.
- **PAX_RANDKSTACK** — kstack offset rerolled at madvise syscall entry.
- **RLIMIT_MEMLOCK enforcement** — `MADV_DONTNEED_LOCKED` re-checks the lock budget so the call cannot side-channel-elevate locked-page count.
- **MFD_NOEXEC_SEAL interplay** — `MADV_HUGEPAGE` on a memfd-sealed exec region cannot acquire `VM_EXEC` via merging.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `process_madvise(2)` (separate Tier-5 if expanded).
- `posix_madvise(3)` POSIX wrapper (userspace).
- THP internals (`khugepaged`).
- KSM merge / unmerge internals.
- Userfaultfd internals.
- `fadvise64(2)` (different syscall; `posix_fadvise` analogue for file ranges, not VMAs).
- Implementation code.
