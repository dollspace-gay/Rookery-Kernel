---
title: "Tier-5 syscall: pkey_free(2) — syscall 331"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pkey_free(2)` releases a Memory Protection Key previously returned by `pkey_alloc(2)`. The kernel marks the slot in `mm->context.pkey_allocation_map` as free so a future `pkey_alloc` can reuse the number. **It does not touch any VMA**: pages still tagged with the freed key continue to have it as their tag, so faults against those pages will use whatever PKRU bits exist for that key — which after free are undefined per-thread. Userspace is expected to `pkey_mprotect(2)` such ranges back to key 0 (or another live key) before calling `pkey_free`, or accept that those mappings may behave unpredictably.

Critical for: long-running services that allocate temporary keys (JIT regions, secret-memory short-life), graceful library shutdown, container teardown.

### Acceptance Criteria

- [ ] AC-1: `pkey_free(k)` after `pkey_alloc`: returns 0; map bit cleared.
- [ ] AC-2: `pkey_free(k)` twice: second call `-EINVAL`.
- [ ] AC-3: `pkey_free(0)`: `-EINVAL`.
- [ ] AC-4: `pkey_free(-1)`: `-EINVAL`.
- [ ] AC-5: `pkey_free(arch_max_pkey())`: `-EINVAL`.
- [ ] AC-6: On CPU without PKU: `-ENOSYS`.
- [ ] AC-7: After `pkey_free`, `pkey_alloc` returns the same number again.
- [ ] AC-8: Pages still tagged with freed key remain mapped; access may produce SIGSEGV depending on PKRU.
- [ ] AC-9: Concurrent free from two threads on the same key: exactly one succeeds.
- [ ] AC-10: After `execve`, all previously allocated keys cease to be considered allocated.
- [ ] AC-11: Grsec-EBUSY path: with `GRKERNSEC_PKEY_STRICT` and VMA still tagged: `-EBUSY`.

### Architecture

```rust
#[syscall(nr = 331, abi = "sysv")]
pub fn sys_pkey_free(pkey: i32) -> isize {
    Pkey::do_pkey_free(pkey)
}
```

`Pkey::do_pkey_free(pkey) -> isize`:
1. if !arch_pkeys_enabled(): return -ENOSYS;
2. if pkey < 1: return -EINVAL;
3. if (pkey as usize) >= arch_max_pkey(): return -EINVAL;
4. let mm = current().mm;
5. let guard = mm.context.pkey_lock.lock();
6. let map = &mut mm.context.pkey_allocation_map;
7. if !map.test(pkey as usize) {
8.   return -EINVAL;        // not allocated / already freed
9. }
10. /* Grsec hardened mode: ensure no VMA still tagged */
11. #[cfg(GRKERNSEC_PKEY_STRICT)]
12. if Pkey::any_vma_tagged(mm, pkey as u8) {
13.    return -EBUSY;
14. }
15. map.clear(pkey as usize);
16. Ok(0)

`Pkey::any_vma_tagged(mm, pkey) -> bool`:
1. for vma in mm.vmas() {
2.   if vma.pkey == pkey { return true; }
3. }
4. false

### Out of Scope

- `pkey_alloc(2)` mechanics (separate Tier-5 doc).
- `pkey_mprotect(2)` (separate Tier-5 doc / `mprotect.md`).
- PKRU register save/restore on context switch (Tier-3 `arch/x86/kernel/process.md`).
- Implementation code.

### signature

```c
int pkey_free(int pkey);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pkey` | `int` | in | Key previously returned by `pkey_alloc(2)`. Must be in `[1, arch_max_pkey)` and currently allocated in the caller's mm. |

### return value

| Value | Meaning |
|---|---|
| `0` | Key freed. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `pkey < 1`; `pkey >= arch_max_pkey()`; key not currently allocated in caller's mm. |
| `ENOSYS` | Kernel built without `CONFIG_ARCH_HAS_PKEYS` / hardware lacks PKU/MPK. |
| `EBUSY` | Grsec-hardened: any VMA still tagged with this pkey (refusing free until ranges re-tagged). |

### abi surface

```text
__NR_pkey_free  (x86_64)  = 331
__NR_pkey_free  (arm64)   = 290
__NR_pkey_free  (riscv)   = 290
__NR_pkey_free  (i386)    = 382
```

### compatibility contract

REQ-1: Syscall number is **331** on x86_64. ABI-stable since Linux 4.9.

REQ-2: `pkey` validation: `1 ≤ pkey < arch_max_pkey()`. Else `-EINVAL`.

REQ-3: Per-mm allocation check: `mm_pkey_is_allocated(mm, pkey)` must return true; otherwise `-EINVAL`. Prevents double-free and bogus-free.

REQ-4: Per-mm bitmap update: atomically clears bit `pkey` in `mm->context.pkey_allocation_map`. Holds `mm->context.pkey_lock`.

REQ-5: No VMA scan; the kernel does not auto-retag pages on free. Caller must `pkey_mprotect(addr, len, prot, 0)` first if it wants those pages to use key 0.

REQ-6: No PKRU update: the calling thread's PKRU bits for `pkey` are left as-is. (A future `pkey_alloc` that returns the same number will re-program PKRU with the new owner's `access_rights`.)

REQ-7: Key 0 cannot be freed: `pkey_free(0)` ⟹ `-EINVAL`.

REQ-8: Concurrent `pkey_free(k)` from sibling threads (same mm): one succeeds (returns 0), the other gets `-EINVAL` (already free).

REQ-9: `fork(2)`: parent's allocation map copied; child's free of an inherited key only frees the child's bit. Note: upstream's allocation map is per-mm and not copied across fork — child starts empty.

REQ-10: `execve(2)`: mm replaced; allocation map cleared (implicit mass-free).

REQ-11: `mm` teardown (`exit_mmap`) iterates and frees any remaining keys (implicit; no userspace return).

REQ-12: Not blocking; not interruptible.

REQ-13: No special capability required; key only affects caller's own mm.

REQ-14: Per-thread PKRU is not synchronized across siblings on free; if sibling thread still holds restricted PKRU bits for `pkey`, those bits persist until that thread updates them.

REQ-15: Pages tagged with a freed key continue to map (PTE pkey field unchanged); access behavior depends on each accessing thread's PKRU value for that key number.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pkey_bounds` | INVARIANT | pkey ∈ [1, arch_max_pkey) ∨ EINVAL. |
| `key_zero_protected` | INVARIANT | pkey_free(0) ⟹ EINVAL. |
| `double_free_rejected` | INVARIANT | already-free ⟹ EINVAL. |
| `allocation_lock_held` | INVARIANT | map mutation under pkey_lock. |
| `no_pte_update` | INVARIANT | success path does not modify any PTE. |

