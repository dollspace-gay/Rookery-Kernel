# Tier-3: kernel/printk/printk.c (console subset) + nbcon.c — Console driver registration + dispatch

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/printk/printk.md
upstream-paths:
  - kernel/printk/printk.c (console subset; register_console, console_unlock, etc.)
  - kernel/printk/nbcon.c (~2002 lines; new non-blocking console)
  - include/linux/console.h
-->

## Summary

Linux console subsystem dispatches kernel log-records to per-device drivers (serial UART, framebuffer-tty, netconsole, virtio-console, hvc, sclp, etc.). Per-driver registers via `register_console(con)`. Per-record from printk-ring-buffer routed through enabled consoles. Per-`console_unlock` (legacy) or per-nbcon kthread (modern) iterates pending records and dispatches per-console `con->write()`. Per-CON_ flags: CON_PRINTBUFFER (replay buffer on enable), CON_CONSDEV (default console), CON_BOOT (boot-time), CON_BRL (Braille), CON_NBCON (non-blocking). Critical for: kernel-log visibility (boot, panic, runtime), serial debugging.

This Tier-3 covers the console-driver subsystem.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct console` | per-driver console | `Console` |
| `register_console()` | per-driver register | `Console::register` |
| `unregister_console()` | per-driver unregister | `Console::unregister` |
| `console_lock()` / `console_unlock()` | global console-mutex | `Console::lock` / `unlock` |
| `console_trylock()` | non-block lock | `Console::trylock` |
| `console_emit_next_record()` | per-record dispatch | `Console::emit_next_record` |
| `nbcon_acquire()` / `_release()` | per-nbcon non-blocking acquire | `NbCon::acquire` / `release` |
| `nbcon_emit_next_record()` | per-nbcon emit | `NbCon::emit_next_record` |
| `nbcon_kthread_create()` / `_stop()` | per-nbcon-kthread | `NbCon::kthread_*` |
| `console_drivers` | global list | shared |
| `console_force_preferred_locked()` | per-cmdline override | `Console::force_preferred_locked` |
| `add_preferred_console()` | per-`console=` cmdline | `Console::add_preferred` |
| `CON_*` flags | per-feature | UAPI |
| `console_lock_dep_map` | per-lockdep | shared |
| `console_set_on_cmdline` | per-cmdline-set flag | shared |

## Compatibility contract

REQ-1: struct console:
- name: driver name (e.g. "ttyS", "tty", "ttyAMA", "netcon").
- write: per-record output fn.
- read: per-key-input (optional for kdb).
- device: per-tty-driver lookup.
- unblank: per-blank-restore.
- setup: per-init from cmdline.
- exit: per-shutdown.
- match: per-match for cmdline.
- flags: CON_*.
- index: per-device index (e.g. ttyS0 → 0).
- cflag: per-tty mode (e.g. B115200).
- level: per-console log-level.
- next: ListLink.

REQ-2: CON_* flags:
- CON_PRINTBUFFER: replay log-buffer when this console enabled.
- CON_CONSDEV: default-console (/dev/console).
- CON_ENABLED: per-active.
- CON_BOOT: per-boot-time-only (deregistered post-boot).
- CON_ANYTIME: callable from any context (NMI, atomic).
- CON_BRL: Braille (special).
- CON_EXTENDED: per-extended-format printk.
- CON_NBCON: non-blocking console (kthread-driven).

REQ-3: register_console(con):
- /* Validate */
- if !con.write: -EINVAL.
- /* Match cmdline */
- For each `console=` cmdline entry: if matches con.name + con.index: con.flags |= CON_ENABLED + CON_CONSDEV.
- Insert into console_drivers list.
- if CON_PRINTBUFFER: replay log-buf into con.write.

REQ-4: console_unlock():
- /* Iterate pending records */
- while record_pending:
   - Pop record from ring-buffer.
   - For each console in console_drivers:
     - if CON_ENABLED ∧ !CON_NBCON ∧ level-acceptable:
       - con.write(con, record.text, record.text_len).

REQ-5: console_lock / _unlock semantics:
- Mutual exclusion across all consoles.
- May block (sleeping spinlock).
- Per-CONFIG_PREEMPT_RT: rt_mutex.

REQ-6: nbcon (non-blocking console):
- Per-CON_NBCON consoles use kthread (nbcon_kthread).
- Per-record nbcon_emit_next_record invoked by kthread, not under console_lock.
- Per-nbcon_acquire/_release per-console-atomic state.

REQ-7: Per-record dispatch (console_emit_next_record):
- record = pop from printk_ringbuffer.
- Build per-record text with prefix (kernel-timestamp, log-level).
- For each enabled console: write(con, text, len).

REQ-8: Per-cmdline `console=ttyS0,115200n8`:
- Per-add_preferred_console (called pre-register).
- At register: if name + index matches: enable.

REQ-9: Per-/dev/console:
- VFS open of /dev/console → tty_open → /dev/tty0 (text) or active console.

REQ-10: Per-CON_BOOT registration:
- Boot consoles deregistered after permanent register.
- console_remove_boot.

REQ-11: Per-loglevel filter:
- KERN_EMERG (0) ... KERN_DEBUG (7).
- per-console.level: only emit records ≤ this level.

REQ-12: Per-/proc/consoles:
- /proc/consoles: lists registered consoles + flags.

## Acceptance Criteria

- [ ] AC-1: register_console(ttyS0-con) at boot: ttyS0 in console_drivers list.
- [ ] AC-2: cmdline "console=ttyS0,115200": ttyS0 enabled at register.
- [ ] AC-3: CON_PRINTBUFFER: per-prior-log replayed to new console.
- [ ] AC-4: printk("hello"): per-record routed to all enabled consoles.
- [ ] AC-5: console_unlock iterates pending records.
- [ ] AC-6: console_trylock: returns immediately if locked.
- [ ] AC-7: CON_NBCON console: per-kthread emits without console_lock.
- [ ] AC-8: Per-loglevel filter: KERN_DEBUG filtered when level < 7.
- [ ] AC-9: CON_BOOT console: removed after CON_PRINT-real registered.
- [ ] AC-10: /proc/consoles: lists active consoles.
- [ ] AC-11: panic(): per-record emitted on all CON_ANYTIME consoles.

## Architecture

Per-console:

```
struct Console {
  name: [u8; 16],
  write: fn(co: &Console, s: &[u8]),
  read: Option<fn(co: &Console, s: &mut [u8], count: u32) -> i32>,
  device: fn(co: &Console, index: &mut i32) -> *TtyDriver,
  unblank: Option<fn()>,
  setup: Option<fn(co: &Console, options: &[u8]) -> i32>,
  exit: Option<fn(co: &Console) -> i32>,
  match_: Option<fn(co: &Console, name: &str, idx: i32, options: &[u8]) -> i32>,
  flags: i16,                                    // CON_*
  index: i16,
  cflag: i32,
  ispeed: u32,
  ospeed: u32,
  level: u32,
  next: ListLink,
  // For nbcon:
  nbcon_state: AtomicULong,
  nbcon_prev_state: AtomicULong,
  nbcon_kthread: Option<*TaskStruct>,
  nbcon_seq: AtomicU64,
  pbufs: Option<&[u8]>,
}
```

`Console::register(co) -> Result<()>`:
1. if !co.write: return -EINVAL.
2. /* Match cmdline */
3. console_cmdline = find_in_console_cmdline(co.name).
4. if console_cmdline matched:
   - co.flags |= CON_ENABLED | CON_CONSDEV.
5. /* Insert into list */
6. mutex_lock(&console_mutex).
7. list_add(&co.next, &console_drivers).
8. /* Per-CON_PRINTBUFFER: replay */
9. if co.flags & CON_PRINTBUFFER: replay-buffer.
10. mutex_unlock.

`Console::unlock()`:
1. /* Iterate pending records */
2. while record_pending(this_cpu):
   - record = prb_read_valid(printk_rb, seq, &info).
   - for con in console_drivers:
     - if con.flags & CON_ENABLED ∧ !(con.flags & CON_NBCON):
       - if record.level <= con.level:
         - con.write(con, record.text, record.text_len).
   - this_console_seq++.
3. mutex_unlock(&console_mutex).

`NbCon::acquire(co) -> bool`:
1. /* Atomic state transition */
2. old = co.nbcon_state.load().
3. new = transition_to_owner(old, current).
4. return co.nbcon_state.compare_exchange(old, new).

`NbCon::emit_next_record(co)`:
1. if !NbCon::acquire(co): return.
2. record = prb_read_valid(printk_rb, co.nbcon_seq, &info).
3. co.write(co, record.text, record.text_len).
4. co.nbcon_seq++.
5. NbCon::release(co).

`Console::add_preferred(name, index, options, brl_options) -> Result<()>`:
1. /* Insert into console_cmdline array */
2. cmdline = &console_cmdline[console_set_on_cmdline++].
3. cmdline.name = name; cmdline.index = index; cmdline.options = options.

`Console::force_preferred_locked(co) -> Result<()>`:
1. /* Demote current preferred; promote this */
2. for c in console_drivers: c.flags &= ~CON_CONSDEV.
3. co.flags |= CON_CONSDEV.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `write_non_null` | INVARIANT | per-register: co.write != NULL. |
| `flags_valid_combo` | INVARIANT | per-co.flags: subset of CON_* bits. |
| `con_in_list_iff_registered` | INVARIANT | co in console_drivers ⟺ registered. |
| `nbcon_state_atomic` | INVARIANT | per-CON_NBCON: nbcon_state mutated via cmpxchg. |
| `boot_console_removed_after_real` | INVARIANT | per-CON_BOOT: removed when CON_PRINTBUFFER non-CON_BOOT registered. |

### Layer 2: TLA+

`kernel/printk/console.tla`:
- Per-register + per-emit + per-unregister + per-nbcon.
- Properties:
  - `safety_per_record_emit_per_enabled_console` — per-record: each enabled console.write called.
  - `safety_nbcon_atomic_owner` — per-nbcon record-emit: nbcon_state owner exclusive.
  - `liveness_per_pending_eventually_emitted` — per-pending record + console_unlock: emitted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Console::register` post: con in console_drivers; per-cmdline enabled | `Console::register` |
