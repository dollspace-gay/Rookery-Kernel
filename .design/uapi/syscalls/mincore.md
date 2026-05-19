# Tier-5 syscall: mincore(2) — syscall 27

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mincore.c (SYSCALL_DEFINE3(mincore, ...), mincore_pte_range, mincore_unmapped_range)
  - mm/swap_state.c (find_get_incore_page)
  - include/uapi/asm-generic/mman-common.h (MINCORE_INCORE)
  - arch/x86/entry/syscalls/syscall_64.tbl (27  common  mincore)
-->

## Summary

`mincore(2)` reports whether each page in a virtual-address range is **resident in physical memory** at the time of the call. The kernel walks the page tables (and, for file-backed memory, the page cache + swap cache) and writes a one-byte-per-page status vector. Bit 0 of each byte indicates "core resident" (`MINCORE_INCORE`); other bits are reserved.

Historically used by databases (Postgres `pg_prewarm`, MySQL buffer-pool inspection), redis persistence tuning, and language runtimes for GC heuristics. Since CVE-2019-5489 ("page-cache side channel"), `mincore` was hardened: it now reports residency only for **anonymous** pages owned by the caller's mm, and reports zero for file-backed pages unless the caller has write permission to the file. This blocks cross-process side-channel reconnaissance of file-cache state.

Critical for: madvise-prefetch decisions, pre-warmed buffer-pool startup, GC residency hints; security-sensitive after CVE-2019-5489.

## Signature

```c
#include <sys/mman.h>

int mincore(void *addr, size_t length, unsigned char *vec);

#define MINCORE_INCORE  0x1   /* page is resident */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `void *` | in | Start of range. Must be page-aligned. |
| `length` | `size_t` | in | Length in bytes. Rounded up to page granularity. |
| `vec` | `unsigned char *` | out | Caller-allocated array of ceil(length / PAGE_SIZE) bytes. Each byte's bit 0 set ⟺ page resident; other bits reserved (zero on read). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `vec` populated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `vec` user-pointer faults during copy_to_user. |
| `EINVAL` | `addr` not page-aligned, or `length` arithmetic overflow. |
| `ENOMEM` | Range contains an unmapped or non-VMA hole. |
| `EAGAIN` | (Linux ≥ 4.19; rare) mmap_lock contention failure on a retry-loop. |

## ABI surface

```text
__NR_mincore  (x86_64)   = 27
__NR_mincore  (arm64)    = 232
__NR_mincore  (riscv64)  = 232
__NR_mincore  (i386)     = 218

MINCORE_INCORE  = 0x1     /* only defined bit since CVE-2019-5489 hardening */

/* Pre-CVE-2019-5489 (historic): MINCORE_REFERENCED, MINCORE_MODIFIED bits.
   These are no longer reported, but bit 0 (INCORE) is preserved. */
