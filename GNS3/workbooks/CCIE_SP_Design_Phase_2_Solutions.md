# CCIE SP Design Phase 2 — Solutions

Companion solutions document for `CCIE_SP_Design_Phase_2.md`. Each item states the correct answer, a brief explanation of why it's correct (and why alternatives are wrong), and the relevant RFC/Cisco reference.

---

## Item 1
**Correct Answer:** Enable BFD for IS-IS on all AS 65100 core links, not just R5–R6 and R3–R4.

**Explanation:** BFD decouples failure detection from IS-IS Hello timers: once bound to the IS-IS adjacency, the interface's BFD session (sub-second, millisecond-interval hellos in the forwarding path) declares the neighbor down and IS-IS tears down the adjacency immediately, instead of waiting out the Hold Time (3x Hello interval, which produced the ~8s detection in the scenario). Limiting BFD to only two links leaves every other core link exposed to the same slow-detection failure mode the NOC just experienced, so the only defensible recommendation for a backbone carrying inter-AS transit is fleet-wide BFD on all core links. The aggressive-BFD-vs-SPF-churn concern is real but is solved by SPF/LSP throttling (Item 2), not by avoiding BFD.

**Reference:** RFC 5880 (BFD), Cisco IS-IS BFD integration ("bfd all-interfaces" / "isis bfd").

---

## Item 2
**Correct Answer:** `spf-interval` (or `spf-interval level-1-2`) with values `<initial-delay> <hold-interval> <max-interval>`, e.g. `spf-interval 50 200 5000` (fast first SPF at 50ms, then exponential backoff doubling up to a 5000ms ceiling if more events occur in the window). The overload-bit mechanism is **IS-IS Overload Bit on Startup** (`set-overload-bit on-startup <seconds> [wait-for-bgp]`), and it does NOT affect routes to R7/R8's own loopback — only transit paths through them.

**Explanation:** The three-timer throttle (initial, hold, max) lets the first topology change trigger SPF almost immediately while subsequent events within the hold/max window back off exponentially, preventing SPF churn from a flapping link — this is the standard Cisco `spf-interval`/`lsp-gen-interval` exponential-backoff model. Overload-bit-on-startup sets the IS-IS Overload bit in the router's own LSP for a bounded time after boot; other routers compute SPF around it for transit but a router with the overload bit still advertises (and other routers still install routes to) its own directly-attached prefixes, including its loopback, because the overload bit only suppresses use of the router as a *transit* hop in SPF, not reachability to the originating router itself.

**Reference:** Cisco IS-IS SPF/LSP throttling (`spf-interval`, `lsp-gen-interval`); ISO/IEC 10589 Overload bit; Cisco "set-overload-bit on-startup" / RFC 3277 context on overload semantics.

---

## Item 3
**Correct Answer:** OSPFv2 and OSPFv3 are structurally different protocols (different LSA encodings, OSPFv3 moves addressing out of the LSA header into prefix-based LSAs, uses link-local addressing for adjacencies, and the AF is carried per-instance rather than per-process-config) — running them in parallel ("ships in the night") is the correct low-risk approach rather than retrofitting one process. Router IDs must still be explicitly configured (32-bit, same syntax as OSPFv2) even on an IPv6-only router since OSPFv3 has no IPv4 address to auto-derive one from; without an explicit `router-id` the process will not fully initialize.

**Explanation:** OSPFv3 (RFC 5340) redefines LSA types to be address-family-agnostic (Router-LSA/Network-LSA no longer carry addresses; Intra-Area-Prefix-LSA carries the prefixes) and runs over link-local IPv6 for neighbor discovery, so it is a separate process/instance from OSPFv2 rather than an extension of it — this is exactly why "parallel" is correct instead of trying to dual-stack a single OSPFv2 process. OSPFv2 auto-derives its Router ID from the highest IPv4 loopback/interface address if not configured, but OSPFv3 has no such fallback once there's no IPv4 address present at all, so Elena's team must explicitly configure `router-id` on any IPv6-only router or the process stays down.

**Reference:** RFC 5340 (OSPF for IPv6); RFC 2328 (OSPFv2) for RID derivation contrast.

---

## Item 4
**Correct Answer:** The **DN-bit (Down bit)** combined with the **OSPF Domain-ID** on the VPN-aware PE routers.

**Explanation:** When a PE redistributes a VPNv4 route back into OSPF toward a CE, it sets the DN-bit in the resulting Type-3 (summary) LSA or Type-5/Type-7 (external) LSA; any other PE in the same OSPF domain that receives this LSA and sees the DN-bit set knows the route already traversed MP-BGP/VPN and will NOT redistribute it back into BGP again, breaking the loop. The Domain-ID further distinguishes which "OSPF domain" (i.e., which customer VPN) a route is tied to so that a route from a *different* OSPF domain sharing the same backbone is correctly treated as a Type-5/7 external rather than mistaken for an intra-domain Type-3. If misconfigured (DN-bit stripped, or Domain-ID mismatched/absent), the second site would treat the re-redistributed Type-3 as native inter-area or the Type-5/7 as a native external, causing routing loops or suboptimal/incorrect route preference.

**Reference:** RFC 4576 (DN-bit for OSPF PE-CE loop prevention); RFC 4577 (OSPF as PE-CE protocol); RFC 1583bis/2328 LSA type semantics.

---

## Item 5
**Correct Answer:** Sham-link is NOT the right tool for the CE-to-CE backdoor link — that is a standard OSPF backdoor-link scenario (no sham-link involved, since no PE/MPLS VPN component sits on that path). Sham-link IS the correct tool for the separate hypothetical PE-to-PE backbone-bypassing physical link, and it is required specifically to make OSPF treat the VPN path as **intra-area** rather than inter-area.

**Explanation:** Sham-link exists to solve a specific problem: when two sites of the same OSPF VPN are connected via MPLS VPN *and* an OSPF backbone-area adjacency would otherwise form across the VPN backbone (which OSPF sees as separate "inter-area" routes via Type-3 LSAs, since the MPLS core isn't part of OSPF Area 0 from the customer's perspective) — the sham-link is a /32-to-/32 unicast tunnel configured between the two PEs' loopbacks that OSPF treats as a numbered intra-area link, forcing proper Type-1/Type-2 LSA exchange and cost-based preference between the VPN path and any real backdoor link. A genuine CE-to-CE leased line is just two OSPF-speaking CEs with a direct link — ordinary OSPF backdoor-link behavior (both paths intra-area already, since neither traverses the PE/VPN backbone) applies, with no need for sham-link because there's no VPN "fake inter-area" path to correct in that part of the topology. This distinction matters for route preference because without a sham-link, the VPN path is seen as inter-area (via Type-3) and the OSPF intra-area path (even if objectively worse) would always be preferred per OSPF's A>IA>E1>E2 route-type-based preference rule, regardless of cost.

