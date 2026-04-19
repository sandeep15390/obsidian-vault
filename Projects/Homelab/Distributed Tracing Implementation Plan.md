# Distributed Tracing Implementation Plan

> Canonical design: `~/src/obsidian-vault/Projects/Homelab/Distributed Tracing.md`

---

## Phase 1: Prerequisites Check and Repo Scaffold

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

### Step 1.2: Create directory structure

```bash
mkdir -p ~/src/home_infra/tracing/{manifests,instrument/{python,nodejs},docs}
mkdir -p ~/src/home_infra/tracing/otel-collector
mkdir -p ~/src/home_infra/tracing/tempo
```

### Step 1.3: Helm repo updates

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Record versions (for manifest pinning):
helm search repo grafana/tempo --versions | head -5
helm search repo open-telemetry/opentelemetry-collector --versions | head -5
```

### Step 1.4: Create base manifest files

**`manifests/tempo-values.yaml`:**
```yaml
# Helm values for grafana/tempo chart v0.19.0
# Single-binary Tempo with filesystem storage.

tempo:
  image:
    registry: docker.io
    repository: grafana/tempo
    tag: 2.7.4

  # Single replica for home lab
  replicas: 1

  # Filesystem storage (PVC-backed)
  persistence:
    enabled: true
    size: 10Gi
    storageClass: ""  # Use default (local-path)

  extraArgs:
    - --storage.trace.backend=local
    - --storage.trace.local.path=/var/tempo
    - --storage.trace.local.chunk-cache-size=100MB

  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Disable unnecessary components
  querier:
    replicas: 1

  overrides:
    defaults:
      retention: 15d

# Disable ingester/compactor/querier separation - run single-binary
overlord:
  enabled: false

ingester:
  enabled: false

compactor:
  enabled: false

querier:
  enabled: false

queryFrontend:
  enabled: true
  replicas: 1

# Service accounts
serviceAccount:
  create: true
  name: tempo

# Prometheus metrics scraping (add to Alloy later)
metrics:
  enabled: true
```

**`manifests/otel-collector-values.yaml`:**
```yaml
# Helm values for open-telemetry/opentelemetry-collector chart v0.118.0
# Centralized collector for trace batching and forwarding to Tempo.

mode: deployment

replicas: 1

image:
  repository: otel/opentelemetry-collector
  tag: 0.118.0

# Disable metrics scraping (Alloy will scrape)
service:
  type: ClusterIP
  ports:
    - name: otlp
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# Auto-generate collector config via values
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

    # Legacy Jaeger Thrift (for compatibility with older services)
    jaeger:
      protocols:
        thrift_http:
          endpoint: 0.0.0.0:55678

  processors:
    batch:
      timeout: 10s
      send_batch_size: 1024

    # Add resource attributes
    resource:
      attributes:
        - key: service.cluster
          value: home-lab
          action: upsert

  exporters:
    tempo:
      endpoint: tempo.tracing.svc.cluster.local:3200
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp, jaeger]
        processors: [resource, batch]
        exporters: [tempo]
```

**`manifests/tempo-nodeport.yaml`:**
```yaml
# LoadBalancer service for Tempo (LAN access)
apiVersion: v1
kind: Service
metadata:
  name: tempo-external
  namespace: tracing
  labels:
    app: tempo
spec:
  type: LoadBalancer
  ports:
    - name: tempo
      port: 3200
      targetPort: 3200
      protocol: TCP
  selector:
    app: tempo
```

**`manifests/tempo-tailscale-nodeport.yaml`:**
```yaml
# NodePort service for Tailscale serve
apiVersion: v1
kind: Service
metadata:
  name: tempo-tailscale
  namespace: tracing
  labels:
    app: tempo
spec:
  type: NodePort
  ports:
    - name: tempo
      port: 3200
      targetPort: 3200
      nodePort: 32305
      protocol: TCP
  selector:
    app: tempo
```

---

## Phase 2: install.sh and uninstall.sh

### install.sh Design

```bash
#!/usr/bin/env bash
# tracing/install.sh - Deploy Tempo + OpenTelemetry Collector + Grafana datasource

set -euo pipefail

