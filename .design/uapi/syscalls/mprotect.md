# Tier-5: syscall 10 — mprotect(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`10  common  mprotect  sys_mprotect`)
  - mm/mprotect.c (`SYSCALL_DEFINE3(mprotect, ...)`, `do_mprotect_pkey`, `mprotect_fixup`, `change_protection`)
  - include/linux/mm.h (`VM_READ`, `VM_WRITE`, `VM_EXEC`, `VM_MAYREAD`, `VM_MAYWRITE`, `VM_MAYEXEC`, `VM_ACCESS_FLAGS`)
  - include/uapi/asm-generic/mman-common.h (`PROT_*`, `PROT_GROWSDOWN`, `PROT_GROWSUP`)
  - arch/x86/include/asm/mmu_context.h / pgtable.h (`calc_vm_prot_bits`, `arch_validate_prot`, `arch_override_mprotect_pkey`)
-->

## Summary

`mprotect(2)` is **x86_64 syscall 10**, the page-protection mutator. It changes the access protection (`PROT_READ|PROT_WRITE|PROT_EXEC`, or `PROT_NONE`) of every page in the range `[start, start+len)`. The range must consist entirely of mapped VMAs; the call splits/merges VMAs as needed to apply the new protection bits, then flushes TLBs via `tlb_gather_mmu`/`tlb_finish_mmu`. Crucially, `mprotect` cannot grant a permission the underlying VMA never had — the kernel checks each new `VM_*` bit against the original `VM_MAY*` cap (e.g., a VMA mapped from an `O_RDONLY` file can never gain `VM_WRITE` via `mprotect`); violations ⟹ `-EACCES`. `mprotect` is also the W^X enforcement chokepoint: under `PAX_MPROTECT` or `map_deny_write_exec` (a mainline-merged subset of the PaX idea), flipping a writable mapping to executable (or vice versa simultaneously) is denied.

Critical for: every JIT (write code, then `mprotect(addr, len, PROT_READ|PROT_EXEC)`), every dynamic linker (`PROT_READ|PROT_WRITE` for relocations then `PROT_READ` for read-only data), every stack-execution disabler, every guard-page implementation.

## Signature

C (POSIX / man-pages):

```c
int mprotect(void *addr, size_t len, int prot);
```

glibc wrapper: `__libc_mprotect` → `INLINE_SYSCALL(mprotect, 3, addr, len, prot)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(mprotect, unsigned long, start, size_t, len, unsigned long, prot);
```

Rookery dispatch:

```rust
pub fn sys_mprotect(start: u64, len: usize, prot: u64) -> SyscallResult<i32>;
```

## Parameters

| name  | type             | constraints                                                       | errno-on-bad |
|-------|------------------|-------------------------------------------------------------------|--------------|
| start | `unsigned long`  | page-aligned (`start & ~PAGE_MASK == 0`)                          | `EINVAL`     |
| len   | `size_t`         | `>= 0`; rounded up via `PAGE_ALIGN(len)`; sum must not overflow   | `ENOMEM`     |
| prot  | `unsigned long`  | subset of `PROT_NONE|READ|WRITE|EXEC|SEM` + optionally `PROT_GROWSDOWN` xor `PROT_GROWSUP` | `EINVAL`     |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno. **Partial application possible**: on per-VMA failure, all VMAs up to that point retain new protection.

## Errors

