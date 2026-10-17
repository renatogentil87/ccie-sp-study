# LAB 4 — CCIE SP v5.1 Full Workbook: SOLUTIONS

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+); CEs on IOS-XE (CSR1000v / IOSv).
**Companion files:** `LAB4_CCIE_SP_Full.md` (tasks), `GNS3/workbooks/00_topology_reference.md` (base addressing reused verbatim).

> Addressing is pulled directly from `00_topology_reference.md` (core/PE-CE/inter-AS subnets, loopbacks, IS-IS NET-IDs, IPv6) plus the LAB4-specific SR plan from Section 0 of the task file:
> - **SRGB 16000–23999**, **SRLB 15000–15999** on all SR nodes.
> - **Prefix-SID index = last octet of Loopback0 × 10** (R14→140→16140, R20→200→16200, R22→220→16220).
> - **Anycast** R14/R15 index **1140**, label **17140**, anycast Lo `150.2.114.114/32`.
> - **SRv6 locator** `fc00:1:<R#>::/48` (Emerald), F3216 uSID.
> - **Flex-Algo 129** min-delay, `include-all BLUE`.
> - **Colors:** 300/301 explicit-SID TE, 400 ODN min-delay.
> - **EVI 2001** (R31↔R16 Red), **EVI 2002** (R14↔R15, AC-IDs 2142/2152).
> - **mVPN Profile 3** (BGP C-multicast, mLDP P2MP).

---

## 0. Topology Reference Recap

| AS | Name | IGP (LAB4) | Transport | Routers |
|----|------|------------|-----------|---------|
| 65100 | Emerald | IS-IS **multi-level** | **SRv6** | R31(PE) R2(PE) R3(P) R4(P) R5(ASBR) R6(ASBR) R7(RR) R8(RR+PCE) |
| 65200 | Garnet | OSPF Area 0 + **RSVP-TE** | SR-MPLS | R11(P) R12(ASBR) R13(P) R14(PE) R15(PE) R16(PE) R19(RR+PCE) |
| 65300 | Gold | IS-IS **multi-level** | **SR-MPLS** | R20(ASBR/PE) R21(ASBR/PE) R22(PE) R23(PE) R30(RR+PCE) |

**Level roles (Emerald/Gold IS-IS):** PEs/ASBRs = L1-only; P routers = L1/L2; RRs = L2-only.

Key links (from topology reference):

| Link | Subnet | Role |
|---|---|---|
| R5↔R11 | 10.5.11.0/24 | Inter-AS 65100↔65200 |
| R6↔R12 | 10.6.12.0/24 | Inter-AS 65100↔65200 |
| R6↔R20 | 10.6.20.0/24 | Inter-AS 65100↔65300 |
| R12↔R21 | 20.12.21.0/24 | Inter-AS 65200↔65300 |
| R14↔R25 | 192.168.5.0/24 | Blue PE-CE eBGP 65025 |
| R21↔R25 | 192.168.10.0/24 | Blue PE-CE eBGP 65025 |
| R23↔R25 | 192.168.12.0/24 | Blue PE-CE eBGP 65025 |
| R15↔R17 | 192.168.6.0/24 | Green PE-CE OSPF |
| R16↔R18 | 192.168.7.0/24 | Green PE-CE eBGP 65910 |

Prefix-SID label map (SRGB 16000, index ×10):

| Node | Lo0 | Index | Label | Node | Lo0 | Index | Label |
|---|---|---|---|---|---|---|---|
| R11 | 150.2.11.11 | 110 | 16110 | R20 | 150.3.20.20 | 200 | 16200 |
| R12 | 150.2.12.12 | 120 | 16120 | R21 | 150.3.21.21 | 210 | 16210 |
| R13 | 150.2.13.13 | 130 | 16130 | R22 | 150.3.22.22 | 220 | 16220 |
| R14 | 150.2.14.14 | 140 | 16140 | R23 | 150.3.23.23 | 230 | 16230 |
| R15 | 150.2.15.15 | 150 | 16150 | R30 | 150.3.30.30 | 300 | 16300 |
| R16 | 150.2.16.16 | 160 | 16160 | R14/R15 anycast | 150.2.114.114 | 1140 | 17140 |
| R19 | 150.2.19.19 | 190 | 16190 | | | | |

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS multi-level design (Emerald AS 65100)

**Design:** single instance `EMERALD`, area 49.0001, dual-stack single-topology, wide metrics at both levels. Level by role: R31/R2/R5/R6 = L1-only; R3/R4 = L1/L2; R7/R8 = L2-only. Overload-bit on RRs. L2→L1 route-leaking of remote-PE loopbacks so SR prefix-SIDs resolve in the L1 edge. Keychain `ISIS-KEY-L4` referenced (built in Task 5.1).

#### R31 (PE, L1-only) — IOS-XR

```
key chain ISIS-KEY-L4
 key 1
  accept-lifetime 00:00:00 january 01 2024 infinite
  send-lifetime   00:00:00 january 01 2024 infinite
  cryptographic-algorithm HMAC-MD5
  key-string <built-in-Task-5.1>
!
router isis EMERALD
 is-type level-1
 net 49.0001.1500.0100.1001.00
 lsp-password keychain ISIS-KEY-L4 level 1
 address-family ipv4 unicast
  metric-style wide
 !
 address-family ipv6 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 10
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  circuit-type level-1
  point-to-point
  hello-password keychain ISIS-KEY-L4
  address-family ipv4 unicast
   metric 10
  address-family ipv6 unicast
   metric 10
 !
```

> R2, R5, R6 identical pattern: `is-type level-1`, NET per topology ref, `circuit-type level-1` on core interfaces, prefix-SID index = last octet ×10 (R2→20, R5→50, R6→60).

#### R3 (P, L1/L2 — the attachment tier) — IOS-XR

```
router isis EMERALD
 is-type level-1-2
 net 49.0001.1500.0100.3003.00
 lsp-password keychain ISIS-KEY-L4 level 1
 lsp-password keychain ISIS-KEY-L4 level 2
 address-family ipv4 unicast
  metric-style wide level 1
  metric-style wide level 2
  ! --- L2 -> L1 leak of remote-PE loopbacks so SR labels resolve in L1 edge ---
  propagate level 2 into level 1 route-policy LEAK-L2-L1
 !
 address-family ipv6 unicast
  metric-style wide level 1
  metric-style wide level 2
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 30
 !
 interface GigabitEthernet0/0/0/0
  circuit-type level-1-2
  point-to-point
  hello-password keychain ISIS-KEY-L4
  address-family ipv4 unicast
   metric 10
 !
!
route-policy LEAK-L2-L1
  if destination in (150.1.0.0/16 ge 32) then
    pass
  endif
end-policy
```

> R4 identical (NET `...4004`, prefix-SID index 40).

#### R7 (RR, L2-only, overload on startup) — IOS-XR

```
router isis EMERALD
 is-type level-2-only
 net 49.0001.1500.0100.7007.00
 set-overload-bit on-startup wait-for-bgp
 lsp-password keychain ISIS-KEY-L4 level 2
 address-family ipv4 unicast
  metric-style wide
 !
 address-family ipv6 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 70
 !
 interface GigabitEthernet0/0/0/0
  circuit-type level-2-only
  point-to-point
  hello-password keychain ISIS-KEY-L4
  address-family ipv4 unicast
   metric 10
 !
```

> R8 identical (NET `...8008`, index 80, overload-bit on-startup wait-for-bgp).

