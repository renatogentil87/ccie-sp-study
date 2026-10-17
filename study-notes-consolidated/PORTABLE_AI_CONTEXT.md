# CCIE SP Study Context — Full AI Reference

**Who:** Renato (rrdog), Senior TAM at AWS, studying for CCIE SP.
**Updated:** October 15, 2026
**Platforms:** GNS3/Dynamips (Cisco 7200, IOS 15.2) + EVE-NG on EC2 (IOS-XRv 9000)

---

## STUDY METHOD

### Flashcard Sessions
- 5 cards per session. ONE card at a time. Wait for my answer before showing next.
- Score /10. Correct mistakes. Add context I missed.
- Cards ≤7 flagged ⚠️ for review — bring back in future sessions.
- Only test topics I've studied (see list below).
- Mix troubleshooting, design scenarios, and concept recall.
- Use my actual lab topology when possible.
- Three card types: **Concept** (explain how something works), **Troubleshooting** (given symptoms, find root cause), **Design** (given requirements, pick best technology).

### Workbook Scoring (CCIE exam standard)
- From memory = full points
- Used ? on router = 2/3 points
- Checked solution/asked for help = 0 points (on exam, no help available)
- Skipped = 0 points

---

## WHAT I'VE STUDIED (fair game for flashcards)

### Mastered (scored 8-10, configured from memory)
- MPLS fundamentals (LDP, label switching, PHP, LFIB)
- L3VPN (VRF, RD, RT, MP-BGP VPNv4, PE-CE BGP + OSPF, VRF isolation, as-override, SoO)
- L2VPN (VPWS/AToM, pseudowires, targeted LDP, VC-ID, control word concept, MTU)
- MPLS-TE (RSVP tunnels, FRR link+node protection, autoroute, forwarding-adjacency, auto-bw, affinity, preemption, MBB/SE)
- BGP (best-path, communities no-export/no-advertise, LOCAL_PREF, AS-PATH prepend, MED, RR design, cluster-ID, Add-Path, PIC, conditional advertisement, maximum-prefix, RT-Constraint concept)
- IS-IS (L2-only, multi-level, metric-style wide, adjacency, LSDB, SPF timers, HMAC-MD5 auth, prefix suppression, overload-bit, BFD, single-topology IPv6)
- OSPF (Area 0, P2P, MD5 auth, SPF timers, BFD, PE-CE with DN-bit/domain-ID/domain-tag, mutual redistribution)
- LDP (label filtering, session protection, LDP-IGP sync, MD5 auth)
- Inter-AS Options A + B (back-to-back VRF, VPNv4 eBGP, RT retention, next-hop-self)
- Carrier Ethernet (G.8032 ERPS, CFM/OAM, MEP/MIP)
- CSC (Carrier Supporting Carrier, 3-label stack)
- 6PE / 6VPE concepts
- BGP-LU / Unified MPLS concepts (send-label on IOS classic, platform limitations)

### Currently Studying (test me, expect some gaps)
- **QoS / DiffServ:** MQC (class-map/policy-map/service-policy), DSCP values (EF, AF classes, CS), policing vs shaping, H-QoS (parent shape + child classify), LLQ/CBWFQ/WRED, DSCP→EXP mapping, uniform vs pipe mode, per-VRF policing, core EXP-based queuing on P routers
- **Multicast:** PIM-SM (RP, shared tree, SPT switchover), PIM-SSM (no RP, IGMPv3, 232/8), PIM Dense Mode (flood-and-prune), BIDIR-PIM (many-to-many, no SPT), IGMP v1/v2/v3, RPF, RP discovery (static, Auto-RP, BSR), MSDP, mVPN Profile 0 + Profile 3
- **Segment Routing SR-MPLS:** prefix-SID, adj-SID, SRGB, SRLB, anycast-SID, globally significant labels, sr-prefer
- **Classic LFA:** inequality check, coverage gaps, why equal fails
- **TI-LFA:** link/node/SRLG protection, P-space/Q-space, tiebreaker preference, 100% coverage
- **Flex-Algo:** multiple SPFs, affinity colors, exclude-any/include-all, single definer, TLV dependency chain, silent failures
- **SR↔LDP coexistence:** mapping server, sr-prefer, migration steps
- **IOS-XR syntax:** commit model, route-policy, VRF under BGP, OSPF area-based, flex-CLI groups

