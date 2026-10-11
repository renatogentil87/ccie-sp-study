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

**Solution:** No configuration required — LPTS is always on. Inspect the programmed hardware entries. Each *flow type* (e.g. `BGP-known`, `BGP-cfg-peer`, `ISIS-known`, `ICMP-local`, `SSH-known`) has a default policer rate (packets/sec) programmed per NPU.

```
! Inspection only — no config
show lpts pifib hardware entry brief location 0/0/CPU0
show lpts pifib hardware police location 0/0/CPU0
show lpts pifib hardware police location 0/0/CPU0 | include BGP|ISIS|ICMP|SSH
show lpts flows                         ! live flow counters (accepted/dropped)
show lpts pifib hardware entry type ipv4 | include BGP
```

- **Flow type:** LPTS's classification of a for-us packet (protocol + state, e.g. a *configured* BGP peer vs an *unknown* BGP source). Configured/known peers get higher rates than unknown sources — this is the built-in anti-DoS behavior.
- **pIFIB (pre-Internal FIB):** the hardware table that maps each flow type to (a) which CPU/queue to punt to and (b) a policer rate, so a flood of one protocol cannot starve the RP of another.

**Verification:**
```
show lpts pifib hardware police location 0/0/CPU0
```
Look for non-zero default rates per flow type, and `Accepted`/`Dropped` counters incrementing under load.

### Task 1.2 — Rate-limit BGP and IS-IS punt rates
**Question:** Tighten the LPTS hardware policers for BGP and IS-IS so a punt flood cannot overwhelm the RP, while keeping legitimate adjacencies stable.

**Solution:** Override the default per-flow-type rates under `lpts pifib hardware police`.

```
lpts pifib hardware police
 flow bgp-known rate 4000
 flow bgp-cfg-peer rate 2000
 flow bgp-default rate 100
 flow isis-known rate 2000
 flow isis-mcast-known rate 2000
!
```

- `bgp-known` = established/known peers (highest trust) → higher rate.
- `bgp-cfg-peer` = configured but not-yet-established peers → moderate rate.
- `bgp-default` = unknown BGP sources → very low rate (DoS protection).

**Verification:**
```
show lpts pifib hardware police location 0/0/CPU0 | include bgp|isis
show lpts pifib hardware entry brief | include BGP
show bgp summary          ! adjacencies stay Established after the change
```
Confirm the new rates are programmed and BGP/IS-IS neighbors remain up.

### Task 1.3 — Rate-limit ICMP and SSH management plane
**Question:** Limit ICMP-to-local and SSH punt rates to protect against ping floods and SSH brute-force punts, without breaking operational reachability.

**Solution:**
```
lpts pifib hardware police
 flow icmp-local rate 500
 flow icmp-app rate 200
 flow ssh-known rate 300
 flow ssh-default rate 50
!
```

**Verification:**
```
show lpts pifib hardware police location 0/0/CPU0 | include icmp|ssh
show lpts flows | include ICMP|SSH        ! Dropped increments under flood; Accepted steady for legit
ssh <asbr-loopback>                       ! management still works
ping <asbr-loopback>                      ! normal ping still succeeds
```

### Task 1.4 — Compare LPTS with CoPP (IOS-XE) and document the mapping
**Question:** Produce the equivalent IOS-XE CoPP config that would approximate the LPTS behavior above, and explain the two key differences.

**Solution (IOS-XE CoPP equivalent — for comparison only, not applied on XR):**
```
! IOS-XE
ip access-list extended ACL-BGP
 permit tcp any any eq bgp
 permit tcp any eq bgp any
class-map match-all CM-BGP
 match access-group name ACL-BGP
class-map match-all CM-SSH
 match access-group name ACL-SSH
policy-map COPP
 class CM-BGP
  police 8000 conform-action transmit exceed-action drop
 class CM-SSH
  police 4000 conform-action transmit exceed-action drop
 class class-default
  police 1000 conform-action transmit exceed-action drop
control-plane
 service-policy input COPP
```

**Differences to memorize:**
1. **On by default vs built from scratch** — LPTS is always running with sane defaults; CoPP does nothing until you write and attach a policy.
2. **Where it runs** — LPTS polices in the **NPU/pIFIB hardware per flow type** (distributed, line-rate); CoPP polices in the RP path via a `service-policy` on `control-plane` (classes you define).

**Verification (XR side stays as tuned in 1.1–1.3):**
```
show lpts pifib hardware police location 0/0/CPU0
```

