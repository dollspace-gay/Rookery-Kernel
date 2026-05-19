---
title: "Tier-5 syscall: pkey_alloc(2) — syscall 330"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pkey_alloc(2)` allocates a **Memory Protection Key** (MPK / Intel PKU on x86_64, MTE-like tagging on arm64) from the per-mm 16-slot pool. The key is a 4-bit value (0..15 on x86_64; key 0 reserved). The caller specifies initial PKRU access-rights (`PKEY_DISABLE_ACCESS` / `PKEY_DISABLE_WRITE`) which the kernel programs into the calling thread's PKRU register. A successfully allocated key is intended to be used with `pkey_mprotect(2)` to associate ranges of VMA with the key, and with `WRPKRU` / `pkey_set` to change a thread's access rights.

Critical for: in-process memory isolation (CRIU, JIT, OpenSSL secret memory, sandboxing libraries like memprotect), zero-syscall sandboxing.

### Acceptance Criteria

- [ ] AC-1: `pkey_alloc(0, 0)`: returns key ≥ 1 on PKU-capable CPU.
- [ ] AC-2: Loop allocating until `-ENOSPC`: 15 successful allocations max on x86_64.
- [ ] AC-3: `flags = 1`: `-EINVAL`.
- [ ] AC-4: `access_rights = 0x4`: `-EINVAL`.
- [ ] AC-5: On CPU without PKU: `-ENOSYS`.
- [ ] AC-6: `pkey_alloc(0, PKEY_DISABLE_ACCESS)`: returns key; PKRU now has access disabled for that key in the calling thread.
- [ ] AC-7: After `pkey_alloc` + `pkey_mprotect` + dereference: SIGSEGV with `si_code == SEGV_PKUERR`.
- [ ] AC-8: Allocation in parent thread, then accessed in sibling thread (different PKRU): sibling's access depends on its own PKRU.
- [ ] AC-9: `fork()` + child `pkey_alloc`: child gets first free key (no inherited allocations from parent).
- [ ] AC-10: `execve()` clears allocation map; subsequent `pkey_alloc` returns 1.
- [ ] AC-11: Per-mm key allocations independent: two threads in one mm share the allocation map (lock-protected).

### Architecture

```rust
#[syscall(nr = 330, abi = "sysv")]
pub fn sys_pkey_alloc(flags: u32, access_rights: u32) -> isize {
    Pkey::do_pkey_alloc(flags, access_rights)
}
```

`Pkey::do_pkey_alloc(flags, access_rights) -> isize`:
1. if flags != 0: return -EINVAL;
2. if access_rights & !PKEY_ACCESS_MASK != 0: return -EINVAL;
3. if !arch_pkeys_enabled(): return -ENOSYS;
4. let mm = current().mm;
5. /* Allocate slot atomically */
6. let pkey = {
7.   let guard = mm.context.pkey_lock.lock();
8.   let map = &mut mm.context.pkey_allocation_map;
9.   match map.first_free_ge(1, arch_max_pkey()) {
10.    Some(k) => { map.set(k); k as i32 }
11.    None => return -ENOSPC,
12.  }
13. };
14. /* Program PKRU for current thread */
15. Pkey::arch_set_user_pkey_access(pkey, access_rights);
16. Ok(pkey as isize)

`Pkey::arch_set_user_pkey_access(pkey, rights)` (x86_64):
1. /* PKRU has 2 bits per key: AD (Access Disable), WD (Write Disable) */
2. let shift = pkey * 2;
3. let mut pkru = read_pkru();
4. pkru &= !(0b11 << shift);
5. if rights & PKEY_DISABLE_ACCESS != 0 { pkru |= (1 << shift); }
6. if rights & PKEY_DISABLE_WRITE  != 0 { pkru |= (1 << (shift + 1)); }
7. write_pkru(pkru);
8. /* PKRU saved/restored across context-switch by xsave */

### Out of Scope

- `pkey_mprotect(2)` mechanics (separate Tier-5 doc).
- `pkey_set` / PKRU manipulation (covered in `arch/x86/mm/pkeys.md` Tier-3).
- arm64 MTE-key semantics (Tier-3 arch doc).
- Implementation code.

### signature

```c
int pkey_alloc(unsigned int flags, unsigned int access_rights);
```

```c
#define PKEY_DISABLE_ACCESS  0x1
#define PKEY_DISABLE_WRITE   0x2
#define PKEY_ACCESS_MASK     (PKEY_DISABLE_ACCESS | PKEY_DISABLE_WRITE)
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `flags` | `unsigned int` | in | Reserved; must be 0. |
| `access_rights` | `unsigned int` | in | Bitmask: `PKEY_DISABLE_ACCESS` (read+write blocked) and/or `PKEY_DISABLE_WRITE` (write blocked). |

### return value

| Value | Meaning |
|---|---|
| `>= 1` | Allocated key (1..15 on x86_64). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags != 0`; `access_rights` has unknown bits. |
| `ENOSPC` | All keys allocated (or all but the reserved key 0) for this mm; or platform does not support pkeys. |
| `ENOSYS` | Kernel built without `CONFIG_ARCH_HAS_PKEYS` / hardware lacks PKU/MPK. |
| `EPERM` | Grsec hard-cap on per-process key allocations exceeded. |

### abi surface

```text
__NR_pkey_alloc  (x86_64)  = 330
__NR_pkey_alloc  (arm64)   = 289
__NR_pkey_alloc  (riscv)   = 289
__NR_pkey_alloc  (i386)    = 381

