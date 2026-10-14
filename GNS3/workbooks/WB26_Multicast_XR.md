# WB26 — Multicast VPN on IOS-XR

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** mVPN in AS 65200 (Garnet SP) — Yellow customer; core R11/R13, PE R16 (+ second PE), RR R19
**Level:** CCIE SP
**Prerequisite:** WB24 (OSPF XR core), WB25 (BGP VPNv4), L3VPN for Yellow established

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). XR multicast-routing, mVPN profiles, mLDP P2MP, and
> `ipv4 mvpn` BGP AF are **not** available on the GNS3 7200/IOS 15.2 lab (not Dynamips).
> CE multicast tasks assume **IOS-XE** CE nodes.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference (AS 65200, Yellow customer):**
> Core P routers **R11** (150.2.11.11) and **R13** (150.2.13.13); PE **R16** (150.2.16.16) + a second PE; RR **R19** (150.2.19.19).
> MDT group range **239.232.0.0/16** (default 239.232.0.1, data 239.232.100.0/24). CE attachments per `00_topology_reference.md` (Yellow).

---

## Pre-flight

- [ ] Confirm AS 65200 OSPF core + MPLS transport is UP (R11, R13, R16, R19)
- [ ] Confirm the Yellow L3VPN (VPNv4) is already working (unicast) on the PEs
- [ ] Confirm RR R19 reflects VPNv4 for Yellow
- [ ] **Snapshot:** `WB26-00-baseline`

---

## Section 1 — Core Multicast (PIM-SM) [3 points]

- [ ] **Task 1.1 [1 point]** — Enable core multicast on the P routers R11 and R13: `multicast-routing` → `address-family ipv4` → enable the core interfaces (`interface <intf> enable`), including Loopback0.
- [ ] **Task 1.2 [1 point]** — Enable **PIM** in the core: `router pim` → `address-family ipv4` → core interfaces; confirm PIM neighbors form between R11/R13 and the PEs.
- [ ] **Task 1.3 [1 point]** — Configure a **PIM-SSM range** (`ipv4 access-list SSM-RANGE` + `ssm range` under multicast-routing) covering the MDT data group range.
- [ ] **Task 1.4 [0 points — verification]** — Confirm PIM adjacencies and SSM range are active.

**Verify / Snapshot**
- [ ] `show pim neighbor` — core PIM adjacencies
- [ ] `show mrib route` — core multicast state
- [ ] `show pim group-map` — SSM range applied
- [ ] **Snapshot:** `WB26-01-core-pim`

---

## Section 2 — mVPN Profile 0 (Default MDT with PIM) [5 points]

- [ ] **Task 2.1 [2 points]** — On the Yellow VRF on R16 and the second PE, configure the **default MDT** under `multicast-routing vrf YELLOW address-family ipv4`: `mdt source Loopback0`, `mdt default ipv4 239.232.0.1`.
- [ ] **Task 2.2 [1 point]** — Add the **data MDT** range `mdt data 239.232.100.0/24` with a threshold so high-rate (S,G)s move off the default MDT.
- [ ] **Task 2.3 [1 point]** — Enable **BGP auto-discovery with PIM** C-multicast signaling (`bgp auto-discovery pim` under the VRF mVPN config / `mdt ... pim`).
- [ ] **Task 2.4 [1 point]** — Confirm the default MDT tunnel forms between the Yellow PEs (PIM over the core) and C-multicast flows.
- [ ] **Task 2.5 [0 points — verification]** — Confirm MDT group join state in the core.

**Verify / Snapshot**
- [ ] `show pim vrf YELLOW neighbor` — PE-PE over MDT
- [ ] `show mrib vrf YELLOW route` — C-(S,G)/(*,G) state
- [ ] `show mfib vrf YELLOW route` — forwarding entries
- [ ] `show pim mdt interface` / `show mvpn vrf YELLOW` — default + data MDT
- [ ] **Snapshot:** `WB26-02-profile0`

---

## Section 3 — mVPN Profile 3 (BGP C-multicast + mLDP P2MP) [5 points]

Replace Profile 0 PIM signaling with BGP-signaled C-multicast over an mLDP P2MP core tree.

- [ ] **Task 3.1 [2 points]** — Enable **mLDP** in the core (`mpls ldp mldp`) and configure the Yellow VRF to use an **mLDP P2MP** default MDT instead of PIM (`mdt default mldp ...`).
- [ ] **Task 3.2 [2 points]** — Switch C-multicast signaling to **BGP** (`bgp auto-discovery mldp` + C-multicast via `ipv4 mvpn`), removing PIM C-signaling across the core.
- [ ] **Task 3.3 [1 point]** — Configure a **route-policy for RPF topology** so RPF uses the correct core tree/next-hop for the mLDP MDT.
- [ ] **Task 3.4 [0 points — verification]** — Confirm the mLDP P2MP LSP is built and C-multicast routes are learned via BGP (not PIM).

**Verify / Snapshot**
- [ ] `show mpls mldp database` — P2MP LSP for the MDT
- [ ] `show bgp ipv4 mvpn` — C-multicast / auto-discovery routes
- [ ] `show mrib vrf YELLOW route` — RPF via mLDP tree
- [ ] **Snapshot:** `WB26-03-profile3`

---

## Section 4 — CE Multicast Config (IOS-XE) [3 points]