**Why:** L1-only PEs install an `ATT`-bit default toward the nearest L1/L2 router (R3/R4). SR prefix-SIDs of remote PEs are *not* in the L1 database by default, so an SR LSP to a remote PE loopback cannot be built from the edge. Leaking the `150.1.0.0/16` host routes L2→L1 at R3/R4 puts the remote loopbacks (and their prefix-SIDs) into L1, so the label-switched path resolves end-to-end. `wait-for-bgp` keeps RRs from drawing transit until BGP is up.

**Verify:**
```
show isis neighbors                 ! L1 on R31/R2/R5/R6, L1L2 on R3/R4, L2 on R7/R8
show isis database level-1
show isis database level-2
show route 150.1.7.7/32             ! on R1: resolves via leaked L1 route (not just default)
show isis adjacency detail
```

### Task 1.2 — BGP Unified MPLS (BGP-LU / Seamless MPLS) across all three ASes

**Design:** `ipv4 labeled-unicast` on every ASBR (R5,R6,R12,R20,R21) and RR (R7,R8,R19,R30). ASBRs redistribute IGP loopbacks into BGP-LU with a label and set next-hop-self; BGP-LU is NOT redistributed back into any IGP. Emerald P-tier (R3/R4) use `allocate-label all`. At R5/R6 the MPLS label (Garnet/Gold side) is stitched to the SRv6 transport of Emerald.

#### R12 (Garnet ASBR) — IOS-XR

```
router bgp 65200
 address-family ipv4 unicast
  allocate-label all
 !
 ! inter-AS eBGP-LU to Gold R21
 neighbor 20.12.21.21
  remote-as 65300
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
 !
 ! inter-AS eBGP-LU to Emerald R6
 neighbor 10.6.12.6
  remote-as 65100
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
 !
 ! iBGP-LU to Garnet RR R19
 neighbor 150.2.19.19
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   next-hop-self
 !
!
route-policy PASS
  pass
end-policy
```

#### R19 (Garnet RR) — IOS-XR

```
router bgp 65200
 address-family ipv4 labeled-unicast
 !
 neighbor-group RRC-LU
  remote-as 65200
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
 !
 neighbor 150.2.12.12 use neighbor-group RRC-LU   ! R12 ASBR
 neighbor 150.2.14.14 use neighbor-group RRC-LU   ! R14 PE
```

**Stitch note (R5/R6):** Emerald is SRv6 (Task 1.5), Garnet/Gold are MPLS. BGP-LU carries MPLS labels in Garnet/Gold; at R5/R6 the BGP-LU next-hop is re-originated so that traffic entering Emerald is encapsulated into an SRv6 End.DT/End.X path instead of a label stack. Document that R5/R6 are the MPLS↔SRv6 boundary — the label-switched path terminates there and SRv6 transport takes over inside Emerald.

**Verify:**
```
show bgp ipv4 labeled-unicast summary
show bgp ipv4 labeled-unicast 150.3.22.22/32
show mpls forwarding | include 150.3.22.22     ! on R14: LSP toward Gold R22
```

### Task 1.3 — Segment Routing on IS-IS (Gold + Garnet): prefix-SID ×10, SRGB/SRLB, anycast SID, TI-LFA, microloop avoidance

**Design:** `segment-routing mpls` under IS-IS IPv4 on all Gold (R20–R23,R30) and Garnet (R11–R19) nodes. SRGB 16000–23999, SRLB 15000–15999. Prefix-SID index = last octet ×10. R14/R15 share anycast index 1140 (label 17140) on `150.2.114.114/32`. TI-LFA + microloop avoidance on all core interfaces.

#### Global SR / SRLB (all SR nodes) — IOS-XR

```
segment-routing
 global-block 16000 23999
 local-block 15000 15999
!
```

#### R22 (Gold PE) — IOS-XR

```
router isis GOLD
 is-type level-1-2
 net 49.0003.1500.0302.2022.00
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
  microloop avoidance protected
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 220          ! label 16220
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
   metric 10
 !
```

> Gold R20→index 200, R21→210, R23→230, R30→300. Garnet R11→110, R13→130, R16→160, R19→190.

#### R14 and R15 — anycast SID (Garnet redundant PE pair) — IOS-XR

```
! identical on BOTH R14 and R15
interface Loopback1
 ipv4 address 150.2.114.114 255.255.255.255
!
router isis GARNET
 interface Loopback1
  passive
  address-family ipv4 unicast
   prefix-sid absolute 17140      ! anycast, both PEs originate label 17140
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 140           ! R14 unique SID (R15 = index 150)
 !
```

**Why anycast:** inter-AS VPN traffic can target `150.2.114.114` and ECMP to whichever of R14/R15 is closer; either PE terminates the service, giving PE redundancy with a single SID.

**Verify:**
```
show isis segment-routing label table
show segment-routing mpls prefix-sid-map
show mpls forwarding labels 16220
show isis fast-reroute ti-lfa
! confirm BOTH R14 and R15 originate 17140:
show mpls forwarding labels 17140
```

### Task 1.4 — SR-TE with explicit SID-lists (prefix-SID + adjacency-SID), binding-SID, autoroute, BGP color autosteer

**Design:** On R14, two explicit (hand-built) SR-TE policies. Policy A (color 300) forces R14→R13→R16→R15 (bypassing the R11↔R15 shortest path). Policy B (color 301) toward Gold R22 pinned via the R12↔R21 inter-AS adjacency-SID. Binding-SID 15300 on Policy A, autoroute include. R15 colors VPNv4 routes 300 to autosteer.

**Step 1 — harvest adjacency-SIDs** (on R13 and the ASBRs):
```
show isis segment-routing label adjacency
show mpls forwarding | include Adj
```
Assume (from the above): R13→R16 protected adj-SID = **24316**, R12→R21 adj-SID = **24221**. Use **protected** adj-SIDs so the segment survives local repair while still pinning the link.

#### R14 — Policy A (color 300 → R15) and Policy B (color 301 → R22) — IOS-XR

```
segment-routing
 traffic-eng
  segment-list SL-TO-R15
   index 10 mpls label 16130      ! prefix-SID R13
   index 20 mpls label 24316      ! adjacency-SID R13->R16 (protected)
   index 30 mpls label 16150      ! prefix-SID R15
  !
  segment-list SL-TO-R22
   index 10 mpls label 16120      ! prefix-SID R12 (ASBR)
   index 20 mpls label 24221      ! adjacency-SID R12->R21 inter-AS
   index 30 mpls label 16220      ! prefix-SID R22 (via BGP-LU/Gold SR)
  !
  policy POLICY-A-R15
   color 300 end-point ipv4 150.2.15.15
   binding-sid mpls 15300
   candidate-paths
    preference 100
     explicit segment-list SL-TO-R15
   autoroute
    include ipv4 150.2.15.15/32
  !
  policy POLICY-B-R22
   color 301 end-point ipv4 150.3.22.22
   candidate-paths
    preference 100
     explicit segment-list SL-TO-R22
 !
!
```

#### R15 — color VPNv4 routes 300 (autosteer at R14) — IOS-XR

```
extcommunity-set opaque COLOR-300
 300
end-set
!
route-policy SET-COLOR-300
  set extcommunity color COLOR-300
  pass
end-policy
!
router bgp 65200
 address-family vpnv4 unicast
 !
 neighbor 150.2.19.19                 ! to RR R19
  address-family vpnv4 unicast
   route-policy SET-COLOR-300 out
 !
```

