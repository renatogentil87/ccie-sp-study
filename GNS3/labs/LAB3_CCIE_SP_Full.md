# LAB 3 — CCIE SP v5.1 Full Workbook (EVE-NG / IOS-XRv 9000)

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+)
**Scope:** Full CCIE SP v5.1 blueprint, including technologies **not supported on Dynamips/IOS 15.2** (Segment Routing, SRv6, Flex-Algo, EVPN, streaming telemetry, NSO).
**Companion lab:** This reuses the same logical topology, AS numbers, and router roles as the GNS3 Dynamips labs (`GNS3/workbooks/00_topology_reference.md`), re-platformed onto IOS-XRv 9000 so SR/SRv6/EVPN/telemetry features that Dynamips cannot run are testable.
**Format:** Tasks only — no solutions. Work each task yourself; verify with `show` commands before moving on.

---

## 0. Topology Reference (map GNS3 → EVE-NG/IOS-XR)

Same 3 autonomous systems, same roles, same R-numbers as the GNS3 base topology — only the platform and loopback-based hostnames below (Emerald/Garnet/Gold) differ for XR readability. If you prefer, keep the plain `R#` hostnames from GNS3; the AS numbers and roles are identical.

| AS | Name | IGP | Transport | Routers (role) |
|----|------|-----|-----------|-----------------|
| **65100** | Emerald | IS-IS L2 | SR-MPLS + SRv6 (migrated from LDP) | R1(PE)/E-R1, R2(PE)/E-R2, R3(P)/E-R3, R4(P)/E-R4, R5(ASBR)/E-R5, R6(ASBR)/E-R6, R7(RR)/E-R7, R8(RR+PCE)/E-R8 |
| **65200** | Garnet | OSPF Area 0 (base) / IS-IS where noted | SR-MPLS | R11(P)/Gar-R3, R12(ASBR)/Gar-R7, R13(P)/Gar-R4, R14(PE)/Gar-R1, R15(PE)/Gar-R2, R16(PE)/Gar-R5, R19(RR+PCE)/Gar-R6 |
| **65300** | Gold | IS-IS L2 | SRv6 native | R20(ASBR/PE)/G-R4, R21(ASBR/PE)/G-R5, R22(PE)/G-R1, R23(PE)/G-R2, R30(RR+PCE)/G-R3 |

**Loopback0 convention (unchanged from GNS3):** `150.<AS-last-digit>.<R#>.<R#>/32` — e.g., R1 = 150.1.1.1/32, R20 = 150.3.20.20/32.
**IS-IS NET-IDs, core link subnets, inter-AS links, and PE-CE links:** identical to `GNS3/workbooks/00_topology_reference.md` — reuse that addressing plan verbatim; only the CLI changes to IOS-XR syntax.
**New IOS-XR-only addressing you will add in this lab:**

| Purpose | Scheme |
|---|---|
| SRGB (SR-MPLS, all ASes) | 16000–23999 |
| SRLB (per node) | 15000–15999 |
| Prefix-SID index | = last octet of Loopback0 (R1→1, R20→20, etc.) |
| Anycast SID (redundant PE pair R14/R15, Garnet) | index 114, label 16114 |
| SRv6 locator block (Gold, AS 65300) | `fc00:3:<R#>::/48`, F3216 uSID format |
| SRv6 locator block (Emerald, AS 65100, SRv6 overlay) | `fc00:1:<R#>::/48` |
| Flex-Algo 128 (low-latency) | affinity-colored links, separate SPF |
| Color communities (SR-TE / SR-ODN) | 100 = low-latency, 200 = low-cost/TE-metric |
| EVI numbers (EVPN-VPWS) | EVI 1001 (R1↔R16 Red circuit), EVI 1002 (Gar-R1↔Gar-R2 secondary) |

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS with Flex-CLI configuration groups, wide metrics, single-topology dual-stack, LDP sync, overload-bit, keychain auth

