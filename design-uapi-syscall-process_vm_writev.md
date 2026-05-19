---
title: "Tier-5 syscall: process_vm_writev(2) — syscall 311"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`process_vm_writev(2)` is the write counterpart to `process_vm_readv(2)`: it copies bytes from the caller's `local_iov` scatter list into another process's address space at addresses described by `remote_iov`, without ptrace-attaching or stopping the target. Used by debuggers patching live variables, sandbox supervisors injecting fix-up state, criu restore for memory regions, and ptrace-free fuzzers. Permission model is stricter than read: the target's pages must be writable for the caller's effective credentials, so `dumpable=0` targets, distinct user-namespaces, and missing CAP_SYS_PTRACE refuse the operation. Critical for: live debugger write, criu restore performance, sanitizer fault injection, container supervisor poke-data.

### Acceptance Criteria

- [ ] AC-1: Parent writes child's known buffer; child observes new bytes.
- [ ] AC-2: `flags = 1` returns `-EINVAL`.
- [ ] AC-3: `liovcnt = 2048` returns `-EINVAL`.
- [ ] AC-4: Remote pid does not exist: `-ESRCH`.
- [ ] AC-5: Remote target suid (`dumpable=0`) without CAP_SYS_PTRACE in init_userns: `-EPERM`.
- [ ] AC-6: Remote iov[0] writable, iov[1] RO-mapped: returns sum of iov[0].iov_len (partial).
- [ ] AC-7: Remote iov[0] first page invalid: returns `-EFAULT`.
- [ ] AC-8: Local userptr faults: `-EFAULT`.
- [ ] AC-9: Aggregate length > SSIZE_MAX: `-EINVAL`.
- [ ] AC-10: Write to MAP_PRIVATE anon: target observes change; CoW happens in target_mm.
- [ ] AC-11: Write to MFD_SEAL_WRITE memfd region: short count returned.
- [ ] AC-12: Audit record dir=write emitted per call.

### Architecture

```rust
#[syscall(nr = 311, abi = "sysv")]
pub fn sys_process_vm_writev(
    pid: i32,
    local_iov: UserPtr<Iovec>,
    liovcnt: u64,
    remote_iov: UserPtr<Iovec>,
    riovcnt: u64,
    flags: u64,
) -> isize {
    ProcessVm::do_rw(pid, local_iov, liovcnt, remote_iov, riovcnt, flags, Dir::Write)
}
```

`ProcessVm::do_rw(..., dir=Write) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. if lc > UIO_MAXIOV || rc > UIO_MAXIOV { return -EINVAL; }
3. let liv = Iovec::import_user(l, lc)?;
4. let riv = Iovec::import_user(r, rc)?;
5. let local_sum = Iovec::checked_sum(&liv)?;
6. let remote_sum = Iovec::checked_sum(&riv)?;
7. let want = min(local_sum, remote_sum);
8. let target = Pid::lookup(pid)?;
9. ProcessVm::permission_check(&target, Dir::Write)?;
10. let copied = ProcessVm::transfer(&target.mm, &liv, &riv, Dir::Write)?;
11. AuditLog::record_pvm(target.pid, want, copied, Dir::Write);
12. copied as isize

`ProcessVm::permission_check(target, Write) -> Result<()>`:
1. if !ptrace_may_access(target, PTRACE_MODE_ATTACH_REALCREDS) { return Err(EPERM); }
2. if target.mm.flags & MMF_DUMPABLE == 0 && !init_userns_cap_sys_ptrace() { return Err(EPERM); }
3. if !get_task_mm(target) { return Err(ESRCH); }
4. Ok(())

`ProcessVm::transfer(target_mm, liv, riv, Write) -> Result<usize>`:
1. let mut copied = 0usize;
2. for chunk over (liv, riv) iter:
3.   let pages = get_user_pages_remote(target_mm, r.iov_base, n_pages, FOLL_WRITE)?;
4.   if pages.len() == 0 && copied == 0 { return Err(EFAULT); }
5.   if pages.len() == 0 { return Ok(copied); }
6.   let n_done = ProcessVm::copy_to_remote_pages(&pages, &liv, n)?;
7.   copied += n_done;
8.   if n_done < n { return Ok(copied); }
9. Ok(copied)

### Out of Scope

- Per-`get_user_pages_remote(FOLL_WRITE)` CoW path (covered in Tier-3 `mm/gup.md`).
- Per-ptrace permission model (covered in Tier-3 `kernel/ptrace.md`).
- Per-MFD_SEAL_WRITE memfd path (covered in Tier-3 `mm/memfd.md`).
- Implementation code.

### signature

```c
ssize_t process_vm_writev(pid_t pid,
                          const struct iovec *local_iov,
                          unsigned long liovcnt,
                          const struct iovec *remote_iov,
                          unsigned long riovcnt,
                          unsigned long flags);
