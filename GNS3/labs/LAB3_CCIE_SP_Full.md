# LAB 3 — CCIE SP v5.1 Full Workbook (EVE-NG / IOS-XRv 9000)

**Platform:** EVE-NG, Cisco IOS-XRv 9000 (7.x+), all tasks in **IOS-XR** configuration syntax
**Scope:** Full CCIE SP v5.1 blueprint, including technologies **not supported on Dynamips/IOS 15.2** — Segment Routing (SR-MPLS), SRv6 (uSID), Flex-Algo, SR-TE / SR-ODN, EVPN-VPWS, mVPN, streaming telemetry, NSO.
**Companion labs:** This reuses the **exact same logical topology** (31 routers, 3 ASes, same AS numbers, same router roles, same IP addressing) as the GNS3 Dynamips labs (LAB1/LAB2). Rebuild the identical topology on EVE-NG with **IOS-XRv 9000** nodes in place of the Dynamips c7200 nodes, then configure using IOS-XR syntax.
**Format:** **Tasks only — no solutions, no configuration snippets.** Each task gives you exact parameters (SID indices, locator prefixes, color values, EVI numbers, timers, IPs). You derive and apply the IOS-XR CLI, then verify with `show` commands.

---

## 0. Topology Reference

Identical topology to the Dynamips labs. Same 3 autonomous systems, same roles, same R-numbers — only the node platform (IOS-XRv 9000) and CLI syntax (IOS-XR) change.

### 0.1 Autonomous systems and roles

| AS | IGP | Transport (this lab) | Routers (role) |
|----|-----|----------------------|-----------------|
| **65100** | IS-IS L2 | **SR-MPLS** (prefix-SIDs, TI-LFA, Flex-Algo) | R31(PE), R2(PE), R3(P), R4(P), R5(ASBR), R6(ASBR), R7(RR), R8(RR) |
| **65200** | OSPF Area 0 | **LDP/MPLS** (legacy — LDP-to-SR migration domain) | R11(P), R12(ASBR), R13(P), R14(PE), R15(PE), R16(PE), R19(RR) |
| **65300** | IS-IS L2 | **SRv6** (uSID locators, SRv6 L3VPN) | R20(ASBR/PE), R21(ASBR/PE), R22(PE), R23(PE), R30(RR+PCE) |

### 0.2 Customer edge (CE) routers

| CE | Customer | PE-CE protocol | Attaches to |
|----|----------|----------------|-------------|
| R9, R10 | Green | eBGP AS 65910 | AS65100 PEs |
| R17 | Green | OSPF | R15 (PE-CE OSPF) |
| R18 | Green | eBGP AS 65910 | AS65200 PE |
| R24 | Yellow | eBGP AS 65024 | **dual-homed** R20 + R22 |
| R26 | Yellow | eBGP AS 65024 | AS65300 PE |
| R25 | Blue | eBGP AS 65025 | AS65200/65300 PE |
| R32 | Blue | eBGP AS 65025 | AS65300 PE |
| R27, R28, R29 | Red | VLAN100 L2 (EVPN/VPWS) | R14, R16 |

### 0.3 Loopback0 convention

`150.<AS-last-digit>.<R#>.<R#>/32`:
- AS65100 → `150.1.X.X/32` (R1=150.1.31.31, R8=150.1.8.8)
- AS65200 → `150.2.X.X/32` (R11=150.2.11.11, R19=150.2.19.19)
- AS65300 → `150.3.X.X/32` (R20=150.3.20.20, R30=150.3.30.30)

### 0.4 Inter-AS links

| Link | Subnet | Purpose |
|------|--------|---------|
| R5 ↔ R11 | 10.5.11.0/24 | AS65100 ↔ AS65200 (Inter-AS **Option A**, Green) |
| R6 ↔ R12 | 10.6.12.0/24 | AS65100 ↔ AS65200 |
| R6 ↔ R20 | 10.6.20.0/24 | AS65100 ↔ AS65300 (Inter-AS **Option B**) |
| R12 ↔ R21 | 20.12.21.0/24 | AS65200 ↔ AS65300 (Inter-AS **Option B**) |

