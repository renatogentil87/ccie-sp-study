# WB14 BGP Labeled Unicast / Unified MPLS — Solutions

**Reference:** `00_topology_reference.md` for IP addressing, NET-IDs, and wiring.
**Platform:** Cisco 7200, IOS 15.2(4)M11. IOS classic syntax.
**Prerequisite:** WB00–WB11 complete (per-AS IGP, LDP, iBGP via RRs, inter-AS links up).

> **Goal:** Build a seamless/unified MPLS LSP across three ASes using BGP Labeled Unicast (BGP-LU, RFC 3107 / `send-label`). PE loopbacks are advertised **with labels** across AS boundaries so an end-to-end LSP exists from R1 (AS65100) to R23 (AS65300) without redistributing one AS's loopbacks into another's IGP. The inter-AS ASBRs are the label-swap boundaries.

---

## Topology of the labeled path

```
AS65100            AS65200                       AS65300
R1 .. R5/R6 === R11/R12 .. R19(RR) .. R12/R21 === R20/R21 .. R23
         ^ eBGP-LU            iBGP-LU          eBGP-LU
Inter-AS BGP-LU links:
  R5 g1/0 10.5.11.5   <-> R11 g1/0 10.5.11.11   (65100<->65200)
  R6 g1/0 10.6.12.6   <-> R12 g1/0 10.6.12.12   (65100<->65200)
  R6 f0/0 10.6.20.6   <-> R20 f0/0 10.6.20.20   (65100<->65300)
  R12 f3/0 20.12.21.12<-> R21 f3/0 20.12.21.21  (65200<->65300)
```

> Each ASBR advertises its own-AS PE loopbacks (and relayed remote loopbacks) with a label over the eBGP-LU session; receiving ASBR sets next-hop-self and re-advertises with a new label into its AS via iBGP-LU through the RR.

---

## Section 1 — eBGP Labeled Unicast on Inter-AS Links

### 1.1 R5 ↔ R11 (AS65100 ↔ AS65200)

```
! R5 (AS65100 ASBR)
router bgp 65100
 neighbor 10.5.11.11 remote-as 65200
 !
 address-family ipv4 unicast
  neighbor 10.5.11.11 activate
  neighbor 10.5.11.11 send-label
  ! advertise AS65100 PE loopbacks with labels
  network 150.1.1.1 mask 255.255.255.255
  network 150.1.2.2 mask 255.255.255.255
 exit-address-family
!
```

```
! R11 (AS65200 ASBR-side / P)
router bgp 65200
 neighbor 10.5.11.5 remote-as 65100
 !
 address-family ipv4 unicast
  neighbor 10.5.11.5 activate
  neighbor 10.5.11.5 send-label
 exit-address-family
!
```

> `send-label` on **both** ends negotiates the Labeled-Unicast SAFI (SAFI 4). Each advertised prefix now carries an MPLS label. The `network` statements inject the PE /32s (they must be in the RIB — present via IGP/connected).

### 1.2 R6 ↔ R12 (AS65100 ↔ AS65200)

```
! R6 (AS65100 ASBR)
router bgp 65100
 neighbor 10.6.12.12 remote-as 65200
 address-family ipv4 unicast
  neighbor 10.6.12.12 activate
  neighbor 10.6.12.12 send-label
  network 150.1.1.1 mask 255.255.255.255
  network 150.1.2.2 mask 255.255.255.255
 exit-address-family
!
```

```
! R12 (AS65200 ASBR)
router bgp 65200
 neighbor 10.6.12.6 remote-as 65100
 address-family ipv4 unicast
  neighbor 10.6.12.6 activate
  neighbor 10.6.12.6 send-label
 exit-address-family
!
```

### 1.3 R6 ↔ R20 (AS65100 ↔ AS65300)

```
! R6 (AS65100 ASBR)
router bgp 65100
 neighbor 10.6.20.20 remote-as 65300
 address-family ipv4 unicast
  neighbor 10.6.20.20 activate
  neighbor 10.6.20.20 send-label
 exit-address-family
!
```

