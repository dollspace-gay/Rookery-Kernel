---
title: "Tier-5 UAPI: include/uapi/linux/perf_event.h — perf_event_open(2) ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`perf_event_open(2)` is the kernel's universal sampling / counting interface. A caller fills a `struct perf_event_attr` (type + config + sample/read formats + flag bitfields) and gets back a file descriptor that represents one event channel. The channel can be read (`read(2)` returns counts per `read_format`), polled (epoll wakes on overflow), mmap'd (a header page followed by a ring buffer of `PERF_RECORD_*` packets, optionally followed by an AUX area for hardware-trace payloads such as Intel PT / CoreSight), and controlled by ioctls (`PERF_EVENT_IOC_ENABLE` / `_DISABLE` / `_REFRESH` / `_RESET` / `_PERIOD` / `_SET_OUTPUT` / `_SET_FILTER` / `_ID` / `_SET_BPF` / `_PAUSE_OUTPUT` / `_QUERY_BPF` / `_MODIFY_ATTRIBUTES`). The `type` selects an event class — `PERF_TYPE_HARDWARE` / `_SOFTWARE` / `_TRACEPOINT` / `_HW_CACHE` / `_RAW` / `_BREAKPOINT` — and `config` (plus `config1` / `config2` / `config3` / `config4`) names the specific event within that class. Critical for: `perf`, `oprofile`, BPF tracing, JIT profilers, hardware-trace tools, all CPU-pinning observability, and userspace `rdpmc` self-monitoring.

This Tier-5 covers the full UAPI surface of `include/uapi/linux/perf_event.h` (~1528 lines).

### Acceptance Criteria

- [ ] AC-1: `attr.size` < kernel base ⇒ kernel zero-extends; `attr.size > kernel current` ⇒ -E2BIG.
- [ ] AC-2: `attr.type == PERF_TYPE_HARDWARE` + `config = PERF_COUNT_HW_CPU_CYCLES` ⇒ fd opens; `read()` returns u64 count per `read_format`.
- [ ] AC-3: `attr.type == PERF_TYPE_SOFTWARE` + `config = PERF_COUNT_SW_PAGE_FAULTS` ⇒ counts page faults of target.
- [ ] AC-4: `attr.type == PERF_TYPE_TRACEPOINT` + `config = <id>` ⇒ requires CONFIG_TRACEPOINTS; with `kernel.perf_event_paranoid >= 2` and no CAP_PERFMON ⇒ -EACCES.
- [ ] AC-5: `attr.type == PERF_TYPE_HW_CACHE` + `config = L1D | (READ << 8) | (MISS << 16)` ⇒ counts L1-D read misses.
- [ ] AC-6: `attr.type == PERF_TYPE_RAW` + arbitrary `config` ⇒ counts the raw PMU event; no kernel translation.
- [ ] AC-7: `attr.type == PERF_TYPE_BREAKPOINT` + `bp_type = HW_BREAKPOINT_W` + `bp_addr` + `bp_len = 4` ⇒ overflows on write to that 4-byte region.
- [ ] AC-8: `attr.sample_type & PERF_SAMPLE_IP` ⇒ SAMPLE records carry `ip`; `precise_ip` controls skid.
- [ ] AC-9: `attr.sample_type & PERF_SAMPLE_CALLCHAIN` ⇒ SAMPLE carries `nr, ips[nr]` with `PERF_CONTEXT_*` markers.
- [ ] AC-10: `attr.sample_type & PERF_SAMPLE_BRANCH_STACK` + `attr.branch_sample_type` non-zero ⇒ SAMPLE carries LBR entries; without branch_sample_type ⇒ -EINVAL.
- [ ] AC-11: `attr.read_format & PERF_FORMAT_GROUP` + `read(leader_fd)` ⇒ returns `{nr, [{value, id?, lost?}] [nr]}`; read of non-leader with GROUP ⇒ -EINVAL.
- [ ] AC-12: `attr.read_format & PERF_FORMAT_TOTAL_TIME_ENABLED/_RUNNING` ⇒ enables scaling.
- [ ] AC-13: `attr.disabled = 1` ⇒ event starts disabled; `IOC_ENABLE` activates.
- [ ] AC-14: `attr.inherit = 1` ⇒ child of monitored task receives a clone of the event.
- [ ] AC-15: `attr.pinned = 1` ⇒ event always counts; if PMU oversubscribed ⇒ event reports `PERF_EVENT_STATE_ERROR`.
- [ ] AC-16: `attr.exclude_user = 1` ⇒ no user-mode samples emitted.
- [ ] AC-17: `attr.freq = 1` + `sample_freq = 1000` ⇒ kernel auto-tunes period to ~1000 Hz; bounded by sysctl.
- [ ] AC-18: `attr.mmap = 1` ⇒ PERF_RECORD_MMAP emitted on PROT_EXEC mappings; `mmap2 = 1` ⇒ MMAP2 with maj/min/ino.
- [ ] AC-19: `attr.task = 1` ⇒ FORK + EXIT records emitted.
- [ ] AC-20: `attr.comm_exec = 1` ⇒ exec-caused COMM records carry MISC_COMM_EXEC bit.
- [ ] AC-21: `attr.context_switch = 1` ⇒ SWITCH records emitted with MISC_SWITCH_OUT for outgoing.
- [ ] AC-22: `attr.sigtrap = 1` ⇒ overflow delivers synchronous SIGTRAP; `siginfo_t::si_perf_data == attr.sig_data`.
- [ ] AC-23: `attr.remove_on_exec = 1` ⇒ execve removes the event (security isolation).
- [ ] AC-24: `mmap()` returns header page; `mmap_page.lock` is a seqlock; reader retries on odd-or-changed.
- [ ] AC-25: `cap_user_rdpmc == 1` + `index > 0` ⇒ `rdpmc(index - 1) << (64-width) >> (64-width)` yields signed delta.
- [ ] AC-26: `cap_user_time == 1` ⇒ time-delta formula yields ns from cyc/TSC.
- [ ] AC-27: `data_head` updated by kernel with implicit smp_wmb; reader smp_rmb before consuming; `data_tail` updated by reader with smp_mb.
- [ ] AC-28: `aux_offset >= data_offset + data_size`; AUX area mmap'd separately at `aux_offset`.
- [ ] AC-29: `PERF_EVENT_IOC_ENABLE(PERF_IOC_FLAG_GROUP)` enables every event in the group.
- [ ] AC-30: `PERF_EVENT_IOC_DISABLE` disables sampling/counting.
- [ ] AC-31: `PERF_EVENT_IOC_REFRESH(N)` arms event for N more overflows then auto-disables.
- [ ] AC-32: `PERF_EVENT_IOC_RESET` zeroes the counter.
- [ ] AC-33: `PERF_EVENT_IOC_PERIOD(period)` ⇒ next overflow uses new period.
- [ ] AC-34: `PERF_EVENT_IOC_SET_OUTPUT(sibling_fd)` ⇒ this event's records emitted into sibling's ring; `arg = -1` detaches.
- [ ] AC-35: `PERF_EVENT_IOC_SET_FILTER("comm==foo")` on a tracepoint event ⇒ filters records.
- [ ] AC-36: `PERF_EVENT_IOC_ID(&id)` ⇒ writes event's unique id matching `sample.id`.
- [ ] AC-37: `PERF_EVENT_IOC_SET_BPF(prog_fd)` attaches BPF prog; prog of wrong type ⇒ -EINVAL.
- [ ] AC-38: `PERF_EVENT_IOC_PAUSE_OUTPUT(1)` pauses ring-buffer writes; (0) resumes.
- [ ] AC-39: `PERF_EVENT_IOC_QUERY_BPF` with `ids_len < prog_cnt` ⇒ -ENOSPC, `prog_cnt` filled.
- [ ] AC-40: `PERF_EVENT_IOC_MODIFY_ATTRIBUTES(&new_attr)` modifies allowed fields live.
- [ ] AC-41: `PERF_RECORD_MMAP`, `_LOST`, `_COMM`, `_EXIT`, `_THROTTLE`, `_UNTHROTTLE`, `_FORK`, `_READ`, `_SAMPLE`, `_MMAP2`, `_AUX`, `_ITRACE_START`, `_LOST_SAMPLES`, `_SWITCH`, `_SWITCH_CPU_WIDE`, `_NAMESPACES`, `_KSYMBOL`, `_BPF_EVENT`, `_CGROUP`, `_TEXT_POKE`, `_AUX_OUTPUT_HW_ID`, `_CALLCHAIN_DEFERRED` carry the per-record layouts documented in upstream comments.
- [ ] AC-42: `PERF_RECORD_MISC_KERNEL/USER/HYPERVISOR/GUEST_KERNEL/GUEST_USER` bits ≤ 7 in `misc` correctly tag each record.
- [ ] AC-43: `PERF_AUX_FLAG_TRUNCATED/OVERWRITE/PARTIAL/COLLISION` reflect AUX state.
- [ ] AC-44: `PERF_FLAG_FD_CLOEXEC` ⇒ returned fd is O_CLOEXEC.
- [ ] AC-45: `PERF_FLAG_PID_CGROUP` + `pid = cgroup_fd` + `cpu >= 0` ⇒ event scoped to cgroup; without `cpu >= 0` ⇒ -EINVAL.
- [ ] AC-46: `PERF_FLAG_FD_NO_GROUP` + `PERF_FLAG_FD_OUTPUT` + `group_fd` ⇒ event not in group but shares ring.
- [ ] AC-47: `perf_event_paranoid >= 2` and unprivileged caller opening tracepoint ⇒ -EACCES.
- [ ] AC-48: `kptr_restrict >= 1` and unprivileged caller with `PERF_SAMPLE_IP` ⇒ kernel IPs redacted (zero or zeroed in callchain).
- [ ] AC-49: `attr.config` for `PERF_TYPE_BREAKPOINT` ignored; `bp_type/bp_addr/bp_len` define event.
- [ ] AC-50: Forward-compat: kernel writing `attr.size = PERF_ATTR_SIZE_VER9` to userspace with VER0 view ⇒ first 64 bytes correct.

### Architecture

