# CCIE SP Workbook 16 — Automation & Assurance

**Domain:** 6 — Automation & Assurance (20% of the CCIE SP lab)
**Platform:** IOS-XRv 9000 + NSO VM
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** All 3 ISPs (Emerald + Gold + Garnet) = 20 XRv routers, plus a Linux NSO VM. All devices reachable via the OOB management network. NSO/collector/workstation share the mgmt subnet (assume `10.0.0.0/24`, NSO/collector = `10.0.0.100`).

**Format:** Each task is **Question → Solution → Verification**. All device syntax is IOS-XR unless noted. Python snippets target Python 3.9+.

---

## Section 1: NETCONF / YANG

### Task 1.1 — Enable NETCONF on all 20 routers

**Question:** Enable NETCONF-over-SSH (port 830) on every XRv router so NSO, ncclient, and gNMI collectors can reach them. What is the minimum config, and how do you confirm the agent is listening?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — YANG model navigation (native + OpenConfig)

**Question:** You must know which YANG model to query. How do you list the models a router advertises, and what is the difference between the Cisco-IOS-XR native models and OpenConfig?


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Retrieve running config with ncclient

**Question:** Use the Python `ncclient` library to pull E-R1's IS-IS configuration via NETCONF and print the XML.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Push an IS-IS metric change with edit-config (candidate → commit)

**Question:** Change the IS-IS metric on E-R1's `GigabitEthernet0/0/0/0` from 10 to 100 using NETCONF `edit-config` against the **candidate** datastore, then commit. Demonstrate you understand the XR candidate→commit model.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2: gRPC / gNMI Model-Driven Telemetry

### Task 2.1 — Enable gRPC on XRv (port 57400)

**Question:** Enable the gRPC server on all XRv routers on port 57400 so a collector can dial-in (gNMI) and the router can dial-out (MDT).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — Dial-out vs dial-in (concept + when to use)

**Question:** Explain MDT **dial-out** vs **dial-in** and identify which one the router initiates.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — Configure dial-out telemetry (sensor-path, subscription, destination)

**Question:** On E-R1 configure a dial-out subscription that streams **interface stats**, **BGP neighbor state**, and **IS-IS adjacency** to the collector `10.0.0.100:57500` every 10 seconds.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4 — Dial-in gNMI subscribe

**Question:** From the workstation, use gNMI SUBSCRIBE (dial-in) to watch BGP neighbor state on E-R1 at a 5s sample interval, then trigger a change.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3: NSO (Network Services Orchestrator)

### Task 3.1 — NSO architecture

**Question:** Describe the core NSO components and how a service commit flows through them.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — Add XRv devices and authgroups

**Question:** Add E-R1 and G-R1 to NSO inventory with the correct NED-id and an authgroup.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — sync-from

**Question:** Pull the running config from all devices into CDB.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4 — L3VPN service package: VRF across E-R1 + G-R1 in one transaction

**Question:** Using an L3VPN service package, provision VRF **CUST-A** on both E-R1 and G-R1 in a single NSO transaction. Preview before committing.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.5 — NSO rollback (undo last commit)

**Question:** The CUST-A rollout was wrong. Undo the last commit atomically.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.6 — Compliance check (consistent IS-IS across all PEs)

**Question:** Verify that all PE routers have consistent IS-IS configuration using an NSO compliance report.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4: Python Scripting

### Task 4.1 — Collect show commands from all routers (netmiko)

**Question:** Write a Python script using netmiko to collect `show isis neighbors` and `show interfaces summary` from all 20 routers and save per-device output.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — Parse output (regex + TextFSM) → topology report

**Question:** Parse the collected `show isis neighbors` output to build a neighbor adjacency table. Show both a regex approach and TextFSM (ntc-templates).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Verify IS-IS adjacency health across all 3 SPs

**Question:** Write a health-check script that flags any router whose IS-IS neighbor count is below expected or has non-`Up` adjacencies.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5: Zero-Touch Provisioning (ZTP)

### Task 5.1 — ZTP concept + iPXE boot + first-boot Python script

**Question:** Explain the ZTP boot flow on IOS-XR and write a first-boot Python ZTP script that sets hostname, a loopback, and IS-IS.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — Secure ZTP (certificate-based)

**Question:** Standard ZTP trusts whatever DHCP hands out. How does Secure ZTP (RFC 8572) harden this?


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6: Assurance Tools

### Task 6.1 — Syslog to a central server

**Question:** Configure all routers to send syslog to `10.0.0.100` at severity informational with source loopback0.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — SNMP v3 (user / group / view)

**Question:** Configure SNMPv3 with authPriv: view (restrict to system + interfaces MIB), group, and user, sending traps to the NMS.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.3 — NetFlow/IPFIX on PE interfaces + BFD echo mode

**Question:** Enable IPFIX (NetFlow v10) export on PE customer-facing interfaces to `10.0.0.100`, and enable BFD echo mode on a core link for fast liveliness.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 7: Troubleshooting

### Task 7.1 — NETCONF session rejected

**Question:** A NETCONF client to PE7:830 either refuses the connection or connects but never returns a `<hello>`. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 7.2 — Telemetry data not arriving

**Question:** The dial-out subscription `SUB-CORE` on E-R1 is configured but the collector receives nothing. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*


---

## Section 8: NSO Service Package Development (CCIE Exam Critical)

### Task 8.1 — NSO dry-run (commit dry-run outformat native)

**Question:** You are about to push a VRF change via NSO to 3 PEs. Before committing, preview exactly what CLI NSO will send to each device. What command do you use, and what's the difference between `outformat native` and `outformat xml`?

> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 8.2 — Write a custom L3VPN service package

**Question:** Create an NSO service package that provisions L3VPN across any set of PEs with a single command. The service takes: customer-name, VRF-name, RD, RT, list of PEs with their CE-facing interfaces and CE ASNs.

> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 8.3 — Multi-device atomic rollback

**Question:** You pushed a wrong RT (65012:999 instead of 65012:100) to E-R1, G-R1, and Gar-R1 via NSO. All 3 PEs now have broken VPN connectivity. Roll back all 3 in one atomic operation.

> *Try this yourself first. Solution available in `solutions/` folder.*

---

## Section 9: Advanced Python Workflows

### Task 9.1 — Python VRF consistency audit

**Question:** Write a Python script that connects to all PEs via NETCONF, pulls VRF CUST_A config, and reports any RT mismatches across PEs.

> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 9.2 — Python SR prefix-SID uniqueness check

**Question:** Write a script that validates no two Garnet routers share the same prefix-SID index.

> *Try this yourself first. Solution available in `solutions/` folder.*

---

## Section 10: MDT Advanced (XPATH + Event-Driven)

### Task 10.1 — XPATH-filtered telemetry subscription

**Question:** Configure a telemetry subscription that streams ONLY BGP neighbor connection state changes — not the entire BGP operational tree.

> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 10.2 — Event-driven vs cadence-based telemetry

**Question:** Configure an event-driven (on-change) subscription for IS-IS adjacency state, and a cadence-based (sample-interval) subscription for interface counters. Explain when to use each.

> *Try this yourself first. Solution available in `solutions/` folder.*
