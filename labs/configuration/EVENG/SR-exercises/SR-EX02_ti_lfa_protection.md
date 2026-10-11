# SR-EX02: Classic LFA + TI-LFA (Link, Node, SRLG Protection)

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 complete

---

## Section 1: Classic LFA

### Task 1
Enable classic LFA (per-prefix fast-reroute) under IS-IS on all routers. Do NOT enable TI-LFA yet.

### Task 2
Check classic LFA coverage: what percentage of prefixes have a backup path? Which prefixes are unprotected and why?

### Task 3
Test classic LFA: continuous ping from R4 to R6. Shut the primary link on the path. How many packets lost? Does classic LFA protect this prefix?

---

## Section 2: TI-LFA Link Protection

### Task 4
Enable TI-LFA on all routers under IS-IS. This replaces classic LFA with a smarter backup computation.

### Task 5
Compare TI-LFA coverage vs classic LFA coverage. Is it higher? Check which prefixes now have backup paths that didn't before.

### Task 6
Verify backup paths in the FIB: pick a prefix and check that both primary and backup next-hop are pre-installed with repair segment labels.

### Task 7
Test: continuous ping R4 → R6. Shut the primary link. Count packet loss — should be 0-1 (sub-50ms). Compare with classic LFA result from Task 3.

---

## Section 3: TI-LFA Node Protection

### Task 8
Configure TI-LFA preference (tiebreakers):
- Node-protecting: highest priority (lowest index)
- SRLG-disjoint: second priority
- Link-protecting (default): fallback

### Task 9
Verify: for a specific prefix, confirm the backup is node-protecting (bypasses the entire next-hop node, not just the link).

### Task 10
Test node failure: continuous ping R3 → R6 through R5. Shut ALL interfaces on R5 (simulating full node failure). Does traffic reroute around R5 entirely?

### Task 11
Compare the repair segment list for link-protecting vs node-protecting backup for the same prefix. How do they differ?

---

## Section 4: SRLG Protection

### Task 12
Define an SRLG: the links R3↔R5 and R4↔R5 share the same fiber duct. Assign the same SRLG value on both links, on both ends of each link (4 interface configs total).

### Task 13
Verify: for a prefix where the primary uses one of the SRLG links, confirm the backup path avoids BOTH SRLG member links.

### Task 14
Test: shut the R3↔R5 link. Verify the backup does NOT use R4↔R5 (same SRLG). Traffic must take an entirely SRLG-disjoint path.

---

## Section 5: TI-LFA Preference Experiment

### Task 15
Change tiebreaker to prefer link protection over node protection (lower index on default). Verify the backup changes from node-protecting to link-protecting.

### Task 16
Restore to standard SP preference: node-protecting first, SRLG second, link-protecting last. Confirm with show commands.

### Task 17
Final check: TI-LFA coverage should be close to 100% across the SR domain.

---

## Snapshot
Take: **"SR-EX02-tilfa"**

## Checklist
```
[ ] Classic LFA enabled, coverage percentage noted
[ ] Classic LFA test: packet loss counted during link failure
[ ] TI-LFA enabled, coverage higher than classic LFA
[ ] Backup paths with repair segments visible in FIB
[ ] TI-LFA link failure test: 0-1 packet loss
[ ] Node protection preference configured
[ ] Node-protecting backup verified for a prefix
[ ] Node failure test: traffic reroutes around dead node
[ ] SRLG defined on R3↔R5 and R4↔R5 (same value, both ends)
[ ] SRLG-disjoint backup verified
[ ] Tiebreaker experiment: link vs node preference tested
[ ] Standard SP preference restored
[ ] ~100% TI-LFA coverage confirmed
```
