# Tier-5 syscall: io_uring_setup(2) — syscall 425

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - io_uring/io_uring.c (sys_io_uring_setup)
  - include/uapi/linux/io_uring.h
  - include/linux/syscalls.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (425)
  - Documentation/userspace-api/io_uring.rst
  - man io_uring_setup(2)
-->

## Summary

`io_uring_setup(2)` is the ring-allocation entry point for the asynchronous I/O subsystem. The syscall allocates a Submission Queue (SQ), Completion Queue (CQ), Submission Queue Entries array (SQE[]), and a wait/poll/IRQ context; pins them into a kernel-resident `struct io_ring_ctx`; reflects layout offsets back into the caller's `struct io_uring_params`; and returns a new file descriptor that userspace then `mmap(2)`s to gain shared access to the ring buffers. Critical for: high-throughput async I/O (Postgres, ScyllaDB, RocksDB, nginx, libuv, glibc aio shim), fixed-buffer zero-copy, network polling, and SQPOLL kernel-thread-driven I/O. Per-`IORING_SETUP_*` flags select ring mode (default-IRQ, SQPOLL, IOPOLL, single-issuer, COOP_TASKRUN, DEFER_TASKRUN, ATTACH_WQ, CQE32/SQE128, NO_MMAP, REGISTERED_FD_ONLY, R_DISABLED).

This Tier-5 covers the userspace ABI of syscall 425; per-ring runtime is owned by the Tier-3 `io_uring/io_uring-core.md` (planned).

## Signature

```c
int io_uring_setup(u32 entries, struct io_uring_params *p);
```

Rust ABI shim (entry-point layer):

```rust
pub fn sys_io_uring_setup(entries: u32, params: *mut IoUringParams) -> isize;
```

Syscall number: **425** (generic table; arch-stable across x86_64 / arm64 / riscv64 / loongarch64).

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `entries` | `u32` | IN | requested number of SQEs; kernel rounds **up** to next power of two; capped by `IORING_MAX_ENTRIES = 32768` (or `IORING_MAX_CQ_ENTRIES = 262144` for CQ when `CQSIZE` set) |
| `p` | `struct io_uring_params *` | IN/OUT | requested flags + reflected offsets / sizes / feature bits |

`struct io_uring_params` (UAPI, fixed layout):

```c
struct io_uring_params {
    __u32 sq_entries;          /* OUT: actual SQ entries */
    __u32 cq_entries;          /* OUT: actual CQ entries (default 2 * sq_entries) */
    __u32 flags;               /* IN:  IORING_SETUP_*       */
    __u32 sq_thread_cpu;       /* IN:  for SQPOLL + SQ_AFF  */
    __u32 sq_thread_idle;      /* IN:  SQPOLL idle, ms      */
    __u32 features;            /* OUT: IORING_FEAT_*        */
    __u32 wq_fd;               /* IN:  ATTACH_WQ ring fd    */
    __u32 resv[3];             /* IN:  MUST be zero         */
    struct io_sqring_offsets sq_off;   /* OUT */
    struct io_cqring_offsets cq_off;   /* OUT */
};
```

## Return

