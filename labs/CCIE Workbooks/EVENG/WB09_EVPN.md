# CCIE SP Workbook 09 — EVPN (Domain 2)

**Platform:** EVE-NG — IOS-XRv 9000
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology focus:**
- **Garnet** — Gar-R1 + Gar-R2 + CE5. CE5 is **dual-homed** to Gar-R1 and Gar-R2 (EVPN, VLAN 100).
- **Gold** — G-R2 + CE7. CE7 is single-homed to G-R2 (EVPN, VLAN 100).
- Inter-AS: **Gar-R6 (Garnet RR) ↔ G-R4** carries BGP EVPN toward Gold, so CE5 (Garnet) can reach CE7 (Gold).

**Reference addressing (used throughout):**

| Node | Loopback0 | Role |
|------|-----------|------|
| Gar-R1  | 3.3.3.3   | Garnet PE, DF candidate for CE5 ES |
| Gar-R2  | 4.4.4.4   | Garnet PE, DF candidate for CE5 ES |
| G-R2  | 6.6.6.6   | Gold PE (CE7) |
| Gar-R6  | 9.9.9.9   | Garnet route-reflector for `l2vpn evpn` |
| G-R4| 13.13.13.13 | Inter-AS EVPN border |

- **CE5 Ethernet Segment ID (ESI):** `0000.0000.0000.0000.0005`
- **EVI 100** = bridged VLAN 100 (MAC-VRF), **EVI 500** = EVPN-VPWS, **EVI 200** = IRB instance
- **BD (bridge-domain) `VLAN100`** maps to EVI 100
- **IP-VRF `TENANT-A`** for symmetric IRB / Type-5 (L3VNI-equivalent label)

**Prerequisites (assumed done in earlier workbooks):**
- IS-IS + Segment Routing MPLS core (Workbook 09-SR), `/32` loopbacks, SRGB 16000–23999.
- iBGP to Gar-R6 (RR) already up for `vpnv4`; we add the `l2vpn evpn` AF here.

> **Convention:** each task is **Question → Solution → Verification**. All syntax is IOS-XR (`evpn`, `l2vpn bridge-domain`, `evi`).

---

## Section 1 — EVPN Route Types (5 tasks)

EVPN is a BGP AF (`l2vpn evpn`) that carries five NLRI route types. Each solves a specific problem. Configure the base MAC-VRF first, then observe each route type.

### Base configuration — MAC-VRF (EVI 100) on Gar-R1, Gar-R2, G-R2

```
! ---- Gar-R1 / Gar-R2 / G-R2 : enable EVPN AF in BGP toward RR (Gar-R6) ----
router bgp 100
 address-family l2vpn evpn
 !
 neighbor 9.9.9.9
  remote-as 100
  update-source Loopback0
  address-family l2vpn evpn
 !
!
! ---- Gar-R6 (RR) : reflect l2vpn evpn ----
router bgp 100
 address-family l2vpn evpn
 neighbor-group RRC
  remote-as 100
  update-source Loopback0
  address-family l2vpn evpn
   route-reflector-client
```

```
! ---- Bridge-domain + EVI mapping (Gar-R1, Gar-R2, G-R2) ----
l2vpn
 bridge group GARNET
  bridge-domain VLAN100
   interface GigabitEthernet0/0/0/1.100      ! CE-facing (dot1q 100)
   !
   evi 100
   !
!
evpn
 evi 100
  bgp
   route-target import  100:100
   route-target export  100:100
  !
  advertise-mac
```

```
! ---- CE-facing sub-interface (Gar-R1/Gar-R2 -> CE5, G-R2 -> CE7) ----
interface GigabitEthernet0/0/0/1.100 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
```

---

### Task 1.1 — Type 2 (MAC/IP Advertisement)

**Question:** CE5 sends a frame with source MAC `0050.5600.0005`. Configure Gar-R1 so it advertises this MAC (and its IP, if ARP is snooped) to remote PEs via BGP instead of relying on data-plane flooding. Explain the purpose of Type 2.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — Type 3 (Inclusive Multicast Ethernet Tag)

**Question:** Configure/verify how BUM (Broadcast, Unknown-unicast, Multicast) traffic is delivered in EVI 100 across Gar-R1, Gar-R2, G-R2. Explain the purpose of Type 3.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Type 4 (Ethernet Segment Route → DF election)

**Question:** CE5 is dual-homed to Gar-R1 and Gar-R2 with the same ESI. Configure the Ethernet Segment so Gar-R1 and Gar-R2 discover each other and elect a **Designated Forwarder** (DF). Explain the purpose of Type 4.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Type 1 (Ethernet Auto-Discovery: per-ES and per-EVI)

**Question:** Explain and verify the two Type 1 (Ethernet A-D) route flavors on the CE5 segment: **per-ES** (mass withdrawal) and **per-EVI** (aliasing). Which config produces each?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.5 — Type 5 (IP Prefix Route)

**Question:** Configure Type 5 so a remote PE can reach a subnet behind CE5 by **IP prefix** (no MAC in the same subnet required). Explain the purpose of Type 5 vs Type 2.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — EVPN-VPWS (3 tasks)

