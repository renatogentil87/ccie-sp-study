# WB29 — Internet Services (CGNAT, NAT64, MAP-T, Peering Policy)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Carrier-grade address translation + internet edge peering policy on the AS 65100 / AS 65200 internet border
**Level:** CCIE SP
**Prerequisite:** WB04 (BGP), WB25 (BGP XR), WB13 (Security) — eBGP edge and VRF basics in place

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent, with a ServiceInfra/CGN-capable image). CGN (NAT44/NAT64/MAP-T)
> relies on an XR service-plugin and is **not** available on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:**
> Internet-facing ASBRs **R5** (150.1.5.5) and **R6** (150.1.6.6, AS65100), peering router **R12** (150.2.12.12, AS65200 ASBR).
> R6↔R20 is the inter-AS/upstream-facing link (10.6.20.0/24). Simulate the "upstream provider" and an "IXP"
> on R20/R21 (AS65300) and a customer edge behind R12. Links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm the ASBRs (R5/R6) have reachability to both an "inside" (subscriber) and "outside" (internet) segment
- [ ] Confirm the eBGP edge from WB04/WB25 is UP (R6↔R20, R12↔R21)
- [ ] Confirm a CGN/ServiceInfra-capable XR image is loaded on the NAT node
- [ ] **Snapshot:** `WB29-00-baseline`

---

## Section 1 — CGNAT / NAT44 on IOS-XR [5 points]

Deploy Carrier-Grade NAT44 on an ASBR (R6) between a private subscriber VRF and the public internet VRF.

- [ ] **Task 1.1 [1 point]** — Create the **CGN service instance** (`service cgn <name>`) and bind it to a **service-type nat44**; attach a service-inline/`service-location` to the forwarding plane on R6.
- [ ] **Task 1.2 [1 point]** — Define the **inside VRF** (private subscribers, e.g. 100.64.0.0/10 RFC6598) and the **outside VRF** (public internet); map inside/outside to the CGN instance.
- [ ] **Task 1.3 [1 point]** — Configure the **NAT44 address pool** (public outside pool) and **port-block allocation** (`port-limit` / `port-block` size, dynamic PBA) so each subscriber gets a deterministic port range.
- [ ] **Task 1.4 [1 point]** — Enable **NAT translation logging** (syslog / NetFlow for CGN translation-create/delete, and bulk port-block allocation logging to reduce log volume).
- [ ] **Task 1.5 [1 point]** — Confirm active NAT44 translations are created when subscriber traffic flows to the outside, with correct inside→outside address+port mapping.
- [ ] **Task 1.6 [0 points — verification]** — Confirm the port-block allocation matches the configured block size and the pool is not exhausted.

**Verify / Snapshot**
- [ ] `show cgn nat44 <name> statistics` — sessions, pool usage, PBA counts
- [ ] `show cgn nat44 <name> inside-translation` / `outside-translation` — translation table
- [ ] `show cgn nat44 <name> pool` — pool + port-block allocation
- [ ] `show logging` / collector — translation + PBA log records present
- [ ] **Snapshot:** `WB29-01-cgnat44`

---

## Section 2 — NAT64 Stateful Translation [4 points]

Allow IPv6-only subscribers to reach the IPv4 internet via stateful NAT64 on R6.

- [ ] **Task 2.1 [1 point]** — Configure a **stateful NAT64** service instance (`service cgn <name>` → `service-type nat64 stateful`), binding an IPv6-only inside and the IPv4 outside.
- [ ] **Task 2.2 [1 point]** — Define the **NAT64 prefix** (e.g. well-known `64:ff9b::/96`) and the **IPv4 translation pool** with port allocation.
- [ ] **Task 2.3 [1 point]** — Document the **DNS64** integration (AAAA synthesis from A records using the NAT64 prefix) and where DNS64 sits relative to the translator; note it is control-plane only in this lab.
- [ ] **Task 2.4 [1 point]** — Verify **end-to-end**: an IPv6-only client reaches an IPv4 server (IPv6 src → NAT64 → IPv4 pool src), confirming the stateful binding.
- [ ] **Task 2.5 [0 points — verification]** — Confirm the IPv6→IPv4 session appears in the NAT64 translation table with the correct synthesized address.

