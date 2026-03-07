# Netflix Interview — Round 6: Project Presentation & Deep Technical Q&A

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | March 5, 2026                                                        |
| **Duration**    | 56 minutes                                                           |
| **Interviewer** | Erica — Netflix Cloud Infrastructure Team, Linux/Kubernetes Specialist, ~1 year at Netflix (prev. SpaceX, 5 years, senior Linux engineer + kernel engineering) |
| **Format**      | Project presentation (architecture diagram shared in advance) + deep technical Q&A |
| **Outcome**     | **STRONG PASS — HR NEXT** — Erica: *"You did a great job"* + *"I will put my feedback in for the recruiter, and HR will be the one to contact you next"* |

> ⚠️ **"HR will contact you next"** is the most definitive closing statement in all 6 rounds. This strongly implies offer stage or final decision pending.

---

## About the Interviewer

**Erica** spent 5 years at SpaceX as a senior Linux engineer doing kernel engineering in a heavy Kubernetes environment. Came to Netflix ~1 year ago. Specializes in Linux systems and Kubernetes on the cloud infrastructure team (compute tools, Titus — Netflix's container orchestration platform). She described SpaceX as extremely reactive ("trauma from SpaceX"), contrasting it with Netflix's proactive culture. She has ADHD and autism, and appreciated the architecture diagram being shared in advance — said it made it much easier for her to follow along during the explanation.

---

## Project Presented: Authentication Event Processing Pipeline (AT&T)

Ramana shared an architecture diagram in advance and walked through the full system end-to-end.

### Problem Statement

**Before the system existed:**
- Auth support was a 30–40 minute scavenger hunt per ticket: Okta admin console + CloudTrail + Splunk + ELK stack — four separate systems, manually correlated
- 15–20 auth tickets per day
- 35% of tickets were the **exact same root causes repeating**: certificate rotation not propagating, client secrets expiring unnoticed, SAML metadata going stale
- Developers had zero visibility — black box. They saw 403/401 errors and opened a ticket
- By the time support got to it, users were already impacted

**Two core problems being solved:**
1. Support team spending time on work that should be automated (toil, not engineering)
2. Developers had no self-service path — completely dependent on support for basic auth failure diagnosis

---

### Architecture: End-to-End

#### Data Sources → Kafka (3 Topics, 1 Per Source)

| Source | Collection Method | Notes |
|--------|------------------|-------|
| **Okta System Log API** | Python poller (every 30 seconds) → Kafka | REST API polling, normalized and published |
| **AWS CloudTrail** | SQS → Python consumer → Kafka | Captures IAM AssumeRole, KMS operations, Secrets Manager access, all AWS API calls |
| **Application Auth Logs** | Fluent Bit agents (running alongside services) → Kafka | Service-level logs: token validation, scope checks, auth failures |

**Why separate Kafka topics per source:**
- Isolation — Okta collector failure doesn't affect CloudTrail ingestion
- Independent scaling per source
- Kafka replay: 72-hour retention window. When a new triage rule is deployed, reprocess the last 72 hours to backfill missed detections — *"turned out to be more valuable than expected"*

#### Processing Layer: Go Service (Single Service, 3 Sequential Operations)

**1. Enrichment**
- Attaches context to raw events from Postgres metadata store
- Which service owns this client ID? Who is the service owner? What SAML certificate is the service provider currently using? When does it expire?

**2. Correlation**
- Matches events across sources using shared identity identifiers: request ID, session ID, client ID
- If a developer's OAuth token request fails → connects the Okta rejection + CloudTrail access denied + application 403 error into one **correlated incident**
- Produces a unified view of what happened across all three sources

**3. Triage Engine**
- Rule-based system: looks at correlated events and matches against known failure patterns
- Example rule: *"If SAML assertion failures are spiking for a specific service provider, AND that provider's certificate was rotated in the last 24 hours → tag as probable certificate propagation gap, high confidence, fire alert"*
- Started with 15 rules, grew to 43

#### Output (3 Destinations)

| Destination | Purpose |
|-------------|---------|
| **Elasticsearch** (30-day retention) | Searchable events powering Grafana dashboards — developers self-diagnose their own auth health |
| **PostgreSQL** | Triage rule definitions + incident metadata |
| **Alerting layer** | PagerDuty (P1 anomalies affecting multiple teams) + Slack (targeted notifications to specific service owners for lower severity) |

**Design philosophy:** *"Give developers the context to diagnose their own issues, instead of making them depend on support for every 403 error. Fundamentally different support model than what we had before."*

---

### Impact Metrics

| Metric | Before | After |
|--------|--------|-------|
| Average resolution time per ticket | 30–40 minutes | 12 seconds |
| Tickets auto-closed without human involvement | 0% | 40% |
| Repeat ticket rate | 35% | 12% |
| Proactive detections (before any user impact) | 0 | 3 certificate rotation failures caught early |

**Most proud metric:** Catching 3 certificate rotation failures before a single developer saw a 403. Before this system, each would have been a P1 incident affecting 100+ people for 30–40 minutes before anyone opened a ticket.

**Hidden challenge not on the architecture diagram:**
Before the pipeline could work, Ramana had to go team by team (6 teams, 2 weeks) to get them to add consistent structured fields to their auth-related log lines (request ID, session ID, client ID) — without these identifiers, cross-source correlation was impossible. This wasn't in the job description and delayed his own delivery timeline, but without it the system's core value wouldn't have existed. Erica's reaction: *"Six teams in two weeks is incredibly quick. In my past experience, even getting one team on board could take more than two weeks."*

---

## Deep Technical Q&A

### Q1: How would this scale from 15M to 15M+ events per day?

**Kafka:** Scales horizontally by design — add partitions and consumer instances. Producers are stateless polling jobs (multiple instances run without coordination). Kafka specifically chosen to decouple ingestion rate from processing rate — traffic spikes queue up, processing service works at its own pace.

**Go processing service:** Stateless for enrichment and correlation (no shared state between instances except in-memory cache). At higher scale, run multiple instances each consuming different Kafka partitions.

**Scaling change needed:** The triage engine's in-memory cache (5-minute window keyed by client ID) is the only coordination point. At significantly higher scale, move this to **Redis with a TTL** so multiple processing instances share the same correlation window.

**Elasticsearch:** Scales horizontally by adding nodes and increasing shard count. Need more aggressive index lifecycle management at scale (move older data to warm/cold tiers faster).

**PostgreSQL (metadata store):** Add **read replicas** — all processing service queries are reads (no writes during event processing).

**Alerting layer:** Fire-and-forget; PagerDuty and Slack handle burst traffic fine.

*"The fundamental architecture doesn't need to change. It was designed to scale horizontally at every layer."*

---

### Q2: How do you handle duplicate events and ordering issues?

**Deduplication — two-level approach:**

| Level | Mechanism |
|-------|-----------|
| **Kafka consumer level** | Go service commits offsets per partition. Restart/rebalance resumes from last committed offset → at-least-once delivery possible |
| **Application level** | Composite deduplication key per source (checked against in-memory dedup set with short TTL) |

**Dedup keys by source:**
- **Okta events:** Okta-native event UUID
- **CloudTrail:** CloudTrail event record event ID
- **Application logs:** `hash(timestamp + request ID + source service)`

If the key was seen in the last 5 minutes → skip event. Catches the vast majority of duplicates with negligible performance cost.

**Ordering:**
- Kafka topics partitioned by **client ID** → all events for a given client land on the same partition → Kafka guarantees ordering within a partition
- For any single client's auth flow, events arrive in order — this is what matters for correlation and triage
- Cross-client ordering is irrelevant (each client's events are independent; triage engine only correlates within the same client ID window)
- **CloudTrail delay edge case (up to 15 min, AWS limitation):** Use the **event timestamp from the CloudTrail record** for correlation, NOT the Kafka ingestion timestamp. Late-arriving CloudTrail events are placed correctly in the correlation window based on when they actually occurred.

---

### Q3: Are triage rules in the codebase or configured somewhere else?

**Current state:** Rules are stored in **PostgreSQL** (each rule is a row: pattern definition, set of conditions that must match across correlated events, confidence score, recommended action). Go service loads the rule set on startup and refreshes on a configurable interval (no DB hit per event).

**Adding a new rule today:**
1. Write the rule definition
2. Add a database migration
3. Write unit tests with synthetic events (should trigger + should not trigger)
4. PR review
5. Deployment cycle

**Known limitation Ramana called out:** Every new rule requires engineering involvement and a deployment cycle. In a support context where new failure patterns emerge constantly, this is a bottleneck.

**What he'd build instead:** A **config layer with a lightweight admin UI and a validation step** — support engineers could author a rule, test it against replayed historical events, and push it live without touching the codebase or waiting on engineering. *"Velocity would have been higher with a better authoring workflow."*

---

### Q4: Limitations of the 3-sigma anomaly detection model?

**Works well for:** Services with consistent, predictable traffic (steady login volume, clean normal distribution). Spike beyond 3 standard deviations = real signal.

**Breaks down for:** Bursty or seasonal traffic patterns:
- End-of-quarter reporting tools with sudden heavy usage
- Batch processing services that authenticate hundreds of service accounts on a schedule
- These would spike auth failure rates legitimately, but the 7-day trailing average baseline would fire false alerts

**Fix implemented:** Changed baseline from one global average per service → **baseline per service, per hour of day, per day of week**. Monday 9am spike compared against previous Monday 9am windows, not the overall weekly average. Eliminated most false positives from predictable burst patterns.

**Cold start problem:** New services have no historical baseline. Solution: **suppress anomaly alerts for the first 7 days** (collect data, build baseline, then start alerting). Avoids a flood of false alerts every time a team ships a new service.

---

### Q5: Is there a Postgres backup and disaster recovery plan?

- **Primary + asynchronous standby replica** — standby stays in sync with streaming replication; auto-promotes on primary failure
- **RPO ≈ 0** — no committed transactions lost on failover
- **Daily automated snapshots to S3** — 30-day backup window for point-in-time recovery (e.g., corrupt table from a bad migration → restore to any point in the last 30 days)

---

### Q6: What would break first under extreme scale pressure?

**Elasticsearch ingest layer** — Kafka and Go handle spikes gracefully (Kafka queues, Go processes at its own pace). But Elasticsearch has hard throughput limits per shard. A sudden event volume spike faster than ES can index → rejected writes → back pressure into Go service.

**Fix:** Pre-size shard count appropriately + add a **small buffer queue between the Go service and Elasticsearch writes** to absorb burst without losing events.

Erica's reaction: *"The sharding used to drive me insane with Elasticsearch. That was always the first thing to break for us too. Eventually we moved everyone to Splunk just to not deal with it anymore. But at Netflix, Elasticsearch seems to never go down — must be some secret sauce."*

---

### Q7: What was the hardest technical problem to solve?

**Cross-source correlation with mismatched latency characteristics:**
- Okta events: arrive within seconds
- Application logs: arrive within seconds
- CloudTrail events: can be delayed up to 15 minutes (AWS limitation, uncontrollable)

**The challenge:** Two out of three signals for an incident arrive immediately; the third shows up minutes later.

**Initial approach (rejected):** Real-time streaming joins across all three sources with time-based windowing. The senior architect shut this down immediately — had built similar systems and streaming joins at scale are the most fragile part of any pipeline. Windowing + watermark logic becomes impossible to debug at scale.

**Solution (the breakthrough):**
Split the problem into two modes:
1. **Enrich each event independently** with enough context that correlation can happen at query time in Elasticsearch — for complete investigation after the fact
2. **In-memory cache with 5-minute window keyed by client ID** for the real-time triage engine — if a CloudTrail event arrives late, it lands in Elasticsearch correctly and the *next* occurrence within the window will catch it

*"The hybrid approach — streaming for fast triage, query-time for complete investigation — was the breakthrough that made the whole system reliable."*

---

### Q8: What part of the project are you most proud of?

**Proactive detection.**

> *"The most proud part is the proactive detection. Every auth incident started the same way — a developer was already impacted, they opened a ticket, and we started investigating from zero. The moment the pipeline caught its first certificate rotation gap and alerted us before any user saw a single 403 error — that completely changed the relationship between the support team and the developer community. We went from 'open a ticket and wait' to 'we already know about it and we're fixing it.' That shift from reactive to proactive is what I'm most proud of — because it's not just a technical win, it fundamentally changed how developers trusted the support team."*

Erica's reaction: *"I have so much trauma from SpaceX because it was also very reactive. Every day was just putting out fires. No time to create a solution to make the fire go away forever. I definitely appreciate proactiveness."*

---

## What Went Well

- **Architecture diagram shared in advance** — Erica explicitly thanked Ramana for this: *"It makes it so I don't have to keep listening and trying to hold the picture in my head"* (she mentioned having ADHD and autism). Small gesture, large impact
- **Full system walkthrough was compelling** — covered data sources, Kafka topology rationale, enrichment/correlation/triage sequence, output destinations, and philosophy in a logical flow
- **Impact numbers were specific and credible** — 40 min → 12 seconds, 40% auto-close, 35% → 12% repeat rate, 3 proactive catches
- **Hidden challenge story (6 teams, 2 weeks)** — Erica's reaction was the strongest in the interview: *"Six teams in two weeks is incredibly quick"*. Shows ability to get cross-team alignment, not just build systems
- **Scalability answer was architectural, not hand-wavy** — specific levers (Redis for correlation cache, read replicas, ES shard pre-sizing, buffer queue)
- **Deduplication answer** showed deep Kafka knowledge (at-least-once semantics, offset commits, composite dedup keys)
- **3-sigma limitation answer** demonstrated willingness to critique own work — named the failure case (seasonal/bursty traffic) and the fix (per-service, per-hour, per-day-of-week baseline)
- **Rule management self-critique** — proactively identified the engineering-gated rule authoring as a limitation and described the admin UI solution unprompted
- **Hardest problem answer** was excellent — named the specific failure (streaming joins), explained why the architect rejected it, and articulated the hybrid solution clearly
- **Closing statement from Erica:** *"I will put my feedback in for the recruiter, and HR will be the one to contact you next"* — the strongest and most definitive closing statement of all 6 rounds

## What Could Have Been Better

- **Kafka ordering** answer initially focused on partitioning correctly but CloudTrail delay handling came as a slightly delayed add-on — could have been integrated earlier as a natural part of the ordering discussion
- Some answers ran long — Erica's questions were concise and direct; tighter responses would have fit her style better

---

## Hiring Signal Assessment

**Verdict: OFFER STAGE** — *"HR will be the one to contact you next"*

This is the clearest and most direct closing statement in the entire Netflix interview process. Erica said:
1. *"You did a great job"*
2. *"I will put my feedback in for the recruiter"*
3. *"HR will be the one to contact you next"*

Number 3 is not standard interview closing language — it's a handoff to the offer/decision stage.

---

## Erica's Answers to Ramana's Questions

### Q: Most exciting thing about Netflix day-to-day?
> *"Being able to act on ideas immediately. I'm someone who's always thinking of ways to solve things — I'll have a thought at a random time during the day or on the weekend. The most exciting thing is being able to act on those — whether it's automating something for the entire team or just my pod. Eliminating toil is exciting. We have so many AI models and so much amazing internal software here that deploying something doesn't take long. I get a dopamine burst from solving stuff. I'm always excited to see what the next challenge is."*

### Q: Biggest difference between SpaceX (reactive) and Netflix (proactive)?
> *"Very proactive here. If you bring up a problem, they try to solve it or put it on their roadmap immediately. At SpaceX I'd wait months for a fix for something that caused headaches every day. Here, teams take feedback seriously — they want to make you happy and they want to make their customers happy. Nobody wants to be reactive. There's not a lot of angry people either. People here seem happy at their jobs."*

### Q: Most common issues in the compute/cloud infrastructure channel?
- **Primary:** Deployment failures (why did my task fail? Docker image issues, app not scaling)
- **Titus-specific:** Netflix's internal container orchestration platform (essentially Kubernetes with containerd) — task failures, waiting state issues, capacity requests for specific instance types
- **Erica's specialty:** Linux and Kubernetes — gets inside containers to troubleshoot why apps aren't starting
- **Teammate Dola's specialty:** AWS (former AWS background) — Erica leans on him for CloudTrail-heavy issues

### Q: How does the team handle knowledge sharing?
- Engineering Help pod: **learning sessions every Monday morning** — someone who found a root cause on a thread walks the team through it
- **ESO Spa Learning Channel** (Ronnie's creation from Round 2) — members post solutions from tickets ("this is how I solved it in case you see it again")
- Erica's suggestion: would be nice to auto-document anything posted to that channel into a proper doc for future use

### Q: First 90 days success?
> *"Just being able to close threads on your own is big for 90 days — not that the bar is low. By 6 months, you can tell who's on an upward trajectory: they're finding solutions to problems in other teams' applications and suggesting fixes, or even submitting PRs. I'll often put in a PR to someone's infrastructure saying 'you're doing this in a really complicated way, here's an easier approach.' Teams are usually really receptive. Don't be too hard on yourself in the first 90 days — I felt huge imposter syndrome because there's so much to learn. New tool or technology every day. Just own it, be confident."*

---

## Key Netflix Compute/Cloud Infrastructure Context

- **Titus**: Netflix's internal container orchestration platform (Kubernetes-based, using containerd). Most common ticket type: why did my task/container fail?
- Everything is available to support engineers — all repos, can submit PRs to anyone's codebase
- Netflix is a big Java company alongside Python — language agnosticism is valued
- **Erica's insight on AI:** Netflix has access to an enormous number of AI models; once you have an idea, the tools make execution fast
- *"They treat you like an adult. Everything is transparent."* — Erica's summary of Netflix culture vs. prior experience
