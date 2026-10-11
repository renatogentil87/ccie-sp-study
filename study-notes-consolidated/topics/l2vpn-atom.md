# L2VPN — AToM (Point-to-Point Pseudowires / VPWS)

*Sources: MPLS Fundamentals Ch 10 + SPCOR Ch 11.*

---

### Chapter 10: Any Transport over MPLS (AToM)

AToM allows you to carry Layer 2 frames (ethernet, frame relay, ATM, PPP, any transport) across an MPLS backbone transparently.
The customer thinks they have a direct wire between two sites, but it is actually traversing the MPLS.

Key Components:

**Pseudowire (PW)**
- Virtual point-to-point connection between two PEs
- Emulates a physical wire across the MPLS network
- Identified by a VC-ID (both PEs must agree on the same VC-ID)

**Attachment Circuit (AC)**
- Physical interface on the PE that connects to the customer CE
- This interface becomes a pure L2 port - no ip address, no routing
- whatever frames arrive on this port get shoved into the pseudowire

**Targeted LDP Session**
- PE uses a special LDP session (not the regular link-based one) to exchange VC labels
- this ldp runs directly between the two PE loopbacks (targeted = no adjacent peers)
- It negotiates the VC label and pseudowire parameters

**Two Label Stack**
- Transport labe (top): gets the packet from ingress PE to egress PE (regular LDP/TE Label)
- VC Label (bottom): Tells the egress PE which pseudowire (which AC) to send the frame out

**Control Word (Optional)**
- a 4-byte header between the label stack and the layer 2 payload
- used for sequencing, padding small frames, identifying the payload type
- Required for some encapsulations (Frame Relay, ATM); optional for ethernet

HOW IT WORKS:
  CE -> Ethernet Frame -> PE -> Push VC Label + Transport Label -> P router MPLS -> PE -> Pop Labels forward new frame - CE

  1. R1 sends a normal Ethernet frame to R2
  2. R2 doesn't look at IP — just takes the entire L2 frame
  3. R2 pushes 2 labels: transport (to reach R8) + VC label (to identify the pseudowire)
  4. P routers swap only the transport label — they never see or touch the customer frame
  5. R8 receives the packet, uses the VC label to identify which AC to forward to
  6. R8 strips all labels and sends the raw Ethernet frame out toward R9
  7. R9 thinks R1 is directly connected on the same LAN
   
  Comparison with L3VPN
  
  ┌────────────────────┬───────────────────────────────────┬────────────────────────────────────┐
  │                    │ L3VPN                             │ AToM (L2VPN)                       │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ PE involvement     │ Routes customer traffic (Layer 3) │ Switches customer frames (Layer 2) │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ PE-CE protocol     │ BGP/OSPF/Static                   │ None — pure L2                     │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Customer awareness │ PE knows customer IP prefixes     │ PE doesn't know/care about payload │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Label bottom       │ VPN label (from BGP vpnv4)        │ VC label (from targeted LDP)       │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Label top          │ Transport (LDP/TE)                │ Transport (LDP/TE) — identical     │
  ├────────────────────┼───────────────────────────────────┼────────────────────────────────────┤
  │ Signaling          │ MP-BGP vpnv4                      │ Targeted LDP                       │
  └────────────────────┴───────────────────────────────────┴────────────────────────────────────┘
   
**IOS Configuration (basic)**
When you configure xconnect command, it automatically creates the target ldp session.
The only prerequisite is that both PEs have mpls ldp discovery targeted-hello accept configured.

  
On R2: 
  interface FastEthernet0/0
   xconnect 8.8.8.8 100 encapsulation mpls
   ! 8.8.8.8 = remote PE loopback
   ! 100 = VC-ID (must match on both sides)

On R8: 
  interface GigabitEthernet1/0
   xconnect 2.2.2.2 100 encapsulation mpls
  
  That's it — two lines per PE. The interface becomes L2, targeted LDP establishes automatically, VC labels are exchanged, pseudowire comes UP.

The PEs must agree on:
- VC-ID: PW never forms - PEs don't even find each other if different pseudowires
- VC Type (encapsulation) - PW DOWN - remote VC type mismatch
- MTU - PW DOWN - MTU Mismatch ( must be identifical on both sides)
- Control Word - PW DOWN - control word mismatch ( both on or both off)

Control Word controls the padding, the additional garbage bytes. If a frame is sent with 40 bytes, because the frame is 64 bytes, the router will add zeros until it
becomes 64 bytes, this is called padding. Then if that frame is sent to the CE it might confuse application, however, with Control Word, it records the size of the frame
in CW Length field and ask PE to strips off the bytes before fowarding to the CE so it doesn't get additional garbage.

#### Port mode: Entire physical port = one pseudowire. Carries everything (all VLANs, untagged, trunk) transparently — like a patch cable.
  
#### VLAN mode: One specific VLAN = one pseudowire. Each VLAN can be a separate service to a different destination — like slicing the wire per customer.

You can also setup tunnel selection by specifying which tunnel the pseudowire should use: 
- pseudowire-class pw1
  - encapsulation mpls
    preferred-path interface tunnel1

