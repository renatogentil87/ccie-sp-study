# CCIE SP Workbook 01 — IS-IS Foundation

**Domain:** Core Routing (25% of CCIE SP lab)
**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `../../configuration/EVENG/00_topology_reference.md` — Emerald AS 65100 + Gold AS 65300 + Garnet AS 65200 (3 ISPs)
**Format:** ccielabpass DOO — every task is **Question → Solution → Verification**

> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3 (NIC0/NIC1 = internal/mgmt)

---

## Topology Quick-Reference (used by every task)

### Node inventory + NET-ID derivation

NET-ID format: `49.<area>.<system-id>.00`. The **system-id** is derived from the Loopback0 IPv4 address by padding each octet to 3 digits and regrouping into three 4-hex-digit blocks (the classic "IP-to-NET" method).

Example — E-R1 Loopback0 `1.1.1.1` → `001.001.001.001` → `0010.0100.1001`.

| SP (area) | Node | Role | Loopback0 | Derived NET-ID |
|-----------|------|------|-----------|----------------|
| **Emerald** 49.0001 | E-R1 | PE | 1.1.1.1 | `49.0001.0010.0100.1001.00` |
| | E-R2 | PE | 2.2.2.2 | `49.0001.0020.0200.2002.00` |
| | E-R3 | P | 3.3.3.3 | `49.0001.0030.0300.3003.00` |
| | E-R4 | P | 4.4.4.4 | `49.0001.0040.0400.4004.00` |
| | E-R6 | ASBR | 5.5.5.5 | `49.0001.0050.0500.5005.00` |
| | E-R5 | RR+Gar-R6 | 6.6.6.6 | `49.0001.0060.0600.6006.00` |
| **Garnet** 49.0002 | Gar-R1 | PE | 11.11.11.11 | `49.0002.0110.1101.1011.00` |
| | Gar-R2 | PE | 12.12.12.12 | `49.0002.0120.1201.2012.00` |
| | Gar-R3 | P | 13.13.13.13 | `49.0002.0130.1301.3013.00` |
| | Gar-R4 | P | 14.14.14.14 | `49.0002.0140.1401.4014.00` |
| | Gar-R5 | P | 15.15.15.15 | `49.0002.0150.1501.5015.00` |
| | Gar-R7 | ASBR | 17.17.17.17 | `49.0002.0160.1601.6016.00` |
| | Gar-R6 | RR+Gar-R6 | 16.16.16.16 | `49.0002.0170.1701.7017.00` |
| **Gold** 49.0003 | G-R4 | ASBR+RR+Gar-R6 | 24.24.24.24 | `49.0003.0210.2102.1021.00` |
| | G-R5 | ASBR | 25.25.25.25 | `49.0003.0220.2202.2022.00` |
| | G-R3 | P | 23.23.23.23 | `49.0003.0230.2302.3023.00` |
| | G-R1 | PE | 21.21.21.21 | `49.0003.0240.2402.4024.00` |
| | G-R2 | PE | 22.22.22.22 | `49.0003.0250.2502.5025.00` |

### Per-router core (IS-IS) interfaces — from the link map

| SP | Node | Core Gi interfaces (run IS-IS) | Non-core (NO IS-IS) |
|----|------|--------------------------------|---------------------|
| Emerald | E-R1 | Gi0/0/0/2 (→E-R3), Gi0/0/0/3 (→E-R2) | Gi0/0/0/0 (CE1), Gi0/0/0/1 (CE2) |
| Emerald | E-R2 | Gi0/0/0/0 (→E-R3), Gi0/0/0/3 (→E-R1) | Gi0/0/0/1 (CE2), Gi0/0/0/2 (CE3) |
| Emerald | E-R3 | Gi0/0/0/0 (→E-R2), Gi0/0/0/1 (→E-R4), Gi0/0/0/2 (→E-R1) | — |
| Emerald | E-R4 | Gi0/0/0/1 (→E-R3), Gi0/0/0/2 (→E-R6), Gi0/0/0/3 (→E-R5) | — |
| Emerald | E-R6 | Gi0/0/0/2 (→E-R4) | Gi0/0/0/1 (inter-AS Garnet), Gi0/0/0/3 (inter-AS Gold) |
| Emerald | E-R5 | Gi0/0/0/3 (→E-R4) | — |
| Garnet | Gar-R1 | Gi0/0/0/1 (→Gar-R4), Gi0/0/0/3 (→Gar-R2) | Gi0/0/0/2 (CE4), Gi0/0/0/0 (CE5) |
| Garnet | Gar-R2 | Gi0/0/0/2 (→Gar-R5), Gi0/0/0/3 (→Gar-R1) | Gi0/0/0/1 (CE5), Gi0/0/0/0 (CE6) |
| Garnet | Gar-R3 | Gi0/0/0/0 (→Gar-R4), Gi0/0/0/1 (→Gar-R5), Gi0/0/0/2 (→Gar-R7), Gi0/0/0/3 (→Gar-R6) | — |
| Garnet | Gar-R4 | Gi0/0/0/0 (→Gar-R3), Gi0/0/0/1 (→Gar-R1), Gi0/0/0/3 (→Gar-R5) | — |
| Garnet | Gar-R5 | Gi0/0/0/1 (→Gar-R3), Gi0/0/0/2 (→Gar-R2), Gi0/0/0/3 (→Gar-R4) | — |
| Garnet | Gar-R7 | Gi0/0/0/2 (→Gar-R3) | Gi0/0/0/1 (inter-AS Emerald), Gi0/0/0/3 (inter-AS Gold) |
| Garnet | Gar-R6 | Gi0/0/0/3 (→Gar-R3) | — |
| Gold | G-R4 | Gi0/0/0/1 (→G-R3), Gi0/0/0/2 (→G-R5) | Gi0/0/0/3 (inter-AS Emerald) |
| Gold | G-R5 | Gi0/0/0/0 (→G-R3), Gi0/0/0/2 (→G-R4) | Gi0/0/0/3 (inter-AS Garnet) |
| Gold | G-R3 | Gi0/0/0/0 (→G-R5), Gi0/0/0/1 (→G-R4), Gi0/0/0/2 (→G-R1), Gi0/0/0/3 (→G-R2) | — |
| Gold | G-R1 | Gi0/0/0/2 (→G-R3) | Gi0/0/0/0 (CE9), Gi0/0/0/1 (CE8) |
| Gold | G-R2 | Gi0/0/0/3 (→G-R3) | Gi0/0/0/0 (CE7), Gi0/0/0/1 (CE8) |

