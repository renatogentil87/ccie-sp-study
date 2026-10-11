# CCIE SP Workbook 14 — High Availability & Fast Convergence

**Platform:** Cisco IOS-XRv 9000 — EVE-NG
🔴 **CCIE Prep Platform:** EVE-NG (IOS-XRv 9000) — see `00_EVENG_Topology.md` for the Emerald+Gold+Garnet topology
**Topology:** All 3 SPs — Emerald (AS 65100) + Gold (AS 65300) + Garnet (AS 65200). Focus on convergence across the **full path** (CE → PE → P → ASBR → P → PE → CE), so a single failure is measured end-to-end, not just locally.
**Initial configs:** IS-IS L2 backbone with wide metrics + /32 loopbacks (WB01), LDP or SR label plane (WB02/WB09), MP-BGP VPNv4 with RRs (WB03/WB11), MPLS-TE where noted (WB08). Dual-RP XRv images (or simulated RP redundancy) for the Platform HA section.
**Blueprint mapping:** Domain 4 — Services & Convergence / High Availability (~10%).

> **The convergence chain.** After any failure, packet loss ends only when the *slowest* layer reconverges. The layers stack: (1) **failure detection** — physical/BFD; (2) **local repair** — TI-LFA or RSVP-FRR pushes traffic onto a precomputed backup in <50 ms; (3) **IGP reconvergence** — SPF + LSP/LSA flooding; (4) **label plane** — LDP resync or SR (instant, IGP-driven); (5) **BGP reconvergence** — PIC swaps a FIB pointer instead of rescanning the RIB. HA on the platform (NSF/NSR/SSO) keeps the *control plane alive across an RP failure* so none of this is triggered at all. This workbook walks every layer in that order.
>
> **Format:** each task is **Question → Solution → Verification**. All syntax is IOS-XR.

---

## Section 1 — Platform High Availability (NSF / NSR / GR / SSO)

### Task 1.1 — NSF, NSR, and Graceful Restart: configure and distinguish

**Question**
On E-R1 (dual-RP), enable **NSR** for IS-IS, OSPF, LDP, and BGP so that an RP switchover is invisible to neighbors *without* relying on them. Then, for the case where NSR is not available on a protocol/peer, enable **Graceful Restart (NSF)**. Explain precisely how NSF, NSR, GR, and SSO relate.

**Solution**

SSO (Stateful Switchover) is the platform mechanism: the standby RP mirrors the active RP's state (interfaces, ARP, CEF/FIB, line-card state) so that when the active RP fails, the standby takes over with the **data plane never interrupted** — the FIB in the line cards keeps forwarding. SSO is the prerequisite for everything else in this section.

On top of SSO you choose how the *routing control plane* survives:

- **NSF / Graceful Restart (GR)** — the restarting router asks its **neighbors** to keep the adjacency and hold the routes ("I'm restarting, don't withdraw me") while it rebuilds its RIB. It is **helper-dependent**: neighbors must support GR and act as helpers. The router's own protocol state is *not* preserved — it is relearned. This is the older, interoperable mechanism.
- **NSR (Non-Stop Routing)** — the router keeps **full protocol state synced to the standby RP** (adjacencies, LSDB, BGP sessions/TCP state). On switchover the standby continues the sessions with **no signaling to neighbors at all** — neighbors never know a switchover happened. It is **self-contained** (no helper needed), which is why it is preferred in multi-vendor cores where you cannot guarantee GR helper support.

```
router isis CORE
 nsf ietf                 ! GR (IETF) — used only if NSR unavailable / for interop
 nsr                      ! Non-Stop Routing — preferred; no neighbor dependency
!
router ospf 1
 nsr
!
mpls ldp
 nsr                      ! LDP NSR — keeps label bindings across switchover
 graceful-restart         ! GR fallback if peer doesn't support NSR
!
router bgp 65100
 nsr                      ! BGP NSR — TCP + session state synced to standby RP
 ! GR alternative if NSR not desired for a neighbor:
 ! neighbor 10.0.0.11
 !  graceful-restart
```

Rule of thumb for the lab: **prefer NSR** (self-contained, no neighbor requirement). Use **GR/NSF** only where NSR isn't supported or where you must interoperate with a device that expects to be a helper.

