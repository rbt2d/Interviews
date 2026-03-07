# NVIDIA Interview — Round 3: Security & DevOps Engineer

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | February 18, 2026                                                    |
| **Duration**    | 51 minutes                                                           |
| **Interviewer** | Anand — 14 years at NVIDIA, manages Information Security and DevOps for the Marketplace |
| **Role**        | Senior DevOps Engineer — Individual Contributor (Remote)            |
| **Outcome**     | **Passed** — Anand explicitly said *"I'm going to give thumbs up to Jose"* and *"I would love to see you, definitely, no doubt about it"* |

---

## About the Interviewer

Anand has been at NVIDIA since 2011 (started when NVIDIA had ~6,000 employees, now ~45–50k). He came from an information security background and now manages both InfoSec and DevOps for the marketplace. He was on the interview panel specifically to assess AWS depth, security knowledge, and intellectual honesty (failure/disagreement stories). He cares deeply about **mindset, discipline, and right attitude** over raw technical skills.

---

## Topics Covered & Answers

### 1. AWS Day-to-Day Activities
- Starts day with CloudWatch dashboards and AWS Config compliance reports
- Multi-account setup (separate accounts for dev/staging/prod), AWS Organizations with SCPs for security boundaries
- Manages VPCs: private subnets for application workloads, public for load balancing, dedicated data subnets for RDS/ElastiCache
- EKS: node groups, pod security policies, IRSA (IAM Roles for Service Accounts)
- RDS: Multi-AZ, automated backups, encryption at rest; DynamoDB with PITR and encryption
- Security: IAM least privilege, Secrets Manager for credential rotation, VPC endpoints, CloudTrail for audit logging, AWS Config + GuardDuty
- CICD: CodeBuild integrated with Jenkins, artifacts in S3 with bucket policies and encryption, Lambda for automation and compliance remediation

### 2. Terraform Multi-Environment Automation
- Terraform workspaces + environment-specific variable files
- Same reusable modules, different configs per env (instance sizes, replica counts)
- Jenkins pipeline: auto-triggers on GitLab push → `terraform plan` on PR (output posted as comment) → `terraform apply` on merge using saved plan file
- State: S3 backend with DynamoDB locking, separate state file per environment (naming: `terraform-state-{env}-{service}`), S3 versioning + MFA delete + access logging

### 3. Who Writes Terraform Code
- Ramana owns foundational infrastructure (VPC, EKS, RDS modules)
- Developers contribute application-specific resources with guidance
- Senior engineer leads TerraForm architecture, security, CICD standardization; two other senior engineers own monitoring/incident response and Jenkins automation

### 4. Migrating 180 Repos from Click-Ops to IaC
Anand's real scenario — NVIDIA currently does most infrastructure manually via console.
- **Step 1**: Audit using Terraform/AWS Config to generate resource inventory
- **Step 2**: Categorize by risk and change frequency — prioritize what breaks most, what's hardest to recreate
- **Step 3**: Start with Dev, not Prod — pick 5–10 pilot repos representing common patterns (web service, DB-heavy service, Lambda-heavy service)
- **Step 4**: Use `terraform import` to bring existing resources under management — fix security issues during migration (tighten overly permissive security groups, enable encryption)
- **Step 5**: Build CICD pipeline that enforces new patterns — no more console access, everything goes through code review
- Honest timeline: months, not weeks — communicate incremental value

### 5. GitLab → Jenkins → AWS Architecture
- GitLab with branch protection rules (no direct pushes to main, all changes via MRs with mandatory code review)
- GitLab webhooks trigger Jenkins jobs automatically on push
- Jenkins runs on AWS
- Code review done by senior engineers — not the person committing the change
- Dev → QA: automatic; QA → Stage: auto on tests passing; Stage → Prod: manual approval gate (authorized senior engineers/leads only, different person from change author)
- Deployment windows enforced: business hours only, never Friday, never before holidays
- Production deployments require documentation: what's changing, expected impact, rollback strategy

### 6. SAST / DAST Security Controls
**SAST (Static Application Security Testing):**
- SonarQube integrated into Jenkins pipeline — every commit scanned for vulnerabilities, code smells, injection flaws
- Custom SonarQube rules for Java/Spring, Python, and Infrastructure as Code patterns

**DAST (Dynamic Application Security Testing):**
- OWASP ZAP integrated into staging environment after deployment
- Tests running application for authentication bypass, exposed sensitive data, malicious payload injection

### 7. Real Vulnerability Example — Information Disclosure
- **Discovered via**: Routine OWASP ZAP scan after deploying new payment processing service to staging
- **Vulnerability**: API returning raw SQL error messages including internal database schema and connection strings on failed queries — classic information disclosure
- **Root cause**: Java Spring Boot application not properly handling DB exceptions; default exception handling bubbling up raw SQL errors to API response
- **Fix**:
  1. Worked with dev team to implement custom exception class — logs detailed error internally, returns generic message to client
  2. Added global exception handler in Spring to sanitize all error responses
  3. Updated SonarQube rules to flag similar information disclosure patterns in future code reviews
  4. Added specific DAST test case for information disclosure in automated security testing
- **Outcome**: Prevented major security vulnerability from reaching production; led to security training for dev team; information disclosure testing became standard in security code reviews

