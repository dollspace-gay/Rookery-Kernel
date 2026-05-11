# Tier-3: kernel/bpf/cpumap.c — BPF cpumap (XDP CPU redirect)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/cpumap.c (~834 lines)
  - include/linux/bpf.h (BPF_MAP_TYPE_CPUMAP, struct bpf_cpumap_val)
  - include/uapi/linux/bpf.h (BPF_XDP_CPUMAP expected_attach_type)
  - include/net/xdp.h (struct xdp_frame, xdp_do_redirect)
  - include/trace/events/xdp.h (trace_xdp_cpumap_enqueue, trace_xdp_cpumap_kthread)
-->

## Summary

The `cpumap` is the backend map for `bpf_redirect_map()` + `XDP_REDIRECT` that pushes raw XDP frames to a **specific remote CPU**, where SKB-allocation and normal-netstack delivery happen — separating the early-driver XDP layer (often on a busy NUMA-local RX core) from the rest of the stack. Per-`BPF_MAP_TYPE_CPUMAP`: key = CPU index (0..NR_CPUS), value = `struct bpf_cpumap_val { qsize, bpf_prog }`. Per-`bpf_cpu_map_entry`: each non-NULL slot owns a `ptr_ring` queue, a dedicated `cpu_map_kthread_run` kthread bound to `cpu`, per-CPU bulk staging in `xdp_bulk_queue[CPU_MAP_BULK_SIZE=8]`, GRO node, and an optional secondary XDP `bpf_prog` (`expected_attach_type == BPF_XDP_CPUMAP`) that re-runs on the consumer CPU. Per-`XDP_REDIRECT-to-cpumap`: NAPI-side enqueues into per-CPU bulk → on `xdp_do_flush()` the bulkq is committed into the remote ptr_ring under `producer_lock` → `wake_up_process(kthread)` → kthread batches up to `CPUMAP_BATCH=8` frames, optionally runs the per-entry XDP prog, builds SKBs via `__xdp_build_skb_from_frame` + `napi_skb_cache_get_bulk`, GRO-receives, and emits `trace_xdp_cpumap_{enqueue,kthread}`. Critical for: per-10G-wirespeed XDP prefiltering, per-NUMA-isolated softirq sinks, per-RPS-style scaling without RX-queue programming.