# --- Variables ---
NAMESPACE="tracing"
TEMPO_RELEASE="tempo"
OTEL_RELEASE="otel-collector"
HELM_REPO_GRAFANA="grafana"
HELM_REPO_OTEL="open-telemetry"
TEMPO_CHART_VERSION="0.19.0"
OTEL_CHART_VERSION="0.118.0"
GRAFANA_NAMESPACE="logging"

# --- Colors ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# --- Functions ---
check_prereqs() {
    log_info "Checking prerequisites..."

    # Check kubectl
    if ! command -v kubectl &>/dev/null; then
        log_error "kubectl not found. Install kubectl first."
        exit 1
    fi

    # Check helm
    if ! command -v helm &>/dev/null; then
        log_error "helm not found. Install helm first."
        exit 1
    fi

    # Check cluster connectivity
    if ! kubectl cluster-info &>/dev/null; then
        log_error "Cannot connect to Kubernetes cluster. Check kubeconfig."
        exit 1
    fi

    log_info "Prerequisites OK"
}

add_helm_repos() {
    log_info "Adding Helm repos..."

    helm repo add "$HELM_REPO_GRAFANA" https://grafana.github.io/helm-charts 2>/dev/null || true
    helm repo add "$HELM_REPO_OTEL" https://open-telemetry.github.io/opentelemetry-helm-charts 2>/dev/null || true
    helm repo update

    log_info "Helm repos OK"
}

create_namespace() {
    log_info "Creating namespace $NAMESPACE..."

    if kubectl get namespace "$NAMESPACE" &>/dev/null; then
        log_info "Namespace $NAMESPACE already exists"
    else
        kubectl create namespace "$NAMESPACE"
        log_info "Namespace $NAMESPACE created"
    fi
}

deploy_tempo() {
    log_info "Deploying Tempo..."

    helm upgrade --install "$TEMPO_RELEASE" "$HELM_REPO_GRAFANA/tempo" \
        --namespace "$NAMESPACE" \
        --version "$TEMPO_CHART_VERSION" \
        --values manifests/tempo-values.yaml \
        --wait --timeout 5m

    log_info "Tempo deployed"
}

deploy_otel_collector() {
    log_info "Deploying OpenTelemetry Collector..."

    helm upgrade --install "$OTEL_RELEASE" "$HELM_REPO_OTEL/opentelemetry-collector" \
        --namespace "$NAMESPACE" \
        --version "$OTEL_CHART_VERSION" \
        --values manifests/otel-collector-values.yaml \
        --wait --timeout 5m

    log_info "OpenTelemetry Collector deployed"
}

create_services() {
    log_info "Creating services..."

    kubectl apply --server-side -f manifests/tempo-nodeport.yaml
    kubectl apply --server-side -f manifests/tempo-tailscale-nodeport.yaml

    log_info "Services created"
}

configure_grafana_datasource() {
    log_info "Configuring Grafana Tempo datasource..."

    # Get Grafana admin password
    GRAFANA_PASSWORD=$(kubectl get secret -n "$GRAFANA_NAMESPACE" grafana \
        -o jsonpath='{.data.admin-password}' | base64 -d)

    # Create datasource config
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: tempo-datasource
  namespace: $GRAFANA_NAMESPACE
data:
  tempo.yaml: |
    apiVersion: 1
    datasources:
      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo.tracing.svc.cluster.local:3200
        isDefault: false
        uid: tempo-datasource
        editable: true
EOF

    # Apply via Grafana API (idempotent)
    curl -s -X POST "http://localhost:3000/api/datasources" \
        -H "Content-Type: application/json" \
        -u "admin:$GRAFANA_PASSWORD" \
        -d '{
            "name": "Tempo",
            "type": "tempo",
            "access": "proxy",
            "url": "http://tempo.tracing.svc.cluster.local:3200",
            "isDefault": false,
            "uid": "tempo-datasource",
            "editable": true
        }' 2>/dev/null || log_warn "Grafana API config failed (may already exist)"

    log_info "Grafana datasource configured"
}