**Verify / Snapshot**
- [ ] `show cgn nat64 stateful <name> statistics`
- [ ] `show cgn nat64 stateful <name> session` — v6→v4 bindings
- [ ] `show cgn nat64 stateful <name> pool` — IPv4 pool usage
- [ ] End-to-end: IPv6 client ping/curl to IPv4 server succeeds via NAT64
- [ ] **Snapshot:** `WB29-02-nat64`

---

## Section 3 — MAP-T (Mapping of Address and Port using Translation) [3 points]

Stateless, double-translation IPv4-over-IPv6 transport on R6 as the MAP-T border relay (BR).

- [ ] **Task 3.1 [1 point]** — Document MAP-T concepts in notes: **BMR** (Basic Mapping Rule — CE provisioning), **FMR** (Forwarding Mapping Rule — mesh CE↔CE), rule IPv6 prefix / IPv4 prefix / EA-bits / PSID; contrast stateless MAP-T vs stateful NAT64.
- [ ] **Task 3.2 [1 point]** — Configure a **MAP-T domain** on the border router (`service cgn <name>` → `service-type map-t`): default-mapping-rule (DMR), basic-mapping-rule with IPv6/IPv4 prefixes and sharing-ratio / port-set.
- [ ] **Task 3.3 [1 point]** — Verify the MAP-T translation (IPv4→IPv6 at the BR and back) and that the derived IPv4 address + PSID match the BMR math.
- [ ] **Task 3.4 [0 points — verification]** — Confirm statelessness (no per-flow session table — mapping is algorithmic).

**Verify / Snapshot**
- [ ] `show cgn map-t <name> statistics`
- [ ] `show cgn map-t <name>` — domain, BMR/DMR, sharing ratio
- [ ] Translation check across the BR matches the BMR-derived address/PSID
- [ ] **Snapshot:** `WB29-03-mapt`

---

## Section 4 — Internet Peering Policy [5 points]

Build a real internet-edge policy on R6 (toward upstream R20) and R12 (toward IXP R21).

- [ ] **Task 4.1 [1 point]** — Establish **eBGP peering** with an **upstream provider** (R6↔R20) and an **IXP peer** (R12↔R21); set `update-source` / `ebgp-multihop` as needed.
- [ ] **Task 4.2 [1 point]** — Apply **inbound filtering** (prefix-list to drop bogons/default from peers, AS-path filter to reject your own ASN / private ASNs, max-prefix limit) and **outbound filtering** (only advertise customer + own aggregates, community-based advertisement control).
- [ ] **Task 4.3 [1 point]** — Implement a **blackhole community (RTBH)** toward the upstream: tag a /32 with the well-known blackhole community (65535:666) and a route-policy that sets next-hop to a discard/null on the upstream, triggering upstream drop.
- [ ] **Task 4.4 [1 point]** — Configure **default-route origination toward customers** (originate 0.0.0.0/0 / ::/0 to the customer-facing neighbor via `default-originate` or route-policy) without leaking it to peers.
- [ ] **Task 4.5 [1 point]** — Configure **conditional advertisement** for a backup peering: advertise a prefix to the backup upstream ONLY when the primary path is withdrawn (`advertise-map` / non-exist-map equivalent via route-policy + tracking).
- [ ] **Task 4.6 [0 points — verification]** — Confirm advertised/received tables reflect the filters exactly (nothing leaks, blackhole works, default only to customers).

