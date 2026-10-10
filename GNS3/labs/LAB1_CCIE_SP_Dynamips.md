# CCIE Service Provider — Lab 1 (Dynamips Edition)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11, GNS3 Local (Dynamips)
**Format:** Full-scale CCIE SP written lab simulation — task statements only, no solutions
**Scope restriction:** This lab is restricted to technologies supported on IOS 15.2(4)M11 / Dynamips. The following CCIE SP v4/v5 topics are **excluded**: Segment Routing (SR-MPLS), SRv6, EVPN, Flex-Algo, and Cisco NSO/model-driven programmability.

**Topology reference:** `00_topology_reference.md` — read this first. All router names, interfaces, loopbacks, link subnets, IS-IS NET-IDs, IPv6 addressing, and customer VPN assignments referenced below are defined there. Do not renumber or reinterpret addressing shown in the reference; where a task does not explicitly restate an IP/mask, use the value from the reference topology.

**AS/IGP summary:**

| AS | IGP | Role summary |
|---|---|---|
| 65100 | IS-IS Level-2 only | R1,R2 = PE; R3,R4 = P; R5,R6 = ASBR; R7,R8 = RR |
| 65200 | OSPF Area 0 | R11,R13 = P; R12 = ASBR; R14,R15,R16 = PE; R19 = RR |
| 65300 | IS-IS Level-2 only | R20,R21 = ASBR/PE; R22,R23 = PE; R30 = RR |

Candidates have read/write access to all 31 routers. No pre-staged configuration exists beyond base IOS, interface addressing, and hostnames unless a task says otherwise. Where a task does not specify a value (timer, metric, process ID, etc.), the candidate must choose a reasonable value and must be internally consistent across all routers touched by that task.

**General exam rules:**
1. Read the entire lab before configuring anything — later tasks depend on earlier ones.
2. Unless stated otherwise, do not use static routes or default routes to achieve reachability.
3. Unless stated otherwise, do not use policy-based routing to achieve a routing requirement that should be solved with routing protocol configuration.
4. Do not disable split-horizon, CDP, or other defaults unless a task requires it.
5. All loopback0 interfaces must remain reachable end-to-end across the entire network (all 3 ASes) at the end of Section 1, and must stay reachable through the rest of the lab.
6. Verification commands are not graded but are expected practice; this document intentionally contains no point values or answer key.

---

## Section 1: Core Network Infrastructure (25%)

### Q1.1 — IS-IS Level-2 in AS 65100 and AS 65300

Enable IS-IS as the IGP inside AS 65100 (R1-R8) and AS 65300 (R20,R21,R22,R23,R30) according to the NET-IDs and areas defined in the topology reference (area `49.0001` for AS65100, area `49.0003` for AS65300).

Requirements:
- Run IS-IS as **Level-2-only** on every router and every core-facing interface in both ASes. Do not form Level-1 adjacencies.
- Advertise all Loopback0 interfaces into IS-IS. Loopback0 interfaces must be advertised as **passive** (no IS-IS hellos sent on loopbacks).
- Enable **wide metrics** (`metric-style wide`) network-wide in both ASes.
- Configure all core point-to-point physical links (R1-R3, R1-R2, R2-R4, R3-R4, R3-R5, R3-R7, R4-R6, R5-R8, R5-R6 in AS65100; R20-R21, R20-R22, R21-R23, R22-R23, R23-R30 in AS65300) for **IS-IS network type point-to-point**, even where the underlying GNS3 link is an Ethernet segment.
- Set the reference bandwidth so that Gi links and Fa links are differentiated: Gi1/0, Gi2/0 interfaces should carry a lower IS-IS metric than Fa0/0, Fa3/0, Fa4/0 interfaces on parallel/alternate paths, reflecting a 1:10 bandwidth differential (Gi ≈ 1000 Mbps, Fa ≈ 100 Mbps).
- Authenticate all IS-IS hello PDUs and all LSPs network-wide using **HMAC-MD5**, key-string `CCIESPL2`. A misconfigured or missing key on any link must not be allowed to form an adjacency.
- Configure **LDP-IGP synchronization** (`mpls ldp sync`) on all core-facing interfaces in both ASes (LDP itself is configured in Q1.3 — IS-IS must be ready to hold routes until LDP comes up once configured).
- Set the **SPF and LSP generation timers** to a fast-converging profile: initial SPF delay 50 ms, SPF hold 200 ms, SPF maximum wait 5000 ms (use `spf-interval` / exponential backoff syntax available in 15.2(4)M11).
- Set the IS-IS **overload-bit** on R7 and R8 (the two RRs in AS65100) to be set automatically on bootup for 180 seconds (`set-overload-bit on-startup 180`), so that during a reload these routers are not used for transit until their LSDB has reconverged.
- R3 and R4 must prefer the direct Gi1/0 link to each other over any 2-hop alternate path for traffic between AS65100 P routers — verify this is achieved purely by the metric scheme in use, not with a policy.

---

### Q1.2 — OSPF Area 0 in AS 65200

Enable OSPF as the sole IGP inside AS 65200 (R11-R16, R19) in a single Area 0, process ID 200, router-id equal to each router's Loopback0 address.

