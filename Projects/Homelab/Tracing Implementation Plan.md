# Distributed Tracing Implementation Plan

> Canonical design: `~/src/obsidian-vault/Projects/Homelab/Distributed Tracing.md`

---

## Phase 1: Prerequisites Check and Repo Scaffold (1-2 hours)

### Step 1.1: Verify StorageClass

```bash
kubectl get storageclass
kubectl get pvc -A  # see current PVC storage classes
```

Expected output:
```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
local-path (default) rancher.io/local-path   Delete          WaitForFirstConsumer   false               30d
```

### Step 1.2: Helm repo updates

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Record versions:
helm search repo grafana/tempo --versions | head -5
helm search repo open-telemetry/opentelemetry-collector --versions | head -5
```

### Step 1.3: Repo scaffold

The directory structure has already been created:

```
tracing/
├── install.sh
├── uninstall.sh
├── test.sh
├── diag.sh
├── manifests/
│   ├── tempo-values.yaml
│   ├── otel-collector-values.yaml
│   ├── tempo-nodeport.yaml
│   └── tempo-networkpolicy.yaml
├── instrument/
│   ├── python/
│   │   ├── opentelemetry-config.py
│   │   └── requirements-otel.txt
│   └── nodejs/
│       ├── tracing.ts
│       └── package-otel.json
└── docs/
    ├── instrumentation-guide.md
    └── traceql-cheatsheet.md
```

---

## Phase 2: install.sh and uninstall.sh

The scripts have been created. Key design decisions:

### install.sh

```bash
./install.sh [--dry-run] [--no-tests]
```

**Flow:**
1. Check prerequisites (kubectl, helm, cluster)
2. Add Helm repos
3. Create `tracing` namespace
4. Deploy Tempo Helm release (wait for ready)
5. Deploy Otel Collector Helm release (wait for ready)
6. Create LoadBalancer + NodePort services
7. Configure Grafana Tempo datasource via API
8. Patch Alloy ConfigMap for Tempo metrics scraping
9. Run test.sh

### uninstall.sh

```bash
./uninstall.sh [--delete-data] [--delete-namespace] [--force]
```

**Flow:**
1. Delete Helm releases (reverse order: otel-collector → tempo)
2. Delete PVCs (if --delete-data)
3. Delete namespace (if --delete-namespace)
4. Remove Tempo block from Alloy ConfigMap
5. Delete Grafana Tempo datasource
6. Residue check

---

## Phase 3: test.sh (27 tests)

Test categories:

| Category | Tests | What's Validated |
|---|---|---|
| Prerequisites | 2 | kubectl, helm available |
| K8s Resources | 6 | Namespace, PVC, Deployments (3), Services (2) |
| Helm Versions | 2 | tempo, otel-collector pinned versions |
| Tempo | 4 | Pod ready, /metrics, /ready, version |
| Otel Collector | 3 | Pod ready, /metrics, OTLP listener |
| Tempo API | 3 | /ready, /loki/api/v1/query, /api/traces |
| Grafana Datasource | 3 | Configured, URL correct, TraceQL works |
| Tailscale | 3 | svc:tempo registered, NodePort reachable, LoadBalancer reachable |

Smoke test mode (`--smoke-test`): Run minimal 5 tests (prerequisites + namespace + pods + basic API).

---

## Phase 4: diag.sh

Read-only diagnostics:
- Tool versions (kubectl, helm)
- Namespace, pods, services, PVCs, ConfigMaps
- Helm releases
- Recent events (last 20)
- Pod logs (last 30 lines each)
- Tempo metrics (key counters)
- Grafana datasources
- Connectivity probes

---

## Phase 5: Instrumentation guides (documentation only)

The instrumentation files have been created:

### Python (orchestrator, broker)

**File:** `instrument/python/opentelemetry-config.py`

```python
# In your service:
import sys
sys.path.insert(0, '/home/melody/src/home_infra/tracing/instrument/python')

from opentelemetry_config import setup_tracer, add_fastapi_instrumentation

app = FastAPI()
add_fastapi_instrumentation(app)
tracer = setup_tracer("orchestrator")

# Manual spans:
with tracer.start_as_current_span("process_task"):
    # ...
    pass
```

### Node.js (adapters)

**File:** `instrument/nodejs/tracing.ts`

```typescript
// In your adapter:
import { initTracer } from './tracing';

initTracer('adapter-claudecode');

// Use tracer:
const tracer = trace.getTracer('adapter-claudecode');
tracer.startActiveSpan('agent_run', (span) => {
    // ...
    span.end();
});
```

---

## Phase 6: Teardown/Reinstall Validation (3 cycles)

Run 3 destructive cycles:

```bash
cd ~/src/home_infra/tracing

