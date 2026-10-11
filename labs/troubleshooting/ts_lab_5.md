# Troubleshooting Lab 5: L2VPN & VPLS — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology, PE-CE interfaces reconfigured as L2
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Load "L2VPN-Base" snapshot (Topology B — OSPF+LDP only, no L3VPN on PE-CE)

---

## Lab Context

The PE-CE interfaces have been reconfigured for Layer-2 services. The MPLS core (OSPF + LDP) remains identical. PEs now provide pseudowire (VPWS/AToM) and VPLS services instead of L3VPN. This lab tests your ability to troubleshoot L2VPN-specific issues.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF, LDP, or core MPLS interface configuration
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers | ASN |
|---|---|---|
| PE | R2, R8, R17, R18 | 64512 |
| P (north) | R3, R4, R5, R6, R7 | — |
| P (south) | R13, R14, R15, R16 | — |
| RR | R3, R7 | 64512 |
| CE (L2) | R1, R9, R11, R12, R19, R20 (bridged/L2 attached) |

**L2VPN Services:**
- VPWS_100: Point-to-point EoMPLS — R1 (on R2) ↔ R9 (on R8), VC-ID 100
- VPWS_200: Point-to-point EoMPLS — R12 (on R2) ↔ R11 (on R8), VC-ID 200
- VPLS_300: Multipoint VPLS — R1 (R2), R9 (R8), R19 (R17), R20 (R18), VFI "CUST-C"
- VPWS_400: PW over TE tunnel — R1 (R2) ↔ R19 (R17), VC-ID 400, preferred-path Tunnel0

**Signaling:** Targeted LDP for VPWS, LDP-based VPLS (RFC 4762) for VPLS_300
**Encapsulation:** Ethernet (default)
**Core:** OSPF area 0, LDP on all P/PE links

---

## Ticket 1

VPWS_100: The pseudowire between R2 and R8 for VC-ID 100 is not establishing. Both PEs can ping each other's loopbacks. The targeted LDP session between R2 and R8 is missing.

Fix the network so that the targeted LDP session establishes and the pseudowire comes UP.

Verify: `show mpls l2transport vc 100` on both R2 and R8 shows status UP.

Score: 2 Points

---

## Ticket 2

VPWS_100: The targeted LDP session between R2 and R8 is UP, but the pseudowire shows "local interface encap mismatch." Both sides have `xconnect` configured but something about the encapsulation doesn't agree.

Fix the network so that the pseudowire negotiation succeeds.

Verify: `show mpls l2transport vc 100 detail` shows no encapsulation mismatch. Status: UP.

Score: 2 Points

---

## Ticket 3

VPWS_200: The pseudowire between R2 and R8 (VC-ID 200) shows "remote VC label not available." R8 has the xconnect configured but is not allocating a VC label for VC-ID 200.

Fix the network so that R8 allocates a VC label and the pseudowire comes UP.

Verify: `show mpls l2transport vc 200` shows status UP on both R2 and R8.

Score: 2 Points

---

## Ticket 4

VPWS_100 is UP (status shows UP/UP) but no traffic passes. R1 can send frames but R9 never receives them. The pseudowire label stack is correct. The issue is on the attachment circuit (AC) side.

Fix the network so that traffic flows through the pseudowire.

Verify: R1 can ping R9's IP (both configured in same L2 broadcast domain). `show mpls l2transport vc 100 detail` shows non-zero bytes in/out.

Score: 3 Points

---

## Ticket 5

VPLS_300: The VFI "CUST-C" on R2 shows pseudowires to R8 and R17 as UP, but the pseudowire to R18 is stuck in "standby" state. All four PEs have the VFI configured with autodiscovery disabled (manual neighbors).

Fix the network so that all four pseudowires in VFI "CUST-C" are in UP state.

Verify: `show vfi CUST-C` on R2 shows all three remote PEs with status UP.

Score: 3 Points

---

## Ticket 6

VPLS_300: All pseudowires in the VFI show UP, but MAC learning is broken. R1 sends an ARP request and R9 responds, but R2 is not learning R1's MAC address in the VPLS MAC table. Flooding works (broadcast reaches all PEs) but unicast is always flooded.

Fix the network so that MAC addresses are learned and unicast traffic is switched (not flooded).

Verify: `show bridge-domain <id> mac-table` or `show l2vpn forwarding` on R2 shows R1's MAC learned on the AC port.

Score: 3 Points

---

## Ticket 7

