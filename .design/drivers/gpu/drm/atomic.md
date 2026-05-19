# Tier-3: drivers/gpu/drm/{drm_atomic,drm_atomic_helper,drm_atomic_state_helper,drm_atomic_uapi}.c — DRM atomic-modeset commit pipeline

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
upstream-paths:
  - drivers/gpu/drm/drm_atomic.c
  - drivers/gpu/drm/drm_atomic_helper.c
  - drivers/gpu/drm/drm_atomic_state_helper.c
  - drivers/gpu/drm/drm_atomic_uapi.c
  - include/drm/drm_atomic.h
  - include/drm/drm_atomic_helper.h
-->

## Summary

DRM atomic-modeset is the modern replacement for the legacy SetCrtc/SetMode IOCTL: every Wayland compositor (gnome-shell, plasma-wayland, sway, kwin, hyprland), every X11 modesetting driver, every kmscube-style direct-KMS app drives display via `DRM_IOCTL_MODE_ATOMIC`. The atomic-commit primitive lets userspace propose a new state for a set of CRTCs+planes+connectors+properties as a single transaction; the driver checks if the proposal is achievable, and if so commits it atomically (typically at vblank boundary). Failure rolls back to prior state — userspace never sees torn intermediate states.

This Tier-3 covers `drm_atomic.c` (~2100 lines: per-state-object lifecycle, generic commit-state machine), `drm_atomic_helper.c` (~4100 lines: per-driver helpers for the most common commit shapes), `drm_atomic_state_helper.c` (per-object state-init helpers), `drm_atomic_uapi.c` (UAPI marshalling).

