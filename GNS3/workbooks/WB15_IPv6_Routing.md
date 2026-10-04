# WB15 — IPv6 Routing (Dual-Stack across 3 ASes)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Format:** Tasks only — NO configurations shown. Produce the config yourself, verify, snapshot.
**Prerequisite:** WB00–WB11 complete (IPv4 IGP/BGP/MPLS operational). This workbook adds native IPv6 as a parallel dual-stack control plane.

> **Goal:** Dual-stack the whole lab — IPv6 IGP per AS (IS-IS single-topology in 65100/65300, OSPFv3 in 65200), MP-BGP IPv6 between/within ASes, and 6VPE for IPv6 L3VPN. End state: IPv6 reachability across all 3 ASes, native in the core (contrast with WB12 where the core stayed IPv4-only).

---

## IPv6 Addressing Scheme

**Link prefix pattern:** `2001:db8:<AS>:XY::/64` where `<AS>` = 1 / 2 / 3 and `XY` = the two router numbers on the link (lower first). Host part = router number.
**Loopbacks:** `2001:db8:<AS>::<R>/128`.

| AS | Link prefix base | Loopback base |
|----|------------------|---------------|
| 65100 | `2001:db8:1:XY::/64` | `2001:db8:1::R/128` |
| 65200 | `2001:db8:2:XY::/64` | `2001:db8:2::R/128` |
| 65300 | `2001:db8:3:XY::/64` | `2001:db8:3::R/128` |

**Examples:**
- R1↔R3 (f0/0, 10.1.3.x) → `2001:db8:1:13::1` / `2001:db8:1:13::3`
- R1 Lo0 → `2001:db8:1::1/128`
- R11↔R12 (20.11.12.x) → `2001:db8:2:1112::11` / `::12`  *(use `1112` when both numbers are 2-digit; keep it unambiguous)*
- R20↔R21 (30.20.21.x) → `2001:db8:3:2021::20` / `::21`
- Inter-AS R6↔R20 → choose a neutral prefix, e.g. `2001:db8:0:620::6` / `::20`

---

## Section 1 — IPv6 Addressing (Dual-Stack)

- [ ] **Task 1.1 — Enable IPv6 globally.**
  `ipv6 unicast-routing` + IPv6 CEF on **every** router R1–R30 (including P routers this time — native IPv6 core).
  *Verify:* `show ipv6 cef` populated on all.

- [ ] **Task 1.2 — Address all core links (per the scheme).**
  Add IPv6 to every intra-AS core link in all 3 ASes alongside existing IPv4 (dual-stack).
  *Verify:* `show ipv6 interface brief`; ping6 each directly-connected neighbor.

- [ ] **Task 1.3 — Address IPv6 loopbacks.**
  `2001:db8:<AS>::<R>/128` on Lo0 of every router.
  *Verify:* loopbacks present in `show ipv6 interface brief`.

