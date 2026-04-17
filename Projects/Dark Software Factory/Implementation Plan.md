# Dark Software Factory — Implementation Plan

> Design doc: [[Architecture]]
> Worker registry: [[Worker Registry]]
> Full plan: `~/src/home_infra/docs/dark-factory/implementation-plan.md`

---

## Milestone Summary

| Phase | What's built | Exit criteria |
|---|---|---|
| **0** | Blocking questions resolved | Q1–Q6 answered; package versions pinned |
| **1** | Namespace, DB schema, secrets | `dark-factory` ns; `dark_factory` PG schema; 3 secrets |
| **2** | ACP Broker | `GET /agents` 200; `POST /runs` 201 with run_id |
| **3** | OpenCode adapter | ACP run dispatches to OpenCode; COMPLETED |
| **4** | OpenHands adapter | ACP run dispatches to sandbox; COMPLETED |
| **5** | Codex adapter | `codex exec` result returned |
| **6** | Claude Code adapter | `claude -p --bare` result returned |
| **6.5** | Qwen Code adapter | `qwen-code -p` result returned |
| **7** | Orchestrator | `POST /tasks` 202; full decompose → route → aggregate → PR |
| **8** | Observability | Prometheus metrics scraping; Grafana `dark-factory-overview` live |
| **9** | Validation | 59/59 tests passing; 3 clean destructive teardown/reinstall cycles |

---

## Phase 0 — Blocking Questions

### Q1 — ACP SDK version

```bash
curl -s https://pypi.org/pypi/acp-sdk/json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['info']['version'])"
```

### Q2 — OpenCode image and serve API shape

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

Prefer `SANDBOX_TYPE=k8s`. Fall back to Docker socket hostPath if unavailable.

### Q4 — Claude Code npm package and `--bare` flag

```bash
npm view @anthropic-ai/claude-code version
npx @anthropic-ai/claude-code@<version> --bare --help 2>&1 | grep bare
```

**Validated:** direct Ollama works with Claude Code when `ANTHROPIC_BASE_URL=http://<ollama>:11434`. Ollama 0.20.4 implements `/v1/messages` natively including structured tool_use blocks.

### Q5 — Codex CLI package and headless flag

```bash
npm view @openai/codex version
npx @openai/codex@<version> exec --help 2>&1 | grep -E "full.access|headless"
```

### Q6 — Qwen Code npm package and headless flag

```bash
npm view @qwen-code/qwen-code version 2>/dev/null || npm view qwen-code version 2>/dev/null
npx @qwen-code/qwen-code@<version> --help 2>&1 | head -20
npx @qwen-code/qwen-code@<version> -p "write hello world" 2>&1 | head -10
# Confirm: headless flag, output format, JSON event schema
```

---

## Phase 1 — Scaffold, Namespace, DB Schema, Secrets

### Directory structure

```
~/src/home_infra/dark-factory/
├── install.sh / uninstall.sh / test.sh / diag.sh / submit-task.sh
├── broker/        install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── adapters/
│   ├── opencode/   install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── openhands/  install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── codex/      install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   ├── claudecode/ install.sh, uninstall.sh, test.sh, diag.sh, manifests/
│   └── qwencode/   install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── orchestrator/  install.sh, uninstall.sh, test.sh, diag.sh, manifests/
├── observability/ install.sh, uninstall.sh, test.sh, manifests/
└── db/schema.sql, migrate.sh
```

### Environment variables at install time

```bash
# Cloud API keys (only for cloud workers cc-sonnet and cx-gpt4o)
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."

# Factory auth token
export DARK_FACTORY_API_KEY="$(openssl rand -hex 20)"

# Infrastructure (cluster-configurable)
export OLLAMA_BASE_URL="http://10.0.0.7:11434"
export DATABASE_URL="postgresql://postgres:<pass>@postgresql.postgresql.svc.cluster.local:5432/homelab"
export REDIS_URL="redis://redis.redis.svc.cluster.local:6379"
```

`install.sh` creates the `dark-factory-api-keys` Secret with just these three keys — no LiteLLM keys needed.

### PostgreSQL schema

Four tables: `tasks`, `runs` (with `worker_id`, `agent_cli`, `model`, `git_branch`), `artifacts`, `worker_evaluations`. Full SQL in the implementation plan source file at `~/src/home_infra/docs/dark-factory/implementation-plan.md`.

`worker_evaluations` is the long-lived routing signal — **never truncate it**.

