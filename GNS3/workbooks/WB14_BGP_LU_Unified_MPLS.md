# WB14 — BGP Labeled Unicast & Unified MPLS

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Format:** Tasks only — NO configurations shown. Produce the config yourself, verify, snapshot.
**Prerequisite:** WB00–WB11 complete (per-AS IGP + LDP, RRs, inter-AS ASBR eBGP all operational).

> **Goal:** Build one end-to-end LSP from **R1 (AS65100)** to **R23 (AS65300)** by distributing PE loopbacks as labeled BGP (RFC 3107 / BGP-LU, SAFI 4) and stitching the label at every AS boundary — "Unified MPLS" / Seamless MPLS. No end-to-end IGP; the inter-AS glue is labeled BGP.

---

## Inter-AS Boundary Map (ASBR Pairs for BGP-LU)

| eBGP-LU session | Left AS | Right AS | Underlying link |
|---|---|---|---|
| R5 ↔ R11 | 65100 | 65200 | 10.5.11.0/24 |
| R6 ↔ R12 | 65100 | 65200 | 10.6.12.0/24 |
| R6 ↔ R20 | 65100 | 65300 | 10.6.20.0/24 |
| R12 ↔ R21 | 65200 | 65300 | 20.12.21.0/24 |

**RRs that reflect labeled-unicast internally:** R7/R8 (65100), R19 (65200), R30 (65300).

---

## Section 1 — BGP Labeled Unicast (eBGP-LU at the boundaries)

**Concept:** Each ASBR advertises reachable PE loopbacks (/32s) to the neighbor AS *with a label* (`send-label`). This creates an inter-AS LSP without exchanging IGP routes.

- [ ] **Task 1.1 — Enable the labeled-unicast AF on each eBGP ASBR pair.**
  On R5↔R11, R6↔R12, R6↔R20, R12↔R21 activate `address-family ipv4 unicast` with `send-label` (or `address-family ipv4 labeled-unicast` depending on IOS syntax — verify what 15.2(4)M11 accepts). The eBGP session now carries labels.
  *Verify:* `show bgp ipv4 labeled-unicast summary` (or `show bgp ipv4 unicast` with labels) → neighbor Up, prefixes carry labels.

- [ ] **Task 1.2 — Advertise PE loopbacks with labels.**
  Originate the local-AS PE loopbacks into BGP-LU:
  - AS65100: R1 `150.1.1.1/32`, R2 `150.1.2.2/32`
  - AS65200: R14/R15/R16 loopbacks
  - AS65300: R20–R23 loopbacks (goal endpoint R23 `150.3.23.23/32`)
  Use `network` or redistribute-with-label; ensure each /32 leaves the AS labeled.
  *Verify:* on the far ASBR, `show bgp ipv4 labeled-unicast <PE-loopback>` shows a received label and valid next-hop.

- [ ] **Task 1.3 — Confirm label swap at the boundary (ASBR as LSR).**
  Each ASBR must swap the incoming BGP label (from its RR/LDP side) for the outgoing BGP-LU label (to the eBGP peer). Confirm the ASBR builds a label-swap entry, not a pop/forward-unlabeled.
  *Verify:* `show mpls forwarding-table <remote-PE>` on the ASBR → inbound label → outbound label (swap), not "No Label".

---

## Section 2 — Unified MPLS (Seamless End-to-End LSP)

**Concept:** Stitch per-AS LSPs into one. Inside each AS, LDP (IGP-driven) carries the packet to the ASBR; between ASes, BGP-LU carries it to the next ASBR; the far PE's /32 is reachable with an unbroken label stack from R1 to R23.

- [ ] **Task 2.1 — Pick and document the end-to-end path.**
  Trace the intended label path R1 → (AS65100 LDP) → R6 → (eBGP-LU) → R20 → (AS65300 LDP/BGP-LU) → R23. (Also validate the alternate via R5→R11→R12→R21.) Document which boundary labels you expect at each hop.

- [ ] **Task 2.2 — Ensure loopback /32s are labeled at every hop (no unlabeled gaps).**
  The LSP breaks if any router forwards the destination /32 unlabeled. Confirm LDP labels the PE /32s inside each AS and BGP-LU labels them across boundaries. Watch for the classic break: an ASBR with the /32 only via IGP/BGP but **no label binding**.
  *Verify:* `show mpls forwarding-table 150.3.23.23` returns a label at R1, R6, R20 — all the way through.

- [ ] **Task 2.3 — Resolve BGP-LU next-hop over a labeled path.**
  The received BGP-LU prefix's next-hop (the remote ASBR) must itself resolve to a *labeled* LSP inside the local AS, otherwise recursion fails. Confirm next-hop reachability is label-switched, not plain IGP.
  *Verify:* `show ip cef 150.3.23.23 detail` → recursive, with a label stack imposed.

---

## Section 3 — Redistribute BGP-LU into iBGP (next-hop-self at ASBRs)

