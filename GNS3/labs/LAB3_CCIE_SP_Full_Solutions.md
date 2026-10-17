# LAB 3 — CCIE SP v5.1 Full Workbook — SOLUTIONS

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+) for all SP routers; CSR1000v / IOSv (IOS-XE) for CE routers.
**Companion to:** `LAB3_CCIE_SP_Full.md` (tasks-only workbook).

> **Conventions used in this solution set**
> - Loopback0: `150.<AS-last>.<R#>.<R#>/32` (R1=150.1.31.31, R11=150.2.11.11, R20=150.3.20.20).
> - IPv6 Loopback0: `2001:db8:<AS-last>::<R#>/128` (R1=2001:db8:1::1, R20=2001:db8:3::20).
> - AS65100 links `10.x.y.z/24`, AS65200 links `20.x.y.z/24`, AS65300 links `30.x.y.z/24`.
> - PE-CE links `192.168.<CE#>.<Y>/24`.
> - Example interfaces: `GigabitEthernet0/0/0/x` on XRv, `GigabitEthernet1..n` on CE (IOS-XE).
> - Interface numbers below are representative — map to your actual EVE-NG cabling.

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS L2 via flex-CLI configuration groups

**Design.** Two reusable `group`s are built: one for the IS-IS *instance* stanza (wide metrics, single-topology, keychain) and one for the per-*interface* stanza (p2p, hello-padding disable, LDP sync, afi metrics, keychain auth). `apply-group` then stamps the common config onto every router, so per-interface repetition disappears.

#### Configuration groups (identical on every IS-IS router — AS65100 R1–R8, AS65300 R20–R23/R30)

```
! ---- Flex-CLI groups (global config scope) ----
group ISIS-INSTANCE-COMMON
 router isis '.*'
  is-type level-2-only
  net '.*'
  nsf cisco
  log adjacency changes
  lsp-gen-interval maximum-wait 5000 initial-wait 50 secondary-wait 200
  address-family ipv4 unicast
   metric-style wide
   metric 200
   microloop avoidance segment-routing
   fast-reroute per-prefix
  !
  address-family ipv6 unicast
   metric-style wide
   metric 400
   single-topology
  !
 !
end-group
!
group ISIS-INTF-COMMON
 router isis '.*'
  interface 'GigabitEthernet.*'
   point-to-point
   hello-padding disable
   address-family ipv4 unicast
    fast-reroute per-prefix ti-lfa
   !
   address-family ipv6 unicast
   !
  !
 !
end-group
!
group ISIS-AUTH-COMMON
 router isis '.*'
  lsp-password keychain ISIS-KC
  interface 'GigabitEthernet.*'
   hello-password keychain ISIS-KC
  !
 !
end-group
```

#### Per-router instance (example R1, AS65100) — references the groups

```
apply-group ISIS-INSTANCE-COMMON ISIS-INTF-COMMON ISIS-AUTH-COMMON
!
router isis CORE
 net 49.0001.1500.1001.0001.00          ! system-id from 150.1.31.31
 address-family ipv4 unicast
 !
 address-family ipv6 unicast
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 1                     ! (SR-MPLS, Task 1.5)
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
  !
 !
```

#### Overload bit on RRs — R7, R8 (AS65100) and R30 (AS65300)

```
router isis CORE
 set-overload-bit on-startup 180
```

**Why each element**
- `is-type level-2-only` — single flat L2 domain, no L1 leakage, matches blueprint.
- `metric-style wide` + `metric 200/400` — wide TLVs are mandatory for SR/TE; IPv4 default cost 200, IPv6 400.
- `single-topology` — one SPF carries both AFs (no MT), as required.
- `point-to-point` + `hello-padding disable` — faster adjacency, no padding overhead on p2p core.
- `passive` Loopback0 — advertise the /32 without forming an adjacency on it.
- `set-overload-bit on-startup 180` — RRs don't attract transit for 180 s after reload while BGP reconverges.
- `lsp-password` / `hello-password keychain` — HMAC-MD5 auth, secret built in Task 5.2.

**Verify**
```
show isis neighbors                      ! all Up, Level-2
show isis adjacency detail               ! auth active, p2p
show running-config inheritance interface GigabitEthernet0/0/0/0   ! group-expanded config
show running-config group
show isis database verbose               ! overload bit on R7/R8/R30 at startup
```

---

### Task 1.2 — OSPF Area 0 (AS65200)

#### Example PE R14 (mirror pattern on R11–R16, R19)

```
router ospf 1
 router-id 150.2.14.14
 nsr
 timers throttle spf 50 200 5000
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
   network point-to-point
   authentication message-digest
   message-digest-key 1 md5 clear C1sco-OSPF-K1
   bfd fast-detect
   bfd minimum-interval 300
   bfd multiplier 3
  !
  interface GigabitEthernet0/0/0/1
   network point-to-point
   authentication message-digest
   message-digest-key 1 md5 clear C1sco-OSPF-K1
   bfd fast-detect
  !
 !
```

