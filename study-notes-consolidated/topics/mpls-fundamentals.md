# MPLS Fundamentals

*Sources: MPLS Fundamentals (Luc De Ghein) + SPCOR Ch 10. Covers labels, forwarding, LDP, CEF, OAM, host-route optimization, troubleshooting.*

---

## Part 1: MPLS Fundamentals

### Chapter 1: The Evolution of MPLS
- MPLS is BGP Core-free - Routers in the backbone network doesn't need to run BGP, only CE-PE needs to run BGP. In the
backbone network routers can just run IP routing protocols such as OSPF, IS-IS. CE-PE runs BGP.
- Basic idea of traffic engineering is to steer the traffic to the best path, not necessarily the shortest path.

### Chapter 2: MPLS Architecture
- MPLS label is a field of 32-bits with a certain structure.
- First 20 bits in label value
- 21-22 is EXP bits for QoS
- 23 is Bottom of Stack Label. Unless this is the bottom label in the stack, if so this bit is set to 1.
- 24-31- TTL Bit to prevent routing loops
- FEC(Forwarding Equivalent Class): Is a group or a flow of packets that are forwarded along the same path and are treated
the same with regard the forwarding treatment. All packets that belongs to the same FEC gets the same label imposed by
ingress LSR. Basically if there is several packets to the same ip prefix, these packets will get the same label.
At each transit hop, the label changes (swap), but all packets belonging to that FEC still travel together — 
they all get swapped to the same new label at each hop. 
The label value changes per hop, but the path and treatment remain the same for all packets in that FEC.
- LSR creates local binding. It bings a label to IPv4 prefix. LSR then distribute this binding to all its LDP neighbors.
- The incoming label is the label from the local binding on the particular LSR.
- The outgoing label is the label from the remote binding chose by the LSR from all possible remote bindings.


### Chapter 3: Forwarding Labeled Packets
Label Operations:
- SWAP: means the top label in the stack is replaced with another
- PUSH: means the top label is replaced with another and then one more additional label are pushed onto the label stack
- POP: the top label is removed

If ingress LSR receives an IP packet and forwards it as labeled, it is called _IP-to-Label_ forwarding case.
- show ip cef: will show the tag imposed (label)

CEF switching is the only IP switching mode that supports MPLS, it needs to be turned on in Cisco devices with
ip cef command.
- show mpls forwarding-table [network[mask/length]] detail: shows all the labels that change on an already labeled
packet. show mpls forwarding-table 10.200.254.4 detail

- Cisco doesn't load balance labeled packets with IPv4. If there are 2 links, one labeled and another non-labeled,
Cisco use labeled path only.

#### Note:
When you issue the command: "no mpls ip propagate-ttl" on the ingress PE hides the entire MPLS core from end-user traceroute 
by setting MPLS TTL to 255 instead of copying IP TTL. The core becomes invisible — appears as a single hop. 
This is used by SPs for security and topology abstraction. Same principle as cloud networks hiding internal fabric.


#### Reserved Labels
- Label 0-15 are reserved labels. LSR cannot use them in normal case for forwarding packets
- Label 0 - Explicit Null Label
- Label 1: Route Alert - used to signal LSR that the packet contains information requiring attention. OAM or RSVP
- Label 2: IPv6 Explicit Null Label
- Label 3: Implicit Null Label: An egress LSR assigned implicit null label back to the LSR neighbor to a FEC if it doesn't
want to assign a label to that FEC, thus requesting upstream LSR to perform a pop operation. Implicit Null label is also 
called Penultimate Hop Popping (PHP). Implicit Null label doesn't deliver EXP bit for QoS. It basically tells its
LSR neighbor to pop the label before forwarding the packet, so when the packet arrives at egress LSR, it does only ip lookup.

Ipv4 -> Label -> Label -> Label->Ipv4 <-Implicit Null Egress LSR Ipv4

