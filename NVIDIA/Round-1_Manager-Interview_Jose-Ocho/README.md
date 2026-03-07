# NVIDIA Interview — Round 1: Manager Screen

## Interview Details

| Field         | Info                                                                 |
|---------------|----------------------------------------------------------------------|
| **Date**      | February 12, 2026                                                    |
| **Duration**  | 1 hour 27 minutes                                                    |
| **Interviewer** | Jose Ocho — Manager, Platform Infrastructure / DevOps / SRE, NVIDIA E-Commerce (Marketplace) Team |
| **Role**      | Senior DevOps Engineer — Individual Contributor (Remote)            |
| **Team Size** | 5 currently → 6 with this hire (potentially 7 with a parallel open role) |
| **Outcome**   | **Moved Forward** — Jose confirmed advancing to the next round      |

---

## About the Role

NVIDIA's Marketplace team (~50 people) operates similarly to Amazon's marketplace — selling NVIDIA products directly and through partners (Asus, Micro Center, etc.). Jose's sub-team of 5 handles all **platform infrastructure, DevOps, and SRE** functions.

**Key needs Jose highlighted:**
- Strong AWS across the stack
- Heavy Terraform experience (currently no IaC — pure click-ops)
- Jenkins / CICD background
- SLO/SLA ownership and alerting/monitoring (SRE mindset)
- EKS implementation and management

---

## Topics Covered & Answers

### 1. AWS — Multi-Tier VPC Architecture
- Described 3-tier VPC: Public (ALB), Private (EKS nodes), Data (RDS/ElastiCache)
- NAT Gateway per AZ for outbound, VPC Gateway Endpoints for S3/DynamoDB, Interface Endpoints for SSM/Secrets Manager/SQS
- Discussed CloudFront → ALB → EKS ingress pattern; also mentioned API Gateway + VPC Link, Istio ingress gateway as alternatives
- Jose noted they use **Akamai** (not CloudFront) as CDN — small nuance, but answer was accepted

### 2. AWS — Route 53 Cross-Region Failover
- Active-passive setup: US-West-2 (primary) + US-East-2 (secondary)
- Health checks on ALB endpoints, failover routing policy, low TTL (~60s)
- Also mentioned latency-based routing with health checks for smarter routing
- S3 failover: Cross-region replication + CloudFront origin groups for automatic failover on 5xx errors

### 3. AWS — Lambda Cold Starts
- Provisioned concurrency + Application Auto Scaling by schedule/utilization
- Runtime optimization: small packages, shared layers, SDK clients outside handler
- Preferred Python/Node.js for latency-sensitive APIs; SnapStart for Java

### 4. AWS — RDS Connection Pooling with Lambda
- Correctly identified **RDS Proxy** as the solution
- Explained multiplexing (1000 lambdas → ~50–100 real DB connections)
- Also noted RDS Proxy handles failover reconnection transparently

### 5. Terraform — Migration from Click-Ops (0 to IaC)
- Start with Dev environment, not Production
- Inventory audit → prioritize frequently-changed resources
- Modular structure from day one, separate state per environment
- Remote state in S3 + DynamoDB locking immediately
- Use **Terraformer** (open source) or **AWS Former2** to generate code from existing resources
- Covered real AT&T experience migrating 200+ services

### 6. Terraform — State Management & Corruption Recovery
- S3 versioning is non-negotiable as state backup
- Break state into logical units (networking, EKS, apps separately) to reduce blast radius
- Real incident: pipeline killed mid-apply → restored previous S3 version → `terraform refresh` → `terraform plan`

### 7. Terraform — Drift Detection
- Scheduled nightly `terraform plan` in pipeline → output to Slack/PagerDuty
- AWS Config rules for real-time drift detection
- Two paths: revert manual change via `terraform apply`, or update code if change was valid

### 8. Jenkins / CICD — Pipeline Design
- Build → Security Scan → Dev → Stage → Prod (manual approval gate)
- Blue/Green: Two target groups behind ALB, deploy to inactive, health check, flip listener
- Canary: Weighted target groups (5% → 25% → 50% → 100%), monitor Datadog error rates/latency at each stage
- TerraForm owns infrastructure scaffolding; Jenkins owns deployment orchestration

### 9. EKS — Secrets Management
- AWS Secrets Manager + **Secrets Store CSI Driver** (IRSA for pod-level IAM)
- Avoids base64 Kubernetes native secrets
- Configurable rotation poll interval (30s–2min) for secret freshness
- Acknowledged trade-off: CSI driver caches at pod startup (zero API calls per request) vs. SDK runtime fetch (fresh but costly and adds latency)

### 10. EKS — Scaling
- **HPA**: CPU/memory or custom metrics (req/sec), scales pod replicas
- **KEDA**: Event-driven scaling (SQS queue depth, etc.)
- **Cluster Autoscaler vs. Karpenter**: Karpenter is faster (<1 min node provisioning), right-sizes instances based on pending pod specs, supports diverse instance types
- **VPA**: Adjusts CPU/memory requests on pods (not replica count)

