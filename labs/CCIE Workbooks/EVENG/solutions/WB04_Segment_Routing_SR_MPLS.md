# CCIE SP Workbook 04 — Segment Routing SR-MPLS

**Domain:** 1 — Core Routing (25%)
**Platform:** Cisco IOS-XRv 9000 (EVE-NG / GNS3 on EC2)
**Primary focus:** **Garnet — AS 65200** (IS-IS L2 + SR-MPLS)
**Secondary focus:** **Emerald — AS 65100** (IS-IS L2 + LDP → SR migration)
**Format:** Question → Solution → Verification

---

## Topology Reference — Garnet AS 65200

| Node | Role | Loopback0 | Prefix-SID index (last octet) | Prefix-SID label (SRGB 16000) |
|------|------|-----------|-------------------------------|-------------------------------|
| Gar-R1 | PE | 11.11.11.11 | 11 | 16011 |
| Gar-R2 | PE | 12.12.12.12 | 12 | 16012 |
| Gar-R3 | P | 13.13.13.13 | 13 | 16013 |
| Gar-R4 | P | 14.14.14.14 | 14 | 16014 |
| Gar-R5 | P | 15.15.15.15 | 15 | 16015 |
| Gar-R7 | ASBR | 17.17.17.17 | 16 | 16016 |
| Gar-R6 | RR + Gar-R6 | 16.16.16.16 | 17 | 16017 |

**Garnet core links (from topology reference):**

```
Gar-R6   Gi0/0/0/3 --- 10.2.1.0/24 --- Gi0/0/0/3  Gar-R3
Gar-R7 Gi0/0/0/2 --- 10.2.2.0/24 --- Gi0/0/0/2  Gar-R3
Gar-R3    Gi0/0/0/0 --- 10.2.3.0/24 --- Gi0/0/0/0  Gar-R4
Gar-R3    Gi0/0/0/1 --- 10.2.4.0/24 --- Gi0/0/0/1  Gar-R5
Gar-R4    Gi0/0/0/3 --- 10.2.5.0/24 --- Gi0/0/0/3  Gar-R5
Gar-R4    Gi0/0/0/1 --- 10.2.6.0/24 --- Gi0/0/0/1  Gar-R1
Gar-R5    Gi0/0/0/2 --- 10.2.7.0/24 --- Gi0/0/0/2  Gar-R2
Gar-R1   Gi0/0/0/3 --- 10.2.8.0/24 --- Gi0/0/0/3  Gar-R2
```

**Emerald AS 65100 (LDP → SR migration section):** E-R1 (1.1.1.1), E-R2 (2.2.2.2), E-R3 (3.3.3.3), E-R4 (4.4.4.4), E-R6 (5.5.5.5), E-R5 (6.6.6.6 RR+Gar-R6). Prefix-SID index = last octet of loopback.

> **Prerequisites:** IS-IS Level-2 backbone with **wide metrics** (mandatory for SR sub-TLVs) and /32 loopbacks already configured (Workbook 01). Wide metrics carry the SR Prefix-SID/Adjacency-SID sub-TLVs; narrow metrics cannot.

---

# Section 1 — SR-MPLS Foundation (5 tasks)

## Task 1.1 — Enable Segment Routing under IS-IS

**Question:** Enable Segment Routing MPLS under IS-IS on all Garnet core/PE nodes and confirm the SR data plane is active. No LDP anywhere in Garnet.

**Solution**

SR reuses the MPLS data plane but distributes labels *through the IGP* — there is no separate label-distribution protocol. Enabling `segment-routing mpls` under the IS-IS address-family turns on the SR sub-TLVs so IS-IS floods Prefix-SIDs and Adjacency-SIDs.

```
! ==== All Garnet nodes (example Gar-R3 = 13.13.13.13) ====
router isis 1
 is-type level-2-only
 net 49.0002.0000.0000.0013.00
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
 !
!
```

**Verification**

```
show isis  ! confirm SR-MPLS enabled in AF ipv4
show mpls interfaces          ! SR-enabled interfaces listed
show isis segment-routing state
```

Expect `Segment-Routing: Enabled` under `address-family IPv4 Unicast`.

---

## Task 1.2 — Configure SRGB 16000–23999

