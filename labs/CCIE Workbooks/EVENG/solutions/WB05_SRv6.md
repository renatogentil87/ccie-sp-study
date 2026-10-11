# CCIE SP Workbook 05 — SRv6 (Domain 1 + Domain 2)

**Platform:** Cisco IOS-XRv 9000 — EVE-NG
**Focus AS:** Gold **AS 65300** (IS-IS + SRv6 native data plane)
**Related:** `09_Segment_Routing_SR_MPLS.md` (SR-MPLS control-plane primitives carry over); `10_EVPN.md` (EVPN VLAN100 on CE7).

## Topology — Gold AS 65300

| Node | Router-ID | Role | SRv6 Locator (/48) |
|------|-----------|------|--------------------|
| G-R4 | 24.24.24.24 | ASBR + Route-Reflector + Gar-R6 | `fc00:0:24::/48` |
| G-R5 | 25.25.25.25 | ASBR (border to Garnet AS via Gar-R7) | `fc00:0:25::/48` |
| G-R3    | 23.23.23.23 | P (transit) | `fc00:0:23::/48` |
| G-R1   | 21.21.21.21 | PE (CUST_A + CUST_B) | `fc00:0:21::/48` |
| G-R2   | 22.22.22.22 | PE (CUST_A) | `fc00:0:22::/48` |

**CEs:** CE8 (AS 65012, **dual-homed** to G-R1 + G-R2, VRF CUST_A) · CE9 (AS 65013, single-homed G-R1, VRF CUST_B) · CE7 (EVPN VLAN100).

> **SRv6 in one sentence:** the segment (SID) is a **128-bit IPv6 address**, so the SR data plane *is* the IPv6 data plane — no MPLS, no LDP, no separate label space. A SID = `Locator (routed by IGP) : Function (behavior on the owning node) : Args`. The IGP advertises the `/48` locator; every SID under it is reachable via plain IPv6 longest-prefix match, and the owning node's My-SID table decides the behavior (End, End.X, End.DT4…).

---

## Section 1 — SRv6 Foundation

### Task 1.1 — IPv6 addressing on all Gold core links
**Question:** Bring up IPv6 on every Gold core link so SRv6 SIDs (which are IPv6 addresses) are natively routable. Use a `fc00:0:<A><B>::/64` scheme keyed on the two node IDs; loopbacks are not needed as `/128` for SRv6 forwarding but keep them for IS-IS router-id continuity.

**Solution**

SRv6 forwards on IPv6 destination address alone, so **every core link must have IPv6 unicast enabled and an IS-IS IPv6 address-family** — otherwise the locator `/48` cannot be flooded and SIDs are black-holed. Unlike SR-MPLS (which only needed `mpls` on the interface implicitly via SR), SRv6 needs genuine end-to-end IPv6 routing. There is no `mpls ip`, no label imposition on the wire; the SID sits in the IPv6 header's destination field (and, for multi-segment paths, in an **SRH** — Segment Routing Header, IPv6 extension header type 43, routing-type 4).

```
! ---- G-R3 <-> G-R1 link example (fc00:0:2324::/64) ----
interface GigabitEthernet0/0/0/0
 description G-R3---G-R1
 ipv6 address fc00:0:2324::23/64
 no shutdown
!
! ---- G-R1 side ----
interface GigabitEthernet0/0/0/0
 description G-R1---P6
 ipv6 address fc00:0:2324::24/64
 no shutdown
```

**Verification**
- `show ipv6 interface brief` — all core links `Up/Up` with global IPv6.
- `ping ipv6 fc00:0:2324::23` from G-R1 across the directly-connected link.

### Task 1.2 — SRv6 locator per router
**Question:** Configure the SRv6 locator block and a per-node locator so each router owns a `/48` from which its SIDs are carved.

**Solution**

The **locator** is the routable prefix that identifies a node in the SRv6 domain; all of that node's SIDs are longest-prefix-matched into it, so advertising the single `/48` reaches every End/End.X/End.DT SID the node owns — the same "one prefix, many behaviors" scaling that makes SRv6 summarizable across domains. The global block groups locators for summarization. IOS-XR splits the SID into **Locator (48) : Function (16) : rest**; the `micro-segment`/uSID variants change the layout, but classic full-SID (used here) keeps Locator = 48 bits.