```
! R20 (AS65300 ASBR/PE)
router bgp 65300
 neighbor 10.6.20.6 remote-as 65100
 address-family ipv4 unicast
  neighbor 10.6.20.6 activate
  neighbor 10.6.20.6 send-label
  ! advertise AS65300 PE loopbacks with labels
  network 150.3.20.20 mask 255.255.255.255
  network 150.3.21.21 mask 255.255.255.255
  network 150.3.22.22 mask 255.255.255.255
  network 150.3.23.23 mask 255.255.255.255
 exit-address-family
!
```

### 1.4 R12 ↔ R21 (AS65200 ↔ AS65300)

```
! R12 (AS65200 ASBR)
router bgp 65200
 neighbor 20.12.21.21 remote-as 65300
 address-family ipv4 unicast
  neighbor 20.12.21.21 activate
  neighbor 20.12.21.21 send-label
 exit-address-family
!
```

```
! R21 (AS65300 ASBR/PE)
router bgp 65300
 neighbor 20.12.21.12 remote-as 65200
 address-family ipv4 unicast
  neighbor 20.12.21.12 activate
  neighbor 20.12.21.12 send-label
  network 150.3.20.20 mask 255.255.255.255
  network 150.3.21.21 mask 255.255.255.255
  network 150.3.22.22 mask 255.255.255.255
  network 150.3.23.23 mask 255.255.255.255
 exit-address-family
!
```

---

## Section 2 — iBGP Labeled Unicast (ASBR → RR → remote ASBR/PE)

Inside each AS, the labeled loopbacks learned from the other AS must reach the far PE. Reflect them with labels via the RR, and ASBRs set **next-hop-self** so the next-hop is a local labeled FEC.

### 2.1 AS65200 — ASBRs R11/R12 to RR R19

```
! R11 (and R12 identical to their RR neighbor)
router bgp 65200
 neighbor 150.2.19.19 remote-as 65200
 neighbor 150.2.19.19 update-source Loopback0
 address-family ipv4 unicast
  neighbor 150.2.19.19 activate
  neighbor 150.2.19.19 send-label
  neighbor 150.2.19.19 next-hop-self
 exit-address-family
!
```

```
! R19 (RR, AS65200)
router bgp 65200
 address-family ipv4 unicast
  neighbor 150.2.11.11 activate
  neighbor 150.2.11.11 send-label
  neighbor 150.2.11.11 route-reflector-client
  neighbor 150.2.12.12 activate
  neighbor 150.2.12.12 send-label
  neighbor 150.2.12.12 route-reflector-client
  ! plus PE clients R14/R15/R16 with send-label so they get the LSP
  neighbor 150.2.14.14 activate
  neighbor 150.2.14.14 send-label
  neighbor 150.2.14.14 route-reflector-client
  neighbor 150.2.15.15 activate
  neighbor 150.2.15.15 send-label
  neighbor 150.2.15.15 route-reflector-client
  neighbor 150.2.16.16 activate
  neighbor 150.2.16.16 send-label
  neighbor 150.2.16.16 route-reflector-client
 exit-address-family
!
```

> **next-hop-self on the ASBR toward the RR** is critical: the external loopbacks are learned with an external next-hop (e.g. 10.5.11.5) that the local IGP does not know. `next-hop-self` rewrites it to the ASBR's own loopback, which is a labeled IGP/LDP FEC inside AS65200. Equivalent effect: `mpls bgp forwarding` on the inter-AS interface forces label allocation for the BGP next-hop when next-hop-self is not used.

### 2.2 AS65100 — ASBRs R5/R6 to RRs R7/R8

```
! R5 (and R6) toward RRs R7, R8
router bgp 65100
 address-family ipv4 unicast
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-label
  neighbor 150.1.7.7 next-hop-self
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-label
  neighbor 150.1.8.8 next-hop-self
 exit-address-family
!
! R7/R8 (RRs): activate PEs R1/R2 + ASBRs R5/R6 with send-label + route-reflector-client
```

### 2.3 AS65300 — ASBRs R20/R21 to RR R30

```
! R20 (and R21) toward RR R30
router bgp 65300
 address-family ipv4 unicast
  neighbor 150.3.30.30 activate
  neighbor 150.3.30.30 send-label
  neighbor 150.3.30.30 next-hop-self
 exit-address-family
!
! R30 (RR): activate R20/R21/R22/R23 with send-label + route-reflector-client
```

---

