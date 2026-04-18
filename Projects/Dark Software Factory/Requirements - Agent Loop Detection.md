# Requirement: Agent Loop Detection & Escalation

> Related: [[Implementation Plan]] · [[Architecture]] · [[Worker Registry]]

---

## Motivation — Observed Incident

**Date:** 2026-04-17  
**Session:** `3b78f660-77c6-4a73-8fee-45a46a4bfad8` (home-infra project)  
**Duration:** ~2.5 hours (17:41 → 20:02)  
**Worker equivalent:** `cc-sonnet` (Claude Code + claude-sonnet-4-6)  
**Task:** Deploy distributed tracing stack (Tempo + OpenTelemetry Collector) to homelab Kubernetes

### What happened

The agent ran `install.sh` **11 times** across 491 conversation turns (290 assistant messages, 116 user messages) — spending 2.5 hours on a task that should have completed in one pass or escalated after 2–3 failures.

| Run # | Index | Command | Outcome |
|---|---|---|---|
| 1–2 | 17, 47 | `cd tracing && ./install.sh` | Helm repo/version mismatch |
| 3 | 50 | `install.sh 2>&1` | Same mismatch |
| 4 | 103 | `install.sh 2>&1` | Ran in background; timeout on StorageClass WaitForFirstConsumer |
| 5 | 142 | `install.sh 2>&1` | Ran in background; CrashLoopBackOff on Tempo pod |
| 6 | 200 | `install.sh 2>&1` | **Core loop error:** OTel Collector schema validation failure |
| 7 | 246 | `install.sh 2>&1` | Same schema error; different property attempted |
| 8 | 272 | `install.sh 2>&1` | Same schema error; different property attempted |
| 9 | 381 | `install.sh --no-tests 2>&1` | Syntax error in patched script |
| 10 | 399 | `install.sh --no-tests 2>&1` | Successful deployment |
| 11 | 438 | `install.sh --no-tests 2>&1` | Teardown/reinstall validation (intentional) |

### Root cause of the loop

The OpenTelemetry Collector Helm chart had breaking schema changes between versions. The error returned repeatedly between runs 6–8:

```
Error: values don't meet the specifications of the schema(s) in the following chart(s):
opentelemetry-collector:
- at '': additional properties 'replicas', 'servicePorts' not allowed
```

The agent responded each time by modifying one or two properties in the values YAML and re-running the full install script — a fix-attempt loop. It never stepped back to inspect the full chart schema before retrying. The same class of error (schema validation) recurred across runs 6, 7, and 8 with different specific properties, consuming ~90 minutes on this phase alone.

### Why this matters for the factory

In the dark factory, every `install.sh` equivalent is a full agent run billed by tokens and time. A single worker looping on the same class of error for 2.5 hours represents:
- **Cost:** potentially significant for cloud workers (`cc-sonnet`, `cx-gpt4o`)
- **Throughput:** one stuck subtask blocks downstream subtasks that depend on it
- **Signal loss:** `worker_evaluations` rows written for failed runs don't distinguish "wrong worker for this task" from "infrastructure issue" — the routing signal is polluted

The correct intervention: after 2–3 repetitions of the same error class, terminate the run and hand it to a different worker combination — different CLI, different model, or both.

---

## Requirement

### Summary

The orchestrator must detect when a coding agent is looping (repeating the same actions without making progress) and automatically escalate to a different worker combination rather than allowing the run to continue indefinitely.

### Detection — Two Layers

#### Layer 1: Within-run repetition detection (adapter level)

**For CLI-based adapters** (Claude Code, Codex, Qwen Code) that emit structured output:

Parse tool call events from the agent's output stream. Maintain a counter of `(tool_name, sha256(input))` pairs per run. If the same `(tool, input)` pair appears **3 or more times**, the run must be terminated immediately with status `LOOP_DETECTED`.

Configuration: `max_tool_repetitions: 3` (per run, configurable in broker ConfigMap).

The Claude Code adapter already parses `stream-json` output and can see every `tool_use` block. This detection can be added with minimal overhead.

**For HTTP-proxy adapters** (OpenHands, OpenCode) that don't expose individual tool calls:

Use **output velocity**. Track `last_output_at` timestamp. If the adapter receives no new bytes for `stall_timeout_seconds` (default: 90), terminate the run with status `STALLED`.

Configuration: `stall_timeout_seconds: 90` (configurable in broker ConfigMap).

#### Layer 2: Cross-run escalation (orchestrator level)

Current routing supports one primary and one fallback worker per task type. This is insufficient — when the fallback also fails, there is no further escalation path.

Replace the `primary/fallback` structure with an **ordered escalation chain** per task type. On any terminal failure status (`FAILED`, `LOOP_DETECTED`, `STALLED`), the orchestrator pops the next worker from the chain and retries. Maximum 3 escalation steps per subtask (after that, mark the subtask as permanently FAILED).

Proposed chains:

```yaml
escalation:
  code-gen-high:  [cc-qwen-coder-80b,   cc-sonnet,       cx-gpt4o]
  code-gen-med:   [oc-qwen-coder-next,  cc-qwen-coder-80b, cc-sonnet]
  code-gen-low:   [oc-qwen-coder-next,  cc-qwen-coder-80b]
  execution:      [oh-devstral,          oh-qwen-coder,   cc-sonnet]
  test-gen:       [oh-devstral,          oh-qwen-coder,   cc-qwen-coder-80b]
  refactor:       [oc-qwen-coder-next,  cc-qwen-coder-80b, cc-sonnet]
  code-review:    [cc-sonnet,           cx-gpt4o]
  decomposition:  [cc-sonnet]
  synthesis:      [cc-sonnet]
```