Requirements:
- Advertise all Loopback0 interfaces into OSPF Area 0 as **host routes** (/32) using the loopback's natural interface, and ensure the loopback network type is **not** broadcast (so no DR/BDR election occurs on any loopback).
- Configure every OSPF-enabled core interface (R11-R12, R11-R13, R11-R15, R12-R14, R13-R16, R13-R14, R13-R19, R14-R19) for **network type point-to-point**, eliminating DR/BDR election network-wide.
- Authenticate every OSPF adjacency using **MD5**, key ID 1, key-string `CCIESPOSPF`.
- Enable **BFD** for OSPF neighbor failure detection on all core links, with an interval of 150 ms, min_rx 150 ms, and multiplier 3. OSPF must register with BFD (`ospf bfd` under the interface or process) so that neighbor loss is detected in well under 1 second.
- Tune OSPF SPF throttling to: initial 50 ms, hold 100 ms, max-wait 5000 ms. Tune LSA throttling (`timers throttle lsa`) to: start 0 ms, hold 50 ms, max 5000 ms.
- Ensure R12 (the ASBR facing both AS65100 and AS65300) does **not** originate a Type-3 summary or any default route into Area 0 as a side effect of later inter-AS tasks — Area 0 should only ever carry native AS65200 OSPF routes at the end of this task.
- Set the reference bandwidth so Gi interfaces and Fa interfaces are differentiated identically to the IS-IS scheme in Q1.1 (10x cost differential, Gi cheaper).

---

### Q1.3 — LDP / MPLS across all three ASes

Enable **LDP** (not TDP) as the label distribution protocol on every core-facing and inter-AS-facing interface in all three ASes — i.e., every link that is already running IS-IS (AS65100, AS65300) or OSPF (AS65200), plus the four inter-AS links (R5-R11, R6-R12, R6-R20, R12-R21).

Requirements:
- Use the Loopback0 address as the **LDP router-id** on every LSR (`mpls ldp router-id Loopback0 force`).
- Enable **LDP session protection** (`mpls ldp session protection`) network-wide, with an infinite duration, so that LDP bindings survive a brief link flap without a full session teardown.
- Enable **LDP-IGP synchronization** verification: confirm that IS-IS/OSPF correctly hold down a link's usability until the LDP session over that link reaches operational state, per the sync feature enabled in Q1.1/Q1.2.
- Restrict advertised/accepted LDP label bindings using **label filtering**: each LSR should only advertise LDP labels for its own Loopback0 /32 and should only accept labels for /32 host routes — use an inbound and outbound label prefix-list/ACL (`mpls ldp advertise-labels` / `mpls ldp neighbor labels accept`) so that no labels are exchanged for the PE-CE or customer-facing subnets listed in the topology reference's PE-CE link table.
- Authenticate every LDP TCP session network-wide with **MD5**, password `CCIESPLDP`.
- Disable LDP on any interface that faces a CE router (per the PE-CE link table in the topology reference) — LDP must never attempt to establish a session toward R9, R10, R17, R18, R24, R25, R26, R27, R28, R29, or R32.
- At the end of this task, every Loopback0 /32 in AS65100 and AS65300, and every Loopback0 /32 in AS65200, must have a corresponding non-aggregate LDP label bound to it and visible in `show mpls ldp bindings`.

---

### Q1.4 — BGP: iBGP via Route-Reflectors, eBGP inter-AS, VPNv4, and path manipulation

Build the full BGP overlay across all three ASes.

**iBGP (per-AS, route-reflector based):**
- AS65100: configure R7 and R8 as **redundant Route-Reflectors**. Every other AS65100 router (R1-R6) peers with both R7 and R8 as RR clients. Do not create a full iBGP mesh.
- AS65200: configure R19 as the single Route-Reflector. R11-R16 peer with R19 as RR clients.
- AS65300: configure R30 as the single Route-Reflector. R20-R23 peer with R30 as RR clients.
- All iBGP sessions must use **Loopback0** as the update-source and must use **next-hop-self** appropriately so that iBGP learned routes resolve via the LDP-labeled IGP path, not a recursive lookup through a client's physical interface.
- Enable both the IPv4 unicast and VPNv4 (`address-family vpnv4`) address-families on every iBGP session network-wide (R1-R8, R11-R16/R19, R20-R23/R30) — the VPNv4 AF is required even though no VRFs exist yet (they are created in Section 2), because BGP-LU/Unified MPLS and later L3VPN tasks depend on it already being up.
- Protect all iBGP sessions with MD5 (coordinate the key with Q5.2 — do not configure authentication twice with different keys).

**eBGP (inter-AS):**
- Configure eBGP directly between the four ASBR pairs identified in the topology reference's inter-AS link table: R5↔R11, R6↔R12, R6↔R20, R12↔R21.
- Each eBGP session uses the directly connected interface IP (not loopback multi-hop) as configured in the topology reference.
- Enable **VPNv4** address-family on at least the R6↔R12 (AS65100↔AS65200) and R12↔R21 (AS65200↔AS65300) eBGP sessions — these two are the sessions that will carry **BGP-LU/Unified MPLS** labeled routes end-to-end in Q1.5. (R5↔R11 and R6↔R20 remain IPv4-unicast-only eBGP sessions and are used for Inter-AS Option A/C testing in Section 2.)

**Path manipulation (apply and demonstrate all four techniques somewhere in the eBGP topology):**
- **LOCAL_PREF:** on R12, set LOCAL_PREF to 150 (above the default of 100) for all routes learned from R21 (AS65300), so that AS65200 always prefers the AS65300 path over any equivalent path available via AS65100, for any prefix where both paths exist.
- **AS-PATH prepend:** on R20, prepend AS65300 **twice** (so the AS-PATH shows 65300 65300 65300) to all routes advertised to R12, to deliberately de-prioritize the R6-R20 and R12-R21 redundant paths relative to a primary path you must identify from the topology.
- **MED:** on R6 and R20, set MED on routes advertised toward each other such that AS65100 is signaled as the preferred ingress for traffic destined to AS65300 prefixes (lower MED wins) — the exact MED values are up to the candidate, but must be consistent and intentional, not accidental defaults.
- **BGP communities:** define and apply a well-known or standard community scheme so that routes originated in AS65300 and sent to AS65200 carry a community value of `65300:200`, and configure R19 (the AS65200 RR) to tag any route carrying that community with LOCAL_PREF 120 inside AS65200. Communities must be sent (`send-community both` or `send-community standard`) on every relevant eBGP and iBGP session along the path.

