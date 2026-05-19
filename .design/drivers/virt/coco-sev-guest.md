# Tier-3: drivers/virt/coco/sev-guest/sev-guest.c — AMD SEV-SNP guest driver (/dev/sev-guest, attestation report + derived key, VMPCK channel)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/virt/coco/sev-guest/sev-guest.c
  - drivers/virt/coco/sev-guest/Kconfig
  - drivers/virt/coco/sev-guest/Makefile
  - include/uapi/linux/sev-guest.h
-->

## Summary

`sev-guest` is the in-kernel attestation surface for an AMD SEV-SNP confidential guest. It binds to the platform device created by the SNP early-boot CPU code and exposes `/dev/sev-guest` (misc-char device) to userspace. Through three ioctls — `SNP_GET_REPORT`, `SNP_GET_DERIVED_KEY`, `SNP_GET_EXT_REPORT` — a relying-party agent inside the VM can ask the AMD-SP (Secure Processor) for a freshly minted SNP attestation report keyed to user-supplied REPORTDATA, or for a derived key sealed to TCB/VMPL state. All requests cross the untrusted host through a VMPCK-encrypted GHCB message channel with a strict monotonically-increasing sequence counter; replay or VMM forgery results in driver lockout. The driver also registers a `tsm_report_ops` provider so the generic TSM (Trusted Security Manager) configfs surface (`/sys/kernel/config/tsm/report/...`) can route reports through SEV-SNP. Optional SVSM (Secure VM Service Module) at VMPL0 short-circuits the request to a paravisor rather than the host-mediated path. Driver scope is strictly guest-side; SNP firmware programming and per-page RMP table maintenance live in `arch/x86/kvm/svm/sev.c` and out of scope here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snp_guest_dev` | per-instance device state (misc + AEAD ctx + msg seqnos) | `drivers::sev_guest::Device` |
| `snp_guest_probe(pdev)` / platform-driver bind | parse platform-data, register misc, initialize VMPCK | `Device::probe` |
| `snp_guest_ioctl(file, cmd, arg)` | dispatch SNP_GET_REPORT / SNP_GET_DERIVED_KEY / SNP_GET_EXT_REPORT | `Device::ioctl` |
| `get_report(dev, &input)` | build MSG_REPORT_REQ, sign, exchange, copy 4 KB resp | `Device::get_report` |
| `get_derived_key(dev, &input)` | build MSG_KEY_REQ from root_key_select + guest_field_select + VMPL + TCB | `Device::get_derived_key` |
| `get_ext_report(dev, &input, &cert_buf)` | report + cached cert chain blob via shared certs page | `Device::get_ext_report` |
| `sev_report_new(report, data)` / `sev_svsm_report_new(...)` | tsm_report_ops backend (host or SVSM path) | `Device::tsm_report_new` |
| `snp_send_guest_request(dev, req, resp)` (in arch glue) | issue GHCB MSG protocol with VMPCK AEAD | `Device::send_request` |
| `vmpck_id` modparam | which Virtual Machine Platform Communication Key (0..3) to use | `Device::vmpck_id` |
| `snp_disable_vmpck(dev)` | one-way invalidation on integrity failure | `Device::disable_vmpck` |
| `SNP_GUEST_REQ_IOC_TYPE 'S'`, `SNP_GET_REPORT`, `SNP_GET_DERIVED_KEY`, `SNP_GET_EXT_REPORT` (uapi) | ioctl numbers | `Uapi::*` |

## Compatibility contract

REQ-1: Bind as a platform driver named `sev-guest` against the platform device created from the SNP early init path; refuse to load if CPU is not SEV-SNP active.

REQ-2: Register a misc-char device at `/dev/sev-guest` with `MISC_DYNAMIC_MINOR`; `file_operations` exposes only `unlocked_ioctl` (no read/write/mmap/llseek).

REQ-3: All three ioctls take `struct snp_guest_request_ioctl` referencing user `req_data` and `resp_data` pointers and return per-request VMM-error and FW-error fields on failure.

REQ-4: `SNP_GET_REPORT` produces a 1184-byte SNP attestation report with the 64-byte user-supplied REPORTDATA copied into the report body; output buffer is 4000 bytes per `struct snp_report_resp`.

REQ-5: `SNP_GET_DERIVED_KEY` returns a 32-byte AES-GCM key bound to (root_key_select, guest_field_select bitmap, VMPL, guest SVN, TCB version); excess `data[64]` covers AEAD framing.

REQ-6: `SNP_GET_EXT_REPORT` additionally copies a host-supplied VCEK/VLEK certificate-chain blob into a user buffer; if user buffer is too small the firmware error returns `SNP_GUEST_VMM_ERR_INVALID_LEN` and required length is reported in the certs_len field.

REQ-7: The VMPCK-encrypted MSG transport must use a monotonically increasing 64-bit sequence; on any AEAD verify failure the driver permanently disables the VMPCK for this session via `snp_disable_vmpck`.

REQ-8: `SVSM_MAX_RETRIES == 3` for the SVSM-mediated path; on exhaustion the request fails with `-EIO`.

REQ-9: When `CONFIG_TSM_REPORTS=y`, register `sev_tsm_report_ops` so configfs `/sys/kernel/config/tsm/report/<name>/` paths funnel into the same backend.

REQ-10: `msg_version` field must be non-zero; reject `0` with `-EINVAL` before issuing any guest request.

REQ-11: Compatible with both legacy host-mediated path (GHCB `SNP_GUEST_REQUEST`) and SVSM ATTEST_SERVICES paravisor path; selection is per-instance at probe.

## Acceptance Criteria

- [ ] AC-1: `modprobe sev-guest` on an SNP guest creates `/dev/sev-guest` with mode 0600 root:root.
- [ ] AC-2: `sevtool --ioctl SNP_GET_REPORT` (or `snpguest report`) on a real SNP-enabled VM returns a 4000-byte resp; offset of REPORTDATA matches the user-supplied nonce.
- [ ] AC-3: `snpguest key` returns 32 bytes that change when (`root_key_select`, `guest_field_select`) changes, are stable across repeated calls with same inputs, and differ from `derived_key` of a sibling VM.
- [ ] AC-4: `SNP_GET_EXT_REPORT` returns VCEK chain in DER; chain validates against AMD root via `openssl verify`.
- [ ] AC-5: Forced VMPCK tampering test (host-side fault injection) results in `EIO` plus permanent `vmpck_id` disable for that session; subsequent ioctls fail.
- [ ] AC-6: Replay test (host re-injecting an old MSG_REPORT_RSP) is rejected by AEAD nonce check; sequence counter never decreases.
- [ ] AC-7: `/sys/kernel/config/tsm/report/foo/outblob` returns a valid SEV-SNP report when TSM provider is `sev_guest`.
- [ ] AC-8: kselftest under `tools/testing/selftests/sev-guest/` (if present) passes on SNP host.

## Architecture

`Device` in `drivers::sev_guest::Device`:

```
struct Device {
  misc: MiscDevice,                  // /dev/sev-guest
  vmpck_id: u32,                     // 0..3, chosen at probe (modparam or platform)
  ctx: KBox<AeadCtx>,                // AES-256-GCM context with VMPCK key material
  msg_seqno: AtomicU64,              // monotonic, never wraps without disable
  certs_data: Option<NonNull<u8>>,   // shared-page mapping for ext-report cert blob
  secrets: NonNull<SnpSecretsPage>,  // BIOS-published Secrets Page (VMPCKs, etc.)
  request_mutex: Mutex<()>,          // serializes ioctl issuance
  tsm: TsmReportProvider,            // configfs ops provider
  svsm: Option<SvsmPath>,            // VMPL0 paravisor path if available
  fw_err: u32, vmm_err: u32,         // last error fields surfaced to userspace
}
```

ioctl dispatch `Device::ioctl`:
1. Copy `struct snp_guest_request_ioctl` from user.
2. Reject `msg_version == 0`.
3. Switch on cmd:
   - `SNP_GET_REPORT` → `get_report` (builds MSG_REPORT_REQ from `snp_report_req` user struct; copies 64 B REPORTDATA + 4 B VMPL + 28 B rsvd; sends via VMPCK channel; copies 4000 B response).
   - `SNP_GET_DERIVED_KEY` → `get_derived_key` (builds MSG_KEY_REQ from `snp_derived_key_req`; returns 32 B key + 32 B AEAD residue).
   - `SNP_GET_EXT_REPORT` → `get_ext_report` (combines report path with cached cert blob mapped through a shared (un-private) page; if user `certs_address` buffer is smaller than blob, returns required length in `certs_len` + `SNP_GUEST_VMM_ERR_INVALID_LEN`).
4. Populate `exitinfo2` with VMM-error << 32 | FW-error and copy_to_user.

VMPCK message protocol `Device::send_request`:
1. Acquire `request_mutex` (single in-flight VMPCK).
2. Allocate request and response message frames in private (encrypted) memory.
3. Build MSG header: msg_type, msg_version, msg_size, msg_seqno (= load + 1), msg_vmpck.
4. AES-256-GCM-encrypt the request payload with VMPCK as key and seqno as part of the IV.
5. Issue GHCB `SNP_GUEST_REQUEST` (or SVSM ATTEST_SERVICES) via the arch helper.
6. On return, AEAD-verify the response with VMPCK; on AEAD failure call `snp_disable_vmpck` and return `-EIO`.
7. Bump `msg_seqno` by 2 (request and expected response slots).
8. Copy response into the caller's output buffer.

SVSM path `Device::send_request_svsm` (when paravisor is present): same shape but goes through SVSM_CALL with up to `SVSM_MAX_RETRIES` retries on `SVSM_ERR_BUSY`; the SVSM at VMPL0 is treated as trusted and replaces the host-mediated leg.

TSM integration `sev_tsm_report_ops`: registered with TSM core in `drivers/virt/coco/tsm-core.c`; configfs attributes `inblob`, `outblob`, `provider`, `privlevel`, `service_provider` map directly onto MSG_REPORT_REQ inputs and outputs.

## Hardening

- All three ioctls require `CAP_SYS_ADMIN` against the device's user namespace; refuse from unprivileged callers regardless of `/dev/sev-guest` permissions.
- `msg_seqno` is `AtomicU64`; one-way disable on AEAD failure prevents replay or rollback attacks from a hostile host.
- VMPCK key material is loaded once from the Secrets Page and never returned to userspace; the page is mapped read-only after copy.
- Per-request buffers (REPORTDATA, derived-key inputs, response blobs) cross `copy_from_user`/`copy_to_user` boundaries with explicit size checks against UAPI sizes; no slab leakage on the response path because buffers are kzalloc'd and explicitly memset on free.
- `request_mutex` prevents concurrent ioctls from interleaving message frames on the shared GHCB channel.
- Bind refuses if `cc_platform_has(CC_ATTR_SEV_SNP)` is false, so the driver cannot bind on a non-SNP CPU and accidentally expose a counterfeit endpoint.
- Errors return both VMM-error and FW-error fields to userspace without exposing kernel pointers; dmesg paths use `pr_err_ratelimited` to deny dmesg-spam DoS.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `snp_guest_dev`, request/response message frames, and per-ioctl staging buffers; `snp_report_req`, `snp_report_resp`, `snp_derived_key_req`, `snp_derived_key_resp`, and `snp_ext_report_req` copies cross only whitelisted regions.
- **PAX_KERNEXEC** — sev-guest text + `snp_guest_fops` + `tsm_report_ops` live in W^X regions; ioctl dispatch and AEAD helpers in `__ro_after_init`.
- **PAX_RANDKSTACK** — kernel-stack offset randomized on each `snp_guest_ioctl` entry and on each guest-request submission to break leaks of REPORTDATA staging frames.
- **PAX_REFCOUNT** — saturating `refcount_t` on `snp_guest_dev` and TSM-report provider; overflow trap defeats refcount-confusion UAFs.
- **PAX_MEMORY_SANITIZE** — VMPCK key copy, derived-key output, attestation-report payload, and shared cert-blob mapping are zero-on-free; staging frames `memzero_explicit` after each request to prevent residual REPORTDATA scraping.
- **PAX_UDEREF** — SMAP/PAN enforced on every `copy_from_user`/`copy_to_user` for `snp_*_req` and `snp_*_resp`; reject user pointers outside canonical helpers.
- **PAX_RAP / kCFI** — `snp_guest_fops`, `sev_tsm_report_ops`, and platform-driver ops marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `snp_guest_dev`, VMPCK pointers, and Secrets Page mapping behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict VMPCK-failure, sequence-mismatch, and AEAD-tag-fail banners to CAP_SYSLOG so a host cannot observe replay-failure timing via dmesg.
- **CAP_SYS_ADMIN strict** — all three ioctls require CAP_SYS_ADMIN against the owning user namespace; deny even when `/dev/sev-guest` is mode 0666.
- **VMPCK key MEMORY_SANITIZE** — key material zeroed on disable and on module unload; Secrets Page mapping cleared on probe failure.
- **`snp-report` PAX_USERCOPY** — 4000-byte response buffer copy back to user gated by whitelisted slab + explicit length check; reject any short user buffer.
- **Msg-seq replay counter** — monotonically increasing `msg_seqno`; one-way `snp_disable_vmpck` on AEAD failure permanently locks the VMPCK channel for the session.
- **Derived-key gating** — `SNP_GET_DERIVED_KEY` reject if VMPL/guest-SVN/TCB inputs are outside spec-defined ranges; refuse to mint keys with empty `guest_field_select` (would produce a near-platform-master derivative).
- **Untrusted-host posture** — the host VMM and the GHCB channel are treated as adversarial; all response data is AEAD-verified before being copied to userspace, and any verification miss permanently disables the channel rather than retrying.

Rationale: `sev-guest` is the only kernel surface that turns SNP firmware secrets and platform certificates into evidence a relying party will trust. A replayed report, a leaked VMPCK byte, or a derived key minted with attacker-chosen `guest_field_select` collapses the entire confidential-computing posture for the workload. RAP/kCFI on `snp_guest_fops`, CAP_SYS_ADMIN on ioctl entry, monotonically-increasing AEAD nonces with one-way disable, USERCOPY whitelist on the four UAPI structs, and MEMORY_SANITIZE on VMPCK key bytes plus REPORTDATA staging buffers turn `/dev/sev-guest` from "any local process can mint attestation evidence" into a CAP_SYS_ADMIN-gated, replay-locked, leak-resistant attestation oracle.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ioctl_buf_no_oob` | OOB | `copy_from_user`/`copy_to_user` for `snp_report_req`/`snp_report_resp`/`snp_derived_key_*` strictly bounded to UAPI sizes. |
| `msg_seqno_monotonic` | INVARIANT | `msg_seqno` strictly increasing across `Device::send_request`; never wraps without `snp_disable_vmpck`. |
| `vmpck_disabled_one_way` | ONE_WAY | once `snp_disable_vmpck` runs, no further request succeeds for that VMPCK in the session. |
| `request_mutex_no_uaf` | UAF | `Arc<Device>` outlives all open files; ioctl holds device reference for the full AEAD round-trip. |

### Layer 2: TLA+

`models/sev_guest/vmpck_replay.tla` (parent-declared): model the host as a Dolev-Yao adversary that can drop/duplicate/reorder GHCB messages; prove that any non-fresh sequence number transitions the driver to a permanently disabled state and that no derived key is returned on a replayed response.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `Device::get_report` post: `resp.user_data == req.user_data` (REPORTDATA preserved into report body) | `Device::get_report` |
| `Device::get_derived_key` post: returned 32 B key is a deterministic function of (`root_key_select`, `guest_field_select`, `vmpl`, `guest_svn`, `tcb_version`) | `Device::get_derived_key` |
| `Device::send_request` post: response AEAD tag verified with VMPCK before any byte reaches userspace | `Device::send_request` |

### Layer 4: Verus/Creusot functional

End-to-end: relying party supplies REPORTDATA nonce N; `SNP_GET_REPORT` returns a report whose REPORTDATA field equals N and whose signature chains to the AMD root via the cert blob returned by `SNP_GET_EXT_REPORT`. Encoded as a Verus invariant tying ioctl output to firmware-signed output.
