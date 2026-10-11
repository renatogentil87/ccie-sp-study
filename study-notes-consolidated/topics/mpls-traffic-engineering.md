# MPLS Traffic Engineering

*Sources: MPLS Fundamentals Ch 8 + SPCOR Ch 14.*

---

### Chapter 8: MPLS Traffic Engineering
### Notes
- MPLS TE provides efficient spreading of traffic throughout the network, avoiding underutilized or overutilized link.
- MPLS TE takes into account the configured bandwidth of the links
- MPLS TE takes links attributes into account (delay, jitter, metric)
- MPLS TE adapts automatically to changing bw and links attributes. It's source based routing is applied to the traffic
engineered load as opposed to IP destination-based routing.

A TE tunnel is unidirectional, because LSP is unidirectional and it has the configuration only on the head end LSR and not on the 
tail end LSR of the LSP.
- TE Database is built from the TE information that the link state protocol sends. This database contains all the links that are enabled
for MPLS TE and their characteristics or attributes. From this MPLS TE database, path calculation (PCALC) or Constrained SPF (CSPF) 
calculates the shortest route that still adheres to all constraings(most importantly, BW) from the head end LSR to the tail end LSR.

- RSVP Path Message:
- When a head end router needs to reserve a path it sends a RSVP PATH message to the next-hop. The next P router will put that BW 
in standby and forward upstream another RSVP Path message. When the RSVP PATH reaches the tail-end it respond back with RESV
message and the label = POP/NUMBER, that RESV Message is forwarded back to the head-end router with a label with outgoing interface.
- The head end use CSPF to find the best and use RSVP to reserve the BW along the path

### Note:
- In some Cisco IOS uses default of 75% of BW available on the interface if any given when configuring rsvp interface command.

There are three ways to configure the tunnel: Dynamic, Explicit (Static), and Semi Dynamic
- Dynamic let's the MPLS-TE router define the path based on the requirements configured on tunnel interface and rsvp interface.
It uses info from type-10 LSA(OSPF) to know the BW available and which path can be selected based on the tunnel bw configured. (PCALC)
- Static (Explicit) - The network operator directly define the path the network will take.
- Semi-Dynamic - You let Dynamic select the path but you put constraints into it, such as exclude some paths from its calculation or 
manually define some paths alongside of dynamic.

on RSVP PATH Message there is BW, ERO (Explicit Route Object) add the next hop.

OSPF OPAQUE LSAs
- LSA Type 9 - Link SLA - can only be advertised locally - Opaque Link
- LSA Type 10 - Area LSA - it is advertised all the area - Opaque AREA. It advertises the contraints of the link in the area.
- LSA Type 11 - Inter-Area - can be advertised across areas - Opaque-AS

Type 10 - 2 type of TLV (Type Length Value):
- Route Address TLV: Loopback Address
- Link State TLV: It contains sub-TLVs - these TLVs are interfaces running MPLS TE

Sub TLVS:
- Sub TLV 1: Contains the link type (point-to-point or multiaccess)
- Sub TLV 2: contains the router id neighbor for p2p or DR id for multi-access
- Sub TLV 3:  Local interface IP address
- Sub TLV 4: Remote interface ip address
- Sub TLV 5: Admin Metric - TE Metric, specify the TE metric
- Sub TLV 6: Max BW - max bw can be used on this link, whatever configured with BW command.
- Sub TLV 7: Max reservable BW - default is the same as interface bw
- Sub TLV 8: Unreserved BW - value is the BW can be reserved with different priorities
- Sub TLV 9: Resource Class

RSVP: TSPEC: Tunnel Specifications - we need reserve of 1.200k
- ERO - Explicit Route Object - the path for the packet

Type 10 LSA contains several Links:
- 1.0.0.0 - local link, for example loopback
- 1.0.0.1 - first interface enabled for MPLS - TE
- 1.0.0.2 - Second interface enabled for MPLS TE

