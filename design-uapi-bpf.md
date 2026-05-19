---
title: "Tier-5: include/uapi/linux/bpf.h — BPF / eBPF UAPI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The eBPF UAPI is the user-visible boundary of the kernel's in-kernel virtual machine and verifier. It exposes (a) the `bpf(2)` syscall command/argument enum (`enum bpf_cmd`) and the massive tagged-union argument `union bpf_attr`; (b) the eBPF instruction encoding (`struct bpf_insn`, instruction classes, ALU/JMP/LD/LDX/ST/STX/ATOMIC opcodes, `BPF_REG_0..R10`, the `BPF_CALL`/`BPF_EXIT` opcodes, `BPF_LOAD_ACQ`/`BPF_STORE_REL` modifiers, conditional pseudo-jumps `BPF_MAY_GOTO`); (c) the program/map/attach/link taxonomies (`enum bpf_prog_type`, `enum bpf_map_type`, `enum bpf_attach_type`, `enum bpf_link_type`); (d) per-map element semantics (LPM trie keys, ringbuf headers, cgroup-storage keys); (e) the BPF helper-function ID enum (`enum bpf_func_id`, generated from `___BPF_FUNC_MAPPER`); (f) the introspection structs (`bpf_prog_info`, `bpf_map_info`, `bpf_btf_info`, `bpf_link_info`, `bpf_token_info`, `bpf_stats_type`, `bpf_perf_event_value`); (g) BTF metadata (`btf_*` types from the companion `linux/btf.h`); (h) context structs visible to BPF programs (`__sk_buff`, `xdp_md`, `bpf_sock_*`, `bpf_sock_ops`, `bpf_sockopt`, `bpf_sysctl`, `bpf_sk_lookup`, `bpf_cgroup_dev_ctx`, `bpf_sock_addr`, `sk_msg_md`, `sk_reuseport_md`, `bpf_pidns_info`); (i) opaque kernel objects exposed by reference (`bpf_spin_lock`, `bpf_timer`, `bpf_task_work`, `bpf_wq`, `bpf_dynptr`, `bpf_list_head`/`_node`, `bpf_rb_root`/`_node`, `bpf_refcount`, `bpf_iter_num`); (j) the per-helper flag enumerations (`BPF_F_*` ~120 distinct flag families); (k) CO-RE relocation metadata (`bpf_core_relo`, `bpf_core_relo_kind`); (l) FIB/MTU lookup, flow-dissector keys, and netfilter link metadata; (m) the `BPF_MAP_TYPE_INSN_ARRAY` instruction-index map value (`bpf_insn_array_value`).

Critical for: GPL-clean BPF surface usable by libbpf, iproute2, bpftool, perf, systemd, Cilium, and any CO-RE bytecode.

This Tier-5 covers `include/uapi/linux/bpf.h` (~7705 lines).

### Acceptance Criteria

- [ ] AC-1: `sizeof(BpfInsn) == 8` with `dst_reg`/`src_reg` packed into a single byte; `off` is `i16` little-endian; `imm` is `i32` little-endian.
- [ ] AC-2: `sizeof(BpfAttr) == sizeof(union bpf_attr)` from upstream at commit `27a26ccfd528da725a999ea1e3102503c61eb655`, with 8-byte alignment.
- [ ] AC-3: Every enum value in `BpfCmd`, `BpfMapType`, `BpfProgType`, `BpfAttachType`, `BpfLinkType`, `BpfPerfEventType`, `BpfCoreReloKind`, `BpfStatsType` matches the upstream integer exactly.
- [ ] AC-4: `BPF_OBJ_NAME_LEN == 16`, `BPF_TAG_SIZE == 8`, `BPF_BUILD_ID_SIZE == 20`.
- [ ] AC-5: `MAX_BPF_REG == __MAX_BPF_REG == 11`.
- [ ] AC-6: `BPF_F_LINK` is bit 13; `BPF_F_TOKEN_FD` is bit 16; `BPF_F_PATH_FD` is bit 14.
- [ ] AC-7: `struct bpf_lpm_trie_key_u8` aliases `bpf_lpm_trie_key_hdr` via union, with `prefixlen` at offset 0.
- [ ] AC-8: `struct bpf_perf_event_value` is 24 bytes.
- [ ] AC-9: `struct bpf_func_info` is 8 bytes; `struct bpf_line_info` is 16 bytes.
- [ ] AC-10: `struct bpf_core_relo` is 16 bytes.
- [ ] AC-11: `struct bpf_insn_array_value` is 16 bytes.
- [ ] AC-12: `struct bpf_spin_lock` is 4 bytes; `bpf_timer` is 16 bytes; `bpf_dynptr` is 16 bytes; `bpf_rb_node` is 32 bytes; `bpf_list_node` is 24 bytes; `bpf_refcount` is 4 bytes.
- [ ] AC-13: `BPF_RB_AVAIL_DATA/RING_SIZE/CONS_POS/PROD_POS/OVERWRITE_POS` are 0..4.
- [ ] AC-14: `BPF_RINGBUF_HDR_SZ == 8`, `BUSY_BIT == 1<<31`, `DISCARD_BIT == 1<<30`.
- [ ] AC-15: `BPF_F_CURRENT_CPU == BPF_F_INDEX_MASK == 0xffffffff`; `BPF_F_CURRENT_NETNS == -1L`.
- [ ] AC-16: `BPF_F_CTXLEN_MASK == 0xfffff << 32`.
- [ ] AC-17: `BPF_ADJ_ROOM_ENCAP_L2_SHIFT == 56`, `_MASK == 0xff`.
- [ ] AC-18: `struct bpf_stack_build_id` size is 32 bytes (4 + 20 + 8) with the trailing union aligned.
- [ ] AC-19: Every `BPF_F_*` flag listed in this document corresponds to the exact bit position upstream.
- [ ] AC-20: libbpf (head of release branch) compiles unmodified against the Rookery headers and successfully loads a CO-RE program against the Rookery kernel.

### Architecture

Rust struct layout uses `#[repr(C)]` and explicit `#[repr(C, align(8))]` for the info / opaque types. C unions are modeled either as Rust `union`s (where members are POD), or as `[u8; N]` byte blobs with accessor methods, since the discriminator is always the syscall command, not a field inside the union.

```
#[repr(C)]
pub struct BpfInsn {
    pub code: u8,
    pub regs: u8,                           // dst_reg:4 (low nibble) | src_reg:4 (high nibble)
    pub off: i16,
    pub imm: i32,
}

impl BpfInsn {
    #[inline] pub const fn dst_reg(&self) -> u8 { self.regs & 0x0f }
    #[inline] pub const fn src_reg(&self) -> u8 { (self.regs >> 4) & 0x0f }
}
```

```
#[repr(u32)]
pub enum BpfCmd {
    MAP_CREATE = 0, MAP_LOOKUP_ELEM, MAP_UPDATE_ELEM, MAP_DELETE_ELEM,
    MAP_GET_NEXT_KEY, PROG_LOAD, OBJ_PIN, OBJ_GET, PROG_ATTACH, PROG_DETACH,
    PROG_TEST_RUN, PROG_GET_NEXT_ID, MAP_GET_NEXT_ID, PROG_GET_FD_BY_ID,
    MAP_GET_FD_BY_ID, OBJ_GET_INFO_BY_FD, PROG_QUERY, RAW_TRACEPOINT_OPEN,
    BTF_LOAD, BTF_GET_FD_BY_ID, TASK_FD_QUERY, MAP_LOOKUP_AND_DELETE_ELEM,
    MAP_FREEZE, BTF_GET_NEXT_ID, MAP_LOOKUP_BATCH, MAP_LOOKUP_AND_DELETE_BATCH,
    MAP_UPDATE_BATCH, MAP_DELETE_BATCH, LINK_CREATE, LINK_UPDATE,
    LINK_GET_FD_BY_ID, LINK_GET_NEXT_ID, ENABLE_STATS, ITER_CREATE,
    LINK_DETACH, PROG_BIND_MAP, TOKEN_CREATE, PROG_STREAM_READ_BY_FD,
    PROG_ASSOC_STRUCT_OPS,
}
```