### NOT Yet Studied (DO NOT test)
- SR-TE policies (explicit/dynamic, PCE-initiated, ODN)
- SRv6 (locators, SID types, SRH, uSID) — know basics but not config-ready
- EVPN deep (route types 1-5, IRB, ESI, MAC mobility)
- NSO / NETCONF / RESTCONF / Ansible / gNMI
- Internet Services (CGNAT, NAT64, MAP-T)
- Access Connectivity (Q-in-Q, G.8032 config, MC-LAG config)
- Security advanced (RPKI, AAA, MACsec, LPTS)
- Assurance (NetFlow, TWAMP, Y.1731, SR PM)
- 5G transport design
- BNG

---

## WORKBOOK PROGRESS (GNS3/Dynamips — IOS Classic)

| WB | Topic | Score | Notes |
|----|-------|-------|-------|
| WB01 | IS-IS | 47.5/55 = 86% | SPF timers gap |
| WB02 | OSPF | 34/44 = 77% | After fixes |
| WB03 | LDP | 24/24 = 100% 🔥 | First attempt, no help |
| WB04 | BGP | 71/81 = 88% | Add-Path + conditional adv needed help |
| WB05 | L3VPN | 54/57 = 95% 🔥 | RTC deferred to Inter-AS |
| WB06 | L2VPN AToM | 29/29 = 100% 🔥 | All from memory |
| WB07 | VPLS | excluded | Platform limitation (Dynamips) |
| WB08 | MPLS-TE | 19.5/23 = 85% | IS-IS TE extensions needed help |
| WB09 | Inter-AS | 22/23 = 96% 🔥 | Only missed RT retention command |
| WB10 | QoS | — | Next workbook, studying QoS now |
| WB11 | Multicast | — | Not started |
| WB12-WB15 | 6PE/Security/BGP-LU/IPv6 | — | Not started |

### Trend
```
Early:   77-86% (learning phase)
Mid:     88-95% (concepts solid)
Recent:  96-100% (exam-ready)
```

---

## MY LAB TOPOLOGIES

### GNS3 Local Lab (31 routers, 3 ASes — for WB00-WB15 + LAB1/LAB2)

**AS 65100 (IS-IS L2):** R1(PE,150.1.1.1), R2(PE,150.1.2.2), R3(P,150.1.3.3), R4(P,150.1.4.4), R5(ASBR,150.1.5.5), R6(ASBR,150.1.6.6), R7(RR,150.1.7.7), R8(RR,150.1.8.8)

**AS 65200 (OSPF Area 0):** R11(P,150.2.11.11), R12(ASBR,150.2.12.12), R13(P,150.2.13.13), R14(PE,150.2.14.14), R15(PE,150.2.15.15), R16(PE,150.2.16.16), R19(RR,150.2.19.19)

**AS 65300 (IS-IS L2):** R20(ASBR/PE,150.3.20.20), R21(ASBR/PE,150.3.21.21), R22(PE,150.3.22.22), R23(PE,150.3.23.23), R30(RR,150.3.30.30)

**CEs:** R9/R10(Green,AS65910), R17(Green OSPF), R18(Green,AS65910), R24(Yellow,AS65024), R25(Blue,AS65025), R26(Yellow,AS65024), R27/R28/R29(Red VLAN100), R32(Blue,AS65025)

**Inter-AS:** R5↔R11(Option A, Green), R6↔R12(Option A, 3 VRFs), R6↔R20(Option B), R12↔R21(Option B)

**IP Scheme:** Loopbacks=150.X.Y.Y/32, AS65100 links=10.X.Y.Z/24, AS65200=20.X.Y.Z/24, AS65300=30.X.Y.Z/24, PE-CE=192.168.X.Y/24, IPv6=2001:db8:X:XY::/64