```
enum Type { Hardware = 0, Software = 1, Tracepoint = 2, HwCache = 3, Raw = 4, Breakpoint = 5 }
bitflags! SampleType : u64 { /* PERF_SAMPLE_* */ }
bitflags! BranchSampleType : u64 { /* PERF_SAMPLE_BRANCH_* */ }
bitflags! ReadFormat : u64 { TOTAL_TIME_ENABLED, TOTAL_TIME_RUNNING, ID, GROUP, LOST }
bitflags! SyscallFlags : u64 { FD_NO_GROUP, FD_OUTPUT, PID_CGROUP, FD_CLOEXEC }

struct PerfEventAttr {            // PERF_ATTR_SIZE_VER9 = 144
  type_: u32,
  size: u32,
  config: u64,
  sample_period_or_freq: u64,
  sample_type: u64,
  read_format: u64,
  flags: u64,                     // packed bitfield (disabled..defer_output)
  wakeup_events_or_watermark: u32,
  bp_type: u32,
  bp_addr_or_kprobe_func_or_uprobe_path_or_config1: u64,
  bp_len_or_kprobe_addr_or_probe_offset_or_config2: u64,
  branch_sample_type: u64,
  sample_regs_user: u64,
  sample_stack_user: u32,
  clockid: i32,
  sample_regs_intr: u64,
  aux_watermark: u32,
  sample_max_stack: u16,
  __reserved_2: u16,
  aux_sample_size: u32,
  aux_action: u32,                // packed (aux_start_paused | aux_pause | aux_resume)
  sig_data: u64,
  config3: u64,
  config4: u64,
}

struct PerfEventHeader { type_: u32, misc: u16, size: u16 }

struct PerfEventMmapPage {
  version: u32, compat_version: u32,
  lock: u32, index: u32, offset: i64,
  time_enabled: u64, time_running: u64,
  capabilities: u64,              // bit-packed cap_user_*
  pmc_width: u16, time_shift: u16, time_mult: u32,
  time_offset: u64, time_zero: u64,
  size: u32, __reserved_1: u32,
  time_cycles: u64, time_mask: u64,
  __reserved: [u8; 116 * 8],
  data_head: u64, data_tail: u64, data_offset: u64, data_size: u64,
  aux_head: u64, aux_tail: u64, aux_offset: u64, aux_size: u64,
}

struct PerfBranchEntry { from: u64, to: u64, flags: u64 }   // bit-packed
union PerfSampleWeight { full: u64, parts: WeightParts }
struct WeightParts { var1_dw: u32, var2_w: u16, var3_w: u16 }
union PerfMemDataSrc { val: u64, parts: MemDataSrcParts }   // bit-packed per header

struct PerfEventQueryBpf { ids_len: u32, prog_cnt: u32, ids: [u32; 0] /* trailing flex */ }

mod ioctl {
  const MAGIC: u32 = b'$' as u32;
  pub const ENABLE: u32             = _IO  (MAGIC,  0);
  pub const DISABLE: u32            = _IO  (MAGIC,  1);
  pub const REFRESH: u32            = _IO  (MAGIC,  2);
  pub const RESET: u32              = _IO  (MAGIC,  3);
  pub const PERIOD: u32             = _IOW (MAGIC,  4, u64);
  pub const SET_OUTPUT: u32         = _IO  (MAGIC,  5);
  pub const SET_FILTER: u32         = _IOW (MAGIC,  6, *const u8);
  pub const ID: u32                 = _IOR (MAGIC,  7, *mut u64);
  pub const SET_BPF: u32            = _IOW (MAGIC,  8, u32);
  pub const PAUSE_OUTPUT: u32       = _IOW (MAGIC,  9, u32);
  pub const QUERY_BPF: u32          = _IOWR(MAGIC, 10, *mut PerfEventQueryBpf);
  pub const MODIFY_ATTRIBUTES: u32  = _IOW (MAGIC, 11, *const PerfEventAttr);
}
```

`Perf::open(attr_user, pid, cpu, group_fd, flags) -> isize`:
1. Copy `attr.size` from user (4 bytes); validate `0 < size ≤ kernel_max`.
2. Copy `min(size, kernel_max)` bytes; zero-extend remainder.
3. If `size > kernel_max` ⇒ -E2BIG.
4. Validate `attr.type` ∈ `{HARDWARE, SOFTWARE, TRACEPOINT, HW_CACHE, RAW, BREAKPOINT}` or dynamic-PMU id; else -ENOENT.
5. Validate `attr.sample_type ⊆ PERF_SAMPLE_*` mask; `branch_sample_type ⊆ PERF_SAMPLE_BRANCH_*`.
6. Permission check: `perf_event_paranoid` + caller's `CAP_PERFMON | CAP_SYS_ADMIN`.
7. If `flags & PERF_FLAG_PID_CGROUP` and `cpu < 0` ⇒ -EINVAL.
8. Allocate `PerfEvent`; install on target task/cpu/cgroup; attach to group_fd's group unless `FD_NO_GROUP`.
9. If `flags & FD_OUTPUT`: redirect ring buffer to group_fd's.
10. Allocate fd with `O_CLOEXEC` if `FD_CLOEXEC`; return fd.

`Perf::mmap(event, vma) -> Result`:
1. Validate `vma.size` is `page_size + 2^k * page_size` (header + power-of-two data) or pure AUX region.
2. If first mmap (data ring): allocate header + data pages; populate `mmap_page.{version, compat_version, data_offset, data_size}`; set `cap_user_*` based on PMU.
3. If `aux_offset` already set + AUX mmap requested at `aux_offset`: allocate AUX area; populate `mmap_page.aux_{offset, size}`.

`Perf::ioctl(event, cmd, arg)`:
1. Validate `_IOC_TYPE(cmd) == '$'` (0x24); else -ENOTTY.
2. Dispatch on `_IOC_NR(cmd)`: ENABLE / DISABLE / REFRESH / RESET / PERIOD / SET_OUTPUT / SET_FILTER / ID / SET_BPF / PAUSE_OUTPUT / QUERY_BPF / MODIFY_ATTRIBUTES.

`Perf::record_sample(event, sample_data)`:
1. Acquire ring-buffer reservation `[head, head+size)`.
2. Write `PerfEventHeader { type = SAMPLE, misc, size }`.
3. Emit each enabled `PERF_SAMPLE_*` field in the documented order.
4. Atomically publish: smp_wmb(); store `data_head += size`.
5. If `wakeup_events`/`wakeup_watermark` threshold ⇒ wake epoll.
6. If `attr.sigtrap` ⇒ deliver synchronous SIGTRAP with `si_perf_data = attr.sig_data`.

`Perf::read(event, buf, len) -> isize`:
1. If `attr.read_format & GROUP` and event is not group leader ⇒ -EINVAL.
2. Layout per `read_format` flags (non-GROUP or GROUP variant).
3. Copy out; return bytes written.

### Out of Scope

- HW breakpoint constants `HW_BREAKPOINT_R/W/X/EMPTY` and `HW_BREAKPOINT_LEN_*` — covered in `uapi/headers/hw_breakpoint.md`.
- BPF program types and load syscall (`include/uapi/linux/bpf.h`) — covered in `uapi/headers/bpf.md`.
- Tracepoint event-id discovery via `debugfs:tracing/events/*/id` — covered in `uapi/headers/tracefs.md`.
- Architecture-specific regs masks (`asm/perf_regs.h`) — separate per-arch Tier-5 docs.
- Implementation code (`kernel/events/core.c`, `kernel/events/ring_buffer.c`).
- `siginfo_t::si_perf_*` field layout — covered in `uapi/headers/signal.md`.
- `prctl(PR_TASK_PERF_EVENTS_DISABLE / _ENABLE)` — covered in `uapi/headers/prctl.md`.
- `intel_pt` / `arm_spe` / `arm_coresight` PMU-specific AUX format — covered in vendor PMU docs.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum perf_type_id` | per-attr.type major class | `perf::Type` |
| `enum perf_hw_id` | per-HARDWARE event id | `perf::HwId` |
| `enum perf_hw_cache_id / _op_id / _op_result_id` | per-HW_CACHE triple | `perf::HwCache{Id,Op,Result}` |
| `enum perf_sw_ids` | per-SOFTWARE event id | `perf::SwId` |
| `enum perf_event_sample_format` | per-sample-type bitmask | `perf::SampleType` |
| `enum perf_branch_sample_type{_shift}` | per-branch-sample bitmask | `perf::BranchSampleType` |
| `enum perf_sample_regs_abi` | per-regs ABI marker | `perf::RegsAbi` |
| `PERF_TXN_*` | per-transaction qualifier bits | `perf::Txn` |
| `enum perf_event_read_format` | per-`read()` layout | `perf::ReadFormat` |
| `PERF_ATTR_SIZE_VER0..VER9` | per-attr forward-compat sizes | `perf::AttrSize` |
| `struct perf_event_attr` | per-event configuration blob | `PerfEventAttr` |
| `struct perf_event_query_bpf` | per-`IOC_QUERY_BPF` | `PerfEventQueryBpf` |
| `PERF_EVENT_IOC_*` | per-fd ioctl commands | `perf::ioctl::*` |
| `enum perf_event_ioc_flags` | per-ioctl flag (FLAG_GROUP) | `perf::IocFlags` |
| `struct perf_event_mmap_page` | per-mmap header page | `PerfEventMmapPage` |
| `PERF_RECORD_MISC_*` | per-record misc field bits | `perf::RecMisc` |
| `struct perf_event_header` | per-record fixed prefix | `PerfEventHeader` |
| `struct perf_ns_link_info` | per-`PERF_RECORD_NAMESPACES` entry | `PerfNsLinkInfo` |
| `enum perf_event_type` | per-`PERF_RECORD_*` type | `perf::RecType` |
| `enum perf_record_ksymbol_type` | per-KSYMBOL kind | `perf::KsymbolType` |
| `PERF_RECORD_KSYMBOL_FLAGS_UNREGISTER` | per-ksym record flag | `perf::KsymbolFlags` |
| `enum perf_bpf_event_type` | per-BPF_EVENT kind | `perf::BpfEventType` |
| `enum perf_callchain_context` | per-callchain marker | `perf::CallchainCtx` |
| `PERF_AUX_FLAG_*` | per-AUX record flag bits | `perf::AuxFlags` |
| `PERF_FLAG_FD_*` | per-syscall `flags` arg | `perf::SyscallFlags` |
| `union perf_mem_data_src` + `PERF_MEM_*` | per-mem-sample data-source | `PerfMemDataSrc` |
| `struct perf_branch_entry` | per-LBR/BRBE entry | `PerfBranchEntry` |
| `union perf_sample_weight` | per-WEIGHT_STRUCT layout | `PerfSampleWeight` |
| `PERF_MAX_STACK_DEPTH / _CONTEXTS_PER_STACK` | per-callchain caps | `perf::limits` |
| `PERF_PMU_TYPE_SHIFT / _HW_EVENT_MASK` | per-HW/HW_CACHE PMU packing | `perf::pmu_pack` |
| `PERF_BRANCH_ENTRY_INFO_BITS_MAX` | per-branch-entry info-bit count | `perf::branch_info_bits` |

### abi surface (constants + structs)

### Major type (`attr.type`)

```
enum perf_type_id {
  PERF_TYPE_HARDWARE   = 0,   /* Generalized HW PMU event */
  PERF_TYPE_SOFTWARE   = 1,   /* Kernel-emitted SW event */
  PERF_TYPE_TRACEPOINT = 2,   /* Static tracepoint */
  PERF_TYPE_HW_CACHE   = 3,   /* Generalized HW cache event */
  PERF_TYPE_RAW        = 4,   /* PMU-specific raw event encoding */
  PERF_TYPE_BREAKPOINT = 5,   /* HW data/instr breakpoint */
  PERF_TYPE_MAX,              /* non-ABI */
};

/*
 * Dynamic PMU types are also valid for attr.type — values >= PERF_TYPE_MAX
 * select a kernel-registered PMU by id (e.g. uncore_*, intel_pt, msr).
 */
