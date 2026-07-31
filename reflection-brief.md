# Reflection Brief — Harness Engineering Capstone

**Name:** Lo Kai Cheung, Stanley
**Date:** 31/07/2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Windows 11 / Python 3.12.0
- Approx. API spend: $0.3 USD total

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.**
   → In my trace for `claim_02_stolen_bike`, the `stop_reason` sequence is: `tool_use → tool_use → tool_use → tool_use → end_turn` (`claim_02_stolen_bike.jsonl`). The continue-vs-stop decision is implemented in `run()` in `loop.py`. Concretely, it returns only when `response.stop_reason == "end_turn"` and continues only when `response.stop_reason == "tool_use"` (`loop.py`). Any other value raises `UnexpectedStopReason`, which prevents silent "half-finished" loops and forces the run to fail loudly (`loop.py`).

2. **Anti-pattern.**
   → One anti-pattern `test_antipatterns.py` checks for is using string-membership tests against assistant text to drive control flow (`test_no_string_membership_against_text_in_loop`, which flags `"token" in text` patterns in `loop.py`). If my loop relied on this instead of `stop_reason`, it would be vulnerable to the free text the model actually produces mid-run — e.g., turn 3's rationale ("The claimant explicitly reports a bike stolen...") and turn 4's `claim_summary` in `route_to_adjuster`. A keyword match against either could misfire on unexpected wording, stopping the loop before `route_to_adjuster` fires or failing to stop at all — leaving the claim silently incomplete instead of ending on a real tool boundary, which is exactly what `stop_reason`-based control prevents.

