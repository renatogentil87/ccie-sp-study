# WB12 — 6PE & 6VPE (IPv6 over IPv4 MPLS)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Format:** Tasks only — NO configurations shown. Produce the config yourself, verify, snapshot.
**Prerequisite:** WB00–WB11 complete (IGP, LDP, MPLS L3VPN, inter-AS, RRs all operational in AS65100).

> **Goal:** Carry IPv6 across an IPv4-only MPLS core without enabling IPv6 on P routers. 6PE = global IPv6; 6VPE = IPv6 inside a VRF. P routers (R3, R4) stay IPv4/MPLS only throughout.

---

## Scope / Device Roles

| Role | Routers | Notes |
|------|---------|-------|
| PE (6PE/6VPE) | R1, R2 | IPv6 edge, label IPv6 to IPv4 next-hop |
| P (unchanged) | R3, R4 | IPv4 LDP only — **do NOT enable IPv6** |
| RR | R7, R8 | Reflect `ipv6` and `vpnv6` address-families |
| CE (6PE global) | R9 (eBGP), R10 | IPv6 PE-CE in global table |
| CE (6VPE VRF) | R32 (Blue), R10 (Green) | IPv6 PE-CE inside VRF |

---

## Section 1 — 6PE (Global IPv6 over IPv4 MPLS)

**Concept:** PEs exchange IPv6 prefixes over the existing IPv4 iBGP session (via RR). Next-hop is encoded as an IPv4-mapped IPv6 address (`::ffff:a.b.c.d`). The core label switches on the IPv4 LSP; PEs impose an IPv6 explicit-null (or per-prefix) label. P routers never see IPv6.

- [ ] **Task 1.1 — Enable IPv6 forwarding on PEs only.**
  On R1 and R2, enable `ipv6 unicast-routing` and global IPv6 CEF. Confirm R3/R4 remain IPv4-only.
  *Verify:* `show ipv6 cef` present on R1/R2; absent/empty on R3/R4.

- [ ] **Task 1.2 — Add IPv6 PE-CE addressing (global).**
  Assign dual-stack IPv6 on the R1↔R9 PE-CE link (`192.168.1.0/24` link) and the R2↔R10 PE-CE link (`192.168.4.0/24` link). Use `2001:db8:1:19::/64` (R1-R9) and `2001:db8:1:210::/64` (R2-R10). Advertise CE IPv6 loopbacks (R9, R10) toward the PEs.
  *Verify:* PE can ping CE IPv6 link + loopback directly.

- [ ] **Task 1.3 — Activate the IPv6 address-family on the PE↔RR iBGP sessions.**
  On R1 and R2, under BGP activate the existing RR neighbors (R7 `150.1.7.7`, R8 `150.1.8.8`) in `address-family ipv6`. The transport session stays IPv4; only the AFI/SAFI is added.
  *Key requirement:* set `send-label` so IPv6 prefixes carry an MPLS label. Without a label the core cannot switch IPv6.
  *Verify:* `show bgp ipv6 unicast summary` shows the RR neighbors Up with a nonzero prefix count once CE routes exist.

- [ ] **Task 1.4 — Fix the IPv6 next-hop so it is IPv4-mapped.**
  Ensure the PE advertises an IPv4-mapped next-hop (`::ffff:150.1.1.1` for R1) so the receiving PE resolves it over the IPv4 LSP. Confirm the RRs reflect `ipv6` without rewriting the next-hop (RRs do not need IPv6 forwarding, only the AF activated for reflection).
  *Verify:* On R2, `show bgp ipv6 unicast <R9-loopback>` → next-hop `::FFFF:150.1.1.1`, with an in/out label bound.

---

## Section 2 — 6VPE (IPv6 L3VPN)

**Concept:** Same PE/RR plumbing but IPv6 lives inside a VRF. The VPN label (bottom) identifies the VRF/CE; the IGP/LDP IPv4 label (top) carries the packet across the core. BGP SAFI is `vpnv6`.

- [ ] **Task 2.1 — Add the IPv6 address-family to an existing VRF (dual-stack).**
  Take the Customer **Blue** VRF already defined on R1 (serving R32). Add an `address-family ipv6` under the VRF and attach the same RD, plus import/export route-targets for Blue. The VRF now carries both IPv4 and IPv6 (dual-stack VRF).
  *Verify:* `show vrf detail <Blue>` lists both IPv4 and IPv6 AFs with RD + RTs.

- [ ] **Task 2.2 — Configure IPv6 PE-CE inside the VRF.**
  On R1 g1/0... — use the Blue PE-CE link to R32 (`192.168.3.0/24`). Add VRF IPv6 addressing `2001:db8:1:332::/64` and run eBGP (AS 65025) for IPv6 in the VRF address-family (per-VRF or per-neighbor IPv6 AF).
  *Verify:* `show bgp vpnv6 unicast vrf Blue summary` → R32 neighbor Up.

