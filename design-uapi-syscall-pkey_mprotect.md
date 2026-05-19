---
title: "Tier-5: syscall 329 — pkey_mprotect(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pkey_mprotect(2)` is **x86_64 syscall 329** (added in Linux 4.9), the protection-key-aware variant of `mprotect(2)`. It changes both the standard `PROT_*` page-protection bits **and** the Memory Protection Key (MPK / PKU on Intel; corresponding pkey hardware on Power and ARM) for every page in `[addr, addr+len)`. The PKEY is a 4-bit identifier stored in the PTE; when the CPU is configured (via `WRPKRU` on x86 or analogous on other ISAs), userspace can revoke access or write-access to a key without a syscall, providing nanosecond-scale enforcement of access barriers. The kernel allocates pkeys via `pkey_alloc(2)` (syscall 330) and frees them via `pkey_free(2)` (syscall 331); `pkey_mprotect` is the binding mechanism that associates a previously-allocated pkey with a memory range.

Critical for: in-process isolation of secrets (Erim, Hodor, libmpk), JIT compiler write-then-execute fences (RW pkey + RX pkey), language-runtime sandboxing (e.g., V8 isolates), userspace garbage-collector barriers, container-side fine-grained address-space isolation.

### Acceptance Criteria

- [ ] AC-1: `pkey_mprotect(p, 4096, PROT_READ, 0) == 0`; PTE PKEY bits reflect key 0.
- [ ] AC-2: `pkey_alloc(0, 0) == k`; `pkey_mprotect(p, 4096, PROT_READ|PROT_WRITE, k) == 0`; PTE PKEY bits reflect key `k`.
- [ ] AC-3: After AC-2, `WRPKRU` setting `PKEY_DISABLE_ACCESS` for key `k` ⟹ subsequent access to `p` raises SIGSEGV with `si_code == SEGV_PKUERR`.
- [ ] AC-4: `pkey_mprotect(p, 4096, PROT_READ, -1)` preserves existing pkey on the VMA.
- [ ] AC-5: `pkey_mprotect(p, 4096, PROT_READ, 99) == -EINVAL` (out of range on x86_64 with `arch_max_pkey == 16`).
- [ ] AC-6: `pkey_mprotect(p, 4096, PROT_READ, never_allocated) == -EINVAL`.
- [ ] AC-7: `pkey_mprotect(unmapped, 4096, PROT_READ, 0) == -ENOMEM`.
- [ ] AC-8: `pkey_mprotect(p+1, 4096, PROT_READ, 0) == -EINVAL` (unaligned).
- [ ] AC-9: `pkey_mprotect(p, 0, PROT_READ, 0) == 0` (no-op).
- [ ] AC-10: `pkey_mprotect(p, 4096, PROT_WRITE, k)` on a VMA from O_RDONLY file ⟹ `-EACCES` (cap-violation).
- [ ] AC-11: `pkey_mprotect(p, 4096, PROT_WRITE|PROT_EXEC, k)` ⟹ denied by `map_deny_write_exec` (`-EACCES`).
- [ ] AC-12: `pkey_mprotect(p, large, PROT_READ, k)` interrupted by SIGKILL ⟹ `-EINTR`.

### Architecture

```
struct PkeyMprotectArgs {
  start: u64, len: usize, prot: u64, pkey: i32,
}
```

`sys_pkey_mprotect(args) -> i32`:

1. Validate `pkey` in `[-1, arch_max_pkey())`. ⟹ else `-EINVAL`.
2. `do_mprotect_pkey(start, len, prot, pkey)`:
   a. Strip address tag.
   b. Extract / validate `PROT_GROWSDOWN`/`PROT_GROWSUP`.
   c. Page-align validation.
   d. `arch_validate_prot`.
   e. `mmap_write_lock_killable(mm)?;`
   f. If `pkey != -1` and `!mm_pkey_is_allocated(mm, pkey)` ⟹ unlock; `-EINVAL`.
   g. Locate first VMA.
   h. Loop over VMAs in range:
      - Compute `newflags` from `prot` + existing `vma->vm_flags`.
      - Enforce `VM_MAY*` cap ⟹ else `-EACCES`.
      - `new_pkey = arch_override_mprotect_pkey(vma, prot, pkey);`
      - `mprotect_fixup(&vmi, vma, &prev, nstart, tmp, newflags, new_pkey);`
      - Advance.
   i. `mmap_write_unlock(mm);`
