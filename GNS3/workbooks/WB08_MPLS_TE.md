# WB08 — MPLS Traffic Engineering

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** MPLS-TE within AS 65100 (IS-IS L2 core, R1-R8)
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

## Section 1 — Enable MPLS-TE + RSVP + IS-IS TE Extensions

Core: R1, R2, R3, R4, R5, R6, R7, R8.

- [ ] **Task 1.1** — Enable the global MPLS-TE capability on every core router (R1-R8).
- [ ] **Task 1.2** — Enable TE on every core-facing interface (R1-R8), per the Full Link Map (AS 65100 Core). Do **not** enable TE on PE-CE or inter-AS interfaces.
- [ ] **Task 1.3** — Enable RSVP on each core interface and reserve an RSVP bandwidth pool on each. Choose a per-interface reservable value appropriate to the link (document your choice).
- [ ] **Task 1.4** — Add IS-IS TE extensions: advertise TE topology at **level-2** and set the MPLS-TE router-id to Loopback0 on each router.
- [ ] **Task 1.5** — Confirm the TE topology database is populated.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng link-management interfaces` — TE enabled, BW pools shown
- [ ] `show ip rsvp interface` — RSVP active, reservable BW correct
- [ ] `show isis mpls traffic-eng advertisements` — TE sub-TLVs present
- [ ] `show mpls traffic-eng topology` — all R1-R8 nodes + links with TE metrics
- [ ] **Snapshot:** `WB08-01-te-enabled`

---

## Section 2 — Basic TE Tunnel with Explicit Path (non-shortest)

Headend **R1**, tailend **R5**. Shortest IGP path is R1→R3→R5. You will force a **non-shortest** explicit path.

- [ ] **Task 2.1** — Build an explicit path object listing the hops R1 → R3 → R5. (Confirm from topology this is actually valid; if R1→R3→R5 *is* the shortest, instead select a deliberately longer path such as R1→R3→R4→R6→R5 and note it — the point is to prove explicit routing overrides CSPF's shortest choice.)
- [ ] **Task 2.2** — Create `Tunnel0` on R1: destination = R5 Loopback0 (150.1.5.5), path-option 1 = explicit (your path), path-option 2 = dynamic (fallback).
- [ ] **Task 2.3** — Bring the tunnel UP and confirm it signalled over the explicit path, not the IGP shortest path.
- [ ] **Task 2.4** — Enable **autoroute announce** on Tunnel0 so R1's routing table uses the tunnel to reach R5 and prefixes behind R5.
- [ ] **Task 2.5** — Confirm a destination behind R5 now resolves via Tunnel0 in the RIB/CEF.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng tunnels tunnel0` — state UP/UP, path = explicit hops
- [ ] `show ip route 150.1.5.5` — next-hop is Tunnel0 (autoroute)
- [ ] `traceroute` from R1 to a prefix behind R5 — transits R3 (and extra hops if longer path chosen)
- [ ] `show mpls traffic-eng tunnels summary`
- [ ] **Snapshot:** `WB08-02-tunnel-explicit`

---

## Section 3 — Fast ReRoute (FRR)

Protect Tunnel0 along its path. Target: traffic restoration **< 50 ms** on failure.

- [ ] **Task 3.1** — Enable FRR (link protection) on Tunnel0 at the headend (`tunnel mpls traffic-eng fast-reroute`).
- [ ] **Task 3.2** — On the PLR (point of local repair, e.g. R3) build a **facility backup / bypass tunnel** that protects the next link toward R5 (NHOP — next-hop link protection). The bypass must route around the protected link.
- [ ] **Task 3.3** — Confirm the primary tunnel is "protected" and the bypass is "ready".
- [ ] **Task 3.4** — **Link failure test:** shut the protected core link on the PLR. Confirm FRR cuts over to the bypass and the tunnel stays UP.
- [ ] **Task 3.5** — Node protection: build an **NNHOP (next-next-hop)** bypass on the PLR that protects against failure of the downstream node (not just the link).
- [ ] **Task 3.6** — **Node failure test:** reload/shut the protected midpoint node. Confirm NNHOP bypass engages.
- [ ] **Task 3.7** — Measure/estimate restoration time (continuous ping with tight interval; count lost packets). Confirm sub-50ms behavior (acknowledge Dynamips timing is imprecise — reason about the mechanism, not just the ping count).

**Verify / Snapshot**
- [ ] `show mpls traffic-eng fast-reroute database` — protected prefixes/tunnels
- [ ] `show mpls traffic-eng tunnels backup` — bypass tunnels + state
- [ ] `show mpls traffic-eng tunnels tunnel0 | include protect|FRR` — "ready"/"active"
- [ ] During failure: `show ip rsvp fast bw-protect` / FRR "active"
- [ ] **Snapshot:** `WB08-03-frr`

---

## Section 4 — Bandwidth Management