- **Success**: non-negative `int` file descriptor; descriptor is `O_RDWR`-equivalent, supports `mmap(2)` against the offsets reflected in `p->sq_off` / `p->cq_off`, and is accepted by `io_uring_enter(2)` / `io_uring_register(2)`.
- **Failure**: `-1` with `errno` set per Errors table; on Rust internal call, return is negated errno (`-EINVAL`, …).

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `entries == 0` or `entries > IORING_MAX_ENTRIES`; unknown flag bit in `p->flags`; non-zero `p->resv[]`; mutually-exclusive flag combinations (e.g., `SQPOLL` + `IOPOLL` on a non-block-device fd; `SINGLE_ISSUER` + `SQPOLL` since SQ thread is the sole issuer; `R_DISABLED` without later `IORING_REGISTER_ENABLE_RINGS`); `wq_fd` not pointing to an existing ring when `ATTACH_WQ`; `CQSIZE` set but `p->cq_entries` < `entries` |
| `EFAULT` | `p` not in user-space-readable address range; `copy_from_user` / `copy_to_user` fails |
| `ENOMEM` | kernel cannot allocate SQ/CQ/SQE backing pages (rlimit RLIMIT_MEMLOCK exceeded; per-uid `kernel.io_uring_disabled`; node memory exhausted) |
| `EPERM` | `kernel.io_uring_disabled == 2`; or `IORING_SETUP_SQPOLL` without `CAP_SYS_NICE` on a kernel that requires it (pre-5.11 legacy); or seccomp filter denies |
| `EBUSY` | `ATTACH_WQ` target ring's wq is being torn down |
| `EOPNOTSUPP` | flag combination requested but not supported on current kernel (`CQE32`, `SQE128` on older configs) |
| `EMFILE` | per-process fd table full |
| `ENFILE` | system-wide fd table full |
| `EAGAIN` | transient: SQPOLL kthread create failed; retry |

## ABI surface

`IORING_SETUP_*` flags (`p->flags`):

| Constant | Value | Purpose |
|---|---|---|
| `IORING_SETUP_IOPOLL` | `1 << 0` | busy-poll completions (O_DIRECT block device, NVMe, polled net) |
| `IORING_SETUP_SQPOLL` | `1 << 1` | kernel SQ-poll thread; eliminates syscall-per-submit |
| `IORING_SETUP_SQ_AFF` | `1 << 2` | pin SQPOLL kthread to `sq_thread_cpu` |
| `IORING_SETUP_CQSIZE` | `1 << 3` | use `p->cq_entries` directly (else 2 * SQ size) |
| `IORING_SETUP_CLAMP` | `1 << 4` | clamp `entries` / `cq_entries` to max instead of returning EINVAL |
| `IORING_SETUP_ATTACH_WQ` | `1 << 5` | share async-worker pool with ring `p->wq_fd` |
| `IORING_SETUP_R_DISABLED` | `1 << 6` | ring starts disabled; enable via `IORING_REGISTER_ENABLE_RINGS` |
| `IORING_SETUP_SUBMIT_ALL` | `1 << 7` | don't stop submitting on first error |
| `IORING_SETUP_COOP_TASKRUN` | `1 << 8` | task work runs cooperatively on issuer return |
| `IORING_SETUP_TASKRUN_FLAG` | `1 << 9` | set `IORING_SQ_TASKRUN` in SQ flags when work pending |
| `IORING_SETUP_SQE128` | `1 << 10` | 128-byte SQEs (extended ops: NVMe passthrough) |
| `IORING_SETUP_CQE32` | `1 << 11` | 32-byte CQEs (extended completion data) |
| `IORING_SETUP_SINGLE_ISSUER` | `1 << 12` | only one thread will ever submit |
| `IORING_SETUP_DEFER_TASKRUN` | `1 << 13` | defer task work until `IORING_ENTER_GETEVENTS` (requires SINGLE_ISSUER) |
| `IORING_SETUP_NO_MMAP` | `1 << 14` | user provides ring memory in `p->cq_off` / `p->sq_off` |
| `IORING_SETUP_REGISTERED_FD_ONLY` | `1 << 15` | suppress installed-fd; return fd via `IORING_REGISTER_RING_FDS` only |
| `IORING_SETUP_NO_SQARRAY` | `1 << 16` | SQE index implicit (no separate SQ-array indirection) |
| `IORING_SETUP_HYBRID_IOPOLL` | `1 << 17` | hybrid sleep+busy for IOPOLL |

`IORING_FEAT_*` (`p->features` reflected to userspace):

