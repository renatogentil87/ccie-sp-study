# L2VPN — AToM, VPLS, Carrier Ethernet & EVPN

**SPCOR Chapters:** 11 (MPLS L2VPN), 13 (Advanced MPLS Services / EVPN)
**Sources:** MPLS Fundamentals Ch 10-11, SPCOR Official Cert Guide, Lab notes

---

## Chapter 11: MPLS L2VPN

### AToM (Point-to-Point Pseudowires / VPWS)


### Chapter 10: Any Transport over MPLS (AToM)

AToM allows you to carry Layer 2 frames (ethernet, frame relay, ATM, PPP, any transport) across an MPLS backbone transparently.
The customer thinks they have a direct wire between two sites, but it is actually traversing the MPLS.

Key Components:

**Pseudowire (PW)**
- Virtual point-to-point connection between two PEs
- Emulates a physical wire across the MPLS network
- Identified by a VC-ID (both PEs must agree on the same VC-ID)

**Attachment Circuit (AC)**
- Physical interface on the PE that connects to the customer CE
- This interface becomes a pure L2 port - no ip address, no routing
- whatever frames arrive on this port get shoved into the pseudowire

**Targeted LDP Session**
- PE uses a special LDP session (not the regular link-based one) to exchange VC labels
- this ldp runs directly between the two PE loopbacks (targeted = no adjacent peers)
- It negotiates the VC label and pseudowire parameters

**Two Label Stack**
- Transport labe (top): gets the packet from ingress PE to egress PE (regular LDP/TE Label)
- VC Label (bottom): Tells the egress PE which pseudowire (which AC) to send the frame out

**Control Word (Optional)**
- a 4-byte header between the label stack and the layer 2 payload
- used for sequencing, padding small frames, identifying the payload type
- Required for some encapsulations (Frame Relay, ATM); optional for ethernet

HOW IT WORKS:
  CE -> Ethernet Frame -> PE -> Push VC Label + Transport Label -> P router MPLS -> PE -> Pop Labels forward new frame - CE

  1. R1 sends a normal Ethernet frame to R2
  2. R2 doesn't look at IP — just takes the entire L2 frame
  3. R2 pushes 2 labels: transport (to reach R8) + VC label (to identify the pseudowire)
  4. P routers swap only the transport label — they never see or touch the customer frame
  5. R8 receives the packet, uses the VC label to identify which AC to forward to
  6. R8 strips all labels and sends the raw Ethernet frame out toward R9
  7. R9 thinks R1 is directly connected on the same LAN
   
  Comparison with L3VPN
  
  ┌────────────────────┬───────────────────────────────────┬────────────────────────────────────┐
  │                    │ L3VPN                             │ AToM (L2VPN)                       │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ PE involvement     │ Routes customer traffic (Layer 3) │ Switches customer frames (Layer 2) │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ PE-CE protocol     │ BGP/OSPF/Static                   │ None — pure L2                     │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Customer awareness │ PE knows customer IP prefixes     │ PE doesn't know/care about payload │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Label bottom       │ VPN label (from BGP vpnv4)        │ VC label (from targeted LDP)       │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Label top          │ Transport (LDP/TE)                │ Transport (LDP/TE) — identical     │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Signaling          │ MP-BGP vpnv4                      │ Targeted LDP                       │
  └────────────────────┴───────────────────────────────────┴────────────────────────────────────┘
   
**IOS Configuration (basic)**
When you configure xconnect command, it automatically creates the target ldp session.
The only prerequisite is that both PEs have mpls ldp discovery targeted-hello accept configured.

  
On R2: 
  interface FastEthernet0/0
   xconnect 8.8.8.8 100 encapsulation mpls
   ! 8.8.8.8 = remote PE loopback
   ! 100 = VC-ID (must match on both sides)

On R8: 
  interface GigabitEthernet1/0
   xconnect 2.2.2.2 100 encapsulation mpls
  
  That's it — two lines per PE. The interface becomes L2, targeted LDP establishes automatically, VC labels are exchanged, pseudowire comes UP.

