# LAB 4 — Full CCIE SP Practice Lab (EVE-NG / IOS-XR)

**Platform:** EVE-NG, IOS-XRv 9000 (recommend 7.x+ for SRv6/Flex-Algo/EVPN feature parity)
**Scope:** Full-scenario lab covering IS-IS/SR, L2VPN/EVPN/L3VPN/QoS, SoO/troubleshooting, HA/FRR, BGP security, and automation (NSO/Ansible/telemetry) — matching the CCIE SP v3 blueprint.
**Relationship to LAB 3:** Same physical topology and addressing plan as LAB 3. Tasks below are DIFFERENT — do not reuse LAB 3 configuration as the answer key. Where LAB 3 used a given technique, this lab deliberately requires the alternate technique (noted in each section).
**Format:** Tasks only. No solutions, no expected-output answer keys. Verification commands are listed so you can self-check.

---

## 0. Topology and Addressing (reference — build once, reuse for all sections)

```
                        RR1 (L2)                    RR2 (L2)
                      10.0.0.11/32                10.0.0.12/32
                           \                          /
                            \________________________/
                            /          |             \
                     P1(L1/L2)   P2(L1/L2)      P3(L1/L2)
                   10.0.0.21/32  10.0.0.22/32   10.0.0.23/32
                      /    \         |              /    \
                    PE1    PE2      PE3           PE4   (ASBR-A/ASBR-B for Inter-AS)
                 10.0.0.1  10.0.0.2 10.0.0.3     10.0.0.4
                  (L1)      (L1)     (L1)         (L1)
                   |          |        |            |
                  CE1        CE2      CE3          CE4
```

| Router | Role | Loopback0 | IS-IS Level | AS |
|---|---|---|---|---|
| PE1 | PE | 10.0.0.1/32 | L1 | 65000 |
| PE2 | PE | 10.0.0.2/32 | L1 | 65000 |
| PE3 | PE | 10.0.0.3/32 | L1 | 65000 |
| PE4 | PE | 10.0.0.4/32 | L1 | 65000 |
| P1 | P | 10.0.0.21/32 | L1/L2 | 65000 |
| P2 | P | 10.0.0.22/32 | L1/L2 | 65000 |
| P3 | P | 10.0.0.23/32 | L1/L2 | 65000 |
| RR1 | RR (BGP) | 10.0.0.11/32 | L2 | 65000 |
| RR2 | RR (BGP) | 10.0.0.12/32 | L2 | 65000 |
| ASBR-A | ASBR | 10.0.0.31/32 | L2 | 65000 |
| ASBR-B | ASBR (remote AS) | 10.0.0.32/32 | L2 | 65100 |
| CE1-CE4 | CE | 192.168.x.x | — | 65511-65514 |

- IGP: IS-IS, multi-level (PEs are L1-only "stub" sites; P1/P2/P3 are L1/L2 Area Border Routers; RR1/RR2/ASBR-A are L2-only backbone).
- IS-IS area: PEs + attached P routers = Area 49.0001 (L1 domain); P1/P2/P3/RR1/RR2/ASBR-A = L2 backbone.
- MPLS/SR: Segment Routing MPLS data plane for the IPv4 core; SRv6 overlay built in parallel on a second loopback/locator set (Section 1).
- Management/mgmt0 addressing and VLAN/bridge wiring: build per your existing EVE-NG lab file; reuse LAB 3's physical `.unl` topology untouched.

Keep this section open in a second pane — every section below references these hostnames/addresses.

---

## Section 1 — IGP, Segment Routing, SRv6, and Traffic Engineering

### Task 1.1 — IS-IS Multi-Level Design

1. Configure IS-IS instance `CORE` in `address-family ipv4 unicast` on all routers.
2. Configure PE1-PE4 as **Level-1-only** routers attached to Area 49.0001.
3. Configure P1, P2, P3 as **Level-1-2** routers — they are the L1/L2 border for Area 49.0001 into the L2 backbone.
4. Configure RR1, RR2, and ASBR-A as **Level-2-only** backbone routers (no L1 adjacencies).
5. Set the IS-IS NET for every router using the pattern `49.0001.0100.0000.00XX.00` where `XX` is the router's last loopback octet (zero-padded).
6. On the L1-only PEs, verify that only an L1 adjacency forms toward the attached P router, and that no L2 adjacency is attempted.
7. Enable wide metrics (`metric-style wide`) on every router, and set the reference bandwidth so a 10 Gbps core link costs 10 and a 1 Gbps PE-uplink costs 100.
8. Verify L1→L2 route leaking: confirm PE loopbacks appear in the L2 backbone's IS-IS database as L1 routes redistributed by P1/P2/P3, and confirm attached-bit / ATT bit behavior from the L1 PEs' perspective.

