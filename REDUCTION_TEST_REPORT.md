# Archon + ComplexityReduction Reduction Problem Test Report

Status: Completed
Date: 2026-08-23
Branch: `complexity-reduction-integration` (Archon repository)

## 1. Test Objectives

Using the curated set of NP-completeness reduction problems in `problems/problems_clean.json`,
we tested how the modified Archon (integrating the CR advisor subagent, search scripts, reference
documents, and prover-mode) performs on reduction proofs. The core question: **Does CR knowledge injection improve the prover's autonomous proving ability?**

Test route: Route A (sorry stub + standard loop + CR advisor); see
[REDUCTION_TEST_PLAN.md](REDUCTION_TEST_PLAN.md) for details.

## 2. Problem Selection

| ID | Reduction | Difficulty | Reason for Selection |
|----|------|------|----------|
| 01-001 | SAT -> DOUBLE-SAT | easy | Simplest entry-level problem; CR has a SAT encoding |
| 02-004 | 3SAT -> NAE-3SAT | hard | Classic reduction; CR has a 3SAT encoding |
| 06-017 | CLIQUE -> INDEPENDENT SET | medium | Graph-theoretic reduction; concise solution (complement graph) |

The three problems cover the three typical reuse scenarios of the CR library: full reuse, building from scratch, and partial reuse.

## 3. Test Workflow

````
problems_clean.json (problems stated in natural language)
  -> select 3 problems
  -> write blueprint .tex sections (semi-formal LaTeX + \lean{} annotations)
  -> Archon formalize mode -> generate Lean sorry stubs (completed at iter-014)
  -> Archon prove mode + CR advisor -> fill in proofs (iter-015~018)
  -> Archon polish mode -> golf optimization + cleanup (iter-020~021)
  -> lake build green, 0 sorries, 0 extra axioms
````

- Project path: `reduction-test/`
- Harness: codex (Codex CLI subprocess)
- Total iterations: 21 rounds (iter-001 ~ iter-021)

## 4. CR Advisor Validation (Core Test Objective)

The CR advisor was dispatched 5 times in total across iter-001 and iter-004, generating 3 detailed reports.
The reports are archived in `logs/iter-006/task_results-archive/`.

### 4.1 Dispatch Records

| Iteration | slug | Duration | Status |
|------|------|------|------|
| iter-001 | nae3sat | 2624s | ok |
| iter-001 | double-sat | (timed out, not recorded) | timeout |
| iter-001 | independent-set | (timed out, not recorded) | timeout |
| iter-004 | independentset | 125s | fail (first attempt) |
| iter-004 | independentset | 2483s | ok (retry) |
| iter-004 | doublesat | (timed out, not recorded) | timeout |

Of the 3 concurrent dispatches in iter-001, only nae3sat completed within 44 minutes;
double-sat and independent-set did not return due to vendor network timeouts.
After redispatch in iter-004, independentset succeeded (after 1 retry), and
doublesat eventually completed its report as well.

### 4.2 Advisor Verdicts vs. Final Proofs

| Problem | Advisor Verdict | Key Recommendations | Adopted in Final Proof? |
|------|-------------|----------|------------------|
| NAE3SAT | **REUSE** | Directly re-export `threeSATToNAE3CostedReduction` | Yes; the proof is a single line `exact (…satisfiable_iff).symm` |
| DoubleSAT | **BUILD_NEW** | Custom `EncodedDecisionProblem` mapping `phi ++ [[pos y, neg y]]`, using `satisfies_append` + `clause_vars_lt_varBound` | Yes; all APIs were adopted |
| IndependentSet | **PARTIAL** | Reuse CR for CLIQUE, build IS from scratch; `complementGraph` from `Karp21.VertexCover`, plus `of_emptyOutputEncoding` | Yes; the proof uses `hasUndirectedEdge_complementGraph_iff` + `of_emptyOutputEncoding` |

**Conclusion: CR advisor knowledge injection is effective.** The final proof strategies for the three
problems match the advisor reports exactly. The advisor accurately distinguished the three reuse levels
(REUSE / BUILD_NEW / PARTIAL), and the prover chose CR library APIs correctly under the advisor's guidance, avoiding blind searching.

### 4.3 Example Advisor Report Content

Taking IndependentSet (PARTIAL) as an example, the advisor report contains:

