# Observability Platform — Production-Grade Architecture Review

**Reviewer context:** Principal Systems Architect / Staff SRE  
**Stack:** Angular/.NET 10 / AWS Elastic Beanstalk → OTel Collector → Prometheus · Loki · Grafana · Tempo · Sentry  
**Region:** `ap-southeast-2` · July 2026

---

## 1. Executive Summary

The architecture is **viable and well-considered for its intended scope** — a small number of services (2–10), moderate traffic, team-managed infrastructure, budget ceiling of ~$82/month. The deliberate `env`-label spine, the single-pane Grafana approach, the config-as-code discipline, and the clean separation between PoC and production-grade tiers are all good engineering.

The plan has **no catastrophic design errors** but carries three structural gaps that will cause pain before production-grade delivery is complete:

1. The OTel Collector has **no memory limiter and no batch processor** — it will OOM and drop data under any meaningful burst.
2. The Loki configuration is **underspecified** in ways that will cause either index thrash or unqueryable data.
3. The correlation story between the three pillars is **half-built** — TraceID/SpanID will not flow into Loki log lines or the Sentry scope without explicit wiring that is missing from the spec.

Everything else is at the level of "important to get right" rather than "blocker." The sections below provide remediation for all gaps.

---

## 2. Critical Flaws (Blockers)

### 2.1 OTel Collector has no memory limiter or batch processor

**Risk: OOM crash → total telemetry loss across all services simultaneously.**

The `otel-collector.yml` in the spec is bare:

```yaml
service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      exporters: [loki]
```

There are **no processors at all**. This means:

- Every OTLP payload is forwarded immediately with no batching → Prometheus and Loki receive a flood of tiny HTTP calls → both backends hit connection-per-request overhead.
- There is no memory ceiling. At 0.25 vCPU / 512 MB Fargate, a burst from two services (Angular OTLP from multiple browser tabs + .NET OTLP on a traffic spike) can push the container past 512 MB. ECS Fargate OOMs the container; the task restarts; all buffered data is lost.

**Required fix — add a processor block before the PoC AWS deployment:**

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 400        # ~80% of the 512 MB Fargate allocation
    spike_limit_mib: 80

  batch:
    send_batch_size: 1000
    timeout: 5s

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
    traces:                         # when Tempo is added
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo]
```

The `memory_limiter` must always be **first** in the processor chain so it can refuse ingestion before downstream processors fill memory. The `batch` processor must be **last** before the exporter.

---

### 2.2 Prometheus remote-write receiver is not enabled by default

The spec exports from the OTel Collector to `http://prometheus:9090/api/v1/write`. This endpoint is Prometheus's **remote-write receiver**, which is **disabled by default** and requires the `--web.enable-remote-write-receiver` startup flag.

The custom `Dockerfile.prometheus` and ECS task definition must pass this flag. Without it, the OTel Collector's `prometheusremotewrite` exporter will receive HTTP 404 on every flush and silently drop all OTel-sourced metrics. (Native Prometheus scrape from `/metrics` still works — so the Prometheus-scraped .NET metrics will appear fine, masking the failure.)

**Fix:**

In the ECS task definition command array:
```json
"command": [
  "--config.file=/etc/prometheus/prometheus.yml",
  "--web.enable-remote-write-receiver"
]
```

And in Docker Compose:
```yaml
prometheus:
  image: prom/prometheus
  command:
    - --config.file=/etc/prometheus/prometheus.yml
    - --web.enable-remote-write-receiver
```

---

### 2.3 Loki configuration is dangerously underspecified

The spec references `loki/loki.yml` as "minimal local config" (Chunk 3) and "S3 backend" (Chunk 6) but never shows the file's contents. This is a blocker for two reasons:

**A. Missing `limits_config` — no ingestion rate limits means Loki will accept an unlimited log burst and either OOM or corrupt its index.**

A minimum safe config:
```yaml
limits_config:
  ingestion_rate_mb: 4         # per-tenant per-second
  ingestion_burst_size_mb: 8
  max_streams_per_user: 10000
  max_label_names_per_series: 15
  max_label_value_length: 2048
  retention_period: 744h       # 31 days — matches the S3 lifecycle rule
```

