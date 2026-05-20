# Tier-3: drivers/xen/{balloon,privcmd}.c — memory balloon + /dev/xen/privcmd hypercall passthrough

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/xen/00-overview.md
upstream-paths:
  - drivers/xen/balloon.c
  - drivers/xen/xen-balloon.c
  - drivers/xen/mem-reservation.c
  - drivers/xen/privcmd.c
  - drivers/xen/privcmd-buf.c
  - drivers/xen/privcmd.h
  - include/xen/balloon.h
  - include/uapi/xen/privcmd.h
-->

## Summary

This Tier-3 pairs two related pieces of the Xen guest support: the **memory balloon driver** that lets the hypervisor dynamically grow or shrink a guest's physical-page allotment (returning pages to the hypervisor when an admin shrinks a guest, asking the hypervisor for more when target is increased, hot-plugging actual memory blocks when the OS allows it), and **`/dev/xen/privcmd`**, the Dom0/stubdom passthrough character device that lets a privileged userspace tool (`xl`, `xenstored`, libxenctrl-using daemons) issue arbitrary hypercalls, do "foreign" memory mappings, and bridge irqfds/ioeventfds between QEMU and HVM guests.

The balloon driver tracks two scheduling states (`BP_DONE`/`BP_WAIT`/`BP_ECANCELED`/`BP_EAGAIN`), a `balloon_stats` block, an LRU of ballooned pages, and a `balloon_thread` kthread; privcmd offers a small but extremely powerful ioctl surface (`IOCTL_PRIVCMD_HYPERCALL`, `_MMAP`, `_MMAPBATCH_V2`, `_MMAP_RESOURCE`, `_DM_OP`, `_RESTRICT`, `_IRQFD`, `_IOEVENTFD`, `_PCIDEV_GET_GSI`).

