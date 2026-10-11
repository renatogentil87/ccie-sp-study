# CCNP-SP Lab Topologies — README

## Overview

This curriculum contains **34 configuration labs** across **7 topologies** (5 on GNS3, 2 on EVE-NG). All GNS3 labs run on Mac M4 using Dynamips with Cisco 7200 (IOS 15.2).

---

## GNS3 Topologies

| Topology | Routers | Labs | GNS3 Project |
|---|---|---|---|
| **A — SP-Core-OSPF** | 20 | 1, 2, 3, 7, 8, 12, 13, 14, 15, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31 | Main build — stack non-destructively |
| **B — SP-L2VPN** | 20 | 4, 5, 11 | PE-CE interfaces become L2 (conflicts with L3VPN) |
| **C — SP-Advanced-VPN** | 20 | 6 | Modifies RT design for Customer_A |
| **D — SP-ISIS** | 20 | 10 | IS-IS replaces OSPF entirely |
| **E — Internet Edge** | 26 | 32, 33, 34 | Multi-ASN, transit, IXP, peering |

## EVE-NG Topologies (AWS c5.metal)

| Topology | Routers | Labs | Platform |
|---|---|---|---|
| **SR Topology** | 10 IOS-XRv | 16, 18 | EVE-NG on AWS c5.metal |
| **EVPN Topology** | 8 (5 NXOSv + 3 IOS-XRv) | 17 | EVE-NG on AWS c5.metal |

---

## Practical Build Strategy

**You only need 2 GNS3 projects:**

### Project 1: Main (26 routers) — Topologies A, C, D, E

Build all 26 routers in one GNS3 project. Use **snapshots** to manage different topology states:

- **Snapshot "Lab 1 — Base OSPF+LDP"** → starting point for most labs
- **Snapshot "Lab 2 — Full L3VPN"** → Topology A golden state
- **Snapshot "Lab 6 — Advanced VPN"** → Topology C (restore after)
- **Snapshot "Lab 10 — IS-IS"** → Topology D (OSPF replaced)
- **R21-R26 stay powered off** until Lab 32 (Topology E)

### Project 2: L2VPN (20 routers) — Topology B

Clone from Lab 1 snapshot (OSPF+LDP only, no VRFs on PE-CE interfaces). Labs 4, 5, 11 remove IP addresses from PE-CE interfaces and convert them to L2 — this destroys the L3VPN setup, so it must be isolated.

---

## Router Inventory — All 26 Routers

### Original 20 Routers (Topologies A-D)

| Router | Console Port | Loopback | Role | AS |
|---|---|---|---|---|
| R1 | 5009 | 1.1.1.1 | CE (Customer_A West) | 65001 |
| R2 | 5000 | 2.2.2.2 | PE (West) | 64512 |
| R3 | 5001 | 3.3.3.3 | P / Route Reflector (North) | 64512 |
| R4 | 5002 | 4.4.4.4 | P (Core) | 64512 |
| R5 | 5003 | 5.5.5.5 | P (Core) | 64512 |
| R6 | 5004 | 6.6.6.6 | P (Core) | 64512 |
| R7 | 5005 | 7.7.7.7 | P / Route Reflector (South) | 64512 |
| R8 | 5006 | 8.8.8.8 | PE (East) | 64512 |
| R9 | 5007 | 9.9.9.9 | CE (Customer_A East) | 65001 |
| R10 | 5008 | 10.10.10.10 | CE → **ASBR-1 in Topology E** | 64512 |
| R11 | 5010 | 11.11.11.11 | CE → **ASBR-2 in Topology E** | 64512 |
| R12 | 5011 | 12.12.12.12 | CE → **ASBR-3/IXP Edge in Topology E** | 64512 |
| R13 | 5012 | 13.13.13.13 | P (South) | 64512 |
| R14 | 5013 | 14.14.14.14 | P (South) | 64512 |
| R15 | 5014 | 15.15.15.15 | P (South) | 64512 |
| R16 | 5015 | 16.16.16.16 | P (South) | 64512 |
| R17 | 5016 | 17.17.17.17 | PE (South) | 64512 |
| R18 | 5017 | 18.18.18.18 | PE (South) | 64512 |
| R19 | 5018 | 19.19.19.19 | CE (Customer_D) | 65019 |
| R20 | 5019 | 20.20.20.20 | CE (Customer_E) | 65020 |