**Verify:** `show isis neighbors`, `show isis database verbose`, `show isis adjacency detail`, `show route isis`

### Task 1.2 — Segment Routing MPLS — New Prefix-SID Scheme

> Different from LAB 3: use a **range-based, role-partitioned SRGB/prefix-SID scheme** instead of LAB 3's scheme.

1. Configure a global SRGB of `16000-23999` on all routers.
2. Assign prefix-SIDs using this partition (do NOT reuse LAB 3's numbering):
   - PEs: index = `100 + last loopback octet` (e.g., PE1 → index 101 → absolute SID 16101)
   - P routers: index = `200 + last loopback octet`
   - RRs/ASBRs: index = `300 + last loopback octet`
3. Enable `segment-routing mpls` under `router isis CORE` / `address-family ipv4 unicast`.
4. Set the `N-flag` (node-SID) explicitly on all loopback prefix-SIDs; do not use `explicit-null`.
5. Advertise an **anycast-SID** of `16050` shared between P1 and P2 only (not P3), representing a redundant anycast exit; confirm ECMP load-balancing across P1/P2 for traffic destined to the anycast SID.
6. Verify the SRGB label range is consistent on every node and that no label collisions occur (check against the LDP label range if LDP is still enabled anywhere — it should NOT be, SR is the sole MPLS control plane in this lab).

**Verify:** `show isis segment-routing label table`, `show mpls label table`, `show route ipv4 label 16101 detail`

### Task 1.3 — Flex-Algo with Include-All (not exclude-any)

> Different from LAB 3: build the Flex-Algo definition using **affinity include-all** semantics instead of exclude-any.

1. Define `segment-routing traffic-eng affinity-map` with two bit names: `LOW-LATENCY` (bit 0) and `HIGH-CAPACITY` (bit 1).
2. Tag the P1-P2 and P2-P3 core links with affinity `LOW-LATENCY`; tag the P1-RR1 and P3-ASBR-A links with affinity `HIGH-CAPACITY`.
3. Define Flex-Algo 128 as **IGP metric**, and constrain it so it will only use links that carry **both** `LOW-LATENCY` and `HIGH-CAPACITY` (i.e., `include-all` with both bits set) — this should visibly restrict the topology to a strict subset of links, unlike an exclude-any policy which merely prunes a smaller set.
4. Advertise Flex-Algo 128 definition from P1 (the metric-definition advertising router) and have all L2 routers (P1-P3, RR1, RR2, ASBR-A) participate in algorithm 128.
5. Allocate a Flex-Algo-specific prefix-SID (algorithm 128) on each participating router's loopback, using index = `128000 + last loopback octet` as the absolute SID (define a second SRGB range `24000-27999` dedicated to Flex-Algo 128 if your platform requires a disjoint range — document which approach you used).
6. Confirm via the IS-IS database that routers NOT satisfying the `include-all` constraint (i.e., links missing one of the two required affinity bits) are excluded from the Flex-Algo 128 SPT, and that this produces a different (likely sparser or disconnected-looking) topology than an exclude-any policy would with the same tags.

**Verify:** `show isis fast-reroute summary`, `show segment-routing traffic-eng ipv4 topology algorithm 128`, `show route algorithm 128`

### Task 1.4 — SR-TE with Explicit SID-Lists and Adjacency-SIDs (not dynamic)

> Different from LAB 3: build the TE path using an **explicit candidate-path with a manually encoded SID-list of adjacency-SIDs**, instead of a dynamic (PCE-computed/CSPF-computed) path.

1. Enable adjacency-SID advertisement on every core-facing interface of P1, P2, P3, RR1, RR2 (do not suppress adj-SID advertisement anywhere in the L2 backbone).
2. Record the adjacency-SID values advertised by each router for each of its L2-backbone links using the appropriate show command — you will need these to hand-build the SID-list.
3. Build an SR-TE policy on PE1 named `PE1-TO-PE4-EXPLICIT` with:
   - Color 100, endpoint PE4's loopback.
   - A single candidate-path with **explicit** segment-list type, preference 200.
   - The segment-list must be composed entirely of **adjacency-SIDs** (not node-SIDs / prefix-SIDs) tracing the specific path PE1 → P1 → P2 → P3 → PE4, forcing strict hop-by-hop adjacency steering.
4. Configure a second, lower-preference (100) candidate-path on the same policy using a **different** explicit adjacency-SID list that fails over through RR1 instead of P2, for manual-failover testing.
5. Steer traffic from PE1 to PE4's VRF prefixes into this policy using an explicit `color 100` BGP color extended-community match (not autoroute).
6. Validate that if a single link in the primary adjacency-SID list goes down, the policy falls back to the lower-preference candidate-path rather than silently blackholing (since explicit paths with adjacency-SIDs do not automatically re-route around failures the way a dynamic/CSPF path would).

**Verify:** `show segment-routing traffic-eng policy name PE1-TO-PE4-EXPLICIT detail`, `show segment-routing traffic-eng forwarding policy name PE1-TO-PE4-EXPLICIT`, `show isis database verbose | include Adjacency-SID`

### Task 1.5 — SRv6 with New Locator Assignments

> Different from LAB 3: use a **different per-role locator block scheme** than LAB 3.

1. Enable `ipv6 unicast` + `segment-routing srv6` on P1, P2, P3, RR1, RR2, ASBR-A, and PE1-PE4.
2. Assign SRv6 locators from the block `fcff:0:0::/48`, partitioned as:
   - PEs: `fcff:0:0:1XX::/64` where `XX` = last loopback octet (e.g., PE1 → `fcff:0:0:101::/64`)
   - P routers: `fcff:0:0:2XX::/64`
   - RRs/ASBRs: `fcff:0:0:3XX::/64`
3. Configure each router's `locator` under `segment-routing srv6`, with the `micro-SID`/End SID allocated as `::1` inside its locator block (document exact behavior/suffix per your IOS-XR version).
4. Enable SRv6-based IS-IS (`address-family ipv6 unicast` with `segment-routing srv6 locator <name>`) so the IS-IS SPF computes native SRv6 reachability.
5. Validate End.X SIDs are generated per-adjacency on P1-P3/RR1-RR2's core-facing links and appear correctly scoped within each router's own locator block.
6. Confirm dual-stack coexistence: SR-MPLS (Task 1.2-1.4) and SRv6 (this task) operate concurrently on the same physical topology without label/SID collision, since they use entirely separate data planes.

**Verify:** `show segment-routing srv6 locator detail`, `show segment-routing srv6 sid`, `show isis segment-routing srv6 all`

### Task 1.6 — SR-ODN with ECMP Margin

> Different from LAB 3: add an **ECMP margin** parameter to the On-Demand Next-hop policy template.

1. Configure an SR-TE "ODN template" (`on-demand color 100`) on PE1-PE4 referencing a dynamic path with metric-type TE and the Flex-Algo 128 constraint from Task 1.3.
2. Within the dynamic path-option, configure a **ECMP margin** of 20 (interpret per your platform's ODN/PCC syntax — this instructs the controller/head-end to consider parallel paths within 20% of the best metric as equal-cost candidates for load-balancing, rather than only installing the single best path).
3. Trigger ODN instantiation by advertising a VPNv4 prefix from PE4 tagged with color extended-community 100, consumed by PE1.
4. Confirm that the auto-instantiated SR-TE policy on PE1 installs **multiple** SID-lists (ECMP) when more than one path falls within the 20% margin of the best metric, rather than a single best path.
5. Explain (in your lab notes) the operational difference between ECMP margin and strict equal-cost-only ECMP, and when a margin is useful in a Flex-Algo-constrained topology with asymmetric metrics.

**Verify:** `show segment-routing traffic-eng policy color 100 detail`, `show segment-routing traffic-eng odn-template`, `show cef vrf <name> <prefix> detail`

---

## Section 2 — Services: EVPN, Inter-AS, QoS, L3VPN

### Task 2.1 — EVPN — New EVI Numbers and AC IDs

> Different from LAB 3: use different EVI numbers and Attachment-Circuit (AC) IDs.

1. Configure EVPN VPWS between PE2 and PE3 using **EVI 4001**, with local/remote AC-ID pair `4001001`/`4001002`.
2. Configure EVPN ELAN (multipoint) spanning PE1, PE2, PE4 using **EVI 4002**, Ethernet-Segment Identifier (ESI) `0000.0000.0000.4002.02` for the PE1/PE2 multi-homed CE1-equivalent access.
3. Configure the EVPN control-plane over the existing MP-BGP sessions (reuse the VPNv4 RR infrastructure for the `l2vpn evpn` address-family — do not stand up a separate RR for EVPN).
4. Set `single-active` multi-homing mode on the PE1/PE2 multi-homed access (not all-active) — verify only one of the two PEs forwards traffic for the attached CE at a time.
5. Verify EVPN Type-1 (Ethernet AD), Type-2 (MAC/IP), and Type-4 (ES) routes are correctly generated and installed for EVI 4002.

**Verify:** `show evpn evi detail`, `show evpn ethernet-segment detail`, `show bgp l2vpn evpn`

### Task 2.2 — Inter-AS Option C (not Option B)

> Different from LAB 3: implement **Inter-AS Option C** — BGP-LU (labeled unicast) between ASBRs + a direct multihop VPNv4 session RR-to-RR, instead of Option B's ASBR-to-ASBR VPNv4 re-origination.

1. Configure ASBR-A (AS 65000) and ASBR-B (AS 65100) with an eBGP IPv4-labeled-unicast (`address-family ipv4 labeled-unicast`) session over their directly connected link, exchanging only loopback /32s with MPLS labels — no VPNv4 routes cross this session.
2. Redistribute/advertise ASBR-A's and ASBR-B's own loopbacks plus their respective RR loopbacks into BGP-LU so that an LSP exists end-to-end from RR1 (AS 65000) to the remote AS's RR (AS 65100).
3. Establish a **multihop eBGP VPNv4** session directly between RR1 (or RR2) in AS 65000 and the remote RR in AS 65100 — this session rides over the BGP-LU-signaled LSP through the ASBRs, which act as pure LSRs for this traffic (next-hop-unchanged, do not re-originate VPNv4 NLRI at the ASBRs).
4. On the multihop VPNv4 session, configure `next-hop-unchanged` so PE-to-PE labeled traffic forwards end-to-end without the ASBRs having VRF awareness.
5. Confirm end-to-end LSP continuity with an MPLS ping/traceroute from a PE in AS 65000 to a PE in AS 65100, and confirm the ASBRs show zero VPNv4 routes in their BGP table (proving they are not Option B ASBRs).

**Verify:** `show bgp ipv4 labeled-unicast`, `show mpls forwarding`, `show bgp vpnv4 unicast summary`, `ping mpls ipv4 <remote-PE-loopback>`

### Task 2.3 — QoS — Ingress Marking + Core EXP Treatment

> Different from LAB 3: build an **ingress-marking + core EXP (traffic-class/EXP bits) treatment** model instead of LAB 3's policy.

1. On PE access-facing interfaces (PE-CE links), build an ingress MQC policy-map that classifies customer traffic by DSCP (EF→voice, AF41→video, default→best-effort) and **re-marks** the MPLS EXP/traffic-class bits of the imposed label stack to EXP 5 (voice), EXP 3 (video), EXP 0 (best-effort) at label imposition.
2. On every P router's core-facing interfaces, build a core-only MQC policy-map that classifies **purely on MPLS EXP bits** (no DSCP inspection in the core) into three queues matching the EXP markings from step 1, with LLQ for EXP 5, a minimum-bandwidth guaranteed queue for EXP 3, and class-default WRED for EXP 0.
3. Apply the ingress marking policy on PE1-PE4's CE-facing interfaces (`service-policy input`), and the EXP-based core policy on P1-P3's and RR1-RR2's inter-router interfaces (`service-policy output`).
4. Confirm that core routers never need to inspect the original IP DSCP (since the lab constraints state the core treats EXP only) — verify this by checking the core policy-map's class-maps match only on `mpls experimental topmost`.
5. Generate test traffic from CE1 marked as EF and AF41, and confirm correct EXP remarking at PE1 ingress and correct EXP-based queuing behavior at P1/P2 in the core via interface queue counters.

**Verify:** `show policy-map interface <if> input`, `show policy-map interface <if> output`, `show mpls forwarding detail`

### Task 2.4 — L3VPN — Redistribute Connected (not network statements)

> Different from LAB 3: use `redistribute connected` into the PE-CE VRF BGP/IGP process instead of explicit `network` statements.

1. Configure VRF `CUST-A` on PE1 and PE3 with appropriate RD/RT.
2. On the CE-facing interface of PE1 and PE3 inside VRF `CUST-A`, instead of using `network x.x.x.x/x` statements under the VRF BGP address-family, configure `redistribute connected` under `router bgp 65000 / vrf CUST-A / address-family ipv4 unicast` to inject the directly connected CE-facing subnet into VPNv4.
3. Apply a route-policy on the redistribution to filter out any connected interfaces that should NOT be advertised (e.g., exclude a management-only connected subnet in the VRF, if present) — demonstrate that `redistribute connected` requires explicit filtering discipline that `network` statements do not.
4. Verify the connected CE subnet appears in VPNv4 at the RR and at the remote PE's VRF table with the correct RT.
5. Document (in lab notes) the operational risk of `redistribute connected` in a VRF context — specifically that any new connected interface placed into the VRF is automatically advertised unless the route-policy is updated, unlike `network` statements which require an explicit per-prefix addition.

**Verify:** `show bgp vrf CUST-A ipv4 unicast`, `show route vrf CUST-A`, `show route-policy <name>`

---

## Section 3 — SoO and Troubleshooting MCQs

### Task 3.1 — Site-of-Origin Scenario (new scenario — not LAB 3's)

1. Customer "Nimbus" is multi-homed to PE2 and PE4 from two different CE sites that must never see each other's routes re-advertised back to them via the VPN backbone (classic SoO loop-prevention requirement), AND Nimbus also has an OSPF PE-CE sham-link requirement between the same two sites for a backdoor-link scenario.
2. Configure the SoO extended-community `65000:4002` on PE2's VRF `NIMBUS` CE-facing BGP neighbor (or OSPF redistribution point, as applicable) and the SoO extended-community `65000:4004` on PE4's equivalent.
3. Ensure that routes learned from the PE2 site carrying SoO `65000:4002` are never re-advertised back out to any CE BGP/OSPF neighbor tagged with the same SoO value, while still allowing those routes to reach PE4's Nimbus site normally.
4. Build the OSPF sham-link between PE2 and PE4's VRF loopbacks for the backdoor-link scenario, and confirm the sham-link does not interact incorrectly with the SoO filtering (sham-link endpoints must be reachable via BGP, not OSPF, inside the VRF).
5. Validate no routing loop occurs if the backdoor link between the two Nimbus CE sites comes up at the same time as both PE uplinks.

**Verify:** `show bgp vrf NIMBUS ipv4 unicast <prefix> detail` (check extcommunity), `show ospf vrf NIMBUS sham-links`, `show route vrf NIMBUS`

### Task 3.2 — Troubleshooting MCQs (answer in your lab notes — no solutions provided here)

1. (G.8032 ERP) In a ring running G.8032 Ethernet Ring Protection with RPL (Ring Protection Link) owner configured on the wrong node, what is the FIRST symptom you should expect to observe on the ring, and what single command confirms the RPL owner/neighbor role on each ring port?
2. (G.8032 ERP) A ring segment reports "Signal Fail" (R-APS SF) on one port but the ring does NOT switch traffic away from the failed link. List the three most likely root causes you would check, in the order you would check them.
3. (G.8032 ERP) After a manual switch (MS) is applied to force traffic off the RPL, what command reverts the ring to its normal (idle) state, and what is the risk of forgetting to issue it?
4. (BGP NSR) A route-processor failover occurs on an RR configured for BGP NSR (Non-Stop Routing), and a client PE briefly logs `%BGP-5-NBR_RESET` instead of maintaining the session. Identify the two most likely configuration gaps that would cause NSR to fail to preserve the session across RP failover.
5. (BGP NSR) Explain the difference in log signature between a BGP session that successfully rode out an RP failover under NSR versus one that fell back to Graceful Restart (GR) — specifically, what log messages or counters distinguish the two outcomes?
6. (BGP NSR) What show command(s) would you use immediately after a simulated RP failover to confirm NSR (not GR) was the mechanism that preserved a given BGP session, and what specific field proves it?

---

## Section 4 — High Availability and Convergence

### Task 4.1 — SRv6 FRR with Priority-Limit Critical

> Different from LAB 3: configure **priority-limit critical** for SRv6 TI-LFA FRR instead of LAB 3's FRR priority model.

1. Enable TI-LFA for SRv6 under `segment-routing srv6` / IS-IS `fast-reroute per-prefix` on P1, P2, P3, RR1, RR2.
2. Configure `fast-reroute priority-limit critical` so that **only** prefixes explicitly tagged with RSVP/FRR priority `critical` receive a pre-computed TI-LFA backup repair path; all other prefixes (default priority) must NOT receive FRR protection in this task.
3. Tag the VPNv4 service loopbacks of PE1 and PE4 (used for Section 2's inter-AS LSP) with priority `critical` using the appropriate IS-IS prefix priority/tag mechanism, while leaving PE2 and PE3's loopbacks at default priority.
4. Validate via a simulated link failure (shut a P1-P2 core link) that PE1/PE4 traffic is protected sub-50ms via the pre-computed SRv6 repair SID, while confirming PE2/PE3 traffic — NOT tagged critical — has no pre-computed backup and must wait for full IS-IS reconvergence.
5. Explain in lab notes why `priority-limit critical` is used in real deployments to conserve FRR computation/FIB resources on routers with large prefix tables, rather than computing TI-LFA backups for every prefix indiscriminately.

**Verify:** `show isis fast-reroute 10.0.0.1/32 detail`, `show segment-routing srv6 sid detail`, `show isis fast-reroute summary`

### Task 4.2 — OSPF NSF MCQ

1. (OSPF NSF) During a planned RP switchover on a PE running OSPF Non-Stop Forwarding (NSF) in a VRF, a neighboring CE router that does NOT support OSPF NSF/graceful-restart helper mode tears down the adjacency and reconverges normally instead of helping. Explain precisely why NSF cannot help in this scenario even though the PE itself supports it, and name the specific OSPF capability the CE is missing that would have let it act as an NSF/GR helper.

---

## Section 5 — BGP Security

### Task 5.1 — Cryptographic Algorithm Selection (TCP-AO Concepts)

> Different from LAB 3: this section is conceptual/MCQ-and-design focused on TCP Authentication Option (TCP-AO, RFC 5925) rather than a pure MD5/keychain config repeat.

1. (Design) Compare TCP-AO against classic BGP MD5 authentication and against keychain-based rotating authentication: list the specific cryptographic weakness in plain MD5 that TCP-AO's mandatory support for SHA-1-based and AES-based MACs is designed to address.
2. (Design) TCP-AO associates a **Master Key Tuple (MKT)** with a `SendID`/`RecvID` pair per session. Explain why this per-session key identification matters operationally when a single router has many BGP peers each requiring independent key rollover, compared to a single shared MD5 password.
3. (Config concept) On PE1's eBGP session to CE1, configure a keychain named `BGP-KC-SEC` with two overlapping-lifetime keys:
   - Key ID 1: cryptographic algorithm `HMAC-SHA1-96`, send-lifetime `00:00:00 Jan 1 2026` to `00:00:00 Feb 1 2026`.
   - Key ID 2: cryptographic algorithm `AES-128-CMAC-96`, send-lifetime `00:00:00 Jan 25 2026` to `00:00:00 Mar 1 2026` (overlap with Key ID 1 to guarantee a hitless rollover window).
4. Apply the keychain to the PE1-CE1 BGP neighbor using the platform's TCP-AO/keychain-based authentication command (not legacy `password` MD5 syntax).
5. Validate the rollover by advancing the lab clock (or using a short demonstration lifetime window) past the Key ID 1 → Key ID 2 boundary and confirming the BGP session does NOT flap during the transition.
6. (Design) Explain why overlapping key lifetimes (rather than a hard cutover at a single timestamp) are mandatory for a hitless rollover, and what happens to the session if the two routers' clocks are not synchronized (e.g., via NTP) when lifetimes are configured in absolute (not duration) form.

**Verify:** `show bgp neighbors <CE1-ip>` (authentication fields), `show keychain BGP-KC-SEC`, `show bgp neighbors <CE1-ip> detail | include Auth`

---

## Section 6 — Automation: NSO, Ansible, Telemetry

### Task 6.1 — NSO Service Troubleshooting — Out-of-Sync Device

1. In your NSO instance, identify a device (choose PE2) that has drifted out-of-sync with NSO's CDB — simulate this by making a CLI change directly on PE2 (e.g., modify a loopback description) that bypasses NSO.
2. Run a `check-sync` against PE2 from NSO and confirm it reports out-of-sync, then capture the diff NSO reports between CDB and the live device config.
3. Decide whether to `sync-from` (pull the live device state into NSO's CDB, accepting the out-of-band change) or `sync-to` (push NSO's CDB back down, discarding the drift) — justify your choice given that the drifted change was an authorized emergency CLI fix made by an on-call engineer.
4. After reconciling with your chosen sync direction, re-deploy the NSO service instance that owns PE2's L3VPN configuration (the `CUST-A` service from Task 2.4) and confirm the **re-deploy reconcile** operation does not revert or duplicate the legitimate drifted change, while still ensuring the service-owned configuration (VRF/RT/RD) matches the service template exactly.
5. Explain the operational difference between a plain `re-deploy` and a `re-deploy reconcile` in NSO, specifically regarding how each treats configuration on the device that NSO does not consider part of the service but that exists within the service's "owned" configuration tree.

**Verify:** NSO CLI `devices device PE2 check-sync`, `devices device PE2 compare-config`, `services l3vpn CUST-A re-deploy reconcile`

### Task 6.2 — Ansible Encoding Error MCQ

1. An Ansible playbook targeting PE3 via `cisco.iosxr.iosxr_config` fails with a traceback referencing a `UnicodeDecodeError` partway through pushing a large candidate configuration that includes a BGP description field copy-pasted from a non-UTF-8 source document. Identify the most likely root cause of this specific failure mode (not a generic "fix your YAML" answer), and name the specific playbook or inventory-level setting that should be audited/corrected to prevent recurrence.

### Task 6.3 — Telemetry Troubleshooting (different scenario from LAB 3)

1. Model-driven telemetry (MDT) is configured on P2 to stream `Cisco-IOS-XR-fib-common-oper` statistics to a collector every 30 seconds via a dial-out gRPC subscription, but the collector shows no incoming data despite the subscription showing `State: Active` on P2.
2. Systematically isolate the fault: check (in order) the subscription state, the sensor-group path validity against the actual YANG model loaded on P2's IOS-XR version, the destination-group transport/encoding match between P2 and the collector, and underlying IP reachability/firewall state to the collector's gRPC port.
3. Identify the specific IOS-XR telemetry show command that will reveal whether P2 believes it IS successfully exporting data (collector-side packet loss / wrong port) versus never attempting export at all (sensor-path or encoding mismatch) — this distinction determines whether you troubleshoot on P2 or on the network path/collector.
4. Correct the fault and confirm data arrives at the collector, documenting which of the four checks in step 2 was the actual root cause.

**Verify:** `show telemetry model-driven subscription <name> detail`, `show telemetry model-driven destination <name>`, `show grpc status`

---

## Completion Checklist

- [ ] Section 1: IS-IS multi-level verified, SR prefix-SID scheme deployed, Flex-Algo 128 include-all constraint validated, SR-TE explicit adjacency-SID policy with failover tested, SRv6 locators assigned per new scheme, SR-ODN ECMP margin validated.
- [ ] Section 2: EVPN EVI 4001/4002 operational with single-active multihoming, Inter-AS Option C (BGP-LU + multihop VPNv4) end-to-end LSP confirmed, ingress-marking/core-EXP QoS validated with test traffic, L3VPN redistribute-connected with filtering validated.
- [ ] Section 3: SoO + sham-link scenario validated with no loop, all troubleshooting MCQs answered in lab notes.
- [ ] Section 4: SRv6 FRR priority-limit critical validated (protected vs. unprotected prefixes behave differently), OSPF NSF MCQ answered.
- [ ] Section 5: TCP-AO concept questions answered, keychain with overlapping lifetimes configured and hitless rollover validated.
- [ ] Section 6: NSO out-of-sync device reconciled and service re-deploy reconcile validated, Ansible encoding MCQ answered, telemetry fault isolated and corrected.
