# Phase 3 Report: Target-Project-Side Integration

Status: Completed
Date: 2026-08-19
Branches: `master` (archon-test repo), `complexity-reduction-integration` (Archon repo)

## Overview

The goal of Phase 3 was to wire the `archon-test` target project into the
ComplexityReduction Lean library, and to activate the prover mode, subagent
descriptor, skill reference, and search scripts added on the Archon source
side in Phase 2. A total of 5 sub-steps + 2 incidental fixes were completed,
involving 6 new/modified files (git tracked) and 4 `.archon/` config files
(gitignored).

## Completed Steps

### 3.1 Lakefile Dependency

Added `require ComplexityReduction from "../ComplexityReduction"` in
`lakefile.lean`. Both `lake update` and `lake build ArchonTest` passed.

### 3.2 Smoke Import

Created `ArchonTest/ComplexityReductionSmoke.lean`, running `#check` on the
four key symbols of `ComplexityReduction.Targets`:

- `NPCTargetDescriptor` (type)
- `threeSATDecisionProblem` (value)
- `threeSATTarget` (descriptor)
- `localTargets` (list)

The build passed and all symbols are visible. Also added a `#print axioms`
check to verify axiom-clean (see item 8 of the verification checklist).

### 3.3 Library Reference Documentation

Created `references/ComplexityReduction.md`, documenting the library's module
layout, key types, and search workflow. Updated `references/summary.md` to add
the new reference entry.

### 3.4 Persistent User Hints

Added four persistent hints in `.archon/USER_HINTS.md`:

1. Before proving a problem NP-complete, first use the search scripts or the
   reference docs to check whether ComplexityReduction already has a
   corresponding encoding, reduction, or target descriptor.
2. New targets must pick the corresponding descriptor from
   `ComplexityReduction.Targets.localTargets`; do not construct NPC targets
   out of thin air.
3. The external Schaefer bridge submodules (`Targets.Schaefer`,
   `Targets.SchaeferFiniteDomain`) are not used for local reasoning by
   default, unless explicitly opted in.
4. Every artifact must be axiom-clean; `sorry` is forbidden; `#print axioms`
   may only show `propext`, `Classical.choice`, `Quot.sound`.

### 3.5 Subagent Descriptor and Configuration

Created `.archon/subagents/complexity-reduction-advisor.md` — a read-only,
non-spawning subagent that checks the CR library for reusable declarations
before a proof begins.

Enabled in `.archon/config.json`:

- Added to `subagents.enabled` (changed from `[]` to
  `["complexity-reduction-advisor"]`)
- Added to `subagents._available`
- Added to `subagents._recommended_plan_phase`

### Build Mode Backfill

Copied `complexity-reduction-build.md` from the Archon source into the target
project's `.archon/prover-modes/` (the target project had been initialized
before this file was added in Phase 2, hence it was missing).

## Verification Checklist

| # | Check | Result |
|---|-------|--------|
| 1 | `lake build complexityreduction` | Passed (7950 artifacts, exit code 0) |
| 2 | `lake build ArchonTest` | Passed |
| 3 | Smoke-file `#check` symbols | Passed (4/4 visible) |
| 4 | `archon doctor` recognizes the packed data | Passed (all OK) |
| 5 | Build mode copied to the target project | Passed |
| 6 | Subagent loaded by the registry | Passed |
| 7 | `archon loop --dry-run` | Passed (advisor enabled, loop starts normally) |
| 8 | `#print axioms` declaration check | Passed (axiom-clean) |

### Axiom-clean Details

```text
'NPCTargetDescriptor' does not depend on any axioms
'threeSATDecisionProblem' depends on axioms: [propext, Classical.choice, Quot.sound]
'threeSATTarget' does not depend on any axioms
'localTargets' does not depend on any axioms
```

Only Mathlib's three standard axioms appear, matching the policy in
USER_HINTS.

### Dry-Run Output

```text
Subagents: 1 of 16 enabled: complexity-reduction-advisor
```

The loop correctly identified the advisor as the only enabled subagent.

## Incidental Fixes

Fixed two pre-existing Windows encoding bugs that blocked `archon doctor` and
`archon loop` from running on GBK-locale systems:

- `src/archon/commands/doctor.py`: added `encoding="utf-8"` when
  `_parse_stage` reads PROGRESS.md.
- `src/archon/state/progress.py`: added `encoding="utf-8"` when `read_stage`
  reads PROGRESS.md.

Also created the verification script `scripts/verify_phase3_integration.py`,
which checks the prover-mode directory, mode-body loading, the subagent
registry, and config.json consistency; 4/4 items passed.

## Changed Files (archon-test)

| File | Status | Notes |
|------|--------|-------|
| `lakefile.lean` | modified | Added the CR dependency |
| `lake-manifest.json` | modified | Auto-updated by `lake update` |
| `ArchonTest/ComplexityReductionSmoke.lean` | new | Smoke import + axiom check |
| `references/ComplexityReduction.md` | new | Library reference doc |
| `references/summary.md` | modified | Added the CR reference entry |
| `.archon/subagents/complexity-reduction-advisor.md` | new (gitignored) | Subagent descriptor |
| `.archon/prover-modes/complexity-reduction-build.md` | new (gitignored) | Build mode |
| `.archon/USER_HINTS.md` | modified (gitignored) | Persistent hints |
| `.archon/config.json` | modified (gitignored) | Enabled the subagent |

## Commits

| Repo | Commit | Content |
|------|--------|---------|
| archon-test | `efd2230` | Phase 3 integration (lakefile, smoke file, reference docs, report) |
| archon-test | `52e7ed5` | Moved PHASE3_REPORT.md to the project root |
| Archon | `8925c36` | Encoding fixes + Phase 3 verification script |

## Status

Phase 3 is complete. All 8 verification items passed. Changes are committed
locally, not pushed.