### Core link subnets (for interface addressing)

| SP | Link | Subnet | Addresses |
|----|------|--------|-----------|
| Emerald | E-R5(Gi3)–E-R4(Gi3) | 10.1.1.0/24 | E-R5 .6, E-R4 .4 |
| Emerald | E-R4(Gi2)–E-R6(Gi2) | 10.1.2.0/24 | E-R4 .4, E-R6 .5 |
| Emerald | E-R4(Gi1)–E-R3(Gi1) | 10.1.3.0/24 | E-R4 .4, E-R3 .3 |
| Emerald | E-R3(Gi2)–E-R1(Gi2) | 10.1.4.0/24 | E-R3 .3, E-R1 .1 |
| Emerald | E-R3(Gi0)–E-R2(Gi0) | 10.1.5.0/24 | E-R3 .3, E-R2 .2 |
| Emerald | E-R1(Gi3)–E-R2(Gi3) | 10.1.6.0/24 | E-R1 .1, E-R2 .2 |
| Garnet | Gar-R6(Gi3)–Gar-R3(Gi3) | 10.2.1.0/24 | Gar-R6 .17, Gar-R3 .13 |
| Garnet | Gar-R7(Gi2)–Gar-R3(Gi2) | 10.2.2.0/24 | Gar-R7 .16, Gar-R3 .13 |
| Garnet | Gar-R3(Gi0)–Gar-R4(Gi0) | 10.2.3.0/24 | Gar-R3 .13, Gar-R4 .14 |
| Garnet | Gar-R3(Gi1)–Gar-R5(Gi1) | 10.2.4.0/24 | Gar-R3 .13, Gar-R5 .15 |
| Garnet | Gar-R4(Gi3)–Gar-R5(Gi3) | 10.2.5.0/24 | Gar-R4 .14, Gar-R5 .15 |
| Garnet | Gar-R4(Gi1)–Gar-R1(Gi1) | 10.2.6.0/24 | Gar-R4 .14, Gar-R1 .11 |
| Garnet | Gar-R5(Gi2)–Gar-R2(Gi2) | 10.2.7.0/24 | Gar-R5 .15, Gar-R2 .12 |
| Garnet | Gar-R1(Gi3)–Gar-R2(Gi3) | 10.2.8.0/24 | Gar-R1 .11, Gar-R2 .12 |
| Gold | G-R4(Gi2)–G-R5(Gi2) | 10.3.1.0/24 | G-R4 .21, G-R5 .22 |
| Gold | G-R4(Gi1)–G-R3(Gi1) | 10.3.2.0/24 | G-R4 .21, G-R3 .23 |
| Gold | G-R5(Gi0)–G-R3(Gi0) | 10.3.3.0/24 | G-R5 .22, G-R3 .23 |
| Gold | G-R3(Gi2)–G-R1(Gi2) | 10.3.4.0/24 | G-R3 .23, G-R1 .24 |
| Gold | G-R3(Gi3)–G-R2(Gi3) | 10.3.5.0/24 | G-R3 .23, G-R2 .25 |

> **IPv6 core convention (Task 1.5):** mirror the IPv4 subnet as `2001:db8:1:X::/64` (Emerald), `2001:db8:2:X::/64` (Garnet), `2001:db8:3:X::/64` (Gold), host bits = the router's loopback octet. Loopback0 IPv6 = `2001:db8::<loop>/128` (e.g., E-R1 = `2001:db8::1/128`).

---

# Section 1 — IS-IS Basic Setup (8 tasks)

---

## Task 1.1 — Enable IS-IS L2-only on all Emerald routers

### Question
Configure IS-IS process **CORE** as **Level-2-only** on all six Emerald routers (E-R1, E-R2, E-R3, E-R4, E-R6, E-R5). Derive each NET-ID from the router's Loopback0 (`49.0001.xxxx.xxxx.xxxx.00`). Use **metric-style wide**. Run IS-IS on Loopback0 and the core interfaces only (see the per-router table); do **not** enable it on PE-CE or inter-AS interfaces.

### Solution

**E-R1 (1.1.1.1):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0010.0100.1001.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**E-R2 (2.2.2.2):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0020.0200.2002.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**E-R3 (3.3.3.3):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0030.0300.3003.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**E-R4 (4.4.4.4):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0040.0400.4004.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**E-R6 (5.5.5.5)** — only Gi0/0/0/2 is core (Gi1/Gi3 are inter-AS):
```
router isis CORE
 is-type level-2-only
 net 49.0001.0050.0500.5005.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**E-R5 (6.6.6.6):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0060.0600.6006.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

### Verification
```
show isis protocol            ! IS-Type = Level-2-only; NET matches; metric-style wide (level-2)
show isis interface brief     ! only core + Loopback0 listed; Loopback0 = passive
show isis neighbors           ! E-R1↔E-R3, E-R1↔E-R2, E-R3↔E-R4, E-R3↔E-R2, E-R4↔E-R6, E-R4↔E-R5 — State Up, type L2
show route isis               ! all 6 Emerald loopbacks (1.1.1.1–6.6.6.6) present as i L2
show isis database level 2    ! 6 LSPs (one per router), ATT/OL bits clear
ping 6.6.6.6 source Loopback0 ! from E-R1 — end-to-end IGP reachability
```
- **Look for:** every core link shows an L2 adjacency in `Up`; no adjacency on PE-CE/inter-AS interfaces; loopbacks appear as `i L2` (wide metrics, not the 63 legacy cap).

---

## Task 1.2 — Enable IS-IS L2-only on all Garnet routers

### Question
Same as 1.1 but for the seven Garnet routers (Gar-R1, Gar-R2, Gar-R3, Gar-R4, Gar-R5, Gar-R7, Gar-R6) using area **49.0002**. NET-ID derived from Loopback0, `metric-style wide`, core interfaces only.

### Solution

**Gar-R1 (11.11.11.11):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0110.1101.1011.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R2 (12.12.12.12):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0120.1201.2012.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R3 (13.13.13.13):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0130.1301.3013.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R4 (14.14.14.14):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0140.1401.4014.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R5 (15.15.15.15):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0150.1501.5015.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R7 (17.17.17.17)** — only Gi0/0/0/2 core:
```
router isis CORE
 is-type level-2-only
 net 49.0002.0160.1601.6016.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Gar-R6 (16.16.16.16):**