**Why:** color 300 on the VPNv4 route resolves its next-hop onto POLICY-A at R14 automatically (no PBR/static). The binding-SID 15300 lets an upstream node invoke the whole explicit path with a single label (hierarchical SR-TE).

**Verify:**
```
show segment-routing traffic-eng policy color 300 detail     ! SID list + binding-sid 15300
show segment-routing traffic-eng policy color 301 detail
show bgp vpnv4 unicast <prefix> detail                       ! color 300 -> POLICY-A
traceroute sr-mpls ...                                       ! forced R13->R16->R15 hops
```

### Task 1.5 — SRv6 on Emerald (AS 65100): F3216 uSID locators, PSP/USD, locator tags, SRv6 L3VPN transport

**Design:** Native SRv6 on R1–R8 using uSID (F3216). Locator per node from `fc00:1:<R#>::/48`. End/End.X SIDs with PSP+USD. Locators tagged 1000 for controlled leaking. R31/R2 VRFs use per-VRF SRv6 SIDs (End.DT4/End.DT6) instead of MPLS labels.

#### R31 (PE) — IOS-XR

```
segment-routing
 srv6
  locators
   locator LOC-R1
    micro-segment behavior unode psp-usd
    prefix fc00:1:1::/48
    tag 1000
   !
  !
 !
!
router isis EMERALD
 address-family ipv6 unicast
  segment-routing srv6
   locator LOC-R1
  !
 !
!
! per-VRF SRv6 transport (Blue/Green)
vrf GREEN
 address-family ipv4 unicast
  segment-routing srv6
   locator LOC-R1
   alloc mode per-vrf
 !
 address-family ipv6 unicast
  segment-routing srv6
   locator LOC-R1
   alloc mode per-vrf
 !
!
router bgp 65100
 address-family vpnv4 unicast
  segment-routing srv6
   locator LOC-R1
 !
 address-family vpnv6 unicast
  segment-routing srv6
   locator LOC-R1
 !
 vrf GREEN
  rd auto
  address-family ipv4 unicast
   segment-routing srv6
    alloc mode per-vrf
```

> R2–R8 identical pattern, `prefix fc00:1:<R#>::/48`, `locator LOC-R<#>`. P/RR nodes need only the locator + IS-IS SRv6 advertisement (no VRF alloc).

**Why PSP/USD:** PSP (Penultimate Segment Pop) lets the penultimate hop remove the SRH; USD (Ultimate Segment Decapsulation) lets the final node decapsulate — together they trim SRH processing on transit and at the egress PE. End.DT4/DT6 are the per-VRF decapsulate-and-lookup SIDs that replace the MPLS VPN label.

**Verify:**
```
show segment-routing srv6 sid
show segment-routing srv6 locator LOC-R1 detail
show isis database verbose | include SRv6|Locator
show bgp vpnv4 unicast vrf GREEN <prefix> detail      ! End.DT4 SID present (no MPLS label)
show cef vrf GREEN <prefix>                            ! IPv6 SRv6 encap, not label stack
```

### Task 1.6 — SR-ODN: color-triggered on-demand policies, PCE latency path, ECMP latency margin 3000 us

**Design:** R19 is SR-PCE. R14 is a PCC delegating to R19. VPNv4 routes arriving with color 400 trigger an ODN policy whose dynamic path optimizes min-delay via PCE, with a 3000 us latency margin so all paths within 3000 us of best are installed as ECMP. R15 tags customer routes color 400.

#### R19 (SR-PCE) — IOS-XR

```
pce
 address ipv4 150.2.19.19
!
segment-routing
 traffic-eng
  pcc
   ! (R19 itself is the PCE; PCC config lives on headends)
```

#### R14 (PCC + ODN template) — IOS-XR

```
segment-routing
 traffic-eng
  pcc
   source-address ipv4 150.2.14.14
   pce address ipv4 150.2.19.19
   report-all
  !
  on-demand color 400
   dynamic
    pcep                            ! delegate computation to R19
    metric type latency
    metric margin absolute 3000     ! all paths within 3000us of best -> ECMP
   !
  !
 !
!
```

#### R15 — tag customer routes color 400 — IOS-XR

```
extcommunity-set opaque COLOR-400
 400
end-set
!
route-policy SET-COLOR-400
  set extcommunity color COLOR-400
  pass
end-policy
!
router bgp 65200
 neighbor 150.2.19.19
  address-family vpnv4 unicast
   route-policy SET-COLOR-400 out
```

**Why the margin:** strict min-delay installs only the single lowest-latency path. `metric margin absolute 3000` relaxes that to any path whose latency is within 3000 us of the best, so multiple near-equal paths load-share (ECMP). Tradeoff: more bandwidth/resilience vs some flows taking a slightly-higher-latency path.

**Verify:**
```
show pce lsp
show pce peer                                                   ! R14 as PCC
show segment-routing traffic-eng policy color 400 detail        ! PCEP delegated, margin 3000, >1 path
show bgp vpnv4 unicast <prefix> detail                          ! color 400 -> ODN policy
```

### Task 1.7 — Flex-Algo 129 with include-all affinity, per-algo prefix-SID, constrained SR policy

**Design:** Flex-Algo 129 = min-delay, `affinity include-all BLUE`. Only BLUE-tagged links (R13↔R14, R13↔R16, R11↔R15) participate. Per-algo prefix-SID distinct from Algo-0. SR-TE policy on R14 constrained to algo 129 toward R16.

#### R14 (and all Garnet nodes) — Flex-Algo definition — IOS-XR

```
router isis GARNET
 flex-algo 129
  metric-type delay
  affinity include-all BLUE
  advertise-definition
 !
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 140                 ! algo 0
   prefix-sid algorithm 129 index 1290  ! algo-129 per-algo SID (distinct)
 !
!
affinity-map
 name BLUE bit-position 0
!
! tag participating links BLUE (example R13<->R14 on R14):
router isis GARNET
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   affinity flex-algo name BLUE
 !
```

> Tag the BLUE link-set endpoints: R13↔R14, R13↔R16, R11↔R15. Any link NOT tagged BLUE is excluded from Algo 129 (include-all requires every listed affinity present). Enable `flex-algo 129` participation on all Garnet nodes.

#### R14 — SR-TE policy constrained to Algo 129 toward R16 — IOS-XR

```
segment-routing
 traffic-eng
  policy POLICY-ALGO129-R16
   color 129 end-point ipv4 150.2.16.16
   candidate-paths
    preference 100
     dynamic
      metric type latency
     constraints
      segments
       sid-algorithm 129
```

**Why include-all vs exclude-any:** `include-all BLUE` is affirmative — a link must carry BLUE to be used, so the algo topology is exactly the BLUE overlay. This is the inverse of LAB3's `exclude-any` (which removed only flagged links).

**Verify:**
```
show segment-routing traffic-eng flex-algo
show isis flex-algo 129
show segment-routing traffic-eng policy name POLICY-ALGO129-R16 detail
! confirm a non-BLUE link is never used even if it is the algo-0 shortest path
show isis topology flex-algo 129
```

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — EVPN-VPWS: T-LDP VPWS (R16↔R1) + EVPN-VPWS (R14↔R15)

**Design:** Classic T-LDP VPWS for Red circuit R16↔R1 (PW-ID 2001) stitched across the SR-MPLS↔SRv6 boundary at the ASBRs. EVPN-VPWS (RFC 8214) R14↔R15, EVI 2002, local AC-IDs 2142 (R14) / 2152 (R15).

