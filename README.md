# Observability Platform

Prometheus + Loki + Grafana + OTel Collector + Blackbox Exporter, running on ECS Fargate (`ap-southeast-2`).

This repo is the single source of truth for all monitoring platform config. The applications being monitored live in their own repos — there is no coupling between them and this repo.

---

## Local development

### Prerequisites

- Docker Desktop running
- `.env` file created from `.env.example`

### Start the stack

```bash
cp .env.example .env   # first time only — fill in values
docker compose up -d
```

Grafana: http://localhost:3000 (login: `admin` / value from `.env`)

### Connect the AI Chat Middleware

In the AI Chat Middleware repo's `docker-compose.yml`, add the app service to the shared network:

```yaml
networks:
  - observability        # join the platform network by name

environment:
  LOKI_URI: http://loki:3100
  OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
  ANGULAR_OTLP_ENDPOINT: http://otel-collector:4318
  ENV_NAME: local
  SENTRY_DSN: <dsn>

external_networks:
  observability:
    external: true
```

---

## Repository layout

```
monitoring/                   Platform config (Chunk 3+)
  prometheus/
    prometheus.yml            Local Docker Compose scrape config
    prometheus-aws.yml        AWS EC2 service discovery scrape config
    rules/
      ai-chat.yml             Recording rules and alert rules
  loki/
    loki.yml
  otel-collector/
    otel-collector.yml
  blackbox/
    blackbox.yml
  grafana/
    provisioning/
      datasources/            Prometheus, Loki (with Derived Fields), Sentry
      dashboards/             Dashboard folder pointer
      alerting/               Unified alerting rules
    dashboards/
      ai-chat.json            AI Chat Middleware dashboard
      platform.json           Platform self-monitoring dashboard
  Dockerfile.prometheus
  Dockerfile.loki
  Dockerfile.grafana
  Dockerfile.otel-collector
  Dockerfile.blackbox

ecs/                          ECS Fargate task definitions (Chunk 4b+)
  task-def-prometheus.json
  task-def-loki.json
  task-def-grafana.json
  task-def-otel-collector.json
  task-def-blackbox.json

specs-and-plans/              Architecture spec, implementation plan, session summaries
docker-compose.yml            Platform services — local dev only
.env.example                  Required local secrets (copy to .env, never commit .env)
```

---

## AWS deployment

See [`specs-and-plans/implementation-plan.md`](specs-and-plans/implementation-plan.md) for the full delivery plan and [`specs-and-plans/spec.md`](specs-and-plans/spec.md) for the architecture spec.

Deployment target: ECS Fargate cluster `monitoring`, `ap-southeast-2`. Pipeline: Azure DevOps `DeployMonitoring` stage (Chunk 4b+).

---

## Variable strategy

| Variable | Local default | Injected in AWS by |
|---|---|---|
| `ENV_NAME` | `local` | EB env property |
| `LOKI_URI` | `http://loki:3100` | EB env property |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector:4317` | EB env property |
| `ANGULAR_OTLP_ENDPOINT` | `http://otel-collector:4318` | EB env property |
| `SENTRY_DSN` | hardcoded fallback in `environment.ts` | Secrets Manager → EB env property |
| `SENTRY_ORG_TOKEN` | `.env` | Secrets Manager → ECS task |
| `GRAFANA_ADMIN_PASSWORD` | `.env` | n/a (Cognito handles AWS auth) |

Full variable table: [`specs-and-plans/implementation-plan.md`](specs-and-plans/implementation-plan.md#variable-strategy-reference-for-all-chunks)