**B. The PoC uses filesystem storage with no defined path, chunk size, or compaction schedule.** Without a `compactor` block and a defined `storage_config`, Loki will accumulate WAL chunks on the ephemeral ECS container filesystem until it fills up or the task restarts and loses everything with no warning. Since PoC data loss is accepted, this is tolerable — but `loki.yml` must at minimum be a valid working config, not a placeholder.

A minimal but complete PoC `loki.yml`:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 8
  max_streams_per_user: 10000
```

The production-grade S3 backend migration (Chunk 6) adds a `storage_config.aws` block and a `compactor` — that part of the plan is fine in principle but needs the full config spelled out before Chunk 6 starts.

---

### 2.4 High-cardinality risk: OTel auto-instrumentation generates per-URL metric labels

**Risk: Prometheus TSDB cardinality explosion → multi-GB memory usage, slow queries.**

`OpenTelemetry.Instrumentation.AspNetCore` by default creates metric series with the HTTP route as a label. With well-structured ASP.NET Core routes (e.g., `/api/conversations/{id}`), this is fine — routes are templated. But:

- The Angular `@opentelemetry/auto-instrumentations-web` will instrument **all outbound XHR/fetch calls** and use the full URL (including query strings) as the `http.url` attribute. If any metric is created with `http.url` as a label dimension, that's unbounded cardinality.
- Any custom metrics that include `TenantId`, `ConversationId`, `UserId`, or similar per-entity values in label names will create one time series per unique value. The SignalR `chat_messages_total` counter uses a `direction` label (fine — only two values), but future contributors may add labels without understanding this constraint.

**Fix — add a transform processor to the OTel Collector metrics pipeline:**

```yaml
processors:
  transform/sanitize_metrics:
    metric_statements:
      - context: datapoint
        statements:
          # Drop query strings from HTTP URL attributes before they become labels
          - replace_pattern(attributes["http.url"], "\\?.*", "")
```

And document in the onboarding prompt (Chunk 7): **never put user IDs, tenant IDs, request IDs, or conversation IDs in Prometheus metric label dimensions**. They go in log line fields only.

---

## 3. High-Value Recommendations

### 3.1 Add `resourcedetection` processor to the OTel Collector

The spec sets `deployment.environment` from `ENV_NAME` in the SDK, but the OTel Collector can enrich all telemetry with AWS resource attributes automatically:

```yaml
processors:
  resourcedetection:
    detectors: [env, ecs, ec2]
    timeout: 5s
    override: false
```

This adds `cloud.region`, `cloud.availability_zone`, `aws.ecs.cluster.arn`, and `aws.ecs.task.arn` to all spans, metrics, and logs without any SDK-side changes. When debugging a production incident, knowing which AZ a task was in is often the first useful narrowing signal.

---

### 3.2 Explicitly restrict Loki labels to prevent high-cardinality index streams

The `Serilog.Sinks.Grafana.Loki` package will by default forward **all Serilog enriched properties as Loki labels** unless explicitly restricted. If the Serilog context includes `TenantId`, `CorrelationId`, `ConversationId`, or `RequestId` (all planned for this stack), each unique value creates a new Loki stream. At 1,000 tenants × 3 environments, you can easily hit 50,000+ active streams — Loki starts rejecting pushes and the index bloats.

**Fix — set `propertiesAsLabels: []` in `appsettings.*.json`:**

```json
{
  "Serilog": {
    "WriteTo": [{
      "Name": "GrafanaLoki",
      "Args": {
        "uri": "#{LOKI_URI}#",
        "labels": [
          { "key": "service", "value": "<service-name>" },
          { "key": "env",     "value": "#{ENV_NAME}#" }
        ],
        "propertiesAsLabels": []
      }
    }]
  }
}
```

`propertiesAsLabels: []` prevents the sink from promoting any Serilog property to a Loki label. `TenantId`, `CorrelationId`, `TraceId`, and `SpanId` will be embedded in the log line's structured JSON body (queryable via `| json` in LogQL), not in the stream index. This is the correct pattern.

---

### 3.3 Prometheus scrape jobs are missing `scrape_interval`, `scrape_timeout`, and `scheme`

The `prometheus.yml` snippet only shows `static_configs`. A complete scrape job for production needs:

```yaml
- job_name: ai-chat-middleware
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: https          # EB environments serve HTTPS
  tls_config:
    insecure_skip_verify: false
  static_configs:
    - targets: ['<service-dev-host>:<port>']
      labels:
        service: ai-chat-middleware
        env: dev