---

### Q1.5 — BGP Labeled Unicast / Unified MPLS end-to-end

Using the VPNv4-enabled eBGP sessions configured in Q1.4 (R6↔R12 and R12↔R21), extend **BGP-LU (RFC 3107, `address-family ipv4 labeled-unicast` or equivalent VPNv4-based unified MPLS technique available in 15.2(4)M11)** so that Loopback0 host routes for PE routers in AS65100 (R1, R2) and AS65300 (R20, R21, R22, R23) are reachable with a **single, continuous labeled path** that never requires IP lookup/recursion at an ASBR.

Requirements:
- R5, R6 (AS65100 ASBRs), R12 (AS65200 ASBR), and R20, R21 (AS65300 ASBRs) must redistribute/advertise their respective domains' PE loopbacks as labeled routes across the inter-AS eBGP sessions, assigning a local label at each ASBR hop (3-hop label stack behavior across the three ASes is acceptable; true end-to-end unified MPLS with a single unchanged label is a bonus but not required given the eBGP-at-every-AS-boundary design).
- Verify with `show mpls forwarding-table` / `show bgp labeled-unicast` (or vendor-equivalent for this IOS train) that an MPLS-switched (not IP-routed) path exists from R1 (AS65100 PE) to R22 (AS65300 PE) loopback, transiting R5/R6→R12→R21/R20 without a label pop/IP-lookup/push cycle at any ASBR.
- This labeled path is a prerequisite for the Inter-AS Option C task in Q2.4 — do not use static labels or MPLS TE to shortcut this requirement.

---

## Section 2: Architecture and Services (25%)

### Q2.1 — L3VPN: VRFs GREEN, YELLOW, BLUE

Build three Layer-3 VPNs end-to-end per the Customer VPN Design table in the topology reference.

**VRF GREEN** (customer AS 65910; CEs R9, R10 in AS65100, R18 in AS65200; plus R17 OSPF PE-CE in AS65200):
- Create VRF `GREEN` on R1 (facing R9 and R10) and on R16 (facing R18) and on R15 (facing R17).
- RD: `65100:910` on AS65100 PEs, `65200:910` on AS65200 PEs (different RDs per AS is intentional — do not force a single RD across AS boundaries).
- Route-targets: export/import `65910:1` on all GREEN VRFs network-wide so that GREEN sites in both ASes see each other's routes via VPNv4.
- PE-CE routing to R9, R10, R18: **eBGP**, customer AS 65910, using the IPs from the PE-CE table (R1↔R9 192.168.1.0/24, R1↔R10 192.168.2.0/24, R16↔R18 192.168.7.0/24).
- PE-CE routing to R17: **OSPF**, a VRF-lite OSPF process on R15 bound to VRF GREEN, redistributing between VRF-OSPF and VPNv4-BGP in both directions, with the correct **domain-ID / down-bit / VPN tag** handling so that R17's OSPF routes do not get looped back into OSPF as an intra-area route (standard MPLS-VPN OSPF PE-CE superbackbone behavior).
- Since R9 and R10 both appear in VRF GREEN **and** R10 also connects to R2 (also GREEN, same AS, see PE-CE table: R2↔R10 192.168.4.0/24) — this creates two active CE routers (R9, R10) each dual/multi-homed into AS65100's GREEN VRF via two different PEs. Configure **AS-override** on R1 and R2 toward their respective GREEN CEs only if/where the customer AS (65910) would otherwise appear twice in a path a CE must accept — identify where this is actually required based on the topology and justify (in your own working notes, not graded) why it is or isn't needed on each PE.

**VRF YELLOW** (customer AS 65024; CEs R24 in AS65300, R26 in AS65200):
- Create VRF `YELLOW` on R20 and R22 (both facing R24 — R24 is dual-homed, see PE-CE table: R20↔R24 192.168.9.0/24 and R22↔R24 192.168.11.0/24) and on R16 (facing R26, 192.168.8.0/24).
- RD: `65300:24` on AS65300 PEs, `65200:24` on R16.
- RT: export/import `65024:1` network-wide for YELLOW.
- PE-CE routing: eBGP, customer AS 65024, on all three PE-CE links.
- This dual-homing of R24 to both R20 and R22 is the scenario that creates the routing-loop risk fixed with **Site of Origin** in Q3.2 — do not pre-emptively configure SoO here; that is graded separately in Section 3.

**VRF BLUE** (customer AS 65025; CEs R32 in AS65100, R25 — which has links into both AS65200 and AS65300):
- Create VRF `BLUE` on R1 (facing R32, 192.168.3.0/24), on R14 (facing R25, 192.168.5.0/24), and on R21 and R23 (both facing R25 — R25 is dual/multi-homed with links to R21 at 192.168.10.0/24 and R23 at 192.168.12.0/24).
- RD: `65100:25` on R1, `65200:25` on R14, `65300:25` on R21 and R23.
- RT: export/import `65025:1` network-wide for BLUE — this is the VPN that spans **all three ASes**, so RT consistency across AS65100/65200/65300 route-reflectors and ASBRs is critical; verify VPNv4 updates for BLUE actually cross both inter-AS boundaries (AS65100↔AS65200 and AS65200↔AS65300) that lie on the path.
- PE-CE routing: eBGP, customer AS 65025, on all BLUE PE-CE links.
- Because R25 is multihomed into two different ASes (65200 and 65300) for the same customer, apply **Site of Origin** on R14, R21, and R23 for the BLUE VRF now (SoO value `65025:2500`, representing "R25's site") to prevent any possibility of a route learned from R25 at one PE being re-advertised back to R25 through a different PE across the inter-AS VPNv4 path.

