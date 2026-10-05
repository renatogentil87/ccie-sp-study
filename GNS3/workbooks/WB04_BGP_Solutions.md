# WB04 BGP — Solutions

**Reference:** `00_topology_reference.md` for loopbacks, core links, and inter-AS wiring.
**Prerequisite:** WB00–WB03 complete (IP, IGP, LDP working). Loopback0 reachable within each AS via IGP.
**Platform:** Cisco 7200, IOS 15.2(4)M11 — IOS classic syntax.

---

## Design Summary

| AS | iBGP model | Route Reflectors | RR clients | ASBRs (eBGP) |
|----|-----------|------------------|------------|--------------|
| 65100 | RR | R7, R8 | R1–R6 | R5, R6 |
| 65200 | RR | R19 | R11–R16 | R12 |
| 65300 | RR | R30 | R20–R23 | R20, R21 |

- All iBGP sessions use `update-source Loopback0` and `next-hop-self` on ASBRs/PEs where needed.
- `no bgp default ipv4-unicast` everywhere — address families activated explicitly.
- VPNv4 AF carries L3VPN routes (see WB05). IPv4 unicast AF used only on eBGP inter-AS links.
- RRs are clients of each other where there are two (R7↔R8 are peers, both reflect).

---

## Section 1: AS 65100 (R1–R8) — iBGP + VPNv4, RRs R7/R8

### R7 (Route Reflector)

```
router bgp 65100
 bgp router-id 150.1.7.7
 no bgp default ipv4-unicast
 neighbor 150.1.1.1 remote-as 65100
 neighbor 150.1.1.1 update-source Loopback0
 neighbor 150.1.2.2 remote-as 65100
 neighbor 150.1.2.2 update-source Loopback0
 neighbor 150.1.3.3 remote-as 65100
 neighbor 150.1.3.3 update-source Loopback0
 neighbor 150.1.4.4 remote-as 65100
 neighbor 150.1.4.4 update-source Loopback0
 neighbor 150.1.5.5 remote-as 65100
 neighbor 150.1.5.5 update-source Loopback0
 neighbor 150.1.6.6 remote-as 65100
 neighbor 150.1.6.6 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.1.1 activate
  neighbor 150.1.1.1 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.1.3.3 activate
  neighbor 150.1.3.3 route-reflector-client
  neighbor 150.1.4.4 activate
  neighbor 150.1.4.4 route-reflector-client
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 route-reflector-client
  neighbor 150.1.6.6 activate
  neighbor 150.1.6.6 route-reflector-client
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

> R7 reflects to all of R1–R6. R8 is a non-client peer (RR-to-RR). This keeps full visibility with redundancy.

### R8 (Route Reflector)

```
router bgp 65100
 bgp router-id 150.1.8.8
 no bgp default ipv4-unicast
 neighbor 150.1.1.1 remote-as 65100
 neighbor 150.1.1.1 update-source Loopback0
 neighbor 150.1.2.2 remote-as 65100
 neighbor 150.1.2.2 update-source Loopback0
 neighbor 150.1.3.3 remote-as 65100
 neighbor 150.1.3.3 update-source Loopback0
 neighbor 150.1.4.4 remote-as 65100
 neighbor 150.1.4.4 update-source Loopback0
 neighbor 150.1.5.5 remote-as 65100
 neighbor 150.1.5.5 update-source Loopback0
 neighbor 150.1.6.6 remote-as 65100
 neighbor 150.1.6.6 update-source Loopback0
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.1.1 activate
  neighbor 150.1.1.1 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.1.3.3 activate
  neighbor 150.1.3.3 route-reflector-client
  neighbor 150.1.4.4 activate
  neighbor 150.1.4.4 route-reflector-client
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 route-reflector-client
  neighbor 150.1.6.6 activate
  neighbor 150.1.6.6 route-reflector-client
  neighbor 150.1.7.7 activate
 exit-address-family
!
```

### R1 (PE) — RR client

```
router bgp 65100
 bgp router-id 150.1.1.1
 no bgp default ipv4-unicast
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

> PE-CE VRF neighbors (R9/R10/R32) are configured in WB05 under the per-VRF address-families.

### R2 (PE) — RR client