### EVE-NG Lab (same R1-R32 topology, IOS-XRv — for WB16-WB32 + LAB3/LAB4)

Same topology rebuilt on EVE-NG with IOS-XRv routers. SR placement:
- AS65100: SR-MPLS (prefix-SIDs, TI-LFA, Flex-Algo)
- AS65300: SRv6 (uSID locators FC00:0300:00XX::/48)
- AS65200: LDP/OSPF (legacy domain for migration scenarios)

---

## COMPLETE STUDY PLATFORM (33 workbooks, 4 labs, 2 design phases)

### IOS Classic / Dynamips (WB00-WB15)
WB00-Foundation, WB01-IS-IS, WB02-OSPF, WB03-LDP, WB04-BGP, WB05-L3VPN, WB06-L2VPN, WB07-VPLS, WB08-MPLS-TE, WB09-Inter-AS, WB10-QoS, WB11-Multicast, WB12-6PE/6VPE, WB13-Security, WB14-BGP-LU, WB15-IPv6

### IOS-XR / EVE-NG (WB16-WB32)
WB16-SR-MPLS, WB17-SRv6, WB18-SR-TE/ODN, WB19-Flex-Algo, WB20-EVPN, WB21-NSO/Automation, WB22-CSC, WB23-IS-IS-XR, WB24-OSPF-XR, WB25-BGP-XR, WB26-Multicast-XR, WB27-QoS-XR, WB28-InterAS-OptionC+EVPN+IPv6, WB29-Internet-Services, WB30-Access-Connectivity, WB31-Assurance, WB32-Advanced-Security

### Labs (same R1-R32 topology)
LAB1+LAB2: Dynamips (IOS classic topics). LAB3+LAB4: Full CCIE SP (IOS-XR, all topics)

### Design Phase
Phase 1 (38 questions): Atlas Global Networks — IS-IS, OSPF, BGP, MPLS, L3VPN, Inter-AS, QoS, Security, 5G, RON, BNG
Phase 2 (30 questions): Meridian Telecom — SR, SRv6, Flex-Algo, EVPN, automation

---

## COMPLETE FLASHCARD DECK (112 cards)

### MPLS Fundamentals (M1-M5) — all scored 9-10/10
M1: LDP vs SR label significance. M2: Two-label stack L3VPN. M3: PHP. M4: Label allocate host-routes. M5: LDP Session Protection.

### L2VPN / VPLS (L1-L4) — all scored 9-10/10
L1: VPWS vs VPLS + MEF. L2: Split-horizon. L3: BGP VPLS label block. L4: H-VPLS tiers.

### Carrier Ethernet (E1-E5) — scored 8-10/10
E1-E5: G.8032, MEP/MIP, CFM levels, OAM.

### Deep Cards (D1-D5)
D1: CSC 3-label stack (7⚠️). D2: Two VPNv4 same RT diff RD (9). D3: BGP best-path router-id (10). D4: RR path hiding (3⚠️⚠️). D5: PW DOWN but LDP up (8).

### MPLS-TE (TE1-TE9)
TE1: FRR scenario (6⚠️). TE2: Autoroute announce (9). TE3-TE9: Not yet tested.

### L3VPN Advanced (V1-V8)
SoO, sham-link, DN-bit, RT-Constraint, as-override, PIC, admin distance, LDP-IGP sync — not formally tested.

### SR-MPLS (SR1-SR10) — scored 6-9/10
SR1: SR PHP (8). SR2: SRv6 SID types (6⚠️). SR3: Option C BGP-LU (7⚠️). SR4-SR10: new cards.

### Classic LFA (CL1-CL5) — new
CL1: What is LFA. CL2: Inequality formula. CL3: LFA vs TI-LFA. CL4: Coverage gap. CL5: Zero-segment.

### TI-LFA (TL1-TL5) — new
TL1: Basics. TL2: P-space/Q-space. TL3: Link vs node. TL4: TI-LFA vs RSVP FRR. TL5: SRLG.

### Flex-Algo (FA1-FA10) — new
FA1-FA10: Concepts, definer, affinity, participation, silent failures, TLV chain, troubleshooting.

