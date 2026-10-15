# WB20 — EVPN

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** EVPN in AS 65200 (OSPF core) — PEs **R14** and **R16** serving the **Red** customer
**Level:** CCIE SP
**Prerequisite:** WB16/WB17 SR concepts (for EVPN-over-SRv6), WB06/WB07 L2VPN background (VPWS/VPLS/AToM for comparison)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab.
> EVPN is **not** supported on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: verify each section before advancing.
> **Snapshot** before/after each section.

> **Reference (AS 65200):** PEs **R14** (150.2.14.14) and **R16** (150.2.16.16); P routers R11/R13; RR **R19** (150.2.19.19).
> **Red customer** L2 attachments (from `00_topology_reference.md`): R16 f3/0 ↔ R27, R14 f4/0 ↔ R28, both VLAN 100.
> BGP AS 65200, RR = R19. EVPN uses BGP AF **l2vpn evpn**.

---

## Pre-flight

- [ ] Confirm AS 65200 core (OSPF + transport LSPs: LDP or SR) is UP between R14, R16, R19
- [ ] Confirm iBGP sessions R14↔R19 and R16↔R19 are UP
- [ ] Confirm the Red customer AC interfaces (R14↔R28, R16↔R27, VLAN 100) exist
- [ ] **Snapshot:** `WB20-00-baseline`

---

## Section 1 — Enable BGP EVPN Address-Family [3 points]

- [ ] **Task 1.1 [1 point]** — Globally enable EVPN on R14 and R16 (`evpn` process) and R19 as the RR.
- [ ] **Task 1.2 [1 point]** — Add the **l2vpn evpn** address-family to BGP on R14, R16 (to R19) and on R19 reflect it (`route-reflector-client`).
- [ ] **Task 1.3 [1 point]** — Confirm the EVPN BGP sessions come up for the l2vpn evpn AF.
- [ ] **Task 1.4 [0 points — verification]** — Confirm no routes yet (no services configured).

**Verify / Snapshot**
- [ ] `show bgp l2vpn evpn summary` — neighbors UP for the AF
- [ ] `show evpn summary` — EVPN enabled, BGP RID
- [ ] **Snapshot:** `WB20-01-evpn-af`

---

## Section 2 — EVPN-VPWS (point-to-point) [5 points]

**EVI 400, AC VLAN 12, point-to-point** between R14 and R16 for Red.

- [ ] **Task 2.1 [2 points]** — Under `l2vpn` → `xconnect group <name>` → `p2p <name>`, define an **EVPN-VPWS** service: local AC (sub-interface on the Red-facing port, e.g. dot1q 12) + `neighbor evpn evi 400 target <remote-ac-id> source <local-ac-id>`. Configure the mirror on R16.
- [ ] **Task 2.2 [2 points]** — Assign the **AC = 12** identifiers (source/target) consistently between R14 and R16 so the two single-homed endpoints stitch into one pseudowire via EVPN route-type 1 (per-EVI AD).
- [ ] **Task 2.3 [1 point]** — Confirm the EVPN-VPWS is UP and forwarding Red traffic R28↔R27 point-to-point.
- [ ] **Task 2.4 [0 points — verification]** — Confirm the per-EVI Ethernet A-D (RT-1) routes are exchanged for EVI 400.

**Verify / Snapshot**
- [ ] `show l2vpn xconnect group <name>` — state UP
- [ ] `show bgp l2vpn evpn` — RT-1 (Ethernet A-D per EVI) for EVI 400
- [ ] `show evpn evi vpn-id 400 detail`
- [ ] End-to-end: Red CE R28 ↔ R27 reachability
- [ ] **Snapshot:** `WB20-02-evpn-vpws`

---

## Section 3 — EVPN Multipoint (ELAN) [5 points]

Bridge-domain + EVI with MAC learning (multipoint L2 for Red).

- [ ] **Task 3.1 [2 points]** — On R14 and R16 create a **bridge-domain** (e.g. `l2vpn bridge group RED bridge-domain RED`) with the Red AC (VLAN 100) as a member and associate an **EVI** (e.g. evi 100) via `member evpn-evi`.
- [ ] **Task 3.2 [1 point]** — Confirm **MAC learning** over EVPN: local MACs advertised as **route-type 2 (MAC/IP)**, remote MACs installed from BGP.
- [ ] **Task 3.3 [1 point]** — Confirm **inclusive multicast (route-type 3)** is used for BUM flooding between R14 and R16.
- [ ] **Task 3.4 [1 point]** — Verify end-to-end multipoint bridging for Red across R14↔R16.
- [ ] **Task 3.5 [0 points — verification]** — Inspect the EVPN MAC table and compare to the bridge-domain MAC table.

