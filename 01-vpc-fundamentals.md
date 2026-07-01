# 01 — VPC Fundamentals

## 1. What you're learning and why it matters

A **VPC (Virtual Private Cloud)** is the outermost box of everything on AWS — the private, isolated network your app, database, and cache live inside. Before any container runs, you decide *how big a range of private IP addresses you own* and *how isolated it is*. Get it wrong and you either run out of address space as you grow, or you overlap with a network you later need to connect to (office VPN, another VPC). Both are painful because the VPC's range is effectively permanent once things run inside it. This lesson makes `10.0.0.0/16` feel obvious and sets up every networking lesson that follows.

### Terms first

- **VPC** — your isolated virtual network inside one AWS region. The boundary all your private IPs live in.
- **CIDR block** — notation like `10.0.0.0/16` for a range of IP addresses. You only need the "range of IPs" meaning.
- **Private IP ranges** — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`: reserved for private networks, never routed on the public internet. VPCs use these.
- **Region** — a geographic AWS location (e.g. `us-east-2`). A VPC lives in exactly one.
- **Availability Zone (AZ)** — a physically isolated datacenter within a region (detail in lesson 02).
- **DNS hostnames / support** — VPC settings that let internal resources get and resolve DNS names. You want both on.

---

## 2. Mental model

> **A VPC is a private building you rent in one city (region): you fix the total floor space up front (the CIDR block), nobody outside can walk in by default, and everything you ever run lives in some room inside it.**

The CIDR is the floor space; subnets (next lesson) are the rooms.

---

## 3. Concept sections

### 3.1 What a VPC isolates (and what it doesn't)

A VPC gives you **network isolation**: resources in one VPC can't reach another VPC or the internet *unless you build a path*. Two VPCs — even your own — are as separate as two companies' networks, and can reuse the same IP ranges without conflict precisely because they're isolated.

What it isolates: **network traffic** (default-deny inbound) and **IP address space** (your `10.0.x.x` means something only inside your VPC).

What it does **not** isolate: **IAM / API access.** IAM is account-global and ignores VPC boundaries — an identity with `s3:GetObject` can read that bucket from any VPC or none. A VPC bounds *packets*, not *API permissions*. This is the two-planes idea from doc 00, and it's the #1 thing app engineers get wrong.

> **Rails analogy:** A VPC is like running in an isolated namespace where `localhost` only sees its own services. Two VPCs both using `10.0.x.x` is like two machines both using `127.0.0.1` — same addresses, zero collision, because they can't see each other.

### 3.2 Reading a CIDR block without fear

`10.0.0.0/16` says two things: the starting address (`10.0.0.0`) and how many bits are *fixed* (`/16`). The rest are free for your addresses.

Rule: **`/N` = first N bits locked, giving `2^(32−N)` addresses. Smaller number = bigger network.**

| CIDR | Addresses | Plain English |
|------|-----------|---------------|
| `10.0.0.0/16` | 65,536 | "I own all of `10.0.x.x`" |
| `10.0.0.0/20` | 4,096 | a big subnet-sized chunk |
| `10.0.0.0/24` | 256 | "I own all of `10.0.0.x`" |
| `10.0.0.0/28` | 16 | "I own `10.0.0.0`–`.15`" |

A `/16` *contains* many `/20`s, which contain many `/24`s — that nesting is exactly how you carve a VPC into subnets next lesson. You'll never compute this by hand — the agent does it. You just need to *read* it: `/16` = big building, room to grow.

📎 [VPC and subnet sizing (AWS)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)

### 3.3 Pick the range deliberately

Use the private ranges (`10.x`, `172.16–31.x`, `192.168.x`) — they're never routed publicly. 

The forward-looking part app engineers miss: **pick a range that won't collide with anything you'll ever connect to.** Here's why it matters: if two networks that need to talk both claim `172.31.0.0/16` (AWS's *default* VPC range, which you should avoid), you can't route between them. An address is ambiguous — the router doesn't know which network's `172.31.5.10` you mean, so the connection dies. 

**Real example: Analytics + Production**

```
Prod VPC: 10.0.0.0/16
├─ RDS (transactional workload)
└─ Live traffic

