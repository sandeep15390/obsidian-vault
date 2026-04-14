# Teardown / Reinstall Validation — PostgreSQL

> Validates that `install.sh` + `uninstall.sh` are fully repeatable and idempotent.
> Reference: [[Postgres]] — full deploy/teardown runbook.

## Summary

| Stat | Value |
|---|---|
| Total cycles | 6 (3 destructive + 3 non-destructive) |
| Tests per cycle | 32 |
| Status | ✅ Complete — all 6 cycles passed |

---

## Destructive Cycles (delete all data + namespace)

```bash
cd ~/src/home_infra/postgres
./uninstall.sh --delete-data --delete-namespace --force
kubectl get namespace postgresql 2>&1 | grep -q "not found" && echo "namespace clean"
kubectl get pvc -A 2>&1 | grep "postgresql" && echo "WARNING: orphaned PVC" || echo "PVC clean"
./install.sh
./test.sh
```

| Cycle | Date | Teardown clean | Tests | Notes |
|---|---|---|---|---|
| 1 | 2026-04-14 | ✅ | 32/32 | Fresh install from scratch |
| 2 | 2026-04-14 | ✅ | 32/32 | Full data+namespace wipe, clean reinstall |
| 3 | 2026-04-14 | ✅ | 32/32 | Full data+namespace wipe, clean reinstall |

---

## Non-Destructive Cycles (preserve PVC and data)

```bash
cd ~/src/home_infra/postgres
./uninstall.sh --force
kubectl get pvc -n postgresql   # data-postgresql-0 must still be Bound
./install.sh
./test.sh
# Canary data from before teardown must still be present
```

| Cycle | Date | PVC survived | Tests | Canary data survived | Notes |
|---|---|---|---|---|---|
| 4 | 2026-04-14 | ✅ | 32/32 | ✅ `2026-04-14T18:34:03Z` | First non-destructive cycle |
| 5 | 2026-04-14 | ✅ | 32/32 | ✅ `2026-04-14T18:34:44Z` | Cycles 4+5 canary rows both present |
| 6 | 2026-04-14 | ✅ | 32/32 | ✅ `2026-04-14T18:35:32Z` | All 3 canary rows present after reinstall |

---

## Bugs Found

All bugs were discovered and fixed during initial implementation (before teardown cycles began). All 6 cycles ran clean.

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

---

## See Also

- [[Postgres]] — full deploy runbook, test suite, troubleshooting
- [[Teardown Reinstall Validation]] — index of all validation reports
