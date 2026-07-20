# Session 5 Summary

## ✅ What got built

### `specs-and-plans/aws-infra-spec.md` — created

A complete AWS infrastructure requirements specification covering:

- **Purpose and scope** — what the platform is, how signals flow, which existing resources it sits alongside
- **Architecture diagram** — ASCII showing all components, ports, trust boundaries, and EB environment connectivity
- **Open questions table** — six items that must be resolved before Infra team starts (EB application name, app port, env tag names, Cognito User Pool reuse, VPC ID, RDS decision)
- **PoC requirements** — every resource stated as a requirement with name, sizing, and justification:
  - `sg-monitoring` security group with fully enumerated inbound and outbound rules
  - IAM task execution role (`ecs-task-execution-monitoring`) and Prometheus task role (`ecs-task-role-prometheus`)
  - Production-grade Loki task role (`ecs-task-role-loki`) — specified now so it is ready when Chunk 6 needs it
  - Three Secrets Manager secrets with exact names and expected value format
  - Five ECR private repositories
  - ECS cluster `monitoring` with ECS Service Connect on `monitoring.local` namespace
  - Five ECS Fargate services (Prometheus, Loki, Grafana, Blackbox Exporter, OTel Collector) with CPU/memory, port mappings, environment variables, and secrets
  - ALB with HTTPS, Cognito User Pool authorizer (Azure AD OIDC), Grafana target group
  - EB EC2 tag requirements (`env` and `elasticbeanstalk:application-name`) with explanation of how EB propagates them automatically
- **Production-grade requirements** — EFS (with per-service access points and Max I/O note for Prometheus), S3 `wbi-loki-chunks`, RDS PostgreSQL for Grafana, Tempo ECS service — clearly separated from PoC
- **Variable strategy** — `environment` array vs `secrets` array, `#{TOKEN}#` pipeline substitution pattern
- **Pipeline agent IAM requirements** — ECR push, ECS task def registration, ECS force-deploy
- **Infra team required outputs** — complete checklist of every value needed before Chunk 5 can proceed, including production-grade outputs

### Files updated
- `specs-and-plans/implementation-plan.md` — Chunk 4 marked complete (session 5)
- `CLAUDE.md` — active work state updated to Chunk 4 complete; next-session starting point updated

---

## 🧪 Test results

Chunk 4's acceptance criterion is peer review, not an automated test suite.

**Checklist:**
- [x] Every resource has a name, a size/spec, and a one-line justification
- [x] Security group inbound rules are fully enumerated — no "open as needed"
- [x] No secret values appear in the document
- [x] PoC tier and production-grade tier are clearly separated
- [x] Open questions explicitly listed in the spec (not silently assumed)
- [x] Infra team required outputs table is complete

---

## ⚠️ Decisions made / deviations from plan

### EFS Max I/O
The implementation plan's Chunk 6 gotcha (EFS Max I/O for Prometheus) was included in the production-grade requirements section of the spec rather than deferred. This is the right time to call it out — once the Infra team provisions EFS it is harder to change the performance mode than to get it right the first time.

### `ecs-task-role-loki` provisioned at PoC tier
The plan deferred the Loki S3 task role to Chunk 6. The spec asks the Infra team to provision it at PoC tier so the task definition can reference it from day one — it is inactive until `loki.yml` switches to the S3 backend. This avoids a round-trip to the Infra team mid-Chunk 6.

### Cognito User Pool reuse flagged as open question
The spec notes this as an open question rather than mandating a new User Pool. If the Infra team's existing infrastructure already has a Cognito User Pool federated to Azure AD (common for other internal ALBs), reusing it saves ~$0/month and reduces operational surface.

### Production-grade RDS secret naming
Specified as `wbi-monitoring/grafana/db-connection` (single secret, all fields) rather than per-field secrets. This matches how other Grafana deployments typically handle it and avoids proliferating secret ARNs in the task definition.

---

## 📋 Verification steps (for Jira)

1. **Tech lead review** — read `specs-and-plans/aws-infra-spec.md`; confirm all six open questions are answered before handing to Infra team
2. **Resolve open questions** — get EB application name, .NET app port, EB environment names, VPC ID, Cognito User Pool decision from the team; fill in the Open Questions table in the spec
3. **Infra team handoff** — provide the approved spec; confirm the Infra team understands they must hand back all items in the "Required Outputs" section before Chunk 5 proceeds

---

---

## Session 5 continuation — additional changes

The following changes were made in the continuation session (context window refill after compaction):

### `OTEL_COLLECTOR_HTTP_ENDPOINT` removed everywhere

After introducing this env var for the .NET OTLP proxy destination, it was identified as unnecessary — the destination is always `http://otel-collector:4318` in every environment (Docker Compose and ECS Service Connect both resolve `otel-collector` to the same service). Removed from:

- `CLAUDE.md` — variable strategy description updated
- `docker-compose.yml` — commented-out example line removed
- `README.md` — snippet and variable table row removed
- `specs-and-plans/aws-infra-spec.md` — variable strategy table row removed; Required Outputs row updated
- `specs-and-plans/implementation-plan.md` — variable table row and Chunk 4c description updated
- `specs-and-plans/spec.md` — step 8a updated to say `appsettings.json` constant
- `specs-and-plans/runbook-chunk-4b.md` — tracker row removed; all three EB env property command lines removed

### ASCII art diagrams converted to Mermaid

All ASCII art replaced with Mermaid diagrams:

- `specs-and-plans/aws-infra-spec.md` — `graph TB` (main architecture) + `graph LR` (production-grade additions)
- `specs-and-plans/spec.md` — `graph LR` (platform overview)
- `specs-and-plans/implementation-plan.md` — `graph TB` (runtime flow) + plain code block (repo layout)

---

## 🚀 Next session starting point

**Chunk 4b — Pipeline and deploy config** (runs in parallel with Infra team provisioning)

Pre-requisites:
- Tech lead has approved `specs-and-plans/aws-infra-spec.md`
- Open questions in the spec are resolved (EB application name, app port, env tag names, VPC ID, Cognito decision)
- Infra team has started provisioning PoC resources

What to build in Chunk 4b:
1. `ecs/task-def-*.json` for all five platform services — full structure with `#{TOKEN}#` placeholders for Infra team-provided values (ECR URIs, Secrets Manager ARNs, `GF_SERVER_ROOT_URL`)
2. `DeployMonitoring` stage in the Azure DevOps pipeline YAML — build → ECR push → token substitution → task def registration → force-deploy
3. `sentry-cli releases upload-sourcemaps` step in the Angular build stage
4. EB environment property update commands (drafted with placeholders, ready to fill from Infra team outputs)
5. `prometheus-aws.yml` placeholder values identified — EB application name and app port are the only gaps after this session

See `specs-and-plans/runbook-chunk-4b.md` (already exists) for the full runbook.
