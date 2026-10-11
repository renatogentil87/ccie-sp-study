# E19: Unified/Seamless MPLS (BGP-LU)

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E06

---

### Task 1: Merge into single AS with two IGP domains
1. Change Garnet from AS 65200 to AS 65100 (same AS as Emerald). E-R6↔Gar-R7 becomes iBGP.
2. Two IGP domains remain: Emerald IS-IS (area 49.0001) + Garnet IS-IS (area 49.0002). No IGP across boundary.

### Task 2: BGP-LU at ASBRs
1. On E-R6: `address-family ipv4 / neighbor <Gar-R7> activate / send-label`. Advertise Emerald PE loopbacks with labels.
2. On Gar-R7: same — advertise Garnet PE loopbacks with labels. Both: `next-hop-self`.
3. Propagate to RRs (E-R4/Gar-R3) → PEs resolve remote PE loopbacks via BGP-LU.

### Task 3: End-to-end LSP
1. E-R1 `ping 11.11.11.11 source 1.1.1.1` → works (labeled path across both domains).
2. Label stack: [domain-A LDP/SR] [BGP-LU stitch label]. 2-label stack at ingress.

### Task 4: VPN over Unified MPLS
1. VPNv4 RR-RR multihop (E-R4↔Gar-R3, iBGP now). `next-hop-unchanged`.
2. CE1 ↔ CE4 in shared VRF CUST_A. 3-label stack: [domain transport] [BGP-LU] [VPN].

## Checklist
```
[ ] Single AS, two IGP domains stitched at ASBRs
[ ] BGP-LU (send-label + next-hop-self) between ASBRs
[ ] End-to-end LSP (E-R1↔Gar-R1 via labeled path)
[ ] VPN over Unified MPLS (CE1↔CE4, 3-label stack)
```
