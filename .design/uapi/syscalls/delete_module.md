# Tier-5 syscall: delete_module(2) — syscall 176

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/module/main.c (SYSCALL_DEFINE2(delete_module), free_module)
  - kernel/module/internal.h
  - include/uapi/linux/module.h (O_NONBLOCK, O_TRUNC)
  - arch/x86/entry/syscalls/syscall_64.tbl (176  common  delete_module)
-->

## Summary

`delete_module(2)` unloads a previously loaded kernel module by name. The kernel verifies the module exists, has refcount zero (no live users), invokes the module's `exit()` function, then unmaps and frees the module's text/data/rodata/bss memory and removes its sysfs / procfs entries. The two `flags` bits relax the "wait until idle" behaviour:
- `O_NONBLOCK` — fail with `-EWOULDBLOCK` if module is in use (do not wait).
- `O_TRUNC` — force unload even if `MODULE_FORCE_UNLOAD` config is set; combined with `O_NONBLOCK` it skips waiting and forces.

Used by `rmmod`, `modprobe -r`, livepatch teardown, and module-self-removal handlers. Critical for: module lifecycle, refcount discipline, livepatch rollback, secure-boot lockdown enforcement.

## Signature

```c
int delete_module(const char *name, unsigned int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `name` | `const char *` | in | NUL-terminated module name (matches `MODULE_NAME` from the loaded module's `.modinfo`). Max length `MODULE_NAME_LEN - 1 = 55` bytes. |
| `flags` | `unsigned int` | in | Bitwise OR of `O_NONBLOCK` (0x800) and `O_TRUNC` (0x200). Unknown bits → `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Module unloaded; `exit()` called; memory freed. |
| `-1` + `errno` | Unload failed; module still loaded. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_MODULE`; `kernel.modules_disabled=1`; module marked permanent (`module->exit == NULL && !O_TRUNC`); `CONFIG_MODULE_FORCE_UNLOAD=n` and `O_TRUNC` requested. |
| `EFAULT` | `name` user pointer faults during copy. |
| `ENOENT` | No loaded module with that name. |
| `EWOULDBLOCK` | `O_NONBLOCK` set and module refcount > 0. |
| `EBUSY` | Module's `init()` still running (race with finit_module). |
| `EINVAL` | Unknown flag bit; module name longer than `MODULE_NAME_LEN`; embedded NUL. |

## ABI surface

```text
__NR_delete_module  (x86_64)  = 176
__NR_delete_module  (arm64)   = 106
__NR_delete_module  (riscv)   = 106
__NR_delete_module  (i386)    = 129

/* Flags reuse classic open(2) bits but semantics are module-unload:
     O_NONBLOCK  -> do not wait for refcount to drain
     O_TRUNC     -> force unload (only honoured if MODULE_FORCE_UNLOAD=y)
   Combination O_NONBLOCK|O_TRUNC -> "force-unload, do not block." */
