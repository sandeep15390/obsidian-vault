# LiteLLM Implementation Plan

> Execution guide for an OpenHands coding agent (or human implementer). Produce only the files listed. Do not modify existing infrastructure files unless instructed. Follow the Redis and PostgreSQL Metrics reference implementations exactly.

## Reference Files (read before implementing anything)

| File | Why |
|---|---|
| `~/src/home_infra/redis/install.sh` | Top-level orchestrator: banner helpers, dry-run, flag parsing, `helm upgrade --install` pattern |
| `~/src/home_infra/redis/uninstall.sh` | Uninstall: `--delete-data`, `--delete-namespace`, `--force` flags; confirm helper; residue check |
| `~/src/home_infra/redis/metrics/install.sh` | Alloy ConfigMap patch pattern; Grafana dashboard download + patch + upload via `kubectl exec` |
| `~/src/home_infra/redis/metrics/test.sh` | Test structure: `pass`/`fail`/`header` helpers; port-forward for distroless; large output to temp file; Alloy API check; Prometheus query check |
| `~/src/home_infra/redis/metrics/uninstall.sh` | Alloy ConfigMap strip (Python regex); residue check |
| `~/src/home_infra/redis/metrics/diag.sh` | Diag structure: numbered sections; port-forward for metrics sample |

---

## Design Doc

`~/src/obsidian-vault/Projects/Homelab/LiteLLM.md` — written, read it for all authoritative decisions.

---

## Phase 1 — Directory Scaffold

Create the following empty files (structure only — content in later phases):

```
~/src/home_infra/litellm/
├── install.sh
├── uninstall.sh
├── test.sh
├── diag.sh
└── manifests/
    ├── litellm-values.yaml
    └── proxy-config.yaml
```

Shell commands:
```bash
mkdir -p ~/src/home_infra/litellm/manifests
touch ~/src/home_infra/litellm/install.sh
touch ~/src/home_infra/litellm/uninstall.sh
touch ~/src/home_infra/litellm/test.sh
touch ~/src/home_infra/litellm/diag.sh
chmod +x ~/src/home_infra/litellm/install.sh
chmod +x ~/src/home_infra/litellm/uninstall.sh
chmod +x ~/src/home_infra/litellm/test.sh
chmod +x ~/src/home_infra/litellm/diag.sh
```

---

## Phase 2 — Manifests

### `manifests/litellm-values.yaml`

This is the Helm values file for `oci://docker.litellm.ai/berriai/litellm-helm` version `1.82.3`.

**Key decisions:**

| Field | Value | Rationale |
|---|---|---|
| `image.repository` | `ghcr.io/berriai/litellm` | Base image without database; simpler, stateless mode |
| `image.tag` | `main-v1.82.3` | Pinned to stable release; never use `main-latest` |
| `image.pullPolicy` | `IfNotPresent` | Avoid re-pulls on pod restart; tag is pinned |
| `environmentSecrets` | `["litellm-api-keys"]` | Injects all keys in the secret as env vars via `envFrom: secretRef` |
| `db.deployStandalone` | `false` | Do not deploy Bitnami PostgreSQL sub-chart |
| `db.useExisting` | `false` | No external database |
| `service.type` | `NodePort` | Required for Tailscale serve |
| `service.port` | `4000` | LiteLLM default |
| `service.nodePort` | `32303` | Next available NodePort; no conflicts |
| `resources.requests.cpu` | `100m` | |
| `resources.requests.memory` | `256Mi` | |
| `resources.limits.cpu` | `1000m` | |
| `resources.limits.memory` | `512Mi` | |
| `livenessProbe.initialDelaySeconds` | `30` | Give LiteLLM time to initialize; increase from chart default of 0 |
| `readinessProbe.initialDelaySeconds` | `20` | |
| `startupProbe.failureThreshold` | `30` | 30 × 10s = 5 min startup window |
| `proxy_config.model_list` | (see below) | Inline YAML; Helm renders into ConfigMap named `litellm` |
| `proxy_config.general_settings.master_key` | `os.environ/PROXY_MASTER_KEY` | Never hardcode |
| `proxy_config.litellm_settings.callbacks` | `["prometheus"]` | Enables `/metrics` endpoint |

**`proxy_config` block (embedded in values.yaml):**

```yaml
proxy_config:
  model_list:
    - model_name: ollama/llama3.2
      litellm_params:
        model: ollama/llama3.2
        api_base: http://10.0.0.7:11434
    - model_name: ollama/gemma3
      litellm_params:
        model: ollama/gemma3
        api_base: http://10.0.0.7:11434
    - model_name: gpt-4o
      litellm_params:
        model: openai/gpt-4o
        api_key: os.environ/OPENAI_API_KEY
    - model_name: gpt-4o-mini
      litellm_params:
        model: openai/gpt-4o-mini
        api_key: os.environ/OPENAI_API_KEY
    - model_name: claude-3-5-sonnet
      litellm_params:
        model: anthropic/claude-3-5-sonnet-20241022
        api_key: os.environ/ANTHROPIC_API_KEY
    - model_name: claude-3-haiku
      litellm_params:
        model: anthropic/claude-3-haiku-20240307
        api_key: os.environ/ANTHROPIC_API_KEY
  general_settings:
    master_key: os.environ/PROXY_MASTER_KEY
  litellm_settings:
    callbacks:
      - prometheus
```

