# WB08 — MPLS Traffic Engineering Solutions (AS 65100)

**Platform:** Cisco 7200, IOS 15.2(4)M11 — **IOS Classic syntax**
**Scope:** MPLS-TE across the IS-IS L2 core (R1–R8)
**Prerequisite:** WB00–WB07 complete (IGP/IS-IS up, LDP running, L3VPN operational)

---

## 1. Topology Recap (AS 65100 core)

```
          R7(RR)                      R8(RR)
            |                           |
          f3/0                        f0/0
            |                           |
   R1 --f0/0--f0/0 R3 --g1/0--g1/0 R4 --g2/0--g2/0 R6
   |                 |                  |            |
   f4/0            g2/0               f0/0         f3/0
   |                 |                  |            |
   R2 --f0/0--f0/0 R4   R5 --f0/0--f0/0 R8   R5 --f3/0--f3/0 R6
```

Core links used by TE (from `00_topology_reference.md`):

| Link | A-side int / IP | B-side int / IP |
|------|-----------------|-----------------|
| R1↔R3 | f0/0 10.1.3.1 | f0/0 10.1.3.3 |
| R1↔R2 | f4/0 10.1.2.1 | f3/0 10.1.2.2 |
| R2↔R4 | f0/0 10.2.4.2 | f0/0 10.2.4.4 |
| R3↔R4 | g1/0 10.3.4.3 | g1/0 10.3.4.4 |
| R3↔R5 | g2/0 10.3.5.3 | g2/0 10.3.5.5 |
| R3↔R7 | f3/0 10.3.7.3 | f3/0 10.3.7.7 |
| R4↔R6 | g2/0 10.4.6.4 | g2/0 10.4.6.6 |
| R5↔R8 | f0/0 10.5.8.5 | f0/0 10.5.8.8 |
| R5↔R6 | f3/0 10.5.6.5 | f3/0 10.5.6.6 |

**Tunnel demo:** `Tunnel1` on R1 → R5 (150.1.5.5) via explicit path **R1 → R3 → R5**.

---

## 2. Global + IS-IS TE enablement (ALL core routers R1–R8)

MPLS-TE requires three things everywhere: a global knob, TE enabled under the IGP, and `mpls traffic-eng tunnels` + `ip rsvp bandwidth` on every core interface.

### 2.1 Global knob (every core router)

```
configure terminal
 mpls traffic-eng tunnels
```

### 2.2 IS-IS TE (every core router — process name CORE)

```
router isis CORE
 metric-style wide                     ! REQUIRED — TE needs wide metrics/sub-TLVs
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
```

> `metric-style wide` is mandatory — narrow metrics cannot carry the TE sub-TLVs. If WB01 already set it, leave it.

---

## 3. Per-router interface configuration

Each core interface needs:
```
 mpls traffic-eng tunnels
 ip rsvp bandwidth <interface-pool-kbps> [single-flow-kbps]
```
Below, FastEthernet pools are set to 75000 kbps (75% of 100M) and GigabitEthernet pools to 750000 kbps (75% of 1G). Adjust to taste; sub-pool left to default.

### R1 (PE — tunnel head-end)

```
interface Loopback0
 ip address 150.1.1.1 255.255.255.255
!
interface FastEthernet0/0
 description R1->R3
 ip address 10.1.3.1 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
interface FastEthernet4/0
 description R1->R2
 ip address 10.1.2.1 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R2 (PE)

```
interface FastEthernet3/0
 description R2->R1
 ip address 10.1.2.2 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
interface FastEthernet0/0
 description R2->R4
 ip address 10.2.4.2 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R3 (P — mid-point of Tunnel1)

```
interface FastEthernet0/0
 description R3->R1
 ip address 10.1.3.3 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
interface GigabitEthernet1/0
 description R3->R4
 ip address 10.3.4.3 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
interface GigabitEthernet2/0
 description R3->R5
 ip address 10.3.5.3 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
interface FastEthernet3/0
 description R3->R7
 ip address 10.3.7.3 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R4 (P)

```
interface FastEthernet0/0
 description R4->R2
 ip address 10.2.4.4 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
interface GigabitEthernet1/0
 description R4->R3
 ip address 10.3.4.4 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
