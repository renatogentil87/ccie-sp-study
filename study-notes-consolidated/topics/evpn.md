# EVPN (SPCOR Ch 13)

**PBB EVPN**
PBB Ethernet VPN is an advanced network technology that combines PBB with EVPN to create scalable and efficient
layer 2 and L3VPN services in large scale service provider network.

Key Components and features:
- PBB: Extension of 802.1Q designed to overcome scalability and vlan id space exhaustion
- PBB uses a two-layer mac-in-mac encapsulation scheme, where C-MAC are encapsulated within provider mac (P-MAC) for transport
across provider network.
- PBB introduces a service instance id (i-SID) to identify each customer service instance, allowing service multiplexing.
- EVPN: next-generation VPN technology that leverages BGP to provide scalable L2, L3VPN over MPLS networks.
- EVPN uses BGP for mac-address learning and advertisement, and distribution, allowing dynamic mac-address mobility across the network.
- EVPN supports both Ethernet Segment (ES) and IP Prefix (IP VRF) routes.

Integration of PBB and EVPN: PBB EVPN uses PBB encapsulation for transport of customer mac across provider network and leverages
evpn for mac distribution and control-plane signaling
- PBB EVPN allows providers to multiplex multiple customers services onto a single provider network using I-SID in the PBB header.
- TE and Traffic Isolation - Providers TE, via MPLS-TE or SR. Also ensures traffic isolation and segmentation between different
customer services using PBB encapsulation and separate I-SIDs.

PBB EVPN Components:
- Bridge Group: Group of Bridge Domain - same as VPLS
- Bridge Domain Two types of Bridge Domain:
  - Edge Bridge: where the attachments are included
  - Core Bridge: Contains BMAC - basically the mac for each provider edge device within the same MPLS network.
- EVI: EVPN Instance ID - You can import/export routes or mac based on RTs as L3VPN
- I-SID - Service Instance ID - Identifies the service instance, best practice to keep unique per service instance.

Pure EVPN puts customer MAC into BGP
PBB EVPN wraps customer frames in BMAC and only advertise BMAC in BGP
B-MAC is the outer mac that mac-in-mac encapsulation adds. 
EVPN - the route = customer MAC (fine grained, control plane learned, limited scale because it learns all customer MACs.) PBB
hides customer macs more scalable.

**EVPN**
VPLS learned MACs by flooding (data-plane flood and learn) and needed a full pseudowire mesh. This doesn't scale (mac table explosion, weak multi-homing)

- EVPN move mac learning into the Control Plane (BGP). Instead of flooding to learn where a MAC lives, each PE advertises its locally learned MACs to other PEs via BGP.
- EVPN = L3VPN BGP machinery, but for mac addresses.

**The 5 Route Types**
EVPN is defined by its BGP NLRI route types. Everything evpn does maps to one of these.

- Type 2 - MAC/IP advertisement
  - PE learns a host mac (and optionally its IP) on a local port, it advertises it as a type-2 route to all other PEs.
  - Remote PE install it: 'mac X is reachable via PE1'
  - it carries: MAC, Optional IP, L2VNI/LAbel, RD/RT, and a MAC mobility Seq Number
This replaces VPLS flood-and-learn

- Type 3 - Inclusive Multicast Ethernet Tag (IMET)
  - Handles BUM traffic (broadcast, unknown-unicast, multicast)
  - Each PE advertises " I participate in EVI 100, send BUM for it to me this way"
  - Sets up BUM distribution - via ingress replication (PE replicates to each remote PE) or P2MP/mLDP 
  - Needed because even in EVPN, some traffic must flood (ARP before mac is known, broadcast) - Type 3 defines how

- Type 1 - Ethernet Auto-Discovery (A-D)
  - Multi-homing machinery, two sub flavours:
    - Per-ES A-D: enables mass withdrawal - if a PE-CE link fails, one type 1 withdrawal tells everyon "all macs behind that segment are gone" instead of withdrawing
thousands of Type 2 one by one. Fast convergence.
    - Per-EVI A-D: enables aliasing - let remote PEs load balance to all PEs on multi-homed segment even if only one of them advertised a given MAC.

- Type 4 - Ethernet Segment Route
  - DF (Designated Forwarder) election. When a CE is multi-homed to 2+ PEs, exactly one must forward BUM toward the CE (or the CE gets duplicate broadcast).
