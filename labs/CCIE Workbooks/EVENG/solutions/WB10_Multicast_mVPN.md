# CCIE SP Workbook 10 — Multicast & mVPN Profiles (Domain 2)

**Platform:** EVE-NG (IOS-XRv 9000 7.11.1)
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** All 3 ISPs (Emerald AS 65100 + Garnet AS 65200 + Gold). Multicast **source in Emerald (CE1, 11.11.11.11)**; **receivers in Garnet (CE4, 32.32.32.32)** and **Gold (CE9)**.
**Format:** Question → Solution → Verification.

> **Topology note:** The base `00_EVENG_Topology.md` documents two SPs (Emerald + Garnet). This workbook follows the task spec, which introduces a third provider **"Gold"** and a receiver **CE9**. Where Gold/CE9 are used they are a documented extension of the base topology (Gold ≈ a third IOS-XR domain peering via a third ASBR); Emerald/Garnet node names, loopbacks and ASNs match the base topology. **RP for the Emerald core = E-R5 (6.6.6.6)** per the task.
>
> **All syntax is IOS-XR** (`multicast-routing`, `router pim`, `router msdp`, `mdt` under the VRF, `router bgp … address-family ipv4 mvpn`). CE nodes (CSR1000v / IOS-XE) use classic IOS multicast syntax where noted.

---

## Section 1 — PIM Basics (4 tasks)

### Task 1.1 — PIM-SM on the Emerald core (RP = E-R5 6.6.6.6, static RP)

**Question**
Enable global multicast routing and PIM sparse-mode on all Emerald core interfaces (E-R1, E-R3, E-R4, E-R6) and set a **static RP = E-R5 (6.6.6.6)**. Advertise 6.6.6.6 as the RP so E-R1 can register the CE1 source and PE-side receivers can build the shared tree.

**Solution**
In IOS-XR, multicast is enabled per-AFI under `multicast-routing`, and PIM is a separate process. Enabling the interface under `multicast-routing address-family ipv4 interface … enable` is what turns PIM on for that link — `router pim` only sets protocol parameters.

```
! ---- Emerald core node (E-R1 shown; repeat on E-R3, E-R4, E-R6) ----
multicast-routing
 address-family ipv4
  interface Loopback0
   enable
  !
  interface GigabitEthernet0/0/0/0
   enable
  !
  interface GigabitEthernet0/0/0/1
   enable
  !
 !
!
router pim
 address-family ipv4
  ! Static RP — everyone points at E-R5
  rp-address 6.6.6.6
 !
!
```

On **E-R5 (the RP, 6.6.6.6)** — it must also run PIM and point at itself:

```
multicast-routing
 address-family ipv4
  interface Loopback0
   enable
  interface GigabitEthernet0/0/0/0
   enable
!
router pim
 address-family ipv4
  rp-address 6.6.6.6
!
```

On the **CE1 source side (IOS-XE, CSR1000v)** classic syntax:

```
ip multicast-routing distributed
interface GigabitEthernet1
 ip pim sparse-mode
ip pim rp-address 6.6.6.6
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show pim rp mapping
  ! Group 224.0.0.0/4 → RP 6.6.6.6 (static)

RP/0/RP0/CPU0:E-R1# show pim neighbor
  ! PIM neighbors on all enabled core interfaces

RP/0/RP0/CPU0:E-R1# show pim interface
  ! sparse-mode, DR elected per segment

RP/0/RP0/CPU0:E-R5# show pim rp mapping
  ! 6.6.6.6 is self / RP for 224/4
```

---

### Task 1.2 — PIM-SSM for the 232.0.0.0/8 range

**Question**
Enable **PIM-SSM** for the standard SSM range **232.0.0.0/8** across the Emerald core so a receiver joining `(S,232.x.x.x)` builds a source-tree directly with **no RP and no shared tree**.

**Solution**
SSM removes the RP entirely: the receiver signals the specific source (IGMPv3 / MLDv2), and PIM builds the SPT straight to the source. In IOS-XR, `232.0.0.0/8` is SSM by default, but you make it explicit (and can add ranges) with an `ssm range` ACL.

```
multicast-routing
 address-family ipv4
  ! Default is 232.0.0.0/8; make explicit / extend if needed
  ssm range SSM-RANGE
!
ipv4 access-list SSM-RANGE
 10 permit ipv4 232.0.0.0 0.255.255.255 any
!
router pim
 address-family ipv4
  ! No rp-address needed for SSM groups
!
```

Receiver CE (IOS-XE) using IGMPv3 for an SSM join:

```
ip pim ssm default
interface GigabitEthernet1
 ip igmp version 3
 ip igmp join-group 232.1.1.1 source 11.11.11.11
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show pim group-map
  ! 232.0.0.0/8 → SSM

RP/0/RP0/CPU0:Gar-R1# show mrib route 232.1.1.1
  ! (11.11.11.11, 232.1.1.1) — SPT only, RPF toward source. No (*,G).

RP/0/RP0/CPU0:E-R1# show pim topology 232.1.1.1
  ! (S,G) with SPT bit set, no RP involvement
```

