# CCIE SP Workbook 08 — L2VPN: VPWS & VPLS (Domain 2)

**Platform:** IOS-XRv 9000 (7.11.1) — EVE-NG
**Topology:** Focus on **Emerald AS 65100** (LDP transport) and **Garnet AS 65200** (SR-MPLS transport) — see `00_EVENG_Topology.md`.
**Format:** Question → Solution → Verification (INE/Narbik style — time yourself per section).

| SP | PEs | Loopbacks | Core Transport |
|----|-----|-----------|----------------|
| Emerald (AS 65100) | E-R1, E-R2 | 1.1.1.1, 2.2.2.2 | LDP (targeted LDP signals PWs) |
| Garnet (AS 65200) | Gar-R1, Gar-R2 | 11.11.11.11, 12.12.12.12 | SR-MPLS (LSP is the SR prefix-SID path) |

> **Scope note:** These loopbacks are the workbook-local L2VPN addressing. The base `00_EVENG_Topology.md` uses different Garnet loopbacks (Gar-R1=24.24.24.24, Gar-R2=25.25.25.25) — if you run this on the full topology, substitute the base loopbacks. All configuration in this workbook uses **IOS-XR `l2vpn` syntax** (there is no `xconnect`-under-interface as in IOS classic; XR uses `l2vpn xconnect group` and `bridge-domain`).

> **Transport-agnostic principle:** VPWS and VPLS are indifferent to how the transport LSP is built. Emerald reaches remote PE loopbacks via **LDP**; Garnet reaches them via **SR-MPLS prefix-SIDs**. The pseudowire's inner **VC label** and the service config are identical either way — only the outer transport label differs. This is the whole point of the MPLS separation of *transport* from *service*.

---

## Section 1 — VPWS / AToM (Point-to-Point Pseudowire)

Point-to-point Ethernet-over-MPLS (E-Line / VPWS) between **E-R1 ↔ E-R2** across the Emerald LDP core. A pseudowire uses a **two-label stack**: outer transport label (LDP/SR) tunnels the frame to the remote PE loopback; inner **VC label** identifies the specific PW at egress. The VC label is signaled by **targeted (directed) LDP** between the two PE loopbacks.

### Task 1.1 — Build the point-to-point pseudowire (VC-ID 100)

**Question**
Configure an AToM pseudowire carrying Customer A's Layer 2 between CE (via E-R1) and CE (via E-R2) using **VC-ID / pw-id 100**, port mode. CEs must reach each other at Layer 2 across the LDP core.

**Solution**
```
! ===== E-R1 (1.1.1.1) =====
interface GigabitEthernet0/0/0/2
 l2transport
!
l2vpn
 xconnect group CUST_A
  p2p E-R1-E-R2-VC100
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 2.2.2.2 pw-id 100
  !
 !
!

! ===== E-R2 (2.2.2.2) =====
interface GigabitEthernet0/0/0/2
 l2transport
!
l2vpn
 xconnect group CUST_A
  p2p E-R1-E-R2-VC100
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 1.1.1.1 pw-id 100
  !
 !
!
```
The `pw-id 100` **must match on both ends** — it is the VC-ID that maps the incoming targeted-LDP label binding to this specific pseudowire. XR auto-establishes a targeted LDP session to `2.2.2.2` when the neighbor is configured; no separate `mpls ldp` targeted statement is required. The whole physical port is mapped (`l2transport` on the main interface = **port mode / EPL**).

**Verification**
```
RP/0/RP0/CPU0:E-R1# show l2vpn xconnect
Legend: ST = State, UP = Up, DN = Down, ...
XConnect                   Segment 1              Segment 2
Group      Name       ST   Description       ST   Description            ST
CUST_A  E-R1-E-R2-VC100 UP   Gi0/0/0/2         UP   2.2.2.2   100          UP
```
- `show l2vpn xconnect detail` — both segments `UP`, PW type `Ethernet`, imposed label stack {transport, VC}.
- `show mpls ldp neighbor 2.2.2.2` — a **targeted** LDP adjacency exists in addition to link-local sessions.
- CE-to-CE ping succeeds (same subnet, L2 adjacency across the core).

### Task 1.2 — VC-ID semantics and VLAN-mode PW

**Question**
Explain the role of the VC-ID, then convert VC-100 to **VLAN mode** so a single access port can carry multiple E-Line services, adding VLAN 200 as a second PW.