```
router isis CORE
 is-type level-2-only
 net 49.0002.0170.1701.7017.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

### Verification
```
show isis neighbors            ! Gar-R3 has 4 L2 neighbors (Gar-R4, Gar-R5, Gar-R7, Gar-R6); Gar-R1↔Gar-R4, Gar-R1↔Gar-R2, Gar-R2↔Gar-R5
show route isis                ! all 7 Garnet loopbacks (11.x–17.x) as i L2
show isis database level 2     ! 7 LSPs
ping 12.12.12.12 source Loopback0   ! from Gar-R1 — Gar-R1↔Gar-R2 reachable through the core
```
- **Look for:** Gar-R3 is the hub (4 adjacencies); no adjacency on Gar-R7 Gi1/Gi3 (inter-AS) or PE Gi0/Gi-CE interfaces.

---

## Task 1.3 — Enable IS-IS L2-only on all Gold routers

### Question
Same pattern for the five Gold routers (G-R4, G-R5, G-R3, G-R1, G-R2), area **49.0003**.

### Solution

**G-R4 (24.24.24.24)** — Gi1/Gi2 core (Gi3 inter-AS):
```
router isis CORE
 is-type level-2-only
 net 49.0003.0210.2102.1021.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**G-R5 (25.25.25.25)** — Gi0/Gi2 core (Gi3 inter-AS):
```
router isis CORE
 is-type level-2-only
 net 49.0003.0220.2202.2022.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**G-R3 (23.23.23.23):**
```
router isis CORE
 is-type level-2-only
 net 49.0003.0230.2302.3023.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

**G-R1 (21.21.21.21):**
```
router isis CORE
 is-type level-2-only
 net 49.0003.0240.2402.4024.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**G-R2 (22.22.22.22):**
```
router isis CORE
 is-type level-2-only
 net 49.0003.0250.2502.5025.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
 !
!
```

### Verification
```
show isis neighbors           ! G-R3 has 4 L2 neighbors (G-R5, G-R4, G-R1, G-R2); G-R4↔G-R5 direct
show route isis               ! 5 Gold loopbacks (21.x–25.x) as i L2
ping 22.22.22.22 source Loopback0    ! from G-R1 — G-R1↔G-R2 via G-R3
```
- **Look for:** no adjacency across G-R4 Gi3 / G-R5 Gi3 (inter-AS links stay bare until BGP in a later workbook).

---

## Task 1.4 — CCIE-ISIS flexible configuration group (regex interface matching)

### Question
Rather than repeat interface knobs on every router, build a **flexible CLI configuration group** named `CCIE-ISIS` that, via a **regex** matching all `GigabitEthernet` interfaces under `router isis CORE`, applies: L2 metric **200** (IPv4) and **400** (IPv6), **hello-padding disable**, **point-to-point**; and, matching `Loopback`, sets **passive**. Apply the group and confirm the inherited config.

### Solution
IOS-XR flexible config groups live at the top level and are applied per-instance with `apply-group`. Regex tokens use `'..*'`.

```
group CCIE-ISIS
 router isis 'CORE'
  interface 'GigabitEthernet.*'
   point-to-point
   hello-padding disable
   address-family ipv4 unicast
    metric 200
   !
   address-family ipv6 unicast
    metric 400
   !
  !
  interface 'Loopback.*'
   passive
   address-family ipv4 unicast
   !
   address-family ipv6 unicast
   !
  !
 !
end-group
!
router isis CORE
 apply-group CCIE-ISIS
!
```

> **Note:** with the group applied, the explicit `point-to-point` / `metric` lines under each interface become redundant — the group supplies them by regex. Keep interface stanzas minimal (just `interface X / address-family …`) and let the group inherit the knobs. `hello-padding disable` stops full-MTU padding of IS-IS hellos (faster adjacency, but see Task 3.2 — padding is what catches MTU mismatches).

### Verification
```
show running-config group CCIE-ISIS               ! group body present
show running-config router isis CORE inheritance  ! shows metric 200/400, point-to-point, hello-padding disable INHERITED per Gi; passive on Loopback0
show isis interface GigabitEthernet0/0/0/2 detail  ! Metric (L2): 200 (v4) / 400 (v6); Hello padding: disabled; Circuit type: p2p
show isis interface GigabitEthernet0/0/0/2 | i "Metric|padding|point"
```
- **Look for:** the `inheritance` output tags each value with the group name — proves the regex matched. Metrics change from default 10 to 200/400. Re-run `show isis neighbors` to confirm adjacencies survived the metric/padding change.

---

## Task 1.5 — IPv6 address-family with single-topology (all 3 IS-IS instances)

### Question
Enable the **IPv6 unicast** address-family in all three `router isis CORE` instances using **single-topology** (IPv6 shares the IPv4 SPF/topology). Add IPv6 addressing on Loopback0 and all core interfaces. Show the config on one router per SP; the pattern repeats.

### Solution
Single-topology is signalled with `single-topology` under the **IPv6** address-family, and requires `metric-style wide` (already set). Example on **E-R1** (Emerald), **Gar-R1** (Garnet), **G-R1** (Gold) — apply the same shape to every node:

**E-R1 (Emerald):**
```
interface Loopback0
 ipv6 address 2001:db8::1/128
!
interface GigabitEthernet0/0/0/2
 ipv6 address 2001:db8:1:4::1/64
!
interface GigabitEthernet0/0/0/3
 ipv6 address 2001:db8:1:6::1/64
!
router isis CORE
 address-family ipv6 unicast
  metric-style wide
  single-topology
 !
 interface Loopback0
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/3
  address-family ipv6 unicast
 !
!
```

**Gar-R1 (Garnet):**
```
interface Loopback0
 ipv6 address 2001:db8::11/128
!
router isis CORE
 address-family ipv6 unicast
  metric-style wide
  single-topology
 !
 interface Loopback0
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/3
  address-family ipv6 unicast
 !
!
```

**G-R1 (Gold):**
```
interface Loopback0
 ipv6 address 2001:db8::24/128
!
router isis CORE
 address-family ipv6 unicast
  metric-style wide
  single-topology
 !
 interface Loopback0
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv6 unicast
 !
