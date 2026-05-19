---
title: "Tier-5 syscall: io_uring_register(2) — syscall 427"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_uring_register(2)` is the **out-of-band control surface** of io_uring — a multiplexed opcode-based syscall that registers, updates, and queries kernel-resident resources attached to a ring: pinned file descriptors, pinned buffer ranges, per-personality credentials, restriction policies, event-fd notification targets, IOWQ worker pools, NAPI poll context, registered ring-fd table, registered region buffers, clock source, zero-copy receive sockets, and more. Every opcode is its own mini-syscall; `arg` and `nr_args` are reinterpreted per opcode. Critical for: zero-copy I/O (fixed buffers), fast-path fixed-fd ops (`IOSQE_FIXED_FILE`), per-op personality switching, ring-level restriction policies, eventfd-backed completion wakeups, registered ring-fd indexing.

This Tier-5 covers the userspace ABI of syscall 427 — the opcode catalogue and per-opcode `arg`/`nr_args` rules. Per-resource lifecycle and pinning is owned by `io_uring/register.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `REGISTER_BUFFERS` with 4 iovecs returns 0; pinned pages charged to MEMLOCK.
- [ ] AC-2: Second `REGISTER_BUFFERS` without unregister → `-EBUSY`.
- [ ] AC-3: `REGISTER_FILES` with array of 8 fds returns 0; `IOSQE_FIXED_FILE` SQE referencing slot 3 resolves correctly.
- [ ] AC-4: `REGISTER_FILES_UPDATE` replaces slot 3 atomically.
- [ ] AC-5: `REGISTER_EVENTFD` with valid eventfd → 0; CQE causes eventfd counter += 1.
- [ ] AC-6: `REGISTER_PERSONALITY` returns id ≥ 1; `UNREGISTER_PERSONALITY` with id → 0; with bogus id → `-EINVAL`.
- [ ] AC-7: `REGISTER_RESTRICTIONS` on already-enabled ring → `-EBADFD`.
- [ ] AC-8: `ENABLE_RINGS` on enabled ring → `-EBADFD`.
- [ ] AC-9: `REGISTER_IOWQ_MAX_WORKERS` with `{4, 16}` returns prior values in same buffer.
- [ ] AC-10: `REGISTER_RING_FDS` with 4 ring fds returns 4; slot indices written to `data`.
- [ ] AC-11: `USE_REGISTERED_RING` bit + slot index works without ring fd in fdtable.
- [ ] AC-12: Unknown opcode (e.g., 99999) → `-EINVAL`.
- [ ] AC-13: `nr_args` mismatch for fixed-size struct opcode → `-EINVAL`.
- [ ] AC-14: `kernel.io_uring_disabled = 2` → `-EPERM` for any register.
- [ ] AC-15: `REGISTER_NAPI` with `busy_poll_to = 50` returns 0; per-ring NAPI list updated.

### Architecture

Rookery surface in `kernel/io_uring/syscall/register.rs`:

```rust
#[repr(u32)]
pub enum IoUringRegisterOp {
    RegisterBuffers          = 0,
    UnregisterBuffers        = 1,
    RegisterFiles            = 2,
    UnregisterFiles          = 3,
    RegisterEventfd          = 4,
    UnregisterEventfd        = 5,
    RegisterFilesUpdate      = 6,
    RegisterEventfdAsync     = 7,
    RegisterProbe            = 8,
    RegisterPersonality      = 9,
    UnregisterPersonality    = 10,
    RegisterRestrictions     = 11,
    RegisterEnableRings      = 12,
    RegisterFiles2           = 13,
    RegisterFilesUpdate2     = 14,
    RegisterBuffers2         = 15,
    RegisterBuffersUpdate    = 16,
    RegisterIowqAff          = 17,
    UnregisterIowqAff        = 18,
    RegisterIowqMaxWorkers   = 19,
    RegisterRingFds          = 20,
    UnregisterRingFds        = 21,
    RegisterPbufRing         = 22,
    UnregisterPbufRing       = 23,
    RegisterNapi             = 24,
    RegisterFileAllocRange   = 25,
    UnregisterNapi           = 26,
    RegisterPbufStatus       = 27,
    RegisterClock            = 29,
    RegisterCloneBuffers     = 30,
    RegisterZcrxIfq          = 32,
    RegisterRegion           = 33,
}

pub const IORING_REGISTER_USE_REGISTERED_RING: u32 = 1 << 31;
```

