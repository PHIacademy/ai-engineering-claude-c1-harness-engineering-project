# Project: Harness Engineering with Claude and Claude Code

Submission for the Harness Engineering capstone: four completed reference systems from the course, each built, run, and verified end-to-end, plus an evidence-grounded reflection brief.

## The four systems

| # | System | Run with | Tests passing |
|---|--------|----------|----------------|
| 1 | Insurance Claims Intake Agent — a `stop_reason`-driven agentic loop with structured tool use | `python -m claims_intake.run --all` | 29 |
| 2 | Retail Support Context Strategy — reduces token load while preserving answerability | `python -m retail_context.run --all` | 30 |
| 3 | E-Commerce Team Claude Code Config — CLAUDE.md hierarchy, path-scoped rules, commands, skills | `python -m ecommerce_team_config .` | 35 |
| 4 | Multi-Shift Quality Monitoring — Layer 3 orchestration with tiered state, crash recovery, forking | `python -m shift_monitor run-shift ...` | 33 |

## Repository structure

- [`README.md`](./README.md) — you are here
- [`reflection-brief.md`](./reflection-brief.md) — evidence-grounded reflection brief
- **`system-1-claims-intake/`**
  - [`summary.md`](./system-1-claims-intake/summary.md) — claim outcomes table
  - [`system1-tests.png`](./system-1-claims-intake/system1-tests.png) — pytest run (29 passed)
  - [`traces_claim_02_stolen_bike.jsonl`](./system-1-claims-intake/traces_claim_02_stolen_bike.jsonl) — per-claim stop_reason trace
- **`system-2-retail-context/`**
  - [`budget.json`](./system-2-retail-context/budget.json) — token accounting / reduction %
  - [`context.md`](./system-2-retail-context/context.md) — assembled context artifact
  - [`case_facts_call.json`](./system-2-retail-context/case_facts_call.json) — structured case-facts extraction call
  - [`eval.jsonl`](./system-2-retail-context/eval.jsonl) — evaluation results
  - [`eval_control.jsonl`](./system-2-retail-context/eval_control.jsonl) — control (facts block stripped)
  - [`system2-tests.png`](./system-2-retail-context/system2-tests.png) — pytest run (30 passed)
- **`system-3-ecommerce-config/`**
  - [`CLAUDE.md`](./system-3-ecommerce-config/CLAUDE.md) — root config, @import hierarchy
  - [`claude_dir_structure.txt`](./system-3-ecommerce-config/claude_dir_structure.txt) — .claude/ structure (rules, commands, skills)
  - [`system3-tests.png`](./system-3-ecommerce-config/system3-tests.png) — pytest run (35 passed)
  - [`system3-validator-output.png`](./system-3-ecommerce-config/system3-validator-output.png) — OK / exit 0
- **`system-4-shift-monitoring/`**
  - [`hot_state.json`](./system-4-shift-monitoring/hot_state.json) — bounded hot-tier state
  - [`shift_scratchpad.jsonl`](./system-4-shift-monitoring/shift_scratchpad.jsonl) — append-only shift log
  - [`system4-output.png`](./system-4-shift-monitoring/system4-output.png) — shift run log
  - [`system4-tests.png`](./system-4-shift-monitoring/system4-tests.png) — pytest run (33 passed)

## Reflection brief

The full evidence-grounded reflection, covering per-system architecture, cross-system synthesis, and an honest assessment of what broke and what I'd change, is here: [`reflection-brief.md`](./reflection-brief.md)
