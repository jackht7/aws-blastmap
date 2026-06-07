---
name: blastmap-web-app
description: >
  Pre-computed blast radius intelligence for web-app. Use when investigating
  incidents on AppDatabase, Stripe Payment API, TasksTable, NotificationQueue,
  or ApiGateway — or symptoms matching their failure modes (connection errors /
  500s, checkout failures, data loss, notifications silently not being
  delivered, or total app-wide outage).
agent-types: [incident, prevention, sre]
version: be1a779
---

# BlastMap Intelligence: web-app

## Resource Risk Registry

| Resource | Type | BlastRadiusScore | SPOFStatus | Top Downstream |
|----------|------|-----------------|------------|----------------|
| AppDatabase | RDS PostgreSQL (single-AZ) | 9/10 | YES | TaskLambda, all authenticated requests |
| Stripe Payment API | EXTERNAL (third-party) | 8/10 | YES (EXTERNAL) | TaskLambda, checkout flow |
| ApiGateway | REST API | 7/10 | NO (AWS-managed HA) | ApiStage, TaskLambda, end users |
| TasksTable | DynamoDB (no PITR) | 6/10 | YES | TaskLambda, task CRUD operations |
| ApiStage | API Gateway Stage | 6/10 | NO (AWS-managed HA) | end users (prod entry point) |
| NotificationQueue | SQS (no DLQ) | 5/10 | YES | NotifyLambda, user notifications |
| TaskLambda | Lambda function | 4/10 | NO (stateless, auto-scales) | ApiGateway, end users |
| NotifyLambda | Lambda function | 3/10 | NO (stateless, auto-scales) | NotificationQueue consumers |
| AssetsBucket | S3 Bucket | 2/10 | NO (AWS-managed HA) | End users |

## Known Cascade Paths

1. **Cascade A — AppDatabase failure**: RDS instance becomes unavailable → TaskLambda connection pool exhausts within 3s → all task CRUD endpoints return 500 via API Gateway → users see "Unable to load tasks" app-wide within 90s. MTTR without runbook: 40–70 min. MTTR with SKILL.md: 10–15 min.

