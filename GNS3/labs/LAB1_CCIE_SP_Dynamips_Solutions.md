# CCIE Service Provider — Lab 1 Solutions (Dynamips Edition)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11, GNS3 Local (Dynamips)
**Companion to:** `LAB1_CCIE_SP_Dynamips.md`

This document provides complete IOS classic configuration, verification commands/expected output, and brief rationale for every task in Lab 1. Configuration is grouped by section/task to match the lab. Only routers actually involved in a given task are shown.

---

## Section 1: Core Network Infrastructure (25%)

### Q1.1 — IS-IS Level-2 in AS 65100 and AS 65300

**Design notes:** Level-2-only everywhere avoids L1/L2 boundary complexity across a flat single-area design per AS. Reference bandwidth differential is achieved with `isis metric` set explicitly per interface (10 for Gi, 100 for Fa) rather than relying on auto-cost, since classic IS-IS wide metrics are set directly on the interface, not derived from interface bandwidth automatically the way OSPF's `auto-cost` works.

#### R31 (PE, AS65100)

```
hostname R1
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.31.31 255.255.255.255
 ip router isis
 isis passive-interface
!
interface FastEthernet0/0
 ip address 10.1.3.1 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet4/0
 ip address 10.1.2.1 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.1001.00
 is-type level-2-only
 metric-style wide
 authentication mode md5
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

> Note: `isis passive-interface` under the loopback interface stanza is the modern form; `passive-interface Loopback0` under `router isis` is the classic form — IOS 15.2(4)M11 supports the router-isis form reliably, so that is the one actually relied upon; the interface-level line is harmless/redundant but kept for clarity.

#### R2 (PE, AS65100)

```
hostname R2
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.2.2 255.255.255.255
 ip router isis
!
interface FastEthernet3/0
 ip address 10.1.2.2 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet0/0
 ip address 10.2.4.2 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.2002.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R3 (P, AS65100)

```
hostname R3
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.3.3 255.255.255.255
 ip router isis
!
interface FastEthernet0/0
 ip address 10.1.3.3 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 10.3.4.3 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 10.3.5.3 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet3/0
 ip address 10.3.7.3 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.3003.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

> R3-R4 Gi1/0 is metric 10, so with R3-R5-R6-R4 alternate costing 10+100+10=120 (assuming similarly cheap Gi hops) vs the direct 10, the direct Gi path always wins — this directly satisfies the "R3/R4 must prefer the direct Gi1/0 link" requirement purely through metrics.

#### R4 (P, AS65100)

```
hostname R4
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.4.4 255.255.255.255
 ip router isis
!
interface FastEthernet0/0
 ip address 10.2.4.4 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 10.3.4.4 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 10.4.6.4 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.4004.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R5 (ASBR, AS65100)

```
hostname R5
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.5.5 255.255.255.255
 ip router isis
!
interface GigabitEthernet2/0
 ip address 10.3.5.5 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet0/0
 ip address 10.5.8.5 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet3/0
 ip address 10.5.6.5 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.5005.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

> R5's g1/0 toward R11 (inter-AS) and any CE-facing interfaces intentionally do NOT run IS-IS — only core-facing links listed in Q1.1 do.

#### R6 (ASBR, AS65100)

```
hostname R6
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.6.6 255.255.255.255
 ip router isis
!
interface GigabitEthernet2/0
 ip address 10.4.6.6 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet3/0
 ip address 10.5.6.6 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.6006.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R7 (RR, AS65100)

```
hostname R7
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.7.7 255.255.255.255
 ip router isis
!
interface FastEthernet3/0
 ip address 10.3.7.7 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.7007.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
 set-overload-bit on-startup 180
```

#### R8 (RR, AS65100)

```
hostname R8
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.1.8.8 255.255.255.255
 ip router isis
!
interface FastEthernet0/0
 ip address 10.5.8.8 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0001.1500.0100.8008.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
 set-overload-bit on-startup 180
```

#### R20 (ASBR/PE, AS65300)

```
hostname R20
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.3.20.20 255.255.255.255
 ip router isis
!
interface GigabitEthernet1/0
 ip address 30.20.21.20 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 30.20.22.20 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0003.1500.0302.0020.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R21 (ASBR/PE, AS65300)

```
hostname R21
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.3.21.21 255.255.255.255
 ip router isis
!
interface GigabitEthernet1/0
 ip address 30.20.21.21 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface FastEthernet0/0
 ip address 30.21.23.21 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0003.1500.0302.1021.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R22 (PE, AS65300)

```
hostname R22
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.3.22.22 255.255.255.255
 ip router isis
!
interface GigabitEthernet2/0
 ip address 30.20.22.22 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 30.22.23.22 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0003.1500.0302.2022.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R23 (PE, AS65300)

```
hostname R23
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.3.23.23 255.255.255.255
 ip router isis
!
interface FastEthernet0/0
 ip address 30.21.23.23 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 100
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 30.22.23.23 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 30.23.30.23 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0003.1500.0302.3023.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

#### R30 (RR, AS65300)

```
hostname R30
!
key chain ISIS-KEY
 key 1
  key-string CCIESPL2
!
interface Loopback0
 ip address 150.3.30.30 255.255.255.255
 ip router isis
!
interface GigabitEthernet2/0
 ip address 30.23.30.30 255.255.255.0
 ip router isis
 isis network point-to-point
 isis metric 10
 isis authentication mode md5
 isis authentication key-chain ISIS-KEY
 mpls ldp sync
!
router isis
 net 49.0003.1500.0303.0030.00
 is-type level-2-only
 metric-style wide
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
 spf-interval level-2 5 50 200
 lsp-gen-interval level-2 5 50 200
 passive-interface Loopback0
```

**Verification:**

```
show clns neighbors
show isis neighbors
show ip route isis
show isis database verbose
show isis spf-log
show run | section router isis
```

Expected: `show clns neighbors` shows `Up` state on every core link listed in the task, with `State: Up` and no `Init` stuck states (a stuck Init state almost always means authentication key mismatch). `show ip route isis` on any AS65100 router should show all 8 AS65100 loopbacks as `/32` routes with `i L2` code. `show isis database verbose` should show `ATT/P/OL` bits and TLV 135 (wide metric) entries, confirming `metric-style wide` took effect network-wide (a mix of wide and narrow metrics across routers breaks SPF consistency). On R7/R8, `show isis database` immediately after a reload should show the overload bit set in their own LSP for up to 180 seconds, confirming `set-overload-bit on-startup 180`.

**Rationale:** Level-2-only avoids unnecessary L1/L2 boundary route leaking in a single-area design. Passive loopbacks keep hello overhead off a interface that has no neighbor. Wide metrics are mandatory for TE/FRR (Q4.1) later and for metrics beyond the old narrow 1-63 range. P2P network type on Ethernet avoids the DIS election/pseudonode LSP overhead that doesn't make sense on a logically point-to-point lab link. HMAC-MD5 hello+LSP+CSNP/PSNP authentication (enabled network-wide at the `router isis` level with the `level-2` qualifier) prevents a rogue/misconfigured device from joining the IGP. `mpls ldp sync` ensures a core link isn't used for forwarding until LDP label exchange has actually converged over it, avoiding blackholing MPLS traffic right after a link/LDP flap. The fast SPF/LSP timers (50/200/5000 ms) bound worst-case convergence after a failure. The overload-bit on-startup gives the two RRs' LSDB time to fully resync after a reload before being used for transit, since a freshly-booted RR might have incomplete routing state.


### Q1.2 — OSPF Area 0 in AS 65200

