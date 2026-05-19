# Tier-3: mm/oom_kill.c — Out-of-Memory killer

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/oom_kill.c (~1257 lines)
  - include/linux/oom.h
  - include/uapi/linux/sysctl.h (vm.panic_on_oom, vm.oom_kill_allocating_task, ...)
-->

## Summary

When the kernel cannot reclaim enough memory to satisfy an allocation, the **OOM killer** picks a victim process and kills it to free pages. Per-OOM context: `struct oom_control` (gfp_mask, nodemask, totalpages, victim, ...). Per-victim selected via `oom_badness(p, oc, totalpages)` heuristic = (RSS + swap + per-shmem-pages) * 1000 / totalpages + oom_score_adj. Per-`__oom_reap_task_mm` reaps anonymous + shmem pages of victim's mm before SIGKILL completion. Per-memcg-OOM has separate selector path. Per-`vm.panic_on_oom` policy: 1 = panic, 2 = panic-always, 0 = kill. Critical for: surviving memory pressure, container-OOM enforcement, dependable kill-policy.

This Tier-3 covers `mm/oom_kill.c` (~1257 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct oom_control` | per-OOM context | `OomControl` |
| `out_of_memory()` | per-trigger | `Oom::out_of_memory` |
| `oom_badness()` | per-task score | `Oom::badness` |
| `select_bad_process()` | per-pick victim | `Oom::select_bad_process` |
| `oom_kill_process()` | per-kill | `Oom::kill_process` |
| `__oom_kill_process()` | per-task SIGKILL | `Oom::kill_process_inner` |
| `oom_reap_task()` | per-reap mm | `Oom::reap_task` |
| `__oom_reap_task_mm()` | per-mm-walk reap | `Oom::reap_task_mm` |
| `oom_reaper()` | per-kthread | `Oom::reaper_kthread` |
| `wake_oom_reaper()` | per-enqueue | `Oom::wake_reaper` |
| `oom_evaluate_task()` | per-iter eval | `Oom::evaluate_task` |
| `mark_oom_victim()` | per-mark | `Oom::mark_victim` |
| `oom_killer_disable()` / `_enable()` | per-suspend (kexec) | `Oom::disable` / `enable` |
| `dump_header()` | per-OOM klog dump | `Oom::dump_header` |
| `dump_tasks()` | per-iter task-dump | `Oom::dump_tasks` |
| `mem_cgroup_oom_synchronize()` | per-memcg sync | `Oom::memcg_synchronize` |
| `oom_panic_on_oom_used` | per-flag | shared |

## Compatibility contract

REQ-1: struct oom_control:
- zonelist: per-allocation zonelist.
- nodemask: per-allocation node-mask.
- gfp_mask: per-allocation gfp.
- order: per-allocation order.
- totalpages: per-allocation universe (system / cgroup).
- chosen: best victim (task_struct).
- chosen_points: best score.
- memcg: per-memcg-OOM context (NULL = global).
- constraint: CONSTRAINT_NONE / _CPUSET / _MEMORY_POLICY / _MEMCG.

REQ-2: out_of_memory(oc):
- /* Determine constraint */
- oc.constraint = constrained_alloc(oc).
- /* Evaluate panic-on-oom policy */
- check_panic_on_oom(oc) — may panic.
- /* Per-vm.oom_kill_allocating_task */
- if sysctl_oom_kill_allocating_task ∧ current.exit_state == 0:
  - oom_kill_process(oc, current).
  - return true.
- /* Otherwise pick worst */
- oc.chosen = NULL; oc.chosen_points = 0.
- select_bad_process(oc).
- if !oc.chosen ∨ oc.chosen == (void *)-1UL:
  - if oc.constraint != CONSTRAINT_MEMCG: panic("Out of memory and no killable processes").
  - return false.
- oom_kill_process(oc, oc.chosen).
- return true.

REQ-3: oom_badness(p, oc, totalpages):
- if oom_unkillable_task(p): return LONG_MIN.
- if p.flags & PF_EXITING: return LONG_MIN (already dying).
- /* Score = RSS + swap + shared-shmem * 0.5 */
- points = get_mm_rss(p.mm) + get_mm_counter(p.mm, MM_SWAPENTS).
- /* Adjust by oom_score_adj */
- adj = p.signal.oom_score_adj.
- /* Per-OOM_SCORE_ADJ_MIN: never kill */
- if adj == OOM_SCORE_ADJ_MIN: return LONG_MIN.
- /* Final score: scale */
- return points + (adj * totalpages / OOM_SCORE_ADJ_MAX).

REQ-4: select_bad_process(oc):
- /* Per-memcg or all-tasks */
- if oc.memcg:
  - mem_cgroup_scan_tasks(oc.memcg, oom_evaluate_task, oc).
- else:
  - rcu_read_lock.
  - for_each_process(p):
    - oom_evaluate_task(p, oc).
  - rcu_read_unlock.

REQ-5: oom_evaluate_task(task, oc):
- if oom_unkillable_task(task): return 0.
- /* Per-thread group leader */
- if task.signal.oom_score_adj == OOM_SCORE_ADJ_MIN: return 0.
- points = oom_badness(task, oc, oc.totalpages).
- if points > oc.chosen_points:
  - if oc.chosen: put_task_struct(oc.chosen).
  - get_task_struct(task).
  - oc.chosen = task; oc.chosen_points = points.
- return 0.

REQ-6: oom_kill_process(oc, victim):
- /* Iterate tasks of victim's process group / OOM-domain */
- task = find_lock_task_mm(victim).
- if !task:
  - put_task_struct(victim).
  - return.
- /* Mark all sharing mm */
- mm = task.mm.
- mark_oom_victim(victim).
- /* Wake reaper if !MMF_OOM_VICTIM */
- if mm.flags & MMF_OOM_VICTIM: skip-redundant.
- else: wake_oom_reaper(victim).
- /* SIGKILL all tasks sharing mm */
- for_each_task_using_mm(mm, p):
  - send_sig(SIGKILL, p, 1).
- /* Coredump-if-not-suppressed */
- task_unlock(task).

REQ-7: oom_reap_task / __oom_reap_task_mm:
- /* Reap anonymous + shmem in mm before kernel handles SIGKILL */
- if !mmap_read_trylock(mm): return false.
- /* Walk VMAs */
- for vma in mm.mm_vmas():
  - if !is_vm_hugetlb_page(vma) ∧ !(vma.vm_flags & (VM_PFNMAP | VM_HUGETLB)):
    - tlb_gather_mmu, unmap_page_range, tlb_finish_mmu.
- mmap_read_unlock(mm).
- /* Mark MMF_OOM_REAPED */
- set_bit(MMF_OOM_REAPED, &mm.flags).

REQ-8: oom_reaper kthread:
- Loops on oom_reaper_list.
- For each task: __oom_reap_task_mm.
- Yields after time-bound.

REQ-9: panic-on-oom:
- 0 = no panic (default).
- 1 = panic if no node has memory (UMA: panic; NUMA: panic only when constraint==NONE).
- 2 = panic always.

REQ-10: oom_killer_disable / enable:
- During kexec / hibernation: disable.
- Pending OOM kills allowed to finish (timeout).

REQ-11: oom_unkillable_task(p):
- p == current ∧ in_atomic.
- p == init (pid 1).
- p.flags & PF_KTHREAD.

REQ-12: Per-CONSTRAINT_*:
- NONE: global.
- CPUSET: cpuset-restricted.
- MEMORY_POLICY: mempolicy-restricted.
- MEMCG: memcg-OOM.

REQ-13: Per-mark_oom_victim:
- task.signal.flags |= SIGNAL_OOM_VICTIM.
- Allows TIF_MEMDIE to grant emergency-page-allocation budget.

REQ-14: Per-MMF_OOM_VICTIM / MMF_OOM_REAPED:
- mm flags tracking OOM lifecycle.

REQ-15: Per-cgroup-OOM:
- mem_cgroup_oom_synchronize at userspace-context.
- Per-OOM-events count + delivery.

REQ-16: Per-dump_header:
- Print "Out of memory: Killed process X" + per-task scores via dump_tasks.

## Acceptance Criteria

- [ ] AC-1: Allocation fails after reclaim: out_of_memory called.
- [ ] AC-2: select_bad_process picks task with highest oom_badness.
- [ ] AC-3: oom_score_adj == -1000: never killed.
- [ ] AC-4: oom_kill_process: SIGKILL to all tasks sharing victim's mm.
- [ ] AC-5: oom_reaper reaps mm anonymous pages pre-exit.
- [ ] AC-6: vm.panic_on_oom = 2: panics on OOM regardless of constraint.
- [ ] AC-7: vm.oom_kill_allocating_task = 1: kills current.
- [ ] AC-8: cgroup-OOM: per-memcg select; not global.
- [ ] AC-9: kthread / init protected from kill.
- [ ] AC-10: dump_header logs "Out of memory" with task list.
- [ ] AC-11: oom_killer_disable / enable: disabled — pending kills wait for timeout.

## Architecture

```
struct OomControl {
  zonelist: *Zonelist,
  nodemask: NodeMask,
  gfp_mask: u32,
  order: i32,
  totalpages: u64,
  chosen: Option<*TaskStruct>,
  chosen_points: i64,
  memcg: Option<*MemCgroup>,
  constraint: u8,                    // CONSTRAINT_*
}
```

`Oom::out_of_memory(oc) -> bool`:
1. /* Constraint determination */
2. oc.constraint = constrained_alloc(oc).
3. /* Panic policies */
4. check_panic_on_oom(oc).
5. /* Allocating-task kill */
6. if sysctl_oom_kill_allocating_task ∧ !oom_unkillable_task(current) ∧ current.exit_state == 0 ∧ current.signal.oom_score_adj != OOM_SCORE_ADJ_MIN:
   - get_task_struct(current).
   - Oom::kill_process(oc, current).
   - return true.
7. /* Pick worst */
8. oc.chosen = None; oc.chosen_points = 0.
9. Oom::select_bad_process(oc).
10. if !oc.chosen:
    - if oc.constraint != CONSTRAINT_MEMCG: panic("Out of memory and no killable").
    - return false.
11. Oom::kill_process(oc, oc.chosen).
12. return true.

`Oom::badness(p, totalpages) -> i64`:
1. if oom_unkillable_task(p): return i64::MIN.
2. p_lock = task_lock(p).
3. mm = p.mm.
4. if !mm:
   - task_unlock(p_lock).
   - return i64::MIN.
5. adj = p.signal.oom_score_adj.
6. if adj == OOM_SCORE_ADJ_MIN ∨ p.flags & PF_EXITING:
   - task_unlock(p_lock).
   - return i64::MIN.
7. points = get_mm_rss(mm) + get_mm_counter(mm, MM_SWAPENTS) + mm_pgtables_bytes(mm) / PAGE_SIZE.
8. task_unlock(p_lock).
9. /* Apply adj */
10. points += adj * totalpages / OOM_SCORE_ADJ_MAX.
11. return points.

`Oom::select_bad_process(oc)`:
1. if oc.memcg:
   - mem_cgroup_scan_tasks(oc.memcg, |p, oc| Oom::evaluate_task(p, oc), oc).
2. else:
   - rcu_read_lock().
   - for_each_process(p): Oom::evaluate_task(p, oc).
   - rcu_read_unlock().

`Oom::evaluate_task(task, oc) -> i32`:
1. if oom_unkillable_task(task): return 0.
2. points = Oom::badness(task, oc.totalpages).
3. if points == i64::MIN: return 0.
4. if points > oc.chosen_points:
   - /* Atomically swap chosen */
   - if oc.chosen.is_some(): put_task_struct(oc.chosen.unwrap()).
   - if !get_task_struct(task): return 0.
   - oc.chosen = Some(task).
   - oc.chosen_points = points.
5. return 0.

`Oom::kill_process(oc, victim)`:
1. p = find_lock_task_mm(victim).
2. if !p:
   - put_task_struct(victim).
   - return.
3. mm = p.mm.
4. /* Mark + reap-eligible */
5. Oom::mark_victim(p).
6. atomic_inc(&mm.mm_users).
7. /* Wake reaper */
8. if !test_bit(MMF_UNSTABLE, &mm.flags):
   - Oom::wake_reaper(p).
9. /* SIGKILL all tasks sharing mm */
10. for_each_thread_using_mm(mm, t):
    - send_sig_mceerr(SIGKILL, t, 0, 0) or do_send_sig_info(SIGKILL, ...).
11. task_unlock(p).
12. mmput(mm).
13. /* dump_header */
14. dump_header(oc, p).

`Oom::reap_task_mm(task, mm) -> bool`:
1. if !mmap_read_trylock(mm): return false.
2. /* Lock acquired; reap */
3. tlb = tlb_gather_mmu(mm).
4. for vma in mm.mm_vmas():
   - if (vma.vm_flags & VM_LOCKED): continue.
   - if vma.vm_flags & (VM_PFNMAP | VM_HUGETLB): continue.
   - unmap_page_range(&tlb, vma, vma.vm_start, vma.vm_end, NULL).
5. tlb_finish_mmu(&tlb).
6. mmap_read_unlock(mm).
7. set_bit(MMF_OOM_REAPED, &mm.flags).
8. return true.

`Oom::mark_victim(task)`:
1. spin_lock_irq(&task.sighand.siglock).
2. task.signal.flags |= SIGNAL_OOM_VICTIM.
3. tsk_set_oom_origin(task).
4. spin_unlock_irq.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chosen_refcount_balanced` | INVARIANT | per-out_of_memory: chosen task ref get'd, put'd before return. |
| `unkillable_excluded` | INVARIANT | per-evaluate_task: unkillable returns 0. |
| `oom_score_adj_min_excluded` | INVARIANT | per-badness: OOM_SCORE_ADJ_MIN ⟹ i64::MIN. |
| `mark_victim_under_siglock` | INVARIANT | per-mark_victim: siglock held. |
| `reap_task_mm_holds_mmap_read_lock` | INVARIANT | per-reap_task_mm: mmap_read_trylock held during reap. |
| `panic_on_oom_obeyed` | INVARIANT | per-vm.panic_on_oom: per-policy panic vs kill. |

### Layer 2: TLA+

`mm/oom-kill.tla`:
- Per-OOM-trigger + per-select + per-kill + per-reap + per-completion.
- Properties:
  - `safety_init_pid1_never_killed` — per-OOM: init process never selected.
  - `safety_kthread_never_killed` — per-OOM: PF_KTHREAD never selected.
  - `safety_oom_score_adj_min_protects` — per-OOM: OOM_SCORE_ADJ_MIN never killed.
  - `safety_panic_on_oom_2_panics` — per-vm.panic_on_oom == 2: panic-not-kill.
  - `liveness_per_OOM_eventually_kills_or_returns` — per-out_of_memory: terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Oom::out_of_memory` post: ret ⟹ chosen != NULL ∨ panic | `Oom::out_of_memory` |
| `Oom::badness` post: returns i64 score ∨ MIN | `Oom::badness` |
| `Oom::select_bad_process` post: oc.chosen has highest score | `Oom::select_bad_process` |
| `Oom::kill_process` post: SIGKILL sent to mm-sharers | `Oom::kill_process` |
| `Oom::reap_task_mm` post: anon+shmem unmapped; MMF_OOM_REAPED set | `Oom::reap_task_mm` |
| `Oom::mark_victim` post: SIGNAL_OOM_VICTIM set | `Oom::mark_victim` |

### Layer 4: Verus/Creusot functional

`Per-OOM trigger → constrained_alloc → select_bad_process → kill_process (SIGKILL all mm-sharers) → wake_reaper → reap_task_mm` semantic equivalence: per-Documentation/admin-guide/sysctl/vm.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

OOM-kill reinforcement:

- **Per-init / kthread excluded** — defense against per-init-kill catastrophe.
- **Per-OOM_SCORE_ADJ_MIN never-kill** — defense against per-systemd-OOM.
- **Per-task ref-count strict get/put** — defense against per-victim UAF.
- **Per-mmap_read_trylock for reaper** — defense against per-reap-while-mutating mm.
- **Per-MMF_OOM_REAPED idempotent** — defense against per-double-reap.
- **Per-SIGNAL_OOM_VICTIM emergency-page budget bounded** — defense against per-OOM-runaway.
- **Per-vm.panic_on_oom honored** — defense against per-policy-violation.
- **Per-cgroup-OOM scoped to memcg** — defense against per-cross-cgroup kill.
- **Per-constraint determination strict (cpuset, mempolicy, memcg)** — defense against per-wrong-domain kill.
- **Per-oom_killer_disable for kexec/hibernate** — defense against per-mid-kexec OOM.
- **Per-dump_header WARN-rate-limited** — defense against per-log-flood.
- **Per-OOM-victim coredump/core-pattern bounded** — defense against per-coredump-flood.
- **Per-oom_score_adj_min CAP_SYS_RESOURCE for-decrease** — defense against unprivileged immortal-task.

## Grsecurity/PaX-style Reinforcement

Baseline hardening that applies to the out-of-memory killer:

- **PAX_USERCOPY** — OOM dumps are emitted via `printk` and reaped by dmesg policy; no `copy_to_user` from oom_kill.
- **PAX_KERNEXEC** — `oom_evaluate_task`, `oom_kill_process`, `__oom_reap_task_mm` reside in `.rodata`.
- **PAX_RANDKSTACK** — `out_of_memory` entered with randomized kernel stack offset.
- **PAX_REFCOUNT** — `task_struct->usage` and `mm_struct->mm_users` use saturating refcount_t across `get_task_struct`/`mmget_not_zero` in the OOM reaper.
- **PAX_MEMORY_SANITIZE** — reaped MM's user pages enter the allocator via `unmap_page_range` + sanitize before reuse.
- **PAX_UDEREF** — OOM never dereferences victim userspace; it tears down the MM via kernel PTE walkers only.
- **PAX_RAP / kCFI** — OOM notifier list (`oom_notify_list`) callbacks dispatched through type-checked vtables.
- **GRKERNSEC_HIDESYM** — victim address-space layout (VMA addresses, RSS breakdown) redacted from non-root dmesg output.
- **GRKERNSEC_DMESG** — `dump_header` rate-limited and gated behind `dmesg_restrict` so OOM events do not become a side-channel.

OOM-killer-specific reinforcement:

- **OOM_SCORE_ADJ_MIN never-kill** — a task with `oom_score_adj == OOM_SCORE_ADJ_MIN` (-1000) is excluded from victim selection unconditionally; the OOM killer cannot promote it back into the candidate set.
- **CAP_SYS_RESOURCE strict for adj decrease** — decreasing `oom_score_adj` (making a task more immortal) requires `CAP_SYS_RESOURCE`; an unprivileged caller cannot create an unkillable task.
- **`panic_on_oom` honored** — when sysctl `vm.panic_on_oom=1` (or `=2` for constrained OOM) the kernel panics rather than silently killing; a configured fail-stop policy is not bypassable from the OOM path.
- **Cgroup-OOM scoped to memcg** — a cgroup-internal OOM cannot escape to kill processes outside the offending memcg; cross-cgroup spillover is refused.
- **Constraint determination strict** — `oom_constraint` distinguishes `MEMCG`, `CPUSET`, `MEMORY_POLICY`, `NONE`; victim selection happens within the constraining domain, not the global task list.
- **`oom_killer_disable` for kexec / hibernate** — OOM is disabled across the critical kexec/hibernate window so a mid-handover OOM cannot corrupt the in-flight image.

Rationale: the OOM killer is the highest-privilege "kill arbitrary process" primitive in the kernel; the above ensure it kills only victims selected within the configured constraint domain, honors administrator fail-stop policy, and cannot be steered by an unprivileged caller into killing a privileged peer.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/page_alloc.c reclaim (covered in `page-allocator.md` Tier-3)
- mm/vmscan.c LRU reclaim (covered in `reclaim.md` Tier-3)
- mm/memcontrol.c memcg (covered in `memcg.md` Tier-3)
- kernel/cgroup/freezer.c (covered separately if expanded)
- panic / kexec (covered separately if expanded)
- Implementation code