### 11. EKS — Helm Charts
- Package manager for Kubernetes — bundles all manifests (Deployment, Service, ConfigMap, Ingress)
- Used for third-party tooling installs (Prometheus, Grafana, nginx ingress, Istio) — not primary microservice deployments at AT&T
- Values files enable same chart across dev/stage/prod with different configs

### 12. SRE — Major Production Incident
- **Scenario**: Cascading payment service failure during flash sale at AT&T
- **Response**: Enabled circuit breakers (stopped retry storm), aggressively scaled pods via Jenkins, increased RDS connection limits, spun read replicas — stabilized in 40 minutes
- Kept stakeholders updated every 10 minutes via Slack incident channel
- **Postmortem**: Root cause = inadequate spike load testing + missing circuit breakers + conservative HPA thresholds → 8 action items → 65% reduction in similar incidents over next 2 quarters

### 13. SRE — SLOs & SLAs
- SLA = external commitment (e.g., 99.9% uptime, payments within 2 seconds at 99th percentile)
- SLO = internal target set tighter than SLA (e.g., 99.95%) to catch trends before customer impact
- Four Golden Signals: Latency, Error Rate, Throughput, Saturation

### 14. SRE — Alerting Strategy
- Threshold alerts (CPU > 80%, error rate > 50%)
- Anomaly detection via **Datadog Watchdog** — learns baseline, fires on deviation (e.g., 0.05% → 0.5% error rate)
- Error budget burn rate alerts (slow burn pages during business hours)
- Composite alerts (error rate high AND latency high) to reduce noise

### 15. Distributed Tracing
- AWS X-Ray (primary at AT&T for AWS workloads) + Datadog APM (on-prem servers)
- Use case: trace request across services to pinpoint bottleneck/root cause, not just surface symptoms

---

## What Went Well

- **Cleared every technical question** — AWS, Terraform, Jenkins, EKS, SRE all covered confidently
- **Terraform alignment** — Jose's #1 pain point is no IaC. Ramana's real AT&T experience importing 200+ services into Terraform was a direct hit
- **Incident story** was compelling — specific, structured (situation → actions → results → postmortem)
- **Asked smart closing questions** — team structure and AI adoption at NVIDIA showed genuine curiosity and culture fit
- **Remote work confirmed** — no relocation required (Dallas stays)
- **Datadog and SRE mindset** — proactive vs. reactive monitoring resonated with Jose
- **Jose explicitly moved forward**: *"Based on my interview with you today, I'm going to move you forward in the interview process"*

---

## What Could Have Been Better

- **Portfolio inconsistency** — Ramana's site listed "Site Reliability Engineer" while the resume emphasized DevOps. Jose noticed and asked directly. The explanation was reasonable but it was a small stumble out of the gate.
- **Secrets refresh timing** — Initially said secrets are fetched "every time there's a request to the cluster" before correcting to the CSI driver model. Jose specifically pushed back on cost implications — shows the answer required prompting to land on the right cost-aware answer.
- **Trailing answers** — A couple of responses (data subnet question, scripting question) trailed off or needed Jose to redirect. Tighter, more concise delivery would have been stronger.
- **Akamai vs CloudFront** — Ramana defaulted to CloudFront for the CDN layer; NVIDIA uses Akamai. Jose was gracious (*"you got the gist"*), but knowing the interviewer's stack would have been sharper.

---

## Hiring Signal Assessment

**Verdict: STRONG POSITIVE — Expecting Offer after Next Round**

Jose was direct: *"I'm going to move you forward."* He also disclosed urgency (*"my manager wants me to close this ASAP"*), a short timeline (all 4 next interviews by next Thursday), and explicitly asked about Python scripting readiness (HackerRank coding round incoming).

**Next Round (4 interviews, ~45 min each, by next Thursday):**
- One will include a **HackerRank coding challenge** (Python) with a software engineer
- One interviewer is based in **India (IST)** — early morning or late evening availability needed
- Monday (Feb 16) is an NVIDIA holiday — Tuesday–Thursday window is key

**Action Items:**
- [ ] Keep calendar open Tuesday–Thursday for back-to-back interviews
- [ ] Be flexible for early morning slot (India interviewer ~8:30–9:00 AM CST)
- [ ] Brush up on Python scripting — LeetCode medium difficulty, automation scripts, AWS SDK (boto3)
- [ ] Wait for coordinator to reach out (Jose submits feedback first, then she schedules)
- [ ] Review Akamai basics since that's their CDN

---

## Notes on the Role & Team

- **Team is small and understaffed** — 2 engineers handling 24/7 for the whole marketplace platform. This hire has real immediate impact.
- **NVIDIA AI culture** — Jensen Huang mandates Cursor usage company-wide. Team is encouraged (not just permitted) to adopt AI tooling and demo POCs to the team. Big green flag for someone already using AI tools.
- **Technical debt opportunity** — No Terraform, manual console changes, growing fast. Greenfield IaC ownership from day one.
- **Compensation/offer**: Not discussed in this round.
