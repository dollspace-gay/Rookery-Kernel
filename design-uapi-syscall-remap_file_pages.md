---
title: "Tier-5 syscall: remap_file_pages(2) — syscall 216"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`remap_file_pages(2)` was introduced to create a non-linear mapping — a single VMA in which the pages map disjoint file offsets — without paying the cost of one VMA per offset. Modern kernels (since 3.16) have removed the in-tree non-linear infrastructure and emulate the syscall as a sequence of normal `mmap()` calls inside the original VMA, paying the cost the call was designed to avoid. The syscall is therefore deprecated and present only for ABI compatibility with the small number of legacy users (early Oracle DB versions, a few HPC frameworks).

The contract today is: validate the arguments, perform an emulated remap by issuing `mmap(addr + pgoff*PAGE_SIZE, PAGE_SIZE, ..., MAP_FIXED, file, pgoff*PAGE_SIZE)` for every page in the range under `mmap_write_lock`, and return success. The kernel logs a one-shot dmesg warning advising callers to migrate to plain `mmap()`. Critical for: legacy ABI continuity; new code MUST NOT call this syscall.

### Acceptance Criteria

- [ ] AC-1: Legacy caller: `remap_file_pages(addr, PAGE_SIZE, 0, 7, 0)` on a `MAP_SHARED` file VMA succeeds; reading `addr` returns the file contents at offset `7 * PAGE_SIZE`.
- [ ] AC-2: `prot = PROT_READ` returns `-EINVAL`.
- [ ] AC-3: `flags = MAP_NONBLOCK` accepted (no-op).
- [ ] AC-4: Anonymous mapping: `-EINVAL`.
- [ ] AC-5: Private mapping (`MAP_PRIVATE`): `-EINVAL`.
- [ ] AC-6: Range crossing multiple VMAs: `-EINVAL`.
- [ ] AC-7: First call from a task: dmesg has `deprecated remap_file_pages() syscall`.
- [ ] AC-8: `sysctl vm.remap_file_pages_disabled = 1`: returns `-EPERM` without further checks.
- [ ] AC-9: Range overflow `addr + size > TASK_SIZE`: `-EINVAL`.
- [ ] AC-10: Concurrent `mmap` on overlapping range serialized via `mmap_write_lock`.

### Architecture

```rust
#[syscall(nr = 216, abi = "sysv")]
pub fn sys_remap_file_pages(
    addr: UserAddr,
    size: usize,
    prot: i32,
    pgoff: usize,
    flags: i32,
) -> isize {
    Mmap::do_remap_file_pages(addr, size, prot, pgoff, flags)
}
```

`Mmap::do_remap_file_pages(addr, size, prot, pgoff, flags) -> isize`:
1. if sysctl_remap_file_pages_disabled { return -EPERM; }
2. if addr & (PAGE_SIZE - 1) != 0 { return -EINVAL; }
3. if size & (PAGE_SIZE - 1) != 0 { return -EINVAL; }
4. if size == 0 { return -EINVAL; }
5. let end = addr.checked_add(size).ok_or(EINVAL)?;
6. if end > TASK_SIZE { return -EINVAL; }
7. if prot != 0 { return -EINVAL; }
8. if (flags & !MAP_NONBLOCK) != 0 { return -EINVAL; }
9. /* One-shot deprecation warn */
10. pr_warn_once_per_task("uses deprecated remap_file_pages() syscall");
11. let mm = current().mm();
12. mm.mmap_write_lock_killable()?;
13. let res = {
14.    let Some(vma) = mm.find_vma(addr) else { return Err(-EINVAL); };
15.    if vma.start() != addr || vma.end() < end { return Err(-EINVAL); }
16.    if !vma.flags().contains(VM_SHARED) { return Err(-EINVAL); }
17.    let Some(file) = vma.file().clone() else { return Err(-EINVAL); };
18.    let prot_bits = vma.flags() & (VM_READ | VM_WRITE | VM_EXEC);
19.    let map_flags = MAP_FIXED | MAP_SHARED;
20.    let file_off = pgoff.checked_mul(PAGE_SIZE).ok_or(-EINVAL)?;
21.    let mut populate: usize = 0;
22.    do_mmap(&file, addr, size, prot_bits, map_flags, file_off, &mut populate)
23. };
24. mm.mmap_write_unlock();
25. match res {
26.    Ok(_) => 0,
27.    Err(e) => e,
28. }

