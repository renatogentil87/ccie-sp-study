# WB04 — BGP (iBGP, Route Reflectors, eBGP Inter-AS, Path Manipulation, Advanced)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Prerequisite:** WB00–WB03 complete — IP addressing, IGP (IS-IS in AS65100/65300, OSPF in AS65200), and LDP operational in all three ASes. All loopbacks reachable within each AS.
**Format:** Tasks only. No configurations provided — you build the config to meet each task's intent.
**Reference:** See `00_topology_reference.md` for addressing, link map, loopbacks, and console ports.

> **Snapshot before you start:** `WB04-00-start` (confirms WB03 IGP+LDP baseline is clean).

---

## Section 1 — iBGP + Route Reflectors [17 points]

Goal: Build the iBGP control plane in each AS. RRs reflect VPNv4 (and IPv4) so PEs/ASBRs do not need a full mesh. All iBGP sessions use loopback0 as update-source.

- [ ] **Task 1.1 — AS65100 RR cluster. [3 points]** Configure R7 and R8 as route reflectors for AS65100. Peer R7 and R8 to each other (RR-to-RR, non-client). Make R1, R2, R5, R6 route-reflector-clients of both R7 and R8. Use `update-source Loopback0` on every session. Activate both the IPv4 unicast and VPNv4 address families (VPNv4 is needed for WB05).
- [ ] **Task 1.2 — AS65100 next-hop handling. [3 points]** Decide where `next-hop-self` is required. The ASBRs (R5, R6) will learn eBGP routes in Section 2 — ensure the eBGP next-hop is resolvable by iBGP speakers. Document whether you use `next-hop-self` on the ASBRs toward the RRs or rely on redistributing the inter-AS link into the IGP. State your choice and why.
- [ ] **Task 1.3 — AS65200 RR. [3 points]** Configure R19 as the single route reflector for AS65200. Make R12 (ASBR), R14, R15, R16 (PEs) route-reflector-clients. Activate IPv4 unicast and VPNv4. Use loopback0 as update-source. Note that R11/R13 are P routers — decide whether they need BGP at all (justify a BGP-free core vs. including them).
- [ ] **Task 1.4 — AS65300 RR. [3 points]** Configure R30 as the route reflector for AS65300. Make R20, R21 (ASBR/PE), R22, R23 (PE) clients. Activate IPv4 unicast and VPNv4. loopback0 update-source.
- [ ] **Task 1.5 — Cluster-ID & loop prevention. [5 points]** On AS65100 where you have two RRs (R7, R8), confirm the default behavior with distinct cluster-IDs vs. a shared cluster-ID. Explain the ORIGINATOR_ID and CLUSTER_LIST protections and verify them with `show bgp vpnv4 unicast all <prefix>`.
- [ ] **Task 1.6 — Verify iBGP. [0 points — verification]** Confirm all iBGP sessions reach Established in every AS (`show bgp ipvpnv4 unicast all summary`, `show ip bgp summary`). No VPN prefixes exist yet — the point here is a clean, fully-adjacent control plane.

---

## Section 2 — eBGP Inter-AS [13 points]

Goal: Bring up eBGP between ASBRs over the inter-AS links. IPv4 unicast only for now (VPNv4 inter-AS is deferred to WB09). Peer on directly-connected interface addresses (not loopbacks) unless you deliberately configure multihop.

- [ ] **Task 2.1 — R5 ↔ R11 (AS65100 ↔ AS65200). [2 points]** eBGP over `10.5.11.0/24`. IPv4 unicast AF. Peer on connected addresses.
- [ ] **Task 2.2 — R6 ↔ R12 (AS65100 ↔ AS65200). [2 points]** eBGP over `10.6.12.0/24`. IPv4 unicast AF. This is the second, redundant 65100↔65200 link.
- [ ] **Task 2.3 — R6 ↔ R20 (AS65100 ↔ AS65300). [2 points]** eBGP over `10.6.20.0/24`. IPv4 unicast AF.
- [ ] **Task 2.4 — R12 ↔ R21 (AS65200 ↔ AS65300). [2 points]** eBGP over `20.12.21.0/24`. IPv4 unicast AF.
- [ ] **Task 2.5 — Advertise loopbacks. [2 points]** Advertise each AS's PE/RR loopback prefixes (150.x.y.z/32) into eBGP so inter-AS reachability for BGP next-hops can be validated. Decide `network` statements vs. redistribution and justify.
- [ ] **Task 2.6 — eBGP next-hop into the AS. [3 points]** Ensure prefixes learned via eBGP are usable inside the receiving AS: the eBGP next-hop must be reachable by iBGP clients. Apply `next-hop-self` on the ASBR toward the RR (or redistribute the inter-AS /24) and verify a PE sees a valid next-hop.
- [ ] **Task 2.7 — Verify Inter-AS. [0 points — verification]** From R1 (AS65100), confirm you can see AS65200 and AS65300 loopbacks in the BGP table with correct AS_PATH. Verify end-to-end reachability loopback-to-loopback across each AS boundary.

> **Snapshot:** `WB04-01-ibgp-ebgp-up` once all iBGP + eBGP sessions are Established and inter-AS loopbacks are reachable.

---

## Section 3 — BGP Path Manipulation [28 points]

Goal: Control inbound/outbound path selection using the standard attribute toolkit. AS65100↔AS65200 is dual-linked (R5↔R11 and R6↔R12) — use that redundancy to demonstrate deterministic path preference.

- [ ] **Task 3.1 — LOCAL_PREF (primary/backup inter-AS link). [5 points]** On AS65100, make the R5↔R11 link the preferred path to AS65200 prefixes and R6↔R12 the backup, using LOCAL_PREF applied inbound at the ASBRs. Verify best-path selection and that failover to R6↔R12 occurs when R5↔R11 is shut.
- [ ] **Task 3.2 — AS-PATH prepend. [5 points]** On AS65200, influence AS65100's inbound path to AS65200 prefixes by prepending the AS on the less-preferred link (R6↔R12). Confirm AS65100 chooses the shorter AS_PATH via R5↔R11. Discuss why prepend is inbound-influence and how it interacts with the Task 3.1 LOCAL_PREF decision (LOCAL_PREF wins — demonstrate the precedence).
- [ ] **Task 3.3 — MED. [5 points]** AS65200 wants to influence how AS65100 sends traffic INTO AS65200. On R11 and R12 (AS65200 ASBRs), advertise routes to AS65100 with different MED values — lower MED on R11 (preferred entry) and higher MED on R12 (backup entry). On AS65100 side, enable `bgp always-compare-med` or `bgp deterministic-med` and explain the difference. Remove LOCAL_PREF and AS-PATH prepend temporarily to verify MED alone breaks the tie. Then restore all three and verify the best-path order: LOCAL_PREF (step 2) → AS-PATH (step 4) → MED (step 6).
- [ ] **Task 3.4 — Well-known communities. [5 points]** Tag a selected AS65300 loopback with `no-export` on R20 and confirm it is not re-advertised past the receiving AS's eBGP boundary. Tag another with `no-advertise` and observe the difference.
- [ ] **Task 3.5 — Extended communities (preview). [3 points]** On R12, set an arbitrary RT-style extended community on routes toward R21 and inspect it with `show bgp ... extended-community`. This primes the extended-community handling used by L3VPN in WB05.
- [ ] **Task 3.6 — Community-based policy. [5 points]** Build a route-map that matches a custom community (e.g., `65100:100`) and sets LOCAL_PREF, applied at an ASBR. Prove the policy only affects tagged prefixes.
- [ ] **Task 3.7 — Verify path selection order. [0 points — verification]** For one prefix reachable via multiple paths, walk the BGP best-path algorithm (`show bgp <prefix>`) and confirm the winning attribute at each step. Document the decision.

> **Snapshot:** `WB04-02-path-manipulation`.

---

## Section 4 — BGP Advanced [23 points]

- [ ] **Task 4.1 — Add-Path on RRs. [5 points]** Enable BGP Additional Paths (advertise/receive) on the RRs (R7/R8 in AS65100, R19 in AS65200, R30 in AS65300) so clients receive more than the single best path. Verify a client receives multiple paths for a dual-homed prefix (`show bgp ... <prefix>` shows additional path-IDs). Explain how this improves convergence and complements Add-Path with BGP PIC.
- [ ] **Task 4.2 — BGP PIC (Prefix-Independent Convergence). [5 points]** Enable BGP PIC Edge/Core so a precomputed backup path is installed in the RIB/FIB. Use the AS65100↔AS65200 dual link (Section 3) as the protected pair. Verify a backup path is present (`show ip cef <prefix> detail` / `show bgp <prefix>` repair path) and that failover is prefix-independent.
- [ ] **Task 4.3 — RT-Constraint (RTC). [5 points]** Enable the `rtfilter` (route-target constraint) address family between the RRs and their clients so VPNv4 routes are only sent to PEs that import the matching RT. Verify RTC NLRI is exchanged (`show bgp rtfilter unicast all`). Note dependency: VRFs/RTs are created in WB05 — document the RTC design now and validate once WB05 VRFs exist.
- [ ] **Task 4.4 — Conditional route advertisement. [5 points]** On an ASBR (e.g., R6), configure `advertise-map`/`non-exist-map` so a backup prefix is advertised to a neighbor only when a primary prefix disappears from the BGP table. Verify the advertise/withdraw triggers correctly by failing the primary.
- [ ] **Task 4.5 — Maximum-prefix on eBGP peers. [3 points]** Apply `neighbor <ip> maximum-prefix` with a warning threshold and restart interval on each inter-AS eBGP session (Tasks 2.1–2.4). Verify the warning fires and the peer is torn down when the hard limit is exceeded (use a test advertisement or lower the limit to trigger).
- [ ] **Task 4.6 — Verify advanced features coexist. [0 points — verification]** Confirm Add-Path, PIC, and RTC are simultaneously active without breaking the Section 1–3 baseline (sessions Established, best-paths unchanged where no policy applies).

> **Snapshot:** `WB04-03-bgp-advanced`.

---

---

## Completion Checklist

- [ ] AS65100: R7+R8 RRs; R1,R2,R5,R6 clients; IPv4 + VPNv4 AF active; all iBGP Established.
- [ ] AS65200: R19 RR; R12,R14,R15,R16 clients; IPv4 + VPNv4 AF active.
- [ ] AS65300: R30 RR; R20,R21,R22,R23 clients; IPv4 + VPNv4 AF active.
- [ ] eBGP up on all four inter-AS links (R5↔R11, R6↔R12, R6↔R20, R12↔R21).
- [ ] Inter-AS loopback reachability verified; BGP next-hops resolvable in each AS.
- [ ] LOCAL_PREF primary/backup, AS-PATH prepend, MED, and communities demonstrated with precedence proven.
- [ ] Add-Path, BGP PIC, RT-Constraint, conditional advertisement, maximum-prefix all configured and verified.
- [ ] Three troubleshooting faults reproduced, diagnosed, and repaired.
- [ ] Snapshots saved: `WB04-00-start`, `WB04-01-ibgp-ebgp-up`, `WB04-02-path-manipulation`, `WB04-03-bgp-advanced`, `WB04-04-complete`.

**Next:** WB05 — L3VPN (VRFs, RD/RT, PE-CE routing) builds directly on the VPNv4 control plane and RTC established here.

---

## Scoring Summary

| Section | Tasks | Points |
|---|---|---|
| Section 1 — iBGP + Route Reflectors | 1.1–1.6 | 17 |
| Section 2 — eBGP Inter-AS | 2.1–2.7 | 13 |
| Section 3 — BGP Path Manipulation | 3.1–3.7 | 28 |
| Section 4 — BGP Advanced | 4.1–4.6 | 23 |
| **Total** | | **81** |

### Rating Scale (% of points earned)

| Score | Rating |
|---|---|
| 90–100% | CCIE ready |
| 70–89% | Strong, minor gaps |
| 50–69% | Needs more practice |
| Below 50% | Review theory |

### Scoring Rules

- **From memory:** full points
- **Used `?` on router:** 2/3 points (round up)
- **Checked solution:** 1/3 points (round up)
- **Skipped:** 0 points

---

## Attempt History

### Attempt 1 — October 9-10, 2026

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: iBGP + RRs** | **17** | **17** | ✅ PERFECT |
| 1.1 iBGP AS65100 | 3 | 3 | ✅ R7/R8 RRs, RR-client, update-source, VPNv4 |
| 1.2 iBGP AS65200 | 3 | 3 | ✅ R19 RR, all clients |
| 1.3 iBGP AS65300 | 3 | 3 | ✅ R30 RR, all clients |
| 1.5 Cluster-ID | 5 | 5 | ✅ Distinct cluster-IDs on R7/R8 |
| 1.6 Verify | 0 | — | |
| **Section 2: eBGP Inter-AS** | **13** | **13** | ✅ PERFECT |
| 2.1-2.4 eBGP sessions | 8 | 8 | ✅ All 4 inter-AS links |
| 2.5 Advertise loopbacks | 2 | 2 | ✅ Network statements on ASBRs |
| 2.6 next-hop-self | 3 | 3 | ✅ All ASBRs toward RRs |
| **Section 3: Path Manipulation** | **28** | **28** | ✅ PERFECT |
| 3.1 LOCAL_PREF | 5 | 5 | ✅ R5 inbound, as-path match, LP 200 |
| 3.2 AS-PATH prepend | 5 | 5 | ✅ R12 outbound, match ^$, 3x prepend |
| 3.3 MED | 5 | 5 | ✅ R11 MED 50, R12 MED 100, always-compare-med |
| 3.4 no-export/no-advertise | 5 | 5 | ✅ R20 communities, send-community both |
| 3.5 Extended community | 3 | 3 | ✅ R12 sets 65200:200 |
| 3.6 Community-based LP | 5 | 5 | ✅ R21 matches community-list, sets LP 200 |
| **Section 4: Advanced** | **23** | **13** | |
| 4.1 Add-Path | 5 | 0 | ❌ Needed help with syntax |
| 4.2 BGP PIC | 5 | 5 | ✅ additional-paths install on PEs |
| 4.3 Conditional advertisement | 5 | 0 | ❌ Needed help with implementation |
| 4.4 Maximum-prefix | 3 | 3 | ✅ 20 80 restart 3 on all eBGP peers |
| 4.5 RT-Constraint | 5 | 5 | ✅ Configured on RRs and PEs |
| **TOTAL** | **81** | **71** | **88% — Strong** |

**Perfect sections:** iBGP + RRs (17/17), eBGP Inter-AS (13/13), Path Manipulation (28/28)
**Review for next attempt:** Add-Path syntax (4.1), Conditional advertisement (4.3)
**Minor fix during lab:** R5 router-id typo (was 150.1.6.6, fixed to 150.1.5.5), R8 missing update-source (typo, fixed)