!
```

> Repeat on every node: add the IPv6 address per the IPv6 core convention, add `address-family ipv6 unicast` under the process (with `single-topology`) and under each core interface + Loopback0. The `CCIE-ISIS` group from 1.4 already supplies the IPv6 metric (400).

### Verification
```
show isis topology                     ! single IS-IS topology; no separate MT-IPv6 topology listed
show isis ipv6 route                   ! IPv6 loopbacks reachable
show route ipv6                        ! 2001:db8::/… loopbacks as i L2
show isis interface Gi0/0/0/2 | i "Topology|IPv6"   ! Topology: IPv4+IPv6 single
ping ipv6 2001:db8::6 source Loopback0 ! from E-R1
```
- **Look for:** exactly **one** topology in `show isis topology` (single-topology). If you accidentally left multi-topology on, you'd see `IPv4 Unicast` and `IPv6 Unicast` as separate topologies — that is Task 2.3's contrast.

---

## Task 1.6 — set-overload-bit on-startup 180 level-2 (all routers)

### Question
On **every** router in all three SPs, set the IS-IS overload bit on startup for **180 seconds** at **level-2**, so the node is not used as transit until it has fully converged (BGP, LDP/SR, LSDB). Show the config; it is identical on all nodes.

### Solution
```
router isis CORE
 set-overload-bit on-startup 180 level 2
!
```
> Optional CCIE-grade variant that waits for BGP to converge instead of a fixed timer:
> ```
> router isis CORE
>  set-overload-bit on-startup wait-for-bgp level 2
> !
> ```
> The task asks for the fixed **180 level-2** form; use that. In IOS-XR the level is expressed as `level 2` (not `level-2`).

### Verification
```
show isis database <own-system-id> detail | i "Overload|LSP"   ! within 180s of reload: OL bit SET on the L2 LSP
show isis database level 2 | i OL                              ! nodes still in startup show (OL); after 180s clears
show isis protocol | i "overload|Overload"                     ! "set-overload-bit on-startup for 180 secs (Level 2)"
```
- **Look for:** immediately after `reload`/process restart the router's own L2 LSP carries the overload bit; other routers route **around** it (verify a transit path avoids it during startup); after 180 s the OL bit clears and normal transit resumes. On a running box (no recent restart) the bit is not set — that is expected.

---

## Task 1.7 — LDP auto-config under IS-IS (Emerald only)

### Question
Emerald uses **LDP** for transport (Gold=SRv6, Garnet=SR-MPLS — no LDP there). Enable **LDP IGP auto-config** under the Emerald IS-IS instance so LDP is enabled automatically on every IS-IS core interface. Configure MPLS LDP on the six Emerald routers.

### Solution
Auto-config is set under the IS-IS **interface** (or globally) with `mpls ldp auto-config`, plus a base `mpls ldp` stanza. Example on **E-R1**; repeat on E-R2, E-R3, E-R4, E-R6, E-R5 (each with its own router-id = Loopback0).

**E-R1:**
```
mpls ldp
 router-id 1.1.1.1
 address-family ipv4
 !
!
router isis CORE
 address-family ipv4 unicast
  mpls ldp auto-config
 !
!
```
> `mpls ldp auto-config` under the IPv4 AF enables LDP on all interfaces IS-IS is running on. Because E-R6 only runs IS-IS on Gi0/0/0/2, LDP comes up only there — never on the inter-AS links (correct: no LDP toward Garnet/Gold).

**Router-ids to use:** E-R1=1.1.1.1, E-R2=2.2.2.2, E-R3=3.3.3.3, E-R4=4.4.4.4, E-R6=5.5.5.5, E-R5=6.6.6.6.

### Verification
```
show mpls ldp discovery         ! hellos on every Emerald core interface only (auto-config)
show mpls ldp neighbor brief    ! LDP sessions == IS-IS adjacencies (6 links)
show mpls forwarding            ! labels bound for all 6 Emerald loopbacks
show mpls interfaces            ! LDP enabled on core Gi; NOT on PE-CE / inter-AS
traceroute mpls ipv4 6.6.6.6/32 ! from E-R1 — PUSH at E-R1, SWAP mid, POP (PHP) at E-R5 neighbor
```
- **Look for:** LDP neighbor count matches IS-IS neighbor count on each router; **no** LDP on Garnet/Gold (confirm `show mpls ldp neighbor` returns nothing there).

---

## Task 1.8 — LDP–IGP synchronization (Emerald)

### Question
Prevent traffic blackholing when an LDP session is down but the IS-IS adjacency is up: enable **LDP–IGP synchronization** at **level-2** on Emerald. Until LDP synchronizes on a link, IS-IS should advertise that link with **max metric** so it isn't used as a labeled transit path.

### Solution
In IOS-XR, LDP-IGP sync is enabled under the IS-IS process (per address-family / per-level), with an optional holddown.

**On each Emerald router:**
```
router isis CORE
 address-family ipv4 unicast
  mpls ldp sync level 2
 !
!
mpls ldp
 igp sync delay 10
!
```
> `mpls ldp sync level 2` ties the L2 IGP metric to LDP readiness on each interface. Until the session forms, IS-IS advertises the max wide metric (16777214) for that link. `igp sync delay 10` gives LDP 10 s after the interface/adjacency comes up before IS-IS may restore the normal metric.

### Verification
```
show isis interface Gi0/0/0/2 | i "LDP sync|Sync"    ! "LDP Sync: Enabled / Achieved"
show mpls ldp igp sync                               ! per-interface: Sync achieved: Yes
show isis interface Gi0/0/0/2 | i Metric             ! normal metric (200) once synced
```
Break-test:
```
! shut LDP on one end:  mpls ldp / interface Gi0/0/0/2 / (or clear the session)
show isis interface Gi0/0/0/2 | i "LDP sync|Metric"  ! Sync: NOT achieved; L2 metric jumps to 16777214 (max)
show route 6.6.6.6/32                                 ! path shifts to an alternate labeled link, not the unsynced one
```
- **Look for:** while LDP is down the affected link carries **max metric** and IS-IS steers around it; when LDP recovers (after the delay) the metric returns to 200 and the path restores. No unlabeled transit through the core during the outage.

---

# Section 2 — IS-IS Advanced (6 tasks)

---

## Task 2.1 — IS-IS authentication (key-chain, HMAC-MD5), per-level + per-interface (Emerald)

### Question
Secure Emerald IS-IS with **HMAC-MD5** authentication using a **key-chain**. Apply authentication two ways: **per-level** (LSP/SNP authentication for the whole L2 domain) and **per-interface** (hello authentication on a specific link). All Emerald routers must share the key.

### Solution
Define the key-chain first (identical on every Emerald router), then reference it under the process (LSP/SNP, per-level) and optionally override on an interface (hello).

**Key-chain (all Emerald routers):**
```
key chain ISIS-KEY
 key 1
  accept-lifetime 00:00:00 january 01 2020 infinite
  key-string cisco123
  send-lifetime 00:00:00 january 01 2020 infinite
  cryptographic-algorithm HMAC-MD5
 !
