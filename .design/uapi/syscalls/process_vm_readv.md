# Tier-5 syscall: process_vm_readv(2) — syscall 310

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/process_vm_access.c (SYSCALL_DEFINE6(process_vm_readv), process_vm_rw, process_vm_rw_core)
  - kernel/ptrace.c (__ptrace_may_access, ptrace_may_access)
  - include/uapi/linux/uio.h (struct iovec)
  - arch/x86/entry/syscalls/syscall_64.tbl (310  common  process_vm_readv)
-->

## Summary

`process_vm_readv(2)` copies pages from another process's address space into the caller's user buffers without ptrace-attaching or stopping the target. It scatter-gathers via a local `iovec` array (caller-side destination) and a remote `iovec` array (target-side source) and transfers up to `min(sum(local), sum(remote))` bytes. It is the read half of the pair (writev is 311). Used by debuggers, profilers, crash dumpers, sandbox monitors (e.g., gdb live-attach, perf, criu, lldb-server) and any cross-process introspection that wants page-aligned bulk copies without the slow `PTRACE_PEEKDATA` word-at-a-time path. Critical for: live debugging throughput, criu image dump, observability tooling, sanitizer cross-checks.

## Signature

```c
ssize_t process_vm_readv(pid_t pid,
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

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | PID of target process (in caller's PID namespace). |
| `local_iov` | `const struct iovec *` | in | Array of caller-side destination buffers. |
| `liovcnt` | `unsigned long` | in | Count of `local_iov` entries; `0..=UIO_MAXIOV (1024)`. |
| `remote_iov` | `const struct iovec *` | in | Array of target-side source ranges. |
| `riovcnt` | `unsigned long` | in | Count of `remote_iov` entries; `0..=UIO_MAXIOV`. |
| `flags` | `unsigned long` | in | Reserved; must be `0`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes successfully read (may be short on partial-mapping). |
| `-1` + `errno` | Failure (nothing copied on hard error; partial-success returns short count). |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `liovcnt > UIO_MAXIOV`, `riovcnt > UIO_MAXIOV`, `flags != 0`, sum overflow. |
| `EFAULT` | `local_iov` / `remote_iov` userptr fault, or remote address invalid (unmapped page). |
| `ENOMEM` | Internal iovec kalloc failed. |
| `EPERM` | Caller cannot ptrace target (no `CAP_SYS_PTRACE`, distinct userns, restricted YAMA). |
| `ESRCH` | `pid` does not exist or has exited. |

## ABI surface

```text
__NR_process_vm_readv (x86_64) = 310
__NR_process_vm_readv (arm64)  = 270
__NR_process_vm_readv (riscv)  = 270
__NR_process_vm_readv (i386)   = 347