**Probe overrides (override chart defaults):**

```yaml
livenessProbe:
  path: /health/liveliness
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  path: /health/readiness
  initialDelaySeconds: 20
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

startupProbe:
  path: /health/readiness
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 30
```

### `manifests/proxy-config.yaml`

Standalone copy of the `proxy_config` block in standard YAML format (not embedded in Helm values). This file is for human readability and diff tracking only. It is NOT applied to the cluster directly — `litellm-values.yaml` is the authoritative source. Content is identical to the `proxy_config:` block above but at the top level (no `proxy_config:` wrapper key).

---

## Phase 3 — `install.sh`

**File:** `~/src/home_infra/litellm/install.sh`

**Shebang and header:**
```bash
#!/usr/bin/env bash
# install.sh — Deploy LiteLLM proxy + observability (Alloy scrape + Grafana dashboard)
# Idempotent: safe to run multiple times.
#
# Prerequisites (env vars must be set before running):
#   export PROXY_MASTER_KEY="sk-homelab-..."
#   export OPENAI_API_KEY="sk-..."
#   export ANTHROPIC_API_KEY="sk-ant-..."
#
# Usage:
#   ./install.sh            # full install
#   ./install.sh --dry-run  # print what would be done without doing it
set -euo pipefail
```

**Constants (at top of script):**

```
SCRIPT_DIR       — $(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
CHART_OCI        — oci://docker.litellm.ai/berriai/litellm-helm
CHART_VERSION    — 1.82.3
RELEASE_NAME     — litellm
NAMESPACE        — litellm
SECRET_NAME      — litellm-api-keys
VALUES_FILE      — ${SCRIPT_DIR}/manifests/litellm-values.yaml
ALLOY_NAMESPACE  — metrics
ALLOY_CONFIGMAP  — alloy-metrics-config
ALLOY_KEY        — config.alloy
GRAFANA_NAMESPACE — logging
GRAFANA_DEPLOY   — grafana
GRAFANA_SECRET   — grafana
DASHBOARD_UID    — litellm-overview
DASHBOARD_SOURCE — https://raw.githubusercontent.com/BerriAI/litellm/main/cookbook/litellm_proxy_server/grafana_dashboard/dashboard_v2/grafana_dashboard.json
PROM_DS_UID      — afilsg21fj18ga
```

**Flag parsing:** same pattern as redis/install.sh — `--dry-run` and `--help` only (no `--no-tests` since this is a flat project, not a sub-project).

**Helper functions:** copy `info`, `ok`, `die`, `banner`, `dry_run` verbatim from redis/install.sh.

**Steps (in order):**

### Step 1 — Prerequisites check (banner: "Checking prerequisites")

```
1. command -v helm    || die
2. command -v kubectl || die
3. command -v curl    || die
4. command -v python3 || die
5. kubectl cluster-info || die
6. [ -f "${VALUES_FILE}" ] || die
7. Check env vars are set:
   - [ -n "${PROXY_MASTER_KEY:-}" ] || die "PROXY_MASTER_KEY env var not set"
   - [ -n "${OPENAI_API_KEY:-}" ]   || die "OPENAI_API_KEY env var not set"
   - [ -n "${ANTHROPIC_API_KEY:-}" ]|| die "ANTHROPIC_API_KEY env var not set"
8. Check Alloy DaemonSet exists: kubectl get daemonset alloy-metrics -n metrics || die
9. Check Grafana deployment exists: kubectl get deployment grafana -n logging || die
```

**Dry-run early exit:** if `DRY_RUN=true`, print numbered plan (same style as redis) and `exit 0`.

### Step 2 — Create `litellm-api-keys` Secret (banner: "1/4 — API Keys Secret")

```bash
# Idempotent: check if secret exists; if it does, skip creation
if kubectl get secret "${SECRET_NAME}" -n "${NAMESPACE}" >/dev/null 2>&1; then
  ok "Secret '${SECRET_NAME}' already exists — skipping creation"
else
  kubectl create namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
  kubectl create secret generic "${SECRET_NAME}" \
    --namespace "${NAMESPACE}" \
    --from-literal=PROXY_MASTER_KEY="${PROXY_MASTER_KEY}" \
    --from-literal=OPENAI_API_KEY="${OPENAI_API_KEY}" \
    --from-literal=ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY}"
  ok "Secret '${SECRET_NAME}' created"
fi
```

> **Important:** Create the namespace and secret BEFORE helm upgrade --install. The Helm chart references the secret in `environmentSecrets`; if the secret does not exist, the pod will fail to start with `secret not found`.