!
```

**Per-level (process-wide L2 LSP/SNP authentication) — all Emerald routers:**
```
router isis CORE
 lsp-password keychain ISIS-KEY level 2
!
```

**Per-interface (hello authentication) — example E-R1↔E-R2 link (both ends):**
```
router isis CORE
 interface GigabitEthernet0/0/0/3
  hello-password keychain ISIS-KEY level 2
 !
!
```

> `lsp-password` authenticates LSPs/SNPs (the database exchange) at the level; `hello-password` authenticates IIH hellos on the circuit. Both must match on every participating router/interface or adjacencies/LSPs are rejected (see Task 3.2).

### Verification
```
show key chain ISIS-KEY                       ! key 1, HMAC-MD5, lifetimes infinite
show isis interface Gi0/0/0/3 | i "authentication|password"   ! Hello authentication: enabled (keychain)
show isis neighbors                           ! adjacency STILL Up after both ends configured
show isis database level 2 detail | i Auth    ! LSPs carry authentication TLV
debug isis authentication                     ! (lab only) no "authentication failure" messages
```
Negative test:
```
! change key-string on ONE router -> its adjacency drops / LSPs rejected -> revert to restore
```
- **Look for:** adjacencies remain Up only when keys match on both ends; a mismatch drops the hello adjacency (interface auth) or rejects LSPs (lsp-password).

---

## Task 2.2 — IS-IS prefix suppression on core P-P links

### Question
Reduce the LSDB and speed SPF by **suppressing transit (P-to-P core link) prefixes** from IS-IS — advertise only loopbacks. Configure prefix suppression on all routers (global) and confirm core /24s no longer appear as IS-IS routes, while loopbacks still do.

### Solution
IOS-XR supports both a global "suppress all connected except loopback" via `advertise passive-only`, and per-interface `prefix suppression`. Use the global form (cleanest for "advertise only loopbacks"):

**All routers (per SP instance):**
```
router isis CORE
 address-family ipv4 unicast
  advertise passive-only
 !
 address-family ipv6 unicast
  advertise passive-only
 !
!
```
> `advertise passive-only` advertises prefixes **only** from passive interfaces (Loopback0 is passive from Task 1.1/1.4). All active point-to-point core links are suppressed. Alternative per-link form: under an interface `address-family ipv4 unicast / prefix-suppression`.

### Verification
```
show route isis | i "10.1.|10.2.|10.3."   ! core /24 transit links NO LONGER present as i L2
show route isis | i "/32"                  ! all loopbacks still present
show isis database <system-id> detail      ! LSP advertises only the loopback prefix(es), no link /24s
ping 6.6.6.6 source Loopback0              ! reachability intact (loopbacks are what LDP/BGP need)
```
- **Look for:** transit link subnets gone from the routing table, loopbacks retained. LSPs shrink (fewer IP-reachability TLVs). Connectivity to loopbacks is unaffected — this is safe because forwarding uses loopbacks/labels, not the link subnets.

---

## Task 2.3 — Multi-topology vs single-topology (explain + configure single)

### Question
Explain **when** to use IS-IS multi-topology (MT) versus single-topology, then confirm all three instances run **single-topology** (as set in Task 1.5).

### Solution — explanation

| | Single-topology | Multi-topology (MT) |
|---|-----------------|---------------------|
| **SPF** | One SPF for IPv4 + IPv6 (shared metrics, shared next-hops) | Separate SPF per address-family (independent metrics/paths) |
| **Requirement** | IPv6 must be enabled on the **same** interfaces as IPv4; topologies must be congruent | IPv4 and IPv6 topologies may differ |
| **Config knob** | `single-topology` under IPv6 AF | (default when omitted) MT — uses MT TLVs (#222/#229) |
| **Use when** | IPv4 and IPv6 run on **all the same links** with the same metrics (our lab) | Partial IPv6 deployment, different IPv6 metrics/paths, or IPv6 links that IPv4 doesn't use |
| **Risk** | If a link is IPv4-only, IPv6 SPF still assumes it's usable → blackhole | More TLVs / complexity, but correct on incongruent topologies |

**Rule of thumb:** congruent dual-stack core → single-topology (simpler, fewer TLVs). Incongruent (IPv6 only on some links, or different TE for v6) → multi-topology.

**Config (already applied in 1.5) — confirm on every instance:**
```
router isis CORE
 address-family ipv6 unicast
  single-topology
 !