**Verify / Snapshot**
- [ ] `show bgp neighbor <upstream> advertised-routes` / `received routes`
- [ ] `show bgp <blackhole /32>` — community 65535:666, next-hop discard
- [ ] `show bgp neighbor <customer> advertised-routes` — default present
- [ ] Conditional advertisement: withdraw primary → backup prefix appears
- [ ] **Snapshot:** `WB29-04-peering-policy`

---

## Section 5 — Verification & Troubleshooting Drills [3 points]

- [ ] **Drill 5.1 — CGN pool exhaustion [1 point]:** shrink the NAT44 pool / port-block so a subscriber cannot allocate. Confirm translation failures and the exhaustion counter; diagnose via `show cgn nat44 statistics` and restore.
- [ ] **Drill 5.2 — RTBH not dropping [1 point]:** remove the blackhole route-policy on the upstream session. Confirm the /32 is no longer discarded; diagnose via `show bgp <prefix>` and fix.
- [ ] **Drill 5.3 — Default leaking to peer [1 point]:** misconfigure the outbound policy so 0.0.0.0/0 is advertised to the IXP peer. Detect it in `advertised-routes`, explain the risk, and fix.
- [ ] **Snapshot:** `WB29-05-tshoot`

**Required show commands (fluency):**
- [ ] `show cgn nat44 | nat64 stateful | map-t <name> statistics`
- [ ] `show cgn nat44 <name> inside-translation`
- [ ] `show bgp neighbor <n> advertised-routes` / `received routes`
- [ ] `show bgp <prefix>` (community + next-hop)
- [ ] `show logging`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: CGNAT / NAT44 | 5 | CCIE-level: CGN instance, PBA, logging |
| Section 2: NAT64 stateful | 4 | CCIE-level: stateful v6→v4, DNS64 concept |
| Section 3: MAP-T | 3 | CCIE-level: BMR/FMR, stateless domain |
| Section 4: Internet peering policy | 5 | CCIE-level: filtering, RTBH, conditional adv |
| Section 5: Troubleshooting Drills | 3 | CCIE-level: break-and-fix |
| **Total** | **20** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
18-20 points:  CCIE ready for this topic
14-17 points:  Strong, minor gaps to review
9-13 points:   Needs more practice
Below 9:       Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] CGNAT NAT44 with inside/outside VRF, pool + port-block allocation, translation logging
- [ ] Stateful NAT64 end-to-end (IPv6 client → IPv4 server), DNS64 documented
- [ ] MAP-T domain configured with BMR/DMR; stateless translation verified
- [ ] Internet peering policy: in/out filtering, RTBH, default origination, conditional advertisement
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB29-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: CGNAT / NAT44** | **5** | | |
| 1.1 CGN service instance | 1 | | |
| 1.2 Inside/outside VRF | 1 | | |
| 1.3 Pool + port-block | 1 | | |
| 1.4 Translation logging | 1 | | |
| 1.5 Translations created | 1 | | |
| **Section 2: NAT64 stateful** | **4** | | |
| 2.1 NAT64 instance | 1 | | |
| 2.2 Prefix + IPv4 pool | 1 | | |
| 2.3 DNS64 documented | 1 | | |
| 2.4 End-to-end v6→v4 | 1 | | |
| **Section 3: MAP-T** | **3** | | |
| 3.1 BMR/FMR concepts | 1 | | |
| 3.2 MAP-T domain | 1 | | |
| 3.3 Translation verified | 1 | | |
| **Section 4: Peering policy** | **5** | | |
| 4.1 eBGP upstream + IXP | 1 | | |
| 4.2 In/out filtering | 1 | | |
| 4.3 RTBH blackhole | 1 | | |
| 4.4 Default origination | 1 | | |
| 4.5 Conditional advertisement | 1 | | |
| **Section 5: Troubleshooting** | **3** | | |
| 5.1 CGN pool exhaustion | 1 | | |
| 5.2 RTBH not dropping | 1 | | |
| 5.3 Default leaking | 1 | | |
| **TOTAL** | **20** | | |
