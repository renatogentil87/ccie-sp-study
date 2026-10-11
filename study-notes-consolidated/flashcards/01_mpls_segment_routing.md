# CCNP SPCOR Flashcards — MPLS & Segment Routing (20%)

**Exam:** 350-501 SPCOR | **Domain:** MPLS and Segment Routing
**Format:** Q on front, A on back. Mix of theory, config reading, output interpretation, and scenario.

---

## Theory Cards

### Card 1
**Q:** What are the three MPLS label operations?
**A:** PUSH (impose label at ingress), SWAP (exchange label at transit), POP (remove label at egress). The penultimate hop does PHP (Penultimate Hop Popping) — pops the label one hop before the destination so the egress router doesn't need to do two lookups.

---

### Card 2
**Q:** What is the MPLS label range reserved for special use?
**A:** Labels 0-15 are reserved. Key ones: 0 = explicit-null (IPv4), 1 = router-alert, 2 = explicit-null (IPv6), 3 = implicit-null (signals PHP — never appears on the wire).

---

### Card 3
**Q:** What protocol distributes transport labels in a traditional MPLS core? What port does it use?
**A:** LDP (Label Distribution Protocol). Uses TCP port 646 for session establishment and UDP port 646 for neighbor discovery (hello messages).

---

### Card 4
**Q:** What is the difference between LDP downstream-on-demand vs downstream-unsolicited?
**A:** **Downstream-unsolicited** (default on Cisco): LSR advertises labels to all peers without being asked. **Downstream-on-demand**: LSR only sends a label when a peer explicitly requests it. SP networks use unsolicited.

---

### Card 5
**Q:** What is the difference between liberal label retention and conservative label retention?
**A:** **Liberal** (Cisco default): keep ALL labels received from ALL peers, even non-best-path. Faster convergence (backup label ready) but more memory. **Conservative**: keep only the label from the best next-hop. Less memory but slower convergence on failover.

---

### Card 6
**Q:** What is PHP and why does it exist?
**A:** Penultimate Hop Popping — the second-to-last router pops the transport label so the egress router receives a plain IP packet (or just the VPN label). It exists to avoid a double lookup on the egress router (label lookup + IP/VRF lookup). Signaled by advertising implicit-null (label 3) to the upstream neighbor.

---

### Card 7
**Q:** What is the MPLS label stack order for a VPN packet? (bottom to top)
**A:** Bottom: **VPN label** (identifies the VRF/service on the egress PE). Top: **Transport label** (LDP or SR, gets the packet to the egress PE loopback). The P routers only see/swap the transport label. The VPN label is invisible to them.

---

### Card 8
**Q:** In Segment Routing, what is a prefix-SID vs an adjacency-SID?
**A:** **Prefix-SID**: globally significant, identifies a node (typically the loopback). SRGB base + index = label. Same value network-wide for a given destination. **Adjacency-SID**: locally significant, identifies a specific link to a neighbor. Used for traffic engineering (explicit path steering).

---

### Card 9
**Q:** What is the SRGB? Default range on IOS-XR?
**A:** Segment Routing Global Block — the label range reserved for SR prefix-SIDs. Default on IOS-XR: **16000–23999** (8000 labels). Prefix-SID label = SRGB base + index. Example: index 1 = label 16001.

---

### Card 10
**Q:** What replaces RSVP-TE signaling in an SR-TE network?
**A:** Nothing — SR-TE is **stateless**. The headend encodes the path as a stack of SIDs (label stack). No per-flow state on transit routers. No RSVP sessions, no soft-state refresh, no bandwidth reservation (use Flex-Algo or PCE for constraints instead).

---

### Card 11
**Q:** What is TI-LFA and what problem does it solve?
**A:** Topology-Independent Loop-Free Alternate. Provides sub-50ms protection for SR networks (like FRR for RSVP-TE). Pre-computes a backup path using segment routing — works for ANY topology (unlike basic LFA which only works if a loop-free neighbor exists). Uses a repair label stack (post-convergence path).

---

