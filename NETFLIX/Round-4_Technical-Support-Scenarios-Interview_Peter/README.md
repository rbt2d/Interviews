# Netflix Interview — Round 4: Technical Support Scenarios

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | March 3, 2026 (same day as Round 3 — back-to-back sessions)         |
| **Duration**    | 45 minutes                                                           |
| **Interviewers** | **Peter E** (Java SME, ESO, ~14 months at Netflix; former software engineer) + **Peter D** / Peter Dalbanjan (CICD/Delivery focus, ESO, ~9 months at Netflix; former AWS Solutions Architect + Sales) |
| **Format**      | 3 customer-emulation troubleshooting scenarios + Q&A                 |
| **Outcome**     | **Positive** — Peter E: *"Best of luck for the rest of the rounds"* — more rounds expected |

---

## About the Interviewers

**Peter E** — Java SME in ESO. Answers Java/Spring/Spring Boot questions and some CICD. Was a software engineer before Netflix. Excited about: the caliber of problems AND working with people he used to go to conferences to hear.

**Peter D** (Dalbanjan) — CICD/Delivery specialist in ESO. ~9 months at Netflix. Background: sysadmin → sales → solutions architect at AWS (~10 years) → Netflix. Excited about: colleagues and infinite depth of learning. Noted that Netflix's abstractions are 10 steps ahead of what he was recommending to AWS customers 5 years ago.

**Format note:** Peter E was explicit — *"I am acting as a customer. I am not fully concerned about technical answers. I am looking for how you're approaching the situation, with and without the information you have."*

---

## Scenario 1: CICD Pipeline Failure

**Setup:** User reports a CICD pipeline failure. Pipeline could be Jenkins or GitHub Actions.

### Ramana's Approach

**Clarifying questions first:**
1. Has this pipeline worked before, or is it brand new? (Regression vs. first-time config issue)
2. What changed recently? (Code push, dependency update, pipeline config change, or nothing obvious?)
3. What error is the user actually seeing? (Specific error, stack trace, timeout, or just "failed"?)
4. What's the blast radius? (Just them, or the whole team blocked from deploying?)

**After context, go to build logs:**
- Find the first point of failure — which stage broke? (Checkout, dependency install, compilation, test, deployment)
- Each failure stage points in a different troubleshooting direction
- Pause and check in with the user before going deeper

### Follow-up 1: User Inherited the Pipeline and Knows Nothing About It

**Ramana's shift:** Move from asking the user questions → letting the artifacts tell the story

- The **pipeline config** (Jenkinsfile / GitHub Actions workflow) is its own documentation — read it to understand intent step by step
- **Git history** on the pipeline config: when was it last modified, by whom, what changed?
- **Build history**: when was the last successful run? Diff that run vs. the failed one — often tells you exactly what broke
- If pipeline touches infrastructure: trace dependencies independently (IAM role in config → CloudTrail logs → was the assume-role call succeeding?)
- Keep user in the loop: *"Hey, I can see from the pipeline config that it's doing X. Let me look at the recent run and I'll get back to you with what I find."*

**Key insight:** *"My approach shifts from interview-style with the customer to investigating on my own — but I keep them in the loop so they know I'm not blocked just because they don't have the answers."*

### Follow-up 2: Root Cause Is a Platform-Level Misconfiguration

**Communication to platform team:**
- Reach out with full context, factual and collaborative — not accusatory
- *"I've been working a ticket where a user's pipeline is failing. I traced it to this specific platform-level change. Here's the evidence [CloudTrail entry / config diff], here's the error it's causing, here's the user impact. Can you confirm if this was intentional, and if so, what's the expected path forward for users?"*
- Frame it as: here's what I found, let's work together — because sometimes platform changes are intentional with poor communication, sometimes it's an unintentional side effect

**Communication to the affected user:**
- Close the loop clearly: *"I found the root cause. It's a platform-level configuration change. It's not anything you did. The platform team is aware and working on it. In the meantime, here's a workaround to get unblocked right now."*
- User doesn't care about internal back-and-forth — they want to know what happened, that it's being handled, and what they can do short-term

**Proactive prevention (broader impact):**
- Post a notice in relevant Slack channels: *"We've identified an issue with X that may affect pipelines. Here are the symptoms, here's the workaround while the fix is in progress."*
- Reduces incoming ticket volume, saves other users from having to wait
- Once resolved: follow up with confirmation the fix is in place
- Document the whole thing (root cause, symptoms, workaround, resolution) — so the next occurrence is a 5-minute lookup, not a 40-minute investigation

---

## Scenario 2: Should I Roll Back a Broken Deployment?

**Setup:** User deployed an application, it's broken. They're asking: should I roll back?

### Initial Questions
- How broken is it? Completely down vs. partial functionality? (Determines urgency and time pressure)
- What was in this deployment? Code change, config change, dependency update?

