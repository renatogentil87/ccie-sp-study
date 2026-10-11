# SR-EX08: SRv6 (Segment Routing over IPv6)

**Topology:** `00_SR_topology_reference.md`
**Prerequisite:** SR-EX01 IS-IS foundation complete. This exercise reconfigures the SR domain to run SRv6 instead of SR-MPLS.

**End Goal:** SRv6 on R3-R6 (IS-IS domain) with IPv6 underlay, locators, SID allocation, L3VPN over SRv6, and SRv6-TE.

---

## Important: SRv6 requires IPv6 on all core interfaces. The packets ARE IPv6 packets — no MPLS.

---

## Section 1: IPv6 Underlay

### Task 1
Configure IPv6 addresses on all core interfaces for R3, R4, R5, R6. Use this scheme:
- Loopbacks: fc00::X/128 (X = router number, e.g., R3 = fc00::3)
- Core links: fc00:0:XY::/64 (XY = router pair, e.g., R3↔R4 = fc00:0:34::/64, ::1 and ::2 per end)

Keep IPv4 addresses — dual-stack.

### Task 2
Enable IS-IS IPv6 address-family on R3-R6:
- `address-family ipv6 unicast` under the IS-IS process
- `single-topology` (IPv4 and IPv6 share the same SPF)
- `address-family ipv6 unicast` under each core interface

All routers MUST use the same topology mode (single-topology). Mismatch = IPv6 routes don't install.

### Task 3
Verify IPv6 reachability: ping fc00::6 from R3 using source fc00::3. All IPv6 loopbacks reachable via IS-IS. Check `show isis ipv6 route` — all /128 loopbacks present.

---

## Section 2: SRv6 Locators and SIDs

### Task 4
Configure SRv6 locator on each router (R3-R6):
- R3: fc00:0:3::/64
- R4: fc00:0:4::/64
- R5: fc00:0:5::/64
- R6: fc00:0:6::/64

Set the encapsulation source-address to each router's IPv6 loopback. Name the locator "MAIN".

### Task 5
Reference the SRv6 locator under IS-IS IPv6 address-family so IS-IS advertises the locator and allocates SIDs.

### Task 6
Verify SID allocation on each router:
- `show segment-routing srv6 sid` — should show End SID (node reachability) and End.X SIDs (per adjacency) auto-allocated within each locator
- `show segment-routing srv6 locator` — locator status active

### Task 7
Verify IS-IS is advertising SRv6 capabilities:
- `show isis database <router>.00-00 verbose` — look for SRv6 Locator and SRv6 End SID entries in the LSP

---

## Section 3: SRv6 Reachability

### Task 8
Traceroute from R3 to R6 using IPv6 (fc00::6). This is a plain IPv6 traceroute — no labels. Transit routers just forward based on IPv6 destination. Each hop shows an IPv6 address, not an MPLS label. This IS SRv6 working for underlay reachability.

### Task 9
Verify the forwarding table for a remote locator:
- `show cef ipv6 fc00:0:6::/64` — shows the outgoing interface and next-hop to reach R6's locator

### Task 10
Understand what changed from SR-MPLS:
- SR-MPLS: traceroute shows MPLS labels (16001, 16006). Same label every hop. PHP at penultimate.
- SRv6: traceroute shows IPv6 addresses. No labels. Transit routers do normal IPv6 routing.
- Compare `show mpls forwarding` (SR-MPLS) with `show segment-routing srv6 sid` (SRv6). Different data planes.

---

## Section 4: SRv6 SID Types

### Task 11
Identify the SID types allocated on R6:
- **End** — reach this node (like prefix-SID in SR-MPLS)
- **End.X** — use a specific link (like adj-SID in SR-MPLS)
- Check: `show segment-routing srv6 sid` — which SIDs are End, which are End.X?

### Task 12
Understand the additional SID types (will appear when VPN is configured):
- **End.DT4** — decapsulate outer IPv6, lookup inner IPv4 in VRF (VPN delivery)
- **End.DT6** — same for IPv6 VRF
- **End.DX4** — decapsulate and forward to specific CE (per-CE delivery)
- For now, only End and End.X are allocated. End.DT4 appears in Section 5.

---

## Section 5: L3VPN over SRv6

