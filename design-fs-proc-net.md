---
title: "Tier-3: fs/proc/proc-net — /proc/net/* per-netns network state"
tags: ["design-doc", "tier-3", "fs", "procfs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the `/proc/net/*` per-netns rendering — every per-netns network subsystem registers procfs entries here, scoped per the calling task's network namespace. Per-netns enables containers / network-namespace-isolated tasks to see only their own network state (own interfaces, own conntrack, own routes, etc.).

This Tier-3 covers the **infrastructure** (the per-netns directory + framework + lifetime mgmt). The **content** of individual `/proc/net/*` files is owned by per-subsystem Tier-3s:
- `/proc/net/tcp` ↔ `net/ipv4/tcp.md`
- `/proc/net/udp` ↔ `net/ipv4/udp.md`
- `/proc/net/unix` ↔ `net/unix.md`
- `/proc/net/dev` ↔ `net/netdev.md`
- `/proc/net/route` ↔ `net/ipv4/00-overview.md` per FIB
- `/proc/net/ipv6_route` ↔ `net/ipv6/route.md`
- `/proc/net/snmp` ↔ `net/ipv4/00-overview.md` per-MIB
- `/proc/net/snmp6` ↔ `net/ipv6/00-overview.md`
- `/proc/net/nf_conntrack` ↔ `net/netfilter/conntrack-core.md`
- `/proc/net/ip_tables_*`, `/proc/net/ip6_tables_*` ↔ `net/netfilter/iptables.md`
- `/proc/net/packet` ↔ `net/packet.md`
- `/proc/net/raw`, `/proc/net/raw6` ↔ `net/ipv4/raw.md`, `net/ipv6/raw-v6.md`
- `/proc/net/protocols` ↔ `net/00-overview.md`
- `/proc/net/wireless` ↔ `net/wireless/00-overview.md`
- `/proc/net/netlink` ↔ `net/netlink/00-overview.md` (deferred)
- `/proc/net/anycast6`, `/proc/net/dev_mcast`, `/proc/net/igmp`, `/proc/net/sockstat`, `/proc/net/sockstat6`, `/proc/net/stat/`, `/proc/net/netfilter/`, `/proc/net/rt_acct`, `/proc/net/rt_cache`, `/proc/net/arp`, `/proc/net/icmp`, `/proc/net/icmp6`

Sub-tier-3 of `fs/proc/00-overview.md`. Pairs with `net/core/net_namespace.c` (per-netns lifecycle).

### Requirements

- REQ-1: Per-netns `/proc/net/` directory structure: each netns has independent procfs subtree.
- REQ-2: Per-task netns resolution: read of `/proc/net/...` resolves to caller's netns.
- REQ-3: `proc_create_net*` registration family: 4 variants (multi-iter / multi-data / single / single-write); identical signatures.
- REQ-4: `register_pernet_subsys` / `unregister_pernet_subsys` per-subsys lifecycle: init for each existing netns + each new netns; exit on netns destroy.
- REQ-5: Standard /proc/net entries (per the table) registered per-netns; per-subsys content.
- REQ-6: Per-netns GC: RCU-quiesce + per-pernet exit + procfs unregister + struct net free.
- REQ-7: `/proc/net` symlink resolves to `/proc/self/net` (per-task scope).
- REQ-8: Per-netns isolation: netns A sees only A's TCP sockets / IFs / conntrack / etc.
- REQ-9: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `readlink /proc/net` returns `/proc/self/net`. (covers REQ-7)
- [ ] AC-2: Per-netns isolation test: netns A creates TCP socket; netns B's `cat /proc/net/tcp` doesn't show A's socket. (covers REQ-1, REQ-2, REQ-8)
- [ ] AC-3: `proc_create_net` test: synthetic per-netns module registers a procfs file; subsequent `cat /proc/net/<file>` returns synthetic content. (covers REQ-3)
- [ ] AC-4: pernet_operations test: register a `pernet_operations`; create new netns; `init` callback fires for the new netns. (covers REQ-4)
- [ ] AC-5: Standard entries existence: each entry in the table (tcp/udp/unix/packet/dev/route/snmp/etc.) exists in /proc/net. (covers REQ-5)
- [ ] AC-6: Netns destroy test: create netns, register subsys; destroy netns; subsys `exit` fires; per-netns procfs entries gone. (covers REQ-6)
- [ ] AC-7: Subset=pid mount test: with `mount -t proc -o subset=pid proc /proc`, `/proc/net/` not visible. (covers REQ-1)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-9)