> **Why not `kubectl apply` for secrets?** `kubectl create secret generic` with `--dry-run=client -o yaml | kubectl apply -f -` is the idempotent approach. However, a simpler guard (`if secret exists, skip`) is used here to avoid re-encoding values on re-runs — the existing secret is kept as-is, just like Bitnami Redis preserves its auth secret.

### Step 3 — Helm deploy (banner: "2/4 — LiteLLM Helm Deploy")

```bash
dry_run helm upgrade --install "${RELEASE_NAME}" "${CHART_OCI}" \
  --version "${CHART_VERSION}" \
  --namespace "${NAMESPACE}" \
  --create-namespace \
  --atomic \
  --timeout 5m \
  -f "${VALUES_FILE}"
ok "Helm release '${RELEASE_NAME}' deployed"
```

Wait for pod Ready:
```bash
dry_run kubectl rollout status deployment/litellm -n "${NAMESPACE}" --timeout=120s
ok "LiteLLM deployment is Ready"
```

Print access info block (same style as redis):
```
  In-cluster API: http://litellm.litellm.svc.cluster.local:4000
  NodePort: http://<node-ip>:32303
  Tailnet: https://litellm.tailc98a25.ts.net (after tailscale serve setup)
  Master key: kubectl get secret litellm-api-keys -n litellm -o jsonpath='{.data.PROXY_MASTER_KEY}' | base64 -d; echo
```

### Step 4 — Alloy ConfigMap patch (banner: "3/4 — Alloy Scrape Config")

Follow the exact pattern from redis/metrics/install.sh:

```bash
LITELLM_BLOCK='
// # BEGIN litellm
prometheus.scrape "litellm" {
  targets = [{
    "__address__" = "litellm.litellm.svc.cluster.local:4000",
  }]
  job_name        = "litellm"
  scrape_interval = "15s"
  metrics_path    = "/metrics"
  forward_to      = [prometheus.remote_write.default.receiver]
}
// # END litellm'
```

Idempotency guard: `if echo "${CURRENT_CONFIG}" | grep -q '# BEGIN litellm'; then skip; else patch + restart Alloy`.

Alloy restart: `kubectl rollout restart daemonset/alloy-metrics -n metrics` + `kubectl rollout status ... --timeout=120s`.

### Step 5 — Grafana dashboard (banner: "4/4 — Grafana Dashboard")

Follow the exact pattern from redis/metrics/install.sh.

**Dashboard source URL:**
```
https://raw.githubusercontent.com/BerriAI/litellm/main/cookbook/litellm_proxy_server/grafana_dashboard/dashboard_v2/grafana_dashboard.json
```

**Python patching script** (after `curl` download):
```python
import sys, json

raw = sys.stdin.read()
d = json.loads(raw)

# Walk recursively and fix datasource UIDs to Prometheus
def fix_datasource(obj):
    if isinstance(obj, dict):
        if 'datasource' in obj and isinstance(obj['datasource'], dict):
            ds = obj['datasource']
            if ds.get('type') == 'prometheus':
                ds['uid'] = 'afilsg21fj18ga'
        for v in obj.values():
            fix_datasource(v)
    elif isinstance(obj, list):
        for item in obj:
            fix_datasource(item)

fix_datasource(d)

# Override uid and title
d['uid'] = 'litellm-overview'
d['title'] = 'LiteLLM Overview'
d.pop('id', None)
d.pop('__inputs', None)
d.pop('__elements', None)
d.pop('__requires', None)

print(json.dumps(d))
```

**Upload pattern:** identical to redis/metrics/install.sh — base64-encode payload, `kubectl exec deploy/grafana -- sh -c "echo '...' | base64 -d | curl -sf -X POST ..."`.

### Step 6 — Run tests

```bash
bash "${SCRIPT_DIR}/test.sh"
```

---

## Phase 4 — `uninstall.sh`

**File:** `~/src/home_infra/litellm/uninstall.sh`

**Flags:**
- `--delete-namespace` — also delete the `litellm` namespace (and the secret inside it)
- `--force` — skip confirmation prompt

**No `--delete-data` flag** (no PVC). Validate at startup: `--delete-namespace` does not require `--delete-data`. This is simpler than Redis.

**Steps (in order):**

1. Parse flags; validate
2. Confirm (unless `--force`)
3. **Delete Grafana dashboard** (banner: "1/4 — Grafana Dashboard"):
   - Retrieve Grafana password from secret
   - `kubectl exec deploy/grafana -- curl -sf -X DELETE -u "admin:${PASS}" http://localhost:3000/api/dashboards/uid/litellm-overview`
   - 404 is not an error
4. **Strip Alloy ConfigMap block** (banner: "2/4 — Alloy ConfigMap"):
   - Python regex strip: `re.sub(r'\n// # BEGIN litellm.*?// # END litellm', '', config, flags=re.DOTALL)`
   - Patch ConfigMap; restart Alloy DaemonSet; wait for rollout
   - No-op if block not present
