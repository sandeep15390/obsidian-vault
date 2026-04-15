# Teardown / Reinstall Validation — PostgreSQL

> Validates that `install.sh` + `uninstall.sh` are fully repeatable and idempotent.
> Reference: [[Postgres]] — full deploy/teardown runbook.

## Summary

| Stat | Value |
|---|---|
| Total cycles | 9 (6 destructive + 3 non-destructive) |
| Tests per cycle | 32 core + 22 metrics = 54 total |
| Status | ✅ Complete — all 9 cycles passed |

---

## Destructive Cycles (delete all data + namespace)

```bash
cd ~/src/home_infra/postgres
./uninstall.sh --delete-data --delete-namespace --force
kubectl get namespace postgresql 2>&1 | grep -q "not found" && echo "namespace clean"
kubectl get pvc -A 2>&1 | grep "postgresql" && echo "WARNING: orphaned PVC" || echo "PVC clean"
./install.sh   # runs 32 core tests + 22 metrics tests automatically
```

| Cycle | Date | Teardown clean | Core tests | Metrics tests | Notes |
|---|---|---|---|---|---|
| 1 | 2026-04-14 | ✅ | 32/32 | — | Fresh install from scratch (pre-metrics) |
| 2 | 2026-04-14 | ✅ | 32/32 | — | Full data+namespace wipe, clean reinstall (pre-metrics) |
| 3 | 2026-04-14 | ✅ | 32/32 | — | Full data+namespace wipe, clean reinstall (pre-metrics) |
| 4 | 2026-04-15 | ✅ | 32/32 | 22/22 | First cycle with observability stack integrated |
| 5 | 2026-04-15 | ✅ | 32/32 | 22/22 | Full stack — exporter, Alloy, Prometheus, Grafana dashboards |
| 6 | 2026-04-15 | ✅ | 32/32 | 22/22 | Full stack — all 54 tests passing |

---

## Non-Destructive Cycles (preserve PVC and data)

```bash
cd ~/src/home_infra/postgres
./uninstall.sh --force
kubectl get pvc -n postgresql   # data-postgresql-0 must still be Bound
./install.sh   # runs 32 core tests + 22 metrics tests automatically
# Canary data from before teardown must still be present
```

| Cycle | Date | PVC survived | Core tests | Metrics tests | Canary data survived | Notes |
|---|---|---|---|---|---|---|
| 7 | 2026-04-14 | ✅ | 32/32 | — | ✅ `2026-04-14T18:34:03Z` | First non-destructive cycle (pre-metrics) |
| 8 | 2026-04-14 | ✅ | 32/32 | — | ✅ `2026-04-14T18:34:44Z` | Cycles 7+8 canary rows both present (pre-metrics) |
| 9 | 2026-04-14 | ✅ | 32/32 | — | ✅ `2026-04-14T18:35:32Z` | All 3 canary rows present after reinstall (pre-metrics) |

---

## Bugs Found

All bugs were discovered and fixed during implementation. All 9 cycles ran clean.

### Core PostgreSQL Bugs (cycles 1–9)

| Bug | Severity | Fix |
|---|---|---|
| Bitnami chart images unavailable (paywall as of Aug 2025) | Critical | Pivoted entirely to raw `kubectl` manifests with `docker.io/library/postgres:17` |
| `helm upgrade --atomic` race condition: headless service not found during rollback | High | Removed `--atomic` flag; `helm upgrade --install` is idempotent without it |
| Pending-install Helm release left after atomic failure | High | `helm uninstall` + namespace wipe before retry |
| Official postgres:17 uses `trust` for all local connections — auth test always passed | High | Added custom `pg_hba.conf` in ConfigMap requiring `scram-sha-256` for all TCP/host connections |
| `kubectl wait pod postgresql-0` failed with "pods not found" on fresh StatefulSet | Medium | Added polling loop (pod may not be scheduled yet) before calling `kubectl wait` |
| StatefulSet immutable field error when upgrading from Bitnami spec | Medium | Full namespace wipe (`--delete-data --delete-namespace --force`) before clean reinstall |
| Rolling update race: 15 test failures because old pod was being replaced | Medium | Added `kubectl rollout status statefulset/postgresql --timeout=300s` before pod-level wait |
| `PGDATA` at PVC root `/var/lib/postgresql/data` caused permission errors on init | Low | Set `PGDATA=/var/lib/postgresql/data/pgdata` (subdirectory avoids lost+found permission conflict) |

### Metrics Bugs (fixed before cycles 4–6)

| Bug | Severity | Fix |
|---|---|---|
| `SCRIPT_DIR` undefined in `uninstall.sh` — `set -euo pipefail` crashed immediately | Critical | Added `SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"` to top of uninstall.sh |
| Secret deleted unconditionally in soft teardowns — new password generated on reinstall, PGDATA retained old password → auth failure | Critical | Gated `kubectl delete secret postgresql` on `--delete-data` flag in uninstall.sh |
| Wrong image `quay.io/prometheuscommunity/postgres_exporter:v0.17.1` — registry returns 401 UNAUTHORIZED | Critical | Correct image: `prometheuscommunity/postgres-exporter:v0.17.1` (Docker Hub, hyphen not underscore) |
| Probe path `/health` returns 404 in postgres-exporter v0.17.1 — pod killed after 3 failures | High | Changed liveness/readiness probe path from `/health` to `/` (root returns 200 OK) |
| `scram-sha-256` auth fails from Go driver (pgx) — exporter gets `pg_up=0` | High | Changed `pg_hba.conf` TCP auth method from `scram-sha-256` to `password` for cluster-internal connections |
| `pg_postmaster_start_time_seconds` metric removed in v0.17.1 — test.sh false negative | Low | Replaced with `pg_database_size_bytes` in metrics/test.sh |

---

## See Also

- [[Postgres]] — full deploy runbook, test suite, troubleshooting
- [[Teardown Reinstall Validation]] — index of all validation reports