### 0.5 IOS-XR-specific addressing you will build in this lab

| Purpose | Scheme |
|---|---|
| SRGB (SR-MPLS, AS65100) | **default** 16000–23999 |
| SRLB (per node, AS65100) | **25000–26000** |
| Prefix-SID index | = **last octet of Loopback0** (R31→1, R5→5, R8→8) |
| Anycast SID (R1 + R2) | Loopback99, **SID 99**, **n-flag-clear** |
| SRv6 locator block (AS65300) | **`FC00:0300:00XX::/48`** per router, **F3216** C-SID (uSID) format (X = router number) |
| SRv6 locator tag | **200** |
| Flex-Algo (AS65100) | **Algo 233** (exclude red affinity), **Algo 234** (exclude blue affinity) |
| SR-TE binding-SID (R31→R5) | **24333** |
| Color communities | **333** (SR-TE autosteer R5→R31), **30** (SR-ODN Platinum/latency, R22 Blue) |
| EVPN-VPWS EVI | **400** (Red, R14↔R16, AC 12) |
| AToM/VPWS VC-ID | **100** (Red, R14↔R16, VLAN100) |
| mVPN default MDT | **239.232.0.1**, data MDT range **239.232.100.0/24**, SSM core **239.232.0.0/16** |

---

## Section 1 — Core Network Infrastructure (25%)

### Task 1.1 — IS-IS L2 (AS65100 + AS65300) via flex-CLI configuration groups

Enable **IS-IS Level-2-only** on **AS65100 (R1–R8)** and **AS65300 (R20, R21, R22, R23, R30)** using IOS-XR **flex-CLI `group` configuration groups** so common per-interface and per-instance stanzas are defined once and applied by `apply-group`, not repeated per interface.

Requirements for both IS-IS domains:
1. Single IS-IS instance, **Level 2 only** on every router and every interface (no L1, no L1/L2).
2. **Wide metrics** — set the default metric to **200 for IPv4** unicast and **400 for IPv6** unicast under the respective address-families.
3. **Single-topology** carrying both IPv4 and IPv6 unicast (do **not** use multi-topology / MT).
4. All core links **point-to-point** (`point-to-point`).
5. **Passive** Loopback0 (and Loopback99 where it exists) in IS-IS.
6. **Disable hello-padding** on all IS-IS interfaces.
7. **LDP IGP-sync** enabled under IS-IS (relevant where LDP runs — carry the knob in the group so it is uniform).
8. **`set-overload-bit on-startup 180`** on **R7** and **R8** (AS65100 RRs) and on **R30** (AS65300 RR) so they do not attract transit during the 180 s post-reload window.
9. **HMAC-MD5 authentication** referencing a **keychain** (hello + LSP). Define the keychain name here and reference it from the IS-IS instance/interface group; the key lifetimes and secret are fully built in **Task 5.2**.

Build at least:
- A **flex-CLI group** for the common IS-IS interface stanza (afi/safi metrics, p2p, hello-padding disable, LDP sync, keychain auth) applied to all core interfaces.
- A **flex-CLI group** for the common IS-IS instance stanza (metric-style wide, single-topology, keychain reference).

Verify: `show isis neighbors`, `show isis adjacency detail`, `show isis database verbose`, `show running-config group`, `show running-config inheritance interface`.

### Task 1.2 — OSPF Area 0 (AS65200)

On **AS65200 (R11, R12, R13, R14, R15, R16, R19)** build a single **OSPF process, Area 0**:
1. All core links **point-to-point** (`network point-to-point`).
2. **MD5 authentication** (OSPF area or per-interface message-digest) on all core adjacencies.
3. **BFD** enabled for OSPF on all core links (tie to the BFD template; interval tuning per Section 4 where applicable).
4. **SPF throttle timers**: `timers throttle spf 50 200 5000` (initial 50 ms, min-hold 200 ms, max-wait 5000 ms).
5. Passive Loopback0.

Verify: `show ospf neighbor`, `show ospf interface`, `show bfd session`, `show ospf`.

### Task 1.3 — LDP / MPLS (AS65200 core)

