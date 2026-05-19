# Tier-5 syscall: shmat(2) — syscall 30

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - ipc/shm.c (SYSCALL_DEFINE3(shmat), do_shmat, ksys_shmat)
  - ipc/shm.c (shm_obtain_object_check, shp->shm_perm permission gate)
  - mm/mmap.c (vm_mmap_pgoff fallback, get_unmapped_area)
  - include/uapi/linux/shm.h (SHM_RDONLY, SHM_RND, SHM_REMAP, SHM_EXEC, SHMLBA)
  - include/linux/shm.h (struct shmid_kernel)
  - arch/x86/entry/syscalls/syscall_64.tbl (30  common  shmat)
-->

## Summary

`shmat(2)` attaches the System V shared-memory segment identified by `shmid` into the calling process's address space, returning the kernel-chosen (or caller-suggested) attachment address. The caller may request a specific address with `shmaddr`, may opt into read-only attachment, may request a coarse SHMLBA round-down of the address, may forcibly replace an existing mapping, and (legacy) may request executable mapping. The kernel allocates a VMA backed by the shmem file underlying the segment, increments the segment's attach count, records the per-process `shmid_kernel` reference, and returns the userspace address.

Critical for: SysV-IPC consumers (PostgreSQL shared buffers, X11 MIT-SHM, legacy databases, scientific HPC ranks, glibc test harnesses), container shared-mem isolation, namespace-scoped IPC keys, and the `ipcs(1)` / `ipcrm(1)` accounting fabric.

## Signature

```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `shmid` | `int` | in | Segment id returned by a prior `shmget(2)`. |
| `shmaddr` | `const void *` | in | Suggested attach address; `NULL` lets the kernel choose. |
| `shmflg` | `int` | in | Bitmask of `SHM_RDONLY`, `SHM_RND`, `SHM_REMAP`, `SHM_EXEC`. |

## Return value

| Value | Meaning |
|---|---|
| `void *` (non-`(void *)-1`) | Userspace address of the new attachment. |
| `(void *)-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | Calling process lacks attach permission per `shm_perm` (or `SHM_RDONLY` requested without read perm). |
| `EINVAL` | Invalid `shmid`, unaligned `shmaddr` without `SHM_RND`, or `shmaddr` collides with existing VMA without `SHM_REMAP`. |
| `ENOMEM` | Out of memory for VMA/page tables, or `RLIMIT_AS` would be exceeded. |
| `EIDRM` | Segment was marked `IPC_RMID` and is being destroyed. |
| `EPERM` | `SHM_EXEC` requested but mount options or LSM/grsec policy forbid executable shm. |
| `EMFILE` | Per-process attach-count cap reached (rare, `SHMSEG`). |
| `EFAULT` | `shmaddr` is a kernel-side address (validated via `access_ok`). |
| `EOVERFLOW` | `shmaddr + segsz` would wrap or escape `TASK_SIZE`. |

## ABI surface

```text
__NR_shmat (x86_64)   = 30
__NR_shmat (i386)     = 397 (post 5.1 direct syscall; pre-5.1 via ipc(SHMAT, ...))
__NR_shmat (arm64)    = 196
__NR_shmat (generic)  = 196

#define SHM_RDONLY  010000   /* attach read-only */
#define SHM_RND     020000   /* round shmaddr down to multiple of SHMLBA */
#define SHM_REMAP   040000   /* take over existing mapping */
#define SHM_EXEC   0100000   /* allow PROT_EXEC on the segment */
#define SHMLBA      PAGE_SIZE  /* arch-dependent; some MIPS use 4 * PAGE_SIZE */
```

## Compatibility contract

REQ-1: Syscall number is **30** on x86_64; **196** on arm64 / generic; on i386 the historical multiplexer `ipc(SHMAT, ...)` remains for ancient libc, with the direct syscall `__NR_shmat = 397` exposed since 5.1.

REQ-2: `shmid` must reference a live segment in the caller's IPC namespace. Cross-namespace ids return `EINVAL`.

