# Running the Factory

> Related: [[Overview]] · [[Implementation Plan]] · [[Worker Registry]]

---

## Prerequisites

- Kubernetes cluster running (homelab k3s on `melody-beast`, `10.0.0.7`)
- Redis and PostgreSQL already deployed in cluster
- Ollama running at `http://10.0.0.7:11434` with required models loaded
- `kubectl` configured and pointing at the cluster
- Cloud API keys for `cc-sonnet` and `cx-gpt4o` workers (local-only workers need neither)

---

## Install

### 1. Set environment variables

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export DARK_FACTORY_API_KEY="$(openssl rand -hex 20)"
```

Save `DARK_FACTORY_API_KEY` — you'll need it to submit tasks and check status.

### 2. Run the install script

```bash
cd ~/src/home_infra/dark-factory
./install.sh
```

Install order (automated):
1. Namespace `dark-factory`
2. Secrets (`dark-factory-api-keys`)
3. DB schema migration (PostgreSQL `dark_factory` schema)
4. ACP Broker
5. OpenCode adapter
6. OpenHands adapter
7. Codex adapter
8. Claude Code adapter
9. Qwen Code adapter
10. Orchestrator
11. Observability (Prometheus ServiceMonitor + Grafana dashboard)

Takes ~5 minutes. Each component waits for rollout before proceeding.

### 3. Verify

```bash
./test.sh
```

---

## Submit a Task

```bash
export DARK_FACTORY_API_KEY="<your key>"

./submit-task.sh "Add input validation to user signup" \
  "The /api/signup endpoint needs email format validation and password strength check"
```

Returns:
```json
{"task_id": "abc-123", "status": "PENDING"}
```

### Check task status

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

curl -s -H "Authorization: Bearer $DARK_FACTORY_API_KEY" \
  http://$NODE_IP:32304/tasks/<task_id> | python3 -m json.tool
```

Task status lifecycle: `PENDING → DECOMPOSING → RUNNING → COMPLETED / FAILED`

### Direct API

The orchestrator NodePort is `32304`. All endpoints require `Authorization: Bearer <DARK_FACTORY_API_KEY>`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/tasks` | Submit task — body: `{title, description, task_type}` |
| `GET` | `/tasks/{task_id}` | Get task status and metadata |
| `POST` | `/ingest/github` | GitHub webhook ingestion (validates `X-Hub-Signature-256`) |
| `GET` | `/metrics` | Prometheus metrics |
| `GET` | `/health` | Health check |

Broker internal endpoints (port `8000`, in-cluster only):

| Method | Path | Description |
|---|---|---|
| `GET` | `/agents` | List all registered workers |
| `POST` | `/runs` | Dispatch a run to a worker — body: `{worker_id, input}` |
| `GET` | `/health` | Health check |
| `GET` | `/metrics` | Prometheus metrics |

---

## Routing

The orchestrator routes subtasks based on task type and complexity using the routing ConfigMap. Defaults:

| Task type | Primary worker | Fallback |
|---|---|---|
| `decomposition` | `cc-sonnet` | — |
| `synthesis` | `cc-sonnet` | — |
| `code-review` | `cc-sonnet` | `cx-gpt4o` |
| `code-gen-high` | `cc-sonnet` | `cx-gpt4o` |
| `code-gen-med` | `cc-qwen-coder-80b` | `cc-sonnet` |
| `code-gen-low` | `oc-qwen-coder-next` | `cc-qwen-coder-80b` |
| `execution` | `oh-devstral` | `oh-qwen-coder` |
| `test-gen` | `oh-devstral` | `oh-qwen-coder` |
| `refactor` | `oc-qwen-coder-next` | `cc-qwen-coder-80b` |

To adjust routing without redeployment:
```bash
kubectl edit configmap acp-broker-routing -n dark-factory
```

The broker hot-reloads the config every 60 seconds.

---

## Observability

**Grafana dashboard:** `dark-factory-overview`

Key panels:
- Task Throughput (5m rate)
- Queue Depth by lane (priority / normal / batch)
- Task Duration p50/p95 by task type
- Runs by Worker (24h)
- Worker Success Rate per `worker_id`

Prometheus metrics (all carry `worker_id` label):

| Metric | Type | Description |
|---|---|---|
| `dark_factory_tasks_total` | Counter | by status, task_type |
| `dark_factory_task_duration_seconds` | Histogram | by task_type |
| `dark_factory_runs_total` | Counter | by worker_id, agent_cli, status |
| `dark_factory_queue_depth` | Gauge | by queue |
| `dark_factory_broker_runs_total` | Counter | by worker_id, agent_cli, status |
| `dark_factory_broker_run_duration_seconds` | Histogram | by worker_id, agent_cli |
| `dark_factory_worker_success_rate` | Gauge | rolling rate by worker_id |

---

## Diagnose

```bash
# Check all pods
kubectl get pods -n dark-factory

# Orchestrator logs
kubectl logs -n dark-factory deploy/orchestrator -f

# Broker logs
kubectl logs -n dark-factory deploy/acp-broker -f

# Specific adapter logs
kubectl logs -n dark-factory deploy/adapter-claudecode -f

# Run diag scripts
./diag.sh
./broker/diag.sh
./orchestrator/diag.sh
```

---

## Teardown

```bash
# Remove deployments, keep data and namespace
./uninstall.sh --force

# Full wipe — deletes namespace, data, and schema
./uninstall.sh --force --delete-data --delete-namespace
```

---

## What's Not Done Yet (Phase 9)

- Test suite (`./test.sh`) has not been run against the live stack — 59 tests need to pass
- 3 destructive teardown/reinstall cycles not yet validated
- Subtask execution routing uses `preferred_agent` (CLI name) from decomposition output, but the broker expects a `worker_id` — this mismatch will cause 404s on subtask dispatch until the routing lookup in `orchestrator/src/main.py` is wired to the ConfigMap properly
- Agent loop detection (see [[Requirements - Agent Loop Detection]]) not yet implemented