`IoUring::register(fd, opcode, arg, nr_args) -> isize`:
1. /* sysctl gate */
2. match sysctl.io_uring_disabled {
     - 2 => return -EPERM,
     - 1 if !privileged() => return -EPERM,
     - _ => (),
   }
3. /* resolve ring */
4. let use_reg = (opcode & IORING_REGISTER_USE_REGISTERED_RING) != 0;
5. let raw_op = opcode & !IORING_REGISTER_USE_REGISTERED_RING;
6. let op = IoUringRegisterOp::try_from(raw_op).map_err(|_| -EINVAL)?;
7. let ctx = if use_reg {
     - current.io_uring_reg_rings.get(fd).ok_or(-EINVAL)?
   } else {
     - current.fdtable.get(fd).ok_or(-EBADF)?.as_io_uring().ok_or(-EBADFD)?
   };
8. /* per-opcode dispatch */
9. match op {
   - RegisterBuffers           => ctx.register_buffers(arg, nr_args),
   - UnregisterBuffers         => ctx.unregister_buffers(),
   - RegisterFiles             => ctx.register_files(arg, nr_args),
   - UnregisterFiles           => ctx.unregister_files(),
   - RegisterEventfd           => ctx.register_eventfd(arg, false),
   - RegisterEventfdAsync      => ctx.register_eventfd(arg, true),
   - UnregisterEventfd         => ctx.unregister_eventfd(),
   - RegisterFilesUpdate       => ctx.register_files_update(arg, nr_args, false),
   - RegisterFilesUpdate2      => ctx.register_files_update(arg, nr_args, true),
   - RegisterFiles2            => ctx.register_files2(arg, nr_args),
   - RegisterBuffers2          => ctx.register_buffers2(arg, nr_args),
   - RegisterBuffersUpdate     => ctx.register_buffers_update(arg, nr_args),
   - RegisterPersonality       => ctx.register_personality(),
   - UnregisterPersonality     => ctx.unregister_personality(nr_args),
   - RegisterRestrictions      => ctx.register_restrictions(arg, nr_args),
   - RegisterEnableRings       => ctx.register_enable_rings(),
   - RegisterIowqAff           => ctx.register_iowq_aff(arg, nr_args),
   - UnregisterIowqAff         => ctx.unregister_iowq_aff(),
   - RegisterIowqMaxWorkers    => ctx.register_iowq_max_workers(arg),
   - RegisterRingFds           => ctx.register_ring_fds(arg, nr_args),
   - UnregisterRingFds         => ctx.unregister_ring_fds(arg, nr_args),
   - RegisterPbufRing          => ctx.register_pbuf_ring(arg),
   - UnregisterPbufRing        => ctx.unregister_pbuf_ring(arg),
   - RegisterPbufStatus        => ctx.register_pbuf_status(arg),
   - RegisterNapi              => ctx.register_napi(arg),
   - UnregisterNapi            => ctx.unregister_napi(arg),
   - RegisterClock             => ctx.register_clock(arg),
   - RegisterRegion            => ctx.register_region(arg),
   - RegisterCloneBuffers      => ctx.register_clone_buffers(arg),
   - RegisterFileAllocRange    => ctx.register_file_alloc_range(arg),
   - RegisterZcrxIfq           => ctx.register_zcrx_ifq(arg),
   - RegisterProbe             => ctx.register_probe(arg, nr_args),
}

`ctx.register_buffers(arg, nr_args)`:
1. if nr_args == 0 || nr_args > UIO_MAXIOV { return Err(-EINVAL); }
2. if ctx.buffers.is_some() { return Err(-EBUSY); }
3. let iov: Vec<IoVec> = copy_from_user_array(arg, nr_args)?;
4. let pinned = pin_user_pages_all(&iov)?;
5. current.mm.charge_locked(pinned.pages)?;
6. ctx.buffers = Some(pinned);
7. Ok(0).

`ctx.register_restrictions(arg, nr_args)`:
1. if !ctx.is_disabled() { return Err(-EBADFD); }
2. let r: Vec<IoUringRestriction> = copy_from_user_array(arg, nr_args)?;
3. for rest in r { ctx.restrictions.apply(rest)?; }
4. Ok(0).

`ctx.register_enable_rings()`:
1. if !ctx.is_disabled() { return Err(-EBADFD); }
2. ctx.set_disabled(false);
3. Ok(0).

### Out of Scope