On the **Emerald AS 65100** core (R1–R8), build the Level-2-only IS-IS process using **configuration groups** (`isis-interface-group` / interface config-group under `router isis`) so common per-interface stanzas (metric, address-families, authentication) are defined once and applied to all core-facing interfaces instead of being repeated.

Requirements:
1. Single IS-IS instance, **Level 2 only** on every interface (no L1, no L1/L2).
2. **Wide metrics** (`metric-style wide`) under both the IPv4 and IPv6 unicast address-families — mandatory prerequisite for Task 1.3's SR sub-TLVs.
3. **Single-topology** IS-IS carrying both IPv4 and IPv6 unicast (do **not** use multi-topology / MT).
4. Build an **interface configuration group** containing: wide-metric address-families, LDP sync enablement, and HMAC-MD5 authentication reference — apply the group to all core-facing GigabitEthernet interfaces on R3, R4, R5, R6 (the "P" tier).
5. Enable **LDP IGP-sync** (`mpls ldp sync`) on all core links in AS 65100 so IS-IS holds a link's metric at max until LDP session + label exchange with that neighbor completes.
6. Set the **overload-bit** on R7 and R8 (the RRs) with `set-overload-bit on-startup wait-for-bgp` so they do not become a mid-path forwarding node for transit traffic until their BGP sessions converge after a reload.
7. Configure a **keychain** named `ISIS-KEY` under `key chain ISIS-KEY` with HMAC-MD5 algorithm, and reference it from the IS-IS instance for both **hello authentication** and **LSP authentication** on every Level-2 interface (do not configure this in Task 1.1 beyond referencing — the keychain lifetime/key details are built out fully in Task 5.2).

Verify: `show isis neighbors`, `show isis database verbose`, `show mpls ldp neighbor`, `show isis adjacency detail` (confirm `authentication` and LDP-sync hold-down states clear once LDP comes up).

### Task 1.2 — BGP Unified MPLS: BGP-LU, inline RRs on P routers, allocate-label all

Extend MPLS label-switched reachability for PE loopbacks **end-to-end across all three ASes** using the Unified MPLS / Seamless MPLS architecture (BGP-LU, RFC 3107).

1. On every ASBR (R5, R6, R12, R20, R21) and every RR (R7, R8, R19, R30), enable the **`address-family ipv4 labeled-unicast`** (or `address-family ipv4 unicast` with `allocate-label all` depending on XR version — use `labeled-unicast` AF explicitly) BGP session.
2. Configure **`allocate-label all`** under the IPv4 unicast AF on R3 and R4 (the inline P-tier routers in Emerald) so that **every** IGP-learned prefix — not just loopbacks — receives an MPLS label and is redistributed into BGP-LU. This makes R3/R4 function as **inline Route Reflectors** for BGP-LU, removing the need for a dedicated RR in the label-switched path.
3. Redistribute each AS's IS-IS/OSPF loopbacks into BGP-LU at the ASBR boundary; do not redistribute BGP-LU prefixes back into the IGP.
4. Build the Unified MPLS core such that a PE in Garnet (R14) can resolve a BGP-LU label-switched path all the way to a PE in Gold (R22) across both ASBR hops (R12→R21 is **not** directly connected — verify the actual path transits R6/R20 or the direct R12↔R21 inter-AS link per the topology reference).
5. On R3 and R4, confirm `show mpls label range` and `show bgp labeled-unicast ipv4 unicast` reflect the inline-RR label allocations.

Verify: `show bgp ipv4 labeled-unicast summary`, `show mpls forwarding | include <PE-loopback>`, end-to-end `ping mpls ipv4 <remote-PE-loopback>/32` across AS boundaries.

### Task 1.3 — Segment Routing on IS-IS: prefix-SID, SRGB/SRLB, anycast SID, TI-LFA, microloop avoidance

Migrate the Emerald core (R1–R8) from LDP to **SR-MPLS** under IS-IS while Garnet (R11–R19) is **built SR-native** from the start.

