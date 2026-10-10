# OSPF — Study Notes

**SPCOR Chapter:** 6 — OSPF

*Notes to be added as you read the chapter and lab it. Key topics: multi-area, summarization, stub/NSSA, virtual links, OSPF-TE interaction, sham-link, DN-bit, convergence tuning.*

---

## Key Concepts from Labs

### OSPF + MPLS Conflict
- OSPF summarization breaks LDP label bindings (LDP needs /32 loopbacks). This is WHY SPs prefer IS-IS (flat L2, no summarization needed).

### Sham-Link
- Creates intra-area OSPF adjacency across MPLS VPN core
- Source/destination = VRF loopbacks advertised via BGP (not OSPF)
- Prevents backdoor link from being preferred over VPN path (intra-area vs inter-area)

### DN-Bit
- Set by PE when redistributing BGP VPN routes into OSPF
- Prevents another PE from re-redistributing back into BGP (loop prevention)

### LDP-IGP Sync
- Configure under `router ospf 1` → `mpls ldp sync`
- OSPF advertises max-metric on interface when LDP session is down
- Test by removing `mpls ip` on the REMOTE side (not local)

### Administrative Distances
- eBGP = 20, OSPF = 110, iBGP = 200
- With eBGP PE-CE: VPN path (AD 20) preferred over OSPF backdoor (AD 110)
- With OSPF PE-CE: backdoor (intra-area) preferred over VPN (inter-area) → sham-link needed

### SPF Throttle and LSA
You can throttle the times the router runs SPF algorithm when a link flap. If a link flaps 10 times in 5 seconds
SPF will run 10 times, burning CPU. Throttle control how long it will run.


## OSPF SPF Timers

SPF throttle controls how quickly and how often OSPF runs SPF after a topology change. Same exponential backoff concept as IS-IS but different syntax order.

**IOS classic / IOS-XE syntax (initial comes FIRST, unlike IS-IS):**

```
timers throttle spf <initial-wait> <hold-time> <max-wait>
timers throttle spf 50 200 5000
                     │   │    │
                     │   │    └── 5000ms ceiling (never waits longer)
                     │   └── 200ms hold (doubles each consecutive SPF)
                     └── 50ms first SPF delay
```

**LSA throttle (same pattern, controls LSA generation):**

```
timers throttle lsa all <initial> <hold> <max>
timers throttle lsa all 50 200 5000
```

**LSA arrival (minimum gap between accepting same LSA from neighbor):**

```
timers lsa arrival 100
```

**IOS-XR syntax:**

```
router ospf 1
 timers throttle spf 50 200 5000
 timers throttle lsa 50 200 5000
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

**CRITICAL — syntax order difference from IS-IS:**

```
IS-IS:  spf-interval <MAX> <initial> <hold>        ← max FIRST
OSPF:   timers throttle spf <initial> <hold> <MAX>  ← max LAST
```

Same three values, different order. Classic exam gotcha.

**Standard SP production values:** `timers throttle spf 50 200 5000`
