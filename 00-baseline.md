---
title: "Foundation: Upstream Linux baseline"
tags: ["design-doc", "foundation"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---

# Foundation: Upstream Linux baseline

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - Makefile
  - (this doc IS the project-wide baseline pin; other paths are referenced in the verification commands within)
status: draft
-->

## Summary
Pins the exact upstream Linux source revision that all design documents in this project are reverse-engineered against. Every subsystem design doc must declare the baseline version it was authored against; when the baseline is bumped, docs whose subsystem changed must be re-reviewed.

## Requirements

- REQ-1: A single canonical upstream commit hash is recorded here, refreshable on a documented cadence.
- REQ-2: Every design doc in tiers 2-5 (subsystem, component, driver, uapi) MUST declare its baseline commit in a frontmatter line so stale docs are detectable after a rebaseline.
- REQ-3: Scope statistics (file counts, top-level directory layout) are recorded here so future readers can see what fraction of the kernel a given design pass has covered.
- REQ-4: A documented refresh procedure exists so the baseline can be advanced without breaking outstanding design work.

## Acceptance Criteria

- [ ] AC-1: A "Pinned baseline" section names the exact commit hash, version string, clone date, and clone path.
- [ ] AC-2: A "Scope statistics" section enumerates file counts by language and top-level directories, with both numbers verifiable via shell commands listed in this doc.
- [ ] AC-3: A "Refresh procedure" section describes how to advance the baseline (git fetch, update this file, scan design docs for stale frontmatter, file rebaseline review issues).
- [ ] AC-4: A "Frontmatter convention" section gives the exact line every other design doc must include.

## Pinned baseline

| Field | Value |
|---|---|
| Upstream commit | `27a26ccfd528da725a999ea1e3102503c61eb655` |
| Kernel version | `7.1.0-rc2` (per `Makefile`: VERSION=7, PATCHLEVEL=1, SUBLEVEL=0, EXTRAVERSION=-rc2) |
| Tip subject | `Merge tag 'arm64-fixes' of git://git.kernel.org/pub/scm/linux/kernel/git/arm64/linux` |
| Tip author | Linus Torvalds <torvalds@linux-foundation.org> |
| Tip date | 2026-05-08 16:18:35 -0700 |
| Clone date | 2026-05-09 |
| Clone path | `/home/doll/linux-src/` |
| Clone depth | 1 (shallow) |
| Source URL | `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git` |

To verify the pin at any time:

```sh
cd /home/doll/linux-src
git rev-parse HEAD              # must print 27a26ccfd528da725a999ea1e3102503c61eb655
head -5 Makefile                # must show VERSION=7 PATCHLEVEL=1 SUBLEVEL=0 EXTRAVERSION=-rc2
```

## Scope statistics

Top-level directories in the upstream tree (24 total):

```
arch/      block/     certs/     crypto/    Documentation/   drivers/   fs/
include/   init/      io_uring/  ipc/       kernel/          lib/       LICENSES/
mm/        net/       rust/      samples/   scripts/         security/  sound/
tools/     usr/       virt/
```

Source file counts in this baseline:

| Language | File count | Notes |
|---|---|---|
| C  (`*.c`)        | 36,678 | Bulk of the kernel; reverse-engineering target |
| C  (`*.h`)        | 26,673 | Headers, including `include/uapi/` (the userspace ABI surface) |
| Rust (`*.rs`)     |    349 | Existing rust-for-linux abstractions in `rust/kernel/`; we extend, not replace |
| Assembly (`*.S`)  |  1,364 | Per-arch entry, boot, and crypto primitives. v0 only cares about `arch/x86/` |

Verify with:

```sh
cd /home/doll/linux-src
find . -name '*.c' | wc -l
find . -name '*.h' | wc -l
find . -name '*.rs' | wc -l
find . -name '*.S' | wc -l
```

## Frontmatter convention

Every design doc in tiers 2-5 MUST begin with this comment block, immediately after the title:

```markdown
# Feature: <title>

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - <linux-relative-path-1>
  - <linux-relative-path-2>
status: draft | reviewed | approved
-->
```

`upstream-paths` lists every Linux source path the doc describes. When the baseline is advanced, a refresh script can:

1. List every file that changed between old and new baseline:
   `git -C ~/linux-src diff --name-only <old-commit> <new-commit>`
2. Cross-reference against the union of `upstream-paths` across `.design/**/*.md`
3. Open a crosslink rebaseline-review issue for each design doc whose paths intersect changed files.

(The script itself does not yet exist; tracked separately in the implementation backlog.)

## Refresh procedure

Advancing the baseline (e.g. when Linux 7.1 final lands):

1. **Fetch**: `cd /home/doll/linux-src && git fetch origin && git checkout <new-tag-or-commit>`
2. **Update this file**: replace the "Pinned baseline" table with new values; recompute scope statistics.
3. **Refresh-review issue**: open a single crosslink issue titled `Rebaseline design docs: <old> → <new>`. Use it as the parent for per-subsystem re-review subissues.
4. **Per-subsystem re-review**: for each design doc whose `upstream-paths` intersect the diff, open a child issue. The reviewer checks the doc against the new source and either bumps `baseline-commit` (no semantic change) or amends the doc (semantic drift).
5. **Mark approved**: only after re-review, bump `baseline-commit` in the doc's frontmatter. The status field never advances past `reviewed` on a stale baseline.

Cadence: review at least once per year, or on each Linux LTS bump (~ every 8 months). Faster cadence as design coverage matures.

## Out of Scope

- Pinning per-subsystem to *different* baselines — we use a single project-wide baseline. Subsystem-level baseline pinning would multiply rebaseline complexity without obvious gain.
- Tracking individual file content hashes for staleness detection — `upstream-paths` granularity is sufficient.
- Tracking the rust-for-linux `rust/` subtree as a separate baseline — it ships in the main Linux tree, so the same commit hash governs both.