---

### Task 1.3 — Static RP vs Auto-RP vs BSR

**Question**
Compare and demonstrate the three RP-learning mechanisms on the Emerald core. Keep **static RP** as baseline, then show the **Auto-RP** (Cisco) and **BSR** (RFC 5059) alternatives and when to use each.

**Solution**
All three tell routers *which RP serves which group* for ASM/PIM-SM. They differ in how that mapping is distributed:

- **Static RP** — manually configured on every router (`rp-address`). Simple, deterministic, no failover unless you add Anycast-RP. Best for labs / small stable cores.
- **Auto-RP** — Cisco-proprietary. **Candidate-RPs** announce to `224.0.1.39`; a **Mapping Agent** elects and floods the RP-set to `224.0.1.40`. Needs those two groups flooded (dense-mode fallback or `autorp listen`).
- **BSR** — standards-based. **Candidate-RPs** unicast to the elected **BSR**, which floods the RP-set hop-by-hop in BSR messages (no special dense groups). Preferred in multi-vendor networks.

```
! ---- Auto-RP: E-R5 as Candidate-RP + Mapping Agent ----
router pim
 address-family ipv4
  auto-rp candidate-rp Loopback0 scope 32 group-list AUTORP-GRPS interval 60
  auto-rp mapping-agent Loopback0 scope 32 interval 60
!
ipv4 access-list AUTORP-GRPS
 10 permit ipv4 any 224.0.0.0 15.255.255.255
!
multicast-routing
 address-family ipv4
  ! ensure auto-rp control groups are handled
  interface all enable        ! (lab convenience)
!

! ---- BSR: E-R5 as BSR + Candidate-RP ----
router pim
 address-family ipv4
  bsr candidate-bsr 6.6.6.6 hash-mask-len 30 priority 100
  bsr candidate-rp 6.6.6.6 group-list BSR-GRPS interval 60 priority 100
!
ipv4 access-list BSR-GRPS
 10 permit ipv4 any 224.0.0.0 15.255.255.255
!
```

> Use **only one** dynamic method at a time in the lab (or static). Auto-RP and BSR RP-sets can coexist but complicate election ordering.

**Verification**
```
! Static
show pim rp mapping                 ! "static" as the info source

! Auto-RP
show pim rp mapping                 ! info source = "auto-rp"
show pim auto-rp mapping

! BSR
show pim bsr election               ! elected BSR = 6.6.6.6
show pim bsr rp-cache               ! RP-set learned via BSR
show pim rp mapping                 ! info source = "BSR"
```

---

### Task 1.4 — Verify the mroute table: (S,G) and (\*,G) entries

**Question**
With the CE1 source active to an ASM group (e.g. `239.1.1.1`) and a receiver on a PE, read the **MRIB/MFIB** and explain each entry: the shared tree `(*,G)`, the source tree `(S,G)`, RPF interface, and incoming/outgoing lists. Show the SPT switchover.

**Solution**
In IOS-XR the control-plane multicast route table is the **MRIB** (`show mrib route`) and the forwarding table is the **MFIB** (`show mfib route`) — analogous to IOS `show ip mroute`. ASM starts on the RP-rooted shared tree `(*,G)`; once traffic flows, the last-hop router switches to the shortest-path tree `(S,G)` (SPT) toward the source and prunes off the shared tree. SSM is `(S,G)`-only.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show mrib route
(*,239.1.1.1)
   RPF nbr: 6.6.6.6  (toward RP)
   Incoming Interface List: <toward RP>
   Outgoing Interface List: <toward receivers>
(11.11.11.11,239.1.1.1)
   RPF nbr: <toward source CE1>
   Incoming Interface List: <RPF interface>
   Outgoing Interface List: <toward receivers>
   Flags: (SPT bit set once switched)

RP/0/RP0/CPU0:E-R1# show mfib route 239.1.1.1
  ! hardware forwarding counters incrementing (packets/bytes)

RP/0/RP0/CPU0:E-R1# show pim topology 239.1.1.1
  ! JoinPruneState, RPF interface, SPT bit

RP/0/RP0/CPU0:E-R1# show mrib route summary
  ! count of (*,G) and (S,G)
```
Key checks: RPF **must** succeed (RPF neighbor toward the source/RP) — an RPF failure drops the multicast and shows in `show pim topology`. After SPT switchover the `(S,G)` incoming interface points at the source, not the RP.

---

## Section 2 — MSDP (2 tasks)

### Task 2.1 — MSDP peering between Emerald RP and Garnet RP

**Question**
The Emerald RP (E-R5, 6.6.6.6) and the Garnet RP (P3/RR2, 23.23.23.23) each serve their **own PIM-SM domain**. Configure **MSDP** between them so a source active in Emerald is learned by the Garnet RP, enabling **inter-domain ASM** (Emerald source → Garnet receiver on CE4).

**Solution**
MSDP connects independent PIM-SM domains: when a source registers with its local RP, that RP originates an **SA (Source-Active) message** describing `(S,G)` and floods it to MSDP peers. A remote RP with interested receivers then joins the SPT toward the source across the domain boundary. MSDP runs over **TCP/639**, typically peered loopback-to-loopback. Use the RP address as the MSDP originator-ID so SA RPF checks pass.

```
! ---- Emerald RP: E-R5 (6.6.6.6) ----
router msdp
 originator-id Loopback0
 peer 23.23.23.23
  connect-source Loopback0
 !
