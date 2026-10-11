# Troubleshooting Lab 2: MPLS SP Core — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — 4 PEs (R2, R8, R17, R18), 9 P routers, 7 CEs
**Difficulty:** CCNP-SP to CCIE-SP progressive

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change routing protocol boundaries or remove OSPF area assignments
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
| P (core) | R3, R4, R5, R6, R7 | — |
| P (south) | R13, R14, R15, R16 | — |
| RR | R3, R7 | 64512 |
| CE | R1, R9 (AS 65001), R11 (AS 65011), R12 (AS 65012), R19 (AS 65019), R20 (AS 65020) |

**VPNs:** Customer_A (R1↔R9), Customer_B (R12↔R11), Customer_D (R19), Customer_E (R20)
**TE:** Tunnel0 on R2 → R8 (dynamic path, autoroute announce)
**IGP:** OSPF area 0, all P/PE routers
**Label Distribution:** LDP on all core interfaces
**MPLS TE:** Enabled on all core interfaces with RSVP bandwidth 100000

---

## Ticket 1  - DONE

A P router has lost an OSPF adjacency with one of its directly connected P router neighbors. Both routers are reachable via alternate paths — there is no total outage. The LDP session on that specific link is also missing.

Fix the network so that the OSPF adjacency forms and the LDP session establishes on the affected link.

Verify: `show ip ospf neighbor` on both affected routers shows FULL state. `show mpls ldp neighbor` shows the session established.

Score: 2 Points

---

## Ticket 2 - DONE

Traffic from R2 to R8 is taking a suboptimal path. The traceroute from R2 to 8.8.8.8 shows more hops than the shortest path should require. All OSPF adjacencies that ARE formed are in FULL state. No reachability issues exist.

Fix the network so that traffic between R2 and R8 follows the shortest IGP path.

Verify: `traceroute 8.8.8.8 source 2.2.2.2` shows the minimum number of hops.

Score: 2 Points

---

## Ticket 3 - DONE

An LDP session between two directly connected P routers is not forming. The OSPF adjacency on that same link IS in FULL state. MPLS labels for destinations reachable via that link are missing from the LFIB.

Fix the network so that the LDP session establishes and labels are allocated for all prefixes.

Verify: `show mpls ldp neighbor` shows the session UP on the affected link. `show mpls forwarding-table` shows labels for all PE loopbacks.

Score: 2 Points

---

## Ticket 4 - DONE

R1 (Customer_A CE) cannot ping R9's loopback 9.9.9.9. The transport between PE routers (R2 can ping R8 loopback 8.8.8.8) is working correctly. The issue is in the VPN control plane.

Fix the network so that R1 can ping 9.9.9.9.

Verify: `ping 9.9.9.9` from R1 succeeds. `show ip bgp vpnv4 vrf Customer_A` on R8 shows R1's routes.

Score: 3 Points

---

## Ticket 5 - DONE

R12 (Customer_B CE) cannot reach R11 (Customer_B CE) at 11.11.11.11. Other VPNs (Customer_A) ARE working correctly. The problem is isolated to Customer_B.

Fix the network so that R12 can ping R11's loopback 11.11.11.11.

Verify: `ping 11.11.11.11 source 12.12.12.12` from R12 succeeds. `show ip route vrf Customer_B` on R2 and R8 shows routes from both CEs.

Score: 3 Points

---

## Ticket 6 - DONE

Customer_A VPN has a route leaking problem. R1 (Customer_A) can suddenly reach R12's addresses (Customer_B). This should NOT be possible — VPN isolation is broken.

Fix the network so that Customer_A and Customer_B are fully isolated from each other.

Verify: `ping 12.12.12.12 source 1.1.1.1` from R1 FAILS. `show ip route vrf Customer_A` on R2 does NOT contain any Customer_B routes.

Score: 3 Points

---

## Ticket 7 - DONE

The Route Reflector infrastructure is partially broken. R8 is not receiving vpnv4 routes from either RR. Other PEs (R2, R17, R18) are receiving routes normally from at least one RR.

