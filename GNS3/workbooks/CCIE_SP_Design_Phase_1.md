# CCIE Service Provider — Design Phase 1
## Atlas Global Networks — Multi-AS Service Provider Design Scenario

**Exam format:** CCIE SP Lab — Design Module (modeled after the CLC CCIE Enterprise Infrastructure Design exam)
**Candidate role:** Network Design Consultant engaged by Atlas Global Networks
**Primary contact:** James Carvalho, Director of Network Architecture — `james@ccie-sp-lab.com`

---

## Scenario Overview

Atlas Global Networks (AGN) is a multi-national service provider formed through the merger of three regional carriers. Each region operates as its own Autonomous System and retains its own IGP heritage from its pre-merger engineering culture. You have been retained as the design consultant of record for the integration program. Throughout this exam you will receive emails and resources from AGN network operations staff. Each item asks you to evaluate the resource(s) provided and answer the associated design question(s).

### Topology Summary (GNS3 Lab — 31 Routers)

| Region | AS | IGP | Routers | Route Reflectors | Role |
|---|---|---|---|---|---|
| Emerald | 65100 | IS-IS (L2 core) | 8 | Dual RR (RR1-EMR, RR2-EMR) | Legacy long-haul backbone, oldest region |
| Garnet | 65200 | OSPF (Area 0) | 7 | Single RR (RR1-GRN) | Acquired metro/access-heavy carrier |
| Kyanite | 65300 | IS-IS (L2 core) | 5 | Single RR (RR1-KYT) | Newest region, greenfield build |

Remaining 11 routers are inter-AS border (ASBR) pairs, customer-facing PE routers, and CE devices representing the four anchor customers below. Four physical inter-AS links interconnect the three regions:

- **Link 1:** Emerald ASBR1 (AS 65100) ↔ Garnet ASBR1 (AS 65200)
- **Link 2:** Emerald ASBR2 (AS 65100) ↔ Kyanite ASBR1 (AS 65300)
- **Link 3:** Garnet ASBR2 (AS 65200) ↔ Kyanite ASBR2 (AS 65300)
- **Link 4:** Emerald ASBR3 (AS 65100) ↔ Garnet ASBR3 (AS 65200) — diverse-path backup to Link 1

### Anchor Customers

| Customer | Service | Footprint |
|---|---|---|
| Green Corp | L3VPN | Multi-AS (Emerald + Garnet) |
| Yellow Corp | L3VPN | Single-AS (Garnet only) |
| Blue Corp | L3VPN | Multi-AS, spans all three regions (Emerald + Garnet + Kyanite) |
| Red Corp | L2VPN / VPLS | Multi-site, Emerald + Kyanite |

---

## Instructions

- Each item presents one or more resources (emails, configuration excerpts, topology diagrams in text form) followed by 1–3 design questions.
- Questions may be standard multiple choice (select one), "choose two" / "choose three," ordered-step (drag-and-drop equivalent), or matrix/table matching.
- Answer in the space provided. This document does **not** contain answer keys — refer to the companion **CCIE_SP_Design_Phase_1_Solutions.md** document after completing all items.
- Assume all routers referenced are running current Cisco IOS-XR software unless otherwise stated.
- Unless a question states otherwise, assume the current running configuration reflects the stable, pre-change baseline described in the Topology Summary above.

---

## Section 1: IS-IS Design (Emerald & Kyanite Regions)

---

**Item 1 of 30**

Refer to the new resource(s) available.

Atlas Global Networks is preparing to extend the Kyanite region (AS 65300) from 5 routers to 14 routers over the next two quarters as part of a metro buildout, with new PoPs expected in three distinct geographic clusters. Evaluate the email below and select the design that best satisfies the stated requirements.

Which IS-IS level design should be recommended for the expanded Kyanite region?

A. Keep Kyanite as a single flat Level-2-only domain regardless of size, since IS-IS Level-2 scales linearly with router count and additional levels add unnecessary complexity.

B. Migrate Kyanite to a multi-level design with each geographic cluster as its own Level-1 area, interconnected via a Level-1/Level-2 backbone, so that intra-cluster SPF churn does not propagate across the whole domain.

C. Convert Kyanite to Level-1-only operation and rely on a default route injected by the Level-1/Level-2 border routers, eliminating the need for a Level-2 backbone entirely.

D. Leave Kyanite as Level-2-only but reduce the SPF hold-down timers to compensate for the larger topology, avoiding the operational overhead of introducing Level-1 areas.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Kyanite Metro Expansion — IGP Scaling Concerns**
From: Priya Natarajan, Kyanite Regional Engineering Lead <priya.natarajan@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Kyanite Metro Expansion — IGP Scaling Concerns

James,

We're about to triple the size of the Kyanite region. Right now we're flat Level-2-only across all 5 routers, and that's worked fine, but I'm nervous about what happens when we're at 14 routers across three metro clusters (North Cluster, Central Cluster, Harbor Cluster) all tied back to the two inter-AS ASBRs.

My biggest worry is that a fiber cut or a flapping access ring inside one cluster is going to force SPF recomputation across the entire 14-router domain, including routers in clusters that have nothing to do with the failure. We've seen this bite one of our peer SPs during a similar buildout. I want your formal recommendation on how we should structure the IS-IS hierarchy before we start turning up the new PoPs, including whether a backbone is actually necessary given how small we still are compared to Emerald.

Thanks,
Priya

---

**Item 2 of 30**

Refer to the new resource(s) available.

Atlas Global's security team flagged during a merger-integration audit that IS-IS adjacencies across all three regions are running without any authentication, and that the Emerald region is still using the original narrow (old-style) metric style inherited from its legacy carrier, while Kyanite was built using wide metrics from day one.

Select **two** actions that correctly address the issues raised in the audit email below. (Choose two.)

A. Enable IS-IS HMAC-MD5 authentication at the interface level only, and skip domain/area-level authentication since interface-level is sufficient to prevent all forms of LSP database corruption.

B. Migrate Emerald from narrow to wide metrics using the two-phase approach (enable wide-style reception first, then transition origination) to avoid a routing black hole during the metric style transition.

C. Enable IS-IS authentication using a cleartext password at Level-2 only, since Level-1 authentication is not supported in a Level-2-only domain and cleartext is acceptable because the core is a private MPLS backbone.

D. Immediately set all Emerald interfaces to `metric-style wide` in a single maintenance window without a transition phase, since IS-IS wide and narrow metrics are fully backward compatible on the same circuit.

E. Enable authentication for IS-IS Hello PDUs, LSPs, and SNPs, using a key-chain-based mechanism to support non-disruptive key rollover.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Merger Integration Audit — IS-IS Findings**
From: Marcus Webb, Atlas Global Networks Security & Compliance <marcus.webb@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Merger Integration Audit — IS-IS Findings

James,

Pulling you in because this is squarely a design question, not just a config fix. Our integration audit of the Emerald and Kyanite IS-IS domains turned up two findings I need your formal recommendation on before we close out the audit ticket:

1. Finding SEC-114: No IS-IS authentication configured anywhere in Emerald or Kyanite — Hello, LSP, or SNP level. Flagged as a route-forgery/DoS risk given these are now going to be interconnected regions post-merger rather than isolated carriers.

2. Finding SEC-115: Emerald is still running narrow-style (old, 6-bit) metrics across all 8 routers — a holdover from the original carrier's config standards. Kyanite was built fresh and uses wide-style (24-bit, TE-capable) metrics. This mismatch is going to be a real problem once we start running MPLS-TE across the merged core, since wide metrics are required for TE link attributes.

I need to know the safe migration path for the metric style change (we cannot afford a routing black hole during a cutover) and the right authentication approach that supports future key rotation without a flag day across 8 production routers.

Marcus

---

**Item 3 of 30**

Refer to the new resource(s) available.

Review the IS-IS Level-1/Level-2 placement question from the Emerald region lead below, then complete the matching exercise.

**Matrix/Table Matching:** For each Emerald router role described in Column A, select the correct IS-IS level configuration from Column B. Each item in Column B may be used more than once or not at all.

| Column A — Router Role | Column B — Candidate IS-IS Level Configuration |
|---|---|
| 1. Core router with no inter-AS or customer-facing links, interior to the Emerald backbone | a. Level-1-only |
| 2. ASBR1-EMR (inter-AS link to Garnet, carries BGP-LU and iBGP to RR1-EMR) | b. Level-2-only |
| 3. PE router facing Green Corp and Blue Corp VRFs, interior to Emerald, no inter-AS links | c. Level-1/Level-2 (L1L2) |
| 4. RR1-EMR / RR2-EMR (dual route reflectors for the Emerald iBGP mesh) | |

Provide your mapping (e.g., "1 → b") for all four rows.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Emerald IGP Level Placement for the Merger Cutover**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Emerald IGP Level Placement for the Merger Cutover

James,

Emerald has run Level-2-only since it was built — that was fine when we were a standalone carrier with no concept of "areas." Now that we're part of AGN and inter-connecting with Garnet and Kyanite, I want a second opinion on whether every router in Emerald really needs to carry the full Level-2 link-state database, or whether we should be running some of these routers Level-1-only to shrink the LSDB they need to maintain, now that we have a legitimate hierarchy to work with post-merger.

