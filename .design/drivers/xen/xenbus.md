# Tier-3: drivers/xen/{xenbus,events}/* — XenStore client + event channels + grant-table bridge

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/xen/00-overview.md
upstream-paths:
  - drivers/xen/xenbus/xenbus_xs.c
  - drivers/xen/xenbus/xenbus_comms.c
  - drivers/xen/xenbus/xenbus_client.c
  - drivers/xen/xenbus/xenbus_probe.c
  - drivers/xen/xenbus/xenbus_probe_frontend.c
  - drivers/xen/xenbus/xenbus_probe_backend.c
  - drivers/xen/xenbus/xenbus_dev_frontend.c
  - drivers/xen/xenbus/xenbus_dev_backend.c
  - drivers/xen/events/events_base.c
  - drivers/xen/events/events_2l.c
  - drivers/xen/events/events_fifo.c
  - drivers/xen/grant-table.c
  - include/xen/xenbus.h
  - include/xen/events.h
  - include/xen/grant_table.h
-->

## Summary

XenBus is the Linux-guest client for Xen's shared hierarchical configuration store (XenStore — a key/value tree maintained by `xenstored` in Dom0, accessed by every guest via a shared ring and an event channel). On top of XenBus + event-channels + grant-tables, Xen frontend/backend drivers (blkfront/blkback, netfront/netback, xen-pciback, pvcalls, virtio-on-Xen) build their entire IPC: XenBus carries the connection-state machine and feature negotiation, grant-tables carry the I/O buffers, event-channels carry the wakeups.

This Tier-3 covers `xenbus/` (~3000 lines across xs/comms/client/probe/probe_{front,back}end/dev_{front,back}end) + `events/` (~3000 lines across `events_base.c` plus the two ABI variants `events_2l.c` (2-level) and `events_fifo.c` (FIFO ABI v6.2+)) + a summary of `grant-table.c` since it is the IPC peer.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `xenbus_directory(t, dir, node, *num)` / `xenbus_read(t, dir, node, *len)` / `xenbus_write(...)` / `xenbus_mkdir(...)` / `xenbus_rm(...)` / `xenbus_exists(...)` | per-key XenStore operations | `drivers::xen::xenbus::Xs::*` |
| `xenbus_transaction_start(*t)` / `_end(t, abort)` | XenStore transaction begin/end (atomic multi-key) | `Xs::transaction_*` |
| `xenbus_dev_request_and_reply(*msg, par)` | raw passthrough for `/dev/xen/xenbus` | `XsDev::request_and_reply` |
| `register_xenbus_watch(*watch)` / `unregister_xenbus_watch(*watch)` | per-path watch callback registry | `Watch::register` / `_unregister` |
| `xenwatch_thread` | dedicated kthread that drains watch_events queue + invokes callbacks | `Watch::Kthread` |
| `xenbus_register_frontend(*drv)` / `_register_backend(*drv)` | per-driver registration into `bus_type` `xen` | `Driver::register_{front,back}end` |
| `struct xenbus_device` / `_driver` | the bus_type peer for blkfront/netfront/... | `XenbusDevice` / `XenbusDriver` |
| `xenbus_switch_state(dev, state)` | publish per-device `state` key (`Initialising` → `InitWait` → `Initialised` → `Connected` → `Closing` → `Closed`) | `XenbusDevice::switch_state` |
| `xenbus_alloc_evtchn(dev, *port)` / `xenbus_free_evtchn(dev, port)` | per-device unbound event channel alloc/free | `XenbusDevice::evtchn_*` |
| `xenbus_grant_ring(dev, vaddr, nr_pages, *refs)` | grant-ring helper for blk/netfront | `XenbusDevice::grant_ring` |
| `bind_evtchn_to_irq(port)` / `bind_virq_to_irq(virq, cpu)` / `bind_ipi_to_irq(ipi, cpu)` / `bind_interdomain_evtchn_to_irq(remote_dom, remote_port)` | per-port → Linux IRQ mapping | `Events::bind_*_to_irq` |
| `bind_evtchn_to_irqhandler(port, handler, irqflags, devname, dev_id)` | request_irq wrapper for evtchn-as-irq | `Events::bind_evtchn_to_irqhandler` |
| `notify_remote_via_evtchn(port)` / `_irq(irq)` | hypercall to wake the remote evtchn endpoint | `Events::notify_remote` |
| `evtchn_make_refcounted(port, is_static)` | refcounted-evtchn for sharing across IRQ + userspace | `Events::make_refcounted` |
| `xen_send_IPI_*` / `xen_evtchn_do_upcall` | per-CPU upcall dispatch from hypervisor event vector | `Events::upcall` |
| `evtchn_2l_ops` / `evtchn_fifo_ops` | per-ABI hooks (mask/unmask/clear/set_pending/handle_events) | `Events::Abi::TwoLevel` / `_Fifo` |
| `gnttab_grant_foreign_access(domid, frame, readonly)` / `_end_foreign_access(ref, page)` | per-page grant reference issue/revoke | `GrantTable::grant_*` |
| `gnttab_map_refs(map_ops, kmap_ops, pages, count)` / `_unmap_refs(...)` | per-batch foreign mapping (used by gntdev) | `GrantTable::map_refs` / `_unmap_refs` |
| `gnttab_dma_alloc_pages(...)` / `_dma_free_pages(...)` | DMA-coherent grant pages (used by 9pfs/virtio-Xen) | `GrantTable::dma_alloc_pages` |

