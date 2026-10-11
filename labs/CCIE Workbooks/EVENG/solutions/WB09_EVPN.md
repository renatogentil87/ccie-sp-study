# CCIE SP Workbook 09 — EVPN (Domain 2)

**Platform:** EVE-NG — IOS-XRv 9000
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology focus:**
- **Garnet** — Gar-R1 + Gar-R2 + CE5. CE5 is **dual-homed** to Gar-R1 and Gar-R2 (EVPN, VLAN 100).
- **Gold** — G-R2 + CE7. CE7 is single-homed to G-R2 (EVPN, VLAN 100).
- Inter-AS: **Gar-R6 (Garnet RR) ↔ G-R4** carries BGP EVPN toward Gold, so CE5 (Garnet) can reach CE7 (Gold).

**Reference addressing (used throughout):**

| Node | Loopback0 | Role |
|------|-----------|------|
| Gar-R1  | 3.3.3.3   | Garnet PE, DF candidate for CE5 ES |
| Gar-R2  | 4.4.4.4   | Garnet PE, DF candidate for CE5 ES |
| G-R2  | 6.6.6.6   | Gold PE (CE7) |
| Gar-R6  | 9.9.9.9   | Garnet route-reflector for `l2vpn evpn` |
| G-R4| 13.13.13.13 | Inter-AS EVPN border |

- **CE5 Ethernet Segment ID (ESI):** `0000.0000.0000.0000.0005`
- **EVI 100** = bridged VLAN 100 (MAC-VRF), **EVI 500** = EVPN-VPWS, **EVI 200** = IRB instance
- **BD (bridge-domain) `VLAN100`** maps to EVI 100
- **IP-VRF `TENANT-A`** for symmetric IRB / Type-5 (L3VNI-equivalent label)

**Prerequisites (assumed done in earlier workbooks):**
- IS-IS + Segment Routing MPLS core (Workbook 09-SR), `/32` loopbacks, SRGB 16000–23999.
- iBGP to Gar-R6 (RR) already up for `vpnv4`; we add the `l2vpn evpn` AF here.

> **Convention:** each task is **Question → Solution → Verification**. All syntax is IOS-XR (`evpn`, `l2vpn bridge-domain`, `evi`).

---

## Section 1 — EVPN Route Types (5 tasks)

EVPN is a BGP AF (`l2vpn evpn`) that carries five NLRI route types. Each solves a specific problem. Configure the base MAC-VRF first, then observe each route type.

### Base configuration — MAC-VRF (EVI 100) on Gar-R1, Gar-R2, G-R2

```
! ---- Gar-R1 / Gar-R2 / G-R2 : enable EVPN AF in BGP toward RR (Gar-R6) ----
router bgp 100
 address-family l2vpn evpn
 !
 neighbor 9.9.9.9
  remote-as 100
  update-source Loopback0
  address-family l2vpn evpn
 !
!
! ---- Gar-R6 (RR) : reflect l2vpn evpn ----
router bgp 100
 address-family l2vpn evpn
 neighbor-group RRC
  remote-as 100
  update-source Loopback0
  address-family l2vpn evpn
   route-reflector-client
```

```
! ---- Bridge-domain + EVI mapping (Gar-R1, Gar-R2, G-R2) ----
l2vpn
 bridge group GARNET
  bridge-domain VLAN100
   interface GigabitEthernet0/0/0/1.100      ! CE-facing (dot1q 100)
   !
   evi 100
   !
!
evpn
 evi 100
  bgp
   route-target import  100:100
   route-target export  100:100
  !
  advertise-mac
```

```
! ---- CE-facing sub-interface (Gar-R1/Gar-R2 -> CE5, G-R2 -> CE7) ----
interface GigabitEthernet0/0/0/1.100 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
```

---

### Task 1.1 — Type 2 (MAC/IP Advertisement)

**Question:** CE5 sends a frame with source MAC `0050.5600.0005`. Configure Gar-R1 so it advertises this MAC (and its IP, if ARP is snooped) to remote PEs via BGP instead of relying on data-plane flooding. Explain the purpose of Type 2.

**Solution:**
Type 2 is the workhorse route — it advertises a learned **MAC** (and optionally **MAC+IP** when ARP/ND is snooped) so remote PEs install it in their MAC-VRF without flooding. With the base config above, `advertise-mac` under `evpn evi 100` enables it; local learning on the AC triggers the Type 2 advertisement automatically.

```
evpn
 evi 100
  advertise-mac          ! advertise locally-learned MACs as Type 2
!
! (optional) snoop ARP so MAC+IP is advertised, enabling ARP suppression
l2vpn
 bridge group GARNET
  bridge-domain VLAN100
   arp-suppression
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn evi 100 mac
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 2
! On Gar-R2 the MAC must appear as learned via BGP (remote), not flooded:
RP/0/0/CPU0:Gar-R2# show evpn evi 100 mac detail | i "0050.5600.0005|Next Hop"
!   -> Next Hop 3.3.3.3, learned via BGP (Type 2)
```

