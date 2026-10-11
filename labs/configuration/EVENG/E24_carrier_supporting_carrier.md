# E24: Carrier Supporting Carrier (CSC)

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Emerald AS 65100 + Gold AS 65300 + Garnet AS 65200
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 (IS-IS + transport), E02 (L3VPN basics)

**End Goal:** Emerald (AS 65100) acts as the backbone provider carrying Garnet (AS 65200) as a customer SP. Garnet doesn't own the backbone between its two sites — it buys MPLS transit from Emerald. End-to-end labeled path with a 3-label stack.

---

## Scenario

Garnet is a regional SP with two PE sites (Gar-R1 and Gar-R2) but no physical backbone between them. Garnet buys VPN transit from Emerald. Emerald carries Garnet's MPLS traffic transparently across its backbone.

```
Customer ——→ Gar-R1 (Garnet PE) ——→ E-R1 (Emerald PE) ——MPLS——→ E-R2 (Emerald PE) ——→ Gar-R2 (Garnet PE) ——→ Customer
              small SP site A         backbone provider            backbone provider         small SP site B
```

Emerald sees Garnet as a VPN customer. Garnet sees Emerald as transparent MPLS transport.

---

## Section 1: Emerald as Backbone Provider

### Task 1: VRF for Garnet on Emerald PEs
1. On E-R1 and E-R2, create `vrf GARNET_TRANSIT`.
2. RD 65100:500, RT import/export 65100:500.
3. Assign the PE-CE facing interface to this VRF:
   - E-R1: Gi0/0/0/0 → `vrf GARNET_TRANSIT` (connects to Gar-R1)
   - E-R2: Gi0/0/0/0 → `vrf GARNET_TRANSIT` (connects to Gar-R2)
4. IP addressing: E-R1 Gi0 = 172.31.1.1/24, E-R2 Gi0 = 172.31.2.1/24.
5. Verify: `show vrf GARNET_TRANSIT` — interfaces assigned.

### Task 2: eBGP Labeled Unicast between Emerald PE and Garnet PE
1. This is the CSC key — the PE-CE link runs **eBGP with labels**, not regular IP.
2. On E-R1:
   ```
   router bgp 65100
    vrf GARNET_TRANSIT
     neighbor 172.31.1.2
      remote-as 65200
      address-family ipv4 labeled-unicast
       route-policy PASS-ALL in
       route-policy PASS-ALL out
   ```
3. On E-R2: same but neighbor 172.31.2.2 (Gar-R2).
4. **Why labeled-unicast?** Because Garnet is an SP — it sends MPLS-labeled packets. Emerald must carry those labels. BGP-LU on the PE-CE link lets Garnet's PE loopbacks be reachable with a label across Emerald's backbone.
5. Verify: `show bgp vrf GARNET_TRANSIT ipv4 labeled-unicast summary` — sessions Established.

### Task 3: VPNv4 on Emerald RR for GARNET_TRANSIT
1. E-R5 (RR) must have VPNv4 configured (already done in E02).
2. Verify: E-R1 and E-R2 advertise GARNET_TRANSIT routes to E-R5, E-R5 reflects them.
3. `show bgp vpnv4 unicast vrf GARNET_TRANSIT` on E-R1 — should see Gar-R2's loopback from E-R2 via RR.

---

## Section 2: Garnet as Customer SP

### Task 4: Garnet PE-to-Emerald PE link configuration
1. On Gar-R1: interface facing E-R1 = 172.31.1.2/24.
2. On Gar-R2: interface facing E-R2 = 172.31.2.2/24.
3. These are NOT in a VRF on Garnet side — they're global interfaces. Garnet treats them as backbone-facing links.

### Task 5: eBGP Labeled Unicast on Garnet PEs toward Emerald
1. On Gar-R1:
   ```
   router bgp 65200
    neighbor 172.31.1.1
     remote-as 65100
     address-family ipv4 labeled-unicast
      route-policy PASS-ALL in
      route-policy PASS-ALL out
   ```
2. On Gar-R2: same but neighbor 172.31.2.1.
3. Garnet PEs advertise their loopbacks (11.11.11.11/32, 12.12.12.12/32) with labels via BGP-LU.
4. Verify: `show bgp ipv4 labeled-unicast` on Gar-R1 — sees Gar-R2's loopback (12.12.12.12) with a label, reachable via Emerald.

