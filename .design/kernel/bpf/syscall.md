# Tier-3: kernel/bpf/syscall.c — bpf(2) syscall surface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/syscall.c (~6614 lines)
  - include/uapi/linux/bpf.h (union bpf_attr, enum bpf_cmd, enum bpf_map_type, enum bpf_prog_type, enum bpf_attach_type, enum bpf_link_type)
  - include/linux/bpf.h (struct bpf_prog, struct bpf_map, struct bpf_link, struct bpf_token, struct bpf_prog_aux)
  - include/linux/filter.h (struct bpf_prog layout)
-->

## Summary

The `bpf(2)` syscall is the single multiplexed entry point for all eBPF object management. `SYSCALL_DEFINE3(bpf, int cmd, union bpf_attr __user *uattr, unsigned int size)` copies a variable-size `union bpf_attr` from userspace (tail-zero-checked via `bpf_check_uarg_tail_zero`), invokes `security_bpf(cmd, &attr, size, ...)`, and dispatches on `cmd` to one of ~36 per-command handlers in `__sys_bpf()`. Object kinds: **prog** (verified BPF bytecode + JIT), **map** (typed kv container), **link** (an attachment of a prog to a hook with a refcounted lifetime), **btf** (BPF Type Format blob), **token** (delegated capability bundle pinned in a bpffs). Capability gating: `CAP_BPF` (base prog/map ops), `CAP_NET_ADMIN` (networking prog/map types), `CAP_PERFMON` (tracing / perf-event progs), `CAP_SYS_ADMIN` (privileged ops + BTF-fd-by-id). Per-`sysctl_unprivileged_bpf_disabled = 1`: unprivileged prog/map creation blocked. Per-`BPF_F_TOKEN_FD` flag: capability check goes through `bpf_token_capable()` against a `struct bpf_token` instead of `current_cred()`. Critical for: containerized BPF delegation, libbpf userspace, bpftool, observability stack (perf, tracing, networking).

