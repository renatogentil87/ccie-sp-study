# WB09 — Inter-AS L3VPN Solutions (Options A / B / C)

**Platform:** Cisco 7200, IOS 15.2(4)M11 — **IOS Classic syntax**
**Scope:** Inter-AS MPLS L3VPN between AS 65100, AS 65200, AS 65300
**Prerequisite:** WB00–WB07 complete (intra-AS L3VPN working in each AS, MP-iBGP VPNv4 up to the RRs, LDP in each core)

---

## 1. Which option goes where

| Option | ASBR pair | Customer | Technique |
|--------|-----------|----------|-----------|
| **A** | R5 ↔ R11 | Green | Back-to-back VRF, sub-interface per VRF, eBGP (IPv4) inside the VRF |
| **B** | R12 ↔ R21 | Yellow | eBGP **VPNv4** ASBR↔ASBR, `next-hop-self`, `retain route-target all` |
| **C** | R6 ↔ R20 | Blue (spans 3 ASes) | BGP **labeled-unicast** (IPv4+label) ASBR↔ASBR + multihop eBGP VPNv4 RR↔RR, `next-hop-unchanged` |

Inter-AS physical links (from topology):

| Link | A int/IP | B int/IP |
|------|----------|----------|
| R5↔R11 | g1/0 10.5.11.5 | g1/0 10.5.11.11 |
| R6↔R12 | g1/0 10.6.12.6 | g1/0 10.6.12.12 |
| R6↔R20 | f0/0 10.6.20.6 | f0/0 10.6.20.20 |
| R12↔R21 | f3/0 20.12.21.12 | f3/0 20.12.21.21 |

RRs: AS65100 = R7/R8, AS65200 = R19, AS65300 = R30.

---

## 2. OPTION A — Back-to-back VRF (R5 ↔ R11, Green)

Each ASBR treats the other as a **CE**: one sub-interface per VRF, plain eBGP IPv4 inside the VRF. No MPLS on the inter-AS link. Simple, scales poorly (one sub-if per VRF).

### 2.1 R5 (AS 65100 ASBR)

```
ip vrf GREEN
 rd 65100:910
 route-target export 65100:910
 route-target import 65100:910
!
interface GigabitEthernet1/0
 description R5->R11 inter-AS (dot1q trunk for per-VRF sub-ifs)
 no ip address
!
interface GigabitEthernet1/0.910
 description Option-A VRF GREEN to R11
 encapsulation dot1Q 910
 ip vrf forwarding GREEN
 ip address 10.5.11.5 255.255.255.0
!
router bgp 65100
 !
 address-family ipv4 vrf GREEN
  neighbor 10.5.11.11 remote-as 65200
  neighbor 10.5.11.11 activate
 exit-address-family
```

> The VRF GREEN here must already be redistributing/importing the Green customer prefixes from the intra-AS VPNv4 fabric (configured in earlier WBs). The eBGP session into R11 re-advertises those as plain IPv4 inside the VRF.

### 2.2 R11 (AS 65200 ASBR side)

```
ip vrf GREEN
 rd 65200:910
 route-target export 65200:910
 route-target import 65200:910
!
interface GigabitEthernet1/0.910
 description Option-A VRF GREEN to R5
 encapsulation dot1Q 910
 ip vrf forwarding GREEN
 ip address 10.5.11.11 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf GREEN
  neighbor 10.5.11.5 remote-as 65100
  neighbor 10.5.11.5 activate
 exit-address-family
```

> **Note:** topology lists R11 as a P router; for Option A it acts as the AS65200 ASBR/PE for VRF GREEN. If you prefer to keep R12 as the only 65200 ASBR, move this VRF/sub-if config to R12 and point the sub-if at R5. Config pattern is identical.

### 2.3 Option A verification

```
show ip vrf GREEN
show ip route vrf GREEN
show bgp vpnv4 unicast vrf GREEN
show ip bgp vpnv4 vrf GREEN neighbors 10.5.11.11
show ip cef vrf GREEN 150.18.18.18     ! Green CE behind the other AS
```
Expect eBGP session up in VRF GREEN, remote-AS prefixes in the VRF table, data-plane is plain IP across the inter-AS link (no label).

---

## 3. OPTION B — eBGP VPNv4 ASBR↔ASBR (R12 ↔ R21, Yellow)

ASBRs exchange **labeled VPNv4** routes over a single eBGP session. No per-VRF sub-interfaces. The ASBR does **not** need the VRFs locally, but must keep RTs it doesn't import → `no bgp default route-target filter` or `retain route-target all` on the VPNv4 neighbor.

### 3.1 R12 (AS 65200 ASBR)

