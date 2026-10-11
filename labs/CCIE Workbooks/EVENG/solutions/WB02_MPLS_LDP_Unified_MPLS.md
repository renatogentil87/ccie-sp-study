# CCIE SP Workbook 02 — MPLS LDP & Unified MPLS

**Domain 1: Core Routing (25%)**
**Platform:** IOS-XRv 9000

---

## Topology Reference

**Emerald — AS 65100 (IS-IS + LDP)**

| Node | Loopback0 | Role |
|------|-----------|------|
| E-R5 | 6.6.6.6 | Path Computation Element |
| E-R4 | 4.4.4.4 | Core P |
| E-R3 | 3.3.3.3 | Core P |
| E-R6 | 5.5.5.5 | Inter-AS border |
| E-R1 | 1.1.1.1 | Edge PE |
| E-R2 | 2.2.2.2 | Edge PE |

**Garnet — AS 65200 (IS-IS + SR-MPLS)**

| Node | Loopback0 | Role |
|------|-----------|------|
| Gar-R6 | 16.16.16.16 | Path Computation Element |
| Gar-R3–Gar-R5 | — | Core P |
| Gar-R7 | 17.17.17.17 | Inter-AS border |
| Gar-R1 | 11.11.11.11 | Edge PE |
| Gar-R2 | 12.12.12.12 | Edge PE |

**Gold — AS 65300 (IS-IS + SRv6)**

| Node | Loopback0 | Role |
|------|-----------|------|
| G-R4 | 24.24.24.24 | Inter-AS border |
| G-R5 | 25.25.25.25 | Inter-AS border |
| G-R3 | 23.23.23.23 | Core P |
| G-R1 | 21.21.21.21 | Edge PE |
| G-R2 | 22.22.22.22 | Edge PE |

**Inter-AS links:**
- E-R6 ↔ Gar-R7 (direct, Emerald ↔ Garnet)
- E-R6 ↔ G-R4 (Emerald ↔ Gold)
- G-R5 ↔ Gar-R7 (Gold ↔ Garnet)

**Inter-AS addressing convention used in this workbook:**

| Link | A-side | B-side |
|------|--------|--------|
| E-R6 ↔ Gar-R7 | 10.100.200.5/30 (E-R6 Gi0/0/0/2) | 10.100.200.6/30 (Gar-R7) |
| E-R6 ↔ G-R4 | 10.100.300.5/30 (E-R6 Gi0/0/0/3) | 10.100.300.21/30 (G-R4 Gi0/0/0/1) |
| G-R5 ↔ Gar-R7 | 10.300.200.22/30 (G-R5 Gi0/0/0/2) | 10.300.200.16/30 (Gar-R7 Gi0/0/0/1) |

---

## Section 1 — LDP Basics on Emerald

### Task 1.1: LDP Neighbor Discovery (5 points)

**Question:** On the Emerald core, enable LDP on all IS-IS core interfaces so that E-R3 (3.3.3.3), E-R4 (4.4.4.4), E-R6 (5.5.5.5), E-R1 (1.1.1.1), and E-R2 (2.2.2.2) form LDP adjacencies. Use the interface-level `mpls ldp` model. Verify that link-hello (UDP 646) discovery brings up sessions over TCP 646.

**Solution:**

On E-R1 (representative — repeat on every core node, substituting its own core-facing interfaces):

```
mpls ldp
 interface GigabitEthernet0/0/0/0
 !
!
```

On E-R3 (two core links toward E-R1 and E-R4):

```
mpls ldp
 interface GigabitEthernet0/0/0/0
 !
 interface GigabitEthernet0/0/0/1
 !
!
```

Explanation: LDP discovery starts with periodic **link hellos** to the "all routers on this subnet" multicast address 224.0.0.2 on **UDP 646**. Once two LSRs hear each other's hello, the LSR with the higher transport address opens the **TCP 646** session and they exchange label bindings. Enabling LDP under an interface is all that is required for basic discovery — the transport address defaults to the LDP router-ID.

**Verification:**

```
show mpls ldp discovery
show mpls ldp neighbor brief
show mpls ldp neighbor 3.3.3.3 detail
show mpls interfaces
```

Look for `Xmit/Recv` link hellos in `discovery` and `Oper` state in `neighbor brief`.

---

### Task 1.2: Targeted LDP Session (5 points)

