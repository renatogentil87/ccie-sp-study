# CCIE SP Study Context — Full AI Reference

**Who:** Renato (rrdog), Senior TAM at AWS, studying for CCNP SP → CCIE SP.
**Updated:** October 2026
**Platform:** GNS3 on EC2 m8i.24xlarge running IOS-XRv 9000 + Arista vEOS CEs

---

## STUDY METHOD

### Flashcard Sessions
- 5 cards per session. ONE card at a time. Wait for my answer before showing next.
- Score /10. Correct mistakes. Add context I missed.
- Cards ≤7 flagged ⚠️ for review — bring back in future sessions.
- Only test topics I've studied (see list below).
- Mix troubleshooting, design scenarios, and concept recall.
- Use my actual lab topology when possible.
- Three card types: **Concept** (explain how something works), **Troubleshooting** (given symptoms, find root cause), **Design** (given requirements, pick best technology).

---

## WHAT I'VE STUDIED (fair game for flashcards)

### Mastered (scored 8-10)
- MPLS fundamentals (LDP, label switching, PHP, LFIB)
- L3VPN (VRF, RD, RT, MP-BGP VPNv4, PE-CE routing, VRF isolation, RD vs RT purpose)
- L2VPN (VPWS, VPLS, H-VPLS, pseudowires)
- MPLS-TE (RSVP tunnels, FRR, autoroute, CSPF, make-before-break)
- BGP (best-path, communities, LOCAL_PREF, AS-PATH, MED, RR design)
- IS-IS (L2-only, metric-style wide, adjacency, LSDB, SPF, pseudonodes, DIS, NET-ID derivation)
- Carrier Ethernet (G.8032 ERPS, CFM/OAM, MEP/MIP)
- Inter-AS Options A/B/C (concepts + Option C configured awareness)
- CSC (Carrier Supporting Carrier, 3-label stack)

### Currently Studying (test me, expect some gaps)
- **Segment Routing SR-MPLS:** prefix-SID, adj-SID, SRGB, protected/unprotected adj-SID, globally significant labels
- **Classic LFA:** inequality check, coverage gaps, why equal fails, comparison with TI-LFA
- **TI-LFA:** link/node/SRLG protection, P-space/Q-space, tiebreaker preference, zero-segment on LDP
- **Flex-Algo:** multiple SPFs, affinity colors, exclude-any, single definer, TLV dependency chain, silent failures
- **SR↔LDP coexistence:** mapping server, sr-prefer, migration steps, boundary router config
- **IS-IS TLVs:** narrow (2/128) vs wide (22/135), extended sub-TLVs (affinity, delay, prefix-SID, adj-SID)
- **IOS-XR syntax:** commit model, route-policy on eBGP, VRF under BGP, OSPF area-based, point-to-point under IS-IS

### NOT Yet Studied (DO NOT test)
- SR-TE policies (explicit/dynamic, PCE-initiated)
- SRv6 (locators, SID types, SRH) — know basics but not ready for testing
- EVPN deep (route types 1-5, IRB, MAC mobility)
- QoS on IOS-XR
- Multicast / mVPN profiles
- NSO / NETCONF / automation
- 6PE / 6VPE
- BNG / Access
- Security (RPKI, Flowspec, RTBH, LPTS)

---

## MY LAB TOPOLOGIES

### Main Topology (3 ISPs — for E01-E24 exercises)

**Emerald AS 65100 (IS-IS + LDP):**

| Hostname | Role | Loopback |
|----------|------|----------|
| E-R1 | PE | 1.1.1.1 |
| E-R2 | PE | 2.2.2.2 |
| E-R3 | P | 3.3.3.3 |
| E-R4 | P | 4.4.4.4 |
| E-R5 | P + RR + PCE | 5.5.5.5 |
| E-R6 | ASBR | 6.6.6.6 |

**Gold AS 65300 (IS-IS + SRv6, transit):**

