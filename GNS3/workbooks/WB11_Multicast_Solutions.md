# WB11 — Multicast Solutions (AS 65100 + mVPN)

**Platform:** Cisco 7200, IOS 15.2(4)M11 — **IOS Classic syntax**
**Scope:** PIM-SM core multicast in AS 65100, SSM, MSDP anycast/inter-domain, and mVPN Profile 0 (default MDT) for VRF GREEN
**Prerequisite:** WB00–WB07 complete (IGP + MPLS L3VPN operational)

---

## 1. Design

- **PIM sparse-mode** everywhere in the AS 65100 core (R1–R8).
- **RP = R7** (150.1.7.7), statically configured on all routers.
- **SSM** range 232.0.0.0/8 for source-specific delivery.
- **MSDP** between R7 (AS65100 RP) and R19 (AS65200 RR/RP, 150.2.19.19) to share active-source (SA) info across domains.
- **mVPN Profile 0** (classic default-MDT with PIM in the core) for VRF GREEN: default MDT 239.1.1.1, data MDT range 239.1.2.0/24.

Core interfaces (AS 65100) that need PIM — from topology:

| Router | PIM interfaces |
|--------|----------------|
| R1 | f0/0(→R3), f4/0(→R2), Lo0 |
| R2 | f3/0(→R1), f0/0(→R4), Lo0 |
| R3 | f0/0(→R1), g1/0(→R4), g2/0(→R5), f3/0(→R7), Lo0 |
| R4 | f0/0(→R2), g1/0(→R3), g2/0(→R6), Lo0 |
| R5 | g2/0(→R3), f0/0(→R8), f3/0(→R6), Lo0 |
| R6 | g2/0(→R4), f3/0(→R5), Lo0 |
| R7 | f3/0(→R3), Lo0 |
| R8 | f0/0(→R5), Lo0 |

---

## 2. Global multicast + RP + SSM (ALL core routers R1–R8)

```
configure terminal
 ip multicast-routing
!
! Static RP = R7 loopback for ASM groups
 ip pim rp-address 150.1.7.7
!
! SSM range (232.0.0.0/8) bound to ACL 1
 ip pim ssm range 1
 access-list 1 permit 232.0.0.0 0.255.255.255
```

> `ip pim rp-address 150.1.7.7` makes R7 the RP for all ASM (224/4 minus SSM) groups. SSM groups (232/8) bypass the RP entirely and build (S,G) directly.

### 2.1 Enable PIM sparse-mode on every core interface

Apply `ip pim sparse-mode` to each listed interface **plus Loopback0** on every router. Example per router:

**R1**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
interface FastEthernet4/0
 ip pim sparse-mode
```

**R2**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
```

**R3**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
interface GigabitEthernet1/0
 ip pim sparse-mode
interface GigabitEthernet2/0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
```

**R4**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
interface GigabitEthernet1/0
 ip pim sparse-mode
interface GigabitEthernet2/0
 ip pim sparse-mode
```

**R5**
```
interface Loopback0
 ip pim sparse-mode
interface GigabitEthernet2/0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
```

**R6**
```
interface Loopback0
 ip pim sparse-mode
interface GigabitEthernet2/0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
```

**R7 (RP)**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
```

**R8**
```
interface Loopback0
 ip pim sparse-mode
interface FastEthernet0/0
 ip pim sparse-mode
```

> Best practice: also configure a loopback as the RP's identity and (optionally) use `ip pim rp-address 150.1.7.7` with an ACL to scope which groups R7 is RP for. Here it's RP for all ASM groups.

---

## 3. MSDP (inter-domain source sharing: R7 ↔ R19)

MSDP lets R7 (AS65100 RP) learn active sources registered to R19 (AS65200, 150.2.19.19) and vice-versa. This is the inter-domain SA exchange that complements inter-AS PIM.

### 3.1 R7 (AS 65100 RP)

```
ip msdp peer 150.2.19.19 connect-source Loopback0
ip msdp originator-id Loopback0
!
! Optional SA filter to limit which (S,G) are advertised/accepted:
ip msdp sa-filter out 150.2.19.19 list 110
ip msdp sa-filter in  150.2.19.19 list 110
access-list 110 permit ip any 239.0.0.0 0.255.255.255
```

### 3.2 R19 (AS 65200 RP/RR) — mirror toward R7

```
ip msdp peer 150.1.7.7 connect-source Loopback0
ip msdp originator-id Loopback0
```

Key points:
- `connect-source Loopback0` — the TCP (port 639) MSDP session sources from the loopback; the peer's `remote` side must have a route to that loopback (via the inter-AS BGP/IGP configured in WB09).
- `originator-id Loopback0` — stamps SA messages with the loopback (important when using anycast-RP).
- MSDP peering here is **inter-AS**; the two RPs must have unicast reachability to each other's loopbacks.

### 3.3 (Optional) Anycast-RP within AS 65100
If you want R7 and R8 to share the RP role:
```
! On both R7 and R8:
interface Loopback1
 ip address 150.1.77.77 255.255.255.255   ! shared anycast RP address
 ip pim sparse-mode
