# LAB 3 — CCIE SP v5.1 Full Workbook: SOLUTIONS

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+); RRs on IOS-XE (CSR1000v) where noted.
**Companion files:** `LAB3_CCIE_SP_Full.md` (tasks), `GNS3/workbooks/00_topology_reference.md` (base addressing — loopbacks, link subnets, NET-IDs, PE-CE links, IPv6 reused verbatim).

> Addressing used below is pulled directly from `00_topology_reference.md` (unchanged core/PE-CE/inter-AS subnets, loopbacks, IS-IS NET-IDs, IPv6 scheme) plus the new SR/SRv6/Flex-Algo addressing defined in Section 0 of the task workbook (SRGB 16000–23999, SRLB 15000–15999, prefix-SID = last loopback octet, anycast index 114/label 16114 on R14/R15, SRv6 `fc00:3:<R#>::/48` Gold and `fc00:1:<R#>::/48` Emerald, Flex-Algo 128, colors 100/200).

---

## 0. Topology Reference Recap

| AS | Name | IGP | Routers |
|----|------|-----|---------|
| 65100 | Emerald | IS-IS L2 | R1(PE) R2(PE) R3(P) R4(P) R5(ASBR) R6(ASBR) R7(RR) R8(RR+PCE) |
| 65200 | Garnet | OSPF Area 0 (base) / IS-IS where noted | R11(P) R12(ASBR) R13(P) R14(PE) R15(PE) R16(PE) R19(RR+PCE) |
| 65300 | Gold | IS-IS L2 | R20(ASBR/PE) R21(ASBR/PE) R22(PE) R23(PE) R30(RR+PCE) |

Loopback0 = `150.<AS-last-digit>.<R#>.<R#>/32`. RRs in this lab (R7, R8, R19, R30) are run as IOS-XRv for full SR/BGP-LU feature parity — where the lab environment substitutes a CSR1000v/IOS-XE RR, the equivalent IOS-XE syntax is given alongside each RR config block.

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS Flex-CLI config groups, wide metrics, single-topology, LDP sync, overload-bit, keychain stub

**Design:** Level-2-only, single-topology (no MT), wide metrics, interface config-group applied on the P-tier (R3/R4/R5/R6), LDP sync on all AS 65100 core links, overload-bit on RRs (R7/R8), keychain stub referenced (built out in 5.2).

#### R1 (PE) — IOS-XR

```
key chain ISIS-KEY
 key 1
  cryptographic-algorithm HMAC-MD5
  key-string password-encrypted-or-plaintext-TBD-in-5.2
!
router isis EMERALD
 is-type level-2-only
 net 49.0001.1500.0100.1001.00
 address-family ipv4 unicast
  metric-style wide
  mpls traffic-eng level-2
 !
 address-family ipv6 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/0
  circuit-type level-2-only
  point-to-point
  hello-password hmac-md5-keychain ISIS-KEY
  address-family ipv4 unicast
   metric 10
  !
  address-family ipv6 unicast
   metric 10
  !
 !
 interface GigabitEthernet0/0/0/1
  circuit-type level-2-only
  point-to-point
  hello-password hmac-md5-keychain ISIS-KEY
  address-family ipv4 unicast
   metric 10
  !
  address-family ipv6 unicast
   metric 10
  !
```

