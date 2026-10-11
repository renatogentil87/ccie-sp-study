# CCIE SP Workbook 13 — Access Connectivity

**Domain:** 3 — Access Connectivity (10%)
**Platform:** EVE-NG (IOS-XRv 9000)
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** Emerald — E-R1, E-R2 + CEs. Access-layer concepts (L2 access, ERPS, MC-LAG, BNG).
**Format:** Question → Solution → Verification.

> **Note on scope:** IOS-XRv 9000 supports L2VPN/L2 access, VLAN rewrite, and MC-LAG/ICCP.
> G.8032 ERPS and full BNG (subscriber/CUPS) are **platform-limited on XRv** — treat those
> sections as **concept + design module preparation** with reference config where syntax applies.

---

## Section 1: Layer 2 Access

### Task 1 — 802.1Q VLAN tagging on a PE-CE link

**Question:**
CE1 connects to E-R1 on `GigabitEthernet0/0/0/1`. Customer traffic arrives tagged with VLAN 100.
Terminate VLAN 100 into an L2 service (bridge/xconnect) on E-R1 using an 802.1Q sub-interface.

**Solution:**
```
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/1.100 l2transport
RP/0/0/CPU0:E-R1(config-subif)# encapsulation dot1q 100
RP/0/0/CPU0:E-R1(config-subif)# rewrite ingress tag pop 1 symmetric   ! optional: strip the tag on ingress
RP/0/0/CPU0:E-R1(config-subif)# exit
!
! Bind the L2 sub-interface into a bridge-domain
RP/0/0/CPU0:E-R1(config)# l2vpn
RP/0/0/CPU0:E-R1(config-l2vpn)# bridge group ACCESS
RP/0/0/CPU0:E-R1(config-l2vpn-bg)# bridge-domain VLAN100
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd)# interface GigabitEthernet0/0/0/1.100
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd-ac)# commit
```

**Verification:**
```
show interfaces GigabitEthernet0/0/0/1.100
  ! State UP, encapsulation 802.1Q VID 100
show l2vpn bridge-domain bd-name VLAN100 detail
  ! AC (attachment circuit) up, MAC learning active
show ethernet tags interface GigabitEthernet0/0/0/1.100
  ! Outer VLAN 100 matched
```

---

### Task 2 — Q-in-Q (802.1ad double tagging) for SP access

**Question:**
Provide a wholesale access service on E-R1 `Gi0/0/0/2`. The customer sends single-tagged frames
(inner C-VLAN, e.g. any of 200-299); the SP adds an outer S-VLAN (S-Tag) of 500 to carry them
across the provider network. Configure the Q-in-Q access sub-interface.

**Solution:**
```
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/2.500 l2transport
! Match outer S-Tag 500, any inner C-Tag
RP/0/0/CPU0:E-R1(config-subif)# encapsulation dot1q 500 second-dot1q any
! (802.1ad ethertype for the outer S-Tag)
RP/0/0/CPU0:E-R1(config-subif)# rewrite ingress tag pop 1 symmetric   ! pop only the outer S-Tag, keep C-Tag
RP/0/0/CPU0:E-R1(config-subif)# commit
!
! Alternative — explicit 802.1ad ethertype on the main interface:
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/2
RP/0/0/CPU0:E-R1(config-if)# dot1q tunneling ethertype 0x88a8
```

**Verification:**
```
show ethernet tags interface GigabitEthernet0/0/0/2.500
  ! Outer S-VLAN 500 (0x88a8), inner C-VLAN = any
show interfaces GigabitEthernet0/0/0/2.500
  ! encapsulation dot1ad 500 / dot1q any
show l2vpn bridge-domain detail   ! frames retain inner C-Tag across core
```

---

### Task 3 — VLAN translation / rewrite on IOS-XR

**Question:**
CE arrives on E-R1 tagged VLAN 100, but the core service expects VLAN 900. Translate (rewrite)
the ingress VLAN 100 to 900 symmetrically so the return traffic maps back correctly.

