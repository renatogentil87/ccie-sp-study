# CCIE SP Workbook 03 — BGP Advanced

**Domain:** Domain 1 — Core Routing (25%)
**Platform:** IOS-XRv 9000 7.11.1 (EVE-NG)
**Focus:** iBGP + Route Reflectors, eBGP Inter-AS, path manipulation, advanced BGP features, troubleshooting
**Prereq:** IGP (IS-IS L2) + MPLS transport converged inside each AS; Loopback0 reachable within each AS.

> **Note:** This workbook drills the full BGP control plane for a 3-ISP interconnect: hierarchical iBGP with RRs carrying VPNv4, inter-AS eBGP with multihop VPNv4 between RRs (Option C-style), best-path manipulation for transit engineering, and the scaling/convergence features (Add-Path, PIC, AIGP, RT-Constraint, conditional advertisement) that make a real SP core resilient. **All route-policies are IOS-XR RPL** (`route-policy … if/then/else … set … end-policy`).

---

## Topology

Three service providers, each an autonomous system, interconnected at their ASBRs.

```
        EMERALD  AS 65100                    GOLD  AS 65300                    GARNET  AS 65200
   RR = E-R5 (6.6.6.6)                 RR = G-R4 (24.24.24.24)           RR = Gar-R6 (16.16.16.16)

   E-R1 1.1.1.1                          G-R4 24.24.24.24 (RR)             Gar-R1 11.11.11.11
   E-R2 2.2.2.2                          G-R5 25.25.25.25                  Gar-R2 12.12.12.12
   E-R3, E-R4                               G-R3                                 Gar-R3, Gar-R4, Gar-R5
   E-R6 5.5.5.5                        G-R1 21.21.21.21                    Gar-R7 17.17.17.17
   E-R5 6.6.6.6 (RR)                    G-R2 22.22.22.22

                        E-R6 ══════════════ G-R4        (Emerald ↔ Gold,   eBGP)
                        E-R6 ══════════════ Gar-R7        (Emerald ↔ Garnet, eBGP, direct)
                        G-R5 ══════════════ Gar-R7        (Gold   ↔ Garnet,  eBGP)
```

| SP | AS | RR | PEs | ASBRs |
|----|----|----|-----|-------|
| Emerald | 65100 | E-R5 (6.6.6.6) | E-R1 (1.1.1.1), E-R2 (2.2.2.2) | E-R6 (5.5.5.5) |
| Gold | 65300 | G-R4 (24.24.24.24) | G-R1 (21.21.21.21), G-R2 (22.22.22.22) | G-R4 (24.24.24.24), G-R5 (25.25.25.25) |
| Garnet | 65200 | Gar-R6 (16.16.16.16) | Gar-R1 (11.11.11.11), Gar-R2 (12.12.12.12) | Gar-R7 (17.17.17.17) |

**Customers:** A (AS 65012) = Emerald + Gold; B (AS 65013) = Gold + Garnet; C (EVPN) = Gold + Garnet.

**Inter-AS links (example addressing):**

| Link | Left | Right |
|------|------|-------|
| E-R6 ↔ Gar-R7 | 10.0.12.5/30 | 10.0.12.6/30 |
| E-R6 ↔ G-R4 | 10.0.13.5/30 | 10.0.13.21/30 |
| G-R5 ↔ Gar-R7 | 10.0.42.22/30 | 10.0.42.16/30 |

---

## Section 1 — iBGP + Route Reflectors

