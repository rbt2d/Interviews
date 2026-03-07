# Netflix Interview — Round 5: CI/CD Delivery Interview

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | March 4, 2026                                                        |
| **Duration**    | 47 minutes                                                           |
| **Interviewer** | Cameron — Netflix Delivery Org (owns deployment tooling), 12+ years at Netflix |
| **Format**      | Part 1: CICD system walkthrough (CodeSignal whiteboard) + Part 2: Troubleshooting exercise + Q&A |
| **Outcome**     | **STRONG PASS** — Cameron: *"That's exactly the thinking"*. Post-call audio captured: *"It's a yes, yeah"* and *"When it's wrapping in time, then it's a good signal"* |

---

## About the Interviewer

Cameron has been at Netflix for **12+ years**, primarily in the delivery space — building and operating deployment tooling. His team owns the Netflix deployment platform (Spinnaker-like tool with a web UI, single API gateway, and ~15 backend microservices). He described his team as *"a force multiplier in the ability to have a big impact on a lot of engineers."*

Cameron cares deeply about:
- CICD operational depth (not just knowing what tools exist, but having operated and debugged them)
- Support thinking: good escalations, good triage before escalating
- Documentation and tooling that reduces ticket volume

---

## Part 1: CICD System Walkthrough

### System Described: Identity Infrastructure Pipeline at AT&T

**What it deploys:**
- A Go-based authentication event processing service
- 3 Python collectors ingesting auth events from CloudTrail, application logs, and infrastructure scores
- Terraform infrastructure-as-code (modules + config)

**Repo structure:** Mono repo (all services + Terraform + config in one repo), trunk-based development, short-lived feature branches, PR required to merge to main

**Trigger:** GitHub webhook on PR merge to main → triggers Jenkins multi-branch pipeline (each service has its own Jenkinsfile)

---

### Pipeline Stages

| Stage | Details |
|-------|---------|
| **1. Lint & Static Analysis** | golangci-lint (Go), flake8 + bandit (Python), tflint + tfsec (Terraform) |
| **2. Unit Tests** | Comprehensive test coverage for Go services; 100% required for triage rules ("a bad rule in production is worse than no rule") |
| **3. Build & Package** | Multi-stage Docker builds (slim images for Go/Python), Terraform plan saved to S3 |
| **4. Security Scanning** | Trivy for container CVEs + dependency audits + secret detection; fail build on critical CVEs |
| **5. Integration Tests** | Docker Compose spinning up Kafka, Elasticsearch, PostgreSQL locally; feeds synthetic events through full pipeline end-to-end (~8 minutes) |
| **6. Artifact Push** | Docker images → ECR (tagged with git SHA); Terraform plans → S3 |

---

### Deployment Stages

#### Dev (Auto-deploy on every merge to main)
- Quick smoke tests: can services start? Can they connect to Kafka? Can they reach Elasticsearch?
- Catches gross configuration issues
- ~2 minutes
- Auto-promotes to staging on pass

#### Staging — Shadow Mode Validation
**Shadow Mode** (custom-built hybrid solution):
- Triage engine processes real-time events and compares automated output against a known baseline
- For each event: writes the triage output (rule matched, confidence score, recommended action) to a separate Elasticsearch index
- Python comparison script diffs shadow output vs. baseline at end of the shadow window
- Built by Ramana — came from being in the support loop and knowing exactly which failure patterns needed catching

**Shadow Mode Report (JSON summary posted to Slack):**
1. Rules that matched as expected ✅
2. Rules that diverged from the baseline ⚠️
3. New rule matches not in the baseline 🆕

**Grafana dashboard** shows Shadow Mode results over time (match rates during shadow window vs. baseline) — visual trend view, not just individual rule comparisons.

**Pipeline behavior on results:**
- **Clean pass** → Slack green light: *"Shadow Mode passed: 43 rules evaluated, 0 divergences, ready for production approval"* → auto-promotes
- **Divergence** → Pipeline stops; Slack flags the specific rules that diverged with a diff (expected vs. actual); human reviews to decide: intentional rule change (update baseline) or regression (fix the code)

**Key design principle:** *"I'd rather have a false positive that requires a human to say 'this is expected' than let an unintended rule change slip into production."*

#### Production — One-Click Approval + Automated Rollback

**Approval flow:**
- Person who merged the PR gets a Slack message with a one-click Approve button
- They verify the Shadow Mode summary, click Approve
- Rolling deployment begins; final Slack notification on completion or rollback

