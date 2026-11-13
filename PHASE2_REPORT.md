# Phase 2 Report: Changes on the Archon Source Side

Status: Completed (including wrap-up fixes)
Date: 2026-08-19
Branch: `complexity-reduction-integration` (Archon repository)

## Overview

The goal of Phase 2 was to solidify ComplexityReduction into Archon's packaged data as first-class
knowledge. In total, 4 changes were completed, involving 4 new files and 5 modified files, spread
across four layers of Archon: skill documentation, search scripts, prover-mode, and Python code.

## Commit Log

| Commit | Content | Files |
| --- | --- | --- |
| `964c315` | 2.1 Library reference document | `references/complexity-reduction-library.md` (new), `SKILL.md` (modified) |
| `33cf065` | 2.3 Search script | `lib/scripts/search_complexity_reduction.sh` (new) |
| `34aa493` | 2.2 Build mode | `prover-modes/complexity-reduction-build.md` (new) |
| `8401e80` | 2.4 Lake dependency hints | `lake.py`, `project.py`, `semantic_pass.py` (all modified) |
| `e573680` | 2.4 Lake dependency fixes | `lake.py` (modified), `test_init_mathlib_no_update.py` (modified), `test_complexity_reduction_lake.py` (new) |

## Wrap-Up Fixes

During the 2.4 wrap-up stage, `add_complexity_reduction_dependency()` from `8401e80` was found to have
two issues that needed fixing:

- The Lean format incorrectly generated `from Path "..."`, which does not conform to Lake 4.29 syntax;
  changed to `from "..."`.
- The dependency name incorrectly used lowercase `complexityreduction`; changed to the standard package
  name `ComplexityReduction`.

The same commit added regression tests for the new detection/write logic and fixed the existing tests
affected by the new `LakefileInfo` fields.

## 2.1 Library Reference Document

**File:** `src/archon/.archon-src/skills/lean4/skills/lean4/references/complexity-reduction-library.md`

Covered content:

- **Stable import surface** — the 8 aggregated modules
  (Core/SAT/CSP/Schaefer/Karp21/Bridges/Certificates/Targets) + the root import
  `import ComplexityReduction`, plus the standalone module `ComplexityReduction.Templates`.
- **Symbol table** — the public symbols of 7 modules, all verified via `#check`. It notes that the
  declarations of `Bridges` and `Certificates` are not under a same-named namespace (e.g.
  `CostedTMSound` lives directly under `ComplexityReduction`; there is no `Bridges.CostedTMSound`).
- **Targets catalog and trust boundary** — the 7 submodules available by default vs. the 2 Schaefer
  bridge submodules requiring explicit opt-in (`Targets.Schaefer`, `Targets.SchaeferFiniteDomain`),
  with an explanation of the module-name ambiguity.
- **Templates usage scenarios** — a comparison table of the 8 template submodules
  (Lists/FormulaAppend/Renaming/Substitution/Blocks/Structural/StrictLists/SubsetLifting).
- **Build and verification** — the `lake build` command, axiom-clean checks, and an explanation of the
  doubly nested directory structure. Paths are described generically, with no hardcoded absolute paths.

Added a **Libraries** category index to the References section of `SKILL.md`.

## 2.3 Search Script

**File:** `src/archon/.archon-src/skills/lean4/lib/scripts/search_complexity_reduction.sh`

Modeled on the structure of `search_mathlib.sh`, it supports three search modes:

- `name` — search by declaration name (`^(theorem|lemma|def|class|structure|inductive|abbrev).*$QUERY`)
- `type` — search by type signature (`: .*$QUERY`)
- `content` — search by file content (full-text match)

Configuration:

- The CR source path is specified via the `COMPLEXITY_REDUCTION_PATH` environment variable, defaulting
  to `../ComplexityReduction/ComplexityReduction` (the inner directory).
- It prefers `rg` and automatically falls back to `grep` when `rg` is unavailable.

