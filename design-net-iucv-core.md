---
title: "Tier-3: net/iucv/iucv.c — IUCV (Inter-User Communications Vehicle) base infrastructure"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

IUCV (Inter-User Communications Vehicle) is the IBM z/VM hypervisor-mediated message-passing bus that lets two guest userids on the same z/VM host exchange messages without going through TCP/IP. `net/iucv/iucv.c` is the s390-only base layer: a kernel driver that registers with the s390 external-interrupt subsystem (`EXT_IRQ_IUCV`), declares per-CPU IUCV interrupt buffers via the CP `B2F0` instruction, and dispatches incoming IUCV events to registered handlers (af_iucv, hvc_iucv, smsgiucv, netiucv). Per-IUCV operation is issued by populating a per-CPU `union iucv_param` and invoking the privileged `B2F0` instruction with the command number in r0 — `IUCV_DECLARE_BUFFER`, `IUCV_CONNECT`, `IUCV_ACCEPT`, `IUCV_SEND`, `IUCV_RECEIVE`, `IUCV_REPLY`, `IUCV_REJECT`, `IUCV_PURGE`, `IUCV_QUIESCE`, `IUCV_RESUME`, `IUCV_SEVER`, `IUCV_SETMASK`, `IUCV_SETCONTROLMASK`, `IUCV_QUERY`, `IUCV_RETRIEVE_BUFFER`. Per-path is identified by a 16-bit `pathid` indexed into `iucv_path_table[]`. Per-incoming-interrupt is parked on either `iucv_task_queue` (delivered via tasklet) or `iucv_work_queue` (delivered via workqueue for `path_pending` which may sleep). Per-handler registers via `iucv_register(handler, smp)` and receives callbacks `path_pending` / `path_complete` / `path_severed` / `path_quiesced` / `path_resumed` / `message_pending` / `message_complete`. Critical for: z/VM guest-to-guest comms without a network, console-over-IUCV (hvc_iucv), 3270-emulator paths, classic IBM-mainframe networking.

This Tier-3 covers `net/iucv/iucv.c` (~1950 lines).

### Acceptance Criteria

- [ ] AC-1: Boot on z/VM guest: iucv_init succeeds; iucv_available = 1; `iucv_max_pathid` populated.
- [ ] AC-2: Boot on bare-metal / KVM (non-VM): iucv_init returns -EPROTONOSUPPORT; iucv_available = 0; iucv_register returns -ENOSYS.
- [ ] AC-3: First iucv_register(handler, smp=1): iucv_enable runs; iucv_path_table allocated; at least one cpu in iucv_buffer_cpumask.
- [ ] AC-4: Second iucv_register(handler, smp=0): no re-enable; iucv_setmask_up called (single-cpu delivery).
- [ ] AC-5: iucv_path_connect to non-existent userid: B2F0 rc != 0; -EIO or iprcode propagated; path not in table.
- [ ] AC-6: iucv_path_connect success → IRQ-type 0x02 path_complete → handler->path_complete invoked.
- [ ] AC-7: Incoming IRQ-type 0x01 path_pending → workqueue dispatch → handler chain walked → first acceptor wins.
- [ ] AC-8: Path-pending with no matching handler: iucv_sever_pathid("NO LISTENER" EBCDIC); table slot cleared.
- [ ] AC-9: iucv_message_send with IUCV_IPRMDATA + size <= 8: data placed in dpl.iprmmsg; receiver gets msg.flags & IUCV_IPRMDATA.
- [ ] AC-10: iucv_message_send without IPRMDATA: db.ipbfadr1 = virt_to_dma32(buffer); receiver gets msg.length = size.
- [ ] AC-11: iucv_message_receive with IPRMDATA flag: copies 8 bytes from msg.rmmsg into user buffer (no B2F0 call).
- [ ] AC-12: iucv_path_sever: B2F0 IUCV_SEVER; iucv_path_table[pathid] = NULL; list_del_init.
- [ ] AC-13: iucv_unregister with active paths: each severed; table cleared; handler removed.
- [ ] AC-14: CPU offline of last IUCV cpu: down_prep returns -EINVAL.
- [ ] AC-15: Reboot notifier: severs all paths, blocks all IRQs.
- [ ] AC-16: External interrupt with ippathid >= iucv_max_pathid: WARN + sever-with-pathid-error.
- [ ] AC-17: External interrupt with iptype outside [0x01, 0x09]: BUG.
- [ ] AC-18: iucv_path_quiesce + iucv_path_resume: incoming messages on path suspended then resumed.
- [ ] AC-19: iucv_message_send2way + receiver iucv_message_reply: reply moved into sender's answer buffer.
- [ ] AC-20: iucv_message_purge of sent message: B2F0 IUCV_PURGE; audit + tag returned.

### Architecture

