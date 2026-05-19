# Tier-3: drivers/cxl/ — Compute Express Link subsystem (CXL.io / CXL.cache / CXL.mem, type-1/2/3 devices, HDM decoders)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cxl/00-overview.md
upstream-paths:
  - drivers/cxl/acpi.c
  - drivers/cxl/pci.c
  - drivers/cxl/mem.c
  - drivers/cxl/port.c
  - drivers/cxl/pmem.c
  - drivers/cxl/security.c
  - drivers/cxl/cxl.h
  - drivers/cxl/cxlmem.h
  - drivers/cxl/cxlpci.h
  - drivers/cxl/core/mbox.c
  - drivers/cxl/core/memdev.c
  - drivers/cxl/core/port.c
  - drivers/cxl/core/pci.c
  - drivers/cxl/core/regs.c
  - drivers/cxl/core/cdat.c
  - drivers/cxl/core/ras.c
-->

## Summary

The Compute Express Link (CXL) subsystem implements Linux support for the three CXL transaction protocols layered on a PCIe physical link: **CXL.io** (PCIe-equivalent enumeration + configuration + interrupts), **CXL.cache** (device-initiated cache-coherent access to host memory — type-1 / type-2 accelerators), and **CXL.mem** (host-initiated load/store access to device-attached memory — type-2 / type-3 devices). Three device classes are recognized: **type-1** (coherent accelerator with cache; CXL.io + CXL.cache), **type-2** (coherent accelerator with HDM device memory; all three protocols), **type-3** (memory expander / pooling device; CXL.io + CXL.mem). Host-Managed Device Memory (HDM) is mapped into system physical address space through a topology of **HDM decoders** at root, host-bridge, switch-port, and endpoint levels — each decoder programs a System Physical Address (SPA) range, target list, interleave granularity, and interleave ways. The subsystem covers ACPI CEDT (CXL Early Discovery Table) parsing, CDAT (Coherent Device Attribute Table) ingestion, PCI DVSEC enumeration, mailbox command transport (CXL 2.0 §8.2.8.4), event log poll/IRQ, memdev cdev for `/dev/cxl/memN` IOCTL UAPI, region assembly into NUMA-attached HMAT-aware memory tiers, and an nvdimm-bridge for persistent CXL regions exposed as libnvdimm namespaces. RAS coverage is provided through AER (PCIe-side) plus CXL RAS capability (per-component uncorrectable / correctable masks + header log + first-error pointer).

This Tier-3 covers the subsystem root: ACPI provider (`acpi.c`), PCI binder (`pci.c`), endpoint memdev driver (`mem.c`), upstream-port driver (`port.c`), libnvdimm bridge (`pmem.c`, `security.c`), and the shared cxl_core mailbox / memdev / port / region-scaffold infrastructure (`core/mbox.c`, `core/memdev.c`, `core/port.c`, `core/cdat.c`, `core/regs.c`, `core/ras.c`). HDM decoder programming + region assembly are covered in `cxl/core-region.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cxl_port` | per-port topology node (root / host-bridge / switch / endpoint) | `drivers::cxl::Port` |
| `struct cxl_dport` | downstream-port child of a port | `drivers::cxl::Dport` |
| `struct cxl_memdev` | type-3 memory device bus object + cdev | `drivers::cxl::Memdev` |
| `struct cxl_dev_state` | per-device backing state (regs, mailbox, partitions) | `drivers::cxl::DevState` |
| `struct cxl_memdev_state` | per-memdev extra state (security, FW, events) | `drivers::cxl::MemdevState` |
| `struct cxl_mailbox` | per-device primary/secondary mailbox | `drivers::cxl::Mailbox` |
| `struct cxl_mbox_cmd` | mailbox command envelope (opcode + in/out payload) | `drivers::cxl::MboxCmd` |
| `struct cxl_decoder` / `cxl_root_decoder` / `cxl_switch_decoder` / `cxl_endpoint_decoder` | HDM decoder per topology level | `drivers::cxl::Decoder*` |
| `struct cxl_region` | activated HDM region in SPA space | `drivers::cxl::Region` |
| `struct cxl_nvdimm_bridge` / `cxl_nvdimm` | libnvdimm shim for CXL-PMEM regions | `drivers::cxl::NvdimmBridge` |
| `cxl_acpi_probe()` / `cxl_pci_probe()` / `cxl_mem_probe()` / `cxl_port_probe()` | per-stratum binder probes | `Subsystem::probe_*` |
| `cxl_internal_send_cmd(mbox, cmd)` | in-kernel mailbox submit | `Mailbox::send` |
| `cxl_pci_setup_mailbox(cxlds)` | enumerate + map primary mailbox MMIO | `DevState::setup_mailbox` |
| `cxl_setup_regs(map)` / `cxl_probe_component_regs(...)` / `_device_regs(...)` | enumerate per-cap registers | `DevState::setup_regs` |
| `cxl_await_media_ready(cxlds)` / `cxl_dvsec_rr_decode(...)` | gate enumeration on media-ready + parse DVSEC range regs | `DevState::await_media_ready` |
| `cxl_dpa_setup(cxlds, info)` / `cxl_mem_get_partition_info(...)` | partition map (volatile / persistent) | `DevState::dpa_setup` |
| `cxl_cor_error_to_ras()` / `cxl_uncor_error_to_ras()` | CXL RAS log lifting to MCE/EDAC | `Ras::report_*` |
| `devm_cxl_setup_fw_upload(host, mds)` | firmware-upload state machine | `MemdevState::setup_fw_upload` |
| `cxl_security_ops` | nvdimm_security_ops adapter | `Pmem::security_ops` |

