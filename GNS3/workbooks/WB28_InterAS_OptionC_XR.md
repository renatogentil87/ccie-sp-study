# WB28 — Inter-AS Option C + IPv6 + EVPN Multipoint (IOS-XR)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Inter-AS Option C (65100↔65300), 6VPE/VPNv6, EVPN multipoint & multi-homing — all three ASes
**Level:** CCIE SP
**Prerequisite:** WB23 (IS-IS XR), WB24 (OSPF XR), WB25 (BGP XR / Option C basics), WB20 (EVPN)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). BGP-LU, multihop VPNv4/VPNv6, 6VPE, and EVPN
> (bridge-domain/EVI/ESI) are **not** available on the GNS3 7200/IOS 15.2 lab (not Dynamips).
> This workbook closes the three remaining gaps: **Inter-AS Option C**, **deep IPv6**, and **EVPN multipoint**.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:**
> Option C boundary **R6** (AS65100 ASBR, 150.1.6.6) ↔ **R20** (AS65300 ASBR/PE, 150.3.20.20).
> RRs: **R7** (150.1.7.7, AS65100), **R8** (150.1.8.8), **R30** (150.3.30.30, AS65300).
> 6VPE Green customer: **R15↔R17** (OSPFv3 VRF) and **R16↔R18** (eBGP IPv6 VRF).
> EVPN Red customer: PEs **R14**/**R16**, L2 CEs **R27**/**R28**, VLAN 100.
> IPv6: 2001:db8:1::/48 (65100), 2001:db8:2::/48 (65200), 2001:db8:3::/48 (65300). Links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm each AS IGP + intra-AS MPLS transport is UP
- [ ] Confirm the R6↔R20 inter-AS link is up
- [ ] Confirm VPNv4 unicast already works intra-AS (from WB25)
- [ ] **Snapshot:** `WB28-00-baseline`

---

## Section 1 — Inter-AS Option C (BGP-LU + multihop VPNv4) [8 points]

- [ ] **Task 1.1 [2 points]** — On the ASBRs **R6 and R20**, enable **`address-family ipv4 labeled-unicast`** on the eBGP session and set **`allocate-label all`** so PE loopbacks cross the boundary with labels.
- [ ] **Task 1.2 [1 point]** — Apply **`next-hop-self`** toward the RRs so the labeled next-hop resolves inside each AS.
- [ ] **Task 1.3 [2 points]** — On the RRs **R7/R8** (AS65100) and **R30** (AS65300), enable **`address-family ipv4 labeled-unicast`** so the RRs allocate labels and reflect the labeled loopbacks to clients.
- [ ] **Task 1.4 [2 points]** — Build the **multihop eBGP VPNv4** session **R7↔R30** (`ebgp-multihop`, `update-source Loopback0`, VPNv4 AF) with **next-hop unchanged** so the originating PE loopback is preserved end-to-end.
- [ ] **Task 1.5 [1 point]** — Verify an **end-to-end LSP with a 3-label stack** (BGP-LU transport label + VPN label, plus intra-AS LDP/SR label as applicable) for a VPN prefix from an AS65100 PE to an AS65300 PE.
- [ ] **Task 1.6 [0 points — verification]** — Confirm the VPNv4 next-hop at the far AS is the remote PE loopback, resolved over BGP-LU.

**Verify / Snapshot**
- [ ] `show bgp ipv4 labeled-unicast` — labeled loopbacks across boundary
- [ ] `show bgp vpnv4 unicast neighbor <R30>` — multihop session UP
- [ ] `show bgp vpnv4 unicast <prefix>` — next-hop = originating PE (unchanged)
- [ ] `show cef vrf <vrf> <prefix> detail` — label stack (3 labels)
- [ ] `traceroute mpls` / end-to-end VRF ping across ASes
- [ ] **Snapshot:** `WB28-01-option-c`

---

## Section 2 — VPNv6 / 6VPE on IOS-XR [5 points]

- [ ] **Task 2.1 [1 point]** — Add **`address-family vpnv6 unicast`** on all PEs and RRs; confirm the AF negotiates on the iBGP (and multihop inter-AS) sessions.
- [ ] **Task 2.2 [2 points]** — Configure **IPv6 PE-CE for Green**: **R15↔R17** via **OSPFv3 in the VRF**, and **R16↔R18** via **eBGP IPv6 in the VRF**. Redistribute into VPNv6 as needed.
- [ ] **Task 2.3 [1 point]** — Confirm Green IPv6 VPN prefixes are advertised as VPNv6 and installed in the remote VRF across the ASes.
- [ ] **Task 2.4 [1 point]** — Verify **IPv6 VPN end-to-end** reachability for Green across the Inter-AS boundary.
- [ ] **Task 2.5 [0 points — verification]** — Confirm the VPNv6 next-hop uses the IPv4-mapped / BGP-LU transport (6VPE over the MPLS core).

