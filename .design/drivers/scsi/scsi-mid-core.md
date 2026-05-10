# Tier-3: drivers/scsi/{scsi,hosts,scsi_lib}.c — SCSI mid-layer core (host registration + scsi_cmnd lifecycle + blk-mq integration + EH dispatch)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/scsi.c
  - drivers/scsi/hosts.c
  - drivers/scsi/scsi_lib.c
  - drivers/scsi/scsi_priv.h
  - include/scsi/scsi_host.h
  - include/scsi/scsi_device.h
  - include/scsi/scsi_cmnd.h
-->

## Summary

The brain of the SCSI middle layer — every SAS HBA, every iSCSI initiator, every USB-storage device, every virtio-scsi guest driver, every legacy SATA-via-libata-scsi disk routes through here. Owns: per-host (`Scsi_Host`) registration via `scsi_host_alloc` + `scsi_add_host`, per-target + per-device discovery (cross-ref `scsi-scan.md` future Tier-3), per-`scsi_cmnd` lifecycle, blk-mq integration (every `request_queue` for a SCSI device dispatches via mid-layer), per-LUN queue-depth + request-throttling, per-cmd Sense-data handling, per-cmd EH (Error Handler) escalation entry-point, ULD (`sd`/`sr`/`st`/`ch`/`sg`) registration framework.

