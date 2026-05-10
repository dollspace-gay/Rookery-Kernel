# Tier-3: drivers/nvme/host/pci.c — NVMe PCIe transport (SQ/CQ pair per CPU + MSI-X-per-queue + CMB + HMB + PRP/SGL)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nvme/00-overview.md
upstream-paths:
  - drivers/nvme/host/pci.c
  - drivers/nvme/host/nvme.h
  - include/linux/nvme.h
  - include/uapi/linux/nvme_ioctl.h
-->

## Summary

The NVMe-over-PCIe transport — every consumer NVMe SSD, every server-side NVMe drive, every cloud-attached EBS-style volume that exposes NVMe to the guest. Implements `nvme_ctrl_ops` (cross-ref `host-core.md`) over PCIe BAR0 + MSI-X. Owns: per-controller SQ/CQ allocation (one SQ + one CQ per CPU for IO-queues + 1 admin SQ/CQ pair), HW-queue (`hwq`) blk-mq integration, MSI-X-per-queue interrupt routing, CMB (Controller Memory Buffer) for HW-side SQ/CQ for low-latency, HMB (Host Memory Buffer) hand-off so consumer SSDs without internal DRAM can use host RAM as cache, PRP (Physical Region Page) + SGL (Scatter-Gather List) construction from blk-mq sgs, doorbell-stride decoding, controller register access (CC, CSTS, CAP, AQA, ASQ, ACQ, BPRSEL, BPMBL).