```

Without explicit `scrape_interval`, Prometheus uses the global default (usually 1 minute — coarse for real-time alerting). Without `scrape_timeout`, a hung EB instance blocks a Prometheus scrape thread for the full default. If EB environments serve HTTPS, `scheme: https` is required, or Prometheus will attempt HTTP and receive a redirect or TLS handshake failure.

---

### 3.4 Add recording rules for standard dashboard panels

The four standard panels (request rate, p95 latency, error rate, .NET runtime) execute `rate()` or `histogram_quantile()` queries on every Grafana refresh. With three environments and multiple services, these queries run against potentially millions of raw samples on every dashboard load.

Add recording rules for each service's key panels:

```yaml
groups:
  - name: ai_chat_recording
    interval: 60s
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum by (job, env, http_route, http_response_status_code) (rate(http_requests_total[5m]))

      - record: job:http_request_duration_seconds:p95
        expr: histogram_quantile(0.95, sum by (job, env, le) (rate(http_request_duration_seconds_bucket[5m])))
```

Grafana panels then query the pre-computed `job:*` series — queries that took 3–5 seconds drop to under 200ms.

> **Note:** Recording rules must be evaluated by Prometheus, not Grafana. The current spec explicitly avoids a `prometheus/rules/` folder (all alerting in Grafana Unified Alerting). Recording rules are not alerting rules — they must live in a Prometheus rules file, or Grafana's recording rules feature (Grafana Cloud / Enterprise) must be used. This needs a resolution before production dashboards are critical.

---

### 3.5 Blackbox probe interval and alert duration are unspecified

The spec doesn't define probe intervals. At the Prometheus global default of 1 minute, an outage may not be detected for up to 1 minute, and the `EndpointDown` alert adds another 5 minutes — on-call is paged potentially 6 minutes after the outage starts.

**Recommendation:**

```yaml
- job_name: blackbox
  scrape_interval: 30s
  scrape_timeout: 15s
  metrics_path: /probe
  params:
    module: [http_2xx]
```

And set `for: 2m` on the `EndpointDown` alert rule (two consecutive probe failures = real outage, not a transient blip):

```yaml
- alert: ServiceEndpointDown
  expr: probe_success{job="blackbox"} == 0
  for: 2m
  labels:
    severity: critical
    env: production
```

**Location coverage:** One Blackbox Exporter instance in `ap-southeast-2` will not detect a Sydney-region AZ failure or routing issues from other regions. Consider AWS CloudWatch Synthetics for a second probe origin at production-grade delivery.

---

### 3.6 OTel Collector sizing: 0.25 vCPU is insufficient for multi-service production

At 0.25 vCPU Fargate, the Collector gets approximately 250ms of CPU per second. With two or more services sending OTLP (three .NET backends + three Angular frontends = up to 6 concurrent OTLP streams), the Collector will CPU-throttle under any real load. The `batch` processor will buffer, but if CPU can't drain the buffer fast enough, `memory_limiter` will start refusing ingestion.

The spec states the Collector is "Unchanged" from PoC to production-grade. **Recommendation:** Upgrade to **0.5 vCPU / 1 GB** in production-grade (matching the Loki spec). Adds ~$4–5/month to the platform cost.

---

## 4. Correlation & Tooling Blind Spots

### 4.1 TraceID/SpanID are not wired into Loki log lines

The spec defers `WithTracing(...)` to Chunk 6, but **trace context injection into logs must be wired at Serilog configuration time** — not at trace enablement time. If this is added later, existing deployed services will never correlate logs to traces.

**Required addition in Chunk 1:**

Add `Serilog.Enrichers.OpenTelemetry` package, then:

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.WithOpenTelemetryTraceId()
    .Enrich.WithOpenTelemetrySpanId()
    // ... rest of config
```

