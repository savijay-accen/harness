# Capstone Reflection Brief
## Harness Engineering with Claude and Claude Code

## Environment

- **Environment:** Udacity Vocareum VS Code workspace.
- **Operating system:** Linux.
- **Python version:** Python 3.13.18.
- **Model:** `claude-haiku-4-5-20251001` confirmed for System 1.
- **API cost:** System 1 reported an estimated total cost of **$0.1366**. A complete combined API spend across all systems was not recorded.
- **Evidence location:** `capstone-evidence/`

---

# Part 1 — Harness Engineering

## 1. Loop control

The Claims Intake Agent uses the API response's `stop_reason` to decide whether to continue or stop.

The controlling function is `run()` in `claims_intake/loop.py`.

- When `stop_reason` is `tool_use`, the loop executes the requested tools and continues.
- When `stop_reason` is `end_turn`, the loop returns the final state.
- Any unexpected stop reason raises an error.

The water-damage claim trace shows six tool-use responses followed by an `end_turn` response. This demonstrates why the loop must follow the model's actual response rather than assuming a fixed number of turns.

**Evidence:** `system1/claim_03_water_damage.jsonl`

## 2. Anti-pattern

One anti-pattern is using a fixed integer iteration limit as the main way to stop an agent loop.

For example, stopping after five iterations could prevent the agent from completing all required tool calls. The water-damage claim needed seven turns, including a clarification step.

The tests check that the loop uses `stop_reason` rather than an integer-literal iteration cap.

**Evidence:** System 1 source file `claims_intake/loop.py` and `tests/test_antipatterns.py`.

## 3. Tool design

Two tools with related inputs are `classify_claim` and `assess_severity`.

Their descriptions distinguish their responsibilities:

- `classify_claim` determines the type of claim.
- `assess_severity` evaluates how serious the claim is.

Structured tool errors also help the agent understand whether a failure is transient and retryable or permanent and not retryable. This is better than returning an unclear error message because the agent can make a more informed decision about retrying.

**Evidence:** System 1 `claims_intake/tools.py` and `claims_intake/loop.py`.

## 4. Numbers

The water-damage claim completed in **7 turns**, with an estimated cost of **$0.0319** and one clarification.

The System 1 README gives an approximate expected total cost of **$0.05** for the eight-claim run. The actual run cost was **$0.1366**.

The actual routing results also differed from the README expectation: four claims were routed and four were escalated, rather than the expected seven routed and one escalated.

This shows why actual execution evidence matters more than relying only on expected results.

**Evidence:** `system1/summary.md` and `system1/claim_03_water_damage.jsonl`.

## 5. Context reduction

System 2 reduced the baseline context from **38,708 tokens** to **16,885 tokens**.

The reduction was **56.38%**.

The largest section was the active conversation at **15,789 tokens**. The other sections were:

- Case facts: 204 tokens.
- Resolved refund: 428 tokens.
- Resolved subscription: 482 tokens.

The active section remained large because it contains the current conversation context needed to continue the task.

**Evidence:** `system2/budget.json`.

## 6. Summarize versus preserve

The System 2 strategy distinguishes between information that can be summarized and information that must remain exact.

Resolved interactions can be summarized because their full conversation history is not always needed. Active conversation content is preserved byte-exactly so that current context is not changed. Structured case facts are also kept separately to protect important values.

The budget report shows the active section at 15,789 tokens and case facts at 204 tokens.

**Evidence:** `system2/budget.json`.

## 7. Evaluation comparison

The normal evaluation file, `eval.jsonl`, passed all six questions.

The control evaluation, `eval_control.jsonl`, passed only one of two questions.

The control test failed to recover the exact structured status `in_progress`, although it could still retrieve the refund amount of `$22.14`.

This demonstrates that removing structured case facts can cause a loss of important information even when some details remain available in conversation history.

**Evidence:** `system2/eval.jsonl` and `system2/eval_control.jsonl`.

## 8. Path-scoped rules

The System 3 test rule uses these path patterns:

```yaml
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
```

These patterns apply the testing rules to matching files throughout the repository.

This is better than placing a separate directory-level `CLAUDE.md` in every folder because one rule can cover matching files across the project.

**Evidence:** `system3/.claude/rules/tests.md`.

## 9. Forked skill

The System 3 deployment-check skill uses:

```yaml
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git rev-parse:*)
  - Bash(git ls-files:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)
```

The forked context keeps detailed discovery work separate from the main conversation. The allowed-tools list limits the skill to inspection and read-only operations.

Without these restrictions, the skill could have a broader ability to modify files or perform actions that are not needed for a deployment check.

**Evidence:** `system3/.claude/skills/deploy-check/SKILL.md`.

## 10. Project-level versus user-level configuration

A project-level example is the repository's `CLAUDE.md` and `.claude/commands/review.md`. These provide shared instructions and workflows for the team.

A user-level example is `~/.claude/CLAUDE.md`, which can contain personal instructions that should not be shared with every repository.

The main difference is scope: project-level configuration supports shared team conventions, while user-level configuration supports an individual's preferences.

