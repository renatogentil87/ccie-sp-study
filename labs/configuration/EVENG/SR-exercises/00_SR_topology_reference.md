# SR Lab Topology — Reference

**Platform:** IOS-XRv 9000 + Arista vEOS CEs
**NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**AS:** 65100 (single AS for all exercises except SR-EX06)

---

## Routers

| Router | Loopback | Role |
|--------|----------|------|
| R1 | 172.16.1.1/32 | PE (CE1 attached) |
| R2 | 172.16.2.2/32 | PE (CE1 dual-homed) |
| R3 | 172.16.3.3/32 | P (central hub) |
| R4 | 172.16.4.4/32 | P |
| R5 | 172.16.5.5/32 | P |
| R6 | 172.16.6.6/32 | PE (CE2 attached) |
| CE1 | 10.1.1.1/32 | Customer AS 65001 (dual-homed R1+R2) |
| CE2 | 10.2.2.2/32 | Customer AS 65002 (single-homed R6) |

## Links

| From | NIC → Gi | To | NIC → Gi | Subnet |
|------|----------|-----|----------|--------|
| R1 | NIC2 → Gi0 | R3 | NIC2 → Gi0 | 10.0.13.0/24 |
| R1 | NIC5 → Gi3 | R2 | NIC5 → Gi3 | 10.0.12.0/24 |
| R2 | NIC2 → Gi0 | R4 | NIC2 → Gi0 | 10.0.24.0/24 |
| R3 | NIC3 → Gi1 | R6 | NIC3 → Gi1 | 10.0.36.0/24 |
| R3 | NIC4 → Gi2 | R4 | NIC4 → Gi2 | 10.0.34.0/24 |
| R3 | NIC5 → Gi3 | R5 | NIC5 → Gi3 | 10.0.35.0/24 |
| R4 | NIC3 → Gi1 | R5 | NIC3 → Gi1 | 10.0.45.0/24 |
| R5 | NIC4 → Gi2 | R6 | NIC4 → Gi2 | 10.0.56.0/24 |

## PE-CE Links

| PE | NIC → Gi | CE | Subnet |
|----|----------|-----|--------|
| R1 | NIC3 → Gi1 | CE1 e1 | 192.168.1.0/24 |
| R2 | NIC3 → Gi1 | CE1 e2 | 192.168.2.0/24 |
| R6 | NIC2 → Gi0 | CE2 e1 | 192.168.6.0/24 |

## IS-IS NET-IDs (Area 1)

| Router | NET |
|--------|-----|
| R1 | 49.0001.1720.1600.1001.00 |
| R2 | 49.0001.1720.1600.2002.00 |
| R3 | 49.0001.1720.1600.3003.00 |
| R4 | 49.0001.1720.1600.4004.00 |
| R5 | 49.0001.1720.1600.5005.00 |
| R6 | 49.0001.1720.1600.6006.00 |