#### R16 (Garnet PE) — T-LDP VPWS to R1 — IOS-XR

```
mpls ldp
 router-id 150.2.16.16
!
l2vpn
 xconnect group RED
  p2p RED-PW
   interface GigabitEthernet0/0/0/3.100     ! customer access (R27 side)
   neighbor ipv4 150.1.31.31 pw-id 2001
 !
!
```

#### R1 (Emerald PE) — T-LDP VPWS to R16 — IOS-XR

```
l2vpn
 xconnect group RED
  p2p RED-PW
   interface GigabitEthernet0/0/0/2.100     ! customer access (R29 side)
   neighbor ipv4 150.2.16.16 pw-id 2001
 !
!
```

**Stitch note:** R1 is SRv6, R16 is SR-MPLS. The targeted-LDP session rides BGP-LU reachability (Task 1.2); at R5/R6 the PW transport label is stitched to the Emerald SRv6 path. T-LDP needs a targeted session per PW — contrast with EVPN below which uses one BGP control plane.

#### R14 — EVPN-VPWS, EVI 2002, local AC-ID 2142 — IOS-XR

```
router bgp 65200
 address-family l2vpn evpn
 !
 neighbor 150.2.19.19
  address-family l2vpn evpn
!
l2vpn
 xconnect group EVPN-VPWS
  p2p EVPN-R14-R15
   interface GigabitEthernet0/0/0/4.100
   neighbor evpn evi 2002 target 2152 source 2142
 !
!
```

#### R15 — EVPN-VPWS, EVI 2002, local AC-ID 2152 — IOS-XR

```
l2vpn
 xconnect group EVPN-VPWS
  p2p EVPN-R14-R15
   interface GigabitEthernet0/0/0/2.100
   neighbor evpn evi 2002 target 2142 source 2152
 !
!
```

**Why:** EVPN-VPWS cross-connects by matching local `source` AC-ID to remote `target` AC-ID within the EVI; BGP EVPN (RD/RT) is the single control plane over any transport — no per-PW targeted LDP.

**Verify:**
```
show l2vpn xconnect group RED
show evpn evi 2002 detail
show l2vpn xconnect summary                 ! EVPN-VPWS state UP, remote AC-ID matched
show bgp l2vpn evpn
```

### Task 2.2 — L3VPN: Inter-AS Option B on ALL inter-AS links, PE-CE BGP+OSPF, as-override, redistribute-connected

**Design rule:** every inter-AS VPNv4/VPNv6 handoff is Option B (eBGP VPNv4 between ASBR pairs, next-hop-self + fresh label, `retain route-target all`). No Option A anywhere. Green intra-AS: R15↔R17 OSPF, R16↔R18 eBGP 65910. R31/R20 use `redistribute connected` with a scoping route-policy. OSPF PE-CE uses down-bit/domain-tag. as-override on shared-ASN spokes.

#### R12 (ASBR) — Option B VPNv4 to Gold R21 — IOS-XR

```
router bgp 65200
 address-family vpnv4 unicast
  retain route-target all          ! keep RTs we don't import, for re-advertisement
 !
 neighbor 20.12.21.21
  remote-as 65300
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-self                   ! re-originate with fresh label at the boundary
 !
 neighbor 150.2.19.19              ! iBGP to RR R19
  address-family vpnv4 unicast
   next-hop-self
```

> R5↔R11, R6↔R12, R6↔R20 ASBR pairs follow the same Option B pattern (eBGP vpnv4, retain route-target all, next-hop-self). No VRF-to-VRF (Option A) anywhere.

#### R15 — Green VRF, OSPF PE-CE to R17 — IOS-XR

```
vrf GREEN
 address-family ipv4 unicast
  import route-target 65200:100
  export route-target 65200:100
!
router ospf GREEN-PE-CE
 vrf GREEN
  domain-id 0.0.0.100
  address-family ipv4 unicast
  area 0
   interface GigabitEthernet0/0/0/0    ! 192.168.6.15 -> R17
 !
!
router bgp 65200
 vrf GREEN
  rd 65200:100
  address-family ipv4 unicast
   redistribute ospf GREEN-PE-CE
```

#### R16 — Green VRF, eBGP PE-CE to R18 (as-override) — IOS-XR

```
router bgp 65200
 vrf GREEN
  rd 65200:100
  address-family ipv4 unicast
   redistribute connected route-policy GREEN-CONN   ! (style practice; see R31/R20)
  neighbor 192.168.7.18
   remote-as 65910
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override                                      ! rewrite 65910 so spoke accepts
```

#### R31 and R20 — redistribute connected, scoped — IOS-XR

```
! R1 (Emerald PE, Blue VRF toward R32); R20 (Gold PE, Yellow VRF toward R24)
prefix-set PE-CE-CONN
 192.168.3.0/24,       ! R1<->R32 Blue   (on R20: 192.168.9.0/24 R20<->R24 Yellow)
end-set
!
route-policy SCOPE-CONN
  if destination in PE-CE-CONN then
    pass
  else
    drop
  endif
end-policy
!
router bgp 65100
 vrf BLUE
  rd 65100:25
  address-family ipv4 unicast
   redistribute connected route-policy SCOPE-CONN    ! only the PE-CE subnet, never core/mgmt
```

**Why:** `redistribute connected` without a route-policy would leak core and management connecteds into the VPN. The prefix-set guardrail scopes it to exactly the PE-CE subnet. OSPF `domain-id` + down-bit stops the core being seen as OSPF Area 0 (prevents false intra-area inference / routing loops). `as-override` rewrites the provider-seen peer ASN so a spoke CE sharing the ASN doesn't drop the route on AS_PATH loop detection.

**Verify:**
```
show bgp vpnv4 unicast vrf GREEN
show bgp vpnv4 unicast neighbor 20.12.21.21 advertised-routes   ! NH-self + new label each Option B hop
show route vrf GREEN <prefix>
show bgp vpnv4 unicast vrf BLUE                                 ! R31/R20 leak only intended connecteds
```

### Task 2.3 — Multicast VPN: mVPN Profile 3 (BGP C-multicast, mLDP P2MP core)

**Design:** PIM-free core via mLDP P2MP LSPs for Default/Data MDT; BGP MVPN SAFI for auto-discovery and C-multicast (Type 6/7) joins. Garnet PEs R14/R15/R16. Data-MDT range with switchover threshold.

#### R14 (and R15, R16) — Profile 3 — IOS-XR

```
mpls ldp
 mldp
!
multicast-routing
 vrf GREEN
  address-family ipv4
   mdt source Loopback0
   mdt default mldp p2mp              ! Default MDT = mLDP P2MP tree (no PIM core)
   mdt data mldp 100 threshold 1000   ! Data MDT pool, switchover at 1000 kbps
   bgp auto-discovery mldp            ! BGP MVPN auto-discovery
   rate-per-route
 !
!
router bgp 65200
 address-family ipv4 mvpn
 !
 neighbor 150.2.19.19
  address-family ipv4 mvpn
 !
 vrf GREEN
  address-family ipv4 mvpn
   bgp c-multicast-routing            ! C-joins in BGP Type 6/7, not PIM across core
!
router pim
 vrf GREEN
  address-family ipv4
   ! PIM runs ONLY toward CEs, never in the core
   interface GigabitEthernet0/0/0/0   ! PE-CE facing
    enable
```

