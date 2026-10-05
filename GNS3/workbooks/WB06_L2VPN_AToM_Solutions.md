# WB06 L2VPN AToM / VPWS — Solutions

**Reference:** `00_topology_reference.md` for VLAN-100 L2 attachment circuits.
**Prerequisite:** WB00–WB03 (IP, IGP, LDP). AToM is directed-LDP over the MPLS core — the PWs ride targeted LDP sessions between PE loopbacks, so `mpls ldp` and loopback reachability must already be up.
**Platform:** Cisco 7200, IOS 15.2(4)M11 — IOS classic syntax. Uses the `xconnect` form (not the newer `l2vpn xconnect context`), which is what 7200/IOS classic supports.

---

## Attachment Circuits (VLAN 100)

| PE | AC interface | CE | AS | Role |
|----|-------------|----|----|------|
| R14 | f4/0 | R28 | 65200 | PW endpoint A |
| R16 | f3/0 | R27 | 65200 | PW endpoint B |
| R1 | g2/0 | R29 | 65100 | cross-AS endpoint (needs inter-AS stitching) |

> R14 and R16 are both in AS 65200, so a plain intra-AS AToM PW works directly between them. R1 is in AS 65100 — a PW to R14/R16 crosses an AS boundary and is **not** a simple xconnect (see Section 3).

---

## Section 1: Intra-AS PW — R14 ↔ R16 (VC-ID 100)

Both PEs are in AS 65200. The PW uses each PE's Loopback0 as the targeted-LDP endpoint. VC-ID must match on both ends (100).

### Option A — Port-based (EoMPLS, whole interface)

If the entire f4/0 / f3/0 carries only VLAN 100 and you treat the port as one AC:

```
! R14
interface FastEthernet4/0
 no ip address
 no shutdown
 xconnect 150.2.16.16 100 encapsulation mpls
!
```

```
! R16
interface FastEthernet3/0
 no ip address
 no shutdown
 xconnect 150.2.14.14 100 encapsulation mpls
!
```

### Option B — VLAN-based (subinterface, VLAN 100 tag preserved)

Preferred when the AC is specifically VLAN 100 and you want 802.1Q handling:

```
! R14
interface FastEthernet4/0
 no ip address
 no shutdown
!
interface FastEthernet4/0.100
 encapsulation dot1Q 100
 xconnect 150.2.16.16 100 encapsulation mpls
!
```

```
! R16
interface FastEthernet3/0
 no ip address
 no shutdown
!
interface FastEthernet3/0.100
 encapsulation dot1Q 100
 xconnect 150.2.14.14 100 encapsulation mpls
!
```

> The remote peer address is the **far PE's Loopback0**. VC-ID `100` is the demultiplexer that binds the two ends — it must be identical. `encapsulation mpls` selects AToM (vs L2TPv3). The AC type (Ethernet/VLAN) must match on both ends or the PW stays down with a type-mismatch.

### Optional: explicit pseudowire-class (control-word, interworking)

Using a `pseudowire-class` makes the PW attributes explicit and reusable:

```
! On both R14 and R16
pseudowire-class ATOM-PW
 encapsulation mpls
 control-word
!
interface FastEthernet4/0.100
 encapsulation dot1Q 100
 xconnect 150.2.16.16 100 pw-class ATOM-PW
!
```

> `control-word` improves fragmentation/ordering handling and small-packet disambiguation. Must match on both ends. R28 (behind R14) and R27 (behind R16) now share one L2 VLAN-100 segment across the MPLS core.

---

## Section 2: CE devices (R27, R28) — reference

R27 and R28 are plain L2/L3 endpoints on VLAN 100. If used as routers sharing a subnet across the PW:

```
! R28 (behind R14)
interface FastEthernet0/0
 no shutdown
!
interface FastEthernet0/0.100
 encapsulation dot1Q 100
 ip address 172.16.100.28 255.255.255.0
!
! R27 (behind R16)
interface FastEthernet3/0
 no shutdown
!
interface FastEthernet3/0.100
 encapsulation dot1Q 100
 ip address 172.16.100.27 255.255.255.0
!
```

> 172.16.100.0/24 is a lab-chosen customer subnet (not in the topology reference, which lists these links only as "VLAN 100 L2"). R27 and R28 should ping each other once the PW is up — they appear directly L2-adjacent despite being separated by the MPLS core.

---

## Section 3: R1 g2/0 (R29) — Cross-AS PW (NOT a simple xconnect)

