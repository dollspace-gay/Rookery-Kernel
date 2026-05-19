# Tier-3: drivers/virt/coco/tdx-guest/tdx-guest.c — Intel TDX guest driver (/dev/tdx-guest, TDREPORT0, GetQuote, RTMR extend)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/virt/coco/tdx-guest/tdx-guest.c
  - drivers/virt/coco/tdx-guest/Kconfig
  - drivers/virt/coco/tdx-guest/Makefile
  - include/uapi/linux/tdx-guest.h
-->

## Summary

`tdx-guest` is the in-kernel attestation and measurement surface for an Intel TDX confidential guest (a TD — Trust Domain). It registers a misc-char device `/dev/tdx-guest` whose only ioctl, `TDX_CMD_GET_REPORT0`, calls `TDG.MR.REPORT` via the TDCALL instruction inside the TD and returns a 1024-byte TDREPORT0 keyed to a 64-byte user REPORTDATA nonce. Beyond the legacy ioctl, the driver registers a TSM (Trusted Security Manager) `tsm_report_ops` provider that drives a full attestation flow: `TDG.MR.REPORT` to mint a TDREPORT, then `TDG.VP.VMCALL<GetQuote>` (GHCI tdvmcall) to ask the untrusted host's Quote Generation Service for a signed quote — the resulting blob is returned through `/sys/kernel/config/tsm/report/<name>/outblob`. The driver also exposes the four runtime measurement registers RTMR0..RTMR3 as a `tsm_measurements` provider: each RTMR can be read (it is just a tail of TDREPORT) and extended via `TDG.MR.RTMR.EXTEND` (`tdx_do_extend`) under CAP_SYS_RAWIO. CMR0..CMR1 are static and read-only. The driver assumes the host VMM and the VMCALL transport are adversarial: TDREPORT is signed by the TDX module, RTMR extends are unforgeable hash chains, and the GetQuote shared-buffer page is private to a single in-flight quote.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct miscdevice tdx_misc_dev` | `/dev/tdx-guest` registration | `drivers::tdx_guest::Device` |
| `tdx_guest_ioctl(file, cmd, arg)` | dispatch (only `TDX_CMD_GET_REPORT0`) | `Device::ioctl` |
| `tdx_get_report0(req)` | drive `tdx_do_report` from user `tdx_report_req` | `Device::get_report0` |
| `tdx_do_report(data, tdreport)` | invoke TDCALL[TDG.MR.REPORT] with REPORTDATA | `Device::do_report` |
| `tdx_do_extend(mr_ind, data)` | invoke TDCALL[TDG.MR.RTMR.EXTEND] | `Device::do_extend` |
| `tdx_report_new(report, data)` | tsm_report_ops backend (legacy mutex path) | `Device::tsm_report_new` |
| `tdx_report_new_locked(report, data)` | GetQuote with shared buffer + wait_for_quote_completion | `Device::tsm_report_new_locked` |
| `wait_for_quote_completion(quote_buf, timeout)` | poll `GET_QUOTE_IN_FLIGHT` → success | `Device::wait_for_quote_completion` |
| `tdx_measurements` (`tsm_measurements`) | RTMR0..RTMR3 + CMR registration | `Device::measurements` |
| `tdx_mr_refresh(tm)` / `tdx_mr_extend(tm, mr, hash)` | refresh measurements / extend RTMR | `Device::mr_refresh` / `Device::mr_extend` |
| `getquote_timeout` modparam (default 30 s) | bound GetQuote wait | `Device::getquote_timeout` |
| `tdx_guest_ids` `x86_cpu_id` table | bind only when `X86_FEATURE_TDX_GUEST` is set | `Subsystem::cpu_match` |
| `TDX_CMD_GET_REPORT0 _IOWR('T', 1, struct tdx_report_req)` (uapi) | sole ioctl number | `Uapi::GET_REPORT0` |
| `TDX_REPORTDATA_LEN = 64` / `TDX_REPORT_LEN = 1024` (uapi) | sizes | `Uapi::*` |

## Compatibility contract

REQ-1: Module loads only on a TDX guest CPU (`X86_FEATURE_TDX_GUEST`); otherwise `misc_register` is never called.