**Verify / Snapshot**
- [ ] `show bgp vpnv6 unicast summary` — AF neighbors UP
- [ ] `show bgp vpnv6 unicast rd <rd>` — Green IPv6 prefixes + RTs
- [ ] `show route vrf GREEN ipv6` — remote CE prefixes present
- [ ] `ping vrf GREEN ipv6 <remote CE>` — end-to-end across ASes
- [ ] **Snapshot:** `WB28-02-6vpe`

---

## Section 3 — EVPN Multipoint (ELAN) [5 points]

- [ ] **Task 3.1 [2 points]** — On **R14 and R16** create a **bridge-domain** for the Red customer **VLAN 100** with the AC as a member, and associate an **EVI** (BGP auto-discovery, `l2vpn evpn` AF).
- [ ] **Task 3.2 [1 point]** — Confirm **MAC learning across the bridge-domain**: local MACs advertised as **route-type 2 (MAC/IP)**, remote MACs installed from BGP.
- [ ] **Task 3.3 [1 point]** — Confirm **route-type 3 (inclusive multicast)** is established for BUM flooding between R14 and R16.
- [ ] **Task 3.4 [1 point]** — In notes, **compare EVPN with VPLS**: VFI (flood-and-learn, targeted-LDP) vs bridge-domain + EVPN (BGP control plane, RT-2/RT-3). Reference your WB07 VPLS results.
- [ ] **Task 3.5 [0 points — verification]** — Confirm end-to-end multipoint bridging for Red across R14↔R16.

**Verify / Snapshot**
- [ ] `show l2vpn bridge-domain group RED detail` — ACs + EVI, MAC count
- [ ] `show bgp l2vpn evpn` — RT-2 and RT-3 present
- [ ] `show evpn evi vpn-id 100 mac` — local vs remote MACs
- [ ] **Snapshot:** `WB28-03-evpn-elan`

---

## Section 4 — EVPN Multi-homing [4 points]

Dual-home the Red segment (R27/R28) to both R14 and R16.

- [ ] **Task 4.1 [2 points]** — Configure a shared **ESI** on the Red Ethernet segment on both R14 and R16 (`evpn` → `interface <AC>` → `ethernet-segment identifier type 0 <esi>`), using the SAME ESI on both PEs.
- [ ] **Task 4.2 [1 point]** — Select **all-active vs single-active** redundancy and confirm **DF election** (route-type 4 — Ethernet Segment route).
- [ ] **Task 4.3 [1 point]** — Confirm aliasing / mass-withdraw: remote PEs load-share to the ES via RT-1 (per-ES A-D) and converge fast on a PE-CE link failure.
- [ ] **Task 4.4 [0 points — verification]** — Confirm DF role per PE and split-horizon (local-bias) prevents BUM echo.

**Verify / Snapshot**
- [ ] `show evpn ethernet-segment detail` — ESI, mode, DF result
- [ ] `show bgp l2vpn evpn route-type 4` / `route-type 1` — ES + per-ES A-D
- [ ] Fail one PE-CE link → fast convergence (mass-withdraw)
- [ ] **Snapshot:** `WB28-04-evpn-mh`

---

## Section 5 — Full IPv6 Integration Verification [3 points]

- [ ] **Task 5.1 [1 point]** — Confirm the IPv6 control planes are all up: **IS-IS IPv6** (65100/65300), **OSPFv3** (65200), **MP-BGP IPv6** (global/VPN).
- [ ] **Task 5.2 [1 point]** — Confirm the VPN IPv6 planes: **VPNv6 / 6VPE** installed in VRF GREEN across all three ASes.
- [ ] **Task 5.3 [1 point]** — Perform **end-to-end IPv6 ping across the Inter-AS boundary** (dual-stack reachability) for both global and VPN IPv6.
- [ ] **Task 5.4 [0 points — verification]** — Confirm no IPv4/IPv6 asymmetry (both stacks reach the same endpoints).

**Verify / Snapshot**
- [ ] `show route ipv6 isis` / `show ospfv3 neighbor` / `show bgp ipv6 unicast summary`
- [ ] `show bgp vpnv6 unicast summary` — VPNv6 across ASes
- [ ] `ping ipv6` + `ping vrf GREEN ipv6` end-to-end across Inter-AS
- [ ] **Snapshot:** `WB28-05-ipv6-integration`

