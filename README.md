# Archon + ComplexityReduction Project Documentation

This repository collects the planning and reporting documents of the project integrating the Archon automated theorem proving framework with the ComplexityReduction Lean library.

## Document Index

The recommended reading order is:

1. COMPLEXITY_REDUCTION_INTEGRATION_PLAN.md — the overall three-phase integration plan, describing the goals, key conventions, and verification checklist
2. PHASE1_REPORT.md — Phase 1: freezing the ComplexityReduction dependency boundary, verifying the build and import surface
3. PHASE2_REPORT.md — Phase 2: Archon source-side changes (library reference, search script, prover mode, Lake dependency detection)
4. PHASE3_REPORT.md — Phase 3: integration on the target project side (Lake dependency, smoke import, subagent configuration)
5. REDUCTION_TEST_PLAN.md — the reduction test plan, problem selection and route choice
6. REDUCTION_TEST_REPORT.md — the reduction test result report, CR advisor validation and the complete proof processes for 3 problems

Chinese versions of all documents are available as `<name>.zh-CN.md` alongside each English file.

## Workspace Repository Navigation

The workspace covered by these documents contains the following independent git repositories, all of which have been uploaded to GitHub:

- [ComplexityReduction](https://github.com/Archon4CR/ComplexityReduction) — a Lean 4 computational complexity reduction library providing the formalization infrastructure for NPC reductions (EncodedDecisionProblem, CostedReduction, Karp21, Schaefer, etc.); **developed by a senior member of our lab**
- [Archon](https://github.com/Archon4CR/Archon/tree/complexity-reduction-integration) — a Python automated theorem proving framework driving the iterative Lean formalization workflow (blueprint -> formalize -> prove -> polish); a **fork of the upstream frenzymath/Archon with targeted modifications** for the ComplexityReduction integration
- [problems](https://github.com/Archon4CR/problems) — an NP-completeness reduction problem set of 64 reduction problems extracted and cleaned from university course materials; **compiled and curated entirely by the author of this project**
- [reduction-test](https://github.com/Archon4CR/test) — a reduction proof testing project containing the complete Lean proofs for 3 problems and CR advisor reports; **built by the project author** on top of Archon + ComplexityReduction
- [docs](https://github.com/Archon4CR/docs) — this repository, the project documentation
