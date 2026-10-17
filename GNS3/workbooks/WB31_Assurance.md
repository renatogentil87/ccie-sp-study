# WB31 — Network Assurance (NetFlow, IPFIX, TWAMP, Y.1731, SR PM)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Telemetry, flow export, and active/passive performance monitoring across AS 65100 / AS 65200
**Level:** CCIE SP
**Prerequisite:** WB01-WB04 (IGP/BGP), WB16 (SR-MPLS), WB18 (SR-TE) — transport and SR policies in place

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). NetFlow/IPFIX, TWAMP, Y.1731 CFM/SLA, and SR performance
> measurement are **not** available on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:**
> P routers **R3** (150.1.3.3) / **R4** (150.1.4.4, AS65100), PEs **R1** (150.1.31.31) / **R14** (150.2.14.14) as monitored/measurement endpoints.
> Simulate a flow collector / TWAMP reflector reachable from the monitored nodes. SR-TE policies from WB18 reused for SR PM.
> Links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm IGP + BGP converged on the monitored nodes (R1, R3, R4, R14)
- [ ] Confirm a reachable collector IP (NetFlow/IPFIX + syslog + SNMP trap target)
- [ ] Confirm an SR-TE policy exists (from WB18) for the SR PM section
- [ ] **Snapshot:** `WB31-00-baseline`

---

## Section 1 — NetFlow / IPFIX on IOS-XR [4 points]

Export flow records from R3/R4 to a collector.

- [ ] **Task 1.1 [1 point]** — Configure a **flow exporter-map**: collector destination IP, UDP port, source interface, and **transport UDP**; set the export protocol to **IPFIX** (and/or NetFlow v9).
- [ ] **Task 1.2 [1 point]** — Configure a **flow monitor-map** referencing a **record type** (ipv4 / ipv6 / mpls / mpls-ipv4), the exporter, and cache parameters (entries, active/inactive **timeout**).
- [ ] **Task 1.3 [1 point]** — Apply the monitor to interfaces **ingress and egress** via a **sampler-map** (`flow <type> monitor <m> sampler <s> ingress|egress`).
- [ ] **Task 1.4 [1 point]** — Confirm the **IPFIX template** is exported and flow records are reaching the collector (template + data sets).
- [ ] **Task 1.5 [0 points — verification]** — Confirm the cache is populating and ageing per the configured timeouts.

**Verify / Snapshot**
- [ ] `show flow monitor <m> cache` — active flows
- [ ] `show flow exporter <e>` — transport state, templates sent, records exported
- [ ] `show sampler-map` / interface flow binding
- [ ] **Snapshot:** `WB31-01-netflow-ipfix`

---

## Section 2 — SNMP Traps and RMON [3 points]

- [ ] **Task 2.1 [1 point]** — Configure **SNMPv3** with **authPriv**: engine, **user**, **group**, and a **view** (restrict OID subtree); authentication (SHA) + privacy (AES).
- [ ] **Task 2.2 [1 point]** — Enable **traps** for routing-protocol state changes: **BGP**, **OSPF**, and **IS-IS** adjacency/neighbor transitions, directed to the trap host.
- [ ] **Task 2.3 [1 point]** — Configure an **RMON alarm + event** on an interface utilization / counter OID with rising/falling thresholds that log + trap.
- [ ] **Task 2.4 [0 points — verification]** — Trigger a neighbor flap and an RMON threshold cross; confirm traps are generated.

**Verify / Snapshot**
- [ ] `show snmp` / `show snmp users` / `show snmp group` — v3 authPriv
- [ ] `show snmp traps` — enabled trap set
- [ ] `show rmon alarm` / `show rmon event`
- [ ] **Snapshot:** `WB31-02-snmp-rmon`

---

## Section 3 — Syslog Configuration [2 points]

- [ ] **Task 3.1 [1 point]** — Configure the **syslog server destination** (`logging <collector>`), set the **severity level filtering** (e.g. informational to remote, debugging local), and the facility.
- [ ] **Task 3.2 [1 point]** — Set the **logging source-interface** (Loopback0) so log source IP is deterministic; configure timestamps.
- [ ] **Task 3.3 [0 points — verification]** — Generate a log event and confirm it arrives at the collector with the correct source and severity.

