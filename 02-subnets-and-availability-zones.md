# 02 — Subnets & Availability Zones

## 1. What you're learning and why it matters

Last lesson you rented the building (the VPC) and fixed its floor space (the CIDR). Now you carve it into **rooms** — subnets — and decide which rooms have a door to the street (public) and which are locked interior rooms (private). Two decisions happen here, and both have real consequences: **which tier of your app goes in which kind of room** (put a database in a public room and you've exposed it to the internet), and **how you spread rooms across separate buildings** (Availability Zones) so one datacenter catching fire doesn't take you down. This is where "survive a failure" and "the database isn't reachable from the internet" stop being slogans and become concrete placement decisions.

### Terms first

- **Subnet** — a slice of the VPC's CIDR range that lives in exactly **one** Availability Zone. The "room."
- **Availability Zone (AZ)** — one physically isolated datacenter within a region (its own power, cooling, network). A region has several (e.g. `us-east-2a`, `us-east-2b`, `us-east-2c`).
- **Public subnet** — a subnet whose route table has a path to an Internet Gateway. Reachable from / can reach the internet.
- **Private subnet** — a subnet with no direct internet path. The default, and where most things belong.
- **Route table** — the rules deciding where a subnet's traffic goes (full lesson 03). Mentioned here because it's *what makes a subnet public or private*.
- **Availability (HA)** — staying up when a component fails. Achieved by spreading across AZs.
- **Tier** — a layer of your app: web/load-balancer tier, app/compute tier, data tier. Maps onto subnet groups.

---

## 2. Mental model

> **A subnet is a room in one specific building (AZ). "Public vs private" is just whether that room has a door to the street. You put each tier of your app in the kind of room it deserves, and you duplicate every tier across two or three buildings so losing one building doesn't lose the app.**

---

## 3. Concept sections

### 3.1 A subnet is a CIDR slice pinned to one AZ

A subnet takes a chunk of the VPC's address range and binds it to a single AZ. That AZ binding is the important, easy-to-miss part: **a subnet cannot span AZs.** If you want your app in three AZs, you need three subnets — one per AZ. This is why real setups always have subnets in multiples (3 public + 3 private, etc.): it's one-per-AZ, per tier.

```
VPC 10.0.0.0/16
├─ subnet 10.0.0.0/20    → us-east-2a   (4,096 addresses, this AZ only)
├─ subnet 10.0.16.0/20   → us-east-2b
└─ subnet 10.0.32.0/20   → us-east-2c
```

The `/20` slices don't overlap and all sit inside the `/16` — the nesting from lesson 01 §3.2 in action.

> **Note:** AWS reserves **5 addresses in every subnet** (network address, VPC router, DNS, future use, broadcast). So a `/28` (16 addresses) really gives you 11 usable. Irrelevant at `/20`, but it's why a tiny subnet is even tinier than it looks.
AWS reserves 5 specific IP addresses in every subnet for its own use. Here's what they are:

1. **Network address** — the first IP (e.g., 10.0.1.0 in a /24)
2. **VPC router** — the second IP (e.g., 10.0.1.1), your gateway to other subnets
3. **DNS server** — the third IP (e.g., 10.0.1.2), for AWS's internal DNS resolution
4. **Future use** — the fourth IP is reserved by AWS for future features
5. **Broadcast address** — the last IP (e.g., 10.0.1.255 in a /24)

**Why it matters for small subnets:**

- A `/28` gives you 16 total addresses, but lose 5 to reserves = **11 usable IPs for actual resources** (EC2 instances, RDS, etc.)
- A `/24` gives you 256 addresses, so losing 5 is negligible (251 usable)
- A `/20` gives you 4,096 addresses, so it barely registers

This is why tiny subnets get hit hard — a `/28` loses 31% of its addresses. It's one reason many teams avoid ultra-small subnets except for very specific use cases (like NAT gateway subnets where you're not launching many instances).

### 3.2 Public vs private is a *routing* fact, not a flag

There is no `public = true` checkbox. **A subnet is public if and only if its route table sends internet-bound traffic (`0.0.0.0/0`) to an Internet Gateway.** Remove that route and the exact same subnet becomes private. Privacy here is emergent from routing — the single most important idea to carry from lesson 01 into 03.

