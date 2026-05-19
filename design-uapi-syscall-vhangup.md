---
title: "Tier-5 syscall: vhangup(2) — syscall 153"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`vhangup(2)` simulates a **virtual hangup** on the calling process's controlling terminal — the same action that physically losing carrier on a serial line would produce. The kernel walks the tty's session and delivers `SIGHUP` to the session leader (and `SIGCONT` to stopped children), flushes the line-discipline buffer, and revokes any open file descriptors that point at the same tty (subsequent reads return `EIO`, writes return `EIO` or zero). The syscall requires `CAP_SYS_TTY_CONFIG` because hanging up a tty disrupts every process attached to it. The classical caller is `getty(8)` (and `agetty`, `mingetty`): after a login session ends, `getty` calls `vhangup` to disconnect the previous user's processes from the tty before respawning the login prompt for the next session — preventing a previous user's daemons from reading the next user's keystrokes. Critical for: tty session-hygiene at login, kill-all-on-this-tty primitive for sysadmin recovery, SAK ("Secure Attention Key") implementation handoff to `__do_SAK`, container runtime tty re-initialization.

This Tier-5 covers the userspace ABI of syscall 153.

### Acceptance Criteria

- [ ] AC-1: `vhangup()` from non-CAP_SYS_TTY_CONFIG process: returns `-EPERM`.
- [ ] AC-2: `vhangup()` from CAP_SYS_TTY_CONFIG process with no controlling tty: returns `0`; no signals delivered.
- [ ] AC-3: `vhangup()` from CAP_SYS_TTY_CONFIG process with controlling tty: returns `0`; session leader receives SIGHUP.
- [ ] AC-4: After `vhangup()`: previous open fd of the tty: `read(fd)` returns `0` (EOF) or `-EIO`.
- [ ] AC-5: After `vhangup()`: previous open fd of the tty: `write(fd)` returns `-EIO`.
- [ ] AC-6: After `vhangup()`: previous open fd of the tty: `ioctl(fd, TCGETS)` returns `-EIO`.
- [ ] AC-7: After `vhangup()`: `open("/dev/tty", O_RDWR)` from a session-member process returns `-ENXIO`.
- [ ] AC-8: After `vhangup()`: getty respawns and `open("/dev/ttyN", O_RDWR)` from the new getty succeeds.
- [ ] AC-9: After `vhangup()`: stopped processes in session receive SIGCONT (to deliver pending SIGHUP).
- [ ] AC-10: Concurrent `vhangup()` from two CAP-SYS-TTY-CONFIG processes on same tty: both succeed; only one delivers signals (the other observes `tty->session == NULL`).
- [ ] AC-11: `vhangup()` does NOT close caller's own tty fd; `close(0)` still required.
- [ ] AC-12: `vhangup()` is synchronous: signals and fd-revoke complete before return.

### Architecture

Rookery surface in `drivers/tty/syscalls.rs`:

```rust
pub fn sys_vhangup() -> isize {
    if !current().capable(CAP_SYS_TTY_CONFIG) {
        return -EPERM as isize;
    }
    let tty = match current().signal_tty() {
        Some(t) => t,
        None    => return 0, /* no controlling tty: silent no-op */
    };
    Tty::vhangup(&tty);
    0
}
```

`Tty::vhangup(tty)`:

```rust
pub fn vhangup(tty: &TtyStruct) {
    /* Serialise concurrent vhangup */
    let _ctrl = tty.ctrl_lock.lock();
    if tty.session.is_none() {
        return; /* already hung up */
    }
    /* (a) Signal delivery */
    if let Some(sess) = tty.session.clone() {
        Signal::send_session(&sess, SIGHUP);
        Signal::send_session(&sess, SIGCONT);
    }
    /* (b) Line-discipline flush */
    if let Some(ld) = tty.ldisc() {
        ld.flush_buffer(tty);
    }
    /* (c) fd-revoke: remap all open files referencing this tty */
    FdRevoke::revoke_tty(tty);
    /* (d) Release session affiliation */
    tty.session = None;
    tty.pgrp = None;
}
```

`FdRevoke::revoke_tty(tty)`:

```rust
fn revoke_tty(tty: &TtyStruct) {
    /* Walk every task; for each open file whose f_op == TTY_FOPS and private == tty,
       remap to HUNG_UP_FOPS (returns -EIO on read/write/ioctl, 0-byte-eof on poll). */
    for task in Task::for_each() {
        let tab = task.fd_table();
        for fd in tab.iter() {
            if let Some(file) = tab.lookup(fd) {
                if file.f_op == &TTY_FOPS && file.private_tty() == Some(tty) {
                    tab.replace_ops(fd, &HUNG_UP_TTY_FOPS);
                }
            }
        }
    }
}
```

