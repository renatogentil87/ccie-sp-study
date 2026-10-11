# CCIE SP Workbook 03 — BGP Advanced

**Domain:** Domain 1 — Core Routing (25%)
**Platform:** IOS-XRv 9000 7.11.1 (EVE-NG)
**Focus:** iBGP + Route Reflectors, eBGP Inter-AS, path manipulation, advanced BGP features, troubleshooting
**Prereq:** IGP (IS-IS L2) + MPLS transport converged inside each AS; Loopback0 reachable within each AS.

> **Note:** This workbook drills the full BGP control plane for a 3-ISP interconnect: hierarchical iBGP with RRs carrying VPNv4, inter-AS eBGP with multihop VPNv4 between RRs (Option C-style), best-path manipulation for transit engineering, and the scaling/convergence features (Add-Path, PIC, AIGP, RT-Constraint, conditional advertisement) that make a real SP core resilient. **All route-policies are IOS-XR RPL** (`route-policy … if/then/else … set … end-policy`).

---

## Topology

Three service providers, each an autonomous system, interconnected at their ASBRs.

```
        EMERALD  AS 65100                    GOLD  AS 65300                    GARNET  AS 65200
   RR = E-R5 (6.6.6.6)                 RR = G-R4 (24.24.24.24)           RR = Gar-R6 (16.16.16.16)

   E-R1 1.1.1.1                          G-R4 24.24.24.24 (RR)             Gar-R1 11.11.11.11
   E-R2 2.2.2.2                          G-R5 25.25.25.25                  Gar-R2 12.12.12.12
   E-R3, E-R4                               G-R3                                 Gar-R3, Gar-R4, Gar-R5
   E-R6 5.5.5.5                        G-R1 21.21.21.21                    Gar-R7 17.17.17.17
   E-R5 6.6.6.6 (RR)                    G-R2 22.22.22.22

                        E-R6 ══════════════ G-R4        (Emerald ↔ Gold,   eBGP)
                        E-R6 ══════════════ Gar-R7        (Emerald ↔ Garnet, eBGP, direct)
                        G-R5 ══════════════ Gar-R7        (Gold   ↔ Garnet,  eBGP)
```

| SP | AS | RR | PEs | ASBRs |
|----|----|----|-----|-------|
| Emerald | 65100 | E-R5 (6.6.6.6) | E-R1 (1.1.1.1), E-R2 (2.2.2.2) | E-R6 (5.5.5.5) |
| Gold | 65300 | G-R4 (24.24.24.24) | G-R1 (21.21.21.21), G-R2 (22.22.22.22) | G-R4 (24.24.24.24), G-R5 (25.25.25.25) |
| Garnet | 65200 | Gar-R6 (16.16.16.16) | Gar-R1 (11.11.11.11), Gar-R2 (12.12.12.12) | Gar-R7 (17.17.17.17) |

**Customers:** A (AS 65012) = Emerald + Gold; B (AS 65013) = Gold + Garnet; C (EVPN) = Gold + Garnet.

**Inter-AS links (example addressing):**

| Link | Left | Right |
|------|------|-------|
| E-R6 ↔ Gar-R7 | 10.0.12.5/30 | 10.0.12.6/30 |
| E-R6 ↔ G-R4 | 10.0.13.5/30 | 10.0.13.21/30 |
| G-R5 ↔ Gar-R7 | 10.0.42.22/30 | 10.0.42.16/30 |

---

## Section 1 — iBGP + Route Reflectors

### Task 1.1
- Design the iBGP mesh for **Emerald** as a route-reflector topology: **E-R5 (6.6.6.6)** is the RR; E-R1, E-R2, and E-R6 are RR clients.
- All iBGP sessions peer on **Loopback0** with **update-source Loopback0**.

**Solution**

iBGP requires a full mesh (n·(n-1)/2 sessions) because iBGP-learned routes are not re-advertised to other iBGP peers (loop prevention). A **Route Reflector** breaks that rule: the RR *reflects* routes between clients, collapsing the mesh to a hub-and-spoke and scaling O(n). Peering on **Loopback0** decouples the session from any single physical link — the IGP provides multiple paths to the loopback, so a link failure doesn't drop the BGP session. `update-source Loopback0` forces the TCP source to match the loopback the neighbor expects.

```
! ===== E-R5 (Emerald RR, 6.6.6.6) =====
router bgp 65100
 bgp router-id 6.6.6.6
 address-family ipv4 unicast
 !
 neighbor-group EMERALD-RR-CLIENTS
  remote-as 65100
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 1.1.1.1
  use neighbor-group EMERALD-RR-CLIENTS
 !
 neighbor 2.2.2.2
  use neighbor-group EMERALD-RR-CLIENTS
 !
 neighbor 5.5.5.5
  use neighbor-group EMERALD-RR-CLIENTS
 !
!

! ===== E-R1 (RR client, 1.1.1.1) — representative client =====
router bgp 65100
 bgp router-id 1.1.1.1
 neighbor 6.6.6.6
  remote-as 65100
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!
```