Key principle: escalation must change at least one of {agent CLI, model}. Re-running the same worker on the same input that already `LOOP_DETECTED` is never valid.

### New Run Statuses

Add to the `runs.status` enum:

| Status | Meaning |
|---|---|
| `LOOP_DETECTED` | Adapter terminated run because same tool+input repeated ≥ N times |
| `STALLED` | Adapter terminated run because no output received for ≥ N seconds |

Both are terminal failure statuses — the orchestrator treats them identically to `FAILED` for escalation purposes but logs them separately for routing signal quality.

### Schema Changes

**`dark_factory.runs` table** — add columns:

```sql
loop_info JSONB DEFAULT NULL
-- populated on LOOP_DETECTED: {tool, input_hash, count, first_seen_at, last_seen_at}
-- populated on STALLED: {last_output_at, stall_duration_seconds}
```

**`dark_factory.runs.status` enum** — add values:
```sql
ALTER TYPE run_status ADD VALUE 'LOOP_DETECTED';
ALTER TYPE run_status ADD VALUE 'STALLED';
```

**`dark_factory.worker_evaluations`** — add column:

```sql
termination_reason VARCHAR(32) DEFAULT NULL
-- values: null (normal), 'loop_detected', 'stalled', 'timeout', 'error'
```

This lets routing queries filter out polluted signal: a `LOOP_DETECTED` run on `oh-devstral` for task type `execution` doesn't mean that worker is bad at execution — it means the specific subtask description triggered a loop. Distinguish this from genuine capability failures.

### Metrics

Add to broker Prometheus output (all with `worker_id` label — the primary evaluation dimension):

```
dark_factory_loop_detected_total{worker_id, agent_cli, task_type}   # counter
dark_factory_stalled_total{worker_id, agent_cli, task_type}         # counter
dark_factory_escalations_total{from_worker_id, to_worker_id, reason} # counter
```

Add a Grafana panel to `dark-factory-overview`: **Escalation Rate** — `rate(dark_factory_escalations_total[1h])` grouped by `reason`. A spike in `loop_detected` escalations on a specific worker/task-type combination is a signal to update routing defaults.

### Configuration Surface

**Broker ConfigMap** (`acp-broker-config`):

```yaml
loop_detection:
  max_tool_repetitions: 3        # applies to CLI adapters (claude-code, codex, qwen-code)
  stall_timeout_seconds: 90      # applies to HTTP-proxy adapters (opencode, openhands)
```

**Orchestrator ConfigMap** — replace `routing` section with `escalation` chains (backward compatible: single-entry chains are equivalent to current `primary` with no fallback).

### Impacted Files

| File | Change |
|---|---|
| `db/schema.sql` | Add `LOOP_DETECTED`, `STALLED` to `run_status` enum; add `loop_info JSONB` to `runs`; add `termination_reason` to `worker_evaluations` |
| `db/migrate.sh` | Migration statements for new enum values and columns |
| `adapters/claudecode/src/main.py` | Tool-call repetition tracker using stream-json `tool_use` blocks |
| `adapters/codex/src/main.py` | Same, if structured tool output available; else stall detector |
| `adapters/qwencode/src/main.py` | Same pattern as claudecode |
| `adapters/opencode/src/main.py` | Output-velocity stall detector |
| `adapters/openhands/src/main.py` | Output-velocity stall detector |
| `broker/src/main.py` | Accept loop_detection config; new metrics; pass termination reason in run result |
| `orchestrator/src/main.py` | Replace primary/fallback with escalation chain runner; handle new terminal statuses |
| `broker/manifests/configmap.yaml` | Add `loop_detection` section |
| `orchestrator/manifests/configmap.yaml` | Replace `routing` with `escalation` chains |

### Implementation Phase

This feature should be implemented as part of **Phase 7 (Orchestrator)** for the cross-run escalation logic, with adapter-level detection added during **Phase 6 / 6.5** as each CLI adapter is built. The schema changes belong in **Phase 1**.

The detection logic in adapters is self-contained and adds no external dependencies. The escalation chain runner in the orchestrator replaces the existing fallback logic — not an addition on top of it.

---

## Acceptance Criteria

- [ ] A subtask whose agent calls the same Bash command 3× with identical input is terminated with `LOOP_DETECTED` before the 4th call
- [ ] `loop_info` JSONB is populated on the run record with tool name, input hash, repeat count, and timestamps
- [ ] On `LOOP_DETECTED`, the orchestrator selects the next worker in the escalation chain (not the same worker)
- [ ] A subtask that stalls (no output for 90s) is terminated with `STALLED` and escalated
- [ ] Escalation is capped at 3 attempts; after that the subtask status is `FAILED` with `error_message` indicating exhausted escalation chain
- [ ] `dark_factory_loop_detected_total` and `dark_factory_escalations_total` metrics are emitted and visible in Grafana
- [ ] `termination_reason` is recorded in `worker_evaluations` so routing queries can exclude polluted signal
- [ ] The same install.sh scenario (repeated identical command, same error class) would have been caught at the 3rd call, not the 6th
