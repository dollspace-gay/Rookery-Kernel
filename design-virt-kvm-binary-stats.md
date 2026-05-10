---
title: "Tier-3: virt/kvm/binary_stats.c — KVM binary-stats interface (per-VM / per-vCPU statistics)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-VM and per-vCPU binary-stats expose performance counters (vmexit reasons, MMU-faults, halt-poll-time histograms, etc.) via dedicated read-only fd from `KVM_GET_STATS_FD` ioctl. Per-fd reads return a binary blob: header + per-stat descriptors + per-stat values (u64). Each stat described by name + flags (CUMULATIVE / INSTANT / PEAK / LINEAR_HIST / LOG_HIST), unit (NONE / BYTES / SECONDS / CYCLES / BOOLEAN), and exponent. Replaces traditional debugfs/sysfs stats with structured binary ABI for efficient userspace tooling (kvm_stat / kvm_stats / virt-host-validate). Critical for: per-VM performance monitoring at scale.

This Tier-3 covers `binary_stats.c` (~144 lines).

### Acceptance Criteria

- [ ] AC-1: KVM_GET_STATS_FD on VM-fd: returns valid stats-fd.
- [ ] AC-2: read(stats-fd, header_buf, 32): returns kvm_stats_header.
- [ ] AC-3: header.num_desc > 0; desc_offset + size ≤ data_offset.
- [ ] AC-4: read(stats-fd, desc_buf, ...): returns kvm_stats_desc array + name strings.
- [ ] AC-5: read(stats-fd, data_buf, ...): returns u64 array of values.
- [ ] AC-6: KVM_GET_STATS_FD on vCPU-fd: per-vCPU stats fd.
- [ ] AC-7: Stat name "halt_exits" type=CUMULATIVE: increments per HLT vmexit.
- [ ] AC-8: Stat name "mmu_shadow_zapped" type=CUMULATIVE: increments per shadow-zap.
- [ ] AC-9: Stat name "halt_poll_success_ns" type=LOG_HIST: per-bucket count.
- [ ] AC-10: write(stats-fd, ...): -EINVAL (read-only).
- [ ] AC-11: lseek(stats-fd, ...): supported (random access).

### Architecture

Per-VM stats:

```
struct KvmVmStats {
  generic: KvmVmStatsCommon,                     // arch-common
  // x86-specific:
  mmu_shadow_zapped: u64,
  mmu_pte_write: u64,
  mmu_unsync: u64,
  mmu_flooded: u64,
  mmu_recycled: u64,
  mmu_pde_zapped: u64,
  remote_tlb_flush: u64,
  ...
}
```

Per-vCPU stats:

```
struct KvmVcpuStats {
  generic: KvmVcpuStatsCommon,
  pf_fixed: u64,
  pf_guest: u64,
  tlb_flush: u64,
  invlpg: u64,
  exits: u64,
  io_exits: u64,
  mmio_exits: u64,
  signal_exits: u64,
  irq_window_exits: u64,
  nmi_window_exits: u64,
  l1d_flush: u64,
  halt_exits: u64,
  request_irq_exits: u64,
  irq_exits: u64,
  host_state_reload: u64,
  fpu_reload: u64,
  insn_emulation: u64,
  insn_emulation_fail: u64,
  hypercalls: u64,
  irq_injections: u64,
  nmi_injections: u64,
  ...
  halt_poll_success_ns: HistU64<32>,             // log hist
  halt_poll_fail_ns: HistU64<32>,
}
```

Per-fd descriptor:

```
struct KvmStatsHeader {
  flags: u32,
  name_size: u32,
  num_desc: u32,
  id_offset: u32,
  desc_offset: u32,
  data_offset: u32,
}

struct KvmStatsDesc {
  flags: i16,                                    // type + unit + base
  exponent: i16,
  size: u16,                                     // histogram bucket count
  reserved: u16,
  bucket_size: u32,                              // for LINEAR_HIST
  name: VarLenStr,                               // NUL-terminated
}
```

`Stats::get_vm_fd(kvm) -> Result<i32>`:
1. Allocate anon-inode w/ stats_fops.
2. Set inode private = (kvm, KVM_STATS_VM).
3. Return fd.

`Stats::get_vcpu_fd(vcpu) -> Result<i32>`:
1. Same but with vCPU + KVM_STATS_VCPU.

