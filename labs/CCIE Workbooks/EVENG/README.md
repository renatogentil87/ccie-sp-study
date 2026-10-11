# CCIE SP v5.1 Workbooks — EVE-NG (IOS-XRv 9000)

> 🔴 **CCIE Service Provider v5.1 lab-prep track — production IOS-XR platform**

**Platform:** GNS3 on EC2 `m8i.24xlarge` (384 GB RAM, 96 vCPU) running **IOS-XRv 9000**
**Topology:** 3-ISP — **Emerald AS 65100** + **Gold AS 65300** + **Garnet AS 65200** — 20 XRv routers + 9 CEs
**Reference:** See [`00_topology_reference.md`](00_topology_reference.md) for the complete node / link / IP mapping *(base topology also captured in [`../00_EVENG_Topology.md`](../00_EVENG_Topology.md))*
**NIC Mapping:** `NIC2 = Gi0/0/0/0` · `NIC3 = Gi0/0/0/1` · `NIC4 = Gi0/0/0/2` · `NIC5 = Gi0/0/0/3`

---

## 1. Workbook Index (WB01–WB16)

| WB | Workbook | Domain | Wt % | Topics | Key Nodes |
|----|----------|--------|------|--------|-----------|
| **WB01** | [IS-IS Foundation](WB01_ISIS_Foundation.md) | D1 Core Routing | 25% | L2-only IS-IS, wide metrics, config groups, IPv6 single-topology, overload bit, LDP-IGP sync, HMAC-MD5 auth, BFD, mesh-groups, convergence tuning | All 18 core routers (Emerald 49.0001 / Garnet 49.0002 / Gold 49.0003) |
| **WB02** | [MPLS LDP + Unified MPLS](WB02_MPLS_LDP_Unified_MPLS.md) | D1 Core Routing | 25% | LDP discovery, targeted LDP, transport-address, label filtering, session protection, GR, MD5, BGP-LU, inter-AS label stitching | Emerald PEs/ASBRs, E-R6↔G-R4, G-R5↔Gar-R7 |
| **WB03** | [BGP Advanced](WB03_BGP_Advanced.md) | D1 Core Routing | 25% | iBGP + RR design (all 3 ASes), eBGP inter-AS, Option C multihop VPNv4, path manipulation (LP/prepend/MED/community), Add-Path, PIC Edge, AIGP, RT-Constraint | RRs: E-R5 6.6.6.6 / G-R4 24.24.24.24 / Gar-R6 16.16.16.16 |
| **WB04** | [Segment Routing (SR-MPLS)](WB04_Segment_Routing_SR_MPLS.md) | D1 Core Routing | 25% | SR under IS-IS, SRGB 16000–23999, prefix/adjacency-SIDs, TI-LFA, SR-TE (explicit/dynamic/Gar-R6/ODN), Flex-Algo, LDP→SR migration | Garnet (SR native), Emerald (migration), Gar-R6 16.16.16.16 |
| **WB05** | [SRv6](WB05_SRv6.md) | D1 Core Routing | 25% | SRv6 locators, End/End.X/End.DT4/DT6/DX4/B6 SID behaviors, L3VPN over SRv6, SRv6-TE, SRH, inter-domain Option-B stitching, uSID | Gold (SRv6 native), G-R1/G-R2, G-R4 |
| **WB06** | [MPLS Traffic Engineering](WB06_MPLS_TE.md) | D1 Core Routing | 25% | RSVP-TE tunnels, explicit-path, autoroute announce, FRR, priority, auto-bw, DS-TE (MAM/RDM), affinity, forwarding-adjacency | Emerald: E-R1→E-R3→E-R2 (10.1.x.0/24 core) |
| **WB07** | [L3VPN](WB07_L3VPN.md) | D2 Architectures & Services | 25% | VRF/RD/RT, PE-CE (eBGP/OSPF/sham-link/SoO), MP-BGP VPNv4, Inter-AS Options A/B/C, RT-Constraint, PIC Edge, label modes | E-R1/E-R2 (Emerald), G-R1/G-R2 (Gold), Gar-R1 (Garnet) |
| **WB08** | [L2VPN — VPWS / VPLS](WB08_L2VPN_VPWS_VPLS.md) | D2 Architectures & Services | 25% | VPWS/AToM p2p PW, backup/static PW, preferred-path, VPLS full-mesh, H-VPLS, BGP-VPLS (Kompella), EVPN-VPLS | E-R1↔E-R2 PW, N-PE/U-PE tiers |
| **WB09** | [EVPN](WB09_EVPN.md) | D2 Architectures & Services | 25% | Route Types 1–5, EVPN-VPWS, multi-homing (all-active/single-active/DF), IRB (sym/asym), inter-AS EVPN, MAC mobility | Gar-R1/Gar-R2 (dual-homed), CE5/CE7, Gar-R6 RR 9.9.9.9 |
| **WB10** | [Multicast + mVPN](WB10_Multicast_mVPN.md) | D2 Architectures & Services | 25% | PIM-SM/SSM, RP (static/Auto-RP/BSR), MSDP, mVPN Profiles 0/3/11/12/14, data MDT, SR-MVPN Tree-SID, inter-AS mVPN | Emerald RP 6.6.6.6, Garnet RP 23.23.23.23 |
| **WB11** | [QoS / DiffServ](WB11_QoS_DiffServ.md) | D2 Architectures & Services | 25% | XR QoS model, class-map/policy-map, DSCP→TC marking, policing, queuing/scheduling, shaping, end-to-end SLA | E-R1→core→Gar-R1 path |
| **WB12** | [6PE / 6VPE / Dual-Stack](WB12_6PE_6VPE_DualStack.md) | D2 Architectures & Services | 25% | 6PE (IPv6 over IPv4 MPLS, send-label), 6VPE (VPNv6), dual-stack VRF, End.DT46, IPv4-mapped next-hop | E-R1 1.1.1.1 / E-R2 2.2.2.2, CE1/CE3 |
| **WB13** | [Access Connectivity](WB13_Access_Connectivity.md) | D3 Access Connectivity | 10% | 802.1Q/Q-in-Q (802.1ad), VLAN translation, G.8032 ERPS, MC-LAG + ICCP, BNG (IPoE/PPPoE, AAA, CUPS) | E-R1–CE1–CE2–E-R2 ring, Bundle-Ether MC-LAG |
| **WB14** | [High Availability & Convergence](WB14_High_Availability_Convergence.md) | D4 High Availability | 10% | NSR/GR(NSF)/SSO, IGP SPF/LSP throttle, BFD 50ms, LDP-IGP sync, BGP PIC Edge/Core, RSVP-TE FRR vs TI-LFA | All core (Emerald convergence lab) |
| **WB15** | [Security](WB15_Security.md) | D5 Security | 10% | LPTS vs CoPP, IGP HMAC-MD5, BGP TCP-AO/GTSM/RPKI/RTBH/Flowspec, bogon + AS-PATH filtering, MACsec | E-R6↔Gar-R7 (MACsec), all 3 SPs |
| **WB16** | [Automation & Assurance](WB16_Automation_Assurance.md) | D6 Automation & Assurance | 20% | NETCONF/YANG, gRPC/gNMI telemetry (MDT), NSO (FASTMAP/L3VPN/rollback/compliance), Python (netmiko/TextFSM), ZTP, SNMPv3/IPFIX | All 20 routers + NSO VM |

---

## 2. CCIE SP v5.1 Exam Domain Mapping

| Domain | Weight | Description | Workbooks |
|--------|--------|-------------|-----------|
| **Domain 1** | **25%** | **Core Routing** — IS-IS, LDP, BGP, SR-MPLS, SRv6, MPLS-TE | **WB01 – WB06** |
| **Domain 2** | **25%** | **Architectures & Services** — L3VPN, L2VPN, EVPN, Multicast/mVPN, QoS, 6PE/6VPE | **WB07 – WB12** |
| **Domain 3** | **10%** | **Access Connectivity** — L2 access, Q-in-Q, G.8032, MC-LAG, BNG | **WB13** |
| **Domain 4** | **10%** | **High Availability** — NSR/GR/SSO, IGP/BGP convergence, FRR/TI-LFA | **WB14** |
| **Domain 5** | **10%** | **Security** — control-plane protection, IGP/BGP security, RPKI, MACsec | **WB15** |
| **Domain 6** | **20%** | **Automation & Assurance** — NETCONF/YANG, telemetry, NSO, ZTP, Python | **WB16** |
| | **100%** | | **WB01 – WB16** |

---

## 3. Recommended Study Order

Progressive — each workbook builds on the transport and control-plane primitives established by the previous ones. Do them **in order**:

```
Domain 1 — Core Routing (the foundation everything else rides on)
  WB01 (IS-IS)  →  WB02 (LDP)  →  WB03 (BGP)  →  WB04 (SR-MPLS)  →  WB05 (SRv6)  →  WB06 (MPLS-TE)
        │                                                                              │
        └── IGP + label transport + BGP overlay must be solid before services ────────┘
                                          ↓
Domain 2 — Architectures & Services (overlays on top of the transport)
  WB07 (L3VPN)  →  WB08 (L2VPN)  →  WB09 (EVPN)  →  WB10 (Multicast/mVPN)  →  WB11 (QoS)  →  WB12 (6PE/6VPE)
                                          ↓
Domain 3–6 — Edge, resilience, hardening, and operations
  WB13 (Access)  →  WB14 (HA)  →  WB15 (Security)  →  WB16 (Automation)
```

**Why this order:**
- **WB01→WB03** establish the IGP, label distribution, and BGP overlay that *every* later service depends on.
- **WB04→WB06** layer the modern transport planes (SR-MPLS, SRv6, RSVP-TE) that L3VPN/L2VPN/EVPN steer over.
- **WB07→WB12** deploy customer services — they assume the transport from Domain 1 is already converged.
- **WB13→WB16** add access edge, resilience, security, and automation once services exist to protect and operate.

---

## 4. Workbook Format (ccielabpass DOO style)

Every task in every workbook follows the **DOO (Detailed Output Objective)** pattern used by ccielabpass:

1. **Question / Task** — the requirement stated the way the CCIE lab presents it (bulleted objectives, point value, node scope). Read it as if it were an exam item.
2. **Solution** — an explanation of *why*, followed by the complete, copy-pasteable **IOS-XR configuration** for each node involved.
3. **Verification** — the specific `show` commands to run and **what "good" looks like** in the output, so you can prove the objective is met without guessing.

```
### Task X.Y — <objective>  (N points)

**Question**
- <bulleted requirement>

**Solution**
<why it works>
```
<IOS-XR config per node>
```

**Verification**
```
<show commands>
```
<what good output looks like>
```

Most workbooks also close with a **Final Validation Checklist** and, where relevant, **CCIE Challenge Tasks** and an **exam-traps** section.

---

## 5. Reference Materials

These workbooks are study aids built by cross-referencing the following materials. All credit for the underlying methodology and content belongs to their authors:

- **ccielabpass** — DOO-style workbook format (Question → Solution → Verification) and lab task framing.
- **INE CCIE SP v5 (INEv5)** — topology conventions, scenario progression, and the INE/Narbik "Scenario → Configure → Verify → Time yourself" cadence.
- **Quick Guide to CCIE SP v5.1** — exam blueprint domain weightings and topic mapping used in the tables above.

> These are personal study notes for exam preparation and are not affiliated with or endorsed by Cisco or the reference authors.