---

### Task 1.2 — Type 3 (Inclusive Multicast Ethernet Tag)

**Question:** Configure/verify how BUM (Broadcast, Unknown-unicast, Multicast) traffic is delivered in EVI 100 across Gar-R1, Gar-R2, G-R2. Explain the purpose of Type 3.

**Solution:**
Type 3 (IMET) is auto-generated per EVI per PE as soon as the EVI is up. It advertises "I participate in EVI 100, reach my BUM via this label using **ingress replication**." Each PE builds a replication list of the other PEs' Type 3 routes and head-end replicates BUM to each. No extra config beyond the base EVI.

```
! Ingress replication is the IOS-XR default for EVPN BUM.
! (Explicitly, under the EVI you can confirm the PMSI type is ingress-replication.)
evpn
 evi 100
  ! default: control-word disabled, ingress replication for BUM
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 3
!   -> one IMET route per PE (3.3.3.3, 4.4.4.4, 6.6.6.6), PMSI = Ingress Replication + label
RP/0/0/CPU0:Gar-R1# show evpn evi 100 inclusive-multicast detail
!   -> replication list: 4.4.4.4, 6.6.6.6
! Send a broadcast from CE5 and confirm Gar-R1 replicates to both remote PEs.
```

---

### Task 1.3 — Type 4 (Ethernet Segment Route → DF election)

**Question:** CE5 is dual-homed to Gar-R1 and Gar-R2 with the same ESI. Configure the Ethernet Segment so Gar-R1 and Gar-R2 discover each other and elect a **Designated Forwarder** (DF). Explain the purpose of Type 4.

**Solution:**
Type 4 (Ethernet Segment route) is advertised by every PE attached to a given ESI. Its purpose is twofold: **auto-discovery of ES peers** (so PEs learn who shares the segment) and **DF election** (only the DF forwards BUM toward the multi-homed CE, preventing loops/duplicates). Type 4 carries an ES-Import RT (auto-derived from the ESI) so only PEs on the same ES import it.

```
! ---- Gar-R1 and Gar-R2 : identical Ethernet Segment config ----
evpn
 interface Bundle-Ether5                 ! LACP bundle to CE5
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05
   ! DF election is per-EVI, mod-based by default (see Task 3.4)
```

```
! CE5 is reached via a LACP bundle so both PEs share ESI:
interface Bundle-Ether5
 lacp system mac 0000.0000.0005          ! same LACP system-id on Gar-R1 & Gar-R2
!
interface Bundle-Ether5.100 l2transport
 encapsulation dot1q 100
 rewrite ingress tag pop 1 symmetric
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 4
!   -> ES route from 3.3.3.3 and 4.4.4.4 for ESI ...0005
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment interface Bundle-Ether5 detail
!   -> ES-EAD peers: 3.3.3.3, 4.4.4.4 ; DF election result per EVI
```

---

### Task 1.4 — Type 1 (Ethernet Auto-Discovery: per-ES and per-EVI)

**Question:** Explain and verify the two Type 1 (Ethernet A-D) route flavors on the CE5 segment: **per-ES** (mass withdrawal) and **per-EVI** (aliasing). Which config produces each?

**Solution:**
Type 1 is auto-generated once the Ethernet Segment (Task 1.3) and the EVI are up — no extra knobs:
- **Per-ES A-D route** (ESI, Ethernet Tag = MAX-ET `0xFFFFFFFF`): one route per ES. Its withdrawal is the **mass-withdrawal** signal — a single BGP update tells all remote PEs to stop using this PE for *every* MAC behind the ES.
- **Per-EVI A-D route** (ESI, real Ethernet Tag/EVI): one route per ES per EVI. Remote PEs use it for **aliasing** — load-balancing unicast to *all* PEs advertising the per-EVI A-D route for that ESI, even MACs only one PE has actually learned.

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 1
!   Per-ES A-D  : ESI ...0005, EthTag 0xFFFFFFFF  (mass withdrawal)
!   Per-EVI A-D : ESI ...0005, EthTag = EVI 100   (aliasing)
RP/0/0/CPU0:Gar-R2# show evpn ethernet-segment detail | i "AD|Alias|MAC Flush"
```

---

### Task 1.5 — Type 5 (IP Prefix Route)

**Question:** Configure Type 5 so a remote PE can reach a subnet behind CE5 by **IP prefix** (no MAC in the same subnet required). Explain the purpose of Type 5 vs Type 2.

**Solution:**
Type 5 advertises an **IP prefix** in an IP-VRF (L3), decoupled from any MAC — used for inter-subnet routing, summarization, connecting non-EVPN networks, and floating/silent hosts. Type 2 (MAC/IP) is host-route granular and tied to a bridge-domain; Type 5 is prefix granular and tied only to an IP-VRF. Configure an IP-VRF and enable Type-5 advertisement.

```
vrf TENANT-A
 address-family ipv4 unicast
  import  route-target 100:5
  export  route-target 100:5
