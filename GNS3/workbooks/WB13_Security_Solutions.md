# WB13 Security — Solutions

**Reference:** `00_topology_reference.md` for IP addressing, NET-IDs, and wiring.
**Platform:** Cisco 7200, IOS 15.2(4)M11. IOS classic syntax.
**Prerequisite:** WB00–WB11 complete (IGP, LDP, BGP, L3VPN operational).

> Control-plane and routing security across all three ASes. Password `CCIE` used throughout for lab consistency.

---

## Section 1 — IGP Authentication

### 1.1 IS-IS auth (AS65100, AS65300) — already in WB01 solutions

```
key chain ISIS-KEY
 key 1
  key-string CCIE
!
router isis CORE
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
!
```

> AS65100 uses key-string `CCIE`, AS65300 uses `CCNP` (separate domains) per WB01. HMAC-MD5 via `authentication mode md5`. Apply on all routers in each domain; both ends of every adjacency must match or the adjacency drops.

### 1.2 OSPF auth (AS65200) — MD5 on all core interfaces

Apply on **every** OSPF interface in AS65200 (R11–R16, R19 core links). Example per router pattern:

```
! R11 (P) — core interfaces f0/0, g2/0, f3/0
interface FastEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIE
!
interface GigabitEthernet2/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIE
!
interface FastEthernet3/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIE
!
```

> Alternative area-wide: `area 0 authentication message-digest` under `router ospf 1`, then only the key per interface. Interface-level `ip ospf authentication message-digest` shown here is explicit and overrides area setting. Apply the same key 1 / md5 CCIE on:
> - R11: f0/0, g2/0, f3/0
> - R12: f0/0 (→R11), g2/0 (→R14)   *(inter-AS f3/0→R21 is BGP, not OSPF)*
> - R13: g2/0, f0/0, g1/0, f3/0
> - R14: g2/0, g1/0, f0/0
> - R15: f3/0, f0/0 (f0/0→R17 is OSPF PE-CE, auth it too)
> - R16: f0/0 (→R13)
> - R19: f3/0, f0/0

### OSPF Verification

```
show ip ospf interface FastEthernet0/0 | include authentication   ! Message digest
show ip ospf neighbor                                              ! FULL on all adjacencies
debug ip ospf adj                                                   ! "mismatch" if key wrong
```

---

## Section 2 — BGP Session Security

### 2.1 BGP auth (MD5 password) — ALL BGP sessions

```
! Example R5 (AS65100 ASBR) — iBGP to RRs + eBGP to R11
router bgp 65100
 neighbor 150.1.7.7 password CCIE          ! iBGP RR R7
 neighbor 150.1.8.8 password CCIE          ! iBGP RR R8
 neighbor 10.5.11.11 password CCIE         ! eBGP R11 (AS65200)
!
```

> Apply `neighbor X password CCIE` on **every** BGP peering (iBGP + eBGP), both ends. Mismatch → session never establishes, log shows `%TCP-6-BADAUTH`.
> - Inter-AS eBGP: R5↔R11, R6↔R12, R6↔R20, R12↔R21
> - PE-CE eBGP: R1↔R9, R1↔R10, R1↔R32, R2↔R10, R14↔R25, R16↔R18, R16↔R26, R20↔R24, R21↔R25, R22↔R24, R23↔R25
> - iBGP: all PE/ASBR ↔ RR sessions (R7/R8, R19, R30)

### 2.2 GTSM (TTL security) — eBGP peers only

```
! Example R5 toward R11 (directly-connected eBGP)
router bgp 65100
 neighbor 10.5.11.11 ttl-security hops 1
!
```

> `ttl-security hops 1` requires received TTL ≥ 254 (255 − 1). Protects against spoofed eBGP from >1 hop away. Apply on both ends of each directly-connected eBGP session. **Mutually exclusive with `neighbor X ebgp-multihop`** — do not combine. Apply to all inter-AS and PE-CE eBGP peers (hops = actual hop count; 1 for directly connected).

### 2.3 Maximum-prefix — eBGP PE-CE

```
! Example R1 toward CE R9
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.1.9 maximum-prefix 100 80
 exit-address-family
!
```

> `maximum-prefix 100 80` → warn at 80%, tear down session at 100 prefixes. Protects PE from a misbehaving customer leaking the full table. Apply on all PE-CE eBGP neighbors (inside the proper VRF AF where applicable). Add `restart 5` to auto-recover after 5 min, or `warning-only` to log without tearing down.

### Verification (Sections 2.1–2.3)

```
show ip bgp neighbors 10.5.11.11 | include password|MD5|BGP state
show ip bgp neighbors 10.5.11.11 | include External|ttl-security|minimum
show ip bgp neighbors 192.168.1.9 | include maximum-prefix|Threshold
```

---

## Section 3 — RTBH (Remotely Triggered Black Hole)

**Concept:** Trigger router advertises a victim /32 tagged for blackhole; upstream routers set its next-hop to a pre-installed Null0 discard route. Here we blackhole `192.168.200.200/32`.

### 3.1 Trigger router — static to Null0 with tag + BGP redistribute

```
! Trigger router (e.g. R7 RR acting as trigger, or any iBGP speaker)
ip route 192.168.200.200 255.255.255.255 Null0 tag 666
!
route-map RTBH-TRIGGER permit 10
 match tag 666
 set community no-export no-advertise
 set ip next-hop 192.0.2.1          ! well-known "blackhole" next-hop marker
!
router bgp 65100
 redistribute static route-map RTBH-TRIGGER
!
```

> The trigger advertises the /32 with community `no-export`/`no-advertise` (keep it internal) and a next-hop that upstream routers will remap to Null0.

