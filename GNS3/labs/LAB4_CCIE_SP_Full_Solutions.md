# LAB 4 — Full CCIE SP Practice Lab — SOLUTIONS

**Platform:** EVE-NG, IOS-XRv 9000 (7.x+)
**Companion to:** `LAB4_CCIE_SP_Full.md`
**Note:** Configs below use the exact topology/addressing from Section 0 of the task file. Interface names are shown generically as `Gi0/0/0/X` — substitute your actual EVE-NG interface mapping. Loopback0 is the router-id/BGP/IS-IS source for every router unless stated otherwise.

---

## 0. Topology and Addressing (as given)

```
                        RR1 (L2)                    RR2 (L2)
                      10.0.0.11/32                10.0.0.12/32
                           \                          /
                            \________________________/
                            /          |             \
                     P1(L1/L2)   P2(L1/L2)      P3(L1/L2)
                   10.0.0.21/32  10.0.0.22/32   10.0.0.23/32
                      /    \         |              /    \
                    PE1    PE2      PE3           PE4   (ASBR-A/ASBR-B for Inter-AS)
                 10.0.0.1  10.0.0.2 10.0.0.3     10.0.0.4
                  (L1)      (L1)     (L1)         (L1)
                   |          |        |            |
                  CE1        CE2      CE3          CE4
```

Point-to-point link addressing used throughout this solution (if your EVE-NG build already has link IPs assigned, keep yours — only the loopbacks/SIDs/EVIs below are normative to the task):

| Link | Subnet |
|---|---|
| PE1–P1 | 10.1.12.0/30 (PE1=.1, P1=.2) |
| PE2–P1 | 10.1.13.0/30 (PE2=.1, P1=.2) |
| PE3–P2 | 10.1.14.0/30 (PE3=.1, P2=.2) |
| PE4–P3 | 10.1.15.0/30 (PE4=.1, P3=.2) |
| P1–P2 | 10.2.12.0/30 |
| P2–P3 | 10.2.23.0/30 |
| P1–RR1 | 10.2.14.0/30 |
| P2–RR1 | 10.2.24.0/30 |
| P2–RR2 | 10.2.25.0/30 |
| P3–RR2 | 10.2.35.0/30 |
| P3–ASBR-A | 10.2.36.0/30 |
| ASBR-A–ASBR-B | 10.9.0.0/30 (ASBR-A=.1, ASBR-B=.2) |
| PE-CE links | 192.168.x.0/30 per CE, x = CE number |

---
## Section 1 — IGP, Segment Routing, SRv6, and Traffic Engineering

### Task 1.1 — IS-IS Multi-Level Design — SOLUTION

**Design:** PEs = L1-only (stub into Area 49.0001). P1/P2/P3 = L1/L2 border. RR1/RR2/ASBR-A = L2-only backbone.

**PE1 (L1-only), representative for PE1-PE4 (substitute NET/loopback per router):**
```
interface Loopback0
 ipv4 address 10.0.0.1 255.255.255.255
!
router isis CORE
 is-type level-1
 net 49.0001.0100.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
  reference-bandwidth 10000000
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface Gi0/0/0/0
  circuit-type level-1
  address-family ipv4 unicast
   metric 100
```
> PE-uplink is 1 Gbps → cost 100 under `reference-bandwidth 10000000` (10,000,000 kbps = 10 Gbps baseline, so cost = refbw/bw = 10,000,000/1,000,000 = 10... to hit the required 10/100 ratio exactly, set `reference-bandwidth 1000000` (1 Gbps baseline) so a 1G link = cost 100 (scaled x100 for granularity) and a 10G link = cost 10. In practice on IOS-XR, explicitly set `metric` per-interface rather than relying purely on auto-cost — this guarantees the 10/100 ratio the task requires regardless of auto-bandwidth derivation:
> - 10 Gbps core interfaces: `metric 10` (both L1 and L2 AF context)
> - 1 Gbps PE-uplink interfaces: `metric 100`

**P1 (L1/L2 border), representative for P1-P3:**
```
interface Loopback0
 ipv4 address 10.0.0.21 255.255.255.255
!
router isis CORE
 is-type level-1-2
 net 49.0001.0100.0000.0021.00
 address-family ipv4 unicast
  metric-style wide
  reference-bandwidth 10000000
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface Gi0/0/0/0        ! toward PE1 (L1 side)
  circuit-type level-1
  address-family ipv4 unicast
   metric 100
 !
 interface Gi0/0/0/1        ! toward PE2 (L1 side)
  circuit-type level-1
  address-family ipv4 unicast
   metric 100
 !
 interface Gi0/0/0/2        ! toward P2 (L2 core, 10G)
  circuit-type level-2-only
  address-family ipv4 unicast
   metric 10
 !
 interface Gi0/0/0/3        ! toward RR1 (L2 core, 10G)
  circuit-type level-2-only
  address-family ipv4 unicast
   metric 10
```

**RR1 (L2-only), representative for RR1, RR2, ASBR-A:**
```
interface Loopback0
 ipv4 address 10.0.0.11 255.255.255.255
!
router isis CORE
 is-type level-2-only
 net 49.0001.0100.0000.0011.00
 address-family ipv4 unicast
  metric-style wide
  reference-bandwidth 10000000
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface Gi0/0/0/0        ! toward P1
  address-family ipv4 unicast
   metric 10
 !
 interface Gi0/0/0/1        ! toward P2
  address-family ipv4 unicast
   metric 10
```

NET pattern confirms the scheme: `49.0001.0100.0000.00XX.00` where XX = zero-padded last loopback octet (e.g., PE1 → `0001`, P1 → `0021`, RR1 → `0011`, ASBR-A → `0031`).

**Explanation:**
- `is-type level-1` on PEs restricts them to forming only L1 adjacencies — no L2 Hello is ever sent, so no L2 adjacency can form even if a neighbor is L2-capable.
- `is-type level-1-2` on P1-P3 lets them run both L1 (toward PEs, Area 49.0001) and L2 (toward the backbone) simultaneously, acting as the Area Border Router.
- `is-type level-2-only` on RR1/RR2/ASBR-A removes L1 entirely — they only ever appear in the backbone.
- L1→L2 leaking is automatic on L1/L2 routers: P1-P3 redistribute their attached L1 PE loopbacks into L2 LSPs as external L1 routes carrying the ATT (attached) bit set in their own L1 LSP, telling L1-only PEs "I am the way out of this area" (default route candidate).

**Verify:**
```
show isis neighbors
show isis adjacency detail
show isis database verbose
show route isis
```
On PE1: `show isis database verbose` for its own L1 LSP should show the ATT bit set on P1's LSP (not PE1's). On P1: `show isis database level 2` should show PE1/PE2's /32s present as L1-sourced prefixes leaked into L2.

---

### Task 1.2 — Segment Routing MPLS — New Prefix-SID Scheme — SOLUTION

**SRGB:** `16000-23999` on all routers. **Partition:** PEs index `100+X`, P routers `200+X`, RR/ASBR `300+X` (X = last loopback octet). **Anycast-SID** `16050` shared P1+P2 only.

**Global SRGB (all routers, representative PE1):**
```
segment-routing
 global-block 16000 23999
```

**PE1 — prefix-SID 16101 (index 101), N-flag only, no explicit-null:**
```
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 101 n-flag-clear-disable
```
> On IOS-XR, the N-flag (node-SID) is set by default for loopback prefix-SIDs; `explicit-null` is a separate optional keyword you simply omit. Do not add `explicit-null`.

**PE2:** `prefix-sid index 102` → absolute SID 16102
**PE3:** `prefix-sid index 103` → absolute SID 16103
**PE4:** `prefix-sid index 104` → absolute SID 16104

**P1 — index 221 → SID 16221, PLUS anycast-SID 16050 (index 50) on a dedicated secondary loopback shared with P2:**
```
interface Loopback0
 ipv4 address 10.0.0.21 255.255.255.255
!
interface Loopback50
 ipv4 address 10.0.50.50 255.255.255.255
!
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 221
 !
 interface Loopback50
  address-family ipv4 unicast
   prefix-sid index 50
   prefix-sid n-flag-clear          ! anycast prefix-SID: N-flag must be CLEARED (anycast, not unique-node)
```

