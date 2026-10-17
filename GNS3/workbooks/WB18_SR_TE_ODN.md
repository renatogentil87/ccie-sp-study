# WB18 — SR-TE and On-Demand Next-hop (ODN)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** SR-TE policies + ODN; headends in AS 65100 (R1-R8) / AS 65300 (R20-R23); SR-PCE on R30
**Level:** CCIE SP
**Prerequisite:** WB16 (SR-MPLS: SRGB, prefix-SIDs, adjacency-SIDs, TI-LFA all working)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab.
> SR-TE/ODN/PCE are **not** supported on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: verify each section before advancing.
> **Snapshot** before/after each section.

> **Reference (SR-MPLS from WB16):** prefix-SID index = router number (R1=1 … R8=8, 16000+N).
> Headend examples: **R1** (150.1.31.31) → tailend **R8** (150.1.8.8) within AS 65100.
> **SR-PCE** runs on **R30** (150.3.30.30 — RR of AS 65300, reused as PCE).
> Core links + adjacency-SIDs per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm WB16 SR-MPLS is operational (prefix-SIDs + adjacency-SIDs installed)
- [ ] Confirm IS-IS TE metrics / delay metrics are populated (`show segment-routing traffic-eng topology summary`)
- [ ] Confirm BGP is UP where color steering will be tested
- [ ] **Snapshot:** `WB18-00-baseline`

---

## Section 1 — Dynamic SR Policy (TE metric) + Binding-SID [4 points]

Headend **R1**, endpoint **R8**.

- [ ] **Task 1.1 [2 points]** — On R1 create an SR policy to endpoint 150.1.8.8 with color (e.g. 100): `segment-routing traffic-eng policy <name>` → `color 100 end-point ipv4 150.1.8.8` → candidate-path `dynamic` with `metric type te`.
- [ ] **Task 1.2 [1 point]** — Assign an explicit **binding-SID** (BSID) to the policy from the SRLB (e.g. `binding-sid mpls 25100`).
- [ ] **Task 1.3 [1 point]** — Confirm the policy is UP and the headend computed a SID list minimizing the **TE metric** (not IGP metric).
- [ ] **Task 1.4 [0 points — verification]** — Confirm the BSID is installed and can be used to stitch into the policy.

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng policy color 100` — UP, dynamic, metric TE, SID list
- [ ] `show segment-routing traffic-eng policy name <name> detail` — binding-SID 25100
- [ ] `show mpls forwarding labels 25100` — BSID → policy SID list
- [ ] **Snapshot:** `WB18-01-dynamic-policy`

---

## Section 2 — Autoroute Include (steering) [3 points]

- [ ] **Task 2.1 [2 points]** — Add **autoroute include** to the R31→R8 policy so selected prefixes (e.g. 150.1.8.8/32 and prefixes behind R8) are steered into the policy via IGP shortcut (`autoroute include ipv4 <prefix>` or `autoroute include all` under the candidate-path/policy).
- [ ] **Task 2.2 [1 point]** — Optionally set an autoroute **metric** so the policy competes correctly with the native IGP path.
- [ ] **Task 2.3 [0 points — verification]** — Confirm the RIB now resolves the steered prefixes via the SR policy.

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng policy color 100 | include Autoroute`
- [ ] `show route 150.1.8.8/32` — via SR policy
- [ ] `traceroute 150.1.8.8` — follows the TE path
- [ ] **Snapshot:** `WB18-02-autoroute`

---

## Section 3 — BGP Color Community + Autosteering [4 points]

- [ ] **Task 3.1 [2 points]** — On the egress PE (R8), attach a **color extended community** (e.g. color 100) to advertised service routes (via route-policy on BGP, VPNv4 or IPv4). 
- [ ] **Task 3.2 [1 point]** — On the headend R1, enable BGP **automated steering** so routes carrying color 100 with next-hop = 150.1.8.8 automatically resolve onto the color-100 SR policy (no static mapping).
- [ ] **Task 3.3 [1 point]** — Confirm a service prefix with color 100 is installed with the SR policy as its forwarding path (color + next-hop → policy).
- [ ] **Task 3.4 [0 points — verification]** — Confirm removing the color community reverts the prefix to the native path.