> R2, R7, R8 follow the identical pattern (NET per `00_topology_reference.md`, no config-group needed since they aren't in the P-tier bulk-apply set) — only R3/R4/R5/R6 use the explicit config-group below.

#### R3, R4, R5, R6 (P-tier) — IOS-XR interface **config-group**

```
router isis EMERALD
 is-type level-2-only
 net 49.0001.1500.0100.3003.00   ! (R3; substitute R4/R5/R6 NET per topology ref)
 address-family ipv4 unicast
  metric-style wide
 !
 address-family ipv6 unicast
  metric-style wide
 !
 ! --- Flex-CLI interface configuration group ---
 interface-group CORE-GROUP
  circuit-type level-2-only
  point-to-point
  hello-password hmac-md5-keychain ISIS-KEY
  address-family ipv4 unicast
   metric 10
   mpls ldp sync
  !
  address-family ipv6 unicast
   metric 10
  !
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  address-family ipv6 unicast
 !
 interface GigabitEthernet0/0/0/0
  use interface-group CORE-GROUP
 !
 interface GigabitEthernet0/0/0/1
  use interface-group CORE-GROUP
 !
 interface GigabitEthernet0/0/0/2
  use interface-group CORE-GROUP
```

> `use interface-group` is the IOS-XR Flex-CLI mechanism (7.x+) — the group's `address-family`/`circuit-type`/`hello-password` stanzas are applied verbatim to every interface that references it, satisfying requirement 4 without repeating the stanza on R3 g0/0/0/0 (→R4), g0/0/0/1 (→R5 or R7), g0/0/0/2 (→ other core peer) etc. Apply the matching `use interface-group CORE-GROUP` on each of R3's/R4's/R5's/R6's actual core-facing interfaces per the link map (R3↔R4, R3↔R5, R3↔R7, R4↔R6, R5↔R8, R5↔R6).

#### mpls ldp sync (global, all AS 65100 core routers R1–R8)

```
mpls ldp
 interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/1
```

> `mpls ldp sync` is enabled per-interface **under the IS-IS address-family** (as shown in CORE-GROUP above) — this is the IOS-XR mechanism; LDP itself must still be configured globally with `mpls ldp` + `interface` stanzas matching the IS-IS-enabled links, or sync has nothing to hold for.

#### R7, R8 (RRs) — overload-bit on-startup

```
router isis EMERALD
 is-type level-2-only
 net 49.0001.1500.0100.7007.00   ! R7; R8 = ...8008.00
 set-overload-bit on-startup wait-for-bgp
 address-family ipv4 unicast
  metric-style wide
 !
 address-family ipv6 unicast
  metric-style wide
```

**Explanation:** `set-overload-bit on-startup wait-for-bgp` keeps the LSP's overload bit set after a reload until BGP signals convergence (all configured neighbors have exchanged their initial routing updates), so other IS-IS routers continue to route transit traffic around R7/R8 rather than blackholing it into a RR whose BGP table isn't populated yet.

**Verify:**
```
show isis neighbors
show isis database verbose
show mpls ldp neighbor
show isis adjacency detail        ! confirm Authentication: HMAC-MD5, LDP-IGP Sync: Achieved
show isis interface GigabitEthernet0/0/0/0 detail   ! confirm config-group inheritance on R3-R6
```
Expected: all L2 adjacencies `Up`, LDP-IGP sync state `Achieved` once LDP session + label exchange completes (link metric briefly held at `0xFFFFFFFF`/max until then), R7/R8 LSPs show `ATT/P/OL` overload bit set immediately after reload and cleared only after BGP converges.

---

### Task 1.2 — BGP Unified MPLS: BGP-LU, inline RRs (R3/R4, `allocate-label all`)

**Design:** `address-family ipv4 labeled-unicast` on every ASBR/RR; R3/R4 run `allocate-label all` making them inline label-switched RRs for *all* IGP prefixes, not just loopbacks.

#### R3, R4 (inline P-tier / inline RR for BGP-LU) — IOS-XR

```
route-policy ALLOW-ALL
  pass
end-policy
!
router bgp 65100
 bgp router-id 150.1.3.3
 address-family ipv4 unicast
  allocate-label all
  network 150.1.3.3/32
 !
 neighbor 150.1.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
   next-hop-self
  !
 !
 neighbor 150.1.2.2
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
   next-hop-self
  !
 !
 neighbor 150.1.5.5
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
  !
 !
 neighbor 150.1.6.6
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
```

> `allocate-label all` (under `address-family ipv4 unicast` in the BGP process) tells R3/R4 to allocate an MPLS label and advertise a BGP-LU route for **every** prefix in the local RIB/IGP table — not merely ones redistributed into BGP — which is what lets R3/R4 act as inline RRs for the label-switched path without a dedicated loopback-only RR hop.

#### R7, R8 (RR, BGP-LU session reflectors) — IOS-XR

```
router bgp 65100
 bgp router-id 150.1.7.7
 address-family ipv4 labeled-unicast
 !
 neighbor 150.1.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.1.2.2
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.1.5.5
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.1.6.6
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
```

#### R5, R6 (ASBR, Emerald side) — IOS-XR

```
router bgp 65100
 bgp router-id 150.1.5.5
 address-family ipv4 unicast
  redistribute isis EMERALD
 !
 address-family ipv4 labeled-unicast
 !
 neighbor 150.1.7.7
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
 !
 neighbor 150.1.8.8
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
 !
 ! eBGP BGP-LU toward Garnet ASBR (R5-R11 inter-AS link, directly connected)
 neighbor 10.5.11.11
  remote-as 65200
  address-family ipv4 labeled-unicast
   route-policy ALLOW-ALL in
   route-policy ALLOW-ALL out
```

> R6 mirrors R5 but peers `150.1.7.7`/`150.1.8.8` internally and `10.6.12.12` (→R12) + `10.6.20.20` (→R20) externally in `ipv4 labeled-unicast`, since R6 is the Emerald ASBR facing **both** Garnet (R12) and Gold (R20) per the inter-AS link table.

#### R12 (Garnet ASBR) — IOS-XR

```
router bgp 65200
 bgp router-id 150.2.12.12
 address-family ipv4 unicast
  redistribute ospf 1
 !
 address-family ipv4 labeled-unicast
 !
 neighbor 150.2.19.19
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
 !
 neighbor 10.6.12.6
  remote-as 65100
  address-family ipv4 labeled-unicast
 !
 neighbor 20.12.21.21
  remote-as 65300
  address-family ipv4 labeled-unicast
```

#### R19 (Garnet RR) and R30 (Gold RR) — IOS-XR

```
! R19
router bgp 65200
 bgp router-id 150.2.19.19
 address-family ipv4 labeled-unicast
 !
 neighbor 150.2.12.12
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.2.14.14
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.2.15.15
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.2.16.16
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
```

```
! R30
router bgp 65300
 bgp router-id 150.3.30.30
 address-family ipv4 labeled-unicast
 !
 neighbor 150.3.20.20
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.3.21.21
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.3.22.22
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.3.23.23
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
```

#### R20, R21 (Gold ASBR) — IOS-XR

```
! R20
router bgp 65300
 bgp router-id 150.3.20.20
 address-family ipv4 unicast
  redistribute isis GOLD
 !
 address-family ipv4 labeled-unicast
 !
 neighbor 150.3.30.30
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
 !
 neighbor 10.6.20.6
  remote-as 65100
  address-family ipv4 labeled-unicast
```

```
! R21
router bgp 65300
 bgp router-id 150.3.21.21
 address-family ipv4 unicast
  redistribute isis GOLD
 !
 address-family ipv4 labeled-unicast
 !
 neighbor 150.3.30.30
  remote-as 65300
  update-source Loopback0
  address-family ipv4 labeled-unicast
 !
 neighbor 20.12.21.12
  remote-as 65200
  address-family ipv4 labeled-unicast
```

**End-to-end path (R14 → R22):** R14 (PE, Garnet) → R19 (RR, reflect) → R12 (ASBR) → **eBGP-LU** → R6 (Emerald ASBR, dual-homed to both Garnet and Gold) → **eBGP-LU** → R20 (Gold ASBR) → R30 (RR, reflect) → R22. The direct R12↔R21 inter-AS link (`20.12.21.0/24`) exists physically and can be used as an alternate/ECMP BGP-LU eBGP path Garnet↔Gold without transiting Emerald — confirm with `show bgp ipv4 labeled-unicast` which path is actually preferred (shortest AS-path / local-pref will determine R12→R21 direct vs. R12→R6→R20).

**Verify:**
```
show bgp ipv4 labeled-unicast summary
show bgp labeled-unicast ipv4 unicast
show mpls label range
show mpls forwarding | include 150.3.22.22
ping mpls ipv4 150.3.22.22/32 source 150.2.14.14   ! from R14 — end-to-end LSP across 3 ASes
```

---

### Task 1.3 — SR-on-IS-IS: prefix-SID, SRGB/SRLB, anycast SID, TI-LFA, microloop avoidance

#### R1 (PE, Emerald) — IOS-XR

```
router isis EMERALD
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls sr-prefer
  segment-routing prefix-sid-map advertise-local
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 1
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
!
segment-routing
 global-block 16000 23999
 local-block 15000 15999
 microloop-avoidance protected
```

> Repeat per node for R2 (index 2), R3 (index 3, label 16003), R4 (index 4), R5 (index 5), R6 (index 6), R7 (index 7), R8 (index 8) — same `global-block`/`local-block` values (common SRGB across the AS is mandatory for label consistency), per-node SRLB can stay identical (15000–15999) since SRLB is only locally significant.

#### R3 (P) — IOS-XR, showing TI-LFA + microloop-avoidance explicitly on a P node

```
segment-routing
 global-block 16000 23999
 local-block 15000 15999
 microloop-avoidance protected
!
router isis EMERALD
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 3
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
```

#### R14 (Gar-R1, PE) and R15 (Gar-R2, PE) — anycast SID (Garnet, SR-native from the start)

```
! R14
interface Loopback114
 ipv4 address 150.2.114.114 255.255.255.255
!
router isis GARNET
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls sr-prefer
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 14
 !
 interface Loopback114
  passive
  address-family ipv4 unicast
   prefix-sid index 114 n-flag-clear
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
!
segment-routing
 global-block 16000 23999
 local-block 15000 15999
 microloop-avoidance protected
```

```
! R15 — identical anycast loopback/index, own primary index
interface Loopback114
 ipv4 address 150.2.114.114 255.255.255.255
!
router isis GARNET
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 15
 !
 interface Loopback114
  passive
  address-family ipv4 unicast
   prefix-sid index 114 n-flag-clear
!
segment-routing
 global-block 16000 23999
 local-block 15000 15999
 microloop-avoidance protected
```

> `n-flag-clear` on the anycast prefix-SID tells IS-IS not to set the Node-SID flag (since the SID is shared by two physical nodes, not unique to one) — this is required for correct anycast SID behavior/ECMP; both R14 and R15 advertise prefix-SID index **114** → label **16114** for `150.2.114.114/32`, so either can be targeted by inter-AS VPN traffic needing to reach "either active PE" with automatic ECMP/failover at the SR layer.

**Removing LDP from Emerald (after SR proven):**
```
! Per interface, once show mpls forwarding confirms SR label-switching for all prefixes:
interface GigabitEthernet0/0/0/0
 no mpls ldp
```

**Verify:**
```
show segment-routing mpls prefix-sid-map
show isis segment-routing
show mpls traffic-eng topology            ! TI-LFA repair paths
show isis fast-reroute ti-lfa
show route 150.2.114.114/32               ! R14/R15 anycast — confirm ECMP if applicable upstream
show mpls forwarding labels 16114
```

---

### Task 1.4 — SR-TE on Gar-R1 (R14) toward Gar-R2 (R15): dynamic, explicit+binding-SID, autoroute, BGP color autosteer, affinity

#### R14 — IOS-XR, all 6 policies

```
! Affinity-color definition (shared across the AS)
segment-routing
 traffic-eng
  affinity-map
   name RED bit-position 0
  !
  ! Policy 1 — dynamic, color 200, TE-metric optimized
  policy DYN-COLOR200
   color 200 end-point ipv4 150.2.15.15
   candidate-paths
    preference 100
     dynamic
      metric
       type te
      !
     !
    !
   !
   autoroute
    include ipv4 all
  !
  ! Policy 2 — explicit, color 201, forced path via Gar-R4(R13)->Gar-R5(R16)
  policy EXP-COLOR201
   color 201 end-point ipv4 150.2.15.15
   candidate-paths
    preference 100
     explicit segment-list EXPL-R13-R16
    !
   !
   binding-sid mpls 15500
  !
  segment-list EXPL-R13-R16
   index 1 mpls label 16013       ! prefix-SID R13 (Gar-R4)
   index 2 mpls label 16016       ! prefix-SID R16 (Gar-R5)
   index 3 mpls label 16015       ! prefix-SID R15 (Gar-R2, destination)
  !
  ! Policy 5 — exclude-any RED
  policy EXCL-RED
   color 202 end-point ipv4 150.2.15.15
   candidate-paths
    preference 100
     dynamic
      metric
       type te
      !
      constraints
       affinity
        exclude-any
         name RED
  !
  ! Policy 6 — include-all RED
  policy INCL-RED
   color 203 end-point ipv4 150.2.15.15
   candidate-paths
    preference 100
     dynamic
      metric
       type te
      !
      constraints
       affinity
        include-all
         name RED
```

#### Affinity tagging on the Gar-R3↔Gar-R4 link (R11↔R13) — IOS-XR

```
! R11 (Gar-R3)
interface GigabitEthernet0/0/0/0    ! toward R13
 affinity
  attribute-names RED
```
```
! R13 (Gar-R4)
interface GigabitEthernet0/0/0/0    ! toward R11
 affinity
  attribute-names RED
```

#### Autosteer: BGP color extended-community on R15's advertised VPNv4 routes (ties to Task 2.2's VRF)

```
! R15 — route-policy applied outbound to VPNv4 neighbors, setting color 200
extcommunity-set opaque COLOR-200
 200
end-set
!
route-policy SET-COLOR-200
  set extcommunity color COLOR-200
end-policy
!
router bgp 65200
 address-family vpnv4 unicast
 !
 neighbor 150.2.19.19
  address-family vpnv4 unicast
   route-policy SET-COLOR-200 out
```

> With the color-200 extended community present on the VPNv4 route and a color-200 SR-TE policy (`DYN-COLOR200` above) already instantiated on R14 toward the same endpoint (150.2.15.15), R14's BGP next-hop resolution automatically binds matching VPNv4 prefixes to the color-200 policy's tunnel — no static `autoroute` or PBR required. This is distinct from policy 1's own `autoroute include ipv4 all` stanza, which steers **IGP-shortest-path** traffic (not BGP/VPN traffic) into the tunnel.

**Verify:**
```
show segment-routing traffic-eng policy color 200
show segment-routing traffic-eng policy name EXP-COLOR201 detail   ! binding-SID 15500, segment-list
show segment-routing traffic-eng policy color 202 detail            ! exclude-any RED path
show segment-routing traffic-eng policy color 203 detail            ! include-all RED path, confirm differs from 202
show bgp vpnv4 unicast 192.168.7.0/24 detail                        ! color extcommunity + resolved tunnel
```

---

### Task 1.5 — SRv6 on Gold (R20–R23, R30): F3216 uSID, PSP/USD, tagged locator, SRv6 L3VPN

#### R30 (RR+PCE) — IOS-XR locator plan

```
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:30::/48
    behavior usid
```

#### R20, R21, R22, R23 — per-node locator + IS-IS advertisement with tag 3000

```
! R20
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:20::/48
    behavior usid
!
router isis GOLD
 address-family ipv4 unicast
  segment-routing srv6
   locator GOLD-LOC
    tag 3000
```

```
! R21
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:21::/48
    behavior usid
!
router isis GOLD
 address-family ipv4 unicast
  segment-routing srv6
   locator GOLD-LOC
    tag 3000
```

```
! R22
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:22::/48
    behavior usid
!
router isis GOLD
 address-family ipv4 unicast
  segment-routing srv6
   locator GOLD-LOC
    tag 3000
```

```
! R23 — identical pattern, prefix fc00:3:23::/48
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:23::/48
    behavior usid
!
router isis GOLD
 address-family ipv4 unicast
  segment-routing srv6
   locator GOLD-LOC
    tag 3000
```

#### PSP/USD behaviors (End/End.X SIDs) — R22, R23 (penultimate/ultimate hops for most Gold paths)

```
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:22::/48
    behavior usid
  !
  static
   locator GOLD-LOC
    sid <offset> behavior end-x psp-and-usd interface GigabitEthernet0/0/0/0
```

> In F3216 uSID deployments, PSP/USD is typically default-enabled per the uSID behavior spec (the penultimate micro-SID in the SRH is popped automatically, and the last active SID triggers decapsulation) — the explicit `psp-and-usd` behavior keyword above is shown for the **End.X** adjacency-SID case where per-interface override is required; confirm actual default behavior with `show segment-routing srv6 sid` which reports `PSP: Yes` / `USD: Yes` per allocated SID even without this explicit stanza on End-node SIDs.

#### SRv6 L3VPN on R22, R23 — VRF with End.DT4

```
! R22
vrf GOLD-L3VPN
 address-family ipv4 unicast
  import route-target
   65300:400
  !
  export route-target
   65300:400
!
segment-routing
 srv6
  locators
   locator GOLD-LOC
    micro-sid-format f3216
    prefix fc00:3:22::/48
    behavior usid
!
router bgp 65300
 vrf GOLD-L3VPN
  rd 65300:400
  address-family ipv4 unicast
   label mode per-vrf
   alloc mode per-vrf
   encapsulation-type srv6
   segment-routing srv6
    locator GOLD-LOC
!
interface GigabitEthernet0/0/0/1.400
 vrf GOLD-L3VPN
 ipv4 address 192.168.11.22 255.255.255.0
 encapsulation dot1q 400
```

```
! R23 — mirrors R22, same VRF/RT, local SRv6 locator fc00:3:23::/48
vrf GOLD-L3VPN
 address-family ipv4 unicast
  import route-target
   65300:400
  !
  export route-target
   65300:400
!
router bgp 65300
 vrf GOLD-L3VPN
  rd 65300:401
  address-family ipv4 unicast
   label mode per-vrf
   alloc mode per-vrf
   encapsulation-type srv6
   segment-routing srv6
    locator GOLD-LOC
!
interface GigabitEthernet0/0/0/2.500
 vrf GOLD-L3VPN
 ipv4 address 192.168.12.23 255.255.255.0
 encapsulation dot1q 500
```

**Verify:**
```
show segment-routing srv6 sid
show segment-routing srv6 locator detail
show isis database verbose | include SRv6|Tag
show bgp vpnv4 unicast vrf GOLD-L3VPN 192.168.11.0/24 detail   ! confirm End.DT4 SID, not MPLS label
show cef vrf GOLD-L3VPN 192.168.11.0/24
```
Expected: BGP VPNv4 route's tunnel encap shows `SRv6`, with an End.DT4 SID (not a VPN label); CEF entry shows an IPv6 SRH push, not an MPLS label imposition.

---

### Task 1.6 — SR-ODN: Gar-R6 (R19) PCE, Gar-R1 (R14) ODN color 100, Gar-R2 (R15) color-100 tagging

#### R19 (SR-PCE) — IOS-XR

```
pce
 address ipv4 150.2.19.19
!
router isis GARNET
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
```

#### R14 (PCC + ODN template, color 100)

```
segment-routing
 traffic-eng
  pcc
   pce address ipv4 150.2.19.19
   source-address ipv4 150.2.14.14
  !
  on-demand color 100
   dynamic
    pcep
    metric
     type latency
  !
  on-demand color 100
   sla mpls-disjoint-group disable      ! (no SLA/disjoint grouping needed for base ODN)
```

```
! Correct, consolidated ODN stanza (single color 100 template)
segment-routing
 traffic-eng
  pcc
   pce address ipv4 150.2.19.19
   source-address ipv4 150.2.14.14
  !
  on-demand color 100
   dynamic
    pcep
    metric
     type latency
```

> `pcep` under the `dynamic` candidate-path forces path computation delegation to the PCE (R19) rather than local headend SPF — combined with `metric type latency`, R19 computes the min-delay path and signals it back via PCEP, and the resulting policy's PCC/delegation state in `show pce lsp` will reflect `Delegated: Yes` only when PCEP, not local CSPF, is actually in use.

#### R15 (tag outbound VPNv4 routes color 100 for the ODN-targeted customer)

```
extcommunity-set opaque COLOR-100
 100
end-set
!
route-policy SET-COLOR-100
  set extcommunity color COLOR-100
end-policy
!
router bgp 65200
 vrf GREEN
  address-family ipv4 unicast
   neighbor 192.168.7.18
    remote-as 65910
!
router bgp 65200
 address-family vpnv4 unicast
 !
 neighbor 150.2.19.19
  address-family vpnv4 unicast
   route-policy SET-COLOR-100 out
```

**Verify:**
```
show pce peer
show pce lsp
show segment-routing traffic-eng policy color 100 detail   ! confirm "PCC info:" + Delegated: Yes
show bgp vpnv4 unicast vrf GREEN 192.168.7.0/24 detail      ! color 100 + auto-instantiated tunnel endpoint
```

---

### Task 1.7 — Flex-Algo 128 on Garnet: definition, per-algo prefix-SID, constrained SR-TE policy

#### R19 (Gar-R6) — Flex-Algo 128 definition

```
segment-routing
 flex-algo 128
  metric-type delay
  affinity
   exclude-any
    name RED
  advertise-definition
```

#### R11, R12, R13, R14, R15, R16, R19 — per-node Algo-128 prefix-SID + Flex-Algo participation

```
! Example: R14 (Gar-R1), index14/16014 default; 1014/separate-SRGB-relative-label under algo 128
router isis GARNET
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
  flex-algo 128
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 14
   prefix-sid algorithm 128 index 1014
```

> Repeat identically on R11 (index 11 / algo128 index 1011), R12 (12/1012), R13 (13/1013), R15 (15/1015), R16 (16/1016), R19 (19/1019) — the `flex-algo 128` keyword under the IS-IS IPv4 unicast address-family turns on Algo 128 SPF participation on that node, and `prefix-sid algorithm 128 index <n>` advertises the second, Algo-128-specific prefix-SID alongside the default Algo-0 one on Loopback0.

#### R14 — SR-TE policy explicitly constrained to Flex-Algo 128

```
segment-routing
 traffic-eng
  policy ALGO128-TO-R15
   color 128 end-point ipv4 150.2.15.15
   candidate-paths
    preference 100
     dynamic
      metric
       type delay
      !
      constraints
       algorithm 128
```

**Verify:**
```
show isis segment-routing flex-algo
show segment-routing flex-algo
show segment-routing traffic-eng policy name ALGO128-TO-R15 detail
```
Expected: policy's SID-list resolves to the Algo-128 prefix-SIDs (e.g., label corresponding to index 1013/1016 if the low-latency, RED-excluded path differs from the Algo-0 shortest path), confirming the policy did not simply fall back to the default topology.

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — EVPN-VPWS: T-LDP VPWS (R16↔R1, EVI 1001) + EVPN-VPWS over SRv6 (R14↔R15, EVI 1002)

#### R16 (Gar-R5, Garnet PE) — T-LDP VPWS, "Red" circuit

```
l2vpn
 pw-class RED-PW-CLASS
  encapsulation mpls
   transport-mode ethernet
  !
 !
 xconnect group RED
  p2p RED-PW
   interface GigabitEthernet0/0/0/3.100
   neighbor ipv4 150.1.1.1 pw-id 1001
    pw-class RED-PW-CLASS
```

#### R1 (Emerald PE) — far end of the same T-LDP VPWS

```
l2vpn
 pw-class RED-PW-CLASS
  encapsulation mpls
   transport-mode ethernet
  !
 !
 xconnect group RED
  p2p RED-PW
   interface GigabitEthernet0/0/0/2.100
   neighbor ipv4 150.2.16.16 pw-id 1001
    pw-class RED-PW-CLASS
```

> Targeted LDP session: `mpls ldp neighbor 150.2.16.16 targeted` (and the mirror on R16 targeting `150.1.1.1`) must be established across the inter-AS boundary for the PW's label binding to resolve — this requires either an extended/multihop LDP session or `mpls ldp discovery targeted-hello accept` on the ASBRs (R5/R6/R11/R12) along the path, since LDP targeted sessions are not automatically multi-hop without discovery acceptance.

```
! On transit ASBRs R6 and R12 (and R5/R11 if that's the active inter-AS path) to permit the targeted session:
mpls ldp
 discovery targeted-hello accept
```

#### Gar-R1 (R14) and Gar-R2 (R15) — EVPN-VPWS over SRv6, EVI 1002

```
! R14
l2vpn
 xconnect group EVPN-GROUP
  p2p EVPN-VPWS-1002
   interface GigabitEthernet0/0/0/3.200
   neighbor evpn evi 1002 target 1 source 2
!
evpn
 evi 1002
  advertise-mac
!
router bgp 65200
 address-family l2vpn evpn
 !
 neighbor 150.2.19.19
  address-family l2vpn evpn
```

```
! R15 — mirror AC-ID (target/source reversed), same EVI
l2vpn
 xconnect group EVPN-GROUP
  p2p EVPN-VPWS-1002
   interface GigabitEthernet0/0/0/1.200
   neighbor evpn evi 1002 target 2 source 1
!
evpn
 evi 1002
  advertise-mac
!
router bgp 65200
 address-family l2vpn evpn
 !
 neighbor 150.2.19.19
  address-family l2vpn evpn
```

> `encapsulation-type srv6` for this EVPN-VPWS service is inherited from the SRv6 transport already enabled on Garnet's underlying BGP sessions carrying `l2vpn evpn` — unlike MPLS-encapsulated EVPN-VPWS, no per-service PW label negotiation occurs; the EVI/RT pairing in the EVPN NLRI is the entire control plane, and the SRv6 locator/SID (from a Garnet SRv6 deployment analogous to Task 1.5's Gold build, applied here as `fc00:1:14::/48` / `fc00:1:15::/48` per the Section-0 Emerald/overlay addressing table) provides the data-plane path.

