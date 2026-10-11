# About Me — Networking & Certification Context

## Professional Profile
- Senior TAM at AWS, 15 years networking experience
- Strong on protocols (BGP, OSPF, MPLS) but intermediate on Python
- Building a Principal Network Architect portfolio project
- Learn by doing, not by reading code

## Certification Goal
- Target: CCNP Service Provider (SPCOR 350-501 + SPVI 300-515)
- Then: CCIE Service Provider (lab exam)
- NOT pursuing CCIE Enterprise — SP is the better fit for my career trajectory

## SPCOR 350-501 Exam
- Format: 90-110 questions, 120 minutes, multiple choice + drag-and-drop + simlets
- Passing: ~825/1000
- Cost: ~$400 USD
- 5 domains: Architecture (15%), Networking (30%), VPN Services (25%), Security (15%), Infrastructure Services (15%)

## Study Materials
- MPLS Fundamentals (Cisco Press) — almost finished
- Udemy MPLS-TE course — in progress
- CCNP SPCOR 350-501 Official Cert Guide (Dec 2024 edition, ISBN 978-0135324806) — TO BUY
- Boson doesn't exist for SPCOR — use Pearson Test Prep (included with OCG book)
- Udemy practice questions as supplement
- Books for AFTER SPCOR: BGP Design & Implementation, MPLS in the SDN Era, EVPN-VXLAN Fabric Design, Segment Routing Part I & II

## Study Plan
- 2-3 hours/day (1 hour videos/reading + 1 hour labs)
- Weekends: 3-4 hours (longer lab sessions)
- Aggressive timeline: 10-12 weeks (~3 months) for SPCOR
- Study flow: OCG chapter → understand → Lab it → Solidify

## Lab Platform
- GNS3 Local on Mac M4 (Apple Silicon)
- Only Dynamips/Cisco 7200 IOS 15.2 works on GNS3
- Telnet to localhost ports (R1=5009, R2=5000, R3=5001...R20=5019)
- No SSH, no VM — telnet only
- EVE-NG on AWS c5.metal planned for SR/EVPN/NETCONF (IOS-XRv, NXOSv)

## Lab Topology — 20 Routers
- 4 PEs: R2, R8, R17, R18 (AS 64512)
- 9 P routers: R3, R4, R5, R6, R7, R13, R14, R15, R16
- 7 CEs: R1(AS65001), R9(AS65001), R10, R11(AS65011), R12(AS65012), R19(AS65019), R20(AS65020)
- Core: OSPF area 0 + LDP + MPLS
- Label ranges per router (R2=200-299, R3=300-399, etc.)
- Route Reflectors: R3 and R7

## Current Lab Progress (as of Aug 8, 2026)
- Lab 1 (MPLS Basics): DONE
- Lab 2 (L3VPN): Sections 1-4 DONE + CCIE+ Challenges 1-6 ALL DONE
- Lab 3 (MPLS TE): Sections 1-6 IN PROGRESS. Completed: dynamic tunnels, explicit paths, primary/backup failover, bandwidth admission, preemption, affinity/link-coloring, auto-bandwidth, FRR (link+node protection), DS-TE sub-pool. Remaining: forwarding adjacency, QoS tunnel selection (PBR approach needed due to IOS 15.2 limitation), inter-area TE loose hops
- Labs 4-19: Not started (but all lab workbooks created and ready)
- Labs 20-24: NEW — workbooks created Aug 8 (OSPF Advanced, BGP Fundamentals, Carrier Ethernet, HA, Timing)
- Labs 25-32: IN PROGRESS — workbooks being created (Inter-AS B/C, Confederations, mLDP, L2VPN Interworking, mVPN Profiles, PCE, Flex-Algo, Multi-domain SR-TE)

## Troubleshooting Lab Progress (as of Aug 8, 2026)
- ts_lab_2 (MPLS SP Core): Tickets 1-11 DONE (27/64 points). Tickets 12-20 remain. Faults injected on live GNS3 topology.
- ts_lab_1 through ts_lab_9: All rewritten for 20-router topology. 180 total tickets ready.
- Injection method: AI connects via raw socket telnet to inject faults on demand

