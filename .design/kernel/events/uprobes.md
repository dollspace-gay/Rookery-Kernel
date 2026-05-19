# Tier-3: kernel/events/uprobes.c — Userspace probes (uprobes)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/perf/00-overview.md
upstream-paths:
  - kernel/events/uprobes.c (~2914 lines)
  - include/linux/uprobes.h
  - include/uapi/linux/uprobes.h (UPROBE_HANDLER_REMOVE, UPROBE_HANDLER_IGNORE)
  - arch/x86/kernel/uprobes.c (arch-specific glue)
-->

## Summary

A **uprobe** is a userspace breakpoint installed at a `(struct inode *, loff_t offset)` location in an executable backing file. When any process maps that inode and the probed instruction is fetched, an `int3` (or arch-equivalent UPROBE_SWBP_INSN) traps the thread; the kernel then runs a chain of `struct uprobe_consumer` callbacks and emulates the original instruction out of line in a per-`mm` **XOL** (execute-out-of-line) page so the probe is transparent to the workload. Per-uprobe identity = `(inode, offset)` keyed in a system-wide rbtree (`uprobes_treelock` + `uprobes_seqcount`). Per-`struct uprobe` = refcounted (`refcount_t ref`) + RCU-tasks-trace freed + consumer-list under `register_rwsem` / `consumer_rwsem`. Per-XOL = `struct xol_area` (anonymous executable special-mapping, `VM_EXEC|VM_DONTCOPY|VM_IO|VM_SEALED_SYSMAP`, bitmap of `UINSNS_PER_PAGE` slots, wait-queue for slot exhaustion). Per-uretprobe = stack-hijack technique: replace return-address on user stack with `uretprobe_trampoline` (slot 0 of XOL page), record original RA in a per-task `struct return_instance` stack threaded through `utask->return_instances`. Critical for: perf-event uprobe (`perf_event_attr.type == PERF_TYPE_UPROBE`), tracefs `trace_uprobe`, BPF uprobe (`BPF_PROG_TYPE_KPROBE` attached via `bpf_uprobe_multi_link`), SDT/USDT markers (with `ref_ctr_offset` for semaphore-style activation counter).