!

! ---- Garnet RP: P3/RR2 (23.23.23.23) ----
router msdp
 originator-id Loopback0
 peer 6.6.6.6
  connect-source Loopback0
 !
!
```

> Both domains must have IP reachability between RP loopbacks (via inter-AS BGP / E-R6↔Gar-R7) and consistent PIM-SM + static/Anycast RP within each domain.

**Verification**
```
RP/0/RP0/CPU0:E-R5# show msdp peer
  ! State: Established (TCP/639 up)

RP/0/RP0/CPU0:E-R5# show msdp summary
  ! peer 23.23.23.23 Up, SA count
```

---

### Task 2.2 — SA messages and cache

**Question**
Bring up the CE1 source to an ASM group and confirm the **SA message** propagates from the Emerald RP to the Garnet RP, that the Garnet RP builds `(S,G)` state, and that CE4 receives traffic. Verify the SA cache and SA RPF.

**Solution**
When CE1 starts sending, E-R1 sends a PIM Register to E-R5 (Emerald RP). E-R5 originates an SA for `(11.11.11.11, G)` to its MSDP peer 23.23.23.23. The Garnet RP accepts the SA (passing **SA RPF**: the SA must arrive from the correct peer toward the originating RP) and, if it has receivers for G, joins the SPT toward 11.11.11.11 across the inter-AS link. Traffic then flows Emerald→Garnet.

**Verification**
```
RP/0/RP0/CPU0:E-R5# show msdp sa-cache
  ! (11.11.11.11, 239.1.1.1) originated locally, advertised to peer

RP/0/RP0/CPU0:Gar-R3# show msdp sa-cache
  ! (11.11.11.11, 239.1.1.1) learned via MSDP peer 6.6.6.6

RP/0/RP0/CPU0:Gar-R3# show msdp rpf 11.11.11.11
  ! SA RPF check passes toward originating RP

RP/0/RP0/CPU0:Gar-R1# show mrib route 239.1.1.1
  ! (11.11.11.11,239.1.1.1) built after SA → receiver on CE4 gets traffic
```

---

## Section 3 — mVPN Profile 0 (Default MDT with GRE / PIM-GRE) (3 tasks)

### Task 3.1 — Configure the Default MDT with GRE under the VRF (default-group)

**Question**
For **Customer A VRF** (CE1 on E-R1 ↔ CE4 on Gar-R1), build **Profile 0**: a **Default MDT** using **GRE encapsulation** with **PIM** in the core. Configure the **default-group 239.100.0.0** on E-R1 and Gar-R1 so the PEs form a full-mesh MDT and exchange customer multicast in-band over PIM/GRE.

**Solution**
Profile 0 is the original Rosen mVPN: customer (C-) multicast is encapsulated in **GRE** and carried across the provider core as **P-multicast** using a **Default MDT group** shared by all PEs in the VPN. The core runs PIM-SM (using the RP from Section 1). Each PE joins the Default MDT group; the MDT appears as a virtual LAN so PE-to-PE PIM adjacencies form over it and C-joins/registers ride inside. Discovery and C-signaling are **both PIM** (in-band) in classic Profile 0.

```
! ---- E-R1 and Gar-R1 ----
multicast-routing
 address-family ipv4
  ! core interfaces already enabled (Section 1)
 !
 vrf Customer_A
  address-family ipv4
   ! Default MDT with GRE (Profile 0)
   mdt default ipv4 239.100.0.0
   interface all enable
  !
 !
!
router pim
 vrf Customer_A
  address-family ipv4
   ! customer-side RP for the C-multicast (can be a CE or PE)
   rp-address 6.6.6.6
  !
 !
!
```

Core PIM already provisioned in Section 1 with RP = 6.6.6.6, so the Default MDT group **239.100.0.0** is an ASM group in the core.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show pim vrf Customer_A mdt interface
  ! MDT tunnel interface (mdtCustomer_A) up

RP/0/RP0/CPU0:E-R1# show mrib route 239.100.0.0
  ! Default MDT group (*,G)/(S,G) in the GLOBAL table (P-multicast)

RP/0/RP0/CPU0:E-R1# show pim vrf Customer_A neighbor
  ! Gar-R1 seen as a PIM neighbor OVER the MDT
```

---

### Task 3.2 — Data MDT with a threshold (data-group / S-PMSI)

**Question**
Add a **Data MDT** so a **high-bandwidth** C-stream is moved off the Default MDT onto a dedicated data-group, and **only PEs with interested receivers** join it. Use data-group pool **239.101.0.0/24** with a **threshold of 10 kbps**.

