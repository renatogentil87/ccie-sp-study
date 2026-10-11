# Flashcard Deck (running log)

**Agent-maintained file — Renato does not open this.** It's my working memory for running his morning recall sessions and tracking spaced repetition. He challenges himself verbally; I pull/track cards from here. Keep it updated after each session.

**PARKED (not read yet — do NOT surface in warm-ups until the chapter is read):**
- EVPN cards (L5, and any future EVPN) → park until SPCOR Ch 13 is read
- Any card on a topic he hasn't studied yet

Cards accumulate here with last score + date. The morning warm-up prioritizes low/flagged cards (that are fair game), then mixes in others for breadth. Update scores after each session.

Legend: score /10 · ⚠️ = flagged for review (≤7 or unknown) · 🅿️ = parked (topic not read yet)

---

## MPLS Fundamentals

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| M1 | LDP vs SR label significance (locally vs globally significant) + SR's advantage | 10 | 2026-08-18 |
| M2 | Two-label stack in L3VPN — name both labels + which protocol assigns each (LDP=transport, MP-BGP=VPN) | 10 | 2026-08-18 |
| M3 | PHP — what it does, which router, why beneficial (+ implicit-null=3) | 10 | 2026-08-18 |
| M4 | `mpls ldp label allocate global host-routes` — what/why, what still works | 10 | 2026-08-16 |
| M5 | LDP Session Protection — condition to stay up + benefit (targeted LDP session) | 9 | 2026-08-16 |

## L2VPN / VPLS

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| L1 | VPWS vs VPLS + MEF mapping (E-Line / E-LAN / E-Tree) | 10 | 2026-08-18 |
| L2 | Split-horizon in VPLS — what it is + why it forces a full mesh | 9 | 2026-08-17 |
| L3 | BGP VPLS label block — formula Base+(VE-ID − Offset), one advertisement covers all | 10 | 2026-08-18 |
| L4 | H-VPLS tiers (N-PE/U-PE), spoke vs mesh PW, spoke split-horizon exemption | 10 | 2026-08-18 |
| L5 | Three EVPN advantages over VPLS (control-plane MAC learning, active-active MH, less BUM) | 🅿️ parked (read Ch 13 first) | 2026-08-17 |

