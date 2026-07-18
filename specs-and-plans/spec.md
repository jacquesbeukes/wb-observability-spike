# Observability Platform Technical Specification
## Angular / .NET 10 / AWS Elastic Beanstalk — July 2026

---

## Overview

This document specifies an observability platform for services built on Angular (or Vue.js), .NET 10, and AWS Elastic Beanstalk. It has two layers:

1. **Platform** — shared infrastructure (Prometheus, Loki, Grafana, OTel Collector, Blackbox Exporter) deployed once and shared across all services
2. **Service onboarding** — a repeatable pattern applied to each service to connect it to the platform

The Workbench AI Chat Middleware is the first service onboarded and serves as the POC for both layers. A successful POC produces a portable prompt (see [Service Onboarding Prompt](#service-onboarding-prompt)) that can be applied to subsequent services with minimal modification.

**Stack:** Prometheus · Loki · Grafana · Sentry · OpenTelemetry  
**Region:** `ap-southeast-2` (Sydney)  
**Repo path for platform config:** `monitoring/` (dedicated repo or top-level folder in a shared infra repo)

---

## Platform Architecture

```
[Service A: .NET + Angular]  →  OTel Collector  →  Prometheus  →  Grafana
[Service B: .NET + Vue.js ]  →  OTel Collector  →  Loki        →  Grafana
[Service N: ...]             →  OTel Collector  →  Tempo        →  Grafana (traces, production-grade)
                             Sentry (per service, cloud)        →  Grafana (JS error count panel)
Blackbox Exporter  →  Prometheus  →  Grafana (endpoint uptime)
```

Grafana is the single pane of glass. Each service gets its own dashboard and alert rules; the platform shares one Prometheus, one Loki, and one OTel Collector.

Each EB application has multiple environments (e.g. `dev`, `staging`, `production`). The `env` label is the mechanism that keeps them separated throughout the stack — in Prometheus scrape targets, Loki log labels, OTel resource attributes, and Sentry. Grafana dashboards expose `env` as a template variable so a single dashboard covers all environments.

---

## Part 1 — Platform

### 1.1 Platform Services

| Service | Purpose | AWS spec (PoC) | AWS spec (production-grade) |
|---|---|---|---|
| Prometheus | Metrics storage and query | 0.25 vCPU / 512 MB Fargate | Same + EFS for TSDB persistence |
| Loki | Log aggregation and query | 0.5 vCPU / 1 GB Fargate | Same + S3 for chunks, EFS for index |
| Grafana | Dashboards, alerting, single UI | 0.25 vCPU / 512 MB Fargate | Same + RDS PostgreSQL for state |
| Blackbox Exporter | Endpoint uptime probing | 0.125 vCPU / 256 MB Fargate | Unchanged |
| OTel Collector | Receives OTLP from all services, fans out | 0.25 vCPU / 512 MB Fargate | Unchanged |
| Grafana Tempo | Distributed trace storage | Not deployed | 0.5 vCPU / 1 GB Fargate + EFS |

All services run in the same VPC as the Elastic Beanstalk environments. No public ingress — internal traffic only.

All platform services run in a single ECS cluster (`monitoring`) with ECS Service Connect enabled. Service Connect registers each service under its short name (e.g. `prometheus`, `loki`, `otel-collector`) within a private DNS namespace, so inter-service references use the same hostnames in ECS as in Docker Compose.

Sentry is not a platform service — it is external SaaS (sentry.io) with one project per service. It is configured during service onboarding (§2.2), not here.

### 1.2 Platform Configuration Files

All files live under `monitoring/` and are the single source of truth for platform config across all environments.

```
monitoring/
  prometheus/
    prometheus.yml          # scrape jobs for local Docker Compose (static_configs)
    prometheus-aws.yml      # scrape jobs for AWS (ec2_sd_configs) — baked into Dockerfile.prometheus
    rules/
      <service-name>.yml    # recording rules per service (pre-computed series for dashboards)
  loki/
    loki.yml
  blackbox/
    blackbox.yml            # probe targets — one block per service
  otel-collector/
    otel-collector.yml
  grafana/
    provisioning/
      datasources/
        prometheus.yaml
        loki.yaml           # includes Derived Fields for Loki→Tempo trace click-through
        sentry.yaml         # Sentry API data source (org-level token via SENTRY_ORG_TOKEN env var; one data source covers all projects)
        tempo.yaml          # production-grade only
      dashboards/
        dashboards.yaml
      alerting/
        rules.yaml          # all alert rules — Grafana Unified Alerting is the single alerting system
    dashboards/             # one JSON file per service + platform.json for meta-monitoring
      <service-name>.json
      platform.json
  Dockerfile.prometheus
  Dockerfile.loki
  Dockerfile.grafana        # includes grafana-sentry-datasource plugin install
  Dockerfile.blackbox
  Dockerfile.otel-collector
  Dockerfile.tempo          # production-grade only
```

All **alerting** is managed through Grafana Unified Alerting (`grafana/provisioning/alerting/rules.yaml`) — no Alertmanager required. **Recording rules** live in `prometheus/rules/<service-name>.yml` and are evaluated by Prometheus; they pre-compute dashboard series and are not alert rules.

#### prometheus.yml — adding a new service

Prometheus uses EC2 service discovery to find all running instances for a service across all EB environments in the region. This handles multi-instance ASGs and any scaling events automatically — no static hostnames are needed and no config change is required when EB scales out or replaces instances.

Two separate config files exist:

- `monitoring/prometheus/prometheus.yml` — used by the local Docker Compose stack; uses `static_configs` pointing at the Docker Compose service name
- `monitoring/prometheus/prometheus-aws.yml` — baked into `Dockerfile.prometheus` for the AWS image; uses `ec2_sd_configs`

The `Dockerfile.prometheus` copies `prometheus-aws.yml` (not `prometheus.yml`) into the image:

```dockerfile
FROM prom/prometheus
COPY prometheus/prometheus-aws.yml /etc/prometheus/prometheus.yml
COPY prometheus/rules/ /etc/prometheus/rules/
ENTRYPOINT [ "/bin/prometheus", \
  "--config.file=/etc/prometheus/prometheus.yml", \
  "--web.enable-remote-write-receiver" ]
```

The Docker Compose volume mount continues to reference `prometheus.yml` (the local file) unchanged.

EC2 SD config block for `prometheus-aws.yml` — one job per service:

```yaml
- job_name: <service-name>
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http                 # .NET app serves HTTP; TLS terminates at the EB ALB

  ec2_sd_configs:
    - region: ap-southeast-2
      port: <app-port>         # port the .NET app listens on per EC2 instance (contractor-provided)
      filters:
        - name: tag:elasticbeanstalk:application-name
          values: [<eb-application-name>]   # contractor-provided EB application name

  relabel_configs:
    # Drop instances that are not in the running state
    - source_labels: [__meta_ec2_instance_state]
      regex: running
      action: keep
    # Propagate the `env` EC2 tag as a Prometheus label.
    # The contractor sets env=dev/staging/production on each EB environment;
    # EB environment tags propagate to all EC2 instances in that environment automatically.
    - source_labels: [__meta_ec2_tag_env]
      target_label: env
    # Set the service label
    - target_label: service
      replacement: <service-name>
```

The `env` label propagates to all Prometheus metrics, enabling per-environment filtering and alerting. Because Prometheus scrapes each EC2 instance directly (not the EB load balancer), the recording rules aggregate across instances by `job` and `env` — all existing recording rule expressions and dashboard queries work without change regardless of ASG size.

The blackbox job uses a shorter scrape interval — add it as a dedicated job in both `prometheus.yml` (local) and `prometheus-aws.yml` (AWS). The blackbox job uses `static_configs` in both files since it probes public-facing URLs, not EC2 instances directly:

```yaml
- job_name: blackbox
  scrape_interval: 30s
  scrape_timeout: 15s
  metrics_path: /probe
  params:
    module: [http_2xx]
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
  static_configs:
    - targets:
        - https://<service-dev-host>/health
        - https://<service-staging-host>/health
        - https://<service-prod-host>/health
      labels:
        service: <service-name>
```

And one probe target block in `blackbox.yml` with the health endpoint and any unauthenticated WebSocket negotiate endpoint per service.

#### prometheus/rules/<service-name>.yml — recording rules

Pre-computed series for all standard dashboard panels. Grafana queries these instead of raw data, keeping dashboard load times under 200ms regardless of TSDB size.

```yaml
groups:
  - name: <service-name>_recording
    interval: 60s
    rules:
      - record: job:http_requests_total:rate5m
        expr: >
          sum by (job, env, http_route, http_response_status_code)
            (rate(http_requests_total[5m]))

      - record: job:http_request_duration_seconds:p95
        expr: >
          histogram_quantile(0.95,
            sum by (job, env, le)
              (rate(http_request_duration_seconds_bucket[5m])))

      - record: job:http_errors:rate5m
        expr: >
          sum by (job, env)
            (rate(http_requests_total{http_response_status_code=~"5.."}[5m]))
          /
          sum by (job, env)
            (rate(http_requests_total[5m]))
```

Mount the rules directory into the Prometheus container and reference it in `prometheus.yml`:

```yaml
# prometheus-aws.yml — global section (same block applies in prometheus.yml for local)
rule_files:
  - /etc/prometheus/rules/*.yml
```

The Grafana alert rules remain in Grafana Unified Alerting (`grafana/provisioning/alerting/rules.yaml`). Recording rules are a Prometheus concern only — they pre-compute series that both Grafana dashboards and alert rules can query.

#### otel-collector.yml

Receives OTLP from all services on ports 4317 (gRPC) and 4318 (HTTP). Single Collector fans out to Prometheus and Loki. Traces pipeline added in the production-grade delivery.

The `http` receiver must include CORS configuration — Angular apps are served from a different origin than the collector, so all browser OTLP export is cross-origin.

Initial config (metrics + logs, no traces):
```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:
        cors:
          allowed_origins:
            - "http://localhost:*"        # local dev
            - "https://*.your-domain.com" # EB environments — replace with actual domain

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 400        # ~80% of the 512 MB Fargate allocation
    spike_limit_mib: 80
  batch:
    send_batch_size: 1000
    timeout: 5s
  resourcedetection:
    detectors: [env, ecs, ec2]
    timeout: 5s
    override: false
  transform/sanitize_metrics:
    metric_statements:
      - context: datapoint
        statements:
          - replace_pattern(attributes["http.url"], "\\?.*", "")

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

extensions:
  health_check:
    endpoint: 0.0.0.0:8888

service:
  extensions: [health_check]
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, transform/sanitize_metrics, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [loki]
```

The `health_check` extension exposes internal Collector metrics at `http://otel-collector:8888/metrics` — used by the Chunk 5 load test to verify the pipeline is keeping up (`otelcol_exporter_queue_size`, `otelcol_processor_batch_batch_size_trigger_send`).

`memory_limiter` must always be first in the processor chain (refuses ingestion before downstream processors fill memory). `batch` must be last before the exporter. `resourcedetection` enriches all telemetry with `cloud.region`, `aws.ecs.cluster.arn`, and related AWS attributes automatically.

The internal hostnames `prometheus` and `loki` resolve via ECS Service Connect in AWS — identical to Docker Compose service names, no environment-specific config required in this file.

Production-grade addition:
```yaml
exporters:
  otlp/tempo:
    endpoint: http://tempo:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [otlp/tempo]
```

#### loki.yml

PoC config (filesystem storage, single-process mode):
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
  max_label_names_per_series: 15
  max_label_value_length: 2048
  retention_period: 744h        # 31 days — matches the S3 lifecycle rule in production-grade
```

Production-grade addition (Chunk 6) — replace the `common.storage` and `schema_config` sections and add a compactor:
```yaml
common:
  storage:
    s3:
      bucketnames: wbi-loki-chunks
      region: ap-southeast-2
      # IAM task role provides credentials — no static keys needed

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  delete_request_store: s3
```

The `limits_config` block is identical between PoC and production-grade — do not remove it during migration.

#### grafana/provisioning/datasources/loki.yaml

The Loki datasource must include Derived Fields so Grafana auto-links `TraceId` values in log lines to the Tempo trace explorer. Add this in Chunk 3 — it is harmless before Tempo is deployed and cannot be backfilled for historical logs if added later.

```yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    access: proxy
    isDefault: false
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: '"TraceId":"([a-f0-9]{32})"'
          url: '${__value.raw}'
          datasourceUid: tempo   # must match the Tempo datasource uid field
```

`datasourceUid: tempo` references the Tempo datasource (provisioned in Chunk 6 as `grafana/provisioning/datasources/tempo.yaml` with `uid: tempo`). Grafana renders a "TraceID" link button on any log line whose body contains a `TraceId` JSON field, opening the Tempo trace explorer for that trace ID directly.

#### Custom Dockerfiles

Config is baked into images at build time. Config change = new image tag = new ECS deployment.

```dockerfile
# prometheus-aws.yml is the AWS scrape config (ec2_sd_configs).
# prometheus.yml is local Docker Compose only (static_configs) and is NOT baked into this image.
FROM prom/prometheus
COPY prometheus/prometheus-aws.yml /etc/prometheus/prometheus.yml
COPY prometheus/rules/ /etc/prometheus/rules/
# remote-write receiver is disabled by default; required for OTel Collector prometheusremotewrite exporter
ENTRYPOINT [ "/bin/prometheus", \
  "--config.file=/etc/prometheus/prometheus.yml", \
  "--web.enable-remote-write-receiver" ]

FROM grafana/loki
COPY loki/loki.yml /etc/loki/loki.yml

FROM grafana/grafana
RUN grafana-cli plugins install grafana-sentry-datasource
COPY grafana/provisioning /etc/grafana/provisioning
COPY grafana/dashboards /var/lib/grafana/dashboards

FROM prom/blackbox-exporter
COPY blackbox/blackbox.yml /etc/blackbox/blackbox.yml

FROM otel/opentelemetry-collector-contrib
COPY otel-collector/otel-collector.yml /etc/otelcol-contrib/config.yaml
```

### 1.3 Local Docker Compose

Five services added to (or alongside) the application's `docker-compose.yml`. Non-secret config is inline; secrets are read from a local `.env` file (gitignored):

```yaml
prometheus:
  image: prom/prometheus
  command:
    - --config.file=/etc/prometheus/prometheus.yml
    - --web.enable-remote-write-receiver
  volumes:
    - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    - ./monitoring/prometheus/rules:/etc/prometheus/rules
  ports: ["9090:9090"]

loki:
  image: grafana/loki
  command: -config.file=/etc/loki/loki.yml
  volumes:
    - ./monitoring/loki/loki.yml:/etc/loki/loki.yml
  ports: ["3100:3100"]

blackbox-exporter:
  image: prom/blackbox-exporter
  volumes:
    - ./monitoring/blackbox/blackbox.yml:/etc/blackbox/blackbox.yml
  ports: ["9115:9115"]

otel-collector:
  image: otel/opentelemetry-collector-contrib
  command: ["--config=/etc/otelcol-contrib/config.yaml"]
  volumes:
    - ./monitoring/otel-collector/otel-collector.yml:/etc/otelcol-contrib/config.yaml
  ports: ["4317:4317", "4318:4318"]

grafana:
  image: grafana/grafana
  volumes:
    - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
  ports: ["3000:3000"]
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    - GF_SERVER_ROOT_URL=http://localhost:3000
    - SENTRY_ORG_TOKEN=${SENTRY_ORG_TOKEN}
```

Local `.env` file (gitignored, each developer sets their own values):
```
GRAFANA_ADMIN_PASSWORD=admin
SENTRY_ORG_TOKEN=<your-sentry-org-token>
```

### 1.4 AWS Infrastructure — PoC

All resources in the same VPC as the Elastic Beanstalk environments. One ECS cluster named `monitoring` with ECS Service Connect enabled on a private DNS namespace.

| Component | AWS service | Spec |
|---|---|---|
| Prometheus | ECS Fargate task | 0.25 vCPU / 512 MB |
| Loki | ECS Fargate task | 0.5 vCPU / 1 GB |
| Grafana | ECS Fargate task | 0.25 vCPU / 512 MB |
| Blackbox Exporter | ECS Fargate task | 0.125 vCPU / 256 MB |
| OTel Collector | ECS Fargate task | 0.25 vCPU / 512 MB |
| ECR | 5 private repos | `prometheus-wbi`, `loki-wbi`, `grafana-wbi`, `blackbox-wbi`, `otel-collector-wbi` |
| ALB | Application Load Balancer | HTTPS listener; Cognito User Pool authorizer federated to Azure AD; target group → Grafana port 3000 |
| Security group | VPC | Internal only; OTLP 4317/4318 open to EB security group; Prometheus scrapes EB on service port; Grafana 3000 open to ALB security group only |

Storage is ephemeral at this tier — task restart loses data. Accepted for PoC.

Grafana is accessed via the ALB URL, authenticated through Cognito backed by Azure AD (same pattern as other internal services). Grafana trusts the ALB/Cognito layer — no Grafana-level OAuth configuration required at this stage.

**Platform-only cost: ~$44/month** (fixed regardless of how many services are onboarded)

#### ECS task definition variable strategy

Non-secret config goes in the `environment` array; secrets go in the `secrets` array referencing Secrets Manager ARNs. Values that vary per deployment use `#{TOKEN}#` pipeline substitution before task definition registration. No secret values are committed to the repo.

Key variables injected into ECS tasks:

| Variable | Task(s) | Source |
|---|---|---|
| `GF_SERVER_ROOT_URL` | Grafana | ECS task def `environment` array (ALB URL) |
| `GF_DATABASE_HOST` | Grafana (prod-grade) | ECS task def `environment` array |
| `GF_DATABASE_NAME` | Grafana (prod-grade) | ECS task def `environment` array |
| `GF_DATABASE_USER` | Grafana (prod-grade) | ECS task def `environment` array |
| `GF_DATABASE_PASSWORD` | Grafana (prod-grade) | Secrets Manager → `secrets` array |
| `SENTRY_ORG_TOKEN` | Grafana | Secrets Manager (`wbi-monitoring/grafana/sentry-token`) → `secrets` array |

### 1.5 AWS Infrastructure — Production-Grade Additions

| Component | AWS service | Purpose |
|---|---|---|
| EFS volume | EFS | Prometheus TSDB; Loki index; config mount |
| S3 bucket | S3 (`wbi-loki-chunks`, 31-day lifecycle) | Loki log chunk storage |
| RDS database | New DB on existing cluster (or new cluster) | Grafana state (dashboards, alert rules, data sources) |
| Grafana Tempo | ECS Fargate task (0.5 vCPU / 1 GB + EFS) | Distributed trace storage |
| ECR repo | `tempo-wbi` | — |

Grafana switches from SQLite (in-container) to RDS PostgreSQL. Full connection details (host, port, database name, username, password) are stored in a single Secrets Manager secret and injected as individual env vars into the Grafana ECS task (`GF_DATABASE_HOST`, `GF_DATABASE_NAME`, `GF_DATABASE_USER`, `GF_DATABASE_PASSWORD`).

**Estimated total platform cost: ~$82/month**

### 1.6 Azure DevOps Pipeline

A `DeployMonitoring` stage in the existing Azure DevOps pipeline, triggered on `main` and `release/*` branches. The stage runs when the pipeline runs — path filtering is not applied since the platform images must be consistent with the application version they are deployed alongside.

Steps:
1. Build and push each image to ECR (`Docker@2`, tagged with `$(Build.SourceVersion)`)
2. Substitute `#{TOKEN}#` variables in `ecs/task-def-<service>.json` files
3. Register ECS task definitions from `ecs/task-def-<service>.json`
4. Force-deploy each ECS service (`aws ecs update-service --force-new-deployment`)

Task definition files:
```
ecs/
  task-def-prometheus.json
  task-def-loki.json
  task-def-grafana.json
  task-def-blackbox.json
  task-def-otel-collector.json
  task-def-tempo.json          # production-grade only
```

Production-grade config change flow (EFS-based, Chunk 6):
1. Pipeline one-off ECS task mounts EFS and syncs `monitoring/` from repo
2. Pipeline force-deploys affected services
3. New tasks start with updated config from EFS

Rollback: re-run sync with the previous git commit.

### 1.7 Platform Meta-Monitoring

The monitoring platform monitors itself. A dedicated `platform-self` Prometheus scrape job collects internal metrics from each platform service, and a `platform.json` Grafana dashboard provides a single view of platform health. This is how you know if the monitoring stack itself is falling behind, running out of headroom, or losing data — without relying on an external tool to watch the watcher.

#### platform-self scrape job

Add to both `prometheus.yml` (local) and `prometheus-aws.yml` (AWS) as a static job — platform service hostnames are stable in both environments (Docker Compose service names locally; ECS Service Connect names in AWS), so EC2 SD is not needed here.

```yaml
- job_name: platform-self
  scrape_interval: 15s
  scrape_timeout: 10s
  static_configs:
    - targets: ['prometheus:9090']
      labels: { component: prometheus }
    - targets: ['loki:3100']
      labels: { component: loki }
    - targets: ['otel-collector:8888']
      labels: { component: otel-collector }
    - targets: ['blackbox-exporter:9115']
      labels: { component: blackbox-exporter }
```

In `prometheus-aws.yml` the hostnames resolve via ECS Service Connect — no changes needed between local and AWS.

#### platform.json — meta-monitoring dashboard

No `env` template variable — this dashboard is platform-wide. Build and export it in Chunk 3 alongside `ai-chat.json`.

| Panel | Metric | Tells you |
|---|---|---|
| Collector: export queue depth | `otelcol_exporter_queue_size` | Whether the Collector is keeping up; non-zero sustained = downstream pressure |
| Collector: dropped data points | `rate(otelcol_processor_dropped_metric_points[5m])` + `rate(otelcol_processor_dropped_log_records[5m])` | Whether `memory_limiter` is actively shedding data — must be zero |
| Collector: batch flush rate | `rate(otelcol_processor_batch_batch_size_trigger_send[5m])` | Normal batching throughput; sudden drop = pipeline stalled |
| Collector: memory usage % | `otelcol_process_memory_rss` vs 512 MB limit | Headroom before `memory_limiter` kicks in |
| Prometheus: scrape target health | `up{job!="platform-self"}` | Any 0 = a service scrape target is down |
| Prometheus: scrape duration by target | `scrape_duration_seconds` | Targets approaching `scrape_timeout` are a warning sign |
| Prometheus: ingest rate | `rate(prometheus_tsdb_head_samples_appended_total[5m])` | Total samples/sec flowing in; useful for right-sizing |
| Prometheus: TSDB size | `prometheus_tsdb_head_chunks` | Index growth over time; combined with ingest rate predicts when EFS will be needed |
| Loki: ingestion rate (bytes/s) | `rate(loki_distributor_bytes_received_total[5m])` | Log volume hitting Loki |
| Loki: active streams | `loki_ingester_streams_created_total` | Early warning on label cardinality creep — a sudden rise means a high-cardinality label has been introduced |
| Loki: chunk flush lag | `histogram_quantile(0.99, loki_ingester_chunk_age_seconds_bucket)` | How stale the oldest unflushed chunks are; high lag = storage pressure |
| Platform service uptime | `up{job="platform-self"}` per component | Any platform service that is down |

#### CollectorDroppingData alert rule

Add to `grafana/provisioning/alerting/rules.yaml`. This is the only alert rule scoped to the platform itself rather than a service.

```yaml
- alert: CollectorDroppingData
  expr: |
    rate(otelcol_processor_dropped_metric_points[5m]) > 0
    or
    rate(otelcol_processor_dropped_log_records[5m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "OTel Collector is dropping data"
    description: "memory_limiter is shedding telemetry. Check Collector memory usage and consider increasing Fargate allocation."
```

This fires if any data is being dropped for a sustained 5 minutes — at that point metrics and logs are silently incomplete, which is worse than an outage because dashboards appear to work.

#### `monitoring/` file tree additions

```
monitoring/
  grafana/
    dashboards/
      ai-chat.json
      platform.json        ← new
```

`prometheus.yml` and `prometheus-aws.yml` both gain the `platform-self` job block. No new Dockerfiles, no new ECS services.

---

## Part 2 — Service Onboarding Pattern

This pattern is applied to each service to connect it to the platform. It is independent of the platform deployment and can be done any time after the platform is running.

### 2.1 .NET Backend

#### Structured Logging Audit (one-time per service)

Audit all Serilog call sites. All string-interpolated log calls must be converted to message templates so structured properties become queryable Loki fields.

```csharp
// Required — properties become queryable fields in Loki
Log.Information("Request {RequestId} for {TenantId} completed", id, tenantId);

// Not allowed — logged as opaque text, not filterable
Log.Information($"Request {id} for tenant {tenantId} completed");
```

#### Correlation ID Middleware

A short ASP.NET Core middleware class (~20 lines):
- Reads `X-Correlation-ID` request header; generates a GUID if absent
- Adds to `Serilog.Context.LogContext` so every log line carries it
- Echoes in `X-Correlation-ID` response header

#### Prometheus Metrics

**Package:** `prometheus-net.AspNetCore`

```csharp
// Program.cs
app.UseHttpMetrics();
app.MapPrometheusScrapingEndpoint();  // registered outside any RequireAuthorization() policy
```

Out-of-the-box: HTTP request rate, p50/p95/p99 latency by endpoint and status code, .NET GC pause, heap size, thread pool queue depth.

The `/metrics` endpoint must not be protected by Azure AD / Entra authentication — Prometheus scrapes without a token and the VPC security group is the access boundary. If the service uses a global `RequireAuthorization()` fallback policy, `/metrics` must be explicitly excluded. If auth is applied per-route, no exclusion is needed.

Add service-specific custom metrics (gauges, counters, histograms) as appropriate for the domain.

#### Serilog Loki Sink

**Package:** `Serilog.Sinks.Grafana.Loki`

`appsettings.Development.json` (local Docker Compose):
```json
{
  "Serilog": {
    "WriteTo": [{
      "Name": "GrafanaLoki",
      "Args": {
        "uri": "http://loki:3100",
        "labels": [
          { "key": "service", "value": "<service-name>" },
          { "key": "env",     "value": "local" }
        ],
        "propertiesAsLabels": []
      }
    }]
  }
}
```

`appsettings.Production.json` (base for all EB environments):
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

`propertiesAsLabels: []` is required. Without it, the sink promotes every Serilog enriched property (`TenantId`, `CorrelationId`, `RequestId`, etc.) to a Loki stream label, creating thousands of streams per tenant and exhausting `max_streams_per_user`. All per-request properties must live in the structured log body, queryable via `| json` in LogQL — never in the stream index.

`LOKI_URI` and `ENV_NAME` are injected as EB environment properties at deploy time. `LOKI_URI` is provided by the contractor (the ECS Service Connect DNS name for Loki, e.g. `http://loki:3100`). `ENV_NAME` is `dev`, `staging`, or `production` per environment. Each EB environment's logs are separated in Loki by the `env` label and queryable independently.

#### OpenTelemetry SDK

**Packages:** `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`, `Serilog.Enrichers.OpenTelemetry`

Wire in `Program.cs` to export to the OTel Collector via OTLP. The OTLP endpoint and environment name are read from environment variables:

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService("<service-name>")
        .AddAttributes(new[] {
            new KeyValuePair<string, object>("deployment.environment",
                Environment.GetEnvironmentVariable("ENV_NAME") ?? "local")
        }))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter(o => o.Endpoint = new Uri(
            Environment.GetEnvironmentVariable("OTEL_EXPORTER_OTLP_ENDPOINT")
            ?? "http://otel-collector:4317")))
    .WithTracing(/* enabled in production-grade delivery */);
