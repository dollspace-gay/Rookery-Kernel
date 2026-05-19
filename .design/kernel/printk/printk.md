# Tier-3: kernel/printk/printk.c — kernel logging (printk + ring-buffer + console driver dispatch)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/printk/00-overview.md
upstream-paths:
  - kernel/printk/printk.c
  - kernel/printk/printk_ringbuffer.c
  - kernel/printk/printk_ringbuffer.h
  - kernel/printk/printk_safe.c
  - kernel/printk/internal.h
  - kernel/printk/syslog.c
  - kernel/printk/braille.c
  - include/linux/printk.h
-->

## Summary

`printk` is the kernel's universal logging primitive. Every kernel message ultimately flows through `printk` → printk-ringbuffer → console drivers (tty/ttyS/efifb/serial-uart/netconsole). The ringbuffer (printk_ringbuffer.c) is a lockless multi-producer single-consumer descriptor + data ring designed to never lose a message even from NMI/IRQ context (printk-safe). Per-message metadata: severity (KERN_EMERG..KERN_DEBUG), facility, sequence, timestamp, caller-id, dev_t. Console drivers (registered via `register_console`) drain the ringbuffer and emit to physical output. printk_safe handles deadlock-prone re-entry (printk-from-printk-callback or NMI/IRQ stomp). Critical primitive — every panic, oops, BUG, WARN flows through printk.