2. **Cascade B — Stripe Payment API outage (EXTERNAL SPOF)**: Stripe API begins erroring or timing out → TaskLambda checkout calls fail fast (connection-refused, ~2s) or hang toward the 15s function timeout → checkout fails for every user within 8–25s, with no circuit breaker or fallback path → revenue-impacting outage within 5 minutes. MTTR without runbook: 30–60 min (bounded by Stripe's own recovery). MTTR with SKILL.md: 5–10 min to isolate and communicate.

3. **Cascade C — ApiGateway failure**: ApiGateway/ApiStage becomes unavailable (regional event or stage misconfiguration) → ApiStage starts returning 5xx/timeouts within 10s → TaskLambda invocation count drops to near-zero as no traffic reaches it → users see "Cannot connect to server" / blank app screen app-wide within 60s, a total outage rather than a degraded feature. MTTR without runbook: 30–50 min (must distinguish AWS regional issue from account misconfiguration). MTTR with SKILL.md: 8–12 min (go straight to AWS Service Health Dashboard + stage redeploy).

## Investigation Shortcuts

When alert fires on **AppDatabase**:
  - Immediately check: TaskLambda error rate and connection pool metrics, RDS CloudWatch (DatabaseConnections, FreeStorageSpace, ReplicaLag), RDS event log
  - Likely cascade: Cascade A — DB down → Lambda connection errors → 500s → app-wide outage
  - First mitigation action: Check RDS instance status in console; if AZ failure, initiate manual failover or restore from latest snapshot (enabling Multi-AZ would automate this)
  - Escalate if: RDS shows "incompatible-restore" status or free storage is near 0 bytes
  - Expected MTTR: 10–15 minutes with this skill loaded

When alert fires on **Stripe Payment API / checkout failures**:
  - Immediately check: TaskLambda X-Ray traces for the `PAYMENT_API_URL` segment (latency and error rate), the Stripe status page, Lambda timeout/error metrics
  - Likely cascade: Cascade B — external API outage → Lambda fails fast or hangs → checkout fails app-wide
  - First mitigation action: Post a customer-facing banner; there is no internal fallback — track Stripe's status page for an ETA
  - Escalate if: outage exceeds 30 minutes (revenue-impact threshold)
  - Expected MTTR: 5–10 minutes to isolate and communicate; full resolution depends on Stripe's recovery

When alert fires on **ApiGateway / total outage ("cannot connect")**:
  - Immediately check: AWS Service Health Dashboard for the region, ApiStage CloudWatch metrics (5XXError, Latency, IntegrationLatency), recent stage/deployment changes
  - Likely cascade: Cascade C — gateway/stage down → zero traffic reaches TaskLambda → app-wide connection errors
  - First mitigation action: If AWS-side, communicate ETA from Service Health Dashboard; if account-side, redeploy the last known-good stage configuration
  - Escalate if: TaskLambda invocation count is near zero while ApiStage 5XXError is spiking (confirms gateway-layer cause, not Lambda-layer)
  - Expected MTTR: 8–12 minutes with this skill loaded

When alert fires on **TasksTable / missing or corrupted task data**:
  - Immediately check: DynamoDB CloudWatch (UserErrors, SystemErrors, ConsumedWriteCapacityUnits), TaskLambda write-path logs around the time of the reported loss, CloudTrail for recent DeleteItem/UpdateItem calls
  - Likely cascade: silent data-loss path — bad write or delete with no point-in-time recovery to fall back on
  - First mitigation action: Stop further writes to the affected items if possible; there is no PITR snapshot to restore from today — capture a manual export immediately to prevent further drift
  - Escalate if: the affected rows are user-facing (visible data loss) rather than internal/derivable
  - Expected MTTR: hours (recovery is manual reconstruction, not restore — enabling PITR would change this)

When alert fires on **NotificationQueue / "users not receiving notifications"**:
  - Immediately check: NotifyLambda error logs and invocation metrics, SQS `ApproximateNumberOfMessagesVisible` vs `NumberOfMessagesDeleted` without a matching successful Lambda invocation (the silent-failure signal)
  - Likely cascade: processing failures plus no DLQ equals silent message loss
  - First mitigation action: Attach a DLQ via console/CLI immediately to stop further loss, then replay any messages still in flight
  - Escalate if: message loss has been ongoing for several hours (backlog likely unrecoverable)
  - Expected MTTR: 15–20 minutes once investigation starts — detection, not mitigation, is the bottleneck

## Hardening Status

| Change | Priority | AWS Service | Blast Reduction | Status |
|--------|----------|-------------|-----------------|--------|
| Enable RDS Multi-AZ | 1 | AWS::RDS::DBInstance → MultiAZ: true | 9/10 → 4/10 | PENDING |
| Add DLQ + alarm to NotificationQueue | 1 | AWS::SQS::Queue RedrivePolicy + CloudWatch Alarm | 5/10 → 2/10 | PENDING |
| Add circuit breaker + retry-with-backoff around Stripe calls | 1 | Lambda code change or Step Functions | 8/10 → 5/10 | PENDING |
| Enable PITR on TasksTable | 2 | PointInTimeRecoverySpecification: PointInTimeRecoveryEnabled: true | data loss risk reduced | PENDING |
| Add RDS read replica | 2 | AWS::RDS::DBInstanceReadReplica | 9/10 → 6/10 (additional) | PENDING |
| Move checkout to async queue with cached pricing | 3 | SQS + async processing pattern | 8/10 → 6/10 (additional) | PENDING |

## Deployment Notes

- Last analysed: 2026-06-07
- Architecture version: be1a779-web-app.yaml
- Next review trigger: any resource addition or change to AppDatabase, Stripe Payment API integration, TasksTable, NotificationQueue, or ApiGateway

## Agent Findings — Suggested Improvements

[DevOps Agent: after investigating an incident with this skill, append an entry]
  - Date: [investigation date]
  - Resource: [resource investigated]
  - Predicted vs observed: [did the cascade path, timing, or MTTR match Step 3's prediction?]
  - Suggested change: [new dependency found, score to adjust, new SPOF, etc.]

A user reviews these entries and feeds them into the next BlastMap run to keep
the dependency graph and predictions accurate.
