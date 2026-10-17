# WB25 — BGP on IOS-XR (Unified MPLS)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** BGP across all three ASes (65100 / 65200 / 65300) — Unified MPLS / Seamless MPLS
**Level:** CCIE SP
**Prerequisite:** WB23 (IS-IS XR), WB24 (OSPF XR), IGP converged in each AS; MPLS transport (LDP or SR) up

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). Neighbor-groups, labeled-unicast AF, RPL route-policy, and
> Inter-AS Option C are **not** available on the GNS3 7200/IOS 15.2 lab (not Dynamips).
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference (all ASes):**
> **AS 65100** (IS-IS L2): PE R31/R2, P R3/R4, ASBR R5/R6, RR R7/R8 — loopbacks 150.1.X.X
> **AS 65200** (OSPF): P R11/R13, ASBR R12, PE R14/R15/R16, RR R19 — loopbacks 150.2.X.X
> **AS 65300** (IS-IS L2): ASBR/PE R20/R21, PE R22/R23, RR R30 — loopbacks 150.3.X.X
> eBGP ASBR boundaries and inter-AS links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm each AS IGP is fully converged (loopbacks reachable intra-AS)
- [ ] Confirm MPLS transport LSPs exist intra-AS (`show mpls forwarding`)
- [ ] Confirm inter-AS physical links between ASBRs are up
- [ ] **Snapshot:** `WB25-00-baseline`

---

## Section 1 — iBGP with Neighbor-Groups [5 points]

- [ ] **Task 1.1 [2 points]** — On the RRs (R7/R8, R19, R30) define a **`neighbor-group`** carrying `remote-as` (own AS), `update-source Loopback0`, and the needed address-families (`ipv4 unicast`, `vpnv4 unicast`).
- [ ] **Task 1.2 [1 point]** — Under the AF in the neighbor-group, set **`route-reflector-client`** so clients inheriting the group become RR clients.
- [ ] **Task 1.3 [1 point]** — Point each client (PEs/ASBRs) at its RR via `use neighbor-group <name>`; confirm the group's parameters are inherited.
- [ ] **Task 1.4 [1 point]** — Confirm iBGP sessions come up for all configured AFs within each AS.
- [ ] **Task 1.5 [0 points — verification]** — Confirm RR reflects client routes to other clients (reflected routes carry ORIGINATOR_ID / CLUSTER_LIST).

**Verify / Snapshot**
- [ ] `show bgp neighbor-group <name> configuration` — inherited params
- [ ] `show bgp ipv4 unicast summary` / `show bgp vpnv4 unicast summary` — sessions UP
- [ ] `show bgp vpnv4 unicast <prefix>` — ORIGINATOR_ID / CLUSTER_LIST on reflected routes
- [ ] **Snapshot:** `WB25-01-ibgp-ng`

---

## Section 2 — eBGP Between ASBRs [3 points]

- [ ] **Task 2.1 [1 point]** — Configure **eBGP** between the ASBR pairs (AS65100↔AS65200 and AS65200↔AS65300) with per-neighbor `remote-as` and the appropriate address-families.
- [ ] **Task 2.2 [1 point]** — Enable `send-community extended` toward the eBGP peers and apply `next-hop-self` where required so the next-hop resolves inside the receiving AS.
- [ ] **Task 2.3 [1 point]** — Confirm eBGP sessions are UP and prefixes are exchanged across the AS boundary.
- [ ] **Task 2.4 [0 points — verification]** — Confirm AS_PATH shows the traversed ASes end-to-end.

**Verify / Snapshot**
- [ ] `show bgp ipv4 unicast summary` — eBGP neighbors UP
- [ ] `show bgp <prefix>` — AS_PATH across ASes
- [ ] **Snapshot:** `WB25-02-ebgp`

---

## Section 3 — BGP Labeled Unicast (BGP-LU) [5 points]

Build end-to-end LSPs across AS boundaries using `ipv4 labeled-unicast`.

- [ ] **Task 3.1 [2 points]** — Enable **`address-family ipv4 labeled-unicast`** on the relevant BGP sessions and set **`allocate-label all`** so loopbacks are advertised with labels.
- [ ] **Task 3.2 [1 point]** — Configure the P routers **R3 and R4 as inline RRs** for the labeled-unicast AF, applying an **`ibgp policy out enforce-modifications`** so next-hop/label rewrites are honored on reflected routes.
- [ ] **Task 3.3 [1 point]** — Apply **`next-hop-self`** at the ASBRs for the labeled-unicast AF so the label stack is re-originated at each AS edge.
- [ ] **Task 3.4 [1 point]** — Confirm an end-to-end labeled path (BGP-LU LSP) between a PE in one AS and a PE in another.
- [ ] **Task 3.5 [0 points — verification]** — Confirm labels are allocated and present in the forwarding table.