**Question:** E-R1 (1.1.1.1) and E-R2 (2.2.2.2) are not directly connected but require a directed LDP session (e.g., for a future L2VPN/AToM pseudowire). Configure a **targeted** LDP session between E-R1 and E-R2 without relying on link hellos.

**Solution:**

On E-R1:

```
mpls ldp
 neighbor 2.2.2.2 targeted
!
```

On E-R2:

```
mpls ldp
 neighbor 1.1.1.1 targeted
!
```

Explanation: A targeted session replaces multicast link hellos with **targeted (unicast) hellos** sent directly to the peer's transport address. One end must be `active` (initiates the hellos) and the other may be `passive` (only responds). In IOS-XR, `neighbor <ip> targeted` makes the local router actively send targeted hellos; configuring it on both ends makes both active, which is the safe/symmetric approach. Targeted hellos still bring up a normal TCP 646 session and full label exchange.

**Verification:**

```
show mpls ldp discovery
show mpls ldp neighbor 2.2.2.2 detail
```

In `discovery` output the session shows `Targeted Hello ... active/passive`; the neighbor should show `Up` with the loopback as the peer LDP ID.

---

### Task 1.3: LDP Router-ID from Loopback0 (5 points)

**Question:** Ensure every Emerald LSR derives a **deterministic** LDP router-ID and transport address from its `Loopback0` interface, so that LDP IDs match the IS-IS/BGP loopbacks (e.g., E-R1 = 1.1.1.1, E-R3 = 3.3.3.3). Prevent the router from auto-selecting an arbitrary interface address.

**Solution:**

On E-R1 (repeat per node with the matching loopback):

```
mpls ldp
 router-id 1.1.1.1
 address-family ipv4
  discovery transport-address 1.1.1.1
 !
!
```

Explanation: Without an explicit `router-id`, IOS-XR picks the highest operational address, which can flap. Pinning `router-id` to Loopback0 fixes the LDP ID. The `discovery transport-address` under the IPv4 AF forces the TCP transport endpoint to the same loopback so that the LDP session terminates on the routable, always-up loopback rather than a link address. Loopback0 must be reachable in the IGP (advertised as a /32 host route in IS-IS).

**Verification:**

```
show mpls ldp discovery detail
show mpls ldp neighbor detail | include "LDP Id|Transport"
show run mpls ldp
```

Confirm `LDP Id: 1.1.1.1:0` and `Transport address: 1.1.1.1` on the peers.

---

### Task 1.4: Label Allocation Filtering — Host Routes Only (5 points)

**Question:** Reduce LFIB size on the Emerald core. Configure LDP so each LSR only **allocates/originates** local labels for **/32 host routes** (loopbacks), not for the intra-area transit links. Use a prefix-list.

**Solution:**

On every Emerald LSR:

```
prefix-set HOST-ROUTES
  0.0.0.0/0 eq 32
end-set
!
```

(IOS-XR classic prefix-list equivalent:)

```
ipv4 prefix-list HOST-ROUTES
 10 permit 0.0.0.0/0 eq 32
!
mpls ldp
 label
  local
   allocate for prefix-list HOST-ROUTES
  !
 !
!
```

Explanation: `label local allocate for <prefix-list>` restricts which FECs the LSR generates a **local** label for. Matching `0.0.0.0/0 eq 32` allows only /32s (loopbacks), so no labels are allocated for the /30 core links. This keeps the LFIB lean and is standard SP hygiene — traffic for the links themselves rides the IGP, while LSP transport is only needed to reach PE/P loopbacks. Note this controls *allocation*; it is distinct from *advertisement* filtering (Task 2.4).

**Verification:**

```
show mpls ldp bindings summary
show mpls ldp bindings 3.3.3.3/32
show mpls forwarding | include /32
show mpls ldp bindings local-only
```

Only /32s should have a local label; /30 links should show `No Label (local)` / not present.

---

### Task 1.5: LDP Session Protection (5 points)

**Question:** On E-R3 (3.3.3.3), protect the LDP sessions so that if a directly connected link goes down but the neighbor is still reachable via an alternate IGP path, the **label bindings are retained** and the session survives via a targeted hello. Hold retained bindings for 120 seconds.

**Solution:**

On E-R3:

```
mpls ldp
 session protection duration 120
!
```

To scope protection to specific peers with an ACL:

```
ipv4 access-list LDP-SP-PEERS
 10 permit ipv4 host 4.4.4.4 any
 20 permit ipv4 host 1.1.1.1 any
!
mpls ldp
 session protection for LDP-SP-PEERS duration 120
!
```