**Why Profile 3:** Default/Data MDTs are mLDP P2MP LSPs so there is no PIM adjacency in the provider core; customer (*,G)/(S,G) state is carried PE→PE as BGP MVPN Type 6/7 routes. High-bandwidth sources crossing the data-MDT threshold spawn a new mLDP P2MP data tree and the receiver PE issues a BGP Type 7 join.

**Verify:**
```
show mpls mldp database
show mvpn vrf GREEN pe
show bgp ipv4 mvpn
show pim vrf GREEN topology           ! NO PIM neighbors in core; PIM only to CE
show mvpn vrf GREEN database
```

### Task 2.4 — QoS: ingress marking + core MPLS EXP (traffic-class) treatment (uniform mode, no shaper)

**Design:** R14 ingress classifies Green CE traffic (DSCP) into 3 classes and sets MPLS EXP on imposition (voice→5, biz→3, BE→0), uniform mode. Core (R13/R11/R19) matches EXP and applies PHB. Egress PE R16 maps EXP back to DSCP. No shaping anywhere.

#### R14 — ingress marking (customer-facing) — IOS-XR

```
class-map match-any VOICE
 match dscp ef
!
class-map match-any BIZ
 match dscp af31 af32
!
policy-map CE-INGRESS-MARK
 class VOICE
  set mpls experimental imposition 5
  set traffic-class 5
 class BIZ
  set mpls experimental imposition 3
  set traffic-class 3
 class class-default
  set mpls experimental imposition 0
  set traffic-class 0
!
interface GigabitEthernet0/0/0/0            ! Green CE ingress
 service-policy input CE-INGRESS-MARK
!
! Uniform mode is the IOS-XR default for MPLS EXP; make explicit:
mpls traffic-eng
!  (DiffServ tunneling: uniform — EXP stays coupled to customer QoS through the core)
```

#### R13 / R11 / R19 — core per-EXP treatment (egress on core links) — IOS-XR

```
class-map match-any EXP5
 match mpls experimental topmost 5
!
class-map match-any EXP3
 match mpls experimental topmost 3
!
policy-map CORE-EXP-PHB
 class EXP5
  priority level 1                 ! LLQ for voice
  police rate percent 20
 class EXP3
  bandwidth percent 40             ! guaranteed for business
 class class-default
  bandwidth percent 30
  random-detect default            ! WRED on best-effort
!
interface GigabitEthernet0/0/0/1   ! core-facing egress
 service-policy output CORE-EXP-PHB
```

#### R16 — egress PE, EXP→DSCP back to CE (uniform) — IOS-XR

```
policy-map EGRESS-EXP-TO-DSCP
 class EXP5
  set dscp ef
 class EXP3
  set dscp af31
 class class-default
  set dscp default
!
interface GigabitEthernet0/0/0/1   ! toward Green CE R18
 service-policy output EGRESS-EXP-TO-DSCP
```

**Why uniform mode:** the EXP marking stays coupled to the customer's QoS end-to-end — a re-marking in the core propagates back out, keeping consistent PHB. No shaper is used; the design is edge-marking + core per-EXP queuing.

**Verify:**
```
show policy-map interface GigabitEthernet0/0/0/0 input      ! marking/EXP-set counters (R14)
show policy-map interface GigabitEthernet0/0/0/1 output     ! per-EXP class + queue stats (core)
show mpls forwarding                                        ! EXP preserved in label stack
```

### IOS-XE CE reference (Green customers)

#### R17 (IOS-XE, OSPF PE-CE to R15)

```
interface GigabitEthernet0/0
 ip address 192.168.6.17 255.255.255.0
!
router ospf 1
 network 192.168.6.0 0.0.0.255 area 0
 network 150.17.17.17 0.0.0.0 area 0
```

#### R18 (IOS-XE, eBGP 65910 PE-CE to R16, dual-stack for 6VPE)

```
interface GigabitEthernet0/0
 ip address 192.168.7.18 255.255.255.0
 ipv6 address 2001:DB8:7::18/64
!
router bgp 65910
 bgp log-neighbor-changes
 neighbor 192.168.7.16 remote-as 65200
 neighbor 2001:DB8:7::16 remote-as 65200
 !
 address-family ipv4
  network 150.18.18.18 mask 255.255.255.255
  neighbor 192.168.7.16 activate
 !
 address-family ipv6
  network 2001:DB8:18::18/128
  neighbor 2001:DB8:7::16 activate
```

---

## Section 3 — Access Connectivity (25%)

### Task 3.1 — Site of Origin (SoO) on Blue CE R25 (multi-homed to R14, R21, R23)

**Design:** Blue R25 multi-homes into R14 (65200, 192.168.5.0/24), R21 (65300, 192.168.10.0/24), R23 (65300, 192.168.12.0/24). SoO `65025:2500` applied inbound on all three PEs; outbound policy rejects routes already carrying it; SoO must survive Option B re-origination (Task 2.2).

#### R14 — SoO inbound + outbound filter on Blue PE-CE — IOS-XR

```
extcommunity-set soo SOO-R25
 65025:2500
end-set
!
route-policy BLUE-IN-SOO
  set extcommunity soo SOO-R25
  pass
end-policy
!
route-policy BLUE-OUT-SOO
  if extcommunity soo matches-any SOO-R25 then
    drop                      ! never advertise R25's own routes back to R25
  else
    pass
  endif
end-policy
!
router bgp 65200
 vrf BLUE
  rd 65200:25
  neighbor 192.168.5.25
   remote-as 65025
   address-family ipv4 unicast
    route-policy BLUE-IN-SOO in
    route-policy BLUE-OUT-SOO out
    as-override
```

> R21 (neighbor 192.168.10.25) and R23 (neighbor 192.168.12.25) use the **identical** SoO `65025:2500` inbound and the same outbound drop. Same SoO value = same site, so any PE recognizes an R25-originated route and refuses to loop it back.

#### R12 / R21 ASBR — ensure SoO survives Option B — IOS-XR

```
! On the Option B inter-AS vpnv4 policy, do NOT strip extended communities.
route-policy PASS
  pass                        ! passes extcommunities incl. SoO; verify not deleted
end-policy
!
! If a policy elsewhere does `delete extcommunity all`, exempt SoO:
route-policy KEEP-SOO
  delete extcommunity rt all
  pass                        ! deletes RT only, keeps SoO 65025:2500
end-policy
```

**Why:** SoO is a per-site stamp. Applied inbound at every PE facing R25, it marks all routes learned from that site; the outbound filter drops any route still carrying the stamp before re-advertising toward R25 — preventing the multi-homing loop. Across Option B, the ASBR must not delete the SoO extended-community during re-origination or the loop protection breaks at the AS boundary.

**Verify:**
```
show bgp vpnv4 unicast vrf BLUE 150.25.25.25/32 detail    ! SoO 65025:2500 present + used as filter
show bgp vpnv4 unicast vrf BLUE                           ! on R25-facing sessions: own-site routes absent
! after R12<->R21 Option B hop: confirm SoO still attached
```

#### R25 (IOS-XE CE) — three-way multi-home

```
interface GigabitEthernet0/0
 ip address 192.168.5.25 255.255.255.0        ! to R14
interface GigabitEthernet0/1
 ip address 192.168.10.25 255.255.255.0       ! to R21
interface GigabitEthernet0/2
 ip address 192.168.12.25 255.255.255.0       ! to R23
!
router bgp 65025
 network 150.25.25.25 mask 255.255.255.255
 neighbor 192.168.5.14 remote-as 65200
 neighbor 192.168.10.21 remote-as 65300
 neighbor 192.168.12.23 remote-as 65300
```