- Full location of the source problem CLIQUE in CR (file, line numbers, type signature)
- The target problem INDEPENDENT SET does not exist in CR (0 matches in a global search)
- Reusable infrastructure: `complementGraph`, `complementEdges`, `hasUndirectedEdge_complementGraph_iff`
- Construction path: `CostedReduction.ofCostedMap` + `CostedMap.of_emptyOutputEncoding`
- Comparison with proven reductions: simpler than the CLIQUE->VC reduction in `Karp21.VertexCover` (no budget transformation)

## 5. Proof Results

### 5.1 Overview

| File | sorries | Completed in Iteration | Axioms | lake build |
|------|---------|----------|------|------------|
| NAE3SAT.lean | 0 | iter-015 | standard three axioms | green |
| DoubleSAT.lean | 0 | iter-017 | standard three axioms | green |
| IndependentSet.lean | 0 | iter-018 | standard three axioms | green |

Standard three axioms: `propext`, `Classical.choice`, `Quot.sound` (no `sorryAx`, `admit`).
`lake build` full build with 8403 jobs, exit 0.

### 5.2 NAE3SAT.lean — 3SAT -> NAE-3SAT (REUSE)

The simplest case. The CR library `CSP.Hardness.NAE3SAT` already contains the complete `threeSATToNAE3CostedReduction`
and `threeCNFToNAE3Formula_satisfiable_iff`. The prover only had to re-export them and close the proof with a single-line `exact`.

```lean
theorem threeSATToNAE3SAT_correct (phi : SAT.ThreeCNF) :
    SAT.threeSATDecisionProblem.isYes phi <->
    nae3SATDecisionProblem.isYes (CSP.Hardness.NAE3SAT.threeCNFToNAE3Formula phi) := by
  exact (CSP.Hardness.NAE3SAT.threeCNFToNAE3Formula_satisfiable_iff phi).symm
```

Completed: iter-015, 2 sessions (28 + 103 turns).

### 5.3 DoubleSAT.lean — SAT -> DOUBLE-SAT (BUILD_NEW)

Medium difficulty. The CR library has no DOUBLE-SAT encoding, so it had to be built from scratch.
Reduction map: append the always-true clause `[pos y, neg y]` after the CNF (`y = CNF.varBound phi` is a fresh variable).

The proof has two parts:
- **Correctness** (`satToDoubleSAT_correct`, ~80 lines): the forward direction uses `Clause.setVar` to produce two distinct assignments on the new variable; the backward direction extracts the satisfiability of the original formula from `satisfies_append`.
- **Size bound** (`LinearSizeBound.intro_with 3 9`): the appended clause contributes `2y + 9`, with `y <= inputSize(phi)`, for a total of `<= 3*inputSize + 9`.

Key turning point: iter-016 got stuck on the wrong size constants `2 4` (placeholder values from the iter-015 stub).
In iter-017 the planner diagnosed the correct constants `3 9` via the encoding size computation, and the prover then closed the proof.

Completed: iter-017, 5 sessions (155 + 60 + 98 + ... turns).

### 5.4 IndependentSet.lean — CLIQUE -> IS (PARTIAL)

The hardest case. CLIQUE is already encoded in the CR library; IS had to be built from scratch.
Reduction map: take the complement graph (`complementGraph`) and keep the budget `k`.

This problem went through the most twists and turns:

| Iteration | Mode | Sessions | Turns | File Changes | Result |
|------|------|----------|-------|----------|------|
| iter-015 | prove | 3 | 175 | 0 | Stuck: the prover derived the complete proof but never wrote it to a file |
| iter-016 | fine-grained | 3 | 137 | 0 | Stuck: pure file reading, exited on idle_timeout |
| iter-017 | fine-grained | 3 | — | 0 | Stuck: backend 0-token issue |
| iter-018 | complexity-reduction-build | 1 | 184 | 5 | Solved |

The iter-015 prover derived the complete proof inside an 86K-character thinking block but never called
`apply_patch` to write it to a file — a textbook case of "figured it out but never wrote it down".

After switching to the `complexity-reduction-build` mode in iter-018, the problem was completed in
a single session. That mode requires emitting no sorries and forces bottom-up assembly of the CR
certificate; the prover was forced, at every step, to either write compilable code or write nothing.

