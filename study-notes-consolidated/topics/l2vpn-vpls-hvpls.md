# L2VPN — VPLS & H-VPLS

*Sources: MPLS Fundamentals Ch 11 (VPLS, discovery/signaling, H-VPLS) + SPCOR Ch 11.*

---

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