### Task 13
Configure BGP VPNv4 with SRv6 encapsulation on R6 (PE facing CE2):
- Under the BGP neighbor (toward the RR), specify `encapsulation-type srv6`
- Under the VRF, configure `segment-routing srv6 / locator MAIN / alloc mode per-vrf`
- This tells BGP to allocate an End.DT4 SID instead of an MPLS VPN label

### Task 14
Verify: `show segment-routing srv6 sid` on R6 — a new End.DT4 SID should appear, mapped to the customer VRF.

### Task 15
Configure the same on R1 or R3 (whichever PE faces CE1). Both PEs must use `encapsulation-type srv6` toward the RR.

### Task 16
Verify BGP VPN routes carry SRv6 SIDs, not MPLS labels:
- `show bgp vpnv4 unicast vrf CUSTOMER <prefix> detail` — next-hop should be an IPv6 address with an SRv6 SID, NOT an MPLS label

### Task 17
End-to-end test: CE1 ping CE2. The customer traffic is encapsulated in an outer IPv6 header with destination = the remote PE's End.DT4 SID. No MPLS anywhere in the forwarding path.

### Task 18
Check the encapsulation on the PE:
- `show cef vrf CUSTOMER <ce2-prefix>` — should show IPv6 encapsulation (outer destination = End.DT4 SID), NOT MPLS label stack

---

## Section 6: SRv6-TE

### Task 19
Create an explicit SRv6-TE policy on R3 to R6 using a segment-list of SRv6 SIDs:
- Segment 1: fc00:0:5:: (End SID for R5 — waypoint)
- Segment 2: End.DT4 SID on R6 (final delivery)

This forces traffic R3 → R5 → R6 instead of the direct R3 → R6 path.

### Task 20
Verify: the SRv6-TE policy is UP. The packet carries an SRH (Segment Routing Header) with the segment list. Transit router R5 processes its End SID, decrements Segments Left, copies the next SID into the IPv6 destination, and forwards.

### Task 21
Compare SRv6-TE with SR-MPLS-TE:
- SR-MPLS-TE: label stack [16005][16006] — MPLS headers
- SRv6-TE: IPv6 header + SRH with [fc00:0:5::, fc00:0:6:40::] — IPv6 headers
- Same concept (source routing), different encoding

---

## Section 7: SRv6 Troubleshooting

### Task 22
SRv6 SIDs not allocated. `show segment-routing srv6 sid` is empty. What to check?
- Is the locator configured under `segment-routing srv6`?
- Is the locator referenced under IS-IS IPv6 AF?
- Does IS-IS have `address-family ipv6 unicast` enabled?
- Is `single-topology` consistent on all routers?
- Is the locator prefix valid (/64)?

### Task 23
IPv6 reachability works (ping) but L3VPN over SRv6 doesn't. BGP VPN route shows MPLS label instead of SRv6 SID. What's wrong?
- Missing `encapsulation-type srv6` on the BGP neighbor
- Missing `segment-routing srv6 / locator MAIN` under the VRF
- Check: `show bgp vpnv4 unicast <prefix> detail` — does it show SRv6 SID or MPLS label?

### Task 24
SRv6 locator format rejected by IOS-XR. What to try?
- /48 requires micro-SID (uSID) mode — may not be supported on XRv
- Try /64 instead (classic SRv6 format)
- Check: `show version` for platform capabilities

---

## Snapshot
Take: **"SR-EX08-srv6"**

## Checklist
```
[ ] IPv6 addresses on all core interfaces (R3-R6), dual-stack
[ ] IS-IS ipv6 AF with single-topology on all routers (MUST match)
[ ] IPv6 loopback reachability verified (ping + show isis ipv6 route)
[ ] SRv6 locator configured per router (/64)
[ ] Locator referenced under IS-IS ipv6 AF
[ ] End + End.X SIDs auto-allocated (show segment-routing srv6 sid)
[ ] IS-IS LSP shows SRv6 locator and End SID (show isis database verbose)
[ ] IPv6 traceroute works — shows IPv6 hops, no MPLS labels
[ ] CEF shows IPv6 forwarding for remote locators
[ ] SR-MPLS vs SRv6 differences understood (labels vs IPv6 addresses)
[ ] End.DT4 SID allocated when VRF + srv6 configured
[ ] BGP VPN routes carry SRv6 SIDs (not MPLS labels)
[ ] CE1↔CE2 ping works over SRv6 (no MPLS in path)
[ ] SRv6-TE policy with SRH and segment list
[ ] SRv6 troubleshooting: locator issues, single-topology mismatch, encap-type
```