**Verify:**
```
show l2vpn xconnect group RED
show l2vpn xconnect summary
show evpn evi 1002 detail
```
Expected: `RED-PW` state `UP`, EVPN-VPWS EVI 1002 state `UP` with correct remote AC target/source IDs matched (R14's `target 1 source 2` pairs with R15's `target 2 source 1`).

---

### Task 2.2 — L3VPN: Intra-AS Green, Inter-AS Option B Blue, PE-CE OSPF/BGP, as-override

#### Intra-AS Green (Garnet): R17 via OSPF PE-CE on R15, R18 via eBGP AS 65910 on R16

```
! R15 — VRF GREEN, OSPF PE-CE to R17
vrf GREEN
 address-family ipv4 unicast
  import route-target
   65200:910
  export route-target
   65200:910
!
router ospf 10 vrf GREEN
 area 0
  interface GigabitEthernet0/0/0/0
!
interface GigabitEthernet0/0/0/0
 vrf GREEN
 ipv4 address 192.168.6.15 255.255.255.0
!
router bgp 65200
 vrf GREEN
  rd 65200:910
  address-family ipv4 unicast
   redistribute ospf 10
```

```
! R16 — VRF GREEN, eBGP PE-CE to R18 (AS 65910)
vrf GREEN
 address-family ipv4 unicast
  import route-target
   65200:910
  export route-target
   65200:910
!
interface GigabitEthernet0/0/0/0
 vrf GREEN
 ipv4 address 192.168.7.16 255.255.255.0
!
router bgp 65200
 vrf GREEN
  rd 65200:911
  address-family ipv4 unicast
  !
  neighbor 192.168.7.18
   remote-as 65910
   address-family ipv4 unicast
```