```

## Compatibility contract

REQ-1: Syscall number is **27** on x86_64; 232 on arm64/riscv64. ABI-stable since 2.4.

REQ-2: `addr` MUST be page-aligned. `addr % PAGE_SIZE != 0` ⟹ EINVAL.

REQ-3: `length` rounded up to page boundary: `pages = (length + PAGE_SIZE - 1) / PAGE_SIZE`. `pages == 0` ⟹ vec untouched, return 0.

REQ-4: `addr + length` MUST not overflow user-virtual range; addr + length > TASK_SIZE ⟹ EINVAL.

REQ-5: Entire range MUST be mapped (i.e. covered by VMAs in the calling mm). A hole ⟹ ENOMEM. Partial fill is NOT supported (Linux differs here from some BSDs).

REQ-6: For **anonymous** pages owned by the caller's mm: bit 0 set ⟺ pte_present and not pte_special.

REQ-7: For **file-backed** pages (post CVE-2019-5489 hardening): bit 0 set requires that the caller has write permission to the underlying file (i.e. could in principle dirty the page). Otherwise bit 0 reports 0 regardless of cache state. This is the side-channel gate.

REQ-8: For **shared anonymous** (MAP_SHARED | MAP_ANONYMOUS): treated like the file-backed path (uses a tmpfs inode); same write-permission gate.

REQ-9: For **swap-cache** pages (swapped-out but in the swap cache): post-CVE, treated as not-present unless write permission test passes.

REQ-10: For **huge pages** (THP/hugetlb): each PAGE_SIZE chunk's status reported as the corresponding 4K view; if the huge page is present, all sub-pages report 1.

REQ-11: For **special VMAs** (VM_PFNMAP, VM_IO, VM_MIXEDMAP): page table is walked but unmapped slots report 0; mapped slots respect the same permission gate.

REQ-12: Kernel writes `vec` via `copy_to_user`; partial faulting copy ⟹ EFAULT and (Linux) NO state mutation rollback (the vec contents are best-effort but undefined on EFAULT).

REQ-13: `mincore` does NOT itself fault in pages, does NOT change residency, does NOT update referenced bits. It is read-only on mm state.

REQ-14: Concurrent reclaim may evict pages mid-walk; reported result is best-effort and inherently racy. No locks held across return; results stale by the time userspace reads them.

REQ-15: LSM hook `security_file_permission(file, MAY_READ)` (or `MAY_WRITE` for the gate) consulted for each distinct file-backed VMA touched.

## Acceptance Criteria

- [ ] AC-1: mincore(0x10000, 4096, vec) on a freshly-mmaped anonymous page (unfaulted): vec[0] & 1 == 0.
- [ ] AC-2: After touching the page (faulting it in): vec[0] & 1 == 1.
- [ ] AC-3: mincore over an unmapped hole: -ENOMEM.
- [ ] AC-4: mincore(0x10001, ...) unaligned: -EINVAL.
- [ ] AC-5: vec pointing to read-only memory: -EFAULT on the copy_to_user.
- [ ] AC-6: mincore on a file-backed RO mapping where caller cannot write the file: vec bytes all zero regardless of cache state.
- [ ] AC-7: mincore on a file-backed RW mapping owned by caller: cached pages report 1.
- [ ] AC-8: mincore on swapped-out anonymous owned: vec[i] & 1 == 0 (swap is "not core").
- [ ] AC-9: mincore on a THP-backed range: all sub-pages report 1.
- [ ] AC-10: length = 0: vec untouched, return 0.
- [ ] AC-11: Race with munmap during call: ENOMEM OR partial result is consistent with the post-munmap state.
- [ ] AC-12: Bits 1..7 of each vec byte are always 0.

## Architecture

```rust
#[syscall(nr = 27, abi = "sysv")]
pub fn sys_mincore(addr: u64, length: u64, vec: UserPtr<u8>) -> isize {
    Mincore::do_mincore(addr, length, vec)
}
```

`Mincore::do_mincore(addr, length, vec_ptr) -> isize`:
1. if (addr & (PAGE_SIZE - 1)) != 0       { return Err(EINVAL); }
2. if length.checked_add(PAGE_SIZE - 1).is_none() { return Err(EINVAL); }
3. let pages = (length + PAGE_SIZE - 1) >> PAGE_SHIFT;
4. if pages == 0                          { return 0; }
5. if addr.checked_add(pages << PAGE_SHIFT).map(|e| e > TASK_SIZE).unwrap_or(true) {
6.   return Err(EINVAL);
7. }
8. /* Allocate small kernel buffer; chunk loop for large ranges */
9. let mut kbuf = SmallVec::<u8, CHUNK>::new();   // CHUNK = 256
10. let mut remaining = pages;
11. let mut cur = addr;
12. let mut out = vec_ptr;
13. mm::mmap_read_lock(current_mm());
14. while remaining > 0 {
15.   let n = min(remaining, CHUNK as u64);
16.   kbuf.resize(n as usize, 0);
17.   Mincore::walk_range(current_mm(), cur, n, &mut kbuf)?;     // ENOMEM if hole
18.   /* Drop lock for copy_to_user to avoid mm-lock contention with faults */
19.   mm::mmap_read_unlock(current_mm());
20.   out.copy_out_slice(&kbuf[..n as usize])?;                  // EFAULT
21.   mm::mmap_read_lock(current_mm());
22.   cur += n << PAGE_SHIFT;
23.   out  = out.add(n as usize);
24.   remaining -= n;
25. }
26. mm::mmap_read_unlock(current_mm());
27. 0

`Mincore::walk_range(mm, addr, n_pages, out) -> Result<()>`:
1. let mut walker = Walk::new(mm, addr, addr + (n_pages << PAGE_SHIFT));
2. walker.pte_entry = mincore_pte_entry;
3. walker.pmd_entry = mincore_pmd_entry;          // detect THP
4. walker.test_walk = mincore_unmapped_range;     // ENOMEM on hole
5. walker.private = out;
6. walker.walk()?
7. Ok(())

`mincore_pte_entry(pte, addr, end, walker)`:
1. let out: &mut [u8] = walker.private;
2. let i = ((addr - walker.start) >> PAGE_SHIFT) as usize;
3. if pte::is_none(pte) { out[i] = 0; return; }
4. if pte::is_swap(pte) {
5.   /* Post-CVE-2019-5489: do not report swap-cache hits for unauthorized peeker */
6.   out[i] = 0;
7.   return;
8. }
9. /* Present pte */
10. let folio = pte::folio(pte);
11. if folio::is_file_backed(folio) {
12.   /* CVE-2019-5489 gate */
13.   let file = folio::file(folio);
14.   if !file::caller_can_write(file) {
15.     out[i] = 0;
16.     return;
17.   }
18. }
19. out[i] = MINCORE_INCORE;

`mincore_unmapped_range(start, end, walker) -> Result<bool>`:
1. /* Return ENOMEM if there's no VMA covering [start, end) */
2. let vma = vma::find(walker.mm, start);
3. if vma.is_none() || vma.unwrap().vm_start > start { return Err(ENOMEM); }
4. /* Unmapped within a VMA: report zero */
5. let out: &mut [u8] = walker.private;
6. for p in (start..end).step_by(PAGE_SIZE) {
7.   let i = ((p - walker.start) >> PAGE_SHIFT) as usize;
8.   out[i] = 0;
9. }
10. Ok(true)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `addr_alignment_check` | INVARIANT | addr % PAGE_SIZE != 0 ⟹ EINVAL pre-walk. |
| `length_overflow_check` | INVARIANT | addr + roundup(length) ≤ TASK_SIZE. |
| `hole_returns_enomem` | INVARIANT | unmapped range in [addr, addr+length) ⟹ ENOMEM. |
| `file_backed_write_gate` | INVARIANT | file-backed page reports 1 ⟹ caller can write the file. |
| `swap_reports_zero` | INVARIANT | swap-cache hit not exposed (post-CVE). |
| `reserved_bits_zero` | INVARIANT | vec byte bits 1..7 == 0. |

### Layer 2: TLA+

`mm/mincore.tla`:
- States: per-arg-validate, per-walk, per-pte-classify, per-copy-out.
- Properties:
  - `safety_alignment_first` — EINVAL precedes walk.
  - `safety_hole_enomem` — any hole ⟹ ENOMEM.
  - `safety_file_gate` — file-backed cache hit ⟹ write permission.
  - `safety_no_side_channel` — without write permission, mincore output is independent of cache state.
  - `liveness_terminates` — mincore returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mincore` post: vec[i] == 1 ⟹ page i present ∧ (anonymous-owned ∨ write-permitted file-backed) | `Mincore::do_mincore` |