Owns: per-`drm_atomic_state` object table (one per in-flight commit; refcount-tracked across fence-wait + commit-tail), per-property "set" + "get" wrappers, generic commit shapes (`drm_atomic_helper_commit`, `_check`, `_commit_modeset_disables`, `_commit_modeset_enables`, `_commit_planes`, `_wait_for_dependencies`, `_wait_for_vblanks`, `_wait_for_flip_done`), nonblocking-commit infrastructure with implicit/explicit fence support, mode validation cascade (`drm_atomic_helper_check_modeset`, `_check_planes`, `_check_crtc_state`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct drm_atomic_state` | per-commit aggregate state | `drm::atomic::State` |
| `struct drm_crtc_state` | per-CRTC proposed state | `drm::atomic::CrtcState` |
| `struct drm_plane_state` | per-plane proposed state | `drm::atomic::PlaneState` |
| `struct drm_connector_state` | per-connector proposed state | `drm::atomic::ConnectorState` |
| `struct drm_private_state` | per-driver-private object proposed state | `drm::atomic::PrivateState` |
| `drm_atomic_state_alloc(dev)` / `_free(state)` / `_get(state)` / `_put(state)` | state lifecycle | `State::alloc` / `_free` / `_get` / `_put` |
| `drm_atomic_get_crtc_state(state, crtc)` / `_plane_state` / `_connector_state` / `_private_state` | duplicate-on-write per-object state | `State::get_*_state` |
| `drm_atomic_check_only(state)` | run check phase, no commit | `State::check_only` |
| `drm_atomic_commit(state)` | check + commit (blocking) | `State::commit` |
| `drm_atomic_nonblocking_commit(state)` | check + queue async commit | `State::nonblocking_commit` |
| `drm_atomic_helper_check(dev, state)` | generic check helper | `helper::check` |
| `drm_atomic_helper_check_modeset(dev, state)` | modeset-only check | `helper::check_modeset` |
| `drm_atomic_helper_check_planes(dev, state)` | per-plane check helper | `helper::check_planes` |
| `drm_atomic_helper_commit(dev, state, nonblock)` | generic commit helper | `helper::commit` |
| `drm_atomic_helper_swap_state(state, stall)` | swap proposed → active | `helper::swap_state` |
| `drm_atomic_helper_setup_commit(state, nonblock)` | install commit fences | `helper::setup_commit` |
| `drm_atomic_helper_wait_for_dependencies(state)` | wait for IN_FENCE_FD + GEM-implicit fences | `helper::wait_for_dependencies` |
| `drm_atomic_helper_commit_modeset_disables(dev, state)` | disable departing CRTCs/encoders/connectors | `helper::commit_modeset_disables` |
| `drm_atomic_helper_commit_modeset_enables(dev, state)` | enable arriving CRTCs/encoders/connectors | `helper::commit_modeset_enables` |
| `drm_atomic_helper_commit_planes(dev, state, flags)` | commit plane updates (active scanout buffer change) | `helper::commit_planes` |
| `drm_atomic_helper_wait_for_vblanks(dev, state)` | wait for next vblank on each affected CRTC | `helper::wait_for_vblanks` |
| `drm_atomic_helper_wait_for_flip_done(dev, state)` | wait for flip-completion fence | `helper::wait_for_flip_done` |
| `drm_atomic_helper_commit_hw_done(state)` / `_commit_cleanup_done(state)` | per-stage done callbacks | `helper::commit_hw_done` / `_commit_cleanup_done` |
| `drm_atomic_helper_async_commit(dev, state)` | async-commit fast-path for cursor-only updates | `helper::async_commit` |
| `drm_atomic_helper_async_check(dev, state)` | async-commit check | `helper::async_check` |
| `drm_atomic_set_property(state, file, obj, property, value)` | set per-object property in proposed state | `State::set_property` |
| `drm_atomic_get_property(obj, property, &value)` | get per-object property from active state | `Object::get_property` |

## Compatibility contract

REQ-1: `DRM_IOCTL_MODE_ATOMIC` UAPI byte-identical: per-object property-array marshalling + DRM_MODE_ATOMIC_TEST_ONLY / _NONBLOCK / _ALLOW_MODESET flags + cursor-only async-commit detection.

REQ-2: Per-property set semantics byte-identical: every standard property (CRTC_ID, FB_ID, IN_FENCE_FD, OUT_FENCE_PTR, CRTC_X/Y/W/H, SRC_X/Y/W/H, IN_FORMATS, COLOR_ENCODING, COLOR_RANGE, ROTATION, ZPOS, PIXEL_BLEND_MODE, ACTIVE, MODE_ID, OUT_FENCE_PTR, VRR_ENABLED, GAMMA_LUT, CTM, DEGAMMA_LUT, MAX_BPC, HDR_OUTPUT_METADATA, EDID_CRC) accepted with byte-identical validation.

REQ-3: Check phase returns -EINVAL on mode validation failure; userspace can call test-only commit to validate before allocation. No driver state mutated during check (defense-in-depth: driver `atomic_check` callbacks must be side-effect-free).

REQ-4: Commit phase ordering: (1) wait for IN_FENCE_FD + GEM-implicit fences from prior CRTC scanout buffers, (2) swap proposed→active state, (3) commit_modeset_disables (driver `disable` callbacks for CRTCs/encoders/connectors leaving), (4) commit_planes (driver `atomic_update_plane`), (5) commit_modeset_enables (driver `enable` callbacks for CRTCs/encoders/connectors entering), (6) wait for vblank, (7) signal OUT_FENCE_PTR + commit_hw_done, (8) commit_cleanup_done frees prior framebuffers.

REQ-5: Nonblocking commit: check returns synchronously; commit-tail dispatched on dedicated workqueue thread. Multiple in-flight commits to disjoint CRTCs run concurrently; commits to overlapping CRTCs serialized via per-CRTC fences.

REQ-6: Async commit (cursor-only, plane-only-no-mode-change fast-path): returns synchronously after queuing; no vblank wait; used by compositors for sub-vblank cursor updates.

REQ-7: Atomic-check failure semantics: every error path correctly cleans up partial state allocation; returned -errno propagates to userspace; no driver state corruption.

REQ-8: OUT_FENCE_PTR semantics: per-CRTC outgoing sync_file fd installed on commit success; signals at vblank-after-commit. IN_FENCE_FD: per-plane incoming sync-file dependency; commit waits on it before scan-out swap.

REQ-9: Per-`drm_atomic_state` refcount: get'd at alloc, put'd at commit-cleanup-done; cleanup deferred until all per-CRTC commit-tail completes + all OUT_FENCE_PTR fences signaled.

REQ-10: Per-driver `funcs->atomic_*` callbacks invoked at exactly the documented stage; per-object state pointers in callbacks are stable within commit-tail (no concurrent state swap mid-callback).

## Acceptance Criteria

- [ ] AC-1: `kmscube` runs at native refresh rate without dropped frames on amdgpu/i915/xe.
- [ ] AC-2: Sway compositor on amdgpu reference HW: multi-monitor mode set + per-monitor refresh-rate change works.
- [ ] AC-3: Cursor-update test: gnome-shell mouse-cursor movement uses async-commit fast-path; verified via tracepoint.
- [ ] AC-4: Multi-CRTC test: simultaneous mode-set on 2 CRTCs (different displays) commits both via single ATOMIC ioctl.
- [ ] AC-5: VRR test: VRR-enabled monitor receives time-stretched scan-out timings from atomic commit.
- [ ] AC-6: Sync-file in/out fence test: GBM-imported dma-buf with explicit IN_FENCE_FD waits for prior render; OUT_FENCE_PTR signals at vblank.
- [ ] AC-7: Test-only commit test: `DRM_MODE_ATOMIC_TEST_ONLY` returns -EINVAL for invalid mode; subsequent real commit not executed (no state mutation observed).
- [ ] AC-8: kselftest `tools/testing/selftests/drm/` atomic subset passes.
- [ ] AC-9: IGT (`igt-gpu-tools`) `kms_atomic` + `kms_atomic_transition` test suites pass on amdgpu + i915.

## Architecture

`State` lives in `drm::atomic::State`:

```
struct State {
  refcount: Refcount,
  dev: Arc<DrmDevice>,
  allow_modeset: bool,
  legacy_cursor_update: bool,
  async_update: bool,
  duplicated: bool,
  planes: Mutex<Vec<PerObjectState<PlaneState>>>,
  crtcs: Mutex<Vec<PerObjectState<CrtcState>>>,
  connectors: Mutex<Vec<PerObjectState<ConnectorState>>>,
  private_objs: Mutex<Vec<PerObjectState<PrivateState>>>,
  acquire_ctx: Option<AcquireCtx>,        // ww_mutex acquire context
  fake_commit: Option<KBox<CommitInfo>>,  // for fence injection
  commits_per_crtc: ArrayMutex<MAX_CRTCS, Vec<Arc<CommitInfo>>>,
}
```

Atomic-commit pipeline (`State::commit`):

1. **Check phase** — `helper::check(dev, state)`:
   - For each duplicated per-object state, call per-driver `funcs->atomic_check(obj, state)`.
   - Generic helpers: `check_modeset` (validate mode against connector caps + bridge chain), `check_planes` (clip plane against CRTC region + format-modifier compatibility), `check_crtc_state` (active vs inactive transitions).
   - On any -EINVAL: roll back, return error.

2. **Setup commit** — `helper::setup_commit(state, nonblock)`:
   - Allocate per-CRTC `CommitInfo` w/ flip_done + commit_done + hw_done + cleanup_done completions.
   - Stash IN_FENCE_FDs from per-plane state.
   - Install OUT_FENCE_PTR sync-files (allocate per-CRTC `dma_fence`).
   - For nonblocking: link this commit to per-CRTC's prior commit (serialize per-CRTC).

3. **Stall on prior commit** — for each affected CRTC, wait_for_completion(prior.flip_done).

4. **Swap state** — `helper::swap_state(state, stall=true)`:
   - For each per-object: swap `state.<obj>_state` ↔ `obj->state` atomically.
   - After swap, `state.<obj>_state` holds the old (about-to-be-released) state; `obj->state` holds the new.

5. **Commit-tail dispatch** — for nonblocking, schedule per-CRTC commit-tail work; for blocking, run inline:
   - `helper::wait_for_dependencies(state)` — wait IN_FENCE_FD + GEM-implicit dma-fences.
   - `helper::commit_modeset_disables(dev, state)` — per-driver disable callbacks for CRTCs/encoders/connectors with new state `enable=false`.
   - `helper::commit_planes(dev, state, flags)` — per-driver `atomic_update_plane` (DMA scanout buffer + scan-out region update).
   - `helper::commit_modeset_enables(dev, state)` — per-driver enable callbacks for CRTCs/encoders/connectors with new state `enable=true`.
   - `helper::wait_for_vblanks(dev, state)` — block until vblank fires on each affected CRTC (so userspace sees scan-out completed).
   - `helper::commit_hw_done(state)` — completion for hw-done waiters (libdrm DRM_EVENT_FLIP_COMPLETE).
   - Signal per-CRTC OUT_FENCE_PTR sync-file.
   - `helper::commit_cleanup_done(state)` — release prior framebuffers, complete cleanup.
   - Drop `state` refcount.

6. Return to userspace (synchronous for blocking; immediately for nonblocking after step-3 schedule).

Async commit (`helper::async_commit`):
- Cursor-only or plane-only-no-mode-change fast-path.
- Skip ww_mutex dance, skip fence wait, skip vblank wait.
- Used by gnome-shell mouse cursor for sub-vblank latency.
- Per-driver `funcs->atomic_async_update` callback called directly.

ww_mutex: per-CRTC + per-plane + per-connector ww_mutexes acquired in canonical order via `AcquireCtx`. On lock-conflict (-EDEADLK) returned by `drm_modeset_lock` due to wound-wait protocol, caller retries from scratch with backoff.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `state_no_uaf` | UAF | `Arc<State>` outlives all per-CRTC commit-tail workqueue work; cleanup deferred until refcount==0. |
| `swap_state_atomic` | ATOMICITY | per-object state-swap done with proper memory barriers; concurrent reader sees old or new fully (never partial). |
| `nonblock_serialize_per_crtc` | ORDERING | nonblocking commits to same CRTC serialized via prior.flip_done wait; per-CRTC commit-order matches submission order. |
| `check_no_state_mutation` | SIDE-EFFECT | check phase invokes per-driver atomic_check callbacks — verified to not mutate active state (only proposed state). |

### Layer 2: TLA+

`models/drm/atomic_commit.tla` (parent-declared): proves atomic-modeset commit dependency ordering — async commits properly ordered via vblank fences; CHECK + ATOMIC + NONBLOCK paths; concurrent commits on disjoint CRTCs progress in parallel; rollback on validation fail leaves no partial state.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `State::commit` post (success): every per-object's `obj->state` reflects the proposed values; prior state freed after cleanup_done | `State::commit` |
| `State::commit` post (failure): every per-object's `obj->state` unchanged from pre-commit | `State::check_only` (rollback) |
| `helper::commit_modeset_disables` invariant: only invoked for CRTCs/encoders/connectors with new `enable=false` AND old `enable=true` | `helper::commit_modeset_disables` |
| `helper::commit_modeset_enables` invariant: only invoked for CRTCs/encoders/connectors with new `enable=true` AND (old `enable=false` OR mode-changed) | `helper::commit_modeset_enables` |
| Per-CRTC OUT_FENCE_PTR sync-file signals exactly once at vblank-after-commit | `helper::wait_for_vblanks` |

### Layer 4: Verus/Creusot functional

`State::commit` ↔ upstream `drm_atomic_commit` semantic equivalence: for any sequence of property sets followed by commit, the post-commit `obj->state` for every object matches upstream. Encoded as Verus model: `forall props. rookery_atomic_commit(props) == upstream_atomic_commit(props)` (both result in same per-object state OR same -errno).

## Hardening

(Inherits row-1 features from `drivers/gpu/drm/00-overview.md` § Hardening.)

atomic-specific reinforcement:

- **Per-state refcount saturating** — overflow saturates at u32::MAX; defense against ref-count-overflow attack.
- **Check phase no-mutation invariant** — driver `atomic_check` callbacks are read-only on active state; helpers + verifier audit; defense against partial-commit corruption on validation failure.
- **Per-object mutex acquire order canonical** — ww_mutex wound-wait prevents deadlock between concurrent commits; defense against compositor-bug causing system-wide modeset deadlock.
- **OUT_FENCE_PTR signals on commit-failure too** — never leave userspace blocked on a fence that won't signal. On check-fail path, sync-file is created + signaled with error status.
- **DRM_MODE_ATOMIC_TEST_ONLY rejected mid-commit-tail** — flag checked at userspace-entry only; cannot be set on already-running commit (defense against TOCTOU between check + commit decision).
- **Per-CRTC commit-tail count cap = 64** — defense against compositor flooding commits faster than vblank rate; over-cap returns -EBUSY.
- **DRM_MODE_ATOMIC_ALLOW_MODESET requires DRM_MASTER** — non-master clients (e.g., DRM_LEASE recipients) cannot trigger mode-changes outside their leased CRTC subset.
- **Per-driver atomic_check callback time-bounded** — per-driver hook expected to complete within sane bound (~10ms); slow-driver detection via lockdep + tracepoint.
- **DMA-fence dependency wait timeout** — per-fence wait bounded (default 10s); defense against hung GPU causing compositor freeze.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — strict copy_from/to_user bounds on every `drm_mode_atomic` ioctl property blob and on user-provided fence-fd arrays, with allowlisted slab caches for `drm_atomic_state`.
- **PAX_KERNEXEC** — atomic-commit fast path runs with kernel `.text` strictly W^X; no runtime patching of plane/CRTC ops outside dedicated alternatives windows.
- **PAX_RANDKSTACK** — randomize the kernel stack offset across each `drm_atomic_check`/`drm_atomic_commit` ioctl entry to harden against stack-layout disclosure.
- **PAX_REFCOUNT** — saturating, overflow-trapping refcounts on `drm_atomic_state`, `drm_plane_state`, `drm_crtc_state`, and `drm_connector_state` (use `refcount_t`, never raw `atomic_t`).
- **PAX_MEMORY_SANITIZE** — zero-on-free for all `drm_atomic_state` slabs and per-object state caches so transient mode/property data never leaks into a reused allocation.
- **PAX_UDEREF** — enforce SMAP/PAN on all atomic-ioctl entry points; reject any commit path that dereferences a user pointer outside the canonical access helpers.
- **PAX_RAP / kCFI** — plane, CRTC, encoder, and connector helper vtables marked `__ro_after_init`; kCFI-typed indirect calls on `atomic_check`, `atomic_update`, `atomic_enable` so a corrupted helper pointer cannot redirect commit flow.
- **GRKERNSEC_HIDESYM** — keep `kallsyms` and KMS-helper addresses gated to CAP_SYSLOG; suppress `%p` plain pointers in any `drm_dbg_atomic` traces.
- **GRKERNSEC_DMESG** — restrict atomic-commit failure traces (driver-name + symbolised callstack) to CAP_SYSLOG to avoid handing attackers a precise display-stack map.
- **GRKERNSEC_FBSPLASH** — gate raw `/dev/fb*` and unaccelerated framebuffer mapping behind explicit policy; atomic KMS path is the only sanctioned modesetting surface.
- **drm-master capability gate** — keep `DRM_MASTER`/`DRM_RENDER_ALLOW` separation strict; render-node ioctls cannot reach atomic modeset paths even via flink/handle smuggling.
- **Property blob hardening** — bounded-size property blobs, signed length checks on `IN_FENCE_FD` arrays, and explicit rejection of negative/oversized plane source rectangles before reaching driver hooks.
- **TOCTOU on referenced objects** — atomic state takes refcounted snapshots of every referenced framebuffer/connector at duplicate-state time; no late dereference of a user-controlled handle after `atomic_check` returns.

Rationale: the atomic modesetting ioctl is one of the largest user-controlled-data surfaces in the DRM stack — every commit walks dozens of helper vtables across multiple driver-supplied state objects. Without refcount-overflow trapping, RAP/kCFI-typed indirect calls, and strict USERCOPY/UDEREF on property blobs, a single use-after-free in a `drm_atomic_state` or a corrupted plane helper pointer becomes a kernel-text redirect from any `DRM_MASTER` client.

## Out of Scope

- Per-driver atomic_check + atomic_commit callbacks (covered in `amdgpu.md`, `i915-display.md`, `xe.md`, etc. future Tier-3s)
- DRM legacy SetCrtc/SetMode (deprecated; not implemented in modern compositors)
- Per-mode validation (covered in `drivers/gpu/drm/edid-displayid.md` future Tier-3)
- KMS object lifecycle (covered in `drivers/gpu/drm/kms-objects.md` future Tier-3)
- vblank infrastructure (covered in `drivers/gpu/drm/vblank.md` future Tier-3)
- 32-bit-only paths
- Implementation code