**Question:** Set a common **SRGB of 16000–23999** on every Garnet node so prefix-SIDs are globally consistent.

**Solution**

The **SRGB (Segment Routing Global Block)** is the reserved local-label range from which each node maps global prefix-SID *indexes* to actual labels. Making the SRGB identical everywhere means `index 13` resolves to the same label `16013` on every node — this is what makes prefix-SIDs *globally* significant. The default XR SRGB is 16000–23999, but it is configured explicitly here to guarantee consistency and to demonstrate the mismatch troubleshooting scenario later.

```
! ==== All Garnet nodes ====
segment-routing
 global-block 16000 23999
!
```

> Changing the SRGB is disruptive — it forces a reallocation of all SR labels. In production, set it once at design time.

**Verification**

```
show mpls label range
show segment-routing local-block inout   ! SRLB (adj-SIDs) vs SRGB
show isis segment-routing label table
```

Expect `SRGB: 16000 - 23999` identical on all nodes.

---

## Task 1.3 — Prefix-SID index per loopback (match last octet)

**Question:** Assign each node's Loopback0 a **prefix-SID index equal to the last octet** of its loopback (Gar-R3=13 → label 16013, Gar-R6=17 → 16017, etc.). Achieve end-to-end SR reachability with **no LDP**.

**Solution**

The prefix-SID is advertised as an **index** (not an absolute label). Each receiving node computes `label = SRGB_base + index`. Using `absolute` would advertise the label directly; `index` is preferred because it survives differing SRGBs. Below, Gar-R3 gets index 13, Gar-R1 gets index 11, etc.

```
! ==== Gar-R3 (13.13.13.13) ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 13
  !
 !
!

! ==== Gar-R1 (11.11.11.11) ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 11
!

! ==== Gar-R6 (16.16.16.16) ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 17
!
```

Apply the matching index on Gar-R4 (14), Gar-R5 (15), Gar-R7 (16), Gar-R2 (12).

**Verification**

```
show isis segment-routing label table
show cef 11.11.11.11/32 detail     ! label 16011 imposed
show mpls forwarding
traceroute 12.12.12.12 source Loopback0   ! from Gar-R1 to Gar-R2 over SR labels
```

The label for `11.11.11.11/32` is `16011` on **every** node in Garnet (global significance).

---

## Task 1.4 — Verify the global label table & compare with LDP

**Question:** Show the SR global label table and explain how SR **prefix-SIDs (globally significant)** differ from **LDP labels (locally significant)**.

**Solution**

With LDP, each LSR allocates its *own* local label for a FEC and advertises it to neighbors; the same prefix has a *different* label on every hop, and downstream label bindings must be learned per neighbor. With SR, the prefix-SID index is flooded by IS-IS and every node derives the *same* label from the shared SRGB — one label identifies the destination network-wide, and forwarding is driven by the IGP SPF (no LDP session, no label-binding exchange, no LDP-IGP sync issues).

```
! Nothing new to configure — this task is verification/comparison.
```

**Verification**

```
show isis segment-routing label table
show mpls forwarding labels 16011
show mpls ldp neighbor        ! MUST be empty in Garnet — SR only
```

| Property | LDP label | SR prefix-SID |
|----------|-----------|---------------|
| Significance | Local (per-node) | **Global** (network-wide) |
| Distribution | LDP protocol | IS-IS sub-TLV |
| Label for a FEC | Different per hop | **Identical** everywhere |
| Sync dependency | LDP-IGP sync needed | None (one protocol) |

---

## Task 1.5 — Adjacency-SIDs (protected vs unprotected)

**Question:** Examine the **adjacency-SIDs** auto-allocated per IS-IS adjacency and explain **protected vs unprotected** adj-SIDs.

**Solution**

Adjacency-SIDs are **locally significant** labels each node auto-allocates per IGP adjacency out of the **SRLB (Segment Routing Local Block)**. They identify a *specific link* — a stack of adj-SIDs is pure source-routing. By default XR allocates a **protected** adj-SID (backed by TI-LFA so it survives link failure) *and* an **unprotected** adj-SID. You can force behavior per interface.

```
! ==== Force protected/unprotected adjacency-SID on an interface ====
router isis 1
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   adjacency-sid index 100 protected      ! protected adj-SID
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
   adjacency-sid index 200 unprotected    ! unprotected adj-SID
!
```

