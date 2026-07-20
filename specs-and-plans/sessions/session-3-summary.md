# Session 3 Summary — Local Platform Stack

**Branch:** `main` (observability repo)
**Date:** 2026-07-20

---

## ✅ What got built

### New files

| File | Purpose |
|---|---|
| `monitoring/prometheus/prometheus.yml` | Local Docker Compose scrape config — static_configs for ai-chat, blackbox, platform-self |
| `monitoring/prometheus/prometheus-aws.yml` | AWS scrape config — ec2_sd_configs for ai-chat with `#{TOKEN}#` placeholders; static blackbox + platform-self jobs |
| `monitoring/prometheus/rules/ai-chat.yml` | Recording rules: `job:http_requests_total:rate5m`, `job:http_request_duration_seconds:p95`, `job:http_errors:rate5m` |
| `monitoring/loki/loki.yml` | Full PoC Loki config (auth, server, common, schema_config, limits_config — not a placeholder) |
| `monitoring/blackbox/blackbox.yml` | Probe modules: `http_2xx` (GET) and `http_2xx_post` (POST for SignalR negotiate) |
| `monitoring/otel-collector/otel-collector.yml` | OTLP receivers (4317/4318), all processors, prometheusremotewrite + otlphttp/loki exporters, CORS config |
| `monitoring/grafana/provisioning/datasources/prometheus.yaml` | Prometheus datasource (uid: prometheus, isDefault: true) |
| `monitoring/grafana/provisioning/datasources/loki.yaml` | Loki datasource with Derived Fields for Loki→Tempo click-through |
| `monitoring/grafana/provisioning/datasources/sentry.yaml` | Sentry datasource (grafana-sentry-datasource plugin, SENTRY_ORG_TOKEN injected) |
| `monitoring/grafana/provisioning/dashboards/dashboards.yaml` | Dashboard provider pointing at /var/lib/grafana/dashboards |
| `monitoring/grafana/provisioning/alerting/rules.yaml` | 6 alert rules: AiChatHighErrorRate, AiChatHighLatency, AiChatEndpointDown, SignalRConnectionsDrop, TenantErrorSpike (Loki), CollectorDroppingData |
| `monitoring/Dockerfile.prometheus` | Copies prometheus-aws.yml (not prometheus.yml); adds --web.enable-remote-write-receiver |
| `monitoring/Dockerfile.loki` | Copies loki.yml |
| `monitoring/Dockerfile.grafana` | Installs grafana-sentry-datasource plugin; copies provisioning + dashboards |
| `monitoring/Dockerfile.blackbox` | Copies blackbox.yml |
| `monitoring/Dockerfile.otel-collector` | Copies otel-collector.yml |
| `monitoring/grafana/dashboards/ai-chat.json` | AI Chat service dashboard — all standard panels + SignalR + chat throughput |
| `monitoring/grafana/dashboards/platform.json` | Platform meta-monitoring dashboard — all 12 panels from spec §1.7 |

### Modified files

| File | What changed |
|---|---|
| `docker-compose.yml` | Added all 5 service blocks; `loki-data` and `grafana-data` volumes; `observability` network |

---

## 🧪 Test results

All smoke tests passed:

| Check | Result |
|---|---|
| `docker compose up -d` — all 5 services start | ✅ All UP |
| `GET http://localhost:9090/-/healthy` | Prometheus Server is Healthy ✅ |
| `GET http://localhost:3100/ready` | ready ✅ |
| `GET http://localhost:3000/api/health` | `{"database":"ok","version":"13.1.0"}` ✅ |
| `GET http://localhost:9115/health` | Blackbox Exporter up ✅ |
| `GET http://localhost:18888/metrics` | OTel Collector internal metrics in Prometheus format ✅ |
| `platform-self` scrape targets in Prometheus | prometheus ✅, loki ✅, otel-collector ✅, blackbox-exporter ✅ |
| Recording rules loaded in Prometheus | `job:http_requests_total:rate5m`, `job:http_request_duration_seconds:p95`, `job:http_errors:rate5m` ✅ |
| Both dashboards loaded in Grafana | `ai-chat` (uid: ai-chat) ✅, `platform` (uid: platform) ✅ |
| All 6 alert rules loaded in Grafana | AiChatHighErrorRate, AiChatHighLatency, AiChatEndpointDown, SignalRConnectionsDrop, TenantErrorSpike, CollectorDroppingData ✅ |
| Datasources provisioned | Prometheus ✅, Loki ✅, Sentry ✅ |
| OTel Collector internal metrics flowing into Prometheus | `otelcol_exporter_queue_size` series present in Prometheus ✅ |
| `ai-chat` Prometheus target | Down (expected — AI Chat Middleware not running) ✅ |
| Blackbox probes for ai-chat | Failing (expected — AI Chat Middleware not running) ✅ |

The full local observability loop (platform side) is working. Real end-to-end data requires the AI Chat Middleware to join the `observability` network and start running.

---

## ⚠️ Gotchas / decisions that differ from the plan

### `loki` exporter removed from otel-collector-contrib (blocker fixed)
The spec and implementation plan referenced the `loki` exporter for the OTel Collector. This exporter was removed from `otel/opentelemetry-collector-contrib` in v0.80+. Loki 2.9+ natively accepts OTLP logs at `/otlp/v1/logs`. Fixed by using `otlphttp/loki` exporter targeting `http://loki:3100/otlp`. **The `loki` exporter in `otel-collector.yml` section of the spec is now stale — update it for future reference.**

### OTel Collector internal metrics port (local deviation)
Port 8888 is in use by another process on this development machine. The local `prometheus.yml` and Docker Compose use port 18888 instead. `prometheus-aws.yml` keeps port 8888 (correct for ECS where the port is free). The `Dockerfile.otel-collector` does not expose any port — the ECS task definition controls port mappings there.

**Impact on Chunk 5:** The AWS `prometheus-aws.yml` target `otel-collector:8888` is correct and needs no change. The local port discrepancy is documented in `prometheus.yml` with a comment.

### OTel Collector telemetry metrics configuration (API change)
The `health_check` extension in recent otel-collector-contrib versions exposes only a JSON liveness page, not Prometheus metrics. Internal Prometheus metrics require the `service.telemetry.metrics.readers` block using the Prometheus pull exporter reader (introduced in v0.89+). The `health_check` extension uses port 13133 (default) for liveness; `service.telemetry.metrics` uses port 18888 for Prometheus scraping.

### Grafana alert provisioning requires `relativeTimeRange`
The Grafana Unified Alerting provisioning schema requires a `relativeTimeRange` block on every query (`from` and `to` in seconds). Without it, Grafana refuses to start with `[alerting.alert-rule.invalidRelativeTime] Invalid alert rule query A`. All rules now include `relativeTimeRange: { from: 600, to: 0 }`.

### Sentry JS error alert rule omitted
The `AiChatJSErrorSpike` alert rule was dropped. The `grafana-sentry-datasource` plugin cannot be used as an alerting data source in Grafana Unified Alerting — Sentry is a third-party plugin and doesn't support the alerting query contract. Sentry errors are visible in the dashboard panel (for human review), but automated alerting for JS errors would require using the Sentry webhook → Grafana OnCall integration, or polling the Sentry API via a separate metric. This is a known limitation noted in the spec's "Risks / gotchas" section — not a session failure.

### Dashboard JSONs are starter templates
The spec calls for dashboards to be "built interactively in Grafana and exported". The JSON files committed here are well-formed starter templates with all the panels, queries, and template variables defined. They load correctly and show real data once the AI Chat Middleware is running. Ideally a developer should open Grafana, tweak panel layouts for usability, and re-export with "Export for sharing externally" before considering them production-ready. The `__requires` array and the `__inputs` array are already stripped (no datasource UID coupling), so they are portable.