**Verification**
- `show bgp ipv4 unicast summary` on E-R5 — 3 neighbors (1.1.1.1, 2.2.2.2, 5.5.5.5) in `Established`.
- `show bgp neighbor 6.6.6.6 | i Route-Reflector` on E-R1 — session up; on E-R5 the client shows `Route-Reflector Client: TRUE`.
- `show tcp brief | i :179` — TCP source/dest addresses are the Loopback0 /32s.

### Task 1.2
- Repeat the RR design for **Gold** (RR = **G-R4 24.24.24.24**; clients G-R5, G-R1, G-R2, G-R3) and **Garnet** (RR = **Gar-R6 16.16.16.16**; clients Gar-R1, Gar-R2, Gar-R7).

**Solution**

Each AS runs its own independent RR cluster. Gold uses its ASBR as the RR (a common design where the border node doubles as the reflector); Garnet uses a dedicated Gar-R6. The pattern is identical — the RR is the hub, every other iBGP speaker is a client. Keeping the RR on Loopback0 with `update-source` is mandatory in all three.

```
! ===== G-R4 (Gold RR, 24.24.24.24) =====
router bgp 65300
 bgp router-id 24.24.24.24
 neighbor-group GOLD-CLIENTS
  remote-as 65300
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 25.25.25.25
  use neighbor-group GOLD-CLIENTS
 neighbor 21.21.21.21
  use neighbor-group GOLD-CLIENTS
 neighbor 22.22.22.22
  use neighbor-group GOLD-CLIENTS
!

! ===== Gar-R6 (Garnet RR, 16.16.16.16) =====
router bgp 65200
 bgp router-id 16.16.16.16
 neighbor-group GARNET-CLIENTS
  remote-as 65200
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
 !
 neighbor 11.11.11.11
  use neighbor-group GARNET-CLIENTS
 neighbor 12.12.12.12
  use neighbor-group GARNET-CLIENTS
 neighbor 17.17.17.17
  use neighbor-group GARNET-CLIENTS
!
```

**Verification**
- `show bgp all all summary` on each RR — all clients `Established`.
- `show bgp ipv4 unicast <client-loopback>` — client's loopback reflected to the other clients.

### Task 1.3
- Add the **VPNv4 address family** on every Emerald RR/PE and enable `route-reflector-client` under `address-family vpnv4 unicast` so L3VPN routes are reflected.
- Enable `retain route-target all` on the RR so it keeps VPNv4 routes even without a matching import RT locally.

**Solution**

A pure RR imports no VRFs, so by default it would **drop** VPNv4 routes whose RTs it doesn't import — breaking reflection. `retain route-target all` tells the RR to keep every VPNv4 route regardless of RT so it can reflect them. The `route-reflector-client` flag must be set **per address family**: reflecting IPv4 does not reflect VPNv4 unless you say so explicitly. This is the single most common "VPNv4 not reflected" bug (see Section 5, Task 5.3).

```
! ===== E-R5 (Emerald RR) — add VPNv4 =====
router bgp 65100
 address-family vpnv4 unicast
  retain route-target all
 !
 neighbor-group EMERALD-RR-CLIENTS
  address-family vpnv4 unicast
   route-reflector-client
  !
 !
!

! ===== E-R1 (client) — add VPNv4 =====
router bgp 65100
 address-family vpnv4 unicast
 !
 neighbor 6.6.6.6
  address-family vpnv4 unicast
  !
 !
!
```

**Verification**
- `show bgp vpnv4 unicast summary` — VPNv4 AF up between RR and clients.
- `show bgp vpnv4 unicast rd <rd> <prefix>` on a client — route present with `Received from a RR-client`.
- `show bgp vpnv4 unicast` on the RR — VPNv4 NLRIs retained even for RTs the RR doesn't import.

### Task 1.4
- Set a **cluster-id** on each RR so a future second RR in the same cluster shares it (cluster-list loop prevention).
- Emerald cluster-id `6.6.6.6`; Gold `24.24.24.24`; Garnet `16.16.16.16`.

**Solution**

