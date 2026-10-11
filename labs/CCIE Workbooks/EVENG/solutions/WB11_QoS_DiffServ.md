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

**Solution**
```
ipv4 access-list MGMT-ACL
 10 permit tcp any any eq 22
 20 permit tcp any any eq 830
!
class-map match-any CM-VOICE
 match dscp ef
 end-class-map
!
class-map match-any CM-VIDEO
 match dscp af41 af42 af43
 end-class-map
!
class-map match-any CM-CRITICAL
 match dscp af21 af22 af23
 end-class-map
!
class-map match-all CM-MGMT-SSH          ! match-all = ALL conditions must be true
 match access-group ipv4 MGMT-ACL
 match dscp cs2
 end-class-map
!
class-map match-any CM-CONTROL
 match protocol ospf
 match protocol bgp
 match protocol ldp
 end-class-map
```
- `match-any` (default) = logical **OR** across `match` lines; `match-all` = logical **AND**.
- XR ingress classification can match `dscp`, `precedence`, `access-group`, `protocol` (NBAR-style), `mpls experimental topmost`, `qos-group`, `traffic-class`, `cos`, `dei`.
- The **implicit `class-default`** catches everything unmatched — always plan its treatment (this is BE).

**Verification**
```
show class-map CM-VOICE
show running-config class-map
show policy-map interface <GigE> input      ! per-class match counters once applied
```

---

### Task 1.2 — policy-map: mark / police / queue actions
**Question:** Build one ingress `policy-map` that **marks** (set dscp), one that **polices**, and one egress `policy-map` that **queues**. Explain which actions are legal ingress vs egress on XR.

**Solution**
```
! ----- INGRESS: mark + police (no queuing at ingress) -----
policy-map PM-INGRESS-EDGE
 class CM-VOICE
  set dscp ef
  police rate 2 mbps
   conform-action transmit
   exceed-action drop
  !
 !
 class CM-CRITICAL
  set dscp af21
 !
 class class-default
  set dscp default
 !
 end-policy-map
!
! ----- EGRESS: queue (priority / bandwidth) -----
policy-map PM-EGRESS-QUEUE
 class CM-VOICE
  priority level 1
  police rate 2 mbps                 ! LLQ must be policed to protect other queues
 !
 class CM-VIDEO
  bandwidth percent 30
 !
 class CM-CRITICAL
  bandwidth percent 20
 !
 class class-default
  bandwidth remaining percent 100
 !
 end-policy-map
```
- **Ingress-legal:** `set` (marking), `police`. Queuing (`priority`, `bandwidth`, `shape`, `random-detect`, `queue-limit`) is **egress-only** on XR.
- **Egress-legal:** everything, including queuing. LLQ (`priority`) SHOULD be policed so a priority storm cannot starve other queues.

**Verification**
```
show policy-map pmap-name PM-INGRESS-EDGE
show policy-map pmap-name PM-EGRESS-QUEUE
show qos interface <GigE> output            ! hardware programmed queues/policers
```

---

### Task 1.3 — service-policy: apply ingress/egress + understand the 2-level hierarchy
**Question:** Apply `PM-INGRESS-EDGE` inbound on the E-R1→CE link and `PM-EGRESS-QUEUE` outbound. Then explain the **XR 2-level hierarchy** (parent shaper + child queuing) and why queuing lives only in the child.