### 3.2 Upstream routers — static discard + match community → Null0 next-hop

```
! Every edge/upstream router (e.g. ASBRs R5, R6)
! Pre-installed discard route for the "blackhole" next-hop:
ip route 192.0.2.1 255.255.255.255 Null0
!
! BGP prefixes whose next-hop resolves to 192.0.2.1 are now discarded.
! (No route-map needed on receivers if next-hop is set at trigger;
!  recursion to the Null0 discard route does the blackholing.)
```

> Flow: trigger sets next-hop `192.0.2.1` on the victim /32 → upstream has `ip route 192.0.2.1 → Null0` → victim traffic recurses to Null0 and is dropped at the edge, saving core bandwidth. `no-export` prevents the /32 from leaking to other ASes.

### RTBH Verification

```
show ip bgp 192.168.200.200                     ! community no-export no-advertise, next-hop 192.0.2.1
show ip route 192.168.200.200                   ! recurses via 192.0.2.1 → Null0
show ip cef 192.168.200.200                      ! receive→drop / Null0
show ip bgp community no-export                  ! tagged prefixes present
```

---

## Section 4 — CoPP (Control Plane Policing)

Protect the route processor. Permit/limit BGP (TCP 179) as the example class.

```
ip access-list extended COPP-BGP
 permit tcp any any eq 179
 permit tcp any eq 179 any
!
class-map match-all COPP-BGP
 match access-group name COPP-BGP
!
policy-map COPP
 class COPP-BGP
  police 8000 conform-action transmit exceed-action drop
 class class-default
  police 32000 conform-action transmit exceed-action transmit
!
control-plane
 service-policy input COPP
!
```

> Apply on all routers hosting BGP. The BGP class rate-limits control traffic to the RP; `class-default` catches the rest. Tune `police` rates to platform. On Dynamips the policer counters may be approximate.

### CoPP Verification

```
show policy-map control-plane                   ! class match counts + police conform/exceed
show policy-map control-plane | include BGP|drop
```

---

## Section 5 — Bogon / Prefix Filtering (eBGP inbound)

```
ip prefix-list BOGONS deny 10.0.0.0/8 le 32
ip prefix-list BOGONS deny 172.16.0.0/12 le 32
ip prefix-list BOGONS deny 192.168.0.0/16 le 32
ip prefix-list BOGONS deny 127.0.0.0/8 le 32
ip prefix-list BOGONS deny 169.254.0.0/16 le 32
ip prefix-list BOGONS deny 0.0.0.0/0            ! default (optional)
ip prefix-list BOGONS permit 0.0.0.0/0 le 32
!
! Apply inbound on each eBGP ASBR neighbor
router bgp 65100
 neighbor 10.5.11.11 prefix-list BOGONS in
!
```

> Blocks RFC1918/martian space from entering over eBGP. Apply inbound on all inter-AS eBGP sessions (R5↔R11, R6↔R12, R6↔R20, R12↔R21). **Note:** in this lab the customer/VPN space itself uses 192.168.x and 10.x — apply BOGONS only on the public inter-AS edge, not on PE-CE VRF sessions, or the lab VPN prefixes will be filtered. Scope accordingly.

### Bogon Verification

```
show ip prefix-list BOGONS
show ip bgp neighbors 10.5.11.11 | include prefix-list
show ip bgp neighbors 10.5.11.11 received-routes    ! bogons absent
```

---

## Section 6 — LDP Authentication

```
! Both ends of every LDP session in AS65100 (and AS65200/65300 cores)
mpls ldp neighbor 150.1.3.3 password CCIE
mpls ldp neighbor 150.1.7.7 password CCIE
...
```

> Set `mpls ldp neighbor <LDP-ID> password CCIE` for each LDP peer (the LDP router-id, normally the loopback). Both ends must match. Mismatch → `%LDP-5-PWD` and the session fails to form; LSPs break.
> Global alternative: `mpls ldp password required` + `mpls ldp password fallback CCIE` to enforce a password on all sessions.

### LDP Verification

```
show mpls ldp neighbor detail | include Password|State
show mpls ldp discovery
show mpls forwarding-table                       ! labels restored after auth matches
```

---

## Section 7 — BGP Flowspec (platform note)

```
! Conceptual — IOS classic on 7200/Dynamips does NOT support BGP Flowspec.
! Flowspec (RFC 5575) requires IOS XR / XE ASR platforms:
!   flowspec
!     address-family ipv4
!       local-install interface-all
!   route-policy FS-REDIRECT ... match destination-prefix ... redirect ...
```

> **Dynamips limitation:** BGP Flowspec SAFI 133/134 and `flowspec` config are not available on the 7200 IOS 15.2(4)M image. Document the design; implement on real XR/XE hardware. Use RTBH (Section 3) as the Dynamips-supported alternative for traffic blackholing.

---

## Completion Checklist

- [ ] IS-IS auth active AS65100 (CCIE) + AS65300 (CCNP) — adjacencies hold
- [ ] OSPF MD5 (key 1 / CCIE) on all AS65200 interfaces — neighbors FULL
- [ ] BGP password CCIE on every iBGP + eBGP session — no BADAUTH
- [ ] GTSM `ttl-security hops 1` on all directly-connected eBGP peers
- [ ] maximum-prefix 100 80 on all PE-CE eBGP
- [ ] RTBH: 192.168.200.200/32 blackholed via Null0, no-export community
- [ ] CoPP policy applied on control-plane, BGP class policing
- [ ] BOGONS prefix-list inbound on inter-AS eBGP edges
- [ ] LDP password CCIE on all LDP sessions — LFIB intact
- [ ] Flowspec documented as unsupported on Dynamips
- [ ] **GNS3 snapshot:** `wb13-security-complete`