Explanation: **LDP Session Protection** automatically establishes a **targeted** hello to a directly connected peer. If the link fails but the peer remains reachable through another path, the directed session and its learned label bindings stay up for the configured `duration`, avoiding relabeling churn and speeding convergence when the link (or an alternate LDP-enabled path) recovers. `duration 120` holds the protected state for 120s; `infinite` keeps it indefinitely.

**Verification:**

```
show mpls ldp neighbor detail | include "Session Protection|Duration"
show mpls ldp discovery | include Targeted
```

The neighbor detail shows `Session Protection: enabled, state: Ready, duration: 120 sec`.

---

## Section 2 — LDP Advanced

### Task 2.1: LDP Authentication (MD5) (5 points)

**Question:** Secure the LDP TCP sessions on the E-R6 (5.5.5.5) ↔ E-R3 (3.3.3.3) link with **MD5 authentication**. Configure it symmetrically.

**Solution:**

On E-R6 (peer = E-R3 LDP ID 3.3.3.3):

```
mpls ldp
 neighbor 3.3.3.3 password encrypted <key>
!
```

Cleartext form (IOS-XR encrypts it on write):

```
mpls ldp
 neighbor 3.3.3.3 password clear C1sco-LDP-Key!
!
```

On E-R3 (peer = E-R6 5.5.5.5) — must use the **same** key:

```
mpls ldp
 neighbor 5.5.5.5 password clear C1sco-LDP-Key!
!
```

Explanation: LDP MD5 authentication protects the TCP 646 session using the TCP MD5 signature option (RFC 2385). Both peers must configure the identical password keyed to the **peer's LDP router-ID**. A mismatch or one-sided config prevents the TCP session from establishing (the session stays in discovery but never reaches Oper). A global default can be set with `mpls ldp neighbor password ...` and overridden per-neighbor.

**Verification:**

```
show mpls ldp neighbor 3.3.3.3 detail | include "Password|TCP"
show tcp brief | include :646
show mpls ldp neighbor brief
```

Session should be `Oper`; the detail shows password/MD5 in use. A mismatch yields repeated TCP resets on port 646.

---

### Task 2.2: LDP Graceful Restart (5 points)

**Question:** Enable **LDP Graceful Restart (GR)** on E-R1 (1.1.1.1) and its core neighbor E-R3 (3.3.3.3) so that an LDP control-plane restart (e.g., RP failover) does not tear down the forwarding path. Set forwarding-state hold and reconnect timers appropriately.

**Solution:**

On E-R1 and E-R3 (GR must be enabled on both peers before the session comes up):

```
mpls ldp
 graceful-restart
 graceful-restart reconnect-timeout 120
 graceful-restart forwarding-state-holdtime 180
!
```

Explanation: LDP GR lets a neighbor act as a **helper**, preserving the MPLS forwarding state (labels stay programmed) while the restarting LSR rebuilds its LDP sessions and relearns bindings. `reconnect-timeout` is how long the helper waits for the restarting peer's TCP session to re-establish; `forwarding-state-holdtime` is how long the previously learned labels are kept as "stale" during recovery. GR is negotiated via the FT (Fault Tolerant) TLV at session setup, so it must be enabled **before** the session forms — toggling it clears and re-establishes sessions.

**Verification:**

```
show mpls ldp graceful-restart
show mpls ldp neighbor detail | include "Graceful Restart|Reconnect|Recovery"
show mpls ldp neighbor 3.3.3.3 detail
```

Neighbor detail shows `Graceful Restart: Yes (Reconnect Timeout: 120, ...)`.

---

### Task 2.3: LDP-IGP Synchronization Verification (5 points)

**Question:** LDP-IGP sync is configured on the Emerald IS-IS core. On the E-R4 (4.4.4.4) ↔ E-R3 (3.3.3.3) link, **disable `mpls ldp` on the E-R3 end** to simulate an LDP-not-ready condition, and verify that IS-IS advertises the **maximum metric** for that link until LDP re-synchronizes.

**Solution:**

Baseline sync config (should already be present on both ends, under IS-IS):

```
router isis EMERALD
 interface GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   mpls ldp sync
  !
 !
!
```

Simulate the failure on E-R3 (remove LDP from the shared interface):

```
mpls ldp
 no interface GigabitEthernet0/0/0/1
!
```

