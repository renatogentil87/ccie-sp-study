# AWS Networking Fundamentals — Week 3: VPC Connectivity (Peering, Transit Gateway, PrivateLink)

## The Problem Nobody Warns You About

You built your first VPC. The app works. Security Groups are locked down. Life is good.

Then the company acquires a second team. They have their own VPC. Their microservice needs to talk to your database. Your database is in a private subnet with no internet route. Their VPC has a completely different CIDR block.

How do you connect two isolated networks that were designed to be isolated?

This is the VPC connectivity problem — and choosing wrong costs you months of rearchitecting.

---

## Three Tools, Three Use Cases

AWS gives you three ways to connect VPCs. Each solves a different problem:

```
VPC Peering         →  "Connect two VPCs directly, like a crossover cable"
Transit Gateway     →  "Connect many VPCs through a hub, like a core router"
PrivateLink         →  "Expose one service to another VPC, like a reverse proxy"
```

The enterprise networking equivalent:

```
Enterprise Network              →    AWS
────────────────────────────────────────────────
Back-to-back router link        →    VPC Peering
MPLS PE-to-PE / core router     →    Transit Gateway
Reverse proxy / load balancer   →    PrivateLink (VPC Endpoint Service)
DMZ service exposure             →    PrivateLink
```

Choose wrong and you'll hit one of these walls:
- Peering doesn't transit — VPC-A peers with VPC-B, VPC-B peers with VPC-C, but VPC-A **cannot** reach VPC-C through VPC-B
- Transit Gateway adds cost and latency for simple two-VPC scenarios
- PrivateLink only works for specific services, not full network connectivity

---

## VPC Peering — The Direct Link

VPC Peering creates a **point-to-point connection** between two VPCs. Traffic stays on the AWS backbone — never touches the internet.

```
VPC-A (10.0.0.0/16) ←——— Peering ———→ VPC-B (172.16.0.0/16)
```

Think of it as a crossover cable between two routers. Direct, fast, no middlebox. But it only connects those two — nobody else can use the link.

**How to set it up:**

```
Step 1: VPC-A owner sends a peering request to VPC-B
Step 2: VPC-B owner accepts the request
Step 3: Both sides add routes to their route tables:

VPC-A route table:
  10.0.0.0/16      → local
  172.16.0.0/16    → pcx-xxxxx    (peering connection)

VPC-B route table:
  172.16.0.0/16    → local
  10.0.0.0/16      → pcx-xxxxx    (peering connection)
```

That's it. Instances in VPC-A can now reach instances in VPC-B using private IPs. No internet, no NAT, no VPN.

**Critical rules:**

```
1. NO TRANSITIVE ROUTING
   VPC-A ↔ VPC-B ↔ VPC-C does NOT mean VPC-A can reach VPC-C
   Each pair needs its own peering connection

2. NO OVERLAPPING CIDRs
   VPC-A: 10.0.0.0/16 + VPC-B: 10.0.0.0/16 → CANNOT PEER (conflict)
   Plan your IP addressing before you need connectivity

3. CROSS-REGION WORKS
   VPC in us-east-1 can peer with VPC in eu-west-1
   Traffic stays on AWS backbone (encrypted cross-region)

4. CROSS-ACCOUNT WORKS
   Your VPC can peer with a partner's VPC in a different AWS account
```

**Enterprise comparison:** VPC Peering is like a back-to-back BGP session between two PE routers. Fast, simple, but doesn't scale. If you have 10 VPCs, you need 45 peering connections (n × (n-1) / 2). Sound familiar? It's the same full-mesh problem as iBGP without route reflectors.

**When to use Peering:**
- 2-3 VPCs that need full connectivity
- Low-latency, high-throughput requirements (peering has the lowest latency)
- Simple architecture with no growth beyond a few VPCs

**When NOT to use Peering:**
- More than 4-5 VPCs (full mesh becomes unmanageable)
- You need transitive routing (VPC-A through VPC-B to VPC-C)
- You need centralized security inspection

---

## Transit Gateway — The Core Router

Transit Gateway (TGW) is a **regional hub** that connects VPCs, VPNs, and Direct Connect through a single point. Every attached VPC talks to the TGW, and the TGW routes between them.

```
                    ┌─── VPC-A (10.0.0.0/16)
                    │
TGW ────────────────├─── VPC-B (172.16.0.0/16)
                    │
                    ├─── VPC-C (192.168.0.0/16)
                    │
                    ├─── VPN to on-premises
                    │
                    └─── Direct Connect
```

**Enterprise comparison:** Transit Gateway IS your core router. It's the MPLS PE that every branch connects to. Routes propagate through route tables, and you control which VPCs can talk to which using TGW route table associations and propagations.