## Active Lab State (GNS3)
- Topology A running: OSPF + LDP + L3VPN + TE (Tunnel0 on R2→R8 with FRR)
- TE south core (R13-R18): needs TE enablement (Task 1 of Lab 3 incomplete for south)
- Troubleshooting faults: Tickets 12-20 partially injected from ts_lab_2
- VPN customers: Customer_A (R1↔R9), Customer_B (R12↔R11), Customer_D (R19), Customer_E (R20)

## Key Technical Discoveries (Aug 2026)
- ECMP with one unlabeled path = intermittent VPN blackhole (Ticket 4 lesson)
- Explicit paths with loose hops: CSPF on intermediate router may choose wrong link (use interface IPs or strict hops)
- FRR label stack: swap + push (not just push) — R3 swaps 332→433 then pushes 630 for backup tunnel
- DS-TE: sub-pool reservations count against global pool; global tunnels can use unused sub-pool BW
- Auto-bandwidth: measures tunnel interface output rate, not physical link utilization
- Link Manager: one-time gate at RSVP signaling only, does not monitor real-time traffic
- IOS 15.2 limitations: no EXP-based tunnel selection (CBTS), no telnetlib in Python 3.14+
- Administrative-weight: TE-specific metric override (CSPF uses it instead of IGP cost)

## Exam Practice Files Created
- CONFIG_EXAM_PRACTICE.md: INE-style scored workbook (100 pts, Sections 1-5)
- AUTOMATION_EXAM_PRACTICE.md: removed (content moved to lab_19)
- 4 additional INE-style exams: IN PROGRESS (Networking, VPN Services, Security, Infrastructure)

## Books Being Read
- MPLS Fundamentals (Luc De Ghein) — Chapter 10 (AToM) and 11 (VPLS) completed Aug 8
- Next: CCNP SPCOR 350-501 Official Cert Guide (to buy)
- Recommended: Juniper MX Series (O'Reilly) for router architecture understanding
- Reference: xrdocs.io for NCS 5500 NPU/forwarding architecture

## Lab Curriculum — 18 Configuration Labs
- Labs 1-15: GNS3 (Cisco 7200, IOS 15.2)
- Labs 16-18: EVE-NG (IOS-XRv, NXOSv)
- Troubleshooting labs: separate folder, 10+ scenarios per config lab, AI configures broken state via telnet

## Project Structure
- Main project: /Users/rrdog/Desktop/aws-network-operations-platform/
- Study notes: docs/study-notes/ccnp-sp/
- Configuration labs: docs/study-notes/ccnp-sp/labs/configuration/
- Troubleshooting labs: docs/study-notes/ccnp-sp/labs/troubleshooting/
- Study guide: docs/study-notes/ccnp-sp/LAB_STUDY_GUIDE.md
- Router inventory: python/netops/configurator/inventory.yaml

## Key Technical Decisions
- Labs are progressive (build on each other, don't restart)
- Topology A (Labs 1,2,3,7,8,12,13,14,15) — stacks non-destructively
- Topology B (Labs 4,5,11) — L2VPN, separate GNS3 project
- Topology C (Lab 6) — Advanced VPN with RT changes
- Topology D (Lab 10) — IS-IS replaces OSPF
- CCIE+ Challenges in each lab go beyond CCNP into CCIE-SP territory

## SP Career Context
- CCIE SP preferred over CCIE Enterprise because: scarcity premium, AWS role aligns with SP thinking, cloud backbone = SP design principles
- Target roles: Principal Network Architect at SP/hyperscaler/cloud provider ($300K+)
- Portfolio: MPLS/BGP lab + AWS networking automation + Terraform infrastructure

## Key Discussions and Decisions
- MPLS-TE is still used by Tier-1 ISPs but being replaced by SR-TE
- Large ISPs don't use preemption in steady state (all tunnels same priority 7/7)
- Forwarding adjacency rarely used in production (autoroute preferred)
- QoS still relevant (mobile backhaul, 5G voice, IPTV)
- EXP-based tunnel selection or PBR used to steer voice vs data into separate tunnels
- Route Reflectors typically placed on P routers (in data path) — acceptable because RR failure = IGP reconvergence, not VPN failure
- OSPF sham-link requires VRF loopback endpoints advertised via BGP (not OSPF)
- IOS PE rejects vpnv4 routes at BGP level if no matching VRF/RT configured (not just VRF import filtering)
- LSA Type 10 (opaque) carries 8 unreserved bandwidth values (one per setup priority)
- Setup priority X can preempt tunnels with hold priority > X (strictly greater than)