```
segment-routing
 srv6
  encapsulation
   source-address fc00:0:21::24        ! G-R1 outer IPv6 SA for encapsulated (T.Encaps) traffic
  !
  locators
   locator GOLD
    prefix fc00:0:21::/48              ! G-R1's locator; G-R4 uses fc00:0:24::/48, etc.
   !
  !
 !
!
```

> Repeat per node with its own `/48`: G-R4 `fc00:0:24::/48`, G-R5 `fc00:0:25::/48`, G-R3 `fc00:0:23::/48`, G-R1 `fc00:0:21::/48`, G-R2 `fc00:0:22::/48`.

**Verification**
- `show segment-routing srv6 locator GOLD detail` — locator `Up`, prefix correct.
- `show segment-routing srv6 sid` — SIDs beginning to allocate under the locator (populated fully after Task 1.3).

### Task 1.3 — IS-IS SRv6 (advertise the locator)
**Question:** Enable SRv6 under IS-IS so the locator is flooded and the node auto-allocates its topological SIDs.

**Solution**

Under SR-MPLS you wrote `segment-routing mpls` under IS-IS; under SRv6 you bind the **locator** into IS-IS with `segment-routing srv6 locator GOLD`. This does two things: (1) IS-IS floods the `/48` locator as an IPv6 reachability TLV so every node can route toward it, and (2) IS-IS triggers auto-allocation of the topological SIDs — one **End** SID per node (node reachability, the SRv6 analog of a prefix-SID) and one **End.X** SID per IS-IS adjacency (link-specific, the analog of an adjacency-SID). Crucially, the IS-IS **IPv6 address-family must be enabled**, or the locator is not carried (this is the #1 SRv6 bring-up failure — see Section 6).

```
router isis GOLD
 is-type level-2-only
 net 49.0000.0000.0024.00                ! G-R1
 address-family ipv6 unicast              ! MANDATORY for SRv6 locator flooding
  metric-style wide                       ! wide metrics required (SR sub-TLVs)
  segment-routing srv6
   locator GOLD
  !
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv6 unicast
 !
 interface Loopback0
  address-family ipv6 unicast
 !
!
```

**Verification**
- `show isis database verbose <node>` — SRv6 Locator TLV and SRv6 End SID sub-TLV present.
- `show route ipv6 fc00:0:21::/48` on a remote node — locator reachable via IS-IS.

### Task 1.4 — Verify auto-allocated SIDs (End, End.X)
**Question:** Confirm the node auto-allocated an **End** SID and per-adjacency **End.X** SIDs, and read the My-SID table.

**Solution**

The **My-SID table** is the SRv6 forwarding table: for each locally-instantiated SID it maps the IPv6 destination to a behavior (End = "I'm a waypoint, decrement Segments-Left and continue to the next SID"; End.X = "cross-connect out this specific link"). These are the exact equivalents of prefix-SID (global, shortest-path) and adjacency-SID (local, specific link) from SR-MPLS Workbook 09 — the control-plane semantics are identical; only the data plane (IPv6 header vs MPLS label) changes. Auto-allocation means you do not hand-pick these; IS-IS carves them from the locator.

**Verification**
- `show segment-routing srv6 sid` — one `End` SID and per-adjacency `End.X` SIDs, all `InUse`, behavior column shows `End (PSP/USD)` and `End.X`.
- `show segment-routing srv6 sid <sid> detail` — shows owning locator, behavior, and (for End.X) the outgoing interface/next-hop.

### Task 1.5 — Encapsulation source-address
**Question:** Set the SRv6 encapsulation source-address on each node and explain why it matters.

**Solution**

When a PE performs **H.Encaps** (headend encapsulation) — pushing an outer IPv6 header (plus SRH if multi-segment) onto a customer packet destined for a remote End.DT SID — the **source-address** becomes the outer IPv6 SA. It must be a locally-routable, ideally locator-derived address so return-path ICMPv6 errors and any RPF checks succeed. Omitting it (or setting it to a non-routable address) causes silent drops of encapsulated VPN traffic even though the control plane looks healthy. Best practice: use an address inside the node's own locator (or Loopback0 IPv6).

```
segment-routing
 srv6
  encapsulation
   source-address fc00:0:21::24     ! G-R1
```

**Verification**
- `show segment-routing srv6 | include Source` / `show running-config segment-routing srv6 encapsulation`.
- After L3VPN is up (Section 3), capture an encapsulated packet: outer IPv6 SA equals this address.

---

## Section 2 — SRv6 SID Types (Endpoint Behaviors)

> Each SID type is an **endpoint behavior** defined by RFC 8986. The behavior is what the *owning* node does when a packet's IPv6 DA matches that SID in its My-SID table. Below, each is explained with the config that instantiates it.

### Task 2.1 — End SID (node reachability)
**Question:** Explain and verify the **End** SID.

**Solution**

**End** is the SRv6 shortest-path waypoint — "route to me, then look at the next segment." On match, the node executes the **PSP/USP/USD** flavor: it decrements **Segments-Left (SL)** in the SRH, copies the next SID from the SRH segment-list into the IPv6 DA, and forwards. It is the SRv6 equivalent of a prefix-SID and is auto-allocated by IS-IS (Task 1.3) — you do not configure it explicitly; it is derived from the locator (`Locator:End-function`). It provides pure node reachability and is the building block of any SRv6 path.

**Verification**
- `show segment-routing srv6 sid` — `End` behavior row.
- `traceroute srv6` toward the End SID follows the IGP shortest path.

### Task 2.2 — End.X SID (specific link / adjacency)
**Question:** Explain and verify the **End.X** SID.

**Solution**

**End.X** is "cross-connect: decrement SL, next SID → DA, and force the packet out *this specific adjacency*" — bypassing the IGP shortest path for that hop. It is the SRv6 adjacency-SID: locally significant, auto-allocated one-per-IS-IS-adjacency. A segment-list of End.X SIDs is strict source routing (hop-by-hop). It is what SRv6-TE (Section 4) and TI-LFA repair paths are built from.

**Verification**
- `show segment-routing srv6 sid` — `End.X` rows, one per adjacency, each showing the bound interface.
- `show isis segment-routing srv6 adjacency-sid` — per-neighbor End.X SIDs.

### Task 2.3 — End.DT4 (per-VRF IPv4 decapsulation)
**Question:** Explain and configure **End.DT4** for VRF CUST_A on G-R1.

**Solution**

**End.DT4** = "**D**ecapsulate and do a **T**able lookup in an IPv**4** VRF." When a packet's IPv6 DA matches an End.DT4 SID, the node strips the outer IPv6/SRH and performs an IPv4 FIB lookup **in the VRF that owns the SID** — the SRv6 equivalent of the per-VRF VPN label in MPLS L3VPN. Because the SID itself identifies the VRF, thousands of VPNs share one locator with no MPLS label space. `End.DT4` is used with **per-VRF** allocation mode (one SID per VRF; the egress does a route lookup, so it works for any prefix in the VRF).

```
vrf CUST_A
 address-family ipv4 unicast
 !
!
router bgp 65300
 vrf CUST_A
  rd auto
  address-family ipv4 unicast
   segment-routing srv6
    locator GOLD
    alloc mode per-vrf          ! => one End.DT4 SID for the whole VRF
   !
  !
 !
!
```

**Verification**
- `show segment-routing srv6 sid` — an `End.DT4` SID associated with VRF CUST_A.
- `show bgp vpnv4 unicast vrf CUST_A <prefix>` — route carries the End.DT4 SID (not an MPLS label).

### Task 2.4 — End.DT6 and End.DX4
**Question:** Explain **End.DT6** (IPv6 VRF decap) and **End.DX4** (per-CE IPv4 delivery); contrast DT vs DX.

**Solution**

**End.DT6** is the IPv6 twin of End.DT4: decapsulate and look up in the VRF's **IPv6** table (used when the customer runs IPv6 inside the VPN). **End.DX4** is "**D**ecapsulate and **X**-connect to a specific **CE** IPv4 next-hop" — instead of a VRF table lookup it forwards directly to a pre-bound adjacency (per-CE granularity). The distinction mirrors classic L3VPN:
- **DT (per-VRF/per-table)** = decap + FIB lookup → one SID scales to all prefixes/CEs in the VRF (fewer SIDs, needs a lookup).
- **DX (per-CE)** = decap + direct cross-connect to one CE → skips the lookup, but consumes one SID per CE (used for aggregation labels / EVPN, or when you want to steer to an exact CE).

```
! Per-CE (End.DX4) mode example:
router bgp 65300
 vrf CUST_A
  address-family ipv4 unicast
   segment-routing srv6
    locator GOLD
    alloc mode per-ce            ! => End.DX4, one SID per CE next-hop
   !
  !
 !
!
```

**Verification**
- `show segment-routing srv6 sid` — `End.DT6` (per-VRF IPv6) and/or `End.DX4` (per-CE) behaviors present.
- `show bgp vrf CUST_A ipv6 unicast <prefix>` for DT6; per-CE SID appears bound to the CE next-hop for DX4.

---

## Section 3 — L3VPN over SRv6

### Task 3.1 — VRF CUST_A on G-R1 + G-R2 (CE8, AS 65012)
**Question:** Build VRF CUST_A on both G-R1 and G-R2, each with an eBGP session to dual-homed CE8 (AS 65012).

**Solution**

CUST_A is dual-homed, so both PEs run the VRF and peer with CE8; MP-BGP (Section 3.2) carries CE8's prefixes between PEs as VPN routes tagged with an SRv6 End.DT4 SID. Because CE8 attaches to two PEs, standard L3VPN multihoming applies (allocate distinct RDs so both paths are advertised to the RR and best-path/add-path can expose both). RD `auto` derives a unique RD per PE from the router-id, giving that path diversity for free.

```
! ---- G-R1 and G-R2 (identical pattern) ----
vrf CUST_A
 address-family ipv4 unicast
  import route-target 65300:100
  export route-target 65300:100
 !
!
router bgp 65300
 vrf CUST_A
  rd auto
  address-family ipv4 unicast
   redistribute connected
  !
  neighbor 10.8.5.8               ! CE8 (use 10.8.6.8 on G-R2)
   remote-as 65012
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
```

**Verification**
- `show bgp vrf CUST_A summary` — CE8 eBGP session `Established` on both G-R1 and G-R2.
- `show route vrf CUST_A` — CE8 prefixes learned.

### Task 3.2 — VRF CUST_B on G-R1 (CE9, AS 65013)
**Question:** Build VRF CUST_B on G-R1 only, peering eBGP with CE9 (AS 65013).

**Solution**

CUST_B is single-homed to G-R1. Same construction as CUST_A but a distinct route-target (`65300:200`) so the two VPNs stay isolated. Later (Task 3.5) we deliberately leak/route between CUST_A and CUST_B within Gold to prove CE8↔CE9 reachability, which requires importing each other's RTs.

```
vrf CUST_B
 address-family ipv4 unicast
  import route-target 65300:200
  export route-target 65300:200
 !
!
router bgp 65300
 vrf CUST_B
  rd auto
  address-family ipv4 unicast
   redistribute connected
  !
  neighbor 10.9.5.9
   remote-as 65013
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
```

**Verification**
- `show bgp vrf CUST_B summary` — CE9 session `Established`.
- `show route vrf CUST_B` — CE9 prefixes present on G-R1.

### Task 3.3 — MP-BGP with encapsulation-type SRv6
**Question:** Configure the VPNv4 MP-BGP overlay (PEs → RR G-R3) and set **encapsulation-type srv6** so VPN routes carry SRv6 SIDs.

**Solution**

The overlay is ordinary VPNv4/VPNv6 MP-BGP; what makes it SRv6 is two things on each PE: (1) the per-VRF `segment-routing srv6 / alloc mode per-vrf` from Task 2.3 (which allocates the End.DT4/DT6 SID), and (2) advertising the VPN NLRI with the **SRv6 SID attribute** rather than an MPLS VPN label. On IOS-XR the SID is attached automatically once the VRF has an SRv6 locator + alloc mode; the **encapsulation-type srv6** knob (per-neighbor / per-AF) ensures the SID is signaled and that the egress PE encapsulates in IPv6 (H.Encaps) instead of imposing MPLS. G-R4 is the Route-Reflector, so G-R1/G-R2 peer only to it.

```
router bgp 65300
 address-family vpnv4 unicast
 !
 address-family vpnv6 unicast
 !
 neighbor 24.24.24.24              ! G-R4 = RR (+Gar-R6)
  remote-as 65300
  update-source Loopback0
  address-family vpnv4 unicast
   encapsulation-type srv6         ! signal SRv6 SID, not MPLS label
  !
  address-family vpnv6 unicast
   encapsulation-type srv6
  !
 !
!
```

> On G-R4 (RR): reflect the client sessions with `route-reflector-client` under each AF; the RR does not need a VRF or its own DT SID.

**Verification**
- `show bgp vpnv4 unicast summary` — PE↔RR sessions `Established`.
- `show bgp vpnv4 unicast vrf CUST_A <prefix> detail` — path shows an **SRv6 SID** (End.DT4) and **no** MPLS VPN label.

### Task 3.4 — alloc mode per-vrf & verify SRv6 SID (not MPLS label)
**Question:** Confirm per-VRF allocation produced an End.DT4 SID and that VPN routes carry it instead of a label.

**Solution**

`alloc mode per-vrf` produces exactly one **End.DT4** SID per VRF (per-table decap). The proof that the data plane is truly SRv6 — and not accidentally SR-MPLS — is in the BGP VPN path: the "Label" field should be absent/`nolabel` and an **SRv6-VPN SID** should be attached, carved from the locator (`fc00:0:24:eXXX::` style). If you see a real MPLS label here, `encapsulation-type srv6` or the per-VRF SRv6 alloc is missing (Section 6, Task 6.2).

**Verification**
- `show segment-routing srv6 sid` — `End.DT4` SID for CUST_A, `End.DT4`/`End.DT6` for CUST_B.
- `show bgp vpnv4 unicast vrf CUST_A <prefix> detail` — `PSID`/`SRv6-VPN SID: fc00:0:24:...`, `Received Label: nolabel`.
- `show cef vrf CUST_A <prefix>` — next-hop resolves via SRv6 (`SRv6 Headend`), not a labelled path.

### Task 3.5 — CE8 ↔ CE9 reachability within Gold
**Question:** Prove end-to-end reachability between CE8 (CUST_A) and CE9 (CUST_B) across the SRv6 core.

**Solution**

To reach across two different VRFs you must interconnect them — either route-leak by cross-importing RTs on G-R1 (which holds both VRFs) or model it as an extranet. Once leaked, CE8's route to CE9 resolves to CUST_B's End.DT SID on G-R1, and CE9→CE8 resolves to CUST_A's End.DT SID (on G-R1 or G-R2 depending on best-path). The forwarding path is: CE8 → (H.Encaps at ingress PE, outer IPv6 DA = egress PE's DT4 SID) → SRv6 core routes on the locator `/48` → egress PE matches DT4, decaps, VRF lookup → CE. This is a pure IPv6 data plane end to end.

