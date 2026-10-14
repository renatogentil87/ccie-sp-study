# WB19 — Flexible Algorithm (Flex-Algo)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Flex-Algo within AS 65100 (IS-IS L2 core, R1-R8); PEs R1/R2
**Level:** CCIE SP
**Prerequisite:** WB16 (SR-MPLS: SRGB, prefix-SIDs installed) and WB18 affinity-map concepts

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab.
> Flex-Algo is **not** supported on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: verify each section before advancing.
> **Snapshot** before/after each section.

> **Reference:** AS 65100 R1-R8, IS-IS area 49.0001 L2. Prefix-SID index = router number (WB16).
> PEs = R1 (150.1.1.1), R2 (150.1.2.2). Core links per `00_topology_reference.md`.
> **Affinity colors used:** `red` (bit 0), `blue` (bit 1) — same affinity-map concept as WB18.

---

## Pre-flight

- [ ] Confirm WB16 SR-MPLS operational (base algo 0 prefix-SIDs installed)
- [ ] Confirm IS-IS converged on R1-R8
- [ ] Decide which core links will carry `red` vs `blue` affinity (document the plan)
- [ ] **Snapshot:** `WB19-00-baseline`

---

## Section 1 — Affinity-Map + Interface Affinity [3 points]

- [ ] **Task 1.1 [2 points]** — Define the **affinity-map** consistently on all R1-R8: under `router isis <tag>` → `flex-algo`-related `affinity-map`, OR `segment-routing traffic-eng affinity-map`, map names to **bit-positions**: `red bit-position 0`, `blue bit-position 1`.
- [ ] **Task 1.2 [1 point]** — Assign **interface affinity** on selected core links: color some links `red`, some `blue` (`affinity flex-algo red` under the IS-IS interface, per XR syntax). Document which links got which color.
- [ ] **Task 1.3 [0 points — verification]** — Confirm affinity-map and per-link affinity are advertised in IS-IS.

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng affinity-map` (or IS-IS flex-algo affinity)
- [ ] `show isis database verbose <node>` — link affinity / extended admin group sub-TLV
- [ ] **Snapshot:** `WB19-01-affinity-map`

---

## Section 2 — Flex-Algo Definitions 233 & 234 [5 points]

- [ ] **Task 2.1 [2 points]** — On all R1-R8, define **Flex-Algo 233** under `router isis <tag>` → `flex-algo 233`: metric-type (igp or delay), and `advertise-definition`. Constraint: **exclude the `red` affinity** (`affinity exclude-any red`).
- [ ] **Task 2.2 [2 points]** — Define **Flex-Algo 234**: `flex-algo 234` with `advertise-definition` and constraint **exclude the `blue` affinity** (`affinity exclude-any blue`). (Also practice `include-all` / `include-any` on one of the two to demonstrate the alternate constraint form.)
- [ ] **Task 2.3 [1 point]** — Elect/confirm a consistent **FAD (Flex-Algo Definition)** winner fleet-wide (only one definition should win per algo — confirm all nodes agree on 233 and 234 definitions).
- [ ] **Task 2.4 [0 points — verification]** — Confirm each node has computed a per-algo SPF for 233 and 234 that respects the exclusions.

**Verify / Snapshot**
- [ ] `show isis flex-algo 233` / `show isis flex-algo 234` — definition, metric, exclude affinity
- [ ] `show isis flex-algo 233 detail` — winning FAD + participating nodes
- [ ] `show route algorithm 233` / `show route algorithm 234` — per-algo paths avoid excluded color
- [ ] **Snapshot:** `WB19-02-flex-algo-def`

---

## Section 3 — Per-Algo Prefix-SID on PEs [4 points]

- [ ] **Task 3.1 [2 points]** — On PE **R1**, allocate an additional **per-algo prefix-SID** on Loopback0 for algo 233 and algo 234 (`segment-routing prefix-sid index <N> algorithm 233` and `… algorithm 234`). Use distinct indexes per algo (e.g. 233-plane index 1233, 234-plane index 1234, or your scheme — document it).
- [ ] **Task 3.2 [1 point]** — Repeat on PE **R2** with its own per-algo indexes.
- [ ] **Task 3.3 [1 point]** — Confirm remote nodes install per-algo labels toward R1/R2 that follow the algo-233 / algo-234 constrained topology (not the algo-0 shortest path).
- [ ] **Task 3.4 [0 points — verification]** — Compare the algo-0 path vs algo-233 vs algo-234 to R1 (they should differ based on red/blue exclusions).

**Verify / Snapshot**
- [ ] `show isis segment-routing prefix-sid` — algo 0 + 233 + 234 SIDs per PE
- [ ] `show mpls forwarding` — per-algo labels toward R1/R2
- [ ] `show cef 150.1.1.1/32 algorithm 233 detail` vs `… algorithm 234 detail` — different paths
- [ ] **Snapshot:** `WB19-03-per-algo-sid`

---

## Section 4 — SR Policies Constrained to Flex-Algo [4 points]

- [ ] **Task 4.1 [2 points]** — Create an SR policy on a headend (e.g. R8) to endpoint R1 whose dynamic candidate-path is constrained to a flex-algo: `constraints segments sid-algorithm 233` (compute using algo-233 plane). Create a second policy using `sid-algorithm 234`.
- [ ] **Task 4.2 [1 point]** — Confirm each policy's computed SID list uses the per-algo prefix-SIDs and therefore follows the red-avoiding (233) or blue-avoiding (234) topology.
- [ ] **Task 4.3 [1 point]** — Steer a colored service route onto the algo-233 policy and confirm forwarding follows the constrained plane.
- [ ] **Task 4.4 [0 points — verification]** — Confirm the two policies take visibly different paths.

**Verify / Snapshot**
- [ ] `show segment-routing traffic-eng policy color <c> detail` — sid-algorithm 233/234, SID list
- [ ] `traceroute` for each policy — different (color-avoiding) paths
- [ ] **Snapshot:** `WB19-04-policy-flex-algo`

---

## Section 5 — Priority for Algorithm Selection [3 points]

- [ ] **Task 5.1 [2 points]** — Configure the **priority** value in the Flex-Algo definition (`priority <0-255>`) so that if multiple nodes advertise a definition for the same algo, the highest-priority FAD wins deterministically. Set R7/R8 (RRs) as the authoritative definition source with a higher priority than other nodes.
- [ ] **Task 5.2 [1 point]** — Introduce a conflicting definition on a lower-priority node and confirm the **higher priority** (and, on tie, highest system-ID) wins — all nodes converge on the same FAD.
- [ ] **Task 5.3 [0 points — verification]** — Confirm no FAD inconsistency fleet-wide.

**Verify / Snapshot**
- [ ] `show isis flex-algo 233 detail` — elected FAD + priority + advertising node
- [ ] After conflict: all nodes show the SAME winning definition
- [ ] **Snapshot:** `WB19-05-priority`

---

## Section 6 — Troubleshooting Drills [5 points]

- [ ] **Drill 6.1 — FAD mismatch [2 points]:** configure algo 233 with different exclude affinity on one node (no advertise-definition / disagreeing definition). Observe inconsistent per-algo SPF. Diagnose with `show isis flex-algo` and fix (single authoritative definition).
- [ ] **Drill 6.2 — Missing per-algo prefix-SID [2 points]:** remove the algo-233 prefix-SID from R1. Confirm algo-233 policies/paths toward R1 break while algo-0 still works. Diagnose and restore.
- [ ] **Drill 6.3 — Affinity not excluded [1 point]:** forget to color a link `red`; confirm algo-233 traverses a link it should have avoided. Add the affinity and reconverge.
- [ ] **Snapshot:** `WB19-06-tshoot`

**Required show commands (fluency):**
- [ ] `show isis flex-algo [233|234] [detail]`
- [ ] `show route algorithm <n>`
- [ ] `show isis segment-routing prefix-sid`
- [ ] `show cef <prefix> algorithm <n> detail`
- [ ] `show segment-routing traffic-eng policy`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Affinity-map + interface affinity | 3 | Intermediate: bit-position mapping |
| Section 2: Flex-Algo 233 & 234 definitions | 5 | CCIE-level: advertise-definition, exclude/include |
| Section 3: Per-algo prefix-SID on PEs | 4 | CCIE-level: multi-plane SID allocation |
| Section 4: SR policies constrained to algo | 4 | CCIE-level: sid-algorithm constraint |
| Section 5: Priority for algo selection | 3 | Intermediate: FAD election |
| Section 6: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
| **Total** | **24** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
21-24 points:  CCIE ready for this topic
16-20 points:  Strong, minor gaps to review
10-15 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Affinity-map (red bit 0, blue bit 1) + interface affinity advertised in IS-IS
- [ ] Flex-Algo 233 (exclude red) and 234 (exclude blue) defined + advertised fleet-wide
- [ ] Per-algo prefix-SIDs on R1/R2 following constrained planes
- [ ] SR policies constrained via sid-algorithm 233/234
- [ ] Deterministic FAD election via priority
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB19-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Affinity-map** | **3** | | |
| 1.1 Affinity-map bit-position | 2 | | |
| 1.2 Interface affinity | 1 | | |
| **Section 2: Flex-Algo definitions** | **5** | | |
| 2.1 Algo 233 exclude red | 2 | | |
| 2.2 Algo 234 exclude blue | 2 | | |
| 2.3 FAD winner fleet-wide | 1 | | |
| **Section 3: Per-algo prefix-SID** | **4** | | |
| 3.1 R1 per-algo SIDs | 2 | | |
| 3.2 R2 per-algo SIDs | 1 | | |
| 3.3 Remote install per-algo | 1 | | |
| **Section 4: Policies constrained** | **4** | | |
| 4.1 sid-algorithm 233/234 | 2 | | |
| 4.2 SID list follows plane | 1 | | |
| 4.3 Steer service route | 1 | | |
| **Section 5: Priority** | **3** | | |
| 5.1 Priority config | 2 | | |
| 5.2 Election proof | 1 | | |
| **Section 6: Troubleshooting** | **5** | | |
| 6.1 FAD mismatch | 2 | | |
| 6.2 Missing per-algo SID | 2 | | |
| 6.3 Affinity not excluded | 1 | | |
| **TOTAL** | **24** | | |