**Verify / Snapshot**
- [ ] `show bgp <af> <prefix> detail` — color:100 extended community present
- [ ] `show segment-routing traffic-eng forwarding policy color 100` — carrying steered traffic
- [ ] `show cef <service-prefix> detail` — via SR policy BSID
- [ ] **Snapshot:** `WB18-03-color-autosteer`

---

## Section 4 — Explicit Segment-Lists (adjacency-SIDs) [4 points]

- [ ] **Task 4.1 [2 points]** — Define an **explicit segment-list** on R31 that pins a specific hop-by-hop path to R8 using **adjacency-SIDs** (from WB16) — e.g. force R31→R3→R4→R6→R5→R8 rather than the shortest path.
- [ ] **Task 4.2 [1 point]** — Create an SR policy (new color, e.g. 200) whose candidate-path uses the explicit segment-list with higher **preference** than any dynamic path.
- [ ] **Task 4.3 [1 point]** — Confirm the policy is UP and forwards over the exact pinned path (verify each adjacency-SID in the imposed label stack).
- [ ] **Task 4.4 [0 points — verification]** — Fail one adjacency in the list; confirm the explicit path goes down (no auto-reroute unless a lower-pref dynamic path exists).

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng policy color 200 detail` — explicit SID list
- [ ] `show mpls forwarding` — imposed adjacency-SID stack
- [ ] `traceroute` confirms the pinned hop sequence
- [ ] **Snapshot:** `WB18-04-explicit-adj-sid`

---

## Section 5 — Affinity Constraints (include / exclude) [4 points]

- [ ] **Task 5.1 [1 point]** — Define an **affinity-map** (names → bit positions) consistently fleet-wide (e.g. `red` bit 0, `blue` bit 1) under `segment-routing traffic-eng affinity-map`.
- [ ] **Task 5.2 [1 point]** — Assign **interface affinity** colors to selected core links (`affinity name red` / `blue` under the TE interface).
- [ ] **Task 5.3 [2 points]** — On a dynamic SR policy add affinity **constraints**: `constraints affinity exclude-any red` (avoid red links) and/or `include-all` / `include-any`. Prove the computed path avoids excluded-color links.
- [ ] **Task 5.4 [0 points — verification]** — Confirm the SID list changes when you flip a link's affinity color.

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng affinity-map`
- [ ] `show segment-routing traffic-eng topology | include Affinity` — link colors
- [ ] `show segment-routing traffic-eng policy color <c> detail` — constraints + resulting SID list
- [ ] **Snapshot:** `WB18-05-affinity`

---

## Section 6 — SR-PCE + On-Demand Next-hop (ODN) [6 points]

**SR-PCE on R30.** ODN with PCE-delegated computation and a **latency** metric.

- [ ] **Task 6.1 [2 points]** — Configure **SR-PCE** on R30: `pce` address ipv4 150.3.30.30, enable the PCE server. On headends configure a **PCC** (`segment-routing traffic-eng pcc` → `pce address ipv4 150.3.30.30`) and establish the PCEP session.
- [ ] **Task 6.2 [2 points]** — Configure an **ODN template** on the headend keyed on a color (e.g. color 300): `segment-routing traffic-eng on-demand color 300` with `dynamic` + **`metric type latency`** and **`pcep`** (PCE-delegated computation).
- [ ] **Task 6.3 [1 point]** — Advertise a service route colored 300 from a remote PE. Confirm the headend **on-demand** instantiates an SR policy to that endpoint computed by the PCE on the latency metric.
- [ ] **Task 6.4 [1 point]** — Confirm the PCE delegated the path (the policy shows PCE as the computation source) and that no policy exists until the colored route appears (on-demand lifecycle).
- [ ] **Task 6.5 [0 points — verification]** — Confirm PCEP session state and delegated LSP on the PCE.

