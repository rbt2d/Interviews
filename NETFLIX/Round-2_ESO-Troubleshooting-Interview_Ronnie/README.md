# Netflix Interview — Round 2: ESO Troubleshooting Interview

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | February 3, 2026                                                     |
| **Duration**    | 59 minutes                                                           |
| **Interviewer** | Ronnie Moore — ESO (Engineering Support Org), SPA Pod, Dallas TX (Remote) |
| **Role**        | Security Support Solutions Engineer — ESO SPA Pod                   |
| **Format**      | Intro + 4 Troubleshooting Scenarios + Q&A                           |
| **Outcome**     | **PASSED** — Ronnie: *"You did great. I'm going to put in feedback after the call."* Next step: Security partner team interviews |

---

## About the Interviewer & Team

**Ronnie Moore** has been at Netflix for ~1.5 years. Started on the Productivity Engineering team (Engineering Help channel) and expanded to also cover the SPA (Security, Privacy and Assurance) pod. Currently one of 3 team members on the ESO SPA pod (all borrowed from other teams). Peter joined Jan 26 as the first dedicated hire; Ronnie was his onboarding buddy.

### ESO SPA Pod — How It Works

| Aspect | Detail |
|--------|--------|
| **Primary tool** | Slack via "Unthread" (Slack-native ticketing workflow) |
| **Users** | Internal Netflix engineers and developers only |
| **Ticket intake** | Developer picks a tool category → thread opens in the right Slack channel (compute help / engineering help / security help) |
| **Resolution model** | ~30% closed by AI bot; ~50% resolved independently by ESO; ~20-30% escalated to partner engineering teams |
| **Infrastructure** | 95% AWS; small Azure/GCP presence |
| **Support hours** | 9am–5pm Pacific; NO on-call pager duty |
| **On-call owners** | Platform/infrastructure engineering teams (not ESO) |
| **Team size** | 3 currently (US-based); Poland ESO office recently opened for engineering help coverage |
| **Growth direction** | Follow-the-sun coverage model being built across time zones |
| **AI in workflow** | Enterprise ChatGPT, Claude, Gemini, internal search AI, AI-first ticket triage |

---

## Introduction Ramana Gave

Tailored perfectly for this role:
- Senior SRE/Support Solutions Engineer hybrid — 10 years of experience
- Supporting security and identity infrastructure for AT&T developer community (50M+ users)
- Day-to-day: Troubleshooting auth/authz issues — Okta configs, OAuth flows, SAML integrations, FIDO2 implementations; Splunk and CloudTrail log investigation
- Built Python and Go automation tools reducing manual support work by 40%
- Created runbooks enabling developer self-service
- UMKC cloud security + IAM for research platforms; Black Box authentication for logistics platforms
- Most recently: phishing-resistant MFA rollout and device posture validation

---

## Troubleshooting Scenarios

### Scenario 1 — IAM Role: Service Can't Assume S3 Role (Access Denied)

**Problem:** Developer reports their service can't assume an S3 role. Logs show Access Denied.

**Ramana's approach:**

**Clarifying questions first:**
- Is this new behavior or did it work before? (Distinguishes config issue from regression)
- What type of service is trying to assume the role? (EKS pod vs Lambda vs EC2 — mechanisms differ)

**Investigation steps:**
1. Check CloudTrail logs → find the exact denied API call
2. Determine: is it failing on `AssumeRole` itself, or after assuming the role when accessing S3?
   - Failing on `AssumeRole` → trust relationship issue
   - Failing after `AssumeRole` → permission policy doesn't have S3 actions
3. Verify basics: does the role exist? Is the Role ARN correct in config/code?
4. If EKS pod + IRSA: is the service account annotation pointing to the right role?
5. Check for resource-based policy on the S3 bucket blocking access

**Ronnie's feedback:** *"Spot on. Great clarifying questions. IAM boundary (SCPs/permissions boundary) is another angle to check too."*

---

### Scenario 2 — S3 Bucket with Public Read Access (Security Scan Finding)

**Problem:** A security scan reports an S3 bucket has public read access.

**Ramana's approach:**

**Before jumping to remediation — understand context:**
- Is this intentional? (Static website hosting, public documentation are legitimate use cases)
- If not intentional → treat as security incident requiring immediate attention

**Investigation:**
1. Check bucket policy for `Principal: *` with `s3:GetObject`
2. Check bucket ACL for Public Read setting
3. Check Block Public Access settings at bucket AND account level (both can be bypassed)
4. Determine blast radius: what data is in the bucket? PII? Secrets? Or test files? → determines urgency
5. CloudTrail: who made the change and when? (Intentional? Accident? Malicious?)

