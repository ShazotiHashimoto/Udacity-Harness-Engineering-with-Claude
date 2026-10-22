# Reflection Brief — Harness Engineering Capstone

**Name:** Shahzaad Tayob
**Date:** 2026-10-22

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) for Systems 1 and 2, recorded response for System 4.
- OS / Python: macOS; Python 3.14.7.
- Approx. API spend: About $0.30 estimated from captured token usage and the project’s Haiku list-price rates; System 1 reports $0.1294, and System 2 is about $0.1733. These are estimates, not billing-dashboard totals.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
    → In `system1/20261022_190009/traces/claim_04_neighbor_injury.jsonl`, the sequence is `tool_use → tool_use → tool_use → tool_use → tool_use → end_turn`. `claims_intake.loop.run` in `loop.py` continues only when `response.stop_reason == "tool_use"`, returns on `"end_turn"`, and raises on any other value. Claim 4 reached its terminal route in six turns (`system1/run.log`).

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
    → `test_antipatterns.py` checks for an integer-literal iteration cap as the loop’s primary stopping mechanism. A fixed cap could terminate before the model calls a terminal tool; claim 4 needed six turns in `system1/20261022_190009/summary.md`. The configured token and wall-clock `Budget` is the safety limit instead of an arbitrary iteration count (`system1/tests.log`).

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
    → `classify_claim` and `route_to_adjuster` both expose claim-type enums, but their descriptions distinguish classification from the terminal queue action in `claims_intake/tools.py`. Tool errors are serialized with `is_error`, `error_category`, and `is_retryable`, allowing the model to distinguish a transient failure it may retry from a permanent input problem. The successful tool-use sequence for claim 4 is recorded in `system1/20261022_190009/traces/claim_04_neighbor_injury.jsonl`.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
    → Claim 4 took 6 turns and cost an estimated `$0.0276` (`system1/20261022_190009/summary.md`). The README’s approximately `$0.05` is for the whole eight-claim run, while my full run estimated `$0.1294`. It differed because the model sometimes ended without classification or a terminal tool, so the deterministic backstop escalated four claims; claim 3 also ended without the expected clarification (`system1/20261022_190009/traces/claim_03_water_damage.jsonl`).

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
    → `system2/20261022-190202/budget.json` records a baseline of 38,708 tokens, an assembled context of 16,868, and a 56.42% reduction. The active issue dominates at 15,789 tokens, and it stays verbatim because its live troubleshooting details can still affect the next decision. The context itself is in `system2/20261022-190202/context.md`.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
    → Resolved issues are summarized because their outcomes and durable facts have stabilized; the active issue remains byte-exact because it is unresolved. My section counts were 381 tokens for the refund summary, 512 for the subscription summary, 204 for case facts, and 15,789 for active history (`system2/20261022-190202/budget.json`). The assembler’s active-byte and budget consistency checks passed after the run (`system2/tests.log`).

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
    → In `system2/20261022-190202/eval.jsonl`, Q1–Q6 all passed; in `eval_control.jsonl`, Q6 failed while Q1 still passed. Q6’s exact `in_progress` status is absent after removing the case-facts block, showing that the block carries structured information not recoverable from the narrative. The control result was 1/2 regressions (`system2/run.log`); note that Q5’s fragment-based check passed despite an answer that incorrectly claimed the refund was received (`eval.jsonl`).

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
    → The API rule uses `paths: - "src/api/**/*"` in `.claude/rules/api.md` (`system3/config-structure.txt`). Unlike a directory-level `CLAUDE.md`, glob-scoped rules can target cross-cutting conventions such as test files anywhere in the tree (`tests.md` uses `**/*.test.ts` and `**/*.test.tsx`). The validator passed (`system3/validation.log`), confirming the rule set is parseable.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
    → The frontmatter says `context: fork`; its `allowed-tools` are `Read`, `Grep`, `Glob`, `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, `Bash(gh pr view:*)`, and `Bash(gh pr checks:*)` (`system3/config-structure.txt`). Forking isolates exploratory output from the main session, while the allowlist prevents the validation task from modifying the repository. Without those controls, the skill could clutter the main context or perform unintended writes; the validator returned `OK` (`system3/validation.log`).

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → `CLAUDE.md` documents `./CLAUDE.md`, `.claude/standards/`, and `.claude/rules/` as project-level, version-controlled configuration. It gives `~/.claude/CLAUDE.md` and personal commands or skills as user-level examples that are not shared with teammates. The config validator returned `OK` (`system3/validation.log`), confirming the scope documentation is present.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm database held 40 defects, while the README-command run returned 0 for its default eight-hour cutoff (`system4/readme-command.log`). The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, using `idx_defects_ts` (`shift_monitor/warm.py`; plan and row count in `system4/query_evidence.log`). The model receives only the SQL-filtered current window plus hot state, not all historical rows; the cutoff is later than the fixture’s newest timestamp.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.py` returns `resume` only for an incomplete manifest whose latest step is no more than 30 minutes old; empty, complete, and older states start fresh. The 30-minute boundary is exercised at 30 and 31 minutes in `system4/tests.log`. A fresh run with prior findings injected as a summary avoids relying on stale partial context or expired tool results while preserving useful conclusions.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `system4/hot_state.json` is 643 bytes, below the 5,120-byte budget enforced by `HotState.write_atomic` (`system4/shift_output.log`, `shift_monitor/state.py`). The budget prevents prompt state from growing without bound across recurring shifts. A fixed ceiling makes per-shift context and storage predictable even as cold history grows.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `system2/20261022-190202/eval.jsonl` shows Haiku answering context questions, including the exact active status token. This is model behavior evaluated against the assembled prompt.
    → Harness: `claims_intake/loop.py` and `system1/20261022_190009/traces/claim_04_neighbor_injury.jsonl` show the code-controlled stop-reason loop and tool dispatch. The harness owns control flow and records each turn.
    → Orchestration: `shift_monitor/pipeline.py`, `system4/warm.sqlite`, and `system4/hot_state.json` show SQL-filtered shift invocation, durable warm state, and compact hot state. This layer coordinates repeated runs and state across shifts.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic guarantee is the atomic hot-state write with a hard 5,120-byte cap in `shift_monitor/state.py`; the 643-byte result is recorded in `system4/shift_output.log`. A prompt-guided behavior is the claims system prompt’s instruction to ask once when the type is ambiguous; claim 4 did so in `system1/20261022_190009/traces/claim_04_neighbor_injury.jsonl`. Code is right for invariants and safety boundaries, while prompts guide semantic judgment; this run shows prompts do not guarantee compliance.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages context inside one long session: it reduced 38,708 tokens to 16,868 (56.42%) while preserving the active segment (`system2/20261022-190202/budget.json`). System 4 manages context across shifts: hot state is 643 bytes, and the indexed query filters a 40-row warm store rather than passing the full history (`system4/shift_output.log`, `system4/query_evidence.log`). Both bound what reaches the model, but System 2 summarizes resolved conversation while System 4 uses tiers, SQL selection, and compact durable state.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `test_mid_write_read_reveals_prior_complete_lines` verifies that a reader sees only complete manifest entries during incremental writes (`system4/tests.log`). A single successful shift would not prove the recovery record remains usable during an interrupted write. This matters because corrupted or partial state could make a later shift resume from an invalid point.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1’s blast radius is misrouting or delaying real claims. Its terminal tools and confidence guidance constrain the intended route, while the runner’s deterministic backstop escalated four claims when the model ended without a terminal action (`system1/20261022_190009/summary.md`). The operational kill switch is to stop the run, with token and wall-clock budgets limiting runaway work (`claims_intake/budget.py`).

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Initially, the system Python lacked the test dependencies, and `.env` values were not automatically exported to child processes. I used an isolated `/tmp/harness-evidence-venv` and sourced `.env` with export enabled; `system1/environment-check.log` records only that the variables loaded, not their values. The model runs then completed, and final test outputs are in `system1/tests.log` and `system2/tests.log`.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would strengthen System 2’s evaluation oracle: Q5 was marked as passing because the answer contained “prorated,” even though it incorrectly said the customer received the refund (`system2/20261022-190202/eval.jsonl`). A fragment-only check can reward a semantically wrong answer. I would use explicit structured assertions for receipt status, including positive and negative cases, before treating the 6/6 score as a correctness result.