**Verify / Snapshot**
- [ ] `show bgp ipv4 labeled-unicast` — labeled prefixes + local labels
- [ ] `show mpls forwarding` — BGP-LU labels installed
- [ ] `show cef <remote PE loopback>` — labeled path across ASes
- [ ] `traceroute mpls` — end-to-end LSP
- [ ] **Snapshot:** `WB25-03-bgp-lu`

---

## Section 4 — VPNv4 + VPNv6 Address-Families [4 points]

- [ ] **Task 4.1 [2 points]** — Enable **`address-family vpnv4 unicast`** and **`address-family vpnv6 unicast`** on PE↔RR iBGP sessions; confirm the AFs negotiate.
- [ ] **Task 4.2 [1 point]** — On the ASBRs set **`retain route-target all`** so the ASBR keeps VPN routes it is not directly importing (needed for inter-AS transit).
- [ ] **Task 4.3 [1 point]** — Confirm `send-community extended` is on for the VPN AFs so RTs/attributes are carried.
- [ ] **Task 4.4 [0 points — verification]** — Confirm VPNv4/VPNv6 prefixes appear with their RTs on the RRs and ASBRs.

**Verify / Snapshot**
- [ ] `show bgp vpnv4 unicast summary` / `show bgp vpnv6 unicast summary`
- [ ] `show bgp vpnv4 unicast rd <rd>` — RT extended communities present
- [ ] `show bgp vpnv4 unicast` on ASBR — retained routes (retain RT all)
- [ ] **Snapshot:** `WB25-04-vpnv4-vpnv6`

---

## Section 5 — Inter-AS Option C [5 points]

- [ ] **Task 5.1 [2 points]** — Stitch BGP-LU LSPs across the ASBR boundaries (from Section 3) so PE loopbacks are reachable with labels end-to-end across all three ASes.
- [ ] **Task 5.2 [2 points]** — Build a **multihop VPNv4 eBGP** session **R7↔R30** (`ebgp-multihop`, `update-source Loopback0`) and set **next-hop unchanged** so the VPNv4 next-hop (the originating PE) is preserved across the backbone.
- [ ] **Task 5.3 [1 point]** — Verify an **end-to-end LSP** carrying a VPN prefix from an AS65100 PE to an AS65300 PE (label stack: transport + VPN).
- [ ] **Task 5.4 [0 points — verification]** — Confirm the VPNv4 next-hop is the remote PE loopback (unchanged) and resolves over the BGP-LU LSP.

**Verify / Snapshot**
- [ ] `show bgp vpnv4 unicast neighbor <R30> ` — multihop session UP
- [ ] `show bgp vpnv4 unicast <prefix>` — next-hop = originating PE
- [ ] `show cef vrf <vrf> <prefix>` — labeled path across ASes
- [ ] `traceroute mpls` / end-to-end VRF ping
- [ ] **Snapshot:** `WB25-05-option-c`

---

## Section 6 — BGP Path Manipulation (RPL) [5 points]

Use the IOS-XR **Routing Policy Language (RPL)**: `if/then/set`, sets, and attach-points.

- [ ] **Task 6.1 [2 points]** — Write a `route-policy` using **if/then/set** logic referencing an **`as-path-set`**, a **`community-set`**, and an **`extcommunity-set`** (RT-based match).
- [ ] **Task 6.2 [1 point]** — Set **LOCAL_PREF** inbound on one AS boundary to prefer a primary path; confirm best-path selection changes.
- [ ] **Task 6.3 [1 point]** — Apply **AS-PATH prepend** and **MED** outbound to influence inbound path selection from the neighbor AS; confirm the effect.
- [ ] **Task 6.4 [1 point]** — Set/match **communities** to tag and act on routes (e.g. no-export on a subset); confirm propagation behavior.
- [ ] **Task 6.5 [0 points — verification]** — Confirm each policy is attached (inbound/outbound) and taking effect.

**Verify / Snapshot**
- [ ] `show rpl route-policy <name>` — policy body
- [ ] `show bgp <prefix>` — LOCAL_PREF / MED / AS_PATH / communities applied
- [ ] `show bgp policy route-policy <name>` — attach points
- [ ] **Snapshot:** `WB25-06-path-manip`

