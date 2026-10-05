# WB05 L3VPN (MPLS VPN) — Solutions

**Reference:** `00_topology_reference.md` for PE-CE wiring and IPs.
**Prerequisite:** WB04 complete (iBGP + VPNv4 up, RRs reflecting). IGP + LDP in each core.
**Platform:** Cisco 7200, IOS 15.2(4)M11 — IOS classic syntax (uses `ip vrf`, not the newer `vrf definition`, for IPv4-only VRFs).

---

## Design Summary

| VRF | Customer | ASN | Route Target | RD convention | PEs (interface → CE) |
|-----|----------|-----|--------------|---------------|----------------------|
| GREEN | Green | 65910 | 65910:100 | `<PE-loopback>:100` | R1 (g1/0→R9, f3/0→R10), R2 (g1/0→R10), R15 (f0/0→R17 OSPF), R16 (g1/0→R18) |
| YELLOW | Yellow | 65024 | 65024:100 | `<PE-loopback>:100` | R16 (g2/0→R26), R20 (f3/0→R24), R22 (f0/0→R24) |
| BLUE | Blue | 65025 | 65025:100 | `<PE-loopback>:100` | R1 (f4/1→R32), R14 (f3/0→R25), R21 (g2/0→R25), R23 (f3/0→R25) |

**RD per-SP convention:** each PE uses `<its-loopback-as-dotted>:100`, e.g. R1 → `1.1.1.1:100`. Unique RD per PE per VRF guarantees unique VPNv4 NLRI, which is required so a RR/ASBR does not treat two PEs' identical customer prefixes as the same route (important for dual-homing and multipath). Route Target stays constant per customer so routes are imported into the right VRF everywhere.

> Note on cross-AS customers (Green spans 65100+65200, Blue spans 65100+65200+65300, Yellow spans 65200+65300): within each AS the VRF/RT config below is complete. Carrying VPNv4 **between** ASes requires Inter-AS Option A/B/C — see WB09. The per-AS RT value is kept identical (e.g. 65910:100) so that once inter-AS VPNv4 exchange is enabled, import/export "just works."

---

## Section 1: VRF Definitions (per PE)

### VRF GREEN

```
! R1
ip vrf GREEN
 rd 1.1.1.1:100
 route-target export 65910:100
 route-target import 65910:100
!
! R2
ip vrf GREEN
 rd 2.2.2.2:100
 route-target export 65910:100
 route-target import 65910:100
!
! R15
ip vrf GREEN
 rd 15.15.15.15:100
 route-target export 65910:100
 route-target import 65910:100
!
! R16
ip vrf GREEN
 rd 16.16.16.16:100
 route-target export 65910:100
 route-target import 65910:100
!
```

### VRF YELLOW

```
! R16
ip vrf YELLOW
 rd 16.16.16.16:101
 route-target export 65024:100
 route-target import 65024:100
!
! R20
ip vrf YELLOW
 rd 20.20.20.20:100
 route-target export 65024:100
 route-target import 65024:100
!
! R22
ip vrf YELLOW
 rd 22.22.22.22:100
 route-target export 65024:100
 route-target import 65024:100
!
```

> On R16 the RD for YELLOW is `16.16.16.16:101` to keep it distinct from GREEN's `16.16.16.16:100` on the same PE (RD must be unique per VRF per router).

### VRF BLUE

```
! R1
ip vrf BLUE
 rd 1.1.1.1:102
 route-target export 65025:100
 route-target import 65025:100
!
! R14
ip vrf BLUE
 rd 14.14.14.14:100
 route-target export 65025:100
 route-target import 65025:100
!
! R21
ip vrf BLUE
 rd 21.21.21.21:100
 route-target export 65025:100
 route-target import 65025:100
!
! R23
ip vrf BLUE
 rd 23.23.23.23:100
 route-target export 65025:100
 route-target import 65025:100
!
```

> R1 RD for BLUE is `1.1.1.1:102` (GREEN used `:100`), again to keep RDs unique per VRF on that PE.

---

## Section 2: PE-CE Interface Assignment

Assign each PE-CE interface into its VRF, then re-apply the IP (entering a VRF on an interface clears the IP, so order matters).

