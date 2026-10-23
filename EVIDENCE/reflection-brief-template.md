# Reflection Brief — Harness Engineering Capstone

**Name:** Shahzaad Tayob
**Date:** 2026-10-23

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) for Systems 1 and 2, recorded response for System 4.
- OS / Python: macOS; Python 3.14.7.
- Approx. API spend: About $0.30 estimated from captured token usage and the project’s Haiku list-price rates; System 1 reports $0.1294, and System 2 is about $0.1733. These are estimates, not billing-dashboard totals.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
    → I followed claim 4 in `s1/traces/claim_04_neighbor_injury.jsonl`: `tool_use → tool_use → tool_use → tool_use → tool_use → end_turn`. The decision is in `run()` in `Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py`. It keeps going for `tool_use`, returns at `end_turn`, and raises for anything else. The claim took six turns before it routed (`s1/summary.md`).

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
    → `test_antipatterns.py` checks that the loop does not rely on a hard-coded turn limit (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/tests/test_antipatterns.py`). Claim 4 took six turns, so a small fixed cap could have stopped it before the terminal route (`s1/summary.md`). The token and wall-clock budget is the backstop; all 30 tests passed (`s1/tests.log`).

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
    → I chose `classify_claim` and `route_to_adjuster`: both mention claim type, but one records the model's classification and the other sends the claim to a queue (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/claims_intake/tools.py`). The dispatcher returns errors with `is_error`, `error_category`, and `is_retryable`, so the model gets a useful signal about whether retrying makes sense instead of having to interpret an untyped string. Claim 4's actual tool sequence is in `s1/traces/claim_04_neighbor_injury.jsonl`.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
    → Claim 4 used 6 turns and cost about `$0.0276`; the whole run came to `$0.1294` (`s1/summary.md`). That is above the README's roughly `$0.05` estimate for all eight claims. In my run, four claims ended without the model's terminal tools and hit the escalation backstop, and claim 3 never asked the expected clarification (`s1/traces/claim_03_water_damage.jsonl`).

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
    → The active payment-method issue is still unresolved: the attempted card ending in 7782 hit `AVS_MISMATCH` (`s2/context.md`). In my run, that verbatim section takes 15,789 of the assembled 16,868 tokens (`s2/budget.json`). I kept its turns intact so the next troubleshooting response has the original details; only the refund and subscription threads are summarized.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
    → I summarized the two closed threads and kept the current payment-method thread byte-exact. The refund and subscription summaries took 381 and 512 tokens; case facts took 204, compared with 15,789 for the active conversation (`s2/budget.json`). The byte-exact check and the budget consistency check both passed in `s2/tests.log`.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
    → The full context passed all six questions; with case facts stripped, Q6 failed and Q1 still passed (`s2/eval.jsonl`, `s2/eval_control.jsonl`). That is the result I wanted for Q6: the exact `in_progress` token only appears in the facts block, while the refund amount can still be found in the narrative. One wrinkle: Q5 was counted as a pass because the answer included “prorated,” even though it incorrectly said the customer received the refund (`s2/eval.jsonl`).

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
    → I used the API rule as an example: its frontmatter has `paths:` with `- "src/api/**/*"` (`s3/.claude/rules/api.md`). That keeps API guidance attached to matching files, while the tests rule can apply to `**/*.test.ts` and `**/*.test.tsx` anywhere in the repo. A directory `CLAUDE.md` would not express that cross-cutting match as cleanly; the validator returned `OK` (`s3/validation.log`).

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
    → The frontmatter in `s3/.claude/skills/deploy-check/SKILL.md` says `context: fork` and lists `Read`, `Grep`, `Glob`, `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, `Bash(gh pr view:*)`, and `Bash(gh pr checks:*)` under `allowed-tools`. That lets the check investigate separately without filling the main conversation with its working notes, and the allowlist keeps it from changing files. Without the fork, its output would land in the main context; without the allowlist, a check could make unintended edits. The validator returned `OK` (`s3/validation.log`).

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → In `s3/CLAUDE.md`, the project examples are `./CLAUDE.md` and `.claude/standards/`; those are checked in and shared with the team. The user-level examples are `~/.claude/CLAUDE.md` and personal commands or skills, which stay on one developer's machine. The validator accepted the scope explanation (`s3/validation.log`).

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm DB contains 40 fixture defects, but my README-command run selected none: its default eight-hour cutoff is newer than the fixture data (`s4/readme-command.log`). `WarmStore.defects_since` runs `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` against `idx_defects_ts`; the query plan is recorded in `s4/query_evidence.log` and the implementation is in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/warm.py`. The model gets the filtered window plus hot state, not the full table, which is the point of pushing selection into SQL.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/recovery.py`, an incomplete run resumes only if its last step is within 30 minutes; empty, complete, or older manifests start fresh. I also checked `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/fork.py`: each hypothesis gets a copy of the base hot state and its own scratchpad, so experiments do not overwrite the shared state. `merge_findings` appends those separate findings back to the main scratchpad. The recovery boundary and fork independence are covered in `s4/tests.log`; starting fresh is safer when a partial session has gone stale, but its saved summary can still be carried forward.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The hot state from my run is 643 bytes, well under the 5,120-byte limit (`s4/shift_output.log`). `HotState.write_atomic` enforces that limit in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/state.py`. Keeping that ceiling matters when the job runs every shift: otherwise even a small amount of state growth would accumulate into larger prompts over time.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → For the model layer, I used `s2/eval.jsonl`: it shows Haiku answering against the assembled context, including the exact active status token (`s2/context.md`). The harness layer is visible in `claims_intake/loop.py` and claim 4's trace (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py`, `s1/traces/claim_04_neighbor_injury.jsonl`); the code controls the loop and the trace records what happened. For orchestration, `pipeline.py` plus my warm database and hot-state file show the shift-level query and state flow (`Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/pipeline.py`, `s4/warm.sqlite`, `s4/hot_state.json`).

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → The 5,120-byte hot-state limit is enforced in code, and my output was 643 bytes (`state.py`, `s4/shift_output.log`; `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/state.py`). By contrast, asking one clarifying question for an ambiguous claim is prompt guidance; claim 4 did ask one (`s1/traces/claim_04_neighbor_injury.jsonl`). I would use code for hard limits and prompts for judgment, while monitoring prompt-led behavior because the run also had claims that ended before routing.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 trims a single conversation from 38,708 to 16,868 tokens, a 56.42% reduction, while keeping the active section verbatim (`s2/budget.json`). System 4 works across shifts: its hot state is 643 bytes, and the indexed query selects from a 40-row warm store (`s4/shift_output.log`, `s4/query_evidence.log`). Both keep the model's input bounded, but one summarizes resolved topics and the other carries forward compact state and queries history as needed.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → One useful check is `test_mid_write_read_reveals_prior_complete_lines` (`s4/tests.log`). It simulates a reader opening the manifest between writes and verifies that it sees complete entries, not a half-written record. A normal successful shift would not expose that failure mode, but a crash could leave the next run with state it cannot safely resume.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → I would worry most about System 1 misrouting or delaying a claim. The terminal tools constrain the intended action, and the runner escalated four cases when Haiku ended without using one (`s1/summary.md`). The immediate kill switch is stopping the run; token and wall-clock budgets also bound how long it can keep going (`budget.py`, `Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/claims_intake/budget.py`).

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first snag was local setup: system Python had no pytest or project dependencies, and the values in `.env` were not automatically exported to child processes. I made a temporary virtual environment and sourced `.env` with export enabled; `s1/environment-check.log` records that the variables loaded without recording their values. After that, both model-backed runs completed and the test logs were captured (`s1/tests.log`, `s2/tests.log`).

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would change System 2's Q5 evaluation. It passed because the response contained “prorated,” even though the answer said the customer had received the refund, which the transcript does not support (`s2/eval.jsonl`). That made me less willing to treat the 6/6 score as a correctness guarantee. I would check the receipt status explicitly instead of relying on one expected word.
