# Tier-2: drivers/tty — TTY layer (tty core + line disciplines + serial + virtual terminal + pty + sysrq + serdev)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/tty/
  - include/linux/tty.h
  - include/uapi/asm-generic/termios.h
-->

## Summary

Tier-2 wrapper for the TTY (Teletype) layer — every console kernel-printk lands on, every shell session, every serial-console boot output, every USB-serial dongle's `/dev/ttyUSB<N>`, every PTY pair behind ssh/screen/tmux, every virtual terminal `tty<N>` on a Linux-on-bare-metal box, every Bluetooth-modem RFCOMM TTY, every SoC peripheral on a UART. Components: **TTY core** (`tty_io.c`, `tty_baudrate.c`, `tty_buffer.c`, `tty_ldisc.c`, `tty_ldsem.c`, `tty_mutex.c`, `tty_port.c`, `tty_jobctrl.c`, `tty_audit.c`, `tty.h`: `tty_struct` + `tty_driver` + `tty_port` + line-discipline dispatch + job-control + audit), **line disciplines** (`n_tty.c` — canonical/cooked default; `n_hdlc.c` — HDLC framing; `n_gsm.c` — 3GPP TS 27.010 mux for cellular modems; `n_null.c` — null discipline), **PTY** (`pty.c`: `/dev/ptmx` + `/dev/pts/<N>` Unix98 + legacy BSD), **virtual terminal** (`vt/`: `vt.c` console-driver + `vt_ioctl.c` IOCTLs + `keyboard.c` console keymap consumer + `vc_screen.c` `/dev/vcs<N>` + `/dev/vcsa<N>` snapshot + `consolemap*.c` UTF-8↔Unicode + `vt_glyph.c`), **sysrq** (`sysrq.c`: SysRq magic-key handler), **serdev** (`serdev/`: serial-device-bus framework — modern way to attach Bluetooth/GPS/cellular devices via UART without claiming the TTY itself), **serial** (`serial/`: 8250 family + per-vendor 16550-clone drivers + USB-serial chassis); the bulk of `serial/` is hundreds of per-controller drivers — the in-tree Tier-3s focus on the dominant ones (8250-pci, 8250-pnp, 8250-port, max3100, sc16is7xx, msm-geni, etc.). **Hardware modems** (`hvc/` — paravirt console HVC for Hyper-V/PowerVM/IBM-z; many are out of v0 except for hvc_xen for Xen guest console).

## Compatibility contract — outline