Explanation: **LDP-IGP synchronization** prevents an IGP link from being used for LSP transport before LDP is ready on it (which would blackhole labeled traffic). While LDP is not "in sync" on the link, IS-IS advertises that adjacency's metric as the **maximum (max-metric 16777214 for wide metrics / 63 for narrow)**, steering traffic away. Once LDP re-establishes and exchanges labels, IS-IS restores the real metric. Removing LDP from E-R3's interface makes the sync state on E-R4 go down, triggering the max-metric.

**Verification:**

On E-R4 (the still-LDP-enabled end):

```
show mpls ldp igp sync
show isis interface GigabitEthernet0/0/0/1 | include "Metric|LDP Sync"
show isis database <E-R4-hostname>.00-00 detail | include "Metric: 16777214"
```

Expect `Sync status: not achieved` and the IS-IS extended reachability TLV for that link showing metric `16777214`. Re-adding LDP on E-R3 restores `Sync status: achieved` and the normal metric.

---

### Task 2.4: LDP Label Advertisement Filtering — PE Loopbacks Only (5 points)

**Question:** On the Emerald core, configure LDP so that each LSR only **advertises** labels for the **PE loopbacks E-R1 (1.1.1.1/32) and E-R2 (2.2.2.2/32)** to its neighbors — not for P/ASBR/Gar-R6 loopbacks. Use advertise (outbound) filtering.

**Solution:**

On every Emerald LSR:

```
ipv4 prefix-list PE-LOOPBACKS
 10 permit 1.1.1.1/32
 20 permit 2.2.2.2/32
!
mpls ldp
 label
  local
   advertise
    for PE-LOOPBACKS
   !
  !
 !
!
```

Explanation: `label local advertise for <prefix-list>` controls **outbound** label advertisement — which locally allocated FEC-label bindings are announced to peers. Restricting it to PE loopbacks means only E-R1/E-R2 receive transport labels network-wide, which is exactly what you need for MPLS VPN/pseudowire transport (LSPs terminate on PEs). This is *advertisement* filtering, complementary to the *allocation* filtering in Task 1.4: allocation decides whether a label exists locally; advertisement decides whether peers hear about it. You can also filter what you *accept* with `label remote accept from <peer> for <pfx-list>`.

**Verification:**

```
show mpls ldp bindings local advertise-acls
show mpls ldp bindings 1.1.1.1/32 detail
show mpls ldp bindings 3.3.3.3/32 detail
```

`1.1.1.1/32` shows a local label advertised to peers; `3.3.3.3/32` shows the local label present but **not advertised** (no remote bindings generated from it).

---

## Section 3 — BGP Labeled Unicast / Unified MPLS

### Task 3.1: BGP-LU Concept for Inter-AS (5 points)

**Question:** Explain, and then set the stage for, **BGP Labeled Unicast (BGP-LU / RFC 3107)** as the inter-AS transport mechanism for **Unified MPLS (Seamless MPLS)** across Emerald → Gold → Garnet. Why is BGP-LU needed instead of extending LDP across the AS boundaries?

**Solution (conceptual + enabling config):**

BGP-LU carries an MPLS **label bound to an IPv4/IPv6 prefix** inside BGP Update messages, using the address-family `ipv4 labeled-unicast`. Instead of running one flat IGP+LDP domain end to end, each AS keeps its **own** IGP + label distribution (Emerald = LDP, Garnet = SR-MPLS, Gold = SRv6), and the **loopback /32s of remote PEs** are advertised between ASes as labeled BGP routes. The ASBRs **stitch** the intra-AS transport label to the BGP-assigned label, producing a continuous label-switched path across domains — this is **Unified/Seamless MPLS**.

Why not stretch LDP across AS boundaries: it would require a single IGP flooding domain (poor scaling, fate-sharing, no domain isolation), and it can't cross different label technologies (LDP vs SR vs SRv6). BGP-LU scales like BGP, keeps domains isolated, and lets each domain use its native transport while BGP provides the inter-domain label glue with next-hop-self at each ASBR.

Enabling the AF on E-R6 (5.5.5.5) as the pattern used in 3.2/3.3:

```
router bgp 65100
 address-family ipv4 labeled-unicast
 !
!
```

**Verification:**

```
show bgp ipv4 labeled-unicast summary
show bgp ipv4 labeled-unicast
```

Neighbors negotiating the labeled-unicast AF appear in the summary; routes show an attached label (`Received Label` / `Local Label`).

---