### Design Scenarios (DS1-DS10) — CCIE exam format, multiple choice
DS1-DS10: TI-LFA, Option C, 5G slicing, LDP→SR migration, as-override, sham-link, CSC, Flex-Algo+ODN, EVPN MH, SR SID missing.

### Troubleshooting (TS1-TS8)
TS1-TS8: eBGP 0 prefixes, IS-IS IPv6 missing, SR SIDs missing, Flex-Algo affinity, infinity metric, LFA no backup, Option C next-hop, mapping server conflict.

### QoS — DiffServ / MQC (QS1-QS10) — NEW Oct 2026
QS1: MQC 3-step pattern. QS2: DSCP EF value/usage. QS3: AF naming (AFxy). QS4: Policing vs shaping. QS5: H-QoS concept. QS6: P router matches EXP not DSCP. QS7: WRED vs tail-drop. QS8: Uniform vs pipe vs short-pipe. QS9: DSCP→EXP mapping. QS10: LLQ.

### Multicast — PIM / IGMP (MC1-MC10) — NEW Oct 2026
MC1: Sparse vs Dense mode. MC2: RP role in PIM-SM. MC3: PIM-SSM (no RP, IGMPv3). MC4: RP discovery mechanisms. MC5: IGMP versions. MC6: SPT switchover. MC7: RPF. MC8: MSDP. MC9: BIDIR-PIM. MC10: mVPN Profile 0 vs 3.