| Constant | Value | Purpose |
|---|---|---|
| `IORING_FEAT_SINGLE_MMAP` | `1 << 0` | SQ+CQ in single mmap |
| `IORING_FEAT_NODROP` | `1 << 1` | CQ overflow does not drop (overflow list) |
| `IORING_FEAT_SUBMIT_STABLE` | `1 << 2` | SQE consumed before async-work begins |
| `IORING_FEAT_RW_CUR_POS` | `1 << 3` | read/write at current file offset (-1) |
| `IORING_FEAT_CUR_PERSONALITY` | `1 << 4` | issuer creds used by default |
| `IORING_FEAT_FAST_POLL` | `1 << 5` | armed-poll fast path |
| `IORING_FEAT_POLL_32BITS` | `1 << 6` | full 32-bit poll mask |
| `IORING_FEAT_SQPOLL_NONFIXED` | `1 << 7` | SQPOLL works on non-fixed files |
| `IORING_FEAT_EXT_ARG` | `1 << 8` | extended `io_uring_enter` arg |
| `IORING_FEAT_NATIVE_WORKERS` | `1 << 9` | PF_IO_WORKER native worker |
| `IORING_FEAT_RSRC_TAGS` | `1 << 10` | tagged registered resources |
| `IORING_FEAT_CQE_SKIP` | `1 << 11` | `IOSQE_CQE_SKIP_SUCCESS` |
| `IORING_FEAT_LINKED_FILE` | `1 << 12` | file slot dependency tracking |
| `IORING_FEAT_REG_REG_RING` | `1 << 13` | registered-ring registration |
| `IORING_FEAT_RECVSEND_BUNDLE` | `1 << 14` | bundle recv/send |
| `IORING_FEAT_MIN_TIMEOUT` | `1 << 15` | minimum wait time |

`struct io_sqring_offsets` / `struct io_cqring_offsets`: per-mmap offsets for `head`, `tail`, `ring_mask`, `ring_entries`, `flags`, `dropped` (SQ) / `overflow` (CQ), `array` (SQ only), `cqes` (CQ only), `user_addr` (when NO_MMAP). Layout is **stable**: userspace already-compiled binaries depend on byte offsets.

## Compatibility contract

REQ-1: `entries` validation:
- `entries == 0` → `-EINVAL`.
- `entries > IORING_MAX_ENTRIES` (32768) → `-EINVAL` unless `IORING_SETUP_CLAMP` (then clamp).
- Final SQ size = next_pow2(entries).

REQ-2: `p->flags` validation:
- Unknown bit → `-EINVAL`.
- `IORING_SETUP_SQ_AFF` requires `IORING_SETUP_SQPOLL`.
- `IORING_SETUP_DEFER_TASKRUN` requires `IORING_SETUP_SINGLE_ISSUER`.
- `IORING_SETUP_SQPOLL` is mutually exclusive with `IORING_SETUP_SINGLE_ISSUER` (SQ thread is sole issuer implicitly).
- `IORING_SETUP_HYBRID_IOPOLL` requires `IORING_SETUP_IOPOLL`.

REQ-3: `p->resv[0..3]` MUST be zero → else `-EINVAL`.

REQ-4: `p->cq_entries` (when `CQSIZE`):
- Must be ≥ `entries` (post-rounding).
- Must be ≤ `IORING_MAX_CQ_ENTRIES` (262144) unless `CLAMP`.
- Rounded up to next pow2.
- Default (no `CQSIZE`): `cq_entries = 2 * sq_entries`.

REQ-5: `IORING_SETUP_ATTACH_WQ`:
- `p->wq_fd` MUST be an open io_uring fd of the same task or a sibling permitted by credential check.
- Shares the async worker pool — does not share rings.

REQ-6: `IORING_SETUP_SQPOLL`:
- Creates a kernel thread `iou-sqp-<pid>-<idx>` with `PF_IO_WORKER`.
- `p->sq_thread_cpu` honored only if `SQ_AFF` set; else floating.
- `p->sq_thread_idle` interpreted in ms.
- Subject to `RLIMIT_NPROC` and per-cgroup `pids.max`.