**Why**
- `network point-to-point` — no DR/BDR election on p2p core; faster convergence.
- `authentication message-digest` + `message-digest-key 1 md5` — MD5 per-interface auth on every adjacency.
- `bfd fast-detect` — ties OSPF to BFD for sub-second failure detection (interval tuned in Section 4).
- `timers throttle spf 50 200 5000` — initial 50 ms, min-hold 200 ms, max 5000 ms backoff.
- `passive enable` Loopback0 — advertise /32 without an adjacency.

**Verify**
```
show ospf neighbor                       ! FULL on all p2p links
show ospf interface GigabitEthernet0/0/0/0   ! Network type P2P, auth MD5
show bfd session
show ospf                                ! SPF throttle values
```

---

### Task 1.3 — LDP / MPLS (AS65200 core)

#### Loopback-only label filter + session protection + IGP-sync (R11–R16, R19)

```
route-policy LDP-LOOPBACKS-ONLY
  if destination in (150.2.0.0/16 ge 32 le 32) then
    pass
  else
    drop
  endif
end-policy
!
mpls ldp
 router-id 150.2.14.14
 session protection
 igp sync
 address-family ipv4
  label
   local
    allocate for LDP-LOOPBACKS-ONLY     ! advertise labels for /32 loopbacks only
   !
  !
 !
 interface GigabitEthernet0/0/0/0
 !
 interface GigabitEthernet0/0/0/1
 !
```

OSPF side of IGP-sync (already enabled instance-wide, but confirm per interface):
```
router ospf 1
 area 0
  interface GigabitEthernet0/0/0/0
   mpls ldp sync
  !
```

**Why**
- `allocate for LDP-LOOPBACKS-ONLY` — only PE/RR /32s get labels, shrinking the LFIB (LDP label filtering).
- `session protection` — keeps the LDP session (and labels) alive across brief link flaps.
- `igp sync` + `mpls ldp sync` — a link isn't used for forwarding until LDP labels are exchanged, preventing black-holes during convergence.

**Verify**
```
show mpls ldp neighbor
show mpls ldp bindings 150.2.15.15/32
show mpls forwarding
show mpls ldp igp sync
```

---

### Task 1.4 — BGP Unified MPLS (iBGP-LU across all 3 ASes)

**Design.** Seamless/Unified MPLS: PE loopbacks are carried in `ipv4 labeled-unicast`. R7/R8 reflect for the AS65100 core; R3/R4 are inline RRs; `allocate-label all` on label-stitching nodes; `next-hop-self` at every ASBR re-originates the label stack at each AS edge.

#### RR R7 (AS65100) — IOS-XR

```
router bgp 65100
 bgp router-id 150.1.7.7
 address-family ipv4 unicast
  allocate-label all
 !
 af-group LU-CLIENT address-family ipv4 labeled-unicast
  route-reflector-client
  next-hop-self
 !
 neighbor-group RRC
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   use af-group LU-CLIENT
  !
 !
 neighbor 150.1.31.31
  use neighbor-group RRC
 !
 neighbor 150.1.5.5
  use neighbor-group RRC
 !
 neighbor 150.1.8.8
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
  !
 !
```

#### Inline RR R3 (P router, AS65100)

```
router bgp 65100
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 150.1.31.31
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
  !
 !
```

#### ASBR R6 (AS65100 side) — next-hop-self for BGP-LU

```
router bgp 65100
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 150.1.7.7
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   next-hop-self
  !
 !
 neighbor 10.6.20.20
  remote-as 65300
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-self
  !
 !
```

**Why**
- `address-family ipv4 labeled-unicast` — RFC 3107, prefix + label in BGP.
- `allocate-label all` — label every IGP prefix so the LSP is continuous, not just loopbacks.
- `route-reflector-client` on R7/R8 and inline R3/R4 — scales iBGP-LU without full mesh.
- `next-hop-self` at ASBRs — rewrites next-hop to the ASBR loopback, forcing a new label imposition at each AS boundary (label stitching).

**Verify**
```
show bgp ipv4 labeled-unicast summary
show bgp ipv4 labeled-unicast 150.2.15.15/32   ! label + next-hop
show mpls forwarding
traceroute 150.3.22.22 source Loopback0        ! continuous label stack across ASes
```

---

### Task 1.5 — Segment Routing on IS-IS (AS65100, SR-MPLS)

#### Global SR + SRGB/SRLB (all R1–R8)

```
segment-routing
 global-block 16000 23999                 ! default SRGB
 local-block 25000 26000                  ! SRLB
!
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
  microloop avoidance segment-routing
 !
```

#### Prefix-SID on Loopback0 (index = last octet) — example R5

```
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 5                      ! label 16005
  !
 !
```
(R1 index 1 → 16001, R8 index 8 → 16008, etc.)

#### Anycast SID on R31 and R2 (shared Loopback99, SID 99, n-flag-clear)

```
interface Loopback99
 ipv4 address 150.1.99.99 255.255.255.255   ! SAME address on R31 and R2
!
router isis CORE
 interface Loopback99
  passive
  address-family ipv4 unicast
   prefix-sid index 99 n-flag-clear          ! anycast, NOT a node-SID
  !
 !
```

#### TI-LFA on all core interfaces (carried in the Task 1.1 interface group)
```
router isis CORE
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
  !
 !
```