On the **AS65200** OSPF core enable **MPLS LDP** (this is the legacy transport domain, later migrated to SR):
1. Enable LDP on all AS65200 core interfaces (R11, R12, R13, R14, R15, R16, R19).
2. **Label filtering** — advertise labels for **PE/RR /32 loopbacks only** (`mpls ldp label allocate` / `advertise` ACL limiting to the `150.2.x.x/32` set).
3. **LDP session protection** on all sessions.
4. **LDP-IGP synchronization** (coordinate with OSPF so a link is not used until LDP labels are exchanged).

Verify: `show mpls ldp neighbor`, `show mpls ldp bindings`, `show mpls forwarding`, `show mpls ldp igp sync`.

### Task 1.4 — BGP Unified MPLS (iBGP-LU across all 3 ASes)

Build end-to-end label-switched reachability for PE loopbacks across all three ASes (Unified/Seamless MPLS, BGP-LU / RFC 3107):
1. Enable **`address-family ipv4 labeled-unicast`** on the inter-AS and core BGP sessions as needed.
2. **RRs R7 and R8** reflect BGP-LU for the **AS65100 core**.
3. **P routers R3 and R4** act as **inline Route Reflectors** for BGP-LU within AS65100.
4. **`allocate-label all`** under the IPv4 unicast AF so every IGP prefix (not just loopbacks) gets a label pushed into BGP-LU on the label-stitching nodes.
5. **`next-hop-self`** at every ASBR (R5, R6, R12, R20, R21) for the BGP-LU prefixes so the label stack is re-originated at each AS boundary.

Verify: `show bgp ipv4 labeled-unicast`, `show bgp ipv4 labeled-unicast <PE-loopback>/32`, `show mpls forwarding`, end-to-end `traceroute` across AS boundaries showing a continuous label stack.

### Task 1.5 — Segment Routing on IS-IS (AS65100, SR-MPLS)

Overlay **SR-MPLS** on the AS65100 IS-IS domain (R1–R8):
1. **SRGB** = platform **default (16000–23999)** on all 8 routers.
2. **SRLB** = **25000–26000** on all 8 routers.
3. **Prefix-SID index** = **last octet of Loopback0** (R1 index 1 → label 16001, R5 index 5 → 16005, R8 index 8 → 16008, etc.). Advertise the prefix-SID on each Loopback0 under IS-IS.
4. **Anycast SID** on **R31 and R2**: both advertise a shared **Loopback99** with prefix-SID **index 99**, **`n-flag-clear`** (not a node-SID). This provides an anycast "nearest-of-R31/R2" entry point.
5. **TI-LFA** FRR enabled on **all** IS-IS interfaces on R1–R8 (`fast-reroute per-prefix ti-lfa`).
6. **Microloop avoidance** enabled under IS-IS on all 8 routers.

Verify: `show isis segment-routing label table`, `show mpls forwarding labels 16000 16100`, `show isis fast-reroute`, `show route <loopback>` (confirm TI-LFA backup), `show segment-routing local-block inconsistencies`.

### Task 1.6 — SRv6 (AS65300)

Deploy **SRv6 (uSID / C-SID)** natively in AS65300 (R20, R21, R22, R23, R30):
1. Enable SRv6 with **micro-segment (uSID) F3216** C-SID format.
2. **Per-router locator** = **`FC00:0300:00XX::/48`** where XX is the router number (R20 → `FC00:0300:0020::/48`, R21 → `FC00:0300:0021::/48`, R22 → `FC00:0300:0022::/48`, R23 → `FC00:0300:0023::/48`, R30 → `FC00:0300:0030::/48`).
3. **End behavior** = **`psp-usd`** (penultimate segment pop / ultimate segment decap).
4. **Locator tag 200** on each locator (used by Task 4.2 prefix-priority).
5. Enable **SRv6 logging** of locator/SID allocation.
6. **SRv6 source address** = each router's **Loopback0 IPv6** address.
7. Advertise locators into IS-IS (SRv6 under the IS-IS IPv6 AF).
8. Stand up **SRv6-based L3VPN** transport for the **Yellow** customer (service config detailed in Section 2, Task 2.1 — here enable the SRv6 services locator/AF so Yellow VPNv4/VPNv6 can ride SRv6).

