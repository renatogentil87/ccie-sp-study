# Troubleshooting Lab 6: MPLS Traffic Engineering Advanced — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Golden-state snapshot (OSPF + LDP + L3VPN + TE tunnels working)

---

## Lab Context

Your SP core has multiple TE tunnels carrying VPN traffic. This lab tests advanced TE scenarios: CSPF failures, bandwidth admission, affinity/admin-groups, FRR (node/link protection), make-before-break, autoroute vs forwarding-adjacency, and inter-area TE.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF areas, AS numbers, or VPN configurations
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers |
|---|---|
| PE (headend) | R2, R17 |
| PE (tailend) | R8, R18 |
| P (north core) | R3, R4, R5, R6, R7 |
| P (south core) | R13, R14, R15, R16 |
| RR | R3, R7 |
| CE | R1, R9, R11, R12, R19, R20 |

**TE Tunnels (pre-configured):**
- Tunnel0: R2→R8 (dynamic path, BW 50000, autoroute announce)
- Tunnel1: R2→R8 (explicit: R3→R4→R5→R8, BW 80000, backup for Tunnel0)
- Tunnel2: R2→R17 (dynamic, BW 30000, autoroute announce)
- Tunnel3: R17→R8 (explicit: R13→R14→R7→R8, BW 60000, affinity 0x1)
- Tunnel10: R2→R8 (FRR — NHOP backup via R6)

**IGP:** OSPF Area 0 with TE extensions (opaque LSAs)
**RSVP:** Bandwidth 100000 on all core interfaces
**Affinities:** R6↔R13 link colored "gold" (0x1); all others default (0x0)

---

## Ticket 1

Tunnel0 on R2 (dynamic path to R8) is DOWN. The TE topology database on R2 shows all routers and links populated. RSVP signaling is not succeeding. Debug shows "no path satisfying constraints found."

Fix the network so that Tunnel0 comes UP with a valid dynamic path.

Verify: `show mpls traffic-eng tunnels tunnel0` shows Admin/Oper: up/up.

Score: 2 Points

---

## Ticket 2

Tunnel1 on R2 (explicit path R3→R4→R5→R8) will not signal. RSVP PATH message reaches R3 but is rejected. The explicit-path is configured correctly and all routers in the path have TE enabled.

Fix the network so that Tunnel1 signals along the explicit path.

Verify: `show mpls traffic-eng tunnels tunnel1` shows state UP with ERO matching R3→R4→R5→R8.

Score: 2 Points

---

## Ticket 3

Tunnel0 is UP but taking a 5-hop path (R2→R6→R5→R4→R3→R7→R8) when a 2-hop path exists (R2→R3→R7→R8 or R2→R6→R5→R8). The tunnel is dynamic — CSPF should find the shortest path.

Fix the network so that Tunnel0 uses the shortest constrained path.

Verify: `show mpls traffic-eng tunnels tunnel0` shows ERO with minimum hops (2-3).

Score: 2 Points

---

## Ticket 4

Tunnel2 (R2→R17, dynamic, BW 30000) is DOWN. The TE topology database on R2 does NOT contain R17's loopback or ANY south-core links (R13-R16). North-core links are all present. OSPF adjacency between R6 and R13 is FULL.

Fix the network so that TE topology includes south-core routers and Tunnel2 comes UP.

Verify: `show mpls traffic-eng topology` on R2 includes R13, R14, R15, R16, R17. Tunnel2 is UP.

Score: 3 Points

---

## Ticket 5

Tunnel3 on R17 (explicit: R13→R14→R7→R8, affinity 0x1) is DOWN. The tunnel requires links with attribute-flag 0x1, but only the R6↔R13 link has this flag. The explicit path doesn't even traverse R6↔R13.

Fix the network so that Tunnel3 can signal along its explicit path with proper affinity constraints.

Verify: `show mpls traffic-eng tunnels tunnel3` shows UP with ERO R13→R14→R7→R8. Affinity constraint satisfied.

Score: 3 Points

---

## Ticket 6

RSVP bandwidth admission control: Tunnel1 (BW 80000) on R2 is requesting more bandwidth than available on R4→R5 link. The link shows only 50000 remaining (because Tunnel0 already reserved 50000 on that link). Tunnel1 fails to signal.

Fix the network so that Tunnel1 can signal with its requested bandwidth (80000) without removing Tunnel0.

Verify: Both Tunnel0 and Tunnel1 are UP. `show ip rsvp interface` shows sufficient bandwidth on the path.

Score: 3 Points

---

## Ticket 7