**Verification**

```
show isis segment-routing adjacency-sid
show mpls forwarding labels <adj-sid-label>
show segment-routing local-block inout    ! SRLB range for adj-SIDs
```

| | Prefix-SID | Adjacency-SID |
|--|-----------|---------------|
| Significance | Global | **Local** |
| Identifies | Shortest path to a prefix | A specific link/adjacency |
| Source | SRGB | SRLB |
| Protection | via TI-LFA | protected vs **unprotected** |

---

# Section 2 — TI-LFA (4 tasks)

## Task 2.1 — Enable per-prefix TI-LFA on all core interfaces

**Question:** Enable **TI-LFA (per-prefix)** on all Garnet IS-IS core interfaces.

**Solution**

TI-LFA (Topology-Independent Loop-Free Alternate) pre-computes a **post-convergence backup path** for every destination, expressed as a repair **segment list**, and pre-installs it in the FIB. Because the repair is a label stack, it is loop-free in *any* topology (unlike classic LFA/rLFA). Enable it under the IS-IS interface address-family.

```
! ==== All Garnet core interfaces (example Gar-R4) ====
router isis 1
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
!
```

**Verification**

```
show isis fast-reroute summary
show isis interface GigabitEthernet0/0/0/0 | include LFA
```

---

## Task 2.2 — Verify backup paths pre-computed in FIB

**Question:** Prove that TI-LFA backup paths are **pre-computed and installed in the FIB** before any failure.

**Solution**

No new config — this validates that the repair segment list and backup next-hop are already in CEF, so the switchover is pure hardware (no control-plane wait).

**Verification**

```
show isis fast-reroute 12.12.12.12/32 detail
show route 12.12.12.12/32 detail          ! backup path present
show cef 12.12.12.12/32 detail            ! primary + repair label stack
```

Look for `Repair:` / `backup` next-hop and a pushed repair label (the TI-LFA segment list) in `show cef ... detail`.

---

## Task 2.3 — Test link failure with continuous ping (sub-50ms)

**Question:** Run continuous CE-to-CE (or PE-to-PE) traffic, fail a primary Garnet core link, and confirm **sub-50ms / 0–1 packet** loss.

**Solution**

With the FIB repair pre-installed, a link-down event triggers an immediate local switchover to the backup segment list before IS-IS reconverges.

```
! ==== From Gar-R1, generate continuous traffic to Gar-R2 ====
ping 12.12.12.12 source Loopback0 count 100000 size 100

! ==== Fail the primary link (e.g., shut Gar-R4<->Gar-R5) ====
! On Gar-R4:
interface GigabitEthernet0/0/0/3
 shutdown
```

**Verification**

```
! Observe near-zero loss on the running ping (0-1 packets)
show isis fast-reroute summary              ! before/after
show logging | include LINK
```

Compare against disabling TI-LFA (multi-second loss during full IGP reconvergence).

---

## Task 2.4 — Node protection vs link protection

**Question:** Configure and contrast **node protection** vs **link protection** in TI-LFA.

**Solution**

Default TI-LFA gives **link protection** (repair assumes only the link failed). **Node protection** computes a backup that avoids the *entire next-hop node*, protecting against router failure — important where a P router is a single point through which many paths pass. Add `node-protection`; `tiebreaker` can prioritize it.

```
! ==== Enable node protection on a core interface (Gar-R3) ====
router isis 1
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker node-protecting index 100
  !
!
```

**Verification**

```
show isis fast-reroute 14.14.14.14/32 detail    ! repair avoids next-hop node
show isis fast-reroute summary
```

| | Link protection | Node protection |
|--|-----------------|-----------------|
| Protects against | Link failure | Entire next-hop **node** failure |
| Repair path avoids | Failed link | Failed link **and** neighbor node |
| Config | default TI-LFA | `tiebreaker node-protecting` |

---

# Section 3 — SR-TE Policies (5 tasks)

## Task 3.1 — Explicit SR-TE policy (segment-list of prefix-SIDs)

**Question:** On Gar-R1, build an **explicit SR-TE policy** to Gar-R2 (12.12.12.12) that forces a non-shortest path — e.g. Gar-R1 → Gar-R4 → Gar-R5 → Gar-R2 — using a segment-list of prefix-SIDs.