Verify: `show segment-routing srv6 locator`, `show segment-routing srv6 sid`, `show isis database verbose` (locator TLVs), `show cef ipv6 <locator>`, `show segment-routing srv6 manager`.

### Task 1.7 — SR-TE (R1 → R5)

Create an **SR-TE policy** from **R1 to R5** in AS65100:
1. **Dynamic** candidate path optimized on the **TE metric**.
2. **Binding-SID = 24333**.
3. **Autoroute include** for **R5's Loopback100** so IGP shortest-path traffic to that prefix rides the policy.
4. Advertise from **R5 to R1** a service/loopback prefix tagged with **color community 333**, and enable **BGP autosteering (color)** on R31 so color-333 routes resolve onto the R31→R5 SR policy automatically.

Verify: `show segment-routing traffic-eng policy name <...>`, `show segment-routing traffic-eng policy binding-sid 24333`, `show bgp ipv4 unicast <R5-Lo100>` (color ext-comm 333), `show route <R5-Lo100>` (resolved via binding-SID / policy).

### Task 1.8 — SR-ODN (R22, Blue customer)

Configure **On-Demand Next-hop (ODN)** for the **Blue** customer on PE **R22** (AS65300):
1. **ODN color = 30** ("Platinum" / low-latency intent).
2. On-demand SR policy **computed by the PCE** (not locally).
3. **PCE = R30** (AS65300 RR acts as SR-PCE). Configure R22 as a **PCEP client (PCC)** peering to R30.
4. Blue VPN prefixes received with color-30 trigger instantiation of an ODN policy to the egress PE, path computed by R30 using the delay/latency metric.

Verify: `show segment-routing traffic-eng policy color 30`, `show pce ipv4 peer` (on R30), `show segment-routing traffic-eng pcc ipv4 peer` (on R22), `show bgp vpnv4 unicast vrf BLUE` (color 30 ext-comm, ODN-resolved next-hop).

### Task 1.9 — Flex-Algo (R1 ↔ R5)

Define two **IGP Flex-Algos** in AS65100 between **R1 and R5**:
1. **Flex-Algo 233** — exclude links colored with the **red** affinity (`affinity exclude-any red`).
2. **Flex-Algo 234** — exclude links colored with the **blue** affinity (`affinity exclude-any blue`).
3. Assign the **red** and **blue** affinity colors to the appropriate core links (define affinity bit-maps and apply to interfaces so each algo has a distinct constrained path R31↔R5).
4. Advertise **per-algo prefix-SIDs** on R31 and R5 Loopback0 for algo 233 and algo 234 (distinct SID per algorithm).
5. Create **SR policies from R1 to R5 constrained to flex-algo 233 and to flex-algo 234** respectively (constraint `segment-list` / dynamic with `constraints algorithm 233` / `234`).

Verify: `show isis flex-algo`, `show segment-routing traffic-eng policy` (per-algo), `show isis database verbose` (FAD / per-algo prefix-SID), `show route <R5-Lo0> algorithm 233`.

---

## Section 2 — Architecture and Services (25%)

### Task 2.1 — L3VPN (VRF GREEN / YELLOW / BLUE, all ASes)

Build MPLS/SRv6 L3VPN for three customers across all three ASes:
1. Define VRFs **GREEN**, **YELLOW**, **BLUE** on the PEs that attach each customer.
2. **RD/RT scheme** (document and apply consistently):
   - RD = `<PE-ASN>:<customer-id>` (e.g., GREEN = `65100:1`, YELLOW = `65300:2`, BLUE = `65300:3` on AS65300 PEs; mirror per-AS).
   - RT import/export per customer: GREEN `*:1`, YELLOW `*:2`, BLUE `*:3`.
3. **PE-CE eBGP** for Green (R9/R10/R18 AS65910), Yellow (R24/R26 AS65024), Blue (R25/R32 AS65025).
4. **PE-CE OSPF** between **R15 ↔ R17** (Green OSPF site) — redistribute mutually with the VPNv4 backbone (set OSPF domain-id as needed).
5. **`as-override`** on PE-CE eBGP sessions where a CE reuses the same ASN at multiple sites (Green AS65910, Yellow AS65024, Blue AS65025).
6. **Site-of-Origin (SoO)** on the **dual-homed** CE attachments (notably Yellow R24 on R20 and R22) to prevent re-advertisement loops.
7. SRv6-based transport for Yellow in AS65300 (from Task 1.6); MPLS transport for Green/Blue elsewhere.