TE tunnel preemption: Tunnel1 (priority setup 6, hold 6) should preempt Tunnel0 (priority setup 7, hold 7) on the shared link when bandwidth is insufficient. But preemption is not happening — Tunnel1 stays DOWN while Tunnel0 holds the bandwidth.

Fix the network so that preemption works correctly.

Verify: Tunnel1 comes UP (preempts Tunnel0's reservation). Tunnel0 re-signals via alternate path.

Score: 2 Points

---

## Ticket 8

Autoroute announce on Tunnel0: The tunnel is UP but VPN traffic from R2 to R8 is NOT using it. CEF on R2 shows the next-hop for 8.8.8.8 as a physical interface (not Tunnel0). Autoroute announce is supposedly configured.

Fix the network so that autoroute injects R8's loopback into R2's IGP RIB via Tunnel0.

Verify: `show ip route 8.8.8.8` on R2 shows Tunnel0 as outgoing interface. `show ip cef vrf Customer_A 9.9.9.9` shows Tunnel0.

Score: 3 Points

---

## Ticket 9

Make-before-break (MBB) is not working: When R2's Tunnel0 is re-optimized (timer fires or `mpls traffic-eng reoptimize`), traffic drops for 2-3 seconds. MBB should provide hitless re-optimization.

Fix the network so that re-optimization is hitless (MBB active).

Verify: Run `mpls traffic-eng reoptimize` on R2 — traffic counters on Tunnel0 show zero drops during transition. New path is computed before old is torn down.

Score: 2 Points

---

## Ticket 10

FRR (Fast Reroute) link protection: Tunnel10 on R2 is configured with FRR but the backup tunnel is not created. When the primary path's first link (R2→R3) goes down, traffic is lost for full IGP convergence time instead of sub-50ms.

Fix the network so that FRR provides fast protection for Tunnel10's primary path.

Verify: `show mpls traffic-eng tunnels tunnel10` shows "FRR: Enabled, Protection: Ready." `show mpls traffic-eng fast-reroute database` shows a backup path.

Score: 2 Points

---

## Ticket 11

FRR node protection: The FRR backup tunnel protects against R3 node failure (not just R2→R3 link failure). However, the bypass tunnel is only configured as link-protection. If R3 itself fails, the backup path still traverses R3.

Fix the network so that node-protection is provided (bypass avoids R3 entirely).

Verify: `show mpls traffic-eng tunnels` shows bypass tunnel as node-protecting. Bypass path avoids R3.

Score: 3 Points

---

## Ticket 12

Path-option fallback: Tunnel1 has `path-option 1 explicit` and `path-option 2 dynamic`. The explicit path is broken (link down) but the tunnel is NOT falling back to dynamic. It stays DOWN.

Fix the network so that path-option fallback works (dynamic path used when explicit fails).

Verify: With explicit path broken, `show mpls traffic-eng tunnels tunnel1` shows UP with "path-option 2 (dynamic)" active.

Score: 3 Points

---

## Ticket 13

Inter-area TE: Tunnel2 (R2→R17) crosses from the north-core into the south-core. The tunnel uses a loose hop in the explicit path (`next-address loose 17.17.17.17`). But the ABR (R6/R13 boundary) is not expanding the loose hop. Tunnel stays DOWN.

Fix the network so that the loose-hop ERO is expanded at the area boundary and Tunnel2 signals.

Verify: `show mpls traffic-eng tunnels tunnel2` shows UP with full ERO expanded through south-core.

Score: 4 Points

---

## Ticket 14

Forwarding-adjacency: Tunnel0 (R2→R8) is configured with `mpls traffic-eng forwarding-adjacency` instead of autoroute. The tunnel should appear as a link in OSPF. However, other routers in the network are NOT seeing this TE "link" in their OSPF database.

Fix the network so that the forwarding-adjacency is advertised in OSPF and used for forwarding.

Verify: `show ip ospf database` on R3 shows a link (Type-1 LSA) corresponding to R2's Tunnel0. Remote routers' SPF considers this link.

Score: 4 Points

---

## Ticket 15

RSVP reservation tear: After R5 is reloaded, all TE tunnels traversing R5 remain UP on the headend (R2) but traffic is being blackholed at R5. R5 lost the RSVP soft-state but the headend hasn't detected the failure.

Fix the network so that RSVP state is refreshed/resynchronized after the reload and traffic flows.

Verify: `show ip rsvp reservation` on R5 shows active reservations matching Tunnel0/Tunnel1. Traffic flows through R5.

Score: 4 Points

---

## Ticket 16

Tunnel load-sharing: R2 has both Tunnel0 and Tunnel1 to R8 UP, but all traffic uses Tunnel0. Both tunnels have `autoroute announce` and `tunnel mpls traffic-eng load-share` but ECMP over tunnels is not happening.

Fix the network so that traffic is load-shared across both tunnels to R8.

Verify: `show ip cef 8.8.8.8` on R2 shows BOTH Tunnel0 and Tunnel1 in the load-share set.

Score: 4 Points

---

## Ticket 17

Auto-bandwidth: Tunnel0 has `auto-bw` configured to adjust bandwidth based on traffic. The tunnel currently reserves 50000 but actual traffic is 90000. Auto-bandwidth should have increased the reservation but it hasn't changed in 24 hours.

Fix the network so that auto-bandwidth adjusts Tunnel0's reservation to match actual load.

Verify: `show mpls traffic-eng tunnels tunnel0` shows auto-bw active with reservation adjusted toward actual traffic rate.

Score: 4 Points

---

## Ticket 18

All TE tunnels on R2 (Tunnel0, Tunnel1, Tunnel2) are simultaneously DOWN. The TE topology database is EMPTY. OSPF adjacencies are all FULL. The issue is TE not populating from OSPF.

Fix the network so that the TE topology populates and all tunnels re-signal.

Verify: `show mpls traffic-eng topology` shows all links. All tunnels come UP.

Score: 5 Points

---

## Ticket 19

RSVP authentication: Tunnel1's explicit path traverses R3→R4→R5. RSVP authentication has been enabled between R4 and R5 but the keys don't match. Tunnel1 was working before auth was added — now PATH messages are rejected at R5.

Fix the network so that RSVP authentication works and Tunnel1 signals through the authenticated link.

Verify: `show ip rsvp interface` shows authentication active. Tunnel1 is UP. `debug ip rsvp` shows no auth failures.

Score: 5 Points

---

## Ticket 20

Complex multi-tunnel failure: Tunnel0 (autoroute, carries Customer_A), Tunnel2 (autoroute, carries Customer_D to R17), and FRR backup are all impacted simultaneously. Multiple root causes: bandwidth exhaustion on one link, affinity mismatch on another, and missing TE on a third. All VPN traffic is falling back to LDP paths.

Fix the network so that all TE tunnels re-establish and VPN traffic rides the intended tunnels.

Verify: All tunnels UP. `show ip cef vrf Customer_A 9.9.9.9` on R2 shows Tunnel0. `show ip cef vrf Customer_D 19.19.19.19` on R2 shows Tunnel2.

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

**Base state:** Golden-state + all TE tunnels pre-configured and working

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R2 | Tunnel0 BW set to 999999 (exceeds all links) |
| 2 | R3 | `no mpls traffic-eng tunnels` under interface to R4 |
| 3 | R5→R8 Gi2/0 | `mpls traffic-eng attribute-flags 0x2` (forces CSPF away) |
| 4 | R13 | `no mpls traffic-eng area 0` under OSPF (TE opaque LSAs not generated) |
| 5 | R14 | Link R13→R14 attribute-flags 0x0 (doesn't satisfy Tunnel3's 0x1 requirement) |
| 6 | R4 Gi1/0 | `ip rsvp bandwidth 50000` (not enough for 80000) |
| 7 | Tunnel0 | Setup/hold priority same as Tunnel1 (no preemption possible) |
| 8 | R2 Tunnel0 | `no tunnel mpls traffic-eng autoroute announce` |
| 9 | R2 | `no mpls traffic-eng reoptimize events` or MBB disabled |
| 10 | R2 | Missing backup tunnel or `no mpls traffic-eng fast-reroute` on Tunnel10 |
| 11 | Bypass tunnel | Path traverses R3 (link-protect only, not node-protect) |
| 12 | R2 Tunnel1 | Missing `path-option 2 dynamic` |
| 13 | R6/R13 | Missing `mpls traffic-eng area 0 stub` or loose-hop expansion disabled |
| 14 | R2 Tunnel0 | Missing `ospf-adjacency` or OSPF not including the FA link |
| 15 | R5 | `clear ip rsvp reservation *` (simulates state loss) |
| 16 | R2 | Different `load-share` values or missing on one tunnel |
| 17 | R2 Tunnel0 | `auto-bw max-bw 50000` (capped too low) |
| 18 | R2 | `no mpls traffic-eng area 0` under OSPF |
| 19 | R4 or R5 | RSVP auth key mismatch |
| 20 | Multiple | BW exhaustion + affinity mismatch + missing TE on link |
