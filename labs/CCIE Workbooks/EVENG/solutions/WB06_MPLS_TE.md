# CCIE SP Workbook 06 — MPLS Traffic Engineering (RSVP-TE)

**Exam Domain:** Domain 1 — Core Routing (25%)
🔴 **CCIE Prep Platform:** EVE-NG (IOS-XRv 9000 7.11.1) — see `00_EVENG_Topology.md`
**Focus:** Emerald **AS 65100** — IS-IS L2 + LDP + **RSVP-TE**
**Prerequisites:** IS-IS L2 backbone with **wide metrics** and /32 loopbacks (Workbook 01), LDP for baseline label transport (Workbook 03).

> **Note:** This workbook is classic **RSVP-TE** — stateful, per-hop signaled LSPs with soft-state Path/Resv refresh. It contrasts with Workbook 09 (SR-TE), which is stateless and needs no RSVP. RSVP-TE remains heavily weighted in CCIE-SP Core Routing because it teaches CSPF, bandwidth accounting, FRR, and DS-TE — the conceptual foundation SR-TE builds on. All syntax is **IOS-XR**.

---

## Topology (Emerald AS 65100)

```
                       E-R5 (6.6.6.6)
                         │ 10.1.1.0/24
                         │
   E-R6 (5.5.5.5) ──────E-R4 (4.4.4.4)
        10.1.2.0/24      │ 10.1.3.0/24
                         │
                       E-R3 (3.3.3.3)
              10.1.4.0/24 │ │ 10.1.5.0/24
                 ┌────────┘ └────────┐
              E-R1 (1.1.1.1)        E-R2 (2.2.2.2)
                 └──────────────────┘
                     10.1.6.0/24
```

| Link | Subnet | A-end / Z-end |
|------|--------|---------------|
| E-R5 ↔ E-R4   | 10.1.1.0/24 | E-R5=.6 / E-R4=.4 |
| E-R4 ↔ E-R6  | 10.1.2.0/24 | E-R4=.4 / E-R6=.5 |
| E-R4 ↔ E-R3     | 10.1.3.0/24 | E-R4=.4 / E-R3=.3 |
| E-R3 ↔ E-R1    | 10.1.4.0/24 | E-R3=.3 / E-R1=.1 |
| E-R3 ↔ E-R2    | 10.1.5.0/24 | E-R3=.3 / E-R2=.2 |
| E-R1 ↔ E-R2   | 10.1.6.0/24 | E-R1=.1 / E-R2=.2 |

**Path note:** The IGP shortest path E-R1→E-R2 is the direct link `10.1.6.0/24` (1 hop). TE tunnels in this workbook are engineered *away* from that path — via **E-R1→E-R3→E-R2** — so you can observe TE overriding IGP.

---

## Section 1 — RSVP-TE Basics

### Task 1.1 — Enable MPLS-TE and RSVP on all Emerald core interfaces

Enable the RSVP-TE control plane globally and per-interface, extend IS-IS to flood TE link attributes, and bring RSVP up on every core link.

**Configuration**

```
! ---- On every Emerald core node (E-R1, E-R2, E-R3, E-R4, E-R6, E-R5) ----

rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth 1000000        ! total reservable = 1 Gbps (kbps)
 !
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/0
 !
!
! Extend IS-IS to advertise TE (must be level-2, wide metrics already set)
router isis EMERALD
 address-family ipv4 unicast
  metric-style wide
  mpls traffic-eng level-2-only
  mpls traffic-eng router-id Loopback0
 !
!
```

RSVP-TE has two planes. The **IGP** (IS-IS with the TE extensions) floods each link's TE attributes — reservable bandwidth, TE metric, admin-groups, SRLGs — into the **TE topology database (TED)**. **RSVP** is the signaling protocol that walks the computed path hop-by-hop with Path/Resv messages, installing label bindings and decrementing reservable bandwidth as it goes. `mpls traffic-eng router-id Loopback0` gives a stable TE ID; `mpls traffic-eng level-2-only` matches the L2 backbone. Without the IGP TE extensions the head-end has no TED and CSPF cannot run.

**Verification**
- `show rsvp interface` — RSVP enabled, reservable BW shown per interface.
- `show mpls traffic-eng link-management interfaces` — admin/oper UP, bandwidth pools.
- `show isis mpls traffic-eng tunnel` / `show mpls traffic-eng topology` — TED populated with all links + BW.

---

### Task 1.2 — Configure a TE tunnel E-R1→E-R2 via explicit path (E-R1→E-R3→E-R2)

On E-R1 build `tunnel-te1` to E-R2 (2.2.2.2) forced onto the **non-shortest** path E-R1→E-R3→E-R2 using an explicit-path.

**Configuration**