```
! Extranet leak on G-R1 (holds both VRFs):
vrf CUST_A
 address-family ipv4 unicast
  import route-target 65300:200      ! import CUST_B
!
vrf CUST_B
 address-family ipv4 unicast
  import route-target 65300:100      ! import CUST_A
```

**Verification**
- From CE8: `ping <CE9 loopback>` succeeds.
- `traceroute` from CE8 to CE9 traverses the Gold SRv6 core.
- `show route vrf CUST_A <CE9-prefix>` on G-R1 — resolves via CUST_B's SRv6 SID.

---

## Section 4 — SRv6-TE

### Task 4.1 — Explicit SRv6-TE policy (segment-list of SRv6 SIDs)
**Question:** On G-R1, build an **explicit SRv6-TE policy** to G-R2 that traverses G-R3 (non-shortest / pinned path) using a segment-list of SRv6 SIDs.

**Solution**

An SRv6-TE policy encodes the path as an ordered **segment-list of SRv6 SIDs** placed in the **SRH**; transit nodes hold **no policy state** — they just route each SID on its locator (End) or cross-connect the link (End.X), exactly like SR-MPLS SR-TE but with IPv6 SIDs instead of labels. The head-end (G-R1) does H.Encaps, writing the first SID into the IPv6 DA and the remainder into the SRH, with **Segments-Left** = number of segments still to process. To pin via G-R3, list G-R3's End SID then G-R2's End (or End.DT) SID.