### Layer 2: TLA+

`mm/pkey-free.tla`:
- States: per-validate, per-test-allocated, per-clear.
- Properties:
  - `safety_unique_free_winner` — concurrent free serializes; only one returns 0.
  - `safety_key_zero_invariant`.
  - `safety_no_vma_mutation`.
  - `liveness_pkey_free_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pkey_free` post: success ⟹ map.bit(pkey) = 0 ∧ no VMA mutated | `Pkey::do_pkey_free` |
| `any_vma_tagged` post: returns iff exists vma with vma.pkey == pkey | `Pkey::any_vma_tagged` |

### Layer 4: Verus / Creusot functional

Per-`pkey_free(2)` man-page semantic equivalence. selftests/x86/protection_keys parity.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`pkey_free(2)` reinforcement:

- **Per-mm-lock atomic free** — defense against per-concurrent-free races.
- **Per-key-allocated precondition** — defense against per-double-free / bogus-free.
- **Per-key-zero protected** — defense against per-default-page mis-attribution.
- **Per-arch-bound validation** — defense against per-undefined PKRU bit.
- **Per-no-PTE-update** — defense against per-implicit-retag surprise.

### grsecurity / pax surface

- **PaX UDEREF parity** — no user pointers, but full SMAP-strict enforcement on the syscall path.
- **GRKERNSEC_PKEY_STRICT EBUSY when VMA tagged** — grsec adds a paranoid check refusing free while any VMA in the mm is still tagged with the key. Forces userspace to be explicit: `pkey_mprotect(..., new_key)` before `pkey_free`. Eliminates the "use-after-free of pkey number" hazard where the same number is later re-allocated and unexpectedly applies to dangling tagged ranges.
- **GRKERNSEC_PKEY_QUOTA decrement** — pairs with the alloc-side quota; ensures bounded quota accounting on every free.
- **PAX_REFCOUNT on mm pkey_alloc_count** — defense against per-counter underflow.
- **PaX KERNSEAL on `arch_pkey_ops`** — table read-only post-init.
- **GRKERNSEC_MEM /dev/mem block on PKRU shadow** — pkey_free is the only legitimate path to release a key; `/dev/mem` cannot manipulate the allocation map.
- **GRKERNSEC_RBAC pkey_free ACL** — subjects denied entirely.
- **Per-execve double-checked free** — `execve` path that mass-frees keys validates each before clearing; defense against per-corrupted-map.
- **Per-pkey audit** — grsec audit record on each free if `pkey_audit` enabled.
- **PKRU-scrub on free (optional)** — grsec optionally rewrites the calling thread's PKRU to clear AD/WD bits for the freed key, eliminating the per-stale-PKRU information channel that the upstream "no PKRU update" semantic leaves open.
- **Per-CONFIG_PAX_PKEY_STRICT** — entire MPK API disabled; `pkey_free` returns `-ENOSYS`.

