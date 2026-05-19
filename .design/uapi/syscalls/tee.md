# Tier-5 syscall: tee(2) — syscall 276

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/splice.c (SYSCALL_DEFINE4(tee), do_tee)
  - include/linux/pipe_fs_i.h (pipe_inode_info, pipe_buffer)
  - include/uapi/linux/fcntl.h (SPLICE_F_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (276 common tee)
-->

## Summary

`tee(2)` duplicates pipe contents from one pipe to another without consuming the source pipe — it shares the underlying buffer pages by incrementing their refcounts. Unlike `splice(2)`, both endpoints must be pipes, and unlike `cp`-style copy, no data is copied: the destination pipe receives `pipe_buffer` entries whose `page` pointers are shared with the source. Reading either pipe drains its own ring; the underlying pages are freed when both pipes drop their last reference.

Canonical use: shell `tee(1)` analogue with zero-copy fan-out, log mirroring, tap-and-forward observability pipelines. Critical for: high-throughput data fan-out, IPC mirroring, sched_ext / BPF data-relay topologies.

## Signature

```c
ssize_t tee(int fd_in,
            int fd_out,
            size_t len,
            unsigned int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd_in` | `int` | in | Source pipe (readable end). |
| `fd_out` | `int` | in | Destination pipe (writable end). Must be a different pipe than `fd_in`. |
| `len` | `size_t` | in | Maximum bytes to duplicate. Truncated to available + capacity. |
| `flags` | `unsigned int` | in | `SPLICE_F_NONBLOCK | SPLICE_F_MORE | SPLICE_F_MOVE | SPLICE_F_GIFT`; only NONBLOCK and MORE meaningful for tee. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes duplicated (may be less than `len` for short transfer or NONBLOCK). |
| `0` | Source pipe empty AND writer end closed (EOF), or `len == 0`. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | Either fd is not a pipe; both fds refer to the same pipe; reserved flag bit set. |
| `EBADF` | fd_in not opened for read, or fd_out not opened for write. |
| `EAGAIN` | `SPLICE_F_NONBLOCK` and (source empty or destination full). |
| `ENOMEM` | Allocation for pipe_buffer entry on destination ring failed. |

## ABI surface

```text
__NR_tee  (x86_64) = 276
__NR_tee  (arm64)  = 77
__NR_tee  (riscv)  = 77
__NR_tee  (i386)   = 315
```

## Compatibility contract

REQ-1: Syscall number is **276** on x86_64. ABI-stable.

REQ-2: Both `fd_in` and `fd_out` must be pipes. Otherwise `-EINVAL`.

REQ-3: `fd_in` and `fd_out` must refer to **different** `pipe_inode_info` structures. Same-pipe tee returns `-EINVAL` (deadlock prevention).

REQ-4: `fd_in` must be open for read (`FMODE_READ`); `fd_out` must be open for write (`FMODE_WRITE`). Else `-EBADF`.

REQ-5: `flags & ~SPLICE_F_ALL` ⟹ `-EINVAL`. `SPLICE_F_MOVE`/`SPLICE_F_GIFT` are accepted but ignored (no buffer move semantics for tee).

REQ-6: Lock ordering: `pipe_double_lock(in, out)` orders by `pipe_inode_info` address to prevent deadlock against concurrent tee/splice.

REQ-7: Behavior:
- For each buffer in source ring (in head order, up to `len` bytes):
  - Allocate new `pipe_buffer` slot in destination if available.
  - Increment `page` refcount.
  - Copy `offset` and `len` fields.
  - Set destination buffer `ops = &page_cache_pipe_buf_ops` (refcount-based release).
- Return total bytes duplicated.

REQ-8: Source pipe is NOT drained; subsequent `read(fd_in)` still sees the same data.

REQ-9: Blocking semantics:
- `SPLICE_F_NONBLOCK = 0`: if source empty and any writer-end open, wait; if no writer ⟹ return 0 (EOF).
- `SPLICE_F_NONBLOCK = 1`: empty source ⟹ `-EAGAIN`; full destination ⟹ `-EAGAIN`.

REQ-10: Maximum bytes per call ≤ `min(len, destination_free_capacity, source_available_bytes)`.

REQ-11: tee fd-permission boundary: cred check at call-time verifies `fd_in` is read-permitted AND `fd_out` is write-permitted in caller's cred. Re-open inherited fds across `setuid(2)` retain capabilities the original opener had — this syscall does not re-check inode-level perms; only `FMODE_*` bits.

REQ-12: Wakes destination's read waiters on success.

## Acceptance Criteria

- [ ] AC-1: `tee(pipe_a_r, pipe_b_w, 4096, 0)` returns bytes duplicated; `read(pipe_b_r)` and `read(pipe_a_r)` both see the same data.
- [ ] AC-2: `tee(non_pipe, pipe, ...)`: -EINVAL.
- [ ] AC-3: `tee(pipe_w_end, pipe_w_end_of_same_pipe, ...)`: -EBADF (wrong direction).
- [ ] AC-4: `tee(pipe_r, pipe_w_of_SAME_pipe, ...)`: -EINVAL.
- [ ] AC-5: `tee(pipe_r, pipe_w, 0, 0)`: returns 0.
- [ ] AC-6: `flags = 0x100`: -EINVAL.
- [ ] AC-7: Source empty, writer closed: returns 0 (EOF).
- [ ] AC-8: `SPLICE_F_NONBLOCK` and source empty: -EAGAIN.
- [ ] AC-9: Destination full: tee blocks (no NONBLOCK) or returns -EAGAIN (with NONBLOCK).
- [ ] AC-10: Page refcount: after tee, `page_count(p) >= 2`; both consumers drain ⟹ count drops to 1 (and to 0 on final free).
- [ ] AC-11: Concurrent tee A→B and B→A: no deadlock (ordered locks).

## Architecture

```rust
#[syscall(nr = 276, abi = "sysv")]
pub fn sys_tee(fd_in: i32, fd_out: i32, len: u64, flags: u32) -> isize {
    Tee::do_tee(fd_in, fd_out, len as usize, flags)
}
```

`Tee::do_tee(fd_in, fd_out, len, flags) -> isize`:
1. let fin  = fd_to_file(fd_in).map_err(|_| EBADF)?;
2. let fout = fd_to_file(fd_out).map_err(|_| EBADF)?;
3. if !fin.is_pipe() || !fout.is_pipe() { return -EINVAL; }
4. if !fin.is_readable() { return -EBADF; }
5. if !fout.is_writable() { return -EBADF; }
6. if (flags & !SPLICE_F_ALL) != 0 { return -EINVAL; }
7. let pin  = fin.pipe().unwrap();
8. let pout = fout.pipe().unwrap();
9. if Arc::ptr_eq(&pin, &pout) { return -EINVAL; }
10. if len == 0 { return 0; }
11. Tee::link_pipe(pin, pout, len, flags)

`Tee::link_pipe(pin, pout, len, flags) -> isize`:
1. let (g1, g2) = pipe_double_lock(pin, pout);
2. let mut copied = 0usize;
3. loop {
4.   /* Source-empty / dest-full handling. */
5.   if pin.empty() {
6.     if pin.writers() == 0 { break; }
7.     if (flags & SPLICE_F_NONBLOCK) != 0 { copied = if copied > 0 { copied } else { return -EAGAIN; }; break; }
8.     wait_for_data(pin)?;
9.     continue;
10.  }
11.  if pout.full() {
12.    if (flags & SPLICE_F_NONBLOCK) != 0 { copied = if copied > 0 { copied } else { return -EAGAIN; }; break; }
13.    wait_for_slot(pout)?;
14.    continue;
15.  }
16.  /* Per-buffer link. */
17.  let src = pin.head_buf();
18.  let take = min(src.len, len - copied);
19.  let dst = PipeBuffer {
20.    page: src.page.clone(),     // refcount++
21.    offset: src.offset,
22.    len: take,
23.    ops: &page_cache_pipe_buf_ops,
24.    flags: 0,
25.  };
26.  pout.push(dst);
27.  copied += take;
28.  if copied >= len { break; }
29.  pin.advance_head(take);  /* logical advance NOT consume; tee preserves source */
30. }
31. wake_up_interruptible(pout.read_wait());
32. drop((g2, g1));
33. return copied as isize;

`pipe_double_lock(a, b) -> (Guard, Guard)`:
1. /* Order by pipe pointer to prevent ABBA deadlock. */
2. if (&*a as *const _) < (&*b as *const _) {
3.   (a.lock(), b.lock())
4. } else {
5.   (b.lock(), a.lock())
6. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `both_pipes_required` | INVARIANT | non-pipe fd ⟹ EINVAL. |
| `same_pipe_rejected` | INVARIANT | fd_in.pipe == fd_out.pipe ⟹ EINVAL. |
| `direction_checked` | INVARIANT | fd_in must be readable; fd_out must be writable. |
| `flags_reserved_zero` | INVARIANT | flags & ~SPLICE_F_ALL ⟹ EINVAL. |
| `page_refcount_incremented` | INVARIANT | each duplicated buffer ⟹ page_count++. |
| `source_not_drained` | INVARIANT | source pipe head position unchanged after tee. |
| `lock_order_total` | INVARIANT | pipe_double_lock always orders by address. |

### Layer 2: TLA+

`fs/tee.tla`:
- States: per-pipe ring (source, dest), per-page refcount, per-lock acquisition order.
- Properties:
  - `safety_no_deadlock` — pipe_double_lock terminates.
  - `safety_source_preserved` — source contents unchanged.
  - `safety_pages_shared_not_copied` — destination page == source page.
  - `safety_refcount_balanced` — refcount + and - balance on full drain.
  - `liveness_blocking_wakes` — blocking tee progresses on data/slot availability.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_tee` post: EINVAL when same pipe | `Tee::do_tee` |
| `do_tee` post: EBADF when wrong direction | `Tee::do_tee` |
| `link_pipe` post: copied ≤ len AND copied ≤ destination capacity | `Tee::link_pipe` |
| `link_pipe` post: destination buffers reference same pages as source | `Tee::link_pipe` |
| `link_pipe` post: source ring head index unchanged | `Tee::link_pipe` |

### Layer 4: Verus/Creusot functional

Per-`tee(2)` man page + `tools/testing/selftests/splice/` semantic equivalence. Shell `tee(1)` zero-copy equivalent passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`tee(2)` reinforcement:

- **Per-both-fds pipe check** — defense against per-non-pipe deref.
- **Per-same-pipe rejection** — defense against per-self-tee deadlock / self-aliasing.
- **Per-direction (FMODE_READ/FMODE_WRITE) check** — defense against per-wrong-end abuse.
- **Per-flags reserved-bits check** — defense against per-extension-bit smuggling.
- **Per-page refcount via Arc::clone** — defense against per-refcount-imbalance UAF.
- **Per-pipe_double_lock address-ordered** — defense against per-ABBA deadlock.
- **Per-source untouched** — defense against per-implicit-consume bug.

## Grsecurity / PaX surface

- **PaX UDEREF on syscall args** — no user pointer in this syscall, but SMAP forced.
- **tee fd-permission boundary** — grsec re-validates inode permissions at tee-time (not just FMODE_*): if the caller no longer has read on the source inode or write on the destination inode (e.g. after a chmod), tee fails with EPERM even if the fd was opened earlier. This closes the historical "stale fd retains write" class for pipe-fan-out.
- **PAX_REFCOUNT on page/pipe_buffer refcounts** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_PIPE_AUDIT** — tee across UID boundary (source fd opened by other uid via SCM_RIGHTS) logged to grsec_audit; optional GRKERNSEC_TPE_INVERT mode rejects entirely.
- **PaX KERNEXEC on pipe_buf_ops vtable** — read-only; defense against per-ops-overwrite exploit.
- **Per-cred snapshot at entry** — defense against per-cred-elevation between fd-permission-check and link_pipe.
- **GRKERNSEC_HIDESYM on pipe_inode_info pointer** — pointer not exposed via /proc/$pid/fdinfo.
- **Per-pipe-buf flags forced zero** — defense against per-flag-smuggle (Dirty-Pipe family); tee never copies source flags into destination buffer.
- **Per-non-anon-pipe tee path** — when both pipes carry the grsec "anon" tag and one was used for cross-UID SCM_RIGHTS transfer, tee is rejected unless caller has CAP_SYS_ADMIN.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `splice(2)` general (covered in `splice.md`).
- `vmsplice(2)` user-buffer-to-pipe (covered in `vmsplice.md`).
- Pipe ring resize via F_SETPIPE_SZ (covered in `fcntl.md`).
- Implementation code.
