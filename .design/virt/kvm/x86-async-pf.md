# Tier-3: virt/kvm/async_pf.c — KVM async page-fault paravirt (NOT_PRESENT/READY notifications via #PF.0xff or interrupt vector)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - virt/kvm/async_pf.c
  - virt/kvm/async_pf.h
  - arch/x86/kvm/x86.c (async-pf-related sections; see grep "async_pf|kvm_pv_async_pf")
  - arch/x86/include/uapi/asm/kvm_para.h (KVM_FEATURE_ASYNC_PF / _ASYNC_PF_INT / _ASYNC_PF_VMEXIT)
-->

## Summary

KVM async-PF is a paravirt mechanism that lets guest tolerate KVM-side page-faults (host-page-not-resident → host swap-in needed) without the vCPU thread blocking on host-paging. When KVM detects that resolving a guest #PF requires host paging (host page in swap or not-yet-allocated), KVM sends guest a "not-present" #PF (token-encoded) → guest scheduler pauses the faulting task, KVM kicks off a host worker to do `get_user_pages_unlocked`, and on completion KVM sends a "ready" notification → guest scheduler resumes the task. This avoids serializing all vCPU work behind one slow host-paging operation.

This Tier-3 covers `virt/kvm/async_pf.c` (~241 lines) + arch-x86 integration in `arch/x86/kvm/x86.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_async_pf` | per-async-PF event | `kernel::kvm::async_pf::AsyncPf` |
| `kvm_async_pf_init(...)` / `kvm_async_pf_deinit()` | subsystem init | `AsyncPfSubsys::init` / `_deinit` |
| `kvm_async_pf_vcpu_init(vcpu)` | per-vCPU init | `Vcpu::async_pf_init` |
| `kvm_setup_async_pf(vcpu, cr2_or_gpa, gva, &arch)` | begin async-PF | `Vcpu::setup_async_pf` |
| `kvm_async_pf_wakeup_all(vcpu)` | wake all on disable | `Vcpu::async_pf_wakeup_all` |
| `kvm_check_async_pf_completion(vcpu)` | per-vmenter ready-notify check | `Vcpu::check_async_pf_completion` |
| `kvm_clear_async_pf_completion_queue(vcpu)` | clear queue | `Vcpu::clear_async_pf_completion_queue` |
| `async_pf_execute(work)` | host-worker fault-resolve | `AsyncPf::execute` |
| `kvm_arch_async_page_present(...)` | per-arch ready-notify | `arch::async_page_present` |
| `kvm_arch_async_page_present_queued(...)` | per-arch queue ready | `arch::async_page_present_queued` |
| `kvm_arch_async_page_not_present(...)` | per-arch not-present-notify | `arch::async_page_not_present` |
| `kvm_async_pf_hash_reset(vcpu)` (x86.c) | per-vCPU hash reset | `Vcpu::async_pf_hash_reset` |
| `kvm_pv_enable_async_pf(vcpu, data)` (x86.c) | guest WRMSR enable handler | `Vcpu::pv_enable_async_pf` |
| `kvm_pv_enable_async_pf_int(vcpu, data)` (x86.c) | enable async-PF-INT mode | `Vcpu::pv_enable_async_pf_int` |
| `MSR_KVM_ASYNC_PF_EN` (0x4b564d02) | guest enable + control-page-addr MSR | UAPI |
| `MSR_KVM_ASYNC_PF_INT` (0x4b564d06) | async-PF-INT vector MSR | UAPI |
| `MSR_KVM_ASYNC_PF_ACK` (0x4b564d07) | guest-ack MSR | UAPI |
| `kvm_async_pf_task_wait_schedule(...)` (guest-side) | guest scheduler integration | (guest-only; out-of-scope) |

## Compatibility contract