# Cycle 1
./uninstall.sh --delete-data --delete-namespace --force
./install.sh
./test.sh  # All 27/27 should pass

# Cycle 2
./uninstall.sh --delete-data --delete-namespace --force
./install.sh
./test.sh

# Cycle 3
./uninstall.sh --delete-data --delete-namespace --force
./install.sh
./test.sh
```

Record results in Obsidian doc Status section.

---

## Dark Factory Integration

After tracing stack is deployed, instrument the Dark Factory services:

### Orchestrator (Python)

1. Add `opentelemetry-*` dependencies to `requirements.txt`
2. Import `opentelemetry-config.py` in `main.py`
3. Add manual spans for: `process_task`, `decompose_task`, `route_subtask`
4. Auto-instrument: FastAPI, Redis, SQLAlchemy

### ACP Broker (Python)

Same pattern as orchestrator:
1. Add dependencies
2. Initialize tracer with service name "acp-broker"
3. Add manual spans for: `handle_run`, `route_to_agent`, `stream_events`

### Adapters (Node.js)

1. Add `@opentelemetry/*` dependencies to each adapter's `package.json`
2. Import `tracing.ts` at entry point
3. Add manual spans for: `agent_run`, `execute_task`, `stream_output`

### LiteLLM Proxy

Option A: Use built-in telemetry
```bash
export LITELLM_TRACING=otel
```

Option B: Manual instrumentation (if running custom fork):
```python
from opentelemetry import trace

tracer = trace.get_tracer("litellm")

async def custom_completion(model, messages):
    with tracer.start_as_current_span("llm_completion") as span:
        span.set_attribute("llm.model", model)
        result = await litellm.completion(model, messages)
        span.set_attribute("llm.usage.total_tokens", result.usage.total_tokens)
        return result
```

---

## Port Allocations

| Service | Type | Port | Purpose |
|---|---|---|---|
| tempo-external | LoadBalancer | 31902 | LAN access |
| tempo-tailscale | NodePort | 32305 | Tailscale serve |

Cross-reference against existing:
* Logging: 31900 (loki-external), 32300 (grafana-tailscale), 32301 (loki-tailscale)
* Metrics: 31901 (prometheus-external), 32302 (prometheus-tailscale)
* **Tracing: 31902, 32305 — no conflicts**

---

## Open Questions (Answer Before Implementation)

### Q1: Tempo StorageClass

Run: `kubectl get storageclass`

Options:
- `local-path` (default, k3s) — recommended
- Custom StorageClass — if you have specific storage needs
- MinIO S3 — if you want object storage instead of filesystem

### Q2: Node IP for LoadBalancer

The `tempo-external` LoadBalancer will get an IP from the k3s loadbalancer controller (likely `10.0.0.7`). Verify:

```bash
kubectl get svc tempo-external -n tracing -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Update the `TEMPO_PORT_EXTERNAL` variable in `test.sh` if the IP differs from `10.0.0.7`.

### Q3: Tailscale Service Approval

After running `./apply.sh` in the `tailscale/` directory, approve the `svc:tempo` service at `login.tailscale.com/admin/machines`. This is a manual step that cannot be automated.

---

## Dependencies on Existing Projects

| Dependency | Status | Notes |
|---|---|---|
| k3s cluster | Deployed | Single-node cluster on melody-beast |
| Grafana | Deployed | In `logging` namespace |
| Prometheus | Deployed | In `metrics` namespace |
| Alloy | Deployed | ConfigMap patching needed for Tempo metrics |
| Dark Factory | Planned | Will be instrumented after tracing is deployed |
| LiteLLM | Deployed | Optional instrumentation for LLM calls |

---

## Rollback Plan

If Tempo doesn't start properly:

1. Check PVC status: `kubectl get pvc -n tracing`
2. Check pod logs: `kubectl logs -n tracing -l app=tempo`
3. Check events: `kubectl get events -n tracing --sort-by='.lastTimestamp'`
4. Check resource limits: `kubectl describe deploy tempo -n tracing`

Common issues:
* PVC not binding → Check StorageClass
* CrashLoopBackOff → Check resource limits, memory especially
* ConfigMap issues → Check Helm values

---

## Success Criteria

* [ ] `./install.sh` completes without errors
* [ ] All 27 tests pass
* [ ] Grafana shows Tempo datasource
* [ ] Submit test trace from a service → visible in Grafana TraceQL
* [ ] `./uninstall.sh --delete-data --delete-namespace --force` leaves no residue
* [ ] 3 destructive cycles pass (uninstall → install → test)
* [ ] Dark Factory services instrumented and sending traces
* [ ] TraceQL queries return meaningful data
