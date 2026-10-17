# WB05 — L3VPN (VRF, RD/RT, PE-CE Routing, VPNv4)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Prerequisite:** WB04 complete — iBGP RRs with VPNv4 AF, eBGP inter-AS, LDP transport. VPNv4 sessions Established in every AS.
**Format:** Tasks only. No configurations provided.
**Reference:** See `00_topology_reference.md` for PE-CE links, customer VPN design, and addressing.

> **Snapshot before you start:** `WB05-00-start` (WB04 BGP baseline clean).

**Scope note:** This workbook builds each customer VPN *within* a single AS. Cross-AS VPN stitching (Inter-AS Options A/B/C) is **WB09** — do not attempt cross-AS CE-CE reachability here.

---

## Section 1 — VRF + RD + RT [10 points]

Goal: Define per-customer VRFs on the PEs. Use a consistent, documented RD and RT scheme: **RD per-SP** (encode the AS and PE so RDs are globally unique) and **RT per-customer** (same RT across SPs for a given customer so routes import correctly later).

- [ ] **Task 1.1 — RD/RT scheme. [2 points]** Design and write down your scheme before configuring. Suggested: RD = `<AS>:<customer-id>` or `<PE-loopback-derived>:<customer>`; RT = `<customer-global>:<customer>`. The RT must be identical for a customer across all ASes so WB09 inter-AS import works. Record the table.
- [ ] **Task 1.2 — VRF GREEN (AS65100). [2 points]** Create VRF GREEN on R31 and R2. Assign the Green PE-CE interfaces to the VRF (R31 f4/0 to R9, R1 f3/0 to R10, R2 g1/0 to R10). Set RD and import/export RT for Green.
- [ ] **Task 1.3 — VRF GREEN (AS65200). [2 points]** Create VRF GREEN on R15 and R16. Assign R15 f0/0 (to R17, OSPF — see Section 3) and R16 g1/0 (to R18, eBGP) to VRF GREEN. Same Green RT, AS65200-local RD.
- [ ] **Task 1.4 — VRF YELLOW. [2 points]** Create VRF YELLOW on R22 and R23 (AS65300) and on R16 (AS65200). Assign R22 f0/0 (to R24), R23 f3/0 (to R25 — note R25 is Blue; confirm interface/customer mapping against the reference and only place Yellow interfaces here) and R16 g2/0 (to R26). Yellow RT consistent across ASes.
- [ ] **Task 1.5 — VRF BLUE. [2 points]** Create VRF BLUE on R31 (AS65100), assigning R31 g1/0 (to R32). Blue also appears on R25's connections in AS65200/65300 (R14, R21, R23) — create VRF BLUE on those PEs too and assign the correct R25-facing interfaces. Blue spans three ASes; keep the Blue RT identical everywhere.
- [ ] **Task 1.6 — Verify VRFs. [0 points — verification]** `show vrf`, `show ip vrf interfaces`. Confirm each PE-CE interface is in the intended VRF and the global table no longer owns those interfaces.

---

## Section 2 — PE-CE eBGP [15 points]

Goal: Bring up eBGP in each VRF between PE and CE. Watch for the AS_PATH loop problem on customers that reuse the same ASN at multiple sites — use `as-override` (or `allowas-in` on the CE) where the same customer ASN appears on both ends.

- [ ] **Task 2.1 — Green eBGP (AS65100). [3 points]** eBGP in VRF GREEN: R31↔R9 (`192.168.1.0/24`), R31↔R10 (`192.168.2.0/24`), R2↔R10 (`192.168.4.0/24`), all CE side AS65910. Advertise R9/R10 loopbacks and the R9–R10 link (`192.168.100.0/24`).
- [ ] **Task 2.2 — Green eBGP (AS65200). [3 points]** eBGP in VRF GREEN: R16↔R18 (`192.168.7.0/24`), R18 in AS65910.
- [ ] **Task 2.3 — Green as-override. [3 points]** Because R18 (AS65200 site) and R9/R10 (AS65100 sites) all use AS65910, routes crossing between Green sites will be dropped for AS_PATH loop. Apply `as-override` on the PEs (or `allowas-in` on CEs) so same-AS sites exchange routes. State which you chose and why. (Full cross-AS Green reachability still waits for WB09; within-AS this matters where two Green CEs attach in the same AS.)
- [ ] **Task 2.4 — Yellow eBGP. [3 points]** eBGP in VRF YELLOW: R20↔R24 and R22↔R24 (`192.168.9.0/24`, `192.168.11.0/24`, CE AS65024); R16↔R26 (`192.168.8.0/24`, CE AS65024). R24 is dual-homed to R20 and R22 within AS65300 — note it for SoO in Section 5.
- [ ] **Task 2.5 — Blue eBGP. [3 points]** eBGP in VRF BLUE: R31↔R32 (`192.168.3.0/24`, CE AS65025); R25 to R14 (`192.168.5.0/24`), R21 (`192.168.10.0/24`), R23 (`192.168.12.0/24`), all CE AS65025. Apply `as-override` as needed for the reused AS65025.
- [ ] **Task 2.6 — Verify PE-CE. [0 points — verification]** On each PE: `show bgp vpnv4 unicast vrf <name> summary` shows CE sessions Established; CE prefixes appear in the VRF table (`show ip route vrf <name>`).

