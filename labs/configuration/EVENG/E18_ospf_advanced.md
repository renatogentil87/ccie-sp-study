# E18: OSPF Advanced

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

---

### Task 1: OSPF PE-CE (VRF)
1. E-R2 Gi0/0/0/2 → CE3 (OSPF Area 0 in VRF CUST_B). Gar-R2 Gi0/0/0/0 → CE6 (OSPF Area 0 in VRF CUST_D).
2. Redistribute OSPF↔BGP in VRF. Verify routes propagated via RR.

### Task 2: DN-bit
1. When PE redistributes BGP→OSPF, DN-bit set. Second PE doesn't re-redistribute into BGP. Verify: `show ospf database external` — DN bit present.

### Task 3: Sham-link concept
1. If CE3 and CE6 had a backdoor: OSPF intra-area (backdoor) > inter-area (VPN). Sham-link creates intra-area adjacency across VPN core. Document the config (VRF loopbacks advertised via BGP, `area 0 sham-link`).

### Task 4: Multi-area OSPF
1. Split Emerald core OSPF into Area 0 (E-R3/E-R4) + Area 1 (E-R1/E-R2). E-R3 = ABR.
2. Summarization at ABR. Verify impact on LDP (LDP needs /32 — summarization breaks it).

### Task 5: Stub/NSSA
1. Make Area 1 a stub. Verify: no Type 5 LSAs. Then NSSA: PE can redistribute local static as Type 7.

## Checklist
```
[ ] OSPF PE-CE in VRF (CE3 + CE6)
[ ] DN-bit prevents re-redistribution loop
[ ] Sham-link concept documented
[ ] Multi-area + summarization (+ LDP conflict)
[ ] Stub/NSSA areas
```