Note: Cisco only advertises implicit null label for connected routes and summarized routes.

- Label 0: Explicit Null label: Explicit Null Label (O) can deliver EXP bit, QoS. Egress LSR look at label 0, it does a lookup
and forward the packet. This case egress LSR will see the label but it will do two lookups, one for label 0, remove it and then
ip lookup. 
- Explicit null vs implicit null produces identical traceroute output, the difference is only inside the egress PE forwarding
pipeline - explicit null allows QoS classification via EXP bits before label removal, it never adds an entra hop.
Basically explicit null the egress PE receives the labeled packet with label 0, removes the label, decrement the IP TLL and forwards the packet.

#### Unreserved Labels: 
Because label value has 20 bits, the labels from 16-1,048,575 are used for packet forwarding. 
This is enough for normal IGP prefixes, running OSPF, IS-IS. Cisco default range is 16-100,000.
For BGP you may need more labels due to large prefixes tables, then you need to setup using:
- mpls label range min max 

#### TTL of Labeled Packets:
For SWAP: IP TLL is kept, MPLS TTL is decreased, when it hits the IPv4 look, it decreases the IP TTL.
For POP: Removes the label TTL, and decreases IP TTL
For PUSH: Add the label on top and decreses IP TTL Adding MPLS TTL.

#### MPLS MTU
If you know the number of label a LSP can have, you can modify the MPLS MTU with the command:
- mpls mtu _number_
Let's say 2 label of 4 bytes each would give you MTU of 1508.

#### MPLS Maximum Receive Unit (MRU)
Basically, changes the size that a packet can be forwarded on a link depending on the label operation. 
If POP: it has more room for packet because label was dropped, hence more room for packet.
If PUSH: it has less room as the label is added to the packet.
if SWAP: It doesn't change.

### Chapter 4: Label Distribution Protocol (LDP)
LDP Hello messages are UDP port 646 on multicast address 224.0.0.2
- show mpls ldp discovery detail: See Hello Times and Hold Times
- show mpls interfaces: which interface LDP is enabled
LDP ID is the highest ip on the interface or loopback
LPD ID must be an ip that is in the routing table or else doesn't form LDP neighborship.

LSR tries to open a TCP Connection - port 646 to te other LSR, if it is up they negotiate session parameters, such as:
 - Timer; Label Distribution Method; Virtual Path Identifier (VPI)
 - show mpls ldp neighbor IP detail
 - show mpls ldp parameters

#### Note: 
When a router has multiple links toward another LDP router, the same transport address must be advertised on all 
parallel links that uses the same label space, for example: loopback address.
- mpls ldp discovery transport-address IP
- show mpls ldp/ip bindings - LIB on the LSR
  - in-label: refers to local binding
  - out-label: refers to remote binding
  - LDP identifier can also be found

#### MPLS IGP Syncronization: 
If LDP session has a problem on that interface, OSPF advertises the max cost on that interface to prevent packets from traversing
non-labeled LSP. Once LDP is back up then ospf removes the max metric from the interface and let mpls packets flow normally.
For fast failure detection requires tuned IGP timers or BFD. MPLS convergence time = IDP detection time + SFP computation + LDP update.
The failure has to be on the other end so it detects the failure locally on interface running mpls and change its metric.
- show ip ospf mpls ldp interface gi1/0

#### MPLS IDP Session Protection: 
When a network flaps, LDP and IGP has to establish adjacency and advertise routes and labels all over again.
This can cause outage on a network, and to avoid this you can protect the LDP session as long as there is another path to 
reach that LSR. LDP adjancecy is removed on the port that is down but LDP session stays up through alternate path.
- mpls ldp session protection [vrf name] [for acl] [duration seconds]

When you setup session protection, it automatically creates a targeted ldp session via loopbacks. if a direct link drops, labels survive while OSPF/IS-IS reroutes
When the link between two LSRs goes down, the session stays up so LFIB still have the labels, the traffic flow follows different LSP, 
but once the link is back up, the labels are already there, no need to rebuild the LFIB. Traffic might failover depending on IGP settings.