```
#[repr(C, align(8))]
pub union BpfAttr {
    pub map_create: BpfAttrMapCreate,
    pub map_elem: BpfAttrMapElem,
    pub batch: BpfAttrMapBatch,
    pub prog_load: BpfAttrProgLoad,
    pub obj: BpfAttrObj,
    pub prog_attach: BpfAttrProgAttach,
    pub test: BpfAttrTestRun,
    pub get_id: BpfAttrGetId,
    pub info: BpfAttrInfoByFd,
    pub query: BpfAttrProgQuery,
    pub raw_tracepoint: BpfAttrRawTpOpen,
    pub btf_load: BpfAttrBtfLoad,
    pub task_fd_query: BpfAttrTaskFdQuery,
    pub link_create: BpfAttrLinkCreate,
    pub link_update: BpfAttrLinkUpdate,
    pub link_detach: BpfAttrLinkDetach,
    pub enable_stats: BpfAttrEnableStats,
    pub iter_create: BpfAttrIterCreate,
    pub prog_bind_map: BpfAttrProgBindMap,
    pub token_create: BpfAttrTokenCreate,
    pub prog_stream_read: BpfAttrProgStreamRead,
    pub prog_assoc_struct_ops: BpfAttrProgAssocStructOps,
    _raw: [u8; 0xC0], // upper bound; verified via const_assert against upstream
}
```

```
#[repr(C)]
pub struct BpfAttrMapCreate {
    pub map_type: u32, pub key_size: u32, pub value_size: u32, pub max_entries: u32,
    pub map_flags: u32, pub inner_map_fd: u32, pub numa_node: u32,
    pub map_name: [u8; 16],                 // BPF_OBJ_NAME_LEN
    pub map_ifindex: u32, pub btf_fd: u32,
    pub btf_key_type_id: u32, pub btf_value_type_id: u32,
    pub btf_vmlinux_value_type_id: u32,
    pub map_extra: u64,
    pub value_type_btf_obj_fd: i32,
    pub map_token_fd: i32,
    pub excl_prog_hash: u64,                // __aligned_u64
    pub excl_prog_hash_size: u32,
}

#[repr(C)]
pub struct BpfAttrMapElem {
    pub map_fd: u32,
    pub key: u64,                           // __aligned_u64
    pub value_or_next_key: u64,             // union { value | next_key }
    pub flags: u64,
}

#[repr(C)]
pub struct BpfAttrMapBatch {
    pub in_batch: u64, pub out_batch: u64,
    pub keys: u64, pub values: u64,
    pub count: u32, pub map_fd: u32,
    pub elem_flags: u64, pub flags: u64,
}

#[repr(C)]
pub struct BpfAttrProgLoad {
    pub prog_type: u32, pub insn_cnt: u32,
    pub insns: u64, pub license: u64,
    pub log_level: u32, pub log_size: u32, pub log_buf: u64,
    pub kern_version: u32, pub prog_flags: u32,
    pub prog_name: [u8; 16],
    pub prog_ifindex: u32, pub expected_attach_type: u32,
    pub prog_btf_fd: u32,
    pub func_info_rec_size: u32, pub func_info: u64, pub func_info_cnt: u32,
    pub line_info_rec_size: u32, pub line_info: u64, pub line_info_cnt: u32,
    pub attach_btf_id: u32, pub attach_prog_or_btf_obj_fd: u32,
    pub core_relo_cnt: u32, pub fd_array: u64, pub core_relos: u64,
    pub core_relo_rec_size: u32, pub log_true_size: u32, pub prog_token_fd: i32,
    pub fd_array_cnt: u32,
    pub signature: u64, pub signature_size: u32, pub keyring_id: i32,
}
```

```
#[repr(C, align(8))]
pub struct BpfProgInfo {
    pub type_: u32, pub id: u32, pub tag: [u8; 8],
    pub jited_prog_len: u32, pub xlated_prog_len: u32,
    pub jited_prog_insns: u64, pub xlated_prog_insns: u64,
    pub load_time: u64, pub created_by_uid: u32,
    pub nr_map_ids: u32, pub map_ids: u64,
    pub name: [u8; 16], pub ifindex: u32,
    pub flags_pad: u32,                     // gpl_compatible:1 | _:31
    pub netns_dev: u64, pub netns_ino: u64,
    pub nr_jited_ksyms: u32, pub nr_jited_func_lens: u32,
    pub jited_ksyms: u64, pub jited_func_lens: u64,
    pub btf_id: u32, pub func_info_rec_size: u32,
    pub func_info: u64, pub nr_func_info: u32, pub nr_line_info: u32,
    pub line_info: u64, pub jited_line_info: u64,
    pub nr_jited_line_info: u32, pub line_info_rec_size: u32, pub jited_line_info_rec_size: u32,
    pub nr_prog_tags: u32, pub prog_tags: u64,
    pub run_time_ns: u64, pub run_cnt: u64, pub recursion_misses: u64,
    pub verified_insns: u32, pub attach_btf_obj_id: u32, pub attach_btf_id: u32,
}

#[repr(C, align(8))]
pub struct BpfMapInfo {
    pub type_: u32, pub id: u32,
    pub key_size: u32, pub value_size: u32, pub max_entries: u32, pub map_flags: u32,
    pub name: [u8; 16], pub ifindex: u32, pub btf_vmlinux_value_type_id: u32,
    pub netns_dev: u64, pub netns_ino: u64,
    pub btf_id: u32, pub btf_key_type_id: u32, pub btf_value_type_id: u32, pub btf_vmlinux_id: u32,
    pub map_extra: u64, pub hash: u64, pub hash_size: u32,
}

#[repr(C, align(8))]
pub struct BpfBtfInfo {
    pub btf: u64, pub btf_size: u32, pub id: u32,
    pub name: u64, pub name_len: u32, pub kernel_btf: u32,
}

#[repr(C, align(8))]
pub struct BpfLinkInfoHdr {
    pub type_: u32, pub id: u32, pub prog_id: u32,
    // followed by per-link-type body modeled as a separate union
}

#[repr(C)]
pub struct BpfFuncInfo { pub insn_off: u32, pub type_id: u32 }
#[repr(C)]
pub struct BpfLineInfo { pub insn_off: u32, pub file_name_off: u32, pub line_off: u32, pub line_col: u32 }
#[repr(C)]
pub struct BpfPerfEventValue { pub counter: u64, pub enabled: u64, pub running: u64 }
#[repr(C)]
pub struct BpfCoreRelo { pub insn_off: u32, pub type_id: u32, pub access_str_off: u32, pub kind: u32 /* BpfCoreReloKind */ }
#[repr(C)]
pub struct BpfStackBuildId { pub status: i32, pub build_id: [u8; 20], pub offset_or_ip: u64 }
#[repr(C)]
pub struct BpfInsnArrayValue { pub orig_off: u32, pub xlated_off: u32, pub jitted_off: u32, pub _pad: u32 }
#[repr(C)]
pub struct BpfLpmTrieKeyHdr { pub prefixlen: u32 }
#[repr(C)]
pub struct BpfCgroupStorageKey { pub cgroup_inode_id: u64, pub attach_type: u32 }
```

Opaque BPF runtime objects (exact size + alignment required because BPF programs and verifier rely on them):

```
#[repr(C, align(8))] pub struct BpfSpinLock     { pub val: u32 }
#[repr(C, align(8))] pub struct BpfTimer        { pub _opaque: [u64; 2] }
#[repr(C, align(8))] pub struct BpfTaskWork     { pub _opaque: u64 }
#[repr(C, align(8))] pub struct BpfWq           { pub _opaque: [u64; 2] }
#[repr(C, align(8))] pub struct BpfDynptr       { pub _opaque: [u64; 2] }
#[repr(C, align(8))] pub struct BpfListHead     { pub _opaque: [u64; 2] }
#[repr(C, align(8))] pub struct BpfListNode     { pub _opaque: [u64; 3] }
#[repr(C, align(8))] pub struct BpfRbRoot       { pub _opaque: [u64; 2] }
#[repr(C, align(8))] pub struct BpfRbNode       { pub _opaque: [u64; 4] }
#[repr(C, align(4))] pub struct BpfRefcount     { pub _opaque: u32 }
#[repr(C, align(8))] pub struct BpfIterNum      { pub _opaque: u64 }
```

Per-syscall signatures (semantics in Tier-3 docs):

- `bpf(cmd: u32, attr: *mut BpfAttr, size: u32) -> i32`

The third argument is the size of `attr` the caller knows; the kernel uses it to support forward/backward compatibility.

### Out of Scope