This Tier-3 covers `balloon.c` (~825 lines: target/credit math, increase/decrease via `XENMEM_populate_physmap` + `_decrease_reservation`, hotplug-memory integration, `xen_online_page`, shrinker, sysctl) + `privcmd.c` (~1785 lines: ioctl dispatch, foreign-mapping VMA ops, irqfd/ioeventfd machinery, dm_op buffer pipeline).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct balloon_stats` | balloon: current/target/high/low/hotplug counters | `drivers::xen::balloon::Stats` |
| `enum bp_state` | scheduler return: DONE/WAIT/EAGAIN/ECANCELED | `Balloon::State` |
| `balloon_thread()` | kthread main loop — reserve/release until target reached | `Balloon::Thread` |
| `reserve_additional_memory()` | hot-add unpopulated memory range via `add_memory_resource`/`online_pages_movable_noop` | `Balloon::reserve_additional` |
| `increase_reservation(nr_pages)` | call `xenmem_reservation_increase` on `frame_list[]` | `Balloon::increase` |
| `decrease_reservation(nr_pages, gfp)` | unmap + return pages to hypervisor via `xenmem_reservation_decrease` | `Balloon::decrease` |
| `balloon_append(page)` / `balloon_retrieve(require_lowmem)` | per-page enqueue / dequeue on `ballooned_pages` LRU | `Balloon::{append,retrieve}` |
| `xen_online_page(page, order)` | per-page online callback (sets `PageOffline` so memory hotplug doesn't real-online) | `Balloon::on_page_online` |
| `update_schedule()` | exponential backoff between balloon work cycles | `Balloon::update_schedule` |
| `balloon_set_new_target(nr_pages)` | sysfs/xenstore write hook — bumps `target_pages` + wakes thread | `Balloon::set_target` |
| `xenmem_reservation_va_mapping_update(...)` / `_reset(...)` | per-page VA-to-MFN swap during inflate/deflate | `MemReservation::*` |
| `IOCTL_PRIVCMD_HYPERCALL` (`privcmd_ioctl_hypercall`) | raw hypercall passthrough (op + 5 args) | `Privcmd::hypercall` |
| `IOCTL_PRIVCMD_MMAP` (`privcmd_ioctl_mmap`) | per-call list of (mfn, va, npages) foreign-mapping | `Privcmd::mmap` |
| `IOCTL_PRIVCMD_MMAPBATCH_V2` (`privcmd_ioctl_mmap_batch`) | batched foreign-mapping with per-page error array | `Privcmd::mmap_batch` |
| `IOCTL_PRIVCMD_MMAP_RESOURCE` (`privcmd_ioctl_mmap_resource`) | map per-domain "resource" (grant table, IOREQ pages) | `Privcmd::mmap_resource` |
| `IOCTL_PRIVCMD_DM_OP` (`privcmd_ioctl_dm_op`) | dispatch device-model hypercall on behalf of QEMU (HVM) | `Privcmd::dm_op` |
| `IOCTL_PRIVCMD_RESTRICT` (`privcmd_ioctl_restrict`) | one-way restrict this fd to a single target domain | `Privcmd::restrict` |
| `IOCTL_PRIVCMD_IRQFD` (`privcmd_ioctl_irqfd`) | assign/deassign per-eventfd → guest-IRQ injection | `Privcmd::irqfd` |
| `IOCTL_PRIVCMD_IOEVENTFD` | per-eventfd → guest MMIO/PIO trap (KVM-ioeventfd analogue) | `Privcmd::ioeventfd` |
| `IOCTL_PRIVCMD_PCIDEV_GET_GSI` | resolve a PCI BDF → host GSI for passthrough | `Privcmd::pcidev_get_gsi` |
| `privcmd_vm_ops` | per-VMA ops for foreign-mapping VMAs (fault, unmap, …) | `Privcmd::VmOps` |
| `privcmd_buf_fops` (in privcmd-buf.c) | `/dev/xen/privcmd-buf` — allocate-bounce-buffer fd for dm_op | `PrivcmdBuf::FileOps` |
| `MMUEXT_*` hypercall ops | per-mmuext flush/pin/unpin/install (used via `IOCTL_PRIVCMD_HYPERCALL`) | (caller-controlled — gated by allowlist) |

## Compatibility contract

REQ-1: `balloon_stats` exposed via `/sys/devices/system/xen_memory/`:
- `target_kb` (R/W) — desired guest memory size; writing wakes balloon_thread.
- `current_kb` (R) — actual.
- `low_kb` / `high_kb` (R) — split by lowmem/highmem.
- `schedule_delay` (R), `max_schedule_delay` (R/W), `retry_count` (R), `max_retry_count` (R/W).

REQ-2: Inflate (decrease reservation): pop nr_pages pages from buddy allocator, `xenmem_reservation_decrease(nr_pages, frame_list)` hypercall, mark pages `PageOffline`, append to `ballooned_pages` LRU.

REQ-3: Deflate (increase reservation): retrieve pages from `ballooned_pages` (require_lowmem if needed), `xenmem_reservation_increase(nr_pages, frame_list)` to ask hypervisor for backing frames, map VA→new MFN, return page to buddy.

REQ-4: Memory hotplug (`CONFIG_XEN_BALLOON_MEMORY_HOTPLUG`): when balloon target exceeds existing populated memory, `reserve_additional_memory` invokes `add_memory_resource` to create a fresh `memory_block` carrying `PageOffline` pages that the balloon can then deflate.

REQ-5: Shrinker integration: balloon registers `balloon_shrinker` (count = `balloon_stats.balloon_low + .balloon_high`, scan = `balloon_inflate_by(nr_to_scan)` to claw back memory).

REQ-6: Boot-time hotplug-unpopulated knob `xen_hotplug_unpopulated` (sysctl `xen.hotplug_unpopulated`) — when set, balloon may pre-hot-add pages so that gnttab/foreign-map paths have headroom.

REQ-7: `/dev/xen/privcmd` is restricted to a single fd-owner per open via `IOCTL_PRIVCMD_RESTRICT` — irreversible per-fd assignment to a target `domid_t`; subsequent hypercalls + mmaps are filtered.

REQ-8: `IOCTL_PRIVCMD_HYPERCALL` rejects calls outside `privcmd_call_op_allowed[]` allowlist when running in DomU (only Dom0 may freely issue hypercalls). When `unrestricted=1` module-param set (debug-only) the allowlist is bypassed.

REQ-9: Foreign mappings (`IOCTL_PRIVCMD_MMAP_BATCH_V2`) take a per-page MFN array + VMA range, install `privcmd_vm_ops`, return per-page error codes (PAGED_ERROR / MFN_ERROR / generic) in user array; VMA cannot be split.

REQ-10: dm_op buffer pipeline — userspace pre-allocates buffers via `/dev/xen/privcmd-buf` (limited to `privcmd_dm_op_buf_max_size = 4096` bytes each, max `privcmd_dm_op_max_num = 16` per call); kernel pins + passes pointers to `HYPERVISOR_dm_op`.

REQ-11: irqfd assignment — `privcmd_irqfd_assign(irqfd)` installs an eventfd waiter that, on signal, invokes `HYPERVISOR_event_channel_op(EVTCHNOP_send)` to inject into the target guest; lifecycle via `irqfd_cleanup_wq` + per-irqfd shutdown work.

REQ-12: `IOCTL_PRIVCMD_PCIDEV_GET_GSI` — wraps `HYPERVISOR_physdev_op(PHYSDEVOP_pci_get_gsi)`; only valid in Dom0 with PCI passthrough support.

## Acceptance Criteria

- [ ] AC-1: `echo $((mem - 64*1024)) > /sys/devices/system/xen_memory/target_kb` shrinks guest by 64 MiB; `current_kb` converges within `max_schedule_delay`.
- [ ] AC-2: Re-inflate to original target succeeds without OOM.
- [ ] AC-3: `xl mem-set <domain> +64` from Dom0 increases the guest's `target_kb` via xenstore watch.
- [ ] AC-4: Hotplug-unpopulated test: `echo 1 > /proc/sys/xen/hotplug_unpopulated` + `xl mem-max <domain> Nm` adds a fresh memory block visible in `/sys/devices/system/memory/`.
- [ ] AC-5: Shrinker fires under memory pressure (e.g. tmpfs fill); `balloon_low` decreases.
- [ ] AC-6: `xl create domain.cfg && xenstore-ls /local/domain/<id>` works via `/dev/xen/xenbus` + `/dev/xen/privcmd`.
- [ ] AC-7: `IOCTL_PRIVCMD_RESTRICT` on a freshly-opened privcmd fd permanently locks it to one domid; subsequent hypercalls referencing a different domid fail `-EPERM`.
- [ ] AC-8: `IOCTL_PRIVCMD_MMAP_BATCH_V2` of 1024 pages from a target HVM guest into a Dom0 helper succeeds; per-page error array zero.
- [ ] AC-9: `IOCTL_PRIVCMD_IRQFD` from a QEMU helper drives a guest-side IRQ on eventfd signal.

## Architecture

`Balloon` lives in `drivers::xen::balloon::Balloon`:

```
struct Balloon {
  mutex: Mutex,
  thread_wq: WaitQueueHead,
  inflate_wq: WaitQueueHead,
  stats: Stats,                       // balloon_stats
  ballooned: List<Arc<Page>>,         // LRU of offline pages
  frame_list: [u64; PAGE_SIZE/8],     // scratch MFN array
  shrinker: Arc<Shrinker>,
  hotplug_unpopulated: bool,
  boot_timeout: u32,
  state: AtomicEnum<BpState>,
}

