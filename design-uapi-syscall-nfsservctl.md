---
title: "Tier-5 syscall: nfsservctl(2) — syscall 180 (DEPRECATED)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`nfsservctl(2)` is a **deprecated, permanently-stubbed** syscall whose original Linux 2.2 / 2.4 role was to configure the in-kernel NFS daemon (knfsd): add/remove export entries, manage client lists, install authentication state, and control thread counts. The pre-2.6 `rpc.nfsd`, `rpc.mountd`, and `exportfs` userspace utilities called this syscall directly. In Linux 2.6 (2003) the interface was replaced by the `/proc/fs/nfsd/` and (later) `/sys/fs/nfsd/` virtual-filesystem tunables, where each knfsd setting is exposed as a writable file (`threads`, `versions`, `portlist`, `nfsv4leasetime`, `max_block_size`, `exports`, `pool_threads`, `pool_stats`, etc.). Modern `nfs-utils` (rpc.nfsd, rpc.mountd, exportfs) writes to these files; nothing uses syscall 180.

Today the slot returns `-ENOSYS` unconditionally. The syscall is documented for ABI-stability, security-audit signatures (a process invoking syscall 180 is reconnaissance or a very old NFS userspace binary), and grsec deprecated-syscall policy.

Critical for: ABI-table completeness, intrusion-detection (legacy-NFS probe), grsec deprecated-syscall audit, defence against NFS-config downgrade probing.

### Acceptance Criteria

