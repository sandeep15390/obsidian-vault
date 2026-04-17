# Dark Software Factory — Architecture

> Implementation target: `~/src/home_infra/dark-factory/`
> Full design doc: `~/src/home_infra/docs/dark-factory/architecture.md`

## System Diagram

```
┌─────────────────────────────────────────────────────────┐
│  External Inputs                                        │
│  POST /tasks (HTTP)  │  GitHub Webhook  │  CLI submit   │
└──────────────────────┴──────────────────┴───────────────┘
                                │
                    ┌───────────▼───────────┐
                    │  Orchestrator          │
                    │  FastAPI + asyncio     │
                    │  NodePort configurable │
                    └───────────┬───────────┘
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
         Redis queue       PostgreSQL        ACP Broker
       (3 priority lanes)  dark_factory      :8000
                           schema
                                │
           ┌────────────────────┼────────────────────┐
           ▼                    ▼                    ▼
  adapter-claudecode   adapter-opencode   adapter-openhands
       :8004                :8001              :8002
           │                    │                    │
   claude subprocess    opencode :4096      openhands :3000
           │                    │                    │
           ▼                    ▼                    ▼
    adapter-codex        adapter-qwencode
        :8003                 :8005
           │                    │
   codex subprocess    qwen-code subprocess
           │
           ▼
  ┌────────────────────────────────────┐
  │  Model backends (direct, no proxy) │
  │                                    │
  │  Ollama :11434   →  local models   │
  │    Anthropic API →  cc-sonnet      │
  │    OpenAI API    →  cx-gpt4o       │
  └────────────────────────────────────┘
```

## Data Flow: Task to PR

```
1. POST /tasks {title, description}
   → Orchestrator stores in PostgreSQL; LPUSH to Redis queue

2. Worker loop: BRPOP from Redis
   → Decompose via cc-sonnet → [{subtask_type, complexity, description, dependencies}]

3. Route each subtask to worker_id (routing ConfigMap)
   → POST /runs to ACP Broker {worker_id, input}
   → Broker injects: model, api_base, api_key from worker registry
   → Broker proxies to adapter; adapter spawns CLI subprocess

4. CLI subprocess runs with:
   - Local worker:  ANTHROPIC_BASE_URL=http://ollama:11434 (claude-code)
                    OPENAI_BASE_URL=http://ollama:11434/v1  (others)
   - Cloud worker:  ANTHROPIC_API_KEY or OPENAI_API_KEY (no base URL override)

5. Adapter streams ACP events → broker → orchestrator
   → run record written: worker_id, agent_cli, model, status, tokens, cost

6. Git worktree: /workspaces/{worker_id}/{run_id}/ on branch worker/{worker_id}/{run_id}

7. All subtasks complete → synthesis via cc-sonnet → PR created
8. Post-run: write worker_evaluations row
```

## Component Reference

### Orchestrator

| Property | Value |
|---|---|
| Image | Custom Python 3.12 |
| Port | 8080 (internal); NodePort configurable at install time |
| API | `POST /tasks`, `GET /tasks/{id}`, `POST /ingest/github`, `GET /health`, `GET /metrics` |
| Auth | Bearer token (`DARK_FACTORY_API_KEY`) on all write endpoints |
| Deps | Redis (queue), PostgreSQL (store), ACP Broker (dispatch) |

### ACP Broker

| Property | Value |
|---|---|
| Image | Custom Python 3.12 (`acp-sdk`, `httpx`, `pyyaml`, `redis`) |
| Port | 8000 (ClusterIP) |
| Worker registry | `workers.yaml` ConfigMap — hot-reloaded every 60s |
| Model injection | Reads `api_base`, `api_key` from worker config + Secret; injects per-run into adapter env |
| No LiteLLM | Adapters talk directly to Ollama or cloud API |

### Adapters