**Why**
- `global-block 16000 23999` — platform-default SRGB; prefix-SID label = SRGB base + index.
- `local-block 25000 26000` — SRLB reserves adj-SID / BSID label space per node.
- `prefix-sid index N` — index-based (not absolute) so SIDs survive SRGB changes.
- `prefix-sid index 99 n-flag-clear` — same address on R1+R2 with N-flag cleared → anycast SID, traffic goes to nearest of the pair.
- `fast-reroute per-prefix ti-lfa` — topology-independent LFA, 100% coverage backup.
- `microloop avoidance segment-routing` — inserts an explicit SID path during convergence to prevent transient loops.

**Verify**
```
show isis segment-routing label table
show mpls forwarding labels 16000 16100
show isis fast-reroute 150.1.8.8/32 detail      ! TI-LFA backup present
show segment-routing local-block inconsistencies   ! should be empty
```

---

### Task 1.6 — SRv6 (AS65300, uSID / F3216)

#### SRv6 global + locator (example R22; XX = router number)

```
segment-routing
 srv6
  logging locator status
  encapsulation
   source-address 2001:db8:3::22            ! Loopback0 IPv6
  !
  locators
   locator LOC-R22
    micro-segment behavior unode psp-usd    ! F3216 uSID, End psp-usd
    prefix fc00:0300:0022::/48
    tag 200
   !
  !
 !
!
router isis CORE
 address-family ipv6 unicast
  segment-routing srv6
   locator LOC-R22
  !
 !
```
(R20 → `fc00:0300:0020::/48`, R21 → `...0021`, R23 → `...0023`, R30 → `...0030`.)

#### SRv6 services AF for Yellow L3VPN (enables VPNv4/VPNv6 over SRv6)

```
router bgp 65300
 address-family vpnv4 unicast
 !
 address-family vpnv6 unicast
 !
 ! per-VRF SRv6 allocation is set in Task 2.1
```

**Why**
- `micro-segment ... psp-usd` — F3216 uSID (16-bit micro-SIDs in a 32-bit block), End.PSP-USD behavior pops/decaps at penultimate/ultimate node.
- `prefix fc00:0300:00XX::/48` — unique locator per node; carries the uSID block.
- `tag 200` — marks locator prefixes for SPF prefix-priority FRR in Task 4.2.
- `source-address` = Lo0 IPv6 — outer IPv6 SA for SRv6-encapsulated packets.
- `segment-routing srv6 locator` under IS-IS IPv6 AF — advertises the locator (SRv6 Locator TLV) into the IGP.

**Verify**
```
show segment-routing srv6 locator
show segment-routing srv6 sid
show isis database verbose                 ! SRv6 Locator/End-SID TLVs
show cef ipv6 fc00:0300:0022::/48
show segment-routing srv6 manager
```

---

### Task 1.7 — SR-TE policy (R1 → R5)

#### On R1 (headend)

```
segment-routing
 traffic-eng
  policy R1-TO-R5
   binding-sid mpls 24333
   color 333 end-point ipv4 150.1.5.5
   candidate-paths
    preference 100
     dynamic
      metric type te
     !
    !
   !
   autoroute
    include ipv4 150.1.5.100/32           ! R5 Loopback100 rides the policy
   !
  !
 !
```

#### On R5 — advertise Lo100 with color 333

```
interface Loopback100
 ipv4 address 150.1.5.100 255.255.255.255
!
route-policy SET-COLOR-333
  set extcommunity color COLOR-333
  pass
end-policy
!
extcommunity-set opaque COLOR-333
  333
end-set
!
router bgp 65100
 address-family ipv4 unicast
  network 150.1.5.100/32 route-policy SET-COLOR-333
 !
```

#### On R1 — enable color autosteer
```
router bgp 65100
 address-family ipv4 unicast
 !
 ! color-333 routes auto-resolve onto policy R1-TO-R5 (steering is automatic once
 ! an SR-TE policy with matching color+endpoint exists)
```

**Why**
- `binding-sid mpls 24333` — single label that represents the whole policy (SID stack).
- `dynamic metric type te` — headend/PCE computes a path minimizing TE metric.
- `autoroute include` — IGP traffic to Lo100 is steered into the policy without a static route.
- `extcommunity color 333` + matching policy color/endpoint — BGP color automated steering.

**Verify**
```
show segment-routing traffic-eng policy name R1-TO-R5
show segment-routing traffic-eng policy binding-sid 24333
show bgp ipv4 unicast 150.1.5.100/32        ! color:333 ext-comm
show route 150.1.5.100/32                    ! via SR policy / BSID
```

---

### Task 1.8 — SR-ODN (R22, Blue; PCE = R30)

#### R30 — SR-PCE

```
pce
 address ipv4 150.3.30.30
!
```

#### R22 — PCC + ODN template for color 30

```
segment-routing
 traffic-eng
  pcc
   source-address ipv4 150.3.22.22
   pce address ipv4 150.3.30.30
    precedence 10
   !
   report-all
  !
  on-demand color 30
   dynamic
    pcep                                   ! delegate path computation to PCE
    metric type latency                    ! Platinum / low-latency intent
   !
  !
 !
```

Blue VRF import keeps color 30 (set at the remote PE on export); receipt of a color-30 VPN prefix triggers the ODN policy to that egress PE.