#### Inter-AS Option B — Blue customer, VPNv4 eBGP directly between R6 (Emerald ASBR) and R12 (Garnet ASBR)

```
! R6
router bgp 65100
 address-family vpnv4 unicast
 !
 neighbor 10.6.12.12
  remote-as 65200
  ebgp-multihop 1
  address-family vpnv4 unicast
   next-hop-self
   route-policy ALLOW-ALL in
   route-policy ALLOW-ALL out
```

```
! R12
router bgp 65200
 address-family vpnv4 unicast
 !
 neighbor 10.6.12.6
  remote-as 65100
  ebgp-multihop 1
  address-family vpnv4 unicast
   next-hop-self
   route-policy ALLOW-ALL in
   route-policy ALLOW-ALL out
```

> `next-hop-self` under the VPNv4 AF of the direct eBGP session is what makes this Option B, not Option A/C: each ASBR re-originates the VPNv4 NLRI with itself as the new next-hop and allocates a **fresh VPN label**, so the far side never resolves the original PE's label — confirm this at `show mpls forwarding` on R6/R12, where the inbound VPN label from Garnet is swapped for a locally-allocated one before being advertised toward Emerald's RRs (R7/R8).

#### as-override — R16 (hub, Green customer, R18 and other spokes share AS 65910)

