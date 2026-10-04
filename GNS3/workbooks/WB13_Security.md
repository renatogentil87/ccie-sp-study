# WB13 — Service Provider Security

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Format:** Tasks only — NO configurations shown. Produce the config yourself, verify, snapshot.
**Prerequisite:** WB00–WB11 complete (IGP, BGP full mesh/RR, MPLS L3VPN operational across all 3 ASes).

> **Goal:** Harden the control plane end-to-end: IGP + BGP authentication, infrastructure attack mitigation (RTBH, Flowspec, CoPP, uRPF, bogon/private-ASN filtering), and MPLS signaling authentication.

---

## Section 1 — IGP Security

- [ ] **Task 1.1 — IS-IS authentication on AS65100 (HMAC-MD5, per-level, per-interface).**
  On R1–R8, apply IS-IS HMAC-MD5 authentication. Requirements: per-interface authentication keyed by level (this is an L2-only domain, so authenticate Level-2 Hellos), plus LSP/SNP authentication at the IS-IS instance level for Level-2. Use a key chain.
  *Verify:* adjacencies stay Up; `show clns neighbors` full; break the key on one neighbor and confirm the adjacency drops (then fix).

- [ ] **Task 1.2 — IS-IS authentication on AS65300.**
  Repeat 1.1 on R20–R23, R30 (Area 49.0003, L2). Keep AS65100 and AS65300 key chains distinct.
  *Verify:* `show isis` authentication active; adjacencies full.

- [ ] **Task 1.3 — OSPF authentication on AS65200 (MD5).**
  On R11–R16, R19 enable OSPF MD5 (cryptographic) authentication in Area 0, per-interface or area-wide. Include the R15↔R17 OSPF PE-CE link decision (authenticate or intentionally scope it — document which).
  *Verify:* `show ip ospf interface` → "Cryptographic authentication enabled"; neighbors Full; mismatch test drops adjacency.

---

## Section 2 — BGP Security (Session Hardening)

- [ ] **Task 2.1 — MD5 (TCP-AO/MD5) on all BGP sessions.**
  Apply BGP neighbor password to **every** session: iBGP PE↔RR in each AS (R7/R8, R19, R30 clients), and all eBGP ASBR sessions (R5↔R11, R6↔R12, R6↔R20, R12↔R21). Use distinct passwords per AS pair.
  *Verify:* sessions re-establish; `show bgp <afi> summary` Up; `debug` or password mismatch test confirms TCP MD5 enforced.

- [ ] **Task 2.2 — GTSM / ttl-security on eBGP.**
  On the directly-connected eBGP ASBR sessions, apply `ttl-security hops 1`. Confirm it is mutually configured (GTSM must be set on both ends or the session fails).
  *Verify:* session Up; a spoofed low-TTL packet from a non-adjacent source would be dropped. `show bgp neighbor` shows "External BGP neighbor may be up to 1 hop away."
  *Note:* GTSM and `ebgp-multihop` are mutually exclusive — reconcile with any multihop sessions (e.g. inter-AS Option C loopback peering).

- [ ] **Task 2.3 — Maximum-prefix on eBGP peers.**
  Set `maximum-prefix` with a warning threshold on each eBGP ASBR neighbor and each eBGP PE-CE neighbor (R9, R10, R18, R24, R25, R26, R32). Choose a sane limit per peer. Decide `warning-only` vs. teardown per peer type (CE = teardown+restart; ASBR = warning first).
  *Verify:* `show bgp <afi> neighbor` shows the max-prefix policy; exceed it on a CE and observe the configured action.

---

## Section 3 — BGP RTBH (Remote Triggered Black Hole)

**Concept:** A trigger router advertises the victim /32 tagged with a well-known community; receiving routers match the community and set next-hop to Null0 (destination-based RTBH).

- [ ] **Task 3.1 — Build the Null0 discard route + community infrastructure.**
  Pick a trigger router (use RR R7 in AS65100, or R19 in AS65200). Define a route-map that, on a static /32 redistributed into BGP, sets a dedicated RTBH community (e.g. `65100:666`) and tags the route for discard. Pre-stage a static route to Null0 for a known test "trigger" prefix.
  *Verify:* `show bgp` shows the trigger prefix carrying the RTBH community.

- [ ] **Task 3.2 — Configure the enforcing routers (ingress PEs/ASBRs).**
  On the edge devices (R1, R2 and ASBRs), add an inbound policy: match the RTBH community → set next-hop to a pre-defined "black-hole" address that statically resolves to Null0. Ensure the Null0 static exists on every enforcing router.
  *Verify:* once triggered, the victim /32 on the edge router resolves to Null0 (`show ip route <victim>` → Null0).

- [ ] **Task 3.3 — Trigger and confirm the black hole.**
  Advertise the test victim /32 from the trigger router. Confirm all enforcing routers drop traffic to it (and only it). Then withdraw and confirm restoration.
  *Verify:* traffic to victim dropped at edge (never enters core); non-victim traffic unaffected; `show bgp <victim>` propagation across the AS.
  *Stretch:* implement **source-based RTBH** using uRPF (Section 5) so the drop is keyed on the attack source, not the destination.

---

## Section 4 — BGP Flowspec

**Concept:** Distribute firewall-like filters via BGP (AFI/SAFI flowspec). Match on 5-tuple, action = drop / rate-limit / redirect-to-VRF.

