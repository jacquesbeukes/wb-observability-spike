# Chunk 4b Runbook — Pipeline & Deploy Config
## Manual execution guide · Run in parallel with Infra team AWS provisioning

---

## Overview

This runbook authors everything needed to build and deploy the monitoring platform to AWS, using `#{TOKEN}#` placeholders for values the Infra team has not yet handed back. When the Infra team's outputs arrive you fill in the placeholders (Phase 2) and Chunk 5 can start immediately.

**Pre-condition:** Chunk 4 (`specs-and-plans/aws-infra-spec.md`) is approved and the Infra team has started.  
**Blocks:** Chunk 5.  
**Est. time:** Phase 1 ~2–3 hours · Phase 2 ~30 minutes (once Infra team outputs arrive).

---

## Infra team outputs tracker

Fill this in as values arrive. Every `#{TOKEN}#` in the files below maps to a row here.

| Output | Token | Value | Received |
|---|---|---|---|
| AWS account ID | `#{AWS_ACCOUNT_ID}#` | | ☐ |
| ECR registry base URI | `#{ECR_REGISTRY}#` | e.g. `123456789012.dkr.ecr.ap-southeast-2.amazonaws.com` | ☐ |
| ECS cluster name | `#{ECS_CLUSTER}#` | `monitoring` (confirm) | ☐ |
| ECS cluster ARN | `#{ECS_CLUSTER_ARN}#` | | ☐ |
| ECS task execution role ARN | `#{ECS_EXECUTION_ROLE_ARN}#` | | ☐ |
| Prometheus task role ARN (needs `ec2:DescribeInstances`) | `#{PROMETHEUS_TASK_ROLE_ARN}#` | | ☐ |
| VPC ID | `#{VPC_ID}#` | | ☐ |
| Private subnet IDs | `#{SUBNET_IDS}#` | comma-separated | ☐ |
| Monitoring security group ID | `#{MONITORING_SG_ID}#` | | ☐ |
| Loki internal URI | `#{LOKI_URI}#` | e.g. `http://loki:3100` (ECS Service Connect) | ☐ |
| OTel Collector gRPC internal URI | `#{OTEL_GRPC_URI}#` | e.g. `http://otel-collector:4317` | ☐ |
| Grafana ALB URL | `#{GRAFANA_ALB_URL}#` | e.g. `https://grafana.internal.wbi.com` | ☐ |
| Sentry org token Secrets Manager ARN | `#{SENTRY_ORG_TOKEN_ARN}#` | `arn:aws:secretsmanager:ap-southeast-2:...:secret:wbi-monitoring/grafana/sentry-token-...` | ☐ |
| Sentry DSN Secrets Manager ARN | `#{SENTRY_DSN_ARN}#` | `arn:aws:secretsmanager:ap-southeast-2:...:secret:wbi-ai/chat/sentry-dsn-...` | ☐ |
| EB application name | `#{EB_APP_NAME}#` | | ☐ |
| AI Chat Middleware app port (per EC2 instance) | `#{APP_PORT}#` | | ☐ |
| EC2 `env` tag confirmation | — | Infra team confirms `env=dev/staging/production` set on each EB environment | ☐ |

---

## Phase 1 — Author files with placeholders

Complete this phase while the Infra team is working. No AWS access needed.

---

### Step 1 — Create the `ecs/` directory structure

```
ecs/
  task-def-prometheus.json
  task-def-loki.json
  task-def-grafana.json
  task-def-blackbox.json
  task-def-otel-collector.json
```

---

### Step 2 — Author ECS task definition files

Create each file below. The `#{TOKEN}#` values are filled in during Phase 2.

> **Pattern:** `environment` array for plain config values. `secrets` array for Secrets Manager references (value = ARN, not the secret value itself). `#{TOKEN}#` tokens are substituted by the pipeline before task definition registration.

#### `ecs/task-def-prometheus.json`

