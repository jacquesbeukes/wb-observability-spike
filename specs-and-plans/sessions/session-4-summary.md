# Session 4 Summary

## Work completed

### OTel Collector — traces pipeline
- Added `debug/traces` exporter (verbosity: basic) and `traces` pipeline to `otel-collector.yml`
- Root cause of `/v1/traces` 404s: the OTLP HTTP receiver only registers routes for signals that have a configured pipeline. No `traces` pipeline = no `/v1/traces` route.
- `docker compose up -d` does NOT re-read config if the image tag hasn't changed — must use `--force-recreate` when only the mounted config file changes.

### Blackbox probe fix
- Removed the `/hubs/aichat/negotiate` probe from both `prometheus.yml` and `prometheus-aws.yml`
- The URL was wrong (missing hyphen: `aichat` vs `ai-chat`) and the `http_2xx` module uses GET — SignalR negotiate requires POST, so it would always fail regardless
- `/health` is the correct uptime signal; negotiate is not a suitable Blackbox target

### Angular OTel meter provider bug fix (`observability.ts`)
- `(meterProvider as unknown as { register?(): void }).register?.()` was a no-op — `MeterProvider` from `@opentelemetry/sdk-metrics` has no `register()` method
- Fixed by replacing with `metrics.setGlobalMeterProvider(meterProvider)` (importing `metrics` from `@opentelemetry/api`)
- Without this fix the global meter provider stays as the no-op default, so any future `getMeter()` call silently records nothing

### Angular metrics — instrumentation added
- Created `chat-metrics.service.ts` — 8 instruments: `angular_conversation_started_total`, `angular_message_sent_total`, `angular_message_cancelled_total`, `angular_response_failed_total`, `angular_rating_submitted_total`, `angular_signalr_reconnect_total`, `angular_time_to_first_chunk_ms` (histogram), `angular_response_duration_ms` (histogram)
- Created `admin-metrics.service.ts` — 1 instrument: `angular_admin_page_load_ms` (histogram, tagged by `page: summary|list|detail`)
- Injected `ChatMetricsService` into `chat.service.ts` — instruments fire at `startConversation`, `sendMessage`, `cancelResponse`, `rateConversation`, and in hub event handlers for `ResponseChunk` (TTFC), `ResponseCompleted` (duration), `ResponseFailed`, and `onreconnecting`
- Injected `AdminMetricsService` into `admin.service.ts` — records duration on `getSummary`, `getConversations`, `getConversationDetail`
- Auto-instrumentations (`getWebAutoInstrumentations`) produce **traces only**, not metrics — explicit `getMeter()` is required for any browser metrics

### Grafana dashboard — 6 Angular panels added
Six new panels added to `ai-chat.json` at rows y=52–76:
- **panel-20**: Messages Sent Rate (`rate(angular_message_sent_total[5m])`)
- **panel-21**: Conversation Starts by Module (breakout by `{{module}}`)
- **panel-22**: Time to First Chunk p50/p95 (histogram quantile)
- **panel-23**: End-to-End Response Duration p50/p95 (histogram quantile)
- **panel-24**: Cancellations & Failures rate (two series)
- **panel-25**: Admin Page Load p95 (breakout by `{{page}}`)

**Gotcha — Grafana v2 panel schema**: new panels must exactly match the structure of existing exported panels. Key requirements that caused a `"Cannot read properties of null (reading 'map')"` load error when wrong:
- Queries use `kind: 'PanelQuery'` (outer wrapper) containing `kind: 'DataQuery'` (inner)
- Each query needs `refId` and `hidden: false` inside the `PanelQuery.spec`
- Panel spec needs `id: <number>` and `links: []`
- `datasource.name` must be lowercase (`prometheus`, not `Prometheus`)
- `vizConfig` must have `group`, `kind: 'VizConfig'`, `version`, and full `options` block with `legend`/`tooltip`/`annotations`
- `QueryGroup.spec` must include `queryOptions: {}` and `transformations: []`

## Onboarding runbook — required note

**Add to the developer onboarding runbook:**

> **Angular OTel metrics are not automatic.**
> The `getWebAutoInstrumentations()` package instruments XHR/fetch/document-load as *traces*. It does not emit metrics. To get browser-side metrics into Prometheus/Grafana:
> 1. Import `metrics` from `@opentelemetry/api`
> 2. Call `const meter = metrics.getMeter('ai-chat-angular')` inside `initOtel`
> 3. Use the meter to create and record instruments (counters, histograms, etc.)
>
> The `MeterProvider` is wired up and the global provider is set — recording a metric is the only missing step.

> **`docker compose up -d <service>` does not reload config** if the container image is unchanged. When you edit a bind-mounted config file (e.g. `otel-collector.yml`) you must run:
> ```
> docker compose up -d --force-recreate <service>
> ```

## Dashboard metric coverage (as of this session)

| Panel | Source | Status |
|---|---|---|
| HTTP Request Rate by Endpoint | .NET Prometheus scrape | Working |
| HTTP p95 Latency | .NET Prometheus scrape | Working |
| HTTP Error Rate (5xx %) | Recording rule | Working — empty until 5xx occurs |
| SignalR Active Connections | .NET Prometheus scrape | Working |
| Chat Messages Throughput | .NET custom metric | Working |
| Endpoint Uptime (Blackbox) | Blackbox `/health` probe | Working |
| Log Rate by Level | Loki | Depends on `LOKI_URI` env var being set |
| Warnings & Errors | Loki | Depends on `LOKI_URI` env var being set |
| JS Error Count | Sentry datasource | Alerting not supported via Sentry DS |
| Messages Sent Rate | Angular OTel → collector → Prometheus | Added session 4 |
| Conversation Starts by Module | Angular OTel → collector → Prometheus | Added session 4 |
| Time to First Chunk p50/p95 | Angular OTel → collector → Prometheus | Added session 4 |
| Response Duration p50/p95 | Angular OTel → collector → Prometheus | Added session 4 |
| Cancellations & Failures Rate | Angular OTel → collector → Prometheus | Added session 4 |
| Admin Page Load p95 | Angular OTel → collector → Prometheus | Added session 4 |

## Deprecated component names (v0.156.0 warnings)
The collector logs warn about deprecated aliases — these do not cause failures but should be updated before Chunk 6:
- `otlphttp` → `otlp_http`
- `resourcedetection` → `resource_detection`
- `prometheusremotewrite` → `prometheus_remote_write`

## Next
Chunk 4: write `specs-and-plans/aws-infra-spec.md`. See open questions in `CLAUDE.md` before starting.