**Automated rollback logic:**
- Deployment script waits for health check endpoints to return healthy
- Health check runs every 30 seconds; requires 3 consecutive passes before instance is considered good
- Deep health check (not just port ping): verifies service can connect to Kafka, connect to Elasticsearch, and load triage rules from config
- If health check fails 3× in a row on the new instance: auto-rollback to previous ECR image (stored in deployment manifest), halt rollout entirely
- Reasoning: *"If the new version is failing on one instance, it's going to fail on all of them — no point continuing"*
- Slack alert fires: *"Rolling deployment failed on instance 2/3, auto-rollback started, here's the health check error"*

---

### Developer UX — Touch Points

1. **Pull Request**: CI checks (lint, tests, security scan) report back to GitHub before merge — first feedback loop
2. **Slack notifications**: Deploy to dev → smoke test passed → staging → Shadow Mode running → Shadow Mode passed/failed → production approval button → rollout complete or rolled back
3. **Grafana dashboards**: Deployment frequency, failure rates, rollback rates — team-level monitoring, not typically the individual developer's daily view
4. **One-click production approval**: In the Slack message itself — verified Shadow Mode summary + click Approve

*"The day-to-day developer experience is: put up your PR, watch CI checks on GitHub, watch Slack for deployment progress, one click for production. Trying to keep friction low so people aren't avoiding deployments because the process is painful."*

---

### Ramana's Role in This System

| Area | Ramana's Contribution |
|------|----------------------|
| Pipeline foundation | Set up by the 3-person DevOps team (Jenkins, Kafka, Elasticsearch, core CI stages) |
| Deployment workflow | **Ramana designed**: dev→staging→prod stage rollout, Shadow Mode validation, automated rollback logic |
| Application code | **Ramana built**: Go processing service and triage engine that the pipeline deploys (both user AND designer) |
| Day-to-day operations | **Ramana owns**: pipeline debugging, deployment triage (code vs. config vs. infrastructure), iterating on pipeline improvements |
| Iterative improvements | Added security scanning stages, Shadow Mode, automated rollback, upgraded from basic port check to deep health checks |

*"I laid the foundation and have been operating, improving, and extending it for over a year and a half. The dual perspective — being the user of the pipeline and also the designer of the deployment workflow — was really valuable."*

Cameron's reaction: *"This is a pretty cool, comprehensive system."*

---

## Part 2: Troubleshooting Exercise — "Spinnaker is Really Slow Today"

**Setup:** User reports in the support channel: *"Hey, the deployment tool is really slow today."*

### Step 1 — Clarifying Questions

**Before diagnosing:** Narrow down what "slow" actually means.

1. **What specifically is slow?** UI sluggish (pages slow to render) vs. pipeline execution taking longer than usual → completely different root causes
2. **Just them or widespread?** Multiple users = platform-level issue (degraded backend service, API gateway under load, database slow). Just them = user-specific config or target environment issue
3. **When did it start?** Fine yesterday? Fine this morning? Helps correlate with platform deployments or infrastructure events
4. **In parallel:** Check support channel for other similar reports; check if an active incident is already posted

### Step 2 — After Narrowing Down: "Multiple users, pipeline execution view slow to load"

**Architecture reasoning:** Single page app → API gateway → backend service for pipeline execution data → data store. Bottleneck is somewhere in that chain.

**Investigation path:**

| Layer | What to Check | Interpretation |
|-------|--------------|----------------|
| API gateway | Response times for the pipeline execution endpoint specifically | High latency on that route but others are fine → downstream service issue, not the gateway |
| Backend service | Response time, error rate, CPU, memory, GC pauses, traffic volume | Resource pressure or GC pause causing slowness |
| Data store | Query performance, table size, index health, concurrent write contention | Most common actual root cause in this scenario |
| Recent deployments | Any new version of this service recently? New query? Missing pagination? | Regression that works fine at small scale but degrades at production volume |

**Key insight offered:** *"This is often where I find the root cause — look at the database or data store backing that service. The pipeline executions view is essentially a query: give me all pipeline executions for this application. If the table has grown significantly, if an index is missing, or if there's contention from concurrent writes — that shows up as everyone's page loading slowly."*

Cameron's reaction: *"That's exactly the thinking. Don't really have anything more to dig into there."*

---

## What Went Well

