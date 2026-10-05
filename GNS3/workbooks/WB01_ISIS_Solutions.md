# WB01 IS-IS — Solutions

**Reference:** `00_topology_reference.md` for IP addressing, NET-IDs, and wiring.

---

## Section 1: IS-IS L2-Only on AS 65100 (R1-R8)

### R1 (PE)

```
router isis CORE
 net 49.0001.1500.0100.1001.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet4/0
 ip router isis CORE
 isis network point-to-point
!
```

> R1 core interfaces: f0/0 (→R3), f4/0 (→R2). PE-CE interfaces (g1/0→R9, f3/0→R10, f4/1→R32, g2/0→R29) NOT in IS-IS.

### R2 (PE)

```
router isis CORE
 net 49.0001.1500.0100.2002.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 ip router isis CORE
 isis network point-to-point
!
```

> R2 core interfaces: f0/0 (→R4), f3/0 (→R1). PE-CE interface (g1/0→R10) NOT in IS-IS.

### R3 (P)

```
router isis CORE
 net 49.0001.1500.0100.3003.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 ip router isis CORE
 isis network point-to-point
!
```

> R3 core interfaces: f0/0 (→R1), g1/0 (→R4), g2/0 (→R5), f3/0 (→R7). All core — R3 is a P router.

### R4 (P)

```
router isis CORE
 net 49.0001.1500.0100.4004.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
```

> R4 core interfaces: f0/0 (→R2), g1/0 (→R3), g2/0 (→R6). All core.

### R5 (ASBR)

```
router isis CORE
 net 49.0001.1500.0100.5005.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 ip router isis CORE
 isis network point-to-point
!
```

> R5 core interfaces: f0/0 (→R8), g2/0 (→R3), f3/0 (→R6). Inter-AS interface (g1/0→R11) NOT in IS-IS.

### R6 (ASBR)

```
router isis CORE
 net 49.0001.1500.0100.6006.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 ip router isis CORE
 isis network point-to-point
!
```

> R6 core interfaces: g2/0 (→R4), f3/0 (→R5). Inter-AS interfaces (g1/0→R12, f0/0→R20) NOT in IS-IS.

### R7 (RR)

```
router isis CORE
 net 49.0001.1500.0100.7007.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet3/0
 ip router isis CORE
 isis network point-to-point
!
```

> R7 single connection: f3/0 (→R3). That's it — RR with one uplink.

### R8 (RR)

```
router isis CORE
 net 49.0001.1500.0100.8008.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
```

> R8 single connection: f0/0 (→R5). RR with one uplink.

### Verification

```
show isis neighbors                          ! all adjacencies L2 UP
show isis database                           ! one .00-00 per router, no pseudonodes
show ip route isis                           ! all 8 loopbacks (150.1.X.X) reachable
show isis database R1.00-00 detail           ! verify IP-Extended (wide metrics)
ping 150.1.8.8 source 150.1.1.1             ! R1 to R8 end-to-end
```

---

## Section 2: IS-IS L2-Only on AS 65300 (R20-R23, R30)

### R20 (ASBR/PE)

```
router isis CORE
 net 49.0003.1500.0302.0020.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
```

> R20 core: g1/0 (→R21), g2/0 (→R22). Inter-AS (f0/0→R6) and PE-CE (f3/0→R24) NOT in IS-IS.

### R21 (ASBR/PE)

```
router isis CORE
 net 49.0003.1500.0302.1021.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
```

> R21 core: f0/0 (→R23), g1/0 (→R20). Inter-AS (f3/0→R12) and PE-CE (g2/0→R25) NOT in IS-IS.

### R22 (PE)

```
router isis CORE
 net 49.0003.1500.0302.2022.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
```

> R22 core: g1/0 (→R23), g2/0 (→R20). PE-CE (f0/0→R24) NOT in IS-IS.

### R23 (PE)

```
router isis CORE
 net 49.0003.1500.0302.3023.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
```

> R23 core: f0/0 (→R21), g1/0 (→R22), g2/0 (→R30). PE-CE (f3/0→R25) NOT in IS-IS.

### R30 (RR)

```
router isis CORE
 net 49.0003.1500.0303.0030.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet2/0
 ip router isis CORE
 isis network point-to-point
!
```

> R30 single connection: g2/0 (→R23). RR with one uplink.

### Verification

```
show isis neighbors                          ! all AS65300 adjacencies L2 UP
show ip route isis                           ! all 5 loopbacks (150.3.X.X) reachable
ping 150.3.30.30 source 150.3.20.20         ! R20 to R30 end-to-end
```

---

## Section 3: IS-IS Advanced