`Stats::read(file, buf, count, ppos) -> Result<usize>`:
1. (kvm-or-vcpu, kind) = file.private.
2. id_str = format_id(kvm-or-vcpu, kind).
3. desc = if kind == VM: KVM_VM_STATS_DESC; else: KVM_VCPU_STATS_DESC.
4. header = compute_header(desc).
5. src = if kind == VM: kvm.stat else vcpu.stat.
6. Per-ppos region: copy from header / id_str / desc[] / data[u64].

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM debugfs stats (legacy; covered separately if retained)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_stats_header` | per-fd header | UAPI |
| `struct kvm_stats_desc` | per-stat descriptor | UAPI |
| `kvm_stats_read()` | per-fd read | `Stats::read` |
| `KVM_GET_STATS_FD` (VM-level) | per-VM fd ioctl | UAPI |
| `KVM_GET_STATS_FD` (vCPU-level) | per-vCPU fd ioctl | UAPI |
| `KVM_STATS_TYPE_CUMULATIVE` | per-stat type | UAPI |
| `KVM_STATS_TYPE_INSTANT` | per-stat type | UAPI |
| `KVM_STATS_TYPE_PEAK` | per-stat type | UAPI |
| `KVM_STATS_TYPE_LINEAR_HIST` | per-stat type | UAPI |
| `KVM_STATS_TYPE_LOG_HIST` | per-stat type | UAPI |
| `KVM_STATS_UNIT_NONE/BYTES/SECONDS/CYCLES/BOOLEAN` | per-unit | UAPI |
| `KVM_STATS_BASE_POW10/POW2` | per-base for exponent | UAPI |
| `kvm_get_vm_stats_fd()` | per-VM fd creation | `Stats::get_vm_fd` |
| `kvm_get_vcpu_stats_fd()` | per-vCPU fd creation | `Stats::get_vcpu_fd` |

### compatibility contract

REQ-1: KVM_GET_STATS_FD ioctl:
- Per-VM-fd: ioctl on /dev/kvm-vm-fd → returns stats-fd.
- Per-vCPU-fd: ioctl on /dev/kvm-vcpu-fd → returns stats-fd.
- Per-fd is anon-inode, read-only.

REQ-2: Per-fd binary layout:
- Bytes [0..hdr_size]: kvm_stats_header (32 bytes).
- Bytes [hdr.desc_offset..]: array of kvm_stats_desc + zero-terminated name strings.
- Bytes [hdr.data_offset..]: array of u64 values per-stat.

REQ-3: Per-kvm_stats_header:
```c
struct kvm_stats_header {
  __u32 flags;
  __u32 name_size;
  __u32 num_desc;
  __u32 id_offset;
  __u32 desc_offset;
  __u32 data_offset;
};
```

REQ-4: Per-kvm_stats_desc:
```c
struct kvm_stats_desc {
  __s16 flags;        // type + unit + base
  __s16 exponent;
  __u16 size;         // for histograms: bucket count
  __u16 reserved;
  __u32 bucket_size;  // for hist
  char name[];        // NUL-terminated
};
```

REQ-5: Per-flags layout:
- Bits[3:0] type (CUMULATIVE / INSTANT / PEAK / LINEAR_HIST / LOG_HIST).
- Bits[7:4] unit (NONE / BYTES / SECONDS / CYCLES / BOOLEAN).
- Bits[11:8] base (POW10 / POW2).

REQ-6: Per-stat-types:
- CUMULATIVE: monotonic counter.
- INSTANT: gauge (current value).
- PEAK: max-seen.
- LINEAR_HIST: linear-bucket histogram.
- LOG_HIST: log-bucket histogram.

REQ-7: kvm_stats_read(id, header, desc, stats, src, size, offset, buf, count, user):
- Per-read: serve from src struct kvm.stat or kvm.vcpu.stat.
- Per-offset within id_string + desc[] + data[].
- Returns bytes copied.

REQ-8: Per-id_offset region:
- "VM-<pid>-<vmid>" or "vcpu-<id>".
- VM/vCPU-distinguishing string.

REQ-9: Per-VM stats categories (subset):
- mmu_shadow_zapped, mmu_unsync, mmu_flooded, mmu_recycled, lpages, nx_lpage_splits.
- max_mmu_page_hash_collisions.
- max_mmu_rmap_size.

REQ-10: Per-vCPU stats categories (subset):
- nmi_injections, irq_injections, halt_exits.
- exits, io_exits, mmio_exits.
- request_irq_exits, signal_exits.
- pf_fixed, pf_guest, tlb_flush.
- halt_poll_success_ns / fail_ns (histogram).
- l1d_flush.
- guest_mode.
- vcpu_arch.preemption_timer.
- request_irq_window_exits.
- l1_tsc_offset_writes.

REQ-11: Per-userspace tool ABI:
- libkvm-stats / kvm_stat parses the binary.
- Per-stat name allows future-add without ABI break.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `header_offsets_monotonic` | INVARIANT | id_offset ≤ desc_offset ≤ data_offset. |
| `data_within_fd` | INVARIANT | per-data: offset + num_desc × 8 ≤ fd-size. |
| `desc_name_nul_term` | INVARIANT | per-desc.name[len-1] == 0. |
| `fd_read_only` | INVARIANT | stats_fops.write == NULL. |
| `data_u64_aligned` | INVARIANT | data_offset 8-byte-aligned. |

### Layer 2: TLA+

`virt/kvm/binary_stats.tla`:
- Per-VM/vCPU stat increment + per-fd read.
- Properties:
  - `safety_no_write_stats_fd` — per-write to stats-fd ⟹ -EINVAL.
  - `safety_data_consistent_with_desc` — per-data[i] semantics matches desc[i].type.
  - `liveness_stat_increment_visible` — per-incremented stat visible at next read.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Stats::get_vm_fd` post: returned-fd is anon-inode read-only | `Stats::get_vm_fd` |
| `Stats::read` post: returned bytes match per-region layout | `Stats::read` |
| `Stats::compute_header` post: offsets monotonic; sizes consistent | `Stats::compute_header` |

### Layer 4: Verus/Creusot functional

`Per-userspace KVM_GET_STATS_FD → read header → walk desc[] → read data[] → per-name stat value visible` semantic equivalence: per-binary-stats matches Linux kvm.h API doc.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

Binary-stats-specific reinforcement:

- **Per-fd read-only enforced** — defense against userspace mutating kernel-counter.
- **Per-offset bounds-checked** — defense against per-read OOB exposing kernel mem.
- **Per-name-len bounded** — defense against per-name overflow.
- **Per-stat-desc layout fixed** — defense against per-version drift.
- **Per-fd refcount via fget/fput** — defense against UAF.
- **Per-read returns bytes copied (not internal-state ptr)** — defense against per-pointer leak.
- **Per-LINEAR_HIST/LOG_HIST bucket-count valid** — defense against per-config-zero buckets.
- **Per-stat name in fd-string region (separate from desc)** — defense against per-name-injection in desc binary.
- **Per-VM destroy invalidates outstanding fds via fput** — defense against post-VM lingering fd accessing freed kvm.
- **Per-rcu-protected stat reads** — defense against per-mid-read stat-struct realloc.