## Compatibility contract

REQ-1: ACPI CEDT parsing (CHBS, CFMWS, CXIMS, RDPAS sub-tables) yields the root CXL topology: each CHBS produces a host-bridge port; each CFMWS produces a root decoder with SPA window + interleave + QTG.

REQ-2: PCI binding occurs against class code `0x050210` (CXL memory device) and the CXL DVSEC ID range; non-DVSEC PCIe endpoints under a CXL host-bridge bind via `cxl_port` only.

REQ-3: Primary mailbox transport per CXL 2.0 §8.2.8.4: payload-size CAP, doorbell control, background-command bit, mailbox-ready timeout (cmdline-tunable `cxl_pci.mbox_ready_timeout`, default 60s), per-command timeout `CXL_MAILBOX_TIMEOUT_MS` (2 HZ).

REQ-4: Mailbox command allowlist (`cxl_mem_commands[]`) bounds which opcodes pass through the `CXL_MEM_QUERY_COMMANDS` / `CXL_MEM_SEND_COMMAND` IOCTLs; security-set opcodes (0x44 Sanitize, 0x45 PMEM Security, 0x46 Security Passthrough) blocked from RAW path.

REQ-5: CONFIG_CXL_MEM_RAW_COMMANDS opt-in for unfiltered opcode submission; even when on, `cxl_disabled_raw_commands[]` rejects opcodes that bypass kernel coordination (ACTIVATE_FW, SET_PARTITION_INFO, SET_LSA, SET_SHUTDOWN_STATE, SCAN_MEDIA, POISON_*).

REQ-6: Topology probe walks endpoint → switch → host-bridge → root, creating one `cxl_port` per device on the path with `dport` children for each downstream link; depth recorded for HDM decoder enumeration ordering.

REQ-7: CDAT enumeration via DOE (Data Object Exchange) reads device + switch attributes (DSMAS, DSLBIS, DSEMTS, SSLBIS) and feeds the per-region access-coordinates (read/write bandwidth + latency) consumed by HMAT memory tiering.

REQ-8: Event-log servicing via poll (default) or MSI/MSI-X interrupt mode (mailbox CAP `IRQ_MSGNUM`); per-severity sinks (info / warn / fail / fatal) drain to tracepoints + memdev event ring.

REQ-9: Persistent CXL regions register an `nvdimm_bridge` → `nvdimm_bus` → `nd_region` so that libnvdimm namespaces, BTT, and labels apply uniformly; `cxl_security_ops` adapts the nvdimm security state machine onto CXL `GET/SET_PASSPHRASE`, `UNLOCK`, `FREEZE_SECURITY`, `PASSPHRASE_SECURE_ERASE` mailbox opcodes.

REQ-10: PCI AER plus CXL RAS capability provide layered error reporting: PCIe uncorrectable / correctable AER for CXL.io errors, CXL RAS regs for CXL.cache / CXL.mem errors with first-error-pointer + 512-byte header log.