**Solution**
The Default MDT reaches *every* PE in the VPN, wasting bandwidth for streams only a few sites want. When a `(C-S,C-G)` exceeds the **threshold**, the source PE signals a **Data MDT** (S-PMSI) from a data-group pool; PEs with receivers join that group and traffic switches over, sparing uninterested PEs. In Profile 0 this switchover is signaled in-band via PIM.

```
! ---- E-R1 (source PE) ----
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt default ipv4 239.100.0.0
   mdt data 239.101.0.0/24 threshold 10
  !
 !
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show pim vrf Customer_A mdt cache
  ! Data MDT created for the high-bw (C-S,C-G); mapped to a data-group

RP/0/RP0/CPU0:E-R1# show mrib route 239.101.0.0/24
  ! data-group state in the global table; only receiver PEs joined

RP/0/RP0/CPU0:Gar-R1# show pim vrf Customer_A mdt cache
  ! Gar-R1 (has receiver) joined the data-group; PEs without receivers did not
```

---

### Task 3.3 — Verify encapsulation / decapsulation end-to-end

**Question**
Prove the **GRE encap on the ingress PE and decap on the egress PE**: C-multicast from CE1 is GRE-encapsulated into the MDT group on E-R1 and decapsulated on Gar-R1 before delivery to CE4.

**Solution**
The MDT tunnel is a GRE interface (`mdtCustomer_A`). Ingress PE encapsulates C-packets in GRE with the MDT group as outer destination; core forwards as normal P-multicast; egress PE decapsulates and forwards natively toward the receiver. Counters on the MDT interface confirm encap/decap.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show mfib vrf Customer_A route 239.x.x.x
  ! OIF = Encapsulation tunnel (mdtCustomer_A) → encap counters rising

RP/0/RP0/CPU0:Gar-R1# show mfib vrf Customer_A route 239.x.x.x
  ! IIF = Decapsulation tunnel → decap counters rising, OIF toward CE4

RP/0/RP0/CPU0:E-R1# show interfaces mdtCustomer_A
  ! GRE MDT interface packet/byte counters

! End-to-end: CE4 receives the stream sourced by CE1
```

---

## Section 4 — mVPN Profile 3 (BGP AD + PIM C-signaling) (3 tasks)

### Task 4.1 — Enable BGP Auto-Discovery with MVPN NLRI

**Question**
Convert Customer A to **Profile 3**: keep **GRE** transport and **PIM** for C-multicast signaling, but replace PIM-based PE discovery with **BGP Auto-Discovery** using the **`ipv4 mvpn`** address-family. Enable `ipv4 mvpn` on E-R1, Gar-R1 and the RR (P2/RR1, 4.4.4.4).

**Solution**
Profile 3 = **GRE MDT + BGP A-D + PIM C-signaling**. BGP MVPN A-D (Type 1 Intra-AS I-PMSI) replaces the flooding-based discovery of Profile 0 — PEs learn each other's MDT membership via BGP through the RR, which scales far better. C-multicast joins are still carried by **PIM** over the MDT.

```
! ---- E-R1 / Gar-R1 ----
router bgp 65100
 address-family ipv4 mvpn
 !
 neighbor 4.4.4.4          ! RR
  address-family ipv4 mvpn
 !
 vrf Customer_A
  address-family ipv4 mvpn
 !
!
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt default ipv4 239.100.0.0
   ! Profile 3: BGP A-D drives MDT membership
   bgp auto-discovery pim         ! (A-D via BGP, signaling stays PIM)
  !
 !
!

! ---- RR: P2/RR1 (4.4.4.4) ----
router bgp 65100
 address-family ipv4 mvpn
 !
 neighbor 1.1.1.1
  address-family ipv4 mvpn
   route-reflector-client
 neighbor 24.24.24.24
  address-family ipv4 mvpn
   route-reflector-client
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn
  ! Type 1 (Intra-AS I-PMSI A-D) route from each PE (self + Gar-R1)

RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 1
  ! RD:originator — one per PE in the VPN

RP/0/RP0/CPU0:E-R1# show pim vrf Customer_A neighbor
  ! PIM still adjacent over the MDT (C-signaling unchanged)
```

---

### Task 4.2 — Source-active signaling (Type 5) and C-multicast over PIM

**Question**
Bring up the CE1 source and confirm the **MVPN Type 5 (Source Active A-D)** route is originated, while C-joins are still handled by **PIM**.

**Solution**
When a C-source becomes active the source PE originates a **Type 5 (Source Active)** MVPN route announcing `(C-S,C-G)` — this lets receiver PEs learn about active sources via BGP even though the actual C-join uses PIM over the MDT in Profile 3.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 5
  ! (C-S,C-G) Source-Active from E-R1

RP/0/RP0/CPU0:Gar-R1# show bgp ipv4 mvpn
  ! Type 5 learned; PIM builds the C-tree over the MDT

RP/0/RP0/CPU0:Gar-R1# show mrib vrf Customer_A route
  ! (C-S,C-G) present → CE4 receives
```

---

### Task 4.3 — Compare Profile 3 with Profile 0

