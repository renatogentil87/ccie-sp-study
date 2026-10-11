# CCIE SP Workbook 01 — IS-IS Foundation

**Domain:** Core Routing (25% of CCIE SP lab)
**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `../../configuration/EVENG/00_topology_reference.md` — Emerald AS 65100 + Gold AS 65300 + Garnet AS 65200 (3 ISPs)
**Format:** ccielabpass DOO — every task is **Question → Solution → Verification**

> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3 (NIC0/NIC1 = internal/mgmt)

---

## Topology Quick-Reference (used by every task)

### Node inventory + NET-ID derivation

NET-ID format: `49.<area>.<system-id>.00`. The **system-id** is derived from the Loopback0 IPv4 address by padding each octet to 3 digits and regrouping into three 4-hex-digit blocks (the classic "IP-to-NET" method).

Example — E-R1 Loopback0 `1.1.1.1` → `001.001.001.001` → `0010.0100.1001`.

| SP (area) | Node | Role | Loopback0 | Derived NET-ID |
|-----------|------|------|-----------|----------------|
| **Emerald** 49.0001 | E-R1 | PE | 1.1.1.1 | `49.0001.0010.0100.1001.00` |
| | E-R2 | PE | 2.2.2.2 | `49.0001.0020.0200.2002.00` |
| | E-R3 | P | 3.3.3.3 | `49.0001.0030.0300.3003.00` |
| | E-R4 | P | 4.4.4.4 | `49.0001.0040.0400.4004.00` |
| | E-R6 | ASBR | 5.5.5.5 | `49.0001.0050.0500.5005.00` |
| | E-R5 | RR+Gar-R6 | 6.6.6.6 | `49.0001.0060.0600.6006.00` |
| **Garnet** 49.0002 | Gar-R1 | PE | 11.11.11.11 | `49.0002.0110.1101.1011.00` |
| | Gar-R2 | PE | 12.12.12.12 | `49.0002.0120.1201.2012.00` |
| | Gar-R3 | P | 13.13.13.13 | `49.0002.0130.1301.3013.00` |
| | Gar-R4 | P | 14.14.14.14 | `49.0002.0140.1401.4014.00` |
| | Gar-R5 | P | 15.15.15.15 | `49.0002.0150.1501.5015.00` |
| | Gar-R7 | ASBR | 17.17.17.17 | `49.0002.0160.1601.6016.00` |
| | Gar-R6 | RR+Gar-R6 | 16.16.16.16 | `49.0002.0170.1701.7017.00` |
| **Gold** 49.0003 | G-R4 | ASBR+RR+Gar-R6 | 24.24.24.24 | `49.0003.0210.2102.1021.00` |
| | G-R5 | ASBR | 25.25.25.25 | `49.0003.0220.2202.2022.00` |
| | G-R3 | P | 23.23.23.23 | `49.0003.0230.2302.3023.00` |
| | G-R1 | PE | 21.21.21.21 | `49.0003.0240.2402.4024.00` |
| | G-R2 | PE | 22.22.22.22 | `49.0003.0250.2502.5025.00` |

### Per-router core (IS-IS) interfaces — from the link map

