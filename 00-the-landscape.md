# 00 — The Landscape

> The orientation doc. No Terraform to apply, nothing to master yet. The goal is simple: **see the whole system once**, end to end, using a deployment you already run in production — the Workforce Rails API. By the end you'll be able to name every big box, say what it does in a sentence, and know which lesson goes deep on it. That's it. Keep it light.

---

## What this course gives you

You run a Rails + Sidekiq app in production on AWS today. Someone else wrote the Terraform; you've edited it nervously. This course closes that gap. By the end you'll be able to:

- Look at any `resource "aws_..."` block and know what AWS thing it creates and *why*.
- Explain the whole deployment to another backend engineer in 20 minutes.
- Direct an AI agent to write correct infra — and catch it when it writes something insecure or expensive.

We're **not** learning Terraform the language. Terraform is just how we'll write things down. We're learning what each AWS piece *is* and how the pieces connect.

The course walks **one real architecture**, box by box, in the order the boxes depend on each other: network first, then the servers that live in the network, then how traffic gets in, then the data and supporting services. This doc is the map; each later doc zooms into one box.

---

## The system we're going to learn (your real one)

This is the Workforce Rails API production stack — your actual deployment, simplified to the boxes that matter. Don't study it yet. Just look at the *shape*.

```
                         Internet
                            │
                   api.workforce.ensido.com   ← DNS (Route 53) + TLS cert (ACM)
                            │  HTTPS
   ┌────────────────────────▼──────────────────────────────────┐
   │  VPC  10.0.0.0/16   (us-east-2)   ← your private network   │
   │                                                            │
   │  PUBLIC subnets (3 AZs)        PRIVATE subnets (3 AZs)     │
   │  ┌──────────────────┐                                      │
   │  │  ALB :80 → :443  │  ← load balancer, the only public    │
   │  └────────┬─────────┘    thing; redirects HTTP to HTTPS    │
   │           │ forwards to port 3000                          │
   │  ┌────────▼───────────────────────────────────────┐       │
   │  │  ECS Fargate (your containers, no servers)      │       │
   │  │   • api  × 2   (Rails web, port 3000)           │       │
   │  │   • worker × 1 (Sidekiq, IS_WORKER=1)           │       │
   │  └───────┬──────────────────────┬──────────────────┘       │
   │          │                      │                          │
   │  ┌───────▼────────┐     ┌───────▼─────────┐                │
   │  │ RDS PostgreSQL │     │ ElastiCache     │                │
   │  │ 15 (Multi-AZ)  │     │ Redis 7.1       │                │
   │  └────────────────┘     └─────────────────┘                │
   │                                                            │
   │  outbound to internet (pull image, call webhooks):         │
   │     private subnets ──► NAT Gateway ──► Internet Gateway    │
   └────────────────────────────────────────────────────────────┘

   Around the edges (cross-cutting, not "inside" a subnet):
     ECR        → where your Docker image lives so ECS can pull it
     S3         → file storage (uploads, backups)
     IAM        → which identity may call which AWS API
     CloudWatch → logs + metrics from every container
     ACM        → the HTTPS certificate
```

That's the entire course. Fifteen lessons, one box each (plus the connections between them). If you can hold this picture, you're ready.

---

## Follow one request through the stack

The fastest way to "get" the architecture is to trace a single request, the same way you'd trace a Rails request through middleware. A user hits `GET https://api.workforce.ensido.com/employees`:

1. **DNS (Route 53)** turns `api.workforce.ensido.com` into the load balancer's address. → *lesson 09*
2. The request arrives at the **ALB** in a **public subnet**. The ALB terminates HTTPS using a certificate from **ACM**, then forwards the request inward to a healthy Rails container on port 3000. → *lessons 08, 09*
3. The ALB only forwards to containers that pass a **health check** (`GET /up`). That's the **target group** doing its job — same idea as a connection pool only handing out live connections. → *lesson 08*
4. The request reaches a Rails **container** running on **ECS Fargate** in a **private subnet** — no public IP, unreachable from the internet directly. → *lesson 07*
5. That container is *allowed* to receive traffic on port 3000 only from the ALB, because a **security group** says so. → *lesson 04*
6. Rails queries **RDS PostgreSQL** and reads/writes **ElastiCache Redis**. Both sit in private subnets and only accept connections from your app's security group. → *lessons 12, 13*
7. If Rails needs to read a file, it talks to **S3**. → *lesson 14*
8. The whole time, the container's logs stream to **CloudWatch**. → *lesson 15*

