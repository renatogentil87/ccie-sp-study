# WB23 — IS-IS on IOS-XR

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** IS-IS L2 in AS 65100 (R1-R8) and AS 65300 (R20-R23, R30)
**Level:** CCIE SP
**Prerequisite:** WB00 equivalent (IP addressing + loopbacks on XR), basic XR CLI fluency

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). Flex-CLI configuration groups, keychains, and XR IS-IS syntax
> are **not** available on the GNS3 7200/IOS 15.2 lab (not Dynamips).
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference (IS-IS domains):**
> **AS 65100** (area 49.0001, L2): R1(PE) 150.1.1.1 · R2(PE) 150.1.2.2 · R3(P) 150.1.3.3 · R4(P) 150.1.4.4 · R5(ASBR) 150.1.5.5 · R6(ASBR) 150.1.6.6 · R7(RR) 150.1.7.7 · R8(RR) 150.1.8.8
> **AS 65300** (area 49.0003, L2): R20(ASBR/PE) 150.3.20.20 · R21(ASBR/PE) 150.3.21.21 · R22(PE) 150.3.22.22 · R23(PE) 150.3.23.23 · R30(RR) 150.3.30.30
> IPv6: 2001:db8:1::/48 (AS 65100), 2001:db8:3::/48 (AS 65300). Core links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm all Loopback0 /32s and interface IPv4/IPv6 addresses are configured on R1-R8 and R20-R23/R30
- [ ] Confirm no prior IS-IS instance is running (`show isis` — empty) or baseline it cleanly
- [ ] Confirm point-to-point core links are up at L1 (`show interfaces` — line protocol up)
- [ ] **Snapshot:** `WB23-00-baseline`

---

## Section 1 — Flex-CLI Configuration Groups [5 points]

Build a reusable `group CCIE-ISIS` using regex interface matching + `apply-group`, so one group configures the common IS-IS profile fleet-wide.

- [ ] **Task 1.1 [2 points]** — Define `group CCIE-ISIS` containing a `router isis` block with: `is-type level-2-only`, `metric-style wide`, **single-topology** IPv4+IPv6 (`address-family ipv4 unicast` and `address-family ipv6 unicast` under one topology), and `set-overload-bit on-startup 180`.
- [ ] **Task 1.2 [1 point]** — Inside the group, use **regex interface wildcards** to match the core P2P interfaces and set `point-to-point`, `hello-padding disable`, and the metrics **200 (IPv4) / 400 (IPv6)**.
- [ ] **Task 1.3 [1 point]** — Inside the group, mark Loopback0 as **passive** and enable **LDP auto-config** (`mpls ldp auto-config`) + **LDP-IGP sync** (`mpls ldp sync`) on the core interfaces.
- [ ] **Task 1.4 [1 point]** — `apply-group CCIE-ISIS` globally (and/or per-router), then confirm the group expands onto each node with `show running-config group` / `show configuration commit changes`.
- [ ] **Task 1.5 [0 points — verification]** — Confirm the inherited config appears in the effective running-config on at least two nodes.

**Verify / Snapshot**
- [ ] `show running-config group CCIE-ISIS` — group body correct
- [ ] `show isis` — is-type L2-only, metric-style wide, single-topology IPv4+IPv6
- [ ] `show isis interface brief` — P2P, passive loopbacks, metrics 200/400
- [ ] `show mpls ldp igp sync` — LDP sync active on core interfaces
- [ ] **Snapshot:** `WB23-01-flex-cli`

---

## Section 2 — IS-IS Multi-Level Design [5 points]

Impose a level hierarchy: PEs/ASBRs L1-only, P routers L1/L2, RRs L2-only, with controlled L1→L2 propagation.

- [ ] **Task 2.1 [2 points]** — Set `is-type`: PEs/ASBRs (R1/R2/R5/R6, R20-R23) as **level-1**, P routers (R3/R4) as **level-1-2**, RRs (R7/R8, R30) as **level-2-only**. Set `circuit-type` per interface to match the adjacency level.
- [ ] **Task 2.2 [1 point]** — Confirm the correct adjacency level forms on each link (L1 inside an area edge, L2 in the backbone) via `show isis adjacency`.
- [ ] **Task 2.3 [1 point]** — Configure **L1→L2 propagation** with a `route-policy` so only intended loopbacks are propagated; attach a **drop-all** route-policy where leaking must be prevented (`propagate level 1 into level 2 route-policy DROP-ALL`).
- [ ] **Task 2.4 [1 point]** — Confirm that unwanted L1 prefixes are **not** leaked into L2 (drop-all working) while required prefixes are.
- [ ] **Task 2.5 [0 points — verification]** — Confirm L1/L2 database separation on a P router.