#define PERF_PMU_TYPE_SHIFT  32      /* attr.config packs PMU id in upper 32 bits */
#define PERF_HW_EVENT_MASK   0xffffffff   /* lower 32 bits = HW/HW_CACHE event id */
```

### `PERF_TYPE_HARDWARE` event ids (`attr.config & PERF_HW_EVENT_MASK`)

```
enum perf_hw_id {
  PERF_COUNT_HW_CPU_CYCLES              = 0,
  PERF_COUNT_HW_INSTRUCTIONS            = 1,
  PERF_COUNT_HW_CACHE_REFERENCES        = 2,
  PERF_COUNT_HW_CACHE_MISSES            = 3,
  PERF_COUNT_HW_BRANCH_INSTRUCTIONS     = 4,
  PERF_COUNT_HW_BRANCH_MISSES           = 5,
  PERF_COUNT_HW_BUS_CYCLES              = 6,
  PERF_COUNT_HW_STALLED_CYCLES_FRONTEND = 7,
  PERF_COUNT_HW_STALLED_CYCLES_BACKEND  = 8,
  PERF_COUNT_HW_REF_CPU_CYCLES          = 9,
  PERF_COUNT_HW_MAX,                       /* non-ABI */
};
```

### `PERF_TYPE_SOFTWARE` event ids

```
enum perf_sw_ids {
  PERF_COUNT_SW_CPU_CLOCK         = 0,
  PERF_COUNT_SW_TASK_CLOCK        = 1,
  PERF_COUNT_SW_PAGE_FAULTS       = 2,
  PERF_COUNT_SW_CONTEXT_SWITCHES  = 3,
  PERF_COUNT_SW_CPU_MIGRATIONS    = 4,
  PERF_COUNT_SW_PAGE_FAULTS_MIN   = 5,
  PERF_COUNT_SW_PAGE_FAULTS_MAJ   = 6,
  PERF_COUNT_SW_ALIGNMENT_FAULTS  = 7,
  PERF_COUNT_SW_EMULATION_FAULTS  = 8,
  PERF_COUNT_SW_DUMMY             = 9,   /* opens an attachment-only event */
  PERF_COUNT_SW_BPF_OUTPUT        = 10,  /* BPF helper bpf_perf_event_output target */
  PERF_COUNT_SW_CGROUP_SWITCHES   = 11,
  PERF_COUNT_SW_MAX,                     /* non-ABI */
};
```

### `PERF_TYPE_HW_CACHE` triple (`attr.config` lower 32 bits = BB | CC<<8 | DD<<16)

```
/* cache (BB) */
enum perf_hw_cache_id {
  PERF_COUNT_HW_CACHE_L1D  = 0,
  PERF_COUNT_HW_CACHE_L1I  = 1,
  PERF_COUNT_HW_CACHE_LL   = 2,
  PERF_COUNT_HW_CACHE_DTLB = 3,
  PERF_COUNT_HW_CACHE_ITLB = 4,
  PERF_COUNT_HW_CACHE_BPU  = 5,
  PERF_COUNT_HW_CACHE_NODE = 6,
  PERF_COUNT_HW_CACHE_MAX,            /* non-ABI */
};
/* op (CC) */
enum perf_hw_cache_op_id {
  PERF_COUNT_HW_CACHE_OP_READ     = 0,
  PERF_COUNT_HW_CACHE_OP_WRITE    = 1,
  PERF_COUNT_HW_CACHE_OP_PREFETCH = 2,
  PERF_COUNT_HW_CACHE_OP_MAX,         /* non-ABI */
};
/* result (DD) */
enum perf_hw_cache_op_result_id {
  PERF_COUNT_HW_CACHE_RESULT_ACCESS = 0,
  PERF_COUNT_HW_CACHE_RESULT_MISS   = 1,
  PERF_COUNT_HW_CACHE_RESULT_MAX,     /* non-ABI */
};
```

### `PERF_TYPE_RAW`

`attr.config` is the raw PMU-specific event encoding. `attr.config1` / `attr.config2` / `attr.config3` / `attr.config4` carry PMU extension fields (offcore-response masks, frontend masks, intel-pt config, etc.). No kernel translation is performed.

### `PERF_TYPE_BREAKPOINT`

`attr.bp_type` ⊆ `{HW_BREAKPOINT_R | HW_BREAKPOINT_W | HW_BREAKPOINT_X | HW_BREAKPOINT_EMPTY}` (from `<linux/hw_breakpoint.h>`); `attr.bp_addr` = virtual address to watch; `attr.bp_len` = `{1, 2, 4, 8}` bytes (HW_BREAKPOINT_LEN_*).

### `attr.sample_type` bits (`enum perf_event_sample_format`)

```
PERF_SAMPLE_IP              = 1U << 0,
PERF_SAMPLE_TID             = 1U << 1,
PERF_SAMPLE_TIME            = 1U << 2,
PERF_SAMPLE_ADDR            = 1U << 3,
PERF_SAMPLE_READ            = 1U << 4,
PERF_SAMPLE_CALLCHAIN       = 1U << 5,
PERF_SAMPLE_ID              = 1U << 6,
PERF_SAMPLE_CPU             = 1U << 7,
PERF_SAMPLE_PERIOD          = 1U << 8,
PERF_SAMPLE_STREAM_ID       = 1U << 9,
PERF_SAMPLE_RAW             = 1U << 10,
PERF_SAMPLE_BRANCH_STACK    = 1U << 11,
PERF_SAMPLE_REGS_USER       = 1U << 12,
PERF_SAMPLE_STACK_USER      = 1U << 13,
PERF_SAMPLE_WEIGHT          = 1U << 14,
PERF_SAMPLE_DATA_SRC        = 1U << 15,
PERF_SAMPLE_IDENTIFIER      = 1U << 16,
PERF_SAMPLE_TRANSACTION     = 1U << 17,
PERF_SAMPLE_REGS_INTR       = 1U << 18,
PERF_SAMPLE_PHYS_ADDR       = 1U << 19,
PERF_SAMPLE_AUX             = 1U << 20,
PERF_SAMPLE_CGROUP          = 1U << 21,
PERF_SAMPLE_DATA_PAGE_SIZE  = 1U << 22,
PERF_SAMPLE_CODE_PAGE_SIZE  = 1U << 23,
PERF_SAMPLE_WEIGHT_STRUCT   = 1U << 24,
PERF_SAMPLE_MAX             = 1U << 25,    /* non-ABI */
#define PERF_SAMPLE_WEIGHT_TYPE (PERF_SAMPLE_WEIGHT | PERF_SAMPLE_WEIGHT_STRUCT)
```

### `attr.branch_sample_type` bits (`enum perf_branch_sample_type`)

```
PERF_SAMPLE_BRANCH_USER        = 1U << 0,
PERF_SAMPLE_BRANCH_KERNEL      = 1U << 1,
PERF_SAMPLE_BRANCH_HV          = 1U << 2,
PERF_SAMPLE_BRANCH_ANY         = 1U << 3,
PERF_SAMPLE_BRANCH_ANY_CALL    = 1U << 4,
PERF_SAMPLE_BRANCH_ANY_RETURN  = 1U << 5,
PERF_SAMPLE_BRANCH_IND_CALL    = 1U << 6,
PERF_SAMPLE_BRANCH_ABORT_TX    = 1U << 7,
PERF_SAMPLE_BRANCH_IN_TX       = 1U << 8,
PERF_SAMPLE_BRANCH_NO_TX       = 1U << 9,
PERF_SAMPLE_BRANCH_COND        = 1U << 10,
PERF_SAMPLE_BRANCH_CALL_STACK  = 1U << 11,
PERF_SAMPLE_BRANCH_IND_JUMP    = 1U << 12,
PERF_SAMPLE_BRANCH_CALL        = 1U << 13,
PERF_SAMPLE_BRANCH_NO_FLAGS    = 1U << 14,
PERF_SAMPLE_BRANCH_NO_CYCLES   = 1U << 15,
PERF_SAMPLE_BRANCH_TYPE_SAVE   = 1U << 16,
PERF_SAMPLE_BRANCH_HW_INDEX    = 1U << 17,
PERF_SAMPLE_BRANCH_PRIV_SAVE   = 1U << 18,
PERF_SAMPLE_BRANCH_COUNTERS    = 1U << 19,
PERF_SAMPLE_BRANCH_MAX         = 1U << PERF_SAMPLE_BRANCH_MAX_SHIFT,
#define PERF_SAMPLE_BRANCH_PLM_ALL \
  (PERF_SAMPLE_BRANCH_USER | PERF_SAMPLE_BRANCH_KERNEL | PERF_SAMPLE_BRANCH_HV)
```

### Branch-entry classifier enums

```
/* PERF_BR_* — control-flow class for perf_branch_entry.type */
PERF_BR_UNKNOWN    = 0,  PERF_BR_COND     = 1,  PERF_BR_UNCOND   = 2,
PERF_BR_IND        = 3,  PERF_BR_CALL     = 4,  PERF_BR_IND_CALL = 5,
PERF_BR_RET        = 6,  PERF_BR_SYSCALL  = 7,  PERF_BR_SYSRET   = 8,
PERF_BR_COND_CALL  = 9,  PERF_BR_COND_RET = 10, PERF_BR_ERET     = 11,
PERF_BR_IRQ        = 12, PERF_BR_SERROR   = 13, PERF_BR_NO_TX    = 14,
PERF_BR_EXTEND_ABI = 15, PERF_BR_MAX

/* PERF_BR_SPEC_* — speculation outcome */
PERF_BR_SPEC_NA = 0, PERF_BR_SPEC_WRONG_PATH = 1,
PERF_BR_NON_SPEC_CORRECT_PATH = 2, PERF_BR_SPEC_CORRECT_PATH = 3, PERF_BR_SPEC_MAX

/* PERF_BR_NEW_* — secondary classifier (perf_branch_entry.new_type) */
PERF_BR_NEW_FAULT_ALGN = 0, PERF_BR_NEW_FAULT_DATA = 1, PERF_BR_NEW_FAULT_INST = 2,
PERF_BR_NEW_ARCH_1..5 = 3..7, PERF_BR_NEW_MAX
/* arm64 aliases */
PERF_BR_ARM64_FIQ          = PERF_BR_NEW_ARCH_1
PERF_BR_ARM64_DEBUG_HALT   = PERF_BR_NEW_ARCH_2
PERF_BR_ARM64_DEBUG_EXIT   = PERF_BR_NEW_ARCH_3
PERF_BR_ARM64_DEBUG_INST   = PERF_BR_NEW_ARCH_4
PERF_BR_ARM64_DEBUG_DATA   = PERF_BR_NEW_ARCH_5

/* PERF_BR_PRIV_* — privilege classifier (perf_branch_entry.priv) */
PERF_BR_PRIV_UNKNOWN=0, PERF_BR_PRIV_USER=1, PERF_BR_PRIV_KERNEL=2, PERF_BR_PRIV_HV=3
```

### Regs-ABI marker

```
enum perf_sample_regs_abi {
  PERF_SAMPLE_REGS_ABI_NONE = 0,
  PERF_SAMPLE_REGS_ABI_32   = 1,
  PERF_SAMPLE_REGS_ABI_64   = 2,
};
```

### Transaction qualifier (PERF_SAMPLE_TRANSACTION u64)

```
PERF_TXN_ELISION         = 1 << 0,
PERF_TXN_TRANSACTION     = 1 << 1,
PERF_TXN_SYNC            = 1 << 2,
PERF_TXN_ASYNC           = 1 << 3,
PERF_TXN_RETRY           = 1 << 4,
PERF_TXN_CONFLICT        = 1 << 5,
PERF_TXN_CAPACITY_WRITE  = 1 << 6,
PERF_TXN_CAPACITY_READ   = 1 << 7,
PERF_TXN_MAX             = 1 << 8,         /* non-ABI */
/* Bits 32..63 carry the architectural abort code */
PERF_TXN_ABORT_MASK      = 0xffffffffULL << 32
PERF_TXN_ABORT_SHIFT     = 32
```

### `attr.read_format` (returned by `read(fd)`)

```
enum perf_event_read_format {
  PERF_FORMAT_TOTAL_TIME_ENABLED = 1U << 0,
  PERF_FORMAT_TOTAL_TIME_RUNNING = 1U << 1,
  PERF_FORMAT_ID                 = 1U << 2,
  PERF_FORMAT_GROUP              = 1U << 3,
  PERF_FORMAT_LOST               = 1U << 4,
  PERF_FORMAT_MAX                = 1U << 5,   /* non-ABI */
};