The PEs must agree on:
- VC-ID: PW never forms - PEs don't even find each other if different pseudowires
- VC Type (encapsulation) - PW DOWN - remote VC type mismatch
- MTU - PW DOWN - MTU Mismatch ( must be identifical on both sides)
- Control Word - PW DOWN - control word mismatch ( both on or both off)

Control Word controls the padding, the additional garbage bytes. If a frame is sent with 40 bytes, because the frame is 64 bytes, the router will add zeros until it
becomes 64 bytes, this is called padding. Then if that frame is sent to the CE it might confuse application, however, with Control Word, it records the size of the frame
in CW Length field and ask PE to strips off the bytes before fowarding to the CE so it doesn't get additional garbage.

#### Port mode: Entire physical port = one pseudowire. Carries everything (all VLANs, untagged, trunk) transparently — like a patch cable.
  
#### VLAN mode: One specific VLAN = one pseudowire. Each VLAN can be a separate service to a different destination — like slicing the wire per customer.

You can also setup tunnel selection by specifying which tunnel the pseudowire should use: 
- pseudowire-class pw1
  - encapsulation mpls
    preferred-path interface tunnel1

#### Commands
 ! Check pseudowire status (UP/DOWN), VC-ID, local/remote labels
  show mpls l2transport vc [vc-id]
  
  ! Detailed PW info: encap type, MTU, control word, VC labels, stats
  show mpls l2transport vc [vc-id] detail
  
  ! Summary of all pseudowires on this router
  show mpls l2transport summary
  
  ! Verify targeted LDP session to remote PE
  show mpls ldp neighbor [remote-PE-loopback]
  
  ! Check xconnect binding on the AC interface
  show xconnect all
  
  ! Verify VC label in the MPLS forwarding table
  show mpls forwarding-table labels [vc-label]
  
  ! Check the AC interface status (UP/UP, encap type)
  show interface [AC-interface]
  
  ! Verify MPLS transport label to remote PE (tunnel label)
  show mpls forwarding-table [remote-PE-loopback] 32
  
  ! Check for encap/MTU/control-word mismatches
  show mpls l2transport vc [vc-id] detail | include MTU|encap|control
  
  ! Verify traffic flowing through the pseudowire (byte counters)
  show mpls l2transport vc [vc-id] detail | include packet|byte


---

## Inter-AS L2VPN (Multi-Segment Pseudowire / PW Stitching)

