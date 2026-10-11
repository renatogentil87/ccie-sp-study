# Troubleshooting Lab 4: L3VPN Advanced — Inter-AS & Complex VPN Design — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Golden-state snapshot + Inter-AS extensions configured

---

## Lab Context

Your SP network now serves multi-homed customers and uses Inter-AS Option A between the "north domain" (R2, R3-R7, R8) and "south domain" (R17, R13-R16, R18). R6↔R13 acts as the Inter-AS boundary (back-to-back VRF). This lab tests complex VPN scenarios: Inter-AS, hub-spoke, extranet, shared services, and route-leaking.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF areas, AS numbers, or core MPLS configuration
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
| PE (north) | R2, R8 | 64512 |
| PE (south) | R17, R18 | 64512 |
| P (north) | R3, R4, R5, R7 | — |
| P (south) | R14, R15, R16 | — |
| Inter-AS boundary | R6 (north), R13 (south) | 64512 |
| RR | R3, R7 | 64512 |
| CE | R1 (65001), R9 (65001), R11 (65011), R12 (65012), R19 (65019), R20 (65020) |

**Inter-AS Design (Option A):**
- R6↔R13 link: Back-to-back VRF for Customer_A cross-domain traffic
- North Customer_A: R1 (on R2) ↔ south Customer_A: R19 (on R17) via inter-AS
- VRF `Customer_A_InterAS` on R6 and R13 with eBGP between them

**VPN Designs:**
- Customer_A: Full-mesh (R1↔R9 direct + R1↔R19 via Inter-AS)
- Customer_B: Hub-spoke (R2 hub, R8 spoke) — R12 is hub CE, R11 is spoke CE
- Customer_Shared: Shared-services VRF (R2) — accessible from Customer_A and Customer_B

---

## Ticket 1

The Inter-AS eBGP session between R6 and R13 (VRF `Customer_A_InterAS`) is not establishing. IP connectivity across the link (ping in VRF) works. Both sides have the VRF and eBGP neighbor configured.

Fix the network so that the Inter-AS eBGP session establishes.

Verify: `show ip bgp vrf Customer_A_InterAS summary` on R6 shows R13's IP as Established.

Score: 2 Points

---

## Ticket 2

The Inter-AS session is Established but R6 is NOT advertising any Customer_A routes to R13. R6's VRF routing table HAS Customer_A routes (from R2 via MP-BGP). The routes exist but aren't being passed across the inter-AS link.

Fix the network so that R6 advertises Customer_A routes to R13 via the inter-AS eBGP session.

Verify: `show ip bgp vrf Customer_A_InterAS` on R13 shows routes from R6's side (1.1.1.1/32, 9.9.9.9/32).

Score: 2 Points

---

## Ticket 3

R19 (Customer_A CE on R17) cannot reach R1 (Customer_A CE on R2) via the Inter-AS path. The route for 1.1.1.1 exists in R17's VRF table with next-hop R13, but MPLS label switching fails at the inter-AS boundary.

Fix the network so that end-to-end data-plane connectivity works across the Inter-AS link.

Verify: `ping 1.1.1.1 source 19.19.19.19` from R19 succeeds.

Score: 2 Points

---

## Ticket 4

Customer_B hub-spoke design: R11 (spoke CE on R8) can reach R12 (hub CE on R2), but R12 CANNOT reach R11. The hub-spoke RT design requires traffic from spoke→spoke to transit via the hub, but hub→spoke is also broken.

Fix the network so that the hub CE (R12) can reach spoke CE (R11).

Verify: `ping 11.11.11.11 source 12.12.12.12` from R12 succeeds. Route in R2's VRF shows spoke routes via hub.

Score: 3 Points

---

## Ticket 5

Customer_B spoke-to-spoke traffic via hub: R11 (spoke) can reach R12 (hub) but cannot reach another spoke CE if one existed. The issue is that R2 (hub PE) is NOT re-advertising spoke routes back to other spoke PEs. The hub PE receives spoke routes from RR but doesn't export them back.

Fix the network so that hub-spoke route advertisement works correctly.

Verify: Routes from R11 appear in R2's Customer_B VRF with RT indicating they came from spoke. R2 re-exports with hub RT.

Score: 3 Points

---

## Ticket 6

Shared-services VRF on R2: Customer_A CEs should be able to reach the shared-services resources (a loopback on R2 in the shared VRF), but Customer_A routes show no path to the shared network. The shared-services VRF exists but RT import/export between Customer_A and Shared is not working.

Fix the network so that Customer_A CEs can reach shared-services resources.

