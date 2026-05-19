# Tier-3: drivers/usb/core/devio.c — usbfs `/dev/bus/usb/<bus>/<dev>` UAPI (USBDEVFS_* ioctls, urb submit/reap)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/core/devio.c (~2930 lines)
  - include/uapi/linux/usbdevice_fs.h (USBDEVFS_* ioctls, struct usbdevfs_urb)
  - include/linux/usb/hcd.h (usb_dev_state internals)
  - drivers/usb/core/usb.h (struct usb_dev_state)
-->

## Summary

`drivers/usb/core/devio.c` implements **usbfs** — the per-device character interface at `/dev/bus/usb/<bus>/<dev>` (major `USB_DEVICE_MAJOR = 189`) that allows **userspace USB drivers** (libusb, libusbx, libusb-1.0, gPhoto2, sane-backends, fwupd, brltty, ColorHug, USB-flashers, the entire embedded-bring-up ecosystem) to talk to USB devices without an in-kernel class driver. The interface is one character device per `usb_device` (allocated dynamically on hotplug via `usb_register_notify` ⟶ `usbdev_notify(USB_DEVICE_ADD)`), with file_operations dispatching to ~40 `USBDEVFS_*` ioctls plus `read`, `mmap`, and `poll`.

Per-`struct usb_dev_state` (one allocation per `open(2)`) holds: the `usb_device *dev` (with refcount held), the file pointer, the per-state `spinlock_t lock` covering the async lists, the `async_pending` + `async_completed` + `memory_list` linked lists, the `wait` waitqueue (URB reapers block here), the `wait_for_resume` waitqueue (`USBDEVFS_WAIT_FOR_RESUME` blocks here), the `discsignr` + `disc_pid` + `disccontext` for `USBDEVFS_DISCSIGNAL` (signal to user on disconnect), the `ifclaimed` bitmask of interfaces claimed by this fd, `disabled_bulk_eps` (for `USBDEVFS_URB_BULK_CONTINUATION` error-stop semantics), `interface_allowed_mask` (`USBDEVFS_DROP_PRIVILEGES` mask of permitted interfaces), `not_yet_resumed`, `suspend_allowed`, and `privileges_dropped`.

Per-`struct async` (one per submitted async URB) holds: the `asynclist` link (pending or completed), the owning `usb_dev_state *ps`, the submitting `pid` + `cred`, the per-URB `signr` (for SIGIO-style async-IO notification via `kill_pid_usb_asyncio`), the `userbuffer` + `userurb` userland addresses, the `urb *` itself, an optional `usb_memory *usbm` reference into the mmap region, the recorded `mem_usage` for accounting, the bulk endpoint shadow `bulk_addr` + `bulk_status` (for continuation semantics), and the completion `status`.

Per-`proc_do_submiturb` is the central **async-submit** path. The caller passes a `struct usbdevfs_urb { type, endpoint, status, flags, buffer, buffer_length, actual_length, start_frame, number_of_packets|stream_id, error_count, signr, usercontext, iso_frame_desc[] }`. The kernel: validates the flags mask (`SHORT_NOT_OK | BULK_CONTINUATION | NO_FSBR | ZERO_PACKET | NO_INTERRUPT`, plus `ISO_ASAP` for ISO type), checks `buffer_length < USBFS_XFER_MAX = UINT_MAX/2 - 1000000`, resolves the endpoint via `findintfep` + `ep_to_host_endpoint`, enforces `checkintf` (interface must be claimed by this fd), branches on `USBDEVFS_URB_TYPE_{CONTROL,BULK,INTERRUPT,ISO}` (with the special case that a BULK against an INT endpoint is auto-promoted to INTERRUPT), accounts for memory under the global `usbfs_memory_mb` cap, builds scatter-gather chains for large bulk transfers (`USB_SG_SIZE = 16384` per chunk, up to `bus.sg_tablesize`), copies OUT buffers from userspace (or maps via the mmap area if `find_memory_area` returned one), constructs the `struct urb` with the kernel-internal `URB_*` flags translated from the UAPI `USBDEVFS_URB_*` bits, sets the completion callback to `async_completed`, calls `async_newpending` to link it to `ps->async_pending`, and finally `usb_submit_urb(GFP_ATOMIC for bulk under ps->lock, else GFP_KERNEL)`.

Per-`async_completed` is the URB completion callback — runs in HCD softirq context. It moves the `async` from `async_pending` to `async_completed`, stamps `as->status = urb->status`, latches the `signr` + `userurb_sigval` + `pid` + `cred` if a signal was requested, snoops the buffer if `usbfs_snoop` is on, cancels companion BULK_CONTINUATION URBs on error via `cancel_bulk_urbs`, `wake_up(&ps->wait)` to unblock any reaper, and finally outside the spinlock calls `kill_pid_usb_asyncio` to deliver the signal.

Per-`proc_reapurb` is the blocking **reap** path. It blocks on `ps->wait` (sleeping in TASK_INTERRUPTIBLE, releasing the device lock around `schedule()`), pulls an `async` off `async_completed`, calls `processcompl` to copy `actual_length` / `error_count` / per-ISO-packet `actual_length`+`status` back to userland and to copy the IN buffer (`copy_urb_data_to_user`), `put_user`s the `userurb` pointer at `*arg` (giving userland the cookie to identify which URB completed), and frees the `async`. `proc_reapurbnonblock` is the non-blocking sibling (`-EAGAIN` if nothing ready).

Per-`USBDEVFS_DISCARDURB` calls `proc_unlinkurb` to look up an in-flight URB by user-cookie via `async_getpending` and `usb_kill_urb` it. Per-`USBDEVFS_DISCSIGNAL` registers a per-fd signal to be delivered to the submitter on device disconnect (per the CAN-2005-3055 fix history — credentials captured at submit time, not delivery time).

