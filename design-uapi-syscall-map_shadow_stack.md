---
title: "Tier-5 syscall: map_shadow_stack(2) — syscall 453"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`map_shadow_stack(2)` allocates an Intel CET (Control-flow Enforcement Technology) **shadow stack** region in the calling process's address space. The kernel maps `size` bytes (rounded up to a page) as `PROT_READ | VM_SHADOW_STACK`, optionally writes a restore-token at the top so that a later `RSTORSSP`/`SAVEPREVSSP` instruction can switch to this stack, and returns the base of the mapping.

The mapping is **architecturally special**: it is read-only to userspace except via the dedicated CET store instructions, the page-table bits encode "shadow stack" (D=0,W=1 on x86), and ordinary `mmap(PROT_WRITE)` cannot produce equivalent pages. Without this syscall there is no userspace path to create additional shadow stacks (for new threads, fibers, ucontexts).

Critical for: per-thread shadow-stack provisioning under glibc/musl when CET-SS is enabled, makecontext/setcontext / boost::context user-mode fibers, sigaltstack-style alternate-stack for SIGSEGV handlers under CET.

### Acceptance Criteria

- [ ] AC-1: CET-SS enabled, `size = 8192, flags = SHADOW_STACK_SET_TOKEN`: returns page-aligned base; token at `base + 8192 - 8` matches `(base + 8192 - 8) | 1`.
- [ ] AC-2: CET-SS disabled returns `-EOPNOTSUPP`.
- [ ] AC-3: `size = 0` returns `-EINVAL`.
- [ ] AC-4: `flags = 0x100` returns `-EINVAL`.
- [ ] AC-5: Ordinary `STORE` to the returned base traps with SIGSEGV.
- [ ] AC-6: `mprotect(base, size, PROT_WRITE)` returns `-EINVAL`.
- [ ] AC-7: `munmap(base, size)` succeeds.
- [ ] AC-8: `fork()` then child uses the shadow stack: works (COW).
- [ ] AC-9: RLIMIT_STACK exceeded returns `-EAGAIN`.
- [ ] AC-10: Non-CET arch returns `-ENOSYS`.
- [ ] AC-11: `SHADOW_STACK_SET_MARKER` writes both token and previous-SSP marker.

### Architecture

```rust
#[syscall(nr = 453, abi = "sysv")]
pub fn sys_map_shadow_stack(
    addr: usize,
    size: usize,
    flags: u32,
) -> isize {
    Shstk::map(addr, size, flags)
}
```

`Shstk::map(addr, size, flags) -> isize`:
1. if !current.has_shstk_enabled() { return Err(EOPNOTSUPP); }
2. if size == 0 || size > usize::MAX / 2 { return Err(EINVAL); }
3. let known = SHADOW_STACK_SET_TOKEN | SHADOW_STACK_SET_MARKER;
4. if (flags & !known) != 0 { return Err(EINVAL); }
5. let size = page_align_up(size);
6. let hint = if addr == 0 { None } else { Some(addr) };
7. let base = current.mm().mmap_anonymous(
8.     hint,
9.     size,
10.    Prot::Read,
11.    MmapFlags::Private | MmapFlags::Anonymous | MmapFlags::ShadowStack,
12. )?;                                                          // ENOMEM / EAGAIN
13. if flags & SHADOW_STACK_SET_TOKEN != 0 {
14.    let token_va = base + size - 8;
15.    Shstk::write_token(token_va, token_va as u64 | 0x1)?;
16. }
17. if flags & SHADOW_STACK_SET_MARKER != 0 {
18.    let marker_va = base + size - 16;
19.    Shstk::write_marker(marker_va, 0)?;
20. }
21. Ok(base as isize)

`Shstk::write_token(va, value) -> Result<()>`:
1. /* Use the kernel's CET-aware shadow-stack write helper.
2.    Ordinary store-to-user (`copy_to_user`) is rejected for SS pages by hw;
3.    use the WRUSSQ-equivalent kernel intrinsic. */
4. unsafe { x86::wruss_q(va, value)? };
5. Ok(())

### Out of Scope

- arm64 GCS (Guarded Control Stack) wire-up (Tier-3 `arch/arm64/gcs.md` when landed).
- RISC-V Zicfiss support (Tier-3 `arch/riscv/zicfiss.md`).
- `arch_prctl(ARCH_SHSTK_*)` (see `arch_prctl.md` Tier-5).
- Implementation code.

### signature

