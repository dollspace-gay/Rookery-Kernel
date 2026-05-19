# Tier-3: kernel/bpf/btf.c — BTF (BPF Type Format) type metadata

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/btf.c (~9810 lines)
  - include/linux/btf.h
  - include/uapi/linux/btf.h
  - tools/lib/bpf/btf.h (libbpf consumer)
-->

## Summary

BTF (BPF Type Format) is a compact debug-info format embedded in BPF objects + kernel vmlinux that exposes per-type metadata for: BPF program verification (CO-RE relocation, kfunc signatures, helper proto types), bpf_iter / kfunc / struct_ops type-checking, observability tools (bpftrace, libbpf-tools). Per-BTF blob: `struct btf_header` + type section (BTF_KIND_INT/PTR/ARRAY/STRUCT/UNION/ENUM/ENUM64/FWD/TYPEDEF/VOLATILE/CONST/RESTRICT/FUNC/FUNC_PROTO/VAR/DATASEC/FLOAT/DECL_TAG/TYPE_TAG) + string section. Per-vmlinux BTF embedded in `.BTF` ELF section (built via pahole). Per-module BTF in `.BTF` section. Critical for: CO-RE (Compile-Once-Run-Everywhere), kfuncs, fentry/fexit, struct_ops.

This Tier-3 covers `kernel/bpf/btf.c` (~9810 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btf_header` | per-BTF magic+offsets | UAPI |
| `struct btf` | per-instance | `Btf` |
| `btf_parse()` | per-BTF blob → `struct btf` | `Btf::parse` |
| `btf_parse_vmlinux()` | vmlinux BTF | `Btf::parse_vmlinux` |
| `btf_parse_module()` | per-mod BTF | `Btf::parse_module` |
| `btf_check_all_metas()` | per-validate type-section | `Btf::check_all_metas` |
| `btf_resolve()` | per-resolve fwd refs + sizing | `Btf::resolve` |
| `btf_type_id_size()` | per-type size | `Btf::type_id_size` |
| `btf_type_id_resolve()` | per-skip-modifiers | `Btf::type_id_resolve` |
| `btf_resolve_size()` | per-recursive-size | `Btf::resolve_size` |
| `btf_get_by_fd()` | per-FD lookup | `Btf::get_by_fd` |
| `btf_new_fd()` | per-userland install | `Btf::new_fd` |
| `btf_find_by_name_kind()` | per-name+kind lookup | `Btf::find_by_name_kind` |
| `btf_find_func_proto()` | per-FUNC_PROTO | `Btf::find_func_proto` |
| `btf_show()` | per-record-show (debug-print) | `Btf::show` |
| `btf_struct_access()` | per-mem-access type-check | `Btf::struct_access` |
| `btf_check_kfunc_arg_match()` | per-kfunc arg-check | `Btf::check_kfunc_arg_match` |
| `btf_resolve_helper_id()` | per-helper proto bind | `Btf::resolve_helper_id` |
| `btf_id_set_contains()` | per-allowlist check | `Btf::id_set_contains` |
| `btf_check_subprog_arg_match()` | per-static-call check | `Btf::check_subprog_arg_match` |

## Compatibility contract

REQ-1: struct btf_header:
- magic (u16): 0xeB9F.
- version (u8): 1.
- flags (u8): 0.
- hdr_len (u32): header size.
- type_off / type_len (u32, u32): type section offset+len.
- str_off / str_len (u32, u32): string section.

REQ-2: struct btf_type:
- name_off: u32 → string-section offset.
- info: u32: encoded as (kind_flag << 31 | kind << 24 | vlen).
- size_or_type: u32: size (composite) or type-id (referenced).

REQ-3: BTF_KIND_*:
- INT (1): integer (size in bytes; encoding signed/char/bool/...).
- PTR (2): pointer-to-type.
- ARRAY (3): vlen=0; followed by btf_array { type, index_type, nelems }.
- STRUCT (4) / UNION (5): vlen members; followed by btf_member [vlen]{ name_off, type, offset }.
- ENUM (6) / ENUM64 (19): vlen vals.
- FWD (7): forward decl.
- TYPEDEF (8): typedef-of-type.
- VOLATILE (9) / CONST (10) / RESTRICT (11): qualifier.
- FUNC (12): function (size_or_type → FUNC_PROTO).
- FUNC_PROTO (13): vlen params; followed by btf_param.
- VAR (14): global var.
- DATASEC (15): vlen sec_info.
- FLOAT (16): floating-point.
- DECL_TAG (17): per-decl annotation.
- TYPE_TAG (18): per-type annotation.

REQ-4: btf_parse(btf_data, btf_size):
- Validate btf_header.
- Validate type-section: per-type btf_check_meta.
- Validate string-section: NUL-terminated.
- Resolve fwd refs.
- Compute per-type sizes (resolve_size).
- Return struct btf.

REQ-5: btf_check_all_metas:
- For each type: check kind valid, name_off ≤ str_len, vlen sane, type ID < types_count.

REQ-6: btf_resolve:
- Compute size for STRUCT/UNION/ARRAY recursively.
- Detect cycles (-ELOOP).
- Cache per-type size in btf->resolved_sizes.

REQ-7: btf_type_id_resolve(id):
- Skip modifiers (CONST/VOLATILE/RESTRICT/TYPEDEF/TYPE_TAG).
- Return underlying type id.

REQ-8: btf_struct_access(env, t, off, size, atype, next_btf_id):
- For pointer-deref/access: walk struct members at `off`; if member found:
  - btf_type_skip_modifiers.
  - if pointer: PTR_TO_BTF_ID with new id.
  - if scalar: SCALAR_VALUE.
- Per-PTR_TRUSTED / PTR_UNTRUSTED / MEM_RDONLY rules.

REQ-9: btf_check_kfunc_arg_match(env, btf, func_id, regs):
- Look up FUNC_PROTO via btf->type[FUNC.size_or_type].
- For each param: check reg type matches param type:
  - struct ptr / scalar / DYNPTR / KPTR / kfunc-arg-tags.
- Per-KF_ACQUIRE / KF_RELEASE / KF_KPTR_GET / KF_TRUSTED_ARGS / KF_SLEEPABLE flags.

REQ-10: btf_id_set_contains:
- Per-allowlist sorted u32-array binary-search.

REQ-11: bpf(BPF_BTF_LOAD):
- Userland passes BTF blob.
- Kernel: btf_parse, install in id_map, return FD.
- Used by libbpf for prog-load with BTF.

REQ-12: vmlinux BTF:
- Built by pahole during kernel build.
- Embedded in `.BTF` ELF section.
- Loaded at init via __start_BTF / __stop_BTF.

REQ-13: Module BTF:
- Per-loadable module: `.BTF` section.
- module_btf_init at module-load.

REQ-14: BTF deduplication (libbpf-side):
- Per-link merging: not in kernel (kernel sees pre-deduped blob).

REQ-15: CO-RE (libbpf):
- Per-prog: BTF-relocations encoded in BPF bytecode.
- Verifier validates per-relocated bytecode.

REQ-16: bpf_iter / kfunc / struct_ops registration:
- Per-subsystem registers BTF-id-set + handler.

REQ-17: btf_show:
- For tracing/debug: pretty-print value with type info.

REQ-18: BTF_KIND_DECL_TAG:
- Per-attribute (e.g., __percpu, __user, __rcu).
- Verifier interprets per-relevant tag.

## Acceptance Criteria

- [ ] AC-1: bpf(BPF_BTF_LOAD, blob): valid BTF returns FD.
- [ ] AC-2: malformed magic: -EINVAL.
- [ ] AC-3: type-id out-of-range: -EINVAL.
- [ ] AC-4: cyclic type-resolve: -ELOOP.
- [ ] AC-5: btf_find_by_name_kind("task_struct", STRUCT): returns id.
- [ ] AC-6: btf_struct_access at offset: returns BTF_TYPE_SAFE.
- [ ] AC-7: btf_check_kfunc_arg_match for valid args: 0; mismatched: -EINVAL.
- [ ] AC-8: vmlinux BTF loaded at boot.
- [ ] AC-9: Module BTF loaded at module load.
- [ ] AC-10: BTF_KIND_FUNC_PROTO with KF_TRUSTED_ARGS: enforced.
- [ ] AC-11: KPTR field in struct: per-bpf_kptr_xchg semantics.

## Architecture

```
struct Btf {
  data: &[u8],
  data_size: u32,
  hdr: BtfHeader,
  types: Vec<*BtfType>,           // points into data
  resolved_sizes: Vec<u32>,
  resolved_ids: Vec<u32>,
  base_btf: Option<&Btf>,         // for split-BTF (mod inherits vmlinux)
  start_id: u32,                  // first type id (split-BTF)
  start_str_off: u32,
  refcount: AtomicU64,
  id: u32,
  kernel_btf: bool,               // vmlinux ?
  base_btf_index: u32,
}
```

`Btf::parse(data, size, base_btf)`:
1. /* Validate header */
2. if size < sizeof(BtfHeader): return -EINVAL.
3. hdr = data[0..hdr_len].
4. if hdr.magic != 0xeB9F: -EINVAL.
5. /* Parse type-section */
6. type_data = data[hdr_off + hdr.type_off ..].
7. types = parse_types(type_data, hdr.type_len).
8. /* Validate */
9. Btf::check_all_metas(&self).
10. /* Resolve sizes */
11. Btf::resolve(&self).
12. return Ok(Btf { data, hdr, types, ... }).

`Btf::check_all_metas()`:
1. for (id, t) in self.types.iter().enumerate():
   - if t.kind not in valid_kinds: -EINVAL.
   - if t.name_off > self.hdr.str_len: -EINVAL.
   - kind-specific check (vlen, members fit in section, ...).

`Btf::resolve()`:
1. for id in 0..self.types.len():
   - sz = Btf::resolve_size(self, id, &mut visited).
   - if sz == ELOOP: return -ELOOP.
   - self.resolved_sizes[id] = sz.

`Btf::resolve_size(id, visited)`:
1. if visited[id]: -ELOOP.
2. visited[id] = true.
3. t = self.types[id].
4. switch t.kind:
   - INT / FLOAT / ENUM / ENUM64 / PTR: return t.size_or_type (or arch-ptr-size).
   - ARRAY: arr = t.next; nelems * resolve_size(arr.type).
   - STRUCT / UNION: t.size_or_type (size).
   - TYPEDEF / CONST / VOLATILE / RESTRICT / TYPE_TAG: resolve_size(t.size_or_type).
   - FUNC / FUNC_PROTO: 0 (no size).
5. visited[id] = false.

`Btf::find_by_name_kind(name, kind) -> Option<u32>`:
1. for (id, t) in iter:
   - if t.kind == kind ∧ str_at(t.name_off) == name:
     - return id.
2. return None.

`Btf::struct_access(t, off, size, atype) -> Result<TypeInfo>`:
1. /* Find member at offset */
2. members = t.members().
3. for m in members:
   - if m.offset/8 ≤ off < (m.offset/8 + member_size):
     - if member is pointer: return PTR_TO_BTF_ID(new_id).
     - if scalar: return SCALAR_VALUE.
     - if struct: recurse with off - m.offset/8.
4. return -EACCES.

`Btf::check_kfunc_arg_match(func_id, regs)`:
1. func = self.types[func_id].
2. proto = self.types[func.size_or_type].
3. for (i, param) in proto.params().enumerate():
   - reg = &regs[i + 1].
   - match param-type:
     - PTR with KF_TRUSTED_ARGS: reg.type == PTR_TRUSTED.
     - DYNPTR: reg.type == PTR_TO_DYNPTR ∧ valid.
     - KPTR: per-bpf_kptr_xchg.
     - scalar: reg.type == SCALAR_VALUE.
4. return 0.

`Btf::id_set_contains(set, id) -> bool`:
1. /* Binary search sorted u32 array */
2. lo = 0; hi = set.cnt - 1.
3. while lo ≤ hi:
   - mid = (lo + hi) / 2.
   - if set.ids[mid] == id: return true.
   - if set.ids[mid] < id: lo = mid + 1.
   - else: hi = mid - 1.
4. return false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `header_magic_valid` | INVARIANT | per-Btf::parse: hdr.magic == 0xeB9F. |
| `type_id_within_range` | INVARIANT | per-type-ref: t.size_or_type < self.types.len(). |
| `string_off_within_str_len` | INVARIANT | per-type t: t.name_off < hdr.str_len. |
| `resolve_size_acyclic` | INVARIANT | per-resolve: returns size or -ELOOP. |
| `id_set_sorted` | INVARIANT | per-allowlist: ids sorted ascending; binary-search applies. |
| `kfunc_arg_count_match` | INVARIANT | per-check_kfunc_arg_match: regs.len ≥ proto.params.len + 1. |

### Layer 2: TLA+

`kernel/bpf/btf.tla`:
- Per-BTF parse + per-type resolve + per-kfunc-arg-check.
- Properties:
  - `safety_no_oor_type_ref` — per-validation: every type-ref id < types_count.
  - `safety_resolve_terminates` — per-resolve: cycle detection.
  - `safety_kfunc_arg_strict_typed` — per-check: param-type ⇒ reg-type compatibility.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Btf::parse` post: types validated; resolved_sizes filled | `Btf::parse` |
| `Btf::resolve_size` post: returns size ∨ -ELOOP | `Btf::resolve_size` |
| `Btf::find_by_name_kind` post: returned id has matching kind+name | `Btf::find_by_name_kind` |
| `Btf::struct_access` post: per-offset access type-resolved | `Btf::struct_access` |
| `Btf::check_kfunc_arg_match` post: per-arg type-check passes | `Btf::check_kfunc_arg_match` |
| `Btf::id_set_contains` post: returns membership | `Btf::id_set_contains` |

### Layer 4: Verus/Creusot functional

`Per-BTF blob → struct btf with type-section validated, fwd-ref-resolved, sizes-cached → consumer-API for verifier (CO-RE, kfunc, struct_access, helper-proto)` semantic equivalence: per-Documentation/bpf/btf.rst.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

BTF reinforcement:

- **Per-magic + version strict** — defense against per-non-BTF-blob.
- **Per-type-ref bounds-checked** — defense against per-OOR type access.
- **Per-string-off bounds-checked** — defense against per-OOR string read.
- **Per-cycle detection ELOOP** — defense against per-DOS cyclic-resolve.
- **Per-blob size-capped (BTF_MAX_SIZE)** — defense against per-DOS large-BTF.
- **Per-kfunc allowlist binary-search sorted** — defense against per-id-spoof.
- **Per-KF_TRUSTED_ARGS strict reg-type** — defense against per-untrusted kfunc-deref.
- **Per-CAP_BPF for BPF_BTF_LOAD** — defense against unprivileged BTF-load.
- **Per-vmlinux BTF kernel_btf flag set** — defense against per-userland-BTF impersonation.
- **Per-module BTF lifecycle tied to module ref** — defense against per-stale-BTF after rmmod.
- **Per-resolve_size memoized** — defense against per-recompute-DOS.
- **Per-DECL_TAG / TYPE_TAG semantic-validation** — defense against per-typo annotation acceptance.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check `copy_from_user(btf_data, ..., size)` against `BTF_MAX_SIZE` and the slab whitelist; reject any blob whose header `hdr_len`/`type_off`/`str_off` arithmetic crosses the allocation.
- **PAX_KERNEXEC** — keep `btf_kind_ops[]`, `btf_func_proto_*`, the kfunc id-set tables, and `vmlinux_btf` pointer in `__ro_after_init`.
- **PAX_RANDKSTACK** — re-randomize stack on `btf_parse`/`btf_validate`/`btf_resolve` to defeat disclosure of the staged `struct btf_verifier_env`.
- **PAX_REFCOUNT** — saturating refcount on `struct btf`, per-module BTF, and every `btf_id_set8` table entry; wraparound MUST panic.
- **PAX_MEMORY_SANITIZE** — zero `btf->data`, `btf->types`, and `btf->resolved_sizes`/`resolved_ids` on free; never let prior BTF blob bytes leak into a recycled load.
- **PAX_UDEREF** — strict user/kernel separation while parsing the user-supplied BTF blob; the kernel MUST NOT deref a smuggled kernel pointer from `btf_header`.
- **PAX_RAP / kCFI** — type-check every indirect call resolved by BTF (kfunc dispatch, `bpf_struct_ops` vtable, `fmod_ret` trampolines); a BTF-resolved indirect call MUST match the BTF-declared `FUNC_PROTO`.
- **GRKERNSEC_HIDESYM** — hide `btf_parse_*`, `btf_resolve_*`, and kfunc symbols from non-root kallsyms; BTF symbol disclosure is a kfunc-id-spoof oracle.
- **GRKERNSEC_DMESG** — restrict dmesg so BTF verifier spew (which echoes attacker-supplied type names and offsets) is not harvestable.
- **BTF blob PAX_USERCOPY** — the `btf->data` staging buffer MUST be allocated from a usercopy-whitelisted cache because it is later re-read for type-id resolution and `btf_show_*` snprintf paths.
- **`btf_validate` signature** — under grsec, kernel BTF (`vmlinux_btf`, module BTF) MUST carry an integrity signature checked at load; an unsigned module BTF triggers `audit_panic`. User BTF (`BPF_BTF_LOAD`) MUST set `kernel_btf == false` and is never trusted for `bpf_struct_ops` or trampoline resolution.
- **kfunc allowlist** — every kfunc reachable from a BPF program MUST appear in a per-subsystem `BTF_ID_SET8` allowlist with explicit `KF_*` flags; `KF_TRUSTED_ARGS` is mandatory for any kfunc taking a kernel pointer, and PAX_RAP must verify the call-site type before dispatch.
- **Rationale** — BTF is the type-resolution oracle for the entire BPF subsystem (verifier, kfuncs, tracing, struct_ops); a tampered or type-confused BTF blob is a direct kernel R/W primitive. PAX_USERCOPY + PAX_RAP + btf_validate + kfunc allowlist close the historical class of BTF-driven type confusion (e.g., the recurring `bpf_struct_ops` and CO-RE relocation CVEs).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/bpf/verifier.c (covered in `verifier.md` Tier-3)
- kernel/bpf/core.c (covered in `bpf-core.md` Tier-3)
- kernel/bpf/helpers.c (covered separately if expanded)
- libbpf BTF dedup (userland)
- pahole BTF generation (userland build-time)
- Implementation code
