# Troubleshooting Lab 1: IS-IS as SP IGP — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology, OSPF replaced with IS-IS
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Load "IS-IS Migration" snapshot (Topology D) or inject IS-IS base config

---

## Lab Context

Your SP network has been migrated from OSPF to IS-IS. The 20-router topology remains identical but now runs IS-IS as the sole IGP. MPLS LDP, L3VPN, and TE all run on top of IS-IS. This lab tests your ability to troubleshoot IS-IS-specific issues in an SP core.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change IS-IS area assignments or NET addresses
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers | ASN | IS-IS Level |
|---|---|---|---|
| PE | R2, R8, R17, R18 | 64512 | L1/L2 |
| P (north core) | R3, R4, R5, R6, R7 | — | L2-only |
| P (south core) | R13, R14, R15, R16 | — | L1/L2 |
| RR | R3, R7 | 64512 | — |
| CE | R1 (AS 65001), R9 (AS 65001), R11 (AS 65011), R12 (AS 65012), R19 (AS 65019), R20 (AS 65020) |

**IS-IS Areas:**
- Area 49.0001: R2, R3, R4, R5, R6, R7, R8 (North/Core)
- Area 49.0002: R13, R14, R15, R16, R17, R18 (South)

**NET Addressing:** `49.000X.RRRR.RRRR.RRRR.00` (derived from loopback)
- Example: R2 (2.2.2.2) → `49.0001.0002.0002.0002.00`
- Example: R17 (17.17.17.17) → `49.0002.0017.0017.0017.00`

**IS-IS Metric Style:** Wide metrics, reference-bandwidth 10000
**Label Distribution:** LDP on all core interfaces
**MPLS TE:** Enabled with IS-IS TE extensions

---

## Ticket 1

Two directly connected P routers in the north core (R4 and R5) cannot form an IS-IS adjacency. Both interfaces show UP/UP. Other IS-IS adjacencies on both routers are healthy. LDP session on this link is also missing.

Fix the network so that the IS-IS adjacency forms between R4 and R5.

Verify: `show clns neighbors` on both R4 and R5 shows the adjacency UP. `show mpls ldp neighbor` shows the LDP session established on that link.

Score: 2 Points

---

## Ticket 2

R2 (PE) has an IS-IS adjacency with R3 (its directly connected P/RR), but R2's loopback (2.2.2.2) is NOT appearing in the IS-IS database of remote routers. Other PE loopbacks are visible network-wide.

Fix the network so that R2's loopback is advertised into IS-IS and reachable from all routers.

Verify: `show isis database detail` on R8 shows R2's loopback prefix. `ping 2.2.2.2 source 8.8.8.8` succeeds.

Score: 2 Points

---

## Ticket 3

IS-IS adjacency between R6 (L2-only, north core) and R13 (L1/L2, south core) connecting the two areas is stuck in INIT state. Both routers see each other's IIH packets but never transition to UP.

Fix the network so that the adjacency reaches UP state.

Verify: `show clns neighbors` shows UP state. `show isis neighbors detail` shows the adjacency as L2.

Score: 2 Points

---

## Ticket 4

R17 (PE, Area 49.0002) cannot reach R2's loopback (2.2.2.2) despite having healthy IS-IS adjacencies within its area. R17's IS-IS routing table shows routes for other Area 49.0002 routers but is missing ALL routes from Area 49.0001.

Fix the network so that R17 has full reachability to all PE and P loopbacks.

Verify: `show isis route` on R17 shows prefixes from both areas. `ping 2.2.2.2 source 17.17.17.17` succeeds.

Score: 3 Points

---

## Ticket 5

LDP sessions are NOT forming between R2 and R8 even though both routers have full IS-IS reachability to each other's loopbacks. LDP sessions between other router pairs are working correctly.

Fix the network so that the LDP session between R2 and R8 establishes.

Verify: `show mpls ldp neighbor 8.8.8.8` on R2 shows the session as Operational.

Score: 3 Points

---

## Ticket 6

IS-IS metric manipulation has caused a routing loop between R3 and R6. Traceroute from R2 to R8 shows packets bouncing between R3 and R6 with TTL expiry. All adjacencies are UP.

Fix the network so that traffic follows a loop-free path from R2 to R8.

