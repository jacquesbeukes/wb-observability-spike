# Session 2 Summary — Angular Frontend Instrumentation

**Branch:** `Feature/WD-7529-spike` (AI Chat Middleware repo — `ClientApp/` subtree)
**Date:** 2026-07-18

---

## ✅ What got built

### New files

| File | Purpose |
|---|---|
| `ClientApp/src/app/env.d.ts` | TypeScript ambient declaration for `window.__env` — gives the `Window` interface a typed `__env?: WindowEnv` property |
| `ClientApp/src/app/observability/observability.ts` | `initSentry()` and `initOtel()` — called in `main.ts` before `bootstrapApplication()` |
| `ClientApp/src/app/observability/correlation-id.interceptor.ts` | Angular `HttpInterceptorFn` — reads `X-Correlation-ID` response header, calls `setTag('correlation_id', ...)` |
| `ClientApp/src/app/observability/sentry-set-tag.token.ts` | `InjectionToken<typeof Sentry.setTag>` — indirection so tests can substitute a spy without fighting the ES module non-writable descriptor |
| `ClientApp/src/app/observability/correlation-id.interceptor.spec.ts` | 3 Karma/Jasmine tests for the interceptor |
| `src/AiChat.Api/Middleware/WindowEnvMiddleware.cs` | Intercepts `GET /index.html`, injects `<script>window.__env = {...}</script>` from .NET env vars before `</head>` |

### Modified files

| File | What changed |
|---|---|
| `ClientApp/package.json` | Added 8 packages: `@sentry/angular@^10`, `@opentelemetry/sdk-trace-web`, `@opentelemetry/sdk-metrics`, `@opentelemetry/auto-instrumentations-web`, `@opentelemetry/exporter-metrics-otlp-http`, `@opentelemetry/exporter-trace-otlp-http`, `@opentelemetry/resources`, `@opentelemetry/semantic-conventions` |
| `ClientApp/src/environments/environment.ts` | Added `envName`, `sentryDsn`, `otlpEndpoint` — all read from `window.__env` with local fallbacks |
| `ClientApp/src/main.ts` | Calls `initSentry()` + `initOtel()` before `bootstrapApplication()` |
| `ClientApp/src/app/app.config.ts` | Added `withInterceptors([correlationIdInterceptor])` to `provideHttpClient()`; provides `SentryErrorHandler` via `createErrorHandler()` |
| `src/AiChat.Api/Program.cs` | Registers `WindowEnvMiddleware` immediately before `UseStaticFiles()` |

---

## 🧪 Test results

```
TOTAL: 47 SUCCESS  (was 44 before this session — 3 new tests added)
Build: 0 errors, 1 bundle budget warning (see gotchas)
```

Coverage: 70.2% statements / 62.8% branches (up from 67.2% / 60.3%).

---

## ⚠️ Gotchas / decisions that differ from the plan

### Separate `ANGULAR_OTLP_ENDPOINT` env var (deviation from plan)
The variable strategy table says `OTEL_EXPORTER_OTLP_ENDPOINT` feeds both .NET (gRPC, port 4317) and Angular (HTTP/protobuf, port 4318). These are different protocols and ports. Introduced `ANGULAR_OTLP_ENDPOINT` as a separate env var so the two can be set independently. The .NET code still reads `OTEL_EXPORTER_OTLP_ENDPOINT`; the new var feeds `window.__env.otlpEndpoint`.

**Impact on Chunk 3:** Add `ANGULAR_OTLP_ENDPOINT=http://localhost:4318` to the Docker Compose env for the .NET service (or the host running it). Update the variable strategy table in `implementation-plan.md` if desired.

### `Resource` API change in `@opentelemetry/resources` v2
`new Resource({...})` was removed; the correct API is `resourceFromAttributes({...})`. Fixed by importing `resourceFromAttributes`.

### MeterProvider global registration
`MeterProvider` does not have a `register()` method (unlike `WebTracerProvider` which sets itself as the global tracer provider). Metrics from auto-instrumentation are attached to the provider directly via `setMeterProvider(meterProvider)` when the instrumentation is configured. The `(meterProvider as any).register?.()` call is a no-op but harmless — left with a comment. **Note:** If auto-instrumentation metrics are not appearing in Chunk 3 testing, add `metrics.setGlobalMeterProvider(meterProvider)` from `@opentelemetry/api`.

### `@sentry/angular` v10 — `createErrorHandler()` returns an object, not a class
`createErrorHandler()` is the correct factory for Angular 14+ standalone apps. Works correctly.

### Bundle size budget warning
OTel SDK + Sentry adds ~400 kB gzipped to the initial bundle, pushing total initial over the Angular default 500 kB budget. This is expected for a fully-instrumented SPA. The warning is cosmetic — the build succeeds. If the team wants to suppress it, raise `maximumWarning` in `angular.json`; don't raise `maximumError`.