**P2 — index 222 → SID 16222, same anycast Loopback50/10.0.50.50 and index 50:**
```
interface Loopback0
 ipv4 address 10.0.0.22 255.255.255.255
!
interface Loopback50
 ipv4 address 10.0.50.50 255.255.255.255
!
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 222
 !
 interface Loopback50
  address-family ipv4 unicast
   prefix-sid index 50
   prefix-sid n-flag-clear
```

**P3 — index 223 → SID 16223 (no anycast participation):**
```
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 223
```

**RR1 (index 311 → 16311), RR2 (index 312 → 16312), ASBR-A (index 331 → 16331):** same pattern as P1/P2 under `address-family ipv4 unicast segment-routing mpls`.

**Explanation:**
- The anycast-SID 16050 is advertised identically by P1 and P2 from the *same* address (10.0.50.50/32) with the N-flag cleared — this tells the SR control plane "multiple nodes own this SID equally," enabling ECMP: any router computing SPF to 16050 installs both P1's and P2's next-hops.
- Index-based allocation (`prefix-sid index N`) means the absolute label is always `SRGB-start + N` = `16000+N`, so the scheme is portable across SRGB shifts.
- No LDP anywhere — SR is the sole MPLS control plane (verify no `mpls ldp` process is configured/running).

**Verify:**
```
show isis segment-routing label table
show mpls label table
show route ipv4 label 16101 detail
show route ipv4 label 16050 detail      ! should show two paths / ECMP, next-hops = P1 and P2 physical interfaces
show mpls ldp neighbor                   ! should return nothing — confirm LDP is NOT running anywhere
```

---

### Task 1.3 — Flex-Algo 128 with Include-All — SOLUTION

**Affinity bits:** `LOW-LATENCY`=bit0 on P1-P2/P2-P3; `HIGH-CAPACITY`=bit1 on P1-RR1/P3-ASBR-A. Flex-Algo 128 = IGP metric, `include-all` requiring BOTH bits.

**Affinity-map (same on every L2 router — P1-P3, RR1, RR2, ASBR-A):**
```
segment-routing
 traffic-eng
  affinity-map
   name LOW-LATENCY bit-position 0
   name HIGH-CAPACITY bit-position 1
```

**Interface affinity tagging — P1 (toward P2 = LOW-LATENCY; toward RR1 = HIGH-CAPACITY):**
```
router isis CORE
 interface Gi0/0/0/2          ! P1-P2 link
  address-family ipv4 unicast
   affinity
    affinity-name LOW-LATENCY
 !
 interface Gi0/0/0/3          ! P1-RR1 link
  address-family ipv4 unicast
   affinity
    affinity-name HIGH-CAPACITY
```
Mirror on P2 (P1-P2 and P2-P3 interfaces = `LOW-LATENCY`) and P3 (P2-P3 = `LOW-LATENCY`, P3-ASBR-A = `HIGH-CAPACITY`).

**Flex-Algo 128 definition — advertised from P1 only:**
```
segment-routing
 traffic-eng
  flex-algo 128
   metric-type igp
   affinity
    include-all LOW-LATENCY HIGH-CAPACITY
   advertise-definition
```
> Only P1 configures `advertise-definition`; all other participating routers (P2, P3, RR1, RR2, ASBR-A) configure the matching `flex-algo 128` block *without* `advertise-definition` so they learn/accept P1's advertised definition and merely participate.