**Solution**
```
interface GigabitEthernet0/0/0/1              ! E-R1 -> CE (customer facing)
 service-policy input  PM-INGRESS-EDGE
 service-policy output PM-EGRESS-QUEUE
!
! ---- 2-level HQoS: parent shapes aggregate, child queues within it ----
policy-map PM-CHILD-QUEUE
 class CM-VOICE
  priority level 1
  police rate 2 mbps
 !
 class CM-VIDEO
  bandwidth remaining percent 60
 !
 class class-default
  bandwidth remaining percent 40
 !
 end-policy-map
!
policy-map PM-PARENT-SHAPE
 class class-default
  shape average 100 mbps                      ! parent = aggregate rate to customer
  service-policy PM-CHILD-QUEUE               ! child = per-class scheduling INSIDE the shaper
 !
 end-policy-map
```
- **Parent** matches `class-default` only and does the **shape** (aggregate customer rate). The **child** (nested `service-policy`) does the **queuing** (priority/bandwidth) *within* the shaped envelope.
- XR enforces this: you cannot `shape` and `bandwidth`-schedule sibling classes in one flat policy the way IOS lets you — the queuing MUST be one level below the shaper. This is the "parent shaper + child queuing" 2-level rule.
- Only **2 levels** are supported on most XRv line cards (parent + child). Deeper nesting (3-level: port-shaper → subscriber-shaper → class) exists only on specific ASR9K hardware.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1 input
show policy-map interface GigabitEthernet0/0/0/1 output
show policy-map interface <subif> output     ! shows parent shape stats + child class stats nested
```

---

## Section 2 — Classification & Marking

### Task 2.1 — Ingress classification at E-R1 (match VRF customer traffic by DSCP)
**Question:** Customer "Emerald" is in VRF `EMERALD` on E-R1. Classify inbound customer traffic **by the DSCP the CE sends**, into the SP model, on the VRF-attached sub-interface.

**Solution**
```
class-map match-any CM-EMER-VOICE
 match dscp ef
 end-class-map
!
class-map match-any CM-EMER-VIDEO
 match dscp af41 af42 af43
 end-class-map
!
class-map match-any CM-EMER-DATA
 match dscp af21 af22 af23
 end-class-map
!
policy-map PM-EMER-CLASSIFY-IN
 class CM-EMER-VOICE
  set traffic-class 5                     ! hand classification to egress via internal TC
 !
 class CM-EMER-VIDEO
  set traffic-class 4
 !
 class CM-EMER-DATA
  set traffic-class 2
 !
 class class-default
  set traffic-class 0
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 vrf EMERALD
 ipv4 address 10.1.100.1 255.255.255.0
 encapsulation dot1q 100
 service-policy input PM-EMER-CLASSIFY-IN
```
- The customer's DSCP is only **trusted for classification** here — the PE decides what it means. `set traffic-class` marks an **internal (node-local)** class the egress policy will match; it is not carried on the wire.
- Classification is per-VRF because the `service-policy` is bound to the VRF sub-interface.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input
show qos interface GigabitEthernet0/0/0/1.100 input
! generate DSCP-tagged traffic from the CE; confirm each class' packet/byte counters increment
```

---

### Task 2.2 — Re-mark DSCP at PE ingress (trust boundary enforcement)
**Question:** The PE does **not** trust the customer's DSCP for anything beyond classification. Re-mark the customer's traffic to the SP's canonical DSCP values at ingress so the SP owns the marking from here on.

**Solution**
```
policy-map PM-EMER-REMARK-IN
 class CM-EMER-VOICE
  set dscp ef                             ! canonicalize even if CE sent a near-EF value
 !
 class CM-EMER-VIDEO
  set dscp af41
 !
 class CM-EMER-DATA
  set dscp af21
 !
 class class-default
  set dscp default                        ! everything else -> BE (prevents mark spoofing)
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 service-policy input PM-EMER-REMARK-IN
```
- Re-marking **overwrites** the customer's DSCP with the SP's value. `class-default → set dscp default` is the security control: a customer marking everything EF gets flattened to BE unless it landed in a trusted class.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input   ! "transmit" counters per class
! From CE send DSCP=EF on a data flow; capture on core-facing side -> should be BE (0), not EF.
```

---

### Task 2.3 — DSCP→Traffic-Class → MPLS EXP mapping for the core
**Question:** At **label imposition** on E-R1, map the classified traffic into the **MPLS EXP** bits so the P routers (P1/P2/P6) apply PHBs on EXP alone. Do the DSCP→EXP mapping.

**Solution**
```
class-map match-any CM-TC-VOICE
 match traffic-class 5
 end-class-map
!
class-map match-any CM-TC-VIDEO
 match traffic-class 4
 end-class-map
!
class-map match-any CM-TC-DATA
 match traffic-class 2
 end-class-map
