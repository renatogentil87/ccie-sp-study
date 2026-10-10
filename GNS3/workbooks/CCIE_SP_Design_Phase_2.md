# CCIE SP Design Phase 2 — Meridian Telecom

**Format:** CLC-style (CCIE EI Design exam format) — email-based scenario questions, no diagrams provided, no answers included.
**Candidate task:** Read each email thread carefully. Answer only what is asked. Assume all other design elements not mentioned are already correctly implemented unless an email states otherwise.

---

## Company Background

**Meridian Telecom** is a tier-2 service provider operating three regional autonomous systems that were merged through a series of acquisitions. The network engineering team is now standardizing designs across the three regions ahead of a major customer renewal cycle. You have been engaged as the design consultant.

### Topology Summary (Reference Only — matches your lab topology)

| AS | Region | IGP | Routers | Route Reflector(s) | ASBRs | PE Routers | P Routers |
|---|---|---|---|---|---|---|---|
| **65100** | North | IS-IS Level-2 | R1–R8 | R7, R8 (dual RR) | R5, R6 | R1, R2 | R3, R4 |
| **65200** | Central | OSPF Area 0 | R11–R16, R19 | R19 | R12 | R14, R15, R16 | R11, R13 |
| **65300** | South | IS-IS Level-2 | R20–R23, R30 | R30 | R20, R21 | R20, R21, R22, R23 | — |

**Inter-AS links (4 total):**
- R5 (AS65100) ↔ R11 (AS65200)
- R6 (AS65100) ↔ R12 (AS65200)
- R6 (AS65100) ↔ R20 (AS65300)
- R12 (AS65200) ↔ R21 (AS65300)

**Customer base (same as production):** Customer Green (eBGP + OSPF PE-CE, spans North+Central), Customer Yellow (eBGP, spans South+Central), Customer Blue (eBGP, spans all three ASes), Customer Red (L2VPN VPWS/VPLS, spans North+Central).

Candidates should assume IS-IS areas, OSPF Area 0, loopback addressing, and PE-CE protocols match the standard lab topology reference unless an email explicitly overrides it.

---

## SECTION 1 — IS-IS Design (2 Questions)

### Email Thread 1.1 — North Region NOC

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** Sub-second failure detection on R5–R6 core link
>
> We've had two incidents in the last quarter where the R5–R6 Gigabit link between our two ASBRs flapped due to a transceiver issue, and IS-IS took almost 8 seconds to notice because the link stayed "up/up" at Layer 1 for a few seconds before actually failing. Our SLA requires sub-second detection and reroute for core links carrying inter-AS traffic.
>
> We're told we should enable BFD for IS-IS between R5 and R6, and possibly across other core links in AS 65100. I want to understand two things before we commit to this:
>
> 1. What exactly changes in IS-IS adjacency behavior when BFD is enabled on an interface, in terms of what triggers the adjacency to go down?
> 2. Is there any interaction we need to be aware of between BFD timers and the IS-IS SPF/LSP generation throttling we also want to tune (see the next question)? Could aggressive BFD cause SPF churn if a link is merely flapping at the physical layer?
>
> Please give me your recommendation on whether to deploy BFD for IS-IS on all core AS 65100 links or just the R5–R6 and R3–R4 links, and justify the choice.

**Question 1:** Describe how BFD for IS-IS changes failure-detection behavior on the R5–R6 link, and provide your recommendation (with justification) on which AS 65100 links should run BFD for IS-IS.

---

### Email Thread 1.2 — North Region Architecture Review

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Cc:** Marcus Webb, Senior Network Architect
> **Subject:** RE: SPF timers and startup behavior for R7/R8
>
> Following up on the BFD discussion — Marcus wants the SPF throttle timers tuned so that a single link flap doesn't cause repeated full SPF runs, but we still want fast initial convergence on the *first* topology change. Please give us the three timer values we need to configure (and clarify the order they're entered in on IOS) to achieve: fast reaction to the first event, but exponential backoff for subsequent events within a short window.
>
> Separately — Marcus raised a concern about R7 and R8 (our two RRs). When either one reloads after a software upgrade, he's worried it will start forwarding transit traffic for other IS-IS routers before it has synchronized its full LSDB, causing temporary blackholing. He wants a mechanism where the router advertises itself as "do not use me for transit" for a bounded time right after boot, and automatically clears that state once it's reasonable to assume convergence has occurred. What is this mechanism called, how do you configure the automatic-clear behavior, and does it affect prefixes advertised *for* R7/R8 itself (i.e., its own loopback) or only transit paths through it?

