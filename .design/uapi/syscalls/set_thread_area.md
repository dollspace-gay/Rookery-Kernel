# Tier-5 syscall: set_thread_area(2) — syscall 205

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/kernel/tls.c (do_set_thread_area)
  - arch/x86/include/asm/desc.h
  - arch/x86/include/uapi/asm/ldt.h (struct user_desc — shared with modify_ldt)
  - arch/x86/entry/syscalls/syscall_64.tbl (205  64  set_thread_area)
  - arch/x86/entry/syscalls/syscall_32.tbl (243  i386 set_thread_area)
-->

## Summary

`set_thread_area(2)` installs a 32-bit TLS (thread-local storage) descriptor into one of three reserved per-thread GDT slots (`GDT_ENTRY_TLS_MIN=12 .. GDT_ENTRY_TLS_MAX=14`). It is the x86_32 / x32 legacy mechanism by which glibc anchors the TLS pointer (`%gs:0` on i386). Each thread gets up to three TLS slots, individually selectable by writing `GDT_ENTRY_TLS_MIN*8+RPL3=0x33` (slot 0), `0x3B` (slot 1), or `0x43` (slot 2) into `%gs`. On x86_64 native, glibc instead uses `arch_prctl(ARCH_SET_FS, …)` and ignores `set_thread_area`, but the syscall is preserved for x32 ABI and for compat-32 binaries executing under a 64-bit kernel.

Critical for: i386 glibc/musl thread init, 32-bit Wine, 32-bit Java VMs (Hotspot uses TLS), x32 ABI runtimes, and CRIU restoring a 32-bit process's per-thread GDT state.

This Tier-5 covers `set_thread_area(2)` on x86 (both 32-bit native and 32-bit compat under x86_64). The retrieval cousin is `get_thread_area(2)`.

## Signature

```c
int set_thread_area(struct user_desc *u_info);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `u_info` | `struct user_desc *` | in/out | Descriptor to install. `entry_number == -1` requests "any free slot"; kernel writes back the chosen slot. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. `u_info->entry_number` updated if it was `-1`. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`   | `u_info == NULL`; `entry_number` not -1 and not in `[GDT_ENTRY_TLS_MIN, GDT_ENTRY_TLS_MAX]`; descriptor contents not data segment; `lm` bit set (long-mode TLS forbidden); `base_addr` non-canonical on x86_64; reserved bits set. |
| `EFAULT`   | `u_info` not readable / writable. |
| `ESRCH`    | All three TLS slots in use and `entry_number == -1`. |
| `ENOSYS`   | Compiled out (rare; x86_64 native non-compat may stub). |

## ABI surface

```text
__NR_set_thread_area (i386)   = 243
__NR_set_thread_area (x86_64) = 205   (used by x32 ABI and 32-bit compat tasks)

GDT_ENTRY_TLS_MIN = 12
GDT_ENTRY_TLS_MAX = 14
GDT_ENTRY_TLS_ENTRIES = 3

struct user_desc {                /* same layout as modify_ldt */
    unsigned int  entry_number;   /* in: -1 = any free, or 12..14; out: chosen */
    unsigned int  base_addr;      /* 32-bit base of TLS area */
    unsigned int  limit;          /* 20-bit limit */
    unsigned int  seg_32bit:1;    /* must be 1 for TLS */
    unsigned int  contents:2;     /* must be 0 (data, expand-up) */
    unsigned int  read_exec_only:1;
    unsigned int  limit_in_pages:1;
    unsigned int  seg_not_present:1;
    unsigned int  useable:1;
    unsigned int  lm:1;           /* must be 0 — TLS slots are 32-bit */
};

TLS selectors after install:
  slot 0 (entry 12) → selector 0x33  (12<<3 | 3)
  slot 1 (entry 13) → selector 0x3B  (13<<3 | 3)
  slot 2 (entry 14) → selector 0x43  (14<<3 | 3)
```

### Limits

```text
entry_number  : -1 (any), or 12, 13, 14
slots/thread  : 3
base_addr     : 0 .. 0xFFFFFFFF (32-bit only — TLS via GDT cannot cover high half)
limit         : 0 .. 0xFFFFF (byte units) or 0 .. 0xFFFFF pages (4 GiB)
descriptor type: data/expand-up (contents == 0); code descriptors rejected
```

## Compatibility contract

REQ-1: Syscall number is **205** on x86_64, **243** on i386. ABI-stable.

REQ-2: `u_info == NULL` returns `-EINVAL` (Linux historical; some Unices return `-EFAULT`).

REQ-3: `entry_number == -1`: kernel searches `GDT_ENTRY_TLS_MIN..=GDT_ENTRY_TLS_MAX` for a free or "empty" slot. If found, installs there and writes the chosen index back to `u_info->entry_number`. If all three occupied with non-empty descriptors, returns `-ESRCH`.

REQ-4: `entry_number ∈ [12,14]`: kernel installs at that fixed slot, replacing the previous descriptor.