This Tier-3 covers `printk.c` (~5184) + `printk_ringbuffer.c` (~2418) + `printk_safe.c` + `internal.h` + `syslog.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `printk(fmt, ...)` / `printk_emit(...)` | core entry | `kernel::printk::Printk::printk` |
| `vprintk_emit(facility, level, dev_info, fmt, args)` | va_list variant | `Printk::vprintk_emit` |
| `vprintk_default(...)` | non-NMI default | `Printk::vprintk_default` |
| `vprintk_safe(...)` | NMI/IRQ-safe variant | `Printk::vprintk_safe` |
| `console_lock()` / `console_unlock()` | per-console critical-section | `Console::lock` / `_unlock` |
| `console_trylock()` | non-blocking | `Console::try_lock` |
| `register_console(con)` / `unregister_console(con)` | per-driver registration | `Console::register` / `_unregister` |
| `console_flush_all(...)` | drain ringbuffer to consoles | `Console::flush_all` |
| `console_emit_next_record(...)` | per-console record emit | `Console::emit_next_record` |
| `prb_alloc(...)` / `prb_init(...)` (ringbuffer) | per-rb init | `Prb::init` |
| `prb_reserve(e, rb, &r)` | reserve ring slot | `Prb::reserve` |
| `prb_commit(e)` | commit reserved slot | `Prb::commit` |
| `prb_read_valid(...)` | read next record | `Prb::read_valid` |
| `prb_first_seq(rb)` / `prb_next_seq(rb)` | seq query | `Prb::first_seq` / `_next_seq` |
| `prb_record_text_space(...)` | per-record text-buffer | `Prb::record_text_space` |
| `add_preferred_console(...)` | preferred console at boot | `Console::add_preferred` |
| `setup_log_buf(early)` | grow ringbuffer at boot | `Printk::setup_log_buf` |
| `printk_safe_enter()` / `_exit()` | per-CPU re-entry guard | `Printk::safe_enter` / `_exit` |
| `do_syslog(type, buf, len, source)` | syscall(2) syslog dispatch | `Syslog::do_syslog` |
| `klogd` (userspace) | drain via /dev/kmsg | (userspace; out-of-scope) |
| `printk_ratelimited(...)` | rate-limit wrapper | `Printk::ratelimited` |
| `panic_print_sys_info(...)` | panic-time emit | `Printk::panic_print` |
| `dump_stack()` | stack-trace via printk | (covered separately) |

## Compatibility contract

REQ-1: `printk` API:
- `printk(fmt, ...)`: KERN_DEFAULT severity (extracted from KERN_* prefix in fmt).
- `pr_emerg/_alert/_crit/_err/_warning/_notice/_info/_debug`: per-severity wrappers.
- `dev_*` variants prepend driver+device prefix.

REQ-2: Severity levels (KERN_EMERG=0 .. KERN_DEBUG=7):
- Each `KERN_<LEVEL>` is a `"\001N"` 2-char prefix in fmt.
- `vprintk_emit` parses prefix and stores level in record.

REQ-3: printk_ringbuffer layout:
- Fixed-size descriptor ring + variable-size data ring (separate).
- Per-descriptor: state (UNUSED, RESERVED, COMMITTED, FINALIZED, REUSE) + seq + text_buf_offset + text_buf_size + caller_id + facility + level + flags + ts_nsec + dev_info.
- Lockless via atomic state-transitions + sequence numbers.
- Multi-producer-single-consumer for descriptor ring; multi-producer for data ring (with separate state).

REQ-4: prb_reserve flow:
1. Acquire descriptor slot via atomic state-CAS UNUSED → RESERVED.
2. Allocate text-buffer space in data-ring (atomic-CAS head_pos += text_size).
3. Populate metadata.
4. Return reserved entry.

REQ-5: prb_commit:
- State CAS RESERVED → COMMITTED.
- Subsequent FINALIZED transition by reader-side after read.

REQ-6: console_lock semantics:
- Mutex protecting per-console iteration during console_unlock flush.
- Preempt-disabled during emit to prevent re-entry.

REQ-7: Console driver hooks:
- `console.write(co, buf, len)`: emit string to console.
- `console.read(...)`: optional reverse direction.
- `console.match`: per-driver fmt-match for parsing /proc/cmdline `console=` arg.
- Per-driver `console.flags`: CON_PRINTBUFFER (flush ring on register), CON_BOOT (boot-only), CON_ENABLED, etc.

REQ-8: console_unlock flow:
1. While ring has unread records:
   - Read record from ring at console.seq.
   - For each registered console:
     - Call console.write(co, record.text, record.text_len).
   - Advance console.seq.

REQ-9: printk_safe per-CPU re-entry:
- Per-CPU `printk_recurse_count` + per-CPU `printk_safe_seq_buf` (small fallback buffer).
- If recurse > 0 (printk-in-printk-callback): use safe_seq_buf instead of main ringbuffer; defer flush.
- Defense against deadlock from console.write calling printk.

REQ-10: NMI handling:
- NMI may interrupt printk; per-CPU NMI-safe path uses safe_seq_buf.
- After NMI exits: defer to next non-NMI flush.

REQ-11: /dev/kmsg interface:
- Userspace reads via /dev/kmsg or syslog(2) syscall.
- Per-fd seq cursor; reads consume records.

REQ-12: dmesg --follow:
- Userspace polls /dev/kmsg with EPOLL_IN; new records → epoll-readable.

REQ-13: Per-CPU `printk_buffers` (8KiB each):
- Used during printk-from-NMI / printk-recurse to avoid touching main ringbuffer.
- Drained on next safe printk.

REQ-14: Boot-time setup:
- Default ringbuffer size 64KB at boot; grown via cmdline `log_buf_len=2M`.
- Per-message timestamp via `local_clock()`.

REQ-15: console drivers are registered at various boot stages:
- Early-printk: registered very early (e.g., 8250 UART).
- Boot-time: SBI / PCI consoles.
- Late: framebuffer consoles.

## Acceptance Criteria

- [ ] AC-1: Basic printk: `pr_info("hello")` appears in dmesg.
- [ ] AC-2: Severity gating: `pr_debug` not visible by default unless DEBUG set.
- [ ] AC-3: Multi-CPU concurrent printk: 32 CPUs × 1000 prints; ringbuffer captures all (no drops barring overflow).
- [ ] AC-4: NMI-safe printk: simulated NMI calls printk; message captured (defer to next flush).
- [ ] AC-5: console_lock contention: long printk while other CPU also printk; both eventually drain.
- [ ] AC-6: Ringbuffer overflow: messages exceed buffer size; oldest evicted; newest preserved.
- [ ] AC-7: /dev/kmsg follow: dmesg --follow shows live messages.
- [ ] AC-8: Panic flush: `BUG()` causes printk to flush before reset.
- [ ] AC-9: Per-CPU printk_safe: printk inside console.write callback handled (no deadlock).
- [ ] AC-10: Rate-limit: printk_ratelimited drops repeats after N within burst window.

## Architecture

`Prb` (printk_ringbuffer):

```
struct Prb {
  desc_ring: AtomicState<DescRing>,           // descriptor ring atomic ops
  text_data_ring: AtomicState<DataRing>,
  fail: AtomicU64,                             // count of failed reservations
}