!
policy-map PM-EMER-IMPOSE-EXP
 class CM-TC-VOICE
  set mpls experimental imposition 5      ! set EXP on the labels being PUSHED
 !
 class CM-TC-VIDEO
  set mpls experimental imposition 4
 !
 class CM-TC-DATA
  set mpls experimental imposition 2
 !
 class class-default
  set mpls experimental imposition 0
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 service-policy input PM-EMER-IMPOSE-EXP
```
- `set mpls experimental **imposition**` writes EXP on labels **being pushed** at the ingress PE (used on an ingress policy where labels are imposed).
- `set mpls experimental **topmost**` rewrites EXP on the **outer/top** label of an already-labeled packet — used on **P routers** (Section 6) to re-color in the core.
- The P routers have only the 3-bit EXP (8 classes); the customer IP DSCP is invisible to them. This is why the mapping must happen at imposition.

**Verification**
```
show mpls forwarding-table detail             ! confirm labels + EXP handling
show policy-map interface GigabitEthernet0/0/0/1.100 input
! packet capture E-R1->E-R3: outer label EXP = 5 for voice, 4 video, 2 data, 0 BE
```

---

### Task 2.4 — Trust vs re-mark (design decision)
**Question:** Contrast **trusting** the customer DSCP end-to-end vs **re-marking** at the PE. Configure a "trust" variant (classify + map to EXP, keep DSCP) and state when each is correct.

**Solution**
```
! ---- TRUST variant: classify by DSCP, map to EXP, but DO NOT overwrite DSCP ----
policy-map PM-EMER-TRUST-IN
 class CM-EMER-VOICE
  set mpls experimental imposition 5        ! color the core, leave customer DSCP intact
 !
 class CM-EMER-VIDEO
  set mpls experimental imposition 4
 !
 class CM-EMER-DATA
  set mpls experimental imposition 2
 !
 class class-default
  set mpls experimental imposition 0
 !
 end-policy-map
```
- **Trust:** faster, simpler, and preserves the customer's DSCP end-to-end (good when the customer's marking is contractually respected — pairs with **Pipe / Short-Pipe** tunneling). Risk: a misbehaving customer can influence class selection *if* you also queued on their DSCP.
- **Re-mark (Task 2.2):** the SP owns the marking; protects the core LLQ from mark-spoofing. Required when the SLA sells specific classes and the customer marking cannot be trusted. Pairs with **Uniform** mode if you want core changes reflected back to DSCP.
- Rule of thumb: **police + re-mark at the edge, trust EXP in the core.**

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input
! Trust variant: capture egress-to-CE at G-R1 -> customer DSCP unchanged (still what CE sent).
! Re-mark variant: capture -> SP canonical DSCP.
```

---

## Section 3 — Policing

### Task 3.1 — Single-rate policer (conform / exceed / violate)
**Question:** On E-R1 ingress, enforce a **single-rate three-color** policer (srTCM) on the voice class: CIR 2 Mbps, Bc 8000 bytes, Be 8000 bytes. Conform→transmit, exceed→re-mark, violate→drop.

**Solution**
```
policy-map PM-SR-POLICE-IN
 class CM-EMER-VOICE
  police rate 2 mbps burst 8000 bytes peak-burst 8000 bytes
   conform-action transmit
   exceed-action set dscp af11
   violate-action drop
  !
 !
 end-policy-map
```
- Single-rate three-color: one rate (**CIR**) with two buckets (**Bc** = `burst`, **Be** = `peak-burst`). Conform ≤ Bc, exceed spills into Be, violate exceeds both.
- Two-color simplification: omit `violate-action` and `peak-burst` → conform/exceed only.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input
! look for: "conformed / exceeded / violated" packet+byte counters and rates
```

---

### Task 3.2 — Dual-rate policer (CIR + PIR, trTCM)
**Question:** Enforce a **two-rate three-color** policer (trTCM): CIR 4 Mbps, PIR 8 Mbps on the business-data class. Conform (≤CIR)→transmit, exceed (CIR..PIR)→re-mark down, violate (>PIR)→drop.

**Solution**
```
policy-map PM-DR-POLICE-IN
 class CM-EMER-DATA
  police rate 4 mbps peak-rate 8 mbps
   conform-action set dscp af21
   exceed-action set dscp af22           ! CIR..PIR -> higher drop precedence
   violate-action drop                   ! > PIR -> drop
  !
 !
 end-policy-map