!
router bgp 100
 vrf TENANT-A
  rd auto
  address-family ipv4 unicast
   ! prefixes here are exported into l2vpn evpn as Type 5
!
evpn
 evi 200
  bgp
   route-target import 100:200
   route-target export 100:200
 !
 ! Route-type 5 uses the IP-VRF's routes + an EVPN gateway/BVI (see Section 4)
```

**Verification:**
```
RP/0/0/CPU0:G-R2# show bgp l2vpn evpn route-type 5
!   -> IP prefix (e.g. 10.100.0.0/24 behind CE5) with GW-IP + VRF label
RP/0/0/CPU0:G-R2# show route vrf TENANT-A 10.100.0.0/24
!   -> installed via EVPN, next-hop 3.3.3.3/4.4.4.4
```

---

## Section 2 — EVPN-VPWS (3 tasks)

EVPN-VPWS is point-to-point (E-Line) using EVPN Type-1 A-D routes for signaling instead of targeted LDP. Uses `l2vpn xconnect` + `evpn evi ... vpws`.

### Task 2.1 — Point-to-point EVPN-VPWS (single-homed)

**Question:** Build a single-homed EVPN-VPWS between Gar-R1 (toward CE5's data VLAN) and G-R2 (toward CE7). Use EVI 500. Explain how the service is signaled.

**Solution:**
EVPN-VPWS binds a local AC to a **local AC-ID** and a **remote AC-ID** under an EVI. Each PE advertises a **Type-1 per-EVI A-D route** carrying its local AC-ID + MPLS label; the peer matches remote AC-ID → the pseudowire forms. No targeted LDP, no manual PW class — BGP EVPN does the signaling.

```
! ---- Gar-R1 ----
l2vpn
 xconnect group VPWS
  p2p CE5-CE7
   interface GigabitEthernet0/0/0/2.500
   neighbor evpn evi 500 target 65 source 63
!
! ---- G-R2 (mirror source/target) ----
l2vpn
 xconnect group VPWS
  p2p CE5-CE7
   interface GigabitEthernet0/0/0/2.500
   neighbor evpn evi 500 target 63 source 65
!
interface GigabitEthernet0/0/0/2.500 l2transport   ! both PEs
 encapsulation dot1q 500
 rewrite ingress tag pop 1 symmetric
```
> `source` = my local AC-ID, `target` = remote AC-ID. Gar-R1 source 63/target 65; G-R2 mirrors.

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show l2vpn xconnect group VPWS
!   -> state UP
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 1
!   -> per-EVI A-D for EVI 500 with AC-ID 63 (local) and 65 (remote)
RP/0/0/CPU0:Gar-R1# show evpn evi 500 detail
```

---

### Task 2.2 — EVPN-VPWS over SR-MPLS transport (Garnet)

**Question:** Ensure the EVI 500 pseudowire from Gar-R1 rides an **SR-MPLS** LSP (prefix-SID transport) rather than LDP. Optionally steer it into an SR-TE policy. Explain the transport-vs-service label stack.

**Solution:**
EVPN-VPWS is transport-agnostic: the **outer** (transport) label is the prefix-SID toward the remote PE loopback; the **inner** (service) label is the EVPN VPWS label from the Type-1 A-D route. Because Section-1 core already uses SR-MPLS (no LDP), the PW automatically uses SR transport — the BGP next-hop (G-R2 loopback) resolves over the SR label. To pin it to a specific path, bind the service to an SR-TE policy via color (ODN).

```
! Confirm no LDP; SR provides transport labels (already true from SR workbook).
! Optional ODN steering: color the EVPN next-hop toward an SR-TE policy.
extcommunity-set opaque COLOR-VPWS
 100
end-set
!
router bgp 100
 address-family l2vpn evpn
  ! route-policy attaching color 100 -> ODN builds SR-TE policy to G-R2
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show mpls forwarding prefix 6.6.6.6/32
!   -> outer label = SR prefix-SID for G-R2 (16006), NOT an LDP label
RP/0/0/CPU0:Gar-R1# show l2vpn xconnect group VPWS detail | i "MPLS|SR|label"
RP/0/0/CPU0:Gar-R1# show mpls ldp neighbor        ! empty -> transport is SR
```

---

### Task 2.3 — Compare with traditional (LDP/AToM) VPWS

