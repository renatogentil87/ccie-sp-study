# Troubleshooting Lab 7: Segment Routing (SR-MPLS) — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2 with SR support) OR EVE-NG (IOS-XRv)
**Topology:** 20 routers — same physical topology, IS-IS + SR replaces OSPF + LDP
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** SR-enabled snapshot (IS-IS with SR extensions, LDP removed)

---

## Lab Context

Your SP network has been modernized from OSPF+LDP to IS-IS+Segment Routing. No LDP sessions exist — all label forwarding uses prefix-SIDs and adjacency-SIDs. This lab tests SR-specific troubleshooting: prefix-SID conflicts, SRGB issues, SR-TE policies, TI-LFA, and Flex-Algo.

**Note:** If running on GNS3/IOS 15.2, SR features are limited. For full SR (Flex-Algo, SR-TE with PCE), use EVE-NG with IOS-XRv. Tickets 1-12 are achievable on IOS 15.2; tickets 13-20 require IOS-XR or can be done as config-review exercises.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change IS-IS area assignments or SRGB ranges (16000-23999)
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers | Prefix-SID (index) | Label |
|---|---|---|---|
| PE | R2 (idx 2), R8 (idx 8), R17 (idx 17), R18 (idx 18) | 64512 | 16002, 16008, 16017, 16018 |
| P (north) | R3 (idx 3), R4 (idx 4), R5 (idx 5), R6 (idx 6), R7 (idx 7) | — | 16003-16007 |
| P (south) | R13 (idx 13), R14 (idx 14), R15 (idx 15), R16 (idx 16) | — | 16013-16016 |
| CE | R1, R9, R11, R12, R19, R20 (no SR participation) |

**SRGB:** 16000-23999 (all routers)
**IGP:** IS-IS single-area 49.0001, wide metrics, SR extensions enabled
**Prefix-SID formula:** Label = 16000 + router-index (e.g., R5 → 16005)
**SR-TE:** Policies for VPN steering (on PEs)
**TI-LFA:** Enabled on all core links for sub-50ms protection
**Flex-Algo 128:** Low-latency path using south-core links (R13-R16) — GigE preferred

---

## Ticket 1

R2's prefix-SID (16002) is not being advertised in IS-IS. Other routers cannot forward to R2 via SR labels (no LFIB entry for 16002 on any router). R2's IS-IS adjacencies are healthy and its loopback IS in the IS-IS database.

Fix the network so that R2 advertises its prefix-SID and all routers install label 16002.

Verify: `show isis database detail` on R3 shows R2's prefix-SID sub-TLV. `show mpls forwarding-table labels 16002` on R5 shows a valid entry.

Score: 2 Points

---

## Ticket 2

Prefix-SID conflict: R4 and R14 are both advertising prefix-SID index 4 (label 16004). This creates ambiguity — routers cannot determine which destination label 16004 refers to. R14 should be using index 14.

Fix the network so that each router has a unique prefix-SID index with no conflicts.

Verify: `show isis segment-routing prefix-sid-map` shows no conflicts. `show mpls forwarding-table labels 16004` resolves to R4 only. Label 16014 resolves to R14.

Score: 2 Points

---

## Ticket 3

R8's prefix-SID (16008) is advertised but the LFIB on R5 shows "no route" for label 16008. R5 has IS-IS reachability to 8.8.8.8 but the SR label is not programmed. Other prefix-SIDs on R5 are working.

Fix the network so that R5's LFIB has a valid entry for label 16008.

Verify: `show mpls forwarding-table labels 16008` on R5 shows outgoing label and interface toward R8.

Score: 2 Points

---

## Ticket 4

SRGB mismatch: R6 has been configured with SRGB 17000-23999 (shifted by 1000) while all other routers use 16000-23999. Packets arriving at R6 with label 16005 (meant for R5) are being dropped because R6's local SRGB doesn't include 16005.

Fix the network so that R6's SRGB is consistent with the rest of the network.

Verify: `show segment-routing mpls state` on R6 shows SRGB 16000-23999. Labels 16000-16020 are properly forwarded by R6.

Score: 3 Points

---

## Ticket 5

Adjacency-SID: R3→R7 has an adjacency-SID allocated, but traffic steered to this adj-SID is not being forwarded by R3. The adj-SID label exists in R3's LFIB but points to a wrong interface (not the R3→R7 link).

Fix the network so that the adjacency-SID for R3→R7 correctly forwards out the right interface.

Verify: `show mpls forwarding-table labels <adj-SID>` on R3 shows outgoing interface Gi2/0 (toward R7).

Score: 3 Points

---

## Ticket 6

SR-TE explicit path: An SR-TE policy on R2 steers Customer_A traffic via explicit SID-list [16003, 16007, 16008] (R3→R7→R8). The policy shows "inactive" — the SID-list cannot be resolved.

