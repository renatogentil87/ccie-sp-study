# WB11 — Multicast + Multicast VPN (mVPN)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** PIM / SSM / MSDP in AS65100 + inter-domain, and mVPN (Profile 0 & Profile 3) for VRF GREEN
**Level:** CCIE SP
**Prerequisite:** WB00-WB07 complete (unicast IGP, LDP, L3VPN, inter-AS as needed for MSDP/mVPN)

> **Format:** Tasks only. No configurations provided.
> Work **progressively** — native multicast first, then inter-domain, then mVPN.
> **Snapshot** at each section.

> **RP reference:** AS65100 RP = **R7**. AS65200 RP = **R19**.

---

## Pre-flight

- [ ] Unicast reachability solid (multicast RPF depends on it)
- [ ] Decide source/receiver placement (document which routers emulate source & receiver)
- [ ] **Snapshot:** `WB11-00-baseline`

---

## Section 1 — PIM-SM in AS65100 (RP = R7)

- [ ] **Task 1.1** — Enable multicast routing globally on R1-R8.
- [ ] **Task 1.2** — Enable **PIM sparse-mode** on all AS65100 core interfaces (per the Full Link Map) and on relevant PE-CE/source/receiver interfaces.
- [ ] **Task 1.3** — Configure **R7 as the RP** (static RP first; optionally Auto-RP or BSR as a variant — document which).
- [ ] **Task 1.4** — Join a group from a receiver (IGMP join on a receiver interface) and start a source for the same group.
- [ ] **Task 1.5** — Confirm the shared tree **(\*,G)** rooted at R7 forms, then the **(S,G)** shortest-path tree after SPT switchover.

**Verify / Snapshot**
- [ ] `show ip pim neighbor` — all core PIM adjacencies UP
- [ ] `show ip pim rp mapping` — R7 is RP for the group range
- [ ] `show ip mroute` — (\*,G) and (S,G) entries, correct RPF + OIL
- [ ] `show ip rpf <source>` — RPF toward source correct
- [ ] **Snapshot:** `WB11-01-pim-sm`

---

## Section 2 — PIM-SSM (232.0.0.0/8)

- [ ] **Task 2.1** — Enable **SSM** for the default range **232/8** in AS65100.
- [ ] **Task 2.2** — Configure the receiver for an **IGMPv3** source-specific join (S,G) in the SSM range (no RP, no shared tree).
- [ ] **Task 2.3** — Confirm an (S,G) SPT builds directly, with **no (\*,G)** and no RP involvement.

**Verify / Snapshot**
- [ ] `show ip pim rp mapping` — SSM range shows no RP (SSM)
- [ ] `show ip igmp groups` — IGMPv3 (S,G) membership
- [ ] `show ip mroute 232.x.x.x` — only (S,G), SPT
- [ ] **Snapshot:** `WB11-02-ssm`

---

## Section 3 — MSDP (inter-domain, R7 ↔ R19)

Inter-domain source discovery between **R7 (AS65100 RP)** and **R19 (AS65200 RP)**.

- [ ] **Task 3.1** — Ensure inter-AS unicast reachability between R7 and R19 loopbacks (via the inter-AS link / BGP) — MSDP peering + RPF depend on it.
- [ ] **Task 3.2** — Configure an **MSDP peer** session between R7 and R19 (originator-id = loopback).
- [ ] **Task 3.3** — Confirm **SA (Source-Active)** messages are exchanged so each RP learns about active sources in the other domain.
- [ ] **Task 3.4** — Start a source in one AS; confirm the receiver in the other AS joins and receives traffic via the inter-domain SPT.
- [ ] **Task 3.5** — (Optional hardening) apply an SA filter to control which groups/sources are advertised.

**Verify / Snapshot**
- [ ] `show ip msdp peer` — session UP (Established)
- [ ] `show ip msdp sa-cache` — SA entries for remote sources
- [ ] `show ip mroute` on both RPs — cross-domain (S,G)
- [ ] End-to-end receiver in AS65200 gets AS65100 source traffic
- [ ] **Snapshot:** `WB11-03-msdp`

---

## Section 4 — mVPN Profile 0 (Default MDT, GRE/PIM)