```json
{
  "family": "monitoring-prometheus",
  "taskRoleArn": "#{PROMETHEUS_TASK_ROLE_ARN}#",
  "executionRoleArn": "#{ECS_EXECUTION_ROLE_ARN}#",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "prometheus",
      "image": "#{ECR_REGISTRY}#/prometheus-wbi:#{Build.SourceVersion}#",
      "portMappings": [
        { "containerPort": 9090, "name": "prometheus", "appProtocol": "http" }
      ],
      "command": [
        "--config.file=/etc/prometheus/prometheus.yml",
        "--web.enable-remote-write-receiver"
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/monitoring/prometheus",
          "awslogs-region": "ap-southeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

Note: Prometheus has a dedicated task role (not just execution role) for `ec2:DescribeInstances`.

#### `ecs/task-def-loki.json`

```json
{
  "family": "monitoring-loki",
  "executionRoleArn": "#{ECS_EXECUTION_ROLE_ARN}#",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "loki",
      "image": "#{ECR_REGISTRY}#/loki-wbi:#{Build.SourceVersion}#",
      "portMappings": [
        { "containerPort": 3100, "name": "loki", "appProtocol": "http" }
      ],
      "command": ["-config.file=/etc/loki/loki.yml"],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/monitoring/loki",
          "awslogs-region": "ap-southeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### `ecs/task-def-grafana.json`

```json
{
  "family": "monitoring-grafana",
  "executionRoleArn": "#{ECS_EXECUTION_ROLE_ARN}#",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "grafana",
      "image": "#{ECR_REGISTRY}#/grafana-wbi:#{Build.SourceVersion}#",
      "portMappings": [
        { "containerPort": 3000, "name": "grafana", "appProtocol": "http" }
      ],
      "environment": [
        { "name": "GF_SERVER_ROOT_URL", "value": "#{GRAFANA_ALB_URL}#" }
      ],
      "secrets": [
        {
          "name": "SENTRY_ORG_TOKEN",
          "valueFrom": "#{SENTRY_ORG_TOKEN_ARN}#"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/monitoring/grafana",
          "awslogs-region": "ap-southeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### `ecs/task-def-blackbox.json`

```json
{
  "family": "monitoring-blackbox",
  "executionRoleArn": "#{ECS_EXECUTION_ROLE_ARN}#",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "128",
  "memory": "256",
  "containerDefinitions": [
    {
      "name": "blackbox",
      "image": "#{ECR_REGISTRY}#/blackbox-wbi:#{Build.SourceVersion}#",
      "portMappings": [
        { "containerPort": 9115, "name": "blackbox-exporter", "appProtocol": "http" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/monitoring/blackbox",
          "awslogs-region": "ap-southeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

#### `ecs/task-def-otel-collector.json`

```json
{
  "family": "monitoring-otel-collector",
  "executionRoleArn": "#{ECS_EXECUTION_ROLE_ARN}#",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "otel-collector",
      "image": "#{ECR_REGISTRY}#/otel-collector-wbi:#{Build.SourceVersion}#",
      "portMappings": [
        { "containerPort": 4317, "name": "otel-collector-grpc", "appProtocol": "grpc" },
        { "containerPort": 4318, "name": "otel-collector-http", "appProtocol": "http" },
        { "containerPort": 8888, "name": "otel-collector-metrics", "appProtocol": "http" }
      ],
      "command": ["--config=/etc/otelcol-contrib/config.yaml"],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/monitoring/otel-collector",
          "awslogs-region": "ap-southeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

### Step 3 — Add `DeployMonitoring` stage to the Azure DevOps pipeline

Add the following stage to the existing pipeline YAML file (after the existing build/test/deploy stages). Adjust the `dependsOn` value to match the name of whatever stage currently runs last.

```yaml
- stage: DeployMonitoring
  displayName: Deploy Monitoring Platform
  dependsOn: <YourLastExistingStage>
  condition: and(succeeded(), or(eq(variables['Build.SourceBranch'], 'refs/heads/main'), startsWith(variables['Build.SourceBranch'], 'refs/heads/release/')))
  jobs:
    - job: Deploy
      displayName: Build, push, and deploy monitoring images
      pool:
        vmImage: ubuntu-latest
      steps:

        # --- ECR login ---
        - task: AWSShellScript@1
          displayName: ECR login
          inputs:
            awsCredentials: <your-aws-service-connection-name>
            regionName: ap-southeast-2
            scriptType: inline
            inlineScript: |
              aws ecr get-login-password --region ap-southeast-2 \
                | docker login --username AWS --password-stdin #{ECR_REGISTRY}#

        # --- Build and push each image ---
        - task: Docker@2
          displayName: Build and push prometheus
          inputs:
            command: buildAndPush
            dockerfile: monitoring/Dockerfile.prometheus
            buildContext: monitoring
            repository: #{ECR_REGISTRY}#/prometheus-wbi
            tags: $(Build.SourceVersion)

        - task: Docker@2
          displayName: Build and push loki
          inputs:
            command: buildAndPush
            dockerfile: monitoring/Dockerfile.loki
            buildContext: monitoring
            repository: #{ECR_REGISTRY}#/loki-wbi
            tags: $(Build.SourceVersion)

        - task: Docker@2
          displayName: Build and push grafana
          inputs:
            command: buildAndPush
            dockerfile: monitoring/Dockerfile.grafana
            buildContext: monitoring
            repository: #{ECR_REGISTRY}#/grafana-wbi
            tags: $(Build.SourceVersion)

        - task: Docker@2
          displayName: Build and push blackbox
          inputs:
            command: buildAndPush
            dockerfile: monitoring/Dockerfile.blackbox
            buildContext: monitoring
            repository: #{ECR_REGISTRY}#/blackbox-wbi
            tags: $(Build.SourceVersion)

        - task: Docker@2
          displayName: Build and push otel-collector
          inputs:
            command: buildAndPush
            dockerfile: monitoring/Dockerfile.otel-collector
            buildContext: monitoring
            repository: #{ECR_REGISTRY}#/otel-collector-wbi
            tags: $(Build.SourceVersion)

        # --- Token substitution in task definitions ---
        - task: qetza.replacetokens.replacetokens-task.replacetokens@6
          displayName: Substitute tokens in task definitions
          inputs:
            targetFiles: ecs/task-def-*.json
            tokenPattern: default        # matches #{TOKEN}#

        # --- Register task definitions and force-deploy ---
        - task: AWSShellScript@1
          displayName: Register task defs and deploy
          inputs:
            awsCredentials: <your-aws-service-connection-name>
            regionName: ap-southeast-2
            scriptType: inline
            inlineScript: |
              set -e
              for service in prometheus loki grafana blackbox otel-collector; do
                echo "Registering task def: monitoring-${service}"
                aws ecs register-task-definition \
                  --cli-input-json file://ecs/task-def-${service}.json \
                  --region ap-southeast-2

                echo "Force-deploying ECS service: ${service}"
                aws ecs update-service \
                  --cluster #{ECS_CLUSTER}# \
                  --service monitoring-${service} \
                  --task-definition monitoring-${service} \
                  --force-new-deployment \
                  --region ap-southeast-2
              done
```

**Things to confirm before Chunk 5:**
- Replace `<your-aws-service-connection-name>` with the actual Azure DevOps AWS service connection name
- Replace `<YourLastExistingStage>` with the correct `dependsOn` value
- Confirm the `replacetokens@6` task is available in the organisation; if not, use the version that is installed
- Confirm the pipeline agent's AWS service connection has these IAM permissions: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:DescribeServices`

---

### Step 4 — Add source map upload to the Angular build stage

Find the existing Angular build step in the pipeline and add a `sentry-cli` upload step immediately after it:

```yaml
- script: |
    npx sentry-cli releases new "$(Build.SourceVersion)"
    npx sentry-cli releases files "$(Build.SourceVersion)" \
      upload-sourcemaps ./dist/ai-chat --rewrite
    npx sentry-cli releases finalize "$(Build.SourceVersion)"
  displayName: Upload source maps to Sentry
  env:
    SENTRY_AUTH_TOKEN: $(SENTRY_AUTH_TOKEN)   # set as a secret pipeline variable
    SENTRY_ORG: <your-sentry-org-slug>
    SENTRY_PROJECT: ai-chat-angular
```

**Pre-condition:** `SENTRY_AUTH_TOKEN` must be added as a secret pipeline variable (not in code). Obtain the token from sentry.io → Settings → Auth Tokens.

---

### Step 5 — Draft EB environment property commands

These commands are ready to run the moment Infra team outputs arrive. Fill in placeholders from the tracker table above.

```bash
# Run once per EB environment. Substitute values from the Infra team outputs tracker.

# dev
aws elasticbeanstalk update-environment \
  --region ap-southeast-2 \
  --environment-name <your-dev-env-name> \
  --option-settings \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=ENV_NAME,Value=dev \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=LOKI_URI,Value=#{LOKI_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=OTEL_EXPORTER_OTLP_ENDPOINT,Value=#{OTEL_GRPC_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=SENTRY_DSN,Value=<from-secrets-manager-or-paste-directly>

# staging
aws elasticbeanstalk update-environment \
  --region ap-southeast-2 \
  --environment-name <your-staging-env-name> \
  --option-settings \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=ENV_NAME,Value=staging \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=LOKI_URI,Value=#{LOKI_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=OTEL_EXPORTER_OTLP_ENDPOINT,Value=#{OTEL_GRPC_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=SENTRY_DSN,Value=<from-secrets-manager-or-paste-directly>

# production
aws elasticbeanstalk update-environment \
  --region ap-southeast-2 \
  --environment-name <your-production-env-name> \
  --option-settings \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=ENV_NAME,Value=production \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=LOKI_URI,Value=#{LOKI_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=OTEL_EXPORTER_OTLP_ENDPOINT,Value=#{OTEL_GRPC_URI}# \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=SENTRY_DSN,Value=<from-secrets-manager-or-paste-directly>
```

> `SENTRY_DSN` can be set as a plain EB environment property pointing at the Secrets Manager ARN if the EB instance role has `secretsmanager:GetSecretValue`, or pasted directly if you prefer to manage it outside Secrets Manager. The Infra team's output for `#{SENTRY_DSN_ARN}#` covers the Secrets Manager path.

> Run `dev` first. Wait for the environment to update (`aws elasticbeanstalk describe-environments --environment-names <dev-env-name>` shows `Status: Ready`). Confirm data appears in Grafana before running `staging`, then `production`. Leave a 15-minute gap between each.

---

### Step 6 — Prepare `prometheus-aws.yml` for completion

Open `monitoring/prometheus/prometheus-aws.yml`. Two values need filling in once Infra team outputs arrive:

| Placeholder | Token | Source |
|---|---|---|
| EB application name | `#{EB_APP_NAME}#` | Infra team output |
| Per-service app port | `#{APP_PORT}#` | Infra team output |

These are the only remaining gaps in that file after Chunk 3 creates it. No other config in `prometheus-aws.yml` depends on Infra team outputs — all other values (region, tag names, relabel rules) are known now.

---

## Phase 2 — Fill in placeholders (run when Infra team outputs arrive)

Work through the Infra team outputs tracker table at the top of this runbook. For each received value:

1. **Task definitions** (`ecs/task-def-*.json`) — do a global search-and-replace for each `#{TOKEN}#`. After replacing all tokens, verify no `#{` remains in any file: `grep -r '#{' ecs/`

2. **Pipeline YAML** — replace `#{ECR_REGISTRY}#` and `#{ECS_CLUSTER}#`; replace the two `<placeholder>` values for service connection name and `dependsOn`

3. **`prometheus-aws.yml`** — fill in `#{EB_APP_NAME}#` and `#{APP_PORT}#`

4. **EB env property commands** (Step 5 above) — fill in all `#{...}#` tokens and the three `<env-name>` values

5. **Verify** — run `grep -r '#{' ecs/ monitoring/prometheus/prometheus-aws.yml` and confirm zero matches

Once Phase 2 is complete, open Chunk 5 and start the pre-deployment gate.

---

## Phase 2 completion checklist

- [ ] All `#{TOKEN}#` placeholders resolved — `grep -r '#{' ecs/` returns nothing
- [ ] `prometheus-aws.yml` has real EB application name and app port
- [ ] Pipeline YAML has real service connection name and `dependsOn` value
- [ ] CloudWatch log groups exist for all five services: `/ecs/monitoring/prometheus`, `/ecs/monitoring/loki`, `/ecs/monitoring/grafana`, `/ecs/monitoring/blackbox`, `/ecs/monitoring/otel-collector` — create manually if the Infra team has not done so
- [ ] `SENTRY_AUTH_TOKEN` set as a secret pipeline variable in Azure DevOps
- [ ] EB environment names confirmed for dev, staging, production
- [ ] EC2 `env` tags confirmed on each EB environment (Infra team output)
- [ ] Pipeline runs at least one dry run — `DeployMonitoring` stage visible and YAML-valid in Azure DevOps
- [ ] Infra team outputs tracker table fully ticked off
