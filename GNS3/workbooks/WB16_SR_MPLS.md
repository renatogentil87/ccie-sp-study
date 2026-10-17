# WB16 — Segment Routing MPLS (SR-MPLS)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** SR-MPLS within AS 65100 (IS-IS L2 core, R1-R8)
**Level:** CCIE SP
**Prerequisite:** WB00-WB04 equivalents on XR (IS-IS L2 converged on R1-R8, Loopback0 reachable)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). SR-MPLS is **not** supported on the GNS3 7200/IOS 15.2 lab.
> Map the AS 65100 roles (R31/R2 PE, R3/R4 P, R5/R6 ASBR, R7/R8 RR) onto XR nodes.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **AS 65100 Loopback0 reference (SR node planning):**
> R1 150.1.31.31 · R2 150.1.2.2 · R3 150.1.3.3 · R4 150.1.4.4 · R5 150.1.5.5 · R6 150.1.6.6 · R7 150.1.7.7 · R8 150.1.8.8
> IS-IS area 49.0001, L2-only. Core links per `00_topology_reference.md` (AS 65100 Core).

---

## Pre-flight

- [ ] Confirm AS 65100 IS-IS L2 is fully converged (`show isis neighbors`, all core adjacencies UP)
- [ ] Confirm all Loopback0 /32s are in the IS-IS RIB (`show route isis`)
- [ ] Confirm MPLS is NOT yet programmed by SR (`show mpls forwarding` — empty or LDP-only)
- [ ] **Snapshot:** `WB16-00-baseline`

---

## Section 1 — Enable Segment Routing + SRGB/SRLB [3 points]

Core: R1-R8. Default SRGB **16000-23999**, SRLB **25000-26000** (set explicitly).

- [ ] **Task 1.1 [1 point]** — Under `segment-routing global-block 16000 23999` and `local-block 25000 26000`, define the SRGB and SRLB on every node (R1-R8). Set them identically fleet-wide so prefix-SIDs are globally consistent.
- [ ] **Task 1.2 [1 point]** — Enable segment routing MPLS under IS-IS: `router isis <tag>` → `address-family ipv4 unicast` → `segment-routing mpls`.
- [ ] **Task 1.3 [1 point]** — Confirm the SRGB/SRLB are advertised in IS-IS and label ranges are reserved in the LSD.
- [ ] **Task 1.4 [0 points — verification]** — Verify SR is enabled and label blocks reserved.

**Verify / Snapshot**
- [ ] `show segment-routing mpls` — SR enabled
- [ ] `show mpls label table detail` — SRGB (16000-23999) + SRLB (25000-26000) reserved
- [ ] `show isis database verbose <node>` — SR Capability TLV with SRGB
- [ ] **Snapshot:** `WB16-01-sr-enabled`

---

## Section 2 — Prefix-SID / Node-SID Allocation [4 points]

Allocate an absolute prefix-SID per node (index = router number for clarity: R31→1 … R8→8, i.e. labels 16001-16008).

- [ ] **Task 2.1 [2 points]** — Under IS-IS, on each node's Loopback0 interface, assign a **prefix-SID index** (`segment-routing prefix-sid index <N>`). Use index N = router number (R1=1 … R8=8). Mark it as a **node-SID** (n-flag set — this is the default for a /32 loopback).
- [ ] **Task 2.2 [1 point]** — Confirm each node computes the global label (SRGB base 16000 + index) and installs it in CEF/LFIB.
- [ ] **Task 2.3 [1 point]** — Confirm adjacency-SIDs are auto-allocated from the SRLB (dynamic, per-adjacency) on every core link.
- [ ] **Task 2.4 [0 points — verification]** — Verify end-to-end SR label switching from R1 to R8 (no LDP involved).

