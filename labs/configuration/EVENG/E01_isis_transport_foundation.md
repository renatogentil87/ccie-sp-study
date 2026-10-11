# E01: IS-IS + Transport Foundation

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Emerald AS 65100 + Gold AS 65300 + Garnet AS 65200
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3

**End Goal:** IS-IS L2 on all 3 SPs. LDP on Emerald. SR-MPLS on Garnet. Full PE-PE LSP in each AS.

---

## Section 1: Emerald AS 65100 (IS-IS + LDP) - DONE

### Task 1: IP Addressing
1. Configure loopbacks and core link IPs on E-R1, E-R2, E-R3, E-R4, E-R6, E-R5 per the topology reference.
2. Verify: every directly connected link can ping its neighbor.

### Task 2: IS-IS on Emerald core
1. On each Emerald router: `router isis CORE / is-type level-2-only / net 49.0001.0000.0000.000X.00 / address-family ipv4 unicast / metric-style wide`.
2. Enable IS-IS on Loopback0 (passive) + all **core** interfaces (point-to-point). NOT on PE-CE interfaces.
   - E-R1: Gi0/0/0/2, Gi0/0/0/3 (core). NOT Gi0/0/0/0, Gi0/0/0/1 (PE-CE).
   - E-R2: Gi0/0/0/0, Gi0/0/0/3 (core). NOT Gi0/0/0/1, Gi0/0/0/2 (PE-CE).
   - E-R3: Gi0/0/0/0, Gi0/0/0/1, Gi0/0/0/2 (all core).
   - E-R4: Gi0/0/0/1, Gi0/0/0/2, Gi0/0/0/3 (all core).
   - E-R6: Gi0/0/0/2 (core to E-R4). NOT Gi0/0/0/1 (inter-AS to Garnet), Gi0/0/0/3 (inter-AS to Gold).
   - E-R5: Gi0/0/0/3 (core to E-R4).
3. Verify: `show isis neighbors` — all adjacencies L2/UP.
4. Verify: `show route ipv4` — all 6 Emerald loopbacks (1.1.1.1–6.6.6.6) reachable.

### Task 3: LDP on Emerald core
1. `mpls ldp / router-id <loopback> / address-family ipv4` + enable on ALL core interfaces.
2. Verify: `show mpls ldp neighbor` — sessions match IS-IS neighbors.
3. Verify: `show mpls forwarding` — labels for all loopbacks.
4. `traceroute mpls ipv4 2.2.2.2/32` from E-R1 — PUSH/SWAP/POP verified.

---

## Section 2: Garnet AS 65200 (IS-IS + SR-MPLS) - DONE

### Task 4: IP Addressing
1. Configure loopbacks and core link IPs on Gar-R1, Gar-R2, Gar-R3, Gar-R4, Gar-R5, Gar-R7, Gar-R6 per the topology reference.

### Task 5: IS-IS + SR on Garnet core
1. Same IS-IS config as Emerald + add `segment-routing mpls` under address-family.
2. Prefix-SID on each Loopback0:
   - Gar-R1=index 11 (16011), Gar-R2=12, Gar-R3=13, Gar-R4=14, Gar-R5=15, Gar-R6=16, Gar-R7=17
3. Enable IS-IS on core interfaces only:
   - Gar-R1: Gi0/0/0/1, Gi0/0/0/3 (core). NOT Gi0/0/0/0, Gi0/0/0/2 (PE-CE).
   - Gar-R2: Gi0/0/0/2, Gi0/0/0/3 (core). NOT Gi0/0/0/0, Gi0/0/0/1 (PE-CE).
   - Gar-R3: Gi0/0/0/0, Gi0/0/0/1, Gi0/0/0/2, Gi0/0/0/3 (all core).
   - Gar-R4: Gi0/0/0/0, Gi0/0/0/1, Gi0/0/0/3 (all core).
   - Gar-R5: Gi0/0/0/1, Gi0/0/0/2, Gi0/0/0/3 (all core).
   - Gar-R7: Gi0/0/0/2 (core). NOT Gi0/0/0/1 (inter-AS).
   - Gar-R6: Gi0/0/0/3 (core).
4. **No LDP on Garnet** — SR replaces it.
5. Verify: `show isis segment-routing label table` — SIDs 16011–16017.
6. Verify: `show mpls forwarding` — SR labels in LFIB.
7. `ping 12.12.12.12 source 11.11.11.11` from Gar-R1 — works via SR.

### Task 6: Verify no cross-AS connectivity yet
1. E-R1 cannot reach Gar-R1: `ping 11.11.11.11` from E-R1 → fails. Expected. Inter-AS = Lab E06.

---

## Section 3: Snapshot
1. Take GNS3 snapshot: **"E01-isis-ldp-sr-foundation"**

---

## Verification Checklist
```
[ ] Emerald: IS-IS L2 adjacencies on all core links
[ ] Emerald: all 6 loopbacks reachable via IS-IS
[ ] Emerald: LDP sessions match IS-IS neighbors; LFIB populated
[ ] Emerald: traceroute mpls shows PUSH/SWAP/POP
[ ] Garnet: IS-IS L2 + SR on all core links
[ ] Garnet: prefix-SIDs 16011-16017 in label table
[ ] Garnet: SR labels in LFIB (no LDP)
[ ] Garnet: Gar-R1↔Gar-R2 ping works via SR
[ ] No cross-AS connectivity (expected)
```

---

## Section 3: Gold AS 65300 (IS-IS + SRv6)

### Task 7: IP Addressing on Gold (IPv4 + IPv6)

**IPv4 Loopbacks:**

