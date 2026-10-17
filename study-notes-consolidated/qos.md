# QoS / DiffServ (SPCOR Ch 21)

in QoS you have to do the following:
- Mark the packet EF, AF and so on
- Policy, Shape or prioritise the packet
There are different queueing mechanisms used to prioritise the packet, depending on the use case, you need to use one over another

## QoS Models ##
DiffServ - mostly used in today's network
- designed to overcome the limitations of Best Effort and Intserv models
- Network traffic is identified by classes (class-maps)
- Network QoS policy enforce differentiated treatment of traffic classes
- You choose the level of service for each class 

Class-map 1
    voip traffic priority 1 (more prioritise)
class-map 2
    bulk data priority 2 
class-map 3
    everything else priority 3 (less preffered - drop or buffer if network is congested)

**Modular QoS CLI Components**
1 - Class Map: define classes of traffic (What traffic do we care about?)
    - You can use access-list to match the traffic
    - match source interface too 
2 - Policy Map: Define QoS policies for classes (what will be done to this traffic?)
3 - Service Policy - Apply a service policy (Where will this policy be implemented?) - attach to an interface

**Classification & Marking**

- Used to mark packets based on classification, metering, or both
Classification is the process of identifying and categorizing traffic into classes, typically based upon:
-  Incoming Interface
- IP precedence
- DSCP
- Source or destination address
- Application

Marking is the QoS feature component that "colors" a packet so it can be identified and distinguished from other packets in QoS treatment:
Commonly used markers:
- Link Layer:
  - CoS (ISL, 802.1p)
  - MPLS EXP Bits
  - Frame Relay
- Network Layer:
  - DSCP
  - IP Precedence

Layer 3 Marking - IP Precedence
ToS - 8 bits

IP Precedence Decimal - Binay - NAME
0 - 000 - Routine - Less preferred traffic
1 - 001 - Priority
2 - 010 - Immediate
3 - 011 - Flash
4 - 100 - Flash-Override
5 - 101 - Critical
6 - 110 - Internetwork Control
7 - 111 - Network Control

DSCP
- EF - Priority Traffic - Voice Traffic - Real Time
- AF41,42,43 - highest priority
- AF31,32,33 - Business Critical
- AF21,22,23 - http , database
- AF11,12,13 - Bulk Data, FTP, Backup
- Best Effort 0 lowest priority

AF XY where X = class (the higher the better), Y = drop precedence (1= low, 2-medium, 3-high)

Typical SP QoS Policy Mapping:
class-map VOICE           →  match dscp ef        →  priority (LLQ, strict)
class-map VIDEO           →  match dscp af41      →  bandwidth 20% (CBWFQ)
class-map BUSINESS        →  match dscp af31      →  bandwidth 15%
class-map TRANSACTIONAL   →  match dscp af21      →  bandwidth 10%
class-map BULK            →  match dscp af11      →  bandwidth 5%
class-map NETWORK-CTRL    →  match dscp cs6 cs7   →  bandwidth 5% (protect routing)
class class-default       →  (everything else)    →  fair-queue + WRED

DSCP - MPLS EXP Mapping (3 bits = 8 values):
DSCP is 6 bits (64 values) → EXP is 3 bits (8 values)
Default mapping: EXP = top 3 bits of DSCP

EF  (46 = 101110) → EXP 5
AF41 (34 = 100010) → EXP 4
AF31 (26 = 011010) → EXP 3
AF21 (18 = 010010) → EXP 2
AF11 (10 = 001010) → EXP 1
CS6  (48 = 110000) → EXP 6
0    (00 = 000000) → EXP 0


**Congestion Management**
Congestion management with queueing - It puts the packet in a buffer instead of dropping when there is congestion in the network.
We can classify the traffic and put into queue with different queueing/buffering.

There is input queue and output queue 
Input has only FIFO queue and only single queue, if processor is not too busy then system never used input queue
Output is software queue( FIFO, PQ, WRR), hardware (always FIFO) queue that prioritize packets based on classification.
Software queue sends the packets in order based on the queue to hardware queue that sends the traffic out of the interface.

**Weighted Fair Queueing**

Fair Queue:
- Share resources equally
- high bandwidth traffic will not overcome low bw traffic
- allows the sharing of flows unclaimed with other flows

Weighted Fair Queue
- assign weight on each flow on IP Precedence
- the lower the weight, the higher the priority and the share of the bw.
- it forwards the packet based on packet size + weight. So small packets with the same precedence than bigger packets are forwarded first.

Class Based Weighted Fair Queueing
- You define class-maps to mark the traffic and guarantee bw for this traffic.

Low Latency Queueing
- Combines Priority Queue with CBWFQ 
- Strict Priority queieing for sensitive traffic (voice/video)
- Reduces latency/jitter for voice communications
- Priority command - it is going to guarantee the maximum amount of bw for that traffic 
  - allocates bw in kbps to a class and ensure expedited forwarding
  - traffic exceeding the specified bw is dropped if congestion exists

**Congestion Avoidance**
Tail Drop - if software queue reaches maximum limit, it drops the packets automatically. No differentiated drop, it can be voice traffic, video traffic.
Congestion avoidance - RED - drop the packet until it reaches the limit. Drop is selectively based on the weight WRED. Useful for TCP Syncronization issues, TCP Starvation issues

RED (Random Early Detection)
 - minimum threshold - define the level of packets which the begin the drop packets
 - maximum threshold - decide when to maxium threshold start tail drop
 - maximum drop probability

WRED - you can decide which weight is first dropped, likelyhood one packet will be dropped based on QoS marking or weights setup for given packet.
- Apply on interface or Class Level - default is ip precedence values 

Class-Based WRED - you create class map, decide which traffic will be dropped based on class and define the RED values.

**Traffic Policing**
Policy- If the traffic exceeds a limit, we can rate that in the buffer before getting dropped or remark it.

Shaping: Used to enforce a rate limit by delaying packets, using buffers