- [ ] **Task 1.4 — Address the inter-AS links.**
  R5↔R11, R6↔R12, R6↔R20, R12↔R21 get dual-stack IPv6 (neutral `2001:db8:0:XY::/64` or either AS's space — pick a convention and document it).
  *Verify:* ping6 across each inter-AS link.

---

## Section 2 — IS-IS for IPv6 (AS65100 & AS65300, single-topology)

**Concept:** Reuse the existing IS-IS instance; enable the IPv6 address-family in **single-topology** mode (IPv4 and IPv6 share one SPF/topology — requires congruent topology, which this lab has).

- [ ] **Task 2.1 — Enable IPv6 under IS-IS on AS65100.**
  On R1–R8, add `address-family ipv6` (single-topology) to the IS-IS process and `ipv6 router isis` on the core interfaces.
  *Verify:* `show isis ipv6 topology` / `show ipv6 route isis` → all 65100 loopbacks learned.

- [ ] **Task 2.2 — Enable IPv6 under IS-IS on AS65300.**
  Repeat on R20–R23, R30.
  *Verify:* `show ipv6 route isis` on R23 shows R20/R21/R22/R30 loopbacks.

- [ ] **Task 2.3 — Confirm single-topology correctness.**
  Verify IPv4 and IPv6 adjacencies ride the same adjacency (single-topology) and no interface is IPv6-only (would break single-topology). Note where you'd need **multi-topology** instead and why you don't here.
  *Verify:* `show clns neighbors` + `show isis ipv6 topology` consistent.

---

## Section 3 — OSPFv3 for IPv6 (AS65200)

**Concept:** OSPFv3 is a separate process from OSPFv2 (not an AF on the v2 process in this IOS); enable per-interface in Area 0.

- [ ] **Task 3.1 — Enable OSPFv3 Area 0 on AS65200 core.**
  On R11–R16, R19 start an OSPFv3 process (router-id required — reuse the IPv4 loopback as RID) and enable `ipv6 ospf <pid> area 0` on core interfaces + loopbacks.
  *Verify:* `show ipv6 ospf neighbor` Full on all core adjacencies; `show ipv6 route ospf` has all 65200 loopbacks.

- [ ] **Task 3.2 — Handle the OSPF PE-CE case (R15↔R17).**
  Decide whether R17 participates in IPv6 (dual-stack the PE-CE) or stays IPv4-only; document.
  *Verify:* consistent with your choice.

---

## Section 4 — MP-BGP IPv6

**Concept:** Carry IPv6 unicast in BGP: eBGP IPv6 between ASBRs, iBGP IPv6 reflected by the RRs. Use IPv6 transport (peer over IPv6 addresses) for a clean native design.

- [ ] **Task 4.1 — eBGP IPv6 between ASBRs.**
  Bring up IPv6-unicast eBGP on R5↔R11, R6↔R12, R6↔R20, R12↔R21 (peer over the inter-AS IPv6 link addresses). Activate `address-family ipv6 unicast`.
  *Verify:* `show bgp ipv6 unicast summary` Up on each pair.

- [ ] **Task 4.2 — iBGP IPv6 via RRs.**
  Activate IPv6 unicast on the PE/ASBR ↔ RR sessions (R7/R8, R19, R30) and set route-reflector-client in the IPv6 AF. Advertise PE loopbacks/customer prefixes in IPv6.
  *Verify:* `show bgp ipv6 unicast` on RR shows reflected prefixes; interior routers learn remote-AS IPv6 prefixes.

- [ ] **Task 4.3 — Next-hop handling.**
  Ensure ASBRs do `next-hop-self` in the IPv6 AF toward RRs (interior routers can't reach the far-AS link-local/global next-hop otherwise).
  *Verify:* `show bgp ipv6 unicast <remote-prefix>` → next-hop = local ASBR, reachable via IGP.

---

## Section 5 — IPv6 L3VPN (6VPE)

**Concept:** Same as WB12 Section 2 but applied natively across the dual-stacked lab — VPNv6 SAFI, VRF IPv6 AF, IPv6 PE-CE. (If WB12 is already done, extend that config; here reconcile it with the now-native IPv6 core.)

- [ ] **Task 5.1 — Add IPv6 AF to a customer VRF.**
  Pick **Blue** (spans 3 ASes) or **Yellow**. Add `address-family ipv6` to the VRF with RD + import/export RTs.
  *Verify:* `show vrf detail` shows the IPv6 AF.

- [ ] **Task 5.2 — IPv6 PE-CE.**
  Configure IPv6 on a PE-CE link inside the VRF (e.g. R20↔R24 Yellow, or R1↔R32 Blue) and run IPv6 eBGP (or static) PE-CE.
  *Verify:* `show bgp vpnv6 unicast vrf <name> summary` → CE neighbor Up.

- [ ] **Task 5.3 — Activate VPNv6 on PE↔RR + inter-AS.**
  Activate `address-family vpnv6 unicast` PE↔RR, with `send-community extended`. For the cross-AS VPN, confirm the inter-AS method (Option A/B/C from earlier workbooks) carries VPNv6.
  *Verify:* `show bgp vpnv6 unicast all` shows remote VRF prefixes with VPN label + remote-PE next-hop.

---

## Section 6 — Verification (End-to-End IPv6 across all 3 ASes)

- [ ] **Task 6.1 — Per-AS IGP IPv6 complete.**
  Each AS has full intra-AS IPv6 loopback reachability (IS-IS in 65100/65300, OSPFv3 in 65200).

- [ ] **Task 6.2 — Inter-AS IPv6 via BGP.**
  From R1, reach an AS65300 loopback (e.g. `2001:db8:3::23`) via MP-BGP IPv6. `traceroute ipv6` crosses the ASBRs.
  *Verify:* `ping ipv6 2001:db8:3::23 source 2001:db8:1::1` succeeds.

- [ ] **Task 6.3 — 6VPE end-to-end.**
  Customer IPv6 CE-to-CE across ASes inside the VRF (Blue spanning 65100/65200/65300).
  *Verify:* VRF ping6 CE→CE; `traceroute` shows MPLS transport + VPN label.

- [ ] **Task 6.4 — Dual-stack sanity.**
  Confirm IPv4 service (WB00–11) is **unaffected** — all prior IPv4 reachability still works alongside the new IPv6.
  *Verify:* spot-check IPv4 ping across each AS; `show ip route` + `show ipv6 route` both healthy.

---

## Completion Checklist

- [ ] IPv6 + CEF enabled on all routers; core links + loopbacks dual-stacked
- [ ] IS-IS IPv6 single-topology up in AS65100 + AS65300
- [ ] OSPFv3 Area 0 up in AS65200
- [ ] eBGP IPv6 up on all 4 ASBR pairs; iBGP IPv6 reflected by RRs
- [ ] next-hop-self correct in IPv6 AF
- [ ] 6VPE: VRF IPv6 AF + VPNv6 + IPv6 PE-CE working
- [ ] End-to-end IPv6 across all 3 ASes (global + VPN)
- [ ] IPv4 service confirmed unaffected (true dual-stack)
- [ ] **GNS3 snapshot taken:** `wb15-ipv6-routing-complete`

---

## Snapshot Points (Progressive)

1. `wb15-s1-addressing` — after dual-stack addressing (Section 1)
2. `wb15-s23-igp6` — after IS-IS v6 + OSPFv3 up (Sections 2–3)
3. `wb15-s4-mpbgp6` — after MP-BGP IPv6 inter/intra-AS (Section 4)
4. `wb15-s5-6vpe` — after 6VPE (Section 5)
5. `wb15-complete` — after end-to-end verification (Section 6)