EVPN-VPWS is point-to-point (E-Line) using EVPN Type-1 A-D routes for signaling instead of targeted LDP. Uses `l2vpn xconnect` + `evpn evi ... vpws`.

### Task 2.1 — Point-to-point EVPN-VPWS (single-homed)

**Question:** Build a single-homed EVPN-VPWS between Gar-R1 (toward CE5's data VLAN) and G-R2 (toward CE7). Use EVI 500. Explain how the service is signaled.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — EVPN-VPWS over SR-MPLS transport (Garnet)

**Question:** Ensure the EVI 500 pseudowire from Gar-R1 rides an **SR-MPLS** LSP (prefix-SID transport) rather than LDP. Optionally steer it into an SR-TE policy. Explain the transport-vs-service label stack.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — Compare with traditional (LDP/AToM) VPWS

**Question:** Contrast EVPN-VPWS with legacy AToM/L2VPN VPWS. What operational advantages justify EVPN-VPWS?


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — EVPN Multi-Homing (5 tasks)

CE5 dual-homed to Gar-R1+Gar-R2. Section 1.3 built the Ethernet Segment; here we exercise the redundancy modes and their control-plane mechanics.

### Task 3.1 — Ethernet Segment config on Gar-R1 + Gar-R2 for CE5

**Question:** Finalize a consistent Ethernet Segment for CE5 on both PEs, ensuring the ESI matches and the LACP bundle is shared.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — All-active multi-homing

**Question:** Configure CE5's segment for **all-active** so both Gar-R1 and Gar-R2 forward unicast simultaneously. Explain how remote PEs load-balance to both.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — Single-active multi-homing

**Question:** Reconfigure CE5's segment for **single-active** (only one PE forwards; the other is standby). When is this required?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4 — DF election (mod-based / default)

**Question:** Explain and verify the default (mod-based) DF election for the CE5 ES, per EVI. How do you make it deterministic?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.5 — Mass withdrawal on PE failure (Type-1 per-ES route)

**Question:** Simulate Gar-R1's link to CE5 failing. Prove that a **single** Type-1 per-ES A-D withdrawal drains all MACs behind CE5 from remote PEs (fast convergence), rather than per-MAC Type-2 withdrawals.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.6 — Aliasing (Type-1 per-EVI route)

**Question:** Demonstrate **aliasing**: G-R2 load-balances unicast to CE5 across both Gar-R1 and Gar-R2 even for a MAC only Gar-R1 actually learned.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — EVPN IRB (3 tasks)

Integrated Routing and Bridging: a BVI bridges within a subnet and routes between subnets, using an IP-VRF for L3. Uses EVI (L2) + BVI + IP-VRF (L3).

### Task 4.1 — IRB with anycast gateway (L2 + L3 in one EVPN instance)

**Question:** Configure IRB on Gar-R1 and Gar-R2 for EVI 100 so hosts behind CE5 use a **distributed anycast gateway** (same GW IP+MAC on both PEs) and can be both bridged (same subnet) and routed (other subnets).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — Symmetric IRB vs Asymmetric IRB

**Question:** Configure symmetric IRB (route in the ingress VRF, bridge across a common L3 label, route into egress VRF) and contrast with asymmetric IRB.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Verify L2 + L3 coexist in one EVPN instance

**Question:** Prove that within EVI 100 a host can be bridged to another host in VLAN 100 and simultaneously routed to a host in a different subnet, all via one EVPN instance.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — EVPN Inter-AS (3 tasks)

Extend EVPN from **Garnet** (CE5) to **Gold** (CE7) across the AS boundary. BGP EVPN runs between the Garnet RR (Gar-R6) and the Gold border (G-R4).

### Task 5.1 — BGP EVPN session between RRs / borders (Gar-R6 ↔ G-R4)

**Question:** Establish an inter-AS `l2vpn evpn` session so EVPN routes cross from Garnet into Gold. Use Gar-R6 (Garnet RR) ↔ G-R4.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — Type-2 MAC/IP exchange across domains

**Question:** Ensure CE7's MAC (Gold, G-R2) is learned in Garnet and CE5's MAC is learned in Gold, via Type-2 across the AS boundary. Align RTs.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3 — CE5 (Garnet) ↔ CE7 (Gold) end-to-end EVPN

**Question:** Validate CE5 and CE7 communicate at Layer 2 (same VLAN 100 stretched) and/or Layer 3 across the AS boundary.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — MAC Mobility + Troubleshooting (3 tasks)

### Task 6.1 — MAC mobility (sequence number) + sticky MAC

**Question:** A host with MAC `0050.5600.00AA` moves from behind CE5 (Gar-R1) to behind G-R2. Show how EVPN converges via the MAC Mobility extended community (sequence number), and how to pin a MAC with **sticky/static** MAC.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — Duplicate MAC detection

**Question:** A MAC flaps rapidly between Gar-R1 and G-R2 (misconfig / loop). Show EVPN's duplicate-MAC detection and how to tune/clear it.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.3 — Troubleshooting scenarios

**Question:** Diagnose and fix two classic EVPN faults:
(a) **EVPN routes not received** on a PE (nothing in `show bgp l2vpn evpn`).
(b) **DF election is wrong / duplicate BUM** to CE5.


> *Try this yourself first. Solution available in `solutions/` folder.*

