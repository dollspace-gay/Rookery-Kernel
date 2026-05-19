---
title: "Tier-5 syscall: io_submit(2) — syscall 209"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_submit(2)` queues up to `nr` asynchronous I/O requests against a previously-created context. Each entry of `iocbpp` is a userspace pointer to a `struct iocb` describing one operation (PREAD, PWRITE, PREADV, PWRITEV, FSYNC, FDSYNC, POLL). Each iocb is copied in, validated, transformed into a kernel `aio_kiocb` and dispatched via the corresponding `fops` async-capable method.

Submission is best-effort and atomic-per-iocb: if iocb `k` fails validation, the kernel returns `k` (the number of successfully submitted prior iocbs) — not an error. Only if iocb `0` fails does the syscall return a negative errno. This per-prefix-success contract is critical for libaio's resubmission logic.

### Acceptance Criteria

- [ ] AC-1: `io_submit(ctx, 1, &p)` with valid pread iocb returns 1; subsequent `io_getevents` yields one event.
- [ ] AC-2: `io_submit(ctx, 0, ...)` returns `-EINVAL`.
- [ ] AC-3: `io_submit(invalid_ctx, 1, ...)` returns `-EINVAL`.
- [ ] AC-4: `iocb.aio_lio_opcode = 99` (unknown) at index 0: returns `-EINVAL`.
- [ ] AC-5: iocb 0 ok, iocb 1 has bad fd: returns 1, iocb 1 not queued.
- [ ] AC-6: `iocb.aio_reserved2 != 0`: returns `-EINVAL` (iocb 0) or short count.
- [ ] AC-7: `iocb.aio_key != 0`: returns `-EINVAL`.
- [ ] AC-8: Ring full at iocb 2 (after 2 success): returns 2.
- [ ] AC-9: With `IOCB_FLAG_RESFD`: eventfd is signaled on completion.
- [ ] AC-10: Negative `aio_offset` on PREAD: `-EINVAL`.
- [ ] AC-11: Unprivileged caller requests real-time IO class: `-EPERM`.
- [ ] AC-12: File f_op lacks read_iter: opcode PREAD on it: `-EINVAL` for iocb 0 / short count else.

### Architecture

```rust
#[syscall(nr = 209, abi = "sysv")]
pub fn sys_io_submit(ctx_id: u64, nr: i64, iocbpp: UserPtr<UserPtr<Iocb>>) -> isize {
    IoSubmit::do_io_submit(ctx_id, nr, iocbpp)
}
```

`IoSubmit::do_io_submit(ctx_id, nr, iocbpp) -> isize`:
1. if nr <= 0 { return Err(EINVAL); }
2. if nr > MAX_IO_SUBMIT { let nr = MAX_IO_SUBMIT; }   // soft cap
3. let kioctx = current_mm().kioctx_lookup(ctx_id).ok_or(EINVAL)?;
4. let mut i: usize = 0;
5. while (i as i64) < nr {
6.   /* Copy the per-iocb user pointer. */
7.   let iocb_uptr_raw = iocbpp.add(i).copy_in::<u64>().map_err(|e| if i == 0 { e } else { return Ok(i as isize); })?;
8.   let iocb_uptr = UserPtr::<Iocb>::from_raw(iocb_uptr_raw);
9.   match IoSubmit::submit_one(&kioctx, iocb_uptr) {
10.     Ok(()) => i += 1,
11.     Err(e) => { if i == 0 { return Err(e); } else { break; } }
12.   }
13. }
14. /* Wake any io_getevents waiter if events landed inline. */
15. kioctx.wake_waiters();
16. Ok(i as isize)

`IoSubmit::submit_one(kioctx, iocb_uptr) -> Result<()>`:
1. let kiocb_in = iocb_uptr.copy_in::<Iocb>()?;
2. if kiocb_in.aio_reserved2 != 0 { return Err(EINVAL); }
3. if kiocb_in.aio_key != 0 { return Err(EINVAL); }
4. let opcode = IocbCmd::try_from(kiocb_in.aio_lio_opcode).map_err(|_| EINVAL)?;
5. let file = current().files.fget(kiocb_in.aio_fildes).ok_or(EBADF)?;
6. /* Reserve ring slot. */
7. let slot = kioctx.reserve_slot().ok_or(EAGAIN)?;
8. /* Allocate aio_kiocb. */
9. let mut req = AioKiocb::alloc(&kioctx, &file, &kiocb_in, slot)?;
10./* Resolve eventfd resfd if requested. */
11.if (kiocb_in.aio_flags & IOCB_FLAG_RESFD) != 0 {
12.  req.eventfd = Some(eventfd_fget(kiocb_in.aio_resfd)?);
13.}
14./* Validate iopri for hardening. */
15.if (kiocb_in.aio_flags & IOCB_FLAG_IOPRIO) != 0 {
16.  IoSubmit::check_ioprio(kiocb_in.aio_reqprio)?;       // EPERM
17.}
18./* Dispatch. */
19.match opcode {
20.  IocbCmd::Pread   => aio_read(&mut req),
21.  IocbCmd::Pwrite  => aio_write(&mut req),
22.  IocbCmd::Preadv  => aio_readv(&mut req),
23.  IocbCmd::Pwritev => aio_writev(&mut req),
24.  IocbCmd::Fsync   => aio_fsync(&mut req, false),
25.  IocbCmd::Fdsync  => aio_fsync(&mut req, true),
26.  IocbCmd::Poll    => aio_poll(&mut req),
27.  IocbCmd::Noop    => { aio_complete(&mut req, 0, 0); Ok(()) },
28.}
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iocb0_error_returns_errno` | INVARIANT | iocb 0 failure ⟹ negative return. |
| `iocb_n_error_returns_count` | INVARIANT | iocb i>0 failure ⟹ Ok(i). |
| `ring_slot_reserved_pre_dispatch` | INVARIANT | dispatch only after slot reserve. |
| `fget_balanced` | INVARIANT | every fget has matching fput in completion path. |
| `aio_reserved2_zero` | INVARIANT | reserved fields must be zero. |
| `nr_positive` | INVARIANT | nr > 0 required. |
| `ctx_lookup_owned_by_mm` | INVARIANT | ctx must be in current_mm()'s kioctx_table. |

### Layer 2: TLA+

`fs/aio-submit.tla`:
- States: per-kioctx ring, per-iocb dispatch, eventfd notifications.
- Properties:
  - `safety_atomic_prefix` — successful return value k ⟹ first k iocbs queued, none after.
  - `safety_no_slot_leak` — ring slot reserved ⟹ either consumed by event or released on dispatch failure.
  - `safety_fget_fput_balance` — invariant.
  - `liveness_submit_terminates` — call returns even when ring full.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_io_submit` post: returns k ⟹ ring has k new pending entries | `IoSubmit::do_io_submit` |
| `submit_one` post: ok ⟹ slot reserved, file referenced, dispatched | `IoSubmit::submit_one` |
| `check_ioprio` post: returns Ok ⟹ class permitted by current creds | `IoSubmit::check_ioprio` |

### Layer 4: Verus / Creusot functional

Per-`io_submit(2)` man-page semantic equivalence. libaio + fio aio engine tests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_submit(2)` reinforcement:

- **Per-iocb-prefix atomic-error** — defense against per-partial-submit ambiguity.
- **Per-ring-slot reservation pre-dispatch** — defense against per-ring-overrun race.
- **Per-iocb reserved-fields zero check** — defense against per-extension-field smuggling.
- **Per-fget/fput balance** — defense against per-file UAF on cancel race.
- **Per-iopri capability check** — defense against per-realtime-class privilege escalation.
- **Per-aio_key deprecated == 0** — defense against per-old-userspace key collisions.
- **Per-MAX_IO_SUBMIT cap** — defense against per-huge-nr DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on per-iocb copy_from_user** — defense against per-iocb-pointer kernel-deref; SMAP forced; both the outer pointer array and each inner iocb pass through whitelisted slab.
- **PAX_USERCOPY_HARDEN on iocb copy** — sizeof(struct iocb) bounded, whitelisted-slab destination.
- **PAX_REFCOUNT on aio_kiocb refcount** — defense against per-refcount-overflow UAF when io_cancel races completion.
- **GRKERNSEC_RESLOG aio submit metrics** — per-task aio submit count tracked; runaway processes emit warnings.
- **RLIMIT_MEMLOCK + aio_max_nr** — total in-flight aio across system bounded; grsec rejects oversubmission even by root.
- **CAP_SYS_NICE for IOCB_FLAG_IOPRIO real-time class** — grsec enforces strict capability check even within userns.
- **PaX KERNEXEC on aio completion handler text** — completion callback pages RX-only.
- **GRKERNSEC_HIDESYM on aio fastpath** — no kallsyms leak of aio internals to userspace.
- **PaX FORCED_NOFOLLOW on async-side path resolution** — async fsync/fdsync uses direct dentry not symlink resolution.
- **Eventfd resfd cross-userns refused under hardened policy** — defense against per-userns-leak via aio eventfd.
- **Per-iocb opcode whitelist** — under GRKERNSEC_IO_RESTRICT, dangerous opcodes (POLL) require additional caps.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `io_setup(2)` (covered in `io_setup.md`).
- `io_getevents(2)` (covered in `io_getevents.md`).
- `io_cancel(2)` (covered separately if expanded).
- `io_uring_enter(2)` (covered in `io_uring_enter.md`).
- Implementation code.

### signature

```c
int io_submit(aio_context_t ctx_id, long nr, struct iocb **iocbpp);
```

