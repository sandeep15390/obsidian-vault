# Teardown / Reinstall Validation — Metrics Stack

> Performed: 2026-04-09
> Node: `melody-beast` (10.0.0.7)
> Scripts: `~/src/home_infra/metrics/`

## Summary

Two test suites were run — destructive teardown (deletes all metric data) and non-destructive teardown (preserves TSDB). Both passed 3 consecutive cycles with **zero failures and zero manual intervention**.

### Destructive teardown (`--delete-data --delete-namespace --force`)

| Cycle | Teardown | Residue check | Tests |
|---|---|---|---|
| 1 | Clean | namespace + PVCs gone | **36/36 passed** |
| 2 | Clean | namespace + PVCs gone | **36/36 passed** |
| 3 | Clean | namespace + PVCs gone | **36/36 passed** |

### Non-destructive teardown (`--force`, PVC preserved)

| Cycle | Teardown | PVC preserved | Tests | Persisted metrics |
|---|---|---|---|---|
| 1 | Clean | prometheus-server Bound (10Gi) | **36/36 passed** | 256 series at baseline timestamp |
| 2 | Clean | prometheus-server Bound (10Gi) | **36/36 passed** | 256 series at baseline timestamp |
| 3 | Clean | prometheus-server Bound (10Gi) | **36/36 passed** | 256 series at baseline timestamp |

Metric persistence was verified by querying `node_cpu_seconds_total` at a fixed baseline UNIX timestamp (recorded before the first teardown). All 256 series were present after every reinstall.

> Note: 3 Tailscale config tests are included but skipped when `sudo` is unavailable (run `sudo ./test.sh` for full 36-test suite including Tailscale).

---

## Bugs found and fixed

| Bug | Impact | Fix |
|---|---|---|
| `helm list \| grep "prometheus"` matched `prometheus-node-exporter` | Helm version check for `prometheus` chart reported wrong version | Used `helm list --filter "^release-name$"` for exact match |
| `grep -c '"value":\['` always returned 1 on single-line JSON | Pipeline tests counted 1 target even when 13 were present | Changed to `grep -o '"value":\[' \| wc -l` throughout |
| Alloy used `nodes/proxy` permission (not in generated ClusterRole) | kubelet/cAdvisor scraping via Alloy returned `up=0` | Removed kubelet/cAdvisor from Alloy config; Prometheus built-in `kubernetes_sd_configs` handles these natively |
| `set -o pipefail` + `\|\| echo 0` after `grep -o \| wc -l` printed "0\n0" | Count variable had `"0\n0"` causing `[[ $count -gt 0 ]]` syntax errors | Removed `\|\| echo 0` (wc -l always exits 0) |
| Alloy default 1-minute scrape interval caused test timing failures | On destructive cycle 3, `up{job="node-exporter"}` wasn't in Prometheus when tests ran | Added `scrape_interval = "30s"` to all Alloy scrape blocks; increased test retries to 6×15s |
| Prometheus `/-/ready` flaked on cycle 2 non-destructive reinstall | `unreachable` on first curl attempt while Prometheus was still initializing TSDB | Added 5-attempt retry with 5s sleep to `test_prometheus_ready_endpoint` |

---

## Conclusion

The metrics stack (`install.sh` / `uninstall.sh` / `test.sh`) is fully reproducible. A complete teardown leaves zero residue, and a fresh install produces a fully functional stack validated by 36 automated tests covering liveness, connectivity, metric pipeline correctness, Grafana integration, networking, and version pinning. Prometheus TSDB data persists across non-destructive teardown/reinstall cycles, confirmed by fixed-timestamp queries across 3 cycles.

## See Also

- [[Teardown Reinstall Validation]] — index of all component reports
- [[Metrics]] — stack overview and architecture
