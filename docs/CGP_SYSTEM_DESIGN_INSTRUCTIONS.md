# GCP system design template

> Purpose: A pre-filled, GCP-aware template for consistent, high-quality system designs. Start by asking the intake questions and do not propose an architecture until they’re answered or assumptions are explicitly set.

---

## Intake and success criteria

### Mandatory clarification questions (do not proceed until answered)
- **Business goal:** [What outcome are we driving? Who is the primary user?]
- **Core scope:** [What must the MVP do? What is explicitly out of scope?]
- **Traffic profile:** [Peak/avg QPS, concurrent sessions, request/response sizes, write/read ratio, hotspots?]
- **Data characteristics:** [Data model shape, record counts, daily growth, retention/TTL, access patterns, consistency needs.]
- **Reliability targets:** [SLA/SLO, uptime %, P95/P99 latency targets, error budgets, maintenance windows.]
- **Geography:** [Regions, multi-region needed, data residency constraints.]
- **Compliance/security:** [PII/PHI/PCI, GDPR/LGPD/HIPAA/ISO, auditing requirements.]
- **Company standards:** [Approved GCP services, languages/frameworks, database preferences, infra patterns.]
- **Integrations:** [Internal services, third-parties, event sources/sinks, webhooks.]
- **Observability tools:** [Logging/metrics/tracing stack, alerting policy, on-call model.]
- **Deployment & ops:** [CI/CD tools, environments (dev/stage/prod), IaC (Terraform), release cadence.]
- **Cost posture:** [Budget sensitivity, cost ceilings, preferred optimization levers.]
- **Timeline & risks:** [Deadlines, migration constraints, legacy compatibility, data backfills.]

### Assumptions (if unanswered above)
- **Assumption:** [e.g., Single-region deployment in us-central1 with P95 latency under 200 ms.]
- **Assumption:** [e.g., Error budget targets align to 99.9% monthly availability.]
- **Assumption:** [e.g., Eventual consistency acceptable for secondary views.]

### Success metrics
- **SLIs/SLOs:** [Availability, latency, durability, freshness.]
- **Business KPIs:** [Conversion, engagement, throughput, cost per request.]
- **Quality gates:** [Change failure rate, MTTR, on-call toil thresholds.]

---

## Architecture overview and main parts

### High-level summary
- **One-paragraph overview:** [What the system does, who it serves, how it’s accessed.]

### Architecture sketch
- **ASCII diagram (draft):**
```
[Client/Web/Mobile] -> [API Gateway] -> [Services/GKE|Cloud Run] -> [DB/Cache] 
| | 
[AuthN/Z] [Pub/Sub] -> [Workers/Dataflow] 
| | 
[Cloud CDN] [Cloud Storage/BigQuery]
```


### Main parts
- **API/gateway:** [Protocols, versioning, rate limits, pagination, idempotency.]
- **Services/business logic:** [Service boundaries, contracts, synchronous vs. async calls.]
- **Data layer:** [Primary store choice, schema strategy, indexes, transactions.]
- **Caching:** [Where (edge/app/db), TTL/eviction, cache invalidation strategy.]
- **Messaging/events:** [Event model, topic design, ordering, retries, DLQs.]
- **File/object storage:** [Upload patterns, lifecycle, integrity (checksums).]
- **Batch/stream processing:** [ETL, windowing, late data handling, backfills.]
- **Security & access:** [AuthN method, RBAC/ABAC, secrets handling.]
- **Infrastructure & networking:** [Regions, VPC layout, subnets, NAT, egress.]
- **CI/CD & environments:** [Branching, pipelines, promotion gates, feature flags.]

---

## GCP mapping and service choices

### Service selection table
| Component | Primary choice | Alternatives | Rationale / when to use |
|---|---|---|---|
| API gateway | API Gateway | Cloud Endpoints, Load Balancing + Cloud Armor | Managed auth, usage plans, simple routing. |
| Compute (stateless) | Cloud Run | GKE, App Engine | Fast autoscale, per-request billing, container-first. |
| Compute (stateful/complex) | GKE | Compute Engine MIGs | Fine-grained orchestration, sidecars, custom networking. |
| Web/edge | Cloud CDN + HTTPS LB | Cloud Armor (WAF) | Global edge caching, DDoS protection. |
| Relational DB | Cloud SQL (Postgres/MySQL) | Spanner | Transactional workloads; Spanner for global scale/strong consistency. |
| NoSQL (document) | Firestore | Bigtable | Developer-friendly doc model; Bigtable for wide-column at very high throughput. |
| NoSQL (wide-column) | Bigtable | Firestore | Hot-key tolerant time-series/IoT at scale. |
| Global RDBMS | Spanner | — | Horizontally scalable, strong consistency across regions. |
| Caching | Memorystore (Redis) | Client in-memory | Low-latency reads, distributed locks, rate limiting. |
| Messaging | Pub/Sub | Workflows, Cloud Tasks | At-least-once delivery, fan-out, decoupling. |
| Batch/stream | Dataflow | Dataproc, Pub/Sub Lite | Managed Apache Beam; autoscaling pipelines. |
| Analytics | BigQuery | — | Serverless analytics, partitioning, columnar storage. |
| Object storage | Cloud Storage | Filestore | Durable blobs, lifecycle management, event notifications. |
| Search | Elastic on GKE/Elastic Cloud | — | Full-text search, aggregations, synonyms. |
| Monitoring | Cloud Monitoring | Prometheus on GKE | Metrics/alerts/dashboards, SLO tooling. |
| Logging/tracing | Cloud Logging + Cloud Trace | OpenTelemetry | Centralized logs, distributed tracing. |
| Security | IAM, Cloud KMS, Secret Manager | VPC SC | Least privilege, key mgmt, data perimeters. |

