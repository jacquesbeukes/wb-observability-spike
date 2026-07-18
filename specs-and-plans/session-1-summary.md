# Session 1 Summary — .NET Backend Instrumentation

**Branch:** `Feature/WD-7529-spike` (AI Chat Middleware repo)  
**Date:** 2026-07-18

---

## ✅ What got built

### New files
| File | Purpose |
|---|---|
| `src/AiChat.Api/Middleware/CorrelationIdMiddleware.cs` | Reads/generates `X-Correlation-ID`, echoes in response header, pushes to `Serilog.Context.LogContext` |
| `src/AiChat.Api/Observability/AppMetrics.cs` | Singleton Prometheus metrics: `signalr_connections_active` (Gauge), `chat_messages_total{direction}` (Counter) |
| `src/AiChat.Api/Observability/ObservabilityConstants.cs` | Central string constants for all observability code — config keys, defaults, OTel resource names, metric names, label names/values, Loki label keys |
| `src/AiChat.Api/Startup/ObservabilityExtensions.cs` | `AddAppObservability()` — OTel SDK wiring (metrics, OTLP exporter, resource attributes); reads from `Observability:*` config section |
| `src/AiChat.Api/Startup/SerilogExtensions.cs` | `UseAppSerilog()` — full Serilog pipeline: enrichers, OperationCancelled filter, console sink, conditional Loki sink |

### Modified files
| File | What changed |
|---|---|
| `AiChat.Api.csproj` | Added 7 packages: `Serilog.Sinks.Grafana.Loki`, `prometheus-net.AspNetCore`, `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`, `Serilog.Enrichers.OpenTelemetry` |
| `Program.cs` | Wired: `UseAppSerilog()`, `CorrelationIdMiddleware`, `UseHttpMetrics()`, `MapMetrics()`, `AddAppObservability()` |
| `appsettings.json` | Added `Observability` section: `ServiceName`, `OtlpEndpoint`, `EnvironmentName` (local defaults) |
| `appsettings.Production.json` | Added `Observability` section with production placeholders; fixed dangling comma in `Serilog` block |
| `Areas/Chat/Hubs/AiChatHub.cs` | `SignalRConnectionsActive.Inc/Dec()` on connect/disconnect; `ChatMessagesTotal("received").Inc()` on `SendMessage` |
| `Chat/StreamingRelayService.cs` | `ChatMessagesTotal("sent").Inc()` when `ResponseCompleted` fires |

### Structured logging audit
Zero `$"..."` interpolated Serilog calls found — the codebase was already clean.

---

## 🧪 Test results

```
Passed: 64   Failed: 0   Skipped: 9 (RequiresDocker)   Total: 73
Build: 0 warnings, 0 errors
```

Skipped tests require a live Postgres container (`Category=RequiresDocker`). All non-Docker tests pass.

### Manual verification
- `GET /metrics` → HTTP 200, Prometheus text format, no auth required
- HTTP runtime metrics present (`http_request_duration_seconds`, `http_requests_received_total`, .NET GC/heap series)
- `X-Correlation-ID` generated on requests without the header
- `X-Correlation-ID: test-abc-123` echoed back exactly when passed in
- Every structured log line carries `CorrelationId`, `TraceId`, `SpanId` as discrete JSON fields
- OTel SDK initialises without errors; OTLP exporter skipped in dev (no `OTEL_EXPORTER_OTLP_ENDPOINT` set)

---

## ⚠️ Gotchas / decisions that differ from the plan

### Loki sink wired in code, not appsettings
The plan specified configuring the Loki sink in `appsettings.Development.json` / `appsettings.Production.json`. This failed at runtime — `Serilog.Settings.Configuration` cannot deserialise `propertiesAsLabels: []` (empty JSON array) as a `string[]` parameter; it attempts an `Invalid cast from System.String to System.String[]` when the array is empty.

**Fix:** The Loki sink is configured programmatically in `Program.cs` using `.WriteTo.Conditional(...)`. The sink is only registered when `LOKI_URI` is set in the environment, so local dev without Docker Compose produces no errors. When running with Docker Compose (or on EB with the env var set), the sink activates automatically.

**Impact on Chunk 3:** The `appsettings.Production.json` token-substitution approach (`#{LOKI_URI}#`) is not used. `LOKI_URI` and `ENV_NAME` are read directly from `Environment.GetEnvironmentVariable()` in `SerilogExtensions`; the OTLP endpoint and service name are read from the `Observability:*` config section (overridable via `Observability__OtlpEndpoint` env var on EB). No change to the deployment model.

### `prometheus-net.AspNetCore` v8 API
The plan referenced `MapPrometheusScrapingEndpoint()` — this was renamed to `MapMetrics()` in v8. Used `MapMetrics()`.

### Custom metrics: lazy initialisation
`signalr_connections_active` and `chat_messages_total` do not appear in `GET /metrics` output until the first connection/message occurs. This is prometheus-net v8's default behaviour (no zero-value series emitted until first use). Expected and harmless.

### `ENV_NAME` hardcode decision
For local dev (no `ENV_NAME` env var), the Loki label defaults to `"local"` in code. This matches the plan spec.

---

## 📋 Jira test plan / verification steps

1. `dotnet test --filter "Category!=RequiresDocker"` → 64 passed, 0 failed
2. `GET /metrics` returns HTTP 200 with `Content-Type: text/plain; version=0.0.4`
3. Response includes `http_requests_received_total`, `http_request_duration_seconds`, `.NET GC/heap` metrics
4. `curl -I http://<host>/health` → response contains `X-Correlation-ID` header
5. `curl -H "X-Correlation-ID: my-id" -I http://<host>/health` → `X-Correlation-ID: my-id` echoed back
6. Structured log output contains `"CorrelationId"`, `"TraceId"`, `"SpanId"` as discrete fields (not embedded in message string)
7. `grep '\$"' src/**/*.cs` → zero results (no interpolated Serilog calls)
8. Start app with `LOKI_URI=http://loki:3100 ENV_NAME=local` → Loki sink registers, no startup error
9. Start app without `LOKI_URI` → no Loki errors in console
10. After a SignalR connection: `signalr_connections_active 1` appears in `/metrics`
11. After a `SendMessage`: `chat_messages_total{direction="received"} 1` in `/metrics`
12. After relay completes: `chat_messages_total{direction="sent"} 1` in `/metrics`

---

## 🚀 Next session (Session 2) — Angular frontend instrumentation

**What to build:** Chunk 2 from `specs-and-plans/implementation-plan.md`

- `window.__env` runtime config pattern (served by .NET from env vars)
- `environment.ts` updated to read from `window.__env` with local fallbacks
- `@sentry/angular` installed and initialised with `beforeSend` OTel trace hook
- `CorrelationIdInterceptor` Angular `HttpInterceptor`
- `@opentelemetry/sdk-web` + `@opentelemetry/auto-instrumentations-web` (metrics only)

**Setup needed before starting Session 2:**
- Confirm Angular major version in `ClientApp/package.json` → select matching `@sentry/angular` version
- Confirm `angular.json` has no existing `fileReplacements` config (if it does, use those instead of `window.__env`)
- A Sentry project `ai-chat-angular` needs to be created at sentry.io and DSN stored in Secrets Manager (`wbi-ai/chat/sentry-dsn`) — do this before the session or at the start of it