**Verify / Snapshot**
- [ ] `show isis segment-routing prefix-sid` — index + absolute label per node
- [ ] `show mpls forwarding` — SR labels (16001-16008) with correct outgoing label (swap/pop)
- [ ] `show cef 150.1.8.8/32 detail` — labelled path toward R8
- [ ] `show isis segment-routing label adjacency` — adjacency-SIDs from SRLB
- [ ] **Snapshot:** `WB16-02-prefix-sid`

---

## Section 3 — Anycast-SID on R31 + R2 [4 points]

Deploy a shared **anycast** service address on the R31/R2 PE pair so traffic can land on whichever is closer.

- [ ] **Task 3.1 [1 point]** — On both R31 and R2, create a new **Loopback99** with the SAME anycast IP (e.g. 150.1.99.99/32).
- [ ] **Task 3.2 [2 points]** — Advertise Loopback99 into IS-IS on both nodes with the SAME prefix-SID **index/absolute = 99** (`segment-routing prefix-sid absolute 16099` or index 99), and set the **n-flag-clear** flag (anycast = NOT a node-SID, so clear the node flag).
- [ ] **Task 3.3 [0 points — verification]** — From a remote node (R8), confirm two equal-cost paths resolve the anycast SID (ECMP to both R31 and R2), and that failing one PE keeps the SID reachable via the other.
- [ ] **Task 3.4 [1 point]** — Explain in the attempt notes WHY the n-flag must be cleared for anycast (PHP / node-SID semantics vs. shared prefix).

**Verify / Snapshot**
- [ ] `show isis segment-routing prefix-sid | include 16099|99` — both R1 & R2 advertise, n-flag clear
- [ ] `show cef 150.1.99.99/32 detail` — ECMP toward R31 and R2
- [ ] After shutting R1's Loopback99/node: SID still reachable via R2
- [ ] **Snapshot:** `WB16-03-anycast`

---

## Section 4 — TI-LFA on All Interfaces [4 points]

Topology-Independent Loop-Free Alternate: sub-50ms protection using SR repair paths.

- [ ] **Task 4.1 [2 points]** — Under IS-IS, enable **TI-LFA** on every core interface of R1-R8 (`fast-reroute per-prefix` + `fast-reroute per-prefix ti-lfa` under the interface address-family). Enable for the IPv4 unicast topology.
- [ ] **Task 4.2 [1 point]** — Confirm each protected prefix now has a **backup** path computed with a repair SID list (not just a plain LFA).
- [ ] **Task 4.3 [1 point]** — Trigger a core link failure and confirm the backup path (repair list) is used; reason about restoration time (XR programs the backup pre-failure).
- [ ] **Task 4.4 [0 points — verification]** — Confirm node-protecting TI-LFA where a P-node has a viable alternate.

**Verify / Snapshot**
- [ ] `show isis fast-reroute summary` — protected / unprotected counts
- [ ] `show route 150.1.8.8/32 detail` — backup next-hop + repair label stack
- [ ] `show cef 150.1.8.8/32 detail` — primary + backup (repair) paths
- [ ] `show isis fast-reroute 150.1.8.8/32` — TI-LFA repair path & SID list
- [ ] **Snapshot:** `WB16-04-tilfa`

---

## Section 5 — Microloop Avoidance [3 points]

- [ ] **Task 5.1 [2 points]** — Under IS-IS, enable **microloop avoidance segment-routing** (`microloop avoidance segment-routing`) and set the RIB-update delay (`microloop avoidance rib-update-delay <ms>`) on the core nodes.
- [ ] **Task 5.2 [1 point]** — Explain in notes how SR microloop avoidance installs an explicit (loop-free) SID path during convergence, then reverts to the post-convergence path after the delay timer.
- [ ] **Task 5.3 [0 points — verification]** — Trigger a metric change / link event and confirm the convergence behavior (no transient loop) via the microloop avoidance counters.

**Verify / Snapshot**
- [ ] `show isis` configuration confirms microloop avoidance enabled
- [ ] `show segment-routing local-block inconsistencies` — none
- [ ] Event-driven: microloop avoidance activity on convergence
- [ ] **Snapshot:** `WB16-05-microloop`

