# E22: NSO (Network Services Orchestrator)

**Platform:** IOS-XRv 9000 + CSR1000v + Linux NSO VM | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02 (L3VPN working), E14 (NETCONF enabled on all routers)
**Base:** IS-IS L2 on backbone, full loopback reachability, L3VPN operational.

**End Goal:** NSO provisions L3VPN services across all 3 SPs. Automate what you configured manually in E02.

---

### Task 1: NSO setup
1. Deploy NSO Linux VM on EVE-NG (or GNS3). Connect to management network.
2. Install NEDs: cisco-iosxr + cisco-ios (for CSR CEs).
3. Verify: `ncs --version`, `show packages` — NEDs loaded.

### Task 2: Add all devices to NSO
1. In NSO CLI: `devices device E-R1 / address 1.1.1.1 / authgroup lab / device-type cli ned-id cisco-iosxr`.
2. Repeat for E-R2, Gar-R1, Gar-R2, E-R4 (RR), Gar-R6 (RR), E-R6, Gar-R7.
3. Add CSR CEs: CE1-CE6 with ned-id cisco-ios.
4. `devices sync-from` — NSO pulls running config from all devices.
5. Verify: `show running-config devices device E-R1` in NSO matches actual E-R1.

### Task 3: L3VPN service provisioning
1. Create L3VPN service template (or use built-in if available):
   - Customer name, VRF name, RD, RT
   - PE list: E-R1 (interface Gi0/0/0/0, CE1 neighbor, AS 65012)
   - Gar-R1 (interface Gi0/0/0/2, CE4 neighbor, AS 65012)
2. `commit dry-run` — see what NSO WOULD push to E-R1 and Gar-R1.
3. `commit` — NSO pushes VRF + BGP config to both PEs simultaneously.
4. Verify: CE1 ↔ CE4 VPN works (after inter-AS from E06 is configured).

### Task 4: NSO rollback
1. `rollback configuration` — NSO removes the VRF from all devices atomically.
2. Verify: VRF gone from E-R1 and Gar-R1. CE traffic stops.
3. Re-apply → VRF returns.

### Task 5: Compliance check
1. Define golden config template for PE role (IS-IS auth, BFD timers, logging).
2. `compliance check` → NSO compares running vs golden.
3. Report deviations. Optionally auto-remediate.

### Task 6: BFD + IGP tuning via NSO template
1. Create template: enable BFD on all IS-IS interfaces (all 3 SPs).
2. Push to all 13 XRv routers in one commit.
3. Verify: `show bfd neighbors` on each — BFD UP.

### Task 7: SRv6 policy via NSO
1. Create template for SRv6-TE policy (from E21).
2. Push to Gar-R1: SRv6-TE policy auto-deployed.
3. Verify: `show segment-routing traffic-eng policy` — active.

### Task 8: ACL deployment via NSO
1. Define infrastructure ACL (protect loopbacks from CE traffic).
2. Push to all PE routers simultaneously.
3. Verify: `show access-lists` — ACL applied.

## Checklist
```
[ ] NSO installed with NEDs (XR + IOS)
[ ] All devices synced into NSO
[ ] L3VPN provisioned via NSO across E-R1 + Gar-R1 (one commit)
[ ] Rollback works atomically
[ ] Compliance check identifies drift
[ ] BFD template pushed to all routers
[ ] SRv6 policy deployed via NSO
[ ] ACL deployed via NSO
```

---

## Section 2: NSO Advanced (CCIE SP Exam Focus)

### Task 9: NSO dry-run
1. Before pushing any config, use `commit dry-run outformat native` to preview exactly what NSO will send to each device.
2. Compare `outformat native` (device CLI) vs `outformat xml` (YANG XML). Know when to use each.
3. Practice: create a VRF change, dry-run it, review the output, THEN commit.

### Task 10: NSO service package — custom L3VPN
1. Create a simple NSO service package for L3VPN provisioning.
2. Structure: `package-meta-data.xml` (metadata), `src/yang/` (service YANG model), `templates/` (XML device template).
3. The YANG model defines inputs: customer-name, VRF-name, PE-list, RD, RT, PE-CE interface, CE-ASN.
4. The XML template maps inputs to IOS-XR config (VRF + BGP VPNv4 + interface).
5. Deploy: `packages reload`, then instantiate: `services l3vpn CUST_A pe [ E-R1 G-R1 ] rd 65012:100 rt 65012:100`.
6. Verify: `show configuration commit changes last 1` — see what NSO pushed.

### Task 11: NSO compliance reporting
1. Define a compliance template: "all PEs must have IS-IS metric-style wide."
2. Run compliance check: `compliance reports report ISIS-CHECK run`.
3. Review violations. Fix non-compliant devices via NSO.

### Task 12: NSO rollback — multi-device transaction
1. Push a bad config (wrong RT on 3 PEs simultaneously via NSO).
2. Verify traffic is broken.
3. `rollback configuration` — NSO reverts ALL 3 PEs in one atomic transaction.
4. Verify traffic restored. This is the power of NSO — multi-device atomic rollback.

### Updated Checklist
```
[ ] NSO dry-run shows exact CLI before commit
[ ] Custom L3VPN service package created and deployed
[ ] Compliance check identifies non-compliant routers
[ ] Multi-device rollback works atomically
```