VRF **GREEN**. Draft-Rosen style: default MDT with PIM in the core (GRE encapsulation).

- [ ] **Task 4.1** — Enable **multicast routing within VRF GREEN** on the Green PEs.
- [ ] **Task 4.2** — Configure the **default MDT group** for VRF GREEN (a core multicast group that forms the default MDT between all Green PEs via core PIM).
- [ ] **Task 4.3** — Ensure the **core (global) PIM** carries the default MDT group (RP = R7 handles the MDT group in the provider core).
- [ ] **Task 4.4** — Configure a **data MDT** range + **threshold** so high-rate (S,G) customer streams move off the default MDT onto a data MDT.
- [ ] **Task 4.5** — Join/source customer multicast inside VRF GREEN across PEs and confirm it rides the default MDT, then switches to a data MDT above threshold.

**Verify / Snapshot**
- [ ] `show ip pim mdt` / `show ip pim vrf GREEN mdt` — default + data MDT state
- [ ] `show ip mroute` (global) — MDT group tunnel (Tunnel interface auto-created)
- [ ] `show ip mroute vrf GREEN` — customer (S,G)/(\*,G)
- [ ] Confirm data MDT switchover when rate exceeds threshold
- [ ] **Snapshot:** `WB11-04-mvpn-profile0`

---

## Section 5 — mVPN Profile 3 (BGP A-D + PIM C-mcast signaling)

- [ ] **Task 5.1** — Enable the **BGP MVPN (ipv4 mvpn) address-family** on the Green PEs + RR for mVPN auto-discovery.
- [ ] **Task 5.2** — Configure Profile 3 (default MDT via **PIM** in core, but **BGP auto-discovery** of PE membership) — PEs discover each other via BGP MVPN A-D routes rather than relying solely on core PIM MDT joins.
- [ ] **Task 5.3** — Confirm **C-multicast signaling** (customer join state) is carried per the profile (PIM-based C-mcast in Profile 3).
- [ ] **Task 5.4** — Validate Green customer multicast works end-to-end under Profile 3.

**Verify / Snapshot**
- [ ] `show bgp ipv4 mvpn all` — A-D routes (Type 1 intra-AS, membership)
- [ ] `show ip pim vrf GREEN neighbor` — C-PIM neighbors across MDT
- [ ] `show ip mroute vrf GREEN` — customer trees built via A-D discovery
- [ ] **Snapshot:** `WB11-05-mvpn-profile3`

---

## Section 6 — mVPN Verification (consolidated)

Confirm the full mVPN picture with the key show commands:

- [ ] **Task 6.1** — `show ip mroute vrf GREEN` — customer (\*,G)/(S,G), correct RPF via MDT tunnel.
- [ ] **Task 6.2** — `show ip pim vrf GREEN neighbor` — C-PIM neighbors reachable over the MDT.
- [ ] **Task 6.3** — `show bgp ipv4 mvpn all` — MVPN routes (A-D + C-mcast where applicable).
- [ ] **Task 6.4** — `show ip pim mdt` / `show ip pim mdt bgp` — MDT groups + BGP MDT SAFI (Profile 0 uses MDT SAFI; confirm).
- [ ] **Task 6.5** — Correlate: global core mroute for the MDT group ↔ VRF mroute for the customer group (prove the overlay rides the provider tunnel).

**Verify / Snapshot**
- [ ] All four show families consistent and non-empty
- [ ] End-to-end Green multicast receiver gets source traffic across the SP core
- [ ] **Snapshot:** `WB11-06-mvpn-verify`

---

## Completion Criteria

- [ ] PIM-SM with RP=R7 forming (\*,G) and (S,G) in AS65100
- [ ] PIM-SSM (232/8) building (S,G) SPTs with no RP
- [ ] MSDP R7↔R19 exchanging SA, inter-domain source received
- [ ] mVPN Profile 0 (default + data MDT, GRE/PIM) working for VRF GREEN
- [ ] mVPN Profile 3 (BGP A-D + PIM C-mcast) working for VRF GREEN
- [ ] All mVPN verification commands validated
- [ ] **Final snapshot:** `WB11-FINAL`
