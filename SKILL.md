---
name: metaopt-sanity-diagnosis
description: "Use when the ml-metaoptimization orchestrator encounters a sanity failure, code failure, or remote failure. Diagnoses the root cause and provides a concrete fix recommendation or abandonment guidance. Keywords: diagnosis, sanity failure, root cause analysis, debugging, metaoptimization worker."
---

# metaopt-sanity-diagnosis

## Overview

Diagnose failures from `LOCAL_SANITY` checks, code failures, or remote failure payloads during the `ml-metaoptimization` campaign cycle.

This is a leaf worker skill — it receives context from the orchestrator, performs analysis, and returns structured diagnostic output. It does not modify code, update state files, or interact with the queue backend directly.

The orchestrator dispatches this skill into the `diagnosis` auxiliary slot when:
- a `LOCAL_SANITY` command exits non-zero
- the remote backend returns a `failed` lifecycle status (dispatched from `WAIT_FOR_REMOTE_BATCH`)
- materialized code fails to load, import, or run

**Dispatch points:**
- `LOCAL_SANITY` failure → diagnosis may lead to remediation materialization, config block, or abandonment
- `WAIT_FOR_REMOTE_BATCH` failure → diagnosis always leads to a terminal transition (`FAILED` or `BLOCKED_CONFIG`) since remote failures cannot be remediated locally

The diagnosis worker's job is to explain *why* the failure happened and recommend *what to do about it*.

## Lane Assignment

- **Lane:** `diagnosis` (auxiliary slot)
- **Model class:** `strong_reasoner` — prefer a strong reasoning model (e.g. Opus 4.6 fast), fallback to a capable general model (e.g. GPT-5.4)

## Input Contract

The orchestrator provides all necessary context via the subagent prompt. The diagnosis worker does not read files from disk or access external systems.

### Standard Envelope (provided by orchestrator on every dispatch)

| Field | Type | Description |
|-------|------|-------------|
| `campaign_id` | string | Campaign identifier |
| `current_iteration` | integer | Current iteration number |
| `slot_id` | string | The slot ID dispatching this worker |
| `attempt` | integer | Attempt number for this dispatch (1-indexed) |

### Required Inputs

| Field | Type | Description |
|-------|------|-------------|
| `failure_context` | object | One of the three failure types below |
| `experiment_design` | object | The experiment batch spec that was being tested |
| `code_changes` | string | Patch artifact or changeset summary of applied changes |
| `sanity_config` | object | Campaign sanity configuration (command, requirements) |

### Failure Context Variants

**Sanity failure** (from `LOCAL_SANITY`):
- `stdout`: captured standard output from the sanity command
- `stderr`: captured standard error from the sanity command
- `exit_code`: non-zero exit code

**Remote failure** (from backend `status_command`):
- `classification`: failure classification from the backend (see backend-contract.md)
- `message`: human-readable failure message
- `returncode`: process return code if available

**Code failure** (from materialization or import):
- `error_type`: exception class or error category
- `traceback`: full traceback or error output
- `phase`: when the failure occurred (e.g. `import`, `initialization`, `execution`)

### Optional Inputs

| Field | Type | Description |
|-------|------|-------------|
| `previous_diagnoses` | list | Prior diagnosis outputs for this experiment (on retry) |
| `attempt_number` | integer | Current attempt (1-indexed) |
| `max_attempts` | integer | Orchestrator cap (default: 3) |

## Output Contract

Return a structured diagnostic report containing all of the following fields.

| Field | Type | Description |
|-------|------|-------------|
| `root_cause` | string | Clear, specific explanation of what failed and why |
| `classification` | enum | One of the failure classifications below |
| `fix_recommendation` | object | Concrete guidance for remediation (see below) |
| `confidence` | enum | `high`, `medium`, or `low` |

### Fix Recommendation Structure

```
fix_recommendation:
  action: "fix" | "adjust_config" | "abandon"
  details: <string — specific guidance>
  code_guidance: <string | null — targeted code change description>
  config_guidance: <string | null — configuration adjustment>
  reason_to_abandon: <string | null — why remediation is not viable>
```

