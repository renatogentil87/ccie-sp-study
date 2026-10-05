# WB12 6PE & 6VPE — Solutions

**Reference:** `00_topology_reference.md` for IP addressing, NET-IDs, and wiring.
**Platform:** Cisco 7200, IOS 15.2(4)M11. IOS classic syntax.
**Prerequisite:** WB00–WB11 complete (IGP, LDP, iBGP via RRs R7/R8, L3VPN all operational in AS65100).

> **Goal:** Carry IPv6 across the IPv4-only MPLS core of AS65100 without enabling IPv6 on the P routers (R3, R4). 6PE = global IPv6; 6VPE = IPv6 inside a VRF. The core switches on existing IPv4 LDP LSPs.

---

## Section 1 — 6PE (Global IPv6 over IPv4 MPLS)

**Concept:** PEs R1/R2 exchange IPv6 prefixes over the existing IPv4 iBGP sessions to RRs R7/R8. The IPv6 next-hop is encoded as an IPv4-mapped address (`::FFFF:a.b.c.d`) so the receiving PE resolves it over the IPv4 LSP. `send-label` binds an MPLS label to each IPv6 prefix. P routers never see IPv6.

### 1.1 Enable IPv6 forwarding on PEs only (R1, R2)

```
! R1 and R2 — identical
ipv6 unicast-routing
ipv6 cef
```

> Do **NOT** add `ipv6 unicast-routing` on R3/R4/R7/R8 forwarding path. R7/R8 only need the BGP IPv6 AF activated for reflection (control-plane), not IPv6 CEF.

### 1.2 IPv6 PE-CE addressing (global) — R1↔R9

Link R1 g1/0 ↔ R9 g1/0 (`192.168.1.0/24`). Add IPv6 `2001:db8:1:19::/64`.

```
! R1
interface Loopback0
 ipv6 address 2001:db8:1::1/128
!
interface GigabitEthernet1/0
 ipv6 address 2001:db8:1:19::1/64
!
```

```
! R9 (CE, AS65910)
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:db8:9::9/128
!
interface GigabitEthernet1/0
 ipv6 address 2001:db8:1:19::9/64
!
router bgp 65910
 !
 address-family ipv6 unicast
  neighbor 2001:db8:1:19::1 remote-as 65100
  network 2001:db8:9::9/128
  redistribute connected
 exit-address-family
!
```

> R9 advertises its IPv6 loopback `2001:db8:9::9/128` to R1 over eBGP IPv6.

### 1.3 R1 PE — eBGP to R9 (IPv6) + iBGP IPv6 to RRs with labels

```
! R1
router bgp 65100
 ! eBGP IPv6 to CE R9
 neighbor 2001:db8:1:19::9 remote-as 65910
 ! iBGP IPv4 transport sessions to RRs already exist (R7/R8)
 !
 address-family ipv6 unicast
  ! eBGP CE neighbor
  neighbor 2001:db8:1:19::9 activate
  ! iBGP to RRs over the IPv4 transport session — carry IPv6 with a label
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-label
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-label
 exit-address-family
!
```

> `neighbor 150.1.7.7` is the existing IPv4 iBGP session. Activating it under `address-family ipv6 unicast` carries IPv6 NLRI over that IPv4 session. `send-label` is mandatory — without it the core cannot label-switch IPv6.

### 1.4 R2 PE — iBGP IPv6 to RRs with labels (receiving side)

```
! R2
router bgp 65100
 address-family ipv6 unicast
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-label
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-label
 exit-address-family
!
```

### 1.5 RRs R7/R8 — reflect IPv6 unicast with labels

```
! R7 and R8 — identical (clients R1 and R2)
router bgp 65100
 address-family ipv6 unicast
  neighbor 150.1.1.1 activate
  neighbor 150.1.1.1 send-label
  neighbor 150.1.1.1 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 send-label
  neighbor 150.1.2.2 route-reflector-client
 exit-address-family
!
```

> RRs reflect the IPv6 prefix **without rewriting the next-hop**. The next-hop stays the IPv4-mapped address of the originating PE (`::FFFF:150.1.1.1`), which the receiving PE resolves over the IPv4 LDP LSP.