Analytics VPC: 10.0.0.0/16  ← Same CIDR (mistake!)
├─ Redshift Data Warehouse
└─ ETL Workers

Problem:
  Analytics worker needs hourly data pull from prod-db
  Without VPC Peering: can't reach (different VPC, blocked)
  
  Try to enable VPC Peering:
    Route table sees "10.0.2.50" traffic
    Is this prod-db or analytics-db?
    ✗ AMBIGUOUS — peering fails or sends to wrong database
    ✗ Analytics pipeline breaks
```

With deliberate CIDR planning:

```
Prod VPC: 10.20.0.0/16
Analytics VPC: 10.50.0.0/16  ← Distinct range

Route table: "10.20.x.x → prod via peering ✓"
             "10.50.x.x → analytics locally ✓"

Result: Analytics worker queries prod-db reliably every hour
```

The cheap insurance is **deliberate CIDR planning from day one**: assign a distinct `10.x` slice per environment (prod `10.20.0.0/16`, staging `10.30.0.0/16`, analytics `10.50.0.0/16`, etc.) so they can be peered or connected later without collisions. Same principle if you ever need to integrate with an office VPN, another team's infrastructure, or a partner network — if your CIDR overlaps theirs, the integration is impossible unless someone re-IPs everything.

**Why it's hard to change:** the primary CIDR is set at creation. You can *add* secondary ranges but can't shrink or relocate the primary without recreating the VPC — and everything in it. Measure twice.

### 3.3b How VPC Peering Works (and Why CIDR Matters)

**VPC Peering** is a direct network tunnel between two VPCs that lets them communicate as if on the same network.

**Step-by-step:**

1. **Create peering request** — owner of Prod VPC requests connection to Analytics VPC
2. **Accept** — Analytics VPC owner accepts
3. **Add routes** — each VPC's route table learns how to reach the other:
   - Prod route table: "Traffic to 10.50.x.x → Analytics VPC via peering"
   - Analytics route table: "Traffic to 10.20.x.x → Prod VPC via peering"
4. **Allow in security groups** — each VPC permits traffic from the other's CIDR
5. **Now they communicate** — Analytics EC2 can query Prod RDS directly

**Real scenario: Analytics pulls from Prod**

```
Analytics EC2 (10.50.1.100) tries to connect to Prod RDS (10.20.2.50)

