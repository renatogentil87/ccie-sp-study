# WB09 — Inter-AS MPLS L3VPN (Options A / B / C)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** Inter-AS VPNv4 across AS 65100 (IS-IS), AS 65200 (OSPF), AS 65300 (IS-IS)
**Level:** CCIE SP
**Prerequisite:** WB00-WB07 complete (per-AS IGP, LDP, intra-AS L3VPN, RRs all working)

> **Format:** Tasks only. No configurations provided.
> Work **progressively** — each option is independent; verify before advancing.
> **Snapshot** before and after each option so you can isolate issues.

> **ASBR / RR reference:**
> AS65100 — ASBRs R5, R6; RRs R7, R8.
> AS65200 — ASBR R12; RR R19.
> AS65300 — ASBRs R20, R21; RR R30.
> Inter-AS links: R5↔R11, R6↔R12 (65100↔65200); R6↔R20 (65100↔65300); R12↔R21 (65200↔65300).

---

## Pre-flight

- [ ] Each AS IGP converged independently (`show isis nei` / `show ip ospf nei`)
- [ ] Intra-AS LDP + VPNv4 working inside each AS
- [ ] No inter-AS routing exists yet (clean start)
- [ ] **Snapshot:** `WB09-00-baseline`

---

## Section 1 — Inter-AS Option A (back-to-back VRF)

Customer **Green** across **AS65100 ↔ AS65200**, ASBRs **R5 ↔ R11**. Validate CE9 (AS65100) ↔ CE18 (AS65200).

- [ ] **Task 1.1** — On R5 and R11 create a VRF for Green (matching RD/RT policy per existing WB design).
- [ ] **Task 1.2** — On the R5↔R11 link (R5 g1/0 ↔ R11 g1/0) create a **per-VRF sub-interface** (dot1q) placed into the Green VRF on both sides. Option A treats the ASBR peer as a plain CE.
- [ ] **Task 1.3** — Run a **per-VRF eBGP (or static/IGP) PE-CE-style session** between R5 and R11 inside the Green VRF over that sub-interface to exchange Green routes. (No VPNv4, no label exchange across the boundary — this is the Option A hallmark.)
- [ ] **Task 1.4** — Ensure Green routes learned from the far AS are redistributed into each side's VPNv4 so they reach the respective PEs.
- [ ] **Task 1.5** — Verify CE9 (behind AS65100) reaches CE18 (behind AS65200) and back.

**Verify / Snapshot**
- [ ] `show ip route vrf GREEN` on R5 and R11 — far-AS prefixes present
- [ ] `show ip bgp vpnv4 vrf GREEN` on both ASBRs
- [ ] End-to-end: CE9 ping/traceroute CE18 (both directions)
- [ ] Confirm **no** label on the ASBR-ASBR link (IP forwarding per-VRF)
- [ ] **Snapshot:** `WB09-01-optionA`

---

## Section 2 — Inter-AS Option B (VPNv4 eBGP between ASBRs)

Customer **Yellow** across **AS65200 ↔ AS65300**, ASBRs **R12 ↔ R21**. Validate R26 (Yellow, AS65200) ↔ R24 (Yellow, AS65300).

> Note: topology inter-AS link for 65200↔65300 is R12 g? ↔ R21 (20.12.21.0/24). The task text references "R6↔R12" generically; use the **R12↔R21** link that actually joins AS65200 and AS65300.

- [ ] **Task 2.1** — Establish a **VPNv4 eBGP** session directly between ASBRs R12 and R21 (address-family vpnv4).
- [ ] **Task 2.2** — Configure **next-hop-self** so the ASBR sets itself as next-hop for VPNv4 routes sent to its own IBGP, and ensure the ASBR-ASBR link has a label path (the ASBRs exchange labeled VPNv4; the local IGP/LDP resolves the next-hop).
- [ ] **Task 2.3** — Configure **RT retention** on the ASBRs (`no bgp default route-target filter`, i.e. retain all RTs) so VPNv4 routes are not dropped for lack of a locally-imported RT.
- [ ] **Task 2.4** — Confirm labeled VPNv4 is exchanged across R12↔R21 (two-label model, with the ASBR as the next-hop boundary).
- [ ] **Task 2.5** — Verify R26 (AS65200) ↔ R24 (AS65300) end to end.

