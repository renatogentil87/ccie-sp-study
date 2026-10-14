# WB27 — QoS on IOS-XR (H-QoS, MQC)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** QoS (MQC + Hierarchical QoS) on PE/ASBR/P nodes across the topology
**Level:** CCIE SP
**Prerequisite:** WB23/WB24 (IGP), WB25 (BGP/MPLS transport), an L3VPN customer attached for marking tests

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). XR MQC (`policy-map`/`class-map`), H-QoS parent/child shaping,
> and MPLS EXP imposition modes are **not** faithfully modeled on the GNS3 7200/IOS 15.2 lab
> (not Dynamips).
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:** Use an ASBR (e.g. R6 or R20) egress interface for the H-QoS shaper, an ingress
> PE-CE interface (e.g. R15/R16 customer-facing) for marking, and a core P router (R3/R4/R11/R13)
> for EXP-based queuing. Loopbacks/links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm the target interfaces (ASBR egress, PE-CE ingress, core P link) are up and forwarding
- [ ] Confirm MPLS is imposing EXP on labeled traffic through the core
- [ ] Confirm you can generate marked customer traffic for verification
- [ ] **Snapshot:** `WB27-00-baseline`

---

## Section 1 — Classification [2 points]

- [ ] **Task 1.1 [1 point]** — Build **`class-map match-any`** and **`class-map match-all`** examples matching **`match dscp`** values (e.g. EF, AF21, AF11).
- [ ] **Task 1.2 [1 point]** — Add classes matching **`match access-group ipv4`** / **`match access-group ipv6`** and **`match protocol`**; confirm the class-maps resolve traffic correctly.
- [ ] **Task 1.3 [0 points — verification]** — Confirm class-map hit counters increment when matching traffic flows.

**Verify / Snapshot**
- [ ] `show class-map` — class definitions
- [ ] `show policy-map interface <intf>` — per-class match counters
- [ ] **Snapshot:** `WB27-01-classification`

---

## Section 2 — H-QoS Egress Policy [5 points]

Hierarchical: a **parent** shaper with a **child** policy for per-class treatment, applied on an ASBR egress interface.

- [ ] **Task 2.1 [1 point]** — Create the **child policy-map**: class **EF** → `priority` / `bandwidth percent 20` (or `police rate percent 20`).
- [ ] **Task 2.2 [2 points]** — In the child, class **TO-CE** → `police rate percent 50`, `conform-action set dscp af21`, `exceed-action set dscp af12`.
- [ ] **Task 2.3 [1 point]** — In the child, **class class-default** → `set dscp 0` and `random-detect discard-class` (WRED).
- [ ] **Task 2.4 [1 point]** — Create the **parent policy-map**: `class class-default` → `shape average percent 80` with `service-policy <child>` nested; apply the parent **outbound** on the ASBR egress interface.
- [ ] **Task 2.5 [0 points — verification]** — Confirm the hierarchy (parent shape + child classes) is attached and counting.

**Verify / Snapshot**
- [ ] `show policy-map interface <ASBR-egress> output` — parent shape + child classes
- [ ] Counters: EF priority, TO-CE conform/exceed, default WRED drops
- [ ] **Snapshot:** `WB27-02-hqos`

---

## Section 3 — Ingress Marking + DSCP-to-EXP Mapping [3 points]

- [ ] **Task 3.1 [1 point]** — On an **ingress PE-CE** interface, build an input policy that **sets DSCP** per class as traffic enters the SP.
- [ ] **Task 3.2 [1 point]** — Confirm the **DSCP→EXP** imposition when the packet is labeled into the MPLS core (default behavior maps DSCP precedence to EXP).
- [ ] **Task 3.3 [1 point]** — Describe (and where supported, configure) **uniform vs pipe** tunneling modes; explain how each treats EXP↔DSCP at imposition and disposition.
- [ ] **Task 3.4 [0 points — verification]** — Confirm the EXP value carried in the core matches the ingress marking per the chosen mode.

**Verify / Snapshot**
- [ ] `show policy-map interface <PE-CE> input` — set dscp counters
- [ ] Core capture / `show mpls forwarding` reasoning — EXP value
- [ ] Notes: uniform vs pipe behavior
- [ ] **Snapshot:** `WB27-03-marking-exp`

---

## Section 4 — Per-VRF Policing [2 points]