---

## Section 6 — LDP-to-SR Migration (Mapping Server) [5 points]

Interwork a legacy LDP island with SR using a **Segment Routing Mapping Server (SRMS)**.

- [ ] **Task 6.1 [1 point]** — Pick two nodes (e.g. R3, R4) to retain **LDP** (simulate legacy island). Keep R31/R2/R5-R8 SR-enabled.
- [ ] **Task 6.2 [2 points]** — Configure an **SRMS** on R7 (RR): `segment-routing mapping-server prefix-sid-map address-family ipv4` mapping the LDP-only loopbacks (e.g. 150.1.3.3/32 → index 3, 150.1.4.4/32 → index 4). Advertise the mappings into IS-IS (`segment-routing prefix-sid-map advertise-local`).
- [ ] **Task 6.3 [1 point]** — Enable **SR/LDP interworking** so SR LSPs stitch to LDP LSPs across the island (`segment-routing mpls` + confirm SR-labelled-to-LDP imposition / stitching on the border nodes).
- [ ] **Task 6.4 [1 point]** — Confirm end-to-end label switching R31→R4 traverses an SR segment then an LDP segment (label stitching), proving coexistence.
- [ ] **Task 6.5 [0 points — verification]** — Confirm the mapping server entries are received fleet-wide.

**Verify / Snapshot**
- [ ] `show segment-routing mapping-server prefix-sid-map ipv4` — mappings on SRMS
- [ ] `show isis segment-routing prefix-sid-map active-policy` — received mappings
- [ ] `show mpls forwarding` on border node — SR label swapped to LDP label (stitch)
- [ ] `traceroute mpls` / `traceroute 150.1.4.4` — crosses SR→LDP boundary
- [ ] **Snapshot:** `WB16-06-ldp-sr-migration`

---

## Section 7 — Verification & Troubleshooting Drills [5 points]

- [ ] **Drill 7.1 — SRGB mismatch [2 points]:** change R5's SRGB base to a different value. Observe prefix-SID label inconsistency / errors fleet-wide. Diagnose via `show segment-routing mpls` and SRGB-inconsistency output, then fix.
- [ ] **Drill 7.2 — Missing prefix-SID [1 point]:** remove the prefix-SID from R8's Loopback0. Confirm R8 falls back to LDP/plain IP (no SR label) and SR LSPs toward R8 break. Diagnose and restore.
- [ ] **Drill 7.3 — TI-LFA not protecting [2 points]:** disable TI-LFA on one interface. Confirm the affected prefix shows "unprotected" and reason why. Re-enable.
- [ ] **Snapshot:** `WB16-07-tshoot`

**Required show commands (fluency):**
- [ ] `show isis segment-routing`
- [ ] `show cef`
- [ ] `show mpls forwarding`
- [ ] `show segment-routing mpls`
- [ ] `show isis fast-reroute summary`

---

## Section 8 — Segment Routing on OSPF [4 points]

Enable SR-MPLS in the **AS 65200 OSPF domain** (R11-R16, R19) — the OSPF counterpart to the IS-IS SR deployment above.

- [ ] **Task 8.1 [2 points]** — Enable SR-MPLS on the OSPF domain: `router ospf <proc>` → `segment-routing mpls`, and assign a **prefix-SID** under each node's Loopback0 (`prefix-sid index <N>` under the OSPF interface / `area` loopback). Use **prefix-SID index = last octet of the loopback** (R11→11 … R16→16, R19→19). Keep the SRGB consistent with the fleet (16000-23999).
- [ ] **Task 8.2 [1 point]** — Verify: `show ospf segment-routing`, and `show mpls forwarding-table` shows SR labels (**16000 + index**, e.g. R11 = 16011).
- [ ] **Task 8.3 [1 point]** — Enable **TI-LFA** under the OSPF interfaces (`fast-reroute per-prefix ti-lfa`). Verify backup paths are computed.