```

## Compatibility contract

REQ-1: Syscall number is **176** on x86_64. ABI-stable.

REQ-2: Caller MUST have `CAP_SYS_MODULE` in init_user_ns. Capability check precedes copy_from_user.

REQ-3: `flags & ~(O_NONBLOCK | O_TRUNC)` non-zero → `-EINVAL`.

REQ-4: `name` copied with `strncpy_from_user` capped at `MODULE_NAME_LEN`; truncation or `EFAULT` → `-EINVAL` / `-EFAULT`.

REQ-5: `kernel.modules_disabled=1` → `-EPERM` (terminal until reboot).

REQ-6: Lookup module by name in MODULE_LIST under `modules_mutex`. Not found → `-ENOENT`.

REQ-7: Refcount check:
- if `module_refcount(m) > 0`:
  - if `O_NONBLOCK`: return `-EWOULDBLOCK`.
  - else: wait on `module->waitq` for refcount-zero (interruptible).
- if `m->state == MODULE_STATE_COMING` (init() still executing): return `-EBUSY`.

REQ-8: `O_TRUNC` (force): only honoured if `CONFIG_MODULE_FORCE_UNLOAD=y`. Forces unload regardless of refcount, taints kernel `TAINT_FORCED_MODULE`. Under hardened mode this flag is rejected with `-EPERM`.

REQ-9: Permanent modules: a module without an `exit()` function may not be unloaded; `-EBUSY` (or `-EPERM` if force denied).

REQ-10: `exit()` invocation:
- a) Module state → MODULE_STATE_GOING.
- b) Call `m->exit()` — must not fail; return value ignored.
- c) Detach from MODULE_LIST.
- d) Remove `/sys/module/<name>/` entries.
- e) Wait for all readers of removed entries to drop.
- f) Free module memory (text, data, rodata, init, percpu).

REQ-11: Module-self-removal: a module may call `delete_module` on itself; kernel detects (`current's mm == module's mm` is impossible — actually via task module ref accounting), and serializes via `modules_mutex` so the exit() runs in a kthread context, not in the calling syscall's stack-frame ownership of module text.

REQ-12: `kernel_lockdown(integrity)`: delete_module remains permitted (removing code never violates integrity guarantees), but tracing-tainted modules may still impose `kernel_lockdown` violations downstream.

REQ-13: Per-module-notifier chain (MODULE_STATE_GOING) fires before memory free; subsystems (perf, livepatch, kallsyms) remove their per-module data.

REQ-14: Successful unload: module disappears from `/proc/modules`, `/sys/module/<name>/`, `kallsyms`, `/proc/kallsyms`.

REQ-15: Concurrent delete_module of same name: serialized via modules_mutex; loser observes `-ENOENT` after winner completes.

## Acceptance Criteria

- [ ] AC-1: `delete_module("test_mod", 0)` on idle module: returns 0.
- [ ] AC-2: Caller without `CAP_SYS_MODULE`: `-EPERM`.
- [ ] AC-3: `name = "nonexistent"`: `-ENOENT`.
- [ ] AC-4: `name = NULL`: `-EFAULT`.
- [ ] AC-5: Module refcount > 0, `O_NONBLOCK`: `-EWOULDBLOCK`.
- [ ] AC-6: Module refcount > 0, no flag: blocks until refcount drains or signal.
- [ ] AC-7: `flags = 0x100` (unknown bit): `-EINVAL`.
- [ ] AC-8: `O_TRUNC` with `CONFIG_MODULE_FORCE_UNLOAD=n`: `-EPERM`.
- [ ] AC-9: `O_TRUNC` with refcount > 0, `MODULE_FORCE_UNLOAD=y`: unload succeeds; kernel tainted.
- [ ] AC-10: Permanent module (`exit == NULL`): `-EBUSY` (or `-EPERM`).
- [ ] AC-11: `kernel.modules_disabled=1`: `-EPERM`.
- [ ] AC-12: After successful unload: `/proc/modules` entry gone; `lsmod` empty.
- [ ] AC-13: After successful unload: `delete_module` again returns `-ENOENT`.
- [ ] AC-14: Race: two concurrent delete_module same name: one wins (0), other `-ENOENT`.

## Architecture

```rust
#[syscall(nr = 176, abi = "sysv")]
pub fn sys_delete_module(name: UserPtr<u8>, flags: u32) -> isize {
    Module::do_delete_module(name, flags)
}
```

`Module::do_delete_module(name_ptr, flags) -> isize`:
1. Module::check_cap_sys_module(&current_creds())?;     // EPERM
2. if sysctl_modules_disabled { return -EPERM; }
3. const VALID = O_NONBLOCK | O_TRUNC;
4. if (flags & !VALID) != 0 { return -EINVAL; }
5. let name = Module::copy_name_from_user(name_ptr, MODULE_NAME_LEN)?;  // EFAULT / EINVAL
6. if (flags & O_TRUNC) != 0 && !cfg!(MODULE_FORCE_UNLOAD) { return -EPERM; }
7. let mutex = MODULES_MUTEX.lock();
8. let m = Module::find_loaded(&name).ok_or(ENOENT)?;
9. if m.state == MODULE_STATE_COMING { return -EBUSY; }
10. if m.exit.is_none() && (flags & O_TRUNC) == 0 { return -EBUSY; }
11. loop {
12.   let rc = module_refcount(&m);
13.   if rc == 0 { break; }
14.   if (flags & O_TRUNC) != 0 { add_taint(TAINT_FORCED_MODULE); break; }
15.   if (flags & O_NONBLOCK) != 0 { return -EWOULDBLOCK; }
16.   drop(mutex);
17.   wait_event_interruptible(m.waitq, module_refcount(&m) == 0)?;     // EINTR
18.   mutex = MODULES_MUTEX.lock();
19. }
20. m.state = MODULE_STATE_GOING;
21. module_notifier_chain.call(MODULE_STATE_GOING, &m);
22. if let Some(exit_fn) = m.exit { exit_fn(); }
23. Module::detach_from_list(&m);
24. Module::remove_sysfs(&m);
25. Module::free_memory(m);
26. 0

`Module::copy_name_from_user(uptr, limit) -> Result<String>`:
1. let mut buf = [0u8; MODULE_NAME_LEN];
2. let n = strncpy_from_user(&mut buf, uptr, limit)?;
3. if n == limit { return Err(EINVAL); }       // not NUL-terminated
4. if buf[..n].iter().any(|&b| b == b'/' || b == 0) { return Err(EINVAL); }
5. Ok(String::from_utf8(buf[..n].to_vec())?)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM returned before copy_from_user. |
| `name_length_bounded` | INVARIANT | name ≤ MODULE_NAME_LEN-1. |
| `force_requires_config` | INVARIANT | O_TRUNC ∧ !MODULE_FORCE_UNLOAD ⟹ EPERM. |
| `refcount_zero_before_exit` | INVARIANT | exit() invoked only after refcount == 0 ∨ force. |
| `state_transitions_monotonic` | INVARIANT | LIVE → GOING → freed; no back-edge. |
| `mutex_held_during_unlink` | INVARIANT | modules_mutex held during detach_from_list. |

### Layer 2: TLA+

`kernel/module-delete.tla`:
- States: per-cap, per-copy-name, per-lookup, per-refcount-wait, per-state-going, per-exit, per-detach, per-free.
- Properties:
  - `safety_no_unload_with_refcount_unless_force`.
  - `safety_exit_runs_once`.
  - `safety_concurrent_delete_serialized`.
  - `safety_force_blocked_under_hardened`.
  - `liveness_delete_terminates_or_returns_ewouldblock`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_delete_module` post: ret 0 ⟹ module absent from MODULE_LIST | `Module::do_delete_module` |
