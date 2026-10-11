# SR-EX04: Flex-Algo

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 + SR-EX02 complete (all routers running IS-IS + SR with TI-LFA)

---

## Section 1: Flex-Algo 128 — Delay Metric

### Task 1
Define Flex-Algo 128 on one router (R3) with delay as the metric type. Advertise the definition so all IS-IS routers learn it.

### Task 2
Enable delay measurement on all core interfaces across all routers. IS-IS must advertise the measured delay per link.

### Task 3
Allocate a separate prefix-SID for Algo 128 on ALL routers (R1-R6). Use indexes that don't conflict with Algo 0 (e.g., R1=1128, R2=2128, R3=3128, R4=4128, R5=5128, R6=6128).

### Task 4
Verify: all routers participate in Flex-Algo 128. Separate SIDs for Algo 128 appear in the label table alongside the Algo 0 SIDs.

### Task 5
Test: traceroute from R1 to R6 using the Algo 0 prefix-SID (16006), then using the Algo 128 prefix-SID. Do they take different paths? If link delays differ, the delay-optimized path should differ from the IGP-metric path.

---

## Section 2: Flex-Algo 129 — Exclude Links

### Task 6
Define Flex-Algo 129 on the same definition router. Use IGP metric but exclude links tagged with affinity "EXPENSIVE."

### Task 7
Tag the R3↔R6 direct link as EXPENSIVE on both ends. This link should be excluded from Algo 129's topology.

### Task 8
Allocate prefix-SIDs for Algo 129 on all routers (e.g., R1=1129, R2=2129, etc.).

### Task 9
Verify: traceroute to R6 using Algo 129 SID. The path must NOT use the R3↔R6 direct link. Traffic should detour via R4/R5.

---

## Section 3: Selective Participation

### Task 10
Remove R4 from Algo 128 (remove its flex-algo 128 config and Algo 128 prefix-SID). Verify: Algo 128 topology routes around R4. R4 is excluded from that virtual topology.

### Task 11
Restore R4 to Algo 128. Verify it reappears in the topology.

### Task 12
Verify that TI-LFA computes backups per-algorithm: check the backup path for an Algo 128 prefix — it should use the Algo 128 topology, not the default topology.

---

## Snapshot
Take: **"SR-EX04-flexalgo"**

## Checklist
```
[ ] Flex-Algo 128 defined (delay metric) and advertised via IS-IS
[ ] Delay measurement enabled on all core interfaces
[ ] Per-algo prefix-SIDs allocated on all routers for Algo 128
[ ] All routers participating in Algo 128
[ ] Algo 0 vs Algo 128 traceroute shows different paths (if delays differ)
[ ] Flex-Algo 129 defined (exclude EXPENSIVE)
[ ] R3↔R6 link tagged EXPENSIVE on both ends
[ ] Algo 129 traceroute avoids R3↔R6 direct link
[ ] R4 removed from Algo 128 — traffic routes around it
[ ] R4 restored — back in Algo 128 topology
[ ] TI-LFA backup uses per-algo topology
```



 