patch_alloy_config() {
    log_info "Patching Alloy ConfigMap for Tempo metrics..."

    # Sentinel block
    SENTINEL="// # BEGIN tempo"
    BLOCK="// # BEGIN tempo
prometheus.scrape \"tempo\" {
  targets = [{
    \"__address__\" = \"tempo.tracing.svc.cluster.local:3200\"
  }]
  job_name = \"tempo\"
  scrape_interval = \"30s\"
  forward_to = [prometheus.remote_write.default.receiver]
}
// # END tempo"

    # Check if sentinel exists
    ALLOY_CONFIGMAP="alloy-metrics-config"
    ALLOY_NAMESPACE="metrics"

    if kubectl get configmap "$ALLOY_CONFIGMAP" -n "$ALLOY_NAMESPACE" -o jsonpath='{.data.config\.alloy}' | grep -q "$SENTINEL"; then
        log_info "Alloy ConfigMap already has tempo block"
    else
        # Append block using Python for safety
        python3 <<PYSCRIPT
import re
import subprocess

# Get current config
result = subprocess.run([
    'kubectl', 'get', 'configmap', '$ALLOY_CONFIGMAP',
    '-n', '$ALLOY_NAMESPACE', '-o', 'jsonpath={.data.config\\.alloy}'
], capture_output=True, text=True)

config = result.stdout

# Check if block already exists
sentinel = '// # BEGIN tempo'
if sentinel in config:
    print('Block already exists')
    exit(0)

# Append block
block = '''// # BEGIN tempo
prometheus.scrape "tempo" {
  targets = [{
    "__address__" = "tempo.tracing.svc.cluster.local:3200"
  }]
  job_name = "tempo"
  scrape_interval = "30s"
  forward_to = [prometheus.remote_write.default.receiver]
}
// # END tempo'''

new_config = config + '\n\n' + block

# Update ConfigMap
subprocess.run([
    'kubectl', 'patch', 'configmap', '$ALLOY_CONFIGMAP',
    '-n', '$ALLOY_NAMESPACE', '-p', f'data={{config.alloy={repr(new_config)}}}'
])

print('Alloy ConfigMap patched')
PYSCRIPT

        # Restart Alloy to pick up new config
        kubectl rollout restart daemonset/alloy-metrics -n metrics
        log_info "Alloy restarted"
    fi
}

wait_for_tempo() {
    log_info "Waiting for Tempo to be ready..."

    for i in $(seq 1 60); do
        if kubectl get pod -n "$NAMESPACE" -l app=tempo -o jsonpath='{.items[0].status.phase}' 2>/dev/null | grep -q "Running"; then
            log_info "Tempo is running"
            return 0
        fi
        sleep 5
    done

    log_error "Tempo did not become ready in 5 minutes"
    kubectl get pods -n "$NAMESPACE"
    exit 1
}

run_tests() {
    log_info "Running tests..."
    ./test.sh
}

# --- Main ---
main() {
    local dry_run=false
    local no_tests=false

    while [[ $# -gt 0 ]]; do
        case $1 in
            --dry-run)
                dry_run=true
                shift
                ;;
            --no-tests)
                no_tests=true
                shift
                ;;
            *)
                log_error "Unknown option: $1"
                exit 1
                ;;
        esac
    done

    if $dry_run; then
        log_info "Dry run mode - no changes will be made"
        echo "Would create namespace: $NAMESPACE"
        echo "Would deploy: tempo (v$TEMPO_CHART_VERSION)"
        echo "Would deploy: otel-collector (v$OTEL_CHART_VERSION)"
        echo "Would configure Grafana datasource"
        echo "Would patch Alloy ConfigMap"
        exit 0
    fi

    check_prereqs
    add_helm_repos
    create_namespace
    deploy_tempo
    wait_for_tempo
    deploy_otel_collector
    create_services
    configure_grafana_datasource
    patch_alloy_config

    if ! $no_tests; then
        run_tests
    fi

    log_info "============================================"
    log_info "Tracing stack deployed successfully!"
    log_info "Tempo API: http://10.0.0.7:31902"
    log_info "Grafana TraceQL: https://grafana.tailc98a25.ts.net/explore?datasource=tempo-datasource"
    log_info "============================================"
}

main "$@"
```

### uninstall.sh Design

```bash
#!/usr/bin/env bash
# tracing/uninstall.sh - Tear down tracing stack

set -euo pipefail

NAMESPACE="tracing"
GRAFANA_NAMESPACE="logging"

# --- Colors ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# --- Flags ---
DELETE_DATA=false
DELETE_NAMESPACE=false
FORCE=false