**Reference:** RFC 4577 §4.2 (Sham-links); Cisco "OSPF Sham-Link" design guide.

---

## Item 6
**Correct Answer:** BGP **Conditional Route Advertisement**, using `neighbor <peer> advertise-map <route-map> non-exist-map <condition-route-map>` on R21.

**Explanation:** `advertise-map`/`non-exist-map` tells BGP to advertise the routes matched by the advertise-map ONLY IF the prefix matched by the non-exist-map is absent from the BGP table (i.e., R1's primary path is gone) — this exactly implements "only announce backup if primary disappears." The `exist-map` variant does the opposite (advertise only if the condition prefix IS present) and would be the wrong tool here since Blue wants the backup suppressed while the primary is up, not while it's down. The practical risk with a flapping condition prefix is advertisement/withdrawal flapping of the backup route itself — every flap of the primary triggers a BGP UPDATE/WITHDRAW cycle for the backup prefix network-wide, which can cause unnecessary churn and route-dampening penalties on receiving routers.

**Reference:** Cisco "BGP Conditional Advertisement" (advertise-map/exist-map/non-exist-map); RFC 4271 BGP UPDATE semantics.

---

## Item 7
**Correct Answer:** ADD-PATH and BGP PIC solve different problems and are fully independent features; ADD-PATH benefits R7/R8's dual-RR design by letting the RR advertise multiple paths for the same prefix to R1 so R1 has a real backup path pre-installed in its RIB, not just in the RR's.

**Explanation:** BGP PIC (Prefix-Independent Convergence, Edge and Core) is about *precomputing* a backup forwarding entry so that upon failure, the FIB can switch to the alternate next-hop in O(1) time independent of the number of affected prefixes — but it still requires a backup path to actually exist in the BGP table to pre-install. ADD-PATH (RFC 7911) solves the separate problem that, by default, an RR only reflects its single best path per prefix to clients, so a client like R1 may never even learn of an alternate path to pre-install for PIC to use. In the R7/R8 dual-RR design, without ADD-PATH, R1 typically only receives one VPNv4 path per prefix from whichever RR is selected/preferred, so a failure of the advertised path's next-hop can require a new best-path recomputation and reflection cycle; with ADD-PATH between the RRs and R1, R1 can receive paths via both RRs simultaneously (and/or multiple ASBR-origin paths) and PIC can activate instantly against a path that's already resident, which plain PIC Edge alone (without ADD-PATH) cannot guarantee the client even has.

**Reference:** RFC 7911 (BGP ADD-PATH); Cisco "BGP PIC Edge and PIC Core" convergence feature guide.

---

## Item 8
**Correct Answer:** All PEs/ASBRs need extended-community support (BGP communities beyond the basic 2-byte form if 4-byte ASes are involved) with `send-community extended` (or `send-community both`) enabled on relevant sessions; whether the IGP is IS-IS or OSPF is irrelevant since communities are a BGP-layer construct carried independently of the IGP. By default, standard (non-well-known) communities are **stripped at the eBGP boundary** (R6–R12) unless `send-community` is explicitly configured on that eBGP session; Meridian-internal communities need `send-community` enabled toward AS 65200 but then must be filtered back out via an outbound route-map/community-list at R12 (or any further ASBR, e.g., R12–R21 or R20) before readvertisement to AS 65300 to prevent leaking.

**Explanation:** BGP communities are an attribute carried in UPDATE messages at the BGP layer; the underlying IGP (IS-IS vs OSPF) has zero bearing on whether communities propagate, since IGP and BGP are decoupled by design in SP architectures (iBGP/eBGP sit on top of whatever IGP provides reachability). Cisco IOS does NOT send community attributes to a neighbor by default — this is a very common point of confusion precisely because communities DO propagate by default *within* an AS (iBGP, since RRs reflect all attributes unless policy strips them) but do NOT cross an eBGP boundary unless `send-community` is configured on that specific neighbor statement. Because R12 will now also need `send-community` outbound to make Meridian-internal communities usable at the policy layer in AS 65200, an explicit outbound filter (route-map stripping or community-list deny) is required at the next eBGP hop to AS 65300 so those internal-only tags don't leak further than intended.

**Reference:** RFC 1997 (BGP Communities Attribute); RFC 4360 (Extended Communities); Cisco `neighbor send-community` default behavior.

---

## Item 9
**Correct Answer:** By default, MED is only compared between paths received from neighbors in the **same neighboring AS** (same first AS-path hop); `bgp deterministic-med` changes path *grouping/sort order* (groups and compares paths by neighboring AS deterministically regardless of arrival order) without changing eligibility; `bgp always-compare-med` changes *eligibility* by making MED comparable across paths from **different** neighboring ASes. The setting that explains R20 comparing MED across different neighboring ASes is **`bgp always-compare-med`**.

**Explanation:** Default IOS behavior restricts MED comparison to paths that share the same neighboring (adjacent) AS, because MED is meant to be an AS's own hint about its preferred ingress point, not a globally comparable metric across unrelated ASes — comparing it cross-AS is semantically meaningless since different ASes may use wildly different MED scales/policies. `bgp deterministic-med` only fixes a *non-determinism* bug: without it, best-path results can depend on the arbitrary order paths were received in when multiple paths from the same neighboring AS exist; it sorts/groups same-neighboring-AS paths before comparison but never widens eligibility to other ASes. Only `bgp always-compare-med` removes the same-neighboring-AS restriction entirely, so it is the only one of the two that could produce MED comparison across different neighboring ASes on R20 — `deterministic-med` cannot cause that symptom.

**Reference:** Cisco BGP Best Path Selection Algorithm (MED comparison rules); RFC 4271 §9.1.2.2 (MED_COMPARISON caveats).

---

## Item 10
**Correct Answer:** Recommend **facility backup** for the R3-R4-R5-R6 ring given the current ~40 tunnels and projected growth to 150+.

**Explanation:** Facility backup (RFC 4090's "facility" method, implemented via Next-Hop or Next-Next-Hop backup tunnels) protects the *link* (or node) itself with a single backup LSP that all primary LSPs traversing that link/node can share — it relies on the RSVP-TE FRR extensions that signal protection at a per-facility (interface) granularity, scaling as O(number of protected facilities) rather than O(number of primary LSPs). One-to-one backup builds a dedicated detour LSP *per protected LSP*, so backup-LSP count scales linearly with tunnel count — fine at 40 tunnels but a serious state/signaling/control-plane burden as the count triples to 150+. Since Meridian's trajectory is explicit growth, facility backup is the only one of the two that keeps backup-plane overhead flat regardless of how many TE tunnels eventually cross the ring.

**Reference:** RFC 4090 (Fast Reroute Extensions to RSVP-TE); Cisco MPLS TE FRR facility vs. one-to-one backup design guide.

---

## Item 11
**Correct Answer:** DS-TE with a sub-pool for voice (reserved, protected bandwidth with its own bandwidth-constraint model, e.g. RDM or MAM) and the global pool for data traffic, where data tunnels can only consume what's left after sub-pool reservations are honored; auto-bandwidth adjusts the tunnel's **RSVP reservation** (the signaled/reserved bandwidth value), not the physical link allocation — and yes, a misconfigured auto-bandwidth-enabled data tunnel sized against the *global* pool (rather than correctly bounded) could still indirectly pressure available capacity if bandwidth constraints aren't properly partitioned, though it cannot directly "steal" sub-pool-reserved bandwidth if DS-TE is correctly configured with separate pools.

**Explanation:** DS-TE (RFC 4124/4125/4127) partitions a link's bandwidth into two pools evaluated by CSPF with different constraint models, so a voice tunnel admitted into the sub-pool is guaranteed not to be oversubscribed by global-pool (data) tunnels, which can only use remaining global-pool capacity — this is precisely the "guarantee voice, let data use leftover" design Priya wants. Auto-bandwidth (a Cisco TE feature) periodically measures actual traffic through a tunnel and resignals a new RSVP bandwidth *reservation* to match demand — it does not touch physical interface bandwidth/QoS queue allocation, which remains a separate mechanism (LLQ/CBWFQ) operating on the wire. The risk Priya should actually worry about is misconfiguring the *bandwidth-constraint model/pool assignment* (e.g., accidentally letting a data tunnel's auto-bandwidth-driven reservation request bandwidth from the sub-pool, or not correctly capping global-pool max), not auto-bandwidth itself "stealing" — properly scoped DS-TE pools prevent that by construction.

**Reference:** RFC 4124 (Protocol Extensions for DS-TE); RFC 4125/4127 (RDM/MAM Bandwidth Constraints Models); Cisco MPLS TE Auto-Bandwidth feature guide.

---

## Item 12
**Correct Answer:** VRF-Lite provides local-only VRF route separation with no MP-BGP VPNv4 control plane and no MPLS label-based data plane across the backbone; it lacks any-to-any multi-PE connectivity, so expanding to a second site on R15 would require a full redesign (migrating to RD/RT + MP-BGP VPNv4). Recommendation: use **VRF-Lite only if the customer is confirmed single-site permanently**; otherwise build it as full MPLS L3VPN from the start to avoid a disruptive later migration.

**Explanation:** VRF-Lite is simply multiple VRF routing tables on a single router (or statically/IGP-interconnected routers) with no BGP VPNv4 signaling and no MPLS transport — it only separates routing locally and has no mechanism to extend that isolation across the backbone to a second PE. Full MPLS L3VPN uses RD to make routes unique and RT with MP-BGP VPNv4 to automatically propagate and import/export routes between any PEs configured with matching RTs, so adding R15 later is just a config change (new VRF + RT import/export), not an architecture change. Since Sales is already discussing growth to a second site, recommending VRF-Lite purely to save setup cost creates technical debt that forces a redesign the moment Green's second site materializes — the safer recommendation is full L3VPN now, accepting slightly higher day-one overhead for painless scaling.

**Reference:** Cisco "VRF-Lite" vs. "MPLS VPN" design comparison; RFC 4364 (BGP/MPLS IP VPNs).

---

## Item 13
**Correct Answer:** 6VPE uses the **VPN-IPv6 (VPNv6) address family** in MP-BGP — NLRI = Route Distinguisher + IPv6 prefix, carried as an MP_REACH_NLRI AFI=2 (IPv6)/SAFI=128 (MPLS-labeled VPN) attribute with an IPv4-mapped-IPv6 BGP next-hop (so it can still recurse onto the IPv4 LSP in the core). Data plane: the packet from R1 to R21 still carries a standard **2-label stack** — outer transport label (LDP/IGP, swapped hop-by-hop through the IPv4 MPLS core) and inner VPN label identifying the IPv6 VRF at egress — the IPv6 payload itself is simply the content beneath the label stack. **P routers (R3, R4, R11, R13) do NOT need any IPv6 support** — they only ever see IPv4-signaled MPLS labels.

**Explanation:** 6VPE is explicitly designed to reuse the exact same IPv4 MPLS transport infrastructure and VPNv4-style control plane pattern, just extended to a new AFI/SAFI (VPN-IPv6) — it never requires the core to run native IPv6 or dual-stack, which is exactly why it was proposed instead of a parallel 6PE/dual-stack-core redesign. The BGP next-hop for a VPNv6 route is encoded as an IPv4-mapped IPv6 address so that the PE can resolve it right back onto the same IPv4-signaled LSP used for VPNv4 traffic — the IPv6-ness is confined to the control-plane NLRI and the customer payload, never touching the transport label's signaling. Because the P-router data plane only ever swaps MPLS labels (agnostic to what AF the inner payload belongs to), R3/R4/R11/R13 remain completely untouched by this rollout — only the PE routers (R31, R21, and other VRF-holding PEs) need VPNv6 configuration.

**Reference:** RFC 4659 (BGP-MPLS IP VPN Extension for IPv6 VPN / 6VPE); RFC 4364 for the base VPNv4 model it extends.

---

## Item 14
**Correct Answer:** RT-Constraint uses the **RT-Constraint (RTC) address family**, AFI=1 (or 2)/SAFI=132, where a PE/RR signals "I am interested in RT X" as an NLRI itself; R30 uses this to prune which VPNv4 routes it reflects to a given peer, only sending routes whose RT matches an RTC advertisement received from (or on behalf of, via RR-aggregated interest) that peer. For RT-Constraint to work end-to-end rather than silently failing partway, **every router in the propagation path — PEs, RRs, and any ASBRs relaying VPNv4 routes between RT-Constraint domains — must support and have RT-Constraint enabled**; any non-participating hop in the path will fall back to flooding everything through that segment, defeating the filtering.

**Explanation:** RT-Constraint (RFC 4684) inverts the normal "receiver filters after receiving everything" model into "advertise your RT interest first," making the RTC NLRI itself a BGP update that an RR like R30 consumes to build a per-neighbor set of RTs to actually reflect — this is what lets R30 avoid reflecting Green's VPNv4 routes to R22, which has no matching RT interest. Because this filtering only happens at RT-Constraint-aware hops, if even one router in the path (e.g., an older ASBR or a PE without RTC support) doesn't participate, that router will pass through (or request/receive) full unfiltered VPNv4 updates, meaning the "savings" simply don't materialize across that segment — the feature degrades gracefully (doesn't break routing) but silently fails to deliver its scalability benefit unless deployed consistently end-to-end.

