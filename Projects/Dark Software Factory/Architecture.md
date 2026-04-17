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
                    │  NodePort :8080        │
                    └───────────┬───────────┘
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
         Redis queue       PostgreSQL        ACP Broker
       (3 priority lanes)  dark_factory      :8000
                           schema
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       adapter-opencode  adapter-openhands  adapter-codex
            :8001              :8002             :8003
              │                 │
       opencode-server    openhands-server
            :4096              :3000
              ▼                 ▼                 ▼
       adapter-claudecode  adapter-qwencode  (all adapters)
            :8004              :8005              │
                                                  ▼
                                         LiteLLM proxy :4000
                                        ┌────────┴────────┐
                                        ▼                 ▼
                                  Ollama (local)    Cloud APIs
                               qwen2.5-coder:80b   claude-sonnet
                               qwen3-coder-next     gpt-4o
                               gpt-oss:120b         ...
                               devstral-small-2:24b
```

## Data Flow: Task to PR

```
1. POST /tasks {title, description}
   → Orchestrator stores task in PostgreSQL; LPUSH to Redis queue

2. Worker loop: BRPOP from Redis
   → Decompose via cc-sonnet (Claude Code + claude-sonnet-4-6)
   → Returns: [{subtask_type, complexity, description, dependencies}]

3. Route each subtask to a worker_id (routing table in ConfigMap)
   → POST /runs to ACP Broker {worker_id, input}
   → Broker injects model + litellm_key from worker registry
   → Broker proxies to adapter; adapter spawns CLI subprocess

4. Adapter streams ACP run events → broker → orchestrator
   → Run record updated in PostgreSQL (worker_id, agent_cli, model, status)
   → Git worktree: /workspaces/{worker_id}/{run_id}/ on branch worker/{worker_id}/{run_id}

5. All subtasks complete
   → Synthesis via cc-sonnet: merge subtask artifacts → PR description
   → CLI creates PR; URL stored in tasks.result_url

6. Post-run: write worker_evaluations row (duration, cost, diff size, success)
```

## Component Reference

### Orchestrator

| Property | Value |
|---|---|
| Image | Custom Python 3.12 |
| Port | 8080 (internal) |
| External access | NodePort — port configurable at install time |
| API | `POST /tasks`, `GET /tasks/{id}`, `POST /ingest/github`, `GET /health`, `GET /metrics` |
| Auth | Bearer token (`DARK_FACTORY_API_KEY`) on all write endpoints |
| Dependencies | Redis (queue), PostgreSQL (store), ACP Broker (dispatch) |

### ACP Broker

| Property | Value |
|---|---|
| Image | Custom Python 3.12 (`acp-sdk`, `httpx`, `pyyaml`, `redis`) |
| Port | 8000 (ClusterIP) |
| Worker registry | `workers.yaml` ConfigMap — hot-reloaded every 60s |
| Key injection | Reads LiteLLM virtual keys from mounted Secret; injects per-run |
| Persistence | Run-id → worker mapping in Redis (survives broker restart) |

### Adapters (5 total)

| Adapter | Port | CLI invocation | Key env var |
|---|---|---|---|
| adapter-opencode | 8001 | `opencode serve --port 4096` + REST | `OPENAI_API_BASE` → LiteLLM |
| adapter-openhands | 8002 | REST → openhands-server :3000 | `LLM_BASE_URL` → LiteLLM |
| adapter-codex | 8003 | `codex exec --full-access <task>` | `OPENAI_BASE_URL` → LiteLLM |
| adapter-claudecode | 8004 | `claude -p <task> --bare --output-format stream-json` | `ANTHROPIC_BASE_URL` → LiteLLM |
| adapter-qwencode | 8005 | `qwen-code -p <task> --output-format stream-json` | `OPENAI_BASE_URL` → LiteLLM |

All adapters receive model and LiteLLM key via ACP run metadata injected by the broker — no static model config in adapter pods.

### Git Workspace Isolation

```
/workspaces/                              (PVC: dark-factory-workspaces)
  _base/{task_id}/                        cloned once per task; read-only base
  {worker_id}/{run_id}/                   git worktree; branch: worker/{worker_id}/{run_id}
```

Worktrees created before subprocess launch; deleted after artifact stored. Branches persist for post-run review.

## Task Routing Table (v1)

Deterministic rules — first match wins. ConfigMap-driven; no redeployment to adjust.

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

## Database Schema (`dark_factory`)

| Table | Purpose | Never truncate? |
|---|---|---|
| `tasks` | One row per incoming task; status PENDING → COMPLETED | No |
| `runs` | One row per ACP run; tagged with `worker_id`, `agent_cli`, `model`, `git_branch` | No |
| `artifacts` | Code diffs, test output, PR descriptions keyed to runs | No |
| `worker_evaluations` | Per-run perf record: success, duration, cost, quality score | **Yes — routing signal** |

Key columns on `runs`: `worker_id`, `agent_cli`, `model`, `estimated_cost_usd`, `git_branch`, `workspace_path`.

## Observability

All metrics carry `worker_id` label — the primary evaluation dimension in Grafana.

| Metric | Type | Key labels |
|---|---|---|
| `dark_factory_runs_total` | Counter | `worker_id`, `agent_cli`, `model`, `task_type`, `status` |
| `dark_factory_run_duration_seconds` | Histogram | `worker_id`, `agent_cli`, `model`, `task_type` |
| `dark_factory_run_cost_usd` | Histogram | `worker_id`, `agent_cli`, `model` |
| `dark_factory_worker_success_rate` | Gauge | `worker_id`, `task_type` (rolling 24h) |
| `dark_factory_tasks_total` | Counter | `status`, `task_type` |
| `dark_factory_queue_depth` | Gauge | `queue` (priority/normal/batch) |

**Grafana dashboard uid:** `dark-factory-overview`

Dashboard rows:
1. Pipeline health — task throughput, queue depth, duration heatmap
2. Worker comparison — success rate × worker × task_type; duration and cost per worker; runs dispatched pie
3. Error analysis — failed runs by worker; fallback activations

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

When a local worker hits ≥ 85% success on a task type with comparable quality to the cloud worker, promote it to primary in the routing ConfigMap.

## Deploy / Teardown

```bash
cd ~/src/home_infra/dark-factory

# Required env vars
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export DARK_FACTORY_API_KEY="$(openssl rand -hex 20)"
export LITELLM_BASE_URL="http://litellm.litellm.svc.cluster.local:4000"
export LITELLM_MASTER_KEY="..."
export DATABASE_URL="postgresql://..."
export REDIS_URL="redis://..."

./install.sh
./install.sh --dry-run
./test.sh
./test.sh --smoke-test
./diag.sh

./uninstall.sh --force
./uninstall.sh --delete-data --delete-namespace --force
```

## Teardown / Reinstall Validation

| Cycle | Date | Teardown clean | Tests | Notes |
|---|---|---|---|---|
| 1 | TBD | TBD | TBD | |
| 2 | TBD | TBD | TBD | |
| 3 | TBD | TBD | TBD | |
