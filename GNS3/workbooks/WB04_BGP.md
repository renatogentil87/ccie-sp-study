# WB04 — BGP (iBGP, Route Reflectors, eBGP Inter-AS, Path Manipulation, Advanced)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Prerequisite:** WB00–WB03 complete — IP addressing, IGP (IS-IS in AS65100/65300, OSPF in AS65200), and LDP operational in all three ASes. All loopbacks reachable within each AS.
**Format:** Tasks only. No configurations provided — you build the config to meet each task's intent.
**Reference:** See `00_topology_reference.md` for addressing, link map, loopbacks, and console ports.

> **Snapshot before you start:** `WB04-00-start` (confirms WB03 IGP+LDP baseline is clean).

---

## Section 1 — iBGP + Route Reflectors

Goal: Build the iBGP control plane in each AS. RRs reflect VPNv4 (and IPv4) so PEs/ASBRs do not need a full mesh. All iBGP sessions use loopback0 as update-source.

- [ ] **Task 1.1 — AS65100 RR cluster.** Configure R7 and R8 as route reflectors for AS65100. Peer R7 and R8 to each other (RR-to-RR, non-client). Make R1, R2, R5, R6 route-reflector-clients of both R7 and R8. Use `update-source Loopback0` on every session. Activate both the IPv4 unicast and VPNv4 address families (VPNv4 is needed for WB05).
- [ ] **Task 1.2 — AS65100 next-hop handling.** Decide where `next-hop-self` is required. The ASBRs (R5, R6) will learn eBGP routes in Section 2 — ensure the eBGP next-hop is resolvable by iBGP speakers. Document whether you use `next-hop-self` on the ASBRs toward the RRs or rely on redistributing the inter-AS link into the IGP. State your choice and why.
- [ ] **Task 1.3 — AS65200 RR.** Configure R19 as the single route reflector for AS65200. Make R12 (ASBR), R14, R15, R16 (PEs) route-reflector-clients. Activate IPv4 unicast and VPNv4. Use loopback0 as update-source. Note that R11/R13 are P routers — decide whether they need BGP at all (justify a BGP-free core vs. including them).
- [ ] **Task 1.4 — AS65300 RR.** Configure R30 as the route reflector for AS65300. Make R20, R21 (ASBR/PE), R22, R23 (PE) clients. Activate IPv4 unicast and VPNv4. loopback0 update-source.
- [ ] **Task 1.5 — Cluster-ID & loop prevention.** On AS65100 where you have two RRs (R7, R8), confirm the default behavior with distinct cluster-IDs vs. a shared cluster-ID. Explain the ORIGINATOR_ID and CLUSTER_LIST protections and verify them with `show bgp vpnv4 unicast all <prefix>`.
- [ ] **Task 1.6 — Verify iBGP.** Confirm all iBGP sessions reach Established in every AS (`show bgp ipvpnv4 unicast all summary`, `show ip bgp summary`). No VPN prefixes exist yet — the point here is a clean, fully-adjacent control plane.

---

## Section 2 — eBGP Inter-AS

Goal: Bring up eBGP between ASBRs over the inter-AS links. IPv4 unicast only for now (VPNv4 inter-AS is deferred to WB09). Peer on directly-connected interface addresses (not loopbacks) unless you deliberately configure multihop.

- [ ] **Task 2.1 — R5 ↔ R11 (AS65100 ↔ AS65200).** eBGP over `10.5.11.0/24`. IPv4 unicast AF. Peer on connected addresses.
- [ ] **Task 2.2 — R6 ↔ R12 (AS65100 ↔ AS65200).** eBGP over `10.6.12.0/24`. IPv4 unicast AF. This is the second, redundant 65100↔65200 link.
- [ ] **Task 2.3 — R6 ↔ R20 (AS65100 ↔ AS65300).** eBGP over `10.6.20.0/24`. IPv4 unicast AF.
- [ ] **Task 2.4 — R12 ↔ R21 (AS65200 ↔ AS65300).** eBGP over `20.12.21.0/24`. IPv4 unicast AF.
- [ ] **Task 2.5 — Advertise loopbacks.** Advertise each AS's PE/RR loopback prefixes (150.x.y.z/32) into eBGP so inter-AS reachability for BGP next-hops can be validated. Decide `network` statements vs. redistribution and justify.
- [ ] **Task 2.6 — eBGP next-hop into the AS.** Ensure prefixes learned via eBGP are usable inside the receiving AS: the eBGP next-hop must be reachable by iBGP clients. Apply `next-hop-self` on the ASBR toward the RR (or redistribute the inter-AS /24) and verify a PE sees a valid next-hop.
- [ ] **Task 2.7 — Verify Inter-AS.** From R1 (AS65100), confirm you can see AS65200 and AS65300 loopbacks in the BGP table with correct AS_PATH. Verify end-to-end reachability loopback-to-loopback across each AS boundary.

