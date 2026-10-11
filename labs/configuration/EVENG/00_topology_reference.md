# EVE-NG Lab Topology — Complete Reference (3 ISPs)

**Platform:** GNS3 on EC2 — IOS-XRv 9000 + Arista vEOS CEs
**NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3 (NIC0/NIC1=internal/mgmt)

---

## Three Autonomous Systems

| AS | Name | IGP | Transport | Routers |
|----|------|-----|-----------|---------|
| **65100** | **Emerald** | IS-IS L2 | LDP | E-R1, E-R2, E-R3, E-R4, E-R5, E-R6 |
| **65300** | **Gold** (transit) | IS-IS L2 | SRv6 | G-R1, G-R2, G-R3, G-R4, G-R5 |
| **65200** | **Garnet** | IS-IS L2 | SR-MPLS | Gar-R1, Gar-R2, Gar-R3, Gar-R4, Gar-R5, Gar-R6, Gar-R7 |

---

## Node Inventory (18 XRv + 9 CEs = 27 nodes)

### Emerald — AS 65100 (IS-IS + LDP)

| Hostname | R# | Role | Loopback |
|----------|-----|------|----------|
| E-R1 | R1 | PE | 1.1.1.1 |
| E-R2 | R2 | PE | 2.2.2.2 |
| E-R3 | R3 | P | 3.3.3.3 |
| E-R4 | R4 | P | 4.4.4.4 |
| E-R5 | R5 | P + RR + PCE | 5.5.5.5 |
| E-R6 | R6 | ASBR | 6.6.6.6 |

### Gold — AS 65300 (IS-IS + SRv6) — Transit Provider

| Hostname | R# | Role | Loopback IPv4 | Loopback IPv6 | SRv6 Locator |
|----------|-----|------|----------|--------------|-------------|
| G-R1 | R1 | PE | 21.21.21.21 | fc00::21/128 | fc00:0:21::/64 |
| G-R2 | R2 | PE | 22.22.22.22 | fc00::22/128 | fc00:0:22::/64 |
| G-R3 | R3 | P + RR + PCE | 23.23.23.23 | fc00::23/128 | fc00:0:23::/64 |
| G-R4 | R4 | ASBR | 24.24.24.24 | fc00::24/128 | fc00:0:24::/64 |
| G-R5 | R5 | ASBR | 25.25.25.25 | fc00::25/128 | fc00:0:25::/64 |

### Garnet — AS 65200 (IS-IS + SR-MPLS)

| Hostname | R# | Role | Loopback |
|----------|-----|------|----------|
| Gar-R1 | R1 | PE | 11.11.11.11 |
| Gar-R2 | R2 | PE | 12.12.12.12 |
| Gar-R3 | R3 | P | 13.13.13.13 |
| Gar-R4 | R4 | P | 14.14.14.14 |
| Gar-R5 | R5 | P | 15.15.15.15 |
| Gar-R6 | R6 | PCE + RR | 16.16.16.16 |
| Gar-R7 | R7 | ASBR | 17.17.17.17 |

### CEs (Arista vEOS)

| Node | ASN | Protocol | Connected To | Customer |
|------|-----|----------|-------------|----------|
| CE1 | 65012 | eBGP | E-R1 | Customer A |
| CE2 | 65012 | eBGP | E-R1 + E-R2 (dual-homed) | Customer A |
| CE3 | — | OSPF Area 0 | E-R2 | (Emerald only) |
| CE4 | 65013 | eBGP | Gar-R1 | Customer B |
| CE5 | — | EVPN VLAN100 | Gar-R1 + Gar-R2 (dual-homed) | Customer C |
| CE6 | — | OSPF Area 0 | Gar-R2 | (Garnet only) |
| CE7 | — | EVPN VLAN100 | G-R2 | Customer C |
| CE8 | 65012 | eBGP | G-R1 + G-R2 (dual-homed) | Customer A |
| CE9 | 65013 | eBGP | G-R1 | Customer B |

---

## Full Link Map

### Emerald Core

| From | NIC → Gi | To | NIC → Gi | Subnet |
|------|----------|-----|----------|--------|
| E-R5 | NIC5 → Gi3 | E-R4 | NIC5 → Gi3 | 10.1.1.0/24 |
| E-R4 | NIC4 → Gi2 | E-R6 | NIC4 → Gi2 | 10.1.2.0/24 |
| E-R4 | NIC3 → Gi1 | E-R3 | NIC3 → Gi1 | 10.1.3.0/24 |
| E-R3 | NIC4 → Gi2 | E-R1 | NIC4 → Gi2 | 10.1.4.0/24 |
| E-R3 | NIC2 → Gi0 | E-R2 | NIC2 → Gi0 | 10.1.5.0/24 |
| E-R1 | NIC5 → Gi3 | E-R2 | NIC5 → Gi3 | 10.1.6.0/24 |

### Gold Core (dual-stack — IPv6 required for SRv6)