3. **Tool design.**
   → Two tools with overlapping inputs are `classify_claim` and `route_to_adjuster`: both use the same `CLAIM_TYPES` enum (`classify_claim.claim_type`, `route_to_adjuster.queue`), but their descriptions separate decision from terminal action — `classify_claim` says "call this exactly once per claim" with a confidence policy ("if confidence is below 0.6, prefer escalate_to_human"), while `route_to_adjuster` is labeled `TERMINAL TOOL` and instructs the model to stop with `end_turn` right after (`tools.py`). Both terminal tools also guard against double-calls, returning a `permanent`, non-retryable error if called twice (`tools.py`). That structured error shape — `is_error`, `error_category`, `is_retryable` — lets the agent tell a permanent mistake (fix the input, don't retry) from a transient one (retry), which a generic error string wouldn't support.

4. **Your numbers.**
   → For `claim_02_stolen_bike.jsonl`, the loop runs 5 turns (`tool_use` × 4, then `end_turn`). Summing tokens across all turns gives 17,184 input tokens and 802 output tokens. Using `pricing.py`'s rate table for `claude-haiku-4-5-20251001` (`input_per_mtok=1.0`, `output_per_mtok=5.0`), that works out to `estimate_cost_usd` ≈ (17184/1,000,000 × 1.0) + (802/1,000,000 × 5.0) = **$0.0212** for this one claim. The README doesn't give a specific per-claim turn count or cost to compare against directly — only the aggregate "$1–$5 total" across all systems — but at ~$0.02/claim on Haiku, 8 claim fixtures would land around $0.17, comfortably inside that range and on the cheap end since Haiku's rate is far lower than Sonnet's (`claude-sonnet-4-6`: $3/$15 per Mtok) or Opus's ($15/$75).

### System 2 — Context strategy

5. **The reduction.**
   → Per `budget.json`, baseline tokens were 38,708 and the assembled context came to 16,874 tokens — a 56.41% reduction. The `active` section dominates the assembled context by far (15,789 of the 16,874 tokens, ~93.5%), dwarfing `case_facts` (204), `resolved_refund` (436), and `resolved_subscription` (463) combined. That's kept verbatim because it's the live, in-progress conversation the model is actively reasoning over — summarizing it risks dropping details the model still needs to act on correctly, whereas the resolved threads (`resolved_refund`, `resolved_subscription`) are closed and safe to compress since their outcomes, not their blow-by-blow exchange, are what matter going forward.

6. **Summarize vs preserve.**
   → The rule: only closed/resolved threads get summarized, while the live, open case is kept byte-exact. `budget.json`'s `per_section_tokens` shows `resolved_refund` at 436 tokens and `resolved_subscription` at 463 tokens — each collapsed down from a much longer original exchange into an "Outcome" + "Key facts" + "Resolution" block (visible in `context.md`) — whereas `active` sits at 15,789 tokens, the full unsummarized transcript of the still-unresolved payment-method issue. `case_facts` (204 tokens) is kept as a small structured block of exact values (e.g. `payment_update_status: in_progress` per `case_facts_call.json`) rather than prose, precisely because eval Q6 later depends on quoting that exact token, not a paraphrase.

7. **Facts block.**
   → Comparing `eval.jsonl` to `eval_control.jsonl`: Q6 ("exact status token, not a paraphrase") passes in `eval.jsonl` (`in_progress`, per the case record) but fails in the control, where the model instead claims no structured status exists and describes the issue as "active and unresolved" from the raw transcript alone. This proves `case_facts` isn't redundant — it's what carries the literal machine-readable token (`payment_update_status: in_progress`, per `case_facts_call.json`); strip it out, and the model can only paraphrase, not reproduce the exact value the question demands.

### System 3 — Claude Code config

8. **Path-scoped rules.**
   → From `.claude/rules/react.md`: the glob frontmatter is `paths: ["src/components/**/*", "src/pages/**/*"]`. `CLAUDE.md` itself states the reasoning for preferring this over a directory-level `CLAUDE.md`: "we prefer `.claude/rules/` with glob `paths:` instead, so cross-cutting conventions (e.g. test files everywhere) work cleanly" (`CLAUDE.md`). Concretely, this means the same React rules apply whether you're editing `src/pages/Checkout.tsx` or `src/components/Cart.tsx`, without needing a `CLAUDE.md` duplicated in both directories (or one bloated root `CLAUDE.md` the model has to hold in context on every task regardless of relevance).

9. **Forked skill.**
   → From `SKILL.md`: `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git ls-files:*), Bash(gh pr view:*), Bash(gh pr checks:*)]`. Running forked + read-only means the verbose intermediate output — file enumeration, diff parsing, `git status` output — stays inside the fork and only the structured pass/fail summary returns to the main session; the read-only allowlist guarantees the check "never modifies files, never pushes, never deploys" even by accident. Without the fork, that same megabytes-of-output discovery process would pollute the main conversation's context on every deploy check; without the read-only allowlist, nothing would technically stop a future maintainer's prompt or model choice from having the skill run a destructive command mid-check.

10. **Scope.**
    → The validator output confirms the config passes (`OK`, per `system3-validator-output.png`). `CLAUDE.md` itself lays out the scope hierarchy explicitly: project-level (`./CLAUDE.md`, `.claude/standards/`, `.claude/rules/`) "lives in git" and is "shared with the whole team," while user-level (`~/.claude/CLAUDE.md`, `~/.claude/commands/`, `~/.claude/skills/`) is "not shared via version control — they stay on your laptop and never reach teammates" (`CLAUDE.md`). `review.md`'s Notes give a matching concrete pair: the `/review` command is project-scoped (`.claude/commands/review.md`), while a stricter personal variant belongs at `~/.claude/commands/review-strict.md` — user-level, since it "will not affect teammates."

### System 4 — Orchestration

11. **Push work down.**
    → The indexed query is `defects_since()` in `warm.py`, which filters SQL-side only (`SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`), backed by `idx_defects_ts` (and `idx_defects_shift_ts` for shift-scoped lookups) — no Python-side filtering of severity, component, or time. Per the terminal log, this run queried with `since=2026-07-30T22:45:04Z` and returned **0 new defects**, out of **40 total rows** in `warm.sqlite` (`WarmStore.count()`). The model never sees the full history because `defects_since` only surfaces rows newer than the last checkpoint (further bounded by `limit=50`) — pushing the "has anything changed" work down into an indexed SQL scan of 40 rows rather than loading the entire defect history into the model's context every single shift.

12. **Crash recovery.**
    → From `recovery.py`: `decide()` returns `"resume"` only if there are existing steps, the manifest isn't complete, and the last step's timestamp is within `STALE_RESUME_THRESHOLD_MINUTES = 30`; otherwise it returns `"fresh"`. The threshold is ~1/16 of an 8-hour shift cycle — a resume inside that window is still operating on the same shift's working set, while anything older is treated as a stale partial. A fresh start with an injected summary is more reliable beyond that window because the manifest's partial state may no longer reflect the current shift's actual data (new defects could have arrived, the warm tier could have changed) — resuming stale steps risks silently continuing from an outdated snapshot, whereas restarting fresh and re-injecting only the confirmed findings as a summary forces the system to re-verify against current state rather than trust assumptions that may no longer hold.

13. **Small state.**
    → `hot_state.json` is 643 bytes (per `ls -la`), comfortably under the README's ~5 KB budget. That budget matters because this system runs once per shift, indefinitely — if the hot tier grew unbounded (e.g. accumulating every historical defect hash rather than just `recent_defect_hashes`), every subsequent shift's run would have to load, parse, and reason over a steadily larger file, quietly increasing both latency and token cost shift after shift with no natural ceiling. Keeping it small and bounded (a short summary, a handful of active alerts, threshold statuses) means the cost of each run stays flat regardless of how many shifts have already run — the system pushes historical volume down into the warm tier (SQLite) instead, and only carries forward what's operationally relevant right now.

13b. **Fork isolation.**
    → `fork.py`'s `fork_for_hypothesis()` copies the base `hot_state.json` into an isolated `forks_root/<hypothesis_id>/hot_state.json` via `shutil.copyfile` — each fork works from its own file-system-isolated copy, never writing back to the shared state or to other forks. Findings only rejoin the main line explicitly through `merge_findings()`, which reads each fork's own scratchpad and appends its entries into `main_scratchpad` — a one-directional merge rather than shared mutable state, so two competing hypotheses can be explored in parallel without either corrupting the other's view or the main session's.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.**
    → **Model:** the reasoning that lands in `classify_claim`'s `rationale` field in `claim_02_stolen_bike.jsonl` turn 3 ("The claimant explicitly reports a bike stolen without permission...") — free-text judgment the model produces, not code.
    → **Harness:** `loop.py`'s `run()` function, which enforces `stop_reason`-driven control deterministically (`test_stop_reason_is_loop_control` in `test_antipatterns.py`) and `tools.py`'s structured error contract (`is_error`/`error_category`/`is_retryable`).
    → **Orchestration:** `pipeline.py`'s `run_shift()`, which sequences the whole shift cycle deterministically — SQL pre-filter (`gather_new_defects`), a single Claude call, then a code-enforced `_trim_to_budget()` and `write_atomic()` to `hot_state.json` — coordinating state across shifts rather than within one agentic loop.

15. **Deterministic vs prompt.**
    → **Code-guaranteed:** `pipeline.py`'s `_trim_to_budget()` enforces `HOT_STATE_BYTE_BUDGET` by popping `active_alerts` until the state fits, then `write_atomic()` commits it — a byte budget enforced structurally, not by asking the model to be concise. **Prompt-guided:** `classify_claim`'s description in `tools.py` — "if confidence is below 0.6, prefer `escalate_to_human`" — is stated in natural language, and nothing in the code stops the model from ignoring it. The budget is right where a hard limit must never be violated (unbounded state growth); the prompt policy is right where judgment benefits from model reasoning (how confident is "confident enough").

16. **Context, two faces.**
    → System 2 manages context *within* one long conversation: `budget.json` shows 38,708 baseline tokens compressed to 16,874 (56.41% reduction), keeping the `active` thread verbatim (15,789 tokens) while collapsing resolved threads to ~400–460 tokens each. System 4 manages context *across* many separate shift runs: `hot_state.json` stays at a fixed 643 bytes regardless of shift count, while the full 40-row defect history lives in `warm.sqlite`, queried on demand via `defects_since()`. Same principle in both — only what's currently actionable stays "hot" and verbatim, everything resolved/historical gets pushed down and compressed or made queryable — but the mechanism differs: System 2 summarizes in-context, System 4 externalizes to a separate store between sessions.

17. **Reliability you can't see in one run.**
    → `test_no_string_membership_against_text_in_loop` in `test_antipatterns.py` statically guarantees `loop.py` never uses string-matching on assistant text for control flow. A single successful run of `claim_02_stolen_bike` wouldn't reveal this — the trace looks identical whether the loop is driven by `stop_reason` or by a string match that happened to work for this particular claim's wording. It matters before shipping because the failure mode (misfiring on paraphrased or reworded model output) is exactly the kind of thing that passes on your test fixtures and breaks in production on real, unscripted phrasing.

18. **Blast radius.**
    → Picking System 3's `deploy-check` skill: if it misbehaved, the blast radius is capped by its `allowed-tools` allowlist (`Read`, `Grep`, `Glob`, and read-only `Bash(git ...)`/`gh pr ...` commands) — it structurally cannot push, deploy, or modify files, so a misbehaving run at worst produces a wrong verdict, not a broken deploy. The kill switch is that same allowlist plus `/review`'s stated backstop: "even if a future maintainer adds `Write` here, the team's `/review` command should catch the change" (`SKILL.md`) — enforcement lives both in what the skill is *allowed* to do and in the team's own review process for changes to that allowlist.

---

## Part 3 — Honest assessment

19. **What broke.**
    → The first real failure was `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'` when running `python -m claims_intake.run --all` in System 1. Root cause: the project pins `anthropic==0.39.0`, which was built against an older `httpx` that accepted a `proxies=` kwarg — a fresh venv installed a newer `httpx` (0.28+) that dropped it, so `Anthropic(api_key=...)` crashed inside its own `httpx.Client()` init before ever reaching the API call. Fixed with `pip install "httpx<0.28"` to pin a version compatible with the older SDK. A smaller environment issue also surfaced early: the README's `source .venv/bin/activate` assumes macOS/Linux, and on Windows (Git Bash/MINGW64) the venv layout uses `Scripts` instead, so activation needed `source .venv/Scripts/activate`.

20. **What you'd change.**
    → I'd revisit the fixed 30-minute staleness threshold in `recovery.py` (`STALE_RESUME_THRESHOLD_MINUTES = 30`). It's a single hardcoded constant, uniform across every shift regardless of what's happening operationally, justified only as "~1/16 of an 8-hour shift cycle." But that reasoning doesn't obviously generalize: a quiet shift with no active alerts could probably tolerate a much longer resume window safely, while a shift already showing `ALARM` thresholds — like my own Shift C run, where `hot_state.json` records `defect_rate_per_shift: ALARM` and `lot_defect_concentration: ALARM` — arguably shouldn't resume a stale partial at all, even within 30 minutes, since silently continuing on outdated state during an active alert is riskier than during a routine shift. I'd make the threshold scale with the current `threshold_statuses` rather than stay a flat constant — shorter tolerance when any status is `ALARM`, longer when everything is `OK`.
