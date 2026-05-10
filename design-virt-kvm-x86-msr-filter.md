---
title: "Tier-3: arch/x86/kvm/x86.c — KVM_X86_SET_MSR_FILTER (per-VM MSR access policy + userspace-routed MSR ioctls)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM_X86_SET_MSR_FILTER lets userspace VMM (qemu) install a per-VM allow/deny policy for guest RDMSR/WRMSR — each ioctl-supplied filter is a list of (MSR-range, bitmap, action) tuples; on guest MSR access, KVM checks against active filter; if denied, KVM injects #GP to guest; if user-routed, KVM exits to userspace with KVM_EXIT_X86_RDMSR/_WRMSR for QEMU to emulate (e.g., per-policy CPUID-feature emulation, fake TSC, paravirt MSRs not natively supported). Critical for: confidential-VM minimum-attack-surface configurations, per-VM custom MSR emulation, sandboxing.

This Tier-3 covers MSR-filter related code in `arch/x86/kvm/x86.c` (~200-300 lines).

### Acceptance Criteria

- [ ] AC-1: KVM_X86_SET_MSR_FILTER with DEFAULT_ALLOW + range denying MSR_IA32_TSC_ADJUST: guest WRMSR(MSR_IA32_TSC_ADJUST) returns #GP.
- [ ] AC-2: DEFAULT_DENY + range allowing only specific MSRs: any other MSR access denied.
- [ ] AC-3: Userspace-routed MSR: filter marks MSR as routed; guest RDMSR exits to userspace with KVM_EXIT_X86_RDMSR.
- [ ] AC-4: Multi-range filter: 16 ranges configured; per-MSR routes correctly to first-match range.
- [ ] AC-5: Filter swap: change filter mid-VM; per-vCPU sees new filter on next vmenter.
- [ ] AC-6: Spec-violation defense: filter with > 16 ranges returns -E2BIG.
- [ ] AC-7: bitmap copy-from-user fault: invalid user pointer; ioctl returns -EFAULT.
- [ ] AC-8: Userspace MSR completion: QEMU emulates RDMSR; writes back via KVM_SET_MSR; subsequent RDPMC returns emulated value.
- [ ] AC-9: kvm-unit-tests `msr_filter` test passes.
- [ ] AC-10: Performance: per-allowed-MSR access < 10ns overhead vs no-filter.

### Architecture

`KvmX86MsrFilter` per-VM:

```
struct KvmX86MsrFilter {
  ranges: KVec<MsrBitmapRange>,
  default_allow: bool,
  count: u32,
}

struct MsrBitmapRange {
  flags: u32,                                  // READ / WRITE / USER_SPACE
  nmsrs: u32,
  base: u32,                                    // starting MSR index
  bitmap: KBox<[u8]>,                           // (nmsrs + 7) / 8 bytes
}
```

`KvmX86MsrFilter::set(kvm, user_filter)`:
1. Acquire kvm.lock.
2. new_filter := alloc.
3. new_filter.default_allow := !(user_filter.flags & KVM_MSR_FILTER_DEFAULT_DENY).
4. For i in 0..user_filter.count:
   - kvm_x86_msr_filter_range_init(&new_filter.ranges[i], &user_filter.ranges[i]).
   - copy_from_user bitmap.
5. old_filter := rcu_dereference(kvm.arch.msr_filter).
6. rcu_assign_pointer(kvm.arch.msr_filter, new_filter).
7. synchronize_srcu(&kvm.arch.msr_filter_srcu).
8. kvm_make_request(KVM_REQ_MSR_FILTER_CHANGED, all-vCPUs).
9. kfree old_filter.
10. Release lock.

`KvmX86MsrFilter::is_allowed(vcpu, msr, type)`:
1. idx := srcu_read_lock(&kvm.arch.msr_filter_srcu).
2. filter := rcu_dereference(kvm.arch.msr_filter).
3. For each range in filter.ranges:
   - If range.flags & type AND msr ∈ [range.base, range.base + range.nmsrs):
     - bit := test_bit(msr - range.base, range.bitmap).
     - allow := filter.default_allow ? !bit : bit.
     - srcu_read_unlock(idx); return allow.
4. allow := filter.default_allow.
5. srcu_read_unlock(idx).
6. Return allow.