```c
struct iocb {
    __u64 aio_data;            /* user-supplied opaque (returned in io_event.data) */
    __u32 aio_key;             /* deprecated, must be 0 */
    __u32 aio_rw_flags;        /* RWF_* flags */
    __u16 aio_lio_opcode;      /* IOCB_CMD_* */
    __s16 aio_reqprio;         /* IO-prio class+data */
    __u32 aio_fildes;
    __u64 aio_buf;             /* user buffer or iovec ptr */
    __u64 aio_nbytes;          /* bytes or iovec count */
    __s64 aio_offset;
    __u64 aio_reserved2;
    __u32 aio_flags;           /* IOCB_FLAG_RESFD, IOCB_FLAG_IOPRIO */
    __u32 aio_resfd;           /* eventfd to notify on completion */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ctx_id` | `aio_context_t` | in | Handle from `io_setup(2)`. |
| `nr` | `long` | in | Number of iocb pointers in `iocbpp`. Must be > 0; capped at `LONG_MAX` and at ring free-slot count. |
| `iocbpp` | `struct iocb **` | in | Array of `nr` user pointers; each points to an `iocb`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of iocbs successfully queued (may be < `nr`). |
| `-1` + `errno` | Failure submitting iocb 0; nothing queued. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `iocbpp` or one of the per-iocb pointers invalid (only iocb 0 triggers errno return). |
| `EINVAL` | `nr < 0`, `nr == 0`, invalid ctx_id, malformed iocb 0 (bad opcode, aio_reserved2 != 0, aio_key != 0). |
| `EBADF` | iocb 0's `aio_fildes` not open. |
| `EAGAIN` | Ring full at iocb 0 (no slots). |
| `ENOSYS` | `CONFIG_AIO` disabled, or opcode not supported by file's f_op. |
| `EPERM` | `aio_reqprio` requests a higher IO-prio class than allowed; no `CAP_SYS_NICE`. |

### abi surface

```text
__NR_io_submit  (x86_64)  = 209
__NR_io_submit  (arm64)   = 1
__NR_io_submit  (riscv)   = 1
__NR_io_submit  (i386)    = 248

/* IOCB_CMD_* */
#define IOCB_CMD_PREAD     0
#define IOCB_CMD_PWRITE    1
#define IOCB_CMD_FSYNC     2
#define IOCB_CMD_FDSYNC    3
#define IOCB_CMD_POLL      5
#define IOCB_CMD_NOOP      6
#define IOCB_CMD_PREADV    7
#define IOCB_CMD_PWRITEV   8

/* IOCB_FLAG_* */
#define IOCB_FLAG_RESFD     (1 << 0)   /* aio_resfd valid (eventfd notify) */
#define IOCB_FLAG_IOPRIO    (1 << 1)
```

### compatibility contract

REQ-1: Syscall number is **209** on x86_64. ABI-stable.

REQ-2: `nr <= 0` ⟹ `-EINVAL`.

REQ-3: `ctx_id` must reference a live kioctx in current mm's kioctx_table; else `-EINVAL`.

REQ-4: For each iocb `i ∈ [0, nr)`:
- Copy `iocb_user_ptr = iocbpp[i]` (one user pointer).
- Copy `kiocb_in = *iocb_user_ptr` (the struct).
- Validate: `aio_reserved2 == 0`, `aio_key == 0`, `aio_lio_opcode` known, `aio_fildes` valid open fd, `aio_rw_flags` subset of `RWF_*`.
- Allocate `aio_kiocb`, fill, dispatch to fops.

REQ-5: **Atomic-per-iocb error**: if validation/dispatch of iocb `i > 0` fails, syscall returns `i` (success count). Only iocb 0's failure surfaces as errno.

REQ-6: For PREAD/PWRITE/PREADV/PWRITEV: dispatch via `file->f_op->read_iter` / `write_iter` with `IOCB_NOWAIT|IOCB_HIPRI|IOCB_DIO_CALLER_COMP|IOCB_ALLOC` flags as applicable. The kernel uses kiocb completion callback to publish io_event into the ring.

REQ-7: For FSYNC/FDSYNC: dispatch via `file->f_op->fsync` (kernel may queue to workqueue if fsync is blocking).

REQ-8: For POLL: dispatch via `vfs_poll` and arm a wake-up; completion when event mask satisfied.

REQ-9: For IOCB_FLAG_RESFD: kernel resolves `aio_resfd` to an eventfd file; on completion, eventfd is signaled (write 1) and the io_event is also published to the ring.

REQ-10: For IOCB_FLAG_IOPRIO: `aio_reqprio` is honored only with CAP_SYS_NICE or for non-higher classes; else `-EPERM`.

REQ-11: Per-ring slot reservation: each successful submission consumes 1 ring slot; ring-full ⟹ stop (return success count).

REQ-12: Per-iocb refcounted file: `fget` on submission, `fput` on completion or cancel.

REQ-13: For O_DIRECT files: dispatch may be synchronous-inline if device supports it (NVMe poll); otherwise queued and completed asynchronously.

REQ-14: aio_buf address validated for the relevant `iov_iter` direction (read = write-to-user, write = read-from-user).

REQ-15: Per-PREAD/PWRITE: aio_offset signed; negative offsets ⟹ `-EINVAL` at validation.