- **Shadow Mode was the standout moment** — Cameron was visibly impressed. A Python script diffing Elasticsearch indexes isn't glamorous, but it caught real regressions before production and was built from direct support experience. The framing (*"I was the one manually triaging tickets, so I knew exactly which failure patterns the system needed to catch"*) was perfect
- **Dual perspective framing** (user + designer of the pipeline) — Cameron's whole team exists at this intersection; this resonated immediately
- **Deep health checks vs. port checks** — The specific improvement story (upgrading from "is the port open?" to "can it connect to Kafka, Elasticsearch, and load triage rules?") demonstrated operational depth
- **Automated rollback logic** was specific and well-reasoned — 3 consecutive failures, halt entirely, why (failing on one = failing on all)
- **Troubleshooting scenario** — Went straight to the right level: API gateway → backend service → data store. Correctly identified data store as "where I usually find the root cause in these situations"
- **Failure classifier connection** — When Cameron described the failure classifier Netflix is building, Ramana immediately connected it to his triage engine at AT&T (*"the approach is the same — looking at failures, context, matching against known patterns, surfacing probable cause"*). This was a genuine moment of alignment
- **Repeat ticket reduction stat** — 35% → 12% repeat tickets at AT&T, concrete and credible
- **Post-call audio captured:** *"It's a yes, yeah"* and *"When it's wrapping in time, then it's a good signal"* — Cameron's team confirmed positive outcome

## What Could Have Been Better

- **Canary analysis** — Ramana mentioned wanting something like automated Canary analysis with statistical confidence thresholds "if building at larger scale." This could have been framed more proactively as a future direction rather than an aside
- **Shadow Mode tooling** question — When Cameron asked if it was off-the-shelf or built in-house, the answer ("hybrid solution") was accurate but slightly meandering before landing on "Python script querying two Elasticsearch indexes." Could have been crisper.

---

## Hiring Signal Assessment

**Verdict: STRONG PASS — Confirmed by Post-Call Audio**

Post-call audio captured Cameron's team saying *"It's a yes, yeah"* and *"When it's wrapping in time, then it's a good signal"* — the interview wrapping up naturally (without running out of content) was treated as confirmation of a good session.

Cameron's explicit feedback during the interview:
- *"This is a pretty cool, comprehensive system"* (after CICD walkthrough)
- *"That's exactly the thinking"* (after troubleshooting scenario)
- Follow-up: *"Brandon will follow up with you"* — next steps confirmed

The failure classifier discussion was the best moment of the interview. Cameron described Netflix's current work-in-progress; Ramana described an already-deployed version of the same concept at AT&T (triage engine: 15 rules → 43 rules, 40% of tickets resolved without human involvement). Cameron lit up. This is the kind of alignment that changes an interview outcome.

---

## Cameron's Answers to Ramana's Questions

### Q1: Most exciting thing about Netflix day-to-day?

> *"In our delivery space, we're kind of really at the center point of all sorts of capabilities. We're an integration point for the dev loop, pre-deployment, observation of deployment, infrastructure management, resilience validation. It's a really interesting problem space at our scale, and our team is a force multiplier in the ability to have a big impact on a lot of engineers."*
>
> *"The other thing exciting lately is AI tooling adoption — increased velocity, increased rate of change. More change means more things to review, validate, and more chances to queue up a whole bunch of changes. Minimizing change set size is currently top of mind."*
>
> *"Netflix as a company and culture is very refreshing. It's not policy-heavy. I'd summarize it as: treat all employees as highly trusted, highly responsible adults. Let them make good decisions, trust that they will, and build around enabling innovation — not guiding everything through process."*

### Q2: What makes a good escalation from ESO to the delivery team?

- Localize the problem: is it widespread or specific to a user/service/configuration?
- Check the next level of logs (deployment tool logs, common failure scenarios)
- Escalate when: there's probably a bug in the system, or it's doing something that doesn't make sense and needs someone to debug it at the source
- *"It's not expected that ESO will find everything — there are lots of weird edge cases. But thorough troubleshooting with the user is what makes a good escalation."*

### Q3: Best area where documentation/education would reduce ticket volume?

> *"Understanding why a deployment pipeline failed is the biggest one. Users craft pipelines from building blocks and stitch them together — there are a lot of ways to introduce different behaviors, and it's hard to reason about whether it's failing because your deploy was bad or because the platform itself did something."*
>
> *"We're rolling out a failure classifier right now that looks at a failed run plus context — did you just change your configuration? Has this been failing repeatedly? What's the typical behavior? — and surfaces a probable cause to the user. Something like 'your instances didn't come up healthy, that's why this failed.'"*
>
> *"In general, our documentation could use a lot of work. It's something we're trying to invest more in."*

---

## Key Delivery Team Context

- Netflix deployment tooling: web UI + single API gateway + ~15 backend microservices
- Users can build their own deployment pipelines from building blocks — highly configurable, highly complex
- **Failure classifier** being actively rolled out — looks at failed runs + context for probable cause diagnosis (directly overlaps with Ramana's triage engine work)
- Team's biggest documentation gap: pipeline failure reasoning — why did it fail? Was it the deploy or the platform?
- Cameron's team integrates with: dev loop, pre-deployment validation, deployment observation, infrastructure management, resilience tooling, automated campaigns