| SP | Node | Core Gi interfaces (run IS-IS) | Non-core (NO IS-IS) |
|----|------|--------------------------------|---------------------|
| Emerald | E-R1 | Gi0/0/0/2 (→E-R3), Gi0/0/0/3 (→E-R2) | Gi0/0/0/0 (CE1), Gi0/0/0/1 (CE2) |
| Emerald | E-R2 | Gi0/0/0/0 (→E-R3), Gi0/0/0/3 (→E-R1) | Gi0/0/0/1 (CE2), Gi0/0/0/2 (CE3) |
| Emerald | E-R3 | Gi0/0/0/0 (→E-R2), Gi0/0/0/1 (→E-R4), Gi0/0/0/2 (→E-R1) | — |
| Emerald | E-R4 | Gi0/0/0/1 (→E-R3), Gi0/0/0/2 (→E-R6), Gi0/0/0/3 (→E-R5) | — |
| Emerald | E-R6 | Gi0/0/0/2 (→E-R4) | Gi0/0/0/1 (inter-AS Garnet), Gi0/0/0/3 (inter-AS Gold) |
| Emerald | E-R5 | Gi0/0/0/3 (→E-R4) | — |
| Garnet | Gar-R1 | Gi0/0/0/1 (→Gar-R4), Gi0/0/0/3 (→Gar-R2) | Gi0/0/0/2 (CE4), Gi0/0/0/0 (CE5) |
| Garnet | Gar-R2 | Gi0/0/0/2 (→Gar-R5), Gi0/0/0/3 (→Gar-R1) | Gi0/0/0/1 (CE5), Gi0/0/0/0 (CE6) |
| Garnet | Gar-R3 | Gi0/0/0/0 (→Gar-R4), Gi0/0/0/1 (→Gar-R5), Gi0/0/0/2 (→Gar-R7), Gi0/0/0/3 (→Gar-R6) | — |
| Garnet | Gar-R4 | Gi0/0/0/0 (→Gar-R3), Gi0/0/0/1 (→Gar-R1), Gi0/0/0/3 (→Gar-R5) | — |
| Garnet | Gar-R5 | Gi0/0/0/1 (→Gar-R3), Gi0/0/0/2 (→Gar-R2), Gi0/0/0/3 (→Gar-R4) | — |
| Garnet | Gar-R7 | Gi0/0/0/2 (→Gar-R3) | Gi0/0/0/1 (inter-AS Emerald), Gi0/0/0/3 (inter-AS Gold) |
| Garnet | Gar-R6 | Gi0/0/0/3 (→Gar-R3) | — |
| Gold | G-R4 | Gi0/0/0/1 (→G-R3), Gi0/0/0/2 (→G-R5) | Gi0/0/0/3 (inter-AS Emerald) |
| Gold | G-R5 | Gi0/0/0/0 (→G-R3), Gi0/0/0/2 (→G-R4) | Gi0/0/0/3 (inter-AS Garnet) |
| Gold | G-R3 | Gi0/0/0/0 (→G-R5), Gi0/0/0/1 (→G-R4), Gi0/0/0/2 (→G-R1), Gi0/0/0/3 (→G-R2) | — |
| Gold | G-R1 | Gi0/0/0/2 (→G-R3) | Gi0/0/0/0 (CE9), Gi0/0/0/1 (CE8) |
| Gold | G-R2 | Gi0/0/0/3 (→G-R3) | Gi0/0/0/0 (CE7), Gi0/0/0/1 (CE8) |

### Core link subnets (for interface addressing)

