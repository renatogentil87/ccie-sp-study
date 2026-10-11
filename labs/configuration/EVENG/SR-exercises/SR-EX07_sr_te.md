# SR-EX07: SR-TE (Traffic Engineering with Segment Routing)

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 + SR-EX02 complete (IS-IS + SR + TI-LFA working on all routers)

**End Goal:** Explicit and dynamic SR-TE policies, PCE-based path computation, On-Demand Next-hop (ODN) with BGP color steering.

---

## Section 1: Explicit SR-TE Policy

### Task 1
Create an explicit SR-TE policy on R1 that forces traffic to R6 via a non-shortest path: R1 → R2 → R4 → R5 → R6. The IGP shortest path is R1 → R3 → R6 — your policy must override it.

Define a segment-list using prefix-SIDs for each waypoint. Apply the policy with a color and endpoint.

### Task 2
Verify the SR-TE policy is UP. Check the segment-list is installed and the forwarding entry uses the explicit path. Traceroute should follow R1 → R2 → R4 → R5 → R6, not the IGP shortest path.

### Task 3
Create a second SR-TE policy on R1 to R6 with a different color using adjacency-SIDs instead of prefix-SIDs. This gives per-link granularity — force specific interfaces, not just specific nodes.

### Task 4
Verify: compare the two policies. One uses prefix-SIDs (node waypoints), the other uses adj-SIDs (link waypoints). Both reach R6 via the same path but with different label stacks.

---

## Section 2: Dynamic SR-TE Policy

### Task 5
Create a dynamic SR-TE policy on R1 to R6 that uses IGP metric as the optimization objective. The headend computes the path locally using CSPF (no PCE needed).

### Task 6
Verify: the dynamic policy computes the IGP shortest path automatically. Change an IS-IS metric on a link along the path. Verify the policy recomputes and shifts to the new shortest path without manual intervention.

### Task 7
Create a dynamic SR-TE policy with a bandwidth constraint. The path must only use links with bandwidth ≥ 100G. Verify which links qualify and confirm the computed path avoids low-bandwidth links.

---

## Section 3: PCE-Based Path Computation

### Task 8
Configure R3 as a local SR-PCE (Path Computation Element). Enable the PCE daemon and BGP-LS so the PCE has full topology visibility.

### Task 9
Configure R1 as a PCC (Path Computation Client). R1 should delegate path computation for an SR-TE policy to the PCE on R3 via PCEP (TCP 4189).

### Task 10
Create a PCE-delegated SR-TE policy on R1 to R6. R1 requests the path from R3 (PCE). R3 computes the constrained shortest path and returns the segment-list to R1 via PCEP. Verify the policy is UP and shows "Delegated" status.

### Task 11
On the PCE (R3), change the computation constraints (e.g., exclude a link). Verify R1's policy automatically updates with the new path — R1 doesn't need reconfiguration. The PCE pushes the update.

---

## Section 4: On-Demand Next-hop (ODN) with BGP Color

### Task 12
Configure an ODN template on R1: when a BGP VPN route arrives with a specific color community, automatically create an SR-TE policy to the BGP next-hop using that color's constraints.

### Task 13
If SR-EX05 L3VPN is configured: tag CE2's VPN routes with BGP color 100 on R6. R1 receives the route with color 100 + next-hop R6. ODN should auto-create an SR-TE policy matching color 100 to R6.

### Task 14
Verify: the ODN policy is auto-created. VPN traffic to CE2 is steered into the SR-TE policy. Traceroute shows the engineered path, not the IGP default.

### Task 15
Remove the color from CE2's routes. The ODN policy should be torn down automatically. Traffic reverts to the default IGP path. No manual cleanup needed.

---

## Section 5: SR-TE + Flex-Algo Integration

### Task 16
If SR-EX04 Flex-Algo is configured: create an ODN template that maps BGP color 128 to Flex-Algo 128 prefix-SIDs. When a VPN route arrives with color 128, the SR-TE policy pushes the algo 128 SID instead of the algo 0 SID.

### Task 17
Verify: traffic follows the Flex-Algo 128 topology (e.g., avoiding red-tagged links) purely based on the BGP color. No explicit segment-list — the algo SID handles the path.

---

## Section 6: SR-TE Verification and Troubleshooting

### Task 18
Policy shows DOWN. What are the common causes? Check: segment-list has unreachable SIDs, endpoint not reachable, PCEP session to PCE is down, constraint can't be satisfied.

### Task 19
Compare all SR-TE approaches side by side:
- Explicit policy (manual segment-list)
- Dynamic policy (local CSPF computation)
- PCE-delegated (controller computes)
- ODN + BGP color (auto-created on demand)

When would you use each? What are the tradeoffs?

---

## Snapshot
Take: **"SR-EX07-srte"**

## Checklist
```
[ ] Explicit SR-TE policy: R1→R6 via non-shortest path using prefix-SIDs
[ ] Same policy using adj-SIDs — compare label stacks
[ ] Dynamic policy: auto-computes IGP shortest, recomputes on metric change
[ ] Dynamic policy with bandwidth constraint
[ ] R3 configured as SR-PCE with BGP-LS
[ ] R1 as PCC, PCEP session to R3 established
[ ] PCE-delegated policy UP and showing "Delegated"
[ ] PCE constraint change auto-updates R1's policy
[ ] ODN template configured on R1
[ ] BGP color 100 triggers auto-created SR-TE policy
[ ] Color removed → ODN policy torn down automatically
[ ] (Optional) ODN + Flex-Algo integration via color 128
[ ] Troubleshooting: policy DOWN causes understood
```