### Task 3.2 — 6VPE for the Green customer IPv6

**Design:** Add IPv6 to Green VRF on Garnet PEs. R16↔R18 eBGP adds IPv6 AF; R15↔R17 adds OSPFv3. VPNv6 AF to RR R19 and across Option B. Over SRv6 (Emerald) rides End.DT6.

#### R16 — Green VPNv6 (6VPE) with eBGP PE-CE IPv6 — IOS-XR

```
vrf GREEN
 address-family ipv6 unicast
  import route-target 65200:100
  export route-target 65200:100
!
router bgp 65200
 address-family vpnv6 unicast
 !
 neighbor 150.2.19.19
  address-family vpnv6 unicast
 !
 vrf GREEN
  address-family ipv6 unicast
   redistribute connected
  neighbor 2001:db8:7::18
   remote-as 65910
   address-family ipv6 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
```

#### R15 — Green IPv6 PE-CE via OSPFv3 to R17 — IOS-XR

```
router ospfv3 GREEN-V6
 vrf GREEN
  address-family ipv6 unicast
  area 0
   interface GigabitEthernet0/0/0/0
 !
!
router bgp 65200
 vrf GREEN
  address-family ipv6 unicast
   redistribute ospfv3 GREEN-V6
```

**Why 6VPE:** IPv6 VPN prefixes are carried as VPNv6 NLRI over the same IPv4/MPLS or SR core; the per-VRF label (or SRv6 End.DT6 SID on Emerald) decapsulates and does an IPv6 VRF lookup at the egress PE. Over Option B the ASBR re-originates VPNv6 with next-hop-self the same way as VPNv4.

**Verify:**
```
show bgp vpnv6 unicast vrf GREEN summary
show bgp vpnv6 unicast vrf GREEN <prefix> detail    ! label or End.DT6 SID
show route vrf GREEN ipv6 unicast
ping vrf GREEN 2001:db8:18::18
```

### Task 3.3 — Troubleshooting MCQs: EVPN-VPWS ESI/DF, multi-homing mode

**MCQ 1 — duplicate frames to dual-homed CE.**
**Answer: (b)** — the ESI is **type-0 (all-zero)**, so DF election never ran. A single-active service needs a non-zero ESI (type-1/3) for the PEs to run DF election and block the non-DF; with ESI 0 both PEs forward, producing duplicates. (a) a mismatched non-zero ESI would break the service differently; (c)/(d) EVI/RT mismatch would prevent the service coming up at all, not duplicate.
Confirm with: `show evpn ethernet-segment detail` (ESI value + DF/non-DF role).

**MCQ 2 — is EVPN all-active valid for three-way multi-homed Blue R25?**
**Answer: No — loop prevention must be at L3 via SoO (Task 3.1).** EVPN ESI multi-homing is a **single-L2-broadcast-domain / single-administrative-core** construct: the DF election and split-horizon rely on all participating PEs sharing one EVPN control plane and ES. R25 is an **L3VPN CE multi-homed across three separate ASes**, so there is no common Ethernet segment and no shared EVPN domain — ESI multi-homing cannot apply. SoO is the correct tool because it is an L3/BGP site stamp that works across ASes (and survives Option B).
Prove L3 loop prevented with: `show bgp vpnv4 unicast vrf BLUE` on R25-facing PEs (R25's own routes absent on the outbound side).

---

## Section 4 — High Availability (10%)

### Task 4.1 — MPLS-TE (RSVP-TE) in AS 65200 with OSPF TE extensions + FRR bypass

**Design:** Enable MPLS-TE globally + per-core-interface in Garnet, OSPF TE opaque LSAs (Type-10), RSVP bandwidth reservations. Primary tunnel R14→R16 with explicit ERO + dynamic fallback, autoroute announce. FRR link protection via bypass tunnel for sub-50ms repair.

#### R14 (headend) — IOS-XR

```
mpls traffic-eng
 interface GigabitEthernet0/0/0/1       ! core-facing (R13)
!
rsvp
 interface GigabitEthernet0/0/0/1
  bandwidth 500000                      ! kbps reservable
!
router ospf GARNET
 area 0
  mpls traffic-eng
 mpls traffic-eng router-id Loopback0
!
explicit-path name R14-TO-R16
 index 10 next-address strict ipv4 unicast 20.13.14.13   ! via R13
 index 20 next-address strict ipv4 unicast 20.13.16.16   ! to R16
!
interface tunnel-te0
 ipv4 unnumbered Loopback0
 autoroute announce
 destination 150.2.16.16
 path-option 1 explicit name R14-TO-R16
 path-option 2 dynamic
 signalled-bandwidth 200000
 fast-reroute                            ! request node/link protection
!
```

#### R13 (protecting node) — bypass tunnel for FRR — IOS-XR

```
mpls traffic-eng
 interface GigabitEthernet0/0/0/0        ! protected link toward R16
!
rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth 500000
!
interface tunnel-te100
 ipv4 unnumbered Loopback0
 destination 150.2.16.16
 path-option 1 explicit name BYPASS-R13-R16
!
explicit-path name BYPASS-R13-R16
 index 10 next-address strict ipv4 unicast 20.11.13.11   ! R13->R11
 index 20 next-address strict ipv4 unicast 20.11.15.15   ! R11->R15 ... to R16 avoiding protected link
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/0
  backup-path tunnel-te100               ! bind bypass to the protected interface
```

**Why:** OSPF floods TE link attributes as Type-10 opaque LSAs, populating the TE database the headend CSPF uses to honor the ERO + bandwidth. `fast-reroute` on the primary plus `backup-path` on the protecting node pre-programs a bypass so a protected-link failure triggers local repair in <50ms while the headend re-optimizes.

**Verify:**
```
show mpls traffic-eng tunnels
show mpls traffic-eng topology            ! OSPF-sourced TE links present
show rsvp session
show mpls traffic-eng fast-reroute database   ! protected + bypass READY
```

### Task 4.2 — BFD for IS-IS (not BGP)

**Design:** BFD on IS-IS core interfaces of Gold (R20–R23,R30) and Emerald where applicable. Interval 150ms ×3. Ties data-plane failure to IS-IS adjacency teardown to accelerate SPF/TI-LFA. Independent of BGP.

#### R20 (and all Gold core) — IOS-XR

```
router isis GOLD
 interface GigabitEthernet0/0/0/0
  bfd minimum-interval 150
  bfd multiplier 3
  bfd fast-detect ipv4
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
 !
 interface GigabitEthernet0/0/0/1
  bfd minimum-interval 150
  bfd multiplier 3
  bfd fast-detect ipv4
  point-to-point
```

**Why:** without BFD, IS-IS waits the hold-time (seconds) to notice a dead neighbor on a link that stays "up" (e.g., through an intermediate L2 device). BFD detects loss in ~450ms and tears the adjacency down immediately, so SPF runs and the pre-computed TI-LFA backup (Task 1.3) covers the microsecond gap. The objective here is IGP convergence; BGP is not the BFD client.

**Verify:**
```
show bfd session                              ! client includes ISIS
show isis interface GigabitEthernet0/0/0/0 | include BFD
show isis neighbors detail
show logging | include ADJCHANGE|BFD          ! timestamped teardown around induced failure
```

### Task 4.3 — OSPF NSF / Graceful-Restart troubleshooting MCQ

**MCQ — R15 RP restart causes a brief outage despite NSF/GR configured.**
**Answer: (a)** — R15 has NSF/GR enabled but neighbors **R11 and R19 are not graceful-restart helpers**, so instead of holding R15's LSAs for the grace period they age them on the normal dead-interval and reconverge, causing the blip. GR requires the *neighbors* to act as helpers (extend the restarting router's LSA lifetime and keep forwarding) for a hitless restart. (b) NSF doesn't require BFD — in fact BFD would *defeat* NSF by tearing the adjacency immediately; (c) NSF works for OSPFv2 and v3; (d) possible in theory but the restart here is planned/short.