**Verify / Snapshot**
- [ ] `show ip bgp vpnv4 all` on R12/R21 — far-AS VPNv4 with next-hop = peer ASBR
- [ ] `show ip bgp vpnv4 all labels` — label present across the boundary
- [ ] `show mpls forwarding-table` on ASBR — VPNv4 label swap
- [ ] Confirm RT-retain working (routes present even without local import)
- [ ] End-to-end: R26 ↔ R24
- [ ] **Snapshot:** `WB09-02-optionB`

---

## Section 3 — Inter-AS Option C (BGP-LU + multihop VPNv4 RR-to-RR)

Customer **Blue** across **AS65100 ↔ AS65300**, boundary **R6 ↔ R20**. Multihop VPNv4 between RRs **R7/R8 (AS65100) ↔ R30 (AS65300)**. Validate R32 (Blue, AS65100) ↔ R25 (Blue, AS65300).

- [ ] **Task 3.1** — On the ASBRs R6 and R20 enable **BGP labeled-unicast (BGP-LU, send-label)** for IPv4 so the two ASes exchange *labeled loopback (/32 PE+RR) reachability* across the boundary. This builds an end-to-end LSP for the remote RR/PE loopbacks.
- [ ] **Task 3.2** — Redistribute (controlled) the remote PE/RR loopbacks into each AS and advertise them with labels via BGP-LU so an end-to-end label path exists from AS65100 PEs to AS65300 PEs.
- [ ] **Task 3.3** — Build a **multihop eBGP VPNv4** session between RR **R7/R8 (AS65100)** and RR **R30 (AS65300)** (ebgp-multihop, update-source Loopback0). VPNv4 routes are exchanged RR-to-RR; the ASBRs carry **no VPNv4 state** (Option C hallmark).
- [ ] **Task 3.4** — Keep next-hop **unchanged** on the multihop VPNv4 session (remote PE loopback stays as next-hop) and ensure that next-hop is reachable via the BGP-LU LSP from Task 3.1.
- [ ] **Task 3.5** — Verify an **end-to-end labeled path**: ingress PE label stack = VPN label + BGP-LU transport label(s). Validate R32 ↔ R25.

**Verify / Snapshot**
- [ ] `show ip bgp labels` / `show bgp ipv4 unicast labels` on ASBRs — labeled /32s crossing
- [ ] `show ip bgp vpnv4 all summary` on R7/R8/R30 — multihop VPNv4 session UP
- [ ] `show ip bgp vpnv4 all <remote-PE-prefix>` — next-hop = remote PE loopback, resolved via BGP-LU
- [ ] ASBR R6/R20: confirm **no** VPNv4 table entries
- [ ] `traceroute`/label trace R32 → R25 shows full label stack
- [ ] **Snapshot:** `WB09-03-optionC`

---

## Section 4 — Multi-AS Blue (spans 3 ASes, Option C stitching)

Customer **Blue** full reach: **R32 (AS65100) ↔ R25 (AS65200) ↔ R25 (AS65300)**. Stitch Option C across all three ASes using BGP-LU.

- [ ] **Task 4.1** — Extend BGP-LU so AS65200 participates: labeled transport reachability must exist 65100↔65200 (R5/R6↔R11/R12) **and** 65200↔65300 (R12↔R21), giving an unbroken LSP across all three cores.
- [ ] **Task 4.2** — Establish the multihop VPNv4 relationships so Blue VPNv4 routes propagate across all three ASes (RR mesh / chained multihop as appropriate: R7/R8 ↔ R19 ↔ R30, or direct multihop with next-hop reachability through BGP-LU).
- [ ] **Task 4.3** — Confirm Blue prefixes from R32 (AS65100) and both R25 attachments (AS65200 + AS65300) are mutually reachable.
- [ ] **Task 4.4** — Validate the BGP-LU label stitching at each AS boundary (label swap of the transport label at each ASBR, VPN label preserved end-to-end).

**Verify / Snapshot**
- [ ] `show ip bgp vpnv4 all` — Blue prefixes from all 3 ASes
- [ ] `show ip bgp labels` at each ASBR — transport label swapped at boundaries
- [ ] End-to-end traceroute R32 → R25(65300) transiting all three cores
- [ ] **Snapshot:** `WB09-04-multiAS-blue`