REQ-7: `IORING_SETUP_R_DISABLED`:
- Ring is fully allocated but rejects `io_uring_enter` until `IORING_REGISTER_ENABLE_RINGS`.
- Permits a setup-then-restrict pattern (call `IORING_REGISTER_RESTRICTIONS` before enabling).

REQ-8: `IORING_SETUP_NO_MMAP`:
- `p->sq_off.user_addr` and `p->cq_off.user_addr` MUST contain mmaped user addresses sized exactly to ring requirements.
- Userspace owns lifetime; kernel pins.

REQ-9: `IORING_SETUP_REGISTERED_FD_ONLY`:
- Returned `fd` is `-1`; caller MUST have prior `IORING_REGISTER_RING_FDS` slot.

REQ-10: `p->features` reflected:
- All `IORING_FEAT_*` bits supported by this kernel are set.
- Userspace MUST gate behavior on returned features; absence is not an error.

REQ-11: Per-task accounting:
- Pinned ring memory charged to `RLIMIT_MEMLOCK`.
- Per-uid sysctl `kernel.io_uring_disabled`:
  - `0` = enabled for all.
  - `1` = restricted to `CAP_SYS_ADMIN` or `io_uring_group` GID.
  - `2` = disabled.

REQ-12: Returned fd lifecycle:
- `close(2)` triggers ring teardown (sync-quiesce + cancel-all in-flight).
- `dup(2)` / `fork(2)`: shared `struct file`; both holders see same ring.
- `execve(2)`: fd closed iff `O_CLOEXEC` (set by default since 5.5+).

REQ-13: Personality / credentials:
- `IORING_FEAT_CUR_PERSONALITY` always set on Rookery: SQE ops execute with the issuer's creds unless `IOSQE_PERSONALITY` references a registered cred via `IORING_REGISTER_PERSONALITY`.

REQ-14: `kernel.io_uring_disabled` sysctl is read at `io_uring_setup` time only; runtime toggling does not affect existing rings.

REQ-15: Concurrency: `io_uring_setup` is fully reentrant; multiple rings per task supported up to `RLIMIT_NOFILE`.

## Acceptance Criteria

- [ ] AC-1: `io_uring_setup(8, &p)` with `p.flags=0` returns fd ≥ 0; `p.sq_entries == 8`, `p.cq_entries == 16`.
- [ ] AC-2: `entries=0` → `-EINVAL`.
- [ ] AC-3: `entries=100000` without `CLAMP` → `-EINVAL`; with `CLAMP` → `sq_entries == 32768`.
- [ ] AC-4: Unknown flag bit (`1 << 31`) → `-EINVAL`.
- [ ] AC-5: `resv[0]=1` → `-EINVAL`.
- [ ] AC-6: `SQPOLL` without root + restricted sysctl → `-EPERM`; with permission → kthread spawned, `p.features` reflects.
- [ ] AC-7: `DEFER_TASKRUN` without `SINGLE_ISSUER` → `-EINVAL`.
- [ ] AC-8: `ATTACH_WQ` with `wq_fd` pointing to non-io_uring fd → `-EINVAL`.
- [ ] AC-9: `CQSIZE` with `cq_entries < entries` (post-round) → `-EINVAL`.
- [ ] AC-10: `R_DISABLED` ring: `io_uring_enter` returns `-EBADFD` until `REGISTER_ENABLE_RINGS`.
- [ ] AC-11: `kernel.io_uring_disabled=2` → `-EPERM`.
- [ ] AC-12: Returned fd: `mmap` at `IORING_OFF_SQ_RING`, `IORING_OFF_CQ_RING`, `IORING_OFF_SQES` succeeds; offsets match `p.sq_off` / `p.cq_off`.
- [ ] AC-13: `close(fd)` cancels all in-flight ops; subsequent ops on the ring fd return `-EBADF`.
- [ ] AC-14: `p->features` includes at minimum `SINGLE_MMAP | NODROP | SUBMIT_STABLE | CUR_PERSONALITY | NATIVE_WORKERS`.
- [ ] AC-15: Two concurrent `io_uring_setup` calls from same task succeed independently.