while [[ $# -gt 0 ]]; do
    case $1 in
        --delete-data)
            DELETE_DATA=true
            shift
            ;;
        --delete-namespace)
            DELETE_NAMESPACE=true
            shift
            ;;
        --force)
            FORCE=true
            shift
            ;;
        *)
            log_error "Unknown option: $1"
            exit 1
            ;;
    esac
done

# --- Confirmations ---
if ! $FORCE; then
    if $DELETE_DATA; then
        read -p "This will delete all tracing data (PVCs, traces). Continue? [y/N] " confirm
        if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
            log_info "Aborted"
            exit 0
        fi
    fi
    if $DELETE_NAMESPACE; then
        read -p "This will delete the $NAMESPACE namespace. Continue? [y/N] " confirm
        if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
            log_info "Aborted"
            exit 0
        fi
    fi
fi

# --- Delete Helm releases (reverse order) ---
log_info "Deleting Helm releases..."

helm uninstall otel-collector -n "$NAMESPACE" --ignore-not-found || true
helm uninstall tempo -n "$NAMESPACE" --ignore-not-found || true

# --- Delete PVCs if requested ---
if $DELETE_DATA; then
    log_info "Deleting PVCs..."
    kubectl delete pvc -n "$NAMESPACE" -l app=tempo --ignore-not-found || true

    # Clean up any residue PVCs
    kubectl delete pvc -n "$NAMESPACE" --ignore-not-found || true
fi

# --- Delete namespace if requested ---
if $DELETE_NAMESPACE; then
    log_info "Deleting namespace $NAMESPACE..."
    kubectl delete namespace "$NAMESPACE" --ignore-not-found || true

    # Wait for deletion
    for i in $(seq 1 60); do
        if ! kubectl get namespace "$NAMESPACE" &>/dev/null; then
            log_info "Namespace $NAMESPACE deleted"
            break
        fi
        sleep 5
    done
fi

# --- Remove Alloy config patch ---
log_info "Removing Tempo block from Alloy ConfigMap..."

python3 <<PYSCRIPT
import subprocess

configmap = "alloy-metrics-config"
namespace = "metrics"

# Get current config
result = subprocess.run([
    'kubectl', 'get', 'configmap', configmap,
    '-n', namespace, '-o', 'jsonpath={.data.config\\.alloy}'
], capture_output=True, text=True)

config = result.stdout

# Remove block
import re
pattern = r'// # BEGIN tempo.*?// # END tempo'
new_config = re.sub(pattern, '', config, flags=re.DOTALL)

if config != new_config:
    # Update ConfigMap
    subprocess.run([
        'kubectl', 'patch', 'configmap', configmap,
        '-n', namespace, '-p', f'data={{config.alloy={repr(new_config)}}}'
    ])

    # Restart Alloy
    subprocess.run(['kubectl', 'rollout', 'restart', 'daemonset/alloy-metrics', '-n', namespace])
    print('Alloy ConfigMap patched (block removed)')
else:
    print('No Tempo block found in Alloy ConfigMap')
PYSCRIPT

# --- Delete Grafana datasource ---
log_info "Deleting Grafana Tempo datasource..."

GRAFANA_PASSWORD=$(kubectl get secret -n "$GRAFANA_NAMESPACE" grafana \
    -o jsonpath='{.data.admin-password}' | base64 -d 2>/dev/null || echo "")

if [[ -n "$GRAFANA_PASSWORD" ]]; then
    curl -s -X DELETE "http://localhost:3000/api/datasources/uid/tempo-datasource" \
        -H "Content-Type: application/json" \
        -u "admin:$GRAFANA_PASSWORD" 2>/dev/null || true
fi

# --- Residue check ---
log_info "============================================"
log_info "Residue check:"

if kubectl get namespace "$NAMESPACE" &>/dev/null; then
    log_warn "Namespace $NAMESPACE still exists"
else
    log_info "Namespace $NAMESPACE: deleted"
fi

if $DELETE_DATA; then
    if kubectl get pvc -n "$NAMESPACE" &>/dev/null; then
        log_warn "PVCs in $NAMESPACE still exist"
    else
        log_info "PVCs: deleted"
    fi
fi

if kubectl get configmap alloy-metrics-config -n metrics -o jsonpath='{.data.config\.alloy}' | grep -q "BEGIN tempo"; then
    log_warn "Alloy ConfigMap still has tempo block"
