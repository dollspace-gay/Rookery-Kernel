# Tier-2: drivers/scsi — SCSI middle layer (mid-layer + ULDs + transport classes + libsas + libfc + libiscsi + LLDs + target)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/scsi/
  - include/scsi/
  - include/uapi/scsi/
-->

## Summary

Tier-2 overview for the SCSI subsystem — the legacy-but-still-pervasive storage middle layer behind every SAS / FC / iSCSI / SRP / FCoE storage array, every USB-storage / FireWire / VirtIO-SCSI device, every Ultrium tape drive, every CD/DVD/BD-ROM, every parallel-SCSI legacy peripheral, every SD/MMC card behind the ums driver. Architectural sandwich:

- **Upper Level Drivers (ULDs)**: `sd` (disk), `sr` (CD/DVD-ROM), `st` (tape), `ch` (media changer), `sg` (generic SCSI passthrough chardev) — register as `scsi_driver`s and bind to detected SCSI devices to expose `/dev/sd*`, `/dev/sr*`, `/dev/st*`, `/dev/sch*`, `/dev/sg*` block/char devices
- **Mid-Layer**: command queueing (blk-mq backed), error-recovery state machine (`scsi_error.c`), device scan (`scsi_scan.c`) with INQUIRY-based device-blacklist (`scsi_devinfo.c`), persistent-device-handler hooks (`scsi_dh.c`) for multipath path-state queries, sysfs surface (`scsi_sysfs.c`), procfs surface (`scsi_proc.c`), bsg passthrough (`scsi_bsg.c`), netlink event channel (`scsi_netlink.c`), tracepoints (`scsi_trace.c`), per-host PM (`scsi_pm.c`), command-status logging (`scsi_logging.c`)
- **Transport classes**: `scsi_transport_sas`, `scsi_transport_fc`, `scsi_transport_iscsi`, `scsi_transport_spi` (parallel SCSI), `scsi_transport_srp` (SRP/IB) — provide per-transport sysfs + per-transport device discovery glue shared across all LLDs of that transport
- **Lib transports**: `libsas/` (SAS expander/phy/port management — used by every SAS LLD), `libfc/` (FC fabric login + ELS + Exchange management — used by every FC LLD), `libiscsi/` (iSCSI PDU framing + login/logout — used by every iSCSI initiator), `libfcoe/` (FCoE Ethernet encapsulation)
- **Low-Level Drivers (LLDs)**: ~150 vendor LLDs (mpt3sas, megaraid_sas, lpfc, qla2xxx, qla4xxx, hpsa, smartpqi, aacraid, mptfusion, snic, fnic, bnx2fc, bnx2i, csiostor, cxlflash, scsi_debug — to enumerate them all is to enumerate the storage industry; only a small subset binned for active maintenance in v0)
- **target**: `drivers/target/` — LIO target (in-kernel SCSI target supporting iSCSI / iSER / FC / FCoE / SRP / TCM_USER backports / TCM_QLA2XXX / vhost-scsi). Configfs at `/sys/kernel/config/target/` — separate Tier-2 wrapper future, but cross-referenced here

Heavily cross-referenced from `block/00-overview.md` (SCSI ULDs use blk-mq; bsg uses block layer), `drivers/usb/00-overview.md` (USB-storage gadget binds via SCSI), `drivers/pci/00-overview.md` (most LLDs are PCI-based), `drivers/nvme/00-overview.md` (NVMe/FC reuses fc transport class + libfc helpers), `kernel/cgroup/blkio.md` (blkcg accounting on SCSI blockdevs), future `drivers/target/00-overview.md` (LIO target).

## Components

### Mid-layer

