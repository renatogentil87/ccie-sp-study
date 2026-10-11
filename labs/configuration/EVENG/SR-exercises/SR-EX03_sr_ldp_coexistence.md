# SR-EX03: SR over LDP + LDP over SR + Mapping Server

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 complete

---

## Scenario

Split the network into two transport domains to simulate SR↔LDP interworking:
- **LDP domain:** R1, R2 — remove SR from these routers, keep LDP only
- **SR domain:** R4, R5, R6 — remove LDP from these routers, keep SR only
- **Boundary:** R3 — runs BOTH LDP and SR

---

## Section 1: Create the Split

### Task 1
Remove SR (prefix-SID and segment-routing mpls) from R1 and R2. These routers should run IS-IS + LDP only — simulating a legacy part of the network that hasn't migrated to SR yet.

### Task 2
Remove LDP from R4, R5, and R6. These routers should run IS-IS + SR only — the modern part of the network.

### Task 3
R3 keeps both LDP and SR. Verify on R3: both LDP labels and SR labels exist in the LFIB. R3 is the boundary between the two domains.

---

## Section 2: LDP over SR (R1 → R6 direction)

### Task 4
From R1, try to reach R6 (172.16.6.6). Does it work? R1 uses LDP. R1 pushes an LDP label toward R3. At R3, the LDP label is swapped for an SR label toward R6. Verify the label swap at R3.

### Task 5
Traceroute from R1 to R6. Observe: LDP labels in the R1→R3 segment, SR labels in the R3→R6 segment. The boundary swap is visible.

---

## Section 3: SR over LDP (R6 → R1 direction) — Mapping Server

### Task 6
From R6, try to reach R1 (172.16.1.1). Check R6's IS-IS segment routing label table — is there a prefix-SID for R1? There shouldn't be, because R1 has no prefix-SID configured. R6 can't build an SR label path to R1.

### Task 7
Configure a mapping server on R3 to create virtual prefix-SIDs for the LDP-only routers:
- R1 (172.16.1.1): index 101
- R2 (172.16.2.2): index 102

These indexes must NOT conflict with existing prefix-SIDs (1-6).

### Task 8
Verify: on R6, check the IS-IS segment routing label table. R1 should now appear as 16101, R2 as 16102 (from the mapping server). Check `prefix-sid-map active` to confirm the mapping server entries.

### Task 9
From R6, traceroute to R1. Observe: SR labels (using mapped SID 16101) in the R6→R3 segment, then LDP labels in the R3→R1 segment.

### Task 10
Test bidirectional: ping R1 → R6 AND R6 → R1. Both must work across the LDP↔SR boundary.

---

## Section 4: Migration — Remove LDP Completely

### Task 11
On R3, enable `sr-prefer` so SR labels take priority over LDP for all destinations in the SR domain.

### Task 12
Re-enable SR on R1 and R2 (add prefix-SID index 1 and 2 back). Now all routers have real prefix-SIDs.

### Task 13
Remove LDP from R1, R2, and R3. The entire network now runs pure SR. Verify no traffic loss during the transition. Verify: `show mpls ldp neighbor` is empty on all routers.

### Task 14
Remove the mapping server from R3 — it's no longer needed since R1 and R2 have real prefix-SIDs. Verify no traffic loss.

---

## Snapshot
Take: **"SR-EX03-coexistence"**

## Checklist
```
[ ] R1/R2 running LDP only (no SR)
[ ] R4/R5/R6 running SR only (no LDP)
[ ] R3 running both — dual labels in LFIB
[ ] R1→R6 works: LDP labels → swap at R3 → SR labels
[ ] R6 has no prefix-SID for R1 (before mapping server)
[ ] Mapping server on R3: R1=16101, R2=16102
[ ] R6→R1 works: SR mapped labels → swap at R3 → LDP labels
[ ] Bidirectional ping R1↔R6 works
[ ] sr-prefer enabled on R3
[ ] SR re-enabled on R1/R2 with real prefix-SIDs
[ ] LDP removed from entire network — pure SR
[ ] Mapping server removed — real prefix-SIDs take over
[ ] Zero traffic loss throughout migration
```


router isis CORE
 net 49.0001.1720.1600.6006.00
 is-type level-2-only
 add ipv4 unic
 metric-style wide
 int lo0
 passive
 add ipv4 uni
 int gi0/0/0/1
 add ipv4 uni
 int gi0/0/0/2
 add ipv4 uni