**Reference:** RFC 4684 (Constrained Route Distribution for BGP/MPLS IP VPNs — "RT-Constraint").

---

## Item 15
**Correct Answer:** In CSC, the "CE" (the wholesale ISP's actual PE) exchanges **labeled routes (BGP-LU / RFC 3107, or an IGP+label mechanism)** with Meridian's PE, not just plain customer prefixes — because the wholesale customer needs its own VPN *labels* carried across Meridian's core, which requires the inner VPN label to survive untouched end-to-end. Meridian's P routers (R11, R13) need **no awareness whatsoever** of the wholesale customer's inner VPN labels — CSC keeps that completely hidden from the P-router core exactly like a normal L3VPN hides customer payload from P routers.

**Explanation:** In standard L3VPN PE-CE, the CE exchanges plain IP routes (eBGP/OSPF/static) with no labels involved, because the PE itself imposes the VPN label; in CSC the "CE" (really the wholesale PE) must advertise its own routes *with labels attached* (via BGP-LU/RFC 3107, or by running MPLS/LDP on the attachment circuit) because Meridian's PE needs a 3-label stack capability — Meridian's own IGP-transport label, Meridian's VPN label for the wholesale customer's VRF, and the wholesale customer's own inner label underneath that are both pushed through unmodified. Meridian's P routers never need to know this is happening because, exactly as in ordinary L3VPN, P routers only ever process the top (transport) label — the deeper VPN and wholesale-inner labels ride underneath, invisible to anything doing plain LDP/IGP label switching.

**Reference:** RFC 3107 (Carrying Label Information in BGP-4, used for BGP-LU in CSC); Cisco "MPLS VPN Carrier Supporting Carrier" design guide.

---

## Item 16
**Correct Answer:** Full label stack walkthrough R31→R21 (Option C, multihop eBGP VPNv4 + BGP-LU for ASBR loopbacks):
- **R1 (ingress PE):** imposes a 2-label stack: top = IGP/LDP transport label toward R6 (the next-hop *to the ASBR*, learned via BGP-LU resolving to R6's loopback), bottom = VPN label (allocated by R21, the egress PE, since VPNv4 next-hop is R21 directly in Option C).
- **R3 (P router):** swaps only the top (LDP) label; VPN label underneath is untouched/opaque to R3.
- **R6 (ASBR, AS65100 egress):** pops the IGP/LDP label (penultimate-hop-popped or exposed via explicit-null depending on config) since R6 is R1's BGP-LU resolution target; re-pushes a *new* top label — this time the BGP-LU label R20 assigned for reaching R6/R20 loopbacks across the inter-AS eBGP VPNv4+labeled-unicast session — VPN label (R21's) remains unchanged underneath throughout.
- **Inter-AS link R6↔R20:** packet crosses with [BGP-LU label][VPN label].
- **R20 (ASBR, AS65300 ingress):** swaps the BGP-LU label for an IGP/LDP label now valid inside AS 65300 toward R21; VPN label still unchanged.
- **R23 (P router):** swaps only the top (LDP) label.
- **R21 (egress PE):** pops the top (transport) label (PHP from R23 or explicit-null) and uses the VPN label to identify Customer Blue's VRF, pops it, and forwards natively into the VRF.

**Explanation:** Option C's defining characteristic is that the VPNv4 route itself (with its original VPN label from the egress PE, R21) is carried unmodified end-to-end via multihop eBGP between RRs (R7↔R30), while *reachability to the ASBR/PE loopbacks* needed to resolve that BGP next-hop is provided by BGP-LU (RFC 3107) hop-by-hop across the ASBR boundary — this is exactly why the VPN label never changes anywhere in the path (R21 allocated it once; everyone else only manipulates the transport label above it). The transport label is what actually changes identity at R6 and R20: inside each AS it's a normal LDP/IGP label, but across the inter-AS link itself it is a BGP-LU-signaled label specifically because plain LDP doesn't span AS boundaries — this is the one spot in the walkthrough where "LDP label" becomes "BGP-LU label" and then becomes "LDP label" again.

**Reference:** RFC 4364 §10 (Inter-AS Option C / "Multihop EBGP redistribution of labeled VPN-IPv4 routes"); RFC 3107 (BGP-LU).

---

## Item 17
**Correct Answer:** Comparison matrix and recommendation = **Option A (back-to-back VRF)**.

| Criterion | Option A | Option B | Option C |
|---|---|---|---|
| Labels cross AS boundary? | No — plain IPv4/VRF routes exchanged per-VRF, no MPLS | Yes — labeled VPNv4 exchanged ASBR-to-ASBR | Yes — VPN label (end-to-end) + BGP-LU transport label |
| ASBR config overhead | High — one VRF + eBGP session (or sub-interface) per customer at the boundary | Moderate — single multihop/directly-connected labeled VPNv4 session, no per-VRF ASBR config | Low — single multihop eBGP VPNv4 session between RRs, ASBRs only need BGP-LU |
| Scalability as VRF count grows | Poor — linear growth in ASBR interfaces/sessions/VRFs | Good — one session handles all VRFs | Best — ASBRs stay VRF-agnostic entirely |
| Matches "no MPLS/no label exchange across boundary"? | **Yes** | No | No |

**Explanation:** Option A redistributes VPNv4 into plain IPv4 (or vice versa) at the ASBR per-VRF, with ordinary (non-labeled) eBGP exchanged across the boundary — exactly matching the counterpart's explicit requirement to not run MPLS or exchange labels across the new link. Option B keeps labels on VPNv4 routes exchanged directly between ASBRs (requiring MPLS/label-signaling across the boundary) and Option C goes further by not even terminating the VPNv4 session at the ASBR (multihop to RRs) while still requiring BGP-LU-signaled labels across the boundary for next-hop resolution — both options B and C inherently require MPLS label exchange across the AS boundary, directly violating the counterpart's stated constraint, so despite A's worse per-VRF scalability, it is the only option technically compatible with their requirement.

**Reference:** RFC 4364 §10 (Options A/B/C for Interprovider VPNs).

---

## Item 18
**Correct Answer:** SSM requires the receiver to know both the group **and the specific source address** in advance (expressed as an (S,G) join rather than a (*,G) join), eliminates the need for an RP for *that* traffic since there's no shared tree to build, but does **NOT** eliminate the need for BSR/Auto-RP elsewhere in AS 65200 for any other multicast application that uses ASM-style (*,G) joins where receivers don't know the source ahead of time.

**Explanation:** ASM (RFC 1112's original model) lets a receiver join a group without knowing the source, which requires an RP to bootstrap the shared tree (via PIM-SM + BSR/Auto-RP for RP discovery) before SPT switchover — SSM (RFC 4607/3569) instead requires the receiver (or, practically, the IGMPv3/MLDv2-capable last-hop router on its behalf) to already know the exact (S,G) pair, letting PIM build a source tree directly with no RP involved at all. Because IPTV's EPG model gives receivers the source IP in advance, SSM is correctly simpler for that specific application and genuinely removes the RP dependency for it — but any other multicast application in AS 65200 that still relies on receivers only knowing the group (not the source) will still need ASM's RP infrastructure, so BSR/Auto-RP cannot be decommissioned network-wide just because IPTV moves to SSM.

**Reference:** RFC 4607 (Source-Specific Multicast for IP); RFC 4601 (PIM-SM); Cisco "PIM-SSM vs PIM-SM" design comparison.

---

## Item 19
**Correct Answer:**
- **Profile 0:** Default MDT via GRE/mGRE in the core, PIM-based C-multicast signaling (classic draft-rosen style MVPN).
- **Profile 3:** Default MDT via GRE, BGP-based C-multicast signaling (replaces PIM C-mcast signaling, keeps GRE P-tree).
- **Profile 6:** Partitioned MDT via Ingress Replication (no mGRE/P-tunnels, unicast replication over MPLS instead), BGP-based C-multicast signaling.
- **Profile 7:** Partitioned MDT via P2MP TE (RSVP-TE P2MP LSPs in the core), BGP-based C-multicast signaling.
- **Profile 14:** Partitioned MDT via P2MP with BGP-based C-multicast signaling using Global Table Multicast context (no VRF-bound default MDT model, more BGP-centric end to end).

Recommended for Customer Yellow: **Profile 6 (Ingress Replication) or Profile 7 (P2MP TE)** — both satisfy "MPLS/SR in the core, BGP auto-discovery instead of PIM," with Profile 6 preferred if replication fan-out is modest, Profile 7 if traffic volume justifies true P2MP LSP replication efficiency.

**Explanation:** NG-MVPN (RFC 6513/6514) profiles differ along two independent axes — the P-tunnel/core-tree mechanism (GRE/mGRE, Ingress Replication, P2MP RSVP-TE, or P2MP mLDP) and the C-multicast signaling method (classic PIM vs. BGP auto-discovery/signaling per RFC 6514) — and Yellow's requirement explicitly rules out any profile still using mGRE/PIM in the core (eliminating profiles 0 and effectively narrowing the GRE-based profile 3 since it still uses GRE P-tunnels even though its C-mcast signaling is BGP-based). Profiles 6 and 7 both use genuinely MPLS-native P-tunnel mechanisms (Ingress Replication requires no special core P2MP state; P2MP TE uses RSVP-TE's P2MP extensions) paired with BGP-based auto-discovery and C-multicast signaling, matching Yellow's stated requirement precisely, whereas profile 14 is a further architectural variant (global-table context) not specifically necessary for Yellow's VPN-scoped multicast request.

**Reference:** RFC 6513 (MVPN), RFC 6514 (BGP Encodings for MVPN); Cisco NG-MVPN Profile reference table.

---

## Item 20
**Correct Answer:** Per-VRF policing is applied at the **VRF/subinterface (logical) boundary within the forwarding path**, not on the physical interface's aggregate rate — and yes, R1 can absolutely apply independent policing rates per VRF on the SAME physical interface (or across its multiple physical/logical interfaces) because the policer is attached to the traffic class/VRF-scoped policy-map, not to the wire itself.

**Explanation:** Interface-level policing enforces a single rate against all traffic traversing that physical interface regardless of which customer/VRF it belongs to, which is wrong for R1 since g1/0 (or any shared interface) may carry several customers' VRFs that should each get independently enforced contracted rates. Per-VRF policing is implemented via a hierarchical/VRF-aware policy-map (or policing applied on a VRF-specific sub-interface/VLAN) that classifies traffic into its owning VRF context first and then polices within that context, so Customer Green's 50 Mbps contract can be enforced strictly on Green's traffic while a different customer sharing the same physical port is governed by its own independent policer — this is standard practice precisely because policing is a QoS policy-map construct that can be scoped as granularly as the classification allows, not an inherent property "baked into" the physical interface.

**Reference:** Cisco MQC (Modular QoS CLI) hierarchical policy-maps; Cisco "Per-VRF QoS Policy" design guide.

---

## Item 21
**Correct Answer:** WRED provides a real benefit primarily for **TCP-heavy traffic** (Customer Green/Yellow/Blue) by randomly dropping packets before the queue fully fills, triggering TCP congestion-avoidance (slow start/backoff) on a subset of flows early and avoiding the synchronized "global sync" collapse that tail-drop can cause across many TCP sessions hitting the same full queue simultaneously; it offers little/no benefit for UDP-based IPTV traffic (which doesn't react to drops with congestion backoff) and can actually hurt UDP-sensitive real-time flows by dropping packets indiscriminately. EXP remarking should happen only at **PE ingress**, not mid-path on a P router like R3 — core P routers should not remark EXP.

**Explanation:** WRED's entire value proposition rests on TCP's responsiveness to loss (via its congestion window), so for Green/Yellow/Blue's enterprise TCP traffic, WRED's early random drops smooth out queue buildup and prevent the "tail-drop causes every TCP flow to back off simultaneously, then re-ramp simultaneously" oscillation pattern — for UDP/IPTV, there's no congestion-window feedback loop to trigger, so WRED's early-drop benefit doesn't apply and dropping IPTV packets early (vs. at the tail) provides no advantage and can degrade video quality unnecessarily. EXP remarking mid-path on a P router is explicitly against the standard SP design principle that classification/marking should happen once, at the trust boundary (PE ingress, where the provider can inspect/trust the DSCP-to-EXP mapping against the SLA) — core P routers are designed to be fast, simple, EXP-only-queueing devices that trust the marking already applied, and remarking mid-path reflects a provisioning error that should be fixed at the PE, not patched by adding remarking logic deeper in the core (which also breaks the "P routers don't need per-flow awareness" scalability principle).

**Reference:** RFC 2309 (RED recommendations, basis for WRED); Cisco DiffServ QoS design — "mark once at the edge" / MPLS DiffServ Tunneling Modes (RFC 3270) context for EXP handling.

---

## Item 22
**Correct Answer:** TCP-AO (RFC 5925) replaces MD5's weak MD5 hash with modern, pluggable cryptographic algorithms (e.g., HMAC-SHA-1, AES-128-CMAC) and supports **key rollover without bouncing the session** via simultaneous active Master Key Tuples (MKTs) identified by Key ID (KeyID/RNextKeyID) fields, so a new key can be introduced and the peers can transition to it while the session stays up — this is NOT a drop-in replacement; it requires a materially different configuration model (key chains with algorithm + SendID/RecvID + lifetimes) rather than a single shared MD5 password string.

**Explanation:** MD5 for BGP (RFC 2385) uses a single static password hashed with MD5 (a cryptographically weak, collision-prone algorithm by modern standards) and changing that password requires both sides to update simultaneously or the TCP session resets — causing the outages Marcus describes. TCP-AO addresses the hash weakness by supporting modern algorithms via its negotiated/configured MKT, and addresses key rollover by allowing multiple MKTs to be active at once (each tagged with a KeyID), so routers can advertise "I'm about to switch to KeyID N" via RNextKeyID and both sides transition without ever tearing down the TCP connection — this is structurally different from MD5's single-password model, requiring Meridian's ASBRs to configure key chains (algorithm, key strings, send/receive IDs, and optionally lifetimes) rather than a simple `neighbor x.x.x.x password`.

**Reference:** RFC 5925 (TCP Authentication Option); RFC 2385 (MD5 for BGP, for contrast); RFC 5926 (TCP-AO Cryptographic Algorithms).

---

## Item 23
**Correct Answer:** BGP Flowspec (RFC 8955/8956, formerly 5575) can match on far more than destination prefix — source/destination prefix, protocol, source/destination port, ICMP type/code, TCP flags, packet length, DSCP, fragment type — and automates DDoS response by distributing a filter/rate-limit/redirect action as a BGP NLRI update propagated to all participating routers in seconds; the main operational risk is an overly broad Flowspec rule unintentionally matching and dropping/rate-limiting legitimate traffic (collateral damage) network-wide, since the rule propagates automatically and immediately with the same speed/blast-radius as the attack response itself. RPKI/ROV (ROAs validating origin AS) would have caught the Customer Yellow near-miss by invalidating a route that claims the prefix with the wrong origin ASN; RPKI does NOT protect against AS-path manipulation attacks where the hijacker uses the *legitimate* origin AS as the final AS-path element (e.g., sophisticated path-prepending/forged-AS-path hijacks) — it only validates origin, not full AS-path integrity.

**Explanation:** Flowspec's rich n-tuple matching (RFC 8955) is specifically designed so that volumetric/application-layer attack traffic can be matched on characteristics beyond destination prefix (e.g., specific source ports or packet-length signatures typical of a given attack type), and because it's signaled as a normal BGP update it propagates at BGP speed to every Flowspec-enabled router simultaneously — turning a 25-minute manual-ACL process into a near-instant network-wide push; the risk is exactly proportional to that speed/reach — a too-broad match criterion becomes a self-inflicted outage at scale instantly. RPKI (RFC 6480) lets Meridian cryptographically assert via a ROA that AS 65300 (or whichever AS legitimately originates the Yellow prefix) is authorized to originate it, so Route Origin Validation at receiving routers would mark the typo'd other-ISP origination as Invalid and (depending on policy) reject or deprioritize it — but RPKI's ROV only validates the *origin* AS of a prefix, not the full path, so it is blind to a hijack where an attacker forges an AS-path that still ends in the correct/legitimate origin AS (a "path hijack" or more advanced route-leak-style attack), which is why RPKI is a partial, not complete, hijack defense.

**Reference:** RFC 8955/8956 (BGP Flowspec); RFC 6480 (RPKI); RFC 6811 (BGP Prefix Origin Validation / ROV).

---

## Item 24
**Correct Answer:** T-LDP VPWS signals the pseudowire label binding via **targeted LDP sessions** (RFC 4447/8077, directed LDP between the two PE loopbacks) carrying the VC-FEC/VC-label mapping; EVPN-VPWS signals it via **MP-BGP** using the EVPN NLRI (RFC 7432/8214) instead of LDP. EVPN-VPWS enables multihoming redundancy structurally because the pseudowire endpoint is identified by an **Ethernet Segment Identifier (ESI)** advertised in BGP, allowing multiple PEs to represent the same multihomed CE site and support active/active or active/standby forwarding with BGP-driven mass-withdrawal on failure — a model T-LDP VPWS has no native equivalent for.

**Explanation:** Classic T-LDP VPWS is fundamentally a point-to-point, two-PE-only construct — the pseudowire is a directed LDP session between exactly two PE loopbacks with no concept of "multiple PEs serving the same CE site," so any redundancy has to be bolted on via external mechanisms (e.g., a backup PW with LDP status-based preferential forwarding, as in Item 25) rather than being structural. EVPN-VPWS instead advertises per-attachment-circuit routes tagged with an ESI in BGP, so when a CE is multihomed to two or more PEs sharing the same ESI, those PEs can coordinate designated-forwarder election and BGP can trigger a fast, protocol-native "mass withdrawal" of all services tied to a failed PE — because the signaling plane is BGP (already carrying rich multi-path/multihoming semantics via route-type NLRIs) rather than a strictly bilateral LDP session, true active/active multihoming becomes natively expressible in a way T-LDP VPWS's point-to-point design cannot cleanly replicate.

**Reference:** RFC 4447/8077 (LDP-signaled pseudowires); RFC 8214 (EVPN VPWS); RFC 7432 (EVPN, ESI/multihoming concepts).

---

## Item 25
**Correct Answer:** During T-LDP pseudowire signaling, each PE advertises its **interface MTU as part of the Interface Parameters field of the LDP VC-FEC (0x80 or similar sub-TLV)** in the label mapping message; if the two PEs' MTUs don't match, the pseudowire **fails to come up entirely** (it does not come up "degraded") — LDP explicitly refuses to bind the VC label when interface parameters disagree. For PW redundancy, the mechanism is **MPLS/LDP pseudowire status (RFC 4447/8077 "PW status TLV")**, where a backup PW is signaled up but held in **standby** by sending a status code indicating "not forwarding"/"standby," and when the primary PW's status changes to down, the backup transitions to active — all without requiring the CE to run any special protocol since the entire mechanism lives in the LDP control plane between the PEs.

**Explanation:** The LDP Interface Parameters sub-TLV in a pseudowire's label mapping explicitly carries MTU (among other L2-specific parameters like interface description, VLAN ID, etc.), and because an MTU mismatch would mean frames exceeding one side's capacity could never be forwarded correctly, LDP treats a hard MTU mismatch as fatal to that FEC binding — the PW simply never transitions to the "up" state rather than coming up with a reduced/negotiated MTU (there's no "auto-negotiate down" fallback). Pseudowire redundancy works by having a backup PW already signaled (so labels are pre-allocated end-to-end) but marked via the PW status TLV as standby/not-forwarding; this is purely a PE-to-PE LDP control-plane signal, so the CE sees nothing different and needs no awareness of which PW is active — the moment the primary PW's status TLV indicates a fault, the backup PE flips its own PW status to active and traffic switches without CE involvement.

**Reference:** RFC 4447/8077 (Pseudowire Setup and Maintenance Using LDP) — Interface Parameters sub-TLV and PW Status TLV; RFC 6718 (PW Redundancy framework).

---

## Item 26
**Correct Answer:** On R3/R4, each FEC's label source is determined per-prefix by whichever protocol (LDP or SR/ISIS-advertised SID) is configured/preferred for that prefix — Cisco IOS-XR/XE support LDP and SR coexisting on the same interface simultaneously via **LDP-SR interworking**, where the forwarding decision for a given destination prefix depends on which binding is installed (commonly SR is preferred if both exist for the same prefix, controlled by platform-specific preference rules). An **SR Mapping Server** is used to advertise SID bindings for *non-SR-capable, LDP-only remote nodes/prefixes* on behalf of routers that can't participate in SR themselves — it is NOT primarily an LDP/SR interworking boundary mechanism for the R31/R2-to-not-yet-converted-egress scenario Priya describes; that scenario is handled by normal SR/LDP coexistence and the mapping server would only be relevant if some remote PE could never run SR at all.

**Explanation:** Both LDP and IS-IS-advertised Prefix-SIDs can install forwarding entries for the same prefix on R3/R4 simultaneously, and Cisco platforms resolve which one wins per-FEC (SR is typically preferred when both are present, since Prefix-SIDs are IGP-native and considered authoritative once advertised) — this lets R3/R4 forward SR-originated traffic from an already-converted R1 via SR labels while still forwarding LDP-originated traffic from a not-yet-converted neighbor via LDP labels, on the exact same interfaces, with no flag day required. The SR Mapping Server (RFC 8661 context / Cisco SR migration guide) exists specifically for the case where a router can never run SR (e.g., a legacy platform limitation) — it advertises a Prefix-SID *on behalf of* that non-SR node so SR-capable routers elsewhere in the network can still steer traffic toward it via SR; it is a tool for permanent SR/non-SR coexistence boundaries, not the general transient-migration coexistence Priya is describing (where R3/R4 themselves are fully SR-capable and just need to support both signaling types during the window).

**Reference:** Cisco SR Migration Guide ("LDP and Segment Routing Interworking"); RFC 8661 (SR Mapping Server for LDP interworking) for the actual mapping-server use case.

---

## Item 27
**Correct Answer:** Classic LFA's key limitation is that it can only provide a loop-free backup next-hop when the network topology happens to offer one (not guaranteed — many topologies, especially rings or asymmetric-metric networks, have no LFA-eligible neighbor for a given destination). Remote LFA addresses this by tunneling (via a targeted LDP session or similar) traffic to a "PQ node" further away in the topology that IS loop-free, extending coverage beyond what a directly-connected LFA neighbor can offer. TI-LFA (Topology-Independent LFA) improves further by explicitly computing a backup path that matches the **post-convergence path** (the path the network will converge to anyway after SPF reruns), guaranteeing not just loop-freeness but also that traffic doesn't take a suboptimal detour that gets immediately replaced once convergence completes. TI-LFA does **not** have a hard dependency on SR — the concept works with LDP too (using targeted LDP to the PQ node, similar to remote LFA), though it is most commonly deployed and most elegantly implemented with SR because SR's explicit segment stacking makes expressing the repair path to the PQ node trivial without needing a separate tunnel/session.

**Explanation:** Classic per-prefix LFA (RFC 5286) only works when a neighbor other than the primary next-hop already offers a loop-free alternate path by the plain SPF/Dijkstra criterion — in symmetric-cost rings or certain triangle/square topologies, no such neighbor exists, leaving the destination entirely unprotected by LFA. Remote LFA (RFC 7490) extends coverage by computing a "repair tunnel" to a node (the PQ node, at the intersection of the P-space of the point of local repair and the Q-space of the destination) that is guaranteed loop-free even when no direct neighbor is, but it still only guarantees loop-freeness, not that the repair path matches what the network will look like after full convergence — so traffic can take a temporarily suboptimal (though not looping) path. TI-LFA (an SR-centric but conceptually protocol-independent idea) explicitly computes the backup as the segment list that reproduces the exact post-convergence SPF path, so there's no "repair, then reroute again once SPF catches up" transient — while this is overwhelmingly deployed with SR-MPLS/SRv6 because SIDs make expressing an arbitrary explicit path trivial, the core post-convergence-alignment concept does not strictly require SR and the repair-tunnel mechanics underlying it borrow directly from remote LFA's LDP-based approach.

**Reference:** RFC 5286 (Loop-Free Alternates); RFC 7490 (Remote LFA); Cisco/IETF TI-LFA (draft-ietf-rtgwg-segment-routing-ti-lfa).

---

## Item 28
**Correct Answer:** SR-TE ODN triggers automatic SR-TE policy instantiation on R31 (the ingress PE) when a VPNv4 route is learned carrying a specific **color extended community**, which R1 maps (via a locally configured color-to-SLA/intent mapping) to an on-demand SR Policy with a defined optimization objective/constraint (e.g., low-latency). The PCE's role is to perform the actual **constrained path computation** (especially for constraints like end-to-end latency that aren't directly expressed in IGP metrics) using a broader, often multi-domain or TE-metric-aware topology database, whereas R1 (the headend) on its own typically only computes paths using what it locally knows via IGP flooding (e.g., plain SPT or basic SR-TE with simple affinity/metric constraints) — delegating latency-aware computation to a centralized PCE is useful because the PCE can maintain and compute against a latency-specific TE database (e.g., via BGP-LS collected link-delay metrics) spanning the whole network, which a single headend's local view may not fully capture, especially across multiple IGP domains like Meridian's three ASes.

**Explanation:** ODN (On-Demand Next-hop) works by having the ingress PE recognize a BGP color community on a received VPNv4 route and automatically instantiate (rather than requiring an operator to hand-configure) an SR Policy whose color matches, with the policy's intent (e.g., "low-latency") predefined by Meridian's local policy templates — this is exactly the automation Marcus wants instead of manually building SR-TE policies on R31 and R21. A headend can compute simple SR-TE paths on its own using IGP-flooded TE extensions (link delay sub-TLVs, SRLG, affinity), but for a genuinely cross-domain or complex multi-constraint optimization (like guaranteed low latency across AS boundaries where IGP flooding alone may not have visibility, as in Blue's AS65100-to-AS65300 case), a centralized PCE with a richer, often BGP-LS-fed multi-domain topology/latency database can compute a globally better-informed path and simply hand the headend the resulting SID list — letting the PCE do the heavy constrained computation while the headend just signals/steers traffic onto the resulting policy.

**Reference:** Cisco/IETF SR-TE On-Demand Nexthop (draft-ietf-idr-tunnel-encaps / Cisco ODN+Color implementation guide); RFC 8231/8281 (PCEP extensions for stateful PCE); RFC 9552 (BGP-LS) for cross-domain TE data feeding the PCE.

---

## Item 29
**Correct Answer:** Plain SRv6 SIDs are full 128-bit IPv6 addresses, so a multi-segment explicit path requires stacking multiple 128-bit SIDs in the SRH, which quickly consumes significant packet header overhead and can strain hardware SID-list-depth limits — uSID/C-SID (Compressed SID) addresses this by packing multiple 16-32 bit "micro-SIDs" into a single 128-bit SID container, dramatically reducing the per-segment overhead for scaled locator planning. In an SRv6 L3VPN, the VPN "label" function is realized by a **per-VPN SID (a distinct SRv6 SID function bound to the VRF/service)** rather than an MPLS label — the destination/last SID in the SID list itself identifies which VRF to forward the decapsulated payload into, replacing the inner VPN-label lookup. **PSP (Penultimate Segment Pop)** means the second-to-last node in the SID list pops/removes the outer SRv6 encapsulation (the IPv6+SRH header) before forwarding to the final node, analogous to PHP in MPLS; **USD (Ultimate Segment Decapsulation)** means the LAST node itself performs the decapsulation (removing the SRv6 header) rather than the penultimate one — the choice between PSP and USD determines which node (second-to-last vs. last) strips the outer header before the inner payload/VPN-SID lookup happens.

**Explanation:** A full 128-bit SID per segment means a path with several waypoints can produce a very deep SRH (Segment Routing Header) SID list, inflating packet size and taxing ASIC lookup/SID-list-depth limits — uSID (Cisco) / C-SID (IETF-standardizing term) solves this by compressing multiple short (commonly 16-bit) micro-segment identifiers into the space of a single 128-bit SID, so one "SID slot" in the list can represent several hops worth of steering instructions, directly addressing the header-size/scale concern Marcus is asking about. For SRv6 L3VPN, instead of an MPLS VPN label identifying the VRF after a transport label is popped, the SRv6 architecture binds a specific SID (often called the "End.DX4/DX6" or VPN-specific SID function) directly to the per-VRF forwarding behavior, so simply routing the packet to that SID inherently selects the correct VRF — no separate inner label lookup is needed because the SID itself encodes "decapsulate and forward into VRF X." PSP and USD are both end-of-path header-removal behaviors defined in the SRv6 Network Programming model, differing only in which node performs it (second-to-last vs. last), analogous to how MPLS PHP removes the top label one hop before the true destination to save that final node a lookup — PSP shifts the decapsulation burden earlier, while USD leaves it to the final node itself.

**Reference:** RFC 8986 (SRv6 Network Programming — PSP/USD/USP behaviors); draft-ietf-spring-srv6-srh-compression (uSID/C-SID); RFC 9252 (BGP Overlay for SR Policies / SRv6 L3VPN context).

---

## Item 30
**Correct Answer:** gRPC **dial-out** telemetry means the router (the device being monitored) initiates the outbound TCP/gRPC connection to the collector; gRPC **dial-in** means the collector initiates the connection inbound to the router. Dial-out matters for routers behind strict management-plane ACLs that only permit outbound-initiated connections, since dial-out requires no inbound hole in the ACL at all — the router reaches out, just like it would for syslog/SNMP-trap-style push models. The **NSO/YANG service lifecycle** concept means modeling a customer VRF as a declarative "service" (via a YANG service model) that NSO can **create, modify, and delete**, with NSO continuously reconciling the live device configuration against the intended service model (detecting and correcting drift) — this reduces drift compared to manual CLI because every change goes through the same single service definition and reconciliation engine rather than ad hoc per-engineer CLI edits that can diverge between R14/R15/R16 over time.

**Explanation:** In dial-in mode, the collector/controller must be able to open a connection TO the router, which requires an inbound ACL permit on the router's management plane — exactly what Elena says some routers don't allow; dial-out flips the initiation direction so the router (already permitted to make outbound connections per typical management-ACL design) reaches out to the collector, requiring no inbound exception at all, which is precisely why dial-out matters for ACL-restricted routers. NSO's service-lifecycle model (create/modify/delete, backed by continuous reconciliation/"compliance reporting" against actual device state) means a VRF defined once as a YANG service gets pushed and verified identically everywhere it's instantiated, and any manual out-of-band CLI change that drifts from the service's intended state can be detected (and optionally corrected) by NSO's reconciliation — this directly solves the R6-vs-R12 policy-drift problem Daniel raised earlier, because the source of truth becomes the YANG service model instead of whatever each engineer happened to type into each router's CLI.

**Reference:** OpenConfig/gRPC Telemetry dial-in vs. dial-out (gNMI Subscribe vs. gRPC Dial-out "Telemetry" model); Cisco NSO Service Manager / YANG service lifecycle documentation (RFC 7950 YANG, RFC 8040/8341 context for model-driven management).

---

## End of Solutions Document

All 30 items answered, matching the Phase 2 question set distribution: IS-IS (2), OSPF (3), BGP (4), MPLS-TE (2), L3VPN (3), Inter-AS (3), Multicast (2), QoS (2), Security (2), L2VPN (2), Segment Routing (3), SRv6 (1), Automation/Operations (1) = 30 total.