> Notes:
> - **Regionality:** Prefer multi-zone regional deployments; use multi-region only with clear RPO/RTO drivers.
> - **Networking:** Use separate VPCs per environment; constrain egress; use Private Service Connect for managed services.

### API and data contracts
- **API surface:** [REST/gRPC Graph; example endpoints and request/response shapes.]
- **Event schemas:** [Pub/Sub topic names, attributes, JSON schema/Avro, versioning strategy.]
- **DB schemas:** [Tables/collections, primary keys, indexes, migrations approach.]

---

## Scalability, reliability, and data management

### Capacity planning
- **Baseline load:** [Avg/peak QPS, concurrency, payload sizes, traffic diurnal patterns.]
- **Growth forecast:** [Monthly growth %, region expansion, feature roadmap impact.]
- **Hotspots:** [Keys, tenants, endpoints; mitigation via sharding/partitioning.]

### Scaling strategies
- **Stateless services:** [Autoscaling policy (CPU/RPS), concurrency settings (Cloud Run), HPA (GKE).]
- **Stateful services:** [Shard-by-key, sticky sessions avoided, leader election if required.]
- **Data stores:** [Read replicas, partitioning, TTL, query budgeting, backpressure.]

### Reliability & DR
- **Targets:** [Availability %, P95/P99 latency, error budget policy.]
- **Failure modes:** [Zone outage, dependency slowness, partial network partition, hot key.]
- **Mitigations:** [Circuit breakers, timeouts/retries with jitter, bulkheads, load shedding.]
- **Backups:** [Cloud SQL PITR, GCS versioning, Bigtable/Firestore export cadence.]
- **DR plan:** [RPO/RTO, cross-region replication (Spanner/Firestore multi-region), failover runbook.]

### Data lifecycle & governance
- **Retention:** [Per dataset; GCS lifecycle rules; BigQuery partition/cluster strategy.]
- **Privacy:** [PII classification, tokenization/pseudonymization, DLP scans.]
- **Access:** [IAM roles, service perimeters, least privilege audits.]

---

## Security, observability, and operations

### Security controls
- **AuthN/Z:** [Identity-Aware Proxy, OAuth/OIDC, service-to-service with workload identity.]
- **Network:** [Private Google Access, VPC SC for sensitive data, Cloud Armor policies.]
- **Secrets/keys:** [Secret Manager rotation policy, Cloud KMS keyrings/labels, CMEK for storage/BigQuery.]
- **Compliance:** [Audit logging scope, data residency attestations, evidence collection.]

### Observability plan
- **Metrics (SLIs):** [Availability, latency, saturation, errors; golden signals per service.]
- **Logs:** [Structured logs with request IDs; retention and exclusion filters; sampling.]
- **Tracing:** [End-to-end traces, span naming, baggage/propagation; 1–10% sampling.]
- **Dashboards:** [Per-service overviews, SLO burn rate, dependency graphs.]
- **Alerts:** [Multi-window, multi-burn-rate alerts; paging thresholds; runbook links.]

### Ops & delivery
- **Environments:** [Dev/Test/Stage/Prod with separate projects and VPCs.]
- **CI/CD:** [Build triggers, artifact registry, canary/blue-green rollouts, manual approval gates.]
- **IaC:** [Terraform modules, policy as code (OPA), drift detection.]
- **Runbooks:** [Playbooks for common incidents, escalation policy, on-call rotations.]

---

## Cost, rollout, and open items

### Cost model
- **Major drivers:** [Compute, egress, storage, BigQuery scans, Pub/Sub deliveries.]
- **Estimates:** [Per-request cost envelope, monthly forecast tiers, buffer for growth.]
- **Optimizations:** [Autoscaling min=0 (Cloud Run), committed use discounts, storage classes, query partition pruning.]

### Rollout & migration
- **Feature flags:** [Gradual exposure, kill switches.]
- **Canaries:** [Traffic splitting %, automated rollback criteria.]
- **Data migration:** [Backfill plan, dual-write/dual-read window, verification checksums.]

### Risks & open questions
- **Risk:** [Description, likelihood/impact, mitigation/owner.]
- **Open question:** [Unknowns that block design finalization.]

### Decision log
- **Decision:** [What, why, alternatives rejected, date, owner.]
- **Decision:** [Add entries as choices are finalized.]