| Hostname | Role | Loopback |
|----------|------|----------|
| G-R1 | PE | 21.21.21.21 |
| G-R2 | PE | 22.22.22.22 |
| G-R3 | P + RR + PCE | 23.23.23.23 |
| G-R4 | ASBR | 24.24.24.24 |
| G-R5 | ASBR | 25.25.25.25 |

**Garnet AS 65200 (IS-IS + SR-MPLS):**

| Hostname | Role | Loopback |
|----------|------|----------|
| Gar-R1 | PE | 11.11.11.11 |
| Gar-R2 | PE | 12.12.12.12 |
| Gar-R3 | P | 13.13.13.13 |
| Gar-R4 | P | 14.14.14.14 |
| Gar-R5 | P | 15.15.15.15 |
| Gar-R6 | PCE + RR | 16.16.16.16 |
| Gar-R7 | ASBR | 17.17.17.17 |

**Customers:** A(65012)=Emerald+Gold, B(65013)=Gold+Garnet, C(EVPN)=Gold+Garnet, CE3(OSPF Emerald), CE6(OSPF Garnet)

### SR Lab Topology (for SR-EX01 to SR-EX08)

| Router | Loopback | Role |
|--------|----------|------|
| R1 | 172.16.1.1 | PE (CE1) |
| R2 | 172.16.2.2 | PE (CE1 dual-homed) |
| R3 | 172.16.3.3 | P (central hub) |
| R4 | 172.16.4.4 | P |
| R5 | 172.16.5.5 | P |
| R6 | 172.16.6.6 | PE (CE2) |

Single AS 65100. IS-IS CORE. Prefix-SIDs: R1=1, R2=2, R3=3, R4=4, R5=5, R6=6.

**SR Exercise Suite (8 exercises, 126 tasks):**
- SR-EX01: Foundation + Classic LFA (9 tasks)
- SR-EX02: TI-LFA link/node/SRLG (17 tasks)
- SR-EX03: SR↔LDP coexistence + mapping server (14 tasks)
- SR-EX04: Flex-Algo affinity/exclude (12 tasks)
- SR-EX05: L3VPN over SR (13 tasks)
- SR-EX06: Dual IGP OSPF↔IS-IS boundary (18 tasks)
- SR-EX07: SR-TE explicit/dynamic/PCE/ODN (19 tasks)
- SR-EX08: SRv6 underlay/locators/VPN/SRv6-TE (24 tasks)

---

## COMPLETE FLASHCARD DECK (92 cards)

### MPLS Fundamentals
- **M1:** LDP vs SR label significance (locally vs globally significant) → **10/10**
- **M2:** Two-label stack in L3VPN — transport (LDP) + VPN (MP-BGP) → **10/10**
- **M3:** PHP — what, which router, why (implicit-null=3) → **10/10**
- **M4:** `mpls ldp label allocate global host-routes` → **10/10**
- **M5:** LDP Session Protection → **9/10**

### L2VPN / VPLS
- **L1:** VPWS vs VPLS + MEF (E-Line/E-LAN/E-Tree) → **10/10**
- **L2:** Split-horizon in VPLS → **9/10**
- **L3:** BGP VPLS label block formula → **10/10**
- **L4:** H-VPLS tiers (N-PE/U-PE) → **10/10**

### Carrier Ethernet / OAM
- **E1:** G.8032 Idle state → **10/10**
- **E2:** G.8032 failure sequence → **10/10**
- **E3:** MEP vs MIP → **10/10**
- **E4:** CFM MD levels → **8/10 ⚠️**
- **E5:** OAM 3 questions → **9/10**

### Deep Cards
- **D1:** CSC 3-label stack → **7/10 ⚠️**
- **D2:** Two VPNv4 entries same RT different RD → **9/10**
- **D3:** BGP best-path router-id tiebreaker → **10/10**
- **D4:** RR path hiding — Add-Path, unique RD, Best-External → **3/10 ⚠️⚠️**
- **D5:** PW DOWN but LDP up — 5 causes → **8/10**

### MPLS-TE
- **TE1:** FRR scenario → **6/10 ⚠️⚠️**
- **TE2:** Autoroute announce → **9/10**
- **TE3-TE9:** Not yet tested

