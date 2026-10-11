# Flashcard Workflow (Morning Study Routine)

## Trigger phrases
- **"let's do 10 morning flashcards"** → run the 10-card morning warm-up (see below)
- **"let's do 5 new flashcards"** → 5 fresh cards from what was just read
- **"let's do flashcards on <topic>"** → targeted set on one topic

## The daily rhythm
```
1. MORNING WARM-UP  → "let's do 10 morning flashcards"   (~15 min)
2. READ             → finish/continue the SPCOR chapter, take notes
                       (consolidate into topics/ files — one topic per file)
3. NEW CARDS        → "let's do 5 new flashcards"         (~10-15 min)
```
Total morning block ≈ 45 min. Consistency > intensity — a short day still counts.

## What the agent does for "10 morning flashcards"

**Card selection (from `flashcard-deck.md`):**
1. **Prioritize low/flagged cards first** — anything scored ≤ 7 or marked "review" (spaced repetition on weak spots)
2. **Then mix in cards from covered topics** for breadth (MPLS, L2VPN/AToM, VPLS/H-VPLS, Carrier Ethernet/ERPS, CFM/OAM, and any newer topics)
3. Aim for a spread across topics, not 10 from one area

**Delivery:**
- Ask **ONE card at a time**. User answers from memory. Do NOT show the answer first.
- After each answer: **score /10**, give the model answer, and note precisely what to tighten (usually exact vocabulary/mechanism).
- After 10 cards: **summary table** (card / topic / score) + list of flagged items for next time.

**Update the deck:**
- After the session, update `flashcard-deck.md` scores/dates for the cards run (and add any new cards from the "5 new" session).

## Grading guidance
- Reward correct *concepts* but push on **precise terminology** and **mechanism** (that's where Renato's gaps are — he understands concepts, glosses exact terms).
- ≤7 or "unknown" → short review interval (comes back soon).
- 9-10 → long interval.

## Level calibration (current phase = SPCOR/CCNP-SP)
- Keep cards at **concept + precise terminology + config awareness**.
- Do NOT go full CCIE-lab depth yet (config syntax, troubleshooting, packet-level) — that's a later phase.
- Let the material being read/labbed drive the depth.

## Card pool topics (grows as chapters are read)
MPLS fundamentals · L2VPN/AToM · VPLS/H-VPLS · Carrier Ethernet/ERPS · CFM/OAM
(next: L3VPN, EVPN, Segment Routing, BGP, IS-IS, OSPF, multicast, QoS, security, HA, automation)