| `Console::unlock` post: per-pending record emitted to enabled non-NBCON consoles | `Console::unlock` |
| `NbCon::emit_next_record` post: con.nbcon_seq++; con.write called | `NbCon::emit_next_record` |
| `Console::add_preferred` post: cmdline-entry inserted | `Console::add_preferred` |

### Layer 4: Verus/Creusot functional

`Per-record from printk-ringbuffer → per-enabled-console.write; per-nbcon kthread dispatches without console_mutex` semantic equivalence: per-Documentation/admin-guide/serial-console.rst.

## Hardening

(Inherits row-1 features from `kernel/printk/00-overview.md` § Hardening.)

Console reinforcement:

- **Per-write null-checked** — defense against per-driver missing fn-ptr.
- **Per-CON_ANYTIME for atomic-context** — defense against per-NMI-printk recursion.
- **Per-CON_BOOT lifetime bounded** — defense against per-stale-boot-console.
- **Per-nbcon atomic owner cmpxchg** — defense against per-emit race.
- **Per-cmdline parsing strict** — defense against per-malformed cmdline-fail.
- **Per-loglevel filter strict** — defense against per-DEBUG flooding.
- **Per-/proc/consoles RCU-walk** — defense against per-walker UAF.
- **Per-CAP_SYS_ADMIN for console-control** — defense against unprivileged console-hijack.
- **Per-panic CON_ANYTIME-only emit** — defense against per-panic-from-non-anytime hang.
- **Per-console_mutex avoids re-entry** — defense against per-recursive-printk deadlock.
- **Per-replay CON_PRINTBUFFER bounded** — defense against per-replay-storm.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/printk/printk.c core (covered in `printk.md` Tier-3)
- kernel/printk/printk_ringbuffer.c (covered separately if expanded)
- drivers/tty/serial/ (covered separately)
- /dev/console fs (covered in `fs/devpts.md` if added)
- Implementation code