---

## Section 2 — IGP Security

### Task 2.1 — IS-IS HMAC-MD5 authentication (key-chain, per-level, per-interface)
**Question:** Authenticate IS-IS on all Emerald core links using an **HMAC-MD5 key-chain**, applied at **Level-2** and per-interface, so a rogue router cannot form an adjacency.

**Solution:**
```
key chain ISIS-KC
 key 1
  accept-lifetime 00:00:00 january 01 2020 infinite
  send-lifetime   00:00:00 january 01 2020 infinite
  key-string cisco123
  cryptographic-algorithm HMAC-MD5
!
router isis CORE
 lsp-password keychain ISIS-KC level 2          ! authenticates LSPs/SNPs at L2
 interface GigabitEthernet0/0/0/0
  hello-password keychain ISIS-KC               ! authenticates Hellos (adjacency)
 !
 interface GigabitEthernet0/0/0/1
  hello-password keychain ISIS-KC
 !
!
```

- `hello-password` = per-interface **hello** auth → prevents rogue **adjacencies**.
- `lsp-password` = LSP/CSNP/PSNP auth (routing DB integrity), applied per-level.

**Verification:**
```
show key chain ISIS-KC
show isis interface GigabitEthernet0/0/0/0 | include Authentication
show isis adjacency                 ! neighbors re-form after auth applied
! Negative test: mismatch key-string on one side → adjacency DOWN, then fix → UP
```

### Task 2.2 — OSPF authentication (area-level and interface-level)
**Question:** On the Gold domain running OSPF, enable **HMAC-MD5** authentication at the **area** level, then override one link with **interface-level** authentication.

**Solution:**
```
key chain OSPF-KC
 key 1
  key-string cisco123
  cryptographic-algorithm HMAC-MD5
!
router ospf 1
 area 0
  authentication keychain OSPF-KC              ! area-wide (inherited by all its interfaces)
  interface GigabitEthernet0/0/0/0
  !
  interface GigabitEthernet0/0/0/2
   authentication keychain OSPF-KC-ALT         ! interface override wins over area
  !
 !
!
```

**Verification:**
```
show ospf interface GigabitEthernet0/0/0/0 | include authentication
show ospf neighbor                  ! FULL after auth converges
! Negative test: remove key on one neighbor → adjacency stuck in INIT/EXSTART
```

### Task 2.3 — Prevent rogue adjacencies with hello authentication (both IGPs)
**Question:** Demonstrate that hello-level authentication is what actually blocks a rogue neighbor, and prove it for both IS-IS and OSPF.

**Solution:** Auth is already in place from 2.1/2.2 at the **hello** level. Introduce a rogue router with no key (or wrong key) on a shared segment.

- IS-IS: the rogue sends IIH Hellos without the HMAC-MD5 TLV → the legitimate router rejects them → **no adjacency**.
- OSPF: rogue Hellos fail the crypto-auth check → dropped → neighbor never leaves DOWN/INIT.

**Verification:**
```
! IS-IS
show isis adjacency                 ! rogue NOT listed
debug isis adj-packets              ! "authentication failure" on rejected IIH
! OSPF
show ospf neighbor                  ! rogue NOT listed
show ospf statistics                ! auth-fail counter increments
```

---

## Section 3 — BGP Security

### Task 3.1 — BGP authentication: TCP-AO (preferred) with MD5 fallback
**Question:** Authenticate the inter-AS E-R6↔Gar-R7 eBGP session. Prefer **TCP-AO** (RFC 5925) over legacy MD5; show both.

**Solution — TCP-AO (preferred):**
```
key chain BGP-AO
 key 1
  key-string cisco123
  cryptographic-algorithm HMAC-SHA1-160
  send-id 1
  accept-id 1
!
router bgp 65100
 neighbor 10.0.12.2
  ao BGP-AO include-tcp-options enable          ! TCP-AO
  remote-as 65200
!
```

**Solution — legacy MD5 (fallback / interop):**
```
router bgp 65100
 neighbor 10.0.12.2
  password encrypted <hash>                     ! TCP-MD5, RFC 2385
!
```

- **TCP-AO** is preferred: stronger algorithms (SHA-1/SHA-256), key rollover via key-chain, protects against replay. MD5 is a single static key, no rollover, deprecated.

**Verification:**
```
show bgp neighbor 10.0.12.2 | include AO|MD5|Authentication
show bgp summary                  ! session Established
```