**Why**
- `pce address` on R30 — turns the RR into an SR-PCE.
- `pcc ... pce address` on R22 — PCEP session to R30; R22 delegates computation.
- `on-demand color 30 ... pcep metric latency` — a template; the policy is instantiated on-demand per color-30 destination, computed by R30 on the delay metric.

**Verify**
```
show pce ipv4 peer                           ! on R30 — R22 as PCC
show segment-routing traffic-eng pcc ipv4 peer   ! on R22 — PCE up
show segment-routing traffic-eng policy color 30
show bgp vpnv4 unicast vrf BLUE              ! color:30, ODN-resolved next-hop
```

---

### Task 1.9 — Flex-Algo (R1 ↔ R5, algos 233 red-exclude / 234 blue-exclude)

#### Affinity bit-map definitions + FAD (on R1 and R5; replicate on transit)

```
affinity-map
 name RED  bit-position 1
 name BLUE bit-position 2
!
router isis CORE
 flex-algo 233
  metric-type igp
  affinity exclude-any RED
  advertise-definition
 !
 flex-algo 234
  metric-type igp
  affinity exclude-any BLUE
  advertise-definition
 !
```

#### Apply affinity colors to core links
```
router isis CORE
 interface GigabitEthernet0/0/0/0
  affinity flex-algo RED               ! this link excluded from algo 233
 !
 interface GigabitEthernet0/0/0/1
  affinity flex-algo BLUE              ! this link excluded from algo 234
 !
```

#### Per-algo prefix-SIDs on Loopback0 (R1 shown)
```
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 1                    ! algo 0 (base)
   prefix-sid algorithm 233 index 2331
   prefix-sid algorithm 234 index 2341
  !
 !
```

#### SR policies R31→R5 constrained per algo
```
segment-routing
 traffic-eng
  policy R1-R5-ALGO233
   color 2330 end-point ipv4 150.1.5.5
   candidate-paths
    preference 100
     dynamic
      constraints
       segments
        dataplane mpls
       !
      !
      metric type igp
     !
    !
   !
   ! steer onto flex-algo 233 via per-algo prefix-SID of R5
  !
  policy R1-R5-ALGO234
   color 2340 end-point ipv4 150.1.5.5
   candidate-paths
    preference 100
     dynamic
      metric type igp
     !
    !
   !
  !
 !
```

**Why**
- `affinity-map name/bit-position` — names link colors to bit positions in the admin-group.
- `flex-algo 233 affinity exclude-any RED` — algo 233 computes SPF avoiding RED links; 234 avoids BLUE.
- `advertise-definition` — one node floods the FAD so all nodes agree on the algo constraints.
- `prefix-sid algorithm 233/234 index` — distinct SID per algorithm → distinct constrained LSP.

**Verify**
```
show isis flex-algo
show isis database verbose                   ! FAD TLV + per-algo prefix-SIDs
show route 150.1.5.5/32 algorithm 233
show segment-routing traffic-eng policy
```

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — L3VPN (VRF GREEN / YELLOW / BLUE)

**RD/RT scheme:** RD = `<PE-ASN>:<cust-id>`, RT = `*:<cust-id>` (GREEN=1, YELLOW=2, BLUE=3).

#### PE VRF definitions (IOS-XR) — example R20 (AS65300, Yellow over SRv6)

```
vrf YELLOW
 address-family ipv4 unicast
  import route-target
   65300:2
  !
  export route-target
   65300:2
  !
 !
 address-family ipv6 unicast
  import route-target
   65300:2
  !
  export route-target
   65300:2
  !
 !
!
router bgp 65300
 address-family vpnv4 unicast
 !
 address-family vpnv6 unicast
 !
 vrf YELLOW
  rd 65300:2
  address-family ipv4 unicast
   segment-routing srv6                   ! Yellow rides SRv6 (Task 1.6)
    locator LOC-R20
   !
  !
  neighbor 192.168.24.24
   remote-as 65024
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
    site-of-origin 65300:2420             ! SoO, dual-homed R24 (Task 3.1)
   !
  !
 !
```

#### PE-CE OSPF (R15 ↔ R17, Green) — R15 (IOS-XR)

```
router ospf 1
 vrf GREEN
  router-id 150.2.15.15
  domain-id 0.0.0.1
  redistribute bgp 65200
  area 0
   interface GigabitEthernet0/0/0/5
    network point-to-point
   !
  !
 !
!
router bgp 65200
 vrf GREEN
  rd 65200:1
  address-family ipv4 unicast
   redistribute ospf 1 match internal external
  !
 !
```

#### CE R24 (Yellow) — IOS-XE

```
router bgp 65024
 bgp log-neighbor-changes
 neighbor 192.168.24.20 remote-as 65300
 neighbor 192.168.24.22 remote-as 65300
 address-family ipv4
  network 172.24.24.0 mask 255.255.255.0
  neighbor 192.168.24.20 activate
  neighbor 192.168.24.22 activate
 exit-address-family
!
```

#### CE R17 (Green, OSPF) — IOS-XE

```
router ospf 1
 network 192.168.17.0 0.0.0.255 area 0
 network 172.17.17.0 0.0.0.255 area 0
!
```