```
Enterprise MPLS Core          →    Transit Gateway
PE-CE connection              →    TGW Attachment (VPC, VPN, DX)
VRF on the PE                 →    TGW Route Table
Route-target import/export    →    Route Table association/propagation
Inter-VRF leaking             →    Multiple TGW route tables with selective propagation
```

**How it works:**

```
Step 1: Create a Transit Gateway in your region
Step 2: Attach VPCs (creates an ENI in each VPC's subnet)
Step 3: TGW route table auto-learns the attached VPC CIDRs

TGW Route Table (default):
  10.0.0.0/16      → VPC-A attachment
  172.16.0.0/16    → VPC-B attachment
  192.168.0.0/16   → VPC-C attachment

Each VPC route table needs a route pointing to the TGW:
  VPC-A: 0.0.0.0/0 → tgw-xxxxx   (or specific CIDRs)
```

**Transitive routing — the key advantage:**

```
VPC-A → TGW → VPC-C    ✅ works (TGW routes between attachments)
VPC-A → TGW → VPN      ✅ works (on-premises reaches any VPC)
VPN   → TGW → DX       ✅ works (transit between VPN and Direct Connect)
```

This is what Peering CAN'T do. TGW is the hub that makes everything reachable through a single point.

**Segmentation with multiple route tables:**

Just like VRFs on a PE router, you can create multiple TGW route tables to isolate traffic:

```
TGW Route Table "Production":
  VPC-Prod-A, VPC-Prod-B → can reach each other + shared services
  Cannot reach Dev VPCs

TGW Route Table "Development":
  VPC-Dev-A, VPC-Dev-B → can reach each other + shared services
  Cannot reach Prod VPCs

TGW Route Table "Shared Services":
  VPC-DNS, VPC-Monitoring → reachable from both Prod and Dev
```

**Enterprise comparison:** This is exactly RT import/export in MPLS L3VPN. Production VRF imports production RTs. Development VRF imports development RTs. Shared services RT is imported by both. Same concept, different platform.

**When to use Transit Gateway:**
- 4+ VPCs that need connectivity
- Hybrid cloud (VPN / Direct Connect to on-premises)
- Centralized security inspection (route all traffic through a firewall VPC)
- Network segmentation (prod vs dev vs shared services)

**When NOT to use Transit Gateway:**
- Two VPCs with simple connectivity (Peering is cheaper and lower latency)
- You only need to expose a single service, not full network reachability (use PrivateLink)

**Cost consideration:** TGW charges per attachment ($0.05/hr ≈ $36/month per VPC) plus per-GB data processing ($0.02/GB). For two VPCs, Peering is free. For 20 VPCs, TGW is far cheaper than managing 190 peering connections.

---

## PrivateLink — Expose a Service, Not a Network

PrivateLink is fundamentally different from Peering and TGW. It doesn't connect networks. It exposes a **single service** from one VPC to another through a private endpoint.

```
Service Provider VPC                Consumer VPC
┌──────────────────┐               ┌──────────────────┐
│  NLB → App Fleet │──PrivateLink──│  VPC Endpoint     │
│  (10.1.0.0/16)   │               │  (172.16.0.0/16)  │
└──────────────────┘               └──────────────────┘

Consumer sees: a private IP in their OWN VPC (172.16.x.x)
Consumer does NOT see: the provider's network (10.1.x.x)
```

The consumer doesn't get access to the provider's VPC. They get access to one specific service, presented as a local endpoint in their own VPC. No route table changes. No CIDR overlap concerns. No transitive risk.

**Enterprise comparison:** PrivateLink is a **reverse proxy** or **DMZ service publication**. You don't give the partner access to your network — you put a load balancer in front of your service and let them connect to that. They see your service, not your infrastructure.

```
Enterprise:
  Partner → Firewall → Reverse Proxy → Your App
  Partner sees: proxy IP
  Partner doesn't see: your internal network

AWS PrivateLink:
  Consumer VPC → VPC Endpoint → NLB → Your App
  Consumer sees: endpoint IP in their VPC
  Consumer doesn't see: your VPC CIDR
```

**Three types of VPC Endpoints:**

```
1. Interface Endpoint (PrivateLink)
   Creates an ENI in your subnet with a private IP
   Used for: AWS services (SQS, KMS, SSM) and custom services
   Cost: $0.01/hr + $0.01/GB

2. Gateway Endpoint (free)
   Adds a route to your route table
   Used for: S3 and DynamoDB ONLY
   Cost: free (no hourly charge, no data charge)

3. Gateway Load Balancer Endpoint
   For inline security appliances (firewalls, IDS)
   Used for: traffic inspection architectures
```

