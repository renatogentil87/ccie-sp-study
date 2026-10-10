# WB00 — Foundation: IP Addressing

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Reference:** `00_topology_reference.md`
**Prerequisite:** All 31 routers booted, blank configs, console access per Console Port Quick Reference.

> **Style:** Tasks only. No configuration snippets. Figure out the commands yourself — that's the exercise.
> This workbook is the foundation for every later workbook (WB01 IS-IS, WB02 OSPF, WB03 LDP/MPLS).
> Build it correctly: everything downstream assumes Layer 1–3 reachability on directly connected links.

---

## Scope

Bring all physical interfaces, loopbacks, core links, inter-AS links, and PE-CE links to life at Layer 3.
No routing protocols yet — only interface addressing and directly-connected reachability.

**Address plan recap (from topology reference):**
- Loopback0 on every router: `150.<as-digit>.<router>.<router>/32` (CEs use `150.<r>.<r>.<r>/32`).
- AS 65100 core links: `10.X.Y.Z/24`
- AS 65200 core links: `20.X.Y.Z/24`
- AS 65300 core links: `30.X.Y.Z/24`
- Inter-AS links: `10.X.Y.Z/24` (65100↔others) and `20.12.21.x/24` (65200↔65300)
- PE-CE links: `192.168.X.Y/24`
- CE-CE link R9↔R10: `192.168.100.0/24`

---

## Section 1 — Loopbacks & Core Interfaces [5 points]

### Task 1.1 — Base device hygiene (all 31 routers) [1 point]
On every router R1–R32 (including CEs R9, R10, R17, R18, R24, R25, R26, R27, R28, R29, R32):
- Set the hostname to match the node label.
- Disable DNS lookup and configure a console/line setup that won't log you out mid-lab.
- Ensure `ip routing` is enabled (default on, but confirm — it will matter later).

### Task 1.2 — Loopback0 on all routers [1 point]
Configure Loopback0 on every router using the **Loopback Addressing** tables (AS 65100, AS 65200, AS 65300, and CEs).
- All loopbacks are `/32`.
- Confirm each loopback is `up/up` before moving on.

### Task 1.3 — AS 65100 core links (IS-IS domain) [1 point]
Address every link in the **AS 65100 Core** table (R1↔R3, R1↔R2, R2↔R4, R3↔R4, R3↔R5, R3↔R7, R4↔R6, R5↔R8, R5↔R6, R7↔R8).
- Use the exact interface + IP from the link map.
- Bring each interface up (no shutdown).

### Task 1.4 — AS 65200 core links (OSPF domain) [1 point]
Address every link in the **AS 65200 Core** table (R11↔R12, R11↔R13, R11↔R15, R12↔R14, R13↔R16, R13↔R14, R13↔R19, R14↔R19).

### Task 1.5 — AS 65300 core links (IS-IS domain) [1 point]
Address every link in the **AS 65300 Core** table (R20↔R21, R20↔R22, R21↔R23, R22↔R23, R23↔R30).

---

## Section 2 — Inter-AS & Customer Edge [3 points]

### Task 2.1 — Inter-AS links [1 point]
Address the 4 **Inter-AS Links** (R5↔R11, R6↔R12, R6↔R20, R12↔R21).
These are the only L3 touch-points between autonomous systems — double-check the subnet on each side.

### Task 2.2 — PE-CE links [1 point]
Address all PE-CE links from the **PE-CE Links** table that have IPs listed (the eBGP and OSPF PE-CE links).
- Skip the three VLAN 100 L2 links (R1↔R29, R14↔R28, R16↔R27) — those are L2 only, no IP, reserved for later VPLS/VPWS work. Note them as "no L3 address — L2 pseudowire later."

### Task 2.3 — CE-CE link [1 point]
Address the R9↔R10 link (`192.168.100.0/24`) per the **CE-CE Links** table.

---

## Section 3 — Verification & Snapshot [0 points — verification]

### Task 3.1 — Interface sanity [0 points — verification]
On each router, confirm every configured interface is `up/up` with the correct IP. Identify any `administratively down` or `down/down` interfaces and fix the cause (cabling in GNS3, missing `no shutdown`, duplex/speed).

### Task 3.2 — Directly-connected ping sweep [0 points — verification]
From each router, ping the far-end IP of **every directly connected link** it participates in (core, inter-AS, and PE-CE). Build a mental (or written) matrix:
- All AS 65100 core links reachable.
- All AS 65200 core links reachable.
- All AS 65300 core links reachable.
- All 4 inter-AS links reachable.
- All PE-CE L3 links reachable.
- R9↔R10 CE-CE link reachable.

> **Expected:** 100% of directly connected neighbors ping. Loopbacks are NOT yet reachable beyond the local router (no IGP) — that's correct at this stage.

### Task 3.3 — Document failures [0 points — verification]
For any link that fails, note the two routers, interfaces, and the root cause. Do not proceed to WB01 until the ping sweep is clean.

### Task 3.4 — Snapshot [1 point]
Take a GNS3 snapshot once the directly-connected ping sweep is 100% clean.

**Snapshot name:** `WB00-foundation-ip-complete`

---

## Completion Checklist

- [x] Hostnames set on all 31 routers (Task 1.1)
- [x] `ip routing` confirmed enabled everywhere (Task 1.1)
- [x] Loopback0 configured on all routers, `/32`, up/up (Task 1.2)
- [x] AS 65100 core links addressed & up (Task 1.3)
- [x] AS 65200 core links addressed & up (Task 1.4)
- [x] AS 65300 core links addressed & up (Task 1.5)
- [x] All 4 inter-AS links addressed & up (Task 2.1)
- [x] All L3 PE-CE links addressed & up (Task 2.2)
- [x] VLAN 100 L2 links identified and left unaddressed (Task 2.2)
- [x] R9↔R10 CE-CE link addressed & up (Task 2.3)
- [x] All interfaces up/up with correct IPs (Task 3.1)
- [x] Directly-connected ping sweep 100% clean (Task 3.2)
- [x] Failures documented and resolved (Task 3.3)
- [x] Snapshot `WB00-foundation-ip-complete` taken (Task 3.4)

---

## Scoring Summary

| Section | Tasks | Points |
|---|---|---|
| Section 1 — Loopbacks & Core Interfaces | 1.1–1.5 | 5 |
| Section 2 — Inter-AS & Customer Edge | 2.1–2.3 | 3 |
| Section 3 — Verification & Snapshot | 3.1–3.4 (only 3.4 scores) | 1 |
| **Total** | | **9** |

> Foundation addressing is intentionally low-value: it's all basic (1-point) work. The payoff is enabling every downstream workbook.

### Rating Scale (% of points earned)

| Score | Rating |
|---|---|
| 90–100% | CCIE ready |
| 70–89% | Strong, minor gaps |
| 50–69% | Needs more practice |
| Below 50% | Review theory |

### Scoring Rules

- **From memory:** full points
- **Used `?` on router:** 2/3 points (round up)
- **Checked solution:** 1/3 points (round up)
- **Skipped:** 0 points