**Semi Dynamic Tunnels**
Two options:1/ Exclude address and 2/Loose Next Hop
 - Exclude a path, so dynamic selects the path but don't use/select another specific path defined on exclude-address. It is used when you don't want to use specific address for some reason.
 - Loose Next Hop: If you have two links or multiple links to the same router, loose next hop will allow the LSR use any link to reach the next-hop.

OSPF DOWN BIT: When a PE router redistribute BGP VPN routes into OSPF (to advertise it to CE when running OSPF (CE-PE)), those routes becomes OSPF LSA.
If CE is connected to another PE that also run OSPF PE-CE, the second PE would learn those routes via OSPF and redistribute them back into BGP creating a loop.
If a PE generates an OSPF LSA from a BGP route, it sets DN Bit (Downward bit) in the LSA option field. This bit says "this route comes from the VPN, not from a real OSPF domain"
- show ip ospf database external x.x.x.x

MPLS TE AUTO BANDWIDTH
Used to prevent waste on BW. When you configure a tunnel with a BW and that BW is not used, auto BW measures the traffic rates on a tunnel, it automatically reduces the BW of a tunnel
and assign more BW to another tunnel. 
- Sampling rate every 300 seconds, it calculates the rate of traffic.
-  Adjustment time - one day by default. It compares the rate of traffic from sampling rates, then auto-bw adjust the higher volume of traffic to the tunnel.
- to prevent issues when traffic is too low and might spike you can configure minimum and maximum values for the bw.
**Configuration**
- Enable Globally - mpls traffic-eng auto-bw timers [frequency]
- interface - tunnel mpls traffic-eng auto-bandwidth
Collect-BW: Auto-bw can measure the traffic tunnels, just calculating the bw, not applying it. Useful when planning to implement auto-bw.

**Affinity and Attribute Flag**
Add an attribute to the link so it can be added or removed from path calculation. Also known as link colouring.
- 32 bits length - hexa character starting from 0x00000000 - 0xffffffff
After configuring attribute-flag you setup the affinity in the tunnel hexa mask hexa.
- When mask = 1 it means the bit is important on attribute flag, if mask =0 not important
0x00000001 - attribute flag - this is equal to - 0000 0000 0000 000 000 000 000 0001 - 1 important bit
0x0000000f - mask - this is equal to - 0000 000 000 000 000 000 000 1111 - mas bit, the last bit is matching which is the important bit.
If the hexa doesn't match the link is excluded from the path calculation.

If the tunnel has affinity flag configured but the links doesn't then the tunnel won't come up as it is not matching the affinity flags.

**NOTE** By default, Cisco IOS doesn't trigger reoptimization when a link in the network is available to TE again, either by configuration or link is operational again.
To enable the optimization when a link becomes operational for MPLS TE: - mpls traffic-eng reoptimize events link-up

MPLS TE Administrative Weight (TE Metrics)
1. IGP Metric
2. TE Metric = IGP Metric by default

R1 - fa0/0 = metric 1 for Fastethernet
TE Metric is the same of IGP metric. MPLS TE always use TE metric to calculate the best path. Lowest TE metric is chosen for TE metric

MPLS TW HOLD AND SETUP PRIORITY
- 8 levels of priority - preferences
- 0 - 7 where 0 is the highest priority and 7 the lowest priority
By default, the setup priority is 7. Assuming a tunnel is using all BW and you configure a second tunnel, the 2nd tunnel doesn't come up.

Tunnel 0 - 500M - SP = 7 = OK = Data Traffic
Tunnel 1 - 500M - SP = 7 = DOWN - Voice Traffic

The priority gives priority to one tunel over another tunnel, for example above, if you chance voice tunnel priority, the tunnel will be prioritised.

HOLD priority is 7 by default
 - Hold priority of tunnel is up status is compared to setup priority of another tunnel that is trying to be established.

Common rule = setup > hold
Setup = 1, Hold = 0 - means " I need priority 1 to establish, but once I'm up, you need priority 0 to displace me"
When a router advertise unreserved BW in LSA or TLV, it sends the unreserved BW with the priority. 
If tunnel0 is 400K and link is 1000M then router advertises different unreserved BW per priority so depending on the BW of the tunnel can be established.

