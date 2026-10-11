# CCIE SP Workbook 12 — 6PE / 6VPE & Dual-Stack (Domain 2)

**Platform:** IOS-XRv 9000
**Topology:** Emerald AS65100 — IPv4 MPLS core with LDP.
**Core PEs:** E-R1 (1.1.1.1), E-R2 (2.2.2.2)
**CEs:** CE1, CE3 (IPv6 addresses)
**6VPE domain:** Garnet

```
        CE1 ---(IPv6)--- E-R1 ==== IPv4/MPLS Core (LDP) ==== E-R2 ---(IPv6)--- CE3
       AS65001         1.1.1.1        AS65100 (Emerald)     2.2.2.2         AS65003
```

- Core loopbacks: E-R1 Lo0 = 1.1.1.1/32, E-R2 Lo0 = 2.2.2.2/32
- Core IGP: OSPF or IS-IS (IPv4), LDP for transport labels
- The core is **IPv4-only** — no IPv6 in the core. IPv6 is tunneled edge-to-edge using MPLS labels (6PE / 6VPE).

---

## Section 1 — 6PE (IPv6 Provider Edge)

6PE carries IPv6 customer traffic across an **IPv4-only MPLS core**. IPv6 prefixes are advertised via MP-BGP (address-family `ipv6 unicast`) between PEs, and each IPv6 prefix is assigned an MPLS label. The BGP next-hop is encoded as an **IPv4-mapped IPv6 address** (`::FFFF:1.1.1.1`) so it resolves through the existing IPv4 LDP LSP. No IPv6 is required in the core.

---

### Task 1.1 — Enable IPv6 and BGP IPv6 unicast on E-R1

**Question:**
On E-R1, enable IPv6 addressing toward CE1 and configure MP-BGP `address-family ipv6 unicast` toward E-R2 (2.2.2.2). The PE–PE BGP session runs over the IPv4 loopbacks (existing iBGP). Ensure the next-hop advertised to E-R2 uses the IPv4-mapped form so it resolves over the IPv4 LSP.

**Solution:**

```
! --- Interface toward CE1 (IPv6) ---
interface GigabitEthernet0/0/0/0
 description To-CE1
 ipv6 address 2001:db8:11::1/64
 no shutdown
!
! --- Core-facing loopback already 1.1.1.1/32, IPv4 iBGP session to E-R2 exists ---
router bgp 65100
 bgp router-id 1.1.1.1
 address-family ipv6 unicast
 !
 neighbor 2.2.2.2
  remote-as 65100
  update-source Loopback0
  address-family ipv6 unicast
   ! 6PE: allocate labels for IPv6 prefixes and rewrite next-hop to ::FFFF:self
   send-label
   next-hop-self
  !
 !
!
```

Key points:
- `send-label` under the IPv6 unicast neighbor AF is what turns a plain MP-BGP IPv6 session into **6PE** — it triggers MPLS label allocation for each IPv6 prefix.
- `next-hop-self` causes E-R1 to advertise the next-hop as its own loopback. On a 6PE session over an IPv4 transport, IOS-XR encodes this as the IPv4-mapped IPv6 next-hop `::FFFF:1.1.1.1`.

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast summary
! Session to 2.2.2.2 should be Established, State/PfxRcd shows received count

RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast neighbors 2.2.2.2 | include Send.*label
! Confirms "My AS is advertising labeled routes" / send-label capability negotiated
```

---

### Task 1.2 — Configure E-R2 and CE3 side, redistribute/advertise CE IPv6 prefixes

**Question:**
Mirror the 6PE configuration on E-R2 (2.2.2.2) facing CE3, and advertise CE3's IPv6 prefix (`2001:db8:33::/64`) into BGP so E-R1 learns it. Use eBGP toward CE3.

**Solution:**

```
! --- E-R2 interface toward CE3 ---
interface GigabitEthernet0/0/0/0
 description To-CE3
 ipv6 address 2001:db8:33::1/64
 no shutdown
