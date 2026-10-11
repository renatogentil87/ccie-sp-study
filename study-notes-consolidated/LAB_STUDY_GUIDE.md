# CCNP-SP Lab Study Guide — Recommended Order

## Your Current Progress

Based on your completed tasks:

- ✅ **Lab 1** (MPLS Basics) — DONE. OSPF + LDP + MPLS running on all core routers.
- 🟡 **Lab 2** (L3VPN) — Sections 1-4 DONE + CCIE+ Challenges 1-2 DONE. Remaining: Challenges 3 (RT-Constraint), 4 (Inter-AS Option A), 5 (Internet Access), 6 (Export-Map).
- 🟡 **Lab 3** (MPLS TE) — Tunnels working, explicit paths, primary/backup failover confirmed. Remaining: Sections 3-6 (bandwidth, affinity, VPN-over-TE, QoS tunnels).
- ⬜ **Labs 4-18** — Not started.

**You are ready for:** Finish Lab 2 Challenges 3-6, then finish Lab 3 remaining sections, then proceed to Lab 7.

---

## Folder Structure

```
docs/study-notes/ccnp-sp/
├── LAB_STUDY_GUIDE.md          ← This file
├── notes.md                     ← General study notes
└── labs/
    ├── lab_1_details.md  through lab_18_details.md
```

---

## GNS3 Topologies Needed

| Topology | Name Suggestion | Labs | Reason |
|---|---|---|---|
| Topology A | `SP-Core-OSPF` | 1, 2, 3, 7, 8, 12, 13, 14, 15 | Main build — OSPF + LDP + L3VPN + TE. Non-destructive progression. |
| Topology B | `SP-L2VPN` | 4, 5, 11 | L2VPN labs — PE-CE interfaces are pure L2 (conflicts with L3VPN interfaces) |
| Topology C | `SP-Advanced-VPN` | 6 | Advanced L3VPN with RT changes — modifies Customer_A RT design |
| Topology D | `SP-ISIS` | 10 | IS-IS replaces OSPF — different IGP, same physical topology |

All four topologies use the **same physical wiring** (same 20-router topology file in GNS3). You just load a different saved config set for each. Alternatively, use GNS3 snapshots to switch between states.

**Tip:** In GNS3, you can duplicate a project or use snapshots. Build Topology A first (Labs 1-3), then snapshot it. Create Topology B by starting from a Lab 1 snapshot (OSPF+LDP only, no L3VPN interfaces).

---

## Phase 1: Foundation (Topology A)

Complete in this exact order. Each lab builds on the previous — do NOT skip.

| Order | Lab | What You're Building | Cumulative State After |
|---|---|---|---|
| 1st | **Lab 1** | OSPF area 0 + LDP + MPLS on all core routers | IGP reachability, labels allocated |
| 2nd | **Lab 2** | L3VPN — 5 customers, RRs, mixed PE-CE | Full VPN service, RRs on R3/R7 |
| 3rd | **Lab 3** | MPLS TE — tunnels, explicit paths, FRR | TE tunnels carrying VPN traffic |
| 4th | **Lab 7** | OAM & Protection — LSP ping, BFD, LFA, LDP protection | Hardened MPLS core with fast convergence |
| 5th | **Lab 8** | BGP Policy — communities, path control, RT-constraint | Full BGP scalability toolkit |

**Snapshot Topology A here** — this is your "golden state" with everything working.

---

## Phase 2: L2VPN (Topology B)

Create a NEW GNS3 project (or snapshot from Lab 1 state — OSPF+LDP only, no VRFs on PE-CE interfaces).

| Order | Lab | What You're Building | Notes |
|---|---|---|---|
| 6th | **Lab 4** | AToM pseudowires — point-to-point L2VPN | PE-CE interfaces become pure L2 (xconnect) |
| 7th | **Lab 5** | AToM + TE tunnel binding, redundancy | Builds on Lab 4's pseudowires + TE knowledge from Lab 3 |
| 8th | **Lab 11** | VPLS — multipoint L2VPN | Builds on Lab 4's pseudowire concepts |