### Task 3.2: BGP-LU between E-R6 ↔ G-R4 (Emerald ↔ Gold) (5 points)

**Question:** Configure eBGP **labeled-unicast** between E-R6 (Emerald AS 65100, 5.5.5.5) and G-R4 (Gold AS 65300, 24.24.24.24) over the inter-AS link (E-R6 10.100.300.5, G-R4 10.100.300.21). Advertise Emerald PE loopbacks (1.1.1.1/32, 2.2.2.2/32) toward Gold with labels, and set next-hop-self so the label path stitches at the border. eBGP peering is over the directly connected interface addresses.

**Solution:**

On E-R6 (AS 65100):

```
route-policy PASS-ALL
  pass
end-policy
!
router bgp 65100
 address-family ipv4 labeled-unicast
 !
 neighbor 10.100.300.21
  remote-as 65300
  description eBGP-LU to G-R4 (Gold)
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
 ! Redistribute/originate the Emerald PE loopbacks into BGP-LU
 address-family ipv4 unicast
  network 1.1.1.1/32
  network 2.2.2.2/32
  allocate-label all
 !
!
```

Note: for BGP-LU, labels are assigned to the `ipv4 unicast` prefixes via `allocate-label`, and the labeled-unicast AF carries them to the peer. On IOS-XR the common pattern is to originate the /32s in `ipv4 unicast` with `allocate-label all` and enable the `ipv4 labeled-unicast` neighbor AF. Ensure the /32s are in the RIB (from IS-IS/LDP).

On G-R4 (AS 65300):

```
route-policy PASS-ALL
  pass
end-policy
!
router bgp 65300
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 10.100.300.5
  remote-as 65100
  description eBGP-LU to E-R6 (Emerald)
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-self
  !
 !
!
```

Explanation: eBGP-LU across the border exchanges each PE /32 with a label. At G-R4, `next-hop-self` on the iBGP re-advertisement into Gold makes G-R4 the next hop, so Gold's internal nodes resolve the BGP-LU route via G-R4's own transport (SRv6/IGP) — this is the **label stitch**: incoming BGP label from Emerald ↔ outgoing Gold intra-AS transport. `allocate-label all` (with a policy to scope it in production) tells BGP to assign local labels to the advertised prefixes. For eBGP between directly connected ASBRs, no multihop/loopback-source is needed since peering is on the connected /30.

**Verification:**

```
show bgp ipv4 labeled-unicast summary
show bgp ipv4 labeled-unicast 1.1.1.1/32
show bgp ipv4 labeled-unicast neighbors 10.100.300.21
show mpls forwarding prefix 1.1.1.1/32
show cef 1.1.1.1/32
```

`show bgp ipv4 labeled-unicast 1.1.1.1/32` should show a `Local Label` (assigned by E-R6) and, on G-R4, a `Received Label`. The CEF/MPLS forwarding entry should push the correct label.

---

### Task 3.3: BGP-LU between G-R5 ↔ Gar-R7 (Gold ↔ Garnet) (5 points)

**Question:** Configure eBGP **labeled-unicast** between G-R5 (Gold AS 65300, 25.25.25.25) and Gar-R7 (Garnet AS 65200, 17.17.17.17) over the inter-AS link (G-R5 10.300.200.22, Gar-R7 10.300.200.16). Propagate the Emerald PE loopbacks (learned from Task 3.2) onward into Garnet with labels, and have Gar-R7 set next-hop-self so Garnet SR-MPLS resolves the transit. Also advertise Garnet Gar-R1 (11.11.11.11/32) back toward Gold.

**Solution:**

On G-R5 (AS 65300):

```
route-policy PASS-ALL
  pass
end-policy
!
router bgp 65300
 address-family ipv4 unicast
  allocate-label all
 !
 neighbor 10.300.200.16
  remote-as 65200
  description eBGP-LU to Gar-R7 (Garnet)
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-self
  !
 !
!
```

On Gar-R7 (AS 65200):

```
route-policy PASS-ALL
  pass
end-policy
!
router bgp 65200
 address-family ipv4 unicast
  network 11.11.11.11/32
  network 12.12.12.12/32
  allocate-label all
 !
 neighbor 10.300.200.22
  remote-as 65300
  description eBGP-LU to G-R5 (Gold)
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   next-hop-self
  !
 !
!
```