| From | NIC → Gi | To | NIC → Gi | IPv4 Subnet | IPv6 Subnet |
|------|----------|-----|----------|-------------|-------------|
| G-R4 | NIC4 → Gi2 | G-R5 | NIC4 → Gi2 | 10.3.1.0/24 | fc00:3:1::/64 |
| G-R4 | NIC3 → Gi1 | G-R3 | NIC3 → Gi1 | 10.3.2.0/24 | fc00:3:2::/64 |
| G-R5 | NIC2 → Gi0 | G-R3 | NIC2 → Gi0 | 10.3.3.0/24 | fc00:3:3::/64 |
| G-R3 | NIC4 → Gi2 | G-R1 | NIC4 → Gi2 | 10.3.4.0/24 | fc00:3:4::/64 |
| G-R3 | NIC5 → Gi3 | G-R2 | NIC5 → Gi3 | 10.3.5.0/24 | fc00:3:5::/64 |

> **IPv6 convention:** `fc00:3:Y::/64` where 3=Gold, Y=link number. `::1`=first router, `::2`=second.

### Garnet Core

| From | NIC → Gi | To | NIC → Gi | Subnet |
|------|----------|-----|----------|--------|
| Gar-R6 | NIC5 → Gi3 | Gar-R3 | NIC5 → Gi3 | 10.2.1.0/24 |
| Gar-R7 | NIC4 → Gi2 | Gar-R3 | NIC4 → Gi2 | 10.2.2.0/24 |
| Gar-R3 | NIC2 → Gi0 | Gar-R4 | NIC2 → Gi0 | 10.2.3.0/24 |
| Gar-R3 | NIC3 → Gi1 | Gar-R5 | NIC3 → Gi1 | 10.2.4.0/24 |
| Gar-R4 | NIC5 → Gi3 | Gar-R5 | NIC5 → Gi3 | 10.2.5.0/24 |
| Gar-R4 | NIC3 → Gi1 | Gar-R1 | NIC3 → Gi1 | 10.2.6.0/24 |
| Gar-R5 | NIC4 → Gi2 | Gar-R2 | NIC4 → Gi2 | 10.2.7.0/24 |
| Gar-R1 | NIC5 → Gi3 | Gar-R2 | NIC5 → Gi3 | 10.2.8.0/24 |

### Inter-AS Links (3 links)

| From | NIC → Gi | To | NIC → Gi | Subnet | Path |
|------|----------|-----|----------|--------|------|
| E-R6 | NIC3 → Gi1 | Gar-R7 | NIC3 → Gi1 | 10.0.1.0/24 | Emerald↔Garnet DIRECT |
| E-R6 | NIC5 → Gi3 | G-R4 | NIC5 → Gi3 | 10.0.2.0/24 | Emerald↔Gold |
| G-R5 | NIC5 → Gi3 | Gar-R7 | NIC5 → Gi3 | 10.0.3.0/24 | Gold↔Garnet |

### PE-CE Links (Emerald)

| PE | NIC → Gi | CE | Subnet | Protocol |
|----|----------|-----|--------|----------|
| E-R1 | NIC2 → Gi0 | CE1 e0 | 192.168.1.0/24 | eBGP 65012 |
| E-R1 | NIC3 → Gi1 | CE2 e0 | 192.168.2.0/24 | eBGP 65012 |
| E-R2 | NIC3 → Gi1 | CE2 e1 | 192.168.3.0/24 | eBGP 65012 (dual-homed) |
| E-R2 | NIC4 → Gi2 | CE3 e0 | 192.168.4.0/24 | OSPF Area 0 |

### PE-CE Links (Gold)

| PE | NIC → Gi | CE | Subnet | Protocol |
|----|----------|-----|--------|----------|
| G-R1 | NIC2 → Gi0 | CE9 e0 | 192.168.11.0/24 | eBGP 65013 |
| G-R1 | NIC3 → Gi1 | CE8 e0 | 192.168.12.0/24 | eBGP 65012 |
| G-R2 | NIC3 → Gi1 | CE8 e1 | 192.168.13.0/24 | eBGP 65012 (dual-homed) |
| G-R2 | NIC2 → Gi0 | CE7 e0 | 192.168.14.0/24 | EVPN VLAN100 |

### PE-CE Links (Garnet)

| PE | NIC → Gi | CE | Subnet | Protocol |
|----|----------|-----|--------|----------|
| Gar-R1 | NIC4 → Gi2 | CE4 e0 | 172.16.1.0/24 | eBGP 65013 |
| Gar-R1 | NIC2 → Gi0 | CE5 e0 | 172.16.2.0/24 | EVPN VLAN100 |
| Gar-R2 | NIC3 → Gi1 | CE5 e1 | 172.16.3.0/24 | EVPN VLAN100 (dual-homed) |
| Gar-R2 | NIC2 → Gi0 | CE6 e0 | 172.16.4.0/24 | OSPF Area 0 |

---

## RR / PCE Design