| `copy_name_from_user` post: name length ≤ MODULE_NAME_LEN-1 | `Module::copy_name_from_user` |
| `free_memory` post: text/data/rodata/init/percpu pages freed | `Module::free_memory` |
| `module_notifier_chain` post: GOING fired before exit() | `Module::do_delete_module` |

### Layer 4: Verus / Creusot functional

Per-`delete_module(2)` man-page and per-`kernel/module/main.c` semantic equivalence. LTP `delete_module*`, kmod test-suite pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`delete_module(2)` reinforcement:

- **Per-cap-check pre-copy** — defense against per-priv-escalation via name smuggling.
- **Per-name length cap** — defense against per-overflow on copy.
- **Per-refcount-zero-or-force enforcement** — defense against per-UAF on still-in-use module.
- **Per-state-machine monotonic** — defense against per-double-exit.
- **Per-modules_disabled terminal** — defense against per-runtime-attack module removal.

## Grsecurity / PaX surface

- **PaX UDEREF on `name` copy_from_user** — defense against per-name-pointer kernel-deref bug; SMAP forced for the strncpy_from_user range.
- **CAP_SYS_MODULE strict in init_user_ns only** — child userns root denied; GRKERNSEC_CHROOT_CAPS extends this to chroot.
- **GRKERNSEC_KMOD restrictions** — `delete_module` also denied once `kernel.modules_disabled=1` is latched (after late-boot), aligning with `init_module`/`finit_module` restrictions.
- **GRKERNSEC_MODHARDEN (require signed modules)** — applies to load only, but delete_module under hardened mode rejects `O_TRUNC` unconditionally (`-EPERM`), preventing forced unload of trusted modules.
- **GRKERNSEC_KMOD_FORCE_DENY** — `O_TRUNC` is rejected with `-EPERM` regardless of `CONFIG_MODULE_FORCE_UNLOAD` build setting. Forced unloads are a known UAF vector and are entirely disabled.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity`, removal of livepatch modules requires an extra check (livepatch revert path), otherwise `-EPERM`.
- **PAX_USERCOPY_HARDEN on name copy_from_user** — bounded copy via whitelisted bounce buffer.
- **PaX KERNEXEC freed text pages** — module text pages flushed and unmapped from kernel page tables before slab return; defense against per-stale-text execution.
- **GRKERNSEC_HIDESYM on freed module sections** — kallsyms cache invalidated under modules_mutex; address reuse not leaked.
- **PAX_REFCOUNT on module refcount** — defense against per-refcount-underflow on concurrent rmmod.
- **GRKERNSEC_AUDIT_MODULES** — every delete_module logged with PID, comm, module name, flags, return code; force-unloads audit-flagged as high severity.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-module-exit stack overflow.
- **GRKERNSEC_DMESG** — module-exit `printk` output gated.
- **Permanent-module never-unload enforced** — under hardened mode, even with `O_TRUNC` a module without an `exit()` function returns `-EPERM`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Module memory free per-arch (covered in arch Tier-3 docs).
- Livepatch revert (covered in Tier-3 `kernel/livepatch.md`).
- Module notifier subsystems (perf, kallsyms) (covered in their Tier-3 docs).
- Implementation code.
