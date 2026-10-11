# Troubleshooting Lab 8: QoS in SP Networks — 20 Tickets

**Platform:** GNS3 Local (Cisco 7200, IOS 15.2)
**Topology:** 20 routers — same physical topology
**Difficulty:** CCNP-SP to CCIE-SP progressive
**Prerequisite:** Golden-state snapshot + QoS policies applied (DiffServ model)

---

## Lab Context

Your SP network has a full DiffServ QoS model deployed. Classification happens at PE ingress (DSCP-based), core forwarding uses MPLS EXP bits (short-pipe model), and egress PEs reclassify back to DSCP for CE delivery. This lab tests QoS troubleshooting: misclassification, wrong queuing, EXP mapping failures, policer drops, and shaping issues.

---

## Lab Rules

- Do NOT change hostnames, enable passwords, or console/VTY configuration
- Do NOT change OSPF, LDP, MPLS, or BGP configurations
- Do NOT add new interfaces or IP addresses unless explicitly required
- Do NOT remove existing features to resolve a ticket — fix the root cause
- Static routes are NOT permitted unless preconfigured
- Points are awarded for finding AND resolving the fault
- The resolution of one ticket MAY depend on previous tickets
- There are NO physical faults (all interfaces are cabled correctly)
- Use password `cisco` for any authentication if needed

---

## Topology Reference

| Role | Routers |
|---|---|
| PE (ingress classification) | R2, R17 |
| PE (egress delivery) | R8, R18 |
| P (core EXP-based QoS) | R3, R4, R5, R6, R7, R13, R14, R15, R16 |
| CE (traffic sources) | R1 (VoIP+Data), R12 (Video), R19 (Bulk) |
| CE (traffic sinks) | R9, R11, R20 |

**QoS Model (6 classes):**
| Class | DSCP | MPLS EXP | Treatment |
|---|---|---|---|
| Voice | EF (46) | 5 | Strict Priority, 20% BW guarantee |
| Video | AF41 (34) | 4 | CBWFQ, 30% BW |
| Critical Data | AF21 (18) | 3 | CBWFQ, 20% BW |
| Bulk Data | AF11 (10) | 2 | CBWFQ, 10% BW |
| Network Control | CS6 (48) | 6 | CBWFQ, 5% BW |
| Best Effort | 0 | 0 | Default WFQ, remaining BW |

**QoS Architecture:**
- PE ingress: `service-policy input CLASSIFY` (matches DSCP, sets EXP on MPLS imposition)
- Core P: `service-policy output CORE-QOS` (queues based on EXP)
- PE egress: `service-policy output EDGE-DELIVERY` (queues based on DSCP, shapes to CE rate)

**Pipe model:** Short-pipe (egress PE uses inner DSCP, not EXP)
**CE link speed:** FastEthernet (100 Mbps simulated as 10 Mbps shaped)

---

## Ticket 1

Voice traffic (DSCP EF) from R1 enters R2 but is NOT being placed into the priority queue on R2's core-facing interface (Gi1/0 toward R3). All traffic exits in the default queue. The class-map exists but the policy is not applied.

Fix the network so that EF traffic enters the priority queue on R2's egress.

Verify: `show policy-map interface GigabitEthernet1/0 output` on R2 shows packets matching the EF/Voice class.

Score: 2 Points

---

## Ticket 2

The ingress classification policy on R2's CE-facing interface (Fa0/0 from R1) is applied but matching ZERO packets. All traffic from R1 hits the class-default. The class-map should match DSCP EF but R1's traffic arrives with DSCP EF marked.

Fix the network so that ingress classification correctly matches R1's EF traffic.

Verify: `show policy-map interface FastEthernet0/0 input` on R2 shows incrementing match count for Voice class.

Score: 2 Points

---

## Ticket 3

MPLS EXP marking: R2 is classifying correctly (Voice class matched) but the MPLS EXP bits are NOT being set on labeled packets. Packets leave R2 with EXP 0 regardless of DSCP. Core P routers queue everything as best-effort.

Fix the network so that MPLS EXP bits are set correctly at PE imposition.

Verify: On R3, `show policy-map interface <ingress-from-R2> input` shows packets arriving in different EXP classes (not all EXP 0).

Score: 2 Points

---

## Ticket 4

Core queuing on R5: The output policy on R5's interface toward R8 (Gi2/0) is correct, but voice packets (EXP 5) are being POLICED (rate-limited) instead of being placed in strict priority. The policer is dropping 40% of voice traffic during congestion.

Fix the network so that voice traffic gets strict priority queuing (not policing) in the core.

Verify: `show policy-map interface GigabitEthernet2/0 output` on R5 shows Voice class with `priority` or `priority percent` (no exceed-action drop during normal load).

Score: 3 Points

---

## Ticket 5

Video traffic (AF41) on R2 is being reclassified to best-effort. R12 sends video at DSCP AF41 (34), but by the time it reaches the core, it's EXP 0. The ingress policy matches AF41 but the `set` action is wrong.

Fix the network so that AF41 traffic is mapped to EXP 4 in the core.

Verify: On a core router, `show policy-map interface <output>` shows AF41/Video class incrementing. Not in default queue.