**Solution**
```
! ===== E-R1 =====
interface GigabitEthernet0/0/0/2.100 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
!
interface GigabitEthernet0/0/0/2.200 l2transport
 encapsulation dot1q 200
 rewrite ingress tag pop 1 symmetric
!
l2vpn
 xconnect group CUST_A
  p2p VC100
   interface GigabitEthernet0/0/0/2.100
   neighbor ipv4 2.2.2.2 pw-id 100
  !
  p2p VC200
   interface GigabitEthernet0/0/0/2.200
   neighbor ipv4 2.2.2.2 pw-id 200
  !
 !
!
```
The **VC-ID** (pw-id) is the demultiplexer at the egress PE: targeted LDP advertises `<VC-label, VC-ID, PW-type>`, and the remote PE binds the received VC-label to the local AC whose pw-id matches. **Port mode** (whole interface) = EPL; **VLAN mode** (dot1q subinterface per PW) = EVPL, letting one port become several independent services. `rewrite ingress tag pop 1 symmetric` strips the tag on ingress and re-imposes on egress, so the two ends need not agree on the VLAN ID.

**Verification**
- `show l2vpn xconnect group CUST_A` — two separate p2p PWs (VC100, VC200) both `UP`.
- `show l2vpn xconnect detail` on VC100 — shows the `rewrite` action and local/remote VC-ID = 100.

### Task 1.3 — Control word

**Question**
Enable the **control word** on VC-100 and explain why it matters for Ethernet PWs and for fat-PW / ECMP hashing in the core.

**Solution**
```
l2vpn
 pw-class CW-ON
  encapsulation mpls
   control-word
  !
 !
 xconnect group CUST_A
  p2p VC100
   interface GigabitEthernet0/0/0/2.100
   neighbor ipv4 2.2.2.2 pw-id 100
    pw-class CW-ON
   !
  !
 !
!
! (identical pw-class CW-ON applied on E-R2)
```
The **control word** is a 4-byte shim inserted between the VC label and the L2 payload. It (1) preserves frame **sequencing** and prevents mis-ordering, and (2) stops core LSRs from mistaking the customer payload for an IP/MPLS packet during **ECMP load-balancing** — a customer frame whose first nibble is `0x4`/`0x6` could otherwise be hashed as IPv4/IPv6, breaking a flow. The control word **must match on both PEs**; a mismatch brings the PW down (see Section 6). It is mandatory when the PW type requires it (e.g., some VLAN modes) and strongly recommended for Ethernet.

**Verification**
- `show l2vpn xconnect detail` — `Control word enabled` on both segments.
- Mismatch test: enable CW on one side only → PW goes `DN` with `control word mismatch` reason.

### Task 1.4 — PW status signaling

**Question**
Enable **PW status signaling** (LDP status TLV) so AC/PW faults propagate end-to-end, and describe the difference vs label withdrawal.

**Solution**
```
! PW status signaling is on by default in XR l2vpn; verify/force it:
l2vpn
 pw-status                      ! (global — enabled by default)
!
```
With **PW status signaling** (RFC 4447 status TLV), a PE that loses its **attachment circuit** sends a *status notification* to the peer over the targeted-LDP session without tearing the label binding — the far end learns the AC is down and can trigger CE-side action (e.g., interface flap toward the CE, or switchover to a backup PW). Without it, the only signal is **label withdrawal**, which tears the whole PW and loses the granularity of "AC down vs PW down." Status signaling is what makes PW redundancy (Section 2) converge cleanly.

**Verification**
- `show l2vpn xconnect detail` — `PW Status TLV: enabled`; when the AC is shut, status shows `AC DOWN` propagated to the peer.
- `show l2vpn xconnect` on the remote PE reflects the far-end AC fault (`Segment 2` state = `SB`/`DN` accordingly).

---

## Section 2 — VPWS Advanced

### Task 2.1 — Pseudowire redundancy (backup PW)

**Question**
Protect the E-R1 service with a **backup pseudowire** to Gar-R2 (12.12.12.12) so that if the primary PW to E-R2 fails, traffic fails over automatically. Configure immediate switchover and restore.

**Solution**
```
! ===== E-R1 =====
l2vpn
 xconnect group CUST_A
  p2p VC100-REDUNDANT
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 2.2.2.2 pw-id 100          ! primary
    backup neighbor ipv4 12.12.12.12 pw-id 100
     backup-disable-delay 0                  ! revert immediately when primary recovers
    !
   !
  !
 !
!
```
XR VPWS redundancy uses a **primary + backup** PW: only one is active (forwarding) at a time. The backup PW is signaled and held in **standby** (label exchanged but not forwarding). When the primary's PW-status goes down (AC or transport failure), E-R1 activates the backup toward Gar-R2 — driven by the **PW status signaling** from Section 1. `backup-disable-delay 0` reverts to the primary the instant it recovers (set non-zero to dampen flaps).