REQ-11: Firmware-upload state machine (multi-packet `TRANSFER_FW` + `ACTIVATE_FW`) implemented atop the kernel firmware-upload helper with progress reported via sysfs.

REQ-12: Suspend integration: `cxl_mem_active_inc / _dec` blocks S3 while any memdev is bound (volatile CXL memory does not survive S3 unmodified).

## Acceptance Criteria

- [ ] AC-1: `ls /sys/bus/cxl/devices/` enumerates `rootN`, `port*`, `decoderN.*`, `endpointN`, `memN` after probe on a CEDT-bearing platform (or QEMU `-machine q35,cxl=on`).
- [ ] AC-2: `cxl list -v` (ndctl/cxl-cli) reports each memdev with payload-size, partition map (volatile + persistent), and security state.
- [ ] AC-3: `CXL_MEM_QUERY_COMMANDS` IOCTL on `/dev/cxl/memN` returns the allowlisted opcode table; `CXL_MEM_SEND_COMMAND` round-trip with `GET_HEALTH_INFO` succeeds.
- [ ] AC-4: Region create / commit on a CFMWS-backed root decoder produces a `regionN` device which auto-onlines memory under a new NUMA node with HMAT-tiered bandwidth/latency.
- [ ] AC-5: Persistent region create yields `/dev/pmemN` after BTT/namespace activation; security passphrase set + unlock cycle exercised through `ndctl`.
- [ ] AC-6: Injected uncorrectable RAS error (debugfs) surfaces in `dmesg` with first-error-pointer + header log and lifts into the EDAC channel.
- [ ] AC-7: `cxl-cli poison get` and `cxl-cli poison inject/clear` exercise the in-kernel poison-list mediation path (raw opcode blocked, mediated opcode allowed under CAP_SYS_ADMIN).

## Architecture

The subsystem layers into:

```
+--- cxl_acpi   (drivers/cxl/acpi.c)         parse CEDT / CFMWS / CHBS / CXIMS → root + root-decoders
|
+--- cxl_pci    (drivers/cxl/pci.c)          per-endpoint PCI bind: map regs, mailbox setup, AER+RAS, event IRQ
|       |
|       +-- cxl_core / mbox.c                payload size, IOCTL allowlist, command exec + timeout
|       +-- cxl_core / memdev.c              cxl_memdev cdev /dev/cxl/memN, sysfs attrs, partition info
|       +-- cxl_core / regs.c                CXL_DEVICE_CAPS array walk (status, mbox, memdev caps)
|       +-- cxl_core / cdat.c                DOE-based CDAT read + DSMAS/DSLBIS → access_coordinates
|       +-- cxl_core / ras.c                 CXL RAS cap drain → MCE/EDAC
|
+--- cxl_port   (drivers/cxl/port.c)         per-port (root/host-bridge/switch/endpoint) bind
|       |
|       +-- cxl_core / port.c                topology walk, dport/dport_target, depth tracking
|       +-- cxl_core / hdm.c                 HDM decoder enumerate + setup (covered in core-region.md)
|       +-- cxl_core / region.c              region create/commit/teardown (covered in core-region.md)
|
+--- cxl_mem    (drivers/cxl/mem.c)          memdev → endpoint-port join, debugfs (poison, dpa)
|
+--- cxl_pmem   (drivers/cxl/pmem.c)         nvdimm_bridge + nd_region for persistent partition
        |
        +-- security.c                       nvdimm_security_ops → mailbox passphrase ops
```

Per-endpoint probe (cxl_pci):
1. `cxl_pci_probe(pdev)` allocates `cxl_dev_state` + `cxl_memdev_state`.
2. `cxl_pci_setup_regs(pdev, map)` walks PCI DVSEC + component-regs array discovering `mbox`, `status`, `memdev`, `ras` capabilities.
3. `cxl_pci_setup_mailbox(cxlds)` reads payload size, doorbell, IRQ message-number; waits ready under `mbox_ready_timeout`.
4. `cxl_await_media_ready(cxlds)` polls media-ready bit; `cxl_dvsec_rr_decode(...)` reads DVSEC range regs for emulation fallback.
5. `cxl_dev_state_identify(mds)` issues `IDENTIFY` opcode.
6. `cxl_mem_get_partition_info(mds)` issues `GET_PARTITION_INFO` (volatile + persistent extents).
7. `cxl_dpa_setup(cxlds, info)` populates DPA resource tree.
8. `devm_cxl_add_memdev(cxlds, &cxl_mem_attach)` registers `cxl_memdev` on cxl_bus; cdev `/dev/cxl/memN` exposes IOCTL UAPI.
9. AER + CXL RAS handlers registered; event-log IRQ requested if MSI/MSI-X mode supported.
10. `cxl_mem_active_inc()` blocks S3.