**Why separate topology?** Labs 4/5/11 remove IP addresses and VRFs from PE-CE interfaces to make them L2 ports. This destroys the L3VPN setup from Lab 2.

---

## Phase 3: Advanced VPN Design (Topology C)

Create from a snapshot of Topology A (after Lab 2). You need L3VPN working but will modify RT design.

| Order | Lab | What You're Building | Notes |
|---|---|---|---|
| 9th | **Lab 6** | Hub-spoke, shared services, multi-homing, extranet | Changes Customer_A RT from full-mesh to hub-spoke in Section 2. Restore at end of each section if you want to continue. |

**Why separate?** Lab 6 Section 2 changes RT import/export for Customer_A — breaks the full-mesh from Lab 2. If you restore RTs to normal after each section, you can reuse Topology A instead. Your choice.

---

## Phase 4: IPv6 and Multicast (Topology A — Return)

Go back to your Topology A golden state. These ADD to the existing setup without breaking anything.

| Order | Lab | What You're Building | Notes |
|---|---|---|---|
| 10th | **Lab 12** | 6PE/6VPE — IPv6 over MPLS | Adds IPv6 addresses + vpnv6 BGP. IPv4 VPN untouched. |
| 11th | **Lab 13** | Multicast VPN | Adds PIM + MDT to core. Unicast VPN untouched. |

---

## Phase 5: Security and QoS (Topology A — Continue)

Still on Topology A. These overlay security and QoS without changing the routing/VPN design.

| Order | Lab | What You're Building | Notes |
|---|---|---|---|
| 12th | **Lab 14** | Infrastructure Security — CoPP, uRPF, RTBH, auth | Adds security policies on top. Non-destructive. |
| 13th | **Lab 15** | QoS — full DiffServ model | Adds QoS policies on all interfaces. Non-destructive. |

---

## Phase 6: IS-IS Migration (Topology D)

Create a NEW topology from Lab 1 state (or fresh). This replaces OSPF entirely.

| Order | Lab | What You're Building | Notes |
|---|---|---|---|
| 14th | **Lab 10** | IS-IS as SP IGP, multi-level, TE extensions | Replaces OSPF with IS-IS. Everything else (LDP, TE, VPN) re-validated on IS-IS. |

**Why separate?** IS-IS removes OSPF. All subsequent labs assume OSPF. Keep this isolated so you can always return to OSPF-based Topology A.

**Alternative:** Do Lab 10 LAST on Topology A as a "final migration exercise" — migrate OSPF→IS-IS and back (Challenge 5 in Lab 10 covers the reverse migration).

---

## Phase 7: EVE-NG Labs (c5.metal — New Topologies)

These are completely separate from GNS3. Build on EVE-NG when ready.

| Order | Lab | Topology | Images |
|---|---|---|---|
| 15th | **Lab 16** | SR Topology (10 IOS-XRv routers) | IOS-XRv 9000 7.x+ |
| 16th | **Lab 18** | Same SR Topology (reuse Lab 16) | IOS-XRv 9000 7.x+ |
| 17th | **Lab 17** | EVPN Topology (5 NXOSv + 3 IOS-XRv) | NXOSv 9000 9.3+, IOS-XRv 7.x+ |

**Lab 16 before Lab 18** because Lab 18 (programmability) configures the SR network built in Lab 16 via NETCONF instead of CLI.

---

## Quick Reference — Lab Dependencies

```
Lab 1 ──→ Lab 2 ──→ Lab 3 ──→ Lab 7 ──→ Lab 8 ──→ Lab 12 ──→ Lab 13 ──→ Lab 14 ──→ Lab 15
                │
                └──→ Lab 6 (separate topology or restore RTs after)

Lab 1 ──→ Lab 4 ──→ Lab 5
                │
                └──→ Lab 11

Lab 1 ──→ Lab 10 (separate topology — IS-IS replaces OSPF)

Lab 16 ──→ Lab 18 (EVE-NG, SR topology)
Lab 17 (EVE-NG, EVPN topology — independent)
```

---

## Summary: What to Build When

| Step | Action | Time Estimate |
|---|---|---|
| 1 | Build GNS3 20-router topology (if not already done) | Already done ✅ |
| 2 | Complete Labs 1-3, 7, 8 in order on main topology | 4-6 weeks |
| 3 | Snapshot main topology as "golden state" | 5 minutes |
| 4 | Create L2VPN topology (from Lab 1 snapshot), do Labs 4, 5, 11 | 2-3 weeks |
| 5 | Return to golden state, do Lab 6 (or create new topology) | 1-2 weeks |
| 6 | Return to golden state, do Labs 12, 13, 14, 15 | 3-4 weeks |
| 7 | Create IS-IS topology, do Lab 10 | 1 week |
| 8 | Build EVE-NG on c5.metal, do Labs 16, 18, 17 | 3-4 weeks |

**Total estimated time:** 14-20 weeks at consistent lab pace (2-3 hours/day)

---

## GNS3 Snapshot Strategy

```
Snapshot 1: "Lab-1-Complete"     → OSPF + LDP + MPLS only
Snapshot 2: "Lab-2-Complete"     → Full L3VPN (5 customers, RRs, mixed PE-CE)
Snapshot 3: "Lab-3-Complete"     → L3VPN + TE tunnels
Snapshot 4: "Golden-State"       → After Labs 7+8 (full BGP + OAM + protection)
Snapshot 5: "L2VPN-Base"         → From Snapshot 1, PE-CE as L2 ports (for Labs 4/5/11)
Snapshot 6: "ISIS-Migration"     → From Snapshot 1, IS-IS instead of OSPF
```

This way you never lose progress and can jump between lab contexts in seconds.

---

## Recommended Books and Resources

### For SPCOR 350-501 Exam (Buy Now)

| Book | Purpose | When to Read |
|---|---|---|
| **CCNP and CCIE SP Core SPCOR 350-501 Official Cert Guide** (Cisco Press) | Primary exam resource — covers full blueprint | Read alongside labs, chapter by chapter |
| **Boson ExSim for SPCOR 350-501** | Practice exam questions — simulates real exam format | Last 2-3 weeks before sitting the exam |

### For CCIE-SP Lab Exam (Buy After Passing SPCOR)

| Book | Purpose | When to Read |
|---|---|---|
| **BGP Design and Implementation** (Cisco Press, Russ White) | Deep BGP policy, confederation design, RR optimisation | When preparing Labs 8 + CCIE lab scenarios |
| **MPLS in the SDN Era** (Cisco Press, Antonio Sanchez-Monge) | SR-MPLS, SR-TE, LDP-to-SR migration, modern SP architectures | When building EVE-NG Lab 16 (Segment Routing) |
| **EVPN-VXLAN Fabric Design** (Cisco Press) | DC fabric design, EVPN route types, multi-homing, DCI | When building EVE-NG Lab 17 (EVPN) |
| **Segment Routing Part I & II** (Cisco Press, Clarence Filsfils) | Deep SR architecture, Flex-Algo, PCE, TI-LFA internals | Advanced SR study for CCIE lab |

### Free Resources

| Resource | Purpose |
|---|---|
| Cisco DevNet Sandbox (devnetsandbox.cisco.com) | Free CSR1000v labs for NETCONF/YANG practice |
| Cisco dCloud | Pre-built SR and EVPN guided labs (time-limited) |
| INE SPCOR video course | Video walkthrough of all blueprint topics |
| Cisco Live On-Demand (ciscolive.com) | Deep-dive presentations on SR, EVPN, mVPN by Cisco engineers |

