# BGP — Study Notes

**SPCOR Chapters:** 7 (BGP Fundamentals), 8 (BGP Optimization and Convergence)

*Notes to be added as you read chapters 7-8 and lab it.*

---

## Key Concepts from Labs

### BGP Best Path Selection Order
1. Weight (highest, local only)
2. LOCAL_PREF (highest, AS-wide)
3. Locally originated
4. AS-PATH length (shortest)
5. Origin (IGP > EGP > Incomplete)
6. MED (lowest, same neighbor AS by default)
7. eBGP over iBGP
8. IGP metric to next-hop (lowest)
9. Oldest route (most stable)
10. Lowest Router-ID
11. Lowest cluster-list length
12. Lowest neighbor address

### Route Reflectors
- RRs on P routers (not PEs) — failure = IGP reconvergence, not VPN failure
- RT-Constraint (rtfilter): PE tells RR what RTs it wants → RR filters outbound VPNv4
- Must configure rtfilter on BOTH sides (PE + RR)
- Add-Path: RR advertises multiple paths → clients see diversity → faster convergence
- BGP PIC = `bgp additional-paths install` → backup pre-installed in CEF

### SoO (Site of Origin)
- Prevents routing loops for dual-homed CEs
- Set `extcommunity soo` inbound on PE-CE sessions
- PE checks SoO on outbound → doesn't re-advertise route back to same site

### as-override
- Needed when same-ASN CEs at different sites (e.g., R9 and R10 both AS 65001)
- PE replaces customer AS with SP AS before advertising to CE
- Without it: CE rejects route (sees own AS in path = loop prevention)

### Graceful Shutdown (RFC 8326)
- Community GRACEFUL_SHUTDOWN (65535:0) → peers lower LP to 0 → traffic drains