- TTY chardev IOCTLs byte-identical: `TCGETS`, `TCSETS`, `TCSETSW`, `TCSETSF`, `TCSETSF2`, `TCSETSW2`, `TIOCSWINSZ`, `TIOCGWINSZ`, `TIOCSCTTY`, `TIOCNOTTY`, `TIOCGPGRP`, `TIOCSPGRP`, `TIOCGSID`, `TCXONC`, `TCFLSH`, `TIOCEXCL`, `TIOCNXCL`, `TIOCSTI` (root-only — gates input injection), `TIOCMGET`, `TIOCMBIS`, `TIOCMBIC`, `TIOCMSET`, `TIOCGSOFTCAR`, `TIOCSSOFTCAR`, `FIONREAD`/`TIOCINQ`, `TIOCLINUX`, `TIOCCONS`, `TIOCGSERIAL`, `TIOCSSERIAL`, `TIOCPKT`, `FIONBIO`, `TIOCNOTTY`, `TIOCSETD`, `TIOCGETD`, `TCSBRK`, `TCSBRKP`, `TIOCSBRK`, `TIOCCBRK`, `TIOCGSID`, `TCGETX`, `TCSETX`, `TCSETXF`, `TCSETXW`, `TIOCMIWAIT`, `TIOCGICOUNT`, `TIOCGRS485`, `TIOCSRS485`, `TIOCGPTN`, `TIOCSPTLCK`, `TIOCGDEV`, `TCFLSH`, `TIOCSERCONFIG`, `TIOCSERGWILD`, `TIOCSERSWILD`, `TIOCGLCKTRMIOS`, `TIOCSLCKTRMIOS`, `TIOCSERGSTRUCT`, `TIOCSERGETLSR`, `TIOCSERGETMULTI`, `TIOCSERSETMULTI`.
- termios struct + cflag/iflag/oflag/lflag bitmaps byte-identical (every shell + every termcap consumer).
- `/dev/console` + `/dev/tty<N>` + `/dev/ttyS<N>` + `/dev/ttyUSB<N>` + `/dev/pts/<N>` + `/dev/ptmx` chardev naming + minor numbering preserved.
- Console + serial-console redirect via `console=tty<N>` and `console=ttyS<N>,115200n8` cmdline parsed identically.
- VT IOCTLs (`KDSKBMODE`, `KDSKBSENT`, `KDGKBENT`, `KDSKBENT`, `KDGKBLED`, `KDSKBLED`, `KDGETLED`, `KDSETLED`, `KDSIGACCEPT`, `VT_OPENQRY`, `VT_GETMODE`, `VT_SETMODE`, `VT_GETSTATE`, `VT_RELDISP`, `VT_ACTIVATE`, `VT_WAITACTIVE`, `VT_DISALLOCATE`, `VT_RESIZE`, `VT_RESIZEX`, `VT_LOCKSWITCH`, `VT_UNLOCKSWITCH`, `VT_GETHIFONTMASK`) byte-identical.
- SysRq magic-key combos preserved (Alt+SysRq+{b,c,e,f,h,i,k,l,m,n,o,p,q,r,s,t,u,w,x,z,0-9}). `/proc/sys/kernel/sysrq` mask byte-identical.
- Serdev serdev_device + serdev_controller framework source-compat for in-tree consumers (Bluetooth `hci_serdev.c` + GPS `gnss/`).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/tty/tty-core.md` | `tty_io.c` + `tty_buffer.c` + `tty_port.c` + `tty_mutex.c` + `tty_ldsem.c` + `tty_jobctrl.c` + `tty_audit.c`: TTY core lifecycle |
| `drivers/tty/tty-ldisc.md` | `tty_ldisc.c` + `n_tty.c` + `n_hdlc.c` + `n_gsm.c` + `n_null.c`: line discipline framework + impls |
| `drivers/tty/pty.md` | `pty.c`: `/dev/ptmx` + `/dev/pts/` Unix98 PTY |
| `drivers/tty/vt-core.md` | `vt/vt.c` + `vt/vt_ioctl.c` + `vt/keyboard.c` + `vt/consolemap*.c` + `vt/vc_screen.c`: virtual terminal + console keymap + `/dev/vcs<N>` |
| `drivers/tty/sysrq.md` | `sysrq.c`: SysRq magic-key handler |
| `drivers/tty/serial-core.md` | `serial/serial_core.c` + `serial/serial_base*.c`: serial-driver core |
| `drivers/tty/serial-8250.md` | `serial/8250/*`: 8250/16550 family (the dominant x86 UART) |
| `drivers/tty/serial-pci.md` | `serial/8250_pci*.c`: PCI UART cards (multi-port + USB-serial-via-PCI) |
| `drivers/tty/serial-misc.md` | `serial/{altera_uart, atmel_serial, mxs-auart, sc16is7xx, max3100, max310x, sccnxp, ucc_uart, ...}.c`: misc per-controller serial drivers |
| `drivers/tty/serdev.md` | `serdev/`: serial-device-bus framework |
| `drivers/tty/hvc.md` | `hvc/{hvc_console, hvc_xen, hvc_dcc, hvc_iucv, ...}`: hypervisor-console drivers |
| `drivers/tty/console.md` | `console.c` + `vt/consolemap.c`: console driver registration + auto-detect |

## Compatibility outline

- REQ-O1: All TTY IOCTLs + termios struct byte-identical (every shell + every termcap consumer works).
- REQ-O2: PTY Unix98 + legacy BSD wire format byte-identical.
- REQ-O3: VT IOCTLs + console keymap surface byte-identical.
- REQ-O4: SysRq magic-key handler + `/proc/sys/kernel/sysrq` mask byte-identical.
- REQ-O5: Serial-console boot redirect via `console=ttyS<N>,...` cmdline works identically.
- REQ-O6: 8250 family + USB-serial + virtio-console + Xen console enumerate + work.
- REQ-O7: serdev framework source-compat for `hci_serdev` (Bluetooth-via-UART) + `gnss/` (GPS-via-UART) consumers.
- REQ-O8: TLS+ models (per-tty mutex hierarchy, ldisc swap during in-flight read/write, PTY peer-close race, VT switch + concurrent write).
- REQ-O9: Hardening: TIOCSTI input-injection root-gated; per-cgroup tty delegation; serial console kdebug-mode mediated.

## Acceptance Criteria

- [ ] AC-O1: bash + ssh + screen + tmux work over PTY chain.
- [ ] AC-O2: Console boot + serial console mirror via `console=tty0 console=ttyS0,115200n8`.
- [ ] AC-O3: USB-serial dongle (CP210x, FTDI, CH341, PL2303) enumerates as `/dev/ttyUSB<N>`; minicom session works.
- [ ] AC-O4: SysRq Alt+SysRq+s + Alt+SysRq+u + Alt+SysRq+b sequence reboots cleanly.
- [ ] AC-O5: Bluetooth-via-UART HCI brings up `hci<N>` via `hci_serdev`.
- [ ] AC-O6: kselftest `tools/testing/selftests/tty/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/tty/ldisc_swap.tla` | `drivers/tty/tty-ldisc.md` (proves: line-discipline swap via `tty_set_ldisc` while concurrent read/write in-flight; ldisc refcount + ldsem ensure no use-after-free) |
| `models/tty/pty_peer.tla` | `drivers/tty/pty.md` (proves: PTY peer pair lifecycle — close one side + concurrent write from other; SIGHUP delivery + EIO return ordered correctly) |
| `models/tty/vt_switch.tla` | `drivers/tty/vt-core.md` (proves: VT switch via VT_ACTIVATE + concurrent screen write; per-vc lock + global VT_AUTO acknowledgement state machine converges) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-tty_struct + per-tty_port + per-tty_ldisc refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-driver `tty_operations`, per-ldisc `tty_ldisc_ops`, per-console `console` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-buffer flip-buffer + termios baud arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed tty_struct + tty_buffer cleared (carry credentials typed at password prompts) | § Default-on configurable |

TTY-specific reinforcement: **TIOCSTI input-injection requires CAP_SYS_ADMIN** in init userns (defense against TIOCSTI-injection escalation; CVE-2017-5390-class); upstream made TIOCSTI configurable via `dev.tty.legacy_tiocsti=` sysctl with default-1, Rookery defaults to 0 (deny). Per-cgroup tty device delegation respected. Serial console kdebug + KGDB-over-serial gated on CAP_SYS_RAWIO + per-LSM hook. SysRq magic-key default-mask reduced to 0x4 (sync-only) by default; `/proc/sys/kernel/sysrq=1` opt-in for full bitmask. Console keymap loads (`KDSKBSENT`/loadkeys) require CAP_SYS_TTY_CONFIG.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- ARM-only SoC UARTs beyond top-tier set (most `serial/{altera_uart, atmel_serial, …}` compile-gated off for v0)
- Legacy / dead drivers (`amiserial.c`, `goldfish.c`, `ipwireless/`, `nozomi.c`, `mips_ejtag_fdc.c`, `ehv_bytechan.c`)
- 32-bit-only paths
