# SR Workbook — Full Solutions

Complete IOS-XR configurations for all Segment Routing exercises (SR-EX01 through SR-EX05).

> **Note:** These are the SOLUTIONS. Attempt the exercises in `SR_Workbook.md` first, then check your work here.

---

## Topology Reference

```
                 CE1 (AS65001, dual-homed)
                /   \
           Gi? /     \ Gi?
              /       \
        [R1 PE]       [R2 PE]        OSPF Area 0 + LDP domain (R1,R2) + R3 boundary
      172.16.1.1     172.16.2.2
      Gi0 |   \Gi3    Gi3/   | Gi0
          |    \      /      |
          |     \    /       |
   R1<->R3|   R1<->R2        | R2<->R4
          |                  |
      Gi0 |                  | Gi0
       [R3 boundary]======[R4 P]======[R5 P]======[R6 PE]===CE2 (AS65002)
      172.16.3.3         172.16.4.4  172.16.5.5  172.16.6.6
      OSPF+IS-IS         IS-IS+SR    IS-IS+SR    IS-IS+SR
      LDP+SR
```

### Link / Interface Matrix

| Link      | A-end (intf)   | B-end (intf)   | Subnet         | A IP        | B IP        |
|-----------|----------------|----------------|----------------|-------------|-------------|
| R1↔R3     | R1 Gi0         | R3 Gi0         | 10.0.13.0/24   | 10.0.13.1   | 10.0.13.3   |
| R1↔R2     | R1 Gi3         | R2 Gi3         | 10.0.12.0/24   | 10.0.12.1   | 10.0.12.2   |
| R2↔R4     | R2 Gi0         | R4 Gi0         | 10.0.24.0/24   | 10.0.24.2   | 10.0.24.4   |
| R3↔R6     | R3 Gi1         | R6 Gi1         | 10.0.36.0/24   | 10.0.36.3   | 10.0.36.6   |
| R3↔R4     | R3 Gi2         | R4 Gi2         | 10.0.34.0/24   | 10.0.34.3   | 10.0.34.4   |
| R3↔R5     | R3 Gi3         | R5 Gi3         | 10.0.35.0/24   | 10.0.35.3   | 10.0.35.5   |
| R4↔R5     | R4 Gi1         | R5 Gi1         | 10.0.45.0/24   | 10.0.45.4   | 10.0.45.5   |
| R5↔R6     | R5 Gi2         | R6 Gi2         | 10.0.56.0/24   | 10.0.56.5   | 10.0.56.6   |

> NIC mapping: NIC2=Gi0/0/0/0 (Gi0), NIC3=Gi0/0/0/1 (Gi1), NIC4=Gi0/0/0/2 (Gi2), NIC5=Gi0/0/0/3 (Gi3).
> Throughout, `GigabitEthernet0/0/0/N` is abbreviated by its NIC label in the matrix; the configs use full IOS-XR interface names.

### Loopbacks / Router-IDs

| Node | Loopback0     | IS-IS NET                      | Prefix-SID (algo 0) | SR label |
|------|---------------|--------------------------------|---------------------|----------|
| R1   | 172.16.1.1/32 | (OSPF/LDP only)                | 101 (via MS)        | 16101    |
| R2   | 172.16.2.2/32 | (OSPF/LDP only)                | 102 (via MS)        | 16102    |
| R3   | 172.16.3.3/32 | 49.0001.1720.1600.3003.00      | 3                   | 16003    |
| R4   | 172.16.4.4/32 | 49.0001.1720.1600.4004.00      | 4                   | 16004    |
| R5   | 172.16.5.5/32 | 49.0001.1720.1600.5005.00      | 5                   | 16005    |
| R6   | 172.16.6.6/32 | 49.0001.1720.1600.6006.00      | 6                   | 16006    |

- **SRGB:** 16000–23999 (label = 16000 + index).
- Mapping Server (on R3) advertises SIDs for the LDP/OSPF PEs: R1=101 (16101), R2=102 (16102).

---

# SR-EX01 — Base IGP, LDP, IS-IS Core, SR Enable, Classic LFA

## Task 1: OSPF Area 0 on R1, R2, R3

**Question:** Bring up OSPF process 1, Area 0, on the R1/R2/R3 access domain. Loopback0 and all interconnect links (R1↔R2, R1↔R3, R2↔R4-side toward R2, R3↔R1) participate. Loopbacks passive; point-to-point network type on links.

**Solution:**

```
! ===== R1 =====
router ospf 1
 router-id 172.16.1.1
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
   network point-to-point
  !
  interface GigabitEthernet0/0/0/3
   network point-to-point
  !
 !
!

! ===== R2 =====
router ospf 1
 router-id 172.16.2.2
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/3
   network point-to-point
  !
 !
!

! ===== R3 =====
router ospf 1
 router-id 172.16.3.3
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
   network point-to-point
  !
 !
!
```

**Explanation:** OSPF covers only the "classic" access domain (R1, R2, R3-facing-R1). R2↔R4 is an IS-IS core link, so it is *not* in OSPF. `network point-to-point` avoids DR/BDR election on the /24 p2p links and speeds convergence. Loopbacks are passive so no adjacency forms over them but the /32 is advertised.

**Verification:**

```
RP/0/RP0/CPU0:R1# show ospf neighbor

Neighbors for OSPF 1
Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.2.2      1     FULL/  -        00:00:38    10.0.12.2       GigabitEthernet0/0/0/3
172.16.3.3      1     FULL/  -        00:00:31    10.0.13.3       GigabitEthernet0/0/0/0

RP/0/RP0/CPU0:R1# show ospf route | include 172.16
172.16.2.2/32  ...  via 10.0.12.2, GigabitEthernet0/0/0/3
172.16.3.3/32  ...  via 10.0.13.3, GigabitEthernet0/0/0/0
```