**Verification**
- `show redundancy` — confirm SSO is `Standby ready` / RPs in `hot-standby`.
- `show isis nsr` / `show ospf nsr` / `show bgp nsr` — NSR state = `Ready` on standby.
- `show mpls ldp nsr summary` — LDP NSR sync complete.
- `show isis graceful-restart` / `show bgp neighbor <x> | i Graceful` — GR capability negotiated where used.

### Task 1.2 — Trigger an RP switchover and prove data-plane continuity

**Question**
Start continuous CE-to-CE traffic through E-R1. Force an RP switchover (`redundancy switchover`). Prove there is **zero (or near-zero) traffic loss** and that neighbors did **not** reset the adjacency.

**Solution**

Because SSO keeps the FIB alive in the line cards and NSR keeps the routing sessions on the standby RP, the switchover is a control-plane event with **no data-plane impact**: the line cards forward from the already-synced CEF entries the entire time, and NSR continues the IS-IS/OSPF/LDP/BGP sessions from the standby without any neighbor-visible flap.

```
RP/0/RP0/CPU0:E-R1# redundancy switchover
```

If only GR (not NSR) were configured, neighbors would enter *helper* mode and hold the routes for the grace period while E-R1 relearns — still no data loss, but it depends on the neighbors behaving as helpers.

**Verification**
- Traffic generator: 0–1 packet lost across the switchover.
- `show redundancy` — roles swapped (former standby now `Active`), new standby re-syncing.
- On a neighbor: `show isis neighbors` / `show bgp neighbor` — adjacency/session **Up**, uptime **not** reset (NSR) — proof it never flapped.
- `show logging` — no adjacency-down / session-reset events for the switchover window.

### Task 1.3 — When NSF/NSR is the *wrong* answer

**Question**
Explain a scenario where enabling GR/NSF actually **slows** or **harms** convergence, and state the design rule.

**Solution**

GR is a **double-edged sword**: while a neighbor is in helper mode it deliberately **keeps forwarding to a router that may be genuinely down**, delaying real reconvergence for the grace timer. If the "restart" was actually a hard failure (not a graceful RP switchover), GR **black-holes** traffic until the timer expires. Therefore:

- Use **NSF/NSR to survive planned/graceful RP events** on a box you control.
- Do **not** rely on GR across a link where fast failure detection matters — pair convergence with **BFD** and **TI-LFA** so a *real* failure is repaired in <50 ms regardless of any GR helper timer. HA (surviving RP failure) and fast convergence (surviving link/node failure) are **orthogonal** goals; you need both, applied to the right failure type.

**Verification**
- `show bgp neighbor <x> | i Graceful` — note the negotiated restart/stalepath timers; explain the black-hole window they create if the peer truly died.
- Contrast with `show bfd session` / `show isis fast-reroute summary` from Sections 2 & 5 which handle the *link* failure case in sub-second/50 ms.

---

## Section 2 — IGP Convergence (IS-IS & OSPF tuning + BFD)

### Task 2.1 — IS-IS SPF and LSP-generation throttling

**Question**
Tune IS-IS on the Emerald core so the **first** SPF after a change runs almost immediately, but a flapping link **backs off** to protect CPU. Do the same for LSP generation. Explain the three exponential-backoff values.

**Solution**

IOS-XR uses **exponential backoff** for both SPF and LSP generation, expressed as three values: `initial-wait max-wait secondary-wait` (all in ms). `initial-wait` = delay before the *first* run (keep small for fast convergence). `secondary-wait` = the increment added on each *successive* event within the window. `max-wait` = the ceiling. So a single event converges fast; a storm of events backs off exponentially toward `max-wait`, damping CPU churn.

```
router isis CORE
 lsp-gen-interval maximum-wait 5000 initial-wait 5 secondary-wait 50
 spf-interval     maximum-wait 5000 initial-wait 5 secondary-wait 50
```

`initial-wait 5` → first SPF ~5 ms after the event (near-instant). Repeated flaps grow 50 → 100 → 200 … up to 5000 ms.

**Verification**
- `show isis` — confirm the configured SPF / LSP-gen intervals.
- `show isis spf-log` — inspect trigger times and observe the backoff growing under repeated flaps.
- `show isis database` — LSP sequence numbers incrementing as expected.

### Task 2.2 — PRC vs full SPF, and IS-IS fast-flood

**Question**
Explain why a **leaf prefix** change (e.g., a redistributed /32) does **not** require a full SPF, and enable **fast-flood** so the failing router floods its new LSP *before* running SPF.