```
struct IucvIrqData {        // declared interrupt buffer (per cpu)
  ippathid: u16,
  ipflags1: u8,
  iptype: u8,               // 0x01..0x09 event code
  res2: [u32; 9],
}

struct IucvPath {
  pathid: u16,
  msglim: u16,
  flags: u8,
  handler: Option<*IucvHandler>,
  private: *mut (),
  list: ListHead,           // member of handler->paths
}

struct IucvHandler {
  list: ListHead,           // member of iucv_handler_list
  paths: ListHead,          // anchor of owned IucvPath
  path_pending:  Option<fn(&mut IucvPath, vmid: &[u8;8], userdata: &[u8;16]) -> i32>,
  path_complete: Option<fn(&mut IucvPath, userdata: &[u8;16])>,
  path_severed:  Option<fn(&mut IucvPath, userdata: &[u8;16])>,
  path_quiesced: Option<fn(&mut IucvPath, userdata: &[u8;16])>,
  path_resumed:  Option<fn(&mut IucvPath, userdata: &[u8;16])>,
  message_pending:  Option<fn(&mut IucvPath, &IucvMessage)>,
  message_complete: Option<fn(&mut IucvPath, &IucvMessage)>,
}

union IucvParam {           // 8-byte aligned, packed
  ctrl: IucvCmdControl,
  dpl:  IucvCmdDpl,
  db:   IucvCmdDb,
  purge: IucvCmdPurge,
  set_mask: IucvCmdSetMask,
}

struct Iucv {
  bus: BusType,
  root: *Device,
  available: AtomicBool,
  max_pathid: usize,
  path_table: Vec<Option<Box<IucvPath>>>,   // length = max_pathid
  handler_list: List<IucvHandler>,
  table_lock: Spinlock,
  queue_lock: Spinlock,
  register_mutex: Mutex,
  nonsmp_handler: i32,
  active_cpu: i32,                           // sentinel: tasklet/work owner
  buffer_cpumask: CpuMask,
  irq_cpumask: CpuMask,
  task_queue: List<IucvIrqList>,
  work_queue: List<IucvIrqList>,
  tasklet: Tasklet,
  work: WorkStruct,
  reboot_notifier: NotifierBlock,
  online_state: CpuHpState,
  irq_buffers: [*mut IucvIrqData; NR_CPUS],  // per-cpu declared buffers
  params: [*mut IucvParam; NR_CPUS],         // process/softirq context
  params_irq: [*mut IucvParam; NR_CPUS],     // IRQ context
}
```

`Iucv::call_b2f0(cmd, parm) -> i32` (REQ-2):
1. asm: r0 = cmd, r1 = virt_to_phys(parm), .long 0xb2f01000, read cc.
2. if cc == 1: return parm.ctrl.iprcode (IUCV-defined error).
3. else: return cc (0 success; 2/3 host error).

`Iucv::init() -> Result<(), Errno>` (REQ-1):
1. require machine_is_vm() else -EPROTONOSUPPORT.
2. system_ctl_set_bit(0, CR0_IUCV_BIT).
3. iucv_query_maxconn.
4. register_external_irq(EXT_IRQ_IUCV, external_interrupt).
5. iucv_root = root_device_register("iucv").
6. cpuhp_setup_state(CPUHP_NET_IUCV_PREPARE, prepare, dead).
7. cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, online, down_prep) → iucv_online.
8. register_reboot_notifier.
9. ASCEBC sever-error strings.
10. iucv_available = 1.
11. bus_register(&iucv_bus); iucv_if.root/bus populated.

`Iucv::enable() -> Result<(), Errno>` (REQ-6):
1. cpus_read_lock.
2. iucv_path_table = kzalloc(iucv_max_pathid * sizeof(*)).
3. For each online cpu: smp_call_function_single(cpu, declare_cpu).
4. require cpumask_buffer non-empty.
5. cpus_read_unlock; return 0.

`Iucv::register_handler(handler, smp) -> Result<(), Errno>` (REQ-9):
1. !available → -ENOSYS.
2. mutex_lock(register_mutex).
3. !smp → nonsmp_handler++.
4. handler_list empty → enable.
   else !smp ∧ nonsmp_handler == 1 → setmask_up.
5. INIT_LIST_HEAD(&handler.paths).
6. spin_lock_bh(table_lock); list_add_tail(&handler.list, handler_list); unlock.
7. return 0.

`Iucv::unregister_handler(handler, smp)` (REQ-10):
1. mutex_lock.
2. spin_lock_bh(table_lock).
3. list_del_init(&handler.list).
4. For each path in handler.paths:
   - iucv_sever_pathid(p.pathid, NULL).
   - iucv_path_table[p.pathid] = NULL.
   - list_del(&p.list); iucv_path_free(p).
5. spin_unlock_bh.
6. !smp → nonsmp_handler--.
7. handler_list empty → disable.
   else !smp ∧ nonsmp_handler == 0 → setmask_mp.
8. mutex_unlock.

`IucvPath::connect(path, handler, userid, system, userdata, private) -> Result<(), Errno>` (REQ-11):
1. spin_lock_bh(table_lock).
2. cleanup_queue (purge stale tasks for reused pathids).
3. cpumask_buffer empty → -EIO.
4. parm = iucv_param[smp_processor_id()]; memset.
5. populate ctrl (ipmsglim, ipflags1, ipvmid w/ ASCEBC+TOUPPER, iptarget w/ ASCEBC+TOUPPER, ipuser).
6. iucv_call_b2f0(IUCV_CONNECT).
7. on rc == 0: validate ippathid < max_pathid; populate path; iucv_path_table[ippathid] = path; list_add_tail(&path.list, &handler.paths).
8. on invalid ippathid from CP: sever; return -EIO.

`IucvPath::sever(path, userdata) -> Result<(), Errno>` (REQ-13):
1. preempt_disable.
2. cpumask_buffer empty → -EIO.
3. if active_cpu != smp_processor_id(): spin_lock_bh(table_lock) /* else: tasklet holds it */.
4. rc = iucv_sever_pathid(path.pathid, userdata).
5. iucv_path_table[path.pathid] = None.
6. list_del_init(&path.list).
7. drop lock if taken; preempt_enable.