This Tier-3 covers the three central files: `scsi.c` (~1100 lines: subsystem init, scsi_cmd_to_dev mapping), `hosts.c` (~770 lines: host registration), `scsi_lib.c` (~3600 lines: blk-mq integration + cmd dispatch).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct Scsi_Host` | per-host control block | `kernel::scsi::Host` |
| `struct scsi_target` | per-target (typically 1-per-port) | `kernel::scsi::Target` |
| `struct scsi_device` | per-LUN | `kernel::scsi::Device` |
| `struct scsi_cmnd` | per-command (allocated by blk-mq) | `kernel::scsi::Cmnd` |
| `struct scsi_host_template` | per-LLD vtable | `kernel::scsi::HostTemplate` (trait) |
| `struct scsi_driver` | per-ULD driver | `kernel::scsi::Driver` |
| `scsi_host_alloc(template, privsize)` | allocate Scsi_Host | `Host::alloc` |
| `scsi_host_put(host)` / `_get` | refcount | `Arc<Host>` get/put |
| `scsi_add_host(host, dev)` / `_remove_host(host)` | register/unregister with driver model | `Host::add` / `_remove` |
| `scsi_scan_host(host)` | trigger initial scan | `Host::scan` |
| `scsi_host_lookup(host_no)` | lookup host by number | `Host::lookup_by_no` |
| `scsi_alloc_request(...)` | per-request allocation (blk-mq) | `Cmnd::alloc` |
| `scsi_init_command(dev, cmd)` | initialize per-cmd | `Cmnd::init` |
| `scsi_setup_cmnd(...)` | populate cmd from request | `Cmnd::setup` |
| `scsi_dispatch_cmd(cmd)` | dispatch to LLD `queuecommand` callback | `Cmnd::dispatch` |
| `scsi_done(cmd)` | per-cmd completion (called from LLD's IRQ handler) | `Cmnd::done` |
| `scsi_finish_command(cmd)` | post-completion processing | `Cmnd::finish` |
| `scsi_io_completion(cmd, good_bytes)` | I/O-side completion (from sd) | `Cmnd::io_completion` |
| `scsi_check_sense(cmd)` | per-cmd sense-data interpretation | `Cmnd::check_sense` |
| `scsi_normalize_sense(sense, sense_len, &shdr)` | parse sense buffer | `SenseHdr::parse` |
| `scsi_block_requests(host)` / `_unblock_requests(host)` | pause new requests | `Host::block_requests` / `_unblock_requests` |
| `scsi_target_block(starget)` / `_unblock` | pause per-target | `Target::block` / `_unblock` |
| `scsi_device_set_state(sdev, state)` | per-device state transition (CREATED → RUNNING → CANCEL → DEL) | `Device::set_state` |
| `scsi_remove_device(sdev)` / `_remove_target(starget)` / `_remove_host(host)` | offline/online | `Device::remove` / `Target::remove` / `Host::remove` |
| `scsi_init_queue()` | global blk-mq queue init | `Subsystem::init_queue` |
| `scsi_mq_alloc_queue(sdev)` | per-device blk-mq queue alloc | `Device::alloc_blk_mq_queue` |
| `scsi_register_driver(driver)` / `_unregister_driver` | per-ULD register | `Driver::register` / `_unregister` |
| `scsi_eh_done(cmd)` / `_eh_finish_cmd(cmd, ...)` | EH per-cmd completion (cross-ref `scsi-error.md`) | `Cmnd::eh_done` / `_eh_finish` |

## Compatibility contract

REQ-1: `struct scsi_host_template` source-compat: every documented callback (`info`, `name`, `proc_name`, `queuecommand`, `commit_rqs`, `eh_*_handler`, `eh_*_reset_handler`, `slave_alloc`, `slave_configure`, `slave_destroy`, `target_alloc`, `target_destroy`, `scan_finished`, `scan_start`, `change_queue_depth`, `map_queues`, `mq_poll`, `bios_param`, `unlock_native_capacity`, `show_info`, `write_info`, `cmd_size`, `proc_name`, `module`, `dma_boundary`, `virt_boundary_mask`, `max_segment_size`, `dma_alignment`) preserved; in-tree LLDs (mpt3sas, lpfc, qla2xxx, …) compile + work.

REQ-2: Per-host blk-mq integration: each `scsi_device` has a `request_queue`; SCSI ULDs (sd/sr/st/ch/sg) submit via blk-mq; per-LLD `queuecommand` invoked from blk-mq dispatch.

REQ-3: Per-cmd lifecycle: blk-mq alloc → `scsi_init_command` → `scsi_setup_cmnd` → `scsi_dispatch_cmd` → per-LLD `queuecommand` → LLD completes → `scsi_done` (from LLD IRQ) → `scsi_finish_command` (from softirq workqueue).

REQ-4: Per-cmd retry policy: per-cmd `retries` counter; on RETRY status, requeue via blk-mq + decrement.

REQ-5: Sense-data interpretation: per-cmd auto-sense + `scsi_check_sense` decodes sense_key/asc/ascq → blk_status_t conversion via `scsi_io_completion_action`.

REQ-6: EH escalation entry-point: `scsi_eh_*` functions enqueue cmd onto host's eh_work workqueue; per-host eh-thread escalates abort → dev_reset → target_reset → bus_reset → host_reset.

REQ-7: Per-host queue-depth + per-LUN queue-depth tracked via `Scsi_Host->host_busy` + `scsi_device->device_busy`; new requests blocked when at depth.

REQ-8: Per-host registration creates `/sys/class/scsi_host/host<N>/` byte-identical sysfs surface (host_busy, can_queue, cmd_per_lun, host_no, state, unique_id, prot_capabilities, etc.).

REQ-9: ULD registration (`scsi_register_driver`): per-ULD `scsi_driver` with `gendrv.{name, owner, probe, remove}` + ULD-specific `init_command`, `done`, `eh_action`, `cleanup`, `rescan`. Per-LUN: ULDs match by inquiry data.

REQ-10: Per-LUN state machine: CREATED → RUNNING → CANCEL → DEL (via `scsi_device_set_state` w/ valid-transition check).

## Acceptance Criteria

- [ ] AC-1: SAS HBA (mpt3sas) enumerates → per-target/per-LUN devices appear → `lsscsi` matches upstream baseline.
- [ ] AC-2: blk-mq integration: `fio --ioengine=libaio --direct=1 --bs=4k --iodepth=128 --numjobs=8 --rw=randread` against `/dev/sda` (SCSI disk) achieves IOPS within 2% of upstream.
- [ ] AC-3: Per-LUN queue-depth: device with `queue_depth=32` rejects 33rd in-flight cmd via blk-mq backpressure.
- [ ] AC-4: Sense-data test: scsi_debug pseudo-host configured to inject CHECK_CONDITION with sense_key=MEDIUM_ERROR; cmd retried 3x then completed with -EIO + sense data passed up.
- [ ] AC-5: Host-block test: `scsi_block_requests(host)` blocks new submissions; in-flight complete normally; `_unblock` resumes.
- [ ] AC-6: Per-LUN state-machine test: `echo 0 > /sys/block/sda/device/state` sets state to BLOCKED; new requests error.
- [ ] AC-7: kselftest scsi subset passes; blktests scsi/* passes against scsi_debug.

## Architecture

`Host` lives in `kernel::scsi::Host`:

```
struct Host {
  refcount: Refcount,
  hostt: Arc<dyn HostTemplate>,
  shost_gendev: KObject,        // /sys/class/scsi_host/host<N>/
  host_no: u32,
  can_queue: u32,                // max simultaneous cmds
  this_id: u32,                  // initiator's SCSI ID
  cmd_per_lun: u32,
  sg_tablesize: u32,
  sg_prot_tablesize: u32,
  max_sectors: u32,
  opt_sectors: u32,
  dma_boundary: u64,
  virt_boundary_mask: u64,
  max_segment_size: u32,
  dma_alignment: u32,
  hostdata: KBox<dyn Any>,        // per-LLD private
  shost_data: KBox<ShostData>,
  host_busy: AtomicI32,
  host_blocked: AtomicI32,
  host_failed: AtomicI32,
  host_eh_scheduled: AtomicU32,
  host_self_blocked: AtomicI32,
  resetting: AtomicBool,
  last_reset: AtomicI64,
  state: AtomicU32,               // SHOST_CREATED / RUNNING / CANCEL / DEL / RECOVERY / CANCEL_RECOVERY
  __targets_lock: SpinLock<()>,
  __targets: Vec<Arc<Target>>,
  ehandler: Option<Arc<KernelThread>>,
  eh_cmd_q: Mutex<Vec<Arc<Cmnd>>>,
  eh_wait_q: WaitQueue,
}
```

`Cmnd` lives in `kernel::scsi::Cmnd`:

```
struct Cmnd {
  device: Arc<Device>,
  request: Arc<BlkMqRequest>,    // backing blk-mq request
  cmnd: [u8; MAX_COMMAND_SIZE],
  cmnd_len: u8,
  sc_data_direction: DmaDirection,
  sdb: ScsiDataBuffer,            // sg list + nents
  prot_sdb: Option<ScsiDataBuffer>, // protection-info sg list (T10 PI)
  underflow: u32,
  transfersize: u32,
  resid_len: u32,
  result: i32,                    // SAM status / msg / host status
  state: AtomicU32,                // SCSI_MLQUEUE_DEVICE_BUSY / _HOST_BUSY / _TARGET_BUSY / _EH_RESET / etc.
  retries: u8,
  allowed: u8,
  flags: u32,
  jiffies_at_alloc: u64,
  cmd_len: u8,
  done: Option<fn(&Cmnd)>,         // legacy completion handler
  sense_buffer: KBox<[u8; SCSI_SENSE_BUFFERSIZE]>,
  prot_op: u8,                    // PROT_NORMAL / READ_INSERT / READ_STRIP / READ_PASS / WRITE_INSERT / WRITE_STRIP / WRITE_PASS
  prot_type: u8,                  // 0/1/2/3 (T10 PI types)
}
```

Per-cmd dispatch flow:
1. blk-mq dequeues request from per-device queue → calls `scsi_queue_rq(hctx, bd)`.
2. `scsi_queue_rq` calls `scsi_init_command(dev, cmd)` to allocate scsi_cmnd from blk-mq's per-request payload area.
3. ULD-specific `init_command` (e.g., `sd_init_command`) populates `cmnd[]` (READ_10/WRITE_10/etc.), `transfersize`, `prot_op`.
4. `scsi_dispatch_cmd(cmd)` calls per-LLD `hostt->queuecommand(host, cmd)`.
5. LLD submits to HW (e.g., posts SCSI command on FC fabric, sends iSCSI PDU, builds AHCI FIS).
6. LLD's IRQ handler later calls `scsi_done(cmd)` with result/status.
7. `scsi_done` schedules `scsi_finish_command(cmd)` via blk-mq's complete-via-softirq path (`__blk_mq_complete_request`).
8. `scsi_finish_command`:
   - If `result` indicates CHECK_CONDITION: `scsi_check_sense(cmd)` → decode sense key.
   - If retry-needed: `__scsi_queue_insert(cmd, ...)` to requeue.
   - Else: ULD's `done` callback → blk_mq_end_request.

Per-host EH thread: when `host_failed > 0` (cmds with non-zero result), wakes per-host eh-thread. eh-thread walks `eh_cmd_q`, applies per-cmd-state escalation:
1. Try abort (per-LLD `eh_abort_handler`).
2. If fail, dev_reset (per-LLD `eh_device_reset_handler`).
3. If fail, target_reset (per-LLD `eh_target_reset_handler`).
4. If fail, bus_reset (per-LLD `eh_bus_reset_handler`).
5. If fail, host_reset (per-LLD `eh_host_reset_handler`).
6. After successful reset at any level, complete pending cmds (success or DID_ERROR), release device/target/bus block.

Per-LUN state machine valid transitions (`scsi_device_set_state`):
- CREATED → RUNNING / DEL
- RUNNING → BLOCKED / CANCEL / DEL / OFFLINE / TRANSPORT_OFFLINE
- BLOCKED → RUNNING / CANCEL / DEL
- CANCEL → DEL
- OFFLINE → RUNNING / DEL
- TRANSPORT_OFFLINE → RUNNING / DEL
- DEL → (terminal)

Invalid transition returns -EINVAL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmd_no_uaf` | UAF | `Cmnd` lifetime tied to blk-mq request; complete-then-LLD-late-IRQ safe via per-cmd state machine. |
| `host_no_uaf` | UAF | `Arc<Host>` outlives all targets/devices; remove waits for refcount==0. |
| `cmnd_len_no_oob` | OOB | `cmnd[..cmnd_len]` bounded by MAX_COMMAND_SIZE; per-LLD setup respects this. |
| `state_transition_valid` | TRANSITION | `Device::set_state` only allows valid transitions per scsi_device-state diagram. |
| `sense_buffer_no_oob` | OOB | sense-data parsing bounded by SCSI_SENSE_BUFFERSIZE (96 bytes). |