```
- Two-rate: `rate` = **CIR**, `peak-rate` = **PIR**. Conform ≤ CIR; exceed = between CIR and PIR (re-marked to a higher drop-precedence within AF2); violate > PIR (dropped).
- This is the classic AF drop-precedence scheme (af21→af22→af23) letting the core drop the least-important sub-flow first under congestion.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input
! Send 10 Mbps: ~4M conforms (af21), ~4M exceeds (af22), ~2M violates (dropped).
```

---

### Task 3.3 — Per-customer (per-VRF) policing + burst calculation (Tc = Bc/CIR)
**Question:** Police customer Emerald's **aggregate** ingress to their sold rate (50 Mbps) on the VRF sub-interface, and **calculate the burst** for a target Tc of 4 ms.

**Solution**
```
policy-map PM-EMER-AGG-POLICE-IN
 class class-default                        ! aggregate = all of the customer's traffic
  police rate 50 mbps burst 25000 bytes
   conform-action transmit
   exceed-action drop
  !
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 service-policy input PM-EMER-AGG-POLICE-IN
```
**Burst math (Tc = Bc / CIR):**
- Rearranged: **Bc = CIR × Tc**.
- CIR = 50 Mbps = 50,000,000 bits/s. Target Tc = 4 ms = 0.004 s.
- Bc = 50,000,000 × 0.004 = **200,000 bits = 25,000 bytes**.
- So `burst 25000 bytes` gives a ~4 ms interval. Rule of thumb if unspecified: **Bc ≈ CIR × 1.5 ms** (min) up to the RTT for TCP-friendliness; larger Bc = burstier but fewer false drops.
- Per-customer aggregate policing goes on `class-default` of a policy bound to the **VRF sub-interface**, so it counts *all* of that customer's traffic regardless of class.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 input
! Offer 60 Mbps aggregate -> ~50 Mbps conforms, ~10 Mbps drops; verify committed rate.
```

---

## Section 4 — Queuing & Scheduling

### Task 4.1 — Egress queuing (PQ for EF, bandwidth for AF, default for BE)
**Question:** On E-R1's core-facing egress (toward E-R3), classify on **EXP** and build the queuing policy: strict-priority for EF (policed), bandwidth guarantees for AF, and BE takes the remainder.

**Solution**
```
class-map match-any CM-EXP-EF
 match mpls experimental topmost 5
 end-class-map
!
class-map match-any CM-EXP-AF4
 match mpls experimental topmost 4
 end-class-map
!
class-map match-any CM-EXP-AF2
 match mpls experimental topmost 2
 end-class-map
!
policy-map PM-CORE-QUEUE-OUT
 class CM-EXP-EF
  priority level 1
  police rate 30 mbps                     ! LLQ policed so voice can't starve the link
 !
 class CM-EXP-AF4
  bandwidth percent 35
 !
 class CM-EXP-AF2
  bandwidth percent 25
 !
 class class-default
  bandwidth remaining percent 100
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/0                 ! E-R1 -> E-R3 (core)
 service-policy output PM-CORE-QUEUE-OUT
```
- `priority level 1` = strict-priority LLQ (lowest latency, served first); it MUST be policed.
- `bandwidth percent` = a **guaranteed minimum** share under congestion for AF classes.
- BE on `class-default` gets `bandwidth remaining percent 100` (whatever is left after PQ+guarantees).

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/0 output
show qos interface GigabitEthernet0/0/0/0 output
```

---

### Task 4.2 — WRED per queue (weighted random early detection)
**Question:** Add **WRED** to the AF4 and BE queues on the core egress, using **EXP-based** minimum/maximum thresholds so higher drop-precedence traffic is discarded first.