### Key Follow-up: Previous Deployments of This Version Worked

**Ramana's reasoning:**
> *"If this same version was deployed before and it worked, then the code itself isn't the problem. Rolling back to the same version isn't going to fix anything. That tells me something in the **environment** changed between the last successful deployment and this one."*

**Investigation shift — focus on what's different around the application:**
- Did a platform dependency change underneath them?
- Is a config value different in this environment now?
- Did a downstream service change its API or behavior?
- Were credentials or certificates rotated?

**Recommendation:** *"Hold off on the rollback for a moment — if the version isn't the issue, it won't help. Let me pull the deployment logs and check for recent infra changes. Let's compare what's different between the last successful deployment and this one."*

**Caveat:** *"But if users are severely impacted right now and we can't figure it out quickly — rolling back to an even earlier known-good version might still be the right call just to restore service, while we investigate the root cause separately."*

---

## Scenario 3: Unreachable AWS EC2 Instance

**Setup:** User reports an EC2 instance is unreachable.

### Clarifying Questions First
- Which AWS account?
- What's the instance ID?
- How are they trying to reach it? (SSH, HTTP, RDP, application port)
- Have they ever been able to reach it, or is this brand new?
- What error are they seeing? (Connection timeout vs. connection refused)

**Why the error type matters:**
- **Timeout** → traffic isn't reaching the instance at all — network layer issue
- **Connection refused** → traffic is getting there but nothing is listening — service not running

### Troubleshooting Path (Layer by Layer)

**1. EC2 instance health checks (console)**
- System status check failing → hardware issue on underlying host → stop/start to migrate to new host
- Instance status check failing → OS-level issue (OOM, disk full, OS crash)

**2. Security groups**
- Does the attached security group allow inbound traffic on the target port from the source IP/CIDR?
- One of the most common issues: security group rule changed, or user connecting from a different network than what's whitelisted

**3. Network ACLs (NACLs)**
- NACLs are stateless — even if inbound is allowed, if the ephemeral port range for outbound responses is missing, traffic can't return
- *"People forget this one a lot"*

**4. Route tables**
- Public instance: Internet Gateway attached? Route to IGW in route table?
- Private instance: NAT gateway or VPN path?
- Does the instance have a public IP or Elastic IP if being accessed from outside VPC?

**5. VPC Flow Logs (if everything looks correct)**
- Flow logs show whether traffic is being accepted or rejected at the network interface level
- Traffic rejected → confirms security group or ACL issue
- Traffic not appearing at all → upstream problem (DNS resolution failure, wrong IP, routing issue before it reaches the VPC)

---

## Behavioral Question: Prioritizing Team/Customer Needs Over Your Own

**Story: SAML Certificate Rotation Incident — AT&T (Friday Evening)**

**Situation:**
- Major certificate rotation on the SAML Identity Provider didn't propagate correctly
- Service provider configurations got out of sync
- ~200+ developers across multiple teams started losing access to internal tools
- Tickets flooding in fast
- Ramana was technically wrapping up for the day (not primary on-call for that week)

**Actions taken:**
- Spotted the pattern from early symptoms — had seen similar certificate metadata sync issues before
- Jumped in voluntarily: *"Let me take the lead on this, and I'll walk you through it so you can learn the process"*
- Pulled IDP logs, confirmed new certificate hadn't propagated to service provider configs
- Coordinated with 3 different platform teams to push metadata updates
- Full service restored in ~40 minutes from first report

**What Ramana is most proud of — what happened after:**
- Spent the weekend writing a **detailed runbook** for certificate rotation events
- Set up **monitoring alerts** for when certificate metadata is out of sync across providers (proactive detection)
- Ran a **knowledge sharing session** the following week so the whole team understood the pattern
- Next time it happens, whoever is on call can handle it without escalating

**Key principle demonstrated:** *"I'd rather invest time upfront than be the single point of knowledge on something this critical."*

---

## Ramana's Questions & Answers

### Q1: What's most exciting about Netflix day-to-day?

**Peter E:**
> *"Two things: the questions and problems we get — some are business as usual but there are those gems. And the people I'm now working with are the people I used to attend conferences to hear."*

**Peter D:**
> *"Fantastic colleagues — the depth Netflix engineers have is amazing. And infinite things to learn — Netflix has taken 10 steps ahead of what I was suggesting to AWS customers 5 years ago. The level of abstractions keeps me excited."*

### Q2: Peter D — Biggest shift from AWS to Netflix?

Peter D spent ~10 years at AWS as a Solutions Architect (high-level customer success, presentations, talks) before joining Netflix ESO (deeply hands-on daily troubleshooting). The context switching at Netflix is at a completely different level — still something he actively manages. The deep AWS foundation is a huge advantage for infrastructure issues, but the Netflix-specific abstractions require real ramp-up time.

