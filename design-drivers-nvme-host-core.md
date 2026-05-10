---
title: "Tier-3: drivers/nvme/host/core.c — NVMe controller + namespace lifecycle (identify + reset state machine + AEN + sanitize/format dispatch)"
tags: ["tier-3", "drivers-nvme", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The arch- + transport-agnostic core of the NVMe host driver — what every NVMe controller (PCIe, TCP, RDMA, FC) plugs into for: controller probing + identify, namespace discovery + rescan, reset + reconnect state machine, async-event-notification (AEN) processing (NS_CHANGED → rescan, ANA_CHANGE → multipath state update, FW_ACT_STARTING → log), per-controller sysfs surface, controller character-device (`/dev/nvme<N>`), per-namespace block-device (`/dev/nvme<N>n<M>`) creation, sanitize / format / SED-Opal / persistent-reservation dispatch, controller power-state (PSD) management, hwmon temperature reporting, fault injection.

This Tier-3 covers `drivers/nvme/host/core.c` (~5500 lines) and is the "shared" code consumed by per-transport drivers in `host/{pci,tcp,rdma,fc}.c` (each Tier-3 of its own).

### Acceptance Criteria

- [ ] AC-1: PCIe NVMe drive enumerates → `/dev/nvme0` chardev + `/dev/nvme0n1` blockdev appear → `nvme list` matches upstream baseline.
- [ ] AC-2: `nvme id-ctrl /dev/nvme0` + `nvme id-ns /dev/nvme0n1` produce byte-identical strings vs upstream baseline.
- [ ] AC-3: Reset stress: `nvme reset /dev/nvme0` 100x; in-flight IOs survive (re-queued); KASAN clean.
- [ ] AC-4: AEN test: target sends NS_CHANGED → host triggers rescan, new namespace appears as `/dev/nvme0n2`.
- [ ] AC-5: ANA failover test (multipath): target reports new ANA state → host re-routes I/O within reconnect_delay periods.
- [ ] AC-6: Sanitize test: `nvme sanitize /dev/nvme0 -a 1` (block-erase) succeeds + completes asynchronously.
- [ ] AC-7: SED-Opal test: `sedutil-cli --setupOpal $passwd /dev/nvme0n1` works; subsequent locked range reads return -EIO.
- [ ] AC-8: PR test: `nvme resv-acquire /dev/nvme0n1 -k 0xabc` from initiator A succeeds; second from B returns RESERVATION_CONFLICT.
- [ ] AC-9: nvme-cli + fio io_uring engine with `cmd_type=nvme` against `/dev/ng0n1` achieves IOPS within 2% of upstream baseline.
- [ ] AC-10: kselftest `tools/testing/selftests/nvme/` passes.
- [ ] AC-11: blktests nvme/* test suite passes.

### Architecture

`Ctrl` lives in `kernel::nvme::host::Ctrl`:

```
struct Ctrl {
  refcount: Refcount,
  state: AtomicU32,             // NVME_CTRL_NEW / CONNECTING / LIVE / RESETTING / DELETING / DEAD
  ops: &'static dyn NvmeCtrlOps,
  ctrl_kobj: KObject,           // /sys/class/nvme/nvme<N>/
  cdev: CharDev,                // /dev/nvme<N>
  cntlid: u16,
  subsys: Arc<Subsystem>,
  namespaces: Mutex<Vec<Arc<Ns>>>,
  ns_id_array: RcuMutex<HashMap<u32, Arc<Ns>>>,
  scan_work: WorkStruct,
  async_event_work: WorkStruct,
  fw_act_work: WorkStruct,
  ana_work: WorkStruct,
  delete_work: WorkStruct,
  reset_work: WorkStruct,
  ka_work: DelayedWorkStruct,    // keep-alive (fabrics)
  kato: u32,                     // keep-alive timeout
  ctrl_loss_tmo: i32,
  reconnect_delay: u32,
  ctratt: u32,                   // controller attributes from identify
  cmic: u8,                      // multi-controller / shared NS / multi-port flags
  dctype: u8,
  identify_data: KBox<NvmeIdCtrl>,
  ana_log: Mutex<Option<KBox<NvmeAnaLog>>>,
  passthru_filter: KBox<NvmePassthruFilter>,
  cntrltype: u8,                 // I/O / Discovery / Admin
  hwmon: Option<KBox<NvmeHwmon>>,
}
```

`Ns` lives in `kernel::nvme::host::Ns`:

```
struct Ns {
  refcount: Refcount,
  ctrl: Arc<Ctrl>,
  nsid: u32,
  disk: Arc<GenDisk>,            // block-device
  blk_queue: Arc<RequestQueue>,  // blk-mq queue
  ns_kobj: KObject,              // /sys/class/block/.../device/
  features: u32,                  // dataset-mgmt + write-zeroes + etc. capability flags
  lba_shift: u32,
  ms: u16,                       // metadata size
  pi_type: u8,                   // T10 PI type
  ana_state: AtomicU8,           // OPTIMAL / NON_OPTIMAL / INACCESSIBLE / PERSISTENT_LOSS / CHANGE
  ana_grpid: u32,
  multipath_link: Option<Arc<NsHead>>,  // for multipath
  pr_supports: bool,
  features_aen: u32,
  nguid: [u8; 16],
  eui64: [u8; 8],
  uuid: Uuid,
}
```

Lifecycle:
1. Per-transport driver (e.g., `host/pci.c`) probes hardware, calls `Ctrl::init(&self.ctrl, dev, &ops, quirks)`.
2. Per-transport completes connect (transport-specific): for PCIe, reads CC + CAP regs; for fabrics, sends Connect command on admin queue.
3. `Ctrl::start(&ctrl)`:
   - `Ctrl::change_state(LIVE)`.
   - `Ctrl::init_identify(&ctrl)` → submit Identify Controller + Identify Active Namespace IDs.
   - Schedule `scan_work` to enumerate namespaces.
   - For fabrics, schedule `ka_work` (keep-alive at kato/2 interval).
4. `scan_work`:
   - Issue Identify Active Namespace IDs (CNS=2) to enumerate present nsids.
   - For each new nsid: `Ns::alloc(&ctrl, info)` → submit Identify Namespace, populate fields, register `/sys/class/block/...`, `disk_add(&ns.disk)`.
   - For each disappeared nsid: `Ns::remove(&ns)` → `disk_del(&ns.disk)`, drop refcount.
5. `async_event_work` (queued from per-transport completion of admin AEN):
   - Decode result-encoded type + subtype.
   - Per-type dispatch:
     - NOTICE_NS_CHANGED → schedule `scan_work`.
     - NOTICE_FW_ACT_STARTING → schedule `fw_act_work` (logs CRIT).
     - NOTICE_ANA_CHANGE → schedule `ana_work` (re-fetch ANA log + update per-NS ana_state).
     - NOTICE_RESERVATION_LOG_AVAIL → log + queue PR notification.
     - NOTICE_SUBSYS_ERR → log fatal + schedule reset.
   - Re-arm AEN: submit another async-event admin command.

Controller reset state machine:
1. `Ctrl::reset(&ctrl)` → schedule `reset_work`.
2. `reset_work`:
   - `Ctrl::change_state(RESETTING)`.
   - `Ctrl::quiesce_io_queues(&ctrl)` (block new submissions; in-flight requests time out + mark for retry).
   - `Ctrl::disable(&ctrl, false)` (CC.EN=0).
   - Per-transport reset (PCIe: SHN bit + secondary bus reset; fabrics: re-Connect).
   - `Ctrl::enable(&ctrl)` + wait CSTS.RDY=1.
   - `Ctrl::change_state(CONNECTING)` then `Ctrl::change_state(LIVE)`.
   - `Ctrl::unquiesce_io_queues(&ctrl)` + `kick_requeue_lists` to retry pending requests.
3. On failure: `Ctrl::change_state(DELETING)` then `DEAD`; call `nvme_remove_namespaces` + `delete_ctrl`.

Per-IOCTL chardev dispatch in `nvme_dev_ioctl`:
- `NVME_IOCTL_ID` → return per-controller cntlid.
- `NVME_IOCTL_ADMIN_CMD` → CAP_SYS_ADMIN check + admin opcode allowlist + `nvme_user_cmd(...)` → submit on admin queue.
- `NVME_IOCTL_IO_CMD` → namespace lookup + IO opcode allowlist + `nvme_user_cmd(...)`.
- `NVME_IOCTL_RESET` → `Ctrl::reset`.
- `NVME_IOCTL_RESCAN` → schedule `scan_work`.
- `NVME_IOCTL_SUBSYS_RESET` → CAP_SYS_RAWIO + per-transport NSSR (PCIe) or fabrics reset.
- `NVME_URING_CMD_*` → io_uring SQE-based passthrough (cross-ref `io_uring/00-overview.md`).

Per-Ns block-device:
- `Ns::setup_cmd(&ns, req, &mut cmnd)` translates blk-mq req → NVMe SQE: opcode (READ/WRITE/FLUSH/WRITE_ZEROES/DSM/COPY/COMPARE), LBA + nsid + length, PRP/SGL build via `nvme_setup_prp_sgl(...)`.
- `Ns::complete_rq(rq)` translates NVMe completion status → blk_status_t.
- T10 PI handled via per-Ns `pi_type` + `ms` + auto-PRACT.
- ZNS (Zoned Namespace) commands routed to `host/zns.c` (separate Tier-3).

### Out of Scope

- PCIe transport (covered in `host-pci.md` future Tier-3)
- TCP transport (covered in `host-tcp.md` future Tier-3)
- RDMA transport (covered in `host-rdma.md` future Tier-3)
- FC transport (covered in `host-fc.md` future Tier-3)
- Multipath logic (covered in `host-multipath.md` future Tier-3)
- ZNS commands (covered in `host-zns.md` future Tier-3)
- PR (covered in `host-pr.md` future Tier-3)
- Auth (covered in `host-auth.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvme_ctrl` | per-controller control block | `kernel::nvme::host::Ctrl` |
| `struct nvme_ns` | per-namespace control block | `kernel::nvme::host::Ns` |
| `struct nvme_subsystem` | per-NVM subsystem (controllers sharing same NQN) | `kernel::nvme::host::Subsystem` |
| `nvme_init_ctrl(ctrl, dev, ops, quirks)` | construct ctrl struct (per-transport calls before connect) | `Ctrl::init` |
| `nvme_uninit_ctrl(ctrl)` | inverse | `Ctrl::uninit` (Drop) |
| `nvme_start_ctrl(ctrl)` | post-connect startup: identify, scan namespaces, register | `Ctrl::start` |
| `nvme_stop_ctrl(ctrl)` | inverse | `Ctrl::stop` |
| `nvme_change_ctrl_state(ctrl, new_state)` | state machine transition (NEW → CONNECTING → LIVE → RESETTING → CONNECTING → LIVE OR DELETING OR DEAD) | `Ctrl::change_state` |
| `nvme_init_identify(ctrl)` | issue Identify Controller + Identify Namespace, populate ctrl fields | `Ctrl::init_identify` |
| `nvme_scan_work(work)` | namespace rescan worker | `Ctrl::scan_work` |
| `nvme_remove_namespaces(ctrl)` | unregister all namespaces | `Ctrl::remove_namespaces` |
| `nvme_alloc_ns(ctrl, info)` | construct namespace + block device | `Ns::alloc` |
| `nvme_free_ns(ns)` | inverse | `Ns::free` (Drop) |
| `nvme_complete_async_event(ctrl, status, result)` | AEN dispatch | `Ctrl::complete_async_event` |
| `nvme_handle_aen_notice(ctrl, result)` | per-AEN-type dispatch (NS_CHANGED / ANA_CHANGE / FW_ACT_STARTING / RESERVATION_LOG_AVAIL / SUBSYS_ERR) | `Ctrl::handle_aen_notice` |
| `nvme_setup_cmd(ns, req, cmnd)` | populate NVMe command from blk-mq request | `Ns::setup_cmd` |
| `nvme_complete_rq(rq)` | per-request completion | `Ns::complete_rq` |
| `nvme_try_complete_req(rq, status, result)` | try to complete (returns false if needs retry) | `Ns::try_complete_req` |
| `nvme_pr_*` family | persistent-reservations bridge to block pr_ops | `Ns::pr_*` |
| `nvme_sec_submit(...)` | SED-Opal command submit | `Ns::sec_submit` |
| `nvme_revalidate_disk(ns, info)` | re-read namespace identify, update size/format | `Ns::revalidate_disk` |
| `nvme_passthru_start(ctrl, ns, opcode)` / `_end` | passthrough cmd lifecycle | `Ns::passthru_start` / `_end` |
| `nvme_kick_requeue_lists(ctrl)` | re-queue blocked requests after reset | `Ctrl::kick_requeue_lists` |
| `nvme_quiesce_io_queues(ctrl)` / `_unquiesce_*` | pause I/O during reset | `Ctrl::quiesce_io_queues` / `_unquiesce_io_queues` |
| `nvme_reset_ctrl(ctrl)` | trigger controller reset | `Ctrl::reset` |
| `nvme_disable_ctrl(ctrl, shutdown)` | disable controller (CC.EN=0) | `Ctrl::disable` |
| `nvme_enable_ctrl(ctrl)` | enable controller (CC.EN=1, wait CSTS.RDY=1) | `Ctrl::enable` |
| `nvme_keep_alive_work(work)` | keep-alive worker for fabrics | `Ctrl::keep_alive_work` |

### compatibility contract

REQ-1: `struct nvme_ctrl_ops` source-compat for in-tree per-transport drivers (pci/tcp/rdma/fc) — every callback (name, reg_read32, reg_write32, reg_read64, free_ctrl, submit_async_event, delete_ctrl, get_address) preserved.

REQ-2: Controller state machine (NVME_CTRL_NEW → CONNECTING → LIVE → RESETTING → CONNECTING → LIVE → DELETING → DEAD) byte-identical state names + transitions; `/sys/class/nvme/nvme<N>/state` writes show identical strings.

REQ-3: Identify Controller + Identify Namespace per NVMe spec § 5.17 byte-identical; vendor-specific identify pages (Intel / Samsung / Micron / Kioxia) supported via vendor-quirks.

REQ-4: AEN dispatch: per-AEN-type result-encoded subtype field decoded identically; NS_CHANGED triggers `nvme_scan_work`, ANA_CHANGE triggers per-namespace path-state update (cross-ref `multipath.md`).

REQ-5: Per-namespace block-device naming: `/dev/nvme<ctrlid>n<nsid>` byte-identical; with multipath enabled: `/dev/nvme<subsysid>n<nsid>` (per-subsystem namespace).

REQ-6: Per-controller chardev `/dev/nvme<N>` IOCTLs (`NVME_IOCTL_ID`, `_ADMIN_CMD`, `_IO_CMD`, `_RESET`, `_RESCAN`, `_SUBSYS_RESET`, `_ADMIN64_CMD`, `_IO64_CMD`, `_URING_CMD_*`) byte-identical (nvme-cli + io_uring NVMe-passthru consumers run unmodified).

REQ-7: Per-controller sysfs surface (`/sys/class/nvme/nvme<N>/`): model, serial, firmware_rev, state, transport, address, cntlid, subsysnqn, hostnqn, hostid, cntrltype, dctype, nuse, numa_node, queue_count, sqsize, reset_controller, rescan_controller, delete_controller, kato, ctrl_loss_tmo, reconnect_delay, dhchap_secret, dhchap_ctrl_secret, dhchap_hash, dhchap_dhgroup byte-identical.

REQ-8: Per-namespace sysfs (`/sys/class/block/nvme<N>n<M>/device/`): nsid, nguid, eui, uuid, wwid, ana_grpid, ana_state, iopolicy byte-identical.

REQ-9: Reset state machine: in-flight commands during reset terminated cleanly; on successful reset, requests requeued via `kick_requeue_lists`.

REQ-10: Sanitize / Format / SED-Opal / persistent-reservation operations dispatch correctly to the underlying namespace + return identical status to userspace.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ctrl_no_uaf` | UAF | `Arc<Ctrl>` outlives every Arc<Ns>; ns.ctrl always valid. |
| `ns_no_uaf` | UAF | `Arc<Ns>` outlives every in-flight blk-mq request; complete_rq always valid. |
| `state_transition_valid` | TRANSITION | `Ctrl::change_state` allows only spec-valid transitions; invalid transition rejected. |
| `nsid_no_oob` | OOB | nsid lookup in ns_id_array bounds-checked (HashMap); per-ns LBA arithmetic checked. |
| `aen_dispatch_no_overflow` | OVERFLOW | per-AEN-type result-decoded subtype range-checked before per-type table dispatch. |

### Layer 2: TLA+

`models/nvme/ctrl_reset.tla` (parent-declared): proves controller-reset state machine + concurrent in-flight commands during reset terminate cleanly.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ctrl::change_state(NEW → CONNECTING)` is the only entry-point transition; CONNECTING → LIVE only after identify completes; LIVE → RESETTING only via `nvme_reset_ctrl` | `Ctrl::change_state` |
| `Ns::setup_cmd` post: cmnd.cdw10/11 LBA range fits in `[0, ns.size_in_lbas)`; never wraps | `Ns::setup_cmd` |
| `nvme_user_cmd` invariant: admin opcode passed through io_uring fast-path is in allowlist (Identify, GetLogPage, GetFeatures); other opcodes require explicit ADMIN_CMD ioctl | `Ns::user_cmd` |

### Layer 4: Verus/Creusot functional

`Ctrl::start` → `Ctrl::scan_work` → `Ns::alloc` round-trip equivalence: a PCIe NVMe drive with N namespaces exposes N block devices `/dev/nvme0n1..N` with byte-identical sysfs surface vs upstream. Encoded as Verus model: `forall hw. rookery_enumerate(hw) == upstream_enumerate(hw)`.

### hardening

(Inherits row-1 features from `drivers/nvme/00-overview.md` § Hardening.)

host-core specific reinforcement:

- **Admin opcode allowlist for io_uring fast-path** — only Identify / GetLogPage / GetFeatures pass; fw-image-download / sanitize / format-NVM / set-features-LBA-format / namespace-mgmt require explicit `NVME_IOCTL_ADMIN_CMD` path with explicit cap check.
- **DH-HMAC-CHAP secrets keyring-shielded** — secrets written to `/sys/.../dhchap_secret` stored in kernel keyring (cross-ref `security/keys/`); not readable from userspace post-install.
- **State-machine transition table read-only-after-init** — defense against stale function pointer.
- **AEN-dispatch-table bounded** — per-AEN-type table indexed by enum-bounded subtype; out-of-range subtype logs WARN + ignored.
- **Reset rate-limit** — per-controller reset rate-limited at 1/sec to prevent reset-storm DoS.
- **Per-controller ka_work cancel-on-state-change** — keep-alive cancelled when state leaves LIVE; defense against stale-keep-alive after disconnect.
- **CAP_SYS_RAWIO required for NVME_IOCTL_SUBSYS_RESET** — defense against subsystem-reset (which resets all controllers in subsystem) from unprivileged process.
- **Multipath path-selection deterministic per ANA-state** — never selects path with ana_state==INACCESSIBLE; head-failover preserves submission tag.