`IucvMessage::send / send_nolock` (REQ-14):
1. cpumask_buffer empty → -EIO.
2. parm = iucv_param[cpu]; memset.
3. if flags & IUCV_IPRMDATA: dpl variant with iprmmsg[0..8] = buffer.
   else: db variant with ipbfadr1 = virt_to_dma32(buffer), ipbfln1f = size.
4. set ipflags1 |= IUCV_IPNORPY (one-way).
5. iucv_call_b2f0(IUCV_SEND).
6. on success: msg.id = parm.db.ipmsgid.

`IucvMessage::receive` (REQ-15):
1. if msg.flags & IUCV_IPRMDATA: receive_iprmdata (handle scatter-gather IUCV_IPBUFLST: walk iucv_array, copy up to 8 bytes).
2. else: B2F0 IUCV_RECEIVE.
3. on rc in {0, 5}: copy back ipflags1; residual.

`Iucv::external_interrupt(ext_code, param32, param64)` (REQ-17):
1. inc_irq_stat(IRQEXT_IUC).
2. p = iucv_irq_data[smp_processor_id()].
3. validate p.ippathid; out-of-range → WARN + sever + return.
4. BUG_ON iptype out of [0x01, 0x09].
5. work = kmalloc(GFP_ATOMIC); memcpy.
6. spin_lock(queue_lock).
7. iptype == 0x01 → enqueue work_queue + schedule_work.
   else → enqueue task_queue + tasklet_schedule.
8. spin_unlock(queue_lock).

`Iucv::tasklet_fn()` (REQ-18):
1. spin_trylock(table_lock) — on contention, tasklet_schedule self.
2. active_cpu = smp_processor_id().
3. splice task_queue under queue_lock.
4. For each entry: dispatch by iptype via dispatch table (0x02..0x09).
5. kfree entries.
6. active_cpu = -1; spin_unlock(table_lock).

`Iucv::work_fn()` (REQ-19):
1. spin_lock_bh(table_lock); active_cpu = cpu.
2. splice work_queue under queue_lock.
3. iucv_cleanup_queue.
4. For each: iucv_path_pending; kfree.
5. active_cpu = -1; spin_unlock_bh.

`Iucv::handle_path_pending(data)` (REQ-20):
1. BUG_ON(path_table[ippathid].is_some()).
2. path = IucvPath::alloc(ipmsglim, ipflags1, GFP_ATOMIC).
3. path_table[ippathid] = Some(path); EBCASC ipvmid.
4. For each handler with path_pending:
   - list_add(&path.list, &handler.paths); path.handler = handler.
   - rc = handler.path_pending(path, ipvmid, ipuser).
   - rc == 0 → accepted; return.
   - rc != 0 → list_del; clear handler; try next.
5. None accepted → path_table[pathid] = None; iucv_path_free; sever w/ "NO LISTENER".

Dispatch table (iucv_tasklet_fn `irq_fn[]`):
- 0x02 path_complete
- 0x03 path_severed
- 0x04 path_quiesced
- 0x05 path_resumed
- 0x06 message_complete (nonprio)
- 0x07 message_complete (prio)
- 0x08 message_pending (nonprio)
- 0x09 message_pending (prio)

Lock hierarchy:
```
iucv_register_mutex
   └─ iucv_table_lock (spinlock, bh-safe)
         └─ iucv_queue_lock (spinlock)
```
ISR runs without these; defers to tasklet (table_lock) or workqueue (table_lock_bh).
`iucv_active_cpu` provides a non-locking sentinel so handlers re-entering `iucv_path_sever` from within a tasklet do not double-lock.

### Out of Scope