**Verification**
- `show l2vpn xconnect detail` — primary segment `UP (Active)`, backup `UP (Standby)`.
- Fail the primary (`shut` the core path to E-R2): backup transitions to `Active`; measure packet loss.
- `show l2vpn xconnect summary` — one active, one standby PW for the group.

### Task 2.2 — Preferred-path over a TE tunnel

**Question**
Pin VC-100's transport to a specific **MPLS-TE tunnel** (or SR-TE policy) instead of the IGP-shortest LSP, with fallback disabled so the PW stays down if the tunnel is down.

**Solution**
```
! ===== E-R1 (Emerald, TE tunnel to E-R2) =====
l2vpn
 pw-class TE-STEERED
  encapsulation mpls
   preferred-path interface tunnel-te 12 fallback disable
  !
 !
 xconnect group CUST_A
  p2p VC100
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 2.2.2.2 pw-id 100
    pw-class TE-STEERED
   !
  !
 !
!
```
`preferred-path` forces the PW's **outer transport label** to ride a named TE tunnel (or, on the SR core, a `preferred-path segment-routing traffic-eng policy ...`) rather than the LDP/IGP LSP — giving the L2 service explicit routing, bandwidth guarantees, or affinity constraints. `fallback disable` means if the tunnel goes down the PW does **not** revert to the IGP path — it goes down, enforcing the traffic-engineering contract. Omit `fallback disable` to allow graceful fallback to the shortest LSP.

**Verification**
- `show l2vpn xconnect detail` — `Preferred path: tunnel-te12, fallback disabled`; imposed transport label = the tunnel's.
- `show mpls forwarding tunnels tunnel-te 12` — PW traffic accounted on the tunnel.
- Down the tunnel with `fallback disable` → PW goes `DN` (does not use IGP path).

### Task 2.3 — Static pseudowire

**Question**
Build a **static (manually-labeled) pseudowire** E-R1↔E-R2 with no targeted-LDP signaling — you assign the VC labels by hand. State when this is used.

**Solution**
```
! ===== E-R1 =====
l2vpn
 pw-class STATIC-PW
  encapsulation mpls
   control-word                   ! recommended; must match both ends for static
  !
 !
 xconnect group CUST_A
  p2p VC100-STATIC
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 2.2.2.2 pw-id 100
    mpls static label local 6100 remote 6200
    pw-class STATIC-PW
   !
  !
 !
!

! ===== E-R2 (labels mirrored) =====
l2vpn
 xconnect group CUST_A
  p2p VC100-STATIC
   interface GigabitEthernet0/0/0/2
   neighbor ipv4 1.1.1.1 pw-id 100
    mpls static label local 6200 remote 6100
    pw-class STATIC-PW
   !
  !
 !
!
```
A **static PW** skips targeted LDP entirely — you configure the **local** (in) and **remote** (out) VC labels manually, and they must be **mirror images** across the two PEs (E-R1 local = E-R2 remote). Used where the transport core does not run LDP end-to-end (e.g., static-label islands, inter-provider hand-offs, or SR cores without T-LDP), or where you want deterministic labels for troubleshooting. There is no status TLV signaling, so PW-status/OAM (e.g., VCCV BFD) must carry fault detection instead.

**Verification**
- `show l2vpn xconnect detail` — `Signaling: static`, local/remote VC labels 6100/6200; **no targeted LDP** session created.
- `show mpls forwarding` — static VC label 6100 installed as a local label.
- CE-to-CE traffic passes; verify no `show mpls ldp neighbor` targeted session for this PW.

---

## Section 3 — VPLS Full-Mesh (LDP-signaled, RFC 4762)

Multipoint L2 (E-LAN) across the **Emerald PEs (E-R1, E-R2)** using **LDP-signaled VPLS (RFC 4762)**. VPLS makes the SP core behave as one big learning bridge: each PE has a **bridge-domain** containing local ACs plus a **VFI** whose PW neighbors form a full mesh of pseudowires to every other PE.

### Task 3.1 — LDP-signaled VPLS bridge-domain + VFI

**Question**
Build a full-mesh VPLS instance (VPN-ID 500) across E-R1 and E-R2 so all customer sites share one broadcast domain. Use LDP (Martini) signaling.