### Sentry DSN is a placeholder
No real Sentry project existed at session time. `sentryDsn` defaults to `''` in `environment.ts`, and `initSentry()` is a no-op when DSN is empty. A real DSN must be stored in Secrets Manager (`wbi-ai/chat/sentry-dsn`) before end-to-end Sentry verification is possible (see Jira test plan step 1).

### `spyOn(Sentry, 'setTag')` does not work in Karma
`@sentry/angular` exports are non-writable ES module properties — `spyOn()` throws `setTag is not declared writable`. Fixed by introducing `SENTRY_SET_TAG` injection token; tests substitute a `jasmine.createSpy()` via the token instead.

---

## 📋 Jira test plan / verification steps

**Prerequisites:**
1. Create Sentry project `ai-chat-angular` at sentry.io; store the DSN in Secrets Manager (`wbi-ai/chat/sentry-dsn`); set `SENTRY_DSN=<dsn>` as an EB environment property on all environments. Until this is done, steps 1–4 below cannot be verified end-to-end.

**Tests:**

1. **Sentry initialises with DSN**: Set `SENTRY_DSN=<real-dsn>` locally; run app; open browser console — no Sentry errors. Navigate to sentry.io → `ai-chat-angular` project → Issues — project is visible.

2. **Deliberate JS error reaches Sentry under `local` environment**:
   - In a dev build, add `throw new Error('session-2-test')` to any component `ngOnInit`; load the page.
   - In sentry.io → `ai-chat-angular` → Issues: event appears with `environment: local`.

3. **`trace_id` tag on Sentry events**:
   - With OTel running, the `beforeSend` hook attaches `trace_id` and `span_id` from the active span.
   - Check a captured Sentry event's Tags — `trace_id` should be a 32-char hex string (may be `0000...` if no active span, which is fine at this stage).

4. **`correlation_id` tag after an API call**:
   - Trigger any API call (e.g. start a conversation in the chat UI).
   - Inspect the response in Network tab — confirm `X-Correlation-ID` header is present in the response.
   - In sentry.io, a subsequently captured error should carry `correlation_id` tag with that value.

5. **`window.__env` values in browser console**:
   - Start app with `ENV_NAME=staging SENTRY_DSN=<dsn> ANGULAR_OTLP_ENDPOINT=http://localhost:4318`.
   - Open browser console; type `window.__env` → `{ envName: "staging", sentryDsn: "...", otlpEndpoint: "http://localhost:4318" }`.
   - `environment.envName` should be `"staging"` in the running app.

6. **`window.__env` fallback when served by `ng serve` (no .NET host)**:
   - Run `npm start`; open browser console; type `window.__env` → `undefined`.
   - `environment.envName` should be `"local"` (fallback in `environment.ts`).

7. **OTel SDK initialises — network tab shows export attempts**:
   - Open Network tab; filter for `v1/metrics` or `v1/traces`.
   - Requests to `http://localhost:4318/v1/...` appear (they will fail with CORS/connection errors until Chunk 3 — that is expected).

8. **`npm test -- --watch=false --browsers=ChromeHeadless`** → `TOTAL: 47 SUCCESS`

9. **`npm run build`** → `Application bundle generation complete`, 0 errors

---

## 🚀 Next session (Session 3) — Local platform stack

**What to build:** Chunk 3 from `specs-and-plans/implementation-plan.md`

- Full `monitoring/` directory: `prometheus.yml`, `prometheus-aws.yml`, `loki.yml`, `blackbox.yml`, `otel-collector.yml`
- Grafana provisioning: datasource YAMLs (Prometheus, Loki with Derived Fields, Sentry), dashboards pointer, alerting rules
- Five custom Dockerfiles + Docker Compose service blocks
- `ai-chat.json` dashboard and `platform.json` meta-monitoring dashboard

**Setup needed before starting Session 3:**
- Docker Desktop must be running (platform stack runs via Docker Compose)
- Confirm where the Docker Compose file lives (application root or `monitoring/` — the plan says add 5 service blocks to "the application's compose file")
- Confirm `GRAFANA_ADMIN_PASSWORD` will be supplied for the local `.env` file
- Add `ANGULAR_OTLP_ENDPOINT=http://otel-collector:4318` to the .NET service env in Docker Compose (new var from this session)
- OTel Collector CORS config must allow the Angular origin — this is the unblocking fix for the CORS errors observed in the browser network tab after Session 2

**Key decision for Chunk 3 start:**
The bundle size budget warning (896 kB initial > 500 kB budget) — decide whether to raise `maximumWarning` in `angular.json` or leave the warning as-is. It does not block anything.
