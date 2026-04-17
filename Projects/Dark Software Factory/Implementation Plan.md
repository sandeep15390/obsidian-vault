# Dark Software Factory — Implementation Plan

> Design doc: [[Architecture]]
> Worker registry: [[Worker Registry]]
> Full plan: `~/src/home_infra/docs/dark-factory/implementation-plan.md`

---

## Milestone Summary

| Phase | What's built | Exit criteria |
|---|---|---|
| **0** | Blocking questions resolved; versions pinned | All Q1–Q7 answered; package versions recorded |
| **1** | Namespace, DB schema, 13 LiteLLM virtual keys, secrets | `dark-factory` ns; `dark_factory` PG schema; 13 keys in Secret |
| **2** | ACP Broker | `GET /agents` 200; `POST /runs` 201 with run_id |
| **3** | OpenCode adapter | ACP run dispatches to OpenCode; COMPLETED returned |
| **4** | OpenHands adapter | ACP run dispatches to sandbox; COMPLETED returned |
| **5** | Codex adapter | `codex exec` result returned |
| **6** | Claude Code adapter | `claude -p --bare` result returned |
| **6.5** | Qwen Code adapter | `qwen-code -p` result returned via Qwen model |
| **7** | Orchestrator | `POST /tasks` 202; full decompose → route → aggregate → PR |
| **8** | Observability | Prometheus metrics scraping; Grafana `dark-factory-overview` live |
| **9** | Validation | 63/63 tests passing; 3 clean destructive teardown/reinstall cycles |

---

## Phase 0 — Blocking Questions

Resolve all of these before writing implementation code. Pin exact package versions.

### Q1 — ACP SDK

```bash
curl -s https://pypi.org/pypi/acp-sdk/json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['info']['version'])"
# If package name differs: pip index versions beeai-acp
```

Pin version in all adapter Dockerfiles.

### Q2 — OpenCode image and serve API

```bash
curl -s https://api.github.com/repos/opencode-ai/opencode/releases/latest \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tag_name'])"
docker run --rm -p 4096:4096 ghcr.io/opencode-ai/opencode:<tag> serve --port 4096
curl http://localhost:4096/doc | head -30
# Confirm: /session and /session/{id}/message endpoints exist
```

### Q3 — OpenHands workspace runtime

```bash
docker run --rm ghcr.io/openhands/agent-server:latest-python --help 2>&1 \
  | grep -i "kubernetes\|sandbox\|runtime\|docker"
```

Use `SANDBOX_TYPE=k8s` (k8s Job per task) if available — avoids Docker socket hostPath mount. If not available, plan Docker socket mount with dedicated ServiceAccount.

### Q4 — Claude Code npm package and `--bare` flag

```bash
npm view @anthropic-ai/claude-code version
npx @anthropic-ai/claude-code@<version> --bare --help 2>&1 | grep bare
# Confirm CLI is invoked as 'claude'
```

### Q5 — LiteLLM key creation API

```bash
curl -s -X POST "http://${LITELLM_HOST}/key/generate" \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"key_alias":"probe-test","max_budget":0.01,"duration":"1m"}' | python3 -m json.tool
# Expect: {"key": "sk-...", ...}; immediately delete the probe key
```

### Q6 — Codex CLI package and headless flag

```bash
npm view @openai/codex version
npx @openai/codex@<version> exec --help 2>&1 | grep -E "full.access|headless|approval"
```

### Q7 — Qwen Code npm package and headless flag ⚠️

```bash
npm view @qwen-code/qwen-code version 2>/dev/null || npm view qwen-code version 2>/dev/null
npx @qwen-code/qwen-code@<version> --help 2>&1 | head -20
# Confirm: headless flag (-p or --prompt), output format flag, JSON event schema
npx @qwen-code/qwen-code@<version> -p "write hello world" --output-format stream-json 2>&1 | head -5
```

Output JSON schema of Qwen Code must be confirmed before writing the adapter parser.

---

## Phase 1 — Scaffold, Namespace, DB Schema, Secrets

### Directory structure

```
~/src/home_infra/dark-factory/
├── install.sh / uninstall.sh / test.sh / diag.sh / submit-task.sh
├── broker/
│   └── install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── adapters/
│   ├── opencode/   install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── openhands/  install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── codex/      install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── claudecode/ install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   └── qwencode/   install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── orchestrator/
│   └── install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── observability/
│   └── install.sh, uninstall.sh, test.sh, manifests/
└── db/
    └── schema.sql, migrate.sh
```

### Environment variables required at install time

```bash
# Cloud API keys (only needed for cloud workers)
export ANTHROPIC_API_KEY="sk-ant-..."     # cc-sonnet
export OPENAI_API_KEY="sk-..."           # cx-gpt4o

# Factory auth
export DARK_FACTORY_API_KEY="$(openssl rand -hex 20)"

# Infrastructure (cluster-configurable; these are homelab defaults)
export LITELLM_BASE_URL="http://litellm.litellm.svc.cluster.local:4000"
export LITELLM_MASTER_KEY="..."
export DATABASE_URL="postgresql://postgres:<pass>@postgresql.postgresql.svc.cluster.local:5432/homelab"
export REDIS_URL="redis://redis.redis.svc.cluster.local:6379"
```

