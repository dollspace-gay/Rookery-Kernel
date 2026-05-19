# Tier-5 syscall: vserver(2) — syscall 236 (NEVER IMPLEMENTED)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys_ni.c (COND_SYSCALL(vserver))
  - arch/x86/entry/syscalls/syscall_64.tbl (236  64  vserver)
  - https://linux-vserver.org/ (the out-of-tree project that never merged)
  - Documentation/admin-guide/namespaces/ (mainline isolation mechanism)
-->

## Summary

`vserver(2)` is a **never-implemented, permanently-stubbed** syscall whose slot was reserved in Linux 2.4.21+ as a placeholder for the Linux-VServer container project (Jacques Gélinas, Herbert Pötzl). Linux-VServer was an out-of-tree containerization patch set that predates LXC, OpenVZ, and Docker; it provided process / network / filesystem isolation via a "security context" identifier and a fat multiplexing syscall (subcommands for context create/migrate/destroy, capability/resource limits, network address binding, etc.). The slot was reserved in the mainline syscall table to ease userspace compatibility for distributors carrying the patch, but the patch itself never landed: mainline absorbed the use-case via namespaces (`CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWUSER`, etc.), cgroups, and capabilities.

The slot has never been bound to any kernel-side implementation in mainline. It returns `-ENOSYS` unconditionally via `COND_SYSCALL(vserver)`. The syscall is documented for ABI-stability, security-audit signatures (legitimate software in 2026 has zero reason to call this), and grsec deprecated-syscall policy.

Critical for: ABI-table completeness, intrusion-detection (rootkit looking for VServer-context elevation), grsec deprecated-syscall audit, defence against namespace-bypass probing via the reserved slot.

## Signature

```c
/* No mainline prototype — the slot was reserved but never defined.
 * Linux-VServer-patched kernels historically used:
 */
int vserver(uint32_t cmd, uint32_t id, void *data);
```

```c
/* Mainline (every Linux kernel ever) effective prototype: stub */
long vserver(unsigned int cmd, unsigned int id,
             void __user *data);  /* always -ENOSYS */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cmd` | `unsigned int` | in (ignored) | Linux-VServer historical: subcommand selector. |
| `id` | `unsigned int` | in (ignored) | Linux-VServer historical: context / xid. |
| `data` | `void __user *` | in/out (ignored) | Linux-VServer historical: cmd-specific payload. |

All arguments are unread by the stub.

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. |

There has never been a success path in mainline.

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. |

## ABI surface

```text
__NR_vserver (x86_64)    = 236    /* slot reserved, never implemented */
__NR_vserver (i386)      = 273    /* slot reserved, never implemented */
__NR_vserver (arm64)     = N/A    /* never assigned on generic-syscall */
__NR_vserver (riscv)     = N/A
__NR_vserver (loongarch) = N/A

/* sys_ni.c:
 *     COND_SYSCALL(vserver);
 */
```

## Compatibility contract

REQ-1: Syscall number is **236** on x86_64; **273** on i386. Numbers reserved-forever for ABI-stability with distributors that carried Linux-VServer.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT consult `current_cred()`, MUST NOT touch namespaces, MUST NOT examine any context-id, MUST NOT call into the cgroup subsystem.

REQ-4: Historical Linux-VServer `cmd` semantics MUST NOT be honoured even partially. The mainline namespace + cgroup + capability stack is the only legitimate isolation mechanism.

REQ-5: On architectures that never assigned the number (arm64, riscv, loongarch), `-ENOSYS` originates at the syscall-table boundary, not the stub.

REQ-6: Modern userspace MUST use `clone(2)` / `unshare(2)` with `CLONE_NEW*` flags, cgroup v2 controllers (`/sys/fs/cgroup/`), and capability bounding for container isolation. Linux-VServer userspace tools (`vserver`, `vcontext`, `vattribute`, `chxid`) are not provided.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 236 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_vserver` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 236 MUST NOT be reassigned. This is a "burned" number.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`. The Rookery isolation model is namespaces + cgroups + capabilities + (optionally) Landlock; no VServer-style context-id exists.

