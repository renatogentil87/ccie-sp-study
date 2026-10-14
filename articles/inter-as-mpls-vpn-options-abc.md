# Inter-AS MPLS VPN: Why Your Customer's Traffic Crosses Autonomous System Boundaries

## The Problem: One Customer, Multiple Service Providers

Your enterprise customer has offices in São Paulo, London, and Tokyo. No single service provider covers all three cities. SP1 covers Latin America. SP2 covers Europe. SP3 covers Asia-Pacific. Each SP runs its own Autonomous System with its own IGP, its own MPLS core, its own route reflectors.

The customer doesn't care about your AS boundaries. They want one L3VPN that connects all their sites — seamlessly, with a single SLA, as if the three networks were one.

This is the Inter-AS MPLS VPN problem. And every large-scale service provider solves it daily.

---

## Why Can't You Just Extend the IGP?

Inside a single AS, MPLS L3VPN is straightforward:

```
CE → PE → (MPLS core, one IGP, one set of RRs) → PE → CE
```

The PE routers exchange VPNv4 routes via MP-iBGP through route reflectors. LDP or SR builds the transport LSP. Done.

But across AS boundaries, three things break:

1. **IGP doesn't cross AS boundaries** — IS-IS in AS 65100 doesn't talk to OSPF in AS 65200. By design. You don't want SP2's IGP flooding your core.

2. **LDP labels are local** — a label allocated by AS 65100 means nothing in AS 65200. There's no shared label space.

3. **iBGP stays inside the AS** — your route reflector in AS 65100 doesn't peer with AS 65200's route reflector. VPNv4 routes stop at the AS boundary.

Three options exist to solve this. Each trades simplicity for scalability.

---

## Option A — Back-to-Back VRF

The simplest approach. The two ASBRs treat each other as customer edge routers.

**How it works:**

Each ASBR creates a VRF for every customer that needs cross-AS connectivity. The ASBRs peer with each other using a per-VRF eBGP session (or static route) over a dedicated sub-interface per VRF. From the ASBR's perspective, the far-side ASBR is just another CE.

```
AS 65100                          AS 65200
PE ──iBGP──→ ASBR ──per-VRF eBGP──→ ASBR ──iBGP──→ PE
                   (sub-interface)
                   VRF GREEN: .1 ←→ .2
                   VRF YELLOW: .1 ←→ .2
                   VRF BLUE: .1 ←→ .2
```

Each VRF gets its own sub-interface on the inter-AS link. Routes are exchanged as plain IPv4 inside the VRF — no VPNv4 labels, no MPLS on the boundary link. The ASBR receives the VRF route, imports it into its local VPNv4 table, and distributes it via iBGP to its own RR.

**Why SPs use it:**

- Dead simple to configure — it's just PE-CE routing on the ASBR
- Complete isolation between customers at the boundary
- No trust required between the two ASes — each side controls its own VRF policy
- Perfect for a small number of shared customers (2-5 VRFs)

**Why it doesn't scale:**

- Every new customer = a new sub-interface + a new eBGP session on both ASBRs
- 500 customers = 500 sub-interfaces per ASBR pair. Per link. Times redundancy.
- Configuration grows linearly with customer count
- ASBR becomes a bottleneck — every VPN route is redistributed twice (VPNv4 → VRF → VRF → VPNv4)

**Real-world use:** Peering between two SPs that share a handful of enterprise customers. Common at national boundaries where SP1 hands off to SP2 for a few multinational accounts.

---

## Option B — VPNv4 eBGP Between ASBRs

The middle ground. ASBRs exchange VPNv4 routes directly — no per-VRF configuration needed.

**How it works:**

The two ASBRs establish a single VPNv4 eBGP session. ALL customer VPN routes cross the boundary through this one session, carrying their RD and RT. The receiving ASBR doesn't need VRFs — it just passes VPNv4 routes through to its own RR via iBGP.

```
AS 65100                              AS 65200
PE ──iBGP(VPNv4)──→ ASBR ──eBGP(VPNv4)──→ ASBR ──iBGP(VPNv4)──→ PE
                          (single session)
                          ALL VRFs in one session
```

