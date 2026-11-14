# AWS Interview Prep — Java / Spring Boot Developer (9 Years Experience)

> **How to use this file:** Each answer has a **one-line "say this first"** hook, then depth.
> In an interview, lead with the hook, then expand only if they probe. Memory aids are marked 🧠.
>
> At 9 years, interviewers assume you know *what* a service is. They test **why you chose it,
> what broke in production, and what it cost.** Always answer in the shape:
> **Problem → Option chosen → Trade-off rejected → How I verified it worked.**

---

## Table of Contents

1. [Fundamentals & Architecture Basics](#1-fundamentals--architecture-basics)
2. [IAM & Security](#2-iam--security)
3. [Compute: EC2, ECS, EKS, Lambda](#3-compute-ec2-ecs-eks-lambda)
4. [Storage: S3, EBS, EFS](#4-storage-s3-ebs-efs)
5. [Databases: RDS, Aurora, DynamoDB, ElastiCache](#5-databases-rds-aurora-dynamodb-elasticache)
6. [Networking: VPC, Load Balancers, Route 53, CloudFront, API Gateway](#6-networking)
7. [Messaging & Event-Driven: SQS, SNS, EventBridge, Kinesis, MSK](#7-messaging--event-driven)
8. [Observability: CloudWatch, X-Ray, CloudTrail](#8-observability)
9. [Config & Secrets Management](#9-config--secrets-management)
10. [IaC & CI/CD](#10-iac--cicd)
11. [Spring Boot on AWS — The Section That Gets You Hired](#11-spring-boot-on-aws)
12. [Cost Optimization](#12-cost-optimization)
13. [Scenario-Based Questions (30)](#13-scenario-based-questions)
14. [Rapid-Fire Comparisons & Cheat Sheet](#14-rapid-fire-comparisons--cheat-sheet)
15. [Questions YOU Should Ask](#15-questions-you-should-ask)

---

## 1. Fundamentals & Architecture Basics

### Q1. Explain Region, Availability Zone, and Edge Location. Why does it matter to your app?

**Say this first:** "A Region is a geographic area, an AZ is an isolated datacenter cluster inside it, and an Edge Location is a CDN POP. My app lives in ≥2 AZs for HA, in 1 region for latency and data residency, and serves static assets from edge."

**Detail:**
- **Region** — e.g. `ap-south-1` (Mumbai). Independent. Most services are regional. Data does **not** leave a region unless you explicitly replicate it (important for GDPR / RBI data-localisation answers).
- **Availability Zone** — one or more discrete datacenters with independent power/cooling/network, connected to sibling AZs by low-latency links (single-digit ms). AZ names are *account-scoped* — my `ap-south-1a` may be your `ap-south-1c`. Use **AZ IDs** (`aps1-az1`) when comparing across accounts.
- **Edge Location / POP** — 600+ CloudFront POPs. Used by CloudFront, Route 53, Global Accelerator, WAF.
- **Local Zones / Outposts / Wavelength** — extensions for ultra-low latency and on-prem.

**Why it exists and how it plays out in a real app (plain language):**
- *The problem:* a single datacenter can lose power, network, or cooling. If your whole app sits in one, one incident takes you down. AWS therefore splits each Region into several AZs that fail **independently** but are close enough to talk fast.
- *How you use it:* you spread identical copies of your app across AZs (e.g. an Auto Scaling Group over 3 subnets, one per AZ) behind a load balancer. If AZ-a dies, the load balancer's health checks stop sending traffic there and AZ-b/AZ-c carry on. A database does the same with a hot standby in another AZ (Multi-AZ RDS).
- *Why Region choice matters:* (1) **latency** — pick the Region closest to users; (2) **law/compliance** — data stays in the Region you put it in; (3) **price and service availability** — prices differ per Region and new services land in some Regions first.
- *Pitfall:* cross-AZ traffic is **billed per GB** (and adds ~1–2 ms). Chatty services spread randomly across AZs can create a surprising "data transfer" line on the bill — a common follow-up question.
- *Edge in one line:* a user in London hitting your Mumbai app is slow; CloudFront caches static files (JS, images) at a nearby edge POP (Point of Presence) so most requests never travel to Mumbai.

🧠 **Region = country, AZ = building, Edge = corner shop.**

**Interview hook:** "Multi-AZ is for availability and it's cheap. Multi-Region is for disaster recovery and it's expensive — it doubles your infra and forces you to solve data replication and split-brain. I only go multi-region when there's an actual RTO/RPO requirement in writing."

---

### Q2. What is the AWS Shared Responsibility Model?

**Say this first:** "AWS secures the cloud, I secure what I put *in* the cloud."

| AWS owns ("of the cloud") | You own ("in the cloud") |
|---|---|
| Physical DCs, hardware, hypervisor | OS patching on EC2, container base images |
| Managed service internals (RDS engine host) | IAM policies, key rotation |
| Global network backbone | Security groups, NACLs, VPC design |
| Availability of the service itself | Data classification, encryption choices, app-level authz |

**The nuance that impresses:** responsibility *shifts with the abstraction level*.
- **EC2 (IaaS):** you patch the OS, the JVM, everything.
- **ECS Fargate:** AWS patches the host; you still own the container image and its CVEs.
- **Lambda (FaaS):** AWS owns runtime patching; you own code, deps, IAM role scope.

🧠 **"Higher up the stack, less on your plate."**

**Why this model exists:** AWS cannot see inside your account, so it cannot be responsible for how you configure things; you cannot touch AWS's datacenters, so you cannot be responsible for them. The line is drawn where each party has control.

**Concrete example — a data leak:** an S3 bucket holding customer PDFs is found publicly readable. Who is at fault? **You.** AWS provided a secure, durable service (their part), but the bucket policy / Block Public Access setting is *your* configuration. Conversely, if a hypervisor bug exposes your VM's memory to another tenant, that is AWS's.

**Per-layer checklist for a Spring Boot dev:**
- *Your code:* input validation, authN/authZ (Spring Security), no secrets in code, dependency CVE scanning (e.g. Log4Shell-type issues are **yours** to patch).
- *Your config:* IAM least privilege, security groups, encryption on (KMS), CloudTrail enabled.
- *Shared "controls":* some controls are inherited (physical security), some are shared (patching — AWS patches the RDS host, *you* choose the maintenance window and engine version upgrades).

---

### Q3. What are the pillars of the AWS Well-Architected Framework?

🧠 **"COPRS-S"** → **C**ost Optimization, **O**perational Excellence, **P**erformance Efficiency, **R**eliability, **S**ecurity, **S**ustainability.

For each, have one concrete thing you did:
- **Operational Excellence** — everything through CDK/Terraform, no console changes; runbooks in the repo.
- **Security** — IAM roles not keys, secrets in Secrets Manager, TLS everywhere, least privilege.
- **Reliability** — multi-AZ, health checks, circuit breakers, idempotent consumers, chaos-tested.
- **Performance Efficiency** — right-sized instances from CloudWatch data, caching layer, async where possible.
- **Cost Optimization** — Savings Plans, Graviton, S3 lifecycle, killed idle NAT/EBS.
- **Sustainability** — Graviton + Fargate right-sizing = fewer watts; same answer as cost, honestly.

**What each pillar actually means (so you can explain it, not just list it):**

| Pillar | The question it asks | Spring Boot example |
|---|---|---|
| Operational Excellence | Can we deploy, observe, and fix things safely and repeatedly? | Pipeline + dashboards + runbooks; every release is reversible |
| Security | Is data and access protected at every layer? | Task role instead of keys, TLS, KMS-encrypted RDS, WAF |
| Reliability | Does it recover from failure and handle load changes? | Multi-AZ, retries with backoff, circuit breaker, DLQ for failed messages |
| Performance Efficiency | Are we using the *right* resource type and size? | Redis cache for hot reads; Graviton; pick Aurora vs DynamoDB per access pattern |
| Cost Optimization | Are we paying only for value delivered? | Right-size from metrics, Savings Plans for steady load, Spot for batch |
| Sustainability | Are we minimising resources consumed per unit of work? | Higher utilisation, fewer idle instances, efficient (ARM) CPUs |

**Interview tip:** the pillars **conflict** (reliability costs money; extra security adds latency). The mature answer is *"I name the trade-off and let the business requirement decide."* The **Well-Architected Tool** in the console lets you run a questionnaire-style review per workload and produces a risk list.

---

### Q4. How do you design for High Availability vs Disaster Recovery? Explain RTO/RPO.

**RTO** = how long you can be *down*. **RPO** = how much *data* you can lose.
🧠 **"RTO = Time, RPO = Points (of data)."**

| Strategy | RTO | RPO | Cost | How |
|---|---|---|---|---|
| Backup & Restore | Hours | Hours | $ | Snapshots + cross-region copy, IaC to rebuild |
| Pilot Light | ~10s of min | Minutes | $$ | DB replica running, app tier scaled to 0 |
| Warm Standby | Minutes | Seconds | $$$ | Full stack running small, scale up on failover |
| Multi-Site Active/Active | ~0 | ~0 | $$$$ | Route 53 / Global Accelerator, Aurora Global or DynamoDB Global Tables |

**Plain-language definitions:**
- **High Availability (HA)** = the system *keeps running* through a small, common failure (one server or one AZ dies) with little or no human action. Achieved by redundancy + automatic failover.
- **Disaster Recovery (DR)** = the plan to *get back* after a large, rare failure (a whole Region is down, data is corrupted/deleted, ransomware). It usually involves a person or a runbook and a recovery procedure.
- Example: a $50M-revenue app says "we can be down max 1 hour (RTO = 1 h) and lose max 5 minutes of orders (RPO = 5 min)". RPO 5 min means backups/replication must happen at least every 5 minutes; RTO 1 h means the standby environment must be ready to take traffic within the hour.

**How each DR strategy works (cheapest → most expensive):**
1. **Backup & Restore** — nightly snapshots copied to a second Region. On disaster, recreate everything from IaC (CloudFormation/Terraform) and restore data. Slow but nearly free while idle.
2. **Pilot Light** — only the "core" (the database replica) runs in the DR Region; app servers are defined but off. On disaster you launch them and point DNS over.
3. **Warm Standby** — a *scaled-down but complete* copy is always running and serving health checks; on disaster you scale it up and switch Route 53.
4. **Active/Active** — both Regions serve live traffic; losing one just shifts load. Hardest part is **data**: concurrent writes in two Regions need conflict resolution (DynamoDB Global Tables = last-writer-wins; Aurora Global Database = one writer Region, others read-only).

**Pitfalls / follow-ups:** a DR plan that is never *tested* is a hope, not a plan — run game-days. Also remember DR needs more than infra: DNS TTLs, secrets and KMS keys replicated to the DR Region, and AMIs/images copied there.

**Say this:** "HA is about surviving an AZ loss automatically — Multi-AZ RDS, ASG across 3 AZs, ALB health checks. DR is about surviving a *region* loss, which is a business decision because it costs real money. I ask for the RTO/RPO number first; without it, the conversation is just opinions."

---

## 2. IAM & Security

### Q5. IAM User vs Role vs Group vs Policy. Which does a Spring Boot app use?

- **User** — long-lived identity + access keys. For humans/legacy only. Ideally zero of them (use IAM Identity Center / SSO).
- **Group** — collection of users for policy attachment. No credentials of its own.
- **Role** — an identity with **no permanent credentials**, assumed to get **temporary STS credentials**. This is what services use.
- **Policy** — the JSON document of permissions. Identity-based (attached to user/role) or resource-based (attached to S3 bucket, SQS queue, KMS key).

**Say this first:** "My Spring Boot app never has access keys. On EC2 it uses an instance profile, on ECS a task role, on EKS IRSA or Pod Identity, on Lambda the execution role. The AWS SDK's DefaultCredentialsProvider picks these up automatically, so the same JAR runs locally with a profile and in prod with a role — no code change."

**Credential resolution order (SDK v2)** 🧠 "**SEW-PC-I**":
1. **S**ystem properties (`aws.accessKeyId`)
2. **E**nvironment variables (`AWS_ACCESS_KEY_ID`)
3. **W**eb identity token (IRSA / OIDC)
4. **P**rofile file (`~/.aws/credentials`)
5. **C**ontainer credentials (ECS task role via `169.254.170.2`)
6. **I**nstance profile (EC2 IMDSv2 via `169.254.169.254`)

**How a Role actually works (step by step):**
1. Every role has two policies: a **trust policy** (*who may assume me* — e.g. `ecs-tasks.amazonaws.com`) and **permission policies** (*what I can do* — e.g. `sqs:ReceiveMessage` on one queue).
2. When your ECS task starts, ECS calls **STS** (Security Token Service) `AssumeRole` on your behalf and exposes the resulting **temporary** access key + secret + session token to the container at a local endpoint.
3. The SDK reads them, signs each API call, and **refreshes automatically** before they expire (typically within a few hours). Nothing to store, rotate, or leak.

```json
// trust policy of the task role – only ECS tasks can assume it
{ "Effect": "Allow",
  "Principal": { "Service": "ecs-tasks.amazonaws.com" },
  "Action": "sts:AssumeRole" }
```

**Task role vs execution role (classic ECS trap):** the *execution role* is used by **ECS itself** to pull the image from ECR and write logs; the *task role* is what **your code** uses to call S3/SQS/DynamoDB. Putting S3 permissions on the execution role is a frequent bug → `AccessDenied` from the app.

**Pitfalls:** never `*:*` on a role; never bake keys into `application.yml`/Docker images; use groups/Identity Center for humans rather than IAM users.

---

### Q6. How is an IAM policy evaluated? Explain the decision logic.

🧠 **"Deny wins. Always."**

Order:
1. Default = **implicit deny**.
2. Any **explicit `Deny`** anywhere (SCP, identity policy, resource policy, permission boundary, session policy) → **DENY**, no appeal.
3. **SCP** (Organizations) must allow — it's a *filter*, it never grants.
4. **Permission boundary** must allow.
5. Then an **`Allow`** from identity or resource policy → ALLOW.

**Worked example (why same-account vs cross-account differs):**
- *Same account:* an `Allow` in **either** the identity policy (on the role) **or** the resource policy (on the bucket) is enough, provided there is no Deny.
- *Cross account:* **both** sides must allow — the other account's role needs an identity-policy allow *and* your bucket policy must name them. Forgetting one side is the #1 cause of cross-account `AccessDenied`.
- *Deny example:* the role allows `s3:*`, but the bucket policy has `Deny` when `aws:SecureTransport = false`. An HTTP (non-TLS) request is denied even though the role allows everything — "Deny wins".
- *Permission boundary vs SCP:* both only **limit** — they never grant by themselves. SCP limits a whole account/OU; a boundary limits a single user/role (used so developers can create roles but not more powerful than themselves).

**Debugging tip:** the error message in CloudTrail / the **IAM Policy Simulator** tells you which policy type caused the deny ("explicit deny in a service control policy" vs "no identity-based policy allows").

**Key policy elements:** `Version`, `Statement[]` with `Sid`, `Effect`, `Action`, `Resource`, `Condition`, and for resource policies `Principal`.

**Condition keys worth naming:** `aws:SourceIp`, `aws:PrincipalOrgID`, `aws:SecureTransport`, `s3:prefix`, `aws:RequestTag/*`, `aws:MultiFactorAuthPresent`.

**Least-privilege in practice:** "I start from a broad policy in dev, then run **IAM Access Analyzer → policy generation from CloudTrail** to produce the minimal policy, and use **Access Advisor** to strip unused services quarterly."

---

### Q7. How do you handle secrets (DB password, API keys) in Spring Boot on AWS?

**Say this first:** "Secrets Manager for anything with a rotation requirement, SSM Parameter Store SecureString for everything else because it's free. Never in `application.yml`, never in the Docker image, never in an env var checked into a task definition."

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Cost | ~$0.40/secret/month + API calls | Standard tier **free** |
| Rotation | **Built-in, Lambda-based, native for RDS** | DIY |
| Cross-account | Resource policy — yes | Advanced tier only |
| Size | 64 KB | 4 KB (8 KB advanced) |
| Use for | DB creds, third-party API keys | Feature flags, URLs, non-rotating config |

**Spring Boot wiring:**
```yaml
# Spring Cloud AWS 3.x
spring:
  config:
    import:
      - aws-secretsmanager:/prod/orders/db
      - aws-parameterstore:/prod/orders/
  datasource:
    url: ${db-url}
    username: ${username}
    password: ${password}
```
- Fetched at **startup** into the Environment → zero request-path latency.
- **The trap:** rotated secret ≠ reloaded app. Either use **RDS IAM auth** (token-based, no password at all), or an **Aurora/RDS Proxy** which holds the credential, or catch auth failure and refresh, or accept a rolling restart on rotation. Say this out loud — it's the follow-up they're waiting for.

**Best answer for RDS:** "I prefer **IAM database authentication** — the app generates a 15-minute token via `RdsUtilities.generateAuthenticationToken()`; there's no static password to leak or rotate."

**Why this question exists (the problem):** a password in `application.yml` ends up in Git history, Docker image layers, CI logs, and every developer laptop. Managed secret stores keep the value in one place, encrypted with KMS, with access controlled by IAM and every read audited in CloudTrail.

**How it works step by step:**
1. The app's task/instance role is granted `secretsmanager:GetSecretValue` (and `kms:Decrypt` if a customer-managed key is used) on **only** that secret's ARN.
2. At startup, `spring.config.import` makes Spring Cloud AWS call the service once and load the keys (a JSON secret `{"username":"..","password":".."}` becomes properties named `username`, `password`).
3. Spring resolves `${password}` into the DataSource. After that, nothing talks to Secrets Manager on the request path.

**Rotation in plain words:** Secrets Manager can run a rotation Lambda every N days that creates a new DB password, sets it in RDS, and stores the new value. The pitfall is that your running Hikari pool still holds connections authenticated with the *old* password and Spring read the secret only once. Mitigations: (a) RDS IAM auth; (b) the *alternating users* rotation strategy, where two DB users take turns so the old password stays valid for one cycle — giving you time to restart; (c) restart tasks after rotation (EventBridge → trigger a new ECS deployment).

**IAM-auth caveats (follow-up probes):** needs `rds-db:connect` permission, SSL enforced, and the token lives 15 min **only for establishing a connection** — existing connections stay open, but new pool connections need a fresh token, so you need a DataSource that regenerates it (e.g. the AWS Advanced JDBC Wrapper's IAM plugin). It also has a connection-rate limit, so many short-lived connections are a poor fit.

**Local dev:** don't point local runs at prod secrets; use `application-local.yml` with a Docker Postgres/LocalStack, and keep `optional:aws-secretsmanager:` imports so the app boots without AWS.

---

### Q8. Encryption at rest and in transit — how do you actually implement it?

**In transit:** TLS terminated at ALB with an **ACM** cert (free, auto-renewing). Re-encrypt to targets if compliance demands (ALB → HTTPS target group). Enforce `aws:SecureTransport: false → Deny` on S3 buckets. `sslmode=require` on JDBC + RDS CA bundle in the truststore.

**At rest:** KMS everywhere — EBS default encryption on, RDS storage encryption at create time (**can't be enabled later — you must snapshot → copy encrypted → restore**), S3 SSE-KMS or SSE-S3, SQS/SNS/DynamoDB/EFS/Secrets all KMS-backed.

**KMS key types:**
- **AWS managed** (`aws/s3`) — free, no policy control, AWS rotates.
- **Customer managed (CMK)** — $1/month, your key policy, your rotation, **can be revoked/disabled = instant crypto-shredding**, cross-account grants. Use for regulated data.
- **Envelope encryption** 🧠: KMS generates a **data key**, the data key encrypts your data, KMS encrypts the data key. Only the small key hits KMS — that's how you encrypt terabytes with a 4 KB API limit.

**Gotcha to mention:** "S3 with SSE-KMS at high TPS hits KMS request limits — I enable **S3 Bucket Keys** which cuts KMS calls by up to 99%."

**Jargon decoded:**
- **At rest** = data sitting on a disk/volume/backup. **In transit** = data moving over a network. Different threats, different tools.
- **TLS** (Transport Layer Security, the successor of SSL) encrypts the connection so a network sniffer sees gibberish. **ACM** (AWS Certificate Manager) issues and auto-renews the certificate you attach to the ALB listener.
- **KMS** (Key Management Service) stores keys in HSMs (hardware security modules); the key material never leaves KMS in plain form.
- **SSE-S3 vs SSE-KMS:** SSE-S3 = S3 manages keys for you (no extra control). SSE-KMS = encryption uses a KMS key, so you get key policies, audit trail in CloudTrail, and the ability to disable the key.
- **Crypto-shredding** = instead of deleting terabytes, you delete/disable the key; data becomes unreadable.

**Envelope encryption, step by step:**
1. App asks KMS `GenerateDataKey` → gets a **plaintext data key** + the same key **encrypted** under the KMS key.
2. App encrypts the file locally with the plaintext key (AES-256), then throws the plaintext key away.
3. App stores the encrypted data key next to the encrypted file.
4. To read: send the encrypted data key to KMS `Decrypt`, get the plaintext key back, decrypt the file locally.
AWS services (S3, EBS, RDS) do this for you automatically; you only need to write it yourself for app-level field encryption (use the AWS Encryption SDK).

**Spring Boot touch-points:** JDBC URL `?sslmode=verify-full` (Postgres) or `useSSL=true&requireSSL=true` (MySQL) with the RDS CA bundle; HTTPS-only S3 client calls are default; for PII columns consider app-level encryption so DBAs can't read them either.

**Pitfall:** enabling encryption later on an existing unencrypted RDS instance isn't a toggle — plan it at creation time or accept a snapshot-copy-restore with downtime.

---

## 3. Compute: EC2, ECS, EKS, Lambda

### Q9. How do you choose between EC2, ECS/Fargate, EKS, and Lambda for a Spring Boot service?

**Say this first:** "Default to **ECS on Fargate** for a Spring Boot REST service. It's containers without a control plane to babysit. I move to EKS only if the org is already Kubernetes-native or I need K8s ecosystem tooling; to EC2 only for licensing, GPUs, or special networking; to Lambda for spiky/event-driven work where the JVM cold start is acceptable or I use SnapStart/GraalVM."

| | Best for | Watch out for |
|---|---|---|
| **EC2 + ASG** | Full OS control, licensed software, long-lived stateful | You patch it, you scale it, you own the AMI pipeline |
| **ECS Fargate** | 90% of Spring Boot microservices | No daemonsets, per-task price premium, 20 GB ephemeral default |
| **ECS on EC2** | Cost at scale, GPU, host-level agents | Cluster capacity management, bin-packing |
| **EKS** | Multi-team platform, K8s tooling, portability | $73/mo control plane + real operational complexity |
| **Lambda** | Event-driven, cron, glue, unpredictable spikes | Cold start, 15-min cap, 10 GB mem, no long connections |

**Nine-years answer:** "I've regretted choosing EKS more than I've regretted choosing ECS. Pick the least infrastructure that satisfies the requirement."

**Decision walk-through (the reasoning behind the table):**
- *What's being chosen:* "who manages the machine under my JVM and how much of the orchestration (restarting, scaling, placing containers) do I do myself?" From EC2 → Fargate → Lambda you manage less and pay a bit more per unit of compute.
- **Fargate first:** you package the app as a Docker image, define CPU/memory, AWS runs it — no servers, no patching, no capacity planning. Most REST/event-consumer services fit.
- **EC2** when you need something Fargate can't give (GPUs, huge instance sizes, custom kernel/agents, licence tied to a host) or when a steady, large fleet makes the ~15–20% Fargate premium matter.
- **EKS** = managed Kubernetes. Worth it if you already have K8s skills/tools (Helm, operators, service mesh) or must run on several clouds. Cost = control-plane fee + upgrades + networking/RBAC complexity.
- **Lambda** = pay per request/duration, scales to zero. A good fit for S3/SQS-triggered jobs and low-traffic APIs. A plain Spring Boot app takes seconds to start (cold start), so for Lambda you'd use SnapStart (Java snapshot restore), GraalVM native image, or a lighter framework.
- **Example:** an order-service with steady 200 req/s → Fargate behind ALB. A nightly image-resize triggered by S3 uploads → Lambda. A team of 40 engineers with 150 services and Istio already in use → EKS.

**Follow-up probes:** "How does the answer change at 10x scale/cost?" (look at Savings Plans + ECS-on-EC2), "How do you avoid lock-in?" (containers keep you portable; avoid proprietary glue where it doesn't pay).

---

### Q10. Explain Auto Scaling Groups. What's the difference between the scaling policy types?

**What it is, in plain language:** an **Auto Scaling Group (ASG)** is a fleet manager for EC2 instances. You say "always keep between 2 and 10 copies of this server, spread across 3 AZs" and it launches, health-checks, replaces, and removes instances for you. **Why:** without it, a dead server stays dead and traffic spikes need a human to click "launch". **How:** a **Launch Template** is the recipe (which AMI, size, security group, startup script); the ASG creates instances from it, registers them in the load balancer's target group, and a policy decides when to change the *desired* count.

**Components:** Launch Template (AMI, instance type, SG, user-data, IMDSv2), min/desired/max, target groups, health check type (`EC2` vs `ELB` — **use ELB** so a hung JVM gets replaced), termination policy, warm pools, lifecycle hooks.

**Policy types:**
- **Target Tracking** — "keep average CPU at 60%" or `ALBRequestCountPerTarget` at 1000. Simplest, use this by default. 🧠 *like a thermostat.*
- **Step Scaling** — CloudWatch alarm tiers: +1 instance at 70%, +3 at 90%. For known burst shapes.
- **Simple Scaling** — legacy, has a cooldown lock. Avoid.
- **Scheduled** — market opens at 9:15 AM IST, scale to 20 at 9:00.
- **Predictive** — ML on 14 days of history, pre-warms capacity. Great for daily-cycle traffic.

**Choosing between them with an example:** target tracking is like a car's cruise control — you state the goal (60% CPU) and AWS adds/removes instances to hold it; it also creates the CloudWatch alarms for you. Step scaling is manual gear-shifting: you define the alarm thresholds and how many instances each step adds, useful when you know that 90% CPU means *a lot* more capacity is needed *now*. Scheduled is for events you can predict to the minute. Predictive is for repeating daily/weekly patterns. In practice, combine them: scheduled/predictive for the known curve + target tracking as the safety net.

**Common terms:** *cooldown* = wait time after a scaling action so metrics can settle (prevents flapping); *scale-in protection* = flag that stops a busy instance from being terminated; *instance refresh* = rolling replacement of all instances when you change the Launch Template (how you roll out a new AMI).

**Best metric for Spring Boot:** "CPU is a lagging indicator for an I/O-bound Java service. I scale on `ALBRequestCountPerTarget` or a custom CloudWatch metric like SQS `ApproximateNumberOfMessagesVisible` per task, or Tomcat busy-thread ratio published from Micrometer."

**Gotchas to volunteer:**
- **Connection draining / deregistration delay** — set to 30s and make sure it's ≥ your longest request, or you 502 during deploys.
- **JVM warm-up** — a fresh instance is slow (JIT, connection pool, class loading). Use ASG **warm pools**, ALB **slow start** (30–60s), and Spring's `--enable-preview` / AOT / CDS to shorten it.
- **Lifecycle hooks** for graceful shutdown before termination.

---

### Q11. ECS: explain Task, Task Definition, Service, and Cluster. How do you deploy safely?

- **Task Definition** — the blueprint (JSON): image, CPU/memory, port mappings, env, secrets, log config, **task role** (app permissions) vs **execution role** (pull image, write logs), health check. Immutable + versioned (`orders:47`).
- **Task** — one running instance of that definition.
- **Service** — keeps N tasks alive, registers to ALB target group, handles rolling deploys.
- **Cluster** — logical grouping + capacity providers (Fargate, Fargate Spot, EC2 ASG).

**Analogy + flow:** Task Definition = the *recipe*, Task = one *dish cooked from it*, Service = the *manager who ensures 4 dishes are always on the counter* (replacing any that drop), Cluster = the *kitchen*. Typical flow: CI builds image → pushes to ECR (Elastic Container Registry) → registers new Task Definition revision `orders:48` → updates Service to use it → ECS starts new tasks, waits for ALB health checks to pass, then stops old ones.

**Why a new revision each time:** task definitions are immutable, so a rollback is simply "point the service back to `orders:47`" — no mutable state to untangle.

**Deployment strategies:**
- **Rolling** with `minimumHealthyPercent=100`, `maximumPercent=200` → new tasks up before old go down, zero downtime.
- **Blue/Green via CodeDeploy** — shifts ALB listener traffic; supports canary (10% for 5 min) and **automatic rollback on CloudWatch alarm**. This is the answer they want for "safe deploy".
- **ECS deployment circuit breaker** — auto-rollback if tasks keep failing health checks.

**How these differ in practice:**
- *Rolling:* with min 100% / max 200% and 4 tasks, ECS starts up to 4 new tasks (8 total), waits for them to become healthy, then drains and stops the 4 old ones. Cheap and simple; but during the rollout both versions serve traffic, so API/DB changes must be backward compatible. Rollback = redeploy old revision (minutes).
- *Blue/Green:* a full new ("green") task set is created beside the live ("blue") one; CodeDeploy moves the ALB listener from blue to green — all at once, linear (10% every minute), or canary (10% then the rest). You can run test traffic against green first, and rollback is a listener flip (seconds), but you temporarily pay for double capacity.
- *Circuit breaker:* if new tasks repeatedly crash or fail health checks, ECS stops the deployment and reverts automatically — enable it by default (`deploymentCircuitBreaker: {enable: true, rollback: true}`).
- **Pitfall:** health-check grace period too short → ECS kills a slow-starting JVM before it's ready and enters a restart loop that looks like a bad deploy.

**Spring Boot specifics for ECS:**
```
Health check → /actuator/health/readiness  (Kubernetes-style probes work on ALB too)
Grace period  → 60–120s for JVM startup (startPeriod on container health check)
Graceful stop → server.shutdown=graceful + spring.lifecycle.timeout-per-shutdown-phase=25s
                stopTimeout=30 (must be > graceful timeout, must be < deregistration delay + buffer)
Logs          → awslogs driver or awsfirelens → CloudWatch/OpenSearch, JSON structured
```

🧠 **Ordering rule: `graceful timeout` < `stopTimeout` and `deregistration delay` ≥ longest request.**

---

### Q12. What is Fargate? Pros/cons vs EC2 launch type?

Serverless container compute — you declare vCPU/memory, AWS runs it on a managed, task-isolated micro-VM (Firecracker). Each task gets its own ENI (`awsvpc` mode) with its own security group.

**Pros:** no host patching, no cluster capacity math, per-second billing, strong isolation, scales in seconds, **Fargate Spot up to 70% off** for interruptible work.
**Cons:** ~15–20% price premium per vCPU vs EC2 at steady load, no privileged containers or daemonsets, ephemeral storage 20 GB default (max 200 GB), no GPU, cold-ish task start (~30–60s + JVM boot).

**Sizing note for Java:** valid combos are constrained (0.25 vCPU→0.5–2 GB, 1 vCPU→2–8 GB, etc.). **Always set `-XX:MaxRAMPercentage=75`** rather than a fixed `-Xmx`, so the same image works at any task size. Modern JVMs are container-aware, but on Fargate the *task* limit is what matters — verify with `Runtime.maxMemory()`.

**Plain-language explanation:** with the **EC2 launch type** you own a fleet of virtual machines that ECS packs containers onto — you patch them, scale them, and pay for idle headroom. With **Fargate** you skip the machines entirely: you say "this task needs 1 vCPU / 2 GB" and AWS finds the capacity, runs it in isolation, and bills only for the task's vCPU-seconds and GB-seconds.
- *`awsvpc` mode in practice:* each task gets its own network interface (ENI) with a private IP in your subnet, so you attach a security group **per service** (e.g. only the ALB SG may call the orders tasks) — finer-grained than per-host.
- *Example sizing:* a Spring Boot REST API with a 512 MB heap fits 1 vCPU / 2 GB. With `MaxRAMPercentage=75`, a 2 GB task gives ~1.5 GB heap, leaving ~0.5 GB for metaspace, thread stacks, and native memory. Setting heap = 100% of the task causes OOM-kills (exit code 137) that show *no* Java error.
- *When EC2 launch type wins:* steady 24/7 load where Savings Plans/Reserved Instances beat the Fargate premium, GPUs, very large tasks, or host-level agents (e.g. a security daemon).

---

### Q13. Lambda: cold starts, concurrency, and is it viable for Spring Boot?

**Lifecycle:** INIT (download → start runtime → run static/constructor code) → INVOKE → SHUTDOWN. **Cold start = the INIT phase.**

**What this means (plain language):** Lambda runs your code in a short-lived sandbox ("execution environment"). The first request to a new sandbox must download your JAR, boot the JVM, and initialise Spring (scan classes, build beans, create connection pools) before it can handle the request — that delay is the **cold start**. Later requests reuse the same *warm* sandbox and skip INIT. Spring Boot is heavy at startup, which is why Java is the worst case for cold starts.

**Java cold start reality:** plain Spring Boot on Lambda = **5–10s** cold start. Fixes, in order of preference:
1. **AWS Lambda SnapStart** — snapshots the initialized JVM (Firecracker CRaC checkpoint), restores in ~200–500ms. Free. **This is the modern answer.** Caveats: beware cached state, unique IDs, and stale DB connections across restore — implement `CRaC Resource` hooks.
2. **Spring Cloud Function + `spring-cloud-function-adapter-aws`** — skips the servlet container.
3. **GraalVM native image** via `spring-boot-maven-plugin:native` → ~50ms start, but reflection config pain and long builds.
4. **Provisioned Concurrency** — pre-initialized envs, no cold start, but you pay 24/7 (defeats the point for steady load).

**Concurrency:**
- **Account default 1000** concurrent executions per region (raisable).
- **Reserved concurrency** — caps *and* guarantees a function's share. Use it to protect an RDS connection pool from being exhausted by 500 concurrent Lambdas.
- **Provisioned concurrency** — warm instances.
- Scaling: **1000 concurrent executions per 10 seconds per function** burst ramp.

**Concurrency explained with numbers:** *concurrency* = how many requests are being processed **at the same instant** = (requests/sec) × (average duration in sec). 200 req/s × 0.5 s = 100 concurrent executions = 100 sandboxes, each with its **own** JVM and its **own** DB connection. That's the Spring Boot trap: 100 Lambdas × a pool of 10 = 1000 DB connections and an exhausted RDS. Fixes: tiny pool size (1–2) per Lambda, **RDS Proxy** to multiplex connections, and reserved concurrency as a ceiling.
- *Throttling:* beyond the limit callers get HTTP 429; async invokes are retried automatically, SQS-triggered ones go back to the queue.
- *SnapStart caveat example:* a `Random` seed or a UUID generated during init is *frozen into the snapshot* and shared by every restored copy — generate such values per-invoke.

**When I say no to Lambda for Spring Boot:** synchronous low-latency APIs with steady traffic, anything needing >15 min, anything with a big JDBC connection pool, and anything where the team's whole toolchain assumes a long-running process. "I use Lambda for the edges — S3 triggers, EventBridge crons, SQS workers, DynamoDB streams — and ECS for the core APIs."

---

### Q14. Elastic Beanstalk — would you use it in 2026?

**Say this:** "It's the fastest path from a Spring Boot JAR to a running, load-balanced, auto-scaling environment — it provisions EC2, ALB, ASG, CloudWatch for you and gives you rolling/immutable/blue-green deploys via `swap CNAME`. I'd use it for a legacy lift-and-shift or a quick internal tool. For anything new I'd choose ECS Fargate, because Beanstalk's abstraction leaks the moment you need something custom, and `.ebextensions` is a worse config language than a Dockerfile."

Know the deploy policies: **All at once / Rolling / Rolling with additional batch / Immutable / Blue-Green (CNAME swap)**. Immutable is safest; All-at-once causes downtime.

**What Beanstalk is, plainly:** a PaaS (Platform-as-a-Service) wrapper. You upload a Spring Boot JAR (or Docker image); Beanstalk creates the underlying EC2 instances, ASG, ALB, security groups and CloudWatch alarms and wires them together. *Why it exists:* so developers can deploy without learning each building block. *The catch:* the resources are still yours and visible, so manual changes in the console get overwritten on the next update ("configuration drift").

**What the deploy policies mean:**
- *All at once* — update every instance simultaneously → fastest but brief downtime.
- *Rolling* — update in batches (e.g. 25% at a time) → capacity temporarily reduced.
- *Rolling with additional batch* — launches extra instances first so full capacity is kept.
- *Immutable* — builds a **brand-new ASG** with the new version; when healthy it's swapped in and the old one is terminated. Rollback = delete the new ASG. Cost: briefly double instances.
- *Blue/Green* — a second full environment; swap URLs (CNAME) when ready. Needed for DB-affecting or high-risk changes.

**Example config for a Spring Boot JAR:** `Procfile` containing `web: java -jar application.jar`, and set `SERVER_PORT=5000` (Beanstalk's nginx proxy forwards to 5000 by default). Mentioning this detail signals real usage.

**Follow-up probe:** "Why not Beanstalk for new services?" → container-first workflow (ECS/Fargate) gives identical dev/prod artefacts, finer scaling, and a bigger ecosystem; Beanstalk's platform branches also get retired and need migration.

---

## 4. Storage: S3, EBS, EFS

### Q15. S3 fundamentals: consistency, durability, storage classes.

**Say this first:** "S3 is object storage — flat key namespace, 11 nines durability, **strongly read-after-write consistent since Dec 2020** for PUTs, DELETEs and LISTs. That last part matters: the old 'eventual consistency, so add a delay' workaround is dead."

**Durability/availability:** 99.999999999% (11 nines) durability; availability varies by class (99.99% Standard, 99.5% One Zone-IA).

**Storage classes** 🧠 "**Standard → IA → Intelligent → Glacier → Deep**":

| Class | Use | Retrieval | Min duration |
|---|---|---|---|
| Standard | Hot, unpredictable | ms | — |
| **Intelligent-Tiering** | Unknown access pattern — *default choice when unsure* | ms | — (small monitoring fee) |
| Standard-IA | Backups accessed monthly | ms | 30 days |
| One Zone-IA | Reproducible data (thumbnails) | ms | 30 days |
| Glacier Instant Retrieval | Archives needing ms access | ms | 90 days |
| Glacier Flexible | Archive, minutes–hours | 1–5 min to 12 h | 90 days |
| Glacier Deep Archive | Compliance, 7–10 yr | 12–48 h | 180 days |

**Lifecycle policy example answer:** "Logs: Standard 30d → Standard-IA 60d → Glacier Flexible 1y → Deep Archive → expire at 7 years. That one policy cut our S3 bill ~70%."

**Terms explained:**
- **Object storage:** you store whole objects (file + metadata) addressed by `bucket/key`, not blocks on a disk. There are no real folders — `invoices/2026/a.pdf` is just a long key with `/` in it. You can't edit part of an object; you replace it.
- **Durability vs availability:** durability = "will my data still exist?" (11 nines means you'd expect to lose ~1 object in 10 million every 10,000 years); availability = "can I read it right now?" (a class can be durable but slightly less available).
- **Strong consistency:** after a successful PUT, any subsequent GET or LIST sees the new data immediately. Before 2020 a GET right after an overwrite could return the old version.
- **Why classes exist:** price trades off against retrieval speed/cost. Cold data costs ~1/5 to 1/20 of Standard to store but charges for retrieval and has minimum storage durations (deleting early still bills the minimum).
- **Intelligent-Tiering:** S3 watches access and moves objects between tiers automatically — use it when you can't predict access patterns.

**Pitfalls:** lots of tiny objects in IA/Glacier incur per-object overhead and minimum-size charges; lifecycle transitions themselves cost per request, so don't transition millions of 1 KB files. Enable **versioning** carefully — old versions bill too, so add a lifecycle rule to expire non-current versions.

---

### Q16. How do you upload/download large files from Spring Boot to S3 without killing the JVM heap?

**Say this first:** "Never buffer a file in memory. Use the **S3 Transfer Manager** (SDK v2 + CRT client) for multipart parallel transfer, or hand the client a **pre-signed URL** so the bytes never touch my service at all."

```java
// Pre-signed URL — the correct answer for user uploads/downloads
S3Presigner presigner = S3Presigner.create();
PutObjectPresignRequest req = PutObjectPresignRequest.builder()
    .signatureDuration(Duration.ofMinutes(10))
    .putObjectRequest(b -> b.bucket("uploads").key(key).contentType("application/pdf"))
    .build();
URL url = presigner.presignPutObject(req).url();
```
**Why it's the right answer:** browser → S3 direct. No bandwidth cost through your ECS task, no heap pressure, no request timeout, no ALB 100 MB-ish practical limits. Your service only issues a short-lived signed URL and records metadata.

**The problem being solved:** a naive controller using `@RequestParam MultipartFile file` + `file.getBytes()` loads the whole file onto the heap; 20 users uploading 500 MB at once = 10 GB → `OutOfMemoryError`. It also ties up Tomcat threads for minutes.

**How a pre-signed URL works:** your service uses *its own* IAM permissions to compute a signature over the request (bucket, key, method, expiry). The URL embeds that signature. S3 accepts anyone presenting the URL until it expires, **acting with the signer's permissions**, but only for that exact operation. Flow: (1) client asks `POST /uploads` → (2) service returns URL + key → (3) client `PUT`s the file directly to S3 → (4) S3 event (to SQS/Lambda/EventBridge) tells your backend the upload finished → (5) backend processes it.

**Pitfalls:** set a short expiry; fix `Content-Type` (and optionally content-length range via pre-signed POST) so users can't upload arbitrary things; use a server-generated key (e.g. UUID) so users can't overwrite others' files; the URL is a bearer credential — don't log it; browser uploads from your site need **CORS** configured on the bucket.

**Server-side large files (when you must touch the bytes):** use `S3TransferManager` with the CRT-based async client, or stream with `RequestBody.fromInputStream(in, length)` — memory stays at a few part-sized buffers (e.g. 8 MB × parallel parts), not file size. For downloads, stream the `ResponseInputStream` straight to the `HttpServletResponse` (or redirect to a pre-signed GET URL).

**Multipart upload:** files >100 MB should be multipart (required >5 GB, max object 5 TB, max part 5 GB, 10,000 parts).
🧠 **"5 GB single PUT limit, 5 TB object limit."**
- Always add a lifecycle rule to **abort incomplete multipart uploads after 7 days** — otherwise you pay for orphaned parts forever. This is a classic "have you actually run S3 in prod?" tell.

**Multipart in plain language:** the file is cut into parts (say 16 MB each); each part is uploaded independently, **in parallel**, with its own retry; then a final "complete" call stitches them together. Benefits: faster (parallel), resilient (re-send only a failed part, not 5 GB), and resumable. If the upload is abandoned before "complete", the uploaded parts stay invisible but **billed** — hence the abort rule.

---

### Q17. How do you secure an S3 bucket?

Layered answer 🧠 **"BEAP-V"**:
1. **B**lock Public Access — account-level ON. Non-negotiable.
2. **E**ncryption — SSE-S3 default (or SSE-KMS for regulated data) + bucket policy denying `s3:PutObject` without encryption headers, and denying `aws:SecureTransport=false`.
3. **A**ccess via IAM roles + bucket policy with `aws:PrincipalOrgID` condition; **VPC endpoint policy** to restrict to your VPC.
4. **P**re-signed URLs / **CloudFront + OAC** for public content — never a public bucket.
5. **V**ersioning + MFA Delete + **Object Lock (WORM)** for ransomware/compliance; **Access Logs / CloudTrail data events** for audit; **Macie** for PII discovery.

**Trap question — "how do you make a bucket public for a website?"** → "I don't. I put CloudFront in front with **Origin Access Control** and keep the bucket private."

**Each layer explained (what/why):**
1. **Block Public Access** — an override switch that makes S3 ignore any ACL or policy that would grant public access, so one careless policy can't leak data. Most "S3 breach" headlines are this switch being off.
2. **Encryption + deny rules** — a bucket policy can *require* conditions. Example:
```json
{ "Effect": "Deny", "Principal": "*", "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": { "Bool": { "aws:SecureTransport": "false" } } }
```
   → any plain-HTTP request is rejected.
3. **Access control** — give the Spring Boot task role only `s3:GetObject`/`PutObject` on `arn:aws:s3:::bucket/prefix/*`. A **VPC gateway endpoint** (free) lets private subnets reach S3 without NAT, and its endpoint policy limits which buckets can be accessed — also saves NAT data-processing charges.
4. **Pre-signed URL / OAC** — grant *temporary, narrow* access to a single object without making anything public. OAC (Origin Access Control) lets only your CloudFront distribution read the private bucket.
5. **Recovery & audit** — versioning keeps old versions after overwrite/delete; Object Lock makes them immutable for a retention period (WORM = write once, read many); CloudTrail *data events* record who read which object; Macie scans for PII like Aadhaar/PAN/credit-card numbers.

**Pitfall:** ACLs are legacy — set bucket ownership to *Bucket owner enforced* to disable them and use policies only.

---

### Q18. EBS vs EFS vs S3 vs Instance Store — and what would a Spring Boot app use?

| | Type | Attach | Use |
|---|---|---|---|
| **EBS** | Block, single-AZ | 1 instance (io2 multi-attach exception) | DB data dir, JVM temp, boot volume |
| **EFS** | NFS file, multi-AZ | 1000s of instances/tasks | Shared uploads dir for legacy apps, Jenkins home |
| **S3** | Object over HTTPS | Anywhere | Files, backups, static assets, data lake |
| **Instance Store** | NVMe on host | 1, **ephemeral** | Caches, scratch, Kafka/Cassandra local |

**Plain-language difference:** *block* storage (EBS) is like a hard disk plugged into one server — fast, low latency, but only that server uses it and it lives in one AZ. *File* storage (EFS) is a shared network drive many servers mount at once (NFS = Network File System protocol) — convenient but slower and pricier per GB. *Object* storage (S3) is a web API for files — near-unlimited, cheapest, but accessed over HTTP, not as a mounted disk. *Instance store* is a disk physically inside the host: very fast, but its data **vanishes when the instance stops/terminates**.

**Choosing with examples:** MySQL running on your own EC2 → EBS (gp3/io2). Three ECS tasks all needing to read the same legacy `/data/reports` folder → EFS (or better, refactor to S3). User profile photos → S3. A Kafka broker's log segments, replicated anyway → instance store for speed.
**Pitfalls:** EBS can't be attached across AZs (restore from a snapshot to move it); EFS bills for stored GB and throughput mode, so a forgotten EFS keeps costing; never keep anything you can't lose on instance store.

**Volume types:** `gp3` (default — 3000 IOPS baseline, throughput decoupled from size, ~20% cheaper than gp2 → *always migrate gp2→gp3*), `io2 Block Express` (up to 256K IOPS, 99.999% durability, for databases), `st1/sc1` (throughput HDD for logs/big-data).

**The right answer for a modern Spring Boot service:** "Stateless — no persistent volume at all. Config from Parameter Store, files in S3, sessions in ElastiCache/Redis or JWTs, logs to stdout. If I need a local disk it's for temp files only, and I size Fargate ephemeral storage accordingly."

---

## 5. Databases: RDS, Aurora, DynamoDB, ElastiCache

### Q19. RDS Multi-AZ vs Read Replica. Interviewers love this one.

| | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | **Availability (HA)** | **Scalability (read offload)** |
| Replication | Synchronous | Asynchronous |
| Readable? | No (standby is idle)* | Yes |
| Failover | Automatic, DNS CNAME flip, 60–120s | Manual promote |
| Cross-region? | No (same region) | Yes |
| Data loss | Zero (sync) | Possible (lag) |

*Exception: **Multi-AZ DB Cluster** (2 readable standbys, semi-sync, ~35s failover) — mention it to show currency.

**Plain-language explanation:**
- **Multi-AZ** = RDS keeps a *hidden twin* of your database in another AZ. Every committed write is copied to the twin **before** the commit is acknowledged (synchronous), so the twin is always identical. If the primary host/AZ dies, RDS repoints the **same DNS endpoint** to the twin. Your app uses the same hostname — it just reconnects. You cannot send queries to the twin. *Problem solved:* hardware/AZ failure and maintenance with no data loss.
- **Read Replica** = a *separate, readable copy* with its own endpoint, fed asynchronously from the primary's change log. *Problem solved:* the primary is overloaded with reads (reports, search pages). *Cost:* **replica lag** — a row written a moment ago may not be visible yet.
- **Example of the lag bug:** user saves a profile (write → primary), page reloads and reads from a replica, shows old data. Fix: read-your-own-writes from primary for a short window, or only send tolerant queries (reports, dashboards) to replicas.
- **Using both:** Multi-AZ for production HA **and** replicas for read scaling — they're independent. A replica can also be promoted to a standalone DB (a DR tool across Regions).

**Spring Boot wiring for replicas:** define two DataSources and a routing `AbstractRoutingDataSource` that picks the replica when the transaction is `@Transactional(readOnly = true)`.

**Say this:** "Multi-AZ is insurance, not performance — you get zero read throughput from it. If someone says 'we have Multi-AZ so we can scale reads,' that's the bug."

**Spring Boot angle:** "Failover flips a DNS CNAME, so I set the JDBC/DNS TTL low and configure HikariCP to detect dead connections — `validationTimeout`, `keepaliveTime`, and a short `connectionTimeout` — otherwise the pool holds stale sockets for minutes after failover. For Aurora I use the **AWS JDBC Driver (`jdbc:aws-wrapper:postgresql://`)**, which does cluster-aware failover in ~3–10s instead of 60+."

---

### Q20. Aurora vs standard RDS. When Aurora Serverless v2?

**Aurora:** MySQL/PostgreSQL-compatible, storage is a distributed 6-copy/3-AZ log-structured layer that auto-grows to 128 TB. Up to 15 read replicas sharing the same storage → **replica lag in ms, not seconds**. Failover <30s. ~3–5x MySQL throughput. Reader/Writer endpoints + custom endpoints.

**Killer features to name:** Global Database (cross-region, <1s lag, RPO ~1s), Backtrack (rewind the cluster in seconds — brilliant after a bad migration), Cloning (copy-on-write clone of 2 TB in minutes for a test env), Fast Insert, zero-downtime patching, Blue/Green Deployments.

**Aurora Serverless v2:** scales in 0.5 ACU increments in-place, sub-second. Use for dev/test, spiky, or unknown-load workloads. Not cheaper at steady load — do the math against a Graviton provisioned instance.

**Say this:** "Aurora unless there's a reason not to — the reasons being cost at small scale, or needing an engine/version/extension Aurora doesn't support."

**What makes Aurora different (the "why"):** standard RDS runs a normal MySQL/Postgres engine on an EC2 host with an EBS disk attached — replication copies whole data changes between hosts. Aurora re-architected the storage: the database node sends only a small *redo log* to a shared storage cluster that keeps **6 copies across 3 AZs** and repairs itself. Consequences: (1) readers share the same storage so no data copying → replica lag is milliseconds; (2) failover only needs a reader to take over the writer role, no storage rebuild → faster; (3) storage grows automatically (no pre-provisioned GB) and you pay for what's used.

**Endpoints (important for Spring Boot):** the *cluster (writer) endpoint* always points to the current writer; the *reader endpoint* load-balances across replicas. Use the writer endpoint for your `@Transactional` write DataSource and the reader endpoint for read-only queries. Don't hardcode instance endpoints — they change on failover.

**Serverless v2 explained:** capacity is measured in **ACUs** (Aurora Capacity Units, ≈2 GiB RAM each + proportional CPU). You set min/max (e.g. 0.5–16 ACU) and it adjusts as load rises or falls, without dropping connections. *Example:* a dev DB idle at night costs ~0.5 ACU; a month-end report spike scales to 16 ACU automatically. At a flat 24/7 high load a reserved provisioned instance is usually cheaper.

**Pitfall:** Aurora isn't "free speed" — a poorly indexed query is still slow; and Aurora's I/O billing model (Standard vs I/O-Optimized) matters for write-heavy workloads — check which is cheaper for your pattern.

---

### Q21. When do you pick DynamoDB over RDS? Explain partition key design.

**Say this first:** "DynamoDB when I know my access patterns up front and I need single-digit-ms latency at any scale — sessions, carts, event stores, idempotency tables, feature flags. RDS when I need ad-hoc queries, joins, transactions across many entities, or analysts with SQL."

**Core concepts:**
- **Partition key (hash)** determines the physical partition. **Sort key (range)** enables range queries within a partition. Together = primary key.
- **Capacity:** On-Demand (spiky, unknown) vs Provisioned + auto-scaling (predictable, cheaper). **RCU** = 1 strongly-consistent read of 4 KB/s (2 eventually-consistent). **WCU** = 1 write of 1 KB/s. 🧠 **"Read 4, Write 1."**
- **GSI** — different PK/SK, own capacity, **eventually consistent**, can be added anytime. **LSI** — same PK, different SK, strongly consistent, **must be created with the table**, 10 GB per-partition limit.
- **Item limit 400 KB.** Big blobs go in S3 with the key in Dynamo.

**Hot partition / key design — the real question:** "A `status = PENDING` partition key would put all traffic on one partition and throttle. I use high-cardinality keys (`ORDER#<uuid>`), and if I must query by low-cardinality attributes I use **write sharding** — append `#<0-9>` suffix — or a GSI. Adaptive capacity helps but doesn't fix a genuinely bad key."

**How DynamoDB actually works (for people from an RDBMS background):**
- It is a key-value/document store. Data is spread across many physical partitions by **hashing the partition key**. A lookup by the key goes straight to one partition → predictable single-digit-ms latency whatever the table size. That's the "why".
- The price: you can only efficiently query by the key (and GSI keys). There are **no joins**, and a `Scan` reads the *whole table* (slow + expensive). So you design the table **from the queries backwards**: list every access pattern first, then choose keys.
- **Example — orders:** access patterns: "get order by id", "list a customer's orders by date". Table: `PK = CUSTOMER#123`, `SK = ORDER#2026-03-01#<orderId>`. `Query PK = CUSTOMER#123 AND begins_with(SK,'ORDER#2026-03')` returns a customer's March orders in one call. For "get order by id" add a GSI on `orderId`.
- **Capacity math:** writing a 3 KB item = 3 WCU; strongly-consistent read of a 6 KB item = 2 RCU (rounded up to 4 KB units), eventually consistent = 1 RCU.
- **Why a bad key throttles:** each partition has a throughput ceiling (~3000 RCU / 1000 WCU per second). If 90% of requests hit one key value, one partition saturates even when the table's total capacity is huge.

**When NOT DynamoDB:** exploratory/reporting SQL, many-to-many relations with changing queries, strict multi-row ACID across many tables — pick Aurora/RDS (or stream to S3/Athena for analytics).

**Spring Boot angle:** use the AWS SDK v2 **Enhanced Client** (`DynamoDbEnhancedClient`, `@DynamoDbBean`) or Spring Data DynamoDB-style libraries; use `ConditionExpression` for idempotency (`attribute_not_exists(PK)` makes a duplicate `PutItem` fail with `ConditionalCheckFailedException`, which you catch and treat as "already processed").

**Advanced credibility:** single-table design with `PK`/`SK` overloading, **DynamoDB Streams → Lambda** for CDC/outbox, **TTL** for auto-expiry (free deletes), **DAX** for microsecond reads, **transactions** (`TransactWriteItems`, up to 100 items), **conditional writes** for optimistic locking (`attribute_not_exists(PK)` = perfect idempotency guard), **Global Tables** for multi-region active-active.

---

### Q22. How do you use ElastiCache with Spring Boot? Redis vs Memcached.

**Redis** — data structures, persistence, replication + automatic failover (Multi-AZ), pub/sub, Lua, sorted sets, cluster mode for sharding, transactions. **Memcached** — plain multi-threaded key/value, simple horizontal scaling, no persistence, no replication.
🧠 **"Redis = features. Memcached = raw simple speed."** In interviews: pick Redis, 95% of the time.

**What ElastiCache is and why you'd add it:** ElastiCache is AWS-managed Redis/Memcached (it handles node provisioning, patching, failover, backups). A cache keeps **frequently-read, rarely-changed data in memory** so repeat reads (~sub-millisecond) skip the slower database call (~5–50 ms) and reduce DB load. *Example:* a product-catalog endpoint called 2,000 times/sec for data that changes hourly — cache it for 10 minutes and the DB sees a few queries instead of 2,000/sec.

**How `@Cacheable` flows (cache-aside):** (1) method called → Spring builds key `product::42`; (2) looks in Redis; (3) **hit** → return value, method body never runs; (4) **miss** → run the method (DB query), store the result with the TTL, return it. `@CacheEvict` / `@CachePut` on update methods keep it consistent. *Pitfall:* `@Cacheable` works through Spring's proxy, so calling a cached method from another method **in the same class** bypasses the cache.

**Spring wiring:**
```java
@EnableCaching
@Cacheable(value = "product", key = "#id", unless = "#result == null")
public Product get(String id) { ... }
```
```yaml
spring.data.redis.host: my-cluster.xxxx.ng.0001.aps1.cache.amazonaws.com
spring.cache.type: redis
spring.cache.redis.time-to-live: 10m
spring.cache.redis.cache-null-values: false
```
**What to volunteer (this is where seniority shows):**
- **Always set a TTL** — an unbounded cache is a memory leak with extra steps.
- **Cache stampede / thundering herd** — on TTL expiry, 500 threads hit the DB. Fix with jittered TTLs, a mutex/`SETNX` lock, or `Cache-Aside + refresh-ahead`.
- **Patterns:** cache-aside (default), write-through, write-behind. Know which you used.
- **Invalidation:** event-driven eviction over TTL guessing where correctness matters.
- **Serialization:** default JDK serialization is a trap — use `GenericJackson2JsonRedisSerializer` so cache entries survive a class change and stay debuggable.
- **Redis is single-threaded per shard** — no `KEYS *` in prod, use `SCAN`.
- **Also use it for:** distributed locks (Redisson / `SET NX PX`), rate limiting, Spring Session for stateless app tiers.
- **Connection to a cluster:** Lettuce (Spring Boot default, netty, thread-safe) over Jedis; enable `ReadFrom.REPLICA_PREFERRED` for read scaling; use the **configuration endpoint** in cluster mode.

---

### Q23. RDS Proxy — why does a Spring Boot developer care?

**Say this first:** "Connection pooling *outside* the app. It matters when I have many small compute units — Lambdas or 50 Fargate tasks each with a HikariCP pool of 10 — that would collectively exhaust RDS's `max_connections`."

Benefits: pools and multiplexes connections, **cuts failover time by up to 66%** (holds client connections while the DB fails over — the app barely notices), integrates with Secrets Manager + IAM auth so no password in the app.
Cost: ~$0.015/vCPU-hour of the DB instance; adds ~5ms.

**Do the arithmetic out loud:** "200 Lambda concurrency × 1 connection each = 200 connections; a `db.t3.medium` allows ~90. That's the outage. RDS Proxy or reserved concurrency fixes it."

**Why connections are a scarce resource:** every DB connection costs memory on the database server (and in Postgres, a whole backend process). `max_connections` is derived from instance memory, so small instances allow only dozens to a few hundred. A connection pool (HikariCP) works *within one app instance*; it can't coordinate across 50 tasks or hundreds of Lambdas. Total demand = instances × pool size.

**How RDS Proxy helps:** your app connects to the **proxy endpoint** instead of the DB. The proxy keeps a *small* pool of real DB connections and **multiplexes** many client sessions over them (lending a DB connection only for the duration of a transaction when possible). Thousands of clients → a few dozen DB connections.
- *Failover benefit:* during a Multi-AZ/Aurora failover the proxy keeps client connections open and reconnects to the new primary itself, so the app sees a short pause instead of connection errors.
- *Security benefit:* the proxy reads credentials from Secrets Manager and can require IAM auth from clients.

**Pitfalls:** *connection pinning* — if a session uses features such as temp tables, `SET` session variables, or prepared statements in some engines, the proxy must dedicate (pin) a DB connection to that client and multiplexing efficiency drops (check the `DatabaseConnectionsCurrentlySessionPinned` metric). Also, it must be in the same VPC (not publicly accessible), and keep your own Hikari `maxLifetime` shorter than the proxy's idle-client timeout.

---

## 6. Networking

### Q24. Design a VPC for a production Spring Boot application.

**Say this first, drawing as you go:** "One VPC, /16. Three AZs. Three subnet tiers per AZ: public for ALB/NAT, private-app for ECS tasks, private-data for RDS/ElastiCache. Internet Gateway on the public route table, NAT Gateway per AZ for egress, VPC endpoints for S3/DynamoDB/ECR/Secrets to keep traffic off the internet."

```
VPC 10.0.0.0/16
├── AZ-a  10.0.0.0/20   public   → IGW, ALB, NAT-a
│         10.0.16.0/20  app      → ECS tasks, route 0.0.0.0/0 → NAT-a
│         10.0.32.0/20  data     → RDS primary, Redis  (no route to internet)
├── AZ-b  ... same ...
└── AZ-c  ... same ...
Endpoints: S3 + DynamoDB (Gateway, free) | ECR-api, ECR-dkr, logs, secretsmanager, sts, ssm (Interface/PrivateLink)
```

**Why each tier exists (jargon decoded):**
- **VPC** (Virtual Private Cloud) = your own isolated network inside AWS. **CIDR** `10.0.0.0/16` = the private IP range (/16 ≈ 65k addresses) you carve into **subnets**; each subnet lives in exactly one AZ.
- **Public subnet** = its route table sends `0.0.0.0/0` to an **Internet Gateway (IGW)**, so resources with public IPs can talk to the internet. Put only the ALB and NAT here.
- **Private-app subnet** = no inbound route from the internet. Tasks reach out (to download from a SaaS API, etc.) via a **NAT Gateway**, a managed box in the public subnet that translates private IPs to its own public IP for outgoing traffic only.
- **Private-data subnet** = no internet route at all. Databases and caches are reachable only from the app tier (enforced with security groups). If an app is compromised, the attacker still can't reach the DB from the internet.
- **VPC endpoints** = private doorways to AWS services, so e.g. ECS pulling an image from ECR or reading S3 doesn't leave the AWS network or pay NAT charges. *Gateway* endpoints (S3/DynamoDB) are route-table entries; *Interface* endpoints (PrivateLink) are ENIs with private IPs.
- **Request flow:** user → Route 53 → ALB (public subnet) → ECS task (app subnet) → RDS (data subnet). Each hop only accepts traffic from the previous tier's security group.

**Points that earn credit:**
- Size subnets generously — **you can't resize a subnet**, and `awsvpc` mode burns one IP per task. A /24 dies fast.
- AWS reserves **5 IPs per subnet** (first four + last).
- **NAT Gateway is per-AZ for HA** *and* to avoid cross-AZ data charges — a single shared NAT is both an SPOF and a hidden bill.
- **Gateway endpoints (S3/DynamoDB) are free** and remove the biggest NAT data-processing cost. Interface endpoints cost ~$7/mo each + data but are required for ECR/Secrets/Logs in fully private subnets.
- **Never overlap CIDRs** with on-prem or other VPCs you'll peer — plan an IPAM range on day one.

---

### Q25. Security Group vs NACL.

| | Security Group | NACL |
|---|---|---|
| Level | ENI / instance | Subnet |
| State | **Stateful** — return traffic auto-allowed | **Stateless** — must allow both directions |
| Rules | Allow only | Allow **and Deny** |
| Evaluation | All rules evaluated | **Numbered, first match wins** |
| Default | Deny all inbound, allow all outbound | Default NACL allows all |

🧠 **"SG = bouncer who remembers your face (stateful). NACL = a numbered checklist at the street (stateless, first match wins)."**

**The senior answer:** "I use SGs as the primary control and **reference SGs instead of CIDRs** — the RDS SG allows 5432 *from the ECS task SG*, not from `10.0.16.0/20`. That way the rule stays correct when subnets change. NACLs I leave default except for explicit IP blocklists, because stateless rules and ephemeral port ranges (1024–65535) are an easy way to cause a 3 AM outage."

**Classic debug question — "app can't reach RDS":** walk the path: SG outbound on app → SG inbound on DB (port + source SG) → route table → NACL both directions → subnet AZ → DNS resolution → `max_connections` → credentials. Say it in that order; the ordered checklist *is* the answer.

**Plain-language explanation:**
- Both are virtual firewalls. A **Security Group (SG)** is attached to a network interface (an EC2 instance, ECS task, RDS instance, ALB). A **NACL (Network ACL)** is attached to a *subnet* and filters everything entering/leaving it.
- **Stateful** means the firewall remembers connections: if your app's outbound request to the DB is allowed, the DB's reply is automatically allowed back. **Stateless** (NACL) has no memory: the reply is a brand-new packet judged against the rules, so you must also allow the return traffic on **ephemeral ports** (1024–65535, the random ports clients use for replies).
- *Why two layers:* defence in depth. SGs are your day-to-day tool; NACLs are a coarse safety net at the subnet edge.

**Example — three-tier SG chain:**
| SG | Inbound rule |
|---|---|
| `alb-sg` | 443 from `0.0.0.0/0` |
| `app-sg` | 8080 from **`alb-sg`** |
| `db-sg` | 5432 from **`app-sg`** |

Nothing but the ALB can reach the app, nothing but the app can reach the DB — even if an attacker is inside the VPC. When ECS scales from 3 to 30 tasks no rule changes are needed, because the rule references the SG, not IPs.

**When NACLs earn their keep:** blocking a specific abusive IP range for a whole subnet (SGs can't *deny*), or a compliance requirement for subnet-level controls. **Pitfall:** NACL rules are evaluated by number, lowest first; put a `Deny` at 90 and `Allow` at 100, not the other way round.

---

### Q26. ALB vs NLB vs CLB vs Gateway LB. Which for Spring Boot?

**What a load balancer does:** it gives clients one stable address and spreads their requests over many healthy backend copies, removing sick ones using health checks. The "Layer 4 / Layer 7" wording refers to the network model: **Layer 4** sees only TCP/UDP connections (IPs + ports) and is therefore very fast but "blind"; **Layer 7** understands HTTP (URL path, headers, cookies), so it can make smart routing decisions. Example ALB rules: `/api/orders/*` → orders target group, `/api/users/*` → users target group, `host = admin.example.com` → admin service — one ALB fronting many microservices.

- **ALB (Layer 7)** — HTTP/HTTPS/gRPC/WebSocket. Host & path routing, header/query/method conditions, weighted target groups (canary), OIDC/Cognito auth, WAF integration, sticky sessions, redirect/fixed-response rules. **This is the Spring Boot answer.**
- **NLB (Layer 4)** — TCP/UDP/TLS, millions of RPS, ultra-low latency, **static IP per AZ + Elastic IP support**, preserves source IP. Use for non-HTTP protocols, extreme performance, or when a client needs to whitelist a fixed IP.
- **GWLB** — inserts third-party firewalls/IDS inline (GENEVE).
- **CLB** — legacy. Only correct answer: "migrate off it."

**Details that matter in practice:**
- ALB idle timeout default **60s** — must be **less than** your Tomcat `keepAliveTimeout`/`connectionTimeout`, or you get intermittent 502s from a race where the backend closes a connection ALB is reusing. Classic senior-level gotcha: set ALB idle 60s, Tomcat keep-alive 65s.
- **X-Forwarded-For / -Proto** — set `server.forward-headers-strategy=framework` (or `native`) in Spring Boot, otherwise generated URLs and redirects come out `http://` behind a TLS ALB and client IPs are all the ALB's.
- Health check on `/actuator/health/readiness`, healthy threshold 2, interval 10s, timeout 5s.
- **Cross-zone load balancing:** on by default for ALB (free), off by default for NLB (and cross-AZ data charges apply).
- 5xx troubleshooting 🧠: **502** = bad response/connection to target (app crash, keep-alive race), **503** = no healthy targets or capacity, **504** = target too slow (exceeded idle timeout), **460** = client closed early, **HTTP 000 in logs** = TLS issue.

**Why the keep-alive race causes 502 (step by step):** ALB reuses open connections to your app (HTTP keep-alive) to save handshakes. If Tomcat closes an idle connection at the same moment (or earlier than) ALB decides to reuse it — Tomcat's keep-alive timeout defaults to its connection timeout, 60 s, the same as ALB's 60 s idle timeout — ALB sends a request on a connection that's just been closed → connection reset → ALB returns **502**. Making the backend timeout *longer* than ALB's means ALB always closes first. In Spring Boot: `server.tomcat.keep-alive-timeout=65s`.

**Other pieces:** a **target group** is the set of backends (ECS tasks/IPs/instances) plus the health-check definition. **Sticky sessions** pin a user to one target via a cookie — a crutch for in-memory sessions; prefer stateless. **Deregistration delay** = how long ALB keeps finishing in-flight requests to a target being removed (see graceful shutdown in Q41).

**Which when (quick example):** public REST API on ECS → ALB. gRPC/WebSocket chat → ALB (supports both). MQTT/custom TCP protocol or "partner needs a fixed IP to whitelist" → NLB (often with ALB as an NLB target when you need both static IPs and HTTP routing). Inline firewall appliances → GWLB. Anything on CLB → plan a migration.

---

### Q27. Route 53 routing policies.

🧠 **"SWLFGM-Multi"**: **S**imple, **W**eighted, **L**atency, **F**ailover, **G**eolocation, **G**eoproximity, **M**ulti-value.

- **Simple** — one record, no health check.
- **Weighted** — 90/10 for canary or blue/green at the DNS level.
- **Latency-based** — route to the lowest-latency region. The default for multi-region APIs.
- **Failover** — active/passive with health checks, e.g. fail over to an S3 static "we'll be back" page.
- **Geolocation** — compliance/data-residency ("EU users must hit `eu-west-1`").
- **Geoproximity** — with a bias shift; for gradual traffic migration.
- **Multi-value answer** — up to 8 healthy records; poor-man's LB for non-ALB targets.

**Alias vs CNAME:** Alias is AWS-specific, works at the **zone apex** (`example.com`), is **free** to query, and auto-tracks the target's IPs. CNAME can't be at the apex and costs per query. 🧠 **"Apex → Alias."**

**Mention:** DNS TTL is the enemy of fast failover — combine low TTL with health checks, or use **Global Accelerator** (anycast static IPs, failover in seconds with no DNS dependency) when seconds matter.

**What Route 53 does (plain language):** it's AWS's DNS — the phone book that turns `api.example.com` into an IP/ALB address. A *routing policy* decides **which answer** it hands out when multiple exist, optionally depending on health checks, user location, or latency. DNS answers are cached by resolvers for the record's **TTL** (time-to-live), which is why DNS changes aren't instant.

**Scenarios to anchor each policy:**
- *Weighted:* release v2 on a new ALB, send 5% of DNS answers to it, raise to 25%, 100%. Weights are per-answer probability; clients with cached answers shift slowly.
- *Latency:* same API deployed in Mumbai and Frankfurt — each user is directed to whichever Region AWS measures as lower latency for them.
- *Failover:* primary record has a **health check**; when it fails, Route 53 starts returning the secondary (standby Region or S3 maintenance page).
- *Geolocation:* answers based on the user's country/continent — e.g. Indian users must be served by `ap-south-1` for data-localisation; always add a *default* record for unmatched locations.
- *Multi-value:* returns up to 8 healthy IPs; client picks one — simple health-checked distribution without an LB.

**Alias detail:** an Alias record points to an AWS resource (ALB, CloudFront, S3 website) and Route 53 resolves it internally, which is why it can sit at the root domain, where a CNAME is forbidden by the DNS spec.

**Pitfall:** health checks must reflect real readiness (point them at a lightweight endpoint, ideally a calculated check over dependencies), otherwise Route 53 fails over for the wrong reasons — or not at all.

---

### Q28. CloudFront — what do you actually use it for?

CDN over 600+ POPs. Uses:
1. **Static assets / SPA** — S3 origin + **OAC** (bucket stays private), long `max-age` with content-hashed filenames.
2. **Dynamic API acceleration** — even with `CachePolicy: CachingDisabled`, requests ride AWS's backbone from the edge = lower TLS handshake and RTT cost.
3. **Security edge** — AWS WAF, Shield Standard (free DDoS), geo-restriction, signed URLs/cookies for paid content.
4. **Edge compute** — CloudFront Functions (lightweight header/URL rewrites, sub-ms) vs Lambda@Edge (heavier, can call origins/AWS APIs).

**Know:** cache key components (headers/cookies/query strings — **each variation multiplies your cache and lowers hit rate**), invalidation (`/*` is slow and the first 1000 paths/month are free — prefer **versioned filenames** over invalidation), `Cache-Control` vs CloudFront TTL precedence, and origin failover groups.

**How CloudFront works, step by step:** user in Delhi requests `app.example.com/main.3f9a.js` → DNS directs them to the nearest edge location → edge checks its cache: **hit** → returns immediately (a few ms); **miss** → edge fetches from the *origin* (S3 bucket or your ALB) over AWS's backbone, caches it per the TTL, returns it. The next user nearby gets a hit. *Why:* lower latency for far-away users, lower load/egress on your origin, and a shield in front of it.

**Terms:** **origin** = where the real content lives; **distribution** = one CloudFront configuration with a domain name; **behaviour** = path-pattern rule (`/static/*` → S3 cached for a year, `/api/*` → ALB uncached); **OAC** (Origin Access Control) = lets CloudFront sign requests so only it can read the private S3 bucket; **TTL** = how long an object stays cached.

**Spring Boot angle:** serve the SPA from S3+CloudFront and route `/api/*` to the ALB under the *same domain* — this also avoids CORS headaches. For APIs, forward the `Authorization` header and needed query strings to the origin, and keep caching off for personalised responses; otherwise you could serve user A's data to user B (a common, serious cache-key bug). Emit correct `Cache-Control` headers from Spring (`CacheControl.maxAge(...)`).

**Pitfall:** why content-hashed filenames beat invalidations — `main.3f9a.js` changes name when content changes, so the new file is automatically a cache miss, old ones expire naturally, and no invalidation (slow, quota-limited) is needed; only `index.html` needs a short TTL.

---

### Q29. API Gateway vs ALB in front of Spring Boot.

**Choose API Gateway when you need:** per-client API keys and usage plans, throttling/quotas, request validation, Lambda/Cognito/JWT authorizers, request-response transformation, WebSocket APIs, or a stable public contract with staged deployments and canary releases.
**Choose ALB when:** it's a container-based REST service, high throughput, cost matters. ALB is **much cheaper at volume** (API Gateway REST ~$3.50/million requests vs ALB's hourly + LCU model).
**HTTP API (v2)** is ~70% cheaper and lower-latency than REST API — fewer features (no usage plans/WAF on v2 historically), use it when you just need JWT auth + routing.

**Say this:** "For internal service-to-service I use ALB or service discovery. For a public, partner-facing, monetized API I use API Gateway because throttling and API keys are business features I don't want to build in Spring."

**What each is (plain language):** an **ALB** is a load balancer — it spreads HTTP traffic across your running containers and nothing more. **API Gateway** is a fully managed *API front door*: it terminates TLS, authenticates callers, rate-limits them, validates and optionally transforms requests, and then forwards to a backend (Lambda, an HTTP URL, or a private VPC service via VPC Link). It takes over a lot of cross-cutting concerns you'd otherwise code in Spring (filters, bucket4j rate limiting, API-key checking).

**Concrete example:** you expose `/v1/payments` to 20 partner companies. Each gets an API key tied to a *usage plan*: Partner A = 100 req/s and 1M requests/month, Partner B = 10 req/s. API Gateway enforces this and returns **429 Too Many Requests** before the call reaches your service. With only an ALB you'd build and operate that logic yourself (or add WAF rate-based rules, which are per-IP, not per-customer).

**Typical wiring:** Client → API Gateway (HTTP API + JWT authorizer from Cognito/OIDC) → **VPC Link** → internal ALB/NLB → ECS Spring Boot service in private subnets (so the service isn't public at all).

**Gotchas / probes:** API Gateway has an integration timeout of ~29 s by default (a hard limit for REST APIs, extendable only through quotas for Regional/Private APIs), so long requests must be async (return 202 + poll/callback). Payload limit is 10 MB. Cost grows linearly per request, so at hundreds of millions of calls/month a plain ALB is far cheaper. Don't stack API Gateway + ALB + CloudFront without a reason — every hop adds latency, cost, and a place for timeouts to mismatch.

---
## 7. Messaging & Event-Driven

### Q30. SQS vs SNS vs EventBridge vs Kinesis vs MSK. Pick one for each scenario.

🧠 **One-liners to memorize:**
- **SQS** = "a to-do list, one worker takes each task" → **queue, pull, decoupling, buffering**.
- **SNS** = "a megaphone to many subscribers" → **pub/sub fan-out, push**.
- **EventBridge** = "a smart router with filtering and schemas" → **event bus, content-based routing, SaaS + AWS events, cron**.
- **Kinesis Data Streams** = "an ordered replayable tape" → **streaming, ordering per shard, multiple independent readers, replay**.
- **MSK** = "managed Kafka" → when you're already Kafka, need long retention/compaction, or huge throughput with Kafka ecosystem (Connect, Streams).

| Need | Pick |
|---|---|
| Smooth out a traffic spike into a slow downstream | SQS |
| One event → 5 unrelated consumers | SNS (or EventBridge) |
| Route events by content (`detail.orderTotal > 1000`) | EventBridge |
| Scheduled job (cron) | EventBridge Scheduler |
| Strict ordering per customer + replay for 7 days | Kinesis (or SQS FIFO if no replay needed) |
| Clickstream at 1M events/sec into analytics | Kinesis / MSK |
| Existing Kafka app lifted to AWS | MSK |

**The canonical pattern to name-drop: SNS → SQS fan-out.** One publish, multiple durable queues, each consumer scales and retries independently. Add **message filter policies** on the subscriptions so each queue only receives what it cares about. This is the answer to most "how do you decouple these microservices" questions.

**Why these services exist (the problem):** if Order Service calls Email, Inventory, and Analytics directly over HTTP, a slow or failing downstream slows or breaks checkout, and adding a new consumer means changing Order Service. Messaging makes the call **asynchronous and decoupled**: Order Service drops an event and moves on; consumers process at their own pace and can be added without touching the producer.

**Mental model for choosing:**
- *Queue (SQS):* each message is handled by **one** consumer (competing consumers); consumed messages are deleted. Use it to **buffer and distribute work**.
- *Topic (SNS):* each message goes to **all** subscribers right now (push); no storage if a subscriber is down (unless that subscriber is an SQS queue — hence the SNS → SQS combo).
- *Event bus (EventBridge):* like SNS but with rules that inspect the **content** of the event, many targets (Lambda, SQS, Step Functions, API destinations), schema registry, archive/replay, and cron scheduling.
- *Stream (Kinesis/MSK):* an append-only **log**; consumers track their own position and can re-read history. Use when ordering, high throughput, or replay matters.

**Example flow (e-commerce):** `OrderPlaced` → SNS topic → (a) `email-queue` (filter: all), (b) `inventory-queue`, (c) `fraud-queue` (filter: `orderTotal > 50000`). Each queue has its own DLQ; if Inventory is down for an hour, its queue just grows and drains later — checkout never notices.

**Probe:** "Why not just use SNS to HTTP endpoints?" → no buffering or retry visibility, consumer must be up, thundering herd on recovery; SQS in the middle gives durability, backpressure, and DLQs.

---

### Q31. SQS deep dive: standard vs FIFO, visibility timeout, DLQ, long polling.

- **Standard** — nearly unlimited throughput, **at-least-once** delivery, best-effort ordering. → **Your consumer must be idempotent.**
- **FIFO** — exactly-once processing, strict order within a **MessageGroupId**, 300 msg/s (3000 with batching), up to 70,000/s with high-throughput mode. `MessageDeduplicationId` gives a 5-minute dedup window.

**Visibility Timeout** — after a receive, the message is hidden for N seconds (default 30, max 12h). If you don't delete it in time, it reappears and gets processed twice.
🧠 **"Visibility timeout must be longer than your processing time."** Set it to ~6x expected processing, or call `ChangeMessageVisibility` as a heartbeat for long jobs.

**DLQ** — a `RedrivePolicy` with `maxReceiveCount` (typically 3–5). After N failures the message moves to the DLQ. **Then: alarm on `ApproximateNumberOfMessagesVisible > 0` on the DLQ** — a DLQ nobody watches is just a data-loss folder. Use **DLQ redrive** to replay after a fix.

**Long polling** — `WaitTimeSeconds = 20`. Cuts empty receives, cuts cost, cuts latency vs short polling. Always on.

**Other must-knows:** max message 256 KB (use the **Extended Client Library** → payload in S3, pointer in the message), retention 4 days default / 14 max, delay queues (up to 15 min), message groups, and **no** pub/sub — one consumer per message.

**How the SQS lifecycle works (step by step):**
1. Producer `SendMessage` → SQS stores it redundantly across AZs.
2. Consumer `ReceiveMessage` → gets up to 10 messages; SQS does **not** delete them, it just hides them for the *visibility timeout*.
3. Consumer processes, then calls `DeleteMessage`. If it crashes or is too slow, the timer expires and the message becomes visible again for another consumer → this is how SQS gives **at-least-once** delivery (and why duplicates happen).
4. Each re-delivery increments `ApproximateReceiveCount`; once it exceeds `maxReceiveCount`, SQS moves it to the **dead-letter queue (DLQ)** — a normal queue where unprocessable ("poison") messages wait for human attention.

**Worked example — visibility timeout bug:** processing a report takes 90 s but the timeout is the default 30 s. At 30 s another consumer receives the same message and starts the same report; you now have duplicates, wasted compute, maybe double emails. Fix: timeout ≥ 6× typical processing (or heartbeat with `ChangeMessageVisibility`), *and* make the handler idempotent.

**Standard vs FIFO — how to decide:** Standard for almost everything (high throughput, cheaper). FIFO when order matters *per entity* — e.g. `AccountDebited` must be applied before `AccountClosed` for the same account. `MessageGroupId = accountId` keeps each account's messages in order while different accounts process in parallel; one slow/failing message **blocks only its group** (and, with a stuck message, the group's later messages wait — design DLQs accordingly).

**Short vs long polling:** short polling returns immediately (often empty, querying only a subset of servers) → many wasted paid API calls. Long polling waits up to 20 s for a message, so you get fewer calls and fewer false empties.

**Probe — "Is FIFO really exactly-once?"** Exactly-once on the *send* side (dedup within 5 min) and no duplicates once consumed *successfully*; your consumer can still crash after side effects but before delete, so keep idempotency for external effects anyway.

**Spring Boot consumer:**
```java
@SqsListener(value = "orders-queue", maxConcurrentMessages = "10")
public void handle(@Payload OrderEvent event,
                   @Header("MessageId") String messageId) {
    if (processedRepo.existsById(messageId)) return;   // idempotency guard
    orderService.process(event);
    processedRepo.save(new Processed(messageId));      // same TX as the work, ideally
}
```
Say: "Acknowledgement on normal return, message stays for retry on exception. I keep the handler idempotent because Standard SQS is at-least-once — I dedupe on a business key or message ID in DynamoDB with a TTL."

---

### Q32. What is EventBridge and why prefer it over SNS?

An event bus with **content-based routing**. A rule matches a JSON pattern and routes to up to 5 targets.

```json
{
  "source": ["orders.service"],
  "detail-type": ["OrderPlaced"],
  "detail": { "amount": [{ "numeric": [">", 1000] }], "region": ["IN"] }
}
```

**Prefer EventBridge when:** you need filtering on payload content, 20+ target types without glue code, **Schema Registry** with code bindings, archive & **replay** of past events, cross-account/cross-region buses, third-party SaaS events (Stripe, Datadog), or **EventBridge Pipes** (source → filter → enrich → target with zero Lambda).
**Prefer SNS when:** you need very high fan-out throughput at low latency and low cost, or SMS/email/mobile push. SNS latency is lower (~ms) vs EventBridge (~sub-second to seconds).

🧠 **"SNS = fast dumb pipe. EventBridge = slower smart router."**

**Plain-language explanation:** EventBridge is a *managed event bus*: services publish small JSON "facts" (`OrderPlaced`, `EC2 instance stopped`, `Stripe payment failed`) to a bus; **rules** (pattern matchers) decide which events go to which **targets**. The producer doesn't know who listens. *Why prefer it:* with SNS you can only filter on message attributes, while EventBridge can filter on **any field in the body**, route to many AWS services directly (Step Functions, Lambda, SQS, API destinations to call external HTTP APIs), and replay archived events after fixing a bug.

**How it works:** (1) publish with `PutEvents` (`source`, `detail-type`, `detail`) → (2) bus evaluates each rule's pattern → (3) for every match it invokes up to 5 targets, with retries (up to 24 h) and an optional DLQ per target. The example pattern above matches only India orders above ₹1000.

**Worked scenario:** `OrderPlaced` events go to the default/custom bus. Rule A (all orders) → SQS for fulfilment. Rule B (`amount > 1000`) → fraud-check Lambda. Rule C (`region = IN`) → GST-invoicing queue. The Order Service publishes once and never changes when a new rule is added.

**Other features:** **EventBridge Scheduler** replaces cron-style Lambdas (one-off and recurring schedules); **Pipes** connect a source (SQS, DynamoDB stream, Kinesis) to a target with optional filter/enrichment and no glue code; **Archive & Replay** lets you re-send past events.

**Pitfalls:** delivery is **at-least-once** (consumers still need idempotency); no strict ordering; default quotas on `PutEvents` throughput; event size limit 256 KB; always set a DLQ on targets so failed deliveries aren't silently dropped.

---

### Q33. How do you guarantee no lost or duplicated messages between two Spring Boot services?

This is a *design* question. Structure the answer:

1. **Don't lose it on publish → Transactional Outbox.** In the same DB transaction as the business write, insert a row into an `outbox` table. A separate poller (or DynamoDB Streams / Debezium / MSK Connect) publishes it. Solves the dual-write problem — you never have "DB committed but event not sent" or vice versa.
2. **Don't lose it in flight** → durable queue (SQS/Kinesis), DLQ, retention, encryption.
3. **Don't lose it on consume** → delete/ack only after successful commit. Visibility timeout > processing time.
4. **Handle duplicates → idempotent consumer.** Unique constraint on `(message_id)` or `(aggregate_id, version)`, or a conditional `PutItem` in DynamoDB with `attribute_not_exists`. 🧠 **"At-least-once delivery + idempotent consumer = effectively exactly-once."**
5. **Poison messages** → `maxReceiveCount` + DLQ + alarm + redrive.
6. **Ordering** → SQS FIFO `MessageGroupId = customerId`, or Kinesis partition key = aggregate ID. Accept that ordering and parallelism trade off.
7. **Verify it** → correlation IDs end-to-end, X-Ray traces, and a reconciliation job that compares source-of-truth counts.

If you can deliver points 1 and 4 crisply, you sound like someone who's shipped event-driven systems.

**Why points 1 and 4 matter (the underlying problems):**
- **Dual-write problem:** `orderRepo.save(order); sqs.send(event);` are two separate systems with no shared transaction. If the app crashes between them (or the send fails), the DB says "order exists" but nobody was told — a *lost* event. Reversing the order risks announcing an order that was rolled back.
- **Transactional outbox fixes it** by making the event part of the *same* local DB transaction:
```java
@Transactional
public void placeOrder(Order o) {
    orderRepo.save(o);
    outboxRepo.save(new OutboxEvent(UUID.randomUUID(), "OrderPlaced", toJson(o)));  // same TX
}
// separate scheduled publisher (or CDC via Debezium / DynamoDB Streams)
@Scheduled(fixedDelay = 1000)
void publish() {
    outboxRepo.findUnsent(100).forEach(e -> { sqs.send(e); e.markSent(); });
}
```
  If the publisher crashes after sending but before `markSent()`, the event is sent twice — which is acceptable because the consumer is idempotent. So the outbox gives **at-least-once**, and the idempotent consumer converts that into effectively-once.
- **Idempotent consumer** = processing the same message twice has the same result as once. Typical technique: table `processed_messages(message_id PRIMARY KEY)`; insert it in the same transaction as the business update — a duplicate violates the key and is skipped.

**Pitfalls to mention:** don't dedupe with an in-memory set (lost on restart, not shared across instances); clean up old outbox/processed rows; for several publisher instances use `SELECT ... FOR UPDATE SKIP LOCKED` to avoid double-claiming rows.

---

### Q34. Kinesis Data Streams — shards, and how do you scale consumers?

- A **shard** = 1 MB/s or 1000 records/s in, 2 MB/s out. Records land in a shard by **partition key hash**; order is guaranteed **per shard**, not per stream.
- **On-demand mode** removes shard math (scales to 200 MB/s automatically) — say you prefer it unless cost at known scale justifies provisioned.
- **Consumers:** shared throughput (2 MB/s split across all consumers, polling) vs **Enhanced Fan-Out** (2 MB/s *per consumer*, pushed via HTTP/2, ~70ms latency).
- **KCL** checkpoints in a DynamoDB table; one worker per shard → **parallelism is capped by shard count**.
- Retention 24h default → up to 365 days. **Replay is the superpower** SQS doesn't have.
- **Firehose** vs Data Streams: Firehose is zero-admin delivery to S3/Redshift/OpenSearch with buffering and transformation, but no replay and ~60s latency. 🧠 **"Streams = you process it. Firehose = you just want it delivered."**

**Hot shard** answer: "A partition key like `country=IN` funnels everything into one shard. I use a high-cardinality key, or a composite key with a random suffix if strict ordering isn't needed."

**Plain-language explanation:** Kinesis Data Streams is a durable, ordered **log** that many producers append to and many consumers read — think of a conveyor belt that keeps the last 24 h (up to 365 days) of items so you can rewind. *Why not SQS?* SQS deletes messages once consumed and has no replay; if two teams need the same data, or you need to reprocess last Tuesday's events after a bug fix, you need a stream.

**How shards and keys work:** a stream is split into **shards** (lanes); each lane gives a fixed capacity (1 MB/s write). When a producer sends a record with partition key `customer-42`, Kinesis hashes the key to choose the lane — so all of a customer's records go to the same shard in order. More shards = more throughput and more parallel consumers.

**Scaling consumers:** with the KCL (Kinesis Client Library), each shard is processed by exactly one worker at a time, and the worker saves its **checkpoint** (last processed sequence number) in DynamoDB. Example: 8 shards → at most 8 parallel workers; adding a 9th does nothing. To go faster you **reshard** (split shards), not add more consumers. If one consumer is slow its shard's `GetRecords.IteratorAgeMilliseconds` rises — alarm on it.

**Choosing a consumer type:** shared throughput is fine for 1–2 consumers; with 5 consumers each only gets ~0.4 MB/s of the 2 MB/s read limit, so use **Enhanced Fan-Out** (dedicated 2 MB/s per consumer) — pay extra for it.

**Spring Boot angle:** Spring Cloud Stream Kinesis binder or the KCL v3 / SDK `SubscribeToShard`; keep handlers idempotent because retries and shard re-assignment cause re-delivery; use `PutRecords` batching (up to 500 records) on the producer side.

---

## 8. Observability

### Q35. How do you monitor a Spring Boot app on AWS end to end?

**Say this first:** "Three pillars — metrics via Micrometer to CloudWatch, structured JSON logs to CloudWatch Logs, and distributed traces via X-Ray or OTel — all tied together by a correlation ID."

**Metrics:**
```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-cloudwatch2</artifactId>
</dependency>
```
```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.probes.enabled: true       # /readiness and /liveness
  metrics.export.cloudwatch:
    namespace: orders-service
    step: 1m
```
- Actuator + Micrometer gives you `http.server.requests` (with percentiles), `jvm.memory.used`, `jvm.gc.pause`, `hikaricp.connections.active`, `tomcat.threads.busy` for free.
- **Cost warning to volunteer:** custom metrics are $0.30/metric/month and **high-cardinality tags (userId, orderId) will blow up your bill**. Keep dimensions low-cardinality: uri, method, status, outcome.
- Alternative worth naming: **EMF (Embedded Metric Format)** — print a JSON log line, CloudWatch extracts metrics from it. Cheaper and asynchronous. Or run Prometheus/Grafana/AMP if the org already has it.

**Logs:** stdout → `awslogs`/FireLens driver. **JSON structured** (`logstash-logback-encoder`) so **CloudWatch Logs Insights** can query fields:
```
fields @timestamp, level, traceId, message
| filter level = "ERROR" and service = "orders"
| stats count() by errorCode
| sort count() desc
```
Set **log retention** (default is Never Expire = a permanent bill). Use metric filters → alarms for `ERROR` spikes.

**Traces:** X-Ray daemon/ADOT sidecar, `aws-xray-recorder-sdk-spring`, or **OpenTelemetry** (my preference — vendor-neutral, ADOT collector exports to X-Ray *and* anything else). Propagate `X-Amzn-Trace-Id`; put `traceId` in the MDC so logs and traces join up.

**Alarms that matter (SLO-driven, not vanity):** p99 latency, 5xx rate, ALB `UnHealthyHostCount`, ECS `RunningTaskCount` vs desired, SQS queue age (`ApproximateAgeOfOldestMessage` — better than depth), DLQ depth, RDS `DatabaseConnections` + `CPUUtilization` + `FreeableMemory` + `ReadLatency`, HikariCP pending threads. Use **composite alarms** to kill alert fatigue and **anomaly detection** for seasonal metrics.

---

### Q36. CloudWatch vs CloudTrail vs Config vs X-Ray?

🧠 **"Watch performance, Trail people, Config compliance, X-Ray requests."**

- **CloudWatch** — metrics, logs, alarms, dashboards. *"Is it healthy?"*
- **CloudTrail** — API audit log: who called what, when, from which IP. *"Who deleted the RDS instance?"* Enable an **org-wide trail to a locked S3 bucket**; data events (S3/Lambda) are off by default and cost extra.
- **AWS Config** — resource configuration history + compliance rules. *"Was this SG ever open to 0.0.0.0/0, and when did it change?"*
- **X-Ray** — per-request distributed trace with a service map and latency breakdown. *"Which downstream call is making checkout slow?"*

**Why four tools (the problem each solves):** in an incident you ask different questions — *is it broken* (metrics/logs), *what changed* (Config), *who did it* (CloudTrail), *where in the call chain is the time going* (traces). Each tool records a different kind of evidence.

**Plain-language definitions:**
- **Metric** = a number over time (CPU %, requests/min). **Log** = a text/JSON record of something that happened. **Alarm** = a rule on a metric that notifies/acts when breached (e.g. 5xx rate > 2% for 3 of 5 minutes → SNS → pager).
- **Audit log** (CloudTrail) = records *API calls to AWS itself* (`DeleteDBInstance`, `PutBucketPolicy`) — not what your app does internally.
- **Trace** = one request's journey, broken into **spans** (ALB → orders → inventory → RDS), each with its own timing.
- **Config** keeps a timeline of each resource's settings and evaluates **rules** like "all S3 buckets must be encrypted", flagging non-compliance.

**Example investigation:** CloudWatch alarm: checkout p99 jumped. X-Ray trace shows 4 s in the `inventory` call. Logs Insights filtered by that traceId reveal DB timeouts. Config timeline shows the DB security group changed an hour ago. CloudTrail shows *which engineer* changed it. Four tools, one story.

**Pitfall:** CloudTrail event history in the console only keeps 90 days of management events; create a **trail** to S3 for long-term retention and for data events.

---

## 9. Config & Secrets Management

### Q37. How do you manage environment-specific config for Spring Boot across dev/stage/prod?

**Layered answer:**
1. **Non-secret, non-changing** → `application-{profile}.yml` in the artifact. Same immutable image everywhere; profile via `SPRING_PROFILES_ACTIVE`.
2. **Non-secret, environment-specific** → SSM Parameter Store hierarchy `/prod/orders/*`, imported with `spring.config.import: aws-parameterstore:/prod/orders/`.
3. **Secrets** → Secrets Manager, or ECS `secrets` block injecting them as env vars at task start (never in the task definition as plaintext `environment`).
4. **Dynamic runtime flags** → AppConfig (with validators + gradual rollout + auto-rollback) or a DynamoDB-backed flag table with a short cache.

🧠 **The rule: "Build once, deploy everywhere."** The image never changes between environments — only what's injected into it.

**Why this matters (the problem):** if you rebuild the artifact per environment (or bake `prod` values into it), what you tested in stage is *not* what runs in prod, and secrets leak into images. Treating the artifact as immutable and **injecting configuration at runtime** makes deployments reproducible and rollbacks trivial.

**How each layer works in practice:**
- *Profiles:* Spring loads `application.yml` then overlays `application-prod.yml` when `SPRING_PROFILES_ACTIVE=prod` is set on the ECS task. Good for things that rarely change (log levels, feature defaults, thread-pool sizes).
- *Parameter Store hierarchy:* params named `/prod/orders/db-url`, `/stage/orders/db-url`. The app is told only its prefix; the same code reads `db-url` in every environment. IAM policies can restrict each environment's role to its own path (`arn:aws:ssm:...:parameter/prod/orders/*`), so a stage task can't read prod values.
- *ECS `secrets` block:* ECS fetches the secret ARN when launching the task and exposes it to the container as an environment variable (requires the **execution role** to have `secretsmanager:GetSecretValue`). The task definition stores only the ARN, never the value.
```json
"secrets": [{ "name": "DB_PASSWORD",
  "valueFrom": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/orders/db-AbCdEf:password::" }]
```
- *AppConfig:* for values you want to change **without redeploying** (feature flags, rate limits), with staged rollout (e.g. 10% of instances for 10 min) and automatic rollback if a CloudWatch alarm fires; the app polls the AppConfig agent/SDK.

**Pitfalls:** values from the ECS `secrets`/env are read at *task start* — changing the secret requires a new deployment; Parameter Store has API throttling, so cache values and don't call it per request; keep the number of config sources small or precedence becomes confusing (Spring order: command-line args > env vars > `config.import` sources > profile files > defaults, roughly).

**Trap to avoid:** "I'd use Spring Cloud Config Server" — fine, but say *why* you'd choose it over Parameter Store (git-backed history, refresh scope, non-AWS portability) and note it's another service to run and secure.

---

## 10. IaC & CI/CD

### Q38. CloudFormation vs CDK vs Terraform.

- **CloudFormation** — native, declarative YAML/JSON, free, drift detection, StackSets for multi-account, change sets for preview. Verbose; no loops.
- **CDK** — write infra in **Java/TypeScript/Python**, synthesizes to CloudFormation. Constructs, sane defaults, loops and abstractions, unit-testable infra. **Sell this in a Java interview**: "I can write my infra in the same language as my service and unit test it."
- **Terraform** — multi-cloud, huge provider ecosystem, `plan` is best-in-class, state file to manage (S3 + DynamoDB lock). Most common in enterprises.

**Say this:** "I'm pragmatic — I use whatever the org already standardized on, because two IaC tools is worse than the wrong one. Personally CDK for app-adjacent resources and Terraform for the landing zone/networking."

**What IaC is and why:** Infrastructure as Code = describing VPCs, ECS services, queues, IAM roles in version-controlled files instead of clicking in the console. *Problems solved:* reproducibility (spin up an identical stage environment), review (infra changes go through pull requests), auditability, disaster recovery (rebuild from code), and avoiding manual-change "snowflake" environments.

**How they work:**
- **CloudFormation:** you submit a *template*; AWS creates a **stack** and tracks every resource. Updating = submit a changed template; a **change set** shows what will be added/replaced/deleted first. If an update fails, it rolls back automatically.
- **CDK:** you write code, e.g.
```java
Queue dlq = Queue.Builder.create(this, "OrdersDlq").build();
Queue q = Queue.Builder.create(this, "Orders")
    .visibilityTimeout(Duration.seconds(180))
    .deadLetterQueue(DeadLetterQueue.builder().queue(dlq).maxReceiveCount(5).build())
    .build();
q.grantConsumeMessages(taskRole);   // generates the least-privilege IAM policy for you
```
  `cdk synth` turns this into a CloudFormation template; `cdk diff` previews; `cdk deploy` applies. The `grant*` helpers are a big selling point (correct IAM with one line).
- **Terraform:** HCL files → `terraform plan` (shows diff) → `apply`. It tracks reality in a **state file**; keep it in S3 with DynamoDB (or S3-native) locking so two engineers can't apply at once, and never edit it by hand.

**How to choose (decision rule):** stay inside AWS and want typed code → CDK; multi-cloud or want a vendor-neutral language the platform team already knows → Terraform; small/simple or managed by tooling like Service Catalog → raw CloudFormation.

**Pitfalls:** console changes cause **drift** (code and reality differ) — detect with drift detection / `terraform plan`; set `DeletionPolicy: Retain` (or `prevent_destroy`) on databases/buckets; split stacks by lifecycle (network vs app) so a failed app deploy can't threaten the VPC; never put secrets in templates or state in plaintext.

Know: parameters vs mappings vs conditions, `!Ref` vs `!GetAtt`, nested stacks, `DeletionPolicy: Retain`, custom resources, and **stack drift** (someone changed it in the console).

---

### Q39. Describe your CI/CD pipeline for a Spring Boot microservice on AWS.

Walk it end to end:

```
Git push (feature branch)
  → CodeBuild / GitHub Actions
      mvn verify           (unit + Testcontainers integration tests)
      SpotBugs, Checkstyle
      SonarQube quality gate
      OWASP dependency-check / Snyk
  → docker buildx build --platform linux/arm64   (Graviton!)
      multi-stage Dockerfile, distroless/Alpine JRE base, non-root user, layered JAR
  → push to ECR (immutable tags = git SHA, scan-on-push via Inspector)
  → CDK/Terraform diff on infra
  → Deploy to DEV (ECS rolling)
  → Automated smoke + contract tests (Pact)
  → Manual approval gate
  → Deploy to PROD via CodeDeploy Blue/Green
      canary 10% for 10 min, CloudWatch alarms on 5xx + p99
      auto-rollback on alarm
  → Post-deploy verification + Slack notification via SNS/Chatbot
```

**Points to emphasize:**
- **Layered JARs** (`spring-boot-maven-plugin` layertools) → dependencies in a cached Docker layer, so a code change ships a 2 MB layer not a 200 MB one. Big build/deploy speedup.
- **Immutable image tags** (git SHA, never `latest`) — `latest` makes rollbacks unreproducible.
- **Same artifact promoted** through environments, never rebuilt per environment.
- **Rollback plan**: previous task definition revision is one API call away; DB migrations are **backward-compatible** (expand/contract) so the app can roll back even when the schema can't.
- **DB migrations**: Flyway/Liquibase run as a separate ECS task or an init step with a lock, not concurrently from 10 booting tasks.

**Stage-by-stage explanation (what each step prevents):**
- *`mvn verify` + Testcontainers:* runs unit tests and integration tests against a real throwaway Postgres/LocalStack in Docker, catching "works on my H2 database" bugs before deploy.
- *Static analysis/Sonar quality gate/dependency scan:* blocks merges that add bugs, code smells, or libraries with known CVEs (Common Vulnerabilities and Exposures).
- *Build image once:* multi-stage Dockerfile (compile in a JDK stage, copy only the layered JAR into a slim JRE stage), run as non-root, ARM64 for Graviton (cheaper/faster per $).
- *ECR + scan:* ECR stores images; tags are immutable so `orders:9f3c1ab` always means the same bytes; scan-on-push flags vulnerable OS packages.
- *Deploy dev → smoke/contract tests → approval → prod:* the **same image digest** is promoted. **Contract tests (Pact)** verify that a provider's API still satisfies what consumer services expect, so you can deploy services independently.
- *Canary + alarms + auto-rollback:* only 10% of traffic sees the new version for 10 minutes; if 5xx rate or p99 breaches the alarm, CodeDeploy shifts traffic back without a human.

**Example of expand/contract migration (so rollback is safe):** renaming column `phone` → `mobile`: release 1 adds `mobile` and writes to both; backfill; release 2 reads from `mobile`; release 3 (weeks later) drops `phone`. At every point the *previous* app version still works with the current schema.

**Pitfalls:** long-running migrations holding table locks during peak hours; running Flyway at startup in every task causes lock contention (use Flyway's lock or a dedicated job); pipeline secrets stored as plain CI variables instead of OIDC-federated short-lived AWS credentials (GitHub Actions → `aws-actions/configure-aws-credentials` with a role).

---

## 11. Spring Boot on AWS

> This section is where a Java developer wins the interview. Most candidates recite AWS docs;
> few can talk about the **JVM inside a container inside AWS**.

### Q40. What JVM settings do you change when running Spring Boot in a container on ECS/EKS?

```bash
JAVA_TOOL_OPTIONS="\
 -XX:MaxRAMPercentage=75.0 \
 -XX:InitialRAMPercentage=50.0 \
 -XX:+UseG1GC \
 -XX:+ExitOnOutOfMemoryError \
 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof \
 -Djava.security.egd=file:/dev/./urandom \
 -Duser.timezone=UTC"
```
**Explain each:**
- **`MaxRAMPercentage` not `-Xmx`** — the same image then works on a 1 GB and a 4 GB task. 75% leaves headroom for metaspace, thread stacks, direct buffers, and the JVM's own native memory. A container OOM-kill (exit 137) usually means someone set `-Xmx` = container memory and forgot non-heap.
- **`ExitOnOutOfMemoryError`** — a JVM limping along after OOM is worse than a dead one. Let ECS replace the task. 🧠 **"Crash fast, let the orchestrator heal."**
- **GC choice** — G1 for most services; **SerialGC** is genuinely faster for tiny (<1 vCPU) containers and Lambda; ZGC if you need sub-ms pauses and have the memory.
- **CPU** — the JVM sizes GC threads and the common ForkJoinPool from `availableProcessors()`. On Fargate with 0.5 vCPU it may see the host's cores. Verify; set `-XX:ActiveProcessorCount` if needed.
- **Startup** — **CDS/AppCDS** and **Spring AOT** cut startup 20–40%; matters because startup time × number of tasks = deploy duration.

**Why container JVM tuning matters (the background):** a JVM sizes its heap from the memory it *thinks* it has. Container-aware JVMs (Java 10+/8u191+) read the cgroup limit (e.g. the Fargate task's 2 GB) rather than the host's RAM. But heap is only part of a Java process's memory: **metaspace** (class metadata), **thread stacks** (~1 MB each × hundreds of Tomcat/async threads), **direct/NIO buffers** (Netty, S3 SDK), **code cache**, and GC structures all live outside the heap. If `Xmx` ≈ container limit, total usage exceeds the limit and the kernel kills the process (OOM-kill, exit code **137**) with *no Java stack trace* — confusing in CloudWatch.

**Flag-by-flag in plain terms:**
- `MaxRAMPercentage=75` → heap may use up to 75% of container memory. Example: 2 GB task → ~1.5 GB max heap.
- `InitialRAMPercentage=50` → start at a decent size so the heap doesn't resize repeatedly during startup.
- `+UseG1GC` → the default low-pause collector on multi-core machines; the JVM silently picks **SerialGC** when it sees <2 CPUs or <1792 MB RAM — a surprise on small Fargate tasks. Check with `-Xlog:gc*` or `jcmd VM.flags`.
- `ExitOnOutOfMemoryError` → process exits on OOM so ECS's health check/restart policy heals it, instead of a zombie JVM that still answers `/health` but fails requests.
- `HeapDumpOnOutOfMemoryError` → writes a dump for later analysis; **gotcha:** the container filesystem is ephemeral and disappears when the task stops, so point `HeapDumpPath` at a mounted volume/EFS or upload to S3 in a shutdown hook — otherwise you've captured nothing.
- `java.security.egd=file:/dev/./urandom` → avoids slow startup on old JDKs where `SecureRandom` blocked waiting for entropy (largely unnecessary on modern JDKs).
- `user.timezone=UTC` → consistent timestamps across regions and environments; convert to local time only at the edges.

**Verifying:** log `Runtime.getRuntime().maxMemory()` and `availableProcessors()` at startup, export `jvm.memory.*` and `jvm.gc.pause` through Micrometer, and load-test on the *actual* task size (not your laptop). **Probe:** "Pod keeps restarting with exit 137 but heap looks fine" → native/non-heap memory growth (thread leak, direct buffers) — use Native Memory Tracking (`-XX:NativeMemoryTracking=summary`).

---

### Q41. How do you implement graceful shutdown, and why does it matter on AWS?

**Why:** during a scale-in, deploy, or Spot interruption, a task gets `SIGTERM`. Without graceful shutdown, in-flight requests get dropped → 502s in your error budget.

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```
**The full chain (say all of it — this is a differentiator):**
1. ECS/ASG calls **deregister from the ALB target group first**; ALB stops sending new requests.
2. **Deregistration delay** (default 300s → set ~30s) lets in-flight requests finish.
3. `SIGTERM` → Spring stops accepting new requests, drains the Tomcat thread pool, closes `@PreDestroy` beans, closes HikariCP.
4. `stopTimeout` (ECS, default 30s) then `SIGKILL`.
5. **Ordering constraint:** `graceful timeout (25s) < stopTimeout (30s)`, and `deregistration delay ≥ longest request`.
6. Also drain **SQS consumers** (stop polling, finish in-flight) and **scheduled jobs** (`@Scheduled` tasks need `spring.task.scheduling.shutdown.await-termination=true`).
7. For **Spot**, subscribe to the 2-minute interruption notice / ECS Spot draining and shed load early.

**What "graceful" means (plain language):** abrupt shutdown = pulling the plug: a request that's 80% processed is cut off, the client sees an error, a half-written DB transaction rolls back, and a message being processed isn't acknowledged. Graceful shutdown = "stop taking new work, finish current work, then exit."

**Timeline example (deploy replaces a task, deregistration delay 30 s, graceful timeout 25 s, stopTimeout 30 s):**
| Time | What happens |
|---|---|
| t=0 | ECS marks task DRAINING, ALB stops routing **new** requests to it; existing connections continue |
| t=0–30 s | In-flight requests finish (deregistration delay) |
| t≈0+ | ECS sends `SIGTERM` to the container's PID 1 → Spring begins graceful shutdown: Tomcat refuses new connections, waits up to 25 s for active requests |
| t≈25 s | Remaining work cancelled, beans destroyed, Hikari pool closed, JVM exits |
| t=30 s | If the process is still alive, ECS sends `SIGKILL` |

**Common pitfalls (frequent root cause of "502s only during deploys"):**
- **PID 1 problem:** if the Dockerfile uses `CMD java -jar app.jar` through a shell script (`sh -c ...`), `SIGTERM` goes to the shell, not Java, and the JVM never shuts down gracefully (it just gets killed after the timeout). Use the exec form `ENTRYPOINT ["java","-jar","app.jar"]` or `exec java ...` in the script (and `init: true`/tini if needed).
- Race: ALB deregistration is eventually consistent — a request can still arrive just after SIGTERM. Some teams add a short `sleep` (preStop-style) of a few seconds before shutting down; on Kubernetes this is a `preStop` hook.
- Spring's `timeout-per-shutdown-phase` applies to each phase; long-running requests above that are interrupted, so keep requests short or make them async.
- Background threads (custom `ExecutorService`, Kafka/SQS listeners) are not drained automatically unless they're Spring-managed beans with proper `@PreDestroy`/`SmartLifecycle` stops.

**How to verify:** run a load test (e.g. k6/Gatling) while performing a deploy and assert zero 5xx; check ALB `HTTPCode_ELB_5XX_Count` during the rollout.

---

### Q42. Actuator health checks: what should the ALB health check point to, and why not `/actuator/health`?

**Say this first:** "`/actuator/health` aggregates *all* indicators including DB and Redis. If the DB blips, every task reports DOWN, the ALB kills every target, and a degraded dependency becomes a total outage. So the **liveness** probe must be shallow and the **readiness** probe is where dependencies belong."

```yaml
management:
  endpoint.health:
    probes.enabled: true
    group.readiness.include: db, redis, diskSpace
    group.liveness.include: livenessState
    show-details: when-authorized
```
- ALB / ECS container health check → **`/actuator/health/liveness`** (is the JVM alive?) or a readiness group you deliberately scoped.
- Deployment/traffic gating → **`/actuator/health/readiness`**.
- **Secure it:** expose Actuator on a separate `management.server.port` reachable only from the VPC SG, never through the public ALB listener. Never expose `/actuator/env`, `/heapdump`, or `/threaddump` publicly — that's a credential leak.

🧠 **"Liveness = should you restart me? Readiness = should you route to me?"**

**Plain-language background:** a *health check* is the load balancer/orchestrator repeatedly calling an HTTP endpoint and deciding "healthy" if it gets 200. Two different questions exist: (1) **"Is the process fundamentally broken (deadlock, stuck in a bad state)?"** — remedy: restart it (liveness). (2) **"Is it able to serve requests right now (started, caches warm, dependencies reachable)?"** — remedy: temporarily stop sending traffic, but *don't* restart (readiness). Spring Boot Actuator models these as separate groups (`/actuator/health/liveness` and `/readiness`).

**The cascading-failure story (why a deep health check is dangerous):** RDS fails over for 60 s. `/actuator/health` includes the `db` indicator → returns 503 on every task → ALB marks all targets unhealthy → ECS replaces them (tasks restart, cold JVMs) → when RDS returns, you have zero healthy tasks and an app that needs minutes to warm up. A brief dependency blip turned into a prolonged full outage. A shallow check keeps tasks in service; requests fail fast on the DB errors, circuit breakers open, and recovery is instant when the DB returns.

**Which one for what:**
- *ALB target-group health check:* shallow (liveness or a cheap `/ping`) — its job is "is this task process responding?". Failing it removes the target.
- *Readiness:* use for **startup gating** (don't route until Spring has finished initialising) and for deliberately taking an instance out of rotation (e.g. during shutdown Spring flips readiness to `REFUSING_TRAFFIC`).
- *Deep dependency checks:* belong in monitoring/alarms, not in the thing that kills tasks.

**Pitfalls:** set the ECS/ALB health-check **grace period** longer than JVM startup, or the checker marks a booting app unhealthy and loops restarts; an unauthenticated health endpoint must not leak details (`show-details: when-authorized` or `never`); when Spring Security is on, permit the health paths explicitly or the ALB gets 401/403 and fails the check.

---

### Q43. Which AWS SDK v2 practices do you follow in Spring Boot?

```java
@Bean
public S3Client s3Client() {
  return S3Client.builder()
      .region(Region.AP_SOUTH_1)
      .credentialsProvider(DefaultCredentialsProvider.create())   // roles, not keys
      .httpClientBuilder(ApacheHttpClient.builder()
          .maxConnections(100)
          .connectionTimeout(Duration.ofSeconds(2)))
      .overrideConfiguration(c -> c
          .apiCallTimeout(Duration.ofSeconds(10))                 // whole call incl. retries
          .apiCallAttemptTimeout(Duration.ofSeconds(3))           // per attempt
          .retryPolicy(RetryPolicy.builder()
              .numRetries(3)
              .backoffStrategy(BackoffStrategy.defaultStrategy()) // exponential + jitter
              .build()))
      .build();
}
```
**Points:**
- **Clients are thread-safe and expensive to create → singleton `@Bean`.** Creating one per request is a top-3 real-world mistake (it re-resolves credentials and rebuilds connection pools).
- **SDK v2 over v1** — v1 is end-of-support (Dec 2025). v2 gives non-blocking async clients (`S3AsyncClient` + CompletableFuture, good for WebFlux), builder pattern, pluggable HTTP.
- **Always set timeouts.** Defaults are generous; an untimed AWS call is a thread-pool exhaustion waiting to happen.
- **Retries with exponential backoff and jitter** are built in — but combine with a **circuit breaker (Resilience4j)** so you fail fast instead of retry-storming a struggling dependency.
- **Spring Cloud AWS 3.x** for auto-configured `@SqsListener`, S3, Secrets Manager, Parameter Store integration.

**Why each practice exists (plain language):**
- *Singleton client:* an SDK client owns an HTTP connection pool and credential/region configuration. Building one takes tens to hundreds of ms and allocates threads/sockets; building one per request leaks resources and defeats connection reuse. Create it once as a bean and inject it. Close it on shutdown (Spring does this for `AutoCloseable` beans).
- *Timeouts:* by default an SDK call can wait for a long time. If S3 is slow and each request thread waits 30 s, your 200 Tomcat threads are all blocked within seconds and the whole service stops answering — even for endpoints that don't use S3. `apiCallAttemptTimeout` caps a *single try*; `apiCallTimeout` caps the *whole call including retries*.
- *Retries + backoff + jitter:* transient errors (throttling `429/503`, network blips) usually succeed on retry. *Exponential backoff* waits longer each time (100 ms, 200 ms, 400 ms …); *jitter* adds randomness so thousands of clients don't retry at the same instant. SDK v2 already does this for retryable errors — **don't also wrap it in your own retry loop** or retries multiply (3 SDK × 3 app = 9 calls).
- *Credentials:* `DefaultCredentialsProvider` follows the chain described in Q5 — pass a profile only for local dev.
- *Region:* set explicitly or via `AWS_REGION`; a missing region is a common startup failure.

**Example — async client for non-blocking flows:** `S3AsyncClient` returns `CompletableFuture`; in a WebFlux app wrap it with `Mono.fromFuture(...)`. In a classic MVC app the sync client with a bounded Apache pool is simpler and fine.

**Pitfalls / probes:** (1) the default Apache HTTP client pool is small — under load you may see `Timeout waiting for connection from pool`; raise `maxConnections` and set `connectionAcquisitionTimeout`. (2) Don't forget to close streaming responses (`ResponseInputStream`) or you leak connections. (3) The **CRT-based** S3 client gives higher-throughput multipart transfers. (4) In tests use LocalStack/Testcontainers rather than mocking the entire SDK.

---

### Q44. HikariCP tuning for RDS — what values and why?

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10          # NOT 100
  minimum-idle: 10               # = max, avoids churn in a fixed-size container
  connection-timeout: 3000       # fail fast; don't queue threads for 30s
  validation-timeout: 2000
  idle-timeout: 600000
  max-lifetime: 1200000          # < RDS wait_timeout & < any LB/NAT idle timeout
  keepalive-time: 120000         # kills silently-dead sockets after failover
  leak-detection-threshold: 20000
```
**The arithmetic answer they want:** "Pool size isn't 'more is better.' A DB with 4 vCPUs saturates around 8–16 concurrent queries; beyond that you add queueing and context switching, not throughput. And it multiplies: `tasks × pool_size ≤ max_connections × 0.8`, leaving room for migrations, admin sessions, and replicas. 20 tasks × 10 = 200 connections — check that against your instance class before you scale."

**Also mention:** `max-lifetime` must be shorter than the NAT Gateway's 350s idle timeout and the DB's `wait_timeout`, or you'll see mysterious "connection reset by peer" errors. `keepalive-time` is what saves you after an RDS failover.

**What a connection pool is and why it exists:** opening a DB connection is expensive (TCP + TLS + authentication, tens of ms, plus server memory). **HikariCP** (Spring Boot's default pool) keeps a set of open connections; a request *borrows* one, runs its query, and *returns* it. If all are in use, the next request waits (up to `connection-timeout`) — that wait is why a too-small pool shows up as latency, and a too-large pool shows up as a loaded database.

**Why smaller is usually faster (Little's law intuition):** if each query takes 10 ms, one connection can serve ~100 queries/s. A pool of 10 can handle ~1000 queries/s — usually far more than a 4-vCPU DB can really execute in parallel. Extra connections just queue inside the DB and add context switching. A widely quoted starting formula (from the HikariCP wiki): `connections ≈ (core_count × 2) + effective_spindle_count`; then measure.

**Setting-by-setting:**
- `maximum-pool-size` — ceiling for parallel DB work *per app instance*. Multiply by number of instances.
- `minimum-idle = max` — fixed-size pool; no resizing surprises during spikes (Hikari's own recommendation).
- `connection-timeout` — how long a request waits for a free connection before an exception; keep it short so users get fast errors instead of hung threads.
- `max-lifetime` — retires each connection after N ms so none lives long enough to be silently dropped by a firewall/NAT/DB; Hikari staggers retirement slightly to avoid all expiring together.
- `keepalive-time` — periodically pings idle connections so dead ones (after failover/NAT timeout) are found and replaced before a user request hits them.
- `leak-detection-threshold` — logs a stack trace if a connection is held longer than N ms; it finds code that forgets to close connections or holds a transaction while calling a slow remote API.

**Pitfall:** holding a `@Transactional` open while calling an external HTTP service pins a connection for the whole remote call — a classic cause of "pool exhausted" under load even with healthy DB CPU. Check `hikaricp.connections.pending` and `hikaricp.connections.usage` in Micrometer.

---

### Q45. How do you make your Spring Boot service resilient to AWS dependency failures?

🧠 **"Timeout, Retry, Breaker, Bulkhead, Fallback"** (Resilience4j gives you all five):

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "cachedInventory")
@Retry(name = "inventory")
@Bulkhead(name = "inventory", type = SEMAPHORE)
@TimeLimiter(name = "inventory")
public CompletableFuture<Inventory> get(String sku) { ... }
```
- **Timeouts everywhere** — HTTP client, JDBC, SDK, and shorter than the caller's timeout (**timeout budget decreases downstream**).
- **Retry only idempotent operations**, with exponential backoff + **jitter** (without jitter, all your tasks retry in lockstep and DDoS your own dependency).
- **Circuit breaker** to fail fast and let the dependency recover.
- **Bulkhead** so one slow dependency can't consume every Tomcat thread.
- **Graceful degradation** — serve stale cache, a partial response, or a queued "we'll process it" instead of a 500.
- **Idempotency keys** on write APIs so client retries are safe.
- **Test it**: AWS Fault Injection Simulator, chaos days, forced RDS failover in staging (`reboot with failover` is one click — do it during business hours in staging *before* it happens at 3 AM in prod).

**Why these patterns exist (explained simply):** in a distributed system the dependency (DB, another service, an AWS API) *will* be slow or down at some point. Without protection, a slow dependency makes your threads wait; waiting threads pile up; the service runs out of threads and fails for everyone — a **cascading failure**.
- **Timeout** — give up after a bounded time so threads are freed.
- **Retry** — try again for *transient* failures (brief network error). Danger: retrying a non-idempotent operation (e.g. "charge card") can do it twice; retrying a dependency that is overloaded makes it worse — a **retry storm**.
- **Circuit breaker** — like an electrical breaker: after N failures in a window it *opens* and rejects calls instantly (no waiting) for a cool-down period, then lets a few trial calls through (*half-open*); if they succeed it *closes*. It protects both you and the struggling dependency.
- **Bulkhead** — named after ship compartments: limit the number of concurrent calls to one dependency (e.g. max 20 of your 200 threads), so its slowness can't sink the whole ship.
- **Fallback** — a degraded-but-useful answer: cached price list, "recommendations unavailable", or accepting the order and processing it later from a queue.

**Example config (Resilience4j via Spring Boot):**
```yaml
resilience4j:
  circuitbreaker.instances.inventory:
    slidingWindowSize: 20
    failureRateThreshold: 50        # open when ≥50% of last 20 calls fail
    waitDurationInOpenState: 30s
  retry.instances.inventory:
    maxAttempts: 3
    waitDuration: 200ms
    enableExponentialBackoff: true
    exponentialBackoffMultiplier: 2
```
**Ordering matters:** typically Retry wraps CircuitBreaker (or the reverse, but be deliberate) and TimeLimiter is innermost; ensure total retry time still fits within the caller's timeout budget.

**AWS-specific examples:** SQS send failing → local outbox/retry (see Q33); DynamoDB throttling (`ProvisionedThroughputExceededException`) → SDK backoff + on-demand capacity; RDS failover → connection retries with Hikari `keepalive`/AWS JDBC wrapper; S3 slowness → timeout + serve placeholder/cached file; **Multi-AZ** and multiple instances give redundancy, but only these patterns make the *application* tolerate the blips.

---
## 12. Cost Optimization

### Q46. Your AWS bill doubled. Walk me through what you do.

**Say this first:** "Find it, attribute it, fix it, prevent it."

**1. Find it** — Cost Explorer grouped by **service**, then by **usage type**, comparing month over month. **Cost Anomaly Detection** usually already flagged it. For deep dives, **CUR in Athena**.

**2. Attribute it** — this only works if you tagged: `Environment`, `Team`, `Service`, `CostCenter`. Enforce with **tag policies + SCPs**, and activate them as **cost allocation tags**. If nothing is tagged, that's finding #1.

**3. The usual suspects** 🧠 **"CDN-SIRT"**:
- **C**ompute idle — dev/staging running 24/7 (fix: scheduled scale-to-zero, ~65% saving on non-prod), oversized instances (right-size from CloudWatch p99, not p100).
- **D**ata transfer — cross-AZ chatter (keep the call path AZ-local, use `ap-south-1a` app → `ap-south-1a` replica), NAT Gateway data processing ($0.045/GB — add **S3/DynamoDB gateway endpoints**, they're free), internet egress (put CloudFront in front — CDN egress is cheaper than EC2 egress).
- **N**AT/Load balancers — one NAT per AZ is right for HA but check you don't have six in dev.
- **S**torage — unattached EBS volumes, old snapshots, S3 without lifecycle rules, **CloudWatch Logs with infinite retention** (a genuinely huge and invisible line item), incomplete multipart uploads.
- **I**dle resources — Elastic IPs not attached, old ALBs, forgotten OpenSearch/Redshift clusters, orphaned ECS services.
- **R**ate optimization — **Savings Plans** (Compute SP = flexible across EC2/Fargate/Lambda, 1 or 3 yr, up to 66%), Reserved Instances for RDS/ElastiCache, **Spot for stateless/batch (up to 90%)**, **Graviton (ARM) for ~20% better price-performance — and Java runs on ARM with a rebuild, that's basically a free win for a Spring Boot shop**.
- **T**iering — S3 Intelligent-Tiering, Glacier, gp2→gp3, log sampling.

**4. Prevent it** — AWS Budgets with alerts at 80/100/120%, per-team budgets, `infracost` in the PR pipeline so cost shows up in code review, monthly FinOps review, and non-prod auto-shutdown by default.

**A concrete story is worth more than the list.** Have one: *"We were paying ~$4k/month in NAT data processing because every ECS task pulled images and wrote logs through NAT. Two gateway endpoints and interface endpoints for ECR/logs took it to ~$400."*

**Jargon decoded:** **Cost Explorer** = console graphs of spend, filterable by service/tag/account. **Cost Anomaly Detection** = ML alerts for unusual spend. **CUR** (Cost and Usage Report) = the line-item billing export to S3 that you can query with Athena SQL. **Cost allocation tags** = tags (`Team=orders`) that appear on the bill so you can split costs per owner — they must be *activated* in Billing and only apply from activation onward. **Savings Plan** = commit to $X/hour of compute for 1–3 years in exchange for a discount; **Spot** = spare capacity at deep discount that AWS can reclaim with a 2-minute warning.

**A repeatable investigation, with an example:**
1. Cost Explorer → group by *Service* → "EC2-Other" jumped. 2. Regroup by *Usage Type* → `NatGateway-Bytes` is the driver. 3. Group by *Linked account/tag* → it's the `staging` account. 4. VPC Flow Logs/CloudWatch → a test job is downloading 5 TB/day from the internet through NAT. 5. Fix: stop the job, add S3 gateway endpoint, cache the download. 6. Prevent: Budget alert + anomaly alert + tag enforcement.

**Pitfall:** don't start by randomly shrinking instances — first find *what changed* (new deployment, traffic growth, log level change, a forgotten test environment). A doubled bill is almost always one or two line items, not "everything."

---

### Q47. What are the biggest cost wins specific to a Java/Spring Boot workload?

1. **Graviton** — rebuild the image for `linux/arm64`, ~20% cheaper, often *faster* for JVM workloads. Multi-arch buildx makes it a one-line change.
2. **Fargate Spot** for async/queue workers and CI — 70% off, and SQS retries make interruption harmless.
3. **Right-size memory** — Java devs over-provision from habit. Measure real RSS, then set the task size; `MaxRAMPercentage` makes shrinking safe.
4. **Scale on the right metric** — scaling on CPU for an I/O-bound service keeps too many tasks warm.
5. **Cache aggressively** — an ElastiCache node is far cheaper than the bigger RDS instance you'd need without it.
6. **Kill custom-metric cardinality** — Micrometer tags with userId can cost more than the compute.
7. **Log volume** — DEBUG in prod is expensive; sample, set retention, ship cold logs to S3 instead of keeping 2 years in CloudWatch.

**Why each one saves money (and how to do it):**
1. *Graviton:* AWS's ARM CPUs deliver more work per dollar. Java is bytecode, so your code doesn't change; only native dependencies (e.g. some JNI libs) and the base image need ARM builds. Fargate: set `runtimePlatform.cpuArchitecture = ARM64`. Build with `docker buildx build --platform linux/amd64,linux/arm64`. Validate with a load test before switching prod.
2. *Spot:* fine for **idempotent, retryable** workers. Pair with graceful shutdown (Q41) so a 2-minute interruption notice results in finishing or re-queuing work, and keep a small On-Demand base via a capacity-provider strategy (e.g. base=2 on-demand, rest Spot).
3. *Right-size:* look at `MemoryUtilization`/`CPUUtilization` over two weeks at p95. If a service uses 700 MB of a 4 GB task, drop to 2 GB; the cost scales linearly with size × task count.
4. *Scaling metric:* an I/O-bound service has low CPU even when busy, so CPU-based scaling never scales out on time (users suffer) or keeps a large min-count (you overpay). Scale on request count per target or queue depth instead.
5. *Caching:* every cache hit is a DB query you don't pay for; DB instance cost typically grows steeply with size.
6. *Metric cardinality:* each unique tag combination is a separate billable CloudWatch metric.
7. *Logs:* ingestion is billed per GB (~$0.50/GB in many Regions) before storage; one noisy `log.debug` in a hot loop can cost more than the service's compute.

**JVM-specific extra:** startup time × tasks × deploy frequency matters for cost with Lambda/Fargate Spot; faster startup (AppCDS, AOT/native) lets you scale to zero or use smaller reserved capacity.

---

## 13. Scenario-Based Questions

> Answer every scenario in this shape: **clarify → diagnose → immediate mitigation → root-cause fix → prevention.**
> Saying "first I'd check X, and if it's not that, then Y" is what makes you sound senior.

---

### S1. Your Spring Boot API on ECS starts returning intermittent 502s after a deploy. Nothing in the app logs. Diagnose.

**Clarify:** all requests or a percentage? Correlated with deploys or constant? What does the ALB access log show?

**Diagnosis order:**
1. **ALB access logs** → `target_status_code` empty + `elb_status_code 502` means the target closed the connection or sent a malformed response. Look at `target_processing_time`.
2. **Keep-alive race — the #1 cause.** ALB idle timeout (60s) ≥ Tomcat `keepAliveTimeout` (default 60s) means Tomcat can close a pooled connection exactly as ALB reuses it → 502 with nothing in the app log. **Fix: `server.tomcat.keep-alive-timeout=65000` and `server.tomcat.max-keep-alive-requests=-1`, i.e. backend timeout strictly greater than ALB's.**
3. **Container OOM-kill** → check ECS `stoppedReason` for `OutOfMemoryError: Container killed due to memory usage` and exit code 137. Fix `MaxRAMPercentage`/task memory.
4. **Health check grace too short** → tasks get registered before the JVM finishes booting, fail, and cycle. Fix `healthCheckGracePeriodSeconds: 120` and a container `startPeriod`.
5. **Deregistration delay too short** → in-flight requests killed during the rolling deploy. Fix delay ≥ longest request + graceful shutdown config.
6. **Response header size** or a huge header → ALB rejects.

**Prevention:** blue/green with canary + a 5xx alarm that auto-rolls back, so the next one never reaches users.

**How to actually do the diagnosis (concrete commands/evidence):**
- *Why "nothing in app logs" is a clue:* a 502 is produced by the **ALB**, so if the request never reached Spring (connection reset, task killed, not ready) your app can't log it. Look at the layers *outside* the app: ALB access logs (S3), ECS service events, and CloudWatch Container Insights.
- Read an ALB log line: `elb_status_code=502 target_status_code=-` means ALB got no valid HTTP response from the target. If `target_status_code=502/500` came *from* the app, it's an app-level issue instead.
- ECS: `aws ecs describe-tasks` → `stoppedReason` ("Essential container exited", "OutOfMemoryError: Container killed due to memory usage", "Task failed ELB health checks") and `exitCode`. Service **events** show lines like "has begun draining connections" and "deregistered targets".
- Correlate timestamps of the 502s with the deploy timeline: 502 spikes **only during rollout** → shutdown/deregistration/health-check configuration (see Q41/Q42); 502s **continuously at a low rate** → keep-alive race or a flaky task.
- *Quick test of the keep-alive theory:* temporarily raise Tomcat's timeout above 60 s and watch whether the intermittent 502 count drops to zero.

**Why the order matters:** start with the cheapest-to-check and most common (keep-alive, deploy settings) before deeper theories; state each as a hypothesis with the evidence that would confirm it — that is what "diagnose" means to an interviewer.

🧠 **"502 = talk to the target. 503 = no targets. 504 = slow target."**

---

### S2. Traffic is 10x on Diwali/Black Friday for 6 hours. How do you prepare?

1. **Load test first** — Gatling/k6 against a prod-like environment. You cannot plan capacity you haven't measured. Find the bottleneck *before* the event; it's almost always the database.
2. **Compute** — raise ASG/ECS max, **scheduled scaling** to pre-warm (target-tracking reacts too slowly for a 9 AM cliff), plus predictive scaling. Pre-warm the JVM fleet.
3. **Database is the real limit** — it doesn't scale in seconds. Scale up ahead of time, add read replicas and route reads (`@Transactional(readOnly=true)` + a routing DataSource or the Aurora reader endpoint), add **RDS Proxy**, and check `max_connections` against `tasks × pool_size`.
4. **Cache hard** — pre-warm Redis with catalog/pricing, longer TTLs, CloudFront for all static + cacheable API GETs.
5. **Shed and defer** — make the checkout write path async via SQS (accept the order, process it behind the queue), rate-limit non-critical endpoints, disable expensive features (recommendations, analytics) behind a feature flag.
6. **Quotas** — raise service limits *weeks* ahead (they need approval): Lambda concurrency, ENIs per region, ECS tasks, Elastic IPs, ALB rules. Also **pre-warm/notify for extreme ALB spikes**.
7. **Ops readiness** — war room, dashboard per team, runbooks, a freeze on deploys, an announced rollback path, and a "graceful degradation" list agreed with the business.

**Close with:** "And I'd write the post-event report before the event — what we'd measure, so next year the plan is data, not memory."

**Why each step (and an example) in plain terms:**
- *Why load test:* "10x" is a guess until measured. Example: at 1x you serve 300 req/s using 6 tasks with the DB at 25% CPU. A test at 10x (3000 req/s) shows tasks scale fine but the DB hits 100% at 6x and p99 explodes. You've learned the real bottleneck and the number to fix.
- *Why pre-scale, not just auto-scale:* auto-scaling reacts *after* load rises, and a new Spring Boot task needs 1–2 minutes (launch + JVM boot + health check grace). A sale starting at 9:00:00 can overload you before new tasks are ready. So use **scheduled scaling** to set min capacity at 8:45 (e.g. min 6 → 40) and let target-tracking handle fine-tuning.
- *Why the DB is the limit:* app tasks are stateless and clone easily; a single writer DB cannot be cloned. Hence caching, read replicas, connection limits, and async writes.
- *Async checkout example:* instead of the API synchronously doing 6 DB writes + payment + email, it validates, stores an `order(status=PENDING)`, enqueues to SQS, returns `202 Accepted`; workers complete the rest at a sustainable rate. The queue absorbs the 10x spike (back-pressure).
- *Load shedding / feature flags:* decide in advance which features to turn off (recommendations, analytics events) and who has authority; flip via AppConfig, not a deploy.
- *Quotas:* account limits (vCPU, Elastic IPs, ENIs, Lambda concurrency) can block scale-out; increases can take days, so request early. For very sharp spikes AWS can pre-provision ELB capacity — contact support in advance.

**Pitfalls:** a cold cache at the start of the event (do a warm-up); a downstream third-party (payment gateway, SMS) with *its own* limits — confirm their capacity too; a deploy freeze needs an emergency-fix path.

---

### S3. RDS CPU is pinned at 100% in production right now. What do you do?

**Immediate (stop the bleeding):**
1. **Performance Insights** → top SQL by load, top waits. This finds the culprit query in about 60 seconds.
2. Check for a **runaway query / missing index / accidental full scan** from a new deploy. If it correlates with a deploy → **roll back first, investigate after.**
3. Kill the offending session (`pg_terminate_backend` / `KILL`).
4. If it's connection storms, cap it: reduce pool size or Lambda reserved concurrency.

**Short-term:** scale the instance up (Multi-AZ makes this a failover-length blip), add a read replica and shift read traffic, enable/warm the cache.

**Root cause:** `EXPLAIN ANALYZE` the query; add the index; fix the **N+1** (JPA lazy loading in a loop is the most common Spring Boot cause — fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`); add pagination to that endpoint someone wrote without a `LIMIT`; check for missing `readOnly=true`; look for a cache that stopped working (a Redis outage silently turning into a DB stampede).

**Prevention:** slow-query logging + alarms, `p6spy`/Hibernate statistics in CI to catch N+1 before merge, load tests on the read paths, Performance Insights retention on, and a query-review step for new endpoints.

**Terms and how to read the evidence:**
- **Performance Insights (PI)** = RDS's built-in dashboard that shows *DB load* (average active sessions) split by SQL statement, wait event, user, and host. If load is far above the vCPU count, sessions are queued. Top SQL by load tells you which query to look at.
- **Wait events** hint at the problem type: `CPU` = compute-heavy queries (missing index, big sort); `IO:DataFileRead` = scanning disk/not in buffer cache; `Lock:*` = contention between transactions.
- **N+1 problem (very common in JPA):** loading 100 orders in one query, then each `order.getItems()` triggers 100 more queries = 101 queries. Fix: `@EntityGraph`/`JOIN FETCH` loads items in one join query, or `@BatchSize(size=50)` batches the lazy loads. Example: `@Query("select o from Order o join fetch o.items where o.customerId = :id")`.
- **Missing index vs full scan:** `EXPLAIN ANALYZE` shows `Seq Scan` on a big table when it should be `Index Scan`. Adding the right (often composite) index can drop a query from seconds to milliseconds; check the write overhead of the new index.
- **Connection storm:** after a restart, hundreds of tasks all open pools at once and run the same expensive warm-up queries → CPU spike. Stagger startups, reduce pool sizes, or add RDS Proxy.

**Decision hint — rollback vs scale:** if CPU spiked right after a release, rollback is the fastest and safest mitigation. If load is organic (traffic growth), scale up/add replicas, then optimise queries. Don't scale blindly: a bad query scales badly, and a larger instance just delays the problem.

**Caution while killing sessions:** terminate only the offending long-running queries (identify by PID in `pg_stat_activity` / `SHOW PROCESSLIST`); killing everything causes a thundering herd of reconnects.

🧠 **Order: "Observe (PI) → Attribute (which query) → Mitigate (rollback/scale) → Fix (index/N+1) → Prevent (alarm/test)."**

---

### S4. A message keeps failing and blocks your SQS consumer. Design the fix.

- **Standard queue:** a poison message doesn't block others, but it does burn retries and pollute logs. **FIFO with the same MessageGroupId genuinely blocks the group** — that's the harder case and worth calling out.
- **Fix:** `maxReceiveCount=3` → DLQ. Alarm on DLQ depth. Investigate, fix, then **redrive**.
- **Make the handler classify failures:** *transient* (downstream 503, timeout) → let it retry; *permanent* (validation error, bad JSON, unknown enum) → **don't retry, send straight to the DLQ / a `parking-lot` table with the error reason**. Blindly retrying a poison message 5 times is wasted capacity.
- **Idempotency** so retries are safe.
- **Observability:** log the `MessageId` + `ApproximateReceiveCount` + correlation ID on every failure so the DLQ investigation takes minutes, not hours.

**Why it "blocks" and how the retry loop works:** when a handler throws, the message isn't deleted; after the visibility timeout it reappears and is retried, again and again. With Standard SQS, other messages still flow (just with wasted retries). With FIFO, SQS won't hand out later messages of the *same group* until the failing one is deleted or moved to the DLQ, so an entire customer's/account's stream stalls behind it — that's the true "blocked consumer".

**Handler sketch (classify, then act):**
```java
@SqsListener("orders-queue")
public void handle(OrderEvent e) {
    try {
        service.process(e);
    } catch (ValidationException | JsonProcessingException permanent) {
        parkingLot.save(e, permanent);        // permanent: don't retry; record reason for support
        // returning normally acknowledges (deletes) the message
    }                                          // transient exceptions propagate → message retried → DLQ after maxReceiveCount
}
```
**DLQ operations runbook:** alarm on `ApproximateNumberOfMessagesVisible > 0` for the DLQ → inspect a sample message (in console/`aws sqs receive-message`) → fix the bug or data → **start DLQ redrive** (console/API) to send messages back to the source queue → confirm the DLQ is empty. Set the DLQ retention (14 days) *longer* than the source queue's so messages don't expire before you investigate.

**Tuning knobs:** `maxReceiveCount` 3–5 for transient-friendly workloads; exponential backoff between retries by extending visibility timeout per attempt (`ChangeMessageVisibility` based on receive count); partial-batch failure reporting (Lambda `ReportBatchItemFailures`) so one bad message in a batch of 10 doesn't cause the other 9 to be reprocessed.

---

### S5. Migrate a 10-year-old monolithic Spring MVC app on-prem to AWS. How?

**Say this first:** "I don't rewrite it. I move it, then improve it — otherwise I'm doing a migration and a rewrite at the same time and both fail."

**Phase 0 — Assess.** Application Discovery Service, dependency map, data volume, licensing, compliance, RTO/RPO, and the **6 R's** for each component 🧠: **Rehost, Replatform, Refactor, Repurchase, Retire, Retain**.

**Phase 1 — Rehost/Replatform (weeks).** Containerize the WAR (or lift EC2 with Application Migration Service). RDS via **DMS + Schema Conversion Tool** with **CDC for near-zero-downtime cutover**. Sessions are the classic blocker → externalize to Spring Session + ElastiCache so the app becomes horizontally scalable. Files on local disk → S3. Cron on the box → EventBridge Scheduler + ECS task. Hard-coded IPs/hostnames → Parameter Store + Route 53. Local logs → CloudWatch.

**Phase 2 — Cutover.** DMS full load + CDC, run both in parallel, validate with row counts and checksums, switch DNS with a low TTL, keep on-prem as the rollback for a week. Do it in a low-traffic window with a documented, rehearsed rollback.

**Phase 3 — Refactor selectively.** **Strangler fig**: put the ALB (or API Gateway) in front, route one path at a time to a new Spring Boot service, extract the highest-change or highest-load bounded context first. Use the **anti-corruption layer** pattern; share the DB temporarily, then split with an outbox/CDC. Never big-bang.

**What I'd insist on:** don't migrate technical debt silently, keep the monolith deployable throughout, and pick a first slice that's valuable but low-risk to build confidence.

**Terms explained:**
- **6 R's** = migration strategies per component. *Rehost* = lift-and-shift as-is onto EC2. *Replatform* = small tweaks for managed services (own MySQL → RDS; Tomcat on a VM → container). *Refactor* = re-architect to cloud-native. *Repurchase* = replace with SaaS. *Retire* = switch off what nobody uses. *Retain* = leave on-prem for now.
- **DMS (Database Migration Service) + CDC (Change Data Capture):** DMS does a **full load** of existing data, then continuously replays new changes from the source DB's transaction log, keeping the AWS copy within seconds of on-prem. At cutover you stop writes briefly, let the last changes drain, and switch — minutes of downtime instead of hours of dump/restore. **SCT** converts schema/code if the engine changes (Oracle → PostgreSQL).
- **Strangler fig:** like the vine that gradually grows around a tree, new services take over one URL path at a time while the old monolith keeps serving the rest; eventually the monolith shrinks to nothing. The ALB path rule `/api/invoices/*` → new service, `/*` → monolith.
- **Anti-corruption layer:** a translation layer so the new service's clean domain model isn't polluted by the monolith's legacy data model.

**Why sessions/files/cron are "classic blockers":** a 10-year-old Spring MVC app often stores `HttpSession` data in memory, writes uploads to local disk, and runs `@Scheduled` jobs assuming exactly one server. Once you run 3 instances: users get logged out when the LB sends them to a different node, uploaded files exist on only one node, and every node runs the cron job (duplicates). The fixes are Spring Session (Redis), S3, and a single scheduled trigger (EventBridge Scheduler or ShedLock) respectively.

**Example timeline:** weeks 1–2 assess & set up landing zone/VPC/CI; weeks 3–6 containerize + stage environment + DMS replication; weekend cutover with rehearsed rollback; months afterwards extract the first bounded context (e.g. notifications) via strangler.

---

### S6. Design a file-upload feature: users upload 500 MB videos, you generate thumbnails and notify them.

```
Client → GET /uploads/presign  (Spring Boot: validate user, quota, content-type)
       ← presigned S3 POST/PUT URL (10 min TTL) + object key
Client → PUT bytes directly to S3   (multipart, resumable; never through my app)
S3 ObjectCreated:Put → EventBridge → SQS (buffer + retry) → ECS/Lambda worker
Worker: ffmpeg → thumbnails + transcode → write to S3 (or use MediaConvert)
       → update DynamoDB/RDS status = READY
       → SNS/SES/WebSocket notify user
CloudFront + OAC serves the output; signed URLs if content is private.
```
**Why each choice (state these — this is the whole point of the question):**
- **Pre-signed URL:** no 500 MB through my ECS task → no heap, no timeout, no bandwidth cost, no ALB limit.
- **S3 event → SQS (not straight to Lambda):** the queue gives buffering, retry, DLQ, and back-pressure so a batch upload of 10,000 files doesn't melt the workers or a downstream API.
- **Lambda vs ECS:** ffmpeg on a 500 MB video may exceed 15 min / 10 GB → ECS Fargate task (or MediaConvert, which is purpose-built). Lambda for small images.
- **Status table:** the client polls or gets a WebSocket push; never a blocking synchronous HTTP request.
- **Guardrails:** content-length and content-type conditions *in the presigned policy* (a presigned PUT can otherwise upload anything, any size), virus scan (GuardDuty Malware Protection for S3), lifecycle rule to abort incomplete multipart uploads, and S3 key namespacing per tenant (`tenant/{id}/{uuid}`) so you can enforce isolation in IAM.

---

### S7. Two microservices need to share data. One owns Orders, the other Inventory. How do they communicate?

**Ask first:** "Does the caller need an answer *now*, and can it proceed if the other service is down?"

- **Synchronous (REST/gRPC via ALB or Cloud Map service discovery)** when the answer is needed in the request path — e.g. "is this in stock?" at checkout. Costs: temporal coupling, cascading failure. Requires timeouts + circuit breaker + fallback.
- **Asynchronous (SNS→SQS / EventBridge)** when eventual consistency is acceptable — `OrderPlaced` → Inventory reserves stock. Better availability, independent scaling, but you must handle eventual consistency in the UI and design compensating actions.
- **Never share a database.** That's a distributed monolith — you can't deploy or scale independently, and one team's schema change breaks the other.

**For a distributed transaction:** **Saga** pattern, not 2PC. Orchestration (Step Functions — visible state machine, easier to debug) vs choreography (events, looser coupling, harder to trace). Each step needs a **compensating transaction** (`CancelReservation`). Add the **outbox** pattern so the DB write and the event publish can't diverge.

🧠 **"Sync for queries, async for commands/notifications, saga for multi-service transactions."**

**Why not just share the database (the problem):** Orders could simply `SELECT` from the Inventory tables — but then Inventory can never rename a column, change its datastore, or scale on its own without breaking Orders. The schema becomes a hidden public API. Each service should own its data and expose it only through an API or events.

**Worked example — placing an order:**
1. *Sync query:* Orders calls `GET /inventory/{sku}/availability` (timeout 500 ms, circuit breaker, fallback "treat as available, verify later") to give instant feedback.
2. *Async command/event:* Orders saves `Order(PENDING)` + outbox row in one transaction → publishes `OrderPlaced` → Inventory consumes it, reserves stock, publishes `StockReserved` or `StockRejected`.
3. *Saga completion:* Orders listens for those events; on `StockReserved` → `CONFIRMED`; on `StockRejected` → `CANCELLED` and a refund/notification. That "undo" step is the **compensating transaction** — there's no cross-service rollback, so every step needs a defined way to semantically reverse it.

**Terms:** *2PC (two-phase commit)* = a coordinator makes all databases commit or abort together; it needs locks held across services, is slow, and many cloud datastores (DynamoDB, SQS) don't support it. *Saga* = a sequence of local transactions, each publishing an event/command for the next. *Orchestration* = a central coordinator (e.g. Step Functions) tells each service what to do; *choreography* = services react to each other's events with no central brain. *Eventual consistency* = data across services converges shortly after, not instantly — the UI shows "processing" states.

**Pitfalls:** duplicate events (make handlers idempotent), out-of-order events (version numbers/FIFO groups), and a saga stuck midway (timeouts + alarms on "orders PENDING > 15 min").

---

### S8. Your ECS task keeps restarting. There are no application errors. Debug.

Checklist, in order:
1. **`aws ecs describe-tasks`** → `stoppedReason` and `containers[].exitCode`. This is the single most useful command.
   - **137** = SIGKILL, almost always **OOM** (container memory or host memory pressure) → check `stoppedReason` for "OutOfMemory", raise task memory or lower `MaxRAMPercentage`.
   - **143** = SIGTERM, normal (deploy/scale-in) — so is it actually failing, or just being replaced?
   - **1** = app exited (unhandled exception at startup — a missing env var, unreachable DB, bad Parameter Store path).
   - **255 / 139** = segfault or native issue.
2. **Health check failing** → task killed as unhealthy before the JVM finishes booting. Fix `startPeriod` / grace period. A Spring Boot app needing 45s to start with a 30s grace period is a restart loop.
3. **CloudWatch Logs** for the *previous* task, not the current one — and check the log stream actually exists (if not, the **execution role** can't write logs, or the awslogs group doesn't exist).
4. **Image pull failure** → `CannotPullContainerError`: execution role missing `ecr:GetAuthorizationToken`, or no ECR endpoint/NAT route from a private subnet.
5. **Resource exhaustion** → no ENIs available in the subnet (`awsvpc` needs one IP per task), or the EC2 capacity provider is full.
6. **Secrets injection failure** → task role/execution role can't read the Secrets Manager ARN, or the KMS key policy doesn't allow it. Task dies before logging anything.

**Say this:** "The trick is that a Spring Boot container failing at *startup* often produces no app logs at all, so exit code + `stoppedReason` come before log-reading."

---

### S9. Design a rate limiter for your public API on AWS.

**Layers, outermost first (name them all):**
1. **AWS WAF rate-based rules** — per-IP, at the edge, blocks before it costs you compute. Also bot control and IP reputation lists.
2. **API Gateway usage plans + API keys** — per-client throttling and quotas (e.g. 100 rps, 1M/month). If the requirement is *per-customer* limits, this is the cheapest correct answer.
3. **Application-level (Bucket4j + ElastiCache Redis)** — when the limit is per-tenant, per-endpoint, or business-tier-aware ("free tier: 100/day"). Must be **distributed**: an in-memory limiter is wrong the moment you have 2 tasks.
```java
// token bucket in Redis, shared across all tasks
Bucket bucket = proxyManager.builder().build(tenantId,
    () -> BucketConfiguration.builder()
        .addLimit(l -> l.capacity(1000).refillIntervally(1000, Duration.ofMinutes(1)))
        .build());
if (!bucket.tryConsume(1)) return status(429).header("Retry-After","60").build();
```
4. **Back-pressure below that** — bulkheads, connection pool caps, SQS to absorb bursts.

**Details that show maturity:** return **429 with `Retry-After` and `X-RateLimit-*` headers**; distinguish throttling from failure in metrics; fail **open** if Redis is down (don't take the API down to enforce a limit) but alarm on it; use a **token bucket** (allows bursts) over a fixed window (thundering herd at the boundary); and consider a **sliding window** for fairness.

**Token bucket in plain words:** imagine each tenant has a bucket holding up to 1000 tokens, refilled with 1000 tokens per minute. Every request removes one token; if the bucket is empty the request gets **429 Too Many Requests**. Because the bucket can be full, short bursts are allowed, but the long-run average can't exceed the refill rate. *Fixed-window problem:* "100 requests per minute" resets on the minute, so a client can send 100 at 12:00:59 and 100 more at 12:01:00 = 200 in two seconds. Token bucket/sliding window avoid that.

**Why the layers:** each layer is cheaper to run the further out it is. WAF stops obvious abuse (one IP sending 10,000 req/5 min) before it reaches compute; API Gateway enforces *contractual* per-customer limits; the app handles *business* rules; and back-pressure protects the databases when everything else is exceeded. Identify callers by API key / JWT `sub` / tenant ID (not just IP, since many users share corporate NAT IPs and attackers rotate IPs).

**Redis detail:** the check-and-decrement must be **atomic** (Lua script or Bucket4j's Redis integration) or two tasks can both read "1 token left" and both proceed. Also set a TTL on bucket keys so inactive tenants don't use memory forever, and add a short timeout to the Redis call so a slow Redis doesn't add latency to every request.

---

### S10. Your Lambda's cold starts are causing 8-second p99 on a customer-facing API. Fix it.

1. **First, question the choice.** A latency-sensitive, steady-traffic API is not a great Lambda fit. If it's steady traffic → **move it to ECS Fargate**; that's the honest answer and interviewers respect it.
2. If it must stay Lambda:
   - **SnapStart** (Java) → ~200–500ms. Biggest single win, free. Watch out for: cached connections restored stale (use CRaC `beforeCheckpoint`/`afterRestore` hooks), unique IDs/random seeds captured in the snapshot, and pre-warmed credentials expiring.
   - **Provisioned concurrency** on the hot alias + application auto-scaling on it, if the traffic shape justifies the cost.
   - **Trim the init path:** cut the dependency graph (a Spring Boot fat JAR with 60 starters is the problem), lazy-init beans, `spring.main.lazy-initialization=true`, exclude auto-configurations, or use **Spring Cloud Function** instead of the servlet stack.
   - **GraalVM native image** → ~50ms, at the cost of build complexity and reflection config.
   - **Right-size memory upward** — Lambda CPU scales with memory; 1769 MB ≈ 1 full vCPU. Java functions are often *cheaper* at higher memory because they finish faster. Use **Lambda Power Tuning** to find the optimum.
   - Move the function **out of the VPC** if it doesn't need VPC resources (VPC ENI attachment used to add seconds; Hyperplane fixed most of it, but no-VPC is still simpler).
3. **Measure properly** — separate `InitDuration` from `Duration` in CloudWatch. Fix what the data says, not what you assume.

**Why the cold start is 8 s here (explained):** the Lambda service must start a new sandbox, load a large JAR, boot the JVM, then Spring scans classpath, builds beans, runs auto-configurations and may open DB connections — all before the first line of your handler runs. Memory size sets CPU share, so an under-provisioned function (512 MB ≈ a fraction of a vCPU) boots much slower. A p99 (99th percentile) of 8 s means that ~1 in 100 requests hits a cold start — typical when traffic is bursty or low enough that sandboxes are recycled.

**What each fix does, in one line:** *SnapStart* — takes a memory snapshot after initialisation (at version publish) and resumes from it, skipping the JVM+Spring boot. *Provisioned concurrency* — keeps N initialised sandboxes always warm (you pay for them). *Lazy init / fewer starters* — less work at boot. *Native image* — ahead-of-time compiled binary that starts in tens of ms. *More memory* — more CPU, so init finishes sooner.

**Example plan:** (1) check `INIT_DURATION` in the REPORT log lines to confirm cold starts are the problem → (2) enable SnapStart on a published version and point the alias to it → (3) raise memory from 512 MB to 1769 MB and compare cost/latency with Power Tuning → (4) add provisioned concurrency of 5 during business hours via scheduled scaling → (5) if still not meeting the SLO, move the API to Fargate.

**Pitfall:** SnapStart applies only to *published versions/aliases*, not `$LATEST`; and tracing/metrics for restore (`RestoreDuration`) are separate from `InitDuration`.

---

### S11. How do you achieve zero-downtime deployment including a database schema change?

**Say this first:** "The app deploys with blue/green in minutes. The *schema* is the hard part, and the answer is **expand/contract** across multiple releases — never a breaking migration in the same release as the code that needs it."

**Expand → Migrate → Contract:**
- **Release 1 (expand):** add the new nullable column / new table. Old code ignores it. Backward compatible, safely rollback-able.
- **Release 2 (dual-write):** new code writes both old and new, reads old. Backfill existing rows with a batched job (not one giant `UPDATE` that locks the table).
- **Release 3 (switch reads):** read from new, still writing both.
- **Release 4 (contract):** stop writing old, drop the old column.

**Rules to state:** never rename or drop in the same release as the code change; no long-lived locks (`ALTER TABLE` on a large MySQL table can block — use `pt-online-schema-change`/`gh-ost` or Aurora's fast DDL); migrations run as a **single ECS task or a Flyway lock**, not concurrently from 12 booting pods; and **every release must be rollback-able without a DB rollback**.

**App-side:** blue/green via CodeDeploy with canary + alarm-based auto-rollback, graceful shutdown, health checks gating the shift, and **Aurora Blue/Green Deployments** for engine upgrades (creates a synced green cluster, switches in ~1 min).

**Why the schema is the hard part (the problem):** during a rolling or blue/green deploy, **old and new code run at the same time against the same database**, and a rollback means old code must work with whatever schema exists. If a release renames `phone` to `mobile`, old instances immediately break (`column phone does not exist`) the moment the migration runs. So each schema change must be compatible with **both** the previous and next application version.

**Worked example — rename `customer.phone` → `customer.mobile`:**
```sql
-- V12__expand.sql (Release 1, deployed alone)
ALTER TABLE customer ADD COLUMN mobile VARCHAR(20) NULL;       -- old code ignores it
-- Release 2 code: writes phone AND mobile; reads phone
-- Backfill job (batched):  UPDATE customer SET mobile = phone WHERE id BETWEEN :a AND :b AND mobile IS NULL;
-- Release 3 code: reads mobile (falls back to phone), still writes both
-- V15__contract.sql (Release 4, weeks later, after confirming nothing reads phone)
ALTER TABLE customer DROP COLUMN phone;
```
At every step you can roll the app back one version without touching the database.

**Other dangerous changes and safe forms:** adding a `NOT NULL` column → add nullable, backfill, then add the constraint; adding an index on a big table → `CREATE INDEX CONCURRENTLY` (PostgreSQL) so writes aren't blocked; changing a column type → add new column + copy (don't `ALTER TYPE` in place on a large table); removing an enum value → stop writing it, then remove.

**Verification:** run the migration against a production-size copy (restored snapshot) to time it and watch locks; deploy to staging with old+new app versions in parallel; keep the Flyway/Liquibase script immutable once applied (never edit an applied migration — checksum mismatch).

---

### S12. Users in the US complain the app is slow; your infra is in Mumbai. What do you do?

**Order matters — measure before you move.**
1. **Confirm where the time goes** — CloudWatch RUM / real user monitoring. Is it TLS + RTT (network) or backend processing? A 250ms RTT Mumbai↔US, multiplied by TCP + TLS handshakes and 30 asset requests, is often the entire problem.
2. **Cheap wins first:**
   - **CloudFront** for static assets, bundles and cacheable APIs — terminates TLS at a US edge, one long-lived connection over AWS's backbone to origin. Often an 60–80% improvement for a fraction of the effort.
   - **HTTP/2, compression, fewer round trips, asset bundling.**
   - **Global Accelerator** for dynamic/non-cacheable traffic — anycast IPs, edge ingress, backbone transit.
3. **Only then consider multi-region.** Read-local: **Aurora Global Database** (US read replica, <1s lag) with reads served locally and writes forwarded to Mumbai, or **DynamoDB Global Tables** for active-active. Route with **Route 53 latency-based routing**.
4. **Be honest about the cost:** multi-region means data replication, conflict resolution, doubled infra, and a much harder deployment story. "I'd only do it if the business case for US users justifies it — and I'd stage it: CDN → Global Accelerator → read replica → full active-active."

**Why distance makes the app "slow" (plain language):** light in fibre travels ~200 km per millisecond, so Mumbai ↔ US East is ~200–250 ms round trip (RTT) in practice. A page doesn't cost one round trip: DNS lookup + TCP handshake (1 RTT) + TLS handshake (1–2 RTT) + the HTTP request (1 RTT) ≈ 4 RTT ≈ 1 second *before* any data, and every extra API call or asset repeats part of that. Making your server code 50 ms faster won't fix a 1-second network tax.

**How each fix attacks that cost:**
- *CloudFront:* the US user's TLS handshake ends at a nearby edge (~10–20 ms). The edge keeps a warm, persistent connection to Mumbai (so the origin handshake cost is rare) and serves cached assets without going to Mumbai at all.
- *Global Accelerator:* gives static anycast IPs; user traffic enters the AWS backbone at the nearest edge and travels on AWS's private network rather than the congested public internet — better for dynamic, non-cacheable APIs (and TCP/UDP apps CloudFront doesn't suit).
- *HTTP/2 + compression:* multiplexes many requests over one connection and shrinks payloads (gzip/brotli), reducing the number and size of round trips.
- *Reduce chattiness:* a screen that makes 15 sequential API calls costs 15 × RTT — aggregate into a BFF/GraphQL endpoint or parallelise.
- *Multi-region data:* only if users must **write** with low latency; read replicas help reads, but writes still cross the ocean unless you go active-active.

**Verify:** compare p50/p95 page-load from a US synthetic probe (CloudWatch Synthetics) before and after each step; keep the cheapest change that meets the SLO.

---

### S13. Someone accidentally deleted the production RDS instance. What now, and how do you prevent it?

**Now:**
1. **CloudTrail** → who, when, and whether a final snapshot was taken (`DeleteDBInstance` with `SkipFinalSnapshot: false`).
2. **Restore path:** final snapshot → restore (new endpoint!). Otherwise the latest automated snapshot, or **point-in-time recovery** to seconds before deletion (available if automated backups were on; the retained automated backups survive instance deletion for the retention period).
3. **Restore creates a new instance with a new endpoint** → update Route 53 CNAME / Parameter Store, not 40 config files. (If you didn't front the DB with a DNS alias, now you know why you should.)
4. Communicate: status page, RTO estimate, and stop writes to prevent split-brain if a partial failover happened.

**Prevention** 🧠 **"DIABS"**:
- **D**eletion protection enabled (blocks the API call outright).
- **I**AM: no human has `rds:DeleteDBInstance` in prod; break-glass role with MFA + approval only.
- **A**WS Backup with a **vault lock** (immutable) + cross-region, cross-account copies.
- **B**ackup retention 7–35 days, PITR verified by an actual **restore drill** (an untested backup is a hope, not a backup).
- **S**CPs at the org level denying deletion of prod-tagged resources; `DeletionPolicy: Retain` in IaC; console access to prod removed entirely (read-only).

**Terms explained:**
- **Automated backups + PITR (point-in-time recovery):** RDS takes a daily snapshot and continuously uploads transaction logs, so you can restore to *any second* within the retention window (e.g. 14:32:07, just before the delete). Deleting an instance normally deletes its automated backups unless you choose to retain them — which is why the delete dialog offers "create final snapshot" and "retain automated backups".
- **Final snapshot:** a manual snapshot RDS takes at deletion time if `SkipFinalSnapshot=false`. If it was skipped and automated backups weren't retained, only manual or AWS Backup copies remain.
- **Deletion protection:** an instance flag that makes `DeleteDBInstance` fail until someone explicitly disables it — one extra deliberate step.
- **AWS Backup vault lock:** write-once-read-many policy on backups; even admins can't delete recovery points before the retention ends — also protects against ransomware and a malicious insider.
- **SCP (Service Control Policy):** an Organizations-level guardrail that can deny actions regardless of IAM permissions (e.g. deny `rds:DeleteDBInstance` for everything except a break-glass role).

**Restore walk-through (with timings):** (1) identify the target time from CloudTrail (`DeleteDBInstance` event time); (2) `aws rds restore-db-instance-to-point-in-time` (or from snapshot) creating `orders-db-restored`; (3) wait (restore time scales with size — tens of minutes to hours; Multi-AZ/replicas must be re-created); (4) validate row counts and recent data; (5) repoint the app via DNS/Parameter Store; (6) re-enable deletion protection, backups, monitoring, and create read replicas again. State your RTO honestly — *restoring a large DB isn't instant*, which is why an RTO target influences the design (Multi-AZ doesn't protect against a human delete; backups/PITR do).

**Probe — "Why doesn't Multi-AZ save you?"** → Multi-AZ is a standby of the *same* instance: deleting the instance removes both, and a bad `DELETE`/`DROP TABLE` statement is replicated synchronously to the standby. It protects against infrastructure failure, not human mistakes — backups/PITR do.

---

### S14. You need to run a nightly batch job that processes 5 million records. Design it.

**Options, with the trade-off stated:**
- **ECS Scheduled Task** (EventBridge Scheduler → RunTask) with Spring Batch — my default. Full JVM, no time limit, Fargate Spot for ~70% off, and Spring Batch gives chunking, restartability, skip/retry policies, and a job repository for auditability.
- **AWS Batch** — for heavy compute, array jobs, and managed queueing with priorities.
- **Step Functions + Lambda (Map state, distributed map)** — good fan-out to 10,000 parallel workers over S3 objects; watch the 15-min per-Lambda limit.
- **Glue/EMR** — if it's really ETL over a data lake, don't hand-roll it in Java.

**The engineering details that matter more than the service choice:**
- **Chunk and commit** (e.g. 1000 records/transaction) — never one giant transaction; you'll blow up the DB and lose everything on failure.
- **Restartability/idempotency** — a job that fails at record 4.2M must resume, not restart. Spring Batch `JobRepository` + a checkpoint/watermark column.
- **Read from a replica**, throttle writes so you don't starve the OLTP workload; run in a low-traffic window.
- **Only run once** — EventBridge can deliver twice; guard with a distributed lock (`ShedLock` on DynamoDB/Redis) or Batch job de-dup.
- **Observability** — emit progress metrics, alarm on "job didn't complete by 6 AM" (**absence of success is the alarm you forget**), and on duration trending up.
- **Scale strategy** — partitioned Spring Batch steps or work distributed via SQS so you can add workers horizontally.

**What Spring Batch gives you (for the "why"):** a standard model: **Job** → **Step(s)** → each step does *read → process → write* in **chunks**. With chunk size 1000, it reads 1000 records, processes them, writes all 1000, commits once, then moves on. 5 million records = 5,000 small transactions instead of one giant one. The **JobRepository** (a few DB tables) stores progress, so a restart continues from the last committed chunk, and the job can be identified as COMPLETED/FAILED for audit.

```java
@Bean
Step enrichStep(JobRepository repo, PlatformTransactionManager tx) {
  return new StepBuilder("enrich", repo)
      .<Record, Record>chunk(1000, tx)
      .reader(cursorReader)           // streams rows; never loads 5M into memory
      .processor(enricher)
      .writer(jdbcBatchWriter)        // batched INSERT/UPDATE
      .faultTolerant().skipLimit(100).skip(BadRecordException.class)
      .retryLimit(3).retry(TransientDataAccessException.class)
      .build();
}
```
**Rough sizing math:** 5M records at ~2,000 records/s ≈ 42 minutes single-threaded. If the window is 30 min, add partitioning (e.g. 8 partitions by ID range, each processed by a thread or separate task) — but verify the DB can absorb the write rate.

**Operational points:** make the schedule robust (*job didn't start* is also an alarm); pass a deterministic **job parameter** (e.g. `runDate=2026-11-12`) so a duplicate trigger for the same date is rejected by Spring Batch as "already completed"; use a Spot task only if the job is restartable; ensure the container's `stopTimeout` allows a clean checkpoint on shutdown; keep long-lived read cursors from blocking DB maintenance (reading from a replica helps).

---

### S15. Your service is stateful — it stores user sessions in memory. You need to scale to 10 instances. Fix it.

1. **Best: go stateless.** JWT/opaque token with claims, validated per request. No server state → any instance serves any request, scale and deploy freely.
2. **If you need server-side sessions:** **Spring Session + ElastiCache Redis**:
```xml
<artifactId>spring-session-data-redis</artifactId>
```
```yaml
spring.session.store-type: redis
spring.session.timeout: 30m
server.servlet.session.cookie: { http-only: true, secure: true, same-site: lax }
```
3. **Sticky sessions on the ALB are a band-aid** — they cause uneven load, break on scale-in (a terminated task loses its users' sessions), and complicate deploys. Say you know they exist and why you'd avoid them.
4. **Other hidden state to hunt for** (this is the good part of the answer): in-memory caches that must now be distributed or made eventually-consistent, `@Scheduled` jobs now running 10x (fix with ShedLock), local file uploads (→ S3), in-memory rate limiters (→ Redis), local WebSocket registries (→ Redis pub/sub or API Gateway WebSockets), and singleton counters (→ atomic Redis/Dynamo counters).

**The problem in plain terms:** `HttpSession` lives in the memory of the one JVM that created it. With 10 instances behind a load balancer, request 1 (login) may hit instance A and request 2 may hit instance F, which has never heard of that user → the user appears logged out at random. Two fixes: make the *request* carry everything needed (stateless tokens), or move the state to a *shared store* every instance can reach.

**How each option works:**
- *JWT (JSON Web Token):* after login the server issues a signed token containing user ID/roles/expiry; the client sends it on each call; any instance verifies the signature (no lookup). Trade-offs: can't easily revoke before expiry (use short TTL + refresh tokens or a small deny-list), and keep the token small.
- *Spring Session + Redis:* replaces the servlet container's session with one stored in Redis, keyed by the session cookie. Instances stay stateless; Redis is the single source of truth. Trade-offs: an extra network hop per request, Redis must be highly available (Multi-AZ replication group), and session objects must be serialisable (use JSON serialisation rather than JDK to survive class changes).
- *Sticky sessions:* ALB pins a user to one target via a cookie; the state is still in that instance's memory. Scaling in, a crash, or a deploy loses it.

**Migration steps (so it's safe):** (1) find everything in `HttpSession` (search `session.setAttribute`) and trim it to IDs, not big object graphs; (2) add Spring Session behind a flag in staging; (3) deploy with 2 instances and test failover by killing one mid-session; (4) enable autoscaling. **Pitfall:** large session payloads multiply Redis memory and network cost (users × payload).

---

### S16. How would you design a multi-tenant SaaS on AWS?

**Isolation models (state the spectrum, then pick):**
- **Silo** — separate account/VPC/DB per tenant. Max isolation, easiest compliance story, painful ops and cost. For large enterprise customers.
- **Pool** — shared everything with a `tenant_id` column. Cheapest, best utilization, but a bug leaks data across tenants.
- **Bridge** — shared compute, separate schema/database per tenant. Common sweet spot.

**Say this:** "Usually a tiered model — pooled for self-serve/free, siloed for enterprise contracts that demand it — because the isolation requirement is commercial, not technical."

**Implementation points:**
- **Tenant context** — resolve `tenantId` from the JWT (never from a request param), put it in a `ThreadLocal`/`RequestScope`, and enforce it at the **lowest possible layer** — a Hibernate filter, `@TenantId`, or **PostgreSQL Row Level Security**. Relying on every developer to add `WHERE tenant_id = ?` is how data leaks happen.
- **Data isolation in AWS** — S3 key prefixes per tenant + IAM session policies scoped with `${aws:PrincipalTag/tenantId}`; DynamoDB partition key prefixed with tenant; KMS key per tenant for the highest tier.
- **Noisy neighbour** — per-tenant rate limits and quotas, separate queues or shard groups for the biggest tenants, bulkheads.
- **Cost attribution** — tag or meter per tenant so you know your unit economics; that's a real interview differentiator.
- **Onboarding** — automated provisioning via Step Functions + CDK; never manual.

**Plain-language meaning:** a **tenant** is one customer organisation using your shared product (Company A, Company B). **Multi-tenancy** = serving many tenants from one platform so cost per tenant is low, while ensuring no tenant ever sees another's data (isolation) or slows others down (fairness).

**The three models with a concrete example:**
| Model | Example | Good for | Risk |
|---|---|---|---|
| Silo | Each bank gets its own AWS account, VPC, and Aurora cluster | Regulated/enterprise; custom SLAs | Cost and operations scale linearly with tenants |
| Bridge | One ECS service; each tenant has its own Postgres *schema* (or DB) on a shared cluster | Mid-market; moderate isolation | Schema migrations must run per tenant; limits on #schemas |
| Pool | One table `orders(tenant_id, ...)` for everyone | Thousands of small tenants | One missing filter = cross-tenant leak; noisy neighbours |

**How tenant context flows (step by step):** (1) user logs in → JWT contains `tenant_id` claim signed by your IdP; (2) a Spring filter validates the token and stores `tenantId` in a request-scoped holder; (3) data layer uses it automatically — e.g. Postgres RLS:
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
-- app runs:  SET LOCAL app.tenant_id = '<uuid>';  at the start of each transaction
```
so even a forgotten `WHERE` clause returns only that tenant's rows; (4) downstream calls (S3, queues) carry the tenant ID in key prefixes/message attributes; (5) logs and metrics include `tenantId` (low-cardinality *per-tier* metrics, full detail in logs) for support and cost attribution.

**Pitfalls to name:** async workers must restore tenant context from the message (ThreadLocals don't cross threads/queues); caches need tenant-scoped keys (`tenant:42:product:7`) or you'll serve one tenant's data to another; per-tenant AWS resource limits (e.g. number of queues/KMS keys per account) constrain the silo model; testing needs explicit "tenant A can't read tenant B" tests.

---

### S17. An engineer's AWS access keys were committed to a public GitHub repo. Respond.

**Treat it as a live incident, in this order:**
1. **Deactivate the key immediately** (`aws iam update-access-key --status Inactive`) — deactivate before deleting, so you keep the audit trail. Don't wait for approval.
2. **Assess blast radius** — what policies did that principal have? Assume everything it could do, was done.
3. **CloudTrail + Athena** for every action by that access key ID since the commit: new IAM users/roles/keys, `AssumeRole` calls, EC2 launches (crypto-mining is the classic — check every region, not just yours), S3 `GetObject`/`ListBucket` on sensitive buckets, snapshot sharing to external accounts, changes to CloudTrail/GuardDuty itself.
4. **GuardDuty** findings (`UnauthorizedAccess:IAMUser/*`, `CryptoCurrency:EC2/*`) and a Cost Explorer spike check.
5. **Contain** — delete anything the attacker created, revoke active STS sessions (`aws iam put-user-policy` with a `DenyAllExcept` + `aws:TokenIssueTime` condition, since you can't invalidate STS tokens directly), rotate every credential in reach.
6. **Purge & notify** — the key is public forever; rewriting git history doesn't help, treat it as leaked permanently. Notify security/legal; assess breach-notification obligations if data was accessed.
7. **Post-incident prevention:**
   - **No long-lived keys at all** — IAM Identity Center for humans, roles for workloads. This is the root fix; everything else is mitigation.
   - **git-secrets / gitleaks / trufflehog** pre-commit hooks + CI scanning, GitHub push protection.
   - **AWS's automatic quarantine policy** (`AWSCompromisedKeyQuarantine`) — AWS scans public repos and attaches it; know that this exists.
   - Blameless postmortem — the engineer is not the problem, the *possibility* of a long-lived key is.

**Why this is urgent (plain language):** automated bots scan public GitHub commits continuously and can find and use an AWS key within minutes — typically to launch large numbers of GPU/crypto-mining instances in many Regions, which can run up thousands of dollars before you notice. Hence "deactivate first, investigate second."

**Key terms:** an **access key** = `AKIA...` ID + secret that authenticates API calls as an IAM user, valid until deleted. **STS temporary credentials** (from AssumeRole) start with `ASIA...` and expire; the attacker may have exchanged the stolen key for these, so disabling the key alone doesn't kill sessions already issued — hence the deny-policy trick with `aws:TokenIssueTime` (deny everything for tokens issued before *now*). **GuardDuty** = AWS threat-detection service that flags suspicious API activity. **CloudTrail + Athena** = query the API audit log with SQL.

**Example investigation query (Athena over CloudTrail):**
```sql
SELECT eventTime, eventName, awsRegion, sourceIPAddress
FROM cloudtrail_logs
WHERE useridentity.accesskeyid = 'AKIAXXXXXXXXXXXX'
  AND eventTime > '2026-11-12T00:00:00Z'
ORDER BY eventTime;
```
Look for unusual regions, `RunInstances`, `CreateUser`, `CreateAccessKey`, `PutBucketPolicy`, `GetSecretValue`, `StopLogging`.

**What to say about prevention in a Spring Boot context:** keys should never exist in `application.yml`; local dev uses SSO profiles (`aws sso login`) or LocalStack; CI uses **OIDC federation** to assume a role (no stored keys); pre-commit scanners catch `AKIA[0-9A-Z]{16}` patterns; and a **permission boundary/SCP** limits damage even if a credential leaks.

---

### S18. Design a URL shortener on AWS that handles 10,000 reads/sec.

```
Write:  POST /shorten → ALB → ECS Spring Boot
        generate ID (Base62 of a Snowflake/counter, or hash+collision check)
        DynamoDB PutItem (PK=shortCode) with attribute_not_exists condition
Read:   GET /{code} → CloudFront (cache 24h, HUGE hit rate) 
        → API Gateway/ALB → ECS → DAX/Redis → DynamoDB
        → 301/302 redirect
Analytics: async — fire an event to Kinesis Firehose → S3 → Athena. Never block the redirect.
```
**Justify the choices:**
- **DynamoDB over RDS:** the access pattern is a single key lookup at massive scale. Single-digit-ms, on-demand scaling, no connection pool ceiling. This is Dynamo's perfect use case.
- **Cache layers:** CloudFront absorbs most reads (a shortlink is highly cacheable and read-heavy — 100:1 or worse). DAX for microsecond reads on the rest.
- **301 vs 302:** 301 is cached by browsers forever (cheap, but you lose analytics and can never change the target); 302 keeps control. **Say you'd pick based on whether the link must be editable** — this nuance is the whole question for some interviewers.
- **ID generation:** don't use a DB sequence (contention). Base62-encode a Snowflake-style ID, or random 7 chars + a conditional write to handle collisions.
- **Guardrails:** rate limit creation (abuse/spam), URL validation + malware check, TTL for expiring links, and reserved-word blocklist.

**Back-of-envelope math (state it out loud):** 10,000 reads/s ≈ 864M reads/day. If writes are 100:1 vs reads → ~100 creations/s ≈ 8.6M new links/day ≈ 3.1B/year. A 7-character Base62 code gives 62⁷ ≈ 3.5 trillion combinations — plenty. Item size is tiny (~500 bytes), so storage is ~1.5 TB/year; DynamoDB handles this easily. With on-demand mode, 10k reads/s of eventually-consistent 0.5 KB items ≈ 10k × 0.5 RCU = ~5k RCU — and CloudFront/DAX reduce the real hits to a fraction.

**How the flow works:** (1) *Create:* app generates a unique code, writes `{shortCode → longUrl, createdAt, ttl}` using a conditional put (`attribute_not_exists(shortCode)`) so an accidental collision fails instead of overwriting — then it retries with a new code. (2) *Read:* a browser calls `GET /aZ3k9Q1`; CloudFront returns a cached redirect if present; otherwise the request reaches the app, which looks up the code (cache → DynamoDB) and returns `302 Location: https://long.url/...`. The redirect itself is a tiny response, so the whole path is short and cheap.

**ID-generation options explained:** *Counter/Snowflake + Base62* — a unique number (time + machine ID + sequence) converted to letters/digits; no collisions, but IDs are guessable/sequential (fix by shuffling bits or adding randomness if guessing matters). *Random 7 chars + conditional write* — unguessable; at 3.5 trillion space collisions are rare, handled by retry. *Hash of URL* — same URL → same code (dedup) but collisions and the "same URL for two users" analytics problem.

**Pitfalls/probes:** hot keys — a viral link hits one partition; CloudFront/DAX caching absorbs this (and DynamoDB adaptive capacity helps); cache invalidation when a link is edited/disabled (if using 301/long TTL you can't revoke); abuse/phishing links → URL reputation checks; analytics through an async pipeline so the redirect latency stays low (~tens of ms).

---

### S19. Your CloudWatch bill is $8,000/month. Reduce it.

1. **Find the driver in Cost Explorer by usage type**: it's almost always one of — log **ingestion** ($0.50/GB), log **storage** with infinite retention, **custom metrics** ($0.30/metric), or **Logs Insights** queries ($0.005/GB scanned) run by a dashboard on a 1-minute refresh.
2. **Log ingestion:**
   - Turn off DEBUG in prod (someone left it on — check).
   - Stop logging every request body / full stack traces for expected exceptions.
   - **Sample** high-volume INFO logs; keep 100% of WARN/ERROR.
   - Drop ALB/VPC Flow Log noise or send them to **S3 instead of CloudWatch** and query with Athena (an order of magnitude cheaper).
   - Consider **Logs Infrequent Access** class for compliance-only logs.
3. **Retention** — set it on every log group (default is Never Expire). 30 days hot in CloudWatch, export to S3/Glacier for the rest.
4. **Custom metrics** — audit for high-cardinality Micrometer tags. One metric tagged with `userId` = thousands of metrics. Switch to **EMF** or Prometheus/AMP if volume is high.
5. **Dashboards & alarms** — consolidate, drop 1-second high-resolution metrics where 1-minute is fine, use composite alarms.
6. **Prevent** — a log-volume budget per service, a metric-cardinality lint in code review, and a monthly review. Also put a CloudWatch cost dashboard in front of the team that generates the logs.

**Why CloudWatch gets expensive (and how to reason about it):** CloudWatch bills on *volume* — each GB of logs you **ingest** (before you even look at them), each GB **stored** per month, each **custom metric** (unique name + dimension combination), each dashboard/alarm, and each GB scanned by Logs Insights. A chatty app multiplies: 50 tasks × 200 log lines/s × 500 bytes ≈ 5 MB/s ≈ 430 GB/day ≈ 13 TB/month ≈ $6,500/month ingestion alone at $0.50/GB. One DEBUG line inside a hot loop does exactly this.

**Example investigation:** Cost Explorer → service "CloudWatch" → usage type `DataProcessing-Bytes` (log ingestion) is 80% of spend → `aws logs describe-log-groups` sorted by `storedBytes`/CloudWatch `IncomingBytes` per log group shows `/ecs/orders` ingests 300 GB/day → Logs Insights `stats count() by logger` reveals one Hibernate SQL logger at DEBUG (`org.hibernate.SQL`) → set it back to WARN → cost down ~70%.

**Terms:** **EMF** (Embedded Metric Format) = log a specially-structured JSON line and CloudWatch derives metrics from it, avoiding per-metric API calls; **Logs Infrequent Access** = a cheaper log class with limited features for logs you rarely query; **subscription filter** = stream logs to Firehose/S3 for cheap long-term storage; **metric filter** = turn matching log lines into a metric/alarm without custom-metric code.

**Pitfall:** don't "fix" the bill by turning off logging blindly — keep structured ERROR/WARN and key business events, use **sampling** (e.g. log 1 in 100 successful requests) and keep correlation IDs, so incidents are still debuggable.

---

### S20. How do you handle a Spot instance interruption mid-request?

1. **Only put interruptible workloads on Spot** — stateless web tiers behind a queue, batch, CI, async workers. Not the primary DB, not a stateful singleton.
2. **Listen for the 2-minute notice** — the EC2 instance metadata interruption endpoint, EventBridge `EC2 Spot Instance Interruption Warning`, or ECS's automatic **Spot draining**.
3. **Drain properly:** deregister from the ALB target group → stop accepting new work (stop SQS polling, stop taking new HTTP) → finish in-flight → graceful JVM shutdown. Exactly the `S1`/`Q41` machinery, reused.
4. **Make work resumable** — SQS visibility timeout returns unfinished messages automatically; batch jobs checkpoint; HTTP writes are idempotent so the client can retry.
5. **Mix capacity** — ECS **capacity provider strategy**: base of on-demand (say 4 tasks) for the floor, weighted Spot on top for the burst. Diversify across instance types and AZs to lower the correlated-interruption risk.
6. **Measure it** — alarm on interruption rate; if it's high, your instance-type diversification is too narrow.

**Say this:** "Spot is safe when interruption is indistinguishable from a normal scale-in — so I build for graceful scale-in first, and Spot becomes a pricing decision rather than a risk decision."

**What Spot is (plain language):** AWS sells its unused EC2/Fargate capacity at up to ~70–90% discount, but can take it back when it needs it, giving a **2-minute warning** (the *interruption notice*). So Spot = "cheap, but it can vanish." The trick is to design workloads that don't mind a worker disappearing.

**What actually happens to a request, step by step:** (1) AWS posts the 2-minute notice (on ECS Fargate Spot, the task receives `SIGTERM` and has 120 s; with ECS on EC2 Spot, set `ECS_ENABLE_SPOT_INSTANCE_DRAINING=true` so ECS drains the instance's tasks); (2) the task is deregistered from the ALB so no *new* requests arrive; (3) in-flight HTTP requests (usually < a few seconds) finish during the graceful-shutdown window; (4) SQS workers stop polling and either finish the message or leave it — it reappears after the visibility timeout and another worker takes it; (5) after 2 minutes the instance is reclaimed.

**Request-level answer to "mid-request":** if a request is longer than the grace period, it **will** be cut. That's why long operations (exports, video processing) should be **async jobs** with checkpoints, and why synchronous write endpoints should accept an **idempotency key** so the client's retry doesn't create duplicates.

**Config example (ECS capacity provider strategy):**
```json
"capacityProviderStrategy": [
  {"capacityProvider": "FARGATE",      "base": 4, "weight": 1},
  {"capacityProvider": "FARGATE_SPOT", "weight": 3}
]
```
Meaning: always keep 4 tasks on On-Demand; beyond that, 3 of every 4 additional tasks are Spot. If Spot capacity is withdrawn, the On-Demand base still serves traffic while ECS replaces the Spot tasks.

**Pitfalls:** `stopTimeout` on Fargate is capped at 120 s; JVM shutdown work (flush, close pools) must fit; Spot isn't suitable for singletons (cron leader, in-memory state) unless they can fail over quickly.

---

### S21. Design a real-time notification system (in-app + push + email) for 1M users.

```
Producer services → EventBridge (routing by event type + user preference)
   → SQS per channel (email / push / in-app / SMS)
      email  → SES (with a suppression list, bounce/complaint SNS topic)
      push   → SNS mobile push / FCM-APNS
      in-app → API Gateway WebSocket + DynamoDB connection table  (or AppSync subscriptions)
      SMS    → SNS / Pinpoint
   → Worker (ECS) reads prefs from DynamoDB, renders template, sends, records delivery
Persistence: DynamoDB notifications table (PK=userId, SK=timestamp) with TTL for auto-cleanup
```
**Points:**
- **Fan-out per channel with separate queues** so a slow email provider doesn't delay push.
- **User preferences and quiet hours** checked in the worker, not the producer — producers shouldn't know about channels.
- **Idempotency** so a retry doesn't double-notify (dedupe key = `eventId + userId + channel`).
- **Batching and throttling** to respect SES/FCM rate limits; SES sending quota is a real limit to check.
- **Bounces & complaints** → SNS → suppression list, or your SES reputation dies. Interviewers who've run email systems will look for this.
- **WebSocket at scale** — API Gateway WebSockets over self-managed sticky WebSockets on ECS, because connection state lives in Dynamo and scaling is AWS's problem.
- **Priority** — a separate high-priority queue for OTP/security alerts so they never queue behind a marketing blast.

**How to read the design (what each box is for):**
- **EventBridge** receives business events (`OrderShipped`, `PasswordReset`) from producers and routes each to the right channel queues using rules — producers don't know channels exist.
- **One SQS queue per channel** isolates failure/speed differences: email may take 100 ms per send with provider rate limits, push is fast, SMS is costly. If email is slow, only the email queue grows.
- **Worker** (ECS service, scaling on queue age) pulls a message, looks up the user's **preferences** (channel opt-ins, quiet hours, language), renders a **template**, calls the provider, and writes a delivery record.
- **In-app real-time** requires a persistent connection to each online user: **WebSocket** (a long-lived two-way connection). API Gateway WebSocket APIs keep connections for you, and you store `userId → connectionId` in DynamoDB; to notify, the worker calls the API Gateway "post to connection" API with that ID.
- **Offline users** read missed notifications from the DynamoDB `notifications` table when they next open the app (and the TTL cleans up old items).

**Scale arithmetic example:** a campaign to 1M users on 3 channels = 3M sends. At 500 emails/s (SES sending rate) that takes 2,000 s ≈ 33 minutes — so queue them, don't send inline; request a higher SES quota/dedicated IPs for large blasts. Push via FCM/APNs can use thousands of parallel connections.

**Delivery guarantees:** SQS gives at-least-once, so use a deduplication key before sending (conditional `PutItem` on `eventId#userId#channel`). Record statuses (`QUEUED → SENT → DELIVERED/BOUNCED/FAILED`) so support can answer "did I get the notification?".

**Pitfalls:** user-level ordering (not guaranteed with standard queues; use FIFO with `userId` group only where it matters); email reputation (bounce/complaint handling, SPF/DKIM/DMARC DNS records); push tokens go stale — remove on provider "unregistered" errors; respect regional compliance (opt-in for SMS/marketing, DND rules).

---

### S22. A single tenant's traffic is degrading the platform for everyone. Immediate and long-term response?

**Immediate:** identify them (per-tenant metrics — if you don't have them, that's the real finding), apply a **WAF rule or API Gateway throttle** on their key, and if necessary temporarily rate-limit or shed their traffic. Communicate with their account team.

**Diagnose:** are they sending more requests, or *more expensive* requests (an unpaginated export, a N+1-triggering endpoint, a huge search)? The fix differs.

**Long-term:**
- **Per-tenant quotas and rate limits** by contract tier, enforced centrally.
- **Bulkheads** — separate connection pools, separate queues, or **separate compute for the biggest tenants** (shuffle sharding limits the blast radius of any one tenant to a fraction of the fleet).
- **Fix the expensive endpoint** — mandatory pagination, query cost limits, async export via S3 pre-signed download instead of a synchronous 200 MB response.
- **Per-tenant observability + cost attribution** so this is detected by an alarm, not by other customers.

🧠 **"Shuffle sharding: with 8 workers and 2 per tenant, one bad tenant affects 25% of the fleet, not 100%."**

**Why this problem happens (the "noisy neighbour"):** in a shared (pooled) platform all tenants compete for the same CPU, DB connections, queue consumers, and rate limits. One tenant running an expensive export or a script hammering the API can saturate a shared resource, and every other tenant sees latency or errors.

**How to respond, step by step:** (1) *Identify* — slice metrics/logs by `tenantId`: requests/sec, p99, DB time, queue depth. (2) *Contain* — apply a per-tenant rate limit/WAF rule so the platform recovers (return **429 + Retry-After** to that tenant rather than letting everyone degrade). (3) *Communicate* — account manager contacts the customer (maybe it's an integration bug on their side). (4) *Root-cause* — expensive queries vs raw volume. (5) *Prevent* — quotas by plan tier, isolation for large tenants, alerting on per-tenant anomalies.

**How shuffle sharding works (with numbers):** instead of every tenant using all 8 workers, each tenant is deterministically assigned a *random small subset* (say 2 workers, chosen by hashing the tenant ID). A poison/abusive tenant can only take down *its* 2 workers. Another tenant is affected only if they share **both** workers: with 8 workers choose 2 = 28 possible pairs, so the chance of full overlap with a given tenant is 1/28 (~3.6%); partial overlap leaves them a healthy second worker. Combined with client retries, most tenants never notice. (The 25% in the hook refers to the share of capacity lost if the bad tenant exhausts its 2 of 8 workers.)

**Other isolation tools:** per-tenant SQS queues or message groups with concurrency limits; separate Hikari pools or a per-tenant DB statement-timeout/`work_mem` limit; **Lambda reserved concurrency** per tenant tier; "cell-based architecture" — tenants partitioned into independent copies of the stack (cells), so any incident is capped to one cell.

---

### S23. How would you migrate a Spring Boot app from EC2 to ECS Fargate with no downtime?

1. **Containerize** — multi-stage Dockerfile, layered JAR, non-root, JRE-slim/distroless base, `MaxRAMPercentage`. Verify parity locally and in dev.
2. **Externalize state** — sessions to Redis, files to S3, config to Parameter Store, logs to stdout, cron to EventBridge. **This is the actual work**; containerizing is the easy part.
3. **Build the ECS stack in parallel** — new target group behind the **same ALB**, new SGs (referencing pattern), task role with exactly the permissions the instance profile had (audit with Access Advisor first).
4. **Shift traffic gradually** — ALB **weighted target groups**: 5% → 25% → 50% → 100%, watching p99, 5xx, DB connections, and GC metrics at each step. Roll back by changing weights, which is instant.
5. **Watch for the classic surprises:** DNS/DNS-TTL differences, IMDS-based code that assumed EC2 metadata, local disk assumptions, `availableProcessors()` changes affecting thread pools, connection count multiplying because tasks > instances, and timezone/locale differences from a slimmer base image.
6. **Decommission** — keep EC2 running (scaled down) for a week as a rollback, then delete via IaC so it doesn't come back.

**Why "no downtime" is achievable (the mechanism):** both the old EC2 fleet and the new Fargate tasks are registered as *targets* behind the same ALB. The ALB can split traffic across target groups by **weight**, so the cutover is a gradual dial, not a switch. Nothing changes for clients (same DNS name, same ALB).

**Step-by-step example:**
1. Build & push the image to ECR; create the Task Definition (CPU/memory sized from the EC2 instance's observed usage, not its size — e.g. a 4 vCPU/16 GB EC2 running a JVM at 1.5 GB heap may need only 1 vCPU/3 GB).
2. Create an ECS service in private subnets with a new **target group** (type `ip` — required for `awsvpc`/Fargate) and the *same health check path*.
3. Add the new target group to the ALB listener rule with weight 5, the old EC2 group weight 95. Monitor p99, 5xx, DB connections.
4. Increase to 25 → 50 → 100, pausing at each step (e.g. 30 minutes or an hour, across a traffic peak if possible).
5. If anything regresses, set weights back (takes effect within seconds).

**Sizing sanity check for connections:** if EC2 had 4 instances × pool 20 = 80 connections but you now run 12 small tasks × pool 20 = 240, the DB may hit `max_connections`. Reduce pool size per task or add RDS Proxy before shifting traffic.

**EC2-specific assumptions that break in containers:** reading `/etc/hostname`/instance ID for logging (use the ECS metadata endpoint or task ARN); files written under `/var/log` or `/tmp` (container disk is ephemeral and small); `crontab` entries on the host; JVM flags that assume the host's RAM (use percentages); SSH-based debugging (use **ECS Exec** instead); and security-group rules that referenced instance SGs (the DB must now allow the task's SG).

**Pitfall:** don't forget the **task role vs execution role** split (see Q11) and that the ECS service needs outbound access (NAT or VPC endpoints) to pull from ECR and write logs, or tasks fail with `CannotPullContainerError`.

---

### S24. Your team wants Kubernetes. You think ECS is enough. How do you handle it?

**This is a soft-skills question wearing an AWS costume. Answer accordingly.**

"I'd start by asking what problem Kubernetes solves for us that ECS doesn't — that's a real conversation, not a rhetorical one. Good reasons exist: an existing K8s platform team, portability requirements, needing Istio/ArgoCD/Kubeflow, or hiring/skills alignment. Bad reasons: resume-driven development and 'it's the standard.'

Then I'd make the trade-off concrete: EKS adds a control plane cost, upgrade cycles every ~4 months, IRSA/networking/CNI IP-exhaustion complexity, and a much bigger security surface — all of which needs owners. ECS gives us 90% of the value with a fraction of the operational load, and integrates with IAM/ALB/CloudWatch natively.

If we have a platform team who wants it and can own it, EKS is a defensible choice and I'll get good at it. If it's three backend engineers who also own the product roadmap, ECS. I'd write it up as a short ADR with the criteria, let the team decide with the data, and then commit fully to whatever we choose — I'd rather have alignment than be right."

---

### S25. Data must never leave India (RBI/data-localisation). How do you enforce it architecturally?

1. **Region choice** — everything in `ap-south-1`/`ap-south-2`. Multi-AZ for HA, no cross-region replication of regulated data.
2. **Enforce it, don't trust it** — **SCPs** at the org level with `aws:RequestedRegion` conditions denying resource creation outside approved regions. That's the control an auditor will ask for.
3. **Watch the leaks** — S3 Cross-Region Replication, DynamoDB Global Tables, cross-region snapshot copies, CloudFront caching PII responses at foreign edges (set `Cache-Control: private, no-store` for PII), CloudWatch/monitoring SaaS exporting logs abroad, third-party SDKs, and **global services** (IAM, Route 53, CloudFront) whose control planes are in `us-east-1`.
4. **Encryption + key residency** — CMKs in-region; some regimes require the key never leave the jurisdiction. KMS keys are regional, which helps.
5. **Evidence** — AWS Config rules for region compliance, CloudTrail org trail, Audit Manager for the report, Macie to detect PII landing where it shouldn't.
6. **DR within the constraint** — a second Indian region, or a different AZ/account strategy plus tested backups, since a foreign DR region is off the table.

**Say this:** "Compliance requirements should be enforced by guardrails, not documents. If a developer *can* replicate a bucket to Ireland, eventually someone will."

**Plain-language background:** *data localisation* (e.g. RBI's payment-data storage rules) means certain regulated data must be stored/processed only inside a country. AWS Regions are isolated, and your data doesn't leave unless *you* configure it — so the architecture's job is to make leaving **impossible by default** and **provable to an auditor**.

**What the SCP looks like (the key control):**
```json
{ "Effect": "Deny",
  "NotAction": ["iam:*", "organizations:*", "route53:*", "cloudfront:*", "support:*", "sts:*"],
  "Resource": "*",
  "Condition": { "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1", "ap-south-2"] } } }
```
Anyone (even an account admin) trying to create an EC2/S3/RDS resource outside India is denied. The `NotAction` list exempts *global* services that aren't tied to a Region and would otherwise break.

**Subtle leak paths explained:** *Logs and backups* — an app log line containing PAN/Aadhaar shipped to a third-party SaaS in the US is a violation even if the DB is in Mumbai. *Edge caching* — CloudFront stores copies at edge locations worldwide, so personalised/PII responses must be `private, no-store`, or restrict with geo/price class settings. *Support & debugging* — a developer copying a prod DB dump to a laptop abroad. *Control-plane metadata* — IAM/Route 53 configuration lives globally, but it's configuration, not customer data; know the distinction (check with compliance for edge cases).

**Verification story:** AWS Config rule/conformance pack to flag resources outside approved Regions; Macie scans for PII in buckets; periodic access reviews; and Audit Manager/Artifact reports as evidence. Mumbai (`ap-south-1`) and Hyderabad (`ap-south-2`) as the two in-country Regions enable in-country DR — one of the reasons the second Region matters.

---

### S26. Requests intermittently take 30 seconds. p50 is 80ms. Find it.

**Structure: it's a tail-latency problem, so look for things that happen occasionally.**
1. **X-Ray / distributed trace on the slow requests** — get a trace of an actual 30s request. Everything else is guessing.
2. **Usual suspects, ranked:**
   - **Connection pool exhaustion** — `hikaricp.connections.pending` > 0 means threads waiting for a connection. 30s smells exactly like a default `connectionTimeout`. Check pool size vs concurrency.
   - **GC pause** — a full GC on a big heap. Check `jvm.gc.pause` max, GC logs.
   - **Retry + timeout stacking** — a downstream 3s timeout × 3 retries × 2 nested services = 18–30s. **Timeout budgets must shrink downstream.** Extremely common.
   - **DNS resolution** — JVM caching (`networkaddress.cache.ttl`) or a resolver hiccup; also the classic "DNS lookup on every SDK call."
   - **Lock contention / synchronized block**, or a `ThreadLocal`/thread-pool starvation.
   - **Cold start** — first request to a new task pays JIT + lazy bean init + first-connection cost. Check whether slow requests correlate with new task IPs.
   - **NAT Gateway / idle connection reset** at 350s, or an ENI/port exhaustion.
   - **Noisy neighbour / EBS burst credit exhaustion** (gp2 running out of burst = disk latency cliff), or **RDS `t`-class CPU credit exhaustion** — a brutal one because CPU looks fine and then falls off a cliff.
   - **A specific tenant's expensive query** hitting a missing index only for large accounts.
3. **Correlate** — plot p99 against deploys, task age, tenant, endpoint, and instance ID. `stats p99(duration) by uri, taskId` in Logs Insights.
4. **Fix and verify** — and add an alarm on p99, not average. 🧠 **"Averages hide outages. Always alarm on percentiles."**

**What "p50 / p99" and "tail latency" mean:** percentile *p50* = the median — half of requests are faster than 80 ms. *p99* = 99% are faster than this value; the slowest 1% are slower. If 1 request in 100 takes 30 s, the average looks fine (~0.4 s) but users who hit it are furious, and a page that makes 20 backend calls hits a slow one ~18% of the time. Hence "tail latency" matters.

**The clue in "30 seconds":** an exact round number is a **timeout**, not natural slowness. Hunt for a configured 30 s value: Hikari `connectionTimeout` (default 30 s), a Feign/RestTemplate read timeout, the ALB/API Gateway limit (API GW 29 s), a DB lock wait, JDBC socket timeout, or a retry chain adding up (3 s × 10). Find which timeout equals the symptom, then ask *why that resource was unavailable*.

**Worked example:** trace shows the 30 s request spent 29.9 s in `HikariPool.getConnection`. Metrics show `hikaricp.connections.pending` spikes every few minutes; a scheduled report job grabs 10 connections for 2 minutes, starving the API (pool of 10). Fix: separate pool/DataSource for the job, shorter `connection-timeout` (fail fast at 3 s), move reports to a replica, alarm on pending > 0. Verified by re-running the load test and watching p99.

**Method tips:** (1) add the `traceId` to every log line so one slow request can be followed end-to-end; (2) log slow requests (> 2 s) with their dependency timings; (3) check the pattern — periodic (cron/GC/credit exhaustion), per-instance (bad host/cold start), per-tenant/endpoint (data-dependent query), or random (network/DNS); (4) change one thing at a time.

---

### S27. Design a CI/CD setup where developers can spin up an isolated environment per pull request.

```
PR opened → GitHub Actions/CodeBuild
  → build image, tag with PR number
  → cdk deploy PR-123 stack:
      ECS service (1 task, small), ALB listener rule host: pr-123.dev.example.com
      Aurora Serverless v2 (min 0.5 ACU)  OR  a cloned Aurora snapshot (copy-on-write = fast + cheap)
      DynamoDB tables prefixed pr-123-
  → run migrations + seed data
  → post the URL as a PR comment
PR merged/closed → cdk destroy PR-123   (+ a TTL Lambda that nukes anything >5 days old)
```
**What makes this answer good:**
- **Cost control is the hard part** — Aurora Serverless v2 scaling to 0.5 ACU, Fargate Spot, aggressive TTL cleanup, and a hard cap on concurrent environments. Otherwise you wake up to 60 abandoned environments.
- **Aurora cloning** gives realistic data volumes in minutes without copying terabytes.
- **Shared vs isolated** — fully isolated per PR is ideal but expensive; a pragmatic middle ground is isolated compute + a shared, seeded database with namespaced data.
- **Data:** never clone production PII into ephemeral envs — anonymize/synthesize.
- **DNS + certs** — a wildcard ACM cert on `*.dev.example.com` avoids per-environment cert provisioning.

**Why per-PR environments (the problem):** a single shared staging environment becomes a queue — two PRs overwrite each other's changes, tests interfere, and reviewers can't try a change without checking out code. An ephemeral environment per PR lets a reviewer/QA open `https://pr-123.dev.example.com` and see exactly that branch running, then it disappears.

**How the pieces work together:**
- *Naming:* everything is parameterised by the PR number (stack name `orders-pr-123`, DB name, DynamoDB table prefix, S3 prefix) so environments can't collide.
- *Routing:* one shared dev ALB with a **host-header rule** per PR (`pr-123.dev.example.com` → that PR's target group), with a wildcard DNS record `*.dev.example.com` → ALB. No new load balancer per PR (an ALB costs ~$16+/month each).
- *Compute:* one small Fargate task (0.25–0.5 vCPU, Spot) with `desiredCount=1`.
- *Data:* Aurora clone from a sanitised snapshot (copy-on-write: pages are copied only when changed, so a clone is fast and initially near-free) or a seeded Serverless v2 database. Migrations (Flyway) run at deploy time against *that* DB.
- *Secrets/config:* generated per environment (Parameter Store path `/pr/123/...`), never copied from prod.
- *Teardown:* a PR `closed` webhook runs `cdk destroy`; a scheduled Lambda lists stacks tagged `ephemeral=true` older than N days and deletes them.

**GitHub Actions sketch:**
```yaml
on: { pull_request: { types: [opened, synchronize, reopened] } }
jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4     # OIDC role, no stored keys
      - run: ./mvnw -B verify && docker build -t $ECR/orders:pr-${{ github.event.number }} . && docker push ...
      - run: cdk deploy orders-pr-${{ github.event.number }} --require-approval never -c prNumber=${{ github.event.number }}
```
**Pitfalls:** orphaned environments (tags + TTL + budgets), AWS quota limits (ALB rules per listener = 100 by default, so cap concurrent PR envs), secrets leaking via PRs from forks (run deployment workflows only for trusted branches), and the cost of long-running DBs.

---

### S28. How do you decide the number of ECS tasks and their size for a new Spring Boot service?

**Say this first:** "I don't guess — I load test, then set capacity from the data, then let auto-scaling handle the rest."

1. **Baseline a single task** — deploy 1 task, ramp load with k6/Gatling until p99 breaks your SLO. Now you know **requests/sec per task**. That single number drives everything.
2. **Size the task** — Java has a memory floor (heap + metaspace + threads + direct buffers). Start at 1 vCPU / 2 GB, measure actual RSS and CPU at load. Prefer **more small tasks over fewer large ones** (finer scaling granularity, smaller blast radius, better bin-packing) — but not so small that JVM overhead dominates, which is roughly below 0.5 vCPU/1 GB.
3. **Set min tasks** = peak_rps / rps_per_task × safety factor, and **never below 2–3** so you survive an AZ loss and a rolling deploy.
4. **Set max tasks** = what your *database* can survive, not what your budget allows. `max_tasks × pool_size ≤ 0.8 × max_connections`. This is the constraint people forget.
5. **Scaling policy** — target tracking on `ALBRequestCountPerTarget` (derived from step 1) or SQS backlog per task; scale out fast, scale in slow (asymmetric cooldowns) because JVM warm-up is expensive.
6. **Re-evaluate quarterly** and after any major dependency change.

**Worked example with numbers:**
- Load test shows one 1 vCPU/2 GB task handles 150 req/s at p99 < 300 ms (the point where p99 breaks SLO is ~200 req/s; run at ~75% of that for headroom).
- Expected peak: 1,200 req/s → 1,200 / 150 = 8 tasks; with a 1.3 safety factor → ~10 tasks at peak. Baseline traffic (400 req/s) → 3 tasks → **min = 3, max = 16** (room for growth).
- DB check: max 16 tasks × pool 10 = 160 connections; if RDS `max_connections` is 200 → 80% = 160 → just OK. If it were 100, you'd cap max tasks lower, reduce pool size, or add RDS Proxy.
- Scaling target: `ALBRequestCountPerTarget` is counted **per target per minute**. One task handles 150 req/s = 9,000 requests/min at the safe level, so set the target-tracking value somewhat below that (e.g. ~6,000) to scale out before the limit.

**Why min ≥ 2–3 tasks across AZs:** with 1 task there's no redundancy; with 2 tasks in 2 AZs losing an AZ halves capacity at once, so size so that *N−1* tasks can carry normal load.

**Why bigger isn't always better:** one task with 4 vCPU running 4× the traffic means a crash loses 4× the capacity and scaling granularity is coarse; very small tasks (0.25 vCPU) waste a larger fraction on JVM/runtime overhead and have slow JIT warm-up. Choose the size that gives the best *cost per request at SLO*, measured.

**Asymmetric scaling:** scale-out cooldown short (60 s) to react quickly; scale-in cooldown long (5–10 min) to avoid thrashing, since each new JVM costs a warm-up period and a deploy-like load spike.

**Watch for:** CPU throttling on Fargate when bursting above the task's vCPU; memory growth over time (leaks); and that the load test must use realistic data and downstream latency (mocking the DB as instant gives wrong numbers).

---

### S29. Your monolith's deployment takes 45 minutes and the team is afraid to deploy. Improve it.

**Diagnose where the 45 minutes goes** — build, tests, image push, or rollout? Optimize the actual bottleneck.

- **Build:** Maven/Gradle build cache + CodeBuild local caching, parallel test execution, **layered JARs** so Docker reuses the dependency layer, and a CodeBuild instance with more vCPUs (cheaper than engineer time).
- **Tests:** split into fast unit tests (gate the PR) and slow integration tests (run in parallel shards, or post-merge); Testcontainers reuse; kill flaky tests ruthlessly — a flaky suite is why people fear deploying.
- **Rollout:** if it's 20 minutes of rolling restarts because the JVM takes 3 minutes to boot, fix startup (AOT/CDS, lazy init, trim autoconfig) and increase `maximumPercent` so more tasks start in parallel.
- **Fear, not speed, is the real problem:** blue/green with canary + automatic rollback on alarm, feature flags so deploy ≠ release, backward-compatible migrations, and a rollback that's one click and rehearsed. **When rollback is trivial, deployment stops being scary.**
- **Then increase frequency** — small, frequent deploys are safer than big rare ones. Push toward deploy-on-merge.

**Why this is really about risk (the explanation):** a 45-minute deploy of a big monolith means each release bundles weeks of changes; when something breaks, nobody knows which change caused it and rollback is slow and unrehearsed — so people deploy less often, making each release bigger and riskier (a vicious circle). The fix is to shrink the batch size and make failure cheap.

**Make the 45 minutes visible first (example breakdown):** build 8 min, unit+integration tests 20 min, image build/push 5 min, ECS rolling restart 10 min, manual checks 2 min. Fix the biggest slice first: e.g. split the 20-minute test run into 4 parallel shards → 6 min; enable Maven/Gradle caching and layered JARs → build 3 min, push 1 min (only changed layers); allow `maximumPercent=200` and start more tasks at once → rollout 4 min. Total ≈ 15 minutes without any risky change.

**Concepts explained:**
- **Feature flags** separate *deploying* code (turn it on in production but hidden) from *releasing* it (turn it on for users, gradually). A bad feature is switched off instantly with no redeploy.
- **Canary release** exposes the new version to a small slice of traffic and compares its error/latency metrics against the old; **automatic rollback** on alarm removes the "human must notice" step.
- **Expand/contract migrations** (see S11) mean the app can roll back even though the database cannot.
- **DORA metrics** (deployment frequency, lead time, change failure rate, time to restore) are the standard way to show progress to management.

**Longer-term path:** modularise the monolith (clear package/module boundaries, `ArchUnit` tests), then deploy-on-merge behind flags, then extract services only where independent scaling or team autonomy justify it. **Pitfall:** don't jump to microservices to fix a *process* problem — you'll multiply the deployment pipelines and the fear.

---

### S30. Design a system to process 1 million payment transactions per day with exactly-once semantics and a full audit trail.

**Say this first:** "There's no exactly-once *delivery* in a distributed system. I get exactly-once *effect* with at-least-once delivery plus idempotency, and I make money movement auditable and reconcilable rather than assuming it's perfect."

```
API (idempotency-key header, required)
  → ECS Spring Boot: validate, persist PaymentIntent (status=PENDING) + outbox row in ONE transaction
  → Outbox poller / DynamoDB Stream → SQS FIFO (MessageGroupId = accountId)
  → Worker: conditional state transition (PENDING→PROCESSING) via optimistic lock / DB CAS
            call PSP with the same idempotency key
            record result, transition to SUCCESS/FAILED
  → Step Functions saga for multi-step flows with compensating refunds
  → Every state change appended to an immutable audit log:
       Aurora with append-only ledger table (or QLDB-style hash chain) + Kinesis → S3 (Object Lock, WORM)
  → Nightly reconciliation: our ledger vs PSP settlement file → exceptions queue → ops dashboard
```
**Hit these points explicitly:**
1. **Idempotency key from the client**, stored with a unique constraint → a retried POST returns the original result, never charges twice.
2. **Outbox pattern** — no dual-write between DB and queue.
3. **State machine with guarded transitions** — a payment can only move PENDING→PROCESSING once (`UPDATE ... WHERE status='PENDING'` and check the row count). This, not the queue, is what actually enforces once-only processing.
4. **Downstream idempotency** — pass the same key to the payment provider; they enforce it too. Defense in depth.
5. **FIFO ordering per account** so concurrent operations on one balance serialize.
6. **Immutable audit trail** — append-only, hash-chained, WORM S3 with Object Lock + KMS. Never `UPDATE` a ledger row; write a compensating entry.
7. **Reconciliation is mandatory** — assume something will diverge and build the detection for it. This is the answer that separates people who've built payments from people who've read about them.
8. **Security/compliance** — PCI scope minimization (tokenize via the PSP, never store PANs), separate account/VPC for the cardholder data environment, KMS CMKs, least privilege, CloudTrail + Config, encryption in transit and at rest.
9. **Capacity check:** 1M/day ≈ 12 TPS average, maybe 100 TPS peak. That's *small* — say so, because the honest answer is that the difficulty here is correctness and audit, not scale. Interviewers love a candidate who doesn't over-engineer.

**Jargon decoded:**
- **Exactly-once semantics:** each payment must take effect once — never lost (money not moved), never duplicated (double charge). Networks retry, services crash mid-way, and queues can redeliver, so true exactly-once *delivery* can't be guaranteed; instead combine **at-least-once delivery** (retry until acknowledged) with **idempotency** (repeating an operation changes nothing) to achieve an exactly-once *effect*.
- **Idempotency key:** a unique ID the client generates per payment attempt (e.g. UUID in an `Idempotency-Key` header). The server stores `(key → result)` in a table with a unique constraint; if the same key arrives again (client timeout then retry), it returns the stored result instead of charging again.
- **PSP:** Payment Service Provider (the gateway/bank API). **PAN:** Primary Account Number (card number). **PCI-DSS:** the card-data security standard — keep PANs out of your systems by using the PSP's tokenisation.
- **Ledger:** an append-only record of debits/credits; balances are derived. **Reconciliation:** comparing your ledger with the PSP/bank's settlement file to find mismatches (missing, duplicate, or different amounts).

**Guarded state transition, in code:**
```java
// only one worker can win this; others update 0 rows and stop
int updated = jdbc.update(
  "UPDATE payment SET status='PROCESSING', version=version+1 " +
  "WHERE id=? AND status='PENDING'", id);
if (updated == 0) return;          // already taken – duplicate message ignored
pspClient.charge(req, idempotencyKey);   // PSP also dedupes using the same key
```
**Crash scenarios to walk through (this shows depth):** (a) crash after PSP charge but before saving SUCCESS → on retry the worker sees `PROCESSING`, calls the PSP with the *same idempotency key*, gets the original result, records it; (b) PSP timeout with unknown result → don't mark FAILED; query the PSP status API (or leave `UNKNOWN` for reconciliation); (c) duplicate API POST → unique idempotency key returns the first response; (d) duplicate SQS delivery → guarded transition prevents double processing.

**Audit trail details:** each state change is a row `(paymentId, fromState, toState, actor, timestamp, requestId)`; chain a hash of the previous entry so tampering is detectable; export to S3 with Object Lock (compliance mode) for immutability; CloudTrail covers *infrastructure* actions while this ledger covers *business* actions.

---
## 14. Rapid-Fire Comparisons & Cheat Sheet

### The "vs" table — memorize these one-liners

| Comparison | One-line answer |
|---|---|
| **Multi-AZ vs Read Replica** | Availability vs read scaling. Sync vs async. Standby unreadable vs readable. |
| **SG vs NACL** | Stateful/instance/allow-only vs stateless/subnet/allow+deny/first-match. |
| **ALB vs NLB** | L7 routing & features vs L4 speed & static IPs. |
| **SQS vs SNS** | Pull queue, one consumer vs push pub/sub, many subscribers. |
| **SNS vs EventBridge** | Fast dumb fan-out vs slower smart content-based routing. |
| **SQS vs Kinesis** | Delete after processing vs retained & replayable, ordered per shard. |
| **S3 vs EBS vs EFS** | Object/anywhere vs block/one-instance vs NFS/many-instances. |
| **Secrets Manager vs Parameter Store** | Paid + auto-rotation vs free + DIY rotation. |
| **STS AssumeRole vs Access Key** | Temporary + auditable vs permanent + leakable. Always the former. |
| **CloudWatch vs CloudTrail** | Performance/health vs who-did-what audit. |
| **Fargate vs EC2 launch type** | No hosts to manage vs cheaper at steady scale + host control. |
| **Lambda vs Fargate** | Event-driven, sub-15-min, cold starts vs long-running, warm, predictable. |
| **CloudFormation vs CDK vs Terraform** | Native YAML vs real programming language vs multi-cloud + best plan. |
| **API Gateway vs ALB** | API management features vs cheap high-throughput HTTP routing. |
| **DynamoDB vs RDS** | Known access patterns + infinite scale vs ad-hoc SQL + joins + transactions. |
| **GSI vs LSI** | Any key/eventually consistent/add anytime vs same PK/strongly consistent/create-time only. |
| **On-Demand vs Provisioned (Dynamo)** | Spiky/unknown vs predictable + cheaper. |
| **Savings Plan vs Reserved vs Spot** | Flexible commit vs instance-specific commit vs interruptible 90% off. |
| **Alias vs CNAME** | Apex-capable + free vs subdomain-only + charged. |
| **Liveness vs Readiness** | Restart me? vs Route to me? |
| **Horizontal vs Vertical scaling** | More instances (preferred, fault-tolerant) vs bigger instance (has a ceiling). |

---

### Numbers worth memorizing

| Thing | Limit |
|---|---|
| S3 max object | 5 TB (single PUT: 5 GB) |
| S3 multipart | parts 5 MB–5 GB, max 10,000 parts |
| S3 request rate | 5,500 GET / 3,500 PUT per prefix per sec (auto-scales) |
| SQS message size | 256 KB (Extended Client → S3 for larger) |
| SQS visibility timeout | default 30s, max 12h |
| SQS retention | default 4 days, max 14 |
| SQS FIFO throughput | 300 msg/s (3,000 batched); high-throughput mode ~70,000/s |
| Lambda timeout / memory / payload | 15 min / 10 GB / 6 MB sync, 256 KB async |
| Lambda ephemeral `/tmp` | 512 MB–10 GB |
| Lambda default concurrency | 1,000 per region per account |
| DynamoDB item size | 400 KB |
| DynamoDB RCU/WCU | 1 RCU = 4 KB strong read; 1 WCU = 1 KB write |
| DynamoDB partition throughput | 3,000 RCU / 1,000 WCU per partition |
| EBS gp3 baseline | 3,000 IOPS / 125 MB/s (independent of size) |
| VPC reserved IPs per subnet | 5 |
| ALB idle timeout | 60s default |
| NAT Gateway idle timeout | 350s |
| Aurora storage / replicas | 128 TB / 15 readers |
| RDS backup retention | 0–35 days (PITR to 5 min granularity) |
| EventBridge rule targets | 5 per rule |
| Kinesis shard | 1 MB/s in, 2 MB/s out, 1,000 records/s |
| Kinesis retention | 24h default, up to 365 days |

---

### Architecture patterns to name-drop (and be able to explain in one line)

- **Transactional Outbox** — DB write + event publish atomically via an outbox table.
- **Saga** — distributed transaction as a series of local transactions with compensations.
- **CQRS** — separate write model (RDS) from read model (DynamoDB/OpenSearch), synced by events.
- **Strangler Fig** — incrementally route paths from the monolith to new services.
- **Circuit Breaker / Bulkhead / Backpressure** — failure isolation.
- **Cache-Aside / Write-Through / Refresh-Ahead** — caching strategies.
- **Idempotent Consumer** — at-least-once delivery made safe.
- **Sidecar / Ambassador** — cross-cutting concerns in a co-located container.
- **Shuffle Sharding** — limit a noisy tenant's blast radius.
- **Expand/Contract (Parallel Change)** — zero-downtime schema evolution.
- **Fan-out (SNS→SQS)** — one event, many independent durable consumers.
- **Claim Check** — big payload in S3, pointer in the message.
- **Anti-Corruption Layer** — translate between legacy and new domain models.

**Plain-language "problem it solves + AWS/Spring example" for the less obvious ones:**

| Pattern | Problem it solves | Example |
|---|---|---|
| **CQRS** (Command Query Responsibility Segregation) | Reads and writes have different shapes/scale needs; one model serves neither well | Writes go to Aurora (normalised); an event updates a denormalised DynamoDB/OpenSearch view used by search pages |
| **Backpressure** | A fast producer overwhelms a slow consumer | A bounded SQS queue + worker concurrency limits; API returns 429/503 instead of accepting unlimited work |
| **Cache-Aside / Write-Through / Refresh-Ahead** | Where/when the cache is populated | Aside: app loads on miss (`@Cacheable`); Write-through: write to cache and DB together; Refresh-ahead: reload hot keys before they expire to avoid stampedes |
| **Sidecar / Ambassador** | Cross-cutting concerns (logging agent, proxy, TLS) shouldn't be coded into every app | ECS task with the app container + AWS Distro for OpenTelemetry collector or Envoy (App Mesh/Service Connect) container |
| **Claim Check** | Messages have size limits (SQS 256 KB) and large payloads slow brokers | Upload the 20 MB document to S3, send `{bucket, key}` in the message; consumer fetches it (SQS Extended Client Library automates this) |
| **Fan-out (SNS→SQS)** | One event must reach many consumers that fail/scale independently | `OrderPlaced` topic with queues for email, inventory, analytics |
| **Parallel Change (Expand/Contract)** | Old and new code run together during deploys | Add column → dual-write → switch reads → drop old column (S11) |

**How to use these in an answer:** name the pattern *after* describing the problem — "the DB write and the event publish can diverge, so I use a transactional outbox" lands better than leading with the term.

---

### Interview delivery tips for 9-years-experience level

1. **Lead with the decision, then justify.** "I'd use ECS Fargate — because…" beats a 90-second ramble that arrives nowhere.
2. **Always name the trade-off you rejected.** Seniority is demonstrated by knowing the cost of your choice, not by knowing more services.
3. **Ask clarifying questions on scenario/design questions.** Scale? RTO/RPO? Consistency requirement? Budget? Team size? Existing stack? Jumping straight to a diagram is a mid-level move.
4. **Quantify.** "20 tasks × 10 connections = 200, and a db.t3.medium allows ~90" is worth more than "we'd tune the pool."
5. **Bring one real war story per major service** — an outage you debugged, a bill you cut, a migration you led. Structure it **STAR**: Situation, Task, Action, Result (with a number).
6. **Say "I don't know" cleanly, then reason.** "I haven't used MSK Serverless in production — but based on how MSK works, I'd expect…" is a strong answer. Bluffing is the only fatal one.
7. **Connect AWS back to Java.** Anyone can recite service descriptions. Almost nobody talks about `MaxRAMPercentage` on Fargate, HikariCP vs `max_connections`, the ALB/Tomcat keep-alive race, or SnapStart's CRaC hooks. **That is your edge — use it in every answer you can.**
8. **Talk about how you verified it worked.** Metrics, load tests, chaos drills. Engineers who measure sound different from engineers who assume.

---

## 15. Questions YOU Should Ask

Asking good questions is scored. Pick 3–4:

- How is your AWS estate organized — single account or multi-account with Organizations/Control Tower?
- What's the deployment story? How many times a day does a service ship, and how long does a rollback take?
- ECS, EKS, or EC2 — and how did you land there?
- Who owns infrastructure: a platform team, or does each service team own its own Terraform/CDK?
- What's your on-call rotation like, and what were the last two incidents about?
- How do you handle database migrations with zero downtime today?
- What does observability look like — CloudWatch, Datadog, Grafana? Do you have SLOs?
- Where's the biggest chunk of technical debt, and is there appetite to fix it?
- What would you want me to have accomplished in the first 90 days?

---

## Final 10-Minute Revision Card

🧠 **The stack story:** Route 53 → CloudFront → WAF → ALB → ECS Fargate (Spring Boot, multi-AZ) → Aurora Multi-AZ + ElastiCache; async via SQS/SNS/EventBridge; secrets in Secrets Manager; config in Parameter Store; metrics/logs/traces to CloudWatch + X-Ray; deployed by CDK + CodePipeline with blue/green and alarm-based auto-rollback. Roles not keys, everything encrypted, everything tagged.

🧠 **The five rules that answer half of all scenario questions:**
1. **Make it stateless.** State goes to Redis, S3, or the database.
2. **Make it idempotent.** Everything retries; assume it will.
3. **Set timeouts, and shrink them downstream.** Untimed calls are outages.
4. **Fail fast, degrade gracefully, roll back cheaply.**
5. **Measure it, alarm on percentiles, and tag it so you know what it costs.**

**Good luck. Lead with the decision, name the trade-off, and bring the Java details nobody else brings.**