/* Returned key is a small non-negative integer. */
/* x86_64: keys 0..15 (16 slots; key 0 always usable for unspecified pages). */
```

### compatibility contract

REQ-1: Syscall number is **330** on x86_64. ABI-stable since Linux 4.9.

REQ-2: `flags` validation: must be 0; reserved for future extension. Non-zero ⟹ `-EINVAL`.

REQ-3: `access_rights` validation: only `PKEY_DISABLE_ACCESS | PKEY_DISABLE_WRITE` bits permitted; other bits ⟹ `-EINVAL`.

REQ-4: Per-mm allocation: `mm->context.pkey_allocation_map` is a bitmap of 16 slots; key 0 always reserved (free pages start in key 0).

REQ-5: Search picks the lowest-numbered free key ≥ 1; if none, `-ENOSPC`.

REQ-6: Initial PKRU programming: kernel performs `init_pkru_value` then ANDs in caller's `access_rights` so the new key starts with those restrictions in the calling thread.

REQ-7: PKRU is per-thread (a CPU register); allocation only programs *current thread's* PKRU. Sibling threads in the same mm continue to use their existing PKRU value until they explicitly use `pkey_set(key, rights)`.

REQ-8: Allocation is per-mm; survives across threads and is freed only via `pkey_free(2)` or mm teardown (`exit_mmap`).

REQ-9: `fork(2)`: child mm starts with empty allocation map (no inherited keys); the cloned PKRU is a snapshot but newly allocated keys are not "live" without re-alloc.

REQ-10: `execve(2)`: full mm replacement; allocation map cleared.

REQ-11: `CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS` (or arm64 MTE-equivalent) required; otherwise `-ENOSYS`.

REQ-12: Per-platform key count: x86_64 = 16 (4-bit key field in PTE); arm64 MTE = 16 tag values; arch-specific via `arch_max_pkey()`.

REQ-13: Allocation does not associate the key with any address range; `pkey_mprotect(2)` is required to tag VMA(s).

REQ-14: No special capability required (per upstream); the key only restricts memory the caller already controls.

REQ-15: Key 0 (default) cannot be allocated/freed; it is implicitly used for any page lacking an explicit pkey.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_reserved_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `access_rights_mask` | INVARIANT | rights ⊆ PKEY_ACCESS_MASK. |
| `key_zero_never_returned` | INVARIANT | returned key ≥ 1. |
| `allocation_atomic_under_lock` | INVARIANT | pkey_lock held during map update. |
| `pkru_bits_consistent` | INVARIANT | per-key 2 bits programmed in PKRU. |

### Layer 2: TLA+

`mm/pkey-alloc.tla`:
- States: per-validate, per-allocate-slot, per-program-pkru.
- Properties:
  - `safety_unique_key_allocation` — two concurrent allocs return distinct keys.
  - `safety_key_zero_reserved`.
  - `safety_no_program_on_oom`.
  - `liveness_pkey_alloc_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pkey_alloc` post: success ⟹ key ∈ [1, arch_max_pkey) ∧ map.bit(key) = 1 | `Pkey::do_pkey_alloc` |
| `arch_set_user_pkey_access` post: PKRU AD/WD bits for key match rights | `Pkey::arch_set_user_pkey_access` |

### Layer 4: Verus / Creusot functional

Per-`pkey_alloc(2)` man-page semantic equivalence. selftests/x86/protection_keys parity.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`pkey_alloc(2)` reinforcement:

- **Per-mm-lock atomic allocation** — defense against per-concurrent-alloc duplicate-key race.
- **Per-key-zero reserved** — defense against per-default-page mis-attribution.
- **Per-flags-reserved-zero** — defense against per-extension-field smuggling.
- **Per-rights-mask validated** — defense against per-undefined PKRU bits.
- **Per-arch-pkeys-enabled gate** — defense against per-no-hardware spurious success.

### grsecurity / pax surface

- **PaX UDEREF parity** — though no user pointers in this syscall, hardened SMAP enforced throughout.
- **Per-process pkey-quota** — grsec introduces `GRKERNSEC_PKEY_QUOTA`: caps the number of allocated pkeys per `task->signal` (default = arch_max_pkey, but configurable lower e.g. 4). Above the cap ⟹ `-EPERM`. Designed to limit a compromised library from monopolizing the keyspace.
- **GRKERNSEC_MEM block on /dev/mem-driven PKRU manipulation** — `/dev/mem` access to anything that could shadow PKRU prevented; `pkey_alloc` is the only legitimate path to mutate PKRU bits.
- **PaX PKRU-zero-on-exec** — `execve` clears PKRU to "no restrictions" while clearing the allocation map; prevents per-stale-restriction inheritance across SUID transitions.
- **PaX KERNSEAL on `arch_pkey_ops` table** — read-only post-init.
- **PAX_REFCOUNT on mm->context.pkey_alloc_count** — defense against per-counter wrap.
- **GRKERNSEC_RBAC pkey_alloc ACL** — subjects can be denied this syscall entirely (e.g., libraries that should never use MPK).
- **Per-pkey audit** — grsec optionally logs each pkey_alloc/pkey_free to detect anomalous allocation patterns (e.g., a JIT compromised to exhaust the keyspace).
- **PKRU consistency check on context-switch** — grsec adds a paranoid check that the saved PKRU matches the per-mm allocation map's "disabled" bits, catching state-corruption attacks.
- **Per-CONFIG_PAX_PKEY_STRICT** — disables `pkey_alloc` entirely in maximum-hardened mode; relies on segregation via `mprotect`+ASLR instead.