### Task 1.1
- Design the iBGP mesh for **Emerald** as a route-reflector topology: **E-R5 (6.6.6.6)** is the RR; E-R1, E-R2, and E-R6 are RR clients.
- All iBGP sessions peer on **Loopback0** with **update-source Loopback0**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2
- Repeat the RR design for **Gold** (RR = **G-R4 24.24.24.24**; clients G-R5, G-R1, G-R2, G-R3) and **Garnet** (RR = **Gar-R6 16.16.16.16**; clients Gar-R1, Gar-R2, Gar-R7).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3
- Add the **VPNv4 address family** on every Emerald RR/PE and enable `route-reflector-client` under `address-family vpnv4 unicast` so L3VPN routes are reflected.
- Enable `retain route-target all` on the RR so it keeps VPNv4 routes even without a matching import RT locally.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4
- Set a **cluster-id** on each RR so a future second RR in the same cluster shares it (cluster-list loop prevention).
- Emerald cluster-id `6.6.6.6`; Gold `24.24.24.24`; Garnet `16.16.16.16`.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.5
- Configure **next-hop-self** so every PE sets itself as the BGP next-hop for routes it originates into iBGP (customer/eBGP routes and VPNv4).
- Confirm iBGP peers resolve the next-hop via the IGP loopback (recursion), not the external link address.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — eBGP Inter-AS

### Task 2.1
- Configure **direct eBGP** on the E-R6 ↔ Gar-R7 link (Emerald AS 65100 ↔ Garnet AS 65200) peering on the **directly-connected interface addresses**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2
- Configure **eBGP E-R6 ↔ G-R4** (Emerald AS 65100 ↔ Gold AS 65300) and **G-R5 ↔ Gar-R7** (Gold AS 65300 ↔ Garnet AS 65200), both directly connected.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3
- Build **multihop eBGP for VPNv4 between the RRs** (Inter-AS Option C-style): E-R5 (Emerald RR, 6.6.6.6) ↔ G-R4 (Gold RR, 24.24.24.24), peering **loopback-to-loopback** across the AS boundary, exchanging **labeled VPNv4** without importing VRFs on the ASBRs.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4
- Harden the multihop RR session with **TTL security** (GTSM) instead of a plain `ebgp-multihop` hop count.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.5
- Contrast when to use **ebgp-multihop** vs **ttl-security** and document the decision for each inter-AS session in this topology.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — BGP Path Manipulation

### Task 3.1
- Engineer Emerald to **prefer the Gold transit path** (via G-R4) over the **direct** E-R6↔Gar-R7 path for reaching Garnet's prefixes, using **LOCAL_PREF**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2
- Make the **direct** E-R6↔Gar-R7 link *less preferred inbound* (so Garnet reaches Emerald via Gold) using **AS-PATH prepend** outbound on the direct link.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3
- Use **MED** to influence **inbound** traffic across the two links to a *single* neighboring AS (Gold), steering Gold to prefer one Emerald entry point.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4
- Implement **community-based routing**: tag Emerald customer routes with a **well-known** community (NO_EXPORT) and an **extended** community, then act on the community at the RR/egress to set LOCAL_PREF.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.5
- Write a consolidated **IOS-XR RPL route-policy** using full `if/elseif/else/then … set … end-policy` structure (with `apply` for a nested policy) that combines the manipulations above, and document the RPL building blocks.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — BGP Advanced Features

### Task 4.1
- Enable **BGP Add-Path** on the Emerald RR (E-R5) so it advertises **best + additional paths** for a multi-homed prefix, restoring path diversity to the clients.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2
- Enable **BGP PIC Edge** for VPNv4 so an egress-PE / next-hop failure fails over in one FIB operation, independent of prefix count.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3
- Configure **AIGP** (Accumulated IGP metric) so the IGP cost is carried and accumulated **across the AS boundary** between the RRs, letting best-path choose the truly lowest end-to-end IGP cost.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.4
- Enable **RT-Constraint (RTC, RFC 4684)** between the RRs so each RR only receives the VPNv4 routes whose route-targets its clients actually import.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.5
- Configure **conditional route advertisement** on E-R6: advertise a backup aggregate to Garnet **only if** the primary Gold-transit path is *not* present (advertise-map / non-exist-map behavior).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — BGP Troubleshooting

### Task 5.1
- **Symptom:** an iBGP/eBGP neighbor is stuck in **Active** (never reaches Established). Diagnose the TCP/transport issue and fix it.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2
- **Symptom:** routes are **received** (present in `show bgp`) but **not installed** in the RIB/FIB. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3
- **Symptom:** VPNv4 routes are **not being reflected** by the RR to its clients. Diagnose the missing address-family / RT-retain issue and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