**Participation + Flex-Algo-128 prefix-SID — all 6 L2 routers, representative P1 (index 128021, dedicated SRGB 24000-27999):**
```
segment-routing
 global-block 16000 23999
 srgb 24000 27999                       ! disjoint range dedicated to algorithm 128 — IOS-XR requires per-algo SRGB if not using the default algo-0 block
!
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid algorithm 128 index 128021
   fast-reroute per-prefix algorithm 128    ! (ties into Section 4 — participation flag)
```
- P1: index 128021 → absolute SID (24000+128021 is out of range for a flat SRGB; per the task's own numbering of "index = 128000+X" treat this as a *global absolute SID* assignment, not an SRGB-relative index) — on IOS-XR, when the absolute SID exceeds a single practical SRGB window, allocate it as an **absolute** SID instead of an index:
```
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid algorithm 128 absolute 128021
```
- P2: `absolute 128022`, P3: `absolute 128023`, RR1: `absolute 128311`... (continue `128000 + last-loopback-octet` per task wording, e.g. RR1 loopback .11 → `128011`, RR2 → `128012`, ASBR-A → `128031`).

**Documented approach:** this lab uses **absolute** Flex-Algo prefix-SIDs (not index-relative) specifically because `128000+X` does not cleanly fit inside a single contiguous SRGB without reserving an enormous disjoint block; the disjoint `24000-27999` SRGB is retained for any Flex-Algo-128 SIDs you choose to allocate by index on smaller deployments, but the explicit absolute values above are what's used here to match the task's literal numbering.

**Explanation (include-all vs exclude-any):**
- `include-all LOW-LATENCY HIGH-CAPACITY` means a link is eligible for algorithm 128's SPF **only if it carries both** bits set. In this topology, no single physical link carries both affinities simultaneously (P1-P2/P2-P3 only carry LOW-LATENCY; P1-RR1/P3-ASBR-A only carry HIGH-CAPACITY) — so the Flex-Algo-128 topology is **empty/disconnected** by design, visibly demonstrating the strict-subset nature of include-all versus exclude-any (which would have pruned only explicitly excluded links, leaving most of the topology intact).
- This is intentional in the task to contrast the two models — document in your lab notes that an administrator must tag at least one link with both bits for Algorithm 128 to carry live traffic; as configured, it validates the *mechanism*, not a usable path.

**Verify:**
```
show isis fast-reroute summary
show segment-routing traffic-eng ipv4 topology algorithm 128
show route algorithm 128
show isis database verbose | include "Flex-Algo|Affinity"
```

---

### Task 1.4 — SR-TE Explicit Adjacency-SID Policy — SOLUTION

**Step 1 — enable adj-SID advertisement (default-on, but make explicit) on every L2-backbone core-facing interface of P1, P2, P3, RR1, RR2:**
```
router isis CORE
 interface Gi0/0/0/2
  address-family ipv4 unicast
   adjacency-sid                 ! advertise, do not suppress
```
(Repeat per core-facing interface on P1, P2, P3, RR1, RR2 — omit the `suppress` keyword everywhere.)

**Step 2 — record adjacency-SIDs:**
```
show isis adjacency detail
show isis database verbose | include "Adjacency-SID|Local Adj"
```
Example recorded values (yours will differ per EVE-NG allocation — record and substitute):
- P1 → P2 adj-SID: `24001`
- P2 → P3 adj-SID: `24002`
- PE1 → P1 adj-SID: `24010`
- P1 → RR1 adj-SID: `24011`
- RR1 → ... (failover path) adj-SID: `24012`

**Step 3/4 — PE1 SR-TE policy `PE1-TO-PE4-EXPLICIT`:**
```
segment-routing
 traffic-eng
  policy PE1-TO-PE4-EXPLICIT
   color 100 end-point ipv4 10.0.0.4
   candidate-paths
    preference 200
     explicit segment-list PRIMARY-ADJSID
    !
    preference 100
     explicit segment-list BACKUP-VIA-RR1
  !
  segment-list PRIMARY-ADJSID
   index 10 mpls label 24010      ! PE1->P1 adj-SID
   index 20 mpls label 24001      ! P1->P2 adj-SID
   index 30 mpls label 24002      ! P2->P3 adj-SID
   index 40 mpls label 24020      ! P3->PE4 adj-SID (recorded value)
  !
  segment-list BACKUP-VIA-RR1
   index 10 mpls label 24010      ! PE1->P1 adj-SID
   index 20 mpls label 24011      ! P1->RR1 adj-SID
   index 30 mpls label 24013      ! RR1->P3 adj-SID (recorded value)
   index 40 mpls label 24020      ! P3->PE4 adj-SID
```

**Step 5 — steer via explicit color community (not autoroute):**
```
route-policy COLOR-100-SET
 set extcommunity color color-100
end-policy
!
route-policy COLOR-100-SET-VALUE
 set extcommunity color (100, 0)
end-policy
!
router bgp 65000
 address-family vpnv4 unicast
 !
 vrf CUST-A
  address-family ipv4 unicast
   ! on PE4 (the originator), apply the color-setting policy to the redistributed/advertised prefixes:
   redistribute connected route-policy COLOR-100-SET-VALUE
```
On PE1 (the consumer), steering into the policy is **automatic once a route carries the color-100 extended-community and the policy with matching color+endpoint exists** — no autoroute `install-on-color` is used; this is standard color-based (explicit) steering, matching VPNv4 next-hop = PE4 (the policy endpoint) + color 100 to the local `PE1-TO-PE4-EXPLICIT` policy. No `autoroute announce` or `autoroute include-ipv4` is configured anywhere.

**Step 6 — failover validation:** shut the P1-P2 link.
```
interface Gi0/0/0/2
 shutdown
```
Because both candidate-paths are **explicit** (not dynamic/CSPF), the headend does NOT recompute the segment-list on failure. Instead, PE1 detects the primary candidate-path's segment-list is invalid (an adj-SID in the list no longer resolves because the P1-P2 adjacency is down), marks preference-200 path down, and falls back to the preference-100 `BACKUP-VIA-RR1` path. Traffic resumes via P1→RR1→P3→PE4. If BOTH candidate-paths had been dynamic, CSPF would have silently recomputed a fresh SPT instead of needing a pre-built backup — this is the operational distinction being tested.

**Verify:**
```
show segment-routing traffic-eng policy name PE1-TO-PE4-EXPLICIT detail
show segment-routing traffic-eng forwarding policy name PE1-TO-PE4-EXPLICIT
show isis database verbose | include Adjacency-SID
```
After the shut: `show segment-routing traffic-eng policy name PE1-TO-PE4-EXPLICIT detail` should show the active candidate-path transitioned from preference 200 to preference 100.

---

### Task 1.5 — SRv6 New Locator Assignments — SOLUTION

**Locator blocks:** `fcff:0:0::/48`; PEs `fcff:0:0:1XX::/64`, P routers `fcff:0:0:2XX::/64`, RR/ASBR `fcff:0:0:3XX::/64`. End SID = `::1` suffix in-block.

**PE1:**
```
ipv6 unicast-routing
!
segment-routing srv6
 locator PE1-LOC
  prefix fcff:0:0:101::/64
!
interface Loopback0
 ipv6 address fcff:0:0:101::1/128
!
router isis CORE
 address-family ipv6 unicast
  segment-routing srv6
   locator PE1-LOC
 !
 interface Loopback0
  address-family ipv6 unicast
```
(PE2 → `fcff:0:0:102::/64`, PE3 → `fcff:0:0:103::/64`, PE4 → `fcff:0:0:104::/64` — same pattern.)

**P1:**
```
ipv6 unicast-routing
!
segment-routing srv6
 locator P1-LOC
  prefix fcff:0:0:221::/64
!
interface Loopback0
 ipv6 address fcff:0:0:221::1/128
!
router isis CORE
 address-family ipv6 unicast
  segment-routing srv6
   locator P1-LOC
 !
 interface Loopback0
  address-family ipv6 unicast
 !
 interface Gi0/0/0/2
  address-family ipv6 unicast
```
(P2 → `fcff:0:0:222::/64`, P3 → `fcff:0:0:223::/64`.)

**RR1:**
```
segment-routing srv6
 locator RR1-LOC
  prefix fcff:0:0:311::/64
!
interface Loopback0
 ipv6 address fcff:0:0:311::1/128
!
router isis CORE
 address-family ipv6 unicast
  segment-routing srv6
   locator RR1-LOC
 !
 interface Loopback0
  address-family ipv6 unicast
```
(RR2 → `fcff:0:0:312::/64`, ASBR-A → `fcff:0:0:331::/64`.)

**End.X SIDs** are generated automatically per core-facing IS-IS IPv6 adjacency once `segment-routing srv6` + `address-family ipv6 unicast` is enabled under each interface on P1-P3/RR1-RR2 — no separate manual End.X allocation command is required on IOS-XR; they derive from the locator block's dynamic allocation pool (confirm with `show segment-routing srv6 sid` — look for `End.X` type entries scoped to each router's own `/64`).

**Explanation:**
- The locator's `::1` End SID is the Node-SID equivalent for SRv6 (global reachability into the router).
- End.X SIDs (adjacency equivalent) are per-link and remain inside the owning router's own locator block — e.g., P1's End.X for the P1-P2 adjacency is allocated from `fcff:0:0:221::/64`, never from P2's block.
- SR-MPLS (label-based, Tasks 1.2-1.4) and SRv6 (IPv6-based, this task) are entirely separate data planes — SR-MPLS forwards on the label stack in the MPLS header; SRv6 forwards on the IPv6 destination address / SRH. They can coexist on the same links with zero collision because one consumes MPLS label space and the other consumes IPv6 address space.

**Verify:**
```
show segment-routing srv6 locator detail
show segment-routing srv6 sid
show isis segment-routing srv6 all
show isis database verbose | include "SRv6|Locator"
```

---

### Task 1.6 — SR-ODN with ECMP Margin — SOLUTION

**ODN template on PE1-PE4 (representative PE1), dynamic path + Flex-Algo 128 constraint + 20% ECMP margin:**
```
segment-routing
 traffic-eng
  on-demand color 100
   dynamic
    metric-type te
    algorithm 128
   !
   ecmp-margin percent 20
```
> On IOS-XR, the ECMP margin for dynamic/PCC-computed paths is a path-option-level knob (`ecmp-margin percent <N>` under the dynamic path-option or the global ODN template, depending on release). If your running image only exposes it under `explicit`/`dynamic` sub-mode, nest it accordingly:
```
segment-routing
 traffic-eng
  on-demand color 100
   dynamic
    metric-type te
    algorithm 128
    ecmp-margin percent 20
```

**Trigger — PE4 advertises a VPNv4 prefix tagged color 100:**
```
route-policy SET-COLOR-100
 set extcommunity color (100, 0)
end-policy
!
router bgp 65000
 vrf CUST-A
  address-family ipv4 unicast
   redistribute connected route-policy SET-COLOR-100
```
PE1, upon receiving this VPNv4 prefix via RR1/RR2 with color-100 and no matching manually-configured SR-TE policy to PE4, auto-instantiates an ODN SR-TE policy using the `on-demand color 100` template — endpoint = PE4 (the BGP next-hop), algorithm 128 constrained, dynamic path.

**Explanation — ECMP margin vs strict equal-cost:**
- Strict ECMP only installs paths whose metric is **exactly equal** to the best path's metric.
- An ECMP margin of 20% instructs the path computation to also install any path whose metric is within 20% of the best metric (`best_metric * 1.20`) as an additional load-balanced SID-list, even though it is not perfectly equal-cost.
- This is valuable in a Flex-Algo-constrained topology where asymmetric link metrics are common (because the constrained subgraph may have only slightly unequal paths that would otherwise be artificially excluded from load-balancing by strict ECMP, wasting usable capacity).

**Verify:**
```
show segment-routing traffic-eng policy color 100 detail
show segment-routing traffic-eng odn-template
show cef vrf CUST-A 192.168.4.0/24 detail
```
Confirm the auto-instantiated policy's forwarding entry (`show segment-routing traffic-eng forwarding policy color 100 endpoint 10.0.0.4`) lists **multiple** weighted SID-lists if more than one path candidate falls within the 20% margin.