This Tier-3 covers `kernel/bpf/syscall.c` (~6614 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE3(bpf, ...)` | per-syscall entry | `Bpf::sys_bpf` |
| `__sys_bpf(cmd, uattr, size)` | per-dispatch | `Bpf::sys_bpf_inner` |
| `bpf_check_uarg_tail_zero()` | per-tail-zero check | `Bpf::check_uarg_tail_zero` |
| `map_create()` | per-BPF_MAP_CREATE | `Bpf::map_create` |
| `map_lookup_elem()` | per-BPF_MAP_LOOKUP_ELEM | `Bpf::map_lookup_elem` |
| `map_update_elem()` | per-BPF_MAP_UPDATE_ELEM | `Bpf::map_update_elem` |
| `map_delete_elem()` | per-BPF_MAP_DELETE_ELEM | `Bpf::map_delete_elem` |
| `map_get_next_key()` | per-BPF_MAP_GET_NEXT_KEY | `Bpf::map_get_next_key` |
| `map_freeze()` | per-BPF_MAP_FREEZE | `Bpf::map_freeze` |
| `map_lookup_and_delete_elem()` | per-BPF_MAP_LOOKUP_AND_DELETE_ELEM | `Bpf::map_lookup_and_delete_elem` |
| `bpf_map_do_batch()` | per-BPF_MAP_*_BATCH | `Bpf::map_do_batch` |
| `bpf_prog_load()` | per-BPF_PROG_LOAD | `Bpf::prog_load` |
| `bpf_prog_alloc()` / `bpf_prog_alloc_id()` | per-alloc/IDR | `Bpf::prog_alloc` / `prog_alloc_id` |
| `bpf_check()` | per-verifier (covered in verifier.md) | `Bpf::verifier_check` |
| `bpf_prog_new_fd()` / `bpf_prog_release()` | per-fd lifecycle | `Bpf::prog_new_fd` / `prog_release` |
| `__bpf_prog_put()` / `__bpf_prog_put_noref()` | per-refcount | `Bpf::prog_put` / `prog_put_noref` |
| `bpf_audit_prog()` | per-audit LOAD/UNLOAD | `Bpf::prog_audit` |
| `bpf_prog_kallsyms_add()` | per-symbol exposure | `Bpf::prog_kallsyms_add` |
| `bpf_obj_pin()` / `bpf_obj_get()` | per-bpffs pin/get (delegate to inode.c) | `Bpf::obj_pin` / `obj_get` |
| `bpf_prog_attach()` / `bpf_prog_detach()` | per-legacy attach | `Bpf::prog_attach` / `prog_detach` |
| `bpf_prog_attach_check_attach_type()` | per-attach-type validation | `Bpf::prog_attach_check_type` |
| `bpf_prog_query()` | per-BPF_PROG_QUERY | `Bpf::prog_query` |
| `bpf_prog_test_run()` | per-BPF_PROG_TEST_RUN | `Bpf::prog_test_run` |
| `bpf_obj_get_next_id()` | per-IDR walk | `Bpf::obj_get_next_id` |
| `bpf_prog_get_fd_by_id()` / `bpf_map_get_fd_by_id()` / `bpf_link_get_fd_by_id()` / `bpf_btf_get_fd_by_id()` | per-by-id fd | `Bpf::*_get_fd_by_id` |
| `bpf_obj_get_info_by_fd()` | per-BPF_OBJ_GET_INFO_BY_FD | `Bpf::obj_get_info_by_fd` |
| `bpf_prog_get_info_by_fd()` / `bpf_map_get_info_by_fd()` / `bpf_btf_get_info_by_fd()` / `bpf_link_get_info_by_fd()` | per-info-by-fd | `Bpf::*_get_info_by_fd` |
| `bpf_raw_tracepoint_open()` | per-BPF_RAW_TRACEPOINT_OPEN | `Bpf::raw_tracepoint_open` |
| `bpf_btf_load()` | per-BPF_BTF_LOAD | `Bpf::btf_load` |
| `bpf_task_fd_query()` | per-BPF_TASK_FD_QUERY | `Bpf::task_fd_query` |
| `link_create()` / `link_update()` / `link_detach()` | per-BPF_LINK_* | `Bpf::link_create` / `link_update` / `link_detach` |
| `bpf_link_prime()` / `bpf_link_settle()` / `bpf_link_cleanup()` | per-link 2-phase install | `Bpf::link_prime` / `link_settle` / `link_cleanup` |
| `bpf_link_new_fd()` / `bpf_link_release()` / `bpf_link_put()` | per-link fd lifecycle | `Bpf::link_new_fd` / `link_release` / `link_put` |
| `bpf_enable_stats()` | per-BPF_ENABLE_STATS | `Bpf::enable_stats` |
| `bpf_iter_create()` | per-BPF_ITER_CREATE | `Bpf::iter_create` |
| `bpf_prog_bind_map()` | per-BPF_PROG_BIND_MAP | `Bpf::prog_bind_map` |
| `token_create()` / `bpf_token_create()` | per-BPF_TOKEN_CREATE | `Bpf::token_create` |
| `bpf_token_capable()` / `bpf_token_allow_cmd()` / `bpf_token_allow_prog_type()` / `bpf_token_allow_map_type()` | per-token gate | `Bpf::token_capable` / `token_allow_*` |
| `bpf_token_get_from_fd()` / `bpf_token_put()` | per-token refcount | `Bpf::token_get_from_fd` / `token_put` |
| `BPF_CALL_3(bpf_sys_bpf, ...)` | per-syscall-from-prog helper | `Bpf::sys_bpf_helper` |
| `kern_sys_bpf()` | per-kernel-skeleton entry | `Bpf::kern_sys_bpf` |
| `prog_idr` / `map_idr` / `link_idr` / `btf_idr` | per-ID space | `IdrSpace::{prog,map,link,btf}` |

## Compatibility contract

REQ-1: bpf(2) entry & uattr ABI:
- `SYSCALL_DEFINE3(bpf, int cmd, union bpf_attr __user *uattr, unsigned int size)` → `__sys_bpf(cmd, USER_BPFPTR(uattr), size)`.
- `bpf_check_uarg_tail_zero(uattr, sizeof(bpf_attr), size)` must succeed: bytes [size, sizeof(bpf_attr)) (or actual u-buffer extent) must be zero ⟹ -E2BIG / -EFAULT otherwise.
- `size = min(size, sizeof(union bpf_attr))`.
- `memset(&attr, 0, sizeof(attr))` then `copy_from_bpfptr(&attr, uattr, size)` ⟹ -EFAULT on fault.
- `security_bpf(cmd, &attr, size, uattr.is_kernel)` LSM gate ⟹ propagates `< 0`.

REQ-2: Per-cmd dispatch table (`__sys_bpf` switch):
- BPF_MAP_CREATE → `map_create(&attr, uattr)`.
- BPF_MAP_LOOKUP_ELEM → `map_lookup_elem(&attr)`.
- BPF_MAP_UPDATE_ELEM → `map_update_elem(&attr, uattr)`.
- BPF_MAP_DELETE_ELEM → `map_delete_elem(&attr, uattr)`.
- BPF_MAP_GET_NEXT_KEY → `map_get_next_key(&attr)`.
- BPF_MAP_FREEZE → `map_freeze(&attr)`.
- BPF_MAP_LOOKUP_AND_DELETE_ELEM → `map_lookup_and_delete_elem(&attr)`.
- BPF_MAP_LOOKUP_BATCH / BPF_MAP_LOOKUP_AND_DELETE_BATCH / BPF_MAP_UPDATE_BATCH / BPF_MAP_DELETE_BATCH → `bpf_map_do_batch(&attr, uattr.user, cmd)`.
- BPF_PROG_LOAD → `bpf_prog_load(&attr, uattr, size)`.
- BPF_OBJ_PIN → `bpf_obj_pin(&attr)`.
- BPF_OBJ_GET → `bpf_obj_get(&attr)`.
- BPF_PROG_ATTACH → `bpf_prog_attach(&attr)`.
- BPF_PROG_DETACH → `bpf_prog_detach(&attr)`.
- BPF_PROG_QUERY → `bpf_prog_query(&attr, uattr.user)`.
- BPF_PROG_TEST_RUN → `bpf_prog_test_run(&attr, uattr.user)`.
- BPF_PROG_GET_NEXT_ID / BPF_MAP_GET_NEXT_ID / BPF_BTF_GET_NEXT_ID / BPF_LINK_GET_NEXT_ID → `bpf_obj_get_next_id(&attr, uattr.user, &<x>_idr, &<x>_idr_lock)`.
- BPF_PROG_GET_FD_BY_ID → `bpf_prog_get_fd_by_id(&attr)`.
- BPF_MAP_GET_FD_BY_ID → `bpf_map_get_fd_by_id(&attr)`.
- BPF_BTF_GET_FD_BY_ID → `bpf_btf_get_fd_by_id(&attr)`.
- BPF_LINK_GET_FD_BY_ID → `bpf_link_get_fd_by_id(&attr)`.
- BPF_OBJ_GET_INFO_BY_FD → `bpf_obj_get_info_by_fd(&attr, uattr.user)`.
- BPF_RAW_TRACEPOINT_OPEN → `bpf_raw_tracepoint_open(&attr)`.
- BPF_BTF_LOAD → `bpf_btf_load(&attr, uattr, size)`.
- BPF_TASK_FD_QUERY → `bpf_task_fd_query(&attr, uattr.user)`.
- BPF_LINK_CREATE → `link_create(&attr, uattr)`.
- BPF_LINK_UPDATE → `link_update(&attr)`.
- BPF_LINK_DETACH → `link_detach(&attr)`.
- BPF_ENABLE_STATS → `bpf_enable_stats(&attr)`.
- BPF_ITER_CREATE → `bpf_iter_create(&attr)`.
- BPF_PROG_BIND_MAP → `bpf_prog_bind_map(&attr)`.
- BPF_TOKEN_CREATE → `token_create(&attr)`.
- Unknown cmd ⟹ -EINVAL.

REQ-3: map_create (`BPF_MAP_CREATE`):
- CHECK_ATTR(BPF_MAP_CREATE) ⟹ -EINVAL on stray non-zero fields past last permitted.
- `token_flag = attr.map_flags & BPF_F_TOKEN_FD`; clear before per-type checks.
- `f_flags = bpf_get_file_flag(attr.map_flags)`: validates O_RDONLY/O_WRONLY/O_RDWR; BPF_F_RDONLY/BPF_F_WRONLY → file mode.
- `map_type ∈ [0, ARRAY_SIZE(bpf_map_types))`; `array_index_nospec` (Spectre v1 guard); ops = `bpf_map_types[map_type]` ⟹ -EINVAL if NULL.
- if `ops.map_alloc_check`: invoke; propagate err.
- if `attr.map_ifindex`: `ops = &bpf_map_offload_ops` (offload to NIC).
- `!ops.map_mem_usage` ⟹ -EINVAL.
- if token_flag: `token = bpf_token_get_from_fd(attr.map_token_fd)`; if `!bpf_token_allow_cmd(token, BPF_MAP_CREATE) || !bpf_token_allow_map_type(token, attr.map_type)` ⟹ drop token, fall back to cred caps.
- if `sysctl_unprivileged_bpf_disabled ∧ !bpf_token_capable(token, CAP_BPF)` ⟹ -EPERM.
- Per-map-type capability:
  - unpriv set: ARRAY, PERCPU_ARRAY, PROG_ARRAY, PERF_EVENT_ARRAY, CGROUP_ARRAY, ARRAY_OF_MAPS, HASH, PERCPU_HASH, HASH_OF_MAPS, RINGBUF, USER_RINGBUF, CGROUP_STORAGE, PERCPU_CGROUP_STORAGE.
  - CAP_BPF: SK_STORAGE, INODE_STORAGE, TASK_STORAGE, CGRP_STORAGE, BLOOM_FILTER, LPM_TRIE, REUSEPORT_SOCKARRAY, STACK_TRACE, QUEUE, STACK, LRU_HASH, LRU_PERCPU_HASH, STRUCT_OPS, CPUMAP, ARENA, INSN_ARRAY.
  - CAP_NET_ADMIN: SOCKMAP, SOCKHASH, DEVMAP, DEVMAP_HASH, XSKMAP.
  - default ⟹ WARN + -EPERM via `put_token` path.
- `map = ops.map_alloc(attr)`; copy `attr.map_name` (≤BPF_OBJ_NAME_LEN-1 + NUL via `bpf_obj_name_cpy`).
- `map.cookie = gen_cookie_next(&bpf_map_cookie)`; `atomic64_set(&map.refcnt, 1)`; `usercnt = 1`; mutex_init freeze_mutex; spin_lock_init owner_lock.
- If `attr.btf_key_type_id || attr.btf_value_type_id || attr.btf_vmlinux_value_type_id`: `btf_get_by_fd(attr.btf_fd)`; reject kernel BTF (-EACCES); if value_type_id → `map_check_btf` (key/value/record sanity).
- If `attr.excl_prog_hash`: require `excl_prog_hash_size == SHA256_DIGEST_SIZE`; copy SHA into `map.excl_prog_sha`.
- `security_bpf_map_create(map, attr, token, uattr.is_kernel)` LSM gate.
- `bpf_map_alloc_id(map)` → IDR assignment.
- `bpf_map_save_memcg(map)`.
- `bpf_token_put(token)`.
- `bpf_map_new_fd(map, f_flags)` ⟹ if `< 0`: `bpf_map_put_with_uref(map)` (since IDR-published).
- Failure paths: `free_map_sec` ⟹ `security_bpf_map_free`; `free_map` ⟹ `bpf_map_free`; `put_token` ⟹ `bpf_token_put`.

REQ-4: bpf_prog_load (`BPF_PROG_LOAD`):
- CHECK_ATTR(BPF_PROG_LOAD).
- Validate `attr.prog_flags ⊆ {BPF_F_STRICT_ALIGNMENT, BPF_F_ANY_ALIGNMENT, BPF_F_TEST_STATE_FREQ, BPF_F_SLEEPABLE, BPF_F_TEST_RND_HI32, BPF_F_XDP_HAS_FRAGS, BPF_F_XDP_DEV_BOUND_ONLY, BPF_F_TEST_REG_INVARIANTS, BPF_F_TOKEN_FD}`.
- `bpf_prog_load_fixup_attach_type(attr)`.
- If `BPF_F_TOKEN_FD`: `token = bpf_token_get_from_fd(attr.prog_token_fd)`; if `!bpf_token_allow_cmd(token, BPF_PROG_LOAD) || !bpf_token_allow_prog_type(token, attr.prog_type, attr.expected_attach_type)` ⟹ drop token.
- `bpf_cap = bpf_token_capable(token, CAP_BPF)`.
- If `BPF_F_ANY_ALIGNMENT ∧ !HAVE_EFFICIENT_UNALIGNED_ACCESS ∧ !bpf_cap` ⟹ -EPERM.
- If `sysctl_unprivileged_bpf_disabled ∧ !bpf_cap` ⟹ -EPERM.
- `insn_cnt == 0` ⟹ -E2BIG; `insn_cnt > (bpf_cap ? BPF_COMPLEXITY_LIMIT_INSNS : BPF_MAXINSNS)` ⟹ -E2BIG.
- Unprivileged prog types: only BPF_PROG_TYPE_SOCKET_FILTER, BPF_PROG_TYPE_CGROUP_SKB without CAP_BPF.
- `is_net_admin_prog_type(type) ∧ !bpf_token_capable(token, CAP_NET_ADMIN)` ⟹ -EPERM.
- `is_perfmon_prog_type(type) ∧ !bpf_token_capable(token, CAP_PERFMON)` ⟹ -EPERM.
- Resolve `attach_prog_fd` ∨ `attach_btf_obj_fd` ∨ vmlinux BTF (if `attach_btf_id`).
- `bpf_prog_load_check_attach(type, expected_attach_type, attach_btf, attach_btf_id, dst_prog)`.
- `prog = bpf_prog_alloc(bpf_prog_size(insn_cnt), GFP_USER)`.
- Copy `expected_attach_type`; `prog.sleepable = !!(prog_flags & BPF_F_SLEEPABLE)`; move token into `prog.aux.token` (reuse refcnt).
- `prog.aux.user = get_current_user()`; `prog.len = insn_cnt`.
- `copy_from_bpfptr(prog.insns, ..., bpf_prog_insn_size(prog))` ⟹ -EFAULT.
- `strncpy_from_bpfptr(license, ..., sizeof(license)-1)`; `prog.gpl_compatible = license_is_gpl_compatible(license)`.
- If `attr.signature`: `bpf_prog_verify_signature(prog, attr, uattr.is_kernel)`.
- `atomic64_set(&prog.aux.refcnt, 1)`.
- If `bpf_prog_is_dev_bound(prog.aux)`: `bpf_prog_dev_bound_init(prog, attr)`.
- If BPF_PROG_TYPE_TRACING ∧ dst is TRACING: set `attach_tracing_prog`.
- `find_prog_type(type, prog)`.
- `prog.aux.load_time = ktime_get_boottime_ns()`.
- `bpf_obj_name_cpy(prog.aux.name, attr.prog_name, sizeof(attr.prog_name))`.
- `security_bpf_prog_load(prog, attr, token, uattr.is_kernel)`.
- `bpf_check(&prog, attr, uattr, uattr_size)` — full verifier pass (delegates to verifier.md).
- `bpf_prog_mark_insn_arrays_ready(prog)`.
- `bpf_prog_alloc_id(prog)` — IDR.
- `bpf_prog_kallsyms_add(prog)`.
- `perf_event_bpf_event(prog, PERF_BPF_EVENT_PROG_LOAD, 0)`.
- `bpf_audit_prog(prog, BPF_AUDIT_LOAD)`.
- `bpf_prog_new_fd(prog)`; on `< 0`: `bpf_prog_put(prog)`.
- Failure: `free_used_maps` ⟹ `__bpf_prog_put_noref(prog, prog.aux.real_func_cnt)`; `free_prog_sec` ⟹ `security_bpf_prog_free`; `free_prog` ⟹ `free_uid` + `btf_put` + `bpf_prog_free`; `put_token` ⟹ `bpf_token_put`.

REQ-5: bpf_obj_pin / bpf_obj_get (`BPF_OBJ_PIN` / `BPF_OBJ_GET`):
- CHECK_ATTR(BPF_OBJ); `file_flags ⊆ BPF_F_PATH_FD` (pin) or `BPF_OBJ_FLAG_MASK | BPF_F_PATH_FD` (get).
- `BPF_F_PATH_FD` required if `path_fd != 0`; otherwise `path_fd = AT_FDCWD`.
- pin: `bpf_obj_pin_user(attr.bpf_fd, path_fd, u64_to_user_ptr(attr.pathname))` → resolves prog/map/link by fd, mkdentry under bpffs (see inode.md).
- get: `bpf_obj_get_user(path_fd, u64_to_user_ptr(attr.pathname), attr.file_flags)` → reverse-lookup inode → install new fd of correct kind.

REQ-6: bpf_prog_attach / bpf_prog_detach (legacy non-link attach):
- CHECK_ATTR(BPF_PROG_ATTACH/DETACH).
- `ptype = attach_type_to_prog_type(attr.attach_type)` ⟹ -EINVAL if UNSPEC.
- Flag mask depends on prog type: mprog (`BPF_F_REPLACE|BEFORE|AFTER|ID|LINK`) vs cgroup-base (`BPF_F_ALLOW_OVERRIDE|BPF_F_ALLOW_MULTI|BPF_F_REPLACE|BPF_F_PREORDER`) vs simple.
- `bpf_prog_get_type(attr.attach_bpf_fd, ptype)`.
- `bpf_prog_attach_check_attach_type(prog, attr.attach_type)`: per-prog-type validation (CGROUP_SKB requires CAP_NET_ADMIN via token; KPROBE/PERF_EVENT/TRACEPOINT must match attach_type; etc.).
- Dispatch by ptype:
  - SK_SKB / SK_MSG → `sock_map_get_from_fd`.
  - LIRC_MODE2 → `lirc_prog_attach`.
  - FLOW_DISSECTOR → `netns_bpf_prog_attach`.
  - SCHED_CLS w/ TCX_INGRESS|EGRESS → `tcx_prog_attach`; else `netkit_prog_attach`.
  - cgroup family → `cgroup_bpf_prog_attach`.
- Detach mirrors with `bpf_mprog_detach_empty` check; on attach_bpf_fd=0 with mprog, requires non-empty.

REQ-7: link_create (`BPF_LINK_CREATE`):
- CHECK_ATTR(BPF_LINK_CREATE).
- Special-case `attr.link_create.attach_type == BPF_STRUCT_OPS` → `bpf_struct_ops_link_create(attr)`.
- `prog = bpf_prog_get(attr.link_create.prog_fd)`.
- `bpf_prog_attach_check_attach_type(prog, attr.link_create.attach_type)`.
- Dispatch by `prog.type`:
  - cgroup family / SOCK_OPS / CGROUP_SOCKOPT → `cgroup_bpf_link_attach`.
  - EXT → `bpf_tracing_prog_attach(prog, target_fd, target_btf_id, tracing.cookie, attach_type)`.
  - LSM / TRACING: expected_attach_type must equal attach_type; if BPF_TRACE_RAW_TP → `bpf_raw_tp_link_attach`; if BPF_TRACE_ITER → `bpf_iter_link_attach(attr, uattr, prog)`; if BPF_LSM_CGROUP → `cgroup_bpf_link_attach`; else `bpf_tracing_prog_attach`.
  - FLOW_DISSECTOR / SK_LOOKUP → `netns_bpf_link_create`.
  - SK_MSG / SK_SKB → `sock_map_link_create`.
  - XDP → `bpf_xdp_link_attach` (CONFIG_NET).
  - SCHED_CLS → `tcx_link_attach` or `netkit_link_attach`.
  - NETFILTER → `bpf_nf_link_attach`.
  - PERF_EVENT / TRACEPOINT → `bpf_perf_link_attach`.
  - KPROBE → `bpf_perf_link_attach` / `bpf_kprobe_multi_link_attach` / `bpf_uprobe_multi_link_attach` based on attach_type.
- On error: `bpf_prog_put(prog)`.

REQ-8: link_update (`BPF_LINK_UPDATE`):
- CHECK_ATTR(BPF_LINK_UPDATE).
- `flags ⊆ BPF_F_REPLACE`.
- `link = bpf_link_get_from_fd(attr.link_update.link_fd)`.
- If `link.ops.update_map`: route to `link_update_map(link, attr)` (map-based link; BPF_F_REPLACE pairs old_map_fd/new_map_fd).
- Else: `new_prog = bpf_prog_get(attr.link_update.new_prog_fd)`; if BPF_F_REPLACE → old_prog from old_prog_fd; `link.ops.update_prog(link, new_prog, old_prog)` ⟹ -EINVAL if ops missing.

REQ-9: link_detach (`BPF_LINK_DETACH`):
- CHECK_ATTR(BPF_LINK_DETACH).
- `link = bpf_link_get_from_fd(attr.link_detach.link_fd)`.
- `link.ops.detach(link)` ⟹ -EOPNOTSUPP if absent.
- Always `bpf_link_put_direct(link)`.

REQ-10: link primer 2-phase install (`bpf_link_prime` / `bpf_link_settle` / `bpf_link_cleanup`):
- `bpf_link_prime`: reserve unused fd (O_CLOEXEC); allocate id via `bpf_link_alloc_id` (idr_alloc_cyclic, 1..INT_MAX); `anon_inode_getfile("bpf_link", &bpf_link_fops[_poll], link, O_CLOEXEC)`; on failure ⟹ free id, put unused fd. Populates `primer.{link,file,fd,id}`.
- `bpf_link_settle`: under link_idr_lock set `link.id = primer.id`; `fd_install(primer.fd, primer.file)`; returns fd.
- `bpf_link_cleanup`: zero `primer.link.prog`; `bpf_link_free_id(primer.id)`; `fput(primer.file)`; `put_unused_fd(primer.fd)`; caller responsible for prog refcount.

REQ-11: bpf_prog_test_run / bpf_prog_query:
- TEST_RUN: CHECK_ATTR(BPF_PROG_TEST_RUN); `(ctx_size_in ↔ ctx_in)` xor invalid ⟹ -EINVAL; dispatch via `prog.aux.ops.test_run`.
- QUERY: requires `bpf_net_capable()`; `query_flags ⊆ BPF_F_QUERY_EFFECTIVE`; dispatch by attach_type to cgroup_bpf / lirc / netns / sock_map / tcx / netkit prog_query.

REQ-12: bpf_obj_get_next_id / *_get_fd_by_id / *_get_info_by_fd:
- GET_NEXT_ID: walk `<x>_idr` starting at `attr.start_id` returning next allocated id; requires `capable(CAP_SYS_ADMIN)` (enforced per-helper).
- GET_FD_BY_ID: idr_find under idr_lock; bump refcnt with `_inc_not_zero`; install new fd.
- GET_INFO_BY_FD: dispatch by fd type (prog/map/btf/link) to its `_get_info_by_fd` which writes uattr.info struct.

REQ-13: bpf_btf_load (`BPF_BTF_LOAD`):
- CHECK_ATTR(BPF_BTF_LOAD).
- `btf_flags ⊆ BPF_F_TOKEN_FD`.
- If token-fd: `bpf_token_get_from_fd`; if `!bpf_token_allow_cmd(token, BPF_BTF_LOAD)` ⟹ drop.
- `bpf_token_capable(token, CAP_BPF)` else -EPERM.
- `bpf_token_put(token)`.
- `btf_new_fd(attr, uattr, uattr_size)` — parses BTF blob, installs fd.

REQ-14: bpf_btf_get_fd_by_id:
- CHECK_ATTR(BPF_BTF_GET_FD_BY_ID).
- `open_flags ⊆ BPF_F_TOKEN_FD`.
- Token-fd → allow_cmd(BPF_BTF_GET_FD_BY_ID).
- `bpf_token_capable(token, CAP_SYS_ADMIN)` else -EPERM.
- `btf_get_fd_by_id(attr.btf_id)`.

REQ-15: bpf_task_fd_query (`BPF_TASK_FD_QUERY`):
- CHECK_ATTR(BPF_TASK_FD_QUERY).
- `capable(CAP_SYS_ADMIN)` else -EPERM.
- `attr.task_fd_query.flags == 0` else -EINVAL.
- Resolve task via `get_pid_task(find_vpid(pid), PIDTYPE_PID)`.
- `fget_task(task, fd)`; put_task.
- If `file.f_op ∈ {bpf_link_fops, bpf_link_fops_poll} ∧ link.ops == bpf_raw_tp_link_lops`: report `prog_id`, `BPF_FD_TYPE_RAW_TRACEPOINT`, tracepoint name.
- Else `perf_get_event(file)` → `bpf_get_perf_event_info` → copy back via `bpf_task_fd_query_copy`.
- Default ⟹ -ENOTSUPP.

REQ-16: bpf_iter_create (`BPF_ITER_CREATE`):
- CHECK_ATTR(BPF_ITER_CREATE); `iter_create.flags == 0`.
- `link = bpf_link_get_from_fd(attr.iter_create.link_fd)`.
- `bpf_iter_new_fd(link)` (anon-inode); always `bpf_link_put_direct(link)`.

REQ-17: bpf_prog_bind_map (`BPF_PROG_BIND_MAP`):
- CHECK_ATTR(BPF_PROG_BIND_MAP); `prog_bind_map.flags == 0`.
- `prog = bpf_prog_get(prog_fd)`; `map = bpf_map_get(map_fd)`.
- mutex_lock(prog.aux.used_maps_mutex).
- Scan `prog.aux.used_maps[0..used_map_cnt)`; if map already present ⟹ drop ref + return 0.
- Realloc `used_maps_new = kmalloc(used_map_cnt + 1)`.
- If `prog.sleepable`: `atomic64_inc(&map.sleepable_refcnt)`.
- memcpy old → new; append map at index used_map_cnt; bump used_map_cnt; swap pointer; kfree(old).

REQ-18: token_create (`BPF_TOKEN_CREATE`):
- CHECK_ATTR(BPF_TOKEN_CREATE); `token_create.flags == 0`.
- Delegates to `bpf_token_create(attr)`: opens bpffs dir referenced by `attr.token_create.bpffs_fd`; checks delegate masks on that mount; pins a new `struct bpf_token` (cmds, maps, progs, attachs allowed bitmasks); returns fd.

REQ-19: BPF_CALL_3(bpf_sys_bpf) / kern_sys_bpf:
- In-prog helper `bpf_sys_bpf(cmd, attr, attr_size)`: whitelist {BPF_MAP_CREATE, BPF_MAP_DELETE_ELEM, BPF_MAP_UPDATE_ELEM, BPF_MAP_FREEZE, BPF_MAP_GET_FD_BY_ID, BPF_PROG_LOAD, BPF_BTF_LOAD, BPF_LINK_CREATE, BPF_RAW_TRACEPOINT_OPEN}; else -EINVAL; dispatch to `__sys_bpf(cmd, KERNEL_BPFPTR(attr), attr_size)`. Helper guarded by `bpf_token_capable(prog.aux.token, CAP_PERFMON)` in proto getter.
- `kern_sys_bpf` (light-skeleton/module-load path): handles BPF_PROG_TEST_RUN for BPF_PROG_TYPE_SYSCALL inline via `__bpf_prog_enter_sleepable_recur`/`bpf_prog_run`; otherwise `____bpf_sys_bpf`. Exported `EXPORT_SYMBOL_NS(kern_sys_bpf, "BPF_INTERNAL")`.

REQ-20: struct bpf_prog lifecycle:
- alloc: `bpf_prog_alloc(bpf_prog_size(insn_cnt), GFP_USER)` → kvmalloc; sets `aux.refcnt = 1`, `aux.user`, `aux.token`.
- expose: `bpf_prog_alloc_id` (IDR) → `bpf_prog_kallsyms_add` (symbol table) → `bpf_audit_prog(BPF_AUDIT_LOAD)` → `bpf_prog_new_fd`.
- put: `__bpf_prog_put` decrements refcnt; on 0 → `bpf_audit_prog(BPF_AUDIT_UNLOAD)` → `__bpf_prog_put_noref(prog, true)` → defer free via `call_rcu` (or `call_rcu_tasks_trace` for sleepable).
- release file_op: `bpf_prog_release` → `__bpf_prog_put`.

REQ-21: struct bpf_link lifecycle:
- init: `bpf_link_init_sleepable(link, type, ops, prog, attach_type, sleepable)`; refcnt=1; sleepable bit drives free path.
- free: `bpf_link_free` calls `ops.release`; if `ops.dealloc_deferred` → schedule via `call_rcu_tasks_trace` (sleepable / sleepable prog) or `call_tracepoint_unregister_atomic` (non-faultable tracepoint link) or `call_rcu`; else direct `ops.dealloc`.
- `bpf_link_put` (atomic ctx): decref then `schedule_work(bpf_link_put_deferred)`.
- `bpf_link_put_direct` (process ctx): decref + `bpf_link_free` inline.

REQ-22: Capability gates summary:
- `CAP_BPF`: prog load (non-cgroup-skb/socket-filter), privileged map types, BTF load, signed-prog signature override.
- `CAP_NET_ADMIN`: socket-map types (SOCKMAP/SOCKHASH/DEVMAP/DEVMAP_HASH/XSKMAP), `is_net_admin_prog_type` progs, CGROUP_SKB attach.
- `CAP_PERFMON`: `is_perfmon_prog_type` (tracing/perf-event family); `bpf_sys_bpf` helper.
- `CAP_SYS_ADMIN`: BPF_TASK_FD_QUERY, BTF_GET_FD_BY_ID, bpf_stats sysctl, mounting bpffs in non-init userns, delegate_* mount opts.
- Per-`BPF_F_TOKEN_FD`: capability check `bpf_token_capable(token, cap)` instead of `capable(cap)`.

## Acceptance Criteria

- [ ] AC-1: `bpf(2)` rejects `uattr` with non-zero tail bytes (`bpf_check_uarg_tail_zero`).
- [ ] AC-2: Unknown `cmd` returns -EINVAL.
- [ ] AC-3: `BPF_MAP_CREATE` of unprivileged map type succeeds without CAP_BPF when `sysctl_unprivileged_bpf_disabled = 0`.
- [ ] AC-4: `BPF_MAP_CREATE` of SOCKMAP without CAP_NET_ADMIN returns -EPERM (no token).
- [ ] AC-5: `BPF_PROG_LOAD` with `insn_cnt > BPF_MAXINSNS` and !CAP_BPF returns -E2BIG.
- [ ] AC-6: `BPF_PROG_LOAD` with bad license sets `prog.gpl_compatible = 0` (GPL-only helpers reject).
- [ ] AC-7: `BPF_PROG_LOAD` failure after `bpf_prog_alloc_id` uses `__bpf_prog_put_noref` (no double-free).
- [ ] AC-8: `BPF_OBJ_PIN` of prog fd then `BPF_OBJ_GET` returns a valid prog fd referencing same id.
- [ ] AC-9: `BPF_LINK_CREATE` for incompatible (prog.type, attach_type) returns -EINVAL.
- [ ] AC-10: `BPF_LINK_UPDATE` with `BPF_F_REPLACE` and matching old_prog_fd succeeds atomically.
- [ ] AC-11: `BPF_LINK_DETACH` on link without `ops.detach` returns -EOPNOTSUPP.
- [ ] AC-12: `BPF_BTF_LOAD` without CAP_BPF returns -EPERM.
- [ ] AC-13: `BPF_TASK_FD_QUERY` without CAP_SYS_ADMIN returns -EPERM.
- [ ] AC-14: `BPF_PROG_BIND_MAP` is idempotent on duplicate map binding.
- [ ] AC-15: `BPF_TOKEN_CREATE` against a bpffs without `delegate_cmds` set returns -EPERM.
- [ ] AC-16: `bpf_sys_bpf` in-prog helper rejects commands outside whitelist with -EINVAL.

## Architecture

```
enum BpfCmd { MapCreate=0, MapLookupElem, MapUpdateElem, MapDeleteElem, MapGetNextKey,
              ProgLoad, ObjPin, ObjGet, ProgAttach, ProgDetach, ProgTestRun,
              ProgGetNextId, MapGetNextId, ProgGetFdById, MapGetFdById,
              ObjGetInfoByFd, ProgQuery, RawTracepointOpen, BtfLoad,
              BtfGetFdById, TaskFdQuery, MapLookupAndDeleteElem,
              MapFreeze, BtfGetNextId, MapLookupBatch, MapLookupAndDeleteBatch,
              MapUpdateBatch, MapDeleteBatch, LinkCreate, LinkUpdate,
              LinkGetFdById, LinkGetNextId, EnableStats, IterCreate,
              LinkDetach, ProgBindMap, TokenCreate, ProgStreamReadByFd,
              ProgAssocStructOps }

union BpfAttr {
  // BPF_MAP_CREATE
  map: BpfMapCreateAttr { map_type: u32, key_size: u32, value_size: u32,
                          max_entries: u32, map_flags: u32, inner_map_fd: u32,
                          numa_node: u32, map_name: [u8; BPF_OBJ_NAME_LEN],
                          map_ifindex: u32, btf_fd: u32, btf_key_type_id: u32,
                          btf_value_type_id: u32, btf_vmlinux_value_type_id: u32,
                          map_extra: u64, value_type_btf_obj_fd: i32,
                          map_token_fd: i32, excl_prog_hash: u64,
                          excl_prog_hash_size: u32 },
  // BPF_PROG_LOAD
  prog: BpfProgLoadAttr { prog_type: u32, insn_cnt: u32, insns: u64 /*__user*/,
                          license: u64 /*__user*/, log_level: u32, log_size: u32,
                          log_buf: u64 /*__user*/, kern_version: u32,
                          prog_flags: u32, prog_name: [u8; BPF_OBJ_NAME_LEN],
                          prog_ifindex: u32, expected_attach_type: u32,
                          prog_btf_fd: u32, func_info_rec_size: u32,
                          func_info: u64 /*__user*/, func_info_cnt: u32,
                          line_info_rec_size: u32, line_info: u64 /*__user*/,
                          line_info_cnt: u32, attach_btf_id: u32,
                          attach_prog_fd: u32, attach_btf_obj_fd: u32,
                          fd_array: u64, core_relos: u64, core_relo_cnt: u32,
                          core_relo_rec_size: u32, log_true_size: u32,
                          prog_token_fd: i32, signature: u64,
                          signature_size: u32, keyring_id: i32 },
  // BPF_OBJ_PIN / BPF_OBJ_GET
  obj: BpfObjAttr { pathname: u64 /*__user*/, bpf_fd: u32, file_flags: u32, path_fd: i32 },
  // BPF_LINK_CREATE / _UPDATE / _DETACH (subset)
  link_create: { prog_fd, target_fd, attach_type, flags, target_btf_id, tracing.cookie, ... },
  link_update: { link_fd, new_prog_fd, flags, old_prog_fd, new_map_fd, old_map_fd },
  link_detach: { link_fd },
  // BPF_TOKEN_CREATE
  token_create: { flags: u32, bpffs_fd: u32 },
  // ... ~22 more variants
}

struct BpfProg {
  pages: u16,
  jited: bool,
  jit_requested: bool,
  gpl_compatible: bool,
  cb_access: bool,
  dst_needed: bool,
  blinding_requested: bool,
  blinded: bool,
  is_func: bool,
  kprobe_override: bool,
  has_callchain_buf: bool,
  enforce_expected_attach_type: bool,
  call_get_stack: bool,
  call_get_func_ip: bool,
  tstamp_type_access: bool,
  sleepable: bool,
  len: u32,                                    // insn count
  jited_len: u32,
  tag: [u8; BPF_TAG_SIZE],
  stats: *BpfProgStats,
  active: *i32,
  bpf_func: fn(*const ctx, *const Insn) -> u32,
  insnsi: [Insn],                              // verifier instructions (or jited image)
  aux: Box<BpfProgAux>,
  orig_prog: *BpfProgSock,
}

struct BpfProgAux {
  refcnt: AtomicU64,
  used_map_cnt: u32,
  used_btf_cnt: u32,
  max_ctx_offset: u32,
  max_pkt_offset: u32,
  max_tp_access: u32,
  stack_depth: u32,
  id: u32,                                     // assigned by bpf_prog_alloc_id
  func_cnt: u32,
  real_func_cnt: u32,
  func_idx: u32,
  attach_btf_id: u32,
  ctx_arg_info_size: u32,
  max_rdonly_access: u32,
  max_rdwr_access: u32,
  attach_btf: *Btf,
  ctx_arg_info: *const BpfCtxArgAuxInfo,
  dst_prog: *BpfProg,
  dst_trampoline: *BpfTrampoline,
  poke_tab: *BpfJitPokeDescriptor,
  arena: *BpfMap,
  size_poke_tab: u32,
  dyn_pokes: *RhashHead,
  func: *mut *mut BpfProg,
  jit_data: *u8,
  user: *User,
  load_time: u64,
  verified_insns: u32,
  cgroup_atype: AttachType,
  used_maps: *mut *mut BpfMap,
  used_maps_mutex: Mutex,
  used_btfs: *mut *mut Btf,
  cgroup_storage: [*mut BpfMap; MAX_BPF_CGROUP_STORAGE_TYPE],
  name: [u8; BPF_OBJ_NAME_LEN],
  token: *BpfToken,
  rcu: RcuHead,
  rcu_in_progress: WorkStruct,
  sleepable_refcnt: AtomicU64,
  saved_dst_prog_type: BpfProgType,
  saved_dst_attach_type: BpfAttachType,
  attach_tracing_prog: bool,
  dev_bound: bool,
  ksym: BpfKsym,
}

struct BpfLink {
  refcnt: AtomicU64,
  id: u32,
  type: BpfLinkType,
  sleepable: bool,
  flags: u32,
  attach_type: BpfAttachType,
  ops: &'static BpfLinkOps,
  prog: *BpfProg,
  rcu: RcuHead,
  work: WorkStruct,
}

struct BpfLinkPrimer { link: *BpfLink, file: *File, fd: i32, id: u32 }

struct BpfLinkOps {
  release: fn(&mut BpfLink),
  dealloc: Option<fn(&mut BpfLink)>,
  dealloc_deferred: Option<fn(&mut BpfLink)>,
  detach: Option<fn(&mut BpfLink) -> i32>,
  update_prog: Option<fn(&mut BpfLink, new: &BpfProg, old: Option<&BpfProg>) -> i32>,
  update_map: Option<fn(&mut BpfLink, new: &BpfMap, old: Option<&BpfMap>) -> i32>,
  show_fdinfo: Option<fn(&BpfLink, &mut SeqFile)>,
  fill_link_info: Option<fn(&BpfLink, &mut BpfLinkInfo) -> i32>,
  poll: Option<fn(&File, &mut PollTableStruct) -> PollFlags>,
}

struct BpfToken {
  work: WorkStruct,
  // bitmasks of delegated commands / map types / prog types / attach types
  allowed_cmds: u64,
  allowed_maps: u64,
  allowed_progs: u64,
  allowed_attachs: u64,
  userns: *UserNamespace,
}
```

`Bpf::sys_bpf(cmd, uattr, size) -> Result<i32, Errno>`:
1. `bpf_check_uarg_tail_zero(uattr, sizeof(BpfAttr), size)` ⟹ -E2BIG/-EFAULT.
2. `size = min(size, sizeof(BpfAttr))`.
3. `let mut attr = BpfAttr::zeroed(); copy_from_bpfptr(&mut attr, uattr, size)` ⟹ -EFAULT.
4. `security_bpf(cmd, &attr, size, uattr.is_kernel)` ⟹ propagate `< 0`.
5. Dispatch table per REQ-2.
6. Return per-handler result.

`Bpf::map_create(&mut attr, uattr) -> Result<i32, Errno>`:
1. CHECK_ATTR(BPF_MAP_CREATE) per per-cmd boundary BPF_MAP_CREATE_LAST_FIELD.
2. token_flag = attr.map_flags & BPF_F_TOKEN_FD; attr.map_flags &= !BPF_F_TOKEN_FD.
3. /* BTF id sanity */
4. if attr.btf_vmlinux_value_type_id ∧ map_type != STRUCT_OPS ⟹ -EINVAL.
5. if attr.btf_key_type_id ∧ !attr.btf_value_type_id ⟹ -EINVAL.
6. /* file flags */
7. f_flags = bpf_get_file_flag(attr.map_flags).
8. /* numa node */
9. validate numa_node.
10. /* ops table */
11. ops = bpf_map_types[array_index_nospec(map_type, ARRAY_SIZE)].
12. if let Some(ck) = ops.map_alloc_check: ck(attr).
13. if attr.map_ifindex: ops = &bpf_map_offload_ops.
14. require ops.map_mem_usage.
15. /* token */
16. if token_flag: token = bpf_token_get_from_fd(attr.map_token_fd); validate allow_cmd/allow_map_type; drop if mismatched.
17. /* unprivileged gate */
18. if sysctl_unprivileged_bpf_disabled ∧ !bpf_token_capable(token, CAP_BPF) ⟹ -EPERM.
19. /* per-type capability switch */
20. /* alloc */
21. map = ops.map_alloc(attr).
22. /* identity */
23. bpf_obj_name_cpy(&mut map.name, &attr.map_name).
24. map.cookie = gen_cookie_next(&BPF_MAP_COOKIE).
25. /* refcounts */
26. map.refcnt = 1; map.usercnt = 1; mutex_init(&map.freeze_mutex); spin_lock_init(&map.owner_lock).
27. /* BTF binding */
28. if attr has BTF: load btf, validate not kernel-only, map_check_btf.
29. /* exclusive-prog hash */
30. if attr.excl_prog_hash: copy SHA256.
31. /* LSM hook */
32. security_bpf_map_create(map, attr, token, uattr.is_kernel).
33. /* expose */
34. bpf_map_alloc_id(map); bpf_map_save_memcg(map); bpf_token_put(token).
35. /* fd */
36. fd = bpf_map_new_fd(map, f_flags).
37. on fd < 0: bpf_map_put_with_uref(map).
38. return fd.

`Bpf::prog_load(&mut attr, uattr, uattr_size) -> Result<i32, Errno>`:
1. CHECK_ATTR(BPF_PROG_LOAD).
2. Validate prog_flags mask.
3. bpf_prog_load_fixup_attach_type(attr).
4. /* Token */
5. if BPF_F_TOKEN_FD: bpf_token_get_from_fd(attr.prog_token_fd); validate allow_cmd / allow_prog_type; drop if mismatched.
6. bpf_cap = bpf_token_capable(token, CAP_BPF).
7. /* Privilege gates */
8. various permission checks per REQ-4.
9. /* attach target resolution */
10. /* alloc + copy insns */
11. prog = bpf_prog_alloc(bpf_prog_size(insn_cnt), GFP_USER).
12. set sleepable / aux fields / move token.
13. copy_from_bpfptr(prog.insns, ...).
14. /* license */
15. strncpy_from_bpfptr(license).
16. prog.gpl_compatible = license_is_gpl_compatible(license).
17. /* signature */
18. if attr.signature: bpf_prog_verify_signature(prog, attr, ...).
19. atomic64_set(&prog.aux.refcnt, 1).
20. /* find prog_type ops */
21. find_prog_type(type, prog).
22. /* name + load_time */
23. /* LSM */
24. security_bpf_prog_load(prog, attr, token, uattr.is_kernel).
25. /* VERIFIER */
26. bpf_check(&mut prog, attr, uattr, uattr_size).
27. bpf_prog_mark_insn_arrays_ready(prog).
28. /* expose */
29. bpf_prog_alloc_id(prog).
30. bpf_prog_kallsyms_add(prog).
31. perf_event_bpf_event(prog, PERF_BPF_EVENT_PROG_LOAD, 0).
32. bpf_audit_prog(prog, BPF_AUDIT_LOAD).
33. /* fd */
34. fd = bpf_prog_new_fd(prog).
35. on fd < 0: bpf_prog_put(prog).
36. return fd.

`Bpf::link_prime(&mut link, &mut primer) -> Result<(), Errno>`:
1. fd = get_unused_fd_flags(O_CLOEXEC).
2. id = bpf_link_alloc_id(link).
3. file = anon_inode_getfile("bpf_link", link.ops.poll ? &bpf_link_fops_poll : &bpf_link_fops, link, O_CLOEXEC).
4. on file err: bpf_link_free_id(id); put_unused_fd(fd).
5. primer = { link, file, fd, id }.

`Bpf::link_settle(&mut primer) -> i32`:
1. lock link_idr_lock; primer.link.id = primer.id; unlock.
2. fd_install(primer.fd, primer.file).
3. return primer.fd.

`Bpf::token_create(&attr) -> Result<i32, Errno>`:
1. CHECK_ATTR(BPF_TOKEN_CREATE); flags == 0.
2. bpf_token_create(attr): opens bpffs at attr.bpffs_fd, intersects delegate_{cmds,maps,progs,attachs} with mount opts, anon-inode-installs a new BpfToken fd.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `uattr_tail_zero_or_efault` | INVARIANT | per-bpf(2): non-zero tail in uattr ⟹ -E2BIG / -EFAULT, never silent truncation. |
| `unknown_cmd_einval` | INVARIANT | per-bpf(2): cmd ∉ enum ⟹ -EINVAL. |
| `lsm_security_bpf_propagates` | INVARIANT | per-bpf(2): security_bpf < 0 ⟹ syscall returns that errno. |
| `map_type_array_index_nospec` | INVARIANT | per-map_create: map_type indexed with array_index_nospec (Spectre v1). |
| `prog_alloc_id_paired_with_put_noref_on_fail` | INVARIANT | per-bpf_prog_load: after bpf_prog_alloc_id, failure uses __bpf_prog_put_noref (never bpf_prog_free). |
| `prog_refcnt_balanced` | INVARIANT | per-load: success ⟹ refcnt=1, fd holds 1 ref; failure ⟹ refcnt drops to 0. |
| `link_primer_cleanup_on_failure` | INVARIANT | per-link_prime: failure ⟹ no leaked fd / id / file. |
| `link_settle_id_under_idr_lock` | INVARIANT | per-link_settle: link.id assignment under link_idr_lock. |
| `token_fd_dropped_when_not_allowed` | INVARIANT | per-cmd: token disallowing cmd or sub-type ⟹ bpf_token_put + fallback to cred caps. |
| `unprivileged_disabled_no_cap_bpf_eperm` | INVARIANT | per-map_create / prog_load: sysctl_unprivileged_bpf_disabled ∧ !CAP_BPF ⟹ -EPERM. |
| `is_net_admin_prog_type_gated` | INVARIANT | per-prog_load: net-admin prog type ⟹ require CAP_NET_ADMIN via token. |
| `is_perfmon_prog_type_gated` | INVARIANT | per-prog_load: perfmon prog type ⟹ require CAP_PERFMON via token. |
| `prog_used_maps_no_duplicate` | INVARIANT | per-prog_bind_map: duplicate map binding returns 0 with no realloc. |

### Layer 2: TLA+

`kernel/bpf/syscall.tla`:
- Per-bpf-syscall entry → per-dispatch → per-handler → per-result.
- Properties:
  - `safety_uattr_size_validated` — per-syscall: tail-zero check before dispatch.
  - `safety_per_cmd_capability_gate_enforced` — per-cmd: required cap or token allowance checked before object mutation.
  - `safety_prog_load_failure_cleans_up` — per-prog_load: failure path matches setup phase (alloc, id, kallsyms, audit, fd).
  - `safety_link_two_phase_install_atomic` — per-link: prime+settle either both observable or neither.
  - `safety_token_refcount_balanced` — per-cmd: every bpf_token_get_from_fd paired with bpf_token_put.
  - `liveness_each_cmd_terminates` — per-cmd: no path loops indefinitely (bounded by IDR walk size for GET_NEXT_ID).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bpf::sys_bpf` post: ret ∈ Errno ∪ Fd | `Bpf::sys_bpf` |
| `Bpf::map_create` post: success ⟹ refcnt=1 ∧ usercnt=1 ∧ IDR slot allocated | `Bpf::map_create` |
| `Bpf::prog_load` post: success ⟹ kallsyms + audit + IDR + fd ∧ refcnt=1 | `Bpf::prog_load` |
| `Bpf::prog_load` failure post: no IDR leak (id freed in put_noref) | `Bpf::prog_load` |
| `Bpf::link_prime` post: success ⟹ primer.{link,file,fd,id} all set; failure ⟹ none observable | `Bpf::link_prime` |
| `Bpf::link_settle` post: returns primer.fd; link.id set | `Bpf::link_settle` |
| `Bpf::link_cleanup` post: fput(file) + put_unused_fd(fd) + free_id; prog ref untouched | `Bpf::link_cleanup` |
| `Bpf::prog_bind_map` post: success ⟹ map appears exactly once in used_maps[0..used_map_cnt) | `Bpf::prog_bind_map` |
| `Bpf::token_create` post: success ⟹ token fd holds 1 ref; allowed bitmasks ⊆ mount delegate masks | `Bpf::token_create` |
| `Bpf::sys_bpf_helper` (in-prog) post: cmd ∉ whitelist ⟹ -EINVAL | in-prog `bpf_sys_bpf` |

### Layer 4: Verus/Creusot functional

`Per-bpf(2) → tail-zero check → security_bpf → dispatch table (~36 cmds) → per-cmd handler (CHECK_ATTR boundary, capability/token gate, object alloc + IDR + LSM + fd) → return fd|errno` semantic equivalence: per-`Documentation/bpf/syscall_api.rst`, per-`include/uapi/linux/bpf.h` UAPI surface, per-libbpf test corpus (`tools/testing/selftests/bpf/`).

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

bpf(2) reinforcement:

- **Per-uattr tail-zero enforced** — defense against per-future-field-leak (forward compat without smuggling state).
- **Per-`array_index_nospec(map_type, ARRAY_SIZE(bpf_map_types))`** — defense against per-Spectre-v1 speculative type-table OOB.
- **Per-`security_bpf` LSM hook before every cmd** — defense against per-policy bypass (SELinux, AppArmor, BPF LSM itself).
- **Per-`sysctl_unprivileged_bpf_disabled` honored before all object creation** — defense against per-unprivileged-eBPF exploitation surface.
- **Per-CAP_BPF / CAP_NET_ADMIN / CAP_PERFMON / CAP_SYS_ADMIN matrix** — defense against per-privilege-escalation via unintended cmd.
- **Per-BPF_F_TOKEN_FD: token allow_cmd / allow_prog_type / allow_map_type checked before token replaces cred** — defense against per-token-misuse / per-cross-cmd elevation.
- **Per-prog_load post-`bpf_prog_alloc_id` failure uses `__bpf_prog_put_noref`** — defense against per-IDR-leak and per-double-free.
- **Per-`bpf_audit_prog(LOAD/UNLOAD)`** — defense against per-tampering / per-supply-chain (kernel auditd trail).
- **Per-`bpf_prog_verify_signature` when attr.signature set** — defense against per-unauthenticated-prog from less-trusted origin (signed BPF skeletons).
- **Per-link 2-phase prime/settle/cleanup** — defense against per-half-installed link visible by-id before fd installed.
- **Per-`bpf_link_release` schedules dealloc through RCU (tasks-trace for sleepable / SRCU for non-faultable tracepoint)** — defense against per-use-after-free in attach hook readers.
- **Per-`license_is_gpl_compatible` gate on GPL-only helpers** — defense against per-license-laundering of proprietary BPF using GPL kernel helpers.
- **Per-`is_cgroup_prog_type` + per-`bpf_mprog_supported` flag mask split** — defense against per-flag-confusion between mprog and legacy attach.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- BPF verifier (`bpf_check`) — covered in `verifier.md` Tier-3.
- Map type implementations (hash, array, ringbuf, devmap, cpumap, ...) — covered in respective Tier-3 docs.
- BTF parsing / validation — covered in `btf.md` Tier-3.
- bpffs filesystem semantics (BPF_OBJ_PIN / _OBJ_GET path resolution) — covered in `inode.md` Tier-3.
- Cgroup-bpf attach mechanism — covered in `cgroup.md` Tier-3.
- BPF JIT code generation — covered in `bpf-core.md` Tier-3 / arch-specific JIT files.
- Per-link-type implementations (XDP link, kprobe link, perf link, tcx, netkit, raw_tp, struct_ops link) — covered in their owning subsystems.
- Implementation code.