REQ-3: Permission check uses `ipcperms(shp->shm_perm, mode)` where `mode = S_IRUGO` (or `S_IRUGO|S_IWUGO` without `SHM_RDONLY`). Failure: `EACCES`.

REQ-4: `shmaddr == NULL`: kernel chooses any address via `get_unmapped_area` honoring `MAP_HINT` rules; returns page-aligned address.

REQ-5: `shmaddr != NULL` without `SHM_RND`: must be page-aligned, else `EINVAL`. With `SHM_RND`: kernel rounds down to multiple of `SHMLBA`.

REQ-6: `shmaddr != NULL` with overlap and without `SHM_REMAP`: `EINVAL`. With `SHM_REMAP`: kernel calls `do_munmap` over the overlap before mapping.

REQ-7: `SHM_RDONLY`: VMA installed with `PROT_READ` only; writes fault `SIGSEGV`.

REQ-8: `SHM_EXEC`: VMA gains `PROT_EXEC`. Disallowed if shmfs is mounted `noexec`, if MNT_NOEXEC propagates, if `seccomp` filter declines, or if PaX/NX policy forbids — see Grsecurity section.

REQ-9: Attach increments `shp->shm_nattch`; `shm_atime` updated to wall clock; `shm_lpid` set to `current->tgid`.

REQ-10: Attached VMA has `vm_file = shp->shm_file` (a tmpfs inode), `vm_ops = &shm_vm_ops`. Closing the VMA decrements `shm_nattch` via `shm_close`.

REQ-11: If the segment was `IPC_RMID`'d (`shp->shm_perm.mode & SHM_DEST`) and is still mapped elsewhere, attach is **not** permitted: `EIDRM`. Once `shm_nattch == 0` the segment is freed.

REQ-12: `RLIMIT_AS`, `RLIMIT_MEMLOCK` (if `SHM_LOCKED`), and per-mm `total_vm` are accounted; failure yields `ENOMEM`.

REQ-13: `shmaddr` is validated by `access_ok(shmaddr, size)`; kernel-range addresses give `EFAULT`.

REQ-14: `mmap_write_lock(current->mm)` is held across address-space mutation; the IPC ids_rwsem is taken for read on segment lookup.

REQ-15: LSM hook `security_shm_shmat(shp, shmaddr, shmflg)` is invoked; refusal yields `EACCES`.

## Acceptance Criteria

- [ ] AC-1: `shmat(id, NULL, 0)` returns a non-`(void *)-1` page-aligned address.
- [ ] AC-2: Writing to the address propagates to all other attachers (single mm view).
- [ ] AC-3: `shmat(id, NULL, SHM_RDONLY)` then write: `SIGSEGV`.
- [ ] AC-4: `shmat(id, misaligned, 0)`: `EINVAL`.
- [ ] AC-5: `shmat(id, misaligned, SHM_RND)`: success, returns floor-aligned addr.
- [ ] AC-6: `shmat(id, addr, 0)` when `addr` overlaps existing VMA: `EINVAL`.
- [ ] AC-7: `shmat(id, addr, SHM_REMAP)` over existing VMA: success, prior mapping unmapped.
- [ ] AC-8: `shmat(-1, NULL, 0)`: `EINVAL`.
- [ ] AC-9: Caller without read permission: `EACCES`.
- [ ] AC-10: After `shmctl(IPC_RMID)` with no other attachers, new `shmat` returns `EIDRM`.
- [ ] AC-11: `SHM_EXEC` honored only when policy permits; otherwise `EPERM`.
- [ ] AC-12: `shm_nattch` increments by 1 per successful call; `ipcs -m` reflects this.
- [ ] AC-13: Attached VMA visible in `/proc/self/maps` with `SYSV` tag.

## Architecture

```rust
#[syscall(nr = 30, abi = "sysv")]
pub fn sys_shmat(shmid: i32, shmaddr: UserPtr<u8>, shmflg: i32) -> isize {
    Shm::do_shmat(shmid, shmaddr, shmflg)
}
```