/*
 * Non-GROUP layout:
 *   { u64 value;
 *     { u64 time_enabled } && TOTAL_TIME_ENABLED
 *     { u64 time_running } && TOTAL_TIME_RUNNING
 *     { u64 id           } && ID
 *     { u64 lost         } && LOST }
 *
 * GROUP layout:
 *   { u64 nr;
 *     { u64 time_enabled } && TOTAL_TIME_ENABLED
 *     { u64 time_running } && TOTAL_TIME_RUNNING
 *     { u64 value;
 *       { u64 id   } && ID
 *       { u64 lost } && LOST } [nr] }
 */
```

### `PERF_ATTR_SIZE_VER*` — forward/backward compatible sizes

```
PERF_ATTR_SIZE_VER0 =  64   /* first published */
PERF_ATTR_SIZE_VER1 =  72   /* + config2 */
PERF_ATTR_SIZE_VER2 =  80   /* + branch_sample_type */
PERF_ATTR_SIZE_VER3 =  96   /* + sample_regs_user + sample_stack_user */
PERF_ATTR_SIZE_VER4 = 104   /* + sample_regs_intr */
PERF_ATTR_SIZE_VER5 = 112   /* + aux_watermark */
PERF_ATTR_SIZE_VER6 = 120   /* + aux_sample_size */
PERF_ATTR_SIZE_VER7 = 128   /* + sig_data */
PERF_ATTR_SIZE_VER8 = 136   /* + config3 */
PERF_ATTR_SIZE_VER9 = 144   /* + config4 */
```

### `struct perf_event_attr` (full layout)

```
struct perf_event_attr {
  __u32   type;                        /* enum perf_type_id or dynamic PMU id */
  __u32   size;                        /* PERF_ATTR_SIZE_VER* — forward-compat */
  __u64   config;                      /* event id within type (lower 32 = id, upper 32 = PMU id) */
  union { __u64 sample_period; __u64 sample_freq; };   /* selector via attr.freq */
  __u64   sample_type;                 /* PERF_SAMPLE_* mask */
  __u64   read_format;                 /* PERF_FORMAT_* mask */

  /* Bitfield flags (low 40 bits used; upper bits reserved) */
  __u64   disabled                : 1, /* event disabled at open; needs IOC_ENABLE */
          inherit                 : 1, /* clones / forks inherit event */
          pinned                  : 1, /* must always be scheduled on PMU */
          exclusive               : 1, /* sole group on PMU */
          exclude_user            : 1, /* skip user-mode samples */
          exclude_kernel          : 1, /* skip kernel-mode samples */
          exclude_hv              : 1, /* skip hypervisor */
          exclude_idle            : 1, /* skip idle thread */
          mmap                    : 1, /* emit PERF_RECORD_MMAP */
          comm                    : 1, /* emit PERF_RECORD_COMM */
          freq                    : 1, /* sample_freq is freq, not period */
          inherit_stat            : 1, /* per-task counts on inherit */
          enable_on_exec          : 1, /* next execve enables */
          task                    : 1, /* emit fork/exit records */
          watermark               : 1, /* wakeup_watermark vs wakeup_events */
          precise_ip              : 2, /* skid constraint (0=arbitrary, 3=zero) */
          mmap_data               : 1, /* emit non-exec mmap */
          sample_id_all           : 1, /* every record carries sample_id tail */
          exclude_host            : 1, /* exclude host (KVM) */
          exclude_guest           : 1, /* exclude guest */
          exclude_callchain_kernel: 1,
          exclude_callchain_user  : 1,
          mmap2                   : 1, /* emit PERF_RECORD_MMAP2 (with maj/min/ino) */
          comm_exec               : 1, /* mark COMM records caused by exec */
          use_clockid             : 1, /* use attr.clockid for time fields */
          context_switch          : 1, /* emit PERF_RECORD_SWITCH* */
          write_backward          : 1, /* ring buffer written tail-to-head */
          namespaces              : 1, /* emit PERF_RECORD_NAMESPACES */
          ksymbol                 : 1, /* emit PERF_RECORD_KSYMBOL */
          bpf_event               : 1, /* emit PERF_RECORD_BPF_EVENT */
          aux_output              : 1, /* event produces AUX records, not regular samples */
          cgroup                  : 1, /* emit PERF_RECORD_CGROUP */
          text_poke               : 1, /* emit PERF_RECORD_TEXT_POKE */
          build_id                : 1, /* MMAP2 carries build-id */
          inherit_thread          : 1, /* inherit only on CLONE_THREAD */
          remove_on_exec          : 1, /* drop event on execve */
          sigtrap                 : 1, /* send synchronous SIGTRAP on overflow */
          defer_callchain         : 1, /* defer user callchain via CALLCHAIN_DEFERRED */
          defer_output            : 1, /* output the deferred record */
          __reserved_1            : 24;

  union { __u32 wakeup_events; __u32 wakeup_watermark; };   /* via attr.watermark */
  __u32   bp_type;                     /* BREAKPOINT: HW_BREAKPOINT_R|W|X */
  union { __u64 bp_addr; __u64 kprobe_func; __u64 uprobe_path; __u64 config1; };
  union { __u64 bp_len;  __u64 kprobe_addr;  __u64 probe_offset; __u64 config2; };
  __u64   branch_sample_type;          /* enum perf_branch_sample_type */
  __u64   sample_regs_user;            /* regs mask (see asm/perf_regs.h) */
  __u32   sample_stack_user;           /* user-stack bytes to capture */
  __s32   clockid;                     /* CLOCK_* when use_clockid */
  __u64   sample_regs_intr;            /* regs mask for interrupt sample */
  __u32   aux_watermark;               /* AUX wakeup watermark in bytes */
  __u16   sample_max_stack;            /* < kernel.perf_event_max_stack */
  __u16   __reserved_2;
  __u32   aux_sample_size;             /* bytes of AUX to copy per sample */
  union {
    __u32 aux_action;
    struct {
      __u32 aux_start_paused : 1,      /* start AUX tracing paused */
            aux_pause        : 1,      /* on overflow, pause AUX */
            aux_resume       : 1,      /* on overflow, resume AUX */
            __reserved_3     : 29;
    };
  };
  __u64   sig_data;                    /* siginfo_t::si_perf_data for sigtrap */
  __u64   config3;                     /* PMU-extension config word #3 */
  __u64   config4;                     /* PMU-extension config word #4 */
};
```

### `PERF_EVENT_IOC_*` ioctls (magic `'$'` = 0x24)

```
PERF_EVENT_IOC_ENABLE             = _IO  ('$', 0)
PERF_EVENT_IOC_DISABLE            = _IO  ('$', 1)
PERF_EVENT_IOC_REFRESH            = _IO  ('$', 2)              /* arg = nr-overflows */
PERF_EVENT_IOC_RESET              = _IO  ('$', 3)
PERF_EVENT_IOC_PERIOD             = _IOW ('$', 4, __u64)
PERF_EVENT_IOC_SET_OUTPUT         = _IO  ('$', 5)              /* arg = sibling fd or -1 */
PERF_EVENT_IOC_SET_FILTER         = _IOW ('$', 6, char *)
PERF_EVENT_IOC_ID                 = _IOR ('$', 7, __u64 *)
PERF_EVENT_IOC_SET_BPF            = _IOW ('$', 8, __u32)
PERF_EVENT_IOC_PAUSE_OUTPUT       = _IOW ('$', 9, __u32)
PERF_EVENT_IOC_QUERY_BPF          = _IOWR('$', 10, struct perf_event_query_bpf *)
PERF_EVENT_IOC_MODIFY_ATTRIBUTES  = _IOW ('$', 11, struct perf_event_attr *)

enum perf_event_ioc_flags { PERF_IOC_FLAG_GROUP = 1U << 0 };

struct perf_event_query_bpf {
  __u32 ids_len;
  __u32 prog_cnt;             /* OUT: kernel-set total available */
  __u32 ids[];                /* OUT: BPF prog ids attached */
};
```

### `syscall(SYS_perf_event_open, attr, pid, cpu, group_fd, flags)`

```
PERF_FLAG_FD_NO_GROUP   = 1UL << 0   /* do not include in group_fd's group */
PERF_FLAG_FD_OUTPUT     = 1UL << 1   /* redirect mmap output to group_fd */
PERF_FLAG_PID_CGROUP    = 1UL << 2   /* pid arg = cgroup fd (per-CPU mode only) */
PERF_FLAG_FD_CLOEXEC    = 1UL << 3   /* O_CLOEXEC on returned fd */
```

### `struct perf_event_mmap_page` (mmap header page at offset 0)

```
struct perf_event_mmap_page {
  __u32 version;            /* layout version */
  __u32 compat_version;     /* min version this is compat with */

  /* Self-monitor seqlock + rdpmc bits (see header self-monitor loop). */
  __u32 lock;               /* seqlock; reader retries while odd / changed */
  __u32 index;              /* hardware counter index (0 = no rdpmc) */
  __s64 offset;             /* count = pmc + offset */
  __u64 time_enabled;       /* ns the event was enabled */
  __u64 time_running;       /* ns the event ran on PMU */
  union {
    __u64 capabilities;
    struct {
      __u64 cap_bit0              : 1,   /* always 0 (deprecated) */
            cap_bit0_is_deprecated: 1,   /* always 1 (sentinel) */
            cap_user_rdpmc        : 1,   /* userspace rdpmc valid */
            cap_user_time         : 1,   /* time_{shift,mult,offset} valid */
            cap_user_time_zero    : 1,   /* time_zero valid */
            cap_user_time_short   : 1,   /* time_cycles/time_mask valid */
            cap_____res           : 58;
    };
  };
  __u16 pmc_width;          /* bit-width for sign-extending rdpmc result */
  __u16 time_shift;         /* TSC -> ns conversion */
  __u32 time_mult;
  __u64 time_offset;
  __u64 time_zero;          /* hardware-clock anchor */
  __u32 size;               /* size of header up to __reserved[] */
  __u32 __reserved_1;
  __u64 time_cycles;        /* cap_user_time_short anchor */
  __u64 time_mask;
  __u8  __reserved[116*8];  /* pad to 1024 bytes */

  /* Ring-buffer control region (writer = kernel, reader = userspace). */
  __u64 data_head;          /* updated by kernel; reader smp_rmb after read */
  __u64 data_tail;          /* updated by reader (PROT_WRITE only) */
  __u64 data_offset;        /* offset of data buffer within mmap */
  __u64 data_size;          /* size of data buffer */