3. Return `0` or error.

`Mm::mprotect_fixup(vmi, vma, prev_out, start, end, newflags, new_pkey)`:

1. Split VMA front/back if needed (`vma_split_at`).
2. `vma->vm_flags = newflags;`
3. `vma->vm_pkey = new_pkey;`
4. `vma->vm_page_prot = vm_get_page_prot(vma, vma->vm_flags);`
5. `change_protection(vma, start, end, MM_CP_UFFD_WP_RESOLVE);` — walks PTEs, updates protection and pkey bits, gathers TLB.
6. Try merge with neighbors via `vma_merge`.
7. Return `0`.

### Out of Scope

- `mprotect(2)` (separate Tier-5 — `mprotect.md`).
- `pkey_alloc(2)` syscall 330 (separate Tier-5 if expanded).
- `pkey_free(2)` syscall 331 (separate Tier-5 if expanded).
- `WRPKRU` / `RDPKRU` userspace instructions (CPU-side).
- Intel CET shadow-stack pkey interaction.
- ARM MTE / Power AMR analogues.
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int pkey_mprotect(void *addr, size_t len, int prot, int pkey);
```

glibc wrapper: `__pkey_mprotect` → `INLINE_SYSCALL(pkey_mprotect, 4, addr, len, prot, pkey)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(pkey_mprotect, unsigned long, start, size_t, len,
                unsigned long, prot, int, pkey);
```

Rookery dispatch:

```rust
pub fn sys_pkey_mprotect(start: u64, len: usize, prot: u64, pkey: i32) -> SyscallResult<i32>;
```

### parameters

| name  | type             | constraints                                                       | errno-on-bad |
|-------|------------------|-------------------------------------------------------------------|--------------|
| start | `unsigned long`  | page-aligned (`start & ~PAGE_MASK == 0`)                          | `EINVAL`     |
| len   | `size_t`         | `>= 0`; rounded up via `PAGE_ALIGN(len)`; sum must not overflow   | `ENOMEM`     |
| prot  | `unsigned long`  | subset of `PROT_NONE|READ|WRITE|EXEC|SEM` + optionally `PROT_GROWSDOWN` xor `PROT_GROWSUP` | `EINVAL`     |
| pkey  | `int`            | `-1` (retain VMA's existing pkey) or `0..arch_max_pkey()` (typically 0..15 on x86_64) and previously allocated | `EINVAL`     |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno. **Partial application possible**: on per-VMA failure, all VMAs up to that point retain new protection + pkey.

### errors

| errno    | condition                                                                                  |
|----------|--------------------------------------------------------------------------------------------|
| `EINVAL` | `start` not page-aligned; both `PROT_GROWSDOWN` and `PROT_GROWSUP` set; `arch_validate_prot` rejection; `pkey < -1` or `pkey >= arch_max_pkey()` (16 on x86_64); `pkey >= 0` and not allocated in `mm->context.pkey_allocation_map`. |
| `ENOMEM` | `start + len` overflow; some address in range is not mapped; no VMA found.                  |
| `EACCES` | Requested `PROT_*` bit was never permitted on the VMA (`prot & ~VM_MAY*` after cap projection); LSM denial of executable-stack; `map_deny_write_exec` W^X denial. |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                    |
| `EPERM`  | LSM (`security_file_mprotect`) denial.                                                     |

### abi surface (constants + flags)

`prot` bits (composable, identical to `mprotect(2)`):

- `PROT_NONE`    `0x0`.
- `PROT_READ`    `0x1`.
- `PROT_WRITE`   `0x2`.
- `PROT_EXEC`    `0x4`.
- `PROT_SEM`     `0x8` (arch-specific).
- `PROT_GROWSDOWN` `0x01000000`.
- `PROT_GROWSUP`   `0x02000000`.

PKEY ABI:

- **`pkey == -1`** — special sentinel: keep the VMA's existing pkey unchanged. (This is what `mprotect(2)` effectively passes.)
- **`pkey ∈ [0, arch_max_pkey)`** — a previously-allocated pkey index. On x86_64 with Intel MPK / VMX, `arch_max_pkey() == 16`. The pkey must have been allocated via `pkey_alloc(2)` and not yet freed.
- The pkey is stored in PTE bits 59:62 on x86_64 (4 bits). The hardware PKRU register (32 bits, 2 bits per key) controls access:
  - `PKEY_DISABLE_ACCESS` (`0x1` per key in PKRU) — disables both reads and writes for that key.
  - `PKEY_DISABLE_WRITE`  (`0x2` per key in PKRU) — disables writes for that key (reads still allowed).
- `mm->context.pkey_allocation_map` — bitmap of allocated pkeys, initialized to `{0, 1}` (key 0 is the implicit-default key; key 1 is the execute-only key when used).

`VM_*` flags affected by `pkey_mprotect`:

- `VM_READ`, `VM_WRITE`, `VM_EXEC` — updated per `prot` (subject to `VM_MAY*` caps).
- `vma->vm_pkey` — set to `pkey` if `pkey >= 0`; else retained.
- `vm_get_page_prot(vma)` reconstructs the PTE `_PAGE_PKEY_*` bits from `vma->vm_pkey` on each refault.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=start`, `%rsi=len`, `%rdx=prot`, `%r10=pkey`.
- REQ-2: `start = untagged_addr(start);` strip TBI/LAM tags.
- REQ-3: `pkey < -1 || pkey >= arch_max_pkey()` ⟹ `-EINVAL`. (On x86_64 without MPK, `arch_max_pkey() == 1`, so any `pkey > 0` ⟹ `-EINVAL`.)
- REQ-4: If `pkey != -1`, `mm_pkey_is_allocated(mm, pkey)` must be true; else `-EINVAL`.
- REQ-5: `prot & (PROT_GROWSDOWN|PROT_GROWSUP)` extracted; both set ⟹ `-EINVAL`.
- REQ-6: `start & ~PAGE_MASK` ⟹ `-EINVAL`.
- REQ-7: `len == 0` ⟹ `0` (no-op success).
- REQ-8: `len = PAGE_ALIGN(len);` `end = start + len`; `end <= start` ⟹ `-ENOMEM`.
- REQ-9: `arch_validate_prot(prot, start)` — false ⟹ `-EINVAL`.
- REQ-10: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-11: Re-check pkey allocation under the lock (`mm_pkey_is_allocated`); else `-EINVAL` and release.
- REQ-12: `vma = vma_find(&vmi, start)` — if none, or `vma->vm_start > start`, ⟹ `-ENOMEM`.
- REQ-13: For each VMA in `[start, end)`:
   - `tmp = vma->vm_end` (clip to `end`).
   - `arch_override_mprotect_pkey(vma, prot, pkey)` — arch can adjust pkey (e.g., execute-only pkey for `PROT_EXEC`-only).
   - `nstart = max(vma->vm_start, start); next = vma->vm_end;`
   - `mprotect_fixup(&vmi, vma, &prev, nstart, min(end, next), newflags, new_pkey)` — splits VMAs, updates `vma->vm_flags` and `vma->vm_pkey`, recomputes `vma->vm_page_prot`, calls `change_protection(...)` to walk PTEs and update `_PAGE_PKEY_*` bits.
