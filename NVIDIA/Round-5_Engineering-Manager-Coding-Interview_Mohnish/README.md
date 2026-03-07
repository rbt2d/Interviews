# NVIDIA Interview — Round 5: Engineering Manager + Coding/Architecture

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | February 19, 2026 (10:57 PM)                                        |
| **Duration**    | 1 hour 4 minutes                                                     |
| **Interviewer** | Mohnish — Engineering Manager, E-Commerce Team, NVIDIA              |
| **Role**        | Senior DevOps Engineer — Individual Contributor (Remote)            |
| **Format**      | HackerRank coding problem + AWS architecture design (whiteboard) + Terraform brownfield + behavioral |
| **Outcome**     | **Positive** — Mohnish was engaged and satisfied throughout; said *"that's pretty much how it's going to be"* about the future of DevOps/AI |

---

## About the Interviewer

Mohnish is an Engineering Manager for the **development team** (not DevOps) at NVIDIA E-Commerce. He was placed on the interview panel specifically to assess scripting, automation, and systems thinking. His team is responsible for all NVIDIA E-Commerce platforms: NVIDIA website, partner websites, marketplaces, mobile/desktop applications, and even in-store GPU demo software at Target. Tech stack: Angular, React, Python (serverless backend), Java APIs, SQL + NoSQL, all on AWS. He cares about **scalability, cost, revenue impact**, and **developers not being blocked by infrastructure**.

---

## Section 1: HackerRank Coding Problem

### Problem: Pipeline Stability Streak
**Problem statement:**
- `m` microservices report daily health check results as `"Y"` (passed) or `"N"` (failed)
- Given `m` (number of microservices) and `data` (array of strings, one per day)
- Find the **longest consecutive streak of days where ALL services passed** (all chars in the string are `"Y"`)

**Example:**
```
m=3, n=5
data = ["YYY", "YYY", "YNY", "NYY", "YNY"]
Output: 2  (first two days all passed)
```

### Ramana's Approach (Correct)
- Iterate through each day, check if `"N"` is absent from the string
- If all passed: increment `current_streak`, update `max_streak` if greater
- If any failed: reset `current_streak` to 0
- Return `max_streak`

**Time complexity:** O(N × M) — N days × M microservices per string check  
**Space complexity:** O(1) — only two variables

### Solution Written

```python
def pipeline_stability_streak(m, data):
    max_streak = 0
    current_streak = 0
    
    for day in data:
        if "N" not in day:
            current_streak += 1
            max_streak = max(max_streak, current_streak)
        else:
            current_streak = 0
    
    return max_streak
```

**Result:** Passed all test cases ✅ — Mohnish confirmed correct with "you're good"

---

## Section 2: AWS Architecture Design — Serverless Python API

### Requirements Gathered (Discovery Questions)
Before drawing, Ramana asked clarifying questions — Mohnish appreciated this approach:

| Question | Answer |
|----------|--------|
| Expected request volume? | Thousands of RPS; must handle GPU drops (traffic spikes) |
| API type? | REST API — Add to Cart |
| Data model shape? | Product SKU |
| Is product catalog read-heavy/eventually consistent? | Yes — can assume it's cached |
| Authentication required? | Good to have but open is fine for Add to Cart |
| Latency SLA? | Sub-second, max 1-2 seconds (P99) |
| How many engineers own this? | Max 2 developers |
| Single or multi-account? | **One AWS account, three environments** (Dev/Stage/Prod) |

### Architecture Designed

```
Client (Browser/Mobile)
    ↓
Route 53 (dev-store.nvidia.com / store.nvidia.com)
    ↓
CloudFront (WAF rules — bot protection for GPU drops, IP throttling)
    ↓
API Gateway (rate limiting: 10 req/sec/IP on Add to Cart, usage plans with burst limits)
    ↓
Lambda (Python — provisioned concurrency: 500 warm instances for GPU drops)
    ↓
DynamoDB (cart data — user_id PK, sku_id SK, on-demand capacity, single-digit ms reads)
    + ElastiCache/Redis (product catalog cache — sub-ms reads, 5-min TTL, cache-aside pattern)
```

### Key Design Decisions

