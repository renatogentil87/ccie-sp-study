# WB10 — MPLS QoS (DiffServ, MQC)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** QoS on the Green customer L3VPN path: R9 (CE) → R31 (PE) → R3 (P) → R5 (ASBR) → R11 → R13 → R16 (PE) → R18 (CE)
**Level:** CCIE SP
**Prerequisite:** WB00–WB09 complete. Green + Blue VPN working end-to-end. Inter-AS operational.

> **Format:** Tasks only. No configurations provided.
> Every section creates class-maps AND applies an action (police, queue, shape) — no standalone classification.
> **Snapshot** at each section.

> **Test traffic:** Use extended ping with TOS byte to simulate DSCP markings:
> `ping 150.18.18.18 source 150.9.9.9 tos 0xB8 repeat 100` (DSCP EF = voice)
> `ping 150.18.18.18 source 150.9.9.9 tos 0x48 repeat 100` (DSCP AF21 = business)
> `ping 150.18.18.18 source 150.9.9.9 repeat 100` (DSCP 0 = best effort)

> **Router roles:**
> - **R31** (PE): ingress policing for Green customer R9 (via f4/0)
> - **R3** (P): core queuing based on MPLS EXP
> - **R5** (ASBR): H-QoS egress shaping toward R11
> - **R14** (PE): egress SLA shaping toward Blue customer R25

> **DSCP → EXP mapping (uniform mode, default on IOS):**
> DSCP is copied to EXP at label imposition. EXP copied back to DSCP at disposition.
> EF(46)→EXP 5, AF21(18)→EXP 2, CS6(48)→EXP 6, default(0)→EXP 0.

---

## Pre-flight

- [ ] Green VPN: R9 can ping R18 (`ping 150.18.18.18 source 150.9.9.9`) across AS65100→AS65200
- [ ] Blue VPN: R32 can ping R25 (`ping 150.25.25.25 source 150.32.32.32`) across ASes
- [ ] No QoS policy applied anywhere (`show policy-map interface` returns nothing on all routers)
- [ ] Confirm default queuing: `show interface <if>` shows `Queueing strategy: fifo`
- [ ] **Snapshot:** `WB10-00-baseline`

---

## Section 1 — Policing on R31 (PE Ingress, Per-VRF) [5 points]

**Scenario:** Green customer R9 has a 50 Mbps SLA. All traffic from R9 must be policed to 50 Mbps. Voice traffic (DSCP EF) gets a sub-policer at 10 Mbps — excess voice is dropped (not remarked). Business traffic that exceeds the rate is remarked to a higher drop precedence so it's dropped first during congestion in the core.

- [ ] **Task 1.1 [2 points]** — On **R31**, build a policy-map `POLICE-GREEN` with class-maps and actions:
  - class-map `VOICE`: match DSCP EF → police 10 Mbps, conform transmit, exceed drop
  - class-map `BUSINESS`: match-any DSCP AF21 AF22 → police 20 Mbps, conform transmit, exceed set DSCP AF23 (remark to high-drop)
  - class class-default → police 50 Mbps, conform transmit, exceed set DSCP AF13, violate drop
- [ ] **Task 1.2 [1 point]** — Apply `POLICE-GREEN` as `service-policy input` on R31's **f4/0** (facing R9, VRF GREEN).
- [ ] **Task 1.3 [1 point]** — On **R14**, create `POLICE-BLUE` with a single policer on class-default at **30 Mbps**. Conform transmit, exceed drop. Apply on **f3/0** (facing R25, VRF BLUE). This proves each VRF has independent rate limits.
- [ ] **Task 1.4 [1 point]** — Test from R9:
  - `ping 150.18.18.18 source 150.9.9.9 tos 0xB8 repeat 500` (EF → hits VOICE policer)
  - `ping 150.18.18.18 source 150.9.9.9 tos 0x48 repeat 500` (AF21 → hits BUSINESS policer)
  - `ping 150.18.18.18 source 150.9.9.9 repeat 500` (default → hits class-default policer)
  - Check `show policy-map interface f4/0` on R31 — all three classes show conform/exceed/violate counters.

**Verify / Snapshot**
- [ ] `show policy-map interface FastEthernet4/0` on R31 — per-class police counters incrementing
- [ ] `show policy-map interface FastEthernet3/0` on R14 — Blue policer independent
- [ ] **Snapshot:** `WB10-01-policing`

