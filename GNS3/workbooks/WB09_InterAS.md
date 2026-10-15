# WB09 — Inter-AS MPLS L3VPN (Options A & B)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** Inter-AS VPNv4 across AS 65100 (IS-IS), AS 65200 (OSPF), AS 65300 (IS-IS)
**Level:** CCIE SP
**Prerequisite:** WB00-WB07 complete (per-AS IGP, LDP, intra-AS L3VPN, RRs all working)

> **Format:** Tasks only. No configurations provided.
> Work **progressively** — each section builds on the previous; verify before advancing.
> **Snapshot** before and after each section so you can isolate issues.

> **Note:** Inter-AS Option C (BGP-LU + multihop VPNv4) is not supported on IOS classic 7200/Dynamips
> due to label allocation limitations on route reflectors. Option C will be practiced on EVE-NG (IOS-XR).
> This workbook covers Options A and B, which provide full inter-AS connectivity across all 3 ASes.

> **ASBR / RR reference:**
> AS65100 — ASBRs R5, R6; RRs R7, R8.
> AS65200 — ASBR R12; RR R19. (R11 is a P router but serves as Option A ASBR for the R5↔R11 link)
> AS65300 — ASBRs R20, R21; RR R30.
> Inter-AS links: R5↔R11, R6↔R12 (65100↔65200); R6↔R20 (65100↔65300); R12↔R21 (65200↔65300).

---

## Pre-flight

- [ ] Each AS IGP converged independently (`show isis nei` / `show ip ospf nei`)
- [ ] Intra-AS LDP + VPNv4 working inside each AS
- [ ] Intra-AS L3VPN verified (Green/Yellow/Blue VRFs per WB05)
- [ ] **Snapshot:** `WB09-00-baseline`

---

## Section 1 — Inter-AS Option A: Green across AS65100 ↔ AS65200 [5 points]

**Link:** R5 (g1/0) ↔ R11 (g1/0) — `10.5.11.0/24`
**Customer:** Green (CE9/CE10 in AS65100 ↔ CE17/CE18 in AS65200)
**Goal:** End-to-end Green VPN reachability using back-to-back VRF on the ASBRs.

- [ ] **Task 1.1 [1 point]** — On R5 create VRF GREEN (matching RD/RT policy from WB05). On R11 create VRF GREEN (if not already present as a P router, R11 now acts as an Option A ASBR for this link).
- [ ] **Task 1.2 [2 points]** — On the R5↔R11 physical link, create a **per-VRF sub-interface** with dot1q encapsulation placed into VRF GREEN on both sides. Remove the existing global IP from the physical interface. Choose a VLAN tag and new /30 or /24 subnet for the VRF sub-interface.
- [ ] **Task 1.3 [1 point]** — Configure a **per-VRF eBGP session** between R5 and R11 inside VRF GREEN over the sub-interface. Exchange Green customer routes across the boundary. No VPNv4, no label exchange — plain IPv4 inside the VRF.
- [ ] **Task 1.4 [1 point]** — Ensure Green routes learned from the far AS via the per-VRF eBGP are redistributed into each side's VPNv4 table so they reach the PEs (R1/R2 in AS65100, R15/R16 in AS65200) via the RRs.
- [ ] **Task 1.5 [0 points — verification]** — Verify end-to-end: CE9 (AS65100) ↔ CE18 (AS65200), CE10 ↔ CE17. Confirm no MPLS label on the R5↔R11 ASBR link (IP forwarding per-VRF only).

**Verify / Snapshot**
- [ ] `show ip route vrf GREEN` on R5 and R11 — far-AS Green prefixes present
- [ ] `show bgp vpnv4 unicast vrf GREEN` on R5 and R11
- [ ] End-to-end: CE9 ping CE18, CE10 ping CE17 (both directions)
- [ ] Confirm **no** label on the R5↔R11 link (`traceroute` shows IP hop, not MPLS)
- [ ] **Snapshot:** `WB09-01-optionA`

---

## Section 2 — Inter-AS Option B: Yellow + Blue across AS65200 ↔ AS65300 [5 points]

**Link:** R12 (f3/0) ↔ R21 (f3/0) — `20.12.21.0/24`
**Customers:** Yellow (R26 AS65200 ↔ R24 AS65300) + Blue (R14/R25 AS65200 ↔ R21/R23/R25 AS65300)
**Goal:** Full VPNv4 exchange between AS65200 and AS65300 for all shared customers.

- [ ] **Task 2.1 [1 point]** — Establish a **VPNv4 eBGP** session between R12 (AS65200) and R21 (AS65300) under `address-family vpnv4`. Use the directly connected inter-AS addresses.
- [ ] **Task 2.2 [1 point]** — Configure **`next-hop-self`** on both ASBRs toward their respective iBGP peers (RRs) under VPNv4 so the remote ASBR's address is replaced with the local ASBR's loopback — ensuring iBGP peers can resolve the VPNv4 next-hop via their local IGP.
- [ ] **Task 2.3 [1 point]** — Configure **RT retention** on both R12 and R21 (`no bgp default route-target filter`) so VPNv4 routes for customers not locally present on the ASBR are retained and passed through.
- [ ] **Task 2.4 [1 point]** — Ensure **`send-community extended`** is configured on the VPNv4 eBGP session so RTs cross the AS boundary.
- [ ] **Task 2.5 [1 point]** — On the receiving PEs, add **import RTs** for the far-AS customer routes. For example: R22 (AS65300, VRF YELLOW) must import RT 65200:200 to accept Yellow routes from AS65200. R16 (AS65200, VRF YELLOW) must import RT 65300:300 to accept Yellow routes from AS65300. Same for Blue across both ASes.
- [ ] **Task 2.6 [0 points — verification]** — Verify Yellow end-to-end: R26 (AS65200) ↔ R24 (AS65300). Verify Blue end-to-end between AS65200 and AS65300 PEs.

**Verify / Snapshot**
- [ ] `show bgp vpnv4 unicast all` on R12/R21 — far-AS VPNv4 with correct RTs
- [ ] `show bgp vpnv4 unicast all summary` — VPNv4 eBGP session UP, PfxRcvd > 0
- [ ] `show mpls forwarding-table` on ASBR — VPNv4 label swap entries
- [ ] RT-retain confirmed: `show bgp vpnv4 unicast all` shows routes for non-local VRFs
- [ ] End-to-end: R26 ↔ R24 (Yellow), Blue PEs across AS65200 ↔ AS65300
- [ ] **Snapshot:** `WB09-02-optionB`

---

## Section 3 — Full Inter-AS Connectivity: All Customers Across All 3 ASes [8 points]

**Goal:** Complete end-to-end reachability for ALL customers across ALL three ASes using a combination of Option A and Option B on the available inter-AS links.

**Design:**
```
R5 ↔ R11  (65100↔65200): Option A — already configured in Section 1 (Green only)
R6 ↔ R12  (65100↔65200): Option A — per-VRF sub-interfaces for Green, Yellow, Blue (3 VRFs)
R6 ↔ R20  (65100↔65300): Option B — VPNv4 eBGP for all customers
R12 ↔ R21 (65200↔65300): Option B — already configured in Section 2 (Yellow + Blue)
```

- [ ] **Task 3.1 [2 points]** — On R6 ↔ R12, configure **Option A** with three per-VRF sub-interfaces (dot1q) for GREEN, YELLOW, and BLUE. Remove the global IP from the physical interface. Create per-VRF eBGP sessions between R6 and R12 inside each VRF. This provides a second 65100↔65200 path with per-VRF isolation and carries all three customers.
- [ ] **Task 3.2 [2 points]** — On R6 ↔ R20, establish a **VPNv4 eBGP** session (Option B). Configure next-hop-self, send-community extended, and RT retention. This connects AS65100 directly to AS65300 for Blue, Yellow, and Green.
- [ ] **Task 3.3 [2 points]** — Update **import RTs** on all PEs across all three ASes so each customer VRF imports the RTs from every AS where that customer exists:
  - Green PEs: import RT 65100:100 + 65200:100
  - Yellow PEs: import RT 65200:200 + 65300:300
  - Blue PEs: import RT 65100:25 + 65200:25 + 65300:25
- [ ] **Task 3.4 [2 points]** — Verify **full end-to-end reachability** for each customer:
  - **Green:** CE9 (AS65100) ↔ CE17 (AS65200, OSPF PE-CE) ↔ CE18 (AS65200)
  - **Yellow:** R24 (AS65300) ↔ R26 (AS65200) — and via transit if Yellow exists in AS65100
  - **Blue:** R32 (AS65100) ↔ R25/R14 (AS65200) ↔ R25/R21/R23 (AS65300) — Blue spans all 3 ASes

**Verify / Snapshot**
- [ ] `show bgp vpnv4 unicast all summary` on ALL ASBRs — all VPNv4 sessions UP
- [ ] `show bgp vpnv4 unicast vrf GREEN` on PEs in both ASes — far-AS prefixes imported
- [ ] `show bgp vpnv4 unicast vrf YELLOW` on R16 and R22 — cross-AS prefixes present
- [ ] `show bgp vpnv4 unicast vrf BLUE` on R1, R14, R21, R23 — prefixes from all 3 ASes
- [ ] End-to-end ping/traceroute for each customer across AS boundaries
- [ ] Confirm label stacks on traceroute (VPN label + LDP transport within each AS, IP at Option A boundary)
- [ ] **Snapshot:** `WB09-03-full-connectivity`

---

## Section 4 — Cross-AS OSPF VPN (Green OSPF site R17) [5 points]

After inter-AS is working, validate the **OSPF PE-CE** Green site at **R17** (behind R15, AS65200) reaches remote Green sites in AS65100.

- [ ] **Task 4.1 [1 point]** — Confirm R17's OSPF PE-CE (OSPF process 2, VRF GREEN on R15) is up and routes are in VPNv4. Verify R15 has both `redistribute ospf 2` (under BGP VRF GREEN) and `redistribute bgp 65200 subnets` (under OSPF 2 VRF GREEN) — mutual redistribution.
- [ ] **Task 4.2 [2 points]** — Verify that remote Green prefixes from AS65100 (CE9: 150.9.9.9, CE10: 150.10.10.10) are redistributed from VPNv4 into OSPF on R15 and appear on R17 as OSPF routes. Check the **LSA type**: with matching domain-ID they appear as Type-3 (inter-area); with mismatched domain-ID they appear as Type-5 (external).
- [ ] **Task 4.3 [1 point]** — Verify **DN-bit** is set on the LSAs redistributed from BGP into OSPF on R15. Confirm the DN-bit prevents another PE (if one existed) from re-importing these routes back into BGP.
- [ ] **Task 4.4 [1 point]** — Verify **domain-tag** is set on Type-5 LSAs to provide backup loop prevention.
- [ ] **Task 4.5 [0 points — verification]** — End-to-end: R17 ping CE9 (150.9.9.9) and CE10 (150.10.10.10) sourcing from R17's loopback. Traceroute to confirm the MPLS + inter-AS path.

**Verify / Snapshot**
- [ ] `show ip route vrf GREEN` on R15 — remote Green prefixes from BGP
- [ ] `show ip route` on R17 — remote Green prefixes as OSPF routes (IA or E2)
- [ ] `show ip ospf database` on R17 — LSA type and DN-bit visible
- [ ] End-to-end: R17 ↔ CE9, R17 ↔ CE10
- [ ] **Snapshot:** `WB09-04-ospf-vpn`

---

## Scoring Summary

| Section | Max Points | Description |
|---------|-----------|-------------|
| Section 1: Inter-AS Option A (Green, R5↔R11) | 5 | Back-to-back VRF, per-VRF sub-interface, PE-CE eBGP |
| Section 2: Inter-AS Option B (Yellow+Blue, R12↔R21) | 5 | VPNv4 eBGP, RT retention, next-hop-self |
| Section 3: Full Inter-AS (all customers, all 3 ASes) | 8 | Option B on R6↔R12 + R6↔R20, cross-AS RT import, end-to-end verification |
| Section 4: Cross-AS OSPF VPN (R17 Green) | 5 | Mutual redistribution, DN-bit, domain-ID, domain-tag |
| **Total** | **23** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
20-23 points:  CCIE ready for this topic
15-19 points:  Strong, minor gaps to review
10-14 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Option A (R5↔R11) — Green CE9 ↔ CE17/CE18 reachable, no label on ASBR link
- [ ] Option B (R12↔R21) — Yellow R26 ↔ R24 reachable, labeled VPNv4 + RT-retain
- [ ] Option B (R6↔R12) — Second 65100↔65200 path for all customers
- [ ] Option B (R6↔R20) — AS65100↔AS65300 direct path for Blue
- [ ] All customer VRF import RTs updated for cross-AS reachability
- [ ] Blue verified across all 3 ASes (R32 ↔ R14/R25 ↔ R21/R23)
- [ ] Cross-AS OSPF Green (R17) reaching AS65100 sites, correct LSA types
- [ ] **Final snapshot:** `WB09-FINAL`

---

## Attempt History

### Attempt 1 — October 13-14, 2026

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Option A (R5↔R11, Green)** | **5** | **5** | ✅ PERFECT |
| 1.1 VRF GREEN on ASBRs | 1 | 1 | ✅ From memory |
| 1.2 Per-VRF sub-interface (dot1q) | 2 | 2 | ✅ From memory — removed global IP, created sub-interface |
| 1.3 Per-VRF eBGP session | 1 | 1 | ✅ From memory |
| 1.4 Redistribute into VPNv4 | 1 | 1 | ✅ Routes propagate to PEs via RRs |
| **Section 2: Option B (R12↔R21, Yellow+Blue)** | **5** | **4** | |
| 2.1 VPNv4 eBGP session | 1 | 1 | ✅ From memory |
| 2.2 next-hop-self | 1 | 1 | ✅ From memory |
| 2.3 RT retention | 1 | 0 | ❌ Needed help — didn't know `no bgp default route-target filter` |
| 2.4 send-community extended | 1 | 1 | ✅ From memory |
| 2.5 Import RTs on receiving PEs | 1 | 1 | ✅ From memory |
| **Section 3: Full Connectivity (all links, all customers)** | **8** | **8** | ✅ PERFECT |
| 3.1 R6↔R12 Option A (3 VRFs) | 2 | 2 | ✅ From memory — own design choice, Green+Yellow+Blue sub-interfaces |
| 3.2 R6↔R20 Option B | 2 | 2 | ✅ From memory — VPNv4 eBGP, NHS, send-community, RT retention |
| 3.3 Cross-AS import RTs | 2 | 2 | ✅ Updated all PEs across all 3 ASes |
| 3.4 End-to-end verification | 2 | 2 | ✅ All customers reachable across AS boundaries |
| **Section 4: Cross-AS OSPF VPN (R17)** | **5** | **5** | ✅ PERFECT |
| 4.1 OSPF PE-CE confirmed | 1 | 1 | ✅ R15 OSPF 2 VRF GREEN up, routes in VPNv4 |
| 4.2 Remote prefixes in OSPF on R17 | 2 | 2 | ✅ CE9/CE10 reachable from R17, LSA types verified |
| 4.3 DN-bit verified | 1 | 1 | ✅ From memory |
| 4.4 Domain-tag verified | 1 | 1 | ✅ From memory |
| **TOTAL** | **23** | **22** | **96% — Excellent** |

**Perfect sections:** Option A (5/5), Full Connectivity (8/8), Cross-AS OSPF VPN (5/5)
**From memory:** Option A sub-interfaces, per-VRF eBGP, Option B VPNv4, next-hop-self, send-community, import RTs, R6↔R12 3-VRF design, OSPF PE-CE DN-bit/domain-tag
**Needed help:** `no bgp default route-target filter` (0 points — wouldn't know on exam day)
**Design choices:** R6↔R12 as Option A with 3 VRFs (own decision), R6↔R20 as Option B (own decision)
**Note:** Inter-AS Option C not tested (platform limitation on Dynamips, deferred to WB25/WB28 on EVE-NG)