Example of tunnel preemption
Setup:
- Link 1Gbps (1000Mbps Total reservable)
- Tunnel A - hold priority 3, reserved 200Mbps
- Tunnel B - hold priority 6, reserved 600Mbps
- Total reserved = 800Mbps
- Total unreserved = 200Mbps

- Priority 0:
  - Can preempt hold 1,2,3,4,5,6,7
  - Can preempt tunnel A and B - 1000Mbps
- Priority 1:
  - Can preempt hold 2,3,4,5,6,7
  - Can preempt tunnel A and B - 1000Mbps
- Priority 2:
  - Can preempt hold 3,4,5,6,7
  - Can preempt Tunnel A and B - 1000Mbps
- Priority 3:
  - Can preempt hold 4,5,6,7
  - Cannot preempt tunnel A - 200Mbps
  - Can preempt tunnel B = 600 + 200(free) = 800Mbps
- Priority 4:
  - Can preempt hold 5,6,7
  - Can't preemt tunnel A = 200Mbps
  - Can preempt tunnel B = 600 + 200 = 800Mbps

If follows like that until priority 7.

MPLS TE REOPTIMIZATION
- Periodic by default = 1 hour
- Event Driven: When an interface comes back up after failure the tunnel re-routes. If a link change from down to up, the tunnel reoptimize
  - mpls traffic-eng tunnels reoptimize events link-up
- Manual - mpls traffic-eng reoptimization

Feature only for dynamic tunnels
- lockdown feature - we don't want to reoptimize the path, we don't want the change to happen in that tunnel.

MPLS TE AUTOROUTE - Another function to forward traffic to the tunnels, we don't need to configure static route or PBR.
You setup autoroute - mpls traffic-eng autoroute announce on interface and it will be advertise onto the IGP.

**MPLS TE Forwarding Adjacency**
It generates a Router LSA as the tunnel interface. So other routers in the OSPF/ISIS domain will see this tunnel link and depending on the igp metric may prefer
the tunnel to route traffic to the destination.

OSPF sees the tunnel as a link with IGP metric, other route include this link in CSPF. They can only see it a link when forwarding adjacency is enabled on both directions.

MPLS TE CBTS - CLASS BASED TUNNEL SELECTION
Used for QoS

MPLS LABEL = 32 bits [ Label (32) | EXP (3) | BoS (1) | TTL(8)]
EXP = Experimental  = QoS
BoS = Bottom of Stack
IPP - IP Precedence (IP header has ToS Byte - 8 bit length)
ToS - Type of Service 
- There are 2 important parts
  - IPP = 3 bits = 000,001,010,011,100,101,111 (bigger is better)
  - Bigger EXP faster than lower EXP
  - You can setup which tunnel a exp traffic will flow

QoS - Modular QoS CLI - MQC - ClassMap -> Policy Map ->. Service
Configure a master with member tunnels, the router examine the EXP on each tunnel and define which tunnel to forward the traffic

ToS = 000 = 0 = EXP = 0 
      001 = 32 = EXP = 1
      010 = 64 = EXP = 2
      011 = 96 = EXP = 3
      111 = 128 = EXP 4
      100 = 160 = EXP 5
      101 = 192 = EXP 6
      110 = 224 = EXP 7
- You are not allowed to load-balance traffic with the same EXP bits value onto two different tunnels with CBTS.

MPLS TE - Cost Calculation Process
Equal Cost Load Balancer - if the tunnel and IGP have the same metric, the tunnel will be used. If the destination is behind the tunnel tail then load sharing can be done. 
- Path Weight on output is the TE Metric.

MPLS TE AUTOROUTE ANNOUNCE METRIC
3 types of metrics:
1. Direct: It is the default TE metric (IGP Metric) or the manually configured metric with - tunnel mpls traffic-eng autoroute metric [metric] 
   Metrics behind the tunnel endpoint changes and impact the tunnel metric.
