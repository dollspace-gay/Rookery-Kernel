# Tier-3: drivers/iommu/iommufd/eventq.c — PRI fault eventq (per-fault delivery to userspace + ack via Page Response)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/eventq.c
  - drivers/iommu/iommufd/iommufd_private.h
  - include/uapi/linux/iommufd.h (IOMMU_FAULT_QUEUE_*)
-->

## Summary

The iommufd PRI (Page Request Interface) fault eventq — when a PASID-bound PCIe device faults on a non-present page (cross-ref `iommu-sva.md` for the in-kernel SVA path), this code provides the alternative **userspace-handled** fault path. Used by VMM (qemu/cloud-hypervisor) for: nested-IOMMU guest where the guest manages the L1 pagetable + handles its own page faults at guest-userspace level (e.g., guest SVA-bound device → guest application's mm faults → guest handles → guest sends Page Response → host iommufd forwards to device). Per-fault `iommufd_fault_event` UAPI struct read by userspace via `read(eventq_fd)`; userspace processes fault + writes back Page Response struct via `write(eventq_fd)` to ack.

This Tier-3 covers `drivers/iommu/iommufd/eventq.c` (~540 lines): per-fault-eventq alloc/destroy + per-fault enqueue (from per-IOMMU IRQ handler) + per-fault read/write file_operations.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_eventq` | per-eventq object | `kernel::iommu::iommufd::Eventq` |
| `struct iommufd_eventq_iopf` | per-IOPF (IO Page Fault) eventq | `kernel::iommu::iommufd::EventqIopf` |
| `struct iommufd_eventq_vevent` | per-vEVENT eventq for VIOMMU | `kernel::iommu::iommufd::EventqVevent` |
| `iommufd_fault_alloc(ictx, cmd)` | top-level IOMMU_FAULT_QUEUE_ALLOC ioctl | `Ctx::handle_fault_queue_alloc` |
| `iommufd_eventq_iopf_init(eventq, ictx)` / `_deinit(...)` | per-iopf init | `EventqIopf::init` / `_deinit` |
| `iommufd_eventq_iopf_handler(group)` | per-fault-group dispatch (called from per-IOMMU PRQ IRQ) | `EventqIopf::handler` |
| `iommufd_fault_iopf_handler(group)` | direct hook from per-vendor PRQ | `Device::fault_iopf_handler` |
| `iommufd_fault_iopf_enable(idev)` / `_disable(idev)` | per-device iopf register/unregister | `Device::fault_iopf_enable` / `_disable` |
| `iommufd_eventq_iopf_read(filp, buf, count, ppos)` | per-eventq read fop | `EventqIopf::read` |
| `iommufd_eventq_iopf_write(filp, buf, count, ppos)` | per-eventq write fop (Page Response) | `EventqIopf::write` |
| `iommufd_eventq_iopf_poll(filp, wait)` | per-eventq poll fop | `EventqIopf::poll` |
| `iommufd_fault_complete_iopf(eventq, group, status)` | per-group completion (userspace-sent Page Response) | `EventqIopf::complete_iopf` |
| `iommufd_eventq_vevent_enqueue(viommu, vevent_req)` | enqueue VIOMMU virtual event | `EventqVevent::enqueue` |
| `iommufd_eventq_iopf_alloc_resp(group)` | alloc per-group response struct | `EventqIopf::alloc_resp` |
| `iommufd_eventq_iopf_free_group(group)` | free per-group struct | `EventqIopf::free_group` |

## Compatibility contract

REQ-1: `IOMMU_FAULT_QUEUE_ALLOC` UAPI byte-identical:
- Input: `flags` (IOMMU_FAULT_QUEUE_PASID_MODE / etc.).
- Output: `out_fault_id` (eventq object id) + `out_fault_fd` (file descriptor for read/write).

REQ-2: Per-eventq fd `read(eventq_fd, buf, sizeof(struct iommufd_fault_event))`:
- Returns one `iommufd_fault_event` per call (or -EAGAIN if non-blocking + empty).
- Per-event: `pasid`, `prgi` (Page Request Group Index), `iova`, `flags` (READ / WRITE / EXEC / PRIV / LAST_FAULT_IN_GROUP), `source_id` (BDF-encoded device).

REQ-3: Per-eventq fd `write(eventq_fd, &response, sizeof(struct iommufd_fault_response))`:
- Userspace acks per-PRGI group via Page Response.
- Per-response: `pasid`, `prgi`, `code` (SUCCESS / INVALID / FAILURE).
- Eventq queues response → per-vendor `iommu_ops->page_response(dev, msg)` sends PCIe-spec PRG-Response to device.

REQ-4: Per-eventq `poll`/`select`: returns POLLIN if pending fault, POLLOUT if response queue has space.

REQ-5: Per-fault group: PRQ entries with same PRGI coalesced into one group; group dispatched to userspace as one batch (multiple `read()` calls return per-event entries with LAST_FAULT_IN_GROUP bit on last).

REQ-6: Per-device PRI enable/disable: `iommufd_fault_iopf_enable(idev)` registers per-device fault handler routing to this eventq; `_disable(idev)` unregisters.

REQ-7: Per-device fault ownership: each device can route faults to at most one eventq at a time; second registration fails with -EBUSY.

REQ-8: Eventq destroy invariants: cannot be destroyed if devices still routing faults to it; userspace must unregister all devices first.

REQ-9: Per-fault rate-limit applied at per-IOMMU level (cross-ref `intel-iommu.md` § PRQ rate-limit) — eventq is downstream consumer.

REQ-10: Per-eventq ordering: faults read in submission order (FIFO); responses processed in submission order.

## Acceptance Criteria

- [ ] AC-1: qemu vfio-pci passthrough w/ nested-IOMMU + guest-managed pagetable: device fault → host iopf → fault delivered to qemu via eventq fd → qemu walks guest mm → sends Page Response → host forwards → device retries.
- [ ] AC-2: Read-then-write round-trip: pop event from eventq fd, ack via response write, verify per-vendor `iommu_ops->page_response` invoked.
- [ ] AC-3: Per-PRGI group coalescing: 5 PRQ entries with same PRGI grouped; LAST_FAULT_IN_GROUP bit set on 5th read entry.
- [ ] AC-4: poll() returns POLLIN when fault pending; epoll integration works.
- [ ] AC-5: Per-device single-eventq enforcement: register fault routing twice → 2nd returns -EBUSY.
- [ ] AC-6: Eventq destroy refused with attached devices: -EBUSY until all `iommufd_fault_iopf_disable` invoked.
- [ ] AC-7: kselftest iommufd fault subset passes.

## Architecture

`Eventq` lives in `kernel::iommu::iommufd::Eventq`:

```
struct Eventq {
  obj: Object,                          // base iommufd object
  type_: EventqType,                    // IOPF / VEVENT
  filep: Arc<File>,                      // eventq fd
  ...
}

struct EventqIopf {
  base: Eventq,
  attached_devices: Mutex<Vec<Arc<Device>>>,
  fault_groups: Mutex<XArray<KBox<IopfGroup>>>,
  pending_groups: SpinLock<LinkedList<KBox<IopfGroup>>>,      // newly-arrived, not yet read
  delivered_groups: SpinLock<LinkedList<KBox<IopfGroup>>>,    // read by userspace, awaiting response
  lock: Mutex<()>,
  read_wait: WaitQueue,
  write_wait: WaitQueue,
  fault_queue_count: AtomicU32,                                // total queued
}

struct IopfGroup {
  prgi: u32,
  pasid: u32,
  dev: Arc<Device>,
  faults: KVec<KBox<IommuFaultEvent>>,
  status: AtomicU32,                                            // SUCCESS / INVALID / FAILURE / PENDING
  list: ListEntry,
}
```

Alloc flow `Ctx::handle_fault_queue_alloc(cmd)`:
1. Allocate `EventqIopf`.
2. Allocate per-eventq file via `anon_inode_getfd("[iommufd-eventq]", &iopf_eventq_fops, eventq, O_RDWR | O_CLOEXEC)`.
3. Insert eventq into ictx.objects.
4. Return out_fault_id + out_fault_fd.

Per-device PRI register `Device::fault_iopf_enable(idev, eventq)`:
1. Verify device supports PRI + ATS.
2. Verify device not already registered to another eventq (fault.eventq.is_none).
3. `iommu_register_device_fault_handler(dev, iommufd_fault_iopf_handler, &eventq)` (cross-ref `iommu-core.md`).
4. Per-vendor `iommu_ops->dev_enable_feat(dev, IOMMU_DEV_FEAT_IOPF)`.
5. Add idev to eventq.attached_devices.
6. idev.fault.eventq = Arc::clone(&eventq).

Per-fault ingress `iommufd_fault_iopf_handler(group)`:
1. `group.dev` → look up per-device eventq via `idev.fault.eventq`.
2. `EventqIopf::handler(eventq, &group)`:
   - Take eventq.lock.
   - Allocate KBox<IopfGroup>; populate prgi + pasid + dev + faults.
   - Insert into `pending_groups` queue.
   - Insert into `fault_groups` xarray (keyed by prgi+pasid for response lookup).
   - Drop lock.
   - Wake `read_wait`.

Per-fault read `EventqIopf::read(filp, buf, count)`:
1. Wait on `read_wait` if pending_groups empty + blocking.
2. Take lock.
3. Pop head group from pending_groups; move to delivered_groups (waiting for response).
4. For each fault in group: copy `iommufd_fault_event` struct to userspace buffer.
5. Set LAST_FAULT_IN_GROUP bit on last.
6. Return total bytes copied (== count).

Per-response write `EventqIopf::write(filp, buf, count)`:
1. Copy `iommufd_fault_response` from userspace.
2. Take lock.
3. Look up group via `fault_groups.get((prgi, pasid))`.
4. Group.status = response.code.
5. Move group from delivered_groups → completed.
6. Drop lock.
7. `EventqIopf::complete_iopf(eventq, group, status)`:
   - Per-vendor `iommu_ops->page_response(group.dev, msg)` sends PRG-Response to device.
8. Free group.

poll/epoll `EventqIopf::poll(filp, wait)`:
- POLLIN if pending_groups non-empty.
- POLLOUT if delivered_groups < cap (response queue not full).

Eventq destroy `EventqIopf::deinit`:
1. Take lock.
2. Assert `attached_devices.is_empty()` (else -EBUSY).
3. Drain pending_groups + delivered_groups (per-fault default-INVALID response sent before drop).
4. Free eventq struct.

VEVENT path (VIOMMU virtual events) is similar but routes per-VIOMMU virtual events to userspace (used by qemu nested-IOMMU emulation for guest fault forwarding).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `eventq_no_uaf` | UAF | `Arc<EventqIopf>` outlives all per-device fault registrations + pending fault groups. |
| `group_no_uaf` | UAF | `KBox<IopfGroup>` lifetime tied to eventq's queues; complete_iopf moves to completed before free. |
| `prgi_lookup_no_collision` | UNIQUENESS | per-eventq `fault_groups.get((prgi, pasid))` returns at most one in-flight group per (prgi, pasid). |
| `read_write_balanced` | INVARIANT | every group read by userspace eventually responded to (or eventq destroyed); responses processed in correct order. |

### Layer 2: TLA+

`models/iommu/pasid_lifecycle.tla` (parent-declared, includes PRI handshake): proves per-fault flow from device → IOMMU → eventq → userspace → response → device-retry never produces use-after-free of mm or device.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `EventqIopf::handler` post: group inserted in pending_groups + fault_groups xarray; read_wait waiters woken | `EventqIopf::handler` |
| `EventqIopf::read` post: returned bytes count equals sum of `sizeof(struct iommufd_fault_event)` per fault in group; LAST_FAULT_IN_GROUP bit set on last | `EventqIopf::read` |
| `EventqIopf::write` post: per-PRGI group completed via per-vendor page_response; freed from queues | `EventqIopf::write` |
| Per-device single-eventq invariant: each `Device` has at most one eventq.attached at a time | `Device::fault_iopf_enable` |

### Layer 4: Verus/Creusot functional

`device fault → IOMMU PRQ → iopf_handler → EventqIopf::handler → userspace read → userspace handles → write Page Response → per-vendor page_response → device retries access` round-trip equivalence: every PRI fault eventually completes (success / invalid / failure) within bounded time (modulo userspace correctness).

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-eventq specific reinforcement:

- **Per-device single-eventq enforcement** — defense against fault-routing-confusion; second register returns -EBUSY.
- **Per-eventq destroy refused with attached devices** — defense against UAF on per-device fault path.
- **Per-fault rate-limit at per-IOMMU level** (cross-ref `intel-iommu.md`) — eventq is downstream consumer; per-IOMMU rate-limit prevents eventq queue blowup.
- **Per-eventq pending_groups queue cap** — bounded length; over-cap → drop incoming fault + log + send default-INVALID response to device; defense against userspace-blocking-causing fault-queue-blowup.
- **Per-fault response code validated** — only SUCCESS / INVALID / FAILURE allowed; other codes rejected at write().
- **Per-eventq fd CAP_SYS_RAWIO** at IOMMU_FAULT_QUEUE_ALLOC — defense against unprivileged process creating fault eventq for sensitive device.
- **Eventq fd close drains all pending groups** with INVALID response — defense against device-side timeout from unanswered faults.
- **Per-PRGI ordering preserved** — read returns groups in submission order; defense against userspace-side fault-reordering breaking device assumptions.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement above baseline `## Hardening`. The eventq fd is the conduit by which a userspace VMM consumes device PRI faults; it crosses every isolation boundary (device DMA → IOMMU IRQ → fault group → userspace read/write → page-response back to device) and so receives intensive PaX/grsec mitigations.

- **PAX_USERCOPY** on `iommufd_fault_event` read + `iommufd_fault_response` write paths (poll/epoll consumers).
- **PAX_KERNEXEC** on eventq file_operations, `EventqIopf::handler`, and `complete_iopf` (RO post-init).
- **PAX_RANDKSTACK** on read/write/poll syscall chains entering eventq.
- **PAX_REFCOUNT** on `EventqIopf`, `IopfGroup`, attached `Device` refs.
- **PAX_MEMORY_SANITIZE** zeroes `IopfGroup` storage and per-event bufs on free.
- **PAX_UDEREF** on copy_from_user paths in `EventqIopf::write` (Page Response).
- **PAX_RAP/kCFI** on eventq fops vtable and `iommu_ops->page_response` indirect dispatch.
- **GRKERNSEC_HIDESYM** hides `EventqIopf` pointers, per-PRGI xarray, and per-device fault.eventq linkage.
- **GRKERNSEC_DMESG** restricts fault-rate-limit warnings and PRI-disable logs to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** (and **CAP_SYS_RAWIO** for raw eventq alloc) — non-init-userns blocked from owning a PRI eventq.
- **GRKERNSEC_DMA strict-mode** — PRI-capable devices default-detached from eventq until full PCIe PRI/ATS verification.
- **VT-d PRQ rate-limit** strictly inherited; eventq pending_groups cap (default 1024) enforced per userns to deny multi-VM PRI-flood DoS.
- **VFIO-compat refused** for eventq consumers under hardened policy; iommufd-native fd only.
- **ATS/PASID gating** — eventq register refused for devices without verified PRI capability.
- **Per-fault response code allowlist** strict (SUCCESS/INVALID/FAILURE); malformed codes drop the eventq.
- **Userspace-write latency budget** enforced — overdue responses auto-INVALID by kernel to prevent device-side timeout-induced fault storms.

Rationale: the eventq is a user-driven side-channel into IOMMU page-fault servicing; a misbehaving VMM can hang devices, starve other tenants, or pivot via spoofed Page Responses. Hardened Rookery applies refcount, capability, namespace, rate, and validation layers so each defense is independent.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- iommu-sva (in-kernel SVA path) covered in `iommu-sva.md` Tier-3
- iommufd-main top-level dispatch (covered in `iommufd-main.md` Tier-3)
- iommufd-ioas (covered in `iommufd-ioas.md` Tier-3)
- iommufd-hwpt (covered in `iommufd-hwpt.md` Tier-3)
- VIOMMU virtual events (covered in `iommufd-viommu.md` future Tier-3)
- Per-vendor PRQ implementation (covered in `intel-iommu.md` + `amd-iommu.md` Tier-3s)
- 32-bit-only paths
- Implementation code