interface GigabitEthernet2/0
 description R4->R6
 ip address 10.4.6.4 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R5 (ASBR — tunnel tail-end)

```
interface GigabitEthernet2/0
 description R5->R3
 ip address 10.3.5.5 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
interface FastEthernet0/0
 description R5->R8
 ip address 10.5.8.5 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
interface FastEthernet3/0
 description R5->R6
 ip address 10.5.6.5 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R6 (ASBR)

```
interface GigabitEthernet2/0
 description R6->R4
 ip address 10.4.6.6 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 750000 750000
!
interface FastEthernet3/0
 description R6->R5
 ip address 10.5.6.6 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R7 (RR)

```
interface FastEthernet3/0
 description R7->R3
 ip address 10.3.7.7 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

### R8 (RR)

```
interface FastEthernet0/0
 description R8->R5
 ip address 10.5.8.8 255.255.255.0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 75000 75000
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
mpls traffic-eng tunnels
```

---

## 4. Explicit path + Tunnel1 (R1 → R5 via R3 → R5)

### 4.1 Named explicit path on R1

Path uses the **next-hop IP** of each hop along R1→R3→R5. R1 exits via f0/0 to 10.1.3.3 (R3), then R3 to 10.3.5.5 (R5).

```
ip explicit-path name R1-R3-R5 enable
 next-address 10.1.3.3          ! R3 (R1->R3 link)
 next-address 10.3.5.5          ! R5 (R3->R5 link)
```

> Loose vs strict: the default `next-address` is strict. For a loose path use `next-address loose <ip>`.

### 4.2 Tunnel1 interface on R1

```
interface Tunnel1
 description TE R1->R5 explicit via R3
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.5.5
 tunnel mpls traffic-eng autoroute announce
 tunnel mpls traffic-eng priority 2 2
 tunnel mpls traffic-eng bandwidth 20000
 tunnel mpls traffic-eng path-option 10 explicit name R1-R3-R5
 tunnel mpls traffic-eng path-option 20 dynamic
```

Explanation:
- `ip unnumbered Loopback0` — tunnel source address.
- `tunnel destination 150.1.5.5` — R5 Loopback0 (TE router-id).
- `autoroute announce` — IGP installs routes reachable *behind* the tail-end via the tunnel.
- `priority 2 2` — setup/hold priority (0 = highest, 7 = lowest).
- `bandwidth 20000` — reserve 20 Mbps.
- `path-option 10 explicit` preferred; `path-option 20 dynamic` is the CSPF fallback if the explicit path fails.

---

## 5. Autoroute, metrics, and forwarding-adjacency options

### 5.1 Autoroute announce (already on Tunnel1)
Makes the tunnel look like a directly-connected IGP link toward the tail and everything behind it. Verify with `show ip route` — routes to 150.1.5.5 and beyond point out `Tunnel1`.

### 5.2 Autoroute metric (optional tuning)
```
interface Tunnel1
 tunnel mpls traffic-eng autoroute metric relative -5     ! or 'absolute <n>'
```

### 5.3 Forwarding adjacency (advertise tunnel into IGP as a link — optional)
```
interface Tunnel1
 tunnel mpls traffic-eng forwarding-adjacency
 isis metric 10 level-2
```

---

## 6. Fast ReRoute (FRR)

Scenario: protect **Tunnel1** against failure of the R3→R5 link. R3 is the **PLR (Point of Local Repair)**; the protected resource is the R3–R5 link; the **MP (Merge Point)** is R5. We build a NNHOP (next-next-hop) bypass is not possible with a single mid-hop, so here we use a **NHOP bypass** that reroutes R3→R5 traffic around via R3→R4→R6→R5.

### 6.1 Enable FRR on the protected tunnel head-end (R1)

```
interface Tunnel1
 tunnel mpls traffic-eng fast-reroute
```
(Optional: `tunnel mpls traffic-eng fast-reroute bw-protect` to require bandwidth protection.)

### 6.2 Build the bypass tunnel on the PLR (R3)

Bypass path R3 → R4 → R6 → R5 (avoids the protected R3–R5 link). Next-hops: R3 f/g to 10.3.4.4 (R4), R4 to 10.4.6.6 (R6), R6 to 10.5.6.5 (R5).

```
ip explicit-path name BYPASS-R3-R5 enable
 next-address 10.3.4.4          ! R4
 next-address 10.4.6.6          ! R6
 next-address 10.5.6.5          ! R5 (merge point)