### L3VPN Advanced
- **V1-V8:** SoO, sham-link, DN-bit, RT-Constraint, as-override, PIC, admin distance, LDP-IGP sync — not formally tested

### Segment Routing SR-MPLS
- **SR1:** SR-MPLS PHP — same label across hops, Pop at penultimate → **8/10**
- **SR2:** SRv6 SID types — End.DT4, locator vs prefix-SID → **6/10 ⚠️ (DO NOT TEST — SRv6 parked)**
- **SR3:** Inter-AS Option C — BGP-LU not IGP redistribution → **7/10 ⚠️**
- **SR4:** SR vs LDP label count — 1 global vs dozens local → **new ⚠️**
- **SR5:** BGP path manipulation — LOCAL_PREF vs AS-PATH prepend → **9/10**
- **SR6:** prefix-SID config: Loopback0 only. `segment-routing mpls` at process level enables ALL interfaces. Adj-SIDs automatic. → **new**
- **SR7:** SR label imposition 3 rules: FEC match + downstream SR-enabled + sr-prefer or no LDP label → **new**
- **SR8:** Protected vs unprotected adj-SID → **new**
- **SR9:** Mapping server: virtual prefix-SIDs for LDP destinations. Reverse (LDP→SR) works natively. Temporary bridge. → **new**
- **SR10:** LDP→SR migration: enable both → sr-prefer → remove LDP gradually → **new**

### Classic LFA
- **CL1:** What classic LFA is: checks each neighbor with inequality. Only considers direct neighbors. → **new**
- **CL2:** LFA inequality: Distance(N,D) < Distance(N,S) + Distance(S,D). STRICTLY less than. Equal fails. → **new**
- **CL3:** Classic LFA vs TI-LFA: LFA hopes neighbor routes correctly. TI-LFA forces path with labels. LFA ~70-80% coverage. TI-LFA ~100%. → **new**
- **CL4:** Coverage gap scenario: R2 has ECMP to R6, one path loops through R1. 30 < 30 = FAILS. No backup. → **new**
- **CL5:** TI-LFA zero-segment on LDP: naturally loop-free neighbor → zero extra labels. Better than classic LFA but not 100%. → **new**

### TI-LFA
- **TL1:** Basics: reuses LSDB, pre-computes backup via modified SPF, minimal segment list, pre-installed in FIB, sub-50ms → **new**
- **TL2:** P-space/Q-space: PQ-node → 1 label. No PQ-node → 2-3 labels. Minimum labels, not one per hop. → **new**
- **TL3:** Link vs node protection: node = SP standard. `tiebreaker node-protecting index 100`. Falls back to link if needed. → **new**
- **TL4:** TI-LFA vs RSVP-TE FRR: TI-LFA = zero signaling, zero transit state, per-prefix. Replaces RSVP-TE FRR. → **new**
- **TL5:** SRLG: same value on shared-risk links (both ends). `tiebreaker srlg-disjoint`. Backup avoids same-duct links. → **new**

### Flex-Algo
- **FA1:** Concept: multiple SPFs on same LSDB with different criteria. Algo 0 = default. 128-255 = user-defined. → **new**
- **FA2:** Single definer: ONE router with `advertise-definition`. Others just `flex-algo 128`. Multiple definers = conflict. → **new**
- **FA3:** Affinity: tag interfaces with colors. `exclude-any red`. `affinity-map` must be IDENTICAL on every router. → **new**
- **FA4:** Participation: opt-in. Need `flex-algo 128` + `prefix-sid algorithm 128 index X`. No SID = not in topology. → **new**
- **FA5:** Silent failure: missing `metric-style wide` → narrow TLVs → no room for SR/affinity. → **new**
- **FA6:** Silent failure: missing `point-to-point` → pseudonodes → can't carry affinity sub-TLVs. → **new**
- **FA7:** Silent failure: multiple definers → `Definition Equal to Local: No` → inconsistent computation. → **new**
- **FA8:** TLV dependency chain: wide → p2p → affinity → definition → prefix-sid per algo. → **new**
- **FA9:** Troubleshooting: `show isis flex-algo 128`, `show isis database verbose` (check Ext Admin Group), compare LFIB algo 0 vs 128. → **new**
- **FA10:** Flex-Algo + TI-LFA: backup computed per-algorithm using that algo's topology. → **new**