REQ-5: Descriptor validation:
- `contents` MUST equal 0 (data, expand-up). Code descriptors and expand-down forbidden.
- `lm` MUST equal 0. The kernel rejects 64-bit TLS descriptors via this entry; use `arch_prctl(ARCH_SET_FS)` instead.
- `seg_32bit` MUST equal 1.
- `seg_not_present` honored: if 1, the slot is cleared (TLS "delete").
- `useable` is reflected verbatim in the AVL bit.

REQ-6: Per-write: DPL forced to 3. Bits 5-6 of the access byte set to `0b11`.

REQ-7: Per-thread scope: the three GDT TLS slots are saved/restored as part of `task.thread.tls_array[3]` on context switch. They are NOT shared between threads — each thread has its own TLS values.

REQ-8: Per-fork: child inherits `tls_array` from parent verbatim (clone semantics).

REQ-9: Per-clone(CLONE_SETTLS, ...): the kernel calls the equivalent of `set_thread_area` on the child before first dispatch, using the `tls` argument as the `user_desc`.

REQ-10: Per-execve: TLS slots are cleared (zeroed).

REQ-11: Per-context-switch path: `load_TLS()` in `__switch_to` rewrites GDT entries 12..14 from `next.thread.tls_array` before the next task runs. This is per-CPU work.

REQ-12: x86_64-native non-compat: `set_thread_area` is still defined but rarely used; glibc's 64-bit thread init goes via `arch_prctl`. The TLS slots remain usable for legacy 32-bit code embedded in 64-bit processes (e.g., `lcall` to 32-bit segments).

REQ-13: Per-"empty descriptor": `entry_number ∈ [12,14] && base_addr == 0 && limit == 0 && seg_not_present == 1 && contents == 0` clears the slot (logically empty).

REQ-14: x32 ABI: this syscall is the canonical TLS install path for x32 programs since x32 ELF uses 32-bit pointers and `%fs`/`%gs` GDT slots.

REQ-15: Per-CRIU: dump reads `tls_array` via `ptrace(PTRACE_GET_THREAD_AREA, ...)`; restore writes via `set_thread_area` for each thread before resume.

REQ-16: Unknown `entry_number` (e.g., 15, 0, -2) returns `-EINVAL`; no state mutation.

REQ-17: Visibility ordering: after `set_thread_area` returns, a subsequent user-mode load of `%gs` with the corresponding selector observes the new base — barrier inserted in the syscall return path.

## Acceptance Criteria

- [ ] AC-1: `set_thread_area(&desc)` with `entry_number=-1`, valid data descriptor, returns 0 and writes back `entry_number ∈ {12,13,14}`.
- [ ] AC-2: After install, loading the returned selector into `%gs` and reading `%gs:0` returns the byte at `base_addr` (i386 verification).
- [ ] AC-3: `set_thread_area(NULL)` returns `-EINVAL`.
- [ ] AC-4: `set_thread_area(&desc)` with `entry_number=15` returns `-EINVAL`.
- [ ] AC-5: `set_thread_area(&desc)` with `contents=2` (code) returns `-EINVAL`.
- [ ] AC-6: `set_thread_area(&desc)` with `lm=1` returns `-EINVAL`.
- [ ] AC-7: Three sequential calls with `entry_number=-1` and non-empty descriptors fill slots 12,13,14; fourth returns `-ESRCH`.
- [ ] AC-8: After installing slot 12, then re-calling with same slot and `seg_not_present=1 && base=0 && limit=0`, slot is cleared; a re-`-1` query then reuses 12.
- [ ] AC-9: Per-thread isolation: thread A install of slot 12 does NOT affect thread B's slot 12 (verified via `ptrace(PTRACE_GET_THREAD_AREA)`).
- [ ] AC-10: After execve, all three slots are zero (no inherited TLS).
- [ ] AC-11: `clone(CLONE_SETTLS, ...)` with a `user_desc` installs the descriptor in the child's GDT before the child runs.
- [ ] AC-12: Per-LTP `set_thread_area01` passes.

## Architecture

```rust
#[syscall(nr = 205, abi = "sysv")]
pub fn sys_set_thread_area(u_info: UserPtr<UserDesc>) -> isize {
    SetThreadArea::dispatch(u_info)
}
```

`SetThreadArea::dispatch(u_info) -> isize`:
1. if u_info.is_null() { return Err(EINVAL); }
2. let mut desc: UserDesc = u_info.read()?;
3. let slot = SetThreadArea::pick_slot(&desc, current)?;
4. SetThreadArea::validate(&desc)?;
5. SetThreadArea::install(current, slot, &desc);
6. desc.entry_number = slot as u32;
7. u_info.write(&desc)?;     // write back chosen slot
8. Ok(0)