| AS | RR + PCE | Loopback | Clients |
|----|----------|----------|---------|
| Emerald 65100 | E-R5 | 5.5.5.5 | E-R1, E-R2, E-R6 |
| Gold 65300 | G-R3 | 23.23.23.23 | G-R1, G-R2, G-R4, G-R5 |
| Garnet 65200 | Gar-R6 | 16.16.16.16 | Gar-R1, Gar-R2, Gar-R7 |

---

## Customer Design (spans multiple SPs)

| Customer | ASN | CEs | SPs | Protocol | Inter-AS needed? |
|----------|-----|-----|-----|----------|-----------------|
| **Customer A** | 65012 | CE1+CE2 (Emerald) + CE8 (Gold) | Emerald + Gold | eBGP | YES (Emerald↔Gold) |
| **Customer B** | 65013 | CE9 (Gold) + CE4 (Garnet) | Gold + Garnet | eBGP | YES (Gold↔Garnet) |
| **Customer C** | — | CE5 (Garnet) + CE7 (Gold) | Garnet + Gold | EVPN VLAN100 | YES (EVPN inter-AS) |
| CE3 | — | CE3 (Emerald only) | Emerald | OSPF | No |
| CE6 | — | CE6 (Garnet only) | Garnet | OSPF | No |

---

## Addressing Summary

| Range | Purpose |
|-------|---------|
| 1.1.1.1 – 6.6.6.6 | Emerald loopbacks (E-R1 to E-R6) |
| 11.11.11.11 – 17.17.17.17 | Garnet loopbacks (Gar-R1 to Gar-R7) |
| 21.21.21.21 – 25.25.25.25 | Gold loopbacks IPv4 (G-R1 to G-R5) |
| fc00::21 – fc00::25 | Gold loopbacks IPv6 |
| fc00:0:21::/64 – fc00:0:25::/64 | Gold SRv6 locators |
| fc00:3:1::/64 – fc00:3:5::/64 | Gold core links IPv6 |
| 10.1.x.0/24 | Emerald core links |
| 10.2.x.0/24 | Garnet core links |
| 10.3.x.0/24 | Gold core links IPv4 |
| 10.0.1.0/24 | Inter-AS: Emerald↔Garnet direct (E-R6↔Gar-R7) |
| 10.0.2.0/24 | Inter-AS: Emerald↔Gold (E-R6↔G-R4) |
| 10.0.3.0/24 | Inter-AS: Gold↔Garnet (G-R5↔Gar-R7) |
| 192.168.1-4.0/24 | Emerald PE-CE |
| 192.168.11-14.0/24 | Gold PE-CE |
| 172.16.1-4.0/24 | Garnet PE-CE |

---

## Old Name → New Name Mapping (for reference during transition)

| Old Name | New Name | Old Loopback | New Loopback |
|----------|----------|-------------|-------------|
| PE1 | E-R1 | 1.1.1.1 | 1.1.1.1 |
| PE2 | E-R2 | 2.2.2.2 | 2.2.2.2 |
| P1 | E-R3 | 3.3.3.3 | 3.3.3.3 |
| P2 | E-R4 | 4.4.4.4 | 4.4.4.4 |
| PCE1 | E-R5 | 5.5.5.5 | 5.5.5.5 |
| ASBR1 | E-R6 | 6.6.6.6 | 6.6.6.6 |
| PE5 | G-R1 | 24.24.24.24 | 21.21.21.21 |
| PE6 | G-R2 | 25.25.25.25 | 22.22.22.22 |
| P6 | G-R3 | 23.23.23.23 | 23.23.23.23 |
| ASBR3 | G-R4 | 21.21.21.21 | 24.24.24.24 |
| ASBR4 | G-R5 | 22.22.22.22 | 25.25.25.25 |
| PE3 | Gar-R1 | 11.11.11.11 | 11.11.11.11 |
| PE4 | Gar-R2 | 12.12.12.12 | 12.12.12.12 |
| P3 | Gar-R3 | 13.13.13.13 | 13.13.13.13 |
| P4 | Gar-R4 | 14.14.14.14 | 14.14.14.14 |
| P5 | Gar-R5 | 15.15.15.15 | 15.15.15.15 |
| PCE | Gar-R6 | 17.17.17.17 | 16.16.16.16 |
| ASBR2 | Gar-R7 | 16.16.16.16 | 17.17.17.17 |

---

## Key Design Scenarios This Topology Enables

- **Multi-hop inter-AS VPN** — Customer A: CE1(Emerald) → E-R6 → G-R4 → Gold core → G-R1 → CE8. Two AS boundaries.
- **Transit provider** — Gold provides transit between Emerald and Garnet. Direct (E-R6↔Gar-R7) or via Gold (E-R6↔G-R4↔G-R5↔Gar-R7).
- **Three transport technologies** — LDP (Emerald), SR-MPLS (Garnet), SRv6 (Gold).
- **EVPN across SPs** — Customer C: CE5 (Garnet) ↔ CE7 (Gold). EVPN inter-AS.
- **Dual-homed CEs** — CE2 (E-R1+E-R2), CE8 (G-R1+G-R2), CE5 (Gar-R1+Gar-R2).
- **BGP path diversity** — 3 inter-AS links, community-based traffic engineering.
