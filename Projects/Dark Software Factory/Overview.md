# Dark Software Factory

## Overview

A fully automated, human-minimal software development pipeline — a "dark factory" (lights-out manufacturing, no humans on the floor) applied to software. Accepts high-level task descriptions, decomposes them, routes subtasks to the best-fit agent-model combination via the Agent Communication Protocol (ACP), and produces pull requests with minimal human intervention.

**Designed to run on any Kubernetes cluster.** Each adapter talks directly to either Ollama (local models) or a cloud API (Anthropic, OpenAI) — no intermediate gateway. The homelab k3s cluster is the initial deployment target, but nothing in the factory is homelab-specific.

## Goals

- Orchestrate multiple AI coding agents (Claude Code, Codex, OpenCode, OpenHands, Qwen Code) through a standard ACP protocol layer
- Support both local models (via Ollama) and cloud models (Anthropic, OpenAI) with direct API connections per adapter
- Give every (agent, model) combination its own identity, git workspace, cost attribution, and evaluation record
- Accumulate per-worker performance data to continuously improve routing decisions
- Produce a fully scriptable install/uninstall cycle that passes the same test suite on any Kubernetes cluster

## Pages

- [[Architecture]] — system design, component roles, data flow, ACP integration layer
- [[Worker Registry]] — all 13 workers: agent CLI × model pairings, capabilities, direct model endpoints
- [[Implementation Plan]] — 9-phase build plan with blocking questions, per-phase exit criteria, and deploy commands
- [[Running the Factory]] — how to install, submit tasks, check status, and tear down
- [[Requirements - Agent Loop Detection]] — loop detection and escalation requirement, with observed incident from 2026-04-17
- [[OpenCode Adapter]] — how the opencode adapter works: request flow, Ollama config, API, known limitations

## Status

> Last updated: 2026-04-18

- [x] Phase 0: Blocking questions resolved — versions pinned in `VERSIONS.env`; direct Ollama path validated
- [x] Phase 1: Namespace, DB schema, secrets — schema.sql complete with 4 tables + indexes; install.sh creates secrets
- [x] Phase 2: ACP Broker — worker hot-reload, per-run model injection, SSE proxying, Prometheus metrics
- [x] Phase 3: OpenCode adapter — HTTP proxy to OpenCode server, polling, SSE relay
- [x] Phase 4: OpenHands adapter — HTTP proxy to OpenHands server, polling, SSE relay
- [x] Phase 5: Codex adapter — subprocess, streaming output
- [x] Phase 6: Claude Code adapter — subprocess, stream-json parsing, validated with direct Ollama
- [x] Phase 6.5: Qwen Code adapter — subprocess, streaming output
- [x] Phase 7: Orchestrator — task intake, SSE decomposition parsing, routing, aggregation, HMAC webhook validation, duration metrics fixed
- [x] Phase 8: Observability — Prometheus metrics on broker + orchestrator; Grafana `dark-factory-overview` dashboard including `dark_factory_worker_success_rate`
- [ ] Phase 9: 59/59 tests passing; 3 destructive teardown/reinstall cycles validated

## Kubernetes Requirements

Minimal hard dependencies — only Redis and PostgreSQL required. Ollama runs on the host or any reachable endpoint; cloud API keys are optional (only needed for cloud workers).

| Service | Default (homelab) | Config var | Required? |
|---|---|---|---|
| Redis | `redis.redis.svc.cluster.local:6379` | `REDIS_URL` | Yes |
| PostgreSQL | `postgresql.postgresql.svc.cluster.local:5432` | `DATABASE_URL` | Yes |
| Ollama | `http://10.0.0.7:11434` | `OLLAMA_BASE_URL` | For local workers |
| Anthropic API | `https://api.anthropic.com` | `ANTHROPIC_API_KEY` | For cc-sonnet only |
| OpenAI API | `https://api.openai.com` | `OPENAI_API_KEY` | For cx-gpt4o only |
| Prometheus (optional) | `prometheus.metrics.svc.cluster.local:9090` | `PROMETHEUS_URL` | No |
| Grafana (optional) | `grafana.logging.svc.cluster.local:3000` | `GRAFANA_URL` | No |

## Key Design Decisions

**No LiteLLM gateway.** Each adapter talks directly to its model backend — Ollama for local models, Anthropic/OpenAI APIs for cloud models. This was validated in a live test: Claude Code + direct Ollama executes tool use correctly; routing through LiteLLM's Anthropic→Ollama translation broke structured tool_use blocks.

**ACP (Agent Communication Protocol)** — BeeAI's open protocol for agent-to-agent communication. SSE streaming, run state management, and session continuity built in.

**Worker identity** — a Worker = one CLI harness + one model. 13 workers across 5 CLIs. Each has its own model endpoint config, git worktree root, and evaluation record. Routing targets `worker_id`.

**Local-first, cloud-escalate** — simple tasks route to Ollama-backed workers (zero cloud cost); failures escalate to cloud. Routing table is a ConfigMap — no redeployment to adjust.

**Evaluation-driven routing** — every run writes to `worker_evaluations`. SQL query in [[Worker Registry]] produces ranked performance table. Routing updates are manual and data-driven.

## Ideas & Notes

- Temporal.io as the workflow engine — durable execution, retries, visual history
- GitHub Actions integration — per-PR automated review as a factory task
- Tool approval callbacks — ACP `await/interrupt` for human-in-the-loop safety gate
- MCP server federation — expose codebase, issue tracker, docs as MCP servers to all adapters
- LLM-judge scorer — auto-fill `reviewer_score` post-run
- Multi-cluster support — route tasks to workers on different clusters
