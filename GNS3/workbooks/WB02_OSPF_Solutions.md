# WB02 OSPF — Solutions

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11 — IOS Classic
**Reference:** `00_topology_reference.md` for IP addressing and wiring.
**Scope:** OSPF **Area 0** on AS 65200 only (R11, R12, R13, R14, R15, R16, R19).

> Router-ID pinned from Loopback0. Loopback0 passive. All core links set to
> `ip ospf network point-to-point` (no DR/BDR, no type-2 LSAs on transit).
> Core interfaces only — inter-AS links (R11 g1/0→R5, R12 g1/0→R6, R12 f3/0→R21)
> and PE-CE links (R14 f3/0→R25, R15 f0/0→R17, R16 g1/0/g2/0, L2 ports) are NOT in OSPF.

---

## Section 1: OSPF Area 0 on AS 65200 (R11–R16, R19)

### R11 (P) — RID 150.2.11.11

```
router ospf 1
 router-id 150.2.11.11
 passive-interface Loopback0
 network 150.2.11.11 0.0.0.0 area 0
 network 20.11.12.11 0.0.0.0 area 0
 network 20.11.13.11 0.0.0.0 area 0
 network 20.11.15.11 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
interface GigabitEthernet2/0
 ip ospf network point-to-point
!
interface FastEthernet3/0
 ip ospf network point-to-point
!
```

> R11 core: f0/0 (→R12), g2/0 (→R13), f3/0 (→R15). Inter-AS g1/0 (→R5) NOT in OSPF.

### R12 (ASBR) — RID 150.2.12.12

```
router ospf 1
 router-id 150.2.12.12
 passive-interface Loopback0
 network 150.2.12.12 0.0.0.0 area 0
 network 20.11.12.12 0.0.0.0 area 0
 network 20.12.14.12 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
interface GigabitEthernet2/0
 ip ospf network point-to-point
!
```

> R12 core: f0/0 (→R11), g2/0 (→R14). Inter-AS g1/0 (→R6) and f3/0 (→R21) NOT in OSPF.

### R13 (P) — RID 150.2.13.13

```
router ospf 1
 router-id 150.2.13.13
 passive-interface Loopback0
 network 150.2.13.13 0.0.0.0 area 0
 network 20.13.16.13 0.0.0.0 area 0
 network 20.13.14.13 0.0.0.0 area 0
 network 20.11.13.13 0.0.0.0 area 0
 network 20.13.19.13 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
interface GigabitEthernet1/0
 ip ospf network point-to-point
!
interface GigabitEthernet2/0
 ip ospf network point-to-point
!
interface FastEthernet3/0
 ip ospf network point-to-point
!
```

> R13 core: f0/0 (→R16), g1/0 (→R14), g2/0 (→R11), f3/0 (→R19). All core — R13 is a P router.

### R14 (PE) — RID 150.2.14.14

```
router ospf 1
 router-id 150.2.14.14
 passive-interface Loopback0
 network 150.2.14.14 0.0.0.0 area 0
 network 20.14.19.14 0.0.0.0 area 0
 network 20.13.14.14 0.0.0.0 area 0
 network 20.12.14.14 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
interface GigabitEthernet1/0
 ip ospf network point-to-point
!
interface GigabitEthernet2/0
 ip ospf network point-to-point
!
```

> R14 core: f0/0 (→R19), g1/0 (→R13), g2/0 (→R12). PE-CE f3/0 (→R25) and L2 f4/0 (→R28) NOT in OSPF.

### R15 (PE) — RID 150.2.15.15

```
router ospf 1
 router-id 150.2.15.15
 passive-interface Loopback0
 network 150.2.15.15 0.0.0.0 area 0
 network 20.11.15.15 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet3/0
 ip ospf network point-to-point
!
```

> R15 core: f3/0 (→R11). PE-CE f0/0 (→R17) is a *customer* OSPF Area 0 (separate VRF/context in
> the L3VPN workbook) — do NOT place it in the core OSPF 1 process here.

### R16 (PE) — RID 150.2.16.16

```
router ospf 1
 router-id 150.2.16.16
 passive-interface Loopback0
 network 150.2.16.16 0.0.0.0 area 0
 network 20.13.16.16 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
```

> R16 core: f0/0 (→R13). PE-CE g1/0 (→R18), g2/0 (→R26) and L2 f3/0 (→R27) NOT in OSPF.

### R19 (RR) — RID 150.2.19.19

```
router ospf 1
 router-id 150.2.19.19
 passive-interface Loopback0
 network 150.2.19.19 0.0.0.0 area 0
 network 20.14.19.19 0.0.0.0 area 0
 network 20.13.19.19 0.0.0.0 area 0
!
interface Loopback0
 ip ospf 1 area 0
!
interface FastEthernet0/0
 ip ospf network point-to-point
!
interface FastEthernet3/0
 ip ospf network point-to-point
!
```

> R19 core: f0/0 (→R14), f3/0 (→R13).