| SP | Link | Subnet | Addresses |
|----|------|--------|-----------|
| Emerald | E-R5(Gi3)–E-R4(Gi3) | 10.1.1.0/24 | E-R5 .6, E-R4 .4 |
| Emerald | E-R4(Gi2)–E-R6(Gi2) | 10.1.2.0/24 | E-R4 .4, E-R6 .5 |
| Emerald | E-R4(Gi1)–E-R3(Gi1) | 10.1.3.0/24 | E-R4 .4, E-R3 .3 |
| Emerald | E-R3(Gi2)–E-R1(Gi2) | 10.1.4.0/24 | E-R3 .3, E-R1 .1 |
| Emerald | E-R3(Gi0)–E-R2(Gi0) | 10.1.5.0/24 | E-R3 .3, E-R2 .2 |
| Emerald | E-R1(Gi3)–E-R2(Gi3) | 10.1.6.0/24 | E-R1 .1, E-R2 .2 |
| Garnet | Gar-R6(Gi3)–Gar-R3(Gi3) | 10.2.1.0/24 | Gar-R6 .17, Gar-R3 .13 |
| Garnet | Gar-R7(Gi2)–Gar-R3(Gi2) | 10.2.2.0/24 | Gar-R7 .16, Gar-R3 .13 |
| Garnet | Gar-R3(Gi0)–Gar-R4(Gi0) | 10.2.3.0/24 | Gar-R3 .13, Gar-R4 .14 |
| Garnet | Gar-R3(Gi1)–Gar-R5(Gi1) | 10.2.4.0/24 | Gar-R3 .13, Gar-R5 .15 |
| Garnet | Gar-R4(Gi3)–Gar-R5(Gi3) | 10.2.5.0/24 | Gar-R4 .14, Gar-R5 .15 |
| Garnet | Gar-R4(Gi1)–Gar-R1(Gi1) | 10.2.6.0/24 | Gar-R4 .14, Gar-R1 .11 |
| Garnet | Gar-R5(Gi2)–Gar-R2(Gi2) | 10.2.7.0/24 | Gar-R5 .15, Gar-R2 .12 |
| Garnet | Gar-R1(Gi3)–Gar-R2(Gi3) | 10.2.8.0/24 | Gar-R1 .11, Gar-R2 .12 |
| Gold | G-R4(Gi2)–G-R5(Gi2) | 10.3.1.0/24 | G-R4 .21, G-R5 .22 |
| Gold | G-R4(Gi1)–G-R3(Gi1) | 10.3.2.0/24 | G-R4 .21, G-R3 .23 |
| Gold | G-R5(Gi0)–G-R3(Gi0) | 10.3.3.0/24 | G-R5 .22, G-R3 .23 |
| Gold | G-R3(Gi2)–G-R1(Gi2) | 10.3.4.0/24 | G-R3 .23, G-R1 .24 |
| Gold | G-R3(Gi3)–G-R2(Gi3) | 10.3.5.0/24 | G-R3 .23, G-R2 .25 |

> **IPv6 core convention (Task 1.5):** mirror the IPv4 subnet as `2001:db8:1:X::/64` (Emerald), `2001:db8:2:X::/64` (Garnet), `2001:db8:3:X::/64` (Gold), host bits = the router's loopback octet. Loopback0 IPv6 = `2001:db8::<loop>/128` (e.g., E-R1 = `2001:db8::1/128`).

---

# Section 1 — IS-IS Basic Setup (8 tasks)

---

## Task 1.1 — Enable IS-IS L2-only on all Emerald routers

### Question
Configure IS-IS process **CORE** as **Level-2-only** on all six Emerald routers (E-R1, E-R2, E-R3, E-R4, E-R6, E-R5). Derive each NET-ID from the router's Loopback0 (`49.0001.xxxx.xxxx.xxxx.00`). Use **metric-style wide**. Run IS-IS on Loopback0 and the core interfaces only (see the per-router table); do **not** enable it on PE-CE or inter-AS interfaces.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.2 — Enable IS-IS L2-only on all Garnet routers

### Question
Same as 1.1 but for the seven Garnet routers (Gar-R1, Gar-R2, Gar-R3, Gar-R4, Gar-R5, Gar-R7, Gar-R6) using area **49.0002**. NET-ID derived from Loopback0, `metric-style wide`, core interfaces only.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.3 — Enable IS-IS L2-only on all Gold routers

### Question
Same pattern for the five Gold routers (G-R4, G-R5, G-R3, G-R1, G-R2), area **49.0003**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.4 — CCIE-ISIS flexible configuration group (regex interface matching)

### Question
Rather than repeat interface knobs on every router, build a **flexible CLI configuration group** named `CCIE-ISIS` that, via a **regex** matching all `GigabitEthernet` interfaces under `router isis CORE`, applies: L2 metric **200** (IPv4) and **400** (IPv6), **hello-padding disable**, **point-to-point**; and, matching `Loopback`, sets **passive**. Apply the group and confirm the inherited config.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.5 — IPv6 address-family with single-topology (all 3 IS-IS instances)

### Question
Enable the **IPv6 unicast** address-family in all three `router isis CORE` instances using **single-topology** (IPv6 shares the IPv4 SPF/topology). Add IPv6 addressing on Loopback0 and all core interfaces. Show the config on one router per SP; the pattern repeats.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.6 — set-overload-bit on-startup 180 level-2 (all routers)