5. **Helm uninstall** (banner: "3/4 — Helm Release"):
   - `if helm list -n litellm | grep litellm; then helm uninstall litellm -n litellm --wait --timeout 2m`
   - Wait for pods to terminate (loop, same pattern as redis)
6. **Namespace deletion** (banner: "4/4 — Namespace") — only if `--delete-namespace`:
   - `kubectl delete namespace litellm --timeout=60s --ignore-not-found`
7. **Residue check** (banner: "Residue check"):
   - If `--delete-namespace`: `kubectl get namespace litellm` should fail
   - `kubectl get deployment litellm -n litellm` should fail
   - Alloy ConfigMap should not contain `# BEGIN litellm`
   - Grafana should return 404 for dashboard uid `litellm-overview`

---

## Phase 5 — `test.sh`

**File:** `~/src/home_infra/litellm/test.sh`

**Header constants:**
```
NAMESPACE        — litellm
ALLOY_NAMESPACE  — metrics
ALLOY_CONFIGMAP  — alloy-metrics-config
GRAFANA_NAMESPACE — logging
GRAFANA_DEPLOY   — grafana
GRAFANA_SECRET   — grafana
DASHBOARD_UID    — litellm-overview
PROM_NAMESPACE   — metrics
PROM_DEPLOY      — prometheus-server
LITELLM_LOCAL_PORT — 14000   (port-forward target)
```

**Pattern:** `pass`/`fail`/`header` helpers, `PASS_COUNT`/`FAIL_COUNT`, port-forward with trap, temp file for large outputs.

**Port-forward setup** (for all API/metrics tests):
```bash
start_port_forward() {
  kubectl port-forward -n "${NAMESPACE}" svc/litellm "${LITELLM_LOCAL_PORT}":4000 &>/tmp/litellm-pf.log &
  PF_PID=$!
  sleep 3
}
stop_port_forward() {
  [[ -n "${PF_PID}" ]] && kill "${PF_PID}" 2>/dev/null || true
}
trap stop_port_forward EXIT
```

**Retrieve master key once:**
```bash
MASTER_KEY=$(kubectl get secret litellm-api-keys -n "${NAMESPACE}" \
  -o jsonpath="{.data.PROXY_MASTER_KEY}" 2>/dev/null | base64 -d || true)
```

**Sections (implement in this order):**

### 1. Prerequisites (3 tests)
```
command -v helm
command -v kubectl
command -v python3
```

### 2. K8s Resources (5 tests)
```
kubectl get namespace litellm
kubectl get deployment litellm -n litellm
kubectl get service litellm -n litellm && verify nodePort=32303
kubectl get secret litellm-api-keys -n litellm
kubectl get configmap litellm -n litellm
```

For NodePort verification:
```bash
NP=$(kubectl get service litellm -n litellm -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || echo "")
[[ "${NP}" == "32303" ]] && pass "Service NodePort = 32303" || fail "Service NodePort = '${NP}' (expected 32303)"
```

### 3. Helm Version (1 test)
```bash
CHART_VER=$(helm list -n litellm -o json 2>/dev/null | python3 -c "
import sys,json
releases = json.load(sys.stdin)
r = [x for x in releases if x['name'] == 'litellm']
print(r[0]['chart'].split('-')[-1] if r else '')
" 2>/dev/null || echo "")
[[ "${CHART_VER}" == "1.82.3" ]] && pass "..." || fail "..."
```

### 4. Pod Health (3 tests)
```
pod phase = Running
pod Ready = True
pod restartCount <= 2
```
Use `kubectl get pod -n litellm -l app.kubernetes.io/name=litellm` to find the pod.

### 5. Secret Wiring (3 tests)

Use jsonpath to verify `envFrom[*].secretRef.name`:
```bash
SECRET_REF=$(kubectl get deployment litellm -n litellm \
  -o jsonpath='{.spec.template.spec.containers[0].envFrom[?(@.secretRef.name=="litellm-api-keys")].secretRef.name}' \
  2>/dev/null || echo "")
[[ "${SECRET_REF}" == "litellm-api-keys" ]] \
  && pass "envFrom secretRef 'litellm-api-keys' present" \
  || fail "envFrom secretRef 'litellm-api-keys' not found"
```

For the three individual keys, verify they exist in the secret data:
```bash
for KEY in PROXY_MASTER_KEY OPENAI_API_KEY ANTHROPIC_API_KEY; do
  VAL=$(kubectl get secret litellm-api-keys -n litellm \
    -o jsonpath="{.data.${KEY}}" 2>/dev/null || echo "")
  [[ -n "${VAL}" ]] \
    && pass "Secret key ${KEY} present" \
    || fail "Secret key ${KEY} missing"
done
```

### 6. Proxy API (7 tests)

All via port-forward. Start port-forward before this section.

