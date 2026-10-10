# CCIE Service Provider — Design Phase 1: Solutions

## Atlas Global Networks — Multi-AS Service Provider Design Scenario

**Companion answer key to `CCIE_SP_Design_Phase_1.md`.** Each item below states the correct answer(s), explains why the correct answer is right and the distractors are wrong, and references the relevant RFC or Cisco design concept.

---

## Section 1: IS-IS Design (Emerald & Kyanite Regions)

---

## Item 1
**Correct Answer:** B

**Explanation:** Introducing a multi-level IS-IS hierarchy with each metro cluster as its own Level-1 area, stitched together by a Level-1/Level-2 backbone, is the design that directly answers Priya's stated fear: it contains SPF churn. A Level-1 LSDB is bounded to its own area, so a flapping access ring in North Cluster forces SPF recomputation only within North Cluster, while Central and Harbor converge independently — the exact fault-isolation goal. A (flat L2-only) is wrong because IS-IS SPF cost scales with LSDB size and link-state flooding is domain-wide, so churn would not be contained. C (L1-only with no L2 backbone) breaks inter-area routing — Level-1 areas require a Level-2 backbone to exchange routes between areas and reach the ASBRs. D (tuning SPF timers) treats a symptom and still floods/recomputes domain-wide, not a hierarchy fix.

**Reference:** ISO 10589 (IS-IS) level hierarchy; Cisco IS-IS multi-area design — L1 areas with an L1/L2 backbone for fault/flooding domain containment.

---

## Item 2
**Correct Answer:** B and E

**Explanation:** B addresses the metric-style migration safely: the supported transition is `metric-style transition` (originate narrow + wide, accept both) first so every router can *interpret* wide TLVs before any router *originates* wide-only, avoiding a window where a router discards reachability it cannot parse — preventing the black hole Marcus called out. E addresses authentication correctly: securing Hello, LSP, and SNP PDUs with a key-chain gives full coverage and supports non-disruptive key rollover (the "no flag day" requirement). A is wrong because interface (Hello) authentication alone does not protect LSP/SNP exchange, so the LSDB is still forgeable. C is wrong because cleartext fails the audit's forgery concern and Level-1 vs. Level-2 authentication is independent of whether the domain is L2-only. D is wrong because narrow and wide metrics are **not** transparently interoperable — an abrupt single-window flip risks exactly the black hole they must avoid.

**Reference:** RFC 5304 / RFC 5310 (IS-IS HMAC-MD5 / generic cryptographic authentication, key-chains); RFC 5305 (wide/extended metrics, TE link attributes); Cisco `metric-style transition` two-phase migration.

---

## Item 3
**Correct Answer:** 1 → b, 2 → c (or b), 3 → b, 4 → b

**Explanation:** Emerald is a Level-2-only backbone; the cleanest post-merger design keeps the backbone as a single Level-2 flooding domain rather than fragmenting it into L1 areas purely to shrink the LSDB (an 8-router core does not need that). So pure interior P routers (1), interior PE routers (3), and the route reflectors (4) are all **Level-2-only (b)** — they live entirely within the Emerald L2 backbone and need the full L2 topology for correct SPF/next-hop resolution. The ASBRs (2) are the one role where **L1/L2 (c)** could apply if Emerald later introduces L1 areas behind them, but in the current flat-L2 backbone they too remain **Level-2-only (b)**; the exam's intent is that the ASBR is the natural L1/L2 boundary candidate. None of these roles should be **Level-1-only (a)** — an L1-only router cannot reach other areas/ASBRs without an L1/L2 border injecting a default, which would strand the P/PE/RR routers from inter-AS reachability.

**Reference:** ISO 10589 level semantics; Cisco IS-IS design guidance — keep a small SP backbone L2-only; L1/L2 routers sit at area borders (ASBR/edge), not on interior P or RR nodes.

---

## Section 2: OSPF Design (Garnet Region)

---

## Item 4
**Correct Answer:** C

