# E13: EVPN Full

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 + E02

**End Goal:** Complete EVPN on Garnet. All 5 route types, multi-homing, IRB. CE5 dual-homed to Gar-R1(Gi0/0/0/0) + Gar-R2(Gi0/0/0/1).

---

### Task 1: EVPN-VPWS (Gar-R1↔Gar-R2)
1. EVI 100. `l2vpn / xconnect group / p2p` with EVPN signaling. BGP `address-family l2vpn evpn` toward RR Gar-R6.
2. Verify: `show evpn evi 100` — UP. Type 1 A-D route present.

### Task 2: Type 2 — MAC/IP advertisement
1. CE5 sends traffic with known MAC. Gar-R1 learns locally → advertises Type 2 to Gar-R6 (RR) → reflected to Gar-R2.
2. Verify: `show bgp l2vpn evpn route-type 2` — CE5's MAC visible. No flooding.

### Task 3: Type 3 — BUM handling
1. `show bgp l2vpn evpn route-type 3` — each PE advertises participation. Ingress replication for BUM.

### Task 4: Type 4 — DF election (CE5 dual-homed)
1. Same ESI on Gar-R1 (Gi0/0/0/0) + Gar-R2 (Gi0/0/0/1) toward CE5.
2. `show bgp l2vpn evpn route-type 4` — Gar-R1/Gar-R2 discover each other. `show evpn ethernet-segment` — DF elected.
3. Shut DF's link → DF shifts.

### Task 5: Type 5 — IP Prefix (IRB)
1. BVI (Bridge Virtual Interface) with IP in customer subnet on Gar-R1/Gar-R2.
2. `show bgp l2vpn evpn route-type 5` — IP prefix routes for inter-subnet routing.

### Task 6: All-active multi-homing
1. `ethernet-segment / esi 0000.0000.0000.0005.0001 / load-balancing-mode per-flow`.
2. Both PEs forward. Remote PEs load-balance to both (aliasing via Type 1).
3. Shut Gar-R1's link → Gar-R2 takes over all (mass withdrawal via Type 1). Sub-second.

### Task 7: Single-active multi-homing
1. Change to `single-active`. One PE active, other standby. Failover test.

### Task 8: MAC mobility
1. Simulate host move: same MAC appears on Gar-R2 instead of Gar-R1. Type 2 with higher sequence number. Gar-R1 withdraws stale.

### Task 9: ARP suppression
1. PE answers ARP locally from Type 2 knowledge. ARP broadcast doesn't flood to remote PEs.

## Checklist
```
[ ] EVPN-VPWS (Type 1)
[ ] Type 2 MAC/IP (no flooding)
[ ] Type 3 BUM (ingress replication)
[ ] Type 4 DF election (CE5 dual-homed ESI)
[ ] Type 5 IP prefix (IRB)
[ ] All-active + single-active multi-homing
[ ] MAC mobility (sequence number)
[ ] ARP suppression
```

---

## Section 6: EVPN across Gold + Garnet (Customer C)

### Task 10: Inter-AS EVPN
1. Customer C: CE5 (Garnet, dual-homed Gar-R1+Gar-R2) ↔ CE7 (Gold, G-R2).
2. BGP l2vpn evpn between Garnet RR (Gar-R6) and Gold RR (G-R4) — multihop eBGP or via ASBRs.
3. EVPN Type 2 (MAC/IP) exchanged across AS boundary.
4. CE5 ↔ CE7 at L2 across two SPs. Same VLAN100 service.
5. This tests EVPN inter-AS — the modern replacement for inter-AS VPLS.

### Updated Checklist (add)
```
[ ] Inter-AS EVPN: CE5 (Garnet) ↔ CE7 (Gold) in VLAN100
```
