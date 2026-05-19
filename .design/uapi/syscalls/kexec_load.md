# Tier-5 syscall: kexec_load(2) — syscall 246

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/kexec.c (SYSCALL_DEFINE4(kexec_load), do_kexec_load)
  - kernel/kexec_core.c (kimage_alloc_init, machine_kexec)
  - include/uapi/linux/kexec.h (KEXEC_ARCH_*, KEXEC_ON_CRASH, KEXEC_PRESERVE_CONTEXT)
  - arch/x86/entry/syscalls/syscall_64.tbl (246  common  kexec_load)
-->

## Summary

`kexec_load(2)` stages a new kernel image to be booted by a subsequent `reboot(LINUX_REBOOT_CMD_KEXEC)` call. The caller supplies `entry`, a `nr_segments`-element array of `(buf, bufsz, mem, memsz)` segments describing the layout of the new kernel image and any initrd/devicetree blobs, and a `flags` word selecting the target architecture and modifier bits (`KEXEC_ON_CRASH`, `KEXEC_PRESERVE_CONTEXT`).

The kernel copies the segment table, validates non-overlap and bounds, allocates a `struct kimage`, then for each segment copies the user-side bytes into a chain of kernel pages chosen so that the boot-time copier can place each segment at its target physical address without conflicting with the running kernel.

`kexec_load(2)` is the legacy, in-kernel-trusted path. `kexec_file_load(2)` (syscall 320) is the modern signed-image path. Critical for: panic-crash kdump, fast reboot, hypervisor live-update.

## Signature

```c
long kexec_load(unsigned long entry, unsigned long nr_segments,
                struct kexec_segment *segments, unsigned long flags);
```

```c
struct kexec_segment {
    const void *buf;        /* userspace source */
    size_t bufsz;           /* source bytes */
    const void *mem;        /* target physical addr (in new kernel) */
    size_t memsz;           /* target span (zero-padded if > bufsz) */
};

#define KEXEC_ON_CRASH         0x00000001
#define KEXEC_PRESERVE_CONTEXT 0x00000002
#define KEXEC_ARCH_MASK        0xffff0000
#define KEXEC_ARCH_DEFAULT     0
#define KEXEC_ARCH_X86_64      (62 << 16)
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `entry` | `unsigned long` | in | Physical entry point of the new kernel image. |
| `nr_segments` | `unsigned long` | in | Number of segments (cap `KEXEC_SEGMENT_MAX = 16`). |
| `segments` | `struct kexec_segment *` | in | User pointer to segment array. |
| `flags` | `unsigned long` | in | `KEXEC_ARCH_*` in high half, modifier bits in low half. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Image staged; pending `reboot(LINUX_REBOOT_CMD_KEXEC)`. |
| `-1` + `errno` | Staging failed; previous image (if any) unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_BOOT`; `kernel_lockdown(integrity)` blocking unsigned image. |
| `EBUSY` | Image is already staged on this slot (crash vs normal); call with `nr_segments=0` to clear. |
| `EINVAL` | `nr_segments > KEXEC_SEGMENT_MAX`; segments overlap; `KEXEC_ARCH` invalid; `flags` reserved bit set; `KEXEC_PRESERVE_CONTEXT` without crash. |
| `EFAULT` | `segments` or any `buf` user pointer faults. |
| `ENOMEM` | Out of memory allocating `kimage` / segment pages. |
| `ENOEXEC` | Entry point not within any segment's `mem..mem+memsz` range. |

## ABI surface

```text
__NR_kexec_load        (x86_64)  = 246
__NR_kexec_load        (arm64)   = 104
__NR_kexec_load        (riscv)   = 104
__NR_kexec_load        (i386)    = 283

/* KEXEC_SEGMENT_MAX = 16. */
/* Two slots: normal kexec (one image) and crash kexec (one image)
   distinguished by KEXEC_ON_CRASH. */
/* Clearing a slot: kexec_load(0, 0, NULL, flags-with-matching-CRASH-bit). */
```

## Compatibility contract

REQ-1: Syscall number is **246** on x86_64. ABI-stable.

REQ-2: Caller MUST have `CAP_SYS_BOOT` in init_user_ns. Capability check precedes any user copy.

REQ-3: `nr_segments > KEXEC_SEGMENT_MAX` → `-EINVAL`.

REQ-4: `flags`:
- `KEXEC_ARCH_MASK` selects the destination arch (must match running kernel or be `KEXEC_ARCH_DEFAULT`).
- `KEXEC_ON_CRASH`: stage into crash slot, used after `panic()`.
- `KEXEC_PRESERVE_CONTEXT`: keep current CPU/RAM state restorable (hibernate-style); requires `KEXEC_ON_CRASH` or arch support.
- Other low-bit values reserved → `-EINVAL`.

REQ-5: `nr_segments == 0` clears the slot (frees any previously staged kimage); no segment copy.

REQ-6: `copy_from_user` for the segment table; bounded by `nr_segments * sizeof(struct kexec_segment)`.

