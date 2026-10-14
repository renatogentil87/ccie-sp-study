# CCIE SP v5.1 Workbook Gap Analysis

> **Scan date:** 2026-10-14
> **Workbooks scanned:** WB00–WB15 (16 blueprints + matching `_Solutions` files) in
> `/Users/rrdog/Desktop/ccie-sp-study/GNS3/workbooks/`
> **Topology:** `00_topology_reference.md` — 3 ASes (65100 IS-IS, 65200 OSPF, 65300 IS-IS), RRs/ASBRs/PEs/CEs as mapped.

## ⚠️ Platform Reality Check (read first)

The lab runs on **Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11, GNS3/Dynamips** — i.e. **IOS classic, not IOS-XR**, on emulated 7200 hardware.

That matters because the largest blueprint gaps below are **modern IOS-XR features that this platform physically cannot run**:

- **Segment Routing (SR-MPLS, prefix/adjacency/anycast-SID, TI-LFA, uloop avoidance)** — not supported on 7200/IOS 15.2.
- **SRv6 (uSID/C-SID, locators, psp-usd)** — IOS-XR / modern silicon only.
- **SR-TE, SR-ODN, Flex-Algo** — SR-dependent, so unavailable here.
- **EVPN (VPWS/multipoint, ESI, route types 1–5)** — not on 7200/IOS 15.2.
- **NSO** — an external orchestration product (CFS/RFS, YANG), not a device feature; needs a real NSO instance + XR/IOS-XE NEDs.
- **Model-driven Telemetry (gRPC dial-in/out, YANG, TPA)** — IOS-XR / IOS-XE 16+ feature.
- **SRv6-based services / 6PE-SR** — SR-dependent.

**Recommendation:** The request asks for "new IOS-XR workbooks," but the current topology is IOS classic. Two tracks are proposed below:

- **Track A — doable on the existing 7200/IOS 15.2 lab now** (EVPN caveat aside): fills the achievable classic-SP gaps.
- **Track B — requires a separate IOS-XR environment** (CML2 with IOS-XRv9000, or Cisco dCloud/sandbox): the SR/SRv6/EVPN/NSO/telemetry blueprint sections. These cannot be delivered on the current Dynamips topology.

---

## Currently Covered (WB00–WB15)

| WB | Title | Coverage |
|----|-------|----------|
| **WB00** | Foundation: IP Addressing | Base device hygiene (all 31 routers), Loopback0, core/inter-AS/PE-CE interface addressing across AS65100/65200/65300. |
| **WB01** | IS-IS | IS-IS L2-only on AS65100 (R1–R8) and AS65300 (R20–R23, R30): NET-IDs, interface activation, metrics, auth. |
| **WB02** | OSPF | OSPF Area 0 on AS65200 (R11–R16, R19): deterministic RIDs, core link advertisement. |
| **WB03** | LDP / MPLS | LDP on all 3 ASes (IS-IS cores + OSPF core): router-id pinning, LDP on core interfaces, label distribution. |
| **WB04** | BGP | iBGP + Route Reflectors (R7/R8, R19, R30), eBGP inter-AS, path manipulation, advanced BGP, VPNv4 AF. |
| **WB05** | L3VPN | VRF/RD/RT (GREEN/YELLOW/BLUE), PE-CE eBGP, PE-CE OSPF, VPNv4 on RRs. |
| **WB06** | L2VPN AToM / VPWS | Point-to-point pseudowires (port- & VLAN-based EoMPLS), targeted LDP, PW redundancy, troubleshooting. |
| **WB07** | VPLS | VFI "RED" (manual + BGP autodiscovery), split-horizon, intra-AS + cross-AS, **H-VPLS**, MAC learning. |
| **WB08** | MPLS-TE | RSVP + IS-IS TE extensions, explicit-path tunnels, **FRR (NHOP/NNHOP)**, bandwidth/priority/preemption, **auto-bandwidth**, MBB/SE, **affinity/admin-groups**, CSPF, **DS-TE (MAM/RDM)**, TE+VPN integration. |
| **WB09** | Inter-AS L3VPN | Options **A** (back-to-back VRF) & **B** (VPNv4 exchange), full inter-AS connectivity for all customers. |
| **WB10** | QoS (MQC) | DiffServ classification, marking + **DSCP↔EXP mapping**, policing, shaping/WRED, applied on PEs R1/R14. |
| **WB11** | Multicast + mVPN | PIM-SM (RP=R7), PIM-SSM, **MSDP** (R7↔R19), **mVPN Profile 0** (Default MDT GRE/PIM), **mVPN Profile 3** (BGP A-D + PIM C-mcast). |
| **WB12** | 6PE & 6VPE | 6PE (global IPv6 over IPv4 MPLS), 6VPE (IPv6 L3VPN / VPNv6), end-to-end verification. |
| **WB13** | SP Security | IGP auth (IS-IS/OSPF), BGP MD5 + **GTSM** + max-prefix, **RTBH**, **Flowspec**, **CoPP**, bogon/martian filtering, AS-path filtering, **uRPF**, LDP MD5, RSVP auth. |
| **WB14** | BGP-LU / Unified MPLS | eBGP labeled-unicast at AS boundaries, seamless end-to-end LSP (Unified MPLS), redistribute BGP-LU into iBGP, next-hop-self at ASBRs. |
| **WB15** | IPv6 Routing | Dual-stack addressing, IS-IS IPv6 (single-topology), OSPFv3, global IPv6 enable on all routers. |

