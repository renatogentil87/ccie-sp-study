# Study Notes — Organized by Topic

Notes are organized by **topic**, not by source book. Each file is the single canonical note for that subject, fed by all sources (MPLS Fundamentals book, SPCOR Official Cert Guide, labs). Add to the relevant topic file as you read/lab — never duplicate.

## Index

| Topic file | Covers | SPCOR Chapter |
|------------|--------|---------------|
| `routing-isis.md` | IS-IS | Ch 5 |
| `routing-ospf.md` | OSPF | Ch 6 |
| `routing-bgp.md` | BGP fundamentals, path selection, communities, PIC, add-path, best-external | Ch 7-8 |
| `multicast-mvpn.md` | Multicast, mVPN profiles | Ch 9 |
| `mpls-fundamentals.md` | Labels, forwarding, LDP, CEF, OAM, host-route optimization, troubleshooting | Ch 10 |
| `l2vpn-atom.md` | AToM point-to-point pseudowires (VPWS) | Ch 11 |
| `l2vpn-vpls-hvpls.md` | VPLS, BGP/LDP signaling, H-VPLS | Ch 11 |
| `carrier-ethernet.md` | Ethernet Ring Protection (G.8032 ERPS) | Ch 11 |
| `l3vpn.md` | MPLS VPN, RD/RT, Route Reflectors, 6PE/6VPE, inter-AS | Ch 12 |
| `evpn.md` | EVPN (control-plane MAC learning, multi-homing, IRB) | Ch 13 |
| `mpls-traffic-engineering.md` | RSVP-TE, tunnels, FRR | Ch 14 |
| `segment-routing.md` | SR-MPLS, SRv6, TI-LFA, SR-TE, Flex-Algo | Ch 15 |
| `security.md` | Control/management/data plane security | Ch 16-18 |
| `high-availability.md` | NSR, NSF, Graceful Restart, BFD | Ch 20 |
| `qos.md` | DiffServ, classification, marking, MPLS EXP | Ch 21 |
| `automation.md` | NETCONF, YANG, telemetry, model-driven | Ch 22 |

## Principle

- **One topic = one file.** Organize by subject, not by which book it came from.
- **All sources feed one note.** Book + lab + articles → the same topic file.
- File name describes the **subject**, never the **source**.

## Reading list (future books — not topic notes)

See the stub files in the parent directory: `bgp_design_and_implementation.md`, `evpn_vxlan_fabric_design.md`, `mpls_in_the_sdn_era.md`, `segment_routing_part_1_and_2.md` — these are books to buy after SPCOR.
