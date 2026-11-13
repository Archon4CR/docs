# Phase 1 Report: Freezing and Verifying the Dependency Boundary

Status: Completed
Date: 2026-08-17
Branch: `archon-integration-entrypoints`

## 1. Toolchain and Dependency Version Consistency

| Item | Value |
| --- | --- |
| `lean-toolchain` | `leanprover/lean4:v4.29.0` |
| mathlib `rev` (lake-manifest) | `8a178386ffc0f5fef0b77738bb5449d50efeea95` (corresponding to the `v4.29.0` tag) |
| `lakefile.toml` `require.mathlib.rev` | `v4.29.0` |
| `defaultTargets` | `["complexityreduction"]` |

Conclusion: ComplexityReduction's lean-toolchain and mathlib versions are self-consistent, both at
`v4.29.0`. Before integrating the target project (`archon-test`), the same toolchain and locked
mathlib version must be kept to avoid duplicate resolution.

## 2. Stable Archon Import Surface

The root module `ComplexityReduction.lean` already aggregates the following 8 entry surfaces; a target
project only needs `import ComplexityReduction` to obtain all public symbols:

| Entry surface | Aggregated module | Submodule count | Purpose |
| --- | --- | --- | --- |
| `Core` | `ComplexityReduction.Core` | — | Basic types such as `EncodedDecisionProblem` |
| `SAT` | `ComplexityReduction.SAT` | 134 imports | 3SAT decision problem, Cook-Levin wrapper |
| `CSP` | `ComplexityReduction.CSP` | 88 imports | Boolean CSP formulas, constraints, encodings |
| `Schaefer` | `ComplexityReduction.Schaefer` | 32 imports | Schaefer dichotomy theorem, tractability conditions |
| `Karp21` | `ComplexityReduction.Karp21` | 318 imports + definitions | Enumeration and descriptors of the 21 Karp NPC problems |
| `Bridges` | `ComplexityReduction.Bridges` | 57 imports | Costed→TM bridges, soundness proofs |
| `Certificates` | `ComplexityReduction.Certificates` | 5 imports | Reduction certificates (costed / log-space) |
| `Targets` | `ComplexityReduction.Targets` | 11 imports | NPC target catalog and descriptors |

Notes:

- `Karp21.lean` is both an aggregated module and the carrier of the 21 definitions such as
  `Problem` / `ProblemDescriptor`, 744 lines in total (320 lines of imports + 424 lines of definitions).
- The actual declarations of `Bridges` and `Certificates` do not live in same-named namespaces but
  directly under the `ComplexityReduction` namespace (e.g. `CostedTMSound`, `CostedReduction`).
  The Archon reference documentation must give the symbol table by the real fully qualified names.
- `PackagedEdgeTransfers.lean` previously formed a cycle because of `import ComplexityReduction.Karp21`
  (`Karp21 → PackagedEdgeTransfers → Karp21`); it was changed to 44 direct submodule imports, the cycle
  was eliminated, and the full build passes.

## 3. Build Verification

- `lake build ComplexityReduction`: the full 3982 jobs compiled successfully, with only warnings
  remaining (unused simp arguments, `simpa` vs `simp`, unused variables) and no errors.
- Historical fixes:
  - `ClauseFold.lean` kernel deep recursion: split into 4 private lemmas and closed with `omega`
    (commit `5b8c705`).
  - Aggregated module creation and circular import fix (commit `b9dcb2b`).

## 4. Smoke Test

The smoke test file lives at `scripts/smoke_test.lean` (`scripts/` is in `.gitignore`, so it is not
committed); via `import ComplexityReduction`, it runs `#check` on representative public symbols of the
8 entry surfaces:

```text
Karp21.Problem                                      : Type
Karp21.ProblemDescriptor                            : Type
Karp21.problemDescriptors                           : List Karp21.ProblemDescriptor
Karp21.Problem.all                                  : List Karp21.Problem
(Karp21.Problem.all.length = 21)                    : Prop
SAT.threeSATDecisionProblem                         : EncodedDecisionProblem
SAT.TMNPCompleteEncWithCheckedSuffixValidity        : Prop
CSP.Constraint                                      : Type
CSP.BoolLanguage                                    : Type 1
CSP.BoolRel                                         : Type
Schaefer.TractableConditionKind                     : Type
Schaefer.RelationSatisfies                          : Prop
CostedTMSound                                       : Type 1
CostedReduction                                     : Type
Targets.NPCTargetDescriptor                         : Type
Targets.threeSATTarget                              : Targets.NPCTargetDescriptor
```

How to run (must be at the repository root):

```powershell
lake env lean scripts/smoke_test.lean
```

Result: all 16 `#check`s passed, exit code 0, no `unknownIdentifier`.

## 5. Targets Default Availability and Explicit Opt-in

The `Targets` aggregated module imports the following 9 submodules (the two Schaefer bridge submodules
are not among them):

Available by default (brought directly into scope by `import ComplexityReduction.Targets`):

- `Targets.ThreeSAT` — the local 3SAT target.
- `Targets.ThreeSATCompleteBoundary` — the 3SAT complete boundary.
- `Targets.NPCTargetCatalog` — the initial NPC target catalog (`NPCTargetDescriptor`,
  `threeSATTarget`, `localTargets`, etc.).
- `Targets.Karp21Catalog` — the Karp 21 catalog.
- `Targets.CountingCSP` / `Targets.FiniteDomainCSP` — counting and finite-domain CSP targets.
- `Targets.HColoringCatalog` / `Targets.HColoringComplexityCatalog` — H-coloring catalogs.
- `Targets.NPCTransfer` — NPC transfer.

Requiring explicit opt-in (external Schaefer bridges, not used for local reasoning by default):

- `Targets.Schaefer` — only records boundary metadata of the external Schaefer dichotomy project. Its
  module header states explicitly: the sibling project `../schaefer_dichotomy_lean` is also named
  `ComplexityReduction`, which as a Lake dependency would create module-name ambiguity; therefore this
  repository does not import its theorems and only keeps audit-only bridge metadata. The module is
  still imported indirectly by `NPCTargetCatalog` (only referencing the
  `ExternalBridgeKind.trustedBoundary.asString` string constant), but it does not carry over any
  external theorems.
- `Targets.SchaeferFiniteDomain` — Schaefer finite-domain related, likewise part of the external
  bridge category. This module is not in the aggregated module, and no module in the default set
  imports it; an explicit `import ComplexityReduction.Targets.SchaeferFiniteDomain` is required for it
  to enter scope.

The substantive guarantee of the trust boundary is at the **data level** rather than purely the import
level: in `NPCTargetCatalog`, `localTargets = [threeSATTarget]` and `initialTargets = localTargets`, so
the default catalog contains only the local 3SAT target; the external Schaefer descriptors
(`schaeferBridgeTargets`) all carry `requiresExplicitOptIn = true` and
`artifactEligibleByDefault = false`, and only `targetsWithExternalBridges = localTargets ++ schaeferBridgeTargets`
includes them. Therefore, even though the metadata types of `Targets.Schaefer` enter scope together
with `NPCTargetCatalog`, the external bridge targets are not available for artifact generation by
default.

## 6. Known Limitations and Follow-Ups

- The aggregated modules are currently pure import collections and do not re-export symbols under a
  same-named namespace; the Archon symbol table must be maintained by the real fully qualified names
  (see Section 4).
- The smoke test file is not committed (constrained by the `/scripts` rule in `.gitignore`); if it
  needs to be included in CI, a separate test target outside `scripts/` can be created — left for
  Phase 2 to decide.
- Phase 1 is complete; Phase 2 (changes on the Archon source side) can begin.
