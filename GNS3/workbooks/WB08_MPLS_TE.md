# WB08 — MPLS Traffic Engineering

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** MPLS-TE within AS 65100 (IS-IS L2 core, R31, R2-R8)
**Level:** CCIE SP
**Prerequisite:** WB00-WB07 complete (IGP, LDP, L3VPN, inter-AS baseline all UP)

> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take a **GNS3 snapshot** at each checkpoint so you can roll back cleanly.

---

## Pre-flight

- [ ] Confirm AS 65100 IS-IS L2 is fully converged (`show isis neighbors`, all core adjacencies UP)
- [ ] Confirm LDP is UP on all R1-R8 core links (`show mpls ldp neighbor`)
- [ ] Confirm end-to-end LSPs exist before adding TE (`show mpls forwarding-table`)
- [ ] **Snapshot:** `WB08-00-baseline`

---

## Section 1 — Enable MPLS-TE + RSVP + IS-IS TE Extensions [2 points]

Core: R31, R2, R3, R4, R5, R6, R7, R8.

- [ ] **Task 1.1 [1 point]** — Enable the global MPLS-TE capability on every core router (R31, R2-R8).
- [ ] **Task 1.2 [1 point]** — Enable TE on every core-facing interface (R31, R2-R8), per the Full Link Map (AS 65100 Core). Do **not** enable TE on PE-CE or inter-AS interfaces.
- [ ] **Task 1.3 [0 points — included above]** — Enable RSVP on each core interface and reserve an RSVP bandwidth pool on each. Choose a per-interface reservable value appropriate to the link (document your choice).
- [ ] **Task 1.4 [0 points — included above]** — Add IS-IS TE extensions: advertise TE topology at **level-2** and set the MPLS-TE router-id to Loopback0 on each router.
- [ ] **Task 1.5 [0 points — verification]** — Confirm the TE topology database is populated.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng link-management interfaces` — TE enabled, BW pools shown
- [ ] `show ip rsvp interface` — RSVP active, reservable BW correct
- [ ] `show isis mpls traffic-eng advertisements` — TE sub-TLVs present
- [ ] `show mpls traffic-eng topology` — all R1-R8 nodes + links with TE metrics
- [ ] **Snapshot:** `WB08-01-te-enabled`

---

## Section 2 — Basic TE Tunnel with Explicit Path (non-shortest) [3 points]

Headend **R1**, tailend **R5**. Shortest IGP path is R31→R3→R5. You will force a **non-shortest** explicit path.

- [ ] **Task 2.1 [1 point]** — Build an explicit path object listing the hops R1 → R3 → R5. (Confirm from topology this is actually valid; if R31→R3→R5 *is* the shortest, instead select a deliberately longer path such as R31→R3→R4→R6→R5 and note it — the point is to prove explicit routing overrides CSPF's shortest choice.)
- [ ] **Task 2.2 [1 point]** — Create `Tunnel0` on R1: destination = R5 Loopback0 (150.1.5.5), path-option 1 = explicit (your path), path-option 2 = dynamic (fallback).
- [ ] **Task 2.3 [0 points — verification]** — Bring the tunnel UP and confirm it signalled over the explicit path, not the IGP shortest path.
- [ ] **Task 2.4 [1 point]** — Enable **autoroute announce** on Tunnel0 so R1's routing table uses the tunnel to reach R5 and prefixes behind R5.
- [ ] **Task 2.5 [0 points — verification]** — Confirm a destination behind R5 now resolves via Tunnel0 in the RIB/CEF.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng tunnels tunnel0` — state UP/UP, path = explicit hops
- [ ] `show ip route 150.1.5.5` — next-hop is Tunnel0 (autoroute)
- [ ] `traceroute` from R1 to a prefix behind R5 — transits R3 (and extra hops if longer path chosen)
- [ ] `show mpls traffic-eng tunnels summary`
- [ ] **Snapshot:** `WB08-02-tunnel-explicit`

---

## Section 3 — Fast ReRoute (FRR) [5 points]

Protect Tunnel0 along its path. Target: traffic restoration **< 50 ms** on failure.