---
## Section 2 — Services: EVPN, Inter-AS, QoS, L3VPN

### Task 2.1 — EVPN New EVI Numbers and AC IDs — SOLUTION

**EVI 4001 = VPWS (PE2↔PE3), AC-IDs 4001001/4001002. EVI 4002 = ELAN (PE1/PE2/PE4), ESI `0000.0000.0000.4002.02`, single-active on PE1/PE2.**

**Control plane — enable l2vpn evpn AF on existing MP-BGP (RR1/RR2 and all PEs). RR1 (representative):**
```
router bgp 65000
 address-family l2vpn evpn
 !
 neighbor-group RRC-EVPN
  remote-as 65000
  update-source Loopback0
  address-family l2vpn evpn
   route-reflector-client
 !
 neighbor 10.0.0.1
  use neighbor-group RRC-EVPN
 neighbor 10.0.0.2
  use neighbor-group RRC-EVPN
 neighbor 10.0.0.3
  use neighbor-group RRC-EVPN
 neighbor 10.0.0.4
  use neighbor-group RRC-EVPN
```
On each PE, add the EVPN AF to the existing RR session:
```
router bgp 65000
 address-family l2vpn evpn
 !
 neighbor 10.0.0.11
  address-family l2vpn evpn
 neighbor 10.0.0.12
  address-family l2vpn evpn
```

**EVI 4001 — EVPN VPWS point-to-point. PE2:**
```
l2vpn
 xconnect group EVPN-VPWS
  p2p PE2-PE3-VPWS
   interface Gi0/0/0/5                 ! AC toward local CE
   neighbor evpn evi 4001 target 4001002 source 4001001
```
**PE3:**
```
l2vpn
 xconnect group EVPN-VPWS
  p2p PE3-PE2-VPWS
   interface Gi0/0/0/5
   neighbor evpn evi 4001 target 4001001 source 4001002
```
> `source` = local AC-ID, `target` = remote AC-ID. PE2 source 4001001/target 4001002; PE3 mirrors.

**EVI 4002 — EVPN ELAN (multipoint) across PE1, PE2, PE4, with single-active multihoming on PE1/PE2.**