Verify: `show vrf all`, `show bgp vpnv4 unicast summary`, `show bgp vrf GREEN`, `show bgp vrf YELLOW`, `show route vrf BLUE`, PE-CE `show bgp vrf ... neighbors`, `show ospf vrf GREEN neighbor`.

### Task 2.2 — Inter-AS L3VPN (Option A + Option B)

1. **Option A (back-to-back VRF)** on **R5 ↔ R11** for the **Green** customer: each ASBR holds a GREEN VRF, PE-CE-style eBGP VPN hand-off across the 10.5.11.0/24 link, per-VRF subinterface.
2. **Option B (VPNv4/VPNv6 eBGP, next-hop-self)** on **R12 ↔ R21** (20.12.21.0/24) and **R6 ↔ R20** (10.6.20.0/24): exchange labeled VPN routes directly between ASBRs.
3. On the Option B ASBRs configure **RT retention** (`retain route-target all` or filtered) so VPN routes are not dropped for locally-unconfigured RTs.
4. **`next-hop-self`** on the Option B ASBRs for the VPNv4/VPNv6 routes handed to the remote AS.

Verify: `show bgp vpnv4 unicast rd <...>`, `show bgp vpnv6 unicast`, Option A: `show route vrf GREEN` on both ASBRs; Option B: `show bgp vpnv4 unicast neighbor <ASBR> received-routes` and label stitching across the boundary.

### Task 2.3 — L2VPN AToM / VPWS (Red, R14 ↔ R16)

Build a **point-to-point AToM/VPWS pseudowire** for the **Red** customer between **R14 and R16**:
1. **VLAN 100** attachment circuit on each PE (subinterface, `l2transport`).
2. **VC-ID = 100**.
3. Pseudowire signaled by **targeted LDP** (`neighbor <remote-PE-Lo0>` under the xconnect / l2vpn `pw-class` using MPLS/LDP encapsulation).
4. Keep this circuit **alongside** the EVPN-VPWS circuit of Task 2.4 for direct comparison.

Verify: `show l2vpn xconnect`, `show l2vpn xconnect detail`, `show mpls ldp neighbor` (targeted session R14↔R16), end-to-end CE-to-CE L2 reachability over VLAN100.

### Task 2.4 — EVPN-VPWS (Red, R14 ↔ R16)

Build an **EVPN-VPWS** service for the **Red** customer between **R14 and R16**, in parallel with the AToM circuit:
1. **EVI = 400**.
2. **Attachment circuit identifier = 12** (local AC on each side; source/target AC-ID per EVPN-VPWS).
3. VLAN100 attachment circuit (separate subinterface from the Task 2.3 AToM circuit).
4. Transport: use the **MPLS** label path for the AS65200 PEs (R14/R16 are AS65200). (If you later relocate the service onto AS65300 SRv6 PEs, bind it to the SRv6 locator instead.)
5. Also enable an **EVPN multipoint (ELAN)** instance for Red across R27/R28/R29 attachment points to contrast P2P VPWS vs multipoint EVPN.

Verify: `show evpn evi`, `show evpn evi vpn-id 400 detail`, `show l2vpn xconnect group ... detail`, `show bgp l2vpn evpn`, compare MAC learning behavior vs the AToM circuit.

### Task 2.5 — Multicast VPN (mVPN Profile 0, Yellow, AS65300)

Deploy **Default-MDT mVPN (Profile 0 / Rosen GRE-less, SSM)** for the **Yellow** VRF in AS65300:
1. **PIM-SSM** in the core, SSM group range **239.232.0.0/16**.
2. **BGP auto-discovery** (MVPN SAFI) between AS65300 PEs via R30.
3. **Default MDT = 239.232.0.1**.
4. **Data MDT** range **239.232.100.0/24** (switch high-rate (S,G) streams onto a data MDT).
5. Yellow VRF PIM enabled toward the Yellow CEs (R24/R26).

