# E21: SRv6 (Segment Routing over IPv6)

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 (IS-IS on all 3 SPs), E04 (SR-MPLS on Garnet)
**Base:** IS-IS L2 on backbone, full loopback reachability in both ASes.

**End Goal:** SRv6 on Gold (AS 65300) — locators, SIDs, SRv6-TE policies, L3VPN over SRv6. No MPLS labels — pure IPv6 data plane.

---

### Task 1: Enable IPv6 on Garnet core
1. IPv6 addresses on all Gold core (G-R4, G-R5, G-R3, G-R1, G-R2) interfaces (use ULA fc00:2:X::Y/64 or similar).
2. IPv6 loopbacks on G-R1 and Gar-R2, Gar-R3-Gar-R5, Gar-R7, Gar-R6 (e.g., Gar-R1 = fc00::11/128).
3. IS-IS: `address-family ipv6 unicast` on all core interfaces. Verify: `show isis ipv6 route` — all IPv6 loopbacks reachable.

### Task 2: Configure SRv6 locator
1. On each Gold router:
```
segment-routing
 srv6
  locators
   locator MAIN
    prefix fc00:0:X::/48    (X = unique per router, e.g., Gar-R1 = fc00:0:11::/48)
```
2. Under IS-IS: `address-family ipv6 unicast / segment-routing srv6 / locator MAIN`.
3. Verify: `show segment-routing srv6 locator` — locator active.
4. Verify: `show segment-routing srv6 sid` — SIDs allocated (End, End.X, End.DT4, etc.).

### Task 3: Understand SRv6 SID types
1. **End SID** — node segment (like prefix-SID in SR-MPLS). Reach this router.
2. **End.X SID** — adjacency segment (like adj-SID). Force out specific link.
3. **End.DT4 SID** — decapsulate and lookup in IPv4 VRF table (L3VPN).
4. **End.DT6 SID** — decapsulate and lookup in IPv6 VRF table.
5. Verify each on G-R1: `show segment-routing srv6 sid` — identify which SID does what.

### Task 4: SRv6-TE policy
1. On G-R1 and Gar-R2 with explicit SID list:
```
segment-routing
 traffic-eng
  segment-list VIA-Gar-R4
   index 10 srv6 sid fc00:0:14::    (Gar-R4's End SID — get to Gar-R4)
   index 20 srv6 sid fc00:0:12::    (Gar-R2's End SID — get to Gar-R2)
  policy G-R1 and Gar-R2-SRV6
   color 300 end-point ipv6 fc00::12
   candidate-paths
    preference 100
     explicit segment-list VIA-Gar-R4
```
2. Verify: `show segment-routing traffic-eng policy` — UP with SRv6 SID list.
3. Compare with SR-MPLS TE (E04): same concept, IPv6 addresses instead of MPLS labels.

### Task 5: L3VPN over SRv6
1. On G-R1 and Gar-R2: VRF CUST_A with SRv6 encapsulation:
```
vrf CUST_A
 address-family ipv4 unicast
  import/export route-target 65200:100
  segment-routing srv6
   locator MAIN
   alloc mode per-vrf
```
2. BGP VPNv4 between G-R1 and Gar-R2 via RR Gar-R6 — routes carry SRv6 SID (End.DT4) instead of MPLS VPN label.
3. Verify: `show bgp vpnv4 unicast rd 65200:100 <prefix>` — next-hop is IPv6 with SRv6 SID.
4. Verify: CE4 ↔ Gar-R2 VRF traffic via SRv6. **No MPLS labels anywhere.** Pure IPv6 encapsulation.

### Task 6: SRv6 + TI-LFA
1. `router isis CORE / address-family ipv6 unicast / fast-reroute per-prefix`.
2. Kill a link → TI-LFA repair using SRv6 repair SIDs. Sub-50ms.

### Task 7: Compare SR-MPLS vs SRv6
| | SR-MPLS (E04) | SRv6 (this lab) |
|---|---|---|
| Segments encoded as | MPLS labels (20-bit) | IPv6 addresses (128-bit) |
| Data plane | MPLS label switching | IPv6 forwarding + SRH |
| Overhead per segment | 4 bytes | 16 bytes |
| Hardware | Existing MPLS ASICs | IPv6-capable |
| VPN label equivalent | MPLS VPN label | End.DT4/DT6 SID |
| Flexibility | Limited | Network programming (uSID, functions) |

## Checklist
```
[ ] IPv6 on Garnet core; IS-IS ipv6 unicast; all IPv6 loopbacks reachable
[ ] SRv6 locator configured per router; SIDs allocated (End, End.X, End.DT4)
[ ] SRv6 SID types understood (End = node, End.X = adj, End.DT4 = VPN)
[ ] SRv6-TE policy G-R1 and Gar-R2 with explicit SID list
[ ] L3VPN over SRv6 (no MPLS, pure IPv6 encapsulation)
[ ] TI-LFA with SRv6 repair SIDs
[ ] SR-MPLS vs SRv6 comparison documented
```
