# Netflix Interview — Round 1: Security Support Solutions Engineer Screen

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | January 28, 2026                                                     |
| **Duration**    | 41 minutes                                                           |
| **Interviewer** | Unknown (name not captured in transcript)                            |
| **Role**        | Security Support Solutions Engineer (Internal Developer Platform)    |
| **Format**      | Recruiter/Hiring Screen — Technical + Culture                        |
| **Outcome**     | **Unknown** — transcript cut off before closing; conversation was going strongly |

> ⚠️ **Note:** Transcript was cut off at ~15:48 of 41:01. The last topic covered was the "Courage" culture question. Final feedback and outcome are not available from the transcript.

---

## Critical Role Context — Read First

The interviewer was **very transparent** about this role being fundamentally different from a traditional DevOps/SRE engineering role:

- **80% of time**: Taking support tickets from internal Netflix developers via Slack and a third-party ticketing platform
- **70% of tickets**: Resolved independently by the support solutions engineer
- **20–30% of tickets**: Escalated to platform engineering teams (bad code pushes, access restrictions, etc.)
- **Users**: Internal Netflix developers, NOT external customers
- **Common ticket types**: IAM policy denied, S3 access issues, encryption/decryption failures, third-party security tool assessments
- This is a **security-focused support role** — not a pure infrastructure engineering role

The interviewer explicitly asked: *"What's your motivation for moving from a heavy engineering role to a support engineering space?"*

---

## About the Role

Netflix's internal security platform team supports developers across the company. The team uses a centralized Slack channel for inbound tickets, a third-party vendor for operational ticketing management, and escalates to platform engineering teams when issues are out of scope. The security stack includes Okta, MFA, phishing-resistant auth, encryption/decryption tooling, and IAM policy management at Netflix scale.

---

## Topics Covered & Answers

### 1. Introduction & Background
- Senior DevOps/SRE at AT&T — infrastructure automation, incident response, observability
- Hands-on with Python, Go, AWS ecosystem (EKS, EC2, CloudFormation)
- Built monitoring stack with Prometheus and OpenTelemetry for 100+ microservices — reduced MTTR from 2 hours to 15 minutes
- Years of on-call experience, runbook writing, production troubleshooting
- Previous startup role gave full lifecycle experience (development through deployment)
- 8+ years focused on building and scaling cloud infrastructure

### 2. IAM Roles & Authentication
- Uses roles over users for application/service workloads (temporary credentials, auto-rotation)
- **IRSA (IAM Roles for Service Accounts)**: EKS pods assume specific IAM roles to access S3, DynamoDB — no hard-coded credentials
- **EC2 instance profiles**: Instance-level IAM roles for compute workloads
- **Cross-account roles**: Multi-account AWS setup — services in one account access resources in another
- **OIDC Federation**: GitHub Actions assumes IAM roles without storing long-lived credentials
- **Policy conditions**: IP address restrictions, MFA requirements on specific resource access

### 3. Common IAM / Authentication Issues (Day-to-Day)
~30% of AT&T support tickets are IAM-related. Common patterns:

| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| Permission denied errors | IAM policy not updated when new feature needs S3/DynamoDB access | CloudTrail logs → identify denied API call → update policy |
| AssumeRole failures (IRSA) | Trust relationship misconfigured, missing service account annotation, OIDC provider not set up | Fix trust policy, add annotation, verify OIDC setup |
| Session token expiration | CICD jobs running longer than temp credential validity | Break up long-running jobs OR role chaining with longer session duration |
| MFA-related access failures | Dev session lacks MFA when accessing MFA-required production resources | Enforce MFA re-auth before access |
| Confused deputy attacks | Cross-account access missing External ID checks | Add External ID condition to trust policies |

**Process**: CloudTrail logs → identify denied API calls → determine what's missing → fix or document in runbook. Maintains a self-service IAM runbook so teams can unblock themselves.

### 4. Team Structure / Support Model at AT&T
- JIRA for ticket management (P1–P3 priorities)
- Rotating on-call (primary + secondary per week)
- P1 = immediate PagerDuty page (service outage, major integration failure)
- P2/P3 = queue-based response
- Internal developers of AT&T are the users (not external customers)
- Acts as primary point of contact for Kubernetes platform, AWS infrastructure, deployment problems, monitoring gaps

### 5. Secret Management (HashiCorp Vault)
- **HashiCorp Vault** as centralized secret management: database passwords, API keys, certificates, sensitive credentials
- Dynamic secrets with automatic rotation — far superior to hard-coded secrets in config files
- **Vault Transit Secret Engine** for encryption-as-a-service:
  - Application makes API call to Vault Transit with plaintext
  - Vault returns encrypted ciphertext
  - Decryption: send ciphertext back to Vault → returns plaintext
  - Application never handles encryption keys directly