**Solution:**
```
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/1.100 l2transport
RP/0/0/CPU0:E-R1(config-subif)# encapsulation dot1q 100
! Translate: replace ingress tag 100 with 900 (symmetric = auto-reverse on egress)
RP/0/0/CPU0:E-R1(config-subif)# rewrite ingress tag translate 1-to-1 dot1q 900 symmetric
RP/0/0/CPU0:E-R1(config-subif)# commit
```
Rewrite operation reference:
- `pop 1 symmetric` — remove one tag on ingress, push it back on egress
- `push dot1q <vid>` — add a tag (used for Q-in-Q imposition)
- `translate 1-to-1 dot1q <vid>` — swap one tag for another
- `translate 2-to-1` / `1-to-2` — collapse or expand tag stacks (Q-in-Q ↔ single)

**Verification:**
```
show ethernet tags interface GigabitEthernet0/0/0/1.100
  ! Ingress VID 100, Rewrite = translate to 900
show interfaces GigabitEthernet0/0/0/1.100 | include Rewrite
```

---

### Task 4 — Sub-interface per VLAN (service demux)

**Question:**
CE1 trunks VLANs 10, 20, 30 to E-R1 on `Gi0/0/0/1`. Terminate each VLAN into its own L2 service
(one bridge-domain per VLAN) using one sub-interface per VLAN.

**Solution:**
```
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/1.10 l2transport
RP/0/0/CPU0:E-R1(config-subif)#  encapsulation dot1q 10
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/1.20 l2transport
RP/0/0/CPU0:E-R1(config-subif)#  encapsulation dot1q 20
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/1.30 l2transport
RP/0/0/CPU0:E-R1(config-subif)#  encapsulation dot1q 30
!
RP/0/0/CPU0:E-R1(config)# l2vpn
RP/0/0/CPU0:E-R1(config-l2vpn)# bridge group ACCESS
RP/0/0/CPU0:E-R1(config-l2vpn-bg)#  bridge-domain V10
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd)#   interface GigabitEthernet0/0/0/1.10
RP/0/0/CPU0:E-R1(config-l2vpn-bg)#  bridge-domain V20
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd)#   interface GigabitEthernet0/0/0/1.20
RP/0/0/CPU0:E-R1(config-l2vpn-bg)#  bridge-domain V30
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd)#   interface GigabitEthernet0/0/0/1.30
RP/0/0/CPU0:E-R1(config-l2vpn-bg-bd)# commit
```

**Verification:**
```
show interfaces summary            ! three L2 sub-interfaces UP
show l2vpn bridge-domain brief     ! V10, V20, V30 each with 1 AC
show ethernet tags                 ! per-VID demux confirmed
```

---

## Section 2: G.8032 ERPS (Ethernet Ring Protection Switching)

> ⚠️ **Platform note:** G.8032 is not natively configurable on IOS-XRv 9000. This section is
> **concept + design prep**. Config shown is IOS-XR reference syntax (ASR9000-class) for exam recall.

### Task 5 — ERPS concept, RPL, and RPL Owner

**Question:**
Explain G.8032 Ethernet Ring Protection: what problem it solves, and the roles of the RPL and
the RPL Owner in the ring E-R1–CE1–CE2–E-R2–(back to E-R1).

**Solution (concept):**
- **Problem:** A physical Ethernet ring creates a loop. STP converges too slowly for SP SLAs.
  G.8032 provides **sub-50ms** loop-free protection without running STP.
- **RPL (Ring Protection Link):** One designated link in the ring is **blocked** in normal
  operation to break the loop. All other links forward.
- **RPL Owner:** The node responsible for blocking/unblocking the RPL. Under normal state it
  keeps the RPL blocked; on a ring failure it unblocks the RPL to restore connectivity.
- **R-APS (Ring Automatic Protection Switching):** Control messages on a dedicated control VLAN
  used to signal SF (Signal Fail), NR (No Request), and RPL Block state.
- **States:** *Idle* (RPL blocked, ring healthy) → *Protection* (failure detected, RPL unblocked).

Ring layout (logical):
```
        RPL (blocked in Idle)
   E-R1 ═══════════X══════════ CE1
    ║                          ║
   E-R2 ────────────────────── CE2
```

**Verification (design recall):**
- RPL Owner blocks exactly one link → no loop, no STP.
- Failure anywhere else → RPL Owner unblocks RPL → traffic reroutes the other way around the ring.

