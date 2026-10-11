# E11: SP Security

**Platform:** IOS-XRv 9000 | **Topology:** `00_topology_reference.md`
> **NIC Mapping:** NIC2=Gi0/0/0/0, NIC3=Gi0/0/0/1, NIC4=Gi0/0/0/2, NIC5=Gi0/0/0/3
**Prerequisite:** E02

**End Goal:** Full security baseline — auth, Flowspec, RTBH, RPKI, filtering.

---

### Task 1: IS-IS authentication (MD5, all 3 SPs)
1. Keychain + `hello-password hmac-md5` on all IS-IS interfaces. Verify adjacency re-forms with auth.

### Task 2: BGP authentication (all sessions)
1. `neighbor <IP> password <KEY>` on all iBGP (PE↔RR) + eBGP (PE-CE) + inter-AS (E-R6↔Gar-R7).

### Task 3: BGP TTL Security (GTSM)
1. `neighbor <IP> ttl-security` on all eBGP sessions. Only TTL 254 accepted.

### Task 4: BGP Flowspec
1. `address-family ipv4 flowspec` on PEs + RRs. `flowspec / address-family ipv4 / local-install interface-all`.
2. DNS amp rule: match dest/32, UDP, src-port 53, len>512 → drop.
3. SYN flood: match TCP SYN !ACK → rate-limit 10M.
4. Redirect to scrubbing VRF.

### Task 5: RTBH (dest + source-based)
1. Dest RTBH: inject victim /32 with community, next-hop Null0.
2. Source RTBH: inject attacker source /32 → Null0 + uRPF loose on ingress.

### Task 6: RPKI
1. `router bgp / rpki server <IP> / transport tcp port 323`. Route-policy: deny Invalid, prefer Valid.

### Task 7: Filtering
1. Bogon filter on all eBGP. AS-PATH filter on PE-CE. Max-prefix per peer type.

## Checklist
```
[ ] IS-IS auth (MD5) + BGP auth (all sessions) + GTSM
[ ] Flowspec: DNS amp drop, SYN rate-limit, redirect VRF
[ ] RTBH: destination + source-based
[ ] RPKI validation
[ ] Bogon + AS-PATH + max-prefix filtering
```
