# Tier-2: net/unix — AF_UNIX (stream + dgram + seqpacket + abstract + SCM_RIGHTS + sock-diag + GC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/unix/
  - include/net/af_unix.h
-->

## Summary

Tier-2 wrapper for AF_UNIX (Unix domain sockets) — every dbus session/system bus, every Wayland compositor↔client, every X11 display socket, every PulseAudio/PipeWire/JACK control channel, every container runtime control socket (containerd, podman, docker), every nginx/php-fpm/uwsgi proxy socket, every systemd notify-socket. Components: `af_unix.c` (AF_UNIX socket impl — SOCK_STREAM + SOCK_DGRAM + SOCK_SEQPACKET; pathname-bound + abstract-namespace + autobind; SCM_RIGHTS fd passing; SCM_CREDENTIALS peer credential exchange), `garbage.c` (per-fd-graph cycle detector + GC for SCM_RIGHTS-passed fds — defense against fd-reference-cycle leak), `diag.c` (NETLINK_UNIX_DIAG for `ss -x`), `scm.c` (SCM_RIGHTS receive helper), `sysctl_net_unix.c` (sysctl knobs).

## Compatibility contract — outline

- AF_UNIX socket types `SOCK_STREAM`/`_DGRAM`/`_SEQPACKET` byte-identical UAPI.
- `struct sockaddr_un` w/ pathname (max 108 bytes incl null) + abstract-namespace (leading null byte) + unnamed (autobind) byte-identical.
- SCM_RIGHTS fd-passing wire format identical (CMSG buffer with `cmsg_level=SOL_SOCKET` + `cmsg_type=SCM_RIGHTS` + array of int fds).
- SCM_CREDENTIALS struct ucred (pid, uid, gid) byte-identical.
- SCM_SECURITY (LSM peer label) byte-identical.
- `getsockopt(SO_PEERCRED, SO_PEERSEC, SO_PEERGROUPS, SO_PEERPIDFD)` byte-identical.
- `/proc/net/{unix, unix_diag}` byte-identical.
- `/proc/sys/net/unix/max_dgram_qlen` byte-identical.
- NETLINK_UNIX_DIAG byte-identical (ss consumes).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/unix/af-unix.md` | `af_unix.c`: socket impl |
| `net/unix/scm.md` | `scm.c`: SCM_RIGHTS / SCM_CREDENTIALS / SCM_SECURITY |
| `net/unix/garbage.md` | `garbage.c`: SCM_RIGHTS-fd cycle GC |
| `net/unix/diag.md` | `diag.c`: NETLINK_UNIX_DIAG |
| `net/unix/sysctl.md` | `sysctl_net_unix.c` |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: AF_UNIX UAPI byte-identical (every userspace consumer works).
- REQ-O2: SCM_RIGHTS/CREDENTIALS/SECURITY/PIDFD passing byte-identical.
- REQ-O3: Abstract-namespace + autobind + pathname binding byte-identical.
- REQ-O4: TLA+ models (SCM_RIGHTS GC reachability — every fd reachable from userspace held; cycle-only fds collected; per-socket recv queue concurrent send/recv; SOCK_SEQPACKET msg-boundary preservation).
- REQ-O5: AC: dbus-daemon test, Wayland compositor session, systemd notify-socket all work; ss -x lists Unix sockets correctly.
- Hardening: per-namespace UNIX scoping; abstract-namespace requires same userns; SCM_CREDENTIALS lying defended via per-LSM hook (cross-ref `security/selinux/socket-hooks.md`); SCM_RIGHTS fd-pass count capped (`/proc/sys/net/unix/max_pass_fds=`, default 253); pathname socket creation respects FS LSM hooks.

## Out of Scope
- Implementation code; 32-bit-only paths
