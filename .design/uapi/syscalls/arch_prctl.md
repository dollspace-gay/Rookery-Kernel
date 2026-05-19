# Tier-5 syscall: arch_prctl(2) — syscall 158

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/kernel/process_64.c (do_arch_prctl_64, do_arch_prctl_common)
  - arch/x86/include/uapi/asm/prctl.h (ARCH_SET_*, ARCH_GET_*, ARCH_SHSTK_*, ARCH_MAP_VDSO_*)
  - arch/x86/kernel/cet.c (shadow-stack ops)
  - arch/x86/entry/syscalls/syscall_64.tbl (158  64  arch_prctl)
-->

## Summary

`arch_prctl(2)` is the x86 / x86_64 arch-specific cousin of `prctl(2)`. It exposes operations that only make sense for x86: setting the user FS/GS segment base (used by glibc to anchor the TLS pointer), enabling/disabling the cpuid instruction trap, controlling Linear Address Masking (LAM, "tagged pointers" for x86_64), enabling/locking the CET shadow stack and IBT, and mapping the vDSO at a chosen address (used by CRIU). It is the canonical mechanism by which userland establishes its thread-local-storage pointer (`%fs:0`) and by which the runtime opts into CET hardening.

Critical for: every glibc/musl thread (TLS init via `ARCH_SET_FS`), CET-enabled binaries, x86 sandboxing (cpuid trap), LAM-using tagged-pointer runtimes, and CRIU checkpoint/restore (`ARCH_MAP_VDSO_*`).

This Tier-5 covers x86_64 entry; the i386 variant routes through `__NR_arch_prctl = 384` and supports a subset.

## Signature

```c
int arch_prctl(int code, unsigned long addr);
/* For ARCH_GET_*: addr is "unsigned long *" written through. */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `code` | `int` | in | One of `ARCH_*` opcodes (see ABI surface). |
| `addr` | `unsigned long` | in/out | Per-opcode: a value to set, or a user pointer to write to. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success (ARCH_SET_* and most ARCH_GET_* that take a user pointer). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`  | Unknown `code`; non-canonical address for `ARCH_SET_FS/GS`; reserved bits set in LAM/SHSTK arg. |
| `EFAULT`  | User pointer (for `ARCH_GET_*`) unwritable. |
| `EPERM`   | Lacking required cap (e.g. `ARCH_MAP_VDSO_*` requires `CAP_SYS_ADMIN`). |
| `ENODEV`  | Feature unsupported by CPU (e.g. LAM on a CPU without LAM, CET without CET). |
| `EBUSY`   | Shadow stack already locked; vDSO already mapped at a different location. |
| `EOPNOTSUPP` | Operation not implemented on this arch variant. |

## ABI surface

```text
__NR_arch_prctl (x86_64)  = 158
__NR_arch_prctl (i386)    = 384     /* x32 ABI: 0x40000000 | 158 */

ARCH_SET_GS              0x1001     /* addr = new GS base; must be canonical */
ARCH_SET_FS              0x1002     /* addr = new FS base; must be canonical */
ARCH_GET_FS              0x1003     /* addr = unsigned long*; receives current FS base */
ARCH_GET_GS              0x1004     /* addr = unsigned long*; receives current GS base */

ARCH_GET_CPUID           0x1011     /* returns 0 = trap enabled, 1 = trap disabled (cpuid permitted) */
ARCH_SET_CPUID           0x1012     /* addr = 0 disables cpuid (raises SIGSEGV); 1 enables */

ARCH_GET_XCOMP_SUPP      0x1021     /* addr = u64*; returns supported XCR0 mask */
ARCH_GET_XCOMP_PERM      0x1022     /* addr = u64*; returns current XCR0 permission */
ARCH_REQ_XCOMP_PERM      0x1023     /* addr = u32; requests permission to use feature N (AMX) */
ARCH_GET_XCOMP_GUEST_PERM 0x1024
ARCH_REQ_XCOMP_GUEST_PERM 0x1025

ARCH_MAP_VDSO_X32        0x2001     /* map x32 vDSO at addr (CRIU) */
ARCH_MAP_VDSO_32         0x2002     /* map i386 vDSO at addr */
ARCH_MAP_VDSO_64         0x2003     /* map x86_64 vDSO at addr */

ARCH_SHSTK_ENABLE        0x5001     /* enable shadow stack (CET) */
ARCH_SHSTK_DISABLE       0x5002     /* disable shadow stack */
ARCH_SHSTK_LOCK          0x5003     /* irreversibly lock current shstk policy */
ARCH_SHSTK_UNLOCK        0x5004     /* (debug-only; requires CAP_SYS_ADMIN) */
ARCH_SHSTK_STATUS        0x5005     /* addr = u64*; receives current shstk flags */
   /* features: ARCH_SHSTK_SHSTK | ARCH_SHSTK_WRSS */

ARCH_GET_UNTAG_MASK      0x4001     /* addr = u64*; receives LAM untag mask */
ARCH_ENABLE_TAGGED_ADDR  0x4002     /* addr = nr_bits (0, 48, 57); enable LAM_U48 / LAM_U57 */
ARCH_GET_MAX_TAG_BITS    0x4003     /* addr = u64*; receives max supported tag bits */
ARCH_FORCE_TAGGED_SVA    0x4004     /* enable tagged SVA (shared-VA pointers) */
```