**Solution**

An explicit policy pins the path as an operator-defined segment-list (here, prefix-SIDs of the transit nodes). The head-end imposes the stack; transit routers are stateless. Steering is by **color + endpoint**.

```
! ==== Gar-R1 (11.11.11.11) ====
segment-routing
 traffic-eng
  segment-list SL-VIA-P4-P5
   index 10 mpls label 16014     ! Gar-R4
   index 20 mpls label 16015     ! Gar-R5
   index 30 mpls label 16012     ! Gar-R2 (endpoint)
  !
  policy Gar-R2-EXPLICIT
   color 100 end-point ipv4 12.12.12.12
   candidate-paths
    preference 100
     explicit segment-list SL-VIA-P4-P5
    !
   !
  !
 !
!
```

**Verification**

```
show segment-routing traffic-eng policy
show segment-routing traffic-eng policy color 100
traceroute 12.12.12.12 source Loopback0     ! follows Gar-R4->Gar-R5, not shortest path
```

Policy state should be `Admin: up  Operational: up` with the segment-list installed.

---

## Task 3.2 — Dynamic SR-TE policy (metric IGP/TE/latency)

**Question:** Create a **dynamic** SR-TE policy on Gar-R1 to Gar-R2 optimized by a chosen **metric type (igp / te / latency)**; observe recomputation when a metric changes.

**Solution**

A dynamic policy runs constrained SPF (locally or via Gar-R6) and installs the resulting segment list automatically. `metric type latency` optimizes for the accumulated link-delay metric (from performance-measurement or configured delay).

```
! ==== Gar-R1 ====
segment-routing
 traffic-eng
  policy Gar-R2-DYNAMIC
   color 200 end-point ipv4 12.12.12.12
   candidate-paths
    preference 100
     dynamic
      metric
       type latency          ! or 'igp' / 'te'
      !
     !
    !
   !
  !
 !
!
```

**Verification**

```
show segment-routing traffic-eng policy color 200 detail
show segment-routing traffic-eng policy color 200 | include Metric
! Change an IGP/TE/delay metric on a core link, then re-check:
show segment-routing traffic-eng policy color 200 detail   ! recomputed segment list
```

---

## Task 3.3 — Gar-R6-initiated policy (Gar-R6 = 16.16.16.16 as SR-PCE)

**Question:** Configure the Garnet **Gar-R6 (16.16.16.16)** as an **SR-PCE**, have Gar-R1 connect as a PCC, and have the Gar-R6 **initiate/delegate** an SR-TE policy.

**Solution**

The SR-PCE has a full topology view via **BGP-LS** and computes paths centrally over **PCEP**. On the Gar-R6 node run `pce address` + `pce segment-routing`; each head-end (PCC) points at the Gar-R6. Gar-R6-initiated policies are pushed from the controller.

```
! ==== Gar-R6 node (16.16.16.16) — SR-PCE + BGP-LS ====
router isis 1
 address-family ipv4 unicast
  distribute link-state                 ! feed topology to BGP-LS
 !
!
router bgp 65200
 address-family link-state link-state
 !
 neighbor 11.11.11.11
  remote-as 65200
  update-source Loopback0
  address-family link-state link-state
 !
!
pce
 address ipv4 16.16.16.16
 segment-routing
  traffic-eng
  !
 !
!

! ==== Gar-R1 (PCC) — connect to the SR-PCE ====
segment-routing
 traffic-eng
  pcc
   source-address ipv4 11.11.11.11
   pce address ipv4 16.16.16.16
    precedence 10
   !
   report-all
  !
 !
!
```

**Verification**

```
! On Gar-R6:
show pce ipv4 topology summary        ! BGP-LS topology learned
show pce lsp                          ! delegated/initiated LSPs
show pce ipv4 peer                    ! PCEP sessions up

! On PCC (Gar-R1):
show segment-routing traffic-eng pcc ipv4 peer
show segment-routing traffic-eng policy
```

---

## Task 3.4 — On-Demand Next-hop (ODN) with BGP color

**Question:** Configure **ODN** on Gar-R1 so an SR-TE policy is auto-created toward the BGP next-hop when a VPN route arrives carrying a matching **color** community.

**Solution**