struct PrbReservedEntry {
  rb: KArc<Prb>,
  irq_flags: u64,
  id: u64,                                     // descriptor index
  text_size: u32,
}
```

`Console` per-driver:

```
struct Console {
  name: KStr,                                  // e.g., "ttyS0"
  write: ConsoleWriteFn,
  read: Option<ConsoleReadFn>,
  device: Option<ConsoleDeviceFn>,
  unblank: Option<ConsoleUnblankFn>,
  setup: Option<ConsoleSetupFn>,
  match_: Option<ConsoleMatchFn>,
  flags: u32,                                   // CON_BOOT | CON_PRINTBUFFER | CON_ENABLED | ...
  index: i32,
  seq: AtomicU64,                               // seq cursor
  level: i32,
  next: AtomicPtr<Console>,
}
```

`Printk::vprintk_emit(facility, level, dev_info, fmt, args)`:
1. cpu := smp_processor_id().
2. printk_safe_enter; ctx_inc := per_cpu_inc(&printk_recurse_count).
3. If ctx_inc > MAX_PRINTK_RECURSE: drop + return.
4. Format message into buffer.
5. prb_reserve(&entry, rb, text_size).
6. Populate entry.text + entry.text_len.
7. prb_commit(&entry).
8. printk_safe_exit.
9. wake_up_klogd; preempt_disable; console_trylock + flush; preempt_enable.

`Console::flush_all`:
1. Hold console_lock.
2. For each con in console_list:
   - For seq from con.seq..prb_next_seq(rb):
     - prb_read_valid(seq, &record).
     - con.write(con, record.text, record.text_len).
   - con.seq = updated.
3. Release console_lock.

`Prb::reserve(rb, text_size)`:
1. Atomic-CAS allocate descriptor: state UNUSED → RESERVED (find next-unused via free-list / round-robin).
2. Atomic-add allocate text-buffer in data-ring (head += text_size).
3. Populate descriptor: data_blk_lpos = ..., text_size = text_size.
4. Return entry.

`Prb::commit(entry)`:
1. State CAS RESERVED → COMMITTED.

`Printk::safe_enter`:
1. preempt_disable.
2. per_cpu_inc(&printk_recurse_count).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `desc_ring_no_oob` | OOB | per-descriptor index < descriptor-ring-size; lockless allocation bounded. |
| `data_ring_lpos_advances` | INVARIANT | text-data ring head advances monotonic across allocations. |
| `state_transitions_valid` | INVARIANT | per-descriptor state transitions follow UNUSED → RESERVED → COMMITTED → FINALIZED → REUSE. |
| `recurse_count_bounded` | INVARIANT | per-CPU printk_recurse_count ≤ MAX_PRINTK_RECURSE; defense against runaway recursion. |

### Layer 2: TLA+

`kernel/printk/desc_ring_state.tla`:
- Per-descriptor state ∈ {Unused, Reserved, Committed, Finalized, Reuse}.
- Properties:
  - `safety_no_double_reserve` — per-descriptor at most one Reserved at a time.
  - `safety_committed_after_reserved` — Committed transition only from Reserved.
  - `liveness_committed_eventually_finalized` — assuming readers run, Committed eventually Finalized.

`kernel/printk/console_emit.tla`:
- Per-console state: emit-position cursor.
- Properties:
  - `safety_emit_in_seq_order` — per-console emits records in seq order.
  - `liveness_eventually_drained` — assuming console_unlock runs, every Committed record eventually emitted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Prb::reserve` post: descriptor RESERVED; data-buffer alloc'd; text_size matches | `Prb::reserve` |