!
```

### Verification
```
show isis topology                    ! ONE topology (single); NOT "IPv4 Unicast" + "IPv6 Unicast" separately
show isis database detail | i "MT|Topology"   ! no MT TLVs (222/229) present in single-topology
show isis interface Gi0/0/0/2 | i Topology    ! Topology: standard (single)
```
- **Look for:** a single topology list. If you removed `single-topology` you'd see two topologies and MT TLVs appear — that visual difference is the exam giveaway.

---

## Task 2.4 — Route leaking between levels (concept — N/A for L2-only)

### Question
Explain IS-IS inter-level **route leaking**, why it exists, and why it is **not applicable** in this lab (everything is Level-2-only).

### Solution — explanation
- **Default behavior:** L1 routers know only their area + a default route to the nearest L1/L2 (ATT bit). L2 (backbone) routes are **not** leaked down into L1 by default, to keep L1 LSDBs small. L1→L2 *is* automatic (L1/L2 injects L1 prefixes up into L2).
- **Route leaking (RFC 3277 / `domain-wide prefix distribution`)** lets an **L1/L2** router leak **specific L2 prefixes down into L1**, so an L1 router can pick the optimal L1/L2 exit instead of always following the ATT default (which can cause sub-optimal routing / traffic tromboning).
- **IOS-XR config (reference only):** under the process, `propagate level 2 into level 1 route-policy LEAK` (with a route-policy selecting which L2 prefixes to leak). The reverse (`level 1 into level 2`) is on by default.
- **Down-bit:** leaked routes set the up/down bit so an L1/L2 router won't re-advertise them back up into L2 (loop prevention).

**Why N/A here:** this topology is **Level-2-only** in every SP (Task 1.1–1.3). There is no Level-1, so there are no levels to leak between. Each SP is a single flat L2 domain; inter-SP reachability is handled by **BGP across the ASBRs**, not by IS-IS level leaking.

> **When you *would* use it:** a large SP split into IS-IS areas (L1 access regions + L2 backbone) where access routers need optimal egress to specific backbone/PE loopbacks rather than the ATT default.

### Verification (concept — nothing to configure)
```
show isis protocol | i "IS-Type"    ! Level-2-only -> confirms no L1, so no leaking applies
```
- **Look for:** `IS-Type: Level-2-only` on all nodes → documents *why* the task is N/A. (No `propagate` statements exist or are needed.)

---

## Task 2.5 — IS-IS BFD for fast adjacency failure detection (all core links)

### Question
Enable **BFD** for IS-IS on all core links in all three SPs so link/neighbor failures are detected in milliseconds instead of waiting for IS-IS hold timers. Use 300 ms intervals, multiplier 3 (≈900 ms detection).

### Solution
In IOS-XR, BFD-for-IS-IS is enabled per interface under the IS-IS process, with BFD timers set there too. Example on **E-R3** (Emerald); apply to every core interface on every router:

```
router isis CORE
 interface GigabitEthernet0/0/0/0
  bfd minimum-interval 300
  bfd multiplier 3
  bfd fast-detect ipv4
  bfd fast-detect ipv6
 !
 interface GigabitEthernet0/0/0/1
  bfd minimum-interval 300
  bfd multiplier 3
  bfd fast-detect ipv4
  bfd fast-detect ipv6
 !
 interface GigabitEthernet0/0/0/2
  bfd minimum-interval 300
  bfd multiplier 3
  bfd fast-detect ipv4
  bfd fast-detect ipv6
 !
!
```
> Both ends of each link must enable BFD. `bfd fast-detect ipv4`/`ipv6` arms it for each AF (single-topology still uses both). This can be folded into the `CCIE-ISIS` group from Task 1.4 (add the `bfd …` lines under the `'GigabitEthernet.*'` regex) so it lands on every core interface automatically.

### Verification
```
show bfd session                          ! one UP session per core link, State UP, Echo/Async
show bfd session detail | i "Interval|Multiplier|Detect"   ! Tx/Rx 300ms, mult 3, detect ~900ms
show isis interface Gi0/0/0/0 | i BFD      ! BFD: Enabled
show isis neighbors detail | i BFD         ! neighbor BFD state Up
```
Break-test:
```
! pull/shut the far-end link; time the reconvergence
show isis neighbors           ! adjacency drops in <1s (BFD), not after the 30s hold
```
- **Look for:** BFD sessions Up on every core link; on failure the IS-IS adjacency tears down in sub-second time (BFD detection) versus multi-second IS-IS hold expiry.

---

## Task 2.6 — IS-IS mesh-group on fully-meshed segments (reduce flooding)

### Question
On a **fully-meshed** set of routers, LSP flooding is redundant (every router floods to every neighbor). Configure an IS-IS **mesh-group** to suppress redundant flooding. Apply it to the Emerald E-R1–E-R2–E-R3 near-mesh (E-R1↔E-R2, E-R1↔E-R3, E-R2↔E-R3 all adjacent).

### Solution
Mesh-group is a per-interface setting; all interfaces in the same mesh-group number belong to one group, and LSPs received on one member are **not** re-flooded to other members of the same group (blocked), cutting redundant copies. Use the same group number on the meshed interfaces.

**E-R1** — interfaces facing E-R3 (Gi0/0/0/2) and E-R2 (Gi0/0/0/3):
```
router isis CORE
 interface GigabitEthernet0/0/0/2
  mesh-group 10
 !
 interface GigabitEthernet0/0/0/3
  mesh-group 10
 !
!
```
**E-R2** — interfaces facing E-R3 (Gi0/0/0/0) and E-R1 (Gi0/0/0/3):
```
router isis CORE
 interface GigabitEthernet0/0/0/0
  mesh-group 10
 !
 interface GigabitEthernet0/0/0/3
  mesh-group 10
 !
!
```
**E-R3** — interfaces facing E-R2 (Gi0/0/0/0), E-R4 (Gi0/0/0/1), E-R1 (Gi0/0/0/2). Put only the meshed pair (toward E-R1/E-R2) in the group; keep the E-R4 uplink OUT of the mesh-group so LSPs still propagate to the rest of the core:
```
router isis CORE
 interface GigabitEthernet0/0/0/0
  mesh-group 10
 !
 interface GigabitEthernet0/0/0/2
  mesh-group 10
 !
