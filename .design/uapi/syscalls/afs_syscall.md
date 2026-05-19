# Tier-5 syscall: afs_syscall(2) — syscall 183 (NEVER IMPLEMENTED)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys_ni.c (COND_SYSCALL(afs_syscall))
  - arch/x86/entry/syscalls/syscall_64.tbl (183  64  afs_syscall)
  - fs/afs/ (mainline AFS client uses regular VFS, NOT this syscall)
-->

## Summary

`afs_syscall(2)` is a **never-implemented, permanently-stubbed** syscall whose slot was reserved in Linux in the late 1990s as a placeholder for a hypothetical port of the Andrew File System (AFS) multiplexing syscall from IRIX / Solaris. Those proprietary Unixes used a single `afs_syscall(operation, ...)` entry point for client-side AFS operations: cell discovery, PAG (Process Authentication Group) management, token (`SetTokens`/`GetTokens`) injection, cache parameter tuning, and pioctl-style FID operations. Linux's mainline AFS client (`fs/afs/`, originally by David Howells) was designed from the start to use the regular VFS interface plus `keyctl(2)` for credentials, so the multiplexing syscall was never needed. The slot was reserved for compatibility with vendor binaries that referenced the IRIX/Solaris number; the binding never landed.

The slot has never been bound to any kernel-side implementation in mainline. It returns `-ENOSYS` unconditionally via `COND_SYSCALL(afs_syscall)`. The syscall is documented for ABI-stability, security-audit signatures (legitimate software in 2026 has zero reason to call this), and grsec deprecated-syscall policy.

Critical for: ABI-table completeness, intrusion-detection (rootkit probing reserved slots), grsec deprecated-syscall audit, defence against AFS-token-smuggling probing via the reserved slot.

## Signature

```c
/* No mainline prototype — the slot was reserved but never defined.
 * IRIX/Solaris historical AFS multiplexer was approximately:
 */
int afs_syscall(int operation, ...);
```

```c
/* Mainline (every Linux kernel ever) effective prototype: stub */
long afs_syscall(int operation, void __user *arg1,
                 void __user *arg2, void __user *arg3,
                 void __user *arg4);  /* always -ENOSYS */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `operation` | `int` | in (ignored) | Historical: AFS sub-operation selector. |
| `arg1..arg4` | `void __user *` | in/out (ignored) | Historical: operation-specific payload. |

All arguments are unread by the stub.

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. |

## ABI surface

```text
__NR_afs_syscall (x86_64)    = 183    /* slot reserved, never implemented */
__NR_afs_syscall (i386)      = 137    /* slot reserved, never implemented */
__NR_afs_syscall (arm64)     = N/A    /* never assigned on generic-syscall */
__NR_afs_syscall (riscv)     = N/A
__NR_afs_syscall (loongarch) = N/A

/* sys_ni.c:
 *     COND_SYSCALL(afs_syscall);
 */