VPLS_300 split-horizon violation: R9 (on R8) is receiving duplicate frames. Traffic sourced by R19 (on R17) arrives at R8 via the pseudowire from R17 AND via the pseudowire from R2 (who also received it from R17). Split-horizon should prevent PW-to-PW forwarding.

Fix the network so that split-horizon is properly enforced in the VFI.

Verify: R9 receives exactly ONE copy of frames from R19. No duplicate flooding.

Score: 2 Points

---

## Ticket 8

VPWS_400: The pseudowire with preferred-path over Tunnel0 (R2→R17) is DOWN. Tunnel0 is UP. The xconnect shows "preferred path not found." The tunnel exists but the pseudowire won't bind to it.

Fix the network so that VPWS_400 uses Tunnel0 as its preferred path.

Verify: `show mpls l2transport vc 400 detail` shows "preferred path Tunnel0" and status UP.

Score: 3 Points

---

## Ticket 9

VPWS_400: The pseudowire bound to Tunnel0 is UP, but when Tunnel0 goes down, the pseudowire does NOT fail over to the LDP path. It stays DOWN instead of falling back.

Fix the network so that the pseudowire falls back to LDP when the TE tunnel goes down.

Verify: Shut Tunnel0 → `show mpls l2transport vc 400` still shows UP (via LDP fallback path). Bring Tunnel0 back → PW returns to tunnel path.

Score: 3 Points

---

## Ticket 10

VC-ID mismatch: R2 has xconnect with VC-ID 100, but R8 has VC-ID 101 for what should be the same pseudowire. The targeted LDP session is UP but no VC labels are exchanged for this pseudowire.

Fix the network so that both PEs agree on the VC-ID and the pseudowire establishes.

Verify: `show mpls l2transport vc 100` shows UP on both sides.

Score: 2 Points

---

## Ticket 11

VPLS_300: R17's VFI is configured with R2 and R8 as neighbors, but the PW to R18 was accidentally configured as a VPWS (xconnect) instead of a VFI member. R18 has it correct in the VFI. This creates a mismatched signaling state.

Fix the network so that R17↔R18 is properly part of the VPLS VFI mesh.

Verify: `show vfi CUST-C` on R17 shows R18 as a VFI member with status UP.

Score: 3 Points

---

## Ticket 12

VPLS BUM (Broadcast/Unknown-unicast/Multicast) storm: Broadcast traffic is being amplified in the VPLS domain. A bridging loop exists because one PE has accidentally connected two ACs to the same VFI without split-horizon between them.

Fix the network so that no bridging loops exist and BUM traffic is controlled.

Verify: Broadcast from one CE reaches all other CEs exactly once. No MAC flapping in the MAC table. CPU on PEs returns to normal.

Score: 3 Points

---

## Ticket 13

The attachment circuit (AC) on R2 for VPWS_100 is an 802.1Q sub-interface (Fa0/0.100). The CE (R1) is sending tagged frames (VLAN 100) but R2's xconnect is not matching them. The pseudowire is UP but zero traffic counters.

Fix the network so that tagged frames from R1 are properly handed to the pseudowire.

Verify: `show mpls l2transport vc 100 detail` shows incrementing byte counters. R1 can reach R9.

Score: 4 Points

---

## Ticket 14

VLAN interworking: VPWS_200 connects R12 (on R2, VLAN 200) to R11 (on R8, untagged/port-mode). R12 sends tagged frames, R11 expects untagged. The pseudowire is UP but R11 never receives valid Ethernet frames (CRC errors or frame drops).

Fix the network so that VLAN→port interworking strips the tag for R11.

Verify: R12 can ping R11 across the pseudowire. `show mpls l2transport vc 200 detail` shows interworking mode.

Score: 4 Points

---

## Ticket 15

MTU mismatch: VPWS_100 pseudowire is UP but large frames (>1500 bytes) from R1 are being dropped. The PW MTU negotiation shows a mismatch between R2 (MTU 1500) and R8 (MTU 1400). Small pings work but large file transfers fail.

Fix the network so that the MTU is consistent and large frames pass through the pseudowire.

Verify: `ping <R9-IP> source <R1-IP> size 1500 df-bit` from R1 succeeds (no fragmentation, no drops).

Score: 4 Points

---

## Ticket 16

Control-word mismatch: R2 has `pseudowire-class` with control-word enabled for VPWS_100, but R8 does not. The pseudowire shows UP but packets are corrupted (first 4 bytes of payload are being interpreted as control-word by one side).

Fix the network so that control-word usage is consistent on both PEs.

Verify: `show mpls l2transport vc 100 detail` shows control-word status matching on both PEs. Traffic flows cleanly.

Score: 4 Points