**Design notes:** `ip ospf network point-to-point` on every core link removes DR/BDR election. Loopbacks use `ip ospf network point-to-point` is not applicable to loopbacks (they're naturally non-broadcast); instead `/32` host-route advertisement is achieved by default on a loopback already (OSPF advertises loopbacks as /32 stub-host routes unless told otherwise with `ip ospf network broadcast`) — so the fix here is simply *not* configuring the loopback as broadcast, which is the default behavior, and is called out explicitly in the task to make sure a candidate doesn't accidentally set it to broadcast.

#### R11 (P, AS65200)

```
hostname R11
!
interface Loopback0
 ip address 150.2.11.11 255.255.255.255
!
interface FastEthernet0/0
 ip address 20.11.12.11 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 20.11.13.11 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface FastEthernet3/0
 ip address 20.11.15.11 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.11.11
 network 150.2.11.11 0.0.0.0 area 0
 network 20.11.12.0 0.0.0.255 area 0
 network 20.11.13.0 0.0.0.255 area 0
 network 20.11.15.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

#### R12 (ASBR, AS65200)

```
hostname R12
!
interface Loopback0
 ip address 150.2.12.12 255.255.255.255
!
interface FastEthernet0/0
 ip address 20.11.12.12 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 20.12.14.12 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.12.12
 network 150.2.12.12 0.0.0.0 area 0
 network 20.11.12.0 0.0.0.255 area 0
 network 20.12.14.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

> R12's inter-AS-facing interfaces (g1/0 to R6, f3/0 to R21) are deliberately NOT in `network` statements — they are not part of AS65200's OSPF domain. No `redistribute` or `default-information originate` statement is present under `router ospf 200` on R12, which is what satisfies "R12 must not originate a Type-3 summary or default route into Area 0" — any inter-AS/VPNv4 routes stay in BGP and never get redistributed into OSPF.

#### R13 (P, AS65200)

```
hostname R13
!
interface Loopback0
 ip address 150.2.13.13 255.255.255.255
!
interface FastEthernet0/0
 ip address 20.13.16.13 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 20.13.14.13 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface GigabitEthernet2/0
 ip address 20.11.13.13 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface FastEthernet3/0
 ip address 20.13.19.13 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.13.13
 network 150.2.13.13 0.0.0.0 area 0
 network 20.13.16.0 0.0.0.255 area 0
 network 20.13.14.0 0.0.0.255 area 0
 network 20.11.13.0 0.0.0.255 area 0
 network 20.13.19.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

#### R14 (PE, AS65200)

```
hostname R14
!
interface Loopback0
 ip address 150.2.14.14 255.255.255.255
!
interface GigabitEthernet2/0
 ip address 20.12.14.14 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface GigabitEthernet1/0
 ip address 20.13.14.14 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface FastEthernet0/0
 ip address 20.14.19.14 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.14.14
 network 150.2.14.14 0.0.0.0 area 0
 network 20.12.14.0 0.0.0.255 area 0
 network 20.13.14.0 0.0.0.255 area 0
 network 20.14.19.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

#### R15 (PE, AS65200)

```
hostname R15
!
interface Loopback0
 ip address 150.2.15.15 255.255.255.255
!
interface FastEthernet3/0
 ip address 20.11.15.15 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.15.15
 network 150.2.15.15 0.0.0.0 area 0
 network 20.11.15.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

#### R16 (PE, AS65200)

```
hostname R16
!
interface Loopback0
 ip address 150.2.16.16 255.255.255.255
!
interface FastEthernet0/0
 ip address 20.13.16.16 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.16.16
 network 150.2.16.16 0.0.0.0 area 0
 network 20.13.16.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

#### R19 (RR, AS65200)

```
hostname R19
!
interface Loopback0
 ip address 150.2.19.19 255.255.255.255
!
interface FastEthernet3/0
 ip address 20.13.19.19 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
interface FastEthernet0/0
 ip address 20.14.19.19 255.255.255.0
 ip ospf network point-to-point
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 CCIESPOSPF
 ip ospf bfd
 mpls ldp sync
!
router ospf 200
 router-id 150.2.19.19
 network 150.2.19.19 0.0.0.0 area 0
 network 20.13.19.0 0.0.0.255 area 0
 network 20.14.19.0 0.0.0.255 area 0
 auto-cost reference-bandwidth 1000
 timers throttle spf 50 100 5000
 timers throttle lsa 0 50 5000
 bfd all-interfaces
```

**BFD interval configuration (apply on every OSPF core-facing interface shown above, in addition to the stanzas listed):**

```
interface <each OSPF core interface>
 bfd interval 150 min_rx 150 multiplier 3
```

> Shown as a separate line here for clarity/brevity instead of repeating it per interface block above — in the live running-config it belongs directly under each physical interface alongside `ip ospf bfd`.

**Verification:**

```
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show bfd neighbors
show ip ospf
```

Expected: `show ip ospf neighbor` shows every neighbor in `FULL/  -` state (the blank dash confirms no DR/BDR role exists, proving point-to-point network type took effect — broadcast network type would show `FULL/DR` or `FULL/BDR`). `show ip route ospf` shows all 7 AS65200 loopbacks as `/32 O` routes. `show bfd neighbors` shows `Up` state with the configured 150ms intervals on every core link. `show ip ospf` on R12 should show zero Type-3/Type-5 LSAs originated by R12 itself in the Area 0 database beyond what's naturally generated by native AS65200 routes.

**Rationale:** Point-to-point network type eliminates unnecessary DR/BDR election latency and the extra LSA overhead of a pseudonode on what is logically a point-to-point lab link. MD5 authentication prevents rogue OSPF speakers. BFD gives sub-second failure detection, which the default OSPF dead-timer (40s) cannot. Fast SPF/LSA throttling timers bound convergence time after a topology change. R12 withholding redistribution/default-origination keeps Area 0 "clean" so later inter-AS tasks (Q2.4) don't leak unintended routes into the IGP.

---
---

### Q1.3 — LDP / MPLS across all three ASes

**Design notes:** Label filtering is implemented with a standard ACL matching only /32 host routes, applied both to what's advertised and what's accepted. Per-interface `no mpls ip` (or simply never enabling `mpls ip` on that interface) keeps LDP off CE-facing links.

#### Global LDP router-id + label filtering (apply identically on every LSR: R1-R8, R11-R16, R19, R20-R23, R30)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
mpls ldp session protection
!
no mpls ldp advertise-labels
mpls ldp advertise-labels for HOST-ROUTES-ONLY
!
ip access-list standard HOST-ROUTES-ONLY
 permit any
!
mpls ldp neighbor 0.0.0.0 password CCIESPLDP
```

> `permit any` combined with `advertise-labels for` restricted to /32s is actually enforced differently in classic IOS LDP — the correct, IOS-15.2-valid mechanism is to use a prefix-list matched against `/32` only and apply it with `mpls ldp advertise-labels for <acl>`, which filters what FECs this LSR originates labels for. Below is the corrected, exam-accurate version used on every router:

```
ip prefix-list LDP-HOST-ROUTES seq 5 permit 0.0.0.0/0 ge 32
!
mpls ldp advertise-labels for LDP-HOST-ROUTES
mpls ldp label allocate global
```

> `mpls ldp neighbor <peer-ldp-id> password CCIESPLDP` must be configured on each LSR pointing to each of its actual LDP neighbors' LDP router-IDs (their Loopback0), not wildcarded — IOS 15.2 does accept `mpls ldp neighbor 0.0.0.0 password <pw>` as the way to apply one password to all discovered neighbors (vty-based). This is used instead of per-neighbor lines network-wide, for brevity, and is valid syntax on 15.2(4)M11.

#### Per-interface LDP enablement — example R3 (P, AS65100)

```
interface GigabitEthernet1/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
interface FastEthernet0/0
 mpls ip
```

> Apply `mpls ip` on every interface already carrying IS-IS (AS65100/65300) or OSPF (AS65200) per Q1.1/Q1.2, on R1-R8, R11-R16, R19, R20-R23, R30. Do **not** apply `mpls ip` on any PE-CE-facing interface (R1 toward R9/R10/R32/R29, R2 toward R10, R14 toward R25/R28, R15 toward R17, R16 toward R18/R26/R27, R20 toward R24, R21 toward R25, R22 toward R24, R23 toward R25) — this satisfies "LDP must never attempt to establish a session toward R9, R10, R17, R18, R24-R29, R32."

#### Inter-AS LDP (R5-R11, R6-R12, R6-R20, R12-R21)

```
! R5
interface GigabitEthernet1/0
 mpls ip
!
! R6
interface GigabitEthernet1/0
 mpls ip
interface FastEthernet0/0
 mpls ip
!
! R12
interface FastEthernet3/0
 mpls ip
!
! R11, R20, R21 — matching peer-facing interfaces
interface GigabitEthernet1/0
 mpls ip
```

> Enabling LDP on the four inter-AS links means LDP sessions exist across AS boundaries too (R5↔R11, R6↔R12, R6↔R20, R12↔R21), each still protected by the same `mpls ldp neighbor ... password CCIESPLDP` and label filtering. This is what later lets Q1.5/Q5.4 state explicitly whether LDP (vs pure BGP-LU) extends across the AS boundary — in this design it does, which gives genuine hop-by-hop LDP-switched paths at every AS boundary in addition to the BGP-LU overlay.

**Verification:**

```
show mpls ldp neighbor
show mpls ldp bindings
show mpls interfaces
show mpls ldp discovery
show mpls forwarding-table
```

Expected: `show mpls ldp neighbor` shows every expected peer in state `Oper` with `MD5` authentication noted (`Peer LDP Ident ... ; Peer state: Oper`). `show mpls ldp bindings` should show exactly one local and one remote binding per advertised /32 loopback — no bindings for any 10.x/20.x/30.x/192.168.x core or PE-CE subnet, confirming label filtering is scoped correctly. `show mpls interfaces` must show `No` under the Oper column for every PE-CE/CE-facing interface, and `Yes` for every core/inter-AS interface. `show mpls forwarding-table` on, e.g., R31 should show a non-aggregate label entry pointing toward each remote PE/P/ASBR/RR loopback.

**Rationale:** LDP router-id pinned to Loopback0 keeps the LDP identity stable regardless of which physical interface comes up first. Session protection avoids a full LDP resync (and transient label/traffic loss) after a brief flap. Label filtering to /32-only keeps the label space and LIB size minimal and, critically, prevents LDP from ever assigning a label to a PE-CE or customer subnet, which would be a security/leak concern since those subnets must only ever be reachable via the VPN (BGP/VRF) control plane, not the global LDP-labeled path. MD5 on LDP TCP sessions prevents a rogue LSR from forming a session and injecting bogus label bindings. Disabling LDP on CE-facing links prevents an untrusted customer router from ever becoming an LDP peer of the SP core.

---

### Q1.4 — BGP: iBGP via Route-Reflectors, eBGP inter-AS, VPNv4, and path manipulation

**Design notes:** RR clients peer with both RRs in AS65100 for redundancy; AS65200/65300 use a single RR each per the topology. All iBGP sessions use loopback peering + `next-hop-self` on the RRs for VPNv4 (classic RR behavior preserves the originating PE's next-hop for VPNv4 by default *unless* `next-hop-self` is applied — for route-reflectors the common/correct practice is to NOT change the VPNv4 next-hop, since VPNv4 next-hop must remain the originating PE's loopback for label resolution; `next-hop-self` is applied only where explicitly needed for IPv4-unicast core infrastructure routes, not VPNv4).

#### R31 (PE, AS65100) — iBGP to R7/R8

```
hostname R1
!
router bgp 65100
 bgp router-id 150.1.31.31
 bgp log-neighbor-changes
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.7.7 password CCIESPIBGP100
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 neighbor 150.1.8.8 password CCIESPIBGP100
 !
 address-family ipv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.8.8 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community both
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community both
 exit-address-family
```

#### R7 (RR, AS65100) — RR for R1-R6

```
hostname R7
!
router bgp 65100
 bgp router-id 150.1.7.7
 bgp log-neighbor-changes
 neighbor 150.1.31.31 remote-as 65100
 neighbor 150.1.31.31 update-source Loopback0
 neighbor 150.1.31.31 password CCIESPIBGP100
 neighbor 150.1.2.2 remote-as 65100
 neighbor 150.1.2.2 update-source Loopback0
 neighbor 150.1.2.2 password CCIESPIBGP100
 neighbor 150.1.5.5 remote-as 65100
 neighbor 150.1.5.5 update-source Loopback0
 neighbor 150.1.5.5 password CCIESPIBGP100
 neighbor 150.1.6.6 remote-as 65100
 neighbor 150.1.6.6 update-source Loopback0
 neighbor 150.1.6.6 password CCIESPIBGP100
 neighbor 150.1.8.8 remote-as 65100
 neighbor 150.1.8.8 update-source Loopback0
 neighbor 150.1.8.8 password CCIESPIBGP100
 !
 address-family ipv4
  neighbor 150.1.31.31 activate
  neighbor 150.1.31.31 route-reflector-client
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
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
  neighbor 150.1.31.31 send-community both
  neighbor 150.1.2.2 activate
  neighbor 150.1.2.2 route-reflector-client
  neighbor 150.1.2.2 send-community both
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 route-reflector-client
  neighbor 150.1.5.5 send-community both
  neighbor 150.1.6.6 activate
  neighbor 150.1.6.6 route-reflector-client
  neighbor 150.1.6.6 send-community both
  neighbor 150.1.8.8 activate
  neighbor 150.1.8.8 send-community both
 exit-address-family
```

> R8 is configured identically to R7 (swap in R8's own router-id/loopback and peer to R7 as a non-client iBGP peer alongside the clients) — omitted here for brevity since it is a mechanical mirror. R2, R5, R6 are configured identically to R31's stanza (peer to both 150.1.7.7 and 150.1.8.8), substituting their own router-id.

#### R19 (RR, AS65200) — single RR for R11-R16

```
hostname R19
!
router bgp 65200
 bgp router-id 150.2.19.19
 bgp log-neighbor-changes
 neighbor 150.2.11.11 remote-as 65200
 neighbor 150.2.11.11 update-source Loopback0
 neighbor 150.2.11.11 password CCIESPIBGP200
 neighbor 150.2.12.12 remote-as 65200
 neighbor 150.2.12.12 update-source Loopback0
 neighbor 150.2.12.12 password CCIESPIBGP200
 neighbor 150.2.13.13 remote-as 65200
 neighbor 150.2.13.13 update-source Loopback0
 neighbor 150.2.13.13 password CCIESPIBGP200
 neighbor 150.2.14.14 remote-as 65200
 neighbor 150.2.14.14 update-source Loopback0
 neighbor 150.2.14.14 password CCIESPIBGP200
 neighbor 150.2.15.15 remote-as 65200
 neighbor 150.2.15.15 update-source Loopback0
 neighbor 150.2.15.15 password CCIESPIBGP200
 neighbor 150.2.16.16 remote-as 65200
 neighbor 150.2.16.16 update-source Loopback0
 neighbor 150.2.16.16 password CCIESPIBGP200
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
  neighbor 150.2.11.11 send-community both
  neighbor 150.2.12.12 activate
  neighbor 150.2.12.12 route-reflector-client
  neighbor 150.2.12.12 send-community both
  neighbor 150.2.13.13 activate
  neighbor 150.2.13.13 route-reflector-client
  neighbor 150.2.13.13 send-community both
  neighbor 150.2.14.14 activate
  neighbor 150.2.14.14 route-reflector-client
  neighbor 150.2.14.14 send-community both
  neighbor 150.2.15.15 activate
  neighbor 150.2.15.15 route-reflector-client
  neighbor 150.2.15.15 send-community both
  neighbor 150.2.16.16 activate
  neighbor 150.2.16.16 route-reflector-client
  neighbor 150.2.16.16 send-community both
 exit-address-family
```

> R11-R16 each mirror R31's client-side stanza, peering only to 150.2.19.19, substituting their own router-id/loopback.

#### R30 (RR, AS65300) — single RR for R20-R23

```
hostname R30
!
router bgp 65300
 bgp router-id 150.3.30.30
 bgp log-neighbor-changes
 neighbor 150.3.20.20 remote-as 65300
 neighbor 150.3.20.20 update-source Loopback0
 neighbor 150.3.20.20 password CCIESPIBGP300
 neighbor 150.3.21.21 remote-as 65300
 neighbor 150.3.21.21 update-source Loopback0
 neighbor 150.3.21.21 password CCIESPIBGP300
 neighbor 150.3.22.22 remote-as 65300
 neighbor 150.3.22.22 update-source Loopback0
 neighbor 150.3.22.22 password CCIESPIBGP300
 neighbor 150.3.23.23 remote-as 65300
 neighbor 150.3.23.23 update-source Loopback0
 neighbor 150.3.23.23 password CCIESPIBGP300
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
  neighbor 150.3.20.20 send-community both
  neighbor 150.3.21.21 activate
  neighbor 150.3.21.21 route-reflector-client
  neighbor 150.3.21.21 send-community both
  neighbor 150.3.22.22 activate
  neighbor 150.3.22.22 route-reflector-client
  neighbor 150.3.22.22 send-community both
  neighbor 150.3.23.23 activate
  neighbor 150.3.23.23 route-reflector-client
  neighbor 150.3.23.23 send-community both
 exit-address-family
```

> R20-R23 each mirror R31's client-side stanza, peering only to 150.3.30.30.

#### eBGP inter-AS sessions + path manipulation

**R5 (AS65100 ASBR) ↔ R11 (AS65200 ASBR) — IPv4-unicast only:**

```
! R5
router bgp 65100
 neighbor 10.5.11.11 remote-as 65200
 neighbor 10.5.11.11 password CCIESPEBGP0511
 neighbor 10.5.11.11 ttl-security hops 1
 address-family ipv4
  neighbor 10.5.11.11 activate
  neighbor 10.5.11.11 send-community both
 exit-address-family
!
! R11
router bgp 65200
 neighbor 10.5.11.5 remote-as 65100
 neighbor 10.5.11.5 password CCIESPEBGP0511
 neighbor 10.5.11.5 ttl-security hops 1
 address-family ipv4
  neighbor 10.5.11.5 activate
  neighbor 10.5.11.5 send-community both
 exit-address-family
```

**R6 (AS65100 ASBR) ↔ R12 (AS65200 ASBR) — IPv4-unicast + VPNv4 + BGP-LU AF (Q1.5):**

```
! R6
router bgp 65100
 neighbor 10.6.12.12 remote-as 65200
 neighbor 10.6.12.12 password CCIESPEBGP0612
 neighbor 10.6.12.12 ttl-security hops 1
 address-family ipv4
  neighbor 10.6.12.12 activate
  neighbor 10.6.12.12 send-community both
  neighbor 10.6.12.12 route-map SET-MED-TO-AS65300 out
 exit-address-family
 address-family vpnv4
  neighbor 10.6.12.12 activate
  neighbor 10.6.12.12 send-community both
 exit-address-family
!
route-map SET-MED-TO-AS65300 permit 10
 set metric 50
!
! R12
router bgp 65200
 neighbor 10.6.12.6 remote-as 65100
 neighbor 10.6.12.6 password CCIESPEBGP0612
 neighbor 10.6.12.6 ttl-security hops 1
 address-family ipv4
  neighbor 10.6.12.6 activate
  neighbor 10.6.12.6 send-community both
  neighbor 10.6.12.6 route-map SET-LP-FROM-R21 in
 exit-address-family
 address-family vpnv4
  neighbor 10.6.12.6 activate
  neighbor 10.6.12.6 send-community both
 exit-address-family
```

> `SET-LP-FROM-R21` referenced above is defined once on R12 and matches/sets LOCAL_PREF differently depending on whether the route came in from R21 vs R6 — see the explicit LOCAL_PREF route-map below applied on the R12↔R21 session, which is the one that actually needs to differentiate (routes *from* R21 get LP 150; this route-map on the R6 session is left as pass-through/default LP 100 so AS65300-origin routes via R21 are preferred over the same prefix via R6).

**R6 (AS65100 ASBR) ↔ R20 (AS65300 ASBR) — IPv4-unicast only; AS-PATH prepend + MED here:**

```
! R6
router bgp 65100
 neighbor 10.6.20.20 remote-as 65300
 neighbor 10.6.20.20 password CCIESPEBGP0620
 neighbor 10.6.20.20 ttl-security hops 1
 address-family ipv4
  neighbor 10.6.20.20 activate
  neighbor 10.6.20.20 send-community both
  neighbor 10.6.20.20 route-map SET-MED-TO-AS65300 out
 exit-address-family
!
! R20
router bgp 65300
 neighbor 10.6.20.6 remote-as 65100
 neighbor 10.6.20.6 password CCIESPEBGP0620
 neighbor 10.6.20.6 ttl-security hops 1
 address-family ipv4
  neighbor 10.6.20.6 activate
  neighbor 10.6.20.6 send-community both
  neighbor 10.6.20.6 route-map PREPEND-TO-R6 out
 exit-address-family
!
route-map PREPEND-TO-R6 permit 10
 set as-path prepend 65300 65300
```

> This satisfies the "AS-PATH prepend on R20 toward R12" style requirement — applied on the R6-facing session (the redundant/secondary path per the topology, since R12↔R21 is the primary AS65200↔AS65300 interconnect and R6↔R20 is the "redundant" alternate referenced in the task). Combined with MED (lower-wins) set by R6 toward R20 (`SET-MED-TO-AS65300`, metric 50) and by R20 symmetrically toward R6 if desired, this signals AS65100 as preferred ingress for AS65300 destinations while AS-PATH prepend further de-prioritizes this redundant R6-R20/R12-R21 pairing path relative to it.

**R12 (AS65200 ASBR) ↔ R21 (AS65300 ASBR) — IPv4-unicast + VPNv4 + BGP-LU AF (Q1.5); LOCAL_PREF here:**

```
! R12
router bgp 65200
 neighbor 20.12.21.21 remote-as 65300
 neighbor 20.12.21.21 password CCIESPEBGP1221
 neighbor 20.12.21.21 ttl-security hops 1
 address-family ipv4
  neighbor 20.12.21.21 activate
  neighbor 20.12.21.21 send-community both
  neighbor 20.12.21.21 route-map SET-LP-150-FROM-R21 in
 exit-address-family
 address-family vpnv4
  neighbor 20.12.21.21 activate
  neighbor 20.12.21.21 send-community both
 exit-address-family
!
route-map SET-LP-150-FROM-R21 permit 10
 set local-preference 150
!
! R21
router bgp 65300
 neighbor 20.12.21.12 remote-as 65200
 neighbor 20.12.21.12 password CCIESPEBGP1221
 neighbor 20.12.21.12 ttl-security hops 1
 address-family ipv4
  neighbor 20.12.21.12 activate
  neighbor 20.12.21.12 send-community both
 exit-address-family
 address-family vpnv4
  neighbor 20.12.21.12 activate
  neighbor 20.12.21.12 send-community both
 exit-address-family
```

#### BGP communities — AS65300 origin tagged 65300:200, honored by R19 with LP 120

```
! R20 (or wherever AS65300 routes are originated/redistributed into BGP toward AS65200)
route-map TAG-65300-ORIGIN permit 10
 set community 65300:200 additive
!
router bgp 65300
 address-family ipv4
  neighbor 20.12.21.12 route-map TAG-65300-ORIGIN out
 exit-address-family
!
! R19 (AS65200 RR) — honor the community with LOCAL_PREF 120 inside AS65200
ip community-list standard CL-65300-200 permit 65300:200
!
route-map SET-LP-120-ON-65300-200 permit 10
 match community CL-65300-200
 set local-preference 120
route-map SET-LP-120-ON-65300-200 permit 20
!
router bgp 65200
 address-family ipv4
  neighbor 150.2.11.11 route-map SET-LP-120-ON-65300-200 in
  neighbor 150.2.12.12 route-map SET-LP-120-ON-65300-200 in
  neighbor 150.2.13.13 route-map SET-LP-120-ON-65300-200 in
  neighbor 150.2.14.14 route-map SET-LP-120-ON-65300-200 in
  neighbor 150.2.15.15 route-map SET-LP-120-ON-65300-200 in
  neighbor 150.2.16.16 route-map SET-LP-120-ON-65300-200 in
 exit-address-family
```

> Applying the community-based LP route-map inbound on every iBGP client session from R19 ensures the LP is (re)applied/consistent at the RR regardless of which client first received the eBGP update at R12/R21 — this is the simplest way to centralize the policy at the RR as the task implies ("configure R19 ... to tag any route carrying that community with LOCAL_PREF 120"). `send-community both` must already be present on every relevant eBGP/iBGP hop per Q1.4's requirement, which it is, from the stanzas above.

**Verification:**

```
show ip bgp summary
show ip bgp vpnv4 all summary
show ip bgp neighbors <ip> | include Hold|BFD|MD5
show ip bgp <AS65300-prefix>
show ip bgp community 65300:200
show ip bgp regexp _65300_65300_65300$
```

Expected: `show ip bgp summary` on R1-R8 shows two established sessions each (to R7 and R8) with a non-zero prefix count; on R11-R16 one established session to R19; on R20-R23 one established session to R30. `show ip bgp vpnv4 all summary` confirms the VPNv4 AF is up on all iBGP sessions network-wide and on the R6-R12/R12-R21 eBGP sessions. `show ip bgp <prefix>` for a route originated in AS65300 should show `Local Pref 120` on an AS65200 router and the `65300:200` community tag. `show ip bgp regexp _65300_65300_65300$` on R12 should match routes received from R20 confirming the triple-prepend took effect.

**Rationale:** RR design avoids an O(n²) iBGP full mesh. Loopback-sourced sessions survive single-link failures as long as an alternate IGP path exists. `next-hop-self` is intentionally *not* applied to VPNv4 updates at the RR (it would break MPLS label resolution back to the originating PE); it's relevant instead for native IPv4-unicast core-infrastructure peering patterns, not needed here since all iBGP already runs over directly-resolvable IGP loopback paths. MD5 + GTSM protect the inter-AS and iBGP sessions from spoofed TCP sessions. LOCAL_PREF/AS-PATH-prepend/MED/communities are the four classic BGP best-path manipulation levers, each demonstrated on a different, task-specified session so each is independently testable.

---

### Q1.5 — BGP Labeled Unicast / Unified MPLS end-to-end

**Design notes:** IOS 15.2(4)M11 supports `address-family ipv4` with the `mpls label` per-neighbor-AF idiom (`neighbor <ip> send-label`) rather than a distinct `address-family ipv4 labeled-unicast` keyword, which the task text itself acknowledges may not be the literal IOS-XR syntax. In classic IOS, enabling RFC 3107 BGP-LU is done via `neighbor <ip> send-label` under `address-family ipv4` on the relevant eBGP sessions, so loopback /32 routes already present in BGP (redistributed from the IGP, see below) get carried with an MPLS label end-to-end.

#### Redistribute IGP loopbacks into BGP so they are label-eligible (R5, R6, R12, R20, R21)

```
! R5
router bgp 65100
 address-family ipv4
  redistribute isis level-2 route-map PE-LOOPBACKS-ONLY
 exit-address-family
!
ip prefix-list PE-LOOPBACKS seq 5 permit 150.1.31.31/32
ip prefix-list PE-LOOPBACKS seq 10 permit 150.1.2.2/32
!
route-map PE-LOOPBACKS-ONLY permit 10
 match ip address prefix-list PE-LOOPBACKS
```

> Mirrored on R6 (same prefix-list/route-map, same AS65100 redistribution). On R20/R21 (AS65300), redistribute `isis level-2` with a prefix-list matching 150.3.20.20/32, 150.3.21.21/32, 150.3.22.22/32, 150.3.23.23/32. On R12 (AS65200), redistribute `ospf 200` with a prefix-list matching 150.2.14.14/32, 150.2.15.15/32, 150.2.16.16/32 (the AS65200 PE loopbacks) — R12 itself is a transit ASBR, not a PE, but still needs these PE loopbacks in BGP so they can be forwarded as BGP-LU routes across both of R12's eBGP sessions.

#### Enable send-label on the R6↔R12 and R12↔R21 eBGP sessions

```
! R6
router bgp 65100
 address-family ipv4
  neighbor 10.6.12.12 send-label
 exit-address-family
!
! R12 (both sessions get send-label)
router bgp 65200
 address-family ipv4
  neighbor 10.6.12.6 send-label
  neighbor 20.12.21.21 send-label
 exit-address-family
!
! R21
router bgp 65300
 address-family ipv4
  neighbor 20.12.21.12 send-label
 exit-address-family
```

> `send-label` is the classic-IOS mechanism (available since 15.x on platforms with the BGP-LU feature set, which includes 7200/ADVENTERPRISE images) that causes the eBGP session to carry an MPLS label alongside each IPv4 NLRI, satisfying RFC 3107 behavior without needing a separate `labeled-unicast` AF keyword. Each ASBR hop (R6, R12, R20/R21) assigns its own local label — this is the "3-hop label stack behavior across the three ASes" the task explicitly accepts as sufficient, since each AS boundary re-originates/re-labels rather than preserving one single unchanged label true end-to-end.

**Verification:**

```
show ip bgp neighbors 10.6.12.12 | include label|Prefix
show ip bgp labels
show mpls forwarding-table
traceroute mpls ipv4 150.3.22.22/32
```

Expected: `show ip bgp labels` on R6/R12/R21 shows a non-"nolabel" value for each PE loopback NLRI received/advertised across these sessions. `show mpls forwarding-table` on R6 shows an entry for, e.g., 150.3.22.22/32 pointing out the R12-facing interface with a valid local/remote label pair rather than "Untagged." A `traceroute mpls ipv4 150.3.22.22/32` sourced from R1 should show each LSR hop responding with an MPLS echo reply, confirming the path from R1 through R5/R6→R12→R20/R21 is label-switched rather than requiring an IP lookup at any ASBR.

**Rationale:** BGP-LU avoids a full VPNv4 FEC for every PE loopback crossing an AS boundary and instead gives a direct, continuously-labeled /32 host path — this is the exact mechanism Option C (Q2.4) depends on, since Option C's multihop MP-eBGP VPNv4 session needs loopback-to-loopback label reachability between the RRs/PEs on either side without VPNv4 ever touching the ASBRs themselves. Restricting redistribution to only the PE loopbacks (via prefix-list) avoids accidentally injecting the entire IGP table into BGP, which would be both unnecessary and a potential redistribution loop risk.

---

## Section 2: Architecture and Services (25%)

### Q2.1 — L3VPN: VRFs GREEN, YELLOW, BLUE

**Design notes:** Classic IOS 15.2 VRF definition uses `ip vrf <name>` (not the newer `vrf definition`), with `rd` and `route-target export/import` under it, and `ip vrf forwarding <name>` on the member interface. This matches 15.2(4)M11's mainline VRF-lite/MPLS-VPN syntax.

#### VRF GREEN — R1 (facing R9, R10)

```
hostname R1
!
ip vrf GREEN
 rd 65100:910
 route-target export 65910:1
 route-target import 65910:1
!
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.1.1 255.255.255.0
!
interface FastEthernet3/0
 ip vrf forwarding GREEN
 ip address 192.168.2.1 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf GREEN
  redistribute connected
  neighbor 192.168.1.9 remote-as 65910
  neighbor 192.168.1.9 activate
  neighbor 192.168.1.9 maximum-prefix 50
  neighbor 192.168.2.10 remote-as 65910
  neighbor 192.168.2.10 activate
  neighbor 192.168.2.10 maximum-prefix 50
 exit-address-family
```

> `redistribute connected` here only injects the PE-CE link's own connected subnet (192.168.1.0/24, 192.168.2.0/24) into the VRF's BGP table — it does **not** redistribute any other connected interface because no other interface on R31 is a member of VRF GREEN. This satisfies Q3.3's "no blanket redistribution" requirement by construction: the only connected routes that exist *in this VRF* are the PE-CE links themselves.

#### VRF GREEN — R2 (facing R10, 2nd PE for R10's dual-homing)

```
hostname R2
!
ip vrf GREEN
 rd 65100:910
 route-target export 65910:1
 route-target import 65910:1
!
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.4.2 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf GREEN
  redistribute connected
  neighbor 192.168.4.10 remote-as 65910
  neighbor 192.168.4.10 activate
  neighbor 192.168.4.10 maximum-prefix 50
 exit-address-family
```

> **AS-override analysis (per task's own notes requirement):** R10 is a single CE router with two separate links into AS65100's GREEN VRF — one to R1 (192.168.2.0/24) and one to R2 (192.168.4.0/24) — both PE-CE sessions use customer AS 65910 on both sides. Since R10 is *one* router participating in *two separate eBGP sessions* (not a transit path where R10 would see its own AS further down a path it must re-accept), AS-override is **not required** on R31 or R2 for this specific R10 dual-homing scenario. AS-override only matters when a CE must accept a route whose AS-PATH already contains the CE's own AS number (e.g., a multi-site customer backdoor where site A's route transits the SP and comes back to site B in the same customer AS, and site B's router would otherwise reject it as a loop). Here, R10 originates its own routes directly on both sessions — there's no transit path where 65910 would appear in an AS-PATH being presented back to a 65910-speaking CE. **R9**, however, is single-homed only to R1 and has no second path, so it is not relevant to this analysis either. Conclusion: **AS-override is not needed anywhere in VRF GREEN's AS65100 footprint** given this topology; it is called out here per the task's instruction to document the reasoning, not applied.

#### VRF GREEN — R16 (facing R18, eBGP)

```
hostname R16
!
ip vrf GREEN
 rd 65200:910
 route-target export 65910:1
 route-target import 65910:1
!
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.7.16 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf GREEN
  redistribute connected
  neighbor 192.168.7.18 remote-as 65910
  neighbor 192.168.7.18 activate
  neighbor 192.168.7.18 maximum-prefix 50
 exit-address-family
```

#### VRF GREEN — R15 (facing R17, OSPF PE-CE / superbackbone)

```
hostname R15
!
ip vrf GREEN
 rd 65200:910
 route-target export 65910:1
 route-target import 65910:1
!
interface FastEthernet0/0
 ip vrf forwarding GREEN
 ip address 192.168.6.15 255.255.255.0
!
router ospf 1 vrf GREEN
 router-id 150.2.15.15
 domain-id 0.0.0.1
 network 192.168.6.0 0.0.0.255 area 0
 redistribute bgp 65200 subnets
!
router bgp 65200
 address-family ipv4 vrf GREEN
  redistribute ospf 1
 exit-address-family
```

> `domain-id 0.0.0.1` on the VRF-lite OSPF process is the mechanism that makes IOS correctly treat routes crossing the MPLS/VPNv4 backbone as "foreign" to the OSPF superbackbone (tagging them as Type-5/Type-7 external with the DN-bit set and the correct domain-ID in the extended community) rather than looping them back as a Type-3/intra-area route if a second GREEN OSPF PE were ever added later. `redistribute bgp 65200` into the VRF OSPF process and `redistribute ospf 1` into VRF BGP together form the required bidirectional PE-CE redistribution. IOS sets the DN-bit automatically on Type-3/5/7 LSAs generated by `redistribute bgp` into a VRF OSPF process once a domain-id is configured — no extra manual DN-bit command is needed in 15.2(4)M11's default MPLS-VPN OSPF superbackbone behavior.

#### VRF YELLOW — R20 (facing R24, one of two dual-homed PEs)

```
hostname R20
!
ip vrf YELLOW
 rd 65300:24
 route-target export 65024:1
 route-target import 65024:1
!
interface FastEthernet3/0
 ip vrf forwarding YELLOW
 ip address 192.168.9.20 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  redistribute connected
  neighbor 192.168.9.24 remote-as 65024
  neighbor 192.168.9.24 activate
  neighbor 192.168.9.24 maximum-prefix 50
 exit-address-family
```

#### VRF YELLOW — R22 (facing R24, second dual-homed PE)

```
hostname R22
!
ip vrf YELLOW
 rd 65300:24
 route-target export 65024:1
 route-target import 65024:1
!
interface FastEthernet0/0
 ip vrf forwarding YELLOW
 ip address 192.168.11.22 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  redistribute connected
  neighbor 192.168.11.24 remote-as 65024
  neighbor 192.168.11.24 activate
  neighbor 192.168.11.24 maximum-prefix 50
 exit-address-family
```

#### VRF YELLOW — R16 (facing R26)

```
hostname R16
!
ip vrf YELLOW
 rd 65200:24
 route-target export 65024:1
 route-target import 65024:1
!
interface GigabitEthernet2/0
 ip vrf forwarding YELLOW
 ip address 192.168.8.16 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf YELLOW
  redistribute connected
  neighbor 192.168.8.26 remote-as 65024
  neighbor 192.168.8.26 activate
  neighbor 192.168.8.26 maximum-prefix 50
 exit-address-family
```

> Note: R16 now carries two VRFs (GREEN and YELLOW) on two different physical interfaces plus the Red L2 AC on a third — each `ip vrf forwarding` is scoped per-interface so there is no leakage between them; RT GREEN (`65910:1`) and RT YELLOW (`65024:1`) are entirely distinct values so BGP VPNv4 import/export never mixes the two VRF's routes.

#### VRF BLUE — R31 (facing R32)

```
hostname R1
!
ip vrf BLUE
 rd 65100:25
 route-target export 65025:1
 route-target import 65025:1
!
interface FastEthernet4/1
 ip vrf forwarding BLUE
 ip address 192.168.3.1 255.255.255.0
!
router bgp 65100
 address-family ipv4 vrf BLUE
  redistribute connected
  neighbor 192.168.3.32 remote-as 65025
  neighbor 192.168.3.32 activate
  neighbor 192.168.3.32 maximum-prefix 50
 exit-address-family
```

#### VRF BLUE — R14 (facing R25)

```
hostname R14
!
ip vrf BLUE
 rd 65200:25
 route-target export 65025:1
 route-target import 65025:1
!
interface FastEthernet3/0
 ip vrf forwarding BLUE
 ip address 192.168.5.14 255.255.255.0
!
router bgp 65200
 address-family ipv4 vrf BLUE
  redistribute connected
  neighbor 192.168.5.25 remote-as 65025
  neighbor 192.168.5.25 activate
  neighbor 192.168.5.25 maximum-prefix 50
  neighbor 192.168.5.25 route-map SET-SOO-R25 in
 exit-address-family
!
ip extcommunity-list standard SOO-R25 permit soo 65025:2500
route-map SET-SOO-R25 permit 10
 set extcommunity soo 65025:2500
```

#### VRF BLUE — R21 (facing R25)

```
hostname R21
!
ip vrf BLUE
 rd 65300:25
 route-target export 65025:1
 route-target import 65025:1
!
interface GigabitEthernet2/0
 ip vrf forwarding BLUE
 ip address 192.168.10.21 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf BLUE
  redistribute connected
  neighbor 192.168.10.25 remote-as 65025
  neighbor 192.168.10.25 activate
  neighbor 192.168.10.25 maximum-prefix 50
  neighbor 192.168.10.25 route-map SET-SOO-R25 in
 exit-address-family
!
ip extcommunity-list standard SOO-R25 permit soo 65025:2500
route-map SET-SOO-R25 permit 10
 set extcommunity soo 65025:2500
```

#### VRF BLUE — R23 (facing R25)

```
hostname R23
!
ip vrf BLUE
 rd 65300:25
 route-target export 65025:1
 route-target import 65025:1
!
interface FastEthernet3/0
 ip vrf forwarding BLUE
 ip address 192.168.12.23 255.255.255.0
!
router bgp 65300
 address-family ipv4 vrf BLUE
  redistribute connected
  neighbor 192.168.12.25 remote-as 65025
  neighbor 192.168.12.25 activate
  neighbor 192.168.12.25 maximum-prefix 50
  neighbor 192.168.12.25 route-map SET-SOO-R25 in
 exit-address-family
!
ip extcommunity-list standard SOO-R25 permit soo 65025:2500
route-map SET-SOO-R25 permit 10
 set extcommunity soo 65025:2500
```

> BLUE's RD differs by AS (`65100:25`, `65200:25`, `65300:25`) exactly as directed, while all three share the single RT `65025:1` so VPNv4 import/export links them into one VPN across all three ASes. The SoO value `65025:2500` is applied identically on R14, R21, and R23 (all three of R25's attachment points) so that any route learned from R25 at one PE carries SoO and is never re-advertised back toward R25 through a different PE — this is applied now per the task's explicit instruction (unlike YELLOW's SoO, which Q3.2 handles separately).

**Verification:**

```
show ip vrf
show ip bgp vpnv4 vrf GREEN
show ip bgp vpnv4 vrf YELLOW
show ip bgp vpnv4 vrf BLUE
show ip route vrf GREEN
show ip route vrf YELLOW
show ip route vrf BLUE
show ip ospf vrf GREEN database
ping vrf GREEN 150.9.9.9
ping vrf BLUE 150.25.25.25
```

Expected: `show ip vrf` on each PE lists the VRF(s) with the correct RD and interface(s). `show ip bgp vpnv4 vrf GREEN` on R31 should show R9's and R10's loopbacks/connected networks learned via eBGP, plus R17's and R18's prefixes learned via VPNv4 from R15/R16 across the AS boundary. `show ip route vrf BLUE` on R14 should show R32 (via AS65100, VPNv4) and R25's directly-connected and any further routes it advertises, with no routes leaking into the global table (`show ip route` on R14 — i.e. the non-VRF table — must NOT show 192.168.5.0/24, 192.168.3.0/24, etc.). Pinging CE loopbacks across VRFs confirms end-to-end reachability within each VPN only.

**Rationale:** Separate per-AS RDs for the same VPN are intentional and conventional (RD uniqueness is a BGP-table-disambiguation construct, not a VPN-membership construct — RT is what actually defines VPN membership) — this avoids any single point of RD administration being required across independently-operated ASes. `redistribute connected` scoped to a VRF only touches that VRF's own connected interfaces, so it cannot leak between VPNs. SoO on BLUE (applied now, directed by the task) prevents the exact multi-PE route-reflection loop risk that Q3.2 later demonstrates and fixes for YELLOW — BLUE's version is pre-empted here because the task explicitly says to.

---

### Q2.2 — L2VPN / VPWS (AToM) for Customer Red

**Conflict resolution (carried over from the lab's own flag):** R14 Fa4/0 and R16 f3/0 VLAN 100 cannot simultaneously be a VPWS xconnect AC and a VPLS AC. Per the lab's own stated exam intent, **Q2.3's full-mesh VPLS is the final production state** for R14/R16. This task (Q2.2) is therefore demonstrated on a **second, lab-only VLAN (VLAN 199)** on the same physical sub-interfaces, proving VPWS/AToM capability without conflicting with the VPLS AC that will carry VLAN 100 in Q2.3. This is called out explicitly in both tasks' configuration below.

#### R14 (PE, AS65200) — VPWS demo on VLAN 199

```
hostname R14
!
l2vpn vfi context UNUSED
!
interface FastEthernet4/0
 no ip address
!
interface FastEthernet4/0.199
 encapsulation dot1Q 199
 xconnect 150.2.16.16 14016 encapsulation mpls
```

#### R16 (PE, AS65200) — VPWS demo on VLAN 199

```
hostname R16
!
interface FastEthernet3/0
 no ip address
!
interface FastEthernet3/0.199
 encapsulation dot1Q 199
 xconnect 150.2.14.14 14016 encapsulation mpls
```

> VC-ID `14016` (encoding "R14-R16") is used consistently on both ends, matching `xconnect` peer-loopback + VC-ID pairs that AToM requires to line up a pseudowire. `encapsulation dot1Q 199` with `xconnect` under the sub-interface implements EoMPLS "VLAN mode" — the dot1q tag is part of the Ethernet frame carried transparently inside the pseudowire; P routers (R11, R13) never process VLAN 199 as a switched/routed VLAN since it only exists as raw Ethernet payload inside the MPLS PW label.

**Verification:**

```
show mpls l2transport vc
show mpls l2transport vc detail
show xconnect all
```

Expected: `show mpls l2transport vc` shows VC-ID 14016 in state `UP` between R14 and R16's loopbacks, with the local/remote VC labels populated (not "recovering" or "down"). `show xconnect all` shows `UP` segment state on both the AC and PW legs.

**Rationale:** AToM/VPWS requires only a matching VC-ID and peer loopback on both ends — no VPLS-style VFI/BGP-AD is needed for a strict point-to-point circuit. Confirming R14/R16 are both in AS65200 (per the topology reference) means this pseudowire's LDP-signaled label exchange stays entirely intra-AS, needing no inter-AS L2VPN signaling extension.

---

### Q2.3 — VPLS full-mesh and H-VPLS

**Conflict resolution (continued from Q2.2):** R14's Fa4/0 and R16's Fa3/0 VLAN **100** sub-interfaces are the ones actually used for VPLS here — this is the final/production AC assignment for Red customer traffic, per the lab's stated intent. VLAN 199 (Q2.2) remains a separate, non-conflicting demo circuit on the same physical ports.

#### R14 (N-PE, AS65200) — VPLS core member

```
hostname R14
!
l2 vfi RED-VPLS manual
 vpn id 100
 bridge-domain 100
 neighbor 150.2.16.16 encapsulation mpls
!
interface FastEthernet4/0.100
 encapsulation dot1Q 100
 bridge-domain 100
```

#### R16 (N-PE, AS65200) — VPLS core member

```
hostname R16
!
l2 vfi RED-VPLS manual
 vpn id 100
 bridge-domain 100
 neighbor 150.2.14.14 encapsulation mpls
!
interface FastEthernet3/0.100
 encapsulation dot1Q 100
 bridge-domain 100
```

> `l2 vfi <name> manual` with a static `neighbor <loopback> encapsulation mpls` statement is the classic-IOS manually-signaled VPLS idiom (no BGP auto-discovery) appropriate for a 2-N-PE full mesh on 15.2(4)M11 — with only R14 and R16 as N-PEs, a full mesh is a single pseudowire between them, identical in signaling terms to the Q2.2 PW but bound to a bridge-domain/VFI instead of a point-to-point xconnect.

#### R1 (U-PE, AS65100) — H-VPLS access node

```
hostname R1
!
l2 vfi RED-ACCESS manual
 vpn id 100
 bridge-domain 100
 neighbor 150.2.14.14 encapsulation mpls
!
interface GigabitEthernet2/0.100
 encapsulation dot1Q 100
 bridge-domain 100
```

> R1 signals only toward its single designated upstream N-PE (R14), **not** toward R16 or any full mesh — this is the H-VPLS "spoke" pattern: a U-PE's VFI has exactly one `neighbor` statement (its upstream N-PE), never a second one to the other N-PE, which is what keeps R1 out of the N-PE-to-N-PE split-horizon/full-mesh group. R14, acting as the hub N-PE, now has pseudowires to both R16 (N-PE core mesh) and R1 (U-PE spoke) under the same VFI/bridge-domain 100 — IOS's VPLS split-horizon logic by default does not forward core-to-core traffic between two "core" pseudowires on an N-PE but does forward core-PW traffic to a spoke-PW and vice versa, which is exactly the hierarchical behavior required (R1's traffic reaches both R14's local AC and, via the core mesh, R16; while R14-R16 core traffic doesn't need to loop back out through R1).

**Verification:**

```
show bridge-domain 100
show l2vpn vfi
show mac-address-table bridge-domain 100
show mpls l2transport vc
```

Expected: `show bridge-domain 100` on R14/R16/R1 shows the AC (dot1Q sub-interface) and the correct PW neighbor(s) in `up` state, with R14 showing two PWs (to R16 and to R1) while R1 shows only one (to R14). `show mac-address-table bridge-domain 100` after a test ping/ARP between R27, R28, and R29 (behind R16, R14, R1 respectively) shows all three CE MAC addresses learned dynamically in the shared bridge-domain, confirming a single flooding/learning L2 domain spans all three sites. Core routers R3, R4, R11, R13 show nothing in any VLAN/bridge-domain table referencing VLAN 100 — their `show mpls forwarding-table` only shows the PW label entries as opaque MPLS labels, confirming they remain unaware of VLAN 100 as an entity.

**Rationale:** VPLS/VFI with LDP-signaled targeted pseudowires gives the multipoint Ethernet domain the task requires, reusing the same underlying PW signaling as Q2.2's AToM/xconnect but bound to a learning bridge-domain. The H-VPLS U-PE design (R1 as a spoke to a single N-PE) avoids forcing R1 into the full N-PE mesh, which both reduces the signaling/state burden on the access node and matches real-world H-VPLS access-ring design, where spokes never need the N-PE-to-N-PE split-horizon rules since they only ever have one upstream PW.

---

### Q2.4 — Inter-AS Options A, B, and C for Customer Green/Blue

#### Option A — R5↔R11, extending Green as a back-to-back-VRF redundant path

```
! R5 (AS65100 ASBR, acting as a GREEN edge member for this task only)
hostname R5
!
ip vrf GREEN
 rd 65100:910
 route-target export 65910:1
 route-target import 65910:1
!
interface GigabitEthernet1/0.910
 encapsulation dot1Q 910
 ip vrf forwarding GREEN
 ip address 172.31.59.5 255.255.255.252
!
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 172.31.59.11 remote-as 65200
  neighbor 172.31.59.11 activate
  neighbor 172.31.59.11 as-override
 exit-address-family
!
! R11 (AS65200 ASBR, acting as a GREEN edge member for this task only)
hostname R11
!
ip vrf GREEN
 rd 65200:910
 route-target export 65910:1
 route-target import 65910:1
!
interface GigabitEthernet1/0.910
 encapsulation dot1Q 910
 ip vrf forwarding GREEN
 ip address 172.31.59.11 255.255.255.252
!
router bgp 65200
 address-family ipv4 vrf GREEN
  neighbor 172.31.59.5 remote-as 65100
  neighbor 172.31.59.5 activate
  neighbor 172.31.59.5 as-override
 exit-address-family
```

> A dedicated dot1Q sub-interface (`.910`, numbered for the Green customer AS) is created on the otherwise-native R5↔R11 physical link, keeping Option A's per-VPN logical interface distinct from the existing native R5↔R11 IPv4-unicast eBGP session (Q1.4), which remains untouched/unaffected. Each ASBR treats the other as if it were a CE for VRF GREEN — eBGP is run between the two VRF instances directly, with `as-override` applied because, from R11's perspective, AS65100 now appears to "re-originate" GREEN routes under its own ASN boundary identically to how a real CE would, and the reverse eBGP session on the R5 side is itself sourced from inside AS65100 — matching the standard Option A pattern where neither side ever sees the original VPNv4 attributes, just a freshly eBGP-learned prefix. The existing Q2.1 Green PEs (R1, R16, R15) are untouched, making this strictly an additional redundant path as directed.

#### Option B — R12↔R21, Blue VPNv4 direct exchange with next-hop rewrite

```
! R12 (reuses the AF already enabled in Q1.4/Q1.5)
hostname R12
!
ip extcommunity-list standard RT-BLUE permit rt 65025:1
!
router bgp 65200
 address-family vpnv4
  neighbor 20.12.21.21 activate
  neighbor 20.12.21.21 send-community both
  neighbor 20.12.21.21 next-hop-self
 exit-address-family
!
! R21
hostname R21
!
router bgp 65300
 address-family vpnv4
  neighbor 20.12.21.12 activate
  neighbor 20.12.21.12 send-community both
  neighbor 20.12.21.12 next-hop-self
 exit-address-family
```

> `neighbor <ip> next-hop-self` under `address-family vpnv4` on both R12 and R21 is what makes this genuine Option B behavior — each ASBR rewrites the VPNv4 next-hop to itself before re-advertising, meaning the remote AS never resolves a label back to the originating PE's loopback directly; instead it resolves to the local ASBR's own loopback (which is reachable purely via that AS's own IGP/LDP, with no cross-AS label-switched path required). This is the key difference from Option C, where next-hop is explicitly preserved.

#### Option C — R6↔R20, multihop MP-eBGP VPNv4 between R7/R8 (AS65100 RR) and R20/R21/R30 (AS65300), next-hop-unchanged

```
! R7 (AS65100 RR) — multihop VPNv4 eBGP to R30 (AS65300 RR)
hostname R7
!
router bgp 65100
 neighbor 150.3.30.30 remote-as 65300
 neighbor 150.3.30.30 ebgp-multihop 5
 neighbor 150.3.30.30 update-source Loopback0
 neighbor 150.3.30.30 password CCIESPEBGPC
 address-family vpnv4
  neighbor 150.3.30.30 activate
  neighbor 150.3.30.30 send-community both
  neighbor 150.3.30.30 next-hop-unchanged
 exit-address-family
!
! R30 (AS65300 RR) — multihop VPNv4 eBGP to R7 (AS65100 RR)
hostname R30
!
router bgp 65300
 neighbor 150.1.7.7 remote-as 65100
 neighbor 150.1.7.7 ebgp-multihop 5
 neighbor 150.1.7.7 update-source Loopback0
 neighbor 150.1.7.7 password CCIESPEBGPC
 address-family vpnv4
  neighbor 150.1.7.7 activate
  neighbor 150.1.7.7 send-community both
  neighbor 150.1.7.7 next-hop-unchanged
 exit-address-family
```

> This multihop session rides over the loopback-to-loopback BGP-LU-labeled path built in Q1.5 (R7→...→R6→R20→...→R30, all label-switched, no IP lookup at R6/R20). `next-hop-unchanged` on both sides is what makes this true Option C: a Green VPNv4 route for R32 (behind R1, AS65100) keeps R1's own loopback as its BGP next-hop all the way into AS65300 — R20 and R30 resolve that next-hop only via the labeled IGP/BGP-LU path, never via a VPNv4 lookup, and critically **R6 and R20 themselves never run the VPNv4 AF at all** (confirmed by their configuration in Q1.4/Q1.5 only ever enabling `address-family ipv4` with `send-label`, never `address-family vpnv4`) — they only ever forward based on the BGP-LU-assigned MPLS label, exactly as required.

**Verification:**

```
! Option A
show ip bgp vpnv4 vrf GREEN
show bgp vpnv4 unicast vrf GREEN neighbors 172.31.59.11
! Option B
show ip bgp vpnv4 all neighbors 20.12.21.21 | include Next hop
show ip bgp vpnv4 vrf BLUE
! Option C
show ip bgp vpnv4 all neighbors 150.3.30.30
show ip bgp vpnv4 vrf GREEN 150.32.32.32
show mpls forwarding-table
```

Expected: Option A — R5 and R11 each show a locally-learned (not VPNv4-sourced) GREEN route from the other, with a brand-new eBGP-assigned next-hop (the `.910` sub-interface IP), proving no VPNv4 attributes crossed. Option B — `show ip bgp vpnv4 vrf BLUE` on R21 shows the Blue route's BGP next-hop as R12's own loopback (150.2.12.12), not R14's, confirming next-hop rewrite. Option C — `show ip bgp vpnv4 vrf GREEN 150.32.32.32` on R30/R20/R21 shows the next-hop still as R1's loopback (150.1.31.31) unchanged, and `show mpls forwarding-table` on R6/R20 shows zero VPNv4-sourced label entries — only BGP-LU/IGP label entries — proving the ASBRs carry no VPNv4 state for this path.

**Rationale:** The three Inter-AS options differ precisely in how much VPNv4 state and label-stack depth the ASBRs themselves must carry — Option A isolates each side completely (simplest/least scalable, each ASBR is a VRF edge), Option B exchanges VPNv4 directly but hides the original PE identity behind the ASBR's own next-hop (moderate scalability, ASBR becomes a per-prefix hop), and Option C pushes VPNv4 state all the way to the RRs/PEs so the ASBRs stay label-switching-only infrastructure (most scalable, matches large multi-AS SP designs). Demonstrating all three on different ASBR pairs, as directed, lets each be verified independently without interaction.

---

### Q2.5 — Hierarchical QoS egress policy on an ASBR

**Design notes:** The upstream marking (DSCP→EXP) must happen at the PE that imposes the VPN label stack, since a transit LSR/ASBR swapping labels mid-path cannot see the underlying IP DSCP once the packet already has an MPLS label stack — IOS by default copies the IP-precedence (3 MSBs of DSCP) into the MPLS EXP field at label imposition (`mpls experimental` default short-pipe behavior) unless explicitly overridden; here we demonstrate the Blue VPN explicitly to make the EXP marking traceable end-to-end, per the task's requirement to pick one VPN/traffic class and carry it through.

#### Upstream marking at R14 (Blue PE, label-imposition point for R12-bound traffic)

```
hostname R14
!
class-map match-any BLUE-CONTROL
 match ip dscp af31
!
policy-map MARK-BLUE-INGRESS
 class BLUE-CONTROL
  set mpls experimental imposition 3
!
interface FastEthernet3/0
 ip vrf forwarding BLUE
 service-policy input MARK-BLUE-INGRESS
```

> `set mpls experimental imposition 3` explicitly stamps EXP 3 onto the label R14 imposes for VRF BLUE traffic classified as AF31 — this is the "upstream marking policy" the task requires documenting/configuring at the originating PE so R12's EXP-based classification downstream actually has real, traceable traffic to match.

#### R12 — hierarchical (parent-shaper + child-LLQ/CBWFQ) egress policy on the R12↔R21 interface

```
hostname R12
!
class-map match-any VOICE-EQUIV
 match mpls experimental topmost 5
 match ip dscp ef
!
class-map match-any BLUE-CONTROL-EXP
 match mpls experimental topmost 3
 match ip dscp af31
!
policy-map CHILD-R12-R21
 class VOICE-EQUIV
  priority percent 10
 class BLUE-CONTROL-EXP
  bandwidth percent 30
  random-detect
 class class-default
  fair-queue
  random-detect
!
policy-map PARENT-R12-R21-SHAPE
 class class-default
  shape average percent 80
  service-policy CHILD-R12-R21
!
interface FastEthernet3/0
 description Inter-AS link to R21 (AS65300) - HQoS egress
 service-policy output PARENT-R12-R21-SHAPE
```

> `match mpls experimental topmost` classifies on the top (outermost) label's EXP bits, which is the field R12 can always see regardless of whether it is imposing a fresh label (PE-originated traffic) or swapping labels as a transit LSR for traffic already in the MPLS core — this is precisely why the task calls out that classification must work "against the MPLS EXP bits on the top label for transit LSP traffic, not only against the underlying IP DSCP." Each `class-map match-any` also matches the IP DSCP equivalent so that any traffic still visible at the IP layer (e.g., if it happens to reach R12 unlabeled) is classified consistently with its labeled counterpart. The parent shaper caps aggregate egress at 80% of the interface's configured bandwidth; the nested child policy's `priority percent 10` and `bandwidth percent 30` are calculated relative to that shaped (not raw interface) rate, as required.

**Verification:**

```
show policy-map interface FastEthernet3/0
show mpls forwarding-table detail
show class-map
```

Expected: `show policy-map interface FastEthernet3/0` on R12 shows the parent `shape average` queue with the nested child policy's three classes, each showing non-zero matched packet/byte counters once Blue/voice-equivalent test traffic is sent, and `WRED`/`random-detect` drop counters incrementing only under congestion. `show mpls forwarding-table detail` on R14 shows the imposed label's EXP field set to 3 for Blue/AF31-classified traffic.

**Rationale:** A two-level (parent shape / child priority+bandwidth) HQoS policy is required because a flat policy cannot both shape the aggregate rate and give proportional/priority treatment within that shaped envelope — IOS MQC requires the shaper to be the parent with the class-based scheduling policy nested as the child. Marking at label imposition (the PE) rather than relying on EXP being auto-derived mid-core guarantees R12's classification is deterministic and intentional, not an accidental artifact of default precedence-to-EXP copying.

---

## Section 3: Access Connectivity (25%)

### Q3.1 — Multicast VPN (mVPN) for Customer Yellow

#### Global/core PIM-SSM — R20, R22 (AS65300) and R16 (AS65200), plus core P/PE routers in those ASes

```
! R20 (and identically on R21, R22, R23, R30 — AS65300 core multicast)
hostname R20
!
ip multicast-routing
ip pim ssm default
!
interface GigabitEthernet1/0
 ip pim sparse-mode
!
interface GigabitEthernet2/0
 ip pim sparse-mode
```

> `ip pim ssm default` enables PIM-SSM for the default SSM range 232.0.0.0/8 network-wide without needing an explicit ACL — this is the IOS-classic equivalent of "ip pim ssm range" with no range specified, defaulting to 232/8 per RFC 4607. Apply `ip pim sparse-mode` on every AS65300 core-facing interface (same set as Q1.1's IS-IS interfaces) and identically on every AS65200 core interface on R11, R12, R13, R14, R15, R16, R19 (same set as Q1.2's OSPF interfaces).

#### Default MDT for VRF YELLOW — R20, R22 (AS65300)

```
hostname R20
!
ip vrf YELLOW
 rd 65300:24
 route-target export 65024:1
 route-target import 65024:1
 mdt default 239.232.24.1
!
ip multicast-routing vrf YELLOW
!
interface FastEthernet3/0
 ip vrf forwarding YELLOW
 ip address 192.168.9.20 255.255.255.0
 ip pim sparse-mode
```

> Identical `mdt default 239.232.24.1` line added to R22's and R16's existing `ip vrf YELLOW` stanza (from Q2.1), plus `ip multicast-routing vrf YELLOW` and `ip pim sparse-mode` on each PE's customer-facing YELLOW interface. The MDT source defaults to the PE's Loopback0 automatically once `ip multicast-routing vrf YELLOW` is enabled and the PE has a core multicast-enabled loopback — no separate `mdt source` directive is required in classic default-MDT GRE mode as long as Loopback0 is the natural RPF source, but it is best practice to pin it explicitly:

```
ip vrf YELLOW
 mdt source Loopback0
```

#### CE-side PIM-SSM + BSR on R24 (Yellow CE)

```
hostname R24
!
ip multicast-routing
ip pim ssm default
!
interface FastEthernet3/0
 ip address 192.168.9.24 255.255.255.0
 ip pim sparse-mode
!
interface FastEthernet0/0
 ip address 192.168.11.24 255.255.255.0
 ip pim sparse-mode
!
ip pim bsr-candidate FastEthernet3/0 0
ip pim rp-candidate FastEthernet3/0
```

> R24 runs both SSM (toward its own internal hosts and the SP MDT-facing PE interfaces) and a BSR/C-RP candidacy for any legacy ASM groups the customer's internal hosts may still use — this models the "customer still needs ASM/BSR independent of the SP's SSM-only core" requirement explicitly. The BSR scope is local to the YELLOW customer's own routers (R24 and anything further behind it) and never needs to be carried into the SP core, since the SP core only ever sees the GRE-encapsulated default-MDT traffic, never raw customer PIM/BSR messages.

**Verification:**

```
show ip pim vrf YELLOW mdt
show ip mroute vrf YELLOW
show ip pim neighbor
show ip pim rp mapping
```

Expected: `show ip pim vrf YELLOW mdt` on R20/R22/R16 shows all three PEs as members of the same default-MDT group (239.232.24.1) with the GRE tunnel interface `Up`. `show ip mroute vrf YELLOW` after a test multicast stream sourced behind R24 shows an `(S,G)` entry with the Tunnel0 (or equivalent auto-created MDT tunnel) interface in the outgoing interface list toward R16's YELLOW customer-facing interface. `show ip pim rp mapping` on R24 shows itself as the BSR/C-RP for any ASM groups configured, entirely independent of the SP core's SSM state.

**Rationale:** Profile 0 (draft-rosen GRE MDT) is the only mVPN profile broadly supported on IOS classic 15.2 without requiring BGP-AD/MLDP infrastructure, matching the Dynamips/15.2(4)M11 platform constraint. PIM-SSM in the core (rather than ASM/PIM-SM with an RP) avoids needing an RP election/Anycast-RP design just to carry MDT trees, and is the modern best-practice core-multicast transport. The CE-side BSR/ASM capability is kept entirely separate from and invisible to the SP's SSM core, demonstrating that customer-internal multicast complexity doesn't need to leak into or complicate the SP's own multicast design.

---

### Q3.2 — Site of Origin: fix the dual-homed Yellow routing loop

**Before the fix:** R24's own locally-originated/connected prefix 192.168.9.0/24 (learned by R20 via eBGP from R24) and 192.168.11.0/24 (learned by R22 via eBGP from R24) would each propagate via VPNv4/iBGP through R30 (the AS65300 RR) to the other PE — meaning R22 would learn 192.168.9.0/24 (originated on the "wrong side," R20's link) via VPNv4, and could then re-advertise it right back down its own eBGP session to R24. R24 would then see its neighbor's own connected subnet reflected back to it from a path that makes no topological sense (a backdoor loop), and in degenerate cases this could also apply to R24's own loopback or other locally-sourced prefixes if R24 advertises more than just its connected subnet.

#### R20 — SoO on the YELLOW PE-CE session to R24

```
hostname R20
!
ip extcommunity-list standard SOO-R24 permit soo 65300:24
!
route-map SET-SOO-R24-IN permit 10
 set extcommunity soo 65300:24
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.9.24 route-map SET-SOO-R24-IN in
 exit-address-family
```

#### R22 — SoO on the YELLOW PE-CE session to R24

```
hostname R22
!
ip extcommunity-list standard SOO-R24 permit soo 65300:24
!
route-map SET-SOO-R24-IN permit 10
 set extcommunity soo 65300:24
!
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.11.24 route-map SET-SOO-R24-IN in
 exit-address-family
```

> Applying `set extcommunity soo 65300:24` **inbound** on each PE's eBGP session to R24 tags every route learned from R24 with that SoO value. BGP's standard, automatic SoO behavior then prevents a route carrying a given SoO from ever being re-advertised out an eBGP session that is itself configured with that same SoO value — so R22, having received 192.168.9.0/24 via VPNv4 (tagged SoO 65300:24 by R20), will not re-advertise it back out its own R24-facing session (also tagged SoO 65300:24), and vice-versa for R20. Crucially, routes for **other** YELLOW sites (e.g., anything originated by R26 behind R16) carry no such SoO value, so they are unaffected and continue to reach R24 normally from both R20 and R22 — preserving R24's outbound dual-homed redundancy for VPN-sourced-elsewhere routes exactly as required.

**Verification:**

```
show ip bgp vpnv4 vrf YELLOW
show ip bgp vpnv4 vrf YELLOW 192.168.9.0
show ip bgp vpnv4 vrf YELLOW 192.168.11.0
show ip bgp neighbors 192.168.9.24 advertised-routes
show ip bgp neighbors 192.168.11.24 advertised-routes
```

Expected (post-fix): `show ip bgp vpnv4 vrf YELLOW 192.168.9.0` on R22 shows the route learned via VPNv4 from R20 with extended community `SoO:65300:24` attached, but `show ip bgp neighbors 192.168.11.24 advertised-routes` on R22 does **not** list 192.168.9.0/24 — confirming the SoO-tagged route is correctly suppressed from re-advertisement back to R24. Symmetrically on R20. Both `show ip bgp neighbors 192.168.9.24 advertised-routes` (R20) and `show ip bgp neighbors 192.168.11.24 advertised-routes` (R22) continue to show YELLOW routes sourced from R26 (behind R16), confirming redundancy for other sites' routes is preserved. Prior to the fix (as a transient test), the same `advertised-routes` output would have shown the "wrong-direction" prefix — this is the specific looping behavior the task asks to be noted and confirmed resolved.

**Rationale:** SoO is purpose-built for exactly this dual/multi-homed-CE scenario — it tags a route with "the site it came from" so BGP can categorically suppress sending it back to that same site via a different path, without needing any manual filtering logic or affecting any other route in the VPN. Applying it inbound at the PE-CE boundary (rather than, e.g., only on the RR) ensures the SoO tag travels with the route through the entire VPNv4/iBGP path and is enforced at every PE that could potentially re-advertise it toward R24.

---

### Q3.3 — PE-CE routing and redistribution hygiene

**Design notes:** Most of this task is satisfied structurally by the way Q2.1 was built (narrow, VRF-scoped `redistribute connected`, no blanket global redistribution, strict RT-based VPN isolation). This task adds the OSPF domain-ID consolidation note and the mandatory max-prefix 50 on every PE-CE eBGP session.

#### Redistribution scope — confirmed design (no new config; summary of what's already true from Q2.1)

Every VRF's `redistribute connected` statement lives under that VRF's own `address-family ipv4 vrf <name>` and therefore can only ever touch connected interfaces that are themselves members of that VRF via `ip vrf forwarding <name>` — since each PE-CE link's connected subnet is the *only* interface in that VRF on that router, redistribution is inherently scoped to exactly the PE-CE link's own subnet, with no additional candidate connected routes available to leak. No VRF anywhere in this lab redistributes `static` routes (none are configured) and no VRF redistributes another VRF's routes (RT import/export is the only cross-VRF linkage, and RTs are unique per VPN: `65910:1` Green, `65024:1` Yellow, `65025:1` Blue — never shared).

#### OSPF domain-ID consistency — R15 (confirming Q2.1's existing config)

```
hostname R15
!
router ospf 1 vrf GREEN
 domain-id 0.0.0.1
```

> Already configured in Q2.1 — restated here to confirm the domain-ID (`0.0.0.1`) is the single value that must be reused verbatim if a second GREEN OSPF PE is ever added, since domain-ID mismatch between two OSPF PE-CE instances in the same VRF/VPN is what would cause routes crossing the backbone to be misclassified as native intra-area instead of correctly DN-bit-marked external/VPN routes.

#### Max-prefix 50 on every PE-CE eBGP session

```
! R1
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.1.9 maximum-prefix 50
  neighbor 192.168.2.10 maximum-prefix 50
 exit-address-family
 address-family ipv4 vrf BLUE
  neighbor 192.168.3.32 maximum-prefix 50
 exit-address-family
!
! R2
router bgp 65100
 address-family ipv4 vrf GREEN
  neighbor 192.168.4.10 maximum-prefix 50
 exit-address-family
!
! R14
router bgp 65200
 address-family ipv4 vrf BLUE
  neighbor 192.168.5.25 maximum-prefix 50
 exit-address-family
!
! R16
router bgp 65200
 address-family ipv4 vrf GREEN
  neighbor 192.168.7.18 maximum-prefix 50
 exit-address-family
 address-family ipv4 vrf YELLOW
  neighbor 192.168.8.26 maximum-prefix 50
 exit-address-family
!
! R20
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.9.24 maximum-prefix 50
 exit-address-family
!
! R21
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.10.25 maximum-prefix 50
 exit-address-family
!
! R22
router bgp 65300
 address-family ipv4 vrf YELLOW
  neighbor 192.168.11.24 maximum-prefix 50
 exit-address-family
!
! R23
router bgp 65300
 address-family ipv4 vrf BLUE
  neighbor 192.168.12.25 maximum-prefix 50
 exit-address-family
```

> Already present in each VRF's eBGP neighbor stanza shown in Q2.1's configs above (`maximum-prefix 50` lines) — restated here as a consolidated per-session checklist against the task's explicit list: R31↔R9, R31↔R10, R31↔R32, R2↔R10, R14↔R25, R16↔R18, R16↔R26, R20↔R24, R21↔R25, R22↔R24, R23↔R25. All eleven sessions are covered.

**Verification:**

```
show ip bgp vpnv4 vrf GREEN neighbors 192.168.1.9 | include maximum
show ip route vrf YELLOW
show ip route vrf BLUE
show ip ospf vrf GREEN database external
```

Expected: each `show ... neighbors <ce-ip>` output includes a line like `Maximum prefixes allowed 50` with a current prefix count well under that limit in normal operation. `show ip route vrf YELLOW` on any YELLOW PE never shows a Green or Blue CE prefix (150.9.9.9, 150.10.10.10, 150.17.17.17, 150.18.18.18, 150.25.25.25, 150.32.32.32) — confirming VPN isolation holds. `show ip ospf vrf GREEN database external` on R15 shows R17's redistributed-from-BGP routes carrying the DN-bit and the `0.0.0.1` domain-ID extended community.

**Rationale:** Capping PE-CE sessions at a modest, consistent max-prefix value is a standard SP hygiene control against a misbehaving or compromised CE flooding a VRF table (and, transitively, the VPNv4 backbone) with excessive routes. Confirming redistribution scope and VPN isolation by construction (rather than needing an explicit filter) demonstrates the design is correct by the way VRFs and RTs were built in Q2.1, not merely by accident.

---

### Q3.4 — Access-side QoS classification (CE-facing)

#### R16 — ingress classification/marking on three customer-facing ACs

```
hostname R16
!
class-map match-any ALL-IP
 match any
!
policy-map MARK-GREEN-INGRESS
 class ALL-IP
  set dscp af21
!
policy-map MARK-YELLOW-INGRESS
 class ALL-IP
  set dscp af31
!
interface GigabitEthernet1/0
 ip vrf forwarding GREEN
 ip address 192.168.7.16 255.255.255.0
 service-policy input MARK-GREEN-INGRESS
!
interface GigabitEthernet2/0
 ip vrf forwarding YELLOW
 ip address 192.168.8.16 255.255.255.0
 service-policy input MARK-YELLOW-INGRESS
!
interface FastEthernet3/0
 no ip address
!
interface FastEthernet3/0.100
 encapsulation dot1Q 100
 bridge-domain 100
 mls qos trust cos
 service-policy input MARK-RED-COS
!
class-map match-any RED-L2
 match cos 0 1 2 3 4 5 6 7
!
policy-map MARK-RED-COS
 class RED-L2
  set cos 3
  set mpls experimental imposition 3
```

> `set cos 3` marks the 802.1p field directly on the dot1Q-tagged VLAN 100 frame ingressing from R27, since this AC is a pure L2 attachment (no routed IP), matching the task's explicit instruction to mark CoS rather than DSCP here. `set mpls experimental imposition 3` on the same policy-map ensures that marking is carried forward into the EXP field of the pseudowire label R16 imposes for this VPLS AC traffic, giving one consistent, traceable CoS↔EXP marking scheme — matching EXP 3 used for Blue/AF31 in Q2.5, so the overall lab-wide marking scheme is unified (AF21/AF31/CoS3 all map to a documented, consistent "class 3-equivalent" EXP marking across both L3VPN and L2VPN/VPLS traffic), even though R12 itself never actually carries Red VPLS traffic on its own interfaces.

**Verification:**

```
show policy-map interface GigabitEthernet1/0
show policy-map interface GigabitEthernet2/0
show policy-map interface FastEthernet3/0.100
show mpls l2transport vc detail
```

Expected: `show policy-map interface GigabitEthernet1/0` shows the `MARK-GREEN-INGRESS` policy with non-zero matched packets once Green CE traffic from R18 arrives, confirming AF21 marking. Similarly for YELLOW (AF31) on Gi2/0. `show mpls l2transport vc detail` on R16's VLAN 100 VPLS PW shows the EXP field in its imposed label statistics reflecting value 3 for marked frames from R27.

**Rationale:** Marking as close to the network edge as possible (at ingress on the CE-facing AC) ensures consistent, predictable QoS treatment is available for every downstream policy (including R12's Q2.5 egress HQoS, even though it doesn't carry this specific Red traffic) without requiring every transit node to re-classify from scratch. Using CoS instead of DSCP for the L2-only Red AC respects the fact that this traffic has no IP header visible to the PE at the attachment circuit — 802.1p is the only classification field actually present on the wire there.

---

## Section 4: High Availability (10%)

### Q4.1 — MPLS Traffic Engineering with explicit path and FRR

#### Enable MPLS-TE extensions network-wide (R31, R2-R8)

```
hostname R1
!
mpls traffic-eng tunnels
!
router isis
 mpls traffic-eng level-2
 mpls traffic-eng router-id Loopback0
!
interface GigabitEthernet1/0
 mpls traffic-eng tunnels
```

> Apply `mpls traffic-eng tunnels` globally and under every AS65100 core-facing interface, plus `mpls traffic-eng level-2` + `mpls traffic-eng router-id Loopback0` under `router isis` identically on R31, R2, R3, R4, R5, R6, R7, R8 — this floods TE link-state (available bandwidth, affinity, etc.) into IS-IS so CSPF has topology data to compute explicit/constrained paths.

#### Explicit path + Tunnel1 — R1

```
hostname R1
!
ip explicit-path name R1-R6-VIA-R3-R5 enable
 next-address 10.1.3.3
 next-address 10.3.5.5
 next-address 10.5.6.6
!
interface Tunnel1
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.6.6
 tunnel mpls traffic-eng path-option 1 explicit-name R1-R6-VIA-R3-R5
 tunnel mpls traffic-eng bandwidth 10000
 tunnel mpls traffic-eng autoroute announce
 tunnel mpls traffic-eng fast-reroute
```

> `next-address` hops in the explicit path are the strict, in-order next-hops R31→R3→R5→R6, deliberately excluding R2/R4 so the LSP never takes the R31→R2→R4→R6 alternate. `bandwidth 10000` reserves 10 Mbps (units are Kbps in this command, so `10000` = 10 Mbps) along the path. `autoroute announce` causes IGP-shortest-path traffic from R1 toward R6-reachable destinations to prefer this tunnel once up. `fast-reroute` on the tunnel signals FRR desired/required along the LSP's path, needed for R3's midpoint backup tunnel to actually protect it.

#### FRR backup tunnel at R3, protecting the R3→R5 link

```
hostname R3
!
interface Tunnel100
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 150.1.5.5
 tunnel mpls traffic-eng path-option 1 explicit-name R3-BYPASS-R5-VIA-R4-R6
 tunnel mpls traffic-eng fast-reroute
!
ip explicit-path name R3-BYPASS-R5-VIA-R4-R6 enable
 next-address 10.3.4.4
 next-address 10.4.6.6
 next-address 10.5.6.5
!
interface GigabitEthernet2/0
 mpls traffic-eng backup-path Tunnel100
```

> The backup bypasses the protected R3→R5 link via R3→R4→R6→R5 (using the already-validated R3-R4 Gi1/0 and R4-R6 Gi2/0 links plus the R5-R6 Fa3/0 link), terminating at R5 — this is **link protection** (the backup's destination matches the next-hop of the protected link, R5, not the ultimate tunnel endpoint R6), satisfying the task's specific link-protection requirement rather than node protection. `mpls traffic-eng backup-path Tunnel100` applied under R3's Gi2/0 (the R3-R5 physical interface) is what binds this backup tunnel to protect that specific link for any primary TE LSP, including Tunnel1, that signals FRR and transits this link.

**Verification:**

```
show mpls traffic-eng tunnels tunnel1
show mpls traffic-eng fast-reroute database
show mpls traffic-eng tunnels backup
show ip route 150.1.6.6
```

Expected: `show mpls traffic-eng tunnels tunnel1` on R31 shows `Tunnel1` state `up/up`, with the signaled path listing R31→R3→R5→R6 explicitly and bandwidth 10000 reserved. `show mpls traffic-eng fast-reroute database` on R3 shows Tunnel1 listed with a `Backup Tunnel` reference to Tunnel100 and protection state `Ready` (not merely configured) — the task explicitly warns that "configured but not providing protection" does not satisfy the requirement, so `Ready`/`Protected` status in this output (not just the backup tunnel existing in `show mpls traffic-eng tunnels backup`) is the actual pass criterion. `show ip route 150.1.6.6` on R31 shows the route resolving via Tunnel1 once up (autoroute announce), rather than via the native IGP SPF next-hop.

**Rationale:** An explicit strict-hop path guarantees the LSP takes exactly the intended route regardless of any IGP metric changes, satisfying the deliberate-avoidance-of-the-R2/R4-path requirement unambiguously. Bandwidth reservation and `autoroute announce` make the tunnel actually carry traffic rather than existing only as an idle signaled LSP. Link (not node) protection at R3 specifically targets the stated single-point-of-failure link, and verifying `Ready`/active backup state (not just configuration presence) is what distinguishes a working FRR deployment from a merely-configured one.

---

### Q4.2 — BFD for eBGP between ASBRs

#### BFD intervals + eBGP fall-over on each of the four inter-AS ASBR pairs

```
! R5
interface GigabitEthernet1/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65100
 neighbor 10.5.11.11 fall-over bfd
!
! R11
interface GigabitEthernet1/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65200
 neighbor 10.5.11.5 fall-over bfd
!
! R6
interface GigabitEthernet1/0
 bfd interval 150 min_rx 150 multiplier 3
interface FastEthernet0/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65100
 neighbor 10.6.12.12 fall-over bfd
 neighbor 10.6.20.20 fall-over bfd
!
! R12
interface GigabitEthernet1/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65200
 neighbor 10.6.12.6 fall-over bfd
!
! R20
interface FastEthernet0/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65300
 neighbor 10.6.20.6 fall-over bfd
!
! R12 (second BFD-enabled interface, toward R21)
interface FastEthernet3/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65200
 neighbor 20.12.21.21 fall-over bfd
!
! R21
interface GigabitEthernet1/0
 bfd interval 150 min_rx 150 multiplier 3
!
router bgp 65300
 neighbor 20.12.21.12 fall-over bfd
```

> Same 150/150/3 BFD profile as Q1.2's OSPF BFD, applied here on the four physical inter-AS interfaces and registered against each ASBR pair's eBGP session via `neighbor <ip> fall-over bfd`.

**Verification:**

```
show bfd neighbors
show ip bgp summary
```

Expected: `show bfd neighbors` on each ASBR shows the opposite ASBR's directly-connected IP in state `Up` with the 150ms interval. Shutting down one direction of, e.g., R6's Gi1/0 (toward R12) should cause `show bfd neighbors` to immediately show the session drop and the corresponding `show ip bgp summary` eBGP session to R12 transition to `Idle`/re-establishing within well under 1 second — observable via `debug bfd` timestamps or simply the near-instant disappearance of the BGP session from `Established` state, rather than waiting out the default 180s BGP hold-timer.

**Rationale:** Without BFD, eBGP relies on either TCP retransmission timeout or the (default 180s, often tuned to 90s/30s at best) BGP hold-timer to detect a dead peer — far too slow for an SP core where sub-second failover is expected. BFD's lightweight, fast hello mechanism detects the failure at the data-link/interface level and immediately signals BGP to tear down the session, achieving the sub-second convergence the task requires.

---

### Q4.3 — NSF/NSR concepts (written-answer style task, no configuration)

```
! Lab notebook notes — Q4.3 (no configuration applicable; Dynamips has no dual-RP emulation)

! --- NSF vs NSR ---
! NSF (Nonstop Forwarding): the data plane (CEF/FIB) keeps forwarding packets
!   using the last-known-good forwarding table while the control plane restarts
!   (RP switchover, process restart, etc.). NSF REQUIRES cooperation from
!   NEIGHBORING routers via graceful-restart signaling: the restarting router
!   tells its neighbors "I'm restarting, please keep our adjacency/routes alive
!   and don't tear down session state," and the neighbor (acting as an
!   NSF/graceful-restart "helper") holds the old routes/adjacency for a grace
!   period while the restarting router's control plane rebuilds.
!
! NSR (Nonstop Routing): the STANDBY RP maintains a fully synchronized copy of
!   protocol state (IS-IS/OSPF/BGP adjacency and RIB state, not just the FIB)
!   continuously, in real time, via inter-process communication between
!   active and standby RP — so on a switchover, the standby simply takes over
!   as if nothing happened. NSR requires NO neighbor awareness or cooperation
!   at all — from the neighbor's perspective, nothing happened, because the
!   protocol session itself never actually restarted.
!
! Key distinction: NSF = "forwarding survives, control plane restarts, and
!   neighbors must help via graceful restart." NSR = "control plane itself
!   never restarts because the standby already has synchronized state, so
!   neighbors need not know or help at all."