- [ ] **Task 3.1 [1 point]** — Enable FRR (link protection) on Tunnel0 at the headend (`tunnel mpls traffic-eng fast-reroute`).
- [ ] **Task 3.2 [1 point]** — On the PLR (point of local repair, e.g. R3) build a **facility backup / bypass tunnel** that protects the next link toward R5 (NHOP — next-hop link protection). The bypass must route around the protected link.
- [ ] **Task 3.3 [0 points — verification]** — Confirm the primary tunnel is "protected" and the bypass is "ready".
- [ ] **Task 3.4 [1 point]** — **Link failure test:** shut the protected core link on the PLR. Confirm FRR cuts over to the bypass and the tunnel stays UP.
- [ ] **Task 3.5 [2 points]** — Node protection: build an **NNHOP (next-next-hop)** bypass on the PLR that protects against failure of the downstream node (not just the link).
- [ ] **Task 3.6 [0 points — verification]** — **Node failure test:** reload/shut the protected midpoint node. Confirm NNHOP bypass engages.
- [ ] **Task 3.7 [0 points — verification]** — Measure/estimate restoration time (continuous ping with tight interval; count lost packets). Confirm sub-50ms behavior (acknowledge Dynamips timing is imprecise — reason about the mechanism, not just the ping count).

**Verify / Snapshot**
- [ ] `show mpls traffic-eng fast-reroute database` — protected prefixes/tunnels
- [ ] `show mpls traffic-eng tunnels backup` — bypass tunnels + state
- [ ] `show mpls traffic-eng tunnels tunnel0 | include protect|FRR` — "ready"/"active"
- [ ] During failure: `show ip rsvp fast bw-protect` / FRR "active"
- [ ] **Snapshot:** `WB08-03-frr`

---

## Section 4 — Bandwidth Management [3 points]

- [ ] **Task 4.1 [1 point]** — Configure a bandwidth requirement on Tunnel0 and confirm CSPF only admits it on links with sufficient reservable BW.
- [ ] **Task 4.2 [1 point]** — Set **setup** and **holding** priority on Tunnel0 (values 0-7; lower = better). Create a second lower-priority tunnel competing for the same constrained link.
- [ ] **Task 4.3 [1 point]** — **Preemption test:** request bandwidth on the higher-priority tunnel that forces the lower-priority tunnel to be torn down / rerouted. Confirm preemption occurred.
- [ ] **Task 4.4 [0 points — included above]** — Enable **auto-bandwidth** on Tunnel0: periodic adjustment of reserved BW based on measured traffic (set collection interval + min/max).
- [ ] **Task 4.5 [0 points — verification]** — Confirm the applied/current reserved bandwidth and the auto-bw adjustment timers.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng tunnels tunnel0 | include bandwidth|priority`
- [ ] `show mpls traffic-eng link-management bandwidth-allocation` — pool usage per link
- [ ] `show mpls traffic-eng tunnels auto-bw` (or `... | include auto`)
- [ ] Preemption: lower-priority tunnel shows reroute/down event
- [ ] **Snapshot:** `WB08-04-bandwidth`

---

## Section 5 — Advanced TE [5 points]

- [ ] **Task 5.1 — Make-before-break (SE) [1 point]:** change Tunnel0's path/bandwidth and confirm the new LSP is signalled with **Shared-Explicit** reservation style *before* the old LSP is torn down (no traffic drop). Confirm reoptimization behavior.
- [ ] **Task 5.2 — Affinity / admin-groups [2 points]:** assign admin-group color bits (link attribute flags) to selected core links. On Tunnel0 set an affinity/mask so CSPF **includes** some colors and **excludes** others. Prove the tunnel avoids an excluded-color link.
- [ ] **Task 5.3 — CSPF [0 points — included above]:** force Tunnel0 to path-option = dynamic and confirm CSPF computes the constrained shortest path honoring BW + affinity constraints. Compare against plain IGP SPF.
- [ ] **Task 5.4 — DS-TE [2 points]:** enable Differentiated-Services TE. Configure a sub-pool (class-type 1) for priority traffic. Test both bandwidth-constraint models:
  - [ ] **MAM** (Maximum Allocation Model)
  - [ ] **RDM** (Russian Dolls Model)
  Reserve sub-pool BW on a tunnel and confirm admission against the correct class-type.
  - [ ] **MAM** (Maximum Allocation Model)
  - [ ] **RDM** (Russian Dolls Model)
  Reserve sub-pool BW on a tunnel and confirm admission against the correct class-type.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng topology | include affinity|attribute` — color bits on links
