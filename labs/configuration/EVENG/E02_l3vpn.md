# E02: MPLS L3VPN

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 complete (IS-IS + LDP on Emerald, IS-IS + SR on Garnet).

**End Goal:** MP-BGP VPNv4 in all 3 SPs. VRFs, PE-CE eBGP + OSPF. End-to-end VPN within each AS.

---

## Customer-to-SP Mapping (from topology diagram)

| Customer | ASN | CEs | SP | VRF Name |
|----------|-----|-----|-----|----------|
| Customer A | 65012 | CE1 (E-R1), CE2 (E-R1+E-R2 dual-homed) | Emerald | CUST_A |
| Customer A | 65012 | CE8 (G-R1+G-R2 dual-homed) | Gold | CUST_A |
| Customer B | 65013 | CE9 (G-R1) | Gold | CUST_B |
| Customer B | 65013 | CE4 (Gar-R1) | Garnet | CUST_B |
| OSPF local | — | CE3 (E-R2) | Emerald | CUST_OSPF |
| OSPF local | — | CE6 (Gar-R2) | Garnet | CUST_OSPF |
| Customer C (EVPN) | — | CE7 (G-R2), CE5 (Gar-R1+Gar-R2) | Gold + Garnet | Done in E13 |

> **Note:** Customer A spans Emerald + Gold. Customer B spans Gold + Garnet. EVPN (Customer C) spans Gold + Garnet. Garnet does NOT have Customer A.

---

## Section 1: MP-BGP VPNv4 — Emerald

### Task 1: E-R5 as VPNv4 Route Reflector
1. `router bgp 65100` on E-R5 (5.5.5.5). VPNv4 AF. Neighbors E-R1(1.1.1.1), E-R2(2.2.2.2), E-R6(6.6.6.6) as RR clients. All `update-source Loopback0`.
2. Verify: `show bgp vpnv4 unicast summary` — sessions Established.

### Task 2: VRF CUST_A on E-R1 + E-R2 (Customer A, AS 65012)
1. `vrf CUST_A / rd 65100:100 / address-family ipv4 / import/export route-target 65012:100`.
2. E-R1: Gi0/0/0/0 → CUST_A (CE1, 192.168.1.0/24). Gi0/0/0/1 → CUST_A (CE2, 192.168.2.0/24).
3. E-R2: Gi0/0/0/1 → CUST_A (CE2, 192.168.3.0/24, dual-homed).
4. PE-CE eBGP: neighbors CE1/CE2 remote-as 65012. `as-override` (same ASN on both CEs).
5. Route-policy PASS-ALL in/out on all eBGP PE-CE neighbors (IOS-XR requirement).

### Task 3: VRF CUST_OSPF on E-R2 (CE3, OSPF local customer)
1. `vrf CUST_OSPF / rd 65100:200 / route-target 65100:200`.
2. E-R2: Gi0/0/0/2 → CUST_OSPF (CE3, 192.168.4.0/24, OSPF Area 0).
3. `router ospf 1 vrf CUST_OSPF / area 0 / interface Gi0/0/0/2`.
4. Redistribute OSPF into BGP under the VRF — so CE3's routes enter VPNv4 and reach the RR.
5. Do NOT redistribute BGP into OSPF yet — CE3 has no remote sites to reach at this stage. BGP → OSPF with prefix filtering is added in E06 Section 7 after inter-AS is working.

---

## Section 2: MP-BGP VPNv4 — Garnet

### Task 4: Gar-R6 as VPNv4 Route Reflector
1. `router bgp 65200` on Gar-R6 (16.16.16.16). VPNv4 AF. Neighbors Gar-R1(11.11.11.11), Gar-R2(12.12.12.12), Gar-R7(17.17.17.17) as RR clients.
2. Verify: `show bgp vpnv4 unicast summary` — sessions Established.

### Task 5: VRF CUST_B on Gar-R1 (Customer B, AS 65013)
1. `vrf CUST_B / rd 65200:100 / route-target 65013:100`.
2. Gar-R1: Gi0/0/0/2 → CUST_B (CE4, 172.16.1.0/24, eBGP 65013).
3. Route-policy PASS-ALL in/out.

### Task 6: VRF CUST_OSPF on Gar-R2 (CE6, OSPF local customer)
1. `vrf CUST_OSPF / rd 65200:200 / route-target 65200:200`.
2. Gar-R2: Gi0/0/0/0 → CUST_OSPF (CE6, 172.16.4.0/24, OSPF Area 0).
3. `router ospf 1 vrf CUST_OSPF / area 0 / interface Gi0/0/0/0`.
4. Redistribute OSPF into BGP under the VRF — so CE6's routes enter VPNv4 and reach the RR.
5. Do NOT redistribute BGP into OSPF yet — CE6 has no remote sites to reach at this stage. BGP → OSPF with prefix filtering is added in E06 Section 7 after inter-AS is working.

*(VRF for CE5 EVPN = Lab E13)*

---

## Section 3: MP-BGP VPNv4 — Gold

### Task 7: G-R3 as VPNv4 Route Reflector + PCE
1. `router bgp 65300` on G-R3 (23.23.23.23). VPNv4 AF. Neighbors G-R1(21.21.21.21), G-R2(22.22.22.22), G-R4(24.24.24.24), G-R5(25.25.25.25) as RR clients.
2. Verify: `show bgp vpnv4 unicast summary` — sessions Established.