Verification: all three modes were tested with Git Bash against the real CR source and passed.

## 2.2 Build Mode

**File:** `src/archon/.archon-src/prover-modes/complexity-reduction-build.md`

The frontmatter aligns with `mathlib-build.md` (`name`/`description`/`compatible_stages`/`read_blueprint`/`dispatcher_notes`).

Core requirements of the body:

1. **Check CR's existing declarations first** — Section 2 guides the use of the search script + the
   reference document's symbol table, listing 5 self-check questions (does the source problem already
   have an encoding, is the target in `Targets`, is there a composable `CostedReduction`, is there a
   matching template, can a transfer theorem close it).
2. **Prefer reductions/templates/bridges** — Section 4 makes explicit the priority of
   `CostedReduction.comp`, `Templates.*`, `NPCompleteEnc.transfer`, and `composeTMKarp`, to avoid
   proving from scratch.
3. **Axiom-clean for every declaration** — an Invariant section + the verification steps in Section 3;
   `#print axioms` only allows `{propext, Classical.choice, Quot.sound}`.
4. **Give a resumable decomposition before stopping** — Section 7 includes a concrete example,
   requiring precise declaration types and the next method to use.

The Section 5 Targets trust boundary echoes the 2.1 reference document, reminding that Schaefer
bridges require explicit opt-in.

## 2.4 Lake Dependency Hints

**Files:** `src/archon/commands/tooling/lake.py`, `src/archon/commands/tooling/project.py`,
`src/archon/commands/init/steps/semantic_pass.py`

### lake.py

- `LakefileInfo` gains a new `has_complexity_reduction: bool` field.
- `lakefile_info()` detects the CR dependency via regexes:
  - lean format: `require\s+[Cc]omplexity[Rr]eduction\b`
  - toml format: `name\s*=\s*"[Cc]omplexity[Rr]eduction"`
- A new `add_complexity_reduction_dependency(source_path, if_absent=True)` method supports writing
  local-path dependencies in both lean/toml formats. The lean format uses
  `require ComplexityReduction from "../ComplexityReduction"`; toml uses `name = "ComplexityReduction"`
  + `path = "../ComplexityReduction"`.

### project.py

- `ProjectLayout` gains a new `has_complexity_reduction: bool` field.
- `inspect()` passes the field through from `LakefileInfo`.

### semantic_pass.py

- The bootstrap summary's layout dict gains a new `has_complexity_reduction` key, so that Claude can
  perceive the CR dependency status during init.

### Verification

Tested with the Archon venv:

- ComplexityReduction repository: `has_mathlib=True, has_cr=True` (toml format)
- `archon-test` project: `has_mathlib=True, has_cr=False` (lean format)

The detection logic handles both casings; `tests/test_complexity_reduction_lake.py` and
`tests/test_init_mathlib_no_update.py` contain 18 tests in total, all passing, covering lean/toml
detection, write syntax, idempotency, and the no-lakefile error.

### Design Decisions

Per the plan of "add CR-specific recognition first, then generalize to a `declared_dependencies` list
later", this step only does recognition and display, and does not hook into the bootstrap flow that
automatically adds the CR dependency. The `add_complexity_reduction_dependency()` method is ready and
can be called directly when the target project is integrated in Phase 3.

## Known Limitations

- `add_complexity_reduction_dependency()` is not yet wired into `BootstrapOptions` and the `_ensure_*`
  flows; Phase 3 will use it.
- The search script is unavailable in a WSL bash environment due to an `rg` path issue (Codex's
  built-in rg has no permission); Git Bash or native Linux/macOS is required. This is a
  test-environment limitation and does not affect the script itself.
- `semantic_pass.py`'s `has_complexity_reduction` is only surfaced in the bootstrap summary; the
  `doctor` command does not yet add hints for version mismatches or missing dependencies. This can be
  added later.

## Phase 2 complete; Phase 3 (target-project-side integration) can begin