`Shm::do_shmat(shmid, shmaddr, shmflg) -> isize`:
1. let ns = current_ipc_ns();
2. let shp = Shm::obtain_object_check(ns, shmid).map_err(|_| EINVAL)?;     // ids_rwsem read
3. /* Permission + LSM */
4. let want = if shmflg & SHM_RDONLY != 0 { S_IRUGO } else { S_IRUGO | S_IWUGO };
5. Shm::ipcperms(&shp.shm_perm, want)?;                                    // EACCES
6. security_shm_shmat(&shp, shmaddr, shmflg)?;                             // EACCES
7. /* Lifecycle: destroyed-but-still-pinned segments reject new attach */
8. if shp.shm_perm.mode & SHM_DEST != 0 { return Err(EIDRM); }
9. /* SHM_EXEC policy */
10. let mut prot = if shmflg & SHM_RDONLY != 0 { PROT_READ } else { PROT_READ | PROT_WRITE };
11. if shmflg & SHM_EXEC != 0 {
12.   Shm::check_exec_policy(&shp)?;                                        // EPERM
13.   prot |= PROT_EXEC;
14. }
15. /* Address validation */
16. let addr = Shm::resolve_addr(shmaddr, shmflg, shp.shm_segsz)?;          // EINVAL/EFAULT
17. /* Map */
18. let mut flags = MAP_SHARED;
19. if shmflg & SHM_REMAP != 0 { flags |= MAP_FIXED; }
20. let user = vm_mmap(&shp.shm_file, addr, shp.shm_segsz, prot, flags, 0)?;// ENOMEM/EINVAL
21. /* Accounting */
22. shp.shm_nattch.fetch_add(1, Ordering::SeqCst);
23. shp.shm_atime = ktime_get_real_seconds();
24. shp.shm_lpid = current_tgid();
25. user as isize

`Shm::resolve_addr(shmaddr, shmflg, segsz) -> Result<UserAddr>`:
1. if shmaddr.is_null() { return Ok(0); }                       // kernel picks
2. let addr = shmaddr.addr();
3. if !access_ok(addr, segsz) { return Err(EFAULT); }
4. if (addr & (SHMLBA - 1)) != 0 {
5.   if shmflg & SHM_RND == 0 { return Err(EINVAL); }
6.   return Ok(addr & !(SHMLBA - 1));
7. }
8. Ok(addr)

`Shm::check_exec_policy(shp) -> Result<()>`:
1. if shp.shm_file.f_path.mnt.mnt_flags & MNT_NOEXEC != 0 { return Err(EPERM); }
2. if pax_noexec_policy_forbids_shm_exec() { return Err(EPERM); }    // see Grsec
3. Ok(())
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `addr_validated_or_chosen` | INVARIANT | per-shmat: returned addr is page-aligned and inside TASK_SIZE. |
| `nattch_balanced` | INVARIANT | per-shmat success ⟹ shm_nattch incremented exactly once. |
| `perm_before_map` | INVARIANT | per-shmat: ipcperms + LSM run before vm_mmap. |
| `dest_segment_rejected` | INVARIANT | per-SHM_DEST: attach returns EIDRM, no VMA created. |
| `exec_policy_pre_prot` | INVARIANT | per-SHM_EXEC: policy check precedes PROT_EXEC install. |
| `overlap_requires_remap` | INVARIANT | per-overlap: SHM_REMAP required else EINVAL, no munmap collateral. |

### Layer 2: TLA+