```
Public subnet:   route table has  0.0.0.0/0 → Internet Gateway   ✓ internet
Private subnet:  route table has  (no 0.0.0.0/0 → IGW)           ✗ no direct internet
                 (it may have     0.0.0.0/0 → NAT Gateway for outbound-only — lesson 03)
```

So when someone asks "is this thing exposed to the internet?", the answer is never "it's in a private subnet" by name — it's **"follow the route table."** We wire those routes in lesson 03; here you just place things in the *intended* kind of subnet.

### 3.3 Map subnets to your app tiers

The whole point of splitting is to give each tier the exposure it needs and nothing more. The standard three-tier layout:

```
┌─ PUBLIC subnets ──────────────────────────────────┐
│  Only internet-facing things:                      │
│    • Application Load Balancer                      │
│    • NAT Gateway (so private subnets can egress)    │
└────────────────────────────────────────────────────┘
┌─ PRIVATE subnets (app tier) ──────────────────────┐
│    • ECS/Fargate tasks (Rails web, Sidekiq worker) │
│      no public IP; reachable only via the ALB      │
└────────────────────────────────────────────────────┘
┌─ PRIVATE subnets (data tier) ─────────────────────┐
│    • RDS PostgreSQL, ElastiCache Redis             │
│      reachable only from the app tier              │
└────────────────────────────────────────────────────┘
```

Smaller shops collapse the app and data tiers into one set of private subnets (that's what Workforce does — see §6). The principle stands: **only the load balancer is public; your code and your data are private.**

> **Rails analogy:** Think of it like request scoping in a multi-tenant app. The ALB is your public controller endpoint — the one thing the outside world hits. Your Sidekiq workers and your Postgres connection never accept a request straight from the internet; they only ever get called from inside the trusted boundary. Putting RDS in a public subnet is like exposing your database credentials on a public route.

### 3.4 AZs are failure domains — spread across them

An AZ is a separate datacenter. AWS designs them so that a power/cooling/network failure in one AZ doesn't affect another. Your leverage: **run each tier in at least two AZs.** If `us-east-2a` goes dark, your tasks in `2b` keep serving and the load balancer routes around the dead zone.

```
             AZ-a                    AZ-b
   ┌──────────────────┐   ┌──────────────────┐
   │ ALB node         │   │ ALB node         │  ← ALB lives in both
   │ web task ×1      │   │ web task ×1      │  ← tasks split across AZs
   │ RDS primary      │   │ RDS standby      │  ← Multi-AZ = standby here
   └──────────────────┘   └──────────────────┘
        AZ-a dies  ───────────►  traffic + DB fail over to AZ-b, app stays up
```

Single-AZ is the classic rookie outage: everything in `2a`, `2a` has a bad afternoon, you're fully down with no recourse. Spreading costs you nothing extra for the subnets themselves (they're free) — you only pay for the duplicated *resources* you choose to run.

### 3.5 Subnet sizing: carve the /16 sensibly

You're dividing the VPC range into per-AZ, per-tier chunks. A clean, common scheme for a `/16` is `/20` subnets (4,096 addresses each) — big enough that Fargate tasks (one IP each) never exhaust them, with room for dozens of subnets. Don't over-optimize: address space inside a `/16` is plentiful and free, so favor generous, uniform slices over clever tight ones. The failure mode you're avoiding is a subnet so small that scaling out fails with "insufficient free IP addresses."

### 3.6 `map_public_ip_on_launch` — the subtle exposure knob

A subnet has a setting that auto-assigns a public IP to resources launched into it. Turn it **on for public subnets** (so an ALB/NAT gets a public address) and **off for private subnets** (the default). Getting this backwards is a quiet way to hand a public IP to something you meant to keep private — a task that then has a *potential* internet path if the routing also allows it. Treat "on" as a deliberate choice reserved for genuinely public subnets.

---

## 4. Cost & blast radius

