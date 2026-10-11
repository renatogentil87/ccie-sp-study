# CCIE SP Workbook 04 — Segment Routing SR-MPLS

**Domain:** 1 — Core Routing (25%)
**Platform:** Cisco IOS-XRv 9000 (EVE-NG / GNS3 on EC2)
**Primary focus:** **Garnet — AS 65200** (IS-IS L2 + SR-MPLS)
**Secondary focus:** **Emerald — AS 65100** (IS-IS L2 + LDP → SR migration)
**Format:** Question → Solution → Verification

---

## Topology Reference — Garnet AS 65200

| Node | Role | Loopback0 | Prefix-SID index (last octet) | Prefix-SID label (SRGB 16000) |
|------|------|-----------|-------------------------------|-------------------------------|
| Gar-R1 | PE | 11.11.11.11 | 11 | 16011 |
| Gar-R2 | PE | 12.12.12.12 | 12 | 16012 |
| Gar-R3 | P | 13.13.13.13 | 13 | 16013 |
| Gar-R4 | P | 14.14.14.14 | 14 | 16014 |
| Gar-R5 | P | 15.15.15.15 | 15 | 16015 |
| Gar-R7 | ASBR | 17.17.17.17 | 16 | 16016 |
| Gar-R6 | RR + Gar-R6 | 16.16.16.16 | 17 | 16017 |

**Garnet core links (from topology reference):**

```
Gar-R6   Gi0/0/0/3 --- 10.2.1.0/24 --- Gi0/0/0/3  Gar-R3
Gar-R7 Gi0/0/0/2 --- 10.2.2.0/24 --- Gi0/0/0/2  Gar-R3
Gar-R3    Gi0/0/0/0 --- 10.2.3.0/24 --- Gi0/0/0/0  Gar-R4
Gar-R3    Gi0/0/0/1 --- 10.2.4.0/24 --- Gi0/0/0/1  Gar-R5
Gar-R4    Gi0/0/0/3 --- 10.2.5.0/24 --- Gi0/0/0/3  Gar-R5
Gar-R4    Gi0/0/0/1 --- 10.2.6.0/24 --- Gi0/0/0/1  Gar-R1
Gar-R5    Gi0/0/0/2 --- 10.2.7.0/24 --- Gi0/0/0/2  Gar-R2
Gar-R1   Gi0/0/0/3 --- 10.2.8.0/24 --- Gi0/0/0/3  Gar-R2
```

**Emerald AS 65100 (LDP → SR migration section):** E-R1 (1.1.1.1), E-R2 (2.2.2.2), E-R3 (3.3.3.3), E-R4 (4.4.4.4), E-R6 (5.5.5.5), E-R5 (6.6.6.6 RR+Gar-R6). Prefix-SID index = last octet of loopback.

> **Prerequisites:** IS-IS Level-2 backbone with **wide metrics** (mandatory for SR sub-TLVs) and /32 loopbacks already configured (Workbook 01). Wide metrics carry the SR Prefix-SID/Adjacency-SID sub-TLVs; narrow metrics cannot.

---

# Section 1 — SR-MPLS Foundation (5 tasks)

## Task 1.1 — Enable Segment Routing under IS-IS

**Question:** Enable Segment Routing MPLS under IS-IS on all Garnet core/PE nodes and confirm the SR data plane is active. No LDP anywhere in Garnet.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.2 — Configure SRGB 16000–23999

**Question:** Set a common **SRGB of 16000–23999** on every Garnet node so prefix-SIDs are globally consistent.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.3 — Prefix-SID index per loopback (match last octet)

**Question:** Assign each node's Loopback0 a **prefix-SID index equal to the last octet** of its loopback (Gar-R3=13 → label 16013, Gar-R6=17 → 16017, etc.). Achieve end-to-end SR reachability with **no LDP**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.4 — Verify the global label table & compare with LDP

**Question:** Show the SR global label table and explain how SR **prefix-SIDs (globally significant)** differ from **LDP labels (locally significant)**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.5 — Adjacency-SIDs (protected vs unprotected)