```
! R1
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
interface FastEthernet3/0
 ip vrf forwarding GREEN
 ip address 192.168.2.1 255.255.255.0
 no shutdown
!
interface FastEthernet4/1
 ip vrf forwarding BLUE
 ip address 192.168.3.1 255.255.255.0
 no shutdown
!
! R2
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.4.2 255.255.255.0
 no shutdown
!
! R14
interface FastEthernet3/0
 ip vrf forwarding BLUE
 ip address 192.168.5.14 255.255.255.0
 no shutdown
!
! R15
interface FastEthernet0/0
 ip vrf forwarding GREEN
 ip address 192.168.6.15 255.255.255.0
 no shutdown
!
! R16
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.7.16 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 ip vrf forwarding YELLOW
 ip address 192.168.8.16 255.255.255.0
 no shutdown
!
! R20
interface FastEthernet3/0
 ip vrf forwarding YELLOW
 ip address 192.168.9.20 255.255.255.0
 no shutdown
!
! R21
interface GigabitEthernet2/0
 ip vrf forwarding BLUE
 ip address 192.168.10.21 255.255.255.0
 no shutdown
!
! R22
interface FastEthernet0/0
 ip vrf forwarding YELLOW
 ip address 192.168.11.22 255.255.255.0
 no shutdown
!
! R23
interface FastEthernet3/0
 ip vrf forwarding BLUE
 ip address 192.168.12.23 255.255.255.0
 no shutdown
!
```

---

## Section 3: PE-CE eBGP (per VRF address-family)

Add these blocks to each PE's existing `router bgp` process (from WB04). The CE AS is the customer ASN; redistribute connected/BGP as needed so customer prefixes enter VPNv4.

### R1 — GREEN (R9, R10) + BLUE (R32)

```
router bgp 65100
 !
 address-family ipv4 vrf GREEN
  neighbor 192.168.1.9 remote-as 65910
  neighbor 192.168.1.9 activate
  neighbor 192.168.2.10 remote-as 65910
  neighbor 192.168.2.10 activate
  neighbor 192.168.2.10 as-override
  neighbor 192.168.2.10 route-map SOO-R10 in
  redistribute connected
 exit-address-family
 !
 address-family ipv4 vrf BLUE
  neighbor 192.168.3.32 remote-as 65025
  neighbor 192.168.3.32 activate
  redistribute connected
 exit-address-family
!
```

### R2 — GREEN (R10)

```
router bgp 65100
 !
 address-family ipv4 vrf GREEN
  neighbor 192.168.4.10 remote-as 65910
  neighbor 192.168.4.10 activate
  neighbor 192.168.4.10 as-override
  neighbor 192.168.4.10 route-map SOO-R10 in
  redistribute connected
 exit-address-family
!
```

### R16 — GREEN (R18) + YELLOW (R26)

```
router bgp 65200
 !
 address-family ipv4 vrf GREEN
  neighbor 192.168.7.18 remote-as 65910
  neighbor 192.168.7.18 activate
  redistribute connected
 exit-address-family
 !
 address-family ipv4 vrf YELLOW
  neighbor 192.168.8.26 remote-as 65024
  neighbor 192.168.8.26 activate
  redistribute connected
 exit-address-family
!
```

### R14 — BLUE (R25)

```
router bgp 65200
 !
 address-family ipv4 vrf BLUE
  neighbor 192.168.5.25 remote-as 65025
  neighbor 192.168.5.25 activate
  redistribute connected
 exit-address-family
!
```

### R20 / R22 — YELLOW (R24, dual-homed in AS65300)

```
! R20
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.9.24 remote-as 65024
  neighbor 192.168.9.24 activate
  redistribute connected
 exit-address-family
!
! R22
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.11.24 remote-as 65024
  neighbor 192.168.11.24 activate
  redistribute connected
 exit-address-family
!
```

> R24 (Yellow CE) is dual-homed to R20 and R22, both in AS65300 with the same customer AS 65024. If R24 re-advertises prefixes between the two PEs, its own AS 65024 in the path blocks the loop — but if you need the two sites to learn each other's routes through the SP you'd apply `as-override` on R20/R22 toward R24 as well. Shown without it here since R24 is a single CE (dual-attached), not two separate same-AS sites. SoO (Section 6) is the correct loop guard for this dual-homing case.

### R21 / R23 — BLUE (R25, dual-homed in AS65300)

```
! R21
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 remote-as 65025
  neighbor 192.168.10.25 activate
  redistribute connected
 exit-address-family
!
! R23
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 remote-as 65025
  neighbor 192.168.12.25 activate
  redistribute connected
 exit-address-family
!
```

> R25 (Blue CE) attaches to R14 (AS65200) and R21+R23 (AS65300) — three PE attachments across two SP ASes. SoO prevents routes learned from R25 at one PE from being readvertised back to R25 at another PE.

---

## Section 4: CE-side BGP (reference)

CEs run plain eBGP toward their PE(s). Example R10 (dual-homed Green CE to R1 and R2):

```
! R10
router bgp 65910
 bgp router-id 150.10.10.10
 no bgp default ipv4-unicast
 neighbor 192.168.2.1 remote-as 65100
 neighbor 192.168.4.2 remote-as 65100
 !
 address-family ipv4
  neighbor 192.168.2.1 activate
  neighbor 192.168.4.2 activate
  network 150.10.10.10 mask 255.255.255.255
  network 192.168.100.0
 exit-address-family
!
```

