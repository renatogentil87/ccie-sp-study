# E14: NETCONF/YANG + Telemetry

**Platform:** IOS-XRv 9000 + CSR1000v | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

**End Goal:** Model-driven programmability — NETCONF, gRPC telemetry, RESTCONF.

---

### Task 1: Enable NETCONF on all XRv (port 830)
1. `ssh server v2 / ssh server netconf vrf default / netconf-yang agent ssh`. Test SSH to port 830.

### Task 2: Python ncclient — get-config / edit-config / commit
1. `get-config` interfaces from E-R1. Parse XML. `edit-config` create Loopback100. `commit`. Delete it.

### Task 3: Push VRF via NETCONF
1. Push complete L3VPN VRF to Gar-R1 via NETCONF XML. Verify `show vrf`. No CLI used.

### Task 4: gRPC telemetry — dial-out
1. `grpc / port 57400 / no-tls` on E-R1. Sensor-group (interface counters), subscription 10s → collector.
2. Verify: data streams arrive every 10 seconds.

### Task 5: gNMI subscribe — dial-in
1. Subscribe to `/bgp/neighbors/neighbor/state` on E-R1 (sample 5s). Kill BGP session → observe state change real-time.

### Task 6: RESTCONF on CSR CEs
1. On CE3 (CSR1000v): `restconf / ip http secure-server`. GET interfaces via curl. PUT to modify.

## Checklist
```
[ ] NETCONF on all XRv (port 830, get-config/edit-config/commit)
[ ] VRF provisioned via NETCONF (no CLI)
[ ] gRPC dial-out telemetry (10s counters)
[ ] gNMI dial-in (BGP state changes)
[ ] RESTCONF on CSR CEs
```

---

## Section 2: Python SP Automation Workflows (CCIE SP Exam Focus)

### Task 7: Python — audit all PEs for consistent VRF config
1. Write a Python script using ncclient that:
   - Connects to E-R1, E-R2, Gar-R1, Gar-R2, G-R1, G-R2 via NETCONF
   - Pulls VRF configuration from each
   - Compares RT import/export across all PEs for CUST_A
   - Reports any mismatches (e.g., Gar-R1 missing RT 65012:100)
2. Output: table showing PE → VRFs → RTs → MATCH/MISMATCH.

### Task 8: Python — validate SR prefix-SID uniqueness
1. Script connects to all Garnet routers via NETCONF.
2. Pulls prefix-SID index from each Loopback0.
3. Checks for conflicts (two routers with same index).
4. Reports: "Gar-R1=index 11, Gar-R2=index 12, ... NO CONFLICTS" or flags duplicates.

### Task 9: Python — push config with rollback on failure
1. Script pushes IS-IS metric change to 3 routers via NETCONF edit-config.
2. If commit fails on any router, script rolls back ALL changes (mimics NSO behavior).
3. Use ncclient `discard_changes()` on candidate datastore for rollback.

### Task 10: MDT with XPATH filtering
1. Configure telemetry subscription with specific XPATH filter:
   - BGP neighbor state only: `sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/neighbors/neighbor/connection-state`
   - IS-IS adjacency state only: `sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/neighbors/neighbor`
2. Verify: only targeted data streams to the collector, not the entire BGP/IS-IS operational tree.
3. Configure event-driven telemetry (on-change) vs cadence-based (sample-interval). Know when to use each.

### Updated Checklist
```
[ ] Python VRF audit script works across all PEs
[ ] Python SR prefix-SID uniqueness check works
[ ] Python push-with-rollback works (edit-config + discard on failure)
[ ] MDT XPATH filtering delivers only targeted data
[ ] Event-driven vs cadence-based telemetry understood
```
