# 06 — AWS for Backend Engineers

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak. Output blocks are marked **(what you'd see)**.

This is AWS as a *backend* engineer gets asked about it — not "what is S3", but "why is your service slow, expensive or insecure on AWS."

**A warning given your background:** you know this deeply from the DevOps side. The risk is answering as an infrastructure engineer when they asked an application question. Keep landing it back on the service: *"…which matters to the API because…"*

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **IAM role** | A set of permissions you *borrow* temporarily, rather than own. Credentials expire. | "Roles give short-lived credentials — no long-lived keys anywhere." |
| **IRSA** | IAM Roles for Service Accounts. Lets a Kubernetes pod get its own AWS identity. | "Each pod gets least-privilege IAM instead of sharing the node's role." |
| **OIDC** | A standard way for one system to prove identity to another using signed tokens. | "EKS publishes an OIDC provider, and IAM trusts it." |
| **Security group** | A firewall around a network interface. It remembers connections. | "Security groups are stateful — that's the difference from NACLs." |
| **NACL** | A firewall at the subnet level. It does *not* remember connections. | "Stateless, so you must allow the return traffic explicitly." |
| **NAT Gateway** | Lets private servers reach the internet outbound. Charged hourly **and per GB**. | "NAT data processing is a top surprise on the bill." |
| **VPC endpoint** | A private door to an AWS service, skipping the internet and NAT. | "The S3 gateway endpoint is free — it's the first cost fix I'd make." |
| **Multi-AZ** | Spread across separate data centres in one region. | "Multi-AZ is the production baseline — it's availability, not disaster recovery." |
| **RDS Proxy** | A managed connection pooler in front of RDS. | "Same job as PgBouncer — too many pods, too many connections." |
| **Presigned URL** | A temporary signed link letting someone upload or download from S3 directly. | "It keeps big files out of my application entirely." |
| **Spot instance** | Cheap spare capacity AWS can reclaim with two minutes' notice. | "Great for stateless work that handles SIGTERM properly." |
| **Graviton** | AWS's own ARM processors. Cheaper per unit of work. | "About 20% better price-performance — it's just a multi-arch build." |

---

# Section A — Identity & security

## Q1. How does your FastAPI pod get AWS credentials?

### 🗣️ Say this
> "On EKS, IRSA. The chain goes like this: the cluster publishes an OIDC identity provider, which IAM is configured to trust. A Kubernetes ServiceAccount is annotated with a role ARN.
> When a pod uses that ServiceAccount, a webhook injects a signed token file and two environment variables. The AWS SDK finds those automatically, calls `AssumeRoleWithWebIdentity`, and gets temporary credentials that it refreshes on its own.
> The IAM role's trust policy checks both the OIDC issuer *and* the specific service account name, so only that one service account can assume it.
> The point worth landing is that no credential is ever stored anywhere. There's no access key in a Secret, nothing to rotate, nothing to accidentally log."

### 💻 The two halves
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-user-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/gitlab-user-service
```
```json
{
  "Effect": "Allow",
  "Principal": {"Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLE"},
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {"StringEquals": {
    "...:sub": "system:serviceaccount:prod:gitlab-user-service",
    "...:aud": "sts.amazonaws.com"
  }}
}
```
```
What the pod actually sees:
  AWS_ROLE_ARN=arn:aws:iam::123456789012:role/gitlab-user-service
  AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

$ aws sts get-caller-identity
{ "Arn": "arn:aws:sts::123456789012:assumed-role/gitlab-user-service/botocore-session-..." }
```

### ⚠️ The mistake worth naming
> "Using `StringLike` with a wildcard on that `sub` condition, or leaving it out. That makes the role assumable by *every* service account in the cluster, which quietly undoes the entire point of per-pod identity. It's the single most common IRSA misconfiguration and it's invisible until someone audits it."

### ⚠️ Follow-up: "What about EKS Pod Identity?"
> "Newer and simpler to operate — an agent runs on the node, and the association is an EKS API call rather than editing a trust policy per cluster. That matters a lot at twenty clusters. IRSA is still the answer for cross-account access and anything outside EKS. For a new cluster I'd default to Pod Identity."

---

## Q2. How do secrets reach the application?

### 🗣️ Say this
> "Secrets Manager as the source of truth, pulled into the cluster by External Secrets Operator, with IRSA controlling who can read what.
> Two things I'd emphasise beyond the mechanism. First, rotation has to actually work end to end — Secrets Manager rotating an RDS password is useless if the app read it into an environment variable at startup and never re-reads it. So either the operator triggers a rolling restart, or the app re-fetches on an auth failure and retries.
> Second, the IAM policy has to be scoped to specific secret ARNs. `GetSecretValue` on `*` means one compromised service can read every secret in the account.
> And the strongest version is not having a static secret at all — RDS IAM authentication generates a short-lived token instead of a password, so there's nothing to rotate or leak."

| Approach | Verdict |
|---|---|
| Env var from a plain K8s Secret | Base64 is not encryption. Only with strict RBAC and encryption at rest |
| **External Secrets Operator** → Secrets Manager | **Good default** — one source of truth, supports rotation |
| Secrets Store CSI driver | Mounts as files, never touches etcd. Best for compliance |
| **RDS IAM auth** | **Best** — no password exists at all |

---

## Q3. Design the network for a private FastAPI service.

### 🗣️ Say this
> "Three availability zones, three subnet tiers. Public subnets hold only the load balancer and NAT gateways. Private subnets hold the application pods. Isolated subnets — with no route to a NAT at all — hold RDS and ElastiCache.
> Security groups referencing each other rather than IP ranges. The database's group allows 5432 *from the application's security group*, so it stays correct as pods come and go and you never maintain a CIDR list.
> Then the two things people miss. A **gateway VPC endpoint for S3 and DynamoDB** — it's completely free, and without it every S3 call from a private subnet goes through NAT and you pay per gigabyte on your own backups.
> And interface endpoints for the services you call constantly — ECR, Secrets Manager, CloudWatch Logs — because pulling container images through NAT on every pod start is both slow and expensive."

### ⚠️ Security group vs NACL — the trap
> "Security groups are stateful: allow it in, and the reply gets out automatically. NACLs are stateless, so if you allow inbound you must *separately* allow the outbound ephemeral port range, or return traffic is silently dropped. That silent drop is what makes NACL debugging miserable. I use security groups for nearly everything and NACLs only as a coarse subnet guardrail."

📐 **Cost note:** NAT Gateway is charged per hour *per AZ* plus per GB. Three AZs means three NAT gateways, and that hourly cost is often a big slice of a small service's bill before a single byte moves. For non-production, one shared NAT is a legitimate trade.

---

# Section B — Data services

## Q4. RDS or Aurora, and how do you configure it?

### 🗣️ Say this
> "Aurora when I need fast failover, many low-lag read replicas, or the operational features — the cloning is genuinely useful for testing a migration against production-sized data, and Aurora's Blue/Green makes major version upgrades far less frightening.
> RDS when the workload is modest and baseline cost matters, because Aurora's I/O charges can surprise you on a write-heavy workload unless you pick I/O-Optimized.
> For either, what actually matters to my service: Multi-AZ on, storage autoscaling with an alarm well before the ceiling, Performance Insights enabled, `pg_stat_statements` in the parameter group, and the slow query log turned on.
> But the thing that actually bites is connection handling."

### ⚠️ The connection story — tell it as an experience
> "The failure I'd expect on EKS is connection exhaustion. Each pod has its own pool, so twenty pods with a pool of twenty plus overflow is six hundred-plus connections. A `db.t3.medium` has a max around 400, derived from its memory.
> Then the HPA doubles your pods during the exact traffic spike that caused the problem, and you get `FATAL: sorry, too many clients already` — the database starts rejecting connections at the moment you most need it.
> The fix is RDS Proxy or PgBouncer, plus sizing the per-pod pool as total capacity divided by max replicas rather than picking a number."

```
$ psql -c "SELECT count(*), max_conn FROM pg_stat_activity,
           (SELECT setting::int max_conn FROM pg_settings WHERE name='max_connections') s
           GROUP BY max_conn;"
 count | max_conn
-------+----------
   391 |      400        <- 9 connections of headroom during a scaling event
```

---

## Q5. ElastiCache configuration decisions.

### 🗣️ Say this
> "Cluster mode *disabled* with a replica in another AZ is right for most services. You get a single primary endpoint that fails over automatically, and you keep multi-key operations and Lua scripts across keys.
> Cluster mode enabled only when the dataset genuinely exceeds one node's memory or you need to scale writes — and that's a real code change, because you have to deal with hash tags.
> Settings I'd check: `maxmemory-policy` set deliberately, automatic failover actually enabled, encryption in transit and at rest, and reserved memory so a snapshot fork doesn't push the node into swap.
> And I'd decide explicitly whether the app tolerates a Redis outage, because on ElastiCache a failover is about thirty seconds of connection errors — your client needs to handle that rather than crash."

---

## Q6. How does S3 show up in a backend service?

### 🗣️ Say this
> "Three patterns, and the most important is presigned URLs. The client uploads straight to S3 and my service never sees the bytes — so a five-gigabyte upload doesn't occupy a worker for ten minutes, doesn't hit the load balancer timeout, and doesn't need memory.
> Second, S3 event notifications into SQS. Don't have the client tell you the upload finished, because clients crash and clients lie — let S3 emit the event.
> Third, as the destination for async exports, which is how you avoid streaming a million rows through the API.
> One detail people don't know until it bites: a presigned **PUT** can't enforce a maximum size. If you need that, use a presigned **POST** with policy conditions."

### 💻
```python
@router.post("/uploads")
async def create_upload(req: UploadRequest, user=Depends(current_user)):
    key = f"uploads/{user.id}/{uuid4()}/{req.filename}"
    url = s3.generate_presigned_url(
        "put_object",
        Params={"Bucket": BUCKET, "Key": key, "ContentType": req.content_type},
        ExpiresIn=900)
    await repo.record_pending_upload(user.id, key)
    return {"upload_url": url, "key": key}
```

📐 **S3 facts worth having right:** strong read-after-write consistency for all operations since December 2020 — saying "eventually consistent" dates you badly. ~3,500 PUT/s and 5,500 GET/s **per prefix**, so spread keys. Block Public Access at account level, always.

---

## Q7. When would you use DynamoDB over RDS?

### 🗣️ Say this
> "When the access pattern is known and narrow and I need predictable single-digit-millisecond latency at a scale where Postgres would need sharding. Session stores, idempotency keys, high-volume event writes, feature flags.
> The design discipline is the opposite of relational — you model the table around the queries, not the entities. And if you later need a query you didn't plan for, your options are a secondary index or a full scan, neither cheap.
> I'd avoid it when queries are ad-hoc, or when I need rules enforced across many entities, or when the team's instinct is relational — because you get the worst of both worlds if you model DynamoDB like a relational database and then join in the application.
> Where it genuinely shines for us is idempotency keys, because TTL means the table cleans itself up."

---

# Section C — Compute & delivery

## Q8. ECS, EKS, Lambda or App Runner?

### 🗣️ Say this
> "For a steady-traffic FastAPI service, ECS Fargate or EKS. EKS if the organisation already runs Kubernetes and wants the ecosystem — HPA, ArgoCD, one toolchain across many services. ECS if you don't want to operate a control plane; it's genuinely simpler and it's underrated.
> Lambda I'd push back on for a general API, and I'd give the specific reason rather than something vague: connection pooling. A Lambda has no persistent process, so each concurrent execution opens its own database connection. A spike to five hundred concurrent executions is five hundred connections to Postgres.
> RDS Proxy exists exactly for that and it works, but at that point you've added a component to work around the model.
> Lambda is excellent for spiky event-driven work though — processing an S3 upload, handling a webhook, a scheduled job. I'd use it happily there."

| | Lambda | ECS Fargate | EKS |
|---|---|---|---|
| Ops burden | Lowest | Low | Highest |
| Cold start | Real (0.5–3s for Python) | None | None |
| DB connection pooling | ❌ Needs RDS Proxy | ✅ | ✅ |
| Cost at steady high traffic | Expensive | Good | Best |

---

## Q9. What do you configure on an ALB, and what breaks?

### 🗣️ Say this
> "Health check pointed at a readiness endpoint that actually checks dependencies. Deregistration delay slightly above your longest request, so in-flight requests finish during a deploy. And the idle timeout, which defaults to sixty seconds — that's the one that causes mysterious 504s, because if your endpoint takes ninety seconds the ALB gives up at sixty while your application happily completes the work.
> The two production gotchas I'd volunteer are keep-alive and client IP.
> Keep-alive: the ALB reuses backend connections, so if your server's keep-alive timeout is *shorter* than the ALB's idle timeout, the ALB can send a request down a connection the server is closing at that exact moment — and you get sporadic, unexplainable 502s. The rule is server keep-alive must be **longer**.
> Client IP: behind an ALB the socket peer is the load balancer, so if you rate-limit or log by IP without handling `X-Forwarded-For` properly, you either see one IP for everyone or you trust a header an attacker can forge."

### 💻 The keep-alive fix
```bash
# ALB idle timeout = 60s, so the server must be LONGER
uvicorn app.main:app --timeout-keep-alive 75
```

📐 **ALB status codes as diagnosis — knowing these distinguishes you immediately:**
| Code | Means |
|---|---|
| **502** | Target returned something malformed, or closed the connection — usually the keep-alive race |
| **503** | No healthy targets — deployment or health check problem |
| **504** | Target didn't respond within the idle timeout |
| **460** | The **client** gave up and disconnected before the target responded |

---

## Q10. Cut this service's AWS bill by 30%.

### 🗣️ Say this
> "First, look at the actual bill — Cost Explorer grouped by service then by tag. Every answer here is worthless without knowing where the money goes, and it's frequently not compute.
> Then in order of effort-to-saving: add the free S3 and DynamoDB gateway endpoints to kill NAT data processing charges. Right-size using real utilisation data, because most services are provisioned for a peak that happens twice a year. Move to Graviton, which is roughly 20% better price-performance and for Python is just a multi-arch build.
> Then Savings Plans for the steady baseline, Spot for anything stateless and interruptible, S3 lifecycle policies, and CloudWatch Logs retention — the default is *never expire*, and log ingestion is genuinely expensive.
> And the one that isn't a line item: get cost-per-request onto a dashboard. Cost is an engineering metric, and if nobody sees it, it only gets addressed during a panic."

⚠️ **Easy wins people forget:** expire incomplete multipart uploads (they accumulate invisibly and are billed), delete unattached EBS volumes and old snapshots, and shut down non-production environments overnight.

---

## Q11. Design for high availability.

### 🗣️ Say this
> "Multi-AZ is the baseline and it's almost free architecturally — three AZs, pods spread with topology constraints, RDS Multi-AZ, ElastiCache with a cross-AZ replica. That survives losing an AZ with a brief blip, and that's what 'production' should mean.
> Multi-region is a completely different conversation and I'd want the requirement in writing before agreeing to it. Active-passive with cross-region replicas and Route 53 failover is achievable, but the database promotion is a decision you can't take back. Active-active means multi-master writes or partitioning users by region, which is a data-model decision, not infrastructure.
> The cost is roughly double and the complexity is permanent.
> And the point I'd actually make: most outages are self-inflicted — a bad deploy, a bad config, a bad migration. Multi-region helps with none of those. Deployment safety and tested restores buy more real availability per pound than another region."

📐 AZ failure → multi-AZ handles it. Region failure → needs multi-region. **Bad deploy or accidental deletion → neither helps**, and those are far more common.

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| SDK credential chain order? | "Environment variables, then the credentials file, then container/web-identity, then the instance profile. IRSA lands in the web-identity step." |
| Why is IMDSv2 required? | "Version 1 was a simple GET, so an SSRF bug in your app could steal the node's credentials. v2 needs a PUT to get a token with a hop limit. Enforce it." |
| ALB vs NLB? | "ALB is layer 7 — path and host routing, HTTP-aware. NLB is layer 4 — extreme throughput, static IPs, preserves the source IP, TLS passthrough." |
| Parameter Store or Secrets Manager? | "Parameter Store is cheaper with no built-in rotation. Secrets Manager costs per secret but has native rotation and cross-region replication." |
| Step Functions when? | "Orchestrating multi-step workflows with retries, branching and long waits — it's a Saga orchestrator you don't have to write." |
| X-Ray or OpenTelemetry? | "X-Ray is the AWS backend; OTel is the vendor-neutral instrumentation standard and can export to X-Ray. Instrument with OTel so the backend stays swappable." |
| Fargate vs EC2 nodes? | "Fargate: no node management, per-pod billing, but no DaemonSets and slower pod start. EC2: cheaper at scale, full control, you patch the nodes." |
| RDS major version upgrade? | "RDS Blue/Green Deployments — it builds a synced green environment and you switch over in about a minute with safeguards." |
| KMS in one sentence? | "Managed key custody. Watch the request cost on high-volume paths — use data key caching." |

---

**Next:** [07-deployments-and-release.md](07-deployments-and-release.md)
