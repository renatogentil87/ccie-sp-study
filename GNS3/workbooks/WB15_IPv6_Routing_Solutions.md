# WB15 Full IPv6 Routing — Solutions

**Reference:** `00_topology_reference.md` for IPv6 addressing and wiring.
**Platform:** Cisco 7200, IOS 15.2(4)M11. IOS classic syntax.
**Prerequisite:** WB00–WB11 complete (IPv4 IGP/BGP operational).

> **Goal:** Native dual-stack IPv6 routing across all three ASes: IS-IS IPv6 single-topology in AS65100 and AS65300, OSPFv3 in AS65200, and MP-BGP IPv6 unicast between ASes (eBGP at ASBRs, iBGP via RRs). End-to-end IPv6 reachability between all loopbacks.

---

## Section 0 — Enable IPv6 globally (ALL routers)

```
ipv6 unicast-routing
ipv6 cef
```

> Apply on every router R1–R30 (and CEs as needed). Without `ipv6 unicast-routing` no IPv6 routing protocol runs and the box only does host IPv6.

---

## Section 1 — IPv6 Addressing (per topology reference)

Assign loopbacks `2001:db8:X::Y/128` and link addresses per the reference. Examples:

```
! R1 (AS65100)
interface Loopback0
 ipv6 address 2001:db8:1::1/128
interface FastEthernet0/0        ! → R3
 ipv6 address 2001:db8:1:13::1/64
interface FastEthernet4/0        ! → R2
 ipv6 address 2001:db8:1:12::1/64
!
! R11 (AS65200)
interface Loopback0
 ipv6 address 2001:db8:2::11/128
interface FastEthernet0/0        ! → R12
 ipv6 address 2001:db8:2:1112::11/64
interface GigabitEthernet2/0     ! → R13
 ipv6 address 2001:db8:2:1113::11/64
interface FastEthernet3/0        ! → R15
 ipv6 address 2001:db8:2:1115::11/64
!
! R20 (AS65300)
interface Loopback0
 ipv6 address 2001:db8:3::20/128
interface GigabitEthernet1/0     ! → R21
 ipv6 address 2001:db8:3:2021::20/64
interface GigabitEthernet2/0     ! → R22
 ipv6 address 2001:db8:3:2022::20/64
!
! Inter-AS links (2001:db8:0:XY::X/64)
! R5 g1/0 → R11:  2001:db8:0:511::5/64     (R11: ::11)
! R6 g1/0 → R12:  2001:db8:0:612::6/64     (R12: ::12)
! R6 f0/0 → R20:  2001:db8:0:620::6/64     (R20: ::20)
! R12 f3/0 → R21: 2001:db8:0:1221::12/64   (R21: ::21)
```

> Complete all interfaces for every router using the reference tables (Core Links IPv6, Inter-AS IPv6, Loopbacks IPv6).

---

## Section 2 — IS-IS IPv6 (AS65100 + AS65300, single-topology)

Reuse the existing `router isis CORE` process. Add the IPv6 address-family in single-topology mode and `ipv6 router isis CORE` on each core interface.

### 2.1 Example R1 (AS65100)

```
router isis CORE
 address-family ipv6 unicast
  single-topology
 exit-address-family
!
interface Loopback0
 ipv6 router isis CORE
interface FastEthernet0/0
 ipv6 router isis CORE
interface FastEthernet4/0
 ipv6 router isis CORE
!
```

### 2.2 Example R20 (AS65300)

```
router isis CORE
 address-family ipv6 unicast
  single-topology
 exit-address-family
!
interface Loopback0
 ipv6 router isis CORE
interface GigabitEthernet1/0
 ipv6 router isis CORE
interface GigabitEthernet2/0
 ipv6 router isis CORE
!
```

> Apply the same pattern on every router in AS65100 (R1–R8) and AS65300 (R20–R23, R30) on their **core** interfaces only (not inter-AS or PE-CE — those run MP-BGP).
> **CRITICAL:** `single-topology` must be on **every** router in the domain. A mismatch (single vs multi-topology) causes IPv6 routes not to install (TLV 236 vs 237). In single-topology, IPv4 and IPv6 share the same SPF/metrics — every link must be dual-stack.

### IS-IS IPv6 Verification

```
show isis neighbors                              ! adjacencies still UP (shared with IPv4)
show ipv6 route isis                             ! all intra-AS IPv6 loopbacks
show isis ipv6 topology                           ! single topology, all nodes
ping ipv6 2001:db8:1::8 source 2001:db8:1::1     ! R1→R8 (AS65100)
ping ipv6 2001:db8:3::30 source 2001:db8:3::20   ! R20→R30 (AS65300)
```

---

## Section 3 — OSPFv3 (AS65200)

Two valid IOS classic forms — use **OSPFv3 address-family** form (modern) for dual-stack, shown first; the traditional `ipv6 router ospf` form shown as alternative.

### 3.1 OSPFv3 address-family form (R11–R16, R19)

```
! Example R11
router ospfv3 1
 router-id 150.2.11.11
 address-family ipv6 unicast
  passive-interface Loopback0
 exit-address-family
!
interface Loopback0
 ospfv3 1 ipv6 area 0
interface FastEthernet0/0        ! → R12
 ospfv3 1 ipv6 area 0
interface GigabitEthernet2/0     ! → R13
 ospfv3 1 ipv6 area 0
interface FastEthernet3/0        ! → R15
 ospfv3 1 ipv6 area 0
!
```

### 3.2 Alternative — traditional `ipv6 router ospf`

```
! Example R11
ipv6 router ospf 1
 router-id 150.2.11.11
 passive-interface Loopback0
!
interface Loopback0
 ipv6 ospf 1 area 0
interface FastEthernet0/0
 ipv6 ospf 1 area 0
interface GigabitEthernet2/0
 ipv6 ospf 1 area 0
interface FastEthernet3/0
 ipv6 ospf 1 area 0
!
```

> Apply on all AS65200 core interfaces (R11, R12, R13, R14, R15, R16, R19) in area 0. `router-id` is mandatory for OSPFv3 — it has no IPv4 to borrow if interfaces are IPv6-only; set it explicitly to the IPv4 loopback value. Note the R12 inter-AS link to R21 is **not** in OSPFv3 (it runs eBGP IPv6).

### OSPFv3 Verification

```
show ospfv3 neighbor         (or: show ipv6 ospf neighbor)
show ipv6 route ospf                              ! all AS65200 IPv6 loopbacks
ping ipv6 2001:db8:2::16 source 2001:db8:2::11   ! R11→R16
```

---

## Section 4 — MP-BGP IPv6 Unicast (inter-AS + intra-AS)

### 4.1 eBGP IPv6 on ASBR pairs

```
! R5 ↔ R11 (65100 ↔ 65200)
! R5
router bgp 65100
 neighbor 2001:db8:0:511::11 remote-as 65200
 address-family ipv6 unicast
  neighbor 2001:db8:0:511::11 activate
  network 2001:db8:1::1/128                 ! advertise AS65100 PE loopbacks
  network 2001:db8:1::2/128
 exit-address-family
!
! R11
router bgp 65200
 neighbor 2001:db8:0:511::5 remote-as 65100
 address-family ipv6 unicast
  neighbor 2001:db8:0:511::5 activate
 exit-address-family
!
```

```
! R6 ↔ R12 (65100 ↔ 65200)
! R6
router bgp 65100
 neighbor 2001:db8:0:612::12 remote-as 65200
 address-family ipv6 unicast
  neighbor 2001:db8:0:612::12 activate
  network 2001:db8:1::1/128
  network 2001:db8:1::2/128
 exit-address-family
!
! R12
router bgp 65200
 neighbor 2001:db8:0:612::6 remote-as 65100
 address-family ipv6 unicast
  neighbor 2001:db8:0:612::6 activate
 exit-address-family
!
```

```
! R6 ↔ R20 (65100 ↔ 65300)
! R6
router bgp 65100
 neighbor 2001:db8:0:620::20 remote-as 65300
 address-family ipv6 unicast
  neighbor 2001:db8:0:620::20 activate
 exit-address-family
!
! R20
router bgp 65300
 neighbor 2001:db8:0:620::6 remote-as 65100
 address-family ipv6 unicast
  neighbor 2001:db8:0:620::6 activate
  network 2001:db8:3::20/128                ! advertise AS65300 PE loopbacks
  network 2001:db8:3::21/128
  network 2001:db8:3::22/128
  network 2001:db8:3::23/128
 exit-address-family
!
```

```
! R12 ↔ R21 (65200 ↔ 65300)
! R12
router bgp 65200
 neighbor 2001:db8:0:1221::21 remote-as 65300
 address-family ipv6 unicast
  neighbor 2001:db8:0:1221::21 activate
 exit-address-family
!
! R21
router bgp 65300
 neighbor 2001:db8:0:1221::12 remote-as 65200
 address-family ipv6 unicast
  neighbor 2001:db8:0:1221::12 activate
  network 2001:db8:3::20/128
  network 2001:db8:3::21/128
  network 2001:db8:3::22/128
  network 2001:db8:3::23/128
 exit-address-family
!
```

### 4.2 iBGP IPv6 via RRs