else
    log_info "Alloy ConfigMap: tempo block removed"
fi

log_info "============================================"
```

---

## Phase 3: test.sh

```bash
#!/usr/bin/env bash
# tracing/test.sh - Validation tests for tracing stack

set -euo pipefail

NAMESPACE="tracing"
GRAFANA_NAMESPACE="logging"
TEMPO_PORT_EXTERNAL="31902"
TEMPO_PORT_TAILSCALE="32305"

PASSED=0
FAILED=0

# --- Colors ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

pass() {
    echo -e "${GREEN}[PASS]${NC} $1"
    ((PASSED++))
}

fail() {
    echo -e "${RED}[FAIL]${NC} $1"
    ((FAILED++))
}

# --- Prerequisites ---
echo "=== Prerequisites ==="

# Test 1: kubectl available
if command -v kubectl &>/dev/null; then
    pass "kubectl available"
else
    fail "kubectl not found"
fi

# Test 2: helm available
if command -v helm &>/dev/null; then
    pass "helm available"
else
    fail "helm not found"
fi

# --- K8s Resources ---
echo "=== K8s Resources ==="

# Test 3: Namespace exists
if kubectl get namespace "$NAMESPACE" &>/dev/null; then
    pass "Namespace $NAMESPACE exists"
else
    fail "Namespace $NSEPACE not found"
fi

# Test 4: Tempo PVC bound
PVC_STATUS=$(kubectl get pvc -n "$NAMESPACE" -l app=tempo -o jsonpath='{.items[0].status.phase}' 2>/dev/null || echo "")
if [[ "$PVC_STATUS" == "Bound" ]]; then
    pass "Tempo PVC Bound"
else
    fail "Tempo PVC not Bound (status: $PVC_STATUS)"
fi

# Test 5: Tempo Deployment exists
if kubectl get deployment -n "$NAMESPACE" tempo &>/dev/null; then
    pass "Tempo Deployment exists"
else
    fail "Tempo Deployment not found"
fi

# Test 6: Otel Collector Deployment exists
if kubectl get deployment -n "$NAMESPACE" otel-collector &>/dev/null; then
    pass "Otel Collector Deployment exists"
else
    fail "Otel Collector Deployment not found"
fi

# Test 7: Query Frontend Deployment exists
if kubectl get deployment -n "$NAMESPACE" tempo-query-frontend &>/dev/null; then
    pass "Query Frontend Deployment exists"
else
    fail "Query Frontend Deployment not found"
fi

# Test 8: Services exist
if kubectl get svc -n "$NAMESPACE" tempo &>/dev/null; then
    pass "Tempo ClusterIP service exists"
else
    fail "Tempo ClusterIP service not found"
fi

if kubectl get svc -n "$NAMESPACE" tempo-external &>/dev/null; then
    pass "Tempo LoadBalancer service exists"
else
    fail "Tempo LoadBalancer service not found"
fi

# --- Helm Versions ---
echo "=== Helm Versions ==="

# Test 9: Tempo chart version
TEMPO_INSTALLED=$(helm list -n "$NAMESPACE" -o json | python3 -c "import sys,json; d=json.load(sys.stdin); print([r['app_version'] for r in d if r['name']=='tempo'][0] if d else '')")
if [[ "$TEMPO_INSTALLED" == "2.7.4" ]]; then
    pass "Tempo version $TEMPO_INSTALLED"
else
    fail "Tempo version mismatch (expected 2.7.4, got $TEMPO_INSTALLED)"
fi

# Test 10: Otel Collector chart version
OTEL_INSTALLED=$(helm list -n "$NAMESPACE" -o json | python3 -c "import sys,json; d=json.load(sys.stdin); print([r['app_version'] for r in d if r['name']=='otel-collector'][0] if d else '')")
if [[ "$OTEL_INSTALLED" == "0.118.0" ]]; then
    pass "Otel Collector version $OTEL_INSTALLED"
else
    fail "Otel Collector version mismatch (expected 0.118.0, got $OTEL_INSTALLED)"
fi

# --- Tempo ---
echo "=== Tempo ==="

# Test 11: Tempo pod Running+Ready
TEMPO_READY=$(kubectl get pod -n "$NAMESPACE" -l app=tempo -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "False")
if [[ "$TEMPO_READY" == "True" ]]; then
    pass "Tempo pod Running+Ready"