```c
void *map_shadow_stack(
    unsigned long addr,     /* hint or 0 */
    unsigned long size,
    unsigned int flags
);
```

```c
/* flags */
#define SHADOW_STACK_SET_TOKEN     0x1
#define SHADOW_STACK_SET_MARKER    0x2
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `unsigned long` | in | Hint for placement; `0` lets the kernel choose. If non-zero, kernel may still choose elsewhere. |
| `size` | `unsigned long` | in | Size in bytes; rounded up to page. Must be > 0 and ≤ ULONG_MAX/2. |
| `flags` | `unsigned int` | in | `SHADOW_STACK_SET_TOKEN`, `SHADOW_STACK_SET_MARKER`. Unknown bits ⟹ `-EINVAL`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` (page-aligned address) | Base of the new shadow-stack mapping. |
| `< 0` (errno-negative) | Failure; mapping not created. |

### errors

| errno | Trigger |
|---|---|
| `EOPNOTSUPP` | CPU lacks CET-SS, kernel built without `CONFIG_X86_USER_SHADOW_STACK`, or the calling process has not enabled CET-SS via `arch_prctl(ARCH_SHSTK_ENABLE)`. |
| `EINVAL` | `size == 0`, unknown `flags` bits, `addr` not page-aligned (when non-zero), `size` overflows. |
| `ENOMEM` | Address space exhausted, vma slab exhausted, RLIMIT_AS would be exceeded. |
| `EAGAIN` | RLIMIT_STACK or memcg low memory makes allocation infeasible. |
| `EFAULT` | (internal) failure to write the restore-token. |
| `ENOSYS` | Architecture without CET-SS (arm64, riscv, ...) returns `-ENOSYS`. |

### abi surface

```text
__NR_map_shadow_stack  (x86_64) = 453
__NR_map_shadow_stack  (i386)   = 453
__NR_map_shadow_stack  (arm64)  = 453  /* compile-stub: -ENOSYS unless GCS lands */
__NR_map_shadow_stack  (riscv)  = 453  /* compile-stub: -ENOSYS unless Zicfiss */

/* Note: while the syscall number is allocated cross-arch, only x86_64 with
   CET-SS produces a real shadow stack today. arm64 (GCS / Guarded Control
   Stack) and RISC-V (Zicfiss) will land in subsequent releases. */
```

### compatibility contract

REQ-1: Syscall number is **453** on x86_64. ABI-stable.

REQ-2: CET-SS MUST be enabled for the calling process; otherwise `-EOPNOTSUPP`.

REQ-3: `size == 0` ⟹ `-EINVAL`. There is no "destroy the shadow stack" path here — use `munmap(2)`.

REQ-4: Returned address is page-aligned and references a VMA flagged `VM_SHADOW_STACK`.

REQ-5: The VMA is NOT readable / writable by ordinary instructions: a regular load returns the value but stores will `#PF` (write-protect for non-shadow-stack accesses); only `WRSS*`, `CALL`, `RET`, and INCSSP-family instructions touch the page through the SS-bit path.

REQ-6: With `SHADOW_STACK_SET_TOKEN`, the kernel writes a restore-token (`base + size - 8`) so a subsequent `RSTORSSP` can switch to this stack. The token's value is `(base + size - 8) | 0x1` per Intel SDM.

REQ-7: With `SHADOW_STACK_SET_MARKER`, the kernel additionally writes a "previous SSP" marker slot above the token. Used for nested ucontext / fiber switching with `SAVEPREVSSP`.

REQ-8: The mapping is `MAP_PRIVATE | MAP_ANONYMOUS | VM_SHADOW_STACK` and is account-charged like an anonymous mapping.

REQ-9: `fork(2)` COW-duplicates the shadow stack; `execve(2)` discards it (then `ld.so` calls `map_shadow_stack` again).

REQ-10: `munmap(2)` of a shadow-stack region is allowed and frees both the VMA and the underlying pages; live use after free is the caller's responsibility.

REQ-11: `mprotect(2)` on a shadow-stack VMA is rejected: the page-protection bits are architecturally fixed.

REQ-12: `mremap(2)` of a shadow-stack VMA is supported only with `MREMAP_DONTUNMAP=0`; relocation preserves the SS marker.

REQ-13: `madvise(MADV_DONTNEED)` on a shadow-stack VMA is supported; it discards pages, faults will re-zero, the token at top will be re-materialized on next access if `SHADOW_STACK_SET_TOKEN` was originally set.