ODN uses an **on-demand color template**: when a BGP route with color X is received, XR auto-creates an SR-TE policy `(color X, endpoint = BGP next-hop)` using the template's constraints — no per-prefix tunnel config. Scales to thousands of prefixes.

```
! ==== Gar-R1 — ODN template for color 100 ====
segment-routing
 traffic-eng
  on-demand color 100
   dynamic
    metric
     type latency
    !
   !
  !
 !
!
```

The color is attached to VPN routes on the far-end PE via a route-policy (see Task 3.5).

**Verification**

```
show segment-routing traffic-eng policy       ! auto-created ODN policy per color/endpoint
show bgp vpnv4 unicast <prefix>                ! carries color:100 ext-community
show cef vrf CUST-A <prefix>                   ! resolves via SR-TE policy
```

---

## Task 3.5 — Steer L3VPN traffic into an SR-TE policy

**Question:** Steer **L3VPN (VPNv4)** traffic from Gar-R2 → Gar-R1 into the color-100 SR-TE policy by coloring the VPN routes.

**Solution**

Set the **color extended community** on the egress PE's VPN routes with a route-policy applied outbound to VPNv4. The ingress PE (with the ODN template or explicit policy for that color) then resolves the VPN next-hop through the SR-TE policy — this is automated steering (`route-policy` + color), replacing RSVP-TE autoroute.

```
! ==== Gar-R2 (egress) — color the exported VPN routes ====
extcommunity-set opaque COLOR-100
  100
end-set
!
route-policy SET-COLOR-100
  set extcommunity color COLOR-100
  pass
end-policy
!
router bgp 65200
 neighbor 16.16.16.16            ! RR (Gar-R6) is also route-reflector
  address-family vpnv4 unicast
   route-policy SET-COLOR-100 out
  !
 !
!

! ==== Gar-R1 (ingress) — auto-steer via 'steering' (default per-flow on) ====
segment-routing
 traffic-eng
  on-demand color 100
   dynamic
    metric
     type latency
   !
  !
 !
!
```

**Verification**

```
show bgp vpnv4 unicast <prefix>                 ! Color:100 attached
show segment-routing traffic-eng policy color 100
show cef vrf CUST-A <prefix> detail             ! next-hop = SR-TE policy (via <label stack>)
traceroute vrf CUST-A <ce-prefix>               ! follows engineered path
```

---

# Section 4 — Flex-Algo (3 tasks)

## Task 4.1 — Define Flex-Algo 128 (delay metric) on Garnet

**Question:** Define **Flex-Algo 128** using the **delay (min-latency) metric** on all Garnet nodes and advertise the definition from a subset (definition can be flooded by one node but is typically defined consistently).

**Solution**

A Flex-Algo defines a custom constrained topology (metric-type + affinity/SRLG constraints). Nodes participating advertise a **Flex-Algo Definition (FAD)**. Algo 128 here uses `metric-type delay` so its SPF minimizes accumulated link delay rather than IGP cost.

```
! ==== All Garnet nodes ====
router isis 1
 flex-algo 128
  metric-type delay
  advertise-definition
 !
 address-family ipv4 unicast
  segment-routing mpls
 !
!

! Delay must be measurable — enable performance-measurement on core links:
performance-measurement
 interface GigabitEthernet0/0/0/0
  delay-measurement
 !
 interface GigabitEthernet0/0/0/1
  delay-measurement
 !
!
```

**Verification**

```
show isis flex-algo                  ! Algo 128 defined, metric-type delay
show isis flex-algo 128 detail
show performance-measurement summary
```

---

## Task 4.2 — Assign locator / prefix-SID per algo

**Question:** Advertise a **per-algorithm prefix-SID** for Loopback0 so Algo 128 has its own SID separate from the default Algo 0 SID.

**Solution**

Each node advertises an additional prefix-SID *for algorithm 128* on its loopback (offset from the base index so it does not collide with Algo-0 SIDs). Pushing the Algo-128 SID steers a packet onto the low-delay plane.

```
! ==== Gar-R3 (13.13.13.13) — algo-0 index 13, algo-128 index 213 ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 13                       ! algorithm 0 (default)
   prefix-sid algorithm 128 index 213        ! Flex-Algo 128
  !
 !
!

! ==== Gar-R1 (11.11.11.11) — algo-128 index 211 ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 11
   prefix-sid algorithm 128 index 211
!
```