> R9 and R10 are also linked directly (192.168.100.0/24, f0/0↔f0/0). Advertising 192.168.100.0/24 from both gives the SP two paths and exercises as-override + SoO behavior.

---

## Section 5: PE-CE OSPF (R15 ↔ R17, VRF GREEN)

OSPF process 2 runs inside VRF GREEN. Redistribute between OSPF and BGP in both directions. Use a distinct process-id from any core OSPF (core is process 1 in AS65200 per WB02).

```
! R15
router ospf 2 vrf GREEN
 router-id 15.15.15.15
 network 192.168.6.0 0.0.0.255 area 0
 redistribute bgp 65200 subnets
!
router bgp 65200
 address-family ipv4 vrf GREEN
  redistribute ospf 2 vrf GREEN match internal external 1 external 2
 exit-address-family
!
```

```
! R17 (CE) — plain OSPF, no VRF
router ospf 1
 router-id 17.17.17.17
 network 192.168.6.0 0.0.0.255 area 0
 network 150.17.17.17 0.0.0.0 area 0
!
```

> MPLS VPN carries the OSPF domain via the BGP extended community "Domain-ID" and the sham-link mechanism when needed. For a single PE (R15) serving R17, no sham-link is required. The VPNv4 superbackbone transports R17's routes to other Green sites; they appear as inter-area or external (O IA / O E2) at the far CEs depending on redistribution. To preserve intra-area appearance across the MPLS backbone between two OSPF PE sites, a sham-link would be configured — not needed here as R17 is the only OSPF Green site.

---

## Section 6: as-override and SoO (dual-homing loop prevention)

### 6.1 as-override (R1, R2 toward R10)

R10 is in AS 65910, the same AS as R9 and R18. Without `as-override`, when a Green route (origin AS 65910) is sent to R10, R10 rejects it (its own AS in the AS-PATH). `as-override` rewrites occurrences of the CE's AS in the AS-PATH with the SP AS, so R10 accepts it.

Already applied in Section 3 on R1 (`neighbor 192.168.2.10 as-override`) and R2 (`neighbor 192.168.4.10 as-override`).

### 6.2 SoO — Site-of-Origin (R10 dual-homed interfaces)

`as-override` reopens a loop risk: a route R10 originates could come back to R10 via the other PE with the AS rewritten, and R10 would accept it. SoO tags routes with the originating site; a PE will not re-advertise a route back toward a CE interface carrying the same SoO.

Tag both R1→R10 and R2→R10 with the **same** SoO (same physical site = R10):

```
! Route-map referenced in Section 3 on both R1 (f3/0) and R2 (g1/0)
route-map SOO-R10 permit 10
 set extcommunity soo 65910:10
!
```

Apply inbound on the PE-CE neighbor (shown in Section 3). On IOS classic you can also set SoO directly on the neighbor:

```
! Equivalent per-neighbor form (alternative to the route-map)
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.2.10 soo 65910:10
 exit-address-family
!
```

```
! On R2
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.4.10 soo 65910:10
 exit-address-family
!
```

> Same SoO value `65910:10` on both R1 and R2 links to R10 = "these two links reach the same site." A PE drops a route on egress toward a CE if that route carries an SoO matching the egress link. This stops the as-override-induced loop for R10.

> For Blue R25 (three attachments: R14, R21, R23) apply a single SoO (e.g. `65025:25`) on all three PE-CE neighbors if R25 is treated as one site. If R25 represents distinct sites per link, use distinct SoO values instead.

---

## Section 7: Verification Commands

```
! VRF plumbing
show ip vrf
show ip vrf interfaces
show ip route vrf GREEN
show ip route vrf YELLOW
show ip route vrf BLUE

! PE-CE sessions
show ip bgp vpnv4 vrf GREEN summary
show ip bgp vpnv4 vrf BLUE neighbors 192.168.3.32

! VPNv4 label / RD / RT
show ip bgp vpnv4 all
show ip bgp vpnv4 all 150.9.9.9/32        ! confirm RD, RT, label, originator
show ip bgp vpnv4 rd 1.1.1.1:100

! Data plane
show mpls forwarding-table vrf GREEN
ping vrf GREEN 150.9.9.9 source 192.168.7.16    ! R16 -> R9 across the VPN
traceroute vrf BLUE 150.25.25.25

! OSPF PE-CE
show ip ospf 2
show ip route vrf GREEN ospf
show ip bgp vpnv4 vrf GREEN              ! confirm OSPF routes redistributed

! as-override / SoO
show ip bgp vpnv4 vrf GREEN 192.168.100.0
show ip bgp vpnv4 all 192.168.100.0      ! inspect AS-PATH rewrite + SoO extcomm
```

> Verify on R10 that it learns the far Green sites (R9/R18) with the SP AS in place of 65910 (as-override working), and that it does **not** receive its own 192.168.100.0/24 back from the second PE (SoO working).
