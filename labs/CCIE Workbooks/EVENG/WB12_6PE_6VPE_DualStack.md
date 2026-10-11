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


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — Configure E-R2 and CE3 side, redistribute/advertise CE IPv6 prefixes

**Question:**
Mirror the 6PE configuration on E-R2 (2.2.2.2) facing CE3, and advertise CE3's IPv6 prefix (`2001:db8:33::/64`) into BGP so E-R1 learns it. Use eBGP toward CE3.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Verify the IPv4-mapped next-hop and label allocation

**Question:**
On E-R1, confirm that the IPv6 prefix `2001:db8:33::/64` learned from E-R2 arrives with an **IPv4-mapped IPv6 next-hop** (`::FFFF:2.2.2.2`) and carries a BGP-allocated MPLS label. Confirm the label stack (transport LDP label + BGP 6PE label).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Verify end-to-end IPv6 reachability over the IPv4 MPLS backbone

**Question:**
Confirm CE1 (`2001:db8:11::/64`) can reach CE3 (`2001:db8:33::/64`) end-to-end, with the traffic label-switched across the IPv4-only core.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — 6VPE (IPv6 VPN Provider Edge)

6VPE is the VPN version of 6PE. Customer IPv6 routes live in a **VRF** with `address-family ipv6 unicast`, are exported/imported via RTs, and are carried between PEs using the **VPNv6** address-family in MP-BGP. As with 6PE, the transport across the IPv4 core is via MPLS with an IPv4-mapped next-hop. Domain: **Garnet**.

---

### Task 2.1 — Create a VRF with an IPv6 address-family

**Question:**
On E-R1 create VRF `GARNET` with RD `65100:100`, and configure both import/export route-targets for the IPv6 address-family.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — Enable the VPNv6 address-family in MP-BGP

**Question:**
Enable the `vpnv6 unicast` address-family between E-R1 and E-R2 so VRF IPv6 routes are exchanged as VPNv6 (labeled) routes across the IPv4 core.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — Configure a dual-stack VRF (IPv4 + IPv6 in the same VRF)

**Question:**
Extend VRF `GARNET` so it carries **both** IPv4 and IPv6 customer routes. Configure IPv4 and IPv6 AFs under the VRF and under BGP, and enable both VPNv4 and VPNv6 AFs to E-R2.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4 — CE–PE IPv6 routing (eBGP or OSPFv3) and verify VPNv6 exchange

**Question:**
On E-R1 configure CE–PE IPv6 routing for VRF `GARNET`. Provide both the eBGP option and the OSPFv3 option. Then verify VRF IPv6 routes are exchanged as VPNv6 between E-R1 and E-R2 and installed on the remote PE.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — Dual-Stack VRF

Consolidates IPv4 and IPv6 service into a single VRF with RD/RTs for both AFs, introduces the SRv6 `End.DT46` SID concept (Gold), and compares separate vs unified VPN designs.

---

### Task 3.1 — VRF with both IPv4 and IPv6 AFs, RD and RT for both

**Question:**
Define VRF `GOLD` as a fully dual-stacked VRF: single RD, RTs applied to both the IPv4 and IPv6 address-families. Show the complete VRF and BGP VRF stanza.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — End.DT46 SID concept in SRv6 (Gold)

**Question:**
Explain the SRv6 `End.DT46` SID and show how it is provisioned for the dual-stack VRF `GOLD` under a segment-routing SRv6 locator. Contrast `End.DT4`, `End.DT6`, and `End.DT46`.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — Compare VPNv4 + VPNv6 separate vs unified

**Question:**
Compare running **separate** VPNv4 and VPNv6 service (two VRFs / two RDs) against a **unified dual-stack VRF** (one VRF, one RD, RTs on both AFs). When would you choose each?


> *Try this yourself first. Solution available in `solutions/` folder.*

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


> *Try this yourself first. Solution available in `solutions/` folder.*

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


> *Try this yourself first. Solution available in `solutions/` folder.*