> **IOS caveat:** BGP Flowspec support on 7200 / IOS 15.2(4)M11 is limited or absent. **Task 4.0:** verify platform support first (`?` under `address-family ipv4 flowspec`). If unsupported, document the limitation and model the equivalent policy with a manual ACL/QoS policy on the PE as the fallback, and treat this section as design/CLI-recognition rather than live data-plane.

- [ ] **Task 4.1 — Define flow rules.**
  On the controller/PE, define flowspec rules matching source, destination, protocol, and L4 port (e.g. drop UDP/53 reflection, rate-limit ICMP to a victim).
  *Verify:* `show bgp ipv4 flowspec` lists the NLRI with its match criteria.

- [ ] **Task 4.2 — Attach actions.**
  Map rules to actions: `drop` (rate 0), `rate-limit` (traffic-rate bytes/sec), `redirect` (to a scrubbing VRF via extended community). Advertise from the controller to the PEs.
  *Verify:* receiving PE installs the flowspec entry into the data plane; `show flowspec` / `show policy-map` reflects the action.

- [ ] **Task 4.3 — Apply on PEs and validate.**
  Enable flowspec reception on R1, R2 (and R14/R16 if testing in AS65200). Generate matching traffic and confirm drop/rate-limit.
  *Verify:* counters increment on the flowspec action; non-matching traffic passes.

---

## Section 5 — Infrastructure Protection

- [ ] **Task 5.1 — CoPP (Control-Plane Policing).**
  On R1, R6, R12 (representative PE + ASBRs) build a CoPP policy: classify BGP/LDP/IGP/ICMP/management into classes, police each (permit critical control at adequate rate, rate-limit/ drop the rest), apply to `control-plane`.
  *Verify:* `show policy-map control-plane` shows conforming/exceeding counters; routing stays up under a ping flood to the router.

- [ ] **Task 5.2 — Bogon / RFC1918 filtering on eBGP.**
  On all eBGP ASBR and PE-CE neighbors, apply inbound + outbound prefix-lists denying RFC1918 (10/8, 172.16/12, 192.168/16) and other bogons/martians. **Caution:** the lab PE-CE links use `192.168.x.x` — scope the filter to customer-advertised prefixes vs. infrastructure, or document the exception so you don't blackhole the lab.
  *Verify:* a bogon advertised by a CE is denied; legitimate customer prefixes pass.

- [ ] **Task 5.3 — AS-PATH filtering (deny private ASNs).**
  On eBGP ingress from ASBRs, deny routes whose AS-PATH contains private ASNs (64512–65534) using an as-path access-list — except the ones intentionally used in this lab (customer ASNs 65024/65025/65910 and the SP ASNs 65100/65200/65300 are themselves private). Decide the policy carefully: filter *unexpected* private ASNs, and/or use `remove-private-as` on egress toward real peers.
  *Verify:* `show bgp` + `show route-map`; a route with a stray private ASN is rejected.

- [ ] **Task 5.4 — uRPF.**
  Enable Unicast RPF on PE-CE interfaces (strict mode where the path is symmetric, loose mode on asymmetric/multi-homed CEs like R10 and R25). Pair with source-based RTBH from Section 3.
  *Verify:* `show ip interface <int>` → uRPF mode + drop counters; spoofed source from a CE is dropped.

---

## Section 6 — LDP / RSVP Security

- [ ] **Task 6.1 — LDP MD5 authentication.**
  On AS65100 LDP neighbors (R1–R8 core), configure `mpls ldp neighbor <id> password` (MD5). Must match on both ends of each LDP session.
  *Verify:* `show mpls ldp neighbor` stays Up; `show mpls ldp neighbor detail` shows "Password: required, in use"; mismatch test drops the session (LSP withdraws).

- [ ] **Task 6.2 — RSVP authentication.**
  Where RSVP-TE is used (if MPLS-TE tunnels were built in a prior workbook — otherwise enable RSVP on R3↔R4↔R5↔R6 core links for this drill), configure RSVP key-based authentication (`ip rsvp authentication key` + key-chain, window-size, lifetime).
  *Verify:* `show ip rsvp authentication` shows active security associations; TE tunnel/path message exchange succeeds only with matching keys.

---

## Completion Checklist

- [ ] IS-IS HMAC-MD5 active on AS65100 + AS65300 (per-level, per-interface, + LSP/SNP)
- [ ] OSPF MD5 active across AS65200 Area 0
- [ ] BGP MD5 on every iBGP + eBGP session
- [ ] GTSM/ttl-security on eBGP; max-prefix on all eBGP peers
- [ ] RTBH: trigger + enforce + withdraw cycle verified
- [ ] Flowspec: platform support checked; rules/actions modeled or applied
- [ ] CoPP applied + verified under load
- [ ] Bogon/RFC1918 + private-ASN + uRPF filters in place (lab exceptions documented)
- [ ] LDP MD5 + RSVP auth verified
- [ ] **GNS3 snapshot taken:** `wb13-security-complete`

---

## Snapshot Points (Progressive)

1. `wb13-s1-igp-auth` — after IGP authentication (Section 1)
2. `wb13-s2-bgp-session` — after BGP MD5/GTSM/max-prefix (Section 2)
3. `wb13-s3-rtbh` — after RTBH works (Section 3)
4. `wb13-s4-flowspec` — after Flowspec (Section 4, or documented limitation)
5. `wb13-s5-infra` — after CoPP/bogon/uRPF (Section 5)
6. `wb13-complete` — after LDP/RSVP auth (Section 6)