**Verify / Snapshot**
- [ ] `show l2vpn bridge-domain group RED detail` — ACs + EVI, MAC count
- [ ] `show bgp l2vpn evpn` — RT-2 (MAC/IP) and RT-3 (IMET) present
- [ ] `show evpn evi vpn-id 100 mac` — learned MACs (local vs remote)
- [ ] **Snapshot:** `WB20-03-evpn-elan`

---

## Section 4 — ESI Multi-homing [4 points]

Multi-home a Red CE to both R14 and R16 using an **Ethernet Segment Identifier**.

- [ ] **Task 4.1 [2 points]** — Configure an **ESI** on the shared Red Ethernet segment on both R14 and R16 (`evpn` → `interface <AC>` → `ethernet-segment identifier type 0 <esi-value>`). Use the SAME ESI on both PEs.
- [ ] **Task 4.2 [1 point]** — Choose a redundancy mode: **all-active** (if the CE LAGs to both) or **single-active**. Confirm **DF (designated forwarder) election** (route-type 4 — Ethernet Segment route).
- [ ] **Task 4.3 [1 point]** — Confirm **aliasing / mass-withdraw** behavior: remote PEs load-share to the ES via RT-1 (per-ES A-D) and converge fast when one PE's link to the segment fails.
- [ ] **Task 4.4 [0 points — verification]** — Confirm DF role per PE and split-horizon (local-bias) prevents echo.

**Verify / Snapshot**
- [ ] `show evpn ethernet-segment detail` — ESI, mode, DF election result
- [ ] `show bgp l2vpn evpn` — RT-4 (ES route) + RT-1 (per-ES A-D)
- [ ] Fail one PE-CE link → confirm fast convergence (mass-withdraw)
- [ ] **Snapshot:** `WB20-04-esi`

---

## Section 5 — Route Types 1-5 + EVPN over SRv6 [5 points]

- [ ] **Task 5.1 [2 points]** — In the attempt notes, **document each EVPN route type** and show it in your lab:
  - RT-1 Ethernet Auto-Discovery (per-ES + per-EVI)
  - RT-2 MAC/IP Advertisement
  - RT-3 Inclusive Multicast Ethernet Tag (BUM)
  - RT-4 Ethernet Segment (DF election)
  - RT-5 IP Prefix (EVPN L3 / IP-VRF-to-IP-VRF)
- [ ] **Task 5.2 [1 point]** — Configure an **EVPN RT-5 (IP Prefix)** service for a Red IP subnet so an L3 prefix is advertised through EVPN (integrated routing & bridging / symmetric IRB concept).
- [ ] **Task 5.3 [2 points]** — Convert the EVPN services to run over **SRv6** transport: assign an **SRv6 locator** to the EVPN services on R14/R16 (`evpn` + `segment-routing srv6 locator <name>`) so EVPN uses SRv6 SIDs instead of MPLS labels for the data plane.
- [ ] **Task 5.4 [0 points — verification]** — Confirm RT-2/RT-5 carry SRv6 SIDs (not MPLS) when SRv6 transport is enabled.

**Verify / Snapshot**
- [ ] `show bgp l2vpn evpn route-type 1` … `route-type 5` — each type present
- [ ] `show bgp l2vpn evpn route-type 5 detail` — SRv6 SID attached
- [ ] `show evpn evi detail` — SRv6 locator association
- [ ] **Snapshot:** `WB20-05-rt-and-srv6`

---

## Section 6 — Comparison with VPLS and AToM [3 points]

- [ ] **Task 6.1 [2 points]** — Write a structured comparison (in attempt notes) of **EVPN vs VPLS vs AToM/VPWS** covering: control plane (BGP MAC learning vs flood-and-learn vs static/targeted-LDP), multi-homing & all-active (EVPN RT-1/RT-4 vs VPLS none/limited), MAC mobility, BUM handling, and provisioning. Reference your WB06 (AToM) and WB07 (VPLS) results.
- [ ] **Task 6.2 [1 point]** — Describe the migration story: how EVPN-VPWS replaces AToM VPWS, and EVPN-ELAN replaces VPLS, on modern platforms. Note what maps 1:1 and what changes operationally.
- [ ] **Snapshot:** `WB20-06-comparison`

