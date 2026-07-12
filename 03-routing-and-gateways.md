# 03 — Routing & Gateways

## 1. What you're learning and why it matters

In lesson 02 you *labeled* subnets public and private, but that was intention, not fact — you even saw the punchline: "public vs private is a routing decision, not a flag." This lesson builds the actual doors. You'll learn how a **route table** decides where a subnet's traffic goes, how an **Internet Gateway** turns a subnet public, and how a **NAT Gateway** lets your private tasks reach *out* to the internet (to pull a Docker image, call a Stripe webhook) *without* becoming reachable *from* it. This is the lesson where your architecture finally *works* end to end — and it's also where the single most infamous AWS cost surprise lives (the NAT Gateway). Get routing right and everything downstream just connects; get it wrong and you either have an unreachable app or an accidentally exposed database.

### Terms first

- **Route table** — an ordered list of rules ("traffic to *this* destination goes *there*"). Attached to subnets. The thing that makes a subnet public or private.
- **Route** — one rule: a destination CIDR (e.g. `0.0.0.0/0`) → a target (a gateway).
- **`0.0.0.0/0`** — "everywhere / the whole internet." The default route; where traffic goes when no more specific rule matches.
- **Internet Gateway (IGW)** — the VPC's two-way door to the public internet. One per VPC, free.
- **NAT Gateway** — a one-way door: lets private subnets initiate *outbound* internet connections but blocks unsolicited *inbound*. Managed by AWS; **costs money hourly + per GB.**
- **NAT (Network Address Translation)** — rewriting a private source IP to a public one so return traffic can find its way back. What the NAT Gateway does.
- **Elastic IP (EIP)** — a static public IP you own; a NAT Gateway needs one.
- **Route table association** — the link that attaches a route table to a specific subnet.

---

## 2. Mental model

> **The route table is the subnet's signpost at every intersection: "for the local network go here, for the internet go there." An Internet Gateway is a two-way street door (public). A NAT Gateway is a one-way exit — your private tasks can walk out to the internet, but nobody can walk in.**

---

## 3. Concept sections

### 3.1 How a route table actually decides

Every packet leaving a subnet is matched against that subnet's route table, and AWS picks the **most specific matching route** (longest-prefix match). There's always one built-in route you can't remove: the **local** route covering the whole VPC CIDR.

```
Route table for a PUBLIC subnet:
┌──────────────┬─────────────────────┐
│ Destination  │ Target              │
├──────────────┼─────────────────────┤
│ 10.0.0.0/16  │ local               │ ← built-in: talk to anything in the VPC
│ 0.0.0.0/0    │ igw-xxxx (IGW)      │ ← everything else → the internet
└──────────────┴─────────────────────┘
```

A packet to `10.0.5.10` (inside the VPC) matches `local` — stays internal. A packet to `52.x.x.x` (some public API) matches only `0.0.0.0/0` → goes to the IGW. **That one `0.0.0.0/0 → IGW` line is the entire difference between public and private.** Remove it and the subnet can still talk internally but has no way to the internet.

> **Rails analogy:** A route table is like your `routes.rb` matched most-specific-first. `local` is a catch-all internal route that's always there; `0.0.0.0/0` is the fallback `match "*"` at the bottom. Change where that fallback points and you change where "everything else" goes.

### 3.2 The Internet Gateway makes a subnet public

An **IGW** is a horizontally-scaled, no-bandwidth-limit, *free* component you attach to your VPC — one per VPC. By itself it does nothing; a subnet only becomes public when its **route table points `0.0.0.0/0` at the IGW** *and* the resource has a public IP (recall `map_public_ip_on_launch` from 02 §3.6). Both conditions matter:

```
Public subnet + IGW route + public IP  → reachable from internet (e.g. the ALB)
Public subnet + IGW route + NO public IP → can't be reached (no address to hit)
Private subnet (no IGW route)           → no internet path at all, regardless of IP
```

This is why the ALB lives in public subnets with public IPs, and your Fargate tasks live in private subnets with none.

### 3.3 The NAT Gateway: outbound-only for private subnets

Your private Fargate tasks have no internet route — good for security, but they still need to reach *out*: pull the container image from ECR (if not using VPC endpoints), hit a payment provider, deliver a webhook, call an external API. The **NAT Gateway** solves exactly this. It sits in a **public** subnet, and private subnets route their `0.0.0.0/0` to *it* instead of the IGW:

```
Private subnet route table:
┌──────────────┬─────────────────────┐
│ Destination  │ Target              │
├──────────────┼─────────────────────┤
│ 10.0.0.0/16  │ local               │
│ 0.0.0.0/0    │ nat-xxxx (NAT GW)   │ ← outbound → NAT (which sits in a public subnet)
└──────────────┴─────────────────────┘

Flow of an outbound call from a private task:
  task (10.0.128.7) ──► NAT GW (public subnet) ──► IGW ──► internet
  return traffic     ◄── NAT rewrites addresses so replies find their way back
```

The key asymmetry: a private task can **initiate** a connection outward and receive the reply, but the internet **cannot initiate** a connection inward. That's the "one-way exit door." It's how you let Sidekiq deliver a webhook while keeping the worker itself completely unreachable from outside.

### 3.4 The NAT Gateway cost trap

This is the one to burn into memory, because it's the most common "why is my AWS bill $X" story from app teams. A NAT Gateway costs **two ways**:

1. **~$0.045/hour just for existing** ≈ **~$32/month per NAT Gateway**, whether it moves one byte or not.
2. **~$0.045 per GB processed** — every gigabyte your tasks pull through it.

The traps that stack up:
- **One NAT per AZ for "high availability"** → 3 AZs = 3 NAT Gateways ≈ **~$96/month** before any traffic. Many teams don't need per-AZ NAT.
- **Chatty egress** — pulling large images, streaming from external APIs, or (worst) reaching **S3/other AWS services *through* the NAT** when a **VPC endpoint** would keep that traffic off the NAT entirely and often free (lesson 14).
- **Forgetting it exists** in a dev/staging environment that's otherwise idle — it bills 24/7.

```
Cost sanity check:
  1 NAT GW, low traffic:     ~$32/mo    ← Workforce prod today
  3 NAT GW (per-AZ), idle:   ~$96/mo    ← often overkill
  Heavy S3-via-NAT egress:   $32 + $$$  ← fix with an S3 VPC endpoint (lesson 14)
```

**Rule of thumb:** one NAT Gateway is usually fine to start; add per-AZ NAT only when an AZ-isolated egress failure is a real risk you're willing to pay for; and route AWS-service traffic (S3, ECR, Secrets Manager) through **VPC endpoints** instead of the NAT wherever you can.

### 3.5 The public path vs the private path, side by side

Putting 02 and 03 together, here's the whole traffic story for the Workforce-style stack:

```
INBOUND (a user hits your API):
  internet ──► IGW ──► ALB (public subnet, public IP) ──► Fargate task (private subnet)
             the ALB is the only thing with a door facing the street

OUTBOUND (a task calls a webhook / pulls an image):
  Fargate task (private) ──► NAT GW (public subnet) ──► IGW ──► internet
             the task reaches out; nothing reaches in

NEVER:
  internet ──X──► Fargate task directly   (no public IP, no inbound route)
  internet ──X──► RDS / Redis             (private subnet, no internet route)
```

Two gateways, two directions. The IGW carries the ALB's inbound public traffic; the NAT carries the private tier's outbound-only traffic. Your data tier touches neither.

### 3.6 One route table per behavior, associated to the right subnets

Route tables aren't per-subnet by nature — you create a route table that expresses a *behavior* (e.g. "public: send `0.0.0.0/0` to the IGW") and **associate** it with every subnet that should behave that way. Typical minimal setup:

- **One public route table** (`0.0.0.0/0 → IGW`), associated with all public subnets.
- **One private route table** (`0.0.0.0/0 → NAT`), associated with all private subnets — *or* one per AZ if you run per-AZ NAT Gateways (each private subnet points at the NAT in its own AZ).

A subnet with **no explicit association** falls back to the VPC's **main route table** — a subtle footgun: if the main route table happens to have an IGW route, an "unassociated" subnet you thought was private is actually public. Always associate explicitly.

---

## 4. Cost & blast radius

**Cost:**
- **Route tables, routes, associations, Internet Gateway:** **free.** ✅
- **NAT Gateway:** **~$32/month each** + **~$0.045/GB** processed. 💸 The one line item in this whole networking block that actually costs real money, and it scales with both count (per-AZ) and traffic.
- **Elastic IP for the NAT:** free while attached to a running NAT; a stray unattached EIP costs a small hourly fee.

**Blast radius:**
- **Missing `0.0.0.0/0 → IGW` on the public route table:** ALB/public resources can't reach or be reached — your app looks "down" though everything's healthy. *Severity: high (outage), quick fix.*
- **Accidental `0.0.0.0/0 → IGW` on a "private" subnet's route table (or via the main table):** your private tasks/DB are now internet-exposed. *Severity: critical (security).*
- **No NAT route on private subnets:** private tasks can't pull images or call external services — deploys fail, webhooks don't send, and it looks like a DNS/timeout bug. *Severity: high, confusing.*
- **NAT sprawl (per-AZ NAT everywhere by default):** silent 2–3× cost with no benefit for many workloads. *Severity: medium (money).*
- **S3/AWS traffic routed through NAT instead of a VPC endpoint:** per-GB charges that a free/cheap endpoint would erase. *Severity: medium (money), fixed in lesson 14.*

