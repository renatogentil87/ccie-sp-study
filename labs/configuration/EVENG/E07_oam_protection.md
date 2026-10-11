# E07: MPLS OAM + Protection

**Platform:** IOS-XRv 9000 on GNS3/EC2
**Topology:** `00_topology_reference.md` — Emerald AS 65100 (LDP) + Garnet AS 65200 (SR)
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E01 complete (IS-IS + LDP on Emerald, IS-IS + SR on Garnet).

**End Goal:** Validate and protect the transport plane — LSP ping/traceroute, TTL propagation, BFD on IS-IS (all 3 SPs), LDP session protection + graceful restart + LDP-IGP sync (Emerald), TI-LFA (Garnet), and convergence measurement.

---

## Section 1: LSP OAM

### Task 1: LSP ping / traceroute
1. Emerald: `ping mpls ipv4 2.2.2.2/32` and `traceroute mpls ipv4 2.2.2.2/32` from E-R1.
2. Garnet: `ping mpls ipv4 12.12.12.12/32` (SR) from Gar-R1.
3. Verify: replies received; traceroute shows per-hop labels + return codes.

### Task 2: TTL propagation
1. Compare `mpls ip-ttl-propagate` enabled vs disabled behavior.
2. Verify: with propagate ON, `traceroute` reveals core hops; with it OFF (or `disable`), core is hidden (single hop).

---

## Section 2: BFD

### Task 3: BFD for IS-IS (all 3 SPs)
1. Enable `bfd` on IS-IS core interfaces (Emerald + Garnet); set min-interval + multiplier.
2. Verify: `show bfd session` — UP on core links.
3. Test: fail a link; confirm IS-IS drops via BFD faster than IGP hello timers.

---

## Section 3: LDP Resilience (Emerald)

### Task 4: LDP session protection
1. `mpls ldp / session protection` (optionally for specific peers).
2. Verify: `show mpls ldp neighbor detail` — session protection enabled; holdup on link flap.

### Task 5: LDP graceful restart
1. Enable `graceful-restart` under `mpls ldp`.
2. Verify: `show mpls ldp graceful-restart` — GR advertised; forwarding preserved across control-plane restart.

### Task 6: LDP-IGP sync
1. Enable `mpls ldp igp sync` (via `router isis` interface or LDP) on core links.
2. Verify: on LDP-down, IS-IS advertises max-metric for the link until LDP converges — prevents black-holing.
3. Test: bounce LDP on a link; confirm IGP metric behavior.

---

## Section 4: TI-LFA (Garnet)

### Task 7: TI-LFA fast-reroute
1. Enable `fast-reroute per-prefix ti-lfa` under IS-IS on Garnet core (if not from E04).
2. Verify: `show isis fast-reroute summary` — protected prefixes with backup paths.
3. Test: shut a primary core link; confirm local repair pre-installed.

---

## Section 5: Convergence Measurement

### Task 8: Measure convergence
1. Baseline: run continuous ping/traceroute PE↔PE; record loss on link failure WITHOUT protection.
2. With BFD + FRR/TI-LFA enabled: repeat and record improved loss/recovery time.
3. Compare Emerald (LDP + IGP-sync) vs Garnet (SR + TI-LFA) recovery profiles.
4. Verify: documented before/after convergence numbers.

---

## Section 6: Snapshot
1. Take GNS3 snapshot: **"E07-oam-protection"**.

---

## Verification Checklist
```
[ ] LSP ping/traceroute OK on Emerald (LDP) + Garnet (SR)
[ ] TTL propagate ON reveals hops; OFF hides core
[ ] BFD sessions UP on IS-IS (all 3 SPs); faster failure detection verified
[ ] Emerald: LDP session protection enabled
[ ] Emerald: LDP graceful restart advertised; forwarding preserved
[ ] Emerald: LDP-IGP sync sets max-metric until LDP up (no black-hole)
[ ] Garnet: TI-LFA backups computed; local repair verified
[ ] Convergence measured before/after protection; results documented
```