**Solution**
```
! ===== E-R1 (1.1.1.1) =====
interface GigabitEthernet0/0/0/3
 l2transport
!
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   interface GigabitEthernet0/0/0/3          ! local AC
   vfi VFI500
    vpn-id 500
    neighbor 2.2.2.2 pw-id 500               ! PW to E-R2 (full mesh)
   !
  !
 !
!

! ===== E-R2 (2.2.2.2) =====
interface GigabitEthernet0/0/0/3
 l2transport
!
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   interface GigabitEthernet0/0/0/3
   vfi VFI500
    vpn-id 500
    neighbor 1.1.1.1 pw-id 500               ! PW to E-R1
   !
  !
 !
!
```
The **bridge-domain** is the emulated LAN: it contains local **attachment circuits** and the **VFI** (Virtual Forwarding Instance). The VFI holds the **PW neighbors** — one PW to every other PE in the mesh. `vpn-id 500` and matching `pw-id` bind the mesh. Each PE **learns MACs** on both ACs and PWs, and floods **BUM** (Broadcast, Unknown-unicast, Multicast) frames to all ports in the bridge-domain.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show l2vpn bridge-domain
Bridge group: CUST_VPLS, bridge-domain: VLAN500, id: 0, state: up
  ACs: 1 (1 up), VFIs: 1, PWs: 1 (1 up)
  List of ACs:
    Gi0/0/0/3, state: up
  List of VFIs:
    VFI VFI500 (up)
      Neighbor 2.2.2.2 pw-id 500, state: up
```
- `show l2vpn bridge-domain detail` — VFI up, PW to 2.2.2.2 up, MAC limit/aging shown.
- CE broadcast/ARP reaches all sites (E-LAN behaviour).

### Task 3.2 — Split-horizon (loop prevention in the mesh)

**Question**
Explain and verify **split-horizon** in the VPLS mesh — why VPLS needs no spanning tree in the core.

**Solution**
```
! Split-horizon among VFI (mesh) PWs is ON by default in XR — no config needed.
! Verify it is active; it is what makes the full mesh loop-free without STP.
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   vfi VFI500
    ! (mesh PWs auto split-horizon group 0 — no forwarding PW→PW)
   !
  !
 !
!
```
VPLS mandates a **full mesh** of PWs and enforces the **split-horizon rule**: a frame received on a **mesh PW is never forwarded out another mesh PW** — only to local ACs (and, in RFC 4762, out spoke PWs). Because every PE has a direct PW to every other PE, one hop suffices; forwarding PW→PW would create loops. This lets VPLS run **without spanning tree** in the core — split-horizon replaces STP. (Local ACs are in a different split-horizon group, so AC→PW and PW→AC are allowed.)

**Verification**
- `show l2vpn bridge-domain detail` — mesh PWs share **Split Horizon Group** 0; PW-to-PW forwarding is blocked.
- Inject a frame on the E-R1↔E-R2 PW → it is delivered only to E-R2's local ACs, never re-flooded onto another mesh PW.

### Task 3.3 — MAC learning and MAC limits

**Question**
Verify dynamic **MAC learning** on the bridge-domain and impose a MAC-address limit with an action, to protect against MAC flooding.

**Solution**
```
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   mac
    limit
     maximum 5000
     action shutdown            ! or 'no-flood' / default 'flood'
    !
    aging time 300
   !
  !
 !
!
```
Each PE builds a **MAC table per bridge-domain**, associating source MACs with the ingress port (local AC or a specific PW). Known-unicast is then switched directly to the right port instead of flooded. The **MAC limit** caps table size per bridge-domain; on breach the `action` (flood / no-flood / shutdown) protects the control plane from MAC-table exhaustion attacks. Aging removes stale entries.

**Verification**
- `show l2vpn forwarding bridge-domain CUST_VPLS:VLAN500 mac-address location 0/RP0/CPU0` — learned MACs mapped to AC or PW.
- `show l2vpn bridge-domain detail` — MAC limit 5000, current count, aging 300s.
- Exceed the limit → configured action fires (log/shutdown).

### Task 3.4 — BUM flooding behaviour

**Question**
Trace how a **broadcast/unknown-unicast/multicast** frame is handled in the VPLS instance and confirm it reaches all PEs exactly once.

**Solution**
```
! No extra config — BUM handling is intrinsic to the bridge-domain.
! Optional: rate-limit BUM to protect the core (storm control):
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   storm-control broadcast pps 1000
   storm-control unknown-unicast pps 1000
   storm-control multicast pps 1000
  !
 !
