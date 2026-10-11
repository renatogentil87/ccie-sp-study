# CCIE SP Workbook 07 — L3VPN

**Domain:** 2 — Architectures & Services (25%)
**Platform:** EVE-NG — IOS-XRv 9000 7.11.1 (all PE/P/RR/ASBR nodes)
🔴 **Topology:** Emerald + Gold + Garnet — see `../configuration/EVENG/00_topology_reference.md`
**Format:** Question → Solution → Verification. All configs in IOS-XR syntax.

---

## Topology Recap (nodes used in this workbook)

| SP | AS | IGP / Transport | PEs | RR | ASBR |
|----|----|-----------------|-----|----|------|
| **Emerald** | 65100 | IS-IS L2 + LDP | E-R1 (1.1.1.1), E-R2 (2.2.2.2) | **E-R5 (6.6.6.6)** | E-R6 (5.5.5.5) |
| **Gold** (transit) | 65300 | IS-IS L2 + SRv6 | G-R1 (21.21.21.21), G-R2 (22.22.22.22) | **G-R4 (24.24.24.24)** | G-R4 / G-R5 (25.25.25.25) |
| **Garnet** | 65200 | IS-IS L2 + SR-MPLS | Gar-R1 (11.11.11.11), Gar-R2 (12.12.12.12) | **Gar-R6 (16.16.16.16)** | Gar-R7 (17.17.17.17) |

**Customers**

| Customer | ASN | CEs | SPs spanned | PE-CE protocol |
|----------|-----|-----|-------------|----------------|
| **A** | 65012 | CE1 (E-R1), CE2 (E-R1+E-R2 dual-homed) — Emerald; CE8 (G-R1+G-R2 dual-homed) — Gold | Emerald + Gold | eBGP (as-override + SoO) |
| **B** | 65013 | CE9 (G-R1) — Gold; CE4 (Gar-R1) — Garnet | Gold + Garnet | eBGP |
| **C** | — (EVPN) | CE5 (Gar-R1+Gar-R2) — Garnet; CE7 (G-R2) — Gold | Garnet + Gold | EVPN (WB10, not here) |
| — | — | CE3 (E-R2) — Emerald; CE6 (Gar-R2) — Garnet | single SP | OSPF area 0 |

**PE-CE links (relevant)**

| Link | Subnet | PE addr | CE addr |
|------|--------|---------|---------|
| E-R1–CE1 | 192.168.1.0/24 | .1 | .2 |
| E-R1–CE2 | 192.168.2.0/24 | .1 | .2 |
| E-R2–CE2 | 192.168.3.0/24 | .1 | .2 |
| E-R2–CE3 | 192.168.4.0/24 | .1 | .2 |
| G-R1–CE9 | 192.168.11.0/24 | .1 | .2 |
| G-R1–CE8 | 192.168.12.0/24 | .1 | .2 |
| G-R2–CE8 | 192.168.13.0/24 | .1 | .2 |
| Gar-R2–CE6 | 172.16.4.0/24 | .1 | .2 |
| Gar-R1–CE4 | 172.16.1.0/24 | .1 | .2 |