### 6 New Routers (Topology E — Internet Edge)

| Router | Console Port | Loopback | Role | AS |
|---|---|---|---|---|
| R21 | 5020 | 21.21.21.21 | Transit Provider 1 (Cogent sim) | 174 |
| R22 | 5021 | 22.22.22.22 | Transit Provider 2 (Lumen sim) | 3356 |
| R23 | 5022 | 23.23.23.23 | Peer SP (Regional ISP) | 9002 |
| R24 | 5023 | 24.24.24.24 | IXP Route Server | 65000 |
| R25 | 5024 | 25.25.25.25 | IXP Peer 1 (Content/Cloudflare sim) | 13335 |
| R26 | 5025 | 26.26.26.26 | IXP Peer 2 (Hyperscaler/AWS sim) | 16509 |

---

## Physical Connectivity — 6 New Routers (Topology E)

### How R21-R26 Connect to the Existing Topology

```
                              ┌─────────────────────────────────────────────┐
                              │            EXTERNAL NETWORKS                 │
                              │                                             │
                              │   R21 (AS 174)         R22 (AS 3356)        │
                              │   Transit Cogent       Transit Lumen        │
                              │       │                     │                │
                              └───────┼─────────────────────┼────────────────┘
                                      │                     │
                            Se2/0     │ Se2/0               │ Se2/0
                        198.51.100.2  │ 198.51.100.1   198.51.100.6  198.51.100.5
                                      │                     │
┌─────────────────────────────────────┼─────────────────────┼──────────────────────────────────────────┐
│  AS 64512 (YOUR SP NETWORK)         │                     │                                          │
│                                     │                     │                                          │
│                              R10 (ASBR-1)          R11 (ASBR-2)                                      │
│                              Fa0/0↑transit          Fa0/0↑transit                                    │
│                              Gi1/0↓core             Gi1/0↓core                                       │
│                                │                       │                                             │
│                                │ 172.16.10.0/30        │ 172.16.11.0/30                              │
│                                │                       │                                             │
│                                R4 ──────────────── R5                                                │
│                               / \                  / \                                               │
│                         R3(RR)   R6 ──────── R7(RR)   R8(PE)──R9(CE)                                 │
│                           │       │              │                                                   │
│                         R2(PE)    │            R13──R14──R15──R16                                     │
│                           │       │                         │    │                                   │
│                         R1(CE)    │                       R17(PE) R18(PE)                             │
│                                   │                         │      │                                 │
│                                   │ 172.16.12.0/30        R19(CE) R20(CE)                            │
│                                   │                                                                  │
│                              R12 (ASBR-3 / IXP Edge)                                                 │
│                              Gi1/0↑core (to R6)                                                      │
│                              Gi2/0↓IXP LAN                                                           │
│                              Fa4/0↓PNI                                                               │
│                                │        \                                                            │
└────────────────────────────────┼─────────\───────────────────────────────────────────────────────────┘
                                 │          \
                       ┌─────────┼───────────\──────────────────────────────────────────┐
                       │  IXP FABRIC          \ PNI (Private Link)                      │
                       │  (Layer 2 Switch)     \                                        │
                       │  198.18.0.0/24         \ 203.0.113.0/30                        │
                       │         │               \                                      │
                       │    R12: 198.18.0.1       R23 (AS 9002, Peer SP)                │
                       │    R23: 198.18.0.23      203.0.113.2                           │
                       │    R24: 198.18.0.24                                            │
                       │    R25: 198.18.0.25      R23 also on IXP: 198.18.0.23          │
                       │    R26: 198.18.0.26                                            │
                       │         │                                                      │
                       │    R24 (IXP Route Server, AS 65000)                            │
                       │    R25 (Content Network, AS 13335)                             │
                       │    R26 (Hyperscaler, AS 16509)                                 │
                       │                                                                │
                       └────────────────────────────────────────────────────────────────┘
```

### GNS3 Wiring Guide — New Links to Add

#### R10 (ASBR-1, Transit to Cogent)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| FastEthernet0/0 | R21 Se2/0 | 198.51.100.1 | /30 | eBGP to Transit Provider 1 (Cogent) |
| GigabitEthernet1/0 | R4 (new link) | 172.16.10.1 | /30 | Internal — ASBR to core |
| Loopback0 | — | 10.10.10.10 | /32 | Router-ID, iBGP source |