- [ ] `show mpls traffic-eng tunnels tunnel0 | include affinity|Style|SE`
- [ ] `show mpls traffic-eng link-management bandwidth-allocation` — sub-pool vs global pool
- [ ] Confirm reopt did not drop traffic (continuous ping clean across change)
- [ ] **Snapshot:** `WB08-05-advanced`

---

## Section 6 — TE + VPN Integration [5 points]

- [ ] **Task 6.1 [2 points]** — Confirm L3VPN traffic (an existing customer VPN from WB prerequisites that transits R31→R5) rides Tunnel0 via autoroute announce. Verify the VPN label stacks on top of the TE/tunnel label.
- [ ] **Task 6.2 [3 points]** — Configure **forwarding-adjacency** on Tunnel0 so the tunnel is advertised into IS-IS as a link, allowing other routers (not just the headend) to use it in SPF. Set the forwarding-adjacency holdtime/metric.
- [ ] **Task 6.3 [0 points — verification]** — Confirm a *non-headend* router now installs routes over the tunnel because it appears as an IS-IS link.

**Verify / Snapshot**
- [ ] `show mpls forwarding-table vrf <VPN>` — label stack over tunnel
- [ ] `show isis database | include <tunnel>` — forwarding-adjacency advertised as link
- [ ] `traceroute` for VPN prefix shows TE path
- [ ] **Snapshot:** `WB08-06-te-vpn`

---

## Section 7 — Troubleshooting Drills [5 points]

Break it, diagnose from show output, then fix.

- [ ] **Drill 7.1 — Tunnel DOWN (CSPF failed) [2 points]:** request bandwidth on Tunnel0 larger than any link's reservable BW. Observe the tunnel go/stay DOWN. Diagnose via CSPF/path error, then correct.
  - [ ] `show mpls traffic-eng tunnels tunnel0` → "no path"/"CSPF failed"
  - [ ] `show mpls traffic-eng link-management bandwidth-allocation`
- [ ] **Drill 7.2 — Tunnel DOWN (no BW on a hop) [2 points]:** reduce reservable BW on one explicit-path link below the tunnel's requirement. Confirm failure localizes to that hop.
- [ ] **Drill 7.3 — Tunnel UP but no traffic [1 point]:** remove `autoroute announce`. Confirm tunnel is UP/UP yet the RIB does not use it, so traffic takes the IGP path instead. Diagnose (route points to physical, not Tunnel0), then restore autoroute.
  - [ ] `show ip route <dest>` → next-hop is physical, not Tunnel0
  - [ ] `show mpls traffic-eng autoroute`