Confirm a neighbor is a helper: `show ospf neighbor detail` (look for the GR/helper state) and `show ospf` (NSF capability).
Fix: enable GR helper on R11/R19:
```
router ospf GARNET
 nsf ietf                 ! IOS-XR: nsr / nsf; helper mode on by default when peer signals GR
 ! ensure not disabled:  no capability graceful-restart-helper  (must be ABSENT)
```
**RSVP-TE interaction (Task 4.1):** during the restart, RSVP uses its own state-recovery (graceful restart / hello) to keep the TE tunnels and FRR bypass up — confirm the tunnels did not re-signal (`show mpls traffic-eng tunnels` stable across the restart).

---

## Section 5 — Security (10%)

### Task 5.1 — Management plane protection + IS-IS keychain (completes Task 1.1)

**Design:** MPP inband on R31,R6,R12,R14,R20 permitting SSH+SNMP only from 10.255.0.0/24, explicitly denying Telnet/XML/TFTP. Complete `ISIS-KEY-L4` HMAC-MD5 with infinite lifetimes, applied for hello+LSP at both IS-IS levels on Emerald core.

#### R14 (and R1,R6,R12,R20) — MPP inband — IOS-XR

```
ipv4 access-list MGMT-OK
 10 permit tcp 10.255.0.0 0.0.0.255 any eq 22       ! SSH
 20 permit udp 10.255.0.0 0.0.0.255 any eq snmp     ! SNMP
!
control-plane
 management-plane
  inband
   interface all
    allow SSH peer
     address ipv4 10.255.0.0/24
    allow SNMP peer
     address ipv4 10.255.0.0/24
    ! Telnet, XML (NETCONF/XML agent), TFTP are NOT allowed = denied.
    ! Explicit deny posture: only SSH/SNMP from mgmt subnet are permitted.
 !
!
```

> MPP inband only governs for-us (management) traffic on data interfaces; transit/data-plane forwarding is unaffected. Omitting a protocol denies it; here Telnet/XML/TFTP are intentionally not listed.

#### Emerald core (R1–R8) — complete keychain `ISIS-KEY-L4` — IOS-XR

```
key chain ISIS-KEY-L4
 key 1
  accept-lifetime 00:00:00 january 01 2024 infinite
  send-lifetime   00:00:00 january 01 2024 infinite
  key-string Cciesp#Lab4Key
  cryptographic-algorithm HMAC-MD5
!
router isis EMERALD
 lsp-password keychain ISIS-KEY-L4 level 1
 lsp-password keychain ISIS-KEY-L4 level 2
 interface GigabitEthernet0/0/0/0
  hello-password keychain ISIS-KEY-L4          ! both levels (one keychain covers L1+L2)
```

**Design decision:** one keychain `ISIS-KEY-L4` authenticates both levels here (simpler, single rollover point). If per-level isolation were required you would build `ISIS-KEY-L4-L1` / `-L2` and apply `lsp-password keychain ... level 1|2` with distinct keys — document the choice.

**Verify:**
```
show control-plane-protection management inband counters
! from inside 10.255.0.0/24: ssh works; from outside: telnet fails
show key chain ISIS-KEY-L4
show isis interface detail                     ! HMAC-MD5 per level
show isis adjacency detail
```

### Task 5.2 — BGP Flowspec, TCP-AO on eBGP, RSVP authentication

#### 5.2.1 — BGP Flowspec (R19 controller) — IOS-XR

```
flowspec
 address-family ipv4
  service-policy type pbr FS-DDOS
!
class-map type traffic match-all FS-MATCH
 match destination-address ipv4 150.18.18.18 255.255.255.255
 match protocol udp
!
policy-map type pbr FS-DDOS
 class type traffic FS-MATCH
  police rate 1 mbps                 ! rate-limit action
  redirect ipv4 nexthop route-target 65200:999   ! redirect-to-VRF action
 class type traffic class-default
!
router bgp 65200
 address-family ipv4 flowspec
 !
 neighbor 150.2.14.14
  address-family ipv4 flowspec       ! advertise rule to R14/R15/R16
```

> **Dataplane limitation note:** IOS-XRv 9000 may advertise the Flowspec NLRI in the control plane but **not program the drop/rate-limit/redirect in the data plane**. Document control-plane state only; do not claim dataplane enforcement.
> ```
> show bgp ipv4 flowspec             ! rule present in BGP
> show flowspec afi-all              ! installed rule (or note: not programmed in XRv)
> ```

#### 5.2.2 — TCP-AO on inter-AS eBGP (R12↔R21) — IOS-XR

```
key chain BGP-AO
 key 1
  key-string Cciesp#AoKey1
  cryptographic-algorithm HMAC-SHA1-96
  accept-lifetime 00:00:00 january 01 2024 infinite
  send-lifetime   00:00:00 january 01 2024 infinite
!
router bgp 65200
 neighbor 20.12.21.21
  remote-as 65300
  ao BGP-AO include-tcp-options enable
```

**Why TCP-AO vs MD5:** TCP-AO (RFC 5925) supports a Master Key Tuple with multiple key-ids and hitless rollover (negotiate send/receive key-id), and modern HMAC-SHA algorithms — MD5 has a single static key, no in-band rollover, and a weaker hash.

**Verify:** `show bgp neighbor 20.12.21.21 | include AO|Authentication` (session UP with AO).

#### 5.2.3 — RSVP authentication on Task 4.1 TE interfaces — IOS-XR

```
key chain RSVP-AUTH
 key 1
  key-string Cciesp#RsvpKey
  cryptographic-algorithm HMAC-MD5
  accept-lifetime 00:00:00 january 01 2024 infinite
  send-lifetime   00:00:00 january 01 2024 infinite
!
rsvp
 interface GigabitEthernet0/0/0/1
  authentication
   key-source key-chain RSVP-AUTH
```

**Why:** authenticates PATH/RESV so a spoofed message cannot create or tear down reservations. Confirm Task 4.1 tunnels re-establish with auth on.

**Verify:**
```
show rsvp interface detail                 ! authentication enabled
show mpls traffic-eng tunnels              ! Task 4.1 tunnels stay UP
```

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry troubleshooting (gRPC TLS, TPA/VRF source, port vs MPP)

**Scenario:** R14 model-driven telemetry → collector, no data arriving. Baseline config:

```
telemetry model-driven
 destination-group DG1
  address-family ipv4 10.255.0.50 port 57500
   encoding self-describing-gpb
   protocol grpc tls-hostname collector.example   ! TLS must match collector
 !
 sensor-group SG1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription SUB1
  sensor-group-id SG1 sample-interval 30000
  destination-id DG1
 !
!
grpc
 port 57500
 tls
```

