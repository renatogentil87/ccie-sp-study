# Multicast & mVPN (SPCOR Ch 9)

**Introduction**
Class D address range is used only for the group address or destination address of IP multicast traffic

Permanent multicast groups:
 - Non routed purposed ( reserved for routing protocols - 224.0.0.0 - 224.0.0.255)

Source-Specific Multicast (SSM) - 232.0.0.0 - 232.255.255.255

IGMP - Internet Group Management Protocol
- responsible to enable communication between a router and connected hosts.
- Hosts uses this protocol to inform the router they want to join a certain multicast group or leave a certain multicast group

There are 3 versions. IGMPv2 is the default.

Report Message: used by the client to join the group
Query Message: used by router to see if the members of the group still exist "Do you still want to receive the multicast traffic or not?"
 - 224.0.0.1

- Version 2 has leave group message from host to the router (this isn't possible with version1.)
- Version 2 also sends group specific queries. If you have multiple groups, the router can send group specific queries, something it wasn't possible in version 1.

IGMPv3
- an enhacement is that you can select a group and server you want to receive the multicast traffic, source specific multicast.
- (S, G) - source specific multicast server, multicast group 
- (*,G) any source, multicast group


PIM - Protocol Independent Multicast 
- provides loop free router to router communication 
- rely on IGP to figure out the network is loop free (using RPF Check)
- Do not advertise the routes through out the network like IGP

Two Modes:
- PIM Dense Mode: 
  - Simple process, it is considered as implicit join 
  - use flood and prune behavior
  - traffic is forwarded to all the hosts unless the host tells the router it doesn't wanna the traffic (prune message)
  - not applicable for large networks, not efficient in large networks

- PIM Sparse Mode:
  - considered as explicit join
  - Uses RP to process join request
  - RP (root of forwarding path, where join messages will go). Receiver sends a join message to RP, then RP decides how to send the traffic from source to receiver.

Dense Mode floods the traffic to all ports, every router knows all the sources and group address (S,G). If a router doesn't have a host who wants to join the group,
it sends a prune message. 

RPF Check - Reverse Path Forwarding (RPF)
1. Packet Arrival: A router receives a multicast packet on an incoming interface.
2. Source Lookup: Instead of looking at the destination address, the router inspects the source IP address of the multicast packet and searches its unicast routing table (or a dedicated multicast routing table) to find how to reach that source.
3. Interface Comparison: The router compares the incoming interface of the multicast packet with the outgoing interface (or RPF interface) listed in the routing table for that source.
4. Pass or Fail:
	• Pass: If the incoming interface matches the outgoing interface for the source, the RPF check passes. The router replicates and forwards the packet.
	• Fail: If they do not match, the RPF check fails, and the router silently drops the packet.


PIM Sparse Mode-
- There is no flooding, receivers join groups explicitly.
- Sender sends a message to RP, RP build a (S,G) to source.
- Then receiver join the group via RP, the RP has (S,G) and receiver has (*,G) - source tree
- Then receiver sends join message directly to the source, and build a shared tree with (S,G) via shortest mroute path.


RP can be configured statically or dynamically
 - Candidate RP - router advertise itself as RP
 - Mapping Agent - flood the RP information to the multicast domain 