### 1.6 IPv4-mapped next-hop

6PE automatically encodes the next-hop as `::FFFF:<BGP-router-id/update-source>`. Because the PE↔RR session is IPv4 and sourced from Loopback0, R1 advertises next-hop `::FFFF:150.1.1.1`. No extra config is required for IOS classic; just ensure the IPv6 AF neighbor uses the IPv4 session (update-source Loopback0 already set in WB04).

> If the next-hop ever shows a link-local or non-mapped value, apply `neighbor X next-hop-self` under the IPv6 AF on the PE, or `route-map SET-V4MAP out` setting `ipv6 next-hop ::FFFF:150.1.1.1`.

### Section 1 Verification

```
! R1/R2
show bgp ipv6 unicast summary                 ! RRs Up, prefixes exchanged
show bgp ipv6 unicast                          ! R9 loopback present
show bgp ipv6 unicast 2001:db8:9::9/128        ! next-hop ::FFFF:150.1.1.1, in/out label set
show mpls forwarding-table                     ! IPv6 FEC with local label on R1
! R3/R4 (core)
show ipv6 route                                ! EMPTY — core is IPv4-only
show mpls forwarding-table                     ! only IPv4 FECs
! Data plane
R10# ping ipv6 2001:db8:9::9 source 2001:db8:10::10   ! (once R10 side configured, Section mirror)
traceroute ipv6 2001:db8:9::9                   ! P routers appear as MPLS label hops, not IPv6 hops
```

---

## Section 2 — 6VPE (IPv6 L3VPN / VPNv6)

**Concept:** IPv6 inside VRF **BLUE** (customer R32, AS65025). The VPN label (bottom of stack) identifies the VRF; the IPv4 LDP label (top) carries the packet across R3/R4. BGP SAFI = `vpnv6`.

### 2.1 Add IPv6 AF to VRF BLUE (dual-stack VRF) on R1

```
! R1 — VRF BLUE already exists from WB05 with IPv4 AF
vrf definition BLUE
 rd 65100:25
 !
 address-family ipv4
  route-target export 65100:25
  route-target import 65100:25
 exit-address-family
 !
 address-family ipv6
  route-target export 65100:25
  route-target import 65100:25
 exit-address-family
!
```

> `vrf definition` (not `ip vrf`) is required for a dual-stack VRF with IPv6 AF. If WB05 used legacy `ip vrf BLUE`, migrate it with `vrf upgrade-cli multi-af-mode common-policies vrf BLUE` or redefine as above.

### 2.2 IPv6 PE-CE inside VRF BLUE — R1↔R32

Link R1 f4/1 ↔ R32 f0/0 (`192.168.3.0/24`). Add VRF IPv6 `2001:db8:1:332::/64`.

```
! R1
interface FastEthernet4/1
 vrf forwarding BLUE
 ip address 192.168.3.1 255.255.255.0
 ipv6 address 2001:db8:1:332::1/64
!
router bgp 65100
 address-family ipv6 vrf BLUE
  neighbor 2001:db8:1:332::32 remote-as 65025
  neighbor 2001:db8:1:332::32 activate
 exit-address-family
!
```

```
! R32 (CE, AS65025)
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:db8:32::32/128
!
interface FastEthernet0/0
 ipv6 address 2001:db8:1:332::32/64
!
router bgp 65025
 address-family ipv6 unicast
  neighbor 2001:db8:1:332::1 remote-as 65100
  network 2001:db8:32::32/128
  redistribute connected
 exit-address-family
!
```

### 2.3 Activate VPNv6 on PE↔RR sessions

```
! R1 and R2 (PEs)
router bgp 65100
 address-family vpnv6 unicast
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community extended
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community extended
 exit-address-family
!
```

```
! R7 and R8 (RRs)
router bgp 65100
 address-family vpnv6 unicast
  neighbor 150.1.1.1 activate
  neighbor 150.1.1.1 send-community extended
  neighbor 150.1.1.1 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 send-community extended
  neighbor 150.1.2.2 route-reflector-client
 exit-address-family
!
```