- [ ] **Task 2.3 — Activate VPNv6 on the PE↔RR sessions.**
  On R1, R2, R7, R8 activate RR neighbors under `address-family vpnv6 unicast`. On R7/R8 set these as route-reflector-clients and ensure `send-community extended` (RTs are extended communities — without them VPNv6 import fails).
  *Verify:* `show bgp vpnv6 unicast all summary` on PEs shows RR Up; `show bgp vpnv6 unicast all` on RR shows reflected Blue IPv6 prefixes.

- [ ] **Task 2.4 — Build the second VRF endpoint and import.**
  Add the Blue IPv6 VRF to R2 (or confirm Blue reaches R2 per the Blue 3-AS design). Confirm RT import places the remote CE IPv6 prefix into the local VRF with a VPN label.
  *Verify:* `show bgp vpnv6 unicast vrf Blue` → remote prefix present, next-hop = remote PE loopback (IPv4-mapped), VPN label assigned.

---

## Section 3 — Verification (End-to-End IPv6 over IPv4 MPLS)

- [ ] **Task 3.1 — Confirm the core is still IPv4-only.**
  On R3 and R4: `show ipv6 route` empty, `show mpls forwarding-table` shows only IPv4 FECs. The transit label stack must be IPv4 LDP labels only.

- [ ] **Task 3.2 — Label verification on PEs.**
  `show mpls forwarding-table` on R1/R2 shows IPv6 prefixes (6PE) and VPNv6 prefixes (6VPE) with locally allocated labels (bottom-of-stack). Record the labels for your snapshot.

- [ ] **Task 3.3 — Data-plane: 6PE.**
  From R9 (global IPv6 CE) ping R10's IPv6 loopback end-to-end. Run `traceroute ipv6` and confirm the P routers appear as MPLS hops (label present), not as IPv6 hops.

- [ ] **Task 3.4 — Data-plane: 6VPE.**
  From R32 (Blue VRF) reach the remote Blue IPv6 CE. Confirm two-label stack on the imposing PE (transport + VPN). `traceroute` inside the VRF shows MPLS transport across R3/R4.

- [ ] **Task 3.5 — Control-plane sanity.**
  `show bgp ipv6 unicast` and `show bgp vpnv6 unicast all` confirm next-hops are IPv4-mapped and reachable via the IPv4 IGP/LFIB.

---

## Section 4 — Troubleshooting Drills

Break it, diagnose, fix. Snapshot each root cause.

- [ ] **Task 4.1 — IPv6 route not landing in VRF.**
  Symptom: remote Blue IPv6 prefix missing from `show bgp vpnv6 unicast vrf Blue`.
  Investigate in order: (a) RT import/export mismatch between PEs, (b) `send-community extended` missing on PE→RR, (c) VPNv6 AF not activated on RR for that neighbor, (d) RD mismatch.
  *Deliverable:* identify which layer failed and how you confirmed it (which `show` revealed it).

- [ ] **Task 4.2 — Label not allocated for IPv6 prefix.**
  Symptom: 6PE prefix present in BGP but no label → packets black-holed / punted.
  Investigate: missing `send-label` on the IPv6 AF neighbor; or `no mpls ip`/CEF disabled; or next-hop not IPv4-mapped so no LSP resolves.
  *Verify fix:* `show bgp ipv6 unicast <prefix>` shows in/out label; `show mpls forwarding-table` has the FEC.

- [ ] **Task 4.3 — Next-hop unreachable.**
  Symptom: BGP prefix valid but not best / not installed. Confirm the IPv4-mapped next-hop (`::ffff:x.x.x.x`) resolves to a labeled IPv4 LSP. A plain IGP route without a label will still fail for VPN traffic.
  *Verify:* `show ipv6 cef <prefix>` → labeled output path.

---

## Completion Checklist

- [ ] 6PE: R9↔R10 global IPv6 reachable over IPv4 MPLS core
- [ ] 6VPE: Blue IPv6 CE-to-CE reachable inside VRF
- [ ] R3/R4 confirmed IPv6-free (core untouched)
- [ ] All PE↔RR sessions carry `ipv6` + `vpnv6` with labels
- [ ] Next-hops verified IPv4-mapped
- [ ] All 3 troubleshooting root causes reproduced + fixed
- [ ] **GNS3 snapshot taken:** `wb12-6pe-6vpe-complete`

---

## Snapshot Points (Progressive)

1. `wb12-s1-6pe` — after Section 1 (global IPv6 label exchange working)
2. `wb12-s2-6vpe` — after Section 2 (VPNv6 import working)
3. `wb12-complete` — after Sections 3 & 4 (verified + TS drills done)