```

## Compatibility contract

REQ-1: Syscall number is **183** on x86_64; **137** on i386. Numbers reserved-forever to match the historical IRIX/Solaris layout.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT consult AFS state, MUST NOT call into `fs/afs/`, MUST NOT manipulate keyrings, MUST NOT touch PAG state.

REQ-4: Historical IRIX/Solaris `operation` semantics MUST NOT be honoured even partially.

REQ-5: On architectures that never assigned the number (arm64, riscv, loongarch), `-ENOSYS` originates at the syscall-table boundary.

REQ-6: Modern userspace MUST use:
- VFS operations (`open`, `read`, `write`, `stat`) for AFS file I/O.
- `keyctl(2)` for AFS token injection (key type `rxrpc` / `rxrpc_s`).
- `mount(2)` with fstype `afs` for cell mounting.
- `setpag(2)`-style PAG semantics are emulated via session keyrings.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 183 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_afs_syscall` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 183 MUST NOT be reassigned.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`. Rookery's AFS client (if compiled in) lives in `fs/afs/` and uses VFS + keyctl only.

## Acceptance Criteria

- [ ] AC-1: `syscall(SYS_afs_syscall, 0, 0, 0, 0, 0)` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(183, 1, NULL, NULL, NULL, NULL)` returns `-1, ENOSYS`; arguments unread.
- [ ] AC-3: Stub returns `-ENOSYS` regardless of capabilities.
- [ ] AC-4: AFS keyrings (`keyctl(KEYCTL_SEARCH, ...)`) continue to work normally.
- [ ] AC-5: `seccomp` filter denying `afs_syscall` produces configured action.
- [ ] AC-6: `strace -e afs_syscall` traces one `afs_syscall(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 183 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-11: AFS mounts via `mount(2)` with fstype `afs` continue to work.
- [ ] AC-12: OpenAFS userspace client, if present, does NOT call this syscall (mainline uses VFS only).

## Architecture

```rust
#[syscall(nr = 183, abi = "sysv", legacy = true, never_implemented = true)]
pub fn sys_afs_syscall(
    _operation: i32,
    _arg1: UserPtr<u8>,
    _arg2: UserPtr<u8>,
    _arg3: UserPtr<u8>,
    _arg4: UserPtr<u8>,
) -> isize {
    SysNi::enosys("afs_syscall")
}
```

`SysNi::enosys(name: &'static str) -> isize`:
1. /* Shared no-op return path */
2. #[cfg(feature = "grsec_deprecated_audit")]
3.   GrsecAudit::log_legacy_syscall(name, current_creds());
4. -ENOSYS

`GrsecAudit::log_legacy_syscall(name, creds)`:
1. let task = current();
2. let event = LegacySyscallEvent {
3.   name,
4.   pid: task.tgid_in(init_pid_ns()),
5.   uid: creds.euid,
6.   exe: task.exe_file_path(),
7.   timestamp: now_monotonic_ns(),
8. };
9. AuditQueue::enqueue(event);

`SysCallTable::verify_legacy_slots()` boot-time:
1. for (nr, _) in LEGACY_STUB_SLOTS {
2.   assert!(sys_call_table[nr] == sys_ni_syscall as *const _);
3. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-call: return value is exactly `-ENOSYS`. |
| `no_afs_state_touch` | INVARIANT | per-stub: no fs/afs/ function called. |
| `no_keyring_touch` | INVARIANT | per-stub: no keyctl operation invoked. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation.
- Properties:
  - `safety_slot_183_bound_to_ni` — per-boot: slot 183 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_afs_or_keyring_change` — per-call: AFS state + keyring invariants.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_afs_syscall` post: ret == -ENOSYS | `sys_afs_syscall` |
| `sys_afs_syscall` post: no observable state mutation (except audit log) | `sys_afs_syscall` |
| `sys_afs_syscall` post: AFS PAG state + keyrings identical pre/post | `sys_afs_syscall` |

### Layer 4: Verus / Creusot functional

Per-mainline-policy: the slot has never resolved to an implementation. Equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`afs_syscall(2)` reinforcement:

- **Per-stub no-AFS-state-touch** — defense against per-AFS-cache-poisoning bypass.
- **Per-stub no-keyring-mutation** — defense against per-token-smuggling bypass of keyctl.
- **Per-stub no-PAG-id-injection** — defense against per-process-authentication-group escalation.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale arg pointer.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 183 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-keyctl gating intact** — AFS tokens are gated by keyring permission bits; the stub cannot bypass.

## Grsecurity / PaX-style Reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, `operation` value (captured from syscall-entry tracepoint), timestamp. Rate-limited per-uid. Extremely high signal: the slot has NEVER been implemented in Linux.
- **Boot-time sys_call_table verification** — slot 183 asserted `sys_ni_syscall`; mismatch panics. A rootkit that binds slot 183 to a covert credential-injection primitive is blocked at boot.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller. Recommended in hardened distributions; no legitimate Linux software has EVER used this slot.
- **GRKERNSEC_KEY_CONFINED** — even if a future buggy patch reintroduced AFS-token injection here, key-confinement policy would still gate the keyring writes; the stub cannot bypass.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No capability elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_afs_syscall` is not a distinct kallsyms entry.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **AFS-via-VFS hardened** — `mount -t afs`, `keyctl add rxrpc`, and VFS ops are the only legitimate paths; LSM hooks fire there.
- **Kexec preservation** — legacy-slot binding rebuilt identically on post-kexec kernel.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- AFS filesystem implementation (`fs/afs/`) (Tier-3 `fs/afs/overview.md` if compiled in).
- `keyctl(2)` and rxrpc key types (Tier-5 `keyctl.md` + Tier-3 `security/keys/rxrpc.md`).
- AFS mount (`mount -t afs`) semantics.
- IRIX / Solaris AFS multiplexer (not part of Linux).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.
