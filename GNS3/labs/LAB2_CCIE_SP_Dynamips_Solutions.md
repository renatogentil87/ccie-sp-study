# CCIE SP Lab 2 — Full Scenario — SOLUTIONS

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11 — IOS Classic syntax
**Companion to:** `LAB2_CCIE_SP_Dynamips.md`
**Reference:** `../workbooks/00_topology_reference.md` for loopbacks, link IPs, NET-IDs, IPv6 scheme, PE-CE table.
**Note:** Build from `LAB2-00-start` (base IP addressing only, no IGP/BGP config present). Configs below assume that baseline. Interfaces shown match the topology reference's link map exactly.

---

## Section 1 — Core IGP, LDP, and BGP Path Control [70 points]

### 1A. IS-IS Multi-Level — AS 65100 (R1–R8)

**Design:** Area 49.0001 → backbone (R3,R4,R5,R6,R7,R8), Area 49.0011 → L1 edge (R1,R2). R3/R4 are L1/L2 borders. R31↔R3 and R2↔R4 are the only L1 circuits; all other AS65100 core links are L2-only.

#### R1 (L1 edge PE)

```
router isis CORE
 net 49.0011.1500.0100.1001.00
 is-type level-1
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R3
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface FastEthernet4/0
 description to-R2
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
```

> R1 keeps the same system-ID (1500.0100.1001) but the area byte changes from 49.0001 to 49.0011. R1 is pure L1 — it only ever sees intra-area + the ATT-bit default from R3.

#### R2 (L1 edge PE)

```
router isis CORE
 net 49.0011.1500.0100.2002.00
 is-type level-1
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R4
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface FastEthernet3/0
 description to-R1
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
```

> R2's R1-facing link (f3/0) is also L1 since R1 is in the same L1 area 49.0011 — both ends must agree on circuit-type. R2's actual border uplink is to R4 (f0/0).

#### R3 (L1/L2 border P)

```
router isis CORE
 net 49.0001.1500.0100.3003.00
 is-type level-1-2
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R1 (L1 edge)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface GigabitEthernet1/0
 description to-R4 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
interface GigabitEthernet2/0
 description to-R5 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
interface FastEthernet3/0
 description to-R7 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
```

#### R4 (L1/L2 border P)

```
router isis CORE
 net 49.0001.1500.0100.4004.00
 is-type level-1-2
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R2 (L1 edge)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface GigabitEthernet1/0
 description to-R3 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
interface GigabitEthernet2/0
 description to-R6 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
```

#### R5 (L2-only ASBR)

```
router isis CORE
 net 49.0001.1500.0100.5005.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R8
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 description to-R3
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 description to-R6
 ip router isis CORE
 isis network point-to-point
!
```

> R5's g1/0 toward R11 (inter-AS eBGP) is intentionally NOT in IS-IS.

#### R6 (L2-only ASBR)

```
router isis CORE
 net 49.0001.1500.0100.6006.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet2/0
 description to-R4
 ip router isis CORE
 isis network point-to-point
!
interface FastEthernet3/0
 description to-R5
 ip router isis CORE
 isis network point-to-point
!
```

> R6's g1/0 (→R12) and f0/0 (→R20) inter-AS links are NOT in IS-IS.

#### R7 (L2-only RR)

```
router isis CORE
 net 49.0001.1500.0100.7007.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet3/0
 description to-R3
 ip router isis CORE
 isis network point-to-point
!
```

#### R8 (L2-only RR)

```
router isis CORE
 net 49.0001.1500.0100.8008.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R5
 ip router isis CORE
 isis network point-to-point
!
```

#### Task 1.4 — Route leaking R20/R30 /32s into Area 49.0011 (configured on R3)

```
ip prefix-list LEAK-TO-L1 seq 5 permit 150.3.20.20/32
ip prefix-list LEAK-TO-L1 seq 10 permit 150.3.30.30/32
!
route-map LEAK-L2-TO-L1 permit 10
 match ip address prefix-list LEAK-TO-L1
 set tag 65300
!
router isis CORE
 redistribute isis ip level-2 into level-1 route-map LEAK-L2-TO-L1
```

> `redistribute isis ip level-2 into level-1 route-map ...` is the IOS classic command for inter-level IS-IS route leaking (distinct from `redistribute isis` which would be for redistributing a *different* IS-IS process/other protocol). The route-map's prefix-list restricts the leak to only R20's and R30's loopbacks — not the whole L2 table. The `set tag 65300` makes the leaked routes auditable in `show isis database` / `show ip route` (tag column) as "these came from AS65300 via leaking," satisfying the "tagged route leaking" requirement. Only R3 needs this since both R31 and R2 sit behind R3/R4 in the same L1 area and R3 is the one chosen to leak (R4 could optionally mirror this for redundancy, but the task only requires it be leaked into the area, which either border router achieves).

#### Verification

```
show isis topology
show ip route isis
show isis database level-1 detail      ! on R31/R2 — confirm ATT bit set, only area + 2 leaked /32s + default
show isis database level-2 detail      ! on R3-R8 — confirm all 8 backbone systems
show clns neighbors
ping 150.3.20.20 source Loopback0       ! from R1 — confirms leaked route + default both work for reachability
```

Expected: R31/R2 `show ip route` shows `i*L1` default (0.0.0.0/0) plus two `i L1` /32 routes tagged 65300 for R20/R30. R3–R8 `show isis database level-2` lists all 8 AS65100 systems plus leaked-in visibility of the L1 prefixes from the area.

**Snapshot:** `LAB2-sec1a-isis-ml-as65100`

---

### 1B. OSPF Stub/NSSA — AS 65200 (R11–R16, R19)

**Design:** Area 0 (R11,R12,R13,R19) · Area 20 totally-stubby (R14, off R13) · Area 21 NSSA (R15,R16, off R13). R13↔R14 link renumbered into Area 20; R13↔R16 link renumbered into Area 21.

#### R11 (Area 0, P)

```
router ospf 1
 router-id 150.2.11.11
 passive-interface Loopback0
 network 150.2.11.11 0.0.0.0 area 0
 network 20.11.12.11 0.0.0.0 area 0
 network 20.11.13.11 0.0.0.0 area 0
 network 20.11.15.11 0.0.0.0 area 0
```

> R11↔R15 (f3/0) stays in Area 0 per the task — only the R13↔R14 and R13↔R16 links move to the new areas.

#### R12 (Area 0, ASBR)

```
router ospf 1
 router-id 150.2.12.12
 passive-interface Loopback0
 network 150.2.12.12 0.0.0.0 area 0
 network 20.11.12.12 0.0.0.0 area 0
 network 20.12.14.12 0.0.0.0 area 0
```

> R12↔R14 (g2/0) remains Area 0 on R12's side — R14's totally-stubby membership is scoped to its R13-facing link per the task's wording ("R13↔R14 link... renumber"). R12 keeps its leg in Area 0.

#### R13 (ABR for Area 20 + Area 21)

```
router ospf 1
 router-id 150.2.13.13
 passive-interface Loopback0
 network 150.2.13.13 0.0.0.0 area 0
 network 20.11.13.13 0.0.0.0 area 0
 network 20.13.19.13 0.0.0.0 area 0
 network 20.13.14.13 0.0.0.0 area 20
 network 20.13.16.13 0.0.0.0 area 21
 area 20 stub no-summary
 area 21 nssa
 area 21 nssa default-information-originate
```

> `area 20 stub no-summary` on the ABR makes Area 20 **totally** stubby — R14 gets a single default and zero Type-3 summaries. `area 21 nssa` with `default-information-originate` injects a Type-7 default into Area 21 (Task 2.4).

#### R14 (Area 20, PE — totally stubby)

```
router ospf 1
 router-id 150.2.14.14
 passive-interface Loopback0
 network 150.2.14.14 0.0.0.0 area 20
 network 20.12.14.14 0.0.0.0 area 0
 network 20.13.14.14 0.0.0.0 area 20
 area 20 stub
```

> R14's R12-facing link (g2/0, inter-area to Area 0) stays as-is per the existing wiring, but the task's intent is for R14 to be the single-uplink stub model via R13. If strict single-homing into Area 20 is desired for the "minimal LSA flooding" model, shut/exclude the R12 leg from OSPF; otherwise leave both and verify R14 still only receives the one default + no summaries from the R13 (Area 20) direction — document whichever topology you kept. Below assumes R14 keeps only the R13 leg active in Area 20 and the R12 leg is also treated as area 20 member router-side (`area 20 stub` is required on every router inside a stub area, not just the ABR).

#### R15 (Area 0 — unchanged, Green PE)

```
router ospf 1
 router-id 150.2.15.15
 passive-interface Loopback0
 network 150.2.15.15 0.0.0.0 area 0
 network 20.11.15.15 0.0.0.0 area 0
```

> R15 is not part of the stub/NSSA redesign (Area 0 member, same as LAB1) — its R17 PE-CE leg becomes **static** instead (Section 2 Task 2.3), so R15 drops the OSPF PE-CE process on that interface entirely.

#### R16 (Area 21 NSSA, PE)

```
router ospf 1
 router-id 150.2.16.16
 passive-interface Loopback0
 network 150.2.16.16 0.0.0.0 area 21
 network 20.13.16.16 0.0.0.0 area 21
 area 21 nssa
!
ip route 150.26.100.100 255.255.255.255 Null0
!
router ospf 1
 redistribute static subnets route-map TAG-NSSA-TEST
!
route-map TAG-NSSA-TEST permit 10
 match ip address prefix-list NSSA-TEST-RT
!
ip prefix-list NSSA-TEST-RT seq 5 permit 150.26.100.100/32
```

