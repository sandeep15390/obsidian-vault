# Teardown / Reinstall Validation — LiteLLM

> Validates that `install.sh` + `uninstall.sh` are fully repeatable and idempotent.
> Reference: [[LiteLLM]] — full deploy/teardown runbook.

## Summary

| Stat | Value |
|---|---|
| Total cycles | 1 destructive (data + namespace) |
| Tests per cycle | 44 |
| Status | ✅ Complete — 44/44 passed |

---

## Destructive Cycles (delete namespace + PostgreSQL data)

```bash
cd ~/src/home_infra/litellm

# Export credentials before each teardown (or load from this doc)
export PROXY_MASTER_KEY="sk-homelab-e9cb0f4a243b3efddc2a5bac"   # ⚠️ CHANGE AFTER FIRST INSTALL
export DATABASE_URL="postgresql://postgres:0ZD0VE55HPyXD2yjQIRJxCt5QKh072x4@postgresql.postgresql.svc.cluster.local:5432/homelab"

./uninstall.sh --delete-namespace --delete-data --force
# Residue check must show all [OK]

./install.sh   # runs 44 tests automatically
```

| Cycle | Date | Teardown clean | Tests | Notes |
|---|---|---|---|---|
| 1 | 2026-04-16 | ✅ | 44/44 | First full cycle post-architecture review; all review bugs fixed |

---

## Bugs Found During Validation

| Bug | Severity | Fix |
|---|---|---|
| `prisma db execute` never outputs SELECT results — only "Script executed successfully." | Critical | C-3 guard rewrote: removed `--accept-data-loss`, push now fails explicitly on destructive drift |
| `--delete-data` quoting bug: double-quoted Postgres identifiers broke `sh -c "..."` expansion | Critical | Switched to `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` + base64-encoded SQL |
| `--delete-data` table list incomplete: 8 hardcoded tables vs 57 actual in schema v1.82.3 | High | `DROP SCHEMA public CASCADE` covers all 57 tables + future additions |
| `kubectl exec -- test -f <path>` hit Python shim `/usr/bin/test` instead of POSIX built-in | High | Fixed to `sh -c '[ -f ... ]'` which uses the shell built-in |
| `DATABASE_URL` missing from `litellm-api-keys` secret on reinstall (C-2) | High | `install.sh` step 2 now upserts `DATABASE_URL` into existing secret |
| `prisma db push --accept-data-loss` ran destructively on every reinstall (C-3) | High | Removed `--accept-data-loss`; prisma errors explicitly on drift |
| `diag.sh` section 7 used wrong ConfigMap key `proxy_config.yaml` → always `(not found)` | Medium | Fixed to `config.yaml` (W-9) |
| Alloy scrape block had no auth header → 401 if `/metrics` requires bearer token (W-2) | Medium | Added `remote.kubernetes.secret.litellm_keys` + `bearer_token` to scrape block |
| Grafana dashboard fetched from GitHub at deploy time → breaks on no internet (W-3) | Medium | Dashboard committed to `manifests/grafana-dashboard.json`; loaded from disk |
| Prometheus ingestion retry window 15s total — too short for first scrape cycle (W-8) | Medium | Increased to 6 × 15s = 90s |
| `generate-key.sh` used `localhost:32303` — only works on the K8s node (W-4) | Low | Switched to `kubectl port-forward` |
| `set -euo pipefail` in `diag.sh` aborted mid-run on crashed pod logs (W-10) | Low | Changed to `set -eo pipefail`; added `\|\| true` on logs/events |

---

## Credentials Reference

> ⚠️ **STARTER KEY — change after first install.** See [[LiteLLM#Rotating the Master Key]].

| Secret | Value |
|---|---|
| `PROXY_MASTER_KEY` | `sk-homelab-e9cb0f4a243b3efddc2a5bac` |
| `DATABASE_URL` | `postgresql://postgres:0ZD0VE55HPyXD2yjQIRJxCt5QKh072x4@postgresql.postgresql.svc.cluster.local:5432/homelab` |

---

## See Also

- [[LiteLLM]] — full deploy runbook, test suite, troubleshooting
- [[Teardown Reinstall Validation]] — index of all validation reports
