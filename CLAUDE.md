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

Multi-session implementation in progress. Current state: **Chunk 2 complete**.

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

Critical: `ANGULAR_OTLP_ENDPOINT` (HTTP/4318) is separate from `OTEL_EXPORTER_OTLP_ENDPOINT` (gRPC/4317) — browsers cannot make gRPC calls.

## Docker network

The platform compose defines a network named `observability`. App services join it by declaring it as an external network. This is how Loki, OTel Collector, and Prometheus reach the app without any compose file coupling.

## AWS target

ECS Fargate cluster `monitoring`, `ap-southeast-2`. Five services: Prometheus, Loki, Grafana, OTel Collector, Blackbox Exporter. Internal service discovery via ECS Service Connect (stable DNS names matching Docker Compose service names).

## Chunk 3 starting point

Next session builds all of `monitoring/` and the Docker Compose services. Before starting:
- Docker Desktop must be running
- `.env` must exist (copy `.env.example`, fill in `GRAFANA_ADMIN_PASSWORD` and `SENTRY_ORG_TOKEN`)
- The AI Chat Middleware must be running and reachable (for scrape target smoke tests)
- App's compose file needs the external network block and env vars listed in `README.md`

## File naming conventions

- Prometheus config: `prometheus.yml` (local), `prometheus-aws.yml` (AWS) — do NOT mix them up; `Dockerfile.prometheus` copies `prometheus-aws.yml`
- Grafana datasource provisioning: `.yaml` extension (Grafana convention)
- Grafana dashboard exports: use "Export for sharing externally" to strip datasource UIDs
- ECS task defs: `task-def-<service>.json` with `#{TOKEN}#` placeholders for contractor-provided values