### Card 12
**Q:** Name three advantages of SR-MPLS over LDP.
**A:** 1) No LDP sessions to maintain (less protocol overhead). 2) No per-flow state on transit routers (TE without RSVP). 3) Simplified operations — one protocol (IGP) distributes both reachability AND labels. Also: TI-LFA provides 100% topology coverage for FRR (LFA can't always find an alternate).

---

### Card 13
**Q:** What is the difference between SR-MPLS and SRv6?
**A:** **SR-MPLS**: encodes segments as MPLS labels (20-bit). Compatible with existing MPLS hardware. **SRv6**: encodes segments as IPv6 addresses (128-bit) in the SRH (Segment Routing Header). No MPLS at all — native IPv6 forwarding. More flexible (network programming) but larger overhead.

---

### Card 14
**Q:** What does `segment-routing mpls` under the IGP (IS-IS/OSPF) enable?
**A:** Tells the IGP to advertise SR capabilities (SRGB, prefix-SIDs) in its link-state updates. Nodes receiving these extensions can build SR-MPLS forwarding entries. This is the single command that turns on SR in the network — no separate protocol needed.

---

### Card 15
**Q:** What is an SR-TE policy and what are its three components?
**A:** An SR-TE policy steers traffic through a specific path. Components: 1) **Color** (intent/class identifier), 2) **Endpoint** (destination), 3) **Candidate path** (one or more segment-lists defining the actual path). Traffic matching a BGP route with the policy's color+endpoint gets steered into it.

---

## Config Reading Cards

### Card 16
**Q:** What does this config do?
```
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 5
```
**A:** Enables SR-MPLS under IS-IS and assigns prefix-SID index 5 to this router's Loopback0. The router will be reachable via label 16005 (SRGB base 16000 + index 5) from anywhere in the SR domain.

---

### Card 17
**Q:** What's wrong with this config? The LDP session won't come up.
```
mpls ldp
 router-id 1.1.1.1
 interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/1
```
**A:** Missing `address-family ipv4` under `mpls ldp`. On IOS-XR, LDP requires the address-family to be explicitly configured:
```
mpls ldp
 router-id 1.1.1.1
 address-family ipv4
 !
 interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/1
```

---

### Card 18
**Q:** What does `explicit-null` do under the SR prefix-SID config?
```
interface Loopback0
 address-family ipv4 unicast
  prefix-sid index 5 explicit-null
```
**A:** Disables PHP for this prefix-SID. The penultimate hop will NOT pop the label — instead it sends the packet with label 0 (explicit-null) to the egress router. Use case: when you need the egress router to see the EXP bits for QoS classification (PHP strips the label and loses EXP info).

---

### Card 19
**Q:** What does this RSVP-TE config achieve?
```
interface tunnel-te1
 ipv4 unnumbered Loopback0
 destination 12.12.12.12
 signalled-bandwidth 50000
 path-option 10 explicit name VIA-R8
 path-option 20 dynamic
 fast-reroute
```
**A:** Creates a TE tunnel to 12.12.12.12 reserving 50Mbps. Primary path follows explicit route VIA-R8; if that fails, falls back to dynamic CSPF (path-option 20). FRR enabled — transit routers will pre-compute backup paths for sub-50ms protection.

---

### Card 20
**Q:** What does this config do and when would you use it?
```
router isis CORE
 interface GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   mpls ldp sync
```
**A:** LDP-IGP synchronization. If the LDP session on this interface goes down, IS-IS advertises maximum metric (2^24 - 1) for the link — forcing traffic to use other paths that still have working LDP. Prevents black-holing traffic during LDP convergence. Used on all core links in production SPs.

---

## Output Interpretation Cards

### Card 21
**Q:** What does this output tell you? Is there a problem?
```
RP/0/RP0/CPU0:R5# show mpls forwarding prefix 12.12.12.12/32
Local  Outgoing    Prefix          Outgoing     Next Hop
Label  Label       or ID           Interface
------ ----------- --------------- ------------ --------
16012  Pop         12.12.12.12/32  Gi0/0/0/4    10.10.12.12
```
**A:** No problem. R5 is the **penultimate hop** to R12 — it pops the SR label (16012 → Pop/implicit-null) and sends a plain IP packet to R12 via Gi0/0/0/4. This is normal PHP behavior. If it said `16012 → 16012 SWAP` instead, that would mean R5 is NOT the penultimate hop.

