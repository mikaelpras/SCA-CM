# Slide Outline — Partial NTT Masking: A Security Margin Analysis
**Iskander & Kirah · arXiv:2604.03813 (Apr 2026) · target: ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.
**Note:** a *red-team audit of a real shipping design* — not a generic attack or countermeasure paper. Also a **preprint**; say so on Slide 1.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** Partial NTT Masking in PQC Hardware — A Security Margin Analysis
Iskander (Verdict Security) & Kirah (Ain Shams University) · arXiv preprint, **not peer reviewed**

- Audits **Adams Bridge** — the PQC accelerator for the **Caliptra silicon root of trust** (AMD, Google, Microsoft, NVIDIA; Apache 2.0)
- The design pattern: **partial masking** — full NTT masking costs too much area, so mask *some* operations and argue search space for the rest
- Adams Bridge masks **the first INTT layer only**; claims per-butterfly CPA complexity **2⁴⁶ (ML-DSA)** / **2⁹⁶ (ML-KEM)**

> **Verdict:** 2⁴⁶ is correct arithmetic for **the wrong threat model** — and the shuffling meant to backstop it delivers **6 bits, not 296**.

---

## Slide 2 — The Target (45 s)

| | |
|---|---|
| Scale | 30 modules, **~1.17M gate-equivalent cells** |
| Butterflies | Gentleman–Sande (INTT), Cooley–Tukey (NTT) |
| ML-DSA | q = 8,380,417 (23-bit), **8 layers** |
| ML-KEM | q = 3,329 (12-bit), **7 layers** |
| Masking | First-order Boolean, **DOM, 2 shares — first INTT layer + PWM only** |

**The structural detail that matters:** after the masked first layer, **the two shares are recombined into one unmasked value** before continuing through layers 2–8.

> The designers don't hide this — their own docs say *"leakage naturally emerges in the unmasked layers."* The dispute is whether it's **exploitable**.

---

## Slide 3 — Finding 1: The Shuffling Entropy Gap ★ (90 s)

**No attack model needed — just read the public RTL.** (`ntt_ctrl.sv`, lines 648–653)

- Chunk-level RSI selects among **16 start positions** in a 16-element chunk
- Index-within-chunk adds a factor of **4**
- **16 × 4 = 64 orderings per layer → log₂(64) = 6 bits**

**The scaling argument assumed ~296 bits (full random permutation).**

**Why:** processing is **sequential with wraparound** — this is a **rotation**, not a permutation.
- Rotation of 16 elements → **16** orderings
- Permutation of 16 elements → 16! ≈ **2⁴⁴**

The design made the sensible *hardware* choice; the security argument was written as if it had bought the expensive one.

---

## Slide 4 — Finding 2: The Attack-Model Gap ★ (75 s)

| | **Per-butterfly CPA** (designers) | **SASCA / Belief Propagation** (reality) |
|---|---|---|
| Unit of attack | each butterfly, **independently** | the **whole factor graph, at once** |
| Exploits | one butterfly's leakage | **inter-butterfly dependencies** |
| Shuffling | multiplies the search space | **absorbed** — shuffle nodes resolve permutation *and* secret jointly |
| Assumes | **non-profiled** attacker | profiled / algebraic |

> The entire point of BP attacks is that the NTT's algebraic structure links butterflies — which is exactly what a per-butterfly independence assumption discards.

---

## Slide 5 — The Experimental Tracks (60 s)

Seven tracks, each with a **confidence rating** — an unusually honest methodology.

- **Soft-analytical pipeline:** **37-bit** enumeration reduction — *explicitly without key recovery*
- **Full-scale BP** on the complete INTT factor graph: **100% coefficient recovery**
  → resolves an open question — **BP gains do scale to production-size NTTs**
- **Genie-aided bound:** sufficient mutual information for full recovery at **SNR×N ≥ 15**
- **Layer ablation:** four necessary conditions for BP convergence

---

## Slide 6 — Finding 3: Topology Beats Count ★★ (90 s — the payoff)

> **4 evenly spread layers → 100% recovery**
> **4 consecutive layers → 0% recovery**

**Same number of observed layers. Opposite outcomes.**

BP stitches the graph together from spread-out observations; a contiguous observed block leaves a gap it cannot bridge.

**The design rule that falls out:**

| Strategy | Cost | Outcome |
|---|---|---|
| Full masking | 100% | secure, unaffordable |
| **3 consecutive mid-layers** | **43%** | **unrecoverable gap — defeats SASCA** |
| First layer only (Adams Bridge) | ~12% | **worst placement** — leaves everything downstream contiguous |

---

## Slide 7 — Three Independent Analyses Agree (45 s)

| Who | Method | Finding |
|---|---|---|
| Karabulut & Azarderakhsh (FAU) | Empirical CPA **on silicon** | Practical exploitability of the ML-DSA pipeline |
| Saarinen | RTL / pre-silicon review | Key **not arithmetically share-split** |
| Iskander & Kirah | Structural analysis + BP | **8,487 convergence points**; no fresh inter-stage masking |

**But:** the designers have published a concurrent response (IACR ePrint 2026/256). **This is a live dispute, not a settled verdict.**

---

## Slide 8 — Takeaways for Hardware Design (75 s)

- **Count your shuffler's actual orderings.** Randomised start index + wraparound = rotation = log₂(n) bits, not log₂(n!). Enumerate in RTL before putting a number in a datasheet.
- **State the threat model or the complexity figure is meaningless.** 2⁴⁶ is real under per-butterfly independence; against SASCA it isn't the relevant quantity.
- **Placement > coverage.** For a fixed masking budget, a contiguous mid-block beats a boundary layer.
- **Audit the masked/unmasked boundary.** Share recombination at that seam is where the argument fails — same principle as an unaccumulated product reaching a register, one abstraction level up.

**Caveats to state:**
- **No key recovery on silicon** — BP runs on the factor graph; the SASCA pipeline stops short
- Preprint, seven-paper self-citing series; lead author at a commercial security firm
- Genie-aided bounds flatter the attacker — information present ≠ extractable

---

## Cut List (in the deep dive, not the slides)

- The intra-stage vs inter-stage failure-mode taxonomy (`masking_en_ctrl`, rounds 1–3, 2ʳ multiplicity) — from the sibling papers, good verbal aside
- Caliptra 2.0 / 2.1 version history and the ML-KEM addition timeline
- The four BP convergence conditions individually — say "four conditions" and move on
- Montgomery vs Barrett reduction split across the two schemes
- FIPS 140-3 certification angle — one clause on Slide 8 if the audience is industry
- The full QANARY seven-paper series map — Paper 1's 8,487 figure is enough

## Timing Check
60 + 45 + 90 + 75 + 60 + 90 + 45 + 75 s ≈ **9 min** — slightly over. Trim Slide 7 to one line ("two other independent analyses agree; designers dispute it") to recover 30 s.

**Protect Slides 3 and 6.** Slide 3 is the claim that needs no attack model and is independently checkable — the strongest thing in the paper. Slide 6 is the constructive payoff and the only slide that gives the audience something to *do*.