!
```

> **Caution (exam trap):** a mesh-group blocks re-flooding between its members. It is only safe when the members are **fully meshed** (every member reaches every other directly), otherwise an LSP can be blocked from reaching a router that had no other path → LSDB inconsistency. Never put a router's *only* uplink to the rest of the domain into the mesh-group (that's why E-R3's Gi0/0/0/1 → E-R4 is excluded).

### Verification
```
show isis interface Gi0/0/0/2 | i "Mesh"      ! Mesh Group: 10
show isis database level 2                     ! LSDB identical on E-R1, E-R2, E-R3 (consistency preserved)
show isis lsp-log                              ! fewer flooding events on meshed links
```
Consistency test:
```
! force an LSP change (e.g., toggle a passive loopback), confirm all three routers' LSDBs still converge identically
show isis database <changed-system-id> detail  ! same seq number on E-R1/E-R2/P1
```
- **Look for:** mesh-group shown on the meshed interfaces; LSDB stays consistent across the mesh (no missing LSPs) while redundant flooding drops. If any LSP goes missing on a member, the mesh isn't truly full — remove the group from the offending interface.

---

# Section 3 — IS-IS Verification & Troubleshooting (4 tasks)

---

## Task 3.1 — Full verification checklist

### Question
Produce a repeatable IS-IS health-check using the four core show commands and state what "good" looks like for each.

### Solution / Reference commands (run per SP instance)
```
show isis adjacency               ! or: show isis neighbors
show isis database                ! and: show isis database level 2 detail
show isis route                   ! or: show route isis
show isis topology                ! and: show isis topology level 2
```
Supporting:
```
show isis protocol
show isis interface brief
show isis statistics
```

### Verification — what to look for
- **`show isis adjacency` / `show isis neighbors`** — every core link has a neighbor in **`Up`** state, System Type **L2**, correct Hold time counting down and refreshing. Count = number of core links on that router. Any `Init` = one-way hello (see 3.2).
- **`show isis database` (level 2 detail)** — one LSP per router (fragment `-00`), sequence numbers incrementing slowly (stable), **checksum OK**, no `(OL)` overload flag in steady state (except during Task 1.6 startup), correct list of IP-reachability TLVs (loopbacks only, per Task 2.2), authentication TLV present (Task 2.1). LSDB **identical** on all routers in the SP.
- **`show isis route` / `show route isis`** — all remote loopbacks present as `i L2`, with the expected wide metric (sum of link metrics = multiples of 200), correct next-hop interface, ECMP where paths are equal-cost.
- **`show isis topology`** — every router (system-id + hostname via dynamic hostname TLV) reachable, single topology (Task 1.5/2.3), metric to each node matches the routing table.

**Look for red flags:** adjacency stuck `Init`, missing LSP, mismatched LSDB between routers, unexpected OL bit, loopback missing from `show route isis`, IPv4/IPv6 shown as two topologies (means MT crept in).

---

## Task 3.2 — Troubleshooting: adjacency stuck in INIT

### Question
An IS-IS adjacency will not reach `Up` — it sits in **`Init`** (or flaps). Diagnose the three classic causes: **MTU mismatch**, **area/level mismatch**, **authentication mismatch**. Show how to identify and fix each.

### Solution — diagnosis and fixes

**Symptom:**
```
show isis neighbors            ! State = Init (hellos received but not bidirectional) or Down/flapping
show isis adjacency detail     ! "Nbr adjacency state: Init"
```

**Cause A — MTU mismatch.** IS-IS pads hellos to full MTU (unless `hello-padding disable`, Task 1.4). If the two ends have different MTUs, padded hellos are dropped one direction → stuck Init.
```
show isis interface Gi0/0/0/2 | i "MTU|padding"    ! compare both ends
show interface Gi0/0/0/2 | i MTU
```
Fix — make MTUs match (or, as a diagnostic, temporarily disable padding to see if the adjacency forms):
```
interface GigabitEthernet0/0/0/2
 mtu 1514           ! match the far end's L2 MTU
!
! diagnostic only:
router isis CORE
 interface GigabitEthernet0/0/0/2
  hello-padding disable
 !
!
```

**Cause B — area / level / circuit-type mismatch.** L2-only ↔ L1-only will never form (no common level). Different areas are OK for **L2** (L2 is inter-area) but **not** for L1. A p2p-vs-broadcast (network type) mismatch also breaks it.
```
show isis protocol | i "IS-Type|Area|NET"     ! both must have a Level-2 in common
show isis interface Gi0/0/0/2 | i "Circuit|type"   ! both point-to-point
```
Fix — align is-type and circuit type:
```
router isis CORE
 is-type level-2-only
 interface GigabitEthernet0/0/0/2
  point-to-point
 !
!
```

**Cause C — authentication mismatch.** Hello-password mismatch (Task 2.1) rejects IIHs → Init.
```
debug isis adj-packets Gi0/0/0/2    ! "authentication failure" / "auth type mismatch"
show isis interface Gi0/0/0/2 | i auth
```
Fix — same key-string + algorithm + key-id on both ends:
```
key chain ISIS-KEY
 key 1
  key-string cisco123
  cryptographic-algorithm HMAC-MD5
 !
!
```

### Verification (after fix)
```
show isis neighbors            ! State = Up
show isis adjacency detail     ! "Up", hold time refreshing
show logging | i ADJCHANGE     ! "New adjacency ... Up"
```
- **Look for:** transition Init→Up. Confirm the previously-missing loopback now appears in `show route isis`.

---

## Task 3.3 — Troubleshooting: routes not appearing

### Question
Adjacencies are `Up` but some prefixes are missing from the routing table. Diagnose the two classic causes: **passive-interface misconfiguration** and a **core interface missing from IS-IS**.

### Solution — diagnosis and fixes

**Cause A — interface missing under IS-IS.** A core link (or a loopback) was never added to the IS-IS process → its prefix isn't advertised and/or no adjacency forms on it.
```
show isis interface brief            ! is the expected interface listed at all?
show run router isis CORE            ! is `interface Gi0/0/0/x` present with an address-family?
show isis neighbors                  ! adjacency missing on that link
```
Fix — add the interface (and its address-family):
```
router isis CORE
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Cause B — passive misconfiguration.** Two failure modes:
1. A **core** interface was mistakenly set **passive** → IS-IS advertises its subnet but sends **no hellos** → no adjacency → downstream loopbacks unreachable over that link.
2. **Loopback0 is NOT passive / not in IS-IS** → the loopback /32 isn't advertised (and with `advertise passive-only` from Task 2.2, a non-passive loopback won't be advertised at all).
```
show isis interface Gi0/0/0/2 | i "State|passive|Passive"   ! core link should be Active, not Passive
show isis interface Loopback0 | i passive                    ! loopback SHOULD be passive
show route isis | i /32                                      ! is the missing loopback here?
```
Fix — core interfaces active, loopback passive:
```
router isis CORE
 interface GigabitEthernet0/0/0/2
  no passive            ! core link must NOT be passive (needs hellos)
 !
 interface Loopback0
  passive               ! loopback SHOULD be passive (advertise, no hellos)
  address-family ipv4 unicast
 !
!
```

### Verification (after fix)
```
show isis neighbors               ! adjacency now Up on the previously-passive core link
show route isis                   ! the missing loopback(s) now present as i L2
show isis database <sys-id> detail ! LSP now includes the loopback /32 reachability TLV
ping <missing-loopback> source Loopback0
```
- **Look for:** the corrected interface flips from Passive→Active (core) and the loopback appears in the LSDB and route table. Cross-check against Task 2.2 (`advertise passive-only`) — the loopback must be passive to be advertised under that policy.

