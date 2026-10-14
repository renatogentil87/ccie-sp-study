# WB17 — SRv6

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** SRv6 within AS 65300 (IS-IS L2 core, R20-R23, R30)
**Level:** CCIE SP
**Prerequisite:** WB15 equivalent (dual-stack IPv6 + IS-IS IPv6 converged on R20-R23, R30)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). SRv6 is **not** supported on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint.

> **AS 65300 reference:**
> R20 150.3.20.20 (ASBR/PE) · R21 150.3.21.21 (ASBR/PE) · R22 150.3.22.22 (PE) · R23 150.3.23.23 (PE) · R30 150.3.30.30 (RR)
> IPv6 Loopback0: R20 2001:db8:3::20 · R21 2001:db8:3::21 · R22 2001:db8:3::22 · R23 2001:db8:3::23 · R30 2001:db8:3::30
> IS-IS area 49.0003, L2-only. Core links per `00_topology_reference.md` (AS 65300 Core, IPv6 section).
>
> **SRv6 locator plan (uSID / C-SID, F3216 format):** base `FC00:0300::/32`, per-node locator `FC00:0300:00XX::/48`
> where XX = node number (R20→0020, R21→0021, R22→0022, R23→0023, R30→0030).

---

## Pre-flight

- [ ] Confirm AS 65300 IS-IS L2 IPv6 is converged (`show isis neighbors`, IPv6 adjacencies UP)
- [ ] Confirm all IPv6 Loopback0 /128s are reachable (`show route ipv6 isis`)
- [ ] Confirm hardware/image supports SRv6 micro-SID (uSID)
- [ ] **Snapshot:** `WB17-00-baseline`

---

## Section 1 — Enable SRv6 + uSID Locator Planning [4 points]

Format **F3216** (uSID: 32-bit block + 16-bit IDs). Locators `FC00:0300:00XX::/48`.

- [ ] **Task 1.1 [1 point]** — Globally enable SRv6 on each node (R20-R23, R30): `segment-routing srv6`.
- [ ] **Task 1.2 [2 points]** — Define a **uSID locator** per node under `segment-routing srv6 locators locator <name>`: prefix `FC00:0300:00XX::/48`, and set the **micro-segment / F3216** behavior (`micro-segment behavior unode psp-usd`). XX = node number.
- [ ] **Task 1.3 [1 point]** — Set the **SRv6 source address = Loopback0 IPv6** (`segment-routing srv6 … source-address <Lo0 v6>` where applicable, or confirm encapsulation source).
- [ ] **Task 1.4 [0 points — verification]** — Confirm the locator is UP and allocated a SID block.

**Verify / Snapshot**
- [ ] `show segment-routing srv6 locator` — locator name/prefix/status per node
- [ ] `show segment-routing srv6 sid` — allocated SIDs + behaviors
- [ ] `show segment-routing srv6 manager` — SRv6 enabled, encap source
- [ ] **Snapshot:** `WB17-01-srv6-locator`

---

## Section 2 — SRv6 under IS-IS + Locator Tag/Logging [4 points]

- [ ] **Task 2.1 [2 points]** — Enable SRv6 in IS-IS: `router isis <tag>` → `address-family ipv6 unicast` → `segment-routing srv6 locator <name>`. Advertise each node's locator into IS-IS.
- [ ] **Task 2.2 [1 point]** — Apply **locator tag 200** so the locator prefixes carry a route tag (for policy / downstream matching), and enable **logging** of SRv6 locator updates.
- [ ] **Task 2.3 [1 point]** — Confirm locators from all nodes are learned via IS-IS and installed with End-behavior SIDs programmed in CEF.
- [ ] **Task 2.4 [0 points — verification]** — Confirm `psp-usd` End behavior is programmed (penultimate segment pop / ultimate segment decap).

**Verify / Snapshot**
- [ ] `show isis segment-routing srv6 locators` — advertised locators, tag 200
- [ ] `show route ipv6 FC00:0300:0020::/48` — tag 200 present
- [ ] `show segment-routing srv6 sid` — End (psp-usd) behavior per SID
- [ ] `show cef ipv6 FC00:0300:0023::/48` — programmed toward R23
- [ ] **Snapshot:** `WB17-02-isis-srv6`

---

## Section 3 — SRv6-based L3VPN [6 points]

Per-VRF SID allocation under BGP (customer **Yellow**: R24 off R20/R22; or **Blue**: R25 off R21/R23).

- [ ] **Task 3.1 [1 point]** — Create the customer VRF on the PEs (R20/R22 for Yellow or R21/R23 for Blue) with RD/RT matching the WB05 policy.
- [ ] **Task 3.2 [2 points]** — Under BGP, enable SRv6 services and allocate a **per-VRF SID** (`segment-routing srv6` under the VRF address-family, allocation mode `per-vrf`). The PE assigns an **End.DT4 / End.DT46** SID from its locator for the VRF.
- [ ] **Task 3.3 [1 point]** — Advertise VPN prefixes with the SRv6 SID attached (VPNv4/VPNv6 over SRv6, no MPLS labels). Confirm the RR (R30) reflects them.
- [ ] **Task 3.4 [2 points]** — Verify end-to-end customer reachability where the transport is **SRv6** (IPv6 header + SRH / uSID) rather than MPLS. Confirm the egress PE performs End.DT decapsulation into the VRF.
- [ ] **Task 3.5 [0 points — verification]** — Confirm the per-VRF SID in the SID table and the imposition on ingress.

