# WB02 — OSPF (AS 65200)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Reference:** `00_topology_reference.md`
**Prerequisite snapshot:** `WB01-isis-complete` (both IS-IS domains converged; AS 65200 still has only directly-connected L3 from WB00).

> **Style:** Tasks only. No configuration snippets. Derive the commands yourself.
> This workbook builds the OSPF IGP for AS 65200 only. It must not disturb the IS-IS domains from WB01.
> Each section is additive — do not break earlier sections.

---

## Scope

- **AS 65200**: R11, R12, R13, R14, R15, R16, R19 — single-area **OSPF Area 0**.
- Core link subnets only (20.x). Do NOT run OSPF across inter-AS links or (yet) toward CEs, except where a PE-CE link is explicitly OSPF (R15↔R17 is OSPF Area 0 PE-CE — note it but keep CE work for the VPN workbooks unless you choose to stage it here).

---

## Section 1 — OSPF Area 0 on AS 65200 (R11–R16, R19) [6 points]

### Task 1.1 — Enable OSPF, deterministic Router-IDs [1 point]
Enable an OSPF process on R11, R12, R13, R14, R15, R16, R19 and force the **Router-ID from Loopback0** (150.2.x.x) on each — do not rely on dynamic election.

### Task 1.2 — Advertise core links into Area 0 [2 points]
Bring every AS 65200 **core** interface (every interface in the AS 65200 Core link table) and Loopback0 into **Area 0**.
- Do NOT advertise inter-AS links (R5↔R11, R6↔R12, R12↔R21) into OSPF.

### Task 1.3 — Passive loopbacks [1 point]
Make Loopback0 **passive** in OSPF so the /32 is advertised but forms no adjacency on it. (Consider `passive-interface default` + selective activation as the cleaner SP pattern.)

### Task 1.4 — Point-to-point network type on all core links [2 points]
Set the OSPF **network type to point-to-point** on all AS 65200 core links (including the Ethernet/GigE ones) to eliminate DR/BDR election and the associated network LSAs on transit links.

### Task 1.5 — Verify Area 0 adjacencies & routes [0 points — verification]
Confirm all expected OSPF adjacencies reach **FULL**, Router-IDs are the loopbacks, and every 150.2.x.x/32 is reachable from every AS 65200 router. Confirm there are no network (type-2) LSAs on the P2P core links.

---

## Section 2 — OSPF Advanced [18 points]

### Task 2.1 — MD5 authentication [3 points]
Enable OSPF **MD5** authentication across Area 0 (area-wide or per-interface — pick one and be consistent).
- Verify adjacencies survive; test a key mismatch to confirm it drops, then correct.

### Task 2.2 — BFD [3 points]
Enable **BFD** on the AS 65200 core links and register OSPF as a BFD client for sub-second failure detection. Verify BFD sessions UP and OSPF registered.

### Task 2.3 — SPF throttle tuning [5 points]
Tune **SPF throttle timers** (`timers throttle spf`) and LSA throttle/arrival timers for fast-but-damped convergence. Document the start/hold/max values chosen and the reasoning.