- **scsi** (`scsi.c` + `scsi_priv.h` + `hosts.c`): subsystem init + host registration
- **scsi_lib** (`scsi_lib.c` + `scsi_lib_dma.c`): blk-mq submission/completion glue for SCSI commands; SG list construction
- **scsi_lib_test** (`scsi_lib_test.c`): KUnit tests
- **scsi_proto_test** (`scsi_proto_test.c`): KUnit tests for SCSI protocol parsers
- **scsi_scan** (`scsi_scan.c`): bus scan + INQUIRY + device probing
- **scsi_error** (`scsi_error.c`): error-recovery state machine — abort, dev-reset, target-reset, bus-reset, host-reset escalation
- **scsi_devinfo** (`scsi_devinfo.c`): per-device-id blacklist + quirk flags
- **scsi_dh** (`scsi_dh.c`): device-handler registration for multipath
- **scsi_ioctl** (`scsi_ioctl.c`): legacy SCSI IOCTLs (SG_IO, SCSI_IOCTL_*, SCSI_INQUIRY)
- **scsi_sysfs** (`scsi_sysfs.c`): per-host + per-target + per-device sysfs surface
- **scsi_proc** (`scsi_proc.c`): `/proc/scsi/{scsi, <ll>}/N` legacy procfs
- **scsi_bsg** (`scsi_bsg.c`): block-SG (bsg) passthrough — `/dev/bsg/<host>:<target>:<lun>:<channel>`
- **scsi_netlink** (`scsi_netlink.c`): NETLINK_SCSITRANSPORT for transport events
- **scsi_pm** (`scsi_pm.c`): per-host PM (suspend / resume / runtime)
- **scsi_logging** (`scsi_logging.c`): per-cmd error logging with sense-data decode
- **scsi_trace** (`scsi_trace.c`): tracepoints `scsi:*`
- **scsi_debugfs** (`scsi_debugfs.c`): per-host debugfs
- **scsi_common** (`scsi_common.c`): protocol helpers shared across mid-layer + LLDs (sense decode, status decode)
- **scsi_sysctl** (`scsi_sysctl.c`): sysctls for scan-async-default + log-level
- **constants** (`constants.c`): per-opcode + per-status text strings (used by scsi_logging + sysfs)

### ULDs

- **sd** (`sd.c` + `sd_dif.c` + `sd_zbc.c`): SCSI disk + Data Integrity Field (T10 PI) + Zoned-block-device cmd set
- **sr** (`sr.c` + `sr_ioctl.c` + `sr_vendor.c`): SCSI CD/DVD/BD ROM
- **st** (`st.c`): SCSI tape (ST + STP modes)
- **ch** (`ch.c`): SCSI media changer (autoloader)
- **sg** (`sg.c`): generic SCSI chardev passthrough (`/dev/sg<N>`); also exposes SG_IO via block-bsg

### Transport classes

- **scsi_transport_sas** (`scsi_transport_sas.c`): SAS transport class — phy/port/expander sysfs (`/sys/class/sas_*`)
- **scsi_transport_fc** (`scsi_transport_fc.c`): FC transport class — port/rport sysfs (`/sys/class/fc_*`); BSG/FC for fabric mgmt
- **scsi_transport_iscsi** (`scsi_transport_iscsi.c`): iSCSI transport — session/connection sysfs (`/sys/class/iscsi_*`); netlink to iscsid
- **scsi_transport_spi** (`scsi_transport_spi.c`): parallel SCSI (legacy)
- **scsi_transport_srp** (`scsi_transport_srp.c`): SRP/IB transport

### Lib transports

- **libsas** (`libsas/`): SAS expander discovery + phy mgmt + port mgmt + SAS task framework + SATA-SAS bridge (sas_ata.c)
- **libfc** (`libfc/`): FC fabric login + Exchange management + ELS + virtual ports
- **libiscsi** (`libiscsi.c` + `libiscsi_tcp.c`): iSCSI PDU framing + login + R2T handling + iSCSI/TCP socket-glue
- **libfcoe** (`libfcoe/`): FCoE Ethernet encapsulation + FIP

### LLD selection (v0 maintenance set)

The complete LLD list is ~150 drivers; Rookery v0 keeps in-tree only the actively-maintained ones for x86_64 server/desktop:

- **PCI HBAs**: `mpt3sas/`, `mpi3mr/`, `megaraid_sas/`, `aacraid/`, `hpsa.c`, `smartpqi/`, `mptfusion/`, `aic7xxx/`, `aic94xx/`, `arcmsr/`, `bfa/`, `csiostor/`, `cxlflash/`, `esas2r/`, `fnic/`, `hisi_sas/`, `lpfc/`, `mvsas/`, `pm8001/`, `pmcraid.c`, `qedf/`, `qedi/`, `qla1280.c`, `qla2xxx/`, `qla4xxx/`, `snic/`, `stex.c`, `storvsc_drv.c`, `vmw_pvscsi.c`, `xen-scsifront.c`
- **Software/virtual**: `scsi_debug.c`, `iscsi_tcp.c`, `iser.c` (under `infiniband/ulp/iser/` actually), `virtio_scsi.c`, `xen-scsifront.c`, `vmw_pvscsi.c`
- **FCoE software**: `fcoe/`
- **Bluetooth/USB-storage SCSI bridges**: USB-storage uses libusual (out of scope here, in `drivers/usb/`)
- **Compile-gated-off** (out of v0): `3w-*` (older 3ware), `aha152x.c`/`aha1542.c`/`aha1740.c` (legacy AHA), `a100u2w.c`, `a2091.c`/`a3000.c`/`a4000t.c` (Amiga), `am53c974.c`, `arm/` (ARM SCSI), `atari_scsi.c`, `atp870u.c`, `BusLogic.c`, `bvme6000_scsi.c`, `dc395x.c`, `dmx3191d.c`, `esp_scsi.c`, `fdomain*`, `FlashPoint.c`, `g_NCR5380.c`, `imm.c`, `ips.c`, `jazz_esp.c`, `mac53c94.c`, `mac_esp.c`, `mac_scsi.c`, `mesh.c`, `mvme147.c`, `mvme16x_scsi.c`, `myrb.c`, `myrs.c`, `NCR5380.c`, `nsp32.c`, `pas16.c`, `pcmcia/`, `ppa.c`, `qlogicfas408.c`, `qlogicpti.c`, `sgiwd93.c`, `sni_53c710.c`, `stex.c` (oldgen), `sun3_scsi.c`, `sun_esp.c`, `sym53c8xx_2/`, `t128.c`, `tmscsim.c`, `u14-34f.c`, `ufs/` (UFS — keep enabled for x86 laptop UFS storage), `wd33c93.c`, `wd719x.c`, `xen-scsifront.c`

### target (LIO)

`drivers/target/` (separate Tier-2 wrapper future)
- `target_core_*` (mid-layer)
- `iscsi/` (iSCSI target)
- `loopback/` (kernel-side SCSI loopback)
- `tcm_fc/` (FC target)
- `tcm_remote/`
- `sbp/` (FireWire SBP-2 target)

## Scope

This Tier-2 governs `/home/doll/linux-src/drivers/scsi/` (~210 files + `libsas/`, `libfc/`, `libfcoe/`, plus per-LLD subdirs), public headers `include/scsi/` (~30 headers), UAPI `include/uapi/scsi/{scsi_netlink, scsi_netlink_fc, scsi_bsg_fc, scsi_bsg_mpi3mr, scsi_bsg_ufs}.h`, plus the legacy include of `include/uapi/linux/sg.h` (sg.h moved out of scsi UAPI subdir).

## Compatibility contract — outline

### `/dev/sd*`, `/dev/sr*`, `/dev/st*`, `/dev/sch*`, `/dev/sg*`, `/dev/bsg/*` chardev/block

Naming + minor numbering byte-identical so udev rules + /etc/fstab + /etc/multipath.conf consume unchanged. `sg<N>` chardev preserved + `bsg/<host>:<target>:<lun>:<channel>` pseudo-tree present.

### SG_IO + SCSI_IOCTL_* IOCTLs

`SG_IO` (struct sg_io_v4 / sg_io_hdr) byte-identical so smartctl, sg3_utils, sdparm, multipath-tools, lsscsi, fcping, fcoeadm consume unchanged.

### `/proc/scsi/`

Legacy procfs:
- `/proc/scsi/scsi` — top-level device list
- `/proc/scsi/<ll>/<N>` — per-LLD instance dump

Format byte-identical (legacy tools sometimes still consume).

### sysfs surface

Per-host `/sys/class/scsi_host/host<N>/`:
- `proc_name`, `host_busy`, `can_queue`, `cmd_per_lun`, `host_no`, `state`, `unique_id`, `prot_capabilities`, `prot_guard_type`, `unchecked_isa_dma`, `cmd_size`, `nr_hw_queues`, `dixsupport`, `dif_suppoted`, `eh_deadline`, `supported_mode`, `active_mode`, `host_reset` (writable)

Per-target `/sys/class/scsi_target/target<H>:<C>:<T>/`:
- `state`, `expiry`