```

**Also add the `Serilog.Enrichers.OpenTelemetry` enrichers to the Serilog configuration** (alongside the Loki sink, in the same setup block):

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.WithOpenTelemetryTraceId()   // writes TraceId field into every log line
    .Enrich.WithOpenTelemetrySpanId()    // writes SpanId field into every log line
    // ... rest of Serilog config (Loki sink, Console sink, etc.)
    .CreateLogger();
```

This must be added in Chunk 1, before tracing is enabled. The `TraceId` and `SpanId` fields will be empty strings until Chunk 6 enables `WithTracing(...)` — that is expected and harmless. When Tempo is deployed, Grafana's Derived Fields on the Loki datasource will auto-link any log line to its trace in one click. If this enricher is not present from day one, the Loki→Tempo correlation link can never be backfilled for historical logs.

`OTEL_EXPORTER_OTLP_ENDPOINT` is injected as an EB environment property at deploy time (value provided by contractor). Defaults to the Docker Compose service name for local development. Initially metrics only; tracing enabled once Tempo is deployed.

### 2.2 Frontend (Angular / Vue.js)

#### Runtime Environment Config

The .NET backend serves a `window.__env` object before Angular bootstraps, sourced from its own environment variables. This means one Angular build artifact works across all environments — no per-environment build step required.