**Question:** Contrast EVPN-VPWS with legacy AToM/L2VPN VPWS. What operational advantages justify EVPN-VPWS?

**Solution (comparison):**

| Aspect | Traditional VPWS (AToM) | EVPN-VPWS |
|--------|-------------------------|-----------|
| Signaling | Targeted LDP (T-LDP) per PW | BGP EVPN Type-1 A-D (uses existing iBGP) |
| PW config | Manual `neighbor <ip> pw-id` on both ends | AC-ID source/target under EVI; RR distributes |
| Transport | Usually LDP | SR-MPLS / SR-TE (no LDP), or LDP |
| Multi-homing | None (single PW) | All-active/single-active via ESI + Type-1 A-D |
| Redundancy convergence | Slow (per-PW re-signal) | Fast (mass withdrawal, one route) |
| Scale | N² targeted LDP sessions | One BGP AF, RR-reflected |

**Verification:**
```
! No T-LDP sessions for the EVPN-VPWS service:
RP/0/0/CPU0:Gar-R1# show mpls ldp neighbor      ! none for the PW
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn summary  ! service carried in BGP instead
```

---

## Section 3 — EVPN Multi-Homing (5 tasks)

CE5 dual-homed to Gar-R1+Gar-R2. Section 1.3 built the Ethernet Segment; here we exercise the redundancy modes and their control-plane mechanics.

### Task 3.1 — Ethernet Segment config on Gar-R1 + Gar-R2 for CE5

**Question:** Finalize a consistent Ethernet Segment for CE5 on both PEs, ensuring the ESI matches and the LACP bundle is shared.

**Solution:**
Both PEs must present **identical ESI** and **identical LACP system-id** so CE5 sees one logical LAG. Mismatched ESI = no ES peering = duplicate frames / broken DF election (see Task 6.3).

```
! ---- Gar-R1 AND Gar-R2 (identical) ----
interface Bundle-Ether5
 lacp system mac 0000.0000.0005
!
evpn
 interface Bundle-Ether5
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment interface Bundle-Ether5 detail
!   -> ESI matches, ES peers 3.3.3.3 + 4.4.4.4, redundancy state Up
RP/0/0/CPU0:Gar-R1# show bundle Bundle-Ether5     ! LACP up, single logical LAG
```

---

### Task 3.2 — All-active multi-homing

**Question:** Configure CE5's segment for **all-active** so both Gar-R1 and Gar-R2 forward unicast simultaneously. Explain how remote PEs load-balance to both.

**Solution:**
All-active is the IOS-XR default when the CE connects via a single LACP bundle spanning both PEs. Both PEs forward known-unicast; the **per-EVI Type-1 A-D route (aliasing)** lets remote PEs ECMP unicast to both 3.3.3.3 and 4.4.4.4, even for MACs learned by only one of them. Only the **DF** forwards BUM toward CE5 (split-horizon prevents echo).

```
evpn
 interface Bundle-Ether5
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05
   load-balancing-mode all-active      ! (default) both PEs forward
```

**Verification:**
```
RP/0/0/CPU0:G-R2# show bgp l2vpn evpn route-type 1
!   -> per-EVI A-D from BOTH 3.3.3.3 and 4.4.4.4 (aliasing set)
RP/0/0/CPU0:G-R2# show evpn evi 100 mac 0050.5600.0005 detail
!   -> two next-hops (ECMP) toward CE5
```

---

### Task 3.3 — Single-active multi-homing

**Question:** Reconfigure CE5's segment for **single-active** (only one PE forwards; the other is standby). When is this required?

**Solution:**
Single-active is needed when the CE cannot LAG across both PEs (independent links) or when a single forwarding path is desired. Only the DF forwards both BUM and unicast; the non-DF blocks. Failover relies on mass-withdrawal (Task 3.5). No aliasing (only one active path).

```
evpn
 interface Bundle-Ether5              ! or two separate physical interfaces
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05
   load-balancing-mode single-active
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment interface Bundle-Ether5 detail
!   -> Mode: single-active ; one PE Forwarding, other Backup/Blocked
RP/0/0/CPU0:G-R2# show evpn evi 100 mac 0050.5600.0005 detail
!   -> single next-hop (no aliasing)
```

---

### Task 3.4 — DF election (mod-based / default)

**Question:** Explain and verify the default (mod-based) DF election for the CE5 ES, per EVI. How do you make it deterministic?

**Solution:**
Default DF election = **modulo**: candidate PEs (from Type-4 routes) are ordered by IP; `DF = ordinal(EVI-VLAN) mod (number-of-PEs)`. This spreads DF duty across EVIs. To force a specific PE, use **preference-based** DF election with a `df-election` weight/preference (higher wins). A 3-second timer lets all ES peers be discovered before electing.