| Router | Loopback0 IPv4 |
|--------|---------------|
| G-R4 | 21.21.21.21/32 |
| G-R5 | 22.22.22.22/32 |
| G-R3 | 23.23.23.23/32 |
| G-R1 | 24.24.24.24/32 |
| G-R2 | 25.25.25.25/32 |

**IPv6 Loopbacks (SRv6 source address):**

| Router | Loopback0 IPv6 | SRv6 Locator |
|--------|---------------|-------------|
| G-R4 | fc00::21/128 | fc00:0:21::/48 |
| G-R5 | fc00::22/128 | fc00:0:22::/48 |
| G-R3 | fc00::23/128 | fc00:0:23::/48 |
| G-R1 | fc00::24/128 | fc00:0:24::/48 |
| G-R2 | fc00::25/128 | fc00:0:25::/48 |

**Core Link Addressing (dual-stack, IPv4 + IPv6):**

| Link | Interface A | Interface B | IPv4 Subnet | IPv6 Subnet |
|------|------------|------------|-------------|-------------|
| G-R4↔G-R5 | G-R4 Gi0/0/0/2 | G-R5 Gi0/0/0/2 | 10.3.1.0/24 (.1/.2) | fc00:3:1::/64 (::1/::2) |
| G-R4↔G-R3 | G-R4 Gi0/0/0/1 | G-R3 Gi0/0/0/1 | 10.3.2.0/24 (.1/.2) | fc00:3:2::/64 (::1/::2) |
| G-R5↔G-R3 | G-R5 Gi0/0/0/0 | G-R3 Gi0/0/0/0 | 10.3.3.0/24 (.1/.2) | fc00:3:3::/64 (::1/::2) |
| G-R3↔G-R1 | G-R3 Gi0/0/0/2 | G-R1 Gi0/0/0/2 | 10.3.4.0/24 (.1/.2) | fc00:3:4::/64 (::1/::2) |
| G-R3↔G-R2 | G-R3 Gi0/0/0/3 | G-R2 Gi0/0/0/3 | 10.3.5.0/24 (.1/.2) | fc00:3:5::/64 (::1/::2) |

> **Addressing logic:** `fc00::XX` = loopback (XX = last octet of IPv4). `fc00:0:XX::/48` = SRv6 locator. `fc00:3:Y::/64` = core links (3 = Gold, Y = link number matching 10.3.Y.0 IPv4). `::1`/`::2` = same convention as IPv4 .1/.2.

1. Configure both IPv4 AND IPv6 on every loopback and core interface.
2. SRv6 requires IPv6 — the packets ARE IPv6 packets. No IPv6 = no SRv6 forwarding.
3. Verify: every directly connected link can ping its neighbor on both IPv4 and IPv6.

### Task 8: IS-IS + SRv6 on Gold core
1. IS-IS L2 on all Gold routers (net 49.0003.xxxx.xxxx.xxxx.00). `metric-style wide`.
2. Enable **both** `address-family ipv4 unicast` and `address-family ipv6 unicast` under IS-IS (dual-stack, single-topology).
3. Configure SRv6 locator per router:
   ```
   segment-routing
    srv6
     encapsulation
      source-address fc00::XX        ← router's IPv6 loopback
     locators
      locator MAIN
       prefix fc00:0:XX::/48         ← router's locator
   ```
4. Reference the locator under IS-IS: `segment-routing srv6 / locator MAIN` under `address-family ipv6 unicast`.
5. Enable IS-IS on core interfaces (IPv4 + IPv6 address-families):
   - G-R4: Gi0/0/0/1 (G-R3), Gi0/0/0/2 (G-R5). NOT Gi0/0/0/3 (inter-AS to Emerald).
   - G-R5: Gi0/0/0/0 (G-R3), Gi0/0/0/2 (G-R4). NOT Gi0/0/0/3 (inter-AS to Garnet).
   - G-R3: Gi0/0/0/0 (G-R5), Gi0/0/0/1 (G-R4), Gi0/0/0/2 (G-R1), Gi0/0/0/3 (G-R2).
   - G-R1: Gi0/0/0/2 (G-R3). NOT Gi0/0/0/0 (CE9), Gi0/0/0/1 (CE8).
   - G-R2: Gi0/0/0/3 (G-R3). NOT Gi0/0/0/0 (CE7), Gi0/0/0/1 (CE8).
6. G-R4 = RR + Gar-R6 for Gold.
7. **No LDP on Gold** — SRv6 provides transport.
8. Verify: `show segment-routing srv6 sid` on each Gold router — End SIDs allocated per locator.
9. Verify: `show isis adjacency` — all Gold adjacencies L2/UP.
10. Verify: `ping fc00::25 source fc00::24` (G-R1 → G-R2 via IPv6/SRv6) — works.

### Task 9: Verify inter-AS links (no IGP/LDP across them)
1. E-R6(Gi3) ↔ G-R4(Gi3) — link UP, no IS-IS. eBGP later (E06).
2. G-R5(Gi3) ↔ Gar-R7(Gi3) — link UP, no IS-IS. eBGP later (E06).
3. E-R6(Gi1) ↔ Gar-R7(Gi1) — link UP, no IS-IS. eBGP later (E06). Direct Emerald↔Garnet path.

### Updated Checklist
```
[ ] Emerald: IS-IS + LDP (6 routers, all loopbacks reachable)
[ ] Gold: IS-IS + SRv6 (5 routers, SRv6 SIDs allocated, G-R1↔G-R2 works)
[ ] Garnet: IS-IS + SR-MPLS (7 routers, prefix-SIDs, Gar-R1↔Gar-R2 works)
[ ] 3 inter-AS links UP but no IGP across them
[ ] 3 different transport technologies operational (LDP / SRv6 / SR-MPLS)
```