> **Snapshot:** `WB05-01-vrf-pece-up`.

---

## Section 3 — PE-CE OSPF [18 points]

Goal: Run OSPF as the PE-CE protocol for Green between R15 (PE, AS65200) and R17 (CE). Handle the superbackbone, DN-bit, domain-id, and sham-link concepts.

- [ ] **Task 3.1 — OSPF PE-CE. [5 points]** Configure OSPF Area 0 in VRF GREEN between R15 f0/0 and R17 f0/0 (`192.168.6.0/24`). Use an OSPF process per-VRF on R15.
- [ ] **Task 3.2 — Mutual redistribution. [3 points]** Redistribute VRF GREEN BGP (VPNv4) into the VRF OSPF process and OSPF into BGP on R15. Verify R17's routes reach the VPNv4 table and MP-BGP VPN routes appear as OSPF to R17.
- [ ] **Task 3.3 — DN-bit / Down bit. [5 points]** Explain and verify the DN-bit (and the domain-tag for type-5/7) loop-prevention: routes redistributed from MP-BGP into OSPF carry the DN-bit so another PE will not re-inject them. Observe it with `show ip ospf database` detail on the CE/PE.
- [ ] **Task 3.4 — Domain-ID. [5 points]** Set the OSPF `domain-id` so VPN routes appear as OSPF inter-area (type-3) rather than external (type-5) to the CE where desired. Demonstrate the LSA-type difference with matching vs. mismatched domain-ids.
- [ ] **Task 3.5 — Verify OSPF PE-CE. [0 points — verification]** R17 learns Green prefixes; R15 VRF table shows OSPF and BGP sources correctly. Document which routes are type-3 vs type-5 and why.

> **Snapshot:** `WB05-02-ospf-pece`.

---

## Section 4 — VPNv4 on RRs [9 points]

Goal: Confirm the MP-BGP VPNv4 control plane carries VPN routes correctly within each AS via the RRs, with RT import/export and next-hop resolution working end to end.

- [ ] **Task 4.1 — VPNv4 propagation. [0 points — verification]** Verify VPN routes propagate PE → RR → PE within each AS. On the RR: `show bgp vpnv4 unicast all` shows all customer prefixes with RTs. On a receiving PE: the prefix is imported into the correct VRF only.
- [ ] **Task 4.2 — RT import/export correctness. [3 points]** Prove RT policy: a PE imports only the customers it is configured for. Temporarily misconfigure one import RT, observe the VRF losing routes, then restore. Use `show bgp vpnv4 unicast all rt <rt>`.
- [ ] **Task 4.3 — Next-hop resolution. [3 points]** Confirm the VPNv4 next-hop (remote PE loopback) is resolved via the IGP + LDP label (`show ip cef vrf <name> <prefix> detail` shows the VPN label and the LDP transport label — the two-label stack). Verify the imposition stack on an ingress PE.
- [ ] **Task 4.4 — RTC interaction (from WB04 Task 4.3). [3 points]** With VRFs now present, validate the RT-Constraint filtering built in WB04 actually limits VPNv4 advertisement to interested PEs. Confirm a PE with no matching import RT does not receive those VPNv4 NLRI.

> **Snapshot:** `WB05-03-vpnv4-verified`.

---

## Section 5 — Verification & SoO [5 points]

Goal: Validate CE-CE reachability within each AS and apply Site-of-Origin to the dual-homed CE to prevent routing loops.

- [ ] **Task 5.1 — Green CE-CE (within AS65100). [0 points — verification]** From R9, reach R10's loopback and the opposite PE-CE subnet through the Green VPN. Confirm the data path uses the MPLS VPN (traceroute shows the PE loopbacks / label switching, not a direct IGP leak).
- [ ] **Task 5.2 — Yellow CE-CE (within AS65300). [0 points — verification]** From R24, verify reachability to other Yellow prefixes that live within AS65300. (R26 is in AS65200 — defer that cross-AS leg to WB09.)
- [ ] **Task 5.3 — Blue within-AS checks. [0 points — verification]** For each AS where Blue exists, verify the local Blue CE prefixes are present in the Blue VRF. Cross-AS Blue reachability is WB09.
- [ ] **Task 5.4 — SoO on dual-homed CE. [5 points]** R9 is dual-homed (R31 f4/0 and R2 g1/0, both VRF GREEN, same CE AS65910). Apply a Site-of-Origin extended community on both PE-CE interfaces facing R10 so a route originated at the R10 site is not re-advertised back to the same site through the other PE. Verify the SoO is attached (`show bgp vpnv4 unicast vrf GREEN <prefix>` extended-community) and that no loop/suboptimal re-injection occurs.
- [ ] **Task 5.5 — Negative test (no cross-AS leak). [0 points — verification]** Confirm that Green/Yellow/Blue CE prefixes do **not** reach sites in other ASes yet — proving the control plane is correctly AS-scoped before WB09 stitches it.

> **Snapshot:** `WB05-04-complete`.