PE sharing an ESI discovery each other via Type-4 and elect a DF

- Type 5 - IP Prefix Route
  - L3 routing (route evpn/IRB). Carries an IP prefix (not a host mac) so evpn can route between subnets, not just bridge within one.
  - This is how evpn does inter-subnet routing and integrates L2 + L3 in one control plane.

-----
Type 1 = Ethernet A-D - multi-homing (aliasing + mass withdrawal)
Type 2 - MAC/IP - host MAC/IP learning 
Type 3 - Inclusive Mcast - BUM setup
Type 4 - Ethernet Segment - DF election
Type 5 - IP Prefix - L3 Routing (IRB


**Multi-Homing**
VPLS could only do awkard primary/backup. EVPN does all active multi-homing cleanly.
- ESI (Ethernet Segment Identifier): a shared ID configured on the PEs attached to the same CE/LAG. It says "these ports are the same physical segment"
Two Modes:
- All Active: CE load balances across both PEs, both forward - full bw use
- Single Active: one PE active, other standby (used when the CE can't LAG to two PEs)

3 mechanisms that make it work:
- DF election (Type 4) - one PE is designated forwarder for bum toward the segment, so the CE doesn't get duplicate broadcasts.
- Split-Horizon/ ESI Label (Type 1) a BUM frame that entered from the ESI must NOT be sent back out the ESI by the other PE. The ESI Label identifies - this came from that segment
- Aliasing (type 1 per EVI) - even if only PE1 advertised host X's MAC (because PE happened to learn it), remote PE know host X sits on a segment reachable via both PE and PE2, so they load balance unicast to both.
- Mass withdrawal (Type 1 per ES) - Link PE-CE fails - Pe1 sends ONE type 1 withdrawal - all remot ePE instantly stop using PE for every MAC behind that segment.


**MAC Mobility**
When a host (VM)moves from PE1 to PE2:
 - PE2 learns the mac locally, advertises a Type-2 with a higher sequence number
 - Remote PEs see the higher seq number - update to PE2, PE1 withdraws its stale route
 - Prevents flip-flapping and duplicate-mac issues. VPLS had no clean mechanism for this - evpn makes vm mobility first class.

**IRB - Integrated Routing and Bridging (L2 - L3 together)**
EVPN doesn't just bridge L2, it also routes L3 in the same control plane:
- Type 2 carries MAC + Host IP - bridging + ARP suppresion
- Type-5 carries IP prefixes - inter-subnet routing
- Per-tenant VRF (an L3VNI in VXLAN, or a vpn label in mpls )routes between subnets

Two IRB modesl:
- Asymmetric IRB: ingress PE does both the routing and bridging; needs all VNIs/subnets configured on every PE. Simpler less scalable.
- Symmetric IRB: Ingress PE routes into a common L3VNI/Transit VRF, ships it to the egress PE which routes into dest subnet. Only the L3VNI needs to be everwhere.

Data Plane: Two encapsulations, one control plane
EVPN control plane BGP is independent of data plane:
- EVPN-MPLS - labels carry the frames
- EVPN-VXLAN - VXLAN VNIs carry the frames
- EVPN-SRv6 - newer, SRv6 SIDs instead of MPLS labels
- DCI(Data Center InterConnect) EVPN VXLAN in the DC, EVPN MPLS across the WAN, stitched at a border gateway (VNI ↔ EVI/label translation) — one EVPN control plane end-to-end. This is the dominant modern design.

**BUM handling**
Even with control-plane learning, some traffic floods (ARP before MAC known, broadcasts, multicast). Type-3 sets up delivery via:

- Ingress replication — the source PE makes N copies, one per remote PE (simple, fine for small fabrics)
- P2MP / mLDP tree — a multicast tree in the core (scales for large BUM volumes)
- EVPN also does ARP suppression — the PE answers ARP locally from its Type-2 knowledge, so ARP broadcasts don't traverse the fabric.

The RD/RT/EVI model:
- EVI = the EVPN instance ≈ a VRF name (the service container, locally significant)
- RD = uniqueness per PE (change RD, keep RT — for path diversity / avoid RR hiding)
- RT = membership — must match to share the service 
- RRs reflect EVPN routes (Type-1 to 5), just like VPNv4