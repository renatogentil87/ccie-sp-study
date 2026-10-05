# WB10 — QoS (MQC) Solutions on PEs R1 and R14

**Platform:** Cisco 7200, IOS 15.2(4)M11 — **IOS Classic syntax (MQC)**
**Scope:** DiffServ MQC on PE routers **R1 (AS 65100)** and **R14 (AS 65200)**
**Prerequisite:** WB00–WB07 complete (L3VPN forwarding, LDP/MPLS up on core-facing links)

---

## 1. Design intent

- **Ingress (PE-CE facing):** classify customer traffic by DSCP, mark into MPLS EXP as the packet enters the MPLS core, and police video.
- **Egress (core-facing):** schedule with LLQ for voice, guaranteed bandwidth for video, remaining bandwidth for data, and fair-queue + WRED for the rest.

| PE | PE-CE interface(s) (ingress) | Core-facing interface(s) (egress) |
|----|------------------------------|-----------------------------------|
| R1 | g1/0 (R9), f3/0 (R10), f4/1 (R32), g1/0… | f0/0 (→R3), f4/0 (→R2) |
| R14 | f3/0 (R25), f4/0 (R28 L2) | g2/0 (→R12), g1/0 (→R13) |

> On MPLS imposition, `set mpls experimental topmost` writes the EXP bits of the top (and, by default on IOS, imposed) label. Marking EXP at ingress is what lets P routers (R3/R4 etc.) do EXP-based PHB without seeing the customer DSCP.

---

## 2. Class maps (identical on R1 and R14)

```
class-map match-any VOICE
 match dscp ef
!
class-map match-any VIDEO
 match dscp af41
!
class-map match-any DATA
 match dscp af21
```

- `match-any` = logical OR of the match statements (here each has one, so behavior is identical to match-all, but match-any is the requested form and lets you add more DSCPs later, e.g. `match dscp cs5` for VOICE).

---

## 3. Policy maps

### 3.1 INGRESS (marking + policing) — applied on PE-CE interfaces

```
policy-map INGRESS
 class VOICE
  set mpls experimental topmost 5
 class VIDEO
  police cir 10000000
   conform-action transmit
   exceed-action drop
 class DATA
  set mpls experimental topmost 2
 class class-default
  set mpls experimental topmost 0
```

- `set mpls experimental topmost 5` — map EF voice to EXP 5 on label imposition.
- `police cir 10000000` — rate-limit video to 10 Mbps; conform transmit, exceed drop (single-rate). You can add a `set mpls experimental topmost 4` conform-action if you want to mark instead of only police.
- DATA/default EXP marking included so the whole DiffServ mapping is explicit (AF21→EXP2, BE→EXP0). Remove if you only want the two requested actions.

> The task asks specifically for `set mpls exp topmost 5` under VOICE and `police cir 10000000` under VIDEO — those are the two mandatory lines; the DATA/default EXP sets are good-practice additions.

### 3.2 EGRESS (scheduling) — applied on core-facing interfaces

```
policy-map EGRESS
 class VOICE
  priority 1000
 class VIDEO
  bandwidth 5000
 class DATA
  bandwidth remaining percent 50
 class class-default
  fair-queue
  random-detect
```

- `priority 1000` — LLQ: strict-priority queue with a 1000 kbps policer (voice is capped at 1 Mbps during congestion).
- `bandwidth 5000` — minimum guaranteed 5 Mbps for video.
- `bandwidth remaining percent 50` — DATA gets 50% of what's left after LLQ + guaranteed classes.
- `class-default` — `fair-queue` (WFQ among flows) + `random-detect` (WRED) for graceful TCP drop.

> Note: you cannot mix `bandwidth <kbps>` (VIDEO) and `bandwidth remaining percent` (DATA) in a way IOS rejects — IOS **does** allow absolute `bandwidth` and `bandwidth remaining percent` in the same policy-map. If your image complains, change VIDEO to `bandwidth percent 25` and DATA to `bandwidth remaining percent 50`.

---

## 4. Interface application

### 4.1 R1

```
! ---- Ingress on PE-CE interfaces ----
interface GigabitEthernet1/0
 description R1->R9 (CE Green, eBGP 65910)
 service-policy input INGRESS
!
interface FastEthernet3/0
 description R1->R10 (CE Green, eBGP 65910)
 service-policy input INGRESS
!
interface FastEthernet4/1
 description R1->R32 (CE Blue, eBGP 65025)
 service-policy input INGRESS
!
! ---- Egress on core-facing interfaces ----
interface FastEthernet0/0
 description R1->R3 (core)
 service-policy output EGRESS
!
interface FastEthernet4/0
 description R1->R2 (core)
 service-policy output EGRESS
```

### 4.2 R14

```
! ---- Ingress on PE-CE interfaces ----
interface FastEthernet3/0
 description R14->R25 (CE Blue, eBGP 65025)
 service-policy input INGRESS
!
! ---- Egress on core-facing interfaces ----
interface GigabitEthernet2/0
 description R14->R12 (core)
 service-policy output EGRESS
!
interface GigabitEthernet1/0
 description R14->R13 (core)
 service-policy output EGRESS
```

> On a Fast/Gig interface the LLQ/bandwidth values must fit the link rate. If `priority 1000` + `bandwidth 5000` are rejected on a 100M interface due to `max-reserved-bandwidth`, raise it: `interface … / max-reserved-bandwidth 90`.

---

## 5. Verification

```
show policy-map INGRESS
show policy-map EGRESS
show policy-map interface FastEthernet0/0            ! R1 core-facing egress
show policy-map interface GigabitEthernet1/0         ! R1 ingress (R9)
show policy-map interface GigabitEthernet2/0 output  ! R14 egress
show policy-map interface FastEthernet3/0 input      ! R14 ingress (R25)
```

What to look for:
- **Ingress:** class VOICE shows packets matched + `set mpls experimental topmost 5`; class VIDEO shows the policer with conform/exceed counters incrementing under load.
- **Egress:** class VOICE shows `Priority` with the 1000 kbps policer; VIDEO shows `bandwidth 5000 kbps`; DATA shows `bandwidth remaining 50%`; class-default shows WRED/`random-detect` drop stats and `fair-queue`.

Supporting:
```
show mls qos                      ! (platform dependent — 7200 uses sw queueing)
show queueing interface FastEthernet0/0
show interfaces FastEthernet0/0 | include rate|drops
```

---

## 6. Verifying EXP end-to-end (optional)

To confirm EXP is marked on imposition at R1 and honored across the core:

```
! On R1 after sending EF-marked traffic from R9:
show policy-map interface GigabitEthernet1/0 input | section VOICE
! On a P router (R3), classify on EXP to confirm it arrived marked:
class-map match-any CORE-VOICE
 match mpls experimental topmost 5
```

---

## 7. Gotchas

- `set mpls experimental topmost` only has effect where a label is **imposed/present**. On the PE-CE ingress interface the packet is still IP; IOS applies the EXP marking as the label is pushed toward the core — correct and intended here.
- LLQ `priority` without an explicit kbps/percent gives no policer; always bound it (`priority 1000`) to prevent voice starving other classes.
- WRED (`random-detect`) in `class-default` requires queueing; it coexists with `fair-queue` (per-flow WRED).
- Match on `dscp ef` requires the CE to mark EF; otherwise classify on access-list or re-mark at ingress.