Summary: **routing is free and the highest-leverage security surface — one wrong `0.0.0.0/0` line exposes everything.** The NAT Gateway is the one component here you must watch on the bill.

### Blast-radius walkthrough: the "accidental exposure" bug

```
Someone adds this to the PRIVATE subnets' route table "to fix a connectivity issue":
  0.0.0.0/0 → igw-xxxx        ← WRONG target (IGW instead of NAT)

Result:
  • Private Fargate tasks that have a public IP → now reachable from the internet
  • If a task or RDS had map_public_ip or a public IP for any reason → exposed
  • Security groups become the ONLY thing standing between the internet and your app
  • You've silently converted "defense in depth" into "one misconfigured SG from breach"

The fix: private subnets route 0.0.0.0/0 to the NAT Gateway, never the IGW.
```

This is the concrete reason 02 kept insisting "follow the route table."

---

## 5. The Terraform

Routing is where the public/private labels from lesson 02 become behavior. Annotated, minimal, one-NAT setup.

```hcl
# --- The two-way public door: one IGW per VPC (free). ---
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "main-igw" }
}

# --- PUBLIC route table: send "everywhere" to the IGW. This single
#     route is what makes the associated subnets public (§3.1–3.2). ---
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"                       # §3.1 — the default route
    gateway_id = aws_internet_gateway.igw.id       # §3.2 — → IGW = public
  }
  tags = { Name = "public-rt" }
}

# Associate the public route table with each public subnet (§3.6).
resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}
resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# --- The one-way exit: NAT Gateway lives in a PUBLIC subnet and needs
#     a static public IP. THIS is the ~$32/mo + per-GB component (§3.4). ---
resource "aws_eip" "nat" {
  domain = "vpc"
  tags   = { Name = "nat-eip" }
}
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id           # NAT sits in a PUBLIC subnet
  tags          = { Name = "main-nat" }
  depends_on    = [aws_internet_gateway.igw]        # NAT egress needs the IGW first
}

# --- PRIVATE route table: send "everywhere" to the NAT, NOT the IGW.
#     Outbound-only. This is the line that keeps private tasks unreachable
#     from the internet while still able to reach out (§3.3). ---
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"                    # §3.3 — default route...
    nat_gateway_id = aws_nat_gateway.nat.id         # ...→ NAT = outbound-only
  }
  tags = { Name = "private-rt" }
}

# Associate the private route table with each private subnet (§3.6).
# (With one shared NAT, all private subnets can point at the same table.)
resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private.id
}
resource "aws_route_table_association" "private_b" {
  subnet_id      = aws_subnet.private_b.id
  route_table_id = aws_route_table.private.id
}
```

The whole public/private distinction reduces to **which target the `0.0.0.0/0` route points at**: IGW (public, two-way) or NAT (private, outbound-only). Everything else is wiring.

