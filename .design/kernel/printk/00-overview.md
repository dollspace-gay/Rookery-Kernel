# Tier-2: kernel/printk — printk subsystem (printk + ringbuffer + nbcon + safe + sysctl + dmesg + console framework)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/printk/
  - include/linux/printk.h
  - include/linux/console.h
-->

## Summary

Tier-2 wrapper for the kernel printk + console framework — every `pr_info` / `pr_err` / `dev_warn` / `WARN_ON` / `printk(KERN_*)` call lands here. Components: **printk core** (`printk.c` + `internal.h` + `index.c`: lockless printk via printk_safe re-entry path, console-driver registration, log-buffer recording, suppress-rate-limit, `printk_index` for per-message-source-location indexing), **ringbuffer** (`printk_ringbuffer.c` + `printk_ringbuffer.h` + `printk_ringbuffer_kunit_test.c`: lockless multi-producer single-consumer ring used as the kmsg buffer), **nbcon** (`nbcon.c`: Nice-Better-CONsole — newer atomic console-driver framework that allows printing without holding console_lock for emergency/realtime cases), **printk_safe** (`printk_safe.c`: safe re-entry via per-cpu deferred buffer when normal printk path can't run), **sysctl** (`sysctl.c`: `/proc/sys/kernel/{printk, printk_ratelimit, printk_ratelimit_burst, printk_devkmsg, dmesg_restrict, kptr_restrict, ...}`), **braille** (`braille.c` + `braille.h`: braille-display console via Linux braille subsystem — accessibility), **console_cmdline** (`console_cmdline.h`: cmdline `console=...` parsing).

`/proc/kmsg` + `/dev/kmsg` chardevs + `dmesg(1)` userspace + journald `_KERNEL_*` consumer all read from the printk ringbuffer.

## Compatibility contract — outline

- `/dev/kmsg` chardev: read returns one ringbuffer record per read, format `<level>,<seq>,<ts_us>,<flags>;<text>` byte-identical (systemd-journald + dmesg consume).
- `/proc/kmsg` legacy chardev preserved for klogd compat.
- `klogctl(2)` syscall (#103) byte-identical for legacy.
- `/proc/sys/kernel/printk` (4-tuple console_loglevel default_message_loglevel minimum_console_loglevel default_console_loglevel) byte-identical.
- `/proc/sys/kernel/{printk_ratelimit, printk_ratelimit_burst, printk_devkmsg, printk_delay, dmesg_restrict, kptr_restrict, printk_index_show_seq}` byte-identical.
- Cmdline: `loglevel=`, `quiet`, `debug`, `console=`, `earlycon=`, `keep_bootcon`, `printk.devkmsg=`, `printk.always_kmsg_dump`, `printk.time` parsed identically.
- Console-driver registration API source-compat for in-tree consumers (vt console, serial console, hvc, fb console).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/printk/printk-core.md` | `printk.c` + `internal.h` + `index.c`: printk dispatch + console-driver framework + index |
| `kernel/printk/ringbuffer.md` | `printk_ringbuffer.c`: lockless MPSC ringbuffer |
| `kernel/printk/nbcon.md` | `nbcon.c`: Nice-Better-CONsole atomic-printing framework |
| `kernel/printk/safe.md` | `printk_safe.c`: re-entry-safe deferred path |
| `kernel/printk/sysctl.md` | `sysctl.c`: `/proc/sys/kernel/printk*` knobs |
| `kernel/printk/braille.md` | `braille.c`: braille-console accessibility |

## Compatibility outline

- REQ-O1: `/dev/kmsg` + `/proc/kmsg` + `klogctl(2)` wire format byte-identical (systemd-journald + dmesg + klogd consume unchanged).
- REQ-O2: `/proc/sys/kernel/printk*` knobs byte-identical.
- REQ-O3: Cmdline `loglevel=`, `quiet`, `debug`, `console=`, `earlycon=` parsed identically.
- REQ-O4: console-driver registration API source-compat.
- REQ-O5: TLA+ models (lockless MPSC ringbuffer producer-consumer ordering; nbcon atomic-print critical-section; printk_safe re-entry deferred-flush correctness).
- REQ-O6: Hardening: `dmesg_restrict=1` default-on; `kptr_restrict=2` default; printk-rate-limit + printk-from-userspace (`/dev/kmsg` write) requires CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-O1: `dmesg` produces output byte-identical to upstream baseline boot dmesg.
- [ ] AC-O2: `journalctl -k` reads kernel messages via `/dev/kmsg` correctly.
- [ ] AC-O3: `printf 'hello' > /dev/kmsg` (root) appears in dmesg with KERN_INFO prefix.
- [ ] AC-O4: `cat /proc/sys/kernel/printk` returns matching default tuple.
- [ ] AC-O5: kselftest `tools/testing/selftests/printk/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/printk/ringbuffer_mpsc.tla` | `kernel/printk/ringbuffer.md` (proves: lockless MPSC ringbuffer producer + concurrent reader; per-record state machine — RESERVED → COMMITTED → CONSUMED; reader never sees uncommitted record; producer reservation never overwrites unconsumed record) |
| `models/printk/nbcon_critical.tla` | `kernel/printk/nbcon.md` (proves: nbcon atomic-print critical-section + concurrent fallback to legacy console_lock-driven path; emergency printing via nbcon never deadlocks even if console_lock is held by another CPU) |
| `models/printk/safe_reentry.tla` | `kernel/printk/safe.md` (proves: printk_safe per-cpu deferred buffer + flush; re-entry from NMI/IRQ never produces lost message; ordering preserved at flush time) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-console refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-console `console` struct `static const` | § Mandatory |
| **SIZE_OVERFLOW** | record-len + ringbuf-page-count arithmetic checked | § Mandatory |
| **STRICT_KERNEL_RWX** | console-driver dispatch table + nbcon ops vtable read-only-after-init | § Mandatory |

Printk-specific reinforcement: **`dmesg_restrict=1`** default-on (defense against unprivileged process reading kmsg which leaks pointer values); **`kptr_restrict=2`** default-on (`%pK` always sanitized regardless of CAP_SYSLOG); `/dev/kmsg` write requires CAP_SYS_ADMIN; printk rate-limit applied per-source-location to prevent log-flood; `printk_index` (CONFIG_PRINTK_INDEX) audited at boot for unique message-IDs.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- 32-bit-only paths