2. Absolute: For all networks behind the tail-end the metric shouldn't change. So when you setup absolute, you shouldn't change the metric to network behind the tail-end.
3. Relative: it means you can add or subtract the metric from IGP cost. For example if metric is 10, you can add 2 or remove 2 from the IGP metric to calculate the best path to a destination.

**MPLS TE QoS Models From IntServ to RSVP-TE**
Techniques used to implement QoS:
1. Best Effort: No QoS at all, all devices should give the best effort to send traffic.

2. IntServ (Integrated Service)- old days to implement it. works with RSVP.
Today QoS we don't use RSVP. All hops of the path works together to give the QoS on the path.

3. DiffServ (Differentiated Service)
It uses a technique called per hop behavior (PHB). Each router doesn't reserve the amount of bw.
It is used Queuing per hop instead and each router can implement a different type of QoS per hop.
Most used QoS today in the networks.
RSVP has another function which is label request. Called RSVP-TE

0 - Explicit Null - php remove the label but keeps EXP bit so php router knows how to treat the QoS for given traffic
- mpls traffic-eng signalling interpret explicit-null verbatim to change to explicit null in MPLS TE
- mpls traffic-eng signalling advertise implicit-null - router will send label 3 (implicit null label)

**RSVP-TE Messages**
RSVP PATH Message:
- RRO - Object that you can enable, you can see the path of RSVP Path, and RESV Message is passing.
- tunnel mpls traffic-eng record-route
- In the path message it records all the ip address that path message is passing through, you can see the path of rsvp path message
- The same can be seen in resv message when returning from path message

Two types of Label Distribution:
1. UD: Unsolicited Downstream: MPLS VPN, MPLS AND MPLS L3VPN uses unsolicited downstream method.
2. DoD: Downstream on Demand: downstream router should request label from the neighbor - Cell Mode MPLS and MPLS TE

When using LDP every LSR should advertise label to its neighbor without any request = UD
In Traffic Engineering, when we enable TE - Routers don't advertise label until requested = DoD - via path message/resv message

Label Distribution protocols: TDP/ LDP / BGP/ RSVP-TE

In MPLS TE we have important concept called Make-Before-Break. It is the ability of having a MPLS tunnel formed before switching the traffic on primary path.
First, it signals the new path, reserve the bandwidth and establish the new path, then it tears-down the first path.
This is called making the tunnel before breaking to prevent traffic being interrupted

Shared Explicit (SE Style)
- We have a tunnel with reserved bw. Old Path of TE LSP
- To establish the need tunnel we need the bw but it is reserved so it can't be established the new path.
That's when Shared Explicit Style comes into place. It's no double booking of bandwidth, the old and new bw requirement is not reserved at the same time.
- Because the shared explicit is running only for one time the Tail End understand the new LSP becomes from the old LSP and understand it can reserved the bw before tearing down the old path.
- The highest amount of bw is used. If old LSP has 1M but new LSP has 2M, only 2M will be used, no need to double booking the BW.

RSVP Messages:
- Path, PathErr, PathTear
- Resv, ResvErr, ResvTear

- Path message: is used to reserve the bandwidth, initial process and then it receveis back the resv message.
- PathErr message: is a message sent toward the head-end router. Used to notify the head-end router the path is not available anymore, it can be due to a link failure between LSP.
LSR can also receive PathErr with bogus information, different vendors, compatibility issues. 
- PathTear message: used when head-end router wants to shutdown the tunnel to other LSR in the path. Let's say we shutdown the tunnel on head end router, we notify other routes
that the link is tearing down. It is sent from head-end towards tail-end router.
- ResvErr Message: sent towards tail-end router, used rarely cases. It can't reserve the amount of bw requested.
- ResvTear Message: It is much like resv message, sent in direction as resv message, towards head-end router. Meaning the router is tearing down the reservation.
We don't need the reservation anymore.

- debug ip rsvp dump-messages to debug the rsvp messages