**Question:** Examine the **adjacency-SIDs** auto-allocated per IS-IS adjacency and explain **protected vs unprotected** adj-SIDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.1 — Enable per-prefix TI-LFA on all core interfaces

**Question:** Enable **TI-LFA (per-prefix)** on all Garnet IS-IS core interfaces.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.2 — Verify backup paths pre-computed in FIB

**Question:** Prove that TI-LFA backup paths are **pre-computed and installed in the FIB** before any failure.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.3 — Test link failure with continuous ping (sub-50ms)

**Question:** Run continuous CE-to-CE (or PE-to-PE) traffic, fail a primary Garnet core link, and confirm **sub-50ms / 0–1 packet** loss.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.4 — Node protection vs link protection

**Question:** Configure and contrast **node protection** vs **link protection** in TI-LFA.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.1 — Explicit SR-TE policy (segment-list of prefix-SIDs)

**Question:** On Gar-R1, build an **explicit SR-TE policy** to Gar-R2 (12.12.12.12) that forces a non-shortest path — e.g. Gar-R1 → Gar-R4 → Gar-R5 → Gar-R2 — using a segment-list of prefix-SIDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.2 — Dynamic SR-TE policy (metric IGP/TE/latency)

**Question:** Create a **dynamic** SR-TE policy on Gar-R1 to Gar-R2 optimized by a chosen **metric type (igp / te / latency)**; observe recomputation when a metric changes.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.3 — Gar-R6-initiated policy (Gar-R6 = 16.16.16.16 as SR-PCE)

**Question:** Configure the Garnet **Gar-R6 (16.16.16.16)** as an **SR-PCE**, have Gar-R1 connect as a PCC, and have the Gar-R6 **initiate/delegate** an SR-TE policy.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.4 — On-Demand Next-hop (ODN) with BGP color

**Question:** Configure **ODN** on Gar-R1 so an SR-TE policy is auto-created toward the BGP next-hop when a VPN route arrives carrying a matching **color** community.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.5 — Steer L3VPN traffic into an SR-TE policy

**Question:** Steer **L3VPN (VPNv4)** traffic from Gar-R2 → Gar-R1 into the color-100 SR-TE policy by coloring the VPN routes.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.1 — Define Flex-Algo 128 (delay metric) on Garnet

**Question:** Define **Flex-Algo 128** using the **delay (min-latency) metric** on all Garnet nodes and advertise the definition from a subset (definition can be flooded by one node but is typically defined consistently).


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.2 — Assign locator / prefix-SID per algo

**Question:** Advertise a **per-algorithm prefix-SID** for Loopback0 so Algo 128 has its own SID separate from the default Algo 0 SID.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.3 — Verify separate topology computation & use for low-latency path

**Question:** Confirm Algo 128 computes a **separate (delay-optimized) topology** and steer latency-sensitive traffic onto it.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.1 — Enable SR alongside LDP (dual-stack)

**Question:** On Emerald (currently IS-IS + LDP), enable **SR-MPLS under IS-IS** so nodes run **LDP and SR simultaneously** (ships-in-the-night), with no traffic disruption.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.2 — Verify both label types in the LFIB

**Question:** Show that the **LFIB contains both LDP and SR labels** during the dual-stack phase.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.3 — Enable SR-prefer (prefer SR labels over LDP)

**Question:** Flip preference so **SR labels are preferred over LDP** on all Emerald nodes.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.4 — Remove LDP completely & verify no traffic loss

**Question:** Remove LDP from all Emerald nodes and confirm **no traffic loss** during and after the migration.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 6.1 — Prefix-SID conflict (two routers, same index)

**Question:** Two Garnet routers advertise the **same prefix-SID index** (e.g. Gar-R4 and Gar-R5 both use index 14). Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 6.2 — SRGB mismatch

**Question:** One Garnet node has a **different SRGB** (e.g. Gar-R3 = 18000–25999 while everyone else is 16000–23999). Diagnose the impact and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 6.3 — SR-TE policy down (SID not reachable)

**Question:** An explicit SR-TE policy on Gar-R1 is **Operational: down**. The segment-list references a prefix-SID label that is not reachable/valid. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

