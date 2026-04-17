# Teardown / Reinstall Validation - Redis Stack

**Last validated:** 2026-04-16  
**Total cycles:** 3 destructive  
**Tests per cycle:** 50 (28 Redis core + 22 Metrics)  
**Overall tests passed:** 150/150 (100%)

---

## Summary

After applying idempotency fixes to the teardown scripts, all 3 destructive cycles completed successfully with:
- Clean residue checks (no leftover resources)
- 50/50 tests passing each cycle
- `--force` flag required for all uninstall operations

---

## Bug Fixes Applied

| Issue | Severity | File | Fix |
|---|---|---|---|
| Residue check exited 0 on failure | CRITICAL | `metrics/uninstall.sh:136` | Now exits 1 when residue found |
| Grafana dashboard not verified in residue | CRITICAL | `metrics/uninstall.sh:122-131` | Added HTTP 404 check after deletion |
| `run()` used unsafe `eval "$*"` | CRITICAL | `metrics/install.sh:47` | Replaced with `"$@"` |
| Alloy namespace missing caused crash | CRITICAL | `metrics/uninstall.sh:60` | Added namespace existence guard |
| Grafana password retrieval aborts install | WARNING | `metrics/uninstall.sh:92` | Added `|| true` guard with warning |
| `--force` optional (interactive prompt) | WARNING | `uninstall.sh:44` | Now required; no interactive prompts |
| Grafana password retrieval aborts uninstall | WARNING | `metrics/install.sh:127` | Added `|| true` guard with warning |
| Diag.sh queried wrong Prometheus API | WARNING | `metrics/diag.sh:75` | Now queries Alloy `/api/v0/web/components` |
| Dashboard pinned to `latest` revision | INFO | `metrics/install.sh:14` | Now pinned to revision 6 |

---

## Cycle Results

### Cycle 1 — 2026-04-16

**Teardown:**
```
[OK]    redis-exporter Deployment removed
[OK]    redis-exporter Service removed
[OK]    Redis scrape block removed from Alloy ConfigMap
[OK]    Dashboard 'redis-overview' deleted
[OK]    Clean — no residue found
```

**Install:**
```
[OK]    helm upgrade --install redis bitnami/redis 25.3.11
[OK]    redis-exporter Deployment created
[OK]    Alloy ConfigMap patched with redis scrape block
[OK]    Dashboard 'redis-overview' uploaded (status=success)
```

**Tests:** 50/50 passed

### Cycle 2 — 2026-04-16

**Teardown:** Same as cycle 1 — clean residue check  
**Install:** Helm upgrade (not fresh install)  
**Tests:** 50/50 passed

### Cycle 3 — 2026-04-16

**Teardown:** Same as cycle 1 — clean residue check  
**Install:** Fresh install (namespace recreated)  
**Tests:** 50/50 passed

---

## Residue Check Details

The residue check now validates:

1. **Deployment not found:** `kubectl get deployment redis-exporter` returns 404
2. **Service not found:** `kubectl get service redis-exporter` returns 404
3. **Alloy ConfigMap:** No `# BEGIN redis` sentinel present
4. **Grafana dashboard:** HTTP 404 returned (not just "not found" message)
5. **Namespace:** When `--delete-namespace`, confirms namespace gone

---

## Idempotency Verification

| Action | Idempotent? | Notes |
|---|---|---|
| `install.sh` on already-installed stack | Yes | `helm upgrade --install` is idempotent |
| `install.sh` with `--dry-run` | Yes | Prints plan without applying |
| `uninstall.sh --force` | Yes | `--ignore-not-found` for all kubectl delete |
| `uninstall.sh --delete-data` | Yes | PVCs deleted, then namespace (if flag set) |

---

## Commands Used

```bash
# Full destructive cycle
cd ~/src/home_infra/redis
./uninstall.sh --delete-data --delete-namespace --force
./install.sh

# Non-destructive cycle (keeps PVC)
./uninstall.sh --force
./install.sh
```

---

## See Also

- [[Redis]] — Redis deployment documentation
- [[Redis Metrics]] — Metrics stack documentation
- [[Teardown Reinstall Validation]] — Index of all validation reports
