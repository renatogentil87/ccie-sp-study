# SR-EX01: IS-IS + Segment Routing Foundation

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** None

---

## Section 1: IP Addressing

### Task 1
Configure loopback and core link IP addresses on all 6 routers per the topology reference. Verify all directly connected links can ping their neighbor.

---

## Section 2: IS-IS as Core IGP

### Task 2
Configure IS-IS instance `CORE` on ALL routers (R1-R6):
- Level-2 only
- Metric-style wide
- NET format: 49.0001 (Area 1) + loopback address in system-ID format
- Loopback0 as passive interface
- All core-facing interfaces as point-to-point
- Do NOT enable IS-IS on PE-CE interfaces

### Task 3
Verify IS-IS:
- All adjacencies L2/UP
- All 6 loopbacks reachable via IS-IS
- No pseudonodes in the LSDB (all point-to-point)

---

## Section 3: Segment Routing

### Task 4
Enable Segment Routing MPLS on all routers (R1-R6) under the IS-IS process. Assign prefix-SID index on each router's Loopback0:
- R1 = index 1, R2 = 2, R3 = 3, R4 = 4, R5 = 5, R6 = 6

### Task 5
Verify SR:
- `show isis segment-routing label table` — SIDs 16001-16006 present on all routers
- `show mpls forwarding` — SR Pfx entries in LFIB
- Adjacency-SIDs auto-allocated on every adjacency (protected + unprotected)

### Task 6
Traceroute from R1 to R6. Observe:
- Same SR label at every hop (globally significant)
- PHP at the penultimate hop
- Compare this with how LDP would show different labels at each hop

---

## Section 4: LDP alongside SR (coexistence)

### Task 7
Enable LDP on R1, R2, and R3 only (simulating a partial LDP deployment). Keep SR running on all routers.

### Task 8
On R3, verify both LDP and SR labels exist in the LFIB for the same destinations. Identify which label is active and which is backup.

### Task 9
On R3, enable `sr-prefer` so SR labels are preferred over LDP labels. Verify: SR label active, LDP label backup.

---

## Snapshot
Take: **"SR-EX01-foundation"**

## Checklist
```
[ ] IP addressing complete, all links pingable
[ ] IS-IS L2 adjacencies UP on all core links
[ ] All 6 loopbacks reachable
[ ] No pseudonodes (point-to-point on all interfaces)
[ ] Prefix-SIDs 16001-16006 in label table
[ ] Adj-SIDs auto-allocated (protected + unprotected)
[ ] Traceroute shows same SR label at every hop + PHP
[ ] LDP running on R1/R2/R3 alongside SR
[ ] R3 shows both LDP and SR labels
[ ] sr-prefer: SR active, LDP backup on R3
```
