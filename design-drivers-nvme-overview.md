---
title: "Tier-2: drivers/nvme — NVMe host stack + NVMe-oF target (core + pci + tcp + rdma + fc + multipath + zns + auth + nvmet)"
tags: ["tier-2", "drivers", "nvme", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the NVMe stack — the dominant block-storage protocol on modern Linux (every consumer NVMe SSD, every enterprise NVMe-oF JBOF, every cloud-attached EBS-style volume, every container-overlay-fast-path on bare-metal). Owns:

- **NVMe host common**: protocol parsing + admin command set + IO command set + identify-controller / identify-namespace + reservations + power-state mgmt + sanitize + format + crypto-erase + hwmon temperature + character-device passthrough (`/dev/nvme<n>`) + SED-Opal integration
- **NVMe host transports**: PCIe (`pci.c` — submission/completion queue with MSI-X-per-queue + per-CPU queue pinning), TCP (`tcp.c` — kernel-side initiator over TCP), RDMA (`rdma.c` — IB + RoCE + iWARP), FC (`fc.c` — Fibre Channel via fc_class), Apple ANS (`apple.c` — Apple Silicon proprietary; kept compile-gated)
- **NVMe Fabrics**: Discovery + Connect + Authenticate + Property Get/Set abstractions used by tcp/rdma/fc transports
- **Multipath**: NVMe native multipath (ANA — Asymmetric Namespace Access) with per-namespace path-state policies (numa / round-robin / queue-depth)
- **ZNS**: Zoned Namespace command set + zone-state machine + zone-append + zone-finish/reset/open/close
- **Auth**: NVMe in-band authentication (DH-HMAC-CHAP)
- **Persistent Reservations**: NVMe reservation commands (acquire / register / release / report) bridged to block-layer pr_ops (cross-ref `block/00-overview.md`)
- **NVMe target (nvmet)**: in-kernel target — exposes block devs / files / passthru-NVMe-controllers as NVMe-oF subsystems via configfs; transports tcp / rdma / fc / loop (kernel-side loopback) / pci-epf (PCIe Endpoint Function — present an NVMe controller out of an SoC PCIe-EP). Target-side authentication + discovery + admin/io passthrough

Heavily cross-referenced from `block/00-overview.md` (block-mq + bio + request-queue is the NVMe submission/completion glue), `drivers/pci/00-overview.md` (NVMe PCIe transport binds via pci_driver), `drivers/iommu/00-overview.md` (DMA addresses translated by IOMMU), `kernel/dma/00-overview.md` (dma_map_sg for prp/sgl construction), `drivers/iommu/iommu-sva.md` (PASID-bound NVMe sva for SQHD/CQHD), `kernel/irq/00-overview.md` (msi_domain alloc one IRQ per queue), `net/00-overview.md` (TCP transport via kernel TCP socket), future `drivers/infiniband/00-overview.md` (RDMA transport via verbs), future `drivers/scsi/00-overview.md` (FC transport via fc_class shared with SCSI), `kernel/cgroup/blkio.md` (blkcg accounting on NVMe blockdevs).

### Out of Scope

- Per-Tier-3 (Phase D)
- Apple ANS proprietary transport (`host/apple.c`) — out of v0
- pci-epf endpoint-mode target (`target/pci-epf.c`) — out of v0 (ARM-SoC use case; no x86_64 SoC ships as PCIe-EP)
- 32-bit-only paths
- Implementation code

### components

### Host

- **core** (`host/core.c` + `host/nvme.h`): controller + namespace lifecycle, identify-controller / identify-namespace processing, queue setup/teardown abstraction, reset state machine, AEN (Asynchronous Event Notification) processing, persistent feature update, sanitize/format dispatch, namespace-rescan on AEN
- **constants** (`host/constants.c`): per-opcode + per-status-code text strings (used by trace + sysfs error reporting)
- **fabrics** (`host/fabrics.c` + `host/fabrics.h`): shared Connect / Discovery / Property-Get / Property-Set / KATO (keep-alive) / Authenticate logic for tcp + rdma + fc transports
- **pci** (`host/pci.c`): PCIe transport — per-controller submission/completion queue arrays, MSI-X-per-queue, per-CPU queue mapping (one IO-queue per CPU), CMB (Controller Memory Buffer) support, SGL/PRP construction, doorbell-stride handling, host-memory-buffer (HMB)
- **tcp** (`host/tcp.c`): TCP transport — kernel TCP socket per-queue, NVMe/TCP PDU framing (CapsuleCmd / CapsuleResp / H2CData / C2HData / HdrDigest / DataDigest / R2T), io_uring-cmd-style fast path
- **rdma** (`host/rdma.c`): RDMA transport — verbs QP per-queue, mem-reg pool, send/recv/rdma_read/rdma_write
- **fc** (`host/fc.c` + `host/fc.h`): FC transport — uses Fibre Channel transport class (cross-ref `drivers/scsi/00-overview.md`)
- **apple** (`host/apple.c`): Apple ANS proprietary transport (Apple Silicon Macs); compile-gated off for Rookery v0
- **multipath** (`host/multipath.c`): native multipath — per-controller ANA group state, per-namespace path-state policy (numa/round-robin/queue-depth), failover on path-down AEN
- **ioctl** (`host/ioctl.c`): `/dev/nvme<n>` chardev IOCTLs — `NVME_IOCTL_ID`, `_ADMIN_CMD`, `_IO_CMD`, `_RESET`, `_RESCAN`, `_SUBSYS_RESET`, `_ADMIN64_CMD`, `_IO64_CMD`
- **sysfs** (`host/sysfs.c`): per-controller + per-namespace + per-subsystem sysfs tree
- **zns** (`host/zns.c`): Zoned Namespace cmd set + `report_zones` blkdev-op + zone-state mgmt
- **pr** (`host/pr.c`): Persistent Reservations bridge to `pr_ops`
- **auth** (`host/auth.c`): in-band auth (DH-HMAC-CHAP)
- **hwmon** (`host/hwmon.c`): controller temperature reporting via hwmon
- **fault_inject** (`host/fault_inject.c`): debugfs-driven fault injection (CONFIG_NVME_CORE_FAULT_INJECTION)
- **trace** (`host/trace.c` + `host/trace.h`): tracepoints `nvme:*`

### Target (nvmet)

- **core** (`target/core.c` + `target/nvmet.h`): subsystem + namespace + controller lifecycle, command dispatch
- **admin-cmd** (`target/admin-cmd.c`): admin command set processing
- **io-cmd-bdev** (`target/io-cmd-bdev.c`): IO commands backed by struct block_device
- **io-cmd-file** (`target/io-cmd-file.c`): IO commands backed by struct file (sparse-file backing)
- **passthru** (`target/passthru.c`): IO commands proxied to a host-side NVMe controller
- **fabrics-cmd** (`target/fabrics-cmd.c`) + **fabrics-cmd-auth** (`target/fabrics-cmd-auth.c`): Fabrics Connect / Discovery / Property / Authenticate command processing
- **discovery** (`target/discovery.c`): Discovery subsystem (well-known NQN `nqn.2014-08.org.nvmexpress.discovery`)
- **configfs** (`target/configfs.c`): `/sys/kernel/config/nvmet/{subsystems,ports,hosts,referrals}/...` configuration tree
- **debugfs** (`target/debugfs.c` + `target/debugfs.h`)
- **tcp** (`target/tcp.c`): NVMe/TCP target transport
- **rdma** (`target/rdma.c`): NVMe/RDMA target transport
- **fc** (`target/fc.c`) + **fcloop** (`target/fcloop.c`): NVMe/FC target + virtual-loop test transport
- **loop** (`target/loop.c`): kernel-side loopback transport (host + target in the same kernel — useful for testing + for boot-from-NVMe-on-virtio scenarios)
- **pci-epf** (`target/pci-epf.c`): present a target as a PCIe Endpoint Function (out of v0 — ARM-SoC use case)
- **zns** (`target/zns.c`): ZNS target support
- **pr** (`target/pr.c`): Persistent Reservations target side
- **auth** (`target/auth.c`): in-band auth target side
- **trace** (`target/trace.c` + `target/trace.h`)

### Common

- **common** (`common/`): shared protocol headers/helpers used by host + target (e.g., key-management, KATO timer)

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/nvme/` (~50 source files across `host/`, `target/`, `common/` subdirs), public headers `include/linux/nvme*.h`, UAPI `include/uapi/linux/nvme_ioctl.h` + `include/uapi/linux/nvme_keyring.h`.

### compatibility contract — outline

### `/dev/nvme<N>` chardev IOCTLs

Wire format byte-identical:
- `NVME_IOCTL_ID` — get namespace ID
- `NVME_IOCTL_ADMIN_CMD` — admin passthrough (struct nvme_passthru_cmd)
- `NVME_IOCTL_IO_CMD` — io passthrough
- `NVME_IOCTL_ADMIN64_CMD` / `_IO64_CMD` — 64-bit-result variants
- `NVME_IOCTL_RESET` — controller reset
- `NVME_IOCTL_SUBSYS_RESET` — NVM subsystem reset
- `NVME_IOCTL_RESCAN` — namespace rescan
- `NVME_URING_CMD_*` — io_uring-cmd-style passthrough (uring submission)

### `/dev/nvme<N>n<M>` block device

Standard Linux block-device interface plus NVMe-specific block-IOCTLs (passthrough subset). Same minor-numbering scheme + sysfs path as upstream.

### sysfs surface

Per-controller `/sys/class/nvme/nvme<N>/`:
- `model`, `serial`, `firmware_rev`, `state`, `transport`, `address` (transport-specific URI)
- `cntlid`, `subsysnqn`, `hostnqn`, `hostid`, `cntrltype`, `dctype`
- `nuse`, `numa_node`, `queue_count`, `sqsize`
- `reset_controller` (writable: trigger reset)
- `rescan_controller` (writable: rescan namespaces)
- `delete_controller` (writable: tear down)
- `kato`, `ctrl_loss_tmo`, `reconnect_delay`
- `dhchap_secret`, `dhchap_ctrl_secret`, `dhchap_hash`, `dhchap_dhgroup`
- Per-namespace `nvme<N>n<M>/`: `nsid`, `nguid`, `eui`, `uuid`, `wwid`, `nvme_subsystem` symlink
- ANA: `ana_grpid`, `ana_state`, `iopolicy`, `numa_nodes`

Per-subsystem `/sys/class/nvme-subsystem/nvme-subsys<N>/`:
- `subsysnqn`, `model`, `serial`, `firmware_rev`, `subsystype`
- Per-controller path symlinks `nvme<N>`

Layout + content byte-identical so `nvme-cli` + `multipath-tools` consume unchanged.

### NVMe-oF Fabrics connect ABI

`/dev/nvme-fabrics` chardev — userspace nvme-cli writes a comma-separated key=value config string ("transport=tcp,traddr=10.0.0.1,trsvcid=4420,nqn=...") and the kernel performs Discovery + Connect; resulting controller appears at `/dev/nvme<N>`. Wire format byte-identical so nvme-cli connect-all works unchanged.

### Configfs target tree

`/sys/kernel/config/nvmet/`:
- `subsystems/<NQN>/{attr_allow_any_host, attr_serial, attr_model, attr_firmware, attr_version, attr_pi_support, attr_qid_max, attr_ieee_oui, attr_cntlid_min, attr_cntlid_max, attr_type, namespaces/<NSID>/{device_path, device_uuid, device_nguid, ana_grpid, attr_buffered_io, enable, ...}, allowed_hosts/<NQN>, passthru/{device_path, enable}, referrals/, ana_groups/}`
- `ports/<N>/{addr_traddr, addr_trsvcid, addr_trtype, addr_adrfam, addr_treq, addr_tsas, ana_groups/, subsystems/<NQN> symlink, referrals/}`
- `hosts/<NQN>/{dhchap_key, dhchap_ctrl_key, dhchap_hash, dhchap_dhgroup}`

Layout byte-identical so `nvmetcli` works unchanged.

### Tracepoints

`events/nvme/`:
- `nvme_setup_cmd`, `nvme_complete_rq`, `nvme_async_event`, `nvme_sq` (dump SQE)

### NVMe-PCI device-id table modaliases

`MODULE_DEVICE_TABLE(pci, ...)` for nvme — `pci:v...d...sv*sd*bcc01sc08i02` (Mass Storage / NVM Express). Format byte-identical.

### Per-controller reset state machine

NVME_CTRL_NEW → CONNECTING → LIVE → RESETTING → CONNECTING → LIVE (or DELETING / DEAD on failure). State transitions visible at `/sys/class/nvme/nvme<N>/state` byte-identical strings.

### NVMe AEN handling

Async Event Notifications dispatched: NVME_AEN_TYPE_NOTICE_NS_CHANGED → namespace rescan; NVME_AEN_TYPE_NOTICE_FW_ACT_STARTING → log; NVME_AEN_TYPE_NOTICE_ANA_CHANGE → multipath path-state update. Behavior identical.

### NVMe SED-Opal integration

`/dev/nvme<N>` block-device backed by SED-Opal locking ranges; sed-opal IOCTLs from `block/sed-opal.c` work transparently (cross-ref `block/00-overview.md`).

### Multipath dm-multipath cooperation

When NVMe native multipath is enabled (default-on for ANA-supporting controllers), NVMe presents one block device per subsystem (not per-controller); `multipathd` userspace can inspect via sysfs but should NOT layer dm-multipath on top. Cmdline `nvme.multipath=N` disables to fall back to dm-multipath. Behavior identical.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/nvme/host-core.md` | `host/core.c` + `host/nvme.h`: controller/namespace lifecycle, identify, reset state machine, AEN |
| `drivers/nvme/host-fabrics.md` | `host/fabrics.c`: shared connect/disc/auth/keep-alive |
| `drivers/nvme/host-pci.md` | `host/pci.c`: PCIe transport — SQ/CQ pairs + MSI-X-per-queue + CMB + HMB |
| `drivers/nvme/host-tcp.md` | `host/tcp.c`: NVMe/TCP transport |
| `drivers/nvme/host-rdma.md` | `host/rdma.c`: NVMe/RDMA transport |
| `drivers/nvme/host-fc.md` | `host/fc.c`: NVMe/FC transport |
| `drivers/nvme/host-multipath.md` | `host/multipath.c`: native multipath + ANA |
| `drivers/nvme/host-ioctl.md` | `host/ioctl.c`: chardev IOCTLs + uring-cmd passthrough |
| `drivers/nvme/host-sysfs.md` | `host/sysfs.c`: per-ctrl/ns/subsys sysfs surface |
| `drivers/nvme/host-zns.md` | `host/zns.c`: Zoned Namespace cmd set |
| `drivers/nvme/host-pr.md` | `host/pr.c`: Persistent Reservations |
| `drivers/nvme/host-auth.md` | `host/auth.c`: DH-HMAC-CHAP in-band auth |
| `drivers/nvme/host-hwmon.md` | `host/hwmon.c`: controller temperature |
| `drivers/nvme/host-fault-inject.md` | `host/fault_inject.c`: debugfs fault inject |
| `drivers/nvme/target-core.md` | `target/core.c` + `target/nvmet.h`: subsystem/namespace/controller lifecycle |
| `drivers/nvme/target-admin-cmd.md` | `target/admin-cmd.c`: admin cmd processing |
| `drivers/nvme/target-io-bdev.md` | `target/io-cmd-bdev.c`: bdev-backed IO |
| `drivers/nvme/target-io-file.md` | `target/io-cmd-file.c`: file-backed IO |
| `drivers/nvme/target-passthru.md` | `target/passthru.c`: pass IO to a host-side NVMe |
| `drivers/nvme/target-fabrics-cmd.md` | `target/fabrics-cmd.c` + `fabrics-cmd-auth.c` |
| `drivers/nvme/target-discovery.md` | `target/discovery.c`: Discovery subsystem |
| `drivers/nvme/target-configfs.md` | `target/configfs.c`: configfs tree |
| `drivers/nvme/target-tcp.md` | `target/tcp.c`: TCP target transport |
| `drivers/nvme/target-rdma.md` | `target/rdma.c`: RDMA target transport |
| `drivers/nvme/target-fc.md` | `target/fc.c` + `target/fcloop.c`: FC target + virtual-loop test |
| `drivers/nvme/target-loop.md` | `target/loop.c`: kernel-side loopback transport |
| `drivers/nvme/target-zns.md` | `target/zns.c`: ZNS target |
| `drivers/nvme/target-pr.md` | `target/pr.c`: PR target side |
| `drivers/nvme/target-auth.md` | `target/auth.c`: in-band auth target side |

### compatibility outline (top-level)

- REQ-O1: All `/dev/nvme<N>` IOCTLs + UAPI struct (`nvme_passthru_cmd*`, `nvme_uring_cmd`) byte-identical (nvme-cli + io_uring NVMe-passthru consumers run unmodified).
- REQ-O2: `/dev/nvme-fabrics` connect-string format + Discovery flow byte-identical.
- REQ-O3: Per-controller + per-namespace + per-subsystem sysfs surface byte-identical (nvme-cli + multipath-tools + sosreport consume unchanged).
- REQ-O4: nvmet configfs tree byte-identical (nvmetcli consumes unchanged).
- REQ-O5: AEN dispatching (NS_CHANGED rescan, ANA_CHANGE multipath update) byte-identical.
- REQ-O6: NVMe-PCIe + NVMe-TCP + NVMe-RDMA + NVMe-FC transports interoperate with reference targets (Linux nvmet, SPDK target, vendor arrays).
- REQ-O7: Multipath ANA path-state policies (numa / round-robin / queue-depth) selectable via `iopolicy` sysfs byte-identical.
- REQ-O8: ZNS commands (zone-append, zone-mgmt, zone-report) bridged to `report_zones` blkdev-op identically (zonefs + f2fs + btrfs zoned mode work unchanged).
- REQ-O9: Persistent Reservation bridge to block-pr-ops identical.
- REQ-O10: SED-Opal locking range integration unchanged (sed-opal IOCTLs work).
- REQ-O11: io_uring-cmd NVMe passthrough fast path byte-identical (`liburing` + fio io_uring engine NVMe-passthru work unchanged).
- REQ-O12: nvmet target via tcp / rdma / fc / loop transports interoperates with reference initiators (Linux nvme-host, Windows NVMe/TCP, SPDK initiator).
- REQ-O13: TLA+ models declared at this Tier-2 (per-queue submission/completion ordering, controller-reset state machine, ANA path-state convergence, target controller-disconnect cleanup, fabrics auth handshake).
- REQ-O14: Verus/Creusot Layer-4 functional contracts on PRP/SGL construction (no buffer overflow), per-namespace LBA arithmetic, persistent-reservation key handling.
- REQ-O15: Hardening: row-1 features applied per `00-security-principles.md`, with NVMe-specific reinforcement (DH-HMAC-CHAP secrets shielded in keyring, SED-Opal mediation, configfs target writes RBAC-mediated).

### acceptance criteria (top-level)

- [ ] AC-O1: `nvme list`, `nvme id-ctrl /dev/nvme0`, `nvme id-ns /dev/nvme0n1` produce output byte-identical to upstream baseline. (covers REQ-O1, REQ-O3)
- [ ] AC-O2: `nvme connect-all -t tcp -a <ip> -s 4420` connects to a remote nvmet TCP target; `/dev/nvme<N>` appears + IO succeeds. (covers REQ-O2, REQ-O6)
- [ ] AC-O3: fio io_uring engine with `cmd_type=nvme` against `/dev/ng0n1` achieves IOPS within 2% of upstream baseline. (covers REQ-O1, REQ-O11)
- [ ] AC-O4: ANA failover test: target reports new ANA state via AEN; `iopolicy=numa` host re-routes IO to a healthy path within 10 reconnect_delay periods. (covers REQ-O5, REQ-O7)
- [ ] AC-O5: ZNS test: `nvme zone-mgmt-send /dev/nvme0n1 -a open -z 0`; subsequent `blkzone report` shows zone state OPEN. (covers REQ-O8)
- [ ] AC-O6: Persistent reservation test: `nvme resv-acquire` from initiator A succeeds; second from initiator B returns reservation conflict. (covers REQ-O9)
- [ ] AC-O7: SED-Opal lock range setup via `sedutil-cli` works on NVMe; data inaccessible after lock. (covers REQ-O10)
- [ ] AC-O8: nvmetcli `restore` of a saved configfs config restores the target subsystem + ports + namespaces correctly. (covers REQ-O4)
- [ ] AC-O9: Loop-back test: nvmetcli configures a target on the same kernel; `nvme connect -t loop -n <NQN>` connects + IO works. (covers REQ-O12)
- [ ] AC-O10: DH-HMAC-CHAP auth: target requires CHAP; host without correct secret rejected; with correct secret accepted. (covers REQ-O15)
- [ ] AC-O11: kselftest `tools/testing/selftests/nvme/` passes. (covers REQ-O13, REQ-O14)
- [ ] AC-O12: blktests `nvme/*` test suite passes. (covers REQ-O13, REQ-O14)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/nvme/sq_cq.tla` | `drivers/nvme/host-pci.md` (proves: per-queue submission/completion ordering — host writes SQ-tail doorbell, device fetches SQE, posts CQE, host reads CQ-head; concurrent multi-CPU submissions on per-CPU queue serialize via per-queue lock + interrupt coalescing safe) |
| `models/nvme/ctrl_reset.tla` | `drivers/nvme/host-core.md` (proves: controller-reset state machine NEW → CONNECTING → LIVE → RESETTING → CONNECTING → LIVE; concurrent in-flight commands during reset terminated cleanly; no command both completed-on-old-controller and re-issued-on-new) |
| `models/nvme/ana_failover.tla` | `drivers/nvme/host-multipath.md` (proves: ANA-state-change AEN → host path-state update → in-flight retry on alternate path; concurrent IO submission during failover always lands on a path the controller advertises as optimized/non-optimized at submit time) |
| `models/nvme/target_disconnect.tla` | `drivers/nvme/target-core.md` (proves: target controller-disconnect cleanup — initiator drops session; in-flight commands canceled, namespaces detached, controller object freed in correct order with no UAF on namespace bdev_handle) |
| `models/nvme/auth_handshake.tla` | `drivers/nvme/host-auth.md` + `drivers/nvme/target-auth.md` (proves: DH-HMAC-CHAP handshake — Negotiate / Challenge / Reply / Success-1 / Success-2; replay attack rejected; downgrade attack rejected) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/nvme/host-pci.md` | PRP/SGL construction post-condition: every PRP entry / SGL descriptor maps a valid IOMMU-mapped DMA region; total bytes covered equals `nvme_rq->bio` total length |
| `drivers/nvme/host-multipath.md` | per-namespace path-list invariant: at least one optimized path or one non-optimized path is selectable when at least one ana_state ∈ {OPT, NONOPT}; head-failover preserves request ordering for fua/fence cmds |
| `drivers/nvme/target-core.md` | `nvmet_req_init` pre/postcondition: command opcode + length + nsid validated against namespace bounds before bdev/file backend dispatch; LBA arithmetic checked-mul never overflows |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-controller + per-namespace + per-subsystem + per-queue + per-target-controller + per-port refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-transport `nvme_ctrl_ops`, per-target `nvmet_fabrics_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | LBA arithmetic + PRP/SGL byte counts + per-namespace blocks-count multiplications use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-transport ctrl_ops + per-cmd-set opcode-dispatch tables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed namespace + controller state cleared (carries SED-Opal keys + DH-CHAP secrets) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | NVMe has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for NVMe

LSM hooks called: file-LSM hooks on `/dev/nvme*` open + ioctl + read/write, `/sys/kernel/config/nvmet/...` configfs writes, `/sys/class/nvme*/...` sysfs writes (reset/rescan/delete/dhchap_*).

GR-RBAC adds:
- Per-role disallow `/dev/nvme*` admin-cmd ioctl (`NVME_IOCTL_ADMIN_CMD`) — denies arbitrary admin command issuance (defense against fw-update / sanitize / format-namespace from untrusted process).
- Per-role disallow `/sys/kernel/config/nvmet/` writes — only kernel admin can configure target.
- Per-role audit of every controller reset / namespace rescan / dhchap_secret write.
- Per-role disallow `nvme_uring_cmd` passthrough fast path — denies bypass-block-layer access.

### NVMe-specific reinforcement

- **DH-HMAC-CHAP secrets keyring-shielded**: secrets written to `/sys/.../dhchap_secret` stored in kernel keyring (cross-ref `security/keys/`) with per-secret possessor + ACL; not readable from userspace after install.
- **Admin command opcode allowlist for unprivileged uring-cmd**: even with CAP_SYS_ADMIN, the io_uring NVMe passthrough has an allowlist of safe admin opcodes (Identify, GetLogPage, GetFeatures); fw-image-download / sanitize / format-NVM / set-features-LBA-format / namespace-mgmt require explicit `NVME_IOCTL_ADMIN_CMD` path with explicit cap check.
- **SED-Opal mediation**: SED-Opal IOCTLs gated by CAP_SYS_RAWIO + per-LSM hook; secrets passed in IOCTLs zeroed on copy-out.
- **Target-side configfs write quotas**: per-port maximum subsystem count + per-subsystem maximum namespace count capped (default 256 + 1024) to prevent configfs-fuzz DoS.
- **Fabrics-Connect rate-limit**: per-source-IP NVMe-Connect rate-limited at target (default 10/sec) to prevent connect-flood DoS.
- **Discovery subsystem default-deny**: discovery requires explicit allowed_hosts list unless `attr_allow_any_host=1` is explicitly set per-subsystem (Rookery default-N for new subsystems; upstream default matches but Rookery enforces via configfs-write LSM).
- **Multipath head-failover ordering**: re-issued commands on alternate path get the SAME submission tag (defense against double-completion racing on stale path).