### Architecture

### Rust module organization

- `kernel::fs::proc::net::ProcNet` — top-level per-netns directory
- `kernel::fs::proc::net::Resolve` — per-task netns lookup
- `kernel::fs::proc::net::Create` — `proc_create_net*` registration family
- `kernel::fs::proc::net::Pernet` — `pernet_operations` registration framework
- `kernel::fs::proc::net::Lifecycle` — per-netns init/exit
- `kernel::fs::proc::net::Symlink` — `/proc/net` → `/proc/self/net`

### Locking and concurrency

- **`pernet_ops_rwsem`** (rwsem): per-netns subsys list mutator
- **`net_namespace_list_lock`** (mutex): netns lifecycle
- **RCU**: per-netns lookup hot path RCU-side
- **Per-netns `nsfs_mutex`**: per-netns proc dir creation/destruction

### Error handling

- `Err(EACCES)` — permission denied
- `Err(ENOENT)` — netns gone / file not registered
- `Err(EAGAIN)` — netns being torn down
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- Per-subsys file content (cross-ref individual Tier-3s)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/net per-netns dir + per-subsys file registration framework | `fs/proc/proc_net.c` |
| Per-netns lifecycle | `net/core/net_namespace.c` |
| Public API | `include/net/net_namespace.h`, `include/linux/proc_fs.h` |

### compatibility contract

### Per-netns /proc/net directory

For each network namespace (`struct net`), an independent `/proc/net/` virtual directory exists. The directory is per-task-resolved: when a task in netns A reads `/proc/net/`, kernel renders netns-A's view; netns B sees its own. Implemented via `nd_jump_link` / per-task netns-aware procfs lookup.

