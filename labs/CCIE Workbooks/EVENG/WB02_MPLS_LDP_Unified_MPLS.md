# CCIE SP Workbook 02 — MPLS LDP & Unified MPLS

**Domain 1: Core Routing (25%)**
**Platform:** IOS-XRv 9000

---

## Topology Reference

**Emerald — AS 65100 (IS-IS + LDP)**

| Node | Loopback0 | Role |
|------|-----------|------|
| E-R5 | 6.6.6.6 | Path Computation Element |
| E-R4 | 4.4.4.4 | Core P |
| E-R3 | 3.3.3.3 | Core P |
| E-R6 | 5.5.5.5 | Inter-AS border |
| E-R1 | 1.1.1.1 | Edge PE |
| E-R2 | 2.2.2.2 | Edge PE |

**Garnet — AS 65200 (IS-IS + SR-MPLS)**

| Node | Loopback0 | Role |
|------|-----------|------|
| Gar-R6 | 16.16.16.16 | Path Computation Element |
| Gar-R3–Gar-R5 | — | Core P |
| Gar-R7 | 17.17.17.17 | Inter-AS border |
| Gar-R1 | 11.11.11.11 | Edge PE |
| Gar-R2 | 12.12.12.12 | Edge PE |

**Gold — AS 65300 (IS-IS + SRv6)**

| Node | Loopback0 | Role |
|------|-----------|------|
| G-R4 | 24.24.24.24 | Inter-AS border |
| G-R5 | 25.25.25.25 | Inter-AS border |
| G-R3 | 23.23.23.23 | Core P |
| G-R1 | 21.21.21.21 | Edge PE |
| G-R2 | 22.22.22.22 | Edge PE |

**Inter-AS links:**
- E-R6 ↔ Gar-R7 (direct, Emerald ↔ Garnet)
- E-R6 ↔ G-R4 (Emerald ↔ Gold)
- G-R5 ↔ Gar-R7 (Gold ↔ Garnet)

**Inter-AS addressing convention used in this workbook:**

| Link | A-side | B-side |
|------|--------|--------|
| E-R6 ↔ Gar-R7 | 10.100.200.5/30 (E-R6 Gi0/0/0/2) | 10.100.200.6/30 (Gar-R7) |
| E-R6 ↔ G-R4 | 10.100.300.5/30 (E-R6 Gi0/0/0/3) | 10.100.300.21/30 (G-R4 Gi0/0/0/1) |
| G-R5 ↔ Gar-R7 | 10.300.200.22/30 (G-R5 Gi0/0/0/2) | 10.300.200.16/30 (Gar-R7 Gi0/0/0/1) |

---

## Section 1 — LDP Basics on Emerald

### Task 1.1: LDP Neighbor Discovery (5 points)

**Question:** On the Emerald core, enable LDP on all IS-IS core interfaces so that E-R3 (3.3.3.3), E-R4 (4.4.4.4), E-R6 (5.5.5.5), E-R1 (1.1.1.1), and E-R2 (2.2.2.2) form LDP adjacencies. Use the interface-level `mpls ldp` model. Verify that link-hello (UDP 646) discovery brings up sessions over TCP 646.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2: Targeted LDP Session (5 points)

**Question:** E-R1 (1.1.1.1) and E-R2 (2.2.2.2) are not directly connected but require a directed LDP session (e.g., for a future L2VPN/AToM pseudowire). Configure a **targeted** LDP session between E-R1 and E-R2 without relying on link hellos.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3: LDP Router-ID from Loopback0 (5 points)

**Question:** Ensure every Emerald LSR derives a **deterministic** LDP router-ID and transport address from its `Loopback0` interface, so that LDP IDs match the IS-IS/BGP loopbacks (e.g., E-R1 = 1.1.1.1, E-R3 = 3.3.3.3). Prevent the router from auto-selecting an arbitrary interface address.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4: Label Allocation Filtering — Host Routes Only (5 points)

**Question:** Reduce LFIB size on the Emerald core. Configure LDP so each LSR only **allocates/originates** local labels for **/32 host routes** (loopbacks), not for the intra-area transit links. Use a prefix-list.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.5: LDP Session Protection (5 points)

**Question:** On E-R3 (3.3.3.3), protect the LDP sessions so that if a directly connected link goes down but the neighbor is still reachable via an alternate IGP path, the **label bindings are retained** and the session survives via a targeted hello. Hold retained bindings for 120 seconds.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — LDP Advanced

### Task 2.1: LDP Authentication (MD5) (5 points)

**Question:** Secure the LDP TCP sessions on the E-R6 (5.5.5.5) ↔ E-R3 (3.3.3.3) link with **MD5 authentication**. Configure it symmetrically.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2: LDP Graceful Restart (5 points)

**Question:** Enable **LDP Graceful Restart (GR)** on E-R1 (1.1.1.1) and its core neighbor E-R3 (3.3.3.3) so that an LDP control-plane restart (e.g., RP failover) does not tear down the forwarding path. Set forwarding-state hold and reconnect timers appropriately.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3: LDP-IGP Synchronization Verification (5 points)

**Question:** LDP-IGP sync is configured on the Emerald IS-IS core. On the E-R4 (4.4.4.4) ↔ E-R3 (3.3.3.3) link, **disable `mpls ldp` on the E-R3 end** to simulate an LDP-not-ready condition, and verify that IS-IS advertises the **maximum metric** for that link until LDP re-synchronizes.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4: LDP Label Advertisement Filtering — PE Loopbacks Only (5 points)

**Question:** On the Emerald core, configure LDP so that each LSR only **advertises** labels for the **PE loopbacks E-R1 (1.1.1.1/32) and E-R2 (2.2.2.2/32)** to its neighbors — not for P/ASBR/Gar-R6 loopbacks. Use advertise (outbound) filtering.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — BGP Labeled Unicast / Unified MPLS