**Explanation:** The area must simultaneously (1) originate an external route for the redistributed billing network and (2) still remain stubby. Only an **NSSA** does both: external redistribution inside an NSSA is carried as Type-7 LSAs (ordinary stub areas forbid Type-5/external origination), and the ABR translates Type-7 → Type-5 for the rest of the domain. The NSSA still suppresses the full external table while receiving inter-area/summarized reachability. A (stub) is wrong because stub areas cannot originate externals at all. B (totally stubby) is wrong both because it blocks Type-3 summaries and still cannot originate externals, and "always use totally stubby" is not a valid rule. D is wrong — redistributing externals does not force a standard area; that is precisely what NSSA was invented to allow within a stub-like area.

**Reference:** RFC 3101 (OSPF NSSA Option) — Type-7 LSA origination and ABR Type-7→Type-5 translation; RFC 2328 (OSPF v2) stub-area restrictions on Type-5.

---

## Item 5
**Correct Answer:** 5a → A; 5b → B

**Explanation (5a):** The single-ABR exposure is fixed by adding a **second, physically diverse ABR** connecting Harbor Ring into Area 0 — true redundancy for the area's backbone attachment. B is wrong because a virtual link is a backbone-continuity patch (it extends Area 0 through a transit area), not a sanctioned primary-redundancy mechanism — Felicia's instinct is correct. C only speeds reconvergence after the ABR dies; it does not remove the single point of failure. D is a non-sequitur — NSSA vs. standard has no bearing on adding a redundant ABR, and virtual links across an NSSA are not even permitted.
**Explanation (5b):** On modern routers over a stable fiber core, use **SPF/LSA exponential-backoff throttling** with low initial/min and sane max (sub-second initial, millisecond min-hold) — fast convergence while backoff still damps LSA storms. A leaves stock 5000 ms delay, needlessly slow. C (zero throttle) removes storm protection and invites CPU churn on instability. D (max max-wait) deliberately trades away the convergence they are trying to improve.

**Reference:** RFC 2328 ABR/virtual-link semantics (virtual links defined to repair backbone partition, not for routine redundancy); Cisco `timers throttle spf` / `timers throttle lsa` exponential backoff tuning.

---

## Section 3: BGP Design (Cross-AS)

---

## Item 6
**Correct Answer:** B

**Explanation:** Cluster-ID is the identifier a route reflector prepends to the CLUSTER_LIST attribute; a reflector discards any route whose CLUSTER_LIST already contains its own cluster-ID (loop prevention). With a **shared** cluster-ID, RR1-EMR and RR2-EMR each drop routes reflected by the other (they see the common cluster-ID), suppressing redundant reflected copies. **Splitting** them into distinct cluster-IDs removes that mutual suppression: each RR now accepts and re-reflects the other's routes, producing additional/duplicate paths that must be verified not to create loops or churn — exactly what B describes. A is wrong (cluster-ID is functional, not cosmetic). C is wrong — clients do not reject updates based on a mismatched cluster-ID; CLUSTER_LIST loop detection happens at reflectors. D is wrong — CLUSTER_LIST/cluster-ID is strictly an iBGP route-reflection construct, so it very much affects these iBGP sessions.

**Reference:** RFC 4456 (BGP Route Reflection) — CLUSTER_LIST and CLUSTER_ID loop prevention; shared cluster-ID within a redundant RR cluster.

---

## Item 7
**Correct Answer:** 7a → B; 7b → C