**Verify / Snapshot**
- [ ] `show logging` — remote host, severity, source-interface
- [ ] Collector receives events with Loopback0 source
- [ ] **Snapshot:** `WB31-03-syslog`

---

## Section 4 — IP SLA / TWAMP (Two-Way Active Measurement Protocol) [4 points]

- [ ] **Task 4.1 [1 point]** — Configure the **TWAMP server + reflector** on one PE (R1): `twamp` server, `reflector` with the control/test ports.
- [ ] **Task 4.2 [1 point]** — Configure the **TWAMP sender (controller)** on another PE (R14) targeting R1's reflector; set the test session parameters (packet size, interval).
- [ ] **Task 4.3 [1 point]** — Measure **round-trip delay, jitter, and packet loss** from the sender; bind the session to a performance-measurement profile.
- [ ] **Task 4.4 [1 point]** — **Schedule periodic measurements** (continuous / recurring) and confirm history buckets accumulate.
- [ ] **Task 4.5 [0 points — verification]** — Confirm the two-way metrics are symmetric-capable (TWAMP measures RTT without clock sync).

**Verify / Snapshot**
- [ ] `show performance-measurement ...` / `show twamp session` — RTT, jitter, loss
- [ ] `show twamp server` / reflector state
- [ ] History buckets populate on schedule
- [ ] **Snapshot:** `WB31-04-twamp`

---

## Section 5 — Y.1731 Performance Monitoring [4 points]

Ethernet CFM + SLA between R14 and R16 (or R31↔R14 L2 service).

- [ ] **Task 5.1 [1 point]** — Configure **Ethernet CFM**: a **Maintenance Domain**, **Maintenance Association (MA)**, and **MEP** (down/up) on the service endpoints; add a **MIP** on a transit node.
- [ ] **Task 5.2 [1 point]** — Verify CFM continuity (**CCM**) between MEPs (`show ethernet cfm peer meps`), including correct domain level and MA.
- [ ] **Task 5.3 [1 point]** — Configure **delay measurement** (**DMM/DMR**) and **loss measurement** (**LMM/LMR**) under a Y.1731 SLA profile; bind to the MEP.
- [ ] **Task 5.4 [1 point]** — Configure the **SLA profile** (probe type, interval, schedule) and confirm delay / FLR statistics accumulate.
- [ ] **Task 5.5 [0 points — verification]** — Confirm MEP CCM is up before trusting DM/LM results (no CFM = no valid SLA).

**Verify / Snapshot**
- [ ] `show ethernet cfm peer meps` — CCM up, remote MEP learned
- [ ] `show ethernet sla statistics` — delay (DMM/DMR), loss (LMM/LMR)
- [ ] `show ethernet cfm local meps detail`
- [ ] **Snapshot:** `WB31-05-y1731`

---

## Section 6 — Segment Routing Performance Monitoring [3 points]

- [ ] **Task 6.1 [1 point]** — Configure **SR PM delay measurement** on an **SR-TE policy** (from WB18): `performance-measurement` delay-profile, probe, and enable on the policy / candidate path.
- [ ] **Task 6.2 [1 point]** — Enable **liveness detection** for the SR path (PM liveness profile) so the policy invalidates on path failure.
- [ ] **Task 6.3 [1 point]** — Enable **interface delay-metric advertisement into IS-IS** (measured link delay flooded as a TLV) so Flex-Algo / low-latency SR can use it.
- [ ] **Task 6.4 [0 points — verification]** — Confirm the measured delay is advertised and the SR policy reflects liveness state.

**Verify / Snapshot**
- [ ] `show performance-measurement sr-policy ...` — delay, liveness
- [ ] `show performance-measurement interfaces` — measured link delay
- [ ] `show isis interface <intf>` — advertised delay metric
- [ ] **Snapshot:** `WB31-06-sr-pm`

---