A **Sidekiq job** is the same path minus the ALB: the `worker` container pulls a job off the Redis queue and runs it, hitting RDS/S3 as needed.

And when the container needs to reach *out* — pull its image at startup, or call a third-party webhook — it goes through the **NAT Gateway** to the **Internet Gateway**, because private subnets have no direct internet path. → *lesson 03*

That's it. Every lesson is just "zoom into step N."

---

## The big components, one light pass each

A quick reference card per box: **what it is**, **where you use it**, **roughly what it costs**, **how you express it in Terraform**, and **how it actually works** — all shallow on purpose. The full treatment is in the linked lesson. Numbers are from your real Workforce production stack so they're recognizable.

### VPC — your private network → *lesson 01*
- **What:** an isolated private network in one region (`10.0.0.0/16`, `us-east-2`). Everything else lives inside it.
- **Where:** the outermost box; you make one per project/environment.
- **Cost:** free.
- **Terraform:** `aws_vpc`.
- **How it works:** you reserve a range of private IPs; nothing inside is reachable from outside until you build a path.

### Subnets & AZs — rooms in the network → *lesson 02*
- **What:** slices of the VPC's IP range, each in one Availability Zone. Public subnets (can reach the internet) and private subnets (can't, directly). Yours: 3 public + 3 private `/20` subnets across 3 AZs.
- **Where:** ALB goes in public; everything else (containers, DB, Redis) in private.
- **Cost:** free.
- **Terraform:** `aws_subnet`.
- **How it works:** spreading across 3 AZs means one datacenter failing doesn't take you down.

### Routing & gateways — the doors → *lesson 03*
- **What:** route tables decide where traffic goes; the **Internet Gateway** is the public door; the **NAT Gateway** lets private things reach out without being reachable.
- **Where:** one IGW per VPC; NAT in a public subnet so private subnets can egress.
- **Cost:** IGW free; **NAT Gateway ~$32/month + data charges — the famous cost trap.**
- **Terraform:** `aws_route_table`, `aws_internet_gateway`, `aws_nat_gateway`.
- **How it works:** a subnet is "public" only because its route table points at an IGW. Privacy is a routing fact, not a checkbox.

### Security groups — the firewall → *lesson 04*
- **What:** a stateful, allow-only firewall attached to each resource. Yours: ALB allows 80/443 from anywhere; app allows 3000 from the ALB only; RDS allows 5432 from the app only; Redis allows 6379 from the app only.
- **Where:** on every networked resource.
- **Cost:** free.
- **Terraform:** `aws_security_group`, `aws_security_group_rule`.
- **How it works:** rules reference *other security groups*, not IPs — "allow the app, wherever it happens to be." Like a `before_action` that checks where a request came from.

### IAM — who can call which API → *lesson 05*
- **What:** the identity system. A separate plane from security groups: SGs decide if a *packet* can travel; IAM decides if an *API call* is allowed.
- **Where:** every task runs with a role (e.g. `WorkforceProductionECSTaskExecutionRole`) granting exactly what it needs — pull from ECR, write logs, read a secret.
- **Cost:** free.
- **Terraform:** `aws_iam_role`, `aws_iam_policy`.
- **How it works:** a task "assumes" a role and inherits its permissions. Like Pundit for AWS API calls.

### ECR — your image registry → *lesson 06*
- **What:** AWS's private Docker registry. Your built image (`workforce-production`) lives here.
- **Where:** CI pushes here; ECS pulls from here at task start.
- **Cost:** ~$0.10/GB-month storage. Tiny.
- **Terraform:** `aws_ecr_repository`.
- **How it works:** ECS authenticates (via its IAM role) and pulls the `:latest` image when starting a container.

### ECS + Fargate — your containers → *lesson 07*
- **What:** the orchestrator that runs and keeps your containers alive. **Fargate** = no servers to manage. Yours: `api` ×2 (512 CPU / 1024 MB), `worker` ×1.
- **Where:** the heart of the system, in private subnets.
- **Cost:** pay per vCPU/memory per second a task runs. Two small api + one worker ≈ low tens of dollars/month.
- **Terraform:** `aws_ecs_cluster`, `aws_ecs_task_definition`, `aws_ecs_service`.
- **How it works:** a **task definition** is the blueprint (image, CPU/mem, env vars, ports); a **service** keeps N copies running and replaces dead ones — like a supervisor for your web/worker pool.

