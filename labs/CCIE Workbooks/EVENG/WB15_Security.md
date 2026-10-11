# CCIE SP Workbook 15 — Security (Domain 5: 10%)

**Platform:** IOS-XRv 9000
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** All 3 ISPs (Emerald + Gold + Garnet). Focus on control-plane and infrastructure protection.
**Format:** Question → Solution → Verification. All syntax is **IOS-XR** unless a comparison to IOS-XE is called out.

> **Scope note (Domain 5, 10%):** Security on IOS-XR is dominated by the *always-on* control-plane protection layer (**LPTS**), IGP/BGP authentication, RPKI/Flowspec/RTBH edge defenses, prefix hygiene, and MACsec on inter-AS links. Where IOS-XE uses **CoPP** (a service-policy on the control plane), IOS-XR uses **LPTS** — a hardware-programmed policer that is enabled by default and tunable, not built from scratch.

---

## Section 1 — Control Plane Protection (LPTS)

> **Concept — LPTS vs CoPP.** On **IOS-XE** you protect the RP by writing a CoPP `policy-map` (class-map matching BGP/OSPF/ICMP/SSH → `police`) and attaching it with `control-plane / service-policy input`. On **IOS-XR**, **LPTS (Local Packet Transport Services)** already classifies every for-us (locally destined) packet into *flow types* and polices each in the **NPU (network processor) hardware** via the **pre-IFIB (pifib)**. It is on by default — you don't build it, you *tune* it. The pIFIB is the hardware-programmed lookup table (an abstraction of the internal FIB) that decides which slice/CPU a punted packet goes to and at what rate.

### Task 1.1 — Understand and inspect the default LPTS policers
**Question:** Before changing anything, identify the default hardware policer rates for BGP, IS-IS, ICMP and SSH flow types on the ASBR, and explain what "flow type" and "pIFIB" mean.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — Rate-limit BGP and IS-IS punt rates
**Question:** Tighten the LPTS hardware policers for BGP and IS-IS so a punt flood cannot overwhelm the RP, while keeping legitimate adjacencies stable.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — Rate-limit ICMP and SSH management plane
**Question:** Limit ICMP-to-local and SSH punt rates to protect against ping floods and SSH brute-force punts, without breaking operational reachability.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.4 — Compare LPTS with CoPP (IOS-XE) and document the mapping
**Question:** Produce the equivalent IOS-XE CoPP config that would approximate the LPTS behavior above, and explain the two key differences.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — IGP Security

### Task 2.1 — IS-IS HMAC-MD5 authentication (key-chain, per-level, per-interface)
**Question:** Authenticate IS-IS on all Emerald core links using an **HMAC-MD5 key-chain**, applied at **Level-2** and per-interface, so a rogue router cannot form an adjacency.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — OSPF authentication (area-level and interface-level)
**Question:** On the Gold domain running OSPF, enable **HMAC-MD5** authentication at the **area** level, then override one link with **interface-level** authentication.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — Prevent rogue adjacencies with hello authentication (both IGPs)
**Question:** Demonstrate that hello-level authentication is what actually blocks a rogue neighbor, and prove it for both IS-IS and OSPF.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — BGP Security

### Task 3.1 — BGP authentication: TCP-AO (preferred) with MD5 fallback
**Question:** Authenticate the inter-AS E-R6↔Gar-R7 eBGP session. Prefer **TCP-AO** (RFC 5925) over legacy MD5; show both.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — GTSM (ttl-security) on eBGP
**Question:** Protect all directly-connected eBGP sessions with **GTSM** so spoofed multi-hop BGP packets are dropped in hardware.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — BGP RPKI origin validation
**Question:** Connect the ASBR to an **RPKI validator (cache)** and apply an origin-validation policy: drop **Invalid**, prefer **Valid**, accept **NotFound**.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.4 — RTBH (Remote Triggered Black Hole), community-triggered null0
**Question:** Build **destination-based RTBH**: a trigger router advertises a victim /32 tagged with a blackhole community; edge ASBRs set its next-hop to a Null0-bound discard address.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.5 — BGP Flowspec (redirect / rate-limit / drop by flow match)
**Question:** Enable **BGP Flowspec** and inject three rules: (a) **drop** a UDP/53 amplification flow to a victim, (b) **rate-limit** a TCP SYN flood, (c) **redirect** a victim subnet to a scrubbing VRF. Rules originate on a controller PE and are reflected to edge PEs.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — Infrastructure Protection