Example (Razor tag helper or middleware in the .NET app):
```html
<script>
  window.__env = {
    envName: "@System.Environment.GetEnvironmentVariable("ENV_NAME") ?? "local"",
    sentryDsn: "@System.Environment.GetEnvironmentVariable("SENTRY_DSN") ?? """,
    otlpEndpoint: "@System.Environment.GetEnvironmentVariable("OTEL_EXPORTER_OTLP_ENDPOINT") ?? "http://localhost:4318""
  };
</script>
```

`environment.ts` reads from `window.__env` with local dev fallbacks:
```typescript
export const environment = {
  envName: (window as any).__env?.envName ?? 'local',
  sentryDsn: (window as any).__env?.sentryDsn ?? '',
  otlpEndpoint: (window as any).__env?.otlpEndpoint ?? 'http://localhost:4318',
};
```

Services that already use Angular `fileReplacements` in `angular.json` may use that pattern instead and skip the `window.__env` approach.

#### Sentry Error Tracking

**Angular package:** `@sentry/angular`  
**Vue.js package:** `@sentry/vue`

```typescript
Sentry.init({
  dsn: environment.sentryDsn,
  environment: environment.envName,  // "local", "dev", "staging", "production"
  beforeSend(event) {
    // Attach active OTel trace context so Sentry errors link back to Loki/Tempo
    const ctx = trace.getActiveSpan()?.spanContext();
    if (ctx) event.tags = { ...event.tags, trace_id: ctx.traceId, span_id: ctx.spanId };
    return event;
  },
});
```