---

## Completion Checklist

- [ ] RD-per-SP / RT-per-customer scheme documented and applied consistently.
- [ ] VRF GREEN on R31,R2 (65100) and R15,R16 (65200); YELLOW on R22,R23 (65300) and R16 (65200); BLUE on R31 (65100), R14,R21,R23 (R25-facing).
- [ ] PE-CE eBGP up for Green, Yellow, Blue with `as-override`/`allowas-in` applied where ASNs are reused.
- [ ] PE-CE OSPF (R15↔R17) with mutual redistribution, DN-bit and domain-id behavior demonstrated.
- [ ] VPNv4 propagation via RRs verified; RT import/export correct; two-label stack confirmed via CEF.
- [ ] RTC from WB04 validated against real VRFs.
- [ ] Within-AS CE-CE reachability verified for each customer; cross-AS confirmed *not* reachable (WB09).
- [ ] SoO applied on R10 dual-homed interfaces and verified.
- [ ] Snapshots saved: `WB05-00-start`, `WB05-01-vrf-pece-up`, `WB05-02-ospf-pece`, `WB05-03-vpnv4-verified`, `WB05-04-complete`.

**Next:** WB06 — L2VPN (AToM/VPWS) for the Red VLAN 100 service.

---

## Scoring Summary

| Section | Tasks | Points |
|---|---|---|
| Section 1 — VRF + RD + RT | 1.1–1.6 | 10 |
| Section 2 — PE-CE eBGP | 2.1–2.6 | 15 |
| Section 3 — PE-CE OSPF | 3.1–3.5 | 18 |
| Section 4 — VPNv4 on RRs | 4.1–4.4 | 9 |
| Section 5 — Verification & SoO | 5.1–5.5 | 5 |
| **Total** | | **57** |

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

### Attempt 1 — October 10-11, 2026

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: VRF + RD + RT** | **10** | **10** | ✅ PERFECT |
| 1.1 RD/RT scheme | 2 | 2 | ✅ Per-AS RDs, consistent RTs across ASes |
| 1.2 VRF GREEN (AS65100) | 2 | 2 | ✅ R31, R2 — correct interfaces |
| 1.3 VRF GREEN (AS65200) | 2 | 2 | ✅ R15, R16 — RD initially wrong (65100:100 on R16), fixed |
| 1.4 VRF YELLOW | 2 | 2 | ✅ R20, R22, R16 |
| 1.5 VRF BLUE | 2 | 2 | ✅ R1, R14, R21, R23 — spans 3 ASes |
| **Section 2: PE-CE eBGP** | **15** | **15** | ✅ PERFECT |
| 2.1 Green eBGP (AS65100) | 3 | 3 | ✅ R31↔R9, R31↔R10, R2↔R10 all AS65910 |
| 2.2 Green eBGP (AS65200) | 3 | 3 | ✅ R16↔R18 |
| 2.3 as-override | 3 | 3 | ✅ On R31, R2, R16 for same-AS Green CEs |
| 2.4 Yellow eBGP | 3 | 3 | ✅ R20↔R24, R22↔R24, R16↔R26 |
| 2.5 Blue eBGP | 3 | 3 | ✅ R31↔R32, R14↔R25, R21↔R25, R23↔R25 |
| **Section 3: PE-CE OSPF** | **18** | **18** | ✅ PERFECT |
| 3.1 OSPF PE-CE | 5 | 5 | ✅ OSPF 2 VRF GREEN, R15↔R17, adjacency FULL |
| 3.2 Mutual redistribution | 3 | 3 | ✅ ospf→bgp and bgp→ospf both configured |
| 3.3 DN-bit | 5 | 5 | ✅ Understands concept, deferred verification to Inter-AS |
| 3.4 Domain-ID + domain-tag | 5 | 5 | ✅ domain-id 150.2.15.15, domain-tag 65200 |
| **Section 4: VPNv4 on RRs** | **9** | **6** | |
| 4.2 RT import/export | 3 | 3 | ✅ Correct RTs verified |
| 4.3 Next-hop resolution | 3 | 3 | ✅ VPN + LDP labels working |
| 4.4 RTC | 3 | — | ⏸️ Deferred to WB09 (configured but removed — filters routes without Inter-AS endpoints) |
| **Section 5: SoO** | **5** | **5** | ✅ PERFECT |
| 5.4 SoO | 5 | 5 | ✅ R10 (R1+R2), R24 (R20+R22), R25 (R21+R23) — all from memory |
| **TOTAL** | **57** | **54** | **95% — Excellent** (3 pts deferred RTC) |

**Perfect sections:** VRF+RD+RT (10/10), PE-CE eBGP (15/15), PE-CE OSPF (18/18), SoO (5/5)
**Deferred:** RT-Constraint (4.4) — will score after WB09 Inter-AS provides remote VRF endpoints
**Minor fixes during lab:** R16 GREEN RD was 65100:100 (wrong AS), R14 BLUE missing PE-CE neighbor, R15 missing redistribute bgp under OSPF
**Key learning:** Always verify RD per-router, always check PE-CE neighbor exists under VRF AF after creating VRF