```
router bgp 65200
 vrf GREEN
  address-family ipv4 unicast
  !
  neighbor 192.168.7.18
   remote-as 65910
   address-family ipv4 unicast
    as-override
```

> `as-override` on R16's PE-CE eBGP session toward R18 rewrites AS 65910 (appearing in the AS-path from another spoke CE routed through the same hub-and-spoke VRF) with the PE's own AS before re-advertising to R18, so R18 does not see its own ASN in the path and silently drop the route under standard AS_PATH loop prevention.

#### OSPF sham-link note (R15↔R17)

```
! R15 — sham-link to a remote PE in the same VRF/area if backdoor-link ambiguity exists (not needed here since R17 is single-homed into R15 only — down-bit/domain-tag handling is automatic via standard VPN-aware OSPF, no sham-link required unless a backdoor L2 path exists between sites)
router ospf 10 vrf GREEN
 area 0
  ! domain-tag and down-bit are automatically set by IOS-XR's VPN-aware OSPF (type-3/type-5 LSAs translated to/from BGP VPNv4 carry the DN-bit by default) — no additional config required unless a true backdoor link exists between R17's site and another Green site, in which case:
  ! area 0 sham-link 150.2.15.15 150.2.XX.XX cost 1
```

**Verify:**
```
show bgp vpnv4 unicast vrf GREEN
show bgp vpnv4 unicast vrf BLUE neighbor 10.6.12.12 advertised-routes
show route vrf BLUE                  ! confirm label swap at R6/R12 for Option B
show cef vrf GREEN
show bgp vpnv4 unicast vrf GREEN 192.168.7.0/24   ! confirm as-override rewrote AS 65910 occurrence
```

---

### Task 2.3 — NG-MVPN: PIM-SSM core, BGP auto-discovery, mdt default/data (Garnet: R14, R15, R16)

#### Garnet core (R11–R19) — PIM-SSM, no RP

```
multicast-routing
 address-family ipv4
  interface all enable
  mdt source Loopback0
!
router pim
 address-family ipv4
  rpf topology route-policy ALLOW-ALL
  ssm range SSM-RANGE
!
ipv4 access-list SSM-RANGE
 10 permit ipv4 232.0.0.0 0.255.255.255 any
```

#### R14, R15, R16 — VRF GREEN NG-MVPN, BGP auto-discovery, mdt default + data

```
! R14
multicast-routing
 vrf GREEN
  address-family ipv4
   interface all enable
   mdt source Loopback0
   mdt default mldp ipv4 232.1.1.1       ! default MDT group for low-bandwidth/control mcast
   mdt data 10 group-range 232.1.2.0/24 threshold 500   ! data MDT, 500 kbps switchover threshold
!
router bgp 65200
 address-family ipv4 mdt
 !
 neighbor 150.2.19.19
  address-family ipv4 mdt
!
router pim
 vrf GREEN
  address-family ipv4
   rpf topology route-policy ALLOW-ALL
   mdt auto-discovery pim
```

```
! R15, R16 — identical VRF GREEN mdt/pim/bgp stanzas, same mdt default/data group ranges (shared across the VRF)
multicast-routing
 vrf GREEN
  address-family ipv4
   interface all enable
   mdt source Loopback0
   mdt default mldp ipv4 232.1.1.1
   mdt data 10 group-range 232.1.2.0/24 threshold 500
!
router bgp 65200
 address-family ipv4 mdt
 !
 neighbor 150.2.19.19
  address-family ipv4 mdt
!
router pim
 vrf GREEN
  address-family ipv4
   rpf topology route-policy ALLOW-ALL
   mdt auto-discovery pim
```

> `mdt auto-discovery pim` tells the PEs to learn MDT group membership for the default-MDT via BGP (`address-family ipv4 mdt` NLRI) rather than relying purely on native-multicast PIM joins across the P-tier — each PE advertises `(VRF-source, default-MDT-group)` as a BGP MDT-AFI route, and remote PEs join the default MDT tree based on that BGP-learned membership.

**Verify:**
```
show pim vrf GREEN group-map
show mvpn vrf GREEN pe
show mvpn vrf GREEN database
show bgp ipv4 mdt
```
Expected: `show mvpn vrf GREEN pe` lists R14/R15/R16 as discovered PEs via BGP MDT AFI; once a Green multicast source behind R18 exceeds 500 kbps, `show mvpn vrf GREEN database` shows the flow switch from the default MDT (232.1.1.1) to a dedicated data MDT drawn from 232.1.2.0/24.

---

### Task 2.4 — H-QoS on R14 (Gar-R1), Green customer-facing interface

```
class-map match-any VOICE
 match dscp ef
end-class-map
!
class-map match-any BUSINESS-CRITICAL
 match dscp af31
 match dscp cs3
end-class-map
!
class-map match-any BEST-EFFORT
 match dscp default
end-class-map
!
! --- Ingress remark: untrusted CE markings -> AF31 for business-critical ---
class-map match-any BC-INGRESS-UNTRUSTED
 match dscp cs3
end-class-map
!
policy-map REMARK-INGRESS
 class BC-INGRESS-UNTRUSTED
  set dscp af31
 class class-default
end-policy-map
!
! --- Child policy: per-class policing + WRED on best-effort ---
policy-map CHILD-GREEN
 class VOICE
  police rate percent 10
   conform-action transmit
   exceed-action drop
 !
 class BUSINESS-CRITICAL
  police rate percent 30
   conform-action transmit
   exceed-action set-dscp-transmit af32
   violate-action drop
 !
 class BEST-EFFORT
  police rate percent 50
   conform-action transmit
   exceed-action set-dscp-transmit af13
   violate-action drop
  random-detect dscp af13 min-threshold 20 max-threshold 40 mark-probability 10
  random-detect dscp default min-threshold 10 max-threshold 30 mark-probability 10
end-policy-map
!
! --- Parent policy: shaper at 100 Mbps ---
policy-map PARENT-GREEN
 class class-default
  shape average 100000000
  service-policy CHILD-GREEN
end-policy-map
!
interface GigabitEthernet0/0/0/0.100
 vrf GREEN
 service-policy input REMARK-INGRESS
 service-policy output PARENT-GREEN
```

> The parent `shape average 100000000` enforces the 100 Mbps port-level ceiling; `service-policy CHILD-GREEN` nested under `class class-default` of the parent implements the two-level H-QoS hierarchy — the child's per-class `police` CIR/PIR values are only ever evaluated against traffic already constrained to 100 Mbps by the parent shaper. DSCP-based WRED (`random-detect dscp <value> min-threshold <N> max-threshold <N> mark-probability <N>`) gives AF13 (policer-marked best-effort overflow) a tighter/earlier drop profile than default-marked traffic, satisfying the "DSCP-based WRED profiles, not a single min/max" requirement.

**Verify:**
```
show policy-map interface GigabitEthernet0/0/0/0.100
show policy-map interface GigabitEthernet0/0/0/0.100 output class BEST-EFFORT
show qos interface GigabitEthernet0/0/0/0.100
```
Expected: shaped rate reported ≈100 Mbps on the parent, per-class policed rates matching the 10/30/50% splits, non-zero WRED random-drop counters under the BEST-EFFORT class once congestion is induced, and confirmation the ingress `REMARK-INGRESS` policy flips `cs3`→`af31` before the egress hierarchy classifies it into `BUSINESS-CRITICAL`.

---

## Section 3 — Access Connectivity (25%)

### Task 3.1 — SoO for dual-homed CE R25 (Blue customer, dual-homed into Garnet R21 and Gold R23 per topology — note: workbook's "PE-A/PE-B" pairing is R21/R23, the two PEs with direct links to R25 per the PE-CE link table)

#### R21 (Gold ASBR/PE facing R25 via g2/0, `192.168.10.21/24`)

```
extcommunity-set opaque SOO-R25
 65200:2500
end-set
!
route-policy SET-SOO-IN
  set extcommunity soo SOO-R25
end-policy
!
route-policy FILTER-SOO-OUT
  if extcommunity soo matches-any SOO-R25 then
    drop
  endif
  pass
end-policy
!
router bgp 65300
 vrf BLUE
  address-family ipv4 unicast
  !
  neighbor 192.168.10.25
   remote-as 65025
   address-family ipv4 unicast
    route-policy SET-SOO-IN in
    route-policy FILTER-SOO-OUT out
```