Step 1: Analytics EC2 sends packet to 10.20.2.50
Step 2: Route table lookup: "10.20.x.x → use peering"
Step 3: Packet flows through peering tunnel
Step 4: Prod VPC receives packet
Step 5: RDS security group checks: "10.50.0.0/16 allowed?" → Yes ✓
Step 6: Connection established, query runs ✓
```

**Why distinct CIDRs are critical:**

With collision (both use 10.0.0.0/16):
```
Route table sees packet to 10.0.2.50
Question: Is this local (Analytics) or peering (Prod)?
✗ AMBIGUOUS — routing fails
```

Without collision (Analytics 10.50.0.0/16, Prod 10.20.0.0/16):
```
Route table sees packet to 10.20.2.50
Answer: That's Prod's range → route via peering
Route table sees packet to 10.50.2.50
Answer: That's local → stay in Analytics
✓ Clear and unambiguous
```

### 3.4 A VPC spans a region, not an AZ

A VPC lives in one **region** but automatically spans every **AZ** in it. So high availability — surviving a datacenter failure — comes from spreading subnets across AZs *inside the one VPC*, not from multiple VPCs. Lesson 02 uses this directly. For now: **VPC = region-wide; the AZ choice happens at the subnet level.**

### 3.5 Turn DNS hostnames on

Two settings: `enable_dns_support` (resolve DNS at all — default on) and `enable_dns_hostnames` (give resources internal DNS names — default *off* on custom VPCs). You want **both on**, because RDS (lesson 12), ElastiCache (lesson 13), and service discovery (lesson 10) all hand you a *DNS name*, not an IP. With hostnames off, those names won't resolve and you'll chase a phantom "network" bug that's really DNS. Turn both on and forget it.

### 3.6 Ignore the default VPC

Every account has a default VPC (`172.31.0.0/16`) per region, pre-wired with public subnets so beginners can launch instantly. For real, Terraform-managed infra you make your own and ignore it, because: its `172.31` range is identical in every account (collision landmine), its subnets are public by default (opposite of a secure posture), and "AWS magic" contradicts the "everything explicit" model. If a tutorial "just works" with no VPC, it's silently using the default — don't build on it.

---

## 4. Cost & blast radius

**Cost:** A VPC and its subnets, route tables, internet gateway, and security groups are **free**. ✅ The container of your network costs nothing — bills come from things *inside* it (NAT, compute, databases). So you can create and destroy VPCs freely while learning.

**Blast radius:**
- **CIDR too small** (e.g. `/24`): runs out of IPs as you add subnets/tasks → Fargate "insufficient addresses" failures. *Medium; avoid by sizing `/16`.*
- **CIDR overlap** with a future-connected network: peering/VPN impossible without re-IPing. *High, expensive; fully avoidable now.*
- **DNS hostnames off:** internal DB/cache names don't resolve. *Low-medium; confusing but quick to fix.*
- **Building on the default VPC:** inherited public subnets → things exposed you assumed private. *High (security); avoid by making your own.*

Summary: **cheap and forgiving to experiment with, but the CIDR choice is one of the few genuinely costly-to-change decisions.** Spend your care there.

---

## 5. The Terraform

The medium, not the subject — focus on which line expresses which idea from §3.

```hcl
resource "aws_vpc" "main" {
  # §3.2 — the CIDR: the "floor space". /16 = 65,536 addresses, a
  # deliberate 10.x slice (§3.3) chosen to avoid future collisions.
  cidr_block = "10.20.0.0/16"

  # §3.5 — both ON so internal DNS names (RDS, ElastiCache, service
  # discovery) resolve. enable_dns_hostnames is OFF by default, so set it.
  enable_dns_support   = true
  enable_dns_hostnames = true

  # §3.1 — 'default' tenancy = shared hardware. What you want;
  # 'dedicated' is expensive and for rare compliance cases.
  instance_tenancy = "default"

  tags = {
    Name        = "prod-vpc"
    Environment = "production"
  }
}
```

That single block creates the sealed building. Note what's *absent*: no subnets, no internet access, no firewall rules — a bare VPC is a sealed box, the secure default. Everything later is you deliberately furnishing and opening it.

📎 [`aws_vpc` (Terraform registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)

> **For the agent:** let it generate the block, but *you* supply the CIDR after deciding it (§3.3). Never let it pick `172.31` or a default — that's the one decision you own.

---

## 6. How Workforce does it

Your real production VPC (`production/modules/vpc`, region `us-east-2`):

| Decision | Workforce value | Maps to |
|----------|-----------------|---------|
| VPC CIDR | `10.0.0.0/16` | §3.2 — a /16, ~65k addresses, plenty of headroom |
| Range choice | `10.0.x.x` | §3.3 — a clean `10.x` slice, not the `172.31` default ✅ |
| Subnet split | 3 public + 3 private `/20` (4,096 each) across `us-east-2a/b/c` | §3.4 — spread across AZs for failure survival (lesson 02) |
| Placement | ALB in public subnets; ECS, RDS, Redis in **private** | the secure posture this whole course builds |

**What to notice and question:**
- Workforce's production VPC uses `10.0.0.0/16`, the most common "first VPC" range. Fine in isolation — but this decision locks you in. **If staging or dev *also* use `10.0.0.0/16` (very common mistake), you can't peer or connect them later; the overlap is a showstopper.** When you spin up a new environment, pick a deliberately distinct slice (`10.20.0.0/16` for prod, `10.30.0.0/16` for staging, `10.40.0.0/16` for dev) so they can be connected later without rebuild. This is the "cheap insurance" from §3.3.
- The staging and demo environments are *not* Terraform-managed (built in the Console), so their CIDR choices aren't codified — a real-world example of why "decide deliberately" (§3.3) matters: an un-codified range is easy to accidentally reuse and collide.

> Source: `workforce-api-infra/INFRASTRUCTURE.md` → *Networking — VPC*.

---

## 7. How to use this doc with an agent

1. **Build:** *"Generate one `aws_vpc` for production with CIDR `10.20.0.0/16`, DNS support + hostnames on, default tenancy, sensible tags. Then explain each argument and what breaks if I omit it."*
2. **Probe:** *"For this VPC, what's reachable from the internet, and what can resources inside reach? Walk me through why a fresh VPC is a sealed box and name the missing pieces that would change that."*
3. **Quiz:** *"Quiz me on CIDR sizing: give me five blocks, ask how many addresses each holds and whether one contains another, then ask why `172.31.0.0/16` is a poor choice. Don't reveal answers until I respond."*
4. **Refactor:** *"I have a `/24` VPC. Show the problems as I add subnets and Fargate tasks, and my fix options — be explicit about which require a rebuild."*

---

## 8. Checkpoints

1. What does a VPC isolate — and name one thing it does *not*.
2. How many addresses does a `/16` hold vs a `/24`? Which slash number is the bigger network?
3. Why is `172.31.0.0/16` a bad choice for a real VPC?
4. A VPC lives in one ___ but spans all the ___ in it — why does the second matter?
5. Which two DNS settings do you enable, and what breaks if `enable_dns_hostnames` is off?
6. Roughly what does an empty VPC cost, and what does that tell you about experimenting here?

---

## 9. Footguns

- **VPC too small.** A `/24` runs out of IPs fast (each Fargate task eats one). **Default to `/16`; it's free.**
- **Default VPC / `172.31` range.** Identical everywhere → collisions, plus public-by-default subnets. **Make your own.**
- **Forgetting `enable_dns_hostnames = true`.** Off by default on custom VPCs → RDS/Redis/discovery names don't resolve → phantom "network" bug.
- **Treating the VPC as a security boundary for everything.** It bounds traffic, not IAM. Don't reason "it's in the VPC so it's safe."
- **Same CIDR for prod and staging.** Works fine *until* you need to connect them — peering or VPN becomes impossible. The overlap is a showstopper. **Plan environment CIDRs upfront (e.g. prod `10.20.0.0/16`, staging `10.30.0.0/16`, dev `10.40.0.0/16`) even if you don't peer today.** It's free to do now, expensive to fix later.
- **Overlapping your office/VPN range.** Cheapest to avoid now, most expensive to fix later.

---

## 10. Ask-the-agent cheatsheet

- *"Suggest three non-overlapping `/16` ranges for prod/staging/dev that avoid common defaults and leave room for peering."*
- *"Given CIDR `<X>`, how many addresses, and does it overlap `<Y>`? Show the overlap if any."*
- *"Review this `aws_vpc` for secure-by-default posture: anything implicitly exposed, DNS settings correct?"*
- *"What's the smallest VPC CIDR that fits N Fargate tasks across two AZs plus an ALB and an RDS instance? Show the math."*
- *"What symptom would I see at an RDS endpoint if `enable_dns_hostnames = false`?"*

---

## 11. Where this goes next

**→ 02 — Subnets & Availability Zones.** You have the building and its floor space; now carve it into rooms. You'll split the `/16` into public and private subnets across AZs and map that onto a real app tier (ALB public; web/worker/RDS/Redis private). The `cidr_block` math from §3.2 pays off, and "survive a datacenter failure" becomes concrete.

Leans on this lesson: **03** (how a subnet becomes public — a routing fact), **05** (the IAM plane a VPC deliberately doesn't control), **12/13** (why `enable_dns_hostnames` mattered).