### Q3: Peter E — Best advice for someone new to the role?

> *"Just be opinionated. Partner teams want opinions — if it's good, bad, or could be better. If you just say 'this looks good' and nothing else, that doesn't work at Netflix. Be more opinionated."*

### Q4: Recurring ticket patterns → permanent fix — how much ownership does the support team have?

**Peter E's answer:**
- All options are on the table: contribute to the internal product, improve documentation, create runbooks, change platform configs
- It's case by case — is it recurring because the platform is wrong? Because the environment is wrong? Because docs are unclear?
- Full IC autonomy to approach issues however feels best fit
- *"Just make sure the right thing is happening"*

**Ramana's connection:** At AT&T, ~35% of tickets were the same IAM misconfiguration repeating. Built a self-service validation tool in Python that developers could run before deploying. Created a documentation library with common error patterns and decision trees. Repeat rate dropped from ~12%. *"I love that the team here has the same mindset — don't just close the ticket, eliminate the reason."*

### Q5: AI for support?

**Peter E:** Identifying common patterns and providing initial triage have been explored. It is a tool and they will utilize it.

**Peter D:** Seeing early success — AI is good at increasing productivity in troubleshooting and finding resolutions. Still early stages.

**Ramana's insight offered:** Real opportunity is feeding resolved ticket history so AI learns the specific Netflix environment and patterns, not just generic troubleshooting. Also mentioned AT&T experience with AI log analytics — surfacing most likely root cause from stack traces saved significant initial triage time.

### Q6: First 90 days — what does success look like?

**Peter D's answer:**
- **First 2 weeks**: Ramp up. Learn the abstractions. Netflix builds everything at a level far beyond most orgs. AWS knowledge helps but internal tools are different.
- **Early impact**: How you communicate with end users. Are they understanding your responses? Are you resolving in a timely fashion? Gelling with team culture.
- **Most important in 90 days**: Partner relationships. For CICD, that's the delivery team. They'll help you initially but the dependency curve should go **down**, not up — you need to be taking more on yourself over time.
- **Broader impact**: Eventually expanding beyond direct partners to multiple other teams.

---

## What Went Well

- **Structured approach to every scenario** — clarifying questions first, then investigation → root cause → communication → prevention
- **Layered EC2 troubleshooting** was technically complete — security groups, NACLs (explicitly calling out the stateless gotcha), route tables, VPC flow logs in the right order
- **Platform-level misconfiguration answer** was strong — separated communication to platform team (collaborative, evidence-based) from communication to user (plain language, action-focused) from proactive prevention (Slack announcement, documentation)
- **Rollback scenario** — correctly identified that same-version rollback doesn't help if the version already worked; shifted investigation to environment differences
- **Certificate rotation incident** — compelling behavioral story: volunteered outside primary on-call hours, mentored the newer engineer, fixed the issue, AND spent the weekend building permanent infrastructure around the incident (runbook + monitoring + knowledge sharing)
- **Questions asked** showed genuine cultural alignment — AI tooling, recurring patterns → permanent fixes, autonomy via "context not control" — all resonated with both interviewers
- **Self-service tool story** (Python IAM validation tool, 12% repeat rate reduction) was a concrete, numbers-backed example of the permanent fix mindset

## What Could Have Been Better

- **Rollback answer** initially started with general questions before the key insight (same version worked before → rollback won't help). Could have led with the logic more directly
- **AI discussion** — Ramana offered a strong insight (train on resolved ticket history, not generic data) but it came as a suggestion at the end rather than a question — could have been framed as a sharper question to invite more dialogue

---

## Hiring Signal Assessment

**Verdict: Positive — More Rounds Expected**

Peter E's closing: *"Best of luck for the rest of the rounds"* — confirms the process continues. Both Peters were engaged throughout, asked follow-up questions that built on previous answers, and gave substantive responses to all of Ramana's questions. The behavioral question (certificate rotation incident) was the emotional highlight — Peter D particularly validated the "don't be the single point of knowledge" philosophy.

The back-to-back nature of Rounds 3 and 4 on the same day (March 3) suggests the process is accelerating — which is generally a positive signal.

---

## Key Process Notes

- **Ticket volume reduction is valued** — the team explicitly cares about not solving the same ticket 50 times. Self-service tools, documentation improvements, and runbooks are all things support ICs are expected and empowered to own
- **Partner relationships are critical** — the delivery/CICD team (Peter D's main partner) will be Ramana's most important relationship to build early. Dependency on them should decrease over time, not increase
- **Opinionated communication is expected** — don't just say "looks good." Partner teams want real feedback.
- **Context not control is real** — PR directly to any of the 50-60K internal GitHub repos is legitimate and encouraged. No process gatekeeping.