---

## Section 2 — Queuing on R3 (P Router, MPLS Core) [5 points]

**Scenario:** R3 is a P router. During congestion on R3's link toward R5 (g2/0), voice must get strict priority (never dropped unless the priority policer caps it), business data gets guaranteed bandwidth with WRED for graceful drops, and best-effort absorbs the rest. R3 classifies on **MPLS EXP** because traffic is labeled — P routers don't see the IP DSCP inside.

- [ ] **Task 2.1 [2 points]** — On **R3**, build policy-map `CORE-QOS` with class-maps and actions:
  - class-map `VOICE-CORE`: match mpls experimental topmost **5** → `priority percent 20` (strict LLQ)
  - class-map `NETWORK-CTRL`: match mpls experimental topmost **6** → `bandwidth percent 5` (protect routing protocols)
  - class-map `BUSINESS-CORE`: match mpls experimental topmost **2** → `bandwidth percent 25` + `random-detect dscp-based` (WRED drops AF23 before AF21)
  - class class-default → `fair-queue` (best-effort gets whatever remains)
- [ ] **Task 2.2 [1 point]** — Apply `CORE-QOS` as `service-policy output` on R3's **g2/0** (toward R5) and **f0/0** (toward R31, return path).
- [ ] **Task 2.3 [2 points]** — Test all three classes from R9:
  - `ping 150.18.18.18 source 150.9.9.9 tos 0xB8 repeat 200` → check R3: VOICE-CORE matched (EXP 5, priority queue)
  - `ping 150.18.18.18 source 150.9.9.9 tos 0x48 repeat 200` → check R3: BUSINESS-CORE matched (EXP 2, bandwidth + WRED)
  - `ping 150.18.18.18 source 150.9.9.9 repeat 200` → check R3: class-default matched (EXP 0, fair-queue)
  - All three must show incrementing counters in `show policy-map interface g2/0` on R3.

**Verify / Snapshot**
- [ ] `show policy-map interface GigabitEthernet2/0` on R3 — per-class queue depths and counters
- [ ] VOICE-CORE: priority queue, 0 drops
- [ ] BUSINESS-CORE: bandwidth guaranteed, WRED stats visible
- [ ] class-default: fair-queued
- [ ] **Snapshot:** `WB10-02-core-queuing`

---

## Section 3 — H-QoS Shaping on R5 (ASBR Egress) [5 points]

**Scenario:** R5 is an ASBR. The inter-AS link to R11 (g1/0) is 100 Mbps but contracted capacity is 80 Mbps. Shape total egress to 80%. Within the shaped rate: voice gets strict priority (20%), business traffic is policed at 50% with DSCP remarking on exceed, everything else gets DSCP 0 and WRED. This is the **H-QoS pattern** from the CCIE SP reference labs.

- [ ] **Task 3.1 [1 point]** — On **R5**, create child policy `INTER-AS-CHILD`:
  - class-map `VOICE`: match DSCP EF → `priority percent 20` (16 Mbps within 80 Mbps shaped rate)
  - class-map `BUSINESS`: match-any DSCP AF21 AF22 → `police rate percent 50`, conform set DSCP AF21, exceed set DSCP AF12 (remark excess from low-drop to medium-drop)
  - class class-default → `set dscp default` + `random-detect dscp-based`
- [ ] **Task 3.2 [1 point]** — Create parent policy `INTER-AS-PARENT`:
  - class class-default → `shape average percent 80` + `service-policy INTER-AS-CHILD`
- [ ] **Task 3.3 [1 point]** — Apply `INTER-AS-PARENT` as `service-policy output` on R5's **g1/0** (inter-AS link toward R11).
- [ ] **Task 3.4 [2 points]** — Test all classes:
  - `ping 150.18.18.18 source 150.9.9.9 tos 0xB8 repeat 500` → R5: VOICE in child, priority, 0 drops
  - `ping 150.18.18.18 source 150.9.9.9 tos 0x48 repeat 500` → R5: BUSINESS in child, police conform/exceed
  - `ping 150.18.18.18 source 150.9.9.9 repeat 500` → R5: class-default, DSCP set to 0, WRED
  - Check `show policy-map interface g1/0` on R5 — parent shows shaped rate ~80 Mbps, child shows all three classes with counters.

