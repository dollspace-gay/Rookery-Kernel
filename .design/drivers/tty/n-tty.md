# Tier-3: drivers/tty/n_tty.c — N_TTY line discipline (cooked mode)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/tty/00-overview.md
upstream-paths:
  - drivers/tty/n_tty.c (~2535 lines)
  - include/linux/tty.h (struct tty_ldisc_ops)
  - include/linux/tty_ldisc.h
  - include/uapi/asm-generic/termbits.h (ICANON / IEXTEN / ISIG / ECHO* / OPOST / IXON / ...)
-->

## Summary

The **N_TTY line discipline** is the default cooked-mode TTY processor: it sits between the hardware/PTY driver and userspace and implements POSIX-termios input/output semantics. Per-ldisc state is `struct n_tty_data` (per-tty private blob attached via `tty.disc_data`): three lock-free SPSC ring buffers (`read_buf[N_TTY_BUF_SIZE]` for line input, `echo_buf[N_TTY_BUF_SIZE]` for pending echo, and a bitmap `read_flags` marking line-delimiter slots), plus heads/tails (`read_head`/`read_tail`/`commit_head`/`canon_head`/`echo_head`/`echo_commit`/`echo_mark`/`echo_tail`/`line_start`/`column`/`canon_column`), per-bitfield flags (`lnext`,`erasing`,`raw`,`real_raw`,`icanon`,`push`,`no_room`), `char_map` (256-bit set of special chars that need special-case handling), and two mutexes (`atomic_read_lock`, `output_lock`). N_TTY exposes `struct tty_ldisc_ops n_tty_ops` with `open`, `close`, `flush_buffer`, `read`, `write`, `ioctl`, `set_termios`, `poll`, `receive_buf`, `receive_buf2`, `write_wakeup`, `lookahead_buf`. ICANON (canonical) mode: line-buffer until newline / EOL / EOL2 / EOF, with ERASE/WERASE/KILL/LNEXT/REPRINT editing characters and ECHO/ECHOE/ECHOK/ECHOKE/ECHOCTL/ECHOPRT/ECHONL feedback. Non-canonical mode: pass-through with MIN/TIME timing. OPOST output processing: ONLCR (LF→CRLF), OCRNL, ONOCR, ONLRET, OLCUC, XTABS (TAB→spaces). ISIG: convert INTR (C-c)/QUIT (C-\\)/SUSP (C-z) chars to SIGINT/SIGQUIT/SIGTSTP sent to the foreground PGID via `__isig` → `kill_pgrp`. IXON flow control: START_CHAR (C-q) / STOP_CHAR (C-s) start/stop output; lookahead path resolves flow-control chars before the main receive loop. PARMRK doubles `\377` and prefixes break/parity errors with `\377\0`. EOF char (C-d) causes a zero-length read at canon boundary. Critical for: terminal applications, shells, login/getty, ssh sessions, and signal delivery from keyboard.