### Question
On **every** router in all three SPs, set the IS-IS overload bit on startup for **180 seconds** at **level-2**, so the node is not used as transit until it has fully converged (BGP, LDP/SR, LSDB). Show the config; it is identical on all nodes.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.7 — LDP auto-config under IS-IS (Emerald only)

### Question
Emerald uses **LDP** for transport (Gold=SRv6, Garnet=SR-MPLS — no LDP there). Enable **LDP IGP auto-config** under the Emerald IS-IS instance so LDP is enabled automatically on every IS-IS core interface. Configure MPLS LDP on the six Emerald routers.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.8 — LDP–IGP synchronization (Emerald)

### Question
Prevent traffic blackholing when an LDP session is down but the IS-IS adjacency is up: enable **LDP–IGP synchronization** at **level-2** on Emerald. Until LDP synchronizes on a link, IS-IS should advertise that link with **max metric** so it isn't used as a labeled transit path.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.1 — IS-IS authentication (key-chain, HMAC-MD5), per-level + per-interface (Emerald)

### Question
Secure Emerald IS-IS with **HMAC-MD5** authentication using a **key-chain**. Apply authentication two ways: **per-level** (LSP/SNP authentication for the whole L2 domain) and **per-interface** (hello authentication on a specific link). All Emerald routers must share the key.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.2 — IS-IS prefix suppression on core P-P links

### Question
Reduce the LSDB and speed SPF by **suppressing transit (P-to-P core link) prefixes** from IS-IS — advertise only loopbacks. Configure prefix suppression on all routers (global) and confirm core /24s no longer appear as IS-IS routes, while loopbacks still do.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.3 — Multi-topology vs single-topology (explain + configure single)

### Question
Explain **when** to use IS-IS multi-topology (MT) versus single-topology, then confirm all three instances run **single-topology** (as set in Task 1.5).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.4 — Route leaking between levels (concept — N/A for L2-only)

### Question
Explain IS-IS inter-level **route leaking**, why it exists, and why it is **not applicable** in this lab (everything is Level-2-only).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.5 — IS-IS BFD for fast adjacency failure detection (all core links)

### Question
Enable **BFD** for IS-IS on all core links in all three SPs so link/neighbor failures are detected in milliseconds instead of waiting for IS-IS hold timers. Use 300 ms intervals, multiplier 3 (≈900 ms detection).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.6 — IS-IS mesh-group on fully-meshed segments (reduce flooding)

### Question
On a **fully-meshed** set of routers, LSP flooding is redundant (every router floods to every neighbor). Configure an IS-IS **mesh-group** to suppress redundant flooding. Apply it to the Emerald E-R1–E-R2–E-R3 near-mesh (E-R1↔E-R2, E-R1↔E-R3, E-R2↔E-R3 all adjacent).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.1 — Full verification checklist

### Question
Produce a repeatable IS-IS health-check using the four core show commands and state what "good" looks like for each.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.2 — Troubleshooting: adjacency stuck in INIT

### Question
An IS-IS adjacency will not reach `Up` — it sits in **`Init`** (or flaps). Diagnose the three classic causes: **MTU mismatch**, **area/level mismatch**, **authentication mismatch**. Show how to identify and fix each.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.3 — Troubleshooting: routes not appearing

### Question
Adjacencies are `Up` but some prefixes are missing from the routing table. Diagnose the two classic causes: **passive-interface misconfiguration** and a **core interface missing from IS-IS**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.4 — IS-IS convergence timing (SPF / LSP intervals, sub-second tuning)

### Question
Tune IS-IS for **sub-second convergence**: exponential **SPF backoff**, **LSP generation** backoff, and **LSP flooding/refresh** timers. Explain each timer and give an aggressive-but-stable set for the lab core.


> *Try this yourself first. Solution available in `solutions/` folder.*