---

### Task 6 — Configure the ring (reference syntax)

**Question:**
Configure G.8032 on the ring E-R1–CE1–CE2–E-R2. Make E-R1 the RPL Owner, use control VLAN 4090,
and protect data VLANs 100-200.

**Solution (IOS-XR reference syntax):**
```
! --- On each ring node: define the two ring ports as L2 ---
interface GigabitEthernet0/0/0/3.100 l2transport
 encapsulation dot1q 100-200
interface GigabitEthernet0/0/0/4.100 l2transport
 encapsulation dot1q 100-200
!
! --- Ethernet Ring G.8032 instance ---
l2vpn
 ethernet ring g8032 RING1
  port0 interface GigabitEthernet0/0/0/3
  port1 interface GigabitEthernet0/0/0/4
  instance 1
   description ACCESS-RING
   profile RING-PROFILE
   rpl port0 owner              ! <-- ONLY on E-R1 (the RPL Owner)
   inclusion-list vlan-ids 100-200
   aps-channel
    port0 raps-vlan 4090
    port1 raps-vlan 4090
!
ethernet ring g8032 profile RING-PROFILE
 timer wtr 5
 timer guard 500
```
On CE1, CE2, E-R2: same config **without** the `rpl ... owner` line (they are ring nodes, not owners).

**Verification:**
```
show ethernet ring g8032 RING1
  ! E-R1: RPL Owner, port0 = RPL, State = Idle, RPL = Blocked
  ! Others: State = Idle, both ports forwarding
show ethernet ring g8032 status
```

---

### Task 7 — Failure detection and recovery

**Question:**
The CE1–CE2 link fails. Describe detection, the R-APS signaling, and recovery. Then describe the
revertive behavior when the link is restored.

**Solution:**
1. **Detect:** CE1 and CE2 detect Signal Fail (loss of continuity / CFM) on their shared link.
2. **Signal:** They block the failed port and flood **R-APS(SF)** on VLAN 4090 around the ring.
3. **Recover:** RPL Owner (E-R1) receives R-APS(SF) → **unblocks the RPL** → traffic now flows the
   long way around. Sub-50ms convergence. Nodes flush their MAC tables and relearn.
4. **Restore (revertive):** Link comes back → nodes send **R-APS(NR)** → WTR (wait-to-restore,
   5s here) timer runs to avoid flapping → RPL Owner re-blocks the RPL → ring returns to Idle.

**Verification:**
```
show ethernet ring g8032 RING1
  ! During failure: State = Protection, RPL = Unblocked (forwarding)
  ! Failed port: Blocked, Signal Fail
show ethernet ring g8032 statistics
  ! R-APS SF / NR counters incrementing
! after restore + WTR: State back to Idle, RPL Blocked again
```

---

## Section 3: MC-LAG (Multi-Chassis Link Aggregation)

### Task 8 — MC-LAG concept and ICCP

**Question:**
CE2 is dual-homed to E-R1 and E-R2. Explain MC-LAG and the role of ICCP. Why does CE2 believe it is
connected to a single LACP peer?

**Solution (concept):**
- **MC-LAG:** A single LAG (bundle) from the CE spans **two** physical PE chassis. To the CE, it
  looks like one LACP partner (same LACP System ID), giving link + node redundancy.
- **ICCP (Inter-Chassis Communication Protocol, RFC 7275):** Runs between E-R1 and E-R2 over an
  **LDP-based** control channel. It synchronizes:
  - LACP System ID / port state so the CE sees one logical partner,
  - MAC address / forwarding state,
  - active/standby (or active/active) role for the bundle.
- **Redundancy Group (RG):** Groups the two PEs and the shared PoA (Points of Attachment).
- **Modes:** typically **active/standby** per bundle (one PE forwards; ICCP fails over on loss).

**Verification (concept recall):**
- CE runs standard LACP — unaware two chassis are involved.
- ICCP session UP is the prerequisite for coordinated failover.

---

### Task 9 — Configure MC-LAG (CE2 → E-R1 + E-R2)

