# Tier-5 syscall: eventfd2(2) — syscall 290

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/eventfd.c (sys_eventfd, sys_eventfd2, do_eventfd)
  - include/uapi/linux/eventfd.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (290)
  - Documentation/admin-guide/eventfd.rst, man eventfd(2)
-->

## Summary

`eventfd2(2)` allocates a kernel-resident 64-bit unsigned counter and returns a file descriptor that reads/writes it as a lightweight signaling primitive. Supersedes the legacy `eventfd(2)` (syscall 213/284 per arch) by adding a flags argument (`EFD_SEMAPHORE`, `EFD_CLOEXEC`, `EFD_NONBLOCK`). The fd is poll/select/epoll-friendly: `POLLIN` when counter > 0, `POLLOUT` when counter would not saturate. Two read modes: default ("counter dump") returns the entire value and resets to zero; `EFD_SEMAPHORE` returns `1` per read and decrements by one. Critical for: replacing pipe(2)-based wakeups in libuv/glib/io_uring/KVM irqfd/vhost/aio; kernel-to-userspace signaling without `write()` overhead; userspace-to-userspace event aggregation.

This Tier-5 covers the userspace ABI of syscall 290 (the **syscall**, not the header — that's `uapi/headers/eventfd.md`).

## Signature

```c
int eventfd2(unsigned int initval, int flags);
```

Rust ABI shim:

```rust
pub fn sys_eventfd2(initval: u32, flags: i32) -> isize;
```

Syscall number: **290**.

Legacy companion `eventfd(2)` (syscall 284):
- Signature: `int eventfd(unsigned int initval);` — `flags == 0` implied.
- Behavior identical to `eventfd2(initval, 0)`.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `initval` | `u32` | IN | initial counter value (widened to u64) |
| `flags` | `i32` | IN | bitmask: `EFD_SEMAPHORE | EFD_CLOEXEC | EFD_NONBLOCK` |

## Return

- **Success**: non-negative file descriptor.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags` contains an unknown bit |
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`) |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | kernel cannot allocate `struct eventfd_ctx` |
| `ENODEV` | anon_inode mount unavailable (impossible on standard builds) |
| `EPERM` | restrictive policy denies creation (e.g., seccomp lockdown) |

## ABI surface

`EFD_*` flags:

| Constant | Value | Purpose |
|---|---|---|
| `EFD_SEMAPHORE` | `1 << 0` (`0x1`) | semaphore-mode read: returns `1`, decrements counter by 1 |
| `EFD_CLOEXEC` | `O_CLOEXEC` (`0x80000` typically) | close-on-exec |
| `EFD_NONBLOCK` | `O_NONBLOCK` (`0x800` typically) | non-blocking read/write |

Counter:
- Stored as `u64`.
- Valid range: `[0, EFD_MAX = 0xfffffffffffffffe]`.
- `0xffffffffffffffff` reserved as "would overflow" sentinel.

Read frame: 8 bytes (`__u64`).
Write frame: 8 bytes (`__u64`).

## Compatibility contract

REQ-1: `flags` validation:
- Subset of `{EFD_SEMAPHORE, EFD_CLOEXEC, EFD_NONBLOCK}`; else `-EINVAL`.

REQ-2: `initval`:
- Widened to `u64` and stored as initial counter.
- No upper-bound check: `initval` is `u32`, max `0xffffffff`, well below `EFD_MAX`.

REQ-3: Returned fd:
- `O_RDWR`-equivalent.
- `O_CLOEXEC` set iff `EFD_CLOEXEC` flag.
- `O_NONBLOCK` set iff `EFD_NONBLOCK` flag.
- `fcntl(F_GETFL)` reflects these.
- Read/write via standard `read(2)` / `write(2)` syscalls (kernel I/O path).
- Poll-able via `poll(2)` / `select(2)` / `epoll_ctl(2)`.

REQ-4: Initial state:
- `counter = initval as u64`.
- Wait queues empty.

REQ-5: anon_inode:
- Inode is anon_inode `[eventfd]`.
- Visible in `/proc/<pid>/fd/<N>` as `anon_inode:[eventfd]`.
- `/proc/<pid>/fdinfo/<N>` exposes `eventfd-count: <N>` and `eventfd-id: <N>`.

REQ-6: Per-process fd accounting:
- Consumes one `RLIMIT_NOFILE` slot.

REQ-7: Lifetime:
- `close(2)` triggers teardown.
- `dup(2)` / `fork(2)`: shared `struct file`; both holders see same counter.
- `execve(2)`: closed iff `O_CLOEXEC`.

REQ-8: Semantic for read/write/poll covered in `uapi/headers/eventfd.md` and not re-stated here. Summary:
- Default read: returns counter, resets to 0.
- `EFD_SEMAPHORE` read: returns 1, counter -= 1.
- Write: counter += inc, capped at `EFD_MAX`, sentinel rejected.
- Poll: `POLLIN ⟺ counter > 0`; `POLLOUT ⟺ counter + 1 ≤ EFD_MAX`.