Apply matching per-algo indexes (loopback-last-octet + 200) on Gar-R4, Gar-R5, Gar-R7, Gar-R2, Gar-R6.

**Verification**

```
show isis segment-routing label table
show isis segment-routing prefix-sid-map active-policy
show mpls forwarding labels 16211            ! algo-128 SID for Gar-R1 (16000+211)
```

---

## Task 4.3 — Verify separate topology computation & use for low-latency path

**Question:** Confirm Algo 128 computes a **separate (delay-optimized) topology** and steer latency-sensitive traffic onto it.

**Solution**

Algo 0 (IGP cost) and Algo 128 (delay) produce potentially different next-hops for the same destination. Steer traffic either by pushing the Algo-128 prefix-SID directly, or via an SR-TE / ODN policy referencing `constraints segments algorithm 128`.

```
! ==== Gar-R1 — SR-TE policy that uses Flex-Algo 128 path to Gar-R2 ====
segment-routing
 traffic-eng
  policy Gar-R2-LOWLATENCY
   color 128 end-point ipv4 12.12.12.12
   candidate-paths
    preference 100
     dynamic
      metric
       type latency
      !
     constraints
      segments
       sid-algorithm 128
      !
     !
    !
   !
  !
 !
!
```

**Verification**

```
show isis flex-algo 128 detail
show route 12.12.12.12/32                      ! algo-0 next-hop
show isis segment-routing prefix-sid-map active-policy
show segment-routing traffic-eng policy color 128
traceroute 12.12.12.12                         ! algo-128 path differs from IGP path
```

The Algo-128 SPF next-hop differs from Algo 0 where the min-delay path diverges from the min-cost path — proving a **separate logical topology on the same physical network** with no tunnels.

---

# Section 5 — LDP → SR Migration (4 tasks) — Emerald AS 65100

## Task 5.1 — Enable SR alongside LDP (dual-stack)

**Question:** On Emerald (currently IS-IS + LDP), enable **SR-MPLS under IS-IS** so nodes run **LDP and SR simultaneously** (ships-in-the-night), with no traffic disruption.

**Solution**

SR and LDP coexist: LDP keeps distributing its local labels while IS-IS begins advertising prefix-SIDs. By default, when both a prefix-SID and an LDP label exist for a FEC, XR uses **LDP** as the outgoing label (SR is the backup) until you flip preference. Enable SR without removing LDP.

```
! ==== All Emerald nodes (example E-R3 = 3.3.3.3) ====
router isis 1
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls          ! add SR; LDP config left in place
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 3           ! last octet of 3.3.3.3
!

! (LDP remains configured)
mpls ldp
 router-id 3.3.3.3
 interface GigabitEthernet0/0/0/1
!
```

Assign prefix-SIDs matching last octet: E-R1=1, E-R2=2, E-R3=3, E-R4=4, E-R6=5, E-R5=6.

**Verification**

```
show mpls ldp neighbor                 ! LDP still up
show isis segment-routing label table  ! prefix-SIDs now advertised
show mpls forwarding                    ! both label types present
```

---

## Task 5.2 — Verify both label types in the LFIB

**Question:** Show that the **LFIB contains both LDP and SR labels** during the dual-stack phase.

**Solution**

No new config — validation that both control planes populate forwarding, with LDP currently preferred as the outgoing label.

**Verification**

```
show mpls forwarding
show cef 1.1.1.1/32 detail            ! shows LDP outgoing label (SR as backup)
show mpls ldp bindings 1.1.1.1/32
show isis segment-routing label table
```

In `show cef ... detail`, note `labels imposed {LDPlabel}` with the SR prefix-SID listed as an alternate.

---

## Task 5.3 — Enable SR-prefer (prefer SR labels over LDP)

**Question:** Flip preference so **SR labels are preferred over LDP** on all Emerald nodes.

**Solution**

`segment-routing prefer` (a.k.a. `sr-prefer`) under the IS-IS address-family makes SR the chosen outgoing label whenever a prefix-SID exists, while LDP stays up as fallback. This is the key migration step — do it network-wide before removing LDP so forwarding stays consistent.

```
! ==== All Emerald nodes ====
router isis 1
 address-family ipv4 unicast
  segment-routing mpls sr-prefer      ! prefer SR over LDP
 !
!
```