**Solution**

**Full SPF** recomputes the entire shortest-path tree and is only needed when the **topology** (a link/node, i.e., a Router-LSP's adjacency TLV) changes. When only a **leaf/prefix** attached to an unchanged topology changes (IP reachability TLV), IS-IS runs **PRC (Partial Route Calculation)** — it updates just the affected prefixes against the *existing* tree. PRC is far cheaper and much faster, which is why prefix churn (BGP-into-IGP redistribution, loopback add/remove) does not stall the core.

**Fast-flood** tells the router to **flood the changed LSP to neighbors first**, before spending CPU on its own SPF — so downstream routers start converging in parallel instead of serially waiting for each hop to compute then flood.

```
router isis CORE
 lsp-fast-flood threshold 15     ! flood up to 15 LSPs before running SPF
```

**Verification**
- `show isis spf-log` — entries tagged **PRC** vs **FSPF (full)**; a prefix-only change shows PRC.
- Change a topology link → next log entry is a full SPF; change only a leaf prefix → PRC.
- `show isis spf-log detail` — confirm flood-before-compute behavior (LSP flooded ahead of SPF run).

### Task 2.3 — OSPF SPF throttling (Gold ISP)

**Question**
Gold runs OSPF. Apply the equivalent **SPF throttle** so first SPF is fast and repeated events back off.

**Solution**

OSPF in IOS-XR uses `timers throttle spf start hold maximum` (ms): `start` = delay before the first SPF, `hold` = interval that doubles on each successive SPF, `maximum` = ceiling. Same exponential-backoff philosophy as IS-IS, different keywords. Add LSA throttling to damp LSA origination the same way.

```
router ospf 1
 timers throttle spf 5 50 5000       ! start 5ms, hold 50ms (doubles), max 5000ms
 timers throttle lsa all 5 50 5000   ! LSA generation backoff
 timers lsa min-arrival 50
```

**Verification**
- `show ospf` — confirm SPF/LSA throttle timers.
- `show ospf statistics` — SPF run count and last-run intervals; observe backoff under flaps.

### Task 2.4 — BFD for sub-second detection on **all** IGP adjacencies

**Question**
IGP hold timers detect failure in *seconds*. Enable **BFD** so every IGP adjacency (IS-IS on Emerald/Garnet, OSPF on Gold) detects failure in **sub-second** and immediately triggers convergence / TI-LFA.

**Solution**

BFD is a lightweight hello protocol run in hardware/fast-path that detects link failure in **milliseconds** (e.g., 3 × 50 ms = 150 ms), far faster than IGP hellos. The IGP registers as a **BFD client**: when BFD declares the session down, it instantly tears the adjacency and triggers SPF + the precomputed TI-LFA repair — this is the essential first link in the sub-second chain (detection). Without BFD you are stuck with the IGP hold-time (multiple seconds), which is the #1 cause of "convergence > 1 s" (see Section 6).

```
! IS-IS (Emerald / Garnet)
router isis CORE
 interface GigabitEthernet0/0/0/0
  bfd minimum-interval 50
  bfd multiplier 3
  address-family ipv4 unicast
  !
 !
! OSPF (Gold)
router ospf 1
 area 0
  interface GigabitEthernet0/0/0/1
   bfd minimum-interval 50
   bfd multiplier 3
```

**Verification**
- `show bfd session` — every IGP-facing link: state **UP**, 50 ms interval, mult 3.
- `show bfd session detail` — the IGP listed as a registered **client**.
- Fail a link and time convergence: detection now ~150 ms (was seconds). Correlate with `show isis spf-log` timestamp immediately following the BFD-down.

---

## Section 3 — LDP / MPLS Convergence

### Task 3.1 — LDP-IGP synchronization

**Question**
After a link comes up (or LDP restarts), the IGP may start using it **before LDP has exchanged labels**, causing a **blackhole** (IP forwards, but no label → dropped LSP). Configure **LDP-IGP sync** so the IGP does not use the link until LDP is ready.

**Solution**

LDP-IGP sync makes the IGP advertise the link with **max-metric** (effectively unusable for transit) until the LDP session on that link reaches convergence and label bindings are exchanged. Once LDP is up, the real metric is restored and traffic moves onto the link — eliminating the transient blackhole where the IGP path exists but the LSP does not.

```
mpls ldp
 igp sync delay 10        ! optional: hold up to 10s for LDP after IGP up
!
router isis CORE
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   mpls ldp sync
```

**Verification**
- `show mpls ldp igp sync` — per-interface `Sync status: Ready` once LDP converged; `Not ready` (max-metric) while waiting.
- `show isis interface Gi0/0/0/0` — metric shows the max value during the sync-pending window, then reverts.
- Bring the link up and confirm no packet loss / no blackhole during the LDP-convergence window.

### Task 3.2 — LDP session protection

**Question**
A **link flap** normally tears down the directly-connected LDP session and forces relearning all label bindings. Enable **LDP session protection** so the session survives via a **targeted** session over an alternate path.

**Solution**

Session protection establishes a **targeted LDP (tLDP) session** in parallel with the link (directly-connected) session. If the link flaps, the directly-connected session drops but the **targeted session stays up over the remaining IGP path**, so the label bindings are **retained** — no relearning, dramatically faster LDP reconvergence when the link returns.

```
mpls ldp
 session protection duration 60    ! keep tLDP alive 60s after link loss
```

**Verification**
- `show mpls ldp neighbor detail` — neighbor shows both `Link` and `Targeted` (`session protection: enabled`).
- Flap the link → `show mpls ldp bindings` confirms bindings are **retained** (not relearned).
- `show mpls ldp discovery` — targeted hello adjacency present.

### Task 3.3 — LDP backoff, and why SR needs none of this

**Question**
Under repeated LDP setup failures (e.g., a misconfigured/label-conflicting neighbor), tune **LDP exponential backoff** to avoid hammering the peer. Then explain why **Segment Routing** makes this entire section unnecessary.

**Solution**

LDP backoff sets `initial` and `maximum` delays between successive LDP session-setup attempts after a failure; the delay grows exponentially from `initial` toward `maximum`, protecting CPU from a session that keeps failing.

```
mpls ldp
 backoff 15 120       ! initial 15s, max 120s between failed setup retries
```

**SR advantage:** Segment Routing distributes labels **inside the IGP** as prefix-SIDs — there is **no separate label-distribution protocol**. So there is *no LDP session to protect, no IGP-sync blackhole window, no backoff, no targeted session, no LDP reconvergence at all*. When the IGP converges, the label plane has **already** converged, because they are the same protocol. This is the single biggest convergence and operational win of SR over LDP (see WB09).

**Verification**
- `show mpls ldp parameters` — confirm backoff `initial`/`maximum` values.
- `show mpls ldp discovery` — observe increasing retry intervals against a failing peer.
- **SR contrast:** on Garnet, `show mpls ldp neighbor` is **empty**; `show isis segment-routing label table` shows the label plane converging with the IGP — no LDP step exists.

---

## Section 4 — BGP Convergence (PIC, Add-Path, best-external)

### Task 4.1 — BGP PIC Edge (prefix-independent convergence)

**Question**
On Gar-R1 (Garnet), a VPNv4/eBGP prefix has a primary and a backup path. Configure **BGP PIC Edge** so that on primary-path failure, convergence is a **single FIB pointer swap** independent of the number of prefixes.

**Solution**

Without PIC, when the primary next-hop fails, BGP must **re-run bestpath per prefix and reprogram the FIB per prefix** — convergence time scales with the number of prefixes (could be seconds for a full table). **PIC Edge** pre-installs the **backup path in the FIB** as a secondary next-hop under a shared **path-list**. On failure, the router flips **one pointer** (primary → backup) in that shared structure, and **all** prefixes sharing it converge at once — **prefix-independent**, sub-second, regardless of table size. PIC Edge protects against **egress PE / external-link** failure and requires path diversity (Add-Path or best-external so the backup is actually known).

```
router bgp 65200
 address-family vpnv4 unicast
  additional-paths receive
  additional-paths selection route-policy PIC-ADD-PATHS
 !
 address-family ipv4 unicast
  additional-paths selection route-policy PIC-ADD-PATHS
!
route-policy PIC-ADD-PATHS
  set path-selection backup 1 install
end-policy
```

**Verification**
- `show cef <prefix> detail` — shows **primary + repair/backup** next-hop pre-installed, and a shared **path-list**.
- `show bgp <prefix>` — backup path present and marked as backup.
- Fail the primary path: traffic loss <1 s and **independent of prefix count** — the key PIC property.

### Task 4.2 — BGP PIC Core (IGP triggers BGP convergence)

**Question**
The failure is **inside the core** (a P-node or core link on the path to the BGP next-hop), not at the edge. Configure so **IGP convergence directly drives BGP** without a BGP walk.

**Solution**

**PIC Core** handles failures **between the ingress PE and the BGP next-hop (egress PE)**. The BGP next-hop is resolved recursively through the IGP. With PIC Core, when the **IGP reconverges** (new path to the same egress PE loopback), the FIB's recursive resolution updates automatically — BGP does **not** need to re-run bestpath at all, because the next-hop (egress PE) is **unchanged**; only the *path to it* changed. This is enabled by default in XR when the IGP provides a backup (TI-LFA), but the concept to state clearly: **PIC Edge = next-hop changes (edge/link failure); PIC Core = next-hop stays, IGP path to it changes (core failure).**

```
router bgp 65200
 address-family ipv4 unicast
  ! recursive next-hop resolution + IGP TI-LFA backup drive PIC Core
  nexthop trigger-delay critical 0 non-critical 0
```

**Verification**
- `show cef <egress-PE-loopback> detail` — primary + IGP backup (TI-LFA) next-hops for the *next-hop* prefix.
- Fail a **core** link: IGP/TI-LFA repairs in <50 ms; `show bgp <prefix>` bestpath is **unchanged** (next-hop same) — BGP did no work, proving PIC Core.
- Compare log timestamps: IGP reconverges, BGP bestpath count does **not** increment.

### Task 4.3 — BGP Add-Path for RR path diversity

**Question**
A Route Reflector by default advertises **only its single best path**, so clients never learn a backup — defeating PIC. Enable **Add-Path** on the RR so clients receive multiple paths.

**Solution**

A classic RR **hides diversity**: it reflects only *its* bestpath, so a client PE sees one path and has no backup to pre-install for PIC. **Add-Path** lets the RR advertise **N additional paths** (each with a unique path-id) for the same prefix, so clients learn primary **and** backup — which is exactly what PIC Edge needs to pre-program a FIB backup.

```
router bgp 65200
 address-family vpnv4 unicast
  additional-paths send
  additional-paths receive
  additional-paths selection route-policy ADD-PATH-2
!
route-policy ADD-PATH-2
  set path-selection all advertise         ! or 'backup 1 advertise'
end-policy
```

**Verification**
- `show bgp vpnv4 unicast <prefix>` on a client — **multiple** paths with distinct path-ids (was one before Add-Path).
- `show bgp neighbor <client> | i Additional` — Add-Path send/receive negotiated.
- Confirm the extra path enables PIC in Task 4.1 (`show cef` now has a backup).

### Task 4.4 — BGP best-external

**Question**
On a PE that has both an **external (eBGP)** path and an **internal (iBGP via RR)** path where the **iBGP** path wins bestpath, the PE normally **stops advertising** its external path to the core (only bestpath is advertised). Enable **best-external** so the PE still advertises its external path into iBGP.

**Solution**

Normally BGP advertises only the *bestpath*. If a PE's iBGP path wins, its own **external** path is suppressed and the rest of the core never learns that alternate egress exists — again killing diversity/PIC. **best-external** makes the PE advertise its **best external path** to iBGP peers **even when an iBGP path is the overall best**. This injects the alternate egress into the core so other PEs (and RRs with Add-Path) have a real backup to install for PIC and for faster reconvergence on egress-PE failure.

```
router bgp 65200
 address-family vpnv4 unicast
  advertise best-external
```

**Verification**
- Before: `show bgp vpnv4 unicast <prefix>` on the core shows only one (iBGP) egress. After: the external path from this PE is also present.
- `show bgp vpnv4 unicast <prefix> detail` — the external path flagged as **best-external / backup**.
- Combined with Add-Path (4.3) + PIC (4.1): fail the winning egress PE and confirm sub-second, prefix-independent failover to the best-external path.

---

## Section 5 — MPLS-TE FRR vs TI-LFA

### Task 5.1 — RSVP-TE Fast Reroute (link/node protection)

**Question**
On the Emerald core between E-R3 and E-R4, build a primary RSVP-TE tunnel and a **pre-signaled backup tunnel**; enable **FRR** so a protected link failure switches to the backup in <50 ms.

**Solution**

RSVP-TE FRR pre-signals a **backup LSP** around the protected link (link protection) or node (node protection). The PLR (point of local repair) pre-installs the backup label stack; on failure it locally splices traffic onto the backup in <50 ms while the head-end re-optimizes. The cost: **per-tunnel RSVP state on every transit router** (Path/Resv soft-state refreshed periodically), and you must explicitly signal a backup for each protected facility — this **does not scale** to large meshes and is operationally heavy.

```
! Primary tunnel (head-end E-R3)
interface tunnel-te1
 ipv4 unnumbered Loopback0
 destination 10.0.0.2
 path-option 1 dynamic
 fast-reroute                      ! request FRR protection
!
! Backup tunnel on the PLR protecting the E-R3-P2 link
interface tunnel-te10
 ipv4 unnumbered Loopback0
 destination 10.0.0.2
 path-option 1 explicit name AVOID-P1P2-LINK
!
rsvp
 interface GigabitEthernet0/0/0/0
  bandwidth
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/0
  backup-path tunnel-te10          ! bind backup to the protected interface
```

**Verification**
- `show mpls traffic-eng tunnels` — primary UP, `FRR: Ready`, backup bound.
- `show mpls traffic-eng fast-reroute database` — protected LSP → backup mapping installed.
- `show rsvp session` — **per-tunnel RSVP state** on each transit hop (the scaling cost to note).
- Fail the E-R3-P2 link: 0–1 packet lost; `FRR: Active` briefly, then head-end re-optimizes.

### Task 5.2 — TI-LFA (per-prefix, zero transit state)

**Question**
On the same failure scenario, use **TI-LFA** (SR/IGP-computed) instead of RSVP-FRR. Contrast the state model.

**Solution**

TI-LFA precomputes, **per prefix**, a **post-convergence** backup expressed as a **segment list**, and pre-installs it in the FIB. On failure the PLR switches in <50 ms — same speed as RSVP-FRR — but:

- **No signaling, no tunnels, no per-transit state** — the backup is just a label stack the PLR pushes; transit routers are stateless.
- **Every prefix is protected automatically** (per-prefix), not just facilities you remembered to signal.
- **Topology-independent** — it always finds a loop-free repair (using the P-space/Q-space segment list), even where classic LFA/rLFA could not.

```
router isis CORE
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
   ! optional node protection:
   ! fast-reroute per-prefix ti-lfa node-protection
```

**Verification**
- `show isis fast-reroute summary` — % of prefixes protected (aim ~100%).
- `show isis fast-reroute <prefix> detail` — repair path with **segment list**.
- `show cef <prefix> detail` — primary + backup next-hop pre-installed.
- `show rsvp session` — **empty** on transit routers: the key contrast — zero transit state.

### Task 5.3 — Same failure, measure and compare

**Question**
Run **identical** continuous CE-to-CE traffic through the protected E-R3-P2 link. Fail the link once under RSVP-FRR and once under TI-LFA. Compare **convergence (packet loss)** and **state/operational cost**.

**Solution**

Both achieve **<50 ms local repair** (0–1 lost packets) — repair *speed* is a tie because both pre-install a backup in the FIB. The decisive differences are **scale and operations**:

| Dimension | RSVP-TE FRR | TI-LFA |
|---|---|---|
| Repair time | <50 ms | <50 ms |
| Backup granularity | per **tunnel/facility** (must signal each) | per **prefix** (automatic, all) |
| Transit state | RSVP Path/Resv soft-state on every hop | **none** (stateless transit) |
| Provisioning | explicit backup tunnels | one IGP knob |
| Topology coverage | limited by tunnels you build | topology-independent (always finds repair) |

Conclusion for CCIE-SP: **TI-LFA is the modern default** — same speed, zero transit state, per-prefix automatic protection. RSVP-FRR remains where you specifically need RSVP bandwidth admission control / explicit TE alongside protection.

**Verification**
- Same traffic run twice: record packet loss for each (both ~0–1).
- `show rsvp counters` / `show rsvp session` — RSVP has state (FRR) vs none (TI-LFA).
- `show isis fast-reroute summary` (TI-LFA) vs `show mpls traffic-eng fast-reroute database` (RSVP) — per-prefix automatic vs per-facility manual.

---

## Section 6 — Troubleshooting

### Task 6.1 — Convergence > 1 second after a link failure

**Question**
After a core link fails, CE-to-CE traffic is lost for **several seconds** before recovering. Diagnose and fix.

**Solution**

Multi-second loss after a *link* failure points at the **detection** and **SPF-timing** layers of the convergence chain — not the label or BGP layers. Check, in order:

1. **BFD missing** — the failure is only detected when the IGP **hold-timer** expires (seconds). Fix: enable BFD (`bfd minimum-interval 50 / multiplier 3`) on the link so detection is ~150 ms (Task 2.4). This is the most common cause.
2. **SPF/LSP timers too conservative** — a large `spf-interval initial-wait` (or OSPF `timers throttle spf start`) delays the first SPF by seconds. Fix: lower `initial-wait`/`start` to ~5 ms (Tasks 2.1/2.3).
3. **No local repair** — even with fast detection, if there is no precomputed backup the router waits for full SPF+FIB reprogramming. Fix: enable **TI-LFA** `fast-reroute per-prefix ti-lfa` (Task 5.2) so repair is <50 ms.
4. **LDP-IGP sync gap** — if MPLS, verify the LSP reconverged too (Task 3.1); a blackhole here also looks like "slow convergence." SR removes this entirely.

**Verification**
- `show bfd session` — session must exist and be UP on the failed link (fix #1).
- `show isis` / `show ospf` — confirm small SPF `initial-wait`/`start` (fix #2).
- `show isis fast-reroute summary` — prefixes are protected (fix #3).
- Re-run the failure: loss now sub-second. `show isis spf-log` timestamp follows the BFD-down within ms.

### Task 6.2 — BGP slow convergence

**Question**
IGP/TI-LFA converges fast, but a **BGP-learned** (VPNv4 / internet) prefix takes seconds to recover after an **egress PE / external-link** failure. Diagnose and fix.

**Solution**

Fast IGP but slow BGP recovery isolates the problem to the **BGP layer** — the router is doing a **per-prefix bestpath walk** because it has no pre-installed backup. Two linked root causes:

1. **PIC not enabled / no backup in FIB** — without PIC Edge the router recomputes and reprograms **per prefix**, so time scales with table size. Fix: enable **PIC Edge** (`additional-paths selection ... set path-selection backup 1 install`, Task 4.1) so failover is a single prefix-independent pointer swap.
2. **RR reflecting only one path** — even with PIC configured, if the **RR advertises only its bestpath** the client has **no backup to install**, so PIC has nothing to swap to. Fix: enable **Add-Path** on the RR (Task 4.3) and/or **best-external** on the PEs (Task 4.4) so diversity actually reaches the client.

The two are a pair: **PIC needs a backup path, and the RR must actually deliver one.** Fixing PIC without fixing path diversity does nothing.

**Verification**
- `show cef <prefix> detail` — must show a **backup next-hop** and shared **path-list** (proves PIC has a backup; if missing, path diversity is the real gap).
- `show bgp <prefix>` — client must see **multiple** paths (proves Add-Path/best-external delivered diversity).
- `show bgp neighbor <RR> | i Additional` — Add-Path negotiated.
- Fail the egress PE: recovery now **sub-second and independent of prefix count**.

---

## CCIE Challenge Tasks

### Challenge A — Full-path convergence budget
Instrument a single CE-to-CE path across all 3 SPs (Emerald → ASBR → Gold → ASBR → Garnet). Fail one core link and produce a **per-layer timing breakdown**: BFD detect → TI-LFA repair → IGP reconverge → BGP (PIC) → total. Prove the total is dominated by detection, not computation, once every layer is tuned.

### Challenge B — NSR + TI-LFA interaction
Force an RP switchover **during** a link-failure repair. Show that SSO/NSR keeps the control plane while TI-LFA holds the data-plane repair, and that neither event resets neighbors — HA and fast-convergence layers composing correctly.

### Challenge C — LDP→SR convergence delta
On one core, measure LDP reconvergence (with IGP-sync + session-protection) vs the **same** core migrated to SR. Quantify the eliminated LDP convergence step and explain why SR's label plane is "already converged" the moment IS-IS is.

### Challenge D — microloop avoidance
Enable IGP microloop avoidance and demonstrate it removes transient loops during **restoration** (link coming back), correlating with the TI-LFA repair path from Section 5.