---

## Section 5 — Compare Options A / B / C (written analysis)

Capture in the workbook (short written answers, not config):

- [ ] **Task 5.1** — Scalability: how VPNv4 state and label state grow on ASBRs for A vs B vs C. Which scales best and why.
- [ ] **Task 5.2** — State on ASBRs: A = full per-VRF IP state; B = VPNv4 + label state; C = **no** VPNv4 on ASBR (only BGP-LU transport). Record exactly what each ASBR holds.
- [ ] **Task 5.3** — Complexity / security / operational tradeoffs: VRF explosion (A), RT-retain + single-hop trust (B), multihop RR trust + end-to-end LSP (C).
- [ ] **Task 5.4** — When to pick each in a real SP (number of VPNs, trust between providers, label transparency).

- [ ] **Snapshot:** `WB09-05-compare` (notes committed)

---

## Section 6 — Cross-AS OSPF VPN (Green OSPF site R17)

After inter-AS is working, extend the **OSPF PE-CE** Green site at **R17** (behind R15, AS65200) to reach remote Green sites across the AS boundary.

- [ ] **Task 6.1** — Confirm R17's OSPF PE-CE VRF (Green-OSPF) is up on R15 and routes are in VPNv4.
- [ ] **Task 6.2** — Redistribute **BGP (VPNv4) → OSPF** on the PE toward R17 with a **prefix filter** (route-map / prefix-list) so only intended remote Green prefixes are injected (avoid leaking the full table, prevent domain-id / loop issues).
- [ ] **Task 6.3** — Set/verify the OSPF **domain-id** and sham-link considerations if the remote Green site is also OSPF (decide whether a sham-link is needed; document the decision).
- [ ] **Task 6.4** — Confirm R17 learns remote Green prefixes as OSPF routes (correct LSA type: inter-area vs external based on domain-id) and has end-to-end reachability.

**Verify / Snapshot**
- [ ] `show ip route vrf GREEN-OSPF` on R15 and `show ip route` on R17
- [ ] `show ip ospf database` on R17 — remote prefixes present, expected LSA type
- [ ] Prefix filter working: unwanted prefixes absent
- [ ] End-to-end: R17 ↔ a remote Green CE
- [ ] **Snapshot:** `WB09-06-ospf-vpn`

---

## Section 7 — Troubleshooting Drills

- [ ] **Drill 7.1 — Option C next-hop unreachable:** remove/break the BGP-LU labeled path for the remote PE loopback. The multihop VPNv4 session may stay UP but prefixes become **unreachable / inaccessible** (next-hop not resolvable via label). Diagnose that the VPNv4 next-hop has no BGP-LU label path, then restore.
  - [ ] `show ip bgp vpnv4 all <prefix>` → "not advertised / inaccessible" next-hop
  - [ ] `show ip bgp labels` → missing labeled /32
  - [ ] `show mpls forwarding-table <remote-PE>` → no label path
- [ ] **Drill 7.2 — Option A VRF mismatch:** mis-assign the ASBR sub-interface to the wrong VRF (or mismatch RD/RT) on one side of R5↔R11. Confirm Green routes stop crossing. Diagnose VRF/sub-interface/RT mismatch, then fix.
  - [ ] `show ip route vrf GREEN` → far prefixes missing on one side
  - [ ] `show run interface <subif>` / `show vrf` → wrong VRF binding
- [ ] **Snapshot:** `WB09-07-tshoot`

---

## Completion Criteria

- [ ] Option A (R5↔R11) — CE9 ↔ CE18 reachable, no label on ASBR link
- [ ] Option B (R12↔R21) — R26 ↔ R24 reachable, labeled VPNv4 + RT-retain + next-hop-self
- [ ] Option C (R6↔R20, RR R7/R8↔R30) — R32 ↔ R25 end-to-end labeled, no VPNv4 on ASBR
- [ ] Multi-AS Blue reachable across all 3 ASes via BGP-LU stitching
- [ ] A/B/C comparison documented
- [ ] Cross-AS OSPF Green (R17) reaching remote sites with prefix filter
- [ ] Both troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB09-FINAL`