R1 is in AS 65100; R14/R16 are in AS 65200. A pseudowire from R1 to R14 or R16 cannot use a single targeted-LDP session because the loopbacks are not in the same IGP/LDP domain and directed LDP is not natively exchanged across the AS boundary.

**This requires inter-AS PW stitching (multi-segment pseudowire, MS-PW)** — do WB09 (Inter-AS) first, then stitch two PW segments at the ASBR.

Config shown for completeness — **expect it to stay DOWN until inter-AS label/loopback exchange exists:**

```
! R1 — segment toward R14 (will not come up without inter-AS reachability to 150.2.14.14)
interface GigabitEthernet2/0
 no ip address
 no shutdown
!
interface GigabitEthernet2/0.100
 encapsulation dot1Q 100
 xconnect 150.2.14.14 100 encapsulation mpls
!
```

### MS-PW stitching sketch (at the ASBR that bridges 65100↔65200, e.g. R6↔R12)

A multi-segment PW is stitched at a border router with `l2 vfi ... point-to-point` or the `xconnect`-stitch form. On IOS classic 7200 the common approach:

```
! Conceptual — stitch segment-1 (toward AS65100 PE R1) to segment-2 (toward AS65200 PE R14)
! Performed on the stitching PE/ASBR once inter-AS loopback reachability is established.
l2 vfi R29-R28-STITCH point-to-point
 neighbor 150.1.1.1 100 encapsulation mpls      ! segment to R1  (AS65100)
 neighbor 150.2.14.14 100 encapsulation mpls     ! segment to R14 (AS65200)
!
```

> **Blocker:** the stitching node must have LDP/loopback reachability to **both** 150.1.1.1 and 150.2.14.14. That cross-AS reachability is exactly what WB09 Inter-AS (Option B/C or BGP-LU) provides. Until then, this PW is non-functional. Document it, configure it, and revisit after WB09/WB14 (BGP-LU unified MPLS).

---

## Section 4: PW Redundancy (backup pseudowire)

Protect the primary PW with a `backup peer`. Example: R14's AC to R28 primary to R16, backup to an alternate PE (e.g. R15, if R15 also had an AC to the same customer). Template:

```
! R14 — primary PW to R16, backup PW to R15
pseudowire-class ATOM-PW
 encapsulation mpls
!
interface FastEthernet4/0.100
 encapsulation dot1Q 100
 xconnect 150.2.16.16 100 pw-class ATOM-PW
  backup peer 150.2.15.15 100
  backup delay 5 never
!
```

> `backup peer <loopback> <vc-id>` defines the standby PW. `backup delay 5 never` = wait 5s before activating backup after primary failure, and never automatically revert (manual/`never` switchback) — avoids flapping. Only one PW forwards at a time. The backup PE must have a matching AC/xconnect toward the same customer segment for traffic to actually flow.

Alternative redundancy at the attachment side: **MC-LAG / ICCP** or **pseudowire redundancy with status signaling** (`xconnect` + `mpls l2transport` PW status). Status TLV signaling is on by default in modern IOS and lets each end learn the other's AC/PW state.

---

## Section 5: Verification Commands

```
! PW / VC state
show mpls l2transport vc
show mpls l2transport vc 100 detail
show xconnect all
show xconnect peer 150.2.16.16 vcid 100

! Targeted LDP session that carries the PW
show mpls ldp neighbor 150.2.16.16
show mpls ldp discovery

! Labels
show mpls l2transport binding
show mpls forwarding-table

! Redundancy
show mpls l2transport vc detail        ! look for "Status: ACTIVE / STANDBY"
show xconnect all detail

! Data plane (from CEs)
! R27 ping R28 across the PW (if L3 addresses assigned per Section 2)
```

### Reading `show mpls l2transport vc`

```
Local intf     Local circuit        Dest address     VC ID   Status
-------------  -------------------  ---------------  ------  ----------
Fa4/0.100      Eth VLAN 100         150.2.16.16      100     UP
```

> A healthy intra-AS PW shows **Status: UP** on both R14 and R16. Common down-causes: VC-ID mismatch, AC type mismatch (port vs VLAN), control-word mismatch, or no targeted-LDP session (loopback not reachable / LDP not enabled). The R1↔R14 cross-AS PW (Section 3) will show **DOWN / no remote binding** until WB09 is complete — that is expected, not a misconfiguration.