The ASBR acts as a label boundary — it allocates a new VPN label for each route it re-advertises. The inter-AS link carries labeled packets (VPN label only, no transport label since there's no LDP between ASes). The ASBR sets next-hop-self so its own iBGP peers can resolve the path.

**Critical configuration:** The ASBR must retain all route targets, even for VRFs it doesn't locally import. Without `retain route-target all`, the ASBR drops VPNv4 routes for customers that don't exist on the ASBR itself.

**Why SPs use it:**

- One eBGP session handles ALL customers — adding a new VPN doesn't touch the ASBR config
- No per-VRF sub-interfaces — scales to thousands of VPNs
- Clean separation — each AS still owns its own IGP and LDP domain
- RT-based filtering at the boundary — the ASBR can filter which VPNs cross the boundary

**Why it has limitations:**

- The ASBR must process ALL VPNv4 routes — memory and CPU on the ASBR grow with total VPN prefix count
- Next-hop changes at every AS boundary — the ASBR rewrites next-hop-self, which means the LSP is segmented (no end-to-end LSP)
- Label swap at the ASBR — the ASBR allocates new labels for every VPNv4 prefix, consuming label space
- Two ASBRs must trust each other enough to exchange VPNv4 — one side sees the other's RT values

**Real-world use:** The most common Inter-AS option in production. Used between divisions of the same SP group, or between closely partnered SPs. Carrier-neutral peering for MPLS VPN wholesale.

---

## Option C — BGP Labeled Unicast + Multihop VPNv4

The most scalable, most complex option. Builds an end-to-end LSP across AS boundaries so VPNv4 traffic flows without label swapping at every ASBR.

**How it works:**

Two separate BGP sessions do different jobs:

1. **BGP-LU (Labeled Unicast):** ASBRs exchange PE loopback /32 routes WITH labels. This creates an end-to-end LSP from PE in AS 65100 to PE in AS 65300 — through the ASBRs, across boundaries, all labeled.

2. **Multihop VPNv4:** The route reflectors in each AS peer directly with each other (multihop eBGP for VPNv4). VPN routes carry the original PE's next-hop — no next-hop-self at the ASBR. The LSP built by BGP-LU resolves the remote PE's loopback.

```
Control plane:
  RR(65100) ──multihop eBGP(VPNv4)──→ RR(65200)
  ASBR(65100) ──eBGP(BGP-LU)──→ ASBR(65200)

Data plane:
  PE(65100) ──LSP(LDP+BGP-LU labels)──→ PE(65200)
  End-to-end labeled path. No label swap at ASBR boundary.
```

The ASBR doesn't process VPNv4 at all — it only handles labeled unicast for PE loopback reachability. The VPNv4 routes flow directly between RRs. The ASBR just swaps BGP-LU transport labels like a P router.

**Why SPs use it:**

- ASBRs are lightweight — they only handle labeled loopback routes, not thousands of VPNv4 prefixes
- End-to-end LSP — enables services that need a continuous label path (MPLS-TE across domains, end-to-end QoS)
- RR-to-RR peering — scales independently of ASBR capacity
- Most elegant architecture for large-scale multi-AS deployments

**Why it's complex:**

- Three layers of labels: LDP (intra-AS transport) + BGP-LU (inter-AS transport) + VPN (service label)
- Every router in the path must handle 3-label stacks — hardware must support it
- BGP-LU next-hop-self must be configured correctly at every ASBR boundary — one wrong next-hop breaks the LSP
- Troubleshooting is harder — you're debugging three label planes simultaneously
- RR-to-RR peering requires careful security (the RRs see each other's full VPNv4 table)

**Real-world use:** Tier-1 carriers with multiple regions or acquisitions. The "Unified MPLS" or "Seamless MPLS" architecture that large SPs deploy for national/global backbones. Also used in Carrier Supporting Carrier (CSC) models.

---

## The Comparison

| | Option A | Option B | Option C |
|---|---|---|---|
| **Complexity** | Low | Medium | High |
| **Scalability** | Poor (per-VRF config) | Good (single session) | Excellent (ASBR is lightweight) |
| **ASBR load** | High (all VRFs + redistribution) | High (all VPNv4 routes) | Low (only loopback labels) |
| **End-to-end LSP** | No | No | Yes |
| **VPNv4 on ASBR** | No (plain IPv4 per VRF) | Yes | No (on RRs only) |
| **New customer impact** | New sub-interface + BGP per ASBR | Zero ASBR changes | Zero ASBR changes |
| **Trust between ASes** | Low (per-VRF isolation) | Medium (shared VPNv4) | High (RR-to-RR peering) |
| **Label stack depth** | 1 (intra-AS only) | 2 (VPN + transport) | 3 (VPN + BGP-LU + LDP) |
| **Typical use case** | 2-5 shared customers | Hundreds of VPNs | Thousands of VPNs, Unified MPLS |

---

## How Large ISPs Actually Do It

In practice, most large SPs don't pick just one option. They combine them:

**Between internal regions (same company, different ASes):** Option C. They trust their own RRs, they want end-to-end LSPs for traffic engineering, and they have thousands of VPNs. This is Unified MPLS.

**Between partner SPs (different companies):** Option B. They share VPNv4 for wholesale VPN services but don't expose their PE loopbacks or RR infrastructure. RT filtering at the ASBR controls which VPNs cross the boundary.

**Between SP and a small enterprise customer running their own MPLS:** Option A. The enterprise has 2-3 VRFs, low route count, and wants complete isolation. Simple, contained, no risk.

**The trend:** As Segment Routing replaces LDP, Inter-AS connectivity evolves. SR-MPLS uses prefix-SIDs that are globally significant — no LDP, no targeted sessions, no BGP-LU in some designs. The three-option framework still applies, but the transport layer simplifies dramatically.

---

## Key Takeaways

1. **Option A is simple but doesn't scale.** Use it for a handful of shared customers between two SPs. One sub-interface per VRF per link — manageable at 5, painful at 50, impossible at 500.

2. **Option B is the workhorse.** One VPNv4 session handles all customers. Most deployed option worldwide. The ASBR carries the load but the config is clean.

3. **Option C is the architect's choice.** Lightweight ASBRs, end-to-end LSP, RR-to-RR scalability. But three-label stacks and careful next-hop handling make troubleshooting harder.

4. **The right answer depends on scale and trust.** Small peering = Option A. Medium wholesale = Option B. Large unified backbone = Option C. Most SPs use a combination.

5. **Always plan the RT scheme before deploying.** Route targets must be consistent across ASes for the VPN to work end-to-end. A mismatched RT at the boundary means the customer's routes silently disappear.

---

*Currently studying for CCIE Service Provider — configuring all three options on a multi-AS lab topology with Cisco 7200 routers. Next article: MPLS Traffic Engineering — why SPs build tunnels when they already have shortest-path routing.*

---

**#CCIESP #MPLS #InterAS #L3VPN #BGP #ServiceProvider #Networking #LearnInPublic #NetworkEngineering**
