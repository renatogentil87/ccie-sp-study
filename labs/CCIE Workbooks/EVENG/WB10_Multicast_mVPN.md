# CCIE SP Workbook 10 — Multicast & mVPN Profiles (Domain 2)

**Platform:** EVE-NG (IOS-XRv 9000 7.11.1)
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** All 3 ISPs (Emerald AS 65100 + Garnet AS 65200 + Gold). Multicast **source in Emerald (CE1, 11.11.11.11)**; **receivers in Garnet (CE4, 32.32.32.32)** and **Gold (CE9)**.
**Format:** Question → Solution → Verification.

> **Topology note:** The base `00_EVENG_Topology.md` documents two SPs (Emerald + Garnet). This workbook follows the task spec, which introduces a third provider **"Gold"** and a receiver **CE9**. Where Gold/CE9 are used they are a documented extension of the base topology (Gold ≈ a third IOS-XR domain peering via a third ASBR); Emerald/Garnet node names, loopbacks and ASNs match the base topology. **RP for the Emerald core = E-R5 (6.6.6.6)** per the task.
>
> **All syntax is IOS-XR** (`multicast-routing`, `router pim`, `router msdp`, `mdt` under the VRF, `router bgp … address-family ipv4 mvpn`). CE nodes (CSR1000v / IOS-XE) use classic IOS multicast syntax where noted.

---

## Section 1 — PIM Basics (4 tasks)

### Task 1.1 — PIM-SM on the Emerald core (RP = E-R5 6.6.6.6, static RP)

**Question**
Enable global multicast routing and PIM sparse-mode on all Emerald core interfaces (E-R1, E-R3, E-R4, E-R6) and set a **static RP = E-R5 (6.6.6.6)**. Advertise 6.6.6.6 as the RP so E-R1 can register the CE1 source and PE-side receivers can build the shared tree.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — PIM-SSM for the 232.0.0.0/8 range

**Question**
Enable **PIM-SSM** for the standard SSM range **232.0.0.0/8** across the Emerald core so a receiver joining `(S,232.x.x.x)` builds a source-tree directly with **no RP and no shared tree**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Static RP vs Auto-RP vs BSR

**Question**
Compare and demonstrate the three RP-learning mechanisms on the Emerald core. Keep **static RP** as baseline, then show the **Auto-RP** (Cisco) and **BSR** (RFC 5059) alternatives and when to use each.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Verify the mroute table: (S,G) and (\*,G) entries

**Question**
With the CE1 source active to an ASM group (e.g. `239.1.1.1`) and a receiver on a PE, read the **MRIB/MFIB** and explain each entry: the shared tree `(*,G)`, the source tree `(S,G)`, RPF interface, and incoming/outgoing lists. Show the SPT switchover.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — MSDP (2 tasks)

### Task 2.1 — MSDP peering between Emerald RP and Garnet RP

**Question**
The Emerald RP (E-R5, 6.6.6.6) and the Garnet RP (P3/RR2, 23.23.23.23) each serve their **own PIM-SM domain**. Configure **MSDP** between them so a source active in Emerald is learned by the Garnet RP, enabling **inter-domain ASM** (Emerald source → Garnet receiver on CE4).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — SA messages and cache

**Question**
Bring up the CE1 source to an ASM group and confirm the **SA message** propagates from the Emerald RP to the Garnet RP, that the Garnet RP builds `(S,G)` state, and that CE4 receives traffic. Verify the SA cache and SA RPF.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — mVPN Profile 0 (Default MDT with GRE / PIM-GRE) (3 tasks)

### Task 3.1 — Configure the Default MDT with GRE under the VRF (default-group)

**Question**
For **Customer A VRF** (CE1 on E-R1 ↔ CE4 on Gar-R1), build **Profile 0**: a **Default MDT** using **GRE encapsulation** with **PIM** in the core. Configure the **default-group 239.100.0.0** on E-R1 and Gar-R1 so the PEs form a full-mesh MDT and exchange customer multicast in-band over PIM/GRE.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — Data MDT with a threshold (data-group / S-PMSI)

**Question**
Add a **Data MDT** so a **high-bandwidth** C-stream is moved off the Default MDT onto a dedicated data-group, and **only PEs with interested receivers** join it. Use data-group pool **239.101.0.0/24** with a **threshold of 10 kbps**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — Verify encapsulation / decapsulation end-to-end