Verify: `show mrib vrf YELLOW route`, `show pim vrf YELLOW neighbor`, `show mvpn vrf YELLOW database`, `show bgp ipv4 mvpn`, `show mfib vrf YELLOW` (data-MDT switchover).

### Task 2.6 — Hierarchical QoS (egress on R6 ASBR)

Build an **egress H-QoS** policy on **R6** (AS65100 ASBR), applied toward the inter-AS / CE direction:
1. **Parent shaper = 80%** of interface rate.
2. **Voice (EF) = 20% guaranteed** bandwidth (priority / `bandwidth percent 20`) within the shaped parent.
3. **Police = 50%** on a specific CE-sourced traffic class.
4. **DSCP remarking** on a nominated class (e.g., remark scavenger/best-effort on ingress-classified CE traffic).
5. Three-level hierarchy: class policy nested under a parent shaping policy (`service-policy` inside `policy-map`).

Verify: `show policy-map interface <R6-egress>`, `show qos interface <...> output`, offered-load test confirming shape/priority/police counters increment.

---

## Section 3 — Access Connectivity (10%)

### Task 3.1 — SoO loop prevention (dual-homed Yellow CE R24)

**R24** (Yellow, AS65024) is **dual-homed** to **R20 and R22** in AS65300. Routes a CE site originates can be re-learned via the second PE, risking a routing loop / sub-optimal path.
1. Assign a common **Site-of-Origin extended community** to the R24 site on **both** R20 and R22 PE-CE sessions.
2. Confirm SoO-tagged routes are **not re-advertised back** toward the same site (loop broken) and `as-override` interaction is correct.

Verify: `show bgp vrf YELLOW <R24-prefix> detail` (SoO ext-comm present), confirm R24 does not receive its own prefix back from the second PE.

### Task 3.2 — 6PE / 6VPE (AS65100 + Blue IPv6)

1. **6PE** — carry customer/global **IPv6 over the IPv4 MPLS core** in AS65100 (R1–R8): enable `address-family ipv6 labeled-unicast`, IPv6 global prefixes reachable across the IPv4 LSP.
2. **6VPE** — IPv6 L3VPN for the **Blue** customer: enable VPNv6 AF on the Blue PEs, VRF BLUE IPv6 AF, PE-CE IPv6 eBGP (AS65025) for R25/R32.

Verify: `show bgp ipv6 labeled-unicast`, `show bgp vpnv6 unicast vrf BLUE`, `show route vrf BLUE ipv6`, end-to-end IPv6 ping across the IPv4 MPLS core.

---

## Section 4 — High Availability (10%)

### Task 4.1 — BFD for eBGP (R6 ↔ R20)

On the **R6 ↔ R20** inter-AS eBGP session (10.6.20.0/24, Option B boundary):
1. Enable **BFD** for the eBGP neighbor.
2. **min-interval = 800 ms** (tx/rx).
3. **multiplier = 4**.
4. Tie BGP fast-fallover to BFD session state.

Verify: `show bfd session detail` (interval 800 ms, mult 4, state UP), `show bgp neighbor <peer>` (BFD enabled), fail a link and confirm sub-second BGP teardown.

### Task 4.2 — SRv6 backup paths / prefix-priority (AS65300)

In AS65300 IS-IS/SRv6:
1. **`spf prefix-priority critical tag 200`** — prioritize SPF installation of locator prefixes tagged **200** (from Task 1.6).
2. **`fast-reroute per-prefix priority-limit critical`** — compute FRR backups preferentially for critical-tagged (locator) prefixes.
3. Confirm SRv6 locator prefixes get protected first and converge ahead of lower-priority prefixes.

Verify: `show isis fast-reroute summary`, `show route <locator> detail` (backup path present), `show isis spf-log`, confirm tag-200 prefixes install first.

---

## Section 5 — Security (10%)

### Task 5.1 — Management Plane Protection (MPP, control-plane inband)