**Question 2:** Provide the three SPF throttle timer values (and the correct configuration order) that satisfy Marcus's requirement, and explain the overload-bit on-startup mechanism he's asking about, including whether it affects routes to the RR's own loopback.

---

## SECTION 2 — OSPF Design (3 Questions)

### Email Thread 2.1 — Central Region IPv6 Rollout

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** IPv6 in AS 65200 core
>
> We're planning to light up native IPv6 reachability across R11, R12, R13, R14, R15, R16, and R19 so Customer Green's IPv6 CE (R17, which is already OSPF PE-CE for IPv4) can get dual-stack service without us having to re-architect the IGP. Someone on my team suggested just running OSPFv3 in parallel with our existing OSPFv2 Area 0 process rather than trying to retrofit the existing process.
>
> Two questions:
> 1. What are the structural differences between OSPFv2 and OSPFv3 that make "run them side by side" the right call here, versus trying to make one process do both AFs?
> 2. If we do run OSPFv3, do we need new Router IDs, or can we reuse the existing OSPFv2 RIDs? What happens if we don't explicitly configure one on a router that has no IPv4 address at all (hypothetically, if we ever add an IPv6-only P router)?

**Question 3:** Explain the key structural/architectural differences between OSPFv2 and OSPFv3 relevant to this parallel deployment, and answer Elena's Router ID question.

---

### Email Thread 2.2 — Customer Green OSPF PE-CE

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** Customer Green loop prevention on R15–R17
>
> Customer Green runs OSPF as the PE-CE protocol between our PE (R15) and their CE (R17), per the standard design. Green has told us they are planning to add a *second* site, multihomed to both R15 and one of our other PEs reachable via a different VPN path, also using OSPF PE-CE. Their network engineer specifically asked us: "how do you prevent a route that originated in our OSPF domain, got redistributed into MP-BGP as a VPNv4 route, and then got redistributed back into OSPF at the second site, from being mistaken for a route that originated natively inside OSPF at the second site?" He's clearly done this before and expects a real answer, not "trust us."
>
> Please explain the exact mechanism (down to the LSA field involved) and what operational problems arise if it's misconfigured — specifically what happens with Type-3 vs Type-5/Type-7 LSAs in this scenario.

**Question 4:** Explain the DN-bit and Domain-ID mechanism that solves Customer Green's loop-prevention concern, identifying which LSA types carry the DN-bit and the consequence of misconfiguration.

---

### Email Thread 2.3 — Backdoor Link Proposal

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** Customer Yellow wants a backup path that bypasses MPLS core
>
> Customer Yellow (eBGP PE-CE today, spanning AS 65300 and AS 65200) is asking whether we can offer a backup connectivity option where two of their sites have a direct leased line between them that does NOT go through our MPLS backbone, purely as a backup for when their primary VPN paths through us are both down. They want OSPF to still see this backup path as part of the same area for proper cost-based preference, without us having to carry it as a transit VPN route through our P routers.
>
> This sounds like a sham-link to me, but I want you to confirm:
> 1. Is sham-link actually the right tool here, or is this scenario actually describing something else (since the backdoor link is between CE sites, not between PEs)?
> 2. If sham-link IS applicable in a similar future scenario where two of OUR PEs for the same customer have a real backbone-bypassing physical link between them, what's required to make OSPF treat that VPN-path route as an intra-area route instead of inter-area, and why does that matter for route preference?

**Question 5:** Clarify whether sham-link applies to Marcus's CE-to-CE backdoor scenario, and separately explain the sham-link mechanism and its effect on route preference for the PE-to-PE case he describes.

---

## SECTION 3 — BGP Design (4 Questions)

### Email Thread 3.1 — Conditional Advertisement for Customer Blue

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Customer Blue backup path should only announce when primary fails
>
> Customer Blue is multihomed across all three of our ASes (unique in our customer base). Their primary site connects via R1 (AS 65100), and they have a backup site connecting via R21 (AS 65300). Blue has asked that under normal conditions, we should NOT advertise their backup prefix out of AS 65300 toward other peers at all — they don't want any inbound traffic hitting the backup path unless the primary path at R1 is completely gone from our tables. Once R1's path disappears, R21 should start advertising immediately.
>
> What BGP feature implements this "only advertise prefix B if prefix A is absent" behavior, and what are the configuration components involved (map types, conditions)? Also — what's the practical risk if the "exist-map"/"non-exist-map" condition is evaluated against a prefix that's flapping?