Fix the network so that the SR-TE policy becomes active and steers Customer_A traffic.

Verify: `show segment-routing traffic-eng policy` on R2 shows status Active. Customer_A traffic follows the R3→R7→R8 path.

Score: 3 Points

---

## Ticket 7

TI-LFA backup path: R5→R8 (primary path for traffic to R8) has TI-LFA computed, but the backup path is invalid. When R5→R8 link fails, traffic to 16008 is dropped instead of being rerouted via the pre-computed backup.

Fix the network so that TI-LFA provides a valid backup path for R5→R8 link failure.

Verify: `show isis fast-reroute` on R5 shows a valid TI-LFA backup for prefix 8.8.8.8/32. Shut R5→R8 — traffic converges sub-50ms.

Score: 2 Points

---

## Ticket 8

PHP (Penultimate Hop Popping) for prefix-SID: R7 is the penultimate hop for traffic destined to R8 (16008). R7 should pop the label (PHP) but instead forwards with label 16008 intact, causing R8 to receive labeled packets it cannot process.

Fix the network so that PHP works correctly for prefix-SID 16008 at the penultimate hop.

Verify: `show mpls forwarding-table labels 16008` on R7 shows "Pop" as outgoing label. R8 receives packets unlabeled on the connected interface.

Score: 3 Points

---

## Ticket 9

SR-prefer not enabled: Both LDP and SR are running simultaneously (migration state). Traffic should prefer SR labels over LDP, but the LFIB shows LDP labels being used for all destinations. SR labels are allocated but not installed in LFIB.

Fix the network so that SR labels are preferred over LDP where both exist.

Verify: `show mpls forwarding-table` shows SR-originated labels (16xxx) for all PE loopbacks, not LDP labels.

Score: 2 Points

---

## Ticket 10

Mapping-server: R3 is configured as a mapping-server to advertise prefix-SID mappings for routers that don't support SR natively. The mapping entries for R13 (index 13) and R14 (index 14) are configured on R3 but NOT being advertised in IS-IS.

Fix the network so that the mapping-server advertises prefix-to-SID mappings.

Verify: `show isis segment-routing prefix-sid-map received` on R5 shows entries from R3's mapping-server for R13 and R14.

Score: 2 Points

---

## Ticket 11

Microloop avoidance: After R5→R8 link comes back up after a failure, a temporary microloop forms between R3 and R7 during IS-IS convergence. Traffic bounces for 1-2 seconds before stabilizing.

Fix the network so that microloop avoidance is active and prevents transient loops during convergence.

Verify: `show isis microloop-avoidance` shows active. After link restoration, no traffic loops observed (traceroute shows clean path immediately).

Score: 3 Points

---

## Ticket 12

SR global label collision with static MPLS: R2 has a static MPLS label binding (label 16005) configured for a local application. This collides with R5's prefix-SID (also 16005). Traffic to R5 from R2 is misdirected.

Fix the network so that no static label collides with the SR global block.

Verify: `show mpls forwarding-table labels 16005` on R2 correctly points toward R5 (SR prefix-SID). No static conflicts.

Score: 3 Points

---

## Ticket 13

Flex-Algo 128: PEs should steer low-latency VPN traffic via Flex-Algo 128 (south-core path: R13→R14→R15→R16). The Flex-Algo definition exists but R17 is not computing Algo-128 paths. Its LFIB shows no entries for Algo-128 SIDs.

Fix the network so that R17 participates in Flex-Algo 128 and installs Algo-128 prefix-SIDs.

Verify: `show segment-routing mpls forwarding flex-algo 128` on R17 shows entries for Algo-128 destinations. Low-latency traffic follows south-core path.

Score: 4 Points

---

## Ticket 14

SR-TE On-Demand Next-hop (ODN): R2 should create dynamic SR-TE policies for VPN prefixes using color communities. Customer_A routes carry color 100 (low-latency). R2 should auto-create an SR-TE policy to R8 via Flex-Algo 128. The policy is NOT being created.

Fix the network so that ODN creates SR-TE policies based on color community.

Verify: `show segment-routing traffic-eng policy color 100` on R2 shows a dynamically created policy to R8 via Algo-128 path.

Score: 4 Points

---

## Ticket 15

PCE (Path Computation Element): R3 acts as PCE for the network. SR-TE policies on R2 delegate path computation to R3 (PCE-initiated). The PCEP session between R2 and R3 is DOWN.

Fix the network so that the PCEP session establishes and R2 receives PCE-computed paths.

Verify: `show segment-routing traffic-eng pcc` on R2 shows PCEP session to R3 as UP. PCE-delegated policies show computed paths.

Score: 4 Points

---

## Ticket 16

Binding-SID: An SR-TE policy on R2 has binding-SID 15001 allocated. Remote routers should be able to steer traffic to R2 using this binding-SID. However, label 15001 is not in R2's LFIB — it's rejected because it falls outside the SRGB and SRLB.

