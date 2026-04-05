# Homelab

## Overview
A home lab covering infrastructure, services, observability, networking, and automation.

## Services
- **Kubernetes (k3s)** — Single-node k3s cluster running on `melody-beast` (192.168.1.97) ✅

## Observability
- **Logging** — Loki + Grafana Alloy + Grafana, deployed to k3s ✅
- **Metrics** — Prometheus + Grafana *(planned)*
- **Alerting** — Alertmanager *(planned)*
- **Network Monitoring** — Uptime Kuma / ntopng *(planned)*
- **Tracing** — Tempo / Jaeger *(planned)*

## Pages
- [[Commands|Commands]] — Bootstrap, startup, and quick-reference commands
- [[Kubernetes|Kubernetes (k3s)]] — Install plan, network architecture, and ingress setup
- [[Logging|Logging (Loki + Alloy + Grafana)]] — Log aggregation architecture, deploy plan, and access

## Ideas & Notes

## Resources
- Grafana: `http://grafana.homelab.local` (or `kubectl port-forward svc/grafana 3000:80 -n logging`)
- Loki push endpoint: `http://192.168.1.97:31900/loki/api/v1/push`
- Infra repo: `~/src/home_infra`

## Status
- [x] Set up k3s single-node cluster
- [x] Configure kubeconfig
- [x] Install Helm
- [x] Deploy logging stack (Loki + Alloy + Grafana)
- [x] Automate install/uninstall/test with scripts
- [x] Confirm logs flowing from k8s pods into Loki
- [ ] Add `/etc/hosts` entry on LAN machines for `grafana.homelab.local`
- [ ] Install Alloy agent on remote LAN devices
- [ ] Set up metrics stack (Prometheus + Grafana)
- [ ] Set up alerting
- [ ] Set up network monitoring
- [ ] Build unified Grafana dashboards
