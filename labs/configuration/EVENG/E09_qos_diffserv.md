# E09: QoS DiffServ (IOS-XR)

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 + E02

**End Goal:** Full DiffServ in IOS-XR syntax. Classification, marking, scheduling, policing. Path: CE1→E-R1→E-R3→E-R4→E-R2.

---

### Task 1: Classify at E-R1 ingress (Gi0/0/0/0 toward CE1)
1. class-map matching DSCP EF (voice), AF31 (video), AF11 (critical), default.
2. Apply `service-policy input` on E-R1 Gi0/0/0/0.

### Task 2: Mark DSCP→TC (traffic-class) for MPLS
1. policy-map on E-R1: `set traffic-class` per class for MPLS EXP equivalent.

### Task 3: Core QoS on E-R3 and E-R4
1. Classify based on traffic-class. Priority queue (EF), CBWFQ (AF31/AF11), WRED (default).
2. Apply `service-policy output` on all core interfaces.

### Task 4: Per-customer policing on PE-CE
1. E-R1 Gi0/0/0/0: police CE1 at 50Mbps. E-R2 Gi0/0/0/2: police CE3 at 20Mbps.

### Task 5: Pipe model
1. Preserve customer DSCP through core. Verify end-to-end DSCP unchanged.

## Checklist
```
[ ] Classification at PE ingress; DSCP→TC marking
[ ] Core P routers: TC-based scheduling (LLQ + CBWFQ + WRED)
[ ] Per-customer policing (50M vs 20M)
[ ] Pipe model: customer DSCP preserved end-to-end
```