REQ-7: Per-segment validation:
- `bufsz <= memsz` (memsz may pad with zeros).
- `mem` and `mem + memsz` are below `PHYS_ADDR_MAX` and outside reserved/IO regions.
- `bufsz < KEXEC_SEGMENT_MAX_SIZE` (per-segment cap).
- Segments do not overlap with each other.
- Segments do not overlap with the running kernel's image (except in the crash-kexec reserved region).

REQ-8: Entry point: `entry` must lie within some segment's `[mem, mem + memsz)` range; else `-ENOEXEC`.

REQ-9: kimage allocation:
- a) `struct kimage` allocated.
- b) For each segment, kernel allocates pages from the "destination pool" (a set of pages that will be safe to copy from after the running kernel is no longer scheduled).
- c) User bytes copied into those pages.
- d) Boot-time copier (`machine_kexec_prepare`, then `relocate_kernel`) records source-page → dest-phys mapping.

REQ-10: Per-slot exclusivity: only one kimage may be staged per slot (normal / crash). Re-staging without clearing → `-EBUSY`.

REQ-11: `kernel_lockdown(integrity)`: `kexec_load(2)` is unconditionally denied; only `kexec_file_load(2)` (which verifies signatures) is permitted.

REQ-12: `CONFIG_KEXEC=n`: syscall returns `-ENOSYS`.

REQ-13: Crash-kexec target memory region (`crashkernel=...` boot param) must be reserved at boot; if absent, `KEXEC_ON_CRASH` → `-EINVAL`.

REQ-14: Successful staging leaves the system unmodified; the new image is only entered on `reboot(LINUX_REBOOT_CMD_KEXEC)` (normal) or on `panic()` (crash).

REQ-15: Per-arch (`machine_kexec_prepare`) may impose additional constraints (e.g., relocation alignment, EFI memory map preservation).

## Acceptance Criteria

- [ ] AC-1: Caller without `CAP_SYS_BOOT`: `-EPERM`.
- [ ] AC-2: `nr_segments = 0, flags = 0`: clears normal slot, returns 0.
- [ ] AC-3: `nr_segments = 17`: `-EINVAL`.
- [ ] AC-4: Two segments overlapping in `mem`: `-EINVAL`.
- [ ] AC-5: `entry` outside all segments: `-ENOEXEC`.
- [ ] AC-6: Valid staging then re-stage without clearing: `-EBUSY`.
- [ ] AC-7: `flags = KEXEC_ARCH_X86_64 | KEXEC_ON_CRASH` on system without `crashkernel=`: `-EINVAL`.
- [ ] AC-8: `flags` reserved low bit set: `-EINVAL`.
- [ ] AC-9: `kernel_lockdown(integrity)`: `-EPERM`.
- [ ] AC-10: `segments == NULL` with `nr_segments > 0`: `-EFAULT`.
- [ ] AC-11: After successful staging: `reboot(LINUX_REBOOT_CMD_KEXEC)` boots new image.

## Architecture

```rust
#[syscall(nr = 246, abi = "sysv")]
pub fn sys_kexec_load(entry: usize, nr: usize, segs: UserPtr<KexecSegment>, flags: usize) -> isize {
    Kexec::do_kexec_load(entry, nr, segs, flags)
}
```

`Kexec::do_kexec_load(entry, nr, segs_ptr, flags) -> isize`:
1. Kexec::check_cap_sys_boot(&current_creds())?;            // EPERM
2. if kernel_lockdown_active(INTEGRITY) { return -EPERM; }
3. if nr > KEXEC_SEGMENT_MAX { return -EINVAL; }
4. if (flags & KEXEC_FLAGS_RESERVED) != 0 { return -EINVAL; }
5. let arch = (flags & KEXEC_ARCH_MASK) >> 16;
6. if !Kexec::valid_arch(arch) { return -EINVAL; }
7. let crash = (flags & KEXEC_ON_CRASH) != 0;
8. if crash && !crashkernel_reserved() { return -EINVAL; }
9. let slot = if crash { Slot::Crash } else { Slot::Normal };
10. let mutex = KEXEC_MUTEX.lock();
11. if nr == 0 {
12.   Kexec::clear_slot(slot);
13.   return 0;
14. }
15. if Kexec::slot_occupied(slot) { return -EBUSY; }
16. let segs = Kexec::copy_segments_from_user(segs_ptr, nr)?;
17. Kexec::validate_segments(&segs, entry, crash)?;          // EINVAL / ENOEXEC
18. let kimage = Kexec::alloc_kimage(entry, &segs, flags)?;  // ENOMEM
19. for seg in &segs {
20.   Kexec::copy_segment_bytes(&kimage, seg)?;              // EFAULT
21. }
22. Kexec::machine_prepare(&kimage)?;                        // arch validation
23. Kexec::install_slot(slot, kimage);
24. 0