**Why**
- `rd 65300:2` — makes customer prefixes globally unique as VPNv4/VPNv6 NLRI.
- import/export `65300:2` — controls which VRFs exchange routes.
- `segment-routing srv6 locator` under the VRF AF — allocates an SRv6 service SID for Yellow (SRv6 L3VPN).
- `as-override` — rewrites the CE ASN in AS_PATH so a reused ASN at multiple sites isn't rejected.
- `site-of-origin` — tags routes with origin site; blocks re-advertisement back to the same dual-homed site.
- OSPF `domain-id` + mutual redistribution — sham-link-free PE-CE OSPF with correct LSA translation.

**Verify**
```
show vrf all
show bgp vpnv4 unicast summary
show bgp vrf YELLOW
show route vrf BLUE
show ospf vrf GREEN neighbor
show segment-routing srv6 sid               ! Yellow service SID
```

---

### Task 2.2 — Inter-AS L3VPN (Option A + Option B)

#### Option A — back-to-back VRF, R5 ↔ R11 (Green), per-VRF subinterface over 10.5.11.0/24

R5 (IOS-XR, AS65100):
```
interface GigabitEthernet0/0/0/9.100
 vrf GREEN
 ipv4 address 10.5.11.5 255.255.255.0
 encapsulation dot1q 100
!
router bgp 65100
 vrf GREEN
  rd 65100:1
  address-family ipv4 unicast
  !
  neighbor 10.5.11.11
   remote-as 65200
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
```
R11 mirrors with `vrf GREEN`, `remote-as 65100`, address `10.5.11.11`. Each ASBR treats the other as a plain PE-CE eBGP peer inside the VRF — no VPN labels cross the link.

#### Option B — VPNv4/VPNv6 eBGP + next-hop-self, R12 ↔ R21 and R6 ↔ R20

R12 (IOS-XR, AS65200 side) over 20.12.21.0/24:
```
router bgp 65200
 address-family vpnv4 unicast
  retain route-target all                  ! keep VPN routes w/o locally-configured RT
 !
 address-family vpnv6 unicast
  retain route-target all
 !
 neighbor 20.12.21.21
  remote-as 65300
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-self
  !
  address-family vpnv6 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-self
  !
 !
```
R21/R6/R20 mirror this pattern.

**Why**
- Option A: no MPLS on the inter-AS link; simplest, but one VRF subinterface per customer (scaling cost).
- Option B: ASBRs exchange *labeled VPN* routes directly (no per-VRF subinterface).
- `retain route-target all` — ASBR normally drops VPN routes whose RT isn't imported locally; this keeps them for transit.
- `next-hop-self` — ASBR becomes the next-hop and imposes a new VPN label, stitching the LSP across the boundary.

**Verify**
```
show bgp vpnv4 unicast rd 65300:2
show bgp vpnv6 unicast
show route vrf GREEN                         ! Option A, both ASBRs
show bgp vpnv4 unicast neighbor 20.12.21.21 received-routes   ! Option B
```

---

### Task 2.3 — L2VPN AToM / VPWS (Red, R14 ↔ R16, VC-ID 100)

#### R14 (IOS-XR)

```
interface GigabitEthernet0/0/0/7.100 l2transport
 encapsulation dot1q 100
!
l2vpn
 pw-class ATOM-LDP
  encapsulation mpls
   transport-mode ethernet
  !
 !
 xconnect group RED
  p2p RED-ATOM-100
   interface GigabitEthernet0/0/0/7.100
   neighbor ipv4 150.2.16.16 pw-id 100     ! VC-ID 100, targeted LDP to R16
    pw-class ATOM-LDP
   !
  !
 !
```
R16 mirrors with `neighbor ipv4 150.2.14.14 pw-id 100`.

**Why**
- `l2transport` subinterface + `encapsulation dot1q 100` — VLAN100 attachment circuit.
- `pw-id 100` — the VC-ID; must match both ends.
- `encapsulation mpls` pw-class — AToM pseudowire signaled by targeted LDP between PE loopbacks.

**Verify**
```
show l2vpn xconnect
show l2vpn xconnect detail                   ! state UP, VC-ID 100
show mpls ldp neighbor 150.2.16.16           ! targeted session
```

---

### Task 2.4 — EVPN-VPWS (Red, R14 ↔ R16, EVI 400, AC 12)

#### R14 (IOS-XR)

```
interface GigabitEthernet0/0/0/8.100 l2transport
 encapsulation dot1q 100
!
l2vpn
 xconnect group RED-EVPN
  p2p RED-EVPN-400
   interface GigabitEthernet0/0/0/8.100
   neighbor evpn evi 400 target 16 source 12   ! local AC 12 → remote AC 16
  !
 !
!
router bgp 65200
 address-family l2vpn evpn
 !
 neighbor 150.2.19.19
  remote-as 65200
  update-source Loopback0
  address-family l2vpn evpn
  !
 !
```
R16 mirrors: `neighbor evpn evi 400 target 12 source 16`.

#### EVPN multipoint (ELAN) for Red across R27/R28/R29 — bridge-domain example (R14)

```
l2vpn
 bridge group RED
  bridge-domain RED-ELAN
   interface GigabitEthernet0/0/0/9.100
   evi 401
  !
 !
```