  /* AUX ring region (Intel PT / CoreSight / SPE / IBS). */
  __u64 aux_head;
  __u64 aux_tail;
  __u64 aux_offset;         /* >= data_offset + data_size */
  __u64 aux_size;           /* size of AUX area; userspace mmaps separately */
};
```

### `struct perf_event_header` (every record's prefix)

```
struct perf_event_header {
  __u32 type;     /* enum perf_event_type */
  __u16 misc;     /* PERF_RECORD_MISC_* bits */
  __u16 size;     /* total record size incl. this header */
};
```

### `PERF_RECORD_MISC_*` bits in `perf_event_header.misc`

```
PERF_RECORD_MISC_CPUMODE_MASK         = (7 << 0)
PERF_RECORD_MISC_CPUMODE_UNKNOWN      = (0 << 0)
PERF_RECORD_MISC_KERNEL               = (1 << 0)
PERF_RECORD_MISC_USER                 = (2 << 0)
PERF_RECORD_MISC_HYPERVISOR           = (3 << 0)
PERF_RECORD_MISC_GUEST_KERNEL         = (4 << 0)
PERF_RECORD_MISC_GUEST_USER           = (5 << 0)
PERF_RECORD_MISC_PROC_MAP_PARSE_TIMEOUT = (1 << 12)
PERF_RECORD_MISC_MMAP_DATA            = (1 << 13)   /* MMAP* */
PERF_RECORD_MISC_COMM_EXEC            = (1 << 13)   /* COMM */
PERF_RECORD_MISC_FORK_EXEC            = (1 << 13)   /* FORK */
PERF_RECORD_MISC_SWITCH_OUT           = (1 << 13)   /* SWITCH* */
PERF_RECORD_MISC_EXACT_IP             = (1 << 14)   /* SAMPLE */
PERF_RECORD_MISC_SWITCH_OUT_PREEMPT   = (1 << 14)   /* SWITCH* */
PERF_RECORD_MISC_MMAP_BUILD_ID        = (1 << 14)   /* MMAP2 */
PERF_RECORD_MISC_EXT_RESERVED         = (1 << 15)
```

### `enum perf_event_type` (`PERF_RECORD_*`)

```
PERF_RECORD_MMAP                = 1
PERF_RECORD_LOST                = 2
PERF_RECORD_COMM                = 3
PERF_RECORD_EXIT                = 4
PERF_RECORD_THROTTLE            = 5
PERF_RECORD_UNTHROTTLE          = 6
PERF_RECORD_FORK                = 7
PERF_RECORD_READ                = 8
PERF_RECORD_SAMPLE              = 9
PERF_RECORD_MMAP2               = 10
PERF_RECORD_AUX                 = 11
PERF_RECORD_ITRACE_START        = 12
PERF_RECORD_LOST_SAMPLES        = 13
PERF_RECORD_SWITCH              = 14
PERF_RECORD_SWITCH_CPU_WIDE     = 15
PERF_RECORD_NAMESPACES          = 16
PERF_RECORD_KSYMBOL             = 17
PERF_RECORD_BPF_EVENT           = 18
PERF_RECORD_CGROUP              = 19
PERF_RECORD_TEXT_POKE           = 20
PERF_RECORD_AUX_OUTPUT_HW_ID    = 21
PERF_RECORD_CALLCHAIN_DEFERRED  = 22
PERF_RECORD_MAX                          /* non-ABI */
```

Auxiliary record-shape enums:

```
enum perf_record_ksymbol_type {
  PERF_RECORD_KSYMBOL_TYPE_UNKNOWN = 0,
  PERF_RECORD_KSYMBOL_TYPE_BPF     = 1,
  PERF_RECORD_KSYMBOL_TYPE_OOL     = 2,
  PERF_RECORD_KSYMBOL_TYPE_MAX,
};
#define PERF_RECORD_KSYMBOL_FLAGS_UNREGISTER (1 << 0)

enum perf_bpf_event_type {
  PERF_BPF_EVENT_UNKNOWN     = 0,
  PERF_BPF_EVENT_PROG_LOAD   = 1,
  PERF_BPF_EVENT_PROG_UNLOAD = 2,
  PERF_BPF_EVENT_MAX,
};

struct perf_ns_link_info { __u64 dev; __u64 ino; };

enum { NET_NS_INDEX = 0, UTS_NS_INDEX, IPC_NS_INDEX, PID_NS_INDEX,
       USER_NS_INDEX, MNT_NS_INDEX, CGROUP_NS_INDEX, NR_NAMESPACES };
