# WB07 — VPLS (Multipoint L2VPN, Cross-AS, H-VPLS)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Prerequisite:** WB00–WB04 complete (IGP + LDP in all ASes); WB06 complete (AToM/VPWS fundamentals, targeted-LDP).
**Format:** Tasks only. No configurations provided.
**Reference:** See `00_topology_reference.md`. Red VLAN 100 CEs: R27 on R16 f3/0, R28 on R14 f4/0, R29 on R1 g2/0.

> **Snapshot before you start:** `WB07-00-start`.

**IOS caveat:** On the 7200 / IOS 15.2(4)M11, confirm the available VPLS syntax (legacy `l2 vfi ... manual` + `vfi` under bridge, vs. modern `l2vpn vfi context` / `bridge-domain`). Use whichever the image supports and keep terminology consistent. Cross-AS native VPLS is often unsupported on this platform — Section 2 provides a fallback.

---

## Section 1 — VPLS within AS65200

Goal: Build an LDP-signaled (martini) full-mesh VPLS instance across AS65200 so R27 (on R16) and R28 (on R14) share one broadcast domain for VLAN 100. VPLS = multipoint; a VFI meshes pseudowires to every peer PE.

- [ ] **Task 1.1 — VFI definition.** Define a VPLS VFI on R14 and R16 with a shared **VPN-ID** and list the peer PE loopbacks (each PE points at the other). Associate the VFI with VLAN 100.
- [ ] **Task 1.2 — Bridge-domain / instance.** Bind the customer VLAN 100 attachment circuits (R14 f4/0 → R28, R16 f3/0 → R27) and the VFI into the same bridge-domain / VLAN so AC traffic and the PW mesh share one L2 forwarding instance.
- [ ] **Task 1.3 — Split-horizon.** Confirm VPLS split-horizon on the full-mesh PW core (a frame received on one core PW is never forwarded to another core PW) to prevent L2 loops without STP. Explain why this makes a full mesh loop-free and why H-VPLS spokes need special handling (Section 3).
- [ ] **Task 1.4 — MAC learning.** Verify dynamic MAC learning in the VFI/bridge-domain from R27 and R28 traffic.

> **Snapshot:** `WB07-01-vpls-as65200`.

---

## Section 2 — Extend to Cross-AS (R29 / R1 in AS65100)

Goal: Give R29 (on R1, AS65100) Layer-2 reachability into the Red VLAN 100 domain with R27/R28 in AS65200. Evaluate options, then implement the one the IOS supports.

- [ ] **Task 2.1 — Option assessment.** Document the candidate approaches and their IOS 15.2(4)M11 feasibility:
  - (a) **Native inter-AS VPLS** (VFI peers across the AS boundary) — requires inter-AS LDP/BGP-AD support; likely unsupported on 7200.
  - (b) **Multi-segment pseudowire (MS-PW)** stitched at the ASBR — verify switching-PW support.
  - (c) **AToM cross-AS fallback:** a point-to-point PW R1↔R14 (stitched/extended across the inter-AS link via the ASBRs), with R14 bridging that PW into the AS65200 VPLS VFI. This is the recommended fallback if (a)/(b) are unsupported.
- [ ] **Task 2.2 — Confirm support.** Test the chosen option's syntax on the image (`show l2vpn ...` / `show mpls l2transport vc`). If native inter-AS VPLS fails to configure or stay up, fall back to option (c).
- [ ] **Task 2.3 — Implement cross-AS reachability.** Bring R29/R1 into the Red domain using the supported method. For the fallback: build the R1↔R14 pseudowire (across AS65100↔AS65200 transport — note this depends on inter-AS LSP/label exchange, which may itself be a WB09 Inter-AS item; document the dependency), then add that PW as a bridged member of R14's VPLS VFI so R29 shares the broadcast domain with R27/R28.
- [ ] **Task 2.4 — Verify cross-AS L2.** From R29, reach R27 and R28 at Layer 2 (same VLAN 100 subnet). If the inter-AS transport is not yet available, record this as blocked on WB09 and verify as far as the AS boundary.

> **Snapshot:** `WB07-02-vpls-crossas` (or `WB07-02-vpls-crossas-blocked` if gated on WB09 inter-AS transport).

---

## Section 3 — H-VPLS (Hierarchical VPLS)

Goal: Convert the design to hub-and-spoke to reduce the core full mesh. N-PEs form the meshed core; a U-PE connects via a single spoke PW to an N-PE.

