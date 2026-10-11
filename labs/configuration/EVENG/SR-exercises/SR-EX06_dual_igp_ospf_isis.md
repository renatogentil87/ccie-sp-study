# SR-EX06: Dual IGP — OSPF (LDP) ↔ IS-IS (SR) with Boundary Routers

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 concepts understood. Start fresh from base IP addressing.

---

## Scenario

Split the network into two IGP domains with different transport technologies:

```
OSPF + LDP domain:     R1, R2, R3, R4
IS-IS + SR domain:     R3, R4, R5, R6

Boundary routers:       R3 and R4 run BOTH OSPF and IS-IS
```

- R1, R2 run OSPF Area 0 + LDP only (legacy)
- R5, R6 run IS-IS L2 + SR only (modern)
- R3, R4 run both IGPs, both transports — they're the boundary

---

## Section 1: OSPF + LDP Domain (R1, R2, R3, R4)

### Task 1
Configure OSPF Area 0 on R1, R2, R3, and R4. Enable on core-facing interfaces only (not PE-CE). Passive on Loopback0. Verify: all OSPF adjacencies UP, all 4 loopbacks reachable.

### Task 2
Configure LDP on R1, R2, R3, and R4 on all OSPF-enabled core interfaces. Verify: LDP neighbor sessions match OSPF adjacencies. LFIB populated with labels for all OSPF loopbacks.

### Task 3
Traceroute R1 → R4. Verify LDP labels — different label at every hop (locally significant).

---

## Section 2: IS-IS + SR Domain (R3, R4, R5, R6)

### Task 4
Configure IS-IS instance `CORE` on R3, R4, R5, and R6:
- Level-2 only
- Metric-style wide
- NET format: 49.0001 + loopback system-ID
- Point-to-point on all IS-IS interfaces
- Passive on Loopback0

**Important:** R3 and R4 only enable IS-IS on their links toward the SR domain (R3↔R5, R3↔R6, R4↔R5, R3↔R4). They do NOT enable IS-IS on their OSPF-facing links (R1↔R3, R1↔R2, R2↔R4).

### Task 5
Enable Segment Routing MPLS under IS-IS on R3, R4, R5, R6. Prefix-SIDs: R3=3, R4=4, R5=5, R6=6.

### Task 6
Verify: IS-IS adjacencies UP between R3↔R4, R3↔R5, R3↔R6, R4↔R5, R5↔R6. SR label table shows 16003-16006. R5 and R6 have full SR reachability.

---

## Section 3: Boundary — Route Redistribution

### Task 7
R3 and R4 need to redistribute between OSPF and IS-IS so that:
- R1/R2 (OSPF) can learn routes to R5/R6 (IS-IS)
- R5/R6 (IS-IS) can learn routes to R1/R2 (OSPF)

Configure mutual redistribution on R3 and R4:
- Redistribute IS-IS into OSPF
- Redistribute OSPF into IS-IS
- Use route-policies/tags to prevent routing loops (routes learned from OSPF should not be redistributed back into OSPF via the other boundary router)

### Task 8
Verify: R1 can see R6's loopback in its routing table. R6 can see R1's loopback. All 6 routers have full reachability to all 6 loopbacks.

### Task 9
Traceroute R1 → R6. Observe the transport change at the boundary: LDP labels from R1 to R3 (OSPF domain), then SR labels from R3 to R6 (IS-IS domain). Identify where the label swap happens.

---

## Section 4: Mapping Server for SR → LDP

### Task 10
R6 (SR only) wants to push an SR label toward R1. But R1 has no prefix-SID. Configure a mapping server on R3 (or R4) to create virtual prefix-SIDs for R1 (index 101) and R2 (index 102).

### Task 11
Verify: R6's IS-IS label table now shows 16101 (R1) and 16102 (R2) from the mapping server. R6 can build a full SR label path to R1.

### Task 12
Traceroute R6 → R1. Observe: SR labels (including mapped SID 16101) from R6 to R3, then LDP labels from R3 to R1.

---

## Section 5: TI-LFA on the SR Domain

### Task 13
Enable TI-LFA with node protection on the IS-IS domain (R3, R4, R5, R6). Verify coverage and backup paths within the SR domain.

### Task 14
Test: continuous ping R6 → R1. Shut a link in the SR domain. TI-LFA protects the SR segment. LDP domain unaffected. Verify 0-1 packet loss.

---

## Section 6: L3VPN across Both Domains

### Task 15
Configure R3 as Route Reflector (BGP AS 65100, VPNv4). Add R1, R2, R6 as RR clients.

### Task 16
Configure VRF `CUSTOMER` on R1 (CE1, eBGP 65001) and R6 (CE2, eBGP 65002). Same RT.

### Task 17
Verify: CE1 ping CE2 works end-to-end. Traceroute shows the full path: LDP domain → boundary → SR domain. VPN label + transport label visible.

### Task 18
Test VPN resilience: shut a link in the SR domain. TI-LFA reroutes. CE1↔CE2 connectivity maintained with minimal packet loss.

---

## Snapshot
Take: **"SR-EX06-dual-igp"**

## Checklist
```
[ ] OSPF Area 0 on R1/R2/R3/R4 — adjacencies UP
[ ] LDP on R1/R2/R3/R4 — LFIB populated
[ ] IS-IS L2 on R3/R4/R5/R6 — adjacencies UP, no OSPF interfaces in IS-IS
[ ] SR prefix-SIDs 16003-16006 on IS-IS routers
[ ] R3/R4 run both OSPF+LDP and IS-IS+SR simultaneously
[ ] Mutual redistribution OSPF↔IS-IS on R3/R4 with loop prevention
[ ] Full reachability: all 6 loopbacks reachable from every router
[ ] Traceroute R1→R6: LDP labels → swap at boundary → SR labels
[ ] Mapping server: R1=16101, R2=16102 visible on R6
[ ] Traceroute R6→R1: SR mapped labels → swap at boundary → LDP labels
[ ] TI-LFA protecting SR domain, sub-50ms failover
[ ] L3VPN CE1↔CE2 working across both domains
[ ] VPN protected by TI-LFA during link failure
```