**Question 6:** Explain the BGP conditional advertisement mechanism (route-map types and condition logic) that satisfies Customer Blue's requirement, and describe the risk of applying it to a flapping condition prefix.

---

### Email Thread 3.2 — Fast Convergence Comparison

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** ADD-PATH vs BGP PIC — which do we actually need?
>
> Our leadership read a vendor whitepaper and now wants to know why we haven't deployed "both ADD-PATH and PIC everywhere" for faster convergence on PE router R1 (dual-homed toward R3 and R7 for next-hop resolution purposes, and running VPNv4 to our RRs). I need to push back with an accurate technical comparison, not just "we'll do both."
>
> Please explain:
> 1. What problem each of these features actually solves (they are NOT solving the same problem, even though both get pitched as "fast convergence").
> 2. Whether BGP PIC requires ADD-PATH to function, or they're fully independent.
> 3. A scenario specific to our RR design (R7/R8 dual RR in AS 65100) where ADD-PATH between RR and PE would provide a tangible benefit that plain BGP PIC Edge would not.

**Question 7:** Compare ADD-PATH and BGP PIC (Edge/Core as relevant), clarify their independence, and provide a scenario specific to the R7/R8 dual-RR design where ADD-PATH provides benefit beyond PIC alone.

---

### Email Thread 3.3 — Community-Based Policy Standardization

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Standardizing outbound policy with communities instead of per-neighbor route-maps
>
> Right now, every ASBR (R5, R6, R12, R20, R21) has its own hand-built route-maps for inbound/outbound policy toward the other ASes, and they've drifted out of sync — R6 treats Customer Yellow prefixes differently than R12 does for the same customer. We want to move to a community-based policy model where PE routers tag routes with well-known communities at origination (e.g., "customer-tier-gold", "backup-path", "do-not-export-to-AS65300") and ASBRs just match on communities rather than prefix-lists.
>
> 1. What BGP attributes/features need to be enabled network-wide for this to work reliably across all three ASes (remember, two ASes run IS-IS and one runs OSPF — does that matter here)?
> 2. Walk through what happens to a community value by default when a route crosses from AS 65100 into AS 65200 at the R6–R12 boundary, and what has to be configured if we want Meridian-internal communities to survive that hop without leaking to AS 65300 beyond.

**Question 8:** Explain the prerequisites (attributes/features) for a community-based policy model across the three ASes, and describe default community propagation behavior across the R6–R12 boundary plus what's needed to control further propagation into AS 65300.

---

### Email Thread 3.4 — MED Comparison Inconsistency

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** Weird best-path selection for Customer Yellow prefixes
>
> We're seeing inconsistent best-path selection for a Customer Yellow prefix that's advertised into AS 65300 from two different neighbors with different AS-path lengths but we're not supposed to be comparing MED between them at all per the default IOS behavior — yet it looks like we ARE comparing MED across different neighboring ASes on R20. I suspect someone enabled `bgp always-compare-med` or `bgp deterministic-med` globally and we don't remember why, or there's a path hidden by "first AS" grouping that's throwing things off.
>
> Explain the default (non-deterministic, non-always-compare) MED comparison rule regarding which paths are even eligible to be compared, how `bgp deterministic-med` changes path *grouping/ordering* behavior (not the actual MED values), and separately how `bgp always-compare-med` changes *eligibility*. Then tell me which of the two settings could explain comparing MED across different neighboring ASes, since that shouldn't normally happen.

**Question 9:** Explain default MED comparison eligibility rules, how `bgp deterministic-med` differs from default behavior, how `bgp always-compare-med` differs, and identify which setting explains cross-AS MED comparison on R20.

---

## SECTION 4 — MPLS Traffic Engineering (2 Questions)

### Email Thread 4.1 — FRR Protection for Core Links

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** Fast reroute for R3-R4-R5-R6 ring
>
> We want sub-50ms protection for TE tunnels crossing the R3–R4, R4–R6, and R5–R6 links (the "ring" portion of our core). Someone proposed facility backup; someone else insists one-to-one backup gives better guarantees since it's per-LSP. We have roughly 40 TE tunnels crossing this ring today and expect to grow to 150+ over the next year.
>
> Explain the structural difference between facility backup and one-to-one backup (what gets built, how many backup LSPs exist, what signaling extension facility backup relies on), and make a scalability-based recommendation for which to use given our tunnel count trajectory.

