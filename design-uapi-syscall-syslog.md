---
title: "Tier-5 syscall: syslog(2) — syscall 103"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`syslog(2)` (kernel `do_syslog`; not to be confused with the userspace `syslog(3)` libc routine) is the multiplexed interface to the kernel's printk ring buffer. Via a `type` argument, callers can: read pending kernel log lines (destructive or non-destructive), clear the ring, change the console log-level, or query buffer sizes. Most operations require `CAP_SYSLOG` (or `CAP_SYS_ADMIN` legacy fallback) and additionally honor `kernel.dmesg_restrict` sysctl, which is the canonical lockdown for kernel-log info-leak prevention.

Critical for: `dmesg(1)`, klogd, journald's `/dev/kmsg` consumer pair, container-aware log readers, security auditors.

### Acceptance Criteria

- [ ] AC-1: Root: `syslog(SYSLOG_ACTION_READ_ALL, buf, 1024)` returns bytes copied (≤ 1024).
- [ ] AC-2: Unprivileged with dmesg_restrict=1: returns -EPERM.
- [ ] AC-3: `syslog(99, NULL, 0)`: -EINVAL.
- [ ] AC-4: `syslog(SYSLOG_ACTION_CONSOLE_LEVEL, NULL, 0)`: -EINVAL (level out of range).
- [ ] AC-5: `syslog(SYSLOG_ACTION_CONSOLE_LEVEL, NULL, 4)`: 0; kernel.printk console_level == 4.
- [ ] AC-6: `syslog(SYSLOG_ACTION_SIZE_BUFFER, NULL, 0)`: returns `__LOG_BUF_LEN`.
- [ ] AC-7: `syslog(SYSLOG_ACTION_CLEAR, NULL, 0)` as root: 0; subsequent READ_ALL returns 0 bytes.
- [ ] AC-8: `syslog(SYSLOG_ACTION_READ, buf, 256)` with empty ring blocks; signal interrupts ⟹ -EINTR.
- [ ] AC-9: `buf = NULL` for READ: -EFAULT.
- [ ] AC-10: Non-init userns root: -EPERM (no CAP_SYSLOG in init_ns).

### Architecture

```rust
#[syscall(nr = 103, abi = "sysv")]
pub fn sys_syslog(ty: i32, buf: UserPtr<u8>, len: i32) -> isize {
    Syslog::do_syslog(ty, buf, len)
}
```

`Syslog::do_syslog(ty, buf, len) -> isize`:
1. let ty = SyslogAction::try_from(ty).map_err(|_| EINVAL)?;
2. Syslog::check_perm(ty)?;
3. match ty {
4.   Close | Open => 0,
5.   Read => Syslog::read_blocking(buf, len),
6.   ReadAll => Syslog::read_all(buf, len),
7.   ReadClear => Syslog::read_clear(buf, len),
8.   Clear => Syslog::clear(),
9.   ConsoleOff => Syslog::console_off(),
10.  ConsoleOn => Syslog::console_on(),
11.  ConsoleLevel => Syslog::console_level(len),
12.  SizeUnread => Syslog::size_unread() as isize,
13.  SizeBuffer => Syslog::size_buffer() as isize,
14. }

`Syslog::check_perm(ty) -> Result<()>`:
1. match ty {
2.   Close | Open => Ok(()),
3.   ReadAll | SizeUnread | SizeBuffer => {
4.     if dmesg_restrict() && !capable(CAP_SYSLOG) { return Err(EPERM); }
5.     Ok(())
6.   }
7.   _ => {
8.     if !capable(CAP_SYSLOG) && !capable(CAP_SYS_ADMIN) { return Err(EPERM); }
9.     Ok(())
10.  }
11. }

`Syslog::read_blocking(buf, len) -> isize`:
1. if len < 0 || (len > 0 && !buf.access_ok(len as usize)) { return -EFAULT; }
2. let mut cursor = Syslog::current_cursor();
3. loop {
4.   let n = Syslog::ring_read_from(&mut cursor, buf, len as usize);
5.   if n > 0 { return n as isize; }
6.   if signal_pending(current()) { return -EINTR; }
7.   wait_for_log()?;
8. }

### Out of Scope

- `/dev/kmsg` fd interface (covered separately in printk Tier-3).
- printk ring internals (covered in `printk.md`).
- libc `syslog(3)` userspace wrapper (unrelated; talks to `/dev/log`).
- Implementation code.

### signature

```c
int syslog(int type, char *buf, int len);
```