---

### Card 22
**Q:** This PE can't reach remote VPN routes. What's the issue?
```
RP/0/RP0/CPU0:R1# show bgp vpnv4 unicast summary
Neighbor        AS    MsgRcvd  MsgSent  Up/Down  St/PfxRcvd
8.8.8.8         64512  0        0        Active   0
9.9.9.9         64512  0        0        Active   0
```
**A:** Both BGP sessions are stuck in **Active** state (not Established). The PE can't reach the RR loopbacks (8.8.8.8 / 9.9.9.9) via TCP. Most likely cause: the IGP (IS-IS/OSPF) or LDP isn't running correctly, so there's no route to the RR loopbacks. Check `show route 8.8.8.8` and `show isis neighbors` first.

---

### Card 23
**Q:** What does this output tell you about the label stack?
```
RP/0/RP0/CPU0:R1# show cef vrf CUSTOMER_A 13.13.13.13/32
  nexthop 10.1.5.5 GigabitEthernet0/0/0/0 label [16012|24005]
```
**A:** Two-label stack. **Outer (16012)**: SR transport label to reach PE R12 (SRGB base + index 12). **Inner (24005)**: VPN label allocated by R12 for the CUSTOMER_A VRF prefix. R1 pushes both labels; P routers swap only the outer; R12 pops outer (PHP at penultimate), then uses inner 24005 to identify the VRF.

---

### Card 24
**Q:** The TE tunnel is down. What does this output tell you?
```
RP/0/RP0/CPU0:R1# show mpls traffic-eng tunnels tunnel-te2
Name: tunnel-te2  Destination: 12.12.12.12
  Status: Admin: up  Oper: down  Path: not valid
  path-option 10 explicit VIA-R8 (Path not found)
  path-option 20 dynamic (Path not found)
```
**A:** Both path options failed — CSPF can't find ANY valid path to 12.12.12.12. Likely causes: 1) RSVP-TE not enabled on all links in the path, 2) bandwidth constraints can't be satisfied on any path, 3) affinity/color constraints exclude all available links, or 4) the TE topology database is incomplete (`show mpls traffic-eng topology` to verify all nodes visible).

---

### Card 25
**Q:** What does this show? Is the network converged?
```
RP/0/RP0/CPU0:R8# show isis neighbors
IS-IS CORE neighbors:
System Id   Interface        SNPA           State  Holdtime  Type
R5          Gi0/0/0/2        *PtoP*         Up     28        L2
R6          Gi0/0/0/3        *PtoP*         Up     26        L2
R10         Gi0/0/0/1        *PtoP*         Up     29        L2
R11         Gi0/0/0/4        *PtoP*         Init   30        L2
```
**A:** R8's adjacency to R11 is stuck in **Init** state — one-way hello seen but two-way not established. R8 is sending hellos to R11 and receiving them, but R11 isn't acknowledging R8 back. Common causes: mismatched IS-IS level (R8 L2-only but R11 L1-only), authentication mismatch, or interface MTU mismatch. Network is NOT fully converged — traffic to/from R11 will be affected.

---

## Scenario/Drag-and-Drop Style Cards

### Card 26
**Q:** Match the label distribution protocol to its characteristic:

| Protocol | Characteristic |
|----------|---------------|
| 1. LDP | A. Reserves bandwidth per-flow |
| 2. RSVP-TE | B. Labels distributed via IGP extensions |
| 3. SR (IGP) | C. Follows IGP best path, no TE constraints |
| 4. MP-BGP | D. Distributes VPN labels between PEs |

**A:** 1-C, 2-A, 3-B, 4-D

---

### Card 27
**Q:** Order the BGP best-path selection steps (first to last):
- MED
- LOCAL_PREF
- Weight
- AS-PATH length
- Origin code
- eBGP over iBGP
- IGP metric to next-hop

**A:** Weight → LOCAL_PREF → Locally originated → AS-PATH length → Origin (i > e > ?) → MED → eBGP over iBGP → IGP metric to next-hop → Router-ID

---

### Card 28
**Q:** Match the mVPN profile to its transport:

| Profile | Transport |
|---------|-----------|
| Profile 0 | ? |
| Profile 1 | ? |
| Profile 6 | ? |
| Profile 7 | ? |
| Profile 14 | ? |

**A:**
- Profile 0 → GRE (default MDT, PIM in core)
- Profile 1 → mLDP P2MP (in-band signaling)
- Profile 6 → mLDP P2MP (BGP C-multicast signaling)
- Profile 7 → RSVP-TE P2MP (BGP signaling, bandwidth-reserved)
- Profile 14 → mLDP MP2MP (bidirectional, BGP signaling)

---

### Card 29
**Q:** A customer CE is dual-homed to PE R1 and PE R2. The customer's prefix 15.15.15.15/32 is being advertised back to the CE via the second PE (routing loop). What feature prevents this?

**A:** **Site-of-Origin (SoO)**. Configure `set extcommunity soo 64512:15` inbound on both PE-CE sessions. When a PE receives a VPN route with SoO matching the local CE's site, it does NOT advertise it back to that CE.

---

### Card 30
**Q:** You need sub-50ms failover for 500,000 VPN prefixes when a PE-CE link fails. The traditional approach (BGP reconvergence) takes 30+ seconds. What two features combined solve this?

**A:** **BGP Add-Path** (backup path pre-advertised by the RR so the PE already knows the alternate) + **BGP PIC** (Prefix Independent Convergence — backup path pre-installed in CEF/FIB, single pointer swap for ALL prefixes simultaneously regardless of count). Together: sub-second failover for any number of prefixes.

---

### Card 31
**Q:** What is the 3-label stack for an Inter-AS Option C VPN packet at the ingress PE?

**A:** Top to bottom:
1. **IGP/LDP label** — transport to the local ASBR
2. **BGP-LU label** — labeled-unicast, reaches the remote PE loopback across AS boundary
3. **VPN label** — identifies the VRF on the remote PE

The ASBR swaps label 1 for the inter-AS BGP-LU label; remote ASBR swaps for its IGP label. Remote PE receives just the VPN label.

---

### Card 32
**Q:** Drag into correct category — which are **globally significant** vs **locally significant** in SR?

Items: Prefix-SID, Adjacency-SID, Binding-SID, Node-SID, BGP Peer-SID

**A:**
- **Globally significant:** Prefix-SID, Node-SID (same label everywhere)
- **Locally significant:** Adjacency-SID, Binding-SID, BGP Peer-SID (meaning varies per router that allocated it)

---

### Card 33
**Q:** Your SR-TE policy uses this segment list: `[16005, 24001, 16012]`. What does each segment do?

**A:**
- **16005** (prefix-SID): route to node R5 (SRGB 16000 + index 5) — packet goes to R5 first
- **24001** (adjacency-SID): locally significant on R5, forces the packet out a specific interface (e.g., toward R8 instead of the IGP shortest path)
- **16012** (prefix-SID): route to node R12 — from R8, packet follows shortest path to R12

Result: explicit path R1 → R5 → (specific link) → R8 → ... → R12.

---

### Card 34
**Q:** What Flex-Algo metric types are available? Match each to its use case.

| Metric | Use Case |
|--------|----------|
| IGP metric | ? |
| TE metric | ? |
| Delay metric | ? |

**A:**
- **IGP metric** → default routing (bandwidth-based cost, general purpose)
- **TE metric** → traffic engineering optimization (independently tunable, capacity planning)
- **Delay metric** → low-latency applications (5G URLLC, financial trading, real-time)

---

### Card 35
**Q:** An operator configures `bgp bestpath as-path multipath-relax` on a PE. What does this enable?

**A:** Allows BGP to install multiple paths as ECMP (multipath) even when the AS-PATH content differs — as long as the AS-PATH **length** is the same. Without it, BGP only does multipath for identical AS-PATHs. Used when a PE has routes to the same prefix from different transit providers (different AS-PATHs but same length).

---

## End of Deck — MPLS & Segment Routing (35 cards)

> **Next decks to build:** Networking (IS-IS/OSPF/BGP — 30%), Services (L3VPN/L2VPN/EVPN — 20%), Architecture (15%), Automation (15%)