**Link Manager**
Link Manager is part of Cisco router that does link admission control, keeps track of reserved bw reserved by RSVP.
Runs on every router in the path. Tracks how much bandwidth is allocated per interface per priority. Decides admit/reject when an RSVP PATH arrives
In Path Message there is TSPEC = 64000 bytes/sec(example). Router needs to validate whether it has the requested bw to send path message forward to next LSR.
1. That's the function of Link Manager control, it controls the admission of the link, it keep tracks of the available bandwidth.
2. Another function of Link Manager, is preemption, it understand the priorities setup for the tunnel to preempt the tunnels.

3. Trigger IGP to flood link state information: 
OSPF (LSA) /ISIS (LSP) - Incremental Update: Router before advertise LSA to R2, only advertise in case there is a change in LSA, then we advertise the new LSA.
The same happens in ISIS (LSP). OSPF advertise the LSAs every 30 minutes, to ensure all routers receive the latest LSAs. Interval Update :30 min OSPF 15 min ISIS.
Routers need to advertise the change in bandwidth usage on the links. It can't be used Interval Updates because of longer convergence time.
There is a mechanism of tune the advertisement the link usage to other routers. If the usage is bigger than 50%, send LSA to other routers. This is function of Link Manager.

- OSPF ->  default is 30 minutes: router(config-router)# timers pacing lsa-group SECs
- ISIS -> default is 15 minutes: router(config-router)# lsp-refresh-internal SECs

Flooding by the IGP - Link Manager send flooding every 3 minutes with contraints of the links. It can be tuned: mpls traffic-eng link-management timers periodic-flooding SEC
1. Link State Change: A link change, interface added or removed from OSPF
2. Configuration Change: manual configuration of interface, cost etc
3. Periodic Flooding
4. Change in the reserved bandwidth: Manually changed the bw or updated bw on the link. 
5. After a tunnel setup failure: we setup the tunnel for a reason failure. The tunnel should be tear down and the bw should be released.

**show mpls traffic-eng link-management bandwidth-allocation interface_name**
Up Thresholds (available bandwidth increasing — tunnels being removed):
15 30 45 60 75 80 85 90 95 96 97 98 99 100
When a tunnel is torn down and available bandwidth goes UP, OSPF only re-advertises when it crosses one of these percentages of max-reservable.
Down Thresholds (available bandwidth decreasing — tunnels being admitted):
100 99 98 97 96 95 90 85 80 75 60 45 30 15
When a tunnel is admitted and available bandwidth goes DOWN, OSPF only re-advertises when it crosses one of these percentages.
Example: Link has 100 Mbps reservable. 
  1. Tunnel1 reserves 5 Mbps → available = 95 Mbps (95%) → crosses the 95% down threshold → OSPF floods new LSA
  2. Tunnel2 reserves 3 Mbps → available = 92 Mbps (92%) → between 95% and 90%, no threshold crossed → no flood
  3. Tunnel3 reserves 4 Mbps → available = 88 Mbps (88%) → crosses 90% down threshold → OSPF floods
Why it matters for CSPF: Remote headend routers make tunnel path decisions based on the TE topology database. If the database is stale (not updated), a headend might try to signal a tunnel on a link that's
actually full — RSVP will reject it. More aggressive thresholds = more accurate topology but more flooding. The defaults are a balance.

**Auto-Bandwidth**
Auto-bandwidth measures the tunnel's own traffic rate, not the physical link traffic. 
How it works: 
  1. Sampling: IOS periodically reads the tunnel interface's output counter (show interface Tunnel0 — output rate in bps). This is the actual traffic flowing INTO the tunnel.
  2. Collection interval: Every X seconds (default 300 seconds / 5 minutes, configurable with frequency), it records the highest output rate seen during that interval.
  3. Adjustment: At the end of the collection interval, it compares the measured peak rate against the current RSVP reservation:
    - If traffic > current reservation → signal a new reservation with higher bandwidth (up to max-bw)
    - If traffic < current reservation → signal a new reservation with lower bandwidth (down to min-bw)
    - If the difference is below the adjustment-threshold (e.g., 10%) → don't bother re-signaling
  4. Re-signaling: The tunnel does a make-before-break — signals the new bandwidth on the same or new path without dropping traffic