`SetThreadArea::pick_slot(desc, task) -> Result<usize,i32>`:
1. if desc.entry_number == u32::MAX /* -1 */ {
2.   for i in GDT_ENTRY_TLS_MIN..=GDT_ENTRY_TLS_MAX {
3.     if task.thread.tls_array[i - GDT_ENTRY_TLS_MIN].is_empty() { return Ok(i); }
4.   }
5.   return Err(ESRCH);
6. }
7. if desc.entry_number < GDT_ENTRY_TLS_MIN as u32 ||
8.    desc.entry_number > GDT_ENTRY_TLS_MAX as u32 { return Err(EINVAL); }
9. Ok(desc.entry_number as usize)

`SetThreadArea::validate(d) -> Result<(),i32>`:
1. if d.contents != 0 { return Err(EINVAL); }
2. if d.lm != 0 { return Err(EINVAL); }
3. if d.seg_32bit != 1 { return Err(EINVAL); }
4. Ok(())

`SetThreadArea::install(task, slot, desc)`:
1. let enc: u64 = encode_descriptor(desc);    // DPL forced 3
2. let idx = slot - GDT_ENTRY_TLS_MIN;
3. task.thread.tls_array[idx] = enc;
4. /* if currently running on this CPU, update live GDT */
5. if task.is_current() {
6.   gdt::set_tls_entry(slot, enc);
7. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `slot_in_range` | INVARIANT | per-install: slot ∈ {12,13,14}. |
| `contents_data_only` | INVARIANT | per-validate: contents == 0. |
| `lm_zero` | INVARIANT | per-validate: lm == 0 (no 64-bit TLS via this path). |
| `dpl_forced_3` | INVARIANT | per-encode: DPL == 3. |
| `pick_slot_finds_empty` | INVARIANT | per-`-1`: returns first empty slot or ESRCH. |
| `per_thread_isolation` | INVARIANT | per-install: only `current.thread.tls_array` mutated. |

### Layer 2: TLA+

`arch/x86/set_thread_area.tla`:
- States: per-thread tls_array[3]; per-cpu live GDT[12..14].
- Properties:
  - `safety_slot_bounds` — install index ∈ {12,13,14}.
  - `safety_dpl3` — every installed descriptor DPL == 3.
  - `safety_thread_local` — install affects exactly one thread.
  - `safety_writeback` — `entry_number == -1` ⟹ chosen slot written back.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: slot in range, descriptor installed, write-back done | `SetThreadArea::dispatch` |
| `validate` post: ok ⟹ data/expand-up/32-bit | `SetThreadArea::validate` |
| `install` post: tls_array[slot-12] == encode(desc) | `SetThreadArea::install` |
| `pick_slot` post: -1 ⟹ first empty or ESRCH | `SetThreadArea::pick_slot` |

### Layer 4: Verus/Creusot functional

Per-`set_thread_area(2)` man-page semantic equivalence. LTP `set_thread_area01` and glibc 32-bit thread-init smoke pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`set_thread_area(2)` reinforcement:

- **Per-DPL-3 forced** — defense against per-ring-0 TLS descriptor injection.
- **Per-`lm == 0` enforced** — defense against per-32-bit-task forging 64-bit segment.
- **Per-`contents == 0` enforced** — defense against per-TLS-as-code escalation.
- **Per-thread `tls_array` isolation** — defense against per-cross-thread TLS clobber.
- **Per-execve clear** — defense against per-suid binary inheriting attacker TLS.
- **Per-write back of `entry_number`** — defense against per-userspace assuming wrong slot.
- **Per-context-switch barrier** — defense against per-stale-GDT TLS use across migration.

## Grsecurity / PaX surface

- **PaX UDEREF on `u_info`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at set_thread_area entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — `tls_array` never exposed via `/proc/<pid>/mem` or `/dev/kmem`.
- **PaX SEGMEXEC integration** — under SEGMEXEC, TLS slots 12/13/14 are reserved for SEGMEXEC base/limit; `set_thread_area` falls through to the same slots but the base/limit are clipped to the SEGMEXEC code segment.
- **Per-grsec audit** — every install logged with `entry_number, base, limit`.
- **GRKERNSEC_CHROOT_NO_TLS_MUNGING** — inside grsec chroot, calls that would clear slots already in use by SEGMEXEC return `-EPERM`.
- **PaX MPROTECT integration** — TLS descriptor's `base_addr` must lie in a writable, non-executable VMA; else `-EPERM`.
- **Per-grsec RBAC** — `+T` flag (TLS-allow) gates per-subject; default policy permits (TLS is too core to deny).
- **Per-grsec `set_thread_area` rate-limit** — > 1000 calls/sec from one task triggers WARN + log.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- x86 segmentation hardware semantics (Tier-3 in `arch/x86/segments.md`).
- `get_thread_area(2)` (separate Tier-5 doc).
- `arch_prctl(ARCH_SET_FS)` (covered in `arch_prctl.md`).
- `clone(CLONE_SETTLS)` mechanics (Tier-3 in `kernel/clone.md`).
- TLS implementation in glibc / musl.
- Implementation code.