---

## 6. Transport Technology per ISP

Each of the three ISPs runs a **different** data-plane transport so the topology exercises all CCIE SP v5.1 transport technologies and their inter-domain handoffs:

| ISP | AS | Transport | Notes |
|-----|----|-----------|-------|
| **Emerald** | 65100 | **LDP** (classic MPLS) | LDP + LDP-IGP sync; also the **LDP→SR migration** lab (WB04). BGP-LU for inter-AS labels. |
| **Gold** | 65300 | **SRv6** | IPv6 locators, End.* SIDs, L3VPN over SRv6 (End.DT4/DT6/DT46). No LDP. |
| **Garnet** | 65200 | **SR-MPLS** | Prefix/adjacency SIDs, TI-LFA, SR-TE, Flex-Algo. No LDP. |

**Inter-domain handoffs** (where the transport planes meet) are the high-value exam scenarios:
- **Emerald ↔ Gold** (E-R6↔G-R4): LDP/BGP-LU ↔ SRv6.
- **Gold ↔ Garnet** (G-R5↔Gar-R7): SRv6 ↔ SR-MPLS Option-B stitching (End.B6 / label re-encap).
- **Emerald ↔ Garnet** (E-R6↔Gar-R7): LDP/BGP-LU ↔ SR-MPLS.

---

## 7. Customer Design Summary

Three customers ride over the ISPs, deliberately spanning **multiple SPs** to force inter-AS VPN and cross-transport stitching:

| Customer | Spans | VPN Type | Inter-AS Method | Exercises |
|----------|-------|----------|-----------------|-----------|
| **Customer A** | Emerald ↔ Gold | L3VPN | **Option C** (BGP-LU + multihop RR-RR VPNv4) | LDP/BGP-LU ↔ SRv6 handoff (End.DT4 transport) |
| **Customer B** | Gold ↔ Garnet | L3VPN | **Option B** (VPNv4 + retain RT all + next-hop-self) | SRv6 ↔ SR-MPLS stitching at the ASBR |
| **Customer C** | Multi-SP (all three) | L3VPN / L2VPN / EVPN | Mixed (Option A/B/C per segment) | End-to-end service across all three transports |

CE assignments (per `00_topology_reference.md`): AS 65012 (CE1/CE2/CE8) and AS 65013 (CE9/CE4) for PE-CE eBGP; OSPF PE-CE on CE3/CE6; EVPN multi-homing on CE5/CE7. Dual-homed CEs (CE2/CE8) exercise SoO and site-of-origin loop prevention.

---

## 8. Prerequisites

**Do the GNS3 workbooks FIRST.** These EVE-NG workbooks assume you already have protocol fundamentals down on **IOS classic** (IOS-XE / classic CLI). The GNS3 track covers IS-IS, OSPF, LDP, L3VPN, PE-CE, advanced VPN, L2VPN/VPLS, and BGP on the classic platform so that here you can focus on:

- **IOS-XR syntax and operational model** (two-stage commit, config groups, RPL, address-family hierarchy).
- **Modern transport** (SR-MPLS, SRv6, Flex-Algo) that classic IOS doesn't fully cover.
- **Inter-AS and cross-transport stitching** across the 3-ISP design.

Recommended prior track:
```
GNS3: 01(IS-IS) → 02(OSPF) → 03(LDP) → 04(L3VPN) → 05(PE-CE) → 06(Adv VPN) → 07(L2VPN) → 08(VPLS) → 11(BGP)
                                                                                                    ↓
EVE-NG (this folder): WB01 → WB02 → … → WB16
```

Also review [`00_topology_reference.md`](00_topology_reference.md) before starting so the node names, loopbacks, links, and NIC→interface mapping are second nature.

---

*Study notes for CCIE Service Provider v5.1 lab preparation. Platform: IOS-XRv 9000 on GNS3 / EC2 m8i.24xlarge.*