> `send-community extended` is mandatory — route-targets are extended communities. Without them, VPNv6 import fails silently.

### 2.4 Second VRF endpoint — advertise R32's prefix in VPNv6

R1 redistributes/advertises R32's IPv6 loopback into VPNv6. With matching RT `65100:25`, any remote PE with VRF BLUE imports it. Confirm the advertisement:

```
! R1
show bgp vpnv6 unicast all                      ! 2001:db8:32::32/128 under RD 65100:25
show bgp vpnv6 unicast vrf BLUE                  ! local + any remote BLUE IPv6 prefixes
```

> Per the Blue 3-AS design, the remote endpoint (R25 in AS65200/65300) imports this via inter-AS VPNv6. Locally on R1, confirm the prefix carries a VPN label and next-hop `::FFFF:150.1.1.1`.

### Section 2 Verification

```
! R1
show vrf detail BLUE                             ! both IPv4 and IPv6 AFs, RD 65100:25, RT 65100:25
show bgp vpnv6 unicast vrf BLUE summary          ! R32 neighbor Up
show bgp vpnv6 unicast all                        ! R32 loopback under RD, VPN label assigned
show bgp vpnv6 unicast vrf BLUE 2001:db8:32::32/128
show mpls forwarding-table vrf BLUE               ! VPN label for the IPv6 FEC
! Data plane (inside VRF)
R32# ping ipv6 <remote-BLUE-CE-loopback> source 2001:db8:32::32
traceroute ipv6 — two-label stack (transport + VPN) across R3/R4
```

---

## Section 3 — End-to-End & Core Integrity

```
! Core still IPv4-only
R3# show ipv6 route            ! empty
R4# show ipv6 route            ! empty
R3# show mpls forwarding-table ! IPv4 FECs only — IPv6/VPNv6 label-switched transparently
! PE label bindings
R1# show mpls forwarding-table | include ::      ! 6PE IPv6 FECs with local labels
! Control plane
R1# show bgp ipv6 unicast neighbors 150.1.7.7    ! IPv6 AF, send-label negotiated
R1# show bgp vpnv6 unicast all summary
```

---

## Section 4 — Troubleshooting

### 4.1 IPv6 route not landing in VRF BLUE
- RT import/export mismatch between PEs → `show vrf detail BLUE` compare RTs.
- `send-community extended` missing on PE→RR → `show bgp vpnv6 unicast all` shows prefix at RR but not imported at PE.
- VPNv6 AF not activated for that neighbor on RR → `show bgp vpnv6 unicast summary`.
- RD mismatch — different RD is OK for RT import but check RT, not RD.

### 4.2 Label not allocated for 6PE prefix
- Missing `send-label` on the IPv6 AF neighbor → `show bgp ipv6 unicast <prefix>` shows no in/out label.
- CEF disabled → `show ipv6 cef <prefix>` has no labeled output path.
- Fix: add `send-label` on both PE and RR for that neighbor; packets then label-switch.

### 4.3 Next-hop unreachable / not best
- IPv4-mapped next-hop `::FFFF:x.x.x.x` must resolve to a **labeled** IPv4 LSP, not a plain IGP route.
- `show ipv6 cef 2001:db8:9::9/128` must show a labeled output path (transport label).
- If next-hop is link-local or non-mapped, apply `neighbor X next-hop-self` under the IPv6 AF on the advertising PE.

---

## Completion Checklist

- [ ] 6PE: R9 IPv6 loopback reachable from R2 side over IPv4 MPLS core
- [ ] 6VPE: VRF BLUE R32 IPv6 prefix in VPNv6 with VPN label
- [ ] R3/R4 confirmed IPv6-free (`show ipv6 route` empty)
- [ ] All PE↔RR sessions carry `ipv6` (send-label) + `vpnv6` (send-community extended)
- [ ] Next-hops verified `::FFFF:150.1.X.X`
- [ ] **GNS3 snapshot:** `wb12-6pe-6vpe-complete`
