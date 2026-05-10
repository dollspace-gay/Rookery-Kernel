---
title: "Tier-2: fs/nfs — NFS client + NFSv4 + NFSv4.1/4.2 + sunrpc + nfsd target + lockd + nfsacl"
tags: ["tier-2", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for NFS client + nfsd target + sunrpc transport — NFSv2/v3/v4.0/4.1/4.2 with optional pNFS-block, pNFS-flex-files, pNFS-files; integrated with sunrpc RPC framework + lockd + nfsacl + idmap + krb5/krb5i/krb5p sec. Components:

- **NFS client** (`fs/nfs/`): `nfs_inode.c`, `nfs_dir.c`, `nfs_file.c`, `nfs_super.c`, `nfs_mount.c`, `nfs2*.c`, `nfs3*.c`, `nfs4*.c` (per-version code), `nfs4client.c`, `nfs4state.c`, `nfs4session.c`, `pnfs.c` + `pnfs_dev.c` + `pnfs_nfs.c` + `flexfilelayout/` + `blocklayout/` + `filelayout/` (pNFS layouts), `delegation.c` (NFSv4 delegations), `direct.c` (O_DIRECT), `write.c` + `read.c` + `pagelist.c`, `dns_resolve.c` + `idmap.c` (idmapper for NFSv4), `unlink.c`, `getroot.c`, `mount_clnt.c`, `nfs4xdr.c` (XDR), `nfs4callback.c`, `proc.c`, `cache_lib.c`, `iostat.c` + `mountstat.c`
- **NFSv4 server (nfsd)** (`fs/nfsd/`): `nfsd.h`, `nfsd_xdr*.c`, `nfssvc.c`, `nfs2acl.c`, `nfs3acl.c`, `nfs3*.c`, `nfs4*.c`, `nfs4state.c`, `nfs4xdr.c`, `nfsproc.c`, `nfsfh.c`, `lockd.c`, `pnfs.c`, `vfs.c`, `cache.c`, `export.c`
- **sunrpc transport** (`net/sunrpc/`): `clnt.c`, `xprt.c`, `xprtsock.c`, `xprtrdma/` (RPC-over-RDMA), `auth*.c` (RPCSEC_GSS for krb5/krb5i/krb5p), `cache.c`, `sched.c`, `stats.c`, `svc.c` + `svcsock.c` + `svc_xprt.c` (server-side RPC), `xprtmultipath.c`, `socklib.c`, `sysctl.c`, `addr.c`
- **lockd** (`fs/lockd/`): NFSv2/v3 NLM (Network Lock Manager)
- **nfs_common** (`fs/nfs_common/`): nfsacl shared

### compatibility contract — outline

- `mount -t nfs[4]` mount options byte-identical (vers, sec, proto, port, rsize, wsize, timeo, retrans, hard/soft, intr, ac/noac, soft/hard, lookupcache, nordirplus, fsc, local_lock, nfsvers, retry, namlen, namlen, sloppy, _netdev).
- `/proc/fs/nfsd/{exports, threads, max_block_size, ...}` byte-identical.
- `/proc/self/mountstats` NFS section byte-identical.
- /etc/exports format consumed by exportfs unchanged.
- RPC wire format byte-identical (RFC 5531 + RFC 5661 + RFC 7530).
- pNFS layout types: NFS4_FILES, NFS4_OBJECTS, NFS4_BLOCK_VOL, NFS4_FLEX_FILES.
- Tracepoints `events/{nfs, sunrpc, nfsd, lockd}/` byte-identical.

### tier-3 docs governed (selection — full list deferred)

| Tier-3 | Scope |
|---|---|
| `fs/nfs/super-mount.md` | `nfs_super.c` + `nfs_mount.c` |
| `fs/nfs/inode-dir-file.md` | `nfs_inode.c` + `nfs_dir.c` + `nfs_file.c` + `read.c` + `write.c` + `pagelist.c` |
| `fs/nfs/v23.md` | `nfs2*.c` + `nfs3*.c`: NFSv2 + NFSv3 |
| `fs/nfs/v4.md` | `nfs4*.c` + `nfs4state.c` + `nfs4session.c` + `nfs4xdr.c` + `nfs4client.c` + `nfs4callback.c`: NFSv4.x |
| `fs/nfs/pnfs.md` | `pnfs*.c` + `flexfilelayout/` + `blocklayout/` + `filelayout/`: pNFS |
| `fs/nfs/delegation.md` | `delegation.c`: NFSv4 delegations |
| `fs/nfs/direct.md` | `direct.c`: O_DIRECT |
| `fs/nfs/idmap.md` | `idmap.c` + `dns_resolve.c`: NFSv4 idmapper |
| `fs/nfs/iostat.md` | `iostat.c` + `mountstat.c`: per-mount stats |
| `fs/nfsd/svc.md` | `nfssvc.c` + `vfs.c` + `nfsproc.c` + `nfsfh.c`: nfsd core |
| `fs/nfsd/v3.md` | `nfs3*.c` |
| `fs/nfsd/v4.md` | `nfs4*.c` + `nfs4state.c` + `nfs4xdr.c` |
| `fs/nfsd/export.md` | `export.c` + `cache.c` |
| `fs/nfsd/pnfs-server.md` | `pnfs.c` + `blocklayout.c` |
| `fs/lockd.md` | `fs/lockd/`: NLM |
| `net/sunrpc/clnt-xprt.md` | `clnt.c` + `xprt.c` + `xprtsock.c` + `socklib.c` + `addr.c`: client-side RPC |
| `net/sunrpc/xprtrdma.md` | `xprtrdma/`: RPC-over-RDMA |
| `net/sunrpc/svc.md` | `svc.c` + `svcsock.c` + `svc_xprt.c`: server-side RPC |
| `net/sunrpc/auth.md` | `auth*.c`: RPC auth (AUTH_NULL/AUTH_UNIX/RPCSEC_GSS krb5) |
| `net/sunrpc/cache-sched-stats.md` | `cache.c` + `sched.c` + `stats.c` + `sysctl.c` |

### compatibility outline / ac / verification / hardening

- REQ-O1: NFSv2/v3/v4.0/v4.1/v4.2 client + nfsd target wire format byte-identical (interop with stock Linux/FreeBSD/Solaris/Windows NFS clients/servers).
- REQ-O2: pNFS layouts work; pNFS-block backed by SCSI/iSCSI/iSER tested.
- REQ-O3: RPCSEC_GSS krb5/krb5i/krb5p auth works.
- REQ-O4: TLA+ models (NFSv4.1 sequence-id session-replay protection; delegation recall + concurrent local-fs op; pNFS layout-return + concurrent IO race).
- REQ-O5: AC: NFS server export to remote Linux client + read+write; pjdfstest passes; cthon04 NFS test suite passes.
- Hardening: row-1; nfsd export config requires CAP_SYS_ADMIN; krb5 keytab keyring-shielded; NLM rate-limited.