REQ-9: Legacy `eventfd(initval)` is `eventfd2(initval, 0)`.

REQ-10: `fcntl(fd, F_SETFL, O_NONBLOCK)` toggles `EFD_NONBLOCK` post-create.

## Acceptance Criteria

- [ ] AC-1: `eventfd2(0, 0)` returns fd; counter == 0.
- [ ] AC-2: `eventfd2(5, EFD_SEMAPHORE)` returns fd; semaphore-mode set.
- [ ] AC-3: `eventfd2(0, EFD_CLOEXEC)`: `fcntl(fd, F_GETFD)` has `FD_CLOEXEC`.
- [ ] AC-4: `eventfd2(0, EFD_NONBLOCK)`: `fcntl(fd, F_GETFL)` has `O_NONBLOCK`.
- [ ] AC-5: `eventfd2(0, EFD_SEMAPHORE | EFD_CLOEXEC | EFD_NONBLOCK)` returns fd; all flags reflected.
- [ ] AC-6: `eventfd2(0, 0x10)` (unknown bit) → `-EINVAL`.
- [ ] AC-7: `eventfd2(0, 0xffffffff)` → `-EINVAL`.
- [ ] AC-8: `RLIMIT_NOFILE` exhausted → `-EMFILE`.
- [ ] AC-9: `read(fd, &v, 4)` (count < 8) → `-EINVAL`.
- [ ] AC-10: `write(fd, &1, 8)` → counter += 1; `read(fd, &v, 8)` → v=1, counter=0.
- [ ] AC-11: `EFD_SEMAPHORE`: write 3; three reads each return 1; fourth blocks/EAGAIN.
- [ ] AC-12: Created fd shows as `anon_inode:[eventfd]` in `/proc/<pid>/fd/`.
- [ ] AC-13: After `execve` with `EFD_CLOEXEC`: fd absent.
- [ ] AC-14: Legacy `eventfd(7)` (syscall 284) behaves identical to `eventfd2(7, 0)`.
- [ ] AC-15: `eventfd2(u32::MAX, 0)` → fd; counter == `0xffffffff`.

## Architecture

Rookery surface in `kernel/eventfd/syscall.rs`:

```rust
bitflags! {
    pub struct EventfdFlags: i32 {
        const SEMAPHORE = 1 << 0;
        const CLOEXEC   = 0x80000;
        const NONBLOCK  = 0x800;
    }
}

pub const EFD_MAX: u64 = 0xfffffffffffffffe;

pub struct EventfdCtx {
    pub count:  AtomicU64,
    pub flags:  EventfdFlags,
    pub wq:     WaitQueue,
    pub id:     u64,                 /* monotonic for /proc/fdinfo */
}
```

`Eventfd::eventfd2(initval, flags) -> isize`:
1. /* flag validation */
2. let f = EventfdFlags::from_bits(flags).ok_or(-EINVAL)?;
3. /* allocate ctx */
4. let ctx = Arc::new(EventfdCtx {
     - count: AtomicU64::new(initval as u64),
     - flags: f,
     - wq: WaitQueue::new(),
     - id: next_eventfd_id(),
   });
5. /* open flags translation */
6. let open_flags = O_RDWR
     | if f.contains(EventfdFlags::CLOEXEC) { O_CLOEXEC } else { 0 }
     | if f.contains(EventfdFlags::NONBLOCK) { O_NONBLOCK } else { 0 };
7. let fd = anon_inode_getfd("[eventfd]", &EVENTFD_FOPS, ctx, open_flags)?;
8. return fd as isize;

`Eventfd::eventfd_legacy(initval) -> isize`:
1. Eventfd::eventfd2(initval, 0).

`EVENTFD_FOPS`:
1. read     ⟹ Eventfd::read (per uapi/headers/eventfd.md REQ-3).
2. write    ⟹ Eventfd::write (REQ-4).
3. poll     ⟹ Eventfd::poll (REQ-5).
4. release  ⟹ free ctx; wake all waiters with HUP.
5. show_fdinfo ⟹ write "eventfd-count: <count>\neventfd-id: <id>\n".

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_subset` | INVARIANT | `flags & !ALL_FLAGS != 0 ⟹ -EINVAL` |
| `cloexec_propagates` | INVARIANT | `EFD_CLOEXEC ⟹ FD_CLOEXEC set` |
| `nonblock_propagates` | INVARIANT | `EFD_NONBLOCK ⟹ O_NONBLOCK set` |
| `counter_init` | INVARIANT | post: `ctx.count == initval` |
| `fd_install_atomic` | INVARIANT | fd visible iff ctx fully initialized |
| `legacy_equivalence` | INVARIANT | `eventfd(initval) ≡ eventfd2(initval, 0)` |

### Layer 2: TLA+

`uapi/eventfd2.tla`:
- States: `validating`, `allocating`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_no_partial_install` — fd visible iff ctx fully initialized.
  - `safety_flags_reflected` — fd's open flags reflect creation flags.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `eventfd2` post(ok): `0 ≤ ret` ∧ `ctx.count == initval` | `Eventfd::eventfd2` |