Each service gets its own Sentry project. The `environment` field maps to the EB environment name so errors are filterable by environment in sentry.io and in the Grafana Sentry panel.

`SENTRY_DSN` is injected as an EB environment property from Secrets Manager and surfaced to Angular via `window.__env.sentryDsn`.

#### OpenTelemetry JS SDK

**Packages:** `@opentelemetry/sdk-web`, `@opentelemetry/auto-instrumentations-web`

```typescript
const exporter = new OTLPMetricExporter({ url: environment.otlpEndpoint });
```

Exports metrics to the OTel Collector via OTLP HTTP. The endpoint is read from `window.__env.otlpEndpoint` (set from `OTEL_EXPORTER_OTLP_ENDPOINT` on the .NET side). The `deployment.environment` resource attribute is set from `environment.envName`. Initially metrics only; tracing enabled once Tempo is deployed. Sentry runs alongside — OTel does not replace it.

#### CorrelationId Interceptor

Add an Angular `HttpInterceptor` that reads the `X-Correlation-ID` response header from API responses and attaches it to the active Sentry scope. This links frontend Sentry errors to the corresponding backend Loki log lines by correlation ID.

```typescript
@Injectable()
export class CorrelationIdInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          const correlationId = event.headers.get('X-Correlation-ID');
          if (correlationId) Sentry.setTag('correlation_id', correlationId);
        }
      })
    );
  }
}
```

