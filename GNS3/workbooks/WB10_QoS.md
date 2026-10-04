# WB10 — MPLS QoS (DiffServ, MQC)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Scope:** QoS across the L3VPN path PE→P→ASBR→P→PE (AS65100 + AS65200)
**Level:** CCIE SP
**Prerequisite:** WB00-WB07 complete (L3VPN + MPLS forwarding working end-to-end)

> **Format:** Tasks only. No configurations provided (MQC: class-map / policy-map / service-policy).
> Work **progressively** — build classification first, then marking, then enforcement.
> **Snapshot** at each section.

> **Reference path for Section 6:** R1 → R3 → R5 → R11 → R13 → R14 (ingress PE R1, egress PE R14).

---

## Pre-flight

- [ ] End-to-end VPN forwarding works R1 ↔ R14 (baseline, no QoS)
- [ ] Confirm default MPLS EXP behavior (uniform vs pipe) before changes
- [ ] **Snapshot:** `WB10-00-baseline`

---

## Section 1 — Classification

- [ ] **Task 1.1** — Create class-maps matching on **DSCP** (e.g. EF, AF41, AF21, default) for voice / interactive / bulk / best-effort.
- [ ] **Task 1.2** — Create a class-map matching on **protocol** (NBAR `match protocol`) for an application class.
- [ ] **Task 1.3** — Create a class-map matching an **access-group** (ACL) for a customer subnet / flow.
- [ ] **Task 1.4** — Combine with `match-any` vs `match-all` and confirm you understand the logic difference.

**Verify / Snapshot**
- [ ] `show class-map` — all classes + match criteria
- [ ] **Snapshot:** `WB10-01-classification`

---

## Section 2 — Marking + DSCP↔EXP

Mark at PE ingress: **R1** (AS65100) and **R14** (AS65200).

- [ ] **Task 2.1** — Build an ingress policy on R1 and R14 that **sets DSCP** on classified customer traffic as it enters the VPN.
- [ ] **Task 2.2** — Confirm the PE imposition behavior maps **DSCP → MPLS EXP** on the imposed labels. Decide and configure the tunneling mode:
  - [ ] **Uniform** mode (EXP derived from DSCP, changes propagate back on disposition), or
  - [ ] **Pipe / short-pipe** mode (provider EXP independent of customer DSCP).
- [ ] **Task 2.3** — Apply the ingress marking policy to the PE-CE interfaces (R1→R9/R10/R32, R14→R25) as appropriate.
- [ ] **Task 2.4** — Confirm EXP is set on labels in the core and that disposition at the egress PE restores/handles DSCP per the chosen mode.

**Verify / Snapshot**
- [ ] `show policy-map interface <PE-CE>` — marking counters incrementing
- [ ] On a P router: inspect EXP on labeled packets (`show mpls forwarding` + capture / `show policy-map interface` on core)
- [ ] Confirm DSCP preserved/handled correctly at egress per uniform vs pipe
- [ ] **Snapshot:** `WB10-02-marking-exp`

---

## Section 3 — Policing

- [ ] **Task 3.1** — **Single-rate three-color** policer (CIR with conform / exceed / violate actions — e.g. transmit / set-dscp-transmit / drop).
- [ ] **Task 3.2** — **Dual-rate** policer (CIR + PIR, two token buckets) with distinct conform/exceed/violate actions.
- [ ] **Task 3.3** — **Per-VRF policing** at the PE: apply a policer scoped to a customer VRF's ingress so one VPN cannot exceed its contracted rate.
- [ ] **Task 3.4** — Generate traffic above CIR (and above PIR for dual-rate) and confirm the exceed/violate actions trigger.

**Verify / Snapshot**
- [ ] `show policy-map interface <if>` — conform/exceed/violate byte+packet counters
- [ ] Confirm per-VRF policer only affects that VRF
- [ ] **Snapshot:** `WB10-03-policing`

---

## Section 4 — Queuing (egress) + WRED

Apply on a congested core/egress interface.

- [ ] **Task 4.1** — **Priority queue (LLQ)** for EF voice (strict priority + implicit policer).
- [ ] **Task 4.2** — **bandwidth / CBWFQ** guarantees for AF classes (AF4x, AF2x) — percentage or absolute.
- [ ] **Task 4.3** — **fair-queue** for the best-effort / class-default.
- [ ] **Task 4.4** — **WRED per queue** (DSCP-based or EXP-based WRED inside the AF classes) for graceful congestion handling; confirm min/max thresholds + drop profiles.
- [ ] **Task 4.5** — Attach the policy outbound on the chosen interface and generate congestion to exercise all queues.

**Verify / Snapshot**
- [ ] `show policy-map interface <egress>` — per-class queue depth, drops, WRED stats
- [ ] Confirm EF gets priority, AF gets guaranteed BW, BE fair-queued
- [ ] WRED dropping before tail-drop under load
- [ ] **Snapshot:** `WB10-04-queuing-wred`

---

## Section 5 — Shaping

- [ ] **Task 5.1** — Configure **sub-line-rate shaping** at a PE egress (e.g. shape to a customer's purchased SLA rate below physical line rate).
- [ ] **Task 5.2** — Nest a **CBWFQ/LLQ queuing policy inside the shaper** (hierarchical policy: parent shape, child queue) so the SLA sub-rate is internally scheduled.
- [ ] **Task 5.3** — Confirm shaping holds the aggregate to the SLA rate and the child policy prioritizes within it.

**Verify / Snapshot**
- [ ] `show policy-map interface <PE-egress>` — shaper target rate, queue stats under shaper
- [ ] Confirm output rate caps at SLA value under offered overload
- [ ] **Snapshot:** `WB10-05-shaping`

---

## Section 6 — End-to-End Verification

Full path **R1 → R3 → R5 → R11 → R13 → R14**.

- [ ] **Task 6.1** — Mark at ingress R1, confirm EXP across R3/R5 (AS65100 core), across the R5↔R11 inter-AS link, and R11/R13 (AS65200 core).
- [ ] **Task 6.2** — Confirm per-hop behavior: at each hop verify the class treatment with `show policy-map interface`.
- [ ] **Task 6.3** — Confirm disposition at egress R14 handles DSCP per the uniform/pipe decision from Section 2.
- [ ] **Task 6.4** — Run representative traffic (voice-marked + bulk + BE simultaneously) and confirm EF is protected end-to-end under congestion.

**Verify / Snapshot**
- [ ] `show policy-map interface` on R1, R3, R5, R11, R13, R14 — consistent per-hop behavior
- [ ] EXP preserved across inter-AS boundary (R5↔R11)
- [ ] End-to-end latency/drop favoring EF under load
- [ ] **Snapshot:** `WB10-06-end-to-end`

---

## Completion Criteria

- [ ] Classification by DSCP, protocol, and ACL working
- [ ] Ingress marking + DSCP↔EXP mapping (uniform or pipe) confirmed at R1 and R14
- [ ] Single-rate and dual-rate policers + per-VRF policing enforcing rates
- [ ] Egress queuing (LLQ/CBWFQ/fair-queue) + WRED under congestion
- [ ] Sub-line-rate hierarchical shaping at PE egress
- [ ] End-to-end per-hop behavior verified R1→R14
- [ ] **Final snapshot:** `WB10-FINAL`