Mailbox dispatch (`cxl_internal_send_cmd` / `cxl_pci_mbox_send`):
1. Acquire per-device `mbox_mutex`.
2. Wait doorbell clear (`cxl_pci_mbox_wait_for_doorbell`).
3. Write opcode + payload-length + payload to MB registers.
4. Set doorbell; either spin-wait or sleep on completion.
5. On completion, read return-code (mapped via `CMD_CMD_RC_TABLE` to errno), copy payload-out, clear doorbell.
6. Background-command opcodes: poll status word until BG_ABORT_REQ / BG_DONE.

UAPI surface (`include/uapi/linux/cxl_mem.h`):
- `CXL_MEM_QUERY_COMMANDS` — enumerate per-device allowlisted opcodes + max payload.
- `CXL_MEM_SEND_COMMAND` — submit an allowlisted opcode (size_in / size_out per table); CAP_SYS_ADMIN required.

## Hardening

- **Mailbox payload bounded by CAP register** — driver refuses sends with `size_in > cap.payload_size`; refuses receives whose declared `out_length > cap.payload_size`; defense against oversized-payload bounds violation in MMIO copy.
- **Mailbox command table allowlist** — `cxl_mem_commands[]` enumerates supported opcodes with fixed `size_in / size_out`; user requests with mismatched sizes rejected.
- **RAW opcode gate** — `CONFIG_CXL_MEM_RAW_COMMANDS` opt-in; even when on, security opcode sets (0x44/0x45/0x46) and FW-activate / partition-set / scan-media / poison-* are hard-blocked.
- **GET_LSA bounded** — Label Storage Area reads bounded by per-device LSA size from `GET_LSA_CAP`; defense against unbounded LSA copy.
- **Doorbell timeout** — `CXL_MAILBOX_TIMEOUT_MS` (2 HZ) bounds spin on stuck doorbell; mailbox-ready wait bounded by `mbox_ready_timeout`.
- **Event-log ring bounded** — per-severity ring with drop-counter on overflow; defense against event-flood causing memory pressure.
- **AER + CXL RAS handlers ratelimited** — `dev_err_ratelimited` on uncorrectable error paths bounds dmesg spew.
- **HDM decoder programming gated by region rwsem** (cross-ref `cxl/core-region.md`) — defense against concurrent reconfig races.
- **S3 suspend block while bound** — volatile CXL memory cannot survive suspend cleanly; `cxl_mem_active_inc` interlocks PM.
- **Per-region access_coordinates derived from CDAT** — refuses to fabricate latency/bandwidth if DSLBIS missing.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `cxl_memdev`, `cxl_dev_state`, `cxl_memdev_state`, `cxl_port`, `cxl_decoder*`, `cxl_region`, `cxl_mbox_cmd`, mailbox payload bounce buffers; reject userspace copies outside `payload_size`.
- **PAX_KERNEXEC** — CXL core entry points (`cxl_pci_probe`, `cxl_internal_send_cmd`, `cxl_setup_regs`, `cxl_pci_mbox_send`, AER + RAS callbacks) in `__ro_after_init` text; W^X enforced on probe/IRQ paths.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `cxl_memdev_ioctl`, mailbox-send entry, event-log IRQ, AER handler.
- **PAX_REFCOUNT** — saturating `refcount_t` on `cxl_port`, `cxl_memdev`, `cxl_region`, `cxl_nvdimm_bridge`, `cxl_mailbox`; overflow trap defeats unbind-race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for mailbox payload buffers, LSA cache, security-passphrase staging, event-log ring, HDM decoder shadow state so stale device data, plaintext passphrases, and label-area contents cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `cxl_memdev` cdev IOCTL entry; reject user pointers outside the QUERY/SEND envelope helpers.
- **PAX_RAP / kCFI** — `nvdimm_security_ops`, `dax_operations`, `nvdimm_bus_descriptor.ndctl`, `cxl_mbox_cmd` dispatch vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-mailbox / per-region pointer disclosure behind CAP_SYSLOG; suppress `%p` in mailbox/event/RAS tracepoints.
- **GRKERNSEC_DMESG** — restrict mailbox-timeout, AER, CXL-RAS first-error-log, and poison-inject banners to CAP_SYSLOG so attackers cannot probe device behavior via dmesg.
- **CAP_SYS_ADMIN strict on mailbox** — `CXL_MEM_SEND_COMMAND` and `CXL_MEM_QUERY_COMMANDS` IOCTLs require CAP_SYS_ADMIN in the owning user namespace; debugfs poison-inject + sanitize gated identically.
- **Vendor-command allowlist** — vendor-specific opcodes refused unless explicitly added to the per-device allowlist at probe; defense against undocumented opcode classes used as covert channels.
- **GET_LSA bounded** — LSA reads policed against device `GET_LSA_CAP` size; refusal on overflow attempts.
- **RAW opcode gate hardened** — even with `CONFIG_CXL_MEM_RAW_COMMANDS=y`, `cxl_disabled_raw_commands[]` and security-command-set checks remain enforced unconditionally.
- **Suspend lockdown** — refuse to resume from S3 if any memdev's media-ready bit fails revalidation; defense against silent volatile-memory corruption across suspend.