### Design Scenarios (CCIE SP Design Module — multiple choice, exam format)
- **DS1:** Sub-50ms failover, zero transit state → (A) RSVP-TE FRR (B) Classic LFA (C) TI-LFA node-protecting (D) static routes. **Answer: C** → **new**
- **DS2:** 3 ASes, minimal ASBR state, end-to-end labeled → (A) Option A (B) Option B (C) Option C (D) static MPLS. **Answer: C** → **new**
- **DS3:** 5G three slices, no controller, no RSVP → (A) RSVP-TE (B) Flex-Algo (C) BGP communities (D) multiple IS-IS. **Answer: B** → **new**
- **DS4:** LDP→SR zero downtime, some routers can't run SR → (A) big-bang (B) coexist+sr-prefer+mapping server (C) redistribute (D) replace hardware first. **Answer: B** → **new**
- **DS5:** Same-AS CEs, eBGP PE-CE, CE rejects own AS → (A) allowas-in on CE (B) as-override on PE (C) remove-private-as (D) local-as. **Answer: B** → **new**
- **DS6:** OSPF backdoor beats MPLS VPN path → (A) increase cost (B) sham-link (C) change AD (D) static routes. **Answer: B** → **new**
- **DS7:** Transparent MPLS transit for another SP, backbone blind to VPNs → (A) Option C (B) Option A (C) CSC (D) regular L3VPN. **Answer: C** → **new**
- **DS8:** Premium vs standard, no controller, no RSVP, thousands of prefixes → (A) RSVP-TE (B) Flex-Algo+color+ODN (C) PBR (D) QoS only. **Answer: B** → **new**
- **DS9:** L2 all-active multi-homing, no MAC storms → (A) VPLS H-VPLS (B) AToM (C) EVPN all-active MH (D) Q-in-Q+MC-LAG. **Answer: C** → **new**
- **DS10:** SR deployed, SIDs missing on some routers, adjacencies UP — pick THREE causes → (A) metric-style wide (B) SRGB mismatch (C) /24 loopback (D) point-to-point (E) BGP missing. **Answer: A,C,D** → **new**

### Latest Flashcard Session Scores (Oct 1, 2026)
- LDP→SR migration (design): **8/10**
- Flex-Algo affinity not working (troubleshooting): **10/10** 🔥
- Classic LFA inequality (concept): **7/10 ⚠️** — knows concept, needs the formula
- SR prefix-SID missing remotely (troubleshooting): **9/10**
- EVPN vs VPLS all-active MH (design): **9/10**