**Question**
Summarize the difference between Profile 0 and Profile 3 and why you would move from 0 to 3.

**Solution**

| Aspect | Profile 0 | Profile 3 |
|---|---|---|
| Transport | GRE MDT | GRE MDT (same) |
| PE discovery | **PIM** (in-band, flood) | **BGP A-D** (Type 1 via RR) |
| C-multicast signaling | PIM | PIM (same) |
| Scaling | Poor (PIM adjacency mesh over MDT) | Better (BGP/RR for discovery) |
| Source-active | PIM Register/SA | **BGP Type 5** |

Profile 3 keeps the familiar GRE + PIM data/signaling plane but offloads **discovery** to BGP, removing PIM's flood-and-prune scaling problem while requiring no core data-plane change. It is the natural stepping-stone toward fully-BGP profiles (11) and label-based transport (12/14).

**Verification**
```
show bgp ipv4 mvpn route-type 1     ! present in Profile 3, absent in Profile 0
show pim vrf Customer_A neighbor    ! present in BOTH (PIM signaling retained)
```

---

## Section 5 — mVPN Profile 11 (BGP AD + BGP C-signaling, P-tunnel = mLDP) (3 tasks)

### Task 5.1 — Fully BGP-based mVPN with mLDP P-tunnels

**Question**
Migrate Customer A to **Profile 11**: **no PIM anywhere in the core**, discovery **and** C-multicast signaling entirely via **BGP MVPN**, with the **P-tunnel built by mLDP** (P2MP LSP). Enable `mpls mldp` on the core and BGP-driven overlay.

**Solution**
Profile 11 is fully BGP-signaled mVPN over an **mLDP** P-tunnel. The provider core distributes multicast state as **P2MP LSPs via mLDP** — there is **zero PIM** in the P-network. Discovery is BGP Type 1/3; C-multicast joins are BGP **Type 6/7** (Shared-tree / Source-tree C-multicast). This is the most scalable label-based ASM mVPN and the direction real SPs take to eliminate core PIM.

```
! ---- Core P routers (E-R3, E-R4, E-R6, …): enable mLDP ----
mpls ldp
 mldp
!

! ---- E-R1 / Gar-R1 ----
router bgp 65100
 address-family ipv4 mvpn
 neighbor 4.4.4.4
  address-family ipv4 mvpn
 vrf Customer_A
  address-family ipv4 mvpn
!
multicast-routing
 vrf Customer_A
  address-family ipv4
   ! mLDP P2MP default MDT, discovery + signaling via BGP (Profile 11)
   mdt default mldp p2mp
   bgp auto-discovery mldp
   mdt overlay use-bgp
   interface all enable
  !
 !
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R3# show pim neighbor          ! (core) — EMPTY, no core PIM
RP/0/RP0/CPU0:E-R1# show mpls mldp database    ! P2MP LSP root/branches built
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn         ! Types 1,3,4,5,6,7 as applicable
```

---

### Task 5.2 — Verify the BGP MVPN route types (Type 1–7)

**Question**
Enumerate and verify the **7 BGP MVPN route types** (RFC 6514) present in Profile 11 and state what each does.

**Solution**

| Type | Name | Purpose |
|---|---|---|
| 1 | Intra-AS I-PMSI A-D | PE advertises its presence in the VPN (default MDT) |
| 2 | Inter-AS I-PMSI A-D | Carried by ASBRs across AS boundaries |
| 3 | S-PMSI A-D | Source PE announces a Data MDT / selective tree |
| 4 | Leaf A-D | Receiver PE responds to a Type 3, expressing "leaf" interest |
| 5 | Source Active A-D | Advertises an active `(C-S,C-G)` source |
| 6 | Shared Tree C-multicast Join | Receiver PE join toward the C-RP `(C-*,C-G)` |
| 7 | Source Tree C-multicast Join | Receiver PE join toward the C-source `(C-S,C-G)` |

Types **1–5** are **A-D** (auto-discovery); **6–7** are the **C-multicast** (join) routes that replace PIM signaling between PEs.

**Verification**
```
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 1     ! I-PMSI A-D (per PE)
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 3     ! S-PMSI (data MDT)
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 4     ! Leaf A-D responses
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 5     ! Source-Active
RP/0/RP0/CPU0:Gar-R1# show bgp ipv4 mvpn route-type 7     ! C-Source-Tree Join from receiver PE
RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn summary          ! all types learned via RR
```

---

### Task 5.3 — Confirm no core PIM and end-to-end delivery

**Question**
Prove that the P-network carries multicast with **labels only** (mLDP) and that CE1→CE4 delivery works with **no PIM state on any P router**.

**Solution**
With Profile 11 the P routers only hold **mLDP P2MP LSP** state and MPLS forwarding entries — customer multicast is label-switched. PE-to-PE signaling is BGP Type 6/7. Verifying empty PIM on P nodes plus a working stream demonstrates the fully BGP + label-based design.