Score: 3 Points

---

## Ticket 6

Egress PE (R8) delivery to R9: The short-pipe model means R8 should use the INNER DSCP (preserved from PE ingress) for egress queuing. Instead, R8 is using the EXP bits from the outer label (which are stripped at PHP on R7). Traffic exits R8 all in default queue.

Fix the network so that R8 uses the inner DSCP for egress classification toward R9.

Verify: `show policy-map interface GigabitEthernet1/0 output` on R8 (toward R9) shows Voice/Video/Data classes matching correctly based on IP DSCP.

Score: 3 Points

---

## Ticket 7

Policer on R2's CE-facing ingress: Customer_A (R1) is rate-limited to 50 Mbps. Traffic within the contract should pass unmarked, traffic exceeding should be marked DOWN (EF→AF11). But the policer is dropping ALL excess instead of marking down.

Fix the network so that the policer marks-down instead of dropping excess traffic.

Verify: `show policy-map interface Fa0/0 input` on R2 shows conform-action transmit, exceed-action set-dscp-transmit 10. No drops on in-contract traffic.

Score: 2 Points

---

## Ticket 8

WRED on core router R4: Best-effort class should use WRED for congestion avoidance. But WRED is not configured — tail-drop is occurring instead. Bursty TCP traffic from R12 experiences synchronized drops causing throughput collapse.

Fix the network so that WRED is active for the best-effort class on R4's egress.

Verify: `show policy-map interface <output> output` on R4 shows WRED parameters for default/best-effort class. `show policy-map interface <output>` shows WRED drops (not tail-drops) during congestion.

Score: 3 Points

---

## Ticket 9

Traffic shaping on R8 toward R9: R8 should shape Customer_A traffic to 10 Mbps (CE link speed). The shaper is configured but set to 1 Mbps, causing massive packet drops and latency. Voice quality is terrible.

Fix the network so that the shaper rate matches the CE link capacity (10 Mbps or no shaping).

Verify: `show policy-map interface Gi1/0 output` on R8 shows shape rate ≥ 10000000 bps. Voice latency drops below 150ms.

Score: 2 Points

---

## Ticket 10

Class-map matching priority: R2 has overlapping class-maps. A `match dscp ef` class AND a `match access-group 101` class both could match voice traffic. The ACL class is evaluated first (by order in policy-map) and incorrectly captures voice into the wrong queue.

Fix the network so that DSCP EF traffic always matches the Voice class, regardless of ACL overlap.

Verify: Voice traffic matches the priority queue class, not the ACL-based class. `show policy-map interface` confirms zero EF packets in wrong class.

Score: 2 Points

---

## Ticket 11

EXP-to-DSCP mapping on egress: When R8 receives labeled traffic and removes the label (PHP happened at R7), the original DSCP is preserved (short-pipe). But someone configured `policy-map type qos` on R8 that REWRITES DSCP to 0 on all traffic. Downstream R9 sees all traffic as best-effort.

Fix the network so that original DSCP values are preserved through to R9.

Verify: Capture on R9 shows voice traffic arrives with DSCP EF (46), video with AF41 (34), etc.

Score: 3 Points

---

## Ticket 12

Hierarchical QoS (HQoS): R2's CE-facing policy should be hierarchical — outer policy shapes to line-rate, inner policy provides per-class queuing within the shaped rate. The inner policy is not being evaluated (flat shape only, no per-class queuing within).

Fix the network so that HQoS provides per-class queuing within the shaped aggregate.

Verify: `show policy-map interface Fa0/0 output` shows parent (shape) and child (class-based queuing) policies both active with matching packets.

Score: 3 Points

---

## Ticket 13

Network Control (CS6) traffic: OSPF hello packets and LDP keepalives between R5 and R8 are being DROPPED during congestion because they're not in the CS6/Network-Control class. If these drop, adjacencies flap under load.

Fix the network so that locally-generated control-plane traffic is marked CS6 and queued appropriately.

Verify: During simulated congestion, `show ip ospf neighbor` stays FULL. `show policy-map interface` shows CS6/Network-Control class incrementing.

Score: 4 Points

---

## Ticket 14

Per-VRF QoS: Customer_A and Customer_B share the same PE egress interface (R2→R3). Customer_A should get 70% of bandwidth, Customer_B 30%. But both get equal (50/50) treatment. No per-VRF differentiation exists in the output policy.

Fix the network so that per-VRF bandwidth allocation is enforced on the shared PE egress.

Verify: Under congestion, Customer_A traffic gets ~70% throughput and Customer_B ~30%. `show policy-map interface` shows VRF-aware class matching.

Score: 4 Points

---

## Ticket 15

Tunnel-based QoS: TE Tunnel0 (R2→R8) should have its own QoS policy applied to the tunnel interface. Traffic entering the tunnel should be queued per-class on the tunnel, not the physical interface. The physical interface policy is overriding.

Fix the network so that QoS is applied at the tunnel level for tunnel-carried traffic.

Verify: `show policy-map interface Tunnel0 output` shows per-class queuing active with matching packets.

Score: 4 Points

---

## Ticket 16

