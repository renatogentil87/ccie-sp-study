# E10: Multicast VPN — All Profiles

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

**End Goal:** All CCIE SP mVPN profiles. Source CE1→E-R1, receiver CE4→Gar-R1.

---

### Task 1: Profile 0 — GRE MDT + PIM (Emerald)
1. Enable PIM sparse-mode on Emerald core. RP = E-R4 (4.4.4.4). VRF CUST_A: `mdt default 239.1.1.1`.
2. Verify: MDT tunnel, multicast flows CE1→CE4 (within Emerald first).

### Task 2: Profile 3 — GRE + BGP MVPN AD + PIM signaling
1. `address-family ipv4 mvpn` on PEs + RR E-R5. BGP Type 1 AD replaces static MDT membership. PIM still handles C-joins.

### Task 3: Profile 11 — GRE + BGP AD + BGP C-signaling
1. `mdt overlay use-bgp`. C-multicast joins via BGP Type 7 (not PIM between PEs). Verify Type 5 (Source Active) + Type 7 (C-Join).

### Task 4: Profile 12 — mLDP P2MP (Garnet, no GRE, no core PIM)
1. `mpls mldp` on Garnet core. `mdt default mpls mldp <Gar-R1-loopback>`. P routers: ZERO PIM state.
2. Verify: `show mpls mldp database` — P2MP tree. `show pim neighbor` on P routers — empty.

### Task 5: Profile 14 — Partitioned MDT mLDP P2MP
1. `mdt partitioned mldp p2mp`. Each PE roots its own tree. Only interested PEs join.

### Task 6: Data MDT
1. `mdt data mpls mldp 100 / threshold 10`. High-bw stream → data MDT. Verify S-PMSI AD (Type 3).

### Task 7: Inter-AS mVPN (after E06 inter-AS is configured)
1. Source CE1 (Emerald) → Receiver CE4 (Garnet). BGP AD Type 2 across ASBR boundary.

## Checklist
```
[ ] Profile 0: GRE + PIM (Emerald)
[ ] Profile 3: GRE + BGP AD + PIM signaling
[ ] Profile 11: GRE + BGP AD + BGP C-signaling
[ ] Profile 12: mLDP P2MP (Garnet, no core PIM)
[ ] Profile 14: Partitioned MDT
[ ] Data MDT with threshold
[ ] Inter-AS mVPN concept
```