**Verification**
```
RP/0/RP0/CPU0:E-R3# show pim neighbor         ! empty (no core PIM)
RP/0/RP0/CPU0:E-R3# show mpls forwarding      ! P2MP LSP labels installed
RP/0/RP0/CPU0:Gar-R1# show mrib vrf Customer_A route   ! (C-S,C-G) via BGP Type 7
! CE4 receives the CE1 stream over the mLDP P2MP tree
```

---

## Section 6 — mVPN Profile 12 / 14 (mLDP profiles, partitioned MDT, inter-AS) (3 tasks)

### Task 6.1 — Profile 12: shared mLDP P2MP MDT, default vs data MDT

**Question**
Configure **Profile 12** (mLDP P2MP, BGP A-D + BGP signaling, **shared** default MDT) for Customer A and add a **data MDT**. Contrast the default MDT (all PEs) with the data MDT (only interested PEs).

**Solution**
Profile 12 uses a **shared** mLDP P2MP tree as the Default MDT: all PEs in the VPN are leaves, so every PE receives all default-MDT traffic — like Profile 11 but emphasising the *shared* default tree. A **Data MDT** (S-PMSI, BGP Type 3/4) spins a separate P2MP LSP for high-bandwidth streams that only receiver PEs join.

```
! ---- E-R1 / Gar-R1 (Profile 12) ----
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt default mldp p2mp
   mdt data mldp 100 threshold 10        ! data MDT pool of 100 P2MP LSPs
   bgp auto-discovery mldp
   mdt overlay use-bgp
  !
 !
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show mpls mldp database
  ! default P2MP LSP (all PEs) + separate data P2MP LSP (subset)

RP/0/RP0/CPU0:E-R1# show bgp ipv4 mvpn route-type 3   ! S-PMSI for the data MDT
RP/0/RP0/CPU0:Gar-R1# show bgp ipv4 mvpn route-type 4   ! Leaf A-D (Gar-R1 joins data MDT)
```

---

### Task 6.2 — Profile 14: partitioned MDT (per-source-PE trees)

**Question**
Configure **Profile 14** (**Partitioned MDT**, mLDP P2MP, BGP). Show that each **source PE builds its own P2MP tree** and receiver PEs join **only** the trees that have active sources — contrast with Profile 12's shared tree.

**Solution**
Profile 14 = **Partitioned MDT**. Instead of one shared default tree carrying everything to all PEs, **each source PE roots its own P2MP LSP**; a receiver PE joins (via BGP Type 4 Leaf A-D) **only** the partitioned MDT of a PE that has an active source it wants. This eliminates the "all PEs get everything" waste of shared trees — the most scalable mLDP mVPN and common in large SP deployments.

```
! ---- E-R1 / Gar-R1 (Profile 14) ----
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt partitioned mldp p2mp          ! per-source-PE partitioned trees
   bgp auto-discovery mldp
   mdt overlay use-bgp
  !
 !
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R1# show mpls mldp database
  ! a SEPARATE P2MP LSP rooted at each source PE (not one shared tree)

RP/0/RP0/CPU0:Gar-R1# show bgp ipv4 mvpn route-type 4
  ! Gar-R1 sends Leaf A-D ONLY toward source PEs it wants (partitioned join)

RP/0/RP0/CPU0:Gar-R1# show pim vrf Customer_A mdt cache
  ! receiver PE joined only the partitioned MDT with an active source
```

**Profile 12 vs 14**

| | Profile 12 | Profile 14 |
|---|---|---|
| Default tree | **Shared** — all PEs are leaves | **Partitioned** — one tree per source PE |
| Unwanted traffic | All PEs receive default-MDT traffic | Only interested PEs join a given tree |
| Join model | I-PMSI to everyone | On-demand Leaf A-D per source PE |
| Scale | Good | Best (large SP scale) |

---

### Task 6.3 — Inter-AS mVPN concepts (Emerald ↔ Garnet ↔ Gold)

**Question**
Describe and stage **inter-AS mVPN** so the CE1 source in Emerald reaches receivers in Garnet (CE4) and Gold (CE9). Cover the role of **Type 2 (Inter-AS I-PMSI A-D)** and ASBR behaviour.

**Solution**
Inter-AS mVPN extends the MDT across AS boundaries. **ASBRs** exchange **Type 2 (Inter-AS I-PMSI A-D)** routes and stitch the P-tunnels (Options A/B/C analogues, and for mLDP a **recursive FEC** rooted per-AS). MVPN A-D/C-multicast routes propagate PE→RR→ASBR→(inter-AS)→ASBR→RR→PE. For a segmented inter-AS mLDP tree, each AS builds its own P2MP LSP and the ASBRs graft them. With three domains (Emerald source, Garnet + Gold receivers), each remote AS's ASBR re-originates the A-D toward its PEs so receiver PEs can join across the fabric.

```
! ---- ASBR (inter-AS MVPN A-D) ----
router bgp 65100
 address-family ipv4 mvpn
 neighbor <remote-ASBR>
  address-family ipv4 mvpn         ! exchange Type 2 across AS boundary
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R6# show bgp ipv4 mvpn route-type 2   ! Inter-AS I-PMSI A-D
RP/0/RP0/CPU0:E-R6# show mpls mldp database           ! recursive/segmented P2MP per AS
! End-to-end: CE1 (Emerald) → CE4 (Garnet) AND CE9 (Gold) both receive
```