REQ-2: Register `/dev/tdx-guest` as a `miscdevice` with `MISC_DYNAMIC_MINOR`, fops `tdx_guest_fops` exposing only `unlocked_ioctl`.

REQ-3: `TDX_CMD_GET_REPORT0` copies a 64-byte `reportdata` from user, calls `TDG.MR.REPORT` via TDCALL, and copies a 1024-byte TDREPORT back into the user buffer.

REQ-4: TDREPORT contains the REPORTDATA verbatim plus the TD measurement quad (MRTD, RTMR0..RTMR3) plus the TDX module signature; the driver does not alter or re-sign it.

REQ-5: TSM provider path: `tsm_report_new` builds a TDREPORT, then issues `TDG.VP.VMCALL<GetQuote>` (GHCI tdvmcall, leaf 0x10002) using a `tdx_quote_buf` page shared with the host; the host's QGS fills in a signed quote and clears `GET_QUOTE_IN_FLIGHT` (0xffffffffffffffff). Driver returns `outblob` to configfs only after `status == GET_QUOTE_SUCCESS (0)`.

REQ-6: `GET_QUOTE_BUF_SIZE == SZ_128K`; `TDX_QUOTE_MAX_LEN == GET_QUOTE_BUF_SIZE - sizeof(struct tdx_quote_buf)`; the driver rejects any returned `out_len` that exceeds this bound.

REQ-7: GetQuote wait is bounded by `getquote_timeout` seconds (default 30); on timeout the GetQuote call is treated as failed and the shared page is reclaimed.

REQ-8: RTMR registration exposes RTMR0..RTMR3 via `tsm_measurements`; CMR registers are read-only. `tdx_mr_extend` for `RTMR0` is refused (kernel/firmware-owned), `RTMR1..RTMR3` accept SHA-384 extends.

REQ-9: RTMR extend payload is exactly 48 bytes (SHA-384 digest); `tdx_do_extend` rejects any other length.

REQ-10: Misc registration carries attribute group `tdx_attr_groups` (populated by `tdx_mr_init`) so `/sys/class/misc/tdx-guest/measurements/...` exposes per-RTMR readouts.

REQ-11: The driver never trusts host-returned data without TDX-module signature validation (TDREPORT is signed; quote is validated by relying-party with vendor key).

## Acceptance Criteria

- [ ] AC-1: `modprobe tdx-guest` on a TDX-enabled guest creates `/dev/tdx-guest` with mode 0600 root:root.
- [ ] AC-2: `tdxctl report --nonce=<64 B>` returns a 1024-byte TDREPORT whose REPORTDATA matches the nonce.
- [ ] AC-3: Configfs flow `mkdir /sys/kernel/config/tsm/report/X; echo nonce > X/inblob; cat X/outblob` returns a signed TDX quote whose embedded REPORTDATA matches the nonce.
- [ ] AC-4: GetQuote timeout test: with QGS stalled, `cat outblob` returns `-EIO` after `getquote_timeout` seconds; shared page is freed.
- [ ] AC-5: RTMR extend test: `echo "<48-byte SHA-384>" > /sys/class/misc/tdx-guest/measurements/rtmr1/extend` returns success; subsequent `cat rtmr1/digest` shows `H_new = SHA384(H_old || input)`.
- [ ] AC-6: RTMR0 extend is rejected with `-EPERM`.
- [ ] AC-7: kselftest `tools/testing/selftests/tdx/tdx_guest_test` REPORTDATA-roundtrip case passes.
- [ ] AC-8: Concurrent ioctls on `/dev/tdx-guest` from multiple processes serialize on the GetQuote shared buffer without leaking another process's nonce into a returned report.

## Architecture

`Device` in `drivers::tdx_guest::Device`:

```
struct Device {
  misc: MiscDevice,                 // /dev/tdx-guest
  measurements: TsmMeasurements,    // RTMR0..3 + CMR registration
  tsm_ops: TsmReportOps,            // configfs report provider
  quote_mutex: Mutex<()>,           // single in-flight GetQuote
  quote_buf: Option<NonNull<u8>>,   // shared (un-private) 128 KB page
  getquote_timeout: u32,            // seconds, modparam, default 30
}
```