**Verification**

```
show cef 1.1.1.1/32 detail            ! outgoing label now the SR prefix-SID (16001)
show mpls forwarding prefix 1.1.1.1/32
show isis segment-routing label table
```

Outgoing label for `1.1.1.1/32` should now be `16001` (SR), not the LDP local label.

---

## Task 5.4 — Remove LDP completely & verify no traffic loss

**Question:** Remove LDP from all Emerald nodes and confirm **no traffic loss** during and after the migration.

**Solution**

Because forwarding already uses SR labels (Task 5.3), removing LDP is non-disruptive — the SR data plane is already carrying traffic. Start continuous traffic, remove LDP, confirm zero loss, then verify LDP is fully gone.

```
! ==== Before removal — continuous traffic E-R1 -> E-R2 ====
ping 2.2.2.2 source Loopback0 count 100000

! ==== Remove LDP (all Emerald nodes) ====
no mpls ldp
! also remove any 'mpls ldp sync' / interface ldp references under IS-IS

! (optional) remove sr-prefer since LDP is gone:
router isis 1
 address-family ipv4 unicast
  segment-routing mpls              ! sr-prefer no longer needed
!
```

**Verification**

```
! Running ping shows 0 loss during 'no mpls ldp'
show mpls ldp neighbor                 ! empty — LDP gone
show mpls forwarding                    ! SR labels only
show cef 2.2.2.2/32 detail             ! SR label, no LDP alternate
traceroute 2.2.2.2 source Loopback0
```

**Migration summary:** enable SR (dual-stack) → verify both labels → `sr-prefer` → remove LDP. Traffic uses SR before LDP is ever touched, so the cutover is hitless.

---

# Section 6 — Troubleshooting (3 tasks)

## Task 6.1 — Prefix-SID conflict (two routers, same index)

**Question:** Two Garnet routers advertise the **same prefix-SID index** (e.g. Gar-R4 and Gar-R5 both use index 14). Diagnose and fix.

**Solution**

A prefix-SID index must be unique network-wide. When two nodes advertise the same index, IS-IS SR detects a **conflict**; XR flags it and (per the SR conflict-resolution rules) may ignore one or both bindings, breaking reachability to the affected loopback.

**Diagnosis**

```
show isis segment-routing prefix-sid-map active-policy
show isis segment-routing label table
show logging | include SR|CONFLICT|prefix-sid
show isis database verbose | include SID       ! see who advertises which index
```

Look for a `Conflict`/`Prefix-SID conflict` log and two loopbacks bound to the same index.

**Fix**

```
! ==== Gar-R5 — correct its index back to 15 ====
router isis 1
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 15        ! was 14 (conflict) -> restore last-octet value
!
```

**Verify fix**

```
show isis segment-routing label table   ! 16014=Gar-R4, 16015=Gar-R5, unique again
show cef 15.15.15.15/32 detail
```

---

## Task 6.2 — SRGB mismatch

**Question:** One Garnet node has a **different SRGB** (e.g. Gar-R3 = 18000–25999 while everyone else is 16000–23999). Diagnose the impact and fix.

**Solution**

Because prefix-SIDs are advertised as *indexes*, each node computes `label = its_own_SRGB_base + index`. A mismatched base means the offending node imposes the *wrong* label toward SR destinations (and neighbors program the wrong incoming label for it), breaking SR forwarding through/at that node. Symptom: pings fail or take a non-SR path even though IS-IS adjacencies are up.

**Diagnosis**

```
show mpls label range                    ! compare SRGB on each node
show segment-routing local-block inout
show isis segment-routing label table    ! Gar-R3 derives different labels
show mpls forwarding labels 16013        ! label programming inconsistent
traceroute 13.13.13.13                   ! fails / wrong labels
```

The tell-tale: `show mpls label range` shows `SRGB 18000-25999` on Gar-R3 vs `16000-23999` elsewhere.

**Fix**

```
! ==== Gar-R3 — restore the common SRGB ====
segment-routing
 global-block 16000 23999
!
```

> SRGB changes are disruptive; expect a brief SR label reprogramming. In a real migration, coordinate a maintenance window.

**Verify fix**

