# Archon + ComplexityReduction Reduction Problem Test Plan

## Goal

Use the curated NP-completeness reduction problem set in
`problems/problems_clean.json` to test how the modified Archon (with the
CR advisor subagent, search scripts, reference docs, and prover-mode
integrated) performs on reduction proofs.

## Route Choice: Route A (sorry stub + standard loop + CR advisor)

### Why Route A

| Dimension | Route A | Route B (complexity-reduction-build) |
|------|--------|-------------------------------------|
| What is tested | Whether CR knowledge helps the standard proof pipeline | The CR-build prover-mode itself |
| Preprocessing | Requires writing a blueprint + generating stubs | Only needs PROGRESS.md describing the target |
| Proof target | Fill in sorry stubs (close existing declarations) | Build a CostedReduction certificate from scratch |
| Relation to CR | Advisor assists in finding existing encodings/templates | The mode itself is building CR infrastructure |
| Applicable scenario | Verify the integration effect (this round's goal) | Verify the build mode's from-scratch capability |

Route A tests "whether CR knowledge injection improves the prover's
performance", which is exactly the core goal of our three-phase integration
work.

### Why Route B Is Not Chosen (This Round)

The `complexity-reduction-build` mode requires producing no `sorry`,
because it builds reduction infrastructure (`CostedReduction` certificates)
rather than closing existing stubs.
A `CostedReduction` containing a `sorry` is meaningless as a certificate.
That mode tests from-scratch building capability, which is not fully aligned
with this round's goal of "testing the CR integration effect".

---

## Full Workflow

```
problems_clean.json (natural-language problems)
        |
        v
  1. Problem selection (3-5 problems, mixed difficulty)
        |
        v
  2. Write blueprint .tex chapters (semi-formal LaTeX + \lean{} annotations)
        |
        v
  3. Archon formalize mode -> generate Lean sorry stubs (compiles)
        |
        v
  4. Archon prove mode + CR advisor -> fill in proofs
        |
        v
  5. Check dispatch.jsonl / task_results to confirm the CR infrastructure took effect
```

### Step 1: Problem Selection

Pick 3-5 problems from `problems_clean.json` (63 problems), preferring:
- Problems for which the CR library already has related encodings (3SAT family)
- Mixed difficulty (easy / medium / hard)
- Problems with a complete solution (making correctness easy to verify)

**Initial shortlist:**

| ID | Reduction | Difficulty | Has solution | Reason for selection |
|----|------|------|------|----------|
| 01-001 | SAT -> DOUBLE-SAT | easy | yes | Simplest warm-up problem; CR has a SAT encoding |
| 02-004 | 3SAT -> NAE-3SAT | hard | yes | Classic reduction; CR has a 3SAT encoding |
| 06-017 | CLIQUE -> INDEPENDENT SET | -- | yes | Graph-theoretic reduction with a concise solution (complement graph) |

Extensible candidates:
- 06-018: INDEPENDENT SET -> VERTEX COVER (forms a chain with 06-017)
- 02-007: VERTEX COVER -> SET COVER (classic covering problem)

### Step 2: Write blueprint .tex

Write one blueprint chapter (LaTeX) per problem, containing:
- `\section{...}` chapter heading
- `\begin{theorem}...\end{theorem}` theorem block with a `\lean{declName}`
  annotation inside
- `\begin{proof}...\end{proof}` proof draft

File naming convention: `blueprint/src/chapters/ReductionTest_<Name>.tex`
(corresponding to the Lean file `ReductionTest/<Name>.lean`)

The blueprint needs to be aware of the CR library's type system:
- `EncodedDecisionProblem` -- the basic decision-problem type
- `CostedReduction A B` -- reduction certificate (with `f`, `correct`, `costed`)
- `SAT.threeSATDecisionProblem` -- the existing 3SAT encoding

### Step 3: Generate Lean sorry stubs

Two options:

**Option 1 (recommended): hand it to the Archon formalize mode**

Archon's `formalize` prover-mode is designed exactly for this:
- Reads blueprint `.tex` chapters
- Generates Lean declarations with `sorry` for every theorem/lemma/definition
  block
- Verifies the file compiles

Simply set the stage to `autoformalize` in PROGRESS.md, and the formalize
mode will generate stubs from the blueprint automatically.

**Option 2: write stubs manually**

Write declaration signatures + `sorry` directly in `.lean` files.

Option 1 is recommended because it also tests Archon's formalize capability.

### Step 4: Run Archon prove + CR advisor

In `.archon/config.json`:
- `subagents.enabled: ["complexity-reduction-advisor"]`
- `harness: "codex"` (or the user's preferred harness)
- The prover stage automatically uses the `prove` mode

Run command:
```powershell
archon loop .
```

The CR advisor is dispatched during the plan stage (see "CR infrastructure
verification" below), checking the CR library for reusable
encodings/templates/theorems and producing a report for the planner to
consult.

### Step 5: Verify whether the CR infrastructure took effect

See "CR Infrastructure Verification Plan" below for details.

---

## Project Structure

```
reduction-test/
|-- lakefile.lean              # require ComplexityReduction from "../ComplexityReduction"
|-- lean-toolchain             # leanprover/lean4:v4.29.0
|-- ReductionTest.lean         # root module import
|-- ReductionTest/
|   |-- DoubleSAT.lean         # SAT -> DOUBLE-SAT reduction
|   |-- NAE3SAT.lean           # 3SAT -> NAE-3SAT reduction
|   `-- IndependentSet.lean    # CLIQUE -> INDEPENDENT SET reduction
|-- blueprint/
|   `-- src/
|       |-- content.tex        # \input of each chapter
|       |-- chapters/
|       |   |-- ReductionTest_DoubleSAT.tex
|       |   |-- ReductionTest_NAE3SAT.tex
|       `-- ReductionTest_IndependentSet.tex
|       `-- macros/
|           `-- common.tex
|-- references/
|   `-- ComplexityReduction.md # CR library reference doc (copied from archon-test)
|-- .archon/
|   |-- config.json            # harness: codex, CR advisor enabled
|   |-- subagents/
|   |   `-- complexity-reduction-advisor.md
|   |-- prover-modes/
|   |   |-- prove.md
|   |   `-- formalize.md
|   |-- AGENTS.md
|   |-- PROGRESS.md
|   `-- ...
`-- scripts/                   # temp scripts (per AGENTS.md convention)
```

---

## CR Infrastructure Verification Plan

### Signal 1: dispatch.jsonl (strongest evidence)

Each time a subagent is dispatched, `Subagent.run()` appends JSON lines to
`.archon/logs/iter-NNN/dispatch.jsonl`:

```json
{"event":"dispatch_start","role":"complexity-reduction-advisor","slug":"...","write_domain":[...]}
{"event":"dispatch_end","role":"complexity-reduction-advisor","slug":"...","ok":true,"duration_s":42}
```

How to check:
```powershell
Get-Content .archon\logs\iter-001\dispatch.jsonl |
  Where-Object { $_ -match "complexity-reduction-advisor" }
```

If records appear here, the planner did dispatch the CR advisor.

### Signal 2: task_results report (content evidence)

The advisor's actual output lives in
`.archon/task_results/complexity-reduction-advisor-<slug>.md`,
containing the verdict (REUSE / PARTIAL / BUILD_NEW) and concrete findings.

### Signal 3: persistent advice

`.archon/advice/<slug>.md`, persisted across iterations, for later iterations
to consult.

### Signal 4: mandatory audit (enforced safeguard)

The frontmatter of `complexity-reduction-advisor.md` currently does **not**
have a `mandatory` field, so the audit will not warn about its absence.

**Suggested improvement**: add `mandatory: ["plan"]` to the advisor
frontmatter, so `check_mandatory_dispatched` prints a warning in
`dashboard.log` and requires the planner to provide a skip reason in the
`## Subagent skips` section of `iter/iter-NNN/plan.md`.

### Signal 5: plan-side references

Check whether `iter/iter-NNN/plan.md` references the advisor's findings.

### Actual Verification Procedure

After each iteration:
1. Check `dispatch.jsonl` -- confirm the advisor was dispatched
2. Check `task_results/` -- confirm the advisor produced meaningful content
3. Check `plan.md` -- confirm the planner consumed the advisor's report
4. (Optional) check the `prove` `task_results` -- confirm the prover used CR
   encodings

---

## Generating Lean stubs from natural language: can it be delegated to Archon?

Yes. Archon's `formalize` prover-mode is designed exactly for this.

### formalize Mode Workflow

1. Read blueprint `.tex` chapters (with `\lean{}` annotations)
2. Generate Lean declarations with `sorry` for every `\begin{theorem}` /
   `\begin{lemma}` / `\begin{definition}` block
3. Verify the file compiles (all sorries in place)
4. Output to `task_results/<file>.md`

### Division of Labor

| Step | Who | Output |
|------|------|------|
| Problem selection + writing blueprint .tex | Us | LaTeX chapters with `\lean{declName}` annotations |
| Generating Lean sorry stubs | Archon `formalize` mode | `.lean` files that compile |
| Filling in proofs | Archon `prove` mode + CR advisor | Complete proofs |

### Prerequisites

- The problem descriptions in the blueprint need to be aware of the CR
  library's type system (`CostedReduction`, `EncodedDecisionProblem`, etc.)
- The `\lean{}` annotations need to give declaration names (naming decided by
  us)

### Additional Resources

The `lean-scaffolder` subagent is similar in function but is dispatched
proactively by the planner, injecting strategy comments
(`/- Planner strategy: ... -/`).
It can serve as a supplement.

---

## To-Do Items

- [ ] Create the `reduction-test` project (lakefile.lean, ReductionTest.lean, lean-toolchain)
- [ ] Write blueprint .tex chapters for the 3 problems
- [ ] Configure `.archon/` (config.json, subagents, prover-modes, AGENTS.md)
- [ ] Copy references/ComplexityReduction.md
- [ ] Add `mandatory: ["plan"]` to the CR advisor frontmatter
- [ ] Run `archon init .` to initialize project state
- [ ] Run `archon loop .` to start testing
- [ ] After iterations, check the CR infrastructure verification signals