| `walk_range` post: every byte written; never skips. | `Mincore::walk_range` |
| `mincore_pte_entry` post: file-backed RO-to-caller ⟹ out = 0 | `mincore_pte_entry` |

### Layer 4: Verus / Creusot functional

Per-`mincore(2)` man-page; LTP `mincore01..mincore04`; CVE-2019-5489 selftest reproducer reports zero.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mincore(2)` reinforcement:

- **Per-CVE-2019-5489 file-backed write gate** — defense against per-page-cache side-channel reconnaissance.
- **Per-swap-cache zero report** — defense against per-swap-cache side channel.
- **Per-alignment-first ordering** — defense against per-DoS via near-overflow lengths.
- **Per-hole ENOMEM** — defense against per-silent partial fill.
- **Per-vec copy_to_user SMAP-guarded** — defense against per-vec-pointer kernel-deref bug.
- **Per-mmap_lock dropped for copy** — defense against per-mm-lock contention DoS during long-range walks.

## Grsecurity / PaX surface

- **CVE-2019-5489 side-channel gate enforced by default** — grsec treats the file-backed write-permission check as mandatory; even root cannot read mincore residency of files it cannot write (unlike upstream where root + CAP_SYS_ADMIN may bypass via debug paths). The defense is unconditional under GRKERNSEC_HIDESYM.
- **PaX UDEREF on vec copy_to_user** — SMAP-enforced; per-byte SMAP-guarded.
- **PAX_USERCOPY_HARDEN on vec slice** — vec must lie within a whitelisted slab/anonymous mapping.
- **GRKERNSEC_PROC_USERGROUP** — mincore reflects only the caller's own mm; cross-process mincore is impossible because the syscall always uses current_mm().
- **PAX_MEMORY_SANITIZE** — kernel-side scratch buffer cleared before each copy_to_user (no leftover page-table fragments).
- **GRKERNSEC_AUDIT_MINCORE** — auditable for security-sensitive groups; large mincore sweeps over many file-backed VMAs flagged.
- **No swap-cache leak** — grsec verifies that even the legitimate file-backed gate cannot be tricked into reporting swap-cache state.
- **GRKERNSEC_CHROOT neutral** — mincore is mm-local; chroot does not affect.
- **PAX_RANDMMAP increases entropy** — mincore over a randomized mapping is bounded by the address-layout randomization.
- **No CAP requirement** — mincore is unprivileged; grsec preserves this but enforces the gate strictly.
- **No_new_privs neutral** — mincore is not a privilege boundary.
- **THP/hugetlb correctness** — grsec audits per-sub-page reporting to ensure no information leak about huge-page consolidation patterns.
- **PaX KERNEXEC neutral** — read-only walk; no exec.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `madvise(2)` (separate Tier-5 doc).
- `mlock(2)` / `mlock2(2)` / `mlockall(2)` (separate Tier-5 docs).
- Page-cache architecture (Tier-3 `mm/filemap.md`).
- THP architecture (Tier-3 `mm/huge_memory.md`).
- Implementation code.