This Tier-3 covers `drivers/tty/n_tty.c` (~2535 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct n_tty_data` | per-tty ldisc state | `NTtyData` |
| `n_tty_ops` (struct tty_ldisc_ops) | per-N_TTY vtable | `NTty::OPS` |
| `n_tty_open()` | per-attach to tty | `NTty::open` |
| `n_tty_close()` | per-detach from tty | `NTty::close` |
| `n_tty_read()` | per-read entry from iterate_tty_read | `NTty::read` |
| `n_tty_write()` | per-write entry from iterate_tty_write | `NTty::write` |
| `n_tty_poll()` | per-EPOLL state | `NTty::poll` |
| `n_tty_ioctl()` | per-TIOCOUTQ / TIOCINQ + helper | `NTty::ioctl` |
| `n_tty_set_termios()` | per-termios reconfigure | `NTty::set_termios` |
| `n_tty_flush_buffer()` | per-discipline flush input | `NTty::flush_buffer` |
| `n_tty_write_wakeup()` | per-driver-empty notify | `NTty::write_wakeup` |
| `n_tty_receive_buf()` / `_buf2()` | per-soft-irq input | `NTty::receive_buf` / `_buf2` |
| `n_tty_receive_buf_common()` | per-shared receive loop | `NTty::receive_buf_common` |
| `__receive_buf()` | per-mode-dispatch | `NTty::receive_buf_inner` |
| `n_tty_receive_buf_real_raw()` | per-real_raw fast path | `NTty::receive_buf_real_raw` |
| `n_tty_receive_buf_raw()` | per-raw path | `NTty::receive_buf_raw` |
| `n_tty_receive_buf_standard()` | per-cooked path | `NTty::receive_buf_standard` |
| `n_tty_receive_buf_closing()` | per-closing-tty path | `NTty::receive_buf_closing` |
| `n_tty_receive_char()` | per-cooked non-special char | `NTty::receive_char` |
| `n_tty_receive_char_special()` | per-cooked special char | `NTty::receive_char_special` |
| `n_tty_receive_char_canon()` | per-ICANON char | `NTty::receive_char_canon` |
| `n_tty_receive_char_closing()` | per-closing char | `NTty::receive_char_closing` |
| `n_tty_receive_char_lnext()` | per-LNEXT-prefixed char | `NTty::receive_char_lnext` |
| `n_tty_receive_char_flagged()` | per-error-flagged byte | `NTty::receive_char_flagged` |
| `n_tty_receive_char_flow_ctrl()` | per-IXON START/STOP | `NTty::receive_char_flow_ctrl` |
| `n_tty_is_char_flow_ctrl()` | per-char-is-flow predicate | `NTty::is_char_flow_ctrl` |
| `n_tty_receive_signal_char()` | per-INTR/QUIT/SUSP | `NTty::receive_signal_char` |
| `n_tty_receive_handle_newline()` | per-canonical newline publish | `NTty::receive_handle_newline` |
| `n_tty_receive_break()` | per-RS232 break | `NTty::receive_break` |
| `n_tty_receive_overrun()` | per-driver-overrun report | `NTty::receive_overrun` |
| `n_tty_receive_parity_error()` | per-parity/frame error | `NTty::receive_parity_error` |
| `n_tty_lookahead_flow_ctrl()` | per-pre-scan IXON | `NTty::lookahead_flow_ctrl` |
| `__isig()` / `isig()` | per-signal kill_pgrp | `NTty::isig_inner` / `isig` |
| `n_tty_packet_mode_flush()` | per-pty packet flush notify | `NTty::packet_mode_flush` |
| `put_tty_queue()` | per-read_buf enqueue | `NTty::put_tty_queue` |
| `add_echo_byte()` | per-echo_buf enqueue | `NTty::add_echo_byte` |
| `read_buf()` / `read_buf_addr()` | per-byte accessor | `NTty::read_buf` / `_addr` |
| `echo_buf()` / `echo_buf_addr()` | per-byte accessor with smp_rmb | `NTty::echo_buf` / `_addr` |
| `read_cnt()` / `chars_in_buffer()` | per-occupancy | `NTty::read_cnt` / `NTty::chars_in_buffer` |
| `input_available_p()` | per-poll predicate | `NTty::input_available` |
| `copy_from_read_buf()` | per-non-canonical drain | `NTty::copy_from_read_buf` |
| `canon_copy_from_read_buf()` | per-canonical drain | `NTty::canon_copy_from_read_buf` |
| `canon_skip_eof()` | per-EOF skip on canon | `NTty::canon_skip_eof` |
| `n_tty_continue_cookie()` | per-iterate_tty_read continuation | `NTty::continue_cookie` |
| `n_tty_wait_for_input()` | per-blocking-wait | `NTty::wait_for_input` |
| `job_control()` | per-SIGTTIN POSIX 7.1.1.4 | `NTty::job_control` |
| `do_output_char()` | per-OPOST single | `NTty::do_output_char` |
| `process_output()` | per-OPOST single locked | `NTty::process_output` |
| `process_output_block()` | per-OPOST block | `NTty::process_output_block` |
| `n_tty_process_echo_ops()` | per-echo op encoder | `NTty::process_echo_ops` |
| `__process_echoes()` / `process_echoes()` / `commit_echoes()` / `flush_echoes()` | per-echo emit | `NTty::*_echoes` |
| `echo_char()` / `echo_char_raw()` | per-char echo | `NTty::echo_char` / `_raw` |
| `echo_move_back_col()` / `echo_set_canon_col()` / `echo_erase_tab()` | per-column tracking | `NTty::echo_*` |
| `eraser()` | per-ERASE/WERASE/KILL editor | `NTty::eraser` |
| `finish_erasing()` | per-ECHOPRT post-erase | `NTty::finish_erasing` |
| `reset_buffer_flags()` | per-flush reset | `NTty::reset_buffer_flags` |
| `is_utf8_continuation()` / `is_continuation()` | per-IUTF8 multibyte test | `NTty::is_*_continuation` |
| `n_tty_kick_worker()` | per-flip-buf restart | `NTty::kick_worker` |
| `n_tty_check_throttle()` / `_unthrottle()` | per-RX backpressure | `NTty::*throttle` |
| `inq_canon()` | per-TIOCINQ canon count | `NTty::inq_canon` |
| `zero_buffer()` / `tty_copy()` | per-secret scrub on copy out | `NTty::zero_buffer` / `tty_copy` |
| `n_tty_inherit_ops()` | per-subclass inherit | `NTty::inherit_ops` |
| `n_tty_init()` | per-module init | `NTty::init` |

## Compatibility contract

REQ-1: struct n_tty_data layout (must match upstream order/padding for ldisc-private size):
- producer-published (mutated by receive_buf path under non-exclusive termios_rwsem):
  - read_head: per-ring producer index (bytes ever enqueued, monotonic).
  - commit_head: per-non-canon visible cursor (smp_store_release).
  - canon_head: per-ICANON line cursor (last published newline / EOL / EOL2 / EOF).
  - echo_head: per-echo producer.
  - echo_commit: per-echo commit barrier.
  - echo_mark: per-echo speculative end.
  - char_map: per-DECLARE_BITMAP(256) special-char set.
- private to n_tty_receive_overrun (single-threaded, no lock):
  - overrun_time: per-jiffies last warn.
  - num_overrun: per-count since last warn.
- non-atomic:
  - no_room: per-flip-buf throttled flag.
- bitfields under exclusive termios_rwsem (reset on set_termios mode change):
  - lnext:1 — last char was LNEXT.
  - erasing:1 — ECHOPRT active.
  - raw:1, real_raw:1 — receive mode classification.
  - icanon:1 — current canonical state.
  - push:1 — set_termios push semantics.
- shared producer↔consumer:
  - read_buf[N_TTY_BUF_SIZE] (4096) — input ring.
  - read_flags: DECLARE_BITMAP(N_TTY_BUF_SIZE) marking line-delimiter slots.
  - echo_buf[N_TTY_BUF_SIZE] — echo ring.
- consumer-published (under termios_rwsem read + atomic_read_lock):
  - read_tail: per-consumer head (monotonic).
  - line_start: per-canon line begin (for ICANON erase).
- lookahead_count: per-bytes already pre-scanned by lookahead_buf.
- under output_lock:
  - column: per-current column (for OPOST).
  - canon_column: per-line-begin column.
  - echo_tail: per-echo consumer.
- atomic_read_lock: per-mutex serializing readers.
- output_lock: per-mutex serializing echo/output emit.

REQ-2: n_tty_open(tty):
- ldata = vzalloc(sizeof(*ldata)) ∨ -ENOMEM.
- ldata.overrun_time = jiffies.
- mutex_init(&ldata.atomic_read_lock).
- mutex_init(&ldata.output_lock).
- tty.disc_data = ldata.
- tty.closing = 0.
- clear_bit(TTY_LDISC_HALTED, &tty.flags) — resume buffer work.
- n_tty_set_termios(tty, NULL) — populate char_map + flags.
- tty_unthrottle(tty).
- return 0.

REQ-3: n_tty_close(tty):
- if tty.link: n_tty_packet_mode_flush(tty) — notify pty peer.
- guard(rwsem_write)(&tty.termios_rwsem).
- vfree(tty.disc_data).
- tty.disc_data = NULL.

REQ-4: n_tty_flush_buffer(tty):
- guard(rwsem_write)(&tty.termios_rwsem).
- reset_buffer_flags(tty.disc_data):
  - read_head = canon_head = commit_head = read_tail = line_start = 0.
  - bitmap_zero(read_flags, N_TTY_BUF_SIZE).
  - erasing = lnext = 0.
- n_tty_kick_worker(tty).
- if tty.link: n_tty_packet_mode_flush(tty).

REQ-5: n_tty_set_termios(tty, old):
- If old==NULL OR (old.c_lflag ^ new) & (ICANON | EXTPROC):
  - bitmap_zero(read_flags).
  - line_start = read_tail.
  - if !L_ICANON ∨ !read_cnt: canon_head = read_tail; push = 0.
  - else: set_bit(MASK(read_head-1), read_flags); canon_head = read_head; push = 1.
  - commit_head = read_head.
  - erasing = 0; lnext = 0.
- icanon = (L_ICANON != 0).
- /* Special-char map population */
- If any of I_ISTRIP, I_IUCLC, I_IGNCR, I_ICRNL, I_INLCR, L_ICANON, I_IXON, L_ISIG, L_ECHO, I_PARMRK:
  - bitmap_zero(char_map, 256).
  - if I_IGNCR ∨ I_ICRNL: set_bit('\r', char_map).
  - if I_INLCR: set_bit('\n', char_map).
  - if L_ICANON: set_bit(ERASE_CHAR, KILL_CHAR, EOF_CHAR, '\n', EOL_CHAR).
    - if L_IEXTEN: set_bit(WERASE_CHAR, LNEXT_CHAR, EOL2_CHAR); if L_ECHO: set_bit(REPRINT_CHAR).
  - if I_IXON: set_bit(START_CHAR, STOP_CHAR).
  - if L_ISIG: set_bit(INTR_CHAR, QUIT_CHAR, SUSP_CHAR).
  - clear_bit(__DISABLED_CHAR).
  - raw = real_raw = 0.
- Else:
  - raw = 1.
  - real_raw = (I_IGNBRK ∨ (!I_BRKINT ∧ !I_PARMRK)) ∧ (I_IGNPAR ∨ !I_INPCK) ∧ (driver.flags & TTY_DRIVER_REAL_RAW).
- If !I_IXON ∧ old.c_iflag & IXON ∧ !tco_stopped: start_tty + process_echoes.
- wake_up_interruptible(&write_wait); wake_up_interruptible(&read_wait).

REQ-6: Ring-buffer discipline:
- N_TTY_BUF_SIZE == 4096; MASK(x) = x & (N_TTY_BUF_SIZE-1).
- Counters monotonic; difference == bytes pending.
- read_cnt = read_head - read_tail.
- chars_in_buffer = min(read_head - read_tail, commit_head - read_tail) when icanon → canon-based.
- Producer publishes via smp_store_release on commit_head / canon_head.
- Consumer reads via smp_load_acquire on commit_head / canon_head.
- Echo writes use add_echo_byte (paired smp_wmb in add_echo_byte / smp_rmb in echo_buf).

REQ-7: n_tty_receive_buf_common(tty, cp, fp, count, flow):
- guard(rwsem_read)(&tty.termios_rwsem).
- do:
  - tail = smp_load_acquire(&read_tail).
  - room = N_TTY_BUF_SIZE - (read_head - tail).
  - if I_PARMRK: room = DIV_ROUND_UP(room, 3).
  - room--.
  - if room <= 0:
    - overflow = icanon ∧ canon_head == tail.
    - if overflow ∧ room < 0: read_head--.
    - room = overflow; WRITE_ONCE(no_room, flow ∧ !room).
  - else: overflow = 0.
  - n = min(count, room).
  - if !n: break.
  - if !overflow ∨ !fp ∨ fp[0] != TTY_PARITY: __receive_buf(tty, cp, fp, n).
  - cp += n; fp += n if fp; count -= n; rcvd += n.
- while !test_bit(TTY_LDISC_CHANGING, &tty.flags).
- tty.receive_room = room.
- if pty ∧ overflow: tty_set_flow_change(tty, TTY_UNTHROTTLE_SAFE); tty_unthrottle_safe; __tty_set_flow_change(0).
- else: n_tty_check_throttle(tty).
- if no_room: smp_mb; if !chars_in_buffer: n_tty_kick_worker.
- return rcvd.

REQ-8: __receive_buf(tty, cp, fp, count):
- la_count = min(lookahead_count, count).
- /* Dispatch by mode */
- if real_raw: n_tty_receive_buf_real_raw(tty, cp, count) — memcpy into read_buf with 2-segment wrap.
- elif raw ∨ (L_EXTPROC ∧ !preops): n_tty_receive_buf_raw(tty, cp, fp, count) — flagged-byte aware enqueue.
- elif tty.closing ∧ !L_EXTPROC: split la_count vs remainder into n_tty_receive_buf_closing(.., lookahead_done={true,false}).
- else /* cooked */: split la_count vs remainder into n_tty_receive_buf_standard(.., lookahead_done={true,false}); flush_echoes; ops.flush_chars.
- lookahead_count -= la_count.
- if icanon ∧ !L_EXTPROC: return (canon_head publishes are inside the per-char path).
- /* Non-canon publish */
- smp_store_release(&commit_head, read_head).
- if read_cnt > 0: kill_fasync(SIGIO, POLL_IN); wake_up_interruptible_poll(&read_wait, EPOLLIN | EPOLLRDNORM).

REQ-9: n_tty_receive_buf_standard(tty, cp, fp, count, lookahead_done):
- For each byte c with flag f:
  - if f == TTY_NORMAL:
    - if lnext: n_tty_receive_char_lnext(tty, c, TTY_NORMAL).
    - elif (char_map test c): n_tty_receive_char_special(tty, c, lookahead_done).
    - else: n_tty_receive_char(tty, c).
  - else: n_tty_receive_char_flagged(tty, c, f).

REQ-10: n_tty_receive_char_special(tty, c, lookahead_done):
- if I_IXON ∧ n_tty_receive_char_flow_ctrl(tty, c, lookahead_done): return.
- if L_ISIG:
  - c == INTR_CHAR: n_tty_receive_signal_char(tty, SIGINT, c); return.
  - c == QUIT_CHAR: n_tty_receive_signal_char(tty, SIGQUIT, c); return.
  - c == SUSP_CHAR: n_tty_receive_signal_char(tty, SIGTSTP, c); return.
- if flow.stopped ∧ !flow.tco_stopped ∧ I_IXON ∧ I_IXANY: start_tty + process_echoes.
- if c == '\r': if I_IGNCR: return; if I_ICRNL: c = '\n'.
- elif c == '\n' ∧ I_INLCR: c = '\r'.
- if icanon ∧ n_tty_receive_char_canon(tty, c): return.
- if L_ECHO: finish_erasing; echo_char_raw('\n', ldata) if c == '\n', else { if canon_head == read_head: echo_set_canon_col; echo_char(c, tty); }; commit_echoes.
- if c == '\377' ∧ I_PARMRK: put_tty_queue('\377', ldata).
- put_tty_queue(c, ldata).

REQ-11: n_tty_receive_char_canon(tty, c) -> bool:
- ERASE / KILL / (WERASE if L_IEXTEN): eraser(c, tty); commit_echoes; return true.
- LNEXT (if L_IEXTEN): lnext = 1; if L_ECHO ∧ L_ECHOCTL: echo_char_raw('^'); echo_char_raw('\b'); commit_echoes. return true.
- REPRINT (if L_ECHO ∧ L_IEXTEN): re-echo canon_head..read_head; return true.
- '\n': if L_ECHO ∨ L_ECHONL: echo_char_raw('\n'); commit_echoes. n_tty_receive_handle_newline(tty, '\n'). return true.
- EOF_CHAR: c = __DISABLED_CHAR; n_tty_receive_handle_newline(tty, c). return true.
- EOL_CHAR or (EOL2 ∧ L_IEXTEN): optional echo; PARMRK double; n_tty_receive_handle_newline(tty, c). return true.
- else: return false.

REQ-12: n_tty_receive_handle_newline(tty, c):
- set_bit(MASK(read_head), read_flags) — mark line-delimiter slot.
- put_tty_queue(c, ldata).
- smp_store_release(&canon_head, read_head) — publish new line.
- kill_fasync(&tty.fasync, SIGIO, POLL_IN).
- wake_up_interruptible_poll(&tty.read_wait, EPOLLIN | EPOLLRDNORM).

REQ-13: eraser(c, tty) — ERASE / WERASE / KILL:
- if read_head == canon_head: return (nothing to erase).
- kill_type = c == ERASE ? ERASE : c == WERASE ? WERASE : KILL.
- if KILL ∧ (!L_ECHO ∨ !L_ECHOK ∨ !L_ECHOKE ∨ !L_ECHOE):
  - read_head = canon_head; finish_erasing; echo_char(KILL_CHAR); if L_ECHOK: echo_char_raw('\n'). return.
- seen_alnums = 0.
- while MASK(read_head) != MASK(canon_head):
  - walk back over UTF-8 continuation bytes; if partial: break.
  - if WERASE: if isalnum(c) ∨ c=='_': seen_alnums++; elif seen_alnums: break.
  - cnt = read_head - head; read_head = head.
  - if L_ECHO:
    - if L_ECHOPRT: enter erasing mode ('\\') and echo erased chars.
    - elif ERASE ∧ !L_ECHOE: echo_char(ERASE_CHAR).
    - elif c == '\t': count columns since prev tab; echo_erase_tab(num_chars, after_tab).
    - else: if iscntrl ∧ L_ECHOCTL: echo "\b \b" twice (control + visible glyph).
  - if kill_type == ERASE: break.
- if read_head == canon_head ∧ L_ECHO: finish_erasing.

REQ-14: __isig(sig, tty):
- tty_pgrp = tty_get_pgrp(tty) — under ctrl.lock.
- if tty_pgrp: kill_pgrp(tty_pgrp, sig, 1); put_pid(tty_pgrp).

REQ-15: isig(sig, tty):
- if L_NOFLSH: __isig(sig, tty); return.
- /* signal AND flush; require exclusive termios_rwsem */
- up_read(&termios_rwsem).
- scoped_guard(rwsem_write, &termios_rwsem):
  - __isig(sig, tty).
  - scoped_guard(mutex, &ldata.output_lock): echo_head = echo_tail = 0; echo_mark = echo_commit = 0.
  - tty_driver_flush_buffer(tty) — clear driver TX.
  - reset_buffer_flags(tty.disc_data) — clear input ring.
  - if tty.link: n_tty_packet_mode_flush(tty).
- down_read(&termios_rwsem).

REQ-16: n_tty_receive_signal_char(tty, signal, c):
- isig(signal, tty).
- if I_IXON: start_tty(tty).
- if L_ECHO: echo_char(c, tty); commit_echoes; else: process_echoes.

REQ-17: n_tty_receive_char_flow_ctrl(tty, c, lookahead_done) -> bool:
- if !n_tty_is_char_flow_ctrl(tty, c): return false.
- if lookahead_done: return true (already handled by lookahead).
- if c == START_CHAR: start_tty(tty); process_echoes; return true.
- /* STOP_CHAR */ stop_tty(tty); return true.

REQ-18: n_tty_lookahead_flow_ctrl(tty, cp, fp, count):
- lookahead_count += count.
- if !I_IXON: return.
- for each (c, flag=*fp): if flag==TTY_NORMAL: n_tty_receive_char_flow_ctrl(tty, c, false). cp++.

REQ-19: n_tty_receive_break(tty):
- if I_IGNBRK: return.
- if I_BRKINT: isig(SIGINT, tty); return.
- if I_PARMRK: put_tty_queue('\377', ldata); put_tty_queue('\0', ldata).
- put_tty_queue('\0', ldata).

REQ-20: n_tty_receive_parity_error(tty, c):
- if I_INPCK:
  - if I_IGNPAR: return.
  - if I_PARMRK: put_tty_queue('\377'); put_tty_queue('\0'); put_tty_queue(c).
  - else: put_tty_queue('\0', ldata).
- else: put_tty_queue(c, ldata).

REQ-21: n_tty_receive_overrun(tty):
- ldata.num_overrun++.
- if jiffies > overrun_time + HZ: tty_warn "%u input overrun(s)"; reset.

REQ-22: do_output_char(c, tty, space) → int:
- if !space: return -1.
- '\n': if O_ONLRET: column=0; if O_ONLCR: emit "\r\n" (need 2); canon_column = column = 0; return 2. else canon_column = column.
- '\r': if O_ONOCR ∧ column==0: return 0; if O_OCRNL: c='\n'; canon_column = column = 0 if O_ONLRET; break.
- '\t': spaces = 8 - (column & 7); if O_TABDLY == XTABS: emit spaces; column += spaces; return spaces. else column += spaces.
- '\b': if column > 0: column--.
- default: if !iscntrl(c): if O_OLCUC: c = toupper(c); if !is_continuation(c, tty): column++.
- tty_put_char(tty, c). return 1.

REQ-23: process_output(c, tty) → int:
- guard(mutex)(&ldata.output_lock).
- if do_output_char(c, tty, tty_write_room(tty)) < 0: return -1.
- return 0.

REQ-24: process_output_block(tty, buf, nr):
- guard(mutex)(&ldata.output_lock).
- space = tty_write_room(tty); if 0: return 0.
- nr = min(nr, space).
- scan for first OPOST-significant char; on hit: ops.write(tty, buf, i); return i (caller continues with process_output for that char).
- otherwise: ops.write(tty, buf, nr); column tracking updated. return nr.

REQ-25: __process_echoes(tty) → size_t:
- guard(mutex)(&ldata.output_lock).
- space = tty_write_room(tty).
- tail = echo_tail; head = echo_commit (after smp_load_acquire).
- For each byte at echo_buf[MASK(tail)]:
  - If ECHO_OP_START (0xff): decode op (see REQ-26).
  - else: do_output_char(c, tty, space) — same OPOST handling.
- echo_tail = tail.
- return chars-emitted.

REQ-26: n_tty_process_echo_ops(tty, &tail, space) — encoded op decode:
- ECHO_OP_START followed by:
  - ECHO_OP_START: literal 0xff char — do_output_char(0xff, tty, space).
  - ECHO_OP_MOVE_BACK_COL: column -= echo_buf[tail+2] (1-byte count).
  - ECHO_OP_SET_CANON_COL: canon_column = column.
  - ECHO_OP_ERASE_TAB: column adjust for tab erase (3-byte payload).
- Bump tail by op length; return -1 if insufficient space.

REQ-27: echo_char_raw(c, ldata):
- if c == ECHO_OP_START: add_echo_byte(c, ldata); add_echo_byte(ECHO_OP_START, ldata) — escape.
- else: add_echo_byte(c, ldata).

REQ-28: echo_char(c, tty):
- if iscntrl(c) ∧ !is_continuation(c, tty) ∧ c != '\t' ∧ c != '\n':
  - if L_ECHOCTL: echo_char_raw('^', ldata); echo_char_raw(c ^ 0100, ldata).
- else: echo_char_raw(c, ldata).

REQ-29: process_echoes(tty) (non-aggressive): if echo_mark != echo_commit AND !L_FLUSHO: __process_echoes(tty). Trigger from receive paths.

REQ-30: commit_echoes(tty):
- guard(mutex)(&ldata.output_lock).
- old = echo_commit; head = echo_head.
- if old != head: smp_store_release(&echo_commit, head).
- if !L_FLUSHO: __process_echoes(tty).

REQ-31: flush_echoes(tty):
- if echo_mark == echo_commit: return.
- commit_echoes(tty).

REQ-32: n_tty_read(tty, file, kbuf, nr, cookie, offset) → ssize_t:
- if *cookie: return n_tty_continue_cookie(tty, kbuf, nr, cookie).
- retval = job_control(tty, file); if retval < 0: return retval.
- /* Serialize readers */
- if O_NONBLOCK: mutex_trylock(&atomic_read_lock) ∨ -EAGAIN.
- else: mutex_lock_interruptible(&atomic_read_lock) ∨ -ERESTARTSYS.
- down_read(&termios_rwsem).
- minimum = time = 0; timeout = MAX_SCHEDULE_TIMEOUT.
- if !icanon: minimum = MIN_CHAR; if minimum: time = (HZ/10)*TIME_CHAR; else: timeout = (HZ/10)*TIME_CHAR; minimum = 1.
- packet = tty.ctrl.packet; old_tail = read_tail.
- add_wait_queue(&read_wait, &wait).
- loop while nr:
  - if packet ∧ tty.link.ctrl.pktstatus: if kb!=kbuf: break. emit cs byte; nr--; break.
  - if !input_available_p(tty, 0): tty_buffer_flush_work; if still none: n_tty_wait_for_input(tty, file, &wait, &timeout); on -ret: break.
  - if icanon ∧ !L_EXTPROC: canon_copy_from_read_buf(tty, &kb, &nr) → "more_to_be_read" path on saturating return.
  - else: if packet ∧ kb==kbuf: emit TIOCPKT_DATA. copy_from_read_buf(tty, &kb, &nr) → "more_to_be_read" if true ∧ minimum satisfied.
  - n_tty_check_unthrottle(tty).
  - if kb-kbuf >= minimum: break.
  - if time: timeout = time.
- if old_tail != read_tail: smp_mb; n_tty_kick_worker.
- up_read(&termios_rwsem).
- remove_wait_queue(&read_wait, &wait); mutex_unlock(&atomic_read_lock).
- return kb-kbuf or retval.
- more_to_be_read: remove_wait_queue; *cookie = cookie; return kb-kbuf (locks still held — released by continue_cookie when done).

REQ-33: n_tty_continue_cookie(tty, kbuf, nr, cookie):
- if icanon ∧ !L_EXTPROC: if !nr: canon_skip_eof; else if canon_copy_from_read_buf: return kb-kbuf.
- else: if copy_from_read_buf: return kb-kbuf.
- /* No more data — drop the held locks */
- n_tty_kick_worker; n_tty_check_unthrottle.
- up_read(&termios_rwsem); mutex_unlock(&atomic_read_lock).
- *cookie = NULL.

REQ-34: canon_copy_from_read_buf(tty, &kbp, &nr) → bool:
- /* Copy up to and including line-delim char */
- tail = MASK(read_tail).
- canon_head = smp_load_acquire(canon_head).
- n = min(*nr, canon_head - read_tail).
- /* Stop at first line-delimiter in read_flags */
- find delim index d (first set bit between tail..tail+n).
- if d found: copy_to (kbp, n=d+1-tail); read_tail += d+1-tail.
- else: copy_to (kbp, n).
- audit + zero_buffer (scrub erased bytes if !L_ECHO).
- return true if more data may follow (saturating user buffer).

REQ-35: copy_from_read_buf(tty, &kbp, &nr) → bool:
- head = smp_load_acquire(commit_head); tail = MASK(read_tail).
- n = min3(head - read_tail, N_TTY_BUF_SIZE - tail, *nr).
- if !n: return false.
- memcpy(kbp, &read_buf[tail], n).
- is_eof = (n == 1 ∧ *from == EOF_CHAR).
- tty_audit_add_data; zero_buffer.
- smp_store_release(&read_tail, read_tail + n).
- /* EXTPROC: collapse EOF to zero-byte read */
- if L_EXTPROC ∧ icanon ∧ is_eof ∧ head == read_tail: return false.
- *kbp += n; *nr -= n.
- return head != read_tail.

REQ-36: input_available_p(tty, poll) → int:
- amt = (poll ∧ !TIME_CHAR ∧ MIN_CHAR) ? MIN_CHAR : 1.
- if icanon ∧ !L_EXTPROC: return canon_head != read_tail.
- else: return commit_head - read_tail >= amt.

REQ-37: n_tty_wait_for_input(tty, file, wait, *timeout):
- if TTY_OTHER_CLOSED: -EIO.
- if tty_hung_up_p ∨ TTY_HUPPING: 0.
- if !*timeout: 0.
- if O_NONBLOCK: -EAGAIN.
- if signal_pending: -ERESTARTSYS.
- up_read(&termios_rwsem). *timeout = wait_woken(wait, TASK_INTERRUPTIBLE, *timeout). down_read.
- return 1.

REQ-38: job_control(tty, file):
- if file.f_op.write_iter == redirected_tty_write: return 0 /* /dev/console */.
- return __tty_check_change(tty, SIGTTIN).

REQ-39: n_tty_write(tty, file, buf, nr):
- if L_TOSTOP ∧ file.f_op.write_iter != redirected_tty_write: tty_check_change(tty) — may send SIGTTOU.
- guard(rwsem_read)(&termios_rwsem).
- process_echoes(tty).
- add_wait_queue(&write_wait, &wait).
- loop:
  - if signal_pending: -ERESTARTSYS.
  - if tty_hung_up_p ∨ (link ∧ !link.count): -EIO.
  - if O_OPOST:
    - while nr > 0: num = process_output_block(tty, b, nr); if num<0 ∧ num != -EAGAIN: -err; break on -EAGAIN; advance; if nr==0: break. process_output(*b, tty); advance b/nr.
    - if ops.flush_chars: ops.flush_chars(tty).
  - else:
    - while nr > 0: scoped output_lock: num = ops.write(tty, b, nr); advance.
  - if !nr: break.
  - if O_NDELAY-ish (tty_io_nonblock): -EAGAIN.
  - up_read(&termios_rwsem). wait_woken(&wait, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT). down_read.
- remove_wait_queue.
- if nr ∧ tty.fasync: set TTY_DO_WRITE_WAKEUP.
- return (b-buf) ? b-buf : retval.

REQ-40: n_tty_poll(tty, file, wait):
- poll_wait(file, &read_wait, wait); poll_wait(file, &write_wait, wait).
- mask = 0.
- if input_available_p(tty, 1): EPOLLIN | EPOLLRDNORM.
  - else: tty_buffer_flush_work; recheck.
- if packet ∧ link.ctrl.pktstatus: EPOLLPRI | EPOLLIN | EPOLLRDNORM.
- if TTY_OTHER_CLOSED ∨ tty_hung_up_p: EPOLLHUP.
- if ops.write ∧ !tty_is_writelocked ∧ tty_chars_in_buffer < WAKEUP_CHARS ∧ tty_write_room > 0: EPOLLOUT | EPOLLWRNORM.

REQ-41: n_tty_ioctl(tty, cmd, arg):
- TIOCOUTQ: put_user(tty_chars_in_buffer(tty)).
- TIOCINQ: scoped(rwsem_write, termios_rwsem):
  - if L_ICANON ∧ !L_EXTPROC: inq_canon(ldata) — bytes up to canon_head, excluding __DISABLED_CHAR slots.
  - else: read_cnt(ldata).
- default: n_tty_ioctl_helper(tty, cmd, arg) — TCXONC, TCFLSH, TIOCPKT etc.

REQ-42: inq_canon(ldata) → ulong:
- if canon_head == read_tail: return 0.
- head = canon_head; tail = read_tail; nr = head - tail.
- /* Skip EOF chars (marked in read_flags ∧ value == __DISABLED_CHAR) */
- while MASK(head) != MASK(tail): if test_bit(MASK(tail), read_flags) ∧ read_buf(ldata, tail) == __DISABLED_CHAR: nr--. tail++.
- return nr.

REQ-43: n_tty_ops (struct tty_ldisc_ops):
- owner = THIS_MODULE.
- num = N_TTY.
- name = "n_tty".
- open = n_tty_open; close = n_tty_close.
- flush_buffer = n_tty_flush_buffer.
- read = n_tty_read; write = n_tty_write.
- ioctl = n_tty_ioctl.
- set_termios = n_tty_set_termios.
- poll = n_tty_poll.
- receive_buf = n_tty_receive_buf.
- write_wakeup = n_tty_write_wakeup.
- receive_buf2 = n_tty_receive_buf2 (returns count consumed for flow ctrl).
- lookahead_buf = n_tty_lookahead_flow_ctrl.

REQ-44: n_tty_inherit_ops(ops):
- *ops = n_tty_ops; ops.owner = NULL — subclass discipline can override fields.

REQ-45: n_tty_init():
- tty_register_ldisc(&n_tty_ops) — registers N_TTY (index 0) in the global tty_ldiscs table.

REQ-46: Per-zero-buffer secret-scrub:
- zero_buffer(tty, buffer, size): if L_ICANON ∧ !L_ECHO: memset(buffer, 0, size).
- tty_copy: after memcpy, zero source region (passwords typed under stty -echo).

REQ-47: Per-PARMRK doubling:
- '\377' input under PARMRK: queued as '\377' '\377'.
- Break / parity errors under PARMRK: queued as '\377' '\0' [c].

REQ-48: Per-EXTPROC:
- When L_EXTPROC is set, n_tty mostly defers processing to userspace (BSD tip(1) compatibility).
- Receive: raw enqueue (no OPOST), EOF collapses to zero-length read.
- Set-termios: treats ICANON|EXTPROC change as buffer reset trigger.

REQ-49: Per-IUTF8 column tracking:
- is_continuation(c, tty) = I_IUTF8 ∧ (c & 0xc0) == 0x80 — UTF-8 continuation byte does not advance column.
- Used in do_output_char (column++) and eraser (partial-multibyte protection).

## Acceptance Criteria

- [ ] AC-1: ICANON: read returns one line at a time up to newline / EOL / EOL2 / EOF; multiple short reads if buffer too small.
- [ ] AC-2: ICANON: ERASE_CHAR removes last char; WERASE removes word; KILL removes whole line; LNEXT escapes next char.
- [ ] AC-3: Non-canonical: MIN/TIME timing per termios(3); MIN=0,TIME=0 returns immediately.
- [ ] AC-4: ECHO/ECHOE/ECHOK/ECHOKE/ECHOCTL/ECHOPRT emit correct echo bytes to driver write path.
- [ ] AC-5: OPOST + ONLCR: '\n' emitted as "\r\n" (2 bytes consumed by driver write).
- [ ] AC-6: ISIG: INTR_CHAR (default C-c) generates SIGINT to foreground PGID via kill_pgrp.
- [ ] AC-7: ISIG: QUIT_CHAR (C-\\) generates SIGQUIT; SUSP_CHAR (C-z) generates SIGTSTP.
- [ ] AC-8: NOFLSH set: signal sent without flushing input/output buffers.
- [ ] AC-9: NOFLSH unset: signal causes flush of input, echo, and driver TX buffers.
- [ ] AC-10: IXON: STOP_CHAR (C-s) stops output; START_CHAR (C-q) resumes.
- [ ] AC-11: EOF char (default C-d) at canon position yields zero-length read; mid-line yields current line without LF.
- [ ] AC-12: PARMRK: '\377' input doubled; break and parity errors prefixed "\377\0".
- [ ] AC-13: IGNBRK / BRKINT / IGNPAR / INPCK / IGNCR / ICRNL / INLCR honored per termios(3).
- [ ] AC-14: zero_buffer scrubs read_buf on copy_out when ICANON ∧ !ECHO (password protection).
- [ ] AC-15: receive_buf overflow handling: ICANON full canon line beyond 4096 chars truncated; no oops.
- [ ] AC-16: smp_store_release(canon_head) publishes line; reader's smp_load_acquire pairs.
- [ ] AC-17: TIOCINQ in ICANON returns inq_canon (skipping __DISABLED_CHAR EOF slots).
- [ ] AC-18: TIOCOUTQ returns tty_chars_in_buffer (driver's TX pending).
- [ ] AC-19: n_tty_poll: EPOLLIN when input_available_p; EPOLLOUT when write_room > 0 ∧ chars_in_buffer < WAKEUP_CHARS; EPOLLHUP on TTY_OTHER_CLOSED.
- [ ] AC-20: TOSTOP: background process write triggers SIGTTOU via tty_check_change.

## Architecture

```
struct NTtyData {
  // producer-published
  read_head: usize,
  commit_head: usize,
  canon_head: usize,
  echo_head: usize,
  echo_commit: usize,
  echo_mark: usize,
  char_map: BitSet256,

  // private to receive_overrun
  overrun_time: u64,                       // jiffies
  num_overrun: u32,

  // non-atomic
  no_room: bool,

  // bitfields under exclusive termios_rwsem
  lnext: u1,
  erasing: u1,
  raw: u1,
  real_raw: u1,
  icanon: u1,
  push: u1,

  // shared producer/consumer
  read_buf: [u8; N_TTY_BUF_SIZE],          // 4096
  read_flags: BitSet<N_TTY_BUF_SIZE>,
  echo_buf: [u8; N_TTY_BUF_SIZE],

  // consumer-published
  read_tail: usize,
  line_start: usize,

  lookahead_count: usize,

  // protected by output_lock
  column: u32,
  canon_column: u32,
  echo_tail: usize,

  atomic_read_lock: Mutex,
  output_lock: Mutex,
}

const N_TTY_BUF_SIZE: usize = 4096;
fn MASK(x: usize) -> usize { x & (N_TTY_BUF_SIZE - 1) }
```

`NTty::open(tty) -> i32`:
1. ldata = vzalloc(NTtyData).
2. ldata.overrun_time = jiffies.
3. mutex_init(&atomic_read_lock); mutex_init(&output_lock).
4. tty.disc_data = ldata; tty.closing = 0; clear TTY_LDISC_HALTED.
5. NTty::set_termios(tty, NULL).
6. tty_unthrottle(tty).
7. return 0.

`NTty::receive_buf_common(tty, cp, fp, count, flow) -> usize`:
1. guard(rwsem_read)(&termios_rwsem).
2. do:
   - tail = smp_load_acquire(&read_tail).
   - room = N_TTY_BUF_SIZE - (read_head - tail); if I_PARMRK: /=3; room--.
   - if room <= 0: overflow handling (drop char or shrink read_head).
   - n = min(count, room); if !n: break.
   - if !overflow ∨ !fp ∨ fp[0] != TTY_PARITY: NTty::receive_buf_inner(tty, cp, fp, n).
   - advance.
3. while !TTY_LDISC_CHANGING.
4. tty.receive_room = room.
5. throttle / unthrottle bookkeeping.
6. return rcvd.

`NTty::receive_buf_inner(tty, cp, fp, count)` /* __receive_buf */:
1. la_count = min(lookahead_count, count).
2. Mode dispatch:
   - real_raw: NTty::receive_buf_real_raw.
   - raw ∨ (L_EXTPROC ∧ !preops): NTty::receive_buf_raw.
   - tty.closing ∧ !L_EXTPROC: NTty::receive_buf_closing × {true,false}.
   - else: NTty::receive_buf_standard × {true,false}; flush_echoes; ops.flush_chars.
3. lookahead_count -= la_count.
4. if icanon ∧ !L_EXTPROC: return (canon publishes inline).
5. smp_store_release(&commit_head, read_head).
6. if read_cnt > 0: kill_fasync(&fasync, SIGIO, POLL_IN); wake_up_interruptible_poll(&read_wait, EPOLLIN | EPOLLRDNORM).

`NTty::receive_char_canon(tty, c) -> bool`:
1. ERASE / KILL / (WERASE if L_IEXTEN): NTty::eraser(c, tty); commit_echoes. return true.
2. LNEXT if L_IEXTEN: lnext = 1; echo ^H pair if L_ECHO ∧ L_ECHOCTL. return true.
3. REPRINT if L_ECHO ∧ L_IEXTEN: echo full line. return true.
4. '\n': echo '\n' if L_ECHO ∨ L_ECHONL. NTty::receive_handle_newline(tty, c). return true.
5. EOF_CHAR: c = __DISABLED_CHAR; NTty::receive_handle_newline(tty, c). return true.
6. EOL_CHAR / EOL2_CHAR (with L_IEXTEN): optional echo; PARMRK double; NTty::receive_handle_newline. return true.
7. else: return false.

`NTty::receive_handle_newline(tty, c)`:
1. set_bit(MASK(read_head), read_flags).
2. put_tty_queue(c, ldata).
3. smp_store_release(&canon_head, read_head).
4. kill_fasync(&fasync, SIGIO, POLL_IN).
5. wake_up_interruptible_poll(&read_wait, EPOLLIN | EPOLLRDNORM).

`NTty::eraser(c, tty)` /* ERASE / WERASE / KILL */:
1. if read_head == canon_head: return.
2. kill_type from c.
3. KILL fast path (no ECHOE/ECHOK/ECHOKE) → blast canon line; echo KILL_CHAR; return.
4. while MASK(read_head) != MASK(canon_head):
   - walk back over UTF-8 continuation bytes; abort on partial.
   - WERASE: track seen_alnums to find word boundary.
   - reduce read_head; emit echo bytes per ECHO/ECHOPRT/ECHOCTL/ECHOE rules.
   - kill_type == ERASE: break.
5. finish_erasing if cursor at canon_head and L_ECHO.

`NTty::isig(sig, tty)`:
1. if L_NOFLSH: NTty::isig_inner(sig, tty). return.
2. up_read(&termios_rwsem).
3. scoped_guard(rwsem_write, &termios_rwsem):
   - NTty::isig_inner(sig, tty) — kill_pgrp under ctrl.lock-aware tty_get_pgrp.
   - scoped(output_lock): echo ring → empty.
   - tty_driver_flush_buffer(tty).
   - reset_buffer_flags(tty.disc_data).
   - if tty.link: NTty::packet_mode_flush(tty).
4. down_read(&termios_rwsem).

`NTty::read(tty, file, kbuf, nr, cookie, offset) -> isize`:
1. if *cookie: return NTty::continue_cookie(tty, kbuf, nr, cookie).
2. retval = NTty::job_control(tty, file); if <0: return.
3. atomic_read_lock acquire (trylock if O_NONBLOCK).
4. down_read(&termios_rwsem).
5. /* non-canon: minimum / time from MIN/TIME */
6. add wait_queue.
7. loop while nr:
   - pty packet mode status byte.
   - if !input_available_p: flush flip work; if still none: NTty::wait_for_input.
   - if icanon ∧ !L_EXTPROC: NTty::canon_copy_from_read_buf → "more_to_be_read".
   - else: TIOCPKT_DATA; NTty::copy_from_read_buf → "more_to_be_read".
   - check_unthrottle.
   - if kb-kbuf >= minimum: break.
8. if old_tail != read_tail: smp_mb; kick_worker.
9. up_read; remove_wait_queue; mutex_unlock.
10. return kb-kbuf or retval.
11. more_to_be_read: remove wait_queue; *cookie = cookie; locks remain (released by continue_cookie).

`NTty::write(tty, file, buf, nr) -> isize`:
1. L_TOSTOP + non-/dev/console: tty_check_change (SIGTTOU).
2. guard(rwsem_read)(&termios_rwsem).
3. process_echoes(tty).
4. add wait_queue.
5. loop:
   - signal_pending → -ERESTARTSYS.
   - tty_hung_up_p ∨ (link ∧ !link.count) → -EIO.
   - if O_OPOST: process_output_block + process_output drain.
   - else: ops.write under output_lock.
   - !nr → break.
   - tty_io_nonblock → -EAGAIN.
   - up_read; wait_woken; down_read.
6. remove wait_queue.
7. if nr ∧ tty.fasync: set TTY_DO_WRITE_WAKEUP.
8. return (b-buf) or retval.

`NTty::poll(tty, file, wait) -> u32`:
1. poll_wait(file, &read_wait); poll_wait(file, &write_wait).
2. mask=0.
3. if input_available_p(tty, 1): EPOLLIN | EPOLLRDNORM.
4. if pty packet ∧ link.pktstatus: EPOLLPRI | EPOLLIN | EPOLLRDNORM.
5. if TTY_OTHER_CLOSED ∨ tty_hung_up_p: EPOLLHUP.
6. if writable: EPOLLOUT | EPOLLWRNORM.

`NTty::set_termios(tty, old)`:
1. If ICANON | EXTPROC changed: reset read_flags + adjust canon_head/commit_head depending on previous data.
2. icanon = (L_ICANON != 0).
3. Recompute char_map (special-char bitmap) based on flags.
4. raw / real_raw classification.
5. IXON drop while STOP_CHAR held: start_tty + process_echoes.
6. Wake write_wait + read_wait.

`NTty::flush_buffer(tty)`:
1. guard(rwsem_write)(&termios_rwsem).
2. reset_buffer_flags.
3. kick_worker.
4. if link: packet_mode_flush.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ring_indices_monotonic` | INVARIANT | per-receive: read_head only increases (within u-modulo); commit_head <= read_head; read_tail <= commit_head; canon_head ∈ [read_tail, read_head]. |
| `canon_head_under_termios_rwsem` | INVARIANT | per-receive: canon_head publish only with non-exclusive termios_rwsem held. |
| `commit_head_under_termios_rwsem` | INVARIANT | per-receive: commit_head publish only with non-exclusive termios_rwsem. |
| `smp_release_paired_with_acquire` | INVARIANT | per-commit_head, canon_head: smp_store_release on producer; smp_load_acquire on consumer. |
| `read_tail_under_atomic_read_lock` | INVARIANT | per-NTty::read consumer: atomic_read_lock held when advancing read_tail. |
| `output_lock_held_for_column` | INVARIANT | per-do_output_char: output_lock held when mutating column. |
| `output_lock_held_for_echo_tail` | INVARIANT | per-__process_echoes: output_lock held when advancing echo_tail. |
| `isig_signal_to_foreground_pgrp` | INVARIANT | per-isig: kill_pgrp targets tty.ctrl.pgrp only. |
| `noflsh_skips_flush` | INVARIANT | per-isig with L_NOFLSH: no buffer flush. |
| `zero_buffer_on_canon_no_echo` | INVARIANT | per-copy_out: L_ICANON ∧ !L_ECHO ⟹ memset 0 on source bytes. |
| `lnext_consumed_after_one_char` | INVARIANT | per-receive_char_lnext: ldata.lnext cleared after first byte consumed. |
| `eraser_no_partial_utf8` | INVARIANT | per-eraser: aborts erase of a multi-byte char if first byte is a continuation. |
| `n_tty_buf_size_power_of_two` | INVARIANT | N_TTY_BUF_SIZE & (N_TTY_BUF_SIZE-1) == 0; MASK is bit-AND. |

### Layer 2: TLA+

`drivers/tty/n-tty.tla`:
- Per-receive (producer) + per-read (consumer) + per-echo (producer/consumer) + per-set_termios.
- Properties:
  - `safety_no_overflow_silent_corrupt` — per-receive: ring overflow either drops oldest canon line (icanon) or refuses new input; no overlap of producer / consumer windows.
  - `safety_canon_line_termination` — per-icanon-publish: every visible byte at offset i where read_flags[i] is set is `\n`, EOL, EOL2, or __DISABLED_CHAR.
  - `safety_echo_op_decode` — per-process_echoes: ECHO_OP_START never followed by truncated payload (commit boundary respected).
  - `safety_isig_only_when_lisig` — per-receive_char_special: SIGINT / SIGQUIT / SIGTSTP only sent when L_ISIG is set.
  - `safety_eof_zero_length_read` — per-canon EOF_CHAR at canon position with empty preceding line: read returns 0.
  - `safety_extproc_raw_passthrough` — per-EXTPROC: receive enqueues bytes without OPOST.
  - `liveness_n_tty_read_eventually_returns` — per-blocking read with signal_pending: returns -ERESTARTSYS.
  - `liveness_n_tty_write_drains` — per-write with flow stopped → resumed: write eventually completes or returns -EIO/-ERESTARTSYS.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `NTty::open` post: tty.disc_data != NULL ∧ icanon == (L_ICANON != 0) | `NTty::open` |
| `NTty::close` post: tty.disc_data == NULL ∧ ldata freed | `NTty::close` |
| `NTty::receive_buf_common` post: rcvd <= count ∧ read_head - read_tail <= N_TTY_BUF_SIZE | `NTty::receive_buf_common` |
| `NTty::receive_char_canon` post: returns true ⟹ char consumed (no put_tty_queue duplicate) | `NTty::receive_char_canon` |
| `NTty::eraser` post: read_head ∈ [canon_head, original read_head]; never below canon_head | `NTty::eraser` |
| `NTty::isig` post: kill_pgrp(tty.ctrl.pgrp, sig, 1) called exactly once | `NTty::isig` |
| `NTty::canon_copy_from_read_buf` post: return data ends with line-delim OR fills *nr | `NTty::canon_copy_from_read_buf` |
| `NTty::copy_from_read_buf` post: n bytes copied; smp_store_release of read_tail | `NTty::copy_from_read_buf` |
| `NTty::write` post: returns bytes sent or negative errno; respects O_OPOST mode | `NTty::write` |
| `NTty::poll` post: mask reflects input_available_p ∧ writability ∧ hangup | `NTty::poll` |
| `NTty::set_termios` post: char_map matches I_*/L_* flag set per REQ-5 | `NTty::set_termios` |
| `NTty::ioctl(TIOCOUTQ)` post: returns tty_chars_in_buffer(tty) | `NTty::ioctl` |
| `NTty::ioctl(TIOCINQ)` post: ICANON ⟹ inq_canon (skip __DISABLED_CHAR); else read_cnt | `NTty::ioctl` |

### Layer 4: Verus/Creusot functional

`Per-input → receive_buf → cooked processing → canon publish or non-canon publish → user read` and `per-write → OPOST → driver write` semantic equivalence: per-IEEE Std 1003.1-2017 (POSIX.1-2017) §11 (General Terminal Interface), per-`termios(3)` manpage, per-Documentation/driver-api/tty/n_tty.rst, and per the tty-self-tests in `tools/testing/selftests/tty/`. SIGINT/SIGQUIT/SIGTSTP delivery to foreground PGID equivalent to upstream `__isig` model: targets `tty_get_pgrp(tty)` only and respects L_NOFLSH flush semantics.

## Hardening

(Inherits row-1 features from `drivers/tty/00-overview.md` § Hardening.)

N_TTY reinforcement:

- **Per-zero_buffer secret scrub** — defense against per-/proc/<pid>/mem leaking previously-typed password bytes that were read but lived in `read_buf`.
- **Per-tty_copy zero-after-memcpy** — defense against per-stale-secret in `read_buf` cells after copy-to-user.
- **Per-SMP store_release / load_acquire on canon_head / commit_head** — defense against per-CPU consumer reading torn or premature line.
- **Per-non_exclusive termios_rwsem on receive** — defense against per-set_termios racing with receive_buf.
- **Per-exclusive termios_rwsem on set_termios + isig-flush** — defense against per-reader observing inconsistent canon state.
- **Per-atomic_read_lock single-reader** — defense against per-multi-reader interleaving canon line consumption.
- **Per-output_lock around echo / OPOST column** — defense against per-tearing of column tracking + echo ring.
- **Per-PARMRK doubling** — defense against per-data byte masquerading as parity-error marker.
- **Per-EXTPROC opt-out** — defense against per-double-processing when userspace already does discipline (BSD tip).
- **Per-no_partial_UTF8 in eraser** — defense against per-corrupting multibyte sequence.
- **Per-EOF_CHAR collapsed to __DISABLED_CHAR** — defense against per-EOF byte leaking into user buffer mid-line.
- **Per-overflow drop in ICANON ⟹ canon_head==tail** — defense against per-runaway-line DoS.
- **Per-NOFLSH respected** — defense against per-debugger losing buffered ttymouse-state on signal.
- **Per-WAKEUP_CHARS hysteresis on EPOLLOUT** — defense against per-write-storm thundering herd.
- **Per-lookahead_buf for IXON** — defense against per-deferred flow control storing characters before STOP_CHAR is honored.
- **Per-TTY_LDISC_CHANGING tested in receive_buf_common loop** — defense against per-receive into half-installed n_tty after TIOCSETD.
- **Per-tty_audit_add_data** — defense against per-audit gap on copy-out.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `n_tty.read_buf` (the canonical line buffer) copied to userspace through bounded `copy_to_user` with explicit `min(nr, can_read)`; defends against ICANON line-buffer underflow leaking adjacent slab.
- **PAX_KERNEXEC** — `tty_ldisc_ops n_tty_ops` and the receive_buf / write callback table placed in `__ro_after_init`; defends ldisc dispatch against vtable rewrite.
- **PAX_RANDKSTACK** — entropy added on every `n_tty_read` / `n_tty_receive_buf_common` entry; neutralises stack-shape probing under repeated TIOCSETD races.
- **PAX_REFCOUNT** — `tty_struct.count`, `n_tty_data.column`, and `tty_ldisc.users` use saturating counters; defends against count-wrap UAF on rapid open/close.
- **PAX_MEMORY_SANITIZE** — `read_buf`, `echo_buf`, and `lookahead_buf` zero-on-free so cleartext keystrokes (passwords typed at the password: prompt) never bleed into the next allocation.
- **PAX_UDEREF** — `n_tty_read` / `n_tty_write` dereference user pointers only via user-AS-annotated `copy_*_user`.
- **PAX_RAP / kCFI** — every `tty_ldisc_ops` callback (`open`, `close`, `receive_buf`, `write_wakeup`, `set_termios`) type-tagged; defends against ROP through a swapped ldisc pointer set via TIOCSETD.
- **GRKERNSEC_HIDESYM** — `/proc/tty/ldiscs`, `/sys/class/tty/*` attribute reads sanitised; defends against ldisc-pointer disclosure.
- **GRKERNSEC_DMESG** — `n_tty` overflow / `tty_audit_buf` warnings gated behind `CAP_SYSLOG`.
- **ICANON line buffer PAX_USERCOPY** — `read_buf` copy-out length clipped to `tail - canon_head` and to `read_cnt`; defends against the historic ICANON line-buffer wraparound CVE class.
- **ECHO restriction** — `ECHO` is suppressed for noecho-flagged sessions; `n_tty_set_termios` enforces ECHO can only be set on a tty the caller can write; defends against echo-pollution password capture.
- **SIGINT/SIGQUIT generation** — `__isig` and `isig` deliver signals only to the controlling session's foreground process group; defends against cross-session signal injection through a smuggled tty.
- **Rationale** — n_tty is in the trusted path of every login, sudo, ssh, and serial console; PAX_USERCOPY on read_buf + RAP on the ldisc vtable + ECHO/SIG controls together protect both keystroke confidentiality and session integrity.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/tty/tty_io.c` core (covered in `tty-io.md` Tier-3)
- `drivers/tty/tty_ldisc.c` discipline registration / refcount (covered separately)
- `drivers/tty/tty_buffer.c` flip buffers (covered separately)
- `drivers/tty/tty_port.c` (covered separately)
- `drivers/tty/tty_jobctrl.c` (SIGTTIN/SIGTTOU machinery in `__tty_check_change`; covered separately)
- `drivers/tty/pty.c` PTY pair driver (covered separately)
- `drivers/tty/tty_ioctl.c` termios ioctls (TCSETS/TCGETS, etc.; covered separately)
- Other line disciplines (N_PPP, N_SLIP, N_HDLC, N_GSM, N_R3964; covered if expanded)
- Implementation code