`KvmX86MsrFilter::user_space(vcpu, msr, op, data, ...)`:
1. vcpu.kvm_run.exit_reason = (op == READ) ? KVM_EXIT_X86_RDMSR : KVM_EXIT_X86_WRMSR.
2. vcpu.kvm_run.msr.error = 0.
3. vcpu.kvm_run.msr.index = msr.
4. If op == WRITE: vcpu.kvm_run.msr.data = data.
5. vcpu.run_state = COMPLETE_USERSPACE_IO.
6. Return -KVM_EUSERSPACE.

`KvmX86MsrFilter::complete_user_space(vcpu)`:
1. status := vcpu.kvm_run.msr.error.
2. If status: inject #GP to guest.
3. Else if vcpu.kvm_run.exit_reason == _RDMSR:
   - Set vcpu.regs.rax = vcpu.kvm_run.msr.data & 0xFFFFFFFF.
   - Set vcpu.regs.rdx = vcpu.kvm_run.msr.data >> 32.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- Per-vendor MSR handling (covered in `x86-vmx.md` / `x86-svm.md` Tier-3s)
- KVM_X86_SET_MCE filter (different ioctl; covered separately)
- KVM_CAP_X86_USER_SPACE_MSR mask (related but separate ioctl)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_x86_msr_filter` | per-VM filter state | `kernel::kvm::x86::msr_filter::KvmX86MsrFilter` |
| `struct msr_bitmap_range` | per-range allow/deny bitmap | `MsrBitmapRange` |
| `kvm_x86_set_msr_filter(kvm, &user_filter)` | KVM_X86_SET_MSR_FILTER ioctl | `KvmX86MsrFilter::set` |
| `kvm_msr_allowed(vcpu, msr, type)` | per-vCPU per-access check | `KvmX86MsrFilter::is_allowed` |
| `kvm_msr_user_space(vcpu, msr, op, data, ...)` | userspace-route MSR access | `KvmX86MsrFilter::user_space` |
| `kvm_set_msr_filter(filter, &range)` | per-range install | `KvmX86MsrFilter::add_range` |
| `kvm_clear_msr_filter(filter)` | clear all ranges | `KvmX86MsrFilter::clear` |
| `KVM_MSR_FILTER_DEFAULT_DENY` / `_DEFAULT_ALLOW` | filter mode flags | UAPI |
| `KVM_MSR_FILTER_RANGE_VALID_MASK` | valid range flags | UAPI |
| `KVM_MSR_FILTER_READ` / `_WRITE` | per-access type | UAPI |
| `KVM_EXIT_X86_RDMSR` / `_WRMSR` | userspace-exit reason | UAPI |
| `kvm_complete_msr_user_space(vcpu)` | post-userspace-emulation completion | `KvmX86MsrFilter::complete_user_space` |

### compatibility contract

REQ-1: `struct kvm_msr_filter` (UAPI):
- `flags` (KVM_MSR_FILTER_DEFAULT_DENY / _ALLOW).
- `ranges[KVM_MSR_FILTER_MAX_RANGES = 16]`: per-range descriptor.

REQ-2: Per-range:
- `flags` (KVM_MSR_FILTER_READ | _WRITE).
- `nmsrs` (number of MSRs in this range).
- `base` (starting MSR index).
- `bitmap` (user-pointer to bitmap of nmsrs bits; bit-set = allow if DEFAULT_DENY else deny).

REQ-3: Per-VM filter state:
- `filter_lock` (mutex for filter mutation).
- `current_filter` (active filter; RCU-pointer).
- `default_allow` (bool, derived from flags).

REQ-4: KVM_X86_SET_MSR_FILTER flow:
1. Validate filter:
   - flags ⊆ KVM_MSR_FILTER_VALID_MASK.
   - Per-range: flags ⊆ KVM_MSR_FILTER_RANGE_VALID_MASK.
   - bitmap_size = (nmsrs + 7) / 8 ≤ KVM_MSR_FILTER_MAX_BITMAP_SIZE.
2. Allocate new kvm_x86_msr_filter.
3. For each range: copy_from_user bitmap; install in new filter.
4. RCU-replace kvm.arch.msr_filter = new.
5. synchronize_srcu (to wait readers).
6. Free old filter.

REQ-5: Per-MSR access check (`kvm_msr_allowed`):
1. Acquire SRCU read lock.
2. filter := rcu_dereference(kvm.arch.msr_filter).
3. For each range in filter.ranges:
   - If range.flags & access_type AND msr ∈ [range.base, range.base + range.nmsrs):
     - bit := (msr - range.base) % 8 of bitmap.
     - If filter.default_allow:
       - allow if bit clear; else deny.
     - Else (default_deny):
       - allow if bit set; else deny.
   - Match found: stop iteration.
4. If no match: return filter.default_allow.
5. SRCU release.

REQ-6: Per-MSR access fail handling:
- Native (in-kernel) RDMSR/WRMSR with denied MSR:
  - Inject #GP to guest.
- Userspace-routed (KVM_MSR_FILTER_FLAG_USER_SPACE):
  - Return to userspace with KVM_EXIT_X86_RDMSR/_WRMSR.
  - Userspace emulates + writes back via KVM_SET_MSR ioctl.
  - kvm_complete_msr_user_space: handle return.

REQ-7: Per-vCPU filter cache:
- Per-vCPU may cache common-MSR allow-list to avoid RCU-walk on hot path.
- KVM_REQ_MSR_FILTER_CHANGED set on filter update; per-vCPU invalidates cache on next vmenter.

REQ-8: KVM_CAP_X86_USER_SPACE_MSR (additional):
- `mask` of MSR types that should always go to userspace (e.g., MSR_TYPE_GP / _IDLE / _UNKNOWN / _FILTER).
- Sets per-VM behavior independent of filter.

REQ-9: KVM_MSR_FILTER_MAX_BITMAP_SIZE = 0x600 (1536 bytes = 12288 bits).

REQ-10: KVM_MSR_FILTER_MAX_RANGES = 16.

REQ-11: Live migration:
- Per-VM filter not migrated by default (userspace re-installs at destination).
- KVM_GET_MSR_FILTER ioctl available for explicit fetch.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `range_count_bounded` | INVARIANT | per-filter.count ≤ KVM_MSR_FILTER_MAX_RANGES. |
| `bitmap_size_bounded` | INVARIANT | per-range bitmap-size ≤ KVM_MSR_FILTER_MAX_BITMAP_SIZE. |
| `srcu_protected_filter` | UAF | per-filter accessed under SRCU read lock; defense against use-after-free during set. |
| `bitmap_idx_bounded` | OOB | per-MSR bit-index < range.nmsrs; defense against bitmap OOB read. |

### Layer 2: TLA+

`virt/kvm/msr_filter_dispatch.tla`:
- Per-MSR access state ∈ {Native, Allowed, Denied, RoutedUserSpace}.
- Properties:
  - `safety_first_match_wins` — first matching range determines allow/deny.
  - `safety_default_when_no_match` — no-range-match falls to default_allow.
  - `liveness_user_space_completes` — RoutedUserSpace eventually returns via complete_user_space.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmX86MsrFilter::set` post: kvm.arch.msr_filter atomic-swapped; old freed after SRCU sync | `KvmX86MsrFilter::set` |