For all three VRFs, confirm at the end of this task that every CE loopback (R9, R10, R17, R18, R24, R25, R26, R32) is visible in the correct VRF routing table on every PE that services that VPN, and that no VRF leaks a route into another VRF or into the global table.

---

### Q2.2 — L2VPN / VPWS (AToM) for Customer Red

Provision a point-to-point Ethernet-over-MPLS pseudowire (AToM, VPWS) connecting the Red customer's R14 and R16 attachment circuits.

Requirements:
- Attachment circuits: R14 Fa4/0 (toward R28) and R16 f3/0 (toward R27), both carrying **VLAN 100** per the topology reference's L2 links.
- Configure `xconnect` on R14 and R16 using each other's Loopback0 as the peer, with a unique, matching **VC-ID** (candidate's choice, document consistently — e.g., 14016) and encapsulation `mpls`.
- The pseudowire must transport **dot1q-tagged VLAN 100 frames transparently** (`l2transport` sub-interface in the correct encapsulation mode) — the P routers and other PEs must never see or process VLAN 100 as a routed/switched VLAN; it must ride purely inside the MPLS pseudowire label.
- Enable **pseudowire redundancy is NOT required** for this task (that is addressed structurally by VPLS in Q2.3) — this is a single, non-redundant VPWS circuit.
- Since R14 and R16 are in different ASes from each other is false — recheck the topology: R14 and R16 are **both in AS65200**. Confirm the pseudowire therefore stays intra-AS and does not require inter-AS AToM/VPLS signaling; verify this assumption against `00_topology_reference.md` before configuring.
- Verify end-to-end Ethernet connectivity between the CEs on either end of the pseudowire (R27 and R28) at Layer 2 only — no IP addressing is required on the pseudowire's attachment circuits unless needed for your own verification.

---

### Q2.3 — VPLS full-mesh and H-VPLS

Extend the Red customer's Layer-2 domain into a multipoint VPLS instance, and add a hierarchical access ring.