#### R23 (Gold PE facing R25 via f3/0, `192.168.12.23/24`)

```
extcommunity-set opaque SOO-R25
 65200:2500
end-set
!
route-policy SET-SOO-IN
  set extcommunity soo SOO-R25
end-policy
!
route-policy FILTER-SOO-OUT
  if extcommunity soo matches-any SOO-R25 then
    drop
  endif
  pass
end-policy
!
router bgp 65300
 vrf BLUE
  address-family ipv4 unicast
  !
  neighbor 192.168.12.25
   remote-as 65025
   address-family ipv4 unicast
    route-policy SET-SOO-IN in
    route-policy FILTER-SOO-OUT out
```

> Both PEs tag routes learned from R25 inbound with SoO `65200:2500` (`SET-SOO-IN`), and both filter outbound (`FILTER-SOO-OUT`) any VPNv4 route already carrying that SoO value before re-advertising toward R25's other link — this is the loop break: a route R21 learns from R25 and reflects into the Blue VPNv4 backbone will reach R23, but R23's outbound policy toward R25 drops it since it already carries R25's own SoO, instead of looping it back to R25 via the second uplink.

**Simulate the loop (temporarily, to prove the problem):**
```
! Remove SET-SOO-IN / FILTER-SOO-OUT from both R21 and R23 neighbor stanzas, observe routes learned from R25 via R21 reappear toward R25 via R23 (and vice versa) in show bgp vpnv4 unicast vrf BLUE — re-apply immediately after confirming.
```

**Verify:**
```
show bgp vpnv4 unicast vrf BLUE 192.168.10.0/24 detail   ! confirm SoO 65200:2500 present
show route vrf BLUE                                       ! confirm no re-advertisement loop / duplicate paths back to R25
```

---

### Task 3.2 — Troubleshooting MCQs

**1. ESI/DF-election duplicate-frame MCQ**

**Answer: (b)** — DF election is not run because the ESI is type-0 (all-zero) instead of a proper type-1 (LACP-based) or type-3 (MAC-based) ESI. A non-zero, properly-formed ESI is the trigger for EVPN's Designated Forwarder election among the multi-homed PEs for a given EVI/VLAN; if both PEs configure the AC with ESI `00:00:00:00:00:00:00:00:00:00` (type-0, i.e., effectively "no ESI"), each PE independently believes it is the sole/DF-equivalent forwarder for that AC and both forward — producing duplicate frames to the CE. Mismatched ESI values between the two PEs (a) would instead typically prevent them from being recognized as co-multi-homing the same segment at all (two different Ethernet Segments, no DF election needed since there's no overlap) rather than causing duplication; EVI mismatch (c) or RT mismatch (d) would break EVPN route exchange entirely (no reachability, not duplication).

Confirm with: `show evpn ethernet-segment detail` (verify ESI type/value matches on both PEs) and `show evpn ethernet-segment esi <esi> detail` (confirm DF election state — look for `DF Election` role: `DF` vs `Non-DF` per PE; if neither PE shows a resolved DF/Non-DF role and both show as forwarding, the ESI is almost certainly type-0 or missing).

**2. MC-LAG vs. EVPN multi-homing MCQ**

- **(a) Independently-uplinked PE pair, no ICCP/ICL:** EVPN multi-homing (all-active or single-active) is the correct choice. MC-LAG requires a dedicated Inter-Chassis Control Protocol (ICCP) session plus an Inter-Chassis Link (ICL) between the two PE chassis to synchronize LACP state, MAC tables, and failover — without that dedicated link, MC-LAG cannot function. EVPN multi-homing needs no ICL/ICCP at all: DF election and split-horizon/aliasing are run purely over the existing BGP EVPN control plane between the PEs (which may not even be directly connected), making it the only viable option here.
- **(b) Two PE chassis with a dedicated ICL capable of running ICCP:** Either MC-LAG or EVPN multi-homing *could* technically work, but EVPN multi-homing is still generally preferred on IOS-XR going forward (no vendor-proprietary ICCP state-sync dependency, cleaner integration with the EVPN control plane already used for the VPWS/L2VPN service itself, and no risk of ICCP split-brain if the ICL fails) — MC-LAG remains viable here specifically because the ICL exists, but it's not the better architectural choice.
- **Lab's R14/R15 pairing (Garnet dual-homed CE):** This matches scenario (a) — R14 and R15 are two independently-deployed Garnet PEs with no dedicated ICCP/ICL link modeled in the topology (per `00_topology_reference.md`, each has its own independent uplinks into the Garnet core; there is no R14↔R15 direct inter-chassis link in the link map). Therefore **EVPN multi-homing, not MC-LAG, is the only topology-appropriate mechanism** — MC-LAG cannot be deployed at all without first building a dedicated ICL between R14 and R15, which doesn't exist in this design.

---

## Section 4 — High Availability (10%)

### Task 4.1 — SRv6 backup paths, `spf prefix-priority critical` on R22/R23 loopbacks (Gold)

#### R30 (RR, or any node computing SPF — apply the priority policy network-wide so all nodes agree on priority ordering)

```
route-policy CRITICAL-PREFIXES
  if destination in (150.3.22.22/32, 150.3.23.23/32) then
    pass
  endif
end-policy
!
router isis GOLD
 address-family ipv4 unicast
  spf prefix-priority critical route-policy CRITICAL-PREFIXES
```

> Apply the identical `router isis GOLD / address-family ipv4 unicast / spf prefix-priority critical route-policy CRITICAL-PREFIXES` stanza on **every** Gold node (R20, R21, R22, R23, R30) — `spf prefix-priority` is a local SPF-computation behavior, so all routers performing SPF (which, in IS-IS, is every participating router) need the same policy to consistently prioritize R22/R23's prefixes during their own TI-LFA backup-path computation.

**Verify:**
```
show isis fast-reroute ti-lfa
show isis spf-log detail        ! confirm critical-priority prefixes (150.3.22.22/32, 150.3.23.23/32) are computed in the first SPF batch
show segment-routing srv6 sid    ! confirm backup End SIDs are pre-installed for R22/R23's locators before lower-priority prefixes
```
Controlled test: fail the R20↔R22 link (or R21↔R23) and confirm (via timestamped `show isis spf-log` and traffic-loss measurement) that reachability to R22/R23's loopbacks is restored before reachability to lower-priority prefixes in the same failure domain converges.

---

### Task 4.2 — BFD for BGP between ASBRs (R6↔R12, R12↔R21)

#### R6 — IOS-XR

```
bfd
 multiplier 3
 interval 150
!
interface GigabitEthernet0/0/0/3
 bfd mode ietf
 bfd address-family ipv4 destination 10.6.12.12 multiplier 3 interval 150 fast-detect
!
router bgp 65100
 neighbor 10.6.12.12
  remote-as 65200
  bfd fast-detect
```

#### R12 — IOS-XR

```
bfd
 multiplier 3
 interval 150
!
interface GigabitEthernet0/0/0/0
 bfd mode ietf
 bfd address-family ipv4 destination 10.6.12.6 multiplier 3 interval 150 fast-detect
!
router bgp 65200
 neighbor 10.6.12.6
  remote-as 65100
  bfd fast-detect
 !
 neighbor 20.12.21.21
  remote-as 65300
  bfd fast-detect
!
interface GigabitEthernet0/0/0/1
 bfd mode ietf
 bfd address-family ipv4 destination 20.12.21.21 multiplier 3 interval 150 fast-detect
```

#### R21 — IOS-XR

```
bfd
 multiplier 3
 interval 150
!
interface GigabitEthernet0/0/0/2
 bfd mode ietf
 bfd address-family ipv4 destination 20.12.21.12 multiplier 3 interval 150 fast-detect
!
router bgp 65300
 neighbor 20.12.21.12
  remote-as 65200
  bfd fast-detect
```