```
router bgp 65100
 bgp router-id 150.1.2.2
 no bgp default ipv4-unicast
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

### R3 (P) and R4 (P) — RR clients

> P routers do not carry VPNv4 by design, but we keep them as RR clients for lab visibility/consistency. In production, P routers run no BGP (label switching only). Config shown for R3; R4 identical with its own router-id/loopback.

```
router bgp 65100
 bgp router-id 150.1.3.3
 no bgp default ipv4-unicast
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

```
router bgp 65100
 bgp router-id 150.1.4.4
 no bgp default ipv4-unicast
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

### R5 (ASBR) — RR client + eBGP to R11 (AS65200)

```
router bgp 65100
 bgp router-id 150.1.5.5
 no bgp default ipv4-unicast
 ! iBGP to RRs
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 ! eBGP to R11 (inter-AS link 10.5.11.0/24)
 neighbor 10.5.11.11 remote-as 65200
 !
 address-family ipv4
  neighbor 10.5.11.11 activate
  network 150.1.5.5 mask 255.255.255.255
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

> eBGP uses the directly-connected interface IP (not loopback) — no `update-source` needed. IPv4 unicast AF only on the inter-AS link. VPNv4 exchange across this ASBR link is handled in WB09 (Inter-AS Options).

### R6 (ASBR) — RR client + eBGP to R12 (AS65200) and R20 (AS65300)

```
router bgp 65100
 bgp router-id 150.1.6.6
 no bgp default ipv4-unicast
 ! iBGP to RRs
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 ! eBGP to R12 (10.6.12.0/24) and R20 (10.6.20.0/24)
 neighbor 10.6.12.12 remote-as 65200
 neighbor 10.6.20.20 remote-as 65300
 !
 address-family ipv4
  neighbor 10.6.12.12 activate
  neighbor 10.6.20.20 activate
  network 150.1.6.6 mask 255.255.255.255
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

---

## Section 2: AS 65200 (R11–R16, R19) — iBGP + VPNv4, RR R19

### R19 (Route Reflector)

```
router bgp 65200
 bgp router-id 150.2.19.19
 no bgp default ipv4-unicast
 neighbor 150.2.11.11 remote-as 65200
 neighbor 150.2.11.11 update-source Loopback0
 neighbor 150.2.12.12 remote-as 65200
 neighbor 150.2.12.12 update-source Loopback0
 neighbor 150.2.13.13 remote-as 65200
 neighbor 150.2.13.13 update-source Loopback0
 neighbor 150.2.14.14 remote-as 65200
 neighbor 150.2.14.14 update-source Loopback0
 neighbor 150.2.15.15 remote-as 65200
 neighbor 150.2.15.15 update-source Loopback0
 neighbor 150.2.16.16 remote-as 65200
 neighbor 150.2.16.16 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.2.11.11 activate
  neighbor 150.2.11.11 route-reflector-client
  neighbor 150.2.12.12 activate
  neighbor 150.2.12.12 route-reflector-client
  neighbor 150.2.13.13 activate
  neighbor 150.2.13.13 route-reflector-client
  neighbor 150.2.14.14 activate
  neighbor 150.2.14.14 route-reflector-client
  neighbor 150.2.15.15 activate
  neighbor 150.2.15.15 route-reflector-client
  neighbor 150.2.16.16 activate
  neighbor 150.2.16.16 route-reflector-client
 exit-address-family
!
```

### R14, R15, R16 (PEs) — RR clients

> Shown for R14. R15 and R16 identical with their own router-id (150.2.15.15 / 150.2.16.16). PE-CE VRF neighbors added in WB05.

```
router bgp 65200
 bgp router-id 150.2.14.14
 no bgp default ipv4-unicast
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.2.19.19 activate
 exit-address-family
!
```

```
router bgp 65200
 bgp router-id 150.2.15.15
 no bgp default ipv4-unicast
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.2.19.19 activate
 exit-address-family
!
```

```
router bgp 65200
 bgp router-id 150.2.16.16
 no bgp default ipv4-unicast
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.2.19.19 activate
 exit-address-family
!
```

### R11, R13 (P) — RR clients

> Shown for R11; R13 identical with router-id 150.2.13.13.

```
router bgp 65200
 bgp router-id 150.2.11.11
 no bgp default ipv4-unicast
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.2.19.19 activate
 exit-address-family