```
! AS65100 — ASBRs R5/R6 to RRs R7/R8 (example R5)
router bgp 65100
 neighbor 2001:db8:1::7 remote-as 65100
 neighbor 2001:db8:1::7 update-source Loopback0
 neighbor 2001:db8:1::8 remote-as 65100
 neighbor 2001:db8:1::8 update-source Loopback0
 address-family ipv6 unicast
  neighbor 2001:db8:1::7 activate
  neighbor 2001:db8:1::7 next-hop-self
  neighbor 2001:db8:1::8 activate
  neighbor 2001:db8:1::8 next-hop-self
 exit-address-family
!
! R7/R8 (RR): activate R1,R2,R5,R6 under ipv6 AF with route-reflector-client
router bgp 65100
 address-family ipv6 unicast
  neighbor 2001:db8:1::1 activate
  neighbor 2001:db8:1::1 route-reflector-client
  neighbor 2001:db8:1::2 activate
  neighbor 2001:db8:1::2 route-reflector-client
  neighbor 2001:db8:1::5 activate
  neighbor 2001:db8:1::5 route-reflector-client
  neighbor 2001:db8:1::6 activate
  neighbor 2001:db8:1::6 route-reflector-client
 exit-address-family
!
```

```
! AS65200 — all speakers to RR R19 (2001:db8:2::19)
! AS65300 — all speakers to RR R30 (2001:db8:3::30)
! Same pattern: update-source Loopback0, activate under ipv6 AF,
! route-reflector-client on the RR, next-hop-self on ASBRs toward RR.
```

> ASBRs use `next-hop-self` toward the RR so the eBGP-learned IPv6 next-hop (inter-AS link address) becomes the ASBR loopback, reachable via the intra-AS IGP (IS-IS/OSPFv3). Transport uses the IPv6 loopback (`update-source Loopback0`).

### MP-BGP IPv6 Verification

```
show bgp ipv6 unicast summary                    ! eBGP + iBGP neighbors Up
show bgp ipv6 unicast                             ! remote-AS loopbacks present
show bgp ipv6 unicast 2001:db8:3::23/128          ! path AS65300, next-hop resolved
show ipv6 route bgp                                ! remote-AS loopbacks installed
```

---

## Section 5 — End-to-End Verification (all 3 ASes)

```
show ipv6 route                                   ! intra (IS-IS/OSPFv3) + inter (BGP) prefixes
show bgp ipv6 unicast summary

! Cross-AS pings (source from loopback)
R1#  ping ipv6 2001:db8:2::14 source 2001:db8:1::1    ! AS65100 → AS65200
R1#  ping ipv6 2001:db8:3::23 source 2001:db8:1::1    ! AS65100 → AS65300
R14# ping ipv6 2001:db8:3::23 source 2001:db8:2::14   ! AS65200 → AS65300
R23# ping ipv6 2001:db8:1::1  source 2001:db8:3::23   ! AS65300 → AS65100 (return path)

traceroute ipv6 2001:db8:3::23                        ! verify ASBR hops R5/R6, R12/R20/R21
```

---

## Section 6 — Troubleshooting

### 6.1 IS-IS IPv6 routes missing
- `single-topology` mismatch across the domain → IPv6 TLVs not processed. Make it consistent on all nodes.
- Interface missing `ipv6 router isis CORE` → link not advertised.
- Link not dual-stacked (single-topology shares IPv4 SPF) — every IS-IS link needs an IPv6 address.

### 6.2 OSPFv3 adjacency down
- Missing `router-id` on an IPv6-only box → OSPFv3 won't start. Set it manually.
- Interface not enabled (`ospfv3 1 ipv6 area 0` / `ipv6 ospf 1 area 0`) or area mismatch.
- MTU mismatch on the link.

### 6.3 BGP IPv6 prefix not best / next-hop unreachable
- ASBR missing `next-hop-self` toward RR → inter-AS link next-hop not in IGP → prefix invalid.
  `show bgp ipv6 unicast <prefix>` shows next-hop inaccessible.
- `activate` missing under `address-family ipv6 unicast` → neighbor up for IPv4 only, no IPv6 NLRI.
- `update-source Loopback0` missing on iBGP → session uses wrong source, loopback-to-loopback fails.

---

## Completion Checklist

- [ ] `ipv6 unicast-routing` + `ipv6 cef` on all routers
- [ ] IPv6 addresses on all loopbacks and links per reference
- [ ] IS-IS IPv6 single-topology on AS65100 (R1–R8) + AS65300 (R20–R23, R30)
- [ ] OSPFv3 area 0 on AS65200 (R11–R16, R19) with explicit router-ids
- [ ] eBGP IPv6 up on all 4 ASBR pairs; PE loopbacks advertised
- [ ] iBGP IPv6 via RRs (R7/R8, R19, R30) with next-hop-self on ASBRs
- [ ] `show bgp ipv6 unicast summary` all neighbors Up
- [ ] Cross-AS IPv6 ping R1↔R23 and R14↔R23 succeed
- [ ] **GNS3 snapshot:** `wb15-ipv6-routing-complete`