**Question:**
Configure MC-LAG so CE2 is dual-homed via `Bundle-Ether1` to E-R1 (primary) and E-R2 (backup),
using ICCP redundancy group 1, LACP System MAC `0000.0000.00cc`.

**Solution (IOS-XR):**
```
! ===== ICCP redundancy group (on BOTH E-R1 and E-R2) =====
redundancy
 iccp
  group 1
   mlacp node 1                       ! node 2 on E-R2
   mlacp system mac 0000.0000.00cc    ! same on both PEs -> CE sees one partner
   mlacp system priority 1
   member
    neighbor 10.0.0.2                 ! E-R2 loopback (E-R1's view); E-R1 loopback on E-R2
!
! LDP must be up between E-R1 and E-R2 (ICCP transport)
mpls ldp
 router-id 10.0.0.1
 neighbor 10.0.0.2
!
! ===== Bundle facing CE2 (on BOTH PEs) =====
interface Bundle-Ether1
 lacp system mac 0000.0000.00cc
 mlacp iccp-group 1
 mlacp switchover recovery-delay 40
 mlacp port-priority 10               ! lower on PRIMARY (E-R1); higher on E-R2
!
interface GigabitEthernet0/0/0/5
 bundle id 1 mode active
```
- On E-R1 set the **lower** `mlacp port-priority` (primary/active).
- On E-R2 use `mlacp node 2` and a higher port-priority (backup/standby).

**Verification:**
```
show iccp group 1
  ! ICCP session state = Connected/Operational; LDP transport UP
show lacp mlacp
  ! Bundle-Ether1: E-R1 = Active, E-R2 = Standby (or Active/Active if configured)
show bundle Bundle-Ether1
  ! Member links up; CE sees single system MAC 0000.0000.00cc
! Failover test: shut E-R1 CE-facing link -> E-R2 becomes Active (ICCP-signaled)
```

---

### Task 10 — MC-LAG vs EVPN multi-homing

**Question:**
Compare MC-LAG (ICCP) with EVPN multi-homing (Type 1/4, ESI). When would you pick each?

**Solution:**

| Aspect | MC-LAG (ICCP) | EVPN Multi-Homing (ESI) |
|---|---|---|
| Control plane | ICCP (LDP-based, pairwise) | BGP EVPN (Type 1 A-D, Type 4 ES) |
| Scale | 2 PEs per RG (pairwise) | N-way, fabric-wide |
| Load balancing | Active/standby (mostly) | All-active (per-flow) or single-active |
| Failover signal | ICCP state sync | Type 1 mass withdrawal (sub-second) |
| DF election | N/A (active/standby role) | DF election via Type 4 |
| Provisioning | Pairwise, tightly coupled | Distributed, loosely coupled |
| Best for | Legacy L2 access, simple dual-homing | Scalable DC/SP fabric, active/active |

**Guidance:**
- **MC-LAG** — brownfield L2 access, two-PE redundancy, no EVPN control plane available.
- **EVPN multi-homing** — greenfield/fabric, need all-active load balancing and N-way scale
  (see Workbook 10, Tasks 6 & 8: Type 4 DF election, all-active ESI).

**Verification (recall):** MC-LAG failover relies on ICCP being UP; EVPN failover relies on BGP
Type 1 withdrawal — no direct PE-to-PE session dependency for the data path.

---

## Section 4: BNG Concepts (Design Module Prep)

> ⚠️ **Platform note:** Full BNG subscriber management is **not deployable on IOS-XRv 9000**.
> This section is **concept + Design module preparation**. Config fragments are IOS-XR reference.

### Task 11 — BNG overview: IPoE and PPPoE

**Question:**
What is a BNG (Broadband Network Gateway)? Contrast the IPoE and PPPoE access models.

**Solution:**
- **BNG:** The aggregation/edge device that terminates residential/business **subscriber**
  sessions, applies per-subscriber policy (QoS, ACLs), assigns addressing, and integrates with AAA.
  (Formerly "BRAS".)
- **PPPoE (Point-to-Point Protocol over Ethernet):**
  - Session-oriented: PADI/PADO/PADR/PADS discovery, then LCP/authentication (PAP/CHAP)/IPCP.
  - Built-in authentication and per-session accounting. Common in DSL.
