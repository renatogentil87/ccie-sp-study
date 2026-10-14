# WB24 — OSPF + OSPFv3 on IOS-XR

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** OSPF / OSPFv3 in AS 65200 (R11-R16, R19) — single-area backbone
**Level:** CCIE SP
**Prerequisite:** WB00 equivalent on XR (IP addressing + loopbacks), basic XR CLI fluency

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). XR OSPF/OSPFv3 process syntax, keychains, and TE extensions
> are **not** available on the GNS3 7200/IOS 15.2 lab (not Dynamips).
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference (AS 65200, OSPF Area 0):**
> R11(P) 150.2.11.11 · R12(ASBR) 150.2.12.12 · R13(P) 150.2.13.13 · R14(PE) 150.2.14.14 · R15(PE) 150.2.15.15 · R16(PE) 150.2.16.16 · R19(RR) 150.2.19.19
> IPv6: 2001:db8:2::/48. Green customer PE-CE on R15 (VRF GREEN). Core links per `00_topology_reference.md` (AS 65200 Core).

---

## Pre-flight

- [ ] Confirm all AS 65200 Loopback0 /32s and interface IPv4/IPv6 addresses are configured
- [ ] Confirm core links between R11-R16/R19 are up (line protocol up)
- [ ] Confirm no prior OSPF instance is running (`show ospf` — empty) or baseline cleanly
- [ ] **Snapshot:** `WB24-00-baseline`

---

## Section 1 — OSPF Area 0 on IOS-XR [3 points]

- [ ] **Task 1.1 [1 point]** — Create `router ospf CORE` with an explicit `router-id` (use Loopback0) on R11-R16 and R19.
- [ ] **Task 1.2 [1 point]** — Under `area 0`, add the core interfaces, set `network point-to-point` on P2P links, and mark Loopback0 as `passive`.
- [ ] **Task 1.3 [1 point]** — Confirm full adjacency (FULL state) across the backbone and all Loopback0 /32s in the OSPF RIB.
- [ ] **Task 1.4 [0 points — verification]** — Confirm no DR/BDR election on P2P links (adjacencies form directly).

**Verify / Snapshot**
- [ ] `show ospf neighbor` — all FULL
- [ ] `show ospf interface brief` — P2P + passive loopbacks
- [ ] `show route ospf` — remote loopbacks present
- [ ] **Snapshot:** `WB24-01-area0`

---

## Section 2 — OSPF Authentication (MD5 keychain) [3 points]

- [ ] **Task 2.1 [1 point]** — Create `key chain OSPF-KEY` with an MD5 key (`cryptographic-algorithm MD5`, key-string, send/accept lifetimes).
- [ ] **Task 2.2 [1 point]** — Apply **message-digest** authentication at the `area 0` level (or per interface) referencing the keychain (`authentication keychain OSPF-KEY`).
- [ ] **Task 2.3 [1 point]** — Confirm adjacencies stay UP with authentication; prove enforcement by mismatching a key and observing the neighbor drop, then restore.
- [ ] **Task 2.4 [0 points — verification]** — Confirm authentication type is cryptographic (not null/plaintext).

**Verify / Snapshot**
- [ ] `show key chain OSPF-KEY` — MD5 key + lifetimes
- [ ] `show ospf interface <intf>` — authentication keychain active
- [ ] `show ospf neighbor` — FULL with auth; mismatch test drops/restores
- [ ] **Snapshot:** `WB24-02-auth`

---

## Section 3 — OSPF Advanced (BFD / SPF throttle / TE) [4 points]

- [ ] **Task 3.1 [1 point]** — Enable **BFD for OSPF** on core interfaces (`bfd fast-detect`, `bfd minimum-interval`, `bfd multiplier`). Confirm sub-second detection.
- [ ] **Task 3.2 [1 point]** — Tune **SPF throttle**: `timers throttle spf <start> <hold> <max>` (ms). State the reasoning in notes.
- [ ] **Task 3.3 [1 point]** — Set `auto-cost reference-bandwidth` consistently fleet-wide so high-speed links cost correctly.
- [ ] **Task 3.4 [1 point]** — Enable **OSPF TE extensions**: `mpls traffic-eng` under `area 0` and `mpls traffic-eng router-id Loopback0` under the OSPF process. Confirm TE opaque LSAs are generated.
- [ ] **Task 3.5 [0 points — verification]** — Confirm the TE topology database is populated from OSPF.

