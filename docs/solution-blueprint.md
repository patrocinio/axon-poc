# Customer Solution Blueprint
## AWS-Native CQRS/Event Sourcing — Loan Originations Platform

**Document Owner:** epatro

**Link to Opportunity:** N/A (PoC / Technical Validation)

**Last Updated Date:** 2026-06-26

**Last Updated By:** epatro

---

## Confidentiality

This solution blueprint is considered customer confidential information. See https://console.harmony.a2z.com/engsec-docs/SA/protecting-customer-data for guidance on handling customer confidential information.

---

## Purpose of System

This system implements a **Loan Originations Platform** using CQRS (Command Query Responsibility Segregation) and Event Sourcing patterns on native AWS services. It replaces a prior Axon Framework proof-of-concept to demonstrate how the same architectural patterns can be achieved with fully managed, serverless AWS infrastructure.

**Business Problem:** Financial services institutions require systems that provide complete audit trails of all state changes for regulatory compliance, while simultaneously delivering low-latency reads for customer-facing applications. Traditional CRUD approaches lose historical state, making compliance auditing difficult and expensive.

**Success Criteria:**
- Full audit trail of every loan lifecycle event (regulatory compliance)
- Sub-100ms read latency for customer-facing queries
- Independent scaling of write and read paths
- Pay-per-use cost model (no idle infrastructure)
- Ability to add new read models (projections) without modifying the write path

**Customers:** Internal loan officers, automated risk engines, and downstream reporting systems that consume loan lifecycle events.

---

## AWS Solutioning Team

| Team Member | Role | Team | Overview of Work |
|-------------|------|------|------------------|
| epatro | SA | Account Team | Accountable SA. Designed serverless CQRS/ES architecture, built PoC, deployed to production, and produced this document. |
| donbarb | Manager | Account Team | Oversight and financial owner |

---

## Customer Contacts

| Customer | Group | Role for this system | Contact Info |
|----------|-------|---------------------|--------------|
| TBD | Engineering | Development Lead | |
| TBD | Operations | Operations Team Lead | |
| TBD | Business | Business Process Owner | |

---

## Environments

| Environment | AWS Account ID |
|-------------|---------------|
| PoC / Dev | 530195587589 |
| Production | TBD |

---

## AWS Services Consumed

| Service | Critical or Supporting | Region (us-east-1) Usage/Scale |
|---------|----------------------|-------------------------------|
| API Gateway (REST) | Critical | 1 REST API, 8 endpoints, ~100 req/s throttle |
| Lambda | Critical | 5 functions (Node.js 20, ARM64), 256MB each, ~1000 concurrent executions |
| DynamoDB | Critical | 1 table (Event Store), PAY_PER_REQUEST, Streams enabled, PITR enabled |
| Kinesis Data Streams | Critical | 1 stream, 1 shard, 24h retention |
| ElastiCache (Redis 7.x) | Critical | 1 cache.t3.micro node (PoC), single-node |
| VPC | Critical | 1 VPC, 4 subnets (2 public, 2 private), 1 NAT Gateway |
| CloudWatch | Supporting | Logs for all Lambda functions, X-Ray tracing |
| IAM | Critical | 5 Lambda execution roles (least-privilege) |
| CloudFormation (CDK) | Supporting | 4 stacks |

**KCI/KCR Risk Services:** DynamoDB (event store — data loss would be catastrophic), Lambda (compute — throttling impacts all operations).

---

## Internal-to-Customer Upstream Dependencies

| Dependency | Interface | Supporting Org | Risks or Cost Concerns |
|------------|-----------|---------------|----------------------|
| AWS Region (us-east-1) | Hosted Platform | AWS | Regional outage would impact all operations |
| Customer Identity Provider | REST/OIDC (future) | Customer IAM | Currently no auth (PoC); production requires integration |

---

## Downstream Dependencies