**Note:** R10 was previously a CE router. In Topology E, it becomes an ASBR. Either reconfigure it or use a snapshot that starts fresh for Labs 32-34.

#### R11 (ASBR-2, Transit to Lumen)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| FastEthernet0/0 | R22 Se2/0 | 198.51.100.5 | /30 | eBGP to Transit Provider 2 (Lumen) |
| GigabitEthernet1/0 | R5 (new link) | 172.16.11.1 | /30 | Internal — ASBR to core |
| Loopback0 | — | 11.11.11.11 | /32 | Router-ID, iBGP source |

#### R12 (ASBR-3, IXP Edge + Private Peering)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| GigabitEthernet1/0 | R6 (new link) | 172.16.12.1 | /30 | Internal — ASBR to core |
| GigabitEthernet2/0 | IXP Switch | 198.18.0.1 | /24 | IXP peering LAN |
| FastEthernet4/0 | R23 Fa0/0 | 203.0.113.1 | /30 | PNI (Private Network Interconnect) |
| Loopback0 | — | 12.12.12.12 | /32 | Router-ID, iBGP source |

#### R21 (Transit Provider 1 — Cogent, AS 174)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| Serial2/0 | R10 Fa0/0 | 198.51.100.2 | /30 | eBGP to your ASBR-1 |
| Loopback0 | — | 21.21.21.21 | /32 | Router-ID |
| Loopback1 | — | 100.64.1.1 | /24 | Simulated Internet prefix 1 |
| Loopback2 | — | 100.64.2.1 | /24 | Simulated Internet prefix 2 |
| Loopback3 | — | 100.64.3.1 | /24 | Simulated Internet prefix 3 |
| Loopback10 | — | 100.64.10.1 | /24 | Simulated Internet prefix 4 |
| Loopback20 | — | 100.64.20.1 | /24 | Simulated Internet prefix 5 |

**Simulating Internet routes:** R21 uses loopbacks + `network` statements under BGP to advertise prefixes as if it were a transit provider with a full table. No need for real Internet routes — 10-50 prefixes with varied AS-paths is sufficient for learning.

#### R22 (Transit Provider 2 — Lumen, AS 3356)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| Serial2/0 | R11 Fa0/0 | 198.51.100.6 | /30 | eBGP to your ASBR-2 |
| Loopback0 | — | 22.22.22.22 | /32 | Router-ID |
| Loopback1 | — | 100.64.1.1 | /24 | Same prefix as R21 (different path!) |
| Loopback2 | — | 100.64.2.1 | /24 | Same prefix as R21 (different path!) |
| Loopback50 | — | 100.64.50.1 | /24 | Unique to Lumen |
| Loopback51 | — | 100.64.51.1 | /24 | Unique to Lumen |

**Key:** R21 and R22 advertise OVERLAPPING prefixes with different AS-paths. This lets you practice BGP best-path selection between two transit providers.

#### R23 (Peer SP — Regional ISP, AS 9002)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| FastEthernet0/0 | R12 Fa4/0 | 203.0.113.2 | /30 | PNI (dedicated peering link) |
| GigabitEthernet1/0 | IXP Switch | 198.18.0.23 | /24 | Also present on IXP fabric |
| Loopback0 | — | 23.23.23.23 | /32 | Router-ID |
| Loopback1 | — | 203.0.113.128 | /25 | Peer's own prefix 1 |
| Loopback2 | — | 192.0.2.128 | /25 | Peer's own prefix 2 |

**Dual connectivity:** R23 connects to you via BOTH a private link (PNI) and the IXP. This lets you practice preferring PNI over IXP (using LOCAL_PREF or MED).

#### R24 (IXP Route Server, AS 65000)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| GigabitEthernet1/0 | IXP Switch | 198.18.0.24 | /24 | Route server on IXP fabric |
| Loopback0 | — | 24.24.24.24 | /32 | Router-ID |