```bash
start_port_forward

# /health/readiness
HTTP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" http://localhost:${LITELLM_LOCAL_PORT}/health/readiness 2>/dev/null || echo "000")
[[ "${HTTP_CODE}" == "200" ]] && pass "GET /health/readiness = 200" || fail "GET /health/readiness = ${HTTP_CODE}"

# /health/liveliness
HTTP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" http://localhost:${LITELLM_LOCAL_PORT}/health/liveliness 2>/dev/null || echo "000")
[[ "${HTTP_CODE}" == "200" ]] && pass "GET /health/liveliness = 200" || fail "..."

# /v1/models (authenticated)
MODELS_RESPONSE=$(curl -sf -H "Authorization: Bearer ${MASTER_KEY}" http://localhost:${LITELLM_LOCAL_PORT}/v1/models 2>/dev/null || echo "")
[[ -n "${MODELS_RESPONSE}" ]] && pass "GET /v1/models returns data" || fail "..."

# At least one ollama/ model
OLLAMA_MODEL=$(echo "${MODELS_RESPONSE}" | python3 -c "
import sys,json
d = json.load(sys.stdin)
models = [m['id'] for m in d.get('data',[]) if m.get('id','').startswith('ollama/')]
print(models[0] if models else '')
" 2>/dev/null || echo "")
[[ -n "${OLLAMA_MODEL}" ]] && pass "Model list contains ollama/ model (${OLLAMA_MODEL})" || fail "No ollama/ model found"

# OpenAI model
OPENAI_MODEL=$(echo "${MODELS_RESPONSE}" | python3 -c "
import sys,json
d = json.load(sys.stdin)
m = [x for x in d.get('data',[]) if 'gpt' in x.get('id','')]
print(m[0]['id'] if m else '')
" 2>/dev/null || echo "")
[[ -n "${OPENAI_MODEL}" ]] && pass "Model list contains OpenAI model (${OPENAI_MODEL})" || fail "No OpenAI model found"

# Anthropic model
ANTHROPIC_MODEL=$(echo "${MODELS_RESPONSE}" | python3 -c "
import sys,json
d = json.load(sys.stdin)
m = [x for x in d.get('data',[]) if 'claude' in x.get('id','')]
print(m[0]['id'] if m else '')
" 2>/dev/null || echo "")
[[ -n "${ANTHROPIC_MODEL}" ]] && pass "Model list contains Anthropic model (${ANTHROPIC_MODEL})" || fail "No Anthropic model found"

# Unauthenticated /v1/models → 401
UNAUTH_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${LITELLM_LOCAL_PORT}/v1/models 2>/dev/null || echo "000")
[[ "${UNAUTH_CODE}" == "401" ]] && pass "Unauthenticated /v1/models → 401" || fail "Unauthenticated /v1/models → ${UNAUTH_CODE} (expected 401)"
```

### 7. Ollama Routing (2 tests)

Send a minimal chat completion request to an Ollama model. Wrap in an Ollama preflight check:

```bash
OLLAMA_REACHABLE=true
OLLAMA_CHECK=$(curl -sf --max-time 3 http://10.0.0.7:11434/api/tags 2>/dev/null && echo "ok" || echo "fail")
if [[ "${OLLAMA_CHECK}" != "ok" ]]; then
  echo "  [SKIP]  Ollama not reachable at 10.0.0.7:11434 — skipping routing tests"
  OLLAMA_REACHABLE=false
fi

if [[ "${OLLAMA_REACHABLE}" == "true" ]]; then
  CHAT_RESPONSE=$(curl -sf \
    -H "Authorization: Bearer ${MASTER_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"model":"ollama/llama3.2","messages":[{"role":"user","content":"Say the word PONG only"}],"max_tokens":5}' \
    --max-time 60 \
    http://localhost:${LITELLM_LOCAL_PORT}/v1/chat/completions 2>/dev/null || echo "")

  [[ -n "${CHAT_RESPONSE}" ]] && pass "Ollama chat completion request returned a response" || fail "Ollama chat completion returned empty"

  HAS_CHOICES=$(echo "${CHAT_RESPONSE}" | python3 -c "
import sys,json
d = json.load(sys.stdin)
print('yes' if d.get('choices') else 'no')
" 2>/dev/null || echo "no")
  [[ "${HAS_CHOICES}" == "yes" ]] && pass "Ollama response contains 'choices' array" || fail "Ollama response missing 'choices' (got: ${CHAT_RESPONSE:0:200})"
fi
```

> **Note on timing:** The Ollama routing test uses `--max-time 60` because the first LLM request after a cold start can take several seconds. This is acceptable in a test suite.

### 8. Metrics Endpoint (4 tests)

Store output in temp file to avoid bash variable truncation:

```bash
METRICS_FILE=$(mktemp /tmp/litellm-metrics-XXXXXX.txt)
trap 'rm -f "${METRICS_FILE}"; stop_port_forward' EXIT

curl -sf "http://localhost:${LITELLM_LOCAL_PORT}/metrics" -o "${METRICS_FILE}" 2>/dev/null || true

[[ -s "${METRICS_FILE}" ]] && pass "/metrics returned data" || fail "/metrics returned empty"

grep -q 'litellm_proxy_total_requests_metric' "${METRICS_FILE}" \
  && pass "litellm_proxy_total_requests_metric present" \
  || fail "litellm_proxy_total_requests_metric missing"

grep -q 'litellm_proxy_failed_requests_metric' "${METRICS_FILE}" \
  && pass "litellm_proxy_failed_requests_metric present" \
  || fail "litellm_proxy_failed_requests_metric missing"

grep -q 'litellm_request_total_latency_metric' "${METRICS_FILE}" \
  && pass "litellm_request_total_latency_metric present" \
  || fail "litellm_request_total_latency_metric missing"
```

> **Why these metrics appear before any requests:** LiteLLM registers metric descriptors at startup (the `_created` variants and histogram bucket series). The metric names should be present in `/metrics` output even with zero requests. If they are absent, `callbacks: ["prometheus"]` was not applied.

### 9. Alloy ConfigMap (2 tests)

```bash
stop_port_forward; PF_PID=""

ALLOY_CONFIG=$(kubectl get configmap "${ALLOY_CONFIGMAP}" -n "${ALLOY_NAMESPACE}" \
  -o jsonpath='{.data.config\.alloy}' 2>/dev/null || echo "")

echo "${ALLOY_CONFIG}" | grep -q '# BEGIN litellm' \
  && pass "Alloy ConfigMap contains litellm scrape block" \
  || fail "Alloy ConfigMap missing litellm scrape block"

echo "${ALLOY_CONFIG}" | grep -q 'job_name.*=.*"litellm"' \
  && pass "Alloy litellm block sets job_name = \"litellm\"" \
  || fail "Alloy litellm block missing job_name"
```

### 10. Prometheus Ingestion (3 tests)

Follow exact pattern from redis/metrics/test.sh (Alloy component API check + Prometheus PromQL query).

**Alloy component check:**
```bash
ALLOY_POD=$(kubectl get pod -n "${ALLOY_NAMESPACE}" -l app.kubernetes.io/name=alloy \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "")
ALLOY_LOCAL_PORT=19124   # choose a port not used by other tests (redis uses 19123)

if [[ -n "${ALLOY_POD}" ]]; then
  kubectl port-forward -n "${ALLOY_NAMESPACE}" "${ALLOY_POD}" "${ALLOY_LOCAL_PORT}":12345 &>/tmp/alloy-litellm-pf.log &
  ALLOY_PF_PID=$!
  sleep 2
fi

COMPONENTS_JSON=$(curl -sf "http://localhost:${ALLOY_LOCAL_PORT}/api/v0/web/components" 2>/dev/null || echo "")
[[ -n "${ALLOY_PF_PID:-}" ]] && kill "${ALLOY_PF_PID}" 2>/dev/null || true

LITELLM_COMPONENT=$(echo "${COMPONENTS_JSON}" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    comps = [c for c in d if c.get('localID') == 'prometheus.scrape.litellm']
    print(comps[0].get('localID','') if comps else '')
except Exception:
    print('')
" 2>/dev/null || echo "")

[[ "${LITELLM_COMPONENT}" == "prometheus.scrape.litellm" ]] \
  && pass "Alloy component prometheus.scrape.litellm is registered" \
  || fail "Alloy component prometheus.scrape.litellm not found"
```

**Prometheus PromQL check:**
```bash
PROM_POD=$(kubectl get pod -n "${PROM_NAMESPACE}" -l "app.kubernetes.io/component=server" \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "")

# Retry loop (metric may not exist yet if zero requests sent)
METRIC_PRESENT=""
for i in $(seq 1 6); do
  METRIC_PRESENT=$(kubectl exec -n "${PROM_NAMESPACE}" "${PROM_POD}" -c prometheus-server -- \
    wget -qO- "http://localhost:9090/api/v1/query?query=litellm_proxy_total_requests_metric_total" 2>/dev/null \
    | python3 -c "
import sys, json
d = json.load(sys.stdin)
results = d.get('data',{}).get('result',[])
print('present' if results else '')
" 2>/dev/null || echo "")
  [[ "${METRIC_PRESENT}" == "present" ]] && break
  [[ "${i}" -lt 6 ]] && echo "  [WAIT]  litellm metric not yet in Prometheus (attempt ${i}/6)..." && sleep 15
done

[[ "${METRIC_PRESENT}" == "present" ]] \
  && pass "litellm_proxy_total_requests_metric_total present in Prometheus" \
  || fail "litellm metric not found in Prometheus after retries"
```

> **Note on metric naming:** LiteLLM registers `litellm_proxy_total_requests_metric` as a Counter. Prometheus appends `_total` suffix to counter metric names. The PromQL query should use `litellm_proxy_total_requests_metric_total`. Verify the exact name from `/metrics` output during test development.

