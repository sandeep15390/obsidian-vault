# Homelab

## Overview
A home lab covering infrastructure, services, observability, networking, and automation.

## Services
- **Kubernetes (k3s)** — Single-node k3s cluster running on `melody-beast` (10.0.0.7) ✅
- **Redis** — In-cluster key/value store, 2GB memory, AOF+RDB persistence, allkeys-lru eviction ✅ — see [[Redis]]
- **PostgreSQL** — In-cluster relational database, standalone primary, 10Gi persistence, `homelab` database ✅ — see [[Postgres]]
- **PostgreSQL Metrics** — postgres_exporter + Prometheus + dual Grafana dashboards (metrics + logs) ✅ — see [[Postgres Metrics]]

## Observability
- **Logging** — Loki + Grafana Alloy + Grafana, deployed to k3s ✅ **Complete** — 34 automated tests, teardown/reinstall validated (6 cycles)
- **Metrics** — Prometheus + Alloy + Grafana ✅ **Complete** — 37+ automated tests, teardown/reinstall validated — see [[Metrics]]
- **GPU Monitoring** — DCGM Exporter + Grafana ✅ **Complete** — 38 automated tests, dashboard uid `gpu-overview`, 2× RTX PRO 6000 Blackwell — see [[GPU Monitoring]]
- **Redis Metrics** — redis_exporter + Alloy scraping + Grafana dashboard uid `redis-overview` ✅ **Complete** — 22 automated tests, 3 teardown cycles — see [[Redis Metrics]]
- **Alerting** — Alertmanager *(planned)*
- **Network Monitoring** — Uptime Kuma / ntopng *(planned)*
- **Tracing** — Tempo / Jaeger *(planned)*

## Pages
- [[Commands|Commands]] — Bootstrap, startup, and quick-reference commands
- [[Kubernetes|Kubernetes (k3s)]] — Install plan, network architecture, and ingress setup
- [[Logging|Logging (Loki + Alloy + Grafana)]] — Log aggregation architecture, deploy plan, and access
- [[iptables Watchdog|iptables Watchdog]] — Runbook for k3s/Tailscale iptables conflict detection and recovery
- [[Teardown Reinstall Validation|Teardown / Reinstall Validation]] — Evidence report: 3 clean cycles, 34/34 tests, bugs fixed
- [[Metrics|Metrics (Prometheus + Alloy + Grafana)]] — Metrics collection architecture, deploy plan, and test suite
- [[GPU Monitoring|GPU Monitoring (DCGM + Grafana)]] — NVIDIA RTX PRO 6000 Blackwell GPU metrics, dashboard, and test suite
- [[Redis|Redis]] — In-cluster Redis: connection strings, auth, application integration, and deploy/teardown
- [[Redis Metrics|Redis Metrics]] — redis_exporter, Alloy scraping, Grafana dashboard `redis-overview`
- [[Postgres|PostgreSQL]] — In-cluster Postgres: connection strings, auth, application integration, and deploy/teardown
- [[LiteLLM]] — LLM API gateway; Ollama + cloud models; 9 local models; virtual key management

## AI / Automation

- **LiteLLM** — OpenAI-compatible LLM API gateway; routes to local Ollama (9 models) + cloud APIs ✅ — see [[LiteLLM]]
- **Dark Software Factory** — Autonomous agent pipeline running on this cluster's infrastructure. Lives in its own project: `Projects/Dark Software Factory/`

---

## Possible Automation Gaps

Things that are manual, unverified, or will require attention during a full rebuild:

### Node IP is DHCP
`melody-beast` gets `10.0.0.7` via DHCP on `wlp9s0`. Install scripts resolve the IP dynamically so they're fine, but the IP showing up in Grafana's Loki push URL and `/etc/hosts` entries on other machines will be stale if it changes.
- **Fix:** Set a static IP or DHCP reservation for `melody-beast` on the router.

### `tailscale serve --service` persistence across reboots ✅
The `svc:grafana` config persists — verified via `sudo tailscale serve get-config --all` which returns the full config after restarts. The config is also stored declaratively in `tailscale/services.json` and can be reapplied with `./apply.sh` (`tailscale serve set-config --all services.json`). After a full reimage, `./apply.sh` must be run once as part of bootstrap.