- [ ] **Task 3.1 — N-PE / U-PE roles.** Designate R14 and R16 as **N-PEs** (meshed core VFI between them) and R1 as a **U-PE** (spoke). The U-PE has a single spoke pseudowire into an N-PE instead of a full mesh.
- [ ] **Task 3.2 — Spoke PW.** Configure the R1 (U-PE) spoke pseudowire to its N-PE and add it to the N-PE's VFI/bridge-domain as a spoke (not a core-mesh) member.
- [ ] **Task 3.3 — Split-horizon on the spoke.** Confirm the N-PE treats the spoke PW outside the core split-horizon group so spoke traffic can be forwarded to the core mesh (unlike core-to-core). Explain how H-VPLS avoids loops while still reaching the spoke.
- [ ] **Task 3.4 — Scalability rationale.** Document why H-VPLS reduces PW count and signaling vs. a flat N-PE full mesh (n·(n−1)/2 core PWs plus one spoke each), and the tradeoff (U-PE single point of failure unless dual-homed).

> **Snapshot:** `WB07-03-hvpls`.

---

## Section 4 — VPLS Verification

- [ ] **Task 4.1 — Bridge-domain state.** `show l2vpn bridge-domain` (or `show bridge-domain` / `show vfi` per image) — VFI UP, member ACs and PWs listed, VLAN 100 bound.
- [ ] **Task 4.2 — MAC learning.** Show the dynamic MAC table for the VPLS instance; confirm R27/R28 (and R29 if cross-AS up) MACs are learned against the correct AC/PW.
- [ ] **Task 4.3 — BUM flooding.** Verify Broadcast/Unknown-unicast/Multicast is flooded to all VFI members except the ingress (and respecting split-horizon). Generate an unknown-unicast/broadcast and confirm the flood set.
- [ ] **Task 4.4 — Split-horizon proof.** Demonstrate that a frame arriving on one core PW is not forwarded to another core PW (core-to-core blocked), while AC and spoke traffic floods normally.

> **Snapshot:** `WB07-04-vpls-verified`.

---

## Section 5 — VPLS Troubleshooting

Introduce one fault at a time; diagnose, repair, restore.

- [ ] **Task 5.1 — VPN-ID / VFI peer mismatch.** Misconfigure a VFI peer loopback or VPN-ID; observe the PW not joining the VFI. Diagnose with `show vfi` / `show mpls l2transport vc`. Fix.
- [ ] **Task 5.2 — MAC flooding loop risk.** Create a condition that would loop (e.g., spoke incorrectly placed in the core split-horizon group, or two ACs bridged externally). Detect via MAC flaps / high flooding. Correct the split-horizon membership.
- [ ] **Task 5.3 — AC/VLAN mismatch.** Put an AC in the wrong VLAN/bridge-domain so a CE cannot reach the domain. Diagnose the missing MAC learning and fix the AC-to-VFI binding.
- [ ] **Task 5.4 — Transport LSP failure.** Break the LDP transport to one VFI peer; show that PW dropping and the resulting partitioned broadcast domain. Restore and confirm MAC relearning.
- [ ] **Task 5.5 — Triage checklist.** Record VPLS fault order: AC/VLAN binding → VFI peer/VPN-ID match → targeted-LDP PW signaling → transport LSP → split-horizon correctness. Map each fault above.

> **Snapshot:** `WB07-05-complete`.

---

## Completion Checklist

- [ ] LDP-signaled VPLS VFI (shared VPN-ID) between R14 and R16 for VLAN 100 (R27, R28); split-horizon and MAC learning verified.
- [ ] Cross-AS option assessed on IOS 15.2(4)M11; chosen method implemented (native/MS-PW or AToM fallback) or documented as blocked on WB09 inter-AS transport.
- [ ] H-VPLS: R14/R16 as N-PEs, R1 as U-PE spoke PW; spoke split-horizon handling correct; scalability rationale recorded.
- [ ] `show l2vpn bridge-domain` healthy; BUM flooding and split-horizon proofs captured.
- [ ] Five faults reproduced, diagnosed, repaired.
- [ ] Snapshots saved: `WB07-00-start`, `WB07-01-vpls-as65200`, `WB07-02-vpls-crossas[-blocked]`, `WB07-03-hvpls`, `WB07-04-vpls-verified`, `WB07-05-complete`.

**Next:** WB08+ — (QoS / MPLS-TE / Inter-AS Options per the study plan). Cross-AS VPN and L2VPN stitching is completed in WB09.