---

## Section 7 — Troubleshooting Drills [5 points]

- [ ] **Drill 7.1 — EVI mismatch [2 points]:** configure mismatched EVI numbers on R14 vs R16 for the VPWS. Confirm the xconnect stays down (no RT-1 match). Diagnose and fix.
- [ ] **Drill 7.2 — Duplicate / wrong ESI [2 points]:** misconfigure the ESI (different value per PE, or type/value typo). Confirm DF election / aliasing breaks. Diagnose with `show evpn ethernet-segment` and fix.
- [ ] **Drill 7.3 — No RT-3 / BUM flooding [1 point]:** break the IMET (RT-3) exchange (RT import/export mismatch). Confirm broadcast/unknown-unicast fails while known-unicast works. Diagnose and restore.
- [ ] **Snapshot:** `WB20-07-tshoot`

**Required show commands (fluency):**
- [ ] `show bgp l2vpn evpn [summary | route-type <1-5>]`
- [ ] `show evpn evi [detail | mac]`
- [ ] `show evpn ethernet-segment detail`
- [ ] `show l2vpn xconnect` / `show l2vpn bridge-domain`

---

## Section 8 — EVPN-IRB (Integrated Routing and Bridging) [5 points]

- [ ] **Task 8.1 [2 points]** — Configure **EVPN-IRB** on R14 and R16. Create a **bridge-domain** with a **BVI (Bridge-group Virtual Interface)** acting as the IP gateway (`interface BVI<n>` + `routing-interface BVI<n>` under the bridge-domain). The Red customer gets both **L2 switching** and **L3 routing** through the same EVPN instance (symmetric IRB with an IP-VRF).
- [ ] **Task 8.2 [2 points]** — Configure a **distributed anycast gateway** — the SAME BVI IP address and SAME virtual MAC on both R14 and R16 (`host-routing` + anycast gateway MAC) so first-hop routing is **active-active**; a host keeps the same default gateway regardless of which PE it attaches to.
- [ ] **Task 8.3 [1 point]** — Verify: `show evpn evi`, `show l2vpn bridge-domain`, `show bgp l2vpn evpn route-type 2` (MAC/IP advertisement **with** the host IP), `show bgp l2vpn evpn route-type 5` (IP-Prefix route for inter-subnet routing).

**Verify / Snapshot**
- [ ] `show l2vpn bridge-domain detail` — BVI as routed interface
- [ ] `show evpn evi detail` — IRB / anycast gateway MAC
- [ ] `show bgp l2vpn evpn route-type 2` — MAC/IP with IP; `route-type 5` — IP prefix
- [ ] Host on R14 pings host on a different subnet via R16 (inter-subnet)
- [ ] **Snapshot:** `WB20-08-evpn-irb`

---

## Section 9 — E-Tree (Hub-and-Spoke L2VPN) [3 points]

- [ ] **Task 9.1 [2 points]** — Configure an **E-Tree** service using EVPN. **Root (hub)** sites may communicate with all sites; **Leaf (spoke)** sites may communicate **only with root**, not with other leaves. Assign the **AC role (root / leaf)** on each attachment circuit (`etree` + `leaf`/root designation) and use the EVPN **split-horizon / leaf-label** mechanism to block leaf-to-leaf.
- [ ] **Task 9.2 [1 point]** — Verify leaf-to-leaf traffic is **blocked** while root-to-leaf works.

**Verify / Snapshot**
- [ ] `show evpn evi detail` — E-Tree root/leaf roles
- [ ] Root CE ↔ Leaf CE: reachable
- [ ] Leaf CE ↔ Leaf CE: blocked
- [ ] **Snapshot:** `WB20-09-etree`

---

## Section 10 — L2VPN Auto-Steering into SR Policy [3 points]

- [ ] **Task 10.1 [2 points]** — Configure an **EVPN-VPWS** service that steers traffic into an **SR-TE policy** using a BGP **color** community. The egress PE attaches a **color** extended community to the EVPN routes; the ingress PE **auto-steers** matching traffic into the SR policy whose (endpoint, color) matches (on-demand next-hop / ODN or a pre-configured color→policy mapping).
- [ ] **Task 10.2 [1 point]** — Verify: `show segment-routing traffic-eng policy` shows the L2VPN traffic resolving over the SR policy (binding-SID / forwarding over the TE path).

