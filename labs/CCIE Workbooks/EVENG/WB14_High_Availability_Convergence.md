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


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.2 — Trigger an RP switchover and prove data-plane continuity

**Question**
Start continuous CE-to-CE traffic through E-R1. Force an RP switchover (`redundancy switchover`). Prove there is **zero (or near-zero) traffic loss** and that neighbors did **not** reset the adjacency.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 1.3 — When NSF/NSR is the *wrong* answer

**Question**
Explain a scenario where enabling GR/NSF actually **slows** or **harms** convergence, and state the design rule.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 2 — IGP Convergence (IS-IS & OSPF tuning + BFD)

### Task 2.1 — IS-IS SPF and LSP-generation throttling

**Question**
Tune IS-IS on the Emerald core so the **first** SPF after a change runs almost immediately, but a flapping link **backs off** to protect CPU. Do the same for LSP generation. Explain the three exponential-backoff values.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.2 — PRC vs full SPF, and IS-IS fast-flood

**Question**
Explain why a **leaf prefix** change (e.g., a redistributed /32) does **not** require a full SPF, and enable **fast-flood** so the failing router floods its new LSP *before* running SPF.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.3 — OSPF SPF throttling (Gold ISP)

**Question**
Gold runs OSPF. Apply the equivalent **SPF throttle** so first SPF is fast and repeated events back off.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 2.4 — BFD for sub-second detection on **all** IGP adjacencies

**Question**
IGP hold timers detect failure in *seconds*. Enable **BFD** so every IGP adjacency (IS-IS on Emerald/Garnet, OSPF on Gold) detects failure in **sub-second** and immediately triggers convergence / TI-LFA.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 3 — LDP / MPLS Convergence

### Task 3.1 — LDP-IGP synchronization

**Question**
After a link comes up (or LDP restarts), the IGP may start using it **before LDP has exchanged labels**, causing a **blackhole** (IP forwards, but no label → dropped LSP). Configure **LDP-IGP sync** so the IGP does not use the link until LDP is ready.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.2 — LDP session protection

**Question**
A **link flap** normally tears down the directly-connected LDP session and forces relearning all label bindings. Enable **LDP session protection** so the session survives via a **targeted** session over an alternate path.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 3.3 — LDP backoff, and why SR needs none of this

**Question**
Under repeated LDP setup failures (e.g., a misconfigured/label-conflicting neighbor), tune **LDP exponential backoff** to avoid hammering the peer. Then explain why **Segment Routing** makes this entire section unnecessary.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 4 — BGP Convergence (PIC, Add-Path, best-external)

### Task 4.1 — BGP PIC Edge (prefix-independent convergence)

**Question**
On Gar-R1 (Garnet), a VPNv4/eBGP prefix has a primary and a backup path. Configure **BGP PIC Edge** so that on primary-path failure, convergence is a **single FIB pointer swap** independent of the number of prefixes.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.2 — BGP PIC Core (IGP triggers BGP convergence)

**Question**
The failure is **inside the core** (a P-node or core link on the path to the BGP next-hop), not at the edge. Configure so **IGP convergence directly drives BGP** without a BGP walk.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.3 — BGP Add-Path for RR path diversity

**Question**
A Route Reflector by default advertises **only its single best path**, so clients never learn a backup — defeating PIC. Enable **Add-Path** on the RR so clients receive multiple paths.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 4.4 — BGP best-external

**Question**
On a PE that has both an **external (eBGP)** path and an **internal (iBGP via RR)** path where the **iBGP** path wins bestpath, the PE normally **stops advertising** its external path to the core (only bestpath is advertised). Enable **best-external** so the PE still advertises its external path into iBGP.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 5 — MPLS-TE FRR vs TI-LFA

### Task 5.1 — RSVP-TE Fast Reroute (link/node protection)

**Question**
On the Emerald core between E-R3 and E-R4, build a primary RSVP-TE tunnel and a **pre-signaled backup tunnel**; enable **FRR** so a protected link failure switches to the backup in <50 ms.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.2 — TI-LFA (per-prefix, zero transit state)

**Question**
On the same failure scenario, use **TI-LFA** (SR/IGP-computed) instead of RSVP-FRR. Contrast the state model.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 5.3 — Same failure, measure and compare

**Question**
Run **identical** continuous CE-to-CE traffic through the protected E-R3-P2 link. Fail the link once under RSVP-FRR and once under TI-LFA. Compare **convergence (packet loss)** and **state/operational cost**.


> *Try this yourself first. Solution available in `solutions/` folder.*

## Section 6 — Troubleshooting

### Task 6.1 — Convergence > 1 second after a link failure

**Question**
After a core link fails, CE-to-CE traffic is lost for **several seconds** before recovering. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

### Task 6.2 — BGP slow convergence

**Question**
IGP/TI-LFA converges fast, but a **BGP-learned** (VPNv4 / internet) prefix takes seconds to recover after an **egress PE / external-link** failure. Diagnose and fix.


> *Try this yourself first. Solution available in `solutions/` folder.*

