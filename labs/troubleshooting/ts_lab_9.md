# Troubleshooting Lab 9: Multicast VPN (mVPN) — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Golden-state + mVPN overlay configured (PIM-SM core, mVPN Profile 0/1)

---

## Lab Context

Your SP network now carries multicast traffic for VPN customers. The core runs PIM-SM/SSM for MDT transport. Each VPN customer has a Default MDT (for low-bandwidth multicast) and optionally a Data MDT (for high-bandwidth streams). This lab tests mVPN troubleshooting: PIM adjacency, MDT formation, IGMP, RPF failures, and data-plane multicast forwarding.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF, LDP, BGP, or unicast VPN configurations
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers | Multicast Role |
|---|---|---|
| PE (source-side) | R2, R17 | First-hop router for multicast sources |
| PE (receiver-side) | R8, R18 | Last-hop router for multicast receivers |
| P (core) | R3, R4, R5, R6, R7 | PIM-SM transit |
| P (south) | R13, R14, R15, R16 | PIM-SM transit |
| RR | R3, R7 | BGP mdt SAFI |
| RP | R5 (core PIM-SM RP) | — |
| CE Source | R1 (sends 239.1.1.1), R19 (sends 239.2.2.2) |
| CE Receiver | R9 (receives 239.1.1.1), R11 (receives 239.1.1.1), R20 (receives 239.2.2.2) |

**mVPN Services:**
- VPN_Mcast_A (Customer_A): Source R1 → Receivers R9, R11
  - Default MDT: 232.1.1.1 (PIM-SSM in core)
  - Data MDT: 232.1.1.100 (triggered at >100 kbps)
  - RP (VRF): R2 loopback (static RP in VRF)
- VPN_Mcast_B (Customer_D): Source R19 → Receiver R20
  - Default MDT: 232.2.2.2 (PIM-SSM in core)
  - RP (VRF): R17 loopback (static RP in VRF)

**Core Multicast:**
- PIM-SM with RP at R5 (5.5.5.5) for default MDT groups
- PIM-SSM range: 232.0.0.0/8
- `ip multicast-routing` on all P/PE routers
- All core interfaces: `ip pim sparse-mode`

**BGP:** mdt SAFI for MDT auto-discovery between PEs

---

## Ticket 1

PIM adjacency between R3 and R7 is not forming. Both interfaces have `ip pim sparse-mode` configured. The interface is UP/UP and OSPF adjacency is FULL. IGMP/PIM hello timers are default.

Fix the network so that the PIM adjacency establishes between R3 and R7.

Verify: `show ip pim neighbor` on R3 shows R7 as a PIM neighbor (and vice versa).

Score: 2 Points

---

## Ticket 2

R5 is the core RP (for MDT groups 232.x.x.x) but no PE routers have R5 mapped as RP. `show ip pim rp mapping` on R2 shows no RP for the 232.0.0.0/8 range. R5 has `ip pim rp-address 5.5.5.5` configured locally.

Fix the network so that all routers know R5 is the RP for MDT groups.

Verify: `show ip pim rp mapping` on R2 and R8 shows 5.5.5.5 as RP for 232.0.0.0/8.

Score: 2 Points

---

## Ticket 3

`ip multicast-routing` is missing on R4 (P router). PIM adjacencies are UP on both sides of R4, but multicast traffic transiting R4 is being dropped. Unicast traffic through R4 works fine.

Fix the network so that R4 forwards multicast traffic in transit.

Verify: `show ip mroute` on R4 shows entries for the MDT group. Multicast traffic transits R4 without drops.

Score: 2 Points

---

## Ticket 4

VPN_Mcast_A: Default MDT (232.1.1.1) is not forming between R2 and R8. R2 has the MDT configured (`mdt default 232.1.1.1`) under the VRF but R8 does NOT. The BGP mdt SAFI session is Established between PEs.

Fix the network so that both R2 and R8 join the default MDT for VPN_Mcast_A.

Verify: `show ip mroute 232.1.1.1` on core routers shows the MDT tree built. `show ip pim mdt` on R2 and R8 shows MDT neighbors.

Score: 3 Points

---

## Ticket 5

MDT auto-discovery via BGP: R8 has the MDT configured but is not discovering R2 as an MDT neighbor. The BGP mdt SAFI session between R8 and the RRs shows 0 prefixes received. The MDT Type-1 routes are not being exchanged.

Fix the network so that BGP mdt auto-discovery works between PEs.

Verify: `show ip bgp ipv4 mdt all` on R8 shows R2's MDT advertisement. `show ip pim mdt` shows R2 as MDT neighbor.

Score: 3 Points

---

## Ticket 6

