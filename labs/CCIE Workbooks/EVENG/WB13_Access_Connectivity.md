# CCIE SP Workbook 13 — Access Connectivity

**Domain:** 3 — Access Connectivity (10%)
**Platform:** EVE-NG (IOS-XRv 9000)
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** Emerald — E-R1, E-R2 + CEs. Access-layer concepts (L2 access, ERPS, MC-LAG, BNG).
**Format:** Question → Solution → Verification.

> **Note on scope:** IOS-XRv 9000 supports L2VPN/L2 access, VLAN rewrite, and MC-LAG/ICCP.
> G.8032 ERPS and full BNG (subscriber/CUPS) are **platform-limited on XRv** — treat those
> sections as **concept + design module preparation** with reference config where syntax applies.

---

## Section 1: Layer 2 Access

### Task 1 — 802.1Q VLAN tagging on a PE-CE link

**Question:**
CE1 connects to E-R1 on `GigabitEthernet0/0/0/1`. Customer traffic arrives tagged with VLAN 100.
Terminate VLAN 100 into an L2 service (bridge/xconnect) on E-R1 using an 802.1Q sub-interface.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2 — Q-in-Q (802.1ad double tagging) for SP access

**Question:**
Provide a wholesale access service on E-R1 `Gi0/0/0/2`. The customer sends single-tagged frames
(inner C-VLAN, e.g. any of 200-299); the SP adds an outer S-VLAN (S-Tag) of 500 to carry them
across the provider network. Configure the Q-in-Q access sub-interface.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3 — VLAN translation / rewrite on IOS-XR

**Question:**
CE arrives on E-R1 tagged VLAN 100, but the core service expects VLAN 900. Translate (rewrite)
the ingress VLAN 100 to 900 symmetrically so the return traffic maps back correctly.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4 — Sub-interface per VLAN (service demux)

**Question:**
CE1 trunks VLANs 10, 20, 30 to E-R1 on `Gi0/0/0/1`. Terminate each VLAN into its own L2 service
(one bridge-domain per VLAN) using one sub-interface per VLAN.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2: G.8032 ERPS (Ethernet Ring Protection Switching)

> ⚠️ **Platform note:** G.8032 is not natively configurable on IOS-XRv 9000. This section is
> **concept + design prep**. Config shown is IOS-XR reference syntax (ASR9000-class) for exam recall.

### Task 5 — ERPS concept, RPL, and RPL Owner

**Question:**
Explain G.8032 Ethernet Ring Protection: what problem it solves, and the roles of the RPL and
the RPL Owner in the ring E-R1–CE1–CE2–E-R2–(back to E-R1).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6 — Configure the ring (reference syntax)

**Question:**
Configure G.8032 on the ring E-R1–CE1–CE2–E-R2. Make E-R1 the RPL Owner, use control VLAN 4090,
and protect data VLANs 100-200.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 7 — Failure detection and recovery

**Question:**
The CE1–CE2 link fails. Describe detection, the R-APS signaling, and recovery. Then describe the
revertive behavior when the link is restored.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3: MC-LAG (Multi-Chassis Link Aggregation)

### Task 8 — MC-LAG concept and ICCP

**Question:**
CE2 is dual-homed to E-R1 and E-R2. Explain MC-LAG and the role of ICCP. Why does CE2 believe it is
connected to a single LACP peer?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 9 — Configure MC-LAG (CE2 → E-R1 + E-R2)

**Question:**
Configure MC-LAG so CE2 is dual-homed via `Bundle-Ether1` to E-R1 (primary) and E-R2 (backup),
using ICCP redundancy group 1, LACP System MAC `0000.0000.00cc`.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 10 — MC-LAG vs EVPN multi-homing

**Question:**
Compare MC-LAG (ICCP) with EVPN multi-homing (Type 1/4, ESI). When would you pick each?


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4: BNG Concepts (Design Module Prep)

> ⚠️ **Platform note:** Full BNG subscriber management is **not deployable on IOS-XRv 9000**.
> This section is **concept + Design module preparation**. Config fragments are IOS-XR reference.

### Task 11 — BNG overview: IPoE and PPPoE

**Question:**
What is a BNG (Broadband Network Gateway)? Contrast the IPoE and PPPoE access models.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 12 — Subscriber management, AAA (RADIUS), dynamic templates

**Question:**
How does the BNG apply per-subscriber policy and authenticate/account subscribers via RADIUS?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 13 — CUPS (Control/User Plane Separation) — Cloud Native BNG

**Question:**
Explain CUPS and how it enables Cloud Native BNG. What are the CP and UP roles?


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5: Troubleshooting

### Task 14 — Q-in-Q outer tag not preserved (missing rewrite rule)

**Question:**
A Q-in-Q access service on E-R1 `Gi0/0/0/2.500` was expected to carry the customer's inner C-Tag
across the core with the SP S-Tag 500 imposed. Customer reports the **outer S-Tag is missing** on
the far end (frames arrive single-tagged / mis-mapped). Diagnose and fix.

**Diagnosis:**
```
show ethernet tags interface GigabitEthernet0/0/0/2.500
show interfaces GigabitEthernet0/0/0/2.500 | include Rewrite
! Observed: Rewrite ingress = "pop 2" (or a pop that removes BOTH tags),
! so the outer S-Tag is stripped and never re-imposed on egress.
```
Root cause: the **rewrite rule is wrong/missing** — a `pop 2` (or plain `pop 1` without a matching
push on the core side) removes the S-Tag so it is not preserved across the core.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 15 — MC-LAG failover not working (ICCP session down)

**Question:**
You shut E-R1's CE2-facing link expecting E-R2 to take over `Bundle-Ether1`, but CE2 loses
connectivity — **failover does not occur**. Diagnose and fix.

**Diagnosis:**
```
show iccp group 1
  ! ICCP session state = NOT Connected (Down)  <-- root cause
show mpls ldp neighbor
  ! No LDP session E-R1 <-> E-R2 (ICCP transport is LDP-based)
show lacp mlacp
  ! E-R2 never promoted to Active because it received no ICCP state sync
```
Root cause chain: **ICCP session is DOWN** → PEs cannot synchronize mLACP state → E-R2 does not
know it must become Active → no failover. Common underlying causes:
- LDP session between E-R1/E-R2 not established (routing/loopback reachability, `mpls ldp neighbor`),
- wrong ICCP `member neighbor` IP,
- mismatched `mlacp system mac` / node IDs.


> *Try this yourself first. Solution available in `solutions/` folder.*

