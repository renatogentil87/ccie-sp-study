# L3VPN — MPLS L3VPN (MP-BGP VPNv4)

**SPCOR Chapter:** 12 (MPLS L3VPN)
**Sources:** MPLS Fundamentals Ch 7, SPCOR Official Cert Guide, Lab notes

---

## Chapter 12: MPLS L3VPN


### Chapter 7 - MPLS VPN
- RD: Route Distinguisher: resolve the problem of having overlapping ip by assigning a unique identifier to destinguish the same prefix from different customers.
- RD is 64-bit field used to make VRF prefixes unique when MP-BGP carries them. 
- Two formats: ASN:nn or ip-address:nn

- RT: Route Target: The communication between sites (Customers A, B, C) is controlled by RTs.
- RT is a BGP extended community that indicates which routes should be imported from MP-BGP into the VRF. 
- Exporting an RT means that the export vpnv4 routes receives an additional BGP extended community
- VRF-to-VRF traffic has two labels in the MPLS network. The TOP label is the IGP label and it is distributed
by LDP or RSVP for TE between all P and PE routers hop by hop. The bottom label is the VPN Label that is advertised
by MP-BGP from PE to PE. P routers use the IGP label to forward the packet to the correct egress PE router. The egress PE
router uses the VPN Label to forward the ip packet to the correct CE router.

- BGP Multiprotocol extensions and capabilities. BGP peers send each other the capabilities that they support. The ones
both peer share can then be used. 
- Sends an Open Message to its peer, it includes the capability optional parameter, listing all capabilities of this BGP peer
- show ip bgp neighbors can show the capabilities

- Multiprotocol Extension for BGPv4 define two new BGP Attributes:
  - Multiprotocol Reachable NLRI
  - Multiprotocol Unreachable NLRI
- These attributes advertise or withdrawn routes. both of them hold two fields. Address Family Identifier (AFI) and 
Subsequent Address Family Identifier (SAFI).
- Basically it tells what is being carried. AFI could be Ipv4, IPv6, AppleTAlk. SAFI can be multicast, ipv4 and label.

#### Note: Only BGP extended communities are sent by default to the vpnv4 neighbors. If you want to use standard communities
you need to use send-community both for the bgp neighbor.

#### Route Reflectors
- An RR is a BGP speaker that reflects routes from other BGP speaker.
- If you want to use RR with MPLS VPN, the RR should reflect vpnv4 prefixes, which carry labels. RRs only change the label if
they become the next-hop for the routes, which they usually do not.
- RRs differ in another way from the other BGP speakers in the MPLS VPN network. They don't inject vpnv4 routes when RT is not
configured for acceptance on the RRs.
- debug ip bgp vpnv4 unicast updates in - shows the capabilities 

RR Group: You can group RR and combine which one accepts routes. This can help scale the network by creating groups of RR
that knows about specific routes.
- bgp rr-group NN
- ip extcommunity-list 1 permit rt 1:1
- ip extcommunity-list 1 deny rt 1:2
This example, the group 1 would receive prefixes from 1:1 but deny from 1:2
-
### Chapter 9: IPv6 over MPLS (6PE/6VPE)

### Notes
LDP doesn't support IPv6 as of yet.
- In networks that are running MPLS today, the labeled packets might be ipv6 packets, without the need for the P router to run Ipv6.
The solution 6PE and 6VPE are based on this.
- Another method to carry IPv6 in MPLS is Any Transport over MPLS (AToM). With this solution, the MPLS payload is a L2 frame.
On the edge LSR, the frames are labeled and then transported across MPLS backbone through virtual circuit or pseudowire
- Last method to carry IPv6 over MPLS backbone is to use MPLS VPN Solution, to carry IPv6 over IPv4, the CE routers need tunnels between them.
CE routers needs to be dual-stack routers.
- Following are the tunneling methods for IPv6 that you can implement with Cisco IOS today:
  - IPv6 over IPv4 GRE Tunnels
  - Manual IPv6 Tunnels
  - IPv4 compatible IPv6 tunnels
  - ISATAP tunnels

- 6PE is the Cisco name directly carrying IPv6 packets over MPLS Backbone
  - IPv6 doesn't belong to a vpn
  - no vrf interface on PE
  - All IPv6 CE routers can see each other as 6PE runs in the global address space on PE routers.

- Operation of 6PE
  - PE router are dual-stack
  - the ipv6 routing distribution between the PE routers is done via MP-iBGP. MP-iBGP distributes the labels to be used for the specific IPv6 prefixes.
  - This BGP label identifies or tags the IPv6 packet at egress PE. PE use label and LFIB lookup to forward traffic to CE router.
- Configuration:
  - enable ipv6 cef - ipv6 cef
  - enable ipv6 unicast routing - ipv6 unicast-routing
  - under ipv6 address family in BGP you tell the router to send label: neighbor x.x.x.x send-label

- 6VPE- IPv6 in VPN across MPLS backbone
Operation of 6VPE
- it has an MPLS core network running IPv4 IGP and LDP or RSVP for TE
- The edge LSR or PE routers are capable of running ipv6
- the edge LSR or PE routers have vrf that designate the vpns towards the customer CE routers
- full-mesh MP-iBGP session exist between PE routers - ipv6 prefixes
- IPv6 packets are transported with 2 labels: an IGP as the top label and a BGP(VPN) label as bottom label.
- PE and CE have ipv6 routing protocol between them.

---

