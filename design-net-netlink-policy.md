---
title: "Tier-3: lib/nlattr.c — Netlink-attribute (NLA) policy validation"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`lib/nlattr.c` implements the netlink-attribute (NLA / TLV) framework used by AF_NETLINK + genetlink. Per-attr type + length + value layout (TLV) walked by `nla_for_each_attr`, validated against per-msg policy table (`struct nla_policy`), with rich type-encodings (NLA_U8/16/32/64, S8/16/32/64, STRING, BINARY, FLAG, NESTED, NESTED_ARRAY, BITFIELD32, REJECT). Per-policy: range/min-len/max-len + nested sub-policy. Per-msg `nlmsg_parse` returns per-attr index → ptr table for handlers. Critical for: every netlink-based control plane (rtnetlink/genetlink/nfnetlink/etc.) — single source of truth for input validation.

This Tier-3 covers `nlattr.c` (~1169 lines) — the attribute-policy validation library.

### Acceptance Criteria

- [ ] AC-1: nla_parse with valid msg + policy: tb[] populated.
- [ ] AC-2: Per-attr type > maxtype: -EINVAL.
- [ ] AC-3: NLA_U32 with len < 4: -EINVAL.
- [ ] AC-4: NLA_RANGE with value out-of-range: -ERANGE; extack populated.
- [ ] AC-5: NLA_STRING without NUL: -EINVAL.
- [ ] AC-6: NLA_STRING > max-len: -EINVAL.
- [ ] AC-7: NLA_NESTED with sub-policy: nested validated.
- [ ] AC-8: NLA_REJECT attr present: -EINVAL with reject_message.
- [ ] AC-9: NL_VALIDATE_STRICT + unknown-type: -EINVAL.
- [ ] AC-10: NL_VALIDATE_LIBERAL + unknown-type: skipped silently.
- [ ] AC-11: nla_put_u32 + nla_get_u32 round-trip preserves value.
- [ ] AC-12: nla_put_string + nla_get_string preserves NUL-terminated string.
- [ ] AC-13: NLA_F_NET_BYTEORDER flag: nla_get_be32 converts correctly.

### Architecture

Per-attr header:

```
#[repr(C)]
struct Nla {
  nla_len: u16,                                  // including header + data + padding
  nla_type: u16,                                 // type | NLA_F_NESTED | NLA_F_NET_BYTEORDER
}
const NLA_ALIGNTO: usize = 4;
const NLA_HDRLEN: usize = round_up(size_of::<Nla>(), NLA_ALIGNTO); // 4 bytes
```

Per-attr policy:

```
struct NlaPolicy {
  ty: NlaType,                                   // NLA_U8 / U16 / ...
  validation_type: NlaValidate,
  len: u16,                                      // for BINARY/STRING max-len
  min: i64,                                      // for RANGE
  max: i64,
  validate: Option<fn(&Nla, &mut NlExtAck) -> Result<()>>,
  reject_message: Option<&'static str>,
  bitfield32_valid: u32,                         // for NLA_BITFIELD32
  mask: u64,                                     // for NLA_VALIDATE_MASK
  nested: Option<&'static [NlaPolicy]>,          // for NLA_NESTED sub-policy
}
```

`Nla::parse(tb, maxtype, head, len, policy, extack, validate_flags)`:
1. tb.fill(None).
2. For each attr in nla_for_each_attr(head, len):
   - Validate attr.nla_len ≤ remaining.
   - type = attr.nla_type & NLA_TYPE_MASK.
   - If type > maxtype: liberal → continue; strict → -EINVAL + extack.
   - If policy[type] != UNSPEC: __nla_validate(attr, policy[type], extack)?
   - tb[type] = Some(attr).
3. Return Ok.

`Nla::validate(attr, policy, extack)`:
1. Match policy.ty:
   - NLA_U8: nla_len ≥ 1; per-VALIDATE_RANGE check value.
   - NLA_U16: nla_len ≥ 2; ...
   - NLA_U32: nla_len ≥ 4; ...
   - NLA_U64: nla_len ≥ 8; ...
   - NLA_STRING: per-data, find NUL ≤ max-len.
   - NLA_NUL_STRING: data NUL-terminated; min-len.
   - NLA_BINARY: data-len ∈ [min, max].
   - NLA_FLAG: nla_len == 0.
   - NLA_NESTED: recurse Nla::parse(sub_tb, sub_maxtype, attr.data, attr.len, policy.nested, extack).
   - NLA_NESTED_ARRAY: walk array of nested entries.
   - NLA_BITFIELD32: nla_len == 8; validate (value & ~bitfield32_valid) == 0.
   - NLA_REJECT: populate extack.set_err_msg(reject_message); return -EINVAL.
