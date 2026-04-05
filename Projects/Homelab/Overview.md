# Homelab

## Overview
A home lab covering infrastructure, services, observability, networking, and automation.

## Goals
- Centralize logs from all homelab services and infrastructure
- Collect and visualize system and application metrics
- Set up alerting for failures, anomalies, and threshold breaches
- Monitor network traffic, latency, and connectivity

## Services
- **Kubernetes** — Container orchestration cluster for running homelab workloads

## Key Features
- **Logging** — Centralized log aggregation and search (e.g. Loki, Elasticsearch)
- **Metrics** — Time-series metrics collection and dashboards (e.g. Prometheus, Grafana)
- **Alerting** — Alert rules and notification routing (e.g. Alertmanager, PagerDuty)
- **Network Monitoring** — Traffic analysis, uptime checks, and latency tracking (e.g. Smokeping, ntopng, Uptime Kuma)
- **Tracing** — Distributed tracing for services (e.g. Tempo, Jaeger)

## Pages
- [[Commands|Commands]] — Bootstrap, startup, and quick-reference commands
- [[Kubernetes|Kubernetes (k3s)]] — Install plan, network architecture, and ingress setup

## Ideas & Notes

## Resources

## Status
- [ ] Define scope — which hosts, services, and network devices to cover
- [ ] Choose and deploy metrics stack (Prometheus + Grafana or Victoria Metrics)
- [ ] Choose and deploy logging stack (Loki + Promtail or ELK)
- [ ] Set up network monitoring
- [ ] Configure alerting rules and notification channels
- [ ] Build unified Grafana dashboards