Explanation: This is the second stitch point of the end-to-end Seamless MPLS chain. G-R5 (Gold) forwards the labeled Emerald PE prefixes it learned from G-R4 via iBGP-LU across the Gold core, then re-advertises them over eBGP-LU to Gar-R7 with `next-hop-self`. Gar-R7 injects them into Garnet's iBGP-LU and, with `next-hop-self`, becomes the resolving next hop so **SR-MPLS** carries transit inside Garnet. The reverse direction (Garnet Gar-R1 11.11.11.11/32) is originated at Gar-R7 with `allocate-label` and travels back toward Emerald the same way. Each ASBR swaps the inter-AS BGP label for the appropriate intra-AS transport label (LDP in Emerald, SRv6/IGP in Gold, SR in Garnet).

**Verification:**

```
show bgp ipv4 labeled-unicast summary
show bgp ipv4 labeled-unicast 1.1.1.1/32
show bgp ipv4 labeled-unicast 11.11.11.11/32
show mpls forwarding
show cef 1.1.1.1/32 detail
```

On Gar-R7, `1.1.1.1/32` (Emerald PE, learned transit through Gold) should show a received label from G-R5 and a locally allocated label re-advertised into Garnet.

---

### Task 3.4: End-to-End Label Path E-R1 → (Gold transit) → Gar-R1 (5 points)

**Question:** Verify and explain the **end-to-end Unified MPLS label path** from E-R1 (Emerald, 1.1.1.1) to Gar-R1 (Garnet, 11.11.11.11) transiting the **Gold** AS via the E-R6→G-R4 and G-R5→Gar-R7 stitches. Ensure E-R1 has a labeled route to 11.11.11.11/32 and trace the label operations at each hop.

**Solution (control-plane requirement + walk-through):**

For E-R1 to reach 11.11.11.11/32, E-R1 must run iBGP-LU with an Emerald route-reflector (or directly with E-R6) so the labeled route reaches the PE. On E-R1:

```
router bgp 65100
 address-family ipv4 labeled-unicast
 !
 neighbor 5.5.5.5
  remote-as 65100
  description iBGP-LU to E-R6 (via loopback)
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

On E-R6, reflect/next-hop-self toward Emerald PEs:

```
router bgp 65100
 neighbor 1.1.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv4 labeled-unicast
   route-reflector-client
   next-hop-self
  !
 !
!
```

**End-to-end label stack / stitch walk-through for a packet E-R1 → Gar-R1 (11.11.11.11):**

1. **E-R1** has 11.11.11.11/32 as a BGP-LU route with next-hop = E-R6 (5.5.5.5). It imposes **{BGP label from E-R6}** and then, to reach E-R6's loopback, pushes the **LDP transport label** for 5.5.5.5 learned from the Emerald core. Stack (top→bottom): `[LDP→E-R6][BGP-LU→11.11.11.11]`.
2. **Emerald core (P1/P2)** label-switches on the outer LDP label to E-R6; penultimate hop pops the LDP label (PHP), exposing the BGP-LU label at E-R6.
3. **E-R6** (stitch #1) swaps the incoming BGP-LU label for the outgoing eBGP-LU label advertised by **G-R4** and forwards across the Emerald↔Gold link.
4. **G-R4 → Gold core → G-R5:** G-R4 imposes Gold's intra-AS transport (SRv6/IGP) to reach G-R5 (which had next-hop-self for this prefix inside Gold), carrying the BGP-LU label to G-R5.
5. **G-R5** (stitch #2) swaps to the eBGP-LU label advertised by **Gar-R7** and forwards across the Gold↔Garnet link.
6. **Gar-R7 → Garnet core → Gar-R1:** Gar-R7 imposes Garnet's **SR-MPLS** transport to reach Gar-R1 (11.11.11.11), carrying the BGP-LU label; Gar-R1 pops and delivers.

The key principle: **transport labels are local to each AS; the BGP-LU label identifies the destination PE prefix end to end and is swapped at each ASBR stitch point.**

**Verification:**

On E-R1:

```
show bgp ipv4 labeled-unicast 11.11.11.11/32
show route 11.11.11.11/32
show cef 11.11.11.11/32 detail
show mpls forwarding prefix 11.11.11.11/32
traceroute 11.11.11.11 source Loopback0
```

Expect the BGP-LU entry to show next-hop 5.5.5.5 with a received label; `show cef ... detail` should display the **two-label stack** (outer LDP transport to E-R6 + inner BGP-LU). `show mpls forwarding` at each ASBR shows the **swap** operation. `traceroute mpls ...` (or a standard traceroute with label display) confirms the labeled path transits Gold.

Additional per-hop checks:

```
! On E-R6
show mpls forwarding labels <in-label>
show bgp ipv4 labeled-unicast 11.11.11.11/32
! On G-R4 / G-R5 (Gold)
show mpls forwarding
show bgp ipv4 labeled-unicast 11.11.11.11/32
! On Gar-R7
show mpls forwarding
show bgp ipv4 labeled-unicast 11.11.11.11/32
```

---

## Section 4 — Troubleshooting

### Task 4.1: LDP Session Down — Transport Address Mismatch (5 points)

**Question:** On the E-R6 (5.5.5.5) ↔ E-R3 (3.3.3.3) link, LDP **discovery** shows hellos being exchanged, but the session never reaches `Oper` (stuck initializing / no TCP). No MD5 is configured. Diagnose and fix.

**Symptom / Diagnosis:**

```
show mpls ldp discovery
! Hellos Xmit/Recv present, but:
show mpls ldp neighbor brief
! Session state = "Initialized" / never "Oper"
show tcp brief | include :646
! No established TCP 646 to the peer's expected address
```

Root cause: a **transport-address mismatch**. One end advertises a `discovery transport-address` (or interface address) that is **not reachable / not in the IGP** by the peer, or the two ends disagree on which address to open the TCP session to (e.g., E-R3 set `discovery transport-address 3.3.3.3` but 3.3.3.3/32 is not advertised into IS-IS, so E-R6 cannot open TCP to it). LDP hellos are UDP and still succeed, but the TCP 646 session fails because the transport address is unroutable.

**Solution:**

Make the transport address a routable loopback and ensure it is in the IGP. On E-R3:

```
router isis EMERALD
 interface Loopback0
  address-family ipv4 unicast
   ! ensure Loopback0 (3.3.3.3/32) is advertised
  !
 !
