---
title: "Tier-3: security/lsm_audit.c — common LSM audit helpers"
tags: ["tier-3", "security", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`security/lsm_audit.c` is the per-LSM **common audit-record emitter**. LSMs (SELinux, AppArmor, Smack, …) construct a `struct common_audit_data` describing the denied/granted access — type-tagged with one of `LSM_AUDIT_DATA_{PATH,NET,CAP,IPC,TASK,KEY,NONE,KMOD,INODE,DENTRY,IOCTL_OP,FILE,IBPKEY,IBENDPORT,LOCKDOWN,NOTIFICATION,ANONINODE,NLMSGTYPE}` — and call `common_lsm_audit(a, pre_audit, post_audit)`. The helper opens an `AUDIT_AVC` audit buffer with `GFP_ATOMIC | __GFP_NOWARN` (kept atomic-safe so it can fire from any context), invokes the per-LSM `pre_audit` callback to print LSM-specific fields, dumps common fields (pid, comm, plus the type-specific payload via `audit_log_lsm_data`), and finally invokes `post_audit`. Per network records `struct lsm_network_audit` is allocated separately so the `common_audit_data::u` union stays small enough (`BUILD_BUG_ON(sizeof(a->u) > sizeof(void *) * 2)`) to live on stack in callers. Per IPv4/IPv6 packet `ipv4_skb_to_auditdata()` / `ipv6_skb_to_auditdata()` extract saddr/daddr/sport/dport for TCP/UDP/SCTP. Critical for: SELinux AVC denials, AppArmor messages, Smack audit trail, MAC accountability.

This Tier-3 covers `security/lsm_audit.c` (~457 lines).

### Acceptance Criteria

- [ ] AC-1: common_lsm_audit(NULL, …) is a no-op.
- [ ] AC-2: When audit_log_start returns NULL (rate-limited / disabled), no further calls are made and no buffer is leaked.
- [ ] AC-3: pre_audit (when non-NULL) runs strictly before dump_common_audit_data; post_audit (when non-NULL) strictly after.
- [ ] AC-4: dump_common_audit_data always emits " pid=… comm=…" prefix.
- [ ] AC-5: BUILD_BUG_ON triggers at build time if sizeof(common_audit_data::u) exceeds 2 * sizeof(void *).
- [ ] AC-6: LSM_AUDIT_DATA_PATH/FILE/IOCTL_OP/DENTRY/INODE emit " dev=… ino=%llu" when a backing inode is reachable.
- [ ] AC-7: LSM_AUDIT_DATA_NET prints AF_INET via %pI4, AF_INET6 via %pI6c, AF_UNIX via path-or-hex.
- [ ] AC-8: LSM_AUDIT_DATA_NET resolves netif against init_net, increments dev refcount, and dev_put on completion.
- [ ] AC-9: LSM_AUDIT_DATA_TASK skips if tsk is NULL or task_tgid_nr(tsk) == 0.
- [ ] AC-10: LSM_AUDIT_DATA_KEY guarded by CONFIG_KEYS; absent symbol when disabled.
- [ ] AC-11: LSM_AUDIT_DATA_LOCKDOWN prints `lockdown_reasons[reason]`; out-of-range reason index is rejected before reaching audit_log_lsm_data (caller invariant).
- [ ] AC-12: ipv4_skb_to_auditdata returns 0 for TCP/UDP/SCTP; -EINVAL for other protocols; 0 (with sport/dport untouched) for non-initial fragments.
- [ ] AC-13: ipv6_skb_to_auditdata returns 0 for TCP/UDP/SCTP; -EINVAL for other nexthdr; 0 (early return) when ipv6_skip_exthdr fails.
- [ ] AC-14: print_ipv4_addr / print_ipv6_addr omit fields when address or port is zero/any.
- [ ] AC-15: audit_log_start uses GFP_ATOMIC | __GFP_NOWARN and AUDIT_AVC type — call must succeed from any sleepable / non-sleepable context.

### Architecture

```
struct LsmNetworkAudit {
  netif:  i32,
  sk:     Option<*const Sock>,
  family: u16,
  dport:  Be16,
  sport:  Be16,
  fam:    LsmNetworkFam,            // union v4 / v6
}

enum LsmNetworkFam {
  V4 { saddr: Be32, daddr: Be32 },
  V6 { saddr: In6Addr, daddr: In6Addr },
}

struct LsmIoctlopAudit { path: Path, cmd: u16 }
struct LsmIbpkeyAudit  { subnet_prefix: u64, pkey: u16 }
struct LsmIbendportAudit { dev_name: *const c_char, port: u8 }

enum LsmAuditDataKind {
  Path = 1, Net = 2, Cap = 3, Ipc = 4, Task = 5, Key = 6, None = 7, Kmod = 8,
  Inode = 9, Dentry = 10, IoctlOp = 11, File = 12, Ibpkey = 13, Ibendport = 14,
  Lockdown = 15, Notification = 16, Anoninode = 17, Nlmsgtype = 18,
}

struct CommonAuditData {
  type_:  LsmAuditDataKind,
  u:      CommonAuditPayload,       // tagged union (≤ 2 * size_of::<*const ()>())
  lsm:    LsmSpecificAuditPtr,      // smack | selinux | apparmor opaque pointer
}

enum CommonAuditPayload {
  Path(Path),
  Dentry(*const Dentry),
  Inode(*const Inode),
  Net(*const LsmNetworkAudit),
  Cap(i32),
  Ipc(i32),
  Task(*const TaskStruct),
  Key { key: KeySerial, desc: *const c_char },
  Kmod(*const c_char),
  IoctlOp(*const LsmIoctlopAudit),
  File(*const File),
  Ibpkey(*const LsmIbpkeyAudit),
  Ibendport(*const LsmIbendportAudit),
  Lockdown(LockdownReason),
  Anoninode(*const c_char),
  Nlmsgtype(u16),
  None,
}
```

`LsmAudit::emit(a: &CommonAuditData, pre: Option<PreCb>, post: Option<PostCb>)`:
1. /* No-op if caller passed NULL */
2. /* GFP_ATOMIC: callable from softirq / spinlock / kprobe */
3. let ab = audit_log_start(audit_context(), Gfp::ATOMIC | Gfp::NOWARN, AUDIT_AVC).
4. if ab.is_null(): return.
5. if let Some(p) = pre: p(ab, a).
6. LsmAudit::dump_common(ab, a).
7. if let Some(p) = post: p(ab, a).
8. audit_log_end(ab).

`LsmAudit::dump_common(ab, a)`:
1. let mut comm = [0u8; TASK_COMM_LEN].
2. audit_log_format!(ab, " pid={} comm=", task_tgid_nr(current())).
3. audit_log_untrustedstring(ab, get_task_comm(&mut comm, current())).
4. LsmAudit::log_data(ab, a).

`LsmAudit::log_data(ab, a)`:
1. const_assert!(size_of::<CommonAuditPayload>() <= 2 * size_of::<*const ()>()).
2. match a.type_:
   - None: return.
   - Ipc:  audit_log_format!(ab, " ipc_key={} ", a.u.ipc_id).
   - Cap:  audit_log_format!(ab, " capability={} ", a.u.cap).
   - Path: audit_log_d_path(ab, " path=", &a.u.path); if let Some(inode) = d_backing_inode(a.u.path.dentry): emit_dev_ino(ab, inode).
   - File: audit_log_d_path(ab, " path=", &a.u.file.f_path); if let Some(inode) = file_inode(a.u.file): emit_dev_ino(ab, inode).
   - IoctlOp: audit_log_d_path(ab, " path=", &a.u.op.path); if let Some(inode) = a.u.op.path.dentry.d_inode: emit_dev_ino(ab, inode); audit_log_format!(ab, " ioctlcmd=0x{:hx}", a.u.op.cmd).
   - Dentry: with spin_lock(&a.u.dentry.d_lock): emit_name(ab, &a.u.dentry.d_name); if let Some(inode) = d_backing_inode(a.u.dentry): emit_dev_ino(ab, inode).
   - Inode: rcu::read(|| { if let Some(dentry) = d_find_alias_rcu(a.u.inode): with spin_lock(&dentry.d_lock): emit_name(ab, &dentry.d_name); emit_dev_ino(ab, a.u.inode); }).
   - Task: if let Some(tsk) = a.u.tsk; if (pid := task_tgid_nr(tsk)) != 0: audit_log_format!(ab, " opid={} ocomm=", pid); audit_log_untrustedstring(ab, get_task_comm(&mut buf, tsk)).
   - Net: LsmAudit::emit_net(ab, a.u.net).
   - Key:  audit_log_format!(ab, " key_serial={}", a.u.key.key); if !a.u.key.desc.is_null(): audit_log_format!(ab, " key_desc="); audit_log_untrustedstring(ab, a.u.key.desc).
   - Kmod: audit_log_format!(ab, " kmod="); audit_log_untrustedstring(ab, a.u.kmod_name).
   - Ibpkey:  let mut sbn_pfx = In6Addr::ZERO; memcpy(&sbn_pfx, &a.u.ibpkey.subnet_prefix, 8); audit_log_format!(ab, " pkey=0x{:x} subnet_prefix={:pI6c}", a.u.ibpkey.pkey, &sbn_pfx).
   - Ibendport: audit_log_format!(ab, " device={} port_num={}", a.u.ibendport.dev_name, a.u.ibendport.port).
   - Lockdown: audit_log_format!(ab, " lockdown_reason=\"{}\"", LOCKDOWN_REASONS[a.u.reason]).
   - Anoninode: audit_log_format!(ab, " anonclass={}", a.u.anonclass).
   - Nlmsgtype: audit_log_format!(ab, " nl-msgtype={}", a.u.nlmsg_type).

`LsmAudit::emit_net(ab, net: &LsmNetworkAudit)`:
1. if let Some(sk) = net.sk:
   - match sk.sk_family:
     - AF_INET: let inet = inet_sk(sk); print_ipv4_addr(ab, inet.inet_rcv_saddr, inet.inet_sport, "laddr", "lport"); print_ipv4_addr(ab, inet.inet_daddr, inet.inet_dport, "faddr", "fport").
     - AF_INET6 (CONFIG_IPV6): print_ipv6_addr(ab, &sk.sk_v6_rcv_saddr, inet_sk(sk).inet_sport, "laddr", "lport"); print_ipv6_addr(ab, &sk.sk_v6_daddr, inet_sk(sk).inet_dport, "faddr", "fport").
     - AF_UNIX: let u = unix_sk(sk); let addr = smp_load_acquire(&u.addr); if addr.is_some():
       - if u.path.dentry.is_some(): audit_log_d_path(ab, " path=", &u.path).
       - else: let p = addr.name.sun_path; if p[0] != 0: audit_log_format!(ab, " path="); audit_log_untrustedstring(ab, p); else: audit_log_format!(ab, " path="); audit_log_n_hex(ab, p, addr.len - sizeof(short)).
2. match net.family:
   - AF_INET: print_ipv4_addr(ab, net.v4info.saddr, net.sport, "saddr", "src"); print_ipv4_addr(ab, net.v4info.daddr, net.dport, "daddr", "dest").
   - AF_INET6: print_ipv6_addr(ab, &net.v6info.saddr, net.sport, "saddr", "src"); print_ipv6_addr(ab, &net.v6info.daddr, net.dport, "daddr", "dest").
3. if net.netif > 0:
   - let dev = dev_get_by_index(&init_net, net.netif).
   - if dev.is_some(): audit_log_format!(ab, " netif={}", dev.name); dev_put(dev).

`LsmAudit::ipv4_skb_to_auditdata(skb, ad, proto) -> KResult<()>`:
1. let ih = ip_hdr(skb).
2. ad.u.net.v4info.saddr = ih.saddr; ad.u.net.v4info.daddr = ih.daddr.
3. if !proto.is_null(): *proto = ih.protocol.
4. if ntohs(ih.frag_off) & IP_OFFSET != 0: return Ok(()).
5. match ih.protocol:
   - IPPROTO_TCP: let th = tcp_hdr(skb); ad.u.net.{sport,dport} = th.{source,dest}.
   - IPPROTO_UDP: let uh = udp_hdr(skb); ad.u.net.{sport,dport} = uh.{source,dest}.
   - IPPROTO_SCTP: let sh = sctp_hdr(skb); ad.u.net.{sport,dport} = sh.{source,dest}.
   - _: return Err(EINVAL).
6. Ok(()).

`LsmAudit::ipv6_skb_to_auditdata(skb, ad, proto) -> KResult<()>` (CONFIG_IPV6):
1. let ip6 = ipv6_hdr(skb).
2. ad.u.net.v6info.{saddr,daddr} = ip6.{saddr,daddr}.
3. let mut offset = skb_network_offset(skb) + size_of::<Ipv6Hdr>().
4. let mut nexthdr = ip6.nexthdr; let mut frag_off: Be16 = 0.
5. offset = ipv6_skip_exthdr(skb, offset, &mut nexthdr, &mut frag_off).
6. if offset < 0: return Ok(()).
7. if !proto.is_null(): *proto = nexthdr.
8. match nexthdr:
   - IPPROTO_TCP | IPPROTO_UDP | IPPROTO_SCTP: let buf = …; let l4 = skb_header_pointer(skb, offset, size_of::<L4Hdr>(), &mut buf); if l4.is_null(): early-break; else ad.u.net.{sport,dport} = l4.{source,dest}.
   - _: return Err(EINVAL).
9. Ok(()).

`print_ipv4_addr(ab, addr: Be32, port: Be16, name1, name2)`:
1. if addr != 0: audit_log_format!(ab, " {}={:pI4}", name1, &addr).
2. if port != 0: audit_log_format!(ab, " {}={}", name2, ntohs(port)).

`print_ipv6_addr(ab, addr: &In6Addr, port: Be16, name1, name2)`:
1. if !ipv6_addr_any(addr): audit_log_format!(ab, " {}={:pI6c}", name1, addr).
2. if port != 0: audit_log_format!(ab, " {}={}", name2, ntohs(port)).

### Out of Scope

- `kernel/audit.c` / `kernel/auditsc.c` / `kernel/auditfilter.c` core audit subsystem (audit_log_start, audit_log_format, audit_log_untrustedstring, audit_log_d_path, audit context, rules) — covered under `kernel/audit-core.md` if expanded.
- `security/security.c` LSM core dispatch and composite blobs (covered in `security-core.md` Tier-3).
- `security/selinux/avc.c` SELinux AVC (per-LSM pre_audit/post_audit callback site — covered under `security/selinux/avc.md` Tier-3).
- `security/apparmor/audit.c` AppArmor audit callbacks (covered under `security/apparmor/`).
- `security/smack/smack_access.c` Smack audit callbacks (covered separately if expanded).
- InfiniBand / RDMA per-LSM hooks (`security_ib_*`) — callsites only; struct definitions here.
- Netlink audit `AUDIT_AVC` record format wire-level details (covered under `kernel/audit-netlink.md` if expanded).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct common_audit_data` | per-audit-record type-tagged payload | `CommonAuditData` |
| `struct lsm_network_audit` | per-net-audit payload (separately allocated) | `LsmNetworkAudit` |
| `struct lsm_ioctlop_audit` | per-ioctl audit payload (path + cmd) | `LsmIoctlopAudit` |
| `struct lsm_ibpkey_audit` | per-InfiniBand-pkey audit payload | `LsmIbpkeyAudit` |
| `struct lsm_ibendport_audit` | per-IB-endport audit payload | `LsmIbendportAudit` |
| `common_lsm_audit()` | per-record emit (pre → common → post) | `LsmAudit::emit` |
| `audit_log_lsm_data()` | per-type payload formatter | `LsmAudit::log_data` |
| `dump_common_audit_data()` | per-record dump (pid/comm + payload) | `LsmAudit::dump_common` |
| `ipv4_skb_to_auditdata()` | per-IPv4 skb → audit fields | `LsmAudit::ipv4_skb_to_auditdata` |
| `ipv6_skb_to_auditdata()` | per-IPv6 skb → audit fields | `LsmAudit::ipv6_skb_to_auditdata` |
| `print_ipv4_addr()` | per-IPv4 addr+port formatter | `LsmAudit::print_ipv4_addr` |
| `print_ipv6_addr()` | per-IPv6 addr+port formatter | `LsmAudit::print_ipv6_addr` |
| `LSM_AUDIT_DATA_*` | per-payload type tag (1..18) | `LsmAuditDataKind` |
| `v4info` / `v6info` | per-fam.v4 / per-fam.v6 alias | shared |
| `lockdown_reasons[]` (declared in security.c) | per-LOCKDOWN_* description (consumed here) | shared |
| `audit_log_start()` / `audit_log_end()` / `audit_log_format()` / `audit_log_untrustedstring()` / `audit_log_d_path()` / `audit_log_n_hex()` | per-audit-buffer primitives | provided by kernel/audit core |
| `AUDIT_AVC` | audit record type code | shared with `kernel/auditsc.c` |

### compatibility contract

REQ-1: struct lsm_network_audit:
- netif: int (network interface index; resolved via `dev_get_by_index(&init_net, ...)` at emit time).
- sk: const struct sock * (optional; if set, formats per-sk per-family addresses).
- family: u16 (AF_INET / AF_INET6).
- dport: __be16 (peer port, network-byte-order).
- sport: __be16 (local port).
- fam: union { struct { __be32 daddr, saddr; } v4; struct { struct in6_addr daddr, saddr; } v6; }.
- /* aliases */ #define v4info fam.v4; #define v6info fam.v6.

REQ-2: struct lsm_ioctlop_audit:
- path: struct path (target path).
- cmd: u16 (ioctl request code, low 16 bits).

REQ-3: struct lsm_ibpkey_audit:
- subnet_prefix: u64 (InfiniBand subnet prefix).
- pkey: u16.

REQ-4: struct lsm_ibendport_audit:
- dev_name: const char * (device name).
- port: u8 (port number).

REQ-5: struct common_audit_data:
- type: char (LSM_AUDIT_DATA_*).
- u: union {
    struct path path;            /* LSM_AUDIT_DATA_PATH */
    struct dentry *dentry;       /* LSM_AUDIT_DATA_DENTRY */
    struct inode *inode;         /* LSM_AUDIT_DATA_INODE */
    struct lsm_network_audit *net;  /* LSM_AUDIT_DATA_NET (heap, due to size) */
    int cap;                     /* LSM_AUDIT_DATA_CAP */
    int ipc_id;                  /* LSM_AUDIT_DATA_IPC */
    struct task_struct *tsk;     /* LSM_AUDIT_DATA_TASK */
#ifdef CONFIG_KEYS
    struct { key_serial_t key; char *key_desc; } key_struct;  /* LSM_AUDIT_DATA_KEY */
#endif
    char *kmod_name;             /* LSM_AUDIT_DATA_KMOD */
    struct lsm_ioctlop_audit *op;  /* LSM_AUDIT_DATA_IOCTL_OP */
    const struct file *file;     /* LSM_AUDIT_DATA_FILE */
    struct lsm_ibpkey_audit *ibpkey;     /* LSM_AUDIT_DATA_IBPKEY */
    struct lsm_ibendport_audit *ibendport;  /* LSM_AUDIT_DATA_IBENDPORT */
    int reason;                  /* LSM_AUDIT_DATA_LOCKDOWN — index into lockdown_reasons[] */
    const char *anonclass;       /* LSM_AUDIT_DATA_ANONINODE */
    u16 nlmsg_type;              /* LSM_AUDIT_DATA_NLMSGTYPE */
  }.
- per-LSM-data union (smack_audit_data / selinux_audit_data / apparmor_audit_data) — opaque, used by LSM-specific pre/post callbacks.
- /* SIZE INVARIANT: BUILD_BUG_ON(sizeof(a->u) > sizeof(void *) * 2) — keeps the type stack-friendly */

REQ-6: LSM_AUDIT_DATA_* tags (stable values, 1..18):
- 1 PATH, 2 NET, 3 CAP, 4 IPC, 5 TASK, 6 KEY, 7 NONE, 8 KMOD, 9 INODE, 10 DENTRY, 11 IOCTL_OP, 12 FILE, 13 IBPKEY, 14 IBENDPORT, 15 LOCKDOWN, 16 NOTIFICATION, 17 ANONINODE, 18 NLMSGTYPE.

REQ-7: ipv4_skb_to_auditdata(skb, ad, proto):
- ih = ip_hdr(skb).
- ad->u.net->v4info.saddr = ih->saddr; ad->u.net->v4info.daddr = ih->daddr.
- if proto != NULL: *proto = ih->protocol.
- if ntohs(ih->frag_off) & IP_OFFSET: return 0 (non-initial fragment; no L4 header).
- switch ih->protocol:
  - IPPROTO_TCP: ad->u.net->{sport,dport} = tcp_hdr(skb)->{source,dest}.
  - IPPROTO_UDP: ad->u.net->{sport,dport} = udp_hdr(skb)->{source,dest}.
  - IPPROTO_SCTP: ad->u.net->{sport,dport} = sctp_hdr(skb)->{source,dest}.
  - default: return -EINVAL.
- return 0.

REQ-8: ipv6_skb_to_auditdata(skb, ad, proto) [CONFIG_IPV6]:
- ip6 = ipv6_hdr(skb).
- ad->u.net->v6info.{saddr,daddr} = ip6->{saddr,daddr}.
- offset = skb_network_offset(skb) + sizeof(*ip6).
- nexthdr = ip6->nexthdr; offset = ipv6_skip_exthdr(skb, offset, &nexthdr, &frag_off).
- if offset < 0: return 0 (skip-ext-hdr failed; addresses already populated).
- if proto: *proto = nexthdr.
- switch nexthdr:
  - IPPROTO_TCP / IPPROTO_UDP / IPPROTO_SCTP: skb_header_pointer(skb, offset, sizeof(L4), &_l4); if !L4: break (truncated); else {sport,dport} = L4->{source,dest}.
  - default: return -EINVAL.
- return 0.

REQ-9: print_ipv4_addr(ab, addr, port, name1, name2):
- if addr != 0: audit_log_format(ab, " %s=%pI4", name1, &addr).
- if port != 0: audit_log_format(ab, " %s=%d", name2, ntohs(port)).

REQ-10: print_ipv6_addr(ab, addr, port, name1, name2):
- if !ipv6_addr_any(addr): audit_log_format(ab, " %s=%pI6c", name1, addr).
- if port: audit_log_format(ab, " %s=%d", name2, ntohs(port)).

REQ-11: audit_log_lsm_data(ab, a) — per-type formatter:
- BUILD_BUG_ON(sizeof(a->u) > sizeof(void *) * 2).
- switch a->type:
  - NONE: return.
  - IPC: " ipc_key=%d ".
  - CAP: " capability=%d ".
  - PATH: audit_log_d_path(" path=", &a->u.path); inode = d_backing_inode(a->u.path.dentry); if inode: " dev=…" + " ino=%llu".
  - FILE: audit_log_d_path(" path=", &a->u.file->f_path); inode = file_inode(a->u.file); if inode: dev + ino.
  - IOCTL_OP: audit_log_d_path(" path=", &a->u.op->path); inode = a->u.op->path.dentry->d_inode; if inode: dev + ino; then " ioctlcmd=0x%hx".
  - DENTRY: spin_lock(&a->u.dentry->d_lock); " name=" untrusted (d_name.name); unlock; inode = d_backing_inode; if inode: dev + ino.
  - INODE: rcu_read_lock; dentry = d_find_alias_rcu(inode); if dentry: " name=" (spin-locked); always print " dev=" + " ino=%llu"; rcu_read_unlock.
  - TASK: tsk = a->u.tsk; if tsk ∧ pid = task_tgid_nr(tsk) > 0: " opid=%d ocomm=%s" via get_task_comm().
  - NET: if a->u.net->sk:
      switch sk->sk_family:
        AF_INET: inet = inet_sk(sk); print_ipv4_addr(laddr/lport from inet_rcv_saddr/inet_sport; faddr/fport from inet_daddr/inet_dport).
        AF_INET6 [CONFIG_IPV6]: print_ipv6_addr(laddr/lport from sk_v6_rcv_saddr/inet_sport; faddr/fport from sk_v6_daddr/inet_dport).
        AF_UNIX: u = unix_sk(sk); addr = smp_load_acquire(&u->addr); if !addr: break; if u->path.dentry: audit_log_d_path(" path=", &u->path); else: " path=" + untrusted-or-hex of sun_path.
    then switch a->u.net->family:
        AF_INET: print_ipv4_addr(saddr/src, daddr/dest from v4info).
        AF_INET6: print_ipv6_addr(saddr/src, daddr/dest from v6info).
    then if a->u.net->netif > 0: dev = dev_get_by_index(&init_net, netif); if dev: " netif=%s"; dev_put(dev).
  - KEY [CONFIG_KEYS]: " key_serial=%u"; if key_desc: " key_desc=" + untrusted(key_desc).
  - KMOD: " kmod=" + untrusted(kmod_name).
  - IBPKEY: memset(sbn_pfx, 0); memcpy from a->u.ibpkey->subnet_prefix; " pkey=0x%x subnet_prefix=%pI6c".
  - IBENDPORT: " device=%s port_num=%u".
  - LOCKDOWN: " lockdown_reason=\"%s\"" with lockdown_reasons[a->u.reason].
  - ANONINODE: " anonclass=%s".
  - NLMSGTYPE: " nl-msgtype=%hu".

REQ-12: dump_common_audit_data(ab, a):
- char comm[sizeof(current->comm)].
- audit_log_format(ab, " pid=%d comm=", task_tgid_nr(current)).
- audit_log_untrustedstring(ab, get_task_comm(comm, current)).
- audit_log_lsm_data(ab, a).

REQ-13: common_lsm_audit(a, pre_audit, post_audit):
- if a == NULL: return.
- ab = audit_log_start(audit_context(), GFP_ATOMIC | __GFP_NOWARN, AUDIT_AVC).
- if ab == NULL: return (audit subsystem dropped / rate-limited).
- if pre_audit: pre_audit(ab, a).
- dump_common_audit_data(ab, a).
- if post_audit: post_audit(ab, a).
- audit_log_end(ab).

REQ-14: !CONFIG_AUDIT fallback (stub):
- common_lsm_audit() and audit_log_lsm_data() compile to empty inline functions in include/linux/lsm_audit.h.

REQ-15: per-arch endianness:
- saddr/daddr/sport/dport are __be32/__be16 — copied raw; conversion to host order happens only at format time (ntohs).

REQ-16: per-NS scoping:
- netif lookup uses `init_net` (NOTE comment in source: "we always use init's namespace") — audit always emits the netdev name from init_net, not the per-task netns.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `null_audit_data_is_noop` | INVARIANT | per-common_lsm_audit(NULL, …): returns without side effect. |
| `audit_buffer_paired_start_end` | INVARIANT | per-emit: every successful audit_log_start has matching audit_log_end (none on NULL). |
| `audit_buffer_no_use_after_end` | INVARIANT | per-emit: ab is not dereferenced after audit_log_end. |
| `payload_size_bound` | BUILD-TIME | per-CommonAuditPayload: size ≤ 2 * sizeof(*const ()). |
| `netdev_refcount_balanced` | INVARIANT | per-emit_net: dev_get_by_index ⇒ dev_put on the same path. |
| `unix_addr_acquire_loads` | INVARIANT | per-emit_net AF_UNIX: smp_load_acquire used (matches store_release on unix bind). |
| `ipv4_proto_classification_total` | INVARIANT | per-ipv4_skb_to_auditdata: returns 0 (success), 0 (fragment), or -EINVAL only. |
| `ipv6_proto_classification_total` | INVARIANT | per-ipv6_skb_to_auditdata: returns 0 (success), 0 (skip-exthdr fail), or -EINVAL only. |
| `lockdown_reason_in_range` | PRECONDITION | per-emit Lockdown variant: reason ∈ 0..=LOCKDOWN_CONFIDENTIALITY_MAX. |
| `dentry_dname_under_dlock` | INVARIANT | per-emit DENTRY: a.u.dentry.d_name access wrapped in spin_lock(&dentry.d_lock). |
| `inode_alias_under_rcu` | INVARIANT | per-emit INODE: d_find_alias_rcu inside rcu_read_lock; dentry.d_name access still spin-locked. |

### Layer 2: TLA+

`security/lsm-audit.tla`:
- States: pre-emit, opened-buffer, pre-cb-run, common-dumped, post-cb-run, closed.
- Actions: emit_null_noop, emit_log_start_fail, emit_full_path.
- Properties:
  - `safety_no_dangling_buffer` — every Opened state transitions to Closed within finite steps.
  - `safety_callback_order_pre_before_common_before_post` — strict ordering of pre / dump / post.
  - `safety_no_emit_without_audit_context` — audit_log_start sees current audit_context().
  - `liveness_emit_terminates` — bounded by the per-record formatter (no unbounded loops).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `LsmAudit::emit` pre: a may be NULL ⇒ no-op; non-NULL ⇒ a.type_ ∈ LsmAuditDataKind | `LsmAudit::emit` |
| `LsmAudit::emit` post: no audit buffer leaked; pre→dump→post invoked iff buffer obtained | `LsmAudit::emit` |
| `LsmAudit::log_data` post: per-variant dispatch is total (every LSM_AUDIT_DATA_* tag handled) | `LsmAudit::log_data` |
| `LsmAudit::emit_net` post: per-sk_family branch covers AF_INET, AF_INET6 (CONFIG_IPV6), AF_UNIX; other families produce no per-sk fields | `LsmAudit::emit_net` |
| `LsmAudit::emit_net` post: netif > 0 ⇒ dev_put follows dev_get_by_index | `LsmAudit::emit_net` |
| `LsmAudit::ipv4_skb_to_auditdata` post: saddr/daddr always populated; sport/dport populated iff TCP/UDP/SCTP and non-fragment | `LsmAudit::ipv4_skb_to_auditdata` |
| `LsmAudit::ipv6_skb_to_auditdata` post: saddr/daddr always populated; sport/dport populated iff TCP/UDP/SCTP and skb_header_pointer succeeds | `LsmAudit::ipv6_skb_to_auditdata` |
| `print_ipv4_addr` post: emits at most two fields ({name1}=%pI4, {name2}=ntohs(port)) | `print_ipv4_addr` |
| `print_ipv6_addr` post: emits at most two fields ({name1}=%pI6c, {name2}=ntohs(port)) | `print_ipv6_addr` |

### Layer 4: Verus/Creusot functional

`Per-audit-record: caller fills CommonAuditData (type + payload) → common_lsm_audit → audit_log_start(AUDIT_AVC, GFP_ATOMIC) → pre_audit (LSM-specific class/perm/scontext/tcontext) → dump_common_audit_data ( pid + comm + type-specific payload via audit_log_lsm_data ) → post_audit → audit_log_end` is semantically equivalent to the upstream C path, modulo the `BUILD_BUG_ON(sizeof(a->u) ≤ 2 * sizeof(void *))` size invariant proven statically. `Per-skb-to-auditdata` is a pure projection of L3/L4 header fields; equivalence is on a per-(IPv4, IPv6) × (TCP, UDP, SCTP, fragment, other) case basis matching the kernel test corpus from `tools/testing/selftests/audit/`.

### hardening

(Inherits row-1 features from `security/00-overview.md` § Hardening.)

LSM-audit reinforcement:

- **Per-GFP_ATOMIC | __GFP_NOWARN allocation** — defense against per-sleep-in-atomic and per-spurious-warning when audit emits from softirq / spinlock / kprobe / RCU read-side.
- **Per-audit_log_start NULL handled** — defense against per-dropped-record UAF when audit is rate-limited or disabled.
- **Per-BUILD_BUG_ON(sizeof(u) ≤ 2*sizeof(void *))** — defense against per-stack-bloat (caller allocates common_audit_data on stack; large unions would explode kernel stack).
- **Per-network audit on heap (lsm_network_audit *)** — defense against per-stack-overflow on per-packet audit paths.
- **Per-netdev refcount balanced (dev_get_by_index + dev_put)** — defense against per-netdev UAF.
- **Per-init_net netif lookup** — defense against per-netns-sensitive-leak (audit deliberately uses init_net to avoid cross-namespace name spoofing in the record).
- **Per-untrustedstring escaping of comm / kmod_name / key_desc / dentry name / unix sun_path** — defense against per-audit-log injection (control chars, newlines).
- **Per-rcu_read_lock around d_find_alias_rcu** — defense against per-dentry-freed-while-walking on the INODE path.
- **Per-spin_lock(&dentry->d_lock) around d_name access** — defense against per-rename race exposing torn names.
- **Per-smp_load_acquire on unix addr** — defense against per-unix-bind-not-yet-published race (paired with store_release in unix_bind path).
- **Per-fragment handling (ip_hdr frag_off & IP_OFFSET)** — defense against per-bogus-port-field read on non-initial fragments.
- **Per-skb_header_pointer for IPv6 L4** — defense against per-non-linear-skb pull-on-the-fly fault.
- **Per-lockdown_reason indexed lookup (no format-string interpolation of user data)** — defense against per-format-string injection.
- **Per-key_desc untrusted-string escape** — defense against per-key-description audit injection from userspace add_key().