**Question 10:** Compare RSVP-TE FRR facility backup vs. one-to-one backup (structure and signaling), and recommend one for Meridian's North region core given current and projected tunnel counts.

---

### Email Thread 4.2 — Bandwidth-Aware TE with Growth

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** DS-TE and auto-bandwidth for seasonal traffic
>
> Our core link bandwidth needs to be partitioned so that voice-bearing TE tunnels get a guaranteed sub-pool that other data tunnels can't starve, especially during our Q4 traffic surge when data tunnel demand roughly doubles. We also don't want to manually resize tunnel bandwidth reservations every quarter.
>
> 1. Explain how DS-TE with two bandwidth pools (global pool and sub-pool) would let us guarantee bandwidth for voice tunnels while still allowing data tunnels to use leftover capacity.
> 2. Separately, explain what auto-bandwidth actually adjusts (the reservation, or the physical allocation) and whether it would conflict with our sub-pool guarantee if misconfigured — i.e., could an auto-bandwidth-enabled data tunnel "steal" from the voice sub-pool over time?

**Question 11:** Explain DS-TE global-pool/sub-pool design for Meridian's voice/data separation, and explain auto-bandwidth's actual mechanism and the risk it poses to the sub-pool guarantee if misconfigured.

---

## SECTION 5 — L3VPN Design (3 Questions)

### Email Thread 5.1 — VRF-Lite vs Full MPLS VPN for a Small Site

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** New small customer — do we really need full MPLS VPN?
>
> A new prospective customer only needs isolation between two departments at a single site connected to R14, with no need for any-to-any connectivity across our backbone (just local route separation on that one PE). Sales wants to undercut a competitor's price and asked whether we can just use VRF-Lite instead of standing up full MPLS L3VPN (RD/RT, MP-BGP VPNv4) for this customer.
>
> Explain the core architectural difference between VRF-Lite and full MPLS L3VPN, specifically what's missing in VRF-Lite that would prevent us from later expanding this customer to a second site on R15 without a redesign, and give your recommendation.

**Question 12:** Explain the architectural difference between VRF-Lite and MPLS L3VPN, identify what VRF-Lite lacks that would block painless multi-site expansion, and give a recommendation for this customer.

---

### Email Thread 5.2 — IPv6 VPN for Customer Blue

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Customer Blue wants IPv6 VPN service across all 3 ASes
>
> Customer Blue (spanning AS 65100, 65200, 65300 — our only 3-AS customer) now wants native IPv6 VPN service on top of their existing IPv4 L3VPN, without us deploying a parallel 6PE/dual-stack-core redesign. We were told "6VPE" is the way to do this using our *existing* IPv4 MPLS core and *existing* VPNv4 infrastructure.
>
> Explain how 6VPE carries IPv6 customer routes across an IPv4-only MPLS core (what BGP address family is used, what the VPN-IPv6 NLRI actually looks like at a high level, and what the data-plane next-hop/label stack looks like for traffic from R1 to R21 for this customer). Also clarify: does 6VPE require the P routers (R3, R4, R11, R13) to run IPv6 in any form?

**Question 13:** Explain the 6VPE control-plane and data-plane mechanics for Customer Blue's IPv6 VPN across the existing IPv4 MPLS core, and clarify whether P routers need IPv6 support.

---

### Email Thread 5.3 — RT-Constraint for RR Scale

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** R19 and R30 are getting overloaded with irrelevant VPNv4 routes
>
> As we onboard more customers, R19 (Central RR) and R30 (South RR) are reflecting VPNv4 routes to PEs that have no VRF configured with the matching Route-Target at all. For example, R22 (South PE) has zero VRFs for Customer Green, yet R30 still sends it every Green VPNv4 route because that's just how route reflection works today. We want ASBRs/RRs to only propagate what's actually needed downstream.
>
> Explain how RT-Constraint (RFC 4684) solves this, specifically: what new address family/NLRI is used to signal "I want RT X," how an RR like R30 uses that signaling to filter what it reflects, and what has to be true about ALL relevant routers in the propagation path (PEs, RRs, ASBRs) for RT-Constraint to work end-to-end rather than silently failing partway.

**Question 14:** Explain the RT-Constraint (RFC 4684) mechanism — the signaling address family, how R30 would use it to prune reflected VPNv4 routes, and the end-to-end deployment requirement across the propagation path.

---

## SECTION 6 — Inter-AS VPN Design (3 Questions)