**Remediation:**
- Enable Block Public Access if it shouldn't be public
- Remove overly permissive bucket policy/ACL
- Investigate root cause to prevent recurrence (better IaC templates, pre-deployment scanning guardrails)
- If bucket was unexpectedly public, consider credential rotation for any service writing to it
- If false alarm → work with security scanner team to whitelist known-public buckets

**Ronnie's additions:** Also consider pulling in legal if PII was exposed. Whitelist legitimate public buckets in the scanner to prevent repeat alerts.

---

### Scenario 3 — OAuth2 `invalid_client` Error After Recent Update

**Problem:** Internal application failing to authenticate users via OAuth2, getting `invalid_client` error after a recent update.

**Ramana's approach:**

**Key clue:** "After a recent update" → narrow down what changed.

**Clarifying questions:**
- What was updated? Application code? Library version? Were credentials rotated? Did something change on the IDP side (Okta)?

**Root cause analysis — `invalid_client` means the auth server doesn't recognize or can't authenticate the client:**

| Likely Cause | Investigation |
|---|---|
| Client ID or secret wrong/expired | Verify config matches what's registered in Okta/IDP; check if secret was rotated and fat-fingered |
| Library update changed auth method | New library version might send credentials in header vs body; check IDP expectations |
| Client registration modified | Someone may have changed client type (confidential → public) or removed allowed grant types |
| IP allowlist enabled on IDP | Application IP not on the list |
| OAuth library breaking change | Redirect URI wildcard pattern no longer matches current environment |

**Steps:**
1. Check application config → client ID + secret match IDP registration
2. Verify credentials sent correctly: authorization header (Basic Auth) or request body (per IDP requirements)
3. Check IDP logs for the authentication attempt and specific rejection reason
4. Check if multiple users affected or just one (platform issue vs individual config)
5. If library was updated → check release notes for breaking changes in credential handling

**Explaining to a non-technical user:** Tailored communication by skill level:
- Senior engineer: *"Your client secret expired, generate a new one in Okta, update your app config, redeploy"*
- Newer engineer: Step-by-step walkthrough with explanation of *why* (the "password" your app uses to prove its identity expired), link to docs, follow up to confirm they restarted the service and updated ALL environments (Dev/Stage/Prod)

**Ronnie's feedback:** *"Great answer. Redirect URIs not matching current workspace patterns is another common one. Teaching them how to fish based on skill level is exactly right."*

---

### Scenario 4 — Employee Can't Enroll in FIDO2/WebAuthn MFA (Generic Error)

**Problem:** Employee unable to enroll in FIDO2 WebAuthn-based MFA. Generic error message. Blocked from sensitive systems. High urgency.

**Ramana's approach:**

**Failure can be at browser, device, authenticator, or IDP policy level:**

**Clarifying questions:**

| Question | Why It Matters |
|---|---|
| What browser and version? | WebAuthn support varies; older versions may not support it |
| Managed corporate laptop or personal device? | Enterprise browser restrictions/security policies may block WebAuthn APIs |
| Physical security key (YubiKey) or platform authenticator (Touch ID, Windows Hello, Face ID)? | Different failure modes for each |
| Is the key new or previously used? | Key may be faulty, uncertified, or need a PIN set first |
| Is it affecting just this person or the whole team? | Determines if it's a platform-wide issue |

**Browser console errors to look for:**
- `NotAllowedError` → user gesture requirement not met
- `NotSupportedError` → browser doesn't support the credential type
- `InvalidStateError` → authenticator already registered

**IDP investigation:**
- Check Okta admin console logs for the enrollment attempt
- Look for policy preventing FIDO2 enrollment for this user's group
- Check device trust requirements — does the device meet the policy?

**Isolation steps:**
- Try a different browser (Firefox → Chrome) to isolate browser-specific issues
- For physical key: try a different USB port; check if key needs a PIN set before enrollment
- Check if there's a broader platform incident

**Urgency management:**
- While troubleshooting, explore if a temporary workaround exists (different MFA method) to unblock access to sensitive systems
- Check the incidents channel for upstream IDP issues

**Ronnie's feedback:** *"You went everywhere I would go. Checking the pin is a great specific example. Jumping on a Google Meet to watch them do it is another approach for complex cases. Screen recording if different time zone."*

---

## Why Netflix Answer

- Culture memo read — *"freedom and responsibility"* and *"context, not control"* resonated
- Dream Team concept: wants to be pushed by top engineers, receive candid feedback
- Building the ESO SPA pod from the ground up — early member shaping security support at Netflix
- Security at scale (authentication/identity for ~500M users globally) is the right problem space
- Phishing-resistant MFA at that scale is exactly where Ramana wants to build expertise
- AT&T's pace and process overhead is the push; Netflix's speed and ownership is the pull

---

## Ramana's Questions & Ronnie's Answers