### Task 4.1 — Bogon filtering on eBGP (deny RFC1918/default, permit customer/peer only)
**Question:** On every eBGP peer, deny bogons (RFC1918, default route, documentation, multicast) inbound and outbound; only accept/announce legitimate customer/peer space.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — AS-PATH filtering (deny private ASNs, deny too-long paths)
**Question:** Reject routes whose AS-PATH contains **private ASNs** (64512–65534 / 4200000000–4294967294) or that are unreasonably **long** (path-length attack / route leak).


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — Maximum-prefix limits on eBGP peers
**Question:** Protect the RIB from a peer that fat-fingers a full table or leaks; set a warning threshold and a hard cap that tears down and restarts the session.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — MACsec (Layer 2 Encryption)

> **Concept.** **MACsec (IEEE 802.1AE)** provides hop-by-hop Layer-2 confidentiality + integrity on a physical link — it encrypts the Ethernet payload between two directly connected interfaces (e.g. E-R6↔Gar-R7), independent of L3/BGP. Session keys are negotiated by **MKA (MACsec Key Agreement, 802.1X)** from a pre-shared **CAK/CKN** carried in a key-chain of type `macsec`. Unlike IPsec (L3, routed, per-flow), MACsec is line-rate hardware crypto on a single link.

### Task 5.1 — MACsec key-chain and MKA policy
**Question:** Create the MACsec pre-shared key (CKN/CAK) key-chain and an MKA policy (cipher, confidentiality offset, SAK rekey) to be used on the inter-AS link.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — Apply MACsec to the inter-AS link (E-R6↔Gar-R7)
**Question:** Apply the key-chain + MKA policy to the physical interface on both ASBRs and confirm the link is encrypted and BGP still runs over it.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — Troubleshooting

### Task 6.1 — BGP peer's prefixes rejected (RPKI Invalid / ROA mismatch)
**Symptom:** A previously-working prefix from an eBGP peer disappears from the RIB after RPKI was enabled. `show bgp summary` shows the session **Established**, but the prefix is gone.

**Diagnosis path:**
```
show bgp ipv4 unicast <prefix>                 ! validation-state: invalid
show bgp rpki table <prefix>                   ! ROA present but origin-AS or max-length mismatches
show bgp rpki server summary                   ! validator up (rules out cache-down false positive)
```
**Root cause:** The peer is announcing the prefix with an **origin-AS** (or a more-specific length) that does **not** match the ROA in the RPKI cache → the prefix is marked **Invalid** → dropped by `RPKI-EBGP-IN` (Task 3.4's `if validation-state is invalid then drop`).

**Fix options:**
1. **Correct the ROA** (right fix): register/update the ROA so its origin-AS and max-length cover the real announcement — the prefix becomes **Valid** on next cache refresh.
2. If the announcement itself is wrong, the peer must fix their origin-AS.
3. Temporary/interop: change policy to `set local-preference` low instead of `drop` (deprefer rather than discard) — document as a workaround, not a solution.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — Flowspec rule not applied (missing `local-install`)
**Symptom:** A Flowspec rule is present in BGP (`show bgp ipv4 flowspec` shows the NLRI) and even shows under `show flowspec ipv4`, but attack traffic is **not** being dropped/rate-limited on the edge PE.

**Diagnosis path:**
```
show bgp ipv4 flowspec                         ! NLRI received/reflected — OK
show flowspec ipv4                             ! rule listed but "not installed in dataplane"
show flowspec ipv4 detail                      ! action present, but no interface programmed
run show controllers np ...                    ! no NP entry for the flow
```
**Root cause:** The `flowspec` config is missing **`local-install interface-all`** (or per-interface `local-install`), so the router **learns** the rule via BGP but never programs it into the forwarding/NPU hardware.

**Fix:**
```
flowspec
 address-family ipv4
  local-install interface-all
!
```


> *Try this yourself first. Solution available in `solutions/` folder.*

