# Tier-3: drivers/nvme/host/fabrics.c — NVMe-over-Fabrics common host code

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nvme/00-overview.md
upstream-paths:
  - drivers/nvme/host/fabrics.c (~1557 lines)
  - drivers/nvme/host/fabrics.h
  - include/linux/nvme.h (nvme_fabrics_command, nvmf_connect_data, ...)
-->

## Summary

`drivers/nvme/host/fabrics.c` is the **transport-agnostic fabrics core** of the NVMe host stack. It owns three concerns: (1) the **option-string parser** that translates a `connect`-style command line (e.g. `nqn=...,traddr=...,trsvcid=...,transport=tcp,...`) into `struct nvmf_ctrl_options`; (2) the **transport-registration registry** (`nvmf_register_transport` / `nvmf_unregister_transport` over `nvmf_transports`); and (3) the **fabrics-command issue surface** — `Connect`, `Property Get`, `Property Set`, `Subsystem Reset`, `Keep-Alive` — that every transport (TCP / RDMA / FC / Loop) submits via its admin queue. Per-controller it provides `nvmf_create_ctrl` (the `/dev/nvme-fabrics` write entry point that fans out to the registered transport's `create_ctrl`), `nvmf_should_reconnect` (the reconnect-budget gate consulted from each transport's error-recovery work), and `nvmf_ip_options_match` (the duplicate-connect detector). Per-NQN it manages a refcounted **host table** (`nvmf_hosts`, `struct nvmf_host` = hostnqn + hostid), so that multiple controllers attached to the same host identity share one `struct nvmf_host`. Per-queue ID 0 (admin) uses `nvmf_connect_admin_queue` (cntlid = 0xFFFF, kato = ctrl->kato * 1000 ms); per-queue qid > 0 uses `nvmf_connect_io_queue` (cntlid = ctrl->cntlid, sqsize = ctrl->sqsize). Critical for: bringing up an NVMe-oF host controller, parsing user-space `nvme connect` arguments, multiplexing transports, enforcing reconnect / loss-timeout policy.