### Limits

```text
canonical-FS / GS-base range:
  bit 63..47 must all be equal (USER_BASE_MAX = 0x7FFF_FFFF_FFFF for 48-bit, 0xFF_FFFF_FFFF_FFFF for 57-bit if LA57)
shadow-stack region: power-of-two pages; minimum 2 pages; maximum RLIMIT_STACK / 4
LAM_U48 untag mask: bits [62:48]
LAM_U57 untag mask: bits [62:57]
```

## Compatibility contract

REQ-1: Syscall number is **158** on x86_64. ABI-stable forever.

REQ-2: `ARCH_SET_FS`: writes `addr` into `task.thread.fsbase` and into the active `MSR_FS_BASE`. `addr` MUST be canonical (sign-extended); otherwise `-EINVAL`. Used by glibc once per thread immediately after `clone`.

REQ-3: `ARCH_SET_GS`: writes `addr` into `task.thread.gsbase` (user GS, NOT kernel GS — kernel GS is preserved across user/kernel transitions via `swapgs`). Canonical-address check identical to FS.

REQ-4: `ARCH_GET_FS` / `ARCH_GET_GS`: copies current FS/GS base to `*addr`. Returns 0 on success; `-EFAULT` if `addr` is invalid.

REQ-5: `ARCH_SET_CPUID(0)`: enables the CPUID-fault feature; subsequent `cpuid` instruction in user mode delivers `SIGSEGV` (with `si_code = SI_KERNEL`). Per-thread. Inherited by `fork`/`clone`. Cleared on `execve`.

REQ-6: `ARCH_SET_CPUID(1)`: re-enables CPUID (default). Allowed unconditionally; the security policy is "off = locked"; you can always turn it back on for yourself but a sandbox keeps it off via `no_new_privs` + seccomp.

REQ-7: `ARCH_GET_XCOMP_*` / `ARCH_REQ_XCOMP_PERM`: governs dynamic XSAVE feature enabling (AMX tile data). Requesting permission for an XCR0 bit may take a per-process irreversible commitment that subsequent vm-rlimit checks honor.

REQ-8: `ARCH_MAP_VDSO_*`: maps the vDSO image of the indicated ABI at `addr` (page-aligned). Requires `CAP_SYS_ADMIN`. Used exclusively by CRIU to restore vDSO at the same address as during checkpoint. Returns the address.

REQ-9: `ARCH_SHSTK_ENABLE`: allocates a shadow-stack vma of size matching `RLIMIT_STACK / 8` (rounded to pages), writes its base+size into MSR_IA32_PL3_SSP, and sets `task.cet.shstk_enabled`. Requires CPUID feature CET_SS.

REQ-10: `ARCH_SHSTK_DISABLE`: deallocates the shstk vma, clears the MSR. Fails `-EBUSY` if `ARCH_SHSTK_LOCK` has been set.

REQ-11: `ARCH_SHSTK_LOCK`: irreversibly locks the current shstk/wrss policy. Future `ARCH_SHSTK_DISABLE` returns `-EBUSY`. Inherited across `execve`.