!
```

### R12 (ASBR) — RR client + eBGP to R5/R6 (AS65100) and R21 (AS65300)

```
router bgp 65200
 bgp router-id 150.2.12.12
 no bgp default ipv4-unicast
 ! iBGP to RR
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 ! eBGP to R6 (10.6.12.0/24) and R21 (20.12.21.0/24)
 neighbor 10.6.12.6 remote-as 65100
 neighbor 20.12.21.21 remote-as 65300
 !
 address-family ipv4
  neighbor 10.6.12.6 activate
  neighbor 20.12.21.21 activate
  network 150.2.12.12 mask 255.255.255.255
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.2.19.19 activate
 exit-address-family
!
```

> Note: the topology also has R5↔R11 (10.5.11.0/24) as an AS65100↔AS65200 link. R11 is a P router; the eBGP peering on that link terminates on R11. If you want R11 to carry the eBGP session, add the block below to R11. The requirement lists "R5↔R11" eBGP, so configure it on R11:

### R11 — eBGP to R5 (AS65100) [per requirement R5↔R11]

```
router bgp 65200
 ! (added to R11's existing iBGP config above)
 neighbor 10.5.11.5 remote-as 65100
 !
 address-family ipv4
  neighbor 10.5.11.5 activate
  network 150.2.11.11 mask 255.255.255.255
 exit-address-family
!
```

---

## Section 3: AS 65300 (R20–R23, R30) — iBGP + VPNv4, RR R30

### R30 (Route Reflector)

```
router bgp 65300
 bgp router-id 150.3.30.30
 no bgp default ipv4-unicast
 neighbor 150.3.20.20 remote-as 65300
 neighbor 150.3.20.20 update-source Loopback0
 neighbor 150.3.21.21 remote-as 65300
 neighbor 150.3.21.21 update-source Loopback0
 neighbor 150.3.22.22 remote-as 65300
 neighbor 150.3.22.22 update-source Loopback0
 neighbor 150.3.23.23 remote-as 65300
 neighbor 150.3.23.23 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.3.20.20 activate
  neighbor 150.3.20.20 route-reflector-client
  neighbor 150.3.21.21 activate
  neighbor 150.3.21.21 route-reflector-client
  neighbor 150.3.22.22 activate
  neighbor 150.3.22.22 route-reflector-client
  neighbor 150.3.23.23 activate
  neighbor 150.3.23.23 route-reflector-client
 exit-address-family
!
```

### R22, R23 (PEs) — RR clients

> Shown for R22; R23 identical with router-id 150.3.23.23. PE-CE VRF neighbors in WB05.

```
router bgp 65300
 bgp router-id 150.3.22.22
 no bgp default ipv4-unicast
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.3.30.30 activate
 exit-address-family
!
```

### R20 (ASBR/PE) — RR client + eBGP to R6 (AS65100)

```
router bgp 65300
 bgp router-id 150.3.20.20
 no bgp default ipv4-unicast
 ! iBGP to RR
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 update-source Loopback0
 ! eBGP to R6 (10.6.20.0/24)
 neighbor 10.6.20.6 remote-as 65100
 !
 address-family ipv4
  neighbor 10.6.20.6 activate
  network 150.3.20.20 mask 255.255.255.255
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.3.30.30 activate
 exit-address-family
!
```

### R21 (ASBR/PE) — RR client + eBGP to R12 (AS65200)

```
router bgp 65300
 bgp router-id 150.3.21.21
 no bgp default ipv4-unicast
 ! iBGP to RR
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 update-source Loopback0
 ! eBGP to R12 (20.12.21.0/24)
 neighbor 20.12.21.12 remote-as 65200
 !
 address-family ipv4
  neighbor 20.12.21.12 activate
  network 150.3.21.21 mask 255.255.255.255
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.3.30.30 activate
 exit-address-family
!
```

---

## Section 4: eBGP Inter-AS Summary

| Link | Local router | Local IP | Remote router | Remote IP | Local AS | Remote AS |
|------|-------------|----------|---------------|-----------|----------|-----------|
| R5↔R11 | R5 | 10.5.11.5 | R11 | 10.5.11.11 | 65100 | 65200 |
| R6↔R12 | R6 | 10.6.12.6 | R12 | 10.6.12.12 | 65100 | 65200 |
| R6↔R20 | R6 | 10.6.20.6 | R20 | 10.6.20.20 | 65100 | 65300 |
| R12↔R21 | R12 | 20.12.21.12 | R21 | 20.12.21.21 | 65200 | 65300 |

> These are IPv4 unicast eBGP sessions. For VPNv4 exchange across these borders (Inter-AS Option A/B/C), see WB09.

---

## Section 5: Path Manipulation Examples

These are generic templates. Apply the route-map inbound/outbound on the relevant eBGP neighbor. Example target: influence how AS65100 reaches AS65200 prefixes.

### 5.1 LOCAL_PREF (inbound — prefer a path into the local AS)

Applied on R6 to prefer routes learned from R12 over R20 for a given prefix.

```
ip prefix-list PFX-CUST seq 5 permit 150.18.18.18/32
!
route-map LP-IN permit 10
 match ip address prefix-list PFX-CUST
 set local-preference 200
route-map LP-IN permit 20
!
router bgp 65100
 address-family ipv4
  neighbor 10.6.12.12 route-map LP-IN in
 exit-address-family
!
```

> Higher LOCAL_PREF (200 vs default 100) wins and is propagated to all iBGP routers in AS65100, steering egress toward R12.

### 5.2 AS-PATH Prepend (outbound — make a path less attractive to neighbor AS)

Applied on R20 so AS65300 appears longer when advertised to AS65100 via R6.

```
route-map PREPEND-OUT permit 10
 set as-path prepend 65300 65300 65300
!
router bgp 65300
 address-family ipv4
  neighbor 10.6.20.6 route-map PREPEND-OUT out
 exit-address-family
!
```

> Receiving AS sees AS-PATH `65300 65300 65300 65300` — longer path, de-preferred vs an alternate entry point.

### 5.3 MED (outbound — hint to neighbor AS which of our links to use)

Applied on R5 and R6 (both AS65100→AS65200 entry points) so AS65200 prefers the lower MED.

```
! On R5 (preferred entry): low MED
route-map MED-R5 permit 10
 set metric 50
!
router bgp 65100
 address-family ipv4
  neighbor 10.5.11.11 route-map MED-R5 out
 exit-address-family
!
```

```
! On R6 (backup entry): high MED
route-map MED-R6 permit 10
 set metric 200
!
router bgp 65100
 address-family ipv4
  neighbor 10.6.12.12 route-map MED-R6 out
 exit-address-family
!
```

> MED is only comparable for routes from the same neighbor AS. AS65200 picks R5 (MED 50) over R6 (MED 200). Enable `bgp always-compare-med` if MEDs from different ASes must be compared.

### 5.4 Communities (tag on ingress, act on egress)

Tag customer routes with a community on the PE, then filter/act on it at the ASBR.

```
ip community-list standard CUST-GREEN permit 65100:910
!
! Tag inbound from CE (on PE, e.g. R1) — note: within VRF context in WB05
route-map SET-COMM permit 10
 set community 65100:910
!
! Match the community at the ASBR to set no-export, etc.
route-map COMM-ACT permit 10
 match community CUST-GREEN
 set community no-export additive
route-map COMM-ACT permit 20
!
router bgp 65100
 address-family vpnv4
  neighbor 150.1.7.7 send-community both
 exit-address-family
!
```

> Always `send-community both` (standard + extended) on VPNv4 sessions — extended communities carry the Route Targets, so this is mandatory for L3VPN, not just optional for policy.

---

## Section 6: Verification Commands

```
! Session state
show ip bgp summary
show bgp vpnv4 unicast all summary
show ip bgp neighbors <ip>

! iBGP / RR reflection
show ip bgp
show bgp vpnv4 unicast all
show ip bgp 150.18.18.18/32          ! trace best-path selection / attributes

! eBGP inter-AS
show ip bgp neighbors 10.5.11.11 advertised-routes
show ip bgp neighbors 10.6.12.6 received-routes      ! requires soft-reconfig inbound

! Path manipulation checks
show ip bgp 150.18.18.18/32          ! confirm local-pref, as-path, med
show ip bgp community 65100:910
show route-map

! RR-specific
show ip bgp vpnv4 all               ! on RR, confirm originator-id / cluster-list
show bgp vpnv4 unicast all neighbors <client> advertised-routes
```

> On a route-reflector, verify `Originator-ID` and `Cluster-list` attributes appear on reflected routes (loop prevention). On RR clients, confirm you receive VPNv4 NLRI only from the RR, not full-mesh.