- `net/iucv/af_iucv.c` (AF_IUCV socket family) — covered separately if expanded
- `drivers/s390/net/netiucv.c` (NETIUCV CTC-like device driver) — covered in `drivers/s390/net.md`
- `drivers/s390/char/sclp_*.c` / `drivers/tty/hvc/hvc_iucv.c` (console over IUCV) — covered separately
- `drivers/s390/char/vmlogrdr.c` / `vmcp.c` (other CP-call interfaces) — covered in `drivers/s390/char.md`
- s390 external-interrupt subsystem (`arch/s390/kernel/irq.c`, `register_external_irq`) — covered in `arch/s390/irq.md`
- s390 CPU control-register manipulation (`system_ctl_set_bit`, `CR0_IUCV_BIT`) — covered in `arch/s390/system.md`
- EBCDIC conversion primitives (`ASCEBC`, `EBCASC`, `EBC_TOUPPER`) — covered in `arch/s390/ebcdic.md`
- z/VM CP Programming Service semantics (SC24-5760) — external spec, referenced not duplicated
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iucv_handler` | per-driver callback set (path_pending / _complete / _severed / _quiesced / _resumed / message_pending / message_complete) | shared `IucvHandler` |
| `struct iucv_path` | per-path (pathid, msglim, flags, handler, private, list) | `IucvPath` |
| `struct iucv_message` | per-message (id, audit, class, tag, length, flags, reply_size, rmmsg[8]) | `IucvMessage` |
| `struct iucv_irq_data` | per-irq buffer header (ippathid, ipflags1, iptype) | `IucvIrqData` |
| `struct iucv_irq_list` | per-queued-irq entry | `IucvIrqList` |
| `struct iucv_cmd_control` | per-CONNECT/ACCEPT/QUIESCE/RESUME/SEVER param | `IucvCmdControl` |
| `struct iucv_cmd_dpl` | per-SEND/REPLY data-in-param-list (IPRMDATA) | `IucvCmdDpl` |
| `struct iucv_cmd_db` | per-RECEIVE/SEND/REJECT/declare data-buffer param | `IucvCmdDb` |
| `struct iucv_cmd_purge` | per-PURGE param | `IucvCmdPurge` |
| `struct iucv_cmd_set_mask` | per-SETMASK / SETCONTROLMASK param | `IucvCmdSetMask` |
| `union iucv_param` | per-CPU param block (8-byte aligned, DMA32) | `IucvParam` |
| `iucv_bus` | per-IUCV `struct bus_type` for sysfs/devicemodel | `Iucv::bus` |
| `iucv_root` | per-IUCV root device | `Iucv::root` |
| `iucv_if` | per-IUCV exported interface struct | `Iucv::interface` |
| `__iucv_call_b2f0()` | per-B2F0 instruction primitive | `Iucv::call_b2f0_raw` |
| `iucv_call_b2f0()` | per-B2F0 + iprcode unwrap | `Iucv::call_b2f0` |
| `iucv_query_maxconn()` | per-IUCV_QUERY (max-pathid) | `Iucv::query_maxconn` |
| `iucv_declare_cpu()` | per-CPU IUCV_DECLARE_BUFFER | `Iucv::declare_cpu` |
| `iucv_retrieve_cpu()` | per-CPU IUCV_RETRIEVE_BUFFER | `Iucv::retrieve_cpu` |
| `iucv_allow_cpu()` / `_block_cpu()` | per-CPU IUCV_SETMASK / SETCONTROLMASK | `Iucv::allow_cpu` / `block_cpu` |
| `iucv_setmask_mp()` / `_up()` | per-allowed-cpus expansion / restriction | `Iucv::setmask_mp` / `setmask_up` |
| `iucv_enable()` / `_disable()` | per-first-handler bring-up / per-last-handler tear-down | `Iucv::enable` / `disable` |
| `iucv_register()` / `_unregister()` | per-handler lifetime | `Iucv::register_handler` / `unregister_handler` |
| `iucv_path_alloc()` / `_free()` | per-path allocator (`net/iucv/iucv.h`) | `IucvPath::alloc` / `free` |
| `iucv_path_accept()` | per-IUCV_ACCEPT | `IucvPath::accept` |
| `iucv_path_connect()` | per-IUCV_CONNECT (with userid + system + userdata + EBCDIC convert) | `IucvPath::connect` |
| `iucv_path_quiesce()` | per-IUCV_QUIESCE | `IucvPath::quiesce` |
| `iucv_path_resume()` | per-IUCV_RESUME | `IucvPath::resume` |
| `iucv_path_sever()` | per-IUCV_SEVER (release pathid) | `IucvPath::sever` |
| `iucv_sever_pathid()` | per-internal severance | `Iucv::sever_pathid` |
| `iucv_message_purge()` | per-IUCV_PURGE | `IucvMessage::purge` |
| `iucv_message_receive()` / `__iucv_message_receive()` | per-IUCV_RECEIVE (locked / unlocked) | `IucvMessage::receive` / `receive_nolock` |
| `iucv_message_receive_iprmdata()` | per-RMDATA-in-message-desc receive | `IucvMessage::receive_iprmdata` |
| `iucv_message_reject()` | per-IUCV_REJECT | `IucvMessage::reject` |
| `iucv_message_reply()` | per-IUCV_REPLY (two-way reply) | `IucvMessage::reply` |
| `iucv_message_send()` / `__iucv_message_send()` | per-IUCV_SEND one-way (locked / unlocked) | `IucvMessage::send` / `send_nolock` |
| `iucv_message_send2way()` | per-IUCV_SEND two-way | `IucvMessage::send2way` |
| `iucv_path_pending()` | per-pathid IRQ-type 0x01 (incoming-connect) handler | `Iucv::handle_path_pending` |
| `iucv_path_complete()` | per-IRQ-type 0x02 (connect-complete) | `Iucv::handle_path_complete` |
| `iucv_path_severed()` | per-IRQ-type 0x03 | `Iucv::handle_path_severed` |
| `iucv_path_quiesced()` | per-IRQ-type 0x04 | `Iucv::handle_path_quiesced` |
| `iucv_path_resumed()` | per-IRQ-type 0x05 | `Iucv::handle_path_resumed` |
| `iucv_message_complete()` | per-IRQ-type 0x06 / 0x07 | `Iucv::handle_message_complete` |
| `iucv_message_pending()` | per-IRQ-type 0x08 / 0x09 | `Iucv::handle_message_pending` |
| `iucv_external_interrupt()` | per-EXT_IRQ_IUCV ISR | `Iucv::external_interrupt` |
| `iucv_tasklet_fn()` | per-bottom-half tasklet for non-pending IRQs | `Iucv::tasklet_fn` |
| `iucv_work_fn()` | per-workqueue path_pending dispatcher | `Iucv::work_fn` |
| `iucv_cleanup_queue()` | per-sever cleanup of stale tasks | `Iucv::cleanup_queue` |
| `iucv_init()` / `_exit()` | per-module load / unload | `Iucv::init` / `exit` |
| `iucv_reboot_event()` | per-reboot sever-all hook | `Iucv::reboot_event` |
| `iucv_cpu_prepare()` / `_dead()` / `_online()` / `_down_prep()` | per-CPU hotplug callbacks | `Iucv::cpu_*` |
| `iucv_alloc_device()` | per-IUCV device factory | `Iucv::alloc_device` |
| `iucv_path_table[]` / `iucv_max_pathid` | per-pathid registry | `Iucv::path_table` |
| `iucv_handler_list` | per-driver registry | `Iucv::handler_list` |
| `iucv_buffer_cpumask` / `iucv_irq_cpumask` | per-CPU enablement state | `Iucv::cpumasks` |
| `iucv_param[NR_CPUS]` / `iucv_param_irq[NR_CPUS]` | per-CPU B2F0 param blocks (one for syscall-context, one for IRQ-context) | `Iucv::params` |
| `iucv_irq_data[NR_CPUS]` | per-CPU declared interrupt buffer | `Iucv::irq_buffers` |
| `iucv_active_cpu` | per-tasklet/work owning-CPU sentinel | `Iucv::active_cpu` |
| `iucv_table_lock` | per-handler-list + path-table spinlock | `Iucv::table_lock` |
| `iucv_queue_lock` | per-irq-queue spinlock | `Iucv::queue_lock` |
| `iucv_register_mutex` | per-register/unregister mutex | `Iucv::register_mutex` |
| `iucv_nonsmp_handler` | per-non-SMP-capable-handler count | `Iucv::nonsmp_handler` |
| `iucv_available` | per-init success flag | `Iucv::available` |
| `iucv_error_no_listener` / `_no_memory` / `_pathid` | per-EBCDIC sever-reason strings | shared |

### compatibility contract

REQ-1: Hardware / hypervisor prerequisites:
- `machine_is_vm()` must be true at `iucv_init` time. Otherwise return -EPROTONOSUPPORT.
- Set CR0 bit `CR0_IUCV_BIT` to enable IUCV externals.
- Register `EXT_IRQ_IUCV` external-interrupt handler.

REQ-2: B2F0 instruction primitive:
- `__iucv_call_b2f0(command, parm)`: load r0 = command, r1 = phys(parm), execute `.long 0xb2f01000`, read condition-code via IPM/SRL.
- `iucv_call_b2f0`: return `cc == 1 ? parm.ctrl.iprcode : cc` (cc=0 success; cc=1 IUCV-specific error in iprcode; cc=2/3 host-level error).
- All params must be in 31-bit-addressable storage (GFP_DMA) and 8-byte aligned.

REQ-3: union iucv_param layout (8-byte aligned, packed):
- ctrl: CONNECT/ACCEPT/QUIESCE/RESUME/SEVER (ippathid, ipflags1, iprcode, ipmsglim, ipvmid[8], ipuser[16], iptarget[8]).
- dpl:  SEND/REPLY data-in-param-list (ippathid, ipflags1, iprcode, ipmsgid, iptrgcls, iprmmsg[8], ipsrccls, ipmsgtag, ipbfadr2, ipbfln2f).
- db:   SEND/RECEIVE/REJECT/DECLARE data-buffer (ippathid, ipflags1, iprcode, ipmsgid, iptrgcls, ipbfadr1, ipbfln1f, ipsrccls, ipmsgtag, ipbfadr2, ipbfln2f).
- purge:PURGE (ippathid, ipmsgid, ipaudit[3], ipsrccls, ipmsgtag).
- set_mask: SETMASK / SETCONTROLMASK (ipmask).

REQ-4: Per-CPU state (allocated in `iucv_cpu_prepare`, freed in `iucv_cpu_dead`):
- `iucv_irq_data[cpu]`: declared interrupt buffer (GFP_DMA on cpu_to_node).
- `iucv_param[cpu]`: B2F0 param for process / softirq context.
- `iucv_param_irq[cpu]`: B2F0 param for hard-IRQ context.
- All zeroed via `memset(parm, 0, sizeof(union iucv_param))` before each call.

REQ-5: iucv_query_maxconn():
- B2F0 cmd = IUCV_QUERY; result max_pathid in r1.
- Stores `iucv_max_pathid` (used to size `iucv_path_table`).

REQ-6: iucv_enable() / _disable():
- enable: cpus_read_lock; allocate `iucv_path_table` of `iucv_max_pathid` slots; for each online cpu call `iucv_declare_cpu`; require at least one cpu in `iucv_buffer_cpumask`.
- disable: on_each_cpu(`iucv_retrieve_cpu`); kfree(iucv_path_table); NULL it.

REQ-7: iucv_declare_cpu(): per-cpu IUCV_DECLARE_BUFFER:
- Skip if cpu already in `iucv_buffer_cpumask`.
- Param: db.ipbfadr1 = virt_to_dma32(iucv_irq_data[cpu]); cmd = IUCV_DECLARE_BUFFER.
- Map rc codes to log-strings: 0x03 Directory error / 0x0a Invalid length / 0x13 Buffer already exists / 0x3e Buffer overlap / 0x5c Paging or storage error.
- On success: cpumask_set_cpu(cpu, &iucv_buffer_cpumask).
- If `iucv_nonsmp_handler == 0 ∨ cpumask_empty(&iucv_irq_cpumask)`: allow this cpu; else block this cpu (single-CPU pinning for non-SMP-capable handlers).

REQ-8: iucv_allow_cpu() / _block_cpu():
- allow: set ipmask = 0xf8 for both SETMASK (message interrupts: 0x80 nonprio pending / 0x40 prio pending / 0x20 nonprio complete / 0x10 prio complete / 0x08 control) and SETCONTROLMASK (control: 0x80 path pending / 0x40 connect complete / 0x20 connection severed / 0x10 connection quiesced / 0x08 connection resumed).
- block: ipmask = 0 (SETMASK only) — control mask retained per design; allow-cpu sets both.

REQ-9: iucv_register(handler, smp):
- if !iucv_available: -ENOSYS.
- Acquire `iucv_register_mutex`.
- if !smp: iucv_nonsmp_handler++.
- if list_empty(&iucv_handler_list): iucv_enable() (allocate table + declare buffers).
- else if !smp ∧ iucv_nonsmp_handler == 1: iucv_setmask_up (single-cpu deliver).
- INIT_LIST_HEAD(&handler->paths); list_add_tail(&handler->list, &iucv_handler_list) under `iucv_table_lock`.
- return 0.

REQ-10: iucv_unregister(handler, smp):
- Acquire `iucv_register_mutex` + `iucv_table_lock`.
- list_del_init(&handler->list).
- For each path in handler->paths: iucv_sever_pathid(p.pathid, NULL); iucv_path_table[p.pathid] = NULL; list_del; iucv_path_free.
- Drop locks.
- if !smp: iucv_nonsmp_handler--.
- if list_empty(&iucv_handler_list): iucv_disable.
- else if !smp ∧ iucv_nonsmp_handler == 0: iucv_setmask_mp (re-enable all cpus).

REQ-11: iucv_path_connect(path, handler, userid, system, userdata, private):
- Acquire `iucv_table_lock` (bh).
- iucv_cleanup_queue (remove stale work for reused pathids).
- if cpumask_empty(&iucv_buffer_cpumask): -EIO.
- Build ctrl param: ipmsglim, ipflags1, ipvmid (8 bytes, ASCII→EBCDIC, uppercase), iptarget (8 bytes, ASCII→EBCDIC, uppercase), ipuser (16 bytes).
- iucv_call_b2f0(IUCV_CONNECT).
- On success: validate parm.ctrl.ippathid < iucv_max_pathid; populate path; iucv_path_table[path.pathid] = path; list_add_tail to handler->paths.
- On invalid pathid: iucv_sever_pathid(...); return -EIO.

REQ-12: iucv_path_accept(path, handler, userdata, private):
- local_bh_disable.
- if cpumask_empty(&iucv_buffer_cpumask): -EIO.
- ctrl param: ippathid, ipmsglim, ipuser, ipflags1.
- iucv_call_b2f0(IUCV_ACCEPT).
- On success: copy back msglim + flags; set private.

REQ-13: iucv_path_quiesce / _resume / _sever:
- quiesce: IUCV_QUIESCE — temporarily stop incoming messages on path.
- resume: IUCV_RESUME — resume after quiesce.
- sever: IUCV_SEVER — terminate path; `iucv_path_table[pathid] = NULL`; list_del_init.
- All: validate cpumask not empty; populate ippathid + (optional) ipuser; call B2F0.
- sever wraps internal `iucv_sever_pathid` and adjusts table+list under `iucv_table_lock` unless invoked from a tasklet (where `iucv_active_cpu == smp_processor_id()` already implies the lock is held).

REQ-14: iucv_message_send / send2way / reply (one-way / two-way / reply):
- send: IUCV_SEND with `IUCV_IPNORPY` flag set (no reply expected).
- send2way: IUCV_SEND without IPNORPY; provides answer buffer (ipbfadr2 / ipbfln2f).
- reply: IUCV_REPLY against received two-way message.
- All accept IUCV_IPRMDATA (8-byte message-in-param-list), IUCV_IPBUFLST (iucv_array scatter-gather), IUCV_IPPRTY (priority).
- On success: msg.id = parm.db.ipmsgid.

REQ-15: iucv_message_receive / __iucv_message_receive:
- If msg.flags & IUCV_IPRMDATA: `iucv_message_receive_iprmdata` copies the 8 bytes from msg.rmmsg into caller buffer or scatter-gather iucv_array.
- Else: B2F0 IUCV_RECEIVE with ipbfadr1 = virt_to_dma32(buffer), ipbfln1f = size, ipflags1 = flags | IUCV_IPFGPID | IUCV_IPFGMID | IUCV_IPTRGCLS.
- On rc == 0 ∨ rc == 5 (partial): copy back ipflags1 and residual.

REQ-16: iucv_message_purge / _reject:
- purge: IUCV_PURGE — cancel a sent message awaiting reply; returns audit + tag.
- reject: IUCV_REJECT — refuse a received message before completion.

REQ-17: iucv_external_interrupt(ext_code, param32, param64) — ISR:
- inc_irq_stat(IRQEXT_IUC).
- p = iucv_irq_data[smp_processor_id()] (the declared buffer).
- if p.ippathid >= iucv_max_pathid: WARN_ON; iucv_sever_pathid(p.ippathid, iucv_error_no_listener); return.
- BUG_ON(p.iptype < 0x01 ∨ p.iptype > 0x09).
- Allocate `iucv_irq_list` (GFP_ATOMIC); memcpy(work->data, p, sizeof).
- If p.iptype == 0x01 (path pending): list_add_tail to `iucv_work_queue`; schedule_work(&iucv_work).
- Else: list_add_tail to `iucv_task_queue`; tasklet_schedule(&iucv_tasklet).

REQ-18: iucv_tasklet_fn — bottom half:
- spin_trylock(&iucv_table_lock); if fails, reschedule self.
- iucv_active_cpu = smp_processor_id().
- list_splice_init(&iucv_task_queue, &task_queue) under `iucv_queue_lock`.
- For each entry: dispatch via `irq_fn[iptype]`:
  - 0x02 → iucv_path_complete.
  - 0x03 → iucv_path_severed.
  - 0x04 → iucv_path_quiesced.
  - 0x05 → iucv_path_resumed.
  - 0x06 / 0x07 → iucv_message_complete.
  - 0x08 / 0x09 → iucv_message_pending.
- kfree each `iucv_irq_list`.
- iucv_active_cpu = -1; spin_unlock(&iucv_table_lock).

REQ-19: iucv_work_fn — path-pending workqueue (may sleep):
- spin_lock_bh(&iucv_table_lock); iucv_active_cpu = cpu.
- list_splice_init(&iucv_work_queue, ...) under queue_lock.
- iucv_cleanup_queue (drain stale).
- For each: iucv_path_pending; kfree.
- iucv_active_cpu = -1; spin_unlock_bh.

REQ-20: iucv_path_pending(data) — incoming connect:
- BUG_ON(iucv_path_table[ipp.ippathid]) /* table slot must be free */.
- path = iucv_path_alloc(ipp.ipmsglim, ipp.ipflags1, GFP_ATOMIC).
- iucv_path_table[path.pathid] = path; EBCDIC→ASCII on ipp.ipvmid.
- Walk iucv_handler_list; for each handler with non-null path_pending:
  - list_add to handler->paths; path->handler = handler; call path_pending(path, ipvmid, ipuser).
  - if returns 0: accepted — keep state.
  - if returns non-zero: list_del; path->handler = NULL; try next handler.
- If no handler accepts: iucv_sever_pathid(ipp.ippathid, iucv_error_no_listener); free path.

REQ-21: iucv_cleanup_queue:
- Caller holds `iucv_table_lock`.
- smp_call_function(__iucv_cleanup_queue, NULL, 1) /* flush pending external interrupts to the work queue via empty function on each cpu */.
- Under `iucv_queue_lock`: iterate `iucv_task_queue`; for entries with `iucv_path_table[ippathid] == NULL` (stale): list_del; kfree.

REQ-22: CPU hotplug hooks:
- prepare (CPUHP_NET_IUCV_PREPARE): alloc `iucv_irq_data[cpu]`, `iucv_param[cpu]`, `iucv_param_irq[cpu]` (kmalloc_node, GFP_KERNEL|GFP_DMA).
- dead: kfree all three.
- online (CPUHP_AP_ONLINE_DYN): if iucv_path_table not null, iucv_declare_cpu.
- down_prep: refuse to offline last IUCV-enabled cpu; otherwise retrieve_cpu + transfer mask to first remaining buffer-cpu.

REQ-23: Reboot notifier:
- if `iucv_irq_cpumask` empty: NOTIFY_DONE.
- Block IRQs on all enabled cpus.
- For each pathid in iucv_path_table: iucv_sever_pathid (preempt-disabled).
- iucv_disable.

REQ-24: iucv_alloc_device:
- kzalloc + dev_set_name (formatted) + bus = &iucv_bus + parent = iucv_root + driver + groups = attrs + release = iucv_release_device.

REQ-25: Exported `struct iucv_interface iucv_if`:
- Aggregates all message_* / path_* / iucv_register / iucv_unregister + `bus` + `root` (populated post-init).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `b2f0_param_phys_aligned` | INVARIANT | per-call_b2f0: virt_to_phys(parm) 8-byte aligned. |
| `params_zeroed_before_use` | INVARIANT | per-iucv_path_*/message_*: `memset(parm, 0, sizeof)` before populating. |
| `iucv_register_requires_available` | INVARIANT | per-register: !available ⟹ -ENOSYS. |
| `path_table_bounds` | INVARIANT | per-path_table accesses: index < iucv_max_pathid. |
| `dispatch_iptype_table_bounded` | INVARIANT | per-tasklet: iptype in [0x02, 0x09]; 0x01 routed to work-queue. |
| `path_pending_unique_handler` | INVARIANT | per-path_pending: only one handler accepts (first to return 0). |
| `unregister_severs_all_paths` | INVARIANT | per-unregister: handler.paths empty post-call. |
| `cpumask_nonempty_before_b2f0` | INVARIANT | per-message_*/path_*: cpumask_buffer non-empty else -EIO. |
| `tasklet_serialized_via_trylock` | INVARIANT | per-tasklet_fn: spin_trylock(table_lock); reschedule on fail. |
| `nonsmp_handler_count_balanced` | INVARIANT | per-register/unregister: nonsmp_handler increments and decrements pair. |

### Layer 2: TLA+

`net/iucv/iucv.tla`:
- States: uninit / available / handler-registered / path-connecting / path-connected / message-inflight / path-severed / iucv-disabled.
- Variables: path_table (function pathid → Option<Path>), handler_list (set), task_queue / work_queue (sequences), buffer_cpumask, irq_cpumask, active_cpu.
- Actions: Register / Unregister / Connect / Accept / Sever / Send / Receive / Reply / ExternalInterrupt / TaskletDispatch / WorkDispatch / PathPending / PathComplete / Quiesce / Resume / Reboot / CpuOnline / CpuDown.
- Properties:
  - `safety_pathid_table_consistent` — pathid in table iff path connected/pending-accept; severed implies None.
  - `safety_handler_paths_subset_of_table` — every handler.path is in path_table.
  - `safety_active_cpu_implies_lock_held` — active_cpu != -1 ⟹ tasklet or work owns table_lock.
  - `safety_no_path_dispatch_after_sever` — IRQ for pathid post-sever is dropped by cleanup_queue.
  - `safety_iptype_routing` — iptype 0x01 → work_queue; 0x02..0x09 → task_queue.
  - `safety_register_serialized` — at most one register/unregister in critical section (register_mutex).
  - `liveness_connect_eventually_completes_or_severs` — every Connect leads to path_complete or sever.
  - `liveness_send_eventually_arrives` — every Send on live path leads to message_pending on peer (under fairness).
  - `liveness_unregister_drains` — Unregister leads to handler.paths empty.
  - `liveness_reboot_severs_all` — Reboot leads to ∀ pathid: path_table[pathid] = None.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `iucv_init` post: machine_is_vm ∧ available ∧ bus_registered, or err with all resources rolled back | `Iucv::init` |
| `iucv_register` post: handler in handler_list ∨ Err; nonsmp count consistent | `Iucv::register_handler` |
| `iucv_unregister` post: handler.paths empty; handler not in handler_list | `Iucv::unregister_handler` |
| `iucv_path_connect` post: rc == 0 ⟹ path in handler.paths ∧ path_table[ippathid] = path; rc != 0 ⟹ no state mutation | `IucvPath::connect` |
| `iucv_path_sever` post: path not in handler.paths; path_table[pathid] = None | `IucvPath::sever` |
| `iucv_message_send` post: rc == 0 ⟹ msg.id populated | `IucvMessage::send` |
| `iucv_path_pending` post: handler accepted ⟹ path in handler.paths; none ⟹ sever + free | `Iucv::handle_path_pending` |
| `external_interrupt` post: irq enqueued on correct queue ∨ severed for OOB pathid | `Iucv::external_interrupt` |
| `tasklet_fn` post: task_queue drained; active_cpu reset | `Iucv::tasklet_fn` |
| `iucv_declare_cpu` post: rc == 0 ⟹ cpu in buffer_cpumask | `Iucv::declare_cpu` |

### Layer 4: Verus/Creusot functional

`Per IUCV connect (iucv_path_connect → B2F0 IUCV_CONNECT → CP allocates pathid → path inserted in iucv_path_table → eventual external IRQ 0x02 path_complete → handler->path_complete callback) → live path` semantic equivalence: per `net/iucv/iucv.c:iucv_path_connect/iucv_path_complete/iucv_tasklet_fn` and per CP Programming Service SC24-5760 IUCV CONNECT description.

`Per IUCV send-receive (iucv_message_send on connected path → B2F0 IUCV_SEND → peer external IRQ 0x08/0x09 message_pending → peer iucv_message_receive → B2F0 IUCV_RECEIVE → user buffer populated) → end-to-end byte equality` semantic equivalence: per `net/iucv/iucv.c:iucv_message_send/__iucv_message_receive/iucv_message_pending` and per SC24-5760 SEND / RECEIVE semantics.

`Per IUCV teardown (iucv_path_sever → B2F0 IUCV_SEVER → iucv_path_table[pathid] = NULL → peer eventually IRQ 0x03 path_severed → peer handler->path_severed callback) → path closed` semantic equivalence: per `net/iucv/iucv.c:iucv_path_sever/iucv_path_severed`.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

IUCV-base reinforcement:

- **Per-`machine_is_vm()` gate at init** — defense against per-non-z/VM use of the privileged B2F0 instruction.
- **Per-`iucv_available` flag checked by register/unregister** — defense against per-init-failed but module loaded.
- **Per-`iucv_max_pathid` upper bound on path_table[]** — defense against per-out-of-range pathid OOB write.
- **Per-`BUG_ON(iptype out of [0x01,0x09])` in ISR** — defense against per-malformed CP delivery.
- **Per-`WARN_ON + sever` for OOB ippathid in ISR** — defense against per-stale-pathid race.
- **Per-`smp_call_function(__iucv_cleanup_queue)` flush before queue scan** — defense against per-pending-IRQ-on-other-cpu missed by cleanup.
- **Per-`iucv_active_cpu` sentinel** — defense against per-tasklet re-entry self-deadlock when handler calls iucv_path_sever.
- **Per-EBCDIC sever-reason strings (NO LISTENER / NO MEMORY / INVALID PATHID)** — defense against per-CP-cannot-decode-ASCII rejection.
- **Per-`iucv_register_mutex` serializing enable/disable** — defense against per-concurrent-enable double-declare.
- **Per-`spin_trylock + reschedule` in tasklet** — defense against per-tasklet-vs-iucv_table_lock priority inversion.
- **Per-`memset(parm, 0)` before every B2F0** — defense against per-leftover-field-leaked into CP call.
- **Per-`GFP_DMA` for params + irq buffers** — defense against per-address-above-2GB unreachable by CP.
- **Per-`cpumask_buffer non-empty` check** — defense against per-call-with-no-declared-buffer.
- **Per-reboot-notifier sever-all** — defense against per-reboot leaving stale paths on CP side.
- **Per-CPU-hotplug down_prep refuses last cpu** — defense against per-loss-of-IUCV-delivery.
- **Per-EBCDIC-uppercase userid + system** — defense against per-CP-userid-mismatch (CP is case-sensitive EBCDIC).
- **Per-`FLAGS_STRICT`-style IPNORPY in one-way send** — defense against per-CP-waiting-for-reply hangs.
- **Per-FUTEX-style `iucv_path_table[pathid] = NULL` after sever** — defense against per-pathid-reuse UAF.
- **Per-non-SMP handler single-cpu mask (`iucv_setmask_up`)** — defense against per-out-of-order delivery to handlers that can't tolerate it.