!
mpls ldp
 router-id 3.3.3.3
 address-family ipv4
  discovery transport-address 3.3.3.3
 !
!
```

If a specific interface must be used, set a **consistent** interface transport address on both ends, or leave it defaulting to the (reachable) router-ID. The fix: the chosen transport address must be reachable by the peer.

**Verification:**

```
show mpls ldp discovery detail | include "Transport|Address"
show route 3.3.3.3/32
show tcp brief | include :646
show mpls ldp neighbor brief
```

3.3.3.3/32 must be in the RIB on E-R6; TCP 646 becomes `ESTAB`; neighbor goes `Oper`.

---

### Task 4.2: Label Not in LFIB — Allocation Filtering Too Aggressive (5 points)

**Question:** After applying the label allocation filter from Task 1.4, traffic to a specific PE loopback **E-R2 (2.2.2.2/32)** is being IP-forwarded (unlabeled) on part of the Emerald core, and `show mpls forwarding` on E-R3 has **no entry** for 2.2.2.2/32. LDP sessions are up. Diagnose and fix.

**Symptom / Diagnosis:**

```
show mpls ldp bindings 2.2.2.2/32
! No local binding on E-R3 -> label was never allocated
show mpls forwarding prefix 2.2.2.2/32
! No LFIB entry
show run formal | include "prefix-list|allocate for"
```

Root cause: the **allocation prefix-list is too aggressive** — it does not match 2.2.2.2/32 (e.g., the prefix-set was written as `0.0.0.0/0 eq 32` but was accidentally scoped to specific hosts like only `1.1.1.1/32`, or a `le/ge` bound excludes it). Because no **local label** is allocated for the FEC, E-R3 has nothing to advertise or install, and forwarding for that prefix falls back to plain IP.

**Solution:**

Correct the prefix-list so all needed /32s (including 2.2.2.2/32) match:

```
ipv4 prefix-list HOST-ROUTES
 no 10
 10 permit 0.0.0.0/0 eq 32
!
```

Or if intentionally listing PEs, ensure E-R2 is included:

```
ipv4 prefix-list HOST-ROUTES
 10 permit 1.1.1.1/32
 20 permit 2.2.2.2/32
 30 permit 3.3.3.3/32
 40 permit 4.4.4.4/32
 50 permit 5.5.5.5/32
!
```

Then confirm LDP re-allocates:

```
mpls ldp
 label
  local
   allocate for prefix-list HOST-ROUTES
  !
 !
