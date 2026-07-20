# Observability Platform — Claude context

## Documentation rules

These rules apply to every session. Follow them without being asked.

**`specs-and-plans/spec.md` and `specs-and-plans/implementation-plan.md` are the source of truth.**
- They must stay internally consistent with each other at all times.
- If a decision in one contradicts the other, resolve the conflict and update both before proceeding.
- Never let a session end with a known inconsistency between them.

**After completing a chunk or session:**
- Mark the chunk complete in `implementation-plan.md` (update the delivery sequence, note any deviations).
- Record the session outcome in `specs-and-plans/sessions/session-N-summary.md`.

**After any codebase change:**
- Update `CLAUDE.md` if the change introduces a new design decision, code pattern, naming convention, or constraint that future sessions need to follow.
- Update `README.md` if the change affects how a developer onboards, runs the project locally, or contributes changes (new env vars, new services, new setup steps, changed commands).

**CLAUDE.md** is for Claude and contributors who already know what the project is — keep it focused on decisions, patterns, and gotchas. Not a tutorial.

**README.md** is for a developer who has never seen this repo — keep it complete enough to go from zero to a running local stack without asking anyone anything.

---

This repo contains the monitoring platform config for Workbench AI Chat Middleware.
The application being monitored lives in a separate repo (`C:\dev\ai-chat-spike\repo`).

## What this repo is

- `monitoring/` — all platform config files (Prometheus, Loki, Grafana, OTel Collector, Blackbox)
- `ecs/` — ECS Fargate task definitions with `#{TOKEN}#` placeholders for pipeline substitution
- `specs-and-plans/` — architecture spec, implementation plan, session summaries
- `docker-compose.yml` — local dev platform stack (no app services — decoupled by design)

## Active work

Multi-session implementation in progress. Current state: **Chunk 4 complete**.

- Full plan: `specs-and-plans/implementation-plan.md`
- Architecture spec: `specs-and-plans/spec.md`
- Session history: `specs-and-plans/sessions/session-N-summary.md`

Always read the latest session summary and `implementation-plan.md` before starting a new chunk.

## Key design decisions

- `env` label is the spine of the whole system — must be identical across Prometheus scrape labels, Loki push labels, and OTel resource attributes
- `monitoring/` is the single source of truth — config is either baked into Docker images (PoC) or EFS-mounted (production-grade, Chunk 6)
- Grafana is provisioned as code — dashboards are exported to JSON and committed; never rely on Grafana state surviving a container restart until EFS is attached (Chunk 6)
- No coupling to the application repo — the platform network is named `observability`; app services join it externally

## Variable strategy

All environment-specific values are injected — nothing hardcoded in committed config except local Docker Compose defaults. See `implementation-plan.md` for the full variable table.

Critical: Angular browser metrics are **never** sent directly to the OTel Collector. The Angular OTel SDK posts to `/otlp/v1/metrics` on the .NET app (same-origin). The .NET app proxies to `http://otel-collector:4318` internally. This keeps the collector off the public internet and works for on-premises deployments. `ANGULAR_OTLP_ENDPOINT` env var is removed — `otlpEndpoint` is always the hardcoded relative path `"/otlp"`. The proxy hardcodes `http://otel-collector:4318` as the destination — this hostname is stable in both Docker Compose and ECS Service Connect, so no env var is needed.

## Docker network

The platform compose defines a network named `observability`. App services join it by declaring it as an external network. This is how Loki, OTel Collector, and Prometheus reach the app without any compose file coupling.

## AWS target

ECS Fargate cluster `monitoring`, `ap-southeast-2`. Five services: Prometheus, Loki, Grafana, OTel Collector, Blackbox Exporter. Internal service discovery via ECS Service Connect (stable DNS names matching Docker Compose service names).

## OTel Collector config gotchas (Chunk 3 findings)

- **`loki` exporter removed** from `otel-collector-contrib` v0.80+. Use `otlphttp/loki` with endpoint `http://loki:3100/otlp` — Loki 2.9+ accepts OTLP natively.
- **Internal metrics port**: `health_check` extension is for liveness only (JSON). Internal Prometheus metrics require `service.telemetry.metrics.readers` with a Prometheus pull exporter reader. Port 8888 is in use on this dev machine; local config uses 18888. `prometheus-aws.yml` correctly targets 8888 (free in ECS).
- **Grafana alert provisioning**: every alert rule query must include `relativeTimeRange: { from: N, to: 0 }` or Grafana refuses to start.
- **Sentry datasource alerting**: `grafana-sentry-datasource` cannot be used as a Grafana Unified Alerting data source. Sentry JS error alerts require a separate integration (Sentry webhook → Grafana OnCall, or polling via custom metric).

## Chunk 4b / 5 starting point

`specs-and-plans/aws-infra-spec.md` is written and ready for tech lead review. Before Chunk 4b:
- Open questions in the spec (EB application name, app port, Cognito User Pool reuse, VPC ID) must be resolved — see the Open Questions table in `aws-infra-spec.md`
- Once questions are resolved and spec is approved, Chunk 4b can run immediately in parallel with Infra team provisioning
- Chunk 4b authors `ecs/task-def-*.json` files and the `DeployMonitoring` Azure DevOps stage with `#{TOKEN}#` placeholders

## File naming conventions

- Prometheus config: `prometheus.yml` (local), `prometheus-aws.yml` (AWS) — do NOT mix them up; `Dockerfile.prometheus` copies `prometheus-aws.yml`
- Grafana datasource provisioning: `.yaml` extension (Grafana convention)
- Grafana dashboard exports: use "Export for sharing externally" to strip datasource UIDs
- ECS task defs: `task-def-<service>.json` with `#{TOKEN}#` placeholders for Infra team-provided values