Rationale: CXL endpoints are firmware-driven memory devices on the cache-coherent attach point of the platform; the mailbox is a privileged command channel into the device firmware, and HDM decoders directly program the physical-address topology. A relaxed mailbox allowlist, an unbounded GET_LSA, a missed security-set classification, or a refcount underflow on `cxl_memdev` will leak passphrases, corrupt the platform memory map, or hand userspace a vendor-cmd primitive into the device firmware. CAP_SYS_ADMIN on IOCTL, RAW-opcode hard-gating, vendor-opcode allowlisting, and zero-on-free of passphrase buffers turn CXL from "trust the device firmware" into a mediated subsystem.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mbox_payload_no_oob` | OOB | mailbox in/out copies bounded by `cap.payload_size`; userspace QUERY/SEND envelopes validated before MMIO copy. |
| `opcode_allowlist_total` | TOTALITY | every dispatched opcode resolves to a known entry in `cxl_mem_commands[]` or is rejected before MMIO write. |
| `memdev_refcount_no_uaf` | UAF | `cxl_memdev` Arc/refcount outlives cdev `/dev/cxl/memN` open file lifetime. |
| `event_ring_no_oob` | OOB | per-severity event ring head/tail bounded by ring size; drop-counter on full. |

### Layer 2: TLA+

`models/cxl/mailbox_doorbell.tla` (parent-declared): models doorbell + background-command + completion ordering between concurrent send and event-log IRQ drain; proves no command is lost across a background-abort race.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mailbox::send` post: when `Ok`, payload-out length ≤ `cap.payload_size` AND ≤ user-declared `size_out`. | `core::mbox` |
| Opcode-table lookup is total over `cxl_mem_commands[]` range; UNKNOWN opcode → `-ENOTTY`. | `core::mbox` |
| Refcount on `cxl_memdev` saturates rather than wraps; `unbind` blocks while cdev opens > 0. | `core::memdev` |

### Layer 4: Verus/Creusot functional

Mailbox send: userspace `CXL_MEM_SEND_COMMAND(opcode, in)` → allowlist lookup → MMIO transfer → device completion → return-code mapped via `CMD_CMD_RC_TABLE` → payload-out copied bounded by min(out_cap, user_size) → `Ok` to userspace; encoded as a Verus refinement from UAPI envelope to MMIO command word.

## Out of Scope

- HDM decoder programming + region assembly (covered in `cxl/core-region.md`)
- CXL switch fabric routing details
- CXL.cache protocol formal model
- vendor-specific CXL features (e.g. Intel CXL-DCD, Samsung CMM-H)
- CXL FM-API (Fabric Manager) interface