Expect two FULL neighbors on R1, and R1/R2/R3 loopbacks reachable across the OSPF domain.

---

## Task 2: LDP on R1, R2, R3

**Question:** Enable LDP so the OSPF access domain has a working label distribution plane. Use loopback for the LDP router-id; enable LDP on the access-domain interfaces only.

**Solution:**

```
! ===== R1 =====
mpls ldp
 router-id 172.16.1.1
 interface GigabitEthernet0/0/0/0
 !
 interface GigabitEthernet0/0/0/3
 !
!

! ===== R2 =====
mpls ldp
 router-id 172.16.2.2
 interface GigabitEthernet0/0/0/3
 !
!

! ===== R3 =====
mpls ldp
 router-id 172.16.3.3
 interface GigabitEthernet0/0/0/0
 !
!
```

**Explanation:** LDP runs on the R1↔R2 and R1↔R3 links (the OSPF-native links). R3's core-facing links (Gi1/Gi2/Gi3) will use SR, not LDP — those come in EX03 when we clean LDP off the IS-IS interfaces. Explicit `router-id` pinned to Loopback0 keeps the LSR-ID stable.

**Verification:**

```
RP/0/RP0/CPU0:R1# show mpls ldp neighbor brief
Peer               GR  Up Time    Discovery  Addresses  Labels
172.16.2.2:0       N   00:05:12       1          3         12
172.16.3.3:0       N   00:05:02       1          3         12

RP/0/RP0/CPU0:R1# show mpls ldp bindings 172.16.3.3/32
172.16.3.3/32
  Local Binding:  label: 24001
  Remote Binding: lsr: 172.16.3.3:0, label: ImpNull
```

Expect LDP neighbors R2 and R3 on R1, and label bindings for each OSPF-learned /32.

---

## Task 3: IS-IS Core on R3, R4, R5, R6 with NET-IDs

**Question:** Build the IS-IS core (`level-2-only`, single area `49.0001`) across R3, R4, R5, R6. Loopback0 passive, all core interconnects point-to-point, wide metrics, IPv4 unicast.

**Solution:**

```
! ===== R3 (boundary — also in OSPF) =====
router isis CORE
 is-type level-2-only
 net 49.0001.1720.1600.3003.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!

! ===== R4 =====
router isis CORE
 is-type level-2-only
 net 49.0001.1720.1600.4004.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
!

! ===== R5 =====
router isis CORE
 is-type level-2-only
 net 49.0001.1720.1600.5005.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!

! ===== R6 (PE) =====
router isis CORE
 is-type level-2-only
 net 49.0001.1720.1600.6006.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/2
  point-to-point
  address-family ipv4 unicast
  !
 !
!
```

**Interface participation recap:**
- R3: Gi1 (→R6), Gi2 (→R4), Gi3 (→R5). Gi0 (→R1) stays OSPF.
- R4: Gi0 (→R2), Gi1 (→R5), Gi2 (→R3).
- R5: Gi1 (→R4), Gi2 (→R6), Gi3 (→R3).
- R6: Gi1 (→R3), Gi2 (→R5).

> **Note on R2↔R4:** R2 is OSPF-only, R4 is IS-IS. This link is IS-IS on R4's side (Gi0) but there is no IS-IS peer on R2. In this topology R2↔R4 is used as an OSPF/LDP-reachable path only if R2 also runs IS-IS; per the exercise R2 is a pure OSPF PE, so R4 Gi0 has no adjacency and R2↔R4 reachability is provided through R3 redistribution in EX03. Keep R4 Gi0 in IS-IS for addressing consistency; it simply has no neighbor.

