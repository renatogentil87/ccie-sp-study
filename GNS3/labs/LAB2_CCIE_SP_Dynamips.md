# CCIE SP Lab 2 — Full Scenario (GNS3 / Dynamips)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Emulator:** GNS3 Local (Dynamips only — no IOS-XRv/CSR/NX-OSv features)
**Topology:** Same physical GNS3 project as LAB1 — 31 routers, 3 Autonomous Systems
**Reference:** `../workbooks/00_topology_reference.md` — addressing, link map, loopbacks, NET-IDs, IPv6 scheme, console ports
**Style:** Tasks only. No configuration snippets or solutions. Specific parameters are given; you derive the CLI.
**Relationship to LAB1:** Same wiring, same loopbacks/link subnets — different protocol choices, different policy design, different VPN/HA/security/IPv6 technology emphasis. Treat this as an independent build from a clean/base-IP-only snapshot; do not assume LAB1's control-plane config is present.

---

## Before You Start

- Start from a clean snapshot with only interface IP addressing applied (equivalent to `WB00-foundation-ip-complete`). If you still have a LAB1 end-state snapshot, revert to the pre-IGP baseline — this lab intentionally runs **different IGP designs** (multi-level IS-IS, OSPF with stub/NSSA) that would conflict with LAB1's L2-only IS-IS / plain OSPF area 0 config.
- Take a snapshot named `LAB2-00-start` before Section 1.
- Take one snapshot at the end of each Section (naming given in each section).
- Scoring is out of **300 points** across 6 sections (see Completion Checklist for the breakdown).

---

## Topology Recap (from topology reference)

| AS | Name | IGP (LAB2) | Routers | RRs | ASBRs | PEs | P routers |
|---|---|---|---|---|---|---|---|
| **65100** | SP-A | IS-IS **multi-level (L1/L2)** | R1–R8 | R7, R8 | R5, R6 | R31, R2 | R3, R4 |
| **65200** | SP-B | OSPF, **stub + NSSA areas** | R11–R16, R19 | R19 | R12 | R14, R15, R16 | R11, R13 |
| **65300** | SP-C | IS-IS **multi-level (L1/L2)** | R20–R23, R30 | R30 | R20, R21 | R20, R21, R22, R23 | — |

CE routers: R9, R10 (Green), R17, R18 (Green/OSPF), R24, R26 (Yellow), R25, R32 (Blue), R27, R28, R29 (Red VPLS).

---

## Section 1 — Core IGP, LDP, and BGP Path Control [70 points]

### 1A. IS-IS Multi-Level — AS 65100 (R1–R8) [16 points]

#### Task 1.1 — Define the area split [2 points]
Split AS 65100 into two IS-IS areas instead of LAB1's single L2-only domain:
- **Area 49.0001** (L1/L2 backbone): R3, R4, R5, R6, R7, R8
- **Area 49.0011** (L1-only edge): R31, R2
- R3 and R4 are the **L1/L2 border routers** connecting the edge area to the backbone (R31↔R3, R2↔R4 per the Core link table).
- Use the existing NET-IDs from the topology reference's system-ID column, but change the **area portion** of R31 and R2's NET to `49.0011` (keep their system-ID octets identical).

#### Task 1.2 — Level assignment per router [3 points]
- R31, R2: `is-type level-1` (pure L1 edge routers).
- R3, R4: `is-type level-1-2` (border routers — must run both levels).
- R5, R6, R7, R8: `is-type level-2-only` (pure backbone — no L1 needed since the only L1 area hangs off R3/R4).

#### Task 1.3 — Interface-level circuit types [3 points]
- On R3's R1-facing interface and R4's R2-facing interface, restrict the circuit to `level-1` only.
- On all other AS 65100 core interfaces (backbone-to-backbone), restrict the circuit to `level-2-only`.
- Loopback0 passive on all 8 routers, in whichever level(s) that router runs.