Per-`USBDEVFS_CLAIM_INTERFACE` / `RELEASE_INTERFACE` claims an interface for usbfs via `usb_driver_claim_interface(&usbfs_driver, intf, ps)` (using the same kernel-driver-binding mechanism but with a stub `usbfs_driver` that has `driver_probe` returning -ENODEV — i.e. it never actually probes anything, it's just a placeholder owner). `USBDEVFS_DISCONNECT` (via `proc_ioctl`) calls `usb_driver_release_interface` on the interface's current driver to evict a kernel driver from an interface so userspace can claim it (the classic "unbind printer from usblp first, then libusb can grab it"). `USBDEVFS_CONNECT` re-attaches.

Per-`USBDEVFS_DISCONNECT_CLAIM` is the atomic compose: under one `usb_lock_device`, optionally evict an existing kernel driver (filtered by name via `USBDEVFS_DISCONNECT_CLAIM_IF_DRIVER` or `_EXCEPT_DRIVER`), then `claimintf` — closes the userspace race window between disconnect-and-claim.

Per-`USBDEVFS_DROP_PRIVILEGES` is the post-claim hardening: userland writes an `interface_allowed_mask` bitmask of permitted interfaces, and `privileges_dropped = true` is latched — subsequent claim attempts on other interfaces, and ALL `USBDEVFS_IOCTL` (driver-ioctl proxy) calls, return `-EACCES`. Per-`USBDEVFS_GET_CAPABILITIES` returns the kernel's feature mask: `CAP_ZERO_PACKET | CAP_NO_PACKET_SIZE_LIM | CAP_BULK_CONTINUATION | CAP_REAP_AFTER_DISCONNECT | CAP_BULK_SCATTER_GATHER | CAP_MMAP | CAP_DROP_PRIVILEGES | CAP_CONNINFO_EX | CAP_SUSPEND`.

Per-`usbdev_mmap` maps a coherent-DMA region (allocated via `usb_alloc_coherent`) into the user's address space so subsequent SUBMITURBs whose `buffer` pointer falls inside the mapped region skip the copy_from_user/copy_to_user — the kernel transfer-DMA goes directly into the mapped buffer. Per-`usbdev_read` returns the device's serialized descriptors (per the deprecated `/proc/bus/usb/<bus>/<dev>` interface kept for libusb-0.x compat).

Per-`USBDEVFS_RESET` calls `proc_resetdevice` ⟶ `usb_reset_device`; per-`USBDEVFS_CLEAR_HALT` ⟶ `usb_clear_halt` for the specified endpoint; per-`USBDEVFS_RESETEP` is the deprecated cousin of CLEAR_HALT. Per-`USBDEVFS_SETCONFIGURATION` ⟶ `usb_set_configuration`; per-`USBDEVFS_SETINTERFACE` ⟶ `usb_set_interface`. Per-`USBDEVFS_ALLOC_STREAMS` / `_FREE_STREAMS` plumb USB-3 bulk-stream allocation (`usb_alloc_streams` / `usb_free_streams`).

Per-`usbdev_notify` listens on `usb_register_notify` for `USB_DEVICE_REMOVE` and tears down all per-fd state (`destroy_all_async`, `wake_up_all`, signal delivery via `kill_pid_usb_asyncio`, list removal). After disconnect, REAP ioctls still succeed (`CAP_REAP_AFTER_DISCONNECT`) so userland can drain pending completions.

Per-CVE-history: `CAN-2005-3055` ("Fix user-triggerable oops in async URB delivery") is explicitly called out in the file header — fixed by capturing pid+cred at submit-time (`get_pid(task_pid(current))`, `get_current_cred()`) and using them at completion-time, not the running task's. The credentials accounting (`get_cred` / `put_cred`) is a security-critical invariant.

Critical for: libusb / libusb-1.0 (every userspace USB tool — `lsusb -v`, scanner drivers, USB programmers, biometric tools, fwupd firmware updates), gPhoto2 cameras, sane scanners, brltty Braille displays, the embedded-bring-up workflow (`dfu-util`, `dfu-programmer`, `openocd` USB-JTAG), the entire ecosystem of devices that have no kernel class driver.

This Tier-3 covers `drivers/usb/core/devio.c` (~2930 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct usb_dev_state` | per-fd state | `UsbDevState` |
| `struct async` | per-async-URB tracking | `UsbAsync` |
| `struct usb_memory` | per-mmap-region | `UsbMemory` |
| `usbfs_driver` | stub `usb_driver` used as interface-claim owner | `UsbDevio::USBFS_DRIVER` |
| `usbdev_file_operations` | char-dev fops | `UsbDevio::FILE_OPS` |
| `usbdev_open()` | per-open(/dev/bus/usb/.../...) | `UsbDevio::open` |
| `usbdev_release()` | per-close | `UsbDevio::release` |
| `usbdev_read()` | per-read (descriptors) | `UsbDevio::read` |
| `usbdev_mmap()` | per-mmap coherent DMA | `UsbDevio::mmap` |
| `usbdev_poll()` | per-poll | `UsbDevio::poll` |
| `usbdev_ioctl()` / `usbdev_do_ioctl()` | per-ioctl dispatch | `UsbDevio::ioctl` / `do_ioctl` |
| `usbdev_lookup_by_devt()` | per-devt → usb_device | `UsbDevio::lookup_by_devt` |
| `usbdev_notify()` | per-USB_DEVICE_REMOVE listener | `UsbDevio::notify` |
| `usbdev_remove()` | per-device tear-down (notify-callback) | `UsbDevio::remove` |
| `usbfs_increase_memory_usage()` / `_decrease_memory_usage()` | per-`usbfs_memory_mb` cap accounting | `UsbDevio::mem_inc` / `mem_dec` |
| `connected()` | per-`udev.state != NOTATTACHED` test | `UsbDevState::connected` |
| `alloc_async()` / `free_async()` | per-async lifecycle | `UsbAsync::alloc` / `free` |
| `async_newpending()` / `async_removepending()` | per-list-add (pending) | `UsbAsync::new_pending` / `remove_pending` |
| `async_getcompleted()` | per-completed-pop | `UsbAsync::get_completed` |
| `async_getpending()` | per-cookie lookup | `UsbAsync::get_pending` |
| `async_completed()` | URB completion callback | `UsbAsync::completed` |
| `destroy_async()` / `destroy_async_on_interface()` / `destroy_all_async()` | per-cancel-all | `UsbAsync::destroy` / `destroy_on_interface` / `destroy_all` |
| `reap_as()` | per-blocking-wait for completion | `UsbAsync::reap` |
| `snoop_urb()` / `snoop_urb_data()` | per-`usbfs_snoop` traffic log | `UsbDevio::snoop_urb` / `snoop_urb_data` |
| `copy_urb_data_to_user()` | per-IN-buffer copyout | `UsbDevio::copy_data_to_user` |
| `cancel_bulk_urbs()` | per-error bulk-stream cancel | `UsbDevio::cancel_bulk_urbs` |
| `driver_probe()` / `driver_disconnect()` / `driver_suspend()` / `driver_resume()` | stub usbfs_driver ops | `UsbDevio::usbfs_driver_*` |
| `claimintf()` / `releaseintf()` | per-interface claim/release helpers | `UsbDevState::claim_intf` / `release_intf` |
| `checkintf()` | per-interface-claimed-by-fd assert | `UsbDevState::check_intf` |
| `findintfep()` | per-endpoint → ifnum lookup | `UsbDevio::find_intf_ep` |
| `ep_to_host_endpoint()` | per-(udev, ep_addr) → usb_host_endpoint | `UsbDevio::ep_to_host_endpoint` |
| `check_ctrlrecip()` | per-CONTROL recipient (DEVICE/INTERFACE/ENDPOINT) policy | `UsbDevio::check_ctrl_recip` |
| `parse_usbdevfs_streams()` | per-USB-3 streams arg parse | `UsbDevio::parse_streams` |
| `do_proc_control()` / `proc_control()` | per-USBDEVFS_CONTROL impl | `UsbDevio::control` |
| `do_proc_bulk()` / `proc_bulk()` | per-USBDEVFS_BULK impl | `UsbDevio::bulk` |
| `proc_resetep()` | per-USBDEVFS_RESETEP | `UsbDevio::resetep` |
| `proc_clearhalt()` | per-USBDEVFS_CLEAR_HALT | `UsbDevio::clearhalt` |
| `proc_resetdevice()` | per-USBDEVFS_RESET | `UsbDevio::resetdevice` |
| `proc_setintf()` / `proc_setconfig()` | per-SETINTERFACE / SETCONFIGURATION | `UsbDevio::setintf` / `setconfig` |
| `proc_getdriver()` | per-USBDEVFS_GETDRIVER | `UsbDevio::getdriver` |
| `proc_connectinfo()` / `proc_conninfo_ex()` | per-CONNECTINFO / CONNINFO_EX | `UsbDevio::connectinfo` / `conninfo_ex` |
| `proc_do_submiturb()` / `proc_submiturb()` | per-USBDEVFS_SUBMITURB core | `UsbDevio::submiturb` |
| `proc_unlinkurb()` | per-USBDEVFS_DISCARDURB | `UsbDevio::unlinkurb` |
| `processcompl()` | per-async copy-to-user | `UsbDevio::process_completion` |
| `proc_reapurb()` / `proc_reapurbnonblock()` | per-REAPURB / REAPURBNDELAY | `UsbDevio::reapurb` / `reapurb_nonblock` |
| `proc_disconnectsignal()` | per-USBDEVFS_DISCSIGNAL | `UsbDevio::discsignal` |
| `proc_claiminterface()` / `proc_releaseinterface()` | per-CLAIMINTERFACE / RELEASEINTERFACE | `UsbDevio::claim_interface` / `release_interface` |
| `proc_ioctl()` / `proc_ioctl_default()` | per-USBDEVFS_IOCTL (driver proxy + DISCONNECT/CONNECT) | `UsbDevio::driver_ioctl` |
| `proc_claim_port()` / `proc_release_port()` | per-CLAIM_PORT / RELEASE_PORT (hub) | `UsbDevio::claim_port` / `release_port` |
| `proc_get_capabilities()` | per-USBDEVFS_GET_CAPABILITIES | `UsbDevio::get_capabilities` |
| `proc_disconnect_claim()` | per-USBDEVFS_DISCONNECT_CLAIM | `UsbDevio::disconnect_claim` |
| `proc_alloc_streams()` / `proc_free_streams()` | per-USB-3 streams | `UsbDevio::alloc_streams` / `free_streams` |
| `proc_drop_privileges()` | per-USBDEVFS_DROP_PRIVILEGES | `UsbDevio::drop_privileges` |
| `proc_forbid_suspend()` / `proc_allow_suspend()` / `proc_wait_for_resume()` | per-FORBID / ALLOW / WAIT_FOR_RESUME | `UsbDevio::forbid_suspend` / `allow_suspend` / `wait_for_resume` |
| `usbfs_notify_suspend()` / `usbfs_notify_resume()` | per-PM-event broadcast to all fds | `UsbDevio::notify_suspend` / `notify_resume` |
| `usbfs_blocking_completion()` / `usbfs_start_wait_urb()` | per-sync-URB helper (used by CONTROL/BULK) | `UsbDevio::blocking_completion` / `start_wait_urb` |
| `find_memory_area()` | per-mmap-region lookup | `UsbDevio::find_memory_area` |
| `usbdev_vm_open()` / `usbdev_vm_close()` | per-mmap VMA refcount ops | `UsbDevio::vm_open` / `vm_close` |
| `usb_devio_init()` / `usb_devio_cleanup()` | per-cdev register / unregister | `UsbDevio::init` / `cleanup` |
| compat32 helpers: `get_urb32`, `proc_submiturb_compat`, `processcompl_compat`, `proc_reapurb_compat`, `proc_reapurbnonblock_compat`, `proc_control_compat`, `proc_bulk_compat`, `proc_disconnectsignal_compat`, `proc_ioctl_compat` | per-32-on-64 compat-layer | (CompatLayer if needed) |

## Compatibility contract

REQ-1: struct usb_dev_state:
- list: per-state list link (in `udev.filelist`).
- dev: per-`usb_device *` (refcount held).
- file: per-`struct file *`.
- lock: per-spinlock protecting async_pending / async_completed / memory_list.
- async_pending: per-list of submitted-not-yet-completed `struct async`.
- async_completed: per-list of completed-not-yet-reaped `struct async`.
- memory_list: per-list of mmap'd `struct usb_memory` regions.
- wait: per-waitqueue for REAPURB blockers.
- wait_for_resume: per-waitqueue for WAIT_FOR_RESUME blockers.
- discsignr: per-fd signal on disconnect (0 = none).
- disc_pid + disccontext + cred: per-fd disconnect-signal target.
- ifclaimed: per-bitmask of interface-numbers claimed by this fd.
- disabled_bulk_eps: per-bitmask of bulk endpoints disabled by error (for BULK_CONTINUATION).
- interface_allowed_mask: per-DROP_PRIVILEGES whitelist of allowed interface-numbers.
- not_yet_resumed: per-FORBID_SUSPEND state (1 = still suspended, 0 = resumed).
- suspend_allowed: per-FORBID/ALLOW_SUSPEND state.
- privileges_dropped: per-DROP_PRIVILEGES latch.

REQ-2: struct async:
- asynclist: per-list link (pending or completed).
- ps: per-owning `usb_dev_state *`.
- pid + cred: per-submitter (captured at submit-time, per CAN-2005-3055).
- signr: per-URB signal on completion.
- ifnum: per-target interface (for `destroy_async_on_interface`).
- userbuffer: per-userland IN buffer (for read-back copyout).
- userurb: per-userland `struct usbdevfs_urb *` cookie.
- userurb_sigval: per-sival_ptr (or sival_int for 32-bit compat).
- urb: per-`struct urb *`.
- usbm: per-`struct usb_memory *` if buffer is in mmap'd region (no copy).
- mem_usage: per-bytes accounted under `usbfs_memory_mb`.
- status: per-URB completion status.
- bulk_addr, bulk_status: per-BULK_CONTINUATION endpoint-disable shadow.

REQ-3: struct usb_memory:
- memlist: per-list link in `usb_dev_state.memory_list`.
- vma_use_count: per-VMA refcount.
- urb_use_count: per-URB refcount (URBs currently using this region).
- size: per-region bytes.
- mem: per-CPU-virtual pointer (from `usb_alloc_coherent`).
- dma_handle: per-DMA address.
- vm_start: per-userland VMA base.
- ps: per-owning `usb_dev_state *`.

REQ-4: usbdev_open(inode, file):
- ps = kzalloc(struct usb_dev_state); on NULL: -ENOMEM.
- if imajor(inode) == USB_DEVICE_MAJOR: dev = usbdev_lookup_by_devt(inode.i_rdev).
- if !dev: -ENODEV.
- usb_lock_device(dev).
- if dev.state == NOTATTACHED: goto unlock; -ENODEV.
- usb_autoresume_device(dev) on error: -ENODEV.
- ps.dev = dev; ps.file = file; ps.interface_allowed_mask = 0xFFFFFFFF (all 32 interfaces allowed pre-DROP_PRIVILEGES).
- spin_lock_init; INIT_LIST_HEAD on list, async_pending, async_completed, memory_list.
- init_waitqueue_head on wait, wait_for_resume.
- ps.disc_pid = get_pid(task_pid(current)); ps.cred = get_current_cred().
- smp_wmb.
- list_add_tail(&ps.list, &dev.filelist).
- file.private_data = ps.
- usb_unlock_device(dev).

REQ-5: usbdev_release(inode, file):
- ps = file.private_data; dev = ps.dev.
- usb_lock_device(dev).
- usb_hub_release_all_ports(dev, ps) — release any USBDEVFS_CLAIM_PORT'd ports.
- mutex_lock(&usbfs_mutex); list_del_init(&ps.list); mutex_unlock.
- For each set bit in ifclaimed: releaseintf(ps, ifnum).
- destroy_all_async(ps).
- if !ps.suspend_allowed: usb_autosuspend_device(dev).
- usb_unlock_device(dev).
- usb_put_dev(dev); put_pid(ps.disc_pid); put_cred(ps.cred).
- Drain async_completed (free_async each); kfree(ps).

REQ-6: claimintf(ps, ifnum):
- if ifnum ≥ 8*sizeof(ifclaimed): -EINVAL.
- if already claimed: return 0.
- if ps.privileges_dropped ∧ !test_bit(ifnum, &ps.interface_allowed_mask): -EACCES.
- intf = usb_ifnum_to_if(dev, ifnum); on NULL: -ENOENT.
- /* Suppress uevents during claim */
- old_suppress = dev_get_uevent_suppress(&intf.dev); dev_set_uevent_suppress(&intf.dev, 1).
- err = usb_driver_claim_interface(&usbfs_driver, intf, ps).
- dev_set_uevent_suppress(&intf.dev, old_suppress).
- if !err: set_bit(ifnum, &ps.ifclaimed).

REQ-7: releaseintf(ps, ifnum):
- Symmetric to claim — clear_bit(ifclaimed); usb_driver_release_interface(&usbfs_driver, intf).

REQ-8: checkintf(ps, ifnum):
- if !test_bit(ifnum, &ps.ifclaimed): -EPERM (caller did not claim this interface).

REQ-9: usbfs_increase_memory_usage(amount):
- guard against `total_mem` overflow.
- if usbfs_memory_mb > 0 ∧ total_mem + amount > usbfs_memory_mb*MB: -ENOMEM.
- usbfs_memory_usage += amount.
- return 0.

REQ-10: proc_do_submiturb(ps, uurb, iso_frame_desc, arg, sigval) (USBDEVFS_SUBMITURB core):
- /* Flag-mask sanity */
- mask = SHORT_NOT_OK | BULK_CONTINUATION | NO_FSBR | ZERO_PACKET | NO_INTERRUPT.
- if type == ISO: mask |= ISO_ASAP.
- if uurb.flags & ~mask: -EINVAL.
- if uurb.buffer_length ≥ USBFS_XFER_MAX: -EINVAL.
- if uurb.buffer_length > 0 ∧ !uurb.buffer: -EINVAL.
- /* Non-default-control-EP needs interface claim */
- if !(type == CONTROL ∧ ep == 0): ifnum = findintfep(dev, uurb.endpoint); checkintf(ps, ifnum).
- ep = ep_to_host_endpoint(dev, uurb.endpoint); on NULL: -ENOENT.
- /* Per-type branch */
- switch type:
  - CONTROL: usb_endpoint_xfer_control(&ep.desc); buffer_length ≥ 8; dr = kmalloc(usb_ctrlrequest); copy_from_user(dr, uurb.buffer, 8); buffer_length ≥ wLength+8; check_ctrlrecip(ps, bRequestType, bRequest, wIndex); buffer_length = wLength; buffer += 8; is_in = (bRequestType & USB_DIR_IN) != 0.
  - BULK: switch (usb_endpoint_type(&ep.desc)): CONTROL/ISO → -EINVAL; INT → promote type to INTERRUPT, fallthrough; default: num_sgs = DIV_ROUND_UP(buffer_length, USB_SG_SIZE); if num_sgs == 1 ∨ > bus.sg_tablesize: num_sgs = 0; if ep.streams: stream_id = uurb.stream_id.
  - INTERRUPT: usb_endpoint_xfer_int(&ep.desc); allow_zero (OUT) or allow_short (IN).
  - ISO: 1 ≤ number_of_packets ≤ 128; usb_endpoint_xfer_isoc(&ep.desc); copy iso_frame_desc array via memdup_user; each isopkt.length ≤ 98304; totlen = sum(lengths); buffer_length = totlen.
  - default: -EINVAL.
- /* User-buffer access check */
- if buffer_length > 0 ∧ !access_ok(uurb.buffer, buffer_length): -EFAULT.
- as = alloc_async(number_of_packets); on NULL: -ENOMEM.
- as.usbm = find_memory_area(ps, uurb).
- /* Account memory; build SG or transfer_buffer */
- usbfs_increase_memory_usage(...).
- if num_sgs: build SG table; for each chunk: kmalloc + copy_from_user if OUT.
- else if buffer_length > 0: use as.usbm if present, else kmalloc + copy_from_user (OUT) or memset 0 for ISO IN (anti-leak).
- /* Build urb */
- as.urb.dev = ps.dev.
- as.urb.pipe = (uurb.type << 30) | __create_pipe(dev, ep & 0xf) | (ep & USB_DIR_IN).
- /* Translate UAPI flags → kernel URB_* flags */
- u = is_in ? URB_DIR_IN : URB_DIR_OUT.
- if flags & ISO_ASAP: u |= URB_ISO_ASAP.
- if allow_short ∧ flags & SHORT_NOT_OK: u |= URB_SHORT_NOT_OK.
- if allow_zero ∧ flags & ZERO_PACKET: u |= URB_ZERO_PACKET.
- if flags & NO_INTERRUPT: u |= URB_NO_INTERRUPT.
- as.urb.transfer_flags = u.
- as.urb.transfer_buffer_length = buffer_length.
- as.urb.setup_packet = (u8*) dr; dr = NULL.
- as.urb.start_frame = uurb.start_frame; .number_of_packets = number_of_packets; .stream_id = stream_id.
- /* ISO/HS/SS interval coding (1 << (interval-1) clamped to 15) */
- if ep.desc.bInterval: as.urb.interval = encoded.
- as.urb.context = as; as.urb.complete = async_completed.
- /* ISO frame offsets */
- for u in 0..number_of_packets: as.urb.iso_frame_desc[u].offset/length = totlen += isopkt[u].length.
- as.ps = ps; as.userurb = arg; as.userurb_sigval = sigval.
- if as.usbm: transfer_flags |= URB_NO_TRANSFER_DMA_MAP; transfer_dma = usbm.dma_handle + (uurb.buffer - usbm.vm_start).
- else if is_in ∧ buffer_length > 0: as.userbuffer = uurb.buffer.
- as.signr = uurb.signr; as.ifnum = ifnum.
- /* Per-CAN-2005-3055: capture pid/cred at submit-time */
- as.pid = get_pid(task_pid(current)); as.cred = get_current_cred().
- snoop_urb(..., SUBMIT, ...).
- async_newpending(as).
- /* Submit — bulk under ps.lock for CONTINUATION semantics */
- if usb_endpoint_xfer_bulk(&ep.desc):
  - spin_lock_irq(&ps.lock).
  - as.bulk_addr = usb_endpoint_num(&ep.desc) | ((bEndpointAddress & DIR_MASK) >> 3).
  - if flags & BULK_CONTINUATION: as.bulk_status = AS_CONTINUATION.
  - else: ps.disabled_bulk_eps &= ~(1 << as.bulk_addr).
  - if ps.disabled_bulk_eps & (1 << as.bulk_addr): ret = -EREMOTEIO.
  - else: ret = usb_submit_urb(as.urb, GFP_ATOMIC).
  - spin_unlock_irq.
- else: ret = usb_submit_urb(as.urb, GFP_KERNEL).
- if ret: snoop_urb(..., ret, COMPLETE, ...); async_removepending; error-path; return ret.
- return 0.

REQ-11: async_completed(urb) (HCD softirq callback):
- as = urb.context; ps = as.ps.
- spin_lock_irqsave(&ps.lock).
- list_move_tail(&as.asynclist, &ps.async_completed).
- as.status = urb.status.
- if as.signr: latch pid + cred + errno + sival.
- snoop + snoop_urb_data (IN).
- if as.status < 0 ∧ as.bulk_addr ∧ as.status ∉ {-ECONNRESET, -ENOENT}: cancel_bulk_urbs(ps, as.bulk_addr).
- wake_up(&ps.wait).
- spin_unlock_irqrestore.
- if as.signr: kill_pid_usb_asyncio(signr, errno, addr, pid, cred); put_pid; put_cred.

REQ-12: cancel_bulk_urbs(ps, bulk_addr):
- /* Releases + re-acquires ps.lock */
- ps.disabled_bulk_eps |= (1 << bulk_addr).
- For each pending `as` with as.bulk_addr == bulk_addr ∧ as.bulk_status != AS_CONTINUATION: usb_unlink_urb(as.urb).
- /* (BULK_CONTINUATION URBs already linked are left to fail naturally) */

REQ-13: reap_as(ps) (blocking wait for one completion):
- DECLARE_WAITQUEUE(wait, current); add_wait_queue(&ps.wait, &wait).
- loop:
  - __set_current_state(TASK_INTERRUPTIBLE).
  - as = async_getcompleted(ps).
  - if as ∨ !connected(ps): break.
  - if signal_pending(current): break.
  - usb_unlock_device(dev); schedule(); usb_lock_device(dev).
- remove_wait_queue.
- set_current_state(TASK_RUNNING).
- return as.

REQ-14: proc_reapurb(ps, arg):
- as = reap_as(ps).
- if as: retval = processcompl(as, arg); free_async; return retval.
- if signal_pending: -EINTR.
- return -ENODEV.

REQ-15: proc_reapurbnonblock(ps, arg):
- as = async_getcompleted(ps).
- if as: retval = processcompl(as, arg); free_async; return retval.
- return connected(ps) ? -EAGAIN : -ENODEV.

REQ-16: processcompl(as, arg):
- urb = as.urb; userurb = as.userurb.
- compute_isochronous_actual_length(urb).
- if as.userbuffer ∧ urb.actual_length: copy_urb_data_to_user(userbuffer, urb).
- put_user(as.status, &userurb.status).
- put_user(urb.actual_length, &userurb.actual_length).
- put_user(urb.error_count, &userurb.error_count).
- if usb_endpoint_xfer_isoc(&urb.ep.desc): for each packet: put_user actual_length + status.
- put_user(userurb, arg) — give userland the cookie identifying which URB completed.

REQ-17: proc_unlinkurb(ps, arg) (USBDEVFS_DISCARDURB):
- spin_lock_irqsave(&ps.lock).
- as = async_getpending(ps, arg).   /* lookup by user-cookie */
- if !as: -EINVAL.
- urb = as.urb; usb_get_urb(urb).
- spin_unlock_irqrestore.
- usb_kill_urb(urb); usb_put_urb(urb).

REQ-18: proc_disconnectsignal(ps, arg) (USBDEVFS_DISCSIGNAL):
- Copy `usbdevfs_disconnectsignal { signr, context }` from user.
- ps.discsignr = ds.signr.
- ps.disccontext.sival_ptr = ds.context.

REQ-19: proc_claiminterface(ps, arg):
- copy ifnum from user.
- claimintf(ps, ifnum).

REQ-20: proc_releaseinterface(ps, arg):
- copy ifnum from user.
- releaseintf(ps, ifnum).
- destroy_async_on_interface(ps, ifnum) — flush in-flight URBs against that interface.

REQ-21: proc_ioctl(ps, ctl) (USBDEVFS_IOCTL — driver-ioctl proxy):
- if ps.privileges_dropped: -EACCES (DROP_PRIVILEGES gates ALL driver ioctls).
- if !connected(ps): -ENODEV.
- size = _IOC_SIZE(ctl.ioctl_code); allocate + copy_from_user if _IOC_WRITE direction.
- if dev.state != CONFIGURED: -EHOSTUNREACH.
- intf = usb_ifnum_to_if(dev, ctl.ifno); on NULL: -EINVAL.
- switch ioctl_code:
  - USBDEVFS_DISCONNECT: if intf.dev.driver: usb_driver_release_interface(driver, intf); else: -ENODATA.
  - USBDEVFS_CONNECT: if !intf.dev.driver: device_attach(&intf.dev); else: -EBUSY.
  - default: if intf.dev.driver ∧ driver.unlocked_ioctl: driver.unlocked_ioctl(intf, code, buf); else: -ENOTTY.
- copy_to_user back if _IOC_READ direction and size > 0.

REQ-22: proc_disconnect_claim(ps, arg) (USBDEVFS_DISCONNECT_CLAIM):
- Copy `usbdevfs_disconnect_claim { interface, flags, driver }` from user.
- intf = usb_ifnum_to_if(dev, dc.interface).
- if intf.dev.driver:
  - if ps.privileges_dropped: -EACCES.
  - if (dc.flags & DISCONNECT_CLAIM_IF_DRIVER) ∧ strncmp(dc.driver, intf.dev.driver.name) != 0: -EBUSY.
  - if (dc.flags & DISCONNECT_CLAIM_EXCEPT_DRIVER) ∧ strncmp == 0: -EBUSY.
  - usb_driver_release_interface(driver, intf).
- claimintf(ps, dc.interface).

REQ-23: proc_drop_privileges(ps, arg) (USBDEVFS_DROP_PRIVILEGES):
- Copy `u32 mask` from user.
- ps.interface_allowed_mask = mask.
- ps.privileges_dropped = true.

REQ-24: proc_get_capabilities(ps, arg) (USBDEVFS_GET_CAPABILITIES):
- caps = CAP_ZERO_PACKET | CAP_NO_PACKET_SIZE_LIM | CAP_REAP_AFTER_DISCONNECT | CAP_MMAP | CAP_DROP_PRIVILEGES | CAP_CONNINFO_EX | MAYBE_CAP_SUSPEND.
- if !bus.no_stop_on_short: caps |= CAP_BULK_CONTINUATION.
- if bus.sg_tablesize: caps |= CAP_BULK_SCATTER_GATHER.
- put_user(caps, arg).

REQ-25: proc_alloc_streams(ps, arg) / proc_free_streams(ps, arg):
- parse_usbdevfs_streams(ps, arg, ...).
- destroy_async_on_interface(ps, intf.altsetting[0].desc.bInterfaceNumber).
- usb_alloc_streams / usb_free_streams.

REQ-26: usbdev_do_ioctl(file, cmd, p) — top-level switch:
- usb_lock_device(dev).
- /* REAPURB family allowed even after disconnect */
- switch cmd: USBDEVFS_REAPURB → proc_reapurb; USBDEVFS_REAPURBNDELAY → proc_reapurbnonblock; (compat: REAPURB32 / REAPURBNDELAY32).
- if !connected(ps): unlock; -ENODEV.
- switch cmd:
  - CONTROL → proc_control.
  - BULK → proc_bulk.
  - RESETEP → proc_resetep.
  - RESET → proc_resetdevice.
  - CLEAR_HALT → proc_clearhalt.
  - GETDRIVER → proc_getdriver.
  - CONNECTINFO → proc_connectinfo.
  - SETINTERFACE → proc_setintf.
  - SETCONFIGURATION → proc_setconfig.
  - SUBMITURB → proc_submiturb.
  - DISCARDURB → proc_unlinkurb.
  - DISCSIGNAL → proc_disconnectsignal.
  - CLAIMINTERFACE → proc_claiminterface.
  - RELEASEINTERFACE → proc_releaseinterface.
  - IOCTL → proc_ioctl_default.
  - CLAIM_PORT / RELEASE_PORT → proc_claim_port / proc_release_port.
  - GET_CAPABILITIES → proc_get_capabilities.
  - DISCONNECT_CLAIM → proc_disconnect_claim.
  - ALLOC_STREAMS / FREE_STREAMS → proc_alloc_streams / proc_free_streams.
  - DROP_PRIVILEGES → proc_drop_privileges.
  - GET_SPEED → ps.dev.speed (returned as ioctl ret).
  - FORBID_SUSPEND / ALLOW_SUSPEND / WAIT_FOR_RESUME → proc_forbid_suspend / proc_allow_suspend / proc_wait_for_resume.
  - (compat: CONTROL32, BULK32, DISCSIGNAL32, SUBMITURB32, IOCTL32).
- variable-length: CONNINFO_EX → proc_conninfo_ex.
- usb_unlock_device.

REQ-27: usbdev_notify(self, action, dev) (USB notifier):
- ADD: no-op (cdev region is global; per-device char-dev appears via udev).
- REMOVE: usbdev_remove(dev).

REQ-28: usbdev_remove(udev):
- mutex_lock(&usbfs_mutex).
- For each ps in udev.filelist:
  - destroy_all_async(ps); wake_up_all(&ps.wait); WRITE_ONCE(ps.not_yet_resumed, 0); wake_up_all(&ps.wait_for_resume); list_del_init(&ps.list).
  - if ps.discsignr: kill_pid_usb_asyncio(discsignr, EPIPE, disccontext, disc_pid, cred).
- mutex_unlock.

REQ-29: usb_devio_init():
- register_chrdev_region(USB_DEVICE_DEV = MKDEV(USB_DEVICE_MAJOR, 0), USB_DEVICE_MAX = 64*128, "usb_device").
- cdev_init(&usb_device_cdev, &usbdev_file_operations).
- cdev_add(&usb_device_cdev, USB_DEVICE_DEV, USB_DEVICE_MAX).
- usb_register_notify(&usbdev_nb).

REQ-30: usb_devio_cleanup():
- usb_unregister_notify; cdev_del; unregister_chrdev_region.

REQ-31: usbdev_poll(file, wait):
- poll_wait(file, &ps.wait, wait).
- if FMODE_WRITE ∧ !list_empty(&ps.async_completed): mask |= EPOLLOUT | EPOLLWRNORM.
- if !connected(ps): mask |= EPOLLHUP.
- if list_empty(&ps.list): mask |= EPOLLERR.

REQ-32: usbdev_mmap(file, vma):
- usb_alloc_coherent(dev, size, GFP_KERNEL, &dma_handle).
- vma.vm_ops = &usbdev_vm_ops (open + close refcount).
- map kernel buffer into VMA via remap_pfn_range / dma_mmap_coherent.
- attach `struct usb_memory` to ps.memory_list.

REQ-33: USBDEVFS_FORBID_SUSPEND / ALLOW_SUSPEND / WAIT_FOR_RESUME:
- FORBID: usb_autoresume_device + suspend_allowed = false.
- ALLOW: suspend_allowed = true; usb_autosuspend_device if no longer in use.
- WAIT_FOR_RESUME: block on ps.wait_for_resume until usbfs_notify_resume wakes it (WRITE_ONCE(not_yet_resumed, 0)).

REQ-34: Compat32 (CONFIG_COMPAT) layer:
- `get_urb32` translates `struct usbdevfs_urb32 → usbdevfs_urb` (pointer fields via `compat_ptr`).
- Parallel `proc_*_compat` entries for CONTROL / BULK / DISCSIGNAL / SUBMITURB / REAPURB / REAPURBNDELAY / IOCTL.
- File-op `.compat_ioctl = compat_ptr_ioctl` for trivial ones; non-trivial ones go through compat dispatch.

REQ-35: Fault-injection / abuse points (documented for review):
- /* Per-USBFS_XFER_MAX cap (~2 GiB) — defense against integer-overflow in length math. */
- /* Per-usbfs_memory_mb cap (default 16 MiB, sysfs writable) — defense against DoS via giant URB queue. */
- /* Per-USB_SG_SIZE = 16 KiB split — defense against giant single allocation. */
- /* Per-`access_ok` check on user buffer pre-submit — defense against user-after-free copy. */
- /* Per-pid+cred captured at submit-time — defense against post-exec UID-change. */
- /* Per-CAN-2005-3055 fix: async URB delivery uses captured creds, never current. */

## Acceptance Criteria

- [ ] AC-1: `usbdev_open` allocates `usb_dev_state`, takes `usb_lock_device`, autoresumes the device, links into `udev.filelist`, captures `disc_pid` + `cred`. Failure path frees ps and puts the device.
- [ ] AC-2: `usbdev_release` releases all claimed interfaces, destroys async lists, drains completed list, releases pid + cred + device, kfrees ps.
- [ ] AC-3: `USBDEVFS_CLAIMINTERFACE` rejects with -EACCES if `privileges_dropped` and the interface is not in `interface_allowed_mask`.
- [ ] AC-4: `USBDEVFS_SUBMITURB` validates flags ⊆ allowed-mask for type, rejects `buffer_length ≥ USBFS_XFER_MAX`, returns -ENOENT on unknown endpoint, -EPERM if interface not claimed.
- [ ] AC-5: `USBDEVFS_SUBMITURB` for `TYPE_CONTROL` parses 8-byte setup, calls `check_ctrlrecip`, computes is_in from `bRequestType & USB_DIR_IN`, buffer pointer advances past setup.
- [ ] AC-6: `USBDEVFS_SUBMITURB` for `TYPE_BULK` against an INT endpoint auto-promotes to `TYPE_INTERRUPT`; against ISO/CTRL endpoints returns -EINVAL.
- [ ] AC-7: `USBDEVFS_SUBMITURB` for `TYPE_ISO` rejects `number_of_packets > 128` and per-packet length > 98304.
- [ ] AC-8: `async_completed` runs in atomic context, captures status, signals via `kill_pid_usb_asyncio` outside the spinlock using the **submit-time** pid+cred (per CAN-2005-3055).
- [ ] AC-9: `USBDEVFS_REAPURB` blocks on `ps.wait` (interruptible, drops device lock around `schedule`), returns -EINTR on signal, -ENODEV on disconnect-with-empty-completed.
- [ ] AC-10: `USBDEVFS_REAPURB` is allowed even after `connected(ps) == false` (per `CAP_REAP_AFTER_DISCONNECT`).
- [ ] AC-11: `USBDEVFS_DISCARDURB` looks up by user-cookie and calls `usb_kill_urb`; the completion then moves the async to `async_completed` with `status == -ENOENT`.
- [ ] AC-12: `USBDEVFS_IOCTL` returns -EACCES if `privileges_dropped`; -EHOSTUNREACH if `state != CONFIGURED`; for `DISCONNECT` evicts kernel driver from interface; for `CONNECT` re-attaches via `device_attach`.
- [ ] AC-13: `USBDEVFS_DISCONNECT_CLAIM` atomically evicts the existing driver (optionally filtered by name via flags) and claims the interface under one `usb_lock_device`.
- [ ] AC-14: `USBDEVFS_DROP_PRIVILEGES` latches `privileges_dropped = true` and the allowed-interface mask; subsequent IOCTL/CLAIM on outside-mask interfaces return -EACCES; cannot be re-tightened (one-way latch within fd lifetime).
- [ ] AC-15: `USBDEVFS_GET_CAPABILITIES` reports `CAP_ZERO_PACKET | CAP_NO_PACKET_SIZE_LIM | CAP_REAP_AFTER_DISCONNECT | CAP_MMAP | CAP_DROP_PRIVILEGES | CAP_CONNINFO_EX` always; `CAP_BULK_CONTINUATION` and `CAP_BULK_SCATTER_GATHER` per bus.
- [ ] AC-16: `usbdev_notify(USB_DEVICE_REMOVE)` wakes all reapers, destroys pending URBs, delivers `discsignr` (if set) with `EPIPE`, unlinks ps from `udev.filelist`.
- [ ] AC-17: `usbfs_increase_memory_usage` enforces `usbfs_memory_mb` cap (system-global), returns -ENOMEM on overrun; `decrease` is symmetric on free_async.
- [ ] AC-18: `usbdev_mmap` allocates coherent DMA, attaches `struct usb_memory` to `ps.memory_list`, refcounted by `vma_use_count` + `urb_use_count`.

## Architecture

```
struct UsbDevState {
  list: ListLink,                  // in udev.filelist
  dev: *mut UsbDevice,             // refcount held
  file: *mut File,
  lock: SpinLock,
  async_pending: ListHead<UsbAsync>,
  async_completed: ListHead<UsbAsync>,
  memory_list: ListHead<UsbMemory>,
  wait: WaitQueueHead,
  wait_for_resume: WaitQueueHead,
  discsignr: u32,
  disc_pid: *mut Pid,
  cred: *const Cred,
  disccontext: SigVal,
  ifclaimed: u64,                  // bitmask of claimed ifnums
  disabled_bulk_eps: u32,          // bitmask of bulk eps disabled by error
  interface_allowed_mask: u64,     // DROP_PRIVILEGES whitelist
  not_yet_resumed: i32,            // WAIT_FOR_RESUME guard
  suspend_allowed: bool,
  privileges_dropped: bool,
}

struct UsbAsync {
  asynclist: ListLink,
  ps: *mut UsbDevState,
  pid: *mut Pid,                   // captured at submit (CAN-2005-3055)
  cred: *const Cred,               // captured at submit
  signr: u32,
  ifnum: i32,
  userbuffer: *mut u8,             // user IN buffer
  userurb: *mut UsbDevfsUrb,       // user cookie
  userurb_sigval: SigVal,
  urb: *mut Urb,
  usbm: Option<*mut UsbMemory>,
  mem_usage: u32,
  status: i32,
  bulk_addr: u8,
  bulk_status: u8,                 // AS_CONTINUATION | …
}

struct UsbMemory {
  memlist: ListLink,
  vma_use_count: i32,
  urb_use_count: i32,
  size: u32,
  mem: *mut u8,
  dma_handle: DmaAddr,
  vm_start: usize,
  ps: *mut UsbDevState,
}

// UAPI structs (verbatim layout)
struct UsbDevfsUrb {
  type_: u8,
  endpoint: u8,
  status: i32,
  flags: u32,
  buffer: *mut u8,
  buffer_length: i32,
  actual_length: i32,
  start_frame: i32,
  number_of_packets_or_stream_id: u32,
  error_count: i32,
  signr: u32,
  usercontext: *mut u8,
  iso_frame_desc: [UsbDevfsIsoPacketDesc; 0],
}
```

`UsbDevio::open(inode, file) -> Result<(), Errno>`:
1. let ps = Box::new_zeroed(UsbDevState::default()).into_raw().
2. let dev = if imajor(inode) == USB_DEVICE_MAJOR { UsbDevio::lookup_by_devt(inode.i_rdev) } else { None }.ok_or(ENODEV)?
3. usb_lock_device(dev).
4. if dev.state == NotAttached: goto unlock_err.
5. UsbPm::autoresume_device(dev).map_err(|_| ENODEV)?
6. ps.dev = dev; ps.file = file; ps.interface_allowed_mask = !0.
7. spin_lock_init(&ps.lock).
8. INIT_LIST_HEAD(&ps.list, &ps.async_pending, &ps.async_completed, &ps.memory_list).
9. init_waitqueue_head(&ps.wait, &ps.wait_for_resume).
10. ps.disc_pid = get_pid(task_pid(current)); ps.cred = get_current_cred(); smp_wmb().
11. list_add_tail(&ps.list, &dev.filelist).
12. file.private_data = ps.
13. usb_unlock_device(dev); Ok(()).

`UsbDevio::submiturb(ps, uurb, iso_desc, arg, sigval) -> Result<(), Errno>`:
1. /* Flag-mask sanity */
2. let mask = SHORT_NOT_OK | BULK_CONTINUATION | NO_FSBR | ZERO_PACKET | NO_INTERRUPT.
3. if uurb.type == ISO: mask |= ISO_ASAP.
4. if uurb.flags & !mask != 0: return Err(EINVAL).
5. if uurb.buffer_length as u32 ≥ USBFS_XFER_MAX: return Err(EINVAL).
6. if uurb.buffer_length > 0 ∧ uurb.buffer.is_null(): return Err(EINVAL).
7. let mut ifnum = -1.
8. if !(uurb.type == CONTROL ∧ (uurb.endpoint & !DIR_MASK) == 0):
   - ifnum = UsbDevio::find_intf_ep(ps.dev, uurb.endpoint)?
   - ps.check_intf(ifnum)?
9. let ep = UsbDevio::ep_to_host_endpoint(ps.dev, uurb.endpoint).ok_or(ENOENT)?
10. let is_in = (uurb.endpoint & USB_DIR_IN) != 0.
11. /* Per-type validation, header parse, ISO frame copy-in */
12. match uurb.type {
    CONTROL => { /* parse 8-byte setup, check_ctrl_recip, recompute buffer */ },
    BULK => { /* validate ep type, INT promotion, SG-chunk count */ },
    INTERRUPT => { /* validate ep type */ },
    ISO => { /* validate 1 ≤ npkts ≤ 128, copy iso_frame_desc, total ≤ 98304 each */ },
    _ => return Err(EINVAL),
}
13. /* User buffer access check */
14. if uurb.buffer_length > 0 ∧ !access_ok(uurb.buffer, uurb.buffer_length): return Err(EFAULT).
15. let mut as_ = UsbAsync::alloc(number_of_packets)?
16. as_.usbm = UsbDevio::find_memory_area(ps, uurb)?
17. /* Memory accounting */
18. UsbDevio::mem_inc(...)?
19. /* SG or direct buffer */
20. build SG | kmalloc transfer_buffer | use usbm.
21. /* Translate UAPI flags → kernel URB_* flags */
22. as_.urb.transfer_flags = is_in_or_out | iso_asap | short_not_ok | zero_packet | no_interrupt | URB_NO_TRANSFER_DMA_MAP-if-usbm.
23. as_.urb.dev = ps.dev; .pipe = encoded; .complete = UsbAsync::completed; .context = as_.
24. /* Per CAN-2005-3055: capture identity at submit */
25. as_.pid = get_pid(task_pid(current)); as_.cred = get_current_cred().
26. as_.ifnum = ifnum; as_.signr = uurb.signr; as_.userurb = arg; as_.userurb_sigval = sigval.
27. UsbAsync::new_pending(&as_).
28. /* Submit */
29. if usb_endpoint_xfer_bulk(&ep.desc):
   - spin_lock_irq(&ps.lock).
   - as_.bulk_addr = encoded.
   - if uurb.flags & BULK_CONTINUATION: as_.bulk_status = AS_CONTINUATION.
   - else: ps.disabled_bulk_eps &= !(1 << bulk_addr).
   - if ps.disabled_bulk_eps & (1 << bulk_addr) != 0: ret = Err(EREMOTEIO).
   - else: ret = usb_submit_urb(as_.urb, GFP_ATOMIC).
   - spin_unlock_irq.
30. else: ret = usb_submit_urb(as_.urb, GFP_KERNEL).
31. if ret.is_err(): snoop + remove_pending + cleanup + return err.
32. Ok(()).

`UsbAsync::completed(urb)` (HCD softirq):
1. let as_ = urb.context as *mut UsbAsync; let ps = as_.ps.
2. spin_lock_irqsave(&ps.lock).
3. list_move_tail(&as_.asynclist, &ps.async_completed).
4. as_.status = urb.status.
5. let (signr, errno, addr, pid, cred) = if as_.signr != 0 { (as_.signr, as_.status, as_.userurb_sigval, get_pid(as_.pid), get_cred(as_.cred)) } else { (0,0,zero,null,null) }.
6. /* Bulk-stream error → cancel continuation chain */
7. if as_.status < 0 ∧ as_.bulk_addr != 0 ∧ as_.status ∉ {ECONNRESET, ENOENT}:
   - UsbDevio::cancel_bulk_urbs(ps, as_.bulk_addr).
8. wake_up(&ps.wait).
9. spin_unlock_irqrestore.
10. if signr != 0:
    - kill_pid_usb_asyncio(signr, errno, addr, pid, cred).
    - put_pid(pid); put_cred(cred).

`UsbAsync::reap(ps) -> Option<*mut UsbAsync>` (blocking):
1. let mut wait = WaitQueueEntry::new(current).
2. add_wait_queue(&ps.wait, &wait).
3. loop:
   - __set_current_state(TASK_INTERRUPTIBLE).
   - let as_ = UsbAsync::get_completed(ps).
   - if as_.is_some() ∨ !ps.connected(): break.
   - if signal_pending(current): break.
   - usb_unlock_device(ps.dev); schedule(); usb_lock_device(ps.dev).
4. remove_wait_queue(&ps.wait, &wait).
5. set_current_state(TASK_RUNNING).
6. as_.

`UsbDevio::do_ioctl(file, cmd, p) -> i64`:
1. let ps = file.private_data as *mut UsbDevState; let dev = ps.dev.
2. if !(file.f_mode & FMODE_WRITE): return -EPERM as i64.
3. usb_lock_device(dev).
4. /* Reap allowed after disconnect */
5. match cmd { REAPURB => reapurb, REAPURBNDELAY => reapurb_nonblock, REAPURB32/REAPURBNDELAY32 (compat) => …, _ => fallthrough }.
6. if !ps.connected(): unlock; return -ENODEV as i64.
7. match cmd { … ~40 cases dispatching to proc_* … }.
8. usb_unlock_device(dev).
9. if ret ≥ 0: update inode atime.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `open_ps_kfree_on_error_path` | INVARIANT | per-usbdev_open: every error path kfrees ps and puts dev. |
| `release_ifclaimed_drained` | INVARIANT | per-usbdev_release: ifclaimed == 0 on exit. |
| `submit_buflen_under_xfer_max` | INVARIANT | per-proc_do_submiturb: buffer_length < USBFS_XFER_MAX always. |
| `submit_iso_pkt_count_bounded` | INVARIANT | per-proc_do_submiturb ISO: 1 ≤ number_of_packets ≤ 128. |
| `submit_iso_pkt_len_bounded` | INVARIANT | per-proc_do_submiturb ISO: each isopkt.length ≤ 98304. |
| `submit_credentials_captured_at_submit` | INVARIANT | per-proc_do_submiturb: as.pid + as.cred = get_pid(current) + get_current_cred() (CAN-2005-3055). |
| `completion_uses_captured_creds` | INVARIANT | per-async_completed: kill_pid_usb_asyncio uses as.pid / as.cred, never current. |
| `completion_signal_outside_spinlock` | INVARIANT | per-async_completed: kill_pid_usb_asyncio called after spin_unlock. |
| `reap_drops_device_lock_around_schedule` | INVARIANT | per-reap_as: usb_unlock_device(dev) before schedule(); usb_lock_device after. |
| `ioctl_privileges_dropped_blocks_driver_ioctl` | INVARIANT | per-proc_ioctl: privileges_dropped ⟹ -EACCES. |
| `mem_inc_caps_at_usbfs_memory_mb` | INVARIANT | per-usbfs_increase_memory_usage: total ≤ usbfs_memory_mb*MB. |
| `mem_inc_no_overflow` | INVARIANT | per-usbfs_increase_memory_usage: addition checked for u64 overflow. |
| `discardurb_kills_existing_urb` | INVARIANT | per-proc_unlinkurb: cookie-found ⟹ usb_kill_urb. |
| `disconnect_claim_atomic_under_device_lock` | INVARIANT | per-proc_disconnect_claim: release-and-claim under one usb_lock_device. |
| `drop_privileges_is_one_way` | INVARIANT | per-proc_drop_privileges: ps.privileges_dropped never cleared. |
| `claim_disallowed_for_unprivileged_intf` | INVARIANT | per-claimintf: privileges_dropped ∧ !allowed_mask[ifnum] ⟹ -EACCES. |

### Layer 2: TLA+

`drivers/usb/core/devio.tla`:
- Per-open + per-submit + per-completion + per-reap + per-cancel + per-disconnect + per-ioctl.
- Properties:
  - `safety_completed_creds_match_submit_creds` — per-async: completion signal uses captured submit-time pid+cred.
  - `safety_one_way_drop_privileges` — per-ps: privileges_dropped is monotone true.
  - `safety_xfer_max_bound` — per-submit: buffer_length < USBFS_XFER_MAX (no integer-overflow in length math).
  - `safety_iso_packet_bounds` — per-ISO-submit: 1 ≤ npkts ≤ 128 ∧ each length ≤ 98304.
  - `safety_memory_cap` — per-mem_inc/dec: usbfs_memory_usage ≤ usbfs_memory_mb*MB at quiescent state.
  - `safety_reap_after_disconnect_works` — per-CAP_REAP_AFTER_DISCONNECT: REAPURB succeeds on async_completed after USB_DEVICE_REMOVE.
  - `safety_claim_under_device_lock` — per-claimintf: usb_driver_claim_interface called under usb_lock_device.
  - `safety_disconnect_claim_atomic` — per-DISCONNECT_CLAIM: release+claim within one device-lock critical section.
  - `safety_ioctl_blocked_when_priv_dropped` — per-proc_ioctl: privileges_dropped ⟹ -EACCES.
  - `liveness_reapurb_eventually_returns` — per-REAPURB: completion ∨ disconnect ∨ signal eventually wakes reaper.
  - `liveness_async_completion_eventually_processed` — per-submitted URB: eventually moves to async_completed (HCD guarantees).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `UsbDevio::open` post: file.private_data = ps ∨ -err | `UsbDevio::open` |
| `UsbDevio::open` post: ps.disc_pid + ps.cred reflect opening task | `UsbDevio::open` |
| `UsbDevio::release` post: ifclaimed = 0; async_pending empty; async_completed drained; ps freed | `UsbDevio::release` |
| `UsbDevio::submiturb` post: success ⟹ async on async_pending; failure ⟹ no leak (mem_dec balanced) | `UsbDevio::submiturb` |
| `UsbDevio::submiturb` post: as.pid + as.cred captured from `current` (not later overwritten) | `UsbDevio::submiturb` |
| `UsbDevio::submiturb` post: URB_DIR_IN/OUT bit consistent with endpoint direction | `UsbDevio::submiturb` |
| `UsbAsync::completed` post: as moved from pending to completed; status = urb.status; signal delivered outside lock | `UsbAsync::completed` |
| `UsbAsync::reap` post: returns Some(as) on completion ∨ None on signal/disconnect | `UsbAsync::reap` |
| `UsbDevio::cancel_bulk_urbs` post: disabled_bulk_eps bit set; all matching pending non-continuation URBs unlinked | `UsbDevio::cancel_bulk_urbs` |
| `UsbDevState::claim_intf` post: privileges_dropped ∧ !allowed[ifnum] ⟹ Err(EACCES) | `UsbDevState::claim_intf` |
| `UsbDevio::drop_privileges` post: ps.privileges_dropped = true; mask = user-supplied | `UsbDevio::drop_privileges` |
| `UsbDevio::disconnect_claim` post: previous driver released ∧ ifnum bit set in ifclaimed (success path) | `UsbDevio::disconnect_claim` |
| `UsbDevio::process_completion` post: actual_length + error_count + status copied; userurb cookie returned | `UsbDevio::process_completion` |
| `UsbDevio::mmap` post: usb_memory linked in ps.memory_list; vma.vm_ops = vm_ops | `UsbDevio::mmap` |
| `UsbDevio::remove` post: every ps in udev.filelist drained, signals delivered, list unlinked | `UsbDevio::remove` |

### Layer 4: Verus/Creusot functional

`Per-open → usb_lock_device → autoresume → ps init → list_add to udev.filelist; per-SUBMITURB validate flags + buffer + endpoint + interface-claim → per-type branch (CONTROL parse, BULK auto-promote/SG/streams, INTERRUPT, ISO packet-array) → memory accounting → URB build with submit-time pid+cred capture → usb_submit_urb (GFP_ATOMIC for bulk under ps.lock for CONTINUATION semantics, else GFP_KERNEL); per-async_completed runs in HCD softirq, moves to completed list, snoops, cancels continuation-chain on error, wakes reapers, delivers signal outside spinlock using captured creds; per-REAPURB blocks on ps.wait with TASK_INTERRUPTIBLE releasing device-lock around schedule, returns -EINTR on signal, -ENODEV on disconnect-empty; per-DISCARDURB cookie-lookup + usb_kill_urb; per-CLAIMINTERFACE / DISCONNECT_CLAIM / DROP_PRIVILEGES enforce per-fd ownership + privilege-drop one-way latch; per-usbdev_notify(USB_DEVICE_REMOVE) wakes all reapers + delivers discsignr=EPIPE + tears down state` semantic equivalence: per-Documentation/driver-api/usb/usb.rst § "/dev/bus/usb" + per-include/uapi/linux/usbdevice_fs.h UAPI contract + libusb-1.0 expected behavior (the de facto reference).

## Hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

usbfs-specific reinforcement:

- **Per-CAN-2005-3055 fix: credentials captured at submit-time** — defense against per-post-submit exec/setuid privilege escalation in async-IO signal delivery.
- **Per-USBFS_XFER_MAX (~2 GiB) hard cap** — defense against per-integer-overflow in length arithmetic on 32-bit and per-DoS via single giant URB.
- **Per-usbfs_memory_mb global cap (default 16 MiB)** — defense against per-DoS via giant async-URB queue exhausting kernel memory; sysfs-writable at module-param.
- **Per-USB_SG_SIZE = 16 KiB chunk split** — defense against per-large-contiguous-allocation pressure on the slab allocator.
- **Per-ISO packet count bounded at 128 + per-packet length bounded at 98304** — defense against per-arithmetic-overflow on ISO totlen.
- **Per-access_ok() pre-submit user-buffer check** — defense against per-bad-pointer fault in HCD context.
- **Per-USBDEVFS_DROP_PRIVILEGES one-way latch + per-interface whitelist** — defense against per-after-claim privilege-escalation in long-running libusb apps (per Documentation/driver-api/usb/usb.rst § "Drop privileges").
- **Per-CLAIMINTERFACE enforces `usb_driver_claim_interface` (LDM authorization-check)** — defense against per-claim of unauthorized interfaces.
- **Per-USBDEVFS_IOCTL gated by privileges_dropped** — defense against per-driver-ioctl-proxy abuse after privilege drop.
- **Per-DISCONNECT_CLAIM atomic under device-lock** — defense against per-disconnect/claim TOCTTOU race.
- **Per-cancel_bulk_urbs propagates errors across BULK_CONTINUATION chain** — defense against per-half-completed bulk transfer.
- **Per-RESET / SETCONFIGURATION require interface-claim ownership** — defense against per-cross-fd device-state perturbation.
- **Per-ISO IN buffer memset-zero** — defense against per-uninitialized-kernel-memory leak via short ISO packets (gap regions).
- **Per-uevent suppression during claim** — defense against per-spurious udev rule-fire during userspace-driver-binding.
- **Per-snoop_urb_data bounded by usbfs_snoop_max (default 64 KiB)** — defense against per-log-flood at debug.
- **Per-USB_DEVICE_REMOVE wakes all reapers + signals submitters** — defense against per-stuck reaper after hot-unplug.
- **Per-CAP_REAP_AFTER_DISCONNECT** — feature, defense against per-data-loss on hot-unplug for in-flight reads.
- **Per-FORBID_SUSPEND increments pm-usage; ALLOW_SUSPEND balances** — defense against per-unbalanced PM refcount across fd lifetime.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `usbdevfs_ctrltransfer`, `usbdevfs_bulktransfer`, `usbdevfs_urb`, and `usbdevfs_iso_packet_desc` arrays copied via bounded `copy_from_user` / `copy_to_user`; URB transfer buffer length validated against the device's max-transfer and against `USBFS_XFER_MAX`.
- **PAX_KERNEXEC** — usbfs `file_operations` and `usb_device.bus->op` tables placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — entropy added on every `usbdev_do_ioctl` / `proc_submiturb` entry to neutralise stack-shape probing under URB-spam races.
- **PAX_REFCOUNT** — `dev_state.openers`, `as_count` (in-flight async URBs), and `usb_device.refcnt` use saturating counters.
- **PAX_MEMORY_SANITIZE** — usbfs URB transfer / setup-packet buffers zero-on-free; defends against keystroke / HID / smartcard data leak across reaped URBs.
- **PAX_UDEREF** — every usbfs ioctl dereferences user pointers only via user-AS-annotated copy helpers; setup-packet `wLength` validated before allocation.
- **PAX_RAP / kCFI** — usbfs URB completion callback, `disconnect`, and async-reap paths type-tagged.
- **GRKERNSEC_HIDESYM** — `/dev/bus/usb/*` and `/sys/bus/usb/devices/*` paths stripped of kernel pointers in dmesg.
- **GRKERNSEC_DMESG** — usbfs URB-fault and disconnect prints gated behind `CAP_SYSLOG`.
- **usbfs USBDEVFS_* ioctl CAP_SYS_RAWIO** — `USBDEVFS_SUBMITURB`, `USBDEVFS_CONTROL`, `USBDEVFS_BULK`, `USBDEVFS_RESET`, `USBDEVFS_CLAIMINTERFACE` all gated behind `CAP_SYS_RAWIO`; defends against unprivileged raw bus access.
- **URB buffer PAX_USERCOPY** — `proc_do_submiturb` copies into a kmalloc'd `urb->transfer_buffer` whose size is bracketed against the endpoint's `wMaxPacketSize` and `USBFS_XFER_MAX`; defends against integer overflow in setup-packet `wLength`.
- **Rationale** — usbfs is the canonical attack surface for unprivileged USB-stack RCE (CVE-2016-3672 family, raw URB-handle LPEs); CAP_SYS_RAWIO gating, URB-buffer USERCOPY bounding, and async-list refcount discipline together close the documented usbfs vulnerability class.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/usb/core/driver.c` (covered in `core-driver.md` Tier-3 — provides `usbfs_driver` claim-target + `usb_driver_claim_interface`/`_release_interface` used here)
- `drivers/usb/core/hub.c` (`usb_hub_claim_port` / `_release_port` / `_release_all_ports` callouts — covered in `core-hub.md` Tier-3)
- `drivers/usb/core/urb.c` (`usb_submit_urb` / `usb_kill_urb` / `usb_unlink_urb` — covered in `core-urb.md` Tier-3)
- `drivers/usb/core/message.c` (sync `usb_set_interface` / `usb_set_configuration` / `usb_clear_halt` / `usb_reset_device` — covered separately)
- `drivers/usb/core/devices.c` (the deprecated `/proc/bus/usb/devices` text format — usbdev_read's cousin)
- `include/uapi/linux/usbdevice_fs.h` numeric ioctl IDs (UAPI const table — covered by uapi-headers crate)
- usbmon packet capture (`drivers/usb/mon/` — separate Tier-3 if expanded)
- libusb / libusb-1.0 userspace library (out of kernel scope)
- Compat32 ABI translation layer details (covered by kernel compat infrastructure)
- Implementation code