- Exactly one of `code_guidance`, `config_guidance`, or `reason_to_abandon` should be non-null.
- When `action` is `"fix"`, `code_guidance` is required.
- When `action` is `"adjust_config"`, `config_guidance` is required. The orchestrator transitions to `BLOCKED_CONFIG` with the `config_guidance` as `next_action`.
- When `action` is `"abandon"`, `reason_to_abandon` is required.

## Failure Classifications

| Classification | When to Use |
|----------------|-------------|
| `code_error` | Bug in the experiment code changes — syntax error, logic error, wrong API usage, type mismatch |
| `config_error` | Wrong configuration, missing dependency, incorrect paths, environment variable issues |
| `data_error` | Data integrity problem, missing dataset, corrupt file, schema mismatch, loading failure |
| `infra_error` | Infrastructure or environment issue — OOM, disk full, network timeout, GPU unavailable |
| `design_error` | The experiment design itself is flawed — invalid hyperparameter range, incompatible architecture choice, metric not computable |

### Classification Decision Guide

1. Start with the error message and traceback. Does it point to a specific line in the patch? → likely `code_error`.
2. Does the error reference a missing file, package, or environment variable? → likely `config_error`.
3. Does the error occur during data loading or mention data shapes/types? → likely `data_error`.
4. Is the error about resources (memory, disk, network, GPU)? → likely `infra_error`.
5. Does the code run but produce nonsensical results, or does the metric computation fail because the approach is invalid? → likely `design_error`.

When uncertain between two classifications, prefer the more actionable one (the one that leads to a concrete fix).

## Behavioral Rules

1. **Focus on the most likely root cause.** Do not enumerate every possible failure mode. Identify the single most probable cause and explain it clearly.

2. **Track retry history.** If `previous_diagnoses` is non-empty, compare the current failure against prior diagnoses. Note what changed and what persisted. If the same root cause recurs after a fix attempt, the fix was insufficient — say so explicitly.

3. **Do not implement fixes.** Provide guidance for the orchestrator or the materialization worker to act on. Describe *what* to change, not the exact patch.

4. **Respect the attempt budget.** If `attempt_number` equals `max_attempts` and the fix is uncertain (confidence is `low` or `medium`), recommend abandonment. Do not recommend another fix attempt when there are no attempts left.

5. **Remote infrastructure failures are terminal.** If the classification is `infra_error` on a remote failure, recommend `abandon` — the orchestrator cannot retry remote submissions (retries are the backend's responsibility via `remote_queue.retry_policy`). For local `infra_error`, recommend `adjust_config` if the issue is environmental, or `abandon` if it is transient.

6. **Do not speculate about unrelated failures.** Analyze only the failure context provided. Do not hypothesize about other experiments, other parts of the codebase, or potential future failures.

7. **Outputs are persisted.** The orchestrator persists each diagnosis output as a record in `state.selected_experiment.diagnosis_history`. Subsequent diagnoses for the same experiment receive all prior records as `previous_diagnoses`. Write outputs that are useful as historical context.

8. **Be specific in code guidance.** Instead of "fix the data loading code", say "the `load_dataset` call on line 42 passes `split='test'` but the dataset only has a `'validation'` split — change the split argument".

9. **Flag temporal leakage violations.** If the sanity config requires zero temporal leakage and the failure or code changes suggest leakage (e.g., using future data in training, test set contamination), classify as `design_error` and recommend abandonment.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Listing every possible cause instead of the most likely one | Commit to one root cause; mention alternatives only if confidence is `low` |
| Recommending code fixes for infrastructure failures | Classify as `infra_error` and recommend retry or abandonment |
| Ignoring previous diagnosis attempts | Always check `previous_diagnoses` and note what changed |
| Recommending another fix on the last attempt with low confidence | Recommend abandonment when the budget is exhausted |
| Providing vague fix guidance like "fix the bug" | Reference specific functions, lines, variables, or configuration keys |
| Implementing the fix directly (writing code) | Provide guidance only — the materialization worker handles code changes |
| Diagnosing problems outside the current experiment scope | Restrict analysis to the provided failure context |

## References

- `ml-metaoptimization/references/worker-lanes.md` — authoritative lane contract for the `diagnosis` slot
- `ml-metaoptimization/references/backend-contract.md` — remote failure classifications and status lifecycle
- `ml-metaoptimization/SKILL.md` — orchestrator state machine and dispatch invariants