```
! Default = mod-based (no config).
! Deterministic / preference-based alternative:
evpn
 interface Bundle-Ether5
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05
   load-balancing-mode all-active
   ! preference DF election (Gar-R1 higher -> DF):
   ! (IOS-XR: 'service-carving preference-based' style)
   bgp route-target ...        ! ES-import auto
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment interface Bundle-Ether5 carving detail
!   -> per-EVI DF: e.g. EVI 100 -> DF = Gar-R2 (mod result), EVI 101 -> DF = Gar-R1
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment detail | i "DF|Elected|Carving"
```

---

### Task 3.5 — Mass withdrawal on PE failure (Type-1 per-ES route)

**Question:** Simulate Gar-R1's link to CE5 failing. Prove that a **single** Type-1 per-ES A-D withdrawal drains all MACs behind CE5 from remote PEs (fast convergence), rather than per-MAC Type-2 withdrawals.

**Solution:**
When the ES goes down on Gar-R1, Gar-R1 withdraws its **per-ES A-D route (EthTag 0xFFFFFFFF)**. Remote PEs treat this as "flush all MACs pointing at Gar-R1 for this ESI" in one operation — sub-second — then continue forwarding via Gar-R2. This is the mass-withdrawal mechanism (vs. waiting for thousands of individual Type-2 withdrawals).

```
! Trigger:
RP/0/0/CPU0:Gar-R1(config)# interface Bundle-Ether5
RP/0/0/CPU0:Gar-R1(config-if)# shutdown
```

**Verification:**
```
! On a remote PE, before/after:
RP/0/0/CPU0:G-R2# show bgp l2vpn evpn route-type 1 | i FFFFFFFF
!   -> Gar-R1's per-ES A-D disappears in one withdrawal
RP/0/0/CPU0:G-R2# show evpn evi 100 mac | i 3.3.3.3
!   -> all CE5 MACs formerly via Gar-R1 now via Gar-R2 only
! Continuous CE-to-CE ping: sub-second loss.
```

---

### Task 3.6 — Aliasing (Type-1 per-EVI route)

**Question:** Demonstrate **aliasing**: G-R2 load-balances unicast to CE5 across both Gar-R1 and Gar-R2 even for a MAC only Gar-R1 actually learned.

**Solution:**
Aliasing uses the **per-EVI Type-1 A-D route**. Because both Gar-R1 and Gar-R2 advertise per-EVI A-D for ESI ...0005/EVI 100, G-R2 knows the ES is reachable via both. When G-R2 receives a Type-2 for MAC X from only Gar-R1, it still installs *both* Gar-R1 and Gar-R2 as next-hops (aliasing) — improving load-balancing and resiliency. Requires all-active mode.

**Verification:**
```
RP/0/0/CPU0:G-R2# show evpn evi 100 mac 0050.5600.0005 detail
!   -> Paths: 3.3.3.3 (Type-2 owner) + 4.4.4.4 (aliased via per-EVI A-D)
RP/0/0/CPU0:G-R2# show cef vrf default ... | i "3.3.3.3|4.4.4.4"   ! ECMP
```

---

## Section 4 — EVPN IRB (3 tasks)

Integrated Routing and Bridging: a BVI bridges within a subnet and routes between subnets, using an IP-VRF for L3. Uses EVI (L2) + BVI + IP-VRF (L3).

### Task 4.1 — IRB with anycast gateway (L2 + L3 in one EVPN instance)

**Question:** Configure IRB on Gar-R1 and Gar-R2 for EVI 100 so hosts behind CE5 use a **distributed anycast gateway** (same GW IP+MAC on both PEs) and can be both bridged (same subnet) and routed (other subnets).

**Solution:**
A **BVI** ties the bridge-domain (L2, EVI 100) to the IP-VRF (L3, TENANT-A). Configure the **same** IP and the **same** virtual MAC on every PE — the anycast gateway — so a host always reaches its local PE regardless of where it moves. This puts L2 (bridging within VLAN 100) and L3 (routing out of the subnet) in a single EVPN instance.

```
! ---- Gar-R1 AND Gar-R2 (identical anycast GW) ----
vrf TENANT-A
 address-family ipv4 unicast
!
interface BVI100
 vrf TENANT-A
 ipv4 address 10.100.0.1 255.255.255.0        ! same on both PEs
 mac-address 0000.aaaa.0100                    ! anycast vMAC, same on both
!
l2vpn
 bridge group GARNET
  bridge-domain VLAN100
   interface Bundle-Ether5.100
   routed interface BVI100                      ! bind BVI to the BD
   evi 100
!
evpn
 evi 100
  advertise-mac
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn evi 100 mac
RP/0/0/CPU0:Gar-R1# show arp vrf TENANT-A
RP/0/0/CPU0:Gar-R1# show route vrf TENANT-A
! Host behind CE5 pings its gateway 10.100.0.1 (answered locally on DF/any PE).
```