1. Enable `segment-routing mpls` under the IS-IS IPv4 unicast address-family on all nodes in both ASes.
2. Set a common **SRGB 16000–23999** and per-node **SRLB 15000–15999** on every SR-enabled router.
3. Assign **prefix-SID index = last octet of Loopback0** to each node (R3 → index 3 → label 16003; Gar-R1/R14 → index 14 → label 16014, etc.).
4. Configure an **anycast SID** across the redundant PE pair R14 and R15 (Garnet) — both advertise prefix-SID **index 114** (label 16114) on a shared anycast loopback (e.g., 150.2.114.114/32) so either PE can be targeted as a single ECMP-capable endpoint for inter-AS VPN traffic.
5. Enable **TI-LFA** (`fast-reroute per-prefix ti-lfa` under IS-IS IPv4 unicast) on every core-facing interface in both ASes for sub-50ms link/node protection without RSVP.
6. Enable **microloop avoidance** (`microloop avoidance protected` or `segment-routing microloop-avoidance` under IS-IS, depending on XR train) on all SR nodes so transient micro-loops during SPF reconvergence after a planned/unplanned topology change are suppressed via a temporary SR detour.
7. Confirm LDP is fully removed from Emerald only after SR-MPLS reachability is proven (ships-in-the-night first, then `no mpls ldp` per interface).

Verify: `show segment-routing mpls prefix-sid-map`, `show isis segment-routing`, `show mpls traffic-eng topology` (TI-LFA repair paths), `show isis fast-reroute ti-lfa`.

### Task 1.4 — SR-TE: dynamic policies, TE metric, binding-SID, autoroute steering, BGP color/autosteering, explicit segment-lists, affinity

On **Gar-R1 (R14)**, build four distinct SR-TE policies toward **Gar-R2 (R15)**:

1. **Dynamic policy**, color **200**, optimized on the **TE metric** (`metric-type te`), recomputed automatically on topology/metric change.
2. **Explicit policy**, color **201**, with a manually specified **segment-list** of prefix-SIDs forcing the path Gar-R1 → Gar-R4(R13) → Gar-R5(R16) → Gar-R2, bypassing the direct link.
3. Configure a **binding-SID** on the explicit policy (Task 2) so it can be referenced as a single label by upstream traffic-engineering or by a parent SR-TE policy (hierarchical SR-TE).
4. Enable **autoroute steering** on the dynamic policy (Task 1) so IGP-shortest-path traffic toward Gar-R2's loopback is automatically forwarded into the policy without requiring explicit route steering.
5. Advertise a **BGP color extended community (color 200)** on VPNv4 routes from Gar-R2 so Gar-R1 **autosteers** matching VPN traffic into the color-200 SR-TE policy purely via BGP (no static `autoroute` or PBR needed) — this is BGP color-based automatic steering.
6. Define **affinity constraints**: tag the Gar-R3↔Gar-R4 link with affinity-color `RED`, then build a 5th policy that explicitly **excludes** affinity `RED` (`exclude-any RED`) and a 6th that **requires** it (`include-all RED`); verify both compute different paths.

Verify: `show segment-routing traffic-eng policy color 200`, `show segment-routing traffic-eng policy name <name> detail` (binding-SID, affinity constraints), `show bgp vpnv4 unicast <prefix> detail` (color community + resolved tunnel).

### Task 1.5 — SRv6: uSID/C-SID F3216 locator plan, PSP/USD, locator advertisement with tags, SRv6 L3VPN transport

Build the **Gold AS 65300** core (R20–R23, R30) natively on **SRv6** using the **uSID (micro-SID) / C-SID** compressed format.

1. Define an **F3216 locator plan**: 32-bit locator block + 16-bit micro-SID function per hop, carved from `fc00:3:<R#>::/48` on each Gold router (e.g., R30 → `fc00:3:30::/48`).
2. Configure the **PSP (Penultimate Segment Pop)** and **USD (Ultimate Segment Decapsulation)** behaviors on the relevant End/End.X SIDs so the penultimate hop pops the active segment and the final hop decapsulates — reducing SRH processing on transit nodes.
3. Advertise each locator under IS-IS SRv6 with a **tag** (e.g., tag 3000 on all Gold locators) so locator-derived routes can be filtered/matched later in route-policy for inter-domain leaking control.
4. Build **SRv6-based L3VPN transport**: configure VRF(s) on R22 and R23 with `encapsulation-type srv6`, per-VRF SID allocation (`alloc mode per-vrf`), and confirm VPNv4 routes between R22/R23 carry an **End.DT4 SRv6 SID** instead of an MPLS label.
5. Confirm the data plane: packet captures or `show cef vrf <name> <prefix>` show an IPv6 SRv6 header, not an MPLS label stack.

