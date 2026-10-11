# CCIE SP Workbook 08 — L2VPN: VPWS & VPLS (Domain 2)

**Platform:** IOS-XRv 9000 (7.11.1) — EVE-NG
**Topology:** Focus on **Emerald AS 65100** (LDP transport) and **Garnet AS 65200** (SR-MPLS transport) — see `00_EVENG_Topology.md`.
**Format:** Question → Solution → Verification (INE/Narbik style — time yourself per section).

| SP | PEs | Loopbacks | Core Transport |
|----|-----|-----------|----------------|
| Emerald (AS 65100) | E-R1, E-R2 | 1.1.1.1, 2.2.2.2 | LDP (targeted LDP signals PWs) |
| Garnet (AS 65200) | Gar-R1, Gar-R2 | 11.11.11.11, 12.12.12.12 | SR-MPLS (LSP is the SR prefix-SID path) |

> **Scope note:** These loopbacks are the workbook-local L2VPN addressing. The base `00_EVENG_Topology.md` uses different Garnet loopbacks (Gar-R1=24.24.24.24, Gar-R2=25.25.25.25) — if you run this on the full topology, substitute the base loopbacks. All configuration in this workbook uses **IOS-XR `l2vpn` syntax** (there is no `xconnect`-under-interface as in IOS classic; XR uses `l2vpn xconnect group` and `bridge-domain`).

> **Transport-agnostic principle:** VPWS and VPLS are indifferent to how the transport LSP is built. Emerald reaches remote PE loopbacks via **LDP**; Garnet reaches them via **SR-MPLS prefix-SIDs**. The pseudowire's inner **VC label** and the service config are identical either way — only the outer transport label differs. This is the whole point of the MPLS separation of *transport* from *service*.

---

## Section 1 — VPWS / AToM (Point-to-Point Pseudowire)

Point-to-point Ethernet-over-MPLS (E-Line / VPWS) between **E-R1 ↔ E-R2** across the Emerald LDP core. A pseudowire uses a **two-label stack**: outer transport label (LDP/SR) tunnels the frame to the remote PE loopback; inner **VC label** identifies the specific PW at egress. The VC label is signaled by **targeted (directed) LDP** between the two PE loopbacks.

### Task 1.1 — Build the point-to-point pseudowire (VC-ID 100)

**Question**
Configure an AToM pseudowire carrying Customer A's Layer 2 between CE (via E-R1) and CE (via E-R2) using **VC-ID / pw-id 100**, port mode. CEs must reach each other at Layer 2 across the LDP core.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — VC-ID semantics and VLAN-mode PW

**Question**
Explain the role of the VC-ID, then convert VC-100 to **VLAN mode** so a single access port can carry multiple E-Line services, adding VLAN 200 as a second PW.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Control word

**Question**
Enable the **control word** on VC-100 and explain why it matters for Ethernet PWs and for fat-PW / ECMP hashing in the core.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — PW status signaling

**Question**
Enable **PW status signaling** (LDP status TLV) so AC/PW faults propagate end-to-end, and describe the difference vs label withdrawal.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — VPWS Advanced

### Task 2.1 — Pseudowire redundancy (backup PW)

**Question**
Protect the E-R1 service with a **backup pseudowire** to Gar-R2 (12.12.12.12) so that if the primary PW to E-R2 fails, traffic fails over automatically. Configure immediate switchover and restore.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — Preferred-path over a TE tunnel

**Question**
Pin VC-100's transport to a specific **MPLS-TE tunnel** (or SR-TE policy) instead of the IGP-shortest LSP, with fallback disabled so the PW stays down if the tunnel is down.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — Static pseudowire

**Question**
Build a **static (manually-labeled) pseudowire** E-R1↔E-R2 with no targeted-LDP signaling — you assign the VC labels by hand. State when this is used.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — VPLS Full-Mesh (LDP-signaled, RFC 4762)