| errno      | condition                                                                                  |
|------------|--------------------------------------------------------------------------------------------|
| `EINVAL`   | `start` not page-aligned; both `PROT_GROWSDOWN` and `PROT_GROWSUP` set; `PROT_GROWSDOWN` and VMA is not `VM_GROWSDOWN`; `PROT_GROWSUP` and VMA is not `VM_GROWSUP`; `arch_validate_prot` rejection (e.g., `PROT_BTI` on non-BTI hw, MTE flag conflict); `arch_validate_flags` rejection; `pkey` not allocated (`pkey_mprotect`). |
| `ENOMEM`   | `start + len` overflow (`end <= start`); some address in range is not mapped; no VMA found via `vma_find`; range crosses an unmapped gap (`tmp < end` at loop end). |
| `EACCES`   | Requested `PROT_*` bit was never permitted on the VMA (i.e., `prot & ~vma->vm_flags >> 4 & VM_ACCESS_FLAGS != 0`); `map_deny_write_exec` (W^X) denial; PAX_MPROTECT denial. |
| `EINTR`    | `mmap_write_lock_killable` interrupted by fatal signal.                                    |
| `EPERM`    | LSM (`security_file_mprotect`) denial (e.g., SELinux `execmem`/`execstack` constraint).    |

## ABI surface (constants + flags)

`prot` bits (composable):

- `PROT_NONE`    `0x0` — no access; SIGSEGV on touch.
- `PROT_READ`    `0x1`.
- `PROT_WRITE`   `0x2`.
- `PROT_EXEC`    `0x4`.
- `PROT_SEM`     `0x8` — atomic-operation-capable (arch-specific).
- `PROT_GROWSDOWN` `0x01000000` — extend change downward to VMA start (only valid if VMA is `VM_GROWSDOWN`, e.g., stack).
- `PROT_GROWSUP`   `0x02000000` — extend change upward to VMA end (only valid if VMA is `VM_GROWSUP`).

`VM_*` caps the kernel uses to decide whether `prot` is even allowed:

- `VM_MAYREAD`, `VM_MAYWRITE`, `VM_MAYEXEC`, `VM_MAYSHARE` — set at mmap time based on file `f_mode` and `MAP_SHARED`/`MAP_PRIVATE`.
- `VM_READ`, `VM_WRITE`, `VM_EXEC` — current effective; these are what `mprotect` modifies.
- `VM_ACCESS_FLAGS = VM_READ | VM_WRITE | VM_EXEC`.

Kernel macro: `newflags & ~(newflags >> 4) & VM_ACCESS_FLAGS` — non-zero ⟹ requested permission exceeds cap ⟹ `-EACCES`. (The `>> 4` shift turns `VM_MAY*` into `VM_*` positions.)

`READ_IMPLIES_EXEC` personality: if set and `prot & PROT_READ`, kernel ORs `PROT_EXEC` into the new bits for any VMA whose `vm_flags & VM_MAYEXEC` (i.e., non-noexec-mounted file).

