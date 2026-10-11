# Troubleshooting Lab 3: MP-BGP & Route Reflectors — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology as ts_lab_2
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Golden-state snapshot (OSPF + LDP + L3VPN + RRs working)

---

## Lab Context

This lab focuses exclusively on BGP control plane issues: iBGP peering, Route Reflector design, vpnv4/vpnv6 address-family activation, path selection, communities, and scalability features. The MPLS transport (OSPF + LDP) is fully working — all faults are in the BGP layer.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF, LDP, or MPLS interface configuration
- Do NOT change BGP AS numbers
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
| RR | R3 (Cluster-A), R7 (Cluster-B) | 64512 |
| CE | R1 (65001), R9 (65001), R11 (65011), R12 (65012), R19 (65019), R20 (65020) |

**RR Design:**
- R3 (Cluster-A): Reflects to R2, R8, R17, R18
- R7 (Cluster-B): Reflects to R2, R8, R17, R18
- Both RRs peer with each other (inter-cluster)
- All PEs peer with BOTH RRs for redundancy

**Address Families:** vpnv4 unicast on all PE↔RR sessions
**VPNs:** Customer_A (R1↔R9), Customer_B (R12↔R11), Customer_D (R19), Customer_E (R20)

---

## Ticket 1

R17 (PE, south) has no iBGP session to either Route Reflector. R17 can ping both R3 (3.3.3.3) and R7 (7.7.7.7) loopbacks successfully. Other PEs (R2, R8, R18) have working sessions.

Fix the network so that R17 establishes iBGP sessions to both RRs.

Verify: `show ip bgp vpnv4 all summary` on R17 shows both R3 and R7 sessions in Established state.

Score: 2 Points

---

## Ticket 2

R2's iBGP sessions to R3 and R7 are Established, but R2 is receiving 0 vpnv4 prefixes from both RRs. Other PEs (R8, R17, R18) are receiving prefixes normally.

Fix the network so that R2 receives vpnv4 prefixes from the RRs.

Verify: `show ip bgp vpnv4 all summary` on R2 shows non-zero PfxRcvd from at least one RR.

Score: 2 Points

---

## Ticket 3

The inter-cluster iBGP session between R3 and R7 is stuck in Active state. Both routers can ping each other's loopbacks. Their BGP sessions to all PEs are working fine.

Fix the network so that the R3↔R7 iBGP session establishes.

Verify: `show ip bgp neighbors 7.7.7.7` on R3 shows state Established. `show ip bgp vpnv4 all summary` confirms prefix exchange.

Score: 2 Points

---

## Ticket 4

R8 has vpnv4 routes for Customer_A (1.1.1.1/32) but the ORIGINATOR_ID shows 8.8.8.8 — which is R8 itself. R8 is rejecting its own reflected route. The route appears in `show ip bgp vpnv4 all` but is marked as "not valid, reason: originator is us."

Fix the network so that R8 accepts the reflected vpnv4 route for Customer_A.

Verify: `show ip bgp vpnv4 vrf Customer_A 1.1.1.1` on R8 shows a valid route with ORIGINATOR_ID 2.2.2.2 (R2).

Score: 3 Points

---

## Ticket 5

R3 (RR) is reflecting vpnv4 routes to R2 and R8, but NOT to R17 and R18. R3's BGP sessions to R17 and R18 are Established. The issue is R17/R18 are not configured as RR clients on R3.

Fix the network so that R3 reflects vpnv4 routes to ALL PEs.

Verify: `show ip bgp vpnv4 all summary` on R17 shows non-zero PfxRcvd from R3. `show ip bgp vpnv4 all neighbors 17.17.17.17` on R3 shows "Route-Reflector Client."

Score: 3 Points

---

## Ticket 6

Customer_D routes (from R19 via R17) are visible on R17 but are NOT being reflected by either RR to other PEs. R2, R8, R18 all show 0 Customer_D routes. R17's vpnv4 session is Established and R17 IS advertising the routes outbound.

Fix the network so that Customer_D routes are reflected to all PEs.

Verify: `show ip bgp vpnv4 all` on R2 shows Customer_D prefixes (RD 64512:400). R8 can see R19's loopback in vpnv4.

Score: 3 Points

---

## Ticket 7

BGP best-path selection is wrong for Customer_A. R8 is selecting the path via R3 (ORIGINATOR_ID 2.2.2.2) but ignoring a shorter path reflected via R7. The route via R7 has a lower IGP cost to the next-hop but is not being preferred.

Fix the network so that R8 selects the best path based on lowest IGP cost to next-hop.