**Verify / Snapshot**
- [ ] `show pce ipv4 peer` / `show pce lsp` (on R30) — PCC sessions + delegated LSPs
- [ ] `show segment-routing traffic-eng pcc ipv4 peer` (headend) — PCEP UP
- [ ] `show segment-routing traffic-eng policy color 300 detail` — on-demand, metric latency, PCE-computed
- [ ] Remove colored route → ODN policy torn down automatically
- [ ] **Snapshot:** `WB18-06-odn-pce`

---

## Section 7 — Troubleshooting Drills [5 points]

- [ ] **Drill 7.1 — Policy DOWN (invalid SID) [2 points]:** put a non-existent adjacency-SID in an explicit segment-list. Observe policy down / invalid. Diagnose and fix.
- [ ] **Drill 7.2 — No autosteering [2 points]:** service route has color but policy exists for a different endpoint/color. Confirm traffic uses native path. Diagnose color/next-hop mismatch, then align.
- [ ] **Drill 7.3 — PCEP down [1 point]:** break the PCC↔PCE session (ACL/address). Confirm ODN policies cannot be delegated. Diagnose and restore.
- [ ] **Snapshot:** `WB18-07-tshoot`

**Required show commands (fluency):**
- [ ] `show segment-routing traffic-eng policy`
- [ ] `show segment-routing traffic-eng forwarding`
- [ ] `show segment-routing traffic-eng topology`
- [ ] `show pce lsp` / `show pce ipv4 peer`
- [ ] `show bgp <af> <prefix> detail` (color community)

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Dynamic policy (TE) + BSID | 4 | Intermediate: dynamic candidate-path, binding-SID |
| Section 2: Autoroute include | 3 | Intermediate: IGP shortcut steering |
| Section 3: BGP color + autosteering | 4 | CCIE-level: color community, automated steering |
| Section 4: Explicit segment-lists (adj-SID) | 4 | CCIE-level: pinned path via adjacency-SIDs |
| Section 5: Affinity constraints | 4 | CCIE-level: affinity-map, include/exclude |
| Section 6: SR-PCE + ODN (latency) | 6 | CCIE-level: PCEP, on-demand, PCE-delegated |
| Section 7: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
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

- [ ] Dynamic TE-metric SR policy with explicit binding-SID
- [ ] Autoroute-include steering working
- [ ] BGP color community + automated steering onto the policy
- [ ] Explicit segment-list pinned via adjacency-SIDs
- [ ] Affinity-map + include/exclude constraints shaping the path
- [ ] SR-PCE on R30 with ODN (on-demand color 300, latency metric, PCE-delegated)
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB18-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Dynamic policy + BSID** | **4** | | |
| 1.1 Dynamic policy (TE) | 2 | | |
| 1.2 Binding-SID | 1 | | |
| 1.3 UP + TE metric | 1 | | |
| **Section 2: Autoroute include** | **3** | | |
| 2.1 Autoroute include | 2 | | |
| 2.2 Autoroute metric | 1 | | |
| **Section 3: Color + autosteer** | **4** | | |
| 3.1 Color community | 2 | | |
| 3.2 Automated steering | 1 | | |
| 3.3 Installed via policy | 1 | | |
| **Section 4: Explicit segment-list** | **4** | | |
| 4.1 Explicit adj-SID list | 2 | | |
| 4.2 Policy w/ preference | 1 | | |
| 4.3 Forwards pinned path | 1 | | |
| **Section 5: Affinity** | **4** | | |
| 5.1 Affinity-map | 1 | | |
| 5.2 Interface affinity | 1 | | |
| 5.3 Constraints exclude/include | 2 | | |
| **Section 6: SR-PCE + ODN** | **6** | | |
| 6.1 PCE + PCC/PCEP | 2 | | |
| 6.2 ODN template latency | 2 | | |
| 6.3 On-demand instantiation | 1 | | |
| 6.4 PCE-delegated proof | 1 | | |
| **Section 7: Troubleshooting** | **5** | | |
| 7.1 Policy down (invalid SID) | 2 | | |
| 7.2 No autosteering | 2 | | |
| 7.3 PCEP down | 1 | | |
| **TOTAL** | **30** | | |