This Tier-3 covers `kernel/bpf/cpumap.c` (~834 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_cpu_map` | per-map container | `BpfCpuMap` |
| `struct bpf_cpu_map_entry` | per-CPU destination | `BpfCpuMapEntry` |
| `struct xdp_bulk_queue` | per-CPU staging bulk | `XdpBulkQueue` |
| `cpu_map_alloc()` | per-map alloc | `CpuMap::alloc` |
| `cpu_map_free()` | per-map teardown | `CpuMap::free` |
| `cpu_map_update_elem()` | per-slot install/replace | `CpuMap::update_elem` |
| `cpu_map_delete_elem()` | per-slot remove | `CpuMap::delete_elem` |
| `cpu_map_lookup_elem()` | per-syscall lookup | `CpuMap::lookup_elem` |
| `cpu_map_get_next_key()` | per-iter walk | `CpuMap::get_next_key` |
| `cpu_map_redirect()` | per-XDP redirect dispatch | `CpuMap::redirect` |
| `__cpu_map_entry_alloc()` | per-entry construct | `CpuMap::entry_alloc` |
| `__cpu_map_entry_replace()` | per-slot xchg + RCU-work free | `CpuMap::entry_replace` |
| `__cpu_map_entry_free()` | per-RCU-work entry free | `CpuMap::entry_free` |
| `__cpu_map_load_bpf_program()` | per-entry secondary prog attach | `CpuMap::load_bpf_program` |
| `cpu_map_kthread_run()` | per-CPU consumer loop | `CpuMap::kthread_run` |
| `cpu_map_bpf_prog_run()` | per-batch secondary XDP run | `CpuMap::bpf_prog_run` |
| `cpu_map_bpf_prog_run_xdp()` | per-frame XDP_REDIRECT/PASS | `CpuMap::bpf_prog_run_xdp` |
| `cpu_map_bpf_prog_run_skb()` | per-skb generic-XDP run | `CpuMap::bpf_prog_run_skb` |
| `cpu_map_gro_flush()` | per-iter GRO flush policy | `CpuMap::gro_flush` |
| `bq_enqueue()` | per-CPU bulk add | `CpuMap::bq_enqueue` |
| `bq_flush_to_queue()` | per-bulkq commit into ptr_ring | `CpuMap::bq_flush` |
| `cpu_map_enqueue()` | per-XDP-frame entry-point | `CpuMap::enqueue` |
| `cpu_map_generic_redirect()` | per-SKB generic redirect | `CpuMap::generic_redirect` |
| `__cpu_map_flush()` | per-NAPI flush_list drain | `CpuMap::flush` |
| `__cpu_map_ring_cleanup()` | per-teardown ptr_ring drain | `CpuMap::ring_cleanup` |
| `cpu_map_mem_usage()` | per-fdinfo accounting | `CpuMap::mem_usage` |
| `cpu_map_ops` | per-map_ops vtable | `CpuMap::OPS` |

## Compatibility contract

REQ-1: struct bpf_cpu_map:
- map: embedded struct bpf_map (refcount, max_entries, key_size=4, value_size, numa_node).
- cpu_map: __rcu **bpf_cpu_map_entry — per-slot array sized to max_entries.

REQ-2: struct bpf_cpu_map_entry:
- cpu: u32 — remote target CPU (== map key).
- map_id: i32 — back-ref to bpf_map.id (for tracepoints).
- bulkq: __percpu *xdp_bulk_queue — per-producer-CPU staging.
- queue: *ptr_ring — multi-producer / single-consumer FIFO sized to value.qsize.
- kthread: *task_struct — bound to cpu via kthread_bind.
- value: struct bpf_cpumap_val (qsize, bpf_prog.{fd,id}).
- prog: optional secondary *bpf_prog (BPF_PROG_TYPE_XDP, expected_attach_type BPF_XDP_CPUMAP).
- gro: struct gro_node — per-kthread GRO context.
- kthread_running: completion — kthread startup gate.
- free_work: rcu_work — deferred-free vehicle.

REQ-3: struct xdp_bulk_queue:
- q[CPU_MAP_BULK_SIZE=8] — staging slots.
- flush_node — links into per-CPU flush_list when count > 0.
- obj — back-ref to owning bpf_cpu_map_entry.
- count — current depth (0..8).
- bq_lock — local_lock_t (PREEMPT_RT serialization).

REQ-4: cpu_map_alloc(attr):
- /* Sanity */
- if attr.max_entries == 0 ∨ attr.key_size != 4: return -EINVAL.
- /* value_size must equal offsetofend(qsize) or offsetofend(bpf_prog.fd) */
- if !(value_size in {sizeof_to_qsize, sizeof_to_bpf_prog_fd}): return -EINVAL.
- if attr.map_flags & ~BPF_F_NUMA_NODE: return -EINVAL.
- if attr.max_entries > NR_CPUS: return -E2BIG.
- /* Allocate */
- cmap = bpf_map_area_alloc(sizeof(bpf_cpu_map)).
- bpf_map_init_from_attr(&cmap.map, attr).
- cmap.cpu_map = bpf_map_area_alloc(max_entries * sizeof(*entry)).
- return &cmap.map.

REQ-5: cpu_map_update_elem(map, key, value, flags):
- key_cpu = *(u32 *)key.
- /* Validate */
- if flags > BPF_EXIST: return -EINVAL.
- if key_cpu >= max_entries: return -E2BIG.
- if flags == BPF_NOEXIST: return -EEXIST.
- if value.qsize > 16384: return -EOVERFLOW.
- if key_cpu >= nr_cpumask_bits ∨ !cpu_possible(key_cpu): return -ENODEV.
- /* Build or null */
- if value.qsize == 0: rcpu = NULL.
- else: rcpu = __cpu_map_entry_alloc(map, &value, key_cpu).
- rcu_read_lock; __cpu_map_entry_replace(cmap, key_cpu, rcpu); rcu_read_unlock.
- return 0.

REQ-6: __cpu_map_entry_alloc(map, value, cpu):
- numa = cpu_to_node(cpu).
- rcpu = bpf_map_kmalloc_node(map, sizeof(*rcpu), GFP_KERNEL|__GFP_NOWARN|__GFP_ZERO, numa).
- /* Per-CPU bulkq */
- rcpu.bulkq = bpf_map_alloc_percpu(...).
- for_each_possible_cpu(i): bq.obj = rcpu; local_lock_init(&bq.bq_lock).
- /* ptr_ring */
- rcpu.queue = bpf_map_kmalloc_node(map, sizeof(ptr_ring), gfp, numa).
- ptr_ring_init(rcpu.queue, value.qsize, gfp).
- rcpu.cpu = cpu; rcpu.map_id = map.id; rcpu.value.qsize = value.qsize.
- gro_init(&rcpu.gro).
- /* Optional secondary XDP prog */
- if value.bpf_prog.fd > 0: __cpu_map_load_bpf_program(rcpu, map, fd).
- /* Kthread */
- init_completion(&rcpu.kthread_running).
- rcpu.kthread = kthread_create_on_node(cpu_map_kthread_run, rcpu, numa, "cpumap/%d/map:%d", cpu, map.id).
- kthread_bind(rcpu.kthread, cpu).
- wake_up_process(rcpu.kthread).
- wait_for_completion(&rcpu.kthread_running).
- return rcpu.

REQ-7: __cpu_map_load_bpf_program(rcpu, map, fd):
- prog = bpf_prog_get_type(fd, BPF_PROG_TYPE_XDP).
- if prog.expected_attach_type != BPF_XDP_CPUMAP: bpf_prog_put + return -EINVAL.
- if !bpf_prog_map_compatible(map, prog): bpf_prog_put + return -EINVAL.
- rcpu.value.bpf_prog.id = prog.aux.id.
- rcpu.prog = prog.

REQ-8: __cpu_map_entry_replace(cmap, key_cpu, rcpu):
- old = unrcu_pointer(xchg(&cmap.cpu_map[key_cpu], RCU_INITIALIZER(rcpu))).
- if old: INIT_RCU_WORK(&old.free_work, __cpu_map_entry_free); queue_rcu_work(system_percpu_wq, &old.free_work).

REQ-9: __cpu_map_entry_free(work):
- rcpu = container_of(to_rcu_work(work), bpf_cpu_map_entry, free_work).
- /* After 1 RCU grace period: no new XDP frames in flight to this rcpu. */
- kthread_stop(rcpu.kthread).
- if rcpu.prog: bpf_prog_put(rcpu.prog).
- gro_cleanup(&rcpu.gro).
- __cpu_map_ring_cleanup(rcpu.queue).
- ptr_ring_cleanup(rcpu.queue, NULL).
- kfree(rcpu.queue); free_percpu(rcpu.bulkq); kfree(rcpu).

REQ-10: cpu_map_delete_elem(map, key):
- key_cpu = *(u32 *)key.
- if key_cpu >= max_entries: return -EINVAL.
- __cpu_map_entry_replace(cmap, key_cpu, NULL).
- return 0.

REQ-11: cpu_map_free(map):
- synchronize_rcu() — drain XDP-side readers + flush ops.
- for i in 0..max_entries: rcpu = rcu_dereference_raw(cmap.cpu_map[i]); if rcpu: __cpu_map_entry_free(&rcpu.free_work.work) directly.
- bpf_map_area_free(cmap.cpu_map); bpf_map_area_free(cmap).

REQ-12: cpu_map_redirect(map, index, flags):
- return __bpf_xdp_redirect_map(map, index, flags, 0, __cpu_map_lookup_elem).
- /* allowed flags mask is 0 — no BPF_F_BROADCAST/EXCLUDE_INGRESS for cpumap */

REQ-13: cpu_map_enqueue(rcpu, xdpf, dev_rx):
- xdpf.dev_rx = dev_rx — captured for SKB construction on remote CPU.
- bq_enqueue(rcpu, xdpf).
- return 0.

REQ-14: bq_enqueue(rcpu, xdpf):
- local_lock_nested_bh(&rcpu.bulkq.bq_lock).
- bq = this_cpu_ptr(rcpu.bulkq).
- if bq.count == CPU_MAP_BULK_SIZE: bq_flush_to_queue(bq).
- bq.q[bq.count++] = xdpf.
- if !bq.flush_node.prev: list_add(&bq.flush_node, bpf_net_ctx_get_cpu_map_flush_list()).
- local_unlock_nested_bh.

REQ-15: bq_flush_to_queue(bq):
- lockdep_assert_held(&bq.bq_lock).
- if !bq.count: return.
- q = rcpu.queue; spin_lock(&q.producer_lock).
- for i in 0..bq.count: err = __ptr_ring_produce(q, bq.q[i]); if err: drops++; xdp_return_frame_rx_napi(xdpf).
- bq.count = 0; spin_unlock(&q.producer_lock).
- __list_del_clearprev(&bq.flush_node).
- trace_xdp_cpumap_enqueue(map_id, processed, drops, to_cpu).

REQ-16: __cpu_map_flush(flush_list):
- list_for_each_entry_safe(bq, tmp, flush_list, flush_node):
  - local_lock_nested_bh(&bq.obj.bulkq.bq_lock).
  - bq_flush_to_queue(bq).
  - local_unlock_nested_bh.
  - wake_up_process(bq.obj.kthread).

REQ-17: cpu_map_generic_redirect(rcpu, skb):
- __skb_pull(skb, skb.mac_len); skb_set_redirected(skb, false).
- __ptr_set_bit(0, &skb) — encodes "SKB, not xdp_frame" in low bit.
- ret = ptr_ring_produce(rcpu.queue, skb).
- if ret >= 0: wake_up_process(rcpu.kthread).
- trace_xdp_cpumap_enqueue(map_id, !ret, !!ret, rcpu.cpu).
- return ret.

REQ-18: cpu_map_kthread_run(data):
- rcpu = data.
- complete(&rcpu.kthread_running).
- set_current_state(TASK_INTERRUPTIBLE).
- while !kthread_should_stop() ∨ !__ptr_ring_empty(rcpu.queue):
  - /* Sleep policy */
  - if __ptr_ring_empty(rcpu.queue): set_current_state(TASK_INTERRUPTIBLE); recheck-empty; schedule()/RUNNING.
  - else: rcu_softirq_qs_periodic; cond_resched.
  - /* Batch consume */
  - n = __ptr_ring_consume_batched(rcpu.queue, frames, CPUMAP_BATCH=8).
  - /* Split SKB-tagged (low-bit) vs xdp_frame */
  - for i in 0..n: if __ptr_test_bit(0, f): skbs[ret.skb_n++] = f & ~1; else frames[ret.xdp_n++] = f; prefetchw(virt_to_page(f)).
  - local_bh_disable.
  - /* Optional secondary XDP prog */
  - cpu_map_bpf_prog_run(rcpu, frames, skbs, &ret, &stats).
  - /* Bulk SKB cache pull */
  - m = napi_skb_cache_get_bulk(skbs, ret.xdp_n); shortfall ⟹ xdp_return_frame remainder.
  - for i in 0..ret.xdp_n: __xdp_build_skb_from_frame(xdpf, skbs[i], xdpf.dev_rx).
  - trace_xdp_cpumap_kthread(map_id, n, kmem_alloc_drops, sched, &stats).
  - for i in 0..ret.xdp_n + ret.skb_n: gro_receive_skb(&rcpu.gro, skbs[i]).
  - /* GRO flush every NAPI_POLL_WEIGHT=64 packets or on empty */
  - packets += n; if packets >= NAPI_POLL_WEIGHT ∨ empty: cpu_map_gro_flush(rcpu, empty); packets = 0.
  - local_bh_enable.
- __set_current_state(TASK_RUNNING); return 0.

REQ-19: cpu_map_bpf_prog_run(rcpu, frames, skbs, ret, stats):
- if !rcpu.prog: skip.
- rcu_read_lock; bpf_net_ctx_set; xdp_set_return_frame_no_direct.
- ret.xdp_n = cpu_map_bpf_prog_run_xdp(rcpu, frames, ret.xdp_n, stats).
- if ret.skb_n: ret.skb_n = cpu_map_bpf_prog_run_skb(rcpu, skbs, ret.skb_n, stats).
- if stats.redirect: xdp_do_flush().
- xdp_clear_return_frame_no_direct; bpf_net_ctx_clear; rcu_read_unlock.
- if ret.skb_n ∧ ret.xdp_n: memmove(&skbs[ret.xdp_n], skbs, ret.skb_n * sizeof(*skbs)).

REQ-20: cpu_map_bpf_prog_run_xdp(rcpu, frames, n, stats):
- xdp.rxq = &(xdp_rxq_info){}.
- for i in 0..n:
  - rxq.dev = xdpf.dev_rx; rxq.mem.type = xdpf.mem_type.
  - xdp_convert_frame_to_buff(xdpf, &xdp).
  - act = bpf_prog_run_xdp(rcpu.prog, &xdp).
  - switch act:
    - XDP_PASS: xdp_update_frame_from_buff → frames[nframes++] (or stats.drop on fail).
    - XDP_REDIRECT: xdp_do_redirect(xdpf.dev_rx, &xdp, rcpu.prog); fail ⟹ stats.drop.
    - default ⟹ bpf_warn_invalid_xdp_action.
    - XDP_ABORTED ⟹ trace_xdp_exception.
    - XDP_DROP ⟹ xdp_return_frame + stats.drop.
- stats.pass += nframes; return nframes.

REQ-21: cpu_map_bpf_prog_run_skb(rcpu, skbs, skb_n, stats):
- for skb in skbs:
  - act = bpf_prog_run_generic_xdp(skb, &xdp, rcpu.prog).
  - XDP_PASS ⟹ skbs[pass++] = skb.
  - XDP_REDIRECT ⟹ xdp_do_generic_redirect; fail ⟹ kfree_skb + stats.drop; else stats.redirect.
  - default/ABORTED ⟹ trace + drop; XDP_DROP ⟹ napi_consume_skb + stats.drop.
- stats.pass += pass; return pass.

REQ-22: cpu_map_gro_flush(rcpu, empty):
- gro_flush_normal(&rcpu.gro, !empty ∧ HZ >= 1000) — full flush iff ring non-empty AND tick < 1ms (else lazy).

REQ-23: __cpu_map_ring_cleanup(ring):
- /* Teardown safety net — ring should already be empty */
- while ptr = ptr_ring_consume(ring): WARN_ON_ONCE(1); if __ptr_test_bit(0, &ptr): __ptr_clear_bit(0, &ptr); kfree_skb(ptr); else xdp_return_frame(ptr).

REQ-24: cpu_map_mem_usage(map):
- usage = sizeof(bpf_cpu_map) + max_entries * sizeof(*entry-ptr).
- (Dynamically-allocated entries not counted.)

## Acceptance Criteria

- [ ] AC-1: cpu_map_alloc rejects max_entries > NR_CPUS with -E2BIG.
- [ ] AC-2: cpu_map_alloc rejects key_size != 4 or value_size not in {qsize-end, bpf_prog.fd-end} with -EINVAL.
- [ ] AC-3: cpu_map_update_elem with qsize > 16384 returns -EOVERFLOW.
- [ ] AC-4: cpu_map_update_elem with !cpu_possible(key_cpu) returns -ENODEV.
- [ ] AC-5: cpu_map_update_elem with qsize == 0 deletes the slot.
- [ ] AC-6: __cpu_map_load_bpf_program rejects prog with expected_attach_type != BPF_XDP_CPUMAP with -EINVAL.
- [ ] AC-7: cpu_map_kthread is bound to its target CPU and runs on it only.
- [ ] AC-8: cpu_map_kthread_run completes &rcpu.kthread_running before returning from entry_alloc.
- [ ] AC-9: bq_enqueue staging up to CPU_MAP_BULK_SIZE=8 then auto-flushes into ptr_ring on overflow.
- [ ] AC-10: __cpu_map_flush drains all bulkqs in flush_list and wakes each rcpu kthread.
- [ ] AC-11: cpu_map_generic_redirect sets low-bit on pointer to encode "SKB"; kthread strips and routes to gro_receive_skb.
- [ ] AC-12: cpu_map_bpf_prog_run_xdp routes XDP_REDIRECT secondary acts through xdp_do_redirect, XDP_DROP frees frame, default emits trace_xdp_exception.
- [ ] AC-13: kthread GRO flush triggers every NAPI_POLL_WEIGHT=64 packets or on empty ring.
- [ ] AC-14: __cpu_map_entry_replace defers free via queue_rcu_work — kthread not stopped until 1 RCU grace period elapses.
- [ ] AC-15: cpu_map_free synchronizes RCU before tearing entries.
- [ ] AC-16: tracepoints trace_xdp_cpumap_enqueue and trace_xdp_cpumap_kthread emitted with map_id, processed, drops.
- [ ] AC-17: cpu_map_redirect rejects all flags (mask = 0): broadcast not supported.

## Architecture

```
struct BpfCpuMap {
  map: BpfMap,
  cpu_map: RcuArray<Option<Arc<BpfCpuMapEntry>>>,   // [max_entries]
}

struct BpfCpuMapEntry {
  cpu: u32,
  map_id: i32,
  bulkq: PerCpu<XdpBulkQueue>,
  queue: Box<PtrRing>,
  kthread: TaskStruct,
  value: BpfCpumapVal,                   // qsize, bpf_prog {fd, id}
  prog: Option<BpfProg>,                  // expected_attach_type == BPF_XDP_CPUMAP
  gro: GroNode,
  kthread_running: Completion,
  free_work: RcuWork,
}

struct XdpBulkQueue {
  q: [PtrTagged; CPU_MAP_BULK_SIZE],      // CPU_MAP_BULK_SIZE = 8
  flush_node: ListHead,
  obj: *BpfCpuMapEntry,
  count: u32,
  bq_lock: LocalLock,
}
```

`CpuMap::alloc(attr) -> BpfMap`:
1. Validate key_size=4, value_size variant, flags ⊆ BPF_F_NUMA_NODE, max_entries ∈ (0, NR_CPUS].
2. Allocate bpf_cpu_map header; init from attr.
3. Allocate cpu_map array (max_entries * Arc<entry>).

`CpuMap::update_elem(map, key, value, flags)`:
1. Validate flags, key_cpu < max_entries, value.qsize ≤ 16384, cpu_possible.
2. If value.qsize == 0 ⟹ entry = None.
3. Else ⟹ entry = entry_alloc(map, value, key_cpu).
4. entry_replace(cmap, key_cpu, entry) under rcu_read_lock.

`CpuMap::entry_alloc(map, value, cpu)`:
1. numa = cpu_to_node(cpu).
2. Allocate BpfCpuMapEntry NUMA-local.
3. Alloc per-CPU bulkq array; init bq_lock + back-ref.
4. Alloc PtrRing of size value.qsize.
5. gro_init(gro).
6. If value.bpf_prog.fd > 0: load_bpf_program (require BPF_XDP_CPUMAP).
7. init_completion(kthread_running).
8. kthread = kthread_create_on_node(cpu_map_kthread_run, ...).
9. kthread_bind(kthread, cpu); wake_up_process.
10. wait_for_completion(kthread_running).

`CpuMap::entry_replace(cmap, key_cpu, new)`:
1. old = xchg(cpu_map[key_cpu], RCU_INITIALIZER(new)).
2. If old: queue_rcu_work(system_percpu_wq, &old.free_work → entry_free).

`CpuMap::entry_free(work)`:
1. kthread_stop(rcpu.kthread) — kthread drains remaining queue before exit.
2. bpf_prog_put(prog) if set.
3. gro_cleanup; ring_cleanup; kfree(queue); free_percpu(bulkq); kfree(rcpu).

`CpuMap::redirect(map, index, flags) -> i64`:
1. return __bpf_xdp_redirect_map(map, index, flags, allowed_flags=0, __cpu_map_lookup_elem).

`CpuMap::enqueue(rcpu, xdpf, dev_rx)`:
1. xdpf.dev_rx = dev_rx.
2. bq_enqueue(rcpu, xdpf).

`CpuMap::bq_enqueue(rcpu, xdpf)`:
1. local_lock_nested_bh(&rcpu.bulkq.bq_lock).
2. bq = this_cpu_ptr(rcpu.bulkq).
3. If bq.count == CPU_MAP_BULK_SIZE: bq_flush(bq) — drain to remote ring.
4. bq.q[bq.count++] = xdpf.
5. If bq not on flush_list: list_add(&bq.flush_node, bpf_net_ctx_get_cpu_map_flush_list()).
6. local_unlock_nested_bh.

`CpuMap::bq_flush(bq)`:
1. Acquire producer_lock on remote ptr_ring.
2. For each staged frame: __ptr_ring_produce → drop+xdp_return on -ENOBUFS.
3. trace_xdp_cpumap_enqueue(map_id, processed, drops, to_cpu).
4. __list_del_clearprev(&bq.flush_node).

`CpuMap::flush(flush_list)`:
1. For each bq in flush_list: lock; bq_flush; unlock; wake_up_process(kthread).

`CpuMap::generic_redirect(rcpu, skb)`:
1. __skb_pull(skb, mac_len); skb_set_redirected(false).
2. __ptr_set_bit(0, &skb) — encode "SKB-flag" via pointer low-bit (alignment ≥ 2 guaranteed).
3. ptr_ring_produce(rcpu.queue, tagged_skb).
4. wake_up_process(rcpu.kthread).
5. trace_xdp_cpumap_enqueue.

`CpuMap::kthread_run(data) -> i32`:
1. complete(kthread_running).
2. Loop while !kthread_should_stop() ∨ !ring_empty:
   - If empty ⟹ TASK_INTERRUPTIBLE + schedule (recheck post-set).
   - Else ⟹ cond_resched + softirq_qs.
   - n = ptr_ring_consume_batched(CPUMAP_BATCH=8).
   - Split tagged-SKBs from xdp_frames; prefetchw(virt_to_page).
   - local_bh_disable.
   - If rcpu.prog: bpf_prog_run on batch (XDP_PASS/REDIRECT/DROP).
   - bulk-pull SKB cache via napi_skb_cache_get_bulk.
   - Build SKBs from xdp_frames via __xdp_build_skb_from_frame.
   - trace_xdp_cpumap_kthread.
   - gro_receive_skb on each SKB.
   - GRO flush every NAPI_POLL_WEIGHT=64 packets OR on empty ring.
   - local_bh_enable.
3. Return 0.

`CpuMap::free(map)`:
1. synchronize_rcu — XDP-side flushes done.
2. For i in 0..max_entries: entry_free direct (no need for RCU work — refcounts are 0).
3. Free header + array.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `max_entries_le_nr_cpus` | INVARIANT | per-alloc: max_entries ≤ NR_CPUS. |
| `key_cpu_possible` | INVARIANT | per-update: cpu_possible(key_cpu) required. |
| `qsize_bounded` | INVARIANT | per-update: qsize ≤ 16384. |
| `bulkq_count_bounded` | INVARIANT | per-bq: 0 ≤ count ≤ CPU_MAP_BULK_SIZE. |
| `kthread_bound_to_cpu` | INVARIANT | per-entry: kthread_bind(cpu) prior to wake. |
| `kthread_started_before_returning` | INVARIANT | per-entry_alloc: wait_for_completion(kthread_running). |
| `secondary_prog_type_cpumap` | INVARIANT | per-load_bpf_program: prog.expected_attach_type == BPF_XDP_CPUMAP. |
| `xchg_then_rcu_work_for_free` | INVARIANT | per-entry_replace: old entry freed via queue_rcu_work, not direct kfree. |
| `ptr_ring_low_bit_skb_tag` | INVARIANT | per-generic_redirect: low-bit set on SKB; cleared in kthread. |
| `flush_does_not_lose_frames` | INVARIANT | per-bq_flush: enqueue-failures freed via xdp_return_frame_rx_napi. |

### Layer 2: TLA+

`kernel/bpf/cpumap.tla`:
- Per-XDP-redirect + per-bulkq-stage + per-flush + per-ptr_ring-consume + per-kthread.
- Properties:
  - `safety_no_concurrent_consumer` — per-rcpu: at most one kthread consuming ptr_ring (single-consumer rule).
  - `safety_no_use_after_replace` — per-XDP: post-xchg, RCU grace before kthread_stop.
  - `safety_bulkq_per_cpu` — per-CPU: bulkq access protected by local_lock_nested_bh.
  - `liveness_eventually_flush` — per-frame enqueued: eventually drained on xdp_do_flush().
  - `liveness_kthread_drains_on_stop` — per-stop: ring drains before kthread exit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CpuMap::alloc` post: max_entries ≤ NR_CPUS ∧ cpu_map array allocated | `CpuMap::alloc` |
| `CpuMap::update_elem` post: cpu_possible(key) ∧ qsize ≤ 16384 | `CpuMap::update_elem` |
| `CpuMap::entry_alloc` post: kthread bound to cpu ∧ completion signalled | `CpuMap::entry_alloc` |
| `CpuMap::bq_enqueue` post: count ≤ CPU_MAP_BULK_SIZE (auto-flush on full) | `CpuMap::bq_enqueue` |
| `CpuMap::bq_flush` post: bq.count == 0 ∧ bq off flush_list | `CpuMap::bq_flush` |
| `CpuMap::kthread_run` post: ring drained before return on kthread_stop | `CpuMap::kthread_run` |
| `CpuMap::entry_free` post: kthread joined, prog put, ring + percpu + entry freed | `CpuMap::entry_free` |
| `CpuMap::redirect` post: rejects all flags (allowed = 0) | `CpuMap::redirect` |

### Layer 4: Verus/Creusot functional

`Per-NAPI XDP_REDIRECT → bq_enqueue → xdp_do_flush → __cpu_map_flush → ptr_ring producer_lock commit → wake_up_process(kthread) → kthread consume-batch → optional secondary BPF_XDP_CPUMAP prog → __xdp_build_skb_from_frame → gro_receive_skb → gro_flush` semantic equivalence: per-Documentation/networking/xdp-rx-metadata.rst + `tools/testing/selftests/bpf/prog_tests/xdp_cpumap_attach.c`.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

CPUMap reinforcement:

- **Per-max_entries ≤ NR_CPUS** — defense against per-array-OOB on possibly-malicious user.
- **Per-cpu_possible() check on update** — defense against per-non-existent target CPU.
- **Per-qsize ≤ 16384 sanity** — defense against per-ring-OOM exhaustion.
- **Per-key_size == 4 strict** — defense against per-malformed map.
- **Per-secondary-prog expected_attach_type == BPF_XDP_CPUMAP** — defense against per-wrong-prog-type attach.
- **Per-kthread kthread_bind to target CPU** — defense against per-cross-CPU pollution.
- **Per-wait_for_completion before returning from entry_alloc** — defense against per-kthread_stop race losing frames.
- **Per-RCU + queue_rcu_work for entry replace** — defense against per-UAF on in-flight frames.
- **Per-PREEMPT_RT local_lock_nested_bh on bulkq** — defense against per-bulkq corruption.
- **Per-single-consumer kthread on ptr_ring** — defense against per-double-dequeue.
- **Per-low-bit pointer-tag SKB encoding (alignment ≥ 2)** — defense against per-pointer-type confusion.
- **Per-XDP secondary prog under rcu_read_lock + bpf_net_ctx_set** — defense against per-prog-UAF.
- **Per-broadcast/exclude-ingress flags disallowed (allowed_mask = 0)** — defense against per-unsupported-fan-out.
- **Per-trace_xdp_cpumap_{enqueue,kthread} rate-limited via tracepoint infra** — defense against per-log-flood.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- include/net/xdp.h `xdp_do_redirect` / `xdp_do_flush` core (covered in `net/xdp.md` Tier-3)
- kernel/bpf/devmap.c (covered in `devmap.md` Tier-3)
- kernel/bpf/cpumask.c (separate map type)
- include/linux/ptr_ring.h primitive (covered in `linux-primitives.md` if expanded)
- net/core/gro.c GRO engine (covered in `gro.md` Tier-3 if expanded)
- Implementation code