| `Prb::commit` post: descriptor COMMITTED; visible to readers | `Prb::commit` |
| `Console::flush_all` post: per-console seq advanced; all records up to prb_next_seq emitted | `Console::flush_all` |
| Per-CPU printk_recurse_count balanced (enter/exit paired) | invariants on safe_enter/_exit |
| Per-record text_buf_offset + text_buf_size within data-ring bounds | `Prb::reserve` |

### Layer 4: Verus/Creusot functional

`printk(fmt) → record committed → console.write called with formatted text` semantic equivalence: per-printk call eventually delivers formatted text to all registered consoles (assuming non-overflow + console_unlock runs).

## Hardening

(Inherits row-1 features from `kernel/printk/00-overview.md` § Hardening.)

printk-specific reinforcement:

- **Per-CPU printk_recurse_count + safe_seq_buf** — defense against printk-in-printk-callback deadlock.
- **NMI-safe path** — defense against NMI in middle of printk causing ringbuffer corruption.
- **Lockless ringbuffer descriptor states** — defense against concurrent printk causing torn-record visibility.
- **Per-record state-CAS** — defense against double-reserve or premature read.
- **MAX_PRINTK_RECURSE cap** — defense against runaway recursion stack overflow.
- **console_lock mutex** — defense against concurrent console.write from multiple CPUs.
- **Per-console seq cursor** — defense against missed records or duplicate emit.
- **printk_ratelimited burst window** — defense against attacker / runaway-driver flooding dmesg.
- **/dev/kmsg permission gated** — defense against unauthorized log read.
- **panic-flush honored** — defense against losing pre-panic context.
- **Message-size capped** at LOG_LINE_MAX — defense against pathologically-long single message.
- **dev_info validation** — defense against attacker-supplied driver-name embedding control characters.
- **/proc/sys/kernel/printk per-level gating** — defense against console-flood from low-level messages in production.
- **Boot-time log_buf_len capped** — defense against attacker requesting GBs of ringbuffer at boot.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **GRKERNSEC_DMESG strict** — unprivileged `dmesg` and unprivileged `/dev/kmsg` reads return -EPERM; only CAP_SYSLOG holders see the buffer.
- **RingBuf bounded by log_buf_len at compile/boot** — `log_buf_len=` cmdline value clamped to a sane maximum so attacker-influenced boot args can't claim GBs.
- **printk_ratelimit shared with audit** — per-source rate-limit state under PAX_REFCOUNT saturating semantics; cannot wrap to "send infinitely."
- **/dev/kmsg permissions 0600 root** — owner write/read only by default; udev does not relax for unprivileged users.
- **kptr_restrict honored by `%pK`** — `%pK` formatter sanitizes pointers for unprivileged readers when `kptr_restrict >= 1`, and unconditionally when `>= 2`.
- **Record CRC / sequence sanity** — printk_ringbuffer descriptor state machine forbids out-of-order finalize; corrupt sequence drops the record rather than emitting a malformed entry.
- **PAX_USERCOPY on `copy_to_user(log_buf...)`** — `/proc/kmsg` / `/dev/kmsg` reads bounded by the user buffer length and the ring-buffer record length.
- **printk's vprintk_emit indirect calls under PAX_RAP** — `console->write` invoked from `console_unlock` carries kCFI signature.

Per-doc rationale: printk is both a high-frequency emit path and the primary log audit channel. An attacker who can read raw dmesg (kallsyms leaks), forge entries (audit confusion), or flip kptr_restrict from userns gets a fully passive way to defeat KASLR. GRKERNSEC_DMESG, `kptr_restrict>=2`, PAX_USERCOPY-bounded reads, and PAX_RAP on the console emit path collectively close the dmesg-as-oracle threat model.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- /dev/kmsg userspace interface (covered separately if needed)
- klogd/journald (userspace concern)
- ftrace ring-buffer (different subsystem; covered in `kernel/trace/ring_buffer.md` future Tier-3)
- panic + oops machinery (covered in `kernel/panic.md` future Tier-3)
- Per-driver console.write implementations
- Implementation code