| `KvmX86MsrFilter::is_allowed` post: returned bool reflects filter+range match | `KvmX86MsrFilter::is_allowed` |
| `KvmX86MsrFilter::user_space` post: vcpu.kvm_run populated with MSR exit info | `KvmX86MsrFilter::user_space` |
| Per-vCPU filter cache invalidated on KVM_REQ_MSR_FILTER_CHANGED | invariants on filter swap |

### Layer 4: Verus/Creusot functional

`Per-MSR access: filter check → allow ⇒ native RDMSR/WRMSR; deny ⇒ #GP; user_space ⇒ exit to userspace` semantic equivalence: per-MSR per-access the resulting effect (native execution, #GP injection, or userspace exit) matches userspace policy.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

MSR-filter-specific reinforcement:

- **MAX_RANGES + MAX_BITMAP_SIZE caps** — defense against unbounded user-supplied filter consuming kernel memory.
- **bitmap copy-from-user with size bound** — defense against malicious user pointer overflow.
- **flags ⊆ VALID_MASK enforced** — defense against unknown flag bits causing future-incompat issues.
- **RCU + SRCU protected swap** — defense against partial-filter-visibility during update.
- **Per-vCPU KVM_REQ_MSR_FILTER_CHANGED notification** — defense against stale per-vCPU filter cache.
- **first-match wins iteration** — defense against ambiguous range overlap.
- **default-allow vs default-deny explicit** — defense against accidental policy gap.
- **userspace_msr exit return-from-userspace validated** — defense against userspace returning malformed data.
- **Per-MSR access check on hot path SRCU-fast** — defense against per-access mutex contention.
- **Filter-set under kvm.lock + ioctl-privileged** — defense against unauthorized filter modification.
- **Per-VM filter not migrated by default** — defense against post-migrate stale filter referencing source-userspace allocations.
- **#GP injection on deny** — defense against silent-failure on denied MSR.