**Why Lambda (not ECS)?**
- 2-person team cannot babysit an ECS cluster
- Zero server management, auto-scales from 0 to thousands
- Pay-per-request (costs nothing outside of GPU drops)
- Provisioned concurrency eliminates cold starts during high-traffic events

**Why DynamoDB (not RDS)?**
- Cart = perfect key-value workload (user_id → cart items)
- No connection pooling to manage (critical with Lambda — each invocation is fresh)
- On-demand capacity mode: auto-scales with zero configuration
- Single-digit millisecond reads/writes at scale
- Data model: `user_id` (PK) + `sku_id` (SK), attributes: quantity, price, added_at, TTL; GSI on `sku_id`

**Why ElastiCache/Redis for product catalog?**
- Product catalog is read-heavy and eventually consistent — perfect cache candidate
- Sub-millisecond reads; 5-minute TTL serves 99% of reads fresh
- Cache-aside pattern: validate SKU from Redis first → fallback to catalog DB on miss
- Eliminates DB reads for the most common path (SKU lookup)

**Provisioned Concurrency + Scheduled Scaling:**
- 500 pre-warmed Lambda instances always ready → handles first 500 RPS with <15ms latency
- Remaining requests auto-scale on-demand (with cold starts for overflow)
- Smart scheduled scaling via AWS Application Auto Scaling:
  - Register Lambda alias as scalable target
  - Define scheduled actions (e.g., pre-warm to 2,000 before a GPU launch date)
  - Supports cron for recurring patterns (business hours) or one-time events (specific launch dates)

**WAF + Bot Protection:**
- WAF rules at CloudFront layer (not API Gateway) for GPU drop bot protection
- Rate limiting: 10 req/sec/IP on Add to Cart endpoint
- Different thresholds per endpoint (stricter on cart, more lenient on static content)

### Single Account, Three Environments Strategy

Since NVIDIA uses one AWS account (not separate accounts per environment):
- **Resource naming**: every resource prefixed with environment (`dev-cart-table`, `prod-cart-table`)
- **IAM boundary isolation** (critical): each Lambda function assumes an environment-scoped IAM role with policy limited to only its own DynamoDB table — a dev bug literally cannot touch the prod table
- **API Gateway stages**: one API, three stages (dev/stage/prod), each with its own stage variables, throttling, logging level, caching config
- **Lambda aliases + versions**: single Lambda function with `dev`, `stage`, `prod` aliases — each alias has its own environment variables (table name, Redis endpoint), IAM role, concurrency limit, CloudWatch log group — code is environment-agnostic
- **Cost allocation**: every resource tagged with environment → AWS Cost Explorer filters by tag → know exactly what prod vs dev costs

### Infrastructure as Code — AWS CDK in Python
- Python CDK since NVIDIA is a Python shop (code and IaC in the same language)
- Reusable `CartServiceStack` construct takes `environment` as parameter
- Same construct instantiated three times with different configs
- API Gateway → Lambda proxy integration (full HTTP request passed: method, headers, path, body, query params)

---

## Section 3: Brownfield Terraform Adoption (Existing Manual Infrastructure)

**Scenario:** All infrastructure built manually over time, running in production. How to bring under Terraform management?

### Approach
1. **Brownfield import strategy** using `terraform import` or open-source Terraformer tool (~80% accurate code generation as baseline)
2. Reverse-engineer existing infra into `.tf` files, then refactor into clean reusable modules
3. Module structure: separate modules for Lambda service, API Gateway, DynamoDB tables
4. Environment configs via `terraform.tfvars` files per environment
5. Backend config in `backend.tf` per environment (separate state files)
6. Integrate with CICD after import — all future changes go through code review
7. Prioritize: start with less critical / less frequently changed resources, then expand coverage incrementally
8. Never attack everything at once — one service at a time, validate, then expand

---

## Section 4: Per-Branch Ephemeral Environments (QA Bottleneck)

### Problem
- 8–10 features ready for QA simultaneously
- Only one QA environment → features queue up → QA becomes bottleneck

