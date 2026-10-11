# SR-EX05: L3VPN End-to-End over SR

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 complete (all routers running IS-IS + SR)

---

## Section 1: BGP VPNv4

### Task 1
Choose an RR for the network. Configure BGP AS 65100 with VPNv4 address-family on the RR. Add R1, R2, and R6 as RR clients. Use Loopback0 as update-source.

### Task 2
Configure VRF `CUSTOMER` on R1:
- Assign the CE1-facing interface to the VRF
- Choose appropriate RD and RT (same RT across all PEs for the same customer)
- Configure eBGP PE-CE neighbor with CE1 (AS 65001)
- Don't forget the route-policy (IOS-XR requirement on eBGP)

### Task 3
Configure VRF `CUSTOMER` on R2:
- Same VRF for CE1's second link (dual-homed)
- Same RT as R1
- eBGP PE-CE with CE1 (AS 65001)
- Apply `as-override` — CE1 uses the same ASN on both links

### Task 4
Configure VRF `CUSTOMER` on R6:
- CE2-facing interface in the VRF
- Same RT so it imports routes from R1/R2
- eBGP PE-CE with CE2 (AS 65002)

---

## Section 2: End-to-End Verification

### Task 5
Verify BGP VPNv4 sessions: all PE↔RR sessions Established with prefixes exchanged.

### Task 6
Verify CE1's routes appear on R6 and CE2's routes appear on R1/R2. Check the VPN next-hop and VPN label.

### Task 7
CE1 ping CE2. Traceroute — observe the SR transport labels carrying the VPN traffic across the network. The VPN label rides underneath the SR prefix-SID.

### Task 8
CE1 cannot reach the SP core (172.16.x.x addresses). Verify VRF isolation — traffic to unknown VRF destinations is dropped.

---

## Section 3: VPN + TI-LFA

### Task 9
If SR-EX02 was completed: continuous ping CE1 → CE2. Shut a link on the SR path. TI-LFA kicks in — 0-1 packet loss. VPN traffic is protected by SR fast-reroute.

### Task 10
Shut a node on the path (if node protection is configured). VPN traffic reroutes around the dead node. CE1↔CE2 still works.

---

## Section 4: VPN + Flex-Algo (optional, requires SR-EX04)

### Task 11
Steer VPN traffic for CE2 using a Flex-Algo. Apply a BGP color community on CE2's routes. The headend PE should match the color to an SR-TE ODN policy using the Flex-Algo prefix-SID.

### Task 12
Verify: traceroute from CE1 to CE2 follows the Flex-Algo path, not the default IGP path.

### Task 13
Remove the color. Traffic reverts to default Algo 0 path.

---

## Snapshot
Take: **"SR-EX05-l3vpn"**

## Checklist
```
[ ] RR configured with VPNv4, PE sessions Established
[ ] VRF CUSTOMER on R1 + R2 + R6 with same RT
[ ] PE-CE eBGP with route-policy on all PEs
[ ] as-override on R1/R2 (same CE ASN both links)
[ ] CE1 routes visible on R6, CE2 routes visible on R1/R2
[ ] CE1 ↔ CE2 ping works end-to-end
[ ] Traceroute shows SR labels + VPN label
[ ] VRF isolation: CE cannot reach SP core
[ ] TI-LFA protects VPN traffic during link/node failure
[ ] (Optional) Flex-Algo steering via BGP color works
```