- `io_uring/register.md` Tier-3 per-resource lifecycle
- `io_uring/restrictions.md` Tier-3 restriction policy engine
- `io_uring/pbuf.md` Tier-3 provided-buffer ring runtime
- `io_uring/zcrx.md` Tier-3 zero-copy receive ring
- `io_uring_setup.md` / `io_uring_enter.md` siblings
- Implementation code

### signature

```c
int io_uring_register(unsigned int fd,
                      unsigned int opcode,
                      void *arg,
                      unsigned int nr_args);
```

Rust ABI shim:

```rust
pub fn sys_io_uring_register(fd: u32,
                             opcode: u32,
                             arg: *mut u8,
                             nr_args: u32) -> isize;
```

Syscall number: **427**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fd` | `u32` | IN | ring fd from `io_uring_setup(2)`, or registered-ring index when `IORING_REGISTER_USE_REGISTERED_RING` bit set in `opcode` |
| `opcode` | `u32` | IN | `IORING_REGISTER_*` opcode plus optional flag bits |
| `arg` | `void *` | IN/OUT | per-opcode buffer or descriptor list |
| `nr_args` | `u32` | IN | per-opcode count or size discriminant |

`opcode` upper bits:

| Bit | Purpose |
|---|---|
| `IORING_REGISTER_USE_REGISTERED_RING` (`1 << 31`) | `fd` is registered-ring slot index, not a regular fd |

### return

- **Success**: zero, or non-negative count/identifier per opcode (e.g., `IORING_REGISTER_PERSONALITY` returns the personality id).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | unknown opcode; per-opcode `nr_args` constraint violated; mutually-exclusive registration (e.g., re-register without unregister); invalid file/buffer descriptor list shape |
| `EBADF` | `fd` not a valid fd |
| `EBADFD` | `fd` is not an io_uring fd; or ring is in a state incompatible with opcode (e.g., `IORING_REGISTER_RESTRICTIONS` after enable) |
| `EFAULT` | `arg` not in user-space-readable address range |
| `ENOMEM` | per-opcode resource allocation failed (RLIMIT_MEMLOCK for fixed buffers; cgroup memory.max) |
| `EBUSY` | resource currently in use and opcode requires it idle |
| `EOVERFLOW` | per-opcode size or count overflows |
| `EOPNOTSUPP` | opcode known in UAPI but disabled at build/runtime |
| `EEXIST` | resource already registered and opcode does not allow replace |
| `ENOENT` | unregister/update opcode targeted a non-existent slot |
| `EPERM` | `kernel.io_uring_disabled` blocks or cap-gated opcode without capability |

### abi surface

`IORING_REGISTER_*` opcodes (selected — full catalogue ~50 entries):

### Buffer pinning

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_BUFFERS` | 0 | `struct iovec[]` | nr iovecs (≤ `UIO_MAXIOV`=1024) |
| `IORING_UNREGISTER_BUFFERS` | 1 | NULL | 0 |
| `IORING_REGISTER_BUFFERS2` | 15 | `struct io_uring_rsrc_register *` | sizeof(struct) |
| `IORING_REGISTER_BUFFERS_UPDATE` | 16 | `struct io_uring_rsrc_update2 *` | sizeof(struct) |
| `IORING_REGISTER_PBUF_RING` | 22 | `struct io_uring_buf_reg *` | 1 |
| `IORING_UNREGISTER_PBUF_RING` | 23 | `struct io_uring_buf_reg *` | 1 |
| `IORING_REGISTER_PBUF_STATUS` | 27 | `struct io_uring_buf_status *` | 1 |

### Fixed files

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_FILES` | 2 | `int[]` | nr files |
| `IORING_UNREGISTER_FILES` | 3 | NULL | 0 |
| `IORING_REGISTER_FILES_UPDATE` | 6 | `struct io_uring_files_update *` | nr |
| `IORING_REGISTER_FILES2` | 13 | `struct io_uring_rsrc_register *` | sizeof(struct) |
| `IORING_REGISTER_FILES_UPDATE2` | 14 | `struct io_uring_rsrc_update2 *` | sizeof(struct) |
| `IORING_REGISTER_FILE_ALLOC_RANGE` | 25 | `struct io_uring_file_index_range *` | sizeof(struct) |

### Events / signal

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_EVENTFD` | 4 | `int *` | 1 |
| `IORING_UNREGISTER_EVENTFD` | 5 | NULL | 0 |
| `IORING_REGISTER_EVENTFD_ASYNC` | 7 | `int *` | 1 |