!
```
A **BUM** frame entering on a local AC is flooded to **all other ports** in the bridge-domain — every local AC and **every mesh PW** (one copy per PW, since there is no core multicast by default → *ingress replication*). Split-horizon then ensures a BUM frame arriving on a mesh PW is flooded only to local ACs, so each remote site receives exactly **one** copy and no loop forms. `storm-control` caps BUM rates to keep a broadcast storm from saturating the core.

**Verification**
- Send an ARP/broadcast from one CE → observe one copy per mesh PW (`show l2vpn forwarding` flood list) and delivery to every remote site once.
- `show l2vpn bridge-domain detail` — storm-control thresholds; drops counted when exceeded.

---

## Section 4 — H-VPLS (Hierarchical VPLS)

H-VPLS reduces the **full-mesh scaling problem** (n PEs need n·(n-1)/2 PWs and n-1 sessions each). It introduces two tiers: **N-PE** (network-facing, in the core full mesh) and **U-PE** (user-facing, at the edge). U-PEs connect to an N-PE by a single **spoke PW**; only N-PEs run the full mesh.

### Task 4.1 — N-PE / U-PE tiers with a spoke PW

**Question**
Make **E-R1 an N-PE** (in the core VPLS mesh) and attach a **U-PE (E-R2 acting as edge)** to it via a single **spoke pseudowire**, so the U-PE needs no full mesh.

**Solution**
```
! ===== N-PE (E-R1, 1.1.1.1) — mesh VFI + spoke toward U-PE =====
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   interface GigabitEthernet0/0/0/3               ! local AC
   neighbor 2.2.2.2 pw-id 600                     ! SPOKE PW to U-PE (in bridge-domain, NOT in VFI)
   vfi VFI500                                     ! core MESH
    vpn-id 500
    neighbor 12.12.12.12 pw-id 500                ! to other N-PE
   !
  !
 !
!

! ===== U-PE (E-R2, 2.2.2.2) — only a spoke PW upward, no mesh =====
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   interface GigabitEthernet0/0/0/3               ! customer AC
   neighbor 1.1.1.1 pw-id 600                     ! single spoke PW up to N-PE
  !
 !
!
```
The **spoke PW** is placed **directly in the bridge-domain** (as `neighbor ... pw-id`), **not** inside the `vfi`. That placement is deliberate: mesh PWs (in the VFI) are split-horizoned, but the spoke PW is **exempt from split-horizon** — an N-PE *must* forward frames between the spoke PW and its mesh PWs, otherwise the U-PE's traffic could never cross the core. The U-PE runs a single PW and no mesh, collapsing its PW count from n-1 to **1**.

**Verification**
- `show l2vpn bridge-domain detail` on N-PE — VFI mesh PWs (split-horizon on) **plus** a spoke PW (split-horizon off).
- U-PE `show l2vpn bridge-domain` — exactly one PW (the spoke), no VFI.

### Task 4.2 — Spoke PW exempt from split-horizon

**Question**
Prove the spoke PW is **not** subject to the mesh split-horizon rule, and explain why that is safe.

**Solution**
```
! Confirming behaviour (no new config): the N-PE forwards
!   spoke-PW  <-> mesh-PW   and   spoke-PW <-> local-AC.
! It is safe because U-PE has only ONE uplink (or active/standby dual-homing),
! so no loop can form even though split-horizon is relaxed on the spoke.
```
On a normal mesh PW, split-horizon blocks PW→PW to prevent loops in the fully-meshed core. A **spoke PW** sits in a different split-horizon group, so the N-PE **relays** spoke↔mesh traffic. This is safe because the U-PE has no second path into the mesh (single-homed, or dual-homed with only one **active** spoke PW). If a U-PE were dual-homed active/active into two N-PEs without loop control, a loop *would* form — which is why H-VPLS spoke redundancy uses **primary/backup** (active/standby) spoke PWs.

**Verification**
- Frame from U-PE arrives on the N-PE spoke PW → N-PE floods it out its **mesh PWs** and local ACs (would be blocked if it were a mesh PW).
- `show l2vpn forwarding bridge-domain ... detail` — spoke PW in split-horizon group ≠ mesh group.

### Task 4.3 — Scaling win vs full-mesh

**Question**
Quantify the scaling reduction H-VPLS delivers and describe the redundancy option for the spoke.

**Solution**
```
! Redundant spoke (U-PE dual-homed to two N-PEs, active/standby):
! ===== U-PE (E-R2) =====
l2vpn
 bridge group CUST_VPLS
  bridge-domain VLAN500
   interface GigabitEthernet0/0/0/3
   neighbor 1.1.1.1 pw-id 600                 ! primary spoke to N-PE (E-R1)
    backup neighbor 12.12.12.12 pw-id 600     ! backup spoke to a second N-PE
   !
  !
 !