Verify: `show ip bgp vpnv4 vrf Customer_A 1.1.1.1` on R8 shows the best path with lowest next-hop IGP metric.

Score: 2 Points

---

## Ticket 8

R18 (PE) is advertising Customer_E routes with the wrong next-hop. The vpnv4 routes reflected by the RRs show next-hop as one of R18's physical interface IPs instead of the loopback. Remote PEs cannot resolve this next-hop via LDP.

Fix the network so that R18's vpnv4 routes have the correct next-hop (18.18.18.18).

Verify: `show ip bgp vpnv4 all 20.20.20.20` on R2 shows next-hop 18.18.18.18. Label switching to R18 works.

Score: 3 Points

---

## Ticket 9

R7 (RR, Cluster-B) is configured with `bgp cluster-id 7` while R3 (RR, Cluster-A) uses `bgp cluster-id 3`. However, someone changed R7's cluster-id to match R3's (`bgp cluster-id 3`). Now routes reflected by R3 are being discarded by R7 due to CLUSTER_LIST loop detection.

Fix the network so that both RRs operate as separate clusters and routes flow freely between them.

Verify: `show ip bgp vpnv4 all` on any PE shows routes with CLUSTER_LIST containing both cluster IDs. Full vpnv4 reachability exists.

Score: 2 Points

---

## Ticket 10

A PE-CE eBGP session between R2 and R1 (Customer_A) is Established but R1 is not receiving any routes FROM R2 (no default, no VPN routes). R2's VRF table has routes but they're not being advertised to R1.

Fix the network so that R1 receives routes from R2 via eBGP.

Verify: `show ip bgp` on R1 shows routes received from R2 (192.168.12.2). R1 can reach 9.9.9.9.

Score: 2 Points

---

## Ticket 11

R2 is receiving R1's routes via eBGP in the VRF, but these routes are NOT being redistributed into the vpnv4 BGP table. `show ip bgp vpnv4 vrf Customer_A` on R2 shows 0 locally originated routes.

Fix the network so that R2 advertises Customer_A VRF routes into vpnv4 BGP.

Verify: `show ip bgp vpnv4 vrf Customer_A` on R2 shows 1.1.1.1/32 as locally originated. RRs reflect it to R8.

Score: 3 Points

---

## Ticket 12

R8's PE-CE BGP session with R9 shows Established, but the AS-PATH for routes received from R9 contains AS 64512 (the SP's own ASN). R8 is rejecting these routes due to iBGP loop detection (own AS in path).

Fix the network so that R8 accepts R9's routes despite the AS-PATH containing 64512.

Verify: `show ip bgp vpnv4 vrf Customer_A 9.9.9.9` on R8 shows the route as valid/best. R1 can ping R9.

Score: 3 Points

---

## Ticket 13

BGP communities are not being propagated through the RRs. R2 attaches community `64512:100` to Customer_A routes, but when R8 receives them via the RR, the community is stripped. Route-maps on R8 that match this community fail.

Fix the network so that BGP communities survive RR reflection.

Verify: `show ip bgp vpnv4 vrf Customer_A 1.1.1.1 community` on R8 shows `64512:100` attached.

Score: 4 Points

---

## Ticket 14

R17 is receiving vpnv4 routes from both RRs but the NEXT_HOP (2.2.2.2 for Customer_A routes) is not resolvable via LDP. `show mpls ldp bindings 2.2.2.2 32` on R17 returns empty. R17 CAN ping 2.2.2.2 via IP. The VPN route shows "inaccessible" in the BGP table.

Fix the network so that R17 can resolve the vpnv4 next-hop via LDP labels.

Verify: `show mpls ldp bindings 2.2.2.2 32` on R17 shows a remote binding. `show ip bgp vpnv4 vrf Customer_D` shows routes as valid with next-hop resolved.

Score: 4 Points

---

## Ticket 15

BGP max-prefix limit has been hit on R2's vpnv4 session with R3. The session shows "Idle (PfxCt)" state. R2 configured `neighbor 3.3.3.3 maximum-prefix 10` but the RR is reflecting more than 10 prefixes.

Fix the network so that R2's session with R3 stays Established with all prefixes accepted.

Verify: `show ip bgp vpnv4 all summary` on R2 shows R3 session Established with correct prefix count. No prefix limit errors in log.

Score: 4 Points

---

## Ticket 16

R18 has a vpnv4 session Established with both RRs and is receiving Customer_E routes. However, when R18 originates Customer_E routes (from R20), only R7 receives and reflects them — R3 does NOT. The R3↔R18 session is Established and shows 0 PfxRcvd.