RPF (Reverse Path Forwarding) failure: Multicast traffic from R1 (source 1.1.1.1, group 239.1.1.1) arrives at R2 but R2 drops it due to RPF check failure. The traffic arrives on the CE-facing interface but RPF expects it on a different interface.

Fix the network so that RPF check passes for the multicast source in the VRF.

Verify: `show ip mroute vrf Customer_A 239.1.1.1` on R2 shows incoming interface as the CE-facing port. No RPF failures in `show ip mroute count`.

Score: 3 Points

---

## Ticket 7

IGMP join from R9 (receiver): R9 sends IGMP join for 239.1.1.1 but R8 is not processing it. `show ip igmp groups vrf Customer_A` on R8 shows no groups. The VRF interface toward R9 exists and is UP.

Fix the network so that R8 processes IGMP joins from R9 and joins the multicast tree.

Verify: `show ip igmp groups vrf Customer_A` on R8 shows 239.1.1.1 with R9's interface. `show ip mroute vrf Customer_A` shows (*, 239.1.1.1) with OIL toward R9.

Score: 2 Points

---

## Ticket 8

Core PIM-SSM: The MDT group (232.1.1.1) requires PIM-SSM (source-specific join) but a core router (R6) is trying to do (*, G) join to the RP instead of (S, G) join. This fails because 232.x.x.x is in the SSM range and doesn't use RPs.

Fix the network so that R6 uses SSM (S, G) joins for the MDT group.

Verify: `show ip mroute 232.1.1.1` on R6 shows (S, G) entry (not (*, G)). Source IP is R2's loopback.

Score: 3 Points

---

## Ticket 9

Data MDT not triggering: VPN_Mcast_A source (R1) is sending 500 kbps to 239.1.1.1. The Data MDT (232.1.1.100) should activate at >100 kbps but all traffic stays on the Default MDT. `show ip pim mdt` shows no data MDT created.

Fix the network so that the Data MDT triggers when traffic exceeds the threshold.

Verify: `show ip pim vrf Customer_A mdt send` on R2 shows Data MDT 232.1.1.100 active for the high-bandwidth stream.

Score: 2 Points

---

## Ticket 10

PIM VRF multicast routing: `ip multicast-routing vrf Customer_A` is configured on R2 but missing on R8. Unicast VPN traffic works, but multicast traffic in the VRF is not being routed by R8 toward receivers.

Fix the network so that VRF multicast routing is enabled on all PEs serving multicast customers.

Verify: `show ip mroute vrf Customer_A` on R8 shows entries. Multicast traffic reaches R9.

Score: 2 Points

---

## Ticket 11

Static RP in VRF: VPN_Mcast_A uses R2's loopback as the VRF RP (static). R8 needs to know this RP to build the shared tree for new receivers. But R8's VRF has no RP configured — it doesn't know where to send (*, G) PIM joins.

Fix the network so that R8 knows the VRF RP and can build shared trees.

Verify: `show ip pim vrf Customer_A rp mapping` on R8 shows RP as R2's VRF loopback. New receiver joins go to the RP.

Score: 3 Points

---

## Ticket 12

Multicast traffic exits the MDT tunnel but is dropped at R8's VRF. R8 receives the encapsulated multicast on the MDT tunnel interface, decapsulates it, but the OIL (Outgoing Interface List) for the VRF mroute is empty. No receiver has joined.

Fix the network so that the OIL is populated and traffic is forwarded to the receiver CE.

Verify: `show ip mroute vrf Customer_A 239.1.1.1` on R8 shows non-empty OIL including the interface toward R9.

Score: 3 Points

---

## Ticket 13

PIM Assert conflict: R2 and R17 are both connected to the same multicast source (R1 is dual-homed). Both PEs are sending the same multicast stream into the MDT, causing duplicate traffic at receivers. PIM Assert should elect a single forwarder.

Fix the network so that only one PE forwards the multicast stream (no duplicates at receivers).

Verify: `show ip mroute vrf Customer_A` shows Assert winner on the shared segment. R9 receives exactly one copy of each multicast packet.

Score: 4 Points

---

## Ticket 14

MDT tunnel interface: The GRE-based MDT tunnel interface on R2 (`interface Tunnel0` type mdt) is DOWN. The MDT group (232.1.1.1) is not being joined in the core. All other multicast and unicast functions work.

Fix the network so that the MDT tunnel interface comes UP and R2 joins the core MDT group.

Verify: `show interface Tunnel0` (MDT type) shows UP/UP. `show ip mroute 232.1.1.1` in global shows R2 as a receiver.

Score: 4 Points

---

## Ticket 15