This Tier-3 covers `drivers/nvme/host/pci.c` (~4300 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvme_dev` | per-PCIe-controller control block | `drivers::nvme::pci::Dev` |
| `struct nvme_queue` | per-SQ+CQ pair | `drivers::nvme::pci::Queue` |
| `struct nvme_iod` | per-blk-mq-request NVMe-side state (PRP/SGL) | `drivers::nvme::pci::Iod` |
| `nvme_pci_disable(dev)` / `_enable(dev)` | per-controller enable/disable (CC.EN bit + admin queue setup) | `Dev::pci_enable` / `_disable` |
| `nvme_pci_configure_admin_queue(dev)` | admin queue alloc + AQA/ASQ/ACQ register write | `Dev::configure_admin_queue` |
| `nvme_pci_init_request(set, req, hctx_idx, numa_node)` | per-blk-mq-request init | `Dev::init_request` |
| `nvme_pci_init_hctx(hctx, data, hctx_idx)` | per-hctx init | `Dev::init_hctx` |
| `nvme_pci_init_admin_hctx(...)` | admin-hctx init | `Dev::init_admin_hctx` |
| `nvme_setup_io_queues(dev)` | alloc IO queues per CPU + MSI-X vectors | `Dev::setup_io_queues` |
| `nvme_alloc_queue(dev, qid, depth)` | per-queue alloc | `Queue::alloc` |
| `nvme_free_queues(dev, lowest)` | inverse | `Dev::free_queues` |
| `nvme_create_io_queues(dev)` | issue Create IO Submission Queue + Completion Queue commands | `Dev::create_io_queues` |
| `nvme_init_queue(nvmeq, qid)` | post-alloc init: tail/head ptrs, doorbells | `Queue::init_queue` |
| `nvme_pci_submit_cmd(nvmeq, cmd)` | per-cmd SQ-tail-doorbell write | `Queue::submit_cmd` |
| `nvme_irq(irq, data)` | per-MSI-X-vector IRQ handler | `Queue::irq` |
| `nvme_irq_check(irq, data)` | shared-MSI-X handler | `Queue::irq_check` |
| `nvme_poll(hctx)` / `_iod_check_and_complete(...)` | per-CQ polling | `Queue::poll` / `_iod_check_and_complete` |
| `nvme_pci_setup_prps(dev, req, total_len, iod)` / `_setup_sgls(...)` | PRP / SGL list construction | `Iod::setup_prps` / `_setup_sgls` |
| `nvme_pci_complete_rq(rq)` | per-request completion (called from CQ polling) | `Iod::complete_rq` |
| `nvme_setup_host_mem(dev)` / `_set_host_mem(...)` | HMB allocation + HMB-set admin command | `Dev::setup_host_mem` |
| `nvme_map_cmb(dev)` | Controller Memory Buffer mmap | `Dev::map_cmb` |
| `nvme_pci_alloc_iod_mempool(dev)` / `_free_iod_mempool` | per-controller iod mempool | `Dev::alloc_iod_mempool` |
| `nvme_pci_reset(dev)` / `_reset_work(work)` / `_reset_done(...)` | controller reset | `Dev::reset` / `_reset_work` |
| `nvme_pci_create_request(...)` | per-IO request alloc | `Dev::create_request` |
| `nvme_pci_complete_batch_req(rq, ...)` | batch-complete optimization | `Iod::complete_batch_req` |
| `nvme_dev_disable(dev, shutdown)` | controller disable on hot-remove or PCI error | `Dev::disable` |

## Compatibility contract

REQ-1: PCI device-id table includes every in-tree NVMe controller (Intel, Samsung, Kioxia, Western Digital, Micron, Crucial, Sabrent, Sk-Hynix, Phison, generic NVMe-spec-compliant); per-vendor quirks (admin-queue depth, MSI-X count limit, no-deepest-power-state, single-vector-only, etc.) preserved.

REQ-2: Per-controller register access via BAR0: `nvme_readq` / `_writeq` for CAP / CC / CSTS / AQA / ASQ / ACQ / BPRSEL / BPMBL / SUBSYS-RESET / NSSR / CRTO / VS.

REQ-3: Per-vendor doorbell-stride decoding from CAP register; per-queue doorbell at `BAR0 + 0x1000 + (qid * 2 * stride)` (SQ) and `+stride` (CQ).

REQ-4: Per-CPU IO-queue allocation: one SQ+CQ pair per online CPU (up to per-controller max-queues); per-queue MSI-X vector pinned to the matching CPU via `irq_set_affinity_hint`.

REQ-5: Admin queue: 32-deep SQ + 32-deep CQ; admin commands routed here (Identify, Create-IO-Queue, Set/Get-Features, Format-NVM, etc.).

REQ-6: PRP construction: per-bio-frag dma-mapped + chained via PRP-list pages when > 2 pages; per-page boundary alignment; CRT-PMRP for command-list-rooted PRP.

REQ-7: SGL construction (when CTRATT.SGLS != 0): per-bio-sg dma-mapped + per-cmd SGL descriptor (Data Block / Bit Bucket / Segment / Last-Segment).

REQ-8: CMB (Controller Memory Buffer): when CMBSZ register non-zero, `nvme_map_cmb` mmap'd as host-visible BAR-region; per-queue SQ optionally placed in CMB for low-latency submit (no PCIe-write-then-read latency).

REQ-9: HMB (Host Memory Buffer): when controller advertises HMBPRE in identify, `nvme_setup_host_mem` allocates host-side buffer + Set-Features(HMB) admin command + maps via PRP-list.

REQ-10: Per-MSI-X-vector IRQ handler `nvme_irq` polls associated CQ; bounded by `dev->q_depth` per-poll budget; over-budget reschedules via blk-mq complete-via-softirq.

REQ-11: Per-queue blk-mq integration: each CPU's queue is one `blk_mq_hw_ctx`; submission via `queue_rq` callback dispatches `Queue::submit_cmd`; completion via per-CQ IRQ.

REQ-12: Per-controller reset (`nvme_reset_work`): disable + AER-protected re-init + namespace rescan; in-flight requests requeued via `kick_requeue_lists` (cross-ref `host-core.md`).

REQ-13: Hot-plug support: surprise-removal detected via PCI AER + per-completion zero-result detection; controller disabled + namespaces removed; on re-plug, full re-probe.

REQ-14: SR-IOV VF: each VF probes as separate `nvme_dev`; admin commands restricted per-spec (no namespace-mgmt from VF).

## Acceptance Criteria

- [ ] AC-1: Reference NVMe drive enumerates → `/dev/nvme0` chardev + `/dev/nvme0n1` blockdev appear → `nvme list` matches upstream.
- [ ] AC-2: fio benchmark on `/dev/nvme0n1`: 4K random read at QD=128 achieves IOPS within 2% of upstream.
- [ ] AC-3: HMB test on consumer DRAM-less SSD: `nvme id-ctrl` shows HMB allocation; sustained writes don't degrade vs DRAM-backed.
- [ ] AC-4: CMB test on supported drive: `nvme cmb-status` shows CMB sizing + usage; latency-sensitive workload uses CMB-backed SQ.
- [ ] AC-5: Reset stress: `nvme reset /dev/nvme0` 100x; in-flight IOs survive (re-queued); KASAN clean.
- [ ] AC-6: Hot-remove test: `echo 1 > /sys/bus/pci/devices/<bdf>/remove` triggers clean controller disable + namespace removal.
- [ ] AC-7: SR-IOV test: `echo 4 > /sys/bus/pci/devices/<bdf>/sriov_numvfs` on capable controller → 4 VFs each with own queue/IO.
- [ ] AC-8: blktests nvme/* PCIe-relevant subset passes.

## Architecture

`Dev` lives in `drivers::nvme::pci::Dev`:

```
struct Dev {
  refcount: Refcount,
  ctrl: Arc<NvmeCtrl>,            // generic core (cross-ref host-core.md)
  pdev: Arc<PciDev>,
  bar: NonNull<u8>,                // BAR0 mmap
  bar_mapped_size: usize,
  cmb: Option<Arc<Cmb>>,
  cmbsz: u64,
  cmbloc: u64,
  hmb: Option<Arc<Hmb>>,
  hmb_descs: Option<Arc<NvmeHmbDescriptors>>,
  page_size: u32,                  // controller's MPSMIN
  doorbell_stride: u32,
  q_depth: u32,                    // max queue depth from CAP.MQES
  online_queues: AtomicI32,
  max_qid: u32,
  io_queues: ArrayMutex<MAX_QUEUES, Option<Arc<Queue>>>,
  admin_q: Arc<Queue>,
  num_vecs: u32,
  reset_work: WorkStruct,
  remove_work: WorkStruct,
  ctrl_config: u32,                // CC register cache
  shutdown_works: AtomicU32,
  attrs: KBox<DevAttrs>,
  iod_mempool: KBox<MemPool>,
  shutdown_lock: Mutex<()>,
}

struct Queue {
  refcount: Refcount,
  dev: Arc<Dev>,
  qid: u16,
  cq_phase: u8,                    // current expected CQ phase bit
  q_depth: u32,
  sq_dma_addr: dma_addr_t,
  cq_dma_addr: dma_addr_t,
  sq_cmds: NonNull<NvmeCommand>,    // SQ entries (mmap'd)
  cqes: NonNull<NvmeCompletion>,    // CQ entries (mmap'd)
  q_db: NonNull<u32>,                // per-queue doorbell mmio
  cq_head: AtomicU32,
  sq_tail: AtomicU32,
  cmd_size: u32,
  poll_cq: AtomicBool,
  cq_vector: i32,                   // MSI-X vector
  napi: Option<KBox<Napi>>,
  task: AtomicPtr<TaskStruct>,
  cqe_seen: AtomicBool,
}

struct Iod {
  ctrl: Arc<NvmeCtrl>,
  npages: u32,
  use_sgl: bool,
  nr_descriptors: u8,
  total_len: u32,
  sg: KBox<ScatterList>,
  meta_sg: Option<KBox<ScatterList>>,
  meta_dma: dma_addr_t,
  first_dma: dma_addr_t,
  list: KBox<NvmeIodList>,
}
```

Probe path:
1. `pci_driver::probe(pdev, id)` matches via per-vendor table.
2. Enable device (`pci_enable_device`); request BAR0 region; memremap BAR0.
3. `Dev::new(pdev, id)`: alloc Dev struct; populate `bar`, `doorbell_stride`, `q_depth` from CAP register.
4. `Dev::pci_enable`:
   - Disable controller (CC.EN=0); wait CSTS.RDY=0.
   - Allocate admin SQ/CQ pair (32 entries each).
   - Configure AQA/ASQ/ACQ registers.
   - Enable controller (CC.EN=1); wait CSTS.RDY=1.
5. `Ctrl::init(&dev.ctrl, dev, &pci_ops, quirks)` via core (cross-ref `host-core.md`).
6. `Dev::setup_io_queues`:
   - Issue Identify Controller → `nr_io_queues` from spec response.
   - Allocate MSI-X vectors via `pci_alloc_irq_vectors_affinity(pdev, 1, num_vecs, PCI_IRQ_MSIX | PCI_IRQ_AFFINITY, &affd)`.
   - For each `qid` in 1..nr_io_queues:
     - `Queue::alloc(&dev, qid, q_depth)`: alloc SQ + CQ DMA-coherent pages, init head/tail.
     - Issue Create-IO-Submission-Queue + Create-IO-Completion-Queue admin commands.
     - `request_irq(msix_vec, nvme_irq, IRQF_SHARED, "nvme", queue)`.
   - `irq_set_affinity_hint(msix_vec, cpumask_of(cpu))` for per-CPU queue pinning.
7. `Ctrl::start(&dev.ctrl)` via core → triggers namespace scan + chardev registration.

Per-IO submission `Queue::queue_rq(hctx, bd)`:
1. Get NVMe command from blk-mq request via `Iod::setup`.
2. `Iod::setup_prps` or `_setup_sgls` based on `cmnd.flags`:
   - For each bio_vec: `dma_map_page(...)` → array of dma_addrs.
   - If <= 2 pages: PRP1 = first dma_addr, PRP2 = second (if any).
   - If > 2 pages: PRP2 = pointer to PRP-list page chain.
   - SGL variant: per-segment SGL descriptor; chained for multi-segment.
3. `Queue::submit_cmd(&queue, &cmd)`:
   - Memcpy cmd into `queue.sq_cmds[sq_tail % q_depth]`.
   - Inc sq_tail.
   - mb() → write `*queue.q_db = sq_tail` (doorbell).

Per-IRQ + per-CQ `Queue::irq`:
1. Read CQ entries starting at `cq_head`; check phase bit.
2. While `cqe.phase == queue.cq_phase`:
   - Look up blk-mq req via `cqe.cid` (16-bit per-queue tag).
   - `Iod::complete_rq(req, cqe.status, cqe.result)` → blk_mq_end_request.
   - Inc cq_head; on wrap, toggle `queue.cq_phase`.
3. Update `*queue.cq_db = cq_head` (CQ doorbell write).
4. Return IRQ_HANDLED.

Per-controller reset `Dev::reset_work`:
1. `Ctrl::change_state(RESETTING)`.
2. `Ctrl::quiesce_io_queues` (cross-ref `host-core.md`) → blk-mq stop dispatch.
3. `Dev::pci_disable` (CC.EN=0; wait CSTS.RDY=0).
4. NSSR + secondary-bus-reset if controller-stuck (escalating reset).
5. `Dev::pci_enable` (re-init).
6. Identify Controller + re-create IO queues.
7. `Ctrl::change_state(LIVE)`.
8. `Ctrl::unquiesce_io_queues` + `kick_requeue_lists` to retry pending IOs.

CMB (Controller Memory Buffer):
- At init, read CMBSZ + CMBLOC; if non-zero, BAR + offset is host-visible HW memory.
- Optionally place per-queue SQ in CMB via `cmb-sqes=on` cmdline; reduces PCIe-write-then-read latency for cmd submit.

HMB (Host Memory Buffer):
- At init, read identify-controller HMBPRE; if non-zero, controller wants to use host memory as cache.
- `Dev::setup_host_mem`: alloc per-descriptor host-buffer chunks + per-descriptor PRP-list page; submit Set-Features(HMB) admin command pointing at descriptors.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `queue_no_uaf` | UAF | `Arc<Queue>` outlives all in-flight blk-mq requests on this queue; reset-during-IO safe. |
| `cqe_phase_invariant` | INVARIANT | per-CQ `cq_phase` toggles exactly once per CQ wrap; consumer never accepts stale CQE. |
| `prp_no_oob` | OOB | PRP-list construction bounded by per-cmd nr_pages; PRP-list page chain length validated. |
| `doorbell_no_oob` | OOB | per-queue doorbell address `BAR0 + 0x1000 + (qid * 2 * stride)` validated against bar_mapped_size. |
| `q_depth_no_overflow` | OVERFLOW | per-queue head/tail wrapped at q_depth via modulo; never overflow. |

### Layer 2: TLA+

`models/nvme/sq_cq.tla` (parent-declared): proves per-queue submission/completion ordering — host writes SQ-tail doorbell, device fetches SQE, posts CQE, host reads CQ-head; concurrent multi-CPU submissions on per-CPU queue serialize via per-queue lock + interrupt coalescing safe.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Queue::submit_cmd` post: cmd written at `sq_cmds[old_sq_tail % q_depth]`; sq_tail incremented; doorbell written | `Queue::submit_cmd` |
| `Queue::irq` invariant: every iteration decrements `cqe_count` (eventually terminates); phase-bit flip is observable when CQ wraps | `Queue::irq` |
| `Iod::setup_prps` post: PRP-list covers exactly `total_len` bytes; PRP1 + PRP2 (or PRP-list-page chain) addresses match dma_mapped page sequence | `Iod::setup_prps` |
| Per-controller `online_queues` matches count of allocated + Create-IO-Queue-cmd-acked queues | `Dev::setup_io_queues` |

### Layer 4: Verus/Creusot functional

`Queue::queue_rq → submit_cmd → HW processes SQE → posts CQE → Queue::irq → blk_mq_end_request` round-trip equivalence: completion's `cid` matches submission's `cid`; status + result fields correctly translated.

## Hardening

(Inherits row-1 features from `drivers/nvme/00-overview.md` § Hardening.)

host-pci specific reinforcement:

- **Per-queue MSI-X vector pinned per-CPU** — defense against IRQ-vec-flood on single CPU under heavy load.
- **CMB mmap region size validated against controller's CMBSZ** — defense against malicious controller advertising oversized CMB.
- **HMB descriptor count cap** — bounded by spec-max + per-vendor quirk; defense against controller requesting excessive HMB.
- **PRP-list page chain length cap** — bounded by per-cmd max-data-transfer-size; defense against malformed cmd causing long PRP-list walk.
- **CC register write fenced** — every CC.EN bit toggle followed by mmio-readback + spin-on-CSTS.RDY; defense against PCIe-posted-write reordering.
- **Doorbell write fenced** — every SQ-tail or CQ-head doorbell write preceded by `wmb()` to ensure SQ entry visible to device before doorbell rings.
- **Reset rate-limit** — per-controller reset rate-limited at 1/sec (cross-ref `host-core.md`).
- **Surprise-removal detected via per-completion zero-result invariant** — defense against AER-not-yet-fired-but-device-physically-gone scenario.
- **NSSR (NVM Subsystem Reset) requires CAP_SYS_RAWIO** — affects all controllers in subsystem (cross-ref `host-core.md`).
- **Per-vendor quirks-table audited** — every entry has rationale comment + spec or erratum reference.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TCP transport (covered in `host-tcp.md` future Tier-3)
- RDMA transport (covered in `host-rdma.md` future Tier-3)
- FC transport (covered in `host-fc.md` future Tier-3)
- Multipath logic (covered in `host-multipath.md` future Tier-3)
- Generic core (covered in `host-core.md` parent Tier-3)
- ZNS (covered in `host-zns.md` future Tier-3)
- Auth (covered in `host-auth.md` future Tier-3)
- 32-bit-only paths
- Implementation code