2. Per-validate_type:
   - RANGE: read typed value; check ∈ [min, max].
   - FUNCTION: invoke policy.validate(attr, extack).
   - MASK: validate (value & ~policy.mask) == 0.

`Nla::get_u32(attr) → u32`:
1. Read 4 bytes at attr.data; return as native-endian.

`Nla::put_u32(skb, type, value)`:
1. Reserve sizeof(Nla) + 4 + padding in skb.
2. Set Nla.nla_type = type; nla_len = 8.
3. Write u32 at data offset.

`Nla::nest_start(skb, type) → &Nla`:
1. Reserve sizeof(Nla); set type | NLA_F_NESTED; len = 0 (placeholder).
2. Return ptr to header for later finalization.

`Nla::nest_end(skb, head)`:
1. head.nla_len = (skb.tail - head_offset) (rounded up to NLA_ALIGNTO).

### Out of Scope

- AF_NETLINK core (covered in `af_netlink.md` Tier-3)
- Generic netlink (covered in `genetlink.md` Tier-3)
- Per-family policy contents (rtnetlink/ethtool/etc.; covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nlattr` | per-TLV header (u16 type, u16 len) | `Nla` |
| `struct nla_policy` | per-attr validation rule | `NlaPolicy` |
| `enum NLA_*` | per-type constant (U8, U16, ..., STRING, NESTED, ...) | `NlaType` |
| `nla_parse()` / `nla_parse_nested()` | per-msg attr walker + policy validator | `Nla::parse` |
| `nlmsg_parse()` | nlmsg + attrs parse | `Nlmsg::parse` |
| `__nla_validate()` | per-attr policy validator | `Nla::validate` |
| `nla_get_u8/u16/u32/u64()` | per-attr typed read | `Nla::get_*` |
| `nla_put_u8/u16/u32/u64()` | per-attr typed write | `Nla::put_*` |
| `nla_get_string()` | per-attr string read | `Nla::get_string` |
| `nla_put_string()` | per-attr string write | `Nla::put_string` |
| `nla_nest_start()` / `nla_nest_end()` | nested attr framing | `Nla::nest_*` |
| `NLA_F_NESTED` | per-type flag bit | shared |
| `NL_SET_ERR_MSG()` | per-extack error str | `NlExtAck::set_err_msg` |
| `NL_SET_BAD_ATTR()` | per-extack error attr | `NlExtAck::set_bad_attr` |

### compatibility contract

REQ-1: Per-NLA wire format:
- `struct nlattr { u16 nla_len; u16 nla_type; }` followed by data.
- nla_len includes header + data; aligned to 4-byte boundary (`NLA_ALIGN`).
- nla_type bits[14] = NLA_F_NESTED; bits[15] = NLA_F_NET_BYTEORDER.

REQ-2: Per-NLA-policy table:
- `nla_policy[type]` per-attribute slot (≤ MAX_TYPE).
- Per-slot fields: `type` (NLA_*), `validation_type`, `len`, `min`, `max`, `validate` (callback), `reject_message`.

REQ-3: Per-NLA type values:
- `NLA_UNSPEC` (0): no validation (pass-through).
- `NLA_U8` (1) / `NLA_U16` / `NLA_U32` / `NLA_U64`.
- `NLA_STRING` (5): NUL-terminated, max-len enforced.
- `NLA_FLAG` (6): zero-length attr; presence == true.
- `NLA_MSECS` (7): u64 milliseconds.
- `NLA_NESTED` (8): nested attribute set.
- `NLA_NESTED_ARRAY` (9): sequence of nested entries.
- `NLA_NUL_STRING` (10): NUL-terminated min-len.
- `NLA_BINARY` (11): raw bytes, min/max len.
- `NLA_S8` (12) / `NLA_S16` / `NLA_S32` / `NLA_S64`.
- `NLA_BITFIELD32` (16): u32 mask + u32 value.
- `NLA_REJECT` (17): always fails (deprecated attrs).

REQ-4: Per-NLA-validation-type:
- `NLA_VALIDATE_NONE`.
- `NLA_VALIDATE_RANGE` (min/max u64).
- `NLA_VALIDATE_RANGE_WARN_TOO_LONG`.
- `NLA_VALIDATE_MIN_LEN`.
- `NLA_VALIDATE_MAX_LEN`.
- `NLA_VALIDATE_RANGE_PTR`.
- `NLA_VALIDATE_FUNCTION` (callback).
- `NLA_VALIDATE_MASK` (per-bit allowed).

REQ-5: nla_parse() flow:
- Per-msg starts at given offset; iterates via `nla_for_each_attr`.
- Per-attr: validate type ≤ maxtype.
- Per-attr: __nla_validate per-policy.
- Per-attr: tb[type] = attr; per-msg returns tb[].

REQ-6: __nla_validate() per-attr:
- Per-NLA_*: enforce minimum-payload-len.
- Per-VALIDATE_RANGE: read u64; check min ≤ value ≤ max.
- Per-NLA_STRING / NUL_STRING: enforce NUL termination + max-len.
- Per-NLA_NESTED: recurse nla_validate with sub-policy.
- Per-VALIDATE_FUNCTION: invoke callback.
- Per-fail: populate extack with NL_SET_BAD_ATTR + NL_SET_ERR_MSG_ATTR.

REQ-7: Per-validate-strict mode:
- `NL_VALIDATE_LIBERAL` (legacy): unknown types ignored.
- `NL_VALIDATE_STRICT`: unknown types → -EINVAL.
- `NL_VALIDATE_DUMP_STRICT`: per-dump strict.

REQ-8: Per-extack:
- `struct netlink_ext_ack`: cookie + bad_attr-offset + err-msg-string.
- Per-extack populates ETXACKDATA in NLMSG_ERROR reply.

REQ-9: Per-NESTED parse:
- `nla_parse_nested(tb, maxtype, nla, policy, extack)`:
  - Walks within nla.data..nla.data+nla.len.
  - Per-NESTED has its own MAX_TYPE + policy.

REQ-10: Per-attr typed-read helpers:
- nla_get_u8/16/32/64: bounds-check len ≥ sizeof(T); read native-endian.
- nla_get_be32/16: convert from network-byte-order.
- nla_get_string: validate NUL-term; copy up to max-len.

REQ-11: Per-attr typed-put helpers:
- `nla_put` + size + type → reserve in skb; emit nlattr header + payload + padding.
- `nla_put_string`: include NUL.

REQ-12: Per-FLAG attr:
- nla_len == 0 (only header).
- Presence in tb[] indicates flag is set.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nla_len_4byte_aligned` | INVARIANT | per-emitted nlattr nla_len padded to NLA_ALIGNTO. |
| `parse_within_buffer` | INVARIANT | per-walked attr ptr + nla_len ≤ buffer-end. |
| `tb_index_lt_maxtype_plus_1` | INVARIANT | per-tb[idx] write: idx ≤ maxtype. |
| `string_nul_terminated` | INVARIANT | per-NLA_STRING validate ⟹ NUL within data. |
| `range_value_in_bounds` | INVARIANT | per-NLA_VALIDATE_RANGE pass ⟹ value ∈ [min, max]. |
| `nested_recursion_bounded` | INVARIANT | per-NLA_NESTED depth ≤ NLA_MAX_DEPTH (32). |

### Layer 2: TLA+

`net/netlink/nla_policy.tla`:
- Per-msg attr walker + per-attr policy validation.
- Properties:
  - `safety_no_oob_read` — per-walked attr stays within buffer.
  - `safety_invalid_attr_rejected` — per-policy-fail attr ⟹ -EINVAL + extack populated.
  - `liveness_parse_terminates` — per-msg parse halts (bounded by buffer-len).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Nla::parse` post: tb[type] populated for valid attrs; rest None | `Nla::parse` |
| `Nla::validate` post: per-policy-pass ⟹ value satisfies policy constraint | `Nla::validate` |
| `Nla::put_u32` post: emitted nlattr.nla_type matches; data == value | `Nla::put_u32` |
| `Nla::get_u32` post: returned == data-at-attr; preconditioned on nla_len ≥ 4 | `Nla::get_u32` |

### Layer 4: Verus/Creusot functional

`Per-skb policy-pass via Nla::parse → tb[] populated for handlers; per-policy-fail → -EINVAL with extack.bad_attr indicating offending attr` semantic equivalence: per-policy validation matches Linux netlink wire-format spec (RFC 3549).

### hardening

(Inherits row-1 features from `net/netlink/af_netlink.md` § Hardening.)

NLA-policy-specific reinforcement:

- **Per-attr nla_len ≤ remaining buffer enforced** — defense against malformed-len OOB read.
- **Per-NLA_STRING NUL-termination required** — defense against unterminated string causing OOB read.
- **Per-NLA_BINARY len ∈ [min, max]** — defense against zero-byte or huge-data crashing handler.
- **Per-NLA_NESTED depth ≤ 32** — defense against stack overflow via deep nesting.
- **Per-NLA_REJECT immediately rejects** — defense against guest using deprecated attr.
- **Per-extack populated on any failure** — defense against silent error.
- **Per-validate_strict default for new families** — defense against unknown-type bypass.
- **Per-NLA_BITFIELD32 mask enforced** — defense against bits beyond declared mask.
- **Per-NLA_VALIDATE_MASK enforced** — defense against value bits outside policy.
- **Per-NLA_F_NESTED flag bit checked** — defense against nesting-flag mismatch.
- **Per-emit nla_put alignment enforced** — defense against unaligned-attr causing parser-skew downstream.
- **Per-msg attr-count limit** — defense against attr-bomb DoS.