- [ ] **Task 4.1 [1 point]** — On the Yellow CEs (IOS-XE), enable `ip multicast-routing` and `ip pim sparse-mode` on the CE interfaces facing the PE and the hosts.
- [ ] **Task 4.2 [1 point]** — Configure a CE as **BSR / RP-candidate** (`ip pim bsr-candidate`, `ip pim rp-candidate`) so the customer PIM domain has an RP.
- [ ] **Task 4.3 [1 point]** — On a test CE/host interface, use `ip igmp join-group <group>` to generate a receiver for an end-to-end test.
- [ ] **Task 4.4 [0 points — verification]** — Confirm the customer RP is elected and the join propagates to the PE.

**Verify / Snapshot**
- [ ] `show ip pim rp mapping` (CE) — RP via BSR
- [ ] `show ip mroute` (CE) — (*,G)/(S,G) toward the group
- [ ] `show ip igmp groups` (CE) — join-group present
- [ ] **Snapshot:** `WB26-04-ce-mcast`

---

## Section 5 — RR Config for mVPN [3 points]

- [ ] **Task 5.1 [1 point]** — On RR **R19**, enable **`address-family ipv4 mvpn`** in BGP.
- [ ] **Task 5.2 [1 point]** — Set **`route-reflector-client`** for the mVPN AF toward the Yellow PEs and `send-community extended`.
- [ ] **Task 5.3 [1 point]** — Confirm the RR reflects mVPN auto-discovery (Type 1) and C-multicast routes between PEs.
- [ ] **Task 5.4 [0 points — verification]** — Confirm PEs receive each other's mVPN routes via the RR.

**Verify / Snapshot**
- [ ] `show bgp ipv4 mvpn summary` — mVPN AF neighbors UP
- [ ] `show bgp ipv4 mvpn` — AD/C-multicast routes reflected
- [ ] **Snapshot:** `WB26-05-rr-mvpn`

---

## Section 6 — Verification & Troubleshooting Drills [3 points]

- [ ] **Drill 6.1 — Default MDT down [1 point]:** mismatch the `mdt default` group between the two PEs. Confirm the MDT tunnel fails and C-multicast breaks; diagnose via `show mvpn`/`show pim vrf` and fix.
- [ ] **Drill 6.2 — RPF failure [1 point]:** break the RPF (missing unicast route / wrong RPF topology policy). Confirm multicast RPF-fails; diagnose with `show mrib route` and fix.
- [ ] **Drill 6.3 — Missing mVPN AF on RR [1 point]:** remove `ipv4 mvpn` on R19. Confirm PEs stop learning each other's mVPN routes; diagnose and restore.
- [ ] **Snapshot:** `WB26-06-tshoot`

**Required show commands (fluency):**
- [ ] `show mrib [vrf YELLOW] route`
- [ ] `show pim [vrf YELLOW] neighbor`
- [ ] `show mfib [vrf YELLOW] route`
- [ ] `show bgp ipv4 mvpn`
- [ ] `show mpls mldp database`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Core multicast (PIM-SM) | 3 | Standard: multicast-routing + PIM + SSM |
| Section 2: Profile 0 (Default MDT/PIM) | 5 | CCIE-level: MDT source/default/data |
| Section 3: Profile 3 (BGP + mLDP P2MP) | 5 | CCIE-level: mLDP tree + BGP C-signaling |
| Section 4: CE multicast (IOS-XE) | 3 | Standard: PIM-SM + BSR/RP + IGMP |
| Section 5: RR config for mVPN | 3 | Intermediate: ipv4 mvpn RR client |
| Section 6: Troubleshooting Drills | 3 | CCIE-level: break-and-fix |
| **Total** | **22** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
19-22 points:  CCIE ready for this topic
15-18 points:  Strong, minor gaps to review
10-14 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Core PIM-SM + SSM range up on R11/R13
- [ ] mVPN Profile 0 default + data MDT (PIM) forwarding Yellow C-multicast
- [ ] mVPN Profile 3 (BGP C-multicast over mLDP P2MP) with RPF topology policy
- [ ] CE multicast (IOS-XE) with BSR/RP and IGMP join test
- [ ] RR R19 reflecting `ipv4 mvpn` to Yellow PEs
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB26-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Core multicast** | **3** | | |
| 1.1 multicast-routing | 1 | | |
| 1.2 router pim | 1 | | |
| 1.3 SSM range | 1 | | |
| **Section 2: Profile 0** | **5** | | |
| 2.1 Default MDT | 2 | | |
| 2.2 Data MDT | 1 | | |
| 2.3 BGP AD pim | 1 | | |
| 2.4 MDT forms | 1 | | |
| **Section 3: Profile 3** | **5** | | |
| 3.1 mLDP P2MP MDT | 2 | | |
| 3.2 BGP C-signaling | 2 | | |
| 3.3 RPF topology policy | 1 | | |
| **Section 4: CE multicast** | **3** | | |
| 4.1 PIM-SM on CE | 1 | | |
| 4.2 BSR/RP-candidate | 1 | | |
| 4.3 IGMP join-group | 1 | | |
| **Section 5: RR mVPN** | **3** | | |
| 5.1 ipv4 mvpn AF | 1 | | |
| 5.2 RR client + ext-comm | 1 | | |
| 5.3 Reflection verified | 1 | | |
| **Section 6: Troubleshooting** | **3** | | |
| 6.1 Default MDT down | 1 | | |
| 6.2 RPF failure | 1 | | |
| 6.3 Missing mVPN AF on RR | 1 | | |
| **TOTAL** | **22** | | |