**Why**
- `neighbor evpn evi 400 target/source` — EVPN-VPWS P2P service; AC IDs identify the local/remote endpoints (local 12).
- `address-family l2vpn evpn` in BGP — EVPN NLRI (per-EVI Ethernet A-D routes) auto-discover the PWs via the RR (R19).
- `bridge-domain ... evi 401` — multipoint EVPN ELAN; contrasts flooding/MAC-learning vs P2P VPWS.

**Verify**
```
show evpn evi
show evpn evi vpn-id 400 detail
show l2vpn xconnect group RED-EVPN detail
show bgp l2vpn evpn
```

---

### Task 2.5 — Multicast VPN (mVPN Profile 0, Yellow, AS65300)

#### PE example R22 (IOS-XR)

```
multicast-routing
 address-family ipv4
  interface all enable
 !
 vrf YELLOW
  address-family ipv4
   mdt default ipv4 239.232.0.1
   mdt data 239.232.100.0/24 threshold 10
   interface all enable
  !
 !
!
router pim
 address-family ipv4
  ! SSM default range covers 239.232.0.0/16
 !
 vrf YELLOW
  address-family ipv4
   rpf topology route-policy MVPN-RPF
  !
 !
!
router bgp 65300
 address-family ipv4 mvpn
 !
 vrf YELLOW
  address-family ipv4 mvpn
  !
 !
```

**Why**
- `mdt default ipv4 239.232.0.1` — default MDT group all Yellow PEs join (control + low-rate traffic).
- `mdt data 239.232.100.0/24 threshold 10` — high-rate (S,G) streams move to a data MDT above threshold (saves bandwidth to uninterested PEs).
- `address-family ipv4 mvpn` in BGP — MVPN SAFI auto-discovery (Profile 0) via R30.
- Core PIM-SSM on 239.232.0.0/16 — no RP needed; (S,G) joins only.

**Verify**
```
show mrib vrf YELLOW route
show pim vrf YELLOW neighbor
show mvpn vrf YELLOW database
show bgp ipv4 mvpn
show mfib vrf YELLOW                          ! data-MDT switchover
```

---

### Task 2.6 — Hierarchical QoS (egress on R6 ASBR)

```
class-map match-any VOICE
 match dscp ef
end-class-map
!
class-map match-any CE-POLICED
 match access-group ipv4 CE-TRAFFIC
end-class-map
!
class-map match-any SCAVENGER
 match dscp cs1
end-class-map
!
policy-map CHILD
 class VOICE
  priority level 1
  police rate percent 20
 !
 class CE-POLICED
  police rate percent 50
 !
 class SCAVENGER
  set dscp default                            ! remark to BE
 !
 class class-default
  bandwidth remaining percent 100
 !
end-policy-map
!
policy-map PARENT-HQOS
 class class-default
  shape average percent 80                    ! parent shaper
  service-policy CHILD                        ! nested child
 !
end-policy-map
!
interface GigabitEthernet0/0/0/0
 service-policy output PARENT-HQOS
!
```

**Why**
- Parent `shape average percent 80` caps egress to 80% of line rate.
- Child `VOICE priority level 1` + `police percent 20` — EF gets strict-priority but capped at 20%.
- `CE-POLICED police percent 50` — rate-limits a specific CE class to 50%.
- `SCAVENGER set dscp default` — DSCP remark down to best-effort.
- Nesting child under parent `class-default` = two-level H-QoS (shape → classes).

**Verify**
```
show policy-map interface GigabitEthernet0/0/0/0 output
show qos interface GigabitEthernet0/0/0/0 output
```

---

## Section 3 — Access Connectivity (10%)

### Task 3.1 — SoO loop prevention (dual-homed Yellow CE R24)

Same SoO value on **both** R20 and R22 PE-CE sessions (already shown on R20 in Task 2.1):

R22 (IOS-XR):
```
router bgp 65300
 vrf YELLOW
  neighbor 192.168.24.22
   remote-as 65024
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
    site-of-origin 65300:2420              ! SAME SoO as R20
   !
  !
 !
```

**Why** — a route originated at the R24 site is tagged SoO `65300:2420` by the ingress PE. When the other PE would re-advertise it back toward the same site, the SoO match blocks it, breaking the loop. `as-override` and SoO coexist: as-override fixes AS_PATH acceptance, SoO fixes the loop.

**Verify**
```
show bgp vrf YELLOW 172.24.24.0/24 detail     ! SoO:65300:2420 present
! confirm R24 does not receive its own prefix back from the second PE
```

---

### Task 3.2 — 6PE / 6VPE

#### 6PE — IPv6 over the IPv4 MPLS core (AS65100), example R1

```
router bgp 65100
 address-family ipv6 labeled-unicast
 !
 neighbor 150.1.7.7
  remote-as 65100
  update-source Loopback0
  address-family ipv6 labeled-unicast     ! 6PE: IPv6 NLRI + label, next-hop ::ffff:v4
  !
 !
```

#### 6VPE — IPv6 L3VPN for Blue, example R23 (IOS-XR)

```
vrf BLUE
 address-family ipv6 unicast
  import route-target
   65300:3
  !
  export route-target
   65300:3
  !
 !
!
router bgp 65300
 address-family vpnv6 unicast
 !
 vrf BLUE
  rd 65300:3
  address-family ipv6 unicast
  !
  neighbor 2001:db8:ce:32::32
   remote-as 65025
   address-family ipv6 unicast
    route-policy PASS in
    route-policy PASS out
    as-override
   !
  !
 !
```