- REQ-14: `mprotect_fixup` enforces the `VM_MAY*` cap: requesting `prot` bits not in `(vma->vm_flags >> 4) & VM_ACCESS_FLAGS` ⟹ `-EACCES`.
- REQ-15: `change_protection` updates each PTE via `ptep_modify_prot_start/commit` ensuring TLB shootdown completes before lock release.
- REQ-16: Partial application: on per-VMA `-EACCES` etc., the loop aborts at that VMA; preceding VMAs retain new state (no rollback). Returned errno is the first failure.
- REQ-17: `pkey_mprotect(-1)` is equivalent to `mprotect(2)` semantics; the existing `vma->vm_pkey` is preserved across the protection change.
- REQ-18: `arch_override_mprotect_pkey` on x86_64: when `prot == PROT_EXEC` only and `pkey == -1`, kernel may auto-assign the execute-only pkey via `execute_only_pkey(mm)` for side-channel hardening.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pkey_in_range` | INVARIANT | `pkey ∈ [-1, arch_max_pkey())`. |
| `pkey_allocated` | INVARIANT | `pkey >= 0 ⟹ mm_pkey_is_allocated(mm, pkey)`. |
| `prot_consistent_with_pte` | INVARIANT | Post-call PTE `_PAGE_RW`/`_PAGE_NX` matches `vma->vm_flags`. |
| `pkey_bits_in_pte` | INVARIANT | Post-call PTE pkey bits `[59:62] == vma->vm_pkey`. |
| `lock_held_during_change` | INVARIANT | `change_protection` runs under `mmap_write_lock`. |
| `tlb_flushed_before_unlock` | INVARIANT | TLB invalidated before `mmap_write_unlock`. |

### Layer 2: TLA+

`uapi/syscalls/pkey_mprotect.tla`:
- States: `{vmas, ptes, pkey_allocation_map, mm_locked}`.
- Actions: `Allocate`, `BindPkey`, `Free`, `RaceFreeBindWithExec`.
- Properties:
  - `safety_pkey_must_be_allocated_when_bound`,
  - `safety_pte_pkey_matches_vma_pkey`,
  - `liveness_pkey_mprotect_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `arch_override_mprotect_pkey` may only return -1 or an allocated pkey | `Mm::do_mprotect_pkey` |