### Chapter 6: Cisco Express Forwarding (CEF)
- Packets can be forwarded through the router in three basic ways: 1/Process switching, 2/ Interrupt switching, 3/ASIC (application specific integration circuit)
- _Process switching_: the slowest of all switching methods. When switching a packet throguh the router, a Cisco IOS process copies of the packet to CPU memory
and looks up at destination IP in the routing table.
- _Fast Switching_: Build a cache called IP Fast Switching route cache. Some timers govern cache entry. If a packet doesn't traverse the switching 
table for a while its removed from cache.
- Build on demand as packets traverse the router. -- show ip cache verbose to see the cache table.

- _CEF Switching_: Switching table is no longer build on demand, built in advance.
- Each prefix in the routing table has an entry in the CEF switching table at the same time.
- CEF Swtiching has two main data structures: FIB(Forwarding Information Base or CEF table); Adjacency table
- The adjacency table is responsible for MAC or L2 rewrite. The L2 rewrite string contains the new L2 header that is used on the
forwarded frame. For Ethernet, this is the new destination and source mac-address and the ethertype.
- An important aspect of the CEF table is that recursive prefixes are immediatelly used. If, for instance, a BGP prefix is in the routing table and it
points to a BGP next-hop - which is learned via IGP - the BGP prefix is inserted into the CEF table with the next-hop that is learned from recursing 
to the BGP next hop.

- show ip bgp 10.10.10.10 - next-hop 100.100.100.100
- show ip cef 10.10.10.10 - next-hop 2.2.2.2 eth0/0 label 23
- show ip cef 100.100.100.100 - next-hop 2.2.2.2. eth0/0 

_Load Balacing_: 1/Per Packet: The load balancing of all packets is round-robin per packet on the outgoing links.
- ip load-sharing per-packet - and you need to configure this command on all the outbound interfaces if you want to configure per-packet CEF load balancing.
- default behavior is per destination, sourceip, destinationip.
Cisco IOS can load-balance in CEF by hashing the source and destination IP address and pointing the result of that hash to a load sharing table.
- This table holds 16 buckets, each of the 16 hash buckets points to one adjacency, and multiple buckets can point to the same adjacency.
- show ip cef x.x.x.x internal / show ip cef exact-route source_address destination_address 
16 hash buckets exist. These hash bucketd distribute the load of traffic among all possible outgoing paths in the best possible way. 
For example: In the case of 2 outgoing paths, 8 hash buckets are assigned to each outgoing path. In the case of 3 outgoing path, 5 hash buckets
are assigned and one bucket is unassigned.
- Each bucket has an outgoing interface, so for each src+dst it hashes to a bucket, so every packet with that src+dst will go to the same bucket which has the same outgoing interface

---
### MPLS OAM
Helps Service Providers to maintain operationality and functionality of label switch paths to quickly isolate forwarding issues and increase fault detection rate.
2 Main techniques:
1. MPLS Ping: Used for testing end-to-end connectivity and validating reachability of LDP signaled LSP.
2. MPLS LSP trace: Used for trace mpls network, hop-by-hop.