### Q1: What does onboarding look like? How long until independent ticket-taking?

**Ronnie's answer:**
- Structured 30-day Airtable checklist: Day 1 (HR), Week 1 (Slack channels, recurring cadences), Week 2+ (shadowing and reverse shadowing)
- Peter (started Jan 26) was already routing tickets independently in week 1
- Ronnie started taking tickets on Day 1 when he joined — "whatever works for you"
- SPA Learning Channel: Ronnie posts case learnings (useful curl commands, new APIs, routing decisions) so new members can reference real case history
- Flexible ramp-up based on the person

### Q2: On-call and after-hours expectations?

**Ronnie's answer:**
- **No on-call for ESO** — 9am–5pm Pacific, period
- After-hours tickets sit in the queue until 9am Pacific
- P1 paging goes to the infrastructure/platform teams that own those systems
- Users can self-select to page the on-call, but it routes to the engineering team, not ESO
- Travel: Once or twice a year to Los Gatos, CA for team meetups
- Poland office recently opened → building toward true follow-the-sun coverage

### Q3: What AI tooling does Netflix use?

**Ronnie's answer:**

| Tool | Usage |
|------|-------|
| Enterprise ChatGPT | Throw in logs/stack traces, debug, doesn't train on Netflix data |
| Claude, Gemini, GPT-4 variants | All available and supported |
| AI ticket triage bot | First responder on tickets — ~30% closure rate; trained on Netflix docs, Slack threads, Confluence |
| Internal AI search | Google-like interface searching Confluence, Slack threads, Google Docs simultaneously |
| Read.ai / Otter | Meeting transcripts analyzed by Gemini |
| Manuals search portal | Searches all internal tool documentation |

**Impact of AI triage:** Handles repetitive known issues, especially after platform changes. ESO gets the harder, more complex, unique problems that require real troubleshooting. Team actively trains the bot (good answer/bad answer feedback). Hallucinates occasionally — actively managed.

---

## What Went Well

- **All 4 scenarios handled confidently** — Ronnie had nothing material to add after any answer
- **Clarifying questions habit** — explicitly praised for asking what changed, is it new, what service type, what's the blast radius — *"You asked a lot of good clarifying questions straight away"*
- **CloudTrail instinct** — going straight to logs before assuming the cause impressed Ronnie
- **Communication tailoring** — different explanation styles for senior vs junior users praised as *"exactly right"* and *"teaching them how to fish"*
- **FIDO2 depth** — specific knowledge of WebAuthn error codes (NotAllowedError, NotSupportedError, InvalidStateError) and PIN requirements for physical keys
- **Introduction** was perfectly tailored — mentioned Okta, OAuth, SAML, FIDO2, phishing-resistant MFA, Splunk, CloudTrail — hit every keyword relevant to this role
- **Questions asked** were all excellent — showed genuine curiosity about the team, not just the job
- **Explicit pass:** *"You did great"* + *"I'm going to put in feedback after the call"*

## What Could Have Been Better

- Minor: On the S3 public access scenario, could have mentioned SCPs (Service Control Policies) at org level as another layer to check
- On OAuth2, redirect URI mismatch (Ronnie's addition) wasn't surfaced — good to add to the mental model

---

## Hiring Signal Assessment

**Verdict: STRONG PASS — Moving to Next Round**

Ronnie was enthusiastic throughout: *"I really like everything I've heard"*, *"That's right up your alley"*, *"You did great"*. He explicitly stated next steps: speaking to security partner team contacts and team leads for a deeper security background review.

The "Dream Team" and culture alignment answers landed well. The troubleshooting answers were strong enough that Ronnie said *"I hardly have anything to add here sometimes"*.

---

## Next Steps After This Round

- **Next round**: Security partner team contacts / team leads — deeper security background review
- Prep areas:
  - Deep dive on specific Netflix security tools mentioned (Okta admin console, FIDO2/WebAuthn, phishing-resistant MFA policies)
  - Be ready for more in-depth security scenario questions (beyond IAM/OAuth/S3/MFA basics)
  - Netflix culture pillars: be ready to demonstrate feedback and courage examples again for different interviewers
  - Review Netflix culture memo again before the security team rounds

---

## Key Netflix ESO Culture Notes from Ronnie

- *"The Dream Team is real"* — impressed by colleagues from day one, hasn't slowed
- Continuous improvement loop: Partner syncs with each engineering team → discuss escalations → how could we solve it without help next time? → improve docs, FAQs, app strings
- Access to 50–60K internal GitHub repos — can submit PRs to improve documentation or fix a broken link for a user
- *"Try to put yourself out of a job"* — automate the repeatable, do the more interesting work
- Slack-first culture — threads are the primary collaboration surface