| `eventfd2` post: ctx.flags == flags | `Eventfd::eventfd2` |
| `eventfd_legacy` post: equivalent to eventfd2(_, 0) | `Eventfd::eventfd_legacy` |

### Layer 4: Verus/Creusot functional

Per-`eventfd(2)` man page, `Documentation/admin-guide/eventfd.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/eventfd2/*.c` and glibc `sysdeps/unix/sysv/linux/eventfd.c` round-trip. Cross-check: liburing `io_uring_register_eventfd`, KVM `kvm_irqfd`, vhost-net.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

eventfd2 reinforcement:

- **CLOEXEC default under restrictive policy** — defense against per-exec leak of counter to setuid child.
- **Counter overflow detection in write path** — defense against per-torn-counter side channel.
- **Sentinel write strictly rejected** — defense against per-sentinel poisoning.
- **anon_inode source isolation** — defense against per-namespace cross-leak.
- **Per-user-namespace open count limit (sysctl)** — defense against per-user fd exhaustion.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on eventfd2 entry** — randomize kernel stack; eventfd2 is a low-frequency creation syscall but the associated read/write paths are high-frequency and grsec doctrine applies stack randomization at creation time to establish layout entropy for subsequent counter operations.
- **PaX UDEREF not directly applicable** — the syscall takes only scalars; no user pointers. UDEREF inherited from generic syscall entry path.
- **GRKERNSEC_BPF_HARDEN considerations** — eventfd is not a VM but is the wakeup primitive that io_uring (`IORING_REGISTER_EVENTFD`), KVM (`irqfd`), and vhost use for kernel-to-userspace signaling. The same audit doctrine applies: log every cross-domain eventfd creation paired with a subsequent register in those subsystems.
- **CAP_BPF gating not directly required** — eventfd is a baseline primitive.
- **GRKERNSEC_FIFO scaled to eventfd flood** — per-uid quota on simultaneous open eventfds and per-uid rate-limit on `eventfd2` creation; refuses with `-EMFILE` past quota. Mirrors grsec's GRKERNSEC_FIFO doctrine for pipes — eventfds are the modern equivalent of pipe wakeup and equally exploitable as a DoS vector via fd-table flooding.
- **EFD_CLOEXEC mandatory under restrictive policy** — when calling task has `PR_SET_NO_NEW_PRIVS=1`, runs under crosslink seccomp lockdown, or is invoked across a setuid boundary, refuse `eventfd2` without `EFD_CLOEXEC`; eliminates a cross-exec covert channel.
- **GRKERNSEC_HIDESYM on `/proc/<pid>/fdinfo/<N>` eventfd entries** — `eventfd-count` and `eventfd-id` leak across the calling task's fd namespace; under `kernel.kptr_restrict ≥ 2`, fdinfo restricted to `CAP_SYS_ADMIN` or same-uid + same-pidns observers.
- **Counter `EFD_MAX` strict ceiling** — refuse writes whose computed `counter + inc > EFD_MAX`; refuse the sentinel value `0xffffffffffffffff` as `inc` with `-EINVAL` plus audit-log entry. Defense against torn-counter primitive farming.
- **EFD_SEMAPHORE per-uid accounting** — semaphore-mode eventfds consume more wake-up overhead than plain counters; under hardened policy cap per-uid semaphore-eventfd count separately and refuse creation past threshold.
- **In-kernel `eventfd_signal()` rate-limit from interrupt context** — refuse in-kernel signals from IRQ handlers that exceed per-IRQ rate-limit; prevents lock-up via signal-flooding from misbehaving drivers (vhost / KVM irqfd / io_uring CQ wake).
- **Audit log on every `eventfd2` from setuid-history task** — task with `dumpable == 0` creating an eventfd is logged at `LOGLEVEL_INFO` for forensic correlation; eventfds are a common building block in privilege-confused fd-table exploits.
- **Cross-namespace eventfd refusal in IORING_REGISTER_EVENTFD** — under hardened policy, refuse `IORING_REGISTER_EVENTFD` whose target eventfd was created in a different pidns or user_ns; defense against cross-domain signaling channels.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `uapi/headers/eventfd.md` Tier-5 — header-level read/write/poll semantics
- `fs/eventfd.md` Tier-3 — `struct eventfd_ctx` lifecycle, in-kernel `eventfd_signal`
- `IORING_REGISTER_EVENTFD` — covered in `io_uring_register.md`
- KVM `irqfd` / `ioeventfd` — covered in `uapi/headers/kvm.md`
- Implementation code