**Verify / Snapshot**
- [ ] `show bgp l2vpn evpn` — color extended community on the service route
- [ ] `show segment-routing traffic-eng policy color <c>` — policy UP, used by L2VPN
- [ ] `show l2vpn xconnect detail` — forwarding via SR policy / binding-SID
- [ ] **Snapshot:** `WB20-10-l2vpn-srte`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Enable BGP EVPN AF | 3 | Standard: evpn + l2vpn evpn AF |
| Section 2: EVPN-VPWS (EVI 400, AC 12) | 5 | CCIE-level: p2p xconnect over EVPN |
| Section 3: EVPN multipoint (ELAN) | 5 | CCIE-level: BD/EVI, RT-2/RT-3 MAC learning |
| Section 4: ESI multi-homing | 4 | CCIE-level: ESI, DF election, aliasing |
| Section 5: Route types 1-5 + EVPN/SRv6 | 5 | CCIE-level: RT-5 + SRv6 transport |
| Section 6: Compare VPLS / AToM | 3 | Conceptual: control-plane comparison |
| Section 7: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
| Section 8: EVPN-IRB | 5 | CCIE-level: BVI + distributed anycast GW |
| Section 9: E-Tree (hub-and-spoke) | 3 | CCIE-level: root/leaf split-horizon |
| Section 10: L2VPN auto-steering into SR policy | 3 | CCIE-level: color community → SR-TE |
| **Total** | **41** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
36-41 points:  CCIE ready for this topic
28-35 points:  Strong, minor gaps to review
18-27 points:  Needs more practice
Below 18:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] BGP l2vpn evpn AF UP across R14, R16, R19
- [ ] EVPN-VPWS (EVI 400, AC 12) point-to-point UP for Red
- [ ] EVPN multipoint (ELAN) with RT-2 MAC learning + RT-3 BUM
- [ ] ESI multi-homing with DF election + aliasing
- [ ] Route types 1-5 observed; EVPN-over-SRv6 with locator
- [ ] Written EVPN vs VPLS vs AToM comparison + migration story
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] EVPN-IRB with BVI gateway + distributed anycast gateway (active-active)
- [ ] E-Tree hub-and-spoke: root-to-leaf works, leaf-to-leaf blocked
- [ ] EVPN-VPWS auto-steered into an SR-TE policy via color community
- [ ] **Final snapshot:** `WB20-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Enable EVPN AF** | **3** | | |
| 1.1 Enable evpn | 1 | | |
| 1.2 l2vpn evpn AF | 1 | | |
| 1.3 Sessions UP | 1 | | |
| **Section 2: EVPN-VPWS** | **5** | | |
| 2.1 p2p EVPN xconnect | 2 | | |
| 2.2 AC 12 identifiers | 2 | | |
| 2.3 UP + forwarding | 1 | | |
| **Section 3: EVPN multipoint** | **5** | | |
| 3.1 BD + EVI | 2 | | |
| 3.2 RT-2 MAC learning | 1 | | |
| 3.3 RT-3 BUM | 1 | | |
| 3.4 End-to-end bridging | 1 | | |
| **Section 4: ESI multi-homing** | **4** | | |
| 4.1 ESI both PEs | 2 | | |
| 4.2 Mode + DF election | 1 | | |
| 4.3 Aliasing/mass-withdraw | 1 | | |
| **Section 5: RT 1-5 + SRv6** | **5** | | |
| 5.1 Document RT 1-5 | 2 | | |
| 5.2 RT-5 IP prefix | 1 | | |
| 5.3 EVPN over SRv6 | 2 | | |
| **Section 6: Comparison** | **3** | | |
| 6.1 EVPN vs VPLS vs AToM | 2 | | |
| 6.2 Migration story | 1 | | |
| **Section 7: Troubleshooting** | **5** | | |
| 7.1 EVI mismatch | 2 | | |
| 7.2 Wrong ESI | 2 | | |
| 7.3 No RT-3/BUM | 1 | | |
| **Section 8: EVPN-IRB** | **5** | | |
| 8.1 BVI gateway | 2 | | |
| 8.2 Distributed anycast GW | 2 | | |
| 8.3 Verify RT-2/RT-5 | 1 | | |
| **Section 9: E-Tree** | **3** | | |
| 9.1 Root/leaf service | 2 | | |
| 9.2 Verify leaf isolation | 1 | | |
| **Section 10: L2VPN auto-steer SR** | **3** | | |
| 10.1 Color → SR policy | 2 | | |
| 10.2 Verify SR policy use | 1 | | |
| **TOTAL** | **41** | | |