```
! LDP/MPLS already on core-facing links. Enable MPLS toward R21:
interface FastEthernet3/0
 description R12->R21 inter-AS VPNv4
 ip address 20.12.21.12 255.255.255.0
 mpls bgp forwarding            ! allow BGP to install labels on this eBGP link
!
router bgp 65200
 neighbor 20.12.21.21 remote-as 65300
 !
 address-family vpnv4
  neighbor 20.12.21.21 activate
  neighbor 20.12.21.21 send-community extended
  neighbor 20.12.21.21 next-hop-self
 exit-address-family
 !
 ! Keep all RTs even for VRFs not locally configured on the ASBR:
 bgp default route-target filter
```

To **retain all route-targets** (so the ASBR holds Yellow routes even without the Yellow VRF locally), use the modern form on the neighbor:

```
router bgp 65200
 address-family vpnv4
  neighbor 20.12.21.21 route-map ... 
  no bgp default route-target filter     ! simplest: disable RT filtering globally
```

Or the per-neighbor retain knob (image dependent):
```
 address-family vpnv4
  neighbor 20.12.21.21 retain route-target all
```

### 3.2 R21 (AS 65300 ASBR)

```
interface FastEthernet3/0
 description R21->R12 inter-AS VPNv4
 ip address 20.12.21.21 255.255.255.0
 mpls bgp forwarding
!
router bgp 65300
 neighbor 20.12.21.12 remote-as 65200
 !
 address-family vpnv4
  neighbor 20.12.21.12 activate
  neighbor 20.12.21.12 send-community extended
  neighbor 20.12.21.12 next-hop-self
  neighbor 20.12.21.12 retain route-target all
 exit-address-family
```

Key points:
- `send-community extended` — RTs must cross the eBGP session.
- `next-hop-self` — the ASBR becomes the BGP next-hop so the far AS resolves it via its own IGP/LDP; this makes the ASBR swap the VPN label and bind it to its own interior label.
- `retain route-target all` (or `no bgp default route-target filter`) — without a local VRF importing Yellow's RT, the ASBR would discard those VPNv4 routes. This keeps them.
- `mpls bgp forwarding` on the inter-AS interface — lets labeled VPNv4 be forwarded on a link that has no LDP.

### 3.3 Option B verification

```
show bgp vpnv4 unicast all summary
show bgp vpnv4 unicast all neighbors 20.12.21.21
show bgp vpnv4 unicast all 150.26.26.26        ! Yellow prefix (R26 behind AS65200) seen on R21
show mpls forwarding-table
show ip bgp vpnv4 all labels
```
Expect the VPNv4 eBGP session up, Yellow prefixes with RT 65xxx:24 crossing, and a VPN label stack present end-to-end.

---

## 4. OPTION C — Multihop VPNv4 RR↔RR + labeled IPv4 ASBRs (R6 ↔ R20, Blue)

Option C separates **label distribution** (ASBRs exchange IPv4+label for the PE/RR loopbacks via BGP labeled-unicast) from **VPNv4 route distribution** (multihop eBGP VPNv4 directly between RRs, next-hop unchanged). This is the most scalable and the one that fits Blue (3 ASes).

Players:
- **ASBRs:** R6 (AS65100) ↔ R20 (AS65300) — BGP labeled-unicast for loopback reachability + label.
- **RRs:** R7/R8 (AS65100) ↔ R30 (AS65300) — multihop eBGP **VPNv4**, `next-hop-unchanged`.

### 4.1 ASBR R6 — labeled-unicast (AS 65100)

```
interface FastEthernet0/0
 description R6->R20 inter-AS Option-C
 ip address 10.6.20.6 255.255.255.0
 mpls bgp forwarding
!
router bgp 65100
 neighbor 10.6.20.20 remote-as 65300
 !
 address-family ipv4
  neighbor 10.6.20.20 activate
  neighbor 10.6.20.20 send-label              ! BGP labeled-unicast (IPv4+label)
  ! Advertise our RR/PE loopbacks (next-hop reachability for the far AS):
  network 150.1.7.7 mask 255.255.255.255
  network 150.1.8.8 mask 255.255.255.255
  network 150.1.6.6 mask 255.255.255.255
 exit-address-family
```

> `send-label` turns the IPv4 session into labeled-unicast so the remote AS learns our RR loopbacks **with a label** → an end-to-end LSP for the multihop VPNv4 next-hop. Redistribute from IGP instead of static `network` statements if you prefer (`redistribute isis level-2 route-map LOOPBACKS`).

### 4.2 ASBR R20 — labeled-unicast (AS 65300)

```
interface FastEthernet0/0
 description R20->R6 inter-AS Option-C
 ip address 10.6.20.20 255.255.255.0
 mpls bgp forwarding
!
router bgp 65300
 neighbor 10.6.20.6 remote-as 65100
 !
 address-family ipv4
  neighbor 10.6.20.6 activate
  neighbor 10.6.20.6 send-label
  network 150.3.30.30 mask 255.255.255.255
  network 150.3.20.20 mask 255.255.255.255
 exit-address-family
```

### 4.3 RR R7 — multihop eBGP VPNv4 to R30 (AS 65100 side)