Verify: `traceroute 8.8.8.8 source 2.2.2.2` shows a clean path with no loops. All IS-IS metrics are consistent.

Score: 3 Points

---

## Ticket 7

R18 (PE, south) is advertising its loopback (18.18.18.18) into IS-IS, and other routers see it in their LSDB. However, the route is NOT being installed in the RIB of north-core routers. `show isis rib` shows the prefix but the next-hop is unreachable.

Fix the network so that R18's loopback is installed in the RIB of all routers.

Verify: `show ip route 18.18.18.18` on R2 shows an IS-IS route. `ping 18.18.18.18 source 2.2.2.2` succeeds.

Score: 2 Points

---

## Ticket 8

IS-IS authentication has been configured between R3 and R7 (the two RR routers). The adjacency has dropped and IIH packets are being rejected. One router shows authentication mismatch in debug output.

Fix the network so that IS-IS authentication works and the adjacency between R3 and R7 reforms.

Verify: `show clns neighbors` on R3 shows R7 as UP. `show isis neighbors detail` confirms authentication is active.

Score: 3 Points

---

## Ticket 9

R13 (P, south, L1/L2) is leaking L1 routes into L2 that should NOT be leaked. Internal south-area /30 transit prefixes are appearing in the L2 database, causing suboptimal routing for north-core routers.

Fix the network so that only appropriate routes (loopbacks, summary) are leaked between levels.

Verify: `show isis database level-2 detail` on R3 does NOT show /30 transit links from Area 49.0002 that should remain L1-internal.

Score: 2 Points

---

## Ticket 10

The IS-IS SPF computation on R5 is running excessively (multiple times per second) causing high CPU. There are no actual topology changes occurring. Other P routers show normal SPF behavior.

Fix the network so that SPF runs at a normal rate on R5.

Verify: `show isis spf-log` on R5 shows reasonable SPF intervals (throttled). CPU utilization drops to normal.

Score: 2 Points

---

## Ticket 11

BFD has been configured for IS-IS on the link between R5 and R8. The BFD session shows UP, but when the link quality degrades, IS-IS still takes the full hello dead interval (30s) to detect the neighbor loss instead of using BFD fast detection.

Fix the network so that BFD is actually coupled to IS-IS neighbor detection.

Verify: `show bfd neighbors` shows the session tied to IS-IS. Simulate a failure — IS-IS reconverges within BFD timer values (sub-second).

Score: 3 Points

---

## Ticket 12

R14 (south core) has been misconfigured with `metric-style narrow` while all other routers use `metric-style wide`. The adjacency with R13 is UP but R14's LSP contains narrow-format TLVs that cannot carry TE information.

Fix the network so that IS-IS metric style is consistent across the network and TE extensions propagate.

Verify: `show isis database detail R14` shows wide-metric TLVs. MPLS TE topology database on R2 includes R14's links.

Score: 3 Points

---

## Ticket 13

R8 (PE, north) is receiving IS-IS routes with a metric that appears correct, but the route to R17's loopback (17.17.17.17) has an inflated metric compared to what it should be. The physical path is R8→R7→R14→R13→R17 (4 hops) but the metric suggests a 7-hop path.

Fix the network so that the IS-IS metric to R17 reflects the actual shortest path cost.

Verify: `show isis route 17.17.17.17` on R8 shows the correct metric matching the 4-hop path. Traceroute confirms the direct path.

Score: 4 Points

---

## Ticket 14

IS-IS overload bit (OL) is set on R7 (P/RR, north core), causing all transit traffic to avoid it. R7 is healthy and should be carrying transit traffic. The overload bit was NOT manually configured — it appears triggered by a condition.

Fix the network so that the overload bit is cleared and R7 carries transit traffic normally.

Verify: `show isis database` — R7's LSP does NOT have the OL bit set. Traffic transits through R7.

Score: 4 Points

---

## Ticket 15

Traffic from R2 to R17 should have TWO equal-cost paths (via R3→R13 and via R6→R13) but only ONE path is installed in the RIB. Both paths exist in the LSDB with identical metrics. R2 has `maximum-paths 16` configured.

Fix the network so that both equal-cost paths are installed in the RIB and CEF.

Verify: `show ip route 17.17.17.17` on R2 shows two next-hops. `show ip cef 17.17.17.17` shows load-sharing.

Score: 4 Points

---