**Also check that at least one label series exists:**
```bash
LABEL_CHECK=$(kubectl exec -n "${PROM_NAMESPACE}" "${PROM_POD}" -c prometheus-server -- \
  wget -qO- "http://localhost:9090/api/v1/labels" 2>/dev/null \
  | python3 -c "
import sys,json
d = json.load(sys.stdin)
labels = d.get('data',[])
print('yes' if 'job' in labels else 'no')
" 2>/dev/null || echo "no")
[[ "${LABEL_CHECK}" == "yes" ]] && pass "Prometheus labels include 'job'" || fail "Prometheus 'job' label missing"
```

### 11. Grafana Dashboard (2 tests)

Follow exact pattern from redis/metrics/test.sh:

```bash
GRAFANA_PASS=$(kubectl get secret "${GRAFANA_SECRET}" -n "${GRAFANA_NAMESPACE}" \
  -o jsonpath="{.data.admin-password}" 2>/dev/null | base64 -d || true)

DASH_RESULT=$(kubectl exec -n "${GRAFANA_NAMESPACE}" deploy/"${GRAFANA_DEPLOY}" -- \
  wget -qO- --header "Authorization: Basic $(echo -n "admin:${GRAFANA_PASS}" | base64)" \
  "http://localhost:3000/api/dashboards/uid/${DASHBOARD_UID}" 2>/dev/null || echo "")

DASH_TITLE=$(echo "${DASH_RESULT}" | python3 -c \
  "import sys,json; print(json.load(sys.stdin).get('dashboard',{}).get('title',''))" 2>/dev/null || echo "")
[[ -n "${DASH_TITLE}" ]] \
  && pass "Dashboard '${DASHBOARD_UID}' exists in Grafana (title: ${DASH_TITLE})" \
  || fail "Dashboard '${DASHBOARD_UID}' not found in Grafana"

DASH_UID_CHECK=$(echo "${DASH_RESULT}" | python3 -c \
  "import sys,json; print(json.load(sys.stdin).get('dashboard',{}).get('uid',''))" 2>/dev/null || echo "")
[[ "${DASH_UID_CHECK}" == "${DASHBOARD_UID}" ]] \
  && pass "Dashboard UID matches '${DASHBOARD_UID}'" \
  || fail "Dashboard UID = '${DASH_UID_CHECK}' (expected '${DASHBOARD_UID}')"
```

### 12. Tailscale NodePort (3 tests)

```bash
# Test NodePort is reachable from host
NP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 \
  http://localhost:32303/health/readiness 2>/dev/null || echo "000")
[[ "${NP_CODE}" == "200" ]] \
  && pass "NodePort 32303 /health/readiness = 200" \
  || fail "NodePort 32303 /health/readiness = ${NP_CODE}"

NP_CODE2=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 \
  http://localhost:32303/health/liveliness 2>/dev/null || echo "000")
[[ "${NP_CODE2}" == "200" ]] \
  && pass "NodePort 32303 /health/liveliness = 200" \
  || fail "NodePort 32303 /health/liveliness = ${NP_CODE2}"

# Check Tailscale serve config
TS_CONFIG=$(sudo tailscale serve status 2>/dev/null || echo "")
echo "${TS_CONFIG}" | grep -q 'litellm\|32303' \
  && pass "tailscale serve shows litellm/32303 entry" \
  || fail "tailscale serve does not show litellm entry (run: sudo tailscale serve --service=svc:litellm --https=443 127.0.0.1:32303)"
```

### Smoke test mode

When `--smoke-test` flag is set, run only:
1. Pod Running + Ready
2. `GET /health/readiness` returns 200 (via port-forward)
3. `GET /v1/models` returns at least one model
4. `GET /metrics` contains `litellm_proxy_total_requests_metric`
5. Grafana dashboard uid `litellm-overview` exists

Then print results and exit.

### Results footer

```bash
echo
echo "=============================="
printf "Results: %d passed, %d failed, 0 warnings\n" "${PASS_COUNT}" "${FAIL_COUNT}"
echo "=============================="
[[ "${FAIL_COUNT}" -eq 0 ]] && exit 0 || exit 1
```

---

## Phase 6 — `diag.sh`

**File:** `~/src/home_infra/litellm/diag.sh`

Read-only diagnostics. Follow redis/metrics/diag.sh structure. Numbered sections:

```
1.  Tool Versions       — kubectl, helm versions
2.  Helm Release        — helm list -n litellm
3.  LiteLLM Deployment  — kubectl get deployment litellm -n litellm -o wide
4.  LiteLLM Pod         — kubectl get pod -n litellm -l app.kubernetes.io/name=litellm -o wide
5.  LiteLLM Services    — kubectl get service -n litellm -o wide (show NodePort 32303)
6.  Secret Existence    — kubectl get secret litellm-api-keys -n litellm (name only, no values)
7.  ConfigMap           — kubectl get configmap litellm -n litellm -o jsonpath='{.data.proxy_config\.yaml}' (print config)
8.  Pod Events (last 10)— kubectl get events -n litellm --sort-by='.lastTimestamp' | tail -10
9.  Pod Logs (last 30)  — kubectl logs deployment/litellm -n litellm --tail=30
10. /health/readiness   — port-forward to :4000, curl /health/readiness
11. /metrics sample     — port-forward, curl /metrics | head -60
12. Key metric values   — grep for litellm_proxy_total_requests_metric, litellm_proxy_failed_requests_metric, litellm_request_total_latency_metric
13. /v1/models          — curl with master key retrieved from secret; list model IDs
14. Alloy ConfigMap block— kubectl get configmap alloy-metrics-config -n metrics; sed -n 'BEGIN litellm/,END litellm/p'
15. Alloy DaemonSet     — kubectl get daemonset alloy-metrics -n metrics
16. Prometheus query    — litellm_proxy_total_requests_metric_total via kubectl exec prometheus
17. Grafana dashboard   — GET /api/dashboards/uid/litellm-overview via kubectl exec grafana
18. Tailscale serve     — sudo tailscale serve status
```

---

## Phase 7 — Teardown / Reinstall Validation

After all scripts are implemented and working:

```bash
cd ~/src/home_infra/litellm

# Set env vars (required for install.sh to recreate the secret)
export PROXY_MASTER_KEY="sk-homelab-..."
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."

# Cycle 1
./uninstall.sh --delete-namespace --force
./install.sh   # expect 38/38

# Cycle 2
./uninstall.sh --delete-namespace --force
./install.sh   # expect 38/38

# Cycle 3
./uninstall.sh --delete-namespace --force
./install.sh   # expect 38/38
```

After each cycle, verify the residue check in `uninstall.sh` output shows no leftovers.

Record results in the Status section of `LiteLLM.md`.

---

## Key Implementation Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| Deploy method | OCI Helm chart `oci://docker.litellm.ai/berriai/litellm-helm` version `1.82.3` | Chart is reachable, natively handles proxy_config ConfigMap, secret injection, NodePort, probes. Raw manifests would replicate what the chart already does. |
| Image | `ghcr.io/berriai/litellm:main-v1.82.3` | Base image (no DB); pinned stable tag |
| Ollama access | Node IP `http://10.0.0.7:11434` | `host.k8s.internal` does not resolve in k3s. Node IP is in CoreDNS NodeHosts (`melody-beast → 10.0.0.7`). Using raw IP is reliable. Could also use `melody-beast` hostname — but raw IP is simpler and avoids DNS dependency. |
| Metrics | Built-in LiteLLM `/metrics` at port 4000 | No exporter pod needed; `callbacks: ["prometheus"]` in proxy_config activates the endpoint |
| Grafana dashboard | Official LiteLLM GitHub `dashboard_v2/grafana_dashboard.json` | Only Prometheus-based LiteLLM dashboard available; grafana.com dashboards (24055, 24064) are Azure Monitor-based |
| No ServiceMonitor | Alloy ConfigMap patch | No ServiceMonitor CRD in cluster; matches existing Redis/PostgreSQL pattern |
| Secret creation | `kubectl create secret generic` before Helm deploy | The secret must exist before the pod starts; `environmentSecrets` requires the named secret to be present |
| No PVC | Stateless mode | Acceptable for homelab; SQLite/Postgres is a noted enhancement |
| Tailscale NodePort | 32303 | Next sequential after 32302 (prometheus-tailscale); no conflicts |

---

## Open Questions

### Blocking (must answer before implementation)

**Q1 — Ollama models available on host**
> What models are currently pulled in Ollama? The `proxy_config.yaml` hardcodes `llama3.2` and `gemma3`. If these are not pulled, Ollama routing tests will fail.
> Run: `curl http://10.0.0.7:11434/api/tags | python3 -m json.tool`
> Update `manifests/litellm-values.yaml` model list and `manifests/proxy-config.yaml` to match.

**Q2 — Node IP for Ollama access**
> The design assumes the node IP is `10.0.0.7` (DHCP). Verify before deploying:
> `kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'`
> If the IP has changed, update the `api_base` in `litellm-values.yaml`.

### Advisory (have defaults, no blocking)

**Q3 — PROXY_MASTER_KEY format**
> The master key can be any string. The LiteLLM convention is `sk-<base>`. A common homelab choice is `sk-homelab-$(openssl rand -hex 16)`. No change to scripts needed; just document your preference.

**Q4 — Which Ollama models to include in proxy_config**
> Currently `ollama/llama3.2` and `ollama/gemma3` are included. Add or remove models by editing `manifests/litellm-values.yaml` and re-running `install.sh`. Default is acceptable.

**Q5 — Chart version pinning strategy**
> Chart `1.82.3` is current as of 2026-04-15. LiteLLM releases frequently. Accept the current version for now; note an upgrade path via `helm upgrade` with updated `CHART_VERSION` constant.