- [ ] **Snapshot:** `WB08-07-tshoot`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Enable MPLS-TE + RSVP + IS-IS TE | 2 | Standard: enable TE/RSVP, TE extensions |
| Section 2: Basic TE Tunnel (explicit path) | 3 | Intermediate: explicit path + autoroute |
| Section 3: Fast ReRoute (FRR) | 5 | CCIE-level: NHOP/NNHOP bypass + failure tests |
| Section 4: Bandwidth Management | 3 | Intermediate: BW, priority, preemption, auto-bw |
| Section 5: Advanced TE | 5 | CCIE-level: MBB/SE, affinity, DS-TE (MAM+RDM) |
| Section 6: TE + VPN Integration | 5 | CCIE-level: label stacking, forwarding-adjacency |
| Section 7: Troubleshooting Drills | 5 | CCIE-level: break-and-fix scenarios |
| **Total** | **28** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
24-28 points:  CCIE ready for this topic
18-23 points:  Strong, minor gaps to review
12-17 points:  Needs more practice
Below 12:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] MPLS-TE + RSVP + IS-IS TE extensions operational across R31, R2-R8
- [ ] Explicit-path tunnel UP and carrying traffic via autoroute
- [ ] FRR link + node protection proven with failure tests
- [ ] Bandwidth, priority, preemption, auto-bandwidth demonstrated
- [ ] MBB(SE), affinity, CSPF, DS-TE (MAM + RDM) demonstrated
- [ ] TE + VPN integration + forwarding-adjacency confirmed
- [ ] All 3 troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB08-FINAL`

---

## Attempt History

### Attempt 1 — October 11-12, 2026

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Enable TE + RSVP + IS-IS TE** | **2** | **1** | |
| 1.1 Global TE + interfaces | 1 | 1 | ✅ From memory — all R1-R6 core interfaces |
| 1.2 RSVP + IS-IS TE extensions | 1 | 0 | ❌ Needed help: forgot `mpls traffic-eng level-2` + `router-id Lo0` under IS-IS |
| **Section 2: Basic TE Tunnel** | **3** | **3** | ✅ PERFECT |
| 2.1 Explicit path | 1 | 1 | ✅ R31→R3→R4→R6→R5 non-shortest path |
| 2.2 Tunnel0 creation | 1 | 1 | ✅ Tunnel mode, destination, path-options, dynamic fallback |
| 2.4 Autoroute announce | 1 | 1 | ✅ Traffic resolves via Tunnel0 |
| **Section 3: FRR** | **5** | **5** | ✅ (config correct, verification excluded — Dynamips limitation) |
| 3.1 FRR on headend | 1 | 1 | ✅ fast-reroute node-protect + record-route from memory |
| 3.2 Bypass tunnel on PLR | 1 | 1 | ✅ R3 bypass to NNHOP R6 via R5, backup-path on gi1/0 |
| 3.5 Node protection | 2 | 2 | ✅ Config correct, NNHOP bypass to R6 |
| 3.4 Link failure test | 1 | 1 | ✅ Config correct — FRR state didn't populate (Dynamips), excluded from verification |
| **Section 4: Bandwidth Management** | **3** | **2.5** | |
| 4.1 Bandwidth requirement | 1 | 1 | ✅ 15000 kbps on Tunnel0, CSPF admits |
| 4.2 Priority/preemption setup | 1 | 1 | ✅ Tunnel0 pri 6/5, Tunnel1 pri 4/3 |
| 4.3 Preemption test | 1 | 0.5 | ⚠️ Half points — understood concept, needed help with bandwidth math |
| 4.4 Auto-bandwidth | 0 | — | ✅ From memory (verification task) |
| **Section 5: Advanced TE** | **5** | **3** | |
| 5.1 Make-before-break (SE) | 1 | 1 | ✅ From memory — SE already default, verified with show ip rsvp reservation |
| 5.2 Affinity/admin-groups | 2 | 2 | ✅ From memory — attribute-flags 0x1 on links, affinity 0x1 mask 0x1 on tunnel |
| 5.4 DS-TE | 2 | — | ⛔ Excluded — not supported on Dynamips |
| **Section 6: TE + VPN Integration** | **5** | **3** | |
| 6.1 VPN over TE tunnel | 2 | 2 | ✅ Autoroute working, traceroute follows explicit path through tunnel |
| 6.2 Forwarding-adjacency | 3 | 3 | ✅ From memory — holdtime 300, advertised as IS-IS link |
| **Section 7: Troubleshooting** | **5** | **—** | ⛔ Excluded — break-and-fix not testable in current lab state |
| **TOTAL (adjusted)** | **23** | **19.5** | **85% — Strong** (excludes DS-TE + troubleshooting) |

**Excluded tasks:** DS-TE (platform), Troubleshooting drills (lab state), FRR verification (Dynamips)
**From memory:** Tunnel creation, explicit path, autoroute, FRR config, auto-bw, affinity, forwarding-adjacency, MBB/SE, VPN over TE
**Needed help:** IS-IS TE extensions (forgot mpls traffic-eng level-2), preemption bandwidth math (half points)
