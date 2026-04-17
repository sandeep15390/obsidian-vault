# Dark Software Factory

## Overview

A fully automated, human-minimal software development pipeline — a "dark factory" (lights-out manufacturing, no humans on the floor) applied to software. Accepts high-level task descriptions, decomposes them, routes subtasks to the best-fit agent-model combination via the Agent Communication Protocol (ACP), and produces pull requests with minimal human intervention.

**Designed to run on any Kubernetes cluster.** Infrastructure dependencies (Redis, PostgreSQL, LiteLLM) are expressed as cluster-internal service endpoints — configurable at install time. The homelab k3s cluster is the initial deployment target, but nothing in the factory is homelab-specific.

## Goals

- Orchestrate multiple AI coding agents (Claude Code, Codex, OpenCode, OpenHands, Qwen Code) through a standard ACP protocol layer
- Support both local models (Ollama) and cloud models (Anthropic, OpenAI) through a unified LiteLLM gateway
- Give every (agent, model) combination its own identity, workspace, cost attribution, and evaluation record
- Accumulate per-worker performance data to continuously improve routing decisions
- Produce a fully scriptable install/uninstall cycle that passes the same test suite on any Kubernetes cluster

## Pages

- [[Architecture]] — system design, component roles, data flow, ACP integration layer
- [[Worker Registry]] — all 13 workers: agent CLI × model pairings, capabilities, LiteLLM keys
- [[Implementation Plan]] — 9-phase build plan with blocking questions, per-phase exit criteria, and deploy commands

## Status

- [ ] Phase 0: Blocking questions resolved — ACP SDK, OpenCode API, OpenHands runtime, Claude Code flags, Qwen Code flags pinned
- [ ] Phase 1: Namespace, DB schema, 13 LiteLLM virtual keys created
- [ ] Phase 2: ACP Broker deployed (`GET /agents` returns 5 adapters)
- [ ] Phase 3: OpenCode adapter deployed and smoke-tested
- [ ] Phase 4: OpenHands adapter deployed and smoke-tested
- [ ] Phase 5: Codex adapter deployed and smoke-tested
- [ ] Phase 6: Claude Code adapter deployed and smoke-tested
- [ ] Phase 6.5: Qwen Code adapter deployed and smoke-tested
- [ ] Phase 7: Orchestrator — task intake, decomposition, routing, aggregation
- [ ] Phase 8: Observability — Prometheus metrics + Grafana dashboard `dark-factory-overview`
- [ ] Phase 9: 63/63 tests passing; 3 destructive teardown/reinstall cycles validated

## Kubernetes Requirements

The factory requires the following services to be reachable within the cluster. All are configurable as env vars at install time — no hardcoded homelab addresses.

| Service | Default (homelab) | Config var |
|---|---|---|
| Redis | `redis.redis.svc.cluster.local:6379` | `REDIS_URL` |
| PostgreSQL | `postgresql.postgresql.svc.cluster.local:5432` | `DATABASE_URL` |
| LiteLLM proxy | `litellm.litellm.svc.cluster.local:4000` | `LITELLM_BASE_URL` |
| Prometheus (optional) | `prometheus.metrics.svc.cluster.local:9090` | `PROMETHEUS_URL` |
| Grafana (optional) | `grafana.logging.svc.cluster.local:3000` | `GRAFANA_URL` |

Any Kubernetes cluster with these services — or equivalents (e.g. a managed Redis, external Postgres, self-hosted LiteLLM) — can run the factory.

## Key Design Decisions

**ACP (Agent Communication Protocol)** — BeeAI's open protocol for agent-to-agent communication. Each adapter is both an ACP server (the broker calls it) and a client of its wrapped CLI. SSE streaming, run state management, and session continuity come for free.

**Worker identity** — a Worker = one CLI harness + one model. 13 workers across 5 CLIs. Each has its own LiteLLM virtual key, git worktree root, branch prefix, and evaluation record. Routing targets `worker_id`, not just agent type.

**Local-first, cloud-escalate** — simple tasks route to local Ollama models (zero cloud cost); failures escalate to cloud. The routing table is a ConfigMap — no redeployment to adjust.

**Evaluation-driven routing** — every run writes to `worker_evaluations`. The SQL query in [[Worker Registry]] produces a ranked performance table. Routing table updates are manual and data-driven, not live ML.

## Ideas & Notes

- Temporal.io as the workflow engine (replaces ad-hoc orchestrator loop with durable execution + retries + visual history)
- GitHub Actions integration — per-PR automated review as a factory task
- Tool approval callbacks — ACP `await/interrupt` for human-in-the-loop safety gate before destructive CLI actions
- MCP server federation — expose codebase, issue tracker, docs as MCP servers consumed uniformly by all adapters
- LLM-judge scorer — auto-fill `reviewer_score` in `worker_evaluations` post-run
- Multi-cluster support — route tasks to workers on different clusters (local GPU cluster vs cloud K8s)