- **AWS Secrets Manager** for AWS-specific secrets (RDS credentials, etc.)

### 6. Motivation for Moving to Support-Heavy Role
- Already spending ~40% of time on support work at AT&T (Slack, JIRA tickets)
- Most satisfying part of the job: unblocking developers stuck on issues — *"When someone's been stuck for hours and I can look at CloudTrail logs, figure out what's missing, and unblock them in 15 minutes"*
- Immediate feedback loop vs. infra engineering (Terraform modules take weeks before anyone uses them)
- Security tooling interest: wants to go deeper into Okta, MFA, phishing-resistant auth — has exposure but wants real expertise
- Netflix's scale + engineering culture = best place to build security expertise
- Comfortable with 80/20 split; already does informal triage of "can I fix this, or does this need to go to platform team"

### 7. Netflix Culture — Feedback
**Example given**: Prometheus monitoring migration project at AT&T
- Built entire monitoring architecture in isolation (Terraform, exporters, dashboards) — technically excellent
- Manager's feedback: *"Technically brilliant, but you completely failed to bring the team along"*
- Had not consulted other SREs who would use it; presentation was too technical, didn't explain the "why"
- **Changed approach**: Scheduled 1:1s with each team member to understand pain points; revised dashboards based on their input; created documentation + training sessions; weekly Slack updates on progress; asked for feedback early instead of at the end
- **Outcome**: Much better adoption — team felt ownership of the tool
- **Lesson learned**: Technical solutions are only as good as the people who use them. Form a design review early, pull people in before going too far down a path.

### 8. Netflix Culture — Courage
**Example given**: Disaster Recovery architecture meeting with senior leadership
- Proposal on table: Active-active multi-region deployment across 3 regions — VP was pushing it, architects agreed, timelines being discussed
- Ramana spotted the problem: would triple infrastructure cost, massive operational complexity, actual SLA requirements didn't justify it (needed 99.99%, not 99.999%)
- **Spoke up despite the room**: Said he disagreed, pulled up 2 years of incident data showing zero regional outages — issues were application-level, not infrastructure-level
- **Proposed alternative**: Active-passive (primary + warm standby) — meets SLA, costs 60% less, team can actually operate it without burning out; can evolve to active-active if requirements change
- VP pushed back: *"Are you willing to risk the business on your opinion?"*
- Held the position, walked through the math
- **Outcome**: VP sent team back for a proper cost-benefit analysis; two weeks later the analysis validated the active-passive approach

---

## What Went Well

- **IAM depth was strong** — IRSA, OIDC Federation, cross-account patterns, confused deputy — all covered confidently
- **Vault transit engine answer** showed genuine encryption-as-a-service knowledge, not just "we use Vault"
- **Support motivation answer** was honest and specific — already doing 40% support, gave a concrete example of satisfaction from unblocking developers
- **Feedback example (Prometheus)** was perfectly structured — failure → honest self-reflection → concrete behavior change → measurable outcome
- **Courage example (DR architecture)** was outstanding — spoke up against VP with data, not emotion; proposed a better solution; held the position; was ultimately validated
- **Transparency alignment** — interviewer appreciated that Ramana was direct about what he enjoys and what he wants to learn

## What Could Have Been Better

- **Go Lang mention** — briefly mentioned Go but no examples were asked or given; could have been explored
- **Security tooling gaps** — acknowledged wanting to learn Okta, phishing-resistant auth, MFA deeply; this could raise concerns for a security-focused role (experience gap)
- The role is significantly more support-oriented than Ramana's current background — the motivation answer was solid but this remains a potential concern for the team

---

## Hiring Signal Assessment

**Verdict: Unknown (transcript cut off) — Conversation was positive**

The interviewer was engaged and asked follow-up questions throughout. The culture examples (feedback + courage) were strong and directly aligned with Netflix's stated values. The main open question is how the team evaluates the security tooling experience gap (Okta, phishing-resistant auth, MFA).

---

## Important Role Consideration

This role is a **significant shift** from a pure engineering/DevOps track:
- 80% support tickets vs. building infrastructure
- Security domain (not DevOps/SRE domain)
- Deep security tooling expertise expected (Okta, MFA, phishing-resistant auth, encryption)

Worth clarifying in next rounds whether there is a growth path back toward platform engineering or if this is permanently a support-focused track. Also worth understanding compensation expectations given the support-heavy nature vs. engineering roles.

---

## Key Netflix Culture Notes

- **Feedback**: Daily, peer-to-peer, upward to leaders and skip levels — considered a gift, not a review process
- **Courage**: Expected to speak unpopular opinions to any audience, including VPs — with data and reasoning, not just opinions
- Culture memo is worth reading before next rounds