**Verify / Snapshot**
- [ ] `show policy-map interface GigabitEthernet1/0` on R5 — parent shape rate + child per-class stats
- [ ] Voice: 0 drops (priority protected)
- [ ] Business: DSCP AF21→AF12 on exceed visible in counters
- [ ] Default: WRED active
- [ ] **Snapshot:** `WB10-03-hqos-shaping`

---

## Section 4 — PE Egress on R14 (SLA Shaping toward Blue R25) [3 points]

**Scenario:** Blue customer R25 has a 30 Mbps SLA. R14 shapes all egress traffic toward R25 to 30 Mbps. Within that, voice gets strict priority at 30% (9 Mbps). Everything else is fair-queued.

- [ ] **Task 4.1 [1 point]** — On **R14**, create H-QoS for PE-CE egress **f3/0** (toward R25):
  - Parent: class class-default → `shape average 30000000` (30 Mbps) + `service-policy PE-EGRESS-CHILD`
  - Child `PE-EGRESS-CHILD`: class VOICE → `priority percent 30`, class class-default → `fair-queue`
- [ ] **Task 4.2 [1 point]** — Apply `service-policy output` on R14 **f3/0**.
- [ ] **Task 4.3 [1 point]** — Test: from R32 (Blue VRF), `ping 150.25.25.25 source 150.32.32.32 repeat 500`. Check `show policy-map interface f3/0` on R14 — parent shape at 30 Mbps, child voice priority queue active. Confirm only Blue VRF egress is shaped.

**Verify / Snapshot**
- [ ] `show policy-map interface FastEthernet3/0` on R14 — shape rate 30 Mbps + voice priority
- [ ] **Snapshot:** `WB10-04-pe-egress`

---

## Section 5 — End-to-End Verification [0 points — verification]

Walk the full QoS path: R9 → R31 (police) → R3 (core queue) → R5 (H-QoS shape) → R11 → R13 → R16 → R18.

- [ ] **Task 5.1** — Send EF traffic: `ping 150.18.18.18 source 150.9.9.9 tos 0xB8 repeat 500`. Walk each hop:
  - R31 `show policy-map interface f4/0` — VOICE class: policed at 10 Mbps
  - R3 `show policy-map interface g2/0` — VOICE-CORE: priority queue (EXP 5)
  - R5 `show policy-map interface g1/0` — VOICE in H-QoS child: priority, 0 drops
- [ ] **Task 5.2** — Send AF21 traffic: `ping ... tos 0x48 repeat 500`. Verify BUSINESS class at R31 (policed), R3 (bandwidth + WRED), R5 (policed + remarked on exceed).
- [ ] **Task 5.3** — Send default traffic: `ping ... repeat 500`. Verify class-default at each hop.
- [ ] **Task 5.4** — Confirm DSCP preserved end-to-end under uniform mode (EXP copies back to DSCP at egress PE).
- [ ] **Final Snapshot:** `WB10-FINAL`

---

## Scoring Summary

| Section | Max Points | Description |
|---------|-----------|-------------|
| Section 1: Policing (R31 + R14) | 5 | Per-VRF policing, voice/business/default classes with actions |
| Section 2: Core Queuing (R3) | 5 | EXP-based LLQ/CBWFQ/WRED on P router |
| Section 3: H-QoS Shaping (R5 ASBR) | 5 | Parent shape 80% + child priority/police/remark |
| Section 4: PE Egress (R14) | 3 | SLA shaping 30 Mbps toward Blue R25 |
| Section 5: End-to-End Verification | 0 | Walk full path, confirm per-hop behavior |
| **Total** | **18** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Skipped: **0 points**

**Rating:**
```
16-18 points:  CCIE ready for this topic
12-15 points:  Strong, minor gaps
8-11 points:   Needs more practice
Below 8:       Review QoS theory before retrying
```

---

## Completion Criteria

- [ ] Policing: per-VRF on R31 (Green 50 Mbps, voice 10 Mbps sub-policer) + R14 (Blue 30 Mbps)
- [ ] Core queuing: EXP-based LLQ/CBWFQ/WRED on R3, tested with 3 TOS values
- [ ] H-QoS: parent shape 80% + child priority/police/remark on R5, all 3 classes verified
- [ ] PE egress: 30 Mbps SLA shaping on R14 toward R25 with voice priority
- [ ] End-to-end: EF/AF21/default traffic verified at each hop (R31→R3→R5)
- [ ] **Final snapshot:** `WB10-FINAL`
