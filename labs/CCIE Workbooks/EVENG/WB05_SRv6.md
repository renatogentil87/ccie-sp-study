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


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — SRv6 locator per router
**Question:** Configure the SRv6 locator block and a per-node locator so each router owns a `/48` from which its SIDs are carved.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — IS-IS SRv6 (advertise the locator)
**Question:** Enable SRv6 under IS-IS so the locator is flooded and the node auto-allocates its topological SIDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Verify auto-allocated SIDs (End, End.X)
**Question:** Confirm the node auto-allocated an **End** SID and per-adjacency **End.X** SIDs, and read the My-SID table.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.5 — Encapsulation source-address
**Question:** Set the SRv6 encapsulation source-address on each node and explain why it matters.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — SRv6 SID Types (Endpoint Behaviors)

> Each SID type is an **endpoint behavior** defined by RFC 8986. The behavior is what the *owning* node does when a packet's IPv6 DA matches that SID in its My-SID table. Below, each is explained with the config that instantiates it.

### Task 2.1 — End SID (node reachability)
**Question:** Explain and verify the **End** SID.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — End.X SID (specific link / adjacency)
**Question:** Explain and verify the **End.X** SID.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — End.DT4 (per-VRF IPv4 decapsulation)
**Question:** Explain and configure **End.DT4** for VRF CUST_A on G-R1.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4 — End.DT6 and End.DX4
**Question:** Explain **End.DT6** (IPv6 VRF decap) and **End.DX4** (per-CE IPv4 delivery); contrast DT vs DX.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — L3VPN over SRv6

### Task 3.1 — VRF CUST_A on G-R1 + G-R2 (CE8, AS 65012)
**Question:** Build VRF CUST_A on both G-R1 and G-R2, each with an eBGP session to dual-homed CE8 (AS 65012).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — VRF CUST_B on G-R1 (CE9, AS 65013)
**Question:** Build VRF CUST_B on G-R1 only, peering eBGP with CE9 (AS 65013).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — MP-BGP with encapsulation-type SRv6
**Question:** Configure the VPNv4 MP-BGP overlay (PEs → RR G-R3) and set **encapsulation-type srv6** so VPN routes carry SRv6 SIDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4 — alloc mode per-vrf & verify SRv6 SID (not MPLS label)
**Question:** Confirm per-VRF allocation produced an End.DT4 SID and that VPN routes carry it instead of a label.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.5 — CE8 ↔ CE9 reachability within Gold
**Question:** Prove end-to-end reachability between CE8 (CUST_A) and CE9 (CUST_B) across the SRv6 core.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — SRv6-TE

### Task 4.1 — Explicit SRv6-TE policy (segment-list of SRv6 SIDs)
**Question:** On G-R1, build an **explicit SRv6-TE policy** to G-R2 that traverses G-R3 (non-shortest / pinned path) using a segment-list of SRv6 SIDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — Traffic steering with color
**Question:** Steer CUST_A traffic into the policy using BGP **color**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Verify SRH in packet (Segments-Left)
**Question:** Confirm the Segment Routing Header is imposed and read **Segments-Left**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — SRv6 Inter-domain (Gold ↔ Garnet)

### Task 5.1 — SRv6 ↔ SR-MPLS boundary at G-R5 ↔ Gar-R7
**Question:** Gold runs SRv6; the Garnet AS runs SR-MPLS. Describe/configure the interworking boundary at G-R5 (Gold) ↔ Gar-R7 (Garnet).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — SRv6-to-MPLS interworking concept
**Question:** Explain the packet-level interworking as a packet crosses G-R5→Gar-R7.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3 — End.B6 binding SID at the boundary
**Question:** Explain the **End.B6** binding SID and its role at the domain boundary.


> *Try this yourself first. Solution available in `solutions/` folder.*

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