!
```
Full-mesh VPLS needs **n·(n-1)/2** PWs and every PE holds **n-1** targeted-LDP sessions — this explodes as sites grow, and each new N-PE must be added to *every* existing PE's mesh. **H-VPLS** keeps the full mesh only among a small set of **N-PEs**, while many **U-PEs** each ride a single spoke PW. Adding a customer edge = adding one U-PE with one spoke, touching one N-PE — no core reconfiguration. Spoke resilience is provided by **active/standby backup spoke PWs** to two N-PEs (no loop, unlike active/active).

**Verification**
- `show l2vpn bridge-domain detail` on U-PE — primary spoke `UP (Active)`, backup `UP (Standby)`.
- Fail the primary N-PE path → backup spoke activates; count PWs to confirm U-PE still holds only its spoke(s), not a mesh.

---

## Section 5 — BGP VPLS (Kompella) & EVPN-VPLS

LDP-VPLS (Section 3, RFC 4762) requires **manual full-mesh** neighbor configuration and has no auto-discovery. **BGP-VPLS (Kompella, RFC 4761)** adds **BGP auto-discovery + BGP signaling**: PEs discover each other and exchange PW labels via a **label block**, eliminating manual mesh config. **EVPN-VPLS** is the modern successor, using BGP EVPN (route-types) with control-plane MAC learning.

### Task 5.1 — BGP auto-discovery + signaling VPLS (Kompella)

**Question**
Configure **BGP-VPLS (RFC 4761)** on the Garnet PEs (Gar-R1, Gar-R2) so PWs are auto-discovered and signaled by BGP, using a **label block**. Compare with LDP-VPLS.

**Solution**
```
! ===== Gar-R1 (11.11.11.11) =====
router bgp 65200
 address-family l2vpn vpls
 !
 neighbor 12.12.12.12
  remote-as 65200
  update-source Loopback0
  address-family l2vpn vpls
 !
!
l2vpn
 bridge group KOMPELLA
  bridge-domain VPLS700
   interface GigabitEthernet0/0/0/3
   vfi VFI700
    vpn-id 700
    autodiscovery bgp
     rd 65200:700
     route-target 65200:700
     signaling-protocol bgp
      ve-id 3                 ! this PE's VPLS-edge ID
      ve-range 16             ! size of the label block
     !
    !
   !
  !
 !
!

! ===== Gar-R2 (12.12.12.12) — same, with ve-id 4 =====
l2vpn
 bridge group KOMPELLA
  bridge-domain VPLS700
   interface GigabitEthernet0/0/0/3
   vfi VFI700
    vpn-id 700
    autodiscovery bgp
     rd 65200:700
     route-target 65200:700
     signaling-protocol bgp
      ve-id 4
      ve-range 16
     !
    !
   !
  !
 !
!
```
`autodiscovery bgp` makes each PE advertise an **L2VPN-VPLS NLRI** carrying its **VE-ID** and a **label block** (base VC label + block size = `ve-range`). A remote PE computes the VC label for the mesh as `label-base + (local VE-ID − VE-block-offset)`, so **one BGP advertisement** provisions PWs to *all* peers at once — no per-neighbor config. The **RD** makes the NLRI unique; the **RT** controls which PEs join the instance. Adding a PE only requires giving it a unique **VE-ID**; discovery and signaling are automatic — the key advantage over manual LDP-VPLS.

**Verification**
```
RP/0/RP0/CPU0:Gar-R1# show bgp l2vpn vpls
   Network            Next Hop     ... (VE-ID / label-block entries per PE)
RP/0/RP0/CPU0:Gar-R1# show l2vpn bridge-domain
   ... VFI VFI700 (up), autodiscovery BGP, PWs auto-created to VE-id 4 ...