## Compatibility contract

REQ-1: XenStore wire protocol — `xsd_sockmsg` framed messages over a shared 1 KB request ring + 1 KB reply ring (`struct xenstore_domain_interface`). XenBus tracks per-message `req_id` (`xs_request_id` counter); replies dispatched to the matching `xb_req_data`.

REQ-2: Per-message transaction tag (`xenbus_transaction t`) — 0 = autocommit (the default); non-zero = inside a `transaction_start..transaction_end` window. Concurrent transactions multiplexed via `xs_state_lock` + per-request kref.

REQ-3: Watches — `register_xenbus_watch(*watch)` posts a WATCH message to xenstored; on every matching key change, a `WATCH_EVENT` message arrives → enqueued on `watch_events` list → drained by `xenwatch_thread` → `watch->callback(watch, path, token)`.

REQ-4: Per-device state machine published as `/local/domain/<domid>/device/<type>/<id>/state` (frontend) or `/local/domain/<domid>/backend/<type>/<otherdom>/<id>/state` (backend); states from `enum xenbus_state` (Unknown→Initialising→InitWait→Initialised→Connected→Closing→Closed→Reconfiguring→Reconfigured).

REQ-5: Event-channel ABIs:
- **2-level (`evtchn_2l_ops`)** — original ABI: per-port pending-bit in a global bitmap + per-domain mask-bit + per-CPU `vcpu_info::evtchn_pending_sel` selector word. Supports `EVTCHN_2L_NR_CHANNELS = 1024` on 32-bit / `4096` on 64-bit.
- **FIFO (`evtchn_fifo_ops`)** — newer ABI: per-event control-block in shared array + per-vCPU linked-list head/tail; allows up to `EVTCHN_FIFO_NR_CHANNELS = 131072`. Selected when `EVTCHNOP_init_control` succeeds.

REQ-6: IRQ binding — every Xen event channel maps to a Linux IRQ via `evtchn_to_irq` table; `xen_dynamic_chip` / `_lateeoi_chip` / `_percpu_chip` / `_pirq_chip` are the corresponding `irq_chip` implementations.

REQ-7: VIRQ + IPI binding — per-vCPU virtual IRQs (`VIRQ_TIMER`, `VIRQ_DEBUG`, `VIRQ_CONSOLE`, …) and inter-processor IPIs share the same evtchn machinery; `bind_virq_to_irq(virq, cpu)` and `bind_ipi_to_irq(ipi, cpu)` install per-CPU bindings.

REQ-8: Per-domain interdomain-evtchn — `bind_interdomain_evtchn_to_irq(remote_dom, remote_port)` lets the frontend talk directly to its backend without going through xenstored.

REQ-9: Grant-table ref allocation — `gnttab_grant_foreign_access(domid, frame, readonly)` allocates a `grant_ref_t` and publishes it in the shared grant table; `_end_foreign_access` revokes. Per-domain grant table sized at boot from `XENMEM_maximum_gpfn` / config option.

REQ-10: `/dev/xen/xenbus` (frontend) + `/dev/xen/xenbus_backend` (backend, Dom0 only) — character devices that pipe raw `xsd_sockmsg` messages to xenstored via the kernel ring; used by `xenstore-{ls,read,write,watch}` tools.

REQ-11: Per-message timeout + cancellation — `xenbus_dev_request_and_reply` uses `kref` lifetime so an interrupted reader doesn't UAF the request.

REQ-12: Suspend/resume — `xs_suspend_enter`/`_exit` halt new requests during PM transition; outstanding watches re-registered on resume.

## Acceptance Criteria

