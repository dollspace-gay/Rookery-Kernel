---
title: "Tier-5 syscall: sysinfo(2) — syscall 99"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sysinfo(2)` returns a snapshot of overall system statistics: uptime, load averages, memory and swap totals/availability, shared-memory totals, number of running processes, and the memory-unit scaling factor. The kernel fills a single `struct sysinfo` buffer in one syscall, no iteration. Per-fields use opaque `unsigned long` and a `mem_unit` exponent that lets the kernel report huge memory totals on 32-bit ABIs without overflowing 32-bit longs.

Critical for: `uptime(1)`, `free(1)`, `top(1)` fallback, container introspection without `/proc` mount, and any tool that needs a syscall-only memory/load snapshot. Long predates `/proc/meminfo` as the historic UNIX way.

### Acceptance Criteria

- [ ] AC-1: `sysinfo(&i)` returns 0; `i.uptime` non-zero on a running system.
- [ ] AC-2: `i.loads[0]` / 65536 matches `/proc/loadavg` first column within rounding.
- [ ] AC-3: `i.totalram * i.mem_unit` matches `/proc/meminfo MemTotal` within rounding.
- [ ] AC-4: `i.freeram + i.bufferram + cached` approximates MemAvailable.
- [ ] AC-5: `i.totalswap` matches `/proc/swaps` aggregate.
- [ ] AC-6: `info = NULL` → `-EFAULT`.
- [ ] AC-7: `info` user pointer crosses page boundary into unmapped region → `-EFAULT` and `info` unmodified.
- [ ] AC-8: On 64-bit kernel, `i.mem_unit == 1`.
- [ ] AC-9: `i._f` reserved bytes are zero.
- [ ] AC-10: Sequential calls within 1s: `uptime` monotonic non-decreasing.

### Architecture

```rust
#[syscall(nr = 99, abi = "sysv")]
pub fn sys_sysinfo(info: UserPtrMut<SysInfo>) -> isize {
    Sched::do_sysinfo(info)
}
```

`Sched::do_sysinfo(info) -> isize`:
1. let mut si = SysInfo::zeroed();
2. /* Uptime */
3. si.uptime = ktime_get_boottime_seconds() as i64;
4. /* Load averages (fixed-point F(16)) */
5. let (l1, l5, l15) = sched_avenrun();
6. si.loads = [l1, l5, l15];
7. /* Memory */
8. Mm::si_meminfo(&mut si);
9. Swap::si_swapinfo(&mut si);
10. /* Process count */
11. let n = nr_threads();
12. si.procs = if n > u16::MAX as i64 { u16::MAX } else { n as u16 };
13. /* Determine mem_unit scaling */
14. si.mem_unit = Sched::compute_mem_unit(&si);
15. if si.mem_unit > 1 { Sched::scale_counters(&mut si); }
16. /* Padding already zero by zeroed() init */
17. info.copy_out(&si)?;                       // EFAULT
18. 0

`Mm::si_meminfo(si) -> ()`:
1. si.totalram   = totalram_pages() as u64 * PAGE_SIZE as u64;
2. si.freeram    = global_zone_page_state(NR_FREE_PAGES) as u64 * PAGE_SIZE as u64;
3. si.sharedram  = global_node_page_state(NR_SHMEM) as u64 * PAGE_SIZE as u64;
4. si.bufferram  = nr_blockdev_pages() as u64 * PAGE_SIZE as u64;
5. si.totalhigh  = totalhigh_pages() as u64 * PAGE_SIZE as u64;
6. si.freehigh   = nr_free_highpages() as u64 * PAGE_SIZE as u64;

`Swap::si_swapinfo(si) -> ()`:
1. let (total, free) = swap_totals();
2. si.totalswap = total * PAGE_SIZE as u64;
3. si.freeswap = free * PAGE_SIZE as u64;

`Sched::compute_mem_unit(si) -> u32`:
1. /* 64-bit: no scaling needed */
2. if size_of::<usize>() == 8 { return 1; }
3. /* 32-bit: scale until largest fits in u32 */
4. let max = max(si.totalram, max(si.totalswap, si.totalhigh));
5. let mut unit: u32 = 1;
6. while max / (unit as u64) > u32::MAX as u64 { unit <<= 1; }
7. unit

### Out of Scope

- `/proc/meminfo`, `/proc/loadavg`, `/proc/uptime` (covered in proc Tier-3).
- Per-cgroup memory accounting (covered in cgroup memcg Tier-3).
- `getloadavg(3)` glibc wrapper (userspace).
- `sysctl(2)` (covered separately under sysctl Tier-5).
- Implementation code.

### signature

```c
int sysinfo(struct sysinfo *info);
```

```c
struct sysinfo {
    __kernel_long_t uptime;        /* seconds since boot */
    __kernel_ulong_t loads[3];     /* 1, 5, 15 minute load averages (fixed-point F(15)) */
    __kernel_ulong_t totalram;     /* total usable main memory */
    __kernel_ulong_t freeram;      /* available memory */
    __kernel_ulong_t sharedram;    /* shared memory */
    __kernel_ulong_t bufferram;    /* memory used by buffers */
    __kernel_ulong_t totalswap;    /* total swap space */
    __kernel_ulong_t freeswap;     /* swap available */
    __u16            procs;        /* number of current processes */
    __u16            pad;          /* explicit padding */
    __kernel_ulong_t totalhigh;    /* total high memory */
    __kernel_ulong_t freehigh;     /* available high memory */
    __u32            mem_unit;     /* memory unit size in bytes */
    char             _f[20 - 2*sizeof(__kernel_ulong_t) - sizeof(__u32)]; /* padding */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `info` | `struct sysinfo *` | out | Caller buffer; kernel fills 16 fields + padding. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `info` user pointer faults during copy_to_user. |

### abi surface

```text
__NR_sysinfo (x86_64) =  99
__NR_sysinfo (arm64)  = 179
__NR_sysinfo (riscv)  = 179
__NR_sysinfo (i386)   = 116

/* struct sysinfo size is 64-bit-arch dependent (~112 B on x86_64). */
/* Reserved tail `_f` exists for forward compat; always zeroed. */
```

### compatibility contract

REQ-1: Syscall number is **99** on x86_64. ABI-stable since 1.0.

REQ-2: `uptime` is `ktime_get_boottime_seconds()` (excludes time during which the system was off; includes suspend duration).

REQ-3: `loads[0..2]` are kernel-internal load averages over 1 / 5 / 15 minutes, in fixed-point with `SI_LOAD_SHIFT = 16`. Userspace divides by `1 << SI_LOAD_SHIFT` (== 65536) to get the float value.

REQ-4: `totalram`, `freeram`, `sharedram`, `bufferram`: filled by `si_meminfo()` from page-allocator counters. Units = pages × `PAGE_SIZE`, then scaled by `mem_unit`.

REQ-5: `totalswap`, `freeswap`: filled by `si_swapinfo()` from swap subsystem counters; same scaling.

REQ-6: `totalhigh`, `freehigh`: highmem totals (32-bit relevant; always 0 on 64-bit). Same scaling.

REQ-7: `procs`: number of forked tasks (signal-group leaders) reported by `nr_threads` / `current_thread_count()`; bounded to `__u16` (clipped at 65535).

REQ-8: `mem_unit`: when `PAGE_SIZE * largest-counter` would overflow `unsigned long`, the kernel sets `mem_unit > 1` (power-of-two scaling factor) and divides counters accordingly. On 64-bit, `mem_unit = 1` (no scaling).

REQ-9: Compat 32-bit ABI: `compat_sysinfo` translates `struct compat_sysinfo` with `compat_ulong_t`; mem_unit scaling triggers more aggressively to keep counters within 32-bit longs.

REQ-10: Per-namespace: the `loads`/`procs`/memory totals are GLOBAL (not per-pidns / per-memcg) in current upstream. (A container-aware syscall would return host-wide figures.)

REQ-11: Per-locking: counters are read RCU-fast / lockless wherever possible (`si_meminfo` uses per-zone atomic reads); the call is intended to be cheap.

REQ-12: Per-`info->_f` padding bytes: explicitly zeroed to prevent kernel-stack info-leak.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `info_pad_zero` | INVARIANT | `_f` and pad bytes always zero in copy_to_user. |
| `mem_unit_pow2` | INVARIANT | mem_unit is power-of-two. |
| `uptime_non_negative` | INVARIANT | uptime >= 0. |
| `procs_clamped_u16` | INVARIANT | procs <= u16::MAX. |
| `counters_consistent_with_unit` | INVARIANT | counter * mem_unit equals raw byte count (modulo scale rounding). |

### Layer 2: TLA+

`kernel/sysinfo.tla`:
- States: per-call read-uptime, read-loadavg, read-meminfo, read-swapinfo, scale, copy_out.
- Properties:
  - `safety_no_info_leak` — `_f` zero on every copy_out.
  - `safety_uptime_monotonic` — sequential calls non-decreasing uptime.
  - `safety_consistent_scale` — totalram, freeram, totalswap all use the same mem_unit.
  - `liveness_terminates` — call always returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_sysinfo` post: returns 0 ∨ -EFAULT | `Sched::do_sysinfo` |
| `si_meminfo` post: counters non-negative | `Mm::si_meminfo` |
| `compute_mem_unit` post: result is power-of-two | `Sched::compute_mem_unit` |

### Layer 4: Verus / Creusot functional

Per-`sysinfo(2)` man-page semantic equivalence. Cross-check with `/proc/meminfo`, `/proc/loadavg`, `/proc/uptime`, `/proc/swaps` agreement within rounding.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sysinfo(2)` reinforcement:

- **Per-`_f` reserved bytes zeroed** — defense against per-kernel-stack info-leak.
- **Per-`procs` clamped at u16::MAX** — defense against per-counter overflow exposing internal task-count.
- **Per-`mem_unit` power-of-two** — defense against per-scale ambiguity.
- **Per-`copy_to_user` all-or-nothing** — defense against per-partial-write info-leak.
- **Per-fast-path lockless reads** — defense against per-DoS-by-cheap-syscall-poll.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `info` copy_to_user** — defense against per-user-pointer kernel-deref; SMAP forced.
- **GRKERNSEC_PROC_GETPID** — load-average and process-count fields filtered for unprivileged callers; non-root sees `procs` clamped at the count of the caller's own pid-namespace, and `loads[]` quantized to 1-decimal precision. Defense against per-system-fingerprinting info-leak and side-channels that infer host activity from load-avg fluctuation.
- **GRKERNSEC_HIDESYM on sched_clock-based fields** — `uptime` rounded to nearest 60s for unprivileged callers; defense against per-uptime-fingerprinting and per-boot-time correlation with public uptime trackers.
- **PAX_USERCOPY_HARDEN on `struct sysinfo` copy_to_user** — bounded copy uses whitelisted slab region.
- **Per-`totalram` / `freeram` quantization under hardened policy** — for non-root callers in a userns, memory totals are reported at 64MB granularity; defense against per-memory-fingerprinting / VM-detection side-channel.
- **GRKERNSEC_NO_GETPID-style hostname/system-info hiding** — the `loads[]` / `procs` fields cannot be used to fingerprint the host across containers; defense against per-container escape-recon.
- **PAX_REFCOUNT on swap subsystem counter reads** — defense against per-swap-counter UAF during concurrent swapoff(2).
- **Per-call rate-limit (token-bucket per-UID)** — defense against per-DoS via sysinfo polling (each `sysinfo` traverses every NUMA node for `si_meminfo`).
- **PaX KERNEXEC on `si_meminfo` inlined helpers** — defense against per-W^X violation.
- **GRKERNSEC_NO_GETPID consistency** — `procs` is per-pidns when the caller is in a non-init pidns; defense against per-container host-task-count leak.