This writes `TraceId` and `SpanId` as structured fields into every log line. The fields will be empty until Chunk 6 enables tracing — that's fine. When Tempo is deployed, Grafana's Derived Fields feature on the Loki datasource can auto-link any log line containing a `TraceId` to the Tempo trace explorer: one click from a log line to the full distributed trace.

Without this, the Loki → Tempo correlation link never works.

---

### 4.2 Grafana Loki datasource needs Derived Fields configured

Even after TraceID appears in log lines, Grafana won't auto-link to Tempo without a **Derived Field** rule on the Loki datasource:

```yaml
# grafana/provisioning/datasources/loki.yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: '"TraceId":"([a-f0-9]{32})"'
          url: '$${__value.raw}'
          datasourceUid: tempo
```

Add this in Chunk 3 (local stack). The derived field rule is harmless when no `TraceId` is present in logs — no reason to wait for Chunk 6.

---

### 4.3 Prometheus exemplars are not enabled

Exemplars annotate a metric sample with the TraceID of the request that produced it — this is the `metric → trace` correlation link. They are disabled by default.

Enable in Prometheus (`prometheus.yml` global section):
```yaml
storage:
  tsdb:
    exemplar_storage:
      enable_exemplars: true
      max_exemplars: 100000
```

Grafana displays exemplars as diamond markers on time-series graphs. Clicking a diamond opens the Tempo trace explorer for that specific request's trace — the `slow request on a dashboard → see the actual trace` flow the spec describes wanting but doesn't implement.

---

### 4.4 Sentry events don't carry TraceID — no path from Sentry back to Loki/Tempo

The spec's correlation flow is `Grafana Sentry panel → sentry.io` (one direction only). When a user reports a frontend JS error, the Sentry `event_id` has no path back to the CorrelationID or TraceID in Loki.

**Fix — attach the active OTel TraceID to Sentry events in `beforeSend`:**

```typescript
Sentry.init({
  dsn: environment.sentryDsn,
  environment: environment.envName,
  beforeSend(event) {
    const span = trace.getActiveSpan();
    if (span) {
      const ctx = span.spanContext();
      event.tags = {
        ...event.tags,
        trace_id: ctx.traceId,
        span_id: ctx.spanId,
      };
    }
    return event;
  }
});
```

With this in place, a Sentry error carries a `trace_id` tag. From sentry.io you copy the trace ID and use it in Grafana's Tempo explorer or as a Loki filter: `{service="ai-chat"} | json | TraceId="<id>"`.

This closes the full loop: `Grafana Sentry panel → sentry.io event → trace_id tag → Loki/Tempo`.

---

### 4.5 CorrelationID is not propagated from .NET to Angular

The `CorrelationIdMiddleware` generates a GUID and echoes it in `X-Correlation-ID` response headers, but the Angular OTel SDK and Sentry SDK don't read this header or include it in outbound requests or error events. A frontend Sentry error and the corresponding backend log line share a CorrelationID in principle but can't be connected without manual timestamp cross-referencing.

**Fix — add an Angular HTTP interceptor:**

```typescript
@Injectable()
export class CorrelationIdInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          const correlationId = event.headers.get('X-Correlation-ID');
          if (correlationId) {
            Sentry.setTag('correlation_id', correlationId);
          }
        }
      })
    );
  }
}
```

This makes Sentry frontend errors linkable to backend log lines by `CorrelationId` — a direct bridge between the two non-OTel systems.

---

## 5. Testing & Rollout Plan

### 5.1 Pre-Chunk-5 pipeline load test (missing from the current plan)

The implementation plan tests each chunk with a manual smoke test — insufficient before going live. The OTel Collector, Prometheus, and Loki have never been tested under concurrent load from multiple environments.

**Run this against the local Docker Compose stack before Chunk 5:**

