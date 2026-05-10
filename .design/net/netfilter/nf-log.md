# Tier-3: net/netfilter/nf-log — packet logging (NFLOG / nflog)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_log.c
  - net/netfilter/nf_log_syslog.c
  - net/netfilter/nfnetlink_log.c
  - net/netfilter/nfnetlink.c
  - include/uapi/linux/netfilter/nfnetlink_log.h
-->

## Summary
Tier-3 design for the netfilter packet-logging infrastructure: per-AF logger registry (`nf_log.c`), the syslog backend (`nf_log_syslog.c` — emits to kernel log via `printk`), and the NFLOG userspace backend (`nfnetlink_log.c` — emits to subscribed userspace daemons via NETLINK_NETFILTER + NFNL_SUBSYS_ULOG). When nftables `log` expression or iptables `LOG` / `NFLOG` target fires, the per-AF logger callback formats the packet and emits via the configured backend.

Used by: distro firewall logging (`/var/log/firewall.log` from journald reading kernel log), `ulogd` userspace logging daemon, fwmon, distro intrusion-detection alerting (Wazuh / OSSEC kernel-log parsing).

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/nf-queue.md` (sibling NFNL infrastructure), `net/netfilter/core.md` (no direct hook integration; consumed by individual log targets).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-AF logger registry | `net/netfilter/nf_log.c` |
| Syslog backend (printk) | `net/netfilter/nf_log_syslog.c` |
| NFLOG userspace backend | `net/netfilter/nfnetlink_log.c` |
| Generic NFNL framework | `net/netfilter/nfnetlink.c` |
| UAPI: NFULA_*, NFULNL_MSG_*, NFULNL_CFG_CMD_* | `include/uapi/linux/netfilter/nfnetlink_log.h` |

## Compatibility contract

### Per-AF logger framework

`struct nf_logger`:
```c
struct nf_logger {
    char *name;
    enum nf_log_type type;
    void (*logfn)(struct net *, u_int8_t, unsigned int, const struct sk_buff *, const struct net_device *, const struct net_device *, const struct nf_loginfo *, const char *prefix);
    struct module *me;
};
```

Per-AF + per-type registry: `(NFPROTO_*, NF_LOG_TYPE_LOG | NF_LOG_TYPE_ULOG)` → registered logger. Default per-AF logger settable via `/proc/sys/net/netfilter/nf_log/<af>` sysctl.

Two log types:
- `NF_LOG_TYPE_LOG` — plain syslog/printk
- `NF_LOG_TYPE_ULOG` — NFLOG-encoded for userspace via NFNL

### Syslog backend (`nf_log_syslog.c`)

Formats packet header fields into a printk line:
```
IN=eth0 OUT= MAC=... SRC=1.2.3.4 DST=5.6.7.8 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=12345 PROTO=TCP SPT=8080 DPT=22 SEQ=42 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0
```

Per-AF format: IPv4 / IPv6 / ARP / Bridge each have their own format function. Identical format byte-by-byte so existing log-parsers (`logwatch`, `fail2ban`, distro-specific firewall summaries) work unchanged.

### NFLOG userspace backend (`nfnetlink_log.c`)

Per-(netns, group_num) waiter list (analogous to NFQUEUE per-queue). When NFLOG target fires, kernel encodes packet via NFULA_* NLAs and emits via NFNL multicast (or unicast for bound listener).

#### NFULNL_MSG_* message types

| Message | Direction | Operation |
|---|---|---|
| `NFULNL_MSG_PACKET` | kernel → user | Per-packet log message |
| `NFULNL_MSG_CONFIG` | user → kernel | Bind / configure group |

Wire format byte-identical so libnetfilter_log + ulogd2 work unchanged.

#### NFULA_* NLA attributes

- `NFULA_PACKET_HDR` (struct nfulnl_msg_packet_hdr — hw_protocol + hook)
- `NFULA_MARK`
- `NFULA_TIMESTAMP`
- `NFULA_IFINDEX_INDEV` / `_OUTDEV` / `_PHYSINDEV` / `_PHYSOUTDEV`
- `NFULA_HWADDR`
- `NFULA_PAYLOAD` (variable-length packet bytes per `copy_range`)
- `NFULA_PREFIX` (per-rule string prefix from rule config — `iptables -j LOG --log-prefix`)
- `NFULA_UID` / `_GID`
- `NFULA_SEQ` / `_SEQ_GLOBAL` (per-group + per-host sequence numbers for log-loss detection)
- `NFULA_HWTYPE` / `_HWHEADER` / `_HWLEN` (full L2 header)
- `NFULA_CT` / `_CT_INFO` (conntrack metadata if FLAG_CONNTRACK)
- `NFULA_VLAN` (VLAN tag if applicable)
- `NFULA_L2HDR` (L2 header bytes)

Wire format byte-identical.

#### NFULNL_CFG_CMD_*

- `NFULNL_CFG_CMD_BIND` — bind daemon to group
- `NFULNL_CFG_CMD_UNBIND`
- `NFULNL_CFG_CMD_PF_BIND` (legacy AF-bind)
- `NFULNL_CFG_CMD_PF_UNBIND`

Plus per-group config NLAs:
- `NFULA_CFG_CMD` (above)
- `NFULA_CFG_MODE` (NFULNL_COPY_NONE / META / PACKET + range)
- `NFULA_CFG_NLBUFSIZ`
- `NFULA_CFG_TIMEOUT` (per-group max-defer time before flush)
- `NFULA_CFG_QTHRESH` (per-group max packets before flush)
- `NFULA_CFG_FLAGS` (FLAG_SEQ / FLAG_SEQ_GLOBAL / FLAG_CONNTRACK)

Identical UAPI.

### Per-group batching (NFULA_CFG_QTHRESH + TIMEOUT)

NFLOG batches packets per-group: kernel accumulates packets in a single skb up to `qthresh` packets or `timeout` ms before flushing as a single NFNL multicast. Reduces per-packet syscall + NETLINK overhead.

### Logger fallback chain

If preferred logger unavailable (module not loaded), kernel falls through to next available per-AF logger. Sysctl `nf_log_all_netns` controls per-netns visibility of loggers.

### `iptables -j LOG --log-prefix "DROP_INPUT: "` integration

Per-rule prefix string passed via `NFULA_PREFIX` for NFLOG, or prefixed to printk format for syslog backend. Identical wire format.

### nftables `log` expression integration

`nft add rule ... log group 5 prefix "DROP: "` → emits NFLOG to group 5 with prefix. Per-expression `nft_log` calls into `nf_log_packet` per-AF dispatch.

## Requirements

- REQ-1: Per-AF logger registry (`nf_log_register / _unregister`); per-(NFPROTO_*, NF_LOG_TYPE_*) registration; default per-AF logger settable via sysctl.
- REQ-2: Syslog backend (`nf_log_syslog.c`): formats packet headers into printk-byte-identical text per-AF.
- REQ-3: NFLOG backend (`nfnetlink_log.c`): per-(netns, group_num) state; NFNL_SUBSYS_ULOG transport.
- REQ-4: NFULNL_MSG_* message types parsed identically.
- REQ-5: NFULA_* NLA attributes byte-identical wire format.
- REQ-6: NFULNL_CFG_CMD_* config commands + NFULA_CFG_* attribute set identical.
- REQ-7: Per-group batching: `qthresh` packet count + `timeout` time threshold; coalesced multi-packet messages.
- REQ-8: Per-rule prefix string (NFULA_PREFIX / printk-prefix); identical contract.
- REQ-9: Logger fallback chain: on missing logger, fall through to next available per-AF.
- REQ-10: NFULA_SEQ / NFULA_SEQ_GLOBAL: per-group + per-host monotonic counters for log-loss detection.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `iptables-legacy -A INPUT -j LOG --log-prefix "DROP: "` test: matching packets logged to kernel printk with format byte-identical to upstream's; `dmesg | grep DROP:` shows correctly. (covers REQ-1, REQ-2, REQ-8)
- [ ] AC-2: `iptables-legacy -A INPUT -j NFLOG --nflog-group 5 --nflog-prefix "BLOCKED: "` test: ulogd subscribed to group 5 receives NFULNL_MSG_PACKET with NFULA_PREFIX="BLOCKED: " + NFULA_PAYLOAD. (covers REQ-3, REQ-4, REQ-5)
- [ ] AC-3: nftables log expression test: `nft add rule ... log group 5 prefix "TRACE: "` → ulogd receives NFLOG identically. (covers REQ-3)
- [ ] AC-4: Batching test: NFLOG with qthresh=10 + timeout=100ms; 5 packets quickly + wait → flush at 100ms; 10+ packets quickly → immediate flush. (covers REQ-7)
- [ ] AC-5: Logger fallback test: register only `nf_log_syslog`; nf_log_packet for AF without ULOG-type logger falls through to syslog. (covers REQ-9)
- [ ] AC-6: SEQ tracking test: NFULA_CFG_FLAGS=NFULNL_CFG_F_SEQ; sequential PACKET messages have monotonic NFULA_SEQ. (covers REQ-10)
- [ ] AC-7: NFULA_CT test: NFULA_CFG_FLAGS=NFULNL_CFG_F_CONNTRACK; PACKET messages include NFULA_CT + NFULA_CT_INFO. (covers REQ-5)
- [ ] AC-8: Per-AF format test: IPv4 vs IPv6 packets logged with per-AF specific formats (SRC= IPv4-decimal vs IPv6-hex etc.). (covers REQ-2)
- [ ] AC-9: nf_log_all_netns sysctl test: with =0, loggers per-netns scoped; with =1, visible across netns. (covers REQ-1)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::netfilter::nf_log::Registry` — per-AF + per-type logger registry
- `kernel::net::netfilter::nf_log::Logger` — `struct nf_logger` wrapper
- `kernel::net::netfilter::nf_log::syslog::SyslogLogger` — printk-text backend
- `kernel::net::netfilter::nf_log::syslog::format::Ipv4`, `Ipv6`, `Arp`, `Bridge` — per-AF formatters
- `kernel::net::netfilter::nf_log::ulog::UlogLogger` — NFLOG NFNL backend
- `kernel::net::netfilter::nf_log::ulog::Group` — per-(netns, group_num) state + batch
- `kernel::net::netfilter::nf_log::ulog::PacketBuilder` — NFULA_* NLA build
- `kernel::net::netfilter::nf_log::ulog::Config` — NFULNL_CFG_CMD_*