Multipoint L2 (E-LAN) across the **Emerald PEs (E-R1, E-R2)** using **LDP-signaled VPLS (RFC 4762)**. VPLS makes the SP core behave as one big learning bridge: each PE has a **bridge-domain** containing local ACs plus a **VFI** whose PW neighbors form a full mesh of pseudowires to every other PE.

### Task 3.1 — LDP-signaled VPLS bridge-domain + VFI

**Question**
Build a full-mesh VPLS instance (VPN-ID 500) across E-R1 and E-R2 so all customer sites share one broadcast domain. Use LDP (Martini) signaling.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — Split-horizon (loop prevention in the mesh)

**Question**
Explain and verify **split-horizon** in the VPLS mesh — why VPLS needs no spanning tree in the core.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — MAC learning and MAC limits

**Question**
Verify dynamic **MAC learning** on the bridge-domain and impose a MAC-address limit with an action, to protect against MAC flooding.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4 — BUM flooding behaviour

**Question**
Trace how a **broadcast/unknown-unicast/multicast** frame is handled in the VPLS instance and confirm it reaches all PEs exactly once.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — H-VPLS (Hierarchical VPLS)

H-VPLS reduces the **full-mesh scaling problem** (n PEs need n·(n-1)/2 PWs and n-1 sessions each). It introduces two tiers: **N-PE** (network-facing, in the core full mesh) and **U-PE** (user-facing, at the edge). U-PEs connect to an N-PE by a single **spoke PW**; only N-PEs run the full mesh.

### Task 4.1 — N-PE / U-PE tiers with a spoke PW

**Question**
Make **E-R1 an N-PE** (in the core VPLS mesh) and attach a **U-PE (E-R2 acting as edge)** to it via a single **spoke pseudowire**, so the U-PE needs no full mesh.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — Spoke PW exempt from split-horizon

**Question**
Prove the spoke PW is **not** subject to the mesh split-horizon rule, and explain why that is safe.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Scaling win vs full-mesh

**Question**
Quantify the scaling reduction H-VPLS delivers and describe the redundancy option for the spoke.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — BGP VPLS (Kompella) & EVPN-VPLS

LDP-VPLS (Section 3, RFC 4762) requires **manual full-mesh** neighbor configuration and has no auto-discovery. **BGP-VPLS (Kompella, RFC 4761)** adds **BGP auto-discovery + BGP signaling**: PEs discover each other and exchange PW labels via a **label block**, eliminating manual mesh config. **EVPN-VPLS** is the modern successor, using BGP EVPN (route-types) with control-plane MAC learning.

### Task 5.1 — BGP auto-discovery + signaling VPLS (Kompella)

**Question**
Configure **BGP-VPLS (RFC 4761)** on the Garnet PEs (Gar-R1, Gar-R2) so PWs are auto-discovered and signaled by BGP, using a **label block**. Compare with LDP-VPLS.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — EVPN-VPLS (control-plane MAC learning)

**Question**
Deliver the same multipoint L2 service using **EVPN** (BGP EVPN) instead of LDP/BGP-VPLS, so MAC addresses are learned in the **control plane** and advertised as **Type-2** routes.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3 — Compare LDP-VPLS vs BGP-VPLS vs EVPN

**Question**
Summarize the trade-offs so you can justify a choice in the exam.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — Troubleshooting

### Task 6.1 — PW down: VC-ID / pw-id mismatch

**Question**
A VPWS PW E-R1↔E-R2 is **down**. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — PW up but no traffic: MTU or control-word mismatch

**Question**
The PW shows **UP** on both ends but the CEs cannot pass traffic (or only small frames pass). Diagnose.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.3 — VPLS MAC not learned: split-horizon block or AC down

**Question**
In the VPLS instance, a remote site's MAC never appears in E-R1's MAC table and its traffic is missing. Diagnose.


> *Try this yourself first. Solution available in `solutions/` folder.*

