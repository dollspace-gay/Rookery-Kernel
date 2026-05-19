# Tier-3: drivers/tty/tty_io.c — TTY core (struct tty_struct, drivers, file_operations)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/tty/00-overview.md
upstream-paths:
  - drivers/tty/tty_io.c (~3672 lines)
  - include/linux/tty.h
  - include/linux/tty_driver.h
  - include/linux/tty_port.h
  - include/linux/tty_ldisc.h
  - include/uapi/asm-generic/ioctls.h
  - include/uapi/asm-generic/termbits.h
-->

## Summary

The **TTY core** is the kernel glue between the character-device VFS layer, a hardware (or PTY) tty driver, and a line discipline. Per-tty state is `struct tty_struct` (count, index, termios, flags, ctrl.{session,pgrp,pktstatus}, ctrl.lock, flow.lock, ldisc_sem, termios_rwsem, atomic_write_lock, port pointer, link for pty pair, fasync list, file list, hangup_work, SAK_work, read_wait / write_wait, write_buf, ldisc, ops, driver, dev). Per-driver state is `struct tty_driver` (major, minor_start, num, type/subtype, flags, ops, termios[idx] table, ttys[idx] table, ports[idx] table, cdevs[idx] table, flip_wq, owner module). Per-driver methods are `struct tty_operations` (open, close, write, put_char, flush_chars, write_room, chars_in_buffer, ioctl, compat_ioctl, set_termios, throttle, unthrottle, stop, start, hangup, break_ctl, flush_buffer, set_ldisc, wait_until_sent, send_xchar, tiocmget, tiocmset, get_serial, set_serial, get_icount, install, remove, lookup, shutdown, cleanup, ...). Per-physical-side state is `struct tty_port` (mutex, buf, itty, lock, count, blocked_open, ops). The line discipline (default N_TTY) plugs in via `tty_ldisc_ops` and is switched at runtime via `TIOCSETD`. `/dev/tty` (TTYAUX_MAJOR:0), `/dev/console` (TTYAUX_MAJOR:1) and `/dev/ttyN` (per driver, dynamic or static) are registered as cdevs via `tty_register_driver` / `tty_init`. Lifecycle: `tty_open` → `tty_init_dev` → driver `install` → `tty_ldisc_setup` → driver `open`; `tty_release` → driver `close` → drop ref → final → `release_tty`. Hangup (`tty_hangup` / `tty_vhangup`): swap `f_op` to `hung_up_tty_fops`, signal session leader (SIGHUP), clear ctrl.session / ctrl.pgrp, call driver `hangup`. Controlling-tty + session/PGID semantics: O_NOCTTY governs acquisition; `vhangup` evicts the session-leader. Critical for: serial consoles, login over getty, pty-based terminals (xterm, sshd), job-control + signal delivery (SIGHUP/SIGTTIN/SIGTTOU), and the kernel-printk console path.