**Concept:** The ASBR learns the remote PE /32 via eBGP-LU and must re-advertise it to its own RR with itself as next-hop, so interior routers resolve the /32 to the ASBR's labeled LSP.

- [ ] **Task 3.1 — next-hop-self on ASBRs toward their RRs.**
  On R5, R6 (→ R7/R8), R12 (→ R19), R20/R21 (→ R30): set `next-hop-self` on the iBGP-LU session so the inbound eBGP-LU prefixes are reflected with the ASBR as next-hop.
  *Verify:* on an interior PE, `show bgp ipv4 labeled-unicast <remote-PE>` → next-hop = local ASBR loopback (not the far-AS ASBR).

- [ ] **Task 3.2 — Reflect labeled-unicast on the RRs.**
  Activate `labeled-unicast` on R7/R8, R19, R30 for their clients and set route-reflector-client. The RR reflects the label along with the prefix.
  *Verify:* `show bgp ipv4 labeled-unicast` on the RR shows the prefix with a label and the ASBR next-hop; clients receive it.

- [ ] **Task 3.3 — Confirm the interior LSP resolves to the ASBR.**
  Interior routers (R3/R4 as P) must label-switch toward the ASBR loopback via LDP; the ASBR then imposes the BGP-LU label.
  *Verify:* end PE `show ip cef <remote-PE> detail` shows a 2-label stack (LDP-to-ASBR + BGP-LU).

---

## Section 4 — Verification (End-to-End)

- [ ] **Task 4.1 — Control plane.**
  `show bgp ipv4 labeled-unicast` on R1 shows R23's loopback with a label and a next-hop resolving internally. Repeat from R23 toward R1 (bidirectional LSP).

- [ ] **Task 4.2 — LFIB continuity.**
  `show mpls forwarding-table 150.3.23.23` at R1, then at each transit hop — confirm an unbroken swap chain to R23.

- [ ] **Task 4.3 — Data plane: traceroute mpls.**
  Run `traceroute mpls ipv4 150.3.23.23/32` from R1 (and `traceroute` from R1 loopback to R23 loopback). Confirm every transit hop reports a label stack and the path crosses all 3 ASes.
  *Verify:* `ping 150.3.23.23 source 150.1.1.1` succeeds; traceroute shows MPLS labels at AS65100, AS65200/AS65300 transit hops.

- [ ] **Task 4.4 — Prove no end-to-end IGP.**
  Confirm R1 has **no IGP route** to `150.3.23.23` — reachability exists only via BGP-LU. This validates true Unified MPLS (not a flat IGP).

---

## Section 5 — Troubleshooting Drills

- [ ] **Task 5.1 — Label not in LFIB.**
  Symptom: `show mpls forwarding-table <remote-PE>` shows "No Label" or "Untagged" → LSP breaks there.
  Investigate: `send-label` missing on a BGP-LU neighbor; or the prefix learned via plain BGP/IGP without a label; or `mpls ip` not enabled on the interface.
  *Fix + verify:* label binding reappears; traceroute mpls completes.

- [ ] **Task 5.2 — Next-hop not resolved.**
  Symptom: BGP-LU prefix present but not installed / CEF has no label stack.
  Investigate: ASBR missing `next-hop-self` (next-hop points to an unreachable far-AS address); or the next-hop resolves via a non-labeled path (recursion to a plain IGP route).
  *Fix + verify:* `show ip cef <remote-PE> detail` → recursive labeled path; prefix becomes best.

- [ ] **Task 5.3 — Partial LSP / asymmetric path.**
  Symptom: forward works, return fails (or vice versa). Verify both directions advertise + label the opposite PE loopback. Check that both ASBRs do next-hop-self and both RRs reflect labeled-unicast.
  *Fix + verify:* bidirectional `traceroute mpls` clean.

---

## Completion Checklist

- [ ] eBGP-LU Up on all 4 ASBR pairs (R5↔R11, R6↔R12, R6↔R20, R12↔R21)
- [ ] PE loopbacks advertised **with labels** in every AS
- [ ] ASBRs swap labels (not pop/unlabeled) at boundaries
- [ ] next-hop-self on all ASBRs toward RRs
- [ ] RRs reflect labeled-unicast to clients
- [ ] R1→R23 end-to-end LSP: `traceroute mpls` shows labeled path across 3 ASes
- [ ] Confirmed no end-to-end IGP route to R23 loopback
- [ ] All 3 troubleshooting root causes reproduced + fixed
- [ ] **GNS3 snapshot taken:** `wb14-bgp-lu-unified-complete`

---

## Snapshot Points (Progressive)

1. `wb14-s1-ebgp-lu` — after eBGP-LU up at all boundaries (Section 1)
2. `wb14-s3-nhs-rr` — after next-hop-self + RR reflection (Section 3)
3. `wb14-s4-e2e` — after end-to-end traceroute mpls works (Section 4)
4. `wb14-complete` — after troubleshooting drills (Section 5)