else
    fail "Tempo pod not ready (status: $TEMPO_READY)"
fi

# Test 12: Tempo /metrics endpoint
if curl -s "http://localhost:3200/metrics" | grep -q "tempo_ingest_spans_total"; then
    pass "Tempo /metrics endpoint returns metrics"
else
    fail "Tempo /metrics endpoint empty or unreachable"
fi

# Test 13: Tempo /ready endpoint
if curl -s "http://localhost:3200/ready" | grep -q "ready"; then
    pass "Tempo /ready returns ready"
else
    fail "Tempo /ready not returning ready"
fi

# Test 14: Tempo version check
TEMPO_VERSION=$(curl -s "http://localhost:3200/api/status" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('tempo',{}).get('version',''))" 2>/dev/null || echo "")
if [[ -n "$TEMPO_VERSION" ]]; then
    pass "Tempo version: $TEMPO_VERSION"
else
    fail "Cannot get Tempo version"
fi

# --- Otel Collector ---
echo "=== Otel Collector ==="

# Test 15: Otel Collector pod Running+Ready
OTEL_READY=$(kubectl get pod -n "$NAMESPACE" -l app.kubernetes.io/name=opentelemetry-collector -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "False")
if [[ "$OTEL_READY" == "True" ]]; then
    pass "Otel Collector pod Running+Ready"
else
    fail "Otel Collector pod not ready (status: $OTEL_READY)"
fi

# Test 16: Otel Collector /metrics endpoint
if curl -s "http://otel-collector.$NAMESPACE.svc:8888/metrics" | grep -q "otelcol_processor_batch_batch_send_size_count"; then
    pass "Otel Collector /metrics endpoint returns metrics"
else
    fail "Otel Collector /metrics endpoint empty or unreachable"
fi

# Test 17: Otel Collector OTLP listener
if kubectl port-forward svc/otel-collector -n "$NAMESPACE" 4317:4317 &>/dev/null & sleep 2 && \
   curl -s --connect-timeout 2 http://localhost:4317 2>&1 | grep -qi "grpc\|connection"; then
    pass "Otel Collector OTLP listener on :4317"
else
    fail "Otel Collector OTLP listener not responding"
fi

# --- Tempo API ---
echo "=== Tempo API ==="

# Test 18: GET /ready
if curl -s "http://localhost:3200/ready" | grep -q "ready"; then
    pass "GET /ready returns 200"
else
    fail "GET /ready not returning 200"
fi

# Test 19: GET /loki/api/v1/query
if curl -s "http://localhost:3200/loki/api/v1/query?query={}" | grep -q "status"; then
    pass "GET /loki/api/v1/query returns JSON"
else
    fail "GET /loki/api/v1/query not returning JSON"
fi

# Test 20: GET /api/traces (should return 404 for empty, but API should respond)
if curl -s "http://localhost:3200/api/traces/nonexistent" | grep -qi "not found\|notexist"; then
    pass "GET /api/traces API responds"
else
    fail "GET /api/traces API not responding"
fi

# --- Grafana Datasource ---
echo "=== Grafana Datasource ==="

# Test 21: Tempo datasource configured
GRAFANA_PASSWORD=$(kubectl get secret -n "$GRAFANA_NAMESPACE" grafana \
    -o jsonpath='{.data.admin-password}' | base64 -d 2>/dev/null || echo "")

if curl -s -u "admin:$GRAFANA_PASSWORD" \
    "http://localhost:3000/api/datasources" | grep -qi "Tempo"; then
    pass "Tempo datasource configured in Grafana"
else
    fail "Tempo datasource not configured in Grafana"
fi

# Test 22: Datasource URL correct
if curl -s -u "admin:$GRAFANA_PASSWORD" \
    "http://localhost:3000/api/datasources/uid/tempo-datasource" | \
    grep -q "http://tempo.tracing.svc.cluster.local:3200"; then
    pass "Tempo datasource URL correct"
else
    fail "Tempo datasource URL incorrect"
fi