Policer color-aware mode: R2's ingress policer should be 2-rate 3-color (conform=green, exceed=yellow, violate=red). But it's running as single-rate, so burst-tolerant video traffic is being marked red (dropped) when it should be yellow (marked down).

Fix the network so that 2-rate 3-color policing handles video bursts as yellow (not red).

Verify: `show policy-map interface Fa0/0 input` shows conform-rate and exceed-rate separately configured. Video class shows exceed-action mark-down (not drop).

Score: 4 Points

---

## Ticket 17

QoS Pre-classify: VPN traffic entering TE Tunnel0 on R2 is classified AFTER MPLS encapsulation (outer header), not based on the inner IP DSCP. The tunnel is stripping QoS visibility. `qos pre-classify` should fix this but isn't working.

Fix the network so that QoS classification on the tunnel uses the inner IP header's DSCP.

Verify: `show policy-map interface Tunnel0 output` shows traffic matching Voice/Video classes (inner DSCP), not all in default.

Score: 4 Points

---

## Ticket 18

Complete QoS blackout: No QoS policies are active ANYWHERE in the network. All interfaces show "no service-policy applied." Core traffic is all best-effort. The configs still have the policy-maps and class-maps, but `service-policy` commands have been removed from all interfaces.

Fix the network so that QoS policies are re-applied on all PE and P router interfaces.

Verify: `show policy-map interface` on R2, R5, R8 all show active policies with matching packets. Voice gets priority, video gets guaranteed BW.

Score: 5 Points

---

## Ticket 19

EXP remarking in core causing cascade failure: One P router (R4) is REMARKING all EXP values to 0 on its egress. This destroys QoS classification for all downstream routers. Voice becomes best-effort from R4 onwards.

Fix the network so that core P routers preserve EXP values (no remarking in transit).

Verify: On R5 (downstream of R4), `show policy-map interface <input>` shows traffic arriving with correct EXP distribution (EXP 5, 4, 3, 2, 0). Not all EXP 0.

Score: 5 Points

---

## Ticket 20

End-to-end QoS validation: Voice traffic from R1 to R9 experiences >300ms latency and 10% loss during network congestion. The issue spans multiple routers — classification, EXP marking, core queuing, and egress delivery all have small errors that compound.

Fix ALL QoS issues so that voice traffic meets SLA (<150ms latency, <1% loss, <30ms jitter).

Verify: Voice traffic (EF) from R1 to R9 shows <150ms latency. `show policy-map interface` on ALL routers in path shows voice in priority queue with minimal drops.

Score: 5 Points

---

## Scoring Summary

| Tickets | Difficulty | Points Each | Total |
|---|---|---|---|
| 1-3 | CCNP-SP (⭐⭐) | 2 | 6 |
| 4-6 | CCNP-SP (⭐⭐⭐) | 3 | 9 |
| 7-9 | CCNP-SP (⭐⭐⭐) | 2-3 | 7 |
| 10-12 | CCNP→CCIE (⭐⭐⭐) | 2-3 | 8 |
| 13-17 | CCIE-SP (⭐⭐⭐⭐) | 4 | 20 |
| 18-20 | CCIE-SP (⭐⭐⭐⭐⭐) | 5 | 15 |
| **Total** | | | **65 Points** |

**Passing:** 49/65 (75%)
**CCIE-ready:** 59/65 (90%)

---

## Injection Notes (for AI fault injector)

**Base state:** Golden-state + full DiffServ QoS policies applied on all interfaces

| Ticket | Router(s) | Fault |
|---|---|---|
| 1 | R2 | `no service-policy output` on Gi1/0 |
| 2 | R2 | Class-map matches `dscp ef` but traffic arrives with `dscp 46` (numeric vs keyword) |
| 3 | R2 | `set mpls experimental topmost 0` in policy-map (wrong value) |
| 4 | R5 | `police` instead of `priority` for voice class |
| 5 | R2 | `set mpls experimental topmost 0` for video class (wrong mapping) |
| 6 | R8 | Egress policy uses `match mpls experimental topmost` instead of `match dscp` |
| 7 | R2 | `police exceed-action drop` instead of `exceed-action set-dscp-transmit` |
| 8 | R4 | Missing `random-detect` in default class |
| 9 | R8 | `shape average 1000000` (1Mbps instead of 10Mbps) |
| 10 | R2 | ACL class-map listed before DSCP class in policy-map order |
| 11 | R8 | `set dscp default` in egress policy rewriting all traffic |
| 12 | R2 | Missing `service-policy <child>` under parent shaper class |
| 13 | R5 | No `ip dscp cs6` on control-plane or missing class for locally-originated |
| 14 | R2 | No VRF-aware match in output policy (no `match vrf`) |
| 15 | R2 | QoS on physical interface, not on Tunnel0 |
| 16 | R2 | Single-rate policer instead of two-rate |
| 17 | R2 | Missing `qos pre-classify` on Tunnel0 |
| 18 | All | `no service-policy` on all interfaces |
| 19 | R4 | `set mpls experimental topmost 0` on all transit traffic |
| 20 | Multiple | Multiple small errors across path (marking + queuing + shaping) |