```bash
# Generate HTTP load against the .NET app (~100 req/s drives prometheus-net at volume)
k6 run --vus 50 --duration 5m script.js
```

Watch for these failure signals:
- OTel Collector: "Dropping data" or "exporter queue is full" in container logs
- Prometheus: scrape duration > `scrape_timeout`
- Loki: HTTP 429 (rate limit hit) or "too many streams" errors
- Grafana: dashboard query timeout

Enable the OTel Collector's internal metrics endpoint (add `extensions: [health_check, pprof, zpages]`) and check `otelcol_processor_batch_*` and `otelcol_exporter_queue_*` metrics at `http://localhost:8888/metrics`.

---

### 5.2 Staged per-environment rollout sequence for Chunk 5

The current plan notes deploy order but doesn't prescribe a safe cutover sequence:

1. **Deploy OTel Collector first** — confirm its health endpoint is reachable from the EB security group before proceeding.
2. **Deploy Prometheus** — confirm scrape targets for `dev` environment only initially.
3. **Deploy Loki** — push a test log line via `curl` from within the VPC before enabling the .NET Loki sink.
4. **Deploy Grafana** — confirm datasource connections in the UI before enabling alerts.
5. **Deploy Blackbox** — confirm probe targets are reachable.
6. **Enable `dev` EB environment properties** (`LOKI_URI`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `ENV_NAME=dev`) → restart EB app. Verify data appears in Grafana.
7. **Enable `staging` then `production`** in sequence with a 15-minute gap each — a misconfigured security group or DNS resolution issue then affects one environment, not all three simultaneously.

---

### 5.3 Alert rule test procedure (currently absent)

Before Chunk 5 goes live, verify each alert rule fires and resolves correctly:

1. **`EndpointDown`:** Block the Blackbox Exporter target port. Confirm alert fires within 2 minutes. Restore and confirm it resolves.
2. **`HighErrorRate`:** Send requests that return 5xx responses. Confirm alert fires within 5 minutes.
3. **`TenantErrorSpike`:** Emit 51 structured log lines with the same tenant ID and `level=error` within 2 minutes via a test script. Confirm the LogQL-based Grafana alert fires.

Document results in Chunk 7's retrospective — they become the regression baseline for the onboarding pattern.

---

### 5.4 Production-grade migration additions for Chunk 6

**A. Loki S3 IAM permissions.** Before switching Loki to the S3 backend, confirm the Loki ECS task execution role has `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` on the `wbi-loki-chunks` bucket. Missing S3 permissions cause Loki to start successfully but fail silently on chunk writes — logs appear to ingest but are unqueryable after the retention window.

**B. Prometheus EFS I/O mode.** Prometheus TSDB performs many small random reads during compaction. EFS general-purpose mode has an IOPS ceiling that compaction will hit, causing Prometheus to stall for 30–60 seconds every few hours. Use **EFS Max I/O mode** for the Prometheus mount point, or consider a GP3 EBS volume if single-AZ availability is acceptable (Fargate supports EBS attachment).

---

## Summary Table

**Post-review audit gaps (fixed July 2026):**

| # | Gap | Fixed in |
|---|---|---|
| A | Chunk 1 deliverables missing `propertiesAsLabels` and `Serilog.Enrichers.OpenTelemetry` enrichers | plan Chunk 1 |
| B | Chunk 2 deliverables missing `beforeSend` TraceID hook and `CorrelationIdInterceptor` | plan Chunk 2 |
| C | Docker Compose `prometheus` service not mounting `rules/` directory | spec §1.3 |
| D | §2.3 onboarding checklist missing `prometheus/rules/<service>.yml` step | spec §2.3 |
| E | Service Onboarding Prompt missing `propertiesAsLabels`, enrichers, `beforeSend`, interceptor | spec prompt |
| F | `health_check` extension missing from `otel-collector.yml` | spec §1.2 |
| G | Chunk 6 gotcha incorrectly claimed Tempo port clashes with Collector | plan Chunk 6 |
| H | Architecture sketch missing `prometheus/rules/` and Derived Fields annotation | plan sketch |
| I | Delivery Order Step 2 list missing `prometheus/rules/ai-chat.yml` | spec §Delivery |
| J | Testing Strategy table weaker than actual success criteria for Chunks 1–3, 5, 6 | plan testing |
| K | Chunk 7 success criteria missing recording rules file from onboarding diff | plan Chunk 7 |