**Explanation:** `is-type level-2-only` — single flat L2 area, no L1 overhead. `metric-style wide` is mandatory for SR (narrow metrics can't carry SR sub-TLVs). Loopback passive advertises the /32 host route that becomes the prefix-SID anchor.

**Verification:**

```
RP/0/RP0/CPU0:R3# show isis adjacency
IS-IS CORE Level-2 adjacencies:
System Id      Interface                SNPA           State  Hold  Changed
R6             Gi0/0/0/1                *PtoP*         Up     23    00:02:11
R4             Gi0/0/0/2                *PtoP*         Up     27    00:02:10
R5             Gi0/0/0/3                *PtoP*         Up     25    00:02:09

RP/0/RP0/CPU0:R3# show isis topology
IS-IS CORE paths to IPv4 Unicast (Level-2) System Id  Metric  Next-Hop  Interface
R3              --
R4              10      R4        Gi0/0/0/2
R5              10      R5        Gi0/0/0/3
R6              10      R6        Gi0/0/0/1
```

Expect three L2 adjacencies on R3, and all core loopbacks in `show isis topology`.

---

## Task 4: SR Enable + Prefix-SID Configuration

**Question:** Enable Segment Routing MPLS in IS-IS with SRGB 16000–23999 and assign each core node its prefix-SID index (R3=3, R4=4, R5=5, R6=6) on Loopback0.

**Solution:**

```
! ===== Global SRGB (all SR nodes: R3,R4,R5,R6) =====
segment-routing
 global-block 16000 23999
!

! ----- R3 -----
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 3
  !
 !
!

! ----- R4 -----
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 4
  !
 !
!

! ----- R5 -----
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 5
  !
 !
!

! ----- R6 -----
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 6
  !
 !
!
```

**Explanation:** The **global-block** must be identical on every SR node so labels are consistent domain-wide (label = 16000 + index → R3=16003, R4=16004, R5=16005, R6=16006). `segment-routing mpls` under the IPv4 AF turns on the SR data plane in IS-IS. `prefix-sid index N` (not `absolute`) is best practice — it's SRGB-relative, so a future SRGB change reallocates automatically.

**Verification:**

```
RP/0/RP0/CPU0:R4# show isis segment-routing label table
Label   Prefix/Interface
16003   172.16.3.3/32
16004   172.16.4.4/32 (local)
16005   172.16.5.5/32
16006   172.16.6.6/32

RP/0/RP0/CPU0:R4# show mpls forwarding labels 16006
Local  Outgoing    Prefix          Outgoing     Next Hop        Bytes
Label  Label       or ID           Interface                    Switched
16006  16006       SR Pfx (172.16.6.6/32)  Gi0/0/0/1  10.0.45.5  0

RP/0/RP0/CPU0:R4# show segment-routing local-block inconsistencies
No inconsistencies found     <-- expect this
```

Expect a consistent label table on every node and no SRGB inconsistencies.

---

## Task 5: Classic (per-prefix) LFA

**Question:** Enable classic per-prefix LFA fast-reroute on the IS-IS core interfaces so a pre-computed backup next-hop protects each prefix.

**Solution:**

```
! Apply on each core node's transit interfaces (example R4).
router isis CORE
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix
  !
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix
  !
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   fast-reroute per-prefix
  !
 !
!
```

Repeat `fast-reroute per-prefix` under each IS-IS-enabled transit interface on R3, R4, R5, R6 (not on Loopback).

**Explanation:** Classic LFA computes a loop-free alternate that is a *directly connected* neighbor whose own shortest path to the destination does not traverse the protected link. It provides sub-50ms protection but only where a topologically valid LFA exists — rings and certain meshes leave coverage gaps (which TI-LFA in EX02 closes).

**Verification:**

```
RP/0/RP0/CPU0:R4# show isis fast-reroute 172.16.6.6/32
172.16.6.6/32
  via 10.0.45.5, Gi0/0/0/1, R5, SR-adj                (primary)
  FRR backup via 10.0.34.3, Gi0/0/0/2, R3             (LFA)
  P: No, TM: 20, LC: No, NP: No, D: No

RP/0/RP0/CPU0:R4# show isis fast-reroute summary
Prefixes reachable in L2
   Total     Protected   Unprotected   Percent
     4           3            1          75%
```

Expect a `FRR backup` next-hop listed for protected prefixes. Coverage <100% is normal for classic LFA.

---

# SR-EX02 — TI-LFA, Node Protection, SRLG

## Task 1: Enable TI-LFA

**Question:** Replace classic LFA with Topology-Independent LFA to guarantee 100% coverage using SR repair (segment) lists.

**Solution:**

```
! On every core node, per transit interface (example R5).
router isis CORE
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 !
 interface GigabitEthernet0/0/0/3
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 !
!
```

Apply the same pair of commands on all IS-IS transit interfaces of R3, R4, R5, R6.

**Explanation:** TI-LFA uses the SR label stack to steer traffic along the *post-convergence* path via the P-space/Q-space intersection, so it protects link/node/SRLG failures even where no classic LFA exists. It builds an explicit repair segment list (e.g., push the SID of a Q-node), giving guaranteed coverage independent of topology.

**Verification:**

```
RP/0/RP0/CPU0:R5# show isis fast-reroute summary
Prefixes reachable in L2
   Total     Protected   Unprotected   Percent
     4           4            0          100%          <-- 100% with TI-LFA

RP/0/RP0/CPU0:R5# show isis fast-reroute 172.16.3.3/32 detail
172.16.3.3/32
  via 10.0.35.3, Gi0/0/0/3, R3                        (primary)
  FRR backup via 10.0.45.4, Gi0/0/0/1, R4
    repair node(s): R4
    P node: R4        label(s): 16003
    TI-LFA (link-protecting)
```

Expect 100% protected and a repair with an explicit `P node`/`label(s)` segment list.

---

## Task 2: Node Protection Tiebreaker

**Question:** Prefer a backup path that protects against *node* failure of the primary next-hop, not just link failure.

**Solution:**

```
! On each core node, under IS-IS interface AF (example R4).
router isis CORE
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker node-protecting index 100
  !
 !
 interface GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker node-protecting index 100
  !
 !
!
```

**Explanation:** When multiple valid TI-LFA backups exist, tiebreakers pick the "best" one. `node-protecting index 100` ranks node-protecting backups highly (lower index = evaluated first / higher priority). A node-protecting backup avoids the failed neighbor entirely, so it survives that neighbor going down (stronger than link-only protection). SRLG-disjointness can be layered as an additional tiebreaker.

**Verification:**

```
RP/0/RP0/CPU0:R4# show isis fast-reroute 172.16.6.6/32 detail
172.16.6.6/32
  via 10.0.45.5, Gi0/0/0/1, R5                        (primary)
  FRR backup via 10.0.34.3, Gi0/0/0/2, R3
    P: Yes  NP: Yes                                    <-- NP: Yes = node-protecting chosen
    TI-LFA (node-protecting)
    repair node(s): R3   label(s): 16006
```

Expect `NP: Yes` (node-protecting) on protected prefixes where a node-protecting backup exists.

---

## Task 3: SRLG Configuration (value 100 on R3↔R5 and R4↔R5)

**Question:** Mark the R3↔R5 and R4↔R5 links as sharing risk group 100, and make FRR prefer SRLG-disjoint backups so both links are treated as one failure domain.

**Solution:**

```
! ===== R3 : R3<->R5 is Gi0/0/0/3 =====
srlg
 interface GigabitEthernet0/0/0/3
  name SRLG-100
  value 100
 !
!
router isis CORE
 interface GigabitEthernet0/0/0/3
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker srlg-disjoint index 90
  !
 !
!

! ===== R5 : R5<->R3 is Gi0/0/0/3, R5<->R4 is Gi0/0/0/1 =====
srlg
 interface GigabitEthernet0/0/0/3
  name SRLG-100
  value 100
 !
 interface GigabitEthernet0/0/0/1
  name SRLG-100
  value 100
 !
!
router isis CORE
 interface GigabitEthernet0/0/0/3
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker srlg-disjoint index 90
  !
 !
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker srlg-disjoint index 90
  !
 !
!

! ===== R4 : R4<->R5 is Gi0/0/0/1 =====
srlg
 interface GigabitEthernet0/0/0/1
  name SRLG-100
  value 100
 !
!
router isis CORE
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   fast-reroute per-prefix ti-lfa
   fast-reroute per-prefix tiebreaker srlg-disjoint index 90
  !
 !
!
```

**Explanation:** SRLG 100 tells IS-IS that R3↔R5 and R4↔R5 physically share fate (e.g., same conduit). The `srlg-disjoint` tiebreaker makes TI-LFA avoid selecting a backup that traverses any link in the same SRLG as the protected link — so protecting R3↔R5 will not fall back onto R4↔R5. Lower index (90) makes SRLG-disjointness the highest-priority tiebreaker here, above node-protecting (100).

**Verification:**

```
RP/0/RP0/CPU0:R5# show srlg interface GigabitEthernet0/0/0/3
Interface           SRLG Name     SRLG Values
Gi0/0/0/3           SRLG-100      100

RP/0/RP0/CPU0:R5# show isis fast-reroute 172.16.6.6/32 detail
172.16.6.6/32
  via 10.0.35.3, Gi0/0/0/3, R3                        (primary, SRLG 100)
  FRR backup via 10.0.56.6 ... (path avoiding SRLG 100 links)
    SRLG disjoint: Yes                                 <-- backup avoids SRLG 100
    TI-LFA (link/node/SRLG-protecting)

RP/0/RP0/CPU0:R5# show isis fast-reroute summary
   Total   Protected   SRLG-disjoint
     4         4            4
```

Expect SRLG 100 shown on the marked interfaces and `SRLG disjoint: Yes` on backups for prefixes reached via an SRLG-100 primary.

---

# SR-EX03 — Redistribution, Mapping Server, SR-Prefer, LDP Cleanup

## Task 1: R3 Mutual Redistribution (IS-IS ↔ OSPF)

**Question:** On the boundary R3, redistribute IS-IS (core) into OSPF (access) and OSPF into IS-IS so R1/R2 reach R4/R5/R6 and vice-versa. Tag routes and control with route-policy.

**Solution:**

```
! ===== R3 =====
route-policy ISIS-TO-OSPF
  set metric 100
  set metric-type type-1
  pass
end-policy
!
route-policy OSPF-TO-ISIS
  set tag 300
  pass
end-policy
!
router ospf 1
 redistribute isis CORE route-policy ISIS-TO-OSPF
!
router isis CORE
 address-family ipv4 unicast
  redistribute ospf 1 route-policy OSPF-TO-ISIS metric 20 level-2
 !
!
```

**Explanation:** R3 is the only ASBR/redistribution point, so no mutual-redistribution loop can form (single point). `metric-type type-1` makes external OSPF cost cumulative (internal + external) for better path selection. Tag 300 on OSPF→IS-IS routes lets you filter/troubleshoot and would block re-redistribution if a second boundary were added later.

**Verification:**

```
RP/0/RP0/CPU0:R1# show ospf route external | include 172.16
172.16.4.4/32  [110/...] via 10.0.13.3 (E1)
172.16.5.5/32  [110/...] via 10.0.13.3 (E1)
172.16.6.6/32  [110/...] via 10.0.13.3 (E1)

RP/0/RP0/CPU0:R6# show route 172.16.1.1
Routing entry for 172.16.1.1/32
  Known via "isis CORE", ... redistributed ..., tag 300
  Routing Descriptor Blocks
    10.0.36.3, from ..., via Gi0/0/0/1
```

Expect R1 to see R4/R5/R6 as OSPF externals, and R6 to see R1/R2 as IS-IS redistributed routes tagged 300.

---

## Task 2: Mapping Server on R3 (SIDs for R1, R2)

**Question:** R1 and R2 are LDP/OSPF-only and cannot advertise their own prefix-SIDs. Configure R3 as a Segment Routing Mapping Server (SRMS) to advertise prefix-SIDs on their behalf: R1=101, R2=102.

**Solution:**

```
! ===== R3 =====
segment-routing
 mapping-server
  prefix-sid-map
   address-family ipv4
    172.16.1.1/32 101
    172.16.2.2/32 102
   !
  !
 !
!
router isis CORE
 address-family ipv4 unicast
  segment-routing prefix-sid-map advertise-local
 !
!
```

**Explanation:** The SRMS lets non-SR nodes (R1, R2) be represented in the SR domain. R3 owns the mapping and floods it in IS-IS via `advertise-local`. Core nodes then install SR labels toward R1 (16101) and R2 (16102), enabling SR-to-LDP interworking at R3 (SR label imposed in the core, swapped to LDP/OSPF forwarding at the boundary).

**Verification:**

```
RP/0/RP0/CPU0:R3# show segment-routing mapping-server prefix-sid-map ipv4
Prefix            SID Index   Range   Flags
172.16.1.1/32     101         1
172.16.2.2/32     102         1

RP/0/RP0/CPU0:R4# show isis segment-routing prefix-sid-map active-policy
 IS-IS CORE active policy
 Prefix            SID Index   Range   Flags
 172.16.1.1/32     101         1
 172.16.2.2/32     102         1

RP/0/RP0/CPU0:R4# show mpls forwarding labels 16101
Local  Outgoing   Prefix                 Outgoing   Next Hop
16101  Pop        SR Pfx (172.16.1.1/32) ...        (toward R3)
```

Expect the mapping on R3 and the *active-policy* mapping learned on R4/R5/R6, with 16101/16102 programmed in MPLS forwarding.

---

## Task 3: SR-Prefer

**Question:** Where both an LDP label and an SR label exist for the same prefix (SR/LDP interworking zone), make SR the preferred label imposition.

**Solution:**

```
! ===== R3 (the SR/LDP interworking node) =====
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
 !
!
```

> If you also need SR preferred in the OSPF-learned world at the boundary, apply `sr-prefer` under whichever IGP/AF is providing the labels; in this lab SR lives in IS-IS, so the IS-IS AF is correct.

**Explanation:** By default IOS-XR may prefer LDP labels when both LDP and SR are available for a prefix. `sr-prefer` flips that so the SR label is used for forwarding, which is required for consistent end-to-end SR LSPs and correct TI-LFA behavior across the interworking boundary.

**Verification:**

```
RP/0/RP0/CPU0:R3# show mpls forwarding prefix 172.16.6.6/32
Local  Outgoing   Prefix                 Outgoing   Next Hop     Label Source
16006  16006      SR Pfx (172.16.6.6/32) Gi0/0/0/1  10.0.36.6    (SR)      <-- SR chosen

RP/0/RP0/CPU0:R3# show cef 172.16.6.6/32 detail | include SR|labels
   local label 16006  labels imposed {16006}   <-- SR label, not LDP
```

Expect the SR label (16xxx) selected over any LDP-assigned label for shared prefixes.

---

## Task 4: LDP Removal from IS-IS Interfaces

**Question:** Now that the core is fully SR, remove LDP from the IS-IS core interfaces so the core forwards purely on SR labels. Keep LDP only on the R1/R2/R3 access links.

**Solution:**

```
! ===== R3 : remove LDP from core-facing Gi1/Gi2/Gi3, keep on Gi0 (->R1) =====
mpls ldp
 no interface GigabitEthernet0/0/0/1
 no interface GigabitEthernet0/0/0/2
 no interface GigabitEthernet0/0/0/3
!

! ===== R4, R5, R6 : if any LDP was ever enabled on core links, remove entirely =====
! (In this lab R4/R5/R6 never ran LDP on core links; verify none is present.)
! Example if cleanup needed on R6:
mpls ldp
 no interface GigabitEthernet0/0/0/1
 no interface GigabitEthernet0/0/0/2
!
```

**Explanation:** With SR providing labels in the core (and the SRMS covering R1/R2), LDP on core links is redundant. Removing it eliminates a second label plane, simplifies troubleshooting, and prevents SR/LDP label races. LDP remains only on R1↔R2 and R1↔R3 (Gi0) where the classic PEs live and SR is injected via the mapping server.

**Verification:**

```
RP/0/RP0/CPU0:R3# show mpls ldp interface brief
Interface           Config   Enabled   Neighbors
Gi0/0/0/0           Yes      Yes       1            <-- access link, LDP still up (R1)
! Gi0/0/0/1, /2, /3 no longer listed

RP/0/RP0/CPU0:R3# show mpls forwarding | include Gi0/0/0/1
16006  16006  SR Pfx (172.16.6.6/32)  Gi0/0/0/1 ...   <-- pure SR on core link
```

Expect LDP only on Gi0 (access) at R3; core links carry SR labels exclusively, with no LDP neighbors on Gi1/Gi2/Gi3.

---

# SR-EX04 — Flexible Algorithm

## Task 1: Flex-Algo 128 — Delay-Optimized (min-delay)

**Question:** Define Flex-Algo 128 to compute paths minimizing measured link delay. Enable performance-measurement delay on core links so the metric is populated.

**Solution:**

```
! ===== On every participating node (R3,R4,R5,R6): define the algo identically =====
router isis CORE
 flex-algo 128
  metric-type delay
 !
!

! ===== Prefix-SID for algo 128 on Loopback0 (per node, unique index) =====
! R3
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid algorithm 128 index 128
  !
 !
!
! R4  -> index 129,  R5 -> index 130,  R6 -> index 131 (see Task 3)

! ===== Performance-measurement: delay probes on core links =====
performance-measurement
 interface GigabitEthernet0/0/0/1
  delay-measurement
 !
 interface GigabitEthernet0/0/0/2
  delay-measurement
 !
 interface GigabitEthernet0/0/0/3
  delay-measurement
 !
!
```

Apply the `performance-measurement ... delay-measurement` block to each node's core transit interfaces.

**Explanation:** A Flex-Algo definition (FAD) must be *identical* on all participating nodes (algo number + metric-type + constraints), otherwise nodes disagree and drop out. `metric-type delay` makes SPF use the dynamic delay metric supplied by performance-measurement, which actively probes one-way/round-trip delay per link. Each algo needs its own prefix-SID so the forwarding plane can distinguish algo-0 vs algo-128 paths.

**Verification:**

```
RP/0/RP0/CPU0:R3# show isis flex-algo 128
Flex-Algo 128:
  Definition Priority: 128   Metric-Type: delay   Enabled
  Participating nodes: R3, R4, R5, R6

RP/0/RP0/CPU0:R3# show performance-measurement interfaces
Interface       Delay(uSec)  State
Gi0/0/0/1       350          Up
Gi0/0/0/3       120          Up

RP/0/RP0/CPU0:R3# show isis route flex-algo 128 172.16.6.6/32
172.16.6.6/32  algo 128  via <lowest-delay path>  label 16xxx
```

Expect all four nodes participating in 128, measured delay values on interfaces, and a delay-optimized path/label for algo-128 prefixes.

---

## Task 2: Flex-Algo 129 — Affinity Exclude

**Question:** Define Flex-Algo 129 (IGP metric) that *excludes* links colored with affinity `RED`, then color the R4↔R5 link RED so 129 routes avoid it.

**Solution:**

```
! ===== Affinity bit-map names (identical on all nodes) =====
affinity-map
 name RED bit-position 0
!

! ===== Flex-Algo 129 definition (identical on all participants) =====
router isis CORE
 flex-algo 129
  metric-type igp
  affinity exclude-any RED
 !
!

! ===== Color the R4<->R5 link RED on both ends =====
! R4 : R4<->R5 = Gi0/0/0/1
router isis CORE
 interface GigabitEthernet0/0/0/1
  affinity flex-algo RED
 !
!
! R5 : R5<->R4 = Gi0/0/0/1
router isis CORE
 interface GigabitEthernet0/0/0/1
  affinity flex-algo RED
 !
!

! ===== Prefix-SID for algo 129 (per node — see Task 3) =====
! R3 index 132, R4 133, R5 134, R6 135
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid algorithm 129 index 132
  !
 !
!
```

**Explanation:** Affinities are named bit positions carried in an extended admin-group; the map must match across nodes. `affinity exclude-any RED` in the FAD prunes any RED link from algo-129's SPF topology. Coloring R4↔R5 RED forces algo-129 traffic between, say, R4 and R6 to detour (R4→R3→R6 or R4→R5 is excluded), while algo-0 and algo-128 still may use it.

**Verification:**

```
RP/0/RP0/CPU0:R4# show isis flex-algo 129
Flex-Algo 129:  Metric-Type: igp   Exclude-any: RED   Enabled

RP/0/RP0/CPU0:R4# show isis interface GigabitEthernet0/0/0/1 | include Affinity
  Affinity (flex-algo): RED

RP/0/RP0/CPU0:R4# show isis route flex-algo 129 172.16.6.6/32
172.16.6.6/32  algo 129  via 10.0.34.3 (R3) ...   <-- avoids RED R4<->R5 link
```

Expect the RED exclude in the FAD, the RED affinity on Gi0/0/0/1, and algo-129 paths that route around the RED link.

---

## Task 3: Per-Algo Prefix-SID Allocation

**Question:** Assign distinct, non-overlapping prefix-SID indices per node for algo 0, 128, and 129 so each algorithm has its own label plane.

**Solution:**

```
! Index plan (all within SRGB 16000-23999):
!   algo 0   : R3=3   R4=4   R5=5   R6=6      (labels 16003..16006)
!   algo 128 : R3=128 R4=129 R5=130 R6=131    (labels 16128..16131)
!   algo 129 : R3=132 R4=133 R5=134 R6=135    (labels 16132..16135)

! ===== R3 =====
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 3
   prefix-sid algorithm 128 index 128
   prefix-sid algorithm 129 index 132
  !
 !
!
! ===== R4 =====
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 4
   prefix-sid algorithm 128 index 129
   prefix-sid algorithm 129 index 133
  !
 !
!
! ===== R5 =====
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 5
   prefix-sid algorithm 128 index 130
   prefix-sid algorithm 129 index 134
  !
 !
!
! ===== R6 =====
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 6
   prefix-sid algorithm 128 index 131
   prefix-sid algorithm 129 index 135
  !
 !
!
```

**Explanation:** Each (prefix, algorithm) pair needs a unique SID so the forwarding plane can program a separate label per algo. Reusing an index across algos would collide. Keeping algo-0 low (3–6), algo-128 in the 128-block, and algo-129 in the 132-block makes the label math (16000+index) readable at a glance.

**Verification:**

```
RP/0/RP0/CPU0:R5# show isis segment-routing label table
Label   Prefix/Interface        Algo
16006   172.16.6.6/32           0
16131   172.16.6.6/32           128
16135   172.16.6.6/32           129
...

RP/0/RP0/CPU0:R5# show mpls forwarding labels 16131
16131  16131  SR Pfx (172.16.6.6/32) Algo 128  Gi... <lowest-delay nh>
```

Expect three separate labels per remote loopback (one per algo) with distinct next-hops where the algo constraints diverge.

---

## Task 4: Selective Participation (remove/add R4 in algo 128)

**Question:** Demonstrate selective participation: remove R4 from Flex-Algo 128, observe the topology heal around it, then add it back.

**Solution — remove R4 from 128:**

```
! ===== R4 : withdraw from algo 128 by removing its algo-128 prefix-SID
!            and (optionally) the FAD participation =====
router isis CORE
 interface Loopback0
  address-family ipv4 unicast
   no prefix-sid algorithm 128 index 129
  !
 !
 no flex-algo 128
!
```

**Solution — add R4 back to 128:**

```
! ===== R4 : re-participate =====
router isis CORE
 flex-algo 128
  metric-type delay
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid algorithm 128 index 129
  !
 !
!
```

**Explanation:** A node participates in a Flex-Algo only if it (a) has the matching FAD and (b) advertises an algo prefix-SID. Removing either drops it from that algo's topology — other nodes recompute algo-128 SPF *excluding* R4 as a transit/endpoint for 128, while R4 still fully participates in algo 0 and 129. This is how operators phase Flex-Algo rollout node-by-node without touching the base topology.

**Verification:**

```
! After removal:
RP/0/RP0/CPU0:R3# show isis flex-algo 128
Flex-Algo 128:  Participating nodes: R3, R5, R6      <-- R4 absent

RP/0/RP0/CPU0:R3# show isis route flex-algo 128 172.16.6.6/32
172.16.6.6/32 algo 128 via <path not transiting R4>

! After re-add:
RP/0/RP0/CPU0:R3# show isis flex-algo 128
Flex-Algo 128:  Participating nodes: R3, R4, R5, R6  <-- R4 back
```

Expect R4 to disappear from the algo-128 participant list (and be avoided) after removal, then reappear after re-adding — with algo 0 unaffected throughout.

---

# SR-EX05 — L3VPN over SR + Color/ODN Steering

## Task 1: BGP VPNv4 — R3 as Route Reflector

**Question:** Configure R3 as the VPNv4 route reflector with iBGP sessions to R1, R2 (OSPF PEs) and R6 (IS-IS PE). All in AS 65000, updates sourced from Loopback0.

**Solution:**

```
! ===== R3 (Route Reflector) =====
router bgp 65000
 bgp router-id 172.16.3.3
 address-family vpnv4 unicast
 !
 neighbor-group RR-CLIENTS
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
 neighbor 172.16.1.1
  use neighbor-group RR-CLIENTS
 !
 neighbor 172.16.2.2
  use neighbor-group RR-CLIENTS
 !
 neighbor 172.16.6.6
  use neighbor-group RR-CLIENTS
 !
!
```

**Explanation:** A single RR (R3) means each PE only needs one iBGP session, and the RR reflects VPNv4 routes between clients. `update-source Loopback0` pins the session endpoint to the SR-reachable /32. R3 sits at the OSPF/IS-IS boundary and is reachable from both R1/R2 (OSPF) and R6 (IS-IS), making it the natural RR.

**Verification:**

```
RP/0/RP0/CPU0:R3# show bgp vpnv4 unicast summary
Neighbor        Spk  AS   ...  St/PfxRcd
172.16.1.1      0  65000  ...  2
172.16.2.2      0  65000  ...  2
172.16.6.6      0  65000  ...  2
```

Expect all three iBGP VPNv4 sessions Established with prefixes received.

---

## Task 2: VRF CUSTOMER on R1, R2, R6 (RD / RT)

**Question:** Create VRF `CUSTOMER` on the three PEs with RD `65000:1` (per-PE loopback-based RD recommended) and import/export RT `65000:1`.

**Solution:**

```
! ===== VRF definition (identical RT; RD per-PE using router-id) =====
! ----- R1 -----
vrf CUSTOMER
 address-family ipv4 unicast
  import route-target
   65000:1
  !
  export route-target
   65000:1
  !
 !
!
router bgp 65000
 vrf CUSTOMER
  rd 172.16.1.1:1
  address-family ipv4 unicast
   redistribute connected
  !
 !
!

! ----- R2 -----
vrf CUSTOMER
 address-family ipv4 unicast
  import route-target
   65000:1
  !
  export route-target
   65000:1
  !
 !
!
router bgp 65000
 vrf CUSTOMER
  rd 172.16.2.2:1
  address-family ipv4 unicast
   redistribute connected
  !
 !
!

! ----- R6 -----
vrf CUSTOMER
 address-family ipv4 unicast
  import route-target
   65000:1
  !
  export route-target
   65000:1
  !
 !
!
router bgp 65000
 vrf CUSTOMER
  rd 172.16.6.6:1
  address-family ipv4 unicast
   redistribute connected
  !
 !
!
```

**Explanation:** RT `65000:1` on both import and export makes it a full-mesh VPN (all three PEs share routes). Using a per-PE RD (`<router-id>:1`) instead of a global `65000:1` keeps VPNv4 NLRIs unique per PE — essential when CE1 is dual-homed to R1 and R2 (the RR sees two distinct VPNv4 routes for the same customer prefix and can offer multipath/backup).

**Verification:**

```
RP/0/RP0/CPU0:R6# show vrf CUSTOMER detail | include "Route Distinguisher|Import|Export"
Route Distinguisher: 172.16.6.6:1
  Import VPN route-target communities: 65000:1
  Export VPN route-target communities: 65000:1

RP/0/RP0/CPU0:R6# show bgp vpnv4 unicast rd 172.16.1.1:1
   Network            Next Hop        ...
*>i<CE1 prefix>       172.16.1.1      ...
```

Expect the VRF with correct RD/RT and remote PE VPNv4 prefixes in the CUSTOMER table.

---

## Task 3: PE-CE eBGP + route-policy PASS-ALL

**Question:** Configure eBGP between the PEs and CEs — CE1 (AS 65001) dual-homed to R1/R2, CE2 (AS 65002) to R6 — inside VRF CUSTOMER, applying an inbound/outbound `PASS-ALL` policy (IOS-XR requires an explicit policy or all routes are dropped).

**Solution:**

```
! ===== route-policy (define on all PEs) =====
route-policy PASS-ALL
  pass
end-policy
!

! ----- R1 : CE1 on VRF interface, CE1 AS 65001 -----
router bgp 65000
 vrf CUSTOMER
  neighbor 10.100.1.1
   remote-as 65001
   address-family ipv4 unicast
    route-policy PASS-ALL in
    route-policy PASS-ALL out
    as-override
   !
  !
 !
!

! ----- R2 : CE1 (dual-homed), CE1 AS 65001 -----
router bgp 65000
 vrf CUSTOMER
  neighbor 10.100.2.1
   remote-as 65001
   address-family ipv4 unicast
    route-policy PASS-ALL in
    route-policy PASS-ALL out
    as-override
   !
  !
 !
!

! ----- R6 : CE2 AS 65002 -----
router bgp 65000
 vrf CUSTOMER
  neighbor 10.100.6.2
   remote-as 65002
   address-family ipv4 unicast
    route-policy PASS-ALL in
    route-policy PASS-ALL out
   !
  !
 !
!
```

**Explanation:** IOS-XR mandates an explicit inbound and outbound route-policy on every eBGP AF — with none, the session comes up but exchanges zero prefixes. `PASS-ALL` (`pass`) accepts everything. `as-override` on R1/R2 rewrites the CE1 AS (65001) in advertisements toward CE2 so CE1's *other* homing site doesn't reject its own routes as a loop (needed because CE1 is dual-homed to two PEs in the same customer AS). CE2 (single-homed) doesn't need it.

**Verification:**

```
RP/0/RP0/CPU0:R1# show bgp vrf CUSTOMER summary
Neighbor    Spk  AS     St/PfxRcd
10.100.1.1   0  65001   3

RP/0/RP0/CPU0:R6# show bgp vrf CUSTOMER
*> <CE1 prefix>   ... (learned via VPNv4 from R1/R2)
*> <CE2 prefix>   10.100.6.2 ...

! End-to-end data plane:
RP/0/RP0/CPU0:R6# ping vrf CUSTOMER <CE1 loopback> source <CE2 loopback>
!!!!!  Success rate is 100 percent
```

Expect CE routes learned on each PE, redistributed into VPNv4, and successful CE1↔CE2 reachability across the SR core.

### route-policy PASS-ALL (standalone reference)

```
route-policy PASS-ALL
  pass
end-policy
!
```

---

## Task 4: BGP Color + ODN for Flex-Algo Steering

**Question:** Use a BGP color extended community + On-Demand Next-hop (ODN) so that VPN traffic to a specific destination is automatically steered onto an SR Policy that follows Flex-Algo 128 (low-delay). Color 128 → ODN template → auto SR Policy to R6.

**Solution:**

```
! ===== On the head-end PE (e.g. R1) : ODN template that builds an
!       SR Policy toward the BGP next-hop using Flex-Algo 128 =====
segment-routing
 traffic-eng
  on-demand color 128
   dynamic
    metric
     type delay
    !
   !
   ! Steer onto Flex-Algo 128 constraints
   constraints
    segments
     dataplane mpls
    !
   !
  !
 !
!

! ===== Tag the routes with color 128 (set on the egress PE R6 export,
!       or inbound at head-end). Example: color set on R6 for CE2 routes =====
route-policy SET-COLOR-128
  set extcommunity color COLOR-128
  pass
end-policy
!
extcommunity-set opaque COLOR-128
  128
end-set
!
router bgp 65000
 vrf CUSTOMER
  address-family ipv4 unicast
   ! apply outbound toward VPNv4 so the color is attached on advertisement
   network <CE2 prefix> route-policy SET-COLOR-128
  !
 !
!

! ===== Head-end (R1) : accept colored VPNv4 and let ODN auto-instantiate =====
router bgp 65000
 address-family vpnv4 unicast
 !
 ! ODN triggers automatically when a received VPNv4 route carries color 128
 ! and no matching SR Policy exists; the on-demand color 128 template above
 ! builds a dynamic SR Policy to the route's next-hop (R6) via Flex-Algo 128.
!
```

**Explanation:** ODN ties a **color** community to an **on-demand SR Policy template**. When R1 receives a VPNv4 route (from R6) carrying color 128, and there is no existing SR Policy to that endpoint/color, R1 auto-instantiates one using the `on-demand color 128` template — here a dynamic path optimized for delay (aligning with Flex-Algo 128). The VPN traffic is then automatically steered onto that low-delay SR Policy without static per-destination policy configuration. This is the scalable way to map service intent (color) to transport behavior (Flex-Algo).

**Verification:**

```
RP/0/RP0/CPU0:R1# show bgp vpnv4 unicast <CE2 prefix> | include Color
    Color: 128

RP/0/RP0/CPU0:R1# show segment-routing traffic-eng policy color 128
SR-TE policy database
---------------------
Color: 128, End-point: 172.16.6.6
  Name: srte_c_128_ep_172.16.6.6
  Status: Admin: up  Operational: up  (auto-instantiated / ODN)
  Candidate-paths:
    Preference 100 (configuration) (active)
      Dynamic (valid)  Metric Type: DELAY
      SID[0]: 16131 (R6 algo-128 prefix-SID)

RP/0/RP0/CPU0:R1# show cef vrf CUSTOMER <CE2 prefix> detail | include policy|labels
   via SR policy srte_c_128_ep_172.16.6.6, labels imposed {16131 <vpn-label>}
```

Expect the received VPNv4 route to carry Color 128, an auto-instantiated SR-TE policy (color 128, endpoint R6) with a delay-optimized dynamic path using the algo-128 SID (16131), and VRF forwarding pointing at that SR Policy.

---

## Appendix — Quick Verification Cheat Sheet

| Purpose | Command |
|---------|---------|
| OSPF neighbors | `show ospf neighbor` |
| LDP neighbors / bindings | `show mpls ldp neighbor brief` / `show mpls ldp bindings` |
| IS-IS adjacencies / topology | `show isis adjacency` / `show isis topology` |
| SR label table | `show isis segment-routing label table` |
| SRGB consistency | `show segment-routing local-block inconsistencies` |
| LFA / TI-LFA coverage | `show isis fast-reroute summary` |
| Per-prefix FRR detail | `show isis fast-reroute <prefix> detail` |
| SRLG membership | `show srlg interface <intf>` |
| Mapping server (SRMS) | `show segment-routing mapping-server prefix-sid-map ipv4` |
| SRMS active policy (client) | `show isis segment-routing prefix-sid-map active-policy` |
| Flex-Algo state | `show isis flex-algo <n>` |
| PM delay | `show performance-measurement interfaces` |
| BGP VPNv4 summary | `show bgp vpnv4 unicast summary` |
| VRF detail | `show vrf <name> detail` |
| SR-TE / ODN policy | `show segment-routing traffic-eng policy color <n>` |
| MPLS forwarding | `show mpls forwarding [labels <n>] [prefix <p>]` |

---

*End of SR Workbook Solutions (SR-EX01 – SR-EX05).*