Auto-bw checks only Tunnel interface output counters. If the tunnel needs 80M, it will ask for that, it doesn't monitor the physical interfaces. Whether the interfaces
will admit that 80M request comes down to the link manager to admit it based on the interface utilization.
Think of it this way: if Tunnel0 carries 80 Mbps of traffic but the physical link has 900 Mbps free, auto-bandwidth doesn't care about the 900 Mbps. It just says "my tunnel needs 80 Mbps reserved" and
re-signals.
Whether that 80 Mbps can actually be admitted on every hop — that's the Link Manager's job.
If the tunnel request 10M but the traffic is 100M, the traffic won't get dropped, auto-bw will request for more link, link manager will try to accomodate, if it can that's fine 
and if it can't it respond with reseverr message it can't accomodate that request.

**MPLS TE Fast ReRoute (FRR)**
**Link Protection**
With Link Protection one particular link used for TE is protected. This means that all TE tunnels that are crossing this link are protected by one backup tunnel.
Head-end router has a tunnel with tail end router. It configures a tunnel backup.

R1 -> R2 ---------> R3
          ---R4 --> R3.
Let's say R1 has a tunnel with link protection, R2 is called PLR and R3 MP(Merge Point)
PLR detects the link failure and automatically uses a backup tunnel through a different link, protecting the whole tunnel from head end router to tail end router.
PLR Point of Local Repair
The backup tunnel is also called Next Hop (NHOP). 
When a head-end router of the protected TE tunnel receives the PathErr, it recalculates a new path for the tunnel LSP and signals it.
The backup tunnel doesn't reserve bw, therefore, when protected tunnels use the backup tunnel, it is possible that not enough bw is available to switch the traffic.
Backup tunnel is preconfigured on the PLR router:
- mpls traffic-end backup-path on the protected link
On the head end router of the protected tunnel, you specify that the tunnel can use as a backup tunnel. 
- tunnel mpls traffic-eng fast-reroute
  
! On R3 (PLR) — the transit router
interface Tunnel10 
 tunnel destination 4.4.4.4
 tunnel mpls traffic-eng path-option 1 explicit name BYPASS-R3-R4
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
interface FastEthernet0/0              ← R3's interface toward R4 (the protected link)
 mpls traffic-eng backup-path Tunnel10  ← "if this link dies, use Tunnel10"

**Node Protection**
You protect the whole router, not a single link.
It works by creating a NNHOP backup tunnel. When you configure the command - tunnel mpls traffic-end fast-reroute node-protect on the head end router
it sets the flag to Ox10 in the Session Attribute of the PATH message, indicating that it wants node protection

R1 - R2 - R3 - R4 
Let's say R1 has a tunnel with R4. in Link Protection R2 would be PLR in case of a failed link between R2 - R3. R2 would have a tunnel with R4 with explicit path bypassing that r2-r3 connection.
In Node Protection, R2 would bypass completely R3 and tunnel with R4 as backup.
For node protection, when a node fails, the PLR needs to know what label the NNHOP expects and this is done via RRO (Record Route Object) in RSVP
 - show mpls traffic-eng fast-reroute database
shows:
 -  Protected Interface
 - Backup Tunnel
 - Backup Label
 - Protection Type

 - **Example Configuration**
interface Tunnel0
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 8.8.8.8
 tunnel mpls traffic-eng path-option 1 explicit name VIA-R4
 tunnel mpls traffic-eng fast-reroute node-protect
  
PLR (R3): backup tunnel skips R4 entirely, reaches R5 (next-next-hop):
ip explicit-path name BYPASS-R4-NODE
 next-address 6.6.6.6
 next-address 5.5.5.5
interface Tunnel11
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 5.5.5.5
 tunnel mpls traffic-eng path-option 1 explicit name BYPASS-R4-NODE
interface FastEthernet0/0
 mpls traffic-eng backup-path Tunnel11