### Task 3.2 — GTSM (ttl-security) on eBGP
**Question:** Protect all directly-connected eBGP sessions with **GTSM** so spoofed multi-hop BGP packets are dropped in hardware.

**Solution:**
```
router bgp 65100
 neighbor 10.0.12.2
  ttl-security                     ! expects TTL=255 arriving (hop count 1 for directly connected)
!
```

- eBGP normally sends TTL=1. With GTSM, packets are sent with **TTL=255** and the receiver only accepts packets whose TTL is ≥ (255 − configured-hops). An off-path attacker cannot forge a TTL that survives the path, so spoofed BGP is dropped **before** TCP processing (LPTS-friendly).

**Verification:**
```
show bgp neighbor 10.0.12.2 | include TTL|ttl
show bgp summary                  ! Established with GTSM active
! Negative: a simulated 2-hop injection is dropped
```

### Task 3.3 — BGP RPKI origin validation
**Question:** Connect the ASBR to an **RPKI validator (cache)** and apply an origin-validation policy: drop **Invalid**, prefer **Valid**, accept **NotFound**.

**Solution:**
```
router bgp 65100
 rpki server 192.0.2.100
  transport tcp port 3323
  refresh-time 600
 !
 bgp origin-as validation enable               ! turn on validation
 address-family ipv4 unicast
  bgp origin-as validation signal ibgp         ! (optional) signal validity into iBGP
!
route-policy RPKI-EBGP-IN
  if validation-state is invalid then
    drop
  elseif validation-state is valid then
    set local-preference 200
    pass
  else
    set local-preference 100                    ! not-found
    pass
  endif
end-policy
!
router bgp 65100
 neighbor 10.0.12.2
  address-family ipv4 unicast
   route-policy RPKI-EBGP-IN in
!
```

**Verification:**
```
show bgp rpki server summary          ! validator session up
show bgp rpki table                   ! ROAs downloaded
show bgp ipv4 unicast | include valid|invalid|not found
show bgp <prefix>                     ! per-prefix validation-state
```

### Task 3.4 — RTBH (Remote Triggered Black Hole), community-triggered null0
**Question:** Build **destination-based RTBH**: a trigger router advertises a victim /32 tagged with a blackhole community; edge ASBRs set its next-hop to a Null0-bound discard address.

**Solution — on each edge ASBR (the "black-holers"):**
```
route 192.0.2.1/32 Null0                        ! static discard next-hop (the blackhole)
!
community-set BLACKHOLE
  65100:666
end-set
!
route-policy RTBH-IN
  if community matches-any BLACKHOLE then
    set next-hop 192.0.2.1                       ! resolves via Null0 → traffic discarded
    pass
  else
    pass
  endif
end-policy
!
router bgp 65100
 neighbor <trigger-router>
  address-family ipv4 unicast
   route-policy RTBH-IN in
!
```

**Solution — on the trigger router (originate the victim /32 with community):**
```
route-policy RTBH-TRIGGER
  set community (65100:666) additive
  set local-preference 200
end-policy
! redistribute/network the victim /32 with this policy outbound
```

**Verification:**
```
show bgp ipv4 unicast 198.51.100.9/32          ! victim /32, community 65100:666 present
show route 198.51.100.9                        ! next-hop 192.0.2.1 → Null0
show cef 198.51.100.9                          ! "drop" / via Null0
! traffic toward victim is discarded at the edge; rest of the network unaffected
```

### Task 3.5 — BGP Flowspec (redirect / rate-limit / drop by flow match)
**Question:** Enable **BGP Flowspec** and inject three rules: (a) **drop** a UDP/53 amplification flow to a victim, (b) **rate-limit** a TCP SYN flood, (c) **redirect** a victim subnet to a scrubbing VRF. Rules originate on a controller PE and are reflected to edge PEs.

**Solution — enable the Flowspec AF everywhere (controller + RRs + edge):**
```
router bgp 65100
 address-family ipv4 flowspec
 !
 neighbor <rr-or-peer>
  address-family ipv4 flowspec
!
flowspec
 address-family ipv4
  local-install interface-all                   ! <-- REQUIRED so rules program into the dataplane
!
```

