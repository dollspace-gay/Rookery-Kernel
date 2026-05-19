# Tier-5 syscall: reboot(2) — syscall 169

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/reboot.c (SYSCALL_DEFINE4(reboot), kernel_restart, kernel_halt, kernel_power_off, kernel_kexec)
  - include/uapi/linux/reboot.h (LINUX_REBOOT_MAGIC1/2, LINUX_REBOOT_CMD_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (169  common  reboot)
-->

## Summary

`reboot(2)` is the kernel-side reboot, halt, power-off, kexec, and Ctrl-Alt-Del configuration entry point. It takes two magic numbers (sanity gate against accidental syscall), a `cmd`, and an optional pointer argument used only by `LINUX_REBOOT_CMD_RESTART2` (carries a NUL-terminated restart-string passed to firmware). Commands:
- `LINUX_REBOOT_CMD_RESTART` — warm reboot via default mechanism.
- `LINUX_REBOOT_CMD_RESTART2` — warm reboot with operator-supplied magic string.
- `LINUX_REBOOT_CMD_HALT` — halt CPU; system remains powered.
- `LINUX_REBOOT_CMD_POWER_OFF` — power down.
- `LINUX_REBOOT_CMD_KEXEC` — boot a staged `kexec_load`/`kexec_file_load` image.
- `LINUX_REBOOT_CMD_CAD_ON` / `_OFF` — enable/disable Ctrl-Alt-Del → reboot binding.
- `LINUX_REBOOT_CMD_SW_SUSPEND` — hibernate (legacy; modern is `/sys/power/state`).

Critical for: orderly shutdown, kdump capture-kernel boot, server provisioning automation, embedded fail-safe.

## Signature

```c
int reboot(int magic, int magic2, int cmd, void *arg);
```

```c
#define LINUX_REBOOT_MAGIC1       0xfee1dead
#define LINUX_REBOOT_MAGIC2       672274793
#define LINUX_REBOOT_MAGIC2A      85072278
#define LINUX_REBOOT_MAGIC2B      369367448
#define LINUX_REBOOT_MAGIC2C      537993216

#define LINUX_REBOOT_CMD_RESTART    0x01234567
#define LINUX_REBOOT_CMD_HALT       0xCDEF0123
#define LINUX_REBOOT_CMD_CAD_ON     0x89ABCDEF
#define LINUX_REBOOT_CMD_CAD_OFF    0x00000000
#define LINUX_REBOOT_CMD_POWER_OFF  0x4321FEDC
#define LINUX_REBOOT_CMD_RESTART2   0xA1B2C3D4
#define LINUX_REBOOT_CMD_SW_SUSPEND 0xD000FCE2
#define LINUX_REBOOT_CMD_KEXEC      0x45584543
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `magic` | `int` | in | MUST equal `LINUX_REBOOT_MAGIC1`. |
| `magic2` | `int` | in | MUST equal one of the four `LINUX_REBOOT_MAGIC2*` constants. |
| `cmd` | `int` | in | One of `LINUX_REBOOT_CMD_*`. |
| `arg` | `void *` | in | Pointer to NUL-terminated string (`CMD_RESTART2` only); else NULL. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Returned only for `CMD_CAD_ON` / `CMD_CAD_OFF` (the no-side-effect commands). All other successful commands do not return — the kernel performs the action. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_BOOT`. |
| `EINVAL` | Magic mismatch; unknown `cmd`; `arg` required and NULL (or vice versa); `CMD_KEXEC` with no staged image. |
| `EFAULT` | `arg` user pointer faults (for `CMD_RESTART2`). |
| `ENOSYS` | `CMD_SW_SUSPEND` when `CONFIG_HIBERNATION=n`; `CMD_KEXEC` when `CONFIG_KEXEC_CORE=n`. |

## ABI surface

```text
__NR_reboot  (x86_64)  = 169
__NR_reboot  (arm64)   = 142
__NR_reboot  (riscv)   = 142
__NR_reboot  (i386)    = 88

/* arg is treated as a user pointer only for CMD_RESTART2;
   for all other commands the kernel ignores arg. */
/* CMD_RESTART2 string length capped at 256 bytes (kernel side). */
```

## Compatibility contract

REQ-1: Syscall number is **169** on x86_64. ABI-stable.

REQ-2: Caller MUST have `CAP_SYS_BOOT` in init_user_ns. Capability check precedes magic check.

REQ-3: Magic-confirm gate:
- `magic != LINUX_REBOOT_MAGIC1` → `-EINVAL`.
- `magic2 ∉ { MAGIC2, MAGIC2A, MAGIC2B, MAGIC2C }` → `-EINVAL`.
- The four magic2 values exist as historical accident; all are equally valid.

REQ-4: PID namespace gate:
- If caller is in a non-init PID namespace: only `CMD_POWER_OFF` and `CMD_RESTART` are permitted, and they translate to "kill init of this PID namespace" (signaling SIGHUP / SIGINT respectively to PID 1 of that namespace).
- Other commands → `-EINVAL` from inside a child pidns.

REQ-5: Command dispatch:
- `CMD_RESTART`: emit `KERN_EMERG "Restarting system."`, call notifier chain `reboot_notifier_list`, then `machine_restart(NULL)`.
- `CMD_RESTART2`: copy string from `arg` (NUL-terminated, ≤256 bytes), pass to `machine_restart(str)`.
- `CMD_HALT`: notifier chain, `machine_halt()`.
- `CMD_POWER_OFF`: notifier chain, `machine_power_off()`.
- `CMD_KEXEC`: requires staged kimage in normal slot; `kernel_kexec()` enters relocate trampoline. Else `-EINVAL`.
- `CMD_CAD_ON`: `C_A_D = 1`; ctrl-alt-del kernel keyboard handler will signal init.
- `CMD_CAD_OFF`: `C_A_D = 0`.
- `CMD_SW_SUSPEND`: enter `software_suspend_enter()`; legacy; modern systems use `/sys/power/state = disk`.

REQ-6: Notifier chain:
- All `reboot_notifier_list` callees invoked with the command code.
- Filesystems, network, hardware drivers, watchdogs notified for orderly shutdown.

REQ-7: `CMD_RESTART` / `CMD_HALT` / `CMD_POWER_OFF` / `CMD_KEXEC` block all CPUs (stop-machine) then perform the action; no return to userspace.

REQ-8: `CMD_KEXEC` requires `crashkernel=` NOT to be involved (this is the normal slot, not crash slot). Crash slot is entered only on panic().

REQ-9: `kernel_lockdown(integrity)` and `kernel_lockdown(confidentiality)` allow all reboot commands (reboot is integrity-preserving), with one caveat: `CMD_KEXEC` requires the staged image to have been verified at stage time (which `kexec_file_load` enforces).

REQ-10: `arg` for `CMD_RESTART2`: copy via `strncpy_from_user(buf, arg, 256)`; result is passed verbatim to firmware (e.g., for "warm vs cold" or "select OS"). Truncation accepted; non-printable bytes platform-defined.

REQ-11: Per-`/proc/sys/kernel/ctrl-alt-del`: 0 = signal init; non-zero = immediate reboot. `CMD_CAD_ON/OFF` toggles the binding between the keyboard combo and reboot path.

REQ-12: Per-task: only thread group leader semantics matter; any thread of a privileged process may invoke.

REQ-13: Per-fail-safe: if `machine_restart` returns (firmware refused), kernel falls through to `machine_halt`; if that returns, infinite loop with WARN.

REQ-14: Per-watchdog: NMI watchdog disabled during reboot path to avoid spurious panics during stop-machine.

REQ-15: Per-emergency reboot: SysRq-b directly invokes `emergency_restart()` bypassing notifiers; the reboot() syscall always runs the notifier chain.

## Acceptance Criteria

- [ ] AC-1: Caller without `CAP_SYS_BOOT`: `-EPERM`.
- [ ] AC-2: `magic = 0`: `-EINVAL`.
- [ ] AC-3: `magic2 = 0xdeadbeef`: `-EINVAL`.
- [ ] AC-4: Valid magics, `cmd = CMD_CAD_ON`: returns 0; ctrl-alt-del armed.
- [ ] AC-5: Valid magics, `cmd = CMD_CAD_OFF`: returns 0; ctrl-alt-del disarmed.
- [ ] AC-6: Valid magics, `cmd = 0xdeadbeef`: `-EINVAL`.
- [ ] AC-7: Valid magics, `cmd = CMD_RESTART2`, `arg = NULL`: `-EFAULT`.
- [ ] AC-8: Valid magics, `cmd = CMD_KEXEC` with no staged image: `-EINVAL`.
- [ ] AC-9: Inside non-init pidns, `cmd = CMD_HALT`: `-EINVAL` (or kills pidns init).
- [ ] AC-10: Inside non-init pidns, `cmd = CMD_POWER_OFF`: SIGHUP to PID 1 of that pidns; returns 0.
- [ ] AC-11: `cmd = CMD_SW_SUSPEND` with CONFIG_HIBERNATION=n: `-ENOSYS`.
- [ ] AC-12: `cmd = CMD_KEXEC` with staged image: does not return.

## Architecture

```rust
#[syscall(nr = 169, abi = "sysv")]
pub fn sys_reboot(magic: i32, magic2: i32, cmd: i32, arg: UserPtr<u8>) -> isize {
    Reboot::do_reboot(magic, magic2, cmd, arg)
}
```

`Reboot::do_reboot(magic, magic2, cmd, arg_ptr) -> isize`:
1. Reboot::check_cap_sys_boot(&current_creds())?;     // EPERM
2. if magic != LINUX_REBOOT_MAGIC1 { return -EINVAL; }
3. if !matches!(magic2 as u32, MAGIC2 | MAGIC2A | MAGIC2B | MAGIC2C) { return -EINVAL; }
4. /* PID-ns scoping */
5. if !task_active_pid_ns(current()).is_init_ns() {
6.   return Reboot::handle_pidns_reboot(cmd);
7. }
8. let mutex = REBOOT_MUTEX.lock();
9. match cmd as u32 {
10.  LINUX_REBOOT_CMD_RESTART => {
11.    blocking_notifier_call_chain(reboot_notifier_list, SYS_RESTART);
12.    machine_restart(NULL);                          // no return
13.  }
14.  LINUX_REBOOT_CMD_RESTART2 => {
15.    let cmd_str = strncpy_from_user_bounded(arg_ptr, 256)?;  // EFAULT
16.    blocking_notifier_call_chain(reboot_notifier_list, SYS_RESTART);
17.    machine_restart(cmd_str.as_ptr());             // no return
18.  }
19.  LINUX_REBOOT_CMD_HALT => {
20.    blocking_notifier_call_chain(reboot_notifier_list, SYS_HALT);
21.    machine_halt();                                 // no return
22.  }
23.  LINUX_REBOOT_CMD_POWER_OFF => {
24.    blocking_notifier_call_chain(reboot_notifier_list, SYS_POWER_OFF);
25.    machine_power_off();                            // no return
26.  }
27.  LINUX_REBOOT_CMD_KEXEC => {
28.    if !kimage_staged(Slot::Normal) { return -EINVAL; }
29.    kernel_kexec();                                 // no return
30.  }
31.  LINUX_REBOOT_CMD_CAD_ON  => { C_A_D.store(1, SeqCst); 0 }
32.  LINUX_REBOOT_CMD_CAD_OFF => { C_A_D.store(0, SeqCst); 0 }
33.  LINUX_REBOOT_CMD_SW_SUSPEND => {
34.    if !cfg!(HIBERNATION) { return -ENOSYS; }
35.    software_suspend_enter()
36.  }
37.  _ => -EINVAL,
38. }

`Reboot::handle_pidns_reboot(cmd) -> isize`:
1. let init = pid_ns_get_init(current_pidns());
2. match cmd as u32 {
3.   LINUX_REBOOT_CMD_RESTART   |
4.   LINUX_REBOOT_CMD_RESTART2 => send_sig(SIGHUP, init, 1),
5.   LINUX_REBOOT_CMD_POWER_OFF |
6.   LINUX_REBOOT_CMD_HALT     => send_sig(SIGINT, init, 1),
7.   _ => return -EINVAL,
8. }
9. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM returned before magic check. |
| `magic_confirm` | INVARIANT | both magics required ⟹ EINVAL if either wrong. |
| `pidns_scoping` | INVARIANT | non-init pidns ⟹ restart/power_off only, as signal to init. |
| `notifier_chain_called` | INVARIANT | restart/halt/power_off invoke notifier chain pre-machine_*. |
| `kexec_requires_staged` | INVARIANT | CMD_KEXEC ∧ !staged ⟹ EINVAL. |
| `unknown_cmd` | INVARIANT | unknown cmd ⟹ EINVAL. |

### Layer 2: TLA+

`kernel/reboot.tla`:
- States: per-cap, per-magic, per-pidns, per-cmd-dispatch, per-notifier, per-machine_action.
- Properties:
  - `safety_magic_required`.
  - `safety_pidns_scoped`.
  - `safety_no_kexec_without_staged_image`.
  - `safety_notifier_before_action`.
  - `liveness_cad_returns`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_reboot` post: CAD_ON/_OFF return 0; others ⟹ no return or -E* | `Reboot::do_reboot` |
| `handle_pidns_reboot` post: SIGHUP/SIGINT to pidns init | `Reboot::handle_pidns_reboot` |
| `machine_restart` post: notifier chain ran | `Reboot::do_reboot` |
| `kernel_kexec` post: staged kimage entered | `Reboot::do_reboot` |

### Layer 4: Verus / Creusot functional

Per-`reboot(2)` man-page and per-`kernel/reboot.c` semantic equivalence. LTP `reboot01..04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`reboot(2)` reinforcement:

- **Per-cap-check pre-magic** — defense against per-priv-escalation via magic guess.
- **Per-magic-confirm gate** — defense against per-accidental syscall.
- **Per-pidns scoping** — defense against per-container-host-reboot.
- **Per-notifier-chain orderly** — defense against per-FS-corruption on abrupt halt.
- **Per-kexec-requires-staged** — defense against per-jump-to-nothing.
- **Per-CAD configurable** — defense against per-physical-keyboard reboot in server racks.

## Grsecurity / PaX surface

- **PaX UDEREF on `arg` copy_from_user (CMD_RESTART2)** — defense against per-arg-pointer kernel-deref bug; SMAP forced on the 256-byte strncpy_from_user range.
- **CAP_SYS_BOOT strict in init_user_ns only** — child userns root denied. PID-namespace scoping ensures container root can only signal its own init, never reboot the host.
- **Reboot magic-confirm + CAP_SYS_BOOT** — both gates enforced; even with CAP_SYS_BOOT a wrong-magic call returns `-EINVAL`. Defense against accidental syscall from unprivileged code with raised cap.
- **GRKERNSEC_CHROOT_CAPS** — a chrooted task may never invoke reboot regardless of cap bag.
- **GRKERNSEC_PIDNS_REBOOT_DENY** — under hardened mode, a non-init pidns root may NOT reboot anything (even its own init); attempts return `-EPERM`. Defense against per-container-side-effect on host orchestration.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` (or higher), `CMD_KEXEC` requires the staged image to have been verified through `kexec_file_load(2)`; legacy `kexec_load(2)`-staged images cannot be entered.
- **GRKERNSEC_REBOOT_NOTIFY_LOCK** — `reboot_notifier_list` register/unregister gated; only built-in entries allowed under hardened mode; out-of-tree modules cannot inject notifier hooks.
- **PAX_USERCOPY_HARDEN on CMD_RESTART2 string** — bounded copy via whitelisted bounce buffer.
- **GRKERNSEC_AUDIT_BOOT** — every reboot() call logged with PID, comm, magic correctness, cmd, return code; failed attempts (wrong magic or no cap) audit-flagged.
- **PAX_REFCOUNT on reboot_mutex** — defense against per-double-acquire path.
- **GRKERNSEC_CTRLALTDEL** — `CMD_CAD_ON/_OFF` requires `CAP_SYS_BOOT` AND no sub-pidns context; defense against per-container Ctrl-Alt-Del trap manipulation.
- **GRKERNSEC_HIDESYM on emergency_restart path** — defense against per-SysRq-b detection by attacker mid-exploit.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against notifier-chain stack overflow.
- **NMI watchdog kept armed during reboot** — under hardened mode, watchdog disable is delayed until after notifier chain completes; defense against hang-during-shutdown.
- **GRKERNSEC_LOCKOUT after panic** — if last shutdown was a panic, the next reboot() requires an explicit `kernel.panic_unlock=1` operator action (defense against runaway-panic loop).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `machine_restart` / `machine_halt` per-arch (covered in arch Tier-3 docs).
- ACPI / EFI poweroff (covered in Tier-3 `drivers/acpi.md`).
- Hibernate `software_suspend_enter` (covered in Tier-3 `kernel/power/hibernate.md`).
- Implementation code.
