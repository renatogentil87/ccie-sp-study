# E15: VPLS / H-VPLS (Emerald)

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01

---

### Task 1: Manual VPLS (LDP-signaled, E-R1+E-R2 full mesh)
1. `l2vpn / bridge group / bridge-domain` with PW neighbors E-R1↔E-R2. VPN-ID 100.
2. Bind AC (CE1/CE2 facing interfaces) to bridge-domain. MAC learning.
3. Verify: `show l2vpn bridge-domain` — PWs UP, MACs learned.

### Task 2: Split-horizon
1. Traffic from PW NOT forwarded to another PW. Only to local ACs.

### Task 3: BGP VPLS auto-discovery
1. `l2vpn vfi context / vpn id 100 / autodiscovery bgp / signaling ldp / rd / route-target`.
2. PEs discover each other via BGP. No manual neighbor config.

### Task 4: H-VPLS
1. E-R1 = hub (N-PE), E-R2 = spoke (U-PE). Spoke only peers with hub. Hub does full mesh.

## Checklist
```
[ ] Manual VPLS (full mesh PWs, MAC learning, split-horizon)
[ ] BGP VPLS auto-discovery
[ ] H-VPLS (hub E-R1, spoke E-R2)
```