### Priority Review Queue (updated Oct 15, 2026)
1. D4 — RR path hiding (3/10)
2. TE1 — FRR facility backup (6/10)
3. SR2 — SRv6 SID types (6/10)
4. SR3 — Option C BGP-LU (7/10)
5. D1 — CSC 3-label stack (7/10)
6. QS1-QS10 — QoS (new, needs first session)
7. MC1-MC10 — Multicast (new, needs first session)
8. QS6 — P router EXP matching (exam trap)
9. QS8 — Uniform vs pipe (design phase favorite)
10. MC3 — PIM-SSM (exam favorite)
11. MC7 — RPF (#1 multicast troubleshooting concept)

---

## KEY CONCEPTS

### IOS-XR Gotchas
- eBGP requires route-policy (no policy = 0 routes)
- No `activate` — address-family under neighbor = activation
- VRF under BGP is its own block (`vrf CUST_A`)
- OSPF: no `network` command, interfaces under `area 0` block
- IS-IS: `point-to-point` not `network point-to-point`
- `show bgp` not `show ip bgp`

### IOS Classic Gotchas (from WB01-WB09 labs)
- IS-IS SPF: `spf-interval <MAX> <initial> <hold>` (max FIRST)
- OSPF SPF: `timers throttle spf <initial> <hold> <MAX>` (initial FIRST) — OPPOSITE ORDER
- `match community` requires a community-list reference, not the community value directly
- `ip bgp-community new-format` needed to see ASN:NN instead of decimal
- BGP `send-label` = IOS classic equivalent of `address-family ipv4 labeled-unicast`
- `no bgp default route-target filter` required on Option B ASBRs
- `mpls bgp forwarding` needed on inter-AS interfaces for BGP-LU
- Option C BGP-LU: IOS classic RRs can't allocate labels for reflected routes (platform limitation)
- VPLS `l2 vfi manual` not supported on 7200 — only `point-to-point`
- DS-TE, FRR state verification, control word — not supported on Dynamips
- BFD doesn't work on Dynamips (configure for learning only)
- SoO needs `clear ip bgp <CE> soft in` after adding — stamps on receive, not retroactive

### QoS Key Concepts
- MQC: class-map → policy-map → service-policy (always 3 steps)
- PE ingress: match DSCP. P router core: match MPLS EXP. PE egress: match DSCP (uniform) or EXP (pipe)
- EF=46 (voice, LLQ priority). AF41=34 (video). AF21=18 (business). AF11=10 (bulk). 0=best-effort
- AF naming: AFxy — x=class(1-4), y=drop(1-3). Higher class=more important. Higher drop=dropped first
- DSCP→EXP: EXP = top 3 bits of DSCP. EF→5, AF41→4, AF21→2, CS6→6, 0→0
- H-QoS: parent shapes total, child classifies within. Used on ASBR egress + PE-CE egress
- WRED: selective drop by precedence BEFORE queue full. Avoids TCP global sync
- Uniform mode (default): DSCP↔EXP copied both ways. Pipe: EXP independent, DSCP preserved

### Multicast Key Concepts
- PIM-SM: explicit join, RP required, shared tree (*,G) then SPT (S,G) switchover
- PIM-SSM: no RP, receivers specify source+group (IGMPv3), only (S,G), range 232/8
- BIDIR-PIM: no SPT switchover, only (*,G), traffic flows both ways on shared tree, many-to-many
- RPF: "did packet arrive on the interface toward the source?" Yes=forward, No=drop
- MSDP: SA messages between RPs in different ASes for inter-domain multicast
- mVPN Profile 0: PIM/GRE default MDT + data MDT + BGP auto-discovery
- mVPN Profile 3: mLDP P2MP core + BGP C-multicast signaling (no PIM in core)

### Inter-AS (from WB09 — all configured and working)
- Option A: VRF on ASBRs, per-VPN sub-interfaces. No MPLS at boundary. Simple, doesn't scale.
- Option B: VPNv4 eBGP between ASBRs. Single session all VPNs. `retain route-target all` required. VPN label at boundary.
- Option C: BGP-LU + multihop VPNv4 RR-RR. End-to-end labeled. ASBR lightweight. (Platform limited on Dynamips)
- My lab design: R5↔R11 Option A (Green), R6↔R12 Option A (3 VRFs), R6↔R20 Option B, R12↔R21 Option B

### VPN Design
- RD = uniqueness (per-SP). RT = import/export (per-customer, same across SPs)
- OSPF PE-CE: mutual redistribution required (ospf→bgp + bgp→ospf). DN-bit prevents loop.
- Domain-ID: matching = Type-3 (inter-area), mismatched = Type-5 (external) on CE
- Domain-tag: backup loop prevention for Type-5 LSAs
- SoO: stamps on EVERY PE-CE neighbor facing the same dual-homed site. Same SoO value.
- `no bgp default route-target filter` on transit ASBRs (Option B) — keeps VPNv4 for VRFs not locally present

### Segment Routing
- Prefix-SID: global, SRGB+index. Same label everywhere.
- Adj-SID: local, auto-allocated. Protected + unprotected per adjacency.
- TI-LFA: post-convergence SPF + segment labels. ~100% coverage. Replaces RSVP-TE FRR.
- Flex-Algo: multiple SPFs, affinity, single definer. THREE layers for steering: algo topology + BGP color + SR-TE policy.
- Mapping server: virtual SIDs for LDP destinations. Temporary bridge during migration.

---

## LINKEDIN ARTICLE SERIES

### AWS Networking Fundamentals (7 weeks)
- Week 1: VPC Fundamentals ✅
- Week 2: Security Groups vs NACLs ✅
- Week 3: VPC Connectivity (Peering, TGW, PrivateLink) ✅
- Week 4: Direct Connect & VPN (next)
- Week 5-7: Route 53, Load Balancing, Troubleshooting

### SP Technical Articles (LinkedIn)
- Inter-AS MPLS VPN Options A/B/C ✅
- SRv6, IS-IS, Flex-Algo (3 articles), SP Automation, MPLS troubleshooting, LDP→SR mapping server, and more

---

## STUDY GUIDANCE FOR AI
- **Don't give configs when I ask to try** — encourage using `?` on router first
- **Score honestly** — 0 points if I needed help (CCIE exam has no help)
- **Two lab platforms:** GNS3 (IOS classic, WB00-15) + EVE-NG (IOS-XR, WB16-32)
- **Workbook format:** Tasks only in workbooks. Solutions in separate files.
- **Verify-after-each-task habit:** I'm building the discipline to check config → check state → check end-to-end before moving on