### 2.3 Platform Config Updates (per service)

When onboarding a new service, update these files in `monitoring/` and redeploy the platform images. The `platform.json` meta-monitoring dashboard (§1.7) is platform-wide and requires no changes per service onboarded.

1. **`prometheus/prometheus.yml`** (local) — add a `static_configs` scrape job for the service's Docker Compose host/port; **`prometheus/prometheus-aws.yml`** (AWS) — add an `ec2_sd_configs` job for the service filtered by its EB application name tag (see §1.2 template)
2. **`prometheus/rules/<service-name>.yml`** — add recording rules for request rate, p95 latency, error rate (copy the template from §1.2)
3. **`blackbox/blackbox.yml`** — add probe targets (health endpoint and any unauthenticated WebSocket negotiate endpoint — do not probe AD-protected endpoints)
4. **`grafana/dashboards/<service-name>.json`** — add a dashboard (see panel list below); panels must query recording rule series, not raw metrics
5. **`grafana/provisioning/alerting/rules.yaml`** — add service-specific alert rules
6. **`grafana/provisioning/datasources/sentry.yaml`** — add the service's Sentry project slug to the data source config so the JS error count panel can filter by project

#### Standard dashboard panels (per service)

All dashboards include an `env` template variable (values: `dev`, `staging`, `production`) that filters every panel. A single dashboard covers all EB environments for the service.