### 8. Security Team Structure
- Dedicated InfoSec team defines security standards (encryption policies, access policies)
- Ramana acts as security champion for DevOps/infrastructure domain
- Implements security controls in CICD pipeline, configures GuardDuty and AWS Config, ensures Terraform modules follow security best practices
- Monthly security reviews where InfoSec audits infrastructure and provides guidance on emerging threats
- Model: **partnership, not a handoff** — InfoSec has security expertise, Ramana has Kubernetes networking and Terraform depth

### 9. AWS WAF & Web Application Firewalls
- Familiar with AWS WAF, Akamai WAF, EdgeKV
- Created rate-limiting rules to prevent DDoS (per-source-IP request thresholds per minute, different thresholds per endpoint — stricter on login pages)
- Implemented geo-blocking (US-only services block traffic from other countries)
- Rules to block requests with suspicious patterns: SQL keywords in URL params, script tags in form submissions
- Used WAF rule-based configurations with automatic request counting and blocking

### 10. Failure Story — Terraform RDS Module Import
- **Situation**: Building new RDS module for multi-region database co-editing; confident in IaC skills, eager to demonstrate value quickly
- **What went wrong**: Module worked perfectly for *creating* new databases but hadn't properly tested the *import* process for existing databases. During staging migration, Terraform treated configuration differences as requiring recreation — attempted to destroy and recreate the staging database instead of modifying it
- **Caught it**: Before production — but lost several hours of staging data and had to restore from backup
- **Outcome**: Admitted mistake to manager and team; implemented mandatory import testing for all infrastructure modules; created detailed runbooks for database migrations; established practice of always testing state operations in isolated environments first
- **Key learning**: Confidence overriding proper validation procedures — now questions own assumptions and validates the things he "knows for certain"

### 11. Disagreement with Manager — Security Scanning in CICD
- **Disagreement**: Proposed integrating vulnerability scanning (SAST) into Jenkins with builds failing on critical CVEs. Manager wanted informational-only scanning — concerned about slowing down delivery schedules
- **Resolution**: Instead of arguing from principle, gathered data. Spent a week analyzing historical incidents — found 30% of production issues in the previous quarter were vulnerabilities catchable by automated scanning. Researched industry data showing fixing in production costs 10x more than in development
- **Outcome**: Presented analysis + phased implementation plan (warnings first → gradually add blocking for critical issues with emergency escape hatches). Manager approved the phased approach
- **Key insight**: Data beats principle in leadership conversations

### 12. Why Leaving AT&T
- Maxed out impact within constraints of large enterprise environment
- Innovation requires months of approvals; new tools take months to get approved
- NVIDIA has the opposite problem: business moves faster than infrastructure — exactly the challenge Ramana wants
- Track record shows can move fast while maintaining 99.99% deployment success rate
- AT&T's caution is justified (telecom outages affect millions) — but ready for an environment where speed is a core value, not the exception

### 13. Visa Status
- On EAD (Employment Authorization Document / Green Card process)

---

## What Went Well

- **AWS depth** was thorough — covered security, networking, compute, data, compliance in a single answer without prompting
- **Terraform migration strategy** matched NVIDIA's exact situation (180 repos, all click-ops) — answer was structured, practical, and realistic
- **Information disclosure vulnerability story** was specific and well-structured (situation → vulnerability type → fix → outcome → process improvement)
- **Failure story** showed genuine self-awareness and intellectual honesty — Anand specifically cares about this
- **Disagreement story** demonstrated data-driven approach rather than stubbornness
- **Why leaving** answer was honest and connected directly to NVIDIA's culture
- Anand explicitly gave a **thumbs up** and shared insider tips for the next two rounds — a strong trust signal

## What Could Have Been Better

- **WAF answer** initially vague — Anand had to ask follow-up "you implemented them in AWS using WAF?" to pin down the actual implementation. Should have led with tool names and specific implementations
- **Security scanning answer** was good but took a moment to land on OWASP ZAP by name — could have been crisper
- Could have asked more specific questions about Anand's security standards/tooling at NVIDIA given his InfoSec background

---

## Hiring Signal Assessment

**Verdict: VERY STRONG POSITIVE — Explicit Thumbs Up**

Anand broke protocol and gave direct feedback: *"I really like the way you come up with your answers"*, *"I'm going to give thumbs up to Jose"*, *"would love to see you, definitely, no doubt about it"*.

He also **shared insider prep tips** for the remaining two rounds — a rare and very positive signal:

---

## Insider Tips from Anand for Remaining Rounds

### Round 4 — Monish (HackerRank / Coding)
- Will likely use **HackerRank** with Python (possibly Go)
- Focus: **logical thinking and analytical problem solving**
- Brush up: core Python, data structures, algorithm patterns (LeetCode medium), clean problem-solving approach
- Monish does Python himself so evaluates Python-specific solutions well

### Round 5 — Dale (Observability / SRE)
- Dale comes from **observability and SRE background** — Datadog, monitoring, alerting
- Focus: Creating alerts, rotating cases, monitoring strategy, SLO/SLA implementation
- Brush up: Datadog alert types (threshold, anomaly, composite), error budget, SLO burn rate alerts, incident response runbooks

---

## Action Items Before Next Rounds

- [ ] LeetCode medium Python problems — focus on arrays, strings, hashmaps, greedy
- [ ] Review boto3 AWS SDK patterns (common automation scripts)
- [ ] Datadog deep dive: alert configuration, monitor types, SLO setup, watchdog
- [ ] Review incident response runbook structure
- [ ] Prepare specific Datadog dashboard and alerting examples from AT&T