```

### Callchain context markers (special pseudo-IPs in callchain ips[])

```
PERF_CONTEXT_HV             = (__u64)-32
PERF_CONTEXT_KERNEL         = (__u64)-128
PERF_CONTEXT_USER           = (__u64)-512
PERF_CONTEXT_USER_DEFERRED  = (__u64)-640
PERF_CONTEXT_GUEST          = (__u64)-2048
PERF_CONTEXT_GUEST_KERNEL   = (__u64)-2176
PERF_CONTEXT_GUEST_USER     = (__u64)-2560
PERF_CONTEXT_MAX            = (__u64)-4095
#define PERF_MAX_STACK_DEPTH          127
#define PERF_MAX_CONTEXTS_PER_STACK     8
```

### AUX-record flags (`PERF_RECORD_AUX.flags`)

```
PERF_AUX_FLAG_TRUNCATED               = 0x0001
PERF_AUX_FLAG_OVERWRITE               = 0x0002
PERF_AUX_FLAG_PARTIAL                 = 0x0004
PERF_AUX_FLAG_COLLISION               = 0x0008
PERF_AUX_FLAG_PMU_FORMAT_TYPE_MASK    = 0xff00
PERF_AUX_FLAG_CORESIGHT_FORMAT_CORESIGHT = 0x0000
PERF_AUX_FLAG_CORESIGHT_FORMAT_RAW       = 0x0100
```

### Memory data-source (`PERF_SAMPLE_DATA_SRC`)

```
union perf_mem_data_src {
  __u64 val;
  struct {
    __u64 mem_op      :  5,   /* PERF_MEM_OP_*  shift = 0  */
          mem_lvl     : 14,   /* PERF_MEM_LVL_* shift = 5  */
          mem_snoop   :  5,   /* PERF_MEM_SNOOP_* shift = 19 */
          mem_lock    :  2,   /* PERF_MEM_LOCK_*  shift = 24 */
          mem_dtlb    :  7,   /* PERF_MEM_TLB_*   shift = 26 */
          mem_lvl_num :  4,   /* PERF_MEM_LVLNUM_* shift = 33 */
          mem_remote  :  1,   /* PERF_MEM_REMOTE_* shift = 37 */
          mem_snoopx  :  2,   /* PERF_MEM_SNOOPX_* shift = 38 */
          mem_blk     :  3,   /* PERF_MEM_BLK_*    shift = 40 */
          mem_hops    :  3,   /* PERF_MEM_HOPS_*   shift = 43 */
          mem_region  :  5,   /* PERF_MEM_REGION_* shift = 46 */
          mem_rsvd    : 13;
  };
};
#define PERF_MEM_S(a, s) (((__u64)PERF_MEM_##a##_##s) << PERF_MEM_##a##_SHIFT)
```

Constants: `PERF_MEM_OP_{NA,LOAD,STORE,PFETCH,EXEC}`; legacy `PERF_MEM_LVL_{NA,HIT,MISS,L1,LFB,L2,L3,LOC_RAM,REM_RAM1,REM_RAM2,REM_CCE1,REM_CCE2,IO,UNC}`; newer `PERF_MEM_LVLNUM_{L0,L1,L2,L2_MHB,L3,L4,MSC,UNC,CXL,IO,ANY_CACHE,LFB,RAM,PMEM,NA}`; `PERF_MEM_SNOOP_{NA,NONE,HIT,MISS,HITM}`; `PERF_MEM_SNOOPX_{FWD,PEER}`; `PERF_MEM_LOCK_{NA,LOCKED}`; `PERF_MEM_TLB_{NA,HIT,MISS,L1,L2,WK,OS}`; `PERF_MEM_BLK_{NA,DATA,ADDR}`; `PERF_MEM_HOPS_0..3`; `PERF_MEM_REGION_{NA,RSVD,L_SHARE,L_NON_SHARE,O_IO,O_SHARE,O_NON_SHARE,MMIO,MEM0..7}`.

### Branch entry (`PERF_SAMPLE_BRANCH_STACK`)

```
struct perf_branch_entry {
  __u64 from;
  __u64 to;
  __u64 mispred   : 1,
        predicted : 1,
        in_tx     : 1,
        abort     : 1,
        cycles    : 16,
        type      : 4,   /* PERF_BR_* */
        spec      : 2,   /* PERF_BR_SPEC_* */
        new_type  : 4,   /* PERF_BR_NEW_* */
        priv      : 3,   /* PERF_BR_PRIV_* */
        reserved  : 31;
};
#define PERF_BRANCH_ENTRY_INFO_BITS_MAX 33
```

### Sample weight (`PERF_SAMPLE_WEIGHT_STRUCT`)

```
union perf_sample_weight {
  __u64 full;                             /* PERF_SAMPLE_WEIGHT */
  struct {                                /* PERF_SAMPLE_WEIGHT_STRUCT (LE) */
    __u32 var1_dw;
    __u16 var2_w;
    __u16 var3_w;
  };
};
```

### VFS-cap link-back

`perf_event.h` includes `<linux/ioctl.h>` for `_IO*` macros; the SAMPLE_RAW payload is opaque per upstream comment ("`PERF_SAMPLE_RAW` contents are not an ABI").

### compatibility contract

REQ-1 `perf_event_open(attr, pid, cpu, group_fd, flags)`:
- `attr` is `struct perf_event_attr *`; `attr->size` must equal one of `PERF_ATTR_SIZE_VER0..VER9` or the kernel's larger current size; mismatch ⇒ -EINVAL or -E2BIG; the kernel zero-extends/truncates as needed for forward-compat.
- `pid`: -1 = all tasks on cpu; 0 = current task; >0 = specific tid; with `PERF_FLAG_PID_CGROUP` ⇒ pid is a cgroup-fd.
- `cpu`: -1 = follow task to any cpu; >=0 = pin to that cpu; `(pid == -1 && cpu == -1)` ⇒ -EINVAL.
- `group_fd`: -1 = new group leader; else fd of an existing event ⇒ join that group.
- `flags` ⊆ `{FD_NO_GROUP | FD_OUTPUT | PID_CGROUP | FD_CLOEXEC}`.

REQ-2 `attr.type`:
- `PERF_TYPE_HARDWARE`: `config` lower 32 bits = `PERF_COUNT_HW_*` event id; upper 32 bits = PMU id (0 = default core PMU).
- `PERF_TYPE_SOFTWARE`: `config` ∈ `PERF_COUNT_SW_*`.
- `PERF_TYPE_TRACEPOINT`: `config` = tracepoint id from `debugfs:tracing/events/*/id`; kernel must have CONFIG_TRACEPOINTS.
- `PERF_TYPE_HW_CACHE`: `config` lower 32 = `(cache_id) | (op_id << 8) | (result_id << 16)`; upper 32 = PMU id.
- `PERF_TYPE_RAW`: `config` is the raw event encoding for the selected PMU; `config1/2/3/4` provide extension words; no kernel translation.
- `PERF_TYPE_BREAKPOINT`: `bp_type`/`bp_addr`/`bp_len` define the HW breakpoint; `config` is ignored.
- `attr.type >= PERF_TYPE_MAX`: kernel checks dynamic PMU registry; ENOENT if unknown.

REQ-3 `attr.size` forward/backward compatibility:
- New userspace + old kernel: kernel rejects size > known ⇒ -E2BIG.
- Old userspace + new kernel: kernel zero-extends to current size; bits in later fields ⇒ 0.
- `attr.size == 0` ⇒ -EINVAL.

REQ-4 Sampling vs counting:
- `attr.freq == 0` + `attr.sample_period > 0`: overflow every period count.
- `attr.freq == 1` + `attr.sample_freq > 0`: kernel auto-adjusts period to target sample_freq events/sec; bounded by `kernel.perf_event_max_sample_rate` sysctl.
- `sample_period == 0` ⇒ counting only (no overflow records).

REQ-5 `attr.sample_type`:
- Subset of `PERF_SAMPLE_*` mask.
- `PERF_SAMPLE_IDENTIFIER` is included in every record's tail when `sample_id_all` is set (regardless of record type).
- `PERF_SAMPLE_RAW` payload is opaque per upstream comment.
- `PERF_SAMPLE_AUX` + `aux_sample_size > 0` ⇒ snapshot AUX into the sample.
- `PERF_SAMPLE_BRANCH_STACK` requires `attr.branch_sample_type` non-zero.

REQ-6 `attr.read_format`:
- `PERF_FORMAT_GROUP` requires `read()` to receive the group leader's fd; non-leader read with GROUP ⇒ -EINVAL.
- `PERF_FORMAT_ID` adds per-event id (matches `sample.id`).
- `PERF_FORMAT_LOST` adds lost-sample count per event.
- `PERF_FORMAT_TOTAL_TIME_ENABLED` / `_RUNNING` enable scaling (`scale = enabled / running`).

REQ-7 Bitfield flags (selected highlights):
- `disabled`: event starts disabled; `IOC_ENABLE` activates.
- `inherit`: forked children inherit the event; mutually exclusive with `PERF_FORMAT_GROUP` reads of per-task aggregation.
- `pinned`: kernel guarantees this event always counts; preempts non-pinned on contention; fail-open ⇒ event in error state.
- `exclusive`: event monopolises the PMU (no other groups co-scheduled).
- `exclude_user/kernel/hv/idle/host/guest`: filter samples by ring/context.
- `precise_ip`: 0 = arbitrary skid, 1 = constant skid, 2 = zero-skid requested, 3 = zero-skid required (Intel PEBS / SPE / IBS).
- `mmap` / `mmap2` / `mmap_data` / `build_id`: emit `PERF_RECORD_MMAP{,2}` for code mappings; `mmap_data` adds non-exec mappings; `build_id` carries build-id in mmap2.
- `comm` / `comm_exec`: emit `PERF_RECORD_COMM`; exec-caused has `MISC_COMM_EXEC` bit.
- `task`: emit `PERF_RECORD_FORK` / `_EXIT`.
- `context_switch`: emit `PERF_RECORD_SWITCH{_CPU_WIDE}`.
- `write_backward`: writer wraps tail-to-head (overwrite mode); reader reads stale data first.
- `namespaces` / `ksymbol` / `bpf_event` / `cgroup` / `text_poke`: emit the corresponding records.
- `aux_output`: this event redirects its samples into the linked AUX area instead of the data ring (Intel PT PEBS-via-PT).
- `inherit_thread`: inherit only for `CLONE_THREAD` (thread-group siblings), not for full fork.
- `remove_on_exec`: drop on execve (security-isolation).
- `sigtrap`: synchronous SIGTRAP delivered on overflow; `sig_data` ⇒ `siginfo_t::si_perf_data`.
- `defer_callchain` + `defer_output`: split user callchain capture out into `PERF_RECORD_CALLCHAIN_DEFERRED`.

REQ-8 `attr.wakeup_events` / `wakeup_watermark`:
- `watermark == 0` ⇒ `wakeup_events`: wake on every N samples.
- `watermark == 1` ⇒ `wakeup_watermark`: wake when ring-buffer has ≥ N bytes new data.

REQ-9 `attr.bp_*` (PERF_TYPE_BREAKPOINT):
- `bp_type` ⊆ `{HW_BREAKPOINT_R | HW_BREAKPOINT_W | HW_BREAKPOINT_X | HW_BREAKPOINT_EMPTY}`; combination per arch (x86: only one of R/W/RW, X separate).
- `bp_len` ∈ `{1, 2, 4, 8}`; arch-specific subset.
- `bp_addr` = virtual address in target task.

REQ-10 `attr.config1` / `config2` / `config3` / `config4`:
- For RAW PMU events: extension words.
- For tracepoints with filters: `IOC_SET_FILTER` sets the filter string instead.
- For PERF_TYPE_BREAKPOINT: union overlay supplies `bp_addr`/`bp_len`.
- For kprobe/uprobe via `kprobe_func`/`kprobe_addr`/`uprobe_path`/`probe_offset`: dynamic probe creation.

REQ-11 `attr.branch_sample_type`:
- Required when `PERF_SAMPLE_BRANCH_STACK` is set.
- Must be a subset of `PERF_SAMPLE_BRANCH_*`.
- Privilege bits (`USER|KERNEL|HV`) — if absent, kernel inherits from event priv level.

REQ-12 `attr.sample_regs_user` / `sample_regs_intr` / `sample_stack_user`:
- `sample_regs_*` = bitmap of regs to dump (see `asm/perf_regs.h`).
- `sample_stack_user` = user-stack bytes to snapshot at sample time; the record carries `size`, `data[size]`, `dyn_size`.

REQ-13 `attr.use_clockid` + `attr.clockid`:
- `clockid` ∈ `{CLOCK_MONOTONIC, CLOCK_MONOTONIC_RAW, CLOCK_REALTIME, CLOCK_BOOTTIME, CLOCK_TAI}`; other ids ⇒ -EINVAL.
- When set, `PERF_SAMPLE_TIME` carries `clockid` time, not the default sched_clock.

REQ-14 `attr.aux_*`:
- `aux_watermark`: AUX-area wakeup threshold in bytes.
- `aux_sample_size`: bytes of AUX to snapshot per regular sample (PERF_SAMPLE_AUX).
- `aux_action`: `aux_start_paused | aux_pause | aux_resume` (mutually exclusive single-bits).

REQ-15 `attr.sig_data` + `attr.sigtrap`:
- `sigtrap` ⇒ kernel delivers synchronous SIGTRAP on overflow; `sig_data` is reflected in `siginfo_t::si_perf_data` (long-sized on 64-bit, truncated on 32-bit).

REQ-16 `PERF_EVENT_IOC_ENABLE` (`_IO('$', 0)`):
- Argument: `PERF_IOC_FLAG_GROUP` (1) to enable the whole group, else just this event.
- Enables counting / sampling; returns 0 on success.

REQ-17 `PERF_EVENT_IOC_DISABLE` (`_IO('$', 1)`):
- Symmetric to ENABLE; `PERF_IOC_FLAG_GROUP` enables group-wide disable.

REQ-18 `PERF_EVENT_IOC_REFRESH` (`_IO('$', 2)`):
- Argument: refresh count (number of overflows after which to auto-disable).
- Used to limit sampling bursts.

REQ-19 `PERF_EVENT_IOC_RESET` (`_IO('$', 3)`):
- Resets the counter value to zero; with `PERF_IOC_FLAG_GROUP`, resets all events in group.

REQ-20 `PERF_EVENT_IOC_PERIOD` (`_IOW('$', 4, __u64)`):
- Argument: new `sample_period` (or sample_freq if attr.freq).
- Effect applied at next sample boundary.

REQ-21 `PERF_EVENT_IOC_SET_OUTPUT` (`_IO('$', 5)`):
- Argument: fd of another event whose ring buffer to share; -1 to detach.
- Records emitted by this event are funneled into the target's mmap'd ring.

REQ-22 `PERF_EVENT_IOC_SET_FILTER` (`_IOW('$', 6, char *)`):
- Argument: NUL-terminated filter expression string (tracepoint filter syntax, or kprobe/uprobe filter, or address filter for HW-trace events).
- Replaces any existing filter; empty string clears it.

REQ-23 `PERF_EVENT_IOC_ID` (`_IOR('$', 7, __u64 *)`):
- Argument: `__u64 *` (out).
- Kernel writes the event's unique id (matches `sample.id` when `PERF_SAMPLE_ID` set).

REQ-24 `PERF_EVENT_IOC_SET_BPF` (`_IOW('$', 8, __u32)`):
- Argument: BPF prog fd (passed as a u32 via the ioctl arg).
- Attaches a BPF program of type `BPF_PROG_TYPE_PERF_EVENT` (or `_KPROBE` / `_TRACEPOINT` for the matching event class).
- Multiple BPF progs may attach if tracepoint and the program type supports multi-attach.

REQ-25 `PERF_EVENT_IOC_PAUSE_OUTPUT` (`_IOW('$', 9, __u32)`):
- Argument: 1 = pause ring-buffer writes; 0 = resume.
- Useful for atomic snapshotting.

REQ-26 `PERF_EVENT_IOC_QUERY_BPF` (`_IOWR('$', 10, struct perf_event_query_bpf *)`):
- Argument: `struct perf_event_query_bpf { __u32 ids_len; __u32 prog_cnt; __u32 ids[]; }`.
- Kernel fills `prog_cnt` = total attached, copies up to `ids_len` ids into `ids[]`; returns -ENOSPC if `ids_len < prog_cnt`.

REQ-27 `PERF_EVENT_IOC_MODIFY_ATTRIBUTES` (`_IOW('$', 11, struct perf_event_attr *)`):
- Argument: replacement attr.
- Subset of fields may be modified live (sampling period, filter mask); others ⇒ -EINVAL.

REQ-28 `mmap(fd, sz, PROT_READ|PROT_WRITE, MAP_SHARED, 0)`:
- Layout: page 0 = `struct perf_event_mmap_page`; pages 1..N-1 = data ring; bytes `aux_offset..aux_offset+aux_size` (separate mmap at `aux_offset`) = AUX area.
- `(N - 1) * page_size` must be a power of two.
- `data_head` written by kernel; reader does `smp_rmb()` after reading head; `data_tail` written by reader after `smp_mb()`.
- `write_backward = 1` ⇒ overwrite mode (data_head wraps over unread data); reader sees newest-first.

REQ-29 `struct perf_event_mmap_page` self-monitor loop:
- Reader reads `lock` (seqlock), barrier, reads counter / time fields, barrier, re-reads `lock`; retries if `lock` changed or odd.
- `cap_user_rdpmc == 1` + `index > 0` ⇒ userspace may issue `rdpmc(index - 1)`; sign-extend using `pmc_width`.
- `cap_user_time == 1` ⇒ `(quot << time_shift) * time_mult + (rem * time_mult >> time_shift)` yields ns since `time_zero`.
- `cap_user_time_short` ⇒ extend 64-bit cycle from short hardware counter using `time_cycles` / `time_mask`.

REQ-30 `PERF_RECORD_MMAP`:
- Carries `{pid, tid, addr, len, pgoff, filename[], sample_id}`.
- Emitted for `PROT_EXEC` mappings (and non-exec if `mmap_data`).

REQ-31 `PERF_RECORD_LOST`:
- Carries `{id, lost, sample_id}` — number of records lost on this event since the last LOST record.

REQ-32 `PERF_RECORD_COMM`:
- Carries `{pid, tid, comm[], sample_id}`; `MISC_COMM_EXEC` bit set if caused by execve.

REQ-33 `PERF_RECORD_EXIT` / `_FORK`:
- Carry `{pid, ppid, tid, ptid, time, sample_id}`.

REQ-34 `PERF_RECORD_THROTTLE` / `_UNTHROTTLE`:
- Carry `{time, id, stream_id, sample_id}` — kernel auto-throttled because sample rate exceeded `kernel.perf_event_max_sample_rate`.

REQ-35 `PERF_RECORD_READ`:
- Carries `{pid, tid, read_format-values, sample_id}` — fired by group leader on context switch when `inherit` + `enable_on_exec`.

REQ-36 `PERF_RECORD_SAMPLE`:
- Variable layout per `attr.sample_type`; fields ordered per header documentation: `{id, ip, pid/tid, time, addr, id, stream_id, cpu/res, period, read_format-values, callchain, raw, branch_stack, regs_user, stack_user, weight, data_src, transaction, regs_intr, phys_addr, cgroup, data_page_size, code_page_size, aux}`.

REQ-37 `PERF_RECORD_MMAP2`:
- Like MMAP plus `{maj, min, ino, ino_generation}` (or `{build_id_size, build_id[20]}` union when `attr.build_id`), plus `prot, flags, filename[]`.

REQ-38 `PERF_RECORD_AUX`:
- Carries `{aux_offset, aux_size, flags, sample_id}`; `flags` ⊆ `PERF_AUX_FLAG_*`.

REQ-39 `PERF_RECORD_ITRACE_START`:
- Marks start of an instruction-trace session; `{pid, tid, sample_id}`.

REQ-40 `PERF_RECORD_LOST_SAMPLES`:
- `{lost, sample_id}` — overflowed samples lost vs the LOST record (which reports per-event lost records).

REQ-41 `PERF_RECORD_SWITCH` / `_SWITCH_CPU_WIDE`:
- Context-switch markers; `_CPU_WIDE` variant carries `{next_prev_pid, next_prev_tid}`.

REQ-42 `PERF_RECORD_NAMESPACES`:
- `{pid, tid, nr_namespaces, [{dev, inode}] [nr_namespaces], sample_id}`; entries indexed by `NET_NS_INDEX`..`CGROUP_NS_INDEX`.

REQ-43 `PERF_RECORD_KSYMBOL`:
- `{addr, len, ksym_type, flags, name[], sample_id}`; `ksym_type` ∈ `enum perf_record_ksymbol_type`; `flags & PERF_RECORD_KSYMBOL_FLAGS_UNREGISTER` for removal.

REQ-44 `PERF_RECORD_BPF_EVENT`:
- `{type, flags, id, tag[BPF_TAG_SIZE], sample_id}`; `type` ∈ `enum perf_bpf_event_type`.

REQ-45 `PERF_RECORD_CGROUP`:
- `{id, path[], sample_id}` — cgroup-id ↔ path binding announcement.

REQ-46 `PERF_RECORD_TEXT_POKE`:
- `{addr, old_len, new_len, bytes[old_len + new_len], sample_id}` — self-modified-code (jump labels, alternatives, ftrace trampolines).

REQ-47 `PERF_RECORD_AUX_OUTPUT_HW_ID`:
- `{hw_id, sample_id}` — per-PMU hardware-id mapping for disambiguating AUX records.

REQ-48 `PERF_RECORD_CALLCHAIN_DEFERRED`:
- `{cookie, nr, ips[nr], sample_id}` — user callchain captured right before user-space return, stitched with earlier kernel-only callchains.

REQ-49 Callchain context markers in `ips[]`:
- Negative pseudo-IPs in the callchain stream tag the following frames' privilege/guest level: `PERF_CONTEXT_KERNEL`, `_USER`, `_HV`, `_GUEST_*`, `_USER_DEFERRED`; `PERF_MAX_STACK_DEPTH = 127`, `PERF_MAX_CONTEXTS_PER_STACK = 8`.

REQ-50 `struct perf_branch_entry` (BRBE / LBR / BHRB):
- 8 bytes `from`, 8 bytes `to`, 8 bytes of bitfields; total 24 bytes.
- `type` ∈ `PERF_BR_*`; `spec` ∈ `PERF_BR_SPEC_*`; `new_type` ∈ `PERF_BR_NEW_*`; `priv` ∈ `PERF_BR_PRIV_*`.

REQ-51 `union perf_sample_weight`:
- `WEIGHT` ⇒ `full` 64-bit metric.
- `WEIGHT_STRUCT` ⇒ `{var1_dw:32, var2_w:16, var3_w:16}` triple (load-latency, retire-latency, instr-latency on Intel).

REQ-52 ABI stability:
- New `attr.size` versions are append-only; old size always accepted with zero-fill.
- New `sample_type` bits append at higher positions; existing bit semantics frozen.
- New `PERF_RECORD_*` types append at higher numbers; existing record layouts frozen.
- New `perf_event_mmap_page` fields go into the `__reserved[116*8]` pad region.

REQ-53 Permission gating (`perf_event_paranoid` sysctl):
- 3 (grsec default): only CAP_SYS_ADMIN may open any perf event.
- 2: tracepoints + cpu-wide forbidden to unprivileged.
- 1: cpu-wide tracepoints forbidden.
- 0: unprivileged kernel-profiling permitted.
- -1: all perf events open to all users.
- `CAP_PERFMON` (since 5.8) is a finer-grained alternative to `CAP_SYS_ADMIN` for opening perf events.
- `kernel.kptr_restrict == 2` redacts kernel IPs from PERF_SAMPLE_IP for unprivileged callers.

REQ-54 `attr.exclude_callchain_kernel` / `_user`:
- Trim callchain in the corresponding ring.

REQ-55 `PERF_FLAG_FD_OUTPUT` (syscall flag):
- New event redirects its ring-buffer output to `group_fd`'s ring; the new event itself does not need an mmap'd buffer.

REQ-56 `PERF_FLAG_PID_CGROUP`:
- `pid` argument reinterpreted as a cgroup file descriptor; only valid with `cpu >= 0`.

REQ-57 `PERF_FLAG_FD_CLOEXEC`:
- Returned fd has `O_CLOEXEC` set.

REQ-58 `PERF_FLAG_FD_NO_GROUP`:
- New event not joined into `group_fd`'s group (used with `FD_OUTPUT`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `attr_size_forward_compat` | INVARIANT | per-open: copy = min(user size, kernel max); zero-fill rest. |
| `attr_size_too_large_rejected` | INVARIANT | per-open: user size > kernel max ⇒ -E2BIG. |
| `type_in_range` | INVARIANT | per-open: type ∈ {HARDWARE, SOFTWARE, TRACEPOINT, HW_CACHE, RAW, BREAKPOINT, dyn_pmu}. |
| `sample_type_mask` | INVARIANT | per-open: sample_type ⊆ (PERF_SAMPLE_MAX - 1). |
| `branch_sample_type_requires_branch_stack` | INVARIANT | per-open: branch_sample_type non-zero ⇒ sample_type has BRANCH_STACK; or rejected. |
| `read_format_group_leader_only` | INVARIANT | per-read: GROUP flag + non-leader fd ⇒ -EINVAL. |
| `mmap_page_seqlock` | INVARIANT | per-mmap_page update: lock incremented odd before write, even after; readers observe atomic snapshot. |
| `ring_buffer_head_tail_ordering` | INVARIANT | per-record-publish: smp_wmb between payload write and head store. |
| `aux_offset_above_data` | INVARIANT | per-mmap: aux_offset ≥ data_offset + data_size. |
| `ioctl_magic_check` | INVARIANT | per-ioctl: _IOC_TYPE == '$' else -ENOTTY. |
| `sigtrap_sig_data_reflected` | INVARIANT | per-overflow with sigtrap: siginfo_t::si_perf_data == attr.sig_data. |
| `remove_on_exec_drops_event` | INVARIANT | per-execve: events with remove_on_exec gone from new mm. |
| `paranoid_gate` | INVARIANT | per-open: perf_event_paranoid ≥ 2 ∧ tracepoint ∧ ¬CAP_PERFMON ⇒ -EACCES. |
| `query_bpf_enospc` | INVARIANT | per-IOC_QUERY_BPF: ids_len < prog_cnt ⇒ -ENOSPC and prog_cnt filled. |

### Layer 2: TLA+

`uapi/perf_event.tla`:
- Models per-event state machine (Opened → {Enabled, Disabled} → {Refresh, AutoDisable} → Closed), ring-buffer head/tail, AUX head/tail, group attach, BPF attach.
- Properties:
  - `safety_attr_size_monotonic` — kernel never reads beyond user-supplied size.
  - `safety_ring_buffer_head_advances_monotonically` — data_head only increments under writer lock.
  - `safety_reader_never_reads_past_head` — for any reader, consumed ≤ data_head (modulo wrap).
  - `safety_mmap_page_lock_seqlock` — odd lock value ⇒ writer in flight ⇒ reader retries.
  - `safety_aux_disjoint` — aux_offset region disjoint from data_offset region.
  - `safety_group_pinning` — pinned events scheduled before non-pinned on contention.
  - `safety_paranoid_gate_holds` — unprivileged + paranoid ≥ N ⇒ class-N events refused.
  - `safety_remove_on_exec_drops` — after execve, event with remove_on_exec ∉ task.perf_events.
  - `liveness_overflow_eventually_records` — every overflow eventually appears in ring (modulo throttle) or is counted as LOST.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Perf::open` post: ret > 0 ⇒ valid fd with attached event ∧ all field validations passed | `Perf::open` |
| `Perf::mmap` post: header at offset 0 ∧ data ring power-of-two ∧ aux_offset ≥ data_offset + data_size | `Perf::mmap` |
| `Perf::ioctl(ENABLE)` post: event.state == ENABLED | `Perf::ioctl_enable` |
| `Perf::ioctl(DISABLE)` post: event.state == DISABLED | `Perf::ioctl_disable` |
| `Perf::ioctl(SET_OUTPUT, sibling)` post: event.ring == sibling.ring | `Perf::ioctl_set_output` |
| `Perf::ioctl(SET_BPF, prog)` post: prog.type compatible ⇒ attached; else -EINVAL | `Perf::ioctl_set_bpf` |
| `Perf::ioctl(QUERY_BPF, q)` post: q.prog_cnt = total_attached ∧ q.ids[0..min(ids_len, prog_cnt)] populated | `Perf::ioctl_query_bpf` |
| `Perf::record_sample` post: payload appears in [data_head_old, data_head_new) ∧ smp_wmb between | `Perf::record_sample` |
| `Perf::read` post: bytes returned = sizeof per read_format layout | `Perf::read` |
| `Perf::overflow_sigtrap` post: SIGTRAP delivered ∧ si_perf_data = sig_data | `Perf::overflow_sigtrap` |
| `Perf::callchain_user_deferred` post: full callchain reconstructible by stitching SAMPLE + CALLCHAIN_DEFERRED with matching cookie | `Perf::deferred_callchain` |

### Layer 4: Verus/Creusot functional

Per `Documentation/admin-guide/perf-security.rst`, `Documentation/userspace-api/perf_ring_buffer.rst`, and `kernel/events/core.c`:
- `perf_event_open` op-table semantic equivalence with upstream.
- `perf_event_mmap_page` self-monitor loop equivalence (seqlock, rdpmc, time conversion).
- Ring-buffer producer/consumer equivalence (head smp_wmb, tail smp_mb).
- Record layouts (all `PERF_RECORD_*`) equivalence with upstream `perf_output_*` paths.
- Sample field ordering equivalence with `perf_output_sample`.
- BPF attach equivalence with `bpf_perf_event_ioctl` plumbing.
- Auto-throttle equivalence with `__perf_event_overflow` rate limiter.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

perf_event_open UAPI reinforcement:

- **Per-attr.size strict bound** — defense against per-overread of user buffer.
- **Per-attr.type whitelist** — defense against per-unknown-PMU enumeration.
- **Per-sample_type / branch_sample_type / read_format mask check** — defense against per-future-bit-leak.
- **Per-mmap_page seqlock** — defense against per-torn-read of self-monitor state.
- **Per-ring-buffer head/tail smp_wmb/smp_mb** — defense against per-stale-data sample reads.
- **Per-AUX area disjoint from data area** — defense against per-overlap data corruption.
- **Per-`perf_event_paranoid` gate** — defense against per-unprivileged kernel-profile leak.
- **Per-CAP_PERFMON gate** — defense against per-CAP_SYS_ADMIN over-privilege.
- **Per-`kptr_restrict` redaction on PERF_SAMPLE_IP** — defense against per-kASLR leak.
- **Per-exclude_kernel / exclude_callchain_kernel** — defense against per-kernel-IP sampling for unprivileged.
- **Per-`remove_on_exec`** — defense against per-event-leak across setuid execve.
- **Per-`sigtrap`+`sig_data` synchronous** — defense against per-signal-coalesce loss.
- **Per-PERF_RECORD_LOST / _LOST_SAMPLES bookkeeping** — defense against per-silent-drop blind-spots.
- **Per-auto-throttle on `perf_event_max_sample_rate`** — defense against per-DoS via high-frequency sampling.
- **Per-`sample_max_stack` ≤ `perf_event_max_stack`** — defense against per-unbounded callchain memory.
- **Per-IOC magic `'$'` strict** — defense against per-misrouted ioctl.
- **Per-`PERF_EVENT_IOC_SET_FILTER` length-bound** — defense against per-OOM via huge filter string.
- **Per-`PERF_EVENT_IOC_QUERY_BPF` -ENOSPC instead of partial-fill+truncation** — defense against per-info-loss.
- **Per-`PERF_FLAG_FD_CLOEXEC`** — defense against per-fd-leak across exec.
- **Per-`attr.aux_action` mutually-exclusive bit check** — defense against per-conflicting-AUX-state.
- **Per-`attr.precise_ip = 3` strict-zero-skid** — required for security-relevant control-flow.
- **Per-`PERF_RECORD_TEXT_POKE` only emitted for caller-visible-text** — defense against per-kernel-text disclosure.

### grsecurity/pax-style reinforcement

- **GRKERNSEC_PERF_HARDEN** — `kernel.perf_event_paranoid` defaulted to 3 (kernel/cpu/tracepoint counters all require CAP_PERFMON); the sysctl is monotonic-down-only under grsec (cannot be relaxed at runtime once tightened); unprivileged callers cannot use `PERF_FLAG_PID_CGROUP`, cannot open `PERF_TYPE_BREAKPOINT`, cannot use `PERF_SAMPLE_REGS_INTR` / `_REGS_USER` / `_STACK_USER` (regs/stack disclosure surface).
- **PAX_RANDKSTACK** — `perf_event_open(2)` entry path re-randomises kernel stack base before allocating the (sometimes large) `PerfEvent` structure; same for the mmap path that reserves the ring-buffer pages, so repeated open attempts cannot leak stack-layout via probe-and-trigger timing.
- **GRKERNSEC_HIDESYM** — `PERF_SAMPLE_IP` and callchain `ips[]` for unprivileged callers are filtered: kernel-mode IPs are replaced with a sentinel (zero) before being written into the ring; `PERF_RECORD_KSYMBOL` is suppressed for unprivileged listeners (the ksym name + address is a kASLR/kptr leak); `PERF_RECORD_MMAP*` of kernel modules suppressed for non-CAP_SYSLOG / non-CAP_PERFMON; `PERF_RECORD_TEXT_POKE` only delivered to CAP_SYS_ADMIN under grsec policy.
- **CAP_PERFMON gate** — installing any event with `attr.type ∈ {TRACEPOINT, BREAKPOINT, RAW}` or `sample_type & (REGS_USER | REGS_INTR | STACK_USER | CALLCHAIN with kernel)` requires `CAP_PERFMON` (or `CAP_SYS_ADMIN` on pre-5.8 backports); `PERF_SAMPLE_RAW` on tracepoints additionally requires `CAP_PERFMON`; `attr.precise_ip > 0` (PEBS) requires `CAP_PERFMON` because the PMU may be wedged on misuse.
- **Tracepoint exposure restricted** — `PERF_TYPE_TRACEPOINT` events for kernel-data tracepoints (`sched_switch`, `kmem_*`, `mm_*`) gated on `CAP_PERFMON`; userspace USDT tracepoints (in mmap'd userland code) permitted to the owning task only; `PERF_EVENT_IOC_SET_FILTER` strings sanitized for `\0` / overlength / printf-format-attack patterns before being passed to the tracing filter engine.
- **PaX UDEREF/USERCOPY** — `copy_from_user` of `struct perf_event_attr` uses hardened-usercopy sized strictly to `min(user attr.size, kernel max)`; `IOC_SET_FILTER` string copy uses `strncpy_from_user` with `PAGE_SIZE` bound; `IOC_QUERY_BPF` `ids[]` copy bounded by `ids_len * sizeof(u32)`; `IOC_MODIFY_ATTRIBUTES` re-validates the full attr.
- **PAX_KERNEXEC** — the mmap'd ring-buffer pages and AUX pages are PROT_READ|PROT_WRITE only (never PROT_EXEC); kernel-side ring-buffer manipulation lives in code pages that are W^X (never both writable and executable simultaneously); BPF programs attached via `IOC_SET_BPF` run under the BPF-JIT W^X policy (write-then-seal).
- **GRKERNSEC_KMEM** — perf does not expose `/dev/mem` / `/dev/kmem`, but the principle applies: `PERF_SAMPLE_PHYS_ADDR` is gated on `CAP_PERFMON` (it leaks the kernel direct map slot); `PERF_SAMPLE_DATA_PAGE_SIZE` / `_CODE_PAGE_SIZE` gated on CAP_PERFMON; per-record `sample_id` carries only opaque kernel-assigned ids, never pointers; `IOC_ID` returns a 64-bit handle uncorrelated with kernel addresses.
- **GRKERNSEC_HARDEN_PTRACE** — opening a perf event on `pid != current` requires the same gating as `ptrace(PTRACE_ATTACH)` under grsec policy (`gr_ptrace_allowed`); cross-uid / cross-gid perf attach is rejected for unprivileged callers; `PERF_RECORD_SWITCH_CPU_WIDE` and `PERF_RECORD_FORK/EXIT` for non-owned tasks suppressed.
- **Per-`attr.sigtrap` audited** — installing a `sigtrap` perf event on a different task is logged in the audit subsystem (AUDIT_PERFMON record) under grsec policy; cross-task sigtrap requires `CAP_PERFMON` and is rate-limited.
- **Per-ring-buffer size cap** — total mmap'd perf memory per unprivileged user bounded by `RLIMIT_MEMLOCK` (or `perf_event_mlock_kb` sysctl), with grsec enforcing a hard cap regardless of rlimit; defense against per-OOM via large rings.
- **Per-`sample_max_stack` ≤ 127** (`PERF_MAX_STACK_DEPTH`) and per-`perf_event_max_stack` sysctl bounded — defense against per-callchain CPU exhaustion.
- **Per-`PERF_RECORD_TEXT_POKE`** carries instruction bytes — restricted to CAP_SYS_ADMIN under grsec (otherwise reveals kernel-text-as-mapped, important for ROP-target hunting).
- **Per-`PERF_EVENT_IOC_SET_BPF`** BPF program type strictly checked against event type (PERF_EVENT | KPROBE | TRACEPOINT); cross-type attach rejected with -EINVAL; under grsec, BPF program load also requires `CAP_BPF` (or `CAP_SYS_ADMIN` on pre-5.8) and per-program verifier slot quota.
- **Per-AUX area (Intel PT / CoreSight / SPE / IBS)** — opening an AUX-producing PMU (`PERF_TYPE_RAW` against `intel_pt`, etc.) requires `CAP_PERFMON`; AUX pages are written only by hardware (DMA-mapped), never by kernel pointer dereference; the AUX format flag (`PERF_AUX_FLAG_PMU_FORMAT_TYPE_MASK`) is PMU-validated to refuse spoofed formats.
- **Per-`PERF_FLAG_PID_CGROUP`** — cgroup-fd must be in the same user-namespace as the caller; cross-namespace cgroup attach rejected; defense against per-cross-container surveillance.
- **Per-`PERF_RECORD_NAMESPACES`** — namespace inode/dev numbers exposed only when caller has `CAP_PERFMON` (or owns all namespaces in the target's chain); defense against per-namespace-fingerprinting.
- **Per-`attr.disabled = 0` + `attr.enable_on_exec`** — combination audited (a pre-armed event that fires on next exec is a classic privilege-acquisition watchpoint); under grsec, only owner-task allowed to install such events on itself.
- **Per-`attr.inherit`** — inherit across `fork()` audited when the parent has `CAP_PERFMON` and the child is or will be setuid; defense against per-setuid-leak via inherited counters.
- **Per-`attr.remove_on_exec`** — recommended for any unprivileged event under grsec policy; the kernel emits a warning if a CAP_PERFMON-installed event is left across execve without it (potential setuid-binary observation).
- **Per-`PERF_RECORD_BPF_EVENT`** — gated on CAP_BPF + CAP_PERFMON; without both, the record stream omits BPF prog load/unload events (these leak prog ids and tags).
- **Per-`PERF_RECORD_CGROUP`** — gated on CAP_PERFMON for cgroups other than the caller's own; defense against per-cgroup-topology enumeration.
- **Per-`siginfo_t::si_perf_data`** — `sig_data` is reflected back as opaque; the kernel does not expose any pointer values via this field, and the kernel rejects setting `sig_data` to look like a kernel pointer (heuristic: top bits in canonical-kernel range refused) under grsec policy.
- **Per-`PERF_EVENT_IOC_MODIFY_ATTRIBUTES`** — only a narrow subset of fields modifiable live (sample_period, sample_freq, filter); type/config changes refused with -EINVAL to prevent post-open privilege smuggle.
- **Per-`mmap()` of perf fd** — PROT_EXEC requested ⇒ refused unconditionally (perf rings are never executable); PROT_WRITE on header page only valid for the data_tail field (kernel re-validates each tail update is within data area).