**Solution**
```
policy-map PM-CORE-QUEUE-OUT
 class CM-EXP-AF4
  bandwidth percent 35
  random-detect exp 4 1000 packets 2000 packets     ! min-th max-th for EXP 4
  random-detect exp 3 500 packets 1500 packets       ! EXP 3 drops earlier (more aggressive)
 !
 class class-default
  bandwidth remaining percent 100
  random-detect default                              ! WRED on BE
 !
 end-policy-map
```
- WRED avoids **tail-drop** and TCP global synchronization by dropping probabilistically as the queue fills between **min-threshold** and **max-threshold**.
- **Weighted:** lower-precedence EXP gets a lower min-threshold → dropped sooner. Above max-threshold, all matching packets drop.
- Thresholds may be in `packets`, `bytes`, or `us` (time). Only valid inside an **egress** class that has a queue.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/0 output
! under congestion: "random drops" / "RED" counters rise before "tail drops"; AF4/EXP3 drops first
```

---

### Task 4.3 — queue-limit
**Question:** Bound the BE queue depth with an explicit **queue-limit** to cap buffering latency, and explain its interaction with WRED.

**Solution**
```
policy-map PM-CORE-QUEUE-OUT
 class class-default
  bandwidth remaining percent 100
  queue-limit 50 ms                        ! hard cap: max buffering = 50 ms at the queue's rate
  random-detect default                    ! WRED acts BELOW the queue-limit ceiling
 !
 end-policy-map
```
- `queue-limit` is the **absolute maximum** queue depth (packets / bytes / **ms**). Once hit → tail-drop.
- WRED starts dropping *before* the queue-limit; the queue-limit is the hard ceiling if WRED's max-threshold is set higher or WRED is absent. WRED **max-threshold MUST be ≤ queue-limit** or it never triggers.
- Expressing it in **ms** ties buffering directly to a latency budget (bufferbloat control).

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/0 output    ! "queue-limit" + drop counters
show qos interface GigabitEthernet0/0/0/0 output           ! programmed depth in HW
```

---

### Task 4.4 — bandwidth remaining ratio vs absolute bandwidth
**Question:** Contrast `bandwidth percent`/`bandwidth <rate>` (absolute guarantee) with `bandwidth remaining ratio` (relative share of leftover). Build a policy that uses **remaining ratio** to divide leftover bandwidth after the PQ.

**Solution**
```
policy-map PM-BRR-OUT
 class CM-EXP-EF
  priority level 1
  police rate 20 mbps
 !
 class CM-EXP-AF4
  bandwidth remaining ratio 4              ! gets 4/(4+2+1) of leftover
 !
 class CM-EXP-AF2
  bandwidth remaining ratio 2              ! 2/7 of leftover
 !
 class class-default
  bandwidth remaining ratio 1              ! 1/7 of leftover
 !
 end-policy-map
```
- **Absolute** (`bandwidth percent 35` / `bandwidth 350 mbps`) = a guaranteed minimum tied to the interface/parent rate. If the parent shaper rate changes, the *meaning* of the percent scales but the intent is a floor.
- **`bandwidth remaining ratio N`** = a *relative weight* dividing only the **leftover** bandwidth (after priority + absolute guarantees). Ratios need not sum to 100; the share = `own_ratio / Σ ratios`.
- Remaining-ratio is the natural fit under a **parent shaper** where the absolute rate is variable — it stays proportionally correct as the shaped rate changes. Do **not** mix `bandwidth percent` and `bandwidth remaining` in ways XR rejects; keep the scheme consistent within one policy.

**Verification**
```
show policy-map interface <egress> output
! congest the link: leftover splits 4:2:1 among AF4:AF2:BE after EF is served.
```

---

## Section 5 — Shaping

### Task 5.1 — Shape at PE egress (sub-line-rate for customer SLA)
**Question:** E-R1's physical port is 1 Gbps but customer Emerald bought **200 Mbps**. Shape the egress toward the CE to 200 Mbps so the customer never sees more than their SLA (and downstream CE buffers don't overrun).