### Troubleshooting Failures
- **TS1:** eBGP 0 prefixes with `!` → missing route-policy (IOS-XR requirement). → **new**
- **TS2:** IS-IS adjacency UP, IPv6 routes missing → single/multi-topology mismatch (TLV 236 vs 237). → **new**
- **TS3:** SR configured, SIDs missing on some routers → missing `metric-style wide`. → **new**
- **TS4:** Flex-Algo affinity not working → missing `point-to-point` (pseudonodes can't carry Ext Admin Group). → **new**
- **TS5:** IS-IS route shows `<infinity>` → overload-bit, broken link chain, or metric mismatch. → **new**
- **TS6:** Classic LFA "No FRR backup" → inequality fails (equal, not strictly less). Fix: TI-LFA. → **new**
- **TS7:** Option C VPN route present but "no route" for next-hop → missing BGP-LU on ASBRs. → **new**
- **TS8:** Mapping server conflict → two servers advertising different indexes for same prefix → inconsistent labels. → **new**

### Priority Review Queue
1. **D4** — RR path hiding (3/10)
2. **SR3** — Option C / BGP-LU (7/10)
3. **SR4** — SR vs LDP label count (new)
4. **TE1** — FRR facility backup (6/10)
5. **D1** — CSC 3-label stack (7/10)
6. **SR6-SR10** — SR cards (new)
7. **TL1-TL5** — TI-LFA cards (new)
8. **CL1-CL5** — Classic LFA cards (new)
9. **FA1-FA10** — Flex-Algo cards (new)
10. **DS1-DS10** — Design Scenarios (new)
11. **TS1-TS8** — Troubleshooting (new)

---

## KEY CONCEPTS

### IOS-XR Gotchas
- eBGP requires route-policy (no policy = 0 routes, `!` in summary)
- No `activate` — `address-family` under neighbor = activation
- VRF under BGP is its own block (`vrf CUST_A`)
- OSPF: no `network` command, interfaces under `area 0` block
- IS-IS: `point-to-point` not `network point-to-point`
- `address-family vpnv4 unicast` at process level BEFORE VRF AFs work
- `show bgp` not `show ip bgp`
- Arista CEs: `no switchport` before IP, `zerotouch cancel` before save

### IS-IS
- NET-ID: pad octets to 3 digits, concatenate, split by 4
- Single vs multi-topology: MUST match all routers. TLV 236 vs 237.
- Pseudonodes (.07, .09): broadcast mode. Fix: `point-to-point`
- Infinity metric: path in LSDB but SPF can't compute. Check OL-bit.
- DIS (like OSPF DR): no BDR, preemption allowed, creates pseudonode LSP
- `metric-style wide`: enables TLV 22/135 with sub-TLVs for SR/TE/Flex-Algo

### Segment Routing
- Prefix-SID: global, SRGB+index. Same label everywhere.
- Adj-SID: local, auto-allocated. Protected + unprotected per adjacency.
- `segment-routing mpls` under IS-IS AF = enables all interfaces at once
- sr-prefer: SR wins over LDP when both exist
- Mapping server: virtual SIDs for LDP destinations. Temporary.

### Classic LFA vs TI-LFA
- Classic: inequality check per neighbor. Equal fails. ~70-80% coverage.
- TI-LFA: post-convergence SPF + segment labels to force path. ~100% coverage.
- Classic hopes. TI-LFA forces.

### TI-LFA
- Pre-computes backup from modified SPF (failure removed). Minimal segment list.
- Tiebreaker: node-protecting (100) > srlg-disjoint (200) > link-protecting (default)
- SRLG: same value on shared-risk links, both ends. Backup avoids SRLG members.
- Only ONE backup installed per prefix (best available per tiebreaker).

### Flex-Algo
- Multiple SPFs per algo with different metrics/constraints
- ONE definer (`advertise-definition`), all others just participate
- Affinity-map must be identical on every router
- Three silent failures: missing wide metrics, missing point-to-point, multiple definers
- TLV chain: wide → p2p → affinity → definition → per-algo prefix-SID
- **Flex-Algo alone doesn't steer traffic** — it creates alternate topology + SIDs
- Traffic steering requires THREE layers: (1) Flex-Algo builds topology, (2) BGP color tags VPN routes at egress PE, (3) SR-TE policy on ingress PE matches color → pushes algo 128 SID instead of algo 0 SID
- Without BGP color + SR-TE policy, algo 128 labels exist in LFIB but no traffic uses them

### Inter-AS
- Option A: VRF on ASBRs, per-VPN sessions. Simple, doesn't scale.
- Option B: VPNv4 eBGP between ASBRs, next-hop-self. Medium scale.
- Option C: BGP-LU + multihop VPNv4 RR-RR. End-to-end labeled. Best scale.
- CSC: backbone carries customer SP's MPLS transparently. 3-label stack. PE-CE = labeled-unicast.

### VPN Design
- RD = uniqueness (per-SP). RT = import/export control (per-customer, same across SPs).
- VRF isolation: CE can't reach SP core. By design.
- OSPF PE-CE: OSPF→BGP always. BGP→OSPF with prefix filter only when needed (E06 Sec 7).
- eBGP PE-CE: as-override for same-ASN CEs. SoO for dual-homed loop prevention.
