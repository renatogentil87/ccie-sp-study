# WB32 — Advanced Security (RPKI, AAA, MACsec, LPTS, ACL Compression)

**Platform:** IOS-XR (EVE-NG — IOS-XRv9000 nodes mapped onto R1-R32 topology)
**Scope:** Control-plane and data-plane hardening on the AS 65100 edge/core (ASBRs + PEs)
**Level:** CCIE SP
**Prerequisite:** WB04/WB25 (BGP XR), WB13 (Security), WB21 (NSO/automation — for gRPC/gNMI context)

> **Platform note:** This workbook uses **IOS-XR syntax** and must be run on an EVE-NG lab
> (IOS-XRv9000 or equivalent). BGP-RPKI, MACsec/MKA, LPTS, hardware ACL compression, and
> gRPC/gNMI TLS are **not** available on the GNS3 7200/IOS 15.2 lab.
>
> **Format:** Tasks only. No configurations provided — you derive the config.
> Work **progressively**: finish and verify each section before moving on.
> Take an **EVE-NG snapshot/export** at each checkpoint so you can roll back cleanly.

> **Reference:**
> ASBRs **R5** (150.1.5.5) / **R6** (150.1.6.6, AS65100), PEs **R1** (150.1.1.1) / **R14** (150.2.14.14).
> Simulate an RPKI validator (RTR cache), a TACACS+ server, and a gNMI client reachable from the nodes.
> MACsec on a PE-PE / PE-CE p2p link (e.g. R5↔R6 10.5.6.0/24 or R1 access). Links per `00_topology_reference.md`.

---

## Pre-flight

- [ ] Confirm the eBGP edge from WB04/WB25 is UP (R6↔R20, R5↔R11)
- [ ] Confirm reachable servers: RPKI RTR cache, TACACS+, gNMI client/collector
- [ ] Confirm a point-to-point link available for MACsec testing
- [ ] **Snapshot:** `WB32-00-baseline`

---

## Section 1 — BGP-RPKI (Origin AS Validation) [5 points]

- [ ] **Task 1.1 [1 point]** — Configure the **RPKI cache (RTR) server** connection under BGP: `rpki server <ip>` with transport (port/username), refresh/purge timers.
- [ ] **Task 1.2 [1 point]** — Enable **origin-AS validation** on the eBGP neighbors (`bgp origin-as validation enable`, signal/use per neighbor).
- [ ] **Task 1.3 [1 point]** — Confirm the three **route validation states** appear: **Valid**, **Invalid**, **NotFound**, driven by the ROAs in the cache.
- [ ] **Task 1.4 [1 point]** — Build a **route-policy** that **drops Invalid** and **prefers Valid over NotFound** (set local-pref / weight by validation state).
- [ ] **Task 1.5 [1 point]** — Verify the policy result: Invalid prefixes are not installed/advertised; Valid preferred in best-path.
- [ ] **Task 1.6 [0 points — verification]** — Confirm cache session is up and ROA count is non-zero before trusting states.

**Verify / Snapshot**
- [ ] `show bgp rpki server summary` — cache session up, ROA count
- [ ] `show bgp <prefix>` — validation state (Valid/Invalid/NotFound)
- [ ] `show bgp origin-as validity invalid` — invalids filtered
- [ ] **Snapshot:** `WB32-01-rpki`

---

## Section 2 — AAA (Authentication, Authorization, Accounting) [4 points]

- [ ] **Task 2.1 [1 point]** — Configure the **TACACS+ server** and a **server-group** (host, key, single-connection / source-interface Loopback0).
- [ ] **Task 2.2 [1 point]** — Configure **AAA authentication** for login with a **method list** `group tacacs+ local` (TACACS+ first, local fallback).
- [ ] **Task 2.3 [1 point]** — Configure **AAA authorization** for **exec** and **commands** (per task group / command level) via TACACS+.
- [ ] **Task 2.4 [1 point]** — Configure **AAA accounting** for **commands** (and exec start-stop) so command execution is logged to TACACS+; create a **local fallback user**.
- [ ] **Task 2.5 [0 points — verification]** — Test fallback: with TACACS+ unreachable, confirm local login still works.

**Verify / Snapshot**
- [ ] `show aaa ...` / `show tacacs` — server reachable, group bound
- [ ] `test aaa group tacacs+ <user> <pass>` — auth success
- [ ] Accounting records reach TACACS+; local fallback works when server down
- [ ] **Snapshot:** `WB32-02-aaa`