- `bpf(2)` syscall implementation (`kernel/bpf/syscall.c`; Tier-3 in `bpf/syscall.md`).
- BPF verifier implementation (`kernel/bpf/verifier.c`; Tier-3 in `bpf/verifier.md`).
- BPF JIT backends (`arch/*/net/bpf_jit_*.c`; Tier-3 per-arch).
- BTF parser implementation (`kernel/bpf/btf.c`; Tier-3 in `bpf/btf.md`).
- Per-helper function bodies (Tier-3 per-subsystem: `net/`, `tracing/`, `cgroup/`, `lsm/`).
- BPF LSM (`security/bpf/`; Tier-3 in `security/bpf-lsm.md`).
- Sockmap / sockhash / reuseport (Tier-3 in `net/sock_map.md`).
- XDP and AF_XDP (Tier-3 in `net/xdp.md`, `net/af_xdp.md`).
- libbpf and bpftool (userspace, out of kernel scope).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_insn` | per-instruction encoding | `BpfInsn` |
| `enum bpf_cmd` | bpf(2) command opcodes | `BpfCmd` |
| `union bpf_attr` | per-command argument tagged-union | `BpfAttr` |
| `enum bpf_map_type` | per-map kind | `BpfMapType` |
| `enum bpf_prog_type` | per-program kind | `BpfProgType` |
| `enum bpf_attach_type` | per-attach point | `BpfAttachType` |
| `enum bpf_link_type` | per-link kind | `BpfLinkType` |
| `enum bpf_perf_event_type` | per-perf-link sub-kind | `BpfPerfEventType` |
| `enum bpf_func_id` | per-helper ID | `BpfFuncId` |
| `struct bpf_prog_info` | per-program introspection | `BpfProgInfo` |
| `struct bpf_map_info` | per-map introspection | `BpfMapInfo` |
| `struct bpf_btf_info` | per-BTF introspection | `BpfBtfInfo` |
| `struct bpf_link_info` | per-link introspection | `BpfLinkInfo` |
| `struct bpf_token_info` | per-token introspection | `BpfTokenInfo` |
| `struct bpf_func_info` | per-BTF function info | `BpfFuncInfo` |
| `struct bpf_line_info` | per-BTF line info | `BpfLineInfo` |
| `struct bpf_core_relo` | per-CO-RE relocation | `BpfCoreRelo` |
| `enum bpf_core_relo_kind` | per-CO-RE relocation kind | `BpfCoreReloKind` |
| `struct bpf_lpm_trie_key` (deprecated) | per-LPM-trie key | `BpfLpmTrieKey` |
| `struct bpf_lpm_trie_key_hdr` / `_u8` | LPM key header + u8 variant | `BpfLpmTrieKey{Hdr,U8}` |
| `struct bpf_cgroup_storage_key` | per-cgroup-storage key | `BpfCgroupStorageKey` |
| `enum bpf_cgroup_iter_order` | per-cgroup iter walk order | `BpfCgroupIterOrder` |
| `union bpf_iter_link_info` | per-iter parameters | `BpfIterLinkInfo` |
| `enum bpf_lwt_encap_mode` | per-LWT-encap mode | `BpfLwtEncapMode` |
| `enum bpf_adj_room_mode` / `bpf_hdr_start_off` | per-skb helpers mode | `Bpf{AdjRoomMode,HdrStartOff}` |
| `struct __sk_buff` | per-skb-program context | `SkBuffCtx` |
| `struct xdp_md` | per-XDP context | `XdpMd` |
| `struct bpf_sock` | per-socket context | `BpfSock` |
| `struct bpf_tcp_sock` | per-TCP-sock context | `BpfTcpSock` |
| `struct bpf_sock_tuple` | per-tuple helper arg | `BpfSockTuple` |
| `struct bpf_xdp_sock` | per-xsk context | `BpfXdpSock` |
| `struct bpf_sock_ops` | per-sockops context | `BpfSockOps` |
| `struct bpf_sock_addr` | per-cgroup-bind/connect ctx | `BpfSockAddr` |
| `struct bpf_sockopt` | per-cgroup-sockopt ctx | `BpfSockopt` |
| `struct bpf_sysctl` | per-sysctl ctx | `BpfSysctl` |
| `struct bpf_sk_lookup` | per-sk-lookup ctx | `BpfSkLookup` |
| `struct bpf_cgroup_dev_ctx` | per-device-cgroup ctx | `BpfCgroupDevCtx` |
| `struct bpf_pidns_info` | per-pidns helper | `BpfPidnsInfo` |
| `struct bpf_raw_tracepoint_args` | per-raw-tp ctx | `BpfRawTracepointArgs` |
| `struct sk_msg_md` / `sk_reuseport_md` | per-msg / reuseport ctx | `SkMsgMd` / `SkReuseportMd` |
| `struct bpf_flow_keys` | per-flow-dissector ctx | `BpfFlowKeys` |
| `struct bpf_fib_lookup` / `bpf_redir_neigh` | per-FIB / redirect-neigh | `BpfFibLookup` / `BpfRedirNeigh` |
| `struct bpf_tunnel_key` / `bpf_xfrm_state` | per-tunnel / per-xfrm ctx | `BpfTunnelKey` / `BpfXfrmState` |
| `struct bpf_devmap_val` / `bpf_cpumap_val` | per-devmap / cpumap value | `BpfDevmapVal` / `BpfCpumapVal` |
| `struct bpf_spin_lock` / `bpf_timer` / `bpf_task_work` / `bpf_wq` | opaque kernel objects in BPF maps | `BpfSpinLock` / `BpfTimer` / `BpfTaskWork` / `BpfWq` |
| `struct bpf_dynptr` / `bpf_list_head` / `bpf_list_node` / `bpf_rb_root` / `bpf_rb_node` / `bpf_refcount` / `bpf_iter_num` | opaque BPF runtime types | `Bpf{Dynptr,ListHead,ListNode,RbRoot,RbNode,Refcount,IterNum}` |
| `struct bpf_perf_event_value` | per-perf counter sample | `BpfPerfEventValue` |
| `struct bpf_stack_build_id` | per-stack-trace build-id | `BpfStackBuildId` |
| `struct bpf_insn_array_value` | per-INSN_ARRAY map value | `BpfInsnArrayValue` |
| `struct btf_ptr` | per-typed-pointer helper arg | `BtfPtr` |
| `struct btf_*` (from linux/btf.h) | per-BTF type record | `Btf*` |

### abi surface (constants + structs)

### Instruction encoding (BPF_*)

Classes (from `linux/bpf_common.h`): `BPF_LD` (0x00), `BPF_LDX` (0x01), `BPF_ST` (0x02), `BPF_STX` (0x03), `BPF_ALU` (0x04), `BPF_JMP` (0x05), `BPF_RET` (0x06; reused in eBPF as `BPF_JMP32`), `BPF_MISC` (0x07; reused as `BPF_ALU64`). eBPF extensions: `BPF_JMP32` (0x06), `BPF_ALU64` (0x07).

Sizes: `BPF_W` (0x00), `BPF_H` (0x08), `BPF_B` (0x10), `BPF_DW` (0x18). Modifier: `BPF_MEMSX` (0x80) load-with-sign-extension. Atomic: `BPF_ATOMIC` (0xc0), legacy alias `BPF_XADD`.

ALU/JMP: `BPF_MOV` (0xb0), `BPF_ARSH` (0xc0).

Endianness conversion: `BPF_END` (0xd0), `BPF_TO_LE` (0x00), `BPF_TO_BE` (0x08), aliases `BPF_FROM_LE`/`BPF_FROM_BE`.

JMP opcodes: `BPF_JA` (0x00), `BPF_JEQ` (0x10), `BPF_JGT` (0x20), `BPF_JGE` (0x30), `BPF_JSET` (0x40), `BPF_JNE` (0x50), `BPF_JSGT` (0x60), `BPF_JSGE` (0x70), `BPF_CALL` (0x80), `BPF_EXIT` (0x90), `BPF_JLT` (0xa0), `BPF_JLE` (0xb0), `BPF_JSLT` (0xc0), `BPF_JSLE` (0xd0), `BPF_JCOND` (0xe0).

Atomic op type (in `imm`): `BPF_FETCH` (0x01), `BPF_XCHG` (0xe0 | FETCH), `BPF_CMPXCHG` (0xf0 | FETCH), `BPF_LOAD_ACQ` (0x100), `BPF_STORE_REL` (0x110).

Pseudo-jumps: `enum bpf_cond_pseudo_jmp { BPF_MAY_GOTO = 0 }`.

Registers: `BPF_REG_0..BPF_REG_10` (`__MAX_BPF_REG = 11`, `MAX_BPF_REG = 11`). R10 is the read-only frame pointer.

### enum bpf_cmd (bpf(2) commands)

`BPF_MAP_CREATE`, `BPF_MAP_LOOKUP_ELEM`, `BPF_MAP_UPDATE_ELEM`, `BPF_MAP_DELETE_ELEM`, `BPF_MAP_GET_NEXT_KEY`, `BPF_PROG_LOAD`, `BPF_OBJ_PIN`, `BPF_OBJ_GET`, `BPF_PROG_ATTACH`, `BPF_PROG_DETACH`, `BPF_PROG_TEST_RUN` (alias `BPF_PROG_RUN`), `BPF_PROG_GET_NEXT_ID`, `BPF_MAP_GET_NEXT_ID`, `BPF_PROG_GET_FD_BY_ID`, `BPF_MAP_GET_FD_BY_ID`, `BPF_OBJ_GET_INFO_BY_FD`, `BPF_PROG_QUERY`, `BPF_RAW_TRACEPOINT_OPEN`, `BPF_BTF_LOAD`, `BPF_BTF_GET_FD_BY_ID`, `BPF_TASK_FD_QUERY`, `BPF_MAP_LOOKUP_AND_DELETE_ELEM`, `BPF_MAP_FREEZE`, `BPF_BTF_GET_NEXT_ID`, `BPF_MAP_LOOKUP_BATCH`, `BPF_MAP_LOOKUP_AND_DELETE_BATCH`, `BPF_MAP_UPDATE_BATCH`, `BPF_MAP_DELETE_BATCH`, `BPF_LINK_CREATE`, `BPF_LINK_UPDATE`, `BPF_LINK_GET_FD_BY_ID`, `BPF_LINK_GET_NEXT_ID`, `BPF_ENABLE_STATS`, `BPF_ITER_CREATE`, `BPF_LINK_DETACH`, `BPF_PROG_BIND_MAP`, `BPF_TOKEN_CREATE`, `BPF_PROG_STREAM_READ_BY_FD`, `BPF_PROG_ASSOC_STRUCT_OPS`.

### enum bpf_map_type

`UNSPEC` (0), `HASH`, `ARRAY`, `PROG_ARRAY`, `PERF_EVENT_ARRAY`, `PERCPU_HASH`, `PERCPU_ARRAY`, `STACK_TRACE`, `CGROUP_ARRAY`, `LRU_HASH`, `LRU_PERCPU_HASH`, `LPM_TRIE`, `ARRAY_OF_MAPS`, `HASH_OF_MAPS`, `DEVMAP`, `SOCKMAP`, `CPUMAP`, `XSKMAP`, `SOCKHASH`, `CGROUP_STORAGE_DEPRECATED` (alias `CGROUP_STORAGE`), `REUSEPORT_SOCKARRAY`, `PERCPU_CGROUP_STORAGE_DEPRECATED` (alias `PERCPU_CGROUP_STORAGE`), `QUEUE`, `STACK`, `SK_STORAGE`, `DEVMAP_HASH`, `STRUCT_OPS`, `RINGBUF`, `INODE_STORAGE`, `TASK_STORAGE`, `BLOOM_FILTER`, `USER_RINGBUF`, `CGRP_STORAGE`, `ARENA`, `INSN_ARRAY`.

### enum bpf_prog_type

`UNSPEC` (0), `SOCKET_FILTER`, `KPROBE`, `SCHED_CLS`, `SCHED_ACT`, `TRACEPOINT`, `XDP`, `PERF_EVENT`, `CGROUP_SKB`, `CGROUP_SOCK`, `LWT_IN`, `LWT_OUT`, `LWT_XMIT`, `SOCK_OPS`, `SK_SKB`, `CGROUP_DEVICE`, `SK_MSG`, `RAW_TRACEPOINT`, `CGROUP_SOCK_ADDR`, `LWT_SEG6LOCAL`, `LIRC_MODE2`, `SK_REUSEPORT`, `FLOW_DISSECTOR`, `CGROUP_SYSCTL`, `RAW_TRACEPOINT_WRITABLE`, `CGROUP_SOCKOPT`, `TRACING`, `STRUCT_OPS`, `EXT`, `LSM`, `SK_LOOKUP`, `SYSCALL`, `NETFILTER`.

### enum bpf_attach_type

`CGROUP_INET_INGRESS`, `CGROUP_INET_EGRESS`, `CGROUP_INET_SOCK_CREATE`, `CGROUP_SOCK_OPS`, `SK_SKB_STREAM_PARSER`, `SK_SKB_STREAM_VERDICT`, `CGROUP_DEVICE`, `SK_MSG_VERDICT`, `CGROUP_INET4_BIND`, `CGROUP_INET6_BIND`, `CGROUP_INET4_CONNECT`, `CGROUP_INET6_CONNECT`, `CGROUP_INET4_POST_BIND`, `CGROUP_INET6_POST_BIND`, `CGROUP_UDP4_SENDMSG`, `CGROUP_UDP6_SENDMSG`, `LIRC_MODE2`, `FLOW_DISSECTOR`, `CGROUP_SYSCTL`, `CGROUP_UDP4_RECVMSG`, `CGROUP_UDP6_RECVMSG`, `CGROUP_GETSOCKOPT`, `CGROUP_SETSOCKOPT`, `TRACE_RAW_TP`, `TRACE_FENTRY`, `TRACE_FEXIT`, `MODIFY_RETURN`, `LSM_MAC`, `TRACE_ITER`, `CGROUP_INET4_GETPEERNAME`, `CGROUP_INET6_GETPEERNAME`, `CGROUP_INET4_GETSOCKNAME`, `CGROUP_INET6_GETSOCKNAME`, `XDP_DEVMAP`, `CGROUP_INET_SOCK_RELEASE`, `XDP_CPUMAP`, `SK_LOOKUP`, `XDP`, `SK_SKB_VERDICT`, `SK_REUSEPORT_SELECT`, `SK_REUSEPORT_SELECT_OR_MIGRATE`, `PERF_EVENT`, `TRACE_KPROBE_MULTI`, `LSM_CGROUP`, `STRUCT_OPS`, `NETFILTER`, `TCX_INGRESS`, `TCX_EGRESS`, `TRACE_UPROBE_MULTI`, `CGROUP_UNIX_CONNECT`, `CGROUP_UNIX_SENDMSG`, `CGROUP_UNIX_RECVMSG`, `CGROUP_UNIX_GETPEERNAME`, `CGROUP_UNIX_GETSOCKNAME`, `NETKIT_PRIMARY`, `NETKIT_PEER`, `TRACE_KPROBE_SESSION`, `TRACE_UPROBE_SESSION`, `TRACE_FSESSION`. Sentinel `__MAX_BPF_ATTACH_TYPE`.

### enum bpf_link_type

`UNSPEC` (0), `RAW_TRACEPOINT` (1), `TRACING` (2), `CGROUP` (3), `ITER` (4), `NETNS` (5), `XDP` (6), `PERF_EVENT` (7), `KPROBE_MULTI` (8), `STRUCT_OPS` (9), `NETFILTER` (10), `TCX` (11), `UPROBE_MULTI` (12), `NETKIT` (13), `SOCKMAP` (14).

### enum bpf_perf_event_type

`UNSPEC` (0), `UPROBE` (1), `URETPROBE` (2), `KPROBE` (3), `KRETPROBE` (4), `TRACEPOINT` (5), `EVENT` (6).

### BPF_F_* attach / load / map flags

Attach/replace: `BPF_F_ALLOW_OVERRIDE` (1<<0), `BPF_F_ALLOW_MULTI` (1<<1), `BPF_F_REPLACE` (1<<2), `BPF_F_BEFORE` (1<<3), `BPF_F_AFTER` (1<<4), `BPF_F_ID` (1<<5), `BPF_F_PREORDER` (1<<6), `BPF_F_LINK` (1<<13).

Prog load: `BPF_F_STRICT_ALIGNMENT` (1<<0), `BPF_F_ANY_ALIGNMENT` (1<<1), `BPF_F_TEST_RND_HI32` (1<<2), `BPF_F_TEST_STATE_FREQ` (1<<3), `BPF_F_SLEEPABLE` (1<<4), `BPF_F_XDP_HAS_FRAGS` (1<<5), `BPF_F_XDP_DEV_BOUND_ONLY` (1<<6), `BPF_F_TEST_REG_INVARIANTS` (1<<7).

Kprobe/uprobe multi flags: `BPF_F_KPROBE_MULTI_RETURN` (1<<0), `BPF_F_UPROBE_MULTI_RETURN` (1<<0).

Netfilter: `BPF_F_NETFILTER_IP_DEFRAG` (1<<0).

Map elem flags (`bpf_attr->elem_flags`): `BPF_ANY` (0), `BPF_NOEXIST` (1), `BPF_EXIST` (2), `BPF_F_LOCK` (4), `BPF_F_CPU` (8), `BPF_F_ALL_CPUS` (16).

Map create flags: `BPF_F_NO_PREALLOC` (1<<0), `BPF_F_NO_COMMON_LRU` (1<<1), `BPF_F_NUMA_NODE` (1<<2), `BPF_F_RDONLY` (1<<3), `BPF_F_WRONLY` (1<<4), `BPF_F_STACK_BUILD_ID` (1<<5), `BPF_F_ZERO_SEED` (1<<6), `BPF_F_RDONLY_PROG` (1<<7), `BPF_F_WRONLY_PROG` (1<<8), `BPF_F_CLONE` (1<<9), `BPF_F_MMAPABLE` (1<<10), `BPF_F_PRESERVE_ELEMS` (1<<11), `BPF_F_INNER_MAP` (1<<12), `BPF_F_LINK` (1<<13), `BPF_F_PATH_FD` (1<<14), `BPF_F_VTYPE_BTF_OBJ_FD` (1<<15), `BPF_F_TOKEN_FD` (1<<16), `BPF_F_SEGV_ON_FAULT` (1<<17), `BPF_F_NO_USER_CONV` (1<<18), `BPF_F_RB_OVERWRITE` (1<<19).

Query/test: `BPF_F_QUERY_EFFECTIVE` (1<<0), `BPF_F_TEST_RUN_ON_CPU` (1<<0), `BPF_F_TEST_XDP_LIVE_FRAMES` (1<<1), `BPF_F_TEST_SKB_CHECKSUM_COMPLETE` (1<<2).

Helper flags (per-family): `BPF_F_RECOMPUTE_CSUM`, `BPF_F_INVALIDATE_HASH`, `BPF_F_HDR_FIELD_MASK` (0xf), `BPF_F_PSEUDO_HDR` (1<<4), `BPF_F_MARK_MANGLED_0` (1<<5), `BPF_F_MARK_ENFORCE` (1<<6), `BPF_F_IPV6` (1<<7), `BPF_F_TUNINFO_IPV6` (1<<0), `BPF_F_SKIP_FIELD_MASK` (0xff), `BPF_F_USER_STACK` (1<<8), `BPF_F_FAST_STACK_CMP` (1<<9), `BPF_F_REUSE_STACKID` (1<<10), `BPF_F_USER_BUILD_ID` (1<<11), `BPF_F_ZERO_CSUM_TX` (1<<1), `BPF_F_DONT_FRAGMENT` (1<<2), `BPF_F_SEQ_NUMBER` (1<<3), `BPF_F_NO_TUNNEL_KEY` (1<<4), `BPF_F_TUNINFO_FLAGS` (1<<4), `BPF_F_INDEX_MASK` (0xffffffff), `BPF_F_CURRENT_CPU` (= INDEX_MASK), `BPF_F_CTXLEN_MASK` (0xfffff << 32), `BPF_F_CURRENT_NETNS` (-1L), `BPF_F_SYSCTL_BASE_NAME` (1<<0), `BPF_LOCAL_STORAGE_GET_F_CREATE` (1<<0), `BPF_F_GET_BRANCH_RECORDS_SIZE` (1<<0), `BPF_F_INGRESS` (1<<0), `BPF_F_BROADCAST` (1<<3), `BPF_F_EXCLUDE_INGRESS` (1<<4), `BPF_F_BPRM_SECUREEXEC` (1<<0), `BPF_F_TIMER_ABS` (1<<0), `BPF_F_TIMER_CPU_PIN` (1<<1), `BPF_F_PAD_ZEROS` (1<<0).

Ringbuf: `BPF_RB_NO_WAKEUP` (1<<0), `BPF_RB_FORCE_WAKEUP` (1<<1), `BPF_RB_AVAIL_DATA` (0), `BPF_RB_RING_SIZE` (1), `BPF_RB_CONS_POS` (2), `BPF_RB_PROD_POS` (3), `BPF_RB_OVERWRITE_POS` (4), `BPF_RINGBUF_BUSY_BIT` (1<<31), `BPF_RINGBUF_DISCARD_BIT` (1<<30), `BPF_RINGBUF_HDR_SZ` (8).

CSUM levels: `BPF_CSUM_LEVEL_QUERY/INC/DEC/RESET` (0..3).

skb_adjust_room: `BPF_F_ADJ_ROOM_FIXED_GSO` (1<<0), `_ENCAP_L3_IPV4` (1<<1), `_ENCAP_L3_IPV6` (1<<2), `_ENCAP_L4_GRE` (1<<3), `_ENCAP_L4_UDP` (1<<4), `_NO_CSUM_RESET` (1<<5), `_ENCAP_L2_ETH` (1<<6), `_DECAP_L3_IPV4` (1<<7), `_DECAP_L3_IPV6` (1<<8). `BPF_ADJ_ROOM_ENCAP_L2_MASK` (0xff), `_SHIFT` (56).

sk_lookup helper flags: `BPF_SK_LOOKUP_F_REPLACE` (1<<0), `BPF_SK_LOOKUP_F_NO_REUSEPORT` (1<<1).

### enum bpf_stats_type / build-id

`BPF_STATS_RUN_TIME` (0). `BPF_BUILD_ID_SIZE` (20). `BPF_STACK_BUILD_ID_EMPTY` (0), `_VALID` (1), `_IP` (2).

### Modes / actions

`enum bpf_adj_room_mode { BPF_ADJ_ROOM_NET, BPF_ADJ_ROOM_MAC }`.
`enum bpf_hdr_start_off { BPF_HDR_START_MAC, BPF_HDR_START_NET }`.
`enum bpf_lwt_encap_mode { BPF_LWT_ENCAP_SEG6, BPF_LWT_ENCAP_SEG6_INLINE, BPF_LWT_ENCAP_IP }`.
`enum bpf_ret_code { BPF_OK, BPF_DROP, BPF_REDIRECT, BPF_LWT_REROUTE, BPF_FLOW_DISSECTOR_CONTINUE, ... }`.
`enum xdp_action { XDP_ABORTED, XDP_DROP, XDP_PASS, XDP_TX, XDP_REDIRECT }`.
`enum sk_action { SK_DROP, SK_PASS }`.
`enum tcx_action_base { TCX_NEXT, TCX_PASS, TCX_DROP, TCX_REDIRECT }`.
`enum bpf_fib_lookup`: `DIRECT` (1<<0), `OUTPUT` (1<<1), `SKIP_NEIGH` (1<<2), `TBID` (1<<3), `SRC` (1<<4), `MARK` (1<<5). `BPF_FIB_LKUP_RET_*` ten enumerators covering SUCCESS, BLACKHOLE, UNREACHABLE, PROHIBIT, NOT_FWDED, FWD_DISABLED, UNSUPP_LWT, NO_NEIGH, FRAG_NEEDED, NO_SRC_ADDR.
`enum bpf_check_mtu_flags { BPF_MTU_CHK_SEGS = 1<<0 }`; `enum bpf_check_mtu_ret { SUCCESS, FRAG_NEEDED, SEGS_TOOBIG }`.
`enum bpf_task_fd_type { RAW_TRACEPOINT, TRACEPOINT, KPROBE, KRETPROBE, UPROBE, URETPROBE }`.
`enum bpf_flow_dissector { BPF_FLOW_DISSECTOR_F_PARSE_1ST_FRAG, _STOP_AT_FLOW_LABEL, _STOP_AT_ENCAP }`.
`enum bpf_devcg { ACC_MKNOD, ACC_READ, ACC_WRITE | DEV_BLOCK, DEV_CHAR }`.
`enum bpf_addr_space_cast`.

### BTF kinds (`enum btf_kind` from companion header)

`BTF_KIND_UNKN` (0), `INT` (1), `PTR` (2), `ARRAY` (3), `STRUCT` (4), `UNION` (5), `ENUM` (6), `FWD` (7), `TYPEDEF` (8), `VOLATILE` (9), `CONST` (10), `RESTRICT` (11), `FUNC` (12), `FUNC_PROTO` (13), `VAR` (14), `DATASEC` (15), `FLOAT` (16), `DECL_TAG` (17), `TYPE_TAG` (18), `ENUM64` (19). `BTF_KIND_MAX = NR_BTF_KINDS - 1`. Companion records: `btf_int`, `btf_enum`, `btf_array`, `btf_member`, `btf_param`, `btf_var`, `btf_var_secinfo`, `btf_decl_tag`, `btf_enum64`.

### BTF display flags

`BTF_F_COMPACT` (1<<0), `BTF_F_NONAME` (1<<1), `BTF_F_PTR_RAW` (1<<2), `BTF_F_ZERO` (1<<3).

### enum bpf_core_relo_kind

`FIELD_BYTE_OFFSET` (0), `FIELD_BYTE_SIZE` (1), `FIELD_EXISTS` (2), `FIELD_SIGNED` (3), `FIELD_LSHIFT_U64` (4), `FIELD_RSHIFT_U64` (5), `TYPE_ID_LOCAL` (6), `TYPE_ID_TARGET` (7), `TYPE_EXISTS` (8), `TYPE_SIZE` (9), `ENUMVAL_EXISTS` (10), `ENUMVAL_VALUE` (11), `TYPE_MATCHES` (12).

### Misc constants

`BPF_OBJ_NAME_LEN` (16), `BPF_TAG_SIZE` (8), `BPF_BUILD_ID_SIZE` (20), `BPF_LINE_INFO_LINE_NUM(line_col)`/`_COL(line_col)` macros, `BPF_STREAM_STDOUT` (1), `BPF_STREAM_STDERR` (2). Sentinel `__MAX_BPF_REG`.

### compatibility contract

REQ-1: `struct bpf_insn` is exactly **8 bytes** little-endian on disk: `code:u8, dst_reg:u4 ⊕ src_reg:u4 (1 byte total), off:s16, imm:s32`. Both `dst_reg` and `src_reg` occupy a single byte via 4-bit bitfields; the layout must match upstream regardless of endianness.

REQ-2: `union bpf_attr` is **8-byte aligned** (`__attribute__((aligned(8)))`) and consists of a tagged union whose discriminator is the `cmd` argument to `bpf(2)`. Each per-command anonymous struct is binary-stable. Per-struct sizes are growth-tolerant via the `info_len` / `attr_size` pattern: callers pass `sizeof(union bpf_attr)` they know, kernel validates that all trailing bytes are zero. The kernel reads only up to its own known size and ignores trailing bytes that are nonzero only if a corresponding `BPF_F_*` opt-in flag is set.

REQ-3: `BPF_MAP_CREATE` anonymous struct: `map_type, key_size, value_size, max_entries, map_flags, inner_map_fd, numa_node, map_name[BPF_OBJ_NAME_LEN], map_ifindex, btf_fd, btf_key_type_id, btf_value_type_id, btf_vmlinux_value_type_id, map_extra, value_type_btf_obj_fd, map_token_fd, excl_prog_hash:__aligned_u64, excl_prog_hash_size:u32`.

REQ-4: `BPF_PROG_LOAD` anonymous struct: `prog_type, insn_cnt, insns:__aligned_u64, license:__aligned_u64, log_level, log_size, log_buf:__aligned_u64, kern_version, prog_flags, prog_name[BPF_OBJ_NAME_LEN], prog_ifindex, expected_attach_type, prog_btf_fd, func_info_rec_size, func_info:__aligned_u64, func_info_cnt, line_info_rec_size, line_info:__aligned_u64, line_info_cnt, attach_btf_id, {attach_prog_fd | attach_btf_obj_fd}, core_relo_cnt, fd_array:__aligned_u64, core_relos:__aligned_u64, core_relo_rec_size, log_true_size, prog_token_fd, fd_array_cnt, signature:__aligned_u64, signature_size, keyring_id`.

REQ-5: `BPF_MAP_*_ELEM` / `BPF_MAP_FREEZE` struct: `map_fd, key:__aligned_u64, {value | next_key}:__aligned_u64, flags:u64`. `flags` carries `BPF_ANY`/`NOEXIST`/`EXIST`/`F_LOCK`/`F_CPU`/`F_ALL_CPUS`; upper bits used for CPU id when `F_CPU` set.

REQ-6: `BPF_MAP_*_BATCH` struct (`.batch`): `in_batch:u64, out_batch:u64, keys:u64, values:u64, count:u32, map_fd:u32, elem_flags:u64, flags:u64`.

REQ-7: `BPF_OBJ_PIN`/`OBJ_GET` struct: `pathname:u64, bpf_fd:u32, file_flags:u32, path_fd:s32`. `BPF_F_PATH_FD` opts into `path_fd` as `openat`-style anchor.

REQ-8: `BPF_PROG_ATTACH/DETACH` struct: `{target_fd | target_ifindex}, attach_bpf_fd, attach_type, attach_flags, replace_bpf_fd, {relative_fd | relative_id}, expected_revision:u64`.

REQ-9: `BPF_PROG_TEST_RUN` struct (`.test`): `prog_fd, retval, data_size_in, data_size_out, data_in:u64, data_out:u64, repeat, duration, ctx_size_in, ctx_size_out, ctx_in:u64, ctx_out:u64, flags, cpu, batch_size`.

REQ-10: `BPF_*_GET_*_ID` struct: `{start_id|prog_id|map_id|btf_id|link_id}, next_id, open_flags, fd_by_id_token_fd:s32`.

REQ-11: `BPF_OBJ_GET_INFO_BY_FD` struct (`.info`): `bpf_fd, info_len, info:u64` — `info_len` is in/out so kernel can report the size it wrote.

REQ-12: `BPF_PROG_QUERY` struct (`.query`): `{target_fd|target_ifindex}, attach_type, query_flags, attach_flags, prog_ids:u64, {prog_cnt|count}, prog_attach_flags:u64, link_ids:u64, link_attach_flags:u64, revision:u64`.

REQ-13: `BPF_RAW_TRACEPOINT_OPEN` struct (`.raw_tracepoint`): `name:u64, prog_fd, cookie:u64`.

REQ-14: `BPF_BTF_LOAD` struct: `btf:u64, btf_log_buf:u64, btf_size, btf_log_size, btf_log_level, btf_log_true_size, btf_flags, btf_token_fd:s32`.

REQ-15: `BPF_TASK_FD_QUERY` struct (`.task_fd_query`): `pid, fd, flags, buf_len, buf:u64, prog_id, fd_type, probe_offset:u64, probe_addr:u64`.

REQ-16: `BPF_LINK_CREATE` struct (`.link_create`): `{prog_fd|map_fd}, {target_fd|target_ifindex}, attach_type, flags`, plus a per-link-type union: `target_btf_id`; `{iter_info, iter_info_len}`; `perf_event { bpf_cookie }`; `kprobe_multi { flags, cnt, syms:u64, addrs:u64, cookies:u64 }`; `tracing { target_btf_id, cookie }`; `netfilter { pf, hooknum, priority, flags }`; `tcx { {relative_fd|relative_id}, expected_revision }`; `uprobe_multi { path:u64, offsets:u64, ref_ctr_offsets:u64, cookies:u64, cnt, flags, pid }`; `netkit { {relative_fd|relative_id}, expected_revision }`; `cgroup { {relative_fd|relative_id}, expected_revision }`.

REQ-17: `BPF_LINK_UPDATE` struct (`.link_update`): `link_fd, {new_prog_fd|new_map_fd}, flags, {old_prog_fd|old_map_fd}` (old fields only valid when `BPF_F_REPLACE` set).

REQ-18: `BPF_LINK_DETACH` struct: `link_fd:u32`.

REQ-19: `BPF_ENABLE_STATS` struct (`.enable_stats`): `type:u32` (one of `enum bpf_stats_type`).

REQ-20: `BPF_ITER_CREATE` struct (`.iter_create`): `link_fd:u32, flags:u32`.

REQ-21: `BPF_PROG_BIND_MAP` struct (`.prog_bind_map`): `prog_fd, map_fd, flags`.

REQ-22: `BPF_TOKEN_CREATE` struct (`.token_create`): `flags:u32, bpffs_fd:u32`.

REQ-23: `BPF_PROG_STREAM_READ_BY_FD` struct (`.prog_stream_read`): `stream_buf:u64, stream_buf_len, stream_id, prog_fd`.

REQ-24: `BPF_PROG_ASSOC_STRUCT_OPS` struct (`.prog_assoc_struct_ops`): `map_fd, prog_fd, flags`.

REQ-25: Info structs are append-only. `struct bpf_prog_info`, `struct bpf_map_info`, `struct bpf_btf_info`, `struct bpf_link_info`, `struct bpf_token_info` MUST be aligned to 8 bytes and exposed with `info_len` semantics — the kernel may write fewer bytes than `info_len` if it does not know all fields the caller declared, and never writes past `info_len`.

REQ-26: `struct bpf_prog_info` field set: `type, id, tag[BPF_TAG_SIZE], jited_prog_len, xlated_prog_len, jited_prog_insns:u64, xlated_prog_insns:u64, load_time:u64, created_by_uid, nr_map_ids, map_ids:u64, name[BPF_OBJ_NAME_LEN], ifindex, gpl_compatible:1, _:31, netns_dev:u64, netns_ino:u64, nr_jited_ksyms, nr_jited_func_lens, jited_ksyms:u64, jited_func_lens:u64, btf_id, func_info_rec_size, func_info:u64, nr_func_info, nr_line_info, line_info:u64, jited_line_info:u64, nr_jited_line_info, line_info_rec_size, jited_line_info_rec_size, nr_prog_tags, prog_tags:u64, run_time_ns:u64, run_cnt:u64, recursion_misses:u64, verified_insns, attach_btf_obj_id, attach_btf_id`.

REQ-27: `struct bpf_map_info` field set: `type, id, key_size, value_size, max_entries, map_flags, name[BPF_OBJ_NAME_LEN], ifindex, btf_vmlinux_value_type_id, netns_dev:u64, netns_ino:u64, btf_id, btf_key_type_id, btf_value_type_id, btf_vmlinux_id, map_extra:u64, hash:u64, hash_size`.

REQ-28: `struct bpf_btf_info` field set: `btf:u64, btf_size, id, name:u64, name_len, kernel_btf`.

REQ-29: `struct bpf_link_info` field set: `type, id, prog_id`, plus a per-type union (raw_tracepoint{tp_name,tp_name_len,cookie}; tracing{attach_type,target_obj_id,target_btf_id,cookie}; cgroup{cgroup_id,attach_type}; iter{target_name,target_name_len, [union map|cgroup|task]}; netns{netns_ino,attach_type}; xdp{ifindex}; struct_ops{map_id}; netfilter{pf,hooknum,priority,flags}; kprobe_multi{addrs,count,flags,missed,cookies}; uprobe_multi{path,offsets,ref_ctr_offsets,cookies,path_size,count,flags,pid}; perf_event{type, [union uprobe|kprobe|tracepoint|event]}; tcx{ifindex,attach_type}; netkit{ifindex,attach_type}; sockmap{map_id,attach_type}).

REQ-30: `struct bpf_lpm_trie_key` (deprecated): `prefixlen:u32, data[]:u8`. Replacements: `bpf_lpm_trie_key_hdr { prefixlen:u32 }` and `bpf_lpm_trie_key_u8 { union { hdr; prefixlen }, data[] }`.

REQ-31: `struct bpf_perf_event_value`: `counter:u64, enabled:u64, running:u64`.

REQ-32: `struct bpf_func_info`: `insn_off:u32, type_id:u32`. `struct bpf_line_info`: `insn_off, file_name_off, line_off, line_col` — with `BPF_LINE_INFO_LINE_NUM(c) = c >> 10` and `BPF_LINE_INFO_LINE_COL(c) = c & 0x3ff`.

REQ-33: Opaque kernel objects exposed in maps MUST match their upstream sizes and alignments: `bpf_spin_lock { u32 val }`, `bpf_timer { u64[2] }`, `bpf_task_work { u64 }`, `bpf_wq { u64[2] }`, `bpf_dynptr { u64[2] }`, `bpf_list_head { u64[2] }`, `bpf_list_node { u64[3] }`, `bpf_rb_root { u64[2] }`, `bpf_rb_node { u64[4] }`, `bpf_refcount { u32[1] }`, `bpf_iter_num { u64[1] }`. All are 8-byte aligned (refcount is 4-byte aligned).

REQ-34: `struct bpf_core_relo`: `insn_off:u32, type_id:u32, access_str_off:u32, kind:enum bpf_core_relo_kind`.

REQ-35: `struct bpf_insn_array_value`: `orig_off:u32, xlated_off:u32, jitted_off:u32, _:u32` — values for `BPF_MAP_TYPE_INSN_ARRAY`.

REQ-36: `struct bpf_lwt_encap` is NOT a standalone struct in this header — encap is selected by `enum bpf_lwt_encap_mode` and supplied by the `bpf_lwt_push_encap()` helper's blob argument; the wire encoding is per-mode (SEG6 / SEG6_INLINE / IP).

REQ-37: Reserved bits / pad fields in every per-cmd struct MUST be zero on input; future kernels may interpret them.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `insn_size_8` | INVARIANT | `sizeof(BpfInsn) == 8`. |
| `regs_packed` | INVARIANT | `BpfInsn::dst_reg()` returns low nibble; `src_reg()` returns high nibble; no overlap. |
| `attr_align_8` | INVARIANT | `align_of::<BpfAttr>() == 8`. |
| `prog_info_align_8` | INVARIANT | All `bpf_*_info` structs are 8-byte aligned. |
| `enum_values_match_upstream` | INVARIANT | Every `BpfCmd`, `BpfMapType`, `BpfProgType`, `BpfAttachType`, `BpfLinkType`, `BpfCoreReloKind`, `BpfStatsType` integer is identical to the C header. |
| `flag_bits_match_upstream` | INVARIANT | Every `BPF_F_*` constant matches its upstream bit position. |
| `name_buf_16` | INVARIANT | `map_name` / `prog_name` arrays are exactly 16 bytes. |
| `tag_8` | INVARIANT | `prog_info.tag` is exactly 8 bytes. |
| `build_id_20` | INVARIANT | `bpf_stack_build_id.build_id` is exactly 20 bytes. |
| `reserved_zero_on_input` | INVARIANT | All pad / reserved fields are zero on syscall input. |
| `current_cpu_alias` | INVARIANT | `BPF_F_CURRENT_CPU == BPF_F_INDEX_MASK`. |
| `ringbuf_busy_discard_disjoint` | INVARIANT | `BPF_RINGBUF_BUSY_BIT` and `_DISCARD_BIT` are distinct high bits. |

### Layer 2: TLA+

`uapi/bpf.tla`:
- Models the `bpf(2)` syscall as a tagged-union transition: `MapCreate(map_type, flags) -> map_fd`, `ProgLoad(prog_type, flags) -> prog_fd`, `ProgAttach(target, attach_type, flags) -> ok`, `LinkCreate(prog, target, attach_type) -> link_fd`, `MapLookup/Update/Delete`, `BtfLoad`, `IterCreate`, `TokenCreate`.
- Properties:
  - `safety_cmd_discriminator` — `bpf_attr` is interpreted only by the per-`cmd` schema; cross-cmd field reuse is forbidden.
  - `safety_info_len_growth_tolerant` — kernel never writes past `info_len`; userspace tolerates a smaller-than-expected reply (post-`info_len`).
  - `safety_token_fd_required_when_F_TOKEN_FD_set` — `BPF_F_TOKEN_FD` ⟹ `map_token_fd` / `prog_token_fd` / `btf_token_fd` is a valid open token fd.
  - `safety_attach_type_for_prog_type` — `(prog_type, attach_type)` pairs are limited to the upstream allow-list.
  - `safety_link_type_matches_prog_type` — `BPF_LINK_TYPE_*` matches the program kind being attached.
  - `safety_F_REPLACE_requires_old_fd` — `BPF_LINK_UPDATE` with `F_REPLACE` requires `old_prog_fd`/`old_map_fd`.
  - `safety_excl_prog_hash_only_with_hash_size` — `excl_prog_hash != 0 ⟹ excl_prog_hash_size > 0`.
  - `safety_reserved_zero` — every pad/reserved bit is zero on input.
  - `liveness_link_create_terminates` — every `LINK_CREATE` either succeeds and returns an fd or fails with a deterministic errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BpfInsn::encode/decode` round-trip equality | `BpfInsn` |