### ALB — getting traffic in → *lesson 08*
- **What:** an HTTP load balancer. Yours: listens on 80 (redirect → 443) and 443 (forward to containers on 3000); health-checks `/up`.
- **Where:** public subnets; the only internet-facing piece.
- **Cost:** ~$16/month + small per-request/data charges.
- **Terraform:** `aws_lb`, `aws_lb_listener`, `aws_lb_target_group`.
- **How it works:** a **target group** tracks which containers are healthy and load-balances across them — conceptually a connection pool that only hands out live connections.

### Route 53 + ACM — DNS & HTTPS → *lesson 09*
- **What:** **Route 53** maps your domain to the ALB; **ACM** issues/renews the TLS cert (`*.api.workforce.ensido.com`).
- **Where:** in front of the ALB.
- **Cost:** Route 53 ~$0.50/zone/month; ACM certs free.
- **Terraform:** `aws_route53_record`, `aws_acm_certificate`.
- **How it works:** ACM validates you own the domain via DNS, then auto-renews forever; the ALB uses the cert to terminate HTTPS.

### Service discovery (Cloud Map) → *lesson 10*
- **What:** lets services find each other by name instead of hardcoded IPs. (Relevant once you have service-to-service calls — e.g. the Rails API and the Workforce Agent talking.)
- **Where:** between internal services.
- **Cost:** small.
- **Terraform:** `aws_service_discovery_service`.
- **How it works:** registers a DNS name like `payments.internal` that resolves to current task IPs.

### Secrets Manager → *lesson 11*
- **What:** where secrets (`DATABASE_URL`, API keys, `RAILS_MASTER_KEY`) live so they're injected at runtime, not baked into images.
- **Where:** referenced by task definitions.
- **Cost:** ~$0.40/secret/month.
- **Terraform:** `aws_secretsmanager_secret`.
- **How it works:** the task's IAM role grants read access; AWS injects the value as an env var at container start.

### RDS — managed PostgreSQL → *lesson 12*
- **What:** managed Postgres you don't patch or back up by hand. Yours: PG 15, `db.t4g.small`, Multi-AZ, 30-day backups.
- **Where:** private subnets; reachable only from your app's security group.
- **Cost:** the instance runs 24/7 — a small Multi-AZ instance ≈ low tens of dollars/month, the biggest steady cost after compute.
- **Terraform:** `aws_db_instance`, `aws_db_subnet_group`.
- **How it works:** AWS runs a standby in another AZ and fails over automatically; you connect via a DNS endpoint.

### ElastiCache — managed Redis → *lesson 13*
- **What:** managed Redis. Yours: 7.1, `cache.t4g.micro`, used for caching + the Sidekiq queue.
- **Where:** private subnets; reachable only from your app.
- **Cost:** a micro node ≈ ~$12/month.
- **Terraform:** `aws_elasticache_cluster` / `aws_elasticache_replication_group`.
- **How it works:** you connect via `redis://<endpoint>:6379`, same as a Redis box you'd run yourself — minus the running it.

### S3 — object storage → *lesson 14*
- **What:** file storage for uploads, backups, pipeline artifacts. Private by default.
- **Where:** reached from your app; optionally via a **VPC endpoint** so traffic never leaves AWS.
- **Cost:** ~$0.023/GB-month + request charges. Cheap.
- **Terraform:** `aws_s3_bucket`, `aws_s3_bucket_policy`.
- **How it works:** objects in buckets, accessed by API; **block public access** unless you truly mean to publish.

### CloudWatch — observability → *lesson 15*
- **What:** logs, metrics, alarms. Yours: each service logs to `/ecs/workforce-production-*` (14-day retention), plus Container Insights.
- **Where:** every container ships logs here; metrics from ALB, RDS, ECS land here.
- **Cost:** pay per GB ingested + stored; modest with sane retention.
- **Terraform:** `aws_cloudwatch_log_group`, `aws_cloudwatch_metric_alarm`.
- **How it works:** the task's log driver streams stdout to a log group — this is how you see a crashing Fargate task instead of guessing.

---

## The supporting cast (named now, not lessons)