### Why it's needed
A single end-to-end pseudowire requires a single end-to-end LSP, which requires a contiguous LDP/label path, which requires a **shared IGP**. But **IGP domains do NOT span autonomous systems** — each SP runs its own IGP + LDP. So you cannot build one PW across two providers. Instead you build one PW **segment per AS** and **stitch** them at the AS boundary (the ASBR acts as an **S-PE / switching-PE**). RFC 6073. Used for inter-provider Carrier Ethernet (e.g., a bank's E-Line crossing two carriers) and to split a long PW for independent per-segment OAM.

Real-world note: it's a **boundary / wholesale** tool (not everyday config). Within your own AS use a single end-to-end PW. New builds increasingly move this use case to **EVPN over SR**; stitched MS-PW is the legacy/exam way.

### The flow
```
PE1 ──PW VC 500──► ASBR-AS1 ──PW VC 500──► ASBR-AS2 ──PW VC 500──► PE2
       (seg 1)      (stitch)     (seg 2)     (stitch)    (seg 3)
                        ▲            ▲            ▲
                   targeted LDP  inter-AS    targeted LDP
                   PE1↔ASBR1    (no IGP)    ASBR2↔PE2
                                targeted LDP
                                ASBR1↔ASBR2
```

- **PE1 / PE2** = true endpoints — simple `xconnect` to their LOCAL ASBR (they think the ASBR is the remote PE).
- **ASBR-AS1 / ASBR-AS2** = S-PEs — each STITCHES its two segments with `l2 vfi <name> point-to-point` (exactly 2 neighbors — NOT `manual`, which is VPLS/multipoint). The S-PE pops the incoming VC label and pushes the next segment's VC label; the customer frame transits untouched.
- **VC ID must match per segment PAIR** (500↔500, etc.). You *can* use different VC IDs per segment, but both ends of each segment must agree.
- **Inter-AS link only:** no IGP — but `mpls ip` + static route to the peer ASBR loopback + targeted LDP (because IGP doesn't cross the boundary). Within each AS, PE↔ASBR loopback reachability + LSP come from the normal IGP/LDP.

### Configuration example

**PE1 (endpoint, AS1):**
```
interface FastEthernet0/0
 description to CE1
 no ip address
 xconnect <ASBR-AS1-loopback> 500 encapsulation mpls
```

**PE2 (endpoint, AS2):**
```
interface FastEthernet3/0
 description to CE2
 no ip address
 xconnect <ASBR-AS2-loopback> 500 encapsulation mpls
```

**ASBR-AS1 (S-PE) — inter-AS link + stitch:**
```
interface GigabitEthernet4/0
 description Inter-AS to ASBR-AS2
 ip address 172.16.136.1 255.255.255.252
 mpls ip
!
ip route <ASBR-AS2-loopback> 255.255.255.255 172.16.136.2   ! reach peer loopback (no IGP)
mpls ldp neighbor <ASBR-AS2-loopback> targeted ldp          ! targeted LDP across boundary
!
l2 vfi INTERAS point-to-point
 neighbor <PE1-loopback>      500 encapsulation mpls   ! segment 1 (toward PE1, via AS1 IGP)
 neighbor <ASBR-AS2-loopback> 500 encapsulation mpls   ! segment 2 (toward ASBR-AS2, inter-AS)
```

**ASBR-AS2 (S-PE) — mirror:**
```
interface FastEthernet3/0
 description Inter-AS to ASBR-AS1
 ip address 172.16.136.2 255.255.255.252
 mpls ip
!
ip route <ASBR-AS1-loopback> 255.255.255.255 172.16.136.1
mpls ldp neighbor <ASBR-AS1-loopback> targeted ldp
!
l2 vfi INTERAS point-to-point
 neighbor <ASBR-AS1-loopback> 500 encapsulation mpls   ! segment 2 (toward ASBR-AS1, inter-AS)
 neighbor <PE2-loopback>      500 encapsulation mpls   ! segment 3 (toward PE2, via AS2 IGP)
```

### Verify
```
show mpls l2transport vc 500          ! on each node — all segments UP
                                      ! (ASBRs show BOTH neighbors/segments)
show mpls ldp neighbor <peer-loop>    ! targeted LDP Oper across the inter-AS link
! CE1 <-> CE2 on same subnet reach each other at L2
```

### Key rules
1. `point-to-point` (2 neighbors) = stitch; `manual` = VPLS multipoint.
2. VC ID matches per segment pair.
3. Inter-AS link: no IGP, but MPLS + static route to peer loopback + targeted LDP.
4. Each AS builds its own segment's LSP internally; ASBRs only need targeted LDP + loopback reachability across the boundary.
5. Inter-AS **L2VPN** (stitched MS-PW) is rarer than inter-AS **L3VPN** (Options A/B/C); modern direction is EVPN over SR.

### VPLS & H-VPLS (Multipoint L2VPN)


### Chapter 11: Virtual Private LAN Service (VPLS)

Virtual Private LAN Service (VPLS) emulates a LAN segment across the MPLS backbone across pseudowires or virtual circuits.
It is an evolution of Ethernet over MPLS, because EoMPLS is one-to-one, point-to-point, where VPLS is like a virtual switch. All PEs would learn
the mac addresses of other CEs, by replicating the broadcast and multicast frames to more than one port. It has dynamic mac addresses learning and mac-addresses aging

If a PE router receives a frame that has an unknown destination mac-address, the frame is replicated and forwarded to all ports that belong to that LAN segment. The 
LAN segment on an etherhet switch might be a collection of ports belonging to the same VLAN. When configuring VPLS, you must specify which VPLS instance a particular
port or vlan belongs to. 

If a CE router sends a brodascat frame to the PE router, the frame is replicated and forwarded to all physical ports on that PE router belonging to the VPLS instace, but 
also to all pseudowires associated with that VPLS instance.


VPLS Components:
1. VFI (Virtual Forwarding Instance)
- The virtual switch on each PE
- Contains mac address table + list of pseudowires to remote PEs + local attachment circuits
- Equivalent of a bridge domain or VLAN on a physical switch

2. Full Mesh of pseudowires
- Every PE in the VPLS instance must have a pseudowire to every other PE
- With N PEs: N*(N-1)/2 pseudowires total
- Signaled via targeted LDP (same as AToM) using a common VPN-ID

3. Mac Address learning
- PE learns source MAC from frames arriving on ACs and on pseudowires
- If destination MAC is known -> forward to specific AC or PW (unicast switching)
- If destination mac is unknown -> flood to all ACs and ALL PWs (except incoming)

4. Split Horizon Rule
- A frame received from a pseudowire is never forwarded to another pseudowire, only to ACs, physical customer facing ports
- this prevents loops in the fullmesh PW topology
- Without split-horizong broadcast storms would occur

With split-horizon: 
  1. CE1 sends broadcast → arrives at PE1
  2. PE1 floods to PE2 AND PE3 (via pseudowires) AND to any other local ACs
  3. PE2 receives the broadcast on the PW from PE1
  4. PE2 forwards to its local ACs only — does NOT send to PE3's PW (split-horizon blocks it)
  5. PE3 receives the broadcast on the PW from PE1
  6. PE3 forwards to its local ACs only — does NOT send to PE2's PW
  
  Result: Every CE receives exactly ONE copy. No loops. No storms.
 

**Signaling**
Same as AToM - targeted LDP between each pair of PEs:
- Each PE advertises a VC label per VPLS instance to every other PE
- VPN-ID (VC-ID) must match across all PEs in the same VPLS instance
- Full Mesh means: with 4 PEs, each PE has 3 targeted LDP session for that VPLS.

**Scalability Problem**
Full mesh = N×(N-1)/2 pseudowires. With 100 PEs that's 4,950 PWs. Solutions: 
1. Hierarchical VPLS (H-VPLS): Split into hub (N-PE) and spoke (U-PE). Spokes only peer with hub - hub does the full mesh, reducing PW count.
2. BGP based VPLS (RFC 4761): Use BGP auto-discovery instead of manual LDP neighbor config. PEs find each other automatically.
  
**Configuration**
l2 vfi customer-c manual
 vpc id 300
 neighbor 8.8.8.8 encapsulation mpls
 neighbor 17.17.17.17 encapsulation mpls

interface vlan 100
 no ip address 
 xconnect vfi customer-c
 
it is also possible to tunnel protocols over VPLS network so customer network does look like a big layer 2 switch, by tunneling CDP, VTP, STP
There are two approaches to tunnel STP:
1. Tunnel STP (Transparent - customer controls STP):
   - PE tunnels BPDUs across VPLS
   - Customer's STP sees all sites as one L2 domain
   - Customers' root bridge blocks redundant paths
   - Risk: Customer STP failure can create loops across the SP backbone

2. Block STP( SP Controls it - default behavior):
   - PE doesn't forward BPDUs
   - Each customer site runs its own independent STP
   - No risk of customer STP affecting the SP network 
   - But customer can't run end-to-end STP.
  
### VPLS Discovery and Signaling 
Manual: It uses LDP and targeted hellos for discovery of the neighbors. You define the neighbors under VFI.
AutoDiscovery with BGP: You use MP-BGP to send automatically discover the neighbors you configure under address-family l2vpn vpls.

Auto-discovery via RD/RT, exactly like L3VPN:
 - Each PE advertises its membership in a VPLS instance as a BGP NLRI
 - RD makes each PE's advertisement unique (same as L3VPN)
 - RT controls which PEs import it - RT defines the VPLS domain membership 
 - When you add a new PE, it advertises via BGP, and every other PE with the matching RY auto-discovers it. No manual neighbor config, no manual full-mesh of pseudowires.

**Signaling (label allocation via "label blocks") - This is the clever part unique to VPLS**
Instead of signaling a separate pseudowire label to each remote PE one-by-one, each PE assigns
 - VE ID (VPLS Edge ID) - a unique number per PE within the VPLS instance (eg., PE1=1, PE2=2, PE3=3)
 - A label base (LB) + VE block offset + VE block Size - a block of labels advertised in a single BGP update.
 - A remote PE derives the specific VC label to reach a given VE ID with simple arithemtic:

The label block is a pre-reserved array of labels, one slot per Remote PE. Instead of the PE saying here is a label for PE2, PE3, PE4 etc, the PE reserved a block of
label upfront - with one slot for every possible remote PE (VE ID) - and advertises the whole block in a single BGP update.

Concrete example
PE1 advertises: Label Base = 1000, VE Block Offset = 1. That creates this array of slots:

Slot (remote VE ID)	Label reserved
VE ID 1 (PE1 itself)	1000 (skipped — that's itself)
VE ID 2 (PE2)	1001
VE ID 3 (PE3)	1002
VE ID 4 (PE4)	1003
Now each remote PE computes its own label to reach PE1, using its own VE ID:

PE2 (VE ID 2) → to reach PE1:  1000 + (2 − 1) = 1001
PE3 (VE ID 3) → to reach PE1:  1000 + (3 − 1) = 1002
PE4 (VE ID 4) → to reach PE1:  1000 + (4 − 1) = 1003
PE1 sent ONE advertisement (the block). PE2, PE3, PE4 each pick out the slot meant for them. That's the scaling win — one update signals the pseudowires from every remote PE to PE1.

- Configuration wise, you configure BGP peer under l2vpn vpls address family with a route-reflector.
- You configure the VPLS domain:
l2vpn vfi context VPLS-A
 vpn id 100
 ve id 2
 rd 65000:100
 route-target 65000:100

The route-target defines which PE belongs to the VPLS domain. If another PE joins the RT 65000:100 it will automatically learn this PE labels/configuration.

1. PE2 advertises its VPLS NLRI (RD + RT 65000:100 + label block) via BGP
2. RR reflects it to all other PEs
3. Every PE importing RT 65000:100 auto-discovers PE2 and computes
   its PW label from PE2's block
4. Full mesh forms — no per-PW neighbor config anywhere

### H-VPLS - Hierarchical VPLS
H-VPLS is an extension of traditional VPLS technology designed to adress scalability challenges in large-scale service provider networks. 
H-VPLS addresses scaliablity challenges by introducing a hierarchical structure to the vpls network, typically consisting of two layers:

- Core Layer: This layer copmrises a set of core PE routers that are responsible for interconnecting multiple VPLS edge networks. The core PE routers participate
only in the Core VPLS instance and are not directly connected to customer sites.
- Edge Layer: This layer consists of multiple VPLS Edge networks, each managed by a set of edge PE routers. The edge PE routers connect directly to customer sites
and participate in the VPLS instance service thos sites.

Customers attach to U-PEs via attachment circuits (not PWs). Each User-PE connects to its N-PE with one spoke PW.
The full mesh of PWs exists only among the N-PEs in the core.
Customer ──AC──► U-PE ──spoke PW──► N-PE ═══mesh PWs═══ N-PE ──spoke PW──► U-PE ──AC──► Customer
          (1)            (2)              (3, full mesh)         (2)              (1)

Basically here, the UPE maps the customer into a VLAN and configured PW to N-PE. If we have 100 customer we need 100 PW to NPE. That's where scalability problems happen.
Then we have QinQ to fix this or EVPN




**QinQ** **Dot1q Tunneling** 
QinQ adds a second VLAN tag (S-tag/outer tag) on top of the customer's existing VLAN tag (C-tag/inner tag), creating a double-tagged frame.
The S-tag identifies the customer/service to the SP, while the C-Tag remains untouched inside - letting multiple customers reuse the same VLAN IDs without conflict.

  ! PE/U-PE access port facing the customer
  interface FastEthernet0/0
   switchport mode dot1q-tunnel     ← enables QinQ (adds S-tag to all incoming frames)
   switchport access vlan 500       ← S-tag value (SP uses this to identify the service)
  
  ! PE trunk port toward the core
  interface GigabitEthernet1/0
   switchport trunk encapsulation dot1q
   switchport mode trunk            ← carries double-tagged frames into the network

Result: Customer sends [C-tag 100][payload] → PE adds S-tag → frame becomes [S-tag 500][C-tag 100][payload] → SP switches based on S-tag only, never touches C-tag.

Customer - VLAN 100 - U-PE -> trunk port to N-PE -> PW VFI to another N-PE -> trunk port to U-PE decapsulates and forwards to customer -vlan 100.


### Commands

Check VFI status and pseudowire neighbors:
- show vfi [name]

Verify pseudowire status (UP/DOWN) and VC Labels
- show mpls l2transport vc [vc-id] [details]

Summary of all L2 transport circuits
- show mpls l2transport summary

Verify targeted LDP sessions to remote PEs
- show mpls ldp neighbor [ip] [detail]

Check MAC address table (which mac learned on which PW or AC)
- show bridge-domain [id]

Verify L2 bindings (VC Labels exchanged per VFI)
- show mpls l2transport binding

Check pseudowire signaling details and MTU/encap negotiation
- show mpls l2transport vc [vc-id] detail

Verify the physical AC interface status
- show xconnect all
  
Check split-horizon and forwarding per VFI
- show l2vpn vfi [name]

### Carrier Ethernet (G.8032 / CFM / OAM)


### Ethernet Ring Protection Switching
- It is another form to prevent loops on a ringed network, with sub-second failover.Ethernet rings use specific connections to protect the entire Ethernet ring.
This special link is called a **ring protection link RPL**. A ring link is connected by two adjacent Ethernet ring nodes and ring link ports (also called as ring ports).

- Loop avoidance in an Ethernet ring is achieved by ensuring that, at any time, traffic flows on all but the ring protection link.
- The following are RPL types (or RPL nodes) and their functions:
  - RPL Owner: The owner responsible for blocking traffic over the RPL so that no loops are formed in the ethernet traffic. There can be only one RPL owner in a ring.
  - RPL Neighbor Node: The ethernet ring node in adjacent to the RPL. It's responsible for block its end of the RPL under normal conditions. 
  - RPL Next Neighbor Node: Next neighbor node is an ethernet ring node adjacent to an RPL owner node or RPL neighbor node. It is mainly used for FDB flush optimization on the ring.

Nodes on the ring use control messages called Ring Automatic Protection Swithcing (R-APS) messages to coordinate the activities of switching the ring protection link on and off.

CFM PROTOCOLS AND LINK FAILURES
**Connectivity Fault Management (CFM)** and line status messages are used to detect ring link and node failures.

Three States of an ERPS(Ethernet Ring Protection Switching) Ring.

Idle (normal): Only RPL is blocked
Protection (Failure): The failed link is blocked, RPL is unblocked
Recovery: transitions back to RPL-only

How detection works (the CFM part)
- CFM (Connectivity Fault Manager) sends continuous CCM (Continuity Check Messages) between ring nodes. If a node stops hearing CCMs failure is detected.
- CFM is the important one because it catches failures that don't show as a local port-down.

Now the failed link is repaired:
- The recovered nodes send R-APS (NR) — "No Request" — along the now-restored link. This announces "the fault is gone." 
But they do NOT immediately unblock — that would risk a loop. 
- The RPL Owner receives R-APS(NR) → it knows the ring is healthy again. It starts the WTR (Wait-To-Restore) timer (prevents flapping if the link is unstable). 
- After WTR expires, the RPL Owner RE-BLOCKS the RPL port — restoring the normal blocking point — and sends R-APS (NR, RB) = "No Request, RPL Blocked."
- All other nodes receive R-APS(NR, RB) → now that the RPL is safely blocked again, they unblock their previously-blocked ports and flush their MAC tables. 
- Ring is back to Idle: RPL blocked, everything else forwarding.

FAILURE sequence:


1. A ring link fails.
2. Adjacent nodes DETECT it (CFM CCM loss, or line-down).
3. They BLOCK their ports facing the failed link.
4. They send R-APS (SF) — Signal Fail — around the ring.
5. RPL Owner receives R-APS(SF) → UNBLOCKS the RPL
   (the ring is now broken elsewhere, so the RPL can safely carry traffic — no loop).
6. All nodes FLUSH their MAC/FDB tables and relearn.
   Traffic now flows using the (previously blocked) RPL.

Even in a 20-switch ring, only the RPL Owner blocks one port. The other 19 switches forward on both ring ports normally. 
That single block is enough — it breaks the one loop. So no, not every switch has a blocked port.

A G.8032 ring can support multiple instances. An instance is a logical ring running on a physical ring. There are several reasons for using such instances, 
such as load balacing vlan across a ring. For example, odd-vlans can run in one direction of the ring, and even vlans in another direction. 

**Ethernet CFM Maintenance Domain**

A maintenance domain is an administrative scope of Ethernet connectivity you monitor. MD levels of (0-7) let customer(5-7), provider (3-4), and operator (0-2) domains
nest on the same path without interfering - higher levels are broader, lower levels pass higher-levels fromes transparently. MEPs mark the domain edges and send CCM heartbeats;
losing them detects a fault - which is exactly what triggers G.8032 ring protection.

So CFM/MD is the Ethernet OAM framework that answers "who is responsible for monitoring which part of the path, and how do we detect/locate faults" — 
and the CCM heartbeat is the piece that ties into the ring protection you were just reading about.

**The real "why use Maintenance Domains"**
The purpose is fault demarcation and accountability across administrative boundaries.
Picture that London→Frankfurt circuit crossing 3 networks. It breaks. Without maintenance domains, everyone points fingers — "it's not us." 
Maintenance domains let each party monitor their own scope independently, so you can pinpoint: "the customer's segment is fine, the provider's is fine — the fault is in operator 2's segment."

**Two things it gives you:**
- Fault localization — each domain knows if the problem is inside its scope 
- Non-interference — the MD levels (0-7) let all three parties run CFM on the same wire without their monitoring traffic colliding (higher levels pass transparently through lower ones)
If everything is within one operator (like a ring), you just use one domain at one level. The hierarchy only earns its keep when boundaries are crossed.

Each domain runs CCMs at its OWN fixed level:

- Customer domain MEPs → CCMs at level 7
- Provider domain MEPs → CCMs at level 4
- Operator domain MEPs → CCMs at level 1
Each set of MEPs only monitors its own level. So:

If level-1 CCMs go missing → the fault is in the operator's segment
If level-1 is fine but level-4 CCMs are missing → fault is in the provider's scope
If everything lower is fine but level-7 breaks → it's outside all of them (the customer's own equipment)
The level of the CCM that went missing tells you which domain owns the problem. 


Each domain's MEPs monitor only their own level. So you compare which levels are healthy vs broken:

- Level-1 (operator) CCMs missing, but 4 & 7 fine  → fault in the OPERATOR's segment
- Level-1 fine, but level-4 CCMs missing           → fault in the PROVIDER's scope
- Everything lower fine, but level-7 breaks         → CUSTOMER's own equipment
The logic: the narrowest (lowest) level that reports the fault localizes it to that party's scope. 
- Because the operator's domain is small/nested, if its CCMs drop, you know the break is inside the operator's piece. 
- If the operator's level is healthy but the provider's isn't, the fault is somewhere in the provider's scope outside any single operator's segment — and so on up.

So it's not just "CCM knows which level is missing" — it's that each party independently watches its own level, and by seeing which levels still have heartbeats vs which went silent, you pinpoint whose segment owns the fault.

**MEP (Maintenance End Point)**
- Sits at the EDGE / boundary of the domain 
- ACTIVE — it sources and sinks CFM frames: generates CCM heartbeats, initiates loopback (ping) and linktrace (traceroute)
- Defines where the domain begins and ends 
- Has an MPID (e.g., mpid 170)
- Analogy: border guard — actively sends/receives the monitoring messages 
- MEPs come in PAIRS — you need one at each end of a service. CCMs flow MEP ↔ MEP. 

**MIP (Maintenance Intermediate Point)**
- Sits INSIDE the domain, between MEPs
- PASSIVE — does NOT generate CCMs. It forwards CFM, but responds to loopback/linktrace 
- Its main job is fault isolation — you trace/ping to it to find WHERE a break is 
- Has no MPID 
- Analogy: a checkpoint inside the territory — doesn't initiate anything, but answers when you trace to it
Important: CCMs stay INSIDE the domain

CCMs flow between MEPs of the SAME domain/level — they are contained inside the domain, NOT sent outside. A MEP is the domain WALL: it contains its own level (stops it leaking out) and passes HIGHER levels through transparently.

MEP ──── MIP ──── MIP ──── MEP
edge    inside   inside    edge
(active) (passive)(passive)(active)
CCM heartbeats flow MEP ↔ MEP
Loopback/linktrace can target MIPs to isolate WHERE a fault is

***The three CFM roles — who is who***

- Customer (Subscriber): buys the service (the bank/enterprise). Owns only their own sites. 
- Service Provider: SELLS the service, holds the customer contract, accountable end-to-end — but may not own all the physical network. 
- Operator: owns/runs a PHYSICAL network segment the provider uses. One provider stitches together several operators.
Mapping to London→Frankfurt: Customer = bank; Provider = BT (sold the circuit); Operators = BT-UK network + transit/submarine operator + Deutsche Telekom.

**What is OAM (Operations, Administration, Maintenance)**

OAM is the umbrella term for the network's health-check and troubleshooting machinery — NOT customer data. It's the frames/protocols the network generates to watch over itself.

It answers 3 questions:

Is it UP? → fault detection (CCM heartbeats)
WHERE's the break? → fault isolation (loopback / linktrace)
Is it meeting SLA? → performance monitoring (delay, jitter, loss)
Every technology has its own OAM:

Ethernet OAM = CFM (802.1ag): CCM, loopback, linktrace
Ethernet Performance OAM = Y.1731 (delay/jitter/loss)
Single-link OAM = 802.3ah (EFM)
MPLS OAM = LSP ping / LSP traceroute
Fast liveness OAM = BFD (sub-second detection)
Same purpose (detect / isolate / verify), different layer. When you hear "OAM," think "monitoring and fault-finding, not customer traffic."

**Provider Bridge 802.1ad (QinQ)** **Dot1q Tunneling** 
QinQ adds a second VLAN tag (S-tag/outer tag) on top of the customer's existing VLAN tag (C-tag/inner tag), creating a double-tagged frame.
The S-tag identifies the customer/service to the SP, while the C-Tag remains untouched inside - letting multiple customers reuse the same VLAN IDs without conflict.

  ! PE/U-PE access port facing the customer
  interface FastEthernet0/0
   switchport mode dot1q-tunnel     ← enables QinQ (adds S-tag to all incoming frames)
   switchport access vlan 500       ← S-tag value (SP uses this to identify the service)
  
  ! PE trunk port toward the core
  interface GigabitEthernet1/0
   switchport trunk encapsulation dot1q
   switchport mode trunk            ← carries double-tagged frames into the network

Result: Customer sends [C-tag 100][payload] → PE adds S-tag → frame becomes [S-tag 500][C-tag 100][payload] → SP switches based on S-tag only, never touches C-tag.

Customer - VLAN 100 - U-PE -> trunk port to N-PE -> PW VFI to another N-PE -> trunk port to U-PE decapsulates and forwards to customer -vlan 100.


**PBB (MAC-in-MAC)**
MAC-in-MAC encapsulation technique - wraps the customer frame inside a provider mac header.
- B-MAC (Backbone) - the outer provider mac added by that encapsulation. It's the source/destination mac the backbone actually
switches on.
- C-MAC - service instance in PBB header that tells you which customer service the frame belongs to.
Packet Structure:
- [MPLS Label][B-MAC header][C-MAC][Payload]
The packet is forwarded based on B-MAC header.

---


---

## Chapter 13: Advanced MPLS Services (EVPN)

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
---

