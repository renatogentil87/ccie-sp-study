# WB03 MPLS LDP — Solutions

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11 — IOS Classic
**Reference:** `00_topology_reference.md` for IP addressing and wiring.
**Scope:** LDP on the **core (IGP-facing)** interfaces of all three ASes.

> `mpls ip` on each core interface, `mpls ldp router-id Loopback0 force` for a stable LDP ID.
> LDP runs **within** each AS only. Inter-AS links (R5↔R11, R6↔R12, R6↔R20, R12↔R21) and all
> PE-CE / L2 links are left label-free (reserved for Inter-AS / L2VPN workbooks).
> Global default `mpls label protocol ldp` is assumed; shown explicitly on each router for clarity.

---

## Section 1: LDP on AS 65100 (IS-IS core, R1–R8)

### R1 (PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface FastEthernet4/0
 mpls ip
!
```

> R1 core: f0/0 (→R3), f4/0 (→R2). PE-CE (g1/0, f3/0, f4/1) and L2 (g2/0) stay label-free.

### R2 (PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

> R2 core: f0/0 (→R4), f3/0 (→R1). PE-CE g1/0 (→R10) label-free.

### R3 (P)

```
mpls label protocol ldp
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

> R3 core: f0/0 (→R1), g1/0 (→R4), g2/0 (→R5), f3/0 (→R7). All core — transit P router.

### R4 (P)

```
mpls label protocol ldp
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
```

> R4 core: f0/0 (→R2), g1/0 (→R3), g2/0 (→R6). All core.

### R5 (ASBR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

> R5 core: f0/0 (→R8), g2/0 (→R3), f3/0 (→R6). Inter-AS g1/0 (→R11) label-free.

### R6 (ASBR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet2/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

> R6 core: g2/0 (→R4), f3/0 (→R5). Inter-AS g1/0 (→R12), f0/0 (→R20) label-free.

### R7 (RR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet3/0
 mpls ip
!
```

> R7 single core: f3/0 (→R3).

### R8 (RR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
```

> R8 single core: f0/0 (→R5).

### Verification (AS 65100)

```
show mpls ldp neighbor                        ! Oper sessions on every core link
show mpls ldp bindings                         ! local+remote labels for all 150.1.x.x/32
show mpls forwarding-table                      ! LFIB populated, out-labels toward each loopback
traceroute 150.1.8.8 source 150.1.1.1          ! R1->R8, labels shown each hop
```

---

## Section 2: LDP on AS 65200 (OSPF core, R11–R16, R19)

### R11 (P)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

> R11 core: f0/0 (→R12), g2/0 (→R13), f3/0 (→R15). Inter-AS g1/0 (→R5) label-free.

### R12 (ASBR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
```

> R12 core: f0/0 (→R11), g2/0 (→R14). Inter-AS g1/0 (→R6), f3/0 (→R21) label-free.

### R13 (P)

```
mpls label protocol ldp
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

> R13 core: f0/0 (→R16), g1/0 (→R14), g2/0 (→R11), f3/0 (→R19). All core.

### R14 (PE)

```
mpls label protocol ldp
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
```

> R14 core: f0/0 (→R19), g1/0 (→R13), g2/0 (→R12). PE-CE f3/0 (→R25), L2 f4/0 (→R28) label-free.

### R15 (PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet3/0
 mpls ip
!
```

> R15 single core: f3/0 (→R11). PE-CE f0/0 (→R17) label-free.

### R16 (PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
```

> R16 single core: f0/0 (→R13). PE-CE g1/0, g2/0 and L2 f3/0 label-free.

### R19 (RR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface FastEthernet3/0
 mpls ip
!
```

> R19 core: f0/0 (→R14), f3/0 (→R13).

### Verification (AS 65200)

```
show mpls ldp neighbor                        ! Oper sessions on every OSPF core link
show mpls ldp bindings                         ! labels for all 150.2.x.x/32
show mpls forwarding-table                      ! LFIB populated
traceroute 150.2.16.16 source 150.2.11.11      ! R11->R16, labels each hop
```

---

## Section 3: LDP on AS 65300 (IS-IS core, R20–R23, R30)

### R20 (ASBR/PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet1/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
```

> R20 core: g1/0 (→R21), g2/0 (→R22). Inter-AS f0/0 (→R6), PE-CE f3/0 (→R24) label-free.

### R21 (ASBR/PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface FastEthernet0/0
 mpls ip
!
interface GigabitEthernet1/0
 mpls ip
!
```

> R21 core: f0/0 (→R23), g1/0 (→R20). Inter-AS f3/0 (→R12), PE-CE g2/0 (→R25) label-free.

### R22 (PE)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet1/0
 mpls ip
!
interface GigabitEthernet2/0
 mpls ip
!
```

> R22 core: g1/0 (→R23), g2/0 (→R20). PE-CE f0/0 (→R24) label-free.

### R23 (PE)

```
mpls label protocol ldp
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
```

> R23 core: f0/0 (→R21), g1/0 (→R22), g2/0 (→R30). PE-CE f3/0 (→R25) label-free.

### R30 (RR)

```
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet2/0
 mpls ip
!
```