### PostgreSQL schema (`dark_factory`)

```sql
CREATE SCHEMA IF NOT EXISTS dark_factory;

-- tasks: one row per incoming task
CREATE TABLE IF NOT EXISTS dark_factory.tasks (
    task_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status        TEXT NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','DECOMPOSING','RUNNING','COMPLETED','FAILED','TIMEOUT')),
    task_type     TEXT,
    title         TEXT NOT NULL,
    description   TEXT,
    source        TEXT,        -- http | github_webhook | cli
    source_ref    TEXT,        -- PR number, issue URL, etc.
    result_url    TEXT,        -- PR URL when completed
    error_message TEXT,
    metadata      JSONB
);

-- runs: one row per ACP run; tagged with worker identity
CREATE TABLE IF NOT EXISTS dark_factory.runs (
    run_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id            UUID NOT NULL REFERENCES dark_factory.tasks(task_id) ON DELETE CASCADE,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status             TEXT NOT NULL DEFAULT 'created'
                         CHECK (status IN ('created','running','completed','failed','cancelled')),
    worker_id          TEXT NOT NULL,   -- e.g. cc-sonnet, qc-qwen-coder-80b
    agent_cli          TEXT NOT NULL,   -- claude-code | codex | opencode | openhands | qwen-code
    model              TEXT NOT NULL,   -- LiteLLM model identifier
    subtask_type       TEXT,
    prompt_tokens      INTEGER,
    completion_tokens  INTEGER,
    estimated_cost_usd NUMERIC(10,6),
    acp_run_id         TEXT,
    git_branch         TEXT,            -- worker/{worker_id}/{run_id}
    workspace_path     TEXT,            -- /workspaces/{worker_id}/{run_id}
    error_message      TEXT,
    metadata           JSONB
);

-- artifacts: code diffs, test outputs, PR descriptions
CREATE TABLE IF NOT EXISTS dark_factory.artifacts (
    artifact_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id        UUID NOT NULL REFERENCES dark_factory.runs(run_id) ON DELETE CASCADE,
    task_id       UUID NOT NULL REFERENCES dark_factory.tasks(task_id) ON DELETE CASCADE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    artifact_type TEXT NOT NULL,   -- code | diff | test_output | pr_description | raw_output
    content_type  TEXT NOT NULL,   -- text/plain | text/x-diff | application/json
    content       TEXT,
    content_url   TEXT
);

-- worker_evaluations: NEVER TRUNCATE — this is the routing improvement signal
CREATE TABLE IF NOT EXISTS dark_factory.worker_evaluations (
    eval_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id             UUID NOT NULL REFERENCES dark_factory.runs(run_id) ON DELETE CASCADE,
    task_id            UUID NOT NULL REFERENCES dark_factory.tasks(task_id) ON DELETE CASCADE,
    worker_id          TEXT NOT NULL,
    agent_cli          TEXT NOT NULL,
    model              TEXT NOT NULL,
    task_type          TEXT NOT NULL,
    succeeded          BOOLEAN NOT NULL,
    duration_ms        INTEGER,
    prompt_tokens      INTEGER,
    completion_tokens  INTEGER,
    estimated_cost_usd NUMERIC(10,6),
    tests_passed       BOOLEAN,      -- null until scored
    diff_lines_added   INTEGER,
    diff_lines_removed INTEGER,
    reviewer_score     SMALLINT,     -- 0-5; null until scored
    scorer_notes       TEXT,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS runs_worker_id_idx   ON dark_factory.runs(worker_id);
CREATE INDEX IF NOT EXISTS evals_worker_task_idx ON dark_factory.worker_evaluations(worker_id, task_type);
CREATE INDEX IF NOT EXISTS evals_created_idx     ON dark_factory.worker_evaluations(created_at DESC);
```

### LiteLLM virtual keys

`install.sh` creates 13 keys idempotently. Idempotent: if alias already exists, fetch and reuse. All keys patched into `dark-factory-api-keys` Secret. See [[Worker Registry]] for full table.

---

## Phase 2 — ACP Broker

- Python 3.12 image; packages: `acp-sdk`, `httpx`, `pyyaml`, `redis`, `prometheus_client`
- Reads `workers.yaml` from ConfigMap; hot-reloads every 60 seconds
- `POST /runs {worker_id, input}` → validates worker in registry → injects model + litellm_key → proxies to adapter → relays SSE back
- `GET /agents` → aggregates from all registered adapters
- Persists run-id → worker mapping in Redis for restart resilience
- Exposes `/metrics` (Prometheus) and `/health`

---

## Phase 3 — OpenCode Adapter

Sidecar pattern (two containers in one Pod):
- Container 1: `opencode serve --port 4096` (image: pinned opencode tag)
- Container 2: Python ACP server :8001; REST client to localhost:4096
- Model + litellm key injected per-run from broker; no static model in pod spec

---

## Phase 4 — OpenHands Adapter