**Verify / Snapshot**
- [ ] `show isis adjacency` — correct level per neighbor
- [ ] `show isis database level-1` / `level-2` — expected LSPs per level
- [ ] `show route isis` — only intended inter-level prefixes present
- [ ] `show rpl route-policy DROP-ALL` — policy drops as intended
- [ ] **Snapshot:** `WB23-02-multilevel`

---

## Section 3 — IS-IS Authentication (keychain) [5 points]

Secure LSP and hello exchange using an HMAC-MD5 keychain referenced from the Flex-CLI group.

- [ ] **Task 3.1 [2 points]** — Create `key chain ISIS-KEY` with a key: `cryptographic-algorithm HMAC-MD5`, `accept-tolerance infinite`, and `send-lifetime` / `accept-lifetime` starting from the current date.
- [ ] **Task 3.2 [1 point]** — Under the `CCIE-ISIS` group `router isis` block, apply **`lsp-password keychain ISIS-KEY`** (per level as required).
- [ ] **Task 3.3 [1 point]** — Under the group's **interface wildcards**, apply **`hello-password keychain ISIS-KEY`** so hellos are authenticated on all matched core interfaces.
- [ ] **Task 3.4 [1 point]** — Confirm adjacencies stay UP with authentication, then prove enforcement by temporarily mismatching a key and observing the adjacency drop.
- [ ] **Task 3.5 [0 points — verification]** — Confirm LSP authentication is active (authentication TLV present).

**Verify / Snapshot**
- [ ] `show key chain ISIS-KEY` — key, algorithm, lifetimes
- [ ] `show isis adjacency detail` — authenticated hellos, adjacency UP
- [ ] `show isis database verbose <node>` — authentication TLV present
- [ ] Mismatch test: adjacency drops; restore and re-converge
- [ ] **Snapshot:** `WB23-03-auth`

---

## Section 4 — IS-IS for IPv6 (single-topology) [3 points]

- [ ] **Task 4.1 [1 point]** — Confirm **`address-family ipv6 unicast`** is enabled under IS-IS (single-topology, inherited from the group) on all nodes in both ASes.
- [ ] **Task 4.2 [1 point]** — Confirm IPv6 is enabled on every participating interface (interface `address-family ipv6 unicast`) and Loopback0 IPv6 is passive-advertised.
- [ ] **Task 4.3 [1 point]** — Verify end-to-end IPv6 reachability within each AS (ping each remote Loopback0 IPv6 inside AS 65100 and inside AS 65300).
- [ ] **Task 4.4 [0 points — verification]** — Confirm IPv6 routes are L2 and share the IPv4 SPF (single-topology, not multi-topology).

**Verify / Snapshot**
- [ ] `show isis ipv6 topology` — single-topology (not MT)
- [ ] `show route ipv6 isis` — remote loopbacks present
- [ ] `ping ipv6 <remote Loopback0>` — reachable within each AS
- [ ] **Snapshot:** `WB23-04-ipv6`

---

## Section 5 — IS-IS Advanced (BFD / prefix suppression / SPF throttle) [4 points]

- [ ] **Task 5.1 [2 points]** — Enable **BFD for IS-IS** on the core interfaces (`bfd minimum-interval`, `bfd multiplier`, and `address-family ipv4/ipv6 ... bfd fast-detect` under the IS-IS interface). Confirm sub-second failure detection.
- [ ] **Task 5.2 [1 point]** — Enable **prefix suppression** so only loopbacks are advertised: `advertise passive-only` under the IS-IS address-family (connected transit prefixes suppressed).
- [ ] **Task 5.3 [1 point]** — Tune **SPF throttle timers**: `spf-interval <initial-wait> <secondary-wait> <maximum-wait>` (XR millisecond form). State the reasoning for the chosen values in notes.
- [ ] **Task 5.4 [0 points — verification]** — Confirm transit prefixes are gone from the IS-IS RIB (only passive/loopback prefixes remain).

