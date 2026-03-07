# NVIDIA Interview — Round 2: Development Manager Screen

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | February 18, 2026                                                    |
| **Duration**    | 56 minutes                                                           |
| **Interviewer** | Aga — Development Manager, NVIDIA E-Commerce Marketplace Team       |
| **Role**        | Senior DevOps Engineer — Individual Contributor (Remote)            |
| **Context**     | Aga leads the software engineering org (~20–21 engineers across US and India) that DevOps supports directly |
| **Outcome**     | **Passed** — Aga was impressed: *"You have given me the longest time for my questions, which is really good. It tells that you know your stuff."* |

---

## About the Interviewer

Aga has been at NVIDIA for ~10 years and leads the development organization for the E-commerce group. His team works hand-in-hand with Jose's DevOps/infrastructure team. He described NVIDIA as *"the most valuable startup"* — fast-paced, startup culture despite being one of the world's largest companies. His team has ~8 engineers in the US and ~12–13 under his counterpart Britesh in India.

**His biggest pain point:** Business is moving faster than infrastructure can scale, creating technical debt.

---

## Topics Covered & Answers

### 1. Career Journey & Background
- Started as a software engineer at Black Box (India, 2015) — writing Python/Java backend services, got pulled into deployment automation
- Master's at UMKC → first real DevOps role supporting research applications (AWS, Terraform, Jenkins, EKS foundations)
- Joined AT&T in 2018 — scale jumped from a few services to 200+ microservices

### 2. Microservices Architecture at AT&T
- Moved from semi-monolithic to domain-driven microservices — each service owns its own data and deployment pipeline
- Compute: EKS with 10,000+ pods across multiple regions, Helm charts, HPA based on request-per-second metrics
- Communication: Synchronous via internal ALB + service discovery; async via SQS/SNS for decoupled event-driven workflows
- Payment processing: SQS buffering to absorb flash sale bursts without overwhelming downstream
- Data: Each service owns its own RDS (Postgres) or DynamoDB — shared databases strictly forbidden
- Cross-service data: Event sourcing — services publish events, consumers subscribe to what they need
- Observability: X-Ray for distributed tracing across 10–15 service hops, Datadog for metrics + anomaly detection

### 3. Microservice Intercommunication Issues
- Pre-condition: Mandatory distributed tracing before any service goes to production
- X-Ray let him pull a trace and see exactly which hop failed — API gateway timeout, Lambda cold start, or RDS connection pool exhaustion
- Without tracing, debugging distributed systems is "just guesswork"

### 4. CICD Strategy — Jenkins Pipelines
- Shared Jenkins pipeline library in GitLab — consistent base template, teams extend as needed
- 5 stages: Build → Scan → Test → Deploy → Verify
- Immutable artifacts: Docker image tagged with commit SHA, same image promotes through all environments — no rebuilds
- Security scanning: Container vulnerability scanning (fail on critical CVEs), shifted security left
- Deployment gates: Dev auto-deploys on merge; Stage requires integration + smoke tests; Prod requires manual approval + deployment window check
- No Friday deployments, no deployments during peak traffic
- Canary deployments + automated rollbacks: Post-deploy health checks for 5 minutes → auto-rollback on anomaly
- Result: Weekly → multiple deployments per day; 50–80 deployments daily across all environments; 10–15 prod deployments on a typical day

### 5. Deployment Frequency
- 50–80 deployments daily across 200+ microservices in all environments
- 10–15 production deployments on a typical day
- Shift happened in phases: smaller + more frequent releases reduced risk, engineers trusted the pipeline

### 6. Environment Structure
- **Dev**: Wide open, auto-deploy on every merge to feature branches, fast feedback, engineers can break things
- **QA**: Integration testing — multiple services deployed together, catches inter-service issues
- **Staging**: Production mirror — same infrastructure, same scaling, anonymized production data. Any drift between staging and prod is a bug to fix. Load tests and chaos experiments run here.
- **Production**: Blue/green sub-environments (active + inactive), Canary slices for gradual rollouts on high-traffic services
- Each environment has its own AWS account (isolation), managed via Terraform workspaces with environment-specific variables

