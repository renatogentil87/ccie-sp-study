# Carrier Ethernet — Ring Protection (G.8032 ERPS)

*Sources: SPCOR Ch 11 / Carrier Ethernet. NOTE: not MPLS — Ethernet ring protection.*

---

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