Specifically I'm trying to figure out the right level for:
- Pure interior P routers (no customer, no inter-AS)
- The three ASBRs that face Garnet and Kyanite
- PE routers serving Green Corp and Blue Corp that don't touch the other ASes directly
- Our two route reflectors, RR1-EMR and RR2-EMR

Want your recommended placement before we touch anything in production.

Devon

---

## Section 2: OSPF Design (Garnet Region)

---

**Item 4 of 30**

Refer to the new resource(s) available.

Garnet's engineering lead wants to carve the 7-router OSPF Area 0 into a hierarchical design ahead of adding two newly acquired access rings. Review the email and select the correct answer.

Garnet needs to introduce a non-backbone area for a newly acquired access ring that must inject external routes learned from a static redistribution of a legacy billing-system network, while still allowing that area to receive type-5 LSAs from the rest of the domain for route summarization purposes. Which OSPF area type should be used for this new area?

A. Stub area, because stub areas efficiently block all external LSAs and reduce LSDB size, which is the primary goal of the Harbor Ring integration.

B. Totally stub area, since totally stubby areas are the Cisco-recommended default for any newly acquired access ring regardless of redistribution requirements.

C. NSSA (Not-So-Stubby Area), because the area needs to originate type-7 LSAs for the redistributed billing-system routes (translated to type-5 at the ABR) while still being able to receive summarized type-5 LSAs from the backbone.

D. Standard (non-stub, non-NSSA) area, because any area that redistributes external routes must remain a standard area and cannot use NSSA or stub functionality.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Harbor Ring Acquisition — OSPF Area Design**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Harbor Ring Acquisition — OSPF Area Design

James,

We just closed on the Harbor Ring access network — six small aggregation routers in a metro ring, currently running RIP(!) against a legacy billing-system host network that we have to keep reachable during the transition. I want to bring this ring in as its own OSPF area off our existing Area 0.

Requirements I need satisfied:
1. The billing-system network has to be redistributed as an external route from inside this new area.
2. We still want the area to be as "stubby" as reasonably possible to protect the six aggregation routers from carrying the full backbone LSDB — most of them are old boxes with limited memory.
3. The area still needs to see summarized reachability (type-5-originated routes) from the rest of Garnet, just not the full unsummarized table.

What area type do you recommend, and why does it satisfy all three requirements where a plain stub area would not?

Felicia

---

**Item 5 of 30**

Refer to the new resource(s) available.

Garnet's single ABR design has become a concern after a recent partial outage. Review the resource below, then answer both questions.

**Question 5a (Multiple Choice):** Garnet currently has exactly one ABR connecting the newly created Harbor Ring NSSA to Area 0. During the recent fiber cut, Harbor Ring lost all backbone reachability for 40 minutes. Which design change most directly addresses this single point of failure without introducing a virtual link?

A. Configure a second physical connection from a different Harbor Ring router into Area 0 and promote that router to be a second ABR for the area.

B. Configure an OSPF virtual link from a Harbor Ring internal router through a non-backbone transit area to Area 0, since virtual links are designed specifically to provide primary (not just backup) backbone connectivity.

C. Leave the single ABR in place and instead reduce the SPF and LSA throttle timers domain-wide to speed up reconvergence after the ABR fails.

D. Convert Harbor Ring from NSSA back to a standard area so that it can establish a virtual link to any other standard area in the domain, which is a requirement for virtual link configuration.

**Question 5b (Multiple Choice):** Given the SPF timer concern raised in Felicia's follow-up note, which SPF/LSA throttle timer adjustment is most appropriate for Garnet's core, consisting of modern routers on a stable fiber backbone?

A. Leave SPF and LSA throttling at the default conservative values (5000ms initial SPF delay) since Garnet is a small, 7-router domain where convergence speed is not operationally significant.

B. Tune `timers throttle spf` and `timers throttle lsa` to use low initial/min values with exponential backoff (e.g., sub-second initial delay, millisecond-scale min-hold) to achieve fast convergence while still protecting against LSA storms during instability.

C. Disable SPF throttling entirely by setting all throttle values to zero, which is the recommended practice for any backbone with fewer than 10 routers.

D. Increase the LSA throttle max-wait value to the maximum supported setting to minimize CPU load, accepting slower convergence as an acceptable trade-off for a 7-router domain.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Harbor Ring Outage Follow-Up — ABR Redundancy and SPF Timers**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Harbor Ring Outage Follow-Up — ABR Redundancy and SPF Timers

James,

Following up on the Harbor Ring NSSA question — we had our first real incident. The single ABR connecting Harbor Ring to Area 0 went down for scheduled maintenance that ran long, and the whole ring was dark to the rest of Garnet for 40 minutes even though the ring itself was fully intact internally. I want a permanent fix for the single-ABR exposure — and please don't tell me "configure a virtual link," because my understanding is virtual links are meant as a backbone-continuity patch for topology constraints, not as standard redundancy for an area's primary backbone access. Correct me if I'm wrong there.

Separately — once the ABR redundancy fix is in, I also want your opinion on our SPF/LSA throttle timers. We've been running stock defaults since day one. We're only 7 routers on good modern gear with a stable fiber core, and this outage made me realize our convergence behavior has never actually been tuned for our hardware. What should we be running instead?

Felicia

---

## Section 3: BGP Design (Cross-AS)

---

**Item 6 of 30**

Refer to the new resource(s) available.

Review the email from AGN's BGP architecture lead regarding the Emerald dual-RR cluster, then answer the question.

Emerald runs two route reflectors, RR1-EMR and RR2-EMR, both configured with the **same cluster-ID** for redundancy. A new engineer on the team is proposing to split them into two different cluster-IDs "for cleaner separation of RR domains." Which outcome correctly describes the operational impact of this proposed change?

A. No impact — cluster-ID is purely cosmetic and used only for show-command organization; it has no bearing on loop prevention or route propagation.

B. Splitting into two distinct cluster-IDs will cause each RR to treat the other's reflected routes as originating from a different cluster, meaning both RRs may now reflect the same route to each other and to clients, requiring careful verification that this does not create duplicate paths or routing loops that the shared cluster-ID previously prevented via the CLUSTER_LIST loop-detection mechanism.

C. Splitting into two distinct cluster-IDs will immediately cause a routing black hole because RR clients will reject updates from any RR whose cluster-ID differs from the one configured locally on the client.

D. Cluster-ID only matters for eBGP sessions; since RR1-EMR and RR2-EMR peer with each other and their clients exclusively over iBGP, the proposed change has zero effect on CLUSTER_LIST processing.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Emerald RR Cluster-ID Change Proposal**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Emerald RR Cluster-ID Change Proposal

James,

One of our newer engineers submitted a change request to give RR1-EMR and RR2-EMR separate cluster-IDs instead of the shared cluster-ID they've used since Emerald stood up its RR pair. His reasoning in the ticket is "cleaner separation of RR domains, each RR should own its own identity." I want your formal read on this before I approve or reject the change — specifically whether this is safe, harmful, or just a no-op, and why, in terms of what cluster-ID actually does in the CLUSTER_LIST loop-prevention mechanism.

Ingrid

---

**Item 7 of 30**

Refer to the new resource(s) available.

AGN is debating whether iBGP full-mesh or route reflection is appropriate at different points in the merged network. Review the two scenarios below and answer both parts.

**Question 7a (Multiple Choice):** Kyanite currently runs a single RR (RR1-KYT) serving all 5 Kyanite routers via route reflection. During the planned expansion to 14 routers (see Item 1), should Kyanite remain on a single-RR design?

A. Yes — a single RR scales to any number of clients as long as the RR itself has sufficient CPU and memory, and client count has no bearing on RR redundancy requirements.

B. No — a single RR becomes a single point of failure for the entire iBGP topology (all PE/P reachability for VPNv4/VPNv6 and any other AFI/SAFI) as the client count grows; AGN should add a second RR, ideally forming a redundant pair similar to Emerald's RR1-EMR/RR2-EMR model, before completing the expansion.

C. No — Kyanite should abandon route reflection entirely and move to full iBGP mesh once it reaches 14 routers, since full mesh is more scalable than route reflection for any domain under 20 routers.

D. Yes, but only if BGP confederations are also introduced, since route reflection without confederation is not valid in a Level-2-only IS-IS domain.

**Question 7b (Multiple Choice):** AGN's inter-AS Blue Corp VPN design (spanning Emerald, Garnet, and Kyanite) requires BGP-LU (RFC 3107 labeled unicast) to be carried across the ASBR links so that an end-to-end LSP can be built across all three ASes for Inter-AS Option C. Which statement correctly describes the BGP session design required to support this?

A. BGP-LU must be carried over the same iBGP sessions used for VPNv4, so no new sessions are required beyond what already exists for the L3VPN service.

B. BGP-LU requires a multihop eBGP session directly between the egress PE routers in each AS (bypassing the ASBRs entirely), since BGP-LU cannot be exchanged across ASBRs.

C. BGP-LU is distributed via eBGP between adjacent ASBRs (exchanging labeled /32 loopback routes for PEs/ASBRs) and then re-advertised via a separate multihop eBGP (or redistributed into iBGP, depending on the chosen variant) session so that labels for inter-AS PE loopbacks are known end-to-end, which is a defining characteristic of Inter-AS Option C.