### Locking and concurrency

- **Per-AF logger registry mutex**: per-AF mutator
- **Per-(netns, group_num) `lock`** (spinlock): protects per-group batch buffer + counters
- **Per-group `qthresh` flush hrtimer**: fires every `timeout` ms or on `qthresh` packets reached
- **RCU**: per-AF logger lookup RCU-side from hot path

### Error handling

- `Err(EEXIST)` — duplicate logger register
- `Err(ENOENT)` — logger not registered (fallback to default)
- `Err(EBUSY)` — group already bound to different daemon
- `Err(EOVERFLOW)` — batch buffer full (counter increments; some logs lost)
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-AF logger registry insert/erase | `kani::proofs::net::netfilter::nf_log::registry_safety` |
| Syslog formatter (printk-text formatting; bounded buffer) | `kani::proofs::net::netfilter::nf_log::syslog_format_safety` |
| NFLOG batch builder (per-skb append + flush atomicity) | `kani::proofs::net::netfilter::nf_log::ulog_batch_safety` |
| Group bind/unbind under group lock | `kani::proofs::net::netfilter::nf_log::ulog_bind_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-AF logger registry | per-(NFPROTO_*, NF_LOG_TYPE_*) at most one default-tagged logger | `kani::proofs::net::netfilter::nf_log::registry_invariants` |
| Per-group batch | batch packet count ≤ qthresh; pending bytes ≤ nlbufsiz | `kani::proofs::net::netfilter::nf_log::batch_invariants` |
| Per-group sequence counter | NFULA_SEQ monotonically increases per-group | `kani::proofs::net::netfilter::nf_log::seq_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/netfilter/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-AF formatter tables + NFNL_SUBSYS_ULOG handler tables `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-emit batch skbs cleared (carries packet payloads, may include sensitive data per pre-NAT info) | § Default-on configurable off |
| **SIZE_OVERFLOW** | batch-buffer + qthresh arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **USERCOPY**: NLA parsing in CONFIG uses bound-checked accessors
- **KERNEXEC**: dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for NFULNL_CFG_CMD_BIND.
- Default GR-RBAC policy: empty.
- DoS-prevention: NFLOG batching + `nlbufsiz` cap prevents log-flood from exhausting kernel memory.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — NFLOG + nf_log_syslog wire format exhaustively specified by upstream + ulogd test corpus + libnetfilter_log integration coverage)

## Out of Scope

- ulogd daemon implementation (userspace; out of scope)
- 32-bit-only paths
- Implementation code