| Panel | Data source | Environment filter |
|---|---|---|
| HTTP request rate by endpoint | Prometheus | `env=~"$env"` label matcher |
| HTTP p95 latency by endpoint | Prometheus | `env=~"$env"` label matcher |
| HTTP error rate (5xx as % of total) | Prometheus | `env=~"$env"` label matcher |
| .NET runtime (GC, heap, thread pool) | Prometheus | `env=~"$env"` label matcher |
| Endpoint uptime (Blackbox) | Prometheus | `env=~"$env"` label matcher |
| Live log stream | Loki | `{service="<name>", env="$env"}` |
| JS error count (links to sentry.io for detail) | Sentry API | filter by `project=<sentry-project>` and `environment=$env` |

The JS error count panel shows counts in Grafana. Clicking through opens sentry.io for full error detail, stack traces, and breadcrumbs — Grafana does not replicate that.

Add service-specific panels for any custom metrics (e.g. SignalR connections, queue depths, domain-specific counters).

#### Standard alert rules (per service)

All alert rules are managed in Grafana Unified Alerting (`grafana/provisioning/alerting/rules.yaml`). Rules are scoped to `env="production"` by default. Lower environments (`dev`, `staging`) can have separate rules with relaxed thresholds if needed, but should not page on-call.

| Rule | Condition | `for` duration | Default env scope |
|---|---|---|---|
| `<Service>HighErrorRate` | HTTP error rate > 5% | `5m` | `production` |
| `<Service>HighLatency` | HTTP p95 latency > 10 seconds | `5m` | `production` |
| `<Service>EndpointDown` | Blackbox `probe_success == 0` | `2m` | `production` |
| `<Service>JSErrorSpike` | JS error count exceeds baseline threshold | `5m` | `production` |

`EndpointDown` uses `for: 2m` (two consecutive 30-second probe failures) to avoid alerting on transient blips. The other rules use `for: 5m` so a brief spike doesn't page on-call.

Add service-specific rules as appropriate (e.g. tenant error spikes, domain-specific gauge thresholds).

---

## Part 3 — POC: Workbench AI Chat Middleware

The AI Chat Middleware is the first service onboarded and validates both the platform and the onboarding pattern. It has one non-standard element: SignalR metrics.

### Service-specific custom metrics (.NET)

```csharp
var activeConns = Metrics.CreateGauge("signalr_connections_active", "Active SignalR connections");
var messagesTotal = Metrics.CreateCounter("chat_messages_total", "Total chat messages",
    new CounterConfiguration { LabelNames = new[] { "direction" } }); // "sent" / "received"
```