| Dependency | Interface | Supporting Org | Link |
|------------|-----------|---------------|------|
| Reporting/Analytics | Kinesis Data Stream | Analytics Team | Events published to Kinesis for downstream consumption |
| Audit/Compliance Systems | DynamoDB Event Store | Compliance | Direct query of immutable event log |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         WRITE PATH (Commands)                        │
│                                                                      │
│  Client ──POST──▶ API Gateway ──▶ Command Lambda                    │
│                                       │                              │
│                                       ├─ Query DynamoDB (replay)     │
│                                       ├─ Validate business rules     │
│                                       ├─ PutItem DynamoDB (event)    │
│                                       └─ PutRecord Kinesis           │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                         READ PATH (Queries)                          │
│                                                                      │
│  Client ──GET───▶ API Gateway ──▶ Query Lambda ──▶ Redis (read)     │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       PROJECTION PATH                                │
│                                                                      │
│  DynamoDB Streams ──▶ Projector Lambda ──▶ Redis (write)            │
└──────────────────────────────────────────────────────────────────────┘
```

**Component Summary:**

| Component | AWS Service | Purpose |
|-----------|-------------|---------|
| API Layer | API Gateway (REST) | HTTP endpoint routing, throttling, CORS |
| Command Processing | Lambda (TypeScript) | Business rule validation, event emission |
| Event Store | DynamoDB | Immutable, append-only event persistence |
| Event Bus | Kinesis Data Streams | Ordered event distribution to consumers |
| Read Model | ElastiCache (Redis) | Low-latency materialized views |
| Projection Engine | Lambda + DynamoDB Streams | Asynchronous read model updates |
| Infrastructure | AWS CDK (TypeScript) | Programmatic infrastructure definition |

---

## Data Flow, Classification and Compliance

### Data Flow

1. **Ingest:** Client submits loan command via HTTPS to API Gateway
2. **Validate:** Command Lambda queries DynamoDB for aggregate events, replays to current state, validates business rules
3. **Persist:** New event written to DynamoDB (Event Store) with optimistic concurrency (conditional write)
4. **Distribute:** Event published to Kinesis for downstream consumers
5. **Project:** DynamoDB Streams triggers Projector Lambda, which updates Redis read model

### Data Classification

| Data Element | Classification | Storage | Encryption |
|-------------|---------------|---------|-----------|
| Loan Events | Confidential (PII: applicant name) | DynamoDB | At-rest (AWS-managed KMS) |
| Read Model | Confidential | ElastiCache | In-transit (TLS) + At-rest |
| API Traffic | Confidential | N/A (transit) | TLS 1.2+ |

### Data Lifecycle

- **Event Store (DynamoDB):** Immutable, never deleted (regulatory requirement). PITR enabled for 35-day recovery window.
- **Read Model (Redis):** Disposable — can be rebuilt from event store at any time by replaying all events.
- **Kinesis:** 24-hour retention, events expire automatically.

### Authoritative Data Source

The **DynamoDB Event Store** is the single source of truth. Redis is a derived, eventually-consistent projection. In case of discrepancy, the event store is authoritative and the read model can be rebuilt.

---

## Performance

### Testing Performed

- **Functional Testing:** Full CQRS lifecycle validated (Submit → Approve → Disburse) with correct state transitions and business rule enforcement.
- **Concurrency:** DynamoDB conditional writes tested to ensure optimistic concurrency prevents duplicate events under concurrent command submission.
- **Read Latency:** Redis GET operations measured at <5ms p99 within VPC.

### Scaling Characteristics

| Component | Scaling Model | Limit |
|-----------|--------------|-------|
| API Gateway | Managed (10,000 req/s default) | Soft limit, can increase |
| Lambda | Concurrent executions (1000 default) | Soft limit |
| DynamoDB | On-demand (auto-scales) | 40,000 RCU/WCU burst |
| Kinesis | Per-shard (1MB/s write, 2MB/s read) | Add shards for scale |
| Redis | Vertical (node size) or cluster mode | cache.t3.micro for PoC |

### 10x Load Scenario

If usage increases 10x:
- **Lambda:** Auto-scales to concurrent execution limit; beyond that, API Gateway returns 429
- **DynamoDB:** On-demand billing auto-scales throughput
- **Kinesis:** Would require additional shards (manual or auto-scaling policy)
- **Redis:** Would require larger node type or cluster mode

---

## Capacity & Right-sizing

| Component | Current Size | Rationale |
|-----------|-------------|-----------|
| Lambda | 256MB / ARM64 | TypeScript handlers require minimal memory; ARM64 is 20% cheaper |
| DynamoDB | On-demand | PoC traffic is unpredictable; on-demand eliminates capacity planning |
| Redis | cache.t3.micro | Sufficient for PoC; production should use cache.r7g.large+ |
| Kinesis | 1 shard | PoC volumes well within 1MB/s write capacity |
| NAT Gateway | 1 (single AZ) | Cost optimization for PoC; production needs multi-AZ |

**Brownout scenario:** If Lambda concurrency is exhausted, API Gateway returns HTTP 429. The system does not lose data — commands simply queue at the client level. DynamoDB event store remains available and consistent.

---

## 1. Security Controls

| Control | Implementation |
|---------|---------------|
| IAM | Least-privilege roles per Lambda function. Command Lambda: DynamoDB read/write + Kinesis write only. Query Lambda: VPC-only (Redis access via SG). |
| Network Isolation | ElastiCache in private subnets, accessible only via Lambda security group (port 6379). No public endpoints for data stores. |
| Encryption at Rest | DynamoDB (AWS-managed KMS), ElastiCache (at-rest encryption enabled) |
| Encryption in Transit | TLS 1.2+ for API Gateway, Redis in-transit encryption enabled |
| API Authentication | None for PoC. Production: API Gateway with Cognito/IAM auth or API keys. |
| DynamoDB Access | Condition-based writes prevent unauthorized version injection |

---

## 2. Exceptions

| Exception | Rationale |
|-----------|-----------|
| No API authentication | PoC only. Production requires Cognito or IAM-based auth on API Gateway. |
| Single-AZ NAT Gateway | Cost optimization for PoC. Production requires NAT in each AZ. |
| No WAF | PoC does not require WAF. Production should enable AWS WAF on API Gateway. |

---

## 3. Lifecycle Management

| Component | Lifecycle Concern |
|-----------|-----------------|
| Lambda (Node.js 20) | Node.js 20 end-of-support: April 2026. Migrate to Node.js 22 when CDK support is GA. See https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html |
| ElastiCache (Redis 7.x) | Monitor for major version deprecation. AWS provides 18-month notice. |
| CDK | Keep CDK version current. Major versions may require migration effort. |
| DynamoDB | Fully managed, no lifecycle concerns. Monitor for any feature deprecations. |

**AWS Health Integration:** CloudWatch configured with EventBridge rules to capture AWS Health events for all services in use.

---

## 4. Logging and Monitoring

| Aspect | Implementation |
|--------|---------------|
| Logs | All Lambda functions log to CloudWatch Logs with structured JSON. Log groups: `/aws/lambda/loan-command-handler`, `/aws/lambda/loan-query-handler`, `/aws/lambda/loan-projector-handler` |
| Retention | 30 days (PoC). Production: 90 days in CloudWatch, archive to S3 for long-term. |
| Tracing | AWS X-Ray enabled on all Lambda functions and API Gateway for end-to-end request tracing. |
| Metrics | API Gateway: 4xx/5xx rates, latency p50/p99. Lambda: invocations, errors, duration, throttles. DynamoDB: consumed RCU/WCU, throttled requests. |
| Alerting | CloudWatch Alarms on: Lambda error rate > 1%, API Gateway 5xx > 0.1%, DynamoDB throttling. |
| Health Check | Dedicated `/api/loans/health` endpoint verifying Redis connectivity. |

---

## 5. Resiliency Overview

### System Classification

| Tier | Category | Rationale |
|------|----------|-----------|
| 3 | Silver (PoC) | Non-critical proof of concept. Production deployment would be Gold or Platinum depending on business criticality. |

### RTO/RPO

| Metric | Target (PoC) | Target (Production) |
|--------|-------------|-------------------|
| RTO | 4 hours | 15 minutes |
| RPO | 0 (event store is durable) | 0 |

### Failure Scenarios

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Single AZ failure | Lambda auto-routes to healthy AZ. Redis node failure causes read degradation. | Production: multi-AZ Redis, NAT in each AZ. |
| DynamoDB failure | Write path unavailable | DynamoDB is multi-AZ by default. Global tables for DR. |
| Redis failure | Reads unavailable | Read model is rebuildable from event store. Auto-failover in production (multi-AZ replication group). |
| Lambda throttling | 429 errors to clients | Request service limit increase. Implement client-side retry with exponential backoff. |

### Data Durability

- **DynamoDB:** 99.999% durability (3-AZ replication). Point-in-time recovery enabled (35-day window).
- **Events are immutable:** Once written, events cannot be modified or deleted. This protects against accidental corruption.
- **Read model is disposable:** Can be fully rebuilt by replaying all DynamoDB events through the projector.

---

## 6. Deployment Overview

| Aspect | Implementation |
|--------|---------------|
| IaC | AWS CDK (TypeScript) — single `cdk deploy --all` provisions entire stack |
| CI/CD | GitHub Actions: build → test → cdk deploy on push to `aws-native` branch |
| Rollback | CDK maintains previous stack versions. `cdk deploy` with prior commit rolls back. Lambda versioning allows instant function rollback. |
| Blue/Green | Not implemented for PoC. Production: API Gateway canary deployments. |
| Health Checks | API Gateway integration with Lambda health endpoint. Failed deploys detected via CloudWatch alarms. |

---

## 7. Cost Considerations

| Component | Monthly Cost (PoC) | Cost Considerations | Options |
|-----------|-------------------|--------------------|---------| 
| Lambda | ~$0 | Free tier covers PoC volume. ARM64 (Graviton) is 20% cheaper than x86. | If sustained high volume, consider provisioned concurrency ($$$) or Fargate. |
| DynamoDB | ~$0 | On-demand pricing. Free tier covers 25 WCU/RCU. | If predictable traffic, reserved capacity saves 50-75%. |
| Kinesis | ~$15 | 1 shard minimum charge ($0.015/hr/shard). | If event ordering per-aggregate is sufficient, consider DynamoDB Streams alone (no Kinesis cost). |
| ElastiCache | ~$13 | cache.t3.micro single-node. | Production: cache.r7g.large for performance. Consider DynamoDB as read model to eliminate Redis entirely. |
| NAT Gateway | ~$33 | $0.045/hr + $0.045/GB processed. Largest PoC cost. | Production: consider VPC endpoints for DynamoDB/Kinesis to reduce NAT traffic. |
| API Gateway | ~$0 | Free tier: 1M requests/month. | At scale: $3.50/M requests. |
| **Total** | **~$61/month** | | |

**Cost optimization for production:** Add VPC endpoints for DynamoDB and Kinesis to eliminate NAT Gateway data processing charges for those services.

---

## 8. Known Solution Gaps and Suggested Future Work

| Risk | Trigger | Impact | Mitigation or Acceptance |
|------|---------|--------|-------------------------|
| No API authentication | External actor discovers API endpoint | Unauthorized loan submissions | Production: Add Cognito or IAM auth. PoC accepted risk. |
| Single-AZ Redis | AZ failure | Read path unavailable until failover | Production: Multi-AZ replication group with automatic failover |
| Single-AZ NAT | AZ failure | VPC-attached Lambdas lose internet (Kinesis publish fails) | Production: NAT Gateway per AZ |
| No event schema versioning | Event schema evolves | Old events may not parse with new code | Implement event upcasters or schema registry |
| In-memory projection only | Lambda cold start or Redis flush | Read model temporarily empty | Implement full replay mechanism (scan all DynamoDB items → project to Redis) |
| No dead letter queue | Projector Lambda fails repeatedly | Read model falls behind event store | Add DLQ on DynamoDB Streams trigger; alert on DLQ depth |
| Kinesis single shard | Traffic exceeds 1MB/s write | Events throttled at Kinesis | Auto-scale shards or increase provisioned capacity |

### Suggested Future Work

| Work Item | Priority | Status |
|-----------|----------|--------|
| Add Cognito authentication to API Gateway | High | Aspirational |
| Multi-AZ Redis with automatic failover | High | Aspirational |
| VPC endpoints for DynamoDB + Kinesis (cost reduction) | Medium | Aspirational |
| Event replay mechanism (rebuild Redis from DynamoDB) | Medium | Aspirational |
| Dead letter queue for projector failures | Medium | Aspirational |
| CloudWatch dashboard for operational visibility | Low | Aspirational |
| WAF on API Gateway | Medium | Aspirational |
| Event schema versioning strategy | Low | Aspirational |

---

## 9. External References

| Document | Link |
|----------|------|
| Source Code (AWS-native branch) | https://github.com/patrocinio/axon-poc/tree/aws-native |
| Source Code (Axon Framework branch) | https://github.com/patrocinio/axon-poc/tree/main |
| AWS Well-Architected Serverless Lens | https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html |
| DynamoDB Best Practices | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html |
| Lambda Quotas | https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html |

---

## 10. Final Review Checklist

- [x] Well Architected Serverless Lens reviewed for alignment
- [x] Critical AWS services identified (DynamoDB, Lambda, ElastiCache, Kinesis)
- [ ] CWRA assessment completed (pending for production)
- [x] Security best practices: IAM least-privilege, encryption, network isolation
- [x] Data classification understood (PII in applicant name)
- [x] Cloud elasticity leveraged (Lambda auto-scale, DynamoDB on-demand)
- [x] Performance requirements documented and tested
- [x] Service limits identified and documented
- [x] Observability in place (CloudWatch, X-Ray, health endpoint)
- [x] Rollback-friendly deployment (CDK, Lambda versioning)
- [x] Data durability strategy (DynamoDB multi-AZ, PITR, immutable events)
- [x] Cost efficiency reviewed (ARM64, on-demand, serverless)
- [ ] AI/ML considerations (N/A for this workload)
- [x] Known gaps documented and shared

---

*Amazon Confidential*