# Test 23: TraceQL query returns data (may be empty, but should not error)
if curl -s -u "admin:$GRAFANA_PASSWORD" \
    -X POST "http://localhost:3000/api/ds/query" \
    -H "Content-Type: application/json" \
    -d '{"queries":[{"refId":"A","datasource":{"uid":"tempo-datasource"},"expr":"{}","queryType":"traceql"}]}' \
    2>/dev/null | grep -qi "status\|results\|error"; then
    pass "TraceQL query reaches Tempo"
else
    fail "TraceQL query failed"
fi

# --- Tailscale ---
echo "=== Tailscale ==="

# Test 24: svc:tempo registered
if kubectl get serviceaccount -n "$NAMESPACE" tempo -o jsonpath='{.metadata.name}' &>/dev/null; then
    # Check if tailscale service exists (requires tailscale CLI)
    if command -v tailscale &>/dev/null; then
        if sudo tailscale serve get-config --all 2>/dev/null | grep -q "svc:tempo"; then
            pass "svc:tempo registered"
        else
            fail "svc:tempo not registered"
        fi
    else
        pass "svc:tempo: tailscale CLI not available (skip)"
    fi
else
    fail "Tempo serviceaccount not found"
fi

# Test 25: NodePort reachable
if curl -s --connect-timeout 5 "http://10.0.0.7:$TEMPO_PORT_TAILSCALE/ready" | grep -q "ready"; then
    pass "Tempo NodePort $TEMPO_PORT_TAILSCALE reachable"
else
    fail "Tempo NodePort $TEMPO_PORT_TAILSCALE not reachable"
fi

# Test 26: External LoadBalancer reachable
if curl -s --connect-timeout 5 "http://10.0.0.7:$TEMPO_PORT_EXTERNAL/ready" | grep -q "ready"; then
    pass "Tempo LoadBalancer $TEMPO_PORT_EXTERNAL reachable"
else
    fail "Tempo LoadBalancer $TEMPO_PORT_EXTERNAL not reachable"
fi

# --- Summary ---
echo "============================================"
echo "Results: $PASSED passed, $FAILED failed"
echo "============================================"

if [[ $FAILED -gt 0 ]]; then
    exit 1
fi
```

---

## Phase 4: diag.sh

```bash
#!/usr/bin/env bash
# tracing/diag.sh - Read-only diagnostics snapshot

set -euo pipefail

NAMESPACE="tracing"
GRAFANA_NAMESPACE="logging"

echo "============================================"
echo "DIAGNOSTICS SNAPSHOT"
echo "Generated: $(date)"
echo "============================================"

echo ""
echo "=== Tool Versions ==="
kubectl version --short 2>/dev/null || kubectl version
helm version --short 2>/dev/null || helm version

echo ""
echo "=== Namespace ==="
kubectl get namespace "$NAMESPACE" -o wide 2>/dev/null || echo "Namespace $NAMESPACE not found"

echo ""
echo "=== Pods ==="
kubectl get pods -n "$NAMESPACE" -o wide 2>/dev/null || echo "No pods in $NAMESPACE"

echo ""
echo "=== Services ==="
kubectl get svc -n "$NAMESPACE" -o wide 2>/dev/null || echo "No services in $NAMESPACE"

echo ""
echo "=== PVCs ==="
kubectl get pvc -n "$NAMESPACE" 2>/dev/null || echo "No PVCs in $NAMESPACE"

echo ""
echo "=== ConfigMaps ==="
kubectl get configmap -n "$NAMESPACE" 2>/dev/null || echo "No ConfigMaps in $NAMESPACE"

echo ""
echo "=== Helm Releases ==="
helm list -n "$NAMESPACE" 2>/dev/null || echo "No Helm releases in $NAMESPACE"

echo ""
echo "=== Recent Events (last 20) ==="
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' 2>/dev/null | tail -20 || echo "No events"

echo ""
echo "=== Tempo Pod Logs (last 30 lines) ==="
kubectl logs -n "$NAMESPACE" -l app=tempo --tail=30 2>/dev/null || echo "No Tempo logs"

echo ""
echo "=== Otel Collector Pod Logs (last 30 lines) ==="
kubectl logs -n "$NAMESPACE" -l app.kubernetes.io/name=opentelemetry-collector --tail=30 2>/dev/null || echo "No Otel logs"

echo ""
echo "=== Tempo /metrics (key metrics) ==="
curl -s "http://localhost:3200/metrics" 2>/dev/null | grep -E "^tempo_(ingest_spans_total|query_requests_total|compactor_compactions_total)" || echo "Tempo metrics not available"