### Additional dashboard panels

| Panel | Data source |
|---|---|
| SignalR active connections | Prometheus |
| Chat messages throughput by direction | Prometheus |

### Additional alert rules

| Rule | Condition |
|---|---|
| `SignalRConnectionsDrop` | `signalr_connections_active` drops to zero |
| `TenantErrorSpike` | > 50 error log lines in 2 minutes from the same tenant |

### Blackbox probe targets

- `GET /health`
- `POST /hubs/aichat/negotiate`

Note: `/api/ai/conversations` is an AD-protected endpoint and is not probed — a broken auth config would produce a false-positive uptime result.

### Sentry

Project name: `ai-chat-angular`. DSN stored in Secrets Manager (`wbi-ai/chat/sentry-dsn`); injected as `SENTRY_DSN` EB environment property on all EB environments and surfaced to Angular via `window.__env.sentryDsn`. `ENV_NAME` is set per EB environment (`dev`, `staging`, `production`) and passed to `Sentry.init({ environment: environment.envName })` via `window.__env.envName`. Source maps uploaded to sentry.io per environment as part of the Angular build step in the Azure DevOps pipeline (`sentry-cli releases upload-sourcemaps`).

---

## Delivery Order

### Step 1 — Instrument the AI Chat Middleware (service onboarding)