`Kexec::validate_segments(segs, entry, crash) -> Result<()>`:
1. let mut covered = false;
2. for (i, s) in segs.iter().enumerate() {
3.   if s.bufsz > s.memsz { return Err(EINVAL); }
4.   if s.memsz > KEXEC_SEGMENT_MAX_SIZE { return Err(EINVAL); }
5.   if s.mem.wrapping_add(s.memsz) > PHYS_ADDR_MAX { return Err(EINVAL); }
6.   for s2 in &segs[i+1..] {
7.     if Kexec::ranges_overlap(s, s2) { return Err(EINVAL); }
8.   }
9.   if entry >= s.mem && entry < s.mem + s.memsz { covered = true; }
10. }
11. if !covered { return Err(ENOEXEC); }
12. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM returned before any copy. |
| `nr_segments_bounded` | INVARIANT | nr_segments ≤ KEXEC_SEGMENT_MAX. |
| `segments_disjoint` | INVARIANT | no two segments overlap in [mem, mem+memsz). |
| `entry_in_segment` | INVARIANT | entry ∈ some segment range. |
| `slot_exclusive` | INVARIANT | per-slot occupancy: at most one staged kimage. |
| `lockdown_blocks` | INVARIANT | kernel_lockdown(integrity) ⟹ EPERM. |

### Layer 2: TLA+

`kernel/kexec-load.tla`:
- States: per-cap, per-flags-validate, per-slot-check, per-copy-segs, per-validate, per-alloc, per-copy-bytes, per-install.
- Properties:
  - `safety_slot_per_kind_exclusive`.
  - `safety_lockdown_blocks_unsigned`.
  - `safety_segment_disjoint`.
  - `safety_entry_covered`.
  - `liveness_load_terminates_or_returns_error`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_kexec_load` post: ret 0 ⟹ slot occupied with valid kimage | `Kexec::do_kexec_load` |
| `validate_segments` post: all pairwise disjoint, entry covered | `Kexec::validate_segments` |
| `alloc_kimage` post: kimage.pages.len() ≥ ceil(sum memsz / PAGE_SIZE) | `Kexec::alloc_kimage` |
| `clear_slot` post: slot empty, prior pages freed | `Kexec::clear_slot` |

### Layer 4: Verus / Creusot functional

Per-`kexec_load(2)` man-page and per-`kernel/kexec.c` semantic equivalence. Kexec-tools selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`kexec_load(2)` reinforcement:

- **Per-cap-check pre-copy** — defense against per-priv-escalation via segment smuggling.
- **Per-nr-segments cap** — defense against per-OOM via giant table.
- **Per-segment overlap reject** — defense against per-runtime-text corruption.
- **Per-entry-in-segment** — defense against per-unbounded jump.
- **Per-slot exclusive** — defense against per-double-stage UAF.
- **Per-lockdown integrity gate** — defense against per-unsigned image staging.

## Grsecurity / PaX surface

- **PaX UDEREF on `segments` and per-segment `buf` copy_from_user** — defense against per-segment-pointer kernel-deref bug; SMAP forced for every per-segment copy_from_user.
- **CAP_SYS_BOOT strict in init_user_ns only** — child userns root denied. GRKERNSEC_CHROOT_CAPS extends: chrooted task may never stage kexec image.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` or `=confidentiality`, `kexec_load(2)` returns `-EPERM` unconditionally. Only `kexec_file_load(2)` (which validates appended-signature on the kernel image fd) is permitted.
- **GRKERNSEC_KEXEC_DISABLE** — under hardened mode, `kexec_load(2)` is disabled even if lockdown is off, leaving only `kexec_file_load(2)` for legitimate kdump configuration. Returns `-EPERM`.
- **PAX_USERCOPY_HARDEN on segment data copy** — bounded copy via whitelisted bounce buffer; no slab leak even from adversarial segment payloads.
- **PaX KERNEXEC on destination pages** — staged segment pages mapped RW kernel-side; only become executable through the relocate_kernel trampoline.
- **GRKERNSEC_HIDESYM on `/proc/iomem` (crashkernel region)** — defense against per-region-leak.
- **PAX_REFCOUNT on kimage refcount** — defense against per-refcount-overflow UAF on stage/clear race.
- **GRKERNSEC_AUDIT_KEXEC** — every kexec_load logged with PID, comm, nr_segments, flags, return code; staged image hash recorded.
- **GRKERNSEC_KMOD restrictions correlation** — if module loading is disabled (`modules_disabled=1`), kexec_load is also locked out as a chained restriction (defense against bypass-via-kexec to load arbitrary kernel).
- **GRKERNSEC_PERF_HARDEN correlation** — kexec staging logged under perf-audit; staging during high-rate perf is flagged.
- **CAP_SYS_RAWIO not implied** — kexec_load requires `CAP_SYS_BOOT`; calling without it returns `-EPERM` even if `CAP_SYS_RAWIO` is present.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-segment-validate stack overflow.
- **Crash-kexec slot integrity** — under hardened mode, crash kimage is verified at panic() time against staged hash; mismatch → fall back to plain panic.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Arch-specific `machine_kexec_prepare` (covered in arch Tier-3 docs).
- Crash-kdump (covered in Tier-3 `kernel/crash_core.md`).
- `kexec_file_load(2)` signature checking (separate doc).
- Implementation code.