- [ ] AC-1: Boot as PV/PVH guest on Xen 4.18: `/sys/bus/xen/devices/` shows frontend devices (vbd-768, vif-0); each device reports state=Connected after probe.
- [ ] AC-2: `xenstore-ls /local/domain/<domid>/device` lists the per-device tree; matches `dmesg | grep xenbus`.
- [ ] AC-3: `register_xenbus_watch("control")` on a Dom0 key + Dom0 write triggers `watch->callback` within one tick.
- [ ] AC-4: 2-level evtchn ABI: 4096 channels bindable; per-vCPU upcall fires correct IRQ.
- [ ] AC-5: FIFO ABI (`xen_evtchn_fifo_init`) succeeds on Xen ≥ 4.4; bind ≥ 8192 channels without falling back to 2-level.
- [ ] AC-6: `bind_interdomain_evtchn_to_irq(Dom0, port)` from a frontend driver completes and `notify_remote_via_irq` wakes the backend handler.
- [ ] AC-7: `gnttab_grant_foreign_access` issues a fresh ref; `gntdev` mmap of that ref from the other domain succeeds.
- [ ] AC-8: Save/restore (`xl save guest && xl restore guest`) preserves XenBus device state.
- [ ] AC-9: kselftest `tools/testing/selftests/xen/` (where present) passes under KASAN.

## Architecture

`Xs` (XenStore client) lives in `drivers::xen::xenbus::Xs`:

```
struct Xs {
  state_lock: Spinlock,
  state_users: AtomicU32,
  suspend_active: AtomicBool,
  request_id: AtomicU32,
  request_enter_wq: WaitQueueHead,
  request_exit_wq: WaitQueueHead,
  watches: Mutex<List<Arc<Watch>>>,
  watch_events: Spinlock<VecDeque<WatchEvent>>,
  watch_rwsem: RwSemaphore,
  xenwatch_pid: AtomicPid,
  ring: KBox<XenstoreDomainInterface>,  // shared with xenstored
  evtchn: EvtchnPort,                   // signals new messages
}

struct Watch {
  list: ListNode,
  token: KString,                       // unique per-watch (auto-generated)
  path: KString,                        // node under XenStore
  callback: fn(&Watch, path: &str, token: &str),
  flags: WatchFlags,                    // XBWF_no_err_msg
}
```

Per-call lifecycle `Xs::write(t, dir, node, val)`:
1. Compose `xsd_sockmsg` (XS_WRITE) header.
2. `xs_talkv(t, type, iov, num_vecs, *reply_len)`:
   a. Allocate `xb_req_data` (`req_id` from atomic).
   b. `xs_request_enter(req)` — bump `state_users`, wait if `suspend_active`.
   c. `xb_write` — write to the request ring; `notify_remote_via_evtchn`.
   d. Block on `req.wait` until reply arrives.
   e. Parse reply, return body.

Per-watch fire `xenwatch_thread`:
1. `wait_event(watch_events_waitq, !list_empty(&watch_events))`.
2. Pop one event under `watch_events_lock`.
3. `watch->callback(watch, path, token)` outside spinlock.

Per-event-channel ABI dispatch `xen_evtchn_do_upcall`:
1. Read hypervisor-supplied `vcpu_info`.
2. `evtchn_ops->handle_events(cpu)` — dispatches to `_2l_handle_events` or `_fifo_handle_events`.
3. Each pending port → `info_for_irq(evtchn_to_irq[port])` → `generic_handle_irq`.

Per-bind `bind_evtchn_to_irq(port)`:
1. `irq_mapping_update_lock`.
2. `xen_irq_alloc_desc` if no existing mapping.
3. `set_evtchn_to_irq(port, irq)`.
4. `xen_irq_info_evtchn_setup(info, port, ...)`.
5. `irq_set_chip_and_handler` to `xen_dynamic_chip` or `_lateeoi_chip`.

Per-grant `gnttab_grant_foreign_access(domid, frame, ro)`:
1. Acquire `gnttab_list_lock`.
2. Pop a free `grant_ref_t` from `gnttab_free_list`.
3. Program `shared[ref] = { .full_page.domid = domid, .full_page.frame = frame, .flags = GTF_permit_access | (ro? GTF_readonly : 0) }`.
4. Memory barrier; return `ref`.

XenBus device probe (`xenbus_probe_frontend.c`):
1. Walk `/local/domain/<self_domid>/device` keys.
2. For each device type subdir, for each device id: alloc `xenbus_device`, register on `xen` bus.
3. Driver `.probe` runs: reads frontend keys, switches state to Initialised.
4. Backend (in Dom0) writes its keys + switches state Connected; frontend's state-watch fires, frontend reads + completes connection.

## Hardening

(Inherits row-1 features from `drivers/xen/00-overview.md` § Hardening.)