### Email Thread 6.1 — Carrier Supporting Carrier for a Wholesale Customer

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** Wholesale customer wants to run their OWN MPLS VPN over our backbone
>
> We have a new wholesale opportunity: a smaller regional ISP wants to buy transit from us but run their *own* MPLS L3VPN service on top of our network — their CE routers (connecting into our R14 and R16) are actually their PE routers, and they want their own customers' VPNv4 routes carried transparently through our AS 65200 core without us ever seeing their VPN customers' routes in our own VRFs.
>
> This sounds like Carrier Supporting Carrier (CSC) to me. Explain:
> 1. What's fundamentally different about the PE-CE relationship in CSC vs normal L3VPN PE-CE — specifically what has to be exchanged between our PE (R14/R16) and their "CE" (which is really their PE) beyond normal customer routes.
> 2. Does our P-router core (R11, R13) need any awareness of the wholesale customer's VPN labels, or does CSC keep that fully hidden from P routers the same way normal L3VPN does?

**Question 15:** Explain how the PE-CE exchange differs in a CSC design compared to standard L3VPN PE-CE, and clarify whether Meridian's P routers (R11, R13) require any awareness of the wholesale customer's inner VPN labels.

---

### Email Thread 6.2 — Option C Label Path Walkthrough

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Need the exact label stack for an Option C packet R1 -> R21
>
> We run Inter-AS Option C across our AS65100↔AS65300 boundary (R6↔R20) today using multihop eBGP VPNv4 between R7 (RR, AS65100) and R30 (RR, AS65300), with BGP-LU (labeled unicast) carrying the ASBR loopbacks between the ASes so VPNv4 next-hops stay reachable.
>
> For a packet going from Customer Blue's site behind PE R1 (AS 65100) to Customer Blue's site behind PE R21 (AS 65300), walk through the COMPLETE label stack as the packet crosses R3 → R6 → (inter-AS link) → R20 → R23 → R21, identifying which label(s) are swapped vs which stay constant, and specifically what the TOP label is doing at each hop (IGP-LDP label vs BGP-LU label vs VPN label).

**Question 16:** Provide the complete Option C label stack walkthrough for a Customer Blue packet from R1 to R21 via R6↔R20, identifying each label's origin (LDP, BGP-LU, VPN) and which labels swap vs. remain constant at each hop.

---

### Email Thread 6.3 — Choosing an Inter-AS Option for a New Boundary

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** New direct AS65300<->AS65200 wholesale link — which Option?
>
> We're adding a 5th inter-AS link directly between AS 65300 and AS 65200 (separate from the existing R12↔R21 link) for a new wholesale deal where the counterpart network's engineering team has said they will only support "redistributing VPNv4 into IPv4 at the border, re-advertise as plain eBGP" — i.e., they explicitly do NOT want to run MPLS or exchange labels across the new boundary at all, for security/administrative reasons on their side.
>
> Build us a comparison matrix across Option A, Option B, and Option C covering: whether labels cross the AS boundary, per-VRF vs aggregate configuration overhead at the ASBR, scalability as VRF count grows, and which option matches the counterpart's stated requirement. State your recommendation.

**Question 17:** Build an Option A / Option B / Option C comparison matrix (label exchange across boundary, ASBR config overhead, scalability with VRF growth) and recommend the option matching the counterpart's "no MPLS across boundary" requirement.

---

## SECTION 7 — Multicast Design (2 Questions)