- [ ] **Task 4.1** — Configure a bandwidth requirement on Tunnel0 and confirm CSPF only admits it on links with sufficient reservable BW.
- [ ] **Task 4.2** — Set **setup** and **holding** priority on Tunnel0 (values 0-7; lower = better). Create a second lower-priority tunnel competing for the same constrained link.
- [ ] **Task 4.3** — **Preemption test:** request bandwidth on the higher-priority tunnel that forces the lower-priority tunnel to be torn down / rerouted. Confirm preemption occurred.
- [ ] **Task 4.4** — Enable **auto-bandwidth** on Tunnel0: periodic adjustment of reserved BW based on measured traffic (set collection interval + min/max).
- [ ] **Task 4.5** — Confirm the applied/current reserved bandwidth and the auto-bw adjustment timers.

**Verify / Snapshot**
- [ ] `show mpls traffic-eng tunnels tunnel0 | include bandwidth|priority`
- [ ] `show mpls traffic-eng link-management bandwidth-allocation` — pool usage per link
- [ ] `show mpls traffic-eng tunnels auto-bw` (or `... | include auto`)
- [ ] Preemption: lower-priority tunnel shows reroute/down event
- [ ] **Snapshot:** `WB08-04-bandwidth`

---

## Section 5 — Advanced TE

- [ ] **Task 5.1 — Make-before-break (SE):** change Tunnel0's path/bandwidth and confirm the new LSP is signalled with **Shared-Explicit** reservation style *before* the old LSP is torn down (no traffic drop). Confirm reoptimization behavior.
- [ ] **Task 5.2 — Affinity / admin-groups:** assign admin-group color bits (link attribute flags) to selected core links. On Tunnel0 set an affinity/mask so CSPF **includes** some colors and **excludes** others. Prove the tunnel avoids an excluded-color link.
- [ ] **Task 5.3 — CSPF:** force Tunnel0 to path-option = dynamic and confirm CSPF computes the constrained shortest path honoring BW + affinity constraints. Compare against plain IGP SPF.
- [ ] **Task 5.4 — DS-TE:** enable Differentiated-Services TE. Configure a sub-pool (class-type 1) for priority traffic. Test both bandwidth-constraint models:
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

## Section 6 — TE + VPN Integration

- [ ] **Task 6.1** — Confirm L3VPN traffic (an existing customer VPN from WB prerequisites that transits R1→R5) rides Tunnel0 via autoroute announce. Verify the VPN label stacks on top of the TE/tunnel label.
- [ ] **Task 6.2** — Configure **forwarding-adjacency** on Tunnel0 so the tunnel is advertised into IS-IS as a link, allowing other routers (not just the headend) to use it in SPF. Set the forwarding-adjacency holdtime/metric.
- [ ] **Task 6.3** — Confirm a *non-headend* router now installs routes over the tunnel because it appears as an IS-IS link.

**Verify / Snapshot**
- [ ] `show mpls forwarding-table vrf <VPN>` — label stack over tunnel
- [ ] `show isis database | include <tunnel>` — forwarding-adjacency advertised as link
- [ ] `traceroute` for VPN prefix shows TE path
- [ ] **Snapshot:** `WB08-06-te-vpn`

---

## Section 7 — Troubleshooting Drills

Break it, diagnose from show output, then fix.

- [ ] **Drill 7.1 — Tunnel DOWN (CSPF failed):** request bandwidth on Tunnel0 larger than any link's reservable BW. Observe the tunnel go/stay DOWN. Diagnose via CSPF/path error, then correct.
  - [ ] `show mpls traffic-eng tunnels tunnel0` → "no path"/"CSPF failed"
  - [ ] `show mpls traffic-eng link-management bandwidth-allocation`
- [ ] **Drill 7.2 — Tunnel DOWN (no BW on a hop):** reduce reservable BW on one explicit-path link below the tunnel's requirement. Confirm failure localizes to that hop.
- [ ] **Drill 7.3 — Tunnel UP but no traffic:** remove `autoroute announce`. Confirm tunnel is UP/UP yet the RIB does not use it, so traffic takes the IGP path instead. Diagnose (route points to physical, not Tunnel0), then restore autoroute.
  - [ ] `show ip route <dest>` → next-hop is physical, not Tunnel0
  - [ ] `show mpls traffic-eng autoroute`
- [ ] **Snapshot:** `WB08-07-tshoot`

---

## Completion Criteria

- [ ] MPLS-TE + RSVP + IS-IS TE extensions operational across R1-R8
- [ ] Explicit-path tunnel UP and carrying traffic via autoroute
- [ ] FRR link + node protection proven with failure tests
- [ ] Bandwidth, priority, preemption, auto-bandwidth demonstrated
- [ ] MBB(SE), affinity, CSPF, DS-TE (MAM + RDM) demonstrated
- [ ] TE + VPN integration + forwarding-adjacency confirmed
- [ ] All 3 troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB08-FINAL`