### Personality

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_PERSONALITY` | 9 | NULL | 0 (returns id) |
| `IORING_UNREGISTER_PERSONALITY` | 10 | NULL | id (as `nr_args`) |

### Restrictions / enable

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_RESTRICTIONS` | 11 | `struct io_uring_restriction *` | nr |
| `IORING_REGISTER_ENABLE_RINGS` | 12 | NULL | 0 |

### IOWQ

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_IOWQ_AFF` | 17 | `cpu_set_t *` | size of mask in bytes |
| `IORING_UNREGISTER_IOWQ_AFF` | 18 | NULL | 0 |
| `IORING_REGISTER_IOWQ_MAX_WORKERS` | 19 | `unsigned int[2]` (bounded, unbounded) | 2 |

### Ring-fd

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_RING_FDS` | 20 | `struct io_uring_rsrc_update[]` | nr |
| `IORING_UNREGISTER_RING_FDS` | 21 | `struct io_uring_rsrc_update[]` | nr |

### NAPI

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_NAPI` | 24 | `struct io_uring_napi *` | 1 |
| `IORING_UNREGISTER_NAPI` | 26 | `struct io_uring_napi *` (out: prior) | 1 |

### Clock / sync / region

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_CLOCK` | 29 | `struct io_uring_clock_register *` | 1 |
| `IORING_REGISTER_CLONE_BUFFERS` | 30 | `struct io_uring_clone_buffers *` | 1 |
| `IORING_REGISTER_SYNC_CANCEL` | 24 (aliased per kernel; verify) | `struct io_uring_sync_cancel_reg *` | 1 |
| `IORING_REGISTER_REGION` | 33 | `struct io_uring_region_desc *` | 1 |

### Zero-copy rx

| Opcode | Value | `arg` | `nr_args` |
|---|---|---|---|
| `IORING_REGISTER_ZCRX_IFQ` | 32 | `struct io_uring_zcrx_ifq_reg *` | 1 |

### Key UAPI structs

```c
struct io_uring_rsrc_register {
    __u32 nr;
    __u32 flags;        /* IORING_RSRC_REGISTER_SPARSE */
    __u64 resv2;
    __u64 data;         /* ptr to iovec[] or int[] */
    __u64 tags;         /* ptr to __u64[nr] */
};

struct io_uring_rsrc_update2 {
    __u32 offset;
    __u32 resv;
    __u64 data;
    __u64 tags;
    __u32 nr;
    __u32 resv2;
};

struct io_uring_files_update {
    __u32 offset;
    __u32 resv;
    __aligned_u64 fds;
};

struct io_uring_restriction {
    __u16 opcode;        /* IORING_RESTRICTION_* */
    union {
        __u8 register_op; /* RESTRICT_REGISTER_OP   */
        __u8 sqe_op;      /* RESTRICT_SQE_OP        */
        __u8 sqe_flags;   /* RESTRICT_SQE_FLAGS_*   */
    };
    __u8 resv;
    __u32 resv2[3];
};

struct io_uring_buf_reg {
    __u64 ring_addr;
    __u32 ring_entries;
    __u16 bgid;
    __u16 pad;
    __u64 resv[3];
};

struct io_uring_napi {
    __u32 busy_poll_to;
    __u8  prefer_busy_poll;
    __u8  opcode;        /* register / unregister */
    __u8  pad[2];
    __u32 op_param;
    __u32 resv;
};
```

### compatibility contract

REQ-1: Opcode dispatch:
- `opcode & ~(IORING_REGISTER_USE_REGISTERED_RING)` indexes per-opcode handler.
- Unknown opcode → `-EINVAL`.

REQ-2: `IORING_REGISTER_BUFFERS`:
- Pins each iovec via `pin_user_pages` and charges pages to `RLIMIT_MEMLOCK` and cgroup memory.max.
- Total iovecs ≤ 1024 default; raised by `IORING_REGISTER_BUFFERS2` up to `UIO_MAXIOV << 8` with sparse.
- Re-register without `UNREGISTER_BUFFERS` first → `-EBUSY`.

REQ-3: `IORING_REGISTER_FILES`:
- `arg` is `int[]` of fds; `nr_args ≤ 32768`.
- Each fd dup'd into the ring; index used as `IOSQE_FIXED_FILE` slot.
- Sparse slots permitted via `IORING_RSRC_REGISTER_SPARSE`.
- `IORING_REGISTER_FILE_ALLOC_RANGE` constrains slot allocation to `[start, start+len)`.

REQ-4: `IORING_REGISTER_EVENTFD`:
- `arg` is `int *` pointing to eventfd fd.
- Kernel calls `eventfd_signal(ctx, 1)` on each CQE completion (default mode) or only on async-from-iowq completions (`_ASYNC` variant).
- Re-register without unregister → `-EBUSY`.