### Layer 2: TLA+

`models/scsi/eh_escalation.tla` (parent-declared): proves error-recovery state machine — abort → dev-reset → target-reset → bus-reset → host-reset escalation; concurrent in-flight commands during EH terminate cleanly.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cmnd::dispatch` post: cmd state ∈ {SUBMITTED, MLQUEUE_DEVICE_BUSY, MLQUEUE_HOST_BUSY, MLQUEUE_TARGET_BUSY}; never UNINITIALIZED | `Cmnd::dispatch` |
| `scsi_done` invariant: called exactly once per dispatch | LLD contract |
| `host->host_busy` invariant: equals count of in-flight cmds with state in {SUBMITTED, COMPLETING}; decremented exactly once on completion | dispatch / complete |
| Per-LUN `device_busy` invariant: <= `queue_depth` | dispatch back-pressure |

### Layer 4: Verus/Creusot functional

`scsi_dispatch_cmd(cmd) → LLD queuecommand → IRQ → scsi_done(cmd) → scsi_finish_command(cmd)` round-trip equivalence: a successful round-trip produces blk-mq end_request with same result code as upstream for any (cmd, status) pair.

## Hardening

(Inherits row-1 features from `drivers/scsi/00-overview.md` § Hardening.)

scsi-mid-core specific reinforcement:

- **Per-host refcount saturating** — overflow saturates; never wraps.
- **Per-LUN queue-depth enforced** — defense against single LUN consuming entire host queue (starvation prevention).
- **EH escalation deadline** — per-host EH timeout (default 60s) prevents EH-thread blocking indefinitely on broken HW.
- **Per-cmd retries cap** — `cmd->allowed` (default 5) prevents infinite retry on persistent error.
- **Per-LLD `queuecommand` time-bounded** — per-LLD hook expected to return quickly (queue + return); slow-LLD detection via softlockup watchdog.
- **State-transition table read-only-after-init** — defense against state-machine corruption via stale function pointer.
- **Sense-data parser bounded** — every sense-buffer access validated against SCSI_SENSE_BUFFERSIZE.
- **Per-cmd buffer size validated against per-LLD `sg_tablesize`** — defense against single cmd consuming all SG entries.
- **Per-host `host_busy` int32 saturating** — defense against integer wraparound under sustained load.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- EH details (covered in `drivers/scsi/scsi-error.md` future Tier-3)
- Scan logic (covered in `drivers/scsi/scsi-scan.md` future Tier-3)
- ULD impls (covered in `drivers/scsi/sd.md`, `sg.md`, `sr.md`, `st.md`, `ch.md` future Tier-3s)
- Transport classes (covered in `drivers/scsi/transport-{sas,fc,iscsi,spi,srp}.md` future Tier-3s)
- Per-LLD impls (mpt3sas, lpfc, qla2xxx, smartpqi, etc.)
- 32-bit-only paths
- Implementation code