REQ-12: `ARCH_ENABLE_TAGGED_ADDR(nr_bits)`: enables Linear Address Masking. `nr_bits` ∈ {0, 48, 57}. `0` disables LAM (if not locked). `48` enables LAM_U48 (top 16 bits of user pointers ignored). `57` enables LAM_U57 (top 7 bits ignored), valid only with LA57 paging. Returns `-ENODEV` if CPU lacks LAM.

REQ-13: After `ARCH_ENABLE_TAGGED_ADDR`, the kernel's `untagged_addr()` strips the mask before dereference; userspace can stash up to `nr_bits` of metadata in pointer high bits without faulting.

REQ-14: `ARCH_GET_UNTAG_MASK`: writes the currently active untag mask. `0` if LAM disabled.

REQ-15: `ARCH_FORCE_TAGGED_SVA`: enables tagged pointers for shared-VA contexts (IOMMU SVA / PASID). Per-mm.

REQ-16: x32 ABI numbers OR'd with `0x40000000` (e.g. `__NR_arch_prctl_x32 = 0x40000000 | 158`).

REQ-17: Unknown `code` returns `-EINVAL`; no partial state mutation.

REQ-18: All `ARCH_SET_*` ops affect ONLY the calling task/thread except `ARCH_FORCE_TAGGED_SVA` (per-mm) and `ARCH_MAP_VDSO_*` (per-mm).

## Acceptance Criteria

- [ ] AC-1: `arch_prctl(ARCH_SET_FS, 0x7fff_1000_0000)` succeeds; subsequent `arch_prctl(ARCH_GET_FS, &x)` reports the same value.
- [ ] AC-2: `arch_prctl(ARCH_SET_FS, 0x8000_0000_0000_0000)` (non-canonical) returns `-EINVAL`.
- [ ] AC-3: `arch_prctl(ARCH_GET_FS, NULL)` returns `-EFAULT`.
- [ ] AC-4: `arch_prctl(ARCH_SET_GS, addr)` does not perturb FS base.
- [ ] AC-5: `arch_prctl(ARCH_SET_CPUID, 0)` then user-mode `cpuid` delivers SIGSEGV.
- [ ] AC-6: `arch_prctl(ARCH_SET_CPUID, 1)` re-enables and `cpuid` works again.
- [ ] AC-7: On a CPU without CET, `arch_prctl(ARCH_SHSTK_ENABLE, 0)` returns `-ENODEV`.
- [ ] AC-8: `arch_prctl(ARCH_SHSTK_ENABLE, 0)` then `ARCH_SHSTK_LOCK` then `ARCH_SHSTK_DISABLE` returns `-EBUSY`.
- [ ] AC-9: After `ARCH_SHSTK_ENABLE`, attempting a ret to a non-shstk-recorded address faults with `SIGSEGV` and `si_code = SEGV_CPERR`.
- [ ] AC-10: `arch_prctl(ARCH_ENABLE_TAGGED_ADDR, 48)` on CPU with LAM_U48 succeeds; subsequent `ARCH_GET_UNTAG_MASK` reports a mask with bits [62:48].
- [ ] AC-11: `arch_prctl(ARCH_ENABLE_TAGGED_ADDR, 57)` without LA57 paging returns `-EINVAL`.
- [ ] AC-12: `arch_prctl(ARCH_MAP_VDSO_64, addr)` without `CAP_SYS_ADMIN` returns `-EPERM`.
- [ ] AC-13: `arch_prctl(ARCH_REQ_XCOMP_PERM, XFEATURE_XTILEDATA)` succeeds on AMX CPU; subsequent `ARCH_GET_XCOMP_PERM` reflects the grant.
- [ ] AC-14: `arch_prctl(0xDEADBEEF, 0)` returns `-EINVAL`.
- [ ] AC-15: After `ARCH_SET_FS`, kernel re-enters user mode with `MSR_FS_BASE` matching the requested value (via thread-switch).
- [ ] AC-16: Cross-thread isolation: thread A's `ARCH_SET_FS` does NOT affect thread B's FS base.

## Architecture