CE R32 (Blue, IOS-XE):
```
router bgp 65025
 address-family ipv6
  network 2001:DB8:3232::/48
  neighbor 2001:DB8:CE:32::23 remote-as 65300
  neighbor 2001:DB8:CE:32::23 activate
 exit-address-family
!
```

**Why**
- `ipv6 labeled-unicast` (6PE) — IPv6 prefixes carried with a label over the existing IPv4 LSP; next-hop is IPv4-mapped IPv6.
- `vpnv6 unicast` + VRF IPv6 AF (6VPE) — IPv6 L3VPN; same RD/RT model as IPv4.

**Verify**
```
show bgp ipv6 labeled-unicast
show bgp vpnv6 unicast vrf BLUE
show route vrf BLUE ipv6
```

---

## Section 4 — High Availability (10%)

### Task 4.1 — BFD for eBGP (R6 ↔ R20)

R6 (IOS-XR, AS65100) over 10.6.20.0/24:
```
router bgp 65100
 neighbor 10.6.20.20
  remote-as 65300
  bfd fast-detect
  bfd minimum-interval 800
  bfd multiplier 4
  address-family ipv4 labeled-unicast
   next-hop-self
  !
 !
```
R20 mirrors with `neighbor 10.6.20.6` / `remote-as 65100`.

**Why**
- `bfd fast-detect` — BGP tracks the BFD session; neighbor is torn down the instant BFD declares the link down (far faster than BGP hold-timer).
- `minimum-interval 800` + `multiplier 4` — detection time = 800 ms × 4 = 3.2 s (per task spec).

**Verify**
```
show bfd session detail                       ! interval 800ms, mult 4, state UP
show bgp neighbor 10.6.20.20                   ! BFD enabled
! fail the link → BGP tears down within the BFD detect window
```

---

### Task 4.2 — SRv6 backup paths / prefix-priority (AS65300)

On all AS65300 IS-IS routers:
```
router isis CORE
 address-family ipv6 unicast
  spf prefix-priority critical tag 200          ! locator prefixes (tag 200) install first
  fast-reroute per-prefix priority-limit critical
 !
```

**Why**
- `spf prefix-priority critical tag 200` — locator prefixes tagged 200 (Task 1.6) are placed in the critical priority queue and installed into RIB/FIB ahead of everything else after an SPF.
- `fast-reroute per-prefix priority-limit critical` — TI-LFA backups are computed for critical-priority prefixes first, so SRv6 transport converges before customer prefixes.

**Verify**
```
show isis fast-reroute summary
show route ipv6 fc00:0300:0022::/48 detail     ! backup path present
show isis spf-log                              ! tag-200 prefixes prioritized
```

---

## Section 5 — Security (10%)

### Task 5.1 — Management Plane Protection (MPP, inband)

```
ipv4 access-list MGMT-HOSTS
 10 permit ipv4 10.255.0.0/24 any
!
control-plane
 management-plane
  inband
   interface GigabitEthernet0/0/0/0
    allow SSH peer
     address ipv4 10.255.0.0/24
    !
    allow SNMP peer
     address ipv4 10.255.0.0/24
    !
   !
  !
 !
!
```
Telnet, XML, and TFTP are simply **not** listed under `allow`, so MPP denies them inband by default.

**Why**
- `management-plane inband` — restricts which protocols reach the control plane over data interfaces.
- `allow SSH/SNMP peer address` — only the 10.255.0.0/24 management subnet is permitted.
- Omitting telnet/XML/TFTP = implicit deny for those management protocols.

**Verify**
```
show management-plane inband interface all     ! SSH/SNMP allowed, others absent
```

---

### Task 5.2 — IS-IS authentication (keychain HMAC-MD5)

Completes the `ISIS-KC` keychain referenced by the Task 1.1 groups (all AS65100 + AS65300 IS-IS routers):
```
key chain ISIS-KC
 key 1
  accept-lifetime 00:00:00 january 01 2024 infinite
  key-string password C1sco-ISIS-Secret
  send-lifetime 00:00:00 january 01 2024 infinite
  cryptographic-algorithm HMAC-MD5
 !
!
```
Hello + LSP auth are already applied by the `ISIS-AUTH-COMMON` group (`hello-password keychain ISIS-KC` / `lsp-password keychain ISIS-KC`) — no per-interface config needed.

**Why**
- `cryptographic-algorithm HMAC-MD5` — keyed-hash auth on hellos and LSPs.
- Keychain accept/send lifetimes allow hitless key rollover.
- Applying via the flex-CLI group keeps it uniform and avoids per-interface drift.

**Verify**
```
show key chain ISIS-KC
show isis adjacency detail                     ! authentication active
show isis database verbose                     ! LSP auth TLV
! mismatch a key → adjacency drops (confirms enforcement)
```

---

### Task 5.3 — BGP security