!
ip pim rp-address 150.1.77.77             ! (on all routers, instead of 150.1.7.7)
!
! R7:
ip msdp peer 150.1.8.8 connect-source Loopback0
ip msdp originator-id Loopback0
! R8:
ip msdp peer 150.1.7.7 connect-source Loopback0
ip msdp originator-id Loopback0
```

---

## 4. mVPN Profile 0 (Default MDT, PIM-in-the-core) for VRF GREEN

Profile 0 = GRE-encapsulated multicast using PIM in the provider core (default + data MDT). Applied on the Green PEs (R1, R2 in AS65100).

### 4.1 VRF GREEN mVPN config (on each Green PE — e.g. R1 and R2)

```
ip vrf GREEN
 rd 65100:910
 route-target export 65100:910
 route-target import 65100:910
 mdt default 239.1.1.1
 mdt data 239.1.2.0 0.0.0.255 threshold 1
!
! Enable multicast routing inside the VRF
ip multicast-routing vrf GREEN
!
! RP for the customer (Green) groups inside the VRF — e.g. a customer RP or the PE
ip pim vrf GREEN rp-address 150.1.7.7
```

- `mdt default 239.1.1.1` — the always-on default MDT group that forms the multicast "tunnel" between all PEs with VRF GREEN.
- `mdt data 239.1.2.0 0.0.0.255 threshold 1` — when a customer stream exceeds 1 kbps, move it off the default MDT onto a data MDT (from the 239.1.2.0/24 pool) so only interested PEs receive it.
- The default-MDT group 239.1.1.1 and the data range 239.1.2.0/24 must themselves be routable ASM groups in the **provider** PIM domain (RP = R7, already set).

### 4.2 PIM sparse-mode on the VRF (PE-CE) interfaces

On R1, the Green PE-CE interfaces are g1/0 (R9) and f3/0 (R10):

```
interface GigabitEthernet1/0
 ip pim sparse-mode
interface FastEthernet3/0
 ip pim sparse-mode
```

On R2, the Green PE-CE interface is g1/0 (R10, 192.168.4.2):
```
interface GigabitEthernet1/0
 ip pim sparse-mode
```

> These interfaces are in `ip vrf forwarding GREEN` (set in earlier L3VPN WBs); `ip pim sparse-mode` on a VRF interface enables customer multicast there. The MTI (multicast tunnel interface) is created automatically from the `mdt default`.

### 4.3 The provider core must carry 239.1.1.1 / 239.1.2.0/24
No extra config beyond section 2 — those groups are ASM and use RP R7. Confirm the core builds (*,239.1.1.1) and the PEs join it.

---

## 5. Verification

### 5.1 Core PIM / RP
```
show ip pim neighbor
show ip pim rp mapping
show ip igmp groups
show ip mroute
show ip mroute 239.1.1.1            ! default MDT group in provider table
show ip pim interface
```
Expect PIM neighbors on all core links, RP = 150.1.7.7 for ASM, and a (*,G)/(S,G) tree for the default-MDT group once PEs come up.

### 5.2 SSM
```
show ip pim ssm                     ! confirms 232.0.0.0/8 bound to ACL 1
show ip mroute 232.1.1.1            ! should be (S,G) only, no (*,G), no RP
```

### 5.3 MSDP
```
show ip msdp peer
show ip msdp peer 150.2.19.19       ! on R7 — state should be 'Up (Established)'
show ip msdp sa-cache               ! learned active sources from the other domain
show ip msdp count
```
Expect the MSDP session **Up**, and SA entries for sources registered in the peer AS.

### 5.4 mVPN
```
show ip pim vrf GREEN neighbor
show ip mroute vrf GREEN
show ip pim mdt                     ! default + data MDT groups and state
show ip pim mdt bgp                 ! MDT SAFI exchange (if using BGP MDT)
show ip mroute 239.1.2.0            ! data MDT once a stream crosses threshold
show ip pim vrf GREEN rp mapping
```
Expect VRF GREEN PIM neighbors over the MTI between PEs, the default MDT (239.1.1.1) established, and data MDTs (239.1.2.x) created on demand when a customer source exceeds 1 kbps.

---

## 6. Common gotchas

- `ip multicast-routing` (global) **and** `ip multicast-routing vrf GREEN` are both required — the VRF command alone is not enough.
- Every transit interface needs `ip pim sparse-mode`, **including loopbacks** used as RP/next-hops, or RPF fails.
- RPF uses the unicast table: if the path to the source/RP is via BGP (inter-AS), ensure `ip mroute` static RPF or multicast BGP (MP-BGP IPv4 mcast) is set where unicast and multicast topologies differ.
- SSM range ACL (`access-list 1`) must be a **standard** ACL permitting the group range; `ip pim ssm range 1` references it.
- For mVPN, the default-MDT group must be reachable in the global/provider PIM domain with a valid RP — if 239.1.1.1 has no RP, the MDT never forms.
- MSDP peers must have unicast reachability to each other's `connect-source` loopback; verify with `ping` sourced from Loopback0 before expecting the session to come up.