```
! ---- E-R1 ----
explicit-path name PE1_via_P1_to_PE2
 index 10 next-address strict ipv4 unicast 10.1.4.3    ! E-R1 -> E-R3
 index 20 next-address strict ipv4 unicast 10.1.5.2    ! E-R3  -> E-R2
!
interface tunnel-te1
 ipv4 unnumbered Loopback0
 destination 2.2.2.2
 path-option 10 explicit name PE1_via_P1_to_PE2
!
```

An RSVP-TE tunnel is **unidirectional** and head-end signaled. `ipv4 unnumbered Loopback0` borrows the loopback address (tunnel interfaces don't need their own subnet). The **explicit-path** with `strict` hops forces the LSP through E-R3 rather than the direct E-R1↔E-R2 link — the whole point of TE. E-R1 sends an RSVP **Path** message hop-by-hop to E-R2; E-R2 replies with **Resv**, distributing labels upstream (downstream-on-demand). The result is an LSP whose forwarding is decoupled from the IGP shortest path.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1` — state **up**, signalled path lists E-R3 then E-R2.
- `show mpls traffic-eng tunnels brief` — `tunnel-te1` UP/UP.
- `traceroute` sourced through the tunnel: hops E-R1→E-R3→E-R2, **not** the direct link.

---

### Task 1.3 — Autoroute announce

Make E-R1's IGP/CEF use `tunnel-te1` to reach E-R2 and prefixes behind it, without static routes.

**Configuration**

```
! ---- E-R1 ----
interface tunnel-te1
 autoroute announce
!
```

`autoroute announce` inserts the tunnel into the head-end's SPF as a **logical link to the tail-end** — the head-end's IGP computes routes to the tail (and destinations behind it) *as if* the tunnel were a direct adjacency. This is how traffic actually enters the tunnel: no policy-based routing or static needed. The tunnel does **not** get flooded to other routers (it's local to E-R1's RIB/CEF). Compare with `forwarding-adjacency` (Task 5.3), which *does* advertise the tunnel into the IGP as a real link.

**Verification**
- `show route 2.2.2.2` — next-hop is `tunnel-te1`.
- `show cef 2.2.2.2` — outgoing interface `tunnel-te1`.
- `show mpls traffic-eng autoroute` — announced destinations via the tunnel.

---

### Task 1.4 — Verify tunnel UP and traffic forwarded

Confirm the LSP is UP end-to-end and data-plane traffic is actually label-switched over E-R1→E-R3→E-R2.

**Configuration**

```
! Generate traffic E-R1 -> E-R2 loopback (via tunnel due to autoroute)
! ---- E-R1 ----
ping 2.2.2.2 source 1.1.1.1
traceroute 2.2.2.2 source 1.1.1.1
```

Tunnel "up" in the control plane (RSVP signaled) is not proof of forwarding — you must confirm the FIB points at the tunnel *and* packets traverse it. `show mpls forwarding` on E-R3 shows the mid-point label swap; the traceroute shows E-R3 as the transit hop, proving TE is overriding the 1-hop IGP path.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 detail` — Admin: up / Oper: up, RSVP Resv received.
- On E-R1: `show mpls forwarding tunnels` — labels imposed for tunnel-te1.
- On E-R3: `show mpls forwarding` — label swap entry for the tunnel LSP (transit).
- `traceroute 2.2.2.2 source 1.1.1.1` — path E-R1→E-R3→E-R2.

---

### Task 1.5 — TE metric vs IGP metric

Configure the tunnel to compute its dynamic path using the **TE metric** instead of the IGP metric, and set per-link TE metrics so the two produce different paths.

**Configuration**

```
! ---- Set an independent TE metric on selected core links (all nodes owning the link) ----
mpls traffic-eng
 interface GigabitEthernet0/0/0/0
  admin-weight 5           ! TE metric, independent of IGP cost
 !
!
! ---- E-R1: add a dynamic path-option that optimizes on TE metric ----
interface tunnel-te1
 path-selection metric te
 path-option 20 dynamic
!
```

Every TE link carries **two** costs: the IGP metric (used by normal SPF and by TE when `metric igp`) and the **TE metric / admin-weight** (used only by CSPF when the tunnel is set to `metric te`). Operators use this to build a *separate* routing plane for TE — e.g., IGP metrics reflect cost/OSPF-style bandwidth, while TE metrics reflect latency. Setting `path-selection metric te` makes CSPF minimize the TE metric; with `metric igp` it minimizes IGP cost. Divergent metrics let one physical topology yield different shortest paths for best-effort vs TE traffic.

**Verification**
- `show mpls traffic-eng topology` — each link shows both IGP metric and TE metric (admin-weight).
- `show mpls traffic-eng tunnels tunnel-te1` — "Metric Type: TE" and the CSPF-chosen path.
- Change a link's `admin-weight` and confirm the dynamic path-option recomputes to a different path than IGP would pick.

---

## Section 2 — FRR Protection

### Task 2.1 — Link protection (facility backup bypass tunnel)

Protect the E-R1→E-R3 link (primary tunnel's first hop) with a **NHOP bypass** tunnel on E-R1 that reroutes around the protected link to E-R3.

**Configuration**

```
! ---- E-R1: bypass around the E-R1->E-R3 link, reaching E-R3 via E-R2->... ----
explicit-path name BYPASS_NHOP_to_P1
 index 10 next-address strict ipv4 unicast 10.1.6.2    ! E-R1 -> E-R2
 index 20 next-address strict ipv4 unicast 10.1.5.3    ! E-R2 -> E-R3
!
interface tunnel-te10
 ipv4 unnumbered Loopback0
 destination 3.3.3.3                                   ! E-R3 = Next-Hop (NHOP)
 path-option 10 explicit name BYPASS_NHOP_to_P1
 backup-bwlimit ...        ! optional bandwidth to protect
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/0                       ! the E-R1->E-R3 physical interface
  backup-path tunnel-te10
 !
!
```

**Facility backup** (RFC 4090) protects the *facility* (a link or node) with one pre-signaled **bypass LSP** that can carry *many* protected LSPs — far more scalable than one-to-one/detour backup. The **PLR** (Point of Local Repair = E-R1) pre-signals the bypass around the protected link to the **MP** (Merge Point). On failure the PLR **pushes an extra label** (the bypass LSP's label) so protected traffic tunnels around the break and re-merges at the MP — all in hardware, before the head-end reroutes. A **NHOP** bypass terminates at the *next hop* (E-R3), protecting against **link** failure.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te10` — bypass UP.
- `show mpls traffic-eng fast-reroute database` — protected LSPs bound to tunnel-te10, state Ready.
- `show mpls traffic-eng link-management interfaces` — interface shows a backup tunnel assigned.

---

### Task 2.2 — Node protection (NNHOP bypass)

Protect against **E-R3 node** failure with a bypass on E-R1 that skips E-R3 entirely and terminates at the **next-next-hop** E-R2.

**Configuration**

```
! ---- E-R1: bypass to NNHOP (E-R2), avoiding node E-R3 ----
explicit-path name BYPASS_NNHOP_to_PE2
 index 10 next-address strict ipv4 unicast 10.1.6.2    ! E-R1 -> E-R2 directly
!
interface tunnel-te11
 ipv4 unnumbered Loopback0
 destination 2.2.2.2                                    ! E-R2 = Next-Next-Hop (NNHOP)
 path-option 10 explicit name BYPASS_NNHOP_to_PE2
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/0                        ! E-R1->E-R3 interface
  backup-path tunnel-te11
 !
!
```

An **NNHOP** bypass terminates at the *next-next-hop* (E-R2), so it protects against failure of the **entire E-R3 node**, not just the E-R1→E-R3 link. The key subtlety: the MP is now E-R2, so the PLR must impose the label that E-R3 *would have* given to E-R2 (the "backup label" learned from RSVP RRO/label recording) beneath the bypass label — otherwise E-R2 would receive an unexpected label. Node protection is strictly stronger than link protection; a node-protecting bypass also covers the link.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te11 detail` — NNHOP, tail 2.2.2.2.
- `show mpls traffic-eng fast-reroute database detail` — protection type **node**, backup label recorded.
- `show mpls traffic-eng tunnels tunnel-te1 detail` — FRR "Node/link protection: desired/available".

---

### Task 2.3 — Configure fast-reroute on the primary tunnel

Arm the primary `tunnel-te1` to actually *use* the bypass LSPs, requesting node protection.

**Configuration**

```
! ---- E-R1 ----
interface tunnel-te1
 fast-reroute
 fast-reroute protection node-protection      ! request node (implies link) protection
!
```

`fast-reroute` on the *protected* tunnel sets the "Local protection desired" flag in the RSVP **Path** SESSION-ATTRIBUTE object, telling every downstream PLR to bind this LSP to a suitable bypass. Without it, bypass tunnels exist but no primary LSP requests protection — nothing gets rerouted on failure. `node-protection` asks PLRs to prefer an NNHOP bypass. Protection is negotiated hop-by-hop; the head-end sees the aggregate protection status in the RRO.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "FRR: enabled", "Protection: node desired".
- `show mpls traffic-eng fast-reroute database` — tunnel-te1 LSP = **Ready**, bound to the correct bypass.
- RRO shows "Local-Protection-Available" flags on the protected hops.

---

### Task 2.4 — Test with link failure (sub-50ms switchover)

Prove FRR delivers sub-50ms protection: run continuous traffic through `tunnel-te1`, fail the E-R1→E-R3 link, count loss.

**Configuration**

```
! ---- Continuous traffic (behind E-R1 toward a E-R2 dest), then fail the link ----
! ---- E-R3 (or E-R1): shut the protected interface ----
interface GigabitEthernet0/0/0/0
 shutdown
```

FRR's sub-50ms guarantee comes from **local repair pre-computed and pre-installed** in the FIB: on link-down (detected by loss of signal or **BFD**, not IGP timers), the PLR flips to the bypass label stack in hardware — no control-plane recomputation in the fast path. The head-end later performs a **make-before-break** reoptimization onto a fresh optimal path (global repair), then tears down the temporary FRR path. Enable **BFD** on the link for detection well under the physical-failure timers to hit the <50ms target reliably.

**Verification**
- Traffic monitor: **0–1 packets lost** at failover (vs seconds without FRR).
- `show mpls traffic-eng fast-reroute database` — state transitions Ready → **Active** on failure.
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "FRR active", then reoptimized after global repair.

---

### Task 2.5 — Protected vs unprotected adjacency-SID comparison (concept)

Contrast RSVP-TE FRR (this workbook) with SR TI-LFA using **protected vs unprotected adjacency-SIDs** (Workbook 09).

**Configuration**

```
! Conceptual — SR reference (see Workbook 09 for full config):
! router isis EMERALD
!  interface Gi0/0/0/0
!   address-family ipv4 unicast
!    adjacency-sid ...            ! protected adj-SID = TI-LFA backup pre-computed
```

In **RSVP-TE**, protection state lives *in the network* — every PLR holds pre-signaled bypass LSPs with soft-state refresh (scaling cost grows with LSP count). In **SR**, an **adjacency-SID** can be **protected** (default) — the IGP pre-computes a **TI-LFA** backup and the FIB carries a repair segment-list for that link — or **unprotected**, where a failure of that link simply drops the LSP (used when the head-end wants to detect failure and re-steer itself). SR achieves the same <50ms local repair with **zero per-LSP transit state**: the repair is a label stack, not a signaled tunnel. This is the fundamental scalability argument for SR-TE over RSVP-TE.

**Verification**
- RSVP-TE: `show mpls traffic-eng fast-reroute database` — per-LSP bypass bindings (state in network).
- SR (WB09): `show isis segment-routing adjacency-sid` — Protected flag; `show isis fast-reroute` — TI-LFA repair list.
- Compare: RSVP FRR state count scales with #LSPs; TI-LFA repair state scales with #links only.

---

## Section 3 — Bandwidth Management

### Task 3.1 — RSVP bandwidth reservation on links

Reserve bandwidth for `tunnel-te1` and confirm RSVP decrements each link's reservable pool along the path.

**Configuration**

```
! ---- Ensure reservable BW is set on each core interface (Task 1.1) ----
rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth 1000000        ! 1 Gbps reservable (kbps)
 !
!
! ---- E-R1: request bandwidth on the tunnel ----
interface tunnel-te1
 signalled-bandwidth 200000    ! reserve 200 Mbps (kbps)
!
```

RSVP bandwidth is an **accounting/admission-control** construct, not a policer — reserving 200 Mbps does *not* rate-limit traffic; it decrements the link's advertised reservable pool so CSPF won't over-subscribe. The IGP re-floods updated reservable BW so all head-ends' TEDs stay accurate. If a link can't satisfy the request, CSPF prunes it and finds another path (or the tunnel fails — see Section 6). Actual rate enforcement requires QoS (Workbook 12). This separation of *control-plane reservation* from *data-plane policing* is a classic exam point.

**Verification**
- `show rsvp interface detail` — reserved vs available BW per interface.
- `show mpls traffic-eng link-management bandwidth-allocations` — allocations per priority pool.
- `show mpls traffic-eng tunnels tunnel-te1` — "Bandwidth: 200000 kbps" reserved along signalled path.

---

### Task 3.2 — Setup/hold priority (0–7)

Assign `tunnel-te1` a setup and hold priority so it can be positioned in the preemption hierarchy.

**Configuration**

```
! ---- E-R1 ----
interface tunnel-te1
 priority 3 3            ! setup-priority 3, hold-priority 3 (0=best,7=worst)
!
```

Every LSP has a **setup priority** (how aggressively it can preempt others to get established) and a **hold priority** (how strongly it resists being preempted). Range 0–7 where **0 = highest**. Rule enforced by IOS-XR: setup priority must be **numerically ≥ hold priority** (setup can't be "better" than hold) to prevent an LSP from preempting itself on reoptimization. These priorities feed the **8 bandwidth pools** RSVP tracks per link — a higher-priority LSP can claim bandwidth held by a lower-priority one.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "Config Parameters: … priority: 3 3".
- `show mpls traffic-eng link-management bandwidth-allocations` — per-priority reserved amounts.

---

### Task 3.3 — Preemption scenario (high-priority tunnel preempts low-priority)

Create a second tunnel `tunnel-te2` (E-R1→E-R2, same E-R1→E-R3→E-R2 path) with a **better** priority and enough bandwidth to force preemption of `tunnel-te1` when the link is congested.

**Configuration**

```
! ---- E-R1: low-priority incumbent already up = tunnel-te1 (priority 3 3, 200 Mbps) ----

! ---- E-R1: high-priority challenger needing bandwidth that only fits by preempting ----
interface tunnel-te2
 ipv4 unnumbered Loopback0
 destination 2.2.2.2
 path-option 10 explicit name PE1_via_P1_to_PE2
 signalled-bandwidth 900000     ! 900 Mbps
 priority 1 1                    ! better than tunnel-te1's 3 3
 autoroute announce
!
```

If the shared link has 1 Gbps reservable and `tunnel-te1` already holds 200 Mbps at hold-priority 3, a 900 Mbps request at setup-priority 1 cannot fit (200+900 > 1000). Because the challenger's **setup priority (1)** is better than the incumbent's **hold priority (3)**, RSVP **preempts** tunnel-te1: it tears down (or with soft-preemption, marks) the lower LSP, frees the bandwidth, and establishes tunnel-te2. The preempted LSP then tries to re-signal on any other feasible path. **Soft-preemption** (`soft-preemption` under mpls-te) lets the incumbent gracefully reroute before teardown, avoiding a hard drop.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te2` — UP with 900 Mbps on the path.
- `show mpls traffic-eng tunnels tunnel-te1` — down/re-routing due to **preemption** (reason logged).
- `show rsvp interface detail` — reservable pool reflects the challenger's reservation.
- `show mpls traffic-eng preemption log` — preemption event recorded.

---

### Task 3.4 — Auto-bandwidth (measure and adjust)

Enable **auto-bandwidth** on `tunnel-te1` so it samples actual traffic and periodically resizes its reservation.

**Configuration**

```
! ---- Global collection interval ----
mpls traffic-eng
 auto-bw collect frequency 5           ! sample every 5 minutes
!
! ---- E-R1: enable on the tunnel ----
interface tunnel-te1
 auto-bw
  bw-limit min 100000 max 800000       ! clamp adjustments (kbps)
  adjustment-threshold 10              ! % change before resizing
  application 30                       ! apply new BW every 30 minutes
 !
!
```

Auto-bandwidth removes manual capacity guessing: the head-end **measures** the tunnel's actual throughput over a collection interval, tracks the peak, and at the application interval **re-signals** the tunnel with the new bandwidth using **make-before-break** (Section 4) so there's no traffic loss. `bw-limit` bounds it (safety), `adjustment-threshold` avoids churn for tiny changes. Combined with CSPF, this lets tunnels grow onto paths that can hold the new size, or shrink to free capacity — the foundation of automated capacity management.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 auto-bw` — current sampled rate, highest BW, next application time.
- `show mpls traffic-eng tunnels tunnel-te1 detail` — signalled BW changes after an application interval.
- Correlate the resize with a make-before-break reoptimization (new LSP-ID before old torn down).

---

## Section 4 — Advanced TE

### Task 4.1 — Make-before-break (SE style)

Reoptimize `tunnel-te1` onto a better path without dropping traffic, using **Shared-Explicit** reservation style.

**Configuration**

```
! ---- E-R1: enable periodic reoptimization ----
mpls traffic-eng
 reoptimize timers frequency 3600      ! reoptimize hourly
!
! Add/adjust a path-option; reoptimization uses MBB automatically.
interface tunnel-te1
 path-option 5 dynamic                 ! better metric option; MBB moves to it
!
! Manual trigger:
! mpls traffic-eng reoptimize tunnel-te1
```

**Make-before-break (MBB)** signals the *new* LSP fully (new LSP-ID, same tunnel/SESSION) and only tears the old one down **after** the new path is up — zero loss. It relies on RSVP's **Shared-Explicit (SE)** reservation style: because both the old and new LSPs belong to the same session, they **share** the bandwidth reservation on any links they have in common instead of double-counting it. Without SE (i.e., Fixed-Filter style) the two LSPs would each demand full bandwidth on shared links and reoptimization could fail on a link that actually has room. MBB is what makes reoptimization, auto-bandwidth, and graceful preemption non-disruptive.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 detail` — reservation **Style: SE (Shared Explicit)**.
- During reoptimization: two LSP-IDs briefly present (old + new) for the same tunnel.
- Traffic monitor across a manual `reoptimize`: **0 packet loss**.

---

### Task 4.2 — DS-TE with MAM / RDM

Enable **DiffServ-Aware TE** so a sub-pool (e.g., voice/EF) has its own bandwidth accounting, and compare the **MAM** and **RDM** bandwidth-constraint models.

**Configuration**

```
! ---- All core nodes: define DS-TE bandwidth model on RSVP interfaces ----
rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth mam max-reservable-bw 1000000 bc0 1000000 bc1 300000
  ! -- or --
  ! bandwidth rdm bc0 1000000 bc1 300000
 !
!
! ---- E-R1: tunnel reserving from the sub-pool (class-type 1) ----
interface tunnel-te1
 signalled-bandwidth 200000 class-type 1
!
```

Plain TE tracks one aggregate bandwidth pool; **DS-TE** splits it into **class-types** (BC0 = global, BC1 = sub-pool for priority traffic like EF/voice) so you can guarantee capacity for a class end-to-end and admission-control it separately. The two models differ in *how* the constraints relate:
- **MAM (Maximum Allocation Model):** each class gets a **hard, non-overlapping** slice — simple, isolated, but can strand unused capacity (BC1 idle can't help BC0).
- **RDM (Russian Dolls Model):** constraints **nest** (BC1 ⊆ BC0), so higher classes can borrow from the shared aggregate — more efficient link utilization, but requires **preemption** to enforce class guarantees under contention.

The whole DS-TE model (MAM vs RDM) must be **consistent on every link** or reservations won't add up.

**Verification**
- `show rsvp interface detail` — model (MAM/RDM), BC0/BC1 values.
- `show mpls traffic-eng link-management bandwidth-allocations` — per class-type allocations.
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "Class-Type: 1", sub-pool reservation.

---

### Task 4.3 — Affinity / admin-group (color bits)

Color links with **admin-groups** and constrain `tunnel-te1` to include/exclude specific colors via **affinity**.

**Configuration**

```
! ---- Color links (attribute-flags / admin-group bit) on each interface ----
mpls traffic-eng
 interface GigabitEthernet0/0/0/0
  attribute-names RED           ! named admin-group (XR named-affinity) 
 !
 affinity-map RED bit-position 0
!
! ---- E-R1: tunnel affinity constraint ----
interface tunnel-te1
 affinity exclude RED           ! do not traverse RED links
 ! or:  affinity include-strict BLUE
!
```

**Admin-groups** are 32 (or 256 with extended/named affinities) **color bits** flooded per link by the IGP. A tunnel's **affinity** is a constraint CSPF applies during path computation: `include` (must have color), `include-strict` (must have *exactly* these), or `exclude` (must not have color). This lets you enforce policy without touching topology — e.g., color trans-oceanic links RED and have latency-sensitive tunnels `exclude RED`, or reserve "gold" links for gold tunnels. CSPF prunes non-matching links **before** running shortest-path, so affinity shapes the candidate topology, not just the final path.

**Verification**
- `show mpls traffic-eng topology` — each link's attribute-flags/admin-group visible.
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "Affinity: exclude RED" and resulting path avoids RED links.
- Color the current path's link RED and confirm CSPF reroutes around it.

---

### Task 4.4 — CSPF constrained path computation

Add a fully **dynamic** path-option and observe **CSPF** honoring all constraints (bandwidth, affinity, TE metric) to compute the path — with no explicit hops.

**Configuration**

```
! ---- E-R1 ----
interface tunnel-te1
 path-selection metric te
 signalled-bandwidth 200000
 affinity exclude RED
 path-option 100 dynamic          ! CSPF computes, subject to all constraints above
!
```

**CSPF (Constrained SPF)** is standard Dijkstra run on a **pruned** TED: before computing shortest path, CSPF removes every link that fails a constraint — insufficient reservable bandwidth at the tunnel's priority, wrong/forbidden affinity, excluded SRLGs — then finds the lowest **TE-metric** (or IGP-metric) path through what remains. This is entirely a **head-end** computation using the IGP-flooded TED; RSVP then signals the result. If pruning leaves no path, CSPF fails and the tunnel goes down with "no path" (Section 6). CSPF is the brain of RSVP-TE — bandwidth, affinity, priority, and metric all converge here.

**Verification**
- `show mpls traffic-eng tunnels tunnel-te1 detail` — "Path-option 100 dynamic (CSPF)", computed hop list.
- `show mpls traffic-eng topology path destination 2.2.2.2 ...` — run CSPF interactively and see pruned links.
- Tighten a constraint (raise BW beyond a link's capacity) and watch CSPF choose a different path or fail.

---

## Section 5 — TE + VPN Integration

### Task 5.1 — Autoroute announce with VPN (L3VPN traffic over TE tunnel)

Carry **VPNv4 (L3VPN)** customer traffic from E-R1 to E-R2 over `tunnel-te1` using autoroute announce, so the VPN label rides inside the TE LSP.

**Configuration**

```
! ---- Assumes L3VPN from Workbook 04: VRF CUST_A, VPNv4 iBGP E-R1<->E-R2 ----
! ---- E-R1: tunnel already has autoroute announce (Task 1.3) ----
interface tunnel-te1
 autoroute announce
!
! Nothing VPN-specific is needed on the tunnel — BGP next-hop resolution does the rest.
```

L3VPN forwarding is a **two-label** stack: the **inner** VPN label (allocated by the egress E-R2, identifies the VRF/prefix) and the **outer** transport label (gets the packet to E-R2). Normally the transport label comes from LDP; with `autoroute announce`, E-R1's route to the **BGP next-hop (2.2.2.2 = E-R2)** resolves via `tunnel-te1`, so the transport label becomes the **TE LSP** instead of an LDP LSP. The VPN label is untouched — VPN and TE are orthogonal layers. Result: all VPNv4 traffic whose next-hop is E-R2 is automatically engineered over the TE tunnel, no per-VRF config.

**Verification**
- `show bgp vpnv4 unicast vrf CUST_A <prefix>` — next-hop 2.2.2.2.
- `show route vrf CUST_A <prefix>` → recurses to 2.2.2.2 → `tunnel-te1`.
- `show cef vrf CUST_A <prefix> detail` — imposes {VPN label, tunnel-te1 transport label}.
- `traceroute vrf CUST_A ...` — path E-R1→E-R3→E-R2.

---

### Task 5.2 — Per-VRF TE forwarding

Steer **only** VRF CUST_A over the TE tunnel while other VRFs / global traffic keep using LDP — using a **static route in the VRF** (or policy) pointing at the tunnel rather than global autoroute.

**Configuration**

```
! ---- E-R1: remove global autoroute if you want per-VRF selectivity, then ----
router static
 vrf CUST_A
  address-family ipv4 unicast
   <customer-prefix>/24 tunnel-te1     ! only CUST_A rides the TE tunnel
  !
 !
!
! Other VRFs still resolve their BGP next-hop via LDP (no tunnel).
```

`autoroute announce` is **all-or-nothing** — every destination behind the tail-end uses the tunnel. For **selective, per-VRF** steering you bypass autoroute and install a VRF-scoped route (static, or a policy/`forward-class` with PBTS) directing that VRF's traffic into `tunnel-te1`, while leaving global and other-VRF next-hop resolution on LDP. This is how an SP sells a premium "engineered path" to one customer without moving everyone. (In modern designs this per-service steering is exactly what SR-TE color/ODN does more elegantly — Workbook 09.)

**Verification**
- `show route vrf CUST_A <prefix>` — next-hop `tunnel-te1`; other VRFs show LDP-resolved next-hop.
- `show cef vrf CUST_A <prefix> detail` vs another VRF — only CUST_A imposes the TE transport label.
- `traceroute` per VRF — CUST_A via E-R3; other VRF via IGP/LDP path.

---

### Task 5.3 — Forwarding-adjacency

Advertise `tunnel-te1` into IS-IS as a **real link** so *other* routers (not just E-R1) can compute paths through it.

**Configuration**

```
! ---- E-R1 ----
interface tunnel-te1
 forwarding-adjacency                ! advertise tunnel into IGP as a link
 ! optionally set the IGP metric of the advertised adjacency
!
```

`autoroute announce` (Task 1.3) is **local** — only the head-end uses the tunnel; the rest of the network never sees it. **`forwarding-adjacency`** goes further: E-R1 injects the tunnel into IS-IS as a **point-to-point link to E-R2**, so *every* router runs SPF *including* that virtual link and may route transit traffic through it. Use it to create a "virtual topology" (e.g., make a multi-hop TE LSP look like a single IGP hop) so remote nodes steer traffic into it. Caution: it can create routing asymmetry and micro-loops if the reverse direction isn't also modeled — usually you configure it as a pair (one FA each direction).

**Verification**
- `show isis database <E-R1-LSP>` — the tunnel appears as an advertised adjacency/link to E-R2.
- On a **remote** node (e.g., E-R4): `show isis topology` includes the FA; `show route 2.2.2.2` may prefer it.
- Contrast with Task 1.3: autoroute → only E-R1's RIB; forwarding-adjacency → whole-IGP visibility.

---

## Section 6 — Troubleshooting

### Task 6.1 — Tunnel down: CSPF failed (insufficient bandwidth)

**Symptom:** `tunnel-te1` is DOWN; `show mpls traffic-eng tunnels tunnel-te1` reports "No path / CSPF: could not find a path".

**Diagnosis & Fix**

```
! Inspect why CSPF failed:
show mpls traffic-eng tunnels tunnel-te1 detail        ! look for "path error / no path"
show mpls traffic-eng topology                         ! check reservable BW on candidate links
show rsvp interface detail                             ! reserved vs available

! Common cause: signalled-bandwidth exceeds every path's reservable pool.
! Fix option A — lower the request:
interface tunnel-te1
 signalled-bandwidth 200000        ! was e.g. 1200000 > link capacity
!
! Fix option B — raise reservable BW on the links (if physically valid):
rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth 1000000
```

CSPF prunes any link that can't satisfy the request **at the tunnel's setup priority**, so if the requested bandwidth exceeds the reservable pool on **every** feasible path, pruning leaves no route and CSPF returns "no path" — the tunnel never signals. This is a **head-end/control-plane** failure (no RSVP Path is even sent). Distinguish it from Task 6.2: here the failure is *local computation*, there it's *downstream signaling*. Fix by aligning the demand with capacity (lower `signalled-bandwidth`, raise link `bandwidth`, relax affinity, or add a looser fallback `path-option`).

**Verification**
- After fix: `show mpls traffic-eng tunnels tunnel-te1` — UP, path computed.
- `show mpls traffic-eng topology path destination 2.2.2.2 bandwidth <req>` — CSPF now returns a path.

---

### Task 6.2 — Tunnel down: RSVP Path Error (no route to destination)

**Symptom:** CSPF succeeds (or explicit path is set) but the tunnel stays DOWN with an RSVP **PathErr** — a mid-point or the tail can't forward the Path message.

**Diagnosis & Fix**

```
show mpls traffic-eng tunnels tunnel-te1 detail        ! "Path Error: routing problem / no route"
show rsvp session                                      ! where the Path stalls
show isis neighbors  /  show route 2.2.2.2             ! is the tail/next-hop actually reachable?

! Common cause: an explicit-path hop or the destination has no IGP route
! (e.g., IS-IS adjacency down on E-R3<->E-R2, or wrong strict next-address).
! Fix — restore reachability / correct the explicit path:
router isis EMERALD
 interface GigabitEthernet0/0/0/2      ! E-R3<->E-R2 link that was down
  no shutdown
!
! or correct a bad hop:
explicit-path name PE1_via_P1_to_PE2
 index 20 next-address strict ipv4 unicast 10.1.5.2    ! correct E-R2 address
```

A **PathErr "no route to destination"** means RSVP's Path message reached a node that has no IGP route to the next explicit hop or to the tail — signaling can't proceed even though the head-end's CSPF (which used a possibly stale or explicit path) thought a path existed. Typical causes: a **strict** explicit hop pointing at an address that isn't directly connected, an IGP adjacency down along the path, or the tail loopback not in the IGP. Fix reachability (restore the adjacency, correct the `next-address`) so RSVP can walk the path to the tail and return Resv. This is a **downstream signaling** failure, distinct from Task 6.1's head-end CSPF failure.

**Verification**
- `show rsvp session` — Path/Resv now complete end-to-end (no PathErr).
- `show mpls traffic-eng tunnels tunnel-te1 detail` — Oper up, RRO lists all hops to E-R2.

---

### Task 6.3 — Tunnel UP but no traffic (missing autoroute announce)

**Symptom:** `tunnel-te1` shows UP/UP and RSVP is fully signaled, yet traffic to E-R2 still takes the direct IGP link — the tunnel carries nothing.

**Diagnosis & Fix**

```
show mpls traffic-eng tunnels tunnel-te1 brief         ! UP/UP — control plane fine
show route 2.2.2.2                                     ! next-hop is the physical link, NOT tunnel-te1
show mpls traffic-eng autoroute                         ! (empty — nothing announced!)

! Root cause: the LSP is built but nothing tells the FIB to use it.
! Fix — add autoroute (or a static/FA) to attract traffic into the tunnel:
interface tunnel-te1
 autoroute announce
```

A signaled RSVP-TE LSP is just a **forwarding path that nothing points at** until you attract traffic into it. `autoroute announce`, a static route to `tunnel-te1`, or `forwarding-adjacency` are the mechanisms that put the tunnel into the RIB/FIB. Without one of them the tunnel is a "ghost" — perfectly UP, zero traffic. The tell-tale is a healthy `show mpls traffic-eng tunnels` combined with a `show route` whose next-hop is *not* the tunnel. This is the single most common "tunnel works but does nothing" issue.

**Verification**
- After `autoroute announce`: `show route 2.2.2.2` — next-hop `tunnel-te1`.
- `show cef 2.2.2.2` — outgoing interface `tunnel-te1`; `show mpls traffic-eng autoroute` now lists E-R2.
- `traceroute 2.2.2.2 source 1.1.1.1` — path E-R1→E-R3→E-R2; interface counters on the tunnel increment.

---

## CCIE Challenge Tasks

### Challenge A — Gar-R6-delegated RSVP-TE
Point E-R1/E-R2 at **E-R5 (6.6.6.6)** as a stateful Gar-R6 (PCEP) and delegate `tunnel-te1` computation. Have the Gar-R6 compute an inter-area/constrained path and push updates. Contrast Gar-R6-computed RSVP-TE with head-end CSPF, and with SR-PCE (WB09).

### Challenge B — Inter-area / inter-AS TE
Extend a tunnel toward **E-R6 (5.5.5.5)** using **loose** explicit hops and per-area **ERO expansion** (or a Gar-R6) since one CSPF cannot see across areas/ASes. Explain why RSVP-TE needs loose hops + boundary re-computation where SR-TE would use a segment-list.

### Challenge C — Bidirectional & co-routed protection
Model a pair of tunnels (E-R1↔E-R2 both directions) with **SE-style MBB**, node-protecting FRR, and DS-TE sub-pool bandwidth for EF. Prove sub-50ms protection *and* zero-loss reoptimization simultaneously, then compare the total per-node state against the SR TI-LFA equivalent from Workbook 09.