REQ-14: RLIMIT_STACK governs the total shadow-stack budget per process (sum of all shadow stacks).

REQ-15: `size` is bounded by `ULONG_MAX / 2` to keep arithmetic on `addr + size` safe.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cet_enabled_precondition` | INVARIANT | CET-SS disabled ⟹ no VMA allocated. |
| `size_zero_rejected` | INVARIANT | `size == 0` ⟹ `-EINVAL`. |
| `flags_validated` | INVARIANT | unknown `flags` bits ⟹ `-EINVAL`. |
| `vma_marked_shadow_stack` | INVARIANT | success ⟹ VMA has VM_SHADOW_STACK. |
| `token_at_top_minus_8` | INVARIANT | SET_TOKEN ⟹ `[top-8] == top-8|1`. |
| `failure_no_partial_vma` | INVARIANT | error during token write ⟹ VMA torn down. |

### Layer 2: TLA+

`arch/x86/shstk-map.tla`:
- Per-call transitions: precond, alloc-vma, write-token, write-marker.
- Properties:
  - `safety_atomic` — success ⟹ VMA + token coherent.
  - `safety_no_token_without_vma` — token never written without a backing VMA.
  - `safety_no_priv_raise` — no operation widens shadow-stack permission to write.
  - `liveness_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Shstk::map` post: success ⟹ ret is page-aligned VMA base | `Shstk::map` |
| `Shstk::map` post: VMA flags include VM_SHADOW_STACK | `Shstk::map` |
| `Shstk::write_token` post: token == (va | 1) | `Shstk::write_token` |
| `Shstk::write_marker` post: marker slot zeroed | `Shstk::write_marker` |

### Layer 4: Verus / Creusot functional

Per-`map_shadow_stack(2)` man page; Intel CET SDM Vol 1 §17 (Shadow Stack); glibc `__init_main_arena_shadow_stack` semantics.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`map_shadow_stack(2)` reinforcement:

- **Per-CET-SS-enabled precondition** — defense against per-non-CET process spuriously gaining write-prohibited VMA.
- **Per-`size > 0` strict** — defense against per-zero-size VMA splatter.
- **Per-`flags` allow-list** — defense against per-future-flag smuggle.
- **Per-VMA `VM_SHADOW_STACK` immutable** — defense against per-mprotect upgrade-to-write.
- **Per-token format fixed** — defense against per-forged-token RSTORSSP.

### grsecurity / pax surface

- **map_shadow_stack as CET hardware enforcement** — `GRKERNSEC_CFI_USER` makes `map_shadow_stack(2)` the sole user-mode shadow-stack allocator and refuses any `mmap(PROT_WRITE)` or `mprotect` that would emulate an SS region in software. Defense against per-CFI bypass via faked SS pages.
- **PaX UDEREF / SMAP** — kernel-side token writes use the `WRUSSQ` intrinsic; ordinary `copy_to_user` to SS pages is architecturally rejected, so SMAP and UDEREF block any kernel bug-pattern that would store data via the wrong path.
- **PAX_REFCOUNT on `mm->shstk_count`** — defense against per-refcount UAF when concurrent threads allocate/munmap shadow stacks.
- **Per-RLIMIT_STACK enforced on shadow-stack budget** — defense against per-DoS via unbounded SS allocation.
- **GRKERNSEC_HIDESYM on returned base** — base address is randomized by ASLR; kernel-side metadata (per-thread `shstk_base`) not exposed via `/proc`.
- **PaX KERNEXEC on `WRUSSQ` callsite** — defense against per-W^X violation; the kernel WRUSSQ helper lives in `.text` only.
- **GRKERNSEC_FORK_SS_COW_AUDIT** — fork-copying of shadow stack audited for excessive duplication (signal of attack pattern).
- **Per-`mprotect` rejection on VM_SHADOW_STACK** — defense against per-permission-flip CFI bypass.
- **Per-`mremap(MREMAP_DONTUNMAP)` rejected** — defense against per-aliasing CFI bypass.
- **Per-`madvise(MADV_FREE)` rejected** — `MADV_DONTNEED` only; defense against per-stale-data side channel.
- **Per-`execve` discards shadow stacks** — defense against per-cross-exec CFI bypass.
- **PAX_USERCOPY on token-marker writes** — bounded WRUSSQ paths use whitelisted slab-touching helper.
- **Per-arch ENOSYS path** — defense against per-cross-arch fingerprint of CET capability from unprivileged userspace running under emulation.