struct Stats {
  current_pages: AtomicU64,
  target_pages: AtomicU64,
  balloon_low: AtomicU64,
  balloon_high: AtomicU64,
  target_unpopulated: AtomicU64,
  total_pages: AtomicU64,
  schedule_delay: AtomicU32,
  max_schedule_delay: AtomicU32,
  retry_count: AtomicU32,
  max_retry_count: AtomicU32,
}
```

balloon_thread loop:
1. `wait_event(thread_wq, current_pages != target_pages)`.
2. `credit = target_pages - current_pages`.
3. If `credit > 0`: `reserve_additional_memory()` (if hot-add needed) then `increase_reservation(min(credit, batch))`.
4. If `credit < 0`: `decrease_reservation(-credit, GFP_HIGHUSER)`.
5. `update_schedule()` — on `BP_EAGAIN` double `schedule_delay`; on success reset to 1.
6. `msleep_interruptible(schedule_delay * 1000)`.

Increase reservation `increase_reservation(nr_pages)`:
1. For each page slot, `retrieve = balloon_retrieve(require_lowmem)`; if absent → `BP_EAGAIN`.
2. Build `frame_list[i] = page_to_xen_pfn(page)`.
3. `xenmem_reservation_increase(nr_pages, frame_list)` hypercall.
4. Per-page `xenmem_reservation_va_mapping_update` to map VA→new MFN.
5. `__free_reserved_page(page)` to return to buddy.

Decrease reservation `decrease_reservation(nr_pages, gfp)`:
1. `alloc_pages(gfp, 0)` ×nr_pages → `frame_list`.
2. `scrub_one_page(page)` if `xen_scrub_pages`.
3. `xenmem_reservation_va_mapping_reset(page)` to unmap VA.
4. `xenmem_reservation_decrease(nr_pages, frame_list)` hypercall.
5. Per-page `balloon_append(page)` onto `ballooned_pages` LRU + `__SetPageOffline`.

privcmd ioctl dispatch:

```
struct PrivcmdFile {
  domid: AtomicDomid,         // INVALID until RESTRICT
  restricted: AtomicBool,
  irqfds: Mutex<List<KernelIrqfd>>,
  ioeventfds: Mutex<List<KernelIoeventfd>>,
}
```

`IOCTL_PRIVCMD_HYPERCALL`:
1. `copy_from_user(&hypercall)`.
2. If `restricted == true` and `hypercall.op` not in `privcmd_call_op_allowed` → `-EPERM`.
3. `privcmd_call(op, arg0..arg4)` — direct hypercall trampoline.

`IOCTL_PRIVCMD_MMAP_BATCH_V2`:
1. Copy `privcmd_mmapbatch_v2` from user.
2. Find/extend caller's VMA; install `privcmd_vm_ops`.
3. `mmap_batch_fn` iterates MFN array; per page `HYPERVISOR_mmu_update(MMU_NORMAL_PT_UPDATE, va, mfn)`.
4. Per page write back PRIVCMD_MMAPBATCH_*_ERROR if hypercall returned per-page error.

`IOCTL_PRIVCMD_DM_OP`:
1. Validate `count <= privcmd_dm_op_max_num` and each buf size `<= privcmd_dm_op_buf_max_size`.
2. For each buf: `lock_pages` (pin user pages), build kernel-virtual `struct xen_dm_op_buf[]`.
3. `HYPERVISOR_dm_op(domid, count, &buf_array)`.
4. `unlock_pages`.

`IOCTL_PRIVCMD_IRQFD` assign:
1. `eventfd_ctx_fdget(irqfd.fd)`.
2. Alloc `KernelIrqfd`, attach poll-wait to eventfd's wqh.
3. On signal: `irqfd_inject` → `HYPERVISOR_event_channel_op(EVTCHNOP_send)` to target domain's port.

## Hardening

(Inherits row-1 features from `drivers/xen/00-overview.md` § Hardening.)

balloon+privcmd-specific reinforcement:

- **`xen_scrub_pages` boot/runtime knob** — zero balloon pages before returning to hypervisor; defense against cross-domain data leak.
- **`PageOffline` set on every ballooned page** — defense against hotplug-online of foreign-owned page.
- **`balloon_mutex` serializes inflate/deflate** — defense against credit miscomputation race.
- **`max_retry_count` + exponential `schedule_delay`** — defense against tight-loop CPU burn when hypervisor cannot satisfy.
- **`privcmd_call_op_allowed` allowlist** — restricted fds may issue only a small set of hypercalls (XEN_DOMCTL, MMUEXT_INVLPG, …); defense against arbitrary-hypercall escape.
- **`privcmd_dm_op_*` size caps** — defense against unbounded buffer-pinning by a malicious QEMU.
- **`irqfd_cleanup_wq` defers free of `KernelIrqfd`** — defense against UAF on eventfd shutdown race.
- **`IOCTL_PRIVCMD_RESTRICT` is irreversible** — once a fd is bound to a domid, no path unbinds; defense against domid-confusion mid-session.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `balloon_stats`, `privcmd_data`, `privcmd_buf_vma_private`, `privcmd_kernel_irqfd`, `privcmd_kernel_ioeventfd`, and `mmap_batch_state`; all `copy_from_user` of ioctl headers bounded by `sizeof(struct)` with `check_object_size`.
- **PAX_KERNEXEC** — `privcmd_fops`, `privcmd_vm_ops`, `privcmd_buf_fops`, balloon sysctl table, and balloon shrinker ops placed in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `privcmd_ioctl_*`, `balloon_thread`, `increase_reservation`, and `decrease_reservation` entries.
- **PAX_REFCOUNT** — saturating refcount on `PrivcmdFile`, `KernelIrqfd`, `KernelIoeventfd`, and balloon page-references; overflow trap defeats irqfd-shutdown race UAF and balloon page-leak.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `KernelIrqfd`, `KernelIoeventfd`, dm_op pinned buffers, `frame_list` scratch arrays, and balloon pages on `decrease_reservation` (via `xen_scrub_pages`) so MFNs + secrets cannot bleed to the hypervisor or to another domain.
- **PAX_UDEREF** — SMAP/PAN enforced on every privcmd ioctl, on `privcmd_buf` mmap fault path, and on balloon sysfs writes; reject mis-sized payloads and unaligned VA arguments.
- **PAX_RAP / kCFI** — `privcmd_fops`, `privcmd_vm_ops`, `privcmd_buf_fops`, the balloon shrinker vtable, and the per-ioctl dispatch table marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `PrivcmdFile`/`KernelIrqfd` pointers in `/proc/<pid>/fdinfo` behind CAP_SYSLOG; suppress `%p` in balloon trace events.
- **GRKERNSEC_DMESG** — restrict balloon retry/credit warnings and privcmd `not permitted` audit lines to CAP_SYSLOG so attackers cannot probe hypervisor topology via dmesg.
- **`/dev/xen/privcmd` CAP_SYS_RAWIO + lockdown** — open() requires `CAP_SYS_RAWIO` and is refused when `kernel_locked_down(LOCKDOWN_XEN_PRIVCMD)`; integrity-mode lockdown disables `IOCTL_PRIVCMD_HYPERCALL` outright.
- **MMUEXT op allowlist** — when `IOCTL_PRIVCMD_HYPERCALL` carries a `__HYPERVISOR_mmuext_op`, the `mmuext_op::cmd` is validated against an allowlist (`MMUEXT_INVLPG_LOCAL/_MULTI/_ALL`, `MMUEXT_TLB_FLUSH_*`, `MMUEXT_FLUSH_CACHE*`); pinning / unpinning of arbitrary L1-L4 PTs is refused for restricted fds.
- **Hypercall argument validation** — `privcmd_ioctl_hypercall` arg pointers validated against `domid` capture (`RESTRICT`-gated); arg-arrays bounded by sane upper-bound per op.
- **Balloon page `MEMORY_SANITIZE`** — `decrease_reservation` always scrubs (`xen_scrub_pages=1` forced); defense against information disclosure to the hypervisor via the freed page contents.
- **irqfd/ioeventfd lifecycle hardened** — `irqfd_cleanup_wq` drains under `irqfds_lock` write-side; `KernelIrqfd` is `kfree_sensitive` so the captured `eventfd_ctx` pointer cannot be re-used after free.

Rationale: `/dev/xen/privcmd` is the most powerful character device in a Xen Dom0 — it can issue any hypercall, map any guest page, and inject IRQs into any guest. The balloon driver, in turn, can hand pages back to the hypervisor; without scrubbing, those pages can carry secrets across domains. RAP/kCFI on the ioctl dispatch table, CAP_SYS_RAWIO + lockdown on the device node, MMUEXT and hypercall-op allowlisting, forced page-scrub on every decrease_reservation, saturating refcount on irqfd/ioeventfd lifetimes, and SMAP/PAN on every user payload convert these two drivers from "kernel-level hypervisor passthrough" into a structurally hardened administrative boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- xenbus / event channels / grant tables (covered in `xenbus.md`)
- gntdev / gntalloc user-grant interfaces (covered in `gntdev.md` future Tier-3)
- xen-blkback / netback per-driver Tier-3
- 32-bit guests — Rookery targets 64-bit guests only
- Implementation code
