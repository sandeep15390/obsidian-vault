# PostgreSQL Metrics (postgres\_exporter + Prometheus + Grafana)

> Scripts and manifests: `~/src/home_infra/postgres/metrics/`
> Extends the base metrics stack — see [[Metrics]]. Grafana lives in the logging namespace — see [[Logging]]. PostgreSQL is deployed in the `postgresql` namespace — see [[Postgres]].

## Status

- [x] Create directory `~/src/home_infra/postgres/metrics/`
- [x] Write `manifests/postgres-exporter-deployment.yaml`
- [x] Write `manifests/postgres-exporter-service.yaml`
- [x] Write `metrics/install.sh`
- [x] Write `metrics/uninstall.sh`
- [x] Write `metrics/test.sh`
- [x] Patch `postgres/install.sh` — add `metrics/install.sh --no-tests` call after StatefulSet ready
- [x] Patch `postgres/uninstall.sh` — add `metrics/uninstall.sh --force` call before StatefulSet deletion
- [x] All 22 metrics tests passing
- [x] `pg_up = 1` in Prometheus
- [x] Alloy component `prometheus.scrape.postgresql` registered
- [x] Grafana dashboard uid `postgres-overview` provisioned
- [x] Grafana dashboard uid `postgres-logs` provisioned
- [x] Validate teardown/reinstall reproducibility — 3 destructive cycles ✅

---

## Stack

| Component | Role | Image / Source | Version | Notes |
|---|---|---|---|---|
| **postgres\_exporter** | Translates PostgreSQL stats to Prometheus metrics | `prometheuscommunity/postgres_exporter` Docker image | `v0.17.1` | Standalone Deployment in `postgresql` ns; reads password from existing `postgresql` secret |
| **Prometheus** | Metrics storage + PromQL engine | Already deployed in `metrics` ns | — | Datasource UID `afilsg21fj18ga`; Alloy scrapes exporter and remote\_writes to Prometheus |
| **Grafana Alloy** | Scraper — adds postgres\_exporter as a new target | Already deployed as `alloy-metrics` in `metrics` ns | — | ConfigMap `alloy-metrics-config` patched with PostgreSQL scrape block using `// # BEGIN postgresql` / `// # END postgresql` sentinels |
| **Grafana** | Dashboard visualization | Already deployed in `logging` ns | — | Two dashboards: `postgres-overview` (downloaded from grafana.com ID 9628) and `postgres-logs` (custom-built in install.sh) |
| **Loki** | Log aggregation | Already deployed in `logging` ns | — | Datasource UID `P8E80F9AEF21F6940`; PostgreSQL pod logs already flowing via Alloy kubernetes pod discovery |

---

## Architecture

```mermaid
graph TD
    subgraph PostgreSQL["PostgreSQL Namespace"]
        PGPod["postgresql-0\n:5432"]
        PGSecret["Secret: postgresql\nkey: postgres-password"]
        ExporterDeploy["postgres-exporter Deployment\n:9187/metrics\nDATA_SOURCE_NAME from secret"]
        ExporterSvc["ClusterIP svc\npostgres-exporter :9187\ninternal only"]
    end

    subgraph Metrics["Metrics Namespace"]
        Alloy["Alloy-metrics DaemonSet\nscrape postgres-exporter.postgresql.svc:9187"]
        Prom["Prometheus\nTSDB storage + PromQL\nUID: afilsg21fj18ga"]
    end

    subgraph Logging["Logging Namespace"]
        Loki["Loki\nlogs from all namespaces\nUID: P8E80F9AEF21F6940"]
        Grafana["Grafana\ndashboard: postgres-overview\ndashboard: postgres-logs"]
    end

    PGSecret -->|"env PGPASSWORD → DATA_SOURCE_NAME"| ExporterDeploy
    ExporterDeploy -->|"pg_stat_* queries via libpq"| PGPod
    ExporterDeploy --> ExporterSvc
    Alloy -->|"scrape :9187 every 15s"| ExporterSvc
    Alloy -->|"remote_write"| Prom
    Alloy -->|"pod log discovery"| PGPod
    Alloy -->|"log forward"| Loki
    Prom -->|"PromQL datasource"| Grafana
    Loki -->|"LogQL datasource"| Grafana
```

### Data Flow