**Solution — define/originate rules on the controller PE:**
```
class-map type traffic match-all FS-DNS-AMP
 match destination-address ipv4 198.51.100.9 255.255.255.255
 match protocol udp
 match source-port 53
 end-class-map
!
class-map type traffic match-all FS-SYN
 match destination-address ipv4 198.51.100.9 255.255.255.255
 match protocol tcp
 match tcp-flag 0x02 mask 0x02                  ! SYN set
 end-class-map
!
policy-map type pbr FS-POLICY
 class type traffic FS-DNS-AMP
  drop
 class type traffic FS-SYN
  police rate 10 mbps
 class type traffic class-default
!
flowspec
 address-family ipv4
  service-policy type pbr FS-POLICY
!
! Redirect example (VRF scrub):
class-map type traffic match-all FS-REDIRECT
 match destination-address ipv4 198.51.100.0 255.255.255.0
 end-class-map
! action inside policy-map: redirect nexthop route-target <scrub-RT>  (or redirect dest-vrf SCRUB)
```

**Verification:**
```
show flowspec ipv4                   ! all 3 rules present on edge PEs
show flowspec ipv4 detail            ! actions: Traffic-rate 0 (drop), rate-limit, redirect
show flowspec vrf all afi-all        ! if using VRF redirect
show bgp ipv4 flowspec               ! NLRI reflected to peers
! Functional: UDP/53 flow dropped; SYN flood capped at 10Mbps; victim subnet steered to SCRUB
```

---

## Section 4 — Infrastructure Protection

### Task 4.1 — Bogon filtering on eBGP (deny RFC1918/default, permit customer/peer only)
**Question:** On every eBGP peer, deny bogons (RFC1918, default route, documentation, multicast) inbound and outbound; only accept/announce legitimate customer/peer space.

**Solution:**
```
prefix-set BOGONS
  0.0.0.0/0,                                    ! default
  0.0.0.0/8 le 32,
  10.0.0.0/8 le 32,                             ! RFC1918
  100.64.0.0/10 le 32,                          ! CGNAT
  127.0.0.0/8 le 32,
  169.254.0.0/16 le 32,
  172.16.0.0/12 le 32,                          ! RFC1918
  192.0.2.0/24 le 32,                           ! TEST-NET-1
  192.168.0.0/16 le 32,                         ! RFC1918
  198.18.0.0/15 le 32,
  224.0.0.0/3 le 32                             ! multicast + reserved
end-set
!
route-policy EBGP-IN
  if destination in BOGONS then
    drop
  else
    pass
  endif
end-policy
!
router bgp 65100
 neighbor 10.0.12.2
  address-family ipv4 unicast
   route-policy EBGP-IN in
   route-policy EBGP-IN out          ! also prevent leaking bogons outbound
!
```

**Verification:**
```
show bgp ipv4 unicast | include 10\.|172\.1[6-9]\.|192\.168\.   ! none present
show bgp neighbor 10.0.12.2 policy in                          ! policy attached
```

### Task 4.2 — AS-PATH filtering (deny private ASNs, deny too-long paths)
**Question:** Reject routes whose AS-PATH contains **private ASNs** (64512–65534 / 4200000000–4294967294) or that are unreasonably **long** (path-length attack / route leak).

**Solution:**
```
as-path-set PRIVATE-ASN
  ios-regex '_(6451[2-9]|645[2-9][0-9]|64[6-9][0-9][0-9]|65[0-4][0-9][0-9]|655[0-2][0-9]|6553[0-4])_'
end-set
!
route-policy EBGP-ASPATH-IN
  if as-path in PRIVATE-ASN then
    drop
  elseif as-path length ge 30 then              ! excessively long path
    drop
  else
    pass
  endif
end-policy
!
router bgp 65100
 neighbor 10.0.12.2
  address-family ipv4 unicast
   route-policy EBGP-ASPATH-IN in
!
```

**Verification:**
```
show bgp ipv4 unicast regexp _65...._          ! private ASNs no longer accepted
show bgp neighbor 10.0.12.2 | include Dropped   ! prefixes dropped by policy
```

### Task 4.3 — Maximum-prefix limits on eBGP peers
**Question:** Protect the RIB from a peer that fat-fingers a full table or leaks; set a warning threshold and a hard cap that tears down and restarts the session.

**Solution:**
```
router bgp 65100
 neighbor 10.0.12.2
  address-family ipv4 unicast
   maximum-prefix 1000 80 restart 5             ! cap 1000, warn at 80%, restart after 5 min
!
```

- `1000` = hard limit; `80` = warning-percent; `restart 5` = auto-reset the peer after 5 minutes (use `warning-only` for a non-disruptive alert instead).