### Tailscale admin approval is a manual step ✅ (accepted)
After registering a named tailnet service (e.g. `svc:grafana`) for the first time on a new machine, an admin must approve it at `login.tailscale.com/admin/machines` before the service URL becomes accessible. This is a one-time step per service per machine and cannot be automated without a Tailscale API key.
- **Action:** Accept as a manual step. Document in rebuild checklist: after running `tailscale/apply.sh`, approve new services at `login.tailscale.com/admin/machines`.

### k3s networking and Tailscale iptables interference
After installing or restarting Tailscale on the same host as k3s, Tailscale can overwrite flannel's iptables rules, breaking pod-to-service-CIDR routing. Symptom: all Traefik ingresses return 404 and Traefik logs flood with `dial tcp 10.43.0.1:443: connect: no route to host`.
- **Fix:** `sudo systemctl restart k3s` restores the rules. See [[Kubernetes#All ingresses return 404 / Traefik loses API server connectivity]].
- **Automation gap:** No health check or watchdog exists to detect and auto-recover this. Consider a systemd service or cron that checks Traefik connectivity and restarts k3s if it fails.

### No bootstrap script *(future)*
The full rebuild sequence (k3s → kubeconfig → helm → logging stack → tailscale serve) is documented but requires running steps manually across multiple pages.
- **Fix:** A single `bootstrap.sh` in the root of `home_infra/` that chains all steps would make reimaging fully scriptable.

## Dev Projects

### Gamestore
A collection of browser-playable classic games hosted as a GitHub repo.

- **Repo:** `git@github.com:sandeep15390/gamestore.git` — local clone at `~/gamestore`
- **Structure:** `games/<game-name>/web/` per game, supports adding more games or platforms

**Games:**

| Game | Platform | How to play |
|------|----------|-------------|
| Snake | Web (HTML/JS) | Open `games/snake/web/index.html` in a browser |

**Adding a new game:** create `games/<name>/web/`, drop in files, update `README.md`.

## Ideas & Notes

## Resources
- Grafana (tailnet): `https://grafana.tailc98a25.ts.net` ← preferred
- Grafana (LAN): `http://grafana.homelab.local` (add `10.0.0.7 grafana.homelab.local` to `/etc/hosts`)
- Grafana (port-forward): `kubectl port-forward svc/grafana 3000:80 -n logging`
- Loki push (tailnet): `https://loki.tailc98a25.ts.net/loki/api/v1/push` ← preferred
- Loki push (LAN): `http://10.0.0.7:31900/loki/api/v1/push`
- Infra repo: `~/src/home_infra`

> **Note:** `10.0.0.7` is a DHCP address on `wlp9s0`. Scripts resolve the node IP dynamically at deploy time. If the IP changes, re-run `install.sh` and update these docs.

## Status
- [x] Set up k3s single-node cluster
- [x] Configure kubeconfig
- [x] Install Helm
- [x] Deploy logging stack (Loki + Alloy + Grafana)
- [x] Automate install/uninstall/test with scripts (34 tests)
- [x] Confirm logs flowing from k8s pods into Loki
- [x] Expose Grafana on tailnet (`https://grafana.tailc98a25.ts.net`)
- [x] Validate teardown/reinstall reproducibility (6 cycles, 0 failures)
- [x] Validate log persistence across non-destructive teardown/reinstall
- [ ] Install Alloy agent on remote LAN devices
- [x] Set up metrics stack (Prometheus + Grafana) — see [[Metrics]]
- [x] Build GPU metrics dashboard (DCGM Exporter + Grafana) — see [[GPU Monitoring]]
- [x] Deploy Redis (in-cluster, 2GB, AOF persistence) — see [[Redis]]
- [x] Redis metrics + Grafana dashboard — see [[Redis Metrics]]
- [ ] Set up alerting
- [ ] Set up network monitoring
- [ ] Build k8s cluster / node overview Grafana dashboards
- [ ] Create `bootstrap.sh` in `home_infra/` to chain full rebuild (k3s → kubeconfig → helm → logging → tailscale)