## Section 3 — mpls bgp forwarding (alternative on inter-AS link)

If you do **not** use next-hop-self and want the ASBR to label-switch toward a BGP next-hop reachable only via BGP-LU:

```
! On the inter-AS interface (example R11 g1/0 toward R5)
interface GigabitEthernet1/0
 mpls bgp forwarding
!
```

> This instructs IOS to allocate/forward labels for prefixes whose next-hop is this directly-connected eBGP-LU peer, even though LDP is not run on that link. Use **either** next-hop-self at the RR edge **or** `mpls bgp forwarding` at the AS edge; the lab uses next-hop-self as primary.

---

## Section 4 — End-to-End Verification (R1 → R23 across 3 ASes)

```
! Labels present for remote PE loopbacks
R1#  show bgp ipv4 unicast labels                 ! 150.3.23.23 with in/out label
R1#  show ip bgp 150.3.23.23/32                     ! next-hop = R5 or R6 (local ASBR loopback)
R1#  show mpls forwarding-table 150.3.23.23         ! labeled FEC, outgoing label + interface

! At each boundary — label swap
R5#  show bgp ipv4 unicast labels | include 150.3   ! AS65300 loopbacks learned with label
R11# show mpls forwarding-table | include 150.3     ! label swapped toward R19
R12# show bgp ipv4 unicast labels | include 150.3
R21# show mpls forwarding-table | include 150.3

! Data plane — labeled traceroute end to end
R1#  traceroute mpls ipv4 150.3.23.23/32            ! LSP ping/trace across all 3 ASes
R1#  traceroute 150.3.23.23 source 150.1.1.1        ! label stack visible at each P/ASBR hop
R1#  ping 150.3.23.23 source 150.1.1.1
```

> A successful `traceroute mpls` shows a continuous label-switched path: LDP label inside AS65100 → BGP-LU swap at R5/R6 → LDP inside AS65200 → BGP-LU swap at R12 → LDP inside AS65300 → pop at R23. No AS's loopbacks were leaked into another AS's IGP.

---

## Section 5 — Troubleshooting

### 5.1 Label not in LFIB
- **Cause:** `send-label` missing on one end → prefix learned as plain IPv4, no label.
  `show bgp ipv4 unicast labels` shows `nolabel`. Fix: add `send-label` both ends, clear the session.
- **Cause:** prefix not in RIB when `network` statement issued → BGP won't originate. Confirm loopback is reachable/connected first.

### 5.2 Next-hop not resolved
- **Cause:** ASBR did not set `next-hop-self` toward the RR → PE sees external next-hop (10.5.11.5) with no IGP/label route → prefix invalid/not installed.
  `show ip bgp 150.3.23.23` shows next-hop not resolved / inaccessible. Fix: `next-hop-self` on ASBR under the labeled AF (or `mpls bgp forwarding` on the inter-AS link).
- **Cause:** RR reflected without send-label to PE clients → PE gets the prefix but no label → LSP breaks at the PE.
  Fix: `send-label` on RR toward each PE client.

### 5.3 LSP breaks mid-path
- Check each ASBR's `show mpls forwarding-table` for a continuous swap. A missing entry means the BGP next-hop at that hop is not a labeled FEC (fix with next-hop-self / LDP on the segment).
- `show mpls ldp bindings` inside each AS to confirm the intra-AS LSP that BGP-LU rides on exists.

---

## Completion Checklist

- [ ] eBGP-LU up on all 4 inter-AS links (R5↔R11, R6↔R12, R6↔R20, R12↔R21) with send-label both ends
- [ ] AS65100 PE loopbacks (150.1.1.1/2.2) advertised with labels into AS65200/65300
- [ ] AS65300 PE loopbacks (150.3.20–23) advertised with labels into AS65100/65200
- [ ] ASBRs set next-hop-self under labeled-unicast toward their RRs
- [ ] RRs reflect labeled unicast to PE clients with send-label
- [ ] `show bgp ipv4 unicast labels` shows remote-AS loopbacks with labels on R1
- [ ] `traceroute mpls ipv4 150.3.23.23/32` from R1 succeeds — labeled path across 3 ASes
- [ ] **GNS3 snapshot:** `wb14-bgplu-unified-mpls-complete`