!
interface Tunnel100
 description FRR NHOP bypass protecting R3->R5
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.5.5
 tunnel mpls traffic-eng path-option 10 explicit name BYPASS-R3-R5
 tunnel mpls traffic-eng bandwidth 0            ! bypass carries protected BW; 0 = best-effort reroute
!
```

### 6.3 Associate the bypass with the protected interface on the PLR (R3)

```
interface GigabitEthernet2/0
 description R3->R5 (protected)
 mpls traffic-eng backup-path Tunnel100
```

### 6.4 FRR verification (on PLR R3)

```
show mpls traffic-eng fast-reroute database
show mpls traffic-eng tunnels backup
show ip rsvp fast-reroute
```

After failing the R3–R5 link (`shutdown` g2/0 on R3), protected LSPs should show state **"ready" → "active"** and traffic reroutes in < 50 ms.

---

## 7. Bandwidth, priority, and auto-bandwidth examples

### 7.1 Static bandwidth + priority (already shown on Tunnel1)
```
interface Tunnel1
 tunnel mpls traffic-eng bandwidth 20000
 tunnel mpls traffic-eng priority 2 2
```

### 7.2 Priority classes example (a higher-priority tunnel pre-empts a lower one)
```
interface Tunnel2
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.6.6
 tunnel mpls traffic-eng priority 0 0        ! highest — can pre-empt Tunnel1 (prio 2)
 tunnel mpls traffic-eng bandwidth 60000
 tunnel mpls traffic-eng path-option 10 dynamic
 tunnel mpls traffic-eng autoroute announce
```

### 7.3 Auto-bandwidth (adjusts reservation from measured load)

Global timers (on head-ends, e.g. R1):
```
mpls traffic-eng auto-bw timers frequency 300          ! sample every 5 min
```

Per-tunnel auto-bw:
```
interface Tunnel1
 tunnel mpls traffic-eng auto-bw frequency 300 max-bw 80000 min-bw 10000
```
- `frequency` — how often the reservation is re-evaluated/resized.
- `max-bw` / `min-bw` — resize guardrails. The head re-signals the LSP when the rolling max exceeds the current reservation.

---

## 8. Verification

### 8.1 Tunnel state
```
show mpls traffic-eng tunnels
show mpls traffic-eng tunnels tunnel 1
show mpls traffic-eng tunnels brief
```
Expect Tunnel1: **Admin: up / Oper: up**, path "R1-R3-R5", bandwidth 20000, Explicit path option 10 **active**.

### 8.2 RSVP reservations
```
show ip rsvp reservation
show ip rsvp reservation detail
show ip rsvp interface
show ip rsvp sender
```
`show ip rsvp interface` should show the reservable/allocated bandwidth per interface. Along R1→R3→R5, you should see 20000 kbps reserved for the Tunnel1 LSP.

### 8.3 TE topology / CSPF database
```
show mpls traffic-eng topology
show isis mpls traffic-eng advertisements
show mpls traffic-eng link-management advertisements
```

### 8.4 Autoroute result
```
show ip route 150.1.5.5
show mpls traffic-eng autoroute
```
Route to 150.1.5.5 should resolve via **Tunnel1**.

### 8.5 FRR
```
show mpls traffic-eng fast-reroute database
show mpls traffic-eng tunnels backup
```

---

## 9. Common gotchas

- Missing `metric-style wide` under IS-IS → tunnels stay down, CSPF finds no path.
- `mpls traffic-eng router-id Loopback0` must point to a /32 that is reachable in the IGP and matches the `tunnel destination` on head-ends.
- `ip rsvp bandwidth` missing on a transit interface → CSPF excludes that link.
- Explicit-path `next-address` must be the **far-end interface IP** of each hop (strict), not the loopback.
- Autoroute only installs routes to/behind the tail-end; it does not create a route to the tunnel destination's directly-connected peers unless they are "behind" the tail.