---

## Section 3 — MACsec (Media Access Control Security) [4 points]

- [ ] **Task 3.1 [1 point]** — Document MACsec concepts in notes: **MKA** (MACsec Key Agreement), **SAK** (Secure Association Key), **CAK** (Connectivity Association Key, pre-shared or EAP-derived), and the SecY data-plane.
- [ ] **Task 3.2 [1 point]** — Configure a **MACsec key-chain** (CKN/CAK) and a **MACsec policy** (cipher suite GCM-AES-XPN-256, confidentiality offset, SAK rekey).
- [ ] **Task 3.3 [1 point]** — Apply MACsec on a **point-to-point PE-PE or PE-CE link** (both ends) and bring up the MKA session.
- [ ] **Task 3.4 [1 point]** — Configure the **should-secure vs must-secure** fallback policy and reason about traffic behavior when the peer lacks MACsec.
- [ ] **Task 3.5 [0 points — verification]** — Confirm the link is encrypted (secured) and MKA peers are live.

**Verify / Snapshot**
- [ ] `show macsec mka session` — secured, peer live
- [ ] `show macsec mka summary` / `show macsec statistics` — encrypt/decrypt counters
- [ ] Should-secure: confirm clear-text allowed until peer secures
- [ ] **Snapshot:** `WB32-03-macsec`

---

## Section 4 — LPTS (Local Packet Transport Services) on IOS-XR [3 points]

- [ ] **Task 4.1 [1 point]** — Document **LPTS** as XR's built-in, always-on **CoPP** (pre-IFIB flow classification protecting the RP) and how it differs from classic CoPP.
- [ ] **Task 4.2 [1 point]** — View the **default LPTS policer rates** per flow: `show lpts pifib hardware police` (identify BGP, IS-IS, OSPF, ICMP, SSH flows).
- [ ] **Task 4.3 [1 point]** — **Customize LPTS flow rates** for specific protocols: `lpts pifib hardware police flow <protocol> rate <pps>` (e.g. raise BGP, lower ICMP) and apply.
- [ ] **Task 4.4 [0 points — verification]** — Confirm the customized rates are programmed and that excess punted traffic for a flow is dropped (policer drops increment).

**Verify / Snapshot**
- [ ] `show lpts pifib hardware police` — per-flow rates (default + customized)
- [ ] `show lpts pifib hardware police location <loc>` — drops/accepts
- [ ] `show drops` — LPTS policer drops on the throttled flow
- [ ] **Snapshot:** `WB32-04-lpts`

---

## Section 5 — ACL Compression and Object Groups [3 points]

- [ ] **Task 5.1 [1 point]** — Configure **object-groups** (network and port) to represent scalable groups of hosts/subnets and port ranges.
- [ ] **Task 5.2 [1 point]** — Build an ACL referencing the object-groups and **apply the compressed ACL** on an interface (ingress).
- [ ] **Task 5.3 [1 point]** — Enable / verify **ACL hardware compression** and reason about **atomic vs non-atomic** programming (traffic hit during update).
- [ ] **Task 5.4 [0 points — verification]** — Confirm the ACL matches as expected and the compressed form is programmed in hardware.

**Verify / Snapshot**
- [ ] `show access-lists <acl> hardware ... ` — compressed, hit counts
- [ ] `show object-group network` / `show object-group port`
- [ ] `show access-lists usage pfilter` — hardware programming
- [ ] **Snapshot:** `WB32-05-acl-objgroup`

---

## Section 6 — TLS/mTLS for gRPC/gNMI [3 points]

- [ ] **Task 6.1 [1 point]** — Generate an **RSA key** and a **self-signed certificate / trustpoint** (crypto key, crypto ca trustpoint) for the gRPC server.
- [ ] **Task 6.2 [1 point]** — Configure the **gRPC server with TLS enabled** (`grpc` → `tls`, port, trustpoint) for gNMI/gNOI.
- [ ] **Task 6.3 [1 point]** — Enable **mutual TLS** (client certificate verification — require and validate the gNMI client cert against the CA/trustpoint).
- [ ] **Task 6.4 [0 points — verification]** — Confirm a gNMI client can connect only with a valid client cert (mTLS), and is rejected without one.