Verify: From R1, `ping <shared-services-IP>` succeeds. `show ip route vrf Customer_A` on R2 shows the shared-services prefix imported.

Score: 3 Points

---

## Ticket 7

Extranet between Customer_A and Customer_B: A specific prefix (12.12.12.12/32 from Customer_B) should be accessible from Customer_A, but the full Customer_B routing table should NOT leak. Currently, either nothing leaks (broken) or everything leaks (over-permissive).

Fix the network so that ONLY 12.12.12.12/32 is accessible from Customer_A, not the full Customer_B table.

Verify: From R1, `ping 12.12.12.12` succeeds. `show ip route vrf Customer_A` on R2 shows ONLY 12.12.12.12/32 from Customer_B, not 10.x.x.x transit links.

Score: 2 Points

---

## Ticket 8

Customer_A has a backup path via the Inter-AS link (R6↔R13) that should only be used when the primary path (direct R2↔R8 reflection) fails. Currently, R17 prefers the inter-AS path over the direct RR-reflected path due to incorrect LOCAL_PREF.

Fix the network so that the direct RR path is preferred and inter-AS is backup only.

Verify: `show ip bgp vpnv4 vrf Customer_A 1.1.1.1` on R17 shows the RR-reflected path as best (not the inter-AS path).

Score: 3 Points

---

## Ticket 9

SOO (Site of Origin) is preventing routes from being installed. R8 advertises Customer_A routes with SOO `64512:901` but R2 is also setting the same SOO on routes received from R1. This causes R2 to reject routes from R8 that originated at R9's site.

Fix the network so that SOO prevents routing loops without blocking legitimate routes.

Verify: `show ip route vrf Customer_A` on R2 shows R9's routes (9.9.9.9/32) received via R8/RR. SOO is correctly set per-site (not duplicated).

Score: 2 Points

---

## Ticket 10

The Inter-AS link (R6↔R13) is a single point of failure. A backup inter-AS link via R7↔R14 has been configured, but traffic never fails over to it when R6↔R13 goes down. The backup VRF interface exists but eBGP session won't establish.

Fix the network so that the backup inter-AS path via R7↔R14 becomes operational.

Verify: Shut R6↔R13 link → R19 can still reach R1 via the R7↔R14 backup path within 30 seconds.

Score: 2 Points

---

## Ticket 11

OSPF PE-CE (Customer_A, R8↔R9): R9 is receiving routes with OSPF cost 1 for ALL remote prefixes. The routes should carry the original OSPF cost from the originating site (varying costs). OSPF domain-id or metric-type handling is incorrect.

Fix the network so that redistributed routes on R9 carry meaningful metrics reflecting distance.

Verify: `show ip route` on R9 shows different OSPF costs for different prefixes (not all cost 1).

Score: 3 Points

---

## Ticket 12

OSPF PE-CE (R8↔R9): A routing loop exists — routes originated by R9 (9.9.9.9/32) are being redistributed back to R9 by R8 as external OSPF routes. R9 sees its own loopback as an O E2 route via R8.

Fix the network so that OSPF loop prevention (down-bit or domain-tag) prevents re-advertisement.

Verify: `show ip route 9.9.9.9` on R9 shows it as a connected route (Loopback0), NOT as O E2 via R8.

Score: 3 Points

---

## Ticket 13

Import-map on R8 for Customer_A is blocking specific prefixes. R8 has an `import-map` configured under the VRF that should permit all Customer_A routes but a deny statement is matching 1.1.1.1/32 by mistake.

Fix the network so that all Customer_A routes are imported into R8's VRF.

Verify: `show ip route vrf Customer_A 1.1.1.1` on R8 shows the route present. `show ip bgp vpnv4 vrf Customer_A` shows all prefixes as valid.

Score: 4 Points

---

## Ticket 14

Multi-homed CE: R1 is connected to both R2 (primary) and R17 (backup via south domain). R1 advertises its loopback to both PEs. However, traffic returning to R1 always takes the R17 path (backup) instead of the R2 path (primary) due to BGP path selection at the RR.

Fix the network so that return traffic to R1 prefers the R2 path (shorter IGP distance).

Verify: Traceroute from R9 to 1.1.1.1 shows path via R8→core→R2→R1 (not via south/R17).

Score: 4 Points

---

## Ticket 15

VRF route limit has been hit on R2. Customer_A VRF has `maximum routes 50 warning-only` but someone changed it to `maximum routes 5`. New routes from R1 are being rejected. The VRF table is frozen at 5 routes.