1. `postgres-exporter` Deployment runs in the `postgresql` namespace, co-located with the database pod
2. It connects to `postgresql.postgresql.svc.cluster.local:5432` and issues `pg_stat_*` queries
3. The connection string is assembled as `DATA_SOURCE_NAME=postgresql://postgres:$(PGPASSWORD)@postgresql.postgresql.svc.cluster.local:5432/homelab?sslmode=disable`; `PGPASSWORD` is injected via `secretKeyRef` from the existing `postgresql` secret (`postgres-password` key)
4. Alloy-metrics (in `metrics` ns) scrapes `postgres-exporter.postgresql.svc.cluster.local:9187/metrics` on a 15s interval
5. Alloy remote\_writes to Prometheus; Grafana queries via PromQL
6. PostgreSQL pod logs flow from the pod to Loki via the existing Alloy DaemonSet kubernetes pod log discovery (no additional config needed); Grafana queries via LogQL

---

## Namespace & Port Allocation

| Service | Namespace | Type | Port | Purpose |
|---|---|---|---|---|
| postgres-exporter | postgresql | ClusterIP | 9187 | Standard postgres\_exporter port; scraped by Alloy |

> No external (LoadBalancer / NodePort) ports allocated — the exporter is internal-only.

**Existing allocations (do not reuse):**

| Port | Service |
|---|---|
| 31900 | loki-external (LoadBalancer) |
| 31901 | prometheus-external (LoadBalancer) |
| 32300 | grafana-tailscale (NodePort) |
| 32301 | loki-tailscale (NodePort) |
| 32302 | prometheus-tailscale (NodePort) |

---

## Repo Layout

```
home_infra/postgres/
├── install.sh                          # PATCHED: calls metrics/install.sh --no-tests after StatefulSet ready
├── uninstall.sh                        # PATCHED: calls metrics/uninstall.sh --force before StatefulSet delete
├── test.sh                             # UNCHANGED: 32-test core suite; metrics tests invoked separately
├── diag.sh
└── metrics/                            # NEW sub-project
    ├── install.sh                      # --dry-run / --dashboard-only / --no-tests
    ├── uninstall.sh                    # --force required
    ├── test.sh                         # 22 metrics tests; --smoke-test for 3-test smoke subset
    ├── diag.sh                         # Read-only metrics diagnostics
    └── manifests/
        ├── postgres-exporter-deployment.yaml
        └── postgres-exporter-service.yaml
```

---

## Deploy / Teardown Commands

```bash
# Full install — PostgreSQL core + metrics (recommended entry point)
cd ~/src/home_infra/postgres
./install.sh

# Metrics sub-project operations
cd ~/src/home_infra/postgres/metrics

# Install metrics only (exporter + Alloy patch + both Grafana dashboards)
./install.sh

# Dry run (prints what would be done)
./install.sh --dry-run

# Re-upload Grafana dashboards only (no k8s resource changes)
./install.sh --dashboard-only

# Run metrics tests standalone (22 tests)
./test.sh

# Smoke test only (3 tests)
./test.sh --smoke-test

# Diagnostics (read-only)
./diag.sh

# Metrics-only teardown
./uninstall.sh --force

# Full PostgreSQL teardown (includes metrics)
cd ~/src/home_infra/postgres
./uninstall.sh --force
./uninstall.sh --delete-data --delete-namespace --force
```

---

## Test Suite (22 tests)

All tests use `kubectl` and port-forwarding from the host — no shell inside the exporter container is required (image is distroless).

| Category | Count | What's Validated |
|---|---|---|
| **Prerequisites** | 2 | `kubectl` available, `python3` available |
| **K8s Resources** | 4 | Deployment `postgres-exporter` exists, ≥ 1 ready replica, ClusterIP Service exists, type = ClusterIP |
| **Exporter Pod Health** | 3 | Pod phase = Running, Ready = True, restartCount ≤ 2 |
| **secretKeyRef Wiring** | 1 | `PGPASSWORD` env sourced from `secretKeyRef.name=postgresql` |
| **Exporter Metrics Endpoint** | 5 | `/metrics` returns data, `pg_up=1`, `pg_postmaster_start_time_seconds` present, `pg_stat_database_numbackends` present, `pg_settings_max_connections` present |
| **Alloy ConfigMap** | 2 | ConfigMap contains `# BEGIN postgresql` sentinel, block has `job_name = "postgresql"` |
| **Alloy Scrape + Prometheus** | 3 | Alloy component `prometheus.scrape.postgresql` registered, `pg_up=1` in Prometheus (retried up to 6×15s), `pg_stat_database_numbackends` present in Prometheus |
| **Grafana Dashboards** | 2 | Dashboard uid `postgres-overview` found via Grafana API, Dashboard uid `postgres-logs` found via Grafana API |

> **Total: 22 tests**

### Smoke test subset (`--smoke-test` flag, 3 tests)