### Email Thread 7.1 — SSM vs ASM for IPTV Rollout

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** IPTV multicast model for Central region
>
> We're launching an IPTV service for residential customers off R14/R15/R16. Each customer's set-top box will know in advance exactly which content-source server to request from (via an EPG/channel guide that encodes source IP). Our video team is asking whether we even need rendezvous-point infrastructure (BSR/Auto-RP) for this, since they keep hearing about PIM-SSM as being simpler.
>
> Explain the fundamental difference between SSM and ASM in terms of what the receiver must know ahead of time, whether an RP is required for SSM, and whether SSM would actually eliminate our need for BSR/Auto-RP infrastructure elsewhere in the Central region (e.g., for other, non-IPTV multicast applications that don't support source-aware receivers).

**Question 18:** Explain the SSM vs. ASM distinction (receiver knowledge requirements, RP necessity) and clarify whether adopting SSM for IPTV eliminates the need for RP discovery infrastructure for other multicast applications in AS 65200.

---

### Email Thread 7.2 — NG-MVPN Profile Selection

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Which NG-MVPN profile for Customer Yellow's multicast VPN?
>
> Customer Yellow wants multicast VPN service across AS 65300 and AS 65200 (their existing eBGP L3VPN footprint). They don't want us relying on GRE-based mGRE/PIM in the core — they specifically asked about "profiles that use MPLS/SR in the core, with BGP-based auto-discovery instead of PIM for the overlay."
>
> Give a short explanation of what profiles 0, 3, 6, 7, and 14 in the NG-MVPN profile framework represent at a high level (core tree mechanism + C-multicast signaling), and recommend which profile fits Customer Yellow's "no PIM/mGRE in the core" requirement, with justification.

**Question 19:** Briefly characterize NG-MVPN profiles 0, 3, 6, 7, and 14 (core tree mechanism and C-multicast signaling for each), and recommend a profile for Customer Yellow matching their "no PIM/mGRE in the core" requirement.

---

## SECTION 8 — QoS Design (2 Questions)

### Email Thread 8.1 — Per-VRF Policing for Fair Usage

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** Customer Green is bursting past their contracted rate on R1
>
> Customer Green's VRF on R1 is contracted for 50 Mbps but we're seeing sustained bursts to 80 Mbps during business hours, degrading other customers' VRFs sharing the same R1 uplink. We want to enforce the contracted rate specifically at the VRF boundary on R1, not on the physical interface (since the physical interface carries multiple customers' VRFs).
>
> Explain how per-VRF policing differs from interface-level policing in terms of where in the forwarding path it's applied, and whether a single physical interface with multiple VRFs (like R1's g1/0, f3/0, f4/1 toward different customers) can have independent policing rates per VRF on the SAME physical interface, or if policing is inherently tied to the physical interface as a whole.

**Question 20:** Explain how per-VRF policing differs from interface-level policing in application point, and clarify whether independent per-VRF policing rates are achievable across R1's multiple customer-facing interfaces/VRFs.

---

### Email Thread 8.2 — Congestion Management Comparison

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** WRED vs tail-drop on P routers, and EXP remarking question
>
> On our P routers (R3, R4), we currently use tail-drop during congestion. Someone suggested WRED would be "better" across the board, but I want a real answer about when tail-drop is actually fine and when WRED meaningfully helps, specific to TCP-heavy customer traffic (Customer Green/Yellow/Blue are mostly TCP-based enterprise traffic) versus any UDP-based traffic we might carry (IPTV).
>
> Separately: if a P router like R3 needs to remark MPLS EXP bits mid-path (not at ingress) due to a provisioning mistake upstream, is that something P routers typically do, or is EXP remarking something that should only ever happen at PE ingress? Explain why.

**Question 21:** Explain when WRED provides a real benefit over tail-drop for Meridian's mixed TCP/UDP traffic, and explain whether EXP remarking on P routers (mid-path) is appropriate practice vs. PE-ingress-only remarking, with justification.

---

## SECTION 9 — Security Design (2 Questions)

### Email Thread 9.1 — BGP Session Hardening

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** MD5 isn't enough anymore for our inter-AS eBGP sessions
>
> Our security team flagged that MD5 authentication on our inter-AS eBGP sessions (R6↔R20, R6↔R12, R5↔R11, R12↔R21) uses a weak hash and doesn't support graceful key rollover — we currently have to bounce sessions to change the MD5 key, which has caused outages during past key-rotation attempts.
>
> Explain how TCP-AO (RFC 5925) addresses both weaknesses specifically — the hash algorithm concern and the key-rollover-without-session-bounce concern — and whether TCP-AO is a drop-in replacement (same config model) or requires a materially different configuration approach on our ASBRs.

**Question 22:** Explain how TCP-AO addresses the hash-strength and key-rollover limitations of MD5 for Meridian's inter-AS sessions, and describe how its configuration model differs from MD5.

---

### Email Thread 9.2 — Flowspec and RPKI for DDoS/Hijack Mitigation

> **From:** Daniel Osei, Peering & Policy Engineer
> **To:** SP Design Consultant
> **Subject:** Automating DDoS response and preventing route hijacks
>
> Two unrelated-but-adjacent asks from leadership this week:
> 1. During a recent volumetric attack against a Customer Blue prefix, it took our NOC 25 minutes to manually push an ACL to drop attack traffic at the edge. We want this automated via BGP so a trigger router can push a drop/rate-limit rule network-wide in seconds.
> 2. Separately, we had a near-miss where another ISP briefly originated one of our Customer Yellow prefixes by typo. We want cryptographic validation that we are the legitimate origin AS for prefixes we announce.
>
> For #1, explain how BGP Flowspec would let us automate the DDoS response, including what kind of match criteria it supports beyond just prefix (since attack traffic often needs to be matched on more than destination prefix alone), and the main operational risk of Flowspec if a filter rule is too broad. For #2, explain how RPKI (ROAs, Route Origin Validation) would have caught the Customer Yellow hijack, and what RPKI does NOT protect against (i.e., what hijack types it's blind to).