```
segment-routing
 traffic-eng
  segment-list VIA_P6
   index 10 sid fc00:0:23::           ! G-R3 End SID (waypoint)
   index 20 sid fc00:0:22::           ! G-R2 End SID (endpoint)
  !
  policy PE5_TO_PE6_VIAP6
   color 100 end-point ipv6 fc00:0:22::25
   candidate-paths
    preference 100
     explicit segment-list VIA_P6
    !
   !
  !
 !
!
```

**Verification**
- `show segment-routing traffic-eng policy` — policy `Up`, segment-list installed, endpoint = G-R2.
- `traceroute srv6` toward G-R2 via the policy transits G-R3 (not the shortest path).

### Task 4.2 — Traffic steering with color
**Question:** Steer CUST_A traffic into the policy using BGP **color**.

**Solution**

As in SR-MPLS, steering is by **color + endpoint**, decoupling "which path" from "which prefixes." Tag CUST_A VPN routes (or the ODN template) with color 100; BGP resolves the VPN next-hop over the color-100 SRv6-TE policy to G-R2. This enables **On-Demand Next-hop (ODN)** for SRv6: a policy is auto-instantiated per color/endpoint when a colored VPN route arrives — no per-prefix tunnel config, scaling to thousands of prefixes.

```
extcommunity-set opaque COLOR_100
 100
end-set
!
route-policy SET_COLOR
 set extcommunity color COLOR_100
end-policy
!
! Apply outbound on the egress PE's VRF export (or inbound at head-end) so CUST_A carries color 100.
```