#### Commands
 ! Check pseudowire status (UP/DOWN), VC-ID, local/remote labels
  show mpls l2transport vc [vc-id]
  
  ! Detailed PW info: encap type, MTU, control word, VC labels, stats
  show mpls l2transport vc [vc-id] detail
  
  ! Summary of all pseudowires on this router
  show mpls l2transport summary
  
  ! Verify targeted LDP session to remote PE
  show mpls ldp neighbor [remote-PE-loopback]
  
  ! Check xconnect binding on the AC interface
  show xconnect all
  
  ! Verify VC label in the MPLS forwarding table
  show mpls forwarding-table labels [vc-label]
  
  ! Check the AC interface status (UP/UP, encap type)
  show interface [AC-interface]
  
  ! Verify MPLS transport label to remote PE (tunnel label)
  show mpls forwarding-table [remote-PE-loopback] 32
  
  ! Check for encap/MTU/control-word mismatches
  show mpls l2transport vc [vc-id] detail | include MTU|encap|control
  
  ! Verify traffic flowing through the pseudowire (byte counters)
  show mpls l2transport vc [vc-id] detail | include packet|byte


---

## Inter-AS L2VPN (Multi-Segment Pseudowire / PW Stitching)

### Why it's needed
A single end-to-end pseudowire requires a single end-to-end LSP, which requires a contiguous LDP/label path, which requires a **shared IGP**. But **IGP domains do NOT span autonomous systems** — each SP runs its own IGP + LDP. So you cannot build one PW across two providers. Instead you build one PW **segment per AS** and **stitch** them at the AS boundary (the ASBR acts as an **S-PE / switching-PE**). RFC 6073. Used for inter-provider Carrier Ethernet (e.g., a bank's E-Line crossing two carriers) and to split a long PW for independent per-segment OAM.

Real-world note: it's a **boundary / wholesale** tool (not everyday config). Within your own AS use a single end-to-end PW. New builds increasingly move this use case to **EVPN over SR**; stitched MS-PW is the legacy/exam way.

### The flow
```
PE1 ──PW VC 500──► ASBR-AS1 ──PW VC 500──► ASBR-AS2 ──PW VC 500──► PE2
       (seg 1)      (stitch)     (seg 2)     (stitch)    (seg 3)
                        ▲            ▲            ▲
                   targeted LDP  inter-AS    targeted LDP
                   PE1↔ASBR1    (no IGP)    ASBR2↔PE2
                                targeted LDP
                                ASBR1↔ASBR2
```

- **PE1 / PE2** = true endpoints — simple `xconnect` to their LOCAL ASBR (they think the ASBR is the remote PE).
- **ASBR-AS1 / ASBR-AS2** = S-PEs — each STITCHES its two segments with `l2 vfi <name> point-to-point` (exactly 2 neighbors — NOT `manual`, which is VPLS/multipoint). The S-PE pops the incoming VC label and pushes the next segment's VC label; the customer frame transits untouched.
- **VC ID must match per segment PAIR** (500↔500, etc.). You *can* use different VC IDs per segment, but both ends of each segment must agree.
- **Inter-AS link only:** no IGP — but `mpls ip` + static route to the peer ASBR loopback + targeted LDP (because IGP doesn't cross the boundary). Within each AS, PE↔ASBR loopback reachability + LSP come from the normal IGP/LDP.

### Configuration example

**PE1 (endpoint, AS1):**
```
interface FastEthernet0/0
 description to CE1
 no ip address
 xconnect <ASBR-AS1-loopback> 500 encapsulation mpls
```

**PE2 (endpoint, AS2):**
```
interface FastEthernet3/0
 description to CE2
 no ip address
 xconnect <ASBR-AS2-loopback> 500 encapsulation mpls
```

**ASBR-AS1 (S-PE) — inter-AS link + stitch:**
```
interface GigabitEthernet4/0
 description Inter-AS to ASBR-AS2
 ip address 172.16.136.1 255.255.255.252
 mpls ip
!
ip route <ASBR-AS2-loopback> 255.255.255.255 172.16.136.2   ! reach peer loopback (no IGP)
mpls ldp neighbor <ASBR-AS2-loopback> targeted ldp          ! targeted LDP across boundary
!
l2 vfi INTERAS point-to-point
 neighbor <PE1-loopback>      500 encapsulation mpls   ! segment 1 (toward PE1, via AS1 IGP)
 neighbor <ASBR-AS2-loopback> 500 encapsulation mpls   ! segment 2 (toward ASBR-AS2, inter-AS)
```

**ASBR-AS2 (S-PE) — mirror:**
```
interface FastEthernet3/0
 description Inter-AS to ASBR-AS1
 ip address 172.16.136.2 255.255.255.252
 mpls ip
!
ip route <ASBR-AS1-loopback> 255.255.255.255 172.16.136.1
mpls ldp neighbor <ASBR-AS1-loopback> targeted ldp
!
l2 vfi INTERAS point-to-point
 neighbor <ASBR-AS1-loopback> 500 encapsulation mpls   ! segment 2 (toward ASBR-AS1, inter-AS)
 neighbor <PE2-loopback>      500 encapsulation mpls   ! segment 3 (toward PE2, via AS2 IGP)
```

### Verify
```
show mpls l2transport vc 500          ! on each node — all segments UP
                                      ! (ASBRs show BOTH neighbors/segments)
show mpls ldp neighbor <peer-loop>    ! targeted LDP Oper across the inter-AS link
! CE1 <-> CE2 on same subnet reach each other at L2
```

### Key rules
1. `point-to-point` (2 neighbors) = stitch; `manual` = VPLS multipoint.
2. VC ID matches per segment pair.
3. Inter-AS link: no IGP, but MPLS + static route to peer loopback + targeted LDP.
4. Each AS builds its own segment's LSP internally; ASBRs only need targeted LDP + loopback reachability across the boundary.
5. Inter-AS **L2VPN** (stitched MS-PW) is rarer than inter-AS **L3VPN** (Options A/B/C); modern direction is EVPN over SR.