**Question 23:** Explain BGP Flowspec's match capabilities and automation value for DDoS response plus its main operational risk, and explain how RPKI/ROV would address the Customer Yellow near-miss while identifying what RPKI does not protect against.

---

## SECTION 10 — L2VPN Design (2 Questions)

### Email Thread 10.1 — Modernizing Customer Red's L2 Service

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** Customer Red's VPWS — should we move off T-LDP to EVPN-VPWS?
>
> Customer Red's cross-AS L2 service (spanning AS 65100 and AS 65200, terminating around R1/R14/R16) currently uses classic T-LDP-signaled pseudowires. We're evaluating EVPN-VPWS as the modern replacement, mainly for better multihoming (today, if the PE facing one of Red's sites fails, there's no clean active/standby or active/active redundancy — just a dead circuit until IGP/LDP reconverges to a backup PW if one even exists).
>
> Explain the core signaling difference between T-LDP VPWS and EVPN-VPWS (what protocol carries the pseudowire label binding in each case), and specifically how EVPN-VPWS enables multihoming redundancy for Customer Red's sites in a way that classic T-LDP VPWS structurally cannot do cleanly.

**Question 24:** Explain the signaling difference between T-LDP VPWS and EVPN-VPWS, and explain how EVPN-VPWS structurally enables multihoming redundancy that classic T-LDP VPWS cannot provide cleanly.

---

### Email Thread 10.2 — Pseudowire Redundancy and MTU Mismatch

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** PW setup keeps failing for new Red site, and we need backup PWs
>
> Two Customer Red issues this week:
> 1. We tried to add a new PW to R16 for a new Red site, and it's failing to come up. The far-end PE's interface MTU doesn't match ours exactly, and I recall PW setup is supposed to negotiate this somehow via T-LDP, but it's clearly not succeeding here. What's actually exchanged during PW signaling regarding MTU, and what happens (does the PW fail entirely, or does it come up in a degraded way) when there's a true mismatch that can't be resolved?
> 2. Separately — we want PW redundancy so if the primary PE for a Red site fails, a backup PW to a secondary PE takes over. What's the mechanism (and relevant signaling, like the use of a specific LDP status/preferential-forwarding indication) that allows a backup PW to exist in a "standby" state and activate without requiring the CE to run any special protocol?

**Question 25:** Explain the pseudowire MTU negotiation process during T-LDP signaling and the outcome of a true mismatch, and explain the PW redundancy mechanism (standby PW activation signaling) requested for Customer Red.

---

## SECTION 11 — Segment Routing Design (3 Questions)