Extranet multicast: Customer_A's multicast stream (239.1.1.1) should also be accessible to Customer_B receivers (R11 on R8). Multicast extranet is configured but R11 cannot receive the stream. Unicast extranet (route leaking) works.

Fix the network so that Customer_B receivers can join Customer_A's multicast group via extranet.

Verify: R11 sends IGMP join for 239.1.1.1 → R8 adds R11's interface to the OIL for the Customer_A mroute. R11 receives the stream.

Score: 4 Points

---

## Ticket 16

mLDP (Label-switched mVPN): VPN_Mcast_B uses mLDP instead of GRE for core transport. The mLDP tree between R17 and R18 is not forming. The mLDP FEC is configured but no label mappings are exchanged.

Fix the network so that the mLDP-based MDT forms between R17 and R18.

Verify: `show mpls mldp database` on R17 shows the mLDP FEC with downstream labels. Multicast traffic from R19 reaches R20.

Score: 4 Points

---

## Ticket 17

PIM DR election: On the segment between R8 and R9 (receiver-side), R8 should be the PIM DR (Designated Router) to handle IGMP. But R9 has a higher IP and wins DR election. This causes R8 to not process R9's IGMP reports.

Fix the network so that R8 is the PIM DR on the receiver-facing segment.

Verify: `show ip pim interface vrf Customer_A` on R8 shows DR role. R8 processes IGMP reports from R9.

Score: 4 Points

---

## Ticket 18

Complete mVPN failure for Customer_A: No multicast traffic flows from R1 to R9. Unicast VPN works. Multiple issues: PIM adjacency missing in core, MDT not configured on one PE, and IGMP disabled on receiver interface.

Fix the network so that multicast traffic flows end-to-end for Customer_A.

Verify: R1 sends to 239.1.1.1 → R9 receives the stream. `show ip mroute vrf Customer_A` on R2 and R8 shows active entries with non-zero packet counts.

Score: 5 Points

---

## Ticket 19

Multicast traffic blackhole during unicast reconvergence: When an OSPF link flaps in the core, multicast traffic (MDT) is lost for 60+ seconds even though unicast recovers in 5 seconds. The RPF check for the MDT source (R2's loopback) fails during convergence.

Fix the network so that multicast RPF reconverges at the same rate as unicast (within 5 seconds).

Verify: Flap a core link — multicast stream resumes within 5-10 seconds. No 60-second outage.

Score: 5 Points

---

## Ticket 20

Full multicast control-plane failure: Neither VPN_Mcast_A nor VPN_Mcast_B works. Core PIM is broken (no neighbors anywhere). VRF multicast routing is disabled on PEs. BGP mdt SAFI shows 0 routes. Multiple simultaneous faults.

Fix the network so that both mVPN services are fully operational.

Verify: R9 receives 239.1.1.1 from R1 (VPN_Mcast_A). R20 receives 239.2.2.2 from R19 (VPN_Mcast_B). MDT tunnels UP on all PEs.

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

**Base state:** Golden-state + mVPN configured (PIM-SM core, MDT, VRF multicast, BGP mdt SAFI)

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R3 or R7 | ACL blocking PIM hellos (224.0.0.13) or `no ip pim` on interface |
| 2 | R2/R8 | Missing `ip pim rp-address 5.5.5.5` on PEs (or wrong group range) |
| 3 | R4 | `no ip multicast-routing` |
| 4 | R8 | Missing `mdt default 232.1.1.1` under VRF |
| 5 | R3/R7 | `no neighbor <PE> activate` under mdt address-family |
| 6 | R2 | `ip mroute vrf Customer_A` RPF wrong or interface config |
| 7 | R8 | `no ip igmp` or `no ip pim sparse-mode` on VRF CE-facing interface |
| 8 | R6 | `no ip pim ssm default` (treats 232.x as ASM) |
| 9 | R2 | `mdt data` threshold set to 999999 kbps (never triggers) |
| 10 | R8 | Missing `ip multicast-routing vrf Customer_A` |
| 11 | R8 | No `ip pim rp-address` in VRF |
| 12 | R8 | No IGMP join received or interface not in OIL |
| 13 | R2+R17 | Both forwarding (no PIM Assert or wrong metric) |
| 14 | R2 | MDT tunnel source wrong or `shutdown` |
| 15 | R8 | Missing multicast extranet import config |
| 16 | R17/R18 | mLDP not enabled or wrong FEC |
| 17 | R9 | Higher IP wins DR; fix with `ip pim dr-priority` on R8 |
| 18 | Multiple | PIM adj missing + MDT not configured + IGMP disabled |
| 19 | Core | Missing `ip pim` on a redundant path (forces single RPF path) |
| 20 | Multiple | `no ip multicast-routing` + `no ip pim` + mdt SAFI deactivated |