Per-device `/sys/class/scsi_device/<H>:<C>:<T>:<L>/`:
- `device/{vendor, model, rev, type, scsi_level, queue_depth, queue_type, queue_ramp_up_period, max_sectors, allow_restart, blacklist, ioerr_cnt, iorequest_cnt, iodone_cnt, evt_*, eh_timeout, timeout, expiry, max_target_blocks_*, modalias, ...}`
- `device/iocounterbits`
- `device/sas_address` (sas devices)
- `device/wwid`
- `device/cdl_supported`, `device/cdl_enable`
- `device/state` (writable: blocked / running / cancel / deleted / created / offline / transport-offline)
- `device/delete` (writable: delete LUN)
- `device/rescan` (writable)
- `device/scsi_disk/<dev>/{cache_type, FUA, manage_start_stop, allow_restart, manage_runtime_start_stop, app_tag_own, max_*, provisioning_mode, thin_provisioning, zoned_cap, ...}`

Per-transport-class:
- `/sys/class/sas_phy/`, `/sys/class/sas_port/`, `/sys/class/sas_end_device/`, `/sys/class/sas_expander/`, `/sys/class/sas_host/`, `/sys/class/sas_device/` (SAS)
- `/sys/class/fc_host/`, `/sys/class/fc_remote_ports/`, `/sys/class/fc_transport/`, `/sys/class/fc_vports/` (FC)
- `/sys/class/iscsi_host/`, `/sys/class/iscsi_session/`, `/sys/class/iscsi_connection/`, `/sys/class/iscsi_endpoint/`, `/sys/class/iscsi_iface/`, `/sys/class/iscsi_flashnode/` (iSCSI)
- `/sys/class/srp_host/`, `/sys/class/srp_remote_ports/`

Layout + content byte-identical so sg3_utils, lsscsi, multipath-tools, open-iscsi (iscsid + iscsiadm), fcping, smartctl, scsiadd, hpacucli, storcli64 consume unchanged.

### Netlink

`NETLINK_SCSITRANSPORT` byte-identical; iscsid + multipathd consume.

### scsi.ko module-param + sysctls

`scsi_mod.scan` (sync / async / manual) + `scsi_mod.default_dev_flags` + `scsi_mod.dev_flags` + `scsi_mod.scsi_logging_level` parsed identically. Sysctls byte-identical.

### scsi_debug