```
router bgp 65100
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 ebgp-multihop 255
 neighbor 150.3.30.30 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.3.30.30 activate
  neighbor 150.3.30.30 send-community extended
  neighbor 150.3.30.30 next-hop-unchanged      ! keep originating PE as next-hop
 exit-address-family
```

### 4.4 RR R30 — multihop eBGP VPNv4 to R7/R8 (AS 65300 side)

```
router bgp 65300
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 ebgp-multihop 255
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 ebgp-multihop 255
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community extended
  neighbor 150.1.7.7 next-hop-unchanged
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community extended
  neighbor 150.1.8.8 next-hop-unchanged
 exit-address-family
```

Key points:
- RR↔RR session is **multihop** (loopback to loopback across the ASBRs) — reachability provided by the labeled-unicast loopbacks in 4.1/4.2.
- `next-hop-unchanged` — the originating PE loopback stays as the VPNv4 next-hop end to end; the ingress PE in the remote AS resolves it via the BGP-LU LSP.
- `update-source Loopback0` on both sides — the multihop session must source from the loopback that the far AS learned (with label).
- For 3-AS Blue, AS65200 participates with its own Option-C relationship (R12/R21 BGP-LU + R19↔R30 or R19↔R7 multihop VPNv4) so Blue (R32 in 65100, R25 in 65200+65300) stitches across all three.

### 4.5 Option C verification

```
show bgp ipv4 unicast labels                 ! loopbacks learned WITH labels across ASBRs
show bgp vpnv4 unicast all summary           ! multihop RR-RR VPNv4 session UP
show bgp vpnv4 unicast all neighbors 150.3.30.30
show bgp vpnv4 unicast all 150.3.21.21
show mpls forwarding-table
show ip cef 150.1.7.7 detail                 ! on R20: next-hop reachable via labeled LSP
traceroute vrf BLUE 150.25.25.25             ! end-to-end across 3 ASes
```

---

## 5. Cross-AS OSPF VPN for R17 Green site (after inter-AS is working)

R17 is a Green customer in AS65200 running **OSPF PE-CE** on R15 (int f0/0, 192.168.6.15, OSPF Area 0). Once the inter-AS path carries Green's VPNv4 across AS boundaries, redistribute the inter-AS-learned BGP routes **into the OSPF VRF process** toward R17 — with a filter so you don't leak the whole table.

### 5.1 R15 (AS 65200 PE for Green/OSPF)

```
ip vrf GREEN
 rd 65200:910
 route-target export 65200:910
 route-target import 65200:910
!
router ospf 10 vrf GREEN
 router-id 150.2.15.15
 domain-id 0.0.0.10
 redistribute bgp 65200 subnets route-map BGP-TO-OSPF-GREEN
 network 192.168.6.15 0.0.0.0 area 0
!
router bgp 65200
 address-family ipv4 vrf GREEN
  redistribute ospf 10 vrf GREEN match internal external 1 external 2
 exit-address-family
!
! Filter: only allow the far-AS Green prefixes you intend to advertise to R17
ip prefix-list GREEN-ALLOW seq 5 permit 150.9.9.9/32
ip prefix-list GREEN-ALLOW seq 10 permit 150.10.10.10/32
ip prefix-list GREEN-ALLOW seq 15 permit 192.168.1.0/24
ip prefix-list GREEN-ALLOW seq 20 permit 192.168.2.0/24
!
route-map BGP-TO-OSPF-GREEN permit 10
 match ip address prefix-list GREEN-ALLOW
 set metric 100
 set metric-type type-2
```

Key points:
- `domain-id` keeps OSPF routes as inter-area (O IA) rather than external when the same OSPF domain spans PEs — set a consistent domain-id for the Green OSPF domain.
- The `route-map BGP-TO-OSPF-GREEN` filter prevents redistributing non-Green or unwanted inter-AS prefixes into R17's OSPF.
- Mutual redistribution: BGP→OSPF (to R17) and OSPF→BGP (into VPNv4). Guard against loops with the OSPF `down bit`/`domain-tag` (automatic on IOS when `router ospf … vrf`).

### 5.2 Verification

```
show ip route vrf GREEN ospf
show ip ospf 10 vrf GREEN database
show bgp vpnv4 unicast vrf GREEN
show ip route vrf GREEN 150.9.9.9       ! far-AS Green prefix reachable at R17 side
```
From R17: `show ip route` should show the remote-AS Green prefixes as O E2 (or O IA with matching domain-id), and only those permitted by `GREEN-ALLOW`.

---

## 6. Summary cheat-sheet

| | Inter-AS link MPLS? | VPNv4 session | Next-hop treatment | RT retention |
|---|---|---|---|---|
| **A** | No | none (IPv4 in VRF) | n/a | per-VRF import on ASBR |
| **B** | `mpls bgp forwarding` | eBGP ASBR↔ASBR | `next-hop-self` | `retain route-target all` |
| **C** | `mpls bgp forwarding` + `send-label` | multihop eBGP RR↔RR | `next-hop-unchanged` | on RRs normally |