```
show mpls label range                    ! 16000-23999 everywhere
show isis segment-routing label table    ! 16013 = Gar-R3 consistently
traceroute 13.13.13.13 source Loopback0
```

---

## Task 6.3 — SR-TE policy down (SID not reachable)

**Question:** An explicit SR-TE policy on Gar-R1 is **Operational: down**. The segment-list references a prefix-SID label that is not reachable/valid. Diagnose and fix.

**Solution**

An explicit policy goes down when the head-end cannot resolve the **first SID** (or a SID in the list) — e.g. the segment-list uses `mpls label 16099` for a node that does not exist, uses an adjacency-SID that no longer applies, or references a prefix-SID whose owner is unreachable in the IGP. XR marks the candidate path invalid → policy down.

**Diagnosis**

```
show segment-routing traffic-eng policy               ! Operational: down
show segment-routing traffic-eng policy name <name> detail
! -> look for 'invalid'/'no valid path'/'first SID unreachable'
show mpls forwarding labels 16099                     ! is the SID installed?
show cef 15.15.15.15/32                                ! is the endpoint/transit reachable?
show isis segment-routing label table
```

Typical cause shown: segment-list points at `16099` (no such prefix-SID) or a transit node's loopback is down.

**Fix**

```
! ==== Gar-R1 — correct the segment-list to valid, reachable prefix-SIDs ====
segment-routing
 traffic-eng
  segment-list SL-VIA-P4-P5
   index 10 mpls label 16014     ! Gar-R4  (was 16099 - invalid)
   index 20 mpls label 16015     ! Gar-R5
   index 30 mpls label 16012     ! Gar-R2 endpoint
  !
 !
!
```

**Verify fix**

```
show segment-routing traffic-eng policy               ! Admin up / Operational up
show segment-routing traffic-eng policy name <name> detail
traceroute 12.12.12.12 source Loopback0                ! follows the segment-list
```

---

## Quick Command Reference (IOS-XR)

| Purpose | Command |
|---------|---------|
| SR label table | `show isis segment-routing label table` |
| SRGB / label range | `show mpls label range` |
| SRLB (adj-SIDs) | `show segment-routing local-block inout` |
| Adjacency-SIDs | `show isis segment-routing adjacency-sid` |
| TI-LFA summary | `show isis fast-reroute summary` |
| TI-LFA per prefix | `show isis fast-reroute <prefix> detail` |
| FIB primary+backup | `show cef <prefix> detail` |
| MPLS forwarding | `show mpls forwarding` |
| SR-TE policies | `show segment-routing traffic-eng policy` |
| PCC → Gar-R6 session | `show segment-routing traffic-eng pcc ipv4 peer` |
| Gar-R6 topology (BGP-LS) | `show pce ipv4 topology summary` |
| Gar-R6 LSPs / peers | `show pce lsp` / `show pce ipv4 peer` |
| Flex-Algo | `show isis flex-algo` / `show isis flex-algo 128 detail` |
| Prefix-SID conflicts | `show isis segment-routing prefix-sid-map active-policy` |
| LDP neighbors (migration) | `show mpls ldp neighbor` |
| VPN color check | `show bgp vpnv4 unicast <prefix>` |

---

## Key Concepts Recap

- **SRGB** = global reserved label range; identical everywhere → prefix-SIDs are **globally significant** (LDP labels are **locally significant**).
- **Prefix-SID** (global, shortest-path via SRGB) vs **Adjacency-SID** (local, specific link, via SRLB; protected/unprotected).
- **TI-LFA** = pre-computed, topology-independent post-convergence backup segment list in the FIB → sub-50ms, link *or* node protection.
- **SR-TE** = path is a label stack on the head-end; **no transit state**. Steer by **color + endpoint**. Explicit (operator list) vs Dynamic (constrained SPF, metric igp/te/latency).
- **SR-PCE** = central PCEP computation with **BGP-LS** topology; `pce address` + `pce segment-routing`.
- **ODN** = auto-create SR-TE policy on matching BGP **color** to the next-hop; scales L3VPN steering.
- **Flex-Algo** = multiple constrained logical topologies (e.g. delay) with **per-algorithm prefix-SIDs**; no tunnels.
- **LDP→SR migration** = dual-stack → verify both labels → **sr-prefer** → remove LDP = **hitless**.
