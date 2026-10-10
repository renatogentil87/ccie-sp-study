# IS-IS — Study Notes

**SPCOR Chapter:** 5 — IS-IS

---

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


ISIS SPF Timers
 






## IS-IS SPF Timers

SPF throttle controls how quickly and how often IS-IS runs the SPF calculation after a topology change. Uses exponential backoff to protect CPU during flap storms while maintaining fast convergence for real failures.

**IOS classic syntax (note: max-wait comes FIRST, unlike OSPF):**

```
spf-interval <max-wait> <initial-wait> <second-wait>
spf-interval 5 50 200
              │  │   │
              │  │   └── 200ms hold (doubles each consecutive SPF)
              │  └── 50ms first SPF delay
              └── 5 seconds ceiling (never waits longer)
```

**IOS-XR syntax:**

```
router isis CORE
 address-family ipv4 unicast
  spf-interval initial-wait 50 secondary-wait 200 maximum-wait 5000
```

**OSPF syntax (different order — initial first):**

```
timers throttle spf <initial> <hold> <max>
timers throttle spf 50 200 5000
```

**How exponential backoff works:**

```
Event 1:  wait 50ms → run SPF          (initial)
Event 2:  wait 200ms → run SPF         (hold)
Event 3:  wait 400ms → run SPF         (doubled)
Event 4:  wait 800ms → run SPF         (doubled)
Event 5:  wait 1600ms → run SPF        (doubled)
Event 6:  wait 3200ms → run SPF        (doubled)
Event 7:  wait 5000ms → run SPF        (capped at max)
Quiet period → resets to 50ms           (ready for next real failure)
```

**Why 50/200/5000:**
- 50ms initial: sub-100ms convergence on first failure (customers barely notice)
- 200ms hold + doubling: prevents CPU waste on rapid changes
- 5000ms max: protects CPU during flap storms (caps the backoff)
- Reset after quiet period: instantly responsive again for real failures

**Standard SP production values:** `spf-interval 5 50 200` (IS-IS) / `timers throttle spf 50 200 5000` (OSPF)
