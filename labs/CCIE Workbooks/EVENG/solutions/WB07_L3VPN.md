# CCIE SP Workbook 07 — L3VPN

**Domain:** 2 — Architectures & Services (25%)
**Platform:** EVE-NG — IOS-XRv 9000 7.11.1 (all PE/P/RR/ASBR nodes)
🔴 **Topology:** Emerald + Gold + Garnet — see `../configuration/EVENG/00_topology_reference.md`
**Format:** Question → Solution → Verification. All configs in IOS-XR syntax.

---

## Topology Recap (nodes used in this workbook)

| SP | AS | IGP / Transport | PEs | RR | ASBR |
|----|----|-----------------|-----|----|------|
| **Emerald** | 65100 | IS-IS L2 + LDP | E-R1 (1.1.1.1), E-R2 (2.2.2.2) | **E-R5 (6.6.6.6)** | E-R6 (5.5.5.5) |
| **Gold** (transit) | 65300 | IS-IS L2 + SRv6 | G-R1 (21.21.21.21), G-R2 (22.22.22.22) | **G-R4 (24.24.24.24)** | G-R4 / G-R5 (25.25.25.25) |
| **Garnet** | 65200 | IS-IS L2 + SR-MPLS | Gar-R1 (11.11.11.11), Gar-R2 (12.12.12.12) | **Gar-R6 (16.16.16.16)** | Gar-R7 (17.17.17.17) |

**Customers**

| Customer | ASN | CEs | SPs spanned | PE-CE protocol |
|----------|-----|-----|-------------|----------------|
| **A** | 65012 | CE1 (E-R1), CE2 (E-R1+E-R2 dual-homed) — Emerald; CE8 (G-R1+G-R2 dual-homed) — Gold | Emerald + Gold | eBGP (as-override + SoO) |
| **B** | 65013 | CE9 (G-R1) — Gold; CE4 (Gar-R1) — Garnet | Gold + Garnet | eBGP |
| **C** | — (EVPN) | CE5 (Gar-R1+Gar-R2) — Garnet; CE7 (G-R2) — Gold | Garnet + Gold | EVPN (WB10, not here) |
| — | — | CE3 (E-R2) — Emerald; CE6 (Gar-R2) — Garnet | single SP | OSPF area 0 |

**PE-CE links (relevant)**

| Link | Subnet | PE addr | CE addr |
|------|--------|---------|---------|
| E-R1–CE1 | 192.168.1.0/24 | .1 | .2 |
| E-R1–CE2 | 192.168.2.0/24 | .1 | .2 |
| E-R2–CE2 | 192.168.3.0/24 | .1 | .2 |
| E-R2–CE3 | 192.168.4.0/24 | .1 | .2 |
| G-R1–CE9 | 192.168.11.0/24 | .1 | .2 |
| G-R1–CE8 | 192.168.12.0/24 | .1 | .2 |
| G-R2–CE8 | 192.168.13.0/24 | .1 | .2 |
| Gar-R2–CE6 | 172.16.4.0/24 | .1 | .2 |
| Gar-R1–CE4 | 172.16.1.0/24 | .1 | .2 |

**RD / RT design decision (used throughout):**
- **RD is per-SP-per-VRF** (unique so overlapping customer prefixes stay unique in VPNv4): `<SP-ASN>:<vrf-id>`.
- **RT is per-customer, common across all SPs** (so the same customer's sites import each other regardless of which SP they attach to): `<customer-ASN>:<vrf-id>`.

| VRF | On | RD | RT (import+export) |
|-----|----|----|--------------------|
| CUST_A | E-R1, E-R2 (Emerald) | 65100:100 (E-R1), 65100:100 (E-R2) | 65012:100 |
| CUST_A | G-R1, G-R2 (Gold) | 65300:100 | 65012:100 |
| CUST_B | G-R1 (Gold) | 65300:200 | 65013:200 |
| CUST_B | Gar-R1 (Garnet) | 65200:200 | 65013:200 |

> RD differs per SP; RT is identical (`65012:100` for Customer A everywhere). That is what makes inter-AS VPN "just work" once VPNv4 is exchanged across the ASBRs.

---

# Section 1 — VRF + RD + RT Basics (5 tasks)

## Task 1.1 — Create VRF CUST_A on E-R1/E-R2 (Emerald)

**Question:** Create VRF `CUST_A` for Customer A on both Emerald PEs. Use a per-SP RD and the common Customer-A RT `65012:100`.

**Solution (E-R1):**
```
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   65012:100
  !
  export route-target
   65012:100
  !
 !
!
router bgp 65100
 vrf CUST_A
  rd 65100:100
  address-family ipv4 unicast
  !
 !
!
```

**Solution (E-R2)** — identical, RD stays per-SP (`65100:100`) but keeps a unique RD-per-PE convention is optional; here Emerald shares `65100:100` because the SP-ASN scopes it. If you prefer per-PE uniqueness use `65100:100` on E-R1 and `65100:101` on E-R2 — both work because RT drives import, RD only guarantees VPNv4 uniqueness.
```
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   65012:100
  export route-target
   65012:100
 !
!
router bgp 65100
 vrf CUST_A
  rd 65100:100
  address-family ipv4 unicast
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show vrf CUST_A
VRF                              RD              AFI SAFI
CUST_A                           65100:100       IPV4 Unicast

RP/0/RP0/CPU0:E-R1# show run vrf CUST_A
RP/0/RP0/CPU0:E-R1# show bgp vrf CUST_A         ! empty until interface + PE-CE up
```

---

## Task 1.2 — Create VRF CUST_A on G-R1/G-R2 (Gold)

**Question:** Extend Customer A into Gold on G-R1 and G-R2. Same RT (`65012:100`), Gold-scoped RD.

**Solution (G-R1 and G-R2 — identical):**
```
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   65012:100
  export route-target
   65012:100
 !
!
router bgp 65300
 vrf CUST_A
  rd 65300:100
  address-family ipv4 unicast
```

**Verification:**
```
RP/0/RP0/CPU0:G-R1# show vrf CUST_A detail | i "Import\|Export\|RD"
VRF CUST_A; RD 65300:100
  Import VPN route-target communities: 65012:100
  Export VPN route-target communities: 65012:100
```

> Note the **RD differs** (65300:100 vs Emerald's 65100:100) but the **RT matches** (65012:100). Customer A is now consistent across two SPs.

---

## Task 1.3 — Create VRF CUST_B on G-R1 (Gold) + Gar-R1 (Garnet)

**Question:** Customer B lives in Gold (CE9→G-R1) and Garnet (CE4→Gar-R1). Create VRF `CUST_B` with common RT `65013:200`.

**Solution (G-R1 — Gold):**
```
vrf CUST_B
 address-family ipv4 unicast
  import route-target
   65013:200
  export route-target
   65013:200
 !
!
router bgp 65300
 vrf CUST_B
  rd 65300:200
  address-family ipv4 unicast
```

**Solution (Gar-R1 — Garnet):**
```
vrf CUST_B
 address-family ipv4 unicast
  import route-target
   65013:200
  export route-target
   65013:200
 !
!
router bgp 65200
 vrf CUST_B
  rd 65200:200
  address-family ipv4 unicast
```

**Verification:**
```
RP/0/RP0/CPU0:G-R1# show vrf CUST_B
CUST_B                           65300:200       IPV4 Unicast
RP/0/RP0/CPU0:Gar-R1# show vrf CUST_B
CUST_B                           65200:200       IPV4 Unicast
```

---

## Task 1.4 — Assign PE-CE interfaces to VRFs

**Question:** Bind each customer-facing interface into the correct VRF and address it. Remember: on IOS-XR, `vrf X` under the interface **clears the IP**, so re-apply the address after.

**Solution (Emerald — Customer A):**
```
! E-R1 → CE1
interface GigabitEthernet0/0/0/0
 vrf CUST_A
 ipv4 address 192.168.1.1 255.255.255.0
 no shutdown
!
! E-R1 → CE2 (dual-homed leg 1)
interface GigabitEthernet0/0/0/1
 vrf CUST_A
 ipv4 address 192.168.2.1 255.255.255.0
 no shutdown
!
! E-R2 → CE2 (dual-homed leg 2)
interface GigabitEthernet0/0/0/1
 vrf CUST_A
 ipv4 address 192.168.3.1 255.255.255.0
 no shutdown
```

**Solution (Gold — Customer A + B):**
```
! G-R1 → CE8 (Customer A)
interface GigabitEthernet0/0/0/1
 vrf CUST_A
 ipv4 address 192.168.12.1 255.255.255.0
 no shutdown
!
! G-R2 → CE8 (Customer A, dual-homed leg 2)
interface GigabitEthernet0/0/0/1
 vrf CUST_A
 ipv4 address 192.168.13.1 255.255.255.0
 no shutdown
!
! G-R1 → CE9 (Customer B)
interface GigabitEthernet0/0/0/0
 vrf CUST_B
 ipv4 address 192.168.11.1 255.255.255.0
 no shutdown
```

**Solution (Garnet — Customer B):**
```
! Gar-R1 → CE4
interface GigabitEthernet0/0/0/2
 vrf CUST_B
 ipv4 address 172.16.1.1 255.255.255.0
 no shutdown
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show ipv4 vrf CUST_A interface brief
Interface                IP-Address      Status       VRF
GigabitEthernet0/0/0/0   192.168.1.1     Up           CUST_A
GigabitEthernet0/0/0/1   192.168.2.1     Up           CUST_A

RP/0/RP0/CPU0:E-R1# show route vrf CUST_A       ! connected routes now present
C    192.168.1.0/24 is directly connected, GigabitEthernet0/0/0/0
C    192.168.2.0/24 is directly connected, GigabitEthernet0/0/0/1
```

---

## Task 1.5 — RD/RT design summary and validation

**Question:** Prove the design rule "per-SP RD, common RT for the same customer" holds across the fabric, and explain why RD uniqueness matters.

**Solution — the design:**
```
Customer A (AS 65012), RT 65012:100 everywhere:
  E-R1 (Emerald)  RD 65100:100
  E-R2 (Emerald)  RD 65100:100
  G-R1 (Gold)     RD 65300:100
  G-R2 (Gold)     RD 65300:100

Customer B (AS 65013), RT 65013:200 everywhere:
  G-R1 (Gold)     RD 65300:200
  Gar-R1 (Garnet)   RD 65200:200
```

Why RD must be unique per SP: two sites could advertise the same prefix (e.g. `10.0.0.0/24`). VPNv4 NLRI = `RD:prefix`. If both SPs used the same RD, the RR would treat them as the same VPNv4 route and best-path would hide one. Unique RD keeps them distinct so the receiving PE sees **both** and imports by RT.

**Verification (after VPNv4 is up — forward reference to Section 3):**
```
RP/0/RP0/CPU0:E-R5# show bgp vpnv4 unicast rd 65100:100
RP/0/RP0/CPU0:E-R5# show bgp vpnv4 unicast rd 65300:100      ! same customer, distinct RD
! Same RT (65012:100) on both → both import into CUST_A on any PE.
RP/0/RP0/CPU0:G-R1# show bgp vpnv4 unicast rt 65012:100
```

---

# Section 2 — PE-CE Routing (5 tasks)

## Task 2.1 — eBGP PE-CE with CE1/CE2/CE8 (AS 65012, as-override)

**Question:** Customer A CEs all use AS 65012. Because multiple sites share the same AS, the far CE will reject routes that carry `65012` in the AS_PATH. Configure eBGP PE-CE and fix the loop-prevention rejection with **as-override**.

**Solution (E-R1 — faces CE1 and CE2):**
```
router bgp 65100
 vrf CUST_A
  rd 65100:100
  address-family ipv4 unicast
  !
  neighbor 192.168.1.2
   remote-as 65012
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
    soo 65100:1               ! see Task 2.5
  !
  neighbor 192.168.2.2
   remote-as 65012
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
    soo 65100:2
  !
 !
!
route-policy PASS
  pass
end-policy
```

**Solution (G-R1 — faces CE8, Gold):**
```
router bgp 65300
 vrf CUST_A
  rd 65300:100
  address-family ipv4 unicast
  neighbor 192.168.12.2
   remote-as 65012
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
    soo 65300:8
```

**Why as-override:** When G-R1 advertises CE1's prefix to CE8, the AS_PATH is `65100 65012`. CE8 (AS 65012) sees its own AS and drops it. `as-override` makes the PE **replace every occurrence of the customer AS (65012) in the AS_PATH with its own AS (65100/65300)** before sending to the CE, so CE8 accepts it.

**Verification:**
```
RP/0/RP0/CPU0:G-R1# show bgp vrf CUST_A neighbors 192.168.12.2 | i override
  My AS number is 65300 ... AS override is enabled

! On CE8 the path shows the PE AS instead of 65012:
CE8# show ip bgp 192.168.1.0
  Refresh Epoch 1
  65300 65300 65300        ! 65012 was overridden to 65300
    192.168.12.1 from 192.168.12.1
```

---

## Task 2.2 — eBGP PE-CE with CE9/CE4 (AS 65013)

**Question:** Customer B uses AS 65013: CE9 in Gold (G-R1) and CE4 in Garnet (Gar-R1). Configure eBGP PE-CE. as-override is still needed because both sites share AS 65013.

**Solution (G-R1 — CE9, Gold):**
```
router bgp 65300
 vrf CUST_B
  rd 65300:200
  address-family ipv4 unicast
  neighbor 192.168.11.2
   remote-as 65013
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
```

**Solution (Gar-R1 — CE4, Garnet):**
```
router bgp 65200
 vrf CUST_B
  rd 65200:200
  address-family ipv4 unicast
  neighbor 172.16.1.2
   remote-as 65013
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
```

**Verification:**
```
RP/0/RP0/CPU0:Gar-R1# show bgp vrf CUST_B summary
Neighbor        Spk AS   MsgRcvd MsgSent  TblVer  InQ OutQ Up/Down  St/PfxRcd
172.16.1.2      0  65013      12      14      45    0    0 00:05:11         3

RP/0/RP0/CPU0:Gar-R1# show bgp vrf CUST_B neighbors 172.16.1.2 routes
```

---

## Task 2.3 — OSPF PE-CE with CE3/CE6 (area 0, DN-bit, domain-id)

**Question:** CE3 (E-R2, Emerald) and CE6 (Gar-R2, Garnet) run OSPF area 0 to the PE. Configure VRF-aware OSPF, redistribute both ways with BGP, and understand the DN-bit and domain-id.

**Solution (E-R2 — CE3, Emerald):**
```
router ospf CUST_A_PECE
 vrf CUST_A
  domain-id type 0005 value 000000010000        ! domain-id 1
  router-id 2.2.2.2
  redistribute bgp 65100                          ! VPN → OSPF (Type 3/5 with DN-bit set by PE)
  area 0
   interface GigabitEthernet0/0/0/2               ! toward CE3 192.168.4.0/24
   !
  !
 !
!
router bgp 65100
 vrf CUST_A
  address-family ipv4 unicast
   redistribute ospf CUST_A_PECE                  ! OSPF → VPN
```

**Solution (Gar-R2 — CE6, Garnet):**
```
router ospf CUST_CE6
 vrf CUST_C6
  domain-id type 0005 value 000000010000
  router-id 12.12.12.12
  redistribute bgp 65200
  area 0
   interface GigabitEthernet0/0/0/2               ! toward CE6 172.16.4.0/24
!
router bgp 65200
 vrf CUST_C6
  address-family ipv4 unicast
   redistribute ospf CUST_CE6
```

**Key concepts:**
- **DN-bit (down bit):** When a PE redistributes a BGP VPN route into OSPF toward the CE, it sets the DN-bit in the LSA. Any PE that receives an OSPF LSA with the DN-bit set will **not** redistribute it back into BGP — this breaks the PE↔PE↔CE loop. (Task 6.4 shows what happens if it's missing.)
- **domain-id:** Identifies the OSPF domain. If two PEs share a domain-id, inter-area/external routes are carried as **inter-area (Type 3)**; if they differ, routes cross as **external Type 5** with metric-type 2. Matching domain-id makes the MPLS core transparent to OSPF (looks like a super-backbone / area 0).

**Verification:**
```
RP/0/RP0/CPU0:E-R2# show ospf CUST_A_PECE vrf CUST_A neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
31.31.31.31       1   FULL/DR         00:00:38     192.168.4.2     Gi0/0/0/2

RP/0/RP0/CPU0:E-R2# show route vrf CUST_A ospf
O    192.168.40.0/24 [110/2] via 192.168.4.2

! Confirm DN-bit set on the LSA the PE injects toward the CE:
RP/0/RP0/CPU0:E-R2# show ospf CUST_A_PECE vrf CUST_A database summary detail | i "Options|DN"
```

---

## Task 2.4 — VRF OSPF sham-link (CE3 backdoor scenario)

**Question:** Customer has a **backdoor link** between two OSPF sites (say CE3's site also has a low-speed direct link to another Customer-A OSPF site). OSPF intra-area routes over the backdoor are always preferred over the MPLS-learned inter-area routes, black-holing the high-speed core. Configure a **sham-link** so the MPLS path competes as an intra-area link.

**Solution:**
1. Create VRF loopbacks on each PE, advertised via **BGP** (not OSPF), to be the sham-link endpoints:
```
! E-R2 (Emerald)
interface Loopback100
 vrf CUST_A
 ipv4 address 10.100.2.2 255.255.255.255
!
router bgp 65100
 vrf CUST_A
  address-family ipv4 unicast
   network 10.100.2.2/32          ! endpoint reachable via VPNv4, NOT OSPF
!
! PE-far (the other Customer-A OSPF PE), endpoint 10.100.9.9/32 advertised the same way
```
2. Build the sham-link between the two VRF loopbacks:
```
router ospf CUST_A_PECE
 vrf CUST_A
  area 0
   sham-link 10.100.2.2 10.100.9.9
    cost 1                          ! lower than the backdoor's cost so MPLS wins
   !
```

**Why loopbacks via BGP:** If the sham-link endpoints were reachable via OSPF, the sham-link would loop through itself. They must be VPNv4-learned so the sham-link rides the MPLS core.

**Verification:**
```
RP/0/RP0/CPU0:E-R2# show ospf CUST_A_PECE vrf CUST_A sham-links
Sham Link OSPF_SL0 to address 10.100.9.9 is up
  Area 0, source address 10.100.2.2
  Cost: 1  State: POINT_TO_POINT

! Remote site prefixes now show as intra-area (O) over the sham-link, beating the backdoor:
RP/0/RP0/CPU0:E-R2# show route vrf CUST_A 192.168.90.0
Routing entry for 192.168.90.0/24
  Known via "ospf", type intra area
  Routing Descriptor Blocks
    10.100.9.9, from ... via sham-link
```

---

## Task 2.5 — SoO on dual-homed CE2 and CE8

**Question:** CE2 (dual-homed to E-R1+E-R2) and CE8 (dual-homed to G-R1+G-R2) risk a routing loop: a route from CE2 enters E-R1 → VPNv4 → E-R2 → back to CE2. `as-override` removes the AS-PATH loop protection, so we need **Site-of-Origin (SoO)** to stop a route from being re-advertised back to the site it came from.

**Solution — tag each interface of the same site with the same SoO:**
```
! CE2 is one site reached via E-R1 (192.168.2.x) and E-R2 (192.168.3.x) → SAME SoO
! E-R1 → CE2
router bgp 65100
 vrf CUST_A
  neighbor 192.168.2.2
   address-family ipv4 unicast
    as-override
    soo 65012:2
!
! E-R2 → CE2  (same SoO value = same site)
router bgp 65100
 vrf CUST_A
  neighbor 192.168.3.2
   address-family ipv4 unicast
    as-override
    soo 65012:2
```
```
! CE8 dual-homed to G-R1+G-R2 (Gold) → same SoO 65012:8 on both legs
! G-R1 → CE8
router bgp 65300
 vrf CUST_A
  neighbor 192.168.12.2
   address-family ipv4 unicast
    as-override
    soo 65012:8
! G-R2 → CE8
router bgp 65300
 vrf CUST_A
  neighbor 192.168.13.2
   address-family ipv4 unicast
    as-override
    soo 65012:8
```

**How SoO stops the loop:** E-R1 tags CE2's routes with SoO `65012:2`. That community rides in VPNv4 to E-R2. Before E-R2 advertises a route to its CE2 neighbor (also SoO `65012:2`), it checks: the route already carries this SoO → **do not send back to the originating site**. Loop prevented even though as-override stripped the AS-PATH defense.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show bgp vrf CUST_A 192.168.20.0/24 detail | i "SoO|Origin"
    Origin-AS validity: not-found
    Site of Origin: 65012:2

RP/0/RP0/CPU0:E-R2# show bgp vrf CUST_A neighbors 192.168.3.2 | i "Site-of-Origin"
  Site-of-Origin (SoO): 65012:2

! Prove the loop is gone: a route sourced at CE2 via E-R1 is NOT re-advertised out E-R2 to CE2.
RP/0/RP0/CPU0:E-R2# show bgp vrf CUST_A neighbors 192.168.3.2 advertised-routes | i 192.168.2
! (no output — SoO filtered it)
```

---

# Section 3 — MP-BGP VPNv4 (4 tasks)

## Task 3.1 — Enable VPNv4 on all RRs and PE↔RR sessions

**Question:** Configure MP-BGP VPNv4 unicast. RRs: **E-R5** (Emerald, 6.6.6.6), **G-R4** (Gold, 24.24.24.24), **Gar-R6** (Garnet, 16.16.16.16). PEs peer to their SP's RR only.

**Solution (Emerald RR = E-R5):**
```
router bgp 65100
 address-family vpnv4 unicast
 !
 neighbor-group RR-CLIENTS
  remote-as 65100
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
 neighbor 1.1.1.1
  use neighbor-group RR-CLIENTS      ! E-R1
 !
 neighbor 2.2.2.2
  use neighbor-group RR-CLIENTS      ! E-R2
 !
 neighbor 5.5.5.5
  use neighbor-group RR-CLIENTS      ! E-R6 (for inter-AS, Section 4)
!
```

**Solution (Emerald PE = E-R1, mirror on E-R2):**
```
router bgp 65100
 address-family vpnv4 unicast
 !
 neighbor 6.6.6.6
  remote-as 65100
  update-source Loopback0
  address-family vpnv4 unicast
```

**Solution (Gold RR = G-R4):**
```
router bgp 65300
 address-family vpnv4 unicast
 neighbor 21.21.21.21              ! G-R1
  remote-as 65300
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
 !
 neighbor 22.22.22.22              ! G-R2
  remote-as 65300
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
```

**Solution (Garnet RR = Gar-R6):**
```
router bgp 65200
 address-family vpnv4 unicast
 neighbor 11.11.11.11              ! Gar-R1
  remote-as 65200
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
 !
 neighbor 12.12.12.12              ! Gar-R2
  remote-as 65200
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
```

**Verification:**
```
RP/0/RP0/CPU0:E-R5# show bgp vpnv4 unicast summary
Neighbor        Spk AS   ... Up/Down  St/PfxRcd
1.1.1.1         0  65100     00:12:03         6
2.2.2.2         0  65100     00:12:01         4
5.5.5.5         0  65100     00:11:44         0
```

---

## Task 3.2 — Verify VPN route propagation within each SP

**Question:** Confirm that Customer A prefixes learned at E-R1 reach E-R2 (Emerald), and Customer A/B prefixes propagate within Gold and Garnet.

**Solution:** No new config — this validates 3.1 + Section 1/2. Trace one prefix end to end.

**Verification (Emerald, CE1's prefix 192.168.10.0/24):**
```
! E-R1 originates into VPNv4:
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24
  ... Local label: 24012
  Paths: (1 available, best)
    Local
      192.168.1.2 from 192.168.1.2 ... best

! RR reflects it:
RP/0/RP0/CPU0:E-R5# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24
    Received from E-R1 (1.1.1.1) ... reflected

! E-R2 imports into VRF via RT 65012:100:
RP/0/RP0/CPU0:E-R2# show route vrf CUST_A 192.168.10.0/24
  B    192.168.10.0/24 [200/0] via 1.1.1.1 (nexthop in vrf default), label ...
```

**Verification (label stack / forwarding):**
```
RP/0/RP0/CPU0:E-R2# show cef vrf CUST_A 192.168.10.0/24
  via 1.1.1.1, ... labels imposed {LDP-label VPN-label}
```

---

## Task 3.3 — RT import/export behavior (extranet demo)

**Question:** Demonstrate RT control by leaking one Customer-B prefix into Customer A (a simple hub/extranet). Show that RT — not RD — drives import.

**Solution (on a PE that holds both VRFs, e.g. G-R1 which has CUST_A and CUST_B):**
```
! Make CUST_A additionally import Customer B's RT (one-way extranet):
router bgp 65300
 vrf CUST_A
  address-family ipv4 unicast
!
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   65012:100
   65013:200        ! now also pulls Customer B routes
```

**Verification:**
```
RP/0/RP0/CPU0:G-R1# show bgp vrf CUST_A | i 192.168.11    ! CE9 (Cust B) prefix now in CUST_A
*>i192.168.110.0/24  ...
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.110.0/24
  B    192.168.110.0/24 [200/0] via ...

! Prove RD is irrelevant to import: the imported route has RD 65300:200 but landed in CUST_A (RD 65300:100).
```
> Roll back afterward (remove the extra import) to keep customers isolated for later tasks.

---

## Task 3.4 — Next-hop handling: next-hop-self on RR vs next-hop-unchanged

**Question:** Explain and configure next-hop behavior. Within an SP the RR must not change the next-hop (so PEs forward to the originating PE). For inter-AS Option B/C the ASBR/RR must alter next-hop. Show both.

**Solution — intra-SP RR keeps next-hop = originating PE (default; do NOT set next-hop-self on the RR for its clients):**
```
router bgp 65100
 neighbor 1.1.1.1
  address-family vpnv4 unicast
   route-reflector-client
   ! (no next-hop-self) → E-R1 stays the next-hop when reflected to E-R2
```

**Solution — inter-AS eBGP VPNv4 (Option B, Section 4) needs next-hop-self on the ASBR toward its own RR, and next-hop-unchanged toward the remote ASBR:**
```
! E-R6 toward remote Gar-R7 (eBGP VPNv4): keep the remote PE next-hop opaque
router bgp 65100
 neighbor 10.0.1.2               ! Gar-R7
  remote-as 65200
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-unchanged            ! preserve originating PE next-hop across the boundary (Option C style)
!
! E-R6 toward its own RR (iBGP): rewrite next-hop to itself so internal PEs resolve it via IGP
router bgp 65100
 neighbor 6.6.6.6
  address-family vpnv4 unicast
   next-hop-self                 ! Option B style: ASBR becomes the next-hop internally
```

**Verification:**
```
! Intra-SP: E-R2 sees E-R1 as next-hop (unchanged by RR)
RP/0/RP0/CPU0:E-R2# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24 | i "next hop"
    Next Hop: 1.1.1.1

! On ASBR after next-hop-self:
RP/0/RP0/CPU0:E-R6# show bgp vpnv4 unicast neighbors 6.6.6.6 advertised-routes detail | i "next hop"
    Next Hop: 5.5.5.5
```

---

# Section 4 — Inter-AS VPN (6 tasks)

> Boundary links: E-R6(Emerald 10.0.1.1) ↔ Gar-R7(Garnet 10.0.1.2); E-R6(10.0.2.1) ↔ G-R4(Gold 10.0.2.2); G-R5(Gold 10.0.3.x) ↔ Gar-R7(Garnet 10.0.3.x).

## Task 4.1 — Option A between Emerald ↔ Garnet (back-to-back VRF on E-R6/Gar-R7)

**Question:** Implement **Inter-AS Option A** (10A / back-to-back VRF): each ASBR treats the other as a CE. One sub-interface per VRF per customer across the boundary, running eBGP in the VRF.

**Solution (E-R6 — Emerald side):**
```
vrf CUST_A
 address-family ipv4 unicast
  import  route-target 65012:100
  export  route-target 65012:100
!
interface GigabitEthernet0/0/0/1.100
 vrf CUST_A
 ipv4 address 10.10.100.1 255.255.255.0
 encapsulation dot1q 100
!
router bgp 65100
 vrf CUST_A
  rd 65100:100
  address-family ipv4 unicast
  neighbor 10.10.100.2           ! Gar-R7 acts like a CE
   remote-as 65200
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
```

**Solution (Gar-R7 — Garnet side, mirror):**
```
vrf CUST_A
 address-family ipv4 unicast
  import  route-target 65012:100
  export  route-target 65012:100
!
interface GigabitEthernet0/0/0/1.100
 vrf CUST_A
 ipv4 address 10.10.100.2 255.255.255.0
 encapsulation dot1q 100
!
router bgp 65200
 vrf CUST_A
  rd 65200:100
  address-family ipv4 unicast
  neighbor 10.10.100.1
   remote-as 65100
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
```

**Traits:** No VPNv4 between ASBRs — plain IPv4 in the VRF. One VRF interface **per customer** (does not scale). Full QoS/filtering visibility at the boundary.

**Verification:**
```
RP/0/RP0/CPU0:E-R6# show bgp vrf CUST_A summary | i 10.10.100.2
10.10.100.2     0  65200 ...          5
RP/0/RP0/CPU0:E-R6# show route vrf CUST_A     ! Garnet-side Cust-A prefixes present
```

---

## Task 4.2 — Option B (VPNv4 on ASBRs, next-hop-self)

**Question:** Implement **Inter-AS Option B** (10B): ASBRs exchange **VPNv4** directly over eBGP, no VRFs on the ASBR. ASBR rewrites next-hop to itself and re-originates labels.

**Solution (E-R6):**
```
router bgp 65100
 address-family vpnv4 unicast
  retain route-target all         ! ASBR keeps all RTs even with no local VRF
 !
 ! eBGP VPNv4 to Gar-R7
 neighbor 10.0.1.2
  remote-as 65200
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
 !
 ! iBGP VPNv4 to Emerald RR, next-hop-self so internal resolves ASBR via IGP
 neighbor 6.6.6.6
  remote-as 65100
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
```

**Solution (Gar-R7 — mirror, AS 65200, RR 16.16.16.16):**
```
router bgp 65200
 address-family vpnv4 unicast
  retain route-target all
 !
 neighbor 10.0.1.1
  remote-as 65100
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
 !
 neighbor 16.16.16.16
  remote-as 65200
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
```

**Traits:** Scales far better than A (one eBGP VPNv4 session carries all customers). ASBR holds VPNv4 in RIB but no VRFs. `retain route-target all` is required (otherwise ASBR drops routes whose RT it can't import). New label allocated by ASBR at the boundary.

**Verification:**
```
RP/0/RP0/CPU0:E-R6# show bgp vpnv4 unicast summary | i 10.0.1.2
10.0.1.2        0  65200 ...

RP/0/RP0/CPU0:E-R6# show bgp vpnv4 unicast rd 65200:100 192.168.120.0/24
  ... received from 10.0.1.2, best
RP/0/RP0/CPU0:E-R1# show route vrf CUST_A 192.168.120.0/24    ! next-hop = E-R6 (5.5.5.5)
```

---

## Task 4.3 — Option C (BGP-LU + multihop VPNv4 between RRs)

**Question:** Implement **Inter-AS Option C** (10C): ASBRs exchange only **BGP labeled-unicast (BGP-LU)** for PE loopbacks; the **RRs** hold a **multihop eBGP VPNv4** session and pass VPNv4 with **next-hop-unchanged**. Data plane is end-to-end LSP; no per-VPN state on ASBRs.

**Solution — ASBRs exchange PE /32 loopbacks via BGP-LU:**
```
! E-R6 (Emerald)
router bgp 65100
 address-family ipv4 unicast
  network 1.1.1.1/32
  network 2.2.2.2/32
  allocate-label all
 !
 neighbor 10.0.1.2               ! Gar-R7
  remote-as 65200
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
 !
 ! Redistribute remote loopbacks into IGP OR keep as BGP-LU + next-hop reachability
 neighbor 6.6.6.6               ! own RR — pass labeled loopbacks internally
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
```

**Solution — multihop eBGP VPNv4 between the two RRs (E-R5 ↔ Gar-R6):**
```
! E-R5 (Emerald RR, 6.6.6.6)
router bgp 65100
 neighbor 16.16.16.16            ! Garnet RR
  remote-as 65200
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-unchanged            ! keep originating PE as next-hop
!
! Gar-R6 (Garnet RR, 16.16.16.16) — mirror
router bgp 65200
 neighbor 6.6.6.6
  remote-as 65100
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-unchanged
```

**Traits:** Most scalable, no VPN state on ASBRs, end-to-end LSP. Requires PE loopback reachability across ASes (BGP-LU) so the remote PE next-hop resolves. `next-hop-unchanged` is mandatory on the RR-to-RR session.

**Verification:**
```
! ASBR exchanges labeled PE loopbacks:
RP/0/RP0/CPU0:E-R6# show bgp ipv4 labeled-unicast 11.11.11.11/32
  ... received-label / local-label present
! RR-to-RR VPNv4 with unchanged next-hop:
RP/0/RP0/CPU0:E-R5# show bgp vpnv4 unicast rd 65200:100 192.168.120.0/24 | i "next hop"
    Next Hop: 11.11.11.11          ! remote PE, not the RR
! End-to-end LSP to remote PE loopback:
RP/0/RP0/CPU0:E-R1# show cef 11.11.11.11/32     ! labeled path across AS boundary
```

---

## Task 4.4 — Customer A across Emerald ↔ Gold (Option C with SRv6 transport on Gold side)

**Question:** Extend Customer A from Emerald (LDP transport) into Gold (SRv6 transport) using Option C. Gold uses SRv6 for the VPN; the E-R6↔G-R4 boundary carries the reachability, and RR-to-RR (E-R5↔G-R4) carries VPNv4 with next-hop-unchanged. Show the transport handoff.

**Solution — boundary E-R6 (Emerald, LDP/MPLS) ↔ G-R4 (Gold, SRv6):**
```
! E-R6 (Emerald) — labeled-unicast for PE loopbacks toward Gold G-R4
router bgp 65100
 neighbor 10.0.2.2               ! G-R4 (Gold)
  remote-as 65300
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
```
```
! Gold VPN over SRv6 — G-R1/G-R2 use an SRv6 locator per VRF (micro-SID / End.DT4)
segment-routing
 srv6
  locators
   locator GOLD
    micro-segment behavior unicast
    prefix fc00:300::/48
!
router bgp 65300
 vrf CUST_A
  rd 65300:100
  address-family ipv4 unicast
   segment-routing srv6
    locator GOLD
    alloc mode per-vrf           ! End.DT4 SID per VRF
   !
   redistribute connected
```

**Solution — RR-to-RR multihop VPNv4 (E-R5 Emerald ↔ G-R4 Gold RR):**
```
! E-R5 (Emerald RR)
router bgp 65100
 neighbor 24.24.24.24            ! G-R4 acts as Gold RR
  remote-as 65300
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-unchanged
!
! G-R4 (Gold RR) — mirror; carries both VPNv4 (from Emerald) and SRv6-VPN (its own PEs)
router bgp 65300
 neighbor 6.6.6.6
  remote-as 65100
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-unchanged
```

**Transport handoff logic:** Emerald encodes the VPN with an **MPLS VPN label** over LDP; Gold encodes it with an **SRv6 End.DT4 SID**. The ASBR/RR boundary translates the service: a prefix from CE1 (Emerald) arrives at G-R1 (Gold) and is programmed into CUST_A with an SRv6 SID for the return path, while the CE8→CE1 direction resolves to an MPLS LSP once it crosses back at the ASBR. Customer A reachability (CE1/CE2 ↔ CE8) is preserved across two different data planes.

**Verification:**
```
! Gold PE installs the Emerald prefix with SRv6 encapsulation:
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24
  B  192.168.10.0/24 [200/0], SRv6 ...
RP/0/RP0/CPU0:G-R1# show bgp vrf CUST_A 192.168.10.0/24 | i "SRv6|SID|next hop"
    SRv6 SID: fc00:300:... : End.DT4
    Next Hop: 1.1.1.1
! End-to-end test: CE8 (Gold) pings CE1 (Emerald):
CE8# ping 192.168.10.2 source 192.168.20.2
```

---

## Task 4.5 — Customer B across Gold ↔ Garnet

**Question:** Customer B: CE9 (Gold, G-R1) ↔ CE4 (Garnet, Gar-R1). Use Option B across the Gold↔Garnet boundary (G-R5 ↔ Gar-R7) for contrast with 4.4's Option C.

**Solution (G-R5 — Gold side):**
```
router bgp 65300
 address-family vpnv4 unicast
  retain route-target all
 !
 neighbor 10.0.3.2               ! Gar-R7 (Garnet)
  remote-as 65200
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
 !
 neighbor 24.24.24.24            ! Gold RR (G-R4)
  remote-as 65300
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
```

**Solution (Gar-R7 — Garnet side):**
```
router bgp 65200
 address-family vpnv4 unicast
  retain route-target all
 !
 neighbor 10.0.3.1               ! G-R5 (Gold)
  remote-as 65300
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
 !
 neighbor 16.16.16.16            ! Garnet RR (Gar-R6)
  remote-as 65200
  update-source Loopback0
  address-family vpnv4 unicast
   next-hop-self
```

**Verification:**
```
RP/0/RP0/CPU0:Gar-R7# show bgp vpnv4 unicast rd 65300:200 192.168.110.0/24
  received from 10.0.3.1 (G-R5), best
RP/0/RP0/CPU0:Gar-R1# show route vrf CUST_B 192.168.110.0/24    ! CE9's prefix, next-hop Gar-R7
CE4# ping 192.168.110.2                                       ! Garnet CE reaches Gold CE
```

---

## Task 4.6 — Compare Options A / B / C

**Question:** Summarize scalability, config complexity, and use cases.

| Aspect | **Option A** (back-to-back VRF) | **Option B** (VPNv4 on ASBR) | **Option C** (BGP-LU + multihop VPNv4 RR-RR) |
|--------|-------------------------------|------------------------------|----------------------------------------------|
| Data plane at ASBR | IP (per-VRF) | Swap VPN label (new label) | No VPN state; label swap for LU only |
| Control plane | eBGP IPv4 per VRF | eBGP VPNv4 (1 session, all VRFs) | BGP-LU (loopbacks) + multihop eBGP VPNv4 RR↔RR |
| Per-VRF config on ASBR | **Yes** (sub-if + VRF + peer each) | No | No |
| Scalability | Poor (state per customer) | Good | **Best** (no per-VPN state on ASBR) |
| Config complexity | Simple concept, heavy per-customer | Moderate (`retain route-target all`, next-hop-self) | Highest (LU + multihop + next-hop-unchanged) |
| QoS / policy at boundary | **Full** (IP visible) | Limited (labeled) | Limited (labeled) |
| Next-hop handling | N/A (IP) | next-hop-self on ASBR | next-hop-unchanged RR↔RR; LU resolves PE loopbacks |
| RT retention on ASBR | via VRF import | `retain route-target all` | not needed (no VPNv4 on ASBR) |
| Typical use case | Few VRFs, strong policy/billing at edge, distrust between SPs | Moderate VRF count, single trusted boundary | Large scale, same admin or tight partners, end-to-end LSP wanted |

**Rule of thumb:** A = simplest but least scalable and most boundary control; B = the common inter-provider choice; C = intra-company or tightly-coupled SPs needing maximum scale and a single end-to-end LSP.

---

# Section 5 — Advanced VPN (4 tasks)

## Task 5.1 — RT-Constraint (route-target filtering, RFC 4684)

**Question:** RRs push all VPNv4 to every PE, wasting memory on PEs that don't hold that VRF. Enable **RT-Constraint** so each PE advertises the RTs it imports and the RR sends only matching routes.

**Solution (RR = E-R5 and each PE):**
```
! E-R1 — advertise RT membership
router bgp 65100
 address-family ipv4 rt-filter
 !
 neighbor 6.6.6.6
  address-family ipv4 rt-filter
!
! E-R5 (RR) — enable rt-filter to all clients
router bgp 65100
 address-family ipv4 rt-filter
 !
 neighbor 1.1.1.1
  address-family ipv4 rt-filter
   route-reflector-client
 !
 neighbor 2.2.2.2
  address-family ipv4 rt-filter
   route-reflector-client
```

**Verification:**
```
RP/0/RP0/CPU0:E-R5# show bgp ipv4 rt-filter
   ... 65012:100 advertised by 1.1.1.1, 2.2.2.2
RP/0/RP0/CPU0:E-R1# show bgp ipv4 rt-filter summary
! A PE with only CUST_A no longer receives CUST_B VPNv4 routes:
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rt 65013:200     ! empty
```

---

## Task 5.2 — BGP PIC Edge for fast VPN failover

**Question:** For dual-homed prefixes (e.g. CE8 via G-R1+G-R2), install a **backup path** in the FIB so failover is prefix-independent (sub-100 ms) instead of waiting for BGP reconvergence.

**Solution:**
```
! Enable additional-paths so the RR/PE keep a second path, plus PIC in the VRF
router bgp 65300
 address-family vpnv4 unicast
  additional-paths receive
  additional-paths send
  additional-paths selection route-policy ADD_PATHS
 !
 vrf CUST_A
  address-family ipv4 unicast
   additional-paths install backup     ! program backup into CEF
!
route-policy ADD_PATHS
  set path-selection backup 1 install
end-policy
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show cef vrf CUST_A 192.168.20.0/24 detail | i "backup|via"
   via 21.21.21.21 ... , protected            ! primary
   via 22.22.22.22 ... , backup, repair       ! backup pre-installed
RP/0/RP0/CPU0:E-R1# show bgp vrf CUST_A 192.168.20.0/24 | i "backup|best"
   Path #2: backup
```

---

## Task 5.3 — Internet access from a VRF (route-leaking, VRF-aware NAT concepts)

**Question:** Give Customer A VRF sites internet access. Show the two common models: (a) **route-leaking** a default from the global table into the VRF (and customer prefixes back), and (b) the VRF-aware NAT concept for overlapping private space.

**Solution (a) — leak default into VRF and customer subnets into global, via RT + policy:**
```
! Global internet default lives in a shared "INTERNET" VRF or the global table.
! Leak a default 0.0.0.0/0 into CUST_A and export CUST_A prefixes to the internet edge:
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   65012:100
   65500:1            ! INTERNET RT (default route tagged with this)
  export route-target
   65012:100
   65500:2            ! customer prefixes tagged for the internet-edge VRF
!
! On the internet-edge PE, originate default into VPN with RT 65500:1:
router bgp 65300
 vrf INTERNET
  address-family ipv4 unicast
   network 0.0.0.0/0
```

**Solution (b) — VRF-aware NAT concept (overlapping RFC1918):**
```
! Customer private prefixes overlap between VRFs → NAT at the internet edge PE.
! IOS-XR: dynamic NAT with vrf awareness (conceptual; requires NAT-capable line card / CGN):
! service cgn CUST_A_NAT
!  service-location preferred-active 0/1/CPU0
!  service-type nat44 nat1
!   portlimit 1024
!   inside-vrf CUST_A
!    map outside-vrf INTERNET address-pool 203.0.113.0/24
```
> Key idea: each VRF is NAT-translated into a **unique public pool** so overlapping `10.0.0.0/8` from different customers becomes globally unique before hitting the internet.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show route vrf CUST_A 0.0.0.0/0
  B*   0.0.0.0/0 [200/0] via <internet-edge> (leaked default)
CE1# ping 8.8.8.8 source 192.168.10.2
RP/0/RP0/CPU0:INET-PE# show cgn nat44 CUST_A_NAT statistics     ! translations counting up
```

---

## Task 5.4 — Per-VRF label vs per-CE label (label allocation modes)

**Question:** Compare the VPN label allocation modes and configure each. Explain the FIB/PPS tradeoff.

**Solution — the three modes:**
```
! Per-prefix (default XR for some releases): one label per VPN prefix — most labels, finest control
router bgp 65100
 vrf CUST_A
  address-family ipv4 unicast
   label mode per-prefix
!
! Per-VRF (aggregate): ONE label for the whole VRF — fewest labels, PE does an IP lookup in the VRF after pop
router bgp 65100
 vrf CUST_A
  address-family ipv4 unicast
   label mode per-vrf
!
! Per-CE (per next-hop): one label per CE next-hop — balance; no second lookup for directly attached CE
router bgp 65100
 vrf CUST_A
  address-family ipv4 unicast
   label mode per-ce
```

**Tradeoffs:**
- **per-prefix:** most label state, but egress PE can forward without a second lookup for every prefix. Highest label consumption.
- **per-vrf (aggregate):** one label per VRF → smallest LFIB, but the egress PE must do an **extra IP lookup** in the VRF after popping (needed for connected/aggregate). Best for scale; slightly more work per packet.
- **per-ce:** one label per CE next-hop → good balance; packet is switched straight to the CE with no VRF IP lookup, while keeping label count low. Preferred default for many designs.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65100:100 labels
   Network            Next Hop        Rcvd Label      Local Label
   192.168.10.0/24    192.168.1.2     nolabel         24016        ! per-prefix: unique
   192.168.11.0/24    192.168.1.2     nolabel         24016        ! per-ce: same CE → same label
RP/0/RP0/CPU0:E-R1# show mpls forwarding labels 24016
   Local  Outgoing  Prefix           Outgoing   Next Hop
   24016  Aggregate CUST_A: <per-vrf>  ...                        ! per-vrf: aggregate label, VRF lookup
```

---

# Section 6 — Troubleshooting (4 tasks)

## Task 6.1 — CE can't reach remote site (RT mismatch)

**Symptom:** CE8 (Gold) cannot reach CE1 (Emerald). VPNv4 route exists on the RR but never appears in CE8's PE VRF.

**Diagnose:**
```
! Route is in VPNv4 on Gold G-R1 but NOT in the VRF:
RP/0/RP0/CPU0:G-R1# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24     ! present
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24                    ! % Not found
! Compare RTs:
RP/0/RP0/CPU0:G-R1# show vrf CUST_A detail | i "Import|Export"
  Import VPN route-target: 65012:999      <-- WRONG (should be 65012:100)
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24 | i "Extended"
  Extended community: RT:65012:100
```

**Root cause:** G-R1's **import RT** (`65012:999`) does not match the **export RT** carried by the route (`65012:100`).

**Fix:**
```
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   no 65012:999
   65012:100
```

**Verify:**
```
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24    ! now installed (B)
CE8# ping 192.168.10.2                                       ! success
```

---

## Task 6.2 — VPN route received but not installed (next-hop unreachable — missing BGP-LU)

**Symptom:** In Option C, E-R1 has the remote Garnet prefix in VPNv4 but marks it **not best / inaccessible**.

**Diagnose:**
```
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65200:200 192.168.110.0/24
   Not advertised to any peer
   Path ... Next Hop: 11.11.11.11
   ... inaccessible                       <-- next-hop unreachable
RP/0/RP0/CPU0:E-R1# show route 11.11.11.11
   % Network not in table                 <-- no LSP/route to remote PE loopback
RP/0/RP0/CPU0:E-R1# show cef 11.11.11.11/32
   0.0.0.0/0 ... drop
```

**Root cause:** Option C relies on **BGP-LU** to carry the remote PE /32 loopbacks with labels across the AS boundary. The labeled-unicast session (or `allocate-label` / redistribution) is missing, so `11.11.11.11` has no LSP → VPNv4 next-hop is unresolved → route not installed.

**Fix (restore BGP-LU on E-R6):**
```
router bgp 65100
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 10.0.1.2
  remote-as 65200
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
```

**Verify:**
```
RP/0/RP0/CPU0:E-R1# show route 11.11.11.11               ! now a BGP-LU / IGP route
RP/0/RP0/CPU0:E-R1# show cef 11.11.11.11/32               ! labeled path (not drop)
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65200:200 192.168.110.0/24 | i best   ! now best
```

---

## Task 6.3 — as-override not configured (CE rejects route — own AS in path)

**Symptom:** CE8 (AS 65012) does not learn CE1's prefix even though G-R1's VRF has it.

**Diagnose:**
```
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24          ! present on PE (B)
RP/0/RP0/CPU0:G-R1# show bgp vrf CUST_A neighbors 192.168.12.2 advertised-routes | i 192.168.10
   192.168.10.0/24 ...                                            ! PE IS advertising it
CE8# show ip bgp 192.168.10.0
   % Network not in table                                         ! CE dropped it
CE8# show ip bgp neighbors 192.168.12.1 | i "denied|filtered"
   ... 3 accepted, 2 denied (AS-PATH loop)
```

**Root cause:** The AS_PATH toward CE8 is `65300 65012` (CE1's origin AS 65012 is still present). CE8's own AS is 65012, so its BGP loop-prevention rejects the update. **as-override** was never enabled on G-R1's CE8 neighbor.

**Fix:**
```
router bgp 65300
 vrf CUST_A
  neighbor 192.168.12.2
   address-family ipv4 unicast
    as-override
```

**Verify:**
```
CE8# show ip bgp 192.168.10.0
   65300 65300 65300        ! 65012 rewritten to PE AS → accepted
CE8# ping 192.168.10.2
```
> Also confirm **SoO** is set (Task 2.5) once as-override is on, or you re-open the dual-homing loop.

---

## Task 6.4 — OSPF PE-CE route loop (missing DN-bit or domain-id mismatch)

**Symptom:** Customer C's OSPF route (CE3/CE6 sites) flaps, or a route learned via MPLS gets pushed back into BGP and loops; or remote routes appear as external Type 5 with a bad metric.

**Diagnose:**
```
! Loop symptom: route toggles, or a VPN route re-enters BGP from OSPF:
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 172.16.40.0/24
   ... route churns between OSPF and BGP
! Check the LSA the PE injects — DN-bit should be SET on PE-originated summaries/externals:
RP/0/RP0/CPU0:Gar-R2# show ospf CUST_CE6 vrf CUST_C6 database summary detail | i "Options|DN"
   Options: (No DN)          <-- WRONG, DN-bit not set
! Domain-id mismatch symptom: routes cross as E2 instead of inter-area:
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 | i "O E2"
   O E2 172.16.10.0/24 ...   <-- should be O IA if domain-ids matched
RP/0/RP0/CPU0:E-R2# show run router ospf | i domain-id
   domain-id type 0005 value 000000010000
RP/0/RP0/CPU0:Gar-R2# show run router ospf | i domain-id
   domain-id type 0005 value 000000020000    <-- DIFFERENT → treated as external
```

**Root cause:**
1. **Missing DN-bit** → the PE that receives its own re-advertised LSA redistributes it back into BGP, creating a PE↔PE loop. XR sets the DN-bit automatically when redistributing BGP→OSPF; if `capability vrf-lite` was enabled (which suppresses DN-bit checking) the protection is defeated.
2. **domain-id mismatch** → routes cross the MPLS core as OSPF **external Type 5 (E2)** instead of **inter-area (Type 3)**, changing metrics/preference and potentially causing suboptimal or looping paths with a backdoor.

**Fix:**
```
! 1) Ensure DN-bit protection is active (do NOT set vrf-lite on a real PE):
router ospf CUST_CE6
 vrf CUST_C6
  no capability vrf-lite            ! keep DN-bit enforcement ON
!
! 2) Align domain-id on all PEs serving this customer:
router ospf CUST_CE6
 vrf CUST_C6
  domain-id type 0005 value 000000010000     ! match E-R2's value 1
```

**Verify:**
```
RP/0/RP0/CPU0:Gar-R2# show ospf CUST_CE6 vrf CUST_C6 database summary detail | i DN
   Options: (DN)                     ! DN-bit now set → no re-redistribution loop
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 | i "O IA"
   O IA 172.16.10.0/24 ...           ! now inter-area, core is transparent
```

---

## Quick Reference — VPN show commands (IOS-XR)

```
show vrf <name> detail                          # RD + import/export RTs
show bgp vpnv4 unicast summary                   # VPNv4 sessions
show bgp vpnv4 unicast rd <RD> <prefix>          # one VPNv4 route
show bgp vpnv4 unicast rt <RT>                   # all routes with an RT
show bgp vpnv4 unicast labels                    # local/rcvd labels
show route vrf <name> <prefix>                   # VRF RIB
show cef vrf <name> <prefix> detail              # VRF FIB + label stack + backup
show bgp vrf <name> neighbors <ce> advertised-routes
show ospf <proc> vrf <name> sham-links
show ospf <proc> vrf <name> database summary detail   # DN-bit check
show bgp ipv4 rt-filter                          # RT-Constraint membership
show bgp ipv4 labeled-unicast <PE-loopback>      # Option C BGP-LU
show mpls forwarding                             # LFIB
```

## Common gotchas

- On IOS-XR, applying `vrf X` to an interface **wipes the IP** — always re-enter `ipv4 address` after.
- `route-policy` is **mandatory** on eBGP neighbors (unlike IOS which permits all by default). A missing in/out policy silently drops everything.
- Inter-AS **Option B** ASBR needs `retain route-target all` or it discards VPNv4 whose RT it can't import.
- Inter-AS **Option C** breaks if PE loopbacks aren't reachable across the boundary (BGP-LU) — the classic "route received but not installed."
- `as-override` without **SoO** on a dual-homed CE re-introduces a routing loop.
- OSPF PE-CE: never enable `capability vrf-lite` on a production PE — it disables DN-bit loop protection.
