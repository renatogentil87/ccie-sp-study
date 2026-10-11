# E08: BGP Path Control

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Emerald AS 65100 (CE2 dual-homed E-R1+E-R2)
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02 complete (L3VPN VPNv4; CE2 dual-homed to E-R1 + E-R2 in CUST_A).

**End Goal:** Influence BGP path selection for CE2 dual-homing using weight, local-preference, AS-PATH prepend, MED; community tagging + RT-Constraint + Add-Path on RR E-R5; BGP PIC; hierarchical RR; graceful shutdown.

---

## Section 1: Path Selection (CE2 dual-homed via E-R1 + E-R2)

### Task 1: Weight (local, ingress-only)
1. On E-R1, set higher `weight` inbound for CE2 prefixes (route-policy on the CE2/VPNv4 path).
2. Verify: `show bgp vrf CUST_A <prefix>` — E-R1 path chosen locally (weight highest).

### Task 2: Local-preference (AS-wide)
1. Prefer E-R2 as exit for CUST_A: set higher `local-preference` on E-R2-learned paths (route-policy).
2. Verify: other Emerald PEs prefer E-R2 path; `show bgp vpnv4 unicast <prefix>` — LOCAL_PREF wins.

### Task 3: AS-PATH prepend (influence inbound)
1. On CE2 toward E-R2 (or E-R2 export), prepend AS to make one path less preferred.
2. Verify: neighbor selects the shorter AS_PATH path.

### Task 4: MED (tie-break to neighbor AS)
1. Set differing `med` on E-R1 vs E-R2 toward CE2's AS (65012).
2. Verify: lower MED preferred; confirm order (weight > LP > AS-PATH > MED) via `show bgp <prefix>`.

---

## Section 2: RR Policy on E-R4

### Task 5: Community tagging on RR (E-R5)
1. On E-R4, attach communities to CUST_A routes via route-policy (e.g., `65100:100`); optionally match to set LP downstream.
2. Verify: `show bgp vpnv4 unicast <prefix> detail` — community present; policy actions applied.

### Task 6: RT-Constraint
1. On E-R5: `address-family rtfilter unicast`; activate toward PE clients so RR only sends RTs each PE imports.
2. Verify: `show bgp rtfilter unicast` — RT membership advertised; unneeded VPNv4 not sent to PEs.

### Task 7: Add-Path on E-R4
1. Enable `additional-paths send/receive` + `advertise` for VPNv4 on RR E-R5 so both CE2 paths (via E-R1 and E-R2) are reflected.
2. Verify: clients receive multiple paths; `show bgp vpnv4 unicast <prefix>` on a PE shows >1 path.

---

## Section 3: Fast Convergence + Scale

### Task 8: BGP PIC
1. `additional-paths install` (PIC edge) on E-R1/E-R2 for CUST_A so a backup path is pre-installed.
2. Verify: `show cef vrf CUST_A <prefix> detail` — primary + backup next-hop; fast switch on primary failure.

### Task 9: Hierarchical RR
1. Model a second-tier RR (e.g., E-R4 as top-level RR, another router as cluster RR client-serving) with `cluster-id`.
2. Verify: `show bgp` cluster-list length increases per tier; no reflection loops (originator-id/cluster-id checks).

### Task 10: Graceful Shutdown
1. Apply BGP `graceful-shutdown` (GRACEFUL_SHUTDOWN community, LOCAL_PREF 0) on a PE↔CE2 session before maintenance.
2. Verify: traffic drains to the alternate PE with no loss before the session is torn down.

---

## Section 4: Snapshot
1. Take GNS3 snapshot: **"E08-bgp-path-control"**.

---

## Verification Checklist
```
[ ] Weight selects E-R1 path locally
[ ] Local-preference makes E-R2 the AS-wide exit
[ ] AS-PATH prepend deprioritizes one inbound path
[ ] MED tie-breaks toward AS 65012; decision order confirmed
[ ] Community tagging applied on RR E-R5
[ ] RT-Constraint limits VPNv4 sent to PE clients
[ ] Add-Path: PEs receive both CE2 paths via RR
[ ] BGP PIC installs backup next-hop; fast switchover verified
[ ] Hierarchical RR: cluster-list correct, no loops
[ ] Graceful shutdown drains traffic before teardown
```