**Solution**
```
policy-map PM-EMER-SHAPE-OUT
 class class-default
  shape average 200 mbps
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 service-policy output PM-EMER-SHAPE-OUT
```
- `shape average` **buffers and meters** to the target rate (smooths bursts) rather than dropping like a policer. Sub-line-rate shaping is how you sell a fraction of a physical port.
- Shaping is **egress-only** and belongs on `class-default` when it's the aggregate customer rate (becomes the parent in HQoS — Task 5.2).

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 output
! offer 400 Mbps -> output metered to ~200 Mbps; "shape" queue depth/delay counters present
```

---

### Task 5.2 — Hierarchical policy (parent shape + child queue)
**Question:** Combine Task 5.1 with per-class scheduling: shape Emerald to 200 Mbps (**parent**) and, *within* that envelope, give voice LLQ and split the rest by bandwidth (**child**). This is the flagship XR 2-level HQoS.

**Solution**
```
policy-map PM-EMER-CHILD-OUT
 class CM-EMER-VOICE
  priority level 1
  police rate 40 mbps
 !
 class CM-EMER-VIDEO
  bandwidth remaining percent 50
 !
 class CM-EMER-DATA
  bandwidth remaining percent 30
 !
 class class-default
  bandwidth remaining percent 20
 !
 end-policy-map
!
policy-map PM-EMER-PARENT-OUT
 class class-default
  shape average 200 mbps                  ! PARENT: aggregate SLA envelope
  service-policy PM-EMER-CHILD-OUT        ! CHILD: per-class queuing INSIDE the 200 Mbps
 !
 end-policy-map
!
interface GigabitEthernet0/0/0/1.100
 service-policy output PM-EMER-PARENT-OUT
```
- **Parent** = `class-default` + `shape` (aggregate rate). **Child** = nested `service-policy` doing `priority`/`bandwidth remaining` — the scheduling only divides the shaped 200 Mbps, not the full 1 Gbps.
- Use `bandwidth remaining percent` in the child so the shares track the shaped rate (see Task 4.4). This is the per-customer edge QoS product.

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.100 output
! nested output: parent shape stats + child class stats; congest -> EF served first, then 50/30/20 split of the 200 Mbps.
```

---

## Section 6 — End-to-End QoS (E-R1 Emerald → G-R1 Gold)

### Task 6.1 — Full-path policy placement
**Question:** Assemble the complete DiffServ chain across `E-R1 → E-R3 → E-R4 → E-R6 → G-R4 → G-R3 → G-R1` and state **which policy goes on which node/direction**.

**Solution**
```
! ===== E-R1 (ingress PE, Emerald) =====
interface GigabitEthernet0/0/0/1.100            ! E-R1 -> CE
 service-policy input  PM-EMER-CLASSIFY-REMARK-POLICE-IN   ! classify + re-mark + per-VRF police + set EXP imposition
!
interface GigabitEthernet0/0/0/0                ! E-R1 -> E-R3 (core)
 service-policy output PM-CORE-QUEUE-OUT                   ! queue on EXP (LLQ + BW + WRED)
!
! ===== E-R3 / E-R4 / G-R3 (core P routers) =====
interface <core-link>
 service-policy input  PM-CORE-EXP-CLASSIFY-IN            ! (optional) re-color: set mpls experimental topmost
 service-policy output PM-CORE-QUEUE-OUT                  ! queue on EXP each hop
!
! ===== E-R6 / G-R4 (inter-AS boundary) =====
interface <ASBR-link>
 service-policy input  PM-ASBR-EXP-TRUST-IN               ! trust/re-color EXP at the AS edge
 service-policy output PM-CORE-QUEUE-OUT
!
! ===== G-R1 (egress PE, Gold) =====
interface <G-R1 core-facing>                     ! disposition: EXP -> DSCP per tunneling mode
 service-policy input  PM-G-R1-DISPOSITION-IN
!
interface GigabitEthernet0/0/0/1.200            ! G-R1 -> Gold CE
 service-policy output PM-GOLD-PARENT-OUT                 ! HQoS shape + child queue toward customer
```
- **Ingress PE (E-R1):** classify → re-mark DSCP → police per-VRF → `set mpls experimental imposition` (color the core). Queue on the core egress.
- **Core (P1/P2/P6) & ASBRs:** classify on **EXP only** (`match mpls experimental topmost`), queue with LLQ/CBWFQ/WRED; optionally `set mpls experimental topmost` to re-color under policy.
- **Egress PE (G-R1):** disposition — map EXP (or preserved DSCP) back to the customer per tunneling mode, then HQoS shape+queue toward the Gold CE.

