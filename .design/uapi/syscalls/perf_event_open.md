# Tier-5 syscall: perf_event_open(2) — syscall 298

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/events/core.c (SYSCALL_DEFINE5(perf_event_open), perf_event_alloc, perf_install_in_context)
  - kernel/events/ring_buffer.c
  - include/uapi/linux/perf_event.h (struct perf_event_attr, enum perf_type_id, PERF_FLAG_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (298  common  perf_event_open)
-->

## Summary

`perf_event_open(2)` creates a kernel performance event tied to a process (and/or CPU) and returns a file descriptor for reading samples, configuring sampling, attaching BPF, or grouping with sibling events. The `struct perf_event_attr` (extensively versioned) describes type (hardware/raw/software/tracepoint/breakpoint/kprobe/uprobe), config (which counter), sampling parameters, and enable state. `pid`/`cpu`/`group_fd` together select scope (per-task vs per-cpu vs grouped).

`perf_event_open(2)` is the kernel's universal counter/profiler/tracer entry, foundational for `perf record/stat/top`, BPF tracing (`bpf_perf_event_output`), the LTTng/Ftrace integration, ARM CoreSight, Intel PT, IBS, AMD SEV-SNP debug. Critical for: profiling, observability, security tracing, intel CET / IBT control-flow telemetry.

## Signature

```c
int perf_event_open(struct perf_event_attr *attr, pid_t pid, int cpu,
                    int group_fd, unsigned long flags);
```

```c
#define PERF_FLAG_FD_NO_GROUP   0x01
#define PERF_FLAG_FD_OUTPUT     0x02
#define PERF_FLAG_PID_CGROUP    0x04
#define PERF_FLAG_FD_CLOEXEC    0x08
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `attr` | `struct perf_event_attr *` | in/out | Event description; first 4 bytes carry `size` for forward-compat. |
| `pid` | `pid_t` | in | Target task. `-1` = any task (per-CPU mode); `0` = current; positive = specific PID. With `PERF_FLAG_PID_CGROUP`, `pid` is a cgroup fd. |
| `cpu` | `int` | in | Target CPU. `-1` = any CPU (per-task mode). |
| `group_fd` | `int` | in | `-1` = standalone or group leader. ≥ 0 = fd of existing event to join as group member. |
| `flags` | `unsigned long` | in | Bitwise OR of `PERF_FLAG_*`; unknown bits → `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | New fd backed by `perf_fops`; `mmap` for ringbuf, `read` for counter value, `ioctl` for enable/disable. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_PERFMON` (or `CAP_SYS_ADMIN` legacy) for the requested event class; `kernel.perf_event_paranoid` denies. |
| `EFAULT` | `attr` user pointer faults. |
| `E2BIG` | `attr.size` larger than kernel's `sizeof(perf_event_attr)` AND trailing bytes non-zero. |
| `EINVAL` | Bad `attr.type`/`config`/`sample_type`; mutually exclusive flags; `pid` and `cpu` both -1; `group_fd` not a perf fd; unknown PERF_FLAG bit; per-CPU event from non-init userns. |
| `EACCES` | Requested counter not available for unprivileged callers (e.g., tracepoint without CAP_PERFMON). |
| `EMFILE` | Process fd limit. |
| `ENFILE` | System-wide fd limit. |
| `ENOMEM` | Out of memory allocating event / ringbuf. |
| `ENODEV` | `cpu` not present; PMU absent for `attr.type`. |
| `EBUSY` | Hardware counter constraint cannot satisfy event group. |
| `EOPNOTSUPP` | Event class not implemented on this kernel/arch. |
| `ESRCH` | `pid` does not exist. |
| `EAGAIN` | Transient resource race (rare). |

## ABI surface

```text
__NR_perf_event_open  (x86_64)  = 298
__NR_perf_event_open  (arm64)   = 241
__NR_perf_event_open  (riscv)   = 241
__NR_perf_event_open  (i386)    = 336

/* attr.size carries the caller's struct size for forward-compat;
   kernel uses min(attr.size, sizeof(perf_event_attr)) for the copy
   and validates trailing bytes are zero. */
/* perf_event_paranoid sysctl: -1 .. 3 controls unprivileged access. */
```

## Compatibility contract

REQ-1: Syscall number is **298** on x86_64. ABI-stable.

REQ-2: `flags & ~(FD_NO_GROUP | FD_OUTPUT | PID_CGROUP | FD_CLOEXEC)` non-zero → `-EINVAL`.

REQ-3: `copy_from_user(&kattr, attr, attr.size)` with `kattr.size = min(attr.size, sizeof kattr)`; trailing kernel fields zeroed.

REQ-4: Forward-compat probe: if `attr.size > sizeof(perf_event_attr)`, kernel verifies bytes `[sizeof(perf_event_attr), attr.size)` are zero; non-zero → `-E2BIG`.

REQ-5: `attr.size < PERF_ATTR_SIZE_VER0 (64)` → `-EINVAL`.

REQ-6: Type/config validation:
- `PERF_TYPE_HARDWARE`: `config` ∈ `PERF_COUNT_HW_*`.
- `PERF_TYPE_SOFTWARE`: `config` ∈ `PERF_COUNT_SW_*`.
- `PERF_TYPE_TRACEPOINT`: `config` = tracepoint id (from `/sys/kernel/tracing/events`).
- `PERF_TYPE_HW_CACHE`: `config` encoded triple (cache, op, result).
- `PERF_TYPE_RAW`: arch-specific raw event code.
- `PERF_TYPE_BREAKPOINT`: `bp_type`/`bp_addr`/`bp_len` fields populated.
- Dynamic PMUs (`>= PERF_TYPE_MAX`): looked up by id.

REQ-7: PID/CPU scoping:
- `pid == -1, cpu == -1` → `-EINVAL` (must scope at least one).
- `pid >= 0, cpu == -1` → per-task, all CPUs.
- `pid == -1, cpu >= 0` → per-CPU, all tasks (requires CAP_PERFMON).
- `pid >= 0, cpu >= 0` → per-task on specific CPU.
- `PERF_FLAG_PID_CGROUP` → `pid` is fd to a cgroup directory; event scoped to that cgroup.

REQ-8: Group semantics:
- `group_fd == -1`: standalone or new group leader.
- `group_fd >= 0`: member of that event's group; group leader determines CPU/PID scope (must match).
- `PERF_FLAG_FD_NO_GROUP`: do not group even though group_fd given.
- `PERF_FLAG_FD_OUTPUT`: redirect this event's ringbuf to `group_fd`'s ringbuf (multiplex).

REQ-9: `kernel.perf_event_paranoid` sysctl:
- -1: unprivileged users may access any event including kernel ones.
- 0: unprivileged may not access raw tracepoints (but everything else OK).
- 1 (default): unprivileged may not access kernel/kernel-pointer tracepoints; no per-CPU profiling.
- 2: unprivileged may not access kernel profile data; no kprobes/uprobes.
- 3 (hardened): unprivileged may not call perf_event_open at all.

REQ-10: `CAP_PERFMON` (or `CAP_SYS_ADMIN` legacy) required for:
- Per-CPU events (`pid == -1`).
- Raw events (`PERF_TYPE_RAW`).
- Tracepoint events (`PERF_TYPE_TRACEPOINT`).
- Kprobe/uprobe events (dynamic PMUs).
- `attr.exclude_kernel == 0`.

REQ-11: `attr.precise_ip` requires PEBS/IBS hardware support; high values (3) require CAP_PERFMON.

REQ-12: Hardware-counter constraint: event groups must fit in PMU counter slots; over-subscription → `-EBUSY` at the leader's `ioctl(PERF_EVENT_IOC_ENABLE)` time (or at open if pre-scheduling fails).

REQ-13: `PERF_FLAG_FD_CLOEXEC`: fd installed with close-on-exec set.

REQ-14: `PERF_FLAG_FD_OUTPUT`: event's samples written to group_fd's mmap'd ringbuf; the event has no ringbuf of its own.

REQ-15: After successful open, event is in DISABLED state unless `attr.disabled == 0`; enable via `ioctl(PERF_EVENT_IOC_ENABLE)` or `attr.enable_on_exec`.

## Acceptance Criteria

- [ ] AC-1: `perf_event_open(HW, INSTRUCTIONS, 0, -1, -1, 0)` returns positive fd; `read(fd)` returns u64 instruction count.
- [ ] AC-2: `pid == -1, cpu == -1`: `-EINVAL`.
- [ ] AC-3: Unprivileged + `pid == -1, cpu == 0`, paranoid=1: `-EACCES`.
- [ ] AC-4: Unprivileged + `paranoid=3`: any call returns `-EPERM`.
- [ ] AC-5: `attr.size = 0`: `-EINVAL`.
- [ ] AC-6: `attr.size = sizeof(perf_event_attr) + 4` with non-zero tail: `-E2BIG`.
- [ ] AC-7: Unknown PERF_FLAG bit: `-EINVAL`.
- [ ] AC-8: `group_fd` not a perf fd: `-EINVAL`.
- [ ] AC-9: `group_fd` belongs to a different cpu/pid scope: `-EINVAL`.
- [ ] AC-10: `PERF_FLAG_PID_CGROUP` without `CAP_PERFMON`: `-EPERM`.
- [ ] AC-11: `attr.type = PERF_TYPE_BREAKPOINT` with valid `bp_addr`/`bp_type`: returns positive fd.
- [ ] AC-12: Tracepoint event without `CAP_PERFMON` and paranoid≥2: `-EPERM`.
- [ ] AC-13: Two hardware counter events grouped exceeding PMU slots: leader enable returns `-EBUSY`.

## Architecture

```rust
#[syscall(nr = 298, abi = "sysv")]
pub fn sys_perf_event_open(attr: UserPtr<PerfEventAttr>, pid: i32, cpu: i32, group_fd: i32, flags: u64) -> isize {
    Perf::do_perf_event_open(attr, pid, cpu, group_fd, flags)
}
```

`Perf::do_perf_event_open(attr_ptr, pid, cpu, group_fd, flags) -> isize`:
1. const VALID = PERF_FLAG_FD_NO_GROUP | PERF_FLAG_FD_OUTPUT | PERF_FLAG_PID_CGROUP | PERF_FLAG_FD_CLOEXEC;
2. if (flags & !VALID) != 0 { return -EINVAL; }
3. let kattr = Perf::copy_attr_from_user(attr_ptr)?;        // EFAULT / E2BIG / EINVAL
4. Perf::sanitize_attr(&mut kattr)?;                        // EINVAL
5. Perf::check_paranoid(&kattr, pid, cpu, &current_creds())?;  // EPERM / EACCES
6. if pid == -1 && cpu == -1 { return -EINVAL; }
7. let target_task = match pid {
8.   -1 => None,
9.    0 => Some(current()),
10.   p => Some(find_task_by_vpid(p).ok_or(ESRCH)?),
11. };
12. let target_cgroup = if (flags & PERF_FLAG_PID_CGROUP) != 0 {
13.   Some(Perf::cgroup_fd_to_css(pid)?)
14. } else { None };
15. let group = if group_fd >= 0 && (flags & PERF_FLAG_FD_NO_GROUP) == 0 {
16.   Some(Perf::resolve_group_leader(group_fd, &target_task, cpu)?)  // EINVAL
17. } else { None };
18. let pmu = Perf::find_pmu(&kattr).ok_or(ENODEV)?;
19. let event = Perf::event_alloc(&kattr, target_task, cpu, group, pmu)?;
20. if (flags & PERF_FLAG_FD_OUTPUT) != 0 {
21.   Perf::redirect_ringbuf(&event, group_fd)?;
22. }
23. Perf::install_in_context(&event)?;                      // EBUSY
24. let fd_flags = if (flags & PERF_FLAG_FD_CLOEXEC) != 0 { O_CLOEXEC } else { 0 };
25. let fd = anon_inode_getfd("perf_event", &perf_fops, &event, fd_flags)?;
26. fd as isize

`Perf::copy_attr_from_user(uptr) -> Result<PerfEventAttr>`:
1. let mut size_hdr = [0u8; 4];
2. uptr.cast::<u8>().copy_in_partial(&mut size_hdr, 4)?;
3. let size = u32::from_ne_bytes(size_hdr) as usize;
4. if size < PERF_ATTR_SIZE_VER0 { return Err(EINVAL); }
5. const K: usize = size_of::<PerfEventAttr>();
6. let n = min(size, K);
7. let mut kattr = PerfEventAttr::zeroed();
8. unsafe { uptr.cast::<u8>().copy_in_partial(&mut kattr as *mut _ as *mut [u8; K], n)?; }
9. if size > K {
10.   let mut probe = [0u8; 64];
11.   for off in step_by(size - K, 64) {
12.     let n = min(64, size - K - off);
13.     uptr.cast::<u8>().add(K + off).copy_in_partial(&mut probe[..n])?;
14.     if probe[..n].iter().any(|b| *b != 0) { return Err(E2BIG); }
15.   }
16. }
17. Ok(kattr)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | unknown bit ⟹ EINVAL pre-copy. |
| `attr_size_bounded` | INVARIANT | size ≥ PERF_ATTR_SIZE_VER0 and tail-zero check. |
| `pid_cpu_disjoint` | INVARIANT | (pid==-1, cpu==-1) ⟹ EINVAL. |
| `paranoid_enforced` | INVARIANT | paranoid level gates per-required cap. |
| `group_scope_matches` | INVARIANT | grouped event's cpu/pid matches leader. |
| `no_fd_before_install` | INVARIANT | fd installed only after event valid in context. |

### Layer 2: TLA+

`kernel/events/perf-open.tla`:
- States: per-flag-validate, per-attr-copy, per-sanitize, per-paranoid, per-scope, per-group-resolve, per-pmu-find, per-alloc, per-install, per-fd-install.
- Properties:
  - `safety_paranoid_3_unprivileged_blocked`.
  - `safety_group_scope_consistent`.
  - `safety_per_cpu_requires_perfmon`.
  - `safety_attr_tail_zero`.
  - `liveness_open_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_perf_event_open` post: ret ≥ 0 ⟹ event installed | `Perf::do_perf_event_open` |
| `copy_attr_from_user` post: kattr ≤ K bytes, padded zero | `Perf::copy_attr_from_user` |
| `check_paranoid` post: caller cred satisfies paranoid level | `Perf::check_paranoid` |
| `resolve_group_leader` post: leader.cpu == event.cpu, leader.pid == event.pid | `Perf::resolve_group_leader` |

### Layer 4: Verus / Creusot functional

Per-`perf_event_open(2)` man-page and per-`kernel/events/core.c` semantic equivalence. perf-tools selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`perf_event_open(2)` reinforcement:

- **Per-attr-size validation** — defense against per-extension-field smuggling.
- **Per-paranoid sysctl** — defense against per-unprivileged-counter-leak.
- **Per-CAP_PERFMON gate on kernel-scope events** — defense against per-symbol-leak via tracepoint.
- **Per-group scope match** — defense against per-cross-context sample injection.
- **Per-fd CLOEXEC default under hardened mode** — defense against per-fd-leak across exec.

## Grsecurity / PaX surface

- **PaX UDEREF on `attr` copy_from_user** — defense against per-attr-pointer kernel-deref bug; SMAP forced for the size-bounded copy.
- **CAP_PERFMON strict in init_user_ns** — child-userns CAP_PERFMON ignored; per-CPU and tracepoint events from non-init userns rejected with `-EPERM`. GRKERNSEC_CHROOT_CAPS extends this to chroot.
- **GRKERNSEC_PERF_HARDEN** — under hardened mode, `kernel.perf_event_paranoid = 3` is forced and write-locked; unprivileged perf_event_open returns `-EPERM` unconditionally. Hardware-event self-counting (e.g., own-process instruction counter) still permitted via `CAP_PERFMON`.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` (or higher), kernel-scope tracepoint / kprobe / uprobe / raw-event opens return `-EPERM` even with `CAP_PERFMON`. Defense against per-side-channel kernel-data extraction.
- **GRKERNSEC_PERF_NO_DYNAMIC_EVENTS** — dynamic PMU types (kprobe/uprobe via `/sys/bus/event_source`) blocked; only built-in PMUs permitted.
- **PAX_USERCOPY_HARDEN on attr copy_from_user** — bounded copy via whitelisted bounce buffer.
- **PaX KERNEXEC on perf_fops vtable** — defense against per-vtable corruption injecting custom mmap/ioctl handlers.
- **GRKERNSEC_HIDESYM on tracepoint addresses** — `/sys/kernel/tracing/events/*/id` exposed only to root; kernel-pointer fields in samples scrubbed for unprivileged callers regardless of `attr.exclude_kernel`.
- **PAX_REFCOUNT on perf_event refcount** — defense against per-refcount-overflow UAF on event group teardown.
- **GRKERNSEC_AUDIT_PERF** — every perf_event_open logged with PID, comm, attr.type, attr.config, pid, cpu, paranoid level, return code; tracepoint and raw events audit-flagged.
- **Per-PMU scheduler isolation** — under hardened mode, per-CPU events cannot intercept hyperthread-sibling counters (defense against SMT side-channel via perf counters).
- **GRKERNSEC_KSTACKOVERFLOW** — defense against ringbuf-callback stack overflow.
- **Sample-data sanitization** — even for `CAP_PERFMON`, sample-data structures (PERF_SAMPLE_REGS_INTR, PERF_SAMPLE_STACK_USER) scrubbed of KASLR-revealing bits before delivery; `bpf_perf_event_output` similarly gated.
- **GRKERNSEC_BPF_HARDEN correlation** — attaching BPF to a perf event additionally requires `CAP_BPF`; unprivileged BPF + perf path closed.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-PMU driver code (covered in Tier-3 `arch/*/events/*.md`).
- Ringbuffer mmap layout (covered in Tier-3 `kernel/events/ring_buffer.md`).
- Tracepoint/kprobe/uprobe wiring (covered in Tier-3 `kernel/trace/*.md`).
- Implementation code.