**Verify / Snapshot**
- [ ] `show grpc` / `show grpc status` — TLS enabled, mTLS required
- [ ] `show crypto ca certificates` — trustpoint + cert present
- [ ] gNMI client: valid cert connects, missing/invalid cert rejected
- [ ] **Snapshot:** `WB32-06-grpc-tls`

---

## Section 7 — Verification & Troubleshooting Drills [2 points]

- [ ] **Drill 7.1 — RPKI cache down → all NotFound [1 point]:** break the RTR cache session. Confirm every prefix reverts to NotFound (no Valid/Invalid) and the drop-invalid policy becomes ineffective; diagnose via `show bgp rpki server summary` and restore.
- [ ] **Drill 7.2 — MACsec key mismatch [1 point]:** set different CKN/CAK on the two ends. Confirm the MKA session fails and (with must-secure) the link drops traffic; diagnose via `show macsec mka session` and realign the key-chain.
- [ ] **Snapshot:** `WB32-07-tshoot`

**Required show commands (fluency):**
- [ ] `show bgp rpki server summary` / `show bgp <prefix>` (validation)
- [ ] `show aaa ...` / `test aaa group tacacs+`
- [ ] `show macsec mka session`
- [ ] `show lpts pifib hardware police` / `show drops`
- [ ] `show access-lists` / `show object-group` / `show grpc`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: BGP-RPKI | 5 | CCIE-level: RTR cache, validation states, policy |
| Section 2: AAA (TACACS+) | 4 | Intermediate: authn/authz/acct, fallback |
| Section 3: MACsec | 4 | CCIE-level: MKA, key-chain, should/must-secure |
| Section 4: LPTS | 3 | CCIE-level: XR CoPP, flow policer tuning |
| Section 5: ACL compression / object-groups | 3 | Intermediate: object-groups, hw compression |
| Section 6: TLS/mTLS gRPC/gNMI | 3 | CCIE-level: trustpoint, mutual TLS |
| Section 7: Troubleshooting Drills | 2 | CCIE-level: break-and-fix |
| **Total** | **24** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` on router: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
21-24 points:  CCIE ready for this topic
16-20 points:  Strong, minor gaps to review
11-15 points:  Needs more practice
Below 11:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] BGP-RPKI origin validation (Valid/Invalid/NotFound) with drop-invalid + prefer-valid policy
- [ ] AAA via TACACS+ (authn/authz/acct) with local fallback tested
- [ ] MACsec/MKA on a p2p link with should/must-secure fallback
- [ ] LPTS default rates viewed and per-protocol flow rates customized + verified
- [ ] ACL object-groups + hardware compression (atomic vs non-atomic) applied
- [ ] gRPC/gNMI TLS + mutual TLS with client-cert verification
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] **Final snapshot:** `WB32-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: BGP-RPKI** | **5** | | |
| 1.1 RTR cache connection | 1 | | |
| 1.2 Origin-AS validation | 1 | | |
| 1.3 Valid/Invalid/NotFound | 1 | | |
| 1.4 Drop-invalid policy | 1 | | |
| 1.5 Policy verified | 1 | | |
| **Section 2: AAA** | **4** | | |
| 2.1 TACACS+ server-group | 1 | | |
| 2.2 Authentication method list | 1 | | |
| 2.3 Authorization exec/commands | 1 | | |
| 2.4 Accounting + local fallback | 1 | | |
| **Section 3: MACsec** | **4** | | |
| 3.1 MKA/SAK/CAK concepts | 1 | | |
| 3.2 Key-chain + policy | 1 | | |
| 3.3 Apply on p2p link | 1 | | |
| 3.4 Should/must-secure | 1 | | |
| **Section 4: LPTS** | **3** | | |
| 4.1 LPTS as XR CoPP | 1 | | |
| 4.2 Default rates viewed | 1 | | |
| 4.3 Customize flow rates | 1 | | |
| **Section 5: ACL / object-groups** | **3** | | |
| 5.1 Object-groups | 1 | | |
| 5.2 Compressed ACL applied | 1 | | |
| 5.3 Atomic vs non-atomic | 1 | | |
| **Section 6: TLS/mTLS gRPC** | **3** | | |
| 6.1 Key + trustpoint | 1 | | |
| 6.2 gRPC TLS enabled | 1 | | |
| 6.3 Mutual TLS | 1 | | |
| **Section 7: Troubleshooting** | **2** | | |
| 7.1 RPKI cache down | 1 | | |
| 7.2 MACsec key mismatch | 1 | | |
| **TOTAL** | **24** | | |
