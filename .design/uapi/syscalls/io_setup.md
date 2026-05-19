# Tier-5 syscall: io_setup(2) — syscall 206

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/aio.c (SYSCALL_DEFINE2(io_setup), ioctx_alloc, aio_setup_ring)
  - include/uapi/linux/aio_abi.h (aio_context_t, io_event, iocb)
  - include/linux/aio.h (kioctx, kioctx_table)
  - arch/x86/entry/syscalls/syscall_64.tbl (206  common  io_setup)
-->

## Summary

`io_setup(2)` creates a kernel-side asynchronous I/O context capable of holding up to `nr_events` outstanding requests, returning an opaque `aio_context_t` handle (an mmap'd ring address) usable by `io_submit(2)`, `io_getevents(2)`, `io_pgetevents(2)`, `io_cancel(2)`, and `io_destroy(2)`.

This is the POSIX AIO **kernel** (libaio / Linux native AIO) ABI, distinct from glibc's user-space POSIX AIO (thread-pool emulation in glibc) and from the newer `io_uring(7)` interface. It is still mandatory for legacy database (Oracle, MariaDB) and userspace block stacks (SPDK fallback paths). Critical for: per-process AIO ring lifecycle, RLIMIT_MEMLOCK accounting, aio_max_nr global cap.

## Signature

```c
int io_setup(unsigned int nr_events, aio_context_t *ctx_idp);
```

```c
typedef unsigned long aio_context_t;  /* opaque to userspace */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `nr_events` | `unsigned int` | in | Desired maximum number of in-flight requests; kernel may round up. Must be > 0. |
| `ctx_idp` | `aio_context_t *` | out | Receives the opaque context handle. Must be zero-initialized by caller (kernel rejects non-zero). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `*ctx_idp` populated with handle. |
| `-1` + `errno` | Failure; `*ctx_idp` unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `ctx_idp` user pointer invalid. |
| `EINVAL` | `nr_events == 0`, `*ctx_idp != 0`, or rounded size overflows. |
| `ENOMEM` | Out of memory for kioctx / ring; or RLIMIT_MEMLOCK exceeded. |
| `EAGAIN` | Global `aio_max_nr` reached (sum of nr_events across all live contexts). |
| `ENOSYS` | `CONFIG_AIO` disabled (returns -ENOSYS via stub). |

## ABI surface

```text
__NR_io_setup  (x86_64)  = 206
__NR_io_setup  (arm64)   = 0
__NR_io_setup  (riscv)   = 0
__NR_io_setup  (i386)    = 245

/* aio_context_t == unsigned long; on 32-bit ABI, 4 bytes;
   on 64-bit ABI, 8 bytes. */
/* Handle is the userspace address of the mmap'd ring (returned by
   the kernel inside ioctx_alloc via vm_mmap into current->mm). */

sysctl fs.aio-max-nr        — global maximum (default 65536).
sysctl fs.aio-nr            — current in-use count (read-only).
```

## Compatibility contract

REQ-1: Syscall number is **206** on x86_64. ABI-stable.

REQ-2: `nr_events == 0` ⟹ `-EINVAL`.

REQ-3: Caller MUST pre-initialize `*ctx_idp = 0`. Non-zero ⟹ `-EINVAL` (defense against pointer leaks).

REQ-4: Kernel rounds `nr_events` up to fit an integral number of ring slots: `nr_events = (PAGE_SIZE * pages - sizeof(aio_ring)) / sizeof(io_event)`.

REQ-5: Per-process kioctx_table tracks all contexts; default soft-limit `AIO_RING_PAGES = 8` per context (configurable via Kconfig).

REQ-6: Global `aio_nr += nr_events` atomically; if `aio_nr + nr_events > aio_max_nr` ⟹ `-EAGAIN`. Counter is decremented in `io_destroy(2)`.

REQ-7: Per-RLIMIT_MEMLOCK: ring pages are accounted against the calling task's locked-memory rlimit (because they are kernel-side pinned VM pages mapped to userspace).

REQ-8: The ring is created in the caller's mm via `vm_mmap`; the returned `aio_context_t` IS this address. Subsequent `fork(2)` does NOT inherit the kioctx (child sees the VMA but no kernel-side context; child must call its own io_setup).

REQ-9: `execve(2)` tears down all kioctx of the calling process.

REQ-10: io_setup does NOT require capability; subject only to RLIMIT_MEMLOCK and aio_max_nr.

REQ-11: Per-`CONFIG_AIO`: when disabled, syscall stubs to `-ENOSYS`.

REQ-12: Per-namespace: aio_max_nr is currently global (not per-userns); a non-init userns sees the global counter and can exhaust it.

REQ-13: io_setup is audit-logged at policy level via `auditctl -a exit,always -F arch=b64 -S io_setup` if requested by admin.

REQ-14: Race-with-fork: child's mm initially shares ring VMA via CoW; first write by either parent or child to a ring page promotes; child still has no kernel-side kioctx so its writes are kernel-invisible (kernel reads from its own pinned pages).

## Acceptance Criteria

- [ ] AC-1: `io_setup(1, &ctx)` with `ctx=0` returns 0 and `ctx != 0`.
- [ ] AC-2: `io_setup(0, &ctx)` returns `-EINVAL`.
- [ ] AC-3: `io_setup(1, NULL)` returns `-EFAULT`.
- [ ] AC-4: Caller pre-sets `ctx = 1`: returns `-EINVAL`.
- [ ] AC-5: Repeated io_setup until aio_max_nr exhausted: returns `-EAGAIN`.
- [ ] AC-6: After io_destroy: aio_nr decremented; further io_setup succeeds.
- [ ] AC-7: nr_events rounded up to fit ring slots: queried via /proc/<pid>/io shows >= requested.
- [ ] AC-8: After execve: all previously-created contexts gone (subsequent io_submit returns -EINVAL).
- [ ] AC-9: RLIMIT_MEMLOCK exceeded: returns `-ENOMEM`.
- [ ] AC-10: Fork: child does not inherit kioctx (independent io_setup required).
- [ ] AC-11: With `CONFIG_AIO=n`: io_setup returns `-ENOSYS`.

## Architecture

```rust
#[syscall(nr = 206, abi = "sysv")]
pub fn sys_io_setup(nr_events: u32, ctx_idp: UserPtr<u64>) -> isize {
    IoSetup::do_io_setup(nr_events, ctx_idp)
}
```

`IoSetup::do_io_setup(nr_events, ctx_idp) -> isize`:
1. if ctx_idp.is_null() { return Err(EFAULT); }
2. let existing = ctx_idp.copy_in::<u64>()?;
3. if existing != 0 { return Err(EINVAL); }
4. if nr_events == 0 { return Err(EINVAL); }
5. /* Round nr_events to fit ring pages. */
6. let nr = IoSetup::round_nr_events(nr_events)?;
7. /* Global aio_nr check. */
8. let aio_nr = atomic::fetch_add(&AIO_NR, nr as u64, Ordering::AcqRel);
9. if aio_nr + nr as u64 > sysctl::aio_max_nr {
10.   atomic::fetch_sub(&AIO_NR, nr as u64, Ordering::AcqRel);
11.   return Err(EAGAIN);
12. }
13. /* RLIMIT_MEMLOCK accounting. */
14. let pages = ring_pages(nr);
15. if !current().group_leader().memlock_charge(pages) {
16.   atomic::fetch_sub(&AIO_NR, nr as u64, Ordering::AcqRel);
17.   return Err(ENOMEM);
18. }
19. /* Allocate kioctx + ring. */
20. let kioctx = KioCtx::alloc(nr).map_err(|e| {
21.   current().group_leader().memlock_uncharge(pages);
22.   atomic::fetch_sub(&AIO_NR, nr as u64, Ordering::AcqRel);
23.   e
24. })?;
25. /* vm_mmap the ring into current mm. */
26. let ring_addr = current_mm().vm_mmap_aio_ring(&kioctx, pages)?;
27. kioctx.user_id = ring_addr;
28. /* Register in per-mm kioctx_table. */
29. current_mm().kioctx_table_insert(&kioctx)?;
30. /* Return handle to user. */
31. ctx_idp.copy_out(&ring_addr)?;
32. Ok(0)

`IoSetup::round_nr_events(nr) -> Result<u32>`:
1. const HDR: usize = size_of::<AioRing>();
2. let want = nr as usize * size_of::<IoEvent>() + HDR;
3. let pages = (want + PAGE_SIZE - 1) / PAGE_SIZE;
4. if pages > AIO_RING_PAGES_MAX { return Err(EINVAL); }
5. let slots = (pages * PAGE_SIZE - HDR) / size_of::<IoEvent>();
6. Ok(slots as u32)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ctx_idp_pre_zero` | INVARIANT | *ctx_idp must equal 0 on entry. |
| `nr_events_positive` | INVARIANT | nr_events > 0 ⟹ proceed. |
| `aio_nr_consistent` | INVARIANT | AIO_NR strictly tracks sum-of-live-ctx nr_events. |
| `memlock_charge_balanced` | INVARIANT | every successful alloc has matching uncharge in io_destroy. |
| `ring_pages_bounded` | INVARIANT | ring_pages <= AIO_RING_PAGES_MAX. |
| `handle_is_ring_addr` | INVARIANT | returned aio_context_t == ring user-vm address. |

### Layer 2: TLA+

`fs/aio-setup.tla`:
- States: AIO_NR counter, per-mm kioctx_table, memlock charge.
- Properties:
  - `safety_aio_nr_never_negative` — counter never wraps below zero.
  - `safety_aio_nr_le_aio_max_nr` — invariant.
  - `safety_handle_uniqueness` — each live ctx has unique handle.
  - `liveness_setup_terminates` — call terminates.
  - `safety_failure_no_leak` — failed path releases all partial state.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_io_setup` post: returns 0 ⟹ kioctx alive, mapped, accounted | `IoSetup::do_io_setup` |
| `round_nr_events` post: result fits in pages and is >= requested | `IoSetup::round_nr_events` |
| `KioCtx::alloc` post: refcount = 1, ring zeroed | `KioCtx::alloc` |

### Layer 4: Verus / Creusot functional

Per-`io_setup(2)` man-page semantic equivalence. libaio testsuite `aio-stress` passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_setup(2)` reinforcement:

- **Per-`*ctx_idp == 0` precondition** — defense against per-stale-handle reuse.
- **Per-RLIMIT_MEMLOCK accounting** — defense against per-pinned-VM DoS.
- **Per-aio_max_nr global cap** — defense against per-resource exhaustion across all processes.
- **Per-ring address randomized by ASLR** — defense against per-deterministic-leak.
- **Per-kioctx refcount strict** — defense against per-UAF on concurrent io_destroy.
- **Per-execve teardown** — defense against per-stale-ctx surviving exec.
- **Per-AIO_RING_PAGES_MAX** — defense against per-huge-ring DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on `ctx_idp` copy_in/copy_out** — defense against per-handle-pointer kernel-deref bug; SMAP enforces userspace-only deref.
- **PAX_USERCOPY_HARDEN on copy_to_user(ctx_idp)** — bounded 8-byte copy to whitelisted slab.
- **RLIMIT_MEMLOCK + aio_max_nr** — grsec enforces both per-task and global caps; even root subject to aio_max_nr (no privileged bypass).
- **GRKERNSEC_RESLOG on io_setup** — logs per-task aio context creation with nr_events and resulting handle (audit trail).
- **PAX_REFCOUNT on kioctx->users** — defense against per-refcount-overflow UAF when concurrent io_destroy + io_submit race.
- **PaX KERNEXEC on aio handler text** — no W^X violation; aio callback pages mapped RX-only.
- **Per-CAP_IPC_LOCK NOT auto-granted** — RLIMIT_MEMLOCK enforced; grsec does not waive memlock for AIO ring.
- **PAX_RANDMMAP for ring placement** — vm_mmap of ring page picks a random gap inside RW areas; handle (=address) entropy bounded by mmap_rnd_bits.
- **GRKERNSEC_PROC_GETPID** — `/proc/<pid>/maps` ring VMA visible only to owner; non-owners see hole.
- **PaX FORCED_SHRINK_AIO** — under memory pressure, idle AIO rings are reclaimed first (informational logs).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `io_submit(2)` (covered in `io_submit.md`).
- `io_getevents(2)` (covered in `io_getevents.md`).
- `io_destroy(2)` (covered separately if expanded).
- `io_cancel(2)` (covered separately if expanded).
- `io_uring(7)` setup (covered in `io_uring_setup.md`).
- Implementation code.