- **IPoE (IP over Ethernet):**
  - No PPP session; subscriber identified by DHCP / option-82 / MAC / VLAN.
  - Lighter weight, common in modern GPON/metro-E. Auth via DHCP + RADIUS.

Reference (IOS-XR access-interface pointing at a subscriber template):
```
interface GigabitEthernet0/0/0/6.10
 ipsubscriber ipv4 l2-connected
  initiator dhcp
```

**Verification (recall / where supported):**
```
show subscriber session all summary
show pppoe summary
```

---

### Task 12 — Subscriber management, AAA (RADIUS), dynamic templates

**Question:**
How does the BNG apply per-subscriber policy and authenticate/account subscribers via RADIUS?

**Solution:**
- **Dynamic templates:** define per-subscriber attributes (IP, QoS, ACL) applied on session bring-up.
- **AAA / RADIUS:** Authentication (who), Authorization (what policy), Accounting (usage records).
  CoA (Change of Authorization, RFC 5176) allows mid-session policy push (e.g., speed change).
```
aaa authentication subscriber default group radius
aaa authorization subscriber default group radius
aaa accounting subscriber default group radius
!
radius-server host 10.10.10.10 auth-port 1812 acct-port 1813 key <secret>
!
dynamic-template
 type ipsubscriber TEMPLATE-GOLD
  ipv4 unnumbered Loopback0
  service-policy input QOS-GOLD
```

**Verification (recall):**
```
show subscriber session all detail    ! per-subscriber IP, template, state
show radius statistics                ! auth/acct request/response counts
show aaa subscriber session
```

---

### Task 13 — CUPS (Control/User Plane Separation) — Cloud Native BNG

**Question:**
Explain CUPS and how it enables Cloud Native BNG. What are the CP and UP roles?

**Solution:**
- **Monolithic BNG:** control plane (session/PPP/AAA/subscriber state) and user plane (packet
  forwarding, per-sub QoS) live in the **same** chassis → scaling is coupled and coarse.
- **CUPS:** decouples them:
  - **Control Plane (CP):** runs the subscriber logic — PPPoE/IPoE session mgmt, AAA/RADIUS,
    address assignment, policy. Can run **virtualized / containerized** and scale independently.
  - **User Plane (UP):** a leaner forwarding node (physical or virtual) that programs and forwards
    subscriber traffic under CP instruction.
  - **State/programming interface** between CP and UP (e.g., a defined control channel) syncs
    session/forwarding state.
- **Benefit (Cloud Native BNG):** scale CP and UP independently, place UPs near subscribers
  (distributed edge), roll upgrades to CP without dropping the data path, elastic capacity.

**Verification (concept recall — design module):**
- CP scaling (sessions) is independent from UP scaling (throughput).
- UP failure/upgrade should not require re-authenticating all subscribers if CP state persists.

---

## Section 5: Troubleshooting

### Task 14 — Q-in-Q outer tag not preserved (missing rewrite rule)

**Question:**
A Q-in-Q access service on E-R1 `Gi0/0/0/2.500` was expected to carry the customer's inner C-Tag
across the core with the SP S-Tag 500 imposed. Customer reports the **outer S-Tag is missing** on
the far end (frames arrive single-tagged / mis-mapped). Diagnose and fix.

**Diagnosis:**
```
show ethernet tags interface GigabitEthernet0/0/0/2.500
show interfaces GigabitEthernet0/0/0/2.500 | include Rewrite
! Observed: Rewrite ingress = "pop 2" (or a pop that removes BOTH tags),
! so the outer S-Tag is stripped and never re-imposed on egress.
```
Root cause: the **rewrite rule is wrong/missing** — a `pop 2` (or plain `pop 1` without a matching
push on the core side) removes the S-Tag so it is not preserved across the core.

**Solution:**
```
RP/0/0/CPU0:E-R1(config)# interface GigabitEthernet0/0/0/2.500 l2transport
RP/0/0/CPU0:E-R1(config-subif)# encapsulation dot1q 500 second-dot1q any
! Keep the inner C-Tag, pop ONLY the outer S-Tag and re-impose it symmetrically
RP/0/0/CPU0:E-R1(config-subif)# rewrite ingress tag pop 1 symmetric
RP/0/0/CPU0:E-R1(config-subif)# commit
```
(If the service must **impose** the S-Tag onto single-tagged customer frames, use
`rewrite ingress tag push dot1q 500 symmetric` on a single-tagged sub-interface instead.)