**RD / RT design decision (used throughout):**
- **RD is per-SP-per-VRF** (unique so overlapping customer prefixes stay unique in VPNv4): `<SP-ASN>:<vrf-id>`.
- **RT is per-customer, common across all SPs** (so the same customer's sites import each other regardless of which SP they attach to): `<customer-ASN>:<vrf-id>`.

| VRF | On | RD | RT (import+export) |
|-----|----|----|--------------------|
| CUST_A | E-R1, E-R2 (Emerald) | 65100:100 (E-R1), 65100:100 (E-R2) | 65012:100 |
| CUST_A | G-R1, G-R2 (Gold) | 65300:100 | 65012:100 |
| CUST_B | G-R1 (Gold) | 65300:200 | 65013:200 |
| CUST_B | Gar-R1 (Garnet) | 65200:200 | 65013:200 |

> RD differs per SP; RT is identical (`65012:100` for Customer A everywhere). That is what makes inter-AS VPN "just work" once VPNv4 is exchanged across the ASBRs.

---

# Section 1 — VRF + RD + RT Basics (5 tasks)

## Task 1.1 — Create VRF CUST_A on E-R1/E-R2 (Emerald)

**Question:** Create VRF `CUST_A` for Customer A on both Emerald PEs. Use a per-SP RD and the common Customer-A RT `65012:100`.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.2 — Create VRF CUST_A on G-R1/G-R2 (Gold)

**Question:** Extend Customer A into Gold on G-R1 and G-R2. Same RT (`65012:100`), Gold-scoped RD.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.3 — Create VRF CUST_B on G-R1 (Gold) + Gar-R1 (Garnet)

**Question:** Customer B lives in Gold (CE9→G-R1) and Garnet (CE4→Gar-R1). Create VRF `CUST_B` with common RT `65013:200`.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.4 — Assign PE-CE interfaces to VRFs

**Question:** Bind each customer-facing interface into the correct VRF and address it. Remember: on IOS-XR, `vrf X` under the interface **clears the IP**, so re-apply the address after.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 1.5 — RD/RT design summary and validation

**Question:** Prove the design rule "per-SP RD, common RT for the same customer" holds across the fabric, and explain why RD uniqueness matters.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.1 — eBGP PE-CE with CE1/CE2/CE8 (AS 65012, as-override)

**Question:** Customer A CEs all use AS 65012. Because multiple sites share the same AS, the far CE will reject routes that carry `65012` in the AS_PATH. Configure eBGP PE-CE and fix the loop-prevention rejection with **as-override**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.2 — eBGP PE-CE with CE9/CE4 (AS 65013)

**Question:** Customer B uses AS 65013: CE9 in Gold (G-R1) and CE4 in Garnet (Gar-R1). Configure eBGP PE-CE. as-override is still needed because both sites share AS 65013.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.3 — OSPF PE-CE with CE3/CE6 (area 0, DN-bit, domain-id)

**Question:** CE3 (E-R2, Emerald) and CE6 (Gar-R2, Garnet) run OSPF area 0 to the PE. Configure VRF-aware OSPF, redistribute both ways with BGP, and understand the DN-bit and domain-id.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.4 — VRF OSPF sham-link (CE3 backdoor scenario)

**Question:** Customer has a **backdoor link** between two OSPF sites (say CE3's site also has a low-speed direct link to another Customer-A OSPF site). OSPF intra-area routes over the backdoor are always preferred over the MPLS-learned inter-area routes, black-holing the high-speed core. Configure a **sham-link** so the MPLS path competes as an intra-area link.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 2.5 — SoO on dual-homed CE2 and CE8

**Question:** CE2 (dual-homed to E-R1+E-R2) and CE8 (dual-homed to G-R1+G-R2) risk a routing loop: a route from CE2 enters E-R1 → VPNv4 → E-R2 → back to CE2. `as-override` removes the AS-PATH loop protection, so we need **Site-of-Origin (SoO)** to stop a route from being re-advertised back to the site it came from.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.1 — Enable VPNv4 on all RRs and PE↔RR sessions

**Question:** Configure MP-BGP VPNv4 unicast. RRs: **E-R5** (Emerald, 6.6.6.6), **G-R4** (Gold, 24.24.24.24), **Gar-R6** (Garnet, 16.16.16.16). PEs peer to their SP's RR only.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.2 — Verify VPN route propagation within each SP

**Question:** Confirm that Customer A prefixes learned at E-R1 reach E-R2 (Emerald), and Customer A/B prefixes propagate within Gold and Garnet.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.3 — RT import/export behavior (extranet demo)

**Question:** Demonstrate RT control by leaking one Customer-B prefix into Customer A (a simple hub/extranet). Show that RT — not RD — drives import.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 3.4 — Next-hop handling: next-hop-self on RR vs next-hop-unchanged

**Question:** Explain and configure next-hop behavior. Within an SP the RR must not change the next-hop (so PEs forward to the originating PE). For inter-AS Option B/C the ASBR/RR must alter next-hop. Show both.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.1 — Option A between Emerald ↔ Garnet (back-to-back VRF on E-R6/Gar-R7)

**Question:** Implement **Inter-AS Option A** (10A / back-to-back VRF): each ASBR treats the other as a CE. One sub-interface per VRF per customer across the boundary, running eBGP in the VRF.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.2 — Option B (VPNv4 on ASBRs, next-hop-self)

**Question:** Implement **Inter-AS Option B** (10B): ASBRs exchange **VPNv4** directly over eBGP, no VRFs on the ASBR. ASBR rewrites next-hop to itself and re-originates labels.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.3 — Option C (BGP-LU + multihop VPNv4 between RRs)

**Question:** Implement **Inter-AS Option C** (10C): ASBRs exchange only **BGP labeled-unicast (BGP-LU)** for PE loopbacks; the **RRs** hold a **multihop eBGP VPNv4** session and pass VPNv4 with **next-hop-unchanged**. Data plane is end-to-end LSP; no per-VPN state on ASBRs.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.4 — Customer A across Emerald ↔ Gold (Option C with SRv6 transport on Gold side)

**Question:** Extend Customer A from Emerald (LDP transport) into Gold (SRv6 transport) using Option C. Gold uses SRv6 for the VPN; the E-R6↔G-R4 boundary carries the reachability, and RR-to-RR (E-R5↔G-R4) carries VPNv4 with next-hop-unchanged. Show the transport handoff.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.5 — Customer B across Gold ↔ Garnet

**Question:** Customer B: CE9 (Gold, G-R1) ↔ CE4 (Garnet, Gar-R1). Use Option B across the Gold↔Garnet boundary (G-R5 ↔ Gar-R7) for contrast with 4.4's Option C.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 4.6 — Compare Options A / B / C

**Question:** Summarize scalability, config complexity, and use cases.

| Aspect | **Option A** (back-to-back VRF) | **Option B** (VPNv4 on ASBR) | **Option C** (BGP-LU + multihop VPNv4 RR-RR) |
|--------|-------------------------------|------------------------------|----------------------------------------------|
| Data plane at ASBR | IP (per-VRF) | Swap VPN label (new label) | No VPN state; label swap for LU only |
| Control plane | eBGP IPv4 per VRF | eBGP VPNv4 (1 session, all VRFs) | BGP-LU (loopbacks) + multihop eBGP VPNv4 RR↔RR |
| Per-VRF config on ASBR | **Yes** (sub-if + VRF + peer each) | No | No |
| Scalability | Poor (state per customer) | Good | **Best** (no per-VPN state on ASBR) |
| Config complexity | Simple concept, heavy per-customer | Moderate (`retain route-target all`, next-hop-self) | Highest (LU + multihop + next-hop-unchanged) |
| QoS / policy at boundary | **Full** (IP visible) | Limited (labeled) | Limited (labeled) |
| Next-hop handling | N/A (IP) | next-hop-self on ASBR | next-hop-unchanged RR↔RR; LU resolves PE loopbacks |
| RT retention on ASBR | via VRF import | `retain route-target all` | not needed (no VPNv4 on ASBR) |
| Typical use case | Few VRFs, strong policy/billing at edge, distrust between SPs | Moderate VRF count, single trusted boundary | Large scale, same admin or tight partners, end-to-end LSP wanted |

**Rule of thumb:** A = simplest but least scalable and most boundary control; B = the common inter-provider choice; C = intra-company or tightly-coupled SPs needing maximum scale and a single end-to-end LSP.

---

# Section 5 — Advanced VPN (4 tasks)

## Task 5.1 — RT-Constraint (route-target filtering, RFC 4684)

**Question:** RRs push all VPNv4 to every PE, wasting memory on PEs that don't hold that VRF. Enable **RT-Constraint** so each PE advertises the RTs it imports and the RR sends only matching routes.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.2 — BGP PIC Edge for fast VPN failover

**Question:** For dual-homed prefixes (e.g. CE8 via G-R1+G-R2), install a **backup path** in the FIB so failover is prefix-independent (sub-100 ms) instead of waiting for BGP reconvergence.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.3 — Internet access from a VRF (route-leaking, VRF-aware NAT concepts)

**Question:** Give Customer A VRF sites internet access. Show the two common models: (a) **route-leaking** a default from the global table into the VRF (and customer prefixes back), and (b) the VRF-aware NAT concept for overlapping private space.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 5.4 — Per-VRF label vs per-CE label (label allocation modes)

**Question:** Compare the VPN label allocation modes and configure each. Explain the FIB/PPS tradeoff.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Task 6.1 — CE can't reach remote site (RT mismatch)

**Symptom:** CE8 (Gold) cannot reach CE1 (Emerald). VPNv4 route exists on the RR but never appears in CE8's PE VRF.

**Diagnose:**
```
! Route is in VPNv4 on Gold G-R1 but NOT in the VRF:
RP/0/RP0/CPU0:G-R1# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24     ! present
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24                    ! % Not found
! Compare RTs:
RP/0/RP0/CPU0:G-R1# show vrf CUST_A detail | i "Import|Export"
  Import VPN route-target: 65012:999      <-- WRONG (should be 65012:100)
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65100:100 192.168.10.0/24 | i "Extended"
  Extended community: RT:65012:100
```

**Root cause:** G-R1's **import RT** (`65012:999`) does not match the **export RT** carried by the route (`65012:100`).

**Fix:**
```
vrf CUST_A
 address-family ipv4 unicast
  import route-target
   no 65012:999
   65012:100
```

**Verify:**
```
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24    ! now installed (B)
CE8# ping 192.168.10.2                                       ! success
```

---

## Task 6.2 — VPN route received but not installed (next-hop unreachable — missing BGP-LU)

**Symptom:** In Option C, E-R1 has the remote Garnet prefix in VPNv4 but marks it **not best / inaccessible**.

**Diagnose:**
```
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65200:200 192.168.110.0/24
   Not advertised to any peer
   Path ... Next Hop: 11.11.11.11
   ... inaccessible                       <-- next-hop unreachable
RP/0/RP0/CPU0:E-R1# show route 11.11.11.11
   % Network not in table                 <-- no LSP/route to remote PE loopback
RP/0/RP0/CPU0:E-R1# show cef 11.11.11.11/32
   0.0.0.0/0 ... drop
```

**Root cause:** Option C relies on **BGP-LU** to carry the remote PE /32 loopbacks with labels across the AS boundary. The labeled-unicast session (or `allocate-label` / redistribution) is missing, so `11.11.11.11` has no LSP → VPNv4 next-hop is unresolved → route not installed.

**Fix (restore BGP-LU on E-R6):**
```
router bgp 65100
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 10.0.1.2
  remote-as 65200
  address-family ipv4 labeled-unicast
   route-policy PASS in
   route-policy PASS out
```

**Verify:**
```
RP/0/RP0/CPU0:E-R1# show route 11.11.11.11               ! now a BGP-LU / IGP route
RP/0/RP0/CPU0:E-R1# show cef 11.11.11.11/32               ! labeled path (not drop)
RP/0/RP0/CPU0:E-R1# show bgp vpnv4 unicast rd 65200:200 192.168.110.0/24 | i best   ! now best
```

---

## Task 6.3 — as-override not configured (CE rejects route — own AS in path)

**Symptom:** CE8 (AS 65012) does not learn CE1's prefix even though G-R1's VRF has it.

**Diagnose:**
```
RP/0/RP0/CPU0:G-R1# show route vrf CUST_A 192.168.10.0/24          ! present on PE (B)
RP/0/RP0/CPU0:G-R1# show bgp vrf CUST_A neighbors 192.168.12.2 advertised-routes | i 192.168.10
   192.168.10.0/24 ...                                            ! PE IS advertising it
CE8# show ip bgp 192.168.10.0
   % Network not in table                                         ! CE dropped it
CE8# show ip bgp neighbors 192.168.12.1 | i "denied|filtered"
   ... 3 accepted, 2 denied (AS-PATH loop)
```

**Root cause:** The AS_PATH toward CE8 is `65300 65012` (CE1's origin AS 65012 is still present). CE8's own AS is 65012, so its BGP loop-prevention rejects the update. **as-override** was never enabled on G-R1's CE8 neighbor.

**Fix:**
```
router bgp 65300
 vrf CUST_A
  neighbor 192.168.12.2
   address-family ipv4 unicast
    as-override
```

**Verify:**
```
CE8# show ip bgp 192.168.10.0
   65300 65300 65300        ! 65012 rewritten to PE AS → accepted
CE8# ping 192.168.10.2
```
> Also confirm **SoO** is set (Task 2.5) once as-override is on, or you re-open the dual-homing loop.

---

## Task 6.4 — OSPF PE-CE route loop (missing DN-bit or domain-id mismatch)

**Symptom:** Customer C's OSPF route (CE3/CE6 sites) flaps, or a route learned via MPLS gets pushed back into BGP and loops; or remote routes appear as external Type 5 with a bad metric.

**Diagnose:**
```
! Loop symptom: route toggles, or a VPN route re-enters BGP from OSPF:
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 172.16.40.0/24
   ... route churns between OSPF and BGP
! Check the LSA the PE injects — DN-bit should be SET on PE-originated summaries/externals:
RP/0/RP0/CPU0:Gar-R2# show ospf CUST_CE6 vrf CUST_C6 database summary detail | i "Options|DN"
   Options: (No DN)          <-- WRONG, DN-bit not set
! Domain-id mismatch symptom: routes cross as E2 instead of inter-area:
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 | i "O E2"
   O E2 172.16.10.0/24 ...   <-- should be O IA if domain-ids matched
RP/0/RP0/CPU0:E-R2# show run router ospf | i domain-id
   domain-id type 0005 value 000000010000
RP/0/RP0/CPU0:Gar-R2# show run router ospf | i domain-id
   domain-id type 0005 value 000000020000    <-- DIFFERENT → treated as external
```

**Root cause:**
1. **Missing DN-bit** → the PE that receives its own re-advertised LSA redistributes it back into BGP, creating a PE↔PE loop. XR sets the DN-bit automatically when redistributing BGP→OSPF; if `capability vrf-lite` was enabled (which suppresses DN-bit checking) the protection is defeated.
2. **domain-id mismatch** → routes cross the MPLS core as OSPF **external Type 5 (E2)** instead of **inter-area (Type 3)**, changing metrics/preference and potentially causing suboptimal or looping paths with a backdoor.

**Fix:**
```
! 1) Ensure DN-bit protection is active (do NOT set vrf-lite on a real PE):
router ospf CUST_CE6
 vrf CUST_C6
  no capability vrf-lite            ! keep DN-bit enforcement ON
!
! 2) Align domain-id on all PEs serving this customer:
router ospf CUST_CE6
 vrf CUST_C6
  domain-id type 0005 value 000000010000     ! match E-R2's value 1
```

**Verify:**
```
RP/0/RP0/CPU0:Gar-R2# show ospf CUST_CE6 vrf CUST_C6 database summary detail | i DN
   Options: (DN)                     ! DN-bit now set → no re-redistribution loop
RP/0/RP0/CPU0:Gar-R2# show route vrf CUST_C6 | i "O IA"
   O IA 172.16.10.0/24 ...           ! now inter-area, core is transparent
```

---

## Quick Reference — VPN show commands (IOS-XR)

```
show vrf <name> detail                          # RD + import/export RTs
show bgp vpnv4 unicast summary                   # VPNv4 sessions
show bgp vpnv4 unicast rd <RD> <prefix>          # one VPNv4 route
show bgp vpnv4 unicast rt <RT>                   # all routes with an RT
show bgp vpnv4 unicast labels                    # local/rcvd labels
show route vrf <name> <prefix>                   # VRF RIB
show cef vrf <name> <prefix> detail              # VRF FIB + label stack + backup
show bgp vrf <name> neighbors <ce> advertised-routes
show ospf <proc> vrf <name> sham-links
show ospf <proc> vrf <name> database summary detail   # DN-bit check
show bgp ipv4 rt-filter                          # RT-Constraint membership
show bgp ipv4 labeled-unicast <PE-loopback>      # Option C BGP-LU
show mpls forwarding                             # LFIB
```

## Common gotchas

- On IOS-XR, applying `vrf X` to an interface **wipes the IP** — always re-enter `ipv4 address` after.
- `route-policy` is **mandatory** on eBGP neighbors (unlike IOS which permits all by default). A missing in/out policy silently drops everything.
- Inter-AS **Option B** ASBR needs `retain route-target all` or it discards VPNv4 whose RT it can't import.
- Inter-AS **Option C** breaks if PE loopbacks aren't reachable across the boundary (BGP-LU) — the classic "route received but not installed."
- `as-override` without **SoO** on a dual-homed CE re-introduces a routing loop.
- OSPF PE-CE: never enable `capability vrf-lite` on a production PE — it disables DN-bit loop protection.