Memory Protection Keys (PKEYs, `CONFIG_ARCH_HAS_PKEYS`) extend this via `pkey_mprotect(2)`: a 4-bit `pkey` is selected and stored in the page-table-entry's PKEY bits; userspace can dynamically deny via `WRPKRU` instruction. `mprotect(2)` itself uses `pkey == -1`, retaining the VMA's existing pkey.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=start`, `%rsi=len`, `%rdx=prot`.
- REQ-2: `start = untagged_addr(start)` strips arch-specific tagging (TBI / LAM).
- REQ-3: `prot & (PROT_GROWSDOWN|PROT_GROWSUP)` extracted; both set ⟹ `-EINVAL`.
- REQ-4: `start & ~PAGE_MASK ⟹ -EINVAL`.
- REQ-5: `len == 0 ⟹ 0` (no-op success).
- REQ-6: `len = PAGE_ALIGN(len)`; `end = start + len`; `end <= start ⟹ -ENOMEM` (overflow).
- REQ-7: `arch_validate_prot(prot, start)` (x86: reject `PROT_SEM`; arm64: validate `PROT_BTI`/`PROT_MTE` compatibility; powerpc: SAO) — false ⟹ `-EINVAL`.
- REQ-8: `mmap_write_lock_killable(current->mm)` — fatal signal during wait ⟹ `-EINTR`.
- REQ-9: For `pkey_mprotect(2)`, `mm_pkey_is_allocated(mm, pkey)` ⟹ else `-EINVAL`.
- REQ-10: `vma_find(start, end)` returns first VMA overlapping range; absence ⟹ `-ENOMEM`.
- REQ-11: `PROT_GROWSDOWN` requires VMA to be `VM_GROWSDOWN`; sets `start = vma.vm_start`; else `-EINVAL`.
- REQ-12: `PROT_GROWSUP` requires VMA to be `VM_GROWSUP`; sets `end = vma.vm_end`; else `-EINVAL`.
- REQ-13: For each VMA in `[start, end)`:
  - VMA's `vm_start` must equal current iteration's `tmp` (no gap) ⟹ else `-ENOMEM`.
  - If `READ_IMPLIES_EXEC && (vma.vm_flags & VM_MAYEXEC)` ⟹ `prot |= PROT_EXEC`.
  - Compute `new_vma_pkey = arch_override_mprotect_pkey(vma, prot, pkey)`.
  - `newflags = calc_vm_prot_bits(prot, new_vma_pkey) | (vma.vm_flags & ~VM_ACCESS_FLAGS)`.
  - Permission cap check: `(newflags & ~(newflags >> 4)) & VM_ACCESS_FLAGS != 0 ⟹ -EACCES`.
  - `map_deny_write_exec(vma, new_vma_flags)` (W^X) ⟹ `-EACCES`.
  - `arch_validate_flags(newflags) == false ⟹ -EINVAL`.
  - `security_file_mprotect(vma, reqprot, prot)` LSM hook (SELinux `execmem` etc.) ⟹ propagate error.
  - If `vma.vm_ops.mprotect` ⟹ call (e.g., for hugetlb).
  - `mprotect_fixup` — split or merge VMA at boundary, call `change_protection` to walk page tables and flip PTE bits.
- REQ-14: After loop: `if !error && tmp < end ⟹ -ENOMEM` (range had a gap).
- REQ-15: `tlb_finish_mmu(&tlb)` flushes deferred TLB shootdowns.
- REQ-16: Partial success is observable to userspace: if VMA N fails, VMAs 0..N-1 retain new protection.
- REQ-17: `mmap_write_unlock(current->mm)` released on every exit path.

## Acceptance Criteria

- [ ] AC-1: `mprotect(addr, 4096, PROT_READ)` on a previously RW page makes it read-only (write ⟹ SIGSEGV).
- [ ] AC-2: `mprotect(unaligned_addr, 4096, PROT_READ) == -EINVAL`.
- [ ] AC-3: `mprotect(addr, very_large, PROT_READ) == -ENOMEM` (overflow).
- [ ] AC-4: `mprotect(unmapped_addr, 4096, PROT_READ) == -ENOMEM`.
- [ ] AC-5: `mprotect(O_RDONLY_mmap, 4096, PROT_READ|PROT_WRITE) == -EACCES` (VM_MAYWRITE not set).
- [ ] AC-6: `mprotect(addr, 4096, PROT_GROWSDOWN|PROT_GROWSUP) == -EINVAL`.
- [ ] AC-7: `mprotect(stack_addr, 4096, PROT_READ|PROT_GROWSDOWN)` extends change to VMA start.
- [ ] AC-8: Under PAX_MPROTECT, `mprotect(addr, 4096, PROT_READ|PROT_WRITE|PROT_EXEC) == -EACCES` (W^X violation).
- [ ] AC-9: `mprotect(addr, 4096, PROT_NONE)` makes the page inaccessible; subsequent read ⟹ SIGSEGV.
- [ ] AC-10: `mprotect` across two adjacent VMAs with no gap succeeds; across VMAs with a gap ⟹ `-ENOMEM` (partial applied).
- [ ] AC-11: `READ_IMPLIES_EXEC` personality + `mprotect(addr, 4096, PROT_READ)` ⟹ resulting VMA has `VM_EXEC` set.
- [ ] AC-12: LSM (SELinux `execmem`) denial ⟹ `-EPERM`/`-EACCES`.

## Architecture

```
struct MprotectArgs { start: u64, len: usize, prot: u64 }
```

`sys_mprotect(args) -> i32`:

1. Return `do_mprotect_pkey(args.start, args.len, args.prot, -1)`.

`Mm::do_mprotect_pkey(start, len, prot, pkey) -> i32`:

1. `start = untagged_addr(start);`
2. `let grows = prot & (PROT_GROWSDOWN | PROT_GROWSUP);`
3. `let rier = (current.personality & READ_IMPLIES_EXEC) && (prot & PROT_READ);`
4. `prot &= ~(PROT_GROWSDOWN | PROT_GROWSUP);`
5. If `grows == (PROT_GROWSDOWN | PROT_GROWSUP)` ⟹ return `-EINVAL`.
6. If `start & ~PAGE_MASK` ⟹ `-EINVAL`.
7. If `len == 0` ⟹ `0`.
8. `len = PAGE_ALIGN(len); end = start + len;`
9. If `end <= start` ⟹ `-ENOMEM`.
10. If `!arch_validate_prot(prot, start)` ⟹ `-EINVAL`.
11. `let reqprot = prot;`
12. `mmap_write_lock_killable(current.mm)?;` (else `-EINTR`).
13. If `pkey != -1 && !mm_pkey_is_allocated(current.mm, pkey)` ⟹ `error = -EINVAL; goto out;`.
14. `vma_iter_init(&vmi, mm, start); let vma = vma_find(&vmi, end);`
15. If `!vma` ⟹ `error = -ENOMEM; goto out;`.
16. Handle `PROT_GROWSDOWN`/`PROT_GROWSUP` boundary adjustments (REQ-11/12).
17. `let prev = vma_prev(&vmi); if start > vma.vm_start { prev = vma; }`
18. `tlb_gather_mmu(&tlb, mm); nstart = start; tmp = vma.vm_start;`
19. For each VMA in range:
    - If `vma.vm_start != tmp` ⟹ `-ENOMEM; break;`
    - If `rier && (vma.vm_flags & VM_MAYEXEC)` ⟹ `prot |= PROT_EXEC;`
    - `new_vma_pkey = arch_override_mprotect_pkey(vma, prot, pkey);`
    - `newflags = calc_vm_prot_bits(prot, new_vma_pkey) | (vma.vm_flags & ~VM_ACCESS_FLAGS);`
    - Cap check: `(newflags & ~(newflags >> 4)) & VM_ACCESS_FLAGS != 0 ⟹ -EACCES; break;`
    - W^X check via `map_deny_write_exec(vma, new_vma_flags) ⟹ -EACCES; break;`
    - `arch_validate_flags(newflags) == false ⟹ -EINVAL; break;`
    - `security_file_mprotect(vma, reqprot, prot)?` ⟹ `break;`
    - `tmp = min(vma.vm_end, end);`
    - `vma.vm_ops.mprotect(vma, nstart, tmp, newflags)?` if present.
    - `mprotect_fixup(&vmi, &tlb, vma, &prev, nstart, tmp, newflags)?` ⟹ `break;`
    - `tmp = vma_iter_end(&vmi); nstart = tmp; prot = reqprot;`
20. `tlb_finish_mmu(&tlb);`
21. If `!error && tmp < end` ⟹ `error = -ENOMEM`.
22. `mmap_write_unlock(mm); return error;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `start_page_aligned` | INVARIANT | `start & ~PAGE_MASK == 0`. |
| `end_no_overflow` | INVARIANT | `end > start` after `PAGE_ALIGN(len)`. |
| `growsdown_growsup_exclusive` | INVARIANT | At most one of `PROT_GROWSDOWN|PROT_GROWSUP`. |
| `permission_capped_by_VM_MAY` | INVARIANT | For all bits set in new `VM_R/W/X`, corresponding `VM_MAY*` set on VMA. |
| `mmap_lock_held` | INVARIANT | `mmap_write_lock` held across all per-VMA mutations. |
| `tlb_balanced` | INVARIANT | Every `tlb_gather_mmu` paired with `tlb_finish_mmu`. |
| `wx_denied_under_PAX_MPROTECT` | INVARIANT | `PROT_WRITE && PROT_EXEC ⟹ -EACCES` when policy enabled. |