---

## Section 7 — SR-MVPN (Tree-SID) (2 tasks)

### Task 7.1 — MVPN with SR-MPLS transport (Tree-SID / P2MP SR Policy)

**Question**
Carry Customer A mVPN over **SR-MPLS** using a **Tree-SID** (multicast **P2MP SR Policy**) instead of mLDP, with the **SR-PCE (E-R5, 6.6.6.6)** computing the P2MP tree. Keep BGP MVPN for A-D/signaling.

**Solution**
SR-MVPN replaces the mLDP P-tunnel with a **P2MP SR Policy (Tree-SID)**: the **SR-PCE** computes a P2MP tree and installs it via **replication segments**, each identified by a **Tree-SID** label. There is **no mLDP and no PIM** in the core — the tree is stateless-ish source-routed multicast steered by SR. BGP MVPN still handles discovery/C-signaling; the overlay just points at a Tree-SID P-tunnel.

```
! ---- SR-PCE (E-R5) computes the P2MP tree ----
segment-routing
 traffic-eng
  p2mp
   policy TREE-CUSTA
    color 100
    endpoint <leaf-PEs>
   !
  !
 !
!

! ---- PE (map the VRF default MDT to the Tree-SID P-tunnel) ----
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt default segment-routing mpls        ! Tree-SID P-tunnel
   bgp auto-discovery segment-routing
   mdt overlay use-bgp
  !
 !
!
```

**Verification**
```
RP/0/RP0/CPU0:E-R5# show segment-routing traffic-eng p2mp policy
  ! P2MP SR policy (Tree-SID) computed, leaves = receiver PEs

RP/0/RP0/CPU0:E-R1# show mrib vrf Customer_A route   ! mapped to Tree-SID P-tunnel
RP/0/RP0/CPU0:E-R3#  show mpls forwarding             ! replication-segment labels installed
```

---

### Task 7.2 — Concept of SR replication segments

**Question**
Explain **SR replication segments** and how they compose a Tree-SID P2MP tree; contrast with mLDP.

**Solution**
A **replication segment** is the SR building block for multicast: at each node it says "receive on this SID, then replicate to this set of downstream SIDs." Chaining replication segments from root through transit to leaves forms the full **P2MP tree (Tree-SID)**. Unlike mLDP — which is a separate label-distribution protocol building trees hop-by-hop with per-node LDP state — SR replication segments are **programmed by the controller (SR-PCE)** and ride the existing SR-MPLS data plane, so there is **no extra multicast control protocol** in the core (no mLDP, no PIM). This unifies unicast and multicast on one SR transport and enables centralized, TE-aware tree placement.

**Verification**
```
RP/0/RP0/CPU0:E-R3# show segment-routing traffic-eng p2mp replication-segment
  ! per-node replication segment: incoming Tree-SID → replicate to downstream SIDs

RP/0/RP0/CPU0:E-R1# show segment-routing traffic-eng p2mp policy detail
  ! root, transit, leaf replication segments composing the Tree-SID tree
```

---

## Section 8 — Troubleshooting (3 tasks)

### Task 8.1 — No multicast in the VRF (missing `mdt default`)

**Question**
Customer A receiver CE4 gets **no traffic**. Unicast VPN works, PIM on the CE side is fine, but no C-multicast crosses the core. Diagnose and fix.

**Solution**
The classic Profile-0/3 fault: the VRF has **no `mdt default`** configured (or it is missing on one PE), so **no MDT forms** and there is no P-tunnel to carry C-multicast between PEs. Symptom: `show pim vrf … mdt` empty, no PE-to-PE MDT neighbor. Fix by configuring a matching default MDT on **both** PEs (and ensuring core PIM/mLDP + RP exist).

```
! ---- Fix: add the missing default MDT on the offending PE ----
multicast-routing
 vrf Customer_A
  address-family ipv4
   mdt default ipv4 239.100.0.0       ! MUST match the other PE (Profile 0/3)
   interface all enable
  !
 !
!
```

**Verification**
```
show pim vrf Customer_A mdt interface      ! MDT now UP
show pim vrf Customer_A neighbor           ! remote PE now adjacent over MDT
show mrib vrf Customer_A route             ! (C-*,C-G)/(C-S,C-G) appear → CE4 receives
show mrib route 239.100.0.0                ! Default MDT group present in global table
```

---

### Task 8.2 — Source not registered with the RP

**Question**
CE1 is sending to an ASM group but receivers get nothing and the RP shows no source. `show mrib route` on the RP has no `(S,G)`. Diagnose and fix.

**Solution**
For ASM the first-hop router (E-R1) must **PIM-Register** the source to the RP. Common causes: (a) E-R1 has the **wrong/no RP address** or can't reach 6.6.6.6, (b) **RPF failure** toward the source (unicast route/MRIB mismatch, missing `multicast-routing … enable` on the RPF interface), or (c) the RP rejects the register (accept-register ACL). Check RP reachability, RP-address consistency, and RPF.