```c
#define SYSLOG_ACTION_CLOSE           0
#define SYSLOG_ACTION_OPEN            1
#define SYSLOG_ACTION_READ            2
#define SYSLOG_ACTION_READ_ALL        3
#define SYSLOG_ACTION_READ_CLEAR      4
#define SYSLOG_ACTION_CLEAR           5
#define SYSLOG_ACTION_CONSOLE_OFF     6
#define SYSLOG_ACTION_CONSOLE_ON      7
#define SYSLOG_ACTION_CONSOLE_LEVEL   8
#define SYSLOG_ACTION_SIZE_UNREAD     9
#define SYSLOG_ACTION_SIZE_BUFFER    10
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `type` | `int` | in | One of `SYSLOG_ACTION_*`. |
| `buf` | `char *` | in/out | User buffer (READ variants write to it; CONSOLE_LEVEL ignores). |
| `len` | `int` | in | Buffer length for READ; level value for CONSOLE_LEVEL; ignored otherwise. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | READ variants: bytes copied. SIZE_UNREAD / SIZE_BUFFER: byte count. Others: 0. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Unknown `type`; bad `len` for READ; bad level for CONSOLE_LEVEL. |
| `EPERM` | Missing `CAP_SYSLOG` (or legacy `CAP_SYS_ADMIN`); `dmesg_restrict=1` and unprivileged. |
| `EFAULT` | `buf` not writable for READ types. |
| `ERESTARTSYS` | Interrupted by signal during blocking READ. |

### abi surface

```text
__NR_syslog  (x86_64) = 103
__NR_syslog  (arm64)  = 116
__NR_syslog  (riscv)  = 116
__NR_syslog  (i386)   = 103

/* /dev/kmsg is the modern fd-based interface; syslog(2) is legacy but supported. */
```

### compatibility contract

REQ-1: Syscall number is **103** on x86_64. ABI-stable.

REQ-2: Capability gating per type:

| Type | Required cap |
|---|---|
| SYSLOG_ACTION_CLOSE (0) | none (legacy klogctl(0) no-op) |
| SYSLOG_ACTION_OPEN (1)  | none (legacy klogctl(1) no-op) |
| SYSLOG_ACTION_READ (2)  | `CAP_SYSLOG` (waits for new lines) |
| SYSLOG_ACTION_READ_ALL (3) | `CAP_SYSLOG` OR (`!dmesg_restrict`) |
| SYSLOG_ACTION_READ_CLEAR (4) | `CAP_SYSLOG` |
| SYSLOG_ACTION_CLEAR (5) | `CAP_SYSLOG` |
| SYSLOG_ACTION_CONSOLE_OFF/ON (6/7) | `CAP_SYS_ADMIN` (legacy) or `CAP_SYSLOG` |
| SYSLOG_ACTION_CONSOLE_LEVEL (8) | `CAP_SYS_ADMIN` or `CAP_SYSLOG` |
| SYSLOG_ACTION_SIZE_UNREAD (9) | `CAP_SYSLOG` OR (`!dmesg_restrict`) |
| SYSLOG_ACTION_SIZE_BUFFER (10) | `CAP_SYSLOG` OR (`!dmesg_restrict`) |

REQ-3: `kernel.dmesg_restrict` (sysctl):
- 0 (default upstream, hardened distros override): unprivileged users may READ_ALL/SIZE_*.
- 1: unprivileged users denied with `-EPERM`; only `CAP_SYSLOG` callers permitted.

REQ-4: `kernel.printk` (sysctl): default console_level, default_message_level, minimum_console_level, default_console_loglevel.

REQ-5: SYSLOG_ACTION_READ (blocking): if `len > 0`, read up to `len` bytes from the ring's read-cursor; wait if empty (signal-interruptible).

REQ-6: SYSLOG_ACTION_READ_ALL: snapshot of last `min(len, ring_size)` bytes; non-destructive.

REQ-7: SYSLOG_ACTION_READ_CLEAR: like READ_ALL then mark all consumed.

REQ-8: SYSLOG_ACTION_CLEAR: discard all current log content (does not reset the ring; just advances cursor).

REQ-9: SYSLOG_ACTION_CONSOLE_OFF: stop kernel printk to console (still buffers).

REQ-10: SYSLOG_ACTION_CONSOLE_ON: resume console printk.

REQ-11: SYSLOG_ACTION_CONSOLE_LEVEL: `len` ∈ [1, 8] sets `console_loglevel`; out-of-range ⟹ `-EINVAL`.

REQ-12: SYSLOG_ACTION_SIZE_UNREAD: bytes available to read from current cursor.

REQ-13: SYSLOG_ACTION_SIZE_BUFFER: total log buffer size (= `__LOG_BUF_LEN` at boot; default 16 KiB scaled by CONFIG_LOG_BUF_SHIFT).

REQ-14: User namespace: CAP_SYSLOG only meaningful in `init_user_ns`; non-init userns callers without init-ns capability cannot read kernel log.

REQ-15: `/dev/kmsg` (read/write) duplicates much of this functionality with per-fd cursor semantics; syslog(2) shares the same ring.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `type_dispatch_total` | INVARIANT | every SyslogAction has one arm. |
| `cap_check_before_work` | INVARIANT | type requiring CAP_SYSLOG ⟹ EPERM if missing; no log read. |
| `dmesg_restrict_gate` | INVARIANT | READ_ALL / SIZE_* gated by dmesg_restrict + cap. |
| `level_range_check` | INVARIANT | CONSOLE_LEVEL: 1 <= len <= 8 else EINVAL. |
| `efault_on_bad_buf` | INVARIANT | READ with bad buf ⟹ EFAULT. |
| `ring_no_overwrite_during_read` | INVARIANT | reader cursor advance does not race with writer. |

### Layer 2: TLA+

`kernel/syslog.tla`:
- States: per-ring head/tail, per-cap state, per-dmesg_restrict, per-cursor.
- Properties:
  - `safety_cap_gate` — protected actions require CAP_SYSLOG.
  - `safety_dmesg_restrict` — restrict mode hides log from unprivileged.
  - `safety_no_torn_read` — READ produces contiguous bytes from snapshot.
  - `liveness_blocking_read_wakes` — pending data eventually wakes reader.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_syslog` post: EINVAL on unknown type | `Syslog::do_syslog` |