Key proof pattern:

```lean
theorem cliqueToIndependentSet_correct (I : CliqueInput) :
    Clique I <-> IndependentSet (cliqueToIndependentSetMap I) := by
  unfold Clique IndependentSet PairwiseAdjacent cliqueToIndependentSetMap
  constructor
  · -- Forward: reuse the clique's witness directly; adjacency -> non-adjacency in the complement graph
    rintro (vs, hlen, hnodup, hbounds, hadj)
    refine (vs, hlen, hnodup, hbounds, ?_)
    rintro u hu v hv huv
    rw [Karp21.VertexCover.hasUndirectedEdge_complementGraph_iff]
    simp [hadj u hu v hv huv]
  · -- Backward: by_contra; non-adjacency in the complement graph -> contradiction with adjacency
    rintro (vs, hlen, hnodup, hbounds, hnonadj)
    refine (vs, hlen, hnodup, hbounds, ?_)
    rintro u hu v hv huv
    by_contra h
    apply hnonadj u hu v hv huv
    rw [Karp21.VertexCover.hasUndirectedEdge_complementGraph_iff]
    exact (hbounds u hu, hbounds v hv, huv, h)
```

## 6. Issues Found and Fixes During Testing

### 6.1 Archon Source Fixes (Committed)

| # | Issue | Impact | Fix | Commit |
|---|------|------|------|--------|
| Fix A | Mode files such as `fine-grained.md` were not copied by `archon init` | The prover received an empty system prompt | `runners.py` falls back to `prove.md` + the 5 files completed manually | `5a6bc7b` |
| Fix B | The prover only explored without writing code | IndependentSet: 7 sessions, 0 output | Added a write-first protocol to `prove.md` | `5a6bc7b` |
| Fix C | `is_complete` did not recognize `polish (complete)` | The loop could not stop automatically | `progress.py` changed to full-text matching + case-insensitive | `343f2ef` |

### 6.2 Project Configuration Fixes (Not Committed to Archon)

| Issue | Fix |
|------|------|
| `features.plugins=false` blocked `exec_command` in the codex subprocess | Removed the setting from `.archon/config.json` |
| A prompt variant referenced the nonexistent `read_file` tool | Changed to `exec_command` + shell reads |
| 5 leftover ArchonTest files misled the model | Rewritten to reflect the actual reduction-test state |
| `.archon/prover-modes/` had only 2 files (7 expected) | Manually copied the 5 missing ones from the Archon source |

### 6.3 Non-Archon Issues

| Issue | Description |
|------|------|
| Vendor network outage (iters 7-13) | Unstable backend API caused multiple idle rounds |
| `sync_leanok` Windows false-negative | Every round deleted all `\leanok` markers, requiring manual restoration |
| CRLF line endings | DoubleSAT.lean had CRLF; apply_patch failed on CRLF files |
| Wrong DoubleSAT size constants `2 4` | Stub placeholder values generated in the formalize phase; the planner corrected them to `3 9` in iter-017 |

## 7. Iteration Timeline

| Iteration | Phase | Key Events |
|------|------|----------|
| 001-004 | autoformalize | CR advisor dispatches (5), 3 reports generated |
| 005-006 | autoformalize | formalize continued |
| 007-013 | (idle) | Vendor network outage |
| 014 | autoformalize | formalize completed: 14 declarations, 6 sorries |
| 015 | prover | First round: NAE3SAT solved, DoubleSAT partial, IS stuck (0 output) |
| 016 | prover | DoubleSAT stuck on the wrong constants `2 4`; IS still stuck |
| 017 | prover | Planner corrected the constants to `3 9`, DoubleSAT solved; IS still failed after switching to fine-grained |
| 018 | prover | IS switched to the `complexity-reduction-build` mode and was solved in 1 session |
| 019 | (idle) | — |
| 020 | polish | Golf optimization + cleanup of scaffolding comments + fixed the duplicate IS declaration |
| 021 | polish | Coverage debt cleanup; project marked COMPLETE |

## 8. Statistics

- Total iterations: 21
- Productive proof iterations: 4 (iter-015, 016, 017, 018)
- Total sorries: 6 -> 0
- Final `lake build`: 8403 jobs, exit 0
- Axioms: only `propext`, `Classical.choice`, `Quot.sound`
- CR advisor dispatches: 5 (iter-001 x 3, iter-004 x 2), 3 reports succeeded
- Archon source fixes: 3 (Fix A/B/C), 2 commits