- [ ] **Task 4.1 [1 point]** — Build an **ingress policer** that limits the **total customer VRF traffic** at the ingress PE (aggregate `police rate` applied to the VRF-facing interface/sub-interface).
- [ ] **Task 4.2 [1 point]** — Confirm the policer enforces the aggregate rate (conform/exceed/violate actions) for the customer.
- [ ] **Task 4.3 [0 points — verification]** — Confirm drops occur only above the configured rate.

**Verify / Snapshot**
- [ ] `show policy-map interface <PE ingress> input` — policer conform/exceed/violate
- [ ] **Snapshot:** `WB27-04-vrf-police`

---

## Section 5 — Core EXP Treatment [3 points]

- [ ] **Task 5.1 [2 points]** — On a core **P router**, build a policy that **matches MPLS EXP** values (`match mpls experimental topmost`) and places high-priority EXP into a **priority queue** (and others into bandwidth-reserved classes).
- [ ] **Task 5.2 [1 point]** — Apply the policy on the core-facing interface(s) and confirm EXP-based queuing in the core.
- [ ] **Task 5.3 [0 points — verification]** — Confirm per-EXP class counters increment under load.

**Verify / Snapshot**
- [ ] `show policy-map interface <core-intf> output` — per-EXP class counters
- [ ] `show qos interface <intf> output` — queue mapping
- [ ] **Snapshot:** `WB27-05-core-exp`

---

## Section 6 — Verification & Troubleshooting Drills [2 points]

- [ ] **Drill 6.1 — Policer action wrong [1 point]:** set an incorrect exceed-action (e.g. transmit instead of mark-down). Confirm traffic is not re-marked; diagnose via `show policy-map interface` and fix.
- [ ] **Drill 6.2 — Policy not applied / direction wrong [1 point]:** attach the egress H-QoS policy in the input direction (or omit it). Confirm no shaping occurs; diagnose and correct the attach direction.
- [ ] **Snapshot:** `WB27-06-tshoot`

**Required show commands (fluency):**
- [ ] `show policy-map interface <intf> [input | output]`
- [ ] `show class-map`
- [ ] `show qos interface <intf> [input | output]`
- [ ] `show policy-map interface` — conform/exceed/violate counters

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Classification | 2 | Standard: class-map match criteria |
| Section 2: H-QoS egress policy | 5 | CCIE-level: parent shape + child police/WRED |
| Section 3: Ingress marking + DSCP→EXP | 3 | Intermediate: set dscp + tunneling modes |
| Section 4: Per-VRF policing | 2 | Standard: aggregate policer |
| Section 5: Core EXP treatment | 3 | Intermediate: match EXP + priority queue |
| Section 6: Troubleshooting Drills | 2 | CCIE-level: break-and-fix |
| **Total** | **17** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
15-17 points:  CCIE ready for this topic
12-14 points:  Strong, minor gaps to review
8-11 points:   Needs more practice
Below 8:       Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Class-maps classifying on DSCP / ACL / protocol with hit counters
- [ ] H-QoS parent shaper (80%) with child EF / TO-CE policer / default WRED on ASBR egress
- [ ] Ingress DSCP marking with DSCP→EXP mapping (uniform vs pipe understood)
- [ ] Aggregate per-VRF ingress policer enforcing customer rate
- [ ] Core EXP-based priority queuing on a P router
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB27-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Classification** | **2** | | |
| 1.1 match dscp (match-all/any) | 1 | | |
| 1.2 ACL / protocol | 1 | | |
| **Section 2: H-QoS egress** | **5** | | |
| 2.1 Child EF | 1 | | |
| 2.2 Child TO-CE policer | 2 | | |
| 2.3 Child default + WRED | 1 | | |
| 2.4 Parent shape + apply | 1 | | |
| **Section 3: Marking + EXP** | **3** | | |
| 3.1 Ingress set dscp | 1 | | |
| 3.2 DSCP→EXP mapping | 1 | | |
| 3.3 Uniform vs pipe | 1 | | |
| **Section 4: Per-VRF police** | **2** | | |
| 4.1 Aggregate policer | 1 | | |
| 4.2 Enforcement | 1 | | |
| **Section 5: Core EXP** | **3** | | |
| 5.1 Match EXP + PQ | 2 | | |
| 5.2 Apply + queuing | 1 | | |
| **Section 6: Troubleshooting** | **2** | | |
| 6.1 Policer action wrong | 1 | | |
| 6.2 Policy direction wrong | 1 | | |
| **TOTAL** | **17** | | |