!
```

Explanation: `label local allocate for` gates **local label creation**. If the FEC isn't matched, no label exists, so it can't enter the LFIB or be advertised — the classic "too-aggressive filter blackholes/unlabels a prefix" trap. Distinguish this from *advertise* filtering (Task 2.4), where the label exists locally but isn't announced.

**Verification:**

```
show mpls ldp bindings 2.2.2.2/32
show mpls forwarding prefix 2.2.2.2/32
show mpls ldp bindings local-only | include 2.2.2.2
traceroute 2.2.2.2 source Loopback0
```

2.2.2.2/32 now has a local label, an LFIB entry, and traceroute shows labeled (MPLS) forwarding across the core.

---

### Task 4.3: BGP-LU Next-Hop Not Resolvable (5 points)

**Question:** On Gar-R7 (Garnet, 17.17.17.17), the Emerald PE prefix 1.1.1.1/32 is present in the BGP-LU table but is marked **not usable** — it is not installed in the RIB/CEF and not re-advertised into Garnet. The eBGP-LU session to G-R5 is up. Diagnose and fix.

**Symptom / Diagnosis:**

```
show bgp ipv4 labeled-unicast 1.1.1.1/32
! Path present but "(inaccessible)" / "not advertised" / next hop not resolved
show bgp ipv4 labeled-unicast 1.1.1.1/32 | include "next hop|inaccessible|not"
show route <bgp-next-hop>
! The BGP next hop has no route
```

Root cause: the **BGP next-hop is not resolvable / not label-reachable**. Because G-R5 did **not** apply `next-hop-self` (or the inter-AS /30 link address it left as next hop isn't redistributed into Garnet's IGP, and there is no label to it), Gar-R7 has no resolving route+label to the next hop. BGP requires the next hop to be reachable **and, for labeled-unicast transit, label-reachable**; an unresolvable next hop makes the route unusable and unadvertised.

**Solution:**

Set `next-hop-self` at the advertising ASBR (G-R5, toward Garnet) so the next hop becomes G-R5's loopback, which Garnet resolves via SR-MPLS. On **G-R5** (AS 65300):

```
router bgp 65300
 neighbor 10.300.200.16
  remote-as 65200
  address-family ipv4 labeled-unicast
   next-hop-self
  !
 !
!
```

Alternatively, if peering across the connected /30 without next-hop-self, ensure the /30 (or the peer address) is reachable **with a label** on Gar-R7 — but `next-hop-self` at the border is the correct Seamless-MPLS pattern because it makes each ASBR the label-stitch point and keeps inter-AS link subnets out of the IGP.

Also verify Gar-R7 can resolve/label the (new) next hop:

```
show route 25.25.25.25/32
show mpls forwarding prefix 25.25.25.25/32
```

Explanation: In Unified MPLS the recursion is **BGP-LU prefix → BGP next hop → intra-AS transport label**. If the next hop can't be resolved to a label-switched path, the labeled route is invalid. `next-hop-self` at each ASBR guarantees the next hop is that ASBR's loopback, which the local IGP/transport (SR-MPLS here) always makes label-reachable.

**Verification:**

```
show bgp ipv4 labeled-unicast 1.1.1.1/32
show bgp ipv4 labeled-unicast 1.1.1.1/32 | include "next hop|Local Label|Received Label"
show route 1.1.1.1/32
show cef 1.1.1.1/32 detail
show mpls forwarding prefix 1.1.1.1/32
```

The path should become **best/valid**, install in RIB/CEF with a label stack, and be re-advertised into Garnet with a locally allocated label.

---

## Quick-Reference: Key Commands

| Purpose | Command |
|---------|---------|
| LDP discovery/hellos | `show mpls ldp discovery [detail]` |
| LDP neighbors | `show mpls ldp neighbor [brief\|detail]` |
| LDP local/remote bindings | `show mpls ldp bindings [<pfx>] [local-only\|local advertise-acls]` |
| LFIB / label forwarding | `show mpls forwarding [prefix <p>\|labels <l>]` |
| LDP-IGP sync | `show mpls ldp igp sync` |
| LDP graceful restart | `show mpls ldp graceful-restart` |
| BGP-LU table | `show bgp ipv4 labeled-unicast [<pfx>\|summary\|neighbors <n>]` |
| CEF label stack | `show cef <pfx> detail` |
| TCP session (LDP 646) | `show tcp brief \| include :646` |
| IS-IS metric/LSP | `show isis interface <i>` / `show isis database <sys>.00-00 detail` |