```rust
#[syscall(nr = 158, abi = "sysv")]
pub fn sys_arch_prctl(code: i32, addr: usize) -> isize {
    ArchPrctl::dispatch(code, addr)
}
```

`ArchPrctl::dispatch(code, addr) -> isize`:
1. match code {
2.   ARCH_SET_FS            => ArchPrctl::set_fs(addr),
3.   ARCH_SET_GS            => ArchPrctl::set_gs(addr),
4.   ARCH_GET_FS            => ArchPrctl::get_fs(UserPtr::from(addr)),
5.   ARCH_GET_GS            => ArchPrctl::get_gs(UserPtr::from(addr)),
6.   ARCH_GET_CPUID         => Ok(task.thread.cpuid_enabled as isize),
7.   ARCH_SET_CPUID         => ArchPrctl::set_cpuid(addr),
8.   ARCH_GET_XCOMP_SUPP    => ArchPrctl::get_xcomp_supp(UserPtr::from(addr)),
9.   ARCH_GET_XCOMP_PERM    => ArchPrctl::get_xcomp_perm(UserPtr::from(addr)),
10.  ARCH_REQ_XCOMP_PERM    => ArchPrctl::req_xcomp_perm(addr as u32),
11.  ARCH_MAP_VDSO_X32      => Vdso::map(addr, VdsoAbi::X32),
12.  ARCH_MAP_VDSO_32       => Vdso::map(addr, VdsoAbi::I386),
13.  ARCH_MAP_VDSO_64       => Vdso::map(addr, VdsoAbi::X86_64),
14.  ARCH_SHSTK_ENABLE      => Cet::shstk_enable(addr as u64),
15.  ARCH_SHSTK_DISABLE     => Cet::shstk_disable(addr as u64),
16.  ARCH_SHSTK_LOCK        => Cet::shstk_lock(addr as u64),
17.  ARCH_SHSTK_STATUS      => Cet::shstk_status(UserPtr::from(addr)),
18.  ARCH_ENABLE_TAGGED_ADDR=> Lam::enable(addr),
19.  ARCH_GET_UNTAG_MASK    => Lam::get_untag_mask(UserPtr::from(addr)),
20.  ARCH_GET_MAX_TAG_BITS  => Lam::get_max_tag_bits(UserPtr::from(addr)),
21.  ARCH_FORCE_TAGGED_SVA  => Lam::force_tagged_sva(),
22.  _                      => Err(EINVAL),
23. }

`ArchPrctl::set_fs(addr)`:
1. if !is_canonical(addr) { return Err(EINVAL); }
2. task.thread.fsbase = addr;
3. wrmsrl(MSR_FS_BASE, addr);
4. Ok(0)

`ArchPrctl::set_cpuid(addr)`:
1. if addr != 0 && addr != 1 { return Err(EINVAL); }
2. if !cpu_feature(X86_FEATURE_CPUID_FAULT) { return Err(ENODEV); }
3. task.thread.cpuid_enabled = (addr == 1);
4. set_cpuid_faulting(!task.thread.cpuid_enabled);
5. Ok(0)

