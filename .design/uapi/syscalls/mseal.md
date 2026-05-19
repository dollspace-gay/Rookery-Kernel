# Tier-5 syscall: mseal(2) — syscall 462

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mseal.c (SYSCALL_DEFINE3(mseal), do_mseal, can_modify_vma)
  - mm/mmap.c (mprotect/munmap/mremap mseal hooks)
  - include/uapi/linux/mman.h (MSEAL_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (462  common  mseal)
-->

## Summary

`mseal(2)` **irreversibly** marks one or more VMAs as sealed. A sealed VMA cannot be:
- unmapped (`munmap`),
- remapped (`mremap`),
- changed in protection (`mprotect`, `pkey_mprotect`),
- expanded/shrunk (`brk` is unaffected for the heap; `MAP_FIXED` cannot overwrite a sealed region),
- discarded (`madvise(MADV_DONTNEED|MADV_FREE|MADV_REMOVE|MADV_COLD|MADV_PAGEOUT|MADV_WIPEONFORK)`).

The seal applies for the lifetime of the mm; there is no `munseal`. Designed as a defense-in-depth primitive: once executable text, vDSO, JIT-compiled-and-finalised code, or read-only data segments are sealed, an attacker who later achieves arbitrary syscall execution cannot use VMA mutation primitives to swap RX pages or downgrade page protections.

Critical for: hardened userspace (glibc, musl, OpenSSH), W^X enforcement, JIT runtimes (V8, JSC, OpenJDK) that finalise compiled code, and any process that wants to commit to an immutable code layout post-startup.

## Signature

```c
int mseal(unsigned long start, size_t len, unsigned long flags);
```

```c
/* Currently no flag bits are defined; `flags` must be 0. */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `start` | `unsigned long` | in | Page-aligned starting virtual address. |
| `len` | `size_t` | in | Length in bytes; rounded up to page boundary. |
| `flags` | `unsigned long` | in | Reserved; must be 0. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; every VMA in `[start, start+len)` is now sealed. |
| `-1` + `errno` | Failure; **no partial sealing**: either all VMAs sealed or none. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `start` not page-aligned; `flags != 0`; range overflows `TASK_SIZE`. |
| `ENOMEM` | Range contains a gap (unmapped page) — sealing requires fully-mapped range. |
| `EPERM` | Range contains a VMA that cannot be sealed (e.g. some VM_SPECIAL types depending on policy). |
| `EFAULT` | (rare) internal mm walk fault. |

## ABI surface

```text
__NR_mseal (x86_64) = 462
__NR_mseal (arm64)  = 462
__NR_mseal (riscv)  = 462
__NR_mseal (i386)   = 462

/* Sealing is irreversible. There is no munseal(). */
```

## Compatibility contract

REQ-1: Syscall number is **462** on x86_64. Introduced in Linux 6.10.

REQ-2: `flags` must be 0. Non-zero ⟹ `-EINVAL` (reserved for future seal types).

REQ-3: `start` must be page-aligned; `len` is rounded up to PAGE_SIZE. `start + len` must not overflow `TASK_SIZE` ⟹ `-EINVAL`.

REQ-4: Range fully-mapped check: every page in `[start, start+len)` must lie within an existing VMA; gaps ⟹ `-ENOMEM`. Atomic: no partial sealing on `ENOMEM`.

REQ-5: For every VMA fully or partially in the range, the kernel sets `VM_SEALED` on the VMA. VMAs partially covered may be split so only the covered portion is sealed (`vma_modify_seal` performs split-then-set).

REQ-6: Sealing is **irreversible** and **inherited** by `fork(2)` into the child mm. `execve(2)` discards the old mm and resets seals (because the address space is replaced).

REQ-7: Operations rejected on a sealed VMA:
- `munmap(2)` on any byte in the VMA ⟹ `-EPERM`.
- `mremap(2)` (including resize and move) ⟹ `-EPERM`.
- `mprotect(2)`, `pkey_mprotect(2)` that *change* protection ⟹ `-EPERM` (no-op mprotect to same flags is permitted).
- `mmap(MAP_FIXED, ...)` that would overwrite any sealed page ⟹ `-EPERM`.
- `madvise(2)` destructive advices: `DONTNEED`, `FREE`, `REMOVE`, `COLD`, `PAGEOUT`, `WIPEONFORK` ⟹ `-EPERM`.

REQ-8: Operations *permitted* on a sealed VMA:
- Read, write, execute according to current prot.
- `mlock`/`munlock`.
- Page faults populating the VMA.
- `madvise(MADV_NORMAL|RANDOM|SEQUENTIAL|WILLNEED)`.

REQ-9: No capability required. `mseal` is voluntary self-restriction by the calling process.

REQ-10: Sealed VMA boundaries cannot be merged with adjacent unsealed VMAs (VM_SEALED differs); affects `/proc/self/maps` layout.

REQ-11: For interaction with grsecurity `PAX_MPROTECT`: where `PAX_MPROTECT` already enforces strict W^X on the entire process, `mseal` is the per-VMA, post-hoc analogue — a process may turn on W^X for specific pages without enabling whole-process PAX_MPROTECT.

## Acceptance Criteria

- [ ] AC-1: `mseal(addr, 4096, 0)` on a valid VMA returns 0; subsequent `mprotect(addr, 4096, PROT_READ)` returns `-EPERM`.
- [ ] AC-2: `munmap(addr, 4096)` on sealed range returns `-EPERM`; mapping still present.
- [ ] AC-3: `mremap(addr, 4096, 8192, 0)` returns `-EPERM`.
- [ ] AC-4: `mmap(MAP_FIXED|MAP_ANONYMOUS, ..., addr, 4096, ...)` over sealed returns `-EPERM`.
- [ ] AC-5: `madvise(addr, 4096, MADV_DONTNEED)` returns `-EPERM`; `MADV_WILLNEED` succeeds.
- [ ] AC-6: Range with a gap returns `-ENOMEM`; no VMA in the range is sealed.
- [ ] AC-7: `flags = 1` returns `-EINVAL`.
- [ ] AC-8: Child after `fork()` inherits seal; same sealed-region ops rejected.
- [ ] AC-9: After `execve()`, seals are gone (new mm).
- [ ] AC-10: `mprotect` to *same* prot on sealed range returns 0 (no-op allowed).

## Architecture

```rust
#[syscall(nr = 462, abi = "sysv")]
pub fn sys_mseal(start: usize, len: usize, flags: usize) -> isize {
    Mseal::do_mseal(start, len, flags)
}
```

`Mseal::do_mseal(start, len, flags) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. if start & (PAGE_SIZE - 1) != 0 { return -EINVAL; }
3. let end = start.checked_add(round_up(len, PAGE_SIZE)).ok_or(EINVAL)?;
4. if end > TASK_SIZE { return -EINVAL; }
5. if start == end { return 0; }
6. let mm = current_mm();
7. mm.mmap_write_lock();
8. /* Pre-flight: range fully mapped */
9. if !Mm::range_fully_mapped(mm, start, end) {
10.   mm.mmap_write_unlock();
11.   return -ENOMEM;
12. }
13. /* Pre-flight: every VMA in range is seal-eligible */
14. for vma in mm.vmas_in_range(start, end) {
15.   if !Mseal::can_seal(vma) {
16.     mm.mmap_write_unlock();
17.     return -EPERM;
18.   }
19. }
20. /* Apply: split as needed, set VM_SEALED */
21. let mut cur = start;
22. while cur < end {
23.   let vma = mm.find_vma(cur).expect("checked above");
24.   let vstart = max(vma.vm_start, start);
25.   let vend = min(vma.vm_end, end);
26.   Mm::vma_modify_seal(mm, vma, vstart, vend);  // splits + ORs VM_SEALED
27.   cur = vend;
28. }
29. mm.mmap_write_unlock();
30. 0

`Mseal::check_can_modify_vma(vma, op) -> Result<()>`:
1. if !(vma.vm_flags & VM_SEALED) { return Ok(()); }
2. match op {
3.   Op::Munmap | Op::Mremap | Op::MprotectChange => Err(EPERM),
4.   Op::MadviseDestructive => Err(EPERM),
5.   Op::MmapFixedOverwrite => Err(EPERM),
6.   _ => Ok(())
7. }

Hooks added to `munmap`, `mremap`, `mprotect`, `mmap(MAP_FIXED)`, `madvise` invoke `check_can_modify_vma` before mutating state.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `page_aligned` | INVARIANT | unaligned start ⟹ EINVAL. |
| `range_fully_mapped` | INVARIANT | gap ⟹ ENOMEM and no VMA modified. |
| `atomic_seal` | INVARIANT | failure ⟹ zero VM_SEALED bits set in the range. |
| `irreversible` | INVARIANT | no syscall path clears VM_SEALED post-mseal (only mm replacement clears). |
| `inherit_on_fork` | INVARIANT | dup_mm() copies VM_SEALED. |

### Layer 2: TLA+

`mm/mseal.tla`:
- States: validate-flags, validate-range, lock-mm, check-mapped, check-eligible, split-and-set, unlock.
- Properties:
  - `safety_no_partial_seal` — ENOMEM/EPERM path leaves no VMA modified.
  - `safety_irreversibility` — no transition clears VM_SEALED.
  - `safety_blocks_mutators` — munmap/mremap/mprotect-change/madvise-destructive on sealed VMA ⟹ EPERM.
  - `liveness_returns` — every mseal returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mseal` post: success ⟹ every byte in [start,end) is in a VM_SEALED vma | `Mseal::do_mseal` |
| `check_can_modify_vma` post: VM_SEALED ∧ mutator-op ⟹ EPERM | mutator paths |
| `vma_modify_seal` post: vma split so seal applies exactly to range | `Mm::vma_modify_seal` |

### Layer 4: Verus / Creusot functional

Per-`mseal(2)` man-page semantic equivalence with kselftest `mm/mseal_test.c`.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

`mseal(2)` reinforcement:

- **Per-VMA irreversible seal** — defense against per-attacker-driven mprotect/munmap-replace W^X downgrade.
- **Per-all-mutator hooks** — defense against per-bypass via mremap, MAP_FIXED-overwrite, or destructive madvise.
- **Per-atomic seal** — defense against per-half-sealed range exploited via gap.
- **Per-flags=0 reserved** — defense against per-flag smuggling for future seal types.
- **Per-fork inheritance** — defense against per-child-mutability bypass.
- **Per-execve reset (only)** — defense against per-stale-seal trap (intentional: new mm is clean).

## Grsecurity / PaX surface

- **mseal as PAX_MPROTECT-equivalent (per-VMA, irreversible W^X)** — where global `PAX_MPROTECT` enforces strict W^X for the entire process by forbidding W+X transitions, `mseal` provides the same guarantee per-VMA and post-hoc; a process that does not run under PAX_MPROTECT can still seal its finalized code segments to achieve the same outcome on those pages.
- **PAX_MPROTECT + mseal interaction** — under PAX_MPROTECT, mseal is redundant but not harmful; the two are designed to compose. PAX_MPROTECT denies the W+X transition; mseal denies *any* protection or layout change.
- **GRKERNSEC_RWXMAPS_NOSEAL_BYPASS** — even with PAX_MPROTECT enabled, mseal additionally blocks the `mmap(MAP_FIXED) over RX` overwrite trick; grsec ensures the seal check runs before any VMA replacement.
- **PAX_REFCOUNT on vma flags read-write** — defense against per-VM_SEALED bit-flip via VMA UAF.
- **GRKERNSEC_HIDESYM** — VM_SEALED visible in /proc/self/smaps but never via kernel-pointer leakage.
- **PaX UDEREF on mseal argument validation** — defense against per-arg kernel-deref bug; SMAP enforced.
- **GRKERNSEC_LOCK_MAPS** — grsec policy: certain process types (sshd, gpg-agent) automatically mseal their text and rodata at exec.
- **Per-vDSO automatic mseal** — vDSO sealed automatically at process startup under grsec; defense against per-vDSO-overwrite.
- **PAX_KERNEXEC alignment** — sealed userspace pages do not affect kernelexec, but grsec policy auto-seals signal trampolines and PLT stubs in glibc-hardened mode.
- **GRKERNSEC_HARDEN_PTRACE** — ptrace cannot use PTRACE_POKEDATA to overwrite sealed text (already enforced by VMA seal at vm_write check).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Future seal subtypes (e.g. selective `MSEAL_NO_UNMAP_ONLY`) (deferred).
- `mprotect(2)`, `mremap(2)`, `mmap(2)`, `munmap(2)`, `madvise(2)` per-syscall docs (separate Tier-5 docs).
- Implementation code.