This Tier-3 covers `drivers/nvme/host/fabrics.c` (~1557 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvmf_ctrl_options` | per-connect parsed option set | `NvmfCtrlOptions` |
| `struct nvmf_host` | per-hostnqn / hostid refcounted entry | `NvmfHost` |
| `struct nvmf_transport_ops` | per-transport vtable | `NvmfTransportOps` |
| `nvmf_register_transport()` | per-module registration | `Nvmf::register_transport` |
| `nvmf_unregister_transport()` | per-module unregister | `Nvmf::unregister_transport` |
| `nvmf_lookup_transport()` | per-name resolve | `Nvmf::lookup_transport` |
| `nvmf_create_ctrl()` | per-write fan-out to transport | `Nvmf::create_ctrl` |
| `nvmf_parse_options()` | per-string token parse | `Nvmf::parse_options` |
| `nvmf_check_required_opts()` | per-mask required-bits check | `Nvmf::check_required_opts` |
| `nvmf_check_allowed_opts()` | per-mask allowed-bits check | `Nvmf::check_allowed_opts` |
| `nvmf_free_options()` | per-free | `Nvmf::free_options` |
| `nvmf_host_add()` / `nvmf_host_alloc()` | per-hostnqn intern | `Nvmf::host_add` |
| `nvmf_host_default()` | per-default-hostnqn | `Nvmf::host_default` |
| `nvmf_host_put()` | per-decref | `Nvmf::host_put` |
| `nvmf_reg_read32()` / `_read64()` / `_write32()` | per-property capsule | `Nvmf::reg_read32` / `reg_read64` / `reg_write32` |
| `nvmf_subsystem_reset()` | per-NSSR write | `Nvmf::subsystem_reset` |
| `nvmf_connect_admin_queue()` | per-qid-0 Connect | `Nvmf::connect_admin_queue` |
| `nvmf_connect_io_queue()` | per-qid>0 Connect | `Nvmf::connect_io_queue` |
| `nvmf_connect_data_prep()` | per-Connect payload build | `Nvmf::connect_data_prep` |
| `nvmf_connect_cmd_prep()` | per-Connect SQE build | `Nvmf::connect_cmd_prep` |
| `nvmf_log_connect_error()` | per-failure diagnostic | `Nvmf::log_connect_error` |
| `nvmf_should_reconnect()` | per-status reconnect gate | `Nvmf::should_reconnect` |
| `nvmf_ip_options_match()` | per-duplicate detect | `Nvmf::ip_options_match` |
| `nvmf_ctlr_matches_baseopts()` | per-base-opt compare | `Nvmf::ctlr_matches_baseopts` |
| `nvmf_set_io_queues()` | per-DEFAULT/READ/POLL split | `Nvmf::set_io_queues` |
| `nvmf_map_queues()` | per-tag-set hctx map | `Nvmf::map_queues` |
| `nvmf_get_address()` | per-sysfs traddr print | `Nvmf::get_address` |
| `nvmf_parse_key()` | per-keyring lookup (TLS) | `Nvmf::parse_key` |
| `opt_tokens[]` | per-token table | `OPT_TOKENS` |
| `enum nvmf_parsing_opts` (`NVMF_OPT_*`) | per-token bit | `NvmfOpt` |
| `NVMF_REQUIRED_OPTS` / `NVMF_ALLOWED_OPTS` | per-generic mask | constants |
| `nvmf_dev_write()` | per-write `/dev/nvme-fabrics` | `Nvmf::dev_write` |
| `nvmf_dev_show()` | per-show supported tokens | `Nvmf::dev_show` |
| `nvmf_init()` / `nvmf_exit()` | per-module life | `Nvmf::init` / `exit` |

## Compatibility contract

REQ-1: `struct nvmf_ctrl_options` fields (verbatim, host-visible):
- `mask: u32` — bitmap of seen `NVMF_OPT_*` tokens.
- `max_reconnects: i32` — -1 = infinite; derived from `ctrl_loss_tmo / reconnect_delay`.
- `transport: char*` — owned string ("tcp" | "rdma" | "fc" | "loop").
- `subsysnqn: char*` — target subsystem NQN, `<= NVMF_NQN_SIZE`.
- `traddr: char*` — transport-specific target address.
- `trsvcid: char*` — transport-specific service id (port).
- `host_traddr: char*` — local bind address (optional).
- `host_iface: char*` — local interface (optional).
- `queue_size: usize` — SQ size; default `NVMF_DEF_QUEUE_SIZE`.
- `nr_io_queues: u32` — default `num_online_cpus()`.
- `reconnect_delay: u32` — seconds; default `NVMF_DEF_RECONNECT_DELAY`.
- `discovery_nqn: bool` — set when subsysnqn = `nqn.2014-08.org.nvmexpress.discovery`.
- `duplicate_connect: bool`.
- `kato: u32` — keep-alive timeout in **ms** (wire = `kato * 1000` per `nvmf_connect_cmd_prep`).
- `host: *NvmfHost` — refcounted host identity.
- `dhchap_secret: char*` / `dhchap_ctrl_secret: char*` — DH-HMAC-CHAP keys (kfree_sensitive on free).
- `keyring: *key` / `tls_key: *key` — TLS key references (key_put on free).
- `tls: bool` / `concat: bool` — TCP TLS / secure-concatenation.
- `disable_sqflow: bool`, `hdr_digest: bool`, `data_digest: bool`.
- `nr_write_queues: u32`, `nr_poll_queues: u32`, `tos: i32`, `fast_io_fail_tmo: i32`.

REQ-2: `enum nvmf_parsing_opts` (`NVMF_OPT_*`) — verbatim bit positions, fabrics ABI:
- `NVMF_OPT_ERR = 0`, `NVMF_OPT_TRANSPORT = 1<<0`, `NVMF_OPT_NQN = 1<<1`, `NVMF_OPT_TRADDR = 1<<2`, `NVMF_OPT_TRSVCID = 1<<3`, `NVMF_OPT_QUEUE_SIZE = 1<<4`, `NVMF_OPT_NR_IO_QUEUES = 1<<5`, `NVMF_OPT_TL_RETRY_COUNT = 1<<6`, `NVMF_OPT_KATO = 1<<7`, `NVMF_OPT_HOSTNQN = 1<<8`, `NVMF_OPT_RECONNECT_DELAY = 1<<9`, `NVMF_OPT_HOST_TRADDR = 1<<10`, `NVMF_OPT_CTRL_LOSS_TMO = 1<<11`, `NVMF_OPT_HOST_ID = 1<<12`, `NVMF_OPT_DUP_CONNECT = 1<<13`, `NVMF_OPT_DISABLE_SQFLOW = 1<<14`, `NVMF_OPT_HDR_DIGEST = 1<<15`, `NVMF_OPT_DATA_DIGEST = 1<<16`, `NVMF_OPT_NR_WRITE_QUEUES = 1<<17`, `NVMF_OPT_NR_POLL_QUEUES = 1<<18`, `NVMF_OPT_TOS = 1<<19`, `NVMF_OPT_FAIL_FAST_TMO = 1<<20`, `NVMF_OPT_HOST_IFACE = 1<<21`, `NVMF_OPT_DISCOVERY = 1<<22`, `NVMF_OPT_DHCHAP_SECRET = 1<<23`, `NVMF_OPT_DHCHAP_CTRL_SECRET = 1<<24`, `NVMF_OPT_TLS = 1<<25`, `NVMF_OPT_KEYRING = 1<<26`, `NVMF_OPT_TLS_KEY = 1<<27`, `NVMF_OPT_CONCAT = 1<<28`.

REQ-3: `opt_tokens[]` (verbatim token strings, fabrics ABI):
- `transport=%s` → `NVMF_OPT_TRANSPORT`; `traddr=%s` → `NVMF_OPT_TRADDR`; `trsvcid=%s` → `NVMF_OPT_TRSVCID`; `nqn=%s` → `NVMF_OPT_NQN`; `queue_size=%d` → `NVMF_OPT_QUEUE_SIZE`; `nr_io_queues=%d` → `NVMF_OPT_NR_IO_QUEUES`; `reconnect_delay=%d` → `NVMF_OPT_RECONNECT_DELAY`; `ctrl_loss_tmo=%d` → `NVMF_OPT_CTRL_LOSS_TMO`; `keep_alive_tmo=%d` → `NVMF_OPT_KATO`; `hostnqn=%s` → `NVMF_OPT_HOSTNQN`; `host_traddr=%s` → `NVMF_OPT_HOST_TRADDR`; `host_iface=%s` → `NVMF_OPT_HOST_IFACE`; `hostid=%s` → `NVMF_OPT_HOST_ID`; `duplicate_connect` → `NVMF_OPT_DUP_CONNECT`; `disable_sqflow` → `NVMF_OPT_DISABLE_SQFLOW`; `hdr_digest` → `NVMF_OPT_HDR_DIGEST`; `data_digest` → `NVMF_OPT_DATA_DIGEST`; `nr_write_queues=%d` → `NVMF_OPT_NR_WRITE_QUEUES`; `nr_poll_queues=%d` → `NVMF_OPT_NR_POLL_QUEUES`; `tos=%d` → `NVMF_OPT_TOS`; `fast_io_fail_tmo=%d` → `NVMF_OPT_FAIL_FAST_TMO`; `discovery` → `NVMF_OPT_DISCOVERY`. CONFIG-gated (CONFIG_NVME_TCP_TLS): `keyring=%d`, `tls_key=%d`, `tls`, `concat`. CONFIG-gated (CONFIG_NVME_HOST_AUTH): `dhchap_secret=%s`, `dhchap_ctrl_secret=%s`.

REQ-4: `nvmf_parse_options(opts, buf)`:
- /* Defaults */ opts.queue_size = NVMF_DEF_QUEUE_SIZE; opts.nr_io_queues = num_online_cpus(); opts.reconnect_delay = NVMF_DEF_RECONNECT_DELAY; opts.kato = 0; opts.duplicate_connect = false; opts.fast_io_fail_tmo = NVMF_DEF_FAIL_FAST_TMO; opts.hdr_digest = opts.data_digest = false; opts.tos = -1; opts.tls = opts.concat = false; opts.tls_key = opts.keyring = NULL.
- /* Default hostid + hostnqn */ uuid_copy(hostid, nvmf_default_host.id); strscpy(hostnqn, nvmf_default_host.nqn, NVMF_NQN_SIZE).
- /* Tokenize by ",\n" */ for each non-empty `p` strsep'd from buf-copy:
  - token = match_token(p, opt_tokens, args). opts.mask |= token.
  - Per-token validation:
    - `NQN` → match_strdup → opts.subsysnqn (len < NVMF_NQN_SIZE; discovery NQN ⟹ opts.discovery_nqn = true).
    - `QUEUE_SIZE` → match_int (NVMF_MIN_QUEUE_SIZE ≤ x ≤ NVMF_MAX_QUEUE_SIZE).
    - `NR_IO_QUEUES` → match_int (x ≥ 1).
    - `KATO` → match_int (x ≥ 0); seconds-input stored as ms (`x * 1000`).
    - `CTRL_LOSS_TMO` → match_int (≥ 0 or -1).
    - `HOSTNQN` / `HOSTID` → strdup / uuid_parse; replaces default.
    - `TRADDR` / `TRSVCID` / `HOST_TRADDR` / `HOST_IFACE` → strdup.
    - `HDR_DIGEST` / `DATA_DIGEST` / `DUP_CONNECT` / `DISABLE_SQFLOW` / `DISCOVERY` / `TLS` / `CONCAT` → flag set.
    - `NR_WRITE_QUEUES` / `NR_POLL_QUEUES` → match_int.
    - `TOS` → match_int (0 ≤ x ≤ 255).
    - `FAIL_FAST_TMO` → match_int.
    - `KEYRING` / `TLS_KEY` → match_int(key_id) → `nvmf_parse_key`.
    - `DHCHAP_SECRET` / `DHCHAP_CTRL_SECRET` → strdup.
- /* Post-loop derivation */ opts.max_reconnects = (ctrl_loss_tmo == -1) ? -1 : DIV_ROUND_UP(ctrl_loss_tmo, opts.reconnect_delay).
- /* Host identity intern */ opts.host = nvmf_host_add(hostnqn, hostid).
- Free options copy buffer.

REQ-5: `nvmf_host_add(hostnqn, id)`:
- Locks `nvmf_hosts_mutex`.
- For each `host` in `nvmf_hosts`: if `strcmp(host.nqn, hostnqn) == 0`:
  - if `!uuid_equal(host.id, id)` ⟹ -EINVAL (NQN collision with different ID).
  - kref_get(host.ref); return host.
- Otherwise `nvmf_host_alloc(hostnqn, id)` → kref_init(ref = 1); list_add_tail to `nvmf_hosts`.
- Return host.

REQ-6: `nvmf_host_default()`:
- Build hostnqn = `"nqn.2014-08.org.nvmexpress:uuid:<random uuid>"`.
- Generate UUID via uuid_gen.
- `nvmf_host_alloc(hostnqn, &uuid)`.
- Stored in `nvmf_default_host` at module init.

REQ-7: `nvmf_register_transport(ops)`:
- if !ops.create_ctrl ⟹ -EINVAL.
- down_write(nvmf_transports_rwsem). list_add_tail(ops.entry, nvmf_transports). up_write.
- return 0.

REQ-8: `nvmf_unregister_transport(ops)`:
- down_write. list_del(ops.entry). up_write.

REQ-9: `nvmf_lookup_transport(opts)`:
- lockdep_assert_held(nvmf_transports_rwsem).
- linear search `nvmf_transports` for ops where `strcmp(ops.name, opts.transport) == 0`.

REQ-10: `nvmf_create_ctrl(dev, buf)`:
- opts = kzalloc.
- `nvmf_parse_options(opts, buf)`.
- `request_module("nvme-%s", opts.transport)`.
- `nvmf_check_required_opts(opts, NVMF_REQUIRED_OPTS = NVMF_OPT_TRANSPORT | NVMF_OPT_NQN)`.
- opts.mask &= ~NVMF_REQUIRED_OPTS (consumed; transport need not re-check).
- down_read(nvmf_transports_rwsem).
- ops = `nvmf_lookup_transport(opts)`; if !ops ⟹ -EINVAL.
- if !try_module_get(ops.module) ⟹ -EBUSY.
- up_read.
- `nvmf_check_required_opts(opts, ops.required_opts)`.
- `nvmf_check_allowed_opts(opts, NVMF_ALLOWED_OPTS | ops.allowed_opts | ops.required_opts)`.
- ctrl = `ops.create_ctrl(dev, opts)` (transport-specific).
- module_put(ops.module).
- on err: `nvmf_free_options(opts)`.

REQ-11: `NVMF_REQUIRED_OPTS` = `NVMF_OPT_TRANSPORT | NVMF_OPT_NQN`. `NVMF_ALLOWED_OPTS` = `NVMF_OPT_QUEUE_SIZE | NVMF_OPT_NR_IO_QUEUES | NVMF_OPT_KATO | NVMF_OPT_HOSTNQN | NVMF_OPT_HOST_ID | NVMF_OPT_DUP_CONNECT | NVMF_OPT_DISABLE_SQFLOW | NVMF_OPT_DISCOVERY | NVMF_OPT_FAIL_FAST_TMO | NVMF_OPT_DHCHAP_SECRET | NVMF_OPT_DHCHAP_CTRL_SECRET`.

REQ-12: `nvmf_check_required_opts(opts, required)`:
- if (opts.mask & required) != required ⟹ for each token in opt_tokens with (token & required) ∧ !(token & opts.mask): pr_warn("missing parameter '%s'", token.pattern); return -EINVAL.
- else 0.

REQ-13: `nvmf_check_allowed_opts(opts, allowed)`:
- if (opts.mask & ~allowed) ⟹ for each token in opt_tokens with (token & opts.mask) ∧ (token & ~allowed): pr_warn("invalid parameter '%s'", token.pattern); return -EINVAL.
- else 0.

REQ-14: `nvmf_connect_cmd_prep(ctrl, qid, cmd)`:
- cmd.connect.opcode = `nvme_fabrics_command`.
- cmd.connect.fctype = `nvme_fabrics_type_connect`.
- cmd.connect.qid = cpu_to_le16(qid).
- if qid != 0: cmd.connect.sqsize = cpu_to_le16(ctrl.sqsize).
- else: cmd.connect.sqsize = cpu_to_le16(NVME_AQ_DEPTH - 1); cmd.connect.kato = cpu_to_le32(ctrl.kato * 1000) (ms → wire ms).
- if ctrl.opts.disable_sqflow: cmd.connect.cattr |= `NVME_CONNECT_DISABLE_SQFLOW`.

REQ-15: `nvmf_connect_data_prep(ctrl, cntlid)`:
- data = kzalloc(sizeof(`nvmf_connect_data`)).
- uuid_copy(data.hostid, ctrl.opts.host.id).
- data.cntlid = cpu_to_le16(cntlid). /* 0xFFFF for admin = "any" */
- strscpy(data.subsysnqn, ctrl.opts.subsysnqn, NVMF_NQN_SIZE).
- strscpy(data.hostnqn, ctrl.opts.host.nqn, NVMF_NQN_SIZE).

REQ-16: `nvmf_connect_admin_queue(ctrl)`:
- `nvmf_connect_cmd_prep(ctrl, 0, &cmd)`.
- data = `nvmf_connect_data_prep(ctrl, 0xffff)`; if !data ⟹ -ENOMEM.
- ret = `__nvme_submit_sync_cmd(ctrl.fabrics_q, &cmd, &res, data, sizeof(data), NVME_QID_ANY, NVME_SUBMIT_AT_HEAD | NVME_SUBMIT_NOWAIT | NVME_SUBMIT_RESERVED)`.
- if ret: `nvmf_log_connect_error(ctrl, ret, le32(res.u32), &cmd, data)`; goto out.
- result = le32(res.u32). ctrl.cntlid = result & 0xFFFF.
- if result & (`NVME_CONNECT_AUTHREQ_ATR` | `NVME_CONNECT_AUTHREQ_ASCR`):
  - if (result & ASCR) ∧ !ctrl.opts.concat ⟹ -EOPNOTSUPP ("secure concatenation not supported").
  - `nvme_auth_negotiate(ctrl, 0)`; `nvme_auth_wait(ctrl, 0)`.
- kfree(data).

REQ-17: `nvmf_connect_io_queue(ctrl, qid)`:
- `nvmf_connect_cmd_prep(ctrl, qid, &cmd)`.
- data = `nvmf_connect_data_prep(ctrl, ctrl.cntlid)`.
- submit on `ctrl.connect_q`.
- if auth required ⟹ `nvme_auth_negotiate(ctrl, qid)` + `nvme_auth_wait(ctrl, qid)`.

REQ-18: `nvmf_reg_read32(ctrl, off, *val)`:
- cmd.prop_get.opcode = nvme_fabrics_command; fctype = nvme_fabrics_type_property_get; offset = cpu_to_le32(off).
- `__nvme_submit_sync_cmd(ctrl.fabrics_q, ...)`; if ret ≥ 0 ⟹ *val = le64_to_cpu(res.u64).
- Same for `_read64` (additional attrib=1) and `_write32` (prop_set, value, attrib=0).

REQ-19: `nvmf_subsystem_reset(ctrl)`:
- if !nvme_wait_reset(ctrl) ⟹ -EBUSY.
- ctrl.ops.reg_write32(ctrl, `NVME_REG_NSSR`, `NVME_SUBSYS_RESET`).
- `nvme_try_sched_reset(ctrl)`.

REQ-20: `nvmf_should_reconnect(ctrl, status)`:
- if status > 0 ∧ (status & `NVME_STATUS_DNR`) ⟹ false (Do-Not-Retry set).
- if status == -EKEYREJECTED ∨ status == -ENOKEY ⟹ false.
- if ctrl.opts.max_reconnects == -1 ∨ ctrl.nr_reconnects < ctrl.opts.max_reconnects ⟹ true.
- else false.

REQ-21: `nvmf_ip_options_match(ctrl, opts)`:
- if !`nvmf_ctlr_matches_baseopts(ctrl, opts)` ∨ strcmp(traddr) ≠ 0 ∨ strcmp(trsvcid) ≠ 0 ⟹ false.
- host_traddr / host_iface comparison: equal-or-both-absent ⟹ true; one-sided ⟹ false; both-present-and-different ⟹ false.

REQ-22: `nvmf_ctlr_matches_baseopts(ctrl, opts)`:
- state ≠ DELETING / DELETING_NOIO / DEAD.
- strcmp(opts.subsysnqn, ctrl.opts.subsysnqn) == 0.
- ctrl.opts.host == opts.host (interned identity).
- ctrl.opts.duplicate_connect == false.

REQ-23: `nvmf_set_io_queues(opts, nr_io_queues, io_queues[HCTX_MAX_TYPES])`:
- if opts.nr_write_queues > 0 ∧ opts.nr_io_queues < nr_io_queues (separate read/write):
  - io_queues[READ] = opts.nr_io_queues; nr_io_queues -= io_queues[READ].
  - io_queues[DEFAULT] = min(opts.nr_write_queues, nr_io_queues); nr_io_queues -= io_queues[DEFAULT].
- else (shared): io_queues[DEFAULT] = min(opts.nr_io_queues, nr_io_queues).
- if opts.nr_poll_queues > 0 ∧ nr_io_queues > 0: io_queues[POLL] = min(opts.nr_poll_queues, nr_io_queues).

REQ-24: `nvmf_map_queues(set, ctrl, io_queues)`:
- separate-queues path: set.map[DEFAULT].{nr_queues=io_queues[DEFAULT], queue_offset=0}; set.map[READ].{nr_queues=io_queues[READ], queue_offset=io_queues[DEFAULT]}.
- shared-queues path: both DEFAULT and READ map onto io_queues[DEFAULT] at offset 0.
- blk_mq_map_queues(DEFAULT). blk_mq_map_queues(READ).
- if nr_poll_queues > 0 ∧ io_queues[POLL] > 0: set.map[POLL].{nr_queues=io_queues[POLL], queue_offset=io_queues[DEFAULT]+io_queues[READ]}; blk_mq_map_queues(POLL).
- dev_info "mapped X/Y/Z default/read/poll queues."

REQ-25: `nvmf_free_options(opts)`:
- nvmf_host_put(opts.host).
- key_put(opts.keyring); key_put(opts.tls_key).
- kfree(transport, traddr, trsvcid, subsysnqn, host_traddr, host_iface).
- kfree_sensitive(dhchap_secret, dhchap_ctrl_secret) (memzero before free).
- kfree(opts).

REQ-26: `nvmf_get_address(ctrl, buf, size)`:
- Append "traddr=%s" if mask&TRADDR; ",trsvcid=%s" if TRSVCID; ",host_traddr=%s" if HOST_TRADDR; ",host_iface=%s" if HOST_IFACE; trailing "\n". Returns length.
- Wired as `.get_address` for fabrics transports.

REQ-27: `/dev/nvme-fabrics` misc device (`nvmf_misc`, `nvmf_dev_fops`):
- write(buf) ⟹ `nvmf_dev_write` ⟹ `nvmf_create_ctrl(nvmf_device, buf)` under `nvmf_dev_mutex`.
- read ⟹ `nvmf_dev_show` ⟹ prints `instance=%d,cntlid=...,transport=...` lines per controller via `__nvmf_concat_opt_tokens`.
- open allocates seq_file private; release frees it.

REQ-28: Module life (`nvmf_init` / `nvmf_exit`):
- nvmf_init: nvmf_host_default ⟹ nvmf_default_host. class_register(nvmf_class). nvmf_device = device_create(... "ctl"). misc_register(nvmf_misc).
- nvmf_exit: misc_deregister. device_destroy. class_unregister. nvmf_host_put(nvmf_default_host). WARN_ON !list_empty(nvmf_transports).

## Acceptance Criteria

- [ ] AC-1: write to `/dev/nvme-fabrics` of "nqn=...,transport=tcp,traddr=...,trsvcid=..." ⟹ `nvmf_create_ctrl` succeeds → returns ctrl.
- [ ] AC-2: missing `transport=` ⟹ `nvmf_check_required_opts` returns -EINVAL with "missing parameter 'transport=%s'".
- [ ] AC-3: `transport=unknown` ⟹ `nvmf_lookup_transport` returns NULL ⟹ -EINVAL after `request_module`.
- [ ] AC-4: unknown token like `bogus=1` ⟹ `nvmf_check_allowed_opts` returns -EINVAL with "invalid parameter".
- [ ] AC-5: `keep_alive_tmo=30` ⟹ opts.kato = 30000 (seconds → ms); admin Connect cmd.kato = 30000000 (ms → wire ms).
- [ ] AC-6: two `nvmf_host_add` with same NQN + same UUID ⟹ same `nvmf_host*` (refcount++); different UUID ⟹ -EINVAL.
- [ ] AC-7: `nvmf_connect_admin_queue` sends opcode = `nvme_fabrics_command`, fctype = `nvme_fabrics_type_connect`, qid=0, sqsize = NVME_AQ_DEPTH-1.
- [ ] AC-8: `nvmf_connect_admin_queue` records ctrl.cntlid = result & 0xFFFF on success.
- [ ] AC-9: server returns AUTHREQ_ATR ⟹ host invokes `nvme_auth_negotiate` + `_wait` before returning success.
- [ ] AC-10: server returns AUTHREQ_ASCR with concat disabled ⟹ -EOPNOTSUPP.
- [ ] AC-11: `nvmf_should_reconnect` returns false when status has `NVME_STATUS_DNR` set.
- [ ] AC-12: `nvmf_should_reconnect` returns false when max_reconnects reached; true when max_reconnects == -1.
- [ ] AC-13: `nvmf_ip_options_match` returns false when one side specifies `host_traddr` and the other does not.
- [ ] AC-14: `nvmf_set_io_queues(opts={nr_write=4, nr_io=8}, nr_io_queues=10)` ⟹ io_queues[READ]=8, [DEFAULT]=2, [POLL]=0.
- [ ] AC-15: `nvmf_free_options`: dhchap_secret memory zeroed before kfree (kfree_sensitive).
- [ ] AC-16: `nvmf_register_transport` with `ops.create_ctrl == NULL` ⟹ -EINVAL.

## Architecture

```
struct NvmfHost {
  nqn: [u8; NVMF_NQN_SIZE],
  id:  Uuid,
  ref: Kref,
  list: ListNode,
}

struct NvmfCtrlOptions {
  mask: u32,                  // bitmap of NvmfOpt seen
  max_reconnects: i32,
  transport: KString,
  subsysnqn: KString,
  traddr: KString,
  trsvcid: KString,
  host_traddr: Option<KString>,
  host_iface: Option<KString>,
  queue_size: usize,
  nr_io_queues: u32,
  reconnect_delay: u32,
  discovery_nqn: bool,
  duplicate_connect: bool,
  kato: u32,                  // milliseconds
  host: KrefHandle<NvmfHost>,
  dhchap_secret: Option<Sensitive<KString>>,
  dhchap_ctrl_secret: Option<Sensitive<KString>>,
  keyring: Option<KeyRef>,
  tls_key: Option<KeyRef>,
  tls: bool,
  concat: bool,
  disable_sqflow: bool,
  hdr_digest: bool,
  data_digest: bool,
  nr_write_queues: u32,
  nr_poll_queues: u32,
  tos: i32,
  fast_io_fail_tmo: i32,
}

struct NvmfTransportOps {
  module: *Module,
  name: &'static str,
  required_opts: u32,         // mask of NvmfOpt
  allowed_opts: u32,
  create_ctrl: fn(dev: &Device, opts: NvmfCtrlOptions) -> KResult<NvmeCtrl>,
  entry: ListNode,
}
```

`Nvmf::create_ctrl(dev, buf) -> KResult<NvmeCtrl>`:
1. opts = NvmfCtrlOptions::default().
2. Nvmf::parse_options(&mut opts, buf)?.
3. request_module("nvme-{}", opts.transport).
4. Nvmf::check_required_opts(&opts, NVMF_REQUIRED_OPTS)?.
5. opts.mask &= !NVMF_REQUIRED_OPTS.
6. let _g = NVMF_TRANSPORTS.read().
7. ops = Nvmf::lookup_transport(&opts).ok_or(EINVAL)?.
8. if !ops.module.try_get() { return -EBUSY }.
9. drop(_g).
10. Nvmf::check_required_opts(&opts, ops.required_opts)?.
11. Nvmf::check_allowed_opts(&opts, NVMF_ALLOWED_OPTS | ops.allowed_opts | ops.required_opts)?.
12. ctrl = (ops.create_ctrl)(dev, opts)?.
13. ops.module.put().
14. Ok(ctrl).

`Nvmf::parse_options(opts, buf) -> KResult<()>`:
1. Set defaults (queue_size, nr_io_queues, reconnect_delay, kato=0, etc.).
2. Generate default hostid + hostnqn from `NVMF_DEFAULT_HOST`.
3. for p in buf.split(",\n"):
   - token = match_token(p, OPT_TOKENS).
   - opts.mask |= token.
   - dispatch per-token: store value into opts.<field>, validating bounds (queue_size ∈ [NVMF_MIN_QUEUE_SIZE, NVMF_MAX_QUEUE_SIZE], etc.).
4. Compute opts.max_reconnects = if ctrl_loss_tmo == -1 { -1 } else { div_round_up(ctrl_loss_tmo, opts.reconnect_delay) }.
5. opts.host = Nvmf::host_add(hostnqn, hostid)?.
6. Ok(()).

`Nvmf::connect_admin_queue(ctrl) -> KResult<()>`:
1. let cmd = NvmeCommand::default().
2. Nvmf::connect_cmd_prep(ctrl, 0, &mut cmd).
3. data = Nvmf::connect_data_prep(ctrl, 0xFFFF)?.
4. res = ctrl.fabrics_q.submit_sync(&cmd, &data, NVME_QID_ANY, AT_HEAD | NOWAIT | RESERVED)?.
5. ctrl.cntlid = res.u32 & 0xFFFF.
6. if res & (AUTHREQ_ATR | AUTHREQ_ASCR) != 0:
   - if (res & AUTHREQ_ASCR) ∧ !ctrl.opts.concat ⟹ return -EOPNOTSUPP.
   - nvme_auth_negotiate(ctrl, 0)?. nvme_auth_wait(ctrl, 0)?.
7. Ok(()).

`Nvmf::should_reconnect(ctrl, status) -> bool`:
1. if status > 0 ∧ (status & NVME_STATUS_DNR) ≠ 0 ⟹ false.
2. if status == -EKEYREJECTED ∨ status == -ENOKEY ⟹ false.
3. if ctrl.opts.max_reconnects == -1 ∨ ctrl.nr_reconnects < ctrl.opts.max_reconnects ⟹ true.
4. else false.

`Nvmf::ip_options_match(ctrl, opts) -> bool`:
1. if !Nvmf::ctlr_matches_baseopts(ctrl, opts) ⟹ false.
2. if strcmp(opts.traddr, ctrl.opts.traddr) ≠ 0 ⟹ false.
3. if strcmp(opts.trsvcid, ctrl.opts.trsvcid) ≠ 0 ⟹ false.
4. compare host_traddr & host_iface (both-absent OK; one-sided ⟹ false; both-present-different ⟹ false).
5. Ok(true).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `parse_options_no_alias` | INVARIANT | per-parse_options: each strdup result owned by exactly one `opts.<field>`; replacing rewrites old kfree. |
| `host_kref_balanced` | INVARIANT | per-host_add/host_put: kref get'd on every duplicate-NQN, put'd on every nvmf_free_options. |
| `required_mask_subset` | INVARIANT | per-create_ctrl: after check_required_opts(NVMF_REQUIRED_OPTS), opts.mask & NVMF_REQUIRED_OPTS == NVMF_REQUIRED_OPTS. |
| `module_get_balanced` | INVARIANT | per-create_ctrl: try_module_get on success ⟹ module_put on all return paths. |
| `kato_ms_units` | INVARIANT | per-parse_options KATO: opts.kato stored in **ms** (sec_input * 1000); connect_cmd_prep multiplies by 1000 to wire. |
| `transport_rwsem_lock_order` | INVARIANT | per-lookup_transport: rwsem held shared during list traversal; held exclusive during list_add/del. |
| `connect_data_clear_on_err` | INVARIANT | per-connect_admin_queue: data buffer kfree'd on every return path. |

### Layer 2: TLA+

`drivers/nvme/host/fabrics.tla`:
- States: ctrl ∈ {parse, lookup, module_get, create_ctrl, connect_admin, connect_io, live, reconnect, delete}.
- Properties:
  - `safety_required_opts_present_before_create_ctrl` — per-create_ctrl: opts.mask has TRANSPORT ∧ NQN.
  - `safety_module_refcount_balanced` — per-create_ctrl: try_module_get matches module_put.
  - `safety_host_ref_balanced` — per-options-lifecycle: host kref +1 per opts ⟹ -1 on free.
  - `safety_dnr_no_reconnect` — per-status with NVME_STATUS_DNR: should_reconnect ≡ false.
  - `safety_max_reconnects_eventually_delete` — per-reconnect: nr_reconnects ≥ max ⟹ delete-ctrl.
  - `liveness_parse_terminates` — per-parse_options: bounded by string length.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Nvmf::create_ctrl` post: returned ctrl backed by valid transport ops, mask ⊇ required | `Nvmf::create_ctrl` |
| `Nvmf::parse_options` post: opts.mask reflects exactly the tokens seen | `Nvmf::parse_options` |
| `Nvmf::check_required_opts` post: (opts.mask & required) == required ⟺ Ok | `Nvmf::check_required_opts` |
| `Nvmf::check_allowed_opts` post: (opts.mask & !allowed) == 0 ⟺ Ok | `Nvmf::check_allowed_opts` |
| `Nvmf::host_add` post: returned host has nqn == hostnqn ∧ id == hostid | `Nvmf::host_add` |
| `Nvmf::connect_cmd_prep` post (qid=0): cmd.connect.{opcode, fctype, qid, sqsize, kato} fully set | `Nvmf::connect_cmd_prep` |
| `Nvmf::should_reconnect` post: DNR ∨ EKEYREJECTED ∨ ENOKEY ⟹ false | `Nvmf::should_reconnect` |
| `Nvmf::free_options` post: dhchap_secret and dhchap_ctrl_secret zeroed | `Nvmf::free_options` |

### Layer 4: Verus/Creusot functional

`Per-user write to /dev/nvme-fabrics → parse_options → lookup_transport → (transport).create_ctrl → connect_admin_queue (qid=0, cntlid=0xFFFF) → property_get(VS, CC, CSTS) → enable_ctrl → connect_io_queue(qid=1..N) → CTRL_LIVE` semantic equivalence: per-NVMe-over-Fabrics spec rev 1.1a §3.3 (Connect command).

`Per-status-code branch (DNR / EKEYREJECTED / max_reconnects) → reconnect_or_remove` semantic equivalence: per-Documentation/nvme/feature-and-quirk-policy.rst + per-`drivers/nvme/host/fabrics.c::nvmf_should_reconnect`.

## Hardening

(Inherits row-1 features from `drivers/nvme/00-overview.md` § Hardening.)

NVMe-oF common reinforcement:

- **Per-NVMF_REQUIRED_OPTS enforcement at create_ctrl** — defense against per-half-baked controller (missing transport/NQN).
- **Per-NVMF_ALLOWED_OPTS reject of unknown tokens** — defense against per-typo silently dropping option (e.g., `traddr` instead of `traddrs`).
- **Per-nvmf_host NQN+UUID intern strict** — defense against per-NQN collision with different UUID (returns -EINVAL).
- **Per-NVMF_NQN_SIZE strncpy-bound** — defense against per-NQN over-read / over-write.
- **Per-kfree_sensitive on dhchap_secret / dhchap_ctrl_secret** — defense against per-credential leak via free-list reuse.
- **Per-key_put on keyring / tls_key paths** — defense against per-TLS-key refleak.
- **Per-try_module_get / module_put pairing across create_ctrl** — defense against per-module rmmod while controller exists.
- **Per-transport rwsem (down_read in lookup, down_write in register/unregister)** — defense against per-transport-list mutation race.
- **Per-NVME_STATUS_DNR honored in should_reconnect** — defense against per-pathological reconnect loop on unrecoverable error.
- **Per-EKEYREJECTED / ENOKEY non-reconnect** — defense against per-stale-PSK reconnect loop.
- **Per-kato ms-vs-sec unit discipline (`opts.kato * 1000`)** — defense against per-unit confusion (kato is **seconds** wire, **ms** internal here, **ms*1000** on wire).
- **Per-disable_sqflow gated by CONNECT_DISABLE_SQFLOW cattr** — defense against per-flow-control violation.
- **Per-discovery NQN auto-tag (opts.discovery_nqn)** — defense against per-discovery-vs-IO command-set confusion.
- **Per-/dev/nvme-fabrics under nvmf_dev_mutex** — defense against per-concurrent-create_ctrl race on shared device.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/nvme/host/core.c` controller lifecycle (covered in `host-core.md` Tier-3).
- `drivers/nvme/host/tcp.c` TCP transport (covered in `host-tcp.md` Tier-3).
- `drivers/nvme/host/rdma.c` RDMA transport (covered separately if expanded).
- `drivers/nvme/host/fc.c` Fibre Channel transport (covered separately if expanded).
- `drivers/nvme/host/auth.c` DH-HMAC-CHAP (covered separately if expanded).
- TLS handshake protocol (`drivers/nvme/host/tcp.c::nvme_tcp_start_tls`, covered in `host-tcp.md`).
- Implementation code.