`Device::get_report0(req)`:
1. Copy 64 B `reportdata` from `req->reportdata` into a kernel `sockptr`.
2. Issue TDCALL[TDG.MR.REPORT] (`tdx_do_report`) — TDX module places signed 1024 B TDREPORT into the destination sockptr.
3. Copy TDREPORT back to `req->tdreport`.
4. Return 0 or `-EIO` on TDCALL failure.

`Device::tsm_report_new_locked(report, data)`:
1. Build a TDREPORT using `inblob` as REPORTDATA (`tdx_do_report`).
2. Allocate a 128 KB shared (un-private, host-accessible) page; construct `struct tdx_quote_buf { ver=GET_QUOTE_CMD_VER, status=GET_QUOTE_IN_FLIGHT, in_len=sizeof(TDREPORT), out_len=0, data[TDREPORT||...] }`.
3. Issue `TDG.VP.VMCALL<GetQuote>` (GHCI leaf 0x10002) with the shared page's GPA.
4. `wait_for_quote_completion`: read `quote_buf->status` until ≠ `GET_QUOTE_IN_FLIGHT` or timeout elapses (per `getquote_timeout`).
5. If `status == GET_QUOTE_SUCCESS && out_len <= TDX_QUOTE_MAX_LEN`, hand `(data, out_len)` to TSM as `outblob`.
6. Reclaim shared page (make it private again, zero it).

`Device::do_extend(mr_ind, data)`:
1. Validate `mr_ind` is RTMR1, RTMR2, or RTMR3 (RTMR0 refused).
2. Validate `data` length is 48 bytes (SHA-384).
3. Issue TDCALL[TDG.MR.RTMR.EXTEND]; TDX module computes new RTMR = SHA384(old_RTMR || data) and stores it inside the TDCS.
4. Bump in-kernel cached digest mirror.

`Device::do_report(data, tdreport)` (low-level): wraps `__tdx_module_call` with leaf TDG.MR.REPORT, register inputs being GPAs of the 64 B REPORTDATA and 1024 B TDREPORT destination.

## Hardening

- `TDX_CMD_GET_REPORT0` requires `CAP_SYS_ADMIN` against the user namespace; the device node is 0600 root:root by default but the cap check defends against misconfigured permissions.
- `RTMR.EXTEND` paths require `CAP_SYS_RAWIO`; RTMR0 is permanently refused at the driver layer.
- GetQuote shared page is single-use per call and zero-wiped before being reclaimed as private; concurrent GetQuote requests serialize on `quote_mutex` so two callers cannot race into one shared page.
- GetQuote wait is bounded by `getquote_timeout` and uses interruptible sleep; a stalled QGS cannot turn into a kernel deadlock.
- `tdvmcall` argument validation: only spec-defined leaves (`GetQuote = 0x10002`) are issued; arguments are passed through helpers that bound the GPA against the shared-memory pool.
- All TDREPORT byte buffers are kzalloc'd and `memzero_explicit` on free; REPORTDATA nonces never live in slabs that can be reallocated.
- Driver refuses to bind on a non-TDX CPU via `x86_match_cpu(tdx_guest_ids)`; no counterfeit `/dev/tdx-guest` is possible on a non-TD VM.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `tdx_report_req` staging, the 1024 B TDREPORT buffer, the 64 B REPORTDATA buffer, and the GetQuote `outblob` staging; all `copy_from_user`/`copy_to_user` cross only whitelisted regions.
- **PAX_KERNEXEC** — `tdx_guest_fops`, `tdx_tsm_ops`, and `tdx_measurements.ops` in W^X text; `tdx_guest_init` populates `tdx_misc_dev` then transitions to `__ro_after_init`.
- **PAX_RANDKSTACK** — kernel-stack offset randomized at each `tdx_guest_ioctl`, `tdx_report_new_locked`, and `tdx_do_extend` entry to break leaks of REPORTDATA/RTMR-extend buffers.
- **PAX_REFCOUNT** — saturating `refcount_t` on `tdx_misc_dev` parent, TSM provider, and measurement provider registrations.
- **PAX_MEMORY_SANITIZE** — TDREPORT staging buffer, GetQuote shared page, and RTMR-extend input zeroed on free; shared page zeroed before being remapped private to prevent residual TDREPORT bytes from being scraped.
- **PAX_UDEREF** — SMAP/PAN enforced on every `copy_from_user`/`copy_to_user` for `tdx_report_req`; reject user pointers outside canonical helpers.
- **PAX_RAP / kCFI** — `tdx_guest_fops`, `tdx_tsm_ops`, and per-RTMR `tsm_measurement_register.ops` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `tdx_misc_dev`, `tdx_measurements`, and the shared-page mapping behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict GetQuote-timeout, TDCALL-fail, and RTMR-extend-fail banners to CAP_SYSLOG so a host VMM cannot observe attestation timing via dmesg.
- **CAP_SYS_ADMIN strict** — `TDX_CMD_GET_REPORT0` and TSM configfs `outblob` reads require CAP_SYS_ADMIN in the owning user namespace.
- **REPORTDATA PAX_USERCOPY** — 64 B nonce copy in and 1024 B TDREPORT copy out gated by USERCOPY whitelist; reject short user buffers explicitly.
- **tdvmcall arg validation** — only spec-defined GHCI leaves accepted; GetQuote shared GPA validated against the driver's shared-memory pool before being passed to `__tdx_hypercall`.
- **EXTEND-RTMR CAP_SYS_RAWIO** — RTMR1..RTMR3 extends require CAP_SYS_RAWIO; RTMR0 extend permanently refused. Input length strictly 48 bytes (SHA-384) — no truncated digests.
- **Host-driver-untrusted boundary** — every host-returned byte (GetQuote status, out_len, payload) is bounds-checked against `TDX_QUOTE_MAX_LEN` and a timeout watchdog; payload validity is the relying party's job, not the driver's.

