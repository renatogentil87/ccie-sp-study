# LAB 4 — CCIE SP v5.1 Full Workbook (EVE-NG / IOS-XRv 9000)

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+)
**Scope:** Full CCIE SP v5.1 blueprint, including technologies **not supported on Dynamips/IOS 15.2** (Segment Routing, SRv6, Flex-Algo, EVPN, mVPN, MPLS-TE, Flowspec, NSO).
**Companion lab:** This reuses the same logical topology, AS numbers, router roles, and IP addressing as the GNS3 Dynamips labs (`GNS3/workbooks/00_topology_reference.md`), re-platformed onto IOS-XRv 9000. It is a **sibling of LAB3** — identical topology, but a deliberately **different task set** so you drill different features and syntax rather than repeating LAB3 from memory.
**Format:** Tasks only — no solutions. Work each task yourself; verify with `show` commands before moving on.

> **How LAB 4 differs from LAB 3 (read this first).** Same R1–R32, same 3 ASes, same loopbacks/links. The *techniques* change:
> - **IS-IS is now multi-level** (L1 on PEs/ASBRs, L1/L2 on P routers) — not L2-only.
> - **Prefix-SID index scheme is different**: `index = last octet of Loopback0 × 10` (R31→10, R14→140, R20→200).
> - **Flex-Algo uses `include-all`** (affirmative affinity) — not `exclude-any`.
> - **SR-TE uses explicit SID-lists built from prefix-SIDs *and* adjacency-SIDs** — not dynamic optimization.
> - **SRv6 moves to AS 65100 (Emerald)**; AS 65300 (Gold) now runs **SR-MPLS** (swapped vs LAB3).
> - **SR-ODN optimizes latency with an ECMP latency margin of 3000 us.**
> - **Inter-AS L3VPN is Option B on *every* inter-AS link** — no Option A anywhere.
> - **mVPN is Profile 3 (Rosen-less, BGP C-multicast, PIM-free core via mLDP/P2MP)** — not Profile 0.
> - **QoS is ingress marking + core MPLS EXP (traffic-class) treatment** — not egress shaping.
> - **SoO is on the Blue CE R25** (multi-homed to R14, R21, R23) — not Yellow R24.
> - **6VPE is for the Green customer IPv6** — not Blue.
> - **MPLS-TE (RSVP-TE) runs in AS 65200 using OSPF TE extensions** for HA — not SR-based HA.
> - **BFD protects IS-IS adjacencies** — not BGP.

---

## 0. Topology Reference (map GNS3 → EVE-NG/IOS-XR)

Same 3 autonomous systems, same roles, same R-numbers and IPs as the GNS3 base topology. Keep the plain `R#` hostnames from GNS3 throughout this lab.

| AS | Name | IGP (LAB4) | Transport (LAB4) | Routers (role) |
|----|------|-----|-----------|-----------------|
| **65100** | Emerald | IS-IS **multi-level** | **SRv6** native (swapped from LAB3) | R31(PE), R2(PE), R3(P), R4(P), R5(ASBR), R6(ASBR), R7(RR), R8(RR+PCE) |
| **65200** | Garnet | OSPF Area 0 + **RSVP MPLS-TE** | SR-MPLS + MPLS-TE | R11(P), R12(ASBR), R13(P), R14(PE), R15(PE), R16(PE), R19(RR+PCE) |
| **65300** | Gold | IS-IS **multi-level** | **SR-MPLS** (swapped from LAB3) | R20(ASBR/PE), R21(ASBR/PE), R22(PE), R23(PE), R30(RR+PCE) |

### Loopback0 (unchanged from topology reference)

| R | Lo0 | R | Lo0 | R | Lo0 |
|---|-----|---|-----|---|-----|
| R31 | 150.1.31.31/32 | R8 | 150.1.8.8/32 | R20 | 150.3.20.20/32 |
| R2 | 150.1.2.2/32 | R11 | 150.2.11.11/32 | R21 | 150.3.21.21/32 |
| R3 | 150.1.3.3/32 | R12 | 150.2.12.12/32 | R22 | 150.3.22.22/32 |
| R4 | 150.1.4.4/32 | R13 | 150.2.13.13/32 | R23 | 150.3.23.23/32 |
| R5 | 150.1.5.5/32 | R14 | 150.2.14.14/32 | R30 | 150.3.30.30/32 |
| R6 | 150.1.6.6/32 | R15 | 150.2.15.15/32 | | |
| R7 | 150.1.7.7/32 | R16 | 150.2.16.16/32 | | |
| | | R19 | 150.2.19.19/32 | | |

**CEs:** R9 150.9.9.9, R10 150.10.10.10, R17 150.17.17.17, R18 150.18.18.18, R24 150.24.24.24, R25 150.25.25.25, R26 150.26.26.26, R32 150.32.32.32.

**Core/inter-AS/PE-CE link subnets and IS-IS NET-IDs:** identical to `GNS3/workbooks/00_topology_reference.md` — reuse verbatim; only CLI changes to IOS-XR. Key links you will touch:

| Link | Subnet | Role |
|---|---|---|
| R5↔R11 | 10.5.11.0/24 | Inter-AS 65100↔65200 |
| R6↔R12 | 10.6.12.0/24 | Inter-AS 65100↔65200 |
| R6↔R20 | 10.6.20.0/24 | Inter-AS 65100↔65300 |
| R12↔R21 | 20.12.21.0/24 | Inter-AS 65200↔65300 |
| R14↔R25 | 192.168.5.0/24 | Blue PE-CE eBGP 65025 |
| R21↔R25 | 192.168.10.0/24 | Blue PE-CE eBGP 65025 |
| R23↔R25 | 192.168.12.0/24 | Blue PE-CE eBGP 65025 |
| R15↔R17 | 192.168.6.0/24 | Green PE-CE OSPF |
| R16↔R18 | 192.168.7.0/24 | Green PE-CE eBGP 65910 |