---

## Section 7 — Verification & Troubleshooting Drills [3 points]

- [ ] **Drill 7.1 — Missing route-target [1 point]:** remove an RT import on a PE. Confirm the VPN prefix disappears from the VRF; diagnose via `show bgp vpnv4 unicast` and fix.
- [ ] **Drill 7.2 — Broken BGP-LU next-hop [1 point]:** remove `next-hop-self`/`allocate-label` on an ASBR. Confirm the end-to-end LSP breaks; diagnose and restore.
- [ ] **Drill 7.3 — RR without reflection [1 point]:** remove `route-reflector-client` on a RR. Confirm clients lose each other's routes; diagnose and restore.
- [ ] **Snapshot:** `WB25-07-tshoot`

**Required show commands (fluency):**
- [ ] `show bgp [ipv4 unicast | ipv4 labeled-unicast | vpnv4 unicast | vpnv6 unicast] summary`
- [ ] `show bgp <prefix>` / `show bgp vpnv4 unicast rd <rd> <prefix>`
- [ ] `show mpls forwarding` / `show cef`
- [ ] `show rpl route-policy <name>`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: iBGP neighbor-groups | 5 | Intermediate: group inheritance + RR client |
| Section 2: eBGP between ASBRs | 3 | Standard: per-neighbor eBGP |
| Section 3: BGP-LU | 5 | CCIE-level: labeled-unicast + inline RR |
| Section 4: VPNv4 + VPNv6 AFs | 4 | Intermediate: retain RT all |
| Section 5: Inter-AS Option C | 5 | CCIE-level: multihop VPNv4 + NH unchanged |
| Section 6: Path manipulation (RPL) | 5 | CCIE-level: RPL if/then/set + sets |
| Section 7: Troubleshooting Drills | 3 | CCIE-level: break-and-fix |
| **Total** | **30** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
26-30 points:  CCIE ready for this topic
20-25 points:  Strong, minor gaps to review
13-19 points:  Needs more practice
Below 13:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] iBGP neighbor-groups with RR clients UP in each AS
- [ ] eBGP between ASBRs exchanging prefixes with extended communities
- [ ] BGP-LU end-to-end LSPs with inline RRs (R3/R4) and ASBR next-hop-self
- [ ] VPNv4 + VPNv6 AFs with retain RT all on ASBRs
- [ ] Inter-AS Option C: multihop VPNv4 R7↔R30, next-hop unchanged, end-to-end LSP
- [ ] RPL path manipulation (LOCAL_PREF / prepend / MED / communities) verified
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB25-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: iBGP neighbor-groups** | **5** | | |
| 1.1 neighbor-group def | 2 | | |
| 1.2 route-reflector-client | 1 | | |
| 1.3 use neighbor-group | 1 | | |
| 1.4 Sessions UP | 1 | | |
| **Section 2: eBGP ASBRs** | **3** | | |
| 2.1 Per-neighbor eBGP | 1 | | |
| 2.2 ext-comm + NH-self | 1 | | |
| 2.3 Prefixes exchanged | 1 | | |
| **Section 3: BGP-LU** | **5** | | |
| 3.1 labeled-unicast + allocate-label | 2 | | |
| 3.2 Inline RR R3/R4 | 1 | | |
| 3.3 NH-self at ASBRs | 1 | | |
| 3.4 End-to-end LSP | 1 | | |
| **Section 4: VPNv4/VPNv6** | **4** | | |
| 4.1 VPNv4+VPNv6 AFs | 2 | | |
| 4.2 retain RT all | 1 | | |
| 4.3 send-community extended | 1 | | |
| **Section 5: Option C** | **5** | | |
| 5.1 BGP-LU stitched | 2 | | |
| 5.2 Multihop VPNv4 R7↔R30 | 2 | | |
| 5.3 End-to-end VPN LSP | 1 | | |
| **Section 6: Path manipulation** | **5** | | |
| 6.1 RPL + sets | 2 | | |
| 6.2 LOCAL_PREF | 1 | | |
| 6.3 Prepend + MED | 1 | | |
| 6.4 Communities | 1 | | |
| **Section 7: Troubleshooting** | **3** | | |
| 7.1 Missing RT | 1 | | |
| 7.2 Broken BGP-LU NH | 1 | | |
| 7.3 RR no reflection | 1 | | |
| **TOTAL** | **30** | | |