Rationale: `tdx-guest` is the boundary between TDX-module-signed evidence and userspace verifiers. A short-bound TDREPORT copy, a forged RTMR extend, or a GetQuote shared page reused while still containing a previous caller's nonce would collapse the attestation guarantee that every TDX deployment depends on. RAP/kCFI on the misc fops, CAP_SYS_ADMIN on the ioctl, CAP_SYS_RAWIO on RTMR extend, USERCOPY whitelist on REPORTDATA/TDREPORT staging, MEMORY_SANITIZE on the GetQuote shared page, and a strict adversarial posture toward host-returned status/length fields turn `/dev/tdx-guest` from "any local process can prod the TDX module" into a measured, replay-safe, untrusted-host-resistant attestation channel.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `reportdata_copy_bounded` | OOB | `copy_from_user` for `tdx_report_req.reportdata` is exactly 64 B; `copy_to_user` for `tdreport` is exactly 1024 B. |
| `getquote_buf_bounded` | OOB | `quote_buf->out_len` strictly ≤ `TDX_QUOTE_MAX_LEN`; reject otherwise. |
| `rtmr_extend_len` | INVARIANT | `tdx_do_extend` accepts only 48-byte SHA-384 inputs. |
| `quote_mutex_no_uaf` | UAF | shared GetQuote page lifetime strictly nested inside `quote_mutex` hold. |

### Layer 2: TLA+

`models/tdx_guest/getquote.tla` (parent-declared): proves that a host that stalls, returns short `out_len`, returns oversized `out_len`, or never clears `GET_QUOTE_IN_FLIGHT` cannot cause the driver to forward unverified bytes to `outblob`; the only attacker-visible outcomes are `EIO` and `ETIMEDOUT`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `Device::get_report0` post: TDREPORT bytes [0..64) (or spec offset) carry REPORTDATA verbatim | `Device::get_report0` |
| `Device::do_extend` post: cached RTMR = SHA384(prev_RTMR \|\| input) | `Device::do_extend` |
| `Device::tsm_report_new_locked` post: `outblob` returned only when `status == GET_QUOTE_SUCCESS` AND `out_len ≤ TDX_QUOTE_MAX_LEN` | `Device::tsm_report_new_locked` |

### Layer 4: Verus/Creusot functional

End-to-end: relying party supplies REPORTDATA N → driver issues TDG.MR.REPORT producing TDREPORT T(N) signed by TDX module → driver issues GetQuote with T(N) → host QGS returns quote Q over T(N) → outblob Q verifiable by relying party against Intel root. Encoded as a Verus invariant tying `inblob` → TDREPORT → `outblob` causality, with the host limited to selecting Q ∈ {valid quote over T(N), failure}.