! --- Per-protocol neighbor-side helper capability (real HA hardware only) ---
! IS-IS: neighbor must support IS-IS graceful restart (RFC 5306, "IS-IS
!   Restart Signaling"). On real Cisco HA platforms (ASR9k/CRS/7600 Sup720),
!   the helper capability is implicit once "isis nsf cisco" or "isis nsf ietf"
!   is enabled on the restarting router; the NEIGHBOR simply needs the IS-IS
!   process itself to support/not disable the capability (there's no separate
!   explicit "helper-only" command on the neighbor side in classic IOS --
!   support is a baseline per-platform capability, advertised automatically
!   via the IS-IS restart TLV).
!
! OSPF: neighbor must support the OSPF graceful-restart helper role (RFC 3623
!   "OSPF Graceful Restart" / Cisco "OSPF NSF"). Enabled on the neighbor via:
!     router ospf <pid>
!      nsf ietf helper disable   ! (disables; helper is ON by default on
!                                   platforms that support it)
!   i.e., the neighbor is a helper by default once it supports NSF-aware OSPF
!   code -- the explicit command is normally only used to DISABLE helper mode,
!   not to enable it.
!
! BGP: neighbor must support BGP graceful restart (RFC 4724), negotiated via
!   BGP capability exchange at session establishment. Enabled on the neighbor
!   (the helper) via:
!     router bgp <asn>
!      bgp graceful-restart
!   This must be configured on BOTH the restarting router and its neighbors --
!   the capability is only negotiated (and thus only usable) if both sides
!   advertise it in the BGP OPEN message.

! --- Platform note ---
! None of the above is testable in this Dynamips/7200 lab since Dynamips
! emulates only a single RP per router -- there is no dual-RP/dual-supervisor
! hardware model to switch over, so "nsf" / "bgp graceful-restart" commands
! may be accepted by the CLI (parser-level) but have no real switchover
! behavior to validate. This task is answered conceptually only, per the
! lab's own instruction.
```

**Verification:** Not applicable — this is a conceptual/written task per the lab's explicit instruction; no configuration exists to verify on Dynamips.

**Rationale:** Understanding the NSF/NSR distinction (data-plane continuity with neighbor-assisted graceful restart vs. fully synchronized standby control-plane state with no neighbor involvement) and which side of a graceful-restart exchange is the "restarter" versus the "helper" is foundational CCIE-SP HA knowledge, independent of whether the specific lab platform can physically demonstrate it.

---