**Verification**
```
show policy-map interface <iface> {input|output}    ! on EVERY hop, walk the chain
```

---

### Task 6.2 — MPLS-EXP per-hop behavior verification
**Question:** Prove each P router applies PHBs on **EXP** and that voice (EXP 5) gets priority hop-by-hop. Show the disposition re-coloring option.

**Solution**
```
! Core re-color example (E-R4 under a policy decision):
policy-map PM-CORE-RECOLOR-IN
 class CM-EXP-EF
  set mpls experimental topmost 5         ! topmost = rewrite the OUTER label's EXP in transit
 !
 end-policy-map
!
! G-R1 disposition (Uniform mode): copy final EXP back into customer DSCP
policy-map PM-G-R1-DISPOSITION-IN
 class CM-EXP-EF
  set dscp ef
 !
 class CM-EXP-AF4
  set dscp af41
 !
 class class-default
  set dscp default
 !
 end-policy-map
```
- **`imposition`** (E-R1) writes EXP on labels being pushed; **`topmost`** (P/ASBR) rewrites the outer label EXP already on the wire. Do not confuse them.
- Uniform-mode disposition copies the (possibly core-changed) EXP back into DSCP; Pipe/Short-Pipe would preserve the original customer DSCP instead.

**Verification**
```
show mpls forwarding-table detail
show policy-map interface <core-link> output      ! EXP5 hits LLQ, EXP4/2 hit CBWFQ, WRED drops
! capture at each hop: outer-label EXP consistent; voice served with lowest latency end-to-end
```

---

### Task 6.3 — End-to-end validation (`show policy-map interface`)
**Question:** Validate the whole design under load: confirm classification at E-R1, marking, per-VRF policing, EXP coloring, per-hop queuing, and egress shaping at G-R1 — using `show policy-map interface` at each node.

**Solution**
```
! Run at each node along E-R1 -> E-R3 -> E-R4 -> E-R6 -> G-R4 -> G-R3 -> G-R1:
show policy-map interface GigabitEthernet0/0/0/1.100 input     ! E-R1: class hits + police conform/exceed + remark
show policy-map interface GigabitEthernet0/0/0/0   output      ! E-R1 core: EXP queues, LLQ/CBWFQ/WRED
show policy-map interface <link> output                        ! P1/P2/P6/ASBR: EXP PHB per hop
show policy-map interface GigabitEthernet0/0/0/1.200 output    ! G-R1: parent shape + child queue toward Gold CE
!
show qos interface <iface> {input|output}                      ! HW-programmed policers/queues
show mpls forwarding-table detail                              ! label + EXP path
```
- Walk the whole chain: **E-R1 in** (classify/mark/police) → **E-R1 out** (EXP queue) → **core hops** (EXP PHB) → **G-R1 out** (shape+queue). Every stage's counters should move under a mixed EF/AF/BE load test.

**Verification**
```
! End-to-end SLA sanity: run IP SLA / synthetic probes for EF across E-R1->G-R1.
! Expect: voice <150 ms one-way, low jitter, near-zero loss even while BE is congested.
```

---

## Section 7 — Troubleshooting

### Task 7.1 — Drops on egress (queue full — check WRED thresholds / queue-limit)
**Question:** Users report loss on the BE/AF4 traffic on E-R1's core egress under load. `show policy-map interface` shows rising drop counters. Diagnose and fix.