---

## Ticket 17

VPLS MAC table overflow: R2's VPLS_300 MAC table has hit its configured limit (100 MACs). New MAC addresses from R1's segment are not being learned, causing all traffic from new hosts to be flooded permanently. Existing MACs work fine.

Fix the network so that the MAC table limit accommodates all hosts or ages correctly.

Verify: `show bridge-domain <id>` shows MAC limit increased or aging properly recycling entries. New MACs are learned.

Score: 4 Points

---

## Ticket 18

Complete VPLS_300 failure: NO pseudowires in the VFI are UP on any PE. All four PEs have the VFI configured but all show "local interface down." The core MPLS transport between all PE loopbacks is working. Targeted LDP sessions are UP.

Fix the network so that VPLS_300 is fully operational across all four PEs.

Verify: `show vfi CUST-C` on all PEs shows all remote peers UP. Traffic flows between all CEs in the VPLS domain.

Score: 5 Points

---

## Ticket 19

VPWS redundancy: VPWS_100 has a backup pseudowire configured (R2↔R17 as backup for R2↔R8). The primary PW (R2↔R8) is DOWN but the backup is NOT activating — it stays in "standby." The backup PW should transition to active when primary fails.

Fix the network so that backup pseudowire activation works correctly.

Verify: With primary PW DOWN, `show mpls l2transport vc 100` on R2 shows backup PW as active/UP. Traffic from R1 reaches R19 (via R17) as backup path.

Score: 5 Points

---

## Ticket 20

Multi-segment pseudowire: A PW from R2 to R18 traverses R8 as a switching-PE (S-PE). The multi-segment PW is not stitching correctly at R8. Both segments (R2↔R8 and R8↔R18) show UP individually, but end-to-end traffic does not flow.

Fix the network so that the multi-segment pseudowire delivers traffic end-to-end.

Verify: R1 (on R2) can reach R20 (on R18) through the stitched pseudowire. `show l2vpn atom vc` on R8 shows both segments stitched.

Score: 5 Points

---

## Scoring Summary

| Tickets | Difficulty | Points Each | Total |
|---|---|---|---|
| 1-3 | CCNP-SP (⭐⭐) | 2 | 6 |
| 4-6 | CCNP-SP (⭐⭐⭐) | 3 | 9 |
| 7-9 | CCNP-SP (⭐⭐⭐) | 2-3 | 8 |
| 10-12 | CCNP→CCIE (⭐⭐⭐) | 2-3 | 8 |
| 13-17 | CCIE-SP (⭐⭐⭐⭐) | 4 | 20 |
| 18-20 | CCIE-SP (⭐⭐⭐⭐⭐) | 5 | 15 |
| **Total** | | | **66 Points** |

**Passing:** 50/66 (75%)
**CCIE-ready:** 60/66 (90%)

---

## Injection Notes (for AI fault injector)

**Base state:** L2VPN-Base snapshot (OSPF + LDP core, PE-CE as L2 ports with xconnect/VFI)

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R2 or R8 | Missing `mpls ldp discovery targeted-hello accept` |
| 2 | R2 or R8 | `interworking ip` on one side, `interworking ethernet` on other |
| 3 | R8 | AC interface shut or `no xconnect` |
| 4 | R2 | AC interface in wrong VLAN or `shutdown` on sub-interface |
| 5 | R18 | `no shutdown` issue or VFI neighbor IP wrong |
| 6 | R2 | `no bridge-domain` or AC not bound to bridge |
| 7 | R2 | Two ACs in same VFI without split-horizon between local ACs |
| 8 | R2 | `preferred-path interface Tunnel0` with wrong tunnel number |
| 9 | R2 | Missing `fallback enable` in pseudowire-class |
| 10 | R8 | VC-ID 101 instead of 100 |
| 11 | R17 | `xconnect 18.18.18.18 300` instead of VFI membership |
| 12 | R8 | Two ACs with same VLAN on same bridge-domain (loop) |
| 13 | R2 | Sub-interface encap mismatch (dot1q 101 vs 100) |
| 14 | R8 | Missing `interworking ethernet` for tag stripping |
| 15 | R8 | `ip mtu 1400` or `mpls mtu 1400` on CE-facing interface |
| 16 | R2 | `control-word` enabled; R8 without it |
| 17 | R2 | `mac limit maximum 100 action drop` |
| 18 | All PEs | VFI binding interface shut on all PEs |
| 19 | R2 | Backup PW missing `pw-class` with `backup delay 0 0` |
| 20 | R8 | Stitching config missing or segment IDs mismatched |
