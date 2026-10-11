# E12: SR-TE Advanced — PCE + Flex-Algo + ODN

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E04

**End Goal:** SR-PCE, stateful Gar-R6, Flex-Algo, ODN, bandwidth-on-demand on Garnet.

---

### Task 1: SR-PCE on Gar-R6 node (16.16.16.16)
1. `pce / address ipv4 17.17.17.17 / segment-routing`. PCCs: Gar-R1, Gar-R2.
2. On PCCs: `segment-routing traffic-eng / pcc / source-address / pce address ipv4 17.17.17.17`.
3. Verify: `show pce ipv4 peer` — sessions from Gar-R1/Gar-R2.

### Task 2: Stateful PCE + delegation
1. Gar-R6: `stateful-client / instantiation / delegate`. PCCs: `pcc / report / delegate`.
2. Verify: `show pce lsp` — all LSPs visible to Gar-R6.
3. Delegate a policy from Gar-R1 → Gar-R6 modifies path proactively.

### Task 3: PCE-initiated LSP
1. Gar-R6 creates tunnel on Gar-R1 without local config. Verify on Gar-R1: policy shows "Initiated by Gar-R6".

### Task 4: Flex-Algo 128 (delay metric)
1. On Gar-R3 (definition source): `router isis CORE / flex-algo 128 / metric-type delay / advertise-definition`.
2. Per-algo prefix-SIDs on all Garnet routers (index +100 for algo 128).
3. Verify: `show isis flex-algo 128` — definition advertised. Independent path computation.

### Task 5: ODN + BGP color
1. `segment-routing traffic-eng / on-demand color 128 / dynamic / metric type latency / sid-algorithm 128`.
2. VPN route with color 128 arrives → policy auto-created using Algo 128. Remove → auto-deleted.

### Task 6: Redundant Gar-R6
1. Configure E-R5 (Emerald, 5.5.5.5) as backup Gar-R6 for Garnet PCCs.
2. Failover: shut Gar-R6 → PCCs switch to E-R5. Restore → revert.

## Checklist
```
[ ] SR-PCE on Gar-R6 node; PCEP sessions from Gar-R1/Gar-R2
[ ] Stateful Gar-R6: delegation, proactive path modification
[ ] Gar-R6-initiated LSP on Gar-R1
[ ] Flex-Algo 128 (delay metric, per-algo SIDs)
[ ] ODN: auto-create policy per BGP color
[ ] Redundant Gar-R6 failover
```