**Verify / Snapshot**
- [ ] `show bgp vpnv4 unicast vrf <VRF>` — prefixes with SRv6 SID
- [ ] `show segment-routing srv6 sid` — End.DT4/DT46 per-VRF SID
- [ ] `show cef vrf <VRF> <remote-prefix> detail` — SRv6 encap (uSID) on ingress
- [ ] `traceroute vrf <VRF>` end-to-end across the SRv6 core
- [ ] **Snapshot:** `WB17-03-srv6-l3vpn`

---

## Section 4 — SRv6 Backup Paths + Prefix-Priority [4 points]

- [ ] **Task 4.1 [2 points]** — Enable TI-LFA for SRv6 under IS-IS IPv6 so locator prefixes get an SRv6 repair path on each core interface.
- [ ] **Task 4.2 [1 point]** — Configure **`spf prefix-priority critical`** under IS-IS to prioritize SPF/FIB installation of critical locator (and Loopback0) prefixes ahead of others, speeding SRv6 service restoration. Tie the priority to a prefix-list / tag (tag 200 from Section 2).
- [ ] **Task 4.3 [1 point]** — Trigger a core link failure and confirm the SRv6 backup path is used and critical prefixes converge first.
- [ ] **Task 4.4 [0 points — verification]** — Confirm backup SID list for a protected locator.

**Verify / Snapshot**
- [ ] `show isis fast-reroute summary` — SRv6 protected prefixes
- [ ] `show route ipv6 FC00:0300:0023::/48 detail` — backup path
- [ ] `show isis` — `spf prefix-priority critical` applied
- [ ] Failure test: critical locators restored first
- [ ] **Snapshot:** `WB17-04-srv6-backup`

---

## Section 5 — Verification & Troubleshooting Drills [5 points]

- [ ] **Drill 5.1 — Locator overlap [2 points]:** misconfigure two nodes with the same /48 locator. Observe the SID/route conflict. Diagnose and fix (unique XX per node).
- [ ] **Drill 5.2 — Missing per-VRF SID [2 points]:** remove the SRv6 allocation from one PE's VRF. Confirm the VPN prefixes lose their SID and traffic blackholes. Diagnose via `show segment-routing srv6 sid` + BGP, then restore.
- [ ] **Drill 5.3 — Wrong source-address [1 point]:** change the SRv6 encap source off Loopback0. Confirm reachability / RPF impact and restore.
- [ ] **Snapshot:** `WB17-05-tshoot`

**Required show commands (fluency):**
- [ ] `show segment-routing srv6 locator`
- [ ] `show segment-routing srv6 sid`
- [ ] `show isis segment-routing srv6 locators`
- [ ] `show cef ipv6 <locator>`
- [ ] `show bgp vpnv4 unicast vrf <VRF>`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Enable SRv6 + uSID locator | 4 | Intermediate: F3216 locator plan |
| Section 2: SRv6 under IS-IS + tag/logging | 4 | Intermediate: advertise, psp-usd, tag 200 |
| Section 3: SRv6-based L3VPN | 6 | CCIE-level: per-VRF SID, End.DT decap |
| Section 4: SRv6 backup + prefix-priority | 4 | CCIE-level: TI-LFA, critical priority |
| Section 5: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
| **Total** | **23** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
20-23 points:  CCIE ready for this topic
15-19 points:  Strong, minor gaps to review
10-14 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] SRv6 enabled with F3216 uSID locators (FC00:0300:00XX::/48) on R20-R23, R30
- [ ] Locators advertised in IS-IS with tag 200 + logging; psp-usd End behavior programmed
- [ ] SRv6 source address = Loopback0 IPv6
- [ ] SRv6-based L3VPN with per-VRF SID, End.DT decap, end-to-end reachability
- [ ] SRv6 TI-LFA backup + `spf prefix-priority critical`
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB17-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Enable SRv6 + locator** | **4** | | |
| 1.1 Enable SRv6 | 1 | | |
| 1.2 uSID locator F3216 | 2 | | |
| 1.3 Source address Lo0 v6 | 1 | | |
| **Section 2: SRv6 under IS-IS** | **4** | | |
| 2.1 IS-IS SRv6 locator | 2 | | |
| 2.2 Tag 200 + logging | 1 | | |
| 2.3 Learned fleet-wide | 1 | | |
| **Section 3: SRv6 L3VPN** | **6** | | |
| 3.1 VRF create | 1 | | |
| 3.2 Per-VRF SID under BGP | 2 | | |
| 3.3 Advertise over SRv6 | 1 | | |
| 3.4 End-to-end + End.DT | 2 | | |
| **Section 4: Backup + priority** | **4** | | |
| 4.1 SRv6 TI-LFA | 2 | | |
| 4.2 spf prefix-priority critical | 1 | | |
| 4.3 Failure test | 1 | | |
| **Section 5: Troubleshooting** | **5** | | |
| 5.1 Locator overlap | 2 | | |
| 5.2 Missing per-VRF SID | 2 | | |
| 5.3 Wrong source-address | 1 | | |
| **TOTAL** | **23** | | |
