# WB01 — IS-IS (AS 65100 + AS 65300)

**Platform:** Cisco 7200 (C7200-ADVENTERPRISEK9-M), IOS 15.2(4)M11
**Reference:** `00_topology_reference.md`
**Prerequisite snapshot:** `WB00-foundation-ip-complete` (all directly-connected links ping).

> **Style:** Tasks only. No configuration snippets. Derive the commands yourself.
> This workbook builds the IGP for both IS-IS autonomous systems. AS 65200 (OSPF) is handled in WB02.
> Each section is additive — do not break earlier sections as you add features.

---

## Scope

- **AS 65100** (Area 49.0001): R1–R8, IS-IS **Level-2-only**.
- **AS 65300** (Area 49.0003): R20, R21, R22, R23, R30, IS-IS **Level-2-only**.
- NET-IDs are pre-computed in the topology reference (**IS-IS NET-IDs** tables) — use them exactly.
- Core link subnets only (10.x for 65100, 30.x for 65300). Do NOT run IS-IS toward CEs or across inter-AS links yet.

---

## Section 1 — IS-IS L2-only on AS 65100 (R1–R8)

### Task 1.1 — Enable IS-IS, assign NET-IDs
Enable an IS-IS process on R1–R8 and assign the NET from the **AS 65100 (Area 49.0001)** table.
- Configure the process as **Level-2-only** domain-wide (these are all in one area; no L1).

### Task 1.2 — Activate IS-IS on core interfaces
Enable IS-IS routing on each AS 65100 **core** interface (every interface in the AS 65100 Core link table) plus Loopback0.
- Do NOT activate IS-IS on inter-AS links (R5↔R11, R6↔R12, R6↔R20) or PE-CE links.

### Task 1.3 — Wide metrics
Set the metric style to **wide** (new-style TLVs) across AS 65100. This is mandatory for later TE/large-metric work and SP best practice.

### Task 1.4 — Passive loopbacks
Make Loopback0 **passive** in IS-IS on R1–R8 so the /32 is advertised but no adjacency is attempted on it.

### Task 1.5 — Point-to-point network type
Set every AS 65100 core link to IS-IS **point-to-point** network type (even the Ethernet/GigE links) to suppress DIS election and pseudonode LSPs on P2P segments.

### Task 1.6 — Verify AS 65100 adjacencies & routes
Confirm all expected L2 adjacencies are **UP** and that every R1–R8 loopback (150.1.x.x/32) is reachable from every other AS 65100 router. Validate the IS-IS topology/database is consistent.

---

## Section 2 — IS-IS L2-only on AS 65300 (R20–R23, R30)

### Task 2.1 — Enable IS-IS, assign NET-IDs
Enable IS-IS on R20, R21, R22, R23, R30 and assign the NET from the **AS 65300 (Area 49.0003)** table. Level-2-only.

### Task 2.2 — Activate IS-IS on core interfaces
Enable IS-IS on each AS 65300 **core** interface (AS 65300 Core link table) plus Loopback0.
- Do NOT activate on inter-AS link R6↔R20 or any PE-CE link.

### Task 2.3 — Wide metrics + passive loopbacks + point-to-point
Apply the same three treatments as AS 65100:
- metric style **wide**,
- Loopback0 **passive**,
- all core links **point-to-point** network type.

### Task 2.4 — Verify AS 65300 adjacencies & routes
Confirm all L2 adjacencies UP and all 65300 loopbacks (150.3.x.x/32) mutually reachable within AS 65300.

---

## Section 3 — IS-IS Advanced (both ASes)

### Task 3.1 — HMAC-MD5 authentication via key-chain
Configure IS-IS authentication using a **key-chain** with **HMAC-MD5** on both domains:
- Apply **IS-IS (LSP/SNP) authentication** at the Level-2 level.
- Use a distinct key-chain per AS.
- Verify adjacencies survive and that mismatched keys drop adjacency (test, then correct).

### Task 3.2 — BFD for fast failure detection
Enable **BFD** on the AS 65100 and AS 65300 core P2P links and register IS-IS as a BFD client so adjacency loss is detected sub-second.
- Verify BFD sessions are UP and that IS-IS is a registered protocol.