## Ticket 16

The L1/L2 boundary router R16 (south) cannot form an L1 adjacency with R15 (south P). Both are configured as L1/L2. The L2 adjacency between them is FINE, but L1 is stuck in INIT.

Fix the network so that both L1 and L2 adjacencies form between R15 and R16.

Verify: `show isis neighbors` shows both L1 and L2 adjacencies in UP state.

Score: 4 Points

---

## Ticket 17

IS-IS prefix suppression has been configured on R5's transit links. However, R5's loopback (5.5.5.5) is ALSO being suppressed incorrectly — it should NEVER be suppressed. R5's loopback is missing from remote routers' tables.

Fix the network so that R5's loopback is advertised while transit /30 prefixes remain suppressed.

Verify: `show isis database detail` shows R5's loopback in its LSP. Remote routers have 5.5.5.5 in their RIB. Transit links remain suppressed.

Score: 4 Points

---

## Ticket 18

Multiple IS-IS adjacencies across the south core (R13, R14, R15, R16) are oscillating — flapping every 20-30 seconds. CPU is elevated. Interfaces show no errors. Physical layer is clean.

Fix the network so that all IS-IS adjacencies in the south core stabilize and remain UP.

Verify: `show isis neighbors` shows all adjacencies stable (uptime increasing). No further flaps for 2+ minutes.

Score: 5 Points

---

## Ticket 19

R2 can reach R17's loopback but the path traverses 6 hops when the optimal path should be 4 hops. IS-IS metrics on individual links appear correct. One router in the path is running narrow metrics while others use wide — causing inconsistent metric comparison.

Fix the network so that path cost is consistent and R2 reaches R17 via the optimal 4-hop path.

Verify: `traceroute 17.17.17.17 source 2.2.2.2` shows exactly 4 intermediate hops. IS-IS metric style is consistent network-wide.

Score: 5 Points

---

## Ticket 20

Full inter-area failure: R17 and R18 (Area 49.0002 PEs) have lost ALL reachability to Area 49.0001 routers. Local adjacencies within Area 49.0002 are fine. The L2 backbone appears fragmented between north and south. Multiple issues may be contributing simultaneously.

Fix the network so that full inter-area reachability is restored.

Verify: `ping 2.2.2.2 source 17.17.17.17` succeeds. `ping 8.8.8.8 source 18.18.18.18` succeeds. `show isis route` on R17 shows prefixes from Area 49.0001.

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

**Base state required:** IS-IS running on all P/PE routers (replaces OSPF). LDP + MPLS on all core interfaces. Same L3VPN, same RRs, same TE. Load Topology D snapshot.

**Fault injection map:**
| Ticket | Router(s) | Fault Type |
|---|---|---|
| 1 | R4 or R5 | Interface circuit-type mismatch or passive |
| 2 | R2 | Missing `ip router isis` on Loopback0 |
| 3 | R6 or R13 | circuit-type L1-only on one side of L2 link |
| 4 | R13 or R14 | L2 adjacency to north broken (ATT bit / route leak) |
| 5 | R2 or R8 | `mpls ldp discovery transport-address interface` (non-loopback) |
| 6 | R3 or R6 | Interface metric set to 1 creating asymmetric cost |
| 7 | R14 | Missing `ip router isis` on link toward R7 |
| 8 | R3 or R7 | Authentication key mismatch |
| 9 | R13 | Route-leak map too permissive |
| 10 | R5 | `lsp-gen-interval 1` or `spf-interval 1 1 1` |
| 11 | R5 or R8 | `bfd all-interfaces` missing under IS-IS |
| 12 | R14 | `metric-style narrow` |
| 13 | R14 | Interface metric inflated on one link |
| 14 | R7 | `set-overload-bit on-startup wait-for-bgp` with BGP not converging |
| 15 | R2 | `no mpls traffic-eng multipath` or interface-level issue |
| 16 | R15 or R16 | `isis circuit-type level-2-only` on one side |
| 17 | R5 | `isis prefix-suppression` on Loopback0 |
| 18 | R13-R16 | Hello-interval mismatch (10 vs 3) causing dead-timer expiry |
| 19 | R14 | `metric-style narrow` while others wide |
| 20 | R6+R14 | Multiple: L2 link R6↔R13 shut + R7↔R14 authentication fail |