## Architecture

Rookery surface in `kernel/io_uring/syscall/setup.rs`:

```rust
bitflags! {
    pub struct IoUringSetupFlags: u32 {
        const IOPOLL              = 1 << 0;
        const SQPOLL              = 1 << 1;
        const SQ_AFF              = 1 << 2;
        const CQSIZE              = 1 << 3;
        const CLAMP               = 1 << 4;
        const ATTACH_WQ           = 1 << 5;
        const R_DISABLED          = 1 << 6;
        const SUBMIT_ALL          = 1 << 7;
        const COOP_TASKRUN        = 1 << 8;
        const TASKRUN_FLAG        = 1 << 9;
        const SQE128              = 1 << 10;
        const CQE32               = 1 << 11;
        const SINGLE_ISSUER       = 1 << 12;
        const DEFER_TASKRUN       = 1 << 13;
        const NO_MMAP             = 1 << 14;
        const REGISTERED_FD_ONLY  = 1 << 15;
        const NO_SQARRAY          = 1 << 16;
        const HYBRID_IOPOLL       = 1 << 17;
    }
}

pub const IORING_MAX_ENTRIES:    u32 = 32_768;
pub const IORING_MAX_CQ_ENTRIES: u32 = 262_144;

#[repr(C)]
pub struct IoUringParams {
    pub sq_entries:      u32,
    pub cq_entries:      u32,
    pub flags:           u32,
    pub sq_thread_cpu:   u32,
    pub sq_thread_idle:  u32,
    pub features:        u32,
    pub wq_fd:           u32,
    pub resv:            [u32; 3],
    pub sq_off:          IoSqringOffsets,
    pub cq_off:          IoCqringOffsets,
}
```

`IoUring::setup(entries, params_uptr) -> isize`:
1. /* copy params */
2. let mut p = copy_from_user::<IoUringParams>(params_uptr).map_err(|_| -EFAULT)?;
3. /* sysctl gate */
4. match sysctl.io_uring_disabled {
     - 2 => return -EPERM,
     - 1 if !current.has_cap(CAP_SYS_ADMIN) && !current.in_group(IO_URING_GROUP) => return -EPERM,
     - _ => (),
   }
5. /* resv must be zero */
6. if p.resv != [0; 3] { return -EINVAL; }
7. /* flag-bits known */
8. let flags = IoUringSetupFlags::from_bits(p.flags).ok_or(-EINVAL)?;
9. /* cross-flag validation per REQ-2 */
10. validate_flag_combo(flags)?;
11. /* entries clamp */
12. let sq_entries = clamp_or_eval(entries, IORING_MAX_ENTRIES, flags.contains(CLAMP))?;
13. let sq_entries = sq_entries.next_power_of_two();
14. /* cq_entries derivation */
15. let cq_entries = if flags.contains(CQSIZE) {
      - let req = p.cq_entries;
      - if req < sq_entries { return -EINVAL; }
      - clamp_or_eval(req, IORING_MAX_CQ_ENTRIES, flags.contains(CLAMP))?.next_power_of_two()
    } else { sq_entries * 2 };
