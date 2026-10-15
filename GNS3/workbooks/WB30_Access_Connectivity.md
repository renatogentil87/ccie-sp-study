# WB30 — Access Connectivity (Q-in-Q, G.8032, MC-LAG)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Carrier Ethernet access + aggregation resiliency in AS 65200 (R13/R14/R16)
**Level:** CCIE SP
**Prerequisite:** WB06 (L2VPN/AToM), WB07 (VPLS), WB20 (EVPN) — L2 service plane understood

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). 802.1ad Q-in-Q EFPs, G.8032, ICCP/MC-LAG and STP-AG are
> **not** available on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:**
> Access-layer PEs **R14** (150.2.14.14) and **R16** (150.2.16.16); aggregation **R13** (150.2.13.13), AS65200 OSPF Area 0.
> Ring nodes R14↔R13↔R16 (ERP). MC-LAG peers R14/R16 dual-home a CE (DHD). Backbone/ICCP link R13↔R14 (20.13.14.0/24),
> R13↔R16 (20.13.16.0/24), R14↔R19 (20.14.19.0/24). Links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm AS 65200 OSPF Area 0 converged (R11-R16, R19) and loopbacks reachable
- [ ] Confirm the access/aggregation links R13-R14-R16 are up at L1/L2
- [ ] Confirm a dual-homed CE (DHD) attachment exists toward R14 and R16
- [ ] **Snapshot:** `WB30-00-baseline`

---

## Section 1 — IEEE 802.1ad Q-in-Q (Provider Bridging) [4 points]

Stack a provider S-VLAN over customer C-VLANs on the R14 access EFP.

- [ ] **Task 1.1 [1 point]** — On an R14 access sub-interface, configure **dot1ad (S-VLAN) encapsulation** with an inner **dot1q (C-VLAN)** — `encapsulation dot1ad <s-vlan> dot1q <c-vlan>` (l2transport EFP).
- [ ] **Task 1.2 [1 point]** — **Map the customer C-VLAN to the provider S-VLAN** and bind the EFP into the L2 service (bridge-domain / xconnect) toward the core.
- [ ] **Task 1.3 [1 point]** — Configure the **rewrite operations** (`rewrite ingress tag push dot1ad <svlan>`, pop, or translate 1-to-1 / 2-to-1) so tagging is correct ingress and egress.
- [ ] **Task 1.4 [1 point]** — Verify **Q-in-Q tagging** end-to-end (double tag preserved across the provider domain) and correct EFP tag matching.
- [ ] **Task 1.5 [0 points — verification]** — Confirm the service only matches the intended S/C-VLAN pair (no VLAN leaking).

**Verify / Snapshot**
- [ ] `show interfaces <efp> | include encapsulation` — dot1ad + dot1q
- [ ] `show l2vpn bridge-domain detail` / `show l2vpn xconnect` — EFP member + rewrite
- [ ] `show ethernet trunk` / packet check — double tag across core
- [ ] **Snapshot:** `WB30-01-qinq`

---

## Section 2 — ITU G.8032 Ethernet Ring Protection (ERP) [5 points]

Protect a 3-node Ethernet ring (R14–R13–R16) with G.8032v2.

- [ ] **Task 2.1 [1 point]** — Configure the **G.8032 ring instance** on all three nodes (R14, R13, R16): ring id, two ring-port interfaces per node, `ethernet ring g8032 <name>`.
- [ ] **Task 2.2 [1 point]** — Define the **APS channel (R-APS) VLAN** and the **data VLAN(s)** carried on the ring; associate the instance's `aps-channel` with the control VLAN.
- [ ] **Task 2.3 [1 point]** — Designate the **RPL owner** and **RPL neighbor**, placing the ring protection link block on the owner's RPL port (normal state = RPL blocked).
- [ ] **Task 2.4 [1 point]** — Confirm **ring-port blocking behavior**: in the idle state the RPL is blocked (loop-free); confirm the forwarding/blocking roles of each ring port.
- [ ] **Task 2.5 [1 point]** — Trigger a **ring link failure** and confirm **switchover** (RPL unblocks, R-APS SF signaling) with sub-second data recovery, then recovery/revertive behavior on restore.
- [ ] **Task 2.6 [0 points — verification]** — Confirm **FDB flush** occurs on the ring topology change (MAC table relearned on the new path).

**Verify / Snapshot**
- [ ] `show ethernet ring g8032 <name> detail` — state (Idle/Protection), RPL owner/neighbor
- [ ] `show ethernet ring g8032 status` — port block/forward roles
- [ ] On failure: RPL unblocks, R-APS(SF) signaled, FDB flushed
- [ ] **Snapshot:** `WB30-02-g8032`

---

## Section 3 — MC-LAG (Multi-Chassis Link Aggregation) [5 points]

Dual-home a CE to R14 and R16 with a single LACP bundle via ICCP.

- [ ] **Task 3.1 [1 point]** — Configure **ICCP** between R14 and R16: `redundancy iccp group <id>`, mLACP, member neighbor (peer loopback), and the backbone interface between the peers.
- [ ] **Task 3.2 [1 point]** — Configure the **Bundle-Ether** interface toward the dual-homed CE (DHD) on both PEs with **mLACP** so the CE sees one LACP partner.
- [ ] **Task 3.3 [1 point]** — Set the **ICCP redundancy group** parameters: shared **system MAC**, **system priority**, and port priority to control active/standby POA selection.
- [ ] **Task 3.4 [1 point]** — Configure the **backbone (core/ICCP) link** between the MC-LAG peers so BUM/sync traffic and the mLACP state exchange flow between R14 and R16.
- [ ] **Task 3.5 [1 point]** — Perform a **failover test**: shut the active POA's bundle; confirm the standby POA becomes active and the CE bundle stays up (minimal loss).
- [ ] **Task 3.6 [0 points — verification]** — Confirm the CE sees a single aggregated bundle (consistent system-id) across both PEs.