#### Task 1.4 — Default route leaking into the L1 area [4 points]
By default, L1-only routers (R31, R2) only know a default route toward the nearest L1/L2 border (ATT bit). Verify this default-route behavior is working. Then, as a CCIE-level exercise, configure **route leaking** (`redistribute` with `level-1-2` route-map / `set-overload-bit` is NOT what's wanted here — use **IS-IS inter-level route leaking**) on R3 to leak **R20's and R30's /32 loopbacks only** (not the whole L2 table) down into Area 49.0011, so R31/R2 have specific routes to AS 65300's RR and one ASBR in addition to the default. Use tagged route leaking (IS-IS route tag or a leaking route-map) so this is auditable.

#### Task 1.5 — Wide metrics + P2P network type [2 points]
Metric style **wide** domain-wide (both levels). All core links **point-to-point** network type.

#### Task 1.6 — Verify multi-level IS-IS [0 points — verification]
Confirm: R31/R2 show `ATT` bit learned from R3/R4 respectively, L1 database on R31/R2 contains only Area 49.0011 + the two leaked /32s + default, L2 database on R3–R8 contains all backbone routers, and full loopback reachability exists end-to-end within AS 65100.

#### Task 1.7 — Snapshot [2 points]
`LAB2-sec1a-isis-ml-as65100`

---

### 1B. OSPF Stub/NSSA — AS 65200 (R11–R16, R19) [14 points]

#### Task 2.1 — Area redesign [3 points]
Unlike LAB1's single Area 0, split AS 65200 into:
- **Area 0** (backbone): R11, R12, R13, R19
- **Area 20** (totally stubby): R14 — models a PE with a single uplink that needs minimal LSA flooding
- **Area 21** (NSSA): R15, R16 — models PEs that must originate **Type-7 LSAs** for externally-learned prefixes (anticipating PE-CE redistribution in Section 2)
- R13 is the ABR for both Area 20 and Area 21 (R13↔R14 and R13↔R16 links already exist per the Core link table; renumber those two links' OSPF area membership accordingly — the IP addressing from the topology reference does not change).

#### Task 2.2 — Totally stubby configuration [3 points]
Configure Area 20 as **totally stubby** from the R13 ABR (`area 20 stub no-summary`). Verify R14 receives only a single default route (0.0.0.0/0) from Area 0/other areas and no inter-area summaries.

#### Task 2.3 — NSSA configuration [3 points]
Configure Area 21 as **NSSA** on R13 (ABR) and R15/R16 (internal). Do not inject a default route into the NSSA yet (no `default-information-originate` on the ABR) — that's validated in Task 2.4.

#### Task 2.4 — NSSA default + Type-7 translation [3 points]
- On R13, configure `area 21 nssa default-information-originate` so Area 21 gets a default route.
- Later (Section 2), R16's static PE-CE customer route will need to appear as a Type-7 LSA in Area 21, translated to Type-5 at R13. For now, originate a **test static route** on R16 (e.g., a loopback-style /32 you choose for testing) and redistribute it into OSPF so it appears as Type-7 in Area 21 and Type-5 once translated into Area 0. Verify translation using `show ip ospf database nssa-external` on R13 vs the view from R11.

#### Task 2.5 — Verify areas [0 points — verification]
Confirm `show ip ospf border-routers`, `show ip ospf database database-summary`, and per-area LSA counts match expectations (no unexpected Type-3 summaries in Area 20; Type-7/Type-5 translation visible for Area 21).

#### Task 2.6 — Snapshot [2 points]
`LAB2-sec1b-ospf-stub-nssa`

---

### 1C. IS-IS Multi-Level — AS 65300 (R20–R23, R30) [10 points]

#### Task 3.1 — Area split [3 points]
- **Area 49.0003** (L1/L2 backbone): R21, R22, R23, R30
- **Area 49.0013** (L1-only edge): R20
- R21 is the L1/L2 border router (R20↔R21 link).
- Change only the area portion of R20's NET to `49.0013`.

#### Task 3.2 — Levels and circuit types [3 points]
- R20: `level-1`. R21: `level-1-2`. R22, R23, R30: `level-2-only`.
- R20↔R21 circuit: `level-1` only. All other AS 65300 core links: `level-2-only`.

#### Task 3.3 — Wide metrics, P2P, passive loopbacks [2 points]
Same treatment as 1A: wide metrics, point-to-point network type, passive Loopback0.

#### Task 3.4 — Verify and snapshot [2 points]
Confirm ATT-bit default routing works for R20, full backbone loopback reachability for R21/R22/R23/R30. Snapshot: `LAB2-sec1c-isis-ml-as65300`.

---

### 1D. LDP Across All Three ASes [8 points]

#### Task 4.1 — Enable LDP on core links [3 points]
Enable LDP on every intra-AS core link in all three ASes (same link set as the Core link tables; do NOT enable LDP on inter-AS or PE-CE links yet). Use `mpls ip` per-interface (not global auto-enable), to keep explicit control given the multi-level/multi-area IGP split.

#### Task 4.2 — LDP Router-ID [2 points]
Explicitly pin the LDP Router-ID to Loopback0 on all MPLS-enabled routers (`mpls ldp router-id Loopback0 force`) so LDP sessions aren't affected by interface flaps.

#### Task 4.3 — Verify LSPs [3 points — includes verification]
Confirm LDP neighbor sessions are `Operational` across each AS and that `show mpls forwarding-table` shows a label for every /32 loopback. Pay special attention to the multi-level IGP boundary (R3/R4 in AS65100, R21 in AS65300) — confirm labels are learned and swapped correctly across the L1/L2 boundary.

---

### 1E. BGP — Weight-Based Path Control, Conditional Advertisement, Communities, BGP-LU [22 points]

#### Task 5.1 — iBGP + Route Reflectors [4 points]
Build the same RR topology as LAB1 (R7+R8 RRs in AS65100; R19 RR in AS65200; R30 RR in AS65300; clients per the topology reference), with IPv4 unicast + VPNv4 AFs activated, `update-source Loopback0`. (This baseline is unchanged from LAB1 — build it fresh since you started from a clean snapshot.)

#### Task 5.2 — eBGP inter-AS sessions [3 points]
Bring up eBGP on all four inter-AS links (R5↔R11, R6↔R12, R6↔R20, R12↔R21), IPv4 unicast AF, peering on connected addresses.

#### Task 5.3 — WEIGHT instead of LOCAL_PREF for primary/backup [5 points]
AS65100 has two links to AS65200 (R5↔R11, R6↔R12). Unlike LAB1 (which used LOCAL_PREF), use **BGP WEIGHT** (a purely local, non-transitive attribute) on R5 and R6 to prefer the R5↔R11 path:
- Set weight **200** on routes learned via R5 from R11, and weight **100** on routes learned via R6 from R12 (default weight for locally-originated is 32768 — ensure your weight values don't accidentally lose to that where irrelevant).
- Because weight is per-router/non-propagated, you must apply it **individually on R5 and on R6** (not centrally on the RR) — explain in your verification notes why weight cannot be set on the RR to achieve this result, unlike LOCAL_PREF.
- Verify AS65100's best path to AS65200 loopbacks prefers the R5↔R11 link, and confirine failover to R6↔R12 if R5↔R11 is shut.

#### Task 5.4 — Conditional advertisement (advertise-map/exist-map) [5 points]
On R6 (AS65100 ASBR facing both AS65200 and AS65300), configure **conditional route advertisement**: advertise a backup summary/aggregate for AS65300 prefixes toward AS65200 (R12) **only when** a specific trigger condition prefix (e.g., R20's loopback as seen via the direct R6↔R20 eBGP session) is **not** present in R6's BGP table — i.e., only advertise the backup path when the primary advertisement source disappears. Use `exist-map`/`non-exist-map` semantics (choose and justify which one fits "advertise only when primary is down"). Verify by shutting the R6↔R20 session and confirming the conditional route appears toward R12, then confirming it is withdrawn when R6↔R20 is restored.

#### Task 5.5 — New community scheme [3 points]
Design and apply a **new, documented community numbering scheme** (different from any LAB1 scheme) for marking route origin and treatment, for example:
- `65100:1100` = originated in AS65100 core
- `65100:1199` = customer-originated (Green/Blue), do-not-prepend
- `65200:2100` / `65300:3100` = equivalent per-AS origin tags
Tag loopbacks and customer-origin prefixes at origin (R1–R8 for 65100-originated; R20/R21 for 65300) with the appropriate community via route-map on redistribution/network statement, and verify communities are visible end-to-end at a remote ASBR (`show ip bgp community`).

#### Task 5.6 — BGP-LU (Labeled Unicast) — different peering strategy than LAB1 [2 points]
Enable BGP Labeled Unicast (`address-family ipv4 ... send-label` / `mpls bgp forwarding`) to extend label-switched reachability for loopbacks across the inter-AS boundaries — but instead of LAB1's RR-centric iBGP-LU design, use a **direct ASBR-to-ASBR eBGP-LU session plus a separate, dedicated internal LU session from each ASBR to its own area's L1/L2 border or RR** (i.e., BGP-LU is carried hop-by-hop through explicit sessions rather than reflected). Build this only for the AS65100 ↔ AS65300 path (via R6↔R20): R6↔R20 eBGP-LU, plus R6→R7/R8 (or R6→R3/R4, your choice — justify) iBGP-LU, plus R20→R21/R30 iBGP-LU on the AS65300 side. Verify an LSP (via `show mpls forwarding-table` / LSP ping) exists end-to-end from an AS65100 PE loopback to an AS65300 PE loopback riding purely on labels (no plain IP core hop).

---

## Section 2 — VPN Services [60 points]

### Task 2.1 — VRF design table [4 points]
Build the VRF/RT/RD plan below (same five customers as LAB1's design, but note the PE-CE protocol and Inter-AS option changes called out per customer):

| Customer | VRF Name | RD (per PE, use `<PE-loopback-last-octet>:<customer-id>`) | RT | PE-CE Protocol (LAB2) |
|---|---|---|---|---|
| Green | `GREEN` | e.g. `1:910` on R31 | `65910:910` both import/export | eBGP (R31↔R9, R31↔R10, R2↔R10) — **unchanged from LAB1** |
| Green-OSPF→**Static** | `GREEN` (same VRF, R15↔R17 leg) | same RD scheme on R15 | same RT | **STATIC PE-CE instead of OSPF** on R15↔R17 |
| Yellow | `YELLOW` | `1:24` scheme | `65024:24` | eBGP — unchanged |
| Blue | `BLUE` | `1:25` scheme | `65025:25` | eBGP — unchanged, but Inter-AS option changes (Task 2.5) |
| Red (VPLS) | n/a (L2) | n/a | New VPN-ID (Task 2.6) | VPLS/VPWS |

### Task 2.2 — Green customer VRF + eBGP legs [5 points]
Create VRF `GREEN` on R31 and R2 per the RD/RT table above. Configure eBGP PE-CE (AS 65910) on R31↔R9, R31↔R10, R2↔R10 exactly as wired in the topology reference's PE-CE link table. Redistribute/exchange VPNv4 routes for Green between R31 and R2 via MP-iBGP (through the AS65100 RRs).

### Task 2.3 — Green-leg static PE-CE (R15↔R17) — replaces LAB1's OSPF PE-CE [8 points]
Where LAB1 used OSPF between R15 and R17, LAB2 uses **static routing**:
- Place R17's loopback and any simulated customer subnet into VRF `GREEN` on R15 (same VRF as the eBGP Green legs, so this customer is single-VRF across AS65100+AS65200, same as LAB1's intent).
- Configure a **static route in the VRF** on R15 pointing to R17's networks via the R15↔R17 interface (`ip route vrf GREEN ...`).
- Configure a **static default (or specific) route on R17 (CE)** pointing back to R15.
- Redistribute the VRF static route into MP-BGP VPNv4 on R15 (`redistribute static` under the VRF address-family) so Green's AS65100 PEs learn R17's prefix.
- Verify two-way reachability from a Green prefix in AS65100 to R17, and confirm the route shows as **static-origin** (not OSPF) in `show ip bgp vpnv4 vrf GREEN`.

### Task 2.4 — Yellow customer VRF [4 points]
Create VRF `YELLOW` on R16 (AS65200) and R20/R22 (AS65300) per the PE-CE link table (R16↔R26, R20↔R24, R22↔R24). eBGP AS 65024 PE-CE. Exchange VPNv4 across the R12↔R21 inter-AS link area (see Task 2.5 for the Inter-AS mechanism actually used).

### Task 2.5 — Blue customer VRF + Inter-AS Option B (R5↔R11) [12 points]
LAB1 used Inter-AS **Option A** (VRF-lite, back-to-back) on R5↔R11. LAB2 uses **Inter-AS Option B** on that same link instead:
- Create VRF `BLUE` on R2 (AS65100, serving R32), and on R14/R21/R23 (AS65200/65300, serving R25) per the PE-CE link table.
- On R5 and R11, remove any per-VRF subinterface approach — instead configure a **single multihop-free eBGP VPNv4 session directly between R5 and R11** (Option B): exchange **labeled VPNv4** routes directly, with each ASBR **rewriting the next-hop to itself** (`next-hop-self` is implicit in the Option B's label-swap behavior — confirm R5/R11 swap the inter-AS label at the ASBR rather than passing through client labels unchanged, which is the key Option A vs. B distinction).
- Verify the VPNv4 label stack: label swap occurs at the R5/R11 ASBR boundary (2 labels used end-to-end: IGP/LDP label + VPN label re-originated by the receiving ASBR), vs. Option A's independent per-AS VRF label spaces.
- Confirm Blue customer reachability from R32 (AS65100) to R25 (reachable via both AS65200 and AS65300 per the topology reference — pick and document which path is preferred and why).
- Document in your lab notes, in 3–5 bullet points, the operational and scaling difference between what you built here (Option B) and what Option A would have required (VRF per neighbor AS at every boundary vs. one VPNv4 session).

### Task 2.6 — Red VPLS — new VPN-ID scheme [10 points]
LAB1 used one VPN-ID for the Red VPLS instance. LAB2 requires **two separate VPLS instances with distinct VPN-IDs**, splitting what was one flat VLAN 100 domain:
- **VPLS instance "RED-NORTH"**, VPN-ID `65100:9100`: members R27 (R16) and R29 (R1) — i.e., the AS65200↔AS65100 pseudowire pair.
- **VPLS instance "RED-SOUTH"**, VPN-ID `65100:9200`: members R28 (R14) only plus a new simulated third site — since only R14/R28 remain, build this as a **VPWS (point-to-point)** pseudowire instead of full VPLS mesh (document why a 2-site "VPLS" degenerates to VPWS, and configure it as such, OR keep it provisioned as VPLS with a single PW for extensibility — your choice, document which and why).
- Keep VLAN 100 as the access encapsulation on all CE-facing ports (R27, R28, R29 unchanged from the physical/VLAN side), but the **core pseudowire signaling must use the two new VPN-IDs above**, not LAB1's single ID.
- Verify MAC learning / flooding is correctly scoped per instance — a MAC seen on RED-NORTH must NOT appear in RED-SOUTH's forwarding table.

### Task 2.7 — QoS: ingress policing (replaces LAB1's egress shaping) [12 points]
LAB1 applied egress shaping toward customers. LAB2 instead applies **ingress policing** at the PE-CE edge:
- On R1 (Green, R9/R10-facing), R16 (Yellow, R26-facing), and R21 (Blue, R25-facing), build a 3-class ingress policy-map (`class-map` matching DSCP or an access-list proxy for customer-marked traffic: e.g., EF→voice, AF31→business-critical, default→best-effort).
- Apply **`police`** (single-rate, two-color: conform/exceed, no "violate"/dual-rate needed) ingress on the PE-CE-facing interface for each class:
  - Voice (EF): police to **512 kbps**, conform-action transmit, exceed-action drop.
  - Business (AF31): police to **2 Mbps**, conform-action transmit, exceed-action set-dscp-transmit (remark down to AF32... actually remark to **AF32** to permit but de-prioritize), i.e. markdown instead of drop.
  - Best-effort (default class): police to **1 Mbps**, conform-action transmit, exceed-action drop.
- Verify with `show policy-map interface` counters that conforming/exceeding/violating packet/byte counts move as expected under a generated test flow (or document expected counter behavior if traffic generation isn't available in Dynamips).
- Explicitly contrast (in notes) why ingress policing (drop/remark at entry, no buffering) behaves differently from LAB1's egress shaping (buffer/delay at exit) for burst absorption.

### Task 2.8 — Verify Section 2 end-to-end [3 points — verification]
From each CE (R9, R10, R17, R18, R24, R25, R26, R32), ping/traceroute the relevant remote CE(s) for its VPN. Confirm VRF route counts match expectations and that no customer VRF leaks routes to another (`show ip route vrf <name>` isolation check).

### Task 2.9 — Snapshot [2 points]
`LAB2-sec2-vpn-services`

---

## Section 3 — Access, Multicast, and SoO [45 points]

### Task 3.1 — mVPN Profile 3 (BGP C-Multicast Signaling) instead of Profile 0 [20 points]
LAB1 used **Profile 0** (PIM-SM default MDTs, PIM C-multicast signaling via the GRE default MDT). LAB2 uses **mVPN Profile 3** (BGP-based MDT and BGP C-multicast signaling, no PIM in the SP core for customer multicast) for the Green VPN:

- **3.1.1 — [4 points]** Configure a **BGP-based MDT** for VRF `GREEN` on R31, R2, R15: enable `address-family ipv4 mdt` under MP-BGP (in addition to the vpnv4 AF) so MDT membership is signaled via BGP (AFI/SAFI MDT, RFC 6037-style) rather than PIM MSDP/Auto-RP style discovery.
- **3.1.2 — [4 points]** Configure the VRF-level multicast with `mdt default` and `mdt data` bound to the BGP-signaled group ranges, confirming no PIM MDT discovery messages are needed in the P-network core for MDT setup (contrast with Profile 0's reliance on PIM in the core — call this out explicitly in notes).
- **3.1.3 — [6 points]** Enable **BGP C-multicast signaling**: configure the `ipv4 mvpn` (or vendor-equivalent `address-family ipv4 mvpn`) BGP AF between the Green PEs (R31, R2, R15) so that PIM Join/Prune information from customer sites is signaled as BGP C-multicast routes (Type 6/7 in RFC 6514 terms) across the backbone instead of native PIM C-instance signaling through a GRE MDT tunnel.
- **3.1.4 — [4 points]** Build a simple multicast source/receiver test: have R9 (or R10) source a multicast stream and R17 (or R18) join it as a receiver, both inside VRF GREEN. Verify the (S,G) state is correctly signaled end-to-end using BGP C-multicast routes (check `show bgp ipv4 mvpn` / `show ip pim vrf GREEN mdt bgp` equivalents) and that the data plane multicast flow is received.
- **3.1.5 — [2 points]** Document the three signaling differences between Profile 0 and Profile 3 you directly observed: (a) MDT discovery mechanism, (b) C-multicast signaling transport, (c) core PIM dependency.

### Task 3.2 — SoO: Blue customer R25 multi-homed (replaces LAB1's Yellow/R24 SoO scenario) [18 points]
LAB1's Site-of-Origin scenario was built around Yellow/R24. LAB2 instead builds SoO around **Blue customer's R25**, which (per the topology reference) is multi-homed into **both AS65200 (via R21) and AS65300 (via R23)**:

- **3.2.1 — [4 points]** Define a Site-of-Origin extended community for R25's site, e.g., `SoO:65025:2500`. Apply it on **both** PE-CE sessions that connect to R25 (R21↔R25 and R23↔R25) using an inbound route-map on the PE side that tags all routes learned from R25 with this SoO community.
- **3.2.2 — [4 points]** Confirm — using the standard SoO loop-prevention behavior — that a route tagged with R25's SoO community, learned on R21, is **not** re-advertised back out to R25 via R23 (and vice versa), which would otherwise create a routing loop or suboptimal path given R25's dual-homing.
- **3.2.3 — [4 points]** Deliberately break the SoO filter (remove it from one leg only) and demonstrate the resulting problem: show the route reappearing at R25 from the "wrong" direction (looped or duplicated), using `show ip bgp vpnv4 vrf BLUE <prefix>` on R25-facing interfaces or a traceroute anomaly. Then restore the filter and reverify the loop is gone.
- **3.2.4 — [4 points]** Extend the design: configure **local-preference or weight-based primary/backup path selection** (your choice; if you choose weight, note the same per-router caveat from Section 1 Task 5.3) so that R21 (AS65200 leg) is R25's primary upstream and R23 (AS65300 leg) is backup, while still retaining the SoO protection from 3.2.1. Verify both the primary-path preference and the SoO loop prevention hold simultaneously.
- **3.2.5 — [2 points]** Document why SoO (not just AS_PATH loop detection) is required here: explain the specific scenario where a multi-homed site's own route could otherwise be learned back from a sibling PE and preferred, absent SoO tagging.

### Task 3.3 — Verify and snapshot [7 points]
Confirm Green mVPN Profile 3 multicast state and Blue SoO protection both operate without disrupting Section 1/2 baselines (iBGP, VPNv4 unicast routes, and IS-IS/OSPF adjacencies all still healthy). Snapshot: `LAB2-sec3-access-multicast-soo`.

---

## Section 4 — High Availability [40 points]

### Task 4.1 — TE tunnels with different paths than LAB1 [16 points]
Build MPLS-TE across AS65100's core using **different explicit paths** than any LAB1 TE design (document your chosen paths so grading/review can confirm divergence from LAB1):

- **4.1.1 — [3 points]** Enable MPLS-TE globally and per-interface on all AS65100 core links (R1–R8), and enable the RSVP and IS-IS (or OSPF, N/A here since 65100 is IS-IS) TE extensions needed for TE LSP signaling (wide metrics already enabled in Section 1 supports this).
- **4.1.2 — [5 points]** Build **Tunnel A**: R1 → R8, explicit path **R31→R3→R5→R8** (avoiding R2/R4/R6 entirely) using an explicit-path `next-address` list.
- **4.1.3 — [5 points]** Build **Tunnel B**: R2 → R7, explicit path **R2→R4→R6→R5→R8→R7** (a deliberately longer, diverse path from Tunnel A, sharing no links with Tunnel A except none — verify path diversity by comparing hop lists).
- **4.1.4 — [3 points]** Verify both tunnels are `Up` (`show mpls traffic-eng tunnels brief`), confirm the actual signaled path matches the explicit path requested (no CSPF substitution), and confirm the tunnels carry no VPN traffic yet (that's Task 4.3).

### Task 4.2 — Node protection instead of link protection for FRR [14 points]
LAB1 used **link protection** for Fast Reroute. LAB2 requires **node protection** (protecting against the failure of an entire downstream node, not just the link to it) for Tunnel A:

- **4.2.1 — [4 points]** Identify Tunnel A's protected node: R3 (the first downstream hop with a viable node-bypass, since R1 has only R3 as its first hop and R3→R5 is the segment to protect against R3's *next-hop* R5 failing, OR choose R5 as the node to protect against — in your lab notes, explicitly state which node you are protecting and why that choice requires a bypass tunnel that avoids the protected node entirely, not just the direct link).
- **4.2.2 — [5 points]** Build a backup/bypass tunnel that provides **Next-Next-Hop (NNHOP)** protection around the chosen protected node — the bypass must physically avoid the protected router, merging back onto Tunnel A's path at a point beyond it. Configure this backup tunnel with an explicit path distinct from Tunnel A's primary path.
- **4.2.3 — [3 points]** Enable `mpls traffic-eng fast-reroute` on Tunnel A (node-protection-capable backup assignment, not merely `fast-reroute` link-only) and confirm the headend/midpoint shows the backup tunnel as **NNHOP** (not NHOP) in `show mpls traffic-eng fast-reroute database`.
- **4.2.4 — [2 points]** Test by shutting down the **entire protected node** (not just one interface) and confirm sub-second FRR activates and traffic reroutes via the NNHOP backup, then restore the node and confirm the primary path is reused once it recovers.

### Task 4.3 — VPN traffic over TE + verify HA [10 points]
- **4.3.1 — [4 points]** Force Green VPN traffic between R1 and R8-reachable destinations (or an equivalent AS65100 VPN endpoint pair you have available) onto Tunnel A using autoroute announce or static CBTS/forwarding-adjacency, whichever your design calls for — document the choice.
- **4.3.2 — [4 points]** During the Task 4.2.4 node-failure test, confirm the VPN traffic itself (not just the raw LSP) survives the FRR event with the expected sub-second-to-low-second convergence, and that it resumes using Tunnel A's primary path post-recovery.
- **4.3.3 — [2 points]** Snapshot: `LAB2-sec4-ha-te-node-protection`.

---

## Section 5 — Security: RTBH, Flowspec Concepts, and RSVP Authentication [40 points]

### Task 5.1 — BGP Remotely-Triggered Black Hole (RTBH) [14 points]
- **5.1.1 — [3 points]** Designate R19 (AS65200 RR) as the RTBH trigger router. Create a dedicated static discard route (`ip route <unused-test-prefix>/32 Null0`) to tag for blackholing.
- **5.1.2 — [4 points]** Configure a route-map on R19 that, when a prefix is tagged with a chosen **RTBH community** (e.g., `65200:666`), sets its **next-hop to the discard address** (the Null0-routed test prefix from 5.1.1) using `set ip next-hop <discard-address>`. Redistribute this trigger into iBGP so all AS65200 PEs install the discard route via normal next-hop-based forwarding (no next-hop-self override breaking it).
- **5.1.3 — [4 points]** Ensure every AS65200 PE/ASBR (R12, R14, R15, R16) has `ip route <discard-address>/32 Null0` locally configured, so that once they receive the tagged BGP route with next-hop = discard-address, traffic to the "attacked" destination is dropped **at the edge**, not centrally.
- **5.1.4 — [3 points]** Test: trigger a blackhole for a chosen test destination (e.g., one of R18's addresses) by injecting a tagged route at R19 (or redistributing a manually tagged static route), and verify traffic toward that destination is dropped at the ingress PE (`show ip cef <addr>` should show Null0 / drop adjacency) rather than being forwarded and dropped deep in the core.

### Task 5.2 — BGP Flowspec concepts (design/documentation exercise) [10 points]
Dynamips/IOS 15.2 on 7200 does not support full BGP Flowspec (`address-family ipv4 flowspec`) in practice — this task is a **documented design exercise** demonstrating CCIE-level understanding without requiring working config:
- **5.2.1 — [4 points]** Document a Flowspec rule (in NLRI-component pseudocode or prose) that would match an attack pattern more granular than RTBH's destination-only match — e.g., match `destination-prefix = <addr>/32 AND source-port = 0 AND protocol = UDP AND packet-length > 1400` (a reflective-amplification-style signature), and specify the Flowspec action (`traffic-rate 0` i.e. discard, vs. `redirect to VRF` vs. `traffic-marking`).
- **5.2.2 — [3 points]** Compare and contrast, in 4-6 bullet points, Flowspec vs. RTBH: granularity (5-tuple vs. destination-only), collateral damage (Flowspec can avoid dropping all traffic to a victim, RTBH drops everything), propagation (both via BGP, but Flowspec uses a dedicated AFI/SAFI with validation rules), and operational risk (malformed Flowspec rules can cause major outages — mention `bgp flowspec` validation/dampening safeguards conceptually).
- **5.2.3 — [3 points]** State specifically why this task is documentation-only on this platform (IOS 15.2 7200/Dynamips lacks the Flowspec AFI/SAFI and PBR-via-BGP infrastructure available on IOS-XR/XE 16+/ASR9k) and what minimum platform/software you would need to actually implement it.

### Task 5.3 — RSVP Authentication (replaces LAB1's LDP-only auth scope) [16 points]
LAB1 covered LDP session authentication. LAB2 adds **RSVP (TE) neighbor authentication** on top:
- **5.3.1 — [4 points]** Configure **RSVP neighbor authentication** (MD5-based, via `ip rsvp authentication` with a key or key-chain) on the AS65100 core links carrying the Section 4 TE tunnels (R31↔R3, R3↔R5, R5↔R8, R2↔R4, R4↔R6, R6↔R5, R5↔R8↔R7 segments as applicable to Tunnels A/B).
- **5.3.2 — [4 points]** Enable **RSVP authentication challenge/response** (the anti-replay/life-cycle extension, `ip rsvp authentication challenge`) in addition to the base key-based auth, and explain in notes what the challenge mechanism protects against beyond a plain shared-key MD5 check (replay attacks on RSVP Path/Resv messages).
- **5.3.3 — [4 points]** Verify RSVP neighbors show as authenticated (`show ip rsvp authentication` / `show ip rsvp neighbor`) and that Tunnels A and B (Section 4) remain `Up` with authentication enabled.
- **5.3.4 — [2 points]** Also apply **LDP session authentication** (MD5, via key-chain or plain password per interface/session) on the same AS65100 core links, so both the LDP and RSVP control planes are authenticated — confirming the two authentication mechanisms are configured independently and don't interfere with each other.
- **5.3.5 — [2 points]** Negative-test: deliberately mismatch the RSVP auth key on one link and confirm the RSVP neighbor relationship fails / the TE tunnel using that link goes down, then correct it and reverify tunnel recovery.

### Task 5.4 — Snapshot [0 points]
`LAB2-sec5-security-rtbh-flowspec-rsvp-auth`

---

## Section 6 — Operations: 6VPE (VPN IPv6) and IPv6 PE-CE [45 points]

### Task 6.1 — IPv6 addressing baseline [4 points]
Apply the IPv6 scheme from the topology reference (loopbacks and core links) to all AS65100 and AS65200 routers that will participate in 6VPE (R31, R2 as PEs; R7/R8 as RRs; plus R15 for the IPv6 PE-CE leg). Enable `ipv6 unicast-routing`.

### Task 6.2 — 6VPE (VPN IPv6) — not just 6PE global [18 points]
LAB1 covered only **6PE** (global IPv6 reachability over an IPv4 MPLS core using a single global 6PE label). LAB2 requires **6VPE**, extending Green's L3VPN to carry **customer IPv6 VPN prefixes** (not global-table IPv6):

- **6.2.1 — [4 points]** Extend VRF `GREEN` on R31, R2, and R15 to be dual-stack: add an IPv6 address family under the VRF (`address-family ipv6 vrf GREEN` style, i.e. a VRF with both an RD/RT for v4 and an IPv6-capable instance — on classic IOS this is typically the same VRF with `ipv6` added via `rd`/route-target reuse, since IOS VRF definitions historically needed `vrf definition` with both AFs, or dual `ip vrf`/separate IPv6 VRF table depending on IOS 15.2 feature support — document which VRF model (classic `ip vrf` vs. `vrf definition` with dual AF) your IOS 15.2(4)M11 image supports and use it).
- **6.2.2 — [5 points]** Enable the **VPNv6 address family** in MP-BGP between R31, R2, and R15 (via the RRs, R7/R8) — `address-family vpnv6` — so customer IPv6 VPN routes are signaled with their own RD/RT (reuse Green's existing RT scheme, extended for v6, e.g., same numeric RT value under the vpnv6 AF).
- **6.2.3 — [4 points]** Assign IPv6 addressing inside VRF GREEN on the R31↔R9, R31↔R10, R2↔R10 PE-CE links (new IPv6 subnets you define, consistent with the topology reference's general IPv6 numbering style) and configure **IPv6 eBGP PE-CE** peering (AS 65910) carrying IPv6 customer prefixes into VPNv6.
- **6.2.4 — [3 points]** Verify a Green IPv6 customer prefix originated at R9 (or R10) is visible in `show bgp vpnv6 unicast vrf GREEN` on R2, and that it carries the correct RT/RD and arrived via MP-iBGP VPNv6 (not via 6PE's global table).
- **6.2.5 — [2 points]** Verify end-to-end IPv6 reachability between a Green CE site in AS65100 (R9/R10) and the Green CE at R17/R18 reachable via the AS65200 leg (R15) — confirming VPNv6 label imposition/disposition works across the iBGP RR path, and that this is clearly VPN-scoped IPv6 (private-equivalent per-customer IPv6 space), not global reachability.

### Task 6.3 — IPv6 PE-CE (new in LAB2) [15 points]
Beyond 6VPE's data-plane extension, add a **dynamic IPv6 PE-CE routing protocol** leg (LAB1 had no IPv6 PE-CE at all):
- **6.3.1 — [5 points]** On the R15↔R17 leg (same physical link as Section 2's IPv4 static PE-CE), additionally run **IPv6 eBGP PE-CE** (reuse AS 65910, or choose a distinct customer ASN for the IPv6-only leg and document your choice) carrying R17's IPv6 prefixes into VRF GREEN's VPNv6 address family.
- **6.3.2 — [5 points]** Confirm this coexists cleanly with Section 2 Task 2.3's **static IPv4** PE-CE on the same link/VRF — i.e., IPv4 customer routing is static, IPv6 customer routing is eBGP, both inside the same VRF GREEN, on the same physical R15↔R17 interface (dual-stack with per-AFI different PE-CE protocols is the point of this task).
- **6.3.3 — [5 points]** Verify route redistribution/exchange correctness per address-family: `show bgp vpnv4 vrf GREEN` shows the static-origin IPv4 route; `show bgp vpnv6 vrf GREEN` shows the eBGP-origin IPv6 route; confirm no cross-AFI leaking or confusion, and confirm end-to-end ping works for both IPv4 (static path) and IPv6 (eBGP path) from R17 toward AS65100 Green sites.

### Task 6.4 — Verify Section 6 and final snapshot [8 points]
- **6.4.1 — [4 points]** Confirm 6VPE and the new IPv6 PE-CE leg do not disturb the Section 1 IS-IS/OSPF IPv4 baselines, Section 2 VPNv4 services, or the Section 1 Task 1 IS-IS IPv6-capable links (if overlapping routers are reused, re-verify both v4 and v6 adjacencies are healthy).
- **6.4.2 — [4 points]** Final snapshot: `LAB2-sec6-6vpe-ipv6-pece-COMPLETE`.

---

## Completion Checklist

- [ ] **Section 1 (70 pts):** IS-IS multi-level L1/L2 in AS65100 and AS65300 with route leaking; OSPF stub (Area 20) + NSSA (Area 21) in AS65200 with Type-7/5 translation; LDP across all ASes; BGP WEIGHT-based primary/backup (not LOCAL_PREF); conditional advertisement (exist/non-exist-map); new community scheme; BGP-LU via direct ASBR-to-ASBR + separate internal sessions.
- [ ] **Section 2 (60 pts):** VRF/RD/RT plan built; Green eBGP legs; Green R15↔R17 leg converted to **static** PE-CE; Yellow VRF; Blue VRF over **Inter-AS Option B** (R5↔R11, label swap at ASBR); Red VPLS split into **two VPN-IDs** (RED-NORTH/RED-SOUTH); **ingress policing** QoS (not egress shaping) at 3 PE-CE edges.
- [ ] **Section 3 (45 pts):** Green multicast on **mVPN Profile 3** (BGP MDT + BGP C-multicast signaling, no PIM core signaling); Blue **SoO** built around multi-homed **R25** (not Yellow/R24), with primary/backup path plus loop-prevention proof.
- [ ] **Section 4 (40 pts):** Two TE tunnels on new explicit paths; **node protection** FRR (NNHOP backup) instead of link protection, verified via full-node-shutdown test; VPN traffic forced onto protected tunnel and survives FRR.
- [ ] **Section 5 (40 pts):** BGP RTBH via community-tagged discard route; BGP Flowspec documented as a design/concept exercise (platform-limited); RSVP neighbor authentication (+ challenge/response) on TE-bearing links, alongside independent LDP session authentication; negative-test for both.
- [ ] **Section 6 (45 pts):** **6VPE** (VPNv6 AF, customer IPv6 VPN prefixes in VRF GREEN) — not just global 6PE; new **IPv6 PE-CE** (eBGP) on the R15↔R17 leg running alongside the Section 2 static IPv4 PE-CE on the same VRF/link.

**Total: 300 points.**