This Tier-3 covers `kernel/events/uprobes.c` (~2914 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct uprobe` | per-uprobe (inode+offset) | `Uprobe` |
| `struct uprobe_consumer` | per-consumer (handler/ret_handler/filter) | `UprobeConsumer` |
| `struct uprobe_task` (utask) | per-task uprobe singlestep state | `UprobeTask` |
| `struct xol_area` | per-mm XOL anonymous mapping | `XolArea` |
| `struct return_instance` | per-uretprobe stack frame | `ReturnInstance` |
| `struct hprobe` | hybrid-lifetime uprobe ref (SRCU/refcount) | `Hprobe` |
| `struct return_consumer` | per-consumer cookie inside return_instance | `ReturnConsumer` |
| `struct delayed_uprobe` | per-mm pending ref_ctr update | `DelayedUprobe` |
| `struct arch_uprobe` | per-arch insn + ixol copy | `ArchUprobe` (arch crate) |
| `uprobe_register()` | per-(inode,offset) install | `Uprobes::register` |
| `uprobe_unregister_nosync()` | per-consumer remove (no sync) | `Uprobes::unregister_nosync` |
| `uprobe_unregister_sync()` | per-RCU-tasks-trace + SRCU sync | `Uprobes::unregister_sync` |
| `uprobe_apply()` | per-consumer enable/disable | `Uprobes::apply` |
| `install_breakpoint()` | per-vma write int3 | `Uprobes::install_breakpoint` |
| `remove_breakpoint()` | per-vma restore orig insn | `Uprobes::remove_breakpoint` |
| `uprobe_write_opcode()` / `uprobe_write()` | per-page write under mm_lock | `Uprobes::write_opcode` |
| `set_swbp()` / `set_orig_insn()` | per-arch weak write | `ArchUprobe::set_swbp` / `set_orig_insn` |
| `uprobe_mmap()` / `uprobe_munmap()` | per-vma mmap/munmap hook | `Uprobes::on_mmap` / `on_munmap` |
| `uprobe_dup_mmap()` | per-fork carry MMF_HAS_UPROBES | `Uprobes::dup_mmap` |
| `uprobe_copy_process()` | per-fork copy utask + xol | `Uprobes::copy_process` |
| `handle_swbp()` | per-trap dispatch | `Uprobes::handle_swbp` |
| `handler_chain()` | per-consumer-list iter (RCU-tasks-trace) | `Uprobes::handler_chain` |
| `prepare_uretprobe()` | per-call hijack RA | `Uprobes::prepare_uretprobe` |
| `uprobe_handle_trampoline()` | per-return dispatch | `Uprobes::handle_trampoline` |
| `handle_uretprobe_chain()` | per-ret_handler chain | `Uprobes::handle_uretprobe_chain` |
| `pre_ssout()` | per-trap alloc XOL slot | `Uprobes::pre_ssout` |
| `handle_singlestep()` | per-singlestep-fini | `Uprobes::handle_singlestep` |
| `xol_get_insn_slot()` / `xol_free_insn_slot()` | per-slot alloc/free | `XolArea::get_slot` / `free_slot` |
| `__create_xol_area()` / `get_xol_area()` | per-mm install XOL | `XolArea::create` / `get` |
| `uprobe_notify_resume()` | per-TIF_UPROBE entry | `Uprobes::notify_resume` |
| `uprobe_pre_sstep_notifier()` / `_post_sstep_notifier()` | per-die-notifier | `Uprobes::pre_sstep` / `post_sstep` |
| `uprobe_deny_signal()` | per-singlestep mask non-fatal signals | `Uprobes::deny_signal` |
| `find_uprobe_rcu()` | per-(inode,offset) rbtree lookup | `Uprobes::find_rcu` |
| `find_active_uprobe_rcu()` / `_speculative()` | per-vaddr lookup | `Uprobes::find_active_rcu` |
| `alloc_uprobe()` / `insert_uprobe()` | per-(inode,offset) rbtree insert | `Uprobes::alloc` / `insert` |
| `put_uprobe()` / `get_uprobe()` / `try_get_uprobe()` | per-refcount | `Uprobe::put` / `get` / `try_get` |
| `hprobe_expire()` / `hprobe_consume()` / `hprobe_finalize()` | per-hprobe state machine | `Hprobe::expire` / `consume` / `finalize` |
| `update_ref_ctr()` / `__update_ref_ctr()` | per-SDT semaphore inc/dec | `Uprobes::update_ref_ctr` |
| `delayed_uprobe_add()` / `_remove()` | per-mm queue pending ref_ctr | `Uprobes::delayed_add` / `delayed_remove` |
| `arch_uretprobe_hijack_return_addr()` | per-arch RA hijack | `ArchUprobe::hijack_ret_addr` |
| `arch_uprobe_pre_xol()` / `_post_xol()` / `_abort_xol()` | per-arch XOL fixup | `ArchUprobe::pre_xol` / `post_xol` / `abort_xol` |
| `arch_uprobe_copy_ixol()` | per-arch copy mangled insn into XOL slot | `ArchUprobe::copy_ixol` |
| `arch_uretprobe_trampoline()` | per-arch trampoline opcode | `ArchUprobe::ret_trampoline` |
| `uprobe_get_trampoline_vaddr()` | per-mm trampoline addr | `Uprobes::trampoline_vaddr` |
| `uprobes_init()` | per-boot init | `Uprobes::init` |
| `uprobes_treelock` + `uprobes_seqcount` | global rbtree lock + seqcount | `Uprobes::tree_lock` |
| `dup_mmap_sem` (percpu rwsem) | per-fork serialization | `Uprobes::dup_mmap_sem` |
| `uretprobes_srcu` | hprobe-leased lifetime | `Uprobes::uretprobes_srcu` |
| `delayed_uprobe_lock` | per-mutex pending list | `Uprobes::delayed_lock` |
| `MMF_HAS_UPROBES` / `MMF_RECALC_UPROBES` | per-mm flags | shared |
| `UPROBE_HANDLER_REMOVE` / `_IGNORE` | per-consumer rc | shared |
| `MAX_URETPROBE_DEPTH` (64) | per-task uretprobe-chain cap | shared |
| `RI_TIMER_PERIOD` | per-task ri_timer expire-leased period | shared |

## Compatibility contract

REQ-1: struct uprobe:
- rb_node: per-(inode,offset) rbtree node in `uprobes_tree`.
- ref: refcount_t (per-instance + per-active-consumer).
- register_rwsem: serializes register/unregister/apply.
- consumer_rwsem: serializes consumer-list iteration vs add/del.
- pending_list, consumers (list_head): per-consumer linkage.
- inode: backing-file inode (held referenced).
- offset: per-file byte-offset of probed insn.
- ref_ctr_offset: per-SDT/USDT activation-counter offset (0 = none).
- flags: bit-0 = UPROBE_COPY_INSN (insn copied + arch-analyzed).
- arch: `struct arch_uprobe` (original insn + ixol mangled copy).
- union { rcu_head; work_struct }: per-free path.

REQ-2: struct uprobe_consumer:
- handler(self, regs, *data): per-call entry; rc ∈ {0, UPROBE_HANDLER_REMOVE=1, UPROBE_HANDLER_IGNORE=2}.
- ret_handler(self, func, regs, *data): per-call exit; opt-in via `arch_uretprobe_hijack_return_addr`.
- filter(self, mm): per-mm gate; required if handler ever returns UPROBE_HANDLER_REMOVE.
- cons_node: per-list linkage in `uprobe->consumers`.
- id: per-consumer identity (assigned on register).

REQ-3: uprobe_register(inode, offset, ref_ctr_offset, uc) -> *uprobe ∨ ERR_PTR:
- /* Precondition: at least one of handler/ret_handler set */
- if !uc.handler ∧ !uc.ret_handler: return -EINVAL.
- if !inode.i_mapping.a_ops.read_folio ∧ !shmem_mapping(inode.i_mapping): return -EIO.
- if offset > i_size_read(inode): return -EINVAL.
- if !IS_ALIGNED(offset, UPROBE_SWBP_INSN_SIZE): return -EINVAL.
- if !IS_ALIGNED(ref_ctr_offset, 2): return -EINVAL.
- uprobe = alloc_uprobe(inode, offset, ref_ctr_offset). /* rbtree insert or find existing */
- down_write(uprobe.register_rwsem).
- consumer_add(uprobe, uc).
- ret = register_for_each_vma(uprobe, uc).
- up_write(uprobe.register_rwsem).
- if ret: uprobe_unregister_nosync(uprobe, uc) + put_uprobe.
- return uprobe.

REQ-4: alloc_uprobe(inode, offset, ref_ctr_offset):
- u = kzalloc(struct uprobe).
- u.inode = igrab(inode). u.offset = offset. u.ref_ctr_offset = ref_ctr_offset.
- refcount_set(u.ref, 1).
- init_rwsem(register_rwsem) + init_rwsem(consumer_rwsem) + INIT_LIST_HEAD(consumers).
- cur = insert_uprobe(u). /* may return existing entry */
- if cur != u: kfree(u); if cur.ref_ctr_offset != ref_ctr_offset: ref_ctr_mismatch_warn; return ERR.
- return cur.

REQ-5: install_breakpoint(uprobe, vma, vaddr):
- /* Holds mm.mmap_lock for writing (caller register_for_each_vma) */
- ret = prepare_uprobe(uprobe, vma.vm_file, mm, vaddr). /* copy_insn + arch_uprobe_analyze_insn under register_rwsem */
- first_uprobe = !mm_flags_test(MMF_HAS_UPROBES, mm).
- if first_uprobe: mm_flags_set(MMF_HAS_UPROBES, mm).
- ret = set_swbp(uprobe.arch, vma, vaddr). /* arch-weak: writes UPROBE_SWBP_INSN via uprobe_write_opcode COW */
- if !ret: mm_flags_clear(MMF_RECALC_UPROBES, mm).
- else if first_uprobe: mm_flags_clear(MMF_HAS_UPROBES, mm).

REQ-6: uprobe_write_opcode / uprobe_write:
- Single-page COW-replace path for the int3 / orig-insn byte(s).
- Locks: mmap_write_lock(mm) held by caller.
- If page is shared (COW), break_cow / fork the page first (orig_page_is_identical check).
- copy_to_page(new_page, vaddr, insn, nbytes).
- /* Update ref_ctr atomically if present */
- if uprobe.ref_ctr_offset && do_update_ref_ctr: update_ref_ctr(uprobe, mm, +1 or -1).
- Pair with smp_wmb so UPROBE_COPY_INSN observable before insn byte.

REQ-7: set_swbp / set_orig_insn (arch-weak):
- set_swbp: write UPROBE_SWBP_INSN at vaddr (x86 = 0xCC int3).
- set_orig_insn: write `uprobe.arch.insn` back.
- Both call uprobe_write with `verify_opcode` to detect mismatched state.

REQ-8: register_for_each_vma(uprobe, new):
- is_register = !!new.
- percpu_down_write(dup_mmap_sem). /* serialize against fork's uprobe_dup_mmap */
- info = build_map_info(uprobe.inode.i_mapping, uprobe.offset, is_register). /* enumerate all mms mapping that file-offset via i_mmap interval-tree */
- for each (mm, vaddr) in info:
  - mmap_write_lock(mm).
  - if check_stable_address_space(mm): skip.
  - vma = find_vma(mm, vaddr); validate (valid_vma, inode-match, offset-match).
  - if is_register ∧ consumer_filter(new, mm): err = install_breakpoint(uprobe, vma, vaddr).
  - else if MMF_HAS_UPROBES ∧ !filter_chain(uprobe, mm): err |= remove_breakpoint(uprobe, vma, vaddr).
  - mmap_write_unlock(mm); mmput(mm).
- percpu_up_write(dup_mmap_sem).

REQ-9: build_map_info(mapping, offset, is_register):
- pgoff = offset >> PAGE_SHIFT.
- i_mmap_lock_read(mapping).
- for each vma in vma_interval_tree_foreach(mapping.i_mmap, pgoff..pgoff):
  - if !valid_vma(vma, is_register): continue.
  - if !mmget_not_zero(vma.vm_mm): continue.
  - prepend (mm, vaddr=offset_to_vaddr(vma, offset)) to result list.
- i_mmap_unlock_read.
- If allocation under i_mmap lock failed, drop lock + retry with pre-allocated nodes.

REQ-10: valid_vma(vma, is_register):
- flags = VM_HUGETLB | VM_MAYEXEC | VM_MAYSHARE; if is_register: flags |= VM_WRITE.
- return vma.vm_file ∧ (vma.vm_flags & flags) == VM_MAYEXEC.

REQ-11: uprobe_unregister_nosync(uprobe, uc) + uprobe_unregister_sync():
- /* No-sync phase */
- down_write(uprobe.register_rwsem).
- consumer_del(uprobe, uc).
- err = register_for_each_vma(uprobe, NULL). /* remove int3 if no remaining consumers */
- up_write(uprobe.register_rwsem).
- if err: uprobe_warn("leaking uprobe"); return.
- put_uprobe(uprobe). /* may free via call_rcu_tasks_trace */
- /* Sync phase (separate call) */
- synchronize_rcu_tasks_trace(). /* drain handler_chain readers */
- synchronize_srcu(uretprobes_srcu). /* drain leased hprobe readers */

REQ-12: uprobe_apply(uprobe, uc, add):
- /* Per-consumer enable/disable: re-evaluate filters across all mms */
- down_write(uprobe.register_rwsem).
- err = register_for_each_vma(uprobe, add ? uc : NULL).
- up_write(uprobe.register_rwsem).

REQ-13: uprobe_mmap(vma):
- /* Hook from mmap; install int3 for any uprobes matching this vma's inode + range */
- if !valid_vma(vma, true): return 0.
- inode = file_inode(vma.vm_file). offset_lo = vma.vm_pgoff << PAGE_SHIFT. offset_hi = offset_lo + vma_size.
- mutex_lock(uprobes_mmap_mutex[hash(inode)]).
- build_probe_list(inode, vma.vm_start, vma.vm_end, probes). /* find_node_in_range on rbtree */
- for u in probes: install_breakpoint(u, vma, offset_to_vaddr(vma, u.offset)).
- mutex_unlock.
- /* Delayed ref_ctr (SDT semaphore) */
- delayed_ref_ctr_inc(vma).

REQ-14: uprobe_munmap(vma, start, end):
- if mm_flags_test(MMF_HAS_UPROBES, vma.vm_mm) ∧ vma_has_uprobes(vma, start, end):
  - mm_flags_set(MMF_RECALC_UPROBES, vma.vm_mm).

REQ-15: handle_swbp(regs) — per-int3 entry:
- bp_vaddr = uprobe_get_swbp_addr(regs). /* arch: IP - UPROBE_SWBP_INSN_SIZE */
- if bp_vaddr == uprobe_get_trampoline_vaddr(): return uprobe_handle_trampoline(regs).
- rcu_read_lock_trace().
- uprobe = find_active_uprobe_rcu(bp_vaddr, &is_swbp).
- if !uprobe:
  - if is_swbp > 0: force_sig(SIGTRAP).
  - else: instruction_pointer_set(regs, bp_vaddr). /* restart */
  - goto out.
- instruction_pointer_set(regs, bp_vaddr).
- if !test_bit(UPROBE_COPY_INSN, uprobe.flags): goto out.
- smp_rmb(). /* pairs with prepare_uprobe smp_wmb */
- if !get_utask(): goto out.
- if arch_uprobe_ignore(uprobe.arch, regs): goto out.
- handler_chain(uprobe, regs). /* runs consumer.handler() under RCU-tasks-trace */
- arch_uprobe_optimize(uprobe.arch, bp_vaddr). /* x86: try jmp-replacement after first hit */
- if instruction_pointer(regs) != bp_vaddr: goto out. /* handler diverted PC */
- if arch_uprobe_skip_sstep(uprobe.arch, regs): goto out. /* trivial-insn emulation */
- pre_ssout(uprobe, regs, bp_vaddr). /* schedule XOL singlestep */
- out: rcu_read_unlock_trace.

REQ-16: handler_chain(uprobe, regs):
- utask.auprobe = uprobe.arch.
- list_for_each_entry_rcu(uc in uprobe.consumers, cons_node, rcu_read_lock_trace_held()):
  - session = uc.handler ∧ uc.ret_handler.
  - if uc.handler: rc = uc.handler(uc, regs, cookie).
  - remove &= (rc == UPROBE_HANDLER_REMOVE).
  - if !uc.ret_handler ∨ ignore_ret_handler(rc): continue.
  - if !ri: ri = alloc_return_instance(utask).
  - if session: ri = push_consumer(ri, uc.id, cookie).
- utask.auprobe = NULL.
- if ri: prepare_uretprobe(uprobe, regs, ri). /* hijack RA */
- if remove ∧ has_consumers: re-check filter under register_rwsem, then unapply for current mm.

REQ-17: pre_ssout(uprobe, regs, bp_vaddr) — XOL setup:
- if !try_get_uprobe(uprobe): return -EINVAL.
- if !xol_get_insn_slot(uprobe, utask): put_uprobe + return -ENOMEM.
- utask.vaddr = bp_vaddr.
- err = arch_uprobe_pre_xol(uprobe.arch, regs). /* set regs.IP = utask.xol_vaddr; arch fixups (rip-relative, etc.) */
- if err: xol_free_insn_slot; put_uprobe; return err.
- utask.active_uprobe = uprobe.
- utask.state = UTASK_SSTEP.
- /* On return to user, CPU executes mangled-insn @ xol_vaddr, single-step trap drops back into handle_singlestep */

REQ-18: handle_singlestep(utask, regs):
- uprobe = utask.active_uprobe.
- if state == UTASK_SSTEP_ACK: err = arch_uprobe_post_xol(uprobe.arch, regs). /* fixup PC, rip-rel */
- else if state == UTASK_SSTEP_TRAPPED: arch_uprobe_abort_xol(uprobe.arch, regs).
- else: WARN.
- put_uprobe(uprobe). utask.active_uprobe = NULL. utask.state = UTASK_RUNNING.
- xol_free_insn_slot(utask).
- if utask.signal_denied: TIF_SIGPENDING re-armed.
- if err: force_sig(SIGILL).

REQ-19: XOL area — `struct xol_area`:
- wq: per-slot-wait waitqueue (UINSNS_PER_PAGE slots).
- bitmap: per-slot allocation (bit-0 reserved for trampoline).
- page: anonymous executable page.
- vaddr: per-mm fixed (or arch-chosen) install address.
- VMA flags on install: VM_EXEC|VM_MAYEXEC|VM_DONTCOPY|VM_IO|VM_SEALED_SYSMAP.
- Special-mapping fault op = xol_fault (return shared page); mremap op = xol_mremap (track new vaddr).

REQ-20: xol_get_insn_slot / xol_free_insn_slot:
- get:
  - area = get_xol_area(); if !area: return false.
  - wait_event(area.wq, slot_nr = find_first_zero_bit(area.bitmap, UINSNS_PER_PAGE), set bit).
  - utask.xol_vaddr = area.vaddr + slot_nr * UPROBE_XOL_SLOT_BYTES.
  - arch_uprobe_copy_ixol(area.page, xol_vaddr, &uprobe.arch.ixol, sizeof(uprobe.arch.ixol)).
- free:
  - slot_nr = (utask.xol_vaddr - area.vaddr) / UPROBE_XOL_SLOT_BYTES.
  - clear_bit(slot_nr, area.bitmap); smp_mb__after_atomic; wake_up(area.wq).

REQ-21: prepare_uretprobe(uprobe, regs, ri) — return-probe hijack:
- if !get_xol_area: goto free.
- if utask.depth >= MAX_URETPROBE_DEPTH (64): printk_ratelimited; goto free.
- trampoline_vaddr = uprobe_get_trampoline_vaddr().
- orig_ret_vaddr = arch_uretprobe_hijack_return_addr(trampoline_vaddr, regs). /* writes trampoline to stack[SP], returns prior value */
- chained = (orig_ret_vaddr == trampoline_vaddr). /* nested uretprobe */
- cleanup_return_instances(utask, chained, regs). /* drop entries invalidated by longjmp */
- if chained: orig_ret_vaddr = utask.return_instances.orig_ret_vaddr.
- srcu_idx = __srcu_read_lock(uretprobes_srcu).
- ri.func = IP. ri.stack = SP. ri.orig_ret_vaddr = orig_ret_vaddr. ri.chained = chained.
- utask.depth++.
- hprobe_init_leased(ri.hprobe, uprobe, srcu_idx). /* SRCU-protected lease */
- ri.next = utask.return_instances; rcu_assign_pointer(utask.return_instances, ri).
- mod_timer(utask.ri_timer, jiffies + RI_TIMER_PERIOD). /* arms hprobe_expire downgrade */

REQ-22: uprobe_handle_trampoline(regs) — return dispatch:
- ri = utask.return_instances.
- /* Walk stack-of-RIs until reaching unchained ancestor */
- next = find_next_ret_chain(ri).
- for each ri in [head..next]: handle_uretprobe_chain(ri, uprobe, regs).
- /* Restore original RA */
- instruction_pointer_set(regs, next.orig_ret_vaddr).
- /* Pop processed entries */
- utask.return_instances = next.next; utask.depth -= popped; free_ret_instance each.

REQ-23: handle_uretprobe_chain(ri, uprobe, regs):
- list_for_each_entry_rcu(uc in uprobe.consumers, cons_node, rcu_read_lock_trace_held()):
  - if !uc.ret_handler: continue.
  - cookie = consumer-cookie lookup via return_consumer_find(ri, &iter, uc.id).
  - uc.ret_handler(uc, ri.func, regs, &cookie).

REQ-24: hprobe state machine (hybrid lifetime):
- enum: HPROBE_LEASED (uretprobes_srcu), HPROBE_STABLE (refcounted via uprobe.ref), HPROBE_GONE (uprobe died), HPROBE_CONSUMED (one-shot consumed).
- hprobe_init_leased: state=LEASED, srcu_idx recorded, *stable=uprobe, *leased=uprobe.
- hprobe_expire (timer): atomic xchg leased -> NULL or __UPROBE_DEAD, on transition try refcount_inc_not_zero; if fail, mark dead. Pair with srcu_read_unlock.
- hprobe_consume / hprobe_finalize: lookup current state under SRCU then put/srcu_read_unlock accordingly.

REQ-25: ri_timer (per-utask periodic):
- Fires every RI_TIMER_PERIOD; iterates return_instances; calls hprobe_expire on each to downgrade LEASED -> STABLE (refcounted) or LEASED -> GONE.
- Uses utask.ri_seqcount to coordinate with free_ret_instance.

REQ-26: uprobe_copy_process(child, clone_flags):
- /* fork(2) / clone(2): copy utask + xol depending on CLONE_VM */
- if CLONE_VM ∧ parent.utask:
  - child.utask = alloc_utask + dup_utask(child, parent.utask). /* copy return_instances stack */
- if !CLONE_VM: child gets fresh xol when needed (uprobe_dup_mmap copies MMF_HAS_UPROBES + MMF_RECALC_UPROBES).
- For CLONE_VM with parent in mid-SSTEP, schedule dup_xol_work to recreate slot.

REQ-27: uprobe_pre_sstep_notifier / uprobe_post_sstep_notifier:
- pre: if current.mm ∧ (MMF_HAS_UPROBES ∨ utask.return_instances): set TIF_UPROBE; return 1.
- post: if active_uprobe: utask.state = UTASK_SSTEP_ACK; set TIF_UPROBE; return 1.
- Both invoked from arch_uprobe_exception_notify (die-notifier priority INT_MAX-1).

REQ-28: uprobe_notify_resume(regs):
- /* Called from exit-to-user when TIF_UPROBE set */
- clear_thread_flag(TIF_UPROBE).
- utask = current.utask.
- if utask ∧ utask.active_uprobe: handle_singlestep(utask, regs).
- else: handle_swbp(regs).

REQ-29: uprobe_deny_signal():
- if !utask ∨ !utask.active_uprobe: return false.
- if task_sigpending(t):
  - utask.signal_denied = true; clear TIF_SIGPENDING.
  - if __fatal_signal_pending ∨ arch_uprobe_xol_was_trapped: utask.state = UTASK_SSTEP_TRAPPED; set TIF_UPROBE.
- return true.

REQ-30: ref_ctr (SDT semaphore):
- ref_ctr_offset != 0 ⟹ inc the u16 at (inode-mapped vaddr + ref_ctr_offset) when first consumer attaches, dec on last detach.
- update_ref_ctr(uprobe, mm, +/-1): get_user_pages_remote → kmap → atomic_add → put_page.
- If mm not yet fully built (uprobe_mmap path before xol exists), enqueue delayed_uprobe_add for later application; processed at next stable point.

REQ-31: find_active_uprobe_speculative / find_active_uprobe_rcu:
- speculative path: mmap_lock_speculate_try_begin → vma_lookup → READ_ONCE(vm_file) → find_uprobe_rcu(inode, offset) → mmap_lock_speculate_retry validation.
- fallback: mmap_read_lock; vma_lookup; find_uprobe_rcu; if !uprobe set is_swbp = is_trap_at_addr(mm, bp_vaddr) (faulty insn vs stale int3).

REQ-32: uprobes_init():
- For UPROBES_HASH_SZ buckets: mutex_init(uprobes_mmap_mutex[i]).
- BUG_ON(register_die_notifier(uprobe_exception_nb)).

## Acceptance Criteria

- [ ] AC-1: uprobe_register on (inode, offset, NULL ref_ctr, uc) installs UPROBE_SWBP_INSN in every existing VMA mapping that file-offset and returns *uprobe with refcount=1.
- [ ] AC-2: A second uprobe_register for the same (inode, offset) returns the same *uprobe (rbtree de-dup) and adds the consumer to the existing list.
- [ ] AC-3: Process executes the probed instruction → handle_swbp runs every consumer.handler() in registration order under RCU-tasks-trace.
- [ ] AC-4: handler returning UPROBE_HANDLER_REMOVE with filter()=false ⟹ breakpoint removed from current mm only (not others).
- [ ] AC-5: pre_ssout allocates an XOL slot, mangled insn executes out-of-line, handle_singlestep restores PC past original insn.
- [ ] AC-6: uretprobe: at function entry, arch_uretprobe_hijack_return_addr replaces stack RA with trampoline; on function return uprobe_handle_trampoline fires ret_handler with original `func` IP, then restores orig_ret_vaddr.
- [ ] AC-7: utask.depth ≥ MAX_URETPROBE_DEPTH (64) ⟹ prepare_uretprobe skips (no hijack); printk_ratelimited.
- [ ] AC-8: XOL slot exhaustion ⟹ xol_get_insn_slot waits on area.wq until a slot frees.
- [ ] AC-9: uprobe_mmap on a freshly mapped region matching an existing uprobe installs the int3 lazily.
- [ ] AC-10: ref_ctr_offset != 0 ⟹ first consumer attach increments u16 semaphore in user-page; last detach decrements; SDT probes observe.
- [ ] AC-11: fork(2) with CLONE_VM duplicates utask (including return-probe stack) into child; without CLONE_VM, MMF_HAS_UPROBES propagated for re-instrumentation.
- [ ] AC-12: uprobe_unregister_nosync followed by uprobe_unregister_sync waits for all in-flight handler_chain readers (RCU-tasks-trace + uretprobes_srcu) before freeing.
- [ ] AC-13: uprobe_deny_signal during UTASK_SSTEP defers non-fatal signals until singlestep finishes; fatal signals trap the XOL slot and re-raise after restore.
- [ ] AC-14: TIF_UPROBE set by pre/post sstep notifier ⟹ uprobe_notify_resume runs on exit-to-user.
- [ ] AC-15: hprobe lease expires (timer) ⟹ downgraded to STABLE (refcounted) or GONE (refcount_inc_not_zero failed).

## Architecture

```
struct Uprobe {
  rb_node: RbNode,
  ref: Refcount,
  register_rwsem: RwSemaphore,
  consumer_rwsem: RwSemaphore,
  pending_list: ListHead,
  consumers: ListHead,            // struct UprobeConsumer.cons_node
  inode: *Inode,                  // igrab'd
  rcu_or_work: Union<RcuHead, WorkStruct>,
  offset: i64,
  ref_ctr_offset: i64,
  flags: AtomicUsize,             // bit UPROBE_COPY_INSN
  arch: ArchUprobe,               // insn + ixol
}

struct UprobeConsumer {
  handler: Option<fn(&Self, &mut PtRegs, &mut u64) -> i32>,
  ret_handler: Option<fn(&Self, u64 /*func*/, &mut PtRegs, &mut u64) -> i32>,
  filter: Option<fn(&Self, &MmStruct) -> bool>,
  cons_node: ListHead,
  id: u64,
}

struct XolArea {
  wq: WaitQueueHead,
  bitmap: *mut u64,               // UINSNS_PER_PAGE bits
  page: *Page,
  vaddr: usize,
}

struct UprobeTask {
  state: UprobeTaskState,         // RUNNING | SSTEP | SSTEP_ACK | SSTEP_TRAPPED
  depth: u32,
  return_instances: *ReturnInstance,
  ri_pool: *ReturnInstance,
  ri_timer: TimerList,
  ri_seqcount: Seqcount,
  union {
    sstep { autask: ArchUprobeTask, vaddr: usize },
    dup   { dup_xol_work: CallbackHead, dup_xol_addr: usize },
  },
  active_uprobe: Option<*Uprobe>,
  xol_vaddr: usize,
  signal_denied: bool,
  auprobe: *ArchUprobe,
}

struct ReturnInstance {
  hprobe: Hprobe,
  func: usize,
  stack: usize,
  orig_ret_vaddr: usize,
  chained: bool,
  cons_cnt: i32,
  next: *ReturnInstance,
  rcu: RcuHead,
  consumer: ReturnConsumer,       // singular fast-path
  extra_consumers: *ReturnConsumer,
}

struct Hprobe {
  state: HprobeState,             // LEASED | STABLE | GONE | CONSUMED
  srcu_idx: i32,
  uprobe: *Uprobe,
}
```

`Uprobes::register(inode, offset, ref_ctr_offset, uc) -> Result<*Uprobe>`:
1. /* Precondition gates */
2. if !uc.handler ∧ !uc.ret_handler: return Err(-EINVAL).
3. if !inode.has_readable_mapping: return Err(-EIO).
4. if offset > inode.size(): return Err(-EINVAL).
5. if !aligned(offset, UPROBE_SWBP_INSN_SIZE): return Err(-EINVAL).
6. if !aligned(ref_ctr_offset, 2): return Err(-EINVAL).
7. /* rbtree alloc-or-find */
8. uprobe = Uprobes::alloc(inode, offset, ref_ctr_offset)?
9. /* Add consumer + propagate to all maps */
10. uprobe.register_rwsem.write():
    - consumer_add(uprobe, uc).
    - register_for_each_vma(uprobe, Some(uc)).
11. if err: uprobe.unregister_nosync(uc).
12. Ok(uprobe).

`Uprobes::handle_swbp(regs)`:
1. bp_vaddr = arch_get_swbp_addr(regs).
2. if bp_vaddr == trampoline_vaddr: return Uprobes::handle_trampoline(regs).
3. rcu_read_lock_trace().
4. uprobe = find_active_uprobe_rcu(bp_vaddr, &mut is_swbp).
5. if uprobe.is_none():
   - if is_swbp > 0: force_sig(SIGTRAP).
   - else: regs.ip = bp_vaddr. /* restart */
   - rcu_read_unlock_trace; return.
6. regs.ip = bp_vaddr.
7. if !uprobe.flags.test(UPROBE_COPY_INSN): rcu_read_unlock_trace; return.
8. smp_rmb(). /* observe arch fields */
9. if !get_utask(): rcu_read_unlock_trace; return.
10. if arch_uprobe_ignore(uprobe.arch, regs): rcu_read_unlock_trace; return.
11. Uprobes::handler_chain(uprobe, regs). /* may hijack RA */
12. arch_uprobe_optimize(uprobe.arch, bp_vaddr).
13. if regs.ip != bp_vaddr: rcu_read_unlock_trace; return. /* handler diverted */
14. if arch_uprobe_skip_sstep(uprobe.arch, regs): rcu_read_unlock_trace; return.
15. Uprobes::pre_ssout(uprobe, regs, bp_vaddr).
16. rcu_read_unlock_trace.

`Uprobes::pre_ssout(uprobe, regs, bp_vaddr) -> Result<()>`:
1. if !try_get_uprobe(uprobe): return Err(-EINVAL).
2. if !XolArea::get_slot(uprobe, utask):
   - put_uprobe(uprobe); return Err(-ENOMEM).
3. utask.vaddr = bp_vaddr.
4. err = arch_uprobe_pre_xol(uprobe.arch, regs). /* IP = utask.xol_vaddr */
5. if err: XolArea::free_slot(utask); put_uprobe(uprobe); return err.
6. utask.active_uprobe = Some(uprobe); utask.state = UTASK_SSTEP.
7. Ok.

`Uprobes::handle_singlestep(utask, regs)`:
1. uprobe = utask.active_uprobe.unwrap.
2. match utask.state:
   - UTASK_SSTEP_ACK => err = arch_uprobe_post_xol(uprobe.arch, regs).
   - UTASK_SSTEP_TRAPPED => arch_uprobe_abort_xol(uprobe.arch, regs).
   - _ => WARN.
3. put_uprobe(uprobe).
4. utask.active_uprobe = None; utask.state = UTASK_RUNNING.
5. XolArea::free_slot(utask).
6. if utask.signal_denied: TIF_SIGPENDING; signal_denied=false.
7. if err: force_sig(SIGILL).

`Uprobes::prepare_uretprobe(uprobe, regs, ri)`:
1. if XolArea::get(current.mm).is_none(): goto free.
2. if utask.depth >= MAX_URETPROBE_DEPTH: printk_ratelimited; goto free.
3. trampoline = trampoline_vaddr().
4. orig_ret = arch_uretprobe_hijack_return_addr(trampoline, regs).
5. if orig_ret == usize::MAX: goto free.
6. chained = (orig_ret == trampoline).
7. cleanup_return_instances(utask, chained, regs).
8. if chained: orig_ret = utask.return_instances.orig_ret_vaddr.
9. srcu_idx = __srcu_read_lock(uretprobes_srcu).
10. ri.func = regs.ip; ri.stack = regs.sp; ri.orig_ret_vaddr = orig_ret; ri.chained = chained.
11. utask.depth += 1.
12. Hprobe::init_leased(&ri.hprobe, uprobe, srcu_idx).
13. ri.next = utask.return_instances; rcu_assign_pointer(utask.return_instances, ri).
14. mod_timer(utask.ri_timer, jiffies + RI_TIMER_PERIOD).

`XolArea::create(vaddr) -> Option<&XolArea>`:
1. area = kzalloc(XolArea).
2. area.bitmap = kcalloc(UINSNS_PER_PAGE_BITS).
3. area.page = alloc_page(GFP_HIGHUSER | __GFP_ZERO).
4. area.vaddr = vaddr ∨ arch_uprobe_get_xol_area().
5. init_waitqueue_head(area.wq).
6. set_bit(0, area.bitmap). /* reserve slot 0 for trampoline */
7. insns = arch_uretprobe_trampoline(&size).
8. arch_uprobe_copy_ixol(area.page, 0, insns, size). /* trampoline = swbp */
9. if !xol_add_vma(mm, area): cleanup; return None.
10. /* smp_store_release(mm.uprobes_state.xol_area, area) */
11. Some(area).

`Uprobes::dup_mmap(oldmm, newmm)`:
1. if MMF_HAS_UPROBES in oldmm:
   - MMF_HAS_UPROBES in newmm.
   - MMF_RECALC_UPROBES in newmm. /* dup_mmap skips VM_DONTCOPY: xol vma not copied */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `uprobe_refcount_balanced` | INVARIANT | per-handle_swbp: try_get + put balanced across pre_ssout / handle_singlestep error paths. |
| `register_rwsem_held_for_consumer_list_mutation` | INVARIANT | consumer_add/del under register_rwsem write. |
| `consumer_rwsem_held_for_filter_chain` | INVARIANT | filter_chain reads consumers under consumer_rwsem read. |
| `mmap_write_lock_held_for_install_breakpoint` | INVARIANT | install_breakpoint runs with mm.mmap_lock writer. |
| `dup_mmap_sem_serializes_register_vs_fork` | INVARIANT | register_for_each_vma percpu_down_write vs uprobe_start_dup_mmap percpu_down_read. |
| `rcu_tasks_trace_held_for_handler_chain` | INVARIANT | handler_chain iterates consumers under rcu_read_lock_trace. |
| `xol_slot_bit_owned` | INVARIANT | get_insn_slot test_and_set returns owned slot; free_insn_slot clears same bit. |
| `xol_slot_in_page` | INVARIANT | utask.xol_vaddr ∈ [area.vaddr, area.vaddr+PAGE_SIZE). |
| `return_instance_stack_lifo` | INVARIANT | return_instances threaded LIFO via next; depth = list-length. |
| `max_uretprobe_depth_enforced` | INVARIANT | depth ≤ MAX_URETPROBE_DEPTH (64). |
| `hprobe_consume_after_init_leased` | INVARIANT | hprobe.state transitions: LEASED → STABLE ∨ GONE; never LEASED → CONSUMED w/o consume. |
| `uretprobes_srcu_balanced` | INVARIANT | per hprobe_init_leased there is matching hprobe_finalize srcu_read_unlock. |
| `inode_held_for_uprobe_lifetime` | INVARIANT | igrab/iput balanced per uprobe.ref. |
| `tif_uprobe_only_set_under_mm` | INVARIANT | TIF_UPROBE set only when current.mm ∧ (MMF_HAS_UPROBES ∨ utask). |

### Layer 2: TLA+

`kernel/events/uprobes.tla`:
- Models: register → install int3 → trap → handler_chain → pre_ssout → singlestep → handle_singlestep → unregister.
- Models: uretprobe stack hijack → handle_trampoline → ret_handler chain.
- Properties:
  - `safety_int3_owns_target_byte` — only ever one writer (mmap_write_lock) per breakpoint update.
  - `safety_xol_slot_disjoint` — no two utasks own the same slot concurrently.
  - `safety_ri_stack_lifo` — return_instances stack pops match pushes (chained correctly).
  - `safety_handler_chain_consumers_stable` — RCU-tasks-trace guarantees consumer list snapshot.
  - `safety_unregister_drains_handlers` — after unregister_sync, no handler_chain reader observes the removed consumer.
  - `liveness_xol_slot_acquired` — under bounded contention, get_insn_slot eventually returns.
  - `liveness_uretprobe_trampoline_fires` — every prepare_uretprobe pushes ri that is eventually popped by handle_trampoline (unless task exits).
  - `safety_max_uretprobe_depth` — depth never exceeds MAX_URETPROBE_DEPTH.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Uprobes::register` post: rbtree contains (inode, offset); consumers includes uc | `Uprobes::register` |
| `Uprobes::install_breakpoint` post: target byte == UPROBE_SWBP_INSN ∨ ret = err | `Uprobes::install_breakpoint` |
| `Uprobes::remove_breakpoint` post: target byte restored from uprobe.arch.insn ∨ ret = err | `Uprobes::remove_breakpoint` |
| `Uprobes::handle_swbp` pre: rcu_read_lock_trace held | `Uprobes::handle_swbp` |
| `Uprobes::handler_chain` post: every uc.handler() invoked at most once | `Uprobes::handler_chain` |
| `Uprobes::pre_ssout` post: utask.active_uprobe == uprobe ∧ state == UTASK_SSTEP | `Uprobes::pre_ssout` |
| `Uprobes::handle_singlestep` post: utask.active_uprobe == None ∧ state == UTASK_RUNNING ∧ slot freed | `Uprobes::handle_singlestep` |
| `XolArea::get_slot` post: slot bit set ∧ xol_vaddr in page | `XolArea::get_slot` |
| `Uprobes::prepare_uretprobe` post: depth' = depth + 1 ∧ ri.next = old-head | `Uprobes::prepare_uretprobe` |
| `Hprobe::expire` post: state ∈ {STABLE, GONE} | `Hprobe::expire` |

### Layer 4: Verus/Creusot functional

`Per-uprobe register → install int3 in all matching VMAs → trap on execute → handler_chain (consumer order) → optional ret-hijack → pre_ssout → XOL singlestep → handle_singlestep → post_xol PC fixup` semantic equivalence: per-Documentation/trace/uprobetracer.rst + arch-specific `arch_uprobe_*` weak overrides.

## Hardening

(Inherits row-1 features from `kernel/perf/00-overview.md` § Hardening.)

uprobes reinforcement:

- **Per-(inode, offset) rbtree de-dup** — defense against per-double-install racing two consumers and corrupting the int3 byte.
- **Per-register_rwsem write for register / unregister / apply** — defense against consumer-list mutation interleaving with register_for_each_vma.
- **Per-RCU-tasks-trace + per-uretprobes_srcu unregister drain** — defense against UAF of just-freed consumer in handler_chain / handle_uretprobe_chain.
- **Per-percpu dup_mmap_sem rwsem** — defense against per-fork interleaving with register_for_each_vma that would skip the newborn mm.
- **Per-mmap_write_lock for install_breakpoint** — defense against per-page-table mutation racing the COW write of int3 byte.
- **Per-UPROBE_COPY_INSN bit + smp_wmb/smp_rmb pair** — defense against per-handler-on-half-initialized-uprobe.
- **Per-XOL VM_DONTCOPY + VM_IO + VM_SEALED_SYSMAP** — defense against fork-inherited XOL leaking arbitrary executable mapping; defense against userspace mremap/munmap of XOL.
- **Per-bitmap slot 0 reserved for trampoline** — defense against trampoline overwrite by ordinary XOL request.
- **Per-MAX_URETPROBE_DEPTH (64) cap** — defense against unbounded stack-of-RIs (recursive function under uretprobe DoS).
- **Per-printk_ratelimited on uretprobe depth-exceed** — defense against per-log-flood.
- **Per-uprobe_deny_signal during SSTEP** — defense against signal-driven re-entry mangling XOL state.
- **Per-arch_uprobe_xol_was_trapped check** — defense against per-fault-in-XOL leaking control to wrong PC.
- **Per-CAP_SYS_ADMIN gate for kernel-space bp** (inherited from perf-hw-breakpoint path when uprobe is misused) — defense against per-int3-in-trap-handler recursion.
- **Per-ref_ctr offset alignment + page-bound check** — defense against per-cross-page atomic-add corrupting unrelated user data.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — copy-in/out of probed insn bytes (`copy_from_page`, XOL slot writes) is bounds-checked; a malicious offset cannot escalate into adjacent kernel memory.
- **PAX_KERNEXEC** — uprobe dispatch tables and `arch_uprobe_*` ops are W^X; the `uprobe_consumer` chain head is const-protected.
- **PAX_RANDKSTACK** — randomize the kernel stack at every `handle_swbp` and `handle_singlestep` entry.
- **PAX_REFCOUNT** — saturating atomics on `struct uprobe.ref`, `xol_area.slot_count`, RI depth counters, and per-mm uprobe maps.
- **PAX_MEMORY_SANITIZE** — scrub freed `struct return_instance` and XOL slot bytes on release so prior probed insns cannot be read back.
- **PAX_UDEREF** — strict user/kernel split when patching the int3 into the user COW page; the kernel writes via `__copy_to_user_inatomic` with usercopy checks engaged.
- **PAX_RAP / kCFI** — type-signed indirect calls for `consumer->handler`, `consumer->ret_handler`, and `arch_uprobe_*` per-arch ops.
- **GRKERNSEC_HIDESYM** — hide `uprobes_treelock`, `xol_area`, `handle_trampoline`, and tracepoint adapters from non-root /proc/kallsyms.
- **GRKERNSEC_DMESG** — gate "uretprobe: depth exceeded" / XOL fault diagnostics behind CAP_SYSLOG.
- **int3 patch lives in COW page only** — `set_swbp` refuses to install on a shared (non-COW) page; the modification must be process-private so it cannot perturb another mm.
- **xol_vma is per-process VM_DONTCOPY|VM_IO|VM_SEALED_SYSMAP** — fork does not inherit it; the user cannot mremap, munmap, or mprotect the XOL region; slot 0 is reserved for the trampoline.
- **URETPROBE stack capped at MAX_URETPROBE_DEPTH=64** — recursive functions under uretprobe cannot DoS-blow the RI stack; overflow is logged once via `printk_ratelimited` and the probe is detached.
- **ref_ctr page-bound atomic** — the SDT semaphore atomic-add is page-bound and alignment-checked so a malicious ELF cannot redirect the increment into adjacent user state.
- **uprobe install requires CAP_PERFMON / CAP_SYS_PTRACE** — userspace probes only attach via perf-event-open or ptrace paths with capability checks; no anon ioctl path.
- **Trampoline page is `__ro_after_init` once mapped** — once the xol_area trampoline is published, the underlying kernel-side template is not rewritten for the lifetime of the mm.
- **Rationale**: uprobes inject foreign control flow into a process via int3, so the trust boundary is exactly the line between "this mm" and "any other state". Forcing the patch onto a COW page, sealing the xol_vma against userspace manipulation, and capping URETPROBE depth eliminate the well-known XOL-leak, cross-mm-patch, and recursion-DoS classes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/kernel/uprobes.c` arch glue (covered separately if expanded; weak symbols documented above)
- `kernel/trace/trace_uprobe.c` tracefs uprobe consumer (covered in `trace.md` Tier-3)
- `kernel/events/core.c` perf-event uprobe PMU (covered in `perf-event-core.md` Tier-3)
- BPF uprobe-multi link (`kernel/trace/bpf_trace.c`) (covered in BPF Tier-3)
- `mm/khugepaged.c` interaction with XOL pages (covered in `khugepaged.md` Tier-3 if expanded)
- Implementation code