### Authentication (ALL routers in AS 65100 — key "CCIE")

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

> Apply on ALL R1-R8. Authentication must match on both sides of every adjacency. If one router has auth and the neighbor doesn't — adjacency drops.

### Authentication (ALL routers in AS 65300 — key "CCNP")

```
key chain ISIS-KEY
 key 1
  key-string CCNP
!
router isis CORE
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
!
```

> Apply on ALL R20-R23, R30. Different key from AS65100 — separate IS-IS domains.

### BFD (all IS-IS interfaces)

```
! On each core-facing interface (example R1):
interface FastEthernet0/0
 bfd interval 500 min_rx 500 multiplier 3
!
router isis CORE
 bfd all-interfaces
!
```

> Apply `bfd interval` on every core interface. `bfd all-interfaces` under IS-IS enables BFD for all IS-IS adjacencies. Note: Dynamips may not support BFD — adjacency may not show BFD status. Configure for learning purposes.

### Prefix Suppression (P routers R3, R4 in AS65100)

```
! On P-P core interfaces (not loopbacks):
interface FastEthernet0/0
 isis prefix-suppression
!
interface GigabitEthernet1/0
 isis prefix-suppression
!
```

> Suppresses transit link /24 prefixes from IS-IS. Only loopback /32s remain. Reduces routing table size. Apply on P routers' core interfaces.

### Overload Bit on Startup (ALL routers)

```
router isis CORE
 set-overload-bit on-startup 180
!
```

> All routers in both ASes. After reboot, advertise overload for 180 seconds. Transit traffic avoids this router until it fully converges.

### Verification

```
show isis database detail | include auth        ! "Authentication" present in LSPs
show isis neighbors detail                       ! BFD: enabled (if supported)
show isis database detail | include 10.          ! suppressed transit prefixes gone from P routers
show isis database | include OL                  ! overload bit after recent reboot
```

---

## Section 4: IS-IS IPv6 (single-topology)

### Enable on ALL AS 65100 routers (example R1):

```
ipv6 unicast-routing
!
interface Loopback0
 ipv6 address 2001:db8:1::1/128
!
interface FastEthernet0/0
 ipv6 address 2001:db8:1:13::1/64
!
interface FastEthernet4/0
 ipv6 address 2001:db8:1:12::1/64
!
router isis CORE
 address-family ipv6 unicast
  single-topology
!
interface Loopback0
 ipv6 router isis CORE
!
interface FastEthernet0/0
 ipv6 router isis CORE
!
interface FastEthernet4/0
 ipv6 router isis CORE
!
```

> Pattern: add IPv6 addresses per topology reference, enable `ipv6 unicast-routing`, add `ipv6 router isis CORE` on each interface, enable `address-family ipv6 unicast / single-topology` under router isis. Repeat for ALL AS65100 routers with their respective IPv6 addresses from the topology reference.

### Enable on ALL AS 65300 routers (same pattern, using 2001:db8:3: addresses)

Same structure — add IPv6 addresses, `ipv6 router isis CORE` on each core interface, `single-topology` under router isis.

> **CRITICAL:** `single-topology` must be on ALL routers in the domain. Mismatch = IPv6 routes don't install (TLV 236 vs 237 issue).

### Verification

```
show ipv6 route isis                            ! all IPv6 loopbacks reachable
show isis database detail | include IPv6         ! IPv6 prefixes in LSPs (no MT prefix = single-topology)
ping ipv6 2001:db8:1::8 source 2001:db8:1::1   ! R1 to R8 IPv6 end-to-end
```

---

## Section 5: Troubleshooting

### Adjacency stuck in INIT
- **Cause 1:** Authentication mismatch — one side has auth, other doesn't. Check `show isis neighbors` — stuck in INIT. Fix: match key-chain and mode on both sides.
- **Cause 2:** MTU mismatch — check `show isis interface detail` for MTU. IS-IS pads hellos to MTU. Mismatch = hello rejected.
- **Cause 3:** Area/level mismatch — one side L1, other L2-only. Check `is-type` matches.

### Routes missing
- **Cause 1:** Interface not in IS-IS — missing `ip router isis CORE` on the interface.
- **Cause 2:** Passive not configured on loopback — missing `passive-interface Loopback0` under router isis.
- **Cause 3:** Wrong `metric-style` — one router narrow, others wide. Check `show isis database detail` — `IP-Extended` (wide) vs `IP` (narrow).

### Pseudonodes in LSDB
- **Cause:** Missing `isis network point-to-point` on the interface. Fix: add it on both ends. Pseudonodes disappear within seconds.