This Tier-3 covers `drivers/tty/tty_io.c` (~3672 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tty_struct` | per-tty instance state | `TtyStruct` |
| `struct tty_driver` | per-driver table | `TtyDriver` |
| `struct tty_operations` | per-driver vtable | `TtyOperations` |
| `struct tty_port` | per-hardware-port state | `TtyPort` |
| `struct tty_ldisc` | per-tty line-discipline binding | `TtyLdisc` |
| `tty_fops` / `console_fops` / `hung_up_tty_fops` | per-fd vtables | `TtyFops` etc. |
| `tty_open()` | per-open dispatch | `Tty::open` |
| `tty_release()` | per-close dispatch | `Tty::release` |
| `tty_read()` / `tty_write()` | per-iov_iter dispatch | `Tty::read` / `Tty::write` |
| `tty_poll()` / `tty_fasync()` | per-fd events | `Tty::poll` / `Tty::fasync` |
| `tty_ioctl()` / `tty_compat_ioctl()` | per-ioctl dispatch | `Tty::ioctl` / `Tty::compat_ioctl` |
| `tty_init_dev()` | per-first-open construction | `Tty::init_dev` |
| `alloc_tty_struct()` / `free_tty_struct()` | per-tty alloc/free | `Tty::alloc_struct` / `Tty::free_struct` |
| `tty_reopen()` | per-re-open shortcut | `Tty::reopen` |
| `release_tty()` / `tty_release_struct()` | per-final destruction | `Tty::release_inner` |
| `tty_kref_put()` | per-refcount drop | `Tty::kref_put` |
| `__tty_hangup()` / `tty_hangup()` / `tty_vhangup()` / `tty_vhangup_session()` | per-hangup paths | `Tty::hangup_*` |
| `do_tty_hangup()` | per-workqueue worker | `Tty::hangup_work` |
| `tty_register_driver()` / `tty_unregister_driver()` | per-driver registration | `Tty::register_driver` / `_unregister` |
| `__tty_alloc_driver()` / `tty_driver_kref_put()` | per-driver alloc/free | `TtyDriver::alloc` / `_kref_put` |
| `tty_register_device()` / `tty_register_device_attr()` / `tty_unregister_device()` | per-instance sysfs/devfs | `Tty::register_device` etc. |
| `tty_cdev_add()` | per-cdev_add wrapper | `Tty::cdev_add` |
| `tty_open_current_tty()` / `tty_open_by_driver()` | per-/dev/tty + per-/dev/ttyN | `Tty::open_current_tty` / `_by_driver` |
| `tty_kopen_exclusive()` / `tty_kopen_shared()` | per-in-kernel open | `Tty::kopen_*` |
| `tty_kclose()` | per-in-kernel close | `Tty::kclose` |
| `tty_driver_lookup_tty()` / `tty_driver_install_tty()` / `tty_driver_remove_tty()` | per-driver ttys[] table | `Tty::driver_*_tty` |
| `tty_init_termios()` / `tty_save_termios()` / `tty_standard_install()` | per-termios slot | `Tty::*_termios` |
| `iterate_tty_read()` / `iterate_tty_write()` | per-chunked transfer | `Tty::iterate_*` |
| `redirected_tty_write()` | per-console-redirect path | `Tty::redirected_write` |
| `tty_write_lock()` / `tty_write_unlock()` | per-atomic_write_lock | `Tty::write_lock` / `_unlock` |
| `tty_wakeup()` | per-driver-ready notify | `Tty::wakeup` |
| `__stop_tty()` / `stop_tty()` / `__start_tty()` / `start_tty()` | per-flow stop/start | `Tty::stop` / `_start` |
| `tty_send_xchar()` | per-X-char emit | `Tty::send_xchar` |
| `tty_do_resize()` | per-TIOCSWINSZ | `Tty::do_resize` |
| `tty_set_ldisc()` (via `tiocsetd`) | per-N_*  switch | `Tty::set_ldisc` |
| `tty_ldisc_ref_wait()` / `tty_ldisc_deref()` / `tty_ldisc_lock()` / `_unlock()` | per-ldisc borrow | `Ldisc::ref_wait` / `_deref` / `_lock` / `_unlock` |
| `tty_ldisc_setup()` / `tty_ldisc_init()` / `tty_ldisc_hangup()` | per-ldisc lifecycle | `Ldisc::setup` / `_init` / `_hangup` |
| `tty_lock()` / `tty_unlock()` / `tty_lock_slave()` / `tty_unlock_slave()` | per-legacy_mutex | `Tty::lock` etc. |
| `tty_get_tiocm()` / `tty_tiocmget()` / `tty_tiocmset()` | per-modem-line | `Tty::tiocmget` / `_set` |
| `tty_get_icount()` / `tty_tiocgicount()` | per-statistics | `Tty::get_icount` |
| `tty_tiocsserial()` / `tty_tiocgserial()` / `compat_tty_tiocsserial()` / `compat_tty_tiocgserial()` | per-serial-cfg | `Tty::tioc*serial` |
| `tiocsti()` | per-fake-input | `Tty::tiocsti` |
| `tiocgwinsz()` / `tiocswinsz()` | per-winsize | `Tty::tiocgwinsz` / `_swinsz` |
| `tioccons()` | per-console-redirect set | `Tty::tioccons` |
| `tiocgetd()` / `tiocsetd()` | per-ldisc id get/set | `Tty::tiocgetd` / `_setd` |
| `send_break()` | per-TCSBRK/TCSBRKP | `Tty::send_break` |
| `tty_pair_get_tty()` | per-pty pair lookup | `Tty::pair_get` |
| `tty_jobctrl_ioctl()` (via tty_jobctrl.c) | per-jobctrl ioctls | external |
| `__do_SAK()` / `do_SAK()` / `do_SAK_work()` | per-secure-attention-key | `Tty::sak` / `_work` |
| `tty_paranoia_check()` / `check_tty_count()` / `tty_release_checks()` | per-invariant validation | `Tty::*_check` |
| `tty_legacy_tiocsti` / `tty_ldisc_autoload` | per-sysctl | `TtySysctl` |
| `tty_class` | per-/sys/class/tty | `TtyClass` |
| `tty_init()` | per-module init | `Tty::init` |
| `tty_default_fops()` | per-vt fops export | `Tty::default_fops` |
| `console_sysfs_notify()` | per-console attribute change | `Tty::console_sysfs_notify` |
| `tty_devnum()` | per-dev_t for tty | `Tty::devnum` |
| `tty_dev_name_to_number()` | per-name-to-dev_t | `Tty::dev_name_to_number` |
| `tty_find_polling_driver()` | per-kgdb/early polling | `Tty::find_polling_driver` |
| `tty_alloc_file()` / `tty_add_file()` / `tty_free_file()` / `tty_del_file()` | per-file handle | `Tty::*_file` |
| `tty_hung_up_p()` | per-hangup probe | `Tty::hung_up_p` |
| `tty_show_fdinfo()` | per-/proc/<pid>/fdinfo/<fd> | `Tty::show_fdinfo` |
| `tty_std_termios` | per-default termios | `Tty::STD_TERMIOS` |
| `tty_table` | per-dev/tty sysctl | `Tty::SYSCTL_TABLE` |

## Compatibility contract

REQ-1: struct tty_struct:
- magic: TTY_MAGIC (for tty_paranoia_check).
- kref: per-refcount.
- dev: per-struct device.
- driver: per-tty_driver.
- ops: per-tty_operations (mirror of driver.ops at alloc time).
- index: per-minor index into driver.ttys[].
- ldisc_sem: per-ldsem (ldisc reference protection).
- ldisc: per-current line discipline.
- legacy_mutex: per-tty_lock (BTM-replacement; ordering tty before slave).
- throttle_mutex: per-throttle/unthrottle serialization.
- termios_rwsem: per-termios protection.
- winsize_mutex: per-winsize protection.
- winsize: per-rows/cols/xpixel/ypixel.
- flow.lock: per-stopped/tco_stopped/{stopped} state spinlock.
- flow.stopped / flow.tco_stopped: per-flow control state.
- ctrl.lock: per-pgrp/session spinlock.
- ctrl.pgrp / ctrl.session: per-pid pointer (PIDTYPE_PGID / SID).
- ctrl.pktstatus: per-packet-mode pty status bits.
- ctrl.packet: per-packet-mode flag.
- atomic_write_lock: per-write serialization mutex.
- legacy port: per-tty_port pointer.
- count: per-open-count integer.
- name: per-display name.
- link: per-pty peer (NULL if not pty).
- fasync: per-fasync_struct list head.
- flags: TTY_THROTTLED, TTY_IO_ERROR, TTY_OTHER_CLOSED, TTY_EXCLUSIVE, TTY_DEBUG, TTY_DO_WRITE_WAKEUP, TTY_LDISC_OPEN, TTY_PTY_LOCK, TTY_NO_WRITE_SPLIT, TTY_HUPPED, TTY_HUPPING, TTY_LDISC_HALTED, TTY_LDISC_CHANGING.
- write_wait / read_wait: per-waitqueue.
- write_buf / write_cnt: per-iterate_tty_write staging buffer.
- hangup_work: per-do_tty_hangup deferred.
- SAK_work: per-do_SAK_work deferred.
- tty_files: per-tty_file_private list.
- files_lock: per-tty_files spinlock.
- disc_data: per-ldisc private (e.g. struct n_tty_data).
- closing: per-closing flag.

REQ-2: struct tty_driver:
- magic: TTY_DRIVER_MAGIC.
- kref: per-refcount.
- cdevs[num]: per-index cdev (when TTY_DRIVER_DYNAMIC_ALLOC).
- owner: per-module.
- driver_name / name: per-strings.
- name_base: per-minor base.
- major / minor_start / num: per-chrdev region.
- type / subtype: TTY_DRIVER_TYPE_{SYSTEM,CONSOLE,SERIAL,PTY,SCC,SYSCONS} × PTY_TYPE_{MASTER,SLAVE}.
- init_termios: per-default termios at install.
- flags: TTY_DRIVER_INSTALLED, _RESET_TERMIOS, _REAL_RAW, _DYNAMIC_DEV, _DYNAMIC_ALLOC, _UNNUMBERED_NODE, _NO_WORKQUEUE.
- proc_entry: per-/proc/tty/driver entry.
- other: per-pty link to peer driver.
- ttys[idx] / termios[idx] / ports[idx]: per-instance pointer tables.
- ops: per-tty_operations.
- flip_wq: per-driver flip workqueue (unless _NO_WORKQUEUE).
- tty_drivers: per-global list link (under tty_mutex).

REQ-3: struct tty_operations:
- lookup(driver, file, idx) → tty_struct*: per-driver lookup.
- install(driver, tty) → int: per-driver tty table install.
- remove(driver, tty): per-driver tty table remove.
- open(tty, filp) → int: per-driver open.
- close(tty, filp): per-driver close.
- shutdown(tty): per-final-close shutdown.
- cleanup(tty): per-async post-shutdown.
- write(tty, buf, count) → ssize_t: per-driver write.
- put_char(tty, ch) → int: per-driver single-char.
- flush_chars(tty): per-driver flush-pending.
- write_room(tty) → unsigned int: per-driver TX-free.
- chars_in_buffer(tty) → unsigned int: per-driver TX-pending.
- ioctl(tty, cmd, arg) → int: per-driver ioctl.
- compat_ioctl(tty, cmd, arg) → long: per-32on64.
- set_termios(tty, old): per-termios apply.
- throttle / unthrottle: per-RX backpressure.
- stop / start: per-flow control.
- hangup(tty): per-hardware hangup.
- break_ctl(tty, state) → int: per-break.
- flush_buffer(tty): per-flush input buffer.
- set_ldisc(tty): per-ldisc change hook.
- wait_until_sent(tty, timeout): per-drain.
- send_xchar(tty, ch): per-XON/XOFF emit.
- tiocmget / tiocmset: per-modem-line.
- resize(tty, ws) → int: per-TIOCSWINSZ hook.
- get_icount / get_serial / set_serial / poll_init / poll_get_char / poll_put_char: per-extension.

REQ-4: file_operations:
- tty_fops = { read_iter=tty_read, write_iter=tty_write, splice_read=copy_splice_read, splice_write=iter_file_splice_write, poll=tty_poll, unlocked_ioctl=tty_ioctl, compat_ioctl=tty_compat_ioctl, open=tty_open, release=tty_release, fasync=tty_fasync, show_fdinfo=tty_show_fdinfo }.
- console_fops = same but write_iter=redirected_tty_write (TIOCCONS-aware).
- hung_up_tty_fops = { read_iter=hung_up_tty_read (=> -EIO), write_iter=hung_up_tty_write (=> -EIO), poll=hung_up_tty_poll (=> EPOLLHUP), unlocked_ioctl=hung_up_tty_ioctl (=> -EIO), compat_ioctl=hung_up_tty_compat_ioctl, release=tty_release, fasync=hung_up_tty_fasync }.
- All marked nonseekable_open(): no llseek.

REQ-5: tty_open(inode, filp):
- nonseekable_open(inode, filp).
- retry_open: tty_alloc_file(filp) — allocate struct tty_file_private linkage.
- /* /dev/tty current task */
- tty = tty_open_current_tty(device, filp).
- /* else dispatch by major */
- if !tty: tty = tty_open_by_driver(device, filp).
- if IS_ERR(tty):
  - tty_free_file(filp).
  - if retval == -EAGAIN ∧ !signal_pending: schedule; goto retry_open.
  - return retval.
- tty_add_file(tty, filp).
- check_tty_count(tty, __func__).
- if tty.ops.open: retval = tty.ops.open(tty, filp); else -ENODEV.
- on error: tty_unlock; tty_release; if ERESTARTSYS ∧ !signal_pending: retry.
- clear_bit(TTY_HUPPED, &tty.flags).
- noctty = O_NOCTTY ∨ device==MKDEV(TTY_MAJOR,0) ∨ device==MKDEV(TTYAUX_MAJOR,1) ∨ pty-master.
- if !noctty: tty_open_proc_set_tty(filp, tty).
- tty_unlock(tty); return 0.

REQ-6: tty_init_dev(driver, idx):
- try_module_get(driver.owner) ∨ return -ENODEV.
- tty = alloc_tty_struct(driver, idx) ∨ -ENOMEM.
- tty_lock(tty).
- retval = tty_driver_install_tty(driver, tty) — calls driver.ops.install (default tty_standard_install which calls tty_init_termios + tty.driver.ttys[idx] = tty).
- tty.port ||= driver.ports[idx] (WARN if NULL).
- retval = tty_ldisc_lock(tty, 5*HZ).
- tty.port.itty = tty.
- retval = tty_ldisc_setup(tty, tty.link) — ldisc open + receive room setup.
- tty_ldisc_unlock(tty).
- return tty (locked).

REQ-7: tty_open_current_tty(device, filp):
- if device != MKDEV(TTYAUX_MAJOR, 0): return NULL.
- tty = get_current_tty() under tasklist_lock.
- if !tty: return ERR_PTR(-ENXIO).
- retval = tty_reopen(tty) — increment count; tty_ldisc_setup if 0→1.
- on success: tty_lock(tty); tty_kref_put(tty); return tty.

REQ-8: tty_open_by_driver(device, filp):
- driver = get_tty_driver(device, &index).
- if !driver: return ERR_PTR(-ENODEV).
- mutex_lock(&tty_mutex).
- if driver.ops.lookup: tty = driver.ops.lookup(driver, filp, index); else tty_driver_lookup_tty.
- if !IS_ERR_OR_NULL(tty): tty_reopen.
- else: tty = tty_init_dev(driver, index).

REQ-9: tty_release(inode, filp):
- tty = file_tty(filp); paranoia.
- tty_lock(tty).
- __tty_fasync(-1, filp, 0).
- idx = tty.index.
- o_tty = (pty-master) ? tty.link : NULL.
- if tty_release_checks: unlock; return 0.
- if tty.ops.close: tty.ops.close(tty, filp).
- tty_lock_slave(o_tty) — stable lock order (master before slave).
- /* Drain waiters if count → 0 */
- while count <= 1 ∧ (waitqueue_active(read_wait) ∨ write_wait): wake; sleep; backoff.
- decrement count fields (warn if negative).
- tty_del_file(filp).
- if !tty.count: read_lock(tasklist_lock); session_clear_tty(ctrl.session); for slave too; unlock.
- final = !tty.count ∧ !(o_tty ∧ o_tty.count).
- tty_unlock_slave; tty_unlock.
- if !final: return 0.
- tty_release_struct(tty, idx).
- return 0.

REQ-10: tty_release_struct(tty, idx):
- tty_ldisc_release(tty) — close ldisc on both sides.
- tty_flush_works(tty).
- release_tty(tty, idx) — clear driver tables; tty_kref_put.

REQ-11: tty_read(iocb, to):
- file = iocb.ki_filp; tty = file_tty(file).
- paranoia_check ∨ -EIO.
- if tty_io_error(tty): -EIO.
- ld = tty_ldisc_ref_wait(tty) — wait for non-changing ldisc.
- if !ld: return hung_up_tty_read.
- ret = -EIO; if ld.ops.read: ret = iterate_tty_read(ld, tty, file, to).
- tty_ldisc_deref(ld).
- if ret > 0: tty_update_time(tty, false).

REQ-12: iterate_tty_read(ld, tty, file, to):
- /* Chunk to PAGE / 2048-byte kernel buffer kbuf */
- kbuf = kbuf_local (4 KiB).
- while iov_iter_count(to) > 0:
  - size = min(count, sizeof(kbuf)).
  - n = ld.ops.read(tty, file, kbuf, size, &cookie, offset).
  - if cookie continuation: loop until cookie cleared.
  - if n < 0: break with error.
  - if !n: break.
  - if copy_to_iter(kbuf, n, to) != n: -EFAULT.
  - written += n.
  - if signal_pending(current): -ERESTARTSYS.

REQ-13: tty_write(iocb, from):
- file = iocb.ki_filp; tty = file_tty.
- paranoia ∨ -EIO.
- ld = tty_ldisc_ref_wait.
- if !ld: hung_up_tty_write.
- ret = iterate_tty_write(ld, tty, file, from).
- tty_ldisc_deref.

REQ-14: iterate_tty_write(ld, tty, file, from):
- tty_write_lock(tty, O_NDELAY) ∨ -EAGAIN / -ERESTARTSYS.
- chunk = 2048 (or 65536 if TTY_NO_WRITE_SPLIT).
- if tty.write_cnt < chunk: kvfree(write_buf); kvmalloc(chunk, GFP_KERNEL|__GFP_RETRY_MAYFAIL); set write_cnt.
- loop:
  - size = min(chunk, count).
  - copy_from_iter(write_buf, size, from) — must equal size or -EFAULT.
  - ret = ld.ops.write(tty, file, write_buf, size).
  - if ret <= 0: break.
  - written += ret.
  - if signal_pending: -ERESTARTSYS.
  - cond_resched().
- if written: tty_update_time(tty, true).
- tty_write_unlock(tty).

REQ-15: tty_poll(filp, wait):
- ld = tty_ldisc_ref_wait.
- if !ld: hung_up_tty_poll (=> EPOLLHUP).
- if ld.ops.poll: ret = ld.ops.poll(tty, filp, wait).
- tty_ldisc_deref(ld).

REQ-16: tty_ioctl(file, cmd, arg):
- tty = file_tty; paranoia ∨ -EINVAL.
- real_tty = tty_pair_get_tty(tty) — slave-side for pty-master ioctls.
- for { TIOCSETD, TIOCSBRK, TIOCCBRK, TCSBRK, TCSBRKP }: tty_check_change; tty_wait_until_sent(tty, 0); if signal_pending: -EINTR.
- Dispatch by cmd:
  - TIOCSTI → tiocsti.
  - TIOCGWINSZ → tiocgwinsz(real_tty).
  - TIOCSWINSZ → tiocswinsz(real_tty).
  - TIOCCONS → tioccons(file) (only on the slave side; -EINVAL on master).
  - TIOCEXCL / TIOCNXCL / TIOCGEXCL → set/clear/test TTY_EXCLUSIVE.
  - TIOCGETD / TIOCSETD → tiocgetd / tiocsetd (TIOCSETD = tty_set_ldisc).
  - TIOCVHANGUP → CAP_SYS_ADMIN; tty_vhangup.
  - TIOCGDEV → new_encode_dev(tty_devnum(real_tty)).
  - TIOCSBRK / TIOCCBRK → tty.ops.break_ctl(tty, -1 / 0).
  - TCSBRK / TCSBRKP → send_break.
  - TIOCMGET / TIOCMSET / TIOCMBIC / TIOCMBIS → tty_tiocmget / _set.
  - TIOCGICOUNT → tty_tiocgicount.
  - TCFLSH (TCIFLUSH / TCIOFLUSH): tty_buffer_flush(tty, NULL).
  - TIOCSSERIAL / TIOCGSERIAL → tty_tiocsserial / _gserial.
  - TIOCGPTPEER → ptm_open_peer(file, tty, arg).
  - default: tty_jobctrl_ioctl(tty, real_tty, file, cmd, arg).
- if -ENOIOCTLCMD: try tty.ops.ioctl(tty, cmd, arg).
- if -ENOIOCTLCMD: ld = tty_ldisc_ref_wait; ld.ops.ioctl(tty, cmd, arg); -ENOTTY if still -ENOIOCTLCMD.

REQ-17: __tty_hangup(tty, exit_session):
- f = tty_release_redirect(tty) — capture and clear TIOCCONS redirect.
- tty_lock(tty).
- if TTY_HUPPED already: unlock; return.
- set_bit(TTY_HUPPING, &flags) — kicks readers out of n_tty_read.
- check_tty_count.
- spin_lock(&files_lock).
- for_each_file (priv ∈ tty.tty_files):
  - if write_iter == redirected_tty_write: cons_filp = filp.
  - if write_iter != tty_write: skip.
  - closecount++; __tty_fasync(-1, filp, 0); filp.f_op = &hung_up_tty_fops.
- spin_unlock(&files_lock).
- refs = tty_signal_session_leader(tty, exit_session) — SIGHUP+SIGCONT to session-leader; -1 if exit_session.
- while refs--: tty_kref_put.
- tty_ldisc_hangup(tty, cons_filp != NULL).
- spin_lock_irq(&ctrl.lock):
  - clear TTY_THROTTLED, TTY_DO_WRITE_WAKEUP.
  - put_pid(ctrl.session); put_pid(ctrl.pgrp); ctrl.session = NULL; ctrl.pgrp = NULL; ctrl.pktstatus = 0.
- spin_unlock_irq.
- if cons_filp ∧ tty.ops.close: for n in 0..closecount: tty.ops.close(tty, cons_filp).
- else if tty.ops.hangup: tty.ops.hangup(tty).
- set TTY_HUPPED; clear TTY_HUPPING.
- tty_unlock(tty); if f: fput(f).

REQ-18: tty_register_driver(driver):
- if !driver.major: alloc_chrdev_region(&dev, minor_start, num, name); driver.major=MAJOR; driver.minor_start=MINOR.
- else: register_chrdev_region(MKDEV(major, minor_start), num, name).
- if !TTY_DRIVER_NO_WORKQUEUE ∧ driver_name:
  - driver.flip_wq = alloc_workqueue("%s-%s", WQ_UNBOUND|WQ_SYSFS, 0, name, driver_name).
  - for i in 0..num: if driver.ports[i]: tty_port_link_driver_wq.
- if TTY_DRIVER_DYNAMIC_ALLOC: tty_cdev_add(driver, dev, 0, num).
- scoped_guard(mutex, &tty_mutex): list_add(&driver.tty_drivers, &tty_drivers).
- if !TTY_DRIVER_DYNAMIC_DEV: for i in 0..num: tty_register_device(driver, i, NULL).
- proc_tty_register_driver(driver).
- driver.flags |= TTY_DRIVER_INSTALLED.
- On err: roll back devs, list_del, destroy_workqueue, unregister_chrdev_region.

REQ-19: tty_unregister_driver(driver):
- unregister_chrdev_region(MKDEV(major, minor_start), num).
- scoped_guard(mutex, &tty_mutex): list_del(&driver.tty_drivers).
- if flip_wq: destroy_workqueue.

REQ-20: tty_init() module init (postcore_initcall on tty_class_init; tty_init proper from chr_dev_init() / per Linux Documentation):
- register_sysctl_init("dev/tty", tty_table) — exposes legacy_tiocsti + ldisc_autoload.
- cdev_init(&tty_cdev, &tty_fops); cdev_add(&tty_cdev, MKDEV(TTYAUX_MAJOR, 0), 1); register_chrdev_region MKDEV(TTYAUX_MAJOR,0). panic on failure.
- device_create(&tty_class, NULL, MKDEV(TTYAUX_MAJOR, 0), NULL, "tty") — /dev/tty.
- cdev_init(&console_cdev, &console_fops); cdev_add MKDEV(TTYAUX_MAJOR, 1); register_chrdev_region. panic on failure.
- consdev = device_create_with_groups(&tty_class, NULL, MKDEV(TTYAUX_MAJOR, 1), NULL, cons_dev_groups, "console").
- if CONFIG_VT: vty_init(&console_fops).

REQ-21: Per-PTY pair semantics:
- pty_master.ops.lookup / install / remove are special-case.
- pty_master.link = pty_slave; pty_slave.link = pty_master.
- master close hangs up slave (TTY_OTHER_CLOSED), and vice versa.
- TIOCSWINSZ on master is forwarded to slave (tty_pair_get_tty).
- TIOCCONS on master is forbidden (-EINVAL).
- TIOCGPTPEER opens the peer side as a new struct file (used by openpty/grantpt+unlockpt).
- tty_release closes both master and slave in stable lock order (master before slave).
- Packet mode (ctrl.packet, ctrl.pktstatus): TIOCPKT_DATA byte + status bits on master read.

REQ-22: Per-line-discipline switch (tty_set_ldisc via TIOCSETD):
- tty_set_ldisc(tty, disc) (defined in tty_ldisc.c, called from tiocsetd):
  - tty_ldisc_lock(tty, 5*HZ).
  - Halt current ldisc (set TTY_LDISC_CHANGING; cancel buffer work).
  - tty_ldisc_close on current; tty_ldisc_open on new.
  - tty_ldisc_unlock.
- tiocsetd: copy_from_user(disc); return tty_set_ldisc(tty, disc).
- tiocgetd: ld = tty_ldisc_ref_wait; copy_to_user ld.ops.num.
- ldisc_autoload sysctl gates automatic request_module("tty-ldisc-%d").

REQ-23: Per-controlling-tty + session/PGID:
- ctrl.session / ctrl.pgrp updated under ctrl.lock.
- tty_open with !O_NOCTTY ∧ session leader ∧ no controlling tty: tty_open_proc_set_tty.
- tioccons (TIOCCONS): redirect printk output to this tty (CAP_SYS_ADMIN; only slave; takes a ref via redirect = filp).
- tty_release_redirect: if redirect == one of tty's filps, return it for fput after hangup; protected by redirect_lock spinlock.
- tty_signal_session_leader: kill_pgrp(session, SIGCONT, 1); kill_pgrp(session, SIGHUP, 1) and SIGCONT to session leader's tty group; clears each task's signal->tty pointer.

REQ-24: Per-flow control:
- __stop_tty(tty): set tty.flow.stopped = true; if tty.ops.stop: tty.ops.stop(tty); under flow.lock.
- stop_tty: locked wrapper.
- __start_tty: clear tty.flow.stopped; if tty.ops.start: tty.ops.start(tty); tty_wakeup(tty).
- tty_send_xchar(tty, ch): if tty.ops.send_xchar: ops.send_xchar(tty, ch); else: tty_write_lock + tty.ops.write(tty, &ch, 1).
- tty_wakeup(tty): if TTY_DO_WRITE_WAKEUP: ld = tty_ldisc_ref(tty); ld.ops.write_wakeup(tty); deref. wake_up_interruptible_poll(&tty.write_wait, EPOLLOUT).

REQ-25: Per-paranoia / accounting:
- tty_paranoia_check(tty, inode, routine): if tty==NULL → -EIO log. if tty.magic != TTY_MAGIC: log; -EIO.
- check_tty_count(tty, routine): walk tty.tty_files; compare against tty.count; log discrepancy.
- tty_release_checks(tty, idx): tty.driver.ttys[idx] != tty → bad; tty.index != idx → bad.

REQ-26: Per-SAK (TIOCSAK / sysrq-k):
- do_SAK(tty): schedule_work(&tty.SAK_work).
- do_SAK_work → __do_SAK(tty): for_each_process(p): scan p.files for tasks holding tty; send SIGKILL.

REQ-27: Per-hung_up_tty_fops:
- All ops return -EIO except poll → EPOLLHUP.
- tty_open detects TTY_HUPPED post-error via tty_hung_up_p (f_op == &hung_up_tty_fops).

REQ-28: Per-tty_legacy_tiocsti sysctl (/proc/sys/dev/tty/legacy_tiocsti):
- Default depends on CONFIG_LEGACY_TIOCSTI.
- If 0: tiocsti requires CAP_SYS_ADMIN.
- If 1: tiocsti allowed when current.signal.tty == tty (legacy behavior).

REQ-29: Per-ldisc_autoload sysctl (/proc/sys/dev/tty/ldisc_autoload):
- {0,1} — 0 disables request_module("tty-ldisc-%d") in tiocsetd path.

REQ-30: Per-/proc/<pid>/fdinfo/<fd>:
- tty_show_fdinfo(m, file): seq_printf "tty:\t%d:%d\n" with MAJOR/MINOR of tty.

## Acceptance Criteria

- [ ] AC-1: tty_open of /dev/ttyN performs tty_init_dev on first open; tty_reopen on subsequent opens (count incremented).
- [ ] AC-2: tty_open of /dev/tty (TTYAUX_MAJOR,0) returns the calling task's controlling tty; -ENXIO when none.
- [ ] AC-3: tty_open of /dev/console (TTYAUX_MAJOR,1) does NOT install a controlling tty; uses console_fops with redirected_tty_write.
- [ ] AC-4: O_NOCTTY suppresses controlling-tty acquisition; otherwise session-leader-without-tty acquires it.
- [ ] AC-5: tty_release final close (master+slave both at 0) tears down ldisc, fires driver.ops.close, calls release_tty, drops kref.
- [ ] AC-6: tty_read forwards to ld.ops.read via iterate_tty_read; copy_to_iter respected; per-signal -ERESTARTSYS.
- [ ] AC-7: tty_write chunks to 2 KiB (or 64 KiB if TTY_NO_WRITE_SPLIT); per atomic_write_lock; ld.ops.write.
- [ ] AC-8: tty_poll forwards to ld.ops.poll; hung_up_tty_poll returns EPOLLHUP for hung-up files.
- [ ] AC-9: tty_ioctl dispatches TIOC* per REQ-16 table; unknown → tty.ops.ioctl → ld.ops.ioctl → -ENOTTY.
- [ ] AC-10: TIOCSETD switches ldisc; tty_ldisc_lock + ldisc close/open; ldisc_autoload gates auto request_module.
- [ ] AC-11: TIOCSTI without CAP_SYS_ADMIN respects tty_legacy_tiocsti and tty == current.signal.tty.
- [ ] AC-12: TIOCCONS (slave only) sets redirect; on hangup tty_release_redirect captures and fput()s after work.
- [ ] AC-13: tty_register_driver: alloc_chrdev_region(major==0) or register_chrdev_region; install cdevs (dyn) and per-device entries (static).
- [ ] AC-14: tty_unregister_driver: unregister_chrdev_region, destroy_workqueue, list_del under tty_mutex.
- [ ] AC-15: tty_hangup: SIGHUP to session-leader via tty_signal_session_leader; swap fop to hung_up_tty_fops; ctrl.session/pgrp cleared; TTY_HUPPED set.
- [ ] AC-16: vhangup (TIOCVHANGUP, CAP_SYS_ADMIN): forces hangup synchronously.
- [ ] AC-17: PTY pair: master close → slave sees TTY_OTHER_CLOSED + EPOLLHUP; TIOCGPTPEER opens peer; TIOCCONS on master → -EINVAL.
- [ ] AC-18: tty_paranoia_check rejects tty.magic mismatch; tty_release_checks validates idx + ttys[] slot.

## Architecture

```
struct TtyStruct {
  magic: u32,                              // TTY_MAGIC
  kref: KRef,
  dev: *Device,
  driver: *TtyDriver,
  ops: *const TtyOperations,
  index: i32,
  ldisc_sem: LdSem,
  ldisc: Option<*TtyLdisc>,
  legacy_mutex: Mutex,                     // tty_lock
  throttle_mutex: Mutex,
  termios_rwsem: RwSem,
  termios: Termios,
  termios_locked: Termios,
  winsize_mutex: Mutex,
  winsize: WinSize,
  flow: Flow,                              // { lock: Spin, stopped, tco_stopped }
  ctrl: Ctrl,                              // { lock: Spin, session, pgrp, pktstatus, packet }
  atomic_write_lock: Mutex,
  legacy_port: Option<*TtyPort>,
  port: Option<*TtyPort>,
  count: i32,
  name: [u8; 64],
  link: Option<*TtyStruct>,                // pty pair
  fasync: Option<*FasyncStruct>,
  flags: u64,
  write_wait: WaitQueue,
  read_wait: WaitQueue,
  write_buf: *mut u8,
  write_cnt: usize,
  hangup_work: Work,
  SAK_work: Work,
  tty_files: ListHead,
  files_lock: SpinLock,
  disc_data: *mut c_void,                  // ldisc-private
  closing: u8,
}

struct TtyDriver {
  magic: u32,                              // TTY_DRIVER_MAGIC
  kref: KRef,
  cdevs: *mut *mut Cdev,
  owner: *Module,
  driver_name: *const u8,
  name: *const u8,
  name_base: i32,
  major: u32,
  minor_start: u32,
  num: u32,
  type_: u16, subtype: u16,
  init_termios: Termios,
  flags: u32,
  proc_entry: Option<*ProcDirEntry>,
  other: Option<*TtyDriver>,               // pty link
  ttys: *mut *mut TtyStruct,
  termios: *mut *mut Termios,
  ports: *mut *mut TtyPort,
  ops: *const TtyOperations,
  flip_wq: Option<*Workqueue>,
  tty_drivers: ListHead,
}

struct TtyOperations {
  lookup: Option<fn(&TtyDriver, &File, i32) -> *mut TtyStruct>,
  install: Option<fn(&TtyDriver, &mut TtyStruct) -> i32>,
  remove: Option<fn(&TtyDriver, &mut TtyStruct)>,
  open: Option<fn(&mut TtyStruct, &File) -> i32>,
  close: Option<fn(&mut TtyStruct, &File)>,
  shutdown: Option<fn(&mut TtyStruct)>,
  cleanup: Option<fn(&mut TtyStruct)>,
  write: Option<fn(&mut TtyStruct, &[u8]) -> isize>,
  put_char: Option<fn(&mut TtyStruct, u8) -> i32>,
  flush_chars: Option<fn(&mut TtyStruct)>,
  write_room: Option<fn(&TtyStruct) -> u32>,
  chars_in_buffer: Option<fn(&TtyStruct) -> u32>,
  ioctl: Option<fn(&mut TtyStruct, u32, usize) -> i32>,
  compat_ioctl: Option<fn(&mut TtyStruct, u32, usize) -> isize>,
  set_termios: Option<fn(&mut TtyStruct, &Termios)>,
  throttle: Option<fn(&mut TtyStruct)>,
  unthrottle: Option<fn(&mut TtyStruct)>,
  stop: Option<fn(&mut TtyStruct)>,
  start: Option<fn(&mut TtyStruct)>,
  hangup: Option<fn(&mut TtyStruct)>,
  break_ctl: Option<fn(&mut TtyStruct, i32) -> i32>,
  flush_buffer: Option<fn(&mut TtyStruct)>,
  set_ldisc: Option<fn(&mut TtyStruct)>,
  wait_until_sent: Option<fn(&mut TtyStruct, i32)>,
  send_xchar: Option<fn(&mut TtyStruct, u8)>,
  tiocmget: Option<fn(&TtyStruct) -> i32>,
  tiocmset: Option<fn(&mut TtyStruct, u32, u32) -> i32>,
  resize: Option<fn(&mut TtyStruct, &WinSize) -> i32>,
  get_icount: Option<fn(&TtyStruct, &mut SerialIcounter) -> i32>,
  get_serial: Option<fn(&TtyStruct, &mut SerialStruct) -> i32>,
  set_serial: Option<fn(&mut TtyStruct, &SerialStruct) -> i32>,
  // ... poll_init / poll_get_char / poll_put_char ...
}
```

`Tty::open(inode, filp) -> i32`:
1. nonseekable_open(inode, filp).
2. loop /* retry_open */:
   - tty_alloc_file(filp).
   - tty = open_current_tty(device, filp).
   - tty ||= open_by_driver(device, filp).
   - if Err(EAGAIN) ∧ !signal_pending: schedule(); continue.
3. tty_add_file(tty, filp).
4. check_tty_count(tty, "open").
5. retval = tty.ops.open.unwrap_or(default)(tty, filp).
6. if retval:
   - tty_unlock(tty); Tty::release(inode, filp).
   - if retval == -ERESTARTSYS ∧ !signal_pending: continue.
   - return retval.
7. clear TTY_HUPPED.
8. noctty = O_NOCTTY ∨ device==MKDEV(TTY_MAJOR,0) ∨ device==MKDEV(TTYAUX_MAJOR,1) ∨ pty-master.
9. if !noctty: Tty::open_proc_set_tty(filp, tty).
10. tty_unlock(tty); return 0.

`Tty::release(inode, filp) -> i32`:
1. tty = file_tty(filp); paranoia.
2. tty_lock(tty); check_tty_count.
3. __tty_fasync(-1, filp, 0).
4. idx = tty.index.
5. o_tty = (tty is pty-master) ? tty.link : NULL.
6. if Tty::release_checks(tty, idx): tty_unlock; return 0.
7. tty.ops.close.map(|f| f(tty, filp)).
8. tty_lock_slave(o_tty).
9. /* Drain waiters when count → 0 */
10. loop with exponential timeout: wake_up_poll(read_wait, EPOLLIN); wake_up_poll(write_wait, EPOLLOUT); schedule_timeout_killable; break if no active waiters.
11. decrement o_tty.count and tty.count.
12. tty_del_file(filp).
13. if !tty.count: tasklist read_lock; session_clear_tty for both; unlock.
14. final = !tty.count ∧ !(o_tty ∧ o_tty.count).
15. tty_unlock_slave; tty_unlock.
16. if !final: return 0.
17. Tty::release_inner(tty, idx):
    - tty_ldisc_release.
    - tty_flush_works.
    - release_tty (clear driver.ttys[idx], decrement kref).
18. return 0.

`Tty::read(iocb, to) -> isize`:
1. file = iocb.file; tty = file_tty(file); paranoia ∨ -EIO.
2. if tty_io_error(tty): -EIO.
3. ld = tty_ldisc_ref_wait(tty); if !ld: hung_up_tty_read => -EIO.
4. ret = ld.ops.read.map(|_| Tty::iterate_read(ld, tty, file, to)).unwrap_or(-EIO).
5. tty_ldisc_deref(ld).
6. if ret > 0: tty_update_time(tty, false).
7. return ret.

`Tty::iterate_read(ld, tty, file, to) -> isize`:
1. kbuf = stack/2 KiB.
2. while iov_iter_count(to) > 0:
   - size = min(remain, sizeof(kbuf)).
   - n = ld.ops.read(tty, file, kbuf, size, &cookie, offset).
   - if cookie continuation: retry loop without unlocking ldisc.
   - if n <= 0: break.
   - copy_to_iter(kbuf, n, to) == n  ∨ -EFAULT.
   - written += n.
   - signal_pending ∨ break -ERESTARTSYS.
3. return written ? written : err.

`Tty::write(iocb, from) -> isize`:
1. file = iocb.file; tty = file_tty.
2. paranoia ∨ -EIO.
3. ld = tty_ldisc_ref_wait; if !ld: hung_up_tty_write.
4. ret = Tty::iterate_write(ld, tty, file, from).
5. tty_ldisc_deref(ld).

`Tty::iterate_write(ld, tty, file, from) -> isize`:
1. tty_write_lock(tty, O_NDELAY).
2. chunk = TTY_NO_WRITE_SPLIT ? 65536 : 2048.
3. if tty.write_cnt < chunk: realloc tty.write_buf to chunk (kvmalloc GFP_KERNEL | __GFP_RETRY_MAYFAIL).
4. loop:
   - size = min(chunk, count).
   - copy_from_iter(write_buf, size, from) ∨ -EFAULT.
   - n = ld.ops.write(tty, file, write_buf, size).
   - if n <= 0: break.
   - if n != size: iov_iter_revert(from, size-n).
   - count -= n; signal_pending ∨ break -ERESTARTSYS.
   - cond_resched().
5. if written: tty_update_time(tty, true).
6. tty_write_unlock(tty).

`Tty::poll(filp, wait) -> u32`:
1. tty = file_tty; paranoia ∨ 0.
2. ld = tty_ldisc_ref_wait; if !ld: hung_up_tty_poll => EPOLLHUP.
3. mask = ld.ops.poll(tty, filp, wait).
4. tty_ldisc_deref.
5. return mask.

`Tty::ioctl(file, cmd, arg) -> i32`:
1. tty = file_tty; paranoia ∨ -EINVAL.
2. real_tty = Tty::pair_get(tty).
3. for prep-cmds (TIOCSETD, TIOCSBRK, TIOCCBRK, TCSBRK, TCSBRKP):
   - tty_check_change(tty).
   - if cmd != TIOCCBRK: tty_wait_until_sent(tty, 0); signal_pending ∨ -EINTR.
4. switch cmd: per REQ-16.
5. fallthrough to tty.ops.ioctl; -ENOIOCTLCMD → ld.ops.ioctl; -ENOIOCTLCMD → -ENOTTY.

`Tty::set_ldisc(tty, disc) -> i32`:
1. tty_ldisc_lock(tty, 5*HZ).
2. set TTY_LDISC_CHANGING, cancel flip_wq.
3. tty_ldisc_close(tty, current).
4. tty_ldisc_open(tty, new) — may request_module("tty-ldisc-%d") gated by ldisc_autoload.
5. tty_ldisc_unlock.
6. clear TTY_LDISC_CHANGING.
7. return 0 / err.

`Tty::hangup_inner(tty, exit_session)`:
1. f = Tty::release_redirect(tty).
2. tty_lock(tty); if TTY_HUPPED: unlock; return.
3. set TTY_HUPPING.
4. spin_lock(&files_lock): swap each tty_file's f_op to hung_up_tty_fops (or capture cons_filp).
5. spin_unlock.
6. refs = Tty::signal_session_leader(tty, exit_session).
7. for refs: tty_kref_put.
8. tty_ldisc_hangup(tty, cons_filp.is_some()).
9. spin_lock_irq(&ctrl.lock): clear TTY_THROTTLED, TTY_DO_WRITE_WAKEUP; put_pid+null session/pgrp; pktstatus=0.
10. if cons_filp: for n in 0..closecount: tty.ops.close(tty, cons_filp). else: tty.ops.hangup(tty).
11. set TTY_HUPPED; clear TTY_HUPPING; tty_unlock.
12. f.map(fput).

`Tty::register_driver(driver) -> i32`:
1. dev = if driver.major == 0: alloc_chrdev_region else register_chrdev_region.
2. if !TTY_DRIVER_NO_WORKQUEUE ∧ driver_name: alloc workqueue + per-port link.
3. if TTY_DRIVER_DYNAMIC_ALLOC: Tty::cdev_add(driver, dev, 0, num).
4. scoped tty_mutex: list_add(&driver.tty_drivers, &tty_drivers).
5. if !TTY_DRIVER_DYNAMIC_DEV: for i in 0..num: Tty::register_device(driver, i, NULL).
6. proc_tty_register_driver.
7. driver.flags |= TTY_DRIVER_INSTALLED.
8. unwind on error.

`Tty::init()`:
1. register_sysctl_init("dev/tty", TTY_TABLE).
2. /dev/tty: cdev_init(&tty_cdev, &tty_fops); cdev_add MKDEV(TTYAUX_MAJOR,0); register_chrdev_region. panic on failure.
3. device_create(&tty_class, MKDEV(TTYAUX_MAJOR,0), "tty").
4. /dev/console: cdev_init(&console_cdev, &console_fops); cdev_add MKDEV(TTYAUX_MAJOR,1); register_chrdev_region. panic on failure.
5. device_create_with_groups(&tty_class, MKDEV(TTYAUX_MAJOR,1), cons_dev_groups, "console").
6. if CONFIG_VT: vty_init(&console_fops).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_paranoia_check` | INVARIANT | per-tty entry: tty.magic == TTY_MAGIC. |
| `kref_balanced_per_open` | INVARIANT | per-Tty::open success: kref += 1; per-release final: kref -= 1. |
| `count_matches_files` | INVARIANT | per-check_tty_count: count == |tty_files for tty|. |
| `legacy_mutex_held_in_release` | INVARIANT | per-Tty::release: tty_lock(tty) held throughout state mutation. |
| `slave_locked_after_master` | INVARIANT | per-release: tty_lock_slave(o_tty) only after tty_lock(tty). |
| `ldisc_ref_balanced` | INVARIANT | per-tty_ldisc_ref_wait: matched with tty_ldisc_deref. |
| `atomic_write_lock_held_in_write` | INVARIANT | per-iterate_write: atomic_write_lock held across copy_from_iter + ld.ops.write. |
| `tty_files_consistent` | INVARIANT | per-tty_add_file / tty_del_file: tty_files list count matches. |
| `redirect_capped_to_one` | INVARIANT | per-tioccons: only one redirect file globally under redirect_lock. |
| `hupped_swaps_fop` | INVARIANT | per-__tty_hangup: every priv.file with f_op == &tty_write gets &hung_up_tty_fops. |
| `noctty_for_dev_console` | INVARIANT | per-Tty::open: device==MKDEV(TTYAUX_MAJOR,1) ⟹ noctty == true. |
| `pty_master_no_ctty` | INVARIANT | per-Tty::open: pty-master ⟹ noctty == true. |

### Layer 2: TLA+

`drivers/tty/tty_io.tla`:
- Per-open + per-close + per-hangup + per-ldisc-switch + per-pty-pair.
- Properties:
  - `safety_kref_never_negative` — per-release: never decrement below 0.
  - `safety_count_never_negative` — per-release: count clamps at 0 (warn).
  - `safety_one_redirect_at_a_time` — per-TIOCCONS: redirect updates serialized under redirect_lock.
  - `safety_hangup_idempotent` — per-__tty_hangup: TTY_HUPPED set ⟹ second call is no-op.
  - `safety_pty_master_release_before_slave` — per-release pair: master tty_lock taken before slave.
  - `safety_ldisc_change_excludes_io` — per-tty_set_ldisc: TTY_LDISC_CHANGING ⟹ no concurrent ld.ops.read / .write.
  - `liveness_open_eventually_resolves` — per-Tty::open: returns within bounded retries.
  - `liveness_release_drains_waiters` — per-Tty::release: read_wait / write_wait drained within timeout.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tty::open` post: success ⟹ tty.count >= 1 ∧ filp in tty.tty_files | `Tty::open` |
| `Tty::release` post: final ⟹ tty.count == 0 ∧ tty removed from driver.ttys[idx] | `Tty::release` |
| `Tty::iterate_read` post: ret > 0 ⟹ bytes copied to iov_iter | `Tty::iterate_read` |
| `Tty::iterate_write` post: holds atomic_write_lock for full duration | `Tty::iterate_write` |
| `Tty::ioctl` post: returns -ENOTTY iff all dispatch layers returned -ENOIOCTLCMD | `Tty::ioctl` |
| `Tty::set_ldisc` post: success ⟹ tty.ldisc.ops.num == disc | `Tty::set_ldisc` |
| `Tty::hangup_inner` post: TTY_HUPPED set ∧ ctrl.session == NULL ∧ ctrl.pgrp == NULL | `Tty::hangup_inner` |
| `Tty::register_driver` post: list_contains(&tty_drivers, &driver.tty_drivers) ∧ flags & TTY_DRIVER_INSTALLED | `Tty::register_driver` |
| `Tty::unregister_driver` post: chrdev region released ∧ flip_wq destroyed | `Tty::unregister_driver` |
| `Tty::send_xchar` post: ops.send_xchar fired or tty.ops.write(&ch, 1) under tty_write_lock | `Tty::send_xchar` |

### Layer 4: Verus/Creusot functional

`Per-fd open → tty_init_dev → ops.open → io → release → hangup → final destroy → kref` semantic equivalence: per-Documentation/driver-api/tty/. PTY pair (master ↔ slave), controlling-tty acquisition (O_NOCTTY / session-leader), and SIGHUP-on-hangup model verified against POSIX.1-2008 (`open`, `tcgetsid`, `tcsetpgrp`, `vhangup`). Console-redirect (TIOCCONS) → printk routing equivalence: per-kernel/printk/printk.c.

## Hardening

(Inherits row-1 features from `drivers/tty/00-overview.md` § Hardening.)

TTY core reinforcement:

- **Per-tty_paranoia_check (magic + null)** — defense against per-stale-fd dispatch and per-driver-bug NULL-tty.
- **Per-tty_release_checks (idx + ttys[] slot)** — defense against per-double-free of tty across driver-bug paths.
- **Per-atomic_write_lock** — defense against per-overlapping userspace write tearing.
- **Per-tty_write_lock interruptible** — defense against per-deadlock from held mutex during signal-pending exit.
- **Per-ldsem ldisc_sem** — defense against per-use-after-free of ldisc during TIOCSETD churn.
- **Per-TTY_LDISC_CHANGING ⟹ I/O drained** — defense against per-receive_buf into half-installed ldisc.
- **Per-TTY_HUPPING reader-abort** — defense against per-indefinite n_tty_read on consoles that never truly hang up.
- **Per-hung_up_tty_fops swap** — defense against per-use-after-hangup driver call.
- **Per-master-then-slave lock order** — defense against per-pty deadlock.
- **Per-O_NOCTTY enforcement** — defense against per-untrusted-fd controlling-tty hijack.
- **Per-CAP_SYS_ADMIN gating (TIOCVHANGUP, TIOCCONS, TIOCSTI when !legacy)** — defense against per-unprivileged session-kill / printk-redirect / fake-input.
- **Per-TIOCCONS master-side -EINVAL** — defense against per-pty-master grabbing printk.
- **Per-redirect_lock spinlock + tty_release_redirect** — defense against per-stale-file-pointer in console redirect.
- **Per-tty_signal_session_leader idempotent** — defense against per-double-SIGHUP send.
- **Per-tty_write chunked to 2 KiB** — defense against per-DoS via huge writev pinning atomic_write_lock.
- **Per-kvmalloc __GFP_RETRY_MAYFAIL for write_buf** — defense against per-OOM-during-write fault.
- **Per-tty_mutex around tty_drivers list** — defense against per-list corruption under parallel register/unregister.
- **Per-flip_wq workqueue scoped per-driver** — defense against per-cross-driver work-bleed.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `tty_write` / `tty_read` user buffer access goes through bounded copy helpers; `write_buf` allocation length verified against `count`; defends against per-`tty->write_buf` overflow.
- **PAX_KERNEXEC** — `tty_driver`, `tty_operations`, and the per-driver `ops` tables placed in `__ro_after_init`; defends tty dispatch against driver-vtable rewrite.
- **PAX_RANDKSTACK** — entropy added on every `tty_open` / `tty_release` entry; neutralises stack-shape probing under fast open/close churn.
- **PAX_REFCOUNT** — `tty_struct.count`, `tty_port.count`, and `tty_driver.refcount` use saturating counters.
- **PAX_MEMORY_SANITIZE** — `tty_buffer` flip buffers and `write_buf` allocations zero-on-free; defends against stale keystroke / serial-line data bleeding across tty grants.
- **PAX_UDEREF** — `TIOC*` ioctls (`TIOCSTI`, `TIOCSCTTY`, `TIOCSETD`, `TIOCSWINSZ`) dereference user pointers only via user-AS-annotated copy helpers.
- **PAX_RAP / kCFI** — `tty_operations` callbacks (`open`, `close`, `write`, `ioctl`, `set_termios`, `hangup`) type-tagged; defends against ROP through a swapped driver pointer.
- **GRKERNSEC_HIDESYM** — `/proc/tty/drivers`, `/sys/class/tty/*` listings stripped of kernel pointers.
- **GRKERNSEC_DMESG** — tty hangup / driver-unregister prints gated behind `CAP_SYSLOG`.
- **/dev/tty CAP_SYS_ADMIN for set_ldisc** — `TIOCSETD` (set line discipline) gated behind `CAP_SYS_ADMIN`; closes the historic ldisc-pointer-swap LPE class (CVE-2020-25656 / CVE-2017-2636 family).
- **Controlling-tty session enforcement** — `TIOCSCTTY` enforces "session leader" + "no current controlling tty" rule and requires `CAP_SYS_ADMIN` for `TIOCSCTTY_FORCE`; defends against cross-session tty hijack.
- **Rationale** — `tty-io` is the dispatch core that ldisc-swap LPEs and TIOCSTI injection attacks target; combining CAP_SYS_ADMIN on TIOCSETD, controlling-tty enforcement, and RAP on tty_operations closes the entire historic tty-LPE family.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/tty/n_tty.c` (covered in `n-tty.md` Tier-3)
- `drivers/tty/tty_ldisc.c` (covered separately if expanded)
- `drivers/tty/tty_buffer.c` flip buffers (covered separately)
- `drivers/tty/tty_port.c` (covered separately)
- `drivers/tty/tty_jobctrl.c` (covered separately)
- `drivers/tty/pty.c` (covered separately)
- `drivers/tty/vt/*` (virtual terminal; covered separately)
- `drivers/tty/serial/*` UART drivers (covered separately)
- Implementation code