---

## Task 3.4 — IS-IS convergence timing (SPF / LSP intervals, sub-second tuning)

### Question
Tune IS-IS for **sub-second convergence**: exponential **SPF backoff**, **LSP generation** backoff, and **LSP flooding/refresh** timers. Explain each timer and give an aggressive-but-stable set for the lab core.

### Solution
IOS-XR exponential backoff timers (all in **milliseconds**): `<initial-wait> <secondary-wait> <maximum-wait>`. Start fast, back off under churn.

**On every router (per SP instance):**
```
router isis CORE
 lsp-gen-interval maximum-wait 5000 initial-wait 50 secondary-wait 200
 spf-interval    maximum-wait 5000 initial-wait 50 secondary-wait 200
 !
 address-family ipv4 unicast
  ! (optional) fast-reroute per-prefix ti-lfa   -> covered in the SR workbook
 !
 lsp-refresh-interval 65000
 max-lsp-lifetime 65535
!
```
Interface hello/hold (pair BFD from Task 2.5 for the real speed):
```
router isis CORE
 interface GigabitEthernet0/0/0/2
  hello-interval 3
  hello-multiplier 3        ! hold = 3 x 3 = 9s (BFD does the sub-second detection)
 !
!
```

**Timer meanings:**
- **`spf-interval`** — how soon SPF runs after a topology change. `initial-wait 50` = first SPF 50 ms after the first change; `secondary-wait 200` = spacing if changes keep arriving; `maximum-wait 5000` = ceiling under sustained churn (damping).
- **`lsp-gen-interval`** — how soon this router *regenerates its own LSP* after a local change (same initial/secondary/max backoff logic). Fast initial = quick advertisement; backoff = flood damping.
- **`lsp-refresh-interval` / `max-lsp-lifetime`** — periodic LSP refresh (65000 s) well under lifetime (65535 s) so LSPs never age out.
- **`hello-interval` / `hello-multiplier`** — IIH cadence and hold time. With BFD (Task 2.5) doing sub-second failure detection, keep hellos modest to avoid CPU churn.

> **Detection vs. computation:** sub-second convergence = fast **detection** (BFD ~900 ms, Task 2.5) + fast **reaction** (spf/lsp-gen initial-wait 50 ms) + optional **pre-computed backup** (TI-LFA, SR workbook). All three together get you the sub-second number.

### Verification
```
show isis protocol | i "SPF|LSP|interval"   ! configured backoff values echoed
show isis spf-log                            ! SPF run timestamps + trigger + duration; runs fire ~50ms after a change
show isis lsp-log                            ! local LSP regen events
show isis database <sys-id> | i "Lifetime|Seq"   ! lifetime near 65535, refreshing before expiry
```
Timed break-test:
```
! shut a core link, watch spf-log + route table
show isis spf-log            ! new SPF within tens of ms of the BFD-driven adjacency-down
show route isis              ! path reconverged; measure with timestamps
```
- **Look for:** SPF fires within ~50 ms of a change (initial-wait), backs off toward 5 s only under repeated churn; combined with BFD the end-to-end reconvergence is sub-second. LSPs refresh at 65000 s and never expire.

---

## Master Verification Checklist
```
Section 1 — Basic
[ ] 1.1 Emerald: 6 routers L2-only, NET-IDs from loopback, wide metrics, core+Loopback0 only
[ ] 1.2 Garnet: 7 routers L2-only (area 49.0002); Gar-R3 is 4-adjacency hub
[ ] 1.3 Gold: 5 routers L2-only (area 49.0003); G-R3 is hub; no adj on inter-AS links
[ ] 1.4 CCIE-ISIS group: regex GigabitEthernet.* -> metric 200/400, hello-padding disable, p2p; Loopback.* -> passive (inheritance verified)
[ ] 1.5 IPv6 AF single-topology on all 3 instances; one topology in show isis topology
[ ] 1.6 set-overload-bit on-startup 180 level 2 on all routers; OL clears after 180s
[ ] 1.7 Emerald LDP auto-config: LDP sessions == IS-IS adjacencies; NO LDP on Gold/Garnet
[ ] 1.8 Emerald mpls ldp sync level 2: unsynced link gets max metric, IS-IS steers around
Section 2 — Advanced
[ ] 2.1 HMAC-MD5 key-chain; lsp-password (per-level) + hello-password (per-interface); adj survives
[ ] 2.2 advertise passive-only: core /24s suppressed, loopbacks retained, reachability intact
[ ] 2.3 single-topology confirmed (MT contrast explained)
[ ] 2.4 route leaking concept documented; N/A because L2-only (no L1)
[ ] 2.5 BFD 300ms x3 on all core links; sub-second adjacency-down on failure
[ ] 2.6 mesh-group 10 on fully-meshed E-R1/E-R2/P1; LSDB stays consistent; flooding reduced
Section 3 — Verify/TS
[ ] 3.1 adjacency/database/route/topology checklist — all green
[ ] 3.2 INIT fixed: MTU / area-level / auth causes identified and corrected
[ ] 3.3 missing routes fixed: interface-missing + passive misconfig corrected
[ ] 3.4 spf/lsp-gen initial-wait 50ms, backoff to 5s; refresh 65000s; sub-second reconvergence with BFD
```

---

## Notes / Exam Traps
- **NET-ID:** system-id must be **unique** per router and **identical length** domain-wide; the area (`49.0001/0002/0003`) differs per SP but L2 forms across different areas anyway.
- **`metric-style wide`** is mandatory for TE, SR, and >63 metrics — set it before anything advanced (single-topology IPv6 also requires it).
- **hello-padding disable** speeds adjacency but hides MTU mismatches — leave padding on when you *want* MTU validation (Task 3.2).
- **`advertise passive-only`** (2.2) means a non-passive loopback won't be advertised — a subtle 3.3-style trap.
- **mesh-group** (2.6) is only safe on a true full mesh; never put a router's sole uplink in the group.
- **Overload bit** (1.6): `level 2` in XR (not `level-2`); `wait-for-bgp` is the production-grade variant.
- **LDP-IGP sync** (1.8) is IGP-triggered max-metric, not an LDP feature — the IGP advertises the poison metric until LDP is ready.