REQ-1: Guest enable via WRMSR(MSR_KVM_ASYNC_PF_EN):
- data bit 0: enable.
- data bit 1: send-always (don't gate on guest-irq-enabled).
- data bits[63:6] (4KiB-aligned): control-page guest-PA.
- Control-page contains:
  - 32-bit token (per-event identifier).
  - 32-bit reason (1=not-present-pending, 2=ready).

REQ-2: Async-PF-INT mode (newer):
- Instead of #PF.0xff vector (subject to guest #PF intercept-on-host-paging-fault), use dedicated interrupt vector via MSR_KVM_ASYNC_PF_INT.
- Reduces collisions with guest's normal #PF handling.

REQ-3: Per-vCPU async-PF queue + hash:
- `vcpu.async_pf.queue`: pending events (those for which host-fault not yet complete).
- `vcpu.async_pf.done`: completed events (ready to send-ready-notify to guest).
- `vcpu.async_pf.gfns[ASYNC_PF_PER_VCPU=64]`: hash for fast token → gfn lookup; defends against duplicate.

REQ-4: not-present-flow:
1. Guest accesses gva → vmexit EPT-violation (TDP) or #PF (shadow) → KVM faults page.
2. KVM determines page is not-resident on host (gfn → user HVA → !page_present).
3. `kvm_setup_async_pf(vcpu, gpa, gva, &arch)`:
   - Allocate `kvm_async_pf` work; populate (vcpu, cr2_or_gpa, addr, mm, arch).
   - Add to vcpu.async_pf.queue.
   - Schedule async_pf_execute on `kvm_async_pf_wq` workqueue.
   - Return: 0 (async path taken) — KVM will inject not-present notify, resume guest.
4. `kvm_arch_async_page_not_present`:
   - Compose token (auto-incrementing per-vCPU).
   - Write control-page: token + reason=NOT_PRESENT.
   - Inject #PF with cr2 = special-token-encoded-gva, error-code = special bit.
   - Or (async-PF-INT mode): inject async-pf-INT vector with token in control-page.
5. Guest #PF handler / async-pf-INT handler:
   - Read control-page; recognize token.
   - Block faulting task; schedule another task on this vCPU.

REQ-5: ready-flow:
1. async_pf_execute (workqueue):
   - get_user_pages_unlocked(work.mm, work.addr, ...) → faults page in.
   - On completion: list_move(&work, &vcpu.async_pf.done).
   - kvm_make_request(KVM_REQ_APF_READY, vcpu).
   - kvm_vcpu_kick(vcpu).
2. On next vmenter: `kvm_check_async_pf_completion(vcpu)`:
   - For each work in vcpu.async_pf.done:
     - `kvm_arch_async_page_present(vcpu, work)`:
       - Write control-page: token + reason=READY.
       - Inject #PF or async-pf-INT.
     - Free work.
3. Guest #PF handler:
   - Read control-page; reason=READY → unblock task with token.

REQ-6: KVM_FEATURE_ASYNC_PF CPUID bit:
- Advertised in CPUID 0x40000001 EAX bit KVM_FEATURE_ASYNC_PF=4.
- KVM_FEATURE_ASYNC_PF_INT=14 if vector-mode supported.
- KVM_FEATURE_ASYNC_PF_VMEXIT=10 if guest can request VMexit at HLT during async-PF (lets host re-schedule).

REQ-7: Guest disable via WRMSR(MSR_KVM_ASYNC_PF_EN, 0):
- Clear pv_async_pf_enabled.
- `kvm_clear_async_pf_completion_queue` + `kvm_async_pf_wakeup_all`.

REQ-8: Live migration:
- Per-vCPU async_pf.queue not migrated (in-flight host-paging won't carry across).
- Per-vCPU MSRs (EN, INT, ACK) migrated.
- Post-migrate: guest may re-fault; host resolves on new node.

REQ-9: Per-vCPU MSR_KVM_ASYNC_PF_ACK:
- Guest writes after handling each notify; KVM uses to track guest-completed acknowledgments.

REQ-10: KVM_REQ_APF_HALT:
- Per-vCPU request to halt vCPU until guest-acknowledged async-pf in-flight; defense against guest spinning on faulted page.

REQ-11: ASYNC_PF_PER_VCPU cap (64):
- Bounds per-vCPU concurrent in-flight async-PF; defense against guest-induced unbounded host-paging requests.

REQ-12: kvm_async_pf_wq:
- Per-VM unbound workqueue for resolution work.
- Configurable via module param.

## Acceptance Criteria

- [ ] AC-1: KVM_FEATURE_ASYNC_PF advertised: guest CPUID 0x40000001 EAX bit set.
- [ ] AC-2: Memory-overcommit test: VM with 4GB ballooned + memory-stress; async-PF events visible via /sys/kernel/debug/kvm/.
- [ ] AC-3: Guest pid in async-PF: faulting task suspended; other tasks on same vCPU continue.
- [ ] AC-4: ready-notify: faulted page resolved on host; guest task unblocked + resumes.
- [ ] AC-5: Async-PF-INT mode: guest enables MSR_KVM_ASYNC_PF_INT with vector V; not-present arrives as INT V (not #PF).
- [ ] AC-6: ASYNC_PF_PER_VCPU bound: 100 simultaneous faults on different gfns; only 64 in-flight; rest fall-back to sync paging.
- [ ] AC-7: Disable test: guest WRMSR(MSR_KVM_ASYNC_PF_EN, 0); subsequent host-paging causes sync vCPU-block (no async-PF inject).
- [ ] AC-8: kvm-unit-tests `async_pf` test passes.
- [ ] AC-9: Spec-violation defense: control-page not 4KiB-aligned at WRMSR → no enable; KVM logs.

## Architecture

`AsyncPf` per-event:

```
struct AsyncPf {
  link: ListNode,
  vcpu: KArc<Vcpu>,
  mm: KArc<MmStruct>,
  addr: u64,                                  // host-VA of guest page
  cr2_or_gpa: u64,                            // guest cr2 / GPA at fault
  arch: ArchAsyncPfData,                      // arch-specific
  notpresent_injected: bool,
  work: Work,                                 // workqueue item
  page: Option<KArc<Page>>,
}
```

Per-vCPU additions:

```
struct VcpuArch {
  ...
  apf: VcpuAsyncPf,
}

struct VcpuAsyncPf {
  enabled: bool,
  send_always: bool,
  apf_msr_val: u64,                           // MSR_KVM_ASYNC_PF_EN
  apf_int_msr_val: u64,                       // MSR_KVM_ASYNC_PF_INT
  control_page_pa: u64,                       // 4KiB-aligned guest-PA
  vec: u8,                                    // async-pf-INT vector
  gfns: [u64; ASYNC_PF_PER_VCPU],            // hash for active tokens
  queue: ListHead,                            // in-flight events
  done: ListHead,                              // completed events
  queued: u32,
  notpresent_injected: AtomicBool,
}
```

`Vcpu::setup_async_pf(vcpu, gpa, gva, &arch)`:
1. Allocate work := KArc<AsyncPf>; populate.
2. Lock vcpu.async_pf.queue.
3. If queued >= ASYNC_PF_PER_VCPU: return false (sync path).
4. list_add_tail(&work.link, &queue); queued++.
5. Hash insert work-gfn for dedup.
6. queue_work(kvm_async_pf_wq, &work.work).
7. Inject not-present-notify via `kvm_arch_async_page_not_present`.
8. Return true.

`AsyncPf::execute(work)` (workqueue):
1. mmap_read_lock(work.mm).
2. get_user_pages_remote(work.mm, work.addr, 1, FOLL_WRITE, &page, ...).
3. mmap_read_unlock.
4. work.page = page.
5. Lock vcpu.async_pf.
6. list_move(&work.link, &done).
7. kvm_make_request(KVM_REQ_APF_READY, work.vcpu).
8. kvm_vcpu_kick(work.vcpu).

`Vcpu::check_async_pf_completion(vcpu)`:
1. While !vcpu.async_pf.done.empty:
   - work := list_first(&done).
   - list_del(&work.link).
   - `kvm_arch_async_page_present(vcpu, work)`:
     - Write control-page: token=work.token, reason=READY.
     - Inject #PF or async-pf-INT.
   - kvm_async_pf_hash_remove(vcpu, work.gfn).
   - Drop work refcount.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `async_pf_per_vcpu_bounded` | INVARIANT | per-vCPU queued ≤ 64; defense against unbounded growth. |
| `control_page_aligned` | INVARIANT | per-vCPU control-page PA 4KiB-aligned; defense against straddling page boundary on write. |
| `token_unique_per_in_flight` | INVARIANT | per-vCPU token unique across in-flight events; defense against guest token-collision confusing handler. |
| `gfn_hash_dedup` | INVARIANT | per-vCPU per-gfn at most one async-PF in queue at a time. |

### Layer 2: TLA+

`virt/kvm/async_pf_lifecycle.tla`:
- States: NotInflight, NotPresentInjected, Resolving, ReadyInjected, Acked.
- Transitions per setup → workqueue resolution → ready-inject → guest-ack.
- Properties:
  - `safety_no_ready_before_resolved` — ReadyInjected only after Resolving completes.
  - `safety_token_match_pair` — NotPresentInjected.token == ReadyInjected.token for each pair.
  - `liveness_resolving_eventually_ready` — assuming workqueue runs, every Resolving eventually ReadyInjected.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::setup_async_pf` post: work in queue; workqueue scheduled; not-present-inject pending | `Vcpu::setup_async_pf` |
| `AsyncPf::execute` post: work moved to done list; KVM_REQ_APF_READY set | `AsyncPf::execute` |
| `Vcpu::check_async_pf_completion` post: all done-list events ready-injected; queue cleared | `Vcpu::check_async_pf_completion` |
| Per-vCPU per-gfn dedup hash maintained across setup/execute | `Vcpu::async_pf_hash_*` |

### Layer 4: Verus/Creusot functional

`Per-async-pf-event: setup → not-present-inject → host-fault → ready-inject → guest-ack` semantic equivalence: per-event the guest sees exactly one not-present-notify followed by exactly one ready-notify with matching token.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

async-PF-specific reinforcement:

- **Per-vCPU concurrency cap (64)** — defense against guest-induced unbounded host-paging causing kernel-mem exhaustion.
- **Per-gfn dedup hash** — defense against guest re-fault same gfn before resolution causing double-not-present-inject.
- **Control-page PA validated** at WRMSR — defense against guest pointing to host kernel memory.
- **Per-token uniqueness** — defense against guest token-collision causing wrong-task-unblock.
- **send_always bit gating** — defense against not-present-inject when guest IRQ disabled (would lose notify).
- **kvm_async_pf_wq unbound + WQ_MEM_RECLAIM** — defense against deadlock with memory reclaim path.
- **work.mm KArc-held during execute** — defense against mm-free during get_user_pages.
- **Per-vCPU async-pf-disable wake_up_all** — defense against task-stuck waiting on disabled async-pf.
- **KVM_REQ_APF_HALT for spin-defense** — defense against guest spinning on faulted page consuming vCPU.
- **MSR_KVM_ASYNC_PF_INT vector validated** in [32, 255] range — defense against vector clash with reserved IDT entries.
- **Per-vmenter check_async_pf_completion** — defense against ready-notify lost on rapid vmexit/vmenter cycle.
- **Live-migrate clears in-flight queue** — defense against post-migrate stale work referencing prior-host mm.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- TDP MMU page-fault-resolve (covered in `x86-mmu-tdp.md` Tier-3)
- Shadow MMU page-fault-resolve (covered in `x86-mmu-shadow.md` Tier-3)
- VMX/SVM intercept setup (covered in `x86-vmx.md` / `x86-svm.md` Tier-3s)
- Guest-side scheduler integration (guest-only; out-of-scope)
- Hyper-V Stuffer/Doorbell async-PF (different paravirt; out-of-scope)
- Implementation code
