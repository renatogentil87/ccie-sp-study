# WB03 — MPLS LDP (All 3 ASes)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Reference:** `00_topology_reference.md`
**Prerequisite snapshot:** `WB02-ospf-complete` (IS-IS in AS 65100/65300 and OSPF in AS 65200 all converged; every loopback reachable within its own AS).

> **Style:** Tasks only. No configuration snippets. Derive the commands yourself.
> This workbook enables LSP forwarding inside each AS using LDP. It must not disturb the IGPs from WB01/WB02.
> LDP runs **within** each AS only — inter-AS label exchange is reserved for the Inter-AS VPN workbooks.
> Each section is additive — do not break earlier sections.

---

## Scope

- Enable **LDP** on every **core (IGP-facing)** interface in all three ASes.
- LDP router-id anchored to Loopback0 everywhere (stable, matches IGP RID practice).
- Do NOT enable LDP on inter-AS links or PE-CE links.
- Goal: a full mesh of LSPs so any loopback-to-loopback path inside an AS is label-switched with PHP at the penultimate hop.

---

## Section 1 — LDP on AS 65100 (IS-IS core) [3 points]

### Task 1.1 — Enable MPLS/LDP, pin router-id [1 point]
Enable MPLS forwarding and the **LDP** label protocol on R1–R8. Force the **LDP router-id from Loopback0** (and make it stable so a flap doesn't re-elect it).

### Task 1.2 — Activate LDP on all IS-IS core interfaces [2 points]
Enable MPLS/LDP on **every AS 65100 core interface** (the AS 65100 Core link table). Leave inter-AS (R5↔R11, R6↔R12, R6↔R20) and PE-CE interfaces label-free.

### Task 1.3 — Verify AS 65100 LDP [0 points — verification]
Confirm:
- LDP **neighbors/sessions UP** on every core link (operational, Oper).
- Local & remote **label bindings** exist for all 150.1.x.x/32 loopbacks (LIB).
- The **LFIB** is populated and shows outgoing labels toward each loopback.

---

## Section 2 — LDP on AS 65200 (OSPF core) [3 points]

### Task 2.1 — Enable MPLS/LDP, pin router-id [1 point]
Enable MPLS forwarding and LDP on R11–R16, R19 with **LDP router-id from Loopback0**.

### Task 2.2 — Activate LDP on all OSPF core interfaces [2 points]
Enable MPLS/LDP on **every AS 65200 core interface** (AS 65200 Core link table). Leave inter-AS and PE-CE interfaces label-free.

### Task 2.3 — Verify AS 65200 LDP [0 points — verification]
Same validation as Sec 1: sessions UP, label bindings for all 150.2.x.x/32, LFIB populated.

---

## Section 3 — LDP on AS 65300 (IS-IS core) [3 points]

### Task 3.1 — Enable MPLS/LDP, pin router-id [1 point]
Enable MPLS forwarding and LDP on R20, R21, R22, R23, R30 with **LDP router-id from Loopback0**.

### Task 3.2 — Activate LDP on all IS-IS core interfaces [2 points]
Enable MPLS/LDP on **every AS 65300 core interface** (AS 65300 Core link table). Leave inter-AS (R6↔R20) and PE-CE interfaces label-free.

### Task 3.3 — Verify AS 65300 LDP [0 points — verification]
Same validation: sessions UP, label bindings for all 150.3.x.x/32, LFIB populated.

---

## Section 4 — LDP Advanced (all ASes) [14 points]

### Task 4.1 — Label allocation filtering (host routes only) [5 points]
Restrict LDP label allocation to **/32 host routes (loopbacks) only** so the core doesn't bind labels for the /24 transit subnets (reduces LFIB size; standard SP practice). Apply in all three ASes.
- Verify only /32s have labels and transit /24s no longer appear in the LIB/LFIB.

### Task 4.2 — Session protection [3 points]
Enable **LDP session protection** (keep LDP sessions alive via the loopback through a targeted hello if a link flaps) on the core routers. Verify the protection timer/targeted-hello behavior.

### Task 4.3 — LDP-IGP synchronization [3 points]
Enable **LDP-IGP sync** so the IGP holds a link at max metric until LDP is up on it (prevents black-holing labeled traffic over a link whose LDP session hasn't established).
- Do this for IS-IS (AS 65100, 65300) and OSPF (AS 65200).
- Verify the IGP advertises max metric on a link while LDP is still forming, then reverts.

### Task 4.4 — LDP authentication (MD5) [3 points]
Protect LDP sessions with **MD5 authentication** (TCP-MD5) between LDP peers in each AS. Verify sessions stay UP with matching keys and drop with a mismatch (test, then correct).

---

## Section 5 — Verification (end-to-end label switching) [1 point]

### Task 5.1 — traceroute mpls / labeled path [0 points — verification]
Run an MPLS-aware **traceroute (traceroute mpls / LSP ping)** between loopbacks across each AS and confirm each hop shows a **label** in the path.

### Task 5.2 — PHP at penultimate hop [0 points — verification]
Confirm **penultimate-hop popping**: the second-to-last router pops the label (imposes implicit-null / shows "Pop Label" outgoing) so the egress PE receives an unlabeled packet. Verify on a specific 3+ hop path in each AS.

### Task 5.3 — LFIB populated & consistent [0 points — verification]
On a transit P router in each AS, confirm the **LFIB** has in-label → out-label swaps for remote loopbacks and the pop entry for its directly-attached egress loopbacks. Cross-check in-label on upstream matches out-label expectation downstream for one full LSP.

### Task 5.4 — Full-mesh LSP spot check [0 points — verification]
Spot-check loopback-to-loopback label switching across each AS (e.g. R1→R8, R11→R16, R20→R30). Confirm CEF/LFIB chooses labeled next-hops, not plain IP.

### Task 5.5 — Snapshot [1 point]
Take a GNS3 snapshot with LDP fully operational in all three ASes and all advanced features active.

**Snapshot name:** `WB03-ldp-mpls-complete`

---

## Completion Checklist

**AS 65100 (Sec 1)**
- [ ] MPLS/LDP enabled R1–R8, router-id from Lo0 (1.1)
- [ ] LDP on all IS-IS core interfaces (1.2)
- [ ] Sessions UP, /32 bindings, LFIB populated (1.3)

**AS 65200 (Sec 2)**
- [ ] MPLS/LDP enabled R11–R16/R19, router-id from Lo0 (2.1)
- [ ] LDP on all OSPF core interfaces (2.2)
- [ ] Sessions UP, /32 bindings, LFIB populated (2.3)

**AS 65300 (Sec 3)**
- [ ] MPLS/LDP enabled R20–R23/R30, router-id from Lo0 (3.1)
- [ ] LDP on all IS-IS core interfaces (3.2)
- [ ] Sessions UP, /32 bindings, LFIB populated (3.3)

**Advanced (Sec 4)**
- [ ] Label allocation filtering — host routes only, all ASes (4.1)
- [ ] LDP session protection (4.2)
- [ ] LDP-IGP sync (IS-IS + OSPF) (4.3)
- [ ] LDP MD5 authentication, mismatch tested (4.4)

**Verification (Sec 5)**
- [ ] traceroute mpls shows labels (5.1)
- [ ] PHP confirmed at penultimate hop (5.2)
- [ ] LFIB in→out swaps + pop entries verified (5.3)
- [ ] Full-mesh LSP spot check passed (5.4)
- [ ] Snapshot `WB03-ldp-mpls-complete` taken (5.5)

---

## Scoring Summary

| Section | Tasks | Points |
|---|---|---|
| Section 1 — LDP on AS 65100 | 1.1–1.3 | 3 |
| Section 2 — LDP on AS 65200 | 2.1–2.3 | 3 |
| Section 3 — LDP on AS 65300 | 3.1–3.3 | 3 |
| Section 4 — LDP Advanced | 4.1–4.4 | 14 |
| Section 5 — Verification | 5.1–5.5 | 1 |
| **Total** | | **24** |

### Rating Scale (% of points earned)

| Score | Rating |
|---|---|
| 90–100% | CCIE ready |
| 70–89% | Strong, minor gaps |
| 50–69% | Needs more practice |
| Below 50% | Review theory |

### Scoring Rules

- **From memory:** full points
- **Used `?` on router:** 2/3 points (round up)
- **Checked solution:** 1/3 points (round up)
- **Skipped:** 0 points

---

## Attempt History

### Attempt 1 — October 8, 2026

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: LDP AS 65100** | **3** | **3** | |
| 1.1 Enable LDP + router-id | 1 | 1 | ✅ All R1-R8 |
| 1.2 LDP on core interfaces | 2 | 2 | ✅ Correct interfaces, correct neighbors |
| **Section 2: LDP AS 65200** | **3** | **3** | |
| 2.1 Enable LDP + router-id | 1 | 1 | ✅ All R11-R16, R19 |
| 2.2 LDP on core interfaces | 2 | 2 | ✅ All OSPF core interfaces |
| **Section 3: LDP AS 65300** | **3** | **3** | |
| 3.1 Enable LDP + router-id | 1 | 1 | ✅ All R20-R23, R30 |
| 3.2 LDP on core interfaces | 2 | 2 | ✅ All IS-IS core interfaces |
| **Section 4: LDP Advanced** | **14** | **14** | |
| 4.1 Label filtering (host-routes) | 5 | 5 | ✅ ACL 10 + advertise-labels for 10 on ALL routers |
| 4.2 Session protection | 3 | 3 | ✅ ACL 20 + session protection on ALL routers |
| 4.3 LDP-IGP sync | 3 | 3 | ✅ Sync achieved on all interfaces (verified show mpls ldp igp sync) |
| 4.4 LDP authentication (MD5) | 3 | 3 | ✅ Per-neighbor passwords + password required for 20 |
| **Section 5: Verification** | **1** | **1** | |
| 5.5 Snapshot | 1 | 1 | ✅ Taken |
| **TOTAL** | **24** | **24** | **100% — CCIE ready** 🔥 |

**All tasks from memory. No solution checked. Perfect score.**
