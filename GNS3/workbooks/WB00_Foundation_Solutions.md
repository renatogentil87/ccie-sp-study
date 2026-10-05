# WB00 Foundation: IP Addressing — Solutions

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11 — IOS Classic
**Reference:** `00_topology_reference.md` for IP addressing and wiring.

> Base hygiene applied to **every** router: `hostname`, `no ip domain-lookup`, `line con 0`
> with `logging synchronous` + `exec-timeout 0 0`. `ip routing` is on by default but confirmed.
> Only interfaces that are wired in the GNS3 topology are configured. VLAN-100 L2 links
> (R1 g2/0→R29, R14 f4/0→R28, R16 f3/0→R27) carry **no L3 address** — reserved for VPWS/VPLS.

---

## Base hygiene template (applied to all 31 routers)

```
no ip domain-lookup
ip routing
!
line con 0
 exec-timeout 0 0
 logging synchronous
line vty 0 4
 exec-timeout 0 0
 logging synchronous
 transport input all
!
```

> Below, each router shows its hostname + full interface addressing. The hygiene block
> above is implied on every device and not repeated per router.

---

## AS 65100 (R1–R8)

### R1 (PE) — 150.1.1.1

```
hostname R1
!
interface Loopback0
 ip address 150.1.1.1 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R3 f0/0
 ip address 10.1.3.1 255.255.255.0
 no shutdown
!
interface FastEthernet4/0
 description CORE -> R2 f3/0
 ip address 10.1.2.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description PE-CE -> R9 (eBGP 65910)
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description PE-CE -> R10 (eBGP 65910)
 ip address 192.168.2.1 255.255.255.0
 no shutdown
!
interface FastEthernet4/1
 description PE-CE -> R32 (eBGP 65025 Blue)
 ip address 192.168.3.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description L2 VLAN100 -> R29 (Red VPLS - no L3)
 no ip address
 no shutdown
!
```

### R2 (PE) — 150.1.2.2

```
hostname R2
!
interface Loopback0
 ip address 150.1.2.2 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R4 f0/0
 ip address 10.2.4.2 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R1 f4/0
 ip address 10.1.2.2 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description PE-CE -> R10 (eBGP 65910)
 ip address 192.168.4.2 255.255.255.0
 no shutdown
!
```

### R3 (P) — 150.1.3.3

```
hostname R3
!
interface Loopback0
 ip address 150.1.3.3 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R1 f0/0
 ip address 10.1.3.3 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R4 g1/0
 ip address 10.3.4.3 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R5 g2/0
 ip address 10.3.5.3 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R7 f3/0
 ip address 10.3.7.3 255.255.255.0
 no shutdown
!
```

### R4 (P) — 150.1.4.4

```
hostname R4
!
interface Loopback0
 ip address 150.1.4.4 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R2 f0/0
 ip address 10.2.4.4 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R3 g1/0
 ip address 10.3.4.4 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R6 g2/0
 ip address 10.4.6.4 255.255.255.0
 no shutdown
!
```

### R5 (ASBR) — 150.1.5.5

```
hostname R5
!
interface Loopback0
 ip address 150.1.5.5 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R8 f0/0
 ip address 10.5.8.5 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R3 g2/0
 ip address 10.3.5.5 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R6 f3/0
 ip address 10.5.6.5 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description INTER-AS -> R11 (AS65100<->AS65200)
 ip address 10.5.11.5 255.255.255.0
 no shutdown
!
```

### R6 (ASBR) — 150.1.6.6

```
hostname R6
!
interface Loopback0
 ip address 150.1.6.6 255.255.255.255
!
interface GigabitEthernet2/0
 description CORE -> R4 g2/0
 ip address 10.4.6.6 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R5 f3/0
 ip address 10.5.6.6 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description INTER-AS -> R12 (AS65100<->AS65200)
 ip address 10.6.12.6 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description INTER-AS -> R20 (AS65100<->AS65300)
 ip address 10.6.20.6 255.255.255.0
 no shutdown
!
```

### R7 (RR) — 150.1.7.7

```
hostname R7
!
interface Loopback0
 ip address 150.1.7.7 255.255.255.255
!
interface FastEthernet3/0
 description CORE -> R3 f3/0
 ip address 10.3.7.7 255.255.255.0
 no shutdown
!
```

### R8 (RR) — 150.1.8.8

```
hostname R8
!
interface Loopback0
 ip address 150.1.8.8 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R5 f0/0
 ip address 10.5.8.8 255.255.255.0
 no shutdown
!
```

---

## AS 65200 (R11–R16, R19)

### R11 (P) — 150.2.11.11

```
hostname R11
!
interface Loopback0
 ip address 150.2.11.11 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R12 f0/0
 ip address 20.11.12.11 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R13 g2/0
 ip address 20.11.13.11 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R15 f3/0
 ip address 20.11.15.11 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description INTER-AS -> R5 (AS65200<->AS65100)
 ip address 10.5.11.11 255.255.255.0
 no shutdown
!
```

### R12 (ASBR) — 150.2.12.12

