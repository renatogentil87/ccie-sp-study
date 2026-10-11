# E20: LDP → SR Migration (Emerald)

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 + E03 + E04

**End Goal:** Migrate Emerald from LDP+RSVP-TE to full SR. Zero LDP. Zero RSVP. IGP + SR only.

---

### Task 1: Enable SR on Emerald alongside LDP
1. Under IS-IS on all Emerald routers: `segment-routing mpls`. Prefix-SIDs: E-R1=1, E-R2=2, E-R3=3, E-R4=4, E-R6=5, E-R5=6.
2. Both LDP and SR labels exist in LFIB. Verify: `show mpls forwarding 2.2.2.2` — two entries (LDP + SR).

### Task 2: Prefer SR
1. `segment-routing mpls sr-prefer` (or under IS-IS: `address-family ipv4 / segment-routing mpls sr-prefer`).
2. Verify: SR label active, LDP label backup. Traffic uses SR.

### Task 3: Remove LDP gradually
1. Remove `mpls ldp` interface by interface (one at a time). After each: verify ping still works, LFIB still has SR labels.
2. After all removed: `show mpls ldp neighbor` — empty. SR provides all transport.

### Task 4: Enable TI-LFA (replaces RSVP-TE FRR)
1. `router isis CORE / address-family ipv4 / fast-reroute per-prefix`.
2. Verify: 100% coverage. Kill a link → sub-50ms failover. No backup tunnels needed.

### Task 5: Replace RSVP-TE tunnels with SR-TE policies
1. Remove RSVP-TE tunnel (from E03) on E-R1.
2. Create equivalent SR-TE policy: `segment-routing traffic-eng / policy / candidate-paths / explicit segment-list`.
3. Verify: same path, same traffic steering. Zero RSVP state.

### Task 6: Remove RSVP entirely
1. `no mpls traffic-eng` + `no rsvp` on all Emerald routers.
2. Verify: `show rsvp interface` — empty. `show mpls traffic-eng tunnels` — empty.
3. All 3 SPs now fully SR. One protocol (IGP + SR). Zero LDP. Zero RSVP.

---

## Section 2: Inter-AS LDP↔SR Interworking (Mapping Server)

> **Prerequisite:** E01 complete (Emerald running LDP, Garnet running SR-MPLS), E06 Sections 1-3 (inter-AS connectivity between Emerald↔Garnet).

**Scenario:** Before migrating Emerald (Tasks 1-6 above), Emerald still runs LDP only. Garnet runs SR-MPLS only. Inter-AS is working for VPN traffic via E06. But Garnet SR routers can't build a pure SR label path to Emerald PE loopbacks — because Emerald routers don't advertise prefix-SIDs.

The **mapping server** bridges the gap.

### Task 7: Understand the problem
1. On Gar-R1: `show isis segment-routing label table` — Garnet prefix-SIDs (16011-16017) present. No Emerald prefix-SIDs.
2. Gar-R1 wants to push an SR label to reach E-R1 (1.1.1.1). E-R1 has no prefix-SID — LDP only.
3. Without a prefix-SID for 1.1.1.1, Garnet's SR data plane can't build a label path to Emerald destinations.

### Task 8: Configure the mapping server
1. The mapping server creates **virtual prefix-SIDs** for Emerald's LDP-only loopbacks and advertises them into Garnet's IS-IS domain.
2. Configure on Gar-R6 (the RR/PCE — central, stable node):
   ```
   segment-routing
    mapping-server
     prefix-sid-map
      address-family ipv4
       1.1.1.1/32 index 101 range 1
       2.2.2.2/32 index 102 range 1
       3.3.3.3/32 index 103 range 1
       4.4.4.4/32 index 104 range 1
       5.5.5.5/32 index 105 range 1
       6.6.6.6/32 index 106 range 1
      !
     !
    !
   commit
   ```
3. Indexes 101-106 must NOT conflict with Garnet's own prefix-SIDs (11-17).
4. IS-IS floods these mappings to all Garnet routers automatically.

### Task 9: Verify mapping server advertisements
1. On Gar-R1:
   ```
   show isis segment-routing label table
   ```
   Should now include:
   ```
   16101   1.1.1.1/32    (from mapping server)
   16102   2.2.2.2/32    (from mapping server)
   16103   3.3.3.3/32
   16104   4.4.4.4/32
   16105   5.5.5.5/32
   16106   6.6.6.6/32
   16011   11.11.11.11/32  (local Garnet prefix-SID)
   16012   12.12.12.12/32  (local Garnet prefix-SID)
   ...
   ```
2. `show isis segment-routing prefix-sid-map active` — shows the mapping server entries.
3. These mapped SIDs are only meaningful within Garnet's IS-IS domain.

### Task 10: Test SR-labeled path to Emerald destinations
1. From Gar-R1: `show cef 1.1.1.1/32` — should show SR label 16101 toward Gar-R7 (ASBR).
2. At Gar-R7 (boundary): SR label popped, packet crosses to Emerald via inter-AS. Emerald forwards via LDP.
3. End-to-end: Gar-R1 → SR labels → Garnet core → Gar-R7 (boundary) → inter-AS → E-R6 → LDP labels → Emerald core → E-R1.
4. `traceroute 1.1.1.1 source 11.11.11.11` — SR labels in Garnet, LDP labels in Emerald.

### Task 11: Verify the reverse direction (LDP→SR works natively)
1. E-R1 wants to reach Gar-R1 (11.11.11.11). E-R1 runs LDP.
2. **No mapping server needed in this direction.** LDP works natively. E-R1 pushes LDP label → E-R6 (ASBR) → inter-AS → Gar-R7 → SR labels within Garnet → Gar-R1.
3. LDP doesn't care about prefix-SIDs. It just uses its own labels.
4. Verify: `traceroute 11.11.11.11 source 1.1.1.1` from E-R1 — LDP labels in Emerald, SR labels in Garnet.

### Task 12: Remove mapping server after Emerald migrates to SR
1. After completing Tasks 1-6 (Emerald now runs SR with real prefix-SIDs), the mapping server entries are redundant.
2. Emerald routers now advertise their own prefix-SIDs natively via inter-AS mechanisms.
3. Remove the mapping server config on Gar-R6:
   ```
   no segment-routing mapping-server
   commit
   ```
4. Verify: no traffic loss. Real prefix-SIDs replace the mapped ones.
5. The mapping server was a **temporary bridge** during LDP↔SR coexistence.

---

## Checklist
```
[ ] Section 1: Internal Migration
[ ] SR enabled alongside LDP (both labels in LFIB)
[ ] sr-prefer: SR active, LDP backup
[ ] LDP removed gradually (no outage)
[ ] TI-LFA: 100% protection, no backup tunnels
[ ] RSVP-TE tunnels replaced with SR-TE policies
[ ] RSVP removed entirely

[ ] Section 2: Inter-AS Mapping Server
[ ] Mapping server on Gar-R6: virtual prefix-SIDs 16101-16106 for Emerald loopbacks
[ ] All Garnet routers see mapped SIDs in label table
[ ] SR-labeled path from Gar-R1 to E-R1 works (SR in Garnet → LDP in Emerald)
[ ] Reverse direction (LDP→SR) works natively without mapping server
[ ] Mapping server removed after Emerald migrates to SR
[ ] All 3 SPs: full SR. Zero LDP. Zero RSVP.
```