**Solution**
```
show policy-map interface GigabitEthernet0/0/0/0 output
! Symptom: "tail drops" climbing (not "random/RED drops") -> queue hitting queue-limit with no/late WRED.
```
**Root causes & fixes:**
1. **WRED max-threshold > queue-limit** → WRED never engages, everything tail-drops. Fix: ensure `random-detect` max-th **≤** `queue-limit`.
2. **queue-limit too small** for the class's bandwidth → premature tail-drop. Fix: raise `queue-limit` (express in `ms` to bound latency) OR raise the class `bandwidth`.
3. **No WRED at all** on a TCP-heavy class → global sync + tail-drop. Fix: add `random-detect`.
```
policy-map PM-CORE-QUEUE-OUT
 class CM-EXP-AF4
  bandwidth percent 35
  queue-limit 50 ms
  random-detect exp 4 25 ms 40 ms          ! min/max-th BELOW the 50 ms queue-limit
 !
 class class-default
  bandwidth remaining percent 100
  queue-limit 60 ms
  random-detect default
 !
 end-policy-map
```

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/0 output
! After fix: "random drops" rise gently, "tail drops" ~0; throughput smoother, no TCP sawtooth collapse.
show qos interface GigabitEthernet0/0/0/0 output      ! confirm programmed thresholds vs queue-limit
```

---

### Task 7.2 — Wrong DSCP at destination (re-marking / service-policy direction)
**Question:** The Gold CE behind G-R1 receives packets with the **wrong DSCP** (e.g., voice arriving as BE, or SP-internal marks leaking to the customer). Diagnose and fix.

**Solution**
```
! Check every stage's policy AND its direction:
show policy-map interface GigabitEthernet0/0/0/1.100 input    ! E-R1 ingress: is re-mark/EXP set applied?
show policy-map interface GigabitEthernet0/0/0/1.200 output   ! G-R1 egress: is disposition applied here?
show running-config interface GigabitEthernet0/0/0/1.200      ! confirm 'service-policy OUTPUT' present
```
**Root causes & fixes:**
1. **Service-policy on the wrong direction** — disposition/queuing policy applied `input` instead of `output` (or vice-versa). Queuing is egress; DSCP disposition toward the CE is on the **G-R1 egress**. Fix the direction.
2. **Missing disposition policy at G-R1** — EXP was never mapped back to DSCP (Uniform) so the customer sees the core's marks. Fix: apply `PM-G-R1-DISPOSITION-IN`.
3. **Wrong tunneling-mode expectation** — Short-Pipe should hand back the **customer's original DSCP**, but a Uniform-style `set dscp` overwrote it from EXP. Match the disposition policy to the intended mode.
4. **`set traffic-class`/`set qos-group` treated as on-wire** — these are node-internal only; a downstream node won't see them. Use `set dscp` / `set mpls experimental` for anything that must survive the hop.
```
interface GigabitEthernet0/0/0/1.200
 no service-policy input  PM-G-R1-DISPOSITION-IN     ! remove wrong-direction application
 service-policy output PM-GOLD-PARENT-OUT           ! queuing/shaping toward CE = OUTPUT
!
interface <G-R1 core-facing>
 service-policy input PM-G-R1-DISPOSITION-IN          ! EXP->DSCP disposition on the core-facing INGRESS
```

**Verification**
```
show policy-map interface GigabitEthernet0/0/0/1.200 output
! capture toward Gold CE: DSCP now correct per mode (Uniform=core-derived, Short-Pipe=customer-original).
! confirm SP-internal traffic-class/qos-group are NOT visible on the wire.
```

---

## CCIE Challenge Tasks

### Challenge A — Trust-boundary attack simulation
- From the Emerald CE, mark **all** traffic EF. Prove E-R1's ingress police + `class-default → set dscp default` re-mark protects the core LLQ (EXP 5). Then remove the re-mark and show the failure mode (BE flooding the priority queue end-to-end to G-R1).

### Challenge B — Tunneling-mode behavior across the path
- Configure **Short-Pipe** end-to-end. Force a core P router to re-color EXP (WRED/policer). Confirm at G-R1 that the **customer DSCP is untouched** while the **egress PHB is chosen from the customer DSCP** (not the tunnel EXP). Repeat in **Uniform** and show the DSCP now reflects the core change.

### Challenge C — End-to-end SLA validation
- Instrument EF with IP SLA / synthetic probes across `E-R1→G-R1`. Under a saturating BE load, confirm the voice class meets **<150 ms one-way latency, low jitter, ~0 loss**. Tie the measured result back to each hop's `show policy-map interface` counters (LLQ served, AF weighted, BE/WRED absorbing the congestion).