**Isolation sequence:**
1. **TLS match** — if the collector listens no-TLS but DG1 has `protocol grpc tls-...` (or vice-versa) the dial-out fails silently. Confirm both ends agree (`show grpc status`). *Most common cause.*
2. **TPA source-address / VRF** — ensure the session sources from a collector-reachable, advertised address and the correct VRF; a loopback invisible to the mgmt network or wrong VRF = no session (`show tpa`).
3. **Port vs MPP** — the gRPC port (57500) must match the collector listener **and** be permitted by Task 5.1 MPP inband; if MPP denies it the session silently fails.

**Verify:**
```
show telemetry model-driven destination-group DG1
show telemetry model-driven subscription SUB1       ! state ACTIVE vs Connecting
show grpc status
show tpa
```

### Task 6.2 — NSO: onboard, sync-from, device template, deploy L3VPN service

```
! --- onboard R14/R15 (NSO CLI, config mode) ---
devices authgroups group IOSXR-AUTH umap admin remote-name admin remote-password <pw>
devices device R14 address 150.2.14.14 port 830
 device-type cli ned-id cisco-iosxr-cli-7.x
 authgroup IOSXR-AUTH
 state admin-state unlocked
!
devices device R15 address 150.2.15.15 port 830
 device-type cli ned-id cisco-iosxr-cli-7.x
 authgroup IOSXR-AUTH
 state admin-state unlocked
commit

! --- fetch host keys + sync CDB BEFORE any change ---
request devices device R14 ssh fetch-host-keys
request devices device R15 ssh fetch-host-keys
request devices device R14 sync-from
request devices device R15 sync-from

! --- device template (baseline SNMP) ---
devices template BASELINE
 config
  cisco-ios-xr:snmp-server community LAB4-RO view ... RO
!
devices device-group GARNET-PE device [ R14 R15 ]
request devices device-group GARNET-PE apply-template template-name BASELINE

! --- L3VPN service instance ---
l3vpn CUST-NEW
 vrf-name CUST_NEW
 rd 65200:777
 rt 65200:777
 endpoint R14
  pe-interface GigabitEthernet0/0/0/5
  ce-address 192.168.77.2
  pe-address 192.168.77.1
 endpoint R15
  pe-interface GigabitEthernet0/0/0/5
  ce-address 192.168.78.2
  pe-address 192.168.78.1
commit dry-run       ! inspect native XR config first
commit
```

**Verify:**
```
show devices device R14 sync-from
show devices device R14   state  oper-state
! on routers:
show running-config vrf CUST_NEW
```

### Task 6.3 — NSO troubleshooting: identify out-of-sync device, re-deploy reconcile

**Scenario:** a manual CLI change on R14 or R15 made the L3VPN service `check-sync` fail; which device drifted is unknown.

```
! 1. find the drifted device
request devices device * check-sync
!   -> out-of-sync reported for exactly one device (e.g., R14)

! 2. see WHAT drifted
request devices device R14 compare-config

! 3. remediate — two choices:
!   (a) KEEP the manual change (learn it into CDB):
request devices device R14 sync-from
!   (b) OVERWRITE drift with NSO intent (service authoritative), preserving non-service config:
request l3vpn CUST-NEW re-deploy reconcile
!   or push stored CDB back to the device:
request devices device R14 sync-to

! 4. confirm back in-sync
request devices device R14 check-sync      ! -> in-sync
```

**Tradeoff:** `sync-from` accepts the operator's out-of-band change as new truth (risk: codifies unreviewed config). `re-deploy reconcile` / `sync-to` reasserts NSO as source of truth and reverts the drift (risk: discards a change that may have been an intentional emergency fix) — `reconcile` keeps NSO-owned service config authoritative while leaving unrelated non-service config intact.

**Verify:**
```
request devices device * check-sync
request devices device R14 compare-config    ! empty after remediation
show devices commit-queue
! on router: show running-config vrf CUST_NEW matches NSO intent
```

### Task 6.4 — Ansible troubleshooting MCQ (UnicodeDecodeError)

**MCQ — `cisco.iosxr` playbook to R15 fails with `UnicodeDecodeError: 'ascii' codec can't decode byte ...`.**
**Answer: (a)** — the control node's locale is `C`/`POSIX` (ASCII), so Python cannot decode a non-ASCII byte in device output or a template (e.g., an accented character in an interface description or a byte in a banner). (b) wrong NED version, (c) NETCONF disabled, and (d) a BGP AF error would produce different, protocol-level failures — not a codec error.

**Why:** under a `C`/ASCII locale Python's default text codec is ASCII; any byte ≥ 0x80 raises `UnicodeDecodeError`. Python 3 with a UTF-8 locale decodes the full byte range, so the output/template parses cleanly.

**Fix (control node):**
```
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```
Inventory / `ansible.cfg`:
```
# inventory
[iosxr:vars]
ansible_python_interpreter=/usr/bin/python3
```
**MPP interaction (Task 5.1):** make sure the transport Ansible uses (NETCONF over SSH / gRPC) is permitted by the inband MPP policy — a transport blocked by MPP is a *connectivity* failure, not an encoding fault; don't confuse the two.

---

## Appendix — Verification Quick Index

| Section | Headline checks |
|---|---|
| 1.1 | `show isis neighbors` (L1/L1L2/L2), `show route 150.1.7.7/32` on R31 (leaked) |
| 1.2 | `show bgp ipv4 labeled-unicast 150.3.22.22/32`, `show mpls forwarding` |
| 1.3 | `show isis segment-routing label table`, both R14/R15 originate 17140 |
| 1.4 | `show segment-routing traffic-eng policy color 300 detail` (SID list + BSID 15300) |
| 1.5 | `show segment-routing srv6 sid`, End.DT4/DT6 in `show bgp vpnv4 ... detail` |
| 1.6 | `show pce lsp`, color 400 policy with margin 3000 + >1 ECMP path |
| 1.7 | `show isis flex-algo 129`, Algo-129 stays on BLUE links |
| 2.1 | `show evpn evi 2002 detail` (AC-IDs 2142/2152), `show l2vpn xconnect` |
| 2.2 | `show bgp vpnv4 ... advertised-routes` (NH-self + new label per Option B hop) |
| 2.3 | `show mpls mldp database`, `show pim vrf GREEN topology` (no core PIM) |
| 2.4 | `show policy-map interface ... input/output` (EXP set / per-EXP queues) |
| 3.1 | SoO `65025:2500` present + own-site routes absent; survives Option B |
| 3.2 | `show bgp vpnv6 unicast vrf GREEN` (label/End.DT6) |
| 4.1 | `show mpls traffic-eng tunnels`, FRR database protected+bypass |
| 4.2 | `show bfd session` (client ISIS) |
| 5.1 | MPP counters; SSH-in works, Telnet-out fails; `show key chain` |
| 5.2 | `show bgp ipv4 flowspec` (+ dataplane note), BGP AO up, RSVP auth on |
| 6.1 | `show grpc status`, `show telemetry ... subscription` |
| 6.3 | `devices device * check-sync` → in-sync after reconcile |

> **LAB4 vs LAB3 recap:** multi-level IS-IS · prefix-SID ×10 · Flex-Algo include-all · explicit SID-lists w/ adj-SIDs · SRv6 in Emerald · ODN 3000us ECMP margin · Option B everywhere · EVI 2001/2002 · mVPN Profile 3 · QoS ingress-mark+EXP · redistribute-connected R31/R20 · SoO on Blue R25 · 6VPE for Green · RSVP MPLS-TE AS65200 · BFD for IS-IS · Flowspec+TCP-AO+RSVP-auth · NSO reconcile · Ansible encoding MCQ.
