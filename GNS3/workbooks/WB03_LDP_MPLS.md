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

## Section 1 — LDP on AS 65100 (IS-IS core)

### Task 1.1 — Enable MPLS/LDP, pin router-id
Enable MPLS forwarding and the **LDP** label protocol on R1–R8. Force the **LDP router-id from Loopback0** (and make it stable so a flap doesn't re-elect it).

### Task 1.2 — Activate LDP on all IS-IS core interfaces
Enable MPLS/LDP on **every AS 65100 core interface** (the AS 65100 Core link table). Leave inter-AS (R5↔R11, R6↔R12, R6↔R20) and PE-CE interfaces label-free.

### Task 1.3 — Verify AS 65100 LDP
Confirm:
- LDP **neighbors/sessions UP** on every core link (operational, Oper).
- Local & remote **label bindings** exist for all 150.1.x.x/32 loopbacks (LIB).
- The **LFIB** is populated and shows outgoing labels toward each loopback.

---

## Section 2 — LDP on AS 65200 (OSPF core)

### Task 2.1 — Enable MPLS/LDP, pin router-id
Enable MPLS forwarding and LDP on R11–R16, R19 with **LDP router-id from Loopback0**.

### Task 2.2 — Activate LDP on all OSPF core interfaces
Enable MPLS/LDP on **every AS 65200 core interface** (AS 65200 Core link table). Leave inter-AS and PE-CE interfaces label-free.

### Task 2.3 — Verify AS 65200 LDP
Same validation as Sec 1: sessions UP, label bindings for all 150.2.x.x/32, LFIB populated.

---

## Section 3 — LDP on AS 65300 (IS-IS core)

### Task 3.1 — Enable MPLS/LDP, pin router-id
Enable MPLS forwarding and LDP on R20, R21, R22, R23, R30 with **LDP router-id from Loopback0**.

### Task 3.2 — Activate LDP on all IS-IS core interfaces
Enable MPLS/LDP on **every AS 65300 core interface** (AS 65300 Core link table). Leave inter-AS (R6↔R20) and PE-CE interfaces label-free.

### Task 3.3 — Verify AS 65300 LDP
Same validation: sessions UP, label bindings for all 150.3.x.x/32, LFIB populated.

---

## Section 4 — LDP Advanced (all ASes)

### Task 4.1 — Label allocation filtering (host routes only)
Restrict LDP label allocation to **/32 host routes (loopbacks) only** so the core doesn't bind labels for the /24 transit subnets (reduces LFIB size; standard SP practice). Apply in all three ASes.
- Verify only /32s have labels and transit /24s no longer appear in the LIB/LFIB.

### Task 4.2 — Session protection
Enable **LDP session protection** (keep LDP sessions alive via the loopback through a targeted hello if a link flaps) on the core routers. Verify the protection timer/targeted-hello behavior.

### Task 4.3 — LDP-IGP synchronization
Enable **LDP-IGP sync** so the IGP holds a link at max metric until LDP is up on it (prevents black-holing labeled traffic over a link whose LDP session hasn't established).
- Do this for IS-IS (AS 65100, 65300) and OSPF (AS 65200).
- Verify the IGP advertises max metric on a link while LDP is still forming, then reverts.

### Task 4.4 — LDP authentication (MD5)
Protect LDP sessions with **MD5 authentication** (TCP-MD5) between LDP peers in each AS. Verify sessions stay UP with matching keys and drop with a mismatch (test, then correct).

---

## Section 5 — Verification (end-to-end label switching)

### Task 5.1 — traceroute mpls / labeled path
Run an MPLS-aware **traceroute (traceroute mpls / LSP ping)** between loopbacks across each AS and confirm each hop shows a **label** in the path.

### Task 5.2 — PHP at penultimate hop
Confirm **penultimate-hop popping**: the second-to-last router pops the label (imposes implicit-null / shows "Pop Label" outgoing) so the egress PE receives an unlabeled packet. Verify on a specific 3+ hop path in each AS.

### Task 5.3 — LFIB populated & consistent
On a transit P router in each AS, confirm the **LFIB** has in-label → out-label swaps for remote loopbacks and the pop entry for its directly-attached egress loopbacks. Cross-check in-label on upstream matches out-label expectation downstream for one full LSP.

### Task 5.4 — Full-mesh LSP spot check
Spot-check loopback-to-loopback label switching across each AS (e.g. R1→R8, R11→R16, R20→R30). Confirm CEF/LFIB chooses labeled next-hops, not plain IP.

### Task 5.5 — Snapshot
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