| `BpfAttr::map_create.map_name` is NUL-terminated within 16 bytes | `BpfAttrMapCreate` |
| `BpfAttr::prog_load.prog_name` is NUL-terminated within 16 bytes | `BpfAttrProgLoad` |
| `BpfProgInfo::flags_pad & 0xFFFFFFFE == 0` (only `gpl_compatible` bit defined) | `BpfProgInfo` |
| `BpfLpmTrieKeyU8::prefixlen <= 128` | `BpfLpmTrieKey*` |
| `BpfStackBuildId::status ∈ {EMPTY, VALID, IP}` | `BpfStackBuildId` |
| `BpfCoreRelo::kind ∈ enum bpf_core_relo_kind` | `BpfCoreRelo` |
| `BpfMapBatch::out_batch advances monotonically across calls` | `BpfAttrMapBatch` |

### Layer 4: Verus/Creusot functional

`Per-binary-equivalence with libbpf and bpftool`:
- Build a CO-RE program with `clang -target bpf` against the Rookery `include/uapi/linux/bpf.h` and a generated `vmlinux.h`; verifier load must succeed.
- `bindgen` parses the Linux header at `27a26ccfd528da725a999ea1e3102503c61eb655` and diffs every struct size and offset against the Rookery Rust types; CI fails on any drift.
- `bpftool prog/map/link list` and `info` output is byte-identical between Linux and Rookery for an identically-loaded program (mod `id`/`tag`/`load_time` which are nondeterministic).
- Round-trip property: every `BpfAttr` per-cmd struct serialized via `core::ptr::copy_nonoverlapping` to a buffer and deserialized back is bit-identical.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening once that overview exists.)

