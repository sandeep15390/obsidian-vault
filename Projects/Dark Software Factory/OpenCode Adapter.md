# OpenCode Adapter

> Part of: [[Overview]]
> Related: [[Architecture]], [[Worker Registry]], [[Implementation Plan]]

The OpenCode adapter bridges the ACP broker and the [opencode](https://github.com/sst/opencode) AI coding assistant. It runs opencode in headless server mode and translates ACP run requests into opencode's HTTP API calls, returning results as SSE events.

---

## What It Does

The adapter is a single container that runs two processes:

1. **opencode serve** — the opencode binary running in headless server mode, exposing a local REST API on port 4096
2. **FastAPI adapter** — a Python HTTP server on port 8001 that accepts ACP-style requests from the broker, forwards them to the opencode server, and streams results back as SSE

The broker only ever talks to the FastAPI adapter on port 8001. The opencode server is an internal implementation detail.

---

## Request Flow

```
Broker
  │
  │  POST /runs  {input: "...", worker_id: "oc-devstral"}
  ▼
FastAPI Adapter :8001
  │
  │  POST /session          {"title": "run_<timestamp>"}
  │  ← {"id": "ses_..."}
  │
  │  POST /session/{id}/message
  │       {"parts": [{"type": "text", "text": "<input>"}]}
  │  ← {"info": {...}, "parts": [...]}   ← synchronous, blocks until done
  │
  ▼
opencode serve :4096
  │
  │  OpenAI-compatible chat request
  │  model: devstral-small-2:24b
  ▼
Ollama  http://10.0.0.7:11434/v1
```

After the opencode message call returns, the adapter emits two SSE events back to the broker:

```
data: {"type": "run_status", "status": "completed"}

data: {"type": "run_output", "output": {"parts": [{"content": "...", "content_type": "text/plain"}]}}
```

---

## Key Design Decisions

### opencode API is synchronous

Unlike most agent APIs that return a run ID and require polling, `POST /session/{id}/message` **blocks and returns the full result inline**. The adapter does not poll — it just waits on a single HTTP call (up to 300 s timeout) and parses the response immediately.

### Message format uses `parts`

The opencode API uses a `parts` array, not a plain `content` string:

```json
{
  "parts": [
    {"type": "text", "text": "your task here"}
  ]
}
```

The response similarly contains a `parts` array. The adapter extracts only `type == "text"` parts and joins them into a single string output.

### Custom provider config is required for Ollama

opencode v1.4.11 validates all model names against an internal registry (sourced from models.dev). Local Ollama models are not in that registry, so they trigger a `ProviderModelNotFoundError` when specified naively.

The workaround is to define a **custom provider** with an inline `models` block in `config.json`. The `models` block registers the model metadata at runtime, bypassing registry validation:

```json
{
  "model": "ollama/devstral-small-2",
  "provider": {
    "ollama": {
      "api": "openai",
      "npm": "@ai-sdk/openai",
      "options": {
        "baseURL": "http://10.0.0.7:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "devstral-small-2": {
          "id": "devstral-small-2:24b",
          "tool_call": true,
          "temperature": true,
          "cost": {"input": 0, "output": 0},
          "limit": {"context": 32000, "output": 16000}
        }
      }
    }
  }
}
```

- `api: "openai"` — use the OpenAI-compatible SDK path inside opencode
- `npm: "@ai-sdk/openai"` — the Vercel AI SDK package bundled in the opencode binary
- `options.baseURL` — Ollama's OpenAI-compatible endpoint (`/v1`, not `/v1/messages`)
- `models.<key>.id` — the actual model name sent to Ollama in the API request body
- The map key (`devstral-small-2`) becomes the model ID used in `model: "ollama/devstral-small-2"`

### Why not the Anthropic endpoint?

Ollama also supports an Anthropic-compatible endpoint at `/v1/messages`. However, opencode's `@ai-sdk/anthropic` package appends `/messages` to `ANTHROPIC_BASE_URL`, resulting in `<base>/messages` — not `/v1/messages`. Setting `ANTHROPIC_BASE_URL=http://10.0.0.7:11434/v1` produces the correct URL, but the anthropic provider still validates the model name against Anthropic's registry (`claude-*` names only). The OpenAI-compatible path with a custom provider definition is cleaner.

---

## File Layout

```
dark-factory/adapters/opencode/
├── Dockerfile          # Downloads opencode binary + installs Python deps
├── start.sh            # Starts opencode serve, waits for ready, starts uvicorn
├── config.json         # opencode config: custom Ollama provider + model registry
├── requirements.txt    # fastapi, uvicorn, httpx, pyyaml
├── src/
│   └── main.py         # FastAPI adapter (health, agents, runs endpoints)
├── manifests/
│   ├── deployment.yaml # Single-container k8s Deployment
│   └── service.yaml    # ClusterIP Service on port 8001
├── install.sh
├── uninstall.sh
├── test.sh
└── diag.sh
```

---

## Container Internals

### Startup sequence (`start.sh`)

```
opencode serve --port 4096 --hostname 0.0.0.0  (background)
    │
    ├── reads /root/.config/opencode/config.json
    ├── loads Ollama provider + model registry
    └── listens on :4096

poll GET /session until 200  (up to 30 s)

uvicorn src.main:app --host 0.0.0.0 --port 8001  (foreground, PID 1)
```

opencode runs as a background process. If it crashes, the pod stays alive (uvicorn keeps running) but all `/runs` calls will fail with a connection error to `localhost:4096`. The liveness probe on the FastAPI `/health` endpoint does not detect this. A future improvement would be to add a liveness check that pings the opencode server.

### Dockerfile

The image is built on `python:3.12-slim`. The opencode binary is downloaded from GitHub releases at build time:

```
https://github.com/anomalyco/opencode/releases/download/v1.4.11/opencode-linux-x64.tar.gz
```

This avoids the `ghcr.io/opencode-ai/opencode` image (which is private and returns 403).

The opencode config is baked into the image at `/root/.config/opencode/config.json`. To change the model or provider without rebuilding, mount a ConfigMap over that path.

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `OPENCODE_SERVER_URL` | `http://localhost:4096` | Where the FastAPI adapter reaches opencode |
| `OPENCODE_SERVE_PORT` | `4096` | Port opencode serve listens on |
| `OPENCODE_DATA_DIR` | `/data/opencode` | opencode's SQLite DB and session storage |
| `PORT` | `8001` | Port the FastAPI adapter listens on |

---

## API Endpoints

### `GET /health`
Returns `{"status": "ok"}`. Used by k8s startup, liveness, and readiness probes.

### `GET /agents`
Returns the worker metadata for this adapter instance:
```json
[{
  "worker_id": "oc-devstral",
  "agent_cli": "opencode",
  "model": "ollama/devstral-small-2:24b",
  "capabilities": ["code-gen", "refactor"]
}]
```

### `POST /runs`
Accepts a broker run request and streams SSE events.

**Request:**
```json
{
  "input": "task description",
  "worker_id": "oc-devstral",
  "metadata": {}
}
```

**Response** (SSE stream):
```
data: {"type": "run_status", "status": "completed"}

data: {"type": "run_output", "output": {"parts": [{"content": "...", "content_type": "text/plain"}]}}
```

On error:
```
data: {"type": "run_status", "status": "failed", "error": "..."}
```

---

## opencode Session Lifecycle

Each `/runs` call creates a **new opencode session**. Sessions are not reused across runs. This means:

- No conversation history bleeds between tasks ✓
- No session management overhead in the adapter ✓
- opencode's session storage grows over time in `/data/opencode` — the emptyDir volume is wiped on pod restart, so this is bounded

Sessions are named `run_<unix_timestamp>` for traceability in opencode's internal DB.

---

## Adding More Ollama Models

To add a model (e.g., `qwen3-coder-next:latest`), add an entry to the `models` block in `config.json`:

```json
"qwen3-coder-next": {
  "id": "qwen3-coder-next:latest",
  "name": "Qwen3 Coder Next",
  "tool_call": true,
  "temperature": true,
  "cost": {"input": 0, "output": 0},
  "limit": {"context": 32000, "output": 16000}
}
```

Then set `"model": "ollama/qwen3-coder-next"` (or keep devstral as default and switch per deployment via a mounted ConfigMap).

Each worker in the [[Worker Registry]] that uses a different model should ideally get its own deployment with a model-specific config. The current deployment serves `oc-devstral` only.

---

## Deploying a Second opencode Worker

To deploy `oc-qwen-coder-next` alongside `oc-devstral`:

1. Copy the deployment manifest, change `name` and `app` labels to `adapter-opencode-qwen`
2. Mount a ConfigMap with `"model": "ollama/qwen3-coder-next"` over `/root/.config/opencode/config.json`
3. Add a separate Service for the new deployment
4. Register `adapter_url: http://adapter-opencode-qwen.dark-factory.svc.cluster.local:8001` in the broker ConfigMap

---

## Known Limitations

| Limitation | Details |
|---|---|
| opencode crash not detected | Liveness probe only checks FastAPI `/health`, not the internal opencode server. Runs will silently fail if opencode crashes. |
| Model baked into image | Changing the model requires rebuilding or mounting a ConfigMap. |
| No per-run model injection | Unlike the claude-code adapter, opencode's model is fixed at server startup. The broker cannot inject a different model per run. |
| Session not linked to git workspace | opencode's `--cwd` is set to the container's working directory, not a per-run git worktree. File edits by the agent go to the container filesystem, not a real branch. |
| No tool approval callbacks | opencode tool use (Bash, Edit, etc.) runs unattended. The ACP `await/interrupt` mechanism is not wired up. |

---

## Troubleshooting

**Pod not starting / stuck in startup probe:**
```bash
kubectl logs -n dark-factory -l app=adapter-opencode
```
Look for opencode config validation errors — usually an `Unrecognized key` in config.json.

**Runs returning `failed` immediately:**
```bash
kubectl exec -n dark-factory deploy/adapter-opencode -- curl -s http://localhost:4096/session
```
If this returns HTML or connection refused, opencode serve is down. Check the process:
```bash
kubectl exec -n dark-factory deploy/adapter-opencode -- ps aux | grep opencode
```

**Model not found error:**
Verify the model name in `config.json` matches a key in the `models` block, and that `model` at the top level is `"ollama/<key>"` — not `"ollama/<id>"`. The key and the `id` field are different: key is what you reference in `model:`, `id` is what gets sent to Ollama.

**Rebuilding and redeploying:**
```bash
cd dark-factory/adapters/opencode
docker build -t dark-factory/adapter-opencode:latest .
docker save dark-factory/adapter-opencode:latest | k3s ctr images import -
kubectl rollout restart deployment/adapter-opencode -n dark-factory
```