### Task 3.3 — Prefix suppression on P-P links
Suppress advertisement of the **point-to-point transit link prefixes** (the /24 core subnets) so only loopbacks and real services are in the IGP (reduces LSDB size, best practice for SP cores).
- Verify core /24s disappear from remote routing tables while loopbacks remain.

### Task 3.4 — Overload bit on-startup
Configure the **overload bit on startup** (with a timer) on the P routers / transit nodes so a freshly reloaded router does not attract transit traffic before convergence completes.
- Verify the overload bit sets on reload and clears after the timer.

---

## Section 4 — IS-IS IPv6 (single-topology, dual-stack)

### Task 4.1 — Assign IPv6 on core links & loopbacks
Add IPv6 addressing to AS 65100 and AS 65300 core interfaces and loopbacks (use a consistent scheme, e.g. map the IPv4 subnet into a documentation prefix). Enable IPv6 unicast routing.

### Task 4.2 — Enable the IPv6 address family in IS-IS
Enable IS-IS for **IPv6** on the same interfaces, running **single-topology** mode (shared SPF with IPv4).

### Task 4.3 — Verify dual-stack
Confirm:
- IPv6 IS-IS adjacencies/topology match IPv4 (single-topology implies one adjacency carrying both).
- Every loopback is reachable over both IPv4 and IPv6 within each AS.
- `show` the IS-IS IPv6 RIB/topology and confirm no single-topology mismatch warnings.

---

## Section 5 — Verification & Troubleshooting

### Task 5.1 — Adjacency stuck in INIT
On one AS 65100 P2P link, intentionally create then diagnose an adjacency **stuck in INIT**. Identify likely causes (one-way auth mismatch, MTU mismatch, one side not P2P / mismatched network type, missing `isis` on one interface). Document the diagnostic path and fix.

### Task 5.2 — Routes missing from the table
Create a scenario where a loopback is in the IS-IS database but not installed in the RIB (e.g. metric-style mismatch narrow vs wide, or overload bit stuck set). Diagnose and resolve.

### Task 5.3 — Pseudonode cleanup
If any Ethernet core link was left as broadcast earlier, confirm a **pseudonode LSP** exists, then convert the link to point-to-point and verify the pseudonode LSP ages out / is purged. Confirm the LSDB is clean (no stale pseudonode entries).

### Task 5.4 — Final convergence check
Full reachability verification: from R1 reach every 65100 loopback; from R20 reach every 65300 loopback; confirm authentication, BFD, prefix-suppression, and dual-stack are all simultaneously active without breaking any adjacency.

### Task 5.5 — Snapshot
Take a GNS3 snapshot with both IS-IS domains fully converged and all advanced features active.

**Snapshot name:** `WB01-isis-complete`

---

## Completion Checklist

**AS 65100 (Sec 1)**
- [ ] NET-IDs assigned R1–R8, Level-2-only (1.1)
- [ ] IS-IS active on all core interfaces + Lo0 (1.2)
- [ ] Wide metrics (1.3)
- [ ] Passive loopbacks (1.4)
- [ ] Point-to-point network type on all core links (1.5)
- [ ] All adjacencies UP, all 150.1.x.x reachable (1.6)

**AS 65300 (Sec 2)**
- [ ] NET-IDs assigned R20–R23/R30, Level-2-only (2.1)
- [ ] IS-IS active on all core interfaces + Lo0 (2.2)
- [ ] Wide metrics + passive loopbacks + P2P (2.3)
- [ ] All adjacencies UP, all 150.3.x.x reachable (2.4)

**Advanced (Sec 3)**
- [ ] HMAC-MD5 key-chain auth, both ASes (3.1)
- [ ] BFD for IS-IS on core links (3.2)
- [ ] Prefix suppression on P-P links (3.3)
- [ ] Overload bit on-startup (3.4)

**IPv6 (Sec 4)**
- [ ] IPv6 addressing on core + loopbacks (4.1)
- [ ] IS-IS IPv6 AF, single-topology (4.2)
- [ ] Dual-stack reachability verified (4.3)

**Troubleshooting (Sec 5)**
- [ ] INIT adjacency diagnosed & fixed (5.1)
- [ ] Missing-route scenario diagnosed & fixed (5.2)
- [ ] Pseudonode cleanup verified (5.3)
- [ ] Final convergence check passed (5.4)
- [ ] Snapshot `WB01-isis-complete` taken (5.5)