### New IOS-XR-only addressing you will add in LAB 4

| Purpose | Scheme (LAB4) |
|---|---|
| SRGB (SR-MPLS: Garnet + Gold) | 16000–23999 |
| SRLB (per node) | 15000–15999 |
| **Prefix-SID index** | **= last octet of Loopback0 × 10** (R3→30→16030; R14→140→16140; R20→200→16200) |
| Anycast SID (redundant PE pair R14/R15, Garnet) | index **1140**, label 17140, anycast loopback 150.2.114.114/32 |
| **SRv6 locator block (Emerald, AS 65100)** | `fc00:1:<R#>::/48`, F3216 uSID format |
| Flex-Algo 129 (low-latency, **include-all**) | affinity-colored links, separate SPF |
| Color communities (SR-TE / SR-ODN) | 300 = explicit SID-list TE, 400 = ODN low-latency |
| EVI numbers (EVPN-VPWS) | EVI **2001** (R31↔R16 Red circuit), EVI **2002** (R14↔R15 secondary) |
| mVPN | **Profile 3** (BGP C-multicast, mLDP P2MP core) |

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS multi-level design: L1 edge, L1/L2 backbone, wide metrics, dual-stack, overload-bit, keychain reference

On the **Emerald AS 65100** core (R1–R8), build the IS-IS process as a **multi-level** design (this is the headline change from LAB3's L2-only build):

1. Single IS-IS instance, area **49.0001**, dual-stack (IPv4 + IPv6 unicast), **single-topology** (no MT).
2. **Level assignment by role:**
   - **PEs and ASBRs** (R31, R2, R5, R6) run **Level-1 only** on their core-facing interfaces.
   - **P routers** (R3, R4) run **Level-1-2** on every core interface — they are the L1/L2 attachment points between the L1 edge and the L2 backbone.
   - **RRs** (R7, R8) run **Level-2 only** toward the P tier.
   - Set `is-type` appropriately per node (`level-1`, `level-1-2`, `level-2-only`) so the LSDB splits correctly into L1 and L2.
3. **Wide metrics** (`metric-style wide`) under both IPv4 and IPv6 unicast AFs at **both levels** — mandatory for Task 1.3's SR sub-TLVs.
4. Confirm the **L1/L2 attached-bit** behavior: the L1-only PEs/ASBRs should install a default route toward the nearest L1/L2 P router (R3/R4) rather than carrying full L2 reachability. Decide whether you need **route-leaking** (L2→L1) of specific remote-PE loopbacks so that intra-AS LSPs/SR labels resolve end-to-end (hint: SR prefix-SIDs of remote PEs must be reachable in the L1 edge — leak them down with a route-policy).
5. Set the **overload-bit** on R7 and R8 with `set-overload-bit on-startup wait-for-bgp` so the RRs do not attract transit before BGP converges after reload.
6. Reference a keychain `ISIS-KEY-L4` for **hello + LSP authentication** at both levels on all core interfaces (full keychain built in Task 5.x; here, only the reference).

Verify: `show isis neighbors` (confirm L1 vs L1L2 vs L2 adjacency types), `show isis database level-1`, `show isis database level-2`, `show route 150.1.7.7/32` on R31 (confirm it resolves via leaked route or default), `show isis adjacency detail`.

### Task 1.2 — BGP Unified MPLS (BGP-LU / Seamless MPLS) across all three ASes

Extend MPLS label-switched reachability for PE loopbacks **end-to-end across all three ASes** using BGP-LU (RFC 3107).

1. On every ASBR (R5, R6, R12, R20, R21) and RR (R7, R8, R19, R30), enable `address-family ipv4 labeled-unicast`.
2. Redistribute each AS's IGP loopbacks into BGP-LU at the ASBR boundary with a label; do **not** redistribute BGP-LU prefixes back into the IGP.
3. On R3 and R4 (Emerald P tier), configure `allocate-label all` under IPv4 unicast so inline label allocation covers all IGP prefixes — note that because Emerald is now **SRv6** (Task 1.5), confirm how BGP-LU interoperates with an SRv6 transport island at the Emerald edge (BGP-LU stitches MPLS labels in Garnet/Gold; the Emerald re-origination point translates to SRv6 — document the stitch at R5/R6).
4. Build the path so a Garnet PE (R14) can resolve a label-switched path to a Gold PE (R22) across the ASBR hops (R12↔R21 direct inter-AS link, or via R6/R20 — verify actual path per topology).

Verify: `show bgp ipv4 labeled-unicast summary`, `show mpls forwarding | include 150.3.22.22`, `show bgp labeled-unicast ipv4 unicast 150.3.22.22/32`.

### Task 1.3 — Segment Routing on IS-IS (Gold + Garnet): new prefix-SID scheme, SRGB/SRLB, anycast SID, TI-LFA, microloop avoidance

Build **SR-MPLS** under IS-IS on **Gold AS 65300** (R20–R23, R30) and SR-native on **Garnet AS 65200** (R11–R19). (Emerald is SRv6 — see Task 1.5.)

1. Enable `segment-routing mpls` under the IS-IS IPv4 unicast AF on all Gold and Garnet nodes.
2. Common **SRGB 16000–23999**, per-node **SRLB 15000–15999** on every SR node.
3. **Prefix-SID index = last octet of Loopback0 × 10** (the LAB4 scheme):
   - R20 → index **200** → label 16200; R22 → index **220** → 16220; R23 → 230 → 16230; R30 → 300 → 16300.
   - R11 → 110 → 16110; R13 → 130 → 16130; R14 → 140 → 16140; R15 → 150 → 16150; R16 → 160 → 16160; R19 → 190 → 16190.
   - R12 → 120 → 16120 (ASBR); R21 → 210 → 16210 (ASBR).
4. **Anycast SID** across redundant PE pair **R14 and R15**: both advertise prefix-SID **index 1140** (label 17140) on shared anycast loopback **150.2.114.114/32**, so inter-AS VPN traffic can target the pair as one ECMP endpoint.
5. **TI-LFA** (`fast-reroute per-prefix ti-lfa`) on every core-facing interface in Gold and Garnet.
6. **Microloop avoidance** on all SR nodes (`segment-routing microloop-avoidance` / `microloop avoidance protected` per XR train).

Verify: `show isis segment-routing label table`, `show segment-routing mpls prefix-sid-map`, `show mpls forwarding labels 16220`, `show isis fast-reroute ti-lfa`, confirm R14 and R15 both originate label 17140.

### Task 1.4 — SR-TE with **explicit SID-lists** (prefix-SID + adjacency-SID), binding-SID, autoroute, BGP color autosteer

On **R14 (Garnet PE)**, build SR-TE policies toward **R15** and toward **R22 (Gold PE)** using **explicit, hand-built SID-lists** — not dynamic optimization (this is the LAB3 contrast):

1. First, harvest the **adjacency-SIDs** you will need. On each Garnet node confirm the per-link adjacency-SID (`show isis segment-routing label adjacency` / `show mpls forwarding | include Adj`). Decide whether to use **protected** or **unprotected** adjacency-SIDs in the list and justify (protected adj-SIDs survive the local repair; unprotected pin the exact link).
2. **Policy A — color 300, to R15:** explicit segment-list forcing R14 → R13 → R16 → R15 using a **mix of prefix-SIDs and adjacency-SIDs** (e.g., prefix-SID of R13, then the **adjacency-SID** R13→R16, then prefix-SID of R15). The path must bypass the direct R11↔R15 shortest path.
3. **Policy B — color 301, to R22:** explicit segment-list that pushes the BGP-LU/inter-AS labels needed to reach the Gold PE, pinned via the R12↔R21 inter-AS link's adjacency-SID at the ASBR.
4. Assign a **binding-SID** (from SRLB, e.g., 15300) to Policy A so a parent policy or upstream node can invoke the whole explicit path with one label (hierarchical SR-TE).
5. Enable **autoroute include** on Policy A so IGP traffic to R15's loopback uses the explicit path.
6. On **R15**, attach **BGP color extended-community 300** to a set of VPNv4 routes so R14 **autosteers** that VPN traffic into Policy A by color — no PBR, no static steering.

Verify: `show segment-routing traffic-eng policy color 300 detail` (confirm the hand-built SID list and binding-SID), `show segment-routing traffic-eng policy color 301 detail`, `show bgp vpnv4 unicast <prefix> detail` (color 300 → resolved explicit tunnel), `traceroute` to confirm the forced hop sequence.

### Task 1.5 — **SRv6 on Emerald (AS 65100)**: F3216 uSID locators, PSP/USD, locator tags, SRv6 L3VPN transport

Build the **Emerald AS 65100** core (R1–R8) natively on **SRv6** using uSID / C-SID compressed format. (This is swapped from LAB3, where SRv6 was in Gold.)

1. Define an **F3216 locator plan** carved from `fc00:1:<R#>::/48` on each Emerald router (R7 → `fc00:1:7::/48`, R1 → `fc00:1:1::/48`, etc.): 32-bit block + 16-bit uSID function per hop.
2. Configure **PSP (Penultimate Segment Pop)** and **USD (Ultimate Segment Decapsulation)** behaviors on the End / End.X SIDs so the penultimate hop pops and the final hop decapsulates, trimming SRH processing on transit.
3. Advertise each locator under IS-IS SRv6 with a **tag 1000** so locator-derived routes can be filtered in route-policy for controlled inter-domain leaking (ties into the Task 1.1 L2→L1 leaking decision).
4. Build **SRv6 L3VPN transport**: on R31 and R2, configure the Blue/Green VRFs with `segment-routing srv6` encapsulation, per-VRF SID allocation (`locator <name> ... alloc mode per-vrf`), and confirm VPNv4/VPNv6 routes between R31/R2 carry an **End.DT4 / End.DT6 SRv6 SID** instead of an MPLS label.
5. Confirm the data plane shows an IPv6 SRv6 header (`show cef vrf <name> <prefix>`), not an MPLS stack.

Verify: `show segment-routing srv6 sid`, `show segment-routing srv6 locator detail`, `show isis database verbose | include SRv6|Locator`, `show bgp vpnv4 unicast vrf <name> <prefix> detail` (End.DT4 SID present).

### Task 1.6 — SR-ODN: color-triggered on-demand policies, PCE latency path, **ECMP latency margin 3000 us**

1. On **R19 (Garnet RR+PCE)**, enable **SR-PCE** (`pce address ipv4 150.2.19.19` + `segment-routing traffic-eng`), and configure **R14** as a PCC delegating computation to R19.
2. Configure **On-Demand Next-hop (ODN)** on R14: when a VPNv4 route arrives with **color 400**, auto-instantiate an SR-TE policy to that route's next-hop using an ODN template whose dynamic path optimizes the **latency (min-delay) metric** and requests PCE computation from R19.
3. **ECMP latency margin:** on the ODN template's dynamic path, set a **latency margin of 3000 microseconds** (`metric margin absolute 3000` under the min-delay optimization) so that **all paths within 3000 us of the lowest-latency path are installed as ECMP**, rather than a single best path. Justify why a margin widens the ECMP set and the tradeoff vs strict min-latency.
4. On **R15**, tag a specific customer's outbound VPNv4 routes with **color-400** so inbound traffic at R14 toward that customer triggers the ODN policy end-to-end.
5. Confirm the PCE (not headend SPF) computed the path and that multiple equal-ish-latency paths are programmed as ECMP.

Verify: `show pce lsp`, `show pce peer`, `show segment-routing traffic-eng policy color 400 detail` (confirm PCEP delegation + the 3000 us margin + >1 ECMP path), `show bgp vpnv4 unicast <prefix> detail`.

### Task 1.7 — Flex-Algo with **include-all** affinity, per-algo prefix-SID, SR policy constrained to the algo

1. On **R19**, define **Flex-Algo 129** under `segment-routing flex-algo 129`: metric-type **delay (min-latency)**, with constraint **`affinity include-all BLUE`** — Algo 129's SPF uses **only** links that carry the `BLUE` affinity (affirmative inclusion). This is the deliberate inverse of LAB3's `exclude-any` Flex-Algo.
2. Tag the desired low-latency link set in Garnet (e.g., R13↔R14, R13↔R16, R11↔R15) with affinity **BLUE** (`affinity flex-algo name BLUE` on the interfaces). Any link **not** tagged BLUE is excluded from Algo 129 by the include-all rule.
3. On every Garnet node, advertise a **per-algorithm prefix-SID** for Loopback0 under Algo 129, distinct from the Algo-0 SID (e.g., R14 Algo-0 index 140/label 16140; Algo-129 a separate SRGB-relative label such as index 1140-range — keep it distinct from the Task 1.3 anycast label).
4. Enable Flex-Algo 129 participation (`flex-algo 129`) under IS-IS SR on all Garnet nodes.
5. Build an SR-TE policy on R14 explicitly **constrained to Flex-Algo 129** (`constraints ... algorithm 129` on the dynamic path) toward R16; confirm it resolves to the Algo-129 SID and only traverses BLUE-tagged links.

Verify: `show segment-routing traffic-eng flex-algo`, `show isis flex-algo 129`, `show segment-routing traffic-eng policy name <name> detail` (algo-129 constraint, SID list stays on BLUE links), confirm a non-BLUE link is never used by Algo 129 even when it is the Algo-0 shortest path.

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — EVPN-VPWS: T-LDP VPWS + EVPN-VPWS, new EVI/AC numbers

1. Build a classic **T-LDP-signaled VPWS** pseudowire between **R16 (Garnet PE)** and **R1 (Emerald PE)** for the "Red" cross-AS circuit, **EVI/PW-ID 2001**, using `l2vpn xconnect group RED p2p RED-PW` with a targeted LDP session across the inter-AS boundary. Note R1 is SRv6-based (Task 1.5) and R16 is SR-MPLS — document how the PW is stitched across the two transport islands at the ASBR.
2. Build an **EVPN-VPWS** service (RFC 8214) between **R14** and **R15**, **EVI 2002**, with **local AC-ID 2142 on R14 and 2152 on R15** (new AC numbering vs LAB3): `l2vpn evpn evi 2002`, EVPN-VPWS service with matching local/remote AC-IDs, carried over SR-MPLS.
3. Contrast: T-LDP VPWS needs a targeted LDP session per PW; EVPN-VPWS uses BGP EVPN (RD/RT + EVI) as a single control plane over any transport.
4. Attach a customer-facing Ethernet sub-interface on each PE's access side to each PW.

Verify: `show l2vpn xconnect group RED`, `show evpn evi 2002 detail`, `show l2vpn xconnect summary`, confirm EVPN-VPWS state UP with remote AC-ID 2152/2142 matched.

### Task 2.2 — L3VPN: Inter-AS **Option B on ALL inter-AS links**, PE-CE BGP + OSPF, as-override, redistribute-connected

**Design rule for LAB4: every inter-AS VPNv4/VPNv6 handoff is Option B. Do not configure Option A (VRF-to-VRF back-to-back) anywhere.**

1. Build **intra-AS L3VPN** within Garnet for the **Green** customer: R17 via OSPF PE-CE on R15, R18 via eBGP AS 65910 on R16.
2. **Inter-AS Option B on all four inter-AS links** — R5↔R11, R6↔R12, R6↔R20, R12↔R21 — carrying VPNv4 (and VPNv6 for 6VPE, Task 3.x): VPNv4 **eBGP directly between the ASBR pairs**, each ASBR re-originating with **next-hop-self and a fresh label** (`retain route-target all` on the ASBRs so RTs are not filtered before re-advertisement). No RR relay between ASes.
3. On **some PEs use `redistribute connected`** into the VRF BGP AF (instead of `network` statements) — specifically on **R31 and R20** redistribute the connected PE-CE subnets into VPNv4 with a route-policy that scopes exactly which connected interfaces leak (avoid leaking core/management connecteds). On the remaining PEs you may keep explicit `network`/redistribute-BGP as appropriate; the point is to practice both styles and the route-policy guardrail that `redistribute connected` demands.
4. On the OSPF PE-CE link (R15↔R17) handle backbone transparency via **down-bit / domain-tag** (or sham-link if continuity is required) so Area 0 isn't falsely inferred across the core.
5. On a hub-and-spoke eBGP VPN where spoke CEs share one ASN, configure **`as-override`** on the PE so AS_PATH loop detection doesn't drop the route at the spoke.

Verify: `show bgp vpnv4 unicast vrf GREEN`, `show bgp vpnv4 unicast neighbor <peer-ASBR> advertised-routes` (confirm NH-self + label swap at each Option B hop), `show route vrf GREEN <prefix>`, confirm `redistribute connected` on R31/R20 leaks only the intended PE-CE connecteds.

### Task 2.3 — Multicast VPN: **mVPN Profile 3** (BGP C-multicast, mLDP P2MP core)

Replace LAB3's Profile 0 (Rosen/PIM-GRE) with **Profile 3**: a **PIM-free core using mLDP** for the provider tunnels and **BGP C-multicast routing** for customer join/prune signaling, across the Garnet PEs (R14, R15, R16).

1. Enable **mLDP** in the Garnet core (`mpls ldp mldp`) so Default/Data MDTs are built as **mLDP P2MP LSPs** — no PIM in the core (contrast with Profile 0's PIM-GRE MDT).
2. Configure the mVPN VRF for **Profile 3**: `vrf <name> / address-family ipv4` with the **default MDT as an mLDP P2MP tree** and **BGP MVPN SAFI** (`address-family ipv4 mvpn`) for auto-discovery (Type 1) and **C-multicast routes carried in BGP (Type 6/7)** rather than PIM across the core.
3. Enable **BGP C-multicast routing** (`bgp c-multicast-routing`) so customer (*,G)/(S,G) joins travel PE→PE in BGP MVPN Type 6/7 updates, not PIM.
4. Configure a **data MDT** range as additional mLDP P2MP trees for high-bandwidth sources, with a switchover threshold.
5. Validate: a source behind R18 (Green) crossing the data-MDT threshold triggers a new mLDP P2MP data tree, and the receiver PE issues a BGP Type 7 join.

Verify: `show mpls mldp database`, `show mvpn vrf <name> pe`, `show bgp ipv4 mvpn`, `show pim vrf <name> topology` (confirm no PIM neighbors in the core — PIM only toward CEs), `show mvpn vrf <name> database`.

### Task 2.4 — QoS: **ingress marking + core MPLS EXP (traffic-class) treatment** (not egress shaping)

Replace LAB3's hierarchical egress shaper with an **ingress classification/marking + uniform-mode MPLS EXP** design across Garnet:

1. On **R14**'s customer-facing (Green) ingress interface, build an **ingress policy-map** that classifies CE traffic into 3 classes (voice, business-critical, best-effort) matched on inbound DSCP, and **sets the MPLS experimental (traffic-class) bits** on imposition: voice→EXP 5, business-critical→EXP 3, best-effort→EXP 0 (`set mpls experimental imposition`). Also set the internal `qos-group`/`traffic-class` so the marking is carried through the box.
2. Choose and document the **MPLS DiffServ tunneling mode** — **uniform** vs **pipe/short-pipe** — and configure **uniform mode** so the EXP marking stays coupled to the customer's QoS through the SR/MPLS core and back.
3. In the **core** (R13, R11, R19 transit interfaces), build an **egress policy-map that matches on MPLS EXP (traffic-class)** and gives each EXP value its PHB: EXP 5 → low-latency priority queue, EXP 3 → bandwidth-guaranteed class, EXP 0 → default/best-effort with WRED. This is EXP-based core treatment — **no shaper**.
4. On the **egress PE (R16)** toward the CE, map the core EXP back to the correct egress DSCP (consistent with uniform mode) before handing to the customer.
5. Confirm there is no shaping anywhere in this task — the design is marking (edge) + per-EXP queuing (core).

Verify: `show policy-map interface <ingress-intf> input` (marking/EXP-set counters), `show policy-map interface <core-intf> output` (per-EXP class match + queue stats), `show mpls forwarding` to confirm EXP preservation, packet capture to confirm EXP bits in the label stack.

---

## Section 3 — Access Connectivity (25%)

### Task 3.1 — Site of Origin (SoO) on the **Blue CE R25** (multi-homed to R14, R21, R23)

The **Blue** customer CE **R25** is multi-homed into **three PEs across three ASes**: **R14 (AS 65200, 192.168.5.0/24), R21 (AS 65300, 192.168.10.0/24), and R23 (AS 65300, 192.168.12.0/24)**. Configure **Site of Origin (SoO)** so a route originated by R25 and learned via one PE is not re-advertised back toward R25 via another PE (loop prevention for the multi-homed site). This is the LAB3→LAB4 move from Yellow R24 to Blue R25, and it spans AS boundaries, so the SoO must survive the **Option B** re-origination (Task 2.2).

1. Define a single **SoO extended community** for the Blue R25 site (e.g., `65025:2500`) and apply it **inbound** in the VRF neighbor policy on **all three** PEs facing R25 (R14, R21, R23).
2. Ensure the **outbound** VRF route-policy on each of the three PEs **rejects** any VPNv4 route already carrying `65025:2500` before advertising toward its own R25-facing eBGP session.
3. Confirm the SoO is **preserved across the Option B ASBR re-origination** between AS 65200 and AS 65300 (verify the extended community is not stripped when R12↔R21 re-originates with next-hop-self) — if it is stripped, add it to the ASBR's allowed extended-community set.
4. Simulate the loop (remove SoO on one PE), observe R25 receiving its own route back, then re-apply and confirm resolution.

Verify: `show bgp vpnv4 unicast vrf BLUE <prefix> detail` on R14/R21/R23 (SoO `65025:2500` present and used as a filter), `show bgp vpnv4 unicast vrf BLUE` on R25-facing sessions (own-site routes absent), confirm SoO intact after the Option B hop.

### Task 3.2 — **6VPE for the Green customer IPv6** (not Blue)

Deliver IPv6 L3VPN (6VPE) to the **Green** customer (this is swapped from LAB3's Blue 6VPE):

1. Add **IPv6** to the Green VRF on the Garnet PEs (`vrf GREEN / address-family ipv6 unicast`) and bring up **dual-stack PE-CE**: R16↔R18 eBGP now also carries IPv6 (`address-family ipv6 unicast` on the eBGP neighbor), and R15↔R17 adds **OSPFv3** (or eBGP) for the IPv6 side.
2. Enable the **VPNv6 (6VPE) address-family** in iBGP toward the Garnet RR (R19) and across the **Option B** inter-AS handoffs so Green IPv6 prefixes reach remote PEs. Over SR-MPLS this is 6VPE (IPv6 VPN over an IPv4/MPLS or SR core); over the Emerald SRv6 segment it rides the **End.DT6** SID from Task 1.5.
3. Allocate per-VRF IPv6 labels (or SRv6 End.DT6 SIDs on the Emerald side) and confirm the control plane carries VPNv6 NLRI with the correct RD/RT.
4. Validate reachability: a Green IPv6 prefix behind R18 is reachable from a Green IPv6 prefix behind R17 across the core, and (if Blue spans in) across the Option B boundary.

Verify: `show bgp vpnv6 unicast vrf GREEN summary`, `show bgp vpnv6 unicast vrf GREEN <prefix> detail` (label or End.DT6 SID), `show route vrf GREEN ipv6 unicast`, `ping vrf GREEN <remote-IPv6-prefix>`.

### Task 3.3 — Troubleshooting MCQs: EVPN-VPWS ESI/DF election, multi-homing mode selection

Reference the EVPN-VPWS service from Task 2.1 and the multi-homed Blue CE R25.

1. **MCQ:** An EVPN-VPWS service with a **single-active, non-zero ESI** on both PE access interfaces is forwarding **duplicate frames** to the dual-homed CE. The most likely cause: (a) ESI value mismatched between the two PEs, (b) DF election never ran because the ESI is **type-0 (all zero)** instead of a proper type-1/3 ESI, (c) EVI mismatch, (d) EVPN RT mismatch. Justify, and give the `show` command that confirms ESI/DF state.
2. **MCQ:** For the **three-way multi-homed Blue CE R25** (into R14, R21, R23 across different ASes), is **EVPN all-active multi-homing** a valid loop/duplication-prevention mechanism here, or must loop prevention be handled at **L3 via SoO (Task 3.1)**? Explain why EVPN ESI multi-homing is a **single-L2-domain / single-administrative-core** construct and therefore does **not** apply to an L3VPN CE multi-homed across three separate ASes, making **SoO the correct tool**. Identify which `show` command on R25-facing PEs proves the L3 loop was prevented.

---

## Section 4 — High Availability (10%)

### Task 4.1 — **MPLS-TE (RSVP-TE) in AS 65200 with OSPF TE extensions**

Replace LAB3's SR-based HA with classic **RSVP-TE** in the OSPF-based Garnet AS (R11–R16, R19). This exercises OSPF TE opaque LSAs, RSVP signaling, and TE tunnel protection.

1. Enable **MPLS Traffic Engineering** globally and per-core-interface in Garnet (`mpls traffic-eng`), and enable **OSPF TE extensions** (`mpls traffic-eng area 0` under `router ospf`, plus `mpls traffic-eng router-id Loopback0`) so OSPF floods **Type-10 opaque TE LSAs** (link bandwidth, affinity, TE metric).
2. Reserve **RSVP bandwidth** on each TE-enabled core interface (`rsvp interface ... bandwidth <kbps>`).
3. Build a **primary TE tunnel** on **R14 → R16** with an **explicit path** (ERO) and a **dynamic path option** as fallback, bandwidth reservation, and `autoroute announce` so IGP uses the tunnel.
4. Configure **FRR (fast-reroute) link protection**: a **backup/bypass tunnel** protecting the primary tunnel's first hop, with `fast-reroute` on the primary and `backup-path` on the protecting node, so a protected-link failure triggers sub-50ms local repair onto the bypass.
5. Confirm the TE database is populated from OSPF opaque LSAs and that the tunnel's computed path honors the ERO/bandwidth constraints.

Verify: `show mpls traffic-eng tunnels`, `show mpls traffic-eng topology` (confirm OSPF-sourced TE links), `show rsvp session`, `show mpls traffic-eng fast-reroute database` (protected + bypass state), induced link failure to observe FRR cutover.

### Task 4.2 — **BFD for IS-IS** (not BGP)

Protect **IS-IS adjacencies** with BFD (the LAB3→LAB4 move from BGP to IGP):

1. Enable **BFD** (e.g., `bfd minimum-interval 150` / `bfd multiplier 3`) on the IS-IS core interfaces of the **Gold AS 65300** (R20–R23, R30) and on the **Emerald** core where applicable, under the IS-IS interface config (`address-family ... / bfd fast-detect` or interface `bfd` + `isis ... bfd`).
2. Tie BFD to IS-IS so a **data-plane link failure** tears down the IS-IS adjacency in milliseconds rather than waiting on the IS-IS hold-time, accelerating SPF/TI-LFA convergence.
3. Confirm BFD is **independent of BGP** here — the objective is IGP convergence, which in turn drives SR/TI-LFA repair.
4. Fail a core link and measure IS-IS adjacency teardown time with BFD on vs off (temporarily) to quantify the improvement, and confirm TI-LFA (Task 1.3) pre-programmed backup is used during the gap.

Verify: `show bfd session` (confirm clients include ISIS), `show isis interface <intf> | include BFD`, `show isis neighbors detail`, timestamped `show logging` around the induced failure.

### Task 4.3 — OSPF NSF / Graceful-Restart troubleshooting MCQ

**MCQ/short-answer:** During a planned RP/process restart on **R15** (Garnet OSPF PE), neighbors briefly flush R15's LSAs and the network sees a short outage despite **NSF / Graceful Restart** being configured. Identify the most likely root cause: (a) NSF is enabled on R15 but neighbors **R11 and R19 are not acting as graceful-restart helpers**, so they age R15's LSAs on the normal dead-interval instead of extending it; (b) NSF requires BFD and BFD was not configured; (c) NSF only works with OSPFv3, not OSPFv2; (d) the restart exceeded the grace-period LSA's advertised duration. Explain which `show` command confirms a neighbor is a **helper**, and the config change (if any) on R11/R19 to resolve it. Note the interaction with Task 4.1's RSVP-TE tunnels during the restart.

---

## Section 5 — Security (10%)

### Task 5.1 — Management plane protection + IS-IS keychain (complete the Task 1.1 reference)

1. **MPP inband** on every PE and ASBR (minimum R1, R6, R12, R14, R20): `control-plane management-plane protection inband`, permitting **SSH and SNMP only** from management subnet `10.255.0.0/24` via an ACL, and **explicitly denying Telnet, XML-agent (NETCONF/XML), and TFTP** with explicit deny peer-filters (not just omission). Confirm no effect on transit/data-plane traffic.
2. Complete **keychain `ISIS-KEY-L4`** referenced in Task 1.1: `key chain ISIS-KEY-L4 / key 1 / cryptographic-algorithm HMAC-MD5 / key-string <...>` with **infinite send/accept lifetime**. Apply it for **both hello-password and lsp-password at both IS-IS levels** (L1 and L2) on all Emerald core routers — remember Task 1.1 is multi-level, so authenticate **L1 and L2 LSPs/hellos separately** if the design uses distinct passwords per level (decide and document whether one keychain covers both levels or you split L1 vs L2 keys).
3. Induced-failure test: break one router's key, observe the adjacency drop at the correct level, restore, confirm recovery.

Verify: `show control-plane-protection management inband counters`, attempt a Telnet from outside `10.255.0.0/24` (expect fail) and SSH from inside (expect success), `show key chain ISIS-KEY-L4`, `show isis interface detail` (HMAC-MD5 per level), `show isis adjacency detail`.

### Task 5.2 — **BGP Flowspec** (concepts + config attempt), **TCP-AO** on BGP, **RSVP authentication** on MPLS-TE

1. **BGP Flowspec (document even if the XRv image can't fully program it in dataplane):**
   - Explain the Flowspec AF (`address-family ipv4 flowspec`) and how a controller/PE originates a Flowspec rule (match 5-tuple → action: drop, rate-limit, redirect-to-VRF, remark DSCP) distributed via BGP (RFC 5575/8955).
   - Attempt to configure a Flowspec rule on **R19** (as the Flowspec controller) matching a mock DDoS flow toward a Green customer prefix with action **rate-limit** and **redirect-to-VRF**, and advertise it to R14/R15/R16.
   - If the XRv platform does not install the Flowspec action in the data plane, **document the control-plane state** (`show flowspec afi-all`, `show bgp ipv4 flowspec`) and clearly note the dataplane limitation rather than claiming success.
2. **TCP-AO (TCP Authentication Option, RFC 5925) on BGP:** discuss TCP-AO vs legacy TCP-MD5 for the inter-AS eBGP sessions (R6↔R12, R12↔R21). Configure a **keychain-backed TCP-AO** on at least one eBGP session (`neighbor ... ao <keychain> include-tcp-options`), explain the MKT/key-id rollover advantage over MD5, and verify the session stays up with AO.
3. **RSVP authentication on the MPLS-TE interfaces (Task 4.1):** configure **RSVP authentication** (`rsvp interface ... authentication / key-source key-chain <name>`) on the Garnet TE-enabled core interfaces so RSVP PATH/RESV messages are authenticated, preventing spoofed reservation/teardown. Confirm TE tunnels from Task 4.1 re-establish with RSVP auth enabled.

Verify: `show bgp ipv4 flowspec`, `show flowspec afi-all` (+ documented dataplane note), `show bgp neighbor <addr> | include AO|Authentication`, `show rsvp interface detail` (authentication enabled), confirm Task 4.1 tunnels stay UP.

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry troubleshooting: gRPC TLS, TPA/VRF source-address, port vs MPP

**Troubleshooting scenario:** A model-driven telemetry subscription from **R14** to a collector produces no data at the collector.

1. Check the destination-group: `telemetry model-driven / destination-group <name> / address-family ipv4 <collector-ip> port <port>`. Determine whether **gRPC-with-TLS vs no-TLS** is matched on both ends (most common dial-out failure).
2. Confirm the **TPA source-address** (`source-address <ip>` / correct **VRF**) is bound to a reachable, advertised interface — not a loopback invisible to the collector's management network, and not the wrong VRF.
3. Confirm the configured **port** matches the collector's listener **and** is permitted by the Task 5.1 **MPP inband** policy (a telemetry session silently fails if MPP denies the gRPC port — this is the cross-link to Section 5).
4. Give the correct isolation sequence to decide which of {TLS mismatch, TPA/VRF, port/MPP} is the actual fault.

Verify: `show telemetry model-driven destination-group <name>`, `show telemetry model-driven subscription <name>`, `show grpc status`, `show tpa`.

### Task 6.2 — NSO: onboard, sync-from, device template, deploy L3VPN service

Using Cisco NSO against the Garnet PEs:

1. **Onboard** R14 and R15 as NSO-managed devices (IOS-XR NED), with authgroup and connection parameters.
2. `sync-from` both devices into CDB before any change.
3. Apply a **device template** pushing a baseline snippet (loopback description or SNMP community) to prove template push works.
4. Create and deploy an **L3VPN service package** instance provisioning a new customer VRF across R14 and R15 (VRF name, RD/RT, PE-CE interface, routing protocol); commit and confirm NSO renders correct native IOS-XR config to both.

Verify: `show devices device <name> sync-from`, dry-run `commit dry-run` before commit, `show devices device <name>` operational state, and on the routers `show running-config vrf <name>`.

### Task 6.3 — NSO troubleshooting: **identify the out-of-sync device, re-deploy reconcile**

**Troubleshooting scenario:** Someone made a manual CLI change directly on one of the Garnet PEs outside NSO, and the Task 6.2 L3VPN service now reports a `check-sync` discrepancy. You do **not** yet know which device drifted.

1. Run a **compliance / check-sync across all devices** (`devices check-sync` or `devices device * check-sync`) to **identify which device is out-of-sync** (expected: the one with the manual change — determine whether it is R14 or R15 from the output, don't assume).
2. Produce the drift detail with `devices device <drifted> compare-config` so you can see exactly what the manual change was.
3. Decide and justify remediation:
   - `sync-from` if the manual change should be **kept** and learned by NSO (CDB updated to match device), **or**
   - **service `re-deploy reconcile`** / `sync-to` if NSO's intended state should **overwrite** the drift (reconcile keeps NSO-owned config authoritative while preserving non-service config).
   - Explain the production tradeoff of each.
4. Run **`request services <l3vpn-service> re-deploy reconcile`** (or `sync-to`) and confirm the service returns **in-sync** with the drift corrected (or intentionally absorbed per step 3).

Verify: `devices device * check-sync` (locate the drifted device), `devices device <drifted> compare-config`, post-remediation `devices device <drifted> check-sync` returns in-sync, `show devices commit-queue`, and on the router `show running-config vrf <name>` matches NSO intent.

### Task 6.4 — Ansible troubleshooting MCQ (encoding error)

**MCQ/short-answer:** An Ansible playbook using the `cisco.iosxr` collection to push config to **R15** fails mid-run with an error resembling:

```
UnicodeDecodeError: 'ascii' codec can't decode byte 0x... in position ...: ordinal not in range(128)
```

Identify the most likely root cause and fix:
- (a) The control node's locale is set to `C`/`POSIX` (ASCII) so Python cannot decode non-ASCII bytes in the device output/template — **fix: set `LANG`/`LC_ALL` to a UTF-8 locale** (e.g., `en_US.UTF-8`) on the Ansible control node (and/or set `ansible_python_interpreter` to a Py3 interpreter).
- (b) The IOS-XR NED is the wrong version.
- (c) NETCONF is disabled on R15.
- (d) The BGP AF is misconfigured.

Explain **why** a `C`/ASCII locale produces a `UnicodeDecodeError` when the playbook or device output contains a non-ASCII character (e.g., an accented character in a description, or a byte in a banner), why Python 3 + a UTF-8 locale resolves it, and which environment variables / `ansible.cfg` or inventory settings you would set. Also note the interaction with Task 5.1 MPP (ensure the transport Ansible uses — NETCONF/SSH — is permitted by the inband policy and not mistaken for an encoding fault).

---

## Appendix — Section Weighting Reference

| Section | Weight | Tasks |
|---|---|---|
| 1. Core Network Infrastructure | 25% | 1.1–1.7 |
| 2. Architecture and Services | 25% | 2.1–2.4 |
| 3. Access Connectivity | 25% | 3.1–3.3 |
| 4. High Availability | 10% | 4.1–4.3 |
| 5. Security | 10% | 5.1–5.2 |
| 6. Automation | 5% | 6.1–6.4 |

> **Dependency sequencing.** Section 1 (multi-level IS-IS, SR-MPLS in Gold/Garnet, SRv6 in Emerald, Flex-Algo include-all) must be functional before Section 2's services ride on it. Section 1.4's explicit SID-lists and 1.6's ODN color steering feed Section 2's L3VPN/EVPN. Section 2.2's Option-B-everywhere boundary is what Section 3.1's SoO must survive. Section 4.1's RSVP-TE interfaces are the ones Section 5.2 authenticates (RSVP auth). Section 5.1's MPP inband policy is the cross-check for Section 6.1 telemetry and 6.4 Ansible transport. Keychains referenced in Task 1.1 are completed in Task 5.1.

> **LAB 4 vs LAB 3 — quick diff for self-checking.** Multi-level IS-IS (not L2-only) · prefix-SID ×10 (not ×1) · Flex-Algo include-all (not exclude-any) · explicit SID-lists w/ adjacency-SIDs (not dynamic) · SRv6 in Emerald (not Gold) · SR-ODN 3000 us ECMP margin · Option B everywhere (no Option A) · EVI 2001/2002 (not 1001/1002) · mVPN Profile 3 (not Profile 0) · QoS ingress-mark+EXP (not egress shape) · redistribute-connected on R31/R20 · SoO on Blue R25 (not Yellow R24) · 6VPE for Green (not Blue) · RSVP MPLS-TE in AS65200 (not SR HA) · BFD for IS-IS (not BGP) · Flowspec + TCP-AO + RSVP-auth · NSO reconcile + Ansible encoding MCQ.