**Verification:**
```
show ethernet tags interface GigabitEthernet0/0/0/2.500
  ! Outer S-VLAN 500 present, inner C-Tag preserved
show l2vpn bridge-domain detail
  ! Frames egress the core still double-tagged (S=500 + original C-Tag)
```

---

### Task 15 — MC-LAG failover not working (ICCP session down)

**Question:**
You shut E-R1's CE2-facing link expecting E-R2 to take over `Bundle-Ether1`, but CE2 loses
connectivity — **failover does not occur**. Diagnose and fix.

**Diagnosis:**
```
show iccp group 1
  ! ICCP session state = NOT Connected (Down)  <-- root cause
show mpls ldp neighbor
  ! No LDP session E-R1 <-> E-R2 (ICCP transport is LDP-based)
show lacp mlacp
  ! E-R2 never promoted to Active because it received no ICCP state sync
```
Root cause chain: **ICCP session is DOWN** → PEs cannot synchronize mLACP state → E-R2 does not
know it must become Active → no failover. Common underlying causes:
- LDP session between E-R1/E-R2 not established (routing/loopback reachability, `mpls ldp neighbor`),
- wrong ICCP `member neighbor` IP,
- mismatched `mlacp system mac` / node IDs.

**Solution:**
```
! 1) Restore LDP transport reachability between PE loopbacks
RP/0/0/CPU0:E-R1(config)# mpls ldp
RP/0/0/CPU0:E-R1(config-ldp)#  router-id 10.0.0.1
RP/0/0/CPU0:E-R1(config-ldp)#  neighbor 10.0.0.2       ! E-R2 loopback
!
! 2) Fix ICCP neighbor / identifiers
RP/0/0/CPU0:E-R1(config)# redundancy iccp group 1
RP/0/0/CPU0:E-R1(config-iccp-group)#  member
RP/0/0/CPU0:E-R1(config-iccp-group-member)#   neighbor 10.0.0.2   ! correct E-R2 loopback
RP/0/0/CPU0:E-R1(config)# interface Bundle-Ether1
RP/0/0/CPU0:E-R1(config-if)#  lacp system mac 0000.0000.00cc      ! MUST match on both PEs
RP/0/0/CPU0:E-R1(config-if)#  mlacp iccp-group 1
RP/0/0/CPU0:E-R1(config)# commit
```

**Verification:**
```
show mpls ldp neighbor          ! E-R1<->E-R2 LDP session UP
show iccp group 1               ! ICCP session = Connected/Operational
show lacp mlacp                 ! E-R1 Active, E-R2 Standby (roles synced)
! Re-test: shut E-R1 CE-facing link -> E-R2 promotes to Active, CE2 stays up
```

---

## Final Validation
```
[ ] 802.1Q sub-interface terminates a tagged CE link into an L2 service
[ ] Q-in-Q (802.1ad) outer S-Tag + inner C-Tag access service
[ ] VLAN translation/rewrite (translate 1-to-1, pop/push, symmetric)
[ ] One sub-interface per VLAN (service demux 10/20/30)
[ ] G.8032 ERPS concept: RPL, RPL Owner, R-APS, Idle vs Protection (design)
[ ] ERPS failure detection + sub-50ms recovery + revertive WTR (design)
[ ] MC-LAG concept + ICCP (RFC 7275) single-partner LACP view
[ ] MC-LAG config (Bundle-Ether + ICCP RG, primary/backup)
[ ] MC-LAG vs EVPN multi-homing tradeoffs
[ ] BNG IPoE vs PPPoE models (concept)
[ ] Subscriber mgmt + AAA/RADIUS + CoA + dynamic templates (concept)
[ ] CUPS / Cloud Native BNG: CP vs UP separation (design)
[ ] TS: Q-in-Q outer tag not preserved -> fix rewrite rule
[ ] TS: MC-LAG failover -> restore LDP + ICCP session
```
