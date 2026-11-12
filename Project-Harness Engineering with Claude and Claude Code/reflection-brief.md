# Reflection Brief — Harness Engineering Capstone

**Name:** Zied Besbes  
**Date:** 2026-11-12

**Environment**

- Model(s): `claude-haiku-4-5-20251001`
- OS / Python: Linux / Python 3.13.0
- Approx. API spend: System 1 recorded `$0.1230`; complete cross-system spend was not recorded and is therefore not verified.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

   In `runs/20261112_102455/traces/claim_02_stolen_bike.jsonl`, the sequence was `tool_use → tool_use → tool_use → tool_use → tool_use → end_turn`. The control decision is implemented in `claims_intake/loop.py` in the `run()` function. The loop continues only when `response.stop_reason == "tool_use"` and returns when it receives `end_turn`. Any other stop reason raises `UnexpectedStopReason`.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

   `tests/test_antipatterns.py` verifies that the loop does not search assistant text for magic strings to control execution. It also prevents a fixed integer-literal iteration limit from becoming the primary stopping mechanism. Text matching would make control flow dependent on natural-language wording instead of the structured API response. In run `20261112_102455`, this could have caused the loop to stop before routing or escalation, or continue after a terminal result.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

   `route_to_adjuster` and `escalate_to_human` are both terminal tools, but their descriptions define different conditions. Routing requires classification confidence of at least `0.6` and an assessed severity, while escalation is intended for low confidence, unresolved ambiguity, missing critical facts, or policy disputes. Structured errors contain fields such as `is_error`, `error_category`, and retryability information, allowing the harness to distinguish a recoverable tool problem from a permanent failure. A generic error string would require unreliable text interpretation and could lead to unsafe routing.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

   In `runs/20261112_102455/summary.md`, `claim_02_stolen_bike` was routed in 6 turns with an estimated cost of `$0.0243`. It consumed 20,463 input tokens and 762 output tokens. The supplied System 1 solution directory did not contain a README sample that could be verified, so a numeric comparison is not available. Differences between valid runs may result from model behavior and generated tool-call sequences.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

   Run `20261112-103303` reduced the raw baseline from 38,708 tokens to 16,774 assembled tokens, a reduction of `56.67%`. The active section dominates the result with 15,789 tokens. It remains verbatim because it represents the unresolved current interaction, where exact wording and recent details are needed to answer the active payment-update questions. The resolved issues can be compressed because their important outcomes have already been established.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

   Resolved conversation segments are summarized, while the active segment is preserved byte-exact. In `runs/20261112-103303/budget.json`, the refund segment was compressed from 12,334 input tokens to 354 output tokens, and the subscription segment from 11,475 to 419. In the final assembled context, the resolved sections used 367 and 432 tokens, while the active section remained at 15,789 tokens. The Case Facts block added 204 structured tokens for details that must remain directly answerable.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

   The complete context passed all 6 evaluation questions in `eval.jsonl`. In `eval_control.jsonl`, Q6 regressed after the Case Facts block was removed. Q6 asked for the exact structured status token of the payment-method update. This shows that the Case Facts block preserves precise structured values that may not remain reliably recoverable from summaries or the active transcript alone.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

   `.claude/rules/tests.md` declares the paths `"**/*.test.tsx"` and `"**/*.test.ts"`. These patterns apply test conventions to matching files anywhere in the repository. A directory-level `CLAUDE.md` would only govern one subtree and would either miss co-located tests in other directories or require duplicated configuration. Path-scoped rules express the real file-based scope directly and can layer on top of React or API rules.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

   `.claude/skills/deploy-check/SKILL.md` specifies `context: fork`. Its `allowed-tools` include `Read`, `Grep`, `Glob`, and restricted read-oriented Git and GitHub commands such as `git status`, `git diff`, and `gh pr checks`. Forking keeps verbose discovery and validation output out of the main conversation, while the read-only allowlist prevents the skill from modifying files, pushing, or deploying. Without the fork, the main context would be polluted by temporary output; without the allowlist, a pre-deployment check could create an unacceptable mutation risk.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

   The validator returned `OK` with exit code 0, and all 35 configuration tests passed. A project-level example is `./CLAUDE.md` together with `.claude/standards/` and `.claude/rules/`; these files are versioned and shared by the team. A user-level example would be `~/.claude/CLAUDE.md` or a personal command such as `/morning`, which remains local and is not shared through version control. The current repository intentionally keeps team conventions at project level.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

   The warm tier contained 40 defects, while the shift run returned 17 defects newer than `2026-04-01T00:00:00Z`. The indexed query is implemented by `WarmStore.defects_since()` in `shift_monitor/warm.py` using `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`. The database has the `idx_defects_ts` index, so filtering occurs in SQLite before prompt construction. The model therefore receives only the relevant slice rather than the complete defect history.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

   `shift_monitor/recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30`. An incomplete operation with a last step at most 30 minutes old is resumed; a completed, missing, or older state starts fresh. A fresh session with an injected summary avoids relying on stale partial context whose assumptions may no longer match the current shift state. The 33-test suite verifies the decision boundary at 29, 30, and 31 minutes.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

   After the recorded shift-C run, `data/hot_state.json` was 965 bytes, well below the approximate 5 KB budget. The shift processed 17 new defects and appended its finding to `data/shift_scratchpad.jsonl`. A strict hot-state budget prevents context and storage from growing with every shift. Without the limit, an indefinitely running system would gradually reload historical details into each session and undermine the tiered-state design.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

   **Model:** `claims_intake/loop.py` invokes `claude-haiku-4-5-20251001`, and trace `runs/20261112_102455/traces/claim_02_stolen_bike.jsonl` records the model’s tool-use sequence.

   **Harness:** System 3’s `CLAUDE.md`, `.claude/rules/`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/SKILL.md` define how Claude Code receives repository guidance and constrained capabilities.

   **Orchestration:** System 4’s `shift_monitor/pipeline.py`, `shift_monitor/warm.py`, `shift_monitor/recovery.py`, and `data/hot_state.json` coordinate shift execution, tiered state, recovery, and persistent findings.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code and one guided by prompt. When is each right?

   System 4 deterministically enforces SQL filtering in `WarmStore.defects_since()` and atomically maintains bounded hot state; these behaviors should not depend on model interpretation. System 1 uses tool descriptions to guide when the model should call `route_to_adjuster` or `escalate_to_human`. Deterministic code is appropriate for safety boundaries, storage guarantees, and access restrictions. Prompt guidance is appropriate when the decision depends on semantic interpretation of the claim, provided terminal tools and tests constrain the possible outcomes.

16. **Context, two faces.** Compare context management in System 2 and System 4 with cited numbers from both. Same principle, different mechanism — how?

   System 2 manages context inside one long conversation. Run `20261112-103303` reduced 38,708 tokens to 16,774, or `56.67%`, by summarizing resolved segments while preserving 15,789 active tokens and a 204-token Case Facts block. System 4 manages context across repeated sessions by querying only 17 of 40 defects and keeping `hot_state.json` at 965 bytes. Both designs retain the information needed for the current decision and move older detail into a more compact or external representation.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

   System 4 tests verify that recovery resumes at 30 minutes but starts fresh at 31 minutes. A single successful shift run cannot demonstrate this boundary or prove behavior after an interrupted write. The tests also verify atomic hot-state writes and that two forked scratchpads remain independent. These guarantees matter because recovery and isolation failures normally appear only during crashes or concurrent investigations.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

   In System 1, incorrect behavior could route a claim to the wrong adjuster queue or fail to escalate an ambiguous claim. The terminal tools limit the final action to `route_to_adjuster` or `escalate_to_human`, while the confidence threshold and structured tool schemas constrain the decision. `claims_intake/loop.py` raises on unexpected stop reasons, and its token and wall-clock budgets provide operational backstops. Stopping the runner prevents further claims from being processed, while already generated queue and escalation artifacts remain available for review.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it.

   The first System 1 API attempt failed with HTTP 400 because the placeholder API key was used instead of the actual workspace key. The base URL was also initially pasted with surrounding HTML rather than as a clean environment-variable value. I retrieved the active `voc-` key, set `ANTHROPIC_API_KEY` and the correct `ANTHROPIC_BASE_URL`, and reran the command. Run `20261112_102455` then completed all 8 fixtures and produced `summary.md`, queue files, escalation output, and traces.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

   I would add an explicit startup validation that rejects placeholder credentials and malformed base URLs before creating a run directory. The initial System 1 failure occurred only after the program began processing `claim_01_kitchen_fire`, even though the configuration problem was detectable immediately. A small preflight check could validate the key prefix and parse the base URL without exposing the credential. This would improve failure clarity without changing the agentic loop or its routing logic.