> R30 single core: g2/0 (→R23).

### Verification (AS 65300)

```
show mpls ldp neighbor                        ! Oper sessions on every core link
show mpls ldp bindings                         ! labels for all 150.3.x.x/32
show mpls forwarding-table                      ! LFIB populated
traceroute 150.3.30.30 source 150.3.20.20      ! R20->R30, labels each hop
```

---

## Section 4: LDP Advanced (all ASes)

### Label allocation filtering — host routes (loopbacks) only

```
ip prefix-list LOOPBACKS-65100 seq 5 permit 150.1.0.0/16 ge 32
!
mpls ldp label
 allocate global prefix-list LOOPBACKS-65100
!
```

> **Goal:** bind labels only to /32 loopbacks, not transit /24s — shrinks LIB/LFIB (standard SP
> practice). On 15.2M the global form is `mpls ldp label` → `allocate global prefix-list <name>`
> (or, where supported, `mpls ldp label allocate global host-routes`). Use the per-AS prefix-list
> (`150.1.0.0/16 ge 32`, `150.2.0.0/16 ge 32`, `150.3.0.0/16 ge 32`). If the exact keyword isn't
> present on this image, fall back to `no mpls ldp advertise-labels` + an advertise ACL permitting
> only loopback /32s:
>
> ```
> access-list 1 permit 150.1.0.0 0.0.255.255
> no mpls ldp advertise-labels
> mpls ldp advertise-labels for 1
> ```
>
> Verify: `show mpls ldp bindings` shows only /32s; transit /24s no longer have labels.

### LDP Session Protection (all core routers)

```
mpls ldp session protection
```

> Keeps the LDP session up via a **targeted hello** through the loopback if the direct link flaps,
> so labels don't have to be relearned on recovery. Scope with
> `mpls ldp session protection for <acl> [duration <sec>]` if you only want it toward specific peers.
> Verify: `show mpls ldp neighbor detail` → "LDP Session Protection: enabled".

### LDP Password / MD5 Authentication (per AS, key "CCIE")

```
! AS 65100 — authenticate all 150.1.x.x peers:
access-list 10 permit 150.1.0.0 0.0.255.255
mpls ldp neighbor password CCIE
! (image-dependent form)
mpls ldp password required for 10
mpls ldp password option 1 for 10 CCIE
```

> Protects the LDP TCP session with MD5. Both peers must share the key. On 15.2M use the
> `mpls ldp password option <n> for <acl> <password>` form (ACL selects peer loopbacks).
> Test: matching key → session stays Oper; mismatch → session drops (`show mpls ldp neighbor`
> shows it never reaches Oper). Use key "CCIE" per AS, scoping the ACL to that AS's 150.x.0.0/16.

### LDP–IGP Synchronization

```
! AS 65100 / AS 65300 (IS-IS):
router isis CORE
 mpls ldp sync
!
! AS 65200 (OSPF):
router ospf 1
 mpls ldp sync
!
```

> Holds a link at **max IGP metric** until LDP is Oper on it, preventing a labeled-traffic black
> hole over a link whose LDP session hasn't come up yet. Pair with `mpls ldp igp sync holddown
> <ms>` if you want a bounded wait. Verify: `show mpls ldp igp sync` → "Sync: Enabled / Peer
> reachable", and during convergence the IGP advertises max-metric on the forming link, then reverts.

### Verification (advanced)

```
show mpls ldp neighbor detail | include Protection   ! session protection enabled
show mpls ldp neighbor detail | include Password|MD5  ! authentication in effect
show mpls ldp igp sync                                 ! sync enabled, peer reachable
show mpls ldp bindings | include 150\.                 ! only loopback /32s have labels
```

---

## Section 5: Verification — end-to-end label switching

```
! Labeled path with per-hop labels (run in each AS):
traceroute mpls ipv4 150.1.8.8/32              ! AS65100 R1->R8 LSP, label at each hop
traceroute mpls ipv4 150.2.16.16/32            ! AS65200 R11->R16
traceroute mpls ipv4 150.3.30.30/32            ! AS65300 R20->R30
!
! LSP ping (connectivity + MTU of the LSP):
ping mpls ipv4 150.1.8.8/32
!
! PHP check — penultimate hop pops (implicit-null / "Pop Label" outgoing):
show mpls forwarding-table 150.1.8.8 32         ! on R5 (penultimate to R8) -> Pop Label
!
! LFIB in->out swap check on a transit P router:
show mpls forwarding-table                       ! R3/R13/R23: in-label swapped to out-label
!
! Confirm CEF chooses labeled next-hop, not plain IP:
show ip cef 150.1.8.8 detail                     ! shows "fast tag rewrite" / outgoing label
```

> **Expected:**
> - `show mpls ldp neighbor` → every core link has an **Oper** session (no PE-CE/inter-AS sessions).
> - `show mpls ldp bindings` → a label for every loopback /32 in the local AS (only /32s if the
>   host-route filter from Sec 4.1 is applied).
> - `traceroute mpls` → a label on each transit hop; the penultimate hop pops (PHP), egress PE
>   receives an unlabeled packet.
> - LDP does **not** cross AS boundaries — no sessions on inter-AS links.