Verify: `show segment-routing srv6 sid`, `show segment-routing srv6 locator detail`, `show isis database verbose | include SRv6|Tag`, `show bgp vpnv4 unicast vrf <name> <prefix> detail`.

### Task 1.6 — SR-ODN: color-triggered on-demand policies, PCE-computed latency paths

1. On **Gar-R6 (R19)**, enable **SR-PCE** (`pce address ipv4 <loopback>` + `segment-routing traffic-eng pcc`), and configure **Gar-R1 (R14)** as a PCC delegating path computation to it.
2. Configure **On-Demand Next-hop (ODN)** on Gar-R1: when a VPNv4 route arrives carrying **color 100**, an SR-TE policy toward that route's BGP next-hop is **automatically instantiated** (no manual policy definition) using an ODN template that references a **dynamic path optimized on the latency (min-delay) metric** and requests computation from the Gar-R6 PCE.
3. Tag outbound VPNv4 routes on **Gar-R2 (R15)** for a specific customer with **color-100** extended community so inbound traffic at Gar-R1 toward that customer triggers the ODN policy end-to-end.
4. Confirm the PCE actually computed the path (not purely headend SPF) by checking the policy's delegation/PCEP state.

Verify: `show pce peer`, `show pce lsp`, `show segment-routing traffic-eng policy color 100 detail` (confirm `PCC info:` and delegation state), `show bgp vpnv4 unicast <prefix> detail` (color 100 + auto-created tunnel).

### Task 1.7 — Flex-Algo: algorithm definition, affinity constraints, per-algo prefix-SID, SR policy constrained to flex-algo