**SRLG - Shared Risk Link Groups**
SRLG is used when a backup tunnel can potentially be routed across a link that is on the same fiber or conduit as the protected link.
If you configure the protected link and all other links that share the same fiber with the same SRLG identifier, the backup tunnel avoids those links.
Two Keywords:
 - force: ensure that backup TE tunnel is never routed over a link that has the same SRLG as the protected link. If a link with another SRLG isn't available tunnel doesn't come up
 - preferred: indicates that if a link with another SRLG is not found first to route the backup tunnel across, the backup tunnel is created across a link with the same SRLG.

- mpls traffic-eng auto-tunnel backup srlg exclude [ force | preferred] (global)
- mpls traffic-end srlg NUMBER (interface level)
- show mpls traffic-end topology brief - see SRLG ids.

**Multiple Backup Tunnel**
Multiple backup tunnels can protect the same link or node, and they can terminate at different tail end routers.
They can be a mix of NHOP and NNHOP. the PLR prefers an NNHOP over a NHOP backup tunnel when assigning a protected TE LSP to a backup tunnel.

**MPLS TE and MPLS VPN**
- RSVP-TE replaces LDP for transport to the tunnel destination only
- BGP still provides the VPN label (always — regardless of LDP or RSVP)
- You keep LDP running for destinations without TE tunnels and as fallback
- In a full-mesh TE deployment (tunnel to every PE), you COULD remove LDP — but most SPs don't

**VRF-to-PE Tunnel**
Traffic from one customer goes to one tunnel and traffic from another customer goes to another tunnel
- The problem is that PE router receives an update with the same next-hop for two different tunnel and end up doing load balacing. To solve this problem you setup two different loopback as next hop so PE router
learns the destination with two different IP addresses to other PE loopbacks.

vrf definition NAME
 address-family ipv4
 bgp next-hop Int_name

That would tell PE to use different next hop for each VRF.

**MPLS VPN PE-to-P Tunnel**
we need to connect customer A to customer A in another site via VRFs, there are multiple paths via IGP.
You can configure TE Tunnel from PE to P tunnel to choose the path to send the traffic to steer the traffic and better usage of the bw depending on the cost of the igp.
The problem of establishing a TE Tunnel from PE to P is that the LSR will POP the label to the tail-end tunnel which is P, and that P won't know how to route the traffic to the destination.
Because they will have only VPNv4 Label.
- For this scenario, there is a must to configure LDP.
- P uses LDP to tell PE (Head End Tunnel) which label is necessary to be used in order to send traffic to P router (tail-end router). 
Now there will be two labels, vpnv4 label, and the LDP label which then allow communication to happen.
1. Solution: enable MPLS on the Tunnel Interface so the tunnel can learn label and how to forward labeled packets to P router.
In the capture, you will have 3 labels, VPNv4 Label, RSVP Label and LDP Label. LDP Label to forward the traffic to P router, RSVP to signaling the tunnel and vpnv4 to carry the vpn label.
2. Solution: Targeted LDP Session: so head-end receives the label from P router and knows how to send the labeled traffic, as long as mpls is also enabled on tunnel interface on head-end router.
   - mpls ldp neighbor x.x.x.x targeted ldp 

**SubPool and global Pool**
Subpool is like a premium capacity allocated to the tunnel. Let's say you have 100Mbps on the tunnel and you create a subpool of 50Mbps.
When a tunel is configured with subpool they can consume the 50mbps but it is guaranteed for them, it is taken from the global pool. Example:
tunnel 1 - global pool 100mbps
tunnel 2 - subpool 60mbps
Data flows 40mbps on subpool, so subpool still has 20mbps to server this tunnel, while global pool still have 40Mbps to allocate to its tunnel.
Global pool reservations are consumed only from global pool, and subpool only from subpool.



### Commands
show mpls traffic-eng tunnels tun0
mpls traffic-eng tunnels
ip rsvp bandwidth RESERVED BW
tunnel mode mpls traffic-eng
tunnel mpls traffic-end bw BW
tunnel mpls traffic-eng path-option NAME/NUMBER Dyamic/Explicit
mpls traffic-eng link-management timers periodic-flooding SEC
show mpls traffic-eng link-management bandwidth-allocation
show mpls traffic-eng link-management addimission-control 

---