---

### Task 4.2 — Symmetric IRB vs Asymmetric IRB

**Question:** Configure symmetric IRB (route in the ingress VRF, bridge across a common L3 label, route into egress VRF) and contrast with asymmetric IRB.

**Solution:**
- **Asymmetric IRB:** ingress PE does bridge→route→bridge; it must have **every** destination bridge-domain/EVI locally. Simpler, but every PE needs all VLANs → poor scale.
- **Symmetric IRB:** ingress PE routes into a common **L3 (IP-VRF) label** (Type-5 / MAC+IP with L3 label), transports over the fabric, egress PE routes into the destination VLAN. Each PE only needs the VLANs it locally serves + the shared IP-VRF → scales. This is the recommended model.

```
! Symmetric IRB: advertise an L3 (VRF) label; use Type-5 for prefixes.
router bgp 100
 vrf TENANT-A
  rd auto
  address-family ipv4 unicast
   redistribute connected           ! subnets -> Type-5
!
evpn
 evi 100
  advertise-mac                       ! Type-2 MAC+IP carries L2 + L3 label
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 5      ! symmetric: prefixes present
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 2 detail | i "L3 Label|Label2"
!   -> Type-2 carries both L2 and L3 labels (symmetric IRB)
RP/0/0/CPU0:Gar-R1# show route vrf TENANT-A                ! remote subnets via EVPN
```

---

### Task 4.3 — Verify L2 + L3 coexist in one EVPN instance

**Question:** Prove that within EVI 100 a host can be bridged to another host in VLAN 100 and simultaneously routed to a host in a different subnet, all via one EVPN instance.

**Solution:**
With the BVI bound into the bridge-domain and the IP-VRF attached, intra-VLAN traffic is bridged (Type-2 MAC) while inter-VLAN traffic hits the anycast BVI and is routed (Type-2 MAC+IP or Type-5). No separate instance needed.

**Verification:**
```
! Intra-subnet (bridged):
RP/0/0/CPU0:Gar-R1# show evpn evi 100 mac                 ! peer MAC learned via Type-2
! Inter-subnet (routed):
RP/0/0/CPU0:Gar-R1# show route vrf TENANT-A 10.200.0.0/24 ! via BVI/EVPN L3
! CE5 host -> host in VLAN100 (bridge) AND host in 10.200.0.0/24 (route) both succeed.
```

---

## Section 5 — EVPN Inter-AS (3 tasks)

Extend EVPN from **Garnet** (CE5) to **Gold** (CE7) across the AS boundary. BGP EVPN runs between the Garnet RR (Gar-R6) and the Gold border (G-R4).

### Task 5.1 — BGP EVPN session between RRs / borders (Gar-R6 ↔ G-R4)

**Question:** Establish an inter-AS `l2vpn evpn` session so EVPN routes cross from Garnet into Gold. Use Gar-R6 (Garnet RR) ↔ G-R4.

**Solution:**
Bring up **multihop eBGP** with `l2vpn evpn` between Gar-R6 (AS 100) and G-R4 (AS 200). Keep next-hop reachability by either next-hop-unchanged (Option-B style, G-R4 rewrites NH and swaps labels) or a redistribution of loopbacks. RTs must line up so Gold imports Garnet's EVI 100.

```
! ---- Gar-R6 (AS 100) ----
router bgp 100
 neighbor 13.13.13.13
  remote-as 200
  ebgp-multihop 5
  update-source Loopback0
  address-family l2vpn evpn
   next-hop-unchanged            ! Option-B: preserve originator NH
!
! ---- G-R4 (AS 200) ----
router bgp 200
 neighbor 9.9.9.9
  remote-as 100
  ebgp-multihop 5
  update-source Loopback0
  address-family l2vpn evpn
   next-hop-unchanged
```

**Verification:**
```
RP/0/0/CPU0:Gar-R6# show bgp l2vpn evpn summary
!   -> neighbor 13.13.13.13 (AS 200) Established, prefixes exchanged
RP/0/0/CPU0:G-R4# show bgp l2vpn evpn summary
```

---

### Task 5.2 — Type-2 MAC/IP exchange across domains

**Question:** Ensure CE7's MAC (Gold, G-R2) is learned in Garnet and CE5's MAC is learned in Gold, via Type-2 across the AS boundary. Align RTs.

**Solution:**
For the same broadcast domain to span both ASes, EVI 100 on G-R2 must import/export the **same RT** (or a border route-policy translates RTs). Once the Gar-R6↔G-R4 session carries Type-2, CE5's and CE7's MACs propagate both ways.