### Solution: Feature Branch Environments (Ephemeral / PR Environments)
- Every open PR gets its own fully isolated, temporary environment automatically spun up
- Serverless makes this cheap and fast: create Lambda functions + configure stage variables = seconds + pennies (no EC2/EKS provisioning)
- CICD pipeline spins up environment on PR open, runs integration tests, posts URL back to PR as a comment
- QA clicks the link → tests the feature in complete isolation
- After PR merge → CICD pipeline destroys the environment automatically

**For S3-hosted front ends with custom domains:**
- Wildcard DNS record set up once: `*.qa.store.nvidia.com`
- Every new branch automatically gets a subdomain: `feature-xyz.qa.store.nvidia.com`
- No manual DNS changes per branch
- CloudFront with S3 prefix-based routing for frontend isolation

---

## Section 5: AI Workloads on AWS

Mohnish asked about AI deployment experience:
- Built RAG (Retrieval-Augmented Generation) POC at AT&T for internal knowledge base
- Employee asks natural language question → system retrieves relevant internal docs → LLM generates answer
- Stack: Amazon Bedrock + open-source vector store + Lambda
- Still learning; RAG didn't make it to production before starting job search

**Mohnish's AI context at NVIDIA:**
- 50% of development work is now shifted to AI tools (Cursor, etc.)
- Shareable workflows within AI tools — developer A shares prompts/business logic with developer B, who imports and continues at the same context level
- AI tool licenses are expensive; if the company pays for it, you're expected to use it — not using it is discouraged
- Next frontier: ML Ops pipelines — teams are building AI chatbots/models as POCs and will soon need DevOps to provide SageMaker and Bedrock deployment pipelines

---

## What Went Well

- **HackerRank problem solved correctly and efficiently** — clean solution, correct complexity analysis, passed all tests
- **Discovery questions before architecture** — Mohnish appreciated the structured approach; didn't just start drawing
- **Serverless-first thinking** matched NVIDIA's stack (Python serverless backend)
- **DynamoDB selection rationale** was strong — specifically called out the connection pooling benefit with Lambda (no RDS Proxy needed)
- **Provisioned concurrency + scheduled scaling** showed deep Lambda knowledge relevant to GPU drops
- **Single-account, multi-environment strategy** was practical and specific — IAM role boundaries enforced by system, not developer discipline
- **Per-branch ephemeral environments** answer was creative and matched NVIDIA's serverless stack — Mohnish validated it as very doable
- **Wildcard DNS for front-end feature branches** was a thoughtful extension to the S3 challenge Mohnish raised
- **AI answer** was honest about current level while showing genuine curiosity — resonated well

## What Could Have Been Better

- **Lambda alias answer** took a prompt from Mohnish (*"can we manage three environments in a single lambda function?"*) — should have led with aliases before recommending separate functions
- **CDK answer** was introduced unprompted but not fully fleshed out before moving on — could have given a more concrete code structure example
- **RAG POC answer** was somewhat brief — could have gone deeper on the vector store choice or the chunking/retrieval strategy

---

## Hiring Signal Assessment

**Verdict: POSITIVE**

Mohnish ran the full 64-minute interview (no time pressure), engaged deeply on every topic, and ended with genuine enthusiasm about the future of DevOps and AI at NVIDIA. He didn't give explicit feedback but his body language (continued probing, validating answers with "yeah that sounds doable", sharing insider context about AI tool adoption) was consistently positive.

His closing comment about needing MLOps pipelines for SageMaker/Bedrock was essentially a preview of upcoming work — framed as "we need to be ready to cater to this" — which signals he was already thinking about Ramana fitting into the team.

---

## Notes on Mohnish's Domain & Team

- Engineering manager for **development team**, not DevOps — put on panel specifically for scripting/automation assessment
- Tech stack: Angular, React, Python (serverless), Java, SQL + NoSQL, AWS
- Fastest growing challenge: developer velocity outpacing environment availability (QA bottleneck)
- High bar: GPU drop failures get escalated directly to Jensen — zero tolerance for launch day infrastructure failures
- High performers: technical excellence + cost/time-to-market optimization + culture fit (not just finishing your own work)
- Mistakes are acceptable; owning them and not repeating them is required
- Team collaboration over individual standards: sometimes have to compromise best practices for team velocity — that's expected and valued