### Task 3.1: BGP-LU Concept for Inter-AS (5 points)

**Question:** Explain, and then set the stage for, **BGP Labeled Unicast (BGP-LU / RFC 3107)** as the inter-AS transport mechanism for **Unified MPLS (Seamless MPLS)** across Emerald → Gold → Garnet. Why is BGP-LU needed instead of extending LDP across the AS boundaries?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2: BGP-LU between E-R6 ↔ G-R4 (Emerald ↔ Gold) (5 points)

**Question:** Configure eBGP **labeled-unicast** between E-R6 (Emerald AS 65100, 5.5.5.5) and G-R4 (Gold AS 65300, 24.24.24.24) over the inter-AS link (E-R6 10.100.300.5, G-R4 10.100.300.21). Advertise Emerald PE loopbacks (1.1.1.1/32, 2.2.2.2/32) toward Gold with labels, and set next-hop-self so the label path stitches at the border. eBGP peering is over the directly connected interface addresses.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3: BGP-LU between G-R5 ↔ Gar-R7 (Gold ↔ Garnet) (5 points)

**Question:** Configure eBGP **labeled-unicast** between G-R5 (Gold AS 65300, 25.25.25.25) and Gar-R7 (Garnet AS 65200, 17.17.17.17) over the inter-AS link (G-R5 10.300.200.22, Gar-R7 10.300.200.16). Propagate the Emerald PE loopbacks (learned from Task 3.2) onward into Garnet with labels, and have Gar-R7 set next-hop-self so Garnet SR-MPLS resolves the transit. Also advertise Garnet Gar-R1 (11.11.11.11/32) back toward Gold.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4: End-to-End Label Path E-R1 → (Gold transit) → Gar-R1 (5 points)

**Question:** Verify and explain the **end-to-end Unified MPLS label path** from E-R1 (Emerald, 1.1.1.1) to Gar-R1 (Garnet, 11.11.11.11) transiting the **Gold** AS via the E-R6→G-R4 and G-R5→Gar-R7 stitches. Ensure E-R1 has a labeled route to 11.11.11.11/32 and trace the label operations at each hop.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — Troubleshooting

### Task 4.1: LDP Session Down — Transport Address Mismatch (5 points)

**Question:** On the E-R6 (5.5.5.5) ↔ E-R3 (3.3.3.3) link, LDP **discovery** shows hellos being exchanged, but the session never reaches `Oper` (stuck initializing / no TCP). No MD5 is configured. Diagnose and fix.

**Symptom / Diagnosis:**

```
show mpls ldp discovery
! Hellos Xmit/Recv present, but:
show mpls ldp neighbor brief
! Session state = "Initialized" / never "Oper"
show tcp brief | include :646
! No established TCP 646 to the peer's expected address
```

Root cause: a **transport-address mismatch**. One end advertises a `discovery transport-address` (or interface address) that is **not reachable / not in the IGP** by the peer, or the two ends disagree on which address to open the TCP session to (e.g., E-R3 set `discovery transport-address 3.3.3.3` but 3.3.3.3/32 is not advertised into IS-IS, so E-R6 cannot open TCP to it). LDP hellos are UDP and still succeed, but the TCP 646 session fails because the transport address is unroutable.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2: Label Not in LFIB — Allocation Filtering Too Aggressive (5 points)

**Question:** After applying the label allocation filter from Task 1.4, traffic to a specific PE loopback **E-R2 (2.2.2.2/32)** is being IP-forwarded (unlabeled) on part of the Emerald core, and `show mpls forwarding` on E-R3 has **no entry** for 2.2.2.2/32. LDP sessions are up. Diagnose and fix.

**Symptom / Diagnosis:**

```
show mpls ldp bindings 2.2.2.2/32
! No local binding on E-R3 -> label was never allocated
show mpls forwarding prefix 2.2.2.2/32
! No LFIB entry
show run formal | include "prefix-list|allocate for"
```

Root cause: the **allocation prefix-list is too aggressive** — it does not match 2.2.2.2/32 (e.g., the prefix-set was written as `0.0.0.0/0 eq 32` but was accidentally scoped to specific hosts like only `1.1.1.1/32`, or a `le/ge` bound excludes it). Because no **local label** is allocated for the FEC, E-R3 has nothing to advertise or install, and forwarding for that prefix falls back to plain IP.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3: BGP-LU Next-Hop Not Resolvable (5 points)

**Question:** On Gar-R7 (Garnet, 17.17.17.17), the Emerald PE prefix 1.1.1.1/32 is present in the BGP-LU table but is marked **not usable** — it is not installed in the RIB/CEF and not re-advertised into Garnet. The eBGP-LU session to G-R5 is up. Diagnose and fix.

**Symptom / Diagnosis:**

```
show bgp ipv4 labeled-unicast 1.1.1.1/32
! Path present but "(inaccessible)" / "not advertised" / next hop not resolved
show bgp ipv4 labeled-unicast 1.1.1.1/32 | include "next hop|inaccessible|not"
show route <bgp-next-hop>
! The BGP next hop has no route
```

Root cause: the **BGP next-hop is not resolvable / not label-reachable**. Because G-R5 did **not** apply `next-hop-self` (or the inter-AS /30 link address it left as next hop isn't redistributed into Garnet's IGP, and there is no label to it), Gar-R7 has no resolving route+label to the next hop. BGP requires the next hop to be reachable **and, for labeled-unicast transit, label-reachable**; an unresolvable next hop makes the route unusable and unadvertised.


> *Try this yourself first. Solution available in `solutions/` folder.*

