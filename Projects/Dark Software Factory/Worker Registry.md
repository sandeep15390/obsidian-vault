# Dark Software Factory — Worker Registry

> A **Worker** = one agent CLI harness + one model. The unit of evaluation and routing.
> Every run is tagged `worker_id`. Cost, duration, and quality are tracked per worker.

## All Workers (13 workers, 5 CLIs)

| Worker ID | CLI | Model | Tier | Capabilities | Monthly Budget |
|---|---|---|---|---|---|
| `cc-sonnet` | claude-code | `claude-sonnet-4-6` | ☁️ cloud | code-gen (complex), code-review, decomposition, synthesis | $15 |
| `cc-qwen-coder-80b` | claude-code | `ollama/qwen2.5-coder:80b` | 🏠 local | code-gen, refactor, code-review | $0 |
| `cc-qwen-coder-next` | claude-code | `ollama/qwen3-coder-next` | 🏠 local | code-gen, refactor | $0 |
| `cx-gpt4o` | codex | `gpt-4o` | ☁️ cloud | code-gen, code-review | $10 |
| `cx-qwen-coder-80b` | codex | `ollama/qwen2.5-coder:80b` | 🏠 local | code-gen | $0 |
| `cx-oss-120b` | codex | `ollama/gpt-oss:120b` | 🏠 local | code-gen, large-context | $0 |
| `oc-qwen-coder-next` | opencode | `ollama/qwen3-coder-next` | 🏠 local | code-gen, refactor | $0 |
| `oc-devstral` | opencode | `ollama/devstral-small-2:24b` | 🏠 local | code-gen, refactor | $0 |
| `oh-devstral` | openhands | `ollama/devstral-small-2:24b` | 🏠 local | test-gen, execution | $0 |
| `oh-qwen-coder` | openhands | `ollama/qwen2.5-coder:32b` | 🏠 local | test-gen, execution | $0 |
| `qc-qwen-coder-80b` | qwen-code | `ollama/qwen2.5-coder:80b` | 🏠 local | code-gen, refactor | $0 |
| `qc-qwen-coder-next` | qwen-code | `ollama/qwen3-coder-next` | 🏠 local | code-gen, refactor | $0 |
| `qc-qwen-coder-32b` | qwen-code | `ollama/qwen2.5-coder:32b` | 🏠 local | code-gen, refactor | $0 |

**11 local workers / 2 cloud workers.** Default routing is local-first; cloud workers are primary only for decomposition, synthesis, and high-complexity code-gen where reliability of structured output matters.

---

## CLI Harnesses

| CLI | Adapter | Port | How it runs | Why include it |
|---|---|---|---|---|
| **claude-code** | adapter-claudecode | :8004 | subprocess `claude -p --bare --output-format stream-json` | Highest reasoning quality; best for decomposition and synthesis |
| **codex** | adapter-codex | :8003 | subprocess `codex exec --full-access` | OpenAI-native harness; comparison vs claude-code on same task |
| **opencode** | adapter-opencode | :8001 | `opencode serve` REST API (sidecar) | Go TUI agent; HTTP-first design; good for iterative edits |
| **openhands** | adapter-openhands | :8002 | REST → openhands-server | **Only harness that can execute code** in an isolated sandbox |
| **qwen-code** | adapter-qwencode | :8005 | subprocess `qwen-code -p --output-format stream-json` | Native Qwen CLI — model-specific prompting not available in third-party harnesses |

---

## Why qwen-code alongside claude-code and codex for Qwen models?

The same model (e.g. `qwen2.5-coder:80b`) routes through three different harnesses:

| Worker | Harness | Same model |
|---|---|---|
| `cc-qwen-coder-80b` | claude-code | qwen2.5-coder:80b |
| `cx-qwen-coder-80b` | codex | qwen2.5-coder:80b |
| `qc-qwen-coder-80b` | qwen-code | qwen2.5-coder:80b |

The `worker_evaluations` table will reveal which harness extracts the most from the model. If `qc-qwen-coder-80b` consistently outscores the others on code-gen tasks, that's signal to promote it to primary and deprioritise the others for that model. **This is the core evaluation hypothesis the factory is designed to test.**

---

## Model Cross-Reference

| Model | Workers | Harnesses compared |
|---|---|---|
| `ollama/qwen2.5-coder:80b` | `cc-qwen-coder-80b`, `cx-qwen-coder-80b`, `qc-qwen-coder-80b` | claude-code vs codex vs qwen-code |
| `ollama/qwen3-coder-next` | `cc-qwen-coder-next`, `oc-qwen-coder-next`, `qc-qwen-coder-next` | claude-code vs opencode vs qwen-code |
| `ollama/qwen2.5-coder:32b` | `oh-qwen-coder`, `qc-qwen-coder-32b` | openhands vs qwen-code |
| `ollama/devstral-small-2:24b` | `oc-devstral`, `oh-devstral` | opencode vs openhands |

---

## LiteLLM Virtual Keys

One key per worker. All stored in the `dark-factory-api-keys` Secret. Budget caps enforced at LiteLLM layer — not in application code.

| Key alias | Secret env var | Worker | Budget |
|---|---|---|---|
| `df-cc-sonnet` | `LITELLM_KEY_CC_SONNET` | cc-sonnet | $15/mo |
| `df-cc-qwen-80b` | `LITELLM_KEY_CC_QWEN80B` | cc-qwen-coder-80b | $0 |
| `df-cc-qwen-next` | `LITELLM_KEY_CC_QWEN_NEXT` | cc-qwen-coder-next | $0 |
| `df-cx-gpt4o` | `LITELLM_KEY_CX_GPT4O` | cx-gpt4o | $10/mo |
| `df-cx-qwen-80b` | `LITELLM_KEY_CX_QWEN80B` | cx-qwen-coder-80b | $0 |
| `df-cx-oss-120b` | `LITELLM_KEY_CX_OSS120B` | cx-oss-120b | $0 |
| `df-oc-qwen-next` | `LITELLM_KEY_OC_QWEN_NEXT` | oc-qwen-coder-next | $0 |
| `df-oc-devstral` | `LITELLM_KEY_OC_DEVSTRAL` | oc-devstral | $0 |
| `df-oh-devstral` | `LITELLM_KEY_OH_DEVSTRAL` | oh-devstral | $0 |
| `df-oh-qwen` | `LITELLM_KEY_OH_QWEN` | oh-qwen-coder | $0 |
| `df-qc-qwen-80b` | `LITELLM_KEY_QC_QWEN80B` | qc-qwen-coder-80b | $0 |
| `df-qc-qwen-next` | `LITELLM_KEY_QC_QWEN_NEXT` | qc-qwen-coder-next | $0 |
| `df-qc-qwen-32b` | `LITELLM_KEY_QC_QWEN32B` | qc-qwen-coder-32b | $0 |

The ACP broker reads these from the mounted Secret and injects the correct key per-run. Adapters never hold multiple keys.

---

## Workspace Layout

```
/workspaces/                              (PVC: dark-factory-workspaces)
  _base/{task_id}/                        base clone; read-only; shared across workers for same task
  cc-sonnet/{run_id}/                     git worktree; branch: worker/cc-sonnet/{run_id}
  cc-qwen-coder-80b/{run_id}/             git worktree; branch: worker/cc-qwen-coder-80b/{run_id}
  cx-gpt4o/{run_id}/
  qc-qwen-coder-80b/{run_id}/
  ...
```

---

## Evaluation Queries

```sql
-- Harness comparison for a specific model on code-gen
SELECT worker_id, agent_cli,
       COUNT(*) AS runs,
       ROUND(AVG(CASE WHEN succeeded THEN 1.0 ELSE 0.0 END)*100, 1) AS success_pct,
       ROUND(AVG(duration_ms)/1000.0, 1) AS avg_s,
       ROUND(AVG(estimated_cost_usd), 6) AS avg_cost,
       ROUND(AVG(reviewer_score), 2) AS avg_quality
FROM dark_factory.worker_evaluations
WHERE model = 'ollama/qwen2.5-coder:80b'
  AND task_type = 'code-gen'
  AND created_at > NOW() - INTERVAL '30 days'
GROUP BY worker_id, agent_cli
HAVING COUNT(*) >= 3
ORDER BY avg_quality DESC NULLS LAST;

-- Full routing review — all workers, all task types
SELECT worker_id, task_type,
       COUNT(*) AS n,
       ROUND(AVG(CASE WHEN succeeded THEN 1.0 ELSE 0.0 END)*100, 1) AS success_pct,
       ROUND(AVG(duration_ms)/1000.0, 1) AS avg_s,
       ROUND(AVG(estimated_cost_usd), 5) AS avg_cost,
       ROUND(AVG(reviewer_score), 2) AS avg_quality
FROM dark_factory.worker_evaluations
WHERE created_at > NOW() - INTERVAL '14 days'
GROUP BY worker_id, task_type
HAVING COUNT(*) >= 5
ORDER BY task_type, avg_quality DESC NULLS LAST;
```

---

## Adding a New Worker

1. Add entry to `broker/manifests/configmap.yaml` (`workers.yaml`)
2. Create LiteLLM virtual key; add to `dark-factory-api-keys` Secret
3. Ensure model is available in LiteLLM (Ollama loaded or cloud API key present)
4. If new CLI → new adapter Deployment + Service + test suite (new Phase)
5. If existing CLI → no new adapter; broker routes by `worker_id`, adapter reads model from run metadata
6. Optionally add to routing table ConfigMap if the worker should receive traffic immediately