D. BGP-LU is unnecessary for Option C; Option C relies exclusively on redistributing PE loopbacks into each AS's IGP with no BGP involvement.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: iBGP Scaling and Inter-AS Label Distribution for Blue Corp**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: iBGP Scaling and Inter-AS Label Distribution for Blue Corp

James,

Two separate but related questions bundled into one email since they're both landing on my desk this week:

1. Kyanite RR redundancy: We talked about the Level hierarchy for the Kyanite expansion already, but nobody has signed off on whether RR1-KYT alone is sufficient once we're at 14 routers. I lean toward "no" but want your formal justification.

2. Blue Corp spans all three ASes and sales has already promised the customer an any-to-any L3VPN experience with minimal latency, which is pushing us toward Inter-AS Option C instead of Option B. If we go Option C, I need to understand exactly how label distribution has to work across the ASBR links — specifically whether we need new BGP sessions beyond our existing VPNv4 RR infrastructure, and what role BGP-LU (RFC 3107) plays in stitching the LSP together end to end.

Ingrid

---

**Item 8 of 30**

Refer to the new resource(s) available.

**Ordered Steps (drag-and-drop equivalent):** AGN is deploying a new eBGP multihop session between a Blue Corp PE in Emerald and a Blue Corp PE in Kyanite that are not directly connected (the session must traverse two P routers that are not BGP speakers for this session). Place the following configuration/validation steps in the correct order to bring this eBGP multihop session into service safely.

Steps (unordered):

- Step W: Configure `neighbor <ip> ebgp-multihop <ttl>` with a TTL value sufficient to cover the actual hop count between the two PE loopbacks.
- Step X: Verify IGP (or static/BGP-LU) reachability of the remote PE's loopback address with the correct recursive next-hop resolution before enabling the BGP neighbor.
- Step Y: Configure `neighbor <ip> update-source Loopback0` on both PEs so that the TCP session sources from the stable loopback rather than a physical interface.
- Step Z: Enable the BGP neighbor and verify session establishment, then confirm prefix exchange and correct next-hop-self or third-party next-hop behavior for the exchanged routes.

Provide the correct order (e.g., "X, Y, W, Z").

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp eBGP Multihop Turn-Up Checklist**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp eBGP Multihop Turn-Up Checklist

James,

We're turning up a new eBGP multihop session for the Blue Corp Option C design between an Emerald PE and a Kyanite PE that don't share a direct link — traffic has to transit two intermediate P routers. I keep seeing junior engineers do this out of order (usually they enable the neighbor before confirming loopback reachability, and then spend an hour debugging "neighbor stuck in Active/Idle" before realizing the recursive lookup was never going to work). I want a clean, ordered checklist I can hand to the turn-up team so this doesn't happen again. Can you lay out the correct sequence of configuration and verification steps?

Ingrid

---

## Section 4: MPLS LDP Design

---

**Item 9 of 30**

Refer to the new resource(s) available.

Garnet's core team reports a recurring issue where IS-IS/OSPF reconverges and installs a new IGP path before LDP has finished building the label mapping for that path, resulting in a transient blackhole for LSP traffic. Review the email and answer the question.

Which feature directly addresses the blackhole condition described, where the IGP installs a route to a new next hop before the corresponding LDP label binding is available?

A. LDP Session Protection, which keeps an LDP session alive across a brief link-down event by maintaining targeted LDP Hellos, but does not address IGP/LDP synchronization during normal reconvergence onto a new path.

B. LDP-IGP Synchronization, which holds the IGP metric for a link artificially high (or holds the link down from an IGP routing perspective) until the LDP session on that link is fully established and label bindings are exchanged, preventing traffic from being forwarded via a path with no label binding yet.

C. Targeted LDP, which is used strictly for non-adjacent (e.g., AToM/VPLS pseudowire) LDP sessions and has no bearing on link-local label synchronization with the IGP.

D. MPLS label filtering via `no mpls ldp advertise-labels`, which would actually make the blackhole worse by preventing label advertisement for the affected prefixes entirely.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Transient LSP Blackholes During IGP Reconvergence**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Transient LSP Blackholes During IGP Reconvergence

James,

We're seeing a pattern during planned maintenance windows: when we shift traffic by adjusting an IGP metric, OSPF converges fast (we tuned the timers, remember), but LDP hasn't finished exchanging label bindings with the new next-hop neighbor yet. For a window of a second or two, we get real packet loss on LSP-forwarded traffic because the FIB has an IGP route installed with no usable label. This is exactly the kind of thing that would get flagged hard in a customer SLA review. What's the right LDP feature to prevent the IGP from "winning the race" against LDP during these transitions?

Felicia

---

**Item 10 of 30**

Refer to the new resource(s) available.

Review the resource below describing AGN's planned Red Corp VPLS pseudowire mesh across non-directly-connected PE routers, then answer the question.

Red Corp's VPLS full mesh (see Section 8) requires LDP label bindings between PE routers in Emerald and Kyanite that are not IGP-adjacent to each other (they are several IGP hops apart, connected only through transit P routers). Which LDP mechanism must be configured between these non-adjacent PEs to support the pseudowire label exchange, and which filtering practice should accompany it to protect the P routers from unnecessary label state?

A. Standard link Hello-based LDP discovery between the PEs directly, combined with `mpls ldp neighbor password` for security; no additional filtering is needed since link LDP automatically scopes itself to only the required prefixes.

B. Targeted LDP (extended/directed Hello) sessions between the non-adjacent Red Corp PE loopbacks to establish the label exchange needed for the pseudowires, combined with label filtering (e.g., `mpls ldp label` inbound/outbound ACL-based or `no mpls ldp advertise-labels` scoped to an access-list) on the P routers so that only PE loopback /32s relevant to pseudowire signaling receive distributed label bindings, reducing unnecessary LDP state on transit P routers.

C. BGP-signaled pseudowires exclusively, since targeted LDP is deprecated in favor of BGP for all AToM and VPLS signaling in current IOS-XR releases.

D. No special LDP configuration is required; because the PEs are reachable via the IGP, standard link LDP Hellos sent on every core-facing interface will automatically establish the needed label bindings between non-adjacent PEs without any targeted session.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Red Corp VPLS — Non-Adjacent PE Label Distribution**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Red Corp VPLS — Non-Adjacent PE Label Distribution

James,

Getting ahead of the VPLS design work for Red Corp. Their full-mesh VPLS instance needs pseudowires between PE routers in Emerald and Kyanite, and those PEs are not IGP-adjacent — there are P routers in between. I know we need some form of non-link-local LDP session for this, but I also don't want every P router in the path building label state for every PE loopback in the network if it doesn't need to. What's the right combination of LDP session type and label filtering practice here?

Devon

---

## Section 5: L3VPN Design

---

**Item 11 of 30**

Refer to the new resource(s) available.

AGN's VPN provisioning team is standardizing the RD/RT numbering scheme ahead of onboarding several more customers after Green Corp, Yellow Corp, and Blue Corp. Review the email and answer the question.

Which RD/RT numbering scheme best satisfies the stated requirements of (1) guaranteed RD uniqueness per VRF instance across all three ASes without inter-region coordination overhead, and (2) a scalable RT scheme that supports Blue Corp's any-to-any multi-AS topology?

A. Use the Type 0 RD format (2-byte ASN:4-byte number) with the same arbitrary number (e.g., `100:1`) reused across all regions for the same customer, and a single shared RT identical to the RD value for both import and export on every PE, relying on manual coordination to avoid collisions.

B. Use the Type 1 RD format (4-byte IPv4 address:2-byte number), where the IPv4 address is each PE's unique loopback address combined with a locally significant number, guaranteeing RD uniqueness without inter-region coordination since no two PEs share a loopback; for Blue Corp, use a single common RT imported/exported by every PE in the any-to-any mesh (hub-less full mesh), reserving distinct RTs only for customers requiring hub-and-spoke or extranet policies.

C. Use the Type 1 RD format but intentionally reuse the same RD value across every PE serving a given customer so that routes can be more easily aggregated at the RR, understanding that RD uniqueness per PE is only a "best practice" and not a functional requirement for BGP path selection.

D. Avoid RTs entirely for Blue Corp and instead rely exclusively on RD-based filtering at the RR to control which PEs receive which VPNv4 routes, since RT import/export policy is redundant once RDs are unique.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: RD/RT Numbering Standard for New Customer Onboarding**
From: Priya Natarajan, Kyanite Regional Engineering Lead <priya.natarajan@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: RD/RT Numbering Standard for New Customer Onboarding

James,

We're about to onboard four more L3VPN customers behind Green Corp, Yellow Corp, and Blue Corp, and every region has been winging it on RD/RT numbering so far — Emerald uses one convention, Garnet uses another informally. Before this gets worse I want a single documented standard that (a) guarantees RD uniqueness per VRF instance without the three regional teams having to call each other every time someone provisions a new VRF, and (b) gives us a clean RT scheme that supports Blue Corp's requirement of true any-to-any reachability across all three ASes without needing a hub site. What's the right scheme?

Priya

---

**Item 12 of 30**

Refer to the new resource(s) available.

Blue Corp has a site dual-homed into two different Emerald PEs for resiliency. Review the two scenarios below and answer both parts.