**Verification:**
```
show bgp neighbor 10.0.12.2 | include Maximum|prefix
show bgp summary                  ! State goes to "(PfxCt)" / Idle if the cap is exceeded
show logging | include MAXPFX
```

---

## Section 5 — MACsec (Layer 2 Encryption)

> **Concept.** **MACsec (IEEE 802.1AE)** provides hop-by-hop Layer-2 confidentiality + integrity on a physical link — it encrypts the Ethernet payload between two directly connected interfaces (e.g. E-R6↔Gar-R7), independent of L3/BGP. Session keys are negotiated by **MKA (MACsec Key Agreement, 802.1X)** from a pre-shared **CAK/CKN** carried in a key-chain of type `macsec`. Unlike IPsec (L3, routed, per-flow), MACsec is line-rate hardware crypto on a single link.

### Task 5.1 — MACsec key-chain and MKA policy
**Question:** Create the MACsec pre-shared key (CKN/CAK) key-chain and an MKA policy (cipher, confidentiality offset, SAK rekey) to be used on the inter-AS link.

**Solution:**
```
key chain MACSEC-KC macsec
 key 1234                                        ! CKN (hex, even length)
  key-string 12345678901234567890123456789012 cryptographic-algorithm aes-128-cmac
  lifetime 00:00:00 january 01 2020 infinite
!
macsec-policy MKA-POLICY
 cipher-suite GCM-AES-XPN-256
 conf-offset CONF-OFFSET-0                        ! encrypt entire frame payload
 window-size 64
 sak-rekey-interval seconds 3600
 security-policy must-secure                      ! drop cleartext (fail-closed)
!
```

**Verification:**
```
show key chain MACSEC-KC
show macsec mka policy MKA-POLICY
```

### Task 5.2 — Apply MACsec to the inter-AS link (E-R6↔Gar-R7)
**Question:** Apply the key-chain + MKA policy to the physical interface on both ASBRs and confirm the link is encrypted and BGP still runs over it.

**Solution (identical on both ASBRs, same CKN/CAK):**
```
interface GigabitEthernet0/0/0/0
 macsec psk-keychain MACSEC-KC policy MKA-POLICY
!
```

**Verification:**
```
show macsec mka session interface GigabitEthernet0/0/0/0    ! Secured, MKA "Secured"
show macsec mka statistics interface GigabitEthernet0/0/0/0
show macsec secy statistics interface GigabitEthernet0/0/0/0 ! EncryptedPkts / DecryptedPkts incrementing
show bgp summary                                             ! eBGP still Established over the secured link
! Negative: mismatch CKN/CAK → session stays "Init"/"Pending", link does not secure
```

---

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

**Verification:**
```
show bgp ipv4 unicast <prefix>                 ! validation-state now: valid
show bgp <prefix>                              ! reinstalled, best path
```

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

**Verification:**
```
show flowspec ipv4 detail                      ! now shows programmed on interfaces
show flowspec vrf all afi-all
! Functional: matching attack traffic is now dropped / rate-limited as intended
```

---

## Final Validation Checklist
```
[ ] LPTS defaults inspected; BGP/IS-IS/ICMP/SSH policers tuned (show lpts pifib hardware police)
[ ] LPTS-vs-CoPP mapping documented (on-by-default + NPU/pIFIB vs RP service-policy)
[ ] IS-IS HMAC-MD5 key-chain: hello-password (per-intf) + lsp-password (per-level)
[ ] OSPF auth: area-level + interface-level override; hello-auth blocks rogue adjacency
[ ] BGP auth: TCP-AO preferred (SHA), MD5 fallback shown
[ ] GTSM (ttl-security) on all directly-connected eBGP
[ ] RPKI: validator connected; drop Invalid / prefer Valid / accept NotFound
[ ] RTBH: community 65100:666 → next-hop → Null0 (destination-based)
[ ] Flowspec: drop (UDP/53), rate-limit (TCP SYN), redirect (VRF SCRUB) — with local-install
[ ] Bogon prefix-set + AS-PATH (private ASN / long-path) filters on eBGP
[ ] maximum-prefix cap + warning + restart per eBGP peer
[ ] MACsec: macsec key-chain + MKA policy applied to E-R6↔Gar-R7; session Secured
[ ] TS1: RPKI Invalid (ROA mismatch) diagnosed and fixed
[ ] TS2: Flowspec not-installed diagnosed → missing local-install added
```