```
! ---- G-R2 (Gold) EVI 100 : matching RTs ----
evpn
 evi 100
  bgp
   route-target import 100:100
   route-target export 100:100
!
! G-R4 may re-originate/translate RT if AS policies differ:
!   route-policy EVPN-RT-XLATE (set extcommunity rt ... )
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 2 | i <CE7-MAC>   ! CE7 MAC in Garnet
RP/0/0/CPU0:G-R2# show bgp l2vpn evpn route-type 2 | i <CE5-MAC>   ! CE5 MAC in Gold
RP/0/0/CPU0:G-R2# show evpn evi 100 mac                            ! remote MACs present
```

---

### Task 5.3 — CE5 (Garnet) ↔ CE7 (Gold) end-to-end EVPN

**Question:** Validate CE5 and CE7 communicate at Layer 2 (same VLAN 100 stretched) and/or Layer 3 across the AS boundary.

**Solution:**
With RTs aligned and Type-2/Type-3 crossing the boundary, VLAN 100 is stretched Garnet↔Gold. BUM uses each domain's ingress replication; the ASBR forwards between domains. Transport is SR-MPLS within each AS and Option-B label swap at G-R4.

**Verification:**
```
! From CE5 host to CE7 host (same subnet -> L2, else routed via IRB):
CE5# ping <CE7-host-ip>
RP/0/0/CPU0:Gar-R1# show evpn evi 100 inclusive-multicast    ! includes Gold PE via ASBR
RP/0/0/CPU0:G-R4# show mpls forwarding                    ! label swap for EVPN
```

---

## Section 6 — MAC Mobility + Troubleshooting (3 tasks)

### Task 6.1 — MAC mobility (sequence number) + sticky MAC

**Question:** A host with MAC `0050.5600.00AA` moves from behind CE5 (Gar-R1) to behind G-R2. Show how EVPN converges via the MAC Mobility extended community (sequence number), and how to pin a MAC with **sticky/static** MAC.

**Solution:**
When a MAC re-appears behind a new PE, that PE advertises a Type-2 with an **incremented sequence number** in the MAC Mobility extended community. Higher sequence wins; the old PE withdraws its Type-2. To prevent a MAC from moving (e.g., a gateway), configure a **static/sticky MAC** — it is advertised with the sticky bit and never relearned elsewhere; any move attempt is rejected.

```
! Sticky/static MAC pinned to a bridge-domain port (Gar-R1):
l2vpn
 bridge group GARNET
  bridge-domain VLAN100
   interface Bundle-Ether5.100
    mac
     static-address 0050.5600.00aa            ! sticky -> never moves
```

**Verification:**
```
RP/0/0/CPU0:G-R2# show bgp l2vpn evpn route-type 2 | i "0050.5600.00aa|Seq"
!   -> MAC Mobility seq number increments after a legitimate move
RP/0/0/CPU0:Gar-R1# show evpn evi 100 mac 0050.5600.00aa detail
!   -> for sticky MAC: flagged Static/Sticky, move rejected
```

---

### Task 6.2 — Duplicate MAC detection

**Question:** A MAC flaps rapidly between Gar-R1 and G-R2 (misconfig / loop). Show EVPN's duplicate-MAC detection and how to tune/clear it.

**Solution:**
EVPN counts MAC moves within a window (IOS-XR default: **5 moves / 180 s**). Exceeding it marks the MAC **duplicate** and **freezes** it (stops re-advertising) to protect the control plane until the flap stops or it is cleared. Tune with `mac secure`/mobility knobs; clear manually after fixing the loop.

```
evpn
 evi 100
  ! adjust move count / window (illustrative):
  ! mac mobility  <moves> <window-seconds>
!
! Clear a frozen duplicate MAC after remediation:
RP/0/0/CPU0:Gar-R1# clear evpn evi 100 mac 0050.5600.00aa
```

**Verification:**
```
RP/0/0/CPU0:Gar-R1# show evpn evi 100 mac duplicate
!   -> MAC listed as Duplicate/Frozen with move count
RP/0/0/CPU0:Gar-R1# show logging | i "DUPLICATE|MAC move"
```

---

### Task 6.3 — Troubleshooting scenarios

**Question:** Diagnose and fix two classic EVPN faults:
(a) **EVPN routes not received** on a PE (nothing in `show bgp l2vpn evpn`).
(b) **DF election is wrong / duplicate BUM** to CE5.

**Solution:**

**(a) Missing `l2vpn evpn` address-family.**
Symptom: MAC-VRF configured, ACs up, but `show bgp l2vpn evpn` empty and remote MACs never learned. Root cause: the `address-family l2vpn evpn` was never activated globally and/or per-neighbor toward the RR. Fix:
```
router bgp 100
 address-family l2vpn evpn           ! global AF was missing
 neighbor 9.9.9.9
  address-family l2vpn evpn          ! per-neighbor activation was missing
```
Verify:
```
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn summary        ! neighbor now Established + AF up
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn                 ! routes now present
```