**Verify / Snapshot**
- [ ] `show bfd session` — core sessions UP
- [ ] `show ospf` — spf throttle + reference-bandwidth applied
- [ ] `show ospf mpls traffic-eng link` — TE link info advertised
- [ ] `show mpls traffic-eng topology` — populated from OSPF
- [ ] **Snapshot:** `WB24-03-advanced`

---

## Section 4 — OSPFv3 for IPv6 [3 points]

- [ ] **Task 4.1 [1 point]** — Create `router ospfv3 CORE6` with a router-id, `area 0`, and the core interfaces for the IPv6 topology.
- [ ] **Task 4.2 [1 point]** — Enable OSPFv3 on each interface (`ospfv3 CORE6 area 0` under the interface) and mark Loopback0 passive.
- [ ] **Task 4.3 [1 point]** — Verify IPv6 routing: all remote Loopback0 IPv6 /128s present and end-to-end IPv6 ping across AS 65200.
- [ ] **Task 4.4 [0 points — verification]** — Confirm OSPFv3 neighbors reach FULL independently of OSPFv2.

**Verify / Snapshot**
- [ ] `show ospfv3 neighbor` — all FULL
- [ ] `show route ipv6 ospf` — remote loopbacks present
- [ ] `ping ipv6 <remote Loopback0>` — reachable
- [ ] **Snapshot:** `WB24-04-ospfv3`

---

## Section 5 — OSPF PE-CE on IOS-XR [5 points]

Green customer runs OSPF as the PE-CE protocol on R15; redistribute mutually with BGP VPNv4.

- [ ] **Task 5.1 [2 points]** — Create `router ospf GREEN` as a **VRF** process on R15 (`router ospf GREEN vrf GREEN` or `vrf GREEN` under the process) with the PE-CE interface in `area 0` (or the customer area).
- [ ] **Task 5.2 [2 points]** — Configure **mutual redistribution** BGP↔OSPF in the VRF (redistribute OSPF into VPNv4 BGP and BGP into OSPF). Set the **domain-id** and **domain-tag** so routes are correctly identified as inter-area vs external across the VPN backbone.
- [ ] **Task 5.3 [1 point]** — Verify the **DN-bit** behavior: routes redistributed from BGP into OSPF carry the down-bit so a PE won't re-learn its own VPN routes (loop prevention).
- [ ] **Task 5.4 [0 points — verification]** — Confirm the customer CE sees PE routes as OSPF inter-area/external as expected.

**Verify / Snapshot**
- [ ] `show ospf GREEN vrf GREEN neighbor` — CE adjacency FULL
- [ ] `show bgp vpnv4 unicast vrf GREEN` — redistributed OSPF prefixes
- [ ] `show ospf GREEN vrf GREEN database` — DN-bit on BGP-sourced LSAs
- [ ] `show route vrf GREEN ospf` — correct LSA types
- [ ] **Snapshot:** `WB24-05-pe-ce`

---

## Section 6 — OSPF NSF / Graceful Restart Concepts [3 points]

- [ ] **Task 6.1 [1 point]** — Enable **graceful restart** on the OSPF process (`nsf cisco` or `nsf ietf`) and confirm the capability is advertised.
- [ ] **Task 6.2 [1 point]** — Configure a neighbor as a **graceful-restart helper** and confirm it maintains forwarding during the restarting node's control-plane restart.
- [ ] **Task 6.3 [1 point]** — In notes, **explain NSF vs NSR**: NSF/GR relies on neighbors (helper mode) to keep forwarding across an RP switchover, while NSR keeps protocol state on the standby RP so neighbors never notice (no helper dependency). State when each applies.
- [ ] **Task 6.4 [0 points — verification]** — Confirm GR mode/helper status in the OSPF process output.

**Verify / Snapshot**
- [ ] `show ospf` — NSF/GR enabled, mode (cisco/ietf)
- [ ] `show ospf neighbor detail` — helper/restart state
- [ ] Notes: NSF vs NSR written comparison
- [ ] **Snapshot:** `WB24-06-nsf-gr`

---

