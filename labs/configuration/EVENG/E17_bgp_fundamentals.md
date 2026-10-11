# E17: BGP Fundamentals + Peering

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

---

### Task 1: eBGP PE-CE + iBGP via RRs
1. Verify all eBGP (PE↔CE) and iBGP (PE↔RR) sessions in all 3 SPs. Next-hop-self on RRs.

### Task 2: Path selection on CE2 (dual-homed E-R1+E-R2)
1. Weight (local), LOCAL_PREF (AS-wide), AS-PATH prepend, MED. Demonstrate each.

### Task 3: Community schema + outbound policy
1. Tag customer routes 65100:100; tag peer routes 65100:200 on RR E-R5.
2. Outbound to E-R6: only advertise customer routes (not peer-learned).

### Task 4: Dampening + conditional advertisement + max-prefix
1. Dampening on E-R6 for Garnet routes. Conditional default to CE1. Max-prefix on PE-CE.

### Task 5: Confederations
1. Split Emerald into sub-AS 65101 (E-R1/E-R2/E-R3) + 65102 (E-R4/E-R6/E-R5). Confederation ID 65100.
2. Verify: LP/MED preserved across boundary. CEs see only 65100.

### Task 6: Inter-AS eBGP at ASBRs
1. E-R6 (Gi0/0/0/0) ↔ Gar-R7 (Gi0/0/0/1). eBGP 65100↔65200.

### Task 7: Graceful Shutdown (RFC 8326)
1. GRACEFUL_SHUTDOWN community → LP 0 → drain traffic before maintenance.

## Checklist
```
[ ] eBGP + iBGP working (all 3 SPs)
[ ] Path selection demonstrated (weight→LP→AS-PATH→MED)
[ ] Community outbound policy (customers only to peer)
[ ] Dampening, conditional advert, max-prefix
[ ] Confederations (Emerald split)
[ ] Inter-AS eBGP at E-R6↔Gar-R7
[ ] Graceful Shutdown
```