### Layer 2: TLA+

`uapi/syscalls/mprotect.tla`:
- Per-call → arg validation → mmap_write_lock → per-VMA cap-check + fixup → tlb_finish → unlock.
- Properties:
  - `safety_no_grant_beyond_cap`,
  - `safety_wx_separation_under_policy`,
  - `safety_growsdown_only_on_growsdown_vma`,
  - `liveness_mprotect_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: all PTEs in `[start, end)` reflect new `prot` (or partial-success boundary) | `Mm::change_protection` |
| Post: VMA boundaries match split points (`start`, `end`) | `Mm::mprotect_fixup` |
| `EACCES` ⟹ no PTE on this VMA changed | `Mm::do_mprotect_pkey` |

### Layer 4: Verus/Creusot functional

`mprotect(addr, len, prot) ≡ POSIX.1-2024 mprotect` semantic equivalence, with Linux-specific `PROT_GROWSDOWN`/`PROT_GROWSUP`/`READ_IMPLIES_EXEC` extensions.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mprotect(2)` reinforcement:

- **Per-`VM_MAY*` cap** — defense against retroactively granting write on read-only file mapping (CVE-1999-vintage pattern).
- **Per-`map_deny_write_exec`** — defense against writable-then-executable JIT abuse, baseline W^X.
- **Per-`security_file_mprotect`** — defense against SELinux `execmem`/`execstack`/`execheap` policy bypass.
- **Per-`arch_validate_prot` / `arch_validate_flags`** — defense against arch-incompatible flag (PROT_BTI on non-BTI hw, MTE conflicts).
- **Per-`PROT_GROWSDOWN`/`PROT_GROWSUP` VMA-type gate** — defense against using grows-flags on non-stack VMAs to escalate.
- **Per-partial-application visibility** — defense against silent half-mprotect (caller MUST validate post-state).
- **Per-`untagged_addr`** — defense against TBI / LAM tag-evasion of address comparison.
- **Per-pkey allocation gate** — defense against using unallocated PKEY.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — denies any `mprotect` that would create a writable+executable mapping or flip an existing writable mapping to executable; the only path to executable code is at-load-time RX or `pkey_mprotect` with privileged-policy. Returns `-EACCES`.
- **PAX_NOEXEC** — denies `PROT_EXEC` on heap/stack/anonymous regions; complements PAX_MPROTECT for the initial mmap.
- **PAX_RANDEXEC** — randomizes load addresses of executable VMAs; `mprotect`-driven base disclosure mitigated.
- **PAX_REFCOUNT** — `mm.mm_users` saturating; underflow ⟹ `BUG()`.
- **GRKERNSEC_RWXMAP_LOG** — every `mprotect` attempt that would result in (or denies) WX is logged.
- **PAX_UDEREF** — `start` not dereferenced kernel-side; SMAP guards.
- **PAX_MEMORY_SANITIZE** — pages freed during VMA-split sanitized.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers (VMA addresses, PTE values).
- **GRKERNSEC_BRUTE** — repeated `EACCES` from `mprotect` attempts (heap-spray / ROP probe) counted toward brute-force throttling.
- **PAX_RANDKSTACK** — kstack offset randomized at mprotect syscall entry.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `pkey_mprotect(2)` (separate Tier-5 if expanded; same core but with explicit `pkey` arg).
- `pkey_alloc(2)` / `pkey_free(2)` (separate Tier-5).
- VMA-merging / VMA-splitting algorithm internals (covered in `mm/vma.md` Tier-3).
- Page-table walking and TLB flushing internals (covered in `mm/pgtable.md` Tier-3).
- Implementation code.