`ipc/shmat.tla`:
- States: per-segment attach-set, shm_perm.mode flags, per-mm VMA table.
- Properties:
  - `safety_no_attach_after_dest` — `SHM_DEST` set ⟹ no new attaches.
  - `safety_nattch_matches_vmas` — system-wide invariant: `shm_nattch == |{vma : vma.shp == shp}|`.
  - `safety_perm_pre_map` — every successful attach ran perm + LSM checks first.
  - `safety_addr_in_user_range` — returned addr ∈ [0, TASK_SIZE).
  - `liveness_attach_terminates` — every shmat returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_shmat` post: success ⟹ shp.shm_nattch == old + 1 | `Shm::do_shmat` |
| `resolve_addr` post: result page-aligned ∨ result == 0 | `Shm::resolve_addr` |
| `check_exec_policy` post: success ⟹ shm_file mnt not noexec ∧ pax policy allows | `Shm::check_exec_policy` |
| `obtain_object_check` post: shp.id == shmid ∧ shp.ns == current_ns | `Shm::obtain_object_check` |

### Layer 4: Verus / Creusot functional

Per-`shmat(2)` man-page equivalence; LTP `shmat01..03` pass; PostgreSQL initdb start-of-cluster shared-buffer attach exercise.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`shmat(2)` reinforcement:

- **Per-permission + LSM gate before mapping** — defense against per-bypass mapping of unreadable segments.
- **Per-page-alignment enforced** — defense against per-misaligned-shmaddr mapping smuggling.
- **Per-SHM_DEST attach rejection** — defense against per-resurrect-dying-segment UAF.
- **Per-overlap requires SHM_REMAP** — defense against per-silent-overwrite of existing VMA.
- **Per-RLIMIT_AS accounting** — defense against per-VA exhaustion DoS.
- **Per-`access_ok(shmaddr)`** — defense against per-kernel-range shmaddr smuggle.
- **Per-`mmap_write_lock` for mapping mutation** — defense against per-races with concurrent fork/mmap.

## Grsecurity / PaX surface

- **PAX_NOEXEC default rejection of `SHM_EXEC`** — when PaX `NOEXEC` is active (mprotect / segmexec / pageexec), `SHM_EXEC` returns `EPERM` unconditionally; userspace cannot install executable SysV-shm regardless of mount options. RWX shm is the classic exploit gadget pivot, so it is forbidden.
- **PaX UDEREF on `shmaddr` user pointer** — `access_ok` is forced through the SMAP/UDEREF path; a kernel-range `shmaddr` cannot reach `vm_mmap` even if `access_ok` is buggy on a given arch.
- **GRKERNSEC_CHROOT_SHMAT** — when set, processes inside a grsec chroot cannot `shmat` segments created outside the chroot. Cross-jail IPC is a frequent escape primitive; this option closes it.
- **GRKERNSEC_SYSV_IPC isolation** — SysV-IPC objects gain a per-user / per-namespace owner field. `shmat` of a segment owned by a different uid (without IPC bypass) fails `EACCES` even if the legacy `shm_perm` mode would have allowed it.
- **PAX_RANDMMAP for chosen `shmaddr == NULL` path** — when the kernel picks the attach address, PaX `RANDMMAP` randomizes it inside the per-task RNG-seeded VA window. Defeats fixed-offset shm-spray ROP primitives.
- **GRKERNSEC_RESLOG** — every `shmat` that exceeds `RLIMIT_AS` is logged with task / uid / cap context.
- **PAX_REFCOUNT on `shm_nattch`** — atomic refcount uses saturating PAX_REFCOUNT; overflow panics rather than wrapping, defeating per-attach-count overflow UAF.
- **Per-`SHM_REMAP` audit** — `SHM_REMAP` issues an audit record when used inside a chroot or with cap-bounded creds, since silent VMA replacement is rarely benign.
- **Per-LSM SELinux `shm { attach }` permission** — orthogonal to grsec but stacked; both must allow.
- **No_new_privs neutral but observed** — `shmat` does not change creds; NNP unaffected.
- **PAX_USERCOPY_HARDEN on `shmaddr` validation** — the validation copy uses whitelisted slabs; out-of-slab `shmaddr` parsing rejected before `vm_mmap` call.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `shmget(2)` (Tier-5 separate doc — segment creation).
- `shmdt(2)` (Tier-5 separate doc — detachment).
- `shmctl(2)` (Tier-5 separate doc — control / IPC_RMID).
- tmpfs shmem backing store (Tier-3 in `mm/shmem.md`).
- IPC namespace lifecycle (Tier-3 in `kernel/ipc-ns.md`).
- Implementation code.