1. On **Gar-R6 (R19)**, define **Flex-Algo 128** under `segment-routing flex-algo 128`: metric-type **delay (min-latency)**, with constraint **`exclude-any affinity RED`** (reuse the RED affinity tag from Task 1.4) so Algo 128's SPF never traverses RED-tagged links.
3. On every Garnet node, advertise a **per-algorithm prefix-SID** for Loopback0 under Algo 128 — distinct from the default Algo-0 SID (e.g., Gar-R1 default index 14/label 16014, Algo-128 index 1014/label a separate SRGB-relative label).
4. Enable Flex-Algo participation (`flex-algo 128` under IS-IS SR config) on all Garnet nodes so Algo 128 computes its own constrained SPT in parallel with the default topology.
5. Build an SR-TE policy on Gar-R1 that is explicitly **constrained to Flex-Algo 128** (`algorithm 128` under the dynamic path's constraints) and confirm it resolves to the Algo-128 SID stack, not the default-topology path.

Verify: `show isis segment-routing flex-algo`, `show segment-routing flex-algo`, `show segment-routing traffic-eng policy name <name> detail` (confirm algo 128 constraint and resulting SID list differs from the Algo-0 shortest path).

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — EVPN-VPWS: T-LDP VPWS + EVPN-VPWS over SRv6

1. Build a classic **T-LDP-signaled VPWS** pseudowire between **R16 (Garnet PE)** and **R1 (Emerald PE)** for the "Red" cross-AS circuit, EVI/PW-ID **1001**, using `l2vpn xconnect group RED p2p RED-PW` with `mpls static` or `mpls ldp` neighbor targeted session across the inter-AS boundary.
2. Separately, build an **EVPN-VPWS** service (RFC 8214) between **Gar-R1 (R14)** and **Gar-R2 (R15)** carried over the **SRv6** transport from Task 1.5, EVI **1002**: configure `l2vpn evpn evi 1002`, EVPN-VPWS service with local/remote AC-ID and EVI, and `encapsulation-type srv6` on the EVPN control-plane session so MPLS labels are replaced by SRv6 SIDs end-to-end.
3. Contrast the two: T-LDP VPWS requires a targeted LDP session per PW; EVPN-VPWS uses BGP EVPN (RT/RD + EVI) as its single control plane and can ride any transport (MPLS, SR-MPLS, SRv6) without per-service signaling sessions.
4. Attach a customer-facing Ethernet sub-interface on each PE's access side to each respective PW.

Verify: `show l2vpn xconnect group RED`, `show evpn evi 1002 detail`, `show l2vpn xconnect summary`, confirm the EVPN-VPWS pseudowire state is `UP` with the correct remote AC-ID.

### Task 2.2 — L3VPN: Intra-AS, Inter-AS Option B, PE-CE BGP + OSPF, as-override

1. Build **intra-AS L3VPN** within Garnet for the **Green** customer (per topology reference: R17 via OSPF PE-CE on R15, R18 via eBGP AS 65910 on R16).
2. Build **Inter-AS Option B** for the **Blue** customer spanning Emerald↔Garnet: VPNv4 **eBGP directly between ASBRs R6 and R12** (no RR relay — Option B re-originates the VPNv4 route at each ASBR with next-hop-self and a fresh label).
3. On the Garnet side PE-CE link using OSPF (R15↔R17), ensure the VPN backbone is transparent to OSPF via the standard **sham-link or down-bit/domain-tag** handling so Area 0 continuity isn't falsely inferred across the MPLS core.
4. On a PE-CE eBGP link where the customer's CE sits in the **same ASN** on both ends of a hub-and-spoke VPN (e.g., both R18 and another spoke CE use AS 65910), configure **`as-override`** on the PE so the customer ASN is rewritten in the AS-path and the route is not rejected by AS_PATH loop detection at the spoke CE.
5. Verify route re-origination at both ASBRs for Option B (RD/RT changes expected, AS-path should show the Option B hop).

Verify: `show bgp vpnv4 unicast vrf <name>`, `show bgp vpnv4 unicast neighbor <ASBR> advertised-routes`, `show route vrf <name> <customer-prefix>` (confirm label swap at ASBR for Option B), `show cef vrf <name>`.

### Task 2.3 — Multicast VPN: NG-MVPN, PIM-SSM core, BGP auto-discovery, mdt default/data

1. Enable **PIM-SSM** (`router pim / address-family ipv4 / rpf topology route-policy`, SSM range 232.0.0.0/8) in the Garnet core — no RP needed.
2. Configure **NG-MVPN** on the Garnet PEs (R14, R15, R16): `vrf <name> / address-family ipv4 mdt`, enable **BGP-based auto-discovery** (`mdt auto-discovery pim`) so PE-to-PE MDT membership is learned via BGP instead of a static or PIM-only mechanism.
3. Configure the **MDT default group** per VRF for low-bandwidth/control multicast traffic, and the **MDT data group range** for high-bandwidth source switchover (`mdt data 10 group-range <range> threshold <kbps>`).
4. Verify source-tree switchover: a customer multicast source behind R18 crossing a configured data-threshold triggers the PEs to switch from the default MDT to a dedicated data MDT.

Verify: `show pim vrf <name> group-map`, `show mvpn vrf <name> pe`, `show mvpn vrf <name> database`, `show bgp ipv4 mdt`.

### Task 2.4 — H-QoS: hierarchical MQC with shaping, policing, WRED, DSCP remarking

On **R14 (Gar-R1)**'s customer-facing interface for the Green customer:

1. Build a **parent policy-map** applying a **shaper** (e.g., 100 Mbps) at the interface/port level.
2. Nest a **child policy-map** under the parent with per-class **policing** (CIR/PIR) for at least 3 traffic classes (voice, business-critical, best-effort), matched on DSCP.
3. Apply **WRED** on the best-effort/bulk class queue to manage congestion before hard-drop, using DSCP-based WRED profiles (not just a single min/max threshold).
4. **Remark DSCP** on ingress for one class (e.g., remark untrusted CE markings for "business-critical" traffic to AF31) before the egress hierarchical policy is applied.
5. Apply the two-level (parent/child) `service-policy output` hierarchy on the physical or bundle interface.

Verify: `show policy-map interface <intf>`, `show policy-map interface <intf> output class <name>` (confirm WRED drop counters, shaped rate, policed rate), `show qos interface`.

---

## Section 3 — Access Connectivity (25%)

### Task 3.1 — Site of Origin (SoO) for dual-homed CE

The **Blue** customer's CE (R25 in the topology reference) is dual-homed into two PEs (per the base topology, R25 connects into both Garnet and Gold). Configure **Site of Origin (SoO)** extended community on both PEs facing R25 so that a route learned from R25 via PE-A is not re-advertised back toward R25 via PE-B (preventing a routing loop back to the originating site across the dual-homed CE).

1. Define a unique SoO extended community value (e.g., `65200:2500`) and apply it inbound on both PEs' VRF neighbor policy toward R25.
2. Confirm outbound route-policy on each PE rejects any VPNv4 route carrying that site's own SoO value before re-advertising it toward the other PE's CE-facing peering.
3. Simulate the loop condition (temporarily removing SoO) to prove the problem exists, then re-apply SoO and confirm it's resolved.

Verify: `show bgp vpnv4 unicast vrf <name> <prefix> detail` (confirm SoO extended-community present and used as a filter), `show route vrf <name>` for routing-loop absence.

### Task 3.2 — Troubleshooting MCQs: EVPN-VPWS ESI, MC-LAG topology support

Answer the following as multiple-choice/short-answer troubleshooting items (reference the EVPN-VPWS service built in Task 2.1 and a hypothetical MC-LAG-attached CE):

1. **MCQ:** An EVPN-VPWS service with a **single-active, non-zero ESI** configured on both PE access interfaces is forwarding duplicate frames to the dual-homed CE. Which ESI/DF-election misconfiguration is the most likely cause: (a) ESI value mismatched between the two PEs, (b) DF election not run because ESI is type-0 (all zero) instead of a proper type-1/3 ESI, (c) EVI mismatch, (d) EVPN RT mismatch? Justify your answer and identify the correct `show` command to confirm ESI/DF state.
2. **MCQ:** A customer wants **MC-LAG** (multi-chassis LACP) toward a pair of Garnet PEs for a VPWS-style point-to-point access circuit, but the design review flags that **MC-LAG and EVPN-VPWS single-active multi-homing are not simultaneously required** for this topology on IOS-XR. Explain which multi-homing mechanism (MC-LAG vs. EVPN all-active/single-active) is topology-appropriate for: (a) a pair of PEs each independently uplinked without a dedicated ICCP/ICL link between them, versus (b) two PE chassis cabled with a dedicated inter-chassis link capable of running ICCP. Identify which of the two scenarios matches the EVE-NG lab's PE pairing (R14/R15 Garnet dual-homed CE) and why EVPN multi-homing — not MC-LAG — is the correct choice there.

---

## Section 4 — High Availability (10%)

### Task 4.1 — SRv6 backup paths with `spf prefix-priority critical`

In the Gold AS 65300 SRv6 core:

1. Mark the loopback prefixes of R22 and R23 (the PEs carrying the highest-priority VPN services) with **`spf prefix-priority critical`** under IS-IS so TI-LFA backup-path computation for those prefixes is **computed first**, ahead of lower-priority prefixes, during SPF — bounding worst-case backup-path convergence time for the most important services when the IGP is under CPU pressure from a large topology change.
2. Confirm TI-LFA backup SIDs are pre-installed for R22/R23's loopbacks and validate with a controlled link failure that those prefixes converge before lower-priority prefixes in the same failure domain.

Verify: `show isis fast-reroute ti-lfa`, `show isis spf-log` (confirm prioritized prefixes compute first), `show segment-routing srv6 sid` backup-path state.

### Task 4.2 — BFD for BGP between ASBRs

1. Enable **BFD** (sub-second, e.g., 150ms × 3 multiplier) on the inter-AS link(s) between R6↔R12 and/or R12↔R21 (per topology reference's inter-AS links).
2. Attach BFD to the eBGP session on each ASBR (`neighbor <addr> / bfd fast-detect`), confirm the BFD session is `UP` and independently tied to the BGP neighbor so a data-plane link failure tears down BGP in milliseconds rather than waiting on BGP's own hold-timer.
3. Fail the physical/sub-interface and measure BGP teardown time with BFD enabled vs. disabled (temporarily) to show the convergence improvement.

Verify: `show bfd session`, `show bgp neighbor <addr> | include BFD`, timestamped `show logging` around the simulated failure.

### Task 4.3 — OSPF NSF troubleshooting MCQ

**MCQ/short-answer:** During a planned RP/process restart on **R15** (Garnet OSPF PE), neighbors briefly flush R15's LSAs and the network experiences a short outage, despite **NSF (Non-Stop Forwarding) / Graceful Restart** being configured. Identify the most likely root cause among: (a) NSF is enabled on R15 but the neighboring routers (R11, R19) do not support **OSPF graceful-restart helper mode**, so they age out R15's LSAs on the normal dead-interval instead of extending it, (b) NSF requires BFD and BFD was not configured, (c) NSF only works with OSPFv3, not OSPFv2, (d) the restart exceeded the grace-period LSA's advertised duration. Explain which `show` command confirms whether a neighbor is acting as a **helper**, and what config change on R11/R19 (if any) resolves the issue.

---

## Section 5 — Security (10%)

### Task 5.1 — Management plane protection: control-plane inband, selective SSH/SNMP, deny Telnet/XML/TFTP

On every PE and ASBR (minimum: R1, R6, R12, R14, R20):

1. Configure **Management Plane Protection** using **`control-plane management-plane protection inband`** so management-destined traffic (SSH, SNMP, NETCONF/XML, etc.) is restricted to an explicit allow-list of interfaces/peers rather than being reachable from any interface by default.
2. Under the inband policy, **permit SSH and SNMP** only from a defined management subnet (e.g., `10.255.0.0/24`) via an ACL referenced in the `allow` stanza for those protocols.
3. **Explicitly deny Telnet, XML-agent (NETCONF/XML), and TFTP** in the inband policy (do not just omit them — add explicit deny peer-filters) so even a misconfigured ACL elsewhere cannot accidentally permit them.
4. Confirm the policy does not affect transit/data-plane traffic — only self-destined management protocols.

Verify: `show control-plane-protection management inband counters`, `show run control-plane`, attempt (and expect failure of) a Telnet session from outside the allowed subnet, confirm SSH/SNMP succeed from the allowed subnet.

### Task 5.2 — IS-IS authentication with keychain: HMAC-MD5, infinite lifetime, hello + LSP password

Complete the keychain reference started in Task 1.1:

1. Define `key chain ISIS-KEY` with a single `key 1`, **`cryptographic-algorithm HMAC-MD5`**, a configured key-string, and an **infinite send/accept lifetime** (no start/end time restriction — always valid).
2. Apply the keychain for **both** hello authentication (`hello-password keychain ISIS-KEY`) and **LSP authentication** (`lsp-password keychain ISIS-KEY`) under the IS-IS Level-2 interface/instance configuration on all Emerald core routers (R1–R8).
3. Confirm neighbors using a mismatched or missing keychain fail to form adjacency (test by deliberately breaking one router's key, observing adjacency drop, then restoring it).
4. Confirm LSPs generated without the correct password are rejected/flagged by neighbors (`show isis database` should not show a corrupted/unauthenticated LSP being accepted).

Verify: `show key chain ISIS-KEY`, `show isis interface detail` (confirm HMAC-MD5 auth state per interface), `show isis adjacency detail`, induced-failure test as described above.

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry troubleshooting: gRPC TLS, TPA address, port

**Troubleshooting scenario:** A streaming telemetry subscription from **R14** to a collector is configured but no data arrives at the collector.

1. Review the telemetry destination-group config: `telemetry model-driven / destination-group <name> / address-family ipv4 <collector-ip> port <port>`. Identify whether the **gRPC with TLS** transport is correctly matched between router and collector (mismatched TLS vs. no-TLS is the most common root cause for a dial-out session that never establishes).
2. Confirm the **TPA (Third-Party/Transport) source address** — `source-address <ip>` under the destination-group — is bound to a reachable, correctly-routed interface (common failure: TPA address bound to a loopback not advertised toward the collector's management network, or bound to the wrong VRF).
3. Confirm the configured **port** on the router's destination-group matches the port the collector is actually listening on, and that this port is permitted by the Task 5.1 management-plane-protection inband policy (a telemetry session can silently fail if MPP denies the gRPC port).
4. Walk through the correct verification sequence to isolate which of the three (TLS mismatch, TPA/VRF, port/ACL) is the actual fault in a given broken scenario.

Verify: `show telemetry model-driven destination-group <name>`, `show telemetry model-driven subscription <name>`, `show grpc status`, `show tpa`.

### Task 6.2 — NSO: onboard device, sync, apply template, create L3VPN service

Using Cisco NSO against the Garnet PEs:

1. **Onboard** R14 and R15 as NSO managed devices (NED for IOS-XR), including authgroup and connection parameters.
2. Run **`sync-from`** on both devices to pull their current running configuration into NSO's CDB before making any changes.
3. Apply a **device template** that pushes a baseline config snippet (e.g., a loopback description or SNMP community) to confirm template-based push works before attempting a service.
4. Create and deploy an **L3VPN service package** instance in NSO that provisions a new customer VRF across R14 and R15 (VRF name, RD/RT, PE-CE interface, routing protocol) and commit it, confirming NSO renders and pushes the correct native IOS-XR config to both devices.

Verify (NSO CLI or web UI): `show devices device <name> sync-from`, `show configuration` dry-run before commit, `show devices device <name>` operational state, confirm the resulting VRF config appears correctly on both R14 and R15 via `show running-config vrf <name>` on the routers themselves.

### Task 6.3 — NSO troubleshooting: identify out-of-sync device, re-deploy service

**Troubleshooting scenario:** Someone made a manual CLI change directly on **R15** outside of NSO, and the L3VPN service from Task 6.2 is now reporting a `check-sync` discrepancy.

1. Run **`devices device R15 check-sync`** (or NSO's compliance report) to confirm and characterize the out-of-sync condition between NSO's CDB and the device's actual running config.
2. Decide and justify the correct remediation path: either `sync-from` (if the manual change should be preserved and NSO should learn it) or `sync-to`/service **re-deploy** (if NSO's intended state should overwrite the manual change) — explain the tradeoff for each in a production SP context.
3. Re-deploy the L3VPN service (`request re-deploy`) and confirm the service's reported state returns to in-sync, with the manual drift corrected (or intentionally absorbed, per your Step 2 decision).

Verify: `show devices device R15 check-sync`, `show devices commit-queue`, post-remediation `show devices device R15 check-sync` returns in-sync, and `show running-config vrf <name>` on R15 matches the NSO-intended state.

---

## Appendix — Section Weighting Reference

| Section | Weight | Tasks |
|---|---|---|
| 1. Core Network Infrastructure | 25% | 1.1–1.7 |
| 2. Architecture and Services | 25% | 2.1–2.4 |
| 3. Access Connectivity | 25% | 3.1–3.2 |
| 4. High Availability | 10% | 4.1–4.3 |
| 5. Security | 10% | 5.1–5.2 |
| 6. Automation | 5% | 6.1–6.3 |

> This workbook intentionally sequences dependencies: Section 1 (IS-IS, SR, SRv6, Flex-Algo) must be functional before Section 2's VPN/EVPN services ride on it; Section 1.4/1.6's color-based steering is reused by Section 2.4's QoS-adjacent traffic classes conceptually; Section 5.2's keychain builds directly on the stub left in Task 1.1.
