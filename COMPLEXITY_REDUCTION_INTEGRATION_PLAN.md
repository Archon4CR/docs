# ComplexityReduction and Archon Adaptation Plan

## Background

The goal is for Archon, in formalization tasks that need the ComplexityReduction library, to not merely rely on the agent reading the source code by itself, but to have explicit dependency detection, symbol references, proof modes, and search facilities.

The current directory layout:

- `D:\code\Lean\Archon`: Archon main source code.
- `D:\code\Lean\archon-test`: the existing Archon target project.
- `D:\code\Lean\ComplexityReduction`: the Lean 4 computational complexity reduction library.

`D:\code\Lean\scripts` has been emptied by the user. It is unrelated to Archon's own
`src/archon/.archon-src/scripts`, and no migration is needed.

## Key Conventions

- ComplexityReduction is not copied into the Archon repository.
- ComplexityReduction does not use the Archon `peers` mechanism, because it is not an Archon project but a Lake dependency library.
- The target project and ComplexityReduction must both stay on `leanprover/lean4:v4.29.0`, with mathlib pinned to `v4.29.0`.
- All content landing in the library rejects `sorry` by default and is checked via `#print axioms`, to avoid introducing `sorryAx`.

## Phase 1: Freeze and Verify the Dependency Boundary

1. Confirm that ComplexityReduction and the target project have the same lean-toolchain.
2. Confirm that ComplexityReduction and the target project have the same mathlib version.
3. Maintain a stable Archon import surface in ComplexityReduction; it is recommended to cover:
   - `Core`
   - `SAT`
   - `CSP`
   - `Schaefer`
   - `Karp21`
   - `Bridges`
   - `Certificates`
   - `Targets`
4. Verify:
   - `lake build complexityreduction`
   - a minimal smoke file that runs `#check` on the key public symbols
   - recording which `Targets` are available by default and which need explicit opt-in.

## Phase 2: Changes on the Archon Source Side

The goal is to solidify ComplexityReduction into Archon's packaged data as first-class knowledge.

### 2.1 Add a Library Reference Document

Add or maintain:

```text
src/archon/.archon-src/skills/lean4/skills/lean4/references/complexity-reduction-library.md
```

The content must include at least:

- The stable import surface.
- A symbol table of the core definitions and theorems.
- The available targets in the `Targets` directory and their trust boundaries.
- The explicit opt-in rules for the external Schaefer bridge.
- The use cases of the common templates in `Templates`.

### 2.2 Add a ComplexityReduction Build Mode

Add:

```text
src/archon/.archon-src/prover-modes/complexity-reduction-build.md
```

Keep the frontmatter consistent with the existing `mathlib-build.md`; the body must require:

- Checking first whether ComplexityReduction already has the corresponding definitions and theorems.
- Prioritizing completing goals through reductions, templates, and bridges.
- Every new declaration compiling successfully and being axiom-clean.
- Providing a decomposition that can be continued next time before stopping.

### 2.3 Add a ComplexityReduction Search Script

Add:

```text
src/archon/.archon-src/skills/lean4/lib/scripts/search_complexity_reduction.sh
```

Model the functionality on `search_mathlib.sh`, supporting:

- `name`: search by declaration name.
- `type`: search by type signature.
- `content`: search by file content.

The search root defaults to the ComplexityReduction source location and can be overridden via environment variables.

### 2.4 Extend Lake Dependency Hints

Currently [lake.py](src/archon/commands/tooling/lake.py) records mathlib only through `has_mathlib: Bool`.

A two-step approach is recommended:

1. First add `has_complexity_reduction` or an equivalent CR-specific detection.
2. Later consider generalizing to a `declared_dependencies` list that uniformly supports custom Lean dependencies.

Also update:

- The dependency status display in `ProjectLayout` and `semantic_pass`.
- The version-mismatch and missing-dependency hints in `init` or `doctor`.

## Phase 3: Integration on the Target Project Side

Use `D:\code\Lean\archon-test` as the default target project.

### 3.1 Add a Lake Local Dependency

Add a local dependency pointing to `../ComplexityReduction` in the target project's lakefile, keeping the existing mathlib unchanged.

Also confirm:

- The toolchain remains `v4.29.0`.
- The local dependency does not cause duplicate mathlib resolution.

### 3.2 Import ComplexityReduction

In the corresponding Lean files of the target project, import the stable CR entry surface, for example:

```lean
import ComplexityReduction.Targets
```

If a wider symbol surface is required, import `Core`, `SAT`, `CSP`, `Schaefer`, or `Karp21` as needed.

### 3.3 Add an In-Project Reference Document

Add:

```text
D:\code\Lean\archon-test\references\ComplexityReduction.md
```

Keep it consistent with the library reference in Archon's packaged data, and record the specific targets the target project currently needs.

### 3.4 Add Persistent User Hints

Add the following to the persistent section of the target project's `.archon/USER_HINTS.md`:

- Prefer existing ComplexityReduction declarations.
- New goals must select the corresponding descriptor from the `Targets` directory.
- The external Schaefer bridge is not used for local reasoning by default unless explicitly opted in.
- Every artifact must be axiom-clean; `sorry` is forbidden.

### 3.5 Configure a ComplexityReduction Advisor Subagent

Add:

```text
D:\code\Lean\archon-test\.archon\subagents\complexity-reduction-advisor.md
```

Example frontmatter:

```yaml
---
name: complexity-reduction-advisor
description: "Check ComplexityReduction API, target descriptors, and proof template fit."
read_only: true
mandatory: plan
---
```

Then add to `subagents.enabled` in `.archon/config.json`:

```json
"enabled": ["complexity-reduction-advisor"]
```

If you want the project to use it in every planning phase, you can additionally add it to `_recommended_plan_phase`.

## Verification Checklist

1. `lake build complexityreduction` passes.
2. `lake build ArchonTest` passes, confirming the local dependency is resolved.
3. The smoke file can `#check` the key ComplexityReduction symbols.
4. `archon init` or `archon doctor` discovers the new packaged data.
5. `.archon/prover-modes/complexity-reduction-build.md` is copied into the target project.
6. `complexity-reduction-advisor` is loaded normally by the Registry.
7. Put a minimal goal in `PROGRESS.md` and run one round of `archon loop` in the `complexity-reduction-build` mode.
8. Finally run `#print axioms` on the newly added declarations to confirm there is no `sorryAx`.

## Items Deferred for Now

- Do not copy the ComplexityReduction source into Archon.
- Do not register ComplexityReduction as an Archon peer.
- Do not build the complete generic dependency configuration system in the first round; generalize step by step once CR-specific detection is sufficient.