Applied to all neighbors (example composite):
```
router bgp 65100
 neighbor 10.6.20.20
  remote-as 65300
  password encrypted <hash>                     ! MD5/TCP-AO per session
  ttl-security hops 1                            ! GTSM on eBGP
  address-family ipv4 labeled-unicast
   maximum-prefix 1000 80                        ! limit + 80% warn threshold
  !
 !
 neighbor 150.1.7.7
  remote-as 65100
  password encrypted <hash>                      ! iBGP MD5 too
 !
!
! PE-CE example (VRF)
router bgp 65300
 vrf YELLOW
  neighbor 192.168.24.24
   remote-as 65024
   password encrypted <hash>
   ttl-security hops 1
   address-family ipv4 unicast
    maximum-prefix 500 75
   !
  !
 !
```

**Why**
- `password` — authenticates the TCP session (prevents spoofed BGP).
- `ttl-security hops 1` — GTSM; packets must arrive with TTL ≥ 254, defeating off-link spoofing on eBGP.
- `maximum-prefix N pct` — caps routes from a peer with an early warning, protecting RIB/FIB.

**Verify**
```
show bgp neighbor 10.6.20.20                    ! password set, TTL security, max-prefix
! over-limit or wrong-TTL peer is rejected
```

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry (gRPC dial-out, R22)

```
telemetry model-driven
 destination-group DG-COLLECTOR
  address-family ipv4 10.255.0.50 port 57500
   encoding self-describing-gpb
   protocol grpc tls
  !
 !
 sensor-group SG-INTF
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription SUB-INTF
  sensor-group-id SG-INTF sample-interval 30000   ! 30 s
  destination-id DG-COLLECTOR
 !
!
grpc
 port 57400
 tls-trustpoint TP-TELEMETRY
 address-family ipv4
!
tpa
 vrf default
  address-family ipv4
   update-source Loopback0                        ! TPA source/management address
  !
 !
!
```

**Why**
- `destination-group ... protocol grpc tls` — dial-out: the router pushes samples to the collector over TLS.
- `sensor-group` sensor-path — selects the interface-statistics YANG model.
- `subscription ... sample-interval 30000` — streams every 30 s.
- `grpc tls-trustpoint` — certificate-secured transport.
- `tpa ... update-source Loopback0` — Third-Party-App transport source address.

**Verify**
```
show telemetry model-driven subscription SUB-INTF
show telemetry model-driven destination
show grpc
```

---

### Task 6.2 — NSO service deployment (Red L3VPN on R14)

NSO CLI (ncs_cli -C):
```
! 1. Onboard the device
config
 devices authgroups group XR-AUTH umap admin remote-name admin remote-password C1sco123
 devices device R14
  address 150.2.14.14
  port 830
  authgroup XR-AUTH
  device-type cli ned-id cisco-iosxr-cli-7.x      ! (or netconf)
  state admin-state unlocked
 commit
!
! 2. Sync-from
devices device R14 sync-from
!
! 3. XR-base template
config
 devices device R14 template XR-BASE
 commit
!
! 4. Red L3VPN service instance
config
 services l3vpn RED
  customer RED
  endpoint R14
   pe-device R14
   vrf-rd 65200:3
   rt 65200:3
   ce-interface GigabitEthernet0/0/0/6
  !
 commit dry-run                                    ! preview device config
 commit
!
```

**Why**
- `authgroups` + `device device ... device-type` — registers R14 with credentials and NED.
- `sync-from` — pulls the live config into NSO's CDB so the service maps cleanly.
- template `XR-BASE` — pushes a baseline (logging, NTP, etc.).
- `services l3vpn RED` — a service-model instance; NSO's FASTMAP renders it into device CLI (VRF, RD/RT, PE-CE) and pushes it.

**Verify (NSO CLI)**
```
show devices device R14 sync-from
commit dry-run                                     ! service → device diff
show running-config services l3vpn RED
show running-config devices device R14 config      ! VRF RED present on R14
```

---

## Completion Checklist

| Section | Weight | Tasks (solution provided) |
|---------|--------|---------------------------|
| 1 — Core Network Infrastructure | 25% | 1.1 IS-IS flex-CLI · 1.2 OSPF · 1.3 LDP · 1.4 BGP-LU · 1.5 SR-MPLS · 1.6 SRv6 · 1.7 SR-TE · 1.8 SR-ODN · 1.9 Flex-Algo |
| 2 — Architecture & Services | 25% | 2.1 L3VPN · 2.2 Inter-AS A+B · 2.3 AToM/VPWS · 2.4 EVPN-VPWS · 2.5 mVPN · 2.6 H-QoS |
| 3 — Access Connectivity | 10% | 3.1 SoO · 3.2 6PE/6VPE |
| 4 — High Availability | 10% | 4.1 BFD · 4.2 SRv6 prefix-priority FRR |
| 5 — Security | 10% | 5.1 MPP · 5.2 IS-IS keychain · 5.3 BGP security |
| 6 — Automation | 5% | 6.1 gRPC telemetry · 6.2 NSO |

> **Notes:** Interface IDs, dot1q tags, and management subnets (10.255.0.0/24) are representative — adapt to your EVE-NG cabling. CE snippets use IOS-XE; all SP/RR/PCE snippets use IOS-XR. `route-policy PASS` is a pass-all policy assumed defined on every XR node (`route-policy PASS / pass / end-policy`).