- Container 1: `openhands-server :3000` (k8s Job sandbox if Q3 supports it; else Docker socket)
- Container 2: Python ACP adapter :8002; WebSocket client to openhands-server
- RBAC: ServiceAccount with `jobs/create` + `pods/get` + `pods/list` in `dark-factory` namespace (k8s sandbox mode)

---

## Phase 5 — Codex Adapter

- Single container: `node:22-slim` + Python 3.12 + `@openai/codex@<pinned>`
- Subprocess: `codex exec --full-access <task>`; `OPENAI_BASE_URL` → LiteLLM
- 120s subprocess timeout; non-zero exit → FAILED with stderr

---

## Phase 6 — Claude Code Adapter

- Single container: `node:22-slim` + Python 3.12 + `@anthropic-ai/claude-code@<pinned>` + git
- Subprocess: `claude -p <task> --bare --output-format stream-json --allowedTools Read,Edit,Bash`
- `ANTHROPIC_BASE_URL` → LiteLLM; per-run working directory: `/workspaces/cc-*/{run_id}/`

---

## Phase 6.5 — Qwen Code Adapter

- Single container: `node:22-slim` + Python 3.12 + `@qwen-code/qwen-code@<pinned>` + git
- Subprocess: `qwen-code -p <task> --output-format stream-json` (flags confirmed in Q7)
- `OPENAI_BASE_URL` → LiteLLM (Qwen Code uses OpenAI-compatible API); no cloud key needed
- Model injected per-run via `QWEN_CODE_MODEL` env var set by broker dispatch

> Output JSON parser must be finalised after Q7 — may differ from claude-code's `stream-json` schema.

---

## Phase 7 — Orchestrator

**Worker loop:**
1. `BRPOP` from Redis (priority → normal → batch)
2. Decompose via `cc-sonnet` → structured JSON: `[{subtask_type, complexity, description, dependencies}]`
3. Topological sort subtasks by `dependencies`
4. For each subtask: look up `worker_id` from routing ConfigMap → `POST /runs` to broker
5. Poll `GET /runs/{id}` until COMPLETED or FAILED; on FAILED retry with fallback worker
6. Collect artifacts; synthesise via `cc-sonnet` → PR description
7. Write `worker_evaluations` row for every run (synchronous, same loop)

**Routing ConfigMap** (update without redeployment):

```yaml
routing:
  decomposition: {primary: cc-sonnet}
  synthesis:     {primary: cc-sonnet}
  code-review:   {primary: cc-sonnet,          fallback: cx-gpt4o}
  execution:     {primary: oh-devstral}
  test-gen:      {primary: oh-devstral,         fallback: oh-qwen-coder}
  code-gen-high: {primary: cc-sonnet,           fallback: cx-gpt4o}
  code-gen-med:  {primary: cc-qwen-coder-80b,   fallback: cc-sonnet}
  code-gen-low:  {primary: oc-qwen-coder-next,  fallback: cc-qwen-coder-80b}
  refactor:      {primary: oc-qwen-coder-next,  fallback: cc-qwen-coder-80b}
```

---

## Phase 8 — Observability

**Alloy patch** (if Alloy/Prometheus available on the cluster):

```
// # BEGIN dark-factory
prometheus.scrape "dark_factory_orchestrator" {
  targets = [{"__address__" = "orchestrator.dark-factory.svc.cluster.local:8080"}]
  job_name = "dark-factory-orchestrator"
  forward_to = [prometheus.remote_write.default.receiver]
}
prometheus.scrape "dark_factory_broker" {
  targets = [{"__address__" = "acp-broker.dark-factory.svc.cluster.local:8000"}]
  job_name = "dark-factory-broker"
  forward_to = [prometheus.remote_write.default.receiver]
}
// # END dark-factory
```

`observability/install.sh` checks whether Alloy is available before patching. On clusters without Alloy, installs a minimal Prometheus scrape config via ServiceMonitor CRD if available.

**Grafana dashboard uid:** `dark-factory-overview` — uploaded via Grafana HTTP API.

---

## Phase 9 — Validation

**63-test suite** covers: prerequisites, namespace, DB schema, broker ACP API, 5× adapter (pod ready + ACP smoke + run completed), orchestrator API, task routing, LiteLLM integration, Redis queue, end-to-end pipeline, observability.

**3 destructive teardown cycles:**
```bash
./uninstall.sh --delete-data --delete-namespace --force
# verify: namespace gone, Alloy patch stripped, dashboard 404, dark_factory schema absent
./install.sh
# verify: 63/63 pass
```

---

## Install / Uninstall Order

```
Install:   namespace+secrets → broker → opencode → openhands → codex → claudecode → qwencode → orchestrator → observability → test
Uninstall: observability → orchestrator → qwencode → claudecode → codex → openhands → opencode → broker → LiteLLM keys → secrets → namespace
```

## Image Build Strategy

No pre-built public images for custom components. Build and import into k3s:

```bash
docker build -t dark-factory/broker:v1 ./broker/
k3s ctr images import <(docker save dark-factory/broker:v1)
# Repeat per adapter; use imagePullPolicy: Never
```

For non-k3s clusters: push to a local registry (`localhost:5000/dark-factory/<image>:<tag>`).