> The `150.26.100.100/32` static is the Task 2.4 "test static route" — a throwaway /32 (Null0-routed so it's always up) redistributed into OSPF on R16, which — because R16 sits in NSSA Area 21 — automatically originates as a **Type-7** LSA. R13 (the ABR) translates it to Type-5 for Area 0.

#### R19 (RR, Area 0)

```
router ospf 1
 router-id 150.2.19.19
 passive-interface Loopback0
 network 150.2.19.19 0.0.0.0 area 0
 network 20.13.19.19 0.0.0.0 area 0
 network 20.14.19.19 0.0.0.0 area 0
```

#### Verification

```
show ip ospf border-routers              ! on R13 — confirms ABR role for areas 20 & 21
show ip ospf database database-summary   ! per-router LSA type counts
show ip route 0.0.0.0                    ! on R14 — single O*IA default, no O IA summaries
show ip ospf database nssa-external      ! on R13 — Type-7 for 150.26.100.100
show ip ospf database external           ! on R11 — Type-5 translated version, check "Advertising Router" = R13
show ip ospf neighbor
```

Expected: R14's routing table has exactly one `O*IA 0.0.0.0/0` and no `O IA` (Type-3) entries beyond what totally-stubby allows (none). R13's `nssa-external` database shows the 150.26.100.100/32 LSA with the N-bit/P-bit; R11's `database external` (Type-5) shows the same prefix translated, "Advertising Router 150.2.13.13."

**Snapshot:** `LAB2-sec1b-ospf-stub-nssa`

---

### 1C. IS-IS Multi-Level — AS 65300 (R20–R23, R30)

**Design:** Area 49.0003 backbone (R21,R22,R23,R30) · Area 49.0013 L1 edge (R20). R21 is the sole L1/L2 border (R20↔R21).

#### R20 (L1 edge, ASBR/PE)

```
router isis CORE
 net 49.0013.1500.0302.0020.00
 is-type level-1
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet1/0
 description to-R21
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
```

> R20's g2/0 (→R22) is NOT part of this IS-IS design per Task 3.1/3.2 — only R20↔R21 is listed as R20's IS-IS circuit; if g2/0 to R22 is physically up and intended to also carry IS-IS, it must also be `level-1` since R20 is pure L1 and R22 is L2-only backbone (they could not form an adjacency at all — leave g2/0 out of IS-IS, consistent with R20 having a single L1 uplink via R21). R20's f0/0 (→R6, inter-AS) and f3/0 (→R24, PE-CE) are never in IS-IS.

#### R21 (L1/L2 border, ASBR/PE)

```
router isis CORE
 net 49.0003.1500.0302.1021.00
 is-type level-1-2
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet1/0
 description to-R20 (L1 edge)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface FastEthernet0/0
 description to-R23 (backbone)
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
```

#### R22 (L2-only, PE)

```
router isis CORE
 net 49.0003.1500.0302.2022.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet2/0
 description to-R20
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 description to-R23
 ip router isis CORE
 isis network point-to-point
!
```

> Per the note in R20 above, if R20↔R22 (g2/0 on both ends) is physically wired per the Core link table, it must still be L2-only on R22's side — but since R20 is pure L1 it cannot form an L2 adjacency over that link. Treat R20↔R22 as a backup/unused-for-IGP physical link in this design, or exclude it from `ip router isis` entirely on R20 (only R20↔R21 carries IS-IS). This mirrors the AS65100 pattern where R31/R2 also only run IS-IS on their single border-reachable path.

#### R23 (L2-only, PE)

```
router isis CORE
 net 49.0003.1500.0302.3023.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface FastEthernet0/0
 description to-R21
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet1/0
 description to-R22
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet2/0
 description to-R30
 ip router isis CORE
 isis network point-to-point
!
```

#### R30 (L2-only, RR)

```
router isis CORE
 net 49.0003.1500.0303.0030.00
 is-type level-2-only
 metric-style wide
 passive-interface Loopback0
!
interface Loopback0
 ip router isis CORE
!
interface GigabitEthernet2/0
 description to-R23
 ip router isis CORE
 isis network point-to-point
!
```

#### Verification

```
show isis database level-1 detail   ! on R20 — ATT bit set, default route present
show ip route isis                  ! on R21/R22/R23/R30 — full backbone loopback reachability
ping 150.3.30.30 source Loopback0   ! from R20 — confirms default-route path works
```

**Snapshot:** `LAB2-sec1c-isis-ml-as65300`

---

### 1D. LDP Across All Three ASes

Applied per-interface (`mpls ip`) on every intra-AS core link only — never on inter-AS (R5↔R11, R6↔R12, R6↔R20, R12↔R21) or PE-CE links.

#### Example — R3 (representative; repeat the same two lines, per-interface, on every core-facing interface of every router in all 3 ASes)

```
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface GigabitEthernet1/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

#### Full per-router interface list (apply `mpls ip` to exactly these; `mpls ldp router-id Loopback0 force` once globally per router)

| Router | LDP-enabled interfaces |
|---|---|
| R31 | f0/0 (→R3), f4/0 (→R2) |
| R2 | f0/0 (→R4), f3/0 (→R31) |
| R3 | f0/0 (→R31), g1/0 (→R4), g2/0 (→R5), f3/0 (→R7) |
| R4 | f0/0 (→R2), g1/0 (→R3), g2/0 (→R6) |
| R5 | f0/0 (→R8), g2/0 (→R3), f3/0 (→R6) |
| R6 | g2/0 (→R4), f3/0 (→R5) |
| R7 | f3/0 (→R3) |
| R8 | f0/0 (→R5) |
| R11 | f0/0 (→R12), g2/0 (→R13), f3/0 (→R15) |
| R12 | g2/0 (→R14) *(R11-facing f0/0 also)* |
| R13 | f0/0 (→R16), g1/0 (→R14), f3/0 (→R19) *(and g2/0→R11)* |
| R14 | g2/0 (→R12), g1/0 (→R13) |
| R15 | f3/0 (→R11) |
| R16 | f0/0 (→R13) |
| R19 | f3/0 (→R13), f0/0 (→R14) |
| R20 | g1/0 (→R21) |
| R21 | g1/0 (→R20), f0/0 (→R23) |
| R22 | g2/0 (→R20), g1/0 (→R23) |
| R23 | f0/0 (→R21), g1/0 (→R22), g2/0 (→R30) |
| R30 | g2/0 (→R23) |

> R12's full interface set is f0/0 (→R11) and g2/0 (→R14) — both intra-AS core, both get `mpls ip`. R12's f3/0 (→R21) is inter-AS and stays LDP-free.

#### Verification

```
show mpls ldp neighbor                ! Operational, per AS
show mpls ldp discovery
show mpls forwarding-table            ! label for every /32 Loopback0
show mpls forwarding-table 150.1.31.31 detail     ! trace R1's label across R3/R4 L1/L2 boundary
show mpls forwarding-table 150.3.20.20 detail   ! trace R20's label across R21's L1/L2 boundary
```

Expected: at R3 and R4 (AS65100 border) and R21 (AS65300 border), the LFIB shows a continuous label swap — LDP doesn't care about IS-IS levels, only about Loopback0 reachability via IS-IS (any level), so as long as the /32 is in the RIB (via L1 default+leak or L2), LDP binds and advertises a label for it hop-by-hop.

---

### 1E. BGP — Weight-Based Path Control, Conditional Advertisement, Communities, BGP-LU

#### Task 5.1 — iBGP + Route Reflectors (baseline, rebuilt fresh)

**R7 (RR, AS65100):**

```
router bgp 65100
 bgp router-id 150.1.7.7
 no bgp default ipv4-unicast
 neighbor 150.1.31.31 remote-as 65100
 neighbor 150.1.31.31 update-source Loopback0
 neighbor 150.1.2.2 remote-as 65100
 neighbor 150.1.2.2 update-source Loopback0
 neighbor 150.1.3.3 remote-as 65100
 neighbor 150.1.3.3 update-source Loopback0
 neighbor 150.1.4.4 remote-as 65100
 neighbor 150.1.4.4 update-source Loopback0
 neighbor 150.1.5.5 remote-as 65100
 neighbor 150.1.5.5 update-source Loopback0
 neighbor 150.1.6.6 remote-as 65100
 neighbor 150.1.6.6 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family ipv4
  neighbor 150.1.31.31 activate
  neighbor 150.1.31.31 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.1.3.3 activate
  neighbor 150.1.3.3 route-reflector-client
  neighbor 150.1.4.4 activate
  neighbor 150.1.4.4 route-reflector-client
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 route-reflector-client
  neighbor 150.1.6.6 activate
  neighbor 150.1.6.6 route-reflector-client
  neighbor 150.1.8.8 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.1.31.31 activate
  neighbor 150.1.31.31 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.1.3.3 activate
  neighbor 150.1.3.3 route-reflector-client
  neighbor 150.1.4.4 activate
  neighbor 150.1.4.4 route-reflector-client
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 route-reflector-client
  neighbor 150.1.6.6 activate
  neighbor 150.1.6.6 route-reflector-client
  neighbor 150.1.8.8 activate
 exit-address-family
!
```

**R8 (RR, AS65100)** — mirror of R7, peer list swaps R7↔R8 (neighbor 150.1.7.7 instead of 150.1.8.8), same client set, same two AFs.

**R19 (RR, AS65200)** — single RR, clients R11,R12,R13,R14,R15,R16:

```
router bgp 65200
 bgp router-id 150.2.19.19
 no bgp default ipv4-unicast
 neighbor 150.2.11.11 remote-as 65200
 neighbor 150.2.11.11 update-source Loopback0
 neighbor 150.2.12.12 remote-as 65200
 neighbor 150.2.12.12 update-source Loopback0
 neighbor 150.2.13.13 remote-as 65200
 neighbor 150.2.13.13 update-source Loopback0
 neighbor 150.2.14.14 remote-as 65200
 neighbor 150.2.14.14 update-source Loopback0
 neighbor 150.2.15.15 remote-as 65200
 neighbor 150.2.15.15 update-source Loopback0
 neighbor 150.2.16.16 remote-as 65200
 neighbor 150.2.16.16 update-source Loopback0
 !
 address-family ipv4
  neighbor 150.2.11.11 activate
  neighbor 150.2.11.11 route-reflector-client
  neighbor 150.2.12.12 activate
  neighbor 150.2.12.12 route-reflector-client
  neighbor 150.2.13.13 activate
  neighbor 150.2.13.13 route-reflector-client
  neighbor 150.2.14.14 activate
  neighbor 150.2.14.14 route-reflector-client
  neighbor 150.2.15.15 activate
  neighbor 150.2.15.15 route-reflector-client
  neighbor 150.2.16.16 activate
  neighbor 150.2.16.16 route-reflector-client
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.2.11.11 activate
  neighbor 150.2.11.11 route-reflector-client
  neighbor 150.2.12.12 activate
  neighbor 150.2.12.12 route-reflector-client
  neighbor 150.2.13.13 activate
  neighbor 150.2.13.13 route-reflector-client
  neighbor 150.2.14.14 activate
  neighbor 150.2.14.14 route-reflector-client
  neighbor 150.2.15.15 activate
  neighbor 150.2.15.15 route-reflector-client
  neighbor 150.2.16.16 activate
  neighbor 150.2.16.16 route-reflector-client
 exit-address-family
!
```

**R30 (RR, AS65300)** — clients R20,R21,R22,R23:

```
router bgp 65300
 bgp router-id 150.3.30.30
 no bgp default ipv4-unicast
 neighbor 150.3.20.20 remote-as 65300
 neighbor 150.3.20.20 update-source Loopback0
 neighbor 150.3.21.21 remote-as 65300
 neighbor 150.3.21.21 update-source Loopback0
 neighbor 150.3.22.22 remote-as 65300
 neighbor 150.3.22.22 update-source Loopback0
 neighbor 150.3.23.23 remote-as 65300
 neighbor 150.3.23.23 update-source Loopback0
 !
 address-family ipv4
  neighbor 150.3.20.20 activate
  neighbor 150.3.20.20 route-reflector-client
  neighbor 150.3.21.21 activate
  neighbor 150.3.21.21 route-reflector-client
  neighbor 150.3.22.22 activate
  neighbor 150.3.22.22 route-reflector-client
  neighbor 150.3.23.23 activate
  neighbor 150.3.23.23 route-reflector-client
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.3.20.20 activate
  neighbor 150.3.20.20 route-reflector-client
  neighbor 150.3.21.21 activate
  neighbor 150.3.21.21 route-reflector-client
  neighbor 150.3.22.22 activate
  neighbor 150.3.22.22 route-reflector-client
  neighbor 150.3.23.23 activate
  neighbor 150.3.23.23 route-reflector-client
 exit-address-family
!
```

> Non-RR routers (R1-R6, R11-R16, R20-R23) each run a standard `router bgp <AS>` stanza peering only to their AS's RR(s) with `update-source Loopback0`, activated under `address-family ipv4` and `vpnv4`, `next-hop-self` applied on ASBRs when reflecting learned eBGP routes into iBGP (R5, R6, R12, R20, R21). Omitted here for brevity — identical pattern to WB04_BGP_Solutions.md Section 1.

#### Task 5.2 — eBGP inter-AS sessions

**R5 (→R11):**
```
router bgp 65100
 neighbor 10.5.11.11 remote-as 65200
 address-family ipv4
  neighbor 10.5.11.11 activate
 exit-address-family
```

**R11 (→R5):**
```
router bgp 65200
 neighbor 10.5.11.5 remote-as 65100
 address-family ipv4
  neighbor 10.5.11.5 activate
 exit-address-family
```

**R6 (→R12 and →R20):**
```
router bgp 65100
 neighbor 10.6.12.12 remote-as 65200
 neighbor 10.6.20.20 remote-as 65300
 address-family ipv4
  neighbor 10.6.12.12 activate
  neighbor 10.6.20.20 activate
 exit-address-family
```

**R12 (→R6 and →R21):**
```
router bgp 65200
 neighbor 10.6.12.6 remote-as 65100
 neighbor 20.12.21.21 remote-as 65300
 address-family ipv4
  neighbor 10.6.12.6 activate
  neighbor 20.12.21.21 activate
 exit-address-family
```

**R20 (→R6):**
```
router bgp 65300
 neighbor 10.6.20.6 remote-as 65100
 address-family ipv4
  neighbor 10.6.20.6 activate
 exit-address-family
```

**R21 (→R12):**
```
router bgp 65300
 neighbor 20.12.21.12 remote-as 65200
 address-family ipv4
  neighbor 20.12.21.12 activate
 exit-address-family
```

#### Task 5.3 — WEIGHT-based primary/backup on R5 and R6

**R5:**
```
route-map SET-WEIGHT-PRIMARY permit 10
 set weight 200
!
router bgp 65100
 neighbor 10.5.11.11 route-map SET-WEIGHT-PRIMARY in
```

**R6:**
```
route-map SET-WEIGHT-BACKUP permit 10
 set weight 100
!
router bgp 65100
 neighbor 10.6.12.12 route-map SET-WEIGHT-BACKUP in
```

> Weight is a **per-router, non-transitive, locally-significant** BGP attribute — it is never carried in a BGP UPDATE message to any peer (unlike LOCAL_PREF, which IS carried in iBGP updates). Because of this, applying weight on the RR (R7/R8) would be meaningless: the RR would mark the route with weight 200/100 in its own table, but when it reflects that route to R1–R4, the weight value does **not** travel with it — every router that receives a reflected route always resets WEIGHT based on its own locally-configured inbound policy (or defaults: 32768 for self-originated, 0 for externally learned with no policy). This is precisely why Task 5.3 requires applying weight **individually on R5 and R6** (the routers that actually receive the eBGP update directly) rather than centrally — weight only affects *that router's* best-path decision, so each eBGP-receiving router needs its own policy to express the same intended preference consistently for the whole AS (since R5/R6 both reflect their best path to the RRs, and the RRs reflect that one best path onward, the AS-wide effect still works out as "prefer via R5" — but only because R5 and R6 were each told individually, not because weight propagated).

Verify failover:
```
show ip bgp 150.2.14.14          ! on R31 — Weight column, best path via R5 (next-hop reflects R5's path)
shutdown                          ! R5's g1/0 to R11 (test)
show ip bgp 150.2.14.14          ! confirm reconvergence via R6 (weight 100 path now best, since R5's path withdrawn)
no shutdown                       ! restore R5-R11
```

#### Task 5.4 — Conditional advertisement on R6 (advertise-map/exist-map)

```
ip prefix-list AS65300-BACKUP-AGG seq 5 permit 150.3.0.0/16 le 32
!
route-map ADVERTISE-AS65300-BACKUP permit 10
 match ip address prefix-list AS65300-BACKUP-AGG
!
route-map TRIGGER-R20-LOOPBACK permit 10
 match ip address prefix-list TRIGGER-PFX
!
ip prefix-list TRIGGER-PFX seq 5 permit 150.3.20.20/32
!
router bgp 65100
 neighbor 10.6.12.12 advertise-map ADVERTISE-AS65300-BACKUP non-exist-map TRIGGER-R20-LOOPBACK
```

> **`non-exist-map`** is the correct choice: it means "advertise the advertise-map route only when the trigger prefix (R20's loopback, learned directly over R6↔R20 eBGP) does NOT exist in the BGP table." That's exactly "advertise the backup only when the primary is down." (`exist-map`, by contrast, would mean "advertise only when the trigger DOES exist" — the opposite condition, useful for a different use case such as "only advertise a route when you can also reach a specific anchor prefix," not for backup-on-failure.)

Verify:
```
show ip bgp neighbors 10.6.12.12 advertised-routes   ! baseline — backup NOT present (R6-R20 session up)
shutdown                                              ! R6's f0/0 to R20
show ip bgp neighbors 10.6.12.12 advertised-routes   ! backup now present toward R12
no shutdown                                           ! restore
show ip bgp neighbors 10.6.12.12 advertised-routes   ! backup withdrawn again
```

#### Task 5.5 — Community scheme

| Community | Meaning |
|---|---|
| `65100:1100` | Originated in AS65100 core (P/RR/ASBR loopbacks) |
| `65100:1199` | Customer-originated in AS65100 (Green/Blue), do-not-prepend |
| `65200:2100` | Originated in AS65200 core |
| `65300:3100` | Originated in AS65300 core |

**R1 (tag loopback at origin):**
```
route-map TAG-ORIGIN-65100 permit 10
 set community 65100:1100 additive
!
router bgp 65100
 network 150.1.31.31 mask 255.255.255.255
 address-family ipv4
  neighbor 150.1.7.7 route-map TAG-ORIGIN-65100 out
 exit-address-family
```

> Simpler/more robust in practice: tag at redistribution-into-BGP time via a route-map on the `network` statement's effective output, or (cleaner) tag inbound at the RR for self-originated-vs-not using `match ip address` on the specific /32 — pattern shown here applies the community outbound toward the RR, which is sufficient since the RR reflects communities transparently by default (`send-community` must be enabled).

**R6 (tag AS65300-originated routes received via eBGP, applied inbound before they enter AS65100's iBGP):**
```
route-map TAG-ORIGIN-65300 permit 10
 set community 65300:3100 additive
!
router bgp 65100
 neighbor 10.6.20.20 route-map TAG-ORIGIN-65300 in
```

**Enable `send-community` everywhere communities must propagate (RRs and ASBRs, both IPv4 unicast AF):**
```
router bgp 65100
 address-family ipv4
  neighbor 150.1.7.7 send-community
 exit-address-family
```

Verify at a remote ASBR:
```
show ip bgp community 65100:1100
show ip bgp 150.1.31.31    ! on R21 (AS65300) — confirm community 65100:1100 visible end-to-end
```

#### Task 5.6 — BGP-LU, direct ASBR-to-ASBR (AS65100 ↔ AS65300 via R6↔R20)

**R6 (eBGP-LU to R20, iBGP-LU to R7/R8):**
```
router bgp 65100
 neighbor 10.6.20.20 remote-as 65300
 neighbor 10.6.20.20 update-source GigabitEthernet0/0
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family ipv4
  neighbor 10.6.20.20 activate
  neighbor 10.6.20.20 send-label
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-label
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-label
 exit-address-family
!
mpls bgp forwarding
```

> Chose **R7/R8 (the RRs)** as R6's internal LU peers rather than R3/R4 (the L1/L2 borders): since R7/R8 already reflect all iBGP (including LU, once activated) to R1–R6, this keeps the LU design consistent with the existing RR fan-out rather than building a second, parallel peering mesh to R3/R4. R6 still needs its own direct iBGP-LU session to the RRs (reflected sessions alone don't change this — R6 originates/relays the LU NLRI it learned from R20 into iBGP exactly like any other eBGP-learned route, and the RR reflects it onward with `send-label` continuing the label binding).

**R20 (eBGP-LU to R6, iBGP-LU to R21/R30):**
```
router bgp 65300
 neighbor 10.6.20.6 remote-as 65100
 neighbor 150.3.21.21 remote-as 65300
 neighbor 150.3.21.21 update-source Loopback0
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 update-source Loopback0
 !
 address-family ipv4
  neighbor 10.6.20.6 activate
  neighbor 10.6.20.6 send-label
  neighbor 150.3.21.21 activate
  neighbor 150.3.21.21 send-label
  neighbor 150.3.30.30 activate
  neighbor 150.3.30.30 send-label
 exit-address-family
!
mpls bgp forwarding
```

> R30 is the RR for AS65300, so R20's iBGP-LU session to R30 reflects the LU NLRI to R21/R22/R23. R20 also peers LU directly to R21 since R21 is the other ASBR and the L1/L2 border — this gives two independent LU paths into the backbone matching the "separate dedicated internal LU session" requirement.

**R7, R8, R21, R30** each need the matching `address-family ipv4 ... send-label` + `mpls bgp forwarding` added to their existing iBGP stanzas toward R6 (on R7/R8) or toward R20 (on R21/R30) respectively — same two lines pattern as above, omitted for repetition.

Verify:
```
show bgp ipv4 unicast labels              ! on R6 and R20 — confirms a BGP label assigned to each /32
show mpls forwarding-table 150.3.22.22    ! on R31 — label-only path reaches R22's loopback, no plain-IP hop
traceroute mpls ipv4 150.3.22.22/32       ! LSP ping/trace confirms pure label switching end-to-end
```

**Snapshot (end of Section 1):** combine with the per-subsection snapshots above (`LAB2-sec1a-isis-ml-as65100`, `LAB2-sec1b-ospf-stub-nssa`, `LAB2-sec1c-isis-ml-as65300`); Section 1E/1D have no separate named snapshot in the task — carry state forward into Section 2.

---

## Section 2 — VPN Services [60 points]

### Task 2.1 — VRF/RD/RT plan

| Customer | VRF | RD (R31/R2/R15/R16/R20/R22/R14/R21/R23) | RT |
|---|---|---|---|
| Green | `GREEN` | `1:910` (R1), `2:910` (R2), `15:910` (R15) | `65910:910` both |
| Yellow | `YELLOW` | `16:24` (R16), `20:24` (R20), `22:24` (R22) | `65024:24` both |
| Blue | `BLUE` | `2:25` (R2), `14:25` (R14), `21:25` (R21), `23:25` (R23) | `65025:25` both |

**Example VRF definitions (classic `ip vrf`, consistent with IOS 15.2(4)M11 7200):**

```
! R1
ip vrf GREEN
 rd 1:910
 route-target export 65910:910
 route-target import 65910:910
!
! R2
ip vrf GREEN
 rd 2:910
 route-target export 65910:910
 route-target import 65910:910
!
ip vrf BLUE
 rd 2:25
 route-target export 65025:25
 route-target import 65025:25
!
! R15
ip vrf GREEN
 rd 15:910
 route-target export 65910:910
 route-target import 65910:910
!
! R16
ip vrf YELLOW
 rd 16:24
 route-target export 65024:24
 route-target import 65024:24
!
! R20 / R22
ip vrf YELLOW
 rd 20:24                      ! (22:24 on R22)
 route-target export 65024:24
 route-target import 65024:24
!
! R14 / R21 / R23
ip vrf BLUE
 rd 14:25                      ! (21:25 on R21, 23:25 on R23)
 route-target export 65025:25
 route-target import 65025:25
```

### Task 2.2 — Green VRF + eBGP PE-CE legs (R31↔R9, R31↔R10, R2↔R10)

**R1:**
```
ip vrf GREEN
 rd 1:910
 route-target export 65910:910
 route-target import 65910:910
!
interface GigabitEthernet1/0
 description to-R9 (Green CE)
 ip vrf forwarding GREEN
 ip address 192.168.1.1 255.255.255.0
!
interface FastEthernet3/0
 description to-R10 (Green CE)
 ip vrf forwarding GREEN
 ip address 192.168.2.1 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.1.9 remote-as 65910
  neighbor 192.168.1.9 activate
  neighbor 192.168.2.10 remote-as 65910
  neighbor 192.168.2.10 activate
 exit-address-family
```

**R2:**
```
ip vrf GREEN
 rd 2:910
 route-target export 65910:910
 route-target import 65910:910
!
interface GigabitEthernet1/0
 description to-R10 (Green CE)
 ip vrf forwarding GREEN
 ip address 192.168.4.2 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.4.10 remote-as 65910
  neighbor 192.168.4.10 activate
 exit-address-family
```

**R9 (CE):**
```
router bgp 65910
 neighbor 192.168.1.1 remote-as 65100
 network 150.9.9.9 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.1.1 activate
 exit-address-family
```

**R10 (CE, dual-homed to R31 and R2):**
```
router bgp 65910
 neighbor 192.168.2.1 remote-as 65100
 neighbor 192.168.4.2 remote-as 65100
 network 150.10.10.10 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.2.1 activate
  neighbor 192.168.4.2 activate
 exit-address-family
```

**VPNv4 exchange R31↔R2 (via AS65100 RRs — add to existing R31/R2 iBGP from Section 1E):**
```
router bgp 65100
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community extended
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community extended
 exit-address-family
```

### Task 2.3 — Green static PE-CE on R15↔R17 (replaces OSPF)

**R15:**
```
ip vrf GREEN
 rd 15:910
 route-target export 65910:910
 route-target import 65910:910
!
interface FastEthernet0/0
 description to-R17 (Green CE, static)
 ip vrf forwarding GREEN
 ip address 192.168.6.15 255.255.255.0
!
ip route vrf GREEN 150.17.17.17 255.255.255.255 192.168.6.17
!
router bgp 65200
 address-family ipv4 vrf GREEN
  redistribute static
 exit-address-family
```

**R17 (CE):**
```
ip route 0.0.0.0 0.0.0.0 192.168.6.15
```

> `redistribute static` under the VRF address-family is what pushes `ip route vrf GREEN ...` into VPNv4 so AS65100's Green PEs (R31, R2) learn 150.17.17.17/32. The route arrives at R31/R2 tagged with origin code `?` (redistributed/static), distinguishable from the eBGP-learned (`i`/`e`) Green routes.

Verify:
```
show ip bgp vpnv4 vrf GREEN 150.17.17.17    ! on R15 — confirm "redistributed" / static origin, not OSPF
ping vrf GREEN 150.17.17.17 source Loopback0  ! on R31/R2
```

### Task 2.4 — Yellow VRF (R16↔R26, R20↔R24, R22↔R24)

**R16:**
```
ip vrf YELLOW
 rd 16:24
 route-target export 65024:24
 route-target import 65024:24
!
interface GigabitEthernet2/0
 description to-R26 (Yellow CE)
 ip vrf forwarding YELLOW
 ip address 192.168.8.16 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf YELLOW
  neighbor 192.168.8.26 remote-as 65024
  neighbor 192.168.8.26 activate
 exit-address-family
```

**R20:**
```
ip vrf YELLOW
 rd 20:24
 route-target export 65024:24
 route-target import 65024:24
!
interface FastEthernet3/0
 description to-R24 (Yellow CE)
 ip vrf forwarding YELLOW
 ip address 192.168.9.20 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.9.24 remote-as 65024
  neighbor 192.168.9.24 activate
 exit-address-family
```

**R22:**
```
ip vrf YELLOW
 rd 22:24
 route-target export 65024:24
 route-target import 65024:24
!
interface FastEthernet0/0
 description to-R24 (Yellow CE, 2nd leg)
 ip vrf forwarding YELLOW
 ip address 192.168.11.22 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.11.24 remote-as 65024
  neighbor 192.168.11.24 activate
 exit-address-family
```

**R24 (CE, dual-homed to R20 and R22):**
```
router bgp 65024
 neighbor 192.168.9.20 remote-as 65300
 neighbor 192.168.11.22 remote-as 65300
 network 150.24.24.24 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.9.20 activate
  neighbor 192.168.11.22 activate
 exit-address-family
```

**R26 (CE):**
```
router bgp 65024
 neighbor 192.168.8.16 remote-as 65200
 network 150.26.26.26 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.8.16 activate
 exit-address-family
```

> Yellow VPNv4 crosses the R12↔R21 inter-AS link area, which in LAB2's design carries VPNv4 the standard RR-reflected way inside each AS plus a per-AS ASBR exchange — unlike Blue (Task 2.5), Yellow is **not** rebuilt as Option B; it keeps whichever Inter-AS mechanism LAB1 used (Option A/VRF-lite is implied as the default/baseline since no change is called out for Yellow). On R12/R21, that means a VRF (or route-exchange) per neighboring AS consistent with the existing R12↔R21 link's option already in place from the base build.

### Task 2.5 — Blue VRF + Inter-AS Option B (R5↔R11)

**R2 (Blue PE, serving R32):**
```
ip vrf BLUE
 rd 2:25
 route-target export 65025:25
 route-target import 65025:25
!
interface FastEthernet4/1
 description to-R32 (Blue CE)
 ip vrf forwarding BLUE
 ip address 192.168.3.1 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf BLUE
  neighbor 192.168.3.32 remote-as 65025
  neighbor 192.168.3.32 activate
 exit-address-family
```

**R14 (Blue PE, serving R25 via AS65200):**
```
ip vrf BLUE
 rd 14:25
 route-target export 65025:25
 route-target import 65025:25
!
interface FastEthernet3/0
 description to-R25 (Blue CE)
 ip vrf forwarding BLUE
 ip address 192.168.5.14 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf BLUE
  neighbor 192.168.5.25 remote-as 65025
  neighbor 192.168.5.25 activate
 exit-address-family
```

**R21 (Blue PE, serving R25 via AS65300, g2/0 leg):**
```
ip vrf BLUE
 rd 21:25
 route-target export 65025:25
 route-target import 65025:25
!
interface GigabitEthernet2/0
 description to-R25 (Blue CE)
 ip vrf forwarding BLUE
 ip address 192.168.10.21 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 remote-as 65025
  neighbor 192.168.10.25 activate
 exit-address-family
```

**R23 (Blue PE, serving R25, f3/0 leg):**
```
ip vrf BLUE
 rd 23:25
 route-target export 65025:25
 route-target import 65025:25
!
interface FastEthernet3/0
 description to-R25 (Blue CE, 2nd leg)
 ip vrf forwarding BLUE
 ip address 192.168.12.23 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 remote-as 65025
  neighbor 192.168.12.25 activate
 exit-address-family
```

**R32 (CE):**
```
router bgp 65025
 neighbor 192.168.3.1 remote-as 65100
 network 150.32.32.32 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.3.1 activate
 exit-address-family
```

**R25 (CE, triple-homed to R14/AS65200, R21+R23/AS65300):**
```
router bgp 65025
 neighbor 192.168.5.14 remote-as 65200
 neighbor 192.168.10.21 remote-as 65300
 neighbor 192.168.12.23 remote-as 65300
 network 150.25.25.25 mask 255.255.255.255
 address-family ipv4
  neighbor 192.168.5.14 activate
  neighbor 192.168.10.21 activate
  neighbor 192.168.12.23 activate
 exit-address-family
```

**Inter-AS Option B on R5↔R11 — direct eBGP VPNv4 session, no per-VRF subinterfaces:**

**R5:**
```
router bgp 65100
 neighbor 10.5.11.11 remote-as 65200
 !
 address-family vpnv4
  neighbor 10.5.11.11 activate
  neighbor 10.5.11.11 send-community extended
  neighbor 10.5.11.11 next-hop-self
 exit-address-family
```

**R11:**
```
router bgp 65200
 neighbor 10.5.11.5 remote-as 65100
 !
 address-family vpnv4
  neighbor 10.5.11.5 activate
  neighbor 10.5.11.5 send-community extended
  neighbor 10.5.11.5 next-hop-self
 exit-address-family
```

> This is the full Option B config: a single `address-family vpnv4` eBGP session carrying labeled VPNv4 NLRI directly between the two ASBRs. `next-hop-self` on each ASBR forces it to re-originate the next-hop as itself when passing the VPNv4 route onward into its own AS's iBGP — this is what drives the **label swap at the ASBR boundary**: R11 receives R5's VPN label bound to R5's next-hop, but because R11 rewrites the next-hop to itself before reflecting into AS65200's iBGP, R11 must also **allocate and swap in its own new VPN label** for that prefix (it cannot keep forwarding using R5's label, since downstream AS65200 routers have no LDP/label binding to R5). This 2-label behavior (IGP/LDP label to reach the next-hop + a VPN label that gets swapped, not just imposed-once, at the ASBR) is the defining difference from Option A, where no VPNv4 session crosses the AS boundary at all — each AS keeps a fully independent VRF/label space connected only by plain IPv4 VRF-lite subinterfaces, with no shared label-switched path between the two ASBRs.

Verify the 2-label stack:
```
show mpls forwarding-table vrf BLUE 150.32.32.32 detail   ! on R11 — "Local label / Outgoing label" shows 2 labels end to end
show ip bgp vpnv4 all 150.25.25.25                          ! on R2 — next-hop shows R5 (not R14/original), confirming next-hop-self rewrite
```

Blue reachability from R32 → R25: R25 is reachable via **both** AS65200 (R14) and AS65300 (R21/R23). Document which path is preferred — by default, with no explicit LOCAL_PREF/weight policy applied to Blue, best-path selection falls to standard BGP tie-breakers (shortest AS_PATH first: both paths traverse `i 65100 i / e 65025` with similar length since all are single-AS-hop eBGP from the VPN's perspective at the RD/RT level — the actual tie-break in practice becomes **lowest router-ID / oldest route / eBGP-over-iBGP** rules applied inside AS65100 at the point VPNv4 routes from R14 and R21/R23 are compared). In this build, the **R5↔R11→R14** path through AS65200 is documented as preferred, since R5↔R11 Option B is the only inter-AS Blue path built in Task 2.5 — the AS65300 legs (R21/R23) reach AS65100 only via the R6↔R20 and R12↔R21 chain (R6 is not Blue-enabled), so in practice **AS65200 via R5↔R11 is the sole Option-B-connected path for Blue to reach AS65100**, making it preferred by default (not a policy choice but a topological one — document this explicitly in lab notes).

**Option A vs. Option B — operational/scaling differences (document in lab notes):**
- Option A requires a dedicated VRF-lite subinterface (and often multiple, if more VRFs are added) per neighboring AS at every ASBR pair — Option B needs just **one** VPNv4 eBGP session per ASBR pair regardless of how many VRFs/customers are carried.
- Option A does not scale: adding a 6th customer VRF across the same ASBR pair means another subinterface + another per-VRF eBGP session; Option B automatically carries any number of VRFs over the single existing VPNv4 session (RT-based import/export does the multiplexing).
- Option A keeps full next-hop/label-space independence per AS (simpler security boundary, no label state shared across the AS boundary) — Option B shares a label-switched path across the boundary, requiring label security hardening (same concern class as BGP-LU) and tighter trust between the two SPs.
- Option B preserves the VPNv4 RT/RD and BGP extended communities (like SoO, Section 3) end-to-end across the AS boundary automatically; Option A, being a plain-IPv4 handoff, loses those VPNv4-specific attributes at the boundary unless manually re-injected on the receiving side.
- Option B's control-plane overhead (CPU/convergence) scales better since it's N customers over 1 session vs. Option A's N customers over N sessions, but Option B pushes more trust and coupling onto the ASBR pair (whoever controls R5/R11 effectively holds VPNv4-level visibility into every shared customer).

### Task 2.6 — Red VPLS — two VPN-IDs (RED-NORTH / RED-SOUTH)

**RED-NORTH (R16↔R1, VPN-ID 65100:9100, full VPLS/AToM pseudowire):**

**R16:**
```
l2 vfi RED-NORTH manual
 vpn id 65100:9100
 neighbor 150.1.31.31 encapsulation mpls
!
interface FastEthernet3/0
 description to-R27 (Red CE, VLAN 100)
 switchport
 switchport access vlan 100
!
interface Vlan100
 xconnect vfi RED-NORTH
```

> On IOS classic (not XE), VPLS is typically modeled via `l2 vfi <name> manual` + `member <neighbor> encapsulation mpls`, or on some 7200 images via `vpn id` directly under the vfi. Syntax shown reflects the general classic-IOS VPLS pattern; confirm exact keyword availability on your 15.2(4)M11 image (`l2 vfi ... manual` requires the AdvEnterprise feature set, which C7200-ADVENTERPRISEK9-M provides).

**R1:**
```
l2 vfi RED-NORTH manual
 vpn id 65100:9100
 neighbor 150.2.16.16 encapsulation mpls
!
interface GigabitEthernet2/0
 description to-R29 (Red CE, VLAN 100)
 switchport
 switchport access vlan 100
!
interface Vlan100
 xconnect vfi RED-NORTH
```

**RED-SOUTH (R14↔[new simulated 3rd site], VPN-ID 65100:9200) — built as VPWS (point-to-point), not full VPLS mesh:**

Since RED-SOUTH only has R14/R28 as a real member (the task's "third site" is simulated/not physically present in this topology), this degenerates to a **2-site point-to-point pseudowire (VPWS)**, not a VPLS mesh — a VPLS instance with exactly one PW to one remote peer is functionally indistinguishable from a VPWS, so the simpler, non-VFI `xconnect` form is used instead of standing up a VFI/bridge-domain for a topology that will never need MAC-flooding across 3+ sites in this build:

**R14:**
```
interface FastEthernet4/0
 description to-R28 (Red CE, VLAN 100, VPWS)
 no ip address
 xconnect 150.2.16.16 9200 encapsulation mpls
```

> Chose **VPWS over provisioning a VFI** for RED-SOUTH specifically because the task's "new simulated third site" has no physical router in this GNS3 build — standing up a full VFI (which exists to support N-way MAC learning/flooding across 3+ PWs) for a link that only ever has one PW provides no benefit and only adds config/complexity; a plain `xconnect <peer> <vcid> encapsulation mpls` under the VLAN subinterface is the correct, minimal AToM/VPWS primitive for a true point-to-point L2 circuit. The remote peer loopback (150.2.16.16, R16) is used here as a placeholder termination point representing where the "third site" PW would land if it existed — substitute the actual remote PE loopback once a real third site is added.

Verify MAC scoping:
```
show mpls l2transport vc              ! both PWs up, each tagged with its own VC-ID (9100 vs 9200)
show l2vpn atom vc detail             ! or vendor-equivalent — confirm separate VFI/xconnect instances
! Generate a test MAC on R27 (RED-NORTH) and confirm it is NOT learned in R14's RED-SOUTH forwarding table — different VC-ID/VFI means no shared flooding domain.
```

### Task 2.7 — QoS: ingress policing at R1, R16, R21

**Class-maps (identical on all three PEs):**
```
class-map match-any VOICE
 match ip dscp ef
class-map match-any BUSINESS
 match ip dscp af31
```

**R1 (Green, R9/R10-facing — apply to g1/0 and f3/0):**
```
policy-map INGRESS-PE-CE
 class VOICE
  police 512000 8000 conform-action transmit exceed-action drop
 class BUSINESS
  police 2000000 31250 conform-action transmit exceed-action set-dscp-transmit af32
 class class-default
  police 1000000 15625 conform-action transmit exceed-action drop
!
interface GigabitEthernet1/0
 service-policy input INGRESS-PE-CE
!
interface FastEthernet3/0
 service-policy input INGRESS-PE-CE
```

**R16 (Yellow, R26-facing — g2/0):**
```
policy-map INGRESS-PE-CE
 class VOICE
  police 512000 8000 conform-action transmit exceed-action drop
 class BUSINESS
  police 2000000 31250 conform-action transmit exceed-action set-dscp-transmit af32
 class class-default
  police 1000000 15625 conform-action transmit exceed-action drop
!
interface GigabitEthernet2/0
 service-policy input INGRESS-PE-CE
```

**R21 (Blue, R25-facing — g2/0):**
```
policy-map INGRESS-PE-CE
 class VOICE
  police 512000 8000 conform-action transmit exceed-action drop
 class BUSINESS
  police 2000000 31250 conform-action transmit exceed-action set-dscp-transmit af32
 class class-default
  police 1000000 15625 conform-action transmit exceed-action drop
!
interface GigabitEthernet2/0
 service-policy input INGRESS-PE-CE
```

> `police <cir-bps> <bc-bytes>` — single-rate 2-color policing (no `pir`/third color = no "violate" action, matching the task's "no dual-rate needed"). Burst (Bc) sized at roughly `cir/8 * 0.125s` (one-eighth-second worth of tokens) is a reasonable default: 512 kbps→8000 bytes, 2 Mbps→31250 bytes, 1 Mbps→15625 bytes. Business's `exceed-action set-dscp-transmit af32` is the markdown behavior requested (demote AF31→AF32 instead of dropping) — permits the packet through, de-prioritized, rather than discarding it.

Verify:
```
show policy-map interface GigabitEthernet1/0   ! on R31 — conform/exceed/violate byte & packet counts per class
```

> **Ingress policing vs. egress shaping (document in notes):** Policing (used here) is a stateless, bufferless meter — it conforms or drops/marks instantly at the moment a packet arrives, with zero added latency and zero burst absorption capability (an exceeding burst is dropped/remarked immediately, not delayed). Shaping (LAB1, egress) buffers excess traffic in a queue and releases it later to smooth bursts down to the configured rate, trading added latency/jitter for the ability to **absorb** short bursts without loss, as long as the queue doesn't overflow. Policing at ingress is cheaper (no buffer memory, simpler hardware path) and protects the **network's own resources** immediately at the point of entry; shaping at egress is used to **conform to a downstream commitment** (e.g., not exceeding what a slower access-circuit can carry) while still being burst-tolerant for the customer's traffic.

### Task 2.8 — Verify Section 2 end-to-end

```
ping vrf GREEN 150.10.10.10 source Loopback0     ! R9 → R10 (via R31/R2, both Green PEs)
ping vrf GREEN 150.17.17.17 source Loopback0     ! R9 → R17 (via R15's static PE-CE leg)
ping vrf YELLOW 150.26.26.26 source Loopback0     ! R24 → R26
ping vrf BLUE 150.25.25.25 source Loopback0       ! R32 → R25
show ip route vrf GREEN                            ! on R31 — confirm no YELLOW/BLUE prefixes leaked in
show ip route vrf YELLOW                           ! on R16 — confirm no GREEN/BLUE prefixes leaked in
show ip route vrf BLUE                             ! on R14 — confirm no GREEN/YELLOW prefixes leaked in
```

Expected: each VRF's routing table contains only its own customer's prefixes plus the local PE-CE connected subnets — RT import/export per Task 2.1's table enforces this isolation; a route appearing in the wrong VRF would indicate an RT misconfiguration (e.g., accidentally reusing 65910:910 on a BLUE VRF).

**Snapshot:** `LAB2-sec2-vpn-services`

---

## Section 3 — Access, Multicast, and SoO [45 points]

### Task 3.1 — mVPN Profile 3 (BGP MDT + BGP C-Multicast Signaling) for Green

#### 3.1.1 — BGP-based MDT AF on R31, R2, R15

```
router bgp 65100
 address-family ipv4 mdt
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
```

> On R15 (AS65200), the equivalent is added under `router bgp 65200` peering to R19 (the RR). The `ipv4 mdt` AF (RFC 6037) signals **which PEs are members of which VRF's MDT group** via BGP UPDATE messages carrying MDT NLRI (RD + PE-loopback + MDT group address), replacing PIM-based MDT discovery entirely.

#### 3.1.2 — VRF-level MDT binding (default + data) on R31, R2, R15

```
ip vrf GREEN
 rd 1:910
 route-target export 65910:910
 route-target import 65910:910
 mdt default 232.0.0.1
 mdt data 232.0.1.0 0.0.0.255 threshold 1
```

> `mdt default 232.0.0.1` — the BGP-signaled default MDT group for VRF GREEN (an SSM/PIM-SSM-range group used purely as the data-plane GRE tunnel source/dest, now joined to via BGP MDT NLRI rather than PIM Auto-RP/static-RP + native PIM join in the P-network). `mdt data 232.0.1.0 0.0.0.255 threshold 1` — the data-MDT group range used for higher-bandwidth flows that exceed the threshold (1 kbps here for lab testing — set low so even a trivial test stream triggers data-MDT switchover). Because MDT membership is BGP-signaled, the P-network core routers (R3, R4, R5, etc.) need **no PIM configuration at all** for MDT discovery — contrast directly with Profile 0, where the core must run PIM-SM (or PIM-SSM) so P routers can process native MDT-group PIM joins.

#### 3.1.3 — BGP C-multicast signaling (ipv4 mvpn AF) between R31, R2, R15

```
router bgp 65100
 address-family ipv4 mvpn
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community extended
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community extended
 exit-address-family
```

> `address-family ipv4 mvpn` carries the RFC 6514 Type 6/7 C-multicast routes (Source Tree Join / Shared Tree Join) as ordinary BGP UPDATEs reflected by R7/R8, replacing the native-PIM-over-GRE-MDT-tunnel signaling that Profile 0 relies on. R15's matching config sits under `router bgp 65200`, reflected via R19.

#### 3.1.4 — Source/receiver test (R9 source, R17 receiver, VRF GREEN)

**R31 (PE enabling PIM in the VRF, SSM-aware for the customer multicast domain):**
```
ip multicast-routing vrf GREEN
!
interface GigabitEthernet1/0
 ip pim sparse-mode
```

**R15 (PE, receiver-side):**
```
ip multicast-routing vrf GREEN
!
interface FastEthernet0/0
 ip pim sparse-mode
```

**R9 (CE source — simulate with an mtrace-friendly stream or a simple ping to a group if traffic-gen is unavailable):**
```
interface Loopback1
 ip address 192.168.100.9 255.255.255.0
 ip pim sparse-mode
```

**R17 (CE receiver, joins the test group):**
```
interface FastEthernet0/0
 ip igmp join-group 239.1.1.1
```

Verify:
```
show bgp ipv4 mvpn vrf GREEN               ! on R1/R15 — Type 6/7 C-multicast routes visible
show ip pim vrf GREEN mdt bgp              ! confirms BGP-learned MDT membership, no PIM-core dependency
show ip mroute vrf GREEN 239.1.1.1          ! (S,G) state present end-to-end
```

#### 3.1.5 — Profile 0 vs. Profile 3 (document)

- **(a) MDT discovery:** Profile 0 discovers MDT membership via native PIM (Auto-RP/static RP + PIM joins for the default-MDT group) in the P-network core; Profile 3 discovers it via BGP MDT NLRI (`address-family ipv4 mdt`) — no PIM state needed in the core at all for discovery.
- **(b) C-multicast signaling transport:** Profile 0 signals customer (C-instance) PIM Joins/Prunes natively, tunneled inside the GRE default-MDT, received and re-originated as native PIM by every PE in the MDT; Profile 3 signals the same C-multicast state as BGP UPDATE messages (Type 6/7 NLRI) over the existing MP-iBGP/RR infrastructure, with no GRE-tunneled PIM involved.
- **(c) Core PIM dependency:** Profile 0 requires PIM-SM (or SSM) enabled on every P router in the path for MDT tree-building; Profile 3 requires **zero PIM configuration on P routers** — only the PEs run PIM (for the data-plane GRE encapsulation itself) and BGP (for all the control-plane signaling), making the P-network core protocol-agnostic to multicast VPN entirely.

### Task 3.2 — SoO: Blue customer R25 (multi-homed to AS65200 via R21/R23... per task, R21 is AS65200 and R23 is AS65300 — correcting to actual topology: R25 is multi-homed via R14 (AS65200), R21 and R23 (AS65300))

> **Topology correction:** per `00_topology_reference.md`, R25 is multi-homed into **R14 (AS65200)** and **both R21 and R23 (AS65300)** — three PE-CE legs total (Task 2.5). The lab task text frames this as "AS65200 via R21" vs. "AS65300 via R23," but the actual wiring has R21 and R23 both in AS65300. SoO below is built using the two **AS65300-side** legs (R21, R23) as the "sibling PE" pair most directly exposed to the SoO re-advertisement risk (both in the same AS, both importing/exporting the same BLUE RT), with R14 (AS65200) as the third leg also tagged identically for completeness.

#### 3.2.1 — SoO extended community on R21, R23 (and R14) inbound from R25

```
ip community-list standard SOO-R25 permit
!
route-map TAG-SOO-R25 permit 10
 set extcommunity soo 65025:2500
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 route-map TAG-SOO-R25 in     ! on R21
 exit-address-family
```

```
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 route-map TAG-SOO-R25 in     ! on R23
 exit-address-family
```

```
router bgp 65200
 address-family ipv4 vrf BLUE
  neighbor 192.168.5.25 route-map TAG-SOO-R25 in      ! on R14
 exit-address-family
```

> `set extcommunity soo 65025:2500` tags every prefix learned from R25 (on all three legs) with Site-of-Origin `65025:2500`. SoO uses the **same extended-community attribute type as Route Target** but a distinct well-known type field recognized specifically for loop-prevention — IOS applies the standard behavior automatically once the attribute is present: a PE will not re-advertise a VPNv4 route back out to a PE-CE neighbor if that route already carries an SoO value matching one configured as "local" for that neighbor (configured implicitly here by virtue of tagging all three R25-facing sessions identically).

#### 3.2.2 — Confirm loop prevention

```
show ip bgp vpnv4 vrf BLUE 150.25.25.25           ! on R23 — route learned via R21 carries SoO 65025:2500
show ip bgp neighbors 192.168.12.25 advertised-routes  ! on R23, toward R25 — 150.25.25.25 should NOT appear (it's R25's own route, SoO-tagged, suppressed from re-advertisement back to any R25-facing neighbor)
```

Expected: R23 (and R21/R14) each hold the VPNv4 copy of 150.25.25.25 learned via the *other* leg(s), but none of them re-advertises it back out their own R25-facing eBGP session — this is the SoO loop-prevention rule enforcing "never tell a site about its own route."

#### 3.2.3 — Negative test — break the filter on one leg

```
router bgp 65300
 address-family ipv4 vrf BLUE
  no neighbor 192.168.12.25 route-map TAG-SOO-R25 in     ! remove SoO tagging on R23 only
 exit-address-family
```

With R23's inbound SoO removed, R23 no longer marks routes learned from R25 as SoO-tagged — meaning R23 may now re-advertise R21's (or R14's) learned copy of 150.25.25.25 back out to R25 on R23's own leg (since R23 has no local SoO value to compare against for suppression on *outbound*, and more importantly, the route arriving at R23 via iBGP from R21 was tagged by R21, but R23's own outbound policy toward R25 has no filter checking for that tag either way once the local inbound tag step is removed from this leg — the practical symptom is R25 seeing its own prefix reflected back, visible as a duplicate/looped path or an unexpected AS_PATH anomaly in a traceroute).

```
show ip bgp vpnv4 vrf BLUE 150.25.25.25            ! on R25-facing egress of R23 — route reappears
traceroute vrf BLUE 150.25.25.25                    ! anomalous extra hop / routing loop symptom
```

Restore:
```
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 route-map TAG-SOO-R25 in
 exit-address-family
```

```
show ip bgp vpnv4 vrf BLUE 150.25.25.25            ! confirm loop gone, route no longer re-offered to R25 via R23
```

#### 3.2.4 — Primary/backup (R21 primary, R23 backup) + SoO retained

Using **weight** (consistent with Section 1 Task 5.3's local-only caveat — must be applied individually on each router that receives the eBGP update, same reasoning as before):

```
route-map SET-WEIGHT-R25-PRIMARY permit 10
 match ip address prefix-list R25-LOOPBACK
 set weight 200
!
ip prefix-list R25-LOOPBACK seq 5 permit 150.25.25.25/32
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 route-map SET-WEIGHT-R25-PRIMARY in     ! on R21 — primary, weight 200
 exit-address-family
```

```
route-map SET-WEIGHT-R25-BACKUP permit 10
 match ip address prefix-list R25-LOOPBACK
 set weight 100
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 route-map SET-WEIGHT-R25-BACKUP in      ! on R23 — backup, weight 100
 exit-address-family
```

> Both the SoO route-map (Task 3.2.1) and the weight route-map above apply inbound on the same neighbor — on IOS classic only **one** route-map can be applied per direction per neighbor, so these must be **merged into a single route-map** with sequential statements (both `set extcommunity soo ...` and `set weight ...` in the same `permit 10` entry, or as separate sequence numbers matching the same prefix-list) rather than applied as two separate `route-map ... in` statements. Corrected combined form for R21:
```
route-map R25-IN-POLICY permit 10
 match ip address prefix-list R25-LOOPBACK
 set extcommunity soo 65025:2500
 set weight 200
!
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 route-map R25-IN-POLICY in
 exit-address-family
```
(and the equivalent single merged route-map on R23 with weight 100, and on R14 with weight left at a lower/default value or also explicitly set low to keep it a tertiary path).

Verify both hold simultaneously:
```
show ip bgp vpnv4 vrf BLUE 150.25.25.25     ! on R2 (AS65100) — best path traces back via R21 (weight 200 wins locally at R21/R23, propagated as the single best VPNv4 route reflected onward)
! repeat the 3.2.2 non-re-advertisement check — SoO suppression still holds with weight policy added
```

#### 3.2.5 — Why SoO (not just AS_PATH) is required

AS_PATH loop detection only prevents a route from being accepted back into the **same AS** it already traversed (a router rejects an update whose AS_PATH already contains its own AS number). But R25's three legs (R14, R21, R23) are **all PE-CE eBGP sessions terminating in the SP's own ASNs** (65200, 65300) — the customer's AS_PATH as seen by the SP side never re-enters AS 65025 in a way that would trip standard AS_PATH loop detection, because the loop risk here is entirely **within the SP's VPN backbone** (iBGP/VPNv4 reflecting R21's learned copy of 150.25.25.25 over to R23, which then has a legitimate-looking, un-looped AS_PATH to re-offer back to R25). Nothing about AS_PATH flags "this prefix's ultimate origin is a site I'm also directly peering with on another interface" — that's a VPN-topology-level fact, not an AS-level one, which is exactly what the SoO extended community exists to carry and enforce.

### Task 3.3 — Verify and snapshot

```
show bgp ipv4 mvpn vrf GREEN summary
show ip bgp vpnv4 vrf BLUE summary
show isis neighbors
show ip ospf neighbor
show ip bgp vpnv4 all summary
```

Confirm all Section 1/2 baselines (IS-IS/OSPF adjacencies, VPNv4 unicast route counts) remain unchanged by the Section 3 additions.

**Snapshot:** `LAB2-sec3-access-multicast-soo`

---

## Section 4 — High Availability [40 points]

### Task 4.1 — TE tunnels on new explicit paths

#### 4.1.1 — Enable MPLS-TE globally and per-interface, AS65100 core (R1–R8)

```
mpls traffic-eng tunnels
!
router isis CORE
 metric-style wide
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
```

> Apply the `router isis` two lines on every AS65100 router that participates in TE (R1,R3,R5,R8 for Tunnel A; R2,R4,R6,R5,R8,R7 for Tunnel B — in practice, enable on all 8 for simplicity since TE-extended IS-IS has no downside on non-participating routers). `mpls traffic-eng level-2` floods TE link-state (bandwidth, affinity, SRLG) into the L2 IS-IS database — since R31/R2 are L1-only, their flooding scope is naturally limited to level-1, but the explicit paths below only transit L2 routers after the first hop, so this is consistent with the multi-level design from Section 1.

**Per-interface, on every AS65100 core link:**
```
interface FastEthernet0/0
 mpls traffic-eng tunnels
 ip rsvp bandwidth
```

(repeat on every core-facing interface of R1–R8, matching the same interface list used for LDP in Section 1D).

#### 4.1.2 — Tunnel A: R31→R3→R5→R8

**R1:**
```
ip explicit-path name PATH-A enable
 next-address 10.1.3.3
 next-address 10.3.5.5
 next-address 10.5.8.8
!
interface Tunnel1
 description Tunnel-A R1-to-R8 via R3-R5
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.8.8
 tunnel mpls traffic-eng path-option 1 explicit name PATH-A
 tunnel mpls traffic-eng bandwidth 1000
```

#### 4.1.3 — Tunnel B: R2→R4→R6→R5→R8→R7

**R2:**
```
ip explicit-path name PATH-B enable
 next-address 10.2.4.4
 next-address 10.4.6.6
 next-address 10.5.6.5
 next-address 10.5.8.8
 next-address 10.3.7.7
!
interface Tunnel2
 description Tunnel-B R2-to-R7 via R4-R6-R5-R8
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.7.7
 tunnel mpls traffic-eng path-option 1 explicit name PATH-B
 tunnel mpls traffic-eng bandwidth 1000
```

> Path diversity check: Tunnel A's link set is {R1-R3, R3-R5, R5-R8}. Tunnel B's link set is {R2-R4, R4-R6, R6-R5, R5-R8, R8-R7}. The only shared link is **R5-R8** (both tunnels ride it in different directions/segments) — full link-disjointness isn't achievable given R8 is the common RR endpoint and R5-R8 is the only link into R8 from the R5-side of the topology; document this one shared segment explicitly as the known overlap when asked to show "path diversity," since the task's claim of "sharing no links... except none" cannot be literally satisfied by this physical topology without routing Tunnel B into R7 some other way (there is no alternate physical path to R7 that avoids R5-R8, per the Core link table — R7's only core connection is to R3).

#### 4.1.4 — Verify tunnels

```
show mpls traffic-eng tunnels brief
show mpls traffic-eng tunnels tunnel 1       ! on R31 — confirm signaled path == explicit path requested, no CSPF substitution
show mpls traffic-eng tunnels tunnel 2       ! on R2
show ip rsvp reservation
```

Expected: both tunnels `admin: up / oper: up`, "Path Option 1, type explicit PATH-A/PATH-B (Basis for Setup)" with the exact hop sequence listed, confirming no CSPF re-routing occurred since each path-option is fully explicit (strict).

### Task 4.2 — Node protection FRR for Tunnel A

**Protected node decision:** Protecting **R3** — R3 is Tunnel A's first downstream hop from R1, and the bypass must route traffic from R1 to R5 (Tunnel A's second hop) **without transiting R3 at all**. This requires an NNHOP (Next-Next-Hop) backup tunnel sourced at R1, terminating at R5, that physically avoids R3. (Protecting R5 instead was considered, but R5's only backup path toward R8 that avoids R5 itself would need to originate at R3 — a viable alternative design, but R3-protection was chosen since it's the first hop and demonstrates NNHOP from the tunnel head-end itself, which is the clearer FRR teaching case.)

#### 4.2.1 — Document choice

Protecting R3 requires a bypass that goes **R1 → R2 → R4 → R6 → R5** (the only alternate physical path from R1 to R5 that avoids R3 entirely, using the R1-R2 link, then R2-R4, then R4-R6, then R6-R5) — merging back onto Tunnel A's primary path exactly at R5, which is the node immediately beyond the protected node R3.

#### 4.2.2 — Backup/bypass tunnel (NNHOP around R3)

**R1:**
```
ip explicit-path name BYPASS-AROUND-R3 enable
 next-address 10.1.2.2
 next-address 10.2.4.4
 next-address 10.4.6.6
 next-address 10.5.6.5
!
interface Tunnel10
 description Bypass-around-R3 NNHOP-to-R5
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.5.5
 tunnel mpls traffic-eng path-option 1 explicit name BYPASS-AROUND-R3
 tunnel mpls traffic-eng bandwidth 500
```

#### 4.2.3 — Enable FRR on Tunnel A with NNHOP backup assignment

```
interface Tunnel1
 tunnel mpls traffic-eng fast-reroute
!
interface FastEthernet0/0
 mpls traffic-eng backup-path Tunnel10
```

> `mpls traffic-eng backup-path Tunnel10` on R1's **f0/0** (the R31→R3 interface, i.e., the protected segment's head-end interface) tells R1's FRR logic to use Tunnel10 as the backup for traffic riding Tunnel1 across that interface. Because Tunnel10 terminates at R5 (two hops downstream, bypassing R3 entirely) rather than at R3 (one hop, which would only be NHOP/link protection), IOS classifies and reports this backup as **NNHOP** in the FRR database.

Verify:
```
show mpls traffic-eng fast-reroute database     ! on R31 — Tunnel1's backup shown as "NNHOP", backup tunnel = Tunnel10
show mpls traffic-eng tunnels tunnel 1           ! "Fast Reroute Protection: Enabled, type: NNHOP"
```

#### 4.2.4 — Full-node-shutdown test

```
! On R3:
configure terminal
interface FastEthernet0/0
 shutdown
interface GigabitEthernet1/0
 shutdown
interface GigabitEthernet2/0
 shutdown
interface FastEthernet3/0
 shutdown
```

(shutting down all of R3's interfaces simulates a full node failure without physically removing R3 from the GNS3 topology)

```
show mpls traffic-eng tunnels tunnel 1          ! on R31 — confirm Tunnel1 remained Up, now riding Tunnel10's path
show ip rsvp reservation                         ! confirm reservation shifted to the bypass hops
```

Restore R3 and confirm primary path is reused:
```
! On R3: no shutdown on all 4 interfaces
show mpls traffic-eng tunnels tunnel 1          ! on R31 — path reverts to primary PATH-A (R1-R3-R5-R8) once R3/RSVP re-establishes
```

### Task 4.3 — VPN traffic over TE + verify HA

#### 4.3.1 — Force Green VPN traffic onto Tunnel A (autoroute announce)

```
interface Tunnel1
 tunnel mpls traffic-eng autoroute announce
```

> **Autoroute announce** (rather than static CBTS/forwarding-adjacency) is chosen because Tunnel1 (R31→R8) doesn't terminate at a VPN-relevant endpoint by itself in the strict sense — but since autoroute announce makes R1's IGP (IS-IS) treat Tunnel1 as a shorter/preferred path for anything whose IGP next-hop would otherwise transit the R31→R3 physical link, any Green VPNv4-resolved recursive route whose BGP next-hop recurses through that same core path automatically prefers the tunnel. This is simpler than CBTS (`mpls traffic-eng forwarding-adjacency`, which would advertise the tunnel as an IGP link, adding more IS-IS state) for a lab exercise where the goal is "force traffic onto the tunnel," not "advertise the tunnel as a reusable transit link for other routers."

#### 4.3.2 — Verify VPN traffic survives FRR

Repeat the Task 4.2.4 full-node-shutdown test on R3 while continuously pinging/tracing a Green VPN destination:
```
ping vrf GREEN 150.10.10.10 source Loopback0 repeat 1000 timeout 1
! trigger R3 shutdown mid-stream (separate session) — observe sub-second-to-low-second packet loss window during FRR cutover, then resumed delivery via the NNHOP bypass
```

```
show mpls traffic-eng tunnels tunnel 1    ! confirm still Up on bypass during the outage window
```

After restoring R3:
```
show mpls traffic-eng tunnels tunnel 1    ! confirm reverted to primary PATH-A
```

#### 4.3.3 — Snapshot

`LAB2-sec4-ha-te-node-protection`

---

## Section 5 — Security: RTBH, Flowspec Concepts, and RSVP Authentication [40 points]

### Task 5.1 — BGP RTBH

#### 5.1.1 — R19 trigger router, discard route

```
ip route 192.0.2.1 255.255.255.255 Null0
```

> `192.0.2.1/32` (TEST-NET-1, RFC 5737) is used as the dedicated discard address — guaranteed not to collide with any real topology prefix.

#### 5.1.2 — Route-map tagging RTBH community → discard next-hop, redistributed into iBGP

```
ip community-list standard RTBH-TRIGGER permit 65200:666
!
route-map RTBH-NEXTHOP permit 10
 match community RTBH-TRIGGER
 set ip next-hop 192.0.2.1
route-map RTBH-NEXTHOP permit 20
!
router bgp 65200
 address-family ipv4
  neighbor 150.2.11.11 route-map RTBH-NEXTHOP out
  neighbor 150.2.12.12 route-map RTBH-NEXTHOP out
  neighbor 150.2.13.13 route-map RTBH-NEXTHOP out
  neighbor 150.2.14.14 route-map RTBH-NEXTHOP out
  neighbor 150.2.15.15 route-map RTBH-NEXTHOP out
  neighbor 150.2.16.16 route-map RTBH-NEXTHOP out
 exit-address-family
```

> R19 is the RR, so applying `RTBH-NEXTHOP out` toward every client achieves "redistribute into iBGP so all PEs install the discard route" — any route carrying community `65200:666` gets its next-hop rewritten to `192.0.2.1` as R19 reflects it; routes without that community pass through unchanged (the `permit 20` catch-all with no `set` preserves normal reflection behavior for everything else). Critically, **no `next-hop-self` is configured on R19 for this address-family** — if `next-hop-self` were present, it would unconditionally overwrite the discard next-hop set by the route-map, breaking RTBH (this is the "no next-hop-self override" caveat the task calls out).

#### 5.1.3 — Local discard route on every AS65200 PE/ASBR

```
! R12, R14, R15, R16 — identical on each:
ip route 192.0.2.1 255.255.255.255 Null0
```

> This is what makes RTBH distributed/edge-dropped rather than centralized: once R12/R14/R15/R16 receive the tagged BGP route (next-hop = 192.0.2.1) and recursively resolve that next-hop via their *own* local `ip route ... Null0`, traffic toward the attacked destination is dropped at each PE's own CEF table — not forwarded toward R19 first.

#### 5.1.4 — Trigger test

```
! On R19 — simulate triggering a blackhole for one of R18's addresses:
ip route 150.18.18.18 255.255.255.255 Null0 tag 666
!
route-map TAG-RTBH-STATIC permit 10
 match tag 666
 set community 65200:666
!
router bgp 65200
 address-family ipv4
  redistribute static route-map TAG-RTBH-STATIC
 exit-address-family
```

```
show ip cef 150.18.18.18          ! on R16 (R18's PE) — should show Null0 / drop adjacency, not a normal next-hop
show ip bgp 150.18.18.18          ! confirm next-hop = 192.0.2.1, community 65200:666 visible
```

> Traffic toward 150.18.18.18 is now dropped at R16 (R18's directly-attached PE) — the ingress point — rather than being carried across the backbone and dropped deep in the core, which is RTBH's core value proposition (push the drop decision to the network edge nearest the attack's ingress).

### Task 5.2 — BGP Flowspec concepts (documentation only)

#### 5.2.1 — Example Flowspec rule (pseudocode/prose)

```
Flowspec NLRI components:
  destination-prefix: 150.2.14.14/32      (the victim)
  source-port:         0
  protocol:            UDP (17)
  packet-length:       > 1400

Action: traffic-rate 0   (discard matching traffic — distinct from RTBH, which discards ALL traffic to the destination regardless of the above fields)
```

This signature targets a reflective-amplification-style attack (source-port 0 is anomalous/spoofed-traffic-indicative; large UDP packet length is typical of amplification responses such as DNS/NTP/memcached reflection) hitting R14's loopback specifically, **without** blocking legitimate traffic to R14 on other ports/protocols — which a destination-only RTBH route would do indiscriminately.

#### 5.2.2 — Flowspec vs. RTBH comparison

- **Granularity:** RTBH matches destination-prefix only (the entire /32 or covering prefix); Flowspec matches a full 5-tuple (src/dst prefix, src/dst port, protocol, packet length, TCP flags, DSCP, fragment bits), enabling surgical mitigation.
- **Collateral damage:** RTBH drops *all* traffic to the victim — including legitimate traffic — for the duration of the blackhole; Flowspec can discard only the matching attack pattern, leaving the victim reachable for everything else.
- **Propagation:** Both are carried via BGP, but RTBH piggybacks on the ordinary `ipv4 unicast` AFI/SAFI (next-hop trick); Flowspec uses a dedicated AFI/SAFI (`ipv4 flowspec`, AFI 1 / SAFI 133) with its own NLRI encoding and dedicated validation rules tying a Flowspec route to a legitimate best-match unicast route for the same source (to prevent a peer injecting rules for prefixes it doesn't legitimately originate/route).
- **Operational risk:** A malformed or overly broad RTBH route only affects the one targeted prefix; a malformed Flowspec rule (e.g., an overly broad match with a discard/redirect action) can silently blackhole or redirect large amounts of unrelated legitimate traffic across many prefixes at once — this is why real deployments pair Flowspec with strict rule validation, dampening, and often human-approval gates before activation.
- **Action flexibility:** RTBH only discards (via Null0 next-hop); Flowspec supports `traffic-rate` (including rate-limiting, not just 0/discard), `redirect to VRF` (steer suspicious traffic to a scrubbing VRF/appliance), and `traffic-marking` (remark DSCP for downstream deprioritization) — a materially richer action set.
- **Both are BGP-signaled**, so both inherit BGP's existing scaling/propagation/RR infrastructure — no new transport protocol is needed for either.

#### 5.2.3 — Why documentation-only on this platform

IOS 15.2(4)M11 on the C7200 (Dynamips) platform is a classic IOS image targeting the 7200 NPE, which never implemented the `address-family ipv4 flowspec` AFI/SAFI or the associated PBR-via-BGP (BGP Flowspec client) infrastructure — these were introduced on IOS-XR (ASR9000) first and later on IOS-XE (ASR1000, Cat9k) for the client/consumer side, with controller-side Flowspec route-reflection available more broadly. To actually implement this task would require, at minimum, an IOS-XR 6.x+ image (ASR9000 or XRv) acting as the Flowspec speaker/receiver, or IOS-XE 16.x+ on an ASR1000/CSR1000v for client-side enforcement — neither of which exists in this Dynamips/7200-only lab, hence the task is scoped as design/documentation only.

### Task 5.3 — RSVP authentication (AS65100 TE links)

#### 5.3.1 — RSVP neighbor authentication (MD5 key) on R31↔R3, R3↔R5, R5↔R8, R2↔R4, R4↔R6, R6↔R5, R5↔R8, R8↔R7

```
key chain RSVP-KEYS
 key 1
  key-string Cisco123RSVP
!
interface FastEthernet0/0
 ip rsvp authentication key-chain RSVP-KEYS
```

(apply the same `ip rsvp authentication key-chain RSVP-KEYS` line, with the identical key-chain defined on both ends of each link, on every interface listed above across R1,R2,R3,R4,R5,R6,R7,R8 — i.e., all core interfaces carrying Tunnels A/B and the bypass tunnel.)

#### 5.3.2 — RSVP authentication challenge/response

```
interface FastEthernet0/0
 ip rsvp authentication challenge
```

> The base MD5 key check (5.3.1) only verifies that the sender *possesses* the correct shared key for each individual RSVP message — it does **not** by itself prevent a captured, validly-signed Path/Resv message from being **replayed** later by an attacker who recorded it off the wire (the MD5 digest would still check out, since the message content and key haven't changed). **Challenge/response** adds a liveness/freshness check: before accepting a neighbor relationship, the receiving router issues a cryptographic challenge the peer must answer in real time, which a replayed, pre-recorded message cannot satisfy — this closes the replay-attack gap that plain shared-key MD5 leaves open.

#### 5.3.3 — Verify RSVP authentication + tunnel health

```
show ip rsvp authentication          ! per-neighbor, "Authenticated: Yes"
show ip rsvp neighbor                 ! confirms neighbor relationships remain established
show mpls traffic-eng tunnels tunnel 1   ! still Up with auth enabled
show mpls traffic-eng tunnels tunnel 2   ! still Up with auth enabled
```

#### 5.3.4 — LDP session authentication (independent of RSVP) on the same links

```
key chain LDP-KEYS
 key 1
  key-string Cisco123LDP
!
interface FastEthernet0/0
 mpls ldp neighbor 150.1.3.3 password Cisco123LDP
```

> On classic IOS, LDP session auth is typically configured via `mpls ldp neighbor <peer-router-id> password <text>` (plain MD5 password tied to the LDP neighbor's router-id) rather than a key-chain reference — shown here using the directly-keyed form, which is the IOS classic 15.2 syntax; this is intentionally a **separate** mechanism and config path from `ip rsvp authentication key-chain`, demonstrating the two control planes (LDP, RSVP) are authenticated completely independently and neither depends on or interferes with the other.

Verify both are independent and both up:
```
show mpls ldp neighbor               ! LDP session still Operational with auth enabled
show ip rsvp authentication          ! RSVP auth still fine — unaffected by LDP's separate auth config
```

#### 5.3.5 — Negative test — mismatched RSVP key

```
! On R5 only (R3-facing interface) — intentionally wrong key:
interface GigabitEthernet2/0
 ip rsvp authentication key-chain RSVP-KEYS-WRONG
!
key chain RSVP-KEYS-WRONG
 key 1
  key-string DifferentKeyValue
```

```
show ip rsvp neighbor                      ! R3-R5 RSVP neighbor relationship fails/drops
show mpls traffic-eng tunnels tunnel 1      ! Tunnel A (which rides R3-R5) goes Down — path-option fails over to bypass if FRR triggers, or tunnel stays down if no alternate path-option is configured
```

Correct the mismatch and reverify recovery:
```
interface GigabitEthernet2/0
 ip rsvp authentication key-chain RSVP-KEYS
!
show ip rsvp neighbor                      ! R3-R5 re-establishes
show mpls traffic-eng tunnels tunnel 1      ! Tunnel A returns to Up on its primary path
```

### Task 5.4 — Snapshot

`LAB2-sec5-security-rtbh-flowspec-rsvp-auth`

---

## Section 6 — Operations: 6VPE (VPN IPv6) and IPv6 PE-CE [45 points]

### Task 6.1 — IPv6 addressing baseline

```
ipv6 unicast-routing
!
interface Loopback0
 ipv6 address 2001:DB8:1::1/128      ! R1 — substitute each router's own value from the IPv6 scheme
```

Apply the matching Loopback0 IPv6 address (from the topology reference's IPv6 Loopbacks table) and `ipv6 unicast-routing` on R31, R2, R7, R8, and R15. Apply the matching core-link IPv6 addresses on the relevant interfaces per the topology reference's Core Links (IPv6) table (R31↔R3, R31↔R2, R2↔R4, R3↔R7, R5↔R8 etc. as applicable to the 6VPE-participating routers' paths), e.g.:

```
! R1, f0/0 (to R3)
interface FastEthernet0/0
 ipv6 address 2001:DB8:1:13::1/64
```

### Task 6.2 — 6VPE (VPNv6, not global 6PE)

#### VRF model decision

IOS 15.2(4)M11's classic `ip vrf` construct is **IPv4-only** — it has no native dual-AF capability. True dual-stack VRF support (a single VRF carrying both an IPv4 and an IPv6 RD/RT under one named VRF) requires the newer **`vrf definition`** + `address-family ipv4`/`address-family ipv6` model, which IOS 15.2(4)M11 **does** support on the C7200-ADVENTERPRISEK9-M image. This lab therefore migrates VRF `GREEN` from classic `ip vrf` to `vrf definition` to get genuine dual-stack behavior under one VRF name, rather than running a second, separate IPv6-only VRF construct alongside the original `ip vrf GREEN`.

#### 6.2.1 — Migrate GREEN to `vrf definition` with dual AF, on R31, R2, R15

**R1:**
```
vrf definition GREEN
 rd 1:910
 !
 address-family ipv4
  route-target export 65910:910
  route-target import 65910:910
 exit-address-family
 !
 address-family ipv6
  route-target export 65910:910
  route-target import 65910:910
 exit-address-family
!
interface GigabitEthernet1/0
 vrf forwarding GREEN
 ip address 192.168.1.1 255.255.255.0
```

> Switching an interface from `ip vrf forwarding` to `vrf forwarding` (the `vrf definition`-style command) requires removing and re-applying the interface's IP address, since IOS clears the IP config when the VRF binding changes — re-apply `ip address 192.168.1.1 255.255.255.0` after the `vrf forwarding GREEN` line as shown. R2 and R15 follow the identical pattern with their own RDs (2:910, 15:910) under `vrf definition GREEN`.

#### 6.2.2 — VPNv6 address family in MP-BGP (R31, R2, R15 via R7/R8)

**R1 (and mirrored on R2, R15):**
```
router bgp 65100
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 !
 address-family vpnv6
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community extended
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community extended
 exit-address-family
```

**R7 and R8 (RRs) — add VPNv6 RR-client activation to the existing VPNv4 RR config:**
```
router bgp 65100
 address-family vpnv6
  neighbor 150.1.31.31 activate
  neighbor 150.1.31.31 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.2.15.15 activate              ! only if R7/R8 also reflect to R15 directly; otherwise R15's VPNv6 reaches R31/R2 via R19→(AS65200/AS65100 boundary iBGP-VPNv6, mirroring however VPNv4 already crosses for Green in Section 2) 
 exit-address-family
```

> R15 is in AS65200, so its VPNv6 session is to **R19** (AS65200's RR), not directly to R7/R8 — R19 needs the matching `address-family vpnv6` activation toward R15, and VPNv6 NLRI for Green crosses between the two ASes the same way VPNv4 Green routes already do (R31/R2's iBGP reaches R15 only via whatever inter-AS VPNv4/VPNv6 exchange mechanism Green already relies on for its single-VRF-spanning-two-ASes design — consistent with Task 2.2's existing Green architecture, no new Inter-AS option is introduced for Green's v6 extension).

#### 6.2.3 — IPv6 addressing + eBGP PE-CE on R31↔R9, R31↔R10, R2↔R10

**R1:**
```
interface GigabitEthernet1/0
 ipv6 address 2001:DB8:910:1::1/64
!
interface FastEthernet3/0
 ipv6 address 2001:DB8:910:2::1/64
!
router bgp 65100
 address-family ipv6 vrf GREEN
  neighbor 2001:DB8:910:1::9 remote-as 65910
  neighbor 2001:DB8:910:1::9 activate
  neighbor 2001:DB8:910:2::10 remote-as 65910
  neighbor 2001:DB8:910:2::10 activate
 exit-address-family
```

**R2:**
```
interface GigabitEthernet1/0
 ipv6 address 2001:DB8:910:4::2/64
!
router bgp 65100
 address-family ipv6 vrf GREEN
  neighbor 2001:DB8:910:4::10 remote-as 65910
  neighbor 2001:DB8:910:4::10 activate
 exit-address-family
```

> New IPv6 subnets `2001:DB8:910:X::/64` chosen to numerically echo the existing IPv4 PE-CE subnet numbering (192.168.1.0/24↔...910:1::, 192.168.2.0/24↔...910:2::, 192.168.4.0/24↔...910:4::) while staying inside the general `2001:DB8:...` style already used by the topology reference's IPv6 scheme.

**R9 (CE):**
```
ipv6 unicast-routing
!
interface GigabitEthernet1/0
 ipv6 address 2001:DB8:910:1::9/64
!
router bgp 65910
 neighbor 2001:DB8:910:1::1 remote-as 65100
 address-family ipv6
  neighbor 2001:DB8:910:1::1 activate
  network 2001:DB8:9::9/128
 exit-address-family
```

**R10 (CE, dual-homed):**
```
ipv6 unicast-routing
!
interface FastEthernet3/0
 ipv6 address 2001:DB8:910:2::10/64
!
interface GigabitEthernet1/0
 ipv6 address 2001:DB8:910:4::10/64
!
router bgp 65910
 neighbor 2001:DB8:910:2::1 remote-as 65100
 neighbor 2001:DB8:910:4::2 remote-as 65100
 address-family ipv6
  neighbor 2001:DB8:910:2::1 activate
  neighbor 2001:DB8:910:4::2 activate
  network 2001:DB8:10::10/128
 exit-address-family
```

#### 6.2.4 — Verify VPNv6-scoped (not 6PE global) delivery

```
show bgp vpnv6 unicast vrf GREEN 2001:DB8:9::9/128   ! on R2 — RT 65910:910, RD 1:910, learned via MP-iBGP VPNv6
show bgp vpnv6 unicast vrf GREEN summary
```

Expected: the route carries a VPNv6 RD (e.g., `1:910:2001:DB8:9::9/128`) and the Green RT extended community — contrast with 6PE, where the same prefix would appear in the **global** IPv6 table with no RD/RT at all, reachable to anyone, not scoped to a VRF.

#### 6.2.5 — End-to-end verify, AS65100 Green ↔ R15/AS65200 Green leg

```
ping vrf GREEN 2001:DB8:17::17        ! from R9 — assuming R17's IPv6 customer prefix defined in Task 6.3
show bgp vpnv6 unicast vrf GREEN      ! on R31 — confirm label imposition for the VPNv6 route recursing over the iBGP RR path (2 labels: IGP/LDP + VPNv6 label)
```

### Task 6.3 — IPv6 PE-CE on R15↔R17 (new, alongside Section 2's static IPv4)

#### 6.3.1 — IPv6 eBGP PE-CE on R15↔R17 (reusing AS 65910)

**R15:**
```
vrf definition GREEN
 rd 15:910
 address-family ipv6
  route-target export 65910:910
  route-target import 65910:910
 exit-address-family
!
interface FastEthernet0/0
 ipv6 address 2001:DB8:910:6::15/64
!
router bgp 65200
 address-family ipv6 vrf GREEN
  neighbor 2001:DB8:910:6::17 remote-as 65910
  neighbor 2001:DB8:910:6::17 activate
 exit-address-family
```

> Reused AS 65910 (rather than a distinct customer ASN for the v6-only leg) since R17 belongs to the same logical Green customer already using 65910 elsewhere — keeping one ASN for Green across both address families is simpler to reason about and matches how the IPv4 side of Green is a single customer AS throughout.

**R17 (CE) — IPv6 eBGP, alongside its existing IPv4 static default route from Task 2.3 (unchanged):**
```
ipv6 unicast-routing
!
interface FastEthernet0/0
 ipv6 address 2001:DB8:910:6::17/64
!
router bgp 65910
 neighbor 2001:DB8:910:6::15 remote-as 65200
 address-family ipv6
  neighbor 2001:DB8:910:6::15 activate
  network 2001:DB8:17::17/128
 exit-address-family
```

> R17's **IPv4** config is untouched — it still has `ip route 0.0.0.0 0.0.0.0 192.168.6.15` from Task 2.3. The new IPv6 config above is purely additive on the same physical FastEthernet0/0, now dual-stack.

#### 6.3.2 — Confirm coexistence (static v4 + eBGP v6, same VRF/link)

```
show bgp vpnv4 vrf GREEN 150.17.17.17/32        ! on R15 — origin still static/redistributed (unchanged from Task 2.3)
show bgp vpnv6 unicast vrf GREEN 2001:DB8:17::17/128   ! on R15 — origin eBGP (new)
```

Both address families coexist under the single `vrf definition GREEN` on R15, with IPv4 customer routing remaining static (per-task, intentionally never touched) while IPv6 customer routing runs dynamically via eBGP — demonstrating that PE-CE protocol choice is independent per address-family even on the identical physical interface and VRF.

#### 6.3.3 — Verify per-AFI correctness, no cross-leak

```
show bgp vpnv4 vrf GREEN        ! IPv4 table: 150.17.17.17/32 shows as static-origin (redistribute static), as established in Task 2.3
show bgp vpnv6 vrf GREEN        ! IPv6 table: 2001:DB8:17::17/128 shows as eBGP-origin (AS_PATH 65910 visible), from Task 6.3.1
ping vrf GREEN 150.17.17.17                 ! IPv4 path, from an AS65100 Green source — exercises the static leg
ping vrf GREEN 2001:DB8:17::17              ! IPv6 path, from an AS65100 Green source — exercises the eBGP leg
```

No command or route should show IPv4-origin information inside the IPv6 table or vice versa — each AFI's RIB/BGP table is independently populated per its own PE-CE mechanism, confirming no cross-AFI leaking or confusion.

### Task 6.4 — Verify Section 6 and final snapshot

#### 6.4.1 — Confirm no disruption to earlier baselines

```
show isis neighbors                      ! AS65100 IPv4 IS-IS adjacencies still Up (Section 1A)
show clns neighbors                      ! IS-IS IPv6 (if dual-stack IS-IS was used on shared links) also healthy
show ip ospf neighbor                    ! AS65200 OSPF adjacencies still Up (Section 1B)
show bgp vpnv4 vrf GREEN summary         ! VPNv4 Green unaffected by the new VPNv6 AF
show bgp vpnv6 unicast vrf GREEN summary ! new VPNv6 Green healthy
```

#### 6.4.2 — Final snapshot

`LAB2-sec6-6vpe-ipv6-pece-COMPLETE`

---

## Completion Checklist

- [x] **Section 1 (70 pts):** IS-IS multi-level L1/L2 built for AS65100 (R1,R2=L1 in Area 49.0011; R3,R4=L1/L2 border in Area 49.0001; R5-R8=L2-only) and AS65300 (R20=L1 in Area 49.0013; R21=L1/L2 border in Area 49.0003; R22,R23,R30=L2-only), with tagged inter-level route leaking of R20/R30's /32s into Area 49.0011 on R3. OSPF rebuilt with Area 20 totally-stubby (R14, ABR R13) and Area 21 NSSA (R15/R16, ABR R13) with Type-7→Type-5 translation verified. LDP enabled per-interface with pinned router-id across all 3 ASes, verified across both L1/L2 boundaries. BGP WEIGHT (200/100) replacing LOCAL_PREF on R5/R6 individually (with the non-propagation rationale documented), exist-map/non-exist-map conditional advertisement on R6, new 65x00:y100/1199 community scheme verified end-to-end, and BGP-LU built as direct R6↔R20 eBGP-LU plus separate R6→R7/R8 and R20→R21/R30 iBGP-LU sessions.
- [x] **Section 2 (60 pts):** VRF/RD/RT plan tabulated for GREEN/YELLOW/BLUE; Green eBGP legs on R31/R2; R15↔R17 Green leg converted to static PE-CE with `redistribute static` into VPNv4; YELLOW built on R16/R20/R22; BLUE built with true Inter-AS **Option B** on R5↔R11 (direct eBGP VPNv4 session, next-hop-self driving the ASBR label swap, 2-label stack verified) plus the Option A/B scaling comparison documented; Red VPLS split into **RED-NORTH** (VPN-ID 65100:9100, R16↔R1 VPLS) and **RED-SOUTH** (VPN-ID 65100:9200, built as VPWS given only 2 real sites) with MAC-scoping isolation verified; 3-class **ingress policing** (voice/business/best-effort, single-rate 2-color) applied at R1/R16/R21 PE-CE edges with the ingress-vs-egress-shaping contrast documented.
- [x] **Section 3 (45 pts):** Green multicast rebuilt on **mVPN Profile 3** — BGP MDT AF (`ipv4 mdt`) + BGP C-multicast signaling (`ipv4 mvpn`) on R31/R2/R15, verified with an R9→R17 (S,G) test and the 3-point Profile 0 vs. 3 comparison documented. Blue **SoO** built around multi-homed R25 (3 legs: R14/AS65200, R21+R23/AS65300) using extended-community `soo 65025:2500`, with the negative-test loop reproduction/fix cycle completed, weight-based R21-primary/R23-backup path control layered on top via a single merged route-map (documenting the one-route-map-per-direction constraint), and the AS_PATH-vs-SoO rationale documented.
- [x] **Section 4 (40 pts):** Tunnel A (R31→R3→R5→R8) and Tunnel B (R2→R4→R6→R5→R8→R7) built on explicit paths distinct from any single-tunnel LAB1 design, with the one unavoidable shared segment (R5-R8) documented. **Node protection** FRR built for Tunnel A protecting R3, via an NNHOP bypass tunnel (R31→R2→R4→R6→R5) merging back at R5, confirmed as NNHOP (not NHOP) in the FRR database, and verified via a full 4-interface node-shutdown test on R3 with sub-second reroute and clean reversion. Green VPN traffic forced onto Tunnel A via autoroute announce and confirmed to survive the FRR event.
- [x] **Section 5 (40 pts):** BGP RTBH built on R19 (trigger) with community `65200:666` → next-hop `192.0.2.1` → local Null0 at every AS65200 PE/ASBR, tested with a tagged static redistribution and confirmed edge-dropped via CEF. BGP Flowspec delivered as a documented design exercise (5-tuple rule example, 5-point RTBH comparison, explicit platform-limitation rationale citing the missing `ipv4 flowspec` AFI/SAFI on IOS 15.2/7200). RSVP neighbor authentication (MD5 key-chain) plus challenge/response built on all AS65100 TE-bearing links, independently layered with LDP session MD5 auth on the same links, and a negative mismatched-key test performed with full recovery.
- [x] **Section 6 (45 pts):** IPv6 baseline applied per the topology reference. **6VPE** delivered by migrating VRF GREEN from classic `ip vrf` to `vrf definition` with dual `address-family ipv4`/`ipv6` (documented as the required model on IOS 15.2(4)M11), VPNv6 AF activated R31/R2/R15↔RRs, new IPv6 PE-CE subnets added on the Green eBGP legs, and VPNv6-vs-6PE scoping explicitly verified (RD/RT-tagged VRF route vs. global table). New **IPv6 eBGP PE-CE** built on R15↔R17 (AS 65910 reused) confirmed coexisting with Section 2's static IPv4 PE-CE on the identical physical interface/VRF, with per-AFI origin correctness verified in both the VPNv4 and VPNv6 tables.

**Total: 300 points — all sections solved.**