```
hostname R12
!
interface Loopback0
 ip address 150.2.12.12 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R11 f0/0
 ip address 20.11.12.12 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R14 g2/0
 ip address 20.12.14.12 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description INTER-AS -> R6 (AS65200<->AS65100)
 ip address 10.6.12.12 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description INTER-AS -> R21 (AS65200<->AS65300)
 ip address 20.12.21.12 255.255.255.0
 no shutdown
!
```

### R13 (P) — 150.2.13.13

```
hostname R13
!
interface Loopback0
 ip address 150.2.13.13 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R16 f0/0
 ip address 20.13.16.13 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R14 g1/0
 ip address 20.13.14.13 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R11 g2/0
 ip address 20.11.13.13 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R19 f3/0
 ip address 20.13.19.13 255.255.255.0
 no shutdown
!
```

### R14 (PE) — 150.2.14.14

```
hostname R14
!
interface Loopback0
 ip address 150.2.14.14 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R19 f0/0
 ip address 20.14.19.14 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R13 g1/0
 ip address 20.13.14.14 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R12 g2/0
 ip address 20.12.14.14 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description PE-CE -> R25 (eBGP 65025 Blue)
 ip address 192.168.5.14 255.255.255.0
 no shutdown
!
interface FastEthernet4/0
 description L2 VLAN100 -> R28 (Red VPLS - no L3)
 no ip address
 no shutdown
!
```

### R15 (PE) — 150.2.15.15

```
hostname R15
!
interface Loopback0
 ip address 150.2.15.15 255.255.255.255
!
interface FastEthernet3/0
 description CORE -> R11 f3/0
 ip address 20.11.15.15 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description PE-CE -> R17 (OSPF Area 0 Green)
 ip address 192.168.6.15 255.255.255.0
 no shutdown
!
```

### R16 (PE) — 150.2.16.16

```
hostname R16
!
interface Loopback0
 ip address 150.2.16.16 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R13 f0/0
 ip address 20.13.16.16 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description PE-CE -> R18 (eBGP 65910 Green)
 ip address 192.168.7.16 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description PE-CE -> R26 (eBGP 65024 Yellow)
 ip address 192.168.8.16 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description L2 VLAN100 -> R27 (Red VPLS - no L3)
 no ip address
 no shutdown
!
```

### R19 (RR) — 150.2.19.19

```
hostname R19
!
interface Loopback0
 ip address 150.2.19.19 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R14 f0/0
 ip address 20.14.19.19 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CORE -> R13 f3/0
 ip address 20.13.19.19 255.255.255.0
 no shutdown
!
```

---

## AS 65300 (R20–R23, R30)

### R20 (ASBR/PE) — 150.3.20.20

```
hostname R20
!
interface Loopback0
 ip address 150.3.20.20 255.255.255.255
!
interface GigabitEthernet1/0
 description CORE -> R21 g1/0
 ip address 30.20.21.20 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R22 g2/0
 ip address 30.20.22.20 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description INTER-AS -> R6 (AS65300<->AS65100)
 ip address 10.6.20.20 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description PE-CE -> R24 (eBGP 65024 Yellow)
 ip address 192.168.9.20 255.255.255.0
 no shutdown
!
```

### R21 (ASBR/PE) — 150.3.21.21

```
hostname R21
!
interface Loopback0
 ip address 150.3.21.21 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R23 f0/0
 ip address 30.21.23.21 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R20 g1/0
 ip address 30.20.21.21 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description INTER-AS -> R12 (AS65300<->AS65200)
 ip address 20.12.21.21 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description PE-CE -> R25 (eBGP 65025 Blue)
 ip address 192.168.10.21 255.255.255.0
 no shutdown
!
```

### R22 (PE) — 150.3.22.22

```
hostname R22
!
interface Loopback0
 ip address 150.3.22.22 255.255.255.255
!
interface GigabitEthernet1/0
 description CORE -> R23 g1/0
 ip address 30.22.23.22 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R20 g2/0
 ip address 30.20.22.22 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description PE-CE -> R24 (eBGP 65024 Yellow)
 ip address 192.168.11.22 255.255.255.0
 no shutdown
!
```

### R23 (PE) — 150.3.23.23

```
hostname R23
!
interface Loopback0
 ip address 150.3.23.23 255.255.255.255
!
interface FastEthernet0/0
 description CORE -> R21 f0/0
 ip address 30.21.23.23 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CORE -> R22 g1/0
 ip address 30.22.23.23 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CORE -> R30 g2/0
 ip address 30.23.30.23 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description PE-CE -> R25 (eBGP 65025 Blue)
 ip address 192.168.12.23 255.255.255.0
 no shutdown
!
```

### R30 (RR) — 150.3.30.30

```
hostname R30
!
interface Loopback0
 ip address 150.3.30.30 255.255.255.255
!
interface GigabitEthernet2/0
 description CORE -> R23 g2/0
 ip address 30.23.30.30 255.255.255.0
 no shutdown
!
```

---

## Customer Edge Routers (CEs)