- [ ] AC-1: `syscall(SYS_nfsservctl, NFSCTL_SVC, &arg, NULL)` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(180, NFSCTL_EXPORT, &arg, NULL)` returns `-1, ENOSYS`; export table unmodified.
- [ ] AC-3: Stub returns `-ENOSYS` regardless of whether `nfsd.ko` / knfsd is initialised.
- [ ] AC-4: Stub returns `-ENOSYS` regardless of `CAP_SYS_ADMIN`.
- [ ] AC-5: `seccomp` filter denying `nfsservctl` produces configured action.
- [ ] AC-6: `strace -e nfsservctl` traces one `nfsservctl(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 180 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-11: `/proc/fs/nfsd/*` remains the canonical interface and is unaffected.
- [ ] AC-12: Calls under all NFSCTL_* sub-commands return `-ENOSYS` identically.

### Architecture

```rust
#[syscall(nr = 180, abi = "sysv", legacy = true)]
pub fn sys_nfsservctl(
    _cmd: i32,
    _argp: UserPtr<NfsctlArg>,
    _resp: UserPtr<u8>,
) -> isize {
    SysNi::enosys("nfsservctl")
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

### Out of Scope

- knfsd modern configuration via `/proc/fs/nfsd/*` (Tier-3 `fs/nfsd/proc-interface.md`).
- knfsd implementation (Tier-3 `fs/nfsd/`).
- NFS client (`fs/nfs/`) and NFSv4 state machine.
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.

### signature

```c
/* Pre-2.6 historical prototype, retained ONLY for headers */
int nfsservctl(int cmd, struct nfsctl_arg *argp, void *resp);
```

```c
/* Modern (Linux 2.6+) effective prototype: stub */
long nfsservctl(int cmd, struct nfsctl_arg __user *argp,
                void __user *resp);  /* always -ENOSYS */
```

```c
/* Historical 'cmd' values, preserved for reference */
#define NFSCTL_SVC          0     /* start NFS service */
#define NFSCTL_ADDCLIENT    1     /* add client */
#define NFSCTL_DELCLIENT    2     /* del client */
#define NFSCTL_EXPORT       3     /* export filesystem */
#define NFSCTL_UNEXPORT     4     /* unexport filesystem */
#define NFSCTL_GETFD        6     /* get file descriptor */
#define NFSCTL_GETFS        8     /* get filesystem handle */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cmd` | `int` | in (ignored) | Historical: NFSCTL_* sub-command. |
| `argp` | `struct nfsctl_arg __user *` | in (ignored) | Historical: input arg union. |
| `resp` | `void __user *` | out (ignored) | Historical: response buffer (used by NFSCTL_GETFD/GETFS). |

All arguments are unread by the modern stub.

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. |

### abi surface

```text
__NR_nfsservctl (x86_64)    = 180    /* slot reserved, stubbed */
__NR_nfsservctl (i386)      = 169    /* slot reserved, stubbed */
__NR_nfsservctl (arm64)     = N/A    /* never assigned */
__NR_nfsservctl (riscv)     = N/A
__NR_nfsservctl (loongarch) = N/A

/* sys_ni.c:
 *     COND_SYSCALL(nfsservctl);
 */
```

### compatibility contract

REQ-1: Syscall number is **180** on x86_64; **169** on i386. Numbers reserved-forever.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT touch knfsd state, MUST NOT call into `fs/nfsd/`, MUST NOT mutate export tables, MUST NOT alter the auth-cache.

REQ-4: Historical `cmd` semantics (export/unexport/addclient/delclient/svc/getfd/getfs) MUST NOT be honoured even partially.

REQ-5: On architectures that never assigned the number, `-ENOSYS` originates at the syscall-table boundary.

REQ-6: Modern userspace MUST configure knfsd via `/proc/fs/nfsd/*` (or `/sys/fs/nfsd/*` on newer kernels), gated by `CAP_SYS_ADMIN` and filesystem permissions.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 180 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_nfsservctl` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 180 MUST NOT be reassigned.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`. The Rookery NFS server (`rookery-nfsd`) listens only on `/proc/fs/nfsd/` and `/sys/fs/nfsd/`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-call: return value is exactly `-ENOSYS`. |
| `no_knfsd_state_touch` | INVARIANT | per-stub: no `nfsd_*` function called. |
| `no_export_table_mutation` | INVARIANT | per-stub: export table unchanged. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation.
- Properties:
  - `safety_slot_180_bound_to_ni` — per-boot: slot 180 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_nfsd_state_change` — per-call: knfsd state invariant.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_nfsservctl` post: ret == -ENOSYS | `sys_nfsservctl` |
| `sys_nfsservctl` post: no observable state mutation (except audit log) | `sys_nfsservctl` |
| `sys_nfsservctl` post: knfsd export table identical pre/post | `sys_nfsservctl` |

### Layer 4: Verus / Creusot functional

Per-`nfsservctl(2)` man-page ("This system call no longer exists on current kernels.") equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`nfsservctl(2)` reinforcement:

- **Per-stub no-knfsd-state-touch** — defense against per-export-table-corruption.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale buffer pointer.
- **Per-stub no-fh-disclosure** — defense against per-NFS-file-handle leak (historical NFSCTL_GETFS returned a server-side fh).
- **Per-stub no-auth-cache-touch** — defense against per-authentication-state poisoning.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 180 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-/proc/fs/nfsd gating intact** — modern interface gated by `CAP_SYS_ADMIN` and pseudo-FS permissions.

### grsecurity / pax-style reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, `cmd` value (captured from syscall-entry tracepoint), timestamp. Rate-limited per-uid. Very high signal: legitimate NFS userspace has not used this syscall in two decades.
- **Boot-time sys_call_table verification** — slot 180 asserted `sys_ni_syscall`; mismatch panics. Rootkits that hooked nfsservctl for covert export-table modification are blocked.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller. Recommended for hardened NFS server hosts where the slot is pure attack surface.
- **GRKERNSEC_CHROOT* family** — confined processes cannot escape chroot via NFS file-handle smuggling; the legacy stub cannot bypass.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No CAP_SYS_ADMIN elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_nfsservctl` is not a distinct kallsyms entry.
- **Audit ring-buffer flood-immune** — per-uid summary at 1 Hz, not per-call.
- **knfsd-config-via-sysfs hardened** — the modern `/proc/fs/nfsd/*` and `/sys/fs/nfsd/*` paths are mode-0600 in hardened distros, owned by a dedicated `nfsd` user. The legacy slot cannot subvert this.
- **Kexec preservation** — legacy-slot binding rebuilt identically on post-kexec kernel.