## 9. Conclusions

1. **CR advisor knowledge injection is effective.** The final proof strategies for all three problems
   match the advisor reports exactly; the advisor accurately distinguished the three reuse levels
   REUSE / BUILD_NEW / PARTIAL, and the prover used CR library APIs correctly under the advisor's guidance.

2. **The `complexity-reduction-build` mode is effective in hard scenarios.** IndependentSet produced
   0 output across 7 sessions under the default `prove` and `fine-grained` modes; after switching to
   `complexity-reduction-build`, it was completed in 1 session. The mode's "no sorry output" constraint
   forced the prover to assemble a compilable CR certificate step by step.

3. **The write-first problem deserves attention.** The prover exhibits a "figured it out but never
   wrote it down" behavior pattern — deriving a complete proof inside a thinking block without calling
   `apply_patch`. The write-first protocol (Fix B) constrained this behavior with hard rules, but a more
   systematic solution may require enforcing apply_patch calls at the runner level.

4. **Archon's automatic completion detection was flawed.** The original `is_complete` implementation
   only checked the first word of the stage and failed to recognize non-standard formats such as
   `polish (complete)`, causing the loop to keep idling after the project was complete. Fixed by Fix C.

5. **Windows compatibility is a persistent problem.** Issues such as the `sync_leanok` false-negative,
   CRLF line endings, and `apply_patch` failing on CRLF files kept recurring; adding Windows CI is recommended.

6. **Statement fidelity has a known gap.** The theorems proven in this round capture the existence
   and correctness of each reduction (the yes-instance iff), but not the polynomial-time
   computability of the reduction maps. See Section 10.

## 10. Known Limitation: Statement Fidelity Gap (Polynomial-Time Obligation Not Formalized)

The natural-language claims tested in this round ("X is NP-complete via reduction from Y") carry
three obligations: (i) an instance encoding, (ii) correctness (yes-instances correspond if and
only if), and (iii) the reduction map is polynomial-time computable. The Lean theorems produced
by the loop cover only (i) and (ii):

- `DoubleSAT.lean` proves the iff between `SAT.CNF.Satisfiable phi` and
  `doubleSATDecisionProblem.isYes (satToDoubleSATMap phi)`. The map `satToDoubleSATMap` is a
  plain `def` (computable in Lean's definitional sense), with no statement about its time
  complexity.
- `NAE3SAT.lean` re-exports a `CostedReduction` from the ComplexityReduction library, but the
  theorem proven here (`threeSATToNAE3SAT_correct`) is again only the bare iff; the cost part
  rides on the library's existing result rather than being an obligation discharged by this test.
- `IndependentSet.lean` proves the complement-graph iff, and additionally assembles a
  `CostedReduction` via `CostedMap.of_emptyOutputEncoding`. That constructor charges cost 0 by
  requiring the output encoding to be always empty, and the ComplexityReduction docstring states
  explicitly that it "is not a faithful natural-language encoding theorem by itself" — the cost
  certificate is vacuous.

In other words, every formal theorem produced in this round is **strictly weaker** than its
natural-language original: a proof that compiles with zero `sorry` and is axiom-clean still does
not establish NP-completeness as claimed, because the polynomial-time obligation was silently
dropped during formalization. This is a concrete instance of the "compiling does not imply
semantic fidelity" problem (statement-level semantic weakening) known from the autoformalization
literature: LLM judges are insensitive to this kind of loosening, and no stage of the current
pipeline encoded the obligations as verification targets.

Until the cost obligation is formalized — e.g., by making a `CostedReduction` with a genuine cost
certificate (such as `CostedMap.of_encodedPolynomialSizeBound`), or an equivalent polynomial-time
computability statement, the mandatory proof target instead of the bare iff — the results of this
round should be read as "reduction existence and correctness, formally verified", not as complete
formal NP-completeness proofs. Follow-up work: extract an obligation checklist (encoding / iff /
computability / cost bound) from each natural-language problem statement, make it a hard
constraint of the prove loop, and add automated loosening detection (e.g., implication-strength
checks against Lean).