**Verification**
- `show bgp vpnv4 unicast vrf CUST_A <prefix> detail` — color:100 extended community present.
- `show cef vrf CUST_A <prefix>` — resolves via the SRv6-TE policy (SRH imposed toward G-R2).

### Task 4.3 — Verify SRH in packet (Segments-Left)
**Question:** Confirm the Segment Routing Header is imposed and read **Segments-Left**.

**Solution**

The **SRH** is IPv6 extension header (Next-Header 43, Routing-Type 4). It carries the segment-list (in *reverse* order, last segment first), **Segments-Left (SL)** as a pointer, and **Last-Entry**. At the head-end SL = (#segments − 1) and IPv6 DA = the first active SID. Each **End** node decrements SL and copies `Segment-List[SL]` into DA. When SL reaches 0 the final behavior (e.g., End.DT4) executes. Watching SL decrement hop-by-hop is the definitive proof the path is source-routed through the listed SIDs.

**Verification**
- `show segment-routing traffic-eng policy PE5_TO_PE6_VIAP6 detail` — SID list + expected SRH depth.
- Packet capture (or `show`) on G-R3: SRH present, `Segments Left` decremented by one at the waypoint; DA rewritten to the next SID.
- `show segment-routing srv6 sid <G-R3-End-SID>` on G-R3 — `Packets/Bytes` counters incrementing as steered traffic passes.

---

## Section 5 — SRv6 Inter-domain (Gold ↔ Garnet)

### Task 5.1 — SRv6 ↔ SR-MPLS boundary at G-R5 ↔ Gar-R7
**Question:** Gold runs SRv6; the Garnet AS runs SR-MPLS. Describe/configure the interworking boundary at G-R5 (Gold) ↔ Gar-R7 (Garnet).

**Solution**

The two domains use different data planes — Gold = IPv6/SRv6, Garnet = MPLS labels — so the ASBR pair is a **data-plane stitching point**. The clean pattern is **Inter-AS Option B (VPN route stitching)**: G-R5 and Gar-R7 run eBGP VPNv4/VPNv6 and *rewrite* the transport on their respective side. A VPN route received from Garnet with an MPLS VPN label is re-advertised into Gold with a locally-allocated **SRv6 End.DT SID** (and vice-versa); the ASBR becomes the next-hop, so the SRv6→MPLS translation happens in its forwarding path. This keeps each domain internally pure (no SRv6 in Garnet, no MPLS in Gold) while services span both.

```
! G-R5 (Gold side) — eBGP to Gar-R7, SRv6 into Gold, next-hop-self
router bgp 65300
 neighbor <Gar-R7-link-ip>
  remote-as <Garnet-AS>
  address-family vpnv4 unicast
   route-policy PASS in
   route-policy PASS out
   next-hop-self
  !
 !
 ! toward Gold RR (G-R4): re-advertise with SRv6 encapsulation
 neighbor 24.24.24.24
  address-family vpnv4 unicast
   encapsulation-type srv6
  !
 !
!
```

**Verification**
- `show bgp vpnv4 unicast` on G-R5 — routes from Garnet, next-hop = G-R5, re-originated with SRv6 SID toward Gold.
- End-to-end ping a Garnet CE from a Gold CE traverses the stitched boundary.

### Task 5.2 — SRv6-to-MPLS interworking concept
**Question:** Explain the packet-level interworking as a packet crosses G-R5→Gar-R7.

**Solution**

At the boundary the ASBR **terminates one transport and imposes the other**. For a Gold→Garnet flow: the packet arrives at G-R5 as an SRv6-encapsulated packet whose DA is G-R5's End.DT/stitching SID; G-R5 decapsulates (removes outer IPv6/SRH), does the VPN lookup, and forwards toward Gar-R7 imposing the **MPLS** VPN label + transport label that Garnet's SR-MPLS understands. Return traffic does the mirror: MPLS is popped and an SRv6 SID is imposed. Because BGP already advertised each domain's own transport identifier (SID vs label) for the same VPN prefix, the ASBR's FIB holds both encapsulations and swaps between them — a per-packet re-encapsulation, not tunneling one inside the other.

**Verification**
- `show cef vrf <vrf> <remote-prefix>` on G-R5 — ingress SRv6 decap, egress MPLS label imposition (or vice-versa).
- Captures on each side: IPv6/SRH on the Gold link, MPLS label stack on the Garnet link, same customer payload.

### Task 5.3 — End.B6 binding SID at the boundary
**Question:** Explain the **End.B6** binding SID and its role at the domain boundary.

**Solution**

**End.B6** (Encaps / Insert) is the SRv6 **binding SID (BSID)**: matching it triggers the node to **bind in a new SRv6 policy** — i.e., insert (End.B6.Insert) or encapsulate with (End.B6.Encaps) a fresh SRH/segment-list before forwarding. At a domain boundary this gives **path-stitching and abstraction**: the upstream domain steers to a single BSID (one SID, opaque) and the boundary node expands it into the downstream domain's full segment-list. This hides the downstream topology, reduces the head-end's SRH depth (scalability), and lets each domain manage its own SIDs independently — the SRv6 analog of the SR-MPLS binding-SID that stitches inter-domain SR-TE policies. (End.B6.Encaps adds a new outer IPv6 header + SRH; End.B6.Insert only inserts an SRH into the existing packet.)

**Verification**
- `show segment-routing srv6 sid` on the boundary node — `End.B6.Encaps`/`End.B6.Insert` behavior bound to a local SRv6-TE policy.
- Steer to the BSID from the upstream domain; capture shows the boundary node expanding it into the downstream segment-list (new SRH imposed).

---

## Section 6 — Troubleshooting

### Task 6.1 — SRv6 SID not allocated (locator not under IS-IS)
**Symptom:** `show segment-routing srv6 sid` is empty (or only shows the local End for the locator but nothing propagates); remote nodes have no route to the `/48`.

**Root cause & fix.** The locator was defined under `segment-routing srv6 locators` but **not bound into IS-IS** (`segment-routing srv6 / locator GOLD` under the IS-IS instance/AF). Without that binding IS-IS never floods the locator TLV nor triggers End/End.X allocation. Fix:

```
router isis GOLD
 address-family ipv6 unicast
  segment-routing srv6
   locator GOLD
```

**Verify fix:** `show isis database verbose | include SRv6` shows the Locator/End SID TLVs; `show segment-routing srv6 sid` now lists End + End.X; remote `show route ipv6 fc00:0:21::/48` resolves.

### Task 6.2 — VPN route shows MPLS label instead of SRv6 SID
**Symptom:** `show bgp vpnv4 unicast vrf CUST_A <prefix> detail` shows a real MPLS **label** and no SRv6 SID; data plane tries to impose MPLS on an IPv6-only core → drops.

**Root cause & fix.** Missing **`encapsulation-type srv6`** on the MP-BGP session (and/or missing per-VRF `segment-routing srv6 / alloc mode per-vrf`), so BGP defaulted to MPLS VPN-label signaling. Fix both:

```
router bgp 65300
 neighbor 24.24.24.24
  address-family vpnv4 unicast
   encapsulation-type srv6
!
 vrf CUST_A
  address-family ipv4 unicast
   segment-routing srv6
    locator GOLD
    alloc mode per-vrf
```

**Verify fix:** re-check the path detail — `SRv6-VPN SID: fc00:0:24:...`, `Received Label: nolabel`; `show cef vrf CUST_A <prefix>` shows `SRv6 Headend`.

### Task 6.3 — IPv6 reachability broken (IS-IS IPv6 AF not enabled)
**Symptom:** Locators are configured and bound, but remote nodes still cannot reach the `/48`; `ping ipv6` across the core fails on some links.

**Root cause & fix.** The IS-IS **IPv6 unicast address-family** is missing — either globally (`router isis / address-family ipv6 unicast`) or, more commonly, on specific **interfaces** (`interface … / address-family ipv6 unicast`). IS-IS then does not compute IPv6 SPF over those links, so the locator (an IPv6 prefix) is unreachable even though the SID is allocated locally. This is the most common SRv6 bring-up failure because SR-MPLS habits omit the IPv6 AF. Fix on every core interface and the instance:

```
router isis GOLD
 address-family ipv6 unicast
  metric-style wide
 !
 interface GigabitEthernet0/0/0/0
  address-family ipv6 unicast
 !
 interface Loopback0
  address-family ipv6 unicast
```

**Verify fix:** `show isis interface brief` — IPv6 active on all core links; `show route ipv6 fc00:0:2X::/48` resolves on every node; end-to-end `ping ipv6` succeeds; SRv6 VPN and TE now forward.

---

## CCIE Challenge Tasks

### Challenge A — SRv6 uSID (micro-segment) compression
Convert the locator layout to **uSID (uSID/G-SID)**: a single 128-bit address carries a *stack* of 16- or 32-bit micro-SIDs, so multi-hop TE paths fit in the IPv6 DA with **no SRH** for shallow stacks. Contrast SRH depth and hardware forwarding cost vs the full-SID design used above.

### Challenge B — SRv6 TI-LFA
Enable `fast-reroute per-prefix ti-lfa` for the IPv6 AF and prove sub-50ms protection of an SRv6 VPN flow by failing a core link; inspect the repair path expressed as an SRv6 End.X segment list.

### Challenge C — EVPN over SRv6 (CE7 VLAN100)
Extend the CE7 EVPN VLAN100 service over the SRv6 core (End.DT2U/End.DT2M for L2), and contrast the SRv6 L2VPN SID allocation with the L3VPN End.DT4/DT6 used in Section 3.