### Out of Scope

- `drivers/tty/00-overview.md` Tier-3: tty subsystem internals.
- `drivers/tty/jobctrl.md` Tier-3: session / pgrp / job control.
- `drivers/tty/ldisc.md` Tier-3: line discipline framework.
- `__do_SAK` Secure Attention Key (covered in tty session-management).
- `setsid.md` / `setpgid.md` siblings.
- `getty(8)` / `agetty(8)` userspace.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int vhangup(void);
```

Rust ABI shim:

```rust
pub fn sys_vhangup() -> isize;
```

Syscall number: **153** (x86_64). Generic syscall table: **58**.

### parameters

(none)

### return

- **Success**: `0`.
- **Failure**: `-1` and `errno`.

### errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_TTY_CONFIG`. |

(There are no other errors: the syscall does not fault, does not validate arguments, does not allocate, and silently no-ops if the caller has no controlling terminal.)

### abi surface

```text
__NR_vhangup (x86_64)  = 153
__NR_vhangup (i386)    = 111
__NR_vhangup (arm64)   = 58   (generic-syscall)
__NR_vhangup (generic) = 58

/* Effect */
- Walks current->signal->tty (controlling terminal)
- If no controlling tty: silent no-op, returns 0
- Else: tty_vhangup(tty)
  - Delivers SIGHUP to session leader
  - Delivers SIGCONT to stopped children of session
  - Flushes line-discipline (n_tty / n_hdlc / ...)
  - Closes all open file descriptors pointing at this tty (revoke)
    - Subsequent read(2) on those fds returns 0 (EOF)
    - Subsequent write(2) returns -EIO
  - Releases the tty for a new session
- Returns 0
```

### compatibility contract

REQ-1: Syscall number is **153** on x86_64, **58** on arm64/generic. ABI-stable since 1.0.

REQ-2: Capability check: `capable(CAP_SYS_TTY_CONFIG)` in caller's user-ns; lack ⟹ `-EPERM`. (Historically this was a no-cap-required syscall; the cap requirement was added in 2.0 to prevent rogue users from hanging up other users' ttys.)

REQ-3: No-controlling-tty case: `current->signal->tty == NULL` ⟹ no-op, return 0. (Per POSIX, this is not an error.)

REQ-4: With controlling tty: invoke `tty_vhangup(tty)`:
- a) Signal delivery: SIGHUP to session leader (`tty->session`), then SIGCONT to any process in the session that was stopped (to allow the SIGHUP to reach them).
- b) Line-discipline flush: `ldisc->ops->flush_buffer(tty)` if non-NULL.
- c) Revoke: every open `struct file` whose `f_op == tty_fops` and whose private data points at this `tty` has its fd-table entry remapped to a stub set of file operations that return `-EIO` on read/write/ioctl.
- d) Release tty session affiliation: `tty->session = NULL`, `tty->pgrp = NULL`.

REQ-5: This is **synchronous**: by the time `vhangup` returns, all session members have been signalled and all fds have been revoked.

REQ-6: Reentrancy: concurrent `vhangup` calls on the same tty are serialised under `tty->ctrl_lock`. The second caller sees `tty->session == NULL` after the first completes and is a no-op.

REQ-7: No effect on the underlying physical device (the serial port stays open, the pty pair stays open); only the **logical** controlling-tty relationship is severed.

REQ-8: Inherits / `execve`: not applicable (no state held by caller).

REQ-9: `vhangup` does NOT close the caller's own tty fd; it remaps the fd to the "hung-up" stub. The caller can still `close(0/1/2)` explicitly.

REQ-10: Subsequent `open("/dev/tty", O_RDWR)` from any process in the previous session returns `-ENXIO` (no controlling tty), or `-EIO`. New processes that subsequently acquire a controlling tty (via `setsid` + `open` of a tty device with `O_NOCTTY` cleared) get a fresh `tty->session`.

REQ-11: SAK companion: the kernel uses `__do_SAK(tty)` internally, which is `tty_vhangup` plus `SIGKILL` to all processes with fds open on the tty. `vhangup(2)` itself does NOT kill processes; only `__do_SAK` does.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_required` | INVARIANT | non-`CAP_SYS_TTY_CONFIG` ⟹ `-EPERM`. |
| `no_tty_is_noop` | INVARIANT | no controlling tty ⟹ return 0; no signals. |
| `sighup_delivered_to_session_leader` | INVARIANT | session leader receives SIGHUP. |
| `sigcont_after_sighup` | INVARIANT | stopped session members receive SIGCONT to allow SIGHUP delivery. |
| `fd_revoke_complete_before_return` | INVARIANT | all fds revoked before syscall returns (synchronous). |
| `concurrent_vhangup_idempotent` | INVARIANT | second concurrent call observes `session == None` and is no-op. |

### Layer 2: TLA+

`drivers/tty/vhangup.tla`:
- Variables: `tty.session`, `tty.pgrp`, `tty.ctrl_lock`, `signal_queues[pid]`, `fd_table[task][fd].ops`.
- Properties:
  - `safety_cap_check` — non-cap ⟹ no effect.
  - `safety_synchronous_revoke` — all referencing fds remapped before return.
  - `safety_session_cleared` — `tty.session == None` post.
  - `safety_no_signal_to_unrelated_tasks` — only session-members receive SIGHUP/SIGCONT.
  - `liveness_eventually_completes` — vhangup terminates in bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_vhangup` post: `current().signal_tty().session == None` if cap and tty present | `sys_vhangup` |