### Verification

```
show ip ospf neighbor                         ! all adjacencies FULL, RID = loopback
show ip ospf interface brief                  ! core links P2P, cost, no DR/BDR
show ip route ospf                             ! all 7 loopbacks (150.2.x.x) reachable
show ip ospf database router                   ! no network (type-2) LSAs on P2P transit links
ping 150.2.16.16 source 150.2.11.11           ! R11 to R16 end-to-end
```

---

## Section 2: OSPF Advanced

### MD5 Authentication (area-wide, key "CCIE")

Apply on ALL AS 65200 routers (R11–R16, R19):

```
router ospf 1
 area 0 authentication message-digest
!
! On every OSPF *core* interface (example R11):
interface FastEthernet0/0
 ip ospf message-digest-key 1 md5 CCIE
!
interface GigabitEthernet2/0
 ip ospf message-digest-key 1 md5 CCIE
!
interface FastEthernet3/0
 ip ospf message-digest-key 1 md5 CCIE
!
```

> `area 0 authentication message-digest` turns on MD5 for the whole area; the per-interface
> `message-digest-key 1 md5 CCIE` supplies the key. Both ends of every adjacency must match —
> a mismatch drops the adjacency (verify with `debug ip ospf adj`). Loopback is passive, so no
> key needed there, but configuring it is harmless.

### BFD (all AS 65200 core links)

```
! On each OSPF core interface (example R13):
interface FastEthernet0/0
 bfd interval 500 min_rx 500 multiplier 3
!
interface GigabitEthernet1/0
 bfd interval 500 min_rx 500 multiplier 3
!
interface GigabitEthernet2/0
 bfd interval 500 min_rx 500 multiplier 3
!
interface FastEthernet3/0
 bfd interval 500 min_rx 500 multiplier 3
!
router ospf 1
 bfd all-interfaces
!
```

> `bfd all-interfaces` under the OSPF process registers OSPF as a BFD client on every OSPF
> interface. Sub-second detection without lowering OSPF hello/dead. Note: Dynamips may not
> fully support BFD — configure for learning; session may stay down on emulated platforms.

### SPF Throttle Tuning (all AS 65200 routers)

```
router ospf 1
 timers throttle spf 50 100 5000
 timers throttle lsa 10 100 5000
 timers lsa arrival 80
!
```

> `timers throttle spf 50 100 5000`: first SPF at 50 ms, hold 100 ms (doubling), max 5000 ms.
> Fast initial reaction, exponential back-off to damp churn. LSA throttle/arrival keep LSA
> generation and acceptance in step with the SPF cadence. Apply consistently domain-wide.

### Prefix Suppression (all AS 65200 routers)

```
router ospf 1
 prefix-suppression
!
```

> Suppresses transit (core /24) prefixes from the OSPF LSAs so only loopback /32s and
> stub/service prefixes remain reachable — standard SP practice; shrinks the LSDB/RIB and
> hides transit links. Per-interface override: `ip ospf prefix-suppression [disable]`.
> Verify core 20.x /24s drop from remote `show ip route ospf` while 150.2.x.x/32 stay.

### Verification (advanced)

```
show ip ospf interface FastEthernet0/0 | include authentication   ! "Message digest" + key id
show bfd neighbors                                                 ! BFD sessions Up (if supported)
show ip ospf | include SPF                                         ! SPF throttle values in effect
show ip route ospf | include 20\.                                  ! transit /24s gone (suppressed)
show ip route ospf | include 150\.2\.                              ! loopback /32s still present
```

---

## Section 3: Troubleshooting

### Adjacency stuck in EXSTART/EXCHANGE
- **Cause:** MTU mismatch — OSPF exchanges DBD with the interface MTU; mismatch stalls at EXSTART.
  Check `show ip ospf interface` MTU on both ends. Fix: match MTU, or `ip ospf mtu-ignore` on the
  interface as a workaround.

### Adjacency never leaves INIT / 2-WAY, or never forms
- **Cause 1:** Network-type mismatch — one side P2P, other broadcast. Both ends need the same type.
- **Cause 2:** Hello/dead timer mismatch (changes with network type). Check `show ip ospf interface`.
- **Cause 3:** Authentication mismatch — one side has MD5, other doesn't, or wrong key.

### Routes missing
- **Cause 1:** Interface not in OSPF — missing `network` statement (or `ip ospf 1 area 0`).
- **Cause 2:** Duplicate Router-ID — two routers claim the same RID → LSAs overwrite, routes flap.
  Fix by pinning unique `router-id` from each loopback and `clear ip ospf process`.
- **Cause 3:** Prefix-suppression hiding a prefix you actually need — remember loopbacks survive,
  transit /24s don't.

### Type-2 (network) LSAs appearing on transit links
- **Cause:** A core link left as default broadcast type elected a DR → generated a network LSA.
  Fix: `ip ospf network point-to-point` on both ends; the type-2 LSA disappears.
