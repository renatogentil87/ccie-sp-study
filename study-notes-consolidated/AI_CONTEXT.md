# AI Context — Renato (rrdog) — CCIE Service Provider Study

## About Me
- Senior TAM at AWS, 15+ years networking experience
- Strong on protocols (BGP, OSPF, IS-IS, MPLS), intermediate Python
- Studying for **CCNP Service Provider (SPCOR 350-501)** first, then **CCIE Service Provider**
- Learn by doing — I lab everything before moving on
- I prefer concise, direct answers. Don't over-explain basics I already know.

## Study Materials
- **CCNP SPCOR 350-501 Official Cert Guide** (Bradley Riapolov, Mohammad Khalil, Cisco Press Dec 2024, ISBN 978-0135324806)
- **MPLS Fundamentals** (Luc De Ghein, Cisco Press) — finished
- Udemy MPLS-TE course — completed
- Building flashcards in interactive style (scenario → I answer → score + explain)

## Study Plan
- CCNP SP first (written exam only — multiple choice, drag-and-drop, simlets, 120 minutes)
- Then CCIE SP (8-hour hands-on lab with two SPs: Emerald/Garnet, IOS-XR + IOS-XE + NSO)
- Local GNS3 lab for all protocol learning (IOS 15.2, Cisco 7200)
- EC2 with IOS-XRv 9000 reserved for later: Segment Routing, SRv6, EVPN, NSO, PCE, Flex-Algo
- IOS-XR syntax translation = 2-3 week effort once I know the protocols cold

## Lab Platform
- **GNS3 Local** on Mac M4 (Apple Silicon)
- **Cisco 7200, IOS 15.2(4)M11** (Dynamips)
- Telnet to localhost ports for console access
- 24 routers, 40 links, **two autonomous systems**

## Lab Topology — 24 Routers, Two ASes

### AS 64512 — SP Backbone West (OSPF Area 0 + LDP)
| Router | Loopback | Role |
|--------|----------|------|
| R1 | 1.1.1.1 | PE |
| R2 | 2.2.2.2 | PE |
| R3 | 3.3.3.3 | PE |
| R4 | 4.4.4.4 | P + **Route Reflector** |
| R5 | 5.5.5.5 | P |
| R6 | 6.6.6.6 | P + **Route Reflector** |
| R7 | 7.7.7.7 | ASBR (inter-AS to R20) |
| R8 | 8.8.8.8 | ASBR (inter-AS to R21) |

### AS 64513 — SP Backbone East (IS-IS Level-2 + LDP)
| Router | Loopback | Role |
|--------|----------|------|
| R13 | 13.13.13.13 | PE |
| R14 | 14.14.14.14 | PE |
| R15 | 15.15.15.15 | P + **Route Reflector** |
| R16 | 16.16.16.16 | P |
| R17 | 17.17.17.17 | P + **Route Reflector** |
| R18 | 18.18.18.18 | P |
| R19 | 19.19.19.19 | P |
| R20 | 20.20.20.20 | ASBR (inter-AS to R7) |
| R21 | 21.21.21.21 | ASBR (inter-AS to R8) |

### Customer Edge Routers
| Router | Loopback | ASN | Connected To | Notes |
|--------|----------|-----|-------------|-------|
| R9 | 9.9.9.9 | 65001 | R1 + R2 | Dual-homed, Customer A, backdoor to R10 |
| R10 | 10.10.10.10 | 65001 | R2 + R3 | Dual-homed, Customer A, backdoor to R9 |
| R11 | 11.11.11.11 | 65002 | R3 | Single-homed, Customer B |
| R22 | 22.22.22.22 | — | R13 | OSPF PE-CE (no BGP), Customer C |
| R23 | 23.23.23.23 | 65001 | R13 | Customer A (Y-side) |
| R24 | 24.24.24.24 | 65001 | R13 + R14 | Customer A (Y-side), dual-homed |
| R25 | 25.25.25.25 | 65002 | R14 | Customer B (Y-side) |

### Addressing
- **Core links:** 10.lower.higher.0/24 (each end uses its loopback last octet as host)
- **X-side PE-CE:** 192.168.x.0/24
- **Y-side PE-CE:** 172.16.x.0/24
- **Backdoor R9↔R10:** 192.168.100.0/24
- **Inter-AS:** R7↔R20 = 10.7.20.0/24, R8↔R21 = 10.8.21.0/24

### VRF Design
| VRF | Customer | ASN | X-side PEs | Y-side PEs | CEs | RT |
|-----|----------|-----|-----------|-----------|-----|-----|
| CUST_A | Customer A | 65001 | R1, R2, R3 | R13, R14 | R9, R10, R23, R24 | 64512:100 / 64513:100 |
| CUST_B | Customer B | 65002 | R3 | R14 | R11, R25 | 64512:200 / 64513:200 |
| CUST_C | Customer C | — | — | R13 | R22 | 64513:300 |