!
router bgp 65100
 bgp router-id 2.2.2.2
 address-family ipv6 unicast
 !
 ! iBGP to E-R1 (6PE)
 neighbor 1.1.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv6 unicast
   send-label
   next-hop-self
  !
 !
 ! eBGP to CE3
 neighbor 2001:db8:33::2
  remote-as 65003
  address-family ipv6 unicast
   route-policy PASS in
   route-policy PASS out
  !
 !
!
route-policy PASS
  pass
end-policy
!
```

**Verification:**

```
RP/0/RP0/CPU0:E-R2# show bgp ipv6 unicast
! CE3 prefix 2001:db8:33::/64 present, learned from eBGP neighbor

RP/0/RP0/CPU0:E-R2# show bgp ipv6 unicast neighbors 2001:db8:33::2 received-routes
```

---

### Task 1.3 — Verify the IPv4-mapped next-hop and label allocation

**Question:**
On E-R1, confirm that the IPv6 prefix `2001:db8:33::/64` learned from E-R2 arrives with an **IPv4-mapped IPv6 next-hop** (`::FFFF:2.2.2.2`) and carries a BGP-allocated MPLS label. Confirm the label stack (transport LDP label + BGP 6PE label).

**Solution / Inspection:**

```
RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast 2001:db8:33::/64
```

Expected output highlights:
```
Paths: (1 available, best #1)
  ...
    ::ffff:2.2.2.2 (metric ...) from 2.2.2.2 (2.2.2.2)
      Received Label 24012          <-- BGP 6PE label allocated by E-R2
      Origin IGP, ...
      Local Vpn-handle ...
```

- The next-hop `::ffff:2.2.2.2` is the **IPv4-mapped IPv6 address** of E-R2's loopback. This is the essence of 6PE: an IPv6 control-plane entry whose next-hop resolves via the IPv4 LDP LSP.

**Verification (label stack in CEF):**

```
RP/0/RP0/CPU0:E-R1# show route ipv6 2001:db8:33::/64
! Next-hop resolves recursively via ::ffff:2.2.2.2 -> IPv4 LSP to 2.2.2.2

RP/0/RP0/CPU0:E-R1# show cef ipv6 2001:db8:33::/64
! labels imposed: {LDP_transport_label BGP_6PE_label}
! e.g. labels imposed {24001 24012}
```

The two-label stack = outer LDP transport label (gets swapped hop-by-hop across the IPv4 core) + inner BGP 6PE label (identifies the egress IPv6 prefix on E-R2).

---

### Task 1.4 — Verify end-to-end IPv6 reachability over the IPv4 MPLS backbone

**Question:**
Confirm CE1 (`2001:db8:11::/64`) can reach CE3 (`2001:db8:33::/64`) end-to-end, with the traffic label-switched across the IPv4-only core.

**Solution / Verification:**

```
! From CE1
CE1# ping ipv6 2001:db8:33::2 source 2001:db8:11::2

! On E-R1 — confirm forwarding uses MPLS
RP/0/RP0/CPU0:E-R1# show cef ipv6 2001:db8:33::/64 detail
! Shows label imposition (LDP + 6PE labels), outgoing core interface

! Confirm the LDP LSP to the egress PE exists (IPv4 transport)
RP/0/RP0/CPU0:E-R1# show mpls forwarding prefix 2.2.2.2/32

! On a P (core) router — verify only IPv4/MPLS, no IPv6 knowledge
RP/0/RP0/CPU0:E-R3# show mpls forwarding
! Core swaps labels; it never inspects the IPv6 payload
```

Success criteria: ping succeeds; core routers have no IPv6 RIB entry for the customer prefixes; forwarding is label-switched (two-label stack imposed at E-R1, swapped in core, BGP label popped/looked-up at E-R2).

---

## Section 2 — 6VPE (IPv6 VPN Provider Edge)

6VPE is the VPN version of 6PE. Customer IPv6 routes live in a **VRF** with `address-family ipv6 unicast`, are exported/imported via RTs, and are carried between PEs using the **VPNv6** address-family in MP-BGP. As with 6PE, the transport across the IPv4 core is via MPLS with an IPv4-mapped next-hop. Domain: **Garnet**.

---

### Task 2.1 — Create a VRF with an IPv6 address-family

**Question:**
On E-R1 create VRF `GARNET` with RD `65100:100`, and configure both import/export route-targets for the IPv6 address-family.

**Solution:**

```
vrf GARNET
 address-family ipv6 unicast
  import route-target
   65100:100
  !
  export route-target
   65100:100
  !
 !
!
router bgp 65100
 vrf GARNET
  rd 65100:100
  address-family ipv6 unicast
  !
 !
!
```

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show vrf GARNET detail
! Confirms IPv6 AF, RD 65100:100, import/export RT 65100:100

RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv6 unicast summary
```

---

### Task 2.2 — Enable the VPNv6 address-family in MP-BGP

**Question:**
Enable the `vpnv6 unicast` address-family between E-R1 and E-R2 so VRF IPv6 routes are exchanged as VPNv6 (labeled) routes across the IPv4 core.

**Solution:**

```
router bgp 65100
 address-family vpnv6 unicast
 !
 neighbor 2.2.2.2
  remote-as 65100
  update-source Loopback0
  address-family vpnv6 unicast
  !
 !
!
```

Notes:
- No `send-label` is needed here — VPNv6 (like VPNv4) is inherently labeled; the VPN label is always carried.
- Next-hop toward the remote PE is again the IPv4-mapped IPv6 form (`::FFFF:2.2.2.2`), resolved via the IPv4 LDP LSP.

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show bgp vpnv6 unicast summary
! Session 2.2.2.2 Established under VPNv6

RP/0/RP0/CPU0:E-R1# show bgp vpnv6 unicast rd 65100:100
```

---

### Task 2.3 — Configure a dual-stack VRF (IPv4 + IPv6 in the same VRF)

**Question:**
Extend VRF `GARNET` so it carries **both** IPv4 and IPv6 customer routes. Configure IPv4 and IPv6 AFs under the VRF and under BGP, and enable both VPNv4 and VPNv6 AFs to E-R2.

**Solution:**

```
vrf GARNET
 address-family ipv4 unicast
  import route-target
   65100:100
  !
  export route-target
   65100:100
  !
 !
 address-family ipv6 unicast
  import route-target
   65100:100
  !
  export route-target
   65100:100
  !
 !
!
router bgp 65100
 address-family vpnv4 unicast
 !
 address-family vpnv6 unicast
 !
 neighbor 2.2.2.2
  address-family vpnv4 unicast
  !
  address-family vpnv6 unicast
  !
 !
 vrf GARNET
  rd 65100:100
  address-family ipv4 unicast
  !
  address-family ipv6 unicast
  !
 !
!
! --- Dual-stack CE-facing interface in the VRF ---
interface GigabitEthernet0/0/0/1
 vrf GARNET
 ipv4 address 10.100.11.1 255.255.255.0
 ipv6 address 2001:db8:100:11::1/64
 no shutdown
!
```

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show vrf GARNET detail
! Both ipv4 and ipv6 AFs listed

RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv4 unicast
RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv6 unicast
```

---

### Task 2.4 — CE–PE IPv6 routing (eBGP or OSPFv3) and verify VPNv6 exchange

**Question:**
On E-R1 configure CE–PE IPv6 routing for VRF `GARNET`. Provide both the eBGP option and the OSPFv3 option. Then verify VRF IPv6 routes are exchanged as VPNv6 between E-R1 and E-R2 and installed on the remote PE.

**Solution (eBGP CE–PE):**

```
router bgp 65100
 vrf GARNET
  address-family ipv6 unicast
  !
  neighbor 2001:db8:100:11::2
   remote-as 65011
   address-family ipv6 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
```

**Solution (OSPFv3 CE–PE, alternative):**

```
router ospfv3 1
 vrf GARNET
  address-family ipv6 unicast
  area 0
   interface GigabitEthernet0/0/0/1
   !
  !
 !
!
! Redistribute OSPFv3 <-> BGP within the VRF
router bgp 65100
 vrf GARNET
  address-family ipv6 unicast
   redistribute ospfv3 1
  !
 !
!
router ospfv3 1
 vrf GARNET
  address-family ipv6 unicast
   redistribute bgp 65100
  !
 !
!
```

**Verification:**

```
! On E-R2 — confirm E-R1's VRF IPv6 prefixes arrive via VPNv6 and land in the VRF RIB
RP/0/RP0/CPU0:E-R2# show bgp vpnv6 unicast rd 65100:100
RP/0/RP0/CPU0:E-R2# show bgp vrf GARNET ipv6 unicast
RP/0/RP0/CPU0:E-R2# show route vrf GARNET ipv6

! CE-PE session state (eBGP option)
RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv6 unicast neighbors 2001:db8:100:11::2

! CE-PE adjacency (OSPFv3 option)
RP/0/RP0/CPU0:E-R1# show ospfv3 vrf GARNET neighbor
```

---

## Section 3 — Dual-Stack VRF

Consolidates IPv4 and IPv6 service into a single VRF with RD/RTs for both AFs, introduces the SRv6 `End.DT46` SID concept (Gold), and compares separate vs unified VPN designs.

---

### Task 3.1 — VRF with both IPv4 and IPv6 AFs, RD and RT for both

**Question:**
Define VRF `GOLD` as a fully dual-stacked VRF: single RD, RTs applied to both the IPv4 and IPv6 address-families. Show the complete VRF and BGP VRF stanza.

**Solution:**

```
vrf GOLD
 address-family ipv4 unicast
  import route-target
   65100:200
  !
  export route-target
   65100:200
  !
 !
 address-family ipv6 unicast
  import route-target
   65100:200
  !
  export route-target
   65100:200
  !
 !
!
router bgp 65100
 vrf GOLD
  rd 65100:200
  address-family ipv4 unicast
   redistribute connected
  !
  address-family ipv6 unicast
   redistribute connected
  !
 !
!
```

Design note: A single RD identifies the VRF instance; the same RT (`65100:200`) is reused across both AFs so IPv4 and IPv6 prefixes of the same customer share membership. The RD makes each `(RD:prefix)` VPNv4/VPNv6 NLRI globally unique.

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show vrf GOLD detail
RP/0/RP0/CPU0:E-R1# show bgp vrf GOLD ipv4 unicast
RP/0/RP0/CPU0:E-R1# show bgp vrf GOLD ipv6 unicast
```

---

### Task 3.2 — End.DT46 SID concept in SRv6 (Gold)

**Question:**
Explain the SRv6 `End.DT46` SID and show how it is provisioned for the dual-stack VRF `GOLD` under a segment-routing SRv6 locator. Contrast `End.DT4`, `End.DT6`, and `End.DT46`.

**Solution (concept):**

- In SRv6 L3VPN, the egress PE advertises a **service SID** (an IPv6 address from its locator block) instead of an MPLS VPN label. The SID's function determines the decapsulation/lookup behavior:
  - **End.DT4** — Endpoint with Decapsulation and IPv4 Table lookup (per-VRF, IPv4 only).
  - **End.DT6** — Endpoint with Decapsulation and IPv6 Table lookup (per-VRF, IPv6 only).
  - **End.DT46** — Endpoint with Decapsulation and **dual IPv4/IPv6** Table lookup. A single SID serves a dual-stack VRF: after decapsulation the inner packet's ethertype/IP version selects the IPv4 or IPv6 FIB of the VRF.
- `End.DT46` is the SRv6 analog of a single dual-stack VRF context — one SID covers both AFs, reducing SID consumption versus allocating a separate `End.DT4` + `End.DT6`.

**Solution (config sketch, SRv6 dual-stack VRF):**

```
segment-routing
 srv6
  locators
   locator GOLD_LOC
    prefix 2001:db8:ffff:100::/64
   !
  !
 !
!
router bgp 65100
 address-family vpnv4 unicast
  segment-routing srv6
   locator GOLD_LOC
  !
 !
 address-family vpnv6 unicast
  segment-routing srv6
   locator GOLD_LOC
  !
 !
 vrf GOLD
  rd 65100:200
  address-family ipv4 unicast
   segment-routing srv6
    locator GOLD_LOC
   !
  !
  address-family ipv6 unicast
   segment-routing srv6
    locator GOLD_LOC
   !
  !
 !
!
```

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show segment-routing srv6 sid
! Look for a SID with function End.DT46 associated with VRF GOLD

RP/0/RP0/CPU0:E-R1# show segment-routing srv6 locator GOLD_LOC detail
RP/0/RP0/CPU0:E-R1# show bgp vpnv6 unicast rd 65100:200
! VPNv6 NLRI carries the SRv6 SID (PSID TLV) instead of an MPLS label
```

---

### Task 3.3 — Compare VPNv4 + VPNv6 separate vs unified

**Question:**
Compare running **separate** VPNv4 and VPNv6 service (two VRFs / two RDs) against a **unified dual-stack VRF** (one VRF, one RD, RTs on both AFs). When would you choose each?

**Solution (comparison):**

| Aspect | Separate VPNv4 + VPNv6 | Unified dual-stack VRF |
|---|---|---|
| VRF count | Two VRFs (one per AF) | One VRF, two AFs |
| RD | Distinct RD per VRF | Single RD shared by both AFs |
| RT policy | Independent per service | Shared RT — coupled membership |
| Address planning | Independent IPv4/IPv6 scopes | Aligned per-customer scopes |
| Operational model | Isolated troubleshooting/lifecycle per AF | Single context, fewer objects |
| Label / SID usage (MPLS) | Separate VPN labels per VRF | Per-AF labels within one VRF |
| SRv6 | End.DT4 + End.DT6 (two SIDs) | End.DT46 (single SID) |
| Best when | AFs have different customers, policies, or migration timelines; staged IPv6 rollout | Same customer needs coordinated dual-stack; simpler ops |

Guidance:
- **Unified** is preferred for genuine dual-stack customers — one VRF, one RD, shared RT keeps IPv4/IPv6 membership consistent and (in SRv6) collapses to a single `End.DT46` SID.
- **Separate** is preferred when IPv4 and IPv6 are logically different services, have different RT import/export policies, or when IPv6 is being rolled out on an independent timeline and you want fault/lifecycle isolation.

**Verification (conceptual):**

```
! Unified: single VRF shows both AFs
RP/0/RP0/CPU0:E-R1# show vrf GOLD detail

! Separate: two VRFs each with one AF
RP/0/RP0/CPU0:E-R1# show vrf all detail
```

---

## Section 4 — Troubleshooting

---

### Task 4.1 — IPv6 route not appearing in the VRF (missing IPv6 AF under VRF)

**Question:**
A dual-stack customer's IPv4 routes are present in VRF `GARNET` but its **IPv6 routes never appear**, on both the local and remote PE. VPNv6 sessions are up. Identify and fix the root cause.

**Diagnosis:**

```
RP/0/RP0/CPU0:E-R1# show vrf GARNET detail
! Symptom: only "Address family IPv4 Unicast" listed — no IPv6 AF / no IPv6 RTs

RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv6 unicast
! Symptom: "% No such address family" or empty — VRF has no ipv6 AF to import into
```

Root cause: the VRF is missing `address-family ipv6 unicast` (and its import/export route-targets). Without the IPv6 AF and RTs under the VRF, VPNv6 routes have no place to import — even though the VPNv6 BGP session is Established, imported prefixes are dropped for that VRF.

**Solution:**

```
vrf GARNET
 address-family ipv6 unicast
  import route-target
   65100:100
  !
  export route-target
   65100:100
  !
 !
!
router bgp 65100
 vrf GARNET
  address-family ipv6 unicast
  !
 !
!
```

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show vrf GARNET detail
! Now shows IPv6 Unicast AF with import/export RT 65100:100

RP/0/RP0/CPU0:E-R1# show bgp vrf GARNET ipv6 unicast
RP/0/RP0/CPU0:E-R1# show route vrf GARNET ipv6
! Remote IPv6 prefixes now imported and installed
```

---

### Task 4.2 — 6PE label not allocated (missing IPv6 label allocation in BGP)

**Question:**
IPv6 prefixes are exchanged between E-R1 and E-R2 over the `ipv6 unicast` BGP session and appear in `show bgp ipv6 unicast`, but end-to-end forwarding fails and CEF shows **no label imposed** for the remote IPv6 prefix. The IPv4 LDP LSP between PEs is healthy. Identify and fix.

**Diagnosis:**

```
RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast 2001:db8:33::/64
! Symptom: path present but "Received Label" is absent / "no label"

RP/0/RP0/CPU0:E-R1# show cef ipv6 2001:db8:33::/64 detail
! Symptom: no labels imposed -> attempts native IPv6 forwarding over an IPv4 core -> fails

RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast neighbors 2.2.2.2 | include label
! Symptom: send-label capability not negotiated
```

Root cause: the neighbor's IPv6 unicast AF is missing `send-label`. Without it the session is plain MP-BGP IPv6 (no MPLS label allocation), so there is no 6PE inner label to carry IPv6 across the IPv4-only core.

**Solution:**

```
router bgp 65100
 neighbor 2.2.2.2
  address-family ipv6 unicast
   send-label
   next-hop-self
  !
 !
!
! Apply on BOTH PEs so labels are allocated in each direction.
```

**Verification:**

```
RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast neighbors 2.2.2.2 | include label
! send-label capability now negotiated

RP/0/RP0/CPU0:E-R1# show bgp ipv6 unicast 2001:db8:33::/64
! Now shows "Received Label <n>" and next-hop ::ffff:2.2.2.2

RP/0/RP0/CPU0:E-R1# show cef ipv6 2001:db8:33::/64 detail
! Two-label stack imposed {LDP_transport_label 6PE_label}; forwarding restored
```

---

## Quick Reference — 6PE vs 6VPE

| | 6PE | 6VPE |
|---|---|---|
| BGP AF | `ipv6 unicast` + `send-label` | `vpnv6 unicast` (inherently labeled) |
| Isolation | Global IPv6 table | Per-VRF IPv6 table |
| Next-hop over IPv4 core | `::FFFF:<PE-lo>` | `::FFFF:<PE-lo>` |
| Labels | 6PE label + LDP transport | VPN label + LDP transport |
| RD/RT | None | RD + IPv6 import/export RT |
| Use case | Internet-style IPv6 transit over IPv4 MPLS | IPv6 L3VPN service |

**Golden rules:**
- 6PE = `send-label` under `ipv6 unicast`. Forget it → no IPv6 label → no forwarding (Task 4.2).
- 6VPE = VRF must have `address-family ipv6 unicast` + RTs, and BGP needs `vpnv6 unicast`. Missing VRF AF → no import (Task 4.1).
- Core stays IPv4-only; the IPv4-mapped next-hop `::FFFF:x.x.x.x` is what glues the IPv6 control plane to the IPv4 LSP.
- SRv6 dual-stack VRF → one `End.DT46` SID replaces `End.DT4` + `End.DT6`.
