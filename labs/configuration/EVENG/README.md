# EVE-NG Configuration Labs — IOS-XRv 9000 + CSR1000v
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3 (NIC0/NIC1=internal/mgmt)

**Platform:** EVE-NG on AWS m8i.24xlarge (384GB RAM)
**Topology:** 18 nodes (Emerald AS 65100 + Garnet AS 65200) — see `eveng_topology_reference.md`
**Images:** IOS-XRv 9000 7.11.1 + CSR1000v IOS-XE 17.x
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3 (NIC0/NIC1=internal/mgmt)

---

## What Works Here (11 labs)

| Category | Labs | Topics |
|----------|------|--------|
| Segment Routing | 16 | SR-MPLS, SRv6, SR-TE, BGP color, ODN, TI-LFA |
| EVPN | 17 | EVPN-VPWS, route types, multi-homing |
| QoS | 15 | DiffServ in IOS-XR syntax (policy-map, class-map) |
| mVPN | 13, 29 | Profiles 0, 3, 11, 12, 14 (BGP AD, mLDP) |
| mLDP | 27 | mLDP P2MP, MoFRR |
| PCE | 30 | SR-PCE, PCEP, stateful PCE, ODN |
| Flex-Algo | 31 | Network slicing, delay/TE metrics |
| Security | 14 (partial), 34 | BGP Flowspec, RPKI |
| Automation | 18 | NETCONF/YANG, gRPC telemetry |
| HA | 23 (partial) | NSR/SSO concepts (if dual-RP available) |

## Key Differences from GNS3 Labs

| | GNS3 (7200) | EVE-NG (XRv) |
|---|---|---|
| CLI | IOS classic | **IOS-XR** (`commit`, `route-policy`) |
| Config model | Immediate apply | **Candidate → commit** (atomic) |
| Route filtering | `route-map` | **`route-policy`** (RPL) |
| BGP neighbors | `peer-group` | **`neighbor-group`** |
| Transport | LDP + RSVP-TE | **SR-MPLS + SRv6** |
| FRR | Backup tunnels | **TI-LFA** (one command) |

---

## Topology Quick Reference

```
EMERALD AS 65100 (IS-IS + LDP)              GARNET AS 65200 (IS-IS + SR/SRv6)

CE1,CE2 (65012)  CE3 (OSPF)                 CE4 (65012)  CE5 (EVPN)  CE6 (OSPF)
  ↕                ↕                            ↕           ↕           ↕
PE1 ─── PE2      PE2                          PE3 ──────── PE4
  ↕                                             ↕
P1 ── P2 ── ASBR1 ═══════ ASBR2 ── P3 ── P4 ── P5
       ↕                              ↕
      RR1                             RR2
```

Customer A (AS 65012) spans both SPs: CE1/CE2 (Emerald) ↔ CE4 (Garnet)
