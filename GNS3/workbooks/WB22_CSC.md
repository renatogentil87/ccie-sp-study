# WB22 — Carrier Supporting Carrier (CSC)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Hierarchical VPN (carrier's carrier). Backbone carrier = **AS 65200**; customer carrier = **AS 65300**
**Level:** CCIE SP
**Prerequisite:** WB05 (L3VPN VRF/RD/RT), WB14 (BGP-LU / Unified MPLS), WB16 (SR-MPLS, optional transport)

> **Platform note:** This workbook uses **IOS-XR syntax** and is written for the EVE-NG XR lab so it
> aligns with WB16-WB21. (CSC itself — BGP-LU / LDP label exchange on PE-CE — is a classic feature;
> the IOS-XR syntax is used here for consistency with the SR/EVPN/NSO track.)
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: verify each section before advancing.
> **Snapshot** before/after each section.

> **CSC model used here:**
> - **Backbone carrier (CSC-PE):** AS 65200 — CSC-PE = **R12** (ASBR, 150.2.12.12); backbone core R11/R13; backbone RR R19.
> - **Customer carrier (CSC-CE):** AS 65300 — CSC-CE = **R21** (ASBR, 150.3.21.21).
> - **CSC link:** R12 (f3/0) ↔ R21 (f3/0) — `20.12.21.0/24` (inter-AS link from `00_topology_reference.md`).
> - The customer carrier (AS 65300) runs its **own MPLS/VPN** (e.g. a customer-of-customer L3VPN or internal LSPs
>   between R20/R21/R22/R23/R30). The backbone (AS 65200) carries the customer carrier's **loopbacks + labels**
>   inside a VRF (e.g. **VRF CSC65300**), WITHOUT importing the customer carrier's full internal/customer routes.

---

## Pre-flight

- [ ] AS 65200 backbone: OSPF core + transport LSPs (LDP or SR) UP between R11/R12/R13/R19
- [ ] AS 65300 customer carrier: IS-IS + its own MPLS/VPN operational (R20-R23, R30)
- [ ] CSC link R12↔R21 (`20.12.21.0/24`) up at L3 but not yet in a VRF
- [ ] **Snapshot:** `WB22-00-baseline`

---

## Section 1 — CSC-PE VRF + CSC-CE Setup [4 points]

- [ ] **Task 1.1 [2 points]** — On the backbone CSC-PE **R12**, create **VRF CSC65300** (RD/RT per a documented CSC policy). Place the **CSC link sub-interface/interface toward R21** into VRF CSC65300. The backbone treats AS 65300 as a "customer" whose routes live only in this VRF.
- [ ] **Task 1.2 [1 point]** — On the customer-carrier CSC-CE **R21**, configure the CSC link in the **global** table (R21 is a normal P/ASBR of its own AS; it does NOT see the backbone VRF).
- [ ] **Task 1.3 [1 point]** — Confirm L3 reachability across the CSC link within VRF CSC65300 on R12 and global on R21.
- [ ] **Task 1.4 [0 points — verification]** — Confirm the backbone's other PEs do NOT see AS 65300 routes except via the CSC VRF.

**Verify / Snapshot**
- [ ] `show vrf CSC65300` (R12) — CSC interface a member
- [ ] `show route vrf CSC65300` (R12) / `show route` (R21) — CSC link reachable
- [ ] **Snapshot:** `WB22-01-csc-vrf`

---

## Section 2 — Label Distribution on the CSC Link (CE↔PE) [5 points]

The defining CSC property: the backbone must carry the customer carrier's **labels**, so labels must be exchanged on the PE-CE (CSC) link.

- [ ] **Task 2.1 [3 points]** — Enable **label exchange on the CSC link** using ONE of:
  - **MPLS LDP** inside VRF CSC65300 on R12 ↔ global LDP on R21 (`mpls ldp` on the CSC interface both sides), OR
  - **BGP-LU** (labeled unicast) across the CSC link (eBGP IPv4 `+ label` between R12-VRF and R21).
  Document which you chose and why. (BGP-LU is the common CSC-CE/CSC-PE choice; LDP is the simpler "CSC with LDP" variant.)
- [ ] **Task 2.2 [1 point]** — Ensure the customer carrier's **PE loopbacks** (R20/R22/R23 /32s and R21 /32) are the prefixes that receive labels across the CSC link (these are what the customer carrier needs end-to-end LSPs for — NOT its customer VPN routes).
- [ ] **Task 2.3 [1 point]** — Confirm labels are being allocated/received for those /32s on both ends of the CSC link.
- [ ] **Task 2.4 [0 points — verification]** — Confirm the backbone does NOT carry the customer carrier's internal customer routes — only its labelled infrastructure /32s.

**Verify / Snapshot**
- [ ] `show mpls ldp neighbor vrf CSC65300` (R12) OR `show bgp vrf CSC65300 ipv4 labeled-unicast`
- [ ] `show mpls forwarding vrf CSC65300` (R12) — labels for AS65300 /32s
- [ ] `show mpls forwarding` (R21) — labels toward the backbone for its remote /32s
- [ ] **Snapshot:** `WB22-02-csc-labels`

---

## Section 3 — Backbone Transport of Customer-Carrier Labels [4 points]

- [ ] **Task 3.1 [2 points]** — On the backbone, redistribute / carry the VRF CSC65300 labelled /32s between CSC-PEs via **VPNv4** (if the customer carrier attaches at two backbone PEs) OR confirm single-sided transport. Ensure the backbone imposes its own transport label UNDER the customer carrier's label (label stacking: backbone transport label + CSC VPN label + customer-carrier label).
- [ ] **Task 3.2 [1 point]** — Confirm the **three-level (or more) label stack** on backbone ingress: outer = backbone IGP/transport label, middle = CSC VRF/VPN label, inner = customer carrier's own label.
- [ ] **Task 3.3 [1 point]** — Confirm the customer carrier (AS 65300) sees an **end-to-end LSP** between its own PEs that transparently crosses the backbone.
- [ ] **Task 3.4 [0 points — verification]** — Confirm the backbone P routers never learn AS 65300 /32s in the global table.

**Verify / Snapshot**
- [ ] `show mpls forwarding vrf CSC65300` (R12) — label stack
- [ ] `show bgp vpnv4 unicast vrf CSC65300` — labelled /32s (if dual-attached)
- [ ] On AS65300 PE: `show mpls forwarding` shows a working LSP to a remote AS65300 PE across the backbone
- [ ] **Snapshot:** `WB22-03-backbone-transport`

---

## Section 4 — Hierarchical VPN (customer-of-customer) [4 points]

Optional-but-core CSC payload: AS 65300 runs its OWN L3VPN for its customers, tunneled over the backbone.

- [ ] **Task 4.1 [2 points]** — On AS 65300 PEs (e.g. R22 and R23), stand up a **customer VPN** (a VRF for an end customer, VPNv4 between R22/R23 via RR R30) that relies on the end-to-end LSP from Section 3 to transit the backbone.
- [ ] **Task 4.2 [1 point]** — Confirm the end customer's VPNv4 routes resolve over the LSP that crosses the backbone (the backbone never sees these VPN routes — only the AS65300 PE /32s).
- [ ] **Task 4.3 [1 point]** — Confirm end-to-end reachability for the end customer across the full hierarchy (end-CE → AS65300 PE → backbone AS65200 → AS65300 PE → end-CE).
- [ ] **Task 4.4 [0 points — verification]** — Trace the label stack depth at the deepest point.

**Verify / Snapshot**
- [ ] On AS65300 PE: `show bgp vpnv4 unicast vrf <endcust>` — resolves via remote PE /32 over the LSP
- [ ] `traceroute vrf <endcust>` — end-to-end across the hierarchy
- [ ] **Snapshot:** `WB22-04-hier-vpn`

---

## Section 5 — End-to-End Label Path Verification [4 points]

- [ ] **Task 5.1 [2 points]** — Walk the **complete label path** hop-by-hop from an AS65300 ingress PE to an AS65300 egress PE: record the labels imposed/swapped/popped at each node (AS65300 PE → R21 CSC-CE → R12 CSC-PE → backbone P → R12/other CSC-PE → R21-equivalent → egress PE). Document PHP points.
- [ ] **Task 5.2 [1 point]** — Use `traceroute mpls` / LSP ping to validate the LSP across the CSC boundary.
- [ ] **Task 5.3 [1 point]** — Confirm the label stack matches your Section 3 prediction (count the labels at the backbone core).
- [ ] **Task 5.4 [0 points — verification]** — Confirm no label gaps (every hop has a valid in/out label).

**Verify / Snapshot**
- [ ] `traceroute mpls ipv4 <AS65300-PE-lo>/32` from the CSC-CE/PE
- [ ] `show mpls forwarding` captured at each hop and reconciled into one path diagram (in notes)
- [ ] `ping mpls ipv4 <remote /32>` — success
- [ ] **Snapshot:** `WB22-05-label-path`

---

## Section 6 — Troubleshooting Drills [4 points]

- [ ] **Drill 6.1 — No labels on CSC link [2 points]:** forget to enable LDP/BGP-LU label exchange on the CSC link (plain IP only). Confirm the customer carrier's LSP breaks (reachability may exist but no end-to-end label path). Diagnose with `show mpls forwarding vrf` and fix.
- [ ] **Drill 6.2 — Backbone leaks AS65300 routes [1 point]:** mis-redistribute AS65300 /32s into the backbone global table. Confirm the hierarchy/isolation is violated. Diagnose and restrict to the VRF.
- [ ] **Drill 6.3 — Broken label stack [1 point]:** disable label imposition at one CSC-PE. Confirm the stack is incomplete and traffic blackholes mid-path. Diagnose from the hop-by-hop LFIB and restore.
- [ ] **Snapshot:** `WB22-06-tshoot`

**Required show commands (fluency):**
- [ ] `show mpls forwarding [vrf CSC65300]`
- [ ] `show mpls ldp neighbor vrf CSC65300` / `show bgp vrf CSC65300 ipv4 labeled-unicast`
- [ ] `show bgp vpnv4 unicast vrf CSC65300`
- [ ] `traceroute mpls ipv4 <prefix>` / `ping mpls ipv4 <prefix>`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: CSC-PE VRF + CSC-CE setup | 4 | Intermediate: VRF on CSC link |
| Section 2: Label distribution on CSC link | 5 | CCIE-level: LDP-in-VRF or BGP-LU label exchange |
| Section 3: Backbone transport of labels | 4 | CCIE-level: multi-level label stacking |
| Section 4: Hierarchical VPN (cust-of-cust) | 4 | CCIE-level: AS65300 L3VPN over backbone |
| Section 5: End-to-end label path | 4 | CCIE-level: hop-by-hop label walk |
| Section 6: Troubleshooting Drills | 4 | CCIE-level: break-and-fix |
| **Total** | **25** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
22-25 points:  CCIE ready for this topic
16-21 points:  Strong, minor gaps to review
10-15 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] CSC-PE (R12) VRF CSC65300 + CSC-CE (R21) on the R12↔R21 link
- [ ] Label exchange (LDP-in-VRF or BGP-LU) carrying AS65300 PE /32s across the CSC link
- [ ] Backbone transports the customer-carrier labels with correct label stacking
- [ ] Customer-of-customer L3VPN running over the end-to-end LSP
- [ ] End-to-end label path verified hop-by-hop with traceroute/ping mpls
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB22-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: CSC-PE VRF + CSC-CE** | **4** | | |
| 1.1 VRF CSC65300 on R12 | 2 | | |
| 1.2 CSC-CE on R21 (global) | 1 | | |
| 1.3 CSC link reachability | 1 | | |
| **Section 2: Label distribution** | **5** | | |
| 2.1 LDP-in-VRF or BGP-LU | 3 | | |
| 2.2 Carry AS65300 /32s | 1 | | |
| 2.3 Labels both ends | 1 | | |
| **Section 3: Backbone transport** | **4** | | |
| 3.1 VPNv4 / transport + stacking | 2 | | |
| 3.2 Multi-level label stack | 1 | | |
| 3.3 End-to-end LSP | 1 | | |
| **Section 4: Hierarchical VPN** | **4** | | |
| 4.1 AS65300 customer VPN | 2 | | |
| 4.2 Resolves over LSP | 1 | | |
| 4.3 End-to-end reachability | 1 | | |
| **Section 5: Label path** | **4** | | |
| 5.1 Hop-by-hop label walk | 2 | | |
| 5.2 traceroute/ping mpls | 1 | | |
| 5.3 Stack matches prediction | 1 | | |
| **Section 6: Troubleshooting** | **4** | | |
| 6.1 No labels on CSC link | 2 | | |
| 6.2 Backbone leaks routes | 1 | | |
| 6.3 Broken label stack | 1 | | |
| **TOTAL** | **25** | | |