### Key Design Features
- **Two full ASes** with redundant ASBRs (R7/R8 ↔ R20/R21)
- **Customer A spans BOTH SPs** → requires Inter-AS VPN (Options A/B/C)
- **Customer B spans BOTH SPs** → same inter-AS requirement
- **Customer C** = OSPF PE-CE only (no BGP, redistribute into MP-BGP)
- **R9 dual-homed** to R1+R2 → SoO, BGP PIC, Add-Path scenarios
- **R9↔R10 backdoor link** → OSPF sham-link scenario
- **AS X runs OSPF**, AS Y runs **IS-IS** → practice both IGPs
- **RRs on P routers** in both ASes (R4/R6 in X, R15/R17 in Y)
- `as-override` needed on PEs for same-ASN CEs (R9/R10 both AS 65001)

## Current Lab Progress (as of Sep 2026)
- **Lab 01 (MPLS Basics):** ✅ DONE — OSPF+LDP in AS X, IS-IS+LDP in AS Y
- **Lab 02 (MPLS L3VPN):** ✅ DONE — All sections + all 6 CCIE+ Challenges completed
  - SoO on R9 dual-homing ✅
  - Sham-link R1↔R3 with backdoor R9↔R10 ✅
  - RT-Constraint (rtfilter on RRs R4/R6) ✅
  - as-override for same-ASN CEs ✅
  - Maximum-prefix protection ✅
  - BGP PIC Edge ✅
- **Lab 03 (MPLS-TE):** Next up — RSVP-TE tunnels, explicit paths, FRR, autoroute

## Key Concepts I've Mastered
- MPLS label operations (PUSH/SWAP/POP), PHP, explicit-null
- LDP session protection (auto-creates targeted sessions), LDP-IGP sync
- L3VPN: VRF, RD, RT, MP-BGP VPNv4, PE-CE (eBGP + OSPF)
- BGP path selection algorithm (full order including oldest-route before router-ID)
- SoO for dual-homed CEs (prevents routing loops via extended community)
- OSPF sham-link (VRF loopback endpoints, advertised via BGP not OSPF, creates intra-area adjacency across MPLS core)
- DN-bit (prevents OSPF routes redistributed from BGP from being re-redistributed back)
- RT-Constraint (rtfilter on BOTH sides — PE advertises wanted RTs, RR honors the filter)
- BGP PIC = `bgp additional-paths install` (backup pre-installed in CEF, prefix-independent convergence)
- Administrative distance: eBGP=20, OSPF=110, iBGP=200

## Platform Limitations Discovered (IOS 15.2 / 7200)
- `mpls ldp label allocate for host-routes` NOT available (IOS-XR / IOS-XE 16+ only)
- `mpls ldp advertise-labels for <ACL>` works for advertisement filtering but NOT local allocation
- LDP-IGP sync: configure under `router ospf` (not interface); test by removing `mpls ip` on the REMOTE side
- Session protection prevents LDP-IGP sync from triggering max-metric (by design — LDP session stays up)

## 35 Lab Files Written (all mapped to this topology)
Labs cover: MPLS basics, L3VPN, MPLS-TE, AToM/L2VPN, VPLS, Advanced L3VPN, OAM/Protection, BGP path control, IS-IS, 6PE/6VPE, mVPN, Security, QoS, Segment Routing, OSPF multi-area, BGP fundamentals, Carrier Ethernet, HA, Timing, Inter-AS VPN (Options A/B/C), Unified MPLS, BGP Confederations, mLDP/P2MP-TE, L2VPN Interworking, mVPN Advanced, PCE, Flex-Algo, SP Internet Edge/Peering, Advanced RR/Convergence, BGP Flowspec, Python Automation, NETCONF/YANG, SP Automation Advanced.

## CCIE Workbook (INE/Narbik style)
12 workbook files: IS-IS, OSPF, LDP/MPLS, L3VPN, PE-CE Routing, Advanced L3VPN, L2VPN AToM, VPLS, Segment Routing, BGP Path Control, QoS. Progressive scenario-based — configure from scratch, verify, time yourself.

## Flashcards
- MPLS & Segment Routing deck: 35 cards (theory, config reading, output interpretation, drag-and-drop)
- Interactive style: question → I answer → score + explain
- Next deck: Networking (IS-IS/OSPF/BGP, 30% of SPCOR exam)

## Career Context
- Senior TAM at AWS, targeting Principal Network Architect / NDE roles
- Connected with Josh Dean (Sr. PM, Direct Connect team) — positioning as technical bridge between TAM org and DX team
- Writing LinkedIn articles on SP networking topics (RT-Constraint scalability, carrier connectivity behind DX, etc.)
- CCIE SP preferred over Enterprise: scarcity premium, aligns with cloud backbone design principles

## How to Help Me
- I learn by labbing and discussing — quiz me, challenge my understanding
- Don't give me full configs unless I ask — tell me WHAT to configure and HOW to verify
- Flag when something won't work on IOS 15.2 / 7200 (vs IOS-XR / IOS-XE)
- Reference my actual topology (R1-R25, two ASes, specific VRFs/customers) when explaining concepts
- For flashcards: use the interactive scenario style (question → I answer → you score)