**Blueprint sections already well covered:** Core IGP/BGP/LDP (1), L3VPN (7), L2VPN/VPWS (9), VPLS (10), MPLS-TE incl. FRR/auto-bw/DS-TE/affinity (12), QoS (13), Multicast incl. profiles 0 & 3 (14), 6PE/6VPE (15), Security (16), BGP-LU/Unified MPLS (1), IPv6 (bonus).

---

## Missing Topics (vs. blueprint)

Legend: **[A]** = achievable on current 7200/IOS 15.2 lab · **[B]** = requires separate IOS-XR environment.

| # | Blueprint topic | Status | Notes |
|---|-----------------|--------|-------|
| 2 | **SR-MPLS** (prefix/SRGB/SRLB/adjacency/anycast-SID, TI-LFA, microloop avoidance) | ❌ **[B]** | No SR on IOS 15.2/7200. XR lab required. |
| 3 | **SRv6** (uSID/C-SID, locators, psp-usd, SRv6 services) | ❌ **[B]** | XR + modern silicon only. |
| 4 | **SR-TE** (dynamic/explicit policies, binding-SID, autoroute, color/autosteer, PCE) | ❌ **[B]** | SR-dependent. |
| 5 | **SR-ODN** (on-demand policies, color communities, PCE-delegated) | ❌ **[B]** | SR-dependent. |
| 6 | **Flex-Algo** (algo definition, affinity, include/exclude, per-algo prefix-SID) | ❌ **[B]** | SR-dependent. |
| 11 | **EVPN** (EVPN-VPWS, multipoint, ESI, multi-homing, route-types 1–5) | ❌ **[B]** | Not on 7200/IOS 15.2. XR/XE lab required. |
| 14 | **mVPN Profile 6** (VRF-aware, direct-MDT / partitioned) | ⚠️ **[A/—]** | Profiles 0 & 3 done; **Profile 6** absent. Partial support on IOS classic — verify; likely best deferred to XR. |
| 17 | **NSO** (onboarding, sync, templates, L3VPN service, troubleshooting) | ❌ **[B]** | External orchestrator; needs NSO install + NEDs + manageable devices. |
| 18 | **Telemetry** (gRPC dial-in/out, YANG, TPA) | ❌ **[B]** | MDT is XR/XE-16+ only. |
| 19 | **High Availability** (BFD, NSF/NSR, graceful restart) | ⚠️ **[A]** | **No dedicated workbook.** BFD + IGP/BGP GR + NSF are supported on IOS 15.2 → fillable now. |
| 20 | **Carrier Supporting Carrier (CSC)** | ❌ **[A]** | **No workbook.** CSC (BGP-LU / LDP label exchange on PE-CE) is supported on IOS 15.2 → fillable now, builds directly on WB14. |
| 8 | **Inter-AS Option C** + RT retention + BGP-LU stitching | ⚠️ **[A]** | WB09 covers Options A & B only; **Option C** (multihop eBGP VPNv4 RR-to-RR + BGP-LU for /32s) is missing and is a core v5.1 item. Supported on IOS 15.2. |

**Fully covered blueprint sections:** 1, 7, 9, 10, 12, 13, 15, 16 (and parts of 14).

---

## Suggested New Workbooks

### Track A — buildable on the current 7200 / IOS 15.2 topology now

#### WB16 — Inter-AS Option C + BGP-LU Stitching
- **Scope:** Complete the Inter-AS trio. Multihop eBGP VPNv4 between RRs (R7/R8 ↔ R19 ↔ R30), BGP-LU to carry PE /32 loopbacks across AS boundaries, next-hop preservation, RT retention vs. rewrite, compare label stacks against Option A/B from WB09. Tie into Unified MPLS (WB14).
- **Routers:** ASBRs R5/R6/R12/R20/R21; RRs R7/R8 (65100), R19 (65200), R30 (65300); PEs R1/R2/R14/R22/R23; verify with customer **Blue** (R25/R32, spans all 3 ASes).