REQ-5: `IORING_REGISTER_PERSONALITY`:
- Captures `current->cred` and stores under returned id (≥ 1).
- `IOSQE_PERSONALITY` SQE flag references id; per-op runs under captured creds.
- Total personalities per ring bounded by `IO_URING_MAX_PERSONALITIES = 65536`.

REQ-6: `IORING_REGISTER_RESTRICTIONS`:
- Only valid when ring is `R_DISABLED`; else `-EBADFD`.
- Each `io_uring_restriction` opcode constrains future `register_op` opcodes, `sqe_op` opcodes, or `sqe_flags` bits.
- After `IORING_REGISTER_ENABLE_RINGS`, restrictions are immutable.

REQ-7: `IORING_REGISTER_ENABLE_RINGS`:
- Clears `R_DISABLED`.
- One-shot; calling on already-enabled ring → `-EBADFD`.

REQ-8: `IORING_REGISTER_IOWQ_MAX_WORKERS`:
- `arg` is `unsigned int[2]`: bounded (default I/O) and unbounded (CPU-bound) caps.
- On return, prior values written back.
- Per-cgroup pids.max still applies.

REQ-9: `IORING_REGISTER_RING_FDS`:
- Stores up to `IO_RINGFD_REG_MAX = 16` ring fds for thread-local fast access in `io_uring_enter`.
- Each entry's `offset` field returned, `data` field is the slot index.

REQ-10: `IORING_REGISTER_NAPI`:
- `busy_poll_to` is microseconds.
- `prefer_busy_poll`: 0/1.
- Per-ring NAPI ID list maintained for socket op steering.

REQ-11: `IORING_REGISTER_CLOCK`:
- Selects clock source for timeouts: `CLOCK_MONOTONIC` (default) / `CLOCK_BOOTTIME` / `CLOCK_REALTIME`.
- Affects per-op `LINK_TIMEOUT`, `TIMEOUT`, and `enter` `ts` semantics.

REQ-12: `IORING_REGISTER_PBUF_RING`:
- `arg.ring_addr` is user-mapped buffer ring (separate from SQ/CQ).
- `bgid` is buffer group id (used with `IORING_OP_PROVIDE_BUFFERS` legacy or `IOSQE_BUFFER_SELECT`).
- `ring_entries` MUST be pow2.

REQ-13: `IORING_REGISTER_REGION`:
- Registers a user-supplied memory region for kernel use under `NO_MMAP` / `EXT_ARG_REG`.

REQ-14: Per-opcode counter check:
- Most opcodes require `nr_args` to equal the array length or struct size; mismatch → `-EINVAL`.

REQ-15: `IORING_REGISTER_USE_REGISTERED_RING` bit:
- When set, `fd` is the slot index from `IORING_REGISTER_RING_FDS`.
- Allows `io_uring_register` without holding the real ring fd open in the fd table.

REQ-16: Sysctl gating:
- `kernel.io_uring_disabled == 2` → `-EPERM` on all register opcodes.
- `kernel.io_uring_disabled == 1` and not in `io_uring_group` and not CAP_SYS_ADMIN → `-EPERM`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `opcode_known` | INVARIANT | unknown opcode ⟹ -EINVAL |
| `nr_args_match` | INVARIANT | per-opcode size check ⟹ -EINVAL on mismatch |
| `buffers_reregister_busy` | INVARIANT | second REGISTER_BUFFERS without unregister ⟹ -EBUSY |
| `restrictions_disabled_only` | INVARIANT | REGISTER_RESTRICTIONS ∧ !R_DISABLED ⟹ -EBADFD |
| `enable_one_shot` | INVARIANT | second ENABLE_RINGS ⟹ -EBADFD |
| `memlock_charged_balanced` | INVARIANT | on err: uncharge before return |
| `personality_id_unique` | INVARIANT | returned id non-zero, unique per ring |

### Layer 2: TLA+