```
- `show bgp l2vpn vpls summary` — BGP L2VPN-VPLS session up between Gar-R1/Gar-R2.
- `show l2vpn bridge-domain detail` — VFI with **BGP autodiscovery**, PWs auto-provisioned (no manual `neighbor`).
- `show l2vpn discovery` — VE-IDs and label blocks exchanged.

### Task 5.2 — EVPN-VPLS (control-plane MAC learning)

**Question**
Deliver the same multipoint L2 service using **EVPN** (BGP EVPN) instead of LDP/BGP-VPLS, so MAC addresses are learned in the **control plane** and advertised as **Type-2** routes.

**Solution**
```
! ===== Gar-R1 (11.11.11.11) =====
router bgp 65200
 address-family l2vpn evpn
 !
 neighbor 12.12.12.12
  remote-as 65200
  update-source Loopback0
  address-family l2vpn evpn
 !
!
evpn
 evi 800
  bgp
   rd 65200:800
   route-target import 65200:800
   route-target export 65200:800
  !
!
l2vpn
 bridge group EVPN
  bridge-domain VPLS800
   interface GigabitEthernet0/0/0/3
   evi 800                       ! bind bridge-domain to EVPN instance
  !
 !
!
```
EVPN replaces data-plane flood-and-learn with **BGP control-plane MAC distribution**: a PE that learns a local MAC advertises it as an **EVPN Type-2 (MAC/IP) route** to all peers; Type-3 (inclusive multicast) routes build the BUM flood list; Type-1/Type-4 handle multi-homing (ESI, DF election). This gives **MAC mobility**, **all-active multi-homing**, and far less unknown-unicast flooding than VPLS. It is the modern replacement for both LDP- and BGP-VPLS.

**Verification**
- `show bgp l2vpn evpn` — Type-2 (MAC), Type-3 (IMET) routes present.
- `show l2vpn bridge-domain detail` — bridge-domain bound to `evi 800`, EVPN PWs up.
- `show evpn evi 800 mac` — remote MACs learned via BGP, not flooding.

### Task 5.3 — Compare LDP-VPLS vs BGP-VPLS vs EVPN

**Question**
Summarize the trade-offs so you can justify a choice in the exam.

**Solution**

| Attribute | LDP-VPLS (4762) | BGP-VPLS / Kompella (4761) | EVPN-VPLS |
|-----------|-----------------|----------------------------|-----------|
| Auto-discovery | **No** (manual mesh) | **Yes** (BGP) | **Yes** (BGP) |
| Signaling | Targeted LDP | BGP (label block) | BGP EVPN (route-types) |
| MAC learning | Data-plane flood-and-learn | Data-plane flood-and-learn | **Control-plane** (Type-2) |
| Multi-homing | STP / MST hacks | Limited | **All-active** (ESI, DF) |
| MAC mobility | Reconverge by flooding | Same | **Native** (seq #) |
| Scaling | Poor (n² PWs) | Good (label block) | Best |
| Provisioning to add a PE | Touch every PE | Assign a VE-ID | Assign EVI/ESI |

LDP-VPLS is simplest for a few PEs but does not auto-discover. BGP-VPLS solves discovery/signaling and scales via label blocks but still floods to learn MACs. **EVPN** is the strategic choice: control-plane MAC learning, active/active multi-homing, mobility — which is why CCIE-SP weights EVPN (see Workbook 10) heavily.

**Verification**
- Be able to read each `show` family: `show mpls l2transport vc` (LDP), `show bgp l2vpn vpls` (Kompella), `show bgp l2vpn evpn` (EVPN) and state which learning model each uses.

---

## Section 6 — Troubleshooting

### Task 6.1 — PW down: VC-ID / pw-id mismatch

**Question**
A VPWS PW E-R1↔E-R2 is **down**. Diagnose and fix.

**Solution**
```
RP/0/RP0/CPU0:E-R1# show l2vpn xconnect
CUST_A  VC100  DN   Gi0/0/0/2  UP   2.2.2.2  100  DN
RP/0/RP0/CPU0:E-R1# show l2vpn xconnect detail
  ... PW: neighbor 2.2.2.2, PW ID 100 ...
  ... Status: mismatched pw-id / no remote binding ...
```
Root cause: the two PEs use **different `pw-id`** (e.g., E-R1 pw-id 100, E-R2 pw-id 101). Targeted LDP advertises the VC label keyed by VC-ID, so the far end never finds a matching binding and the PW stays **down** with the AC up. Fix: make **pw-id identical on both ends**.
```
! ===== E-R2 — correct the pw-id =====
l2vpn xconnect group CUST_A p2p VC100
 no neighbor ipv4 1.1.1.1 pw-id 101
 neighbor ipv4 1.1.1.1 pw-id 100