---

## Phase 2 — ACP Broker

- Python 3.12; packages: `acp-sdk`, `httpx`, `pyyaml`, `redis`, `prometheus_client`
- Loads `workers.yaml` ConfigMap; hot-reloads every 60s
- On `POST /runs {worker_id, input}`: looks up worker → injects `model`, `api_base`, `api_key` into run metadata → proxies to adapter → relays SSE
- `api_key` injected from `dark-factory-api-keys` Secret (cloud workers only); empty string for local workers
- No LiteLLM calls — model routing is entirely per-adapter env var injection

---

## Phase 3 — OpenCode Adapter

Sidecar pattern — two containers in one Pod:
- `opencode-server`: `opencode serve --port 4096`; env `OPENAI_API_BASE` from injected `api_base`
- `adapter`: Python ACP server :8001; REST client to `localhost:4096`

Model and api_base injected per-run from broker metadata into container env.

---

## Phase 4 — OpenHands Adapter

- `openhands-server :3000` + Python ACP adapter :8002
- Env: `LLM_BASE_URL` from injected api_base; `LLM_MODEL` from injected model
- RBAC: ServiceAccount with `jobs/create` + `pods/get` in `dark-factory` namespace (k8s sandbox)

---

## Phase 5 — Codex Adapter

- Single container: `node:22-slim` + Python 3.12 + `@openai/codex@<pinned>`
- Subprocess: `codex exec --full-access <task>`
- Env injected per-run: `OPENAI_BASE_URL` (Ollama `/v1` or empty for cloud), `OPENAI_API_KEY`
- 120s timeout; non-zero exit → FAILED

---

## Phase 6 — Claude Code Adapter

- Single container: `node:22-slim` + Python 3.12 + `@anthropic-ai/claude-code@<pinned>` + git
- Subprocess: `claude -p <task> --bare --output-format stream-json --allowedTools Read,Edit,Bash`
- Local workers: `ANTHROPIC_BASE_URL=http://ollama:11434`, `ANTHROPIC_API_KEY=ollama`
- Cloud worker (cc-sonnet): no `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY` from Secret
- Both injected per-run by broker; same adapter pod serves all cc-* workers

**Validated (2026-04-17):** `cc-qwen-coder-next` with direct Ollama correctly executes Write tool and creates files. Direct Ollama path works; LiteLLM path broke tool_use translation.

---

## Phase 6.5 — Qwen Code Adapter

- Single container: `node:22-slim` + Python 3.12 + `@qwen-code/qwen-code@<pinned>` + git
- Subprocess: `qwen-code -p <task>` (flags confirmed in Q6)
- Env: `OPENAI_BASE_URL=http://ollama:11434/v1`, `OPENAI_API_KEY=ollama`, `QWEN_CODE_MODEL` from injected model
- All qc-* workers are local-only

> Output JSON parser finalised after Q6 resolves exact schema.

---

## Phase 7 — Orchestrator

**Worker loop:**
1. `BRPOP` from Redis (priority → normal → batch)
2. Decompose via `cc-sonnet` → JSON subtask list
3. Topological sort by `dependencies`
4. Per subtask: look up `worker_id` from routing ConfigMap → `POST /runs` to broker
5. Poll until COMPLETED or FAILED; on FAILED retry with fallback worker
6. Synthesis via `cc-sonnet`; PR created
7. Write `worker_evaluations` row — tokens from API response, cost from static pricing table

**Cost estimation (no LiteLLM):**
```python
COST_PER_1M = {
    "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
    "gpt-4o":            {"input": 2.50, "output": 10.00},
    # local models
    "_local":            {"input": 0.0,  "output": 0.0},
}
```

**Routing ConfigMap:**
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

Alloy patch (if available), else ServiceMonitor CRD. Grafana dashboard `dark-factory-overview` uploaded via HTTP API.

Key metrics all carry `worker_id` label — the primary evaluation dimension.

---

## Phase 9 — Validation

**59-test suite** (reduced from 63 — 4 LiteLLM integration tests removed).

**3 destructive cycles:**
```bash
./uninstall.sh --delete-data --delete-namespace --force
./install.sh && ./test.sh   # verify 59/59 pass
```

---

## Install Order

```
1. namespace + secrets + DB schema
2. broker
3. adapter-opencode
4. adapter-openhands
5. adapter-codex
6. adapter-claudecode
7. adapter-qwencode
8. orchestrator
9. observability
10. test.sh
```