### LSP Host-Route Optimization
Restricts LDP to allocate labels to /30 within Core MPLS Network when Core MPLS network uses loopbacks /32 to forward traffic.
a customer sends a packet, that packets ingress PE router which adds vpn label for /24 route. Then it pushes the MPLS label on top of the vpn label, that top label
is usually /32 loopback label to travel through Core MPLS Network. LSP Host-Route reduce the label creating for the point-to-point links between LSRs, when they only forward
LSPs through loopback.
- mpls ldp label allocate global host-route
Traffic gets to a Core P router, it looks at the label, it looks at CEF and forwards to the interface, (which doesn't necessary needs a label /30), it needs ldp adjacency.

### Chapter 13: Troubleshooting MPLS Networks

**Traceroute in MPLS Networks**

Standard IP traceroute sends UDP probess with incrementing TTL. When TTL expires on a P router, that route generates ICMP time exceeded. The source ip of that ICMP
message is the interface address where the packet entered the P router - this reveals the internal MPLS topology to customers.

How traceroute works through MPLS (default behavior):
1. Ingress PE receives IP packet, copies IP TTL into MPLS TTL (this is TTL propagation)
2. Each P router decrements the MPLS TTL
3. When MPLS TTL hits 0 on a P router, the p router generates ICMP time exceeded
4. Customers sees every P router hop in their traceroute output

**TTL Propagation Control**
- no mpls ip propagate-tll
Configure on ingress PE routers, disables copying the IP TTL into the mpls label TTL.
- MPLS TTL starts at 255 (regardless of IP TTL Value)
- P routers decrement MPLS TTL, but it never reaches 0 in a normal-sized network
- Customer sees the entire MPLS Core as a single hop
- Both ingress-labeled and locally generated packets are affected

Customer traceroute will look like:
  1  CE → PE (ingress)
  2  PE (egress) ← entire core hidden, appears as one hop
  3  CE (destination)


There is another command: - no mpls ip propagate-ttl forwarded
Same as above but ONLY affects forwarded (transit) packets — packets coming from customers through the MPLS core.

- Locally generated packets by the SP (e.g., SP's own traceroute from PE to PE) still propagate TTL normally
- SP operators can still see their core hops when troubleshooting
- Customers cannot
  
! On ingress PE:no mpls ip propagate-ttl forwarded
  
**Important detail:** This only affects packets that get labeled at the ingress PE. Configure it on ALL ingress PE routers for consistent behavior.
  
- mpls ip ttl-expiration pop
  
When MPLS TTL expires on a P router and the router needs to generate ICMP Time Exceeded:
  
- By default, the P router pops the label(s) and looks at the IP payload to build the ICMP response
- mpls ip ttl-expiration pop controls how many labels are popped to reach the IP header for the ICMP reply
  
The nuance with VPN (2-label stack):
  - The P router has a packet with 2 labels (IGP + VPN). MPLS TTL expires.
  - P router needs to generate ICMP Time Exceeded. To do so, it needs to read the IP header underneath BOTH labels.
  - P router pops both labels, reads IP header, builds ICMP with source = its own interface IP, destination = original source IP
  - The ICMP reply needs to be routed back — but the destination is a VPN address. The P router doesn't have VRF context.
  - Result: ICMP might not reach the customer (P router can't route VPN addresses in global table)
  
This is why traceroute through VPN can show * * * for intermediate hops — the P routers can generate ICMP, but the reply can't find its way back to the customer CE.

**MPLS MTU**

1. Adding labels increases the frame size:
- 1 label = + 4 bytes -> 1504 bytes frame for 1500 bytes ip packet
- 2 labels L3VPN - 8 bytes -> 1508 bytes
- 3 labels (Inter-AS Option C, TE+VPN)= +12Bytes -> 1512 bytes
If a core link has L2 MTU of 1500 bytes, labeled packets between 1501-1512 bytes get dropped or fragmented.

The solution is to configure MPLS MTU on core interfaces: mpls mtu 1512
This tells the router to accept/send L2 frames up to 1512 bytes for labeled traffic.

2. Lower IP MTU on PE-CE interfaces to 1492: ip mtu 1492
  
Forces TCP MSS negotiation to smaller values. Impractical across many customers.
  
3. Rely on Path MTU Discovery — end hosts receive ICMP "Fragmentation Needed" and reduce packet size. Unreliable because many firewalls filter ICMP.
Best practice: Set MPLS MTU to 1512-1524 on ALL core-facing interfaces. Accounts for worst-case 3-label stacks.
  
Verifying:
- show mpls interfaces detail | include MTU
  
**Ping for MPLS Troubleshooting**
  
Testing MTU problems:
Test if 1500-byte packets pass through the MPLS core:
ping 8.8.8.8 source 2.2.2.2 size 1500 df-bit

Sweep to find exact breakpoint:
ping 8.8.8.8 source 2.2.2.2 sweep 1400 1510 1
Sends pings from 1400 to 1510 bytes, incrementing by 1
First failure = your effective MTU limit

Ping with record route: 
ping 8.8.8.8 source 2.2.2.2 record
Adds IP Record Route option — response shows every hop's IP address
Useful to verify the ACTUAL path taken (not just expected path)
  
**MPLS-Aware NetFlow**

- Netflow collects flow statistics. MPLS aware Netflow adds mpls specific fields to the flow records:
  - Label value (top labe, second label, up to 3 labels)
  - label position in the stack
  - EXP bits value per label
  - End-of-Stack (EoS) bit
  - Label Type (LDP, RSVP-TE, BGP, unkonwn)
  - Prefix the label is bound to

Why this is useful:
- Traffic forensics: which label carry the most traffic
- Capacity planning: per VPN traffic volumes
- Billing: charge customers per-VPN based on actual usage
- Troubleshooting: verify traffic is actually labeled (vs IP forwarded)

Configuration:
- interface giga1/0 
  - ip flow ingress
  - mpls netflow egress

ip flow-export destination x.x.x.x 9996
ip flow-export version 9 (Version 9 supports MPLS fields)


Quick Comparison with AWS Cloud
Netflow is the VPC Flow Logs: Top Talkers, Source IP, Port, Traffic type, etc.
SNMP is your CloudWatch Metrics: Link Utilization, bytes in bytes out, etc.
Syslogs is your CloudWatch Logs: Data events, state changes, errors


### Unified MPLS
Unified MPLS is also called seamless MPLS. When a provider grows in a way that a single IGP can become a burden, they split the IGP domain into multiple IGPs and build
end to end LSP across all these domains for services like L3VPN.

the core technology is BGP labeled Unicast (BGP-LU)
 - Each IGP domain run its own IGP + LDP/SR internally (isolated - loopbacks not redistributed between domains)
 - BGP LU carries the PE loopbacks (with labels) across domain boundaries - so a PE in the access domain can reach a PE in another region's loopback via labeled BGP path.

PE-A ←iBGP-LU→ ABR1 ←iBGP-LU→ ABR2 ←iBGP-LU→ PE-B
In this scenario ASBR knows how to reach loopback in ospf domain and asbr 2 knows how to reach loopbacks in is-is. both ASBRs use a routing protocol, and exchange
loopback information for reachability.
There is also an option to have single ASBR running both OSPF and ISIS, and ibgp sessions is established between PE and ASBR

### MPLS Internet Options
VPNs are isolated customers learning prefixes only on that VRFs, but they often need internet access too. The internet is in the global routing table, so there are 
ways to make the customer connect to internet when configuring vrf aware customers.

**Option 1: VRF-Specific Default Route**
This is the simplest approach, you inject a default route into the customer VRF, pointing to the internet gateway.
The VRF gets a default route whose next-hop resolves in the global table, using global command. 

**Option 2: Separate PE-CE Interface**
The customer gets a separated connection to the PE where it learns the internet routing table.

**Option 3: Extranet with Internet VRF (Internet in its own VRF)**
Put the internet routing table in its own VRF, like internet vrf, then use route-target leaking to selectively share routes between the customer VRF and the internet VRF.

**Option 4: VRF-Aware NAT (NAT between VRF and Global/Internet)**
The PE performs NAT that is VRF-Aware, translating customer VRF addresses in public addresses as traffic crosses from the VRF to the global internet table.
The PE receives the packet in vrf and nat into global public addresses.