**Verify / Snapshot**
- [ ] `show bundle Bundle-Ether<n>` — members, state, mLACP
- [ ] `show iccp group <id>` / `show redundancy iccp` — peers UP, RG state
- [ ] `show lacp` — system-id/priority consistent across POAs
- [ ] Failover: active POA down → standby active, CE bundle up
- [ ] **Snapshot:** `WB30-03-mclag`

---

## Section 4 — Spanning Tree Access Gateway (MST-AG, PVST-AG) Concepts [2 points]

- [ ] **Task 4.1 [1 point]** — Document **MST-AG**: the PE runs **MST** toward access switches and maps the access STP region into the VPLS/EVPN service; describe BPDU handling and root placement.
- [ ] **Task 4.2 [1 point]** — Document **PVST-AG**: the PE runs **PVST+** toward the access and **translates** per-VLAN STP into the provider domain; describe when PVST-AG is required.
- [ ] **Task 4.3 [0 points — verification]** — Write the **design rationale**: when to choose MST-AG vs PVST-AG (vendor interop, per-VLAN load-balancing, scaling), and how STP-AG interacts with G.8032/MC-LAG (avoid double protection loops).

**Verify / Snapshot**
- [ ] Notes capture BPDU flow, root bridge placement, and provider/customer STP boundary
- [ ] **Snapshot:** `WB30-04-stp-ag`

---

## Section 5 — Verification & Troubleshooting Drills [3 points]

- [ ] **Drill 5.1 — Q-in-Q tag mismatch [1 point]:** misconfigure the rewrite (wrong S-VLAN push). Confirm the service drops / mismatches; diagnose via `show l2vpn bridge-domain` and the EFP encapsulation, then fix.
- [ ] **Drill 5.2 — G.8032 dual RPL owner [1 point]:** misconfigure two RPL owners on the ring. Observe the loop / incorrect blocking; diagnose with `show ethernet ring g8032 detail` and correct to one owner + neighbor.
- [ ] **Drill 5.3 — MC-LAG system-id mismatch [1 point]:** set different system MAC/priority on the two POAs. Confirm the CE bundle splits (two partners); diagnose via `show lacp` / `show bundle` and align the ICCP RG.
- [ ] **Snapshot:** `WB30-05-tshoot`

**Required show commands (fluency):**
- [ ] `show l2vpn bridge-domain detail` / `show l2vpn xconnect`
- [ ] `show interfaces <efp>` (encapsulation / rewrite)
- [ ] `show ethernet ring g8032 detail`
- [ ] `show bundle` / `show iccp group` / `show lacp`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: 802.1ad Q-in-Q | 4 | Intermediate: EFP, S/C-VLAN, rewrite |
| Section 2: G.8032 ERP | 5 | CCIE-level: ring, RPL, switchover, FDB flush |
| Section 3: MC-LAG | 5 | CCIE-level: ICCP, mLACP, failover |
| Section 4: STP-AG concepts | 2 | Standard: MST-AG / PVST-AG design |
| Section 5: Troubleshooting Drills | 3 | CCIE-level: break-and-fix |
| **Total** | **19** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
17-19 points:  CCIE ready for this topic
13-16 points:  Strong, minor gaps to review
9-12 points:   Needs more practice
Below 9:       Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] 802.1ad Q-in-Q EFP with S/C-VLAN mapping and rewrite verified end-to-end
- [ ] G.8032 ring on R14/R13/R16 with RPL owner/neighbor, switchover + FDB flush proven
- [ ] MC-LAG via ICCP/mLACP dual-homing a CE, failover tested
- [ ] MST-AG / PVST-AG design rationale documented
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB30-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Q-in-Q** | **4** | | |
| 1.1 dot1ad encapsulation | 1 | | |
| 1.2 C-VLAN → S-VLAN map | 1 | | |
| 1.3 Rewrite operations | 1 | | |
| 1.4 Double-tag verified | 1 | | |
| **Section 2: G.8032 ERP** | **5** | | |
| 2.1 Ring instance | 1 | | |
| 2.2 APS channel VLAN | 1 | | |
| 2.3 RPL owner/neighbor | 1 | | |
| 2.4 Ring-port blocking | 1 | | |
| 2.5 Switchover test | 1 | | |
| **Section 3: MC-LAG** | **5** | | |
| 3.1 ICCP config | 1 | | |
| 3.2 Bundle-Ether to DHD | 1 | | |
| 3.3 RG system MAC/priority | 1 | | |
| 3.4 Backbone link | 1 | | |
| 3.5 Failover test | 1 | | |
| **Section 4: STP-AG concepts** | **2** | | |
| 4.1 MST-AG documented | 1 | | |
| 4.2 PVST-AG documented | 1 | | |
| **Section 5: Troubleshooting** | **3** | | |
| 5.1 Q-in-Q tag mismatch | 1 | | |
| 5.2 G.8032 dual RPL owner | 1 | | |
| 5.3 MC-LAG system-id mismatch | 1 | | |
| **TOTAL** | **19** | | |