### 7. Developer Collaboration
- ~40% of the job is working with developers
- Embedded with dev teams: attends standups, sprint planning, understands pain points firsthand
- Ran office hours twice a week — any developer could drop in, no ticket/scheduling needed
- Pairs with developers to understand *why* the current flow frustrates them, not just optimize blindly

### 8. Development Background
- Started as a software engineer (Python/Java, REST APIs, database integration)
- This background helps translate developer pain: if a service times out on a DB call, can look at connection pooling logic and ORM queries to determine if it's app or infrastructure
- Wrote Python automation, deployment scripts, log aggregation tools, custom Datadog integrations, Lambda functions

### 9. AI in DevOps
- Uses Cursor and GitHub Copilot for Terraform module generation and debugging
- Datadog Watchdog: ML-based anomaly detection, catches gradual memory leaks and slow degradations that static thresholds miss
- POC'd AI for log analysis and pattern recognition across millions of log files — not fully productionized before starting job search
- Excited about NVIDIA's AI-first culture (Jensen's mandate for Cursor usage)

### 10. Handling Technical Debt in a Fast-Moving Business
- **"Paving the road while driving on it"** — doesn't ask for dedicated tech debt sprints (rarely approved)
- Improvements embedded into every new feature: new service = follows new TerraForm modules and pipeline standards
- Ruthless prioritization on leverage: what one improvement unblocks the most teams? Usually self-service tooling
- Self-service TerraForm modules: teams fill in variables, module handles networking/security groups/IAM
- Pragmatic on urgency: quick solution now + documented improvement plan later
- Living tech debt backlog with estimated impact/effort
- Transparent with stakeholders on trade-offs: if something takes 4 weeks, says it takes 4 weeks — no surprises

### 11. Stakeholder Communication
- Joined capacity planning meetings with PMs and business ops to understand upcoming launches, traffic projections, seasonal patterns
- During incidents: updates in plain language — "checkout is degraded, estimated recovery in 20 minutes" — no technical jargon
- For surprise traffic (flash sales with 24 hours notice): defensive architecture — extra capacity headroom on critical services, application-level auto-scaling metrics (RPS, queue depth, response time), emergency scaling runbooks (one-command operations)

---

## What Went Well

- **Depth and length of answers** — Aga explicitly noted: *"You have given me the longest time for my questions, which is really good"* and *"It tells that you know your stuff"*
- **Microservices architecture story** was compelling — event sourcing, SQS buffering for flash sales, domain-driven decomposition
- **Developer empathy** came through clearly — office hours, embedded in standups, coding background translated well
- **Technical debt philosophy** resonated — "paving the road while driving on it" matched Aga's exact pain point
- **AI adoption answer** was perfectly timed — matched NVIDIA's culture and Jensen's mandate
- **Stakeholder communication** examples were concrete and practical
- **Stayed past the time limit** — Aga was willing to go over because the conversation was flowing well

## What Could Have Been Better

- **Defensive architecture answer for surprise traffic** — answer was solid but slightly generic; could have cited a specific metric or threshold used at AT&T for pre-warming
- **Cross-service data / event sourcing** — touched on it briefly but could have gone deeper on consistency guarantees (eventual consistency tradeoffs)
- The answer on AI was a bit AT&T-focused early on before pivoting to NVIDIA — could have led with NVIDIA excitement sooner

---

## Hiring Signal Assessment

**Verdict: STRONG POSITIVE**

Aga said *"I'm very impressed with your skill set"* and indicated he'd continue the process. He stayed 10+ minutes over the scheduled time. His biggest concern (business outpacing infrastructure) was directly addressed by Ramana's tech debt philosophy and self-service tooling approach.

**Key insight from this round:** Aga cares deeply about DevOps-developer collaboration and team culture fit. He works closely with Jose's team daily — so being a team player who embeds with developers is as important as technical skills.

---

## Notes on the Role from This Round

- ~20–21 total engineers (dev + DevOps): 8 in US, 12–13 in India under Britesh
- Biggest current challenge: business roadmap growing faster than infrastructure can scale → lots of technical debt opportunity
- In-house platform migration ongoing: moving away from third-party platforms (BigCommerce for order/park management) to in-house builds
- Payment platform: moved from Digital River to in-house UCP (Universal Cloud Payments)
- Culture: Fast-paced startup despite being the world's most valuable company — initiative is rewarded, not just attendance
