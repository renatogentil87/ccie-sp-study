# WB07 VPLS — Solutions

**Reference:** `00_topology_reference.md` — Red VPLS customer, VLAN 100, CEs R27 (R16), R28 (R14), R29 (R1).
**Prerequisite:** WB00–WB03 (IP, IGP, LDP), WB04 (BGP) if using BGP autodiscovery.
**Platform:** Cisco 7200, IOS 15.2(4)M11 — IOS classic syntax.

---

## Scope Decision

VPLS is a multipoint L2VPN: all member PEs join one emulated bridge (one broadcast domain) for VLAN 100. The Red customer spans R1 (AS65100), R14 and R16 (AS65200).

**VPLS is configured within AS 65200 only (R14 ↔ R16).** Reasons:
- VPLS uses a full mesh of directed-LDP PWs between member PEs' loopbacks, requiring all members in one IGP/LDP domain.
- Cross-AS VPLS (adding R1 in AS65100) is not natively supported as a flat mesh — it needs inter-AS stitching / H-VPLS spoke across the border, which depends on WB09 inter-AS reachability.

R14 (toward R28) and R16 (toward R27) form the VPLS core. R1/R29 is added later as an H-VPLS spoke (Section 4) once cross-AS reachability exists.

---

## Section 1: VFI "RED" on R14 and R16

Two member-PE options are shown. VPN-ID 100 ties the VFI to the VLAN-100 service and must match on all members.

### Option A — Manual (static) neighbors

Explicitly list each other member PE's Loopback0. Simple and deterministic for a 2-PE mesh.

```
! R14
l2 vfi RED manual
 vpn id 100
 neighbor 150.2.16.16 encapsulation mpls
!
```

```
! R16
l2 vfi RED manual
 vpn id 100
 neighbor 150.2.14.14 encapsulation mpls
!
```

### Option B — BGP autodiscovery (scales; members found via BGP)

Members are discovered through the L2VPN AF in BGP (requires WB04 BGP up and the `l2vpn` AF activated to R19 RR). Preferred when the mesh grows beyond 2 PEs.

```
! R14 and R16 (shown for R14; mirror on R16 with its own RD/RT)
l2 vfi RED autodiscovery
 vpn id 100
!
router bgp 65200
 address-family l2vpn vpls
  neighbor 150.2.19.19 activate
  neighbor 150.2.19.19 send-community extended
 exit-address-family
!
```

> With autodiscovery, the VFI RD/RT come from the BGP L2VPN AF; PEs sharing the same RT auto-mesh their PWs. On older 7200/IOS classic, autodiscovery VPLS support can be limited — if it does not form, fall back to **Option A manual neighbors**.

---

## Section 2: Bind the AC (VLAN 100) to the VFI

The attachment circuit (customer-facing interface) is tied to the VFI so VLAN-100 frames enter the emulated bridge. On IOS classic 7200 the common binding is through an SVI + `l2 vfi ... manual` with `xconnect vfi`, or via a service-instance/bridge-domain on platforms that support it. For 7200 use the VFI-on-VLAN-interface form:

```
! R14 — AC is f4/0 toward R28, VLAN 100
interface FastEthernet4/0
 no ip address
 no shutdown
!
interface FastEthernet4/0.100
 encapsulation dot1Q 100
!
! Bind VLAN 100 bridge to the VFI
interface Vlan100
 no ip address
 xconnect vfi RED
!
```

```
! R16 — AC is f3/0 toward R27, VLAN 100
interface FastEthernet3/0
 no ip address
 no shutdown
!
interface FastEthernet3/0.100
 encapsulation dot1Q 100
!
interface Vlan100
 no ip address
 xconnect vfi RED
!
```

> On platforms/images that expose `bridge-domain`, the equivalent is: define `bridge-domain 100`, place the AC service-instance and the `member vfi RED` into bridge-domain 100 (see Section 3). The 7200/Dynamips image may only support the `l2 vfi ... manual` + `xconnect vfi` style — use whichever the image accepts; both achieve the same VLAN-100 multipoint bridge.

---

## Section 3: Bridge-domain form (if supported by the image)

Newer syntax (shown for completeness; verify the 7200 image accepts `bridge-domain`):

```
! R14
l2vpn vfi context RED
 vpn id 100
 member 150.2.16.16 encapsulation mpls
!
bridge-domain 100
 member FastEthernet4/0 service-instance 100
 member vfi RED
!
interface FastEthernet4/0
 service instance 100 ethernet
  encapsulation dot1q 100
!
```

```
! R16
l2vpn vfi context RED
 vpn id 100
 member 150.2.14.14 encapsulation mpls
!
bridge-domain 100
 member FastEthernet3/0 service-instance 100
 member vfi RED
!
interface FastEthernet3/0
 service instance 100 ethernet
  encapsulation dot1q 100
!
```

> **Split-horizon:** within a VPLS VFI, PWs between PE members are automatically **split-horizon** — a frame received on one PW is never forwarded out another PW (prevents loops in the full mesh, so no STP needed in the core). This is implicit for a flat (non-hierarchical) VFI mesh. It is why a full mesh of PWs between all N PEs is required: there is no PW-to-PW forwarding. Spoke PWs in H-VPLS (Section 4) are the exception — the N-PE disables split-horizon toward spokes so it can relay between the spoke and the core mesh.

---