**When to use PrivateLink:**
- Exposing a service to another VPC without full network access
- Consuming AWS services (S3, SQS, KMS) from private subnets without NAT/IGW
- Third-party SaaS integration (Datadog, Snowflake, etc. publish PrivateLink endpoints)
- Zero-trust architecture — no network-level trust, only service-level access
- Overlapping CIDRs between VPCs (PrivateLink doesn't care about CIDR overlap)

**When NOT to use PrivateLink:**
- You need full network connectivity between VPCs (use Peering or TGW)
- You need to route between on-premises and cloud (use TGW + VPN/DX)

---

## The Decision Matrix

```
Question                                          Answer
────────────────────────────────────────────────────────────────────
How many VPCs?
  2-3                                            → VPC Peering
  4+                                             → Transit Gateway

Need transitive routing?
  Yes (A→B→C)                                    → Transit Gateway
  No (only A↔B)                                  → VPC Peering

Need on-premises connectivity?
  Yes                                            → Transit Gateway (+ VPN or DX)
  No                                             → Peering or PrivateLink

Exposing a single service, not a full network?
  Yes                                            → PrivateLink
  No (full IP reachability needed)               → Peering or TGW

Overlapping CIDRs?
  Yes                                            → PrivateLink (only option)
  No                                             → Any option works

Centralized security inspection?
  Yes                                            → Transit Gateway (route through firewall VPC)
  No                                             → Any option works

Cost sensitive, simple topology?
  Yes                                            → VPC Peering (free data transfer)
  No (complex, growing)                          → Transit Gateway
```

---

## The Combination — Real-World Architecture

In production, you rarely use just one. A typical enterprise architecture combines all three:

```
                         Transit Gateway
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
        VPC-Prod          VPC-Dev          VPC-Shared
       (workloads)       (workloads)       (DNS, monitoring)
            │
            ├── PrivateLink → S3 (Gateway Endpoint, free)
            ├── PrivateLink → KMS (Interface Endpoint, private access)
            └── PrivateLink → Partner SaaS (no network exposure)

        On-premises ── Direct Connect ── Transit Gateway
```

- **TGW** connects the VPCs and on-premises (hub-and-spoke, route tables for segmentation)
- **PrivateLink** exposes specific AWS services to private subnets without internet
- **Peering** might be used between two VPCs that need ultra-low-latency (e.g., database replication between regions)

---

## Common Mistakes

**1. Using Peering when you need transit**

```
VPC-A ↔ VPC-B (peered)
VPC-B ↔ VPC-C (peered)
VPC-A → VPC-C: "why doesn't this work?"

Because Peering is non-transitive. You need either:
  - A third peering: VPC-A ↔ VPC-C
  - Transit Gateway connecting all three
```

**2. Forgetting route table entries after Peering**

```
Peering is created and accepted. But no traffic flows.
Why? Neither VPC added the route to their route table.
Peering is NOT plug-and-play. You must update routes on BOTH sides.
```

**3. Security Group references don't work across TGW**

```
VPC-A SG: "allow inbound from sg-webapp-vpc-b"
Traffic comes from VPC-B through TGW → SG reference doesn't resolve → DENIED

Fix: use CIDR-based rules for cross-VPC traffic via TGW
  "allow inbound TCP 443 from 172.16.0.0/16"
```

**4. Using NAT Gateway + IGW when PrivateLink exists**

```
Private subnet instance needs to call S3.
Bad:  Route through NAT Gateway → IGW → S3 (public internet path, costs $$)
Good: S3 Gateway Endpoint → free, private, stays on AWS backbone
```

**5. Peering across regions without considering latency**

```
VPC in us-east-1 peered with VPC in ap-southeast-1.
The peering works — but your app now has 200ms latency on every API call.
Cross-region peering is encrypted and private, but physics still applies.
```

---

## Key Takeaways

1. **VPC Peering = direct link.** Fast, free, but doesn't transit and doesn't scale past a few VPCs. Use it when you have 2-3 VPCs with simple connectivity needs.

2. **Transit Gateway = core router.** Scales to thousands of VPCs, supports transitive routing, connects to on-premises. Use it as your default for any architecture with 4+ VPCs or hybrid connectivity.

3. **PrivateLink = service exposure, not network connectivity.** Use it when you want to share a service without sharing your network. The only option when CIDRs overlap.

4. **Always update route tables.** Peering and TGW attachments don't automatically route traffic. You must add routes in every VPC that needs to send traffic through the connection.

5. **Use Gateway Endpoints for S3 and DynamoDB.** They're free. Interface Endpoints cost money. Don't pay for PrivateLink when a Gateway Endpoint does the same job.

6. **Plan your IP addressing before you need connectivity.** Overlapping CIDRs prevent Peering and TGW from working. PrivateLink is the only escape — but it's not full network connectivity. Address planning is cheaper than rearchitecting.

---

*Next week: Direct Connect and VPN — connecting your on-premises data center to AWS. When each makes sense, how they differ, and why you probably need both.*

*This is Week 3 of 7 in the AWS Networking Fundamentals series.*

---

**#AWS #VPC #TransitGateway #VPCPeering #PrivateLink #Networking #CloudNetworking #Fundamentals #Week3**
