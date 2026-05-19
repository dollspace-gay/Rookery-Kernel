# Tier-3: virt/kvm/async_pf.c — KVM async page-fault generic infrastructure

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-async-pf.md
upstream-paths:
  - virt/kvm/async_pf.c (~241 lines)
  - virt/kvm/async_pf.h
  - include/linux/kvm_host.h (kvm_async_pf, kvm_arch_async_page_*)
  - arch/x86/kvm/async_pf.c (arch hooks; covered in x86-async-pf.md)
-->

## Summary

Per-vCPU async-page-fault (APF) infrastructure: when a guest causes a host page-fault that needs swap-in, KVM defers HVA-resolution to a kernel workqueue (`async_pf_execute`) instead of stalling the vCPU thread, schedules the vCPU back-to-userspace as KVM_EXIT_PAGE_NOT_PRESENT (or via paravirt-APF MSR if guest supports it). When the page is ready, the workqueue completes; the vCPU is queued on `done` list and re-resumed via per-arch `kvm_arch_async_page_present`. This generic layer manages the workqueue + per-vCPU queues + lifetime; arch-specific glue (x86 paravirt-APF MSR + #PF (vector 14) injection / interrupt) lives in `arch/x86/kvm/async_pf.c` (covered in `x86-async-pf.md`).

This Tier-3 covers the cross-arch core in `virt/kvm/async_pf.c` (~241 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_async_pf` | per-fault work + state | `KvmAsyncPf` |
| `kvm_async_pf_init()` | per-module init (creates wq) | `Apf::init` |
| `kvm_async_pf_deinit()` | per-module teardown | `Apf::deinit` |
| `async_pf_execute()` | per-fault workqueue handler | `Apf::execute` |
| `kvm_setup_async_pf()` | per-vCPU queue async fault | `Apf::setup_async_pf` |
| `kvm_check_async_pf_completion()` | per-vCPU drain done-queue | `Apf::check_completion` |
| `kvm_async_pf_wakeup_all()` | per-vCPU wake on present | `Apf::wakeup_all` |
| `kvm_arch_async_page_not_present()` | per-arch inject not-present (x86: #PF or token write) | x86 hook |
| `kvm_arch_async_page_present()` | per-arch inject present-event | x86 hook |
| `kvm_arch_async_page_present_queued()` | per-arch interrupt for queued | x86 hook |
| `kvm_arch_async_page_ready()` | per-arch finalize | x86 hook |
| `vcpu->async_pf.queue` | per-vCPU pending list | `VcpuArch::async_pf_queue` |
| `vcpu->async_pf.done` | per-vCPU completion list | `VcpuArch::async_pf_done` |

## Compatibility contract

REQ-1: KVM module init:
- `kvm_async_pf_init`: alloc kmem_cache for KvmAsyncPf entries.
- Single global cache; per-vCPU work_struct embedded in entry.

REQ-2: Per-vCPU async_pf state:
- `queue`: ListHead of pending entries (work submitted).
- `done`: ListHead of completed entries (waiting for vCPU to consume).
- `queued`: u32 count (bounded for backpressure).

REQ-3: kvm_setup_async_pf(vcpu, cr2_or_gpa, hva):
- Validate vcpu.async_pf.queued < ASYNC_PF_PER_VCPU (e.g. 64).
- Allocate KvmAsyncPf entry: { vcpu, cr2_or_gpa, hva, work }.
- INIT_WORK(&entry.work, async_pf_execute).
- entry.notpresent_injected = kvm_arch_async_page_not_present(vcpu, entry).
- list_add(&entry.queue, &vcpu.async_pf.queue).
- vcpu.async_pf.queued++.
- queue_work(kvm_workqueue, &entry.work).

REQ-4: async_pf_execute (wq handler):
- mm = get_task_mm(vcpu.task).
- mmap_read_lock(mm).
- get_user_pages(mm, hva, 1, ...) // resolves via swap-in.
- mmap_read_unlock(mm).
- kvm_arch_async_page_present(vcpu, entry).
- mutex_lock(&vcpu.async_pf.lock).
- list_move(&entry.queue, &vcpu.async_pf.done).
- mutex_unlock.
- kvm_arch_async_page_present_queued(vcpu).
- kvm_vcpu_kick(vcpu).

REQ-5: kvm_check_async_pf_completion(vcpu):
- Per-vmenter (or KVM_REQ_APF_DONE bit set): walk done-list.
- For entry in done:
  - kvm_arch_async_page_ready(vcpu, entry).
  - If !entry.notpresent_injected: kvm_arch_async_page_present(vcpu, entry).
  - list_del; kfree.
  - vcpu.async_pf.queued--.

REQ-6: kvm_async_pf_wakeup_all(vcpu):
- Used at vCPU teardown or APF-disable.
- Drain queue + done lists synchronously (waits for wq to finish).
- Per-entry: kvm_arch_async_page_ready + free.

REQ-7: Per-VM module-lifetime:
- Module exit: flush async_pf_execute work queue (no in-flight refs).
- Per-async-pf: vcpu.task ref taken (mmput releases).

REQ-8: Per-arch APF-MSR (x86 specific):
- See `x86-async-pf.md` Tier-3 for MSR_KVM_ASYNC_PF_EN / _ACK / _INT semantics.

REQ-9: Per-vCPU queued-cap:
- ASYNC_PF_PER_VCPU upper bound prevents per-vCPU unbounded entry alloc.

## Acceptance Criteria

- [ ] AC-1: KVM module load: kvm_async_pf_init succeeds; kmem_cache created.
- [ ] AC-2: Guest write to swapped-out page: kvm_setup_async_pf called.
- [ ] AC-3: vcpu.async_pf.queue length increases.
- [ ] AC-4: async_pf_execute completes: page faulted-in.
- [ ] AC-5: Entry moved from queue to done list.
- [ ] AC-6: kvm_arch_async_page_present_queued kicks vCPU.
- [ ] AC-7: Per-vmenter kvm_check_async_pf_completion drains done.
- [ ] AC-8: vcpu.async_pf.queued decremented to 0 after drain.
- [ ] AC-9: vCPU teardown: kvm_async_pf_wakeup_all flushes; no leaks.
- [ ] AC-10: ASYNC_PF_PER_VCPU exceeded: kvm_setup_async_pf returns false.

## Architecture

Per-fault state:

```
struct KvmAsyncPf {
  link: ListLink,                                // .queue or .done
  vcpu: &Vcpu,
  work: WorkStruct,
  cr2_or_gpa: u64,
  arch_addr: u64,                                // hva
  notpresent_injected: bool,                     // arch already injected #PF
}
```

Per-vCPU state:

```
struct VcpuArchAsyncPf {
  queue: ListHead<KvmAsyncPf>,
  done: ListHead<KvmAsyncPf>,
  queued: AtomicU32,
  lock: Mutex<()>,
}
```

`Apf::init() -> Result<()>`:
1. async_pf_cache = kmem_cache_create("kvm_async_pf", sizeof(KvmAsyncPf), ...).
2. Ok.

`Apf::deinit()`:
1. kmem_cache_destroy(async_pf_cache).

`Apf::execute(work)`:
1. entry = container_of(work, KvmAsyncPf, work).
2. vcpu = entry.vcpu.
3. mm = get_task_mm(vcpu.task).
4. mmap_read_lock(mm).
5. get_user_pages(mm, entry.arch_addr, 1, FOLL_WRITE).
6. mmap_read_unlock(mm).
7. mmput(mm).
8. kvm_arch_async_page_present(vcpu, entry).
9. mutex_lock(&vcpu.async_pf.lock).
10. list_move(&entry.link, &vcpu.async_pf.done).
11. mutex_unlock.
12. kvm_arch_async_page_present_queued(vcpu).
13. kvm_vcpu_kick(vcpu).

`Apf::setup_async_pf(vcpu, cr2_or_gpa, hva) -> bool`:
1. If vcpu.async_pf.queued.load() >= ASYNC_PF_PER_VCPU: return false.
2. entry = kmem_cache_alloc(async_pf_cache).
3. entry.vcpu = vcpu; entry.cr2_or_gpa = cr2_or_gpa; entry.arch_addr = hva.
4. INIT_WORK(&entry.work, Apf::execute).
5. entry.notpresent_injected = kvm_arch_async_page_not_present(vcpu, entry).
6. mutex_lock(&vcpu.async_pf.lock).
7. list_add(&entry.link, &vcpu.async_pf.queue).
8. vcpu.async_pf.queued++.
9. mutex_unlock.
10. queue_work(kvm_async_pf_wq, &entry.work).
11. Return true.

`Apf::check_completion(vcpu)`:
1. mutex_lock(&vcpu.async_pf.lock).
2. while !list_empty(&vcpu.async_pf.done):
   - entry = list_first_entry(&vcpu.async_pf.done).
   - list_del(&entry.link).
   - kvm_arch_async_page_ready(vcpu, entry).
   - if !entry.notpresent_injected: kvm_arch_async_page_present(vcpu, entry).
   - kmem_cache_free(async_pf_cache, entry).
   - vcpu.async_pf.queued--.
3. mutex_unlock.

`Apf::wakeup_all(vcpu)`:
1. /* drain queue: cancel wq + free; wait if in-flight */
2. mutex_lock(&vcpu.async_pf.lock).
3. while !list_empty(&vcpu.async_pf.queue):
   - entry = list_first_entry(&vcpu.async_pf.queue).
   - if work-pending: cancel_work_sync.
   - list_del; kvm_arch_async_page_ready(vcpu, entry); kmem_cache_free.
   - vcpu.async_pf.queued--.
4. while !list_empty(&vcpu.async_pf.done):
   - entry = list_first_entry; list_del; kvm_arch_async_page_ready; free; queued--.
5. mutex_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `queued_le_per_vcpu_max` | INVARIANT | vcpu.async_pf.queued ≤ ASYNC_PF_PER_VCPU. |
| `entry_in_one_list` | INVARIANT | per-entry: at most one of {queue, done}. |
| `queued_eq_total_count` | INVARIANT | vcpu.async_pf.queued == #queue + #done. |
| `mutex_held_for_list_mut` | INVARIANT | per-list-mut: vcpu.async_pf.lock held. |
| `entry_freed_after_arch_ready` | INVARIANT | per-completion: kvm_arch_async_page_ready called pre-free. |

### Layer 2: TLA+

`virt/kvm/async_pf.tla`:
- Per-vCPU setup → wq → done → check_completion lifecycle.
- Properties:
  - `safety_no_double_free` — per-entry kmem_cache_free called at most once.
  - `safety_per_vcpu_cap` — per-vCPU queued ≤ MAX.
  - `liveness_eventual_completion` — per-setup ⟹ eventually entry in done ⟹ check_completion drains.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Apf::setup_async_pf` post: entry in queue; queued++; wq queued | `Apf::setup_async_pf` |
| `Apf::execute` post: entry in done; arch_async_page_present called | `Apf::execute` |
| `Apf::check_completion` post: done-list empty; queued = #queue | `Apf::check_completion` |
| `Apf::wakeup_all` post: both lists empty; queued = 0 | `Apf::wakeup_all` |

### Layer 4: Verus/Creusot functional

`Per-swapped-out-page guest fault → APF setup → wq fault-in → page present → vCPU resumes` semantic equivalence: per-async-PF removes vCPU stall on swap.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-async-pf.md` § Hardening.)

Async-PF-core-specific reinforcement:

- **Per-vCPU queued cap** — defense against per-vCPU unbounded entry alloc DoS.
- **Per-mutex serializes list mut** — defense against per-list torn-mutation.
- **Per-entry vcpu.task ref via get_task_mm** — defense against UAF post-mm-exit.
- **Per-wq cancel_work_sync at teardown** — defense against in-flight UAF.
- **Per-arch_async_page_ready called pre-free** — defense against arch missing notify on cancellation.
- **Per-mmput released after walk** — defense against per-mm leak.
- **Per-kmem_cache scoped to KVM module** — defense against per-cross-module use.
- **Per-queue_work after list_add** — defense against wq seeing entry not yet in list.
- **Per-async_pf.lock outside per-vCPU run** — defense against per-vmenter blocking on async-PF.
- **Per-execute uses get_user_pages for proper page-walk** — defense against per-fault-path skip causing repeat faults.

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide mitigations relied upon by the KVM async-PF subsystem:

- **PAX_USERCOPY** — bounds-checks any guest-physical → host copy that surfaces async-PF data (token, reason) back to userspace; rejects slab-boundary crossings.
- **PAX_KERNEXEC** — `kvm_async_pf_ops` and per-arch `kvm_arch_async_page_ready` callbacks live in RX/RO text; the work-item callback cannot be redirected at runtime.
- **PAX_RANDKSTACK** — per-entry stack-offset randomization on KVM_RUN reentry from an async-PF wake defeats stack-grooming via the async-PF resume path.
- **PAX_REFCOUNT** — saturating ref on `kvm->users_count`, `vcpu->refcount`, and the task/mm refs captured via `get_task_mm`; wq-vs-vCPU-exit races trap cleanly.
- **PAX_MEMORY_SANITIZE** — zeroes freed `struct kvm_async_pf` slab entries so a follow-on async-PF allocation cannot reuse residual guest token/data.
- **PAX_UDEREF** — strict user/kernel pointer separation when delivering page-ready notifications back into guest-accessible shared regions.
- **PAX_RAP / kCFI** — forward-edge CFI on the work-item callback and `arch_async_page_ready`, blocking JOP via a corrupted async_pf entry.
- **GRKERNSEC_HIDESYM** — `kvm_async_pf_wakeup_all`, `kvm_arch_async_page_present`, and the per-vCPU queue helpers hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — restricts dmesg so async-PF queue-full/oom traces do not leak per-VM allocation pressure to other tenants.

KVM-async-PF-specific reinforcement:

- **CAP_SYS_ADMIN on the KVM fd** — gates the entire async-PF feature; an unprivileged caller cannot enable async-PF on a foreign VM.
- **Page-ready injection bounded** — per-vCPU queue depth cap paired with PAX_REFCOUNT prevents async-PF entry-bomb DoS against a target vCPU.
- **mm pin via mmget + mmput around get_user_pages** — paired with PAX_REFCOUNT keeps the guest mm UAF-safe across the wq deferral.
- **cancel_work_sync at vCPU teardown** — paired with PAX_MEMORY_SANITIZE so any racing wq item observes zeroed state, not a half-freed entry.

Rationale: async-PF is one of the few KVM code paths that defers work onto a workqueue holding a guest-mm reference; combining PAX_REFCOUNT on the mm/task pin with PAX_MEMORY_SANITIZE on the slab eliminates the UAF and reuse-disclosure classes that have historically plagued this surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM x86 async-PF arch glue (covered in `x86-async-pf.md` Tier-3)
- Linux mm get_user_pages (covered separately)
- Workqueue (covered separately)
- Implementation code