Fix the network so that R8 receives vpnv4 routes via the Route Reflectors.

Verify: `show ip bgp vpnv4 all summary` on R8 shows non-zero PfxRcvd from at least one RR peer.

Score: 2 Points

---

## Ticket 8 - DONE

The TE tunnel (Tunnel0) on R2 with destination 8.8.8.8 is in DOWN state. The MPLS TE topology database on R2 appears populated with all routers and links.

Fix the network so that Tunnel0 comes UP.

Verify: `show mpls traffic-eng tunnels tunnel0` shows state UP with a valid path.

Score: 3 Points

---

## Ticket 9 - DONE

Tunnel0 on R2 is UP, but VPN traffic from R1 to R9 is NOT using the tunnel. Traffic is taking the normal IGP/LDP path instead of the TE tunnel path.

Fix the network so that VPN traffic for Customer_A rides the TE tunnel.

Verify: `show ip cef vrf Customer_A 9.9.9.9` on R2 shows Tunnel0 as the outgoing interface.

Score: 2 Points

---

## Ticket 10 - DONE

R2 can reach R8, and the VPN route for 9.9.9.9 exists in R2's VRF routing table. However, R1 cannot ping R9. The issue is on the PE-CE edge, not the MPLS core.

Fix the network so that R1 can ping R9.

Verify: `ping 9.9.9.9` from R1 succeeds.

Score: 2 Points

---

## Ticket 11 - DONE

R19 (Customer_D, behind R17) cannot reach its own VPN. `show ip bgp vpnv4 vrf Customer_D` on R17 shows 0 prefixes received from the RRs. R17's vpnv4 sessions to both RRs are Established.

Fix the network so that R19's routes are advertised to the RRs and reflected back.

Verify: `show ip bgp vpnv4 vrf Customer_D` on R17 shows locally originated routes. Other PEs with Customer_D VRF (if any) would receive them.

Score: 3 Points

---

## Ticket 12 - DONE

R2's TE tunnel (Tunnel0) to R8 is UP and on a valid path, but the tunnel is stuck using a longer path than optimal. The tunnel has `path-option 1 dynamic` configured. The TE topology shows all links with full bandwidth available.

Fix the network so that the TE tunnel uses the shortest path.

Verify: `show mpls traffic-eng tunnels tunnel0` shows the path with minimum hop count.

Score: 3 Points

---

## Ticket 13 - DONE

After a recent link flap event, VPN traffic from R1 to R9 black-holes for 30-60 seconds whenever a specific core link bounces, then recovers. The OSPF adjacency reforms within 5 seconds but MPLS traffic drops for much longer.

Fix the network so that MPLS traffic converges within the same timeframe as OSPF after a link flap.

Verify: Flap the affected link — VPN traffic recovers within 5 seconds (not 30-60).

Score: 4 Points

---

## Ticket 14 - DONE

R8 has vpnv4 routes for Customer_A prefix 1.1.1.1/32 in its BGP table with a valid next-hop (2.2.2.2). However, the route is NOT installed in the VRF routing table. `show ip bgp vpnv4 vrf Customer_A 1.1.1.1` shows "RIB-failure" or the route is valid/best but not in `show ip route vrf Customer_A`.

Fix the network so that the BGP vpnv4 route is installed in the VRF routing table.

Verify: `show ip route vrf Customer_A 1.1.1.1` shows the route with next-hop 2.2.2.2.

Score: 4 Points

---

## Ticket 15

R1 can ping R9, but traceroute from R1 to 9.9.9.9 shows packets reaching R8 and then disappearing. Intermittent packet loss (50%+) is occurring. The VPN label on R8 appears correct, the route is installed, but traffic is partially failing.

Fix the network so that traffic flows cleanly with 0% packet loss from R1 to R9.

Verify: `ping 9.9.9.9 source 1.1.1.1 repeat 100` from R1 shows 0% loss.

Score: 4 Points

---

## Ticket 16