**Cost:** Subnets, AZs, and route-table associations are **free**. ✅ You pay only for resources placed in them, and — notably — for **cross-AZ data transfer** (traffic between AZs carries a small per-GB charge). That's a reason to be aware of chatty cross-AZ traffic later, not a reason to avoid multi-AZ; the availability win dwarfs the pennies.

**Blast radius:**
- **Data tier in a public subnet:** RDS/Redis potentially reachable from the internet — a serious exposure. *Severity: high (security). The canonical "don't do this."*
- **Everything in one AZ:** total outage when that AZ fails, no failover possible. *Severity: high (availability). Avoid by spreading across ≥2 AZs.*
- **Subnet too small:** scale-out and new tasks fail with "insufficient IP addresses." *Severity: medium; avoid with `/20`-ish sizing.*
- **`map_public_ip_on_launch` on for a private subnet:** unintended public IPs. *Severity: medium-high; keep it off for private.*

Summary: subnets are free to create, but *placement is a security and availability decision*. The two things you must get right: **data in private subnets, and every tier across multiple AZs.**

---

## 5. The Terraform

Subnets express the two decisions from §3: which AZ, and (via a later route table) public or private. Here's a two-AZ pair of public + private subnets, annotated.

```hcl
# --- PUBLIC subnets: one per AZ. "Public" is finalized by the route
#     table in lesson 03; here we just place them and allow public IPs. ---
resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.0.0/20"        # §3.5 — a /20 slice of the /16
  availability_zone = "us-east-2a"         # §3.1 — pinned to ONE AZ

  # §3.6 — ON for public subnets so the ALB/NAT gets a public IP.
  map_public_ip_on_launch = true

  tags = { Name = "public-us-east-2a", Tier = "public" }
}

resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.16.0/20"
  availability_zone       = "us-east-2b"    # §3.4 — a DIFFERENT AZ = HA
  map_public_ip_on_launch = true
  tags = { Name = "public-us-east-2b", Tier = "public" }
}

# --- PRIVATE subnets: one per AZ. No public IPs; no internet route
#     will be added (except outbound via NAT in lesson 03). ---
resource "aws_subnet" "private_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.128.0/20"
  availability_zone       = "us-east-2a"
  map_public_ip_on_launch = false           # §3.6 — OFF for private (default)
  tags = { Name = "private-us-east-2a", Tier = "private" }
}

resource "aws_subnet" "private_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.144.0/20"
  availability_zone       = "us-east-2b"
  map_public_ip_on_launch = false
  tags = { Name = "private-us-east-2b", Tier = "private" }
}
```

Notice: nothing here says "public" or "private" as a property — the only difference so far is `map_public_ip_on_launch` and the *intended* routing. The subnet becomes truly public only when lesson 03 attaches a route to an Internet Gateway. That's §3.2 made literal.