### Email Thread 11.1 — SR-MPLS Coexistence During Migration

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** Migrating AS 65100 from LDP to SR-MPLS without a flag day
>
> We want to migrate AS 65100's core (R1–R8) from LDP to SR-MPLS, but we can't do a flag-day cutover — R3 and R4 (P routers) need to support both LDP-signaled and SR-signaled LSPs simultaneously during a multi-month migration window, including for traffic whose ingress PE (R1 or R2) has already converted to SR while some egress point hasn't yet.
>
> Explain how LDP and SR-MPLS can coexist on the same router/interface during migration (what determines which label a given FEC uses, and whether there's an interworking mechanism needed at the LDP/SR boundary, like a mapping server, specifically for this scenario — or is a mapping server for something else entirely?).

**Question 26:** Explain how LDP and SR-MPLS coexist on R3/R4 during AS 65100's migration (FEC-to-label-source determination) and clarify the actual purpose of an SR mapping server in this context.

---

### Email Thread 11.2 — Fast Reroute Comparison for SR

> **From:** Priya Nathan, North Region NOC Lead
> **To:** SP Design Consultant
> **Subject:** TI-LFA vs remote-LFA now that we're moving to SR
>
> Once AS 65100 is on SR-MPLS, we want sub-50ms local protection similar to what RSVP-TE FRR gave us, but without maintaining TE tunnels. I keep seeing "remote LFA" and "TI-LFA" mentioned as if they're the same thing, but I don't think they are.
>
> Explain the key limitation of classic LFA that remote LFA was built to address, how TI-LFA improves further on remote LFA (specifically regarding post-convergence path alignment), and whether TI-LFA requires SR to function or could theoretically also work in an LDP-only network.

**Question 27:** Explain classic LFA's key limitation, how remote LFA addresses it, how TI-LFA improves further (post-convergence alignment), and whether TI-LFA has a hard dependency on SR.

---

### Email Thread 11.3 — SR-TE with On-Demand Nexthop and PCE

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** Automating low-latency paths for Customer Blue without manual tunnel config
>
> Customer Blue has asked for a low-latency path guarantee between their AS65100 and AS65300 sites. Rather than hand-building explicit SR-TE policies on R1 and R21, we want this automated: when a VPNv4 route for Blue is learned with a specific color/intent community, the ingress PE should automatically steer that traffic onto an SR-TE policy computed to satisfy the latency constraint, using a centralized PCE rather than relying purely on distributed IGP extensions.
>
> Explain how SR-TE On-Demand Nexthop (ODN) triggers automatic policy instantiation from a VPNv4 color community, and what role the PCE plays versus what the headend router computes on its own (i.e., what's delegated to the PCE and why that's useful for a constraint like latency that IGP metrics alone may not capture).

**Question 28:** Explain the SR-TE ODN mechanism (color community triggering automatic policy instantiation) for Customer Blue, and explain the PCE's role versus local headend computation, specifically for a latency-constrained path.

---

## SECTION 12 — SRv6 Design (1 Question)

### Email Thread 12.1 — SRv6 Locator Planning for a Greenfield Expansion

> **From:** Marcus Webb, Senior Network Architect
> **To:** SP Design Consultant
> **Subject:** SRv6 for our planned 4th region (greenfield, no legacy MPLS)
>
> We're planning a brand-new 4th regional AS with no legacy MPLS equipment — a clean opportunity to deploy SRv6 instead of SR-MPLS. Two questions before we finalize addressing:
>
> 1. We keep hearing "uSID" / "compressed SID" (C-SID) mentioned as critical for locator planning at scale. Explain why plain SRv6 SIDs (128-bit, one per segment) become a scaling concern in the data plane/header size, and how uSID/C-SID addresses that.
> 2. For an SRv6-based L3VPN in this new region (functionally equivalent to what we do today with MPLS L3VPN + VPNv4), explain at a high level how the VPN label function is realized without an MPLS label stack, and what "PSP" and "USD" behaviors mean for the end-of-path SID processing (what gets popped/removed and by which node).

**Question 29:** Explain why uncompressed SRv6 SIDs create data-plane scaling concerns and how uSID/C-SID addresses it, then explain how an SRv6-based L3VPN realizes the VPN function without an MPLS label stack, including what PSP and USD behaviors do at the end of the SID list.

---

## SECTION 13 — Automation & Operations (1 Question)

### Email Thread 13.1 — Telemetry and Service Lifecycle Automation

> **From:** Elena Vasquez, Central Region Design Lead
> **To:** SP Design Consultant
> **Subject:** Moving off SNMP polling, and automating VRF provisioning
>
> Two related modernization asks:
> 1. We want to replace SNMP polling of interface counters on R11–R19 with model-driven telemetry. Our monitoring vendor supports both gRPC dial-in and dial-out — explain the architectural difference (who initiates the session, and why that matters for a network where some routers sit behind strict management-plane ACLs that only allow outbound-initiated connections from the router itself).
> 2. For provisioning new customer VRFs across R14/R15/R16 consistently (to avoid the kind of policy drift Daniel mentioned for BGP communities), we're considering an NSO-based service model driven by YANG. Explain at a high level what "service lifecycle" means in this context (create/modify/delete and reconciliation with device state) and why a YANG-modeled service approach reduces the drift problem compared to engineers hand-editing CLI on each PE.

**Question 30:** Explain the architectural difference between gRPC telemetry dial-in and dial-out and why it matters for ACL-restricted management access, and explain the NSO/YANG service lifecycle concept and why it reduces configuration drift compared to manual CLI provisioning.

---

## End of Phase 2 Document

**Total questions: 30**, distributed as: IS-IS (2), OSPF (3), BGP (4), MPLS-TE (2), L3VPN (3), Inter-AS (3), Multicast (2), QoS (2), Security (2), L2VPN (2), Segment Routing (3), SRv6 (1), Automation/Operations (1).

No answers are provided in this document, consistent with CLC exam-prep format. Attempt all questions under timed conditions before consulting solution references.
