# E16: 6PE / 6VPE

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

---

### Task 1: 6PE (Emerald — global IPv6 across IPv4 MPLS core)
1. IPv6 on E-R1 Gi0/0/0/0 (toward CE1) + E-R2 Gi0/0/0/2 (toward CE3). Core stays IPv4-only.
2. MP-BGP IPv6 unicast between PEs via RR E-R5. `send-label` (6PE).
3. Verify: CE1 IPv6 ↔ CE3 IPv6 via MPLS. P routers no IPv6 awareness.

### Task 2: 6VPE (Garnet — IPv6 in VRF)
1. VRF CUST_A on Gar-R1/Gar-R2: `address-family ipv6 / route-target`. VPNv6 unicast on RR Gar-R6.
2. IPv6 PE-CE on Gar-R1 Gi0/0/0/2 (CE4) and Gar-R2.
3. Verify: `show bgp vpnv6 unicast` — IPv6 VPN routes exchanged.

### Task 3: OSPFv3 PE-CE (CE6)
1. `router ospfv3` on Gar-R2 Gi0/0/0/0 (VRF CUST_D toward CE6). Redistribute into BGP.

## Checklist
```
[ ] 6PE: IPv6 across IPv4 MPLS core (Emerald, no IPv6 on P routers)
[ ] 6VPE: IPv6 in VRF (Garnet, vpnv6 via RR Gar-R6)
[ ] OSPFv3 PE-CE on CE6
```