| `do_syslog` post: EPERM on missing cap | `Syslog::do_syslog` |
| `read_blocking` post: returns ≤ len bytes copied | `Syslog::read_blocking` |
| `console_level` post: console_loglevel updated atomically | `Syslog::console_level` |
| `size_buffer` post: returns __LOG_BUF_LEN | `Syslog::size_buffer` |

### Layer 4: Verus/Creusot functional

Per-`syslog(2)` man page + `dmesg(1)` (klogctl) semantic equivalence. journald's kmsg consumer compatible.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`syslog(2)` reinforcement:

- **Per-type capability gate** — defense against per-unprivileged kernel-log info-leak.
- **Per-dmesg_restrict honored** — defense against per-unprivileged dmesg recon (kASLR leaks, panic dumps).
- **Per-console_level range check** — defense against per-out-of-range log spam.
- **Per-EFAULT pre-check on buf** — defense against per-bad-pointer kernel-deref bug.
- **Per-blocking-read signal interruptible** — defense against per-DoS hang.
- **Per-ring cursor monotonic** — defense against per-cursor rewind exploit.
- **Per-non-init userns rejected** — defense against per-userns-escape log access.

### grsecurity / pax surface

- **PaX UDEREF on copy_to_user for READ buf** — defense against per-buf kernel-deref bug; SMAP forced.
- **GRKERNSEC_DMESG** — restrict SYSLOG_ACTION_READ_* (READ, READ_ALL, READ_CLEAR, SIZE_UNREAD, SIZE_BUFFER) to CAP_SYSLOG even when `kernel.dmesg_restrict=0`. Effectively forces dmesg_restrict=1 at config time so that unprivileged callers cannot enumerate kernel pointers / addresses / module names from the printk ring.
- **CAP_SYSLOG strict** — grsec splits CAP_SYS_ADMIN: the legacy fallback (CAP_SYS_ADMIN as an alias for CAP_SYSLOG) is disabled; only CAP_SYSLOG in init_user_ns is honored. This narrows the privilege surface and prevents broad CAP_SYS_ADMIN holders from doubling as log readers.
- **GRKERNSEC_HIDESYM forcing kernel-pointer redaction in printk** — even with CAP_SYSLOG, `%p` printk specifiers are hashed/redacted; defense against per-log kASLR-leak.
- **PAX_USERCOPY_HARDEN on log buffer slice copy** — bounded copy from log_buf slab; whitelisted region.
- **GRKERNSEC_AUDIT_DMESG** — every syslog(2) call (other than benign CLOSE/OPEN/SIZE_BUFFER) logged via grsec_audit with uid + cap state; defense against silent log exfiltration.
- **Per-non-init userns reject CAP_SYSLOG** — defense against per-userns-escape; CAP_SYSLOG is recognised only in init_user_ns.
- **GRKERNSEC_RBAC_LOG** — when RBAC policy hides specific kernel subsystems from a role, syslog(2) READs are filtered against the role's allowed-subsystem set before copy_to_user.
- **PaX KERNEXEC on do_syslog dispatch** — vtable read-only.
- **Per-CONSOLE_LEVEL minimum-floor** — grsec_console_min_level sysctl forbids lowering console_loglevel below a configured floor to prevent silent attack-noise suppression.