/* Each iovec.iov_len may be up to SSIZE_MAX; aggregate truncated to SSIZE_MAX. */
/* flags is reserved-for-future; pass 0. */
```

## Compatibility contract

REQ-1: Syscall number is **310** on x86_64. ABI-stable.

REQ-2: `flags` MUST be `0`; non-zero returns `-EINVAL`. This keeps the bit-field reserved for future extensions.

REQ-3: `liovcnt` and `riovcnt` MUST be `<= UIO_MAXIOV (1024)`; over-limit returns `-EINVAL`.

REQ-4: Per-iovec sum check: aggregate length is computed as i64 saturating sum; per `EINVAL` if any `iov_len` overflows ssize_t or aggregate exceeds `SSIZE_MAX`.

REQ-5: Local iovec validation walks `access_ok` per buffer; remote iovec validation is deferred to per-page `get_user_pages_remote` since target page table is consulted.

REQ-6: Permission check is `ptrace_may_access(target, PTRACE_MODE_ATTACH_REALCREDS)`:
- caller's real-uid/gid set must dominate target's saved set, OR
- caller has `CAP_SYS_PTRACE` in target's userns, OR
- YAMA `kernel.yama.ptrace_scope` policy permits.

REQ-7: Atomicity: each `remote_iov[i]` segment is iterated in `PAGE_SIZE` chunks; the syscall MAY return a short count if a later page is unmapped — the returned count is the sum of bytes successfully transferred from the start.

REQ-8: Hard `EFAULT` only when the FIRST page of remote IO is unmapped or the local-iov user-pointer itself faults; mid-IO unmap returns the partial byte count.

REQ-9: Pages are obtained via `get_user_pages_remote(target_mm, ..., FOLL_FORCE? NO)`; FOLL_FORCE is NOT set — a write-protected page that fails read is returned as the partial count. Read-only mapping in target is readable.

REQ-10: `pid` is resolved in caller's PID namespace; a target in a sibling namespace not visible to the caller returns `-ESRCH`.

REQ-11: Per-`SCHED_AUTOGROUP`/`SCHED_DEADLINE` neutral: no scheduler interaction beyond regular preemption points.

REQ-12: Per-`PR_SET_DUMPABLE`: target with `dumpable=0` (suid binary, set via setuid()) is rejected with `-EPERM` regardless of CAP_SYS_PTRACE (matches ptrace).

REQ-13: Per-namespace: caller must share or strictly own target's user-namespace ancestry for CAP_SYS_PTRACE to apply.

REQ-14: Per-audit: every call audit-logged with target pid, byte count requested, caller real-uid.

## Acceptance Criteria

- [ ] AC-1: Two-process test: parent reads child's known buffer; bytes match.
- [ ] AC-2: `flags = 1` returns `-EINVAL`.
- [ ] AC-3: `liovcnt = 2048` returns `-EINVAL`.
- [ ] AC-4: Remote pid does not exist: returns `-ESRCH`.
- [ ] AC-5: Remote pid is suid (`dumpable=0`) without CAP_SYS_PTRACE: `-EPERM`.
- [ ] AC-6: Remote iov[0] valid, iov[1] unmapped: returns sum of iov[0].iov_len (partial).
- [ ] AC-7: Remote iov[0] first page unmapped: returns `-EFAULT`.
- [ ] AC-8: Local iovec userptr faults: `-EFAULT`.
- [ ] AC-9: Aggregate length > SSIZE_MAX: `-EINVAL`.
- [ ] AC-10: Audit log records per-call entry with target pid.

## Architecture

```rust
#[syscall(nr = 310, abi = "sysv")]
pub fn sys_process_vm_readv(
    pid: i32,
    local_iov: UserPtr<Iovec>,
    liovcnt: u64,
    remote_iov: UserPtr<Iovec>,
    riovcnt: u64,
    flags: u64,
) -> isize {
    ProcessVm::do_rw(pid, local_iov, liovcnt, remote_iov, riovcnt, flags, Dir::Read)
}
```

`ProcessVm::do_rw(pid, l, lc, r, rc, flags, dir) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. if lc > UIO_MAXIOV || rc > UIO_MAXIOV { return -EINVAL; }
3. let liv = Iovec::import_user(l, lc)?;          // EFAULT/EINVAL
4. let riv = Iovec::import_user(r, rc)?;
5. let local_sum = Iovec::checked_sum(&liv)?;     // EINVAL on overflow
6. let remote_sum = Iovec::checked_sum(&riv)?;
7. let want = min(local_sum, remote_sum);
8. let target = Pid::lookup(pid)?;                // ESRCH
9. ProcessVm::permission_check(&target, dir)?;    // EPERM
10. let copied = ProcessVm::transfer(&target.mm, &liv, &riv, dir)?;
11. AuditLog::record_pvm(target.pid, want, copied, dir);
12. copied as isize

`ProcessVm::permission_check(target, _dir) -> Result<()>`:
1. if !ptrace_may_access(target, PTRACE_MODE_ATTACH_REALCREDS) { return Err(EPERM); }
2. if !get_task_mm(target) { return Err(ESRCH); }   /* dying */
3. Ok(())

`ProcessVm::transfer(target_mm, liv, riv, Read) -> Result<usize>`:
1. let mut copied = 0usize;
2. let (mut li, mut ri) = (0, 0);
3. while li < liv.len() && ri < riv.len() {
4.   let l = &liv[li]; let r = &riv[ri];
5.   let n = min(l.iov_len, r.iov_len);
6.   /* Walk page-by-page in remote */
7.   let n_done = ProcessVm::copy_page_chunk(target_mm, r.iov_base, l.iov_base, n)?;
8.   copied += n_done;
9.   if n_done < n {
10.    /* Partial: short-return */
11.    return Ok(copied);
12.  }
13.  Iovec::advance(&mut liv[li], n); if liv[li].iov_len == 0 { li += 1; }
14.  Iovec::advance(&mut riv[ri], n); if riv[ri].iov_len == 0 { ri += 1; }
15. }
16. Ok(copied)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_reserved_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `iovcnt_bounds` | INVARIANT | liovcnt, riovcnt ≤ UIO_MAXIOV. |
| `aggregate_no_overflow` | INVARIANT | sum(iov_len) ≤ SSIZE_MAX. |
| `partial_short_count` | INVARIANT | mid-IO fault ⟹ short count not EFAULT. |
| `cap_check_before_work` | INVARIANT | permission fails ⟹ no remote access. |
| `dumpable_strict` | INVARIANT | target dumpable=0 ⟹ EPERM regardless of CAP. |