16. /* charge memlock */
17. let pages = ring_pages(sq_entries, cq_entries, flags);
18. current.mm.charge_locked(pages).map_err(|_| -ENOMEM)?;
19. /* allocate context */
20. let ctx = Arc::new(IoRingCtx::new(sq_entries, cq_entries, flags)?);
21. /* attach wq if requested */
22. if flags.contains(ATTACH_WQ) { ctx.attach_wq(p.wq_fd)?; }
23. /* sqpoll kthread */
24. if flags.contains(SQPOLL) { ctx.spawn_sqpoll(p.sq_thread_cpu, p.sq_thread_idle)?; }
25. /* reflect offsets + features */
26. p.sq_entries = sq_entries;
27. p.cq_entries = cq_entries;
28. p.features   = ctx.features_bits();
29. p.sq_off     = ctx.sq_offsets();
30. p.cq_off     = ctx.cq_offsets();
31. /* install fd */
32. let fd = if flags.contains(REGISTERED_FD_ONLY) {
      - ctx.set_registered_only(true);
      - -1_i32 as u32
    } else {
      - anon_inode_getfd("[io_uring]", &IO_URING_FOPS, ctx.clone(), O_RDWR | O_CLOEXEC)?
    };
33. /* if R_DISABLED set, mark ctx disabled */
34. if flags.contains(R_DISABLED) { ctx.set_disabled(true); }
35. /* copy params back */
36. copy_to_user(params_uptr, &p).map_err(|_| -EFAULT)?;
37. return fd as isize;

`validate_flag_combo(f)`:
- if f.contains(SQ_AFF) && !f.contains(SQPOLL): return Err(-EINVAL).
- if f.contains(DEFER_TASKRUN) && !f.contains(SINGLE_ISSUER): return Err(-EINVAL).
- if f.contains(SQPOLL) && f.contains(SINGLE_ISSUER): return Err(-EINVAL).
- if f.contains(HYBRID_IOPOLL) && !f.contains(IOPOLL): return Err(-EINVAL).
- if f.contains(NO_MMAP) && !user_addrs_present(): return Err(-EINVAL).
- Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `entries_nonzero` | INVARIANT | `entries == 0 ⟹ -EINVAL` |
| `entries_bounded` | INVARIANT | post: `sq_entries ≤ IORING_MAX_ENTRIES` |
| `pow2_rounding` | INVARIANT | post: `sq_entries.is_power_of_two() ∧ cq_entries.is_power_of_two()` |
| `unknown_flag_rejected` | INVARIANT | `p.flags & !known ⟹ -EINVAL` |
| `resv_zero` | INVARIANT | `p.resv != [0;3] ⟹ -EINVAL` |
| `flag_combo_exclusive` | INVARIANT | `SQPOLL ∧ SINGLE_ISSUER ⟹ -EINVAL` |
| `memlock_charged_balanced` | INVARIANT | on err-after-charge: uncharge before return |
| `fd_install_atomic` | INVARIANT | fd visible iff full ctx initialized |

### Layer 2: TLA+