Fix the network so that both RRs receive R18's vpnv4 routes.

Verify: `show ip bgp vpnv4 all neighbors 18.18.18.18` on R3 shows non-zero PfxRcvd. Both RRs have Customer_E routes.

Score: 4 Points

---

## Ticket 17

Route Target Constraint (RT-Constraint) has been enabled on the RRs. R8 configured a new VRF (Customer_F) but is NOT receiving any routes for it from the RRs. R2 is originating Customer_F routes and the RRs have them, but filtering prevents delivery to R8.

Fix the network so that R8 receives Customer_F vpnv4 routes via RT-Constraint filtering.

Verify: `show ip bgp vpnv4 vrf Customer_F` on R8 shows received routes. RT-Constraint advertisement from R8 includes the new RT.

Score: 4 Points

---

## Ticket 18

Complete BGP control plane failure for Customer_A: R1 cannot reach R9, R9 cannot reach R1. Customer_B, D, E are ALL working. The MPLS transport is fine (R2 can ping R8). Multiple BGP-layer faults are contributing simultaneously.

Fix the network so that Customer_A VPN is fully operational.

Verify: `ping 9.9.9.9 source 1.1.1.1` from R1 succeeds. `ping 1.1.1.1 source 9.9.9.9` from R9 succeeds.

Score: 5 Points

---

## Ticket 19

Route Reflector failover is broken. When R3 goes down (simulated: clear all BGP sessions on R3), PEs should failover to R7 seamlessly. Instead, R2 and R8 lose ALL vpnv4 routes for 5+ minutes before recovering. R17/R18 failover correctly.

Fix the network so that RR failover is seamless (routes available via R7 within seconds of R3 failure).

Verify: Clear R3's BGP sessions → R2 and R8 retain vpnv4 routes (via R7) within 30 seconds. No VPN outage.

Score: 5 Points

---

## Ticket 20

All four PEs (R2, R8, R17, R18) show BGP sessions Established with both RRs and are receiving vpnv4 prefixes. However, NO VPN traffic actually works — all ping tests between CEs fail. The label stack shows correct VPN labels, but the transport label (LDP) for the BGP next-hop resolves to the wrong outgoing interface on the RRs.

Fix the network so that all VPN traffic flows correctly end-to-end.

Verify: All CE-to-CE pings succeed. `show mpls forwarding-table` on P routers shows correct label paths to all PE loopbacks.

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

**Base state:** Golden-state snapshot (OSPF + LDP + full L3VPN + RRs working)

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R17 | `no neighbor 3.3.3.3 update-source Loopback0` / `no neighbor 7.7.7.7 update-source Loopback0` |
| 2 | R3+R7 | `no neighbor 2.2.2.2 activate` under vpnv4 AF on both RRs |
| 3 | R3 or R7 | Wrong `neighbor 7.7.7.7 remote-as` or missing `update-source` |
| 4 | R3 | `neighbor 8.8.8.8 route-reflector-client` missing → R8 sees its own originator |
| 5 | R3 | `no neighbor 17.17.17.17 route-reflector-client` + `no neighbor 18.18.18.18 route-reflector-client` |
| 6 | R17 | Missing `redistribute connected` or `network` under vpnv4 vrf Customer_D |
| 7 | R8 | `bgp bestpath compare-routerid` forcing wrong selection |
| 8 | R18 | `no neighbor <RR> next-hop-self` missing → physical IP as nexthop |
| 9 | R7 | `bgp cluster-id 3` (matches R3, causing CLUSTER_LIST loop) |
| 10 | R2 | Missing `neighbor 192.168.12.1 default-originate` or outbound policy denying |
| 11 | R2 | Missing `redistribute connected` or `network` in VRF AF |
| 12 | R9 | CE prepending `64512` in AS-PATH; fix with `as-override` on R8 |
| 13 | R3+R7 | `no neighbor <PE> send-community both` |
| 14 | R13 | `no mpls ip` on link toward R17 (breaks LDP label path) |
| 15 | R2 | `neighbor 3.3.3.3 maximum-prefix 10` |
| 16 | R18 | `neighbor 3.3.3.3 next-hop-unchanged` or send-community missing |
| 17 | R8 | Missing VRF + RT config; RR has RT-Constraint enabled filtering it |
| 18 | Multiple | PE-CE deactivated + RR not reflecting + password mismatch |
| 19 | R2+R8 | Only peer with R3, not R7 (single RR dependency) |
| 20 | R3 or R7 | `mpls ldp router-id` changed → LDP labels stale, BGP nexthop unresolvable via MPLS |