| `mprotect_fixup` updates `vma->vm_page_prot` to reflect new pkey | `Mm::mprotect_fixup` |
| Partial-application behavior matches `mprotect(2)` | `Mm::do_mprotect_pkey` |

### Layer 4: Verus/Creusot functional

`pkey_mprotect(addr, len, prot, pkey) ≡ Linux man-page semantics (no POSIX standard).`

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pkey_mprotect(2)` reinforcement:

- **Per `mm_pkey_is_allocated` check** — pkey must be allocated before binding; defense against binding a freed (now-default) pkey.
- **Per `arch_max_pkey()` cap** — denies out-of-range pkey indices.
- **Per `arch_override_mprotect_pkey`** — arch-specific hook can refine pkey selection (e.g., execute-only pkey for `PROT_EXEC`-only ranges; defense against side-channel reads of code pages).
- **Per `VM_MAY*` cap** — `pkey_mprotect` cannot escalate beyond the original mapping's allowed permissions.
- **Per `map_deny_write_exec`** — W^X enforcement at the kernel level; `PROT_WRITE|PROT_EXEC` denied.
- **Per `change_protection` TLB invalidation** — full TLB shootdown before lock release; defense against stale-translation attacks.
- **Per LSM `security_file_mprotect`** — SELinux / AppArmor / Smack observe the change.
- **Per partial-application visibility** — on per-VMA `-EACCES`, return value is the first failure; caller can inspect `vma`s via `/proc/PID/maps`.

### grsecurity/pax-style reinforcement

- **PAX_MPROTECT** — pkey_mprotect cannot upgrade non-EXEC mapping to `PROT_EXEC` under PaX policy; the same W^X enforcement applies. The pkey binding does not bypass PaX page-protection.
- **PAX_PAGEEXEC** — page-level NX enforced; `_PAGE_NX` bit set whenever `VM_EXEC` not in `vma->vm_flags`. Defense against per-pkey W+X attempts.
- **PAX_NOEXEC** — denial of `PROT_EXEC` on anonymous/heap/stack mappings extends to pkey-bound ranges; cannot use pkey_mprotect to grant exec on PaX-forbidden ranges.
- **PAX_RANDMMAP** — irrelevant (no address selection in pkey_mprotect; the address is given).
- **PAX_RANDEXEC** — file-backed PROT_EXEC ranges retain their randomized load address across pkey rebind.
- **GRKERNSEC_RWXMAP_LOG** — any pkey_mprotect that would yield W+X audited (denied or allowed).
- **PAX_REFCOUNT** — VMA refcounts saturating; pkey-allocation-map atomically updated.
- **PAX_UDEREF** — `addr` is numeric; SMAP enforced; PaX user-deref check ensures no kernel-side dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — pages freed by VMA-merge / split after pkey_mprotect are sanitized.
- **GRKERNSEC_BRUTE** — repeated pkey_mprotect failures (e.g., side-channel pkey-bind-then-WRPKRU brute) detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers and PTE values.
- **PAX_RANDKSTACK** — kstack offset rerolled at pkey_mprotect syscall entry.
- **PKRU init / shadow stack interplay** — grsec extends to ensure that pkey changes do not silently override CET (Control-Flow Enforcement Technology) shadow-stack pkey assignments; the shadow-stack pkey is reserved and rebinding it is denied.
- **execute-only pkey hardening** — grsec encourages `arch_override_mprotect_pkey` to assign the execute-only pkey for any `PROT_EXEC`-only range, providing per-pkey side-channel-read defense even when the caller didn't request it.