### Layer 2: TLA+

`mm/process-vm-readv.tla`:
- States: per-arg-validate, per-import-iovec, per-permission, per-transfer, per-page-walk, per-return.
- Properties:
  - `safety_no_read_without_permission` — never read remote without ptrace_may_access.
  - `safety_partial_returns_count` — mid-IO unmap returns sum-so-far, not EFAULT.
  - `safety_flags_zero` — flags != 0 ⟹ EINVAL pre-permission-check.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_rw` post: copied ≤ min(local_sum, remote_sum) | `ProcessVm::do_rw` |
| `permission_check` post: ok ⟹ ptrace_may_access true | `ProcessVm::permission_check` |
| `transfer` post: bytes_copied prefix of requested | `ProcessVm::transfer` |
| `import_user` post: kernel-copy reflects user iovec | `Iovec::import_user` |

### Layer 4: Verus / Creusot functional

Per-`process_vm_readv(2)` man-page semantic equivalence. LTP tests `process_vm_readv01..04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`process_vm_readv(2)` reinforcement:

- **Per-iovec aggregate ssize_t checked sum** — defense against per-overflow byte-count smuggling.
- **Per-UIO_MAXIOV bound** — defense against per-iovec-table DoS.
- **Per-dumpable=0 strict deny** — defense against per-suid-leak.
- **Per-namespace ptrace_may_access** — defense against per-userns-priv-escape.
- **Per-first-page EFAULT vs mid-IO short-count strict** — defense against per-ambiguous-return.
- **Per-audit record per-call** — defense against per-silent-introspection.

## Grsecurity / PaX surface

- **PaX UDEREF on `local_iov`/`remote_iov` user-pointer copy** — defense against per-iovec-array kernel-deref bug; SMAP forced.
- **GRKERNSEC_HARDEN_PTRACE applies to process_vm_*** — gating reused for this syscall: requires same-uid or `CAP_SYS_PTRACE` in init_user_ns; sibling-userns CAP_SYS_PTRACE refused.
- **GRKERNSEC_PROC_GETPID restriction** — target pid only readable if visible per /proc; targets hidden by grsec_pid_restrict are returned as `-ESRCH`, not `-EPERM` (avoids existence-leak).
- **PAX_USERCOPY_HARDEN on local-buffer copy_to_user** — bounded into whitelisted slab/page range; rejects oversized iov_len mapped to wrong slab cache.
- **GRKERNSEC_DENY_PVM toggle** — global off-switch that returns `-EPERM` to every process_vm_readv/writev call; enabled for hardened container hosts.
- **CAP_SYS_PTRACE strict in target userns** — root-in-userns refused unless the userns ancestry chain includes the target.
- **PAX_REFCOUNT on target mm_users refcount** — defense against per-mm-refcount overflow UAF during long-running multi-iov read.
- **GRKERNSEC_AUDIT_PVM** — kernel.audit emits ANOM_PROCESS_VM record for every successful call with caller real-uid + target pid + byte count.
- **PaX KERNEXEC on get_user_pages_remote path** — defense against per-W^X violation; ensures remote page-table walk runs in W^X kernel text.
- **PAX_RAP on kallsyms** — function-pointer indirect call into `process_vm_rw_core` is RAP-tagged; defense against per-ROP gadget.
- **GRKERNSEC_LOG_RATE_LIMIT on EPERM** — defense against per-log-flood via repeated denied access attempts.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-`get_user_pages_remote` slow path (covered in Tier-3 `mm/gup.md`).
- Per-ptrace permission model details (covered in Tier-3 `kernel/ptrace.md`).
- Per-namespace pid resolution (covered in Tier-3 `kernel/pid_namespace.md`).
- Implementation code.