### Sentry org token scope
A Sentry org token was generated with `org:ci` scope (the only available scope). The `grafana-sentry-datasource` plugin likely requires `org:read` or `project:read` to query issue counts. If the "JS Error Count" panel shows an auth error in Grafana, a new token with `org:read` scope will be needed.

---

## 📋 Jira test plan / verification steps

**Prerequisites:**
1. Docker Desktop running
2. `.env` exists (copy `.env.example`, set `GRAFANA_ADMIN_PASSWORD` and optionally `SENTRY_ORG_TOKEN`)
3. AI Chat Middleware running and joined to the `observability` Docker network

**Tests:**

1. **Stack starts cleanly**: `docker compose up -d` → all 5 services `Up` with no restarts. Check with `docker compose ps`.

2. **Platform-self scrape targets all UP**: Open `http://localhost:9090/targets` — `platform-self` job shows `prometheus`, `loki`, `otel-collector`, `blackbox-exporter` all green.

3. **Recording rules appear as series**: In Prometheus expression browser, query `job:http_requests_total:rate5m` — series appears in the autocomplete list (even if value is 0 without the app running, the series exists after the first rule evaluation at `t=60s`). Alternatively, `http://localhost:9090/api/v1/rules` returns the 3 recording rules.

4. **Grafana loads both dashboards**: Open `http://localhost:3000` → login with admin / `GRAFANA_ADMIN_PASSWORD` → Dashboards → confirm "AI Chat Middleware" and "Platform Meta-Monitoring" are present.

5. **Platform dashboard shows real data**: Open "Platform Meta-Monitoring" → "Collector: Export Queue Depth" panel should show non-zero series from `otelcol_exporter_queue_size`. "Platform Service Uptime" stat panels should show UP for prometheus, loki, otel-collector, blackbox-exporter.

6. **Alert rules active in Grafana**: Alerting → Alert rules → confirm 6 rules exist across folders "AI Chat" and "Platform".

7. **AI Chat scrape works (requires app running)**: Start AI Chat Middleware with the network/env vars from `README.md`. In Prometheus targets, `ai-chat` job shows green. Grafana "AI Chat Middleware" dashboard shows real request rate data.

8. **Loki receives logs (requires app running)**: In Grafana → Explore → Loki → query `{service="ai-chat"}` → log lines appear with queryable JSON fields (`TenantId`, `CorrelationId`).

9. **OTel Collector receives OTLP (requires app running)**: Open `http://localhost:18888/metrics` → `otelcol_receiver_accepted_metric_points` counter should be non-zero after the app starts sending metrics.

10. **Blackbox probes (requires app running)**: Grafana "AI Chat Middleware" → "Endpoint Uptime" panel shows `probe_success=1` for both `/health` and `/hubs/aichat/negotiate`.

11. **CORS unblocked for Angular**: In the browser dev tools → Network tab → filter for `v1/metrics` or `v1/traces` — requests to `http://localhost:4318/v1/...` should now succeed (previously failed with CORS error). Confirm no CORS error in browser console.

12. **Sentry datasource** (optional): If `SENTRY_ORG_TOKEN` is set, open Grafana → Connections → Data Sources → Sentry → Test. If auth fails, the token needs `org:read` or `project:read` scope (current token has `org:ci` scope only).

---

## 🚀 Next session (Session 4) — AWS Infrastructure Requirements Spec

**What to build:** Chunk 4 — `specs-and-plans/aws-infra-spec.md`

This is a requirements document for tech lead sign-off and Infra team execution. No code — pure specification writing.

**All open questions resolved:**
- RDS cluster: Infra team's decision (new vs existing) — the spec states the requirement and leaves the choice open
- EB application name, app port, env tag names: included as required Infra team outputs in the spec

**Note for Chunk 5 pre-flight:**
- `prometheus-aws.yml` has `#{AI_CHAT_APP_PORT}#`, `#{EB_APPLICATION_NAME}#`, and per-environment host placeholders — these will be filled once Infra team provides the values
- The local port 18888 for OTel Collector internal metrics is a local-only deviation; ECS will use port 8888 as specified in `prometheus-aws.yml`