### Task 8: VRF CUST_A on G-R1 + G-R2 (Customer A, AS 65012)
1. `vrf CUST_A / rd 65300:100 / route-target 65012:100`.
2. G-R1: Gi0/0/0/1 → CUST_A (CE8 e0, 192.168.12.0/24).
3. G-R2: Gi0/0/0/1 → CUST_A (CE8 e1, 192.168.13.0/24).
4. PE-CE eBGP AS 65012. `as-override` (CE8 same ASN as CE1/CE2 in Emerald).
5. Route-policy PASS-ALL in/out.

### Task 9: VRF CUST_B on G-R1 (Customer B, AS 65013)
1. `vrf CUST_B / rd 65300:200 / route-target 65013:100`.
2. G-R1: Gi0/0/0/0 → CUST_B (CE9 e0, 192.168.11.0/24, eBGP 65013).
3. Route-policy PASS-ALL in/out.

### Task 10: VRF CUST_C on G-R2 (CE7, EVPN VLAN100 — configured in E13)

---

## Section 4: Verification Within Each SP

### Task 11: Emerald verification
1. CE1 `ping` CE2 (via CUST_A) → **must succeed** (as-override needed because both use AS 65012).
2. CE1 cannot reach CE3 (different VRF) → **must fail** (VRF isolation).
3. `show bgp vrf CUST_A summary` — CE1 and CE2 sessions Established, prefixes received.
4. `show route vrf CUST_A` — customer routes present.

### Task 12: Garnet verification
1. CE4 reachable from Gar-R1 VRF CUST_B → **works**.
2. CE4 cannot reach CE6 (different VRF) → **must fail**.

### Task 13: Gold verification
1. CE8 reachable from G-R1 + G-R2 VRF CUST_A → **works** (dual-homed, both paths).
2. CE9 reachable from G-R1 VRF CUST_B → **works**.
3. CE8 cannot reach CE9 (different VRF) → **must fail**.

### Task 14: Cross-AS not yet
1. CE1 (Emerald) cannot reach CE8 (Gold) → fails. Customer A spans two SPs but inter-AS not configured yet → E06.
2. CE9 (Gold) cannot reach CE4 (Garnet) → fails. Customer B spans two SPs → E06.

---

## RT Design Notes

The route-targets are designed for inter-AS in E06:

| Customer | RT | Who imports | Why |
|----------|-----|------------|-----|
| Customer A | 65012:100 | Emerald PEs + Gold PEs | Same RT across both SPs → inter-AS Option C will "just work" once VPNv4 is exchanged |
| Customer B | 65013:100 | Gold PEs + Garnet PEs | Same RT across both SPs |
| OSPF (Emerald) | 65100:200 | Emerald PEs only | Local to Emerald, no inter-AS needed |
| OSPF (Garnet) | 65200:200 | Garnet PEs only | Local to Garnet, no inter-AS needed |

> **RD differs per SP** (65100:100 vs 65300:100 for CUST_A). **RT is the same** (65012:100) across SPs for the same customer. RD provides VPNv4 route uniqueness, RT drives import. This is critical for inter-AS — the same RT ensures routes are imported into the correct VRF regardless of which SP originated them.

---

## CCIE+ Challenges
1. **SoO** on CE2 (dual-homed E-R1+E-R2): `set extcommunity soo 65012:902` inbound on both PEs. Prevents routing loop.
2. **SoO** on CE8 (dual-homed G-R1+G-R2): `set extcommunity soo 65012:908` inbound on both PEs.
3. **RT-Constraint** on E-R5 (RR): `address-family rtfilter unicast` + activate toward PE clients. RR only sends VPN routes the PE actually needs.
4. **BGP PIC**: `bgp additional-paths install` on E-R1/E-R2 under VPNv4. Pre-install backup path for fast failover.
5. **Maximum-prefix**: `neighbor <CE> maximum-prefix 50 80 restart 5`. Protect PE from CE advertising too many routes.

---

## Snapshot
Take: **"E02-l3vpn-working"**

## Checklist
```
[ ] Emerald: VPNv4 sessions E-R1/E-R2 ↔ RR E-R5 (Established)
[ ] Emerald: CUST_A on E-R1+E-R2 (CE1+CE2, AS 65012); CUST_OSPF on E-R2 (CE3)
[ ] Emerald: CE1↔CE2 ping works (as-override); CE1↛CE3 fails (VRF isolation)
[ ] Garnet: VPNv4 sessions Gar-R1/Gar-R2 ↔ RR Gar-R6 (Established)
[ ] Garnet: CUST_B on Gar-R1 (CE4, AS 65013); CUST_OSPF on Gar-R2 (CE6)
[ ] Gold: VPNv4 sessions G-R1/G-R2 ↔ RR G-R3 (Established)
[ ] Gold: CUST_A on G-R1+G-R2 (CE8, AS 65012); CUST_B on G-R1 (CE9, AS 65013)
[ ] CE8 dual-homed: both paths visible in show bgp vrf CUST_A
[ ] Cross-AS not working yet (Customer A: CE1↛CE8, Customer B: CE9↛CE4) → E06
[ ] Route-policy PASS-ALL applied on all eBGP PE-CE neighbors
[ ] (CCIE+) SoO on dual-homed CEs, RT-Constraint, PIC, max-prefix
```