**Question**
Prove the **GRE encap on the ingress PE and decap on the egress PE**: C-multicast from CE1 is GRE-encapsulated into the MDT group on E-R1 and decapsulated on Gar-R1 before delivery to CE4.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — mVPN Profile 3 (BGP AD + PIM C-signaling) (3 tasks)

### Task 4.1 — Enable BGP Auto-Discovery with MVPN NLRI

**Question**
Convert Customer A to **Profile 3**: keep **GRE** transport and **PIM** for C-multicast signaling, but replace PIM-based PE discovery with **BGP Auto-Discovery** using the **`ipv4 mvpn`** address-family. Enable `ipv4 mvpn` on E-R1, Gar-R1 and the RR (P2/RR1, 4.4.4.4).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — Source-active signaling (Type 5) and C-multicast over PIM

**Question**
Bring up the CE1 source and confirm the **MVPN Type 5 (Source Active A-D)** route is originated, while C-joins are still handled by **PIM**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Compare Profile 3 with Profile 0

**Question**
Summarize the difference between Profile 0 and Profile 3 and why you would move from 0 to 3.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — mVPN Profile 11 (BGP AD + BGP C-signaling, P-tunnel = mLDP) (3 tasks)

### Task 5.1 — Fully BGP-based mVPN with mLDP P-tunnels

**Question**
Migrate Customer A to **Profile 11**: **no PIM anywhere in the core**, discovery **and** C-multicast signaling entirely via **BGP MVPN**, with the **P-tunnel built by mLDP** (P2MP LSP). Enable `mpls mldp` on the core and BGP-driven overlay.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — Verify the BGP MVPN route types (Type 1–7)

**Question**
Enumerate and verify the **7 BGP MVPN route types** (RFC 6514) present in Profile 11 and state what each does.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3 — Confirm no core PIM and end-to-end delivery

**Question**
Prove that the P-network carries multicast with **labels only** (mLDP) and that CE1→CE4 delivery works with **no PIM state on any P router**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — mVPN Profile 12 / 14 (mLDP profiles, partitioned MDT, inter-AS) (3 tasks)

### Task 6.1 — Profile 12: shared mLDP P2MP MDT, default vs data MDT

**Question**
Configure **Profile 12** (mLDP P2MP, BGP A-D + BGP signaling, **shared** default MDT) for Customer A and add a **data MDT**. Contrast the default MDT (all PEs) with the data MDT (only interested PEs).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — Profile 14: partitioned MDT (per-source-PE trees)

**Question**
Configure **Profile 14** (**Partitioned MDT**, mLDP P2MP, BGP). Show that each **source PE builds its own P2MP tree** and receiver PEs join **only** the trees that have active sources — contrast with Profile 12's shared tree.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.3 — Inter-AS mVPN concepts (Emerald ↔ Garnet ↔ Gold)

**Question**
Describe and stage **inter-AS mVPN** so the CE1 source in Emerald reaches receivers in Garnet (CE4) and Gold (CE9). Cover the role of **Type 2 (Inter-AS I-PMSI A-D)** and ASBR behaviour.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 7 — SR-MVPN (Tree-SID) (2 tasks)

### Task 7.1 — MVPN with SR-MPLS transport (Tree-SID / P2MP SR Policy)

**Question**
Carry Customer A mVPN over **SR-MPLS** using a **Tree-SID** (multicast **P2MP SR Policy**) instead of mLDP, with the **SR-PCE (E-R5, 6.6.6.6)** computing the P2MP tree. Keep BGP MVPN for A-D/signaling.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 7.2 — Concept of SR replication segments

**Question**
Explain **SR replication segments** and how they compose a Tree-SID P2MP tree; contrast with mLDP.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 8 — Troubleshooting (3 tasks)

### Task 8.1 — No multicast in the VRF (missing `mdt default`)

**Question**
Customer A receiver CE4 gets **no traffic**. Unicast VPN works, PIM on the CE side is fine, but no C-multicast crosses the core. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 8.2 — Source not registered with the RP

**Question**
CE1 is sending to an ASM group but receivers get nothing and the RP shows no source. `show mrib route` on the RP has no `(S,G)`. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 8.3 — Inter-AS multicast broken

**Question**
Emerald source reaches Garnet fine, but the **Gold** receiver (CE9) — reached across an inter-AS boundary — gets nothing. Inter-domain / inter-AS multicast is broken. Diagnose and fix for both the ASM/MSDP case and the mVPN case.


> *Try this yourself first. Solution available in `solutions/` folder.*

