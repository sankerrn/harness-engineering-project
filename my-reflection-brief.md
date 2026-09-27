# Reflection Brief: Harness Engineering with Claude and Claude Code

## Header / Environment

* Name: Sanker R Nath
* Date: 2026-09-26
* Model(s): claude-3-5-sonnet-20241022
* OS / Python: Linux / Python 3.13
* Approx. API spend: $0.13

---

## Part 1 — Per-System Breakdown

### System 1 — Agentic Loop (Claims Intake Agent)

* **Test Evidence:** Verified 29 passed tests via `pytest tests/ -v` (recorded in `evidence/system1_agentic_loop/pytest_S1.log`).

* **Run ID / Artifact:** Run directory `runs/20260927_103358/`, claim trace `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`, and batch results in `evidence/system1_agentic_loop/summary.md`.

* **Core Explanation:** Loop termination is strictly governed by Anthropic's `stop_reason` protocol inside `claims_intake/agent.py`. When Claude returns `stop_reason == "tool_use"`, the loop intercepts the structured tool call, executes the requested utility (such as `lookup_policy`, `record_claim_fact`, or `assess_severity`), and appends the result into the conversation context for the next inference turn. When Claude returns `stop_reason == "end_turn"`, the loop concludes and emits the final outcome determination. Relying on explicit SDK stop reasons rather than open-ended string parsing prevents unbounded infinite execution loops and suppresses tool execution hallucinations.

### System 2 — Context Strategy (Retail Support Copilot)

* **Test Evidence:** Verified 30 passed tests (28 passed, 2 skipped) recorded in `evidence/system2_context_strategy/pytest_S2.log`.

* **Artifact & Metrics:** Run recorded in `evidence/system2_context_strategy/budget.json` and evaluation outputs in `eval.jsonl`. The baseline uncompressed transcript required 38,708 tokens, while the tiered assembled context reduced usage to 16,851 tokens—demonstrating a 56.47% compression reduction (comfortably surpassing the 50% requirement). The final assembled payload allocated 204 tokens for extracted case facts, 360 tokens for resolved refund history, 516 tokens for resolved subscription details, and 15,789 tokens for the active turn segment.

* **Core Explanation:** By pinning static system instructions and extracted case facts at the top of the context window while preserving full, raw recent conversation turns at the end, the architecture protects high-priority state and active dialog from truncation. Summarizing completed sub-threads into compact summaries ensures answerability across long multi-turn sessions (achieving 6/6 correct responses on eval), while preventing context overflow.

### System 3 — Claude Code Configuration (Multi-Surface Monorepo)

* **Test Evidence:** Verified 35 passed tests recorded in `evidence/system3_claude_config/pytest_S3.log` alongside a validator status of `OK` in `evidence/system3_claude_config/validator_output.txt`.

* **Artifacts:** Root `CLAUDE.md`, path rules in `.claude/rules/` (`react.md`, `api.md`, `tests.md`), custom slash command `.claude/commands/review.md`, and deployment skill `.claude/skills/deploy-check/SKILL.md`.

* **Core Explanation:** The root `CLAUDE.md` maintains strict modularity by keeping file size under 200 lines and pulling shared standards via `@import` statements. Path-scoped rules load domain-specific constraints only when modifying matching file globs, preventing instruction bloat. The `/review` slash command standardizes quality audits across code reviews, while the `deploy-check` skill runs inside a forked sub-agent sandbox with restricted read-only tools (`Bash`, `Glob`, `Grep`, `Read`) to isolate pre-flight verification without risking destructive file modifications.

### System 4 — Quality Monitoring Orchestrator (Multi-Shift System)

* **Test Evidence:** Verified 33 passed tests in 48.47s recorded in `evidence/system4_orchestration/pytest_S4.log`.

* **Artifacts:** Hot state budget recorded in `evidence/system4_orchestration/hot_state_size.txt`, scratchpad audit trace `evidence/system4_orchestration/scratchpad_line.jsonl`, and execution log `evidence/system4_orchestration/shift_run_output.txt`.

* **Core Explanation:** State resilience relies on durable filesystem storage rather than ephemeral in-memory variables. Volatile tracking state is constrained to a compact schema in `hot_state.json` (remaining under 5 KB), while historical defect logs persist to SQLite (`data/warm.sqlite`) with index-backed `defects_since` SQL queries to eliminate bulk in-memory table scans. Shift handoffs append progress and crash checkpoints to `shift_scratchpad.jsonl` using durable atomic writes and `fsync`. When exploring anomaly hypotheses, the orchestrator forks dedicated temporary scratchpads, allowing deep diagnostic branching without contaminating the primary production monitoring sequence.

---

## Part 2 — Cross-System Synthesis

### 19. What broke

During initial testing in System 2, uncompressed historical transcripts rapidly exhausted the prompt token budget during multi-turn LLM extraction calls. This was resolved by implementing strict issue-boundary partitioning: completed threads were summarized into structured facts (`resolved/refund` and `resolved/subscription`), leaving the full token budget available for active dialog turns in `evidence/system2_context_strategy/budget.json`. This structural isolation mirrors the tiered memory management in System 4 (`evidence/system4_orchestration/hot_state_size.txt`), where older records are pushed down to SQLite warm storage to prevent the active hot state from exceeding 5 KB.

### 20. What you'd change

I would incorporate the tool permission isolation model demonstrated in System 3 into System 1's claims intake loop. In System 3 (`evidence/system3_claude_config/evidence_rule_skill.md`), the deploy skill runs inside a sandboxed sub-agent restricted strictly to read-only tools. In contrast, the agentic intake loop in System 1 (`evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`) presents all registered claim tools simultaneously across every turn. Restricting tool availability dynamically based on the current intake phase would minimize token overhead and prevent accidental calls to downstream action tools prior to proper policy validation.

---

## Part 3 — Honest assessment

19. **What broke.** During initial testing in System 2, uncompressed historical transcripts rapidly exhausted the prompt token budget during multi-turn LLM extraction calls. This was resolved by implementing strict issue-boundary partitioning: completed threads were summarized into structured facts (`resolved/refund` and `resolved/subscription`), leaving the full token budget available for active dialog turns in `evidence/system2_context_strategy/budget.json`. This structural isolation mirrors the tiered memory management in System 4 (`evidence/system4_orchestration/hot_state_size.txt`), where older records are pushed down to SQLite warm storage to prevent the active hot state from exceeding 5 KB.

20. **What you'd change.** I would incorporate the tool permission isolation model demonstrated in System 3 into System 1's claims intake loop. In System 3 (`evidence/system3_claude_config/evidence_rule_skill.md`), the deploy skill runs inside a sandboxed sub-agent restricted strictly to read-only tools. In contrast, the agentic intake loop in System 1 (`evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`) presents all registered claim tools simultaneously across every turn. Restricting tool availability dynamically based on the current intake phase would minimize token overhead and prevent accidental calls to downstream action tools prior to proper policy validation.