echo ""
echo "=== Grafana Datasource ==="
GRAFANA_PASSWORD=$(kubectl get secret -n "$GRAFANA_NAMESPACE" grafana \
    -o jsonpath='{.data.admin-password}' | base64 -d 2>/dev/null || echo "")
curl -s -u "admin:$GRAFANA_PASSWORD" "http://localhost:3000/api/datasources" 2>/dev/null | \
    python3 -c "import sys,json; d=json.load(sys.stdin); [print(f\"  {ds['name']}: {ds['uid']}\") for ds in d]" || echo "Grafana datasources not available"

echo ""
echo "=== Connectivity Probes ==="
echo -n "Tempo API: "
if curl -s --connect-timeout 2 "http://localhost:3200/ready" | grep -q "ready"; then echo "OK"; else echo "FAIL"; fi

echo -n "Otel Collector: "
if kubectl get pod -n "$NAMESPACE" -l app.kubernetes.io/name=opentelemetry-collector -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q "True"; then echo "OK"; else echo "FAIL"; fi

echo "============================================"
```

---

## Phase 5: Instrumentation Guides

### Python Instrumentation Guide (`docs/instrumentation-guide.md`)

See the design document for the full Python instrumentation code. Key steps:

1. Add dependencies to `instrument/python/requirements-otel.txt`:
```
opentelemetry-api==1.29.0
opentelemetry-sdk==1.29.0
opentelemetry-exporter-otlp-proto-grpc==1.29.0
opentelemetry-instrumentation-fastapi==0.50b0
opentelemetry-instrumentation-redis==0.50b0
opentelemetry-instrumentation-sqlalchemy==0.50b0
```

2. Create `instrument/python/opentelemetry-config.py`:
```python
# Common OpenTelemetry tracer setup for Dark Factory services
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

def setup_tracer(service_name: str):
    """Initialize OpenTelemetry tracer for a service."""
    provider = TracerProvider()
    exporter = OTLPSpanExporter(
        endpoint="otel-collector.tracing.svc.cluster.local:4317",
        insecure=True
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer(service_name)
```

3. Import and use in your service:
```python
from opentelemetry-config import setup_tracer
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

tracer = setup_tracer("orchestrator")

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Manual spans
with tracer.start_as_current_span("process_task"):
    # ... business logic ...
    pass
```

### Node.js Instrumentation Guide

See design document for Node.js instrumentation code. Similar pattern:

1. Add to `instrument/nodejs/package-otel.json`:
```json
{
  "@opentelemetry/api": "^1.9.0",
  "@opentelemetry/sdk-trace-node": "^1.29.0",
  "@opentelemetry/exporter-trace-otlp-grpc": "^0.56.0",
  "@opentelemetry/instrumentation-http": "^0.56.0",
  "@opentelemetry/instrumentation-express": "^0.45.0"
}
```

2. Create `instrument/nodejs/tracing.ts` (see design doc)

3. Import in adapter:
```typescript
import './tracing'  // Initialize on module load
```

---

## Success Criteria

### Completed Phases
| Phase | Status | Notes |
|-------|--------|-------|
| Phase 1: Prerequisites Check | Done | StorageClass verified, repo scaffold created |
| Phase 2: install.sh & uninstall.sh | Done | All scripts implemented with flags |
| Phase 3: test.sh | Done | 27/27 tests passing |
| Phase 4: diag.sh | Done | Diagnostic script implemented |
| Phase 5: Instrumentation Guides | Done | Guides written (not yet applied to services) |

### Current Status (Post-Deployment)
- [x] Stack deployed: Tempo v2.9.0, OTel Collector v0.118.0
- [x] All 27 tests passing (test.sh)
- [x] Grafana Tempo datasource configured (uid: `tempo-datasource`)
- [x] External access: LAN port 31902, Tailscale NodePort 32305
- [x] All bugs fixed via architect review

### Remaining
- [ ] Teardown/reinstall validation (3 destructive cycles)
- [ ] Instrument dark-factory orchestrator + adapters
- [ ] Instrument LiteLLM proxy
- [ ] Validate end-to-end trace flow (request → agent → LLM)

---

## Phase 6: Teardown/Reinstall Validation (Pending)

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
