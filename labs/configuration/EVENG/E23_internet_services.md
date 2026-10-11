# E23: Internet Services from VRF

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02 (L3VPN working)
**Base:** IS-IS L2 on backbone, full loopback reachability, VRFs on PEs.

**End Goal:** VPN customers access the internet. 4 options tested — from simplest to most scalable.

---

### Task 1: Option 1 — VRF-specific default route
1. On E-R1: inject a static default into VRF CUST_A pointing to the global table.
   - `router static / vrf CUST_A / address-family ipv4 / 0.0.0.0/0 <global-next-hop>`
2. CE1 gets a default route → internet traffic exits via E-R1 into global table.
3. Verify: CE1 can reach a "simulated internet" destination (use an ASBR loopback as stand-in).
4. Simplest approach. No NAT. Customer uses SP's public IP space or needs NAT elsewhere.

### Task 2: Option 2 — Separate PE-CE interface for internet
1. On E-R2: CE3 has a second interface (or use the existing one) that connects to global routing (no VRF).
2. CE3 learns internet routes via this non-VRF interface. VPN routes via the VRF interface.
3. Customer controls what goes where (split routing on CE).
4. Verify: CE3 reaches internet destinations via global interface, VPN destinations via VRF interface.

### Task 3: Option 3 — Internet VRF (route leaking)
1. Create VRF INTERNET on E-R1 with its own RD/RT.
2. Import INTERNET RT into CUST_A → customer can reach internet prefixes.
3. Import CUST_A RT into INTERNET → return traffic works.
4. Verify: CE1 reaches internet (via INTERNET VRF routes leaked into CUST_A).
5. Customers isolated from each other. Only shared via INTERNET VRF.

### Task 4: Option 4 — VRF-aware NAT (concept)
1. PE performs NAT between VRF and global table.
2. Customer traffic arrives in VRF → NAT translates source to public IP → exits to global/internet.
3. Return traffic → reverse NAT → back into VRF → to customer.
4. Document IOS-XR config reference (VRF-aware NAT may have limited support on XRv — configure what's supported, document the rest).

### Task 5: Internet access across Inter-AS (after E06)
1. Customer A (CE1 in Emerald) needs internet. Internet gateway is in Garnet (e.g., Gar-R7 or Gar-R2).
2. Default route leaks from Garnet → crosses inter-AS → reaches CE1.
3. Verify: CE1 can reach a destination behind Garnet's "internet exit."

## Checklist
```
[ ] Option 1: static default in VRF → internet via global table
[ ] Option 2: separate PE-CE interface (VRF + global split)
[ ] Option 3: Internet VRF with RT leaking (inter-VRF)
[ ] Option 4: VRF-aware NAT concept documented
[ ] Internet across inter-AS (after E06)
```