| Adapter | Port | CLI invocation | Local backend | Cloud backend |
|---|---|---|---|---|
| adapter-claudecode | :8004 | `claude -p --bare --output-format stream-json` | `ANTHROPIC_BASE_URL=http://ollama:11434` | `ANTHROPIC_API_KEY` |
| adapter-codex | :8003 | `codex exec --full-access` | `OPENAI_BASE_URL=http://ollama:11434/v1` | `OPENAI_API_KEY` |
| adapter-opencode | :8001 | `opencode serve :4096` + REST | `OPENAI_BASE_URL=http://ollama:11434/v1` | — |
| adapter-openhands | :8002 | REST → openhands :3000 | `LLM_BASE_URL=http://ollama:11434/v1` | — |
| adapter-qwencode | :8005 | `qwen-code -p` | `OPENAI_BASE_URL=http://ollama:11434/v1` | — |

### Secrets (minimal)

Only two cloud API keys needed in `dark-factory-api-keys` Secret:

```bash
ANTHROPIC_API_KEY=sk-ant-...   # cc-sonnet only
OPENAI_API_KEY=sk-...          # cx-gpt4o only
DARK_FACTORY_API_KEY=...       # orchestrator auth
```

Local workers use `OLLAMA_BASE_URL` from ConfigMap — no secret needed.

### Git Workspace Isolation

```
/workspaces/                         (PVC: dark-factory-workspaces)
  _base/{task_id}/                   base clone; read-only
  {worker_id}/{run_id}/              git worktree; branch: worker/{worker_id}/{run_id}
```

## Task Routing Table (v1)

| Task Type | Complexity | Primary Worker | Fallback |
|---|---|---|---|
| decomposition | any | `cc-sonnet` | — |
| synthesis | any | `cc-sonnet` | — |
| code-review | any | `cc-sonnet` | `cx-gpt4o` |
| execution | any | `oh-devstral` | — |
| test-gen | any | `oh-devstral` | `oh-qwen-coder` |
| code-gen | high | `cc-sonnet` | `cx-gpt4o` |
| code-gen | medium | `cc-qwen-coder-80b` | `cc-sonnet` |
| code-gen | low | `oc-qwen-coder-next` | `cc-qwen-coder-80b` |
| refactor | any | `oc-qwen-coder-next` | `cc-qwen-coder-80b` |

Routing is a ConfigMap entry — adjust without redeploying.

## Database Schema (`dark_factory`)

| Table | Purpose | Notes |
|---|---|---|
| `tasks` | One row per incoming task | Status: PENDING → COMPLETED |
| `runs` | One row per ACP run | Tagged: `worker_id`, `agent_cli`, `model`, `git_branch` |
| `artifacts` | Code diffs, test output, PR descriptions | Keyed to runs |
| `worker_evaluations` | Per-run perf: success, duration, cost, quality | **Never truncate** |

## Observability

All metrics carry `worker_id` label.

| Metric | Key labels |
|---|---|
| `dark_factory_runs_total` | `worker_id`, `agent_cli`, `model`, `task_type`, `status` |
| `dark_factory_run_duration_seconds` | `worker_id`, `task_type` |
| `dark_factory_run_cost_usd` | `worker_id`, `model` (calculated from token counts × pricing table) |
| `dark_factory_worker_success_rate` | `worker_id`, `task_type` (rolling 24h gauge) |
| `dark_factory_tasks_total` | `status`, `task_type` |
| `dark_factory_queue_depth` | `queue` |

**Grafana dashboard uid:** `dark-factory-overview`

## Routing Improvement Query

```sql
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

## Deploy / Teardown

```bash
cd ~/src/home_infra/dark-factory

export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export DARK_FACTORY_API_KEY="$(openssl rand -hex 20)"
export OLLAMA_BASE_URL="http://10.0.0.7:11434"
export DATABASE_URL="postgresql://..."
export REDIS_URL="redis://..."

./install.sh
./install.sh --dry-run
./test.sh
./test.sh --smoke-test

./uninstall.sh --force
./uninstall.sh --delete-data --delete-namespace --force
```

## Teardown / Reinstall Validation

| Cycle | Date | Teardown clean | Tests | Notes |
|---|---|---|---|---|
| 1 | TBD | TBD | TBD | |
| 2 | TBD | TBD | TBD | |
| 3 | TBD | TBD | TBD | |