**Full-mesh VPLS:**
- Build a VPLS instance (VPLS-ID / VFI, candidate's choice, e.g. VFI name `RED-VPLS`, VC-ID 100) with PEs **R14 and R16** as the N-PE (network-facing PE) VPLS core members, full-meshed with each other via LDP-signaled pseudowires (same signaling family as Q2.2, but multipoint VFI instead of a p2p xconnect).
- Each N-PE's local attachment circuit for VPLS is the same VLAN-100 sub-interface/AC used in Q2.2 — note that a single AC cannot simultaneously be a VPWS xconnect **and** a VPLS AC; restructure R14's and R16's local attachment circuits so that this task's requirement (VPLS) is what is actually active at the end of the lab, and clearly note in your configuration which task (Q2.2 vs Q2.3) is the final intended state for the R14/R16 ACs (the exam intent is that Q2.2 demonstrates VPWS capability on a redundant/lab-only link, while Q2.3's full-mesh VPLS is the production design — resolve the overlap by using a second VLAN or sub-interface for the VPWS demo if your topology allows, or state the conflict explicitly).

**H-VPLS with R1 as U-PE:**
- Add **R1** as a third VPLS site acting as a **U-PE (user-facing PE / access node)** in a hierarchical VPLS (H-VPLS) design, connecting upstream to the N-PE core (R14 and/or R16) via an **MPLS access pseudowire** (QinQ or VLAN-based H-VPLS, candidate's choice given 15.2(4)M11 feature support) rather than joining the full VPLS mesh directly.
- R1's local attachment circuit is Gi2/0 toward R29 (VLAN 100 per the topology reference).
- The H-VPLS design must guarantee that R1 (U-PE) does **not** participate in BGP auto-discovery or LDP signaling with every other VPLS N-PE — it should only signal toward its designated upstream N-PE(s), consistent with the split-horizon / loop-avoidance rules of hierarchical VPLS (U-PE spoke sites do not need full-mesh split-horizon enforcement the way N-PEs do).
- At the end of this task, R27, R28, and R29 (the three Red customer CEs) must all be in the same flooding/learning Layer-2 domain and must be able to reach each other's MAC addresses across the VPLS core, while P routers (R3, R4, R11, R13) remain unaware of VLAN 100 as an entity.

---

### Q2.4 — Inter-AS Options A, B, and C for Customer Green/Blue

Demonstrate all three Inter-AS VPN interconnect models on three different physical link pairs, per the topology reference's inter-AS link table.

- **Option A (back-to-back VRF)** on the **R5↔R11** link: configure R5 and R11 each with a **per-VPN sub-interface or separate logical interface**, each side treating the other as a directly-connected CE for that VPN (no VPNv4 exchanged across this link; each ASBR re-originates the VRF route as a brand new eBGP-learned VRF route on its own side). Use this link to extend the **Green** VPN (RT `65910:1`) between AS65100 and AS65200 as an Option-A alternative/backup path to the native intra-domain Green PEs already configured in Q2.1 — do not remove the Q2.1 Green configuration; Option A here is an **additional** redundant path specifically being tested at the ASBRs themselves (R5 and R11 become edge members of VRF GREEN for the purpose of this task only).
- **Option B (eBGP VPNv4 redistribution, no label exchange required between ASBRs' IGPs)** on the **R12↔R21** link: enable the eBGP VPNv4 session between R12 and R21 (already brought up for labeled-unicast purposes in Q1.5 — reuse that session's AF for standard VPNv4 Option B here too) to directly exchange VPNv4 NLRI for the **Blue** VPN (RT `65025:1`) between AS65200 and AS65300, with each ASBR re-writing the BGP next-hop to itself (standard Option B behavior; do not retain the original PE's next-hop across the AS boundary).
- **Option C (multihop MP-eBGP between RRs/PEs, with ASBRs only exchanging labeled loopbacks)** on the **R6↔R20** link: using the BGP-LU/unified-MPLS labeled path already validated in Q1.5 for this exact ASBR pair, establish a **multihop MP-eBGP VPNv4 session directly between an AS65100 route-reflector (R7 or R8) and an AS65300 PE/RR (R20, R21, or R30)**, so that VPNv4 NLRI for the **Green** VPN's R32 attachment (AS65100 side) can, in principle, reach AS65300 without any VPNv4 state existing on R6 or R20 themselves — R6 and R20 should carry only labeled IPv4 (loopback) reachability for this session's traffic, never VPNv4 routes. Preserve the original PE's BGP next-hop unchanged end-to-end (true Option C behavior — next-hop-unchanged on the relevant eBGP sessions).

Clearly label, in your running-configs' remarks/descriptions, which ASBR pair and which VPN customer corresponds to which Option (A/B/C) so the examiner (or your future self) can verify independently.

---

### Q2.5 — Hierarchical QoS egress policy on an ASBR

On **R12** (the AS65200↔AS65300 and AS65200↔AS65100-facing ASBR), build a hierarchical QoS policy applied **egress** on the R12↔R21 inter-AS physical interface.

Requirements:
- Parent policy: **shape** the aggregate egress rate on the R12-R21 link to 80% of the interface's configured bandwidth.
- Child (nested) policy under the shaper, with three classes:
  - Match **DSCP EF** or **MPLS EXP 5** traffic (voice-equivalent) → **priority queue** (LLQ) with policed peak rate of 10% of shaped rate.
  - Match **DSCP AF31** or **MPLS EXP 3** (Blue VPN control-plane-equivalent traffic, candidate's choice of marking criteria) → minimum bandwidth guarantee of 30% of shaped rate, with WRED enabled for congestion avoidance.
  - Class-default → remaining bandwidth, fair-queued, WRED enabled.
- Because this is an MPLS core-facing interface carrying VPNv4-labeled and BGP-LU-labeled traffic simultaneously, the classification must work correctly against the **MPLS experimental (EXP) bits** on the top label for transit LSP traffic, not only against the underlying IP DSCP (which the P/ASBR data plane cannot always see once imposed into an MPLS label stack) — ensure your classification approach (`match mpls experimental topmost`, or equivalent qos-group marking at label-imposition PEs feeding into R12) is valid given that R12 is both an ASBR and may be swapping labels, not only imposing them, for a given packet.
- Document the marking policy (DSCP↔EXP mapping) that must exist upstream at the originating PEs for this egress policy's classification to actually match any real traffic; configure that upstream marking at R1, R6, R14, R20, or R21 as appropriate to the VPN/traffic class you choose to demonstrate end-to-end.

---

## Section 3: Access Connectivity (25%)

### Q3.1 — Multicast VPN (mVPN) for Customer Yellow

Deploy **mVPN Profile 0 (RFC 6513/6514 "Rosen GRE/MDT" draft-rosen-style default-MDT, no BGP-based auto-discovery, no MLDP)** for VRF YELLOW across AS65300 and AS65200.

Requirements:
- Enable multicast routing globally and `ip pim ssm` with the default SSM range (232.0.0.0/8) in the **global/core table** of AS65200 and AS65300 (the P/PE core must run native multicast in **PIM-SSM mode** to carry the MDT trees — do not use PIM-SM/Any-Source in the core).
- Configure a **default MDT** for VRF YELLOW on R20, R22 (AS65300 YELLOW PEs) and R16 (AS65200 YELLOW PE), using multicast group `239.232.24.1` (chosen to encode "24" for the YELLOW customer AS 65024) and a unique MDT source of each PE's Loopback0.
- Enable `ip multicast-routing vrf YELLOW` and `ip pim vrf YELLOW` on the customer-facing side so that the YELLOW customer's own multicast traffic (sourced potentially from R24 or R26, or a host behind them) is encapsulated into the default MDT GRE tunnel across the SP core.
- On the **CE side** (within VRF YELLOW, between R24/R26 and their respective PEs), run **PIM-SSM** toward the customer, and configure a **BSR (Bootstrap Router)** on the CE (R24) to handle any legacy Any-Source multicast group learning the customer's hosts might still require, even though the SP core itself is pure SSM — this models a common real-world requirement where a customer's internal multicast still needs ASM/BSR group-to-RP mapping distributed by the CE, independent of the SP's SSM-only MDT core.
- Verify with `show ip mroute vrf YELLOW` and `show ip pim vrf YELLOW mdt` (or equivalent) that the default-MDT tunnel is up between all three YELLOW PEs and that a test multicast stream sourced behind R24 is replicated to R16's VRF YELLOW customer-facing interface.

---

### Q3.2 — Site of Origin: fix the dual-homed Yellow routing loop

Customer Yellow's site at **R24** is dual-homed into AS65300 via two PEs: **R20** (192.168.9.0/24) and **R22** (192.168.11.0/24), as built in Q2.1. Because both R20 and R22 are in the same VRF YELLOW and the same RT community, a route learned by R20 from R24 can be reflected back toward R22 (and vice-versa) via VPNv4/iBGP through the RR (R30), and then re-advertised by R22 back down to R24 over the second eBGP session — creating a **routing loop / suboptimal backdoor path** where R24's own routes come back to it from its second uplink.

Requirements:
- Configure **Site of Origin (SoO)** as an extended community on R20 and R22's VRF YELLOW PE-CE eBGP sessions toward R24, using SoO value `65300:24` (representing "the R24 site").
- The SoO value must be applied so that **any route learned from R24 by R20 is never re-advertised back to R24 by R22**, and vice versa, while still allowing both R20 and R22 to advertise R24's routes normally to the rest of the YELLOW VPN (R16 and any other YELLOW site).
- After applying SoO, verify (conceptually/operationally) that R24 still receives a usable default/aggregate or specific route from **both** R20 and R22 for outbound path redundancy (SoO prevents the loop-back of R24's **own** advertised routes, it must not be used in a way that removes R24's dual-homed redundancy for routes originated elsewhere in the YELLOW VPN).
- State explicitly, as a verification note, which specific prefixes were looping before the fix (i.e., R24's own locally-originated/connected prefixes) and confirm those no longer appear in R24's eBGP table as learned from the "wrong" direction after SoO is applied.

---

### Q3.3 — PE-CE routing and redistribution hygiene

Across all VRF PE-CE sessions built in Q2.1 (GREEN, YELLOW, BLUE) and the native Green-OSPF session to R17:

- Ensure that **no VRF PE ever redistributes connected or static routes into the customer-facing PE-CE protocol** except the PE-CE link's own connected subnet (which should be covered by the protocol's network statement or interface participation, not blanket redistribution).
- Ensure that **no two customer VPNs' routes are ever visible to each other's CE routers** — e.g., R24 (Yellow) must never learn a Green or Blue prefix, and R32 (Blue) must never learn a Yellow or Green prefix, even transiently.
- For the OSPF PE-CE session to R17 (VRF GREEN on R15), confirm the **OSPF domain-ID** is configured consistently as `0.0.0.1` (candidate's choice, but must match if GREEN's OSPF PE-CE presence is ever extended to a second OSPF PE in GREEN) and that routes crossing the VPN backbone and re-entering OSPF at a different PE would correctly set the **DN-bit** so they are not treated as intra-area by R17 (even though only one GREEN OSPF PE exists today, configure this defensively/correctly as if a second one could be added).
- Set a **maximum-prefix** limit of 50 on every PE-CE eBGP session (R1↔R9, R1↔R10, R1↔R32, R2↔R10, R14↔R25, R16↔R18, R16↔R26, R20↔R24, R21↔R25, R22↔R24, R23↔R25) as a safety control against a misbehaving CE flooding the VRF table — this ties into Q5.2's broader BGP max-prefix requirement but must specifically be present on every PE-CE session, not only inter-AS/core sessions.

---

### Q3.4 — Access-side QoS classification (CE-facing)

On **R16** (which has three distinct customer-facing attachment circuits: VRF GREEN to R18, VRF YELLOW to R26, and the VPLS/VPWS AC to R27), apply an **ingress** classification-and-marking policy on each customer-facing interface:

- Traffic ingressing from R18 (Green, g1/0) → mark DSCP AF21.
- Traffic ingressing from R26 (Yellow, g2/0) → mark DSCP AF31.
- Traffic ingressing from R27 (Red L2, f3/0) → mark CoS 3 (802.1p, since this AC is a VLAN-100 L2 attachment, not routed IP) and ensure that marking is preserved/mapped appropriately into the MPLS EXP bits of the pseudowire label imposed for this traffic as it enters the VPLS/VPWS core (so Q2.5's EXP-based classification at R12 has a consistent, traceable marking scheme feeding it, even though R12 itself does not carry Red VPLS traffic — this is about building one consistent, auditable marking scheme across the whole lab, not about R12 specifically acting on Red traffic).

---

## Section 4: High Availability (10%)

### Q4.1 — MPLS Traffic Engineering with explicit path and FRR

Build an MPLS-TE tunnel inside **AS 65100** and protect it with Fast Reroute.

Requirements:
- Enable MPLS-TE extensions for IS-IS (`mpls traffic-eng level-2`, or equivalent) on all AS65100 core routers participating in the TE topology: R1, R2, R3, R4, R5, R6, R7, R8.
- Build a TE tunnel, **Tunnel1**, from **R1 to R6**, with an **explicit, strict path** that forces the LSP via R1→R3→R5→R6 (deliberately avoiding the R1→R2→R4→R6 alternate path), using an IP explicit-path with strict next-hops matching the topology reference's AS65100 link table.
- Set the tunnel's bandwidth reservation to 10 Mbps and configure the tunnel for `autoroute announce` so that IGP-derived traffic from R1 toward R6-reachable destinations prefers the TE tunnel over the native IGP-computed SPF path once the tunnel is up.
- Protect the R3→R5 link (the midpoint link of the explicit path, and the one most likely to be a single point of failure in a 7-router topology) with an **MPLS-TE Fast-Reroute (FRR) backup tunnel**: configure a **link-protection** backup tunnel from R3 that bypasses the R3-R5 link via an alternate path (R3→R4→R6→R5, or R3→R7→R5 if that adjacency and sufficient bandwidth exist per the topology — candidate must determine the valid bypass path from the topology reference) and signal `fast-reroute` on the primary Tunnel1 at R3.
- Verify FRR protection is actually in a protected state (`show mpls traffic-eng fast-reroute database` / `show mpls traffic-eng tunnels backup`) before declaring the task complete — a backup tunnel that is configured but not actually providing protection for the primary's path does not satisfy this requirement.

---

### Q4.2 — BFD for eBGP between ASBRs

Enable **BFD** to accelerate failure detection for the eBGP sessions between ASBR pairs.

Requirements:
- Enable BFD (interval 150 ms, min_rx 150 ms, multiplier 3 — matching the profile already used for OSPF in Q1.2 for consistency) on the physical interfaces for all four inter-AS ASBR links: R5↔R11, R6↔R12, R6↔R20, R12↔R21.
- Register each of these interfaces' eBGP sessions with BFD (`neighbor <ip> fall-over bfd`) so that eBGP peer failure is detected via BFD rather than waiting on the default BGP hold-timer expiration.
- Confirm BFD neighbor state is `Up` (`show bfd neighbors`) on all four links and that intentionally shutting down one direction of a link's BFD-protected interface results in the corresponding eBGP session dropping in under 1 second (sub-second failure detection), rather than the default ~180s BGP hold-time behavior.

---

### Q4.3 — NSF/NSR concepts (written-answer style task, no configuration)

Without performing any configuration, answer the following as inline comments/notes within your lab notebook (not graded for config, but expected to demonstrate understanding — IOS 15.2(4)M11 on Dynamips 7200 **does not support real NSF/SSO** since Dynamips has no dual-RP/dual-supervisor emulation):

- Explain the distinction between **NSF (Nonstop Forwarding)** and **NSR (Nonstop Routing)**, specifically which one depends on graceful-restart signaling to neighbors (NSF) versus which one maintains fully synchronized protocol state on a standby RP without neighbor awareness of the restart (NSR).
- Identify, for each of IS-IS, OSPF, and BGP, which graceful-restart/NSF-helper capability would need to be enabled on a **neighboring** router (not the restarting router itself) for that neighbor to assist a restarting router under real NSF, and state what IOS commands would enable that helper capability if this lab were run on real Cisco hardware with dual RPs (ASR9k/CRS/7600 Sup720, etc.) instead of Dynamips.

---

## Section 5: Security (10%)

### Q5.1 — IGP authentication (verification/consolidation task)

This task consolidates and verifies the authentication already mandated in Section 1 — do not reconfigure from scratch, confirm it is complete and correctly scoped:

- Confirm **IS-IS HMAC-MD5** authentication (key-string `CCIESPL2`, from Q1.1) is active on every IS-IS interface and at the IS-IS process level (LSP/CSNP/PSNP authentication, not only Hello authentication) in both AS65100 and AS65300 — add process-level authentication now if Q1.1 only covered interface-level Hello authentication.
- Confirm **OSPF MD5** authentication (key-string `CCIESPOSPF`, from Q1.2) is active on every OSPF interface in AS65200.
- For both IGPs, confirm that a deliberately misconfigured/mismatched key on a test interface (candidate's own transient test, reverted afterward) correctly prevents adjacency formation, proving authentication is enforced and not merely configured-but-inactive.

---

### Q5.2 — BGP security: MD5, GTSM, max-prefix

- Confirm/apply **MD5 authentication** on **every** BGP session network-wide: all iBGP sessions (RR and clients) in all three ASes, and all eBGP sessions (the four inter-AS ASBR pairs, and every PE-CE session). Use a consistent key per session type (one key for all iBGP in a given AS, a separate key per eBGP session is acceptable and more realistic) — document the key-naming/scheme used.
- Enable **GTSM (Generalized TTL Security Mechanism, `neighbor <ip> ttl-security hops N`)** on all **eBGP** sessions only (not iBGP) — the four inter-AS ASBR sessions and all PE-CE eBGP sessions. Since all these eBGP peers are directly connected single-hop, set the GTSM hop count to **1**, which should reject any eBGP OPEN/update arriving with a TTL that implies it originated more than one hop away (defeating a remote-spoofed BGP TCP session attempt).
- Enable **BGP max-prefix** limits:
  - On every PE-CE eBGP session (already required at the specific value of 50 in Q3.3 — confirm it's present).
  - On every inter-AS eBGP ASBR session (R5↔R11, R6↔R12, R6↔R20, R12↔R21), set max-prefix to **5000** for the IPv4 unicast AF and **2000** for the VPNv4 AF (where VPNv4 is enabled per Q1.4/Q2.4), with a warning-only threshold at 80%.
  - Confirm the max-prefix action on breach is to **tear down the session** (default behavior) rather than merely log, for all of the above.

---

### Q5.3 — Control-Plane Policing (CoPP) and Remotely-Triggered Black Hole (RTBH)

**CoPP**, applied on every ASBR (R5, R6, R12, R20, R21) and every RR (R7, R8, R19, R30):
- Build a `control-plane` policy-map with at minimum three classes:
  - **Critical** (OSPF, IS-IS, BGP, LDP, BFD control traffic to the router itself) → policed to a generous rate (candidate's choice, e.g., 2 Mbps) but never dropped below what's needed for routing stability — effectively a high-priority allow with a sanity-check ceiling.
  - **Management** (SSH/Telnet/SNMP sourced from a defined management subnet — candidate may assume `172.16.0.0/16` as the lab's notional OOB management range) → policed to a moderate rate (e.g., 512 kbps).
  - **Default/undesirable** (everything else destined to the router's own control-plane, e.g. ICMP floods, unexpected protocol traffic) → aggressively policed/rate-limited (e.g., 64 kbps) to protect the RP/CPU from a DoS attempt, without using `drop` outright (some visibility into attack traffic should remain).
- Apply this CoPP policy-map under `control-plane` on each of the five ASBRs and four RRs.

**RTBH**, triggered from a dedicated trigger router (candidate's choice — recommend R30, the AS65300 RR, as the trigger source since it is not on a transit path for any inter-AS traffic):
- Build the classic RTBH design: a static route to `Null0` for a /32 "trigger" prefix (e.g., `192.0.2.1/32`, a documentation/test address not used elsewhere in the lab) on every PE/ASBR that should participate in blackholing (R1, R2, R5, R6 in AS65100; R12, R14, R15, R16 in AS65200; R20, R21, R22, R23 in AS65300), each pointing that /32 to their own local Null0.
- On the trigger router (R30), redistribute a **tagged static route** for a victim/attacker test address (candidate's choice, e.g. `192.168.100.10/32`, chosen from the CE-CE link subnet in the topology reference so it's a real, revertible test) into iBGP with next-hop set to the `192.0.2.1` trigger address and a well-known no-export-safe community so the triggered blackhole route stays within the SP's own iBGP/VPNv4 domain and is not accidentally leaked to a customer or peer AS.
- Verify that once triggered, traffic toward the victim address is dropped (sent to Null0) at the **ingress** PE closest to the attack source, not only at the destination PE — this is the entire point of RTBH (push the drop decision to the network edge).
- Revert the RTBH trigger at the end of the exercise so it does not interfere with any later verification in the lab.

---

### Q5.4 — uRPF and LDP authentication (verification/consolidation)

- Enable **unicast Reverse Path Forwarding (uRPF)** in **strict mode** on every PE-CE customer-facing interface (all interfaces listed in the PE-CE table of the topology reference) to prevent source-address-spoofed traffic from a customer site from being accepted by the SP core. Where a CE is legitimately multihomed (R24 to both R20 and R22; R25 to R14, R21, and R23) and strict-mode uRPF would incorrectly drop valid asymmetric-return traffic, use **loose mode** uRPF instead on just those specific multihomed PE-CE interfaces, and justify (in notes) why strict mode is unsafe there.
- Confirm **LDP MD5 authentication** (key `CCIESPLDP`, from Q1.3) is active on every LDP session network-wide, including the two inter-AS LDP-adjacent sessions used for Q1.5's unified-MPLS labeled path (R6↔R12 and R12↔R21, if LDP itself — not just BGP-LU — is extended across those links in your Q1.5 design; if your Q1.5 design is pure BGP-LU without LDP across the AS boundary, state that explicitly here instead).

---

## Section 6: Operations (5%)

### Q6.1 — 6PE and 6VPE: IPv6 over the IPv4 MPLS core

Using the IPv6 addressing plan in the topology reference (loopbacks `2001:db8:1::X`, `2001:db8:2::XX`, `2001:db8:3::XX`, and the per-link `/64`s), provide IPv6 connectivity across the existing IPv4-only MPLS core without converting the core itself to native dual-stack.

**6PE (global IPv6 reachability for provider-edge loopbacks/transit):**
- Enable IPv6 on Loopback0 of every PE/P/ASBR/RR router in all three ASes using the addresses from the topology reference's IPv6 Loopbacks table.
- Configure **6PE** on the PE routers that need to exchange native IPv6 customer or infrastructure routes end-to-end across the IPv4 MPLS core: at minimum R1, R2 (AS65100 PEs), R14, R15, R16 (AS65200 PEs), and R20, R21, R22, R23 (AS65300 PEs/ASBRs).
- Use `address-family ipv6` BGP sessions (iBGP, reusing the existing RR topology from Q1.4 — R7/R8, R19, R30 must also carry the IPv6 AF as route-reflectors even though they are not 6PE edge routers themselves) carrying IPv6 NLRI with an **IPv4-mapped next-hop** (RFC 4798 behavior — `::FFFF:<ipv4>` next-hop encoding) so that the existing IPv4 LDP-labeled LSPs are reused to forward IPv6 traffic without any native IPv6 label distribution in the core.
- Do **not** enable IPv6 CEF/native IPv6 forwarding on any P-only router (R3, R4, R11, R13) — these must remain pure IPv4-MPLS in the data plane for this task; only the edge (PE/ASBR/RR control-plane) needs IPv6 BGP awareness.

**6VPE (IPv6 VPN, applied to one existing VRF):**
- Extend **VRF GREEN** (from Q2.1) to carry IPv6 customer prefixes as a 6VPE deployment: enable `address-family ipv6 vrf GREEN` on R1, R15, and R16, assign IPv6 addressing on the GREEN PE-CE links (candidate's choice of a `/64` per link, consistent with the `2001:db8:0:XY::` inter-link numbering style already used elsewhere in the topology reference, e.g., invent `2001:db8:9:X::` for new PE-CE IPv6 subnets not already defined there since the reference's IPv6 table only covers core/inter-AS links), and run **eBGP IPv6** to R9, R10, and R18 for the GREEN customer dual-stack extension (R17's OSPF GREEN session may remain IPv4-only; 6VPE over OSPFv3 PE-CE is not required for this task).
- Confirm the 6VPE IPv6 VPN routes are carried as **VPNv6 (`address-family vpnv6`)** NLRI across the existing MPLS/LDP IPv4 core and iBGP/RR topology — no new core-facing protocol or label distribution should be required beyond what 6PE already established, since 6VPE reuses the same IPv4-MPLS transport with a VPNv6 (not plain IPv6) BGP NLRI and label.
- Verify dual-stack reachability: every GREEN CE (R9, R10, R18) must be able to ping the other GREEN CEs' IPv6 addresses, in addition to the existing IPv4 reachability already validated in Q2.1.

---

## End of Lab 1

Candidates should expect that a working, fully-verified Lab 1 configuration is a prerequisite baseline for Lab 2 (which will build advanced/edge-case variations on this same topology). Save all final running-configs before ending the session.