> **Snapshot:** `WB04-01-ibgp-ebgp-up` once all iBGP + eBGP sessions are Established and inter-AS loopbacks are reachable.

---

## Section 3 — BGP Path Manipulation

Goal: Control inbound/outbound path selection using the standard attribute toolkit. AS65100↔AS65200 is dual-linked (R5↔R11 and R6↔R12) — use that redundancy to demonstrate deterministic path preference.

- [ ] **Task 3.1 — LOCAL_PREF (primary/backup inter-AS link).** On AS65100, make the R5↔R11 link the preferred path to AS65200 prefixes and R6↔R12 the backup, using LOCAL_PREF applied inbound at the ASBRs. Verify best-path selection and that failover to R6↔R12 occurs when R5↔R11 is shut.
- [ ] **Task 3.2 — AS-PATH prepend.** On AS65200, influence AS65100's inbound path to AS65200 prefixes by prepending the AS on the less-preferred link (R6↔R12). Confirm AS65100 chooses the shorter AS_PATH via R5↔R11. Discuss why prepend is inbound-influence and how it interacts with the Task 3.1 LOCAL_PREF decision (LOCAL_PREF wins — demonstrate the precedence).
- [ ] **Task 3.3 — MED.** Between AS65100 and AS65200 (which share two links), advertise MED so the neighboring AS prefers one entry point. Enable `bgp always-compare-med` or `bgp deterministic-med` as appropriate and explain the difference. Verify MED only breaks ties when LOCAL_PREF and AS_PATH are equal.
- [ ] **Task 3.4 — Well-known communities.** Tag a selected AS65300 loopback with `no-export` on R20 and confirm it is not re-advertised past the receiving AS's eBGP boundary. Tag another with `no-advertise` and observe the difference.
- [ ] **Task 3.5 — Extended communities (preview).** On R12, set an arbitrary RT-style extended community on routes toward R21 and inspect it with `show bgp ... extended-community`. This primes the extended-community handling used by L3VPN in WB05.
- [ ] **Task 3.6 — Community-based policy.** Build a route-map that matches a custom community (e.g., `65100:100`) and sets LOCAL_PREF, applied at an ASBR. Prove the policy only affects tagged prefixes.
- [ ] **Task 3.7 — Verify path selection order.** For one prefix reachable via multiple paths, walk the BGP best-path algorithm (`show bgp <prefix>`) and confirm the winning attribute at each step. Document the decision.

> **Snapshot:** `WB04-02-path-manipulation`.

---

## Section 4 — BGP Advanced