**Verify / Snapshot**
- [ ] `show ospf segment-routing` — SR enabled, SRGB
- [ ] `show ospf segment-routing prefix-sid` — index + absolute label per node
- [ ] `show mpls forwarding-table` — SR labels 16011-16016, 16019
- [ ] `show ospf fast-reroute` / `show route <loopback> detail` — TI-LFA backup
- [ ] **Snapshot:** `WB16-08-sr-ospf`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Enable SR + SRGB/SRLB | 3 | Standard: enable SR, label blocks |
| Section 2: Prefix-SID / Node-SID | 4 | Intermediate: index allocation, adjacency-SID |
| Section 3: Anycast-SID (R1+R2) | 4 | CCIE-level: shared SID, n-flag-clear |
| Section 4: TI-LFA | 4 | CCIE-level: per-prefix repair SID lists |
| Section 5: Microloop Avoidance | 3 | Intermediate: convergence behavior |
| Section 6: LDP-to-SR Migration (SRMS) | 5 | CCIE-level: mapping server + stitching |
| Section 7: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
| Section 8: Segment Routing on OSPF | 4 | CCIE-level: SR + prefix-SID + TI-LFA on OSPF |
| **Total** | **32** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
28-32 points:  CCIE ready for this topic
21-27 points:  Strong, minor gaps to review
14-20 points:  Needs more practice
Below 14:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] SR-MPLS enabled with explicit SRGB/SRLB across R1-R8
- [ ] Prefix-SIDs + adjacency-SIDs allocated and label-switching end-to-end
- [ ] Anycast-SID on R31/R2 (n-flag-clear) with ECMP + failover
- [ ] TI-LFA protecting all core interfaces
- [ ] Microloop avoidance enabled and reasoned
- [ ] LDP↔SR interworking via mapping server proven with stitched LSP
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] SR-MPLS on OSPF (AS 65200) with prefix-SIDs + TI-LFA
- [ ] **Final snapshot:** `WB16-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Enable SR + SRGB/SRLB** | **3** | | |
| 1.1 SRGB/SRLB | 1 | | |
| 1.2 SR under IS-IS | 1 | | |
| 1.3 Advertised/reserved | 1 | | |
| **Section 2: Prefix-SID** | **4** | | |
| 2.1 Prefix-SID index | 2 | | |
| 2.2 Global label install | 1 | | |
| 2.3 Adjacency-SID | 1 | | |
| **Section 3: Anycast-SID** | **4** | | |
| 3.1 Loopback99 both | 1 | | |
| 3.2 Shared SID n-flag-clear | 2 | | |
| 3.4 Explanation | 1 | | |
| **Section 4: TI-LFA** | **4** | | |
| 4.1 Enable TI-LFA | 2 | | |
| 4.2 Backup repair list | 1 | | |
| 4.3 Failure test | 1 | | |
| **Section 5: Microloop Avoidance** | **3** | | |
| 5.1 Enable + delay | 2 | | |
| 5.2 Explanation | 1 | | |
| **Section 6: LDP-to-SR Migration** | **5** | | |
| 6.1 LDP island | 1 | | |
| 6.2 SRMS config | 2 | | |
| 6.3 SR/LDP interworking | 1 | | |
| 6.4 Stitched LSP proof | 1 | | |
| **Section 7: Troubleshooting** | **5** | | |
| 7.1 SRGB mismatch | 2 | | |
| 7.2 Missing prefix-SID | 1 | | |
| 7.3 TI-LFA not protecting | 2 | | |
| **Section 8: SR on OSPF** | **4** | | |
| 8.1 SR + prefix-SID (OSPF) | 2 | | |
| 8.2 Verify SR labels | 1 | | |
| 8.3 TI-LFA on OSPF | 1 | | |
| **TOTAL** | **32** | | |