Fix the network so that the binding-SID is properly allocated and functions for traffic steering.

Verify: `show mpls forwarding-table labels 15001` on R2 shows it mapped to the SR-TE policy. Traffic arriving with label 15001 is steered per-policy.

Score: 4 Points

---

## Ticket 17

IS-IS SR advertisement suppression: R8 has `segment-routing prefix-sid-map advertise-local` configured but is NOT advertising its prefix-SID in LSP. The prefix-SID index is assigned correctly under the loopback interface but isn't in the TLV.

Fix the network so that R8's prefix-SID is advertised in its IS-IS LSP.

Verify: `show isis database detail R8` shows the SR prefix-SID sub-TLV for 8.8.8.8/32 with index 8.

Score: 4 Points

---

## Ticket 18

Network-wide SR label forwarding failure: ALL prefix-SIDs show in IS-IS databases correctly, but NO router has LFIB entries for SR labels. Traffic falls back to IP forwarding. The label manager is not programming labels.

Fix the network so that SR labels are installed in the LFIB across the network.

Verify: `show mpls forwarding-table` on all P routers shows 16xxx labels with valid outgoing interfaces. VPN traffic uses SR labels.

Score: 5 Points

---

## Ticket 19

TI-LFA + Flex-Algo interaction: TI-LFA backup paths for Flex-Algo 128 destinations are computed using the default algorithm (Algo 0) instead of Algo 128. This means backup paths route through links excluded by Algo 128's constraint.

Fix the network so that TI-LFA backup paths honor Flex-Algo constraints.

Verify: `show isis fast-reroute flex-algo 128` shows backup paths only using Algo-128 eligible links. Protection maintains the latency guarantee.

Score: 5 Points

---

## Ticket 20

Complete SR control-plane rebuild: R2 cannot reach R8, R17, or R18 via SR labels. IS-IS adjacencies are ALL UP. Prefix-SIDs are advertised. But LFIB is empty across multiple routers. Multiple issues: SRGB conflict on one router, missing SR config on another, and a static label collision on a third.

Fix the network so that full SR label forwarding works across all paths.

Verify: `ping 8.8.8.8 source 2.2.2.2 mpls` (or equivalent labeled ping) succeeds. `show mpls forwarding-table` on all P routers shows valid SR labels for all PE destinations.

Score: 5 Points

---

## Scoring Summary

| Tickets | Difficulty | Points Each | Total |
|---|---|---|---|
| 1-3 | CCNP-SP (⭐⭐) | 2 | 6 |
| 4-6 | CCNP-SP (⭐⭐⭐) | 3 | 9 |
| 7-9 | CCNP-SP (⭐⭐⭐) | 2-3 | 7 |
| 10-12 | CCNP→CCIE (⭐⭐⭐) | 2-3 | 8 |
| 13-17 | CCIE-SP (⭐⭐⭐⭐) | 4 | 20 |
| 18-20 | CCIE-SP (⭐⭐⭐⭐⭐) | 5 | 15 |
| **Total** | | | **65 Points** |

**Passing:** 49/65 (75%)
**CCIE-ready:** 59/65 (90%)

---

## Injection Notes (for AI fault injector)

**Base state:** IS-IS + SR snapshot (no OSPF, no LDP, all SR prefix-SIDs configured)

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R2 | Missing `segment-routing mpls` under IS-IS or prefix-SID not assigned to loopback |
| 2 | R14 | `prefix-sid index 4` (conflicts with R4) → change to index 14 |
| 3 | R5 | `segment-routing mpls sr-prefer` missing (LDP taking over) |
| 4 | R6 | SRGB set to 17000-23999 |
| 5 | R3 | Adj-SID manually set to wrong interface index |
| 6 | R2 | SID-list references non-existent SID or router down |
| 7 | R5 | `no isis fast-reroute ti-lfa` on interface toward R8 |
| 8 | R8 | `explicit-null` configured (overrides PHP) |
| 9 | All | `segment-routing mpls sr-prefer` missing globally |
| 10 | R3 | Missing `segment-routing prefix-sid-map advertise-local` |
| 11 | All | `microloop-avoidance` not enabled under IS-IS |
| 12 | R2 | `mpls static label 16005 ...` conflicting with SRGB |
| 13 | R17 | Missing `flex-algo 128` participation |
| 14 | R2 | Missing ODN template or color-community not set on routes |
| 15 | R2/R3 | PCEP config mismatch (wrong source/peer IP) |
| 16 | R2 | SRLB not configured (binding-SID outside allocated range) |
| 17 | R8 | `no segment-routing mpls` under interface Loopback0 |
| 18 | Multiple | `segment-routing mpls` not connected to forwarding |
| 19 | TI-LFA | Missing algo-constraint awareness in backup computation |
| 20 | Multiple | SRGB conflict + missing SR + static collision |