R9 was previously receiving R1's routes as OSPF external routes (O E2) via OSPF PE-CE with R8. Today, R9's routing table shows NO routes to R1's loopbacks. R8's VRF has the routes in BGP. OSPF adjacency between R8 and R9 is FULL.

Fix the network so that R9 receives R1's routes via OSPF redistribution.

Verify: `show ip route` on R9 shows R1's loopbacks (1.1.1.1, 11.11.11.11) as OSPF external routes.

Score: 4 Points

---

## Ticket 17

A new TE tunnel (Tunnel1) has been configured on R2 with destination 8.8.8.8, bandwidth 90000, and an explicit path through R3→R4→R5→R8. The tunnel will not signal — RSVP PATH message is being rejected along the path. The existing Tunnel0 (bandwidth 1000, dynamic) is working fine.

Fix the network so that Tunnel1 comes UP with its requested bandwidth on the explicit path.

Verify: `show mpls traffic-eng tunnels tunnel1` shows state UP with bandwidth 90000.

Score: 4 Points

---

## Ticket 18

Customer_A VPN between R1 and R9 is COMPLETELY down. Customer_B VPN between R12 and R11 is WORKING. Customer_D (R19) is WORKING. The problem is isolated to Customer_A only. PE loopbacks (R2↔R8) are mutually reachable.

Fix the network so that Customer_A VPN is restored.

Verify: `ping 9.9.9.9` from R1 succeeds. `ping 1.1.1.1` from R9 succeeds.

Score: 5 Points

---

## Ticket 19

R1 can ping R9 (traffic works R1→R9), but R9 CANNOT ping R1 (traffic fails R9→R1). The VPN has an asymmetric failure — one direction works, the other doesn't.

Fix the network so that traffic works in BOTH directions.

Verify: `ping 9.9.9.9 source 1.1.1.1` from R1 succeeds AND `ping 1.1.1.1 source 9.9.9.9` from R9 succeeds.

Score: 5 Points

---

## Ticket 20

Multiple symptoms simultaneously:
- R17 and R18 have lost ALL vpnv4 routes from the RRs
- R2 and R8 still have vpnv4 routes and Customer_A/B VPNs are working
- R17 and R18 can ping R3 and R7 loopbacks (transport OK)
- BGP vpnv4 sessions on R17/R18 show Established but 0 prefixes received

Fix the network so that R17 and R18 receive vpnv4 routes from the RRs and their VPNs (Customer_D, Customer_E) are fully operational.

Verify: `show ip bgp vpnv4 all summary` on R17 shows non-zero PfxRcvd. R19 can ping R20 if they shared a VPN.

Score: 5 Points

---

## Scoring Summary

| Tickets | Difficulty | Points Each | Total |
|---|---|---|---|
| 1-3 | CCNP-SP (⭐⭐) | 2 | 6 |
| 4-6 | CCNP-SP (⭐⭐⭐) | 3 | 9 |
| 7-9 | CCNP-SP (⭐⭐⭐) | 2-3 | 7 |
| 10-12 | CCNP→CCIE (⭐⭐⭐) | 2-3 | 7 |
| 13-17 | CCIE-SP (⭐⭐⭐⭐) | 4 | 20 |
| 18-20 | CCIE-SP (⭐⭐⭐⭐⭐) | 5 | 15 |
| **Total** | | | **64 Points** |

**Passing:** 48/64 (75%)
**CCIE-ready:** 58/64 (90%)

---

## Active Session: Tickets 1, 2, 3 (Injected Aug 4, 2026)

**Status:** Faults injected — troubleshoot and fix

The base network is configured with:
- OSPF area 0 on all P/PE routers
- LDP + MPLS on all core interfaces
- RRs on R3 and R7
- L3VPN: Customer_A (R1↔R9 via eBGP), Customer_B (R12↔R11 via eBGP)
- Customer_D (R19 via eBGP on R17), Customer_E (R20 via eBGP on R18)
- TE Tunnel0 on R2 → R8 (dynamic, autoroute announce)

**Three faults have been injected.** Find them. Fix them.