---

## Section 6 — Verification & Troubleshooting Drills [3 points]

- [ ] **Drill 6.1 — Option C label stack broken [1 point]:** remove `allocate-label`/`labeled-unicast` on an RR. Confirm the end-to-end LSP collapses (VPN unreachable across ASes); diagnose via `show bgp ipv4 labeled-unicast` and restore.
- [ ] **Drill 6.2 — VPNv6 next-hop unreachable [1 point]:** break the 6VPE transport (missing BGP-LU for the PE loopback). Confirm VPNv6 prefix is received but not installed; diagnose and fix.
- [ ] **Drill 6.3 — EVPN ESI mismatch [1 point]:** misconfigure the ESI (different value per PE). Confirm DF election / aliasing breaks; diagnose with `show evpn ethernet-segment` and fix.
- [ ] **Snapshot:** `WB28-06-tshoot`

**Required show commands (fluency):**
- [ ] `show bgp ipv4 labeled-unicast` / `show bgp vpnv4 unicast` / `show bgp vpnv6 unicast`
- [ ] `show cef vrf <vrf> <prefix> detail`
- [ ] `show bgp l2vpn evpn [route-type 1-5]`
- [ ] `show evpn evi [detail | mac]` / `show evpn ethernet-segment detail`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Inter-AS Option C | 8 | CCIE-level: BGP-LU + multihop VPNv4, 3-label stack |
| Section 2: VPNv6 / 6VPE | 5 | CCIE-level: dual PE-CE (OSPFv3 + eBGP v6) |
| Section 3: EVPN multipoint (ELAN) | 5 | CCIE-level: BD/EVI, RT-2/RT-3, VPLS compare |
| Section 4: EVPN multi-homing | 4 | CCIE-level: ESI, DF election, aliasing |
| Section 5: Full IPv6 integration | 3 | CCIE-level: dual-stack end-to-end |
| Section 6: Troubleshooting Drills | 3 | CCIE-level: break-and-fix |
| **Total** | **28** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
25-28 points:  CCIE ready for this topic
19-24 points:  Strong, minor gaps to review
13-18 points:  Needs more practice
Below 13:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Inter-AS Option C: BGP-LU + multihop VPNv4 R7↔R30, next-hop unchanged, 3-label stack verified
- [ ] VPNv6 / 6VPE for Green (R15↔R17 OSPFv3, R16↔R18 eBGP v6) end-to-end across ASes
- [ ] EVPN multipoint (ELAN) for Red with RT-2/RT-3; VPLS comparison written
- [ ] EVPN multi-homing (ESI, DF election, all-active/single-active, aliasing)
- [ ] Full dual-stack IPv6 reachability across all three ASes (global + VPN)
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB28-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Inter-AS Option C** | **8** | | |
| 1.1 BGP-LU on ASBRs | 2 | | |
| 1.2 next-hop-self to RRs | 1 | | |
| 1.3 labeled-unicast on RRs | 2 | | |
| 1.4 Multihop VPNv4 R7↔R30 | 2 | | |
| 1.5 3-label stack verified | 1 | | |
| **Section 2: VPNv6 / 6VPE** | **5** | | |
| 2.1 vpnv6 AF | 1 | | |
| 2.2 PE-CE OSPFv3 + eBGP v6 | 2 | | |
| 2.3 VPNv6 install | 1 | | |
| 2.4 End-to-end v6 VPN | 1 | | |
| **Section 3: EVPN ELAN** | **5** | | |
| 3.1 BD + EVI | 2 | | |
| 3.2 RT-2 MAC learning | 1 | | |
| 3.3 RT-3 BUM | 1 | | |
| 3.4 VPLS comparison | 1 | | |
| **Section 4: EVPN multi-homing** | **4** | | |
| 4.1 Shared ESI | 2 | | |
| 4.2 Mode + DF election | 1 | | |
| 4.3 Aliasing/mass-withdraw | 1 | | |
| **Section 5: IPv6 integration** | **3** | | |
| 5.1 IPv6 control planes | 1 | | |
| 5.2 VPNv6/6VPE planes | 1 | | |
| 5.3 End-to-end v6 ping | 1 | | |
| **Section 6: Troubleshooting** | **3** | | |
| 6.1 Option C label stack | 1 | | |
| 6.2 VPNv6 next-hop | 1 | | |
| 6.3 EVPN ESI mismatch | 1 | | |
| **TOTAL** | **28** | | |