**Verify:**
```
show bfd session
show bgp neighbor 10.6.12.12 | include BFD
show logging | include BFD|BGP   ! around a simulated interface shutdown — compare BGP teardown timestamp vs. hold-timer-only teardown
```
Expected: `show bfd session` shows state `UP` with 150ms × 3 = 450ms detection time, and `show bgp neighbor ... | include BFD` reports `BFD: Enabled, fast-detect`. Failing the link with BFD enabled tears down BGP within roughly 450ms–1s (interface-down + BFD detection latency); disabling BFD and repeating the test shows BGP waiting the full hold-timer (default 180s, or whatever configured) before teardown — a dramatic convergence-time difference to document in the comparison.

---

### Task 4.3 — OSPF NSF troubleshooting MCQ (R15 Garnet OSPF PE)

**Answer: (a)** — NSF is enabled on R15, but neighboring routers R11 and R19 do not support (or have not enabled) **OSPF graceful-restart helper mode**. Graceful Restart requires cooperation from the restarting router's neighbors: the restarting router (R15) sends a Grace-LSA announcing an upcoming planned restart and a grace-period duration, and neighbors must recognize that Grace-LSA and enter **helper mode** — holding R15's existing LSAs/adjacency state as-is for the duration of the grace period instead of immediately aging them out on the normal dead-interval. If R11/R19 don't support or haven't enabled helper mode, they simply ignore the Grace-LSA and treat R15's RP/process restart as an ordinary adjacency loss, flushing R15's LSAs and triggering a brief SPF-driven outage — exactly the symptom described. (b) is wrong because NSF/GR does not require BFD (they're independent features, though often paired). (c) is wrong because OSPF Graceful Restart (RFC 3623) is defined for OSPFv2 as well as OSPFv3 (RFC 5187) — not OSPFv3-only. (d) is a plausible-sounding but incorrect distractor: if the restart genuinely exceeded the advertised grace period, that actually *would* be expected/correct helper behavior (helpers are supposed to stop helping and age out LSAs after grace-period expiry) — but the symptom described (an outage despite GR being configured, with no indication of an abnormally long restart) is more consistent with helper mode never engaging at all, i.e., (a).

Confirm which `show` command identifies helper-mode state: `show ospf neighbor` plus specifically **`show ospf <process> graceful-restart`** (or `show ospf neighbor detail`) on R11/R19 — this reports whether a neighbor is currently acting as a **Graceful Restart Helper** for a given neighbor's restart event, and whether helper mode is even enabled locally.

**Resolution on R11/R19:**
```
router ospf 1
 nsf cisco
 ! or, for IETF-standard GR helper mode:
 nsf ietf helper disable        ! (shown for reference — the fix is the OPPOSITE: ensure helper is NOT disabled)
```
The actual fix is to ensure graceful-restart **helper** support is enabled (not disabled) on R11 and R19 — on IOS-XR this is typically on by default once `nsf ietf` or `nsf cisco` is configured under `router ospf`, so the remediation is auditing that R11/R19 have NSF/GR configured at all (many default/minimal OSPF configs omit it) rather than it being present-but-disabled:
```
router ospf 1
 nsf ietf
```

---

## Section 5 — Security (10%)

### Task 5.1 — Management Plane Protection, inband, SSH/SNMP allow + Telnet/XML/TFTP explicit deny (R1, R6, R12, R14, R20)

```
ipv4 access-list MGMT-ALLOWED
 10 permit ipv4 10.255.0.0 0.0.0.255 any
!
control-plane
 management-plane
  inband
   interface all
    allow SSH peer
     address ipv4 access-list MGMT-ALLOWED
    !
    allow SNMP peer
     address ipv4 access-list MGMT-ALLOWED
    !
    deny TELNET peer
    deny XML peer
    deny TFTP peer
```

> Apply the identical `control-plane / management-plane / inband` stanza on R1, R6, R12, R14, R20. The `allow <protocol> peer / address ipv4 access-list MGMT-ALLOWED` form restricts SSH and SNMP to only be reachable from `10.255.0.0/24`; the explicit `deny TELNET peer` / `deny XML peer` / `deny TFTP peer` lines (with no address-list, i.e., deny from any peer) ensure those three protocols are blocked outright regardless of any other ACL elsewhere in the config, satisfying the "do not just omit them" requirement. MPP's inband policy only ever evaluates **self-destined** management-protocol traffic terminating on the router's own control plane — transit/data-plane traffic through the box is entirely unaffected since it's never classified as "to-the-box" management traffic in the first place.

**Verify:**
```
show control-plane-protection management inband counters
show run control-plane
telnet <router-mgmt-ip>                 ! from outside 10.255.0.0/24 — expect failure/connection refused
ssh <router-mgmt-ip>                    ! from inside 10.255.0.0/24 — expect success
snmpwalk -v2c -c <community> <router-mgmt-ip>   ! from inside 10.255.0.0/24 — expect success
```

---

### Task 5.2 — IS-IS keychain: HMAC-MD5, infinite lifetime, hello + LSP password (R1–R8, Emerald)

#### Completing the keychain stub from Task 1.1 — R1 (repeat identically on R2–R8)

```
key chain ISIS-KEY
 key 1
  cryptographic-algorithm HMAC-MD5
  key-string password Em3r@ldC0r3Key!
  accept-lifetime 00:00:00 january 01 2024 infinite
  send-lifetime 00:00:00 january 01 2024 infinite
!
router isis EMERALD
 interface GigabitEthernet0/0/0/0
  hello-password hmac-md5-keychain ISIS-KEY
 !
 lsp-password hmac-md5-keychain ISIS-KEY
```

> `accept-lifetime`/`send-lifetime ... infinite` makes the key always valid with no start/end restriction, satisfying the "infinite lifetime" requirement explicitly rather than relying on a key's default (which, depending on XR train, may require an explicit lifetime stanza to avoid an implicit expiry). `hello-password hmac-md5-keychain ISIS-KEY` is applied per-interface (already present via the CORE-GROUP config-group on R3–R6 from Task 1.1 — R1/R2/R7/R8 need it added directly since they weren't part of that group); `lsp-password hmac-md5-keychain ISIS-KEY` is applied once under the IS-IS instance (not per-interface) since LSPs are flooded instance-wide, not link-specific.

**Induced-failure test (break then restore):**
```
! On R2 only, temporarily point to a non-existent/mismatched keychain:
key chain ISIS-KEY
 key 1
  key-string password WRONG-KEY-TEMPORARY
! Observe R2's adjacencies to R1/R4 drop (show isis adjacency) due to hello auth failure.
! Restore the correct key-string and confirm adjacency re-forms.
```

**Verify:**
```
show key chain ISIS-KEY
show isis interface GigabitEthernet0/0/0/0 detail   ! confirm HMAC-MD5 auth state: Enabled
show isis adjacency detail                           ! Authentication: HMAC-MD5, state Up
show isis database                                    ! confirm no corrupted/unauthenticated LSP accepted during the induced-failure test
```

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry troubleshooting (R14 → collector)

**Isolation sequence:**

1. **Check TLS match first** (most common root cause for a dial-out session that never establishes):
```
show run telemetry model-driven
telemetry model-driven
 destination-group DG-COLLECTOR
  address-family ipv4 10.255.1.50 port 57500
   encoding self-describing-gpb
   protocol grpc
    tls
```
If the collector is listening **without** TLS but the router config has `tls` under `protocol grpc` (or vice versa), the gRPC dial-out will fail silently at the TCP/TLS handshake stage — confirm with `show grpc status` (reports whether gRPC/TLS is actually up) and cross-check the collector's own listener config (TLS vs. plaintext) out-of-band.

2. **Check TPA source-address/VRF binding:**
```
telemetry model-driven
 destination-group DG-COLLECTOR
  address-family ipv4 10.255.1.50 port 57500
   source-address ipv4 10.255.0.14
```
Confirm `10.255.0.14` is an address actually configured on R14 (not a stale/unassigned loopback), that it's in the correct VRF (default/global unless the collector is reachable only via a management VRF, in which case the destination-group or global TPA config needs a matching `vrf` stanza), and that it's advertised/routed toward the collector's management network — `show tpa` reports the active TPA source bindings per protocol/VRF.

