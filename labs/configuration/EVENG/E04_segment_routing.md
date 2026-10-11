# E04: Segment Routing (SR-MPLS + SR-TE)

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Garnet AS 65200 (IS-IS + SR-MPLS)
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 (IS-IS + SR prefix-SIDs 16011–16017) + E02 (VPNv4 on Garnet).

**End Goal:** Adjacency-SIDs, SR-TE policy Gar-R1→Gar-R2 via explicit segment list, TI-LFA per-prefix fast-reroute, BGP color steering of VPN routes, and ODN. SR runs on Garnet only.

---

## Section 1: SID Foundation (verify E01)

### Task 1: Confirm prefix-SIDs
1. Verify: `show isis segment-routing label table` — 16011–16017 present.
2. Verify: `show mpls forwarding` — SR labels in LFIB, no LDP.

### Task 2: Adjacency-SIDs
1. Confirm auto-allocated adjacency-SIDs: `show isis adjacency detail` / `show mpls forwarding labels <range>`.
2. Configure one **static/manual** adjacency-SID on a Gar-R3→Gar-R4 link under IS-IS interface (`adjacency-sid absolute <label>` or index).
3. Verify: the manual adj-SID appears and forwards over the intended link only.

---

## Section 2: SR-TE Policy

### Task 3: Explicit segment-list Gar-R1→Gar-R2
1. On Gar-R1: `segment-routing traffic-eng` → `segment-list VIA-Gar-R4-Gar-R5` with SIDs steering Gar-R1→Gar-R4→Gar-R5→Gar-R2 (mix of prefix + adjacency SIDs).
2. Define `policy Gar-R1-TO-Gar-R2` end-point 12.12.12.12, color 100, candidate-path with the explicit segment-list.
3. Verify: `show segment-routing traffic-eng policy` — policy UP (Admin/Oper).
4. Verify: `traceroute` / `show cef` shows label stack matching the segment list.

---

## Section 3: Fast Reroute

### Task 4: TI-LFA
1. Enable `fast-reroute per-prefix` + `ti-lfa` under IS-IS on Garnet core interfaces.
2. Verify: `show isis fast-reroute summary` — protected prefixes; backup paths computed.
3. Verify: `show route 12.12.12.12/32 detail` — backup next-hop / repair path present.
4. Test failover: shut a primary core link on the Gar-R1→Gar-R2 path; confirm sub-50ms style local repair (backup label stack installed pre-failure).

---

## Section 4: BGP Steering into SR-TE

### Task 5: Color VPN routes + auto-steer
1. On Gar-R2 (egress), attach `color 100` extcommunity to CUST_A VPNv4 routes via route-policy on export.
2. On Gar-R1, ensure the SR-TE policy (endpoint 12.12.12.12, color 100) exists; enable automated steering.
3. Verify: `show bgp vpnv4 unicast <prefix>` — color:100 attached.
4. Verify: VPN traffic Gar-R1→Gar-R2 for colored prefixes rides the SR-TE policy (`show cef vrf CUST_A <prefix>` label stack).

### Task 6: On-Demand Next-hop (ODN)
1. On Gar-R1: `segment-routing traffic-eng` → `on-demand color 100` with dynamic (or explicit) path preference.
2. Verify: importing a color:100 route auto-instantiates an ODN policy; `show segment-routing traffic-eng policy` shows an `on-demand` origin policy.
3. Verify: removing the colored route tears down the ODN policy.

---

## Section 5: Snapshot
1. Take GNS3 snapshot: **"E04-sr-te-working"**.

---

## Verification Checklist
```
[ ] Prefix-SIDs 16011–16017 confirmed (from E01)
[ ] Adjacency-SIDs: auto observed + one manual configured/verified
[ ] SR-TE explicit policy Gar-R1→Gar-R2 UP; label stack matches segment list
[ ] TI-LFA: per-prefix backups computed; failover verified on link shut
[ ] BGP color steering: color:100 on VPN routes; traffic auto-steered into policy
[ ] ODN auto-creates/tears-down policy on colored route import/withdraw
```