On representative routers (at minimum one PE and one RR per AS):
1. Configure **`control-plane management-plane inband`** MPP.
2. **Allow SSH and SNMP** only from specific management subnets (define the allowed inband interfaces + peer-filter ACL for the management source subnet).
3. **Deny telnet, XML agent, and TFTP** inbound on the management plane.

Verify: `show management-plane inband interface all`, confirm SSH/SNMP permitted from the management subnet and telnet/XML/TFTP refused.

### Task 5.2 — IS-IS authentication (keychain HMAC-MD5)

Complete the keychain referenced in Task 1.1 on **all** IS-IS instances (AS65100 R1–R8, AS65300 R20/R21/R22/R23/R30):
1. Build the **keychain** with an **HMAC-MD5** key, key-id, and accept/send lifetimes.
2. Apply **hello authentication** and **LSP authentication** via the **flex-CLI configuration groups** from Task 1.1 (do not configure per-interface by hand — carry it in the group).

Verify: `show key chain`, `show isis adjacency detail` (authentication), `show isis database verbose` (LSP auth), mismatch test confirms adjacency drop.

### Task 5.3 — BGP security

On all BGP sessions:
1. **MD5 / TCP-AO password** on every iBGP and eBGP neighbor.
2. **GTSM** (`ttl-security hops 1`) on all **eBGP** sessions (R5↔R11, R6↔R12, R6↔R20, R12↔R21, and PE-CE eBGP).
3. **`maximum-prefix`** limits on all PE-CE and inter-AS sessions (with warning threshold).

Verify: `show bgp neighbor <...>` (password set, TTL security, max-prefix), confirm a spoofed-TTL / over-limit peer is rejected.

---

## Section 6 — Automation (5%)

### Task 6.1 — Streaming telemetry (gRPC dial-out)

On one AS65300 SRv6 PE (e.g., R22):
1. **gRPC dial-out** model-driven telemetry (MDT) streaming **interface statistics** (sensor-path for `Cisco-IOS-XR-infra-statsd-oper` interface counters).
2. Configure a **TPA (Third-Party Application) source/management address** for the telemetry transport.
3. **TLS** on the gRPC session (certificate/trustpoint).
4. Destination-group, sensor-group, and subscription with a sample-interval.

Verify: `show telemetry model-driven subscription`, `show telemetry model-driven destination`, `show grpc`, confirm the collector receives interface-stats samples.

### Task 6.2 — NSO service deployment

Using Cisco **NSO**:
1. **Onboard** an IOS-XRv device (R14) into NSO (add device, auth-group, device-type cli/netconf).
2. **Sync-from** the device.
3. Apply an **XR-base template** (baseline config template).
4. Deploy an **L3VPN service** instance for the **Red** customer via an NSO service package (service model → device config push).

Verify (NSO CLI): `show devices device R14 sync-from`, `commit dry-run`, `show running-config services`, confirm the Red L3VPN service maps to device config on R14.

---

## Completion Checklist

| Section | Weight | Tasks |
|---------|--------|-------|
| 1 — Core Network Infrastructure | 25% | 1.1 IS-IS flex-CLI · 1.2 OSPF · 1.3 LDP · 1.4 BGP-LU · 1.5 SR-MPLS · 1.6 SRv6 · 1.7 SR-TE · 1.8 SR-ODN · 1.9 Flex-Algo |
| 2 — Architecture & Services | 25% | 2.1 L3VPN · 2.2 Inter-AS A+B · 2.3 AToM/VPWS · 2.4 EVPN-VPWS · 2.5 mVPN · 2.6 H-QoS |
| 3 — Access Connectivity | 10% | 3.1 SoO · 3.2 6PE/6VPE |
| 4 — High Availability | 10% | 4.1 BFD · 4.2 SRv6 prefix-priority FRR |
| 5 — Security | 10% | 5.1 MPP · 5.2 IS-IS keychain · 5.3 BGP security |
| 6 — Automation | 5% | 6.1 gRPC telemetry · 6.2 NSO |

**Workflow:** snapshot the EVE-NG lab before each section; complete tasks only (derive the IOS-XR CLI yourself); verify each task with the listed `show` commands before advancing. Solutions live in `LAB3_CCIE_SP_Full_Solutions.md`.