3. **Check port + MPP interaction:**
Confirm the collector's listening port (`57500` in this example) matches the router's `destination-group` port exactly, and that this port is explicitly allowed by the Task 5.1 MPP inband policy — if MPP's inband policy only allowed SSH/SNMP and explicitly denied Telnet/XML/TFTP but said nothing about the gRPC telemetry port, confirm whether MPP's default (implicit-deny for unlisted protocols under `inband`) is silently blocking the dial-out session on a per-protocol basis separate from the explicit denies in 5.1 — add an explicit `allow` for the telemetry port's protocol/peer if MPP is the actual culprit.

**Correct verification sequence to isolate the fault:**
```
show grpc status                                  ! Step 1 — TLS/gRPC transport state
show tpa                                           ! Step 2 — TPA source-address/VRF binding
show telemetry model-driven destination-group DG-COLLECTOR   ! Step 3 — port/encoding config as applied
show telemetry model-driven subscription SUB-NAME             ! overall subscription state — look for "State: Active" vs "State: Invalid"/"ConnectionError"
show control-plane-protection management inband counters      ! confirm MPP isn't silently dropping the gRPC port
```
Walk the four checks in order (gRPC/TLS → TPA/VRF → port/MPP → subscription state) since each is a progressively narrower filter on the same symptom ("no data arrives at the collector").

---

### Task 6.2 — NSO: onboard R14/R15, sync-from, template push, L3VPN service deploy

**NSO CLI (ncs_cli) — onboarding:**
```
configure
devices authgroups group IOSXR-CRED
 default-map remote-name admin remote-password <password>
!
devices device R14
 address 150.2.14.14
 port 22
 authgroup IOSXR-CRED
 device-type netconf ned-id cisco-iosxr-nc-7.4
 state admin-state unlocked
!
devices device R15
 address 150.2.15.15
 port 22
 authgroup IOSXR-CRED
 device-type netconf ned-id cisco-iosxr-nc-7.4
 state admin-state unlocked
!
commit
```

**sync-from both devices before any change:**
```
devices device R14 sync-from
devices device R15 sync-from
```

**Baseline device-template push (confirm template mechanics before attempting a service):**
```
configure
devices template LOOPBACK-DESC-TEMPLATE
 ned-id cisco-iosxr-nc-7.4
 config
  interface Loopback0
   description "NSO-managed: baseline template push test"
!
commit
exit
devices device R14 apply-template template-name LOOPBACK-DESC-TEMPLATE
devices device R15 apply-template template-name LOOPBACK-DESC-TEMPLATE
```

**L3VPN service instance (assumes an `l3vpn` service package already loaded via `packages reload`):**
```
configure
l3vpn CUSTOMER-GREEN-NSO
 rd 65200:999
 route-target 65200:999
 endpoint R14
  pe-interface GigabitEthernet0/0/0/0.999
  ce-protocol bgp
  ce-as 65910
 !
 endpoint R15
  pe-interface GigabitEthernet0/0/0/1.999
  ce-protocol bgp
  ce-as 65910
!
commit dry-run outformat native
commit
```

> `commit dry-run outformat native` surfaces the exact native IOS-XR CLI NSO is about to push to R14/R15 before committing — this is the "dry-run before commit" verification step the task calls for; review it for a correctly-rendered `vrf CUSTOMER-GREEN-NSO`, matching RD/RT, PE-CE sub-interface, and BGP AS stanza on both devices before the real `commit`.

**Verify:**
```
show devices device R14 sync-from
show devices device R14
show devices device R15
```
And on the routers themselves (confirming NSO's push actually landed and matches intent):
```
show running-config vrf CUSTOMER-GREEN-NSO     ! on R14
show running-config vrf CUSTOMER-GREEN-NSO     ! on R15
```

---

### Task 6.3 — NSO troubleshooting: R15 out-of-sync, re-deploy

**Step 1 — confirm and characterize the drift:**
```
devices device R15 check-sync
```
Expected output on an out-of-sync device: `result out-of-sync`. Follow with a compliance diff for detail:
```
devices device R15 compare-config
```
This reports the specific config lines present on R15's live running-config that differ from NSO's CDB (the manual CLI change made outside of NSO).

**Step 2 — remediation decision:**
- **`sync-from`** (pull R15's manual change into NSO's CDB, i.e., let the manual change win) is correct when the manual change was itself a legitimate operational fix (e.g., an emergency ACL tweak during an incident) that should be retroactively adopted as the new intended state — the tradeoff is that NSO's service model may now diverge from what it believes it deployed, and future `re-deploy` operations on the *same* service could silently overwrite the manually-adopted change again unless the service model itself is updated to reflect it.
- **`sync-to` / service re-deploy** (push NSO's CDB-intended config back onto R15, overwriting the manual change) is correct when the manual change was unauthorized, accidental, or violates intended service state (the more common production-SP posture: config drift via out-of-band CLI access is treated as a fault to be corrected, not a new baseline) — the tradeoff is that whatever operational benefit the manual change provided (if any) is lost, and if it was an emergency fix for a real problem, that problem returns until a proper, NSO-modeled fix is deployed.
- Given this scenario describes an *unplanned* manual change discovered via `check-sync` discrepancy (not a documented emergency change ticket), the production-SP-appropriate default is **re-deploy** (overwrite drift), which is what's executed below.

**Step 3 — re-deploy the service:**
```
request l3vpn CUSTOMER-GREEN-NSO re-deploy
```

**Verify:**
```
devices device R15 check-sync         ! expect result in-sync post-remediation
devices commit-queue queue-items      ! confirm no stuck/queued commit items from the re-deploy
```
And on R15 itself:
```
show running-config vrf CUSTOMER-GREEN-NSO   ! confirm it now matches NSO's intended state, manual drift corrected
```

---

## Appendix — Section Weighting Reference

| Section | Weight | Tasks |
|---|---|---|
| 1. Core Network Infrastructure | 25% | 1.1–1.7 |
| 2. Architecture and Services | 25% | 2.1–2.4 |
| 3. Access Connectivity | 25% | 3.1–3.2 |
| 4. High Availability | 10% | 4.1–4.3 |
| 5. Security | 10% | 5.1–5.2 |
| 6. Automation | 5% | 6.1–6.3 |

**Inter-section dependencies honored in these solutions:**
- Section 1.1's `key chain ISIS-KEY` stub (hello-password reference only) is completed with full `key 1`/HMAC-MD5/infinite-lifetime/`lsp-password` detail in Section 5.2 — do not consider Task 1.1 "done" from a security standpoint until 5.2's block is also applied.
- Section 1.3's SRGB/SRLB/prefix-SID plan underpins every label value referenced in Sections 1.4–1.7 and 2.1's EVPN-VPWS-over-SRv6 — Task 1.3 must be functional (verify `show isis segment-routing`) before testing anything in 1.4 onward.
- Section 1.4's RED affinity-color definition is reused verbatim by Section 1.7's Flex-Algo 128 `exclude-any affinity RED` constraint — define affinity once, consume it twice.
- Section 1.5's Gold SRv6 locator/F3216 build is the direct model for Section 2.1's EVPN-VPWS-over-SRv6 EVI 1002 (Garnet overlay, `fc00:1:<R#>::/48`) — the PSP/USD and per-VRF SID allocation mechanics carry over conceptually even though the AS differs.
- Section 1.6's color-100 ODN pattern and Section 1.4's color-200 BGP-autosteer pattern are the same underlying mechanism (BGP color extended-community driving automatic SR-TE tunnel selection) applied to two different trigger conditions (ODN auto-instantiation vs. steering onto an already-existing policy) — understand one to understand both.
- Section 2.4's H-QoS class-map DSCP matches (ef/af31/cs3/default) are the same marking scheme Section 1.6/1.4's color-based VPN traffic conceptually rides over, though QoS and SR-TE path selection remain independently configured (QoS classifies/shapes; SR-TE/color chooses the path) — the workbook's note that "1.4/1.6 color-steering is reused by 2.4's QoS classes conceptually" refers to this shared DSCP/marking vocabulary, not a direct feature dependency.
- Section 5.1's MPP inband policy is directly implicated in Section 6.1's telemetry troubleshooting flow — a telemetry dial-out failure's root cause can be an MPP policy gap, so 6.1's isolation sequence explicitly checks 5.1's policy as step 3.