```

**Verification**
- `show l2vpn xconnect` — segment 2 to 2.2.2.2 now `UP`.
- `show l2vpn xconnect detail` — matching local/remote VC-ID = 100.

### Task 6.2 — PW up but no traffic: MTU or control-word mismatch

**Question**
The PW shows **UP** on both ends but the CEs cannot pass traffic (or only small frames pass). Diagnose.

**Solution**
```
RP/0/RP0/CPU0:E-R1# show l2vpn xconnect detail
  MTU: 1500 (local) / 1500 (remote)          <-- if these differ, PW won't come up or drops
  Control word: enabled (local) / disabled (remote)   <-- mismatch
```
Two classic causes:
1. **MTU mismatch** — the L2VPN advertises an interface MTU in the LDP/BGP binding; if E-R1=1500 and E-R2=1400 the PW may stay down or silently drop oversized frames. Fix by aligning `mtu` on the l2transport interfaces (and ensure core MTU accounts for label stack + control word).
2. **Control-word mismatch** — one PE has `control-word`, the other does not. XR requires **agreement**; a mismatch either keeps the PW down or corrupts framing so payload is discarded.
```
! Align both:
interface Gi0/0/0/2  mtu 1500          ! same on both PEs
l2vpn pw-class CW-ON encapsulation mpls control-word    ! apply on BOTH ends
```

**Verification**
- `show l2vpn xconnect detail` — `MTU` equal on both segments, `Control word: enabled` on both.
- Full-size (1500B / jumbo) CE-to-CE traffic passes without drops.

### Task 6.3 — VPLS MAC not learned: split-horizon block or AC down

**Question**
In the VPLS instance, a remote site's MAC never appears in E-R1's MAC table and its traffic is missing. Diagnose.

**Solution**
```
RP/0/RP0/CPU0:E-R1# show l2vpn bridge-domain detail
  AC Gi0/0/0/3, state: DOWN            <-- (a) attachment circuit down
  VFI VFI500: PW 2.2.2.2 up
RP/0/RP0/CPU0:E-R1# show l2vpn forwarding bridge-domain CUST_VPLS:VLAN500 mac-address location 0/RP0/CPU0
  (remote MAC absent)
```
Two common root causes:
1. **AC down** — the customer-facing `l2transport` interface is down (or not in the bridge-domain), so nothing is learned/forwarded locally. Fix: bring up / correctly place the AC in the bridge-domain.
2. **Split-horizon blocking** — the frame arrived on a **mesh PW** and is being (correctly) blocked from other mesh PWs, but the topology mistakenly relies on **PW→PW** forwarding (a missing full mesh). In a proper full mesh this is expected; if a PE is **not** meshed to the source PE, that MAC can only arrive via another PE's mesh PW and split-horizon drops it. Fix: ensure a **complete full mesh** (every PE has a PW to every other PE), or use an **H-VPLS spoke** (exempt from split-horizon) if the PE is meant to be a U-PE.
```
! (a) fix AC:
interface Gi0/0/0/3 l2transport         ! ensure up + in bridge-domain
! (b) fix mesh: add the missing VFI neighbor
l2vpn bridge group CUST_VPLS bridge-domain VLAN500 vfi VFI500
 neighbor 2.2.2.2 pw-id 500
```

**Verification**
- `show l2vpn bridge-domain detail` — AC `up`, all expected mesh PWs `up`.
- `show l2vpn forwarding bridge-domain CUST_VPLS:VLAN500 mac-address ...` — remote MAC now learned against the correct PW/AC.
- CE-to-CE traffic across the affected site resumes.

---

## CCIE Challenge Tasks

- **Challenge A — Flow-aware transport (FAT-PW):** add a **flow label** (`load-balancing flow-label both`) to a VPWS PW so the SR/LDP core can ECMP-hash per-flow without deep inspection; verify improved core load-balancing.
- **Challenge B — VCCV BFD on a static PW:** enable **VCCV BFD** fault detection on the Section 2.3 static pseudowire (no LDP status TLV available) and prove sub-second failure detection.
- **Challenge C — Inter-AS VPLS:** stitch a VPLS instance across the E-R6↔Gar-R7 hand-off (Emerald LDP ↔ Garnet SR), demonstrating multi-segment PW or EVPN inter-AS, and contrast with single-AS behaviour.
- **Challenge D — Migrate LDP-VPLS → EVPN:** convert the Section 3 LDP-VPLS bridge-domain to EVPN-VPLS (Section 5.2) with minimal outage; confirm MACs move from flood-and-learn to Type-2 control-plane learning.
```