---

| # | Finding | Severity | When to fix | Status |
|---|---|---|---|---|
| 2.1 | OTel Collector missing `memory_limiter` and `batch` processors | **Blocker** | Before Chunk 3 | ✅ Fixed in spec + plan |
| 2.2 | Prometheus `--web.enable-remote-write-receiver` flag missing | **Blocker** | Before Chunk 3 | ✅ Fixed in spec + plan |
| 2.3 | `loki.yml` underspecified — no `limits_config`, no valid storage config | **Blocker** | Before Chunk 3 | ✅ Fixed in spec + plan |
| 2.4 | High-cardinality OTel auto-instrumentation labels (Angular URLs) | **Blocker** | Before Chunk 5 | ✅ Fixed in spec + plan |
| 3.1 | `resourcedetection` processor missing from Collector | High | Chunk 3 | ✅ Fixed in spec (added to processor chain) |
| 3.2 | Loki sink must explicitly set `propertiesAsLabels: []` | High | Chunk 1 | ✅ Fixed in spec + plan |
| 3.3 | Prometheus scrape job missing `scrape_interval`, `scheme`, `tls_config` | High | Chunk 3 | ✅ Fixed in spec + plan |
| 3.4 | No recording rules → slow Grafana dashboards at scale | Medium | Chunk 3 | ✅ Fixed in spec + plan |
| 3.5 | Blackbox probe interval unspecified; alert `for` duration unset | Medium | Chunk 3 | ✅ Fixed in spec + plan |
| 3.6 | OTel Collector sizing too small for multi-service production | Medium | Chunk 6 | ✅ Fixed in plan Chunk 6 — upgrade to 0.5 vCPU / 1 GB |
| 4.1 | TraceID/SpanID not wired into Serilog → Loki log lines | **Blocker** for correlation | Chunk 1 | ✅ Fixed in spec + plan |
| 4.2 | Loki Derived Fields not configured → no Loki→Tempo click-through | High | Chunk 3 | ✅ Fixed in spec + plan |
| 4.3 | Prometheus exemplars not enabled → no metric→trace link | Medium | Chunk 6 | ✅ Fixed in plan Chunk 6 — enable in `prometheus-aws.yml` global section |
| 4.4 | Sentry events don't carry TraceID → no Sentry→Loki/Tempo path | High | Chunk 2 | ✅ Fixed in plan |
| 4.5 | CorrelationID not propagated to Angular → no frontend→backend link | Medium | Chunk 2 | ✅ Fixed in plan |
| 5.1 | No pipeline load test before production deployment | High | Before Chunk 5 | ✅ Fixed in plan |
| 5.2 | No staged per-environment rollout sequence | Medium | Chunk 5 | ✅ Fixed in plan |
| 5.3 | Alert rules have no test procedure | High | Before Chunk 5 | ✅ Fixed in plan |
| 5.4 | EFS I/O mode unspecified for Prometheus; Loki S3 IAM not called out | Medium | Chunk 6 | ✅ Fixed in plan Chunk 6 gotchas — EFS Max I/O flagged for aws-infra-spec; Loki S3 IAM permissions called out explicitly |
| 5.5 | Static Prometheus scrape targets break under multi-instance EB ASG (counter resets, partial data) | **Blocker** | Before Chunk 5 | ✅ Fixed — EC2 SD is the official scrape strategy; `prometheus-aws.yml` uses `ec2_sd_configs` with tag-based filtering and env relabelling; Prometheus task role needs `ec2:DescribeInstances` |
| 5.6 | No visibility into monitoring stack health — silent data loss from Collector OOM or scrape failures goes undetected | High | Chunk 3 | ✅ Fixed — `platform-self` scrape job + `platform.json` meta-monitoring dashboard added to spec §1.7 and plan Chunk 3; `CollectorDroppingData` alert rule added |