### Task 2.4 — Stub / NSSA knowledge check [2 points]
Single-area Area 0 cannot be stub/NSSA (the backbone can't be a stub), so this is a **knowledge task, not a config task**:
- Explain why Area 0 cannot be stub or NSSA.
- Describe what would change if R19/R13 fronted a non-zero area (which LSA types would be filtered under stub vs totally-stubby vs NSSA, and where a type-7→type-5 translation would occur).
- (Optional lab extension) Temporarily carve a second area behind one router to demonstrate stub behavior, then remove it so the single-area baseline is restored before the snapshot.

### Task 2.5 — Prefix suppression [5 points]
Enable OSPF **prefix-suppression** so transit (core /24) prefixes are not advertised, leaving only loopbacks/services in the OSPF RIB. Verify core /24s drop from remote tables while loopbacks remain.

---

## Section 3 — OSPFv3 (IPv6 dual-stack) [4 points]

### Task 3.1 — IPv6 addressing on AS 65200 [2 points]
Add IPv6 to the AS 65200 core interfaces and loopbacks (consistent scheme matching the IPv4 subnets). Enable IPv6 unicast routing.

### Task 3.2 — Enable OSPFv3 for IPv6 [2 points]
Bring up **OSPFv3** (IPv6 AF) on the same core interfaces + loopbacks, Area 0, Router-ID derived from the IPv4 loopback (OSPFv3 still needs a 32-bit RID).
- Reuse the P2P network type and passive-loopback pattern.

### Task 3.3 — Verify dual-stack [0 points — verification]
Confirm OSPFv3 adjacencies FULL, every loopback reachable over IPv6, and both IPv4 (OSPFv2) and IPv6 (OSPFv3) run cleanly side-by-side without affecting each other.

---

## Section 4 — Verification & Troubleshooting [16 points]

### Task 4.1 — Adjacency stuck (EXSTART/EXCHANGE) [5 points]
Create and diagnose an adjacency stuck in **EXSTART/EXCHANGE** — classic **MTU mismatch**. Identify, then fix (match MTU or disable DBD MTU check) and confirm FULL.

### Task 4.2 — Mismatched network type / timers [5 points]
Create a scenario where one side is P2P and the other broadcast (or hello/dead mismatch) so adjacency never forms. Diagnose from the OSPF neighbor/interface state and resolve.

### Task 4.3 — Missing routes / duplicate RID [5 points]
Introduce a **duplicate Router-ID** (two routers claiming the same loopback-derived RID) and observe the symptoms (flapping, missing LSAs). Diagnose and correct.

### Task 4.4 — Final convergence check [0 points — verification]
From R11 reach every AS 65200 loopback; confirm MD5 auth, BFD, SPF throttling, prefix-suppression, and OSPFv3 dual-stack are all simultaneously active. Confirm the IS-IS domains (WB01) are untouched.

### Task 4.5 — Snapshot [1 point]
Take a GNS3 snapshot with AS 65200 OSPF fully converged and all advanced features active.

**Snapshot name:** `WB02-ospf-complete`

---

## Completion Checklist

**Area 0 (Sec 1)**
- [ ] OSPF enabled, Router-ID from Lo0 on R11–R16, R19 (1.1)
- [ ] All core interfaces + Lo0 in Area 0 (1.2)
- [ ] Passive loopbacks (1.3)
- [ ] Point-to-point network type on all core links (1.4)
- [ ] All adjacencies FULL, all 150.2.x.x reachable, no type-2 LSAs on P2P (1.5)

**Advanced (Sec 2)**
- [ ] MD5 authentication, mismatch tested (2.1)
- [ ] BFD for OSPF (2.2)
- [ ] SPF throttle timers tuned & documented (2.3)
- [ ] Stub/NSSA knowledge check answered (2.4)
- [ ] Prefix-suppression verified (2.5)

**OSPFv3 (Sec 3)**
- [ ] IPv6 addressing on AS 65200 (3.1)
- [ ] OSPFv3 Area 0, RID set (3.2)
- [ ] Dual-stack reachability verified (3.3)

**Troubleshooting (Sec 4)**
- [ ] EXSTART/MTU scenario diagnosed & fixed (4.1)
- [ ] Network-type/timer mismatch diagnosed & fixed (4.2)
- [ ] Duplicate RID diagnosed & fixed (4.3)
- [ ] Final convergence check passed (4.4)
- [ ] Snapshot `WB02-ospf-complete` taken (4.5)

---

## Scoring Summary

| Section | Tasks | Points |
|---|---|---|
| Section 1 — OSPF Area 0 on AS 65200 | 1.1–1.5 | 6 |
| Section 2 — OSPF Advanced | 2.1–2.5 | 18 |
| Section 3 — OSPFv3 (IPv6 dual-stack) | 3.1–3.3 | 4 |
| Section 4 — Verification & Troubleshooting | 4.1–4.5 | 16 |
| **Total** | | **44** |

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

---

## Attempt History

### Attempt 1 — October 6, 2026 (combined with WB01, ~1 hour total)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| OSPF enable + router-id | 1 | 1 | ✅ All 7 routers correct |
| Network statements + area 0 | 1 | 1 | ✅ Correct subnets |
| Passive loopback | 1 | 1 | ✅ |
| Point-to-point | 2 | 2 | ✅ Fixed during review — FULL/ - state confirmed on R14 |
| Authentication MD5 | 3 | 2.5 | ✅ Fixed R16 (removed plaintext key). All routers correct now |
| Prefix suppression | 5 | 5 | ✅ ip ospf prefix-suppression on R12, R13, R15, R16 |
| BFD | 3 | 3 | ✅ Config correct |
| SPF tuning | 5 | 5 | ✅ Fixed to 50/200/5000 — confirmed on R14 |
| OSPFv3 IPv6 | 2 | 2 | ✅ All routers |
| **TOTAL** | **~44** | **~22.5 → 34 after fixes** | **77% after fixes — Strong** |

**Initial score (before fixes): ~18/44 (41%)**
**Score after fixes: ~34/44 (77%)**

**Gaps fixed during review:**
- ✅ Added ip ospf network point-to-point on all core interfaces
- ✅ Removed plaintext authentication-key on R16
- ✅ Changed SPF timers to 50/200/5000
**Remaining gap:**
- IS-IS SPF timers still flat (120) — fix in IS-IS domain too