`uapi/io_uring_register.tla`:
- States per resource: `unregistered`, `registered`, `in_update`.
- Properties:
  - `safety_resource_singular` — at most one active registration per resource until unregister.
  - `safety_restrictions_pre_enable` — restrictions immutable post-enable.
  - `safety_personality_isolation` — personality A ops never run under personality B's creds.
  - `liveness_register_terminates` — every opcode invocation terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register` post: per-opcode resource attached or err | `IoUring::register` |
| `register_buffers` post: pinned pages charged | `ctx.register_buffers` |
| `unregister_buffers` post: pinned pages uncharged | `ctx.unregister_buffers` |
| `register_restrictions` post: ring still disabled | `ctx.register_restrictions` |
| `register_enable_rings` post: ring enabled | `ctx.register_enable_rings` |

### Layer 4: Verus/Creusot functional

Per-`io_uring_register(2)` man page and `Documentation/userspace-api/io_uring.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/io_uring/io_uring_register0[1234].c` and liburing `test/register*.c` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

io_uring register reinforcement:

- **Per-resource singular registration** — defense against per-double-pin UAF.
- **Per-restriction immutable post-enable** — defense against per-policy-bypass.
- **Per-personality id strict bounds** — defense against per-id-overflow.
- **MEMLOCK + cgroup charge on REGISTER_BUFFERS** — defense against per-uid memory exhaustion via fixed buffers.
- **`USE_REGISTERED_RING` slot bounds** — defense against per-out-of-bounds slot access.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on io_uring_register entry** — randomize kernel stack at syscall entry; register is the second-highest-frequency io_uring syscall and a known target for layout disclosure.
- **PaX UDEREF on every per-opcode `arg` copy** — register's polymorphic `arg`/`nr_args` is the most dangerous primitive in io_uring (multiplexed type-confusion surface); enforce unsigned userspace deref for every opcode-specific struct (`iovec[]`, `io_uring_rsrc_register`, `io_uring_files_update`, `io_uring_restriction`, `io_uring_buf_reg`, `io_uring_napi`, `io_uring_clock_register`, `io_uring_region_desc`); refuses kernel pointers masqueraded as `arg`.
- **GRKERNSEC_BPF_HARDEN extended to io_uring as register surface** — `IORING_REGISTER_*` is structurally a second BPF: a privileged opcode-based VM that mutates kernel-resident state per syscall. Apply BPF-style hardening: refuse register under `kernel.io_uring_disabled ≥ 1` without `CAP_BPF` or `io_uring_group`; audit every register opcode with `current->comm`, `pid`, `opcode`, `nr_args`; per-uid rate-limit on register calls.
- **CAP_BPF gating for privileged register opcodes** — `IORING_REGISTER_IOWQ_AFF` (CPU affinity), `IORING_REGISTER_IOWQ_MAX_WORKERS` (worker pool sizing), `IORING_REGISTER_NAPI` (kernel poll context), `IORING_REGISTER_CLOCK` (clock source switching), `IORING_REGISTER_ZCRX_IFQ` (zero-copy receive interface) require `CAP_BPF` under hardened policy.
- **GRKERNSEC_FIFO scaled to register-flood** — per-uid quota on register opcodes per second; under flood (rapid register/unregister churn to groom the SLAB allocator) refuse with `-EBUSY` and log.
- **GRKERNSEC_HIDESYM on `REGISTER_PROBE` response** — opcode 8 (`REGISTER_PROBE`) returns per-op support bits, which together fingerprint kernel build; under `kernel.kptr_restrict ≥ 2` filter response to per-task seccomp-permitted ops only.
- **PAX_USERCOPY hardening on every per-opcode struct copy** — validate exact-size copies for every per-opcode `arg`; refuse copies whose source spans SLAB-cache boundaries; the multiplexed nature of register is exactly the SLAB-grooming primitive the kernel attacker community optimizes against.
- **GRKERNSEC_HARDEN_IPC for `REGISTER_RING_FDS` / `REGISTER_PERSONALITY`** — refuse `REGISTER_RING_FDS` cross-namespace and refuse `REGISTER_PERSONALITY` capture of creds that differ from `current->cred` along a setuid-sensitive axis (cap-set delta, gid delta).
- **Strict immutability of restrictions** — after `IORING_REGISTER_ENABLE_RINGS`, any attempt to call `REGISTER_RESTRICTIONS` returns `-EBADFD` *and* is audit-logged at `LOGLEVEL_WARNING`; defense against per-policy-bypass via post-enable mutation race.
- **Mandatory `IORING_REGISTER_RESTRICTIONS` under unprivileged usage** — under hardened policy, a non-CAP_BPF task may only use a ring after registering at least a minimal restriction set; eliminates "register ring, do anything" attack pattern.
- **Personality cred-capture audit** — every `IORING_REGISTER_PERSONALITY` whose captured creds carry `CAP_*` differing from current's creds (e.g., from prior `seteuid`) is logged for forensics.

