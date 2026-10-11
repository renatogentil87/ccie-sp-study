# SPCOR 350-501 Official Cert Guide — Study Notes\n\n**Authors:** Bradley Riapolov, Mohammad Khalil\n**Published:** Dec 2024\n**Status:** TO BUY\n\n---\n

### Chapter 1 - Service Provider Architectures

**Metro Ethernet**
Refers to the use of technology in metropolitan area networks (MANs) to create high-speed, scalable, and cost-effective network infrastructure.
It's based on industry standard Ethernet technologies, 802.1Q VLAN, and QinQ (VLAN Tunneling)

3 aspects of MetroEthernet services and topologies:
- E-Line
- E-LAN
- E-Tree

**E-Line**: Ethernet Line Service
Establishesa point-to-point or point-to-multipoint ethernet connection between one or more locations. Uses EVC (Ethernet Virtual Circuit).
The client doesn't know he is connecting on a provider, both sites are in the same L3 domain.

**E-LAN**: Ethernet LAN Service
Multi-point-to-multipoint connectivity over a shared network. As if they are in the same LAN. If the service provider is running MPLS, it is VPLS, which eliminates
the need of STP.

**E-Tree** Ethernet Tree Services
Connect multiple sites to the central CE1 site to access shared resources at that site. Similar to a hub-spoke topology.

**Unified MPLS**
Large ISPs don't run IGP in the entire Core network, they breakdown the network into modules, access, aggregation, core and each layer runs its own IGP. 
Because LDP labels remains within the IGP, BGP-LU(Label Unicast) is used to send labels across domain boundaries using loopback interface of each device.
BGP uses label mapping to associate mpls labels with routes during route distribution. 
MPLS label is distributed in the BGP update message.

**Segment Routing**
Modern approach for traffic engineering, instead of many router reserving hundreds of stateful tunnels along with backup paths, only the source (ingress router) specifies
the entire path in advance and can force the traffic anywhere on the network.
Segment Routint TE: Integration with Path Computation Element Protocol (PCEP) controller for traffic policy optimization. Network operator can dynamically optimize 
traffic flows based on real time network conditions and policy requirements.
- BGP-LS: BGP Link State: SRTE can gather network topology and traffic information in real time from BGP-LS

-- Transport Technologies --

**DWDM**: Dese Wavelength Division Multiplexing
Sending multiple light signals on the same fiber, each on different color (wavelength)
One fiber has 80-96 different colors of light. Each color carries 100-400Gbps independently.
Wavelenght is one color of the fiber, shared fiber. So a single shared fiber can carry a lot of data by using DWDM because the DWDM lends a color to the sender without
the need to deploy a new fiber.
ROADM makes the optical layer programmable. You can reroute wavelenghts across the network from a management console.

**DOCSIS** Data over Cable Service Interface Specification: Standard for the transmition of data over cable television (CATV) systems.
It runs in many houses today where coaxial cable is delievered as last mile from neighborhood central to the house.
Laptop > Wireless Router > Modem > Coxial Cable > Neighbor Central > Fiber to ISP Backbone.

There are different frequencis that are used for upstream and downstream, and different versions with different speed limits.
  
  ┌────────────┬──────┬──────────┬──────────┬────────────────────────────────────────────────┐
  │ Version    │ Year │ Max Down │ Max Up   │ Key Change                                     │
  ├────────────┼──────┼──────────┼──────────┼────────────────────────────────────────────────┤
  │ DOCSIS 1.0 │ 1997 │ 40 Mbps  │ 10 Mbps  │ First internet over cable                      │
  ├────────────┼──────┼──────────┼──────────┼────────────────────────────────────────────────┤
  │ DOCSIS 2.0 │ 2002 │ 40 Mbps  │ 30 Mbps  │ Better upstream                                │
  ├────────────┼──────┼──────────┼──────────┼────────────────────────────────────────────────┤
  │ DOCSIS 3.0 │ 2006 │ 1 Gbps   │ 200 Mbps │ Channel bonding (combine multiple frequencies) │
  ├────────────┼──────┼──────────┼──────────┼────────────────────────────────────────────────┤
  │ DOCSIS 3.1 │ 2013 │ 10 Gbps  │ 1-2 Gbps │ OFDM, 4096-QAM, much wider spectrum            │
  ├────────────┼──────┼──────────┼──────────┼────────────────────────────────────────────────┤
  │ DOCSIS 4.0 │ 2020 │ 10 Gbps  │ 6 Gbps   │ Full duplex, symmetrical possible              │
  └────────────┴──────┴──────────┴──────────┴────────────────────────────────────────────────┘
  
The modem converts data bits into RF signals to send the data to the Termination System at the ISP headend.
Your modem (CM) ←── coax ──→ CMTS (Cable Modem Termination System) at the ISP headend
The CMTS is the ISP's router equivalent for cable networks. It terminates all the RF signals from hundreds/thousands of modems, converts them back to Ethernet/IP, 
and forwards into the  ISP backbone.