| `Tty::vhangup` post: ctrl_lock held during mutation | `Tty::vhangup` |
| `FdRevoke::revoke_tty` post: all matching fds replaced with HUNG_UP_TTY_FOPS | `FdRevoke::revoke_tty` |

### Layer 4: Verus/Creusot functional

Per-`vhangup(2)` man page; per-LTP `testcases/kernel/syscalls/vhangup/*`; per-`getty(8)` source (util-linux `agetty.c`); per-`Documentation/admin-guide/serial-console.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

vhangup reinforcement:

- **CAP_SYS_TTY_CONFIG strict** — defense against per-rogue-user hanging up other users' ttys (huge DoS surface in shared hosts).
- **Per-fd-revoke synchronous** — defense against post-vhangup reads of the previous session's keystrokes by a leaked fd.
- **Per-session-only signal delivery** — defense against per-cross-session signal leakage.
- **Per-`tty->ctrl_lock` serialisation** — defense against per-double-vhangup race on the same tty.
- **Per-no-controlling-tty silent no-op** — defense against per-EBADF on a benign call (POSIX requires this).
- **SIGCONT-after-SIGHUP** — defense against per-SIGHUP-stuck on stopped session member.

### grsecurity/pax-style reinforcement

- **PaX UDEREF irrelevant** — `vhangup` takes no arguments; no userspace pointer dereference.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every `vhangup` entry; defense against per-stack-spray of fd-revoke walker.
- **CAP_SYS_TTY_CONFIG strict in target user-ns** — capability checked in tty's owning user-ns (not caller's); defense against per-userns CAP-laundering. (Note: ttys live in the initial user-ns by default; cross-userns vhangup requires CAP_SYS_TTY_CONFIG in init_user_ns.)
- **GRKERNSEC_HIDESYM on `TtyStruct`, `TTY_FOPS`, `HUNG_UP_TTY_FOPS` symbols** — tty struct and fops table excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; defense against per-tty-layout reconnaissance.
- **Per-uid vhangup rate-limit** — defense against per-vhangup-flood DoS that thrashes the fd-table walker.
- **Audit every successful vhangup** — every successful `vhangup` audit-logged with caller uid, target tty, session pid; defense-in-depth observability.
- **Refuse vhangup on non-getty tty** — under hardened policy, `vhangup` only permitted on ttys that match a configured getty pattern (e.g., `/dev/tty[1-6]`, `/dev/ttyS[0-3]`); defense against per-vhangup of unrelated terminals (e.g., a sysadmin's xterm).
- **GRKERNSEC_DENYUSB-tty interaction** — paired with USB-tty plug-in restrictions; defense against per-vhangup-of-physical-USB-serial reconnaissance.
- **Refuse vhangup from setuid programs (other than getty/agetty)** — under hardened policy, only the canonical getty binary may call vhangup; defense against per-suid-vhangup-abuse.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `vhangup(2)` to `-ENOSYS`, forcing applications onto explicit `ioctl(tty, TIOCSCTTY, 0)` plus signal delivery; defense against per-legacy-API surface (although this would break `getty`, so the gate is off by default).
- **Audit fd-revoke walk cross-user** — if the fd-revoke walker encounters fds owned by users other than the calling user, audit-log; defense against per-cross-user-tty-hijack observability.
- **GRKERNSEC_PROC_USER restrict `/proc/<pid>/fd/<n>` post-vhangup** — post-revoke fd's symlink target string "(hung-up tty)" may be coarsened to "(unknown)" under hardened policy; defense against per-vhangup-state reconnaissance.
- **CAP_SYS_TTY_CONFIG strict in `__do_SAK` path** — pair with vhangup to ensure SAK and vhangup share identical cap-policy; defense against per-SAK-vs-vhangup divergence.
- **Refuse vhangup on tty with active container** — under hardened policy, vhangup on a tty being used by a container (per `/proc/<pid>/fd` matching) denied; defense against per-container-disrupt DoS.