**Explanation (7a):** A single RR is a single point of failure for the entire iBGP control plane (all AFI/SAFI, including VPNv4/VPNv6). As Kyanite grows to 14 routers, add a **second RR to form a redundant pair** (Emerald's model). A is wrong — RR CPU/memory sizing does not address loss of the RR itself. C is wrong — full mesh scales *worse* (n(n-1)/2 sessions), not better. D is wrong — route reflection does not depend on confederations or on IS-IS level.
**Explanation (7b):** Inter-AS Option C needs an **end-to-end LSP** to each PE loopback across all ASes. BGP-LU (RFC 3107) is exchanged via eBGP between adjacent ASBRs (labeled /32 loopbacks) and re-advertised (multihop eBGP or redistributed into iBGP) so every AS knows a label path to the remote PE loopbacks — the defining Option C behavior. A is wrong — LU is a separate labeled-unicast advertisement, not carried on the VPNv4 session. B is wrong — LU is specifically exchanged *across* ASBRs, not bypassing them. D is wrong — Option C depends on BGP-LU, not pure IGP redistribution of PE loopbacks across ASes.

**Reference:** RFC 4456 (RR redundancy); RFC 3107 / RFC 8277 (BGP labeled unicast); Cisco Inter-AS MPLS/VPN Option C.

---

## Item 8
**Correct Answer:** X, Y, W, Z

**Explanation:** Verify first, configure second, enable last. **X** — confirm recursive loopback reachability to the remote PE (the #1 cause of "stuck in Active/Idle" is that the recursive next-hop lookup can never resolve). **Y** — set `update-source Loopback0` on both ends so the TCP session uses the stable loopback the reachability was verified against. **W** — set `ebgp-multihop <ttl>` large enough for the real hop count (two transit P routers here). **Z** — enable the neighbor and verify session + prefix exchange + next-hop behavior. Enabling before verifying reachability (the junior-engineer anti-pattern Ingrid cites) wastes time debugging a session that structurally cannot form.

**Reference:** Cisco BGP `ebgp-multihop`, `update-source`, loopback-sourced multihop eBGP turn-up; recursive next-hop resolution order of operations.

---

## Section 4: MPLS LDP Design

---

## Item 9
**Correct Answer:** B

**Explanation:** **LDP-IGP Synchronization** is purpose-built for this race: it keeps the link's IGP metric at max (effectively unusable for SPF) until the LDP session on that link is up and label bindings are exchanged, so the IGP never installs a next hop that has no label yet — eliminating the transient LSP blackhole during reconvergence. A (Session Protection) keeps a session alive across a brief outage via targeted Hellos but does not gate IGP use of a *newly* converged path. C (Targeted LDP) is for non-adjacent sessions (AToM/VPLS), not link-local IGP synchronization. D (`no mpls ldp advertise-labels`) would suppress label advertisement and make the blackhole worse.

**Reference:** RFC 5443 (LDP IGP Synchronization); Cisco `mpls ldp sync`.

---

## Item 10
**Correct Answer:** B

**Explanation:** Non-IGP-adjacent PEs (several hops apart) need a **targeted LDP session** (extended/directed Hellos) between their loopbacks to exchange pseudowire (VC) labels — standard link Hellos only form sessions between directly connected neighbors. Pairing this with **label filtering/advertisement scoping** on the transit P routers (advertise only the PE loopback /32s relevant to signaling) keeps unnecessary label state off the P routers, which is exactly Devon's concern. A is wrong — the PEs are not directly connected, so link Hellos cannot form the session, and link LDP does not "auto-scope" to only needed prefixes. C is wrong — targeted LDP is the standard AToM/VPLS signaling method, not deprecated. D is wrong — link Hellos on core interfaces never build a session between non-adjacent PEs.

**Reference:** RFC 5036 (LDP, extended/targeted discovery); Cisco AToM/VPLS targeted LDP; LDP label advertisement control (`mpls ldp label` / `advertise-labels` ACLs).

---

## Section 5: L3VPN Design

---

## Item 11
**Correct Answer:** B

**Explanation:** The **Type 1 RD** (IPv4-address:number) using each PE's unique loopback guarantees per-VRF RD uniqueness with zero inter-region coordination — no two PEs share a loopback, so RDs never collide even if regional teams provision independently. For Blue Corp's hub-less any-to-any mesh, a **single common RT** imported and exported by every participating PE yields full-mesh reachability; distinct/asymmetric RTs are reserved for hub-and-spoke or extranet policies. A reuses an identical RD everywhere (defeats uniqueness) and relies on manual coordination. C deliberately reuses RDs per customer — RD uniqueness per PE is a real functional best practice (it keeps otherwise-identical prefixes distinct in the VPNv4 table and enables correct BGP bestpath/BGP add-path behavior), not merely cosmetic. D is wrong — RTs drive VRF import/export and cannot be replaced by RD filtering; RDs are not a policy mechanism.

**Reference:** RFC 4364 (BGP/MPLS IP VPNs) — RD Type 0/1/2 formats, RT extended communities, import/export policy.

---

## Item 12
**Correct Answer:** 12a → B; 12b → A and B

**Explanation (12a):** Both Blue Corp CE sites use the same private ASN (65501). When the Kyanite-originated route reaches the Garnet PE-CE eBGP session, the CE's own ASN already appears in the AS_PATH, so **eBGP AS-path loop detection rejects it** — the classic duplicate-CE-ASN problem.
**Explanation (12b):** Two correct, mechanically distinct fixes. **A `as-override`** (applied on the PE toward the CE) rewrites occurrences of the CE's ASN in the AS_PATH with the provider ASN *before advertising to the CE*, so the route no longer looks like a loop to the receiving CE. **B `allowas-in`** relaxes loop detection *on inbound* so a router accepts routes containing its own ASN up to *n* times. They act at different points (egress rewrite vs. ingress acceptance) and are not interchangeable, which is Devon's exact question. C (`remove-private-as`) is for cleaning private ASNs toward Internet/eBGP peers and does not reliably solve PE-CE VPN duplicate-ASN loops. D (drop eBGP for static) is unnecessary and not the only workaround.

**Reference:** RFC 4271 (BGP AS_PATH loop detection); RFC 4364 PE-CE eBGP; Cisco `as-override` and `allowas-in`.

---

## Item 13
**Correct Answer:** A

**Explanation:** When a PE redistributes a VPNv4/BGP route into a PE-CE OSPF process, it sets the **DN-bit (Down bit)** in the regenerated LSA. Any PE reading an LSA with the DN-bit set must not redistribute it back into the MPLS/VPN backbone, breaking the Site-A → backbone → Site-B → backbone loop and preventing incorrect intra-area preference across multi-site OSPF VPNs. B (MC-bit) is unrelated (multicast capability). C is wrong — the DN-bit is a standard, automatically applied mechanism in IOS/IOS-XR PE-CE OSPF; a manual route-map is not "the only supported" method. D is wrong — ORIGINATOR_ID is a BGP RR attribute, not an OSPF LSA loop-prevention bit.

**Reference:** RFC 4576 (Using the OSPF DN-bit to Prevent Looping in BGP/MPLS IP VPNs); RFC 4577 (OSPF as PE-CE protocol, superbackbone, domain-id).

---

## Section 6: Inter-AS VPN Design

---

## Item 14
**Correct Answer:** 1 → b, 2 → a, 3 → c

**Explanation:** **Option A (1 → b):** back-to-back VRF/sub-interface per customer at the ASBRs — simplest and most isolated, forwards natively at L3 at the boundary (no end-to-end LSP), but scales poorly (one VRF + (sub)interface per shared customer on both ASBRs). **Option B (2 → a):** ASBRs exchange labeled VPNv4 routes via (multihop) MP-eBGP and hold VPNv4 state for shared customers only — a middle ground between per-VRF scaling and full PE-to-PE knowledge. **Option C (3 → c):** multihop MP-eBGP between PEs plus BGP-LU for a true end-to-end LSP — most scalable/lowest-latency any-to-any, but highest inter-AS trust/coordination and operational complexity.

**Reference:** RFC 4364 §10 Inter-AS Options A / B / C; Cisco Inter-AS MPLS VPN design trade-offs.

---

## Item 15
**Correct Answer:** C

**Explanation:** Only **Option C** delivers a single, unbroken ingress-PE-to-egress-PE LSP with no L3 re-encapsulation at any boundary, and it requires **BGP-LU (RFC 3107) running on every ASBR pair** so labeled /32 reachability to all relevant PE loopbacks is known end-to-end across all three ASes. A is wrong — Option A always forwards natively at L3 at the boundary, which is the opposite of "no L3 lookup." B is wrong — Option B regenerates the label stack hop-by-hop at each ASBR (next-hop-self), so it is *not* a single continuous LSP. D is wrong — mixing Option A at one pair breaks end-to-end LSP continuity at that boundary; options cannot be freely mixed when a continuous LSP is required.

**Reference:** RFC 4364 §10 (Option C); RFC 3107/8277 (BGP-LU for inter-AS end-to-end LSP).

---

## Item 16
**Correct Answer:** B

**Explanation:** In Option B the ASBRs exchange labeled VPNv4 routes, but RTs are opaque extended communities passed only according to **configured policy** — so you must explicitly configure RT retention (e.g., `retain route-target`) and/or an RT-rewrite route-policy to translate Emerald's `65100:500` to Garnet's `65200:500`. Without it, the receiving PE's VRF import fails and routes are silently dropped — Ingrid's precise worry. A is wrong — RT handling is not automatic/mandatory and translation *is* configurable. C is wrong — RTs (not RDs) drive VRF import and matter in Option B. D is wrong — policy manipulation of exchanged VPNv4 routes is a core Option B capability.

**Reference:** RFC 4364 (RT extended communities / VRF import); Cisco Inter-AS Option B `retain route-target` and RT-rewrite route-policy.

---

## Section 7: L2VPN / VPWS Design (Red Corp)

---

## Item 17
**Correct Answer:** B

**Explanation:** The control word is an **optional 4-byte field** carrying flags/sequence number and short-frame padding indication. Enable it when sequencing/flag handling helps — notably to prevent transit LSRs from misparsing a small Ethernet frame as an IP packet for ECMP hashing (causing reordering) and to flag padded short frames. Capability is **negotiated in LDP pseudowire signaling**, so both PEs must agree. A is wrong — the control word is optional for Ethernet AToM, not mandatory/always-on. C is wrong — it applies to VPWS/AToM point-to-point PWs, not only VPLS. D is wrong — it never replaces the VC label; the VC label still identifies the pseudowire.

**Reference:** RFC 4448 (Ethernet over MPLS) and RFC 4385 (PW control word / sequencing); RFC 4447/8077 (LDP PW signaling, control-word negotiation).

---

## Item 18
**Correct Answer:** B

**Explanation:** AToM does **not** fragment/reassemble customer frames; the two attachment-circuit MTUs (and the signaled PW MTU) must match or the pseudowire fails to come up / silently drops oversized frames. Remediation is to align interface + pseudowire-class MTU on both PEs for 9000-byte frames **and** raise the core-facing MPLS MTU so the encapsulated jumbo frame plus label stack transits the core without fragmentation. A is wrong — no transparent fragmentation in AToM. C is wrong — MTU consistency matters for VPWS too (MTU is signaled and checked). D is wrong — the control word does not normalize MTU.

**Reference:** RFC 4447/8077 (PW interface-parameter / MTU signaling and match requirement); Cisco AToM MTU and core MPLS MTU design.

---

## Section 8: VPLS Design (Red Corp Multipoint)

---

## Item 19
**Correct Answer:** B

**Explanation:** Full-mesh VPLS requires **N(N-1)/2** pseudowires (28 at 8 sites), which becomes unwieldy — the case for migrating to **H-VPLS** (N-PE hub core full-mesh, U-PE spokes) to shrink the mesh. Critically, **split-horizon still applies in both models**: a PE must never forward a frame received on one *core* pseudowire out another core pseudowire (that is what prevents L2 loops in a full mesh without STP). A is wrong — full mesh does not scale linearly and is not always preferable. C is wrong — full mesh absolutely requires split-horizon; the mesh does not "inherently" prevent loops. D is wrong — H-VPLS relies on split-horizon, not STP across the PW mesh.

**Reference:** RFC 4762 (VPLS/LDP signaling) — split-horizon forwarding rule; H-VPLS hub (N-PE) / spoke (U-PE) hierarchy.

---

## Item 20
**Correct Answer:** B

**Explanation:** The two symptoms have two mechanisms. **MAC flapping** — the same source MAC relearned on two attachment circuits in quick succession points to a **dual-attached host or a Layer-2 loop at the customer edge**, which Red Corp's team should investigate. **Flooding** — unknown-unicast flooding is *normal* VPLS behavior for unlearned destinations, but *periodic/persistent* flooding suggests MAC entries aging out faster than traffic intervals, so review the **MAC aging timer** vs. Red Corp's real traffic pattern (plus the CE-loop check). A is wrong — VPLS does learn MACs (it is a learning bridge); flapping/flooding are not permanent by design. C is wrong — MTU/RD do not cause MAC flapping or flooding in this way. D is wrong — VPLS does not disable learning past 3 PEs.

**Reference:** RFC 4762 (VPLS MAC learning, unknown-unicast flooding, MAC aging); split-horizon and customer-edge loop considerations.

---

## Section 9: MPLS Traffic Engineering

---

## Item 21
**Correct Answer:** 21a → B; 21b → B

**Explanation (21a):** To protect a single critical **link** against the link's own failure, use **link protection (NHOP) FRR** — a backup tunnel that reroutes around the protected link to the same next-hop node. A is wrong — "node protection everywhere because it's strictly better" is a common misconception; NNHOP protects against *node* failure and is more complex, not the right baseline for a pure link concern. C is wrong — FRR does not require parallel physical paths between every node pair; the backup just needs any diverse path to the merge point. D is wrong — BFD gives fast detection but no pre-signaled backup LSP, so it alone cannot deliver FRR's sub-50ms *protection*.
**Explanation (21b):** Use realistic explicit bandwidth reservations sized to measured peak and reviewed periodically (or auto-bandwidth), with **`autoroute announce`** so only the **headend** treats the tunnel as a link in its own SPF — IGP shortest-path traffic at the headend prefers the tunnel without flooding tunnel knowledge network-wide. A (permanent zero bandwidth + forcing all SPF through the tunnel) breaks admission control and misuses autoroute. C wrongly claims `forwarding-adjacency` is required and identical to autoroute (it is not — see Item 22). D is wrong — FRR tunnels can and do carry bandwidth reservations.

**Reference:** RFC 4090 (FRR: NHOP link protection vs. NNHOP node protection); Cisco MPLS-TE `autoroute announce`; RSVP-TE admission control / auto-bandwidth.

---

## Item 22
**Correct Answer:** B

**Explanation:** **`autoroute announce`** makes the TE tunnel usable for shortest-path routing **only at the headend** — the headend treats the tunnel as a directly connected link in its own SPF/RIB, but the tunnel is **not** advertised into the IGP LSDB, so other routers never see it. **`forwarding-adjacency`** advertises the tunnel as a **link in the IGP link-state database**, so any router in the area can compute SPF paths that transit the tunnel — the right choice when third-party (non-headend) routers must route over the tunnel. A is wrong (they are not identical). C reverses the actual behavior. D is wrong — both mechanisms directly affect SPF path selection, not just admission control.

**Reference:** Cisco MPLS-TE design — `autoroute announce` (headend-only SPF) vs. `mpls traffic-eng forwarding-adjacency` (TE link advertised into IGP, network-wide SPF visibility).

---

## Section 10: QoS Design

---

## Item 23
**Correct Answer:** B

**Explanation:** The two-level requirement — per-sub-customer policing nested under Blue Corp's aggregate shaped/queued class — is the textbook **Hierarchical QoS (H-QoS)** case: a **child policy-map** does per-sub-customer classification/policing, attached inside a **parent policy-map** that shapes/queues Blue Corp's aggregate against its CIR/EIR. A is wrong — a single flat policy cannot express a true nested aggregate-then-per-sub-customer hierarchy. C is wrong — H-QoS is supported for L3VPN PE-side policies; per-sub-customer policing need not live on the CE. D is wrong — H-QoS is expressed as nested policy-maps on one interface, not separate physical interfaces per level.

**Reference:** Cisco MQC Hierarchical QoS — parent (aggregate shape/queue) with nested child (per-class/per-sub-customer police) policy-maps.

---

## Item 24
**Correct Answer:** B

**Explanation:** In **pipe mode**, the MPLS EXP used inside the core is decoupled from the customer's IP DSCP: core EXP remarking (e.g., congestion on Garnet links) is not copied back into the IP DSCP at egress, so the customer's **original DSCP is preserved end-to-end** — satisfying Blue Corp's hard requirement. **Uniform mode** treats the whole stack as one QoS domain and propagates core remarking down into the customer-visible DSCP at egress, which would violate the SLA. A inverts this (uniform does *not* guarantee preservation). C is wrong — the modes differ precisely in DSCP preservation, not ingress/egress direction. D is wrong — uniform/pipe govern QoS marking propagation (and separately TTL via the related short-pipe nuance), but here the QoS-marking behavior is the point, so "no relationship to DSCP" is false.

**Reference:** RFC 3270 (MPLS DiffServ) — Uniform, Pipe, and Short-Pipe tunneling models and their DSCP/EXP propagation behavior.

---

## Section 11: Multicast VPN (mVPN)

---

## Item 25
**Correct Answer:** B

**Explanation:** Green Corp's sources are a known, small set of IPTV head-ends and the customer wants to avoid RP/shared-tree complexity — the exact use case for **PIM-SSM** inside the VRF: receivers join (S,G) directly, so there is no RP and no shared-to-source-tree switchover. Pair it with sensible MDT profiling — a **Default MDT** for low-rate control/signaling and **Data MDTs** for high-bandwidth source-specific streams so only interested PEs receive the IPTV flows rather than flooding every PE. A forces PIM-SM/Anycast-RP and Default-MDT-only, the opposite of what the customer asked and prone to flooding. C is wrong — mVPN fully supports SSM. D is wrong — MSDP is not mandatory for every mVPN (and is irrelevant to SSM).

**Reference:** RFC 4607 (SSM); RFC 6513/6514 (mVPN, Default/Data MDT, PIM/BGP mVPN); SSM eliminates RP and SPT switchover.

---

## Item 26
**Correct Answer:** A

**Explanation:** Blue Corp's any-source conferencing rules out SSM, so each region keeps PIM-SM with its own RP. **MSDP** is the mechanism that lets RPs in different domains exchange **Source-Active (SA)** messages, so Kyanite's RP learns about an active Blue Corp source in Emerald and can trigger the appropriate (S,G) join across the inter-AS mVPN transport — working **alongside** each region's existing RP, not replacing it. B is wrong — SSM is not required and MSDP is not deprecated for this case. C is wrong — BGP alone does not perform PIM-SM inter-domain source discovery (that is MSDP's job). D is wrong — MSDP exists precisely so you do *not* need one network-wide RP; per-region RPs peer via MSDP.

**Reference:** RFC 3618 (MSDP) — inter-domain SA exchange between PIM-SM RPs; mVPN inter-AS source discovery.

---

## Section 12: Infrastructure Security

---

## Item 27
**Correct Answer:** B

**Explanation:** **Control Plane Policing (CoPP)** classifies RP-destined traffic into priority classes (critical routing protocols, management, undesirable/known-bad) and applies distinct policing rates, so under a flood the route processor keeps servicing BGP/IS-IS/OSPF while lower-priority traffic is throttled — directly answering Marcus's requirement. A (uRPF) validates *source* legitimacy of transit traffic, not RP-destined rate-limiting. C (GTSM) only hardens eBGP against spoofing via TTL; it is not a general multi-protocol RP flood-protection mechanism. D (RTBH) black-holes traffic toward a victim *destination prefix*; it does not protect the RP itself.

**Reference:** Cisco Control Plane Policing (CoPP) / Local Packet Transport Services; RP protection via classified policing.

---

## Item 28
**Correct Answer:** 28a → B; 28b → A

**Explanation (28a):** **BGP RTBH** gives single-point, network-wide, fast mitigation: a trigger router advertises the victim /32 (or prefix) tagged with a well-known community; participating edge routers match that community and set the next hop to a discard (Null0) route, dropping attack traffic toward the victim everywhere at once. A is wrong — CoPP protects the RP, not arbitrary transit-to-customer-prefix discard. C is wrong — loose-mode uRPF validates source plausibility, it does not blackhole a destination prefix. D is wrong — GTSM is about TTL, not destination-prefix blackholing.
**Explanation (28b):** **GTSM** validates that the received IP TTL on BGP packets is within the expected range for a near/directly-connected peer (e.g., TTL ≥ 254 for a one-hop peer), so off-path spoofed segments — which arrive with lower TTL — are discarded, hardening customer eBGP sessions against spoofing. B (uRPF strict) does source-RPF validation, not TTL checking. C (RTBH) does not validate TTL. D (CoPP) is RP rate-limiting, not per-session TTL validation.

**Reference:** RFC 5635 (RTBH filtering); RFC 5082 (GTSM / BGP TTL Security Hack).

---

## Section 13: 6PE / 6VPE

---

## Item 29
**Correct Answer:** A

**Explanation:** **6PE** delivers native (global-table, non-VPN) IPv6 transit across an IPv4-only MPLS core. PEs exchange IPv6 prefixes via MP-BGP advertising an **IPv4-mapped IPv6 next-hop** (`::FFFF:<egress-PE-IPv4-loopback>`); the ingress PE resolves a label stack toward that IPv4 loopback using the existing IPv4 IGP/LDP, and IPv6 packets ride MPLS across the core — so the **P routers never need to understand IPv6**. B is wrong — 6VPE is for IPv6 *within a VRF*; native global IPv6 transit uses 6PE. C is wrong — the whole point of 6PE is to avoid dual-stacking the P routers/core IGP/LDP. D is wrong — P routers run no IPv6 BGP and do not validate IPv6 next-hops; that is the ingress/egress PE's job.

**Reference:** RFC 4798 (6PE — Connecting IPv6 Islands over IPv4 MPLS using IPv4-mapped IPv6 next-hop); contrast RFC 4659 (6VPE / IPv6 VPN).

---

## Section 14: Capstone — Cross-Domain Design Review

---

## Item 30
**Correct Answer:** X, V, Z, W, Y

**Explanation:** The dependency chain follows "stable foundation first, premium overlay last":
- **X — per-region IGP hierarchy/area/level design** must be finalized first; everything downstream (LDP, iBGP, inter-AS, TE) depends on stable intra-region routing.
- **V — IGP authentication + metric-style standardization** comes next, hardening and normalizing the now-designed IGP (wide metrics are a prerequisite for later MPLS-TE link attributes, and a converged IGP is required before LDP-IGP sync is meaningful).
- **Z — RR placement, cluster-ID, core iBGP/LDP** depends on each region's IGP being finalized and precedes any inter-AS BGP work (loopback reachability and LSPs must exist first).
- **W — Inter-AS VPN option selection + BGP-LU (Option C)** depends on the intra-region iBGP/LDP and ASBR/PE sessions already existing.
- **Y — MPLS-TE/FRR and QoS/H-QoS for premium customers** is last because it depends on both the core LSP (LDP or RSVP-TE) and the VPN service (L3VPN/VPWS/VPLS) already being operational.

**Reference:** SP network bring-up dependency ordering — IGP → IGP hardening → core BGP/LDP (RR) → inter-AS VPN → TE/QoS overlay (consolidates RFC 4364, RFC 3107, RFC 4090, RFC 3270 dependencies covered in Items 1–29).

---

## End of Phase 1 Solutions

**Answer-key summary:**

| Item | Topic | Answer |
|---|---|---|
| 1 | IS-IS level design | B |
| 2 | IS-IS auth + metric-style | B, E |
| 3 | IS-IS level placement matrix | 1→b, 2→c/b, 3→b, 4→b |
| 4 | OSPF area type | C |
| 5 | ABR redundancy / SPF timers | 5a→A, 5b→B |
| 6 | RR cluster-ID | B |
| 7 | RR redundancy / BGP-LU | 7a→B, 7b→C |
| 8 | eBGP multihop turn-up order | X, Y, W, Z |
| 9 | LDP-IGP sync | B |
| 10 | Targeted LDP + filtering | B |
| 11 | RD/RT scheme | B |
| 12 | Duplicate CE ASN | 12a→B, 12b→A, B |
| 13 | OSPF DN-bit | A |
| 14 | Inter-AS option trade-offs | 1→b, 2→a, 3→c |
| 15 | Blue Corp Option C | C |
| 16 | Option B RT handling | B |
| 17 | AToM control word | B |
| 18 | PW MTU mismatch | B |
| 19 | VPLS full-mesh vs H-VPLS | B |
| 20 | VPLS MAC flap/flood | B |
| 21 | FRR link protection / autoroute | 21a→B, 21b→B |
| 22 | autoroute vs forwarding-adjacency | B |
| 23 | H-QoS nesting | B |
| 24 | Uniform vs pipe mode | B |
| 25 | mVPN SSM + MDT | B |
| 26 | Inter-AS MSDP | A |
| 27 | CoPP | B |
| 28 | RTBH / GTSM | 28a→B, 28b→A |
| 29 | 6PE | A |
| 30 | Capstone sequencing | X, V, Z, W, Y |