### Out of Scope

- `mmap(2)` (covered in `mmap.md`).
- `mremap(2)` (covered in `mremap.md`).
- Historical nonlinear VMA code (removed upstream).
- Implementation code.

### signature

```c
int remap_file_pages(void *addr, size_t size, int prot,
                     size_t pgoff, int flags);
```

```c
/* `prot` is ignored on modern kernels; the original VMA's protection is
   used.  `flags` accepts MAP_NONBLOCK (no-op since 2.6) and MAP_FIXED
   semantics implicitly. */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `void *` | in | Start of an existing file-backed VMA; must equal the original `mmap` base. |
| `size` | `size_t` | in | Byte length of the range; must be page-aligned and within the VMA. |
| `prot` | `int` | in | Ignored (must be 0); the VMA's existing protection is reused. |
| `pgoff` | `size_t` | in | File offset (in pages) to remap the range to. |
| `flags` | `int` | in | Accepts `MAP_NONBLOCK` (0x10000); other bits MUST be 0. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success — range now maps file at `pgoff`. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `addr` not page-aligned; `addr + size` overflows; range does not lie inside a single VMA; VMA is not file-backed; VMA is not `MAP_SHARED`; `prot` is non-zero; `flags` has bits other than `MAP_NONBLOCK`. |
| `ENOMEM` | Out of memory during emulation `mmap` calls. |
| `EPERM` | (since 4.0) Issued when `sysctl vm.remap_file_pages_disabled = 1`, allowing administrators to neuter this attack surface entirely. |

### abi surface

```text
__NR_remap_file_pages  (x86_64)  = 216
__NR_remap_file_pages  (arm64)   = 234
__NR_remap_file_pages  (riscv)   = -ENOSYS (not present)
__NR_remap_file_pages  (i386)    = 257

/* Deprecated since 3.16; kernel emits a dmesg warning the first time the
   syscall is invoked by a given task. */
/* `prot` argument is preserved in the signature for ABI but ignored. */
```

### compatibility contract

REQ-1: Syscall number is **216** on x86_64. ABI-stable but DEPRECATED.

REQ-2: Argument validation (in this order):
- `addr & (PAGE_SIZE - 1) != 0` ⟹ `-EINVAL`.
- `size & (PAGE_SIZE - 1) != 0` ⟹ `-EINVAL`.
- `addr + size` overflows ⟹ `-EINVAL`.
- `prot != 0` ⟹ `-EINVAL`.
- `flags & ~MAP_NONBLOCK != 0` ⟹ `-EINVAL`.

REQ-3: VMA validation:
- Single VMA covering `[addr, addr + size)`.
- `vma->vm_file != NULL`.
- `vma->vm_flags & VM_SHARED` set.
- `vma->vm_start == addr` (must equal original mapping base).
- `vma->vm_end >= addr + size`.

Otherwise `-EINVAL`.

REQ-4: Emulation: under `mmap_write_lock`, the kernel issues `do_mmap(vma->vm_file, addr, size, vma->vm_page_prot, MAP_FIXED | MAP_SHARED, pgoff * PAGE_SIZE, &populate)`. This replaces the original VMA with one (or, after VMA splits, several) that map the requested file offset.

REQ-5: One-shot warning: the first time a given task issues `remap_file_pages`, the kernel logs `pr_warn_once("%s (%d) uses deprecated remap_file_pages() syscall.", current->comm, current->pid)`. Subsequent calls from the same task are silent.

REQ-6: `sysctl vm.remap_file_pages_disabled`:
- 0: emulation enabled (default upstream).
- 1: syscall returns `-EPERM` immediately, without inspecting arguments.

REQ-7: No new code may use this syscall. Compilers/static analyzers should flag it as deprecated.

REQ-8: Forward-compat: the signature must remain wire-stable. New flags will NOT be added; alternative behavior would be exposed via `mmap(MAP_FIXED_NOREPLACE)` or `mremap(2)`.

REQ-9: The emulation does not preserve the original `vma->vm_pgoff` linkage that pre-3.16 non-linear VMAs had; tools that scrape `/proc/self/maps` see the original VMA replaced by a normal linear mapping at the new offset.

REQ-10: Per-VMA file refcount accounting goes through `do_mmap` as in any other mapping creation; refcount underflow on tear-down handled by the standard mmap path.

REQ-11: Audit subsystem records the call with `AUDIT_PERM_EXEC | AUDIT_PERM_READ` if the original mapping was readable/executable, mirroring `mmap` rules.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prot_zero_required` | INVARIANT | non-zero prot ⟹ EINVAL. |
| `flags_bits_validated` | INVARIANT | only MAP_NONBLOCK accepted. |
| `range_inside_single_vma` | INVARIANT | success ⟹ entire range covered by one shared file VMA. |
| `sysctl_disable_short_circuit` | INVARIANT | disabled ⟹ EPERM before arg checks. |
| `deprecation_warn_once_per_task` | INVARIANT | per-task warn-once. |
| `mmap_write_lock_balanced` | INVARIANT | locked iff dropped on every exit edge. |