`Cet::shstk_enable(features)`:
1. if !cpu_feature(X86_FEATURE_SHSTK) { return Err(ENODEV); }
2. if task.cet.locked { return Err(EBUSY); }
3. let size = round_up_pages(rlimit(RLIMIT_STACK) / 8);
4. let shstk = Mm::map_shstk(size)?;
5. wrmsrl(MSR_IA32_PL3_SSP, shstk.base + shstk.size);
6. task.cet.shstk_base = shstk.base;
7. task.cet.shstk_enabled = true;
8. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fs_gs_canonical` | INVARIANT | per-`ARCH_SET_FS/GS`: addr canonical or `-EINVAL`. |
| `cpuid_arg_validated` | INVARIANT | per-`ARCH_SET_CPUID`: arg ∈ {0,1}. |
| `shstk_locked_no_disable` | INVARIANT | per-`ARCH_SHSTK_DISABLE`: locked ⟹ `-EBUSY`. |
| `lam_nr_bits_valid` | INVARIANT | per-`ARCH_ENABLE_TAGGED_ADDR`: nr_bits ∈ {0,48,57}. |
| `vdso_map_caps` | INVARIANT | per-`ARCH_MAP_VDSO_*`: `CAP_SYS_ADMIN` enforced. |
| `xcomp_perm_monotone` | INVARIANT | per-`ARCH_REQ_XCOMP_PERM`: granted bits only grow. |
| `unknown_code_einval` | INVARIANT | unknown `code` ⟹ `-EINVAL`, no state change. |

### Layer 2: TLA+

`arch/x86/arch_prctl.tla`:
- States per task: {fsbase, gsbase, cpuid_enabled, shstk_enabled, shstk_locked, lam_mode, xcomp_perm}.
- Properties:
  - `safety_fs_gs_canonical` — invariant.
  - `safety_shstk_lock_monotone` — locked never unlocked except via `CAP_SYS_ADMIN` debug path.
  - `safety_xcomp_perm_grows` — XCR0 permission bits never decrease.
  - `safety_cpuid_per_thread` — set on one thread does not affect others.
  - `liveness_dispatch_returns` — every code terminates with definite errno or 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `set_fs` post: MSR_FS_BASE == addr | `ArchPrctl::set_fs` |
| `set_cpuid` post: faulting bit matches enabled | `ArchPrctl::set_cpuid` |
| `shstk_enable` post: SSP MSR holds top of shstk vma | `Cet::shstk_enable` |
| `lam_enable` post: untag mask covers requested nr_bits | `Lam::enable` |

### Layer 4: Verus / Creusot functional

Per-`arch_prctl(2)` man-page semantic equivalence. LTP `arch_prctl0[1-3]` and the CET self-tests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`arch_prctl(2)` reinforcement:

- **Per-`ARCH_SET_FS/GS` canonical check** — defense against per-non-canonical-MSR-write fault inside kernel.
- **Per-`ARCH_SET_CPUID(0)` SIGSEGV** — defense against per-fingerprint via cpuid leaf 0x40000000.
- **Per-`ARCH_SHSTK_LOCK` irreversible** — defense against per-CET-bypass via runtime disable.
- **Per-`ARCH_SHSTK_ENABLE` shstk vma R^W^X** — defense against per-shstk-corruption via writable mapping.
- **Per-`ARCH_MAP_VDSO_*` `CAP_SYS_ADMIN`** — defense against per-vDSO-relocation as ROP gadget surface.
- **Per-`ARCH_ENABLE_TAGGED_ADDR` LAM checks paging mode** — defense against per-LAM_U57 on non-LA57.
- **Per-`ARCH_REQ_XCOMP_PERM` monotone** — defense against per-XSAVE feature confusion.

## Grsecurity / PaX surface

- **PaX UDEREF on `ARCH_GET_FS/GS` user pointer** — defense against per-MSR-leak via kernel deref bug.
- **PAX_RANDKSTACK at arch_prctl entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — kernel does not expose raw MSR values via any side channel; only the FS/GS base seen by user is the value user set.
- **Per-grsec `ARCH_MAP_VDSO_*` audit** — every vDSO remap logged with task creds and address.
- **PaX MPROTECT integration with CET** — `ARCH_SHSTK_ENABLE` shstk vma is enforced read-only-except-shstk by MPROTECT lock; CET hardware further restricts writes to `wrss` only.
- **GRKERNSEC: CET lock mandatory on suid exec** — `execve` of a CET-enabled binary auto-issues `ARCH_SHSTK_LOCK` after enable.
- **PaX LAM disable in chroot** — inside grsec chroot, `ARCH_ENABLE_TAGGED_ADDR(nonzero)` returns `-EPERM` (tagged pointers in chroots are an explicit policy decision).
- **Per-grsec CPUID-fault default-on for sandboxed tasks** — sandbox launcher invokes `ARCH_SET_CPUID(0)` automatically; grsec policy can force this even without explicit launcher action.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- CET hardware semantics (Tier-3 in `arch/x86/cet.md`).
- LAM page-table interaction (Tier-3 in `arch/x86/lam.md`).
- AMX / XSAVE dynamic-feature machinery (Tier-3 in `arch/x86/xsave.md`).
- vDSO image build / signing (Tier-3 in `arch/x86/vdso.md`).
- Implementation code.