Your real stack has a few more pieces you should *recognize* but that aren't core lessons. They're built on the same ideas, so they'll make sense once you finish the course:

- **CI/CD (CodePipeline + CodeBuild):** Bitbucket → build Docker image → push to ECR → rolling deploy to ECS. The pipeline is just orchestration around boxes you'll already understand (ECR, ECS, IAM, S3 for artifacts).
- **Terraform state (S3 + DynamoDB):** where Terraform records what it created. Plumbing for Terraform itself, not AWS architecture.
- **KMS:** encryption-at-rest keys, mostly transparent (RDS, S3).
- **SES:** transactional email.
- **AWS Chatbot:** Slack deploy notifications.

You'll see these in your repo; just know they sit *around* the core architecture, not inside the request path. We may add a short appendix later if you want CI/CD covered explicitly.

> There are ~250 AWS services. You need maybe 15 to run a Rails app well. This course teaches those 15. Everything else you can learn on demand once these mental models are in place.

---

## The one idea to carry into every lesson

If you keep only one thing from this doc, keep this: **there are two independent security systems, and they fail differently.**

- **Security groups** decide whether a *network packet* can reach a port. (lesson 04)
- **IAM** decides whether an *identity* can call an AWS *API*. (lesson 05)

Your app can have perfect network access to S3 and still get `AccessDenied` (IAM said no). Or have permission to read a secret and still time out (no network path). When something's broken, the first question is always: *is this a network problem (security group / routing) or a permission problem (IAM)?* Almost every infra bug you'll hit is one or the other — and people from the app layer constantly fix the wrong one.

Everything else is details that hang off the picture above.

---

## How each lesson works

Every lesson from 01 on follows the same shape so you always know where to look:

1. What you're learning & why (+ a short terms list) · 2. Mental model · 3. Concepts · 4. **Cost & blast radius** · 5. The Terraform (annotated) · 6. **How Workforce does it** · 7. How to use this doc with an agent · 8. Checkpoints · 9. Footguns · 10. Ask-the-agent cheatsheet · 11. Where this goes next.

The **How Workforce does it** section ties each concept back to your real production stack (with pointers into `INFRASTRUCTURE.md`), so you always see the generic idea *and* the concrete instance you already run. Where Workforce doesn't use a feature yet, that section says so explicitly rather than inventing one.

---

## Working with the agent (how you'll actually build this)

You won't hand-write the Terraform — you'll drive an agent that has a terminal. In practice it does two concrete things for you:

1. **Inspects what already exists.** The agent can run AWS CLI commands to read your live account — `aws ec2 describe-vpcs`, `aws ecs describe-services`, `aws elbv2 describe-target-groups`, etc. — so it builds on reality instead of guessing. Use this constantly: *"Check what VPCs and subnets already exist in `us-east-2` before proposing anything."*
2. **Writes the Terraform.** It creates and edits the `.tf` files and can run `terraform plan` / `apply`. **But it must explain and get your confirmation before applying** — that's the rule, not a nicety.

The loop you'll repeat for every change:

> **Inspect → Propose → Explain → You confirm → Apply → Verify.**
>
> 1. Ask it to inspect current state with the CLI.
> 2. Ask it to write the Terraform for the change.
> 3. Make it run `terraform plan` and walk you through *what this creates/changes, what it costs idle per month, and what breaks if a CIDR / security group / IAM rule is wrong*. If it can't explain a line, don't apply.
> 4. **You** confirm — explicitly. The agent never applies on its own.
> 5. It applies.
> 6. It verifies with the CLI (re-describe the resource) and answers the security question: *"What's now reachable from the internet that wasn't? What identity can now call what API?"*

Two guardrails that save you:

- **Docs outrank the agent.** Concepts → [AWS docs](https://docs.aws.amazon.com/). Resource arguments → [Terraform AWS registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs). Agents invent arguments and reach for insecure defaults (`0.0.0.0/0` ingress, public buckets, `Action: "*"`) to "make it work." The docs and the `plan` output are the judges, not the agent's confidence.
- **You own the architecture.** The agent writes the code; if you can't say why a resource exists and what breaks without it, you're not ready to confirm the apply.

---

## Where this goes next

**→ 01 — VPC Fundamentals.** We start at the outermost box: your `10.0.0.0/16` network. What it isolates, how to read that CIDR notation without fear, and why it's the one decision that's genuinely hard to change later. Everything else in this map lives inside it.