`scsi_debug.c` parameters + `/sys/bus/pseudo/drivers/scsi_debug/` writable knobs for in-tree blktests/regression suite. Behavior byte-identical so blktests scsi/* runs.

## Tier-3 docs governed by this Tier-2

(Phase D will add these incrementally; LLD-specific docs deferred to per-LLD Tier-3s as they're stabilized.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/scsi/scsi-mid-core.md` | `scsi.c` + `hosts.c` + `scsi_priv.h`: subsystem init + host registration + cmd lifecycle |
| `drivers/scsi/scsi-lib.md` | `scsi_lib.c` + `scsi_lib_dma.c`: blk-mq glue + SG list construction |
| `drivers/scsi/scsi-scan.md` | `scsi_scan.c`: bus scan + INQUIRY + LUN scan |
| `drivers/scsi/scsi-error.md` | `scsi_error.c`: EH state machine — abort → dev-reset → target-reset → bus-reset → host-reset |
| `drivers/scsi/scsi-devinfo.md` | `scsi_devinfo.c`: device-id blacklist + quirk flags |
| `drivers/scsi/scsi-dh.md` | `scsi_dh.c`: device-handler registration for multipath |
| `drivers/scsi/scsi-ioctl.md` | `scsi_ioctl.c`: legacy SCSI IOCTLs + SG_IO + bsg |
| `drivers/scsi/scsi-sysfs.md` | `scsi_sysfs.c`: per-host/target/device sysfs |
| `drivers/scsi/scsi-bsg.md` | `scsi_bsg.c`: block-SG passthrough |
| `drivers/scsi/scsi-netlink.md` | `scsi_netlink.c`: NETLINK_SCSITRANSPORT |
| `drivers/scsi/scsi-pm.md` | `scsi_pm.c`: per-host PM |
| `drivers/scsi/scsi-logging.md` | `scsi_logging.c`: sense-data decode + per-cmd error logging |
| `drivers/scsi/scsi-debugfs.md` | `scsi_debugfs.c` |
| `drivers/scsi/sd.md` | `sd.c` + `sd_dif.c` + `sd_zbc.c`: SCSI disk + T10 PI + ZBC |
| `drivers/scsi/sr.md` | `sr.c` + `sr_ioctl.c` + `sr_vendor.c`: SCSI CD/DVD/BD ROM |
| `drivers/scsi/st.md` | `st.c`: SCSI tape |
| `drivers/scsi/ch.md` | `ch.c`: SCSI media changer |
| `drivers/scsi/sg.md` | `sg.c`: generic SCSI chardev |
| `drivers/scsi/transport-sas.md` | `scsi_transport_sas.c`: SAS transport class |
| `drivers/scsi/transport-fc.md` | `scsi_transport_fc.c`: FC transport class |
| `drivers/scsi/transport-iscsi.md` | `scsi_transport_iscsi.c`: iSCSI transport class + netlink to iscsid |
| `drivers/scsi/transport-spi.md` | `scsi_transport_spi.c`: parallel SCSI (legacy) |
| `drivers/scsi/transport-srp.md` | `scsi_transport_srp.c`: SRP/IB transport class |
| `drivers/scsi/libsas.md` | `libsas/`: SAS lib — expander disc + phy/port/task |
| `drivers/scsi/libfc.md` | `libfc/`: FC lib — fabric login + Exchange mgmt + ELS |
| `drivers/scsi/libiscsi.md` | `libiscsi.c` + `libiscsi_tcp.c`: iSCSI PDU framing + login + R2T |
| `drivers/scsi/libfcoe.md` | `libfcoe/`: FCoE Ethernet encapsulation + FIP |
| `drivers/scsi/scsi-debug.md` | `scsi_debug.c`: in-tree pseudo-host for testing |
| `drivers/scsi/iscsi-tcp.md` | `iscsi_tcp.c`: software iSCSI initiator |
| `drivers/scsi/virtio-scsi.md` | `virtio_scsi.c`: virtio-SCSI guest LLD |
| `drivers/scsi/storvsc.md` | `storvsc_drv.c`: Hyper-V Storage VSC LLD |
| `drivers/scsi/vmw-pvscsi.md` | `vmw_pvscsi.c`: VMware paravirt SCSI LLD |
| `drivers/scsi/xen-scsifront.md` | `xen-scsifront.c`: Xen PV SCSI front |

(LLD-specific Tier-3s for mpt3sas, megaraid_sas, lpfc, qla2xxx, smartpqi, hpsa, hisi_sas, etc., deferred to Phase D as each driver is stabilized; each gets its own Tier-3.)

## Compatibility outline (top-level)

- REQ-O1: `/dev/sd*`, `/dev/sr*`, `/dev/st*`, `/dev/sch*`, `/dev/sg*`, `/dev/bsg/*` naming + IOCTLs byte-identical (sg3_utils + smartctl + lsscsi consume unchanged).
- REQ-O2: `/proc/scsi/` legacy procfs format byte-identical.
- REQ-O3: Per-host + per-target + per-device + per-transport-class sysfs surface byte-identical.
- REQ-O4: SG_IO (sg_io_hdr + sg_io_v4) byte-identical so smartctl/sdparm/sg_inq/etc. work unchanged.
- REQ-O5: NETLINK_SCSITRANSPORT byte-identical (iscsid + multipathd consume).
- REQ-O6: All transport classes (sas/fc/iscsi/spi/srp) sysfs surface byte-identical.
- REQ-O7: libsas + libfc + libiscsi + libfcoe API source-compatible for in-tree LLDs.
- REQ-O8: Error-recovery state machine + escalation (abort → dev-reset → target-reset → bus-reset → host-reset) byte-identical.
- REQ-O9: scsi_dh framework (device-handler for multipath) source-compat for kernel-side handlers (rdac, alua, hp_sw, emc, etc.).
- REQ-O10: T10 PI (sd_dif) + ZBC (sd_zbc) work identically; in-tree consumers (zonefs, btrfs zoned-mode, f2fs zoned-mode) work unchanged.
- REQ-O11: scsi_debug pseudo-host parameters + sysfs surface byte-identical so blktests scsi/* runs.
- REQ-O12: TLA+ models declared at this Tier-2 (EH escalation, scan-async LUN-discovery, transport-class device hotplug, libsas expander rediscovery, libfc Exchange-table consistency).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on SCSI command-buffer construction + sense-data parsing + sg-list arithmetic.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with SCSI-specific reinforcement (SG_IO write-cmd CAP_SYS_RAWIO, /dev/sg* LSM-mediated, transport-class FC+iSCSI mgmt-cmd allowlist).

## Acceptance Criteria (top-level)

- [ ] AC-O1: `lsscsi -v` on a multi-LUN SAS HBA produces output byte-identical to upstream baseline. (covers REQ-O1, REQ-O3)
- [ ] AC-O2: `smartctl -a /dev/sda` works on SAS + SATA-via-SAS + USB-storage devices. (covers REQ-O4)
- [ ] AC-O3: `iscsiadm -m discovery -t st -p <ip>` discovers LIO target; subsequent `iscsiadm -m node --login` connects + `/dev/sd<X>` appears + IO works. (covers REQ-O1, REQ-O5, REQ-O6)
- [ ] AC-O4: `multipath -v3` shows correct path-state for multipath SAS array via scsi_dh_alua. (covers REQ-O9)
- [ ] AC-O5: T10 PI test: write/read with sd_dif APP_TAG check; corruption injected via scsi_debug + correctly detected. (covers REQ-O10)
- [ ] AC-O6: ZBC test: SMR drive enumerates as zoned-block-device; zone-mgmt-send via sg_zone works. (covers REQ-O10)
- [ ] AC-O7: scsi_debug + blktests `scsi/*` test suite passes. (covers REQ-O11, REQ-O12)
- [ ] AC-O8: virtio-scsi LLD test: qemu `-device virtio-scsi-pci -drive id=hd0,if=none,format=raw,file=disk.img -device scsi-hd,drive=hd0` exposes `/dev/sda` in guest. (covers REQ-O7)
- [ ] AC-O9: FC LLD test (lpfc or qla2xxx): switched FC fabric login completes; `/sys/class/fc_host/host<N>/{port_name, node_name, port_state}` populated. (covers REQ-O6, REQ-O7)
- [ ] AC-O10: SAS LLD test (mpt3sas): expander discovery enumerates downstream phys + SATA tunneled through libsas-sata. (covers REQ-O7)
- [ ] AC-O11: Tape test: `mt -f /dev/st0 status` + `tar -cf /dev/st0 ...` + `tar -tf /dev/st0` round-trip. (covers REQ-O1)
- [ ] AC-O12: Error injection: scsi_debug fault-inject CHECK CONDITION sense-key MEDIUM_ERROR triggers EH abort → dev-reset escalation correctly. (covers REQ-O8)

## Verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/scsi/eh_escalation.tla` | `drivers/scsi/scsi-error.md` (proves: error-recovery state machine — abort → dev-reset → target-reset → bus-reset → host-reset escalation; concurrent in-flight commands during EH terminate cleanly; no command both completed-on-old-state and re-issued-on-new) |
| `models/scsi/scan_async.tla` | `drivers/scsi/scsi-scan.md` (proves: async LUN-scan correctness — concurrent host-add + LUN scan + remove never produces ghost LUN or missed LUN; sync-scan and async-scan equivalent at quiescent points) |
| `models/scsi/transport_hotplug.tla` | `drivers/scsi/transport-sas.md` + `drivers/scsi/transport-fc.md` (proves: per-transport hotplug — phy-event / FC RSCN triggers expander rediscovery / fabric rescan; concurrent hotplug add + remove + IO never produces UAF on rport object) |
| `models/scsi/libsas_disco.tla` | `drivers/scsi/libsas.md` (proves: libsas expander discovery — broadcast-change handling + SMP DISCOVER walk + per-phy state machine; redundant broadcasts coalesce, no missed phy-state-change) |
| `models/scsi/libfc_exch.tla` | `drivers/scsi/libfc.md` (proves: libfc Exchange table — per-OXID/RXID lookup + sequence-state mgmt + retransmission; no two concurrent sequences claim the same OXID; abort-Exchange tearcdown cleanly) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/scsi/scsi-lib.md` | `scsi_alloc_request` post-condition: returned `scsi_cmnd` has cmd_len <= MAX_COMMAND_SIZE; sg-list mapped; sense-buffer allocated; refcount=1 |
| `drivers/scsi/scsi-error.md` | EH escalation invariant: each level invoked at most once per cmd before escalating; on host-reset, all in-flight cmds either completed or re-queued (no leaked inflight) |
| `drivers/scsi/sd.md` | LBA arithmetic invariant: every sd command's LBA range fits within `sdkp->capacity` (no wrap, no out-of-range read/write); WRITE_SAME / UNMAP / WRITE_ZEROES bounds-checked the same way |

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-host + per-target + per-device + per-cmd refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-LLD `scsi_host_template`, per-transport `scsi_transport_template`, per-ULD `scsi_driver` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | LBA arithmetic + sg-list byte counts + cmd-buffer + sense-buffer arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-LLD scsi_host_template + per-transport templates read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed scsi_cmnd + sense-buffer + per-device state cleared (carries SED-Opal keys, SCSI passthrough cmd buffers) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | SCSI has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for SCSI

LSM hooks called: file-LSM hooks on `/dev/sg*` open + ioctl, `/dev/bsg/*` open + ioctl, `/sys/class/scsi_*/.../{state,delete,rescan,host_reset}` writes.

GR-RBAC adds:
- Per-role disallow `/dev/sg*` open (denies SCSI passthrough).
- Per-role disallow `SG_IO` write-cmd opcodes (FORMAT_UNIT, WRITE_LONG, MODE_SELECT, LOG_SELECT, SET_*) but allow read-cmd opcodes (INQUIRY, READ_CAPACITY, READ_*, MODE_SENSE, LOG_SENSE).
- Per-role disallow `host_reset` / `delete` / `rescan` writes.
- Per-role disallow per-transport mgmt cmds (FC vport_create/delete, iSCSI session login).
- Per-role audit of every SG_IO write-cmd execution (which uid, which opcode, which LUN).

### SCSI-specific reinforcement

- **SG_IO write-cmd CAP_SYS_RAWIO**: write-cmd opcodes (FORMAT_UNIT, WRITE_LONG, etc.) require CAP_SYS_RAWIO in the init userns; read-cmd opcodes only require CAP_SYS_RAWIO if the device is opened O_RDWR.
- **Transport-class management-cmd allowlist**: per-transport (fc/iscsi/sas) management commands invoked via sysfs writes go through a per-transport allowlist + LSM mediation; arbitrary commands not in allowlist rejected.
- **scsi_dh device-handler registration**: only kernel-built-in handlers may register (no module loading paths from userspace via netlink).
- **EH state-machine recovery quotas**: per-host EH command count + EH timeout capped (default 60s host_reset deadline) — prevents buggy LLD from blocking EH thread indefinitely.
- **Scan-async serialization**: per-host async-scan workqueue concurrency capped (default 4 parallel scans) to prevent scan-storm overwhelming weak HBAs.
- **bsg passthrough opcode validation**: bsg requests validated against the same allowlist as SG_IO.
- **iSCSI iface secrets keyring-shielded**: CHAP secrets stored in kernel keyring (cross-ref `security/keys/`) not raw in iface attrs.

## Open Questions

(none at Tier-2 — defer to per-Tier-3 once Phase D begins; expected hot questions: which legacy LLDs to keep compile-gated-off vs delete entirely, scsi-mq default queue-depth, EH timeout default for SAS vs FC vs iSCSI, T10 PI default-on-when-supported)

## Out of Scope

- Per-Tier-3 (Phase D) — most LLD Tier-3s deferred to as-stabilized
- ARM/PPC/Sparc/m68k/Alpha legacy LLDs (`a2091.c`, `a3000.c`, `a4000t.c`, `arm/`, `atari_scsi.c`, `bvme6000_scsi.c`, `jazz_esp.c`, `mac*.c`, `mvme*.c`, `sgiwd93.c`, `sni_53c710.c`, `sun3_scsi.c`, `sun_esp.c`, etc.) — out of v0
- Most legacy ISA HBAs (`aha152x.c`, `aha1542.c`, `aha1740.c`, `BusLogic.c`, `dc395x.c`, `dmx3191d.c`, `fdomain*`, `g_NCR5380.c`, `imm.c`, `ips.c`, `nsp32.c`, `pas16.c`, `ppa.c`, `qlogicfas408.c`, `t128.c`, `tmscsim.c`, `u14-34f.c`, `wd*.c`) — compile-gated off
- 3w-* (older 3ware), AIC79xx generation 1 — compile-gated off
- pcmcia/ — out of v0 (legacy PC-Card)
- LIO target proper (`drivers/target/`) — separate Tier-2 wrapper future
- 32-bit-only paths
- Implementation code
