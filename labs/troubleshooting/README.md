# Troubleshooting Labs

All troubleshooting labs use the **same 20-router GNS3 topology** — the same physical wiring, same router hardware (Cisco 7200, IOS 15.2), same telnet ports. Each lab loads a different configuration snapshot appropriate to its technology area.

**Total: 9 labs × 20 tickets = 180 troubleshooting tickets**
**Total points available: 585**

---

## How It Works

1. Load the appropriate GNS3 snapshot for the lab you want
2. Tell the AI which lab + ticket(s) you want injected
3. The AI connects to routers via telnet and configures the broken state
4. You receive ONLY the symptoms — then troubleshoot blind
5. You fix the issue and verify
6. Each lab is progressive (INE-style): fixing all 20 tickets results in a fully working topology

---

## Lab Overview

| TS Lab | Technology | Snapshot Required | Status |
|---|---|---|---|
| **ts_lab_1** | IS-IS as SP IGP | Topology D (IS-IS replaces OSPF) | ✅ Ready |
| **ts_lab_2** | MPLS SP Core (OSPF+LDP+L3VPN+TE) | Golden-state (Topology A) | ✅ Active — Tickets 4-20 injected |
| **ts_lab_3** | MP-BGP & Route Reflectors | Golden-state (Topology A) | ✅ Ready |
| **ts_lab_4** | L3VPN Advanced (Inter-AS, Hub-Spoke) | Golden-state + Inter-AS config | ✅ Ready |
| **ts_lab_5** | L2VPN & VPLS | L2VPN-Base (Topology B) | ✅ Ready |
| **ts_lab_6** | MPLS TE Advanced | Golden-state + TE tunnels | ✅ Ready |
| **ts_lab_7** | Segment Routing (SR-MPLS) | SR snapshot (IS-IS + SR, no LDP) | ✅ Ready |
| **ts_lab_8** | QoS in SP Networks | Golden-state + QoS policies | ✅ Ready |
| **ts_lab_9** | Multicast VPN (mVPN) | Golden-state + mVPN overlay | ✅ Ready |

---

## Topology (All Labs)

```
                    ┌─────────── NORTH CORE ───────────┐
                    │                                   │
    R1(CE)──R2(PE)──R3(P/RR)──R4(P)──R5(P)──R8(PE)──R9(CE)
    R12(CE)─┘       │    │              │    │    └──R11(CE)
                     R6(P)──────────────R7(P/RR)
                     │                   │
                    ┌┘                   └┐
                    │   SOUTH CORE        │
                   R13(P)──R14(P)──R15(P)──R16(P)
                    │       │              │
                   R17(PE)  R18(PE)       
                    │        │
                   R19(CE)  R20(CE)
```

| Role | Routers | ASN |
|---|---|---|
| PE | R2, R8, R17, R18 | 64512 |
| P (north) | R3, R4, R5, R6, R7 | — |
| P (south) | R13, R14, R15, R16 | — |
| RR | R3, R7 | 64512 |
| CE | R1, R9, R10, R11, R12, R19, R20 | Various |

---

## Difficulty Progression (per lab)

| Tickets | Level | Points Each |
|---|---|---|
| 1-3 | CCNP-SP ⭐⭐ | 2 |
| 4-6 | CCNP-SP ⭐⭐⭐ | 3 |
| 7-9 | CCNP/CCIE ⭐⭐⭐ | 2-3 |
| 10-12 | CCNP→CCIE ⭐⭐⭐ | 2-3 |
| 13-17 | CCIE-SP ⭐⭐⭐⭐ | 4 |
| 18-20 | CCIE-SP ⭐⭐⭐⭐⭐ | 5 |

**Passing per lab:** 75% (≈49/65 points)
**CCIE-ready per lab:** 90% (≈59/65 points)

---

## Workflow

```
You: "Inject ts_lab_3 tickets 1-6"
AI:  Connects to routers → configures base + 6 faults → "Ready. Start with Ticket 1."
You: Troubleshoot using show commands
You: Fix each issue
You: "Done with ticket 1 — verify"
AI:  Confirms fix or tells you what's still broken
You: Move to ticket 2...
```

---

## GNS3 Snapshots Needed

| Snapshot | Labs Using It |
|---|---|
| **Golden-State** (OSPF+LDP+L3VPN+RR+TE) | ts_lab_2, ts_lab_3, ts_lab_6, ts_lab_8, ts_lab_9 |
| **L2VPN-Base** (OSPF+LDP, no L3VPN on PE-CE) | ts_lab_5 |
| **IS-IS** (IS-IS replaces OSPF) | ts_lab_1 |
| **SR** (IS-IS+SR, no LDP) | ts_lab_7 |
| **Inter-AS** (Golden-state + R6↔R13 inter-AS VRF) | ts_lab_4 |
| **mVPN** (Golden-state + PIM-SM + MDT + mdt SAFI) | ts_lab_9 |
| **QoS** (Golden-state + DiffServ policies) | ts_lab_8 |

---

## Recommended Study Order

1. **ts_lab_2** — MPLS SP Core fundamentals (do this first — covers OSPF, LDP, L3VPN, TE basics)
2. **ts_lab_3** — BGP/RR deep-dive (after you're comfortable with VPN control plane)
3. **ts_lab_6** — TE Advanced (after you understand basic TE from ts_lab_2 tickets 8-9, 12)
4. **ts_lab_1** — IS-IS (when ready for IGP migration scenarios)
5. **ts_lab_4** — Inter-AS (after mastering single-domain L3VPN)
6. **ts_lab_5** — L2VPN/VPLS (different PE-CE model, separate snapshot)
7. **ts_lab_8** — QoS (overlay on working topology)
8. **ts_lab_9** — Multicast/mVPN (most complex overlay)
9. **ts_lab_7** — Segment Routing (modern SP, may need EVE-NG for full coverage)