**Verify / Snapshot**
- [ ] `show bfd session` — core sessions UP, timers negotiated
- [ ] `show route isis` — only passive/loopback prefixes (transit suppressed)
- [ ] `show isis` — spf-interval timers applied
- [ ] **Snapshot:** `WB23-05-advanced`

---

## Section 6 — Verification & Troubleshooting Drills [5 points]

- [ ] **Drill 6.1 — Level mismatch [2 points]:** change one interface `circuit-type` so an L1/L2 pair mismatches. Confirm the adjacency fails or forms at the wrong level; diagnose via `show isis adjacency` and fix.
- [ ] **Drill 6.2 — Metric-style mismatch [1 point]:** set one node to `metric-style narrow`. Observe wide/narrow interop impact on route selection; diagnose and restore.
- [ ] **Drill 6.3 — Overload bit [1 point]:** manually set the overload bit on R3. Confirm transit traffic avoids R3 (loopbacks still reachable). Diagnose via `show isis database` and clear.
- [ ] **Drill 6.4 — Auth key expiry [1 point]:** use a key whose send-lifetime has not started. Confirm the adjacency fails; diagnose with `show key chain` and fix the lifetime.
- [ ] **Snapshot:** `WB23-06-tshoot`

**Required show commands (fluency):**
- [ ] `show isis adjacency [detail]`
- [ ] `show isis database [verbose]`
- [ ] `show route isis` / `show route ipv6 isis`
- [ ] `show isis interface brief`
- [ ] `show bfd session`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: Flex-CLI config groups | 5 | CCIE-level: group + regex + apply-group |
| Section 2: Multi-level design | 5 | CCIE-level: is-type/circuit-type + L1→L2 policy |
| Section 3: Authentication (keychain) | 5 | CCIE-level: keychain, lsp/hello-password |
| Section 4: IS-IS for IPv6 | 3 | Standard: single-topology IPv6 AF |
| Section 5: Advanced (BFD/suppress/SPF) | 4 | Intermediate: fast-detect + tuning |
| Section 6: Troubleshooting Drills | 5 | CCIE-level: break-and-fix |
| **Total** | **27** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
24-27 points:  CCIE ready for this topic
18-23 points:  Strong, minor gaps to review
12-17 points:  Needs more practice
Below 12:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Flex-CLI `CCIE-ISIS` group built, applied, and inheriting correctly fleet-wide
- [ ] Multi-level hierarchy (L1-only PEs/ASBRs, L1/L2 P, L2-only RRs) with drop-all L1→L2 policy
- [ ] HMAC-MD5 keychain authenticating LSPs (lsp-password) and hellos (hello-password)
- [ ] Single-topology IPv6 reachable within AS 65100 and AS 65300
- [ ] BFD fast-detect, prefix suppression (passive-only), SPF throttle tuned
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB23-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Flex-CLI groups** | **5** | | |
| 1.1 Group router isis block | 2 | | |
| 1.2 Regex interface + metrics | 1 | | |
| 1.3 Passive + LDP auto/sync | 1 | | |
| 1.4 apply-group + inherit | 1 | | |
| **Section 2: Multi-level** | **5** | | |
| 2.1 is-type/circuit-type | 2 | | |
| 2.2 Adjacency levels | 1 | | |
| 2.3 L1→L2 drop-all policy | 1 | | |
| 2.4 No unwanted leaking | 1 | | |
| **Section 3: Authentication** | **5** | | |
| 3.1 keychain ISIS-KEY | 2 | | |
| 3.2 lsp-password | 1 | | |
| 3.3 hello-password | 1 | | |
| 3.4 Enforcement test | 1 | | |
| **Section 4: IPv6** | **3** | | |
| 4.1 IPv6 AF under IS-IS | 1 | | |
| 4.2 IPv6 per-interface | 1 | | |
| 4.3 Reachability | 1 | | |
| **Section 5: Advanced** | **4** | | |
| 5.1 BFD for IS-IS | 2 | | |
| 5.2 Prefix suppression | 1 | | |
| 5.3 SPF throttle | 1 | | |
| **Section 6: Troubleshooting** | **5** | | |
| 6.1 Level mismatch | 2 | | |
| 6.2 Metric-style mismatch | 1 | | |
| 6.3 Overload bit | 1 | | |
| 6.4 Auth key expiry | 1 | | |
| **TOTAL** | **27** | | |
