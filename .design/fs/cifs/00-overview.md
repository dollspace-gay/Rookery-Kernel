# Tier-2: fs/smb — SMB/CIFS client + ksmbd server (SMB1/2/3 + multichannel + RDMA + DFS + krb5)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/smb/
  - include/uapi/linux/cifs/
-->

## Summary

Tier-2 wrapper for SMB — Linux's SMB1/2/3 client (`fs/smb/client/` — formerly `fs/cifs/`) + ksmbd kernel server (`fs/smb/server/`). Backbone of `mount -t cifs //server/share` for accessing Windows/Samba shares + Linux-to-Linux SMB serving via ksmbd (faster than Samba userspace for some workloads). Components: **client** (`client/cifsfs.c`, `connect.c`, `transport.c`, `cifsglob.h`, `inode.c`, `dir.c`, `file.c`, `link.c`, `ioctl.c`, `cifsacl.c`, `xattr.c`, `dfs*.c` (DFS referrals), `smb1ops.c` + `smb2ops.c` + `smb2pdu.c` (per-version op tables), `smb2transport.c`, `smb2maperror.c`, `smb2glob.h`, `cifsencrypt.c` + `smb2crypto.c` (signing + encryption), `link.c` (symlinks), `nterr.c` (NT error code mapping), `readdir.c`, `cached_dir.c`, `compress.c` (SMB compression), `fscache.c`, `multichannel.c`, `rdma.c` (SMB-Direct RDMA), `reparse.c` (NTFS reparse points), `unc.c`, `cifs_debug.c`, `cifs_swn.c` (Switch-Witness Network)), **server** (`server/`: `ksmbd.h`, `connection.c`, `crypto_ctx.c`, `mgmt/{*}`, `smb_common.c`, `smbacl.c`, `smb2*.c`, `transport_ipc.c` + `transport_tcp.c` + `transport_rdma.c`, `auth.c`, `oplock.c`, `vfs.c`, `vfs_cache.c`, `ksmbd_netlink.h`, `unicode.c`, `xattr.h`, `nterr.h`, `glob.h`, `compress.c`, `misc.c` — kernel SMB3 server).

## Compatibility contract — outline

- `mount -t cifs` mount options byte-identical (vers, sec, user, password, domain, uid, gid, file_mode, dir_mode, hard, soft, mfsymlinks, multiuser, noperm, sign, seal, multichannel, rdma, …).
- DFS referrals + WSP-CSC follow-up identical.
- ksmbd configuration via `/sys/kernel/config/ksmbd/` configfs + `/etc/ksmbd/ksmbd.conf` userspace consumes byte-identical.
- SMB2 wire format (RFC 8447 / [MS-SMB2]) byte-identical.
- SMB3-encryption AES-128-CCM/GCM/AES-256-GCM byte-identical.
- SMB-Direct RDMA RFC 5040/RFC 7146 byte-identical (interop with Windows Server / Samba / netapp SMB-Direct).

## Tier-3 docs governed (selection)

| Tier-3 | Scope |
|---|---|
| `fs/cifs/client-mount.md` | `client/cifsfs.c` + `connect.c` |
| `fs/cifs/client-inode-dir-file.md` | `client/{inode, dir, file, link, readdir, cached_dir, reparse}.c` |
| `fs/cifs/client-smb1-smb2.md` | `client/{smb1ops, smb2ops, smb2pdu, smb2transport, smb2maperror}.c` |
| `fs/cifs/client-crypto.md` | `client/{cifsencrypt, smb2crypto}.c` + signing + AES-CCM/GCM |
| `fs/cifs/client-dfs.md` | `client/dfs*.c` |
| `fs/cifs/client-multichannel.md` | `client/multichannel.c` |
| `fs/cifs/client-rdma.md` | `client/rdma.c`: SMB-Direct |
| `fs/cifs/client-fscache.md` | `client/fscache.c`: client-side caching |
| `fs/cifs/client-acl.md` | `client/{cifsacl, smbacl, xattr}.c` |
| `fs/cifs/server-core.md` | `server/{ksmbd.h, connection, smb_common, vfs, vfs_cache}.c` |
| `fs/cifs/server-smb2.md` | `server/smb2*.c` |
| `fs/cifs/server-auth.md` | `server/{auth, crypto_ctx}.c` |
| `fs/cifs/server-transport.md` | `server/{transport_ipc, transport_tcp, transport_rdma}.c` |
| `fs/cifs/server-acl.md` | `server/smbacl.c` |
| `fs/cifs/server-oplock.md` | `server/oplock.c` |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: SMB2/3 wire format byte-identical (interop Win Server / Samba / netapp).
- REQ-O2: SMB-Direct RDMA works.
- REQ-O3: ksmbd configfs byte-identical.
- REQ-O4: TLA+ models (SMB2 lock + oplock break race; per-session sequence-id; SMB-Direct credit-flow control).
- REQ-O5: AC: mount Windows share + read/write; ksmbd serves Linux share to Win10 client; smbtorture / pjdfstest pass.
- Hardening: row-1; krb5 keytab keyring-shielded; SMB1 default-disabled (CVE risk); SMB-Encryption required default for new mounts; ksmbd configfs writes CAP_SYS_ADMIN.

## Out of Scope
Implementation code; userspace Samba (separate project).