📎 Concepts: [Route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) · [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) · [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
📎 Resources: [`aws_route_table`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table) · [`aws_internet_gateway`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway) · [`aws_nat_gateway`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway)

> **For the agent:** *"Before applying, tell me the monthly cost of every NAT Gateway in this plan and whether any AWS-service traffic (S3/ECR) could be routed through a VPC endpoint instead."* Make it cost-check NAT every single time.

---

## 6. How Workforce does it

From `production/modules/vpc`, region `us-east-2`:

| Decision | Workforce value | Maps to |
|----------|-----------------|---------|
| Internet Gateway | `workforce-production-igw`, one per VPC | §3.2 — the public door |
| NAT Gateway | **1** NAT + 1 Elastic IP, in a public subnet | §3.3, §3.4 — single NAT, cost-conscious |
| Public route | `0.0.0.0/0 → IGW` | §3.1–3.2 — makes public subnets public (ALB) |
| Private routes | `3 × 0.0.0.0/0 → NAT` | §3.3, §3.6 — all three private subnets egress via the one NAT |

**What to notice and question:**
- **One NAT Gateway, not three.** Workforce runs a single NAT (~$32/mo) shared by all three private AZs rather than one-per-AZ (~$96/mo). The tradeoff: if the NAT's AZ fails, *outbound* internet from private tasks in the other AZs breaks (inbound via the ALB still works, since the ALB is multi-AZ). For this workload that's an accepted, sensible cost tradeoff — worth being able to explain it as a *deliberate* choice, not an oversight.
- Note the private route table has **3** entries because there are 3 private subnets to associate — but they all point at the same NAT. If Workforce later wanted per-AZ NAT resilience, this is exactly the spot that changes (one NAT + one private route table per AZ).
- Compare with the **Workforce Agent production** environment, which drops the NAT Gateway entirely and uses **VPC endpoints** (S3 gateway + interface endpoints for ECR, Secrets Manager, logs) instead — a different, often cheaper egress strategy you'll fully understand after lesson 14. Same problem (private tasks need to reach AWS services), different solution (endpoints vs NAT).

> Source: `workforce-api-infra/INFRASTRUCTURE.md` → *Networking — VPC* (routes) and the Agent project's *VPC Endpoints* section.

---

## 7. How to use this doc with an agent

1. **Build:** *"Create an Internet Gateway, a public route table (`0.0.0.0/0 → IGW`) associated with my public subnets, one NAT Gateway with an EIP in a public subnet, and a private route table (`0.0.0.0/0 → NAT`) associated with my private subnets. Explain which single line makes each subnet public or private."*
2. **Probe:** *"Trace two packets for me: (a) a user request from the internet to my ALB, and (b) an outbound webhook from a private Fargate task. Name every route-table lookup and gateway each one crosses, and show why the internet can't initiate a connection to the task."*
3. **Quiz:** *"Ask me: what makes a subnet public? what's the difference between an IGW and a NAT? what breaks if a private subnet's default route points at the IGW instead of the NAT? and how much does an idle 3-AZ NAT setup cost? Withhold answers until I respond."*
4. **Refactor:** *"I have 3 NAT Gateways (one per AZ). Analyze whether I actually need per-AZ NAT for my workload, what I'd save by dropping to one, what availability I'd lose, and whether any of my NAT traffic should move to VPC endpoints."*

---

## 8. Checkpoints

1. What single route-table entry is the entire difference between a public and a private subnet?
2. How does the route table choose between the `local` route and `0.0.0.0/0`? (What rule?)
3. What does a NAT Gateway let a private task do, and what does it *prevent*?
4. Roughly what does one NAT Gateway cost per month idle, and name two ways NAT costs balloon.
5. Where does a NAT Gateway itself have to live, and why (which subnet type)?
6. A subnet with no route-table association uses which route table — and why is that a footgun?

---

## 9. Footguns

- **`0.0.0.0/0 → IGW` on a private subnet.** Instantly exposes anything in it with a public IP. Private subnets route to the **NAT**, never the IGW.
- **Forgetting to associate route tables.** Unassociated subnets fall back to the main route table — which may not do what you assume. Associate explicitly, every subnet.
- **No NAT route on private subnets.** Deploys fail to pull images, webhooks silently don't send; looks like a timeout/DNS bug, is actually "no egress path."
- **Per-AZ NAT by reflex.** 2–3× the cost for HA many workloads don't need. Decide deliberately.
- **Routing S3/ECR/Secrets traffic through the NAT.** Pays per-GB for what a VPC endpoint does cheaper/free (lesson 14).
- **Stray Elastic IPs.** An EIP not attached to a running resource bills hourly. Clean them up.
- **NAT in a private subnet.** It can't work there — the NAT needs the IGW path, so it must sit in a public subnet.

---

## 10. Ask-the-agent cheatsheet

- *"Audit every route table in this VPC: for each, is it public or private, and does any 'private' table accidentally route to the IGW?"*
- *"List all NAT Gateways with their monthly base cost, and estimate per-GB charges from my egress. Flag any that could be replaced by VPC endpoints."*
- *"Show the full inbound and outbound path for a request to my ALB and an outbound call from a private task — every gateway and route hop."*
- *"Am I running per-AZ NAT? If I consolidated to one, what's the cost saving and the availability tradeoff?"*
- *"Which subnets have no explicit route-table association, and what route table are they actually using?"*

---

## 11. Where this goes next

**→ 04 — Security Groups & NACLs.** Routing decides whether a path *exists*; security groups decide whether traffic is *allowed* along it. You'll add the stateful firewall layer that lets the ALB talk to your tasks on 3000, your tasks talk to RDS on 5432, and nothing else — the second half of "the app is reachable, the database is not." Remember from §4's exposure walkthrough that when routing is loose, security groups become your *only* defense; lesson 04 is about not relying on that.

Leans on this lesson: **04** (SGs, the allow-layer on top of routing), **07/08** (the ALB in public subnets uses the IGW; tasks in private subnets egress via the NAT), **14** (VPC endpoints — the cheaper alternative to NAT for AWS-service traffic).