### Task 6: Garnet iBGP VPNv4 over the CSC backbone
1. Gar-R1 and Gar-R2 establish iBGP VPNv4 peering using their loopbacks (11.11.11.11 ↔ 12.12.12.12).
2. The loopbacks are reachable via BGP-LU through Emerald's backbone — that's the transport.
3. On Gar-R1:
   ```
   router bgp 65200
    neighbor 12.12.12.12
     remote-as 65200
     update-source Loopback0
     address-family vpnv4 unicast
   ```
4. Verify: `show bgp vpnv4 unicast summary` on Gar-R1 — session to 12.12.12.12 Established.

---

## Section 3: End-to-End Customer VPN over CSC

### Task 7: Customer VRFs on Garnet PEs
1. Create `vrf CUST_X` on Gar-R1 and Gar-R2.
2. RD 65200:800, RT 65200:800.
3. Assign customer-facing interfaces:
   - Gar-R1: Gi0/0/0/2 → `vrf CUST_X` (connects to CE4 or a test loopback)
   - Gar-R2: Gi0/0/0/2 → `vrf CUST_X` (connects to CE6 or a test loopback)
4. Configure PE-CE routing (static or eBGP) in the VRF.

### Task 8: Verify 3-label stack
1. From Gar-R1, `traceroute vrf CUST_X` toward the customer prefix on Gar-R2 side.
2. On E-R1, check the LFIB: `show mpls forwarding` — you should see a 3-label stack:
   ```
   [Emerald transport label][Emerald VPN label for GARNET_TRANSIT][Garnet's own VPN label]
         LDP/SR                    MP-BGP VPNv4                      Garnet internal
   ```
3. The innermost label is Garnet's VPN label — Emerald doesn't know what it is, it just carries it.
4. Verify: `show cef vrf GARNET_TRANSIT <Gar-R2-loopback> detail` on E-R1 — shows label stack.

### Task 9: Verify transparency
1. From Gar-R1: `ping vrf CUST_X <customer-prefix-on-Gar-R2-side>` — works.
2. Emerald has NO visibility into CUST_X — it only sees GARNET_TRANSIT VRF.
3. `show vrf` on E-R1 — shows GARNET_TRANSIT but NOT CUST_X. Emerald is just transport.
4. Garnet runs its own MPLS, its own VPNs, its own BGP — all transparently over Emerald's backbone.

---

## Section 4: Troubleshooting CSC

### Task 10: BGP-LU session up but no labels exchanged
1. Symptom: Gar-R1 can't reach Gar-R2's loopback through Emerald.
2. Check: `show bgp vrf GARNET_TRANSIT ipv4 labeled-unicast` on E-R1 — are Garnet PE loopbacks present with labels?
3. Common cause: missing `address-family ipv4 labeled-unicast` (configured `ipv4 unicast` instead — no labels exchanged).
4. Fix: change to `labeled-unicast` on both sides.

### Task 11: Customer traffic blackholed
1. Symptom: Garnet VPNv4 session Established, but `traceroute vrf CUST_X` fails.
2. Check label stack: `show cef vrf GARNET_TRANSIT <gar-r2-loopback> detail` on E-R1 — is there a label?
3. Common cause: Emerald PE not pushing the BGP-LU label for Garnet PE loopback — the VPN next-hop (12.12.12.12) is unreachable from Garnet's perspective.
4. Check: `show bgp ipv4 labeled-unicast 12.12.12.12/32` on Gar-R1 — label present?

### Task 12: Verify CSC vs Inter-AS Option C
1. Compare the two:
   - **CSC:** Emerald carries Garnet's MPLS. Garnet is a VPN customer. 3-label stack. Garnet runs its own VPNs.
   - **Option C:** Emerald and Garnet are peers. They share the same customer. BGP-LU + multihop VPNv4 between RRs. 3-label stack but different purpose.
2. Key difference: in CSC, the inner label belongs to the **customer SP** (Garnet). In Option C, the inner label belongs to the **same VPN** across both SPs.

---

## Checklist
```
[ ] VRF GARNET_TRANSIT on E-R1 + E-R2 (Emerald PEs)
[ ] eBGP labeled-unicast between E-R1↔Gar-R1 and E-R2↔Gar-R2
[ ] Garnet PE loopbacks reachable via BGP-LU through Emerald backbone
[ ] Garnet iBGP VPNv4 (Gar-R1↔Gar-R2) over the CSC transport
[ ] Customer VRF CUST_X on Gar-R1 + Gar-R2
[ ] 3-label stack visible on Emerald PEs
[ ] Customer traffic end-to-end through CSC (ping + traceroute)
[ ] Emerald has NO visibility into Garnet's customer VRFs
[ ] Understand CSC vs Inter-AS Option C difference
```