1. `postgres-exporter` pod phase = Running
2. `pg_up=1` from direct `/metrics` endpoint (port-forward)
3. Dashboard uid `postgres-overview` found in Grafana

---

## Key Metrics Surfaced

| Metric | Meaning |
|---|---|
| `pg_up` | Exporter connected to DB (1=yes) — core health indicator |
| `pg_postmaster_start_time_seconds` | DB process start time — detects unexpected restarts |
| `pg_stat_database_numbackends` | Active connections per database — alert if near max\_connections |
| `pg_settings_max_connections` | Configured max\_connections |
| `pg_stat_bgwriter_checkpoint_write_time_total` | Checkpoint write duration — I/O health |
| `pg_stat_database_blks_hit` / `blks_read` | Buffer cache hit ratio — memory tuning signal |
| `pg_stat_database_tup_inserted/updated/deleted` | DML rates — activity baseline |
| `pg_stat_database_deadlocks` | Deadlock count — application locking issue signal |

---

## Dashboard Specifications

### `postgres-overview` (Grafana dashboard ID 9628)

- **Source:** Downloaded from `grafana.com/api/dashboards/9628/revisions/latest/download` at install time
- **Patching:** datasource UID set to `afilsg21fj18ga`, uid overridden to `postgres-overview`, title set to `PostgreSQL Overview`, job variable default set to `postgresql`

### `postgres-logs` (custom-built inline)

- **Source:** Built in Python inside `install.sh` — no external download
- **Datasource:** Loki UID `P8E80F9AEF21F6940`
- **Panels:**
  1. **PostgreSQL Logs** — `logs` panel, LogQL: `{namespace="postgresql"} |= ""`
  2. **Error Rate** — `timeseries`, LogQL: `sum(rate({namespace="postgresql"} |~ "(?i)(error|fatal|panic)" [5m]))`
  3. **Log Volume** — `barchart`, LogQL: `sum(rate({namespace="postgresql"}[5m]))`

---

## Teardown / Reinstall Validation Plan

Results documented in [[Teardown Reinstall Validation - Postgres]].

```bash
cd ~/src/home_infra/postgres

# Destructive cycle (full wipe — deletes PVC, namespace, metrics)
./uninstall.sh --delete-data --delete-namespace --force
./install.sh   # 32 core tests + 22 metrics tests must all pass

# Soft cycle (keep PVC/data)
./uninstall.sh --force
./install.sh
```

| Cycle | Type | Date | Core tests | Metrics tests |
|---|---|---|---|---|
| 1 | Destructive | 2026-04-15 | 32/32 | 22/22 |
| 2 | Destructive | 2026-04-15 | 32/32 | 22/22 |
| 3 | Destructive | 2026-04-15 | 32/32 | 22/22 |

---

## Troubleshooting

### `pg_up` is 0 or exporter in CrashLoopBackOff

```bash
kubectl logs deployment/postgres-exporter -n postgresql
kubectl get secret postgresql -n postgresql \
  -o jsonpath="{.data.postgres-password}" | base64 -d; echo
kubectl port-forward -n postgresql svc/postgres-exporter 19187:9187 &
curl -s http://localhost:19187/metrics | grep '^pg_up'
kill %1
```

### Alloy ConfigMap not patched

```bash
kubectl get configmap alloy-metrics-config -n metrics \
  -o jsonpath='{.data.config\.alloy}' | grep 'BEGIN postgresql'
# If missing: cd ~/src/home_infra/postgres/metrics && ./install.sh --no-tests
```

### Grafana shows "No data"

1. Verify datasource UID: `afilsg21fj18ga` for Prometheus, `P8E80F9AEF21F6940` for Loki
2. Wait 30–60s for first scrape
3. Check `job` variable in `postgres-overview` is set to `postgresql`

### `postgres-logs` shows no logs

```bash
# Verify Loki is receiving postgresql namespace logs
# In Grafana Explore: {namespace="postgresql"}
kubectl get daemonset alloy-metrics -n metrics
kubectl logs -l app.kubernetes.io/name=alloy -n metrics --tail=20
```

---

## See Also

- [[Postgres]] — PostgreSQL deployment; `postgres-exporter` targets the StatefulSet deployed there
- [[Metrics]] — Prometheus + Alloy base stack; Alloy ConfigMap is patched by this project
- [[Redis Metrics]] — Reference implementation this project follows exactly
- [[Logging]] — Grafana and Loki instances used for dashboards and logs
- [[Overview]] — Homelab overview and service registry
- [[Teardown Reinstall Validation - Postgres]] — Cycle results, including metrics validation