📎 Concept: [VPCs and subnets (AWS)](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) · Resource: [`aws_subnet` (Terraform registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)

> **For the agent:** *"List the AZs available in `us-east-2`, then generate public and private subnets one per AZ, sized `/20`, with public-IP-on-launch only on the public ones."* Make it query real AZs first — AZ names/counts differ per region and account.

---

## 6. How Workforce does it

From `production/modules/vpc`, region `us-east-2`:

| Decision | Workforce value | Maps to |
|----------|-----------------|---------|
| Public subnets | 3 × `/20` — `10.0.0.0/20`, `10.0.16.0/20`, `10.0.32.0/20` (AZ a/b/c) | §3.1, §3.5 — one per AZ, generous sizing |
| Private subnets | 3 × `/20` — `10.0.128.0/20`, `10.0.144.0/20`, `10.0.160.0/20` (AZ a/b/c) | §3.4 — spread across all 3 AZs |
| Public subnets hold | **only** the ALB (and the NAT Gateway) | §3.3 — only the load balancer is public |
| Private subnets hold | ECS tasks (api ×2, worker ×1), RDS, ElastiCache | §3.3 — app *and* data tier collapsed into private |

**What to notice and question:**
- Workforce uses **three** AZs, not two — more resilience, and the subnets are free; you only pay for whatever you actually run in each. The `api` service (desired count 2) can land in different AZs, so an AZ failure leaves a serving task.
- App and data tiers **share** the private subnets rather than getting separate subnet groups. Fine for this size — the isolation between app and data is enforced by **security groups** (lesson 04), not by separate subnets. Worth knowing the distinction: separate subnets would let you add NACL boundaries or distinct routing per tier if you ever needed it.
- The RDS and Redis sitting in private subnets is exactly §3.3 — neither is reachable from the internet regardless of any security-group mistake, because there's simply no public route.

> Source: `workforce-api-infra/INFRASTRUCTURE.md` → *Networking — VPC* (subnet CIDRs) and the architecture diagram.

---

## 7. How to use this doc with an agent

1. **Build:** *"Query the AZs in `us-east-2`, then generate 3 public + 3 private `/20` subnets (one per AZ) inside my existing VPC. Explain how each subnet's CIDR fits inside the VPC `/16` and why public-IP-on-launch differs between them."*
2. **Probe:** *"For each subnet you created, tell me: is it public or private *right now*, and how do you know? What single change would flip a private subnet to public?"* (Answer must reference the route table, not a flag.)
3. **Quiz:** *"Ask me which tier (ALB / ECS / RDS / Redis) belongs in which subnet and why, then ask what breaks if I put RDS in a public subnet, and what breaks if I put everything in a single AZ. Don't reveal answers until I respond."*
4. **Refactor:** *"My subnets are all in one AZ. Show me how to spread each tier across two more AZs, what it changes in cost (call out cross-AZ data transfer), and what I gain in availability."*

---

## 8. Checkpoints

1. Can a single subnet span two AZs? What does that imply about how many subnets you need for a 3-AZ setup?
2. What actually makes a subnet "public"? (Not a flag — what specifically?)
3. Which tier goes in a public subnet, and why is it the *only* one?
4. Why does putting RDS in a private subnet protect it even if a security group is misconfigured?
5. What's the failure you're preventing by spreading subnets across AZs, and roughly what does the extra availability cost at the subnet level?
6. What does `map_public_ip_on_launch` do, and what's the right value for a private subnet?

---

## 9. Footguns

- **Database in a public subnet.** The classic exposure. RDS/Redis belong in private subnets, always.
- **Single-AZ everything.** One AZ blip = full outage with no failover. Spread every tier across ≥2 AZs.
- **Subnets too small.** A `/26` or `/27` runs out of IPs as tasks scale; new tasks fail to launch. Use `/20`-ish inside a `/16`.
- **`map_public_ip_on_launch = true` on private subnets.** Silently hands out public IPs. Keep it off for private.
- **Assuming "private subnet" means safe by name.** Safety is the *routing*; a misapplied route can still open it. Verify the route table (lesson 03).
- **Forgetting subnets are AZ-bound when reading `Multi-AZ` features.** RDS Multi-AZ, ALB across AZs, etc. all require you to have provided subnets *in those AZs* first.

---

## 10. Ask-the-agent cheatsheet

- *"List available AZs in `<region>` and propose a subnet plan: N public + N private `/20`s, one per AZ, inside VPC `<cidr>`. Show each subnet's range and confirm none overlap."*
- *"Audit my subnets: for each, is it currently public or private (check the route tables), and does anything sensitive sit in a public one?"*
- *"Is my RDS/ElastiCache in a private subnet? Prove it by showing there's no `0.0.0.0/0 → IGW` route on its subnet."*
- *"Am I single-AZ anywhere? List each tier and which AZs it currently spans."*
- *"Estimate cross-AZ data-transfer cost if my app tier in AZ-a talks heavily to RDS primary in AZ-b."*

---

## 11. Where this goes next

**→ 03 — Routing & Gateways.** You've placed the rooms; now you wire the doors. This is where "public vs private" stops being an intention and becomes real: route tables, the Internet Gateway that makes a subnet public, and the **NAT Gateway** that lets private subnets reach *out* without being reachable — including its famous cost trap. Everything you tagged "public" or "private" here gets its actual behavior in 03.

Leans on this lesson: **03** (the routes that finalize public/private), **04** (security groups — the tier isolation that lets Workforce share private subnets), **07/08** (ECS tasks and the ALB placed into these exact subnets), **12/13** (RDS/Redis in the private subnets).