PE1 (multihomed, single-active):
```
evpn
 interface Bundle-Ether1
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.40.02.02
   load-balancing-mode single-active
!
l2vpn
 bridge group BG-4002
  bridge-domain BD-4002
   interface Bundle-Ether1
   evi 4002
!
interface Bundle-Ether1
 ! bundle facing the multihomed CE access
```
PE2 (second leg of the same ES — identical ESI, single-active):
```
evpn
 interface Bundle-Ether1
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.40.02.02
   load-balancing-mode single-active
!
l2vpn
 bridge group BG-4002
  bridge-domain BD-4002
   interface Bundle-Ether1
   evi 4002
```
PE4 (single-homed member of EVI 4002):
```
l2vpn
 bridge group BG-4002
  bridge-domain BD-4002
   interface Gi0/0/0/6
   evi 4002
```
> ESI `0000.0000.0000.4002.02` is written in IOS-XR type-0 nibble form as `00.00.00.00.00.00.40.02.02` (the task's dotted form maps to the 9-octet ESI value after the type byte). Both PE1 and PE2 configure the **identical** ESI so the control plane recognizes them as the two legs of one Ethernet Segment.

**Explanation:**
- `single-active` → only the Designated Forwarder (DF) elected for the ES forwards BUM/known-unicast toward the CE at any time; the non-DF PE blocks its AC. All-active (not used here) would hash flows across both PEs.
- Reusing the VPNv4 RR infra for `l2vpn evpn` keeps a single BGP overlay — no separate EVPN RR.
- Route types: Type-1 (Ethernet Auto-Discovery, per-ES and per-EVI, used for fast withdrawal/aliasing), Type-2 (MAC/IP advertisement), Type-3 (Inclusive Multicast, for BUM replication in ELAN), Type-4 (Ethernet Segment route, for DF election among PE1/PE2).

**Verify:**
```
show evpn evi detail
show evpn evi vpn-id 4002 detail
show evpn ethernet-segment detail           ! confirm DF election + single-active
show bgp l2vpn evpn
show l2vpn xconnect group EVPN-VPWS detail   ! EVI 4001 state = UP
```

---

### Task 2.2 — Inter-AS Option C — SOLUTION

**Model:** eBGP IPv4+label (BGP-LU) ASBR-A↔ASBR-B exchanging only /32 loopbacks; multihop eBGP VPNv4 RR1↔remote-RR with `next-hop-unchanged`; ASBRs are pure LSRs (zero VPNv4 state).

**ASBR-A (AS 65000) — BGP-LU to ASBR-B + advertise loopbacks needed for the end-to-end LSP:**
```
route-policy BGP-LU-OUT
 pass
end-policy
!
route-policy BGP-LU-IN
 pass
end-policy
!
router bgp 65000
 address-family ipv4 unicast
  network 10.0.0.31/32                  ! ASBR-A own loopback
  network 10.0.0.11/32                  ! RR1 loopback (so remote AS can reach our RR)
  network 10.0.0.12/32                  ! RR2 loopback
  allocate-label all                    ! turn IPv4 unicast into labeled-unicast
 !
 neighbor 10.9.0.2                       ! ASBR-B
  remote-as 65100
  address-family ipv4 labeled-unicast
   route-policy BGP-LU-IN in
   route-policy BGP-LU-OUT out
```
> IOS-XR implements BGP-LU via `address-family ipv4 unicast` + `allocate-label all`, with the neighbor AF declared as `ipv4 labeled-unicast`. The PE loopbacks (10.0.0.1-4) must also be reachable+labeled — redistribute them from IS-IS/SR into BGP-LU or add explicit `network` statements for each PE loopback that must be reachable across the inter-AS boundary. Simplest: add `network 10.0.0.1/32 ... 10.0.0.4/32` as well, or `redistribute isis CORE route-policy <PE-loopbacks-only>`.

**ASBR-B (AS 65100) — mirror:**
```
router bgp 65100
 address-family ipv4 unicast
  network 10.0.0.32/32                  ! ASBR-B own loopback
  network <remote-RR-loopback>/32
  allocate-label all
 !
 neighbor 10.9.0.1
  remote-as 65000
  address-family ipv4 labeled-unicast
   route-policy BGP-LU-IN in
   route-policy BGP-LU-OUT out
```

**Multihop eBGP VPNv4 RR1 (AS 65000) ↔ remote-RR (AS 65100):**
```
router bgp 65000
 neighbor <remote-RR-loopback>
  remote-as 65100
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-unchanged
```
> `next-hop-unchanged` is the Option C linchpin: RR1 passes PE next-hops (10.0.0.1-4) to the remote RR **unmodified**, so remote PEs resolve the VPNv4 next-hop over the end-to-end BGP-LU LSP that transits the ASBRs as plain LSRs. The RR must also carry the color/label infra; this VPNv4 session rides the BGP-LU LSP because the remote-RR loopback is reachable+labeled end-to-end.

**ASBRs carry ZERO VPNv4 routes** — they are never configured with `address-family vpnv4 unicast` toward each other; only `ipv4 labeled-unicast`. This is what makes it Option C (not Option B, where ASBRs would hold and re-originate VPNv4 NLRI).

**Explanation:**
- Option C decouples the VPN control plane (RR-to-RR, end-to-end) from the inter-AS label-transport plane (ASBR BGP-LU). It scales best for many VPNs because ASBRs don't hold per-VPN state.
- The end-to-end LSP is built by stitching: PE→(SR/IGP label)→ASBR-A→(BGP-LU label)→ASBR-B→(SR/IGP label)→remote-PE.

**Verify:**
```
show bgp ipv4 labeled-unicast
show bgp ipv4 labeled-unicast 10.0.0.1/32      ! remote PE loopback carries a label
show mpls forwarding
show bgp vpnv4 unicast summary                 ! RR1 shows the multihop neighbor UP
show bgp vpnv4 unicast                          ! on ASBR-A/B: EMPTY (proves not Option B)
ping mpls ipv4 <remote-PE-loopback>/32
traceroute mpls ipv4 <remote-PE-loopback>/32
```

---

### Task 2.3 — QoS Ingress Marking + Core EXP Treatment — SOLUTION

**PE ingress (CE-facing): classify by DSCP, re-mark MPLS EXP at imposition (EF→EXP5, AF41→EXP3, default→EXP0). Core: classify ONLY on EXP, LLQ EXP5 / min-BW EXP3 / WRED EXP0.**

**PE1-PE4 ingress policy (representative PE1):**
```
class-map match-any VOICE
 match dscp ef
 end-class-map
!
class-map match-any VIDEO
 match dscp af41
 end-class-map
!
policy-map PE-INGRESS-MARK
 class VOICE
  set mpls experimental imposition 5
 !
 class VIDEO
  set mpls experimental imposition 3
 !
 class class-default
  set mpls experimental imposition 0
 end-policy-map
!
interface Gi0/0/0/5                       ! CE-facing
 service-policy input PE-INGRESS-MARK
```
> `set mpls experimental imposition` marks the EXP of the label(s) pushed at this ingress PE — exactly "re-mark the EXP of the imposed label stack."

**Core policy on P1-P3 + RR1-RR2 core-facing interfaces — classify on EXP only:**
```
class-map match-any CORE-VOICE
 match mpls experimental topmost 5
 end-class-map
!
class-map match-any CORE-VIDEO
 match mpls experimental topmost 3
 end-class-map
!
policy-map CORE-EXP-OUT
 class CORE-VOICE
  priority level 1
  police rate percent 30
 !
 class CORE-VIDEO
  bandwidth percent 40
 !
 class class-default
  random-detect default
 end-policy-map
!
interface Gi0/0/0/2                       ! core-facing
 service-policy output CORE-EXP-OUT
```

**Explanation:**
- Core class-maps match **only** `mpls experimental topmost` — no `match dscp` anywhere in the core, satisfying the "core treats EXP only, never inspects IP DSCP" constraint.
- `priority level 1` = LLQ for voice (EXP5); `bandwidth percent 40` = guaranteed minimum for video (EXP3); `random-detect` = WRED on best-effort (EXP0).
- EXP is set once at ingress imposition and read at every core hop — the IP header's DSCP is never re-examined inside the MPLS core (the label hides it).

**Verify:**
```
show policy-map interface Gi0/0/0/5 input          ! PE ingress — marking counters per class
show policy-map interface Gi0/0/0/2 output         ! core — EXP-based queue counters
show mpls forwarding detail
show qos interface Gi0/0/0/2 output
```
Send EF + AF41 from CE1; confirm EXP5/EXP3 counters increment at PE1 ingress and the matching core queues increment at P1/P2.

---

### Task 2.4 — L3VPN redistribute connected (not network) — SOLUTION

**VRF CUST-A on PE1 and PE3; inject CE-facing connected subnet via `redistribute connected` + filtering route-policy.**

**PE1:**
```
vrf CUST-A
 address-family ipv4 unicast
  import route-target
   65000:100
  export route-target
   65000:100
!
route-policy CONN-FILTER-CUST-A
 if destination in (192.168.1.0/24 le 32) then
  pass
 else
  drop
 endif
end-policy
!
interface Gi0/0/0/5
 vrf CUST-A
 ipv4 address 192.168.1.1 255.255.255.252
!
router bgp 65000
 vrf CUST-A
  rd 65000:1
  address-family ipv4 unicast
   redistribute connected route-policy CONN-FILTER-CUST-A
```

**PE3:**
```
vrf CUST-A
 address-family ipv4 unicast
  import route-target
   65000:100
  export route-target
   65000:100
!
route-policy CONN-FILTER-CUST-A
 if destination in (192.168.3.0/24 le 32) then
  pass
 else
  drop
 endif
end-policy
!
interface Gi0/0/0/5
 vrf CUST-A
 ipv4 address 192.168.3.1 255.255.255.252
!
router bgp 65000
 vrf CUST-A
  rd 65000:3
  address-family ipv4 unicast
   redistribute connected route-policy CONN-FILTER-CUST-A
```

**Explanation / operational risk:**
- `redistribute connected` auto-injects **every** connected interface placed in the VRF into VPNv4 — including a management or transit subnet you did not intend to advertise. The `route-policy CONN-FILTER-CUST-A` is therefore mandatory discipline: it explicitly permits only the intended CE subnet and drops everything else.
- Contrast with `network` statements, which advertise only the exact prefixes you list — adding a new VRF interface does nothing until you add a matching `network` line. With `redistribute connected`, a newly-added VRF interface is advertised **immediately and silently** unless the route-policy is updated to exclude it. Document this as the key risk: the failure mode is accidental over-advertisement, not under-advertisement.

**Verify:**
```
show bgp vrf CUST-A ipv4 unicast
show route vrf CUST-A
show route-policy CONN-FILTER-CUST-A
show bgp vpnv4 unicast rd 65000:1            ! on RR — connected subnet present with RT 65000:100
```
On remote PE: `show route vrf CUST-A 192.168.1.0/30` should resolve via VPNv4 with the correct RT.

---
## Section 3 — SoO and Troubleshooting MCQs

### Task 3.1 — Site-of-Origin Scenario — SOLUTION

**Scenario:** Nimbus multihomed PE2+PE4, SoO `65000:4002` (PE2 site) / `65000:4004` (PE4 site), plus an OSPF sham-link backdoor between the two VRF sites.

**PE2 — VRF NIMBUS, SoO on the CE-facing BGP neighbor (or OSPF redistribution point):**
```
vrf NIMBUS
 address-family ipv4 unicast
  import route-target
   65000:400
  export route-target
   65000:400
!
router bgp 65000
 vrf NIMBUS
  rd 65000:2
  address-family ipv4 unicast
   redistribute connected
  !
  neighbor 192.168.21.2                 ! CE at PE2's Nimbus site
   remote-as 65521
   address-family ipv4 unicast
    site-of-origin 65000:4002
```

**PE4 — VRF NIMBUS, SoO `65000:4004`:**
```
vrf NIMBUS
 address-family ipv4 unicast
  import route-target
   65000:400
  export route-target
   65000:400
!
router bgp 65000
 vrf NIMBUS
  rd 65000:4
  address-family ipv4 unicast
   redistribute connected
  !
  neighbor 192.168.41.2                 ! CE at PE4's Nimbus site
   remote-as 65521
   address-family ipv4 unicast
    site-of-origin 65000:4004
```

**Loop-prevention effect:** a route learned from the PE2 CE neighbor is tagged with SoO `65000:4002` as it enters BGP. BGP's SoO rule (RFC 4364 §12) automatically suppresses re-advertisement of any route carrying a given SoO value back out to **any** eBGP neighbor in the VRF that is itself configured with that same SoO value — so PE2 will never hand Nimbus-site-2's own routes back to the PE2 CE, while PE4 (different SoO) still receives and advertises them normally to its own Nimbus CE.

**OSPF PE-CE with sham-link (backdoor-link scenario):**
```
! VRF loopback used as sham-link endpoint (reachable only via BGP/MPLS core, NOT via the OSPF backdoor)
interface Loopback100
 vrf NIMBUS
 ipv4 address 10.100.2.2 255.255.255.255
!
router ospf 100 vrf NIMBUS
 area 0
  interface Loopback100
   passive
  !
  sham-link 10.100.2.2 10.100.4.4
```
On PE4:
```
interface Loopback100
 vrf NIMBUS
 ipv4 address 10.100.4.4 255.255.255.255
!
router ospf 100 vrf NIMBUS
 area 0
  interface Loopback100
   passive
  !
  sham-link 10.100.4.4 10.100.2.2
```
> The sham-link endpoint loopbacks (10.100.2.2/10.100.4.4) must be advertised into **BGP VPNv4** (via `redistribute connected` under the VRF BGP AF, already in place from the base config above) and specifically **excluded from OSPF** so they are reachable only across the MPLS/BGP core — this is why the sham-link "works" at all: without it, OSPF would prefer the physical backdoor link between the two Nimbus CE sites as an intra-area path (lower cost than the "external" BGP-learned VPN route), causing the PE-CE OSPF routes to never traverse the VPN backbone as intended. The sham-link forces the two sites to see each other as a single OSPF area over the MPLS core, matching the BGP-based VPN path's preference.

**Loop validation:** if the backdoor link between the two Nimbus CE sites and both PE uplinks are all up simultaneously, no loop occurs because:
1. SoO prevents PE2/PE4 from re-advertising a site's own BGP-learned routes back to that same site.
2. The OSPF sham-link makes OSPF itself treat the VPN backbone as an intra-area (not external) path, so OSPF's own loop-prevention (standard SPF, no redistribution loop) applies cleanly — the sham-link exists precisely to avoid the backdoor link being preferred over (and conflicting with) the VPN path, not to create a second redundant forwarding path that could loop.

**Verify:**
```
show bgp vrf NIMBUS ipv4 unicast 192.168.21.0/30 detail   ! confirm extcommunity SoO:65000:4002
show ospf vrf NIMBUS sham-links
show route vrf NIMBUS
```

---

### Task 3.2 — Troubleshooting MCQ Answers

**1. (G.8032) RPL owner misconfigured on the wrong node — first symptom + confirm command:**
First symptom: a **traffic black hole or sub-optimal forwarding path** on the ring under normal (no-fault) conditions — specifically, the RPL (the link meant to stay blocked) does NOT block, so you either get a bridging loop (if two nodes both think they're *not* the owner and both unblock) or the wrong link ends up permanently blocked instead of the intended RPL. You typically see this surface as unexpected MAC flapping / duplicate traffic on the ring, not a failure-triggered symptom, because it's visible even with no link fault present.
Confirm command: `show ethernet ring g8032 detail` (or `show interfaces <if> ethernet ring`) — look at the `Node State` / `RPL Role` field per ring port to confirm which node/port is actually acting as RPL Owner vs RPL Neighbor vs normal ring port.

**2. (G.8032) SF reported but ring doesn't switch — three root causes, in check order:**
1. **R-APS messages are not actually being flooded/received by the other ring nodes** — check VLAN/control-VLAN membership for the R-APS channel (the ring's dedicated control VLAN) on every ring port; a misconfigured or missing R-APS VLAN means other nodes never learn about the SF.
2. **The RPL owner/neighbor is not correctly configured or the ring-id/instance on this node doesn't match the rest of the ring** — a mismatched `ring-id` or sub-ring/major-ring instance means this node's R-APS state machine never transitions, so it never unblocks the RPL.
3. **The port is not actually in forwarding-eligible G.8032 "ring port" role** (e.g., it's administratively excluded, or `wtr`/`guard timer` is still running from a prior event) — check the per-port ring-port role and current protection-state timers.

**3. (G.8032) Reverting from Manual Switch (MS) — command + risk:**
Command: `clear ethernet ring g8032 <ring-name>` (or the equivalent `no` form of the manual-switch command actually applied, e.g. removing the `manual-switch` CLI entry/triggering a `clear` on the ring instance) returns the ring to Idle state, re-evaluating RPL blocking normally.
Risk of forgetting: the ring remains **stuck in Manual Switch state indefinitely** — meaning if a *real* fault later occurs elsewhere on the ring, the ring's protection switching logic may not react correctly (since it believes an administrative override is still active), potentially leaving traffic black-holed on both the originally-forced-away link state AND the new fault, rather than cleanly protection-switching.

**4. (BGP NSR) Two most likely config gaps causing NSR to fail across RP failover:**
1. **NSR not actually enabled/supported for that specific address-family or session** — on IOS-XR, NSR has AF-specific support boundaries (and requires both RPs to be in sync `ready` state before failover); if the session/AF combination isn't NSR-capable on this platform/release, the session falls back to a hard reset instead of being preserved.
2. **The standby RP was not fully synchronized (BGP NSR "cold" / not-ready) at the moment of failover** — if the standby hadn't completed state replication (check `show bgp nsr` / sync status) when the active RP failed, there's no valid preserved session state to hand off, so the client PE sees a full session reset (`NBR_RESET`) instead of a seamless NSR switchover.

**5. (BGP NSR) Log signature — NSR success vs GR fallback:**
- **Successful NSR**: the BGP session on the RR shows **no `NBR_RESET` and no re-negotiation** — the TCP session and BGP FSM state are preserved across the RP switch; you'll see RP failover syslogs (`%HA-REDCON...`/RP switchover messages) but the peer's `show bgp neighbor` uptime does **not** reset, and no OPEN/UPDATE renegotiation occurs. The peer-side router sees zero impact.
- **GR fallback**: the client PE logs `%BGP-5-NBR_RESET` or `%BGP-5-ADJCHANGE` (session down then re-established), BGP uptime resets, and if Graceful Restart was negotiated you'll additionally see GR-specific messages (stale-route hold timer start, `(Restart Timer Expired)` or `graceful restart capability` renegotiation in the OPEN exchange) as the session is fully torn down and rebuilt with stale-path preservation rather than a true in-place continuation.

**6. (BGP NSR) show commands to confirm NSR (not GR) preserved the session:**
```
show bgp neighbor <peer> | include "NSR State|Up time"
show bgp nsr
```
Specific proving field: `NSR State: In sync` / `NSR State: NSR Ready` on the neighbor AND an **unbroken/continuing `Up time`** that did not reset across the failover timestamp. If GR had been used instead, `Up time` would show a value starting from *after* the failover (session was torn down and rebuilt), and the neighbor detail would show GR capability/stale-timer fields rather than an NSR-sync indicator.

---
## Section 4 — High Availability and Convergence

### Task 4.1 — SRv6 FRR with Priority-Limit Critical — SOLUTION

**Only prefixes tagged `critical` get a pre-computed TI-LFA backup; default-priority prefixes get none.**

**Enable TI-LFA for SRv6 + priority-limit critical — P1, P2, P3, RR1, RR2 (representative P1):**
```
router isis CORE
 address-family ipv6 unicast
  fast-reroute per-prefix
  fast-reroute per-prefix priority-limit critical
```

**Tag PE1 and PE4 loopbacks as critical priority (configured on PE1/PE4 themselves, propagated via IS-IS prefix attribute):**
```
! PE1
router isis CORE
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid algorithm 0            ! (base SRv6 locator End SID already configured in Task 1.5)
   fast-reroute per-prefix priority critical
```
```
! PE4 — identical pattern
router isis CORE
 interface Loopback0
  address-family ipv6 unicast
   fast-reroute per-prefix priority critical
```
PE2 and PE3's loopbacks are left at their IS-IS **default** priority (no `fast-reroute per-prefix priority` line at all) — they remain unprotected under this task's `priority-limit critical` policy on the L2 backbone routers.

**Validation — shut the P1-P2 core link:**
```
interface Gi0/0/0/2
 shutdown
```
Expected: PE1/PE4 traffic (critical-tagged) is immediately protected via the pre-computed SRv6 TI-LFA repair SID installed in the FIB *before* the failure — sub-50ms, data-plane-only reroute. PE2/PE3 traffic (default priority, not protected under the `priority-limit critical` policy) has no pre-installed backup and experiences a full control-plane IS-IS reconvergence (SPF recompute + FIB update), which is materially slower.

**Explanation:**
- `priority-limit critical` tells the TI-LFA computation engine: "only spend CPU/FIB resources precomputing backup paths for prefixes marked critical; skip backup computation entirely for everything else." This is a deliberate resource/scale tradeoff — on a router with a very large prefix table (e.g., full Internet or large VPN table), computing per-prefix TI-LFA backups for every single prefix is expensive in both CPU during SPF and FIB memory for the extra backup next-hops. Restricting FRR computation to only the operationally critical prefixes (such as backbone infra loopbacks used for high-value inter-AS LSPs) conserves those resources while still protecting what matters most.

**Verify:**
```
show isis fast-reroute 10.0.0.1/32 detail         ! PE1 — shows a pre-computed backup
show isis fast-reroute 10.0.0.2/32 detail         ! PE2 — shows NO backup (unprotected, by design)
show segment-routing srv6 sid detail
show isis fast-reroute summary
```

---

### Task 4.2 — OSPF NSF MCQ — SOLUTION

**Why NSF can't help when the CE doesn't support the helper role:**
OSPF NSF/Graceful Restart is a **cooperative, two-sided** mechanism. The restarting router (the PE, undergoing RP switchover) sends a Grace-LSA announcing "I'm restarting, please keep our adjacency up and continue forwarding using stale routes for N seconds." This only works if the **neighbor** (the CE) recognizes the Grace-LSA and enters **GR Helper mode** — holding the adjacency and continuing to use the last-known-good LSDB/routes instead of immediately tearing down and reconverging. If the CE does not implement the Opaque-LSA-based Grace-LSA / Helper capability (RFC 3623 for OSPFv2 GR, or the Cisco NSF-aware equivalent), it simply has no mechanism to interpret the Grace-LSA — from the CE's perspective, the PE's control plane blip looks like a normal neighbor-down event (missed Hellos / dead-timer expiry), so the CE tears down the adjacency and runs a full SPF/reconverge exactly as it would for any other failure. The specific missing capability is **OSPF Graceful-Restart Helper mode support** (the ability to process a received Grace-LSA and suppress normal adjacency teardown during the grace period) — without it on the CE side, NSF on the PE is necessarily a one-sided no-op for that adjacency, because NSF has no unilateral way to prevent the neighbor from treating the event as a failure.

---

## Section 5 — BGP Security

### Task 5.1 — TCP-AO Concepts and Keychain Config — SOLUTION

**1. (Design) MD5 weakness TCP-AO addresses:**
Plain BGP MD5 (RFC 2385) relies on the MD5 hash algorithm alone, which has well-documented cryptographic weaknesses (collision attacks, and MD5's digest is now considered too weak for new designs). It also uses a **single static key per session with no formal rotation mechanism** — changing the password requires a coordinated, often session-disruptive change on both routers simultaneously. TCP-AO (RFC 5925) mandates support for stronger, modern MAC algorithms (HMAC-SHA-1-96 as a baseline, AES-128-CMAC-96, etc.), explicitly supports **multiple concurrent keys with defined lifetimes** for coordinated/hitless rollover, and also protects TCP segments more robustly (covering more of the TCP header/options, with explicit replay protection via sequence-number-derived traffic keys) — closing both the "weak algorithm" and "no safe rotation" gaps in classic MD5.

**2. (Design) Why per-session MKT (SendID/RecvID) matters at scale:**
A router with many BGP peers using a single shared MD5 password has an all-or-nothing blast radius: rotating the key means touching every peer relationship simultaneously (or tolerating a mismatch window causing session flaps), and a single compromised password compromises every session using it. TCP-AO's Master Key Tuple with independent `SendID`/`RecvID` per session lets each peer relationship have its **own** key identity and its own independent rotation schedule — you can roll Key ID 1→2 on the PE1-CE1 session without touching any other peer's keys at all, and a compromise of one peer's key doesn't expose any other peer's session. This is essential operationally once you have more than a handful of peers requiring independent, staggered key-rotation windows.

**3/4. (Config concept) Keychain `BGP-KC-SEC` on PE1 with overlapping-lifetime keys, applied via TCP-AO (not legacy MD5 password):**
```
keychain BGP-KC-SEC
 key 1
  key-string password 7 <encrypted-key-1>
  cryptographic-algorithm HMAC-SHA1-96
  send-lifetime 00:00:00 january 1 2026 00:00:00 february 1 2026
  accept-lifetime 00:00:00 january 1 2026 00:00:00 february 1 2026
 !
 key 2
  key-string password 7 <encrypted-key-2>
  cryptographic-algorithm AES-128-CMAC-96
  send-lifetime 00:00:00 january 25 2026 00:00:00 march 1 2026
  accept-lifetime 00:00:00 january 25 2026 00:00:00 march 1 2026
!
router bgp 65000
 neighbor <CE1-ip>
  remote-as 65511
  ao BGP-KC-SEC
```
> `ao <keychain-name>` under the neighbor is IOS-XR's TCP-AO application command (distinct from the legacy `password encrypted ...` MD5 syntax). The overlap window (Jan 25 – Feb 1, 2026) is where **both** Key ID 1 and Key ID 2 are simultaneously valid for send/accept.

**5. Validation of hitless rollover:**
Advance the lab clock (or use short demonstration lifetimes, e.g., minutes instead of weeks) across the Key-1→Key-2 boundary and confirm via `show bgp neighbor <CE1-ip>` that the `Up time` counter does **not** reset and no `%BGP-5-ADJCHANGE`/`NBR_RESET` is logged during the transition — the session continues uninterrupted because both routers accept either key during the overlap, so whichever key each side happens to be using to send at the exact rollover instant is still a valid key for the other side to verify.

**6. (Design) Why overlapping lifetimes are mandatory, and the clock-sync risk:**
TCP-AO key lifetimes configured in **absolute** time require both routers to independently decide, from their own local clock, exactly when a key becomes valid/invalid for sending and receiving. If lifetimes were a hard single-timestamp cutover (Key 1 expires at exactly T, Key 2 starts at exactly T) with zero overlap, any clock skew between the two routers — even a few seconds — creates a window where one router is already sending with Key 2 while the other hasn't yet activated Key 2 for *accept* (or vice versa), causing authentication failures and a session drop. The overlap window absorbs realistic clock drift: as long as both routers' clocks are within the overlap duration of each other, there is always at least one key both sides agree is simultaneously valid. If NTP is not synchronized and the clock skew exceeds the configured overlap window, the keys' valid windows may not actually intersect in real wall-clock time on both routers, defeating the purpose of the overlap and potentially causing exactly the hard-cutover authentication failure/session flap the overlap was meant to prevent.

**Verify:**
```
show bgp neighbors <CE1-ip>
show keychain BGP-KC-SEC
show bgp neighbors <CE1-ip> detail | include Auth
```

---

## Section 6 — Automation: NSO, Ansible, Telemetry

### Task 6.1 — NSO Out-of-Sync Device — SOLUTION

**Simulate drift on PE2:**
```
! Directly on PE2 CLI, bypassing NSO:
interface Loopback0
 description EMERGENCY-FIX-oncall-$(date)
```

**Check-sync + diff from NSO CLI:**
```
devices device PE2 check-sync
devices device PE2 compare-config
```
`check-sync` reports `out-of-sync`; `compare-config` shows a diff limited to the `description` line under `interface Loopback0` that NSO's CDB doesn't know about.

**Decision — `sync-from` vs `sync-to`:**
Choose **`sync-from`** (pull the live device state into CDB), **not** `sync-to`. Justification: the drift was an **authorized emergency CLI change** made by an on-call engineer during a live incident — discarding it with `sync-to` would silently revert a deliberate, approved operational fix, which is both disruptive and a trust-breaking surprise for the on-call engineer. `sync-from` accepts the reality of the device as the new source of truth for anything NSO wasn't already managing as a service-owned leaf, which is the correct behavior for an authorized out-of-band change.
```
devices device PE2 sync-from
```

**Re-deploy the `CUST-A` L3VPN service with reconcile:**
```
services l3vpn CUST-A re-deploy reconcile
```
`re-deploy reconcile` re-applies the service template's intended configuration for everything the service **does** own (VRF name, RD, RT, redistribute-connected route-policy from Task 2.4) while explicitly reconciling (preserving, not reverting) configuration that exists on the device within the service's owned tree but that the service template itself didn't put there and doesn't manage line-by-line — in this case, the loopback description is outside the L3VPN service's owned configuration tree entirely (the service only touches the VRF/BGP/interface-VRF-binding config, not the loopback0 description), so it is untouched by either a plain `re-deploy` or a `re-deploy reconcile`. What `reconcile` specifically protects against is the service "fighting" extra device-level config *inside its own managed objects* that it didn't originate — ensuring the VRF/RT/RD still match the template exactly without erasing adjacent out-of-band changes elsewhere on the device.

**Explanation — plain re-deploy vs re-deploy reconcile:**
A plain `re-deploy` recalculates and pushes exactly what the service template says should exist, and can be more aggressive about removing device configuration that NSO associates with the service but no longer matches the template (potentially clobbering an out-of-band tweak that happened to land inside the service's owned config tree). `re-deploy reconcile` instructs NSO to first absorb (reconcile) any existing device configuration found within the service's owned tree into FASTMAP's view of "already applied," rather than treating it purely as drift to be purged — so legitimate out-of-band changes *inside the service's managed scope* survive the re-deploy instead of being silently reverted, while the service's core managed attributes (VRF/RT/RD here) are still verified/enforced to match the template.

**Verify:**
```
devices device PE2 check-sync
devices device PE2 compare-config
services l3vpn CUST-A re-deploy reconcile
show configuration services l3vpn CUST-A
```

---

### Task 6.2 — Ansible UnicodeDecodeError MCQ — SOLUTION

**Root cause:** The `UnicodeDecodeError` is almost certainly **not** a YAML syntax problem — it occurs because the BGP description field was copy-pasted from a non-UTF-8 source document (commonly Windows-1252/CP1252 from Word/Outlook, which can contain byte sequences such as smart quotes, em-dashes, or non-breaking spaces that are invalid or differently-encoded in UTF-8). `cisco.iosxr.iosxr_config` (and the underlying `ncclient`/Paramiko transport) assumes UTF-8 end-to-end when it reads the candidate configuration file/string and when it decodes the device's CLI response; a stray CP1252 byte sequence embedded in that description text causes the decode step to throw partway through the push.
**Setting to audit/correct:** the **source document/text encoding of the variable file or template feeding the task** — specifically, ensure the inventory/vars file (or the Jinja2 template rendering the BGP description) is saved and read as UTF-8, and sanitize/normalize any copy-pasted text (strip non-ASCII "smart" punctuation) before it enters the playbook variables. If the vars are loaded from an external file, explicitly set `encoding: utf-8` wherever Ansible's `lookup`/`file`/template read happens, rather than only "fixing the YAML," since the YAML itself can be syntactically perfect and still carry an invalid byte sequence in a string value.

---

### Task 6.3 — Telemetry Troubleshooting — SOLUTION

**Scenario:** P2 streams `Cisco-IOS-XR-fib-common-oper` via dial-out gRPC every 30s; subscription shows `State: Active`; collector sees nothing.

**Systematic isolation (in order):**
1. **Subscription state** — `show telemetry model-driven subscription <name> detail` on P2: confirms `State: Active` (per the scenario) — if this had shown `Invalid` or `No Memory`, that would already be the root cause; since it's Active, move on.
2. **Sensor-group path validity against the loaded YANG model** — `show telemetry model-driven sensor-group <name>` and cross-check the exact sensor-path string against the actual supported YANG paths for the running IOS-XR version (`show yang-data schema <path>` or the release's YANG model bundle). A sensor-path that was valid on a different IOS-XR version/release can silently fail to resolve or return empty data on this version if the model changed.
3. **Destination-group transport/encoding match** — `show telemetry model-driven destination <name>` on P2 versus the collector's actual listener configuration: confirm the dial-out transport (`grpc` vs `grpc-tls`) and encoding (`self-describing-gpb`, `gpb`, `json`) P2 is sending **exactly match** what the collector expects to receive on that port — a transport/encoding mismatch typically shows the session as "connected" at the TCP level but the collector discards/can't parse the stream, matching the "subscription Active, collector sees nothing" symptom.
4. **IP reachability / firewall to the collector's gRPC port** — basic `ping`/`traceroute` from P2 to the collector IP, and confirm no ACL/firewall is blocking the specific gRPC destination port in the dial-out direction.

**The specific distinguishing show command:**
```
show telemetry model-driven destination <name>
```
This reveals the **per-destination transport statistics** — specifically counters for `Total bytes sent`, `Last Sent time`, and any `collection errors`. If this shows bytes actively being sent with a recent `Last Sent` timestamp and no errors, P2 believes it IS successfully exporting — the fault is downstream (collector-side packet loss, wrong port, firewall, or an encoding the collector can't parse), so you troubleshoot the network path/collector next. If this instead shows zero bytes sent / no successful collection attempts ever recorded, P2 never attempted a real export — the fault is upstream of the wire, i.e., a sensor-path/YANG mismatch (step 2) that produced no data to send in the first place, so you troubleshoot on P2's sensor-group/model configuration instead.

**Resolution (most likely root cause for this scenario, matching "sensor-group was built against a different XR release"):** the sensor-path for `Cisco-IOS-XR-fib-common-oper` changed container structure between releases; correct the sensor-group to the path matching the currently loaded YANG model:
```
telemetry model-driven
 sensor-group FIB-STATS
  sensor-path Cisco-IOS-XR-fib-common-oper:fib-statistics-summary/nodes/node/afs/af/safs/saf/protos/proto/paths     ! corrected path matching the running release's model
```
Re-verify with `show telemetry model-driven destination <name>` showing non-zero `Total bytes sent` and the collector receiving data.

**Verify:**
```
show telemetry model-driven subscription <name> detail
show telemetry model-driven destination <name>
show grpc status
show yang-data schema <corrected-sensor-path>
```

---

## Completion Checklist

- [x] Section 1: IS-IS multi-level configured and verified (L1-only PEs, L1/L2 P-border, L2-only RR/ASBR); SR prefix-SID partition deployed with P1/P2 anycast-SID 16050; Flex-Algo 128 include-all (both affinity bits) configured and its empty/restrictive topology explained; SR-TE explicit adjacency-SID policy with primary/backup candidate-paths built and failover behavior explained; SRv6 locators assigned per the new per-role `/64` scheme; SR-ODN template with 20% ECMP margin configured.
- [x] Section 2: EVPN EVI 4001 (VPWS, PE2↔PE3) and EVI 4002 (ELAN, PE1/PE2/PE4, single-active ES) configured; Inter-AS Option C (BGP-LU ASBR-A↔ASBR-B + multihop VPNv4 RR-to-RR with next-hop-unchanged) configured with ASBRs carrying zero VPNv4 state; ingress-DSCP-marking/core-EXP-only QoS policies built per PE/P role; L3VPN CUST-A built with `redistribute connected` + explicit filtering route-policy and documented risk.
- [x] Section 3: SoO (`65000:4002`/`65000:4004`) + OSPF sham-link backdoor scenario configured with loop-prevention explained; all 6 troubleshooting MCQs (3 G.8032, 3 BGP NSR) answered.
- [x] Section 4: SRv6 TI-LFA FRR with `priority-limit critical` configured (PE1/PE4 loopbacks tagged critical and protected; PE2/PE3 left unprotected by design) with resource-conservation rationale explained; OSPF NSF MCQ answered (missing GR-Helper capability on CE).
- [x] Section 5: TCP-AO concept/design questions answered (MD5 weakness, per-session MKT rationale); keychain `BGP-KC-SEC` with HMAC-SHA1-96/AES-128-CMAC-96 overlapping-lifetime keys configured and applied via `ao`; hitless-rollover validation and clock-sync risk explained.
- [x] Section 6: NSO out-of-sync PE2 reconciled via `sync-from` (justified for an authorized emergency change) + `CUST-A` `re-deploy reconcile` explained; Ansible `UnicodeDecodeError` MCQ answered (non-UTF-8 source text, not YAML syntax); MDT/gRPC telemetry fault isolated via the 4-step order and the destination-stats show command that distinguishes export-side vs path-side failures, with corrected sensor-path example.
