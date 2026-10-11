# CCIE SP Workbook 11 — QoS DiffServ (Domain 2)

**Platform:** IOS-XRv 9000 — EVE-NG
🔴 **CCIE Prep Platform:** EVE-NG (IOS-XRv 9000) — see `00_EVENG_Topology.md` for the Emerald+Gold topology
**Topology (path under test):** `E-R1(Emerald) → E-R3 → E-R4 → E-R6 → G-R4 → G-R3 → G-R1(Gold)`
**Focus:** ingress classification & marking at the PE trust boundary, DSCP→Traffic-Class (MPLS EXP) in the core, per-hop behaviors, and egress queuing/shaping toward the customer.
**Initial configs:** IGP + MPLS/LDP (or SR) + at least one L3VPN so there is real per-VRF customer traffic to classify (Workbooks 01/03/04).

> **Note:** This workbook is 100% IOS-XR **MQC** — `class-map` → `policy-map` → `service-policy`. The XR QoS model differs from IOS in three important ways you must internalize:
> 1. **No `bandwidth`/`priority` under IOS-style CBWFQ** — XR uses `priority level N`, `bandwidth`, and `bandwidth remaining {ratio|percent}` inside a policy-map class.
> 2. **Marking uses `set`** on the *internal* qualifiers: `set dscp`, `set mpls experimental {imposition|topmost}`, `set traffic-class`, `set qos-group` (the last two are XR-internal labels used to hand classification from an ingress policy to an egress policy on the same node).
> 3. **HQoS is 2-level**: a *parent* class-default with `shape`, and a *child* `service-policy` doing `priority`/`bandwidth`/`queue-limit`/`random-detect`. Queuing actions are **egress-only**; marking/policing can be ingress or egress.
>
> **SP class model used throughout:** EF=voice, AF41=video/real-time-data, AF21=business-critical, default=best-effort (BE). MPLS EXP mapping: EF→5, AF41→4, AF21→2, BE→0.

---

## Section 1 — IOS-XR QoS Model (MQC)

### Task 1.1 — class-map: match DSCP / ACL / protocol
**Question:** On E-R1, define classification for the SP class model. Match EF and AF classes by **DSCP**, match a management flow by **ACL**, and match a control protocol by **protocol**. Show `match-any` vs `match-all` semantics.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — policy-map: mark / police / queue actions
**Question:** Build one ingress `policy-map` that **marks** (set dscp), one that **polices**, and one egress `policy-map` that **queues**. Explain which actions are legal ingress vs egress on XR.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — service-policy: apply ingress/egress + understand the 2-level hierarchy
**Question:** Apply `PM-INGRESS-EDGE` inbound on the E-R1→CE link and `PM-EGRESS-QUEUE` outbound. Then explain the **XR 2-level hierarchy** (parent shaper + child queuing) and why queuing lives only in the child.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — Classification & Marking

### Task 2.1 — Ingress classification at E-R1 (match VRF customer traffic by DSCP)
**Question:** Customer "Emerald" is in VRF `EMERALD` on E-R1. Classify inbound customer traffic **by the DSCP the CE sends**, into the SP model, on the VRF-attached sub-interface.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — Re-mark DSCP at PE ingress (trust boundary enforcement)
**Question:** The PE does **not** trust the customer's DSCP for anything beyond classification. Re-mark the customer's traffic to the SP's canonical DSCP values at ingress so the SP owns the marking from here on.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — DSCP→Traffic-Class → MPLS EXP mapping for the core
**Question:** At **label imposition** on E-R1, map the classified traffic into the **MPLS EXP** bits so the P routers (P1/P2/P6) apply PHBs on EXP alone. Do the DSCP→EXP mapping.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4 — Trust vs re-mark (design decision)
**Question:** Contrast **trusting** the customer DSCP end-to-end vs **re-marking** at the PE. Configure a "trust" variant (classify + map to EXP, keep DSCP) and state when each is correct.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — Policing

### Task 3.1 — Single-rate policer (conform / exceed / violate)
**Question:** On E-R1 ingress, enforce a **single-rate three-color** policer (srTCM) on the voice class: CIR 2 Mbps, Bc 8000 bytes, Be 8000 bytes. Conform→transmit, exceed→re-mark, violate→drop.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — Dual-rate policer (CIR + PIR, trTCM)
**Question:** Enforce a **two-rate three-color** policer (trTCM): CIR 4 Mbps, PIR 8 Mbps on the business-data class. Conform (≤CIR)→transmit, exceed (CIR..PIR)→re-mark down, violate (>PIR)→drop.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — Per-customer (per-VRF) policing + burst calculation (Tc = Bc/CIR)
**Question:** Police customer Emerald's **aggregate** ingress to their sold rate (50 Mbps) on the VRF sub-interface, and **calculate the burst** for a target Tc of 4 ms.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — Queuing & Scheduling

### Task 4.1 — Egress queuing (PQ for EF, bandwidth for AF, default for BE)
**Question:** On E-R1's core-facing egress (toward E-R3), classify on **EXP** and build the queuing policy: strict-priority for EF (policed), bandwidth guarantees for AF, and BE takes the remainder.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — WRED per queue (weighted random early detection)
**Question:** Add **WRED** to the AF4 and BE queues on the core egress, using **EXP-based** minimum/maximum thresholds so higher drop-precedence traffic is discarded first.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — queue-limit
**Question:** Bound the BE queue depth with an explicit **queue-limit** to cap buffering latency, and explain its interaction with WRED.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.4 — bandwidth remaining ratio vs absolute bandwidth
**Question:** Contrast `bandwidth percent`/`bandwidth <rate>` (absolute guarantee) with `bandwidth remaining ratio` (relative share of leftover). Build a policy that uses **remaining ratio** to divide leftover bandwidth after the PQ.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — Shaping

### Task 5.1 — Shape at PE egress (sub-line-rate for customer SLA)
**Question:** E-R1's physical port is 1 Gbps but customer Emerald bought **200 Mbps**. Shape the egress toward the CE to 200 Mbps so the customer never sees more than their SLA (and downstream CE buffers don't overrun).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — Hierarchical policy (parent shape + child queue)
**Question:** Combine Task 5.1 with per-class scheduling: shape Emerald to 200 Mbps (**parent**) and, *within* that envelope, give voice LLQ and split the rest by bandwidth (**child**). This is the flagship XR 2-level HQoS.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — End-to-End QoS (E-R1 Emerald → G-R1 Gold)

### Task 6.1 — Full-path policy placement
**Question:** Assemble the complete DiffServ chain across `E-R1 → E-R3 → E-R4 → E-R6 → G-R4 → G-R3 → G-R1` and state **which policy goes on which node/direction**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — MPLS-EXP per-hop behavior verification
**Question:** Prove each P router applies PHBs on **EXP** and that voice (EXP 5) gets priority hop-by-hop. Show the disposition re-coloring option.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.3 — End-to-end validation (`show policy-map interface`)
**Question:** Validate the whole design under load: confirm classification at E-R1, marking, per-VRF policing, EXP coloring, per-hop queuing, and egress shaping at G-R1 — using `show policy-map interface` at each node.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 7 — Troubleshooting

### Task 7.1 — Drops on egress (queue full — check WRED thresholds / queue-limit)
**Question:** Users report loss on the BE/AF4 traffic on E-R1's core egress under load. `show policy-map interface` shows rising drop counters. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 7.2 — Wrong DSCP at destination (re-marking / service-policy direction)
**Question:** The Gold CE behind G-R1 receives packets with the **wrong DSCP** (e.g., voice arriving as BE, or SP-internal marks leaking to the customer). Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