#### WB17 — Carrier Supporting Carrier (CSC)
- **Scope:** CSC-CE / CSC-PE model: a customer carrier running MPLS over a backbone carrier's L3VPN. LDP label exchange (or BGP-LU) on the PE-CE link so the customer carrier's loopbacks/labels transit the backbone. Build on WB05 VRFs + WB14 BGP-LU. End-to-end LSP across the CSC boundary.
- **Routers:** Backbone carrier = AS65200 (PE R14/R16, P R11/R13). Customer carrier = AS65300 reframed as a CSC customer reaching AS65100 PEs; CSC link on an R12↔R21 / R16↔R26 style boundary. Verify LFIB on the carrier-of-carriers path.

#### WB18 — High Availability (BFD, NSF/NSR, Graceful Restart)
- **Scope:** BFD for IS-IS/OSPF/BGP (fast failure detection, async mode, echo), BFD-triggered FRR, IGP/BGP **Graceful Restart** + **NSF**, SSO behavior where emulatable. Measure convergence before/after with continuous ping. Pair BFD with WB08 TE-FRR and WB13.
- **Routers:** AS65100 core R3/R4/R5/R6 (BFD on IS-IS core links), inter-AS eBGP R5↔R11 / R6↔R12 (BGP BFD + GR), PE R1/R2.

#### WB19 — mVPN Profile 6 (optional, verify platform support)
- **Scope:** Only if IOS 15.2 supports it on 7200 — VRF-aware / partitioned MDT variant to complete the 0/3/6 profile set from WB11. If unsupported, document as a Track-B/XR item.
- **Routers:** mVPN PEs R1/R2 (65100) and R14 (65200), RP R7, MSDP peer R19.

### Track B — require a separate IOS-XR lab (CML2 IOS-XRv9000 / dCloud); NOT runnable on current topology

#### WB20 (XR) — SR-MPLS Fundamentals
- **Scope:** SRGB/SRLB, prefix-SID & node-SID, adjacency-SID, anycast-SID, SR over IS-IS and OSPF, **TI-LFA**, microloop avoidance, SR/LDP interworking (mapping server).
- **Routers (XR relab):** map existing R1–R8 roles onto XRv nodes; anycast-SID on a dual-RR/ABR pair.

#### WB21 (XR) — SR-TE Policies
- **Scope:** Explicit + dynamic candidate paths, binding-SID, autoroute-announce, color + on-demand steering, SR-PCE (dynamic computation), per-flow steering.
- **Routers (XR):** headend PEs, SR-PCE node, color-based service endpoints.

#### WB22 (XR) — SR On-Demand Next-hop (ODN) + Flex-Algo
- **Scope:** ODN templates keyed on color communities, PCE-delegated computation; Flex-Algo definitions (128/129…), affinity include/exclude, per-algo prefix-SID, mapping to SR-TE.
- **Routers (XR):** multi-PE with differentiated metric/affinity planes.

#### WB23 (XR) — SRv6
- **Scope:** uSID/C-SID, locator planning, End/End.X/End.DT behaviors, psp-usd flavors, SRv6-based L3VPN (VPNv4/v6) and L2VPN services.
- **Routers (XR):** SRv6-capable XRv9000 nodes.

#### WB24 (XR/XE) — EVPN
- **Scope:** EVPN-VPWS, EVPN multipoint (ELAN), ESI + all-active/single-active multi-homing, route-types 1–5, integration with VPLS/L2VPN migration story.
- **Routers (XR/XE):** two-PE multi-homed CE for ESI; replaces WB06/WB07 L2 story on modern platforms.

#### WB25 (XR + NSO server) — NSO Service Orchestration
- **Scope:** Device onboarding + sync-from, device templates, an **L3VPN CFS/RFS** service package, service re-deploy / dry-run, out-of-band change reconciliation, troubleshooting (sync, commit, NED).
- **Infra:** NSO install (local VM/container) + NEDs; manage the XR nodes from WB20–WB24 (or the IOS devices where NEDs allow).

#### WB26 (XR/XE) — Model-Driven Telemetry
- **Scope:** gRPC dial-out + dial-in, YANG model paths (sensor-paths), subscription/destination groups, TPA (Telemetry Path Advisor), collector (TIG/Pipeline) ingest + validation.
- **Routers (XR/XE):** telemetry-capable nodes exporting IGP/BGP/interface counters.

---

## Priority Order

1. **WB16 Inter-AS Option C** — pure blueprint gap, runs today, high exam weight.
2. **WB17 CSC** — explicit blueprint item (#20), runs today, builds on WB14.
3. **WB18 High Availability (BFD/NSF/NSR/GR)** — blueprint #19, runs today.
4. **WB19 mVPN Profile 6** — only if platform-verified; else fold into Track B.
5. **Track B (WB20–WB26)** — stand up an IOS-XR lab; these are the dominant modern v5.1 sections (SR/SRv6/EVPN/NSO/telemetry) and cannot be met on the current Dynamips topology.