`/proc/net` is implemented as a symlink to `/proc/self/net` (reading from `/proc/self/task/<tid>/net` resolves to the calling task's netns).

Identical per-netns semantics.

### `proc_create_net*` family of registration functions

Per-subsystem code registers per-netns procfs files via:
```c
struct proc_dir_entry *proc_create_net(const char *name, umode_t mode,
                                        struct proc_dir_entry *parent,
                                        const struct seq_operations *ops,
                                        unsigned int state_size);
struct proc_dir_entry *proc_create_net_data(...);
struct proc_dir_entry *proc_create_net_single(...);
struct proc_dir_entry *proc_create_net_single_write(...);
```

Where:
- `proc_create_net` — multi-line seq_file, per-state-iteration (e.g., per-tcp-socket)
- `proc_create_net_data` — variant with extra per-file private data
- `proc_create_net_single` — single-shot seq_file (e.g., /proc/net/snmp)
- `proc_create_net_single_write` — supports write (e.g., /proc/net/dev_snmp6 reset)

Identical signatures.

### Per-netns subsystem registration (`pernet_operations`)

```c
struct pernet_operations {
    int (*init)(struct net *net);
    void (*exit)(struct net *net);
    void (*exit_batch)(struct list_head *net_exit_list);
    void (*pre_exit)(struct net *net);
    unsigned int *id;
    size_t size;
};

int register_pernet_subsys(struct pernet_operations *ops);
void unregister_pernet_subsys(struct pernet_operations *ops);
```

Per-subsystem registers; `init` is called for each existing netns + each new netns; `exit` for tear-down. Per-netns subsystem state stored at `net->ip4_pernet_data[id]` (or per-subsys-specific extension).

Identical contract.

### Standard /proc/net entries (per-netns rendered)

Across all netns the following entries exist (when their respective subsystems are active):

| Path | Owner Tier-3 | Content |
|---|---|---|
| `/proc/net/tcp`, `tcp6` | `net/ipv4/tcp.md` | Per-socket TCP state |
| `/proc/net/udp`, `udp6`, `udplite` | `net/ipv4/udp.md`, `net/ipv6/udp-v6.md` | Per-socket UDP state |
| `/proc/net/raw`, `raw6` | `net/ipv4/raw.md`, `net/ipv6/raw-v6.md` | Per-socket RAW state |
| `/proc/net/unix` | `net/unix.md` | Per-socket UNIX state |
| `/proc/net/packet` | `net/packet.md` | Per-socket AF_PACKET state |
| `/proc/net/dev` | `net/netdev.md` | Per-iface stats |
| `/proc/net/dev_mcast` | `net/netdev.md` | Per-iface multicast |
| `/proc/net/route` | `net/ipv4/route.md` Tier-3 | IPv4 FIB |
| `/proc/net/ipv6_route` | `net/ipv6/route.md` | IPv6 FIB |
| `/proc/net/arp` | `net/core/00-overview.md` neigh | ARP cache |
| `/proc/net/snmp` | `net/ipv4/00-overview.md` | per-MIB stats |
| `/proc/net/snmp6` | `net/ipv6/00-overview.md` | per-MIB stats v6 |
| `/proc/net/netstat` | `net/ipv4/00-overview.md` | extended stats |
| `/proc/net/sockstat`, `sockstat6` | `net/00-overview.md` | per-AF socket counts |
| `/proc/net/protocols` | `net/00-overview.md` | per-protocol register |
| `/proc/net/nf_conntrack` | `net/netfilter/conntrack-core.md` | conntrack table dump |
| `/proc/net/netfilter/*` | per-subsys netfilter Tier-3 | nfqueue/nflog stats |
| `/proc/net/igmp`, `igmp6` | per-AF mcast Tier-3 | mcast group memberships |
| `/proc/net/anycast6`, `if_inet6` | `net/ipv6/addrconf.md` | v6 addresses + anycast |
| `/proc/net/wireless` | `net/wireless/00-overview.md` (deferred) | per-iface wireless stats |
| `/proc/net/netlink` | `net/netlink/00-overview.md` (deferred) | per-socket NETLINK |
| `/proc/net/stat/*` | per-subsys | per-CPU stats |
| `/proc/net/rt_cache` | `net/ipv4/route.md` | route cache |
| `/proc/net/tcp_sock_*` | TCP probes | TCP-internal state |

Wire format byte-identical for each subsys (per its owning Tier-3).

### Per-netns GC + lifecycle

When a netns is destroyed:
1. RCU-quiesce; pernet `exit` callbacks called per-subsys
2. Per-netns procfs entries unregistered
3. `struct net` freed after RCU sync

Identical lifecycle.

### Hidden netns (hidden from /proc/net for sandboxed views)

When mounted with `subset=pid`, `/proc/net/` may be hidden. Per-netns + per-mount visibility configurable.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-netns lookup (RCU-side; no use-after-free of struct net) | `kani::proofs::fs::proc::net::lookup_safety` |
| `proc_create_net*` registration under pernet_ops_rwsem | `kani::proofs::fs::proc::net::create_safety` |
| Per-netns lifecycle (init→exit pairing) | `kani::proofs::fs::proc::net::lifecycle_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns subsys list | every registered `pernet_operations` has unique `id`; init/exit paired per-netns | `kani::proofs::fs::proc::net::pernet_invariants` |
| Per-netns dir | every netns has own /proc/net directory; no cross-netns leak | `kani::proofs::fs::proc::net::dir_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/proc/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-pernet_operations vtables `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-`struct net` (cross-ref `net/core/00-overview.md`)
- **CONSTIFY**: see above
- **USERCOPY**: seq_file consumers use bound-checked accessors (per-subsys)
- **SIZE_OVERFLOW**: per-netns id arithmetic uses checked operators
- **KERNEXEC**: per-pernet dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` per-file gate.
- LSM hook `security_task_to_inode` per-task /proc/net inode labeling.
- Default useful GR-RBAC policy: deny `/proc/net/nf_conntrack` reads outside gradm-marked `firewall_admin`; reading conntrack reveals all flow tuples on the host.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