```

```c
struct iovec {
    void  *iov_base;
    size_t iov_len;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | PID of target process (in caller's PID namespace). |
| `local_iov` | `const struct iovec *` | in | Array of caller-side source buffers. |
| `liovcnt` | `unsigned long` | in | Count of `local_iov` entries; `0..=UIO_MAXIOV (1024)`. |
| `remote_iov` | `const struct iovec *` | in | Array of target-side destination ranges. |
| `riovcnt` | `unsigned long` | in | Count of `remote_iov` entries; `0..=UIO_MAXIOV`. |
| `flags` | `unsigned long` | in | Reserved; must be `0`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes successfully written (may be short on partial-mapping or RO page). |
| `-1` + `errno` | Failure (nothing written on hard error; partial-success returns short count). |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `liovcnt > UIO_MAXIOV`, `riovcnt > UIO_MAXIOV`, `flags != 0`, sum overflow. |
| `EFAULT` | Local userptr fault, or remote target's first destination page invalid. |
| `ENOMEM` | Internal iovec kalloc failed; or write requires CoW that cannot allocate. |
| `EPERM` | Caller cannot ptrace target (no `CAP_SYS_PTRACE`, distinct userns, YAMA restrict, dumpable=0). |
| `ESRCH` | `pid` does not exist or has exited. |

### abi surface

```text
__NR_process_vm_writev (x86_64) = 311
__NR_process_vm_writev (arm64)  = 271
__NR_process_vm_writev (riscv)  = 271
__NR_process_vm_writev (i386)   = 348

/* Each iovec.iov_len up to SSIZE_MAX; aggregate truncated to SSIZE_MAX. */
/* flags reserved; pass 0. */
/* Remote write may trigger CoW on target shared pages (e.g., glibc text). */
```

### compatibility contract

REQ-1: Syscall number is **311** on x86_64. ABI-stable.

REQ-2: `flags` MUST be `0`; non-zero returns `-EINVAL`.

REQ-3: `liovcnt`, `riovcnt` MUST be `<= UIO_MAXIOV (1024)`; otherwise `-EINVAL`.

REQ-4: Per-iovec aggregate check: i64 saturating sum; over `SSIZE_MAX` returns `-EINVAL`.

REQ-5: Permission check `ptrace_may_access(target, PTRACE_MODE_ATTACH_REALCREDS)`; identical model to `process_vm_readv` but enforces target's `dumpable != 0` strictly.

REQ-6: Write goes through `get_user_pages_remote` with `FOLL_WRITE`; a non-writable target page (RO mapping, sealed memfd, RO file mmap) returns short count on subsequent iov entries.

REQ-7: Per-page CoW: writing to a target shared page (private MAP_PRIVATE) triggers per-page-fault CoW in target_mm under mmap_read_lock; if `__alloc_pages` fails, returns `-ENOMEM` or short count.

REQ-8: Atomicity: each iov segment iterated `PAGE_SIZE` chunks; short return on mid-IO fault; hard `EFAULT` only when the FIRST remote page is invalid.

REQ-9: `pid` resolved in caller's PID namespace; not-visible target returns `-ESRCH`.

REQ-10: Per-`PR_SET_DUMPABLE`: target with `dumpable=0` returns `-EPERM` even with `CAP_SYS_PTRACE` in distinct userns.

REQ-11: Per-`MFD_SEAL_WRITE` / `F_SEAL_WRITE`: sealed shared pages refuse remote write; returns short count.

REQ-12: Per-CAP_SYS_PTRACE: required in target's user namespace; sibling userns CAP refused.

REQ-13: Per-audit: every call emits audit record with target pid, byte count requested, caller real-uid, dir=write.

REQ-14: Per-anonymous-vs-file: writes to MAP_PRIVATE anonymous pages CoW; writes to MAP_SHARED file pages write-back; writes to MAP_PRIVATE file pages CoW and do NOT mark file page dirty.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_reserved_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `iovcnt_bounds` | INVARIANT | liovcnt, riovcnt ≤ UIO_MAXIOV. |
| `aggregate_no_overflow` | INVARIANT | sum(iov_len) ≤ SSIZE_MAX. |
| `foll_write_for_write` | INVARIANT | dir=Write ⟹ FOLL_WRITE set. |
| `dumpable_strict_write` | INVARIANT | dumpable=0 ⟹ EPERM (write tighter than read). |
| `cap_check_before_work` | INVARIANT | permission denied ⟹ no remote write. |

### Layer 2: TLA+

`mm/process-vm-writev.tla`:
- States: per-arg-validate, per-permission, per-page-walk (with FOLL_WRITE), per-CoW, per-return.
- Properties:
  - `safety_no_write_without_permission` — never write remote without ptrace_may_access.
  - `safety_no_write_to_sealed` — MFD_SEAL_WRITE pages never modified.
  - `safety_partial_returns_count` — mid-IO RO page returns sum-so-far.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_rw` post: copied ≤ min(local_sum, remote_sum) | `ProcessVm::do_rw` |
| `permission_check` post (Write): dumpable=1 OR init-userns CAP | `ProcessVm::permission_check` |
| `transfer` post (Write): FOLL_WRITE used; CoW respects target_mm | `ProcessVm::transfer` |

### Layer 4: Verus / Creusot functional

Per-`process_vm_writev(2)` man-page semantic equivalence. LTP `process_vm_writev01..04` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`process_vm_writev(2)` reinforcement:

- **Per-iovec aggregate ssize_t checked sum** — defense against per-overflow byte-count smuggling.
- **Per-UIO_MAXIOV bound** — defense against per-iovec-table DoS.
- **Per-dumpable=0 strict deny on write** — defense against per-suid memory corruption.
- **Per-FOLL_WRITE strict** — defense against per-RO-page silent write.
- **Per-namespace ptrace_may_access** — defense against per-userns priv escape.
- **Per-MFD_SEAL_WRITE honored** — defense against per-sealed-memfd tamper.
- **Per-audit record per-call** — defense against per-silent memory injection.

### grsecurity / pax surface

- **PaX UDEREF on `local_iov`/`remote_iov` copy** — defense against per-iovec-array kernel-deref; SMAP forced.
- **GRKERNSEC_HARDEN_PTRACE strict for write** — write side requires same-uid OR `CAP_SYS_PTRACE` in init_user_ns AND target dumpable=1; sibling-userns CAP refused; root-in-non-init refused even with CAP.
- **GRKERNSEC_DENY_PVM toggle** — global off-switch returns `-EPERM` to every process_vm_readv/writev; recommended for hardened container hosts and minijail-style sandboxes.
- **PAX_USERCOPY_HARDEN on copy_from_user from local-iov** — bounded into whitelisted slab; rejects oversized iov_len mapped to wrong slab.
- **PaX KERNEXEC on get_user_pages_remote FOLL_WRITE path** — defense against per-W^X violation while CoWing target page.
- **CAP_SYS_PTRACE strict in target userns** — root-in-userns refused unless userns ancestry includes target.
- **PAX_REFCOUNT on target mm_users** — defense against per-mm refcount overflow during long-running multi-iov write.
- **GRKERNSEC_AUDIT_PVM (dir=write)** — kernel.audit emits ANOM_PROCESS_VM_WRITE record per call.
- **GRKERNSEC_PROC_GETPID consistency** — invisible-target returns `-ESRCH` (not EPERM) to avoid existence-leak via timing.
- **PaX MPROTECT compliance** — refuses to elevate an RX-only mapping in target to RW; respects PaX_NOMPROTECT.
- **GRKERNSEC_LOG_RATE_LIMIT on EPERM** — defense against per-log-flood via repeated denied write attempts.