xenbus-specific reinforcement:

- **Per-request kref** — `xb_req_data` lifetime survives interrupted readers; defense against UAF on cancel.
- **`xs_state_lock` serializes ring writes** — defense against ring-overrun under concurrent transactions.
- **`xs_watch_rwsem` write-locks watch list during suspend** — defense against watch-callback after suspend.
- **`xenwatch_thread` runs at default priority + carries no caller cred** — defense against credential-confusion.
- **`evtchn_to_irq` table indexed under `irq_mapping_update_lock`** — defense against port-irq race during bind/unbind.
- **`xen_lateeoi_chip` for revocable per-domain channels** — defense against IRQ-flood from a hostile peer domain; lateeoi semantics let the handler ack only after work is done.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `xb_req_data`, `xenbus_watch`, `xenbus_device`, `xenbus_file_priv`, and `irq_info`; `/dev/xen/xenbus` user copies bounded by `xsd_sockmsg::len` with `check_object_size`.
- **PAX_KERNEXEC** — `xenbus_frontend_dev_ops`, `xenbus_backend_dev_ops`, `xenbus_dev_fops`, `evtchn_2l_ops`/`evtchn_fifo_ops`, and `xen_dynamic_chip`/`_lateeoi_chip`/`_percpu_chip`/`_pirq_chip` placed in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `xenbus_dev_ioctl`, `xs_talkv`, `xenwatch_thread`, and `xen_evtchn_do_upcall` entries.
- **PAX_REFCOUNT** — saturating refcount on `xb_req_data` (kref), `xenbus_device`, `irq_info`, and per-grant-page refs; overflow trap defeats race UAFs around bind/unbind and grant revoke.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `xb_req_data` (including reply body), `xenbus_watch` (token + path strings), and `irq_info` so stale evtchn ports + watch paths cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on `/dev/xen/xenbus[_backend]` read/write, on every `xs_talkv` reply copy, and on `IOCTL_EVTCHN_*` user payloads.
- **PAX_RAP / kCFI** — `xenbus_driver` callbacks (`.probe`, `.remove`, `.resume`, `.suspend`, `.uevent`), `xenbus_watch::callback`, and `evtchn_ops` vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `xenbus_device`/`irq_info` pointers in `/sys/bus/xen/` behind CAP_SYSLOG; suppress `%p` in xenstore tracepoints.
- **GRKERNSEC_DMESG** — restrict `xenbus: ...` per-device probe + state-transition banners and per-evtchn bind banners to CAP_SYSLOG.
- **`/dev/xen/xenbus_backend` CAP_SYS_ADMIN** — backend-side ioctl requires `CAP_SYS_ADMIN` in init user_ns plus Dom0 privilege; defense against unprivileged backend-impersonation.
- **Watch-path PAX_USERCOPY** — `XS_WATCH` path strings copied through `strndup_user`-equivalent with length bounded by `XENSTORE_PAYLOAD_MAX`; reject embedded NULs and overlong paths.
- **Transaction id rate-limit** — per-domain `xs_request_id` increment paired with a per-process throttle in `xenbus_dev_request_and_reply`; defense against id-exhaustion DoS on the shared ring.
- **Event-channel allowlist** — per-domain `evtchn_to_irq` table caps the channel pool at the ABI maximum; refuse to bind beyond and refuse `bind_interdomain` from a non-allowlisted remote-dom unless lateeoi-armed.
- **Late-EOI mandatory for hostile peers** — frontends bound to untrusted backend domains use `xen_lateeoi_chip` exclusively, ensuring IRQ-flood backpressure is structural rather than policy.

Rationale: XenBus + event-channels + grant-tables are the entire control-plane between a guest and the hypervisor (and between guests). A flaw in the ring framing, a missing PAX_USERCOPY on a watch path, an unbounded interdomain-evtchn bind, or a slow EOI on a hostile-peer channel can be weaponized for VM escape or for cross-VM denial-of-service. RAP/kCFI on driver/watch/abi vtables, saturating refcount on every request/watch/irq, SMAP/PAN on every `/dev/xen/*` user copy, mandatory lateeoi for untrusted peers, and CAP_SYS_ADMIN on the backend dev-node convert XenBus from "the place where everything talks" into a structurally hardened guest↔hypervisor boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- balloon + privcmd (covered in `balloon.md`)
- blkfront/blkback, netfront/netback per-driver Tier-3
- gntdev + gntalloc (covered in `gntdev.md` future Tier-3)
- PCI passthrough (covered in `xen-pciback.md` future Tier-3)
- 32-bit guest paths — Rookery targets 64-bit guests only
- Implementation code