BPF UAPI reinforcement:

- **Per-`#[repr(C)]` exact layout** — defense against per-ABI drift across compilers; bitfield in `bpf_insn` modeled byte-explicit.
- **Per-`__aligned_u64` u64-alignment of every pointer slot** — defense against per-arch misalignment trap on 32-bit ABIs.
- **Per-`info_len` truncation discipline** — defense against per-future-kernel struct-growth and per-buffer-overrun on info copies.
- **Per-`copy_from_user(attr, size)` bounded by `min(size, sizeof(bpf_attr))`** — defense against per-attr-overread; trailing bytes that exceed kernel knowledge must be zero.
- **Per-cmd-specific argument shape** — defense against per-cross-cmd field aliasing; only the per-cmd struct fields are read.
- **Per-`prog_name`/`map_name` 16-byte fixed buffer NUL-terminated** — defense against per-name-overrun in `/proc/`/bpftool listings.
- **Per-`tag` fingerprint 8-byte fixed** — defense against per-tag-spoof; tag is a SHA-1 prefix of the program text.
- **Per-`BPF_F_*` flag mask validated** — defense against per-unknown-flag drift; unknown flag bits return -EINVAL.
- **Per-enum bound** — `prog_type < __MAX_BPF_PROG_TYPE`, `map_type < __MAX_BPF_MAP_TYPE`, etc; reject out-of-range.
- **Per-`expected_attach_type` matched against `attach_type`** — defense against per-attach-type confusion.
- **Per-`BPF_F_TOKEN_FD` capability delegation** — defense against per-unprivileged-bypass via stale token fds.
- **Per-instruction array bounded by `insn_cnt`** — defense against per-OOB instruction read.
- **Per-`bpf_attr.batch.count` capped** — defense against per-DOS via huge batch.
- **Per-BTF size bounded** — defense against per-DOS via huge BTF blob.
- **Per-signature verification** — defense against per-tampered prog when `keyring_id` is set; load gated on signature.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY** — every `bpf(2)` argument copy goes through hardened `copy_from_user`/`copy_to_user`; the `attr` blob is bounded by `sizeof(union bpf_attr)` known to the running kernel, and any caller-supplied size larger than that must have zero trailing bytes (verified inside the SMAP window). Pointer slots inside `bpf_attr` (`insns`, `log_buf`, `func_info`, `line_info`, `fd_array`, `core_relos`, `signature`, `keys`, `values`, `in_batch`, `out_batch`) are dereferenced only via `copy_from_user` with explicit element-size multiplication.
- **PaX USERCOPY whitelist** — `bpf_prog_info`, `bpf_map_info`, `bpf_btf_info`, `bpf_link_info`, `bpf_token_info` reside in dedicated slabs whose usercopy whitelists allow only the exact struct bytes to be copied out; the jited image buffer is a separate read-only mapping.
- **GRKERNSEC_HIDESYM** — `bpf_prog_info.jited_prog_insns`, `.jited_ksyms`, `.jited_line_info`, `.jited_func_lens` reveal kernel addresses; under HIDESYM these fields are zeroed or replaced with offsets-from-base for non-`CAP_SYS_ADMIN` callers. `BPF_OBJ_GET_INFO_BY_FD` respects HIDESYM by gating raw kernel pointers.
- **GRKERNSEC_PROC restrictions** — `/proc/sys/kernel/unprivileged_bpf_disabled`, `kernel.bpf_stats_enabled` and `/sys/kernel/btf/*` are restricted under proc-hide; `bpf_prog_id`s and `bpf_link_id`s are scoped per user-namespace under grsec.
- **PAX_RANDKSTACK** — `bpf(2)` is a syscall entry and benefits from per-syscall kernel-stack randomization; verifier state (which is large) is heap-allocated rather than stack-anchored so RANDKSTACK does not blow the stack.
- **PAX_REFCOUNT** — every userland-provided count (`insn_cnt`, `func_info_cnt`, `line_info_cnt`, `core_relo_cnt`, `fd_array_cnt`, `nr_map_ids`, `count` in batch, `cnt` in kprobe_multi/uprobe_multi/netfilter) feeds bounded allocations via overflow-checked arithmetic (`refcount_t`/`size_mul_overflow`).
- **GRKERNSEC_NO_SIMULT_CONNECT** — applies indirectly via BPF `SOCK_OPS`, `SK_LOOKUP`, `SK_REUSEPORT`, `CGROUP_SOCK_ADDR` programs; rate limits and per-task lockouts in the network stack still apply when BPF defers a verdict.
- **PAX_KERNEXEC W^X** — the BPF JIT is the canonical W^X case: image is built in a writable scratch region, frozen with `set_memory_ro` + `set_memory_x` before reachable from any dispatcher, and removed only via `set_memory_rw` after all RCU graces ensure no executor holds the page. `bpf_jit_harden` ≥ 1 is enforced.
- **GRKERNSEC_BPF_HARDEN** — `BPF_PROG_LOAD` requires `CAP_BPF` (or the legacy `CAP_SYS_ADMIN`) for any prog type; sleepable programs and JIT'd LSM hooks require additionally `CAP_PERFMON` or `CAP_NET_ADMIN` per the standard split. The `unprivileged_bpf_disabled` sysctl defaults to `2` (disabled, no override) under grsec.
- **CAP_BPF / CAP_NET_ADMIN / CAP_PERFMON delta** — capability set is split from `CAP_SYS_ADMIN`: `CAP_BPF` for prog/map create and load; `CAP_NET_ADMIN` for `SCHED_CLS`, `SCHED_ACT`, `XDP`, `LWT_*`, `SK_*`, `CGROUP_SKB`, `FLOW_DISSECTOR`, `NETFILTER`, `NETKIT`, `TCX`; `CAP_PERFMON` for `KPROBE`, `TRACEPOINT`, `PERF_EVENT`, `RAW_TRACEPOINT*`, `TRACING`. `BPF_TOKEN_CREATE` lets a privileged daemon delegate a capability subset to unprivileged callers via a bpffs token.
- **PAX_MEMORY_SANITIZE** — `bpf_*_info` structs are zero-initialized before being filled, and the unused region between `info_len` and `sizeof(struct ...)` is zero on output to prevent kernel-memory leak.
- **GRKERNSEC_DMESG-style logging** — verifier rejection messages, JIT failure messages, and signature failures are rate-limited and scrubbed of pointers/PIDs before reaching `dmesg`; the BPF verifier log (`log_buf`) is per-process and not globally readable.
- **PAX_USERCOPY size-cap on programs** — `insn_cnt` is capped by `BPF_COMPLEXITY_LIMIT_INSNS`; `signature_size`, `func_info_cnt * rec_size`, `line_info_cnt * rec_size`, `core_relo_cnt * rec_size`, and `log_size` are each bounded before any allocation.
- **GRKERNSEC_KMEM-style** — the JIT image, the verifier state, and the BTF blob each live in their own slabs/vmalloc regions, never overlapping with general kmalloc; the verifier's scratch is `kfree`d, and the JIT region is unmapped on `BPF_PROG_RELEASE`.
- **PaX SMAP/SMEP** — every BPF code path that touches user memory is bracketed by `stac`/`clac`; SMEP forbids the user pages used for `attr` to be executed as kernel code. The BPF JIT image is in kernel pages with X but not W (W^X), and is never user-accessible.
- **PaX kernel-pointer scrubbing** — `bpf_link_info.netns_ino`, `.cgroup_id`, `.target_obj_id` are stable namespaced identifiers, not pointers. Whenever a field could otherwise leak a kernel pointer (e.g. `.jited_func_lens`), `CAP_SYS_ADMIN` is required and HIDESYM may still override.
- **GRKERNSEC_HARDEN_PTRACE-style** — `BPF_TASK_FD_QUERY` honors ptrace permissions on the queried PID; a non-`CAP_SYS_PTRACE` caller cannot enumerate another process's BPF fds even with `CAP_BPF`.