**No transit links.** R24 only sits on the IXP LAN. It peers with all IXP members and reflects their routes transparently (doesn't modify AS-path). Acts as a multilateral peering facilitator.

#### R25 (IXP Peer 1 — Content Network, AS 13335)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| GigabitEthernet1/0 | IXP Switch | 198.18.0.25 | /24 | Present on IXP fabric |
| Loopback0 | — | 25.25.25.25 | /32 | Router-ID |
| Loopback1 | — | 104.16.1.1 | /24 | Simulated Cloudflare prefix 1 |
| Loopback2 | — | 104.16.2.1 | /24 | Simulated Cloudflare prefix 2 |
| Loopback3 | — | 104.16.3.1 | /24 | Simulated Cloudflare prefix 3 |

#### R26 (IXP Peer 2 — Hyperscaler, AS 16509)

| Interface | Connects To | IP Address | Subnet | Purpose |
|---|---|---|---|---|
| GigabitEthernet1/0 | IXP Switch | 198.18.0.26 | /24 | Present on IXP fabric |
| Loopback0 | — | 26.26.26.26 | /32 | Router-ID |
| Loopback1 | — | 52.0.1.1 | /24 | Simulated AWS prefix 1 |
| Loopback2 | — | 52.0.2.1 | /24 | Simulated AWS prefix 2 |
| Loopback3 | — | 3.0.1.1 | /24 | Simulated AWS prefix 3 |

---

## GNS3 Implementation — IXP Fabric

The IXP is a **Layer 2 broadcast domain** (like a real IXP switch fabric). In GNS3:

1. Add an **Ethernet Switch** node (built-in GNS3 switch, no image needed)
2. Connect these routers to the switch:
   - R12 Gi2/0 → Switch port 1
   - R23 Gi1/0 → Switch port 2
   - R24 Gi1/0 → Switch port 3
   - R25 Gi1/0 → Switch port 4
   - R26 Gi1/0 → Switch port 5
3. All on the same VLAN (default) — all in subnet 198.18.0.0/24
4. They can ARP and peer directly (just like a real IXP peering LAN)

```
GNS3 View:

    R12 ──┐
    R23 ──┤
    R24 ──┼── [Ethernet Switch] ── All on 198.18.0.0/24
    R25 ──┤
    R26 ──┘
```

---

## Core-to-ASBR Links — What to Add in GNS3

These are the NEW links between existing core routers and the ASBRs:

| From | Interface | To | Interface | Subnet | Notes |
|---|---|---|---|---|---|
| R4 | Gi2/0 (new) | R10 | Gi1/0 | 172.16.10.0/30 | R4=.2, R10=.1 |
| R5 | Fa3/0 (new) | R11 | Gi1/0 | 172.16.11.0/30 | R5=.2, R11=.1 |
| R6 | Gi2/0 (reuse or new) | R12 | Gi1/0 | 172.16.12.0/30 | R6=.2, R12=.1 |

**Important:** These internal links run OSPF area 0 + LDP (same as all other core links). The ASBRs participate in IGP so their loopbacks are reachable for iBGP sessions.

---

## Memory Requirements

| Topology | Routers Powered On | RAM per Router | Total RAM |
|---|---|---|---|
| A (Labs 1-31) | 20 | 256 MB | ~5.0 GB |
| E (Labs 32-34) | 26 | 256 MB | ~6.5 GB |
| B (Labs 4,5,11) | 20 | 256 MB | ~5.0 GB |

**Mac M4 (16GB+ RAM):** comfortable for all topologies. If you have 24GB+ you can run Topology E without issues. Set each router to 256MB in GNS3 (sufficient for BGP with 50-100 prefixes; only needs 512MB if loading 200K+ routes).

---

## Lab Progression Quick Reference

```
PHASE 1 — MPLS Foundation (Topology A):
  Lab 1 → Lab 2 → Lab 3 → Lab 7 → Lab 8

PHASE 2 — L2VPN (Topology B):
  Lab 4 → Lab 5 → Lab 11

PHASE 3 — Advanced VPN (Topology C):
  Lab 6

PHASE 4 — IPv6 + Multicast (Topology A):
  Lab 12 → Lab 13

PHASE 5 — Security + QoS (Topology A):
  Lab 14 → Lab 15

PHASE 6 — IS-IS (Topology D):
  Lab 10

PHASE 7 — EVE-NG (Separate platform):
  Lab 16 → Lab 18 → Lab 17

PHASE 8 — SP Operations (Topology A):
  Lab 9 → Lab 19 → Lab 20 → Lab 21 → Lab 22 → Lab 23 → Lab 24

PHASE 9 — Advanced MPLS VPN (Topology A):
  Lab 25 → Lab 26 → Lab 27 → Lab 28 → Lab 29 → Lab 30 → Lab 31

PHASE 10 — Advanced BGP / Internet Edge (Topology E):
  Lab 32 → Lab 33 → Lab 34
```

---

## File Locations

```
docs/study-notes/ccnp-sp/
├── README_TOPOLOGIES.md          ← This file
├── LAB_STUDY_GUIDE.md            ← Recommended order and progress tracking
├── PROFILE.md                    ← Personal study context
├── notes.md                      ← General study notes
└── labs/
    ├── configuration/
    │   ├── lab_01_mpls_forwarding_basics.md
    │   ├── lab_02_mpls_l3vpn.md
    │   ├── ...
    │   ├── lab_32_sp_internet_edge_peering.md
    │   ├── lab_33_advanced_route_reflectors_convergence.md
    │   └── lab_34_bgp_flowspec_operational_security.md
    └── troubleshooting/
        ├── ts_lab_1.md
        ├── ...
        └── ts_lab_9.md
```

---

## Base Snapshot — "OSPF-LDP-Core" (Starting Point for All Labs)

This is the ONE snapshot you build first. Every lab branches from this state.

**What's configured:** OSPF area 0 + LDP on all core links. Loopbacks reachable. Nothing else.
**What's NOT configured:** No BGP, no VRFs, no TE, no customers, no PE-CE routing.

### P Routers (9 routers) — Full OSPF + LDP

These run OSPF + LDP on ALL their interfaces (except management).

| Router | Loopback | Interfaces Running OSPF+LDP | Notes |
|---|---|---|---|
| **R3** | 3.3.3.3 | Fa0/0, Gi1/0, Gi2/0, Fa3/0 | Future RR (North) |
| **R4** | 4.4.4.4 | Fa0/0, Gi1/0 | Core |
| **R5** | 5.5.5.5 | Fa0/0, Gi1/0, Gi2/0, Fa3/0 | Core |
| **R6** | 6.6.6.6 | Fa0/0, Gi1/0, Gi2/0, Fa3/0, Fa4/0 | Core |
| **R7** | 7.7.7.7 | Fa0/0, Gi1/0, Gi2/0, Fa3/0, Fa4/0 | Future RR (South) |
| **R13** | 13.13.13.13 | Fa0/0, Gi1/0, Gi2/0, Fa3/0 | South core |
| **R14** | 14.14.14.14 | Fa0/0, Gi1/0, Gi2/0, Fa3/0 | South core |
| **R15** | 15.15.15.15 | Fa0/0, Gi1/0, Gi2/0 | South core |
| **R16** | 16.16.16.16 | Fa0/0, Gi1/0, Gi2/0 | South core |

**Config template for P routers:**
```
hostname R<X>
!
interface Loopback0
 ip address <X.X.X.X> 255.255.255.255
 ip ospf 1 area 0
!
interface <each core-facing interface>
 ip address <addr> <mask>
 ip ospf 1 area 0
 mpls ip
 no shutdown
!
router ospf 1
 router-id <X.X.X.X>
 mpls ldp autoconfig
!
mpls ldp router-id Loopback0 force
mpls label range <min> <max>
```

### PE Routers (4 routers) — OSPF + LDP on CORE side only

PE routers run OSPF + LDP only on their **core-facing** interfaces. CE-facing interfaces are physically up but have **no IP, no VRF** — left blank for each lab to configure.

| Router | Loopback | Core Interfaces (OSPF+LDP) | CE Interfaces (UNCONFIGURED) |
|---|---|---|---|
| **R2** | 2.2.2.2 | Gi1/0 (to R3), Gi2/0 (to R6) | Fa0/0 (→R1), Fa3/0 (→R12) |
| **R8** | 8.8.8.8 | Fa0/0 (to R7), Gi2/0 (to R5) | Gi1/0 (→R9), Fa4/0 (→R11) |
| **R17** | 17.17.17.17 | Fa0/0 (to R13), Gi2/0 (to R15) | Fa3/0 (→R19) |
| **R18** | 18.18.18.18 | Fa0/0 (to R14), Gi1/0 (to R16) | Gi2/0 (→R20) |

**Config template for PE routers:**
```
hostname R<X>
!
interface Loopback0
 ip address <X.X.X.X> 255.255.255.255
 ip ospf 1 area 0
!
! CORE-FACING (OSPF + LDP):
interface <core-intf>
 ip address <addr> <mask>
 ip ospf 1 area 0
 mpls ip
 no shutdown
!
! CE-FACING (LEFT BLANK — configured per lab):
interface <ce-intf>
 no ip address
 no shutdown
 ! Ready for: VRF (Lab 2), xconnect (Lab 4), or whatever the lab needs
!
router ospf 1
 router-id <X.X.X.X>
 mpls ldp autoconfig
!
mpls ldp router-id Loopback0 force
mpls label range <min> <max>
```

### CE Routers (7 routers) — Minimal Config (Powered On, Waiting)

CEs have only a loopback and a physical interface toward their PE. No routing protocols. Each lab tells you what to configure on them (eBGP, OSPF, static, or pure L2).

| Router | Loopback | Interface Toward PE | Peer PE | Used By Labs |
|---|---|---|---|---|
| **R1** | 1.1.1.1 | Fa0/0 (→R2) | R2 | 2, 4, 6, 25 |
| **R9** | 9.9.9.9 | Gi1/0 (→R8) | R8 | 2, 4, 6, 25 |
| **R10** | 10.10.10.10 | — | R2 | Topology E: becomes ASBR |
| **R11** | 11.11.11.11 | Fa0/0 (→R8) | R8 | 2 (Customer_C) |
| **R12** | 12.12.12.12 | Gi2/0 (→R2) | R2 | 2 (Customer_B) |
| **R19** | 19.19.19.19 | Fa3/0 (→R17) | R17 | 2 (Customer_D) |
| **R20** | 20.20.20.20 | Gi2/0 (→R18) | R18 | 2 (Customer_E) |

**Config template for CE routers:**
```
hostname R<X>
!
interface Loopback0
 ip address <X.X.X.X> 255.255.255.255
!
interface <pe-facing-intf>
 no ip address
 no shutdown
 ! IP + routing configured per lab
!
no ip routing   ← optional, some labs enable it
```

---

## Snapshot Branching Diagram

```
[OSPF-LDP-Base] ← You are here. Build this FIRST.
    │
    ├──► + VRFs + MP-BGP + PE-CE routing
    │        → Snapshot "02-L3VPN"
    │            │
    │            ├──► + TE tunnels → Snapshot "03-TE"
    │            │        │
    │            │        └──► - TE + OAM/BFD → Lab 7 → Lab 8 → Snapshot "08-Golden"
    │            │                                                    │
    │            │                                                    ├──► Labs 12-15, 19-31
    │            │                                                    └──► Labs 32-34 (power on R21-R26)
    │            │
    │            └──► + RT changes → Lab 6 (Topology C) → restore RTs after
    │
    ├──► + xconnect / L2VPN (no VRFs)
    │        → Labs 4, 5, 11 (Topology B)
    │
    ├──► - OSPF + IS-IS (replace IGP)
    │        → Lab 10 (Topology D)
    │
    └──► + ASBR roles on R10/R11/R12 + power on R21-R26
             → Labs 32-34 (Topology E) — can also branch from "08-Golden"
```

---

## Quick Verification — Base Snapshot Is Correct

After building the base, verify these pass before saving the snapshot:

```
! From any PE (e.g., R2):
show ip ospf neighbor
! Should see neighbors on ALL core-facing interfaces

show mpls ldp neighbor
! Should see LDP sessions to all directly connected P routers

show mpls forwarding-table
! Should see labels for all PE/P loopbacks (2.2.2.2, 3.3.3.3, ..., 18.18.18.18)

! End-to-end label-switched path:
ping 8.8.8.8 source 2.2.2.2
! Should succeed (IP routed via OSPF)

traceroute 8.8.8.8 source 2.2.2.2
! Should show MPLS labels in the path (LSP working)

! CE interfaces should have NO config:
show ip interface brief
! CE-facing interfaces: up/up but no IP address
```