## Section 7 — Verification & Troubleshooting Drills [4 points]

- [ ] **Drill 7.1 — Network-type mismatch [1 point]:** set one P2P link to broadcast on one side. Confirm adjacency stuck or DR/BDR issues; diagnose via `show ospf interface` and fix.
- [ ] **Drill 7.2 — MTU mismatch [1 point]:** change interface MTU on one side. Confirm neighbor stuck in EXSTART/EXCHANGE; diagnose and fix.
- [ ] **Drill 7.3 — Auth mismatch [1 point]:** mismatch the OSPF keychain. Confirm adjacency drop; diagnose with `show ospf interface` and restore.
- [ ] **Drill 7.4 — Redistribution loop / DN-bit [1 point]:** remove the domain-id/down-bit handling and observe route re-learning or suboptimal routing in VRF GREEN; diagnose and restore.
- [ ] **Snapshot:** `WB24-07-tshoot`

**Required show commands (fluency):**
- [ ] `show ospf [neighbor | interface | database]`
- [ ] `show ospfv3 neighbor`
- [ ] `show route ospf` / `show route ipv6 ospf`
- [ ] `show bgp vpnv4 unicast vrf GREEN`
- [ ] `show bfd session`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: OSPF Area 0 | 3 | Standard: process, area, P2P, passive |
| Section 2: Authentication (keychain) | 3 | Intermediate: MD5 keychain |
| Section 3: Advanced (BFD/SPF/TE) | 4 | Intermediate: fast-detect + TE opaque |
| Section 4: OSPFv3 for IPv6 | 3 | Standard: separate v6 process |
| Section 5: OSPF PE-CE | 5 | CCIE-level: redistribution, domain-id, DN-bit |
| Section 6: NSF / Graceful Restart | 3 | Conceptual: GR helper, NSF vs NSR |
| Section 7: Troubleshooting Drills | 4 | CCIE-level: break-and-fix |
| **Total** | **25** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
22-25 points:  CCIE ready for this topic
17-21 points:  Strong, minor gaps to review
11-16 points:  Needs more practice
Below 11:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] OSPF Area 0 FULL across R11-R16/R19 with P2P + passive loopbacks
- [ ] MD5 keychain authentication enforced and tested
- [ ] BFD, SPF throttle, reference-bandwidth, and OSPF-TE extensions active
- [ ] OSPFv3 IPv6 backbone converged and reachable
- [ ] OSPF PE-CE on R15 with mutual redistribution, domain-id/tag, DN-bit verified
- [ ] NSF/GR enabled; NSF vs NSR written up
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB24-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: OSPF Area 0** | **3** | | |
| 1.1 Process + router-id | 1 | | |
| 1.2 Area 0 + P2P + passive | 1 | | |
| 1.3 FULL + RIB | 1 | | |
| **Section 2: Authentication** | **3** | | |
| 2.1 keychain OSPF-KEY | 1 | | |
| 2.2 message-digest apply | 1 | | |
| 2.3 Enforcement test | 1 | | |
| **Section 3: Advanced** | **4** | | |
| 3.1 BFD for OSPF | 1 | | |
| 3.2 SPF throttle | 1 | | |
| 3.3 Reference-bandwidth | 1 | | |
| 3.4 OSPF-TE extensions | 1 | | |
| **Section 4: OSPFv3** | **3** | | |
| 4.1 ospfv3 process | 1 | | |
| 4.2 Per-interface enable | 1 | | |
| 4.3 IPv6 reachability | 1 | | |
| **Section 5: PE-CE** | **5** | | |
| 5.1 VRF OSPF process | 2 | | |
| 5.2 Mutual redist + domain-id | 2 | | |
| 5.3 DN-bit verification | 1 | | |
| **Section 6: NSF/GR** | **3** | | |
| 6.1 Enable GR | 1 | | |
| 6.2 Helper | 1 | | |
| 6.3 NSF vs NSR writeup | 1 | | |
| **Section 7: Troubleshooting** | **4** | | |
| 7.1 Network-type mismatch | 1 | | |
| 7.2 MTU mismatch | 1 | | |
| 7.3 Auth mismatch | 1 | | |
| 7.4 Redist loop/DN-bit | 1 | | |
| **TOTAL** | **25** | | |
