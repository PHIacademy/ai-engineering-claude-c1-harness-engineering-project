# Capstone — Harness Engineering with Claude and Claude Code

Submission for the Harness Engineering capstone: four completed reference systems from the course, each built, run, and verified end-to-end, plus an evidence-grounded reflection brief.

## The four systems

| # | System | Run with | Tests passing |
|---|--------|----------|----------------|
| 1 | Insurance Claims Intake Agent — a `stop_reason`-driven agentic loop with structured tool use | `python -m claims_intake.run --all` | 29 |
| 2 | Retail Support Context Strategy — reduces token load while preserving answerability | `python -m retail_context.run --all` | 30 |
| 3 | E-Commerce Team Claude Code Config — CLAUDE.md hierarchy, path-scoped rules, commands, skills | `python -m ecommerce_team_config .` | 35 |
| 4 | Multi-Shift Quality Monitoring — Layer 3 orchestration with tiered state, crash recovery, forking | `python -m shift_monitor run-shift ...` | 33 |

## Repository structure

```
.
├── README.md                       ← you are here
├── reflection-brief.md             ← evidence-grounded reflection brief
├── system-1-claims-intake/
│   ├── test-output.txt             ← pytest run (29 passed)
│   ├── summary.md                  ← claim outcomes table
│   └── traces/                     ← per-claim stop_reason traces
├── system-2-retail-context/
│   ├── test-output.txt             ← pytest run (30 passed)
│   ├── budget.json                 ← token accounting / reduction %
│   ├── eval.jsonl                  ← evaluation results
│   └── eval_control.jsonl          ← control (facts block stripped)
├── system-3-ecommerce-config/
│   ├── test-output.txt             ← pytest run (35 passed)
│   ├── validator-output.txt        ← OK / exit 0
│   └── claude-config/              ← .claude/ structure (rules, commands, skills) + root CLAUDE.md (@import hierarchy)
└── system-4-shift-monitoring/
    ├── test-output.txt             ← pytest run (33 passed)
    ├── shift-output.txt            ← shift run log
    ├── hot_state.json              ← bounded hot-tier state
    └── shift_scratchpad.jsonl      ← append-only shift log
```

## Reflection brief

The full evidence-grounded reflection, covering per-system architecture, cross-system synthesis, and an honest assessment of what broke and what I'd change, is here: [`reflection-brief.md`](./reflection-brief.md)