### Layer 2: TLA+

`mm/remap-file-pages.tla`:
- States: per-sysctl-check, per-arg-validate, per-vma-validate, per-do-mmap-emul.
- Properties:
  - `safety_anon_rejected` — VMA without vm_file ⟹ EINVAL.
  - `safety_private_rejected` — VMA without VM_SHARED ⟹ EINVAL.
  - `safety_disabled_blocks` — sysctl=1 ⟹ EPERM regardless of args.
  - `liveness_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_remap_file_pages` post: returns EINVAL for misaligned addr/size | `Mmap::do_remap_file_pages` |
| `do_remap_file_pages` post: returns EPERM if disabled | `Mmap::do_remap_file_pages` |
| `do_remap_file_pages` post: success ⟹ one do_mmap call issued | `Mmap::do_remap_file_pages` |
| `pr_warn_once_per_task` post: at-most-once per task_struct | `Mmap::do_remap_file_pages` |

### Layer 4: Verus / Creusot functional

Per-`remap_file_pages(2)` man-page + glibc wrapper semantics; LTP `kernel/syscalls/remap_file_pages/` selftests pass on enabled-default kernels.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`remap_file_pages(2)` reinforcement:

- **Per-deprecation warn-once** — defense against per-log-flood.
- **Per-sysctl global disable** — defense against per-attack-surface exposure.
- **Per-prot-must-be-zero** — defense against per-privilege escalation via re-permissioning.
- **Per-VMA shared-file-backed only** — defense against per-anonymous-or-private confusion.
- **Per-bounded emulation cost** — defense against per-OOM via giant size argument.
- **Per-mmap_write_lock atomicity** — defense against per-race remap during fault.

### grsecurity / pax surface

- **PaX UDEREF on `addr` validation** — defense against per-user-addr kernel-deref bug.
- **GRKERNSEC_DEPRECATED_SYSCALL** — `remap_file_pages` is gated behind a Kconfig knob; if disabled, the syscall returns `-ENOSYS` and emits an `audit_log` record. Recommended setting: disabled in production.
- **remap_file_pages deprecation-audit** — every successful call emits `audit_log_syscall(AUDIT_REMAP)` with task, file, original offset, and new pgoff so SIEM can detect legacy callers.
- **GRKERNSEC_HARDEN_MMAP_SEMANTICS** — strict argument mask; `prot` must be zero, `flags` may only have `MAP_NONBLOCK`.
- **PaX MPROTECT interaction** — emulation `do_mmap` honors PAX_MPROTECT, so non-linear remap cannot relax W^X.
- **PAX_REFCOUNT on file refcount** — emulation path goes through `fput`/`fget`; saturating overflow defense.
- **GRKERNSEC_HARDEN_USERFAULTFD interaction** — refuse remap_file_pages on a uffd-registered VMA (would invalidate uffd state machine).
- **GRKERNSEC_TRUSTED_PATH constraints** — remapping a file from a non-trusted mount denied when policy forbids cross-mount mapping changes.
- **PaX KERNEXEC on do_mmap dispatch** — function-pointer trampolined.
- **GRKERNSEC_HIDESYM on dmesg** — pr_warn_once includes task name + PID but no kernel addresses.
- **Per-sysctl vm.remap_file_pages_disabled** — grsec ships with default 1 (disabled). Toggling it requires CAP_SYS_ADMIN and is itself audited.