- [ ] **Task 4.1 — Add-Path on RRs.** Enable BGP Additional Paths (advertise/receive) on the RRs (R7/R8 in AS65100, R19 in AS65200, R30 in AS65300) so clients receive more than the single best path. Verify a client receives multiple paths for a dual-homed prefix (`show bgp ... <prefix>` shows additional path-IDs). Explain how this improves convergence and complements Add-Path with BGP PIC.
- [ ] **Task 4.2 — BGP PIC (Prefix-Independent Convergence).** Enable BGP PIC Edge/Core so a precomputed backup path is installed in the RIB/FIB. Use the AS65100↔AS65200 dual link (Section 3) as the protected pair. Verify a backup path is present (`show ip cef <prefix> detail` / `show bgp <prefix>` repair path) and that failover is prefix-independent.
- [ ] **Task 4.3 — RT-Constraint (RTC).** Enable the `rtfilter` (route-target constraint) address family between the RRs and their clients so VPNv4 routes are only sent to PEs that import the matching RT. Verify RTC NLRI is exchanged (`show bgp rtfilter unicast all`). Note dependency: VRFs/RTs are created in WB05 — document the RTC design now and validate once WB05 VRFs exist.
- [ ] **Task 4.4 — Conditional route advertisement.** On an ASBR (e.g., R6), configure `advertise-map`/`non-exist-map` so a backup prefix is advertised to a neighbor only when a primary prefix disappears from the BGP table. Verify the advertise/withdraw triggers correctly by failing the primary.
- [ ] **Task 4.5 — Maximum-prefix on eBGP peers.** Apply `neighbor <ip> maximum-prefix` with a warning threshold and restart interval on each inter-AS eBGP session (Tasks 2.1–2.4). Verify the warning fires and the peer is torn down when the hard limit is exceeded (use a test advertisement or lower the limit to trigger).
- [ ] **Task 4.6 — Verify advanced features coexist.** Confirm Add-Path, PIC, and RTC are simultaneously active without breaking the Section 1–3 baseline (sessions Established, best-paths unchanged where no policy applies).

> **Snapshot:** `WB04-03-bgp-advanced`.

---

## Section 5 — BGP Troubleshooting

Introduce each fault, diagnose from symptoms, then repair. Work one fault at a time and restore before the next.

- [ ] **Task 5.1 — Peer stuck in Active/Idle.** Break one iBGP session so a neighbor sits in Active (e.g., wrong update-source, missing loopback reachability, or ACL on the peering interface). Diagnose with `show bgp ... summary`, `show tcp brief`, `debug ip bgp` and identify the exact cause. Fix and confirm Established.
- [ ] **Task 5.2 — Routes received but not installed.** Create a scenario where a prefix is in the BGP table but not the RIB (next-hop unreachable, or lost to a better IGP/static with lower AD, or `bgp suppress-inactive`). Use `show bgp <prefix>`, `show ip route <prefix>` to find why it is not best/installed. Resolve.
- [ ] **Task 5.3 — Next-hop unreachable.** Remove the fix from Task 2.6 (drop `next-hop-self` / un-redistribute the inter-AS link) so an iBGP client has an inaccessible eBGP next-hop. Show the `(inaccessible)` next-hop and the path marked invalid. Restore next-hop resolution and verify the path becomes best.
- [ ] **Task 5.4 — Document the triage order.** Write a short checklist: session state → TCP → next-hop reachability → best-path → RIB install. Confirm your three faults map to steps in that order.

> **Snapshot:** `WB04-04-complete` after all faults are repaired and the full BGP baseline is verified.

---

## Completion Checklist

- [ ] AS65100: R7+R8 RRs; R1,R2,R5,R6 clients; IPv4 + VPNv4 AF active; all iBGP Established.
- [ ] AS65200: R19 RR; R12,R14,R15,R16 clients; IPv4 + VPNv4 AF active.
- [ ] AS65300: R30 RR; R20,R21,R22,R23 clients; IPv4 + VPNv4 AF active.
- [ ] eBGP up on all four inter-AS links (R5↔R11, R6↔R12, R6↔R20, R12↔R21).
- [ ] Inter-AS loopback reachability verified; BGP next-hops resolvable in each AS.
- [ ] LOCAL_PREF primary/backup, AS-PATH prepend, MED, and communities demonstrated with precedence proven.
- [ ] Add-Path, BGP PIC, RT-Constraint, conditional advertisement, maximum-prefix all configured and verified.
- [ ] Three troubleshooting faults reproduced, diagnosed, and repaired.
- [ ] Snapshots saved: `WB04-00-start`, `WB04-01-ibgp-ebgp-up`, `WB04-02-path-manipulation`, `WB04-03-bgp-advanced`, `WB04-04-complete`.

**Next:** WB05 — L3VPN (VRFs, RD/RT, PE-CE routing) builds directly on the VPNv4 control plane and RTC established here.
