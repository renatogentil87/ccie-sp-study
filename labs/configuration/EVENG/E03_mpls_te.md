# E03: MPLS Traffic Engineering (RSVP-TE)

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Emerald AS 65100 (IS-IS + LDP)
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 complete (IS-IS L2 + LDP on Emerald).

**End Goal:** RSVP-TE tunnels E-R1→E-R2 with explicit paths, FRR facility backup, autoroute, affinity, and auto-bandwidth. TE runs on Emerald only (LDP domain).

---

## Section 1: TE Infrastructure

### Task 1: Enable MPLS-TE + RSVP on Emerald core
1. Enable `mpls traffic-eng` on E-R1, E-R2, E-R3, E-R4, E-R6 core interfaces.
2. Enable `rsvp` on the same core interfaces; set per-interface reservable bandwidth.
3. Add `mpls traffic-eng` under `router isis CORE` (level-2) + `metric-style wide` (already set in E01).
4. Verify: `show mpls traffic-eng link-management interfaces` — TE-enabled links UP with bandwidth.
5. Verify: `show rsvp interface` — RSVP active on core links.
6. Verify: `show mpls traffic-eng topology` — full TE topology learned via IS-IS.

---

## Section 2: Basic + Explicit-Path Tunnels

### Task 2: Dynamic tunnel E-R1→E-R2
1. On E-R1: `interface tunnel-te1`, destination 2.2.2.2, `path-option 1 dynamic`.
2. Verify: `show mpls traffic-eng tunnels tunnel-te1` — state UP, dynamically computed path.

### Task 3: Explicit path via E-R4
1. Define `explicit-path name VIA-E-R4` listing next-hops E-R1→E-R4→E-R2.
2. Add `path-option 5 explicit name VIA-E-R4` (preferred), keep dynamic as fallback (higher index).
3. Verify: tunnel takes E-R4 path; `show mpls traffic-eng tunnels tunnel-te1 detail` — RECORDED ROUTE via E-R4.
4. Define second `explicit-path name VIA-E-R3` (E-R1→E-R3→E-R2) for a backup/comparison tunnel-te2.

---

## Section 3: Traffic Steering

### Task 4: Autoroute announce
1. On tunnel-te1: `autoroute announce`.
2. Verify: `show route 2.2.2.2/32` on E-R1 — next-hop is tunnel-te1 (IGP shortcut).
3. Verify: LDP-over-TE or plain TE forwarding to E-R2 loopback uses the tunnel.

### Task 5: Affinity / attribute-flags
1. Tag a core link with `attribute-flags 0x1` (admin-group) via `mpls traffic-eng`.
2. On tunnel-te2: set `affinity 0x0 mask 0x1` to exclude that link.
3. Verify: computed path avoids the tagged link; `show mpls traffic-eng tunnels tunnel-te2 detail`.

---

## Section 4: Protection

### Task 6: FRR facility backup
1. Enable `fast-reroute` on tunnel-te1.
2. On the PLR (E-R4 or E-R1), build a next-hop/next-next-hop backup tunnel protecting the primary link.
3. `mpls traffic-eng` on backup interfaces; associate backup tunnel to protected interface.
4. Verify: `show mpls traffic-eng fast-reroute database` — primary marked "Ready".
5. Test: shut the protected core link; confirm sub-second local repair; `show mpls traffic-eng tunnels backup`.

---

## Section 5: Auto-Bandwidth

### Task 7: Auto-bandwidth on tunnel-te1
1. `auto-bw` with collection frequency + min/max bandwidth.
2. Generate traffic E-R1→E-R2; wait for an adjustment interval.
3. Verify: `show mpls traffic-eng tunnels tunnel-te1 auto-bw` — sampled rate and applied bandwidth change.

---

## Section 6: Snapshot
1. Take GNS3 snapshot: **"E03-mpls-te-working"**.

---

## Verification Checklist
```
[ ] TE + RSVP enabled on all Emerald core links; TE topology complete
[ ] Dynamic tunnel E-R1→E-R2 UP
[ ] Explicit-path tunnel via E-R4 UP (RRO confirms path); alt via E-R3 defined
[ ] Autoroute announce: E-R1 uses tunnel for 2.2.2.2/32
[ ] Affinity excludes tagged link on tunnel-te2
[ ] FRR facility backup Ready; local repair on link failure verified
[ ] Auto-bandwidth adjusts reserved BW from measured traffic
```