`uapi/io_uring_setup.tla`:
- States: `validating`, `allocating`, `attaching_wq`, `spawning_sqpoll`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_no_partial_install` — fd never in user table unless ctx fully initialized.
  - `safety_memlock_balanced` — charge ⇔ free under any failure.
  - `safety_disabled_blocks_enter` — `R_DISABLED` ring ⟹ `io_uring_enter` returns `-EBADFD` until enable.
  - `liveness_eventually_returns` — every call terminates with fd or errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `setup` post(ok): `0 ≤ ret ∧ ctx.sq_size == p.sq_entries` | `IoUring::setup` |
| `setup` post(err): `current.mm.memlock unchanged` | `IoUring::setup` |
| `validate_flag_combo` post: returned-ok ⟹ no exclusive pair | `validate_flag_combo` |
| `cq_entries_derivation` post: `!CQSIZE ⟹ cq == 2*sq` | `IoUring::setup` |

### Layer 4: Verus/Creusot functional

Per-`io_uring_setup(2)` man page + `Documentation/userspace-api/io_uring.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/io_uring/io_uring_setup0[123].c` and liburing `test/setup.c` round-trip. Cross-check against `tools/io_uring/io_uring-bench`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

io_uring setup reinforcement:

- **Per-uid pinned ring quota** — defense against per-uid memlock exhaustion via thousands of rings.
- **Per-task max-rings limit** — defense against per-task fd table flood.
- **`R_DISABLED` mandatory under restrictive policy** — defense against per-issuer running ops before restrictions register.
- **Per-flag exclusive matrix enforced at validation** — defense against per-undefined-state ring.
- **Per-`O_CLOEXEC` default** — defense against per-exec leak of issuer ring to setuid child.
- **Personality-required for cross-cred submission** — defense against per-cred-confusion.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on io_uring_setup entry** — randomize kernel stack at syscall entry; prevents stack-layout disclosure via crafted `params` returning consistent offsets exploitable in a leak-then-pivot chain.
- **PaX UDEREF on `copy_from_user`/`copy_to_user` of `struct io_uring_params`** — enforces unsigned-only userspace deref; refuses kernel pointers masqueraded as `params *` (closes the historical io_uring SLAB-grooming pattern that uses params as a probe oracle).
- **GRKERNSEC_BPF_HARDEN extended to io_uring as register surface** — io_uring's `IORING_REGISTER_*` opcodes form a second mini-instruction-set into the kernel comparable to BPF; the same hardening doctrine applies — refuse unprivileged ring setup under `kernel.io_uring_disabled ≥ 1`, audit every setup with `flags & (SQPOLL | NO_MMAP | SQE128 | CQE32)` to syslog with `current->comm` + `pid` + flag-mask, and reject ring creation when crosslink seccomp lockdown forbids `io_uring`.
- **CAP_BPF gating for privileged ring features** — `IORING_SETUP_SQPOLL`, `IORING_SETUP_NO_MMAP`, `IORING_SETUP_ATTACH_WQ` require `CAP_BPF` (or the per-uid `io_uring_group` GID) under hardened policy; mirrors the way grsec restricts BPF program load to the BPF-cap holder.
- **GRKERNSEC_FIFO scaled to ring flood** — per-uid quota on simultaneous open io_uring fds and per-uid rate-limit on `io_uring_setup` itself; refuses creation with `-EMFILE` past quota to break ring-flood DoS that grooms the SLAB.
- **GRKERNSEC_HIDESYM on ring offset reflection** — `p->sq_off` / `p->cq_off` contain kernel-stable layout offsets that do not leak addresses, but the `features` bitmask leaks kernel-version information; under `kernel.kptr_restrict ≥ 2` the feature reflection is restricted to features matching the calling task's seccomp profile, denying probes to fingerprint the kernel.
- **PAX_USERCOPY hardening on `struct io_uring_params`** — validate `usercopy_kernel(params, sizeof(IoUringParams), false)`; refuses copies whose destination overlaps a SLAB cache boundary that crosses into another object — the io_uring setup path is a known target for cross-cache write primitives.
- **GRKERNSEC_HARDEN_IPC for `ATTACH_WQ`** — refuse `ATTACH_WQ` across distinct security contexts (different SELinux domain / AppArmor profile / crosslink namespace) even when ptrace policy would permit; an attached worker pool is a shared kernel-execution channel.
- **Audit-log on every `IORING_SETUP_SQPOLL`** — SQPOLL spawns a persistent PF_IO_WORKER kernel thread that retains the issuer's credentials; log the spawn at `LOGLEVEL_WARNING` and reject if per-uid SQPOLL kthread count exceeds threshold.
- **Per-cgroup io_uring quota** — pids.max charge for SQPOLL kthread; memlock charge for ring pages must be enforced against cgroup memory.max (not just RLIMIT_MEMLOCK).

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `io_uring/io_uring-core.md` (Tier-3 runtime: SQ/CQ scheduler, op dispatch)
- `io_uring/sqpoll.md` (Tier-3 SQPOLL kthread loop)
- `io_uring/cqring.md` (Tier-3 CQ overflow + NODROP)
- `IORING_REGISTER_*` ABI — covered in `io_uring_register.md` sibling
- `io_uring_enter(2)` — covered in `io_uring_enter.md` sibling
- `uapi/headers/io_uring.md` (Tier-5 header reference)
- Implementation code