### R9 (CE, Green AS65910) — 150.9.9.9

```
hostname R9
!
interface Loopback0
 ip address 150.9.9.9 255.255.255.255
!
interface GigabitEthernet1/0
 description CE-PE -> R1 g1/0
 ip address 192.168.1.9 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description CE-CE -> R10 f0/0
 ip address 192.168.100.9 255.255.255.0
 no shutdown
!
```

### R10 (CE, Green AS65910) — 150.10.10.10

```
hostname R10
!
interface Loopback0
 ip address 150.10.10.10 255.255.255.255
!
interface FastEthernet3/0
 description CE-PE -> R1 f3/0
 ip address 192.168.2.10 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0
 description CE-PE -> R2 g1/0
 ip address 192.168.4.10 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description CE-CE -> R9 f0/0
 ip address 192.168.100.10 255.255.255.0
 no shutdown
!
```

### R17 (CE, Green OSPF) — 150.17.17.17

```
hostname R17
!
interface Loopback0
 ip address 150.17.17.17 255.255.255.255
!
interface FastEthernet0/0
 description CE-PE -> R15 f0/0 (OSPF Area 0)
 ip address 192.168.6.17 255.255.255.0
 no shutdown
!
```

### R18 (CE, Green AS65910) — 150.18.18.18

```
hostname R18
!
interface Loopback0
 ip address 150.18.18.18 255.255.255.255
!
interface GigabitEthernet1/0
 description CE-PE -> R16 g1/0
 ip address 192.168.7.18 255.255.255.0
 no shutdown
!
```

### R24 (CE, Yellow AS65024) — 150.24.24.24

```
hostname R24
!
interface Loopback0
 ip address 150.24.24.24 255.255.255.255
!
interface FastEthernet3/0
 description CE-PE -> R20 f3/0
 ip address 192.168.9.24 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 description CE-PE -> R22 f0/0
 ip address 192.168.11.24 255.255.255.0
 no shutdown
!
```

### R25 (CE, Blue AS65025) — 150.25.25.25

```
hostname R25
!
interface Loopback0
 ip address 150.25.25.25 255.255.255.255
!
interface FastEthernet0/0
 description CE-PE -> R14 f3/0
 ip address 192.168.5.25 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 description CE-PE -> R21 g2/0
 ip address 192.168.10.25 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 description CE-PE -> R23 f3/0
 ip address 192.168.12.25 255.255.255.0
 no shutdown
!
```

### R26 (CE, Yellow AS65024) — 150.26.26.26

```
hostname R26
!
interface Loopback0
 ip address 150.26.26.26 255.255.255.255
!
interface GigabitEthernet2/0
 description CE-PE -> R16 g2/0
 ip address 192.168.8.26 255.255.255.0
 no shutdown
!
```

### R27 (CE, Red VPLS) — L2 only

```
hostname R27
!
interface Loopback0
 ip address 150.27.27.27 255.255.255.255
!
interface FastEthernet3/0
 description L2 VLAN100 -> R16 f3/0 (Red VPLS)
 no ip address
 no shutdown
!
```

### R28 (CE, Red VPLS) — L2 only

```
hostname R28
!
interface Loopback0
 ip address 150.28.28.28 255.255.255.255
!
interface FastEthernet0/0
 description L2 VLAN100 -> R14 f4/0 (Red VPLS)
 no ip address
 no shutdown
!
```

### R29 (CE, Red VPLS) — L2 only

```
hostname R29
!
interface Loopback0
 ip address 150.29.29.29 255.255.255.255
!
interface GigabitEthernet2/0
 description L2 VLAN100 -> R1 g2/0 (Red VPLS)
 no ip address
 no shutdown
!
```

### R32 (CE, Blue AS65025) — 150.32.32.32

```
hostname R32
!
interface Loopback0
 ip address 150.32.32.32 255.255.255.255
!
interface FastEthernet0/0
 description CE-PE -> R1 f4/1
 ip address 192.168.3.32 255.255.255.0
 no shutdown
!
```

> R27/R28/R29 loopbacks (150.27/28/29) are assigned for device identity/management even
> though their customer-facing ports are L2-only. Their /32s are not in the reference's CE
> loopback table but follow the `150.<r>.<r>.<r>` convention; adjust/remove if your lab omits them.

---

## Verification

```
show ip interface brief | exclude unassigned     ! every wired port up/up with correct IP
show ip route connected                          ! all directly-connected subnets present
!
! Directly-connected ping sweep examples:
ping 10.1.3.3 source 10.1.3.1                     ! R1 -> R3 core
ping 10.5.11.11 source 10.5.11.5                  ! R5 -> R11 inter-AS
ping 192.168.1.9 source 192.168.1.1               ! R1 -> R9 PE-CE
ping 192.168.100.10 source 192.168.100.9          ! R9 -> R10 CE-CE
```

> **Expected at this stage:** 100% of *directly connected* neighbors ping. Remote loopbacks are
> NOT reachable yet (no IGP) — that is correct until WB01/WB02.