```
! Likely fixes:
router pim
 address-family ipv4
  rp-address 6.6.6.6            ! ensure correct RP on the first-hop PE
!
multicast-routing
 address-family ipv4
  interface <RPF-interface-toward-source>
   enable                       ! multicast must be enabled on the RPF path
!
```

**Verification**
```
show pim rp mapping                       ! FHR points at 6.6.6.6, reachable
show pim topology <group>                 ! Register state; no RPF failure
show mrib route <group> on E-R5           ! (S,G) now present on the RP
show pim vrf … | i Register               ! Register/Register-Stop exchange healthy
! RPF sanity:
show pim rpf 11.11.11.11                  ! RPF neighbor resolves toward CE1
```

---

### Task 8.3 — Inter-AS multicast broken

**Question**
Emerald source reaches Garnet fine, but the **Gold** receiver (CE9) — reached across an inter-AS boundary — gets nothing. Inter-domain / inter-AS multicast is broken. Diagnose and fix for both the ASM/MSDP case and the mVPN case.

**Solution**
Two layers to check depending on design:

- **Inter-domain ASM (MSDP, Section 2):** the remote RP never learns the source → **MSDP SA not propagating**. Causes: MSDP peer down (TCP/639 blocked / no loopback reachability), **SA RPF failure** (SA arriving from the wrong peer relative to the originating RP), or missing MSDP mesh-group. Fix peering/RPF.

- **Inter-AS mVPN (Section 6.3):** ASBRs not exchanging **Type 2 Inter-AS I-PMSI A-D**, missing `address-family ipv4 mvpn` on the inter-AS BGP session, or the segmented mLDP/Tree-SID P-tunnel not stitched across the ASBR → receiver PE never joins. Fix ASBR MVPN AFI + P-tunnel stitching, and confirm inter-AS unicast (loopback) reachability underneath.

```
! ---- MSDP case: bring peer up / fix RPF ----
router msdp
 originator-id Loopback0
 peer <remote-RP>
  connect-source Loopback0
!

! ---- mVPN case: enable MVPN AFI on the inter-AS BGP session ----
router bgp <asn>
 neighbor <remote-ASBR>
  address-family ipv4 mvpn
!
```

**Verification**
```
! MSDP path
show msdp peer                    ! Established
show msdp sa-cache                ! source SA present on the remote RP
show msdp rpf <source>            ! SA RPF passes

! mVPN path
show bgp ipv4 mvpn route-type 2   ! Inter-AS I-PMSI A-D present on ASBRs
show bgp ipv4 mvpn summary        ! AFI up on inter-AS session
show mpls mldp database           ! segmented P2MP tree stitched across ASBR
! End-to-end: CE9 (Gold) now receives the CE1 stream
```

---

## Final Validation Checklist
```
[ ] 1.1 PIM-SM core, static RP = E-R5 6.6.6.6 (show pim rp mapping)
[ ] 1.2 PIM-SSM 232.0.0.0/8 — (S,G) only, no RP (show pim group-map)
[ ] 1.3 Static vs Auto-RP vs BSR demonstrated (show pim bsr election / auto-rp mapping)
[ ] 1.4 MRIB/MFIB (*,G) + (S,G), RPF, SPT switchover verified
[ ] 2.1 MSDP peering Emerald RP ↔ Garnet RP Established
[ ] 2.2 SA message + sa-cache, SA RPF, inter-domain delivery to CE4
[ ] 3.1 Profile 0 Default MDT/GRE, default-group 239.100.0.0
[ ] 3.2 Data MDT threshold (S-PMSI), only interested PEs join
[ ] 3.3 GRE encap/decap counters verified end-to-end
[ ] 4.1 Profile 3 BGP A-D (Type 1 MVPN NLRI) + PIM signaling
[ ] 4.2 Source-Active Type 5; C-tree via PIM
[ ] 4.3 Profile 0 vs 3 comparison
[ ] 5.1 Profile 11 fully-BGP, mLDP P-tunnel, no core PIM
[ ] 5.2 BGP MVPN route Types 1–7 enumerated/verified
[ ] 5.3 Core labels-only, no PIM state, CE1→CE4 works
[ ] 6.1 Profile 12 shared mLDP MDT + data MDT
[ ] 6.2 Profile 14 partitioned MDT (per-source-PE trees)
[ ] 6.3 Inter-AS mVPN Type 2, ASBR stitching (Emerald→Garnet→Gold)
[ ] 7.1 SR-MVPN Tree-SID (P2MP SR policy via SR-PCE)
[ ] 7.2 SR replication segments explained/verified
[ ] 8.1 Fix: missing mdt default
[ ] 8.2 Fix: source not registered with RP (RPF/RP reachability)
[ ] 8.3 Fix: inter-AS multicast (MSDP SA RPF / MVPN AFI + P-tunnel stitching)
```