**(b) DF election wrong due to ESI mismatch.**
Symptom: CE5 receives duplicate BUM, or both PEs think they are DF, or ES peers don't form. Root cause: Gar-R1 and Gar-R2 have **different ESI** (or different LACP system-id), so they never see each other's Type-4 → no shared segment → both forward. Fix — make the ESI identical:
```
! On the PE with the wrong value:
evpn
 interface Bundle-Ether5
  ethernet-segment
   identifier type 0 00.00.00.00.00.00.00.00.05   ! must match the peer exactly
!
interface Bundle-Ether5
 lacp system mac 0000.0000.0005                    ! same LACP system-id on both
```
Verify:
```
RP/0/0/CPU0:Gar-R1# show evpn ethernet-segment interface Bundle-Ether5 detail
!   -> ES peers now list BOTH 3.3.3.3 and 4.4.4.4, single DF elected per EVI
RP/0/0/CPU0:Gar-R1# show bgp l2vpn evpn route-type 4    ! matching ESI from both PEs
```

---

## Final Validation Checklist

```
Section 1 — Route Types
[ ] Type 2  MAC/IP advertised & learned via BGP (no flooding)      show bgp l2vpn evpn route-type 2
[ ] Type 3  IMET / ingress replication per PE                      show bgp l2vpn evpn route-type 3
[ ] Type 4  ES route + DF election (CE5 dual-homed)                show evpn ethernet-segment
[ ] Type 1  per-ES (mass withdrawal) + per-EVI (aliasing)          show bgp l2vpn evpn route-type 1
[ ] Type 5  IP prefix route in IP-VRF                              show bgp l2vpn evpn route-type 5

Section 2 — EVPN-VPWS
[ ] Single-homed EVPN-VPWS (EVI 500) UP via Type-1 A-D             show l2vpn xconnect
[ ] EVPN-VPWS over SR-MPLS transport (no LDP)                      show mpls forwarding / ldp neighbor
[ ] Documented comparison vs traditional AToM VPWS

Section 3 — Multi-Homing
[ ] Consistent ESI/LACP on Gar-R1+Gar-R2 for CE5                         show evpn ethernet-segment detail
[ ] All-active forwarding + aliasing (ECMP to both PEs)            show evpn evi 100 mac detail
[ ] Single-active (one forwarder, one standby)
[ ] DF election mod-based (deterministic option noted)            show evpn ethernet-segment carving
[ ] Mass withdrawal on PE failure (single Type-1 per-ES)          show bgp l2vpn evpn route-type 1
[ ] Aliasing via per-EVI Type-1 A-D

Section 4 — IRB
[ ] Anycast gateway BVI (same IP+MAC on both PEs)                  show arp vrf / show route vrf
[ ] Symmetric vs asymmetric IRB documented + L3 label present     show bgp l2vpn evpn route-type 2 detail
[ ] L2 + L3 coexist in one EVPN instance

Section 5 — Inter-AS
[ ] Gar-R6 ↔ G-R4 l2vpn evpn session Established                    show bgp l2vpn evpn summary
[ ] Type-2 MAC/IP exchanged across AS boundary (RTs aligned)
[ ] CE5 (Garnet) ↔ CE7 (Gold) reachable

Section 6 — Mobility + Troubleshooting
[ ] MAC mobility sequence increments on move; sticky MAC pinned    show bgp l2vpn evpn route-type 2
[ ] Duplicate MAC detected & frozen; cleared after fix             show evpn evi 100 mac duplicate
[ ] Fixed: missing l2vpn evpn AF (routes not received)
[ ] Fixed: ESI mismatch (wrong DF election)
```

---

## Quick Reference — key IOS-XR EVPN show commands

```
show bgp l2vpn evpn summary                     ! AF neighbor state
show bgp l2vpn evpn                             ! all EVPN NLRI
show bgp l2vpn evpn route-type {1|2|3|4|5}      ! per route type
show evpn evi                                   ! list EVIs
show evpn evi 100 mac [detail]                  ! MAC table (L2)
show evpn evi 100 inclusive-multicast detail    ! BUM replication list
show evpn ethernet-segment [interface X] detail ! ES peers, redundancy
show evpn ethernet-segment carving detail       ! per-EVI DF result
show evpn evi 100 mac duplicate                 ! duplicate-MAC state
show l2vpn xconnect                             ! EVPN-VPWS state
show l2vpn bridge-domain [detail]               ! BD status
clear evpn evi 100 mac <mac>                    ! clear/unfreeze a MAC
```
