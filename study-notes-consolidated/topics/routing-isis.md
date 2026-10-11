# IS-IS (SPCOR Ch 5)

- Link State Protocol
- Incremental / trigger updates
- Updates are send as unicast / multicast (Layer 2)
- Metric = default (Cisco)
- Administrative distance = 115
- Widely used in ISP environment
- Protocol independent, it supports IP, IPX, Apple Talk 

- Fast convergence:
  - default hello = 10 sec - dead timer = 3 times of hello 

- Default metric in ISIS is 10 on the interface.

- ISIS has a 2 layer hierarchy.
  - Level 2 (the backbone, similar to area 0)
  - Level 1 ( the other areas )

We have:
- L1 routers - intra area routing
  - Establish neighborship with only L1 and L1/L2 routers 
  - It acts like OSPF totally stub
  - It generates a default route pointing towards nearest l1/l2 router
  - L1 router only exchange routes with other L1 routers or L1/L2 within the area
  - L1 forms neighborship with other routers that are in the same area only.
  
- L2 router - inter-area routing
  - responsible for exchange routes between areas - L2 -> L2 or L2 -> L1/L2 
    - There is a level mismatch is when the L2 tries to establish neighbor with L1 router, they won't establish neighborship
    - it forms neighborship with other areas.

- L1/L2 router - intra and inter area routing
  - Usually the border routers connecting to other areas 
  - it establishes neighbor with L1 and L2 routers
  - Default behavior of every router

Addressing in ISIS
- NSAP - Network Service Access Point- The simplest NSAP format used by most companies running IS IS as their IGP is as follows:
  - AFI set to 49.
    - Reserved for Private Use
  - Area ID:
    - Must be at least one byte
  - SystemID:
    - Defines as ES or IS in an area. Cisco implements a fixed length of 6 octets for the SystemID
  - NSEL (NSAP selector):
    - Always set to 00 for a ISIS in router.
- 49.0001.0000.0000.0007.00
  - 49 - Private Use
  - 0000.0000.0007 - SystemId - 48 bits - it must be unique in the network, usually host ip address 
  - 0001 - Area ID
  - last part 00 - Nselector address - always 00, means implementing routing, no transport layer.



DIS - Designed Intermediate System 
- ISIS support only broadcast network and point to point 
- in case of point - to - point there is no DIS election
- In case of broadcast network there is DIS election. the highest priority value becomes DIS ( default is 64)
- if the priority value matches, the highest mac-address will become the DIS
- There is no backup DIS, in case R1 goes down R2 becomes DIS, if R1 is back, then it comes back as DIS.
- to manually change you do isis priority VALUE inside the interface 

ISIS Metric
the highest metric wins, the default is 10
- show clns interface X
- isis metric X to change it

ISIS Authentication
for authentication is ISIS, is similar to EIGRP, you need to create the key-chain and then use the key chain under the interface
 - key chain ccie
   - key 1
     - key string CISCO
 - interface fa0/0 - 
   - isis authentication mode (text/md5) level-1 (depending on the type of neighborship they are forming) 
   - isis authentication key-chain ccie




