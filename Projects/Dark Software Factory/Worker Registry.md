# Dark Software Factory — Worker Registry

> A **Worker** = one agent CLI harness + one model. The unit of evaluation and routing.
> Every run is tagged `worker_id`. Cost, duration, and quality tracked per worker.
> Model routing is direct — no intermediate gateway.

## All Workers (13 workers, 5 CLIs)

| Worker ID | CLI | Model | Backend | Capabilities |
|---|---|---|---|---|
| `cc-sonnet` | claude-code | `claude-sonnet-4-6` | Anthropic API ☁️ | code-gen (complex), code-review, decomposition, synthesis |
| `cc-qwen-coder-80b` | claude-code | `qwen2.5-coder:80b` | Ollama 🏠 | code-gen, refactor, code-review |
| `cc-qwen-coder-next` | claude-code | `qwen3-coder-next` | Ollama 🏠 | code-gen, refactor |
| `cx-gpt4o` | codex | `gpt-4o` | OpenAI API ☁️ | code-gen, code-review |
| `cx-qwen-coder-80b` | codex | `qwen2.5-coder:80b` | Ollama 🏠 | code-gen |
| `cx-oss-120b` | codex | `gpt-oss:120b` | Ollama 🏠 | code-gen, large-context |
| `oc-qwen-coder-next` | opencode | `qwen3-coder-next` | Ollama 🏠 | code-gen, refactor |
| `oc-devstral` | opencode | `devstral-small-2:24b` | Ollama 🏠 | code-gen, refactor |
| `oh-devstral` | openhands | `devstral-small-2:24b` | Ollama 🏠 | test-gen, execution |
| `oh-qwen-coder` | openhands | `qwen2.5-coder:32b` | Ollama 🏠 | test-gen, execution |
| `qc-qwen-coder-80b` | qwen-code | `qwen2.5-coder:80b` | Ollama 🏠 | code-gen, refactor |
| `qc-qwen-coder-next` | qwen-code | `qwen3-coder-next` | Ollama 🏠 | code-gen, refactor |
| `qc-qwen-coder-32b` | qwen-code | `qwen2.5-coder:32b` | Ollama 🏠 | code-gen, refactor |

**11 local (Ollama) / 2 cloud.** Cloud workers are primary only for decomposition, synthesis, and high-complexity code-gen.

---

## Model Endpoints per CLI

Each CLI harness uses a different env var to point at its model backend:

| CLI | Local (Ollama) env vars | Cloud env vars |
|---|---|---|
| **claude-code** | `ANTHROPIC_BASE_URL=http://<ollama>:11434` + `ANTHROPIC_API_KEY=ollama` | `ANTHROPIC_API_KEY=sk-ant-...` (unset BASE_URL) |
| **codex** | `OPENAI_BASE_URL=http://<ollama>:11434/v1` + `OPENAI_API_KEY=ollama` | `OPENAI_API_KEY=sk-...` (unset BASE_URL) |
| **opencode** | `OPENCODE_API_BASE=http://<ollama>:11434/v1` + `OPENCODE_API_KEY=ollama` | n/a (no cloud workers) |
| **openhands** | `LLM_BASE_URL=http://<ollama>:11434/v1` + `LLM_API_KEY=ollama` | n/a |
| **qwen-code** | `OPENAI_BASE_URL=http://<ollama>:11434/v1` + `OPENAI_API_KEY=ollama` | n/a |

> **Why direct Ollama works for Claude Code:** Ollama 0.20.4+ implements the Anthropic Messages API natively at `/v1/messages` including structured tool_use blocks. Routing through LiteLLM broke tool execution — validated in live testing.

---

## Worker Config in ACP Broker ConfigMap

```yaml
workers:
  - worker_id: cc-sonnet
    agent_cli: claude-code
    adapter_url: http://adapter-claudecode.dark-factory.svc.cluster.local:8004
    model: claude-sonnet-4-6
    api_base: ""                        # empty = use default Anthropic API
    api_key_secret: ANTHROPIC_API_KEY
    workspace_root: /workspaces/cc-sonnet
    git_branch_prefix: worker/cc-sonnet
    capabilities: [code-gen, code-review, decomposition, synthesis]
    cost_tier: cloud

  - worker_id: cc-qwen-coder-next
    agent_cli: claude-code
    adapter_url: http://adapter-claudecode.dark-factory.svc.cluster.local:8004
    model: qwen3-coder-next
    api_base: "${OLLAMA_BASE_URL}"      # http://10.0.0.7:11434
    api_key_secret: ""                  # empty = use "ollama" as key
    workspace_root: /workspaces/cc-qwen-coder-next
    git_branch_prefix: worker/cc-qwen-coder-next
    capabilities: [code-gen, refactor]
    cost_tier: local

  - worker_id: cc-qwen-coder-80b
    agent_cli: claude-code
    adapter_url: http://adapter-claudecode.dark-factory.svc.cluster.local:8004
    model: qwen2.5-coder:80b
    api_base: "${OLLAMA_BASE_URL}"
    api_key_secret: ""
    workspace_root: /workspaces/cc-qwen-coder-80b
    git_branch_prefix: worker/cc-qwen-coder-80b
    capabilities: [code-gen, refactor, code-review]
    cost_tier: local

  - worker_id: cx-gpt4o
    agent_cli: codex
    adapter_url: http://adapter-codex.dark-factory.svc.cluster.local:8003
    model: gpt-4o
    api_base: ""
    api_key_secret: OPENAI_API_KEY
    workspace_root: /workspaces/cx-gpt4o
    git_branch_prefix: worker/cx-gpt4o
    capabilities: [code-gen, code-review]
    cost_tier: cloud

  - worker_id: cx-qwen-coder-80b
    agent_cli: codex
    adapter_url: http://adapter-codex.dark-factory.svc.cluster.local:8003
    model: qwen2.5-coder:80b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/cx-qwen-coder-80b
    git_branch_prefix: worker/cx-qwen-coder-80b
    capabilities: [code-gen]
    cost_tier: local

  - worker_id: cx-oss-120b
    agent_cli: codex
    adapter_url: http://adapter-codex.dark-factory.svc.cluster.local:8003
    model: gpt-oss:120b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/cx-oss-120b
    git_branch_prefix: worker/cx-oss-120b
    capabilities: [code-gen]
    cost_tier: local

  - worker_id: oc-qwen-coder-next
    agent_cli: opencode
    adapter_url: http://adapter-opencode.dark-factory.svc.cluster.local:8001
    model: qwen3-coder-next
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/oc-qwen-coder-next
    git_branch_prefix: worker/oc-qwen-coder-next
    capabilities: [code-gen, refactor]
    cost_tier: local

  - worker_id: oc-devstral
    agent_cli: opencode
    adapter_url: http://adapter-opencode.dark-factory.svc.cluster.local:8001
    model: devstral-small-2:24b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/oc-devstral
    git_branch_prefix: worker/oc-devstral
    capabilities: [code-gen, refactor]
    cost_tier: local

  - worker_id: oh-devstral
    agent_cli: openhands
    adapter_url: http://adapter-openhands.dark-factory.svc.cluster.local:8002
    model: devstral-small-2:24b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/oh-devstral
    git_branch_prefix: worker/oh-devstral
    capabilities: [test-gen, execution]
    cost_tier: local

  - worker_id: oh-qwen-coder
    agent_cli: openhands
    adapter_url: http://adapter-openhands.dark-factory.svc.cluster.local:8002
    model: qwen2.5-coder:32b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/oh-qwen-coder
    git_branch_prefix: worker/oh-qwen-coder
    capabilities: [test-gen, execution]
    cost_tier: local

  - worker_id: qc-qwen-coder-80b
    agent_cli: qwen-code
    adapter_url: http://adapter-qwencode.dark-factory.svc.cluster.local:8005
    model: qwen2.5-coder:80b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/qc-qwen-coder-80b
    git_branch_prefix: worker/qc-qwen-coder-80b
    capabilities: [code-gen, refactor]
    cost_tier: local

  - worker_id: qc-qwen-coder-next
    agent_cli: qwen-code
    adapter_url: http://adapter-qwencode.dark-factory.svc.cluster.local:8005
    model: qwen3-coder-next
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/qc-qwen-coder-next
    git_branch_prefix: worker/qc-qwen-coder-next
    capabilities: [code-gen, refactor]
    cost_tier: local

  - worker_id: qc-qwen-coder-32b
    agent_cli: qwen-code
    adapter_url: http://adapter-qwencode.dark-factory.svc.cluster.local:8005
    model: qwen2.5-coder:32b
    api_base: "${OLLAMA_BASE_URL}/v1"
    api_key_secret: ""
    workspace_root: /workspaces/qc-qwen-coder-32b
    git_branch_prefix: worker/qc-qwen-coder-32b
    capabilities: [code-gen, refactor]
    cost_tier: local
```

---

## CLI Harnesses

| CLI | Adapter | Port | Invocation | Notes |
|---|---|---|---|---|
| **claude-code** | adapter-claudecode | :8004 | `claude -p <task> --bare --output-format stream-json` | Direct Ollama for local models; Anthropic API for cc-sonnet |
| **codex** | adapter-codex | :8003 | `codex exec --full-access <task>` | Direct Ollama `/v1` for local; OpenAI API for cx-gpt4o |
| **opencode** | adapter-opencode | :8001 | `opencode serve --port 4096` + REST | Direct Ollama `/v1` |
| **openhands** | adapter-openhands | :8002 | REST → openhands-server :3000 | Only sandbox-capable harness; direct Ollama `/v1` |
| **qwen-code** | adapter-qwencode | :8005 | `qwen-code -p <task>` | Direct Ollama `/v1`; native Qwen harness |

---

## Why qwen-code alongside claude-code and codex for Qwen models?

Same model through three different harnesses:

| Worker | Harness | Model |
|---|---|---|
| `cc-qwen-coder-80b` | claude-code | qwen2.5-coder:80b |
| `cx-qwen-coder-80b` | codex | qwen2.5-coder:80b |
| `qc-qwen-coder-80b` | qwen-code | qwen2.5-coder:80b |

The `worker_evaluations` table will show which harness extracts the most from the model. If `qc-qwen-coder-80b` consistently outscores the others on code-gen tasks, promote it to primary.

---

## Model Cross-Reference

| Model | Workers | Harnesses compared |
|---|---|---|
| `qwen2.5-coder:80b` | `cc-qwen-coder-80b`, `cx-qwen-coder-80b`, `qc-qwen-coder-80b` | claude-code vs codex vs qwen-code |
| `qwen3-coder-next` | `cc-qwen-coder-next`, `oc-qwen-coder-next`, `qc-qwen-coder-next` | claude-code vs opencode vs qwen-code |
| `qwen2.5-coder:32b` | `oh-qwen-coder`, `qc-qwen-coder-32b` | openhands vs qwen-code |
| `devstral-small-2:24b` | `oc-devstral`, `oh-devstral` | opencode vs openhands |

---

## Secrets Required

Only cloud workers need API key secrets in the `dark-factory-api-keys` Secret:

| Secret key | Used by | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | cc-sonnet | Direct Anthropic API |
| `OPENAI_API_KEY` | cx-gpt4o | Direct OpenAI API |

All local workers use `OLLAMA_BASE_URL` (from ConfigMap or env) with no authentication.

---

## Cost Estimation (without LiteLLM)

Token counts come from API responses (`usage.input_tokens`, `usage.output_tokens`). Cost is calculated in the orchestrator using a static pricing table:

```python
COST_PER_1M_TOKENS = {
    "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
    "gpt-4o":            {"input": 2.50, "output": 10.00},
    # all Ollama models
    "default_local":     {"input": 0.0,  "output": 0.0},
}
```

---

## Workspace Layout

```
/workspaces/                              (PVC: dark-factory-workspaces)
  _base/{task_id}/                        base clone; read-only
  cc-sonnet/{run_id}/                     branch: worker/cc-sonnet/{run_id}
  cc-qwen-coder-next/{run_id}/
  qc-qwen-coder-80b/{run_id}/
  ...
```

---

## Evaluation Queries

```sql
-- Harness comparison for qwen2.5-coder:80b
SELECT worker_id, agent_cli,
       COUNT(*) AS runs,
       ROUND(AVG(CASE WHEN succeeded THEN 1.0 ELSE 0.0 END)*100, 1) AS success_pct,
       ROUND(AVG(duration_ms)/1000.0, 1) AS avg_s,
       ROUND(AVG(estimated_cost_usd), 6) AS avg_cost,
       ROUND(AVG(reviewer_score), 2) AS avg_quality
FROM dark_factory.worker_evaluations
WHERE model = 'qwen2.5-coder:80b'
  AND task_type = 'code-gen'
  AND created_at > NOW() - INTERVAL '30 days'
GROUP BY worker_id, agent_cli
HAVING COUNT(*) >= 3
ORDER BY avg_quality DESC NULLS LAST;
```

---

## Adding a New Worker

1. Add entry to `broker/manifests/configmap.yaml`
2. Add `api_key_secret` to `dark-factory-api-keys` Secret only if it's a cloud model
3. Ensure model is loaded in Ollama (local) or cloud API key is valid
4. New CLI harness → new adapter Deployment + Service + test suite
5. Existing CLI harness → no new adapter; broker injects model config per-run
6. Optionally update routing ConfigMap to route traffic to the new worker