**Evidence:** `system3/CLAUDE.md` and `system3/.claude/commands/review.md`.

## 11. Database query and warm-tier data

System 4 returned **17 new defects** for Shift C.

The warm-tier SQLite database contained **40 defects** in total.

The `defects_since()` function filters records in SQL using a timestamp condition, orders them by timestamp, and applies a limit. The database also has an index on the timestamp column.

The model does not need to receive the complete database history. The application retrieves only the relevant records, which helps control context size and processing cost.

**Evidence:** `system4/system4-final-run.log`, `system4/hot_state.json`, and the System 4 `shift_monitor/warm.py` implementation.

## 12. Crash recovery

System 4 uses a **30-minute** stale-resume threshold.

- An incomplete run within the threshold can resume.
- An incomplete run older than the threshold starts fresh.
- A completed run starts fresh.
- An empty state also starts fresh.

Starting fresh with an injected summary can be better than continuing an old partial state because it avoids relying on stale intermediate work while retaining important findings.

**Evidence:** System 4 `shift_monitor/recovery.py` and recovery tests.

## 13. Hot-state size

The System 4 `hot_state.json` file measured **965 bytes**.

Keeping this state small matters because the monitoring system may operate across many shifts. If the state grows continuously, it can increase context usage and make future processing more expensive.

A compact state preserves important information without carrying unnecessary history forward.

**Evidence:** `system4/hot-state-size.txt` and `system4/hot_state.json`.

---

# Part 2 — Architecture and Reliability

## 14. Three layers

The three layers are:

**Model layer:** The model interprets the task, reasons about the available information, and selects tools. System 1's prompt and model interactions are examples.

**Harness layer:** The harness controls the interaction. System 1's `loop.py`, `tools.py`, and tests implement loop control, tool execution, error handling, and validation.

**Orchestration layer:** Orchestration coordinates longer-running work and persistent state. System 4's recovery logic, warm database, hot state, and scratchpad demonstrate this layer.

These layers have different responsibilities. The model reasons, the harness enforces execution rules, and orchestration manages work across time.

## 15. Deterministic versus prompt-based behavior

Deterministic behavior is appropriate when a rule must be enforced consistently.

Examples include:

- System 1 stopping based on `stop_reason`.
- System 2 applying a context budget.
- System 3 restricting tool permissions.
- System 4 applying the 30-minute recovery threshold.

Prompt-based behavior is useful for tasks that require language understanding and judgment, such as interpreting a claim description or summarizing resolved customer interactions.

I would use code for strict safety rules and repeatable checks, and prompts for tasks that require semantic understanding.

## 16. Context has two faces

System 2 demonstrates context management within a session. It reduced context from **38,708 tokens to 16,885 tokens**, a **56.38% reduction**, while preserving important active and structured information.

System 4 demonstrates context management across sessions or shifts. It uses a warm database, a compact hot-state file of **965 bytes**, a scratchpad, and a recovery threshold of **30 minutes**.

Both systems aim to keep relevant information available without carrying unnecessary history.

## 17. Reliability

A successful single run cannot prove that every important behavior is correct.

For example, System 4 includes a test named `test_two_forks_produce_independent_scratchpads`. It checks that two forks have independent scratchpads rather than accidentally sharing or changing one another's state.

A single successful run might not expose this type of problem. Dedicated tests are necessary to verify behavior under different conditions.

**Evidence:** System 4 fork tests and `system4/system4-tests.log`.

## 18. Blast radius and kill switch

I would use System 4 as an example because its output can influence quality-monitoring decisions.

If stale or incorrect state produces a wrong defect summary, the impact could extend to shift decisions or a recommended lot quarantine.

A practical safeguard is to stop the shift orchestration before acting on a questionable recommendation, preserve the database and state files, and require human verification before quarantine action.

The evidence includes the warm database, hot-state file, shift output, and recovery tests. The implementation evidence does not establish a fully automated kill switch, so I would treat the manual stop and human approval as a proposed safeguard.

**Evidence:** `system4/system4-final-run.log`, `system4/hot_state.json`, and System 4 recovery implementation.

---

# Part 3 — Lessons Learned

## 19. What broke and how I fixed it

During System 4 execution, I needed to ensure the command used the correct warm-database option and the correct fixture path.

I corrected the command and ran the shift using the recorded response fixture. The execution completed successfully and reported 17 new defects.

This taught me to verify command-line arguments and file paths carefully before troubleshooting the application itself.

**Evidence:** `system4/system4-final-run.log`.

## 20. What I would change

I would improve System 1 by adding a stronger evaluation and review step for routing decisions.

The actual run routed four claims and escalated four claims, while the README expected seven routed and one escalated. This difference deserves investigation.

I would add targeted tests for claims that are unexpectedly escalated, review confidence thresholds, and compare the model's reasoning with the expected routing behavior.

This would help identify whether the issue comes from the prompt, tool descriptions, confidence threshold, or model decision-making.

**Evidence:** `system1/summary.md` and the System 1 README.