## Carrier Ethernet / ERPS / CFM-OAM

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| E1 | G.8032 Idle state — how many links blocked (ONE=RPL), which node (RPL Owner), why | 10 | 2026-08-18 |
| E2 | FAILURE sequence — adjacent nodes block + send R-APS(SF); RPL Owner unblocks RPL | 10 | 2026-08-18 |
| E3 | MEP vs MIP (location, active/passive, CCM) | 10 | 2026-08-17 |
| E4 | CFM MD levels (cust 5-7 / prov 3-4 / oper 0-2) + how missing level localizes fault (narrowest broken level) | 8 ⚠️ | 2026-08-18 |
| E5 | OAM — acronym, 3 questions (up/where/**SLA/performance**), L3 tools (ping/traceroute) | 9 ⚠️ | 2026-08-18 |

---

---

## DEEP CARDS (L3 / CCIE-lab depth — for mastered topics: MPLS, L3VPN, BGP, L2VPN)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| D1 | Egress PE VPNv4 shows in/out 24/nolabel — why nolabel; what appears with a downstream labeled hop (CSC → 3-label); 24 = VPN label from MP-BGP | 7 ⚠️ | 2026-08-18 |
| D2 | Two VPNv4 entries same RT, different RD — compete at VRF import not in RIB; full best-path (IGP metric usual tiebreaker) picks CEF; same RD → RR reflects one to clients → no fast failover | 9 | 2026-08-18 |
| D3 | BGP best-path forensics — best = higher router-id → decided at oldest-path (step 9); confirm via `show ip bgp summary` uptimes; `bgp bestpath compare-routerid` makes deterministic | 10 | 2026-08-18 |
| D4 | RR path hiding — THREE fixes: Add-Path (RFC7911), unique RD per PE, Best-External. (SoO is loop-prevention, NOT this.) All feed BGP PIC for fast failover | 3 ⚠️⚠️ | 2026-08-18 |
| D5 | PW DOWN but LDP up — causes: VC-ID mismatch, AC down, MTU mismatch, encap/PW-type mismatch, broken transport LSP to peer /32 (targeted LDP up ≠ transport LSP up). Lead with `show mpls l2transport vc <id> detail` | 8 | 2026-08-18 |

## Priority review queue (fair-game cards to lead with next warm-up)
1. **TE1** — FRR facility backup: PLR behavior, fast-reroute flag requirement, label stacking (scored 6/10)
2. **D4** — RR path hiding fixes (Add-Path / unique-RD / Best-External) — SoO is a DIFFERENT thing. Big gap.
3. **D1** — CSC / downstream-label case (3-label stack); **D5** — transport-LSP-vs-targeted-LDP distinction
4. **E4** — fault localization = *narrowest broken level*
5. **E5** — 3rd OAM question = **performance / SLA**

*(L5 EVPN PARKED until Ch 13. SR cards PARKED until Ch 15 is read. Surface cards M/L/E mostly 9-10 — rotate for maintenance.)*

---

## MPLS Traffic Engineering (NEW — from Lab 03 + interactive sessions Sep 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| TE1 | FRR scenario: 2 tunnels (60M each), link has 100M RSVP BW. Tunnel2 DOWN why? If link fails, what happens to Tunnel1 (has fast-reroute)? Does Tunnel2 (no fast-reroute) get facility backup protection? | 6 ⚠️⚠️ | 2026-09-09 |
| TE2 | Autoroute announce: why route shows Tunnel as outgoing intf; does VPN traffic auto-use the tunnel; explicit path fails — what sequence happens (re-signal same path-option first, then fallback, make-before-break) | 9 | 2026-09-09 |
| TE3 | Link protection vs Node protection: backup tunnel destination (NHOP vs NNHOP), who is PLR, where is backup tunnel configured (PLR not headend) | — new, not yet tested | — |
| TE4 | Facility backup vs One-to-One: how label stacking shares one bypass for all primaries; `backup-bw` command makes it facility mode | — new, not yet tested | — |
| TE5 | SRLG: what it is (shared physical risk), how configured (`mpls traffic-eng srlg`), `exclude force` vs `exclude preferred`, how it differs from facility backup | — new, not yet tested | — |
| TE6 | Make-before-break: new path signaled before old torn down; Shared Explicit (SE) style prevents double-booking BW | — new, not yet tested | — |
| TE7 | Auto-bandwidth: measures tunnel interface output (not physical link), sampling interval, adjustment threshold, min/max BW | — new, not yet tested | — |
| TE8 | Setup/Hold priority: 0-7 (0=highest), setup X can preempt hold > X (strictly greater), default both = 7 | — new, not yet tested | — |
| TE9 | CSPF vs SPF: CSPF = constrained (considers BW, affinity, TE metric), uses TE database from Type-10 LSA (OSPF) or IS-IS TLVs | — new, not yet tested | — |

## L3VPN Advanced (NEW — from Lab 02 CCIE+ Challenges Sep 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| V1 | SoO: what it prevents (routing loop on dual-homed CE), where configured (inbound on PE-CE), where checked (outbound on remote PE toward same CE) | — from lab discussion, not formally tested | — |
| V2 | Sham-link: why needed (OSPF intra > inter-area, backdoor wins), source/dest = VRF loopbacks advertised via BGP (not OSPF), creates intra-area adjacency across MPLS core | — from lab discussion, not formally tested | — |
| V3 | DN-bit: set by PE on OSPF LSA redistributed from BGP, prevents second PE from re-redistributing back into BGP (loop prevention) | — from lab discussion, not formally tested | — |
| V4 | RT-Constraint: rtfilter unicast on BOTH sides (PE + RR), PE advertises wanted RTs, RR filters outbound VPNv4. Without RR-side config = no filtering | — from lab, tested and working | — |
| V5 | as-override: needed when same-ASN CEs at different sites, PE replaces customer AS with SP AS in path, without it CE rejects (own AS = loop prevention) | — from lab discussion, not formally tested | — |
| V6 | BGP PIC: `bgp additional-paths install` = pre-install backup in CEF, prefix-INDEPENDENT convergence (one pointer swap for all prefixes regardless of count) | — from lab discussion, not formally tested | — |
| V7 | Admin distance: eBGP=20, OSPF=110, iBGP=200. With eBGP PE-CE: VPN path preferred over OSPF backdoor. With OSPF PE-CE: backdoor preferred → sham-link needed | — from lab observation, not formally tested | — |
| V8 | LDP-IGP sync: configure under `router ospf` not interface; OSPF advertises max-metric when LDP down; test by removing `mpls ip` on REMOTE side; session protection prevents it from triggering | — from lab observation, not formally tested | — |

## Segment Routing + SRv6 (from Lab E01 + interactive sessions + SR training Sep 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| SR1 | SR-MPLS PHP troubleshooting: P4 shows `Pop` for PE3's prefix-SID in `show mpls forwarding`. Problem? Why does the label stay the same (17103→17103) across hops but Pop at penultimate? | 8 | 2026-09-16 |
| SR2 | SRv6: locator fc00:0:24::/64 on PE5, VPN route advertised with SID fc00:0:24:40::. What SID type? What does it do? Does remote PE need an SRH? (End.DT4, decap+VRF lookup, no SRH for single SID) | 6 ⚠️ reviewed | 2026-09-26 |
| SR3 | Inter-AS Option C: PE5 (Gold) sees VPN route from CE1 (Emerald) with next-hop 1.1.1.1 but `show cef vrf` says "no route." Root cause? Fix? (BGP-LU needed on ASBRs, NOT IGP redistribution) | 7 ⚠️ reviewed | 2026-09-26 |
| SR4 | SR vs LDP label bindings: 7-router network, how many label bindings for one prefix? LDP = dozens (per-neighbor, locally significant). SR = 1 (global, SRGB+index, same everywhere). Why SR scales better. | ⚠️ new concept | 2026-09-16 |
| SR5 | BGP path manipulation: 3 inter-AS links, prefer Gold transit over direct E-R6↔Gar-R7. Two methods: LOCAL_PREF on E-R6 (preferred — local to your AS) or AS-PATH prepend on Gar-R7 (requires other SP cooperation). LOCAL_PREF beats AS-PATH in best-path algorithm. | 9 | 2026-09-16 |
| SR6 | SR-MPLS prefix-SID: where is it configured? (only Loopback0 under IS-IS, address-family ipv4 unicast, prefix-sid index X). Where is `segment-routing mpls` configured? (process level under IS-IS AF — enables SR on ALL IS-IS interfaces at once). Adj-SIDs = automatic. | — new | 2026-09-26 |
| SR7 | SR label imposition rules: 3 conditions — (1) destination/next-hop matches FEC with prefix-SID, (2) downstream neighbor is SR-enabled, (3) sr-prefer configured OR no LDP label exists. Without sr-prefer and LDP exists → LDP wins by default. | — new | 2026-09-26 |
| SR8 | Protected vs unprotected adj-SID: protected = TI-LFA backup if link fails (rerouted). Unprotected = dropped if link fails. Both auto-allocated per adjacency. Use protected for resilience, unprotected for strict TE where backup path would violate constraints. | — new | 2026-09-26 |
| SR9 | SR over LDP (mapping server): Garnet (SR) needs to reach Emerald (LDP). Emerald has no prefix-SIDs. Fix: mapping server on Gar-R6 creates virtual prefix-SIDs (index 101-106) for Emerald loopbacks, advertised via IS-IS. Reverse direction (LDP→SR) works natively — no mapping needed. Mapping server = temporary bridge during migration. | — new | 2026-09-26 |
| SR10 | LDP→SR migration steps: (1) enable SR alongside LDP (both labels in LFIB), (2) sr-prefer (SR active, LDP backup), (3) remove LDP interface-by-interface, (4) verify no traffic loss at each step. sr-prefer is the key — without it LDP wins when both exist. | — new | 2026-09-26 |

## TI-LFA (from SR training Sep 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| TL1 | TI-LFA basics: uses existing LSDB (no new LSAs/LSPs needed), pre-computes backup by running SPF on modified topology (failed link/node removed), expresses backup as minimal segment list, pre-installs in FIB. Sub-50ms switchover. Zero state on transit routers. | — new | 2026-09-26 |
| TL2 | TI-LFA P-space/Q-space: P-space = nodes reachable from source WITHOUT failed link. Q-space = nodes that can reach destination WITHOUT failed link. PQ-node = in both → single label detour. No PQ-node → 2-3 labels (P-node SID + Q-node SID + destination). Minimum labels, not one per hop. | — new | 2026-09-26 |
| TL3 | Link protection vs node protection: link protection bypasses the LINK (if node dies, traffic still blackholes). Node protection bypasses the entire NODE (covers both link and node failure). Node protection is the SP standard — `fast-reroute per-prefix tiebreaker node-protecting index 100`. Falls back to link protection if node-protecting path can't be computed. | — new | 2026-09-26 |
| TL4 | TI-LFA vs RSVP-TE FRR: RSVP-TE = pre-signal backup tunnel (state on every hop, RSVP PATH/RESV messages, per-tunnel overhead). TI-LFA = local computation only (zero signaling, zero transit state, per-prefix backup). TI-LFA replaces RSVP-TE FRR in SR networks. | — new | 2026-09-26 |
| TL5 | SRLG (Shared Risk Link Group): links sharing physical risk (same fiber duct, same conduit). Configured per interface. TI-LFA can exclude SRLG-member links from backup path computation. `exclude force` = mandatory exclusion, `exclude preferred` = try to exclude but allow if no alternative. | — new | 2026-09-26 |

## SRv6 (from SRv6 deep-dive + lab troubleshooting Sep 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| SV1 | SRv6 fundamentals: pure IPv6 forwarding, NOT MPLS. Packets are IPv6 packets. Core interfaces MUST have IPv6 addresses. Transit routers just do IPv6 route lookup — they don't know SRv6 is happening. No LDP, no RSVP, no MPLS labels. | — new | 2026-09-26 |
| SV2 | SRv6 locator: /64 prefix per router (e.g., fc00:0:21::/64). All SIDs for that router allocated within it. IS-IS advertises the locator. Equivalent of prefix-SID in SR-MPLS but it's an IPv6 address block, not a single label. | — new | 2026-09-26 |
| SV3 | SRv6 SID types: End (reach node), End.X (specific link), End.DT4 (decap into IPv4 VRF), End.DT6 (decap into IPv6 VRF), End.DX4 (deliver to specific CE), End.B6 (binding SID — apply new policy). The SID encodes the ACTION — unlike MPLS where a label is just a number looked up in a table. | — new | 2026-09-26 |
| SV4 | SRH (Segment Routing Header): IPv6 extension header (Routing Header Type 4). Contains Segment List (ordered SIDs) + Segments Left counter. Only needed for MULTIPLE SIDs (TE). Single SID = plain IPv6 packet, no SRH. Transit routers never read SRH — only segment endpoints process it. | — new | 2026-09-26 |
| SV5 | SRv6 encapsulation: router-to-router ping = native IPv6 (no encap). VPN traffic (1 SID) = outer IPv6 header + inner customer packet. VPN + TE (multiple SIDs) = outer IPv6 + SRH + inner customer packet. | — new | 2026-09-26 |
| SV6 | IS-IS for SRv6: needs `address-family ipv6 unicast` + `single-topology` + `segment-routing srv6 locator MAIN` under IS-IS. Single-topology vs multi-topology MUST match on all routers (TLV 236 vs 237 mismatch = IPv6 routes don't install, adjacency still UP). | — new | 2026-09-26 |
| SV7 | SRv6 vs SR-MPLS: SRv6 overhead = 16 bytes per SID (IPv6 address), SR-MPLS = 4 bytes per label. SRv6 = native inter-domain (IPv6 routable), SR-MPLS = needs BGP-LU stitching. SRv6 = needs IPv6 core, SR-MPLS = works on IPv4-only. SRv6 = greenfield/5G, SR-MPLS = brownfield migration. | — new | 2026-09-26 |

## Classic LFA (from SR lab exercises Oct 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| CL1 | Classic LFA: what is it? Per-prefix fast-reroute. Checks each direct neighbor: "is this neighbor's distance to destination STRICTLY LESS than going back through me?" If yes → backup. If no → no backup, give up. Only considers direct neighbors. | — new | 2026-10-01 |
| CL2 | Classic LFA inequality: Distance(N,D) < Distance(N,S) + Distance(S,D). N=neighbor, D=destination, S=source(self). STRICTLY less than — equal fails. Equal means neighbor MIGHT loop back through you. | — new | 2026-10-01 |
| CL3 | Classic LFA vs TI-LFA: Classic LFA hopes the neighbor routes correctly (can't control it). TI-LFA forces the path with segment labels (adj-SID/prefix-SID). Classic LFA coverage ~70-80%. TI-LFA = ~100%. | — new | 2026-10-01 |
| CL4 | Classic LFA coverage gap: R1→R3 primary path to R6. R2 is backup candidate. R2 has ECMP to R6 — one path goes back through R1. LFA inequality: 30 < 10+20 = 30 < 30 = FAILS (equal, not strictly less). R1 has no LFA backup. | — new | 2026-10-01 |
| CL5 | TI-LFA zero-segment (LDP): TI-LFA computation on LDP network. If backup neighbor is naturally loop-free → zero extra labels needed, existing LDP label works. Better coverage than classic LFA (uses post-convergence P/Q-space model). But still can't reach 100% without SR repair segments. | — new | 2026-10-01 |

## Flex-Algo (from SR lab exercises + troubleshooting Oct 2026)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| FA1 | Flex-Algo concept: multiple independent SPF computations on the same LSDB with different optimization criteria. Each algo produces separate forwarding topology + separate prefix-SIDs. Algo 0 = default. Algo 128-255 = user-defined. | — new | 2026-10-01 |
| FA2 | Flex-Algo definition: ONE router advertises the definition (metric-type, constraints) via Router Capability TLV in IS-IS. All others just participate with `flex-algo 128` (no `advertise-definition`). Multiple definers = conflict — highest system-ID wins. | — new | 2026-10-01 |
| FA3 | Flex-Algo affinity: tag interfaces with colors (`affinity flex-algo red`). Define algo constraint (`exclude-any red`). SPF reads Ext Admin Group sub-TLV in TLV 22 and excludes matching links. `affinity-map red bit-position 0` must be IDENTICAL on every router. | — new | 2026-10-01 |
| FA4 | Flex-Algo participation: opt-in per router. Need `flex-algo 128` + `prefix-sid algorithm 128 index X` on Loopback0. No prefix-SID = not in that topology. Traffic routes around non-participating routers. Used to shape each virtual topology by selecting members. | — new | 2026-10-01 |
| FA5 | Flex-Algo silent failure #1: missing `metric-style wide` → IS-IS uses narrow TLVs (2, 128) → no room for prefix-SID or affinity sub-TLVs → Flex-Algo configured but invisible. No error message. | — new | 2026-10-01 |
| FA6 | Flex-Algo silent failure #2: missing `point-to-point` on interfaces → IS-IS creates pseudonodes → pseudonode LSPs can't carry Ext Admin Group (affinity) sub-TLVs → affinity tags exist in config but NOT in LSDB → algo sees no red links, excludes nothing. | — new | 2026-10-01 |
| FA7 | Flex-Algo silent failure #3: multiple routers with `advertise-definition` → definition conflict → highest system-ID wins → if winning definition differs from yours → algo computes differently than expected. `Definition Equal to Local: No` in show output = conflict. | — new | 2026-10-01 |
| FA8 | Flex-Algo TLV dependency chain: metric-style wide (TLV 22/135) → point-to-point (per-link sub-TLVs) → affinity tags (Ext Admin Group) → flex-algo definition (Router Cap TLV) → prefix-sid per algo (TLV 135). Break any link = silent failure. | — new | 2026-10-01 |
| FA9 | Flex-Algo troubleshooting: `show isis flex-algo 128` (check Definition Equal to Local: Yes). `show isis database <router> verbose` (check Ext Admin Group on links). `show mpls forwarding labels <algo-SID>` (compare outgoing interface vs algo 0 SID). If algo 128 path = algo 0 path → affinity not being advertised. | — new | 2026-10-01 |
| FA10 | Flex-Algo + TI-LFA: TI-LFA computes separate backup per algorithm. Algo 128 backup uses algo 128 topology (not default). If algo 128 excludes a link, the TI-LFA backup for algo 128 also avoids that link. | — new | 2026-10-01 |

## Priority review queue (updated 2026-10-01)
1. **D4** — RR path hiding fixes (was 3/10, reviewed — re-test)
2. **SR2** — SRv6 SID types (was 6/10, reviewed — re-test)
3. **SR3** — Inter-AS Option C / BGP-LU (was 7/10, reviewed — re-test)
4. **SR4** — SR vs LDP label count (new)
5. **TE1** — FRR facility backup (6/10)
6. **D1** — CSC 3-label stack (7/10)
7. **SR6-SR10** — SR cards, need testing
8. **TL1-TL5** — TI-LFA cards, need testing
9. **CL1-CL5** — Classic LFA cards, NEW
10. **FA1-FA10** — Flex-Algo cards, NEW
11. **DS1-DS10** — Design Scenario cards, NEW
12. **TS1-TS8** — Troubleshooting Failure cards, NEW

## Design Scenarios (CCIE SP Design Module — multiple choice, scenario-based)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| DS1 | **Scenario:** A Tier-1 SP runs IS-IS + SR-MPLS across 500 routers. A premium voice customer requires sub-50ms failover with zero signaling overhead on transit routers. The SP currently has no RSVP-TE deployed. Which protection mechanism meets all requirements? (A) Deploy RSVP-TE FRR with facility backup tunnels on all transit routers (B) Enable Classic LFA per-prefix on all IS-IS interfaces (C) Enable TI-LFA with node-protecting preference on all IS-IS interfaces (D) Configure static backup routes on each PE. **Answer: C** — TI-LFA provides sub-50ms, zero transit state (segment list pre-computed locally), node protection covers link+node failures. RSVP-TE FRR is sub-50ms but adds per-tunnel state on every hop. Classic LFA has coverage gaps (~70-80%). Static routes don't converge in sub-50ms. | — new | 2026-10-01 |
| DS2 | **Scenario:** Three SPs need to deliver L3VPN for a multinational customer spanning all three ASes. The design must minimize VPN state on ASBRs and maintain an end-to-end labeled path so BGP PIC can provide fast VPN failover. Which inter-AS architecture meets these requirements? (A) Inter-AS Option A with back-to-back VRFs on ASBRs (B) Inter-AS Option B with VPNv4 eBGP between ASBRs (C) Inter-AS Option C with BGP-LU between ASBRs and multihop VPNv4 between RRs (D) Static MPLS cross-connects on ASBRs. **Answer: C** — Option C puts only PE loopbacks on ASBRs (~20 routes), VPN routes flow RR-to-RR bypassing ASBRs entirely, end-to-end labeled path enables BGP PIC. Option B puts all VPN routes on ASBRs. Option A requires per-VRF config on ASBRs. | — new | 2026-10-01 |
| DS3 | **Scenario:** A 5G mobile operator needs to deliver three transport slices over a single physical IS-IS + SR-MPLS network: (1) low-latency for URLLC, (2) high-bandwidth for eMBB, (3) budget paths for mMTC. The operator wants the IGP to compute slice-specific paths automatically without deploying a centralized controller or RSVP-TE tunnels. Which technology best meets this requirement? (A) RSVP-TE with 3 tunnel types per destination (B) Flex-Algo with 3 algorithm definitions (C) BGP communities with route-maps on each PE (D) Multiple IS-IS instances (one per slice). **Answer: B** — Flex-Algo runs multiple SPFs on the same LSDB with different constraints. 3 algo definitions, IGP computes separate topologies, per-algo prefix-SIDs. No controller, no tunnels, scales with IGP. Multiple IS-IS instances add operational complexity. BGP communities don't affect IGP path selection. | — new | 2026-10-01 |
| DS4 | **Scenario:** An SP is migrating a 200-router IS-IS core from LDP to SR-MPLS. The migration must be zero-downtime. Some routers are end-of-life and cannot run SR — they will remain LDP-only until replaced. SR routers need labeled paths to LDP-only destinations. What is the correct migration approach? (A) Remove LDP from all routers simultaneously and enable SR (B) Enable SR alongside LDP, configure sr-prefer, then remove LDP gradually from SR-capable routers. Deploy a mapping server for SR-to-LDP reachability (C) Redistribute LDP labels into SR via BGP (D) Replace all routers before starting the migration. **Answer: B** — Coexistence: SR + LDP run simultaneously, sr-prefer shifts traffic to SR, LDP removed per-interface. Mapping server provides virtual prefix-SIDs for LDP-only routers so SR routers can build labeled paths. Mapping server is temporary — removed after full migration. | — new | 2026-10-01 |
| DS5 | **Scenario:** A customer has two sites connected to the same SP. Both CE routers use eBGP with AS 65012 toward their respective PEs. CE2 rejects VPN routes from CE1 because it sees its own AS (65012) in the AS-PATH. The SP needs to fix this without changing the CE configuration. Which solution is correct? (A) Configure `allowas-in` on both CEs (B) Configure `as-override` on both PEs (C) Configure `remove-private-as` on both PEs (D) Configure `local-as 65099` on both PEs. **Answer: B** — `as-override` on the PE replaces the customer AS in the AS-PATH with the SP's AS before advertising to the remote CE. CE2 no longer sees its own AS and accepts the route. `allowas-in` works but requires CE configuration (SP doesn't control CE). `remove-private-as` only removes private ASNs, not customer ASNs. | — new | 2026-10-01 |
| DS6 | **Scenario:** Two OSPF CE sites are connected via MPLS L3VPN and also have a backdoor physical link between them. Traffic prefers the backdoor link because OSPF intra-area routes (O) have lower admin distance than OSPF inter-area routes (O IA) received via the MPLS core. The SP needs MPLS to be preferred for normal traffic while keeping the backdoor as backup. What is the correct solution? (A) Increase OSPF cost on the backdoor link to 65535 (B) Configure a sham-link between the two PEs (C) Change BGP admin distance to 90 (D) Configure route-maps to filter OSPF routes from BGP. **Answer: B** — Sham-link creates an intra-area OSPF adjacency across the MPLS core using VRF loopbacks advertised via BGP. Routes via MPLS become O (intra-area) instead of O IA, and can now compete with the backdoor on OSPF cost. Increasing backdoor cost works but changes the customer's OSPF design. | — new | 2026-10-01 |
| DS7 | **Scenario:** A Tier-1 backbone provider wants to sell MPLS transit to a regional SP. The regional SP runs its own MPLS network, its own BGP, and its own L3VPN customers. The backbone provider must transport the regional SP's labeled traffic transparently — the backbone must NOT have visibility into the regional SP's customer VPN routes. Which architecture is correct? (A) Inter-AS Option C with multihop VPNv4 between RRs (B) Inter-AS Option A with shared VRFs on ASBRs (C) Carrier Supporting Carrier (CSC) with eBGP labeled-unicast on PE-CE links (D) Regular L3VPN with the regional SP as a customer. **Answer: C** — CSC. The backbone PE-CE link runs eBGP labeled-unicast (not regular IP). The backbone carries the regional SP's MPLS labels transparently in a 3-label stack. The backbone sees only PE loopbacks of the regional SP — zero visibility into customer VPNs. Option C would expose VPN routes to the backbone's RRs. | — new | 2026-10-01 |
| DS8 | **Scenario:** An SP core runs IS-IS + SR-MPLS. The SP wants to differentiate premium customers (low-latency path) from standard customers (default IGP path) without deploying a PCE controller, without RSVP-TE tunnels, and without explicit per-destination SR-TE segment lists. The solution must scale to thousands of VPN prefixes. What technology combination achieves this? (A) RSVP-TE auto-tunnel with bandwidth constraints (B) Flex-Algo with delay metric + BGP color + SR-TE ODN (C) Static routes with PBR on each PE (D) QoS priority queuing with DSCP marking. **Answer: B** — Flex-Algo 128 (delay metric) computes the low-latency topology automatically. BGP color on premium VPN routes triggers ODN to auto-create SR-TE policies using algo 128 prefix-SIDs. Scales to thousands of prefixes — one algo definition serves all. QoS handles congestion, not path selection. | — new | 2026-10-01 |
| DS9 | **Scenario:** A data center customer requires Layer 2 connectivity (same VLAN) between two sites, each dual-homed to two PE routers. The customer wants all-active forwarding with both PEs simultaneously forwarding traffic for load sharing. If one PE fails, the other must take over with minimal packet loss and no MAC relearning storms. Which technology meets all requirements? (A) VPLS with H-VPLS (B) AToM pseudowire with redundancy (C) EVPN with all-active multi-homing (D) Q-in-Q with MC-LAG. **Answer: C** — EVPN all-active MH: both PEs forward unicast simultaneously via aliasing (Type 1 per-EVI). DF election prevents BUM duplication. ESI label provides split-horizon. Mass withdrawal (Type 1 per-ES) handles PE failure with one BGP update — no MAC relearning storm. VPLS can't do all-active (split-horizon blocks it). MC-LAG requires inter-chassis protocol, doesn't scale across WAN. | — new | 2026-10-01 |
| DS10 | **Scenario:** An SP engineer deploys Segment Routing on an existing IS-IS network. After committing the configuration on all routers (`segment-routing mpls` under IS-IS AF, `prefix-sid index` on Loopback0), some routers show prefix-SIDs in the label table but others show only their local SID with no remote SIDs. IS-IS adjacencies are all UP and IPv4 routes are exchanged normally. What are the THREE most likely causes? (A) Missing `metric-style wide` on some routers (B) SRGB range mismatch (C) Loopback configured with /24 mask instead of /32 (D) Missing `point-to-point` on IS-IS interfaces (E) BGP not configured. **Answer: A, C, D** — (A) Without wide metrics, IS-IS uses narrow TLVs that can't carry prefix-SID sub-TLVs. (C) SR node-SIDs are designed for /32 host routes; /24 causes remote installation issues. (D) Without point-to-point, pseudonodes may interfere with SR TLV propagation. BGP is irrelevant — SR is distributed by IS-IS. SRGB mismatch causes label conflicts, not missing SIDs. | — new | 2026-10-01 |

## Troubleshooting Failures (scenario-based — what's wrong and how to fix it)

| # | Question | Last score | Last seen |
|---|----------|-----------|-----------|
| TS1 | BGP VPNv4 session Established. PE-CE eBGP session UP. But PE shows 0 prefixes received with `!` in summary output. What's wrong? (IOS-XR requires route-policy on eBGP neighbors. No policy = silently drops all routes. Fix: route-policy PASS-ALL in/out) | — new | 2026-10-01 |
| TS2 | IS-IS adjacency UP between R1 and R3. IPv4 routes exchanged fine. IPv6 routes NOT installing on R1. R3's LSP shows `MT (IPv6 Unicast) IPv6 fc00::23/128`. R1's LSP shows `IPv6 fc00::21/128` (no MT prefix). What's wrong? (Single-topology vs multi-topology mismatch. R1=single, R3=multi. Different TLVs: TLV 236 vs 237. Must match on ALL routers.) | — new | 2026-10-01 |
| TS3 | Segment Routing configured on all routers. `show isis segment-routing label table` shows SIDs on some routers but not others. Config looks identical. What's the first thing to check? (metric-style wide. Without it, IS-IS uses narrow TLVs (2, 128) that can't carry prefix-SID sub-TLVs. SR config accepted but never advertised.) | — new | 2026-10-01 |
| TS4 | Flex-Algo 128 configured with `exclude-any red`. Links tagged red. But algo 128 traffic still uses the red-tagged links. `show isis database verbose` shows NO Ext Admin Group on those links. What's wrong? (Interfaces in broadcast mode (no point-to-point). Pseudonode LSPs can't carry affinity sub-TLVs. Fix: point-to-point on all core interfaces.) | — new | 2026-10-01 |
| TS5 | `show isis route` shows a prefix with `<infinity>` metric. Adjacency is UP. LSP is in the database. Why? (3 possible causes: (1) overload-bit set on intermediate router (check OL flag in database), (2) broken link in the SPF chain, (3) metric mismatch — one side has IS-IS enabled on the link, other side doesn't.) | — new | 2026-10-01 |
| TS6 | Classic LFA shows "No FRR backup" for a prefix. R1 primary path to R6 is via R3. R2 is the only other neighbor. R2 has two ECMP paths to R6 — one goes back through R1. Why no backup? (LFA inequality fails: Distance(R2,R6) = 30, Distance(R2,R1) + Distance(R1,R6) = 10+20 = 30. 30 < 30 = FALSE. Equal doesn't pass. Fix: enable TI-LFA — it forces the path with segment labels.) | — new | 2026-10-01 |
| TS7 | Inter-AS Option C configured. PE in Gold sees VPN route from Emerald with next-hop 1.1.1.1. `show cef vrf CUST_A` says "no route" for 1.1.1.1. VPN route present in BGP but not installed. What's wrong? (No BGP-LU between ASBRs. PE can't resolve the remote PE's loopback. Fix: configure eBGP labeled-unicast between ASBRs + advertise PE loopbacks with labels. NOT IGP redistribution.) | — new | 2026-10-01 |
| TS8 | Mapping server configured on R3 for R1's loopback (index 101). R6 shows 16101 in label table. But R4 (also in the SR domain) doesn't show 16101. R4 has a different mapping server with index 111 for the same prefix. What's wrong? (Conflicting mapping servers. Two routers advertising different SIDs for the same prefix. R4 picks closer source. Fix: ALL mapping servers must advertise IDENTICAL indexes. Inconsistent = inconsistent labels = forwarding breaks.) | — new | 2026-10-01 |