## Section 7 — Verification & Troubleshooting Drills [2 points]

- [ ] **Drill 7.1 — Flow export dead [1 point]:** misconfigure the exporter transport/source so records never reach the collector. Confirm via `show flow exporter` (transport down / 0 exported), diagnose, and fix.
- [ ] **Drill 7.2 — Y.1731 SLA with no CCM [1 point]:** break the CFM MA (wrong level) so MEPs don't peer; confirm DM/LM returns no valid data; diagnose via `show ethernet cfm peer meps` and restore.
- [ ] **Snapshot:** `WB31-07-tshoot`

**Required show commands (fluency):**
- [ ] `show flow monitor <m> cache` / `show flow exporter <e>`
- [ ] `show snmp` / `show rmon alarm` / `show logging`
- [ ] `show performance-measurement ...` / `show twamp session`
- [ ] `show ethernet cfm peer meps` / `show ethernet sla statistics`
- [ ] `show isis interface <intf>` (delay metric)

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: NetFlow / IPFIX | 4 | Intermediate: exporter/monitor/sampler, IPFIX |
| Section 2: SNMP + RMON | 3 | Intermediate: SNMPv3 authPriv, traps, RMON |
| Section 3: Syslog | 2 | Standard: remote logging, severity, source |
| Section 4: TWAMP | 4 | CCIE-level: server/reflector/sender, RTT/jitter/loss |
| Section 5: Y.1731 PM | 4 | CCIE-level: CFM, DMM/DMR, LMM/LMR, SLA |
| Section 6: SR PM | 3 | CCIE-level: delay, liveness, IS-IS delay TLV |
| Section 7: Troubleshooting Drills | 2 | CCIE-level: break-and-fix |
| **Total** | **22** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
19-22 points:  CCIE ready for this topic
15-18 points:  Strong, minor gaps to review
10-14 points:  Needs more practice
Below 10:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] NetFlow/IPFIX exporter + monitor + sampler exporting records to collector
- [ ] SNMPv3 authPriv with routing traps and RMON alarm/event
- [ ] Syslog to remote collector with severity filtering + source-interface
- [ ] TWAMP server/reflector/sender measuring RTT, jitter, loss on schedule
- [ ] Y.1731 CFM + DMM/DMR + LMM/LMR SLA with CCM verified
- [ ] SR PM delay + liveness + IS-IS delay-metric advertisement
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB31-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: NetFlow / IPFIX** | **4** | | |
| 1.1 Flow exporter-map | 1 | | |
| 1.2 Flow monitor-map | 1 | | |
| 1.3 Ingress/egress + sampler | 1 | | |
| 1.4 IPFIX template exported | 1 | | |
| **Section 2: SNMP + RMON** | **3** | | |
| 2.1 SNMPv3 authPriv | 1 | | |
| 2.2 Routing traps | 1 | | |
| 2.3 RMON alarm/event | 1 | | |
| **Section 3: Syslog** | **2** | | |
| 3.1 Server + severity | 1 | | |
| 3.2 Source-interface | 1 | | |
| **Section 4: TWAMP** | **4** | | |
| 4.1 Server + reflector | 1 | | |
| 4.2 Sender/controller | 1 | | |
| 4.3 RTT/jitter/loss | 1 | | |
| 4.4 Scheduled measurement | 1 | | |
| **Section 5: Y.1731 PM** | **4** | | |
| 5.1 CFM MD/MA/MEP/MIP | 1 | | |
| 5.2 CCM continuity | 1 | | |
| 5.3 DMM/DMR + LMM/LMR | 1 | | |
| 5.4 SLA profile | 1 | | |
| **Section 6: SR PM** | **3** | | |
| 6.1 SR-TE delay measurement | 1 | | |
| 6.2 Liveness detection | 1 | | |
| 6.3 IS-IS delay-metric adv | 1 | | |
| **Section 7: Troubleshooting** | **2** | | |
| 7.1 Flow export dead | 1 | | |
| 7.2 Y.1731 no CCM | 1 | | |
| **TOTAL** | **22** | | |