When a route is reflected, the RR prepends its **cluster-id** to the CLUSTER_LIST. A second RR seeing its own cluster-id in the list discards the update — this is the RR equivalent of AS-path loop prevention for iBGP. RRs in the **same cluster** must share the cluster-id (so they ignore each other's reflections and rely on ORIGINATOR_ID); RRs in **different clusters** use distinct IDs so they *do* reflect to each other for redundancy. By default the cluster-id equals the router-id; setting it explicitly makes clustered-RR designs deterministic.

```
router bgp 65100
 bgp cluster-id 6.6.6.6
!
router bgp 65300
 bgp cluster-id 24.24.24.24
!
router bgp 65200
 bgp cluster-id 16.16.16.16
!
```

**Verification**
- `show bgp vpnv4 unicast rd <rd> <prefix> detail` on a client — shows `Originator: <PE-id>, Cluster list: <cluster-id>`.
- Inject a route and confirm no loop / no duplicate; `show bgp <prefix>` shows a single reflected path with the ORIGINATOR_ID of the true origin.

### Task 1.5
- Configure **next-hop-self** so every PE sets itself as the BGP next-hop for routes it originates into iBGP (customer/eBGP routes and VPNv4).
- Confirm iBGP peers resolve the next-hop via the IGP loopback (recursion), not the external link address.

**Solution**

An eBGP-learned route carries the *external* peer's address as next-hop. Advertised into iBGP unchanged, interior routers would need that external subnet in their IGP — which they don't have, so the route is unusable (`next-hop unreachable`, see Section 5 Task 5.2). **next-hop-self** rewrites the next-hop to the advertising router's Loopback0, which the IGP *does* carry, so every iBGP speaker can recurse to it over MPLS. For VPNv4 this is what makes the label-switched path terminate on the correct egress PE.

```
! ===== E-R6 (Emerald) — rewrite eBGP next-hops into iBGP =====
router bgp 65100
 neighbor 6.6.6.6
  address-family ipv4 unicast
   next-hop-self
  !
  address-family vpnv4 unicast
   next-hop-self
  !
 !
!
```

> On an RR that is **not** in the forwarding path, do **not** set next-hop-self for reflected routes — the RR must preserve the originating PE's next-hop. Use next-hop-self only where the router is the actual traffic ingress/egress (PEs, ASBRs).

**Verification**
- `show bgp ipv4 unicast <ext-prefix>` on a remote PE — next-hop is the ASBR's `5.5.5.5`, not the inter-AS link.
- `show cef <ext-prefix>` — recurses via the IGP path to 5.5.5.5.
- `show bgp vpnv4 unicast rd <rd> <prefix>` — next-hop is the egress PE loopback; `show mpls forwarding` confirms the LSP.

---

## Section 2 — eBGP Inter-AS

### Task 2.1
- Configure **direct eBGP** on the E-R6 ↔ Gar-R7 link (Emerald AS 65100 ↔ Garnet AS 65200) peering on the **directly-connected interface addresses**.

**Solution**

eBGP defaults to **TTL=1** and peers on the connected interface — no update-source, no multihop. This is the classic Inter-AS Option A/B border session. Peering on the interface (not loopback) keeps TTL=1 and gives immediate transport failure detection when the link drops.

```
! ===== E-R6 (Emerald, 5.5.5.5) =====
router bgp 65100
 neighbor 10.0.12.6
  remote-as 65200
  description eBGP to Gar-R7 (Garnet)
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
route-policy PASS-ALL
  pass
end-policy
!

! ===== Gar-R7 (Garnet, 17.17.17.17) =====
router bgp 65200
 neighbor 10.0.12.5
  remote-as 65100
  description eBGP to E-R6 (Emerald)
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

> IOS-XR requires an **inbound and outbound route-policy on eBGP neighbors** or all prefixes are dropped by default (`bgp bestpath` sees nothing). `PASS-ALL` is a placeholder; production policies filter.

**Verification**
- `show bgp ipv4 unicast summary` — neighbor `Established`, `remote-as` differs (eBGP).
- `show bgp neighbor 10.0.12.6 | i Connections|Multihop` — single-hop, TTL 1.

### Task 2.2
- Configure **eBGP E-R6 ↔ G-R4** (Emerald AS 65100 ↔ Gold AS 65300) and **G-R5 ↔ Gar-R7** (Gold AS 65300 ↔ Garnet AS 65200), both directly connected.

**Solution**

Same single-hop eBGP pattern, completing the triangle so every AS has two inter-AS neighbors and thus a direct path plus a transit path to any other AS. This redundancy is what Section 3's path manipulation engineers (prefer the Gold transit vs. the direct link).

```
! ===== E-R6 (Emerald) → G-R4 (Gold) =====
router bgp 65100
 neighbor 10.0.13.21
  remote-as 65300
  description eBGP to G-R4 (Gold)
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!

! ===== G-R5 (Gold, 25.25.25.25) → Gar-R7 (Garnet) =====
router bgp 65300
 neighbor 10.0.42.16
  remote-as 65200
  description eBGP to Gar-R7 (Garnet)
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

**Verification**
- `show bgp ipv4 unicast summary` on E-R6, G-R4, G-R5, Gar-R7 — all inter-AS sessions `Established`.
- `show bgp ipv4 unicast <remote-loopback>` — reachable via two different AS-paths (direct vs. transit).

### Task 2.3
- Build **multihop eBGP for VPNv4 between the RRs** (Inter-AS Option C-style): E-R5 (Emerald RR, 6.6.6.6) ↔ G-R4 (Gold RR, 24.24.24.24), peering **loopback-to-loopback** across the AS boundary, exchanging **labeled VPNv4** without importing VRFs on the ASBRs.

**Solution**

Inter-AS **Option C** keeps VPNv4 off the ASBRs entirely: ASBRs exchange only **labeled IPv4 loopbacks** (BGP-LU / `label-mode`) so the two ASes' PE loopbacks become reachable end-to-end, and the **RRs peer directly over multihop eBGP** to exchange VPNv4 routes with the originating PE's next-hop preserved. Because the RR loopbacks are several hops apart across the AS boundary, the session needs **ebgp-multihop**. This scales far better than Option A/B — the ASBRs hold no VPN state, and only the RRs carry the VPNv4 table.

```
! ===== Prereq: ASBRs advertise labeled loopbacks (BGP-LU) so RR loopbacks are reachable inter-AS =====
! ===== E-R6 (Emerald) — send labeled IPv4 for loopbacks to G-R4 =====
router bgp 65100
 neighbor 10.0.13.21
  remote-as 65300
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
 address-family ipv4 unicast
  network 6.6.6.6/32     ! Emerald RR loopback, labeled
  allocate-label all
 !
!

! ===== E-R5 (Emerald RR, 6.6.6.6) — multihop eBGP VPNv4 to Gold RR =====
router bgp 65100
 neighbor 24.24.24.24
  remote-as 65300
  description Multihop eBGP VPNv4 to Gold RR (G-R4)
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-unchanged
  !
 !
!

! ===== G-R4 (Gold RR, 24.24.24.24) — mirror =====
router bgp 65300
 neighbor 6.6.6.6
  remote-as 65100
  description Multihop eBGP VPNv4 to Emerald RR (E-R5)
  ebgp-multihop 255
  update-source Loopback0
  address-family vpnv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-unchanged
  !
 !
!
```

> `next-hop-unchanged` preserves the originating PE's next-hop across the eBGP boundary (Option C requirement) so the inter-AS LSP terminates on the true egress PE. This depends on the labeled-loopback reachability from the prereq.

**Verification**
- `show bgp vpnv4 unicast summary` on E-R5 — neighbor 24.24.24.24 `Established` (eBGP, VPNv4).
- `show bgp ipv4 labeled-unicast 24.24.24.24/32` on E-R6 — Gold RR loopback learned with a label.
- `show bgp vpnv4 unicast rd <gold-rd> <prefix> detail` on E-R5 — next-hop = originating Gold PE loopback (unchanged).
- End-to-end `traceroute` in a shared VRF crosses both ASes over MPLS.

### Task 2.4
- Harden the multihop RR session with **TTL security** (GTSM) instead of a plain `ebgp-multihop` hop count.

**Solution**

`ebgp-multihop N` only widens the accepted TTL range — it does nothing to *authenticate* the hop distance and can be spoofed. **TTL security (GTSM, RFC 5082)** flips the check: it sends packets with TTL=255 and accepts only packets arriving with TTL ≥ (255 − hops). A spoofed packet from far away arrives with a lower TTL and is dropped in hardware. For a known-distance multihop peer this is both a security control and a cheap DoS filter. Note GTSM and `ebgp-multihop` are mutually exclusive on the same neighbor — GTSM implies multihop.

```
! ===== E-R5 — replace ebgp-multihop with ttl-security =====
router bgp 65100
 neighbor 24.24.24.24
  no ebgp-multihop 255
  ttl-security
  address-family vpnv4 unicast
  !
 !
!
! Mirror on G-R4. Ensure the actual hop count ≤ the GTSM window;
! XR auto-derives the accepted min TTL from the session.
```

**Verification**
- `show bgp neighbor 24.24.24.24 | i TTL` — `External BGP neighbor with TTL security` (GTSM enabled).
- Session stays `Established`; a source more hops away than the window cannot bring up a spoofed session.

### Task 2.5
- Contrast when to use **ebgp-multihop** vs **ttl-security** and document the decision for each inter-AS session in this topology.

**Solution**

Use **ebgp-multihop** when the peer is legitimately more than one hop away and you cannot pin the exact distance (e.g., dynamic multi-path loopback peering) — it's permissive. Use **ttl-security (GTSM)** when the peer distance is fixed and known and you want spoofing protection — it's a security control, not just a reachability enabler. In this lab: direct ASBR↔ASBR sessions (Tasks 2.1–2.2) need neither (single-hop, TTL=1 is its own protection); the RR-to-RR multihop VPNv4 session (Task 2.3) is a fixed-distance loopback peering, so **GTSM is the better choice** (Task 2.4).

**Verification**
- `show bgp neighbor <peer> | i Multihop|TTL` on each session — single-hop sessions show no multihop; the RR session shows GTSM.

---

## Section 3 — BGP Path Manipulation

### Task 3.1
- Engineer Emerald to **prefer the Gold transit path** (via G-R4) over the **direct** E-R6↔Gar-R7 path for reaching Garnet's prefixes, using **LOCAL_PREF**.

**Solution**

LOCAL_PREF is AS-wide and evaluated **before** AS-path length, so it overrides the fact that the direct path has a shorter AS-path (65200 vs. 65300 65200). It controls **outbound** traffic (how *our* AS exits). Setting a higher local-pref on routes learned via the Gold-transit neighbor makes every Emerald router prefer that exit. In RPL you `set local-preference` in an inbound policy on the ASBR facing Gold.

```
route-policy GARNET-VIA-GOLD-IN
  if as-path passes-through '65200' then
    set local-preference 200
  else
    set local-preference 100
  endif
  pass
end-policy
!
router bgp 65100
 neighbor 10.0.13.21          ! E-R6 → G-R4 (Gold)
  address-family ipv4 unicast
   route-policy GARNET-VIA-GOLD-IN in
  !
 !
 neighbor 10.0.12.6           ! E-R6 → Gar-R7 (direct Garnet) — leave default 100
  address-family ipv4 unicast
   route-policy PASS-ALL in
  !
 !
!
```

**Verification**
- `show bgp ipv4 unicast <garnet-prefix>` on any Emerald router — best path has `localpref 200`, next-hop toward Gold; AS-path is the longer `65300 65200`.
- `traceroute <garnet-host>` egresses via G-R4 (Gold), not the direct link.

### Task 3.2
- Make the **direct** E-R6↔Gar-R7 link *less preferred inbound* (so Garnet reaches Emerald via Gold) using **AS-PATH prepend** outbound on the direct link.

**Solution**

You cannot set LOCAL_PREF on a neighbor's router — to influence **inbound** traffic you manipulate attributes the neighbor evaluates: primarily **AS-path length**. Prepending our own AS multiple times on the *direct* advertisement makes it longer than the Gold-transit advertisement, so Garnet's best-path picks the Gold path. Prepend is the blunt, universally-honored inbound tool (MED is often ignored across AS boundaries — see 3.3).

```
route-policy PREPEND-DIRECT-OUT
  prepend as-path 65100 3
  pass
end-policy
!
router bgp 65100
 neighbor 10.0.12.6           ! E-R6 → Gar-R7 direct
  address-family ipv4 unicast
   route-policy PREPEND-DIRECT-OUT out
  !
 !
!
```

**Verification**
- On Gar-R7 (Garnet): `show bgp ipv4 unicast <emerald-prefix>` — direct path AS-path `65100 65100 65100 65100`, Gold path `65300 65100`; best-path = Gold (shorter).
- Return traffic from Garnet enters Emerald via G-R4/Gold.

### Task 3.3
- Use **MED** to influence **inbound** traffic across the two links to a *single* neighboring AS (Gold), steering Gold to prefer one Emerald entry point.

**Solution**

MED ("metric") is the tiebreaker Gold uses **only when comparing paths from the same neighbor AS** (65100) — lower MED wins. It's the right tool when you have two links to the *same* AS and want to hint a preferred entry, but it's not comparable across different neighbor ASes and is frequently stripped/ignored by peers, so it's weaker than prepend. Set a lower MED on the preferred link's outbound advertisement.

```
route-policy MED-PREFER-OUT
  set med 50
  pass
end-policy
!
route-policy MED-BACKUP-OUT
  set med 200
  pass
end-policy
!
router bgp 65100
 neighbor 10.0.13.21          ! primary entry from Gold
  address-family ipv4 unicast
   route-policy MED-PREFER-OUT out
  !
 !
!
! (If a second link to Gold existed, apply MED-BACKUP-OUT there.)
```

> For MED to be compared across all paths regardless of neighbor AS, enable `bgp bestpath med always` on the receiving AS (Gold). Otherwise MED only breaks ties among same-AS paths.

**Verification**
- On the Gold receiving router: `show bgp ipv4 unicast <emerald-prefix>` — the path with `metric 50` is best over `metric 200`.
- Confirm MED survived: `show bgp ... detail` shows the MED attribute on the received path.

### Task 3.4
- Implement **community-based routing**: tag Emerald customer routes with a **well-known** community (NO_EXPORT) and an **extended** community, then act on the community at the RR/egress to set LOCAL_PREF.

**Solution**

Communities decouple *marking* (at ingress) from *action* (elsewhere) so policy scales without per-prefix config. **NO_EXPORT** (well-known) keeps a prefix inside the AS — never advertised to eBGP peers. A custom community like `65100:200` is a policy tag: the RR matches it and sets local-pref, so a single ingress tag drives network-wide behavior. Extended communities carry structured values (e.g., route-targets); here we also match one to demonstrate typed policy.

```
community-set CUST-HIPRI
  65100:200
end-set
!
extcommunity-set rt CUST-RT
  65100:1
end-set
!
route-policy TAG-CUSTOMER-IN
  set community (65100:200, no-export) additive
  pass
end-policy
!
route-policy ACT-ON-COMMUNITY-IN
  if community matches-any CUST-HIPRI then
    set local-preference 300
  endif
  pass
end-policy
!
router bgp 65100
 neighbor <customer-ce>
  address-family ipv4 unicast
   route-policy TAG-CUSTOMER-IN in
  !
 !
 neighbor 6.6.6.6              ! toward RR: honor the tag
  address-family ipv4 unicast
   route-policy ACT-ON-COMMUNITY-IN in
   send-community-ebgp
  !
 !
!
```

**Verification**
- `show bgp ipv4 unicast <cust-prefix>` — communities `65100:200 no-export` attached.
- The prefix is **not** advertised to eBGP neighbors (NO_EXPORT honored): absent from `show bgp neighbor <ebgp-peer> advertised-routes`.
- Where `ACT-ON-COMMUNITY-IN` applies, best path shows `localpref 300`.

### Task 3.5
- Write a consolidated **IOS-XR RPL route-policy** using full `if/elseif/else/then … set … end-policy` structure (with `apply` for a nested policy) that combines the manipulations above, and document the RPL building blocks.

**Solution**

RPL is IOS-XR's structured policy language: **route-policy … end-policy** wraps `if <condition> then <actions> elseif … else … endif`, terminated by `pass` (accept) or `drop`. Conditions match on `destination in <prefix-set>`, `as-path in/passes-through`, `community matches-any/matches-every`, `med`, etc. Actions `set local-preference / med / community / next-hop`, `prepend as-path`, `drop`, `pass`. **`apply <policy>`** calls a nested policy (composition/reuse). Sets (`prefix-set`, `community-set`, `as-path-set`, `extcommunity-set`) are named, reusable match lists. `pass` continues evaluation (multiple `set`s accumulate) whereas `drop` is terminal.

```
prefix-set GARNET-PREFIXES
  172.16.0.0/16 le 32
end-set
!
community-set HIPRI
  65100:200
end-set
!
route-policy SET-BASE-ATTRS
  set origin igp
  pass
end-policy
!
route-policy WB03-INBOUND
  apply SET-BASE-ATTRS
  if destination in GARNET-PREFIXES and as-path passes-through '65200' then
    set local-preference 200
    set community (65100:200) additive
  elseif community matches-any HIPRI then
    set local-preference 300
  else
    set local-preference 100
  endif
  pass
end-policy
!
route-policy WB03-OUTBOUND
  if destination in GARNET-PREFIXES then
    prepend as-path 65100 3
    set med 50
  endif
  pass
end-policy
!
router bgp 65100
 neighbor 10.0.13.21
  address-family ipv4 unicast
   route-policy WB03-INBOUND in
   route-policy WB03-OUTBOUND out
  !
 !
!
```

**Verification**
- `show rpl route-policy WB03-INBOUND` — policy parses; nested `apply` resolves.
- `show bgp ipv4 unicast <garnet-prefix>` — reflects the branch taken (localpref/community/prepend/med as configured).
- `show rpl route-policy WB03-INBOUND detail` — lists referenced sets (GARNET-PREFIXES, HIPRI).

---

## Section 4 — BGP Advanced Features

### Task 4.1
- Enable **BGP Add-Path** on the Emerald RR (E-R5) so it advertises **best + additional paths** for a multi-homed prefix, restoring path diversity to the clients.

**Solution**

An RR runs best-path and reflects only **one** path per NLRI, so clients never see a backup — killing fast failover. **Add-Path (RFC 7911)** tags multiple paths with path-IDs so the RR advertises several per prefix. Configure a selection policy (e.g., `additional-paths selection route-policy` choosing best-N or backup 1) and enable send/receive capability. This is the prerequisite for PIC-Edge (4.2): PIC needs a backup path to exist and be visible.

```
route-policy ADD-PATH-2
  set path-selection all advertise 2     ! advertise up to 2 paths
end-policy
!
router bgp 65100
 address-family ipv4 unicast
  additional-paths receive
  additional-paths send
  additional-paths selection route-policy ADD-PATH-2
 !
 address-family vpnv4 unicast
  additional-paths receive
  additional-paths send
  additional-paths selection route-policy ADD-PATH-2
 !
!
```

**Verification**
- Before: `show bgp ipv4 unicast <mh-prefix>` on a client — ONE path.
- After: same command shows **two** paths, each with a distinct path-id.
- `show bgp neighbor <client> | i Additional-paths` — send/receive negotiated.

### Task 4.2
- Enable **BGP PIC Edge** for VPNv4 so an egress-PE / next-hop failure fails over in one FIB operation, independent of prefix count.

**Solution**

Without PIC, a next-hop failure triggers per-prefix best-path recompute — minutes at scale. **PIC** uses a hierarchical FIB with indirection: prefixes point at a shared next-hop object holding **primary + pre-installed backup**, so on failure you flip ONE object and all prefixes follow — sub-second, prefix-independent. **PIC-Edge** protects the egress PE via a backup BGP next-hop (from Add-Path/unique-RD). PIC needs the backup to be *visible* (4.1), so 4.1 → 4.2 form a chain.

```
router bgp 65100
 address-family vpnv4 unicast
  additional-paths receive
  additional-paths install backup    ! pre-install backup path in FIB (PIC-Edge)
 !
!
! Also enable per-prefix TI-LFA / PIC-core in the IGP for path-to-next-hop protection.
```

**Verification**
- `show cef vrf <vrf> <prefix> detail` — primary **and** backup next-hop pre-installed (`repair:`/`backup`).
- Continuous CE-to-CE traffic; fail the primary egress PE / next-hop → 0–1 packets lost, convergence independent of table size.

### Task 4.3
- Configure **AIGP** (Accumulated IGP metric) so the IGP cost is carried and accumulated **across the AS boundary** between the RRs, letting best-path choose the truly lowest end-to-end IGP cost.

**Solution**

BGP normally hides IGP cost across AS boundaries — best-path only sees the local IGP metric to the next-hop. **AIGP (RFC 7311)** carries the IGP metric as a BGP attribute and **accumulates** it hop-by-hop across a defined AIGP administrative domain (typically a set of cooperating ASes / RR sessions), so best-path can compare true end-to-end cost. It's used in seamless-MPLS and multi-AS designs where all ASes are one administrative domain. AIGP is evaluated high in best-path (just after weight/local-pref-era steps), so it strongly influences selection.

```
route-policy SET-AIGP
  set aigp-metric igp-cost      ! seed AIGP from local IGP metric
  pass
end-policy
!
router bgp 65100
 neighbor 24.24.24.24           ! multihop RR session to Gold
  address-family vpnv4 unicast
   aigp
   route-policy SET-AIGP in
  !
 !
!
```

**Verification**
- `show bgp vpnv4 unicast rd <rd> <prefix> detail` — `aigp-metric: <value>` present and increasing across the boundary.
- Best path selects the lower accumulated AIGP even when AS-path lengths are equal.

### Task 4.4
- Enable **RT-Constraint (RTC, RFC 4684)** between the RRs so each RR only receives the VPNv4 routes whose route-targets its clients actually import.

**Solution**

By default an RR with `retain route-target all` receives *every* VPNv4 route — huge tables and wasted reflection. **RT-Constraint** adds an `rtfilter` address family: each router advertises the **set of RTs it imports**, and the RR sends only VPNv4 routes matching those RTs. This turns push-everything into pull-what-you-need, dramatically cutting inter-AS VPNv4 volume — essential at scale and directly relevant to the 3-customer VPN spread here (A/B/C each need different RTs).

```
router bgp 65100
 address-family ipv4 rt-filter
 !
 neighbor 24.24.24.24           ! RR-to-RR
  address-family ipv4 rt-filter
  !
 !
 neighbor 6.6.6.6               ! (on a PE) advertise its imported RTs to the RR
  address-family ipv4 rt-filter
  !
 !
!
```

**Verification**
- `show bgp ipv4 rt-filter summary` — rtfilter AF up; RTs exchanged.
- `show bgp ipv4 rt-filter` — the RT membership advertised by clients.
- On the RR: VPNv4 table now holds only routes with RTs some client imports (compare table size before/after).

### Task 4.5
- Configure **conditional route advertisement** on E-R6: advertise a backup aggregate to Garnet **only if** the primary Gold-transit path is *not* present (advertise-map / non-exist-map behavior).

**Solution**

Conditional advertisement advertises a prefix based on the **presence or absence** of another prefix in the BGP table — used for backup/failover routing. `non-exist-map` logic: advertise the backup **only when** the tracked (primary) prefix is missing; when the primary returns, withdraw the backup. This automates "use the backup link only during a failure" without static tricks. On IOS-XR this is expressed with a match on the tracked route and an `advertise`/conditional construct tied to a route-policy.

```
prefix-set PRIMARY-TRACK
  10.30.0.0/16          ! primary Gold-transit prefix to watch
end-set
!
prefix-set BACKUP-AGG
  10.99.0.0/16          ! backup aggregate to advertise only on failure
end-set
!
route-policy ADV-BACKUP
  if destination in BACKUP-AGG then
    pass
  else
    drop
  endif
end-policy
!
route-policy TRACK-PRIMARY
  if destination in PRIMARY-TRACK then
    pass
  else
    drop
  endif
end-policy
!
router bgp 65100
 neighbor 10.0.12.6            ! E-R6 → Gar-R7 (Garnet)
  address-family ipv4 unicast
   advertise conditional route-policy ADV-BACKUP non-exist-map TRACK-PRIMARY
  !
 !
!
```

> Syntax varies by XR release; the concept is: **advertise ADV-BACKUP only while TRACK-PRIMARY does not exist**. Verify exact keyword with `advertise ?` on the target image.

**Verification**
- Steady state (primary present): `show bgp neighbor 10.0.12.6 advertised-routes` — backup aggregate **absent**.
- Withdraw the primary (shut the Gold path): backup aggregate **appears** in advertised-routes.
- Restore primary: backup is withdrawn again.

---

## Section 5 — BGP Troubleshooting

### Task 5.1
- **Symptom:** an iBGP/eBGP neighbor is stuck in **Active** (never reaches Established). Diagnose the TCP/transport issue and fix it.

**Solution**

**Active** means BGP is trying to open the TCP session outbound but it never completes — this is a **transport-layer** problem, not a policy problem. Systematic causes: (1) **no route to the neighbor** (loopback not in IGP / next-hop unreachable); (2) **update-source mismatch** — you peer to a loopback but don't source from your own loopback, so the far end's ACL/`neighbor` doesn't match the source IP; (3) **eBGP-multihop needed** but not configured (loopback peering across >1 hop); (4) **ACL/firewall blocking TCP 179**; (5) **wrong remote-as / typo in neighbor IP**; (6) MTU/PMTUD blackhole on the open. Walk the list: reachability → source → hop count → filters → AS.

```
! Diagnose
show bgp ipv4 unicast summary            ! state = Active/Idle
show bgp neighbor <peer>                 ! last error, transport
ping <peer-loopback> source Loopback0    ! reachability + correct source
show route <peer-loopback>               ! is the loopback in the RIB?
show access-lists                        ! TCP/179 blocked?

! Common fixes
router bgp <asn>
 neighbor <peer-loopback>
  update-source Loopback0                ! (1) source from loopback
  ! ebgp-multihop 2   or  ttl-security   ! (2) if eBGP loopback peering
 !
!
! + ensure the peer loopback is advertised into the IGP.
```

**Verification**
- `ping <peer> source Loopback0` succeeds first.
- `show bgp ipv4 unicast summary` — state transitions Active → OpenSent → **Established**.
- `show bgp neighbor <peer> | i state|Last` — no repeating transport errors.

### Task 5.2
- **Symptom:** routes are **received** (present in `show bgp`) but **not installed** in the RIB/FIB. Diagnose and fix.

**Solution**

A BGP route is only a best-path candidate if its **next-hop is resolvable** in the RIB. If it shows in `show bgp <prefix>` but is marked **not valid / inaccessible** and never enters the RIB, the classic cause is an **unreachable next-hop** — typically an eBGP next-hop (the external link subnet) advertised into iBGP unchanged, which interior routers can't resolve. The fix is **next-hop-self** on the advertising ASBR/PE (Section 1 Task 1.5), or redistributing/carrying the next-hop subnet in the IGP (worse). Secondary causes: next-hop points at a down interface; recursion loop; VPNv4 next-hop not label-reachable (no LSP).

```
! Diagnose
show bgp ipv4 unicast <prefix>           ! look for "(inaccessible)" / not > best
show bgp ipv4 unicast <prefix> detail    ! Next Hop: <addr>  -- is it resolvable?
show route <next-hop-addr>               ! RIB lookup on the next-hop
show cef <prefix>                        ! is it in FIB?

! Fix: rewrite next-hop to a loopback the IGP carries
router bgp <asn>
 neighbor <ibgp-peer>
  address-family ipv4 unicast
   next-hop-self
  !
  address-family vpnv4 unicast
   next-hop-self                          ! for VPNv4 label path
  !
 !
!
```

**Verification**
- `show bgp ipv4 unicast <prefix>` — path now `>` (best, valid); next-hop is a reachable loopback.
- `show route <prefix>` — route present via BGP; `show cef <prefix>` — installed in FIB.
- For VPNv4: `show mpls forwarding` confirms the LSP to the (rewritten) next-hop.

### Task 5.3
- **Symptom:** VPNv4 routes are **not being reflected** by the RR to its clients. Diagnose the missing address-family / RT-retain issue and fix.

**Solution**

Two RR-specific traps: (1) **`route-reflector-client` is per address family** — reflecting IPv4 does **not** reflect VPNv4; if the flag is missing under `address-family vpnv4 unicast`, VPNv4 routes are received but never reflected. (2) A pure RR imports no VRFs, so without **`retain route-target all`** it **drops** VPNv4 routes whose RTs it doesn't locally import — nothing to reflect. Also check the VPNv4 AF is actually **enabled** on both the RR-client sessions and that `send-community extended` (RTs travel as extended communities) is on. Fix by adding the AF, the per-AF RR-client flag, and RT retention.

```
! Diagnose
show bgp vpnv4 unicast summary                    ! is the VPNv4 AF even up to clients?
show bgp neighbor <client> | i route-reflector    ! per-AF RR-client set?
show bgp vpnv4 unicast rd <rd> <prefix>           ! present on RR but not reflected?

! Fix
router bgp <asn>
 address-family vpnv4 unicast
  retain route-target all                          ! (2) keep all VPNv4 for reflection
 !
 neighbor-group <clients>
  address-family vpnv4 unicast
   route-reflector-client                          ! (1) per-AF RR-client flag
  !
 !
!
```

**Verification**
- `show bgp neighbor <client> | i Route-Reflector` under VPNv4 — `Route-Reflector Client: TRUE`.
- On a client: `show bgp vpnv4 unicast rd <rd> <prefix>` — route now present, flagged reflected with `Originator`/`Cluster list`.
- `show bgp vpnv4 unicast summary` on the RR — non-zero prefixes sent to each client.

---

## CCIE Challenge Tasks

### Challenge A — Full 3-AS VPN reachability
- Bring up Customer B (AS 65013, spanning Gold + Garnet) end-to-end across the G-R5↔Gar-R7 Option-C path; prove CE-to-CE reachability over two AS boundaries with a single VPNv4 label stack.

### Challenge B — Deterministic failover
- Combine Add-Path + PIC-Edge + Best-External so a multi-homed Customer A prefix (Emerald + Gold) fails over in <1s on egress-PE loss; measure packet loss with continuous traffic.

### Challenge C — Inter-AS traffic engineering
- Use AIGP + LOCAL_PREF + communities together to make Emerald prefer Gold-transit for one customer VRF and direct-Garnet for another — same prefixes, per-VRF policy — and document exactly which best-path step decides each case.

### Challenge D — RTC + conditional advertisement at scale
- Enable RT-Constraint on all RR sessions and prove the inter-AS VPNv4 table shrinks to only imported RTs; layer conditional advertisement so a backup aggregate only appears during a tracked-prefix failure.