Fix the network so that R2's Customer_A VRF accepts all routes without limit issues.

Verify: `show ip route vrf Customer_A` on R2 shows all expected routes. No "maximum route limit" errors in log.

Score: 4 Points

---

## Ticket 16

PE-CE eBGP with AS-override: R8 uses `as-override` for Customer_A (R9 is in AS 65001, same as R1). However, as-override is creating a loop — R8 is overriding its OWN AS (64512) in the path, causing the route to be accepted when it shouldn't be.

Fix the network so that as-override works correctly (only overrides customer's AS, not SP's).

Verify: `show ip bgp vrf Customer_A 1.1.1.1` on R8 shows AS-PATH with 64512 replaced by 64512 only once (no infinite loop). R9 accepts R1's routes.

Score: 4 Points

---

## Ticket 17

VPNv4 route-target rewrite at the Inter-AS boundary (R6): Routes crossing from north to south should have RT `64512:100` rewritten to `64512:1000` for policy reasons. The rewrite is configured but routes arrive at R17 with the ORIGINAL RT (64512:100) and R17's VRF doesn't import them.

Fix the network so that RT rewrite works at the inter-AS boundary and R17 imports the routes.

Verify: `show ip bgp vpnv4 vrf Customer_A` on R17 shows routes with correct RT. `show ip route vrf Customer_A` has the inter-AS routes.

Score: 4 Points

---

## Ticket 18

Complete Inter-AS failure: R19 cannot reach ANY Customer_A CE on the north side (R1, R9). The inter-AS eBGP sessions are DOWN on BOTH links (primary R6↔R13 and backup R7↔R14). North-side Customer_A is fully working (R1↔R9 fine).

Fix the network so that Inter-AS connectivity is fully restored.

Verify: `ping 1.1.1.1 source 19.19.19.19` from R19 succeeds. Both inter-AS eBGP sessions are Established.

Score: 5 Points

---

## Ticket 19

Hub-spoke + shared-services interaction: Customer_B spoke CEs can reach the hub but cannot reach shared-services resources. The shared-services VRF imports from both Customer_A and Customer_B, but Customer_B's spoke RT is not being imported correctly due to RT-rewrite at the hub PE.

Fix the network so that Customer_B spoke CEs can access shared services via the hub.

Verify: From R11 (spoke), ping shared-services IP succeeds. Route path is R11→R8→R2(hub)→shared.

Score: 5 Points

---

## Ticket 20

Full VPN control-plane reconstruction: Customer_A across both domains (north + south) is COMPLETELY down. Multiple faults exist simultaneously across the Inter-AS boundary, RR reflection, and PE-CE peering. Customer_B and D/E are working.

Fix the network so that Customer_A is fully operational across both domains.

Verify: R1↔R9 ping works. R1↔R19 ping works (via inter-AS). R19↔R9 ping works.

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

**Base state:** Golden-state + Inter-AS VRF on R6/R13, hub-spoke RT on Customer_B, shared-services VRF on R2

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R6 or R13 | Wrong remote-AS in VRF eBGP neighbor |
| 2 | R6 | Missing `redistribute bgp` or network statement in inter-AS VRF |
| 3 | R6/R13 | `no mpls ip` on inter-AS link (or label not allocated) |
| 4 | R2 | Hub RT export missing spoke-import RT |
| 5 | R2 | Missing `route-reflector` or RT design: hub not re-exporting |
| 6 | R2 | Shared-services VRF: RT import not including Customer_A RT |
| 7 | R2 | Over-permissive or missing `import-map` for selective extranet |
| 8 | R17 | LOCAL_PREF set higher on inter-AS path |
| 9 | R2 | SOO set to same value as R8's SOO |
| 10 | R7/R14 | Backup VRF interface shut or wrong IP in eBGP neighbor |
| 11 | R8 | OSPF redistribute with `metric 1` or missing `metric-type` |
| 12 | R8 | Missing `domain-id` or DN-bit not set |
| 13 | R8 | import-map denying 1.1.1.1/32 |
| 14 | R17 | LOCAL_PREF 200 on R1's routes (higher than R2's path) |
| 15 | R2 | `maximum routes 5` in VRF |
| 16 | R8 | as-override interacting with allowas-in creating loop |
| 17 | R6 | RT rewrite route-map not applied or wrong match |
| 18 | R6+R7 | VRF interface shut on both inter-AS links |
| 19 | R2 | Hub PE not importing spoke RT into shared-services VRF |
| 20 | Multiple | Inter-AS down + RR not reflecting + PE-CE broken |