**Question 12a (Multiple Choice):** Blue Corp's Kyanite site and Garnet site are both directly connected to AS 65300 and AS 65200 respectively via eBGP PE-CE sessions, and Blue Corp uses the **same AS number** on the CE routers at both sites (a common real-world occurrence when a customer reuses one private ASN across all their sites). Without any mitigation, which problem will occur when Blue Corp's Kyanite-originated route is advertised toward the Garnet site?

A. No problem occurs; AS number reuse on CE routers has no effect on L3VPN route propagation since VPNv4 routes are exchanged over MP-BGP, not plain eBGP, between PEs.

B. The Garnet PE will reject the route due to AS-path loop detection, because the Garnet-side eBGP session to the Blue Corp CE will see its own (the CE's) ASN already present in the AS-path and discard the route as a loop, since both CE sites use the same ASN.

C. The route will be accepted but marked with the `NO_EXPORT` community automatically, silently preventing it from being re-advertised to any other site.

D. The route will loop indefinitely between the two PEs until TTL expiration, since L3VPN eBGP sessions do not perform AS-path loop detection by design.

**Question 12b (Multiple Choice):** Which PE-side configuration options correctly resolve the AS-path loop condition identified in 12a? (Choose two.)

A. `neighbor <CE> as-override` on the Garnet PE's PE-CE eBGP session, which replaces occurrences of the CE's ASN in the AS-path with the PE's own (provider) ASN before advertising, so the route no longer appears to loop back to the same CE ASN.

B. `neighbor <CE> allowas-in <n>` on the Garnet PE's PE-CE eBGP session, which permits the route to be accepted despite containing the local (CE) ASN up to `n` occurrences, bypassing the loop-detection check for that specific neighbor.

C. `neighbor <CE> remove-private-as` on the Garnet PE's PE-CE session, since Blue Corp's CE ASN falls in the private ASN range and removing it from the AS-path resolves the duplicate-ASN condition.

D. Disabling eBGP entirely between the PE and CE in favor of static routes, which is the only supported workaround for duplicate CE ASNs in a multi-site L3VPN.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp Duplicate CE ASN Across Sites**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp Duplicate CE ASN Across Sites

James,

Blue Corp's provisioning paperwork just came back and their network team used the exact same private ASN (65501) on the CE routers at both their Kyanite site and their Garnet site. I already know this is going to cause an AS-path loop problem once we provision the second site's PE-CE eBGP session — routes from one site will look like they're looping back through "their own" ASN when seen from the other site. I need the correct fix (or fixes) at the PE, and I want to understand the actual mechanical difference between the two most common knobs people reach for here, because I've seen engineers use them interchangeably and I don't think that's actually correct.

Devon

---

**Item 13 of 30**

Refer to the new resource(s) available.

Yellow Corp runs OSPF as its PE-CE routing protocol at two sites within Garnet, and has reported that routes learned at Site A are not being correctly recognized as "external to the OSPF domain" when they reappear at Site B after transiting the MPLS VPN backbone and OSPF superbackbone (sham-link-free, standard PE-CE OSPF scenario). Review the resource and answer the question.

Which mechanism is responsible for allowing a Yellow Corp OSPF Type-3 (or Type-5/7, depending on configuration) LSA regenerated at Site B's PE to be correctly recognized by that site's CE as a route that originated from a different OSPF domain reachable only via the VPN backbone, preventing a routing loop or incorrect intra-area preference?

A. The OSPF **DN-bit** (Down bit) set by the originating PE on LSAs regenerated from VPNv4/BGP routes into the PE-CE OSPF process, which signals to downstream OSPF routers (including other PEs) that the LSA originated from the MPLS VPN backbone and should not be redistributed back into the backbone's own OSPF domain, preventing routing loops across multi-site OSPF VPN sites.

B. The OSPF **MC-bit** (Multicast capable bit), which is unrelated to VPN loop prevention and instead governs whether a router participates in OSPF multicast extensions.

C. A manually configured route-map on the PE denying all Type-3 LSAs inbound from the VPN backbone, which is the only supported mechanism for this scenario since the DN-bit is not supported by any current PE-CE OSPF implementation.

D. The BGP `ORIGINATOR_ID` attribute, which is reused by OSPF superbackbone PEs to mark LSAs as VPN-originated.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Yellow Corp Multi-Site OSPF PE-CE Loop Prevention**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Yellow Corp Multi-Site OSPF PE-CE Loop Prevention

James,

Yellow Corp runs OSPF (not eBGP) as their PE-CE protocol at both of their Garnet sites. I was reviewing their design docs and I see mention of "the DN-bit" in an old vendor whitepaper about multi-site OSPF VPNs but no real explanation of what it actually does operationally. Can you confirm: is this the mechanism that prevents Site B from getting confused about a route that actually originated at Site A and came back through the VPN backbone and the OSPF superbackbone concept? I want to make sure we have this correctly enabled (it should be automatic when a VRF is configured with an OSPF PE-CE process in IOS-XR, but I want it confirmed) before Yellow Corp adds their third site next quarter.

Felicia

---

## Section 6: Inter-AS VPN Design

---

**Item 14 of 30**

Refer to the new resource(s) available.

AGN's leadership is requesting a trade-off comparison of the three Inter-AS VPN interconnect options for Green Corp (Emerald + Garnet only — a simpler two-AS case) ahead of a customer-facing architecture review. Review the email and complete the matrix.

**Matrix/Table Matching:** For each Inter-AS VPN option in Column A, match the single best description of its primary operational trade-off from Column B.

| Column A — Option | Column B — Candidate Trade-Off Description |
|---|---|
| 1. Option A (VRF-to-VRF, back-to-back) | a. Requires exchanging labeled VPNv4 routes directly between ASBRs via multihop MP-eBGP, with the ASBRs holding full VPNv4 routes for the shared customers only, offering a middle ground between per-VRF scaling and full-mesh PE knowledge. |
| 2. Option B (eBGP redistribution of labeled VPNv4 routes between ASBRs) | b. Simplest to configure and most isolated per customer, but scales poorly as customer count grows because each shared customer requires a dedicated VRF and sub-interface (or interface) on both ASBRs, and provides no end-to-end LSP (traffic is forwarded at Layer 3 at the AS boundary). |
| 3. Option C (multihop MP-eBGP between PE routers, with BGP-LU establishing the end-to-end LSP) | c. Provides the most scalable and lowest-latency any-to-any experience with a true end-to-end LSP between PEs, but requires the most inter-AS trust and coordination (BGP-LU exchange, PE-to-PE MP-eBGP, consistent RT policy awareness across ASes) and is the most operationally complex to secure and troubleshoot. |

Provide your mapping (e.g., "1 → b").

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Inter-AS Option Trade-Off Summary for Executive Review**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Inter-AS Option Trade-Off Summary for Executive Review

James,

I have an executive review on Thursday and I need a crisp, defensible one-paragraph-per-option summary of Options A, B, and C for Green Corp's two-AS interconnect (Emerald/Garnet). I don't need the full configuration detail, I need the trade-off framing: scalability, operational complexity, trust/security boundary implications, and whether each option provides an end-to-end LSP or not. Can you give me the matched comparison in a form I can drop straight into slides?

Ingrid

---

**Item 15 of 30**

Refer to the new resource(s) available.

Blue Corp's three-AS footprint has forced AGN to choose an Inter-AS option that will work consistently across all three region-pairs (Emerald-Garnet, Emerald-Kyanite, Garnet-Kyanite). Review the email and answer the question.

Blue Corp requires low-latency, true any-to-any connectivity across all three regions with a single end-to-end LSP per customer flow (no intermediate Layer-3 lookup/re-encapsulation at any ASBR), and AGN's security team has already approved the inter-AS trust model needed to support PE-to-PE MP-eBGP sessions with BGP-LU across all three ASBR pairs. Which Inter-AS option should be selected for Blue Corp, and which supporting mechanism is mandatory on each ASBR pair to realize the "single end-to-end LSP" requirement?

A. Inter-AS Option A, with each ASBR pair carrying one VRF per shared customer; the end-to-end LSP requirement is satisfied automatically because Option A always forwards natively at Layer 3 across the AS boundary without requiring any additional label distribution mechanism.

B. Inter-AS Option B, with eBGP redistribution of labeled VPNv4 routes between adjacent ASBRs; next-hop-self is applied at each ASBR so the LSP is regenerated hop-by-hop at every AS boundary, which is functionally equivalent to a true end-to-end LSP.

C. Inter-AS Option C, with multihop MP-eBGP carrying VPNv4 routes directly between the ingress and egress PEs, and BGP-LU (RFC 3107) deployed hop-by-hop across each ASBR pair (eBGP between ASBRs, redistributed/re-advertised toward each AS's PEs) so that labeled /32 reachability to every relevant PE loopback is known end-to-end, allowing a single unbroken LSP to be built from ingress PE to egress PE across all three ASes.

D. A hybrid of Option A at one ASBR pair and Option C at another, since Inter-AS options can be freely mixed per AS-pair without any impact on end-to-end LSP continuity.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp Three-AS Interconnect — Final Option Selection**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp Three-AS Interconnect — Final Option Selection

James,

Security has signed off on the trust model needed for PE-to-PE MP-eBGP across all three ASBR pairs (Emerald-Garnet, Emerald-Kyanite, Garnet-Kyanite), so we're clear to pick whichever Inter-AS option gives Blue Corp the best possible any-to-any, low-latency experience. Sales has been explicit that re-encapsulation or Layer-3 lookups at any AS boundary are not acceptable — they want one continuous LSP from ingress PE to egress PE regardless of how many AS boundaries the traffic crosses. Walk me through the option selection and, critically, what has to be running on every single ASBR pair (not just one) to actually deliver that single end-to-end LSP across three ASes, not just two.

Ingrid

---

**Item 16 of 30**

Refer to the new resource(s) available.

Review the resource below regarding RT (Route Target) handling for Green Corp's Option B deployment between Emerald and Garnet ASBRs, then answer the question.

In an Inter-AS Option B deployment, the ASBRs exchange labeled VPNv4 routes via eBGP but by default **do not** necessarily retain the original Route Target community set by the originating PE unless explicitly configured to do so, and RT values may need translation if the two ASes use incompatible RT numbering. Which statement correctly describes the RT handling requirement for Green Corp's Option B deployment across the Emerald/Garnet ASBR pair?

A. RT retention is automatic and mandatory in all Option B implementations; there is no configuration option to translate or strip RTs during ASBR-to-ASBR eBGP exchange, so Emerald and Garnet must use identical RT numbering for Green Corp with no exceptions.

B. The ASBRs must be explicitly configured (e.g., via `retain route-target` policy, or equivalent RT-rewrite/translation route-policy) to either preserve the original RT end-to-end or translate between Emerald's and Garnet's RT numbering schemes if they differ, since RT values are opaque extended communities that the ASBR eBGP process will pass through only according to configured policy, and mismatched/stripped RTs will cause the receiving PE to fail to import the route into the correct VRF.

C. RT values are irrelevant in Option B because the ASBRs make all forwarding decisions based solely on the RD, and RTs are only evaluated by PE routers in Option C deployments.

D. RT translation is only possible in Option A, not Option B, because Option B does not support any form of policy manipulation on exchanged VPNv4 routes.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Green Corp Option B — RT Numbering Mismatch Between Emerald and Garnet**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Green Corp Option B — RT Numbering Mismatch Between Emerald and Garnet

James,

Found a gap during Green Corp's Option B cutover planning: Emerald's RT numbering for Green Corp (`65100:500`) doesn't match what Garnet independently assigned (`65200:500`) before the two regions were talking to each other during pre-merger buildout. I need to know whether this is something that "just works" via Option B's eBGP exchange between ASBRs, or whether we need explicit route-policy to either keep the original RT intact end-to-end or translate between the two numbering schemes. If it needs explicit policy, I want that confirmed clearly since I don't want us assuming it's automatic and then silently dropping routes at the Garnet PEs because the RT doesn't match their VRF import policy.

Ingrid

---

## Section 7: L2VPN / VPWS Design (Red Corp)

---

**Item 17 of 30**

Refer to the new resource(s) available.

Red Corp requires a point-to-point Ethernet pseudowire (VPWS, AToM-signaled) between one Emerald PE and one Kyanite PE. Review the email and answer the question.

Which statement correctly describes the role of the **control word** in this AToM pseudowire, and when AGN should enable it for Red Corp's circuit?

A. The control word is mandatory for all AToM pseudowires regardless of underlying Layer 2 payload type, and disabling it is never supported in any Cisco IOS-XR AToM implementation.

B. The control word is an optional 4-byte field that provides sequencing and payload-type-specific information (such as padding indication for short frames); it should be enabled when the pseudowire type benefits from explicit sequencing/flag handling (and is sometimes required to avoid ECMP-related reordering or misinterpretation of small frames as padding), and both PEs must agree on its use since control-word capability is negotiated during LDP pseudowire signaling.

C. The control word exists only in VPLS pseudowires, not VPWS/AToM point-to-point pseudowires, and is therefore not a relevant consideration for Red Corp's point-to-point circuit.

D. The control word replaces the VC label entirely in modern AToM implementations, meaning Red Corp's circuit will not use a VC label if the control word is enabled.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Red Corp Point-to-Point Pseudowire — Control Word Question**
From: Priya Natarajan, Kyanite Regional Engineering Lead <priya.natarajan@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Red Corp Point-to-Point Pseudowire — Control Word Question

James,

Setting up the AToM pseudowire template for Red Corp's new point-to-point Ethernet circuit between an Emerald PE and a Kyanite PE. Our standard template has the control word disabled by default (that's just how our Emerald template was built years ago), but I've read that for Ethernet-over-MPLS pseudowires, enabling the control word matters for frame sequencing and can prevent certain small frames from being misread. Can you clarify exactly what the control word does functionally and whether we should be enabling it for this circuit, and confirm whether this needs to match on both ends or is independently configurable per PE?

Priya

---

**Item 18 of 30**

Refer to the new resource(s) available.

Red Corp's Ethernet handoff at the Kyanite site uses a non-standard MTU (9000-byte jumbo frames for a backup/replication application) while the Emerald site handoff is standard 1500-byte Ethernet. Review the email and answer the question.

Given this MTU mismatch between the two Red Corp VPWS endpoints, which outcome correctly describes the AToM pseudowire's behavior, and what is the correct remediation?

A. AToM automatically fragments and reassembles oversized frames at the egress PE regardless of configured interface MTU, so no explicit MTU alignment is required between the two attachment circuits.

B. The pseudowire will not come up (or will come up but silently drop oversized frames) unless the interface MTU (and, where applicable, the pseudowire class / xconnect MTU parameter) is configured consistently on both PEs to accommodate the larger 9000-byte frame size end-to-end, including any core-facing MTU/MPLS MTU needed to carry the larger encapsulated frame without fragmentation across the MPLS core.

C. MTU mismatches are only a concern for VPLS, not VPWS, because VPWS point-to-point pseudowires do not perform any MTU validation between the two attachment circuits.

D. The correct remediation is to enable the control word, which automatically normalizes MTU differences between mismatched attachment circuits.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Red Corp Jumbo Frame Handoff — Pseudowire MTU Concern**
From: Priya Natarajan, Kyanite Regional Engineering Lead <priya.natarajan@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Red Corp Jumbo Frame Handoff — Pseudowire MTU Concern

James,

Found something in Red Corp's circuit design doc that worries me: their Kyanite-side handoff is provisioned for 9000-byte jumbo frames (apparently for some backup/replication traffic), but their Emerald-side handoff is just standard 1500-byte Ethernet. Before we turn this pseudowire up, I want to understand exactly what happens if we leave this mismatched — does AToM just handle it transparently, or are we going to get a circuit that looks "up" in `show xconnect` but silently eats the big frames? And if it's the latter, what do we actually need to configure, including anything on the MPLS core-facing interfaces, to support this end to end?

Priya

---

## Section 8: VPLS Design (Red Corp Multipoint)

---

**Item 19 of 30**

Refer to the new resource(s) available.

Red Corp is adding a fourth site, which will push their VPLS instance from 3 PEs to 4 PEs, all within Emerald and Kyanite. Review the email and answer the question.

Red Corp's VPLS instance currently uses a full-mesh pseudowire topology across 3 PEs (3 pseudowires total). Adding a 4th PE under a continued full-mesh model would require 6 pseudowires total, and Red Corp has indicated plans to eventually reach 8 sites. Which statement correctly evaluates whether to continue scaling full-mesh VPLS versus migrating to Hierarchical VPLS (H-VPLS), and what split-horizon behavior must be preserved in either case?

A. Full-mesh VPLS scales linearly (N pseudowires for N PEs) and should always be preferred regardless of PE count, since H-VPLS introduces unnecessary complexity for any VPLS instance under 50 sites.

B. Full-mesh VPLS scales as N(N-1)/2 pseudowires, becoming operationally unwieldy as Red Corp approaches 8 sites (28 pseudowires); AGN should evaluate migrating to H-VPLS with designated hub PE(s) (N-PEs) and spoke PEs (U-PEs) to reduce the full-mesh requirement to the N-PE core only, while still enforcing split-horizon (a PE must never forward a frame received from one core-facing pseudowire out to another core-facing pseudowire) to prevent Layer 2 loops, which applies identically in both full-mesh and H-VPLS designs.

C. Split-horizon only needs to be enforced in H-VPLS designs; full-mesh VPLS does not require split-horizon because the full mesh itself inherently prevents loops.

D. Migrating to H-VPLS eliminates the need for split-horizon entirely, since H-VPLS uses STP (Spanning Tree Protocol) across the pseudowire mesh instead of split-horizon to prevent loops.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Red Corp VPLS Scaling — Full Mesh vs. H-VPLS**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Red Corp VPLS Scaling — Full Mesh vs. H-VPLS

James,

Red Corp is adding a 4th site to their VPLS instance, and their account team just told us Red Corp is planning for up to 8 sites total within the next 18 months. We're full-mesh today across 3 PEs. I did some back-of-envelope math and the pseudowire count is going to blow up fast if we keep scaling full-mesh. Before we commit to a design for the 4th PE, I want your assessment of whether we should start migrating toward H-VPLS now (even though it adds the complexity of hub/spoke PE roles), and I also want it crystal clear in writing whether split-horizon rules change between the two models, because I've had an engineer on my team insist that "full mesh doesn't need split-horizon since there's no hub to create a loop through," and I want that confirmed or corrected before it ends up in a design doc.

Devon

---

**Item 20 of 30**

Refer to the new resource(s) available.

Red Corp has reported intermittent MAC flapping and flooding across their VPLS instance. Review the email and answer the question.

Which combination of VPLS behaviors most likely explains the symptoms described — intermittent MAC flapping between two PEs and periodic unknown-unicast flooding across the whole VPLS domain?

A. The symptoms are expected and unavoidable in any VPLS deployment, since VPLS does not perform any MAC learning and floods all traffic by design, making flapping and flooding a permanent operational characteristic rather than a fault condition.

B. A Red Corp host (or a misconfigured CE) is likely dual-attached or looping traffic between two PE attachment circuits, causing the same source MAC to be learned via two different pseudowires/attachment circuits in rapid succession (MAC flapping); separately, unknown-unicast flooding across the full mesh is expected VPLS behavior for any destination MAC not yet learned, but persistent/periodic flooding suggests MAC table entries are aging out faster than the traffic pattern justifies, which should prompt a review of the MAC aging timer relative to Red Corp's actual traffic intervals, and a check for an unintended Layer-2 loop at the customer edge.

C. MAC flapping in VPLS can only be caused by a misconfigured pseudowire MTU, and flooding can only be caused by a misconfigured RD, so the fix is purely an MTU/RD review with no relevance to customer-side loops or MAC aging.

D. VPLS instances automatically disable MAC learning once more than 3 PEs are in the full mesh, which explains both symptoms as an inherent scaling limitation rather than a fault.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Red Corp VPLS — MAC Flapping and Flooding Reports**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Red Corp VPLS — MAC Flapping and Flooding Reports

James,

Red Corp's NOC has been reporting weird symptoms for about a week: periodically their traffic looks like it's being flooded everywhere instead of switched directly to the right PE, and separately we're seeing the same source MAC get relearned on two different attachment circuits in quick succession (classic flapping warning in our MAC table logs). I want your read on what's actually going on here mechanically — is this a VPLS core issue, something about our MAC aging timer configuration, or something going on at the customer edge that we should be asking Red Corp's network team about?

Devon

---

## Section 9: MPLS Traffic Engineering

---

**Item 21 of 30**

Refer to the new resource(s) available.

AGN is deploying MPLS-TE with Fast Reroute (FRR) across the Emerald core to protect Blue Corp's latency-sensitive traffic. Review the email and answer both questions.

**Question 21a (Multiple Choice):** Emerald has a single critical long-haul link between two core P routers that carries the majority of inter-region TE tunnels. Which FRR protection type should be applied to this specific link to protect against the failure of the link itself (not the downstream node)?

A. Node protection (next-next-hop, NNHOP) backup tunnel, since node protection always provides a strict superset of link protection and link protection should never be configured independently.

B. Link protection (next-hop, NHOP) backup tunnel, which protects traffic specifically against the failure of the protected link by rerouting around it to the same next-hop node, and is the appropriate baseline protection for a single critical link.

C. No FRR protection is possible for a single physical link; FRR requires at least two parallel physical paths between every protected node pair.

D. BFD alone, configured without any backup tunnel, which is sufficient to achieve sub-50ms protection without requiring a pre-signaled backup LSP.

**Question 21b (Multiple Choice):** Blue Corp's TE tunnels need to avoid relying on static bandwidth guesses that go stale as traffic grows. Which RSVP-TE bandwidth design is most appropriate, and how should the tunnels be made visible to IGP shortest-path routing for traffic not explicitly mapped via policy routing?

A. Configure tunnels with zero reserved bandwidth permanently (treat RSVP purely as a signaling protocol with no admission control), and use `autoroute announce` so that all IGP-computed shortest paths across the network are forced through the TE tunnel regardless of whether the tunnel is actually the shortest IGP path.

B. Configure explicit static bandwidth reservations sized to peak measured traffic, reviewed periodically, combined with `autoroute announce` so the tunnel is treated as a directly connected link of IGP metric cost for SPF purposes by the tunnel headend only (not network-wide), allowing IGP shortest-path traffic at the headend to prefer the tunnel without requiring every router in the network to know about the tunnel's existence.

C. Configure explicit static bandwidth reservations, combined with `mpls traffic-eng forwarding-adjacency`, which is required in all cases and functionally identical to autoroute announce with no behavioral difference.

D. Bandwidth reservation is not supported on RSVP-TE tunnels used for FRR; FRR tunnels must always be configured with unconstrained (unlimited) bandwidth.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp TE/FRR Protection Design for the Emerald Core**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp TE/FRR Protection Design for the Emerald Core

James,

Two things on my plate for the Blue Corp TE rollout:

1. We have one long-haul link in the Emerald core (between two P routers) that's a clear single point of failure for most of our inter-region TE tunnels. I want to confirm the right FRR protection type for that specific link — I have an engineer insisting we should "just do node protection everywhere since it's strictly better," but that doesn't sound right to me for a link-specific concern.

2. Bandwidth sizing — I don't want us hand-picking a static bandwidth number once and never touching it again as Blue Corp's traffic grows, but I also don't fully understand the difference between `autoroute announce` and `forwarding-adjacency` for getting IGP traffic onto these tunnels without doing full policy-routing for every flow. Can you lay out the right approach for both?

Devon

---

**Item 22 of 30**

Refer to the new resource(s) available.

Review the resource below comparing `autoroute announce` and `forwarding-adjacency` in more depth, following up on Devon's question in Item 21.

Which statement correctly distinguishes `autoroute announce` from `mpls traffic-eng forwarding-adjacency` in terms of network-wide visibility of the TE tunnel for IGP SPF calculations?

A. They are functionally identical in every respect; the only difference is the CLI keyword used, with no difference in SPF visibility across the network.

B. `autoroute announce` makes the tunnel usable for shortest-path traffic only at the tunnel headend (the tunnel is treated as a direct link for the headend's own SPF/routing-table calculation, not advertised into the IGP for other routers to use), while `forwarding-adjacency` advertises the tunnel as a link in the IGP's link-state database so that any router in the network can compute SPF paths that transit the tunnel, not just the headend — making forwarding-adjacency appropriate when third-party routers need to route over the tunnel as part of their own shortest path.

C. `forwarding-adjacency` is only visible to the headend router, while `autoroute announce` is flooded network-wide via the IGP, which is the reverse of their actual behavior.

D. Neither mechanism affects IGP SPF calculations in any way; both exist solely to control RSVP bandwidth admission control and have no bearing on which router considers the tunnel for shortest-path routing.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Autoroute vs. Forwarding-Adjacency — Visibility Scope**
From: Devon Okafor, Emerald Regional Engineering <devon.okafor@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Autoroute vs. Forwarding-Adjacency — Visibility Scope

James,

Following up on my last note — I want to drill into the actual mechanical difference between `autoroute announce` and `forwarding-adjacency` one more time because I think I conflated them in my head. Specifically: does the rest of the Emerald core "see" the TE tunnel as a usable path in their own SPF calculation under both options, or just one of them? This matters a lot for whether other P routers (not just the tunnel headend) can route transit traffic across the tunnel as part of their own shortest path.

Devon

---

## Section 10: QoS Design

---

**Item 23 of 30**

Refer to the new resource(s) available.

AGN provides a DiffServ-based QoS SLA to Blue Corp, who in turn resells transport to their own downstream customers (requiring a form of hierarchical QoS, or H-QoS, at the PE). Review the email and answer the question.

Blue Corp requires that AGN police/shape traffic per downstream sub-customer (nested under Blue Corp's own aggregate CIR/EIR contract at the PE), while still applying a single top-level DiffServ marking/queuing policy for Blue Corp's aggregate traffic class across the AGN core. Which QoS architecture satisfies this requirement?

A. A single flat policy-map applied once at the PE physical interface level, since flat policies can express both an aggregate shaping rate and per-sub-customer policing simultaneously without any nested class-map structure.

B. A Hierarchical QoS (H-QoS) policy-map structure, with a child (nested) policy-map performing per-sub-customer classification/policing applied within a parent policy-map that shapes/queues Blue Corp's traffic in aggregate against its contracted CIR/EIR, correctly modeling the two-level (sub-customer, then aggregate-customer) requirement.

C. H-QoS is not supported for L3VPN customers; per-sub-customer policing must be performed exclusively on Blue Corp's own CE equipment, with AGN only applying a single aggregate policy at the PE.

D. Two independent, unrelated policy-maps applied to two different physical interfaces, since H-QoS requires physically separate interfaces per hierarchy level and cannot be expressed as nested policy-maps on a single interface.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp H-QoS Requirement — Nested Sub-Customer Policing**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp H-QoS Requirement — Nested Sub-Customer Policing

James,

Blue Corp's contract has an unusual wrinkle: they resell portions of their contracted bandwidth to their own downstream customers, and they want us to police each of those sub-customers individually at our PE, while Blue Corp's traffic as a whole still gets shaped/queued against their own aggregate contracted rate across our core. This smells like a textbook Hierarchical QoS use case to me, but I want your confirmation on the correct policy-map structure (parent/child relationship) before our QoS team builds this out, since I don't want them building two unrelated flat policies that don't actually nest correctly.

Ingrid

---

**Item 24 of 30**

Refer to the new resource(s) available.

AGN needs to map Blue Corp's DSCP markings (set by the customer at their CE) to MPLS EXP bits for QoS treatment across the Emerald/Garnet/Kyanite MPLS core, and must decide between uniform mode and pipe mode for how EXP markings interact with DSCP as traffic traverses the core and exits at the egress PE. Review the email and answer the question.

Which statement correctly distinguishes uniform mode from pipe mode in this DSCP-to-EXP mapping scenario, and which mode should AGN select if Blue Corp requires their original DSCP markings to be fully preserved and unmodified from ingress CE to egress CE, regardless of any EXP remarking that occurs inside the AGN MPLS core (e.g., due to congestion-based EXP remarking at a P router)?

A. Uniform mode should be selected, because uniform mode guarantees the original customer DSCP value is always preserved end-to-end regardless of any remarking that happens to the EXP field inside the core.

B. Pipe mode should be selected, because in pipe mode the EXP field used inside the MPLS core is decoupled from the customer's original DSCP marking (the DSCP value is not overwritten by core EXP remarking events, since the IP header carrying the DSCP value is not re-derived from the top MPLS label's EXP field at egress), preserving the customer's original DSCP end-to-end even if EXP is remarked in the core for congestion handling, whereas uniform mode propagates core remarking decisions down into the customer-visible DSCP value.

C. Both modes behave identically with respect to DSCP preservation; the only difference between uniform and pipe mode is which direction (ingress vs. egress) the mapping is applied.

D. Neither mode has any relationship to DSCP preservation; uniform and pipe mode exclusively govern IP TTL propagation behavior in MPLS networks, not QoS marking behavior.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp DSCP Preservation Requirement — Uniform vs. Pipe Mode**
From: Ingrid Solberg, Atlas Global Networks BGP Architecture Lead <ingrid.solberg@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp DSCP Preservation Requirement — Uniform vs. Pipe Mode

James,

Blue Corp's SLA includes a hard requirement that their original DSCP markings arrive unmodified at the far-end CE, no matter what happens to the EXP field inside our core (and we do remark EXP during congestion events on a couple of oversubscribed Garnet links, which is normal and expected on our end). I need to know definitively whether we should be running uniform mode or pipe mode for this customer's traffic to satisfy that DSCP-preservation requirement, and I want the actual mechanical reasoning, not just "pipe mode is for VPNs" as a rule of thumb — I want to be able to defend this in Blue Corp's next QBR if asked.

Ingrid

---

## Section 11: Multicast VPN (mVPN)

---

**Item 25 of 30**

Refer to the new resource(s) available.

Green Corp has requested IPTV multicast distribution across their multi-AS (Emerald + Garnet) L3VPN footprint. Review the email and answer the question.

Green Corp's multicast sources are known and limited to a small set of IPTV head-end servers, and the customer wants to avoid the overhead of PIM-SM's shared-tree/rendezvous-point (RP) infrastructure and the associated source-tree switchover process. Which combination of mVPN design elements best satisfies this requirement?

A. PIM-SM with Anycast-RP across both regions, using only Default MDTs (no Data MDTs), since Anycast-RP is the only valid design for any inter-region multicast deployment regardless of the customer's source/receiver model.

B. PIM-SSM (Source-Specific Multicast) within Green Corp's VRF, eliminating the need for RP infrastructure and shared-tree-to-source-tree switchover entirely since receivers join (S,G) state directly for the known IPTV sources, combined with an appropriately profiled mVPN transport (e.g., a Default MDT for low-rate control/signaling multicast and Data MDTs for high-bandwidth source-specific IPTV streams to avoid flooding every PE in the Default MDT).

C. mVPN does not support SSM under any circumstances; Green Corp must use PIM-SM with a statically configured RP regardless of their stated preference.

D. MSDP (Multicast Source Discovery Protocol) is required for Green Corp's design regardless of PIM mode chosen, since MSDP is a mandatory component of every mVPN deployment.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Green Corp IPTV Multicast — PIM Mode Selection**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Green Corp IPTV Multicast — PIM Mode Selection

James,

Green Corp wants to light up IPTV distribution across their Emerald+Garnet VPN footprint. Their sources are a known, small set of head-end servers — not an any-source model — and their network architect specifically asked us to avoid "RP complexity and shared-tree switchover delay," which tells me they already have some multicast design background and are steering us toward SSM. Can you confirm whether SSM is viable inside an mVPN context, and if so, how we should profile the MDTs (Default vs. Data) to avoid flooding every PE in the VRF with high-bandwidth IPTV streams they didn't ask for?

Felicia

---

**Item 26 of 30**

Refer to the new resource(s) available.

Review the resource below regarding Blue Corp's separate multicast requirement, which spans all three regions and includes an any-source conferencing application (unlike Green Corp's known-source IPTV model).

Blue Corp's conferencing application uses an any-source multicast model where receivers do not know the source address in advance, ruling out SSM. Within a single AS this is solved with PIM-SM and a local RP, but Blue Corp's receivers in Kyanite need to join groups whose sources are active in Emerald, across the inter-AS mVPN boundary. Which mechanism is responsible for allowing RP/source information for Blue Corp's groups to be learned across this inter-AS (inter-region) boundary, and how does it relate to the PIM-SM RP already configured within each region?

A. MSDP (Multicast Source Discovery Protocol), which allows RPs in different PIM-SM domains (here, different regions acting as independent mVPN multicast domains for Blue Corp) to exchange Source-Active (SA) messages so that a receiver's local RP can learn about active sources in another region and trigger the appropriate (S,G) join across the inter-AS mVPN transport, working alongside (not replacing) each region's existing PIM-SM RP.

B. SSM is required instead of PIM-SM for all inter-AS multicast; MSDP is deprecated and cannot be used in any current mVPN deployment.

C. BGP alone, with no PIM or MSDP involvement whatsoever, is fully sufficient to propagate source/RP reachability information across inter-AS multicast domains.

D. Each region must share a single common RP across all three ASes (eliminating per-region RPs entirely) because MSDP cannot function unless exactly one RP exists network-wide.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Blue Corp Inter-AS Conferencing Multicast — RP/Source Discovery Across Regions**
From: Priya Natarajan, Kyanite Regional Engineering Lead <priya.natarajan@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Blue Corp Inter-AS Conferencing Multicast — RP/Source Discovery Across Regions

James,

Different multicast use case than the Green Corp IPTV one — Blue Corp has a conferencing app where receivers genuinely don't know the source address ahead of time, so SSM is off the table and we're staying with PIM-SM and per-region RPs. My question is specifically about the inter-AS piece: Kyanite receivers need to discover and join sources that are active in Emerald. Each region already has its own local RP for PIM-SM. What's the mechanism that lets Kyanite's RP learn "hey, there's an active Blue Corp source over in Emerald" so the right join can happen across the mVPN inter-AS transport? I have a vague memory of MSDP from my CCNP days but want to confirm it's still the right tool here and how it interacts with the RPs we already have.

Priya

---

## Section 12: Infrastructure Security

---

**Item 27 of 30**

Refer to the new resource(s) available.

AGN's security team wants to protect route processor CPU on all P and PE routers against control-plane floods following a BGP-based DDoS incident at a peer SP. Review the email and answer the question.

Which security mechanism should be deployed on every edge-facing router (and ideally core-wide) to rate-limit and classify control-plane-destined traffic (BGP, IS-IS/OSPF, SSH, ICMP, etc.) before it can overwhelm the route processor, while still allowing legitimate routing protocol traffic to be prioritized over less-critical management traffic during an attack?

A. Unicast Reverse Path Forwarding (uRPF) in strict mode, since uRPF is specifically designed to rate-limit control-plane-destined traffic by CPU priority class.

B. Control Plane Policing (CoPP), using a classification policy that separates traffic destined for the route processor into priority classes (e.g., critical routing protocols, management, undesirable/known-bad) and applies distinct policing rates to each class, protecting the RP from floods while preserving routing protocol stability.

C. Generalized TTL Security Mechanism (GTSM) alone, since GTSM provides full control-plane flood protection for all protocols, not just eBGP sessions.

D. BGP Remotely-Triggered Black Hole (RTBH) filtering, since RTBH is designed to protect the route processor directly rather than to black-hole traffic toward a specific victim destination prefix.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Post-Incident Hardening — Route Processor Protection**
From: Marcus Webb, Atlas Global Networks Security & Compliance <marcus.webb@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Post-Incident Hardening — Route Processor Protection

James,

One of our peer SPs got hit hard last month by a control-plane flood aimed at overwhelming BGP route processor CPU on their edge routers — they lost BGP sessions network-wide because the RP couldn't keep up with processing the flood alongside legitimate routing traffic. I want to make sure AGN isn't exposed the same way. What's the correct mechanism to classify and rate-limit traffic destined for the route processor itself (not transit traffic), so that even under a flood, our BGP/IS-IS/OSPF control traffic stays prioritized over lower-priority management or "who knows what" traffic hitting the RP?

Marcus

---

**Item 28 of 30**

Refer to the new resource(s) available.

Following up on the same incident, AGN also wants to add a reactive mitigation capability so that if a specific customer-facing destination becomes the target of a volumetric attack, AGN can rapidly stop forwarding attack traffic toward that destination across the whole network from a single point of control. Separately, AGN wants to validate that eBGP sessions with customers cannot be easily spoofed by an attacker who is not directly on the local segment. Review the email and answer both parts.

**Question 28a (Multiple Choice):** Which mechanism allows AGN's security team to trigger, from a single control point, network-wide discard of traffic destined to a specific attacked customer prefix, by having edge/PE routers automatically match that prefix and route it to a discard (Null0-style) next hop?

A. CoPP, since CoPP is also capable of discarding transit traffic destined to any arbitrary customer prefix network-wide, not just route-processor-destined traffic.

B. BGP Remotely-Triggered Black Hole (RTBH) filtering, where a specifically tagged BGP update (typically via a community) advertised from a trigger router causes participating edge routers to install a discard route for the victim prefix, achieving network-wide, rapid, single-point-of-control mitigation.

C. uRPF loose mode, since loose-mode uRPF is specifically designed to discard traffic toward a victim prefix rather than validate source address plausibility.

D. GTSM, since GTSM operates on destination prefixes rather than TTL values and is the correct tool for victim-prefix-based blackholing.

**Question 28b (Multiple Choice):** Which mechanism should be applied to the eBGP sessions with customer CE routers to make it significantly harder for an off-path attacker to spoof TCP segments into the session, by validating that the received TTL in each BGP packet is consistent with an expected number of IP hops from a legitimate (typically directly connected or near) peer?

A. GTSM (Generalized TTL Security Mechanism), which checks that the IP TTL of received BGP packets falls within an expected range implying the peer is at or near the expected number of hops away, discarding packets whose TTL indicates they could only have originated many hops away (as would be expected from most off-path spoofing attempts).

B. uRPF strict mode, which validates TTL consistency for eBGP sessions as its primary function.

C. BGP RTBH, since RTBH's tagging mechanism also implicitly validates TTL for all tagged prefixes.

D. CoPP, since CoPP's classification ACLs are the only mechanism capable of matching on IP TTL values for BGP sessions specifically.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Post-Incident Hardening — RTBH and eBGP Session Protection**
From: Marcus Webb, Atlas Global Networks Security & Compliance <marcus.webb@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Post-Incident Hardening — RTBH and eBGP Session Protection

James,

Two more pieces to close out the post-incident hardening review:

1. If one of our customers becomes the target of a volumetric attack aimed at a specific destination (not at our RP, at the customer's actual prefix), I want a way to blackhole that specific prefix network-wide from one place, fast, without touching every edge router by hand. What's the right mechanism and roughly how does the trigger propagate to all the edge routers?

2. Separately — and this is more theoretical/preventative — I want to understand what we have in place (or should have in place) to make our customer-facing eBGP sessions harder to spoof by an attacker who isn't directly on the customer's local segment. I recall something about TTL-based validation for BGP sessions specifically but want the correct name and mechanism confirmed.

Marcus

---

## Section 13: 6PE / 6VPE

---

**Item 29 of 30**

Refer to the new resource(s) available.

Yellow Corp has requested IPv6 connectivity across their Garnet-only L3VPN footprint, and separately AGN's own core team wants to offer native IPv6 Internet transit to a different (non-VPN) customer across the existing IPv4-only MPLS core without redesigning the core itself. Review the email and answer the question.

AGN's MPLS core (across all three regions) runs IPv4-only IGP, LDP, and BGP sessions, with no plans to run a native IPv6 IGP or dual-stack LDP in the near term. Which technology allows AGN to offer native (non-VPN) IPv6 transit across this IPv4-only MPLS core, and what is the key mechanism for IPv6 next-hop handling at the ingress PE?

A. 6PE (IPv6 Provider Edge), where PE routers exchange IPv6 prefixes via MP-BGP using an IPv4-mapped IPv6 address as the BGP next-hop (encoding the IPv4 address of the egress PE within the next-hop so that the ingress PE can resolve a label stack toward the egress PE's IPv4 loopback via the existing IPv4 LDP/IGP infrastructure), while actual IPv6 packets are encapsulated in MPLS labels and forwarded across the IPv4-only MPLS core without requiring the P routers to understand IPv6 at all.

B. 6VPE, which is required instead of 6PE for any non-VPN (global IPv6 table) transit service, since 6PE only functions within a VRF context.

C. Dual-stack IGP and LDP must be deployed core-wide before any IPv6 service (6PE, 6VPE, or otherwise) can be offered, since MPLS cannot carry IPv6 payloads without native IPv6 IGP/LDP support on P routers.

D. 6PE requires every P router to run native IPv6 BGP sessions in addition to IPv4, since P routers must independently validate IPv6 next-hop reachability for every 6PE prefix.

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Yellow Corp IPv6 and Native IPv6 Transit — 6PE/6VPE Feasibility**
From: Felicia Grant, Garnet Regional Engineering Lead <felicia.grant@atlasglobalnetworks.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Yellow Corp IPv6 and Native IPv6 Transit — 6PE/6VPE Feasibility

James,

Two IPv6 requests landed this week. Yellow Corp wants IPv6 inside their existing L3VPN (that's a 6VPE conversation, I think, and we'll cover it separately). But the more immediate question is from our own sales team: they want to sell native (non-VPN, global table) IPv6 Internet transit to a prospective customer, and our entire MPLS core — IGP, LDP, BGP — is IPv4-only today with zero appetite from the core team to deploy dual-stack IGP/LDP network-wide just for this one deal. Is there a way to offer native IPv6 transit across an IPv4-only MPLS core without touching the P routers at all, and if so, how does the next-hop actually get resolved for an IPv6 prefix when the underlying LSP is built entirely on IPv4 infrastructure?

Felicia

---

## Section 14: Capstone — Cross-Domain Design Review

---

**Item 30 of 30**

Refer to the new resource(s) available.

As a capstone to the Phase 1 design review, James Carvalho has asked for a single consolidated design-sequencing exercise covering the overall merger integration program. Review the email below and complete the ordered-steps exercise.

**Ordered Steps (drag-and-drop equivalent):** AGN's integration program must sequence the following major design workstreams. Place them in the correct dependency order — i.e., the order in which they must be substantially completed or at least architecturally finalized before the next workstream can be safely executed, given everything discussed in Items 1-29.

Steps (unordered):

- Step V: Finalize and deploy the IS-IS/OSPF authentication and metric-style standardization across all three regions (Items 2-3, Item 9's LDP-IGP sync depends on a stable, converged IGP).
- Step W: Select and implement the Inter-AS VPN option(s) (A/B/C) for each customer spanning multiple ASes, including BGP-LU deployment where required for Option C (Items 14-16), which depends on inter-AS ASBR/PE BGP sessions already being designed.
- Step X: Finalize per-region IGP hierarchy and area/level design (Items 1, 3, 4, 5), establishing stable intra-region routing before any inter-AS or RR design is finalized.
- Step Y: Deploy MPLS-TE/FRR and QoS/H-QoS policies for latency- and SLA-sensitive customers such as Blue Corp (Items 21-24), which depends on both the core LSP (via LDP or RSVP-TE) and the relevant VPN service (L3VPN/VPWS/VPLS) already being operational.
- Step Z: Establish RR placement, cluster-ID design, and core iBGP/LDP infrastructure within each region (Items 6-10), which depends on each region's IGP being finalized but precedes any inter-AS BGP work.

Provide the correct order (e.g., "X, Z, W, Y, V").

**Answer:** *(left blank — see Solutions document)*

**Resources**

**Email: Merger Integration Program — Design Sequencing Review**
From: James Carvalho (self-note, forwarded from the Atlas Global Networks Program Steering Committee) <james@ccie-sp-lab.com>
To: "James Carvalho" <james@ccie-sp-lab.com>
Subject: Merger Integration Program — Design Sequencing Review

Team,

Before we present the overall Phase 1 design review to the AGN steering committee, I want a single, defensible sequencing of the major workstreams we've covered across this engagement — IGP hierarchy and hardening, RR/core BGP/LDP infrastructure, inter-AS VPN option selection, and the TE/QoS overlay for premium customers. The steering committee is going to ask "what depends on what, and why can't we just do all of this in parallel," and I want a clean, defensible answer grounded in the actual technical dependencies we've walked through item by item in this review — not just a gut-feel ordering.

Please provide the correct dependency-ordered sequence for the five major workstreams, with the reasoning for why each step cannot be safely finalized before its predecessor.

James

---

## End of Phase 1 Design Document

This concludes the Atlas Global Networks CCIE Service Provider Design Phase 1 exam set (Items 1–30). Record your answers separately. Do not proceed to the Solutions document until all 30 items have been attempted.

**Topic coverage recap:**

| Section | Topic | Items |
|---|---|---|
| 1 | IS-IS Design | 1–3 |
| 2 | OSPF Design | 4–5 |
| 3 | BGP Design | 6–8 |
| 4 | MPLS LDP | 9–10 |
| 5 | L3VPN Design | 11–13 |
| 6 | Inter-AS VPN | 14–16 |
| 7 | L2VPN/VPWS | 17–18 |
| 8 | VPLS | 19–20 |
| 9 | MPLS-TE | 21–22 |
| 10 | QoS | 23–24 |
| 11 | Multicast VPN | 25–26 |
| 12 | Security | 27–28 |
| 13 | 6PE/6VPE | 29 |
| 14 | Capstone (cross-domain) | 30 |
