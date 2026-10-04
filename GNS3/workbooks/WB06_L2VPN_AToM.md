# WB06 — L2VPN AToM / VPWS (Point-to-Point Pseudowires)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Level:** CCIE SP
**Prerequisite:** WB00–WB04 complete — IGP + LDP transport in AS65200; targeted-LDP capable. (L3VPN/WB05 not strictly required but assumed in place.)
**Format:** Tasks only. No configurations provided.
**Reference:** See `00_topology_reference.md`. Red VLAN 100 attachment circuits: R27 on R16 f3/0, R28 on R14 f4/0, R29 on R1 g2/0.

> **Snapshot before you start:** `WB06-00-start`.

**Scope note:** This workbook builds AToM/VPWS **within AS65200** (R14 ↔ R16) carrying the Red VLAN 100 CEs R27 and R28. The R29/R1 (AS65100) cross-AS segment is handled in WB07 (VPLS cross-AS options).

---

## Section 1 — AToM / VPWS Point-to-Point Pseudowire

Goal: Build an Ethernet (VLAN 100) pseudowire between R14 and R16 across the AS65200 MPLS core so R28 (on R14) and R27 (on R16) are in the same Layer-2 segment. Pseudowire signaling is targeted-LDP.

- [ ] **Task 1.1 — Attachment circuits.** Prepare the ACs: R14 f4/0 (to R28) and R16 f3/0 (to R27) as the VLAN 100 customer-facing ports. Choose Ethernet port-mode xconnect vs. VLAN-mode (subinterface) and justify for VLAN 100 transport.
- [ ] **Task 1.2 — Pseudowire (xconnect).** Configure the AToM pseudowire between R14 and R16 using `xconnect <remote-PE-loopback> <VC-ID> encapsulation mpls`. Use a single agreed **VC-ID** on both ends (they must match). Peer to loopback0 addresses.
- [ ] **Task 1.3 — Control word.** Enable the control word on the pseudowire (via a pseudowire-class) and confirm both ends negotiate it. Explain what the control word protects against (fragment reordering / 0-ECMP mis-hash on frames that look like IP).
- [ ] **Task 1.4 — VC-ID & encapsulation consistency.** Confirm the VC-ID, the AF/encap (Ethernet vs Ethernet VLAN, VC type 4 vs 5), and the control-word setting all match on R14 and R16.

---

## Section 2 — AToM Verification

- [ ] **Task 2.1 — VC state.** `show mpls l2transport vc` and `show mpls l2transport vc detail` on R14 and R16 — VC status must be **UP**, with the imposed/disposed VC label and the LDP transport tunnel shown. Confirm control word is enabled on both ends.
- [ ] **Task 2.2 — Xconnect view.** `show xconnect all` — AC and PW segments both `UP`.
- [ ] **Task 2.3 — Targeted LDP.** Verify the targeted LDP session between R14 and R16 (`show mpls ldp neighbor` shows a targeted adjacency, `show mpls ldp discovery` shows the targeted hello).
- [ ] **Task 2.4 — L2 data-plane proof.** From R27, ping R28 across the pseudowire (same VLAN 100 subnet). Confirm MAC reachability — this is a flat L2 segment, so an ARP/ping between the two CEs should succeed. Capture `show arp`/neighbor on the CEs.

> **Snapshot:** `WB06-01-pw-up`.

---

## Section 3 — Pseudowire Redundancy

Goal: Add a backup pseudowire so the R27↔R28 service survives a primary PW/PE path failure.

- [ ] **Task 3.1 — Backup PW.** Configure `backup peer` under the xconnect (or a pseudowire-class redundancy group) so R14 has a primary PW to R16 and a backup PW to an alternate PE/path. Where no second remote PE exists for this pair, implement the backup as a diverse LSP/path to R16 (document the design choice given the topology).
- [ ] **Task 3.2 — Switchover behavior.** Set the backup delay / disable-delay and verify primary→backup switchover when the primary PW goes down (`show mpls l2transport vc` shows primary DOWN, backup ACTIVE).
- [ ] **Task 3.3 — Revert.** Verify the configured revert behavior when the primary recovers; measure/observe the traffic restoration.

> **Snapshot:** `WB06-02-pw-redundancy`.

---

## Section 4 — AToM Troubleshooting

Introduce one fault at a time, diagnose from symptoms, repair, and restore before the next.

- [ ] **Task 4.1 — VC-ID mismatch.** Change the VC-ID on one end. Observe the VC never comes up / mismatch indication (`show mpls l2transport vc detail`, targeted-LDP label-withdraw). Identify and fix.
- [ ] **Task 4.2 — MTU mismatch.** Set a different interface/MPLS MTU on one PE so the pseudowire MTU values disagree. Observe the VC staying DOWN with an MTU mismatch reason. Align MTU and recover.
- [ ] **Task 4.3 — AC down.** Shut the customer-facing AC on one PE. Confirm the local AC goes down and the remote PW reflects the AC failure (PW status signaling to the far end). Restore.
- [ ] **Task 4.4 — Broken transport LSP.** Break the LDP transport LSP between R14 and R16 (e.g., disable LDP on a core interface or filter the loopback label). Observe the VC drop because the tunnel label is gone even though targeted-LDP may still try. Diagnose via `show mpls forwarding-table` / `show ip cef` for the remote loopback, then restore.
- [ ] **Task 4.5 — Triage checklist.** Record the AToM fault order: AC state → PW signaling (VC-ID/CW/MTU match) → targeted-LDP session → transport LSP label. Map each fault above to a step.

> **Snapshot:** `WB06-03-complete`.

---

## Completion Checklist

- [ ] R14↔R16 AToM/VPWS pseudowire for VLAN 100 (R27↔R28), matching VC-ID, control word enabled both ends.
- [ ] VC UP on both PEs; `show xconnect all` AC+PW UP; targeted-LDP adjacency confirmed.
- [ ] L2 ping R27↔R28 succeeds across the pseudowire.
- [ ] Backup pseudowire configured; switchover and revert verified.
- [ ] Four faults (VC-ID, MTU, AC down, transport LSP) reproduced, diagnosed, repaired.
- [ ] Snapshots saved: `WB06-00-start`, `WB06-01-pw-up`, `WB06-02-pw-redundancy`, `WB06-03-complete`.

**Next:** WB07 — VPLS (multipoint L2VPN) extends the Red service to R29/R1 and introduces H-VPLS.