Run the [Service Onboarding Prompt](#service-onboarding-prompt) against the AI Chat Middleware repo. The prompt covers the full application instrumentation: structured logging audit, correlation ID middleware, Prometheus metrics, Serilog Loki sink, OTel .NET SDK, Sentry, and OTel JS SDK.

For the AI Chat Middleware, also add the SignalR and chat custom metrics (see [Part 3](#part-3--poc-workbench-ai-chat-middleware)) — these are called out in the prompt's step 3 as service-specific custom metrics.

Acceptance: `GET /metrics` returns data; structured log properties (TenantId, ConversationId, CorrelationId) visible in local output; Sentry project created and DSN wired in; no string-interpolated log calls remain.

### Step 2 — Local platform setup

1. Create all `monitoring/` files for the AI Chat Middleware as the first service: `prometheus.yml` (local static scrape), `prometheus-aws.yml` (EC2 SD with placeholders for EB app name and port), `prometheus/rules/ai-chat.yml`, `loki.yml`, `blackbox.yml`, `otel-collector.yml`, Grafana provisioning (datasources with Loki Derived Fields, dashboards pointer, alert rules), `ai-chat.json` dashboard, Sentry data source
2. Write the five custom Dockerfiles
3. Add the five Docker Compose services to the application's compose file
4. Start the stack; build the `ai-chat.json` dashboard and the `platform.json` meta-monitoring dashboard in Grafana; export and commit both
5. Commit the entire `monitoring/` folder

Acceptance: full local observability loop — metrics from .NET and Angular visible in Grafana, log queries return structured results with queryable properties, Sentry errors appear in the Grafana panel, all alert rules active (including `CollectorDroppingData`); `platform.json` renders all platform-self scrape targets as UP and shows non-zero ingest rates.

### Step 3 — AWS infrastructure requirements specification

Produce `specs-and-plans/aws-infra-spec.md` specifying what must be provisioned and why, for tech lead sign-off and contractor execution. The document covers PoC and production-grade tiers, all resource names and sizes, security group rules, IAM requirements, ECS Service Connect namespace, Secrets Manager secret names, and the full list of outputs the contractor must hand back before the next step can proceed.

Acceptance: tech lead approves the document; contractor has no open questions.

### Step 4 — PoC AWS deployment

1. Contractor provisions all PoC resources per the approved spec
2. Write ECS task definition JSON files in `ecs/`; plain variables in `environment` array, secrets in `secrets` array referencing Secrets Manager ARNs; per-deployment values use `#{TOKEN}#` pipeline substitution
3. Add `DeployMonitoring` pipeline stage (build → token substitution → push → register task defs → force-deploy)
4. Set EB environment properties (`LOKI_URI`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `ENV_NAME`, `SENTRY_DSN`) on all three EB environments using contractor-provided values
5. Add source map upload step (`sentry-cli releases upload-sourcemaps`) to the Angular build in the pipeline
6. Verify Grafana dashboard `env` template variable filters metrics and logs correctly across all EB environments

Acceptance: Grafana accessible via ALB/Cognito URL; metrics and logs from each EB environment labelled and filterable by `env`; Angular OTel metrics arriving via OTel Collector; Sentry errors visible in Grafana panel filtered by environment; stack traces resolved.

### Step 5 — Production-grade storage, persistence, and tracing

1. Contractor provisions EFS, S3, RDS, Tempo ECS task per the approved spec
2. Attach EFS to Prometheus, Loki, and Grafana tasks; update task definitions
3. Update `loki.yml` to S3 backend
4. Configure Grafana to use RDS; inject connection details from Secrets Manager; migrate Grafana state from SQLite
5. Switch pipeline to EFS-sync config deployment model
6. Deploy Tempo; add ECR repo and task definition
7. Add traces pipeline to `otel-collector.yml`; redeploy Collector
8. Enable tracing in .NET OTel SDK config
9. Enable tracing in Angular OTel SDK config
10. Add Tempo data source and trace explorer panel to the dashboard

Acceptance: task restarts do not lose metrics, logs, or dashboards; end-to-end traces visible (Angular → .NET); Grafana state survives redeployment.

### Step 6 — Validate and refine the onboarding pattern

1. Document any deviations or friction encountered in Steps 1–5
2. Refine the [Service Onboarding Prompt](#service-onboarding-prompt) against actual experience
3. Identify any platform config changes needed to support a second service cleanly

---

## Service Onboarding Prompt

A self-contained prompt for applying the onboarding pattern to a new service. Paste into Claude Code at the root of the target service repo.

---

> **Context:** We run a shared observability platform (Prometheus, Loki, Grafana, OTel Collector, Blackbox Exporter) deployed on ECS Fargate in `ap-southeast-2`. Platform config lives in `monitoring/` in the infra repo. This service needs to be connected to that platform.
>
> **Service stack:** .NET 10 backend / [Angular | Vue.js] frontend / AWS Elastic Beanstalk.
>
> **Tasks — apply to this repo:**
>
> 1. **Structured logging audit** — find all Serilog `Log.*` call sites that use string interpolation (`$"..."`) and convert them to message templates (`"... {Property}"`, value as argument). Properties must be structured to be queryable in Loki.
>
> 2. **Correlation ID middleware** — add a short ASP.NET Core middleware class that reads `X-Correlation-ID` from the request header (or generates a GUID), adds it to `Serilog.Context.LogContext`, and echoes it in the response header.
>
> 3. **Prometheus metrics** — add `prometheus-net.AspNetCore`; call `app.UseHttpMetrics()` and `app.MapPrometheusScrapingEndpoint()` in `Program.cs`. Check whether the service applies `RequireAuthorization()` via a global fallback policy — if so, explicitly exclude the `/metrics` endpoint from that policy (Prometheus scrapes without a token; the VPC security group is the access boundary). Add any service-specific custom metrics (gauges, counters) relevant to this domain.
>
> 4. **Serilog Loki sink** — add `Serilog.Sinks.Grafana.Loki`; configure `appsettings.Development.json` to write to `http://loki:3100` with labels `service=<service-name>` and `env=local`; configure `appsettings.Production.json` to write to `#{LOKI_URI}#` with `env` set from `#{ENV_NAME}#`. Both values are injected as EB environment properties at deploy time (`LOKI_URI` is provided by the infrastructure contractor; `ENV_NAME` is `dev`, `staging`, or `production`). **Set `propertiesAsLabels: []` in both config files** — without it the sink promotes every Serilog property to a Loki stream label, which exhausts `max_streams_per_user` at scale.
>
> 5. **OTel .NET SDK** — add `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`, `Serilog.Enrichers.OpenTelemetry`; wire OTLP export reading the endpoint from `OTEL_EXPORTER_OTLP_ENDPOINT` env var (default `http://otel-collector:4317` for local); set the `deployment.environment` resource attribute from `ENV_NAME`. Metrics only for now. **Also add `.Enrich.WithOpenTelemetryTraceId().Enrich.WithOpenTelemetrySpanId()` to the Serilog configuration** — this writes `TraceId` and `SpanId` into every log line so Loki→Tempo click-through works once tracing is enabled; fields are empty until then, which is harmless.
>
> 6. **`window.__env` runtime config** — check whether the service already has Angular `fileReplacements` configured in `angular.json`; if yes, use those for `envName`, `sentryDsn`, and `otlpEndpoint`. If no, introduce the `window.__env` pattern: the .NET app emits a `<script>window.__env = { envName, sentryDsn, otlpEndpoint }</script>` inline in `index.html` sourced from its own environment variables (`ENV_NAME`, `SENTRY_DSN`, `OTEL_EXPORTER_OTLP_ENDPOINT`). Update `environment.ts` to read from `window.__env` with local fallbacks.
>
> 7. **Sentry** — add `@sentry/angular` (or `@sentry/vue`); initialise with `dsn: environment.sentryDsn` and `environment: environment.envName`. **Add a `beforeSend` hook that attaches the active OTel trace context as tags:** `const ctx = trace.getActiveSpan()?.spanContext(); if (ctx) event.tags = { ...event.tags, trace_id: ctx.traceId, span_id: ctx.spanId };` — this links Sentry errors back to Loki/Tempo. Add a `sentry-cli releases upload-sourcemaps` step to the Angular build in the pipeline so Sentry can resolve minified stack traces back to TypeScript source.
>
> 8. **OTel JS SDK** — add `@opentelemetry/sdk-web` and `@opentelemetry/auto-instrumentations-web`; configure OTLP HTTP export to `environment.otlpEndpoint` (from `window.__env`); set `deployment.environment` resource attribute from `environment.envName`. Metrics only for now. **Add an Angular `HttpInterceptor`** that reads `X-Correlation-ID` from API response headers and calls `Sentry.setTag('correlation_id', value)` — this links frontend Sentry errors to backend Loki log lines.
>
> 9. **Report back** the following so the platform config can be updated:
>    - The service name (to use as the Loki `service` label and Prometheus job name)
>    - The host and port the .NET app runs on in Docker Compose (for the local `prometheus.yml` static scrape target)
>    - The port the .NET app listens on per EC2 instance (for the `ec2_sd_configs` `port` field in `prometheus-aws.yml`)
>    - The EB application name (for the `elasticbeanstalk:application-name` tag filter in `prometheus-aws.yml`)
>    - Confirmation that each EB environment has an `env` EC2 tag set to `dev`, `staging`, or `production`
>    - The key unauthenticated HTTP endpoints to probe for uptime (health endpoint, any unauthenticated WebSocket negotiate endpoint — do not include AD-protected endpoints)
>    - Any custom metrics added, with their names and descriptions
>    - Any additional alert rules that make sense for this service's domain

---

## Constraints and Assumptions

- All AWS resources in `ap-southeast-2` (Sydney)
- Existing Elastic Beanstalk environments and VPC are running; RDS cluster is existing or new (TBD)
- All services in scope are Angular or Vue.js frontends on .NET 10 backends deployed to Elastic Beanstalk
- All services use Entra (Azure AD) app registrations for auth; `/metrics`, Loki push, and OTLP push endpoints are excluded from auth and rely on VPC security group boundaries
- Sentry free tier (5k errors/month per project) assumed sufficient per service; Team plan ($26/month) if exceeded
- Platform cost (~$44–82/month) is fixed and shared — per-service marginal cost is config only
- PoC-tier storage is intentionally ephemeral; accepted until production-grade delivery
- `monitoring/` is the single source of truth for all platform config
- All platform services communicate via ECS Service Connect within the `monitoring` cluster — internal hostnames match Docker Compose service names
- Grafana access is via ALB authenticated through Cognito backed by Azure AD; Grafana trusts the Cognito layer and requires no additional auth configuration