## Acceptance Criteria

- [ ] AC-1: `syscall(SYS_vserver, 0, 0, NULL)` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(236, 1, 42, &data)` returns `-1, ENOSYS`; data unread.
- [ ] AC-3: Stub returns `-ENOSYS` regardless of `CAP_SYS_ADMIN`.
- [ ] AC-4: Stub returns `-ENOSYS` from inside a PID/USER/NET namespace.
- [ ] AC-5: `seccomp` filter denying `vserver` produces configured action.
- [ ] AC-6: `strace -e vserver` traces one `vserver(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 236 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-11: Namespace creation via `clone(CLONE_NEWPID, ...)` continues to work normally.
- [ ] AC-12: Container runtimes (runc, podman, containerd) function with zero invocations of syscall 236.

## Architecture

```rust
#[syscall(nr = 236, abi = "sysv", legacy = true, never_implemented = true)]
pub fn sys_vserver(
    _cmd: u32,
    _id: u32,
    _data: UserPtr<u8>,
) -> isize {
    SysNi::enosys("vserver")
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
| `no_namespace_touch` | INVARIANT | per-stub: no namespace field accessed. |
| `no_cgroup_touch` | INVARIANT | per-stub: no cgroup function called. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation.
- Properties:
  - `safety_slot_236_bound_to_ni` — per-boot: slot 236 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_namespace_or_cgroup_change` — per-call: namespace + cgroup invariants.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_vserver` post: ret == -ENOSYS | `sys_vserver` |
| `sys_vserver` post: no observable state mutation (except audit log) | `sys_vserver` |
| `sys_vserver` post: namespace ids and cgroup membership identical pre/post | `sys_vserver` |

### Layer 4: Verus / Creusot functional

Per-mainline-policy: the slot has never resolved to an implementation. Equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`vserver(2)` reinforcement:

- **Per-stub no-namespace-mutation** — defense against per-context-id-bypass of mainline namespace gating.
- **Per-stub no-capability-elevation** — defense against per-cap-vector smuggling via context-id.
- **Per-stub no-cgroup-mutation** — defense against per-resource-limit bypass.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale data pointer.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 236 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-namespace + cgroup gating intact** — mainline isolation is gated by namespace creation caps and cgroup delegation; the stub cannot bypass.

## Grsecurity / PaX-style Reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, `cmd` value (captured from syscall-entry tracepoint), timestamp. Rate-limited per-uid. Extremely high signal: the slot has NEVER been implemented in mainline; any caller is anomalous.
- **Boot-time sys_call_table verification** — slot 236 asserted `sys_ni_syscall`; mismatch panics. A rootkit that binds slot 236 to a hidden VServer-style "shadow root" context-elevation primitive is blocked at boot.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller. Recommended in hardened distributions; no legitimate software has ever needed this slot.
- **GRKERNSEC_CHROOT_* family** — confined processes cannot escape chroot via VServer-context smuggling; the stub cannot bypass.
- **GRKERNSEC_NETHIDE / NETIF_HIDE** — network-namespace isolation is the only legitimate path; the stub cannot create alternate net contexts.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No CAP_SYS_ADMIN elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_vserver` is not a distinct kallsyms entry.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **Mainline namespace stack hardened** — userns creation gated by `kernel.unprivileged_userns_clone=0` in hardened defaults; this gating is the canonical isolation policy and is not bypassable via #236.
- **Kexec preservation** — legacy-slot binding rebuilt identically on post-kexec kernel.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Linux namespaces (`CLONE_NEW*`) — mainline replacement (Tier-3 `kernel/nsproxy.md`).
- cgroup v2 — mainline resource-control replacement (Tier-3 `kernel/cgroup/`).
- Capability subsystem (Tier-3 `kernel/capability.md`).
- The out-of-tree Linux-VServer patch set itself (not part of Rookery / mainline).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.