## Section 4: H-VPLS — R1 as U-PE spoke to R14 N-PE

To include R29 (behind R1, AS65100) in the Red service without a full cross-AS mesh, make **R1 a U-PE (spoke)** with a single spoke PW to **R14 as N-PE (hub)**. R14 relays spoke traffic into the AS65200 core VFI mesh.

```
! R1 (U-PE / spoke) — single spoke PW up to R14
interface GigabitEthernet2/0
 no ip address
 no shutdown
!
interface GigabitEthernet2/0.100
 encapsulation dot1Q 100
 xconnect 150.2.14.14 100 encapsulation mpls
!
```

```
! R14 (N-PE / hub) — spoke PW from R1 joins the RED VFI
l2 vfi RED manual
 vpn id 100
 neighbor 150.2.16.16 encapsulation mpls      ! core mesh (to R16)
 neighbor 150.1.1.1 100 encapsulation mpls no-split-horizon   ! spoke from R1 (U-PE)
!
```

> The spoke PW to R1 is added with **`no-split-horizon`** so the N-PE (R14) *can* forward between the spoke and the core PWs — a spoke is allowed to break the split-horizon rule because it is not part of the full mesh. Core PWs (R14↔R16) keep split-horizon. R1 (U-PE) runs a single point-to-point PW — simpler, no VFI on the spoke.

> **Cross-AS blocker (same as WB06):** the spoke PW R1→R14 crosses AS65100↔AS65200. It needs loopback 150.2.14.14 reachable from R1 and a targeted-LDP session across the border — provided by **WB09 inter-AS** (Option B/C or BGP-LU / WB14 unified MPLS). Until then the spoke PW stays down; the intra-AS R14↔R16 VPLS mesh works independently.

---

## Section 5: CE reference (R27, R28, R29 on VLAN 100)

All three CEs share one broadcast domain / IP subnet once the (H-)VPLS is up:

```
! R28 (behind R14)
interface FastEthernet0/0.100
 encapsulation dot1Q 100
 ip address 172.16.100.28 255.255.255.0
!
! R27 (behind R16)
interface FastEthernet3/0.100
 encapsulation dot1Q 100
 ip address 172.16.100.27 255.255.255.0
!
! R29 (behind R1) — reachable only after H-VPLS spoke (Section 4) + WB09
interface GigabitEthernet2/0.100
 encapsulation dot1Q 100
 ip address 172.16.100.29 255.255.255.0
!
```

> 172.16.100.0/24 is a lab-chosen subnet for the Red VLAN-100 service. R27 and R28 (both AS65200) should reach each other immediately; R29 joins once the cross-AS spoke is functional.

---

## Section 6: Verification Commands

```
! VFI / VPLS state
show l2vpn vfi                         ! (newer syntax)
show vfi                               ! (classic l2 vfi syntax)
show vfi name RED
show mpls l2transport vc               ! each VFI PW appears as a VC

! Bridge-domain (if used)
show bridge-domain
show bridge-domain 100

! MAC learning in the emulated bridge
show mac-address-table
show l2vpn vfi name RED detail

! Targeted LDP sessions carrying the mesh
show mpls ldp neighbor
show mpls l2transport binding

! H-VPLS spoke
show mpls l2transport vc detail        ! confirm spoke PW + no-split-horizon flag
```

### Expected results

```
show vfi name RED
  VFI name: RED, state: up
  VPN ID: 100
  Local attachment circuit: Vlan100
  Neighbors connected via pseudowires:
   Peer Address    VC ID   S
   150.2.16.16     100     Y     <- core mesh, split-horizon ON
   150.1.1.1       100     N     <- H-VPLS spoke, split-horizon OFF (after WB09)
```

> A healthy intra-AS VPLS shows the VFI **up** on R14 and R16 with the peer PW **up** and split-horizon **Y** between them. R27↔R28 ping and learn each other's MACs. The R1 spoke row only appears/comes up after WB09 provides cross-AS loopback reachability — expected to be down until then.

---

## Section 7: Cross-AS VPLS Limitations on 7200

- **No native inter-AS flat VPLS mesh.** VPLS requires a full PW mesh in one LDP domain; AS65100↔AS65200 crosses domains. Adding R1 directly as a VFI member of the AS65200 RED mesh will not form.
- **Workarounds:**
  1. **H-VPLS spoke (Section 4)** — R1 as U-PE with one spoke PW to an N-PE (R14). Needs inter-AS loopback reachability (WB09 Option B/C) or BGP-LU unified MPLS (WB14).
  2. **Inter-AS Option E / border VFI stitching** — stitch the two domains' PWs at the ASBR; limited/awkward on IOS classic 7200.
  3. **MS-PW** for point-to-point segments (that is AToM/WB06, not multipoint VPLS).
- **Dynamips/7200 image caveats:** `l2vpn vfi context` / `bridge-domain` / `service instance` syntax may not be fully supported — prefer the classic `l2 vfi RED manual` + `interface Vlan100 / xconnect vfi RED` form. Autodiscovery (BGP L2VPN AF) VPLS may also be unavailable; use manual neighbors.
- **Bottom line:** keep Red VPLS inside AS65200 (R14↔R16) for a working multipoint service; bring R1/R29 in via H-VPLS spoke after WB09. Do not expect a single cross-AS VPLS domain to come up on this platform without inter-AS plumbing.
