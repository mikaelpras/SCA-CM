# Slide Outline — Configurable SCA Countermeasures for the NTT
**Ravi, Poussier, Bhasin, Chattopadhyay · SPACE 2020 · target: ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.
**Note:** this is a *countermeasure* paper — structure differs from the attack papers.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** On Configurable SCA Countermeasures Against Single Trace Attacks for the NTT
Ravi, Poussier, Bhasin & Chattopadhyay, SPACE 2020 (NTU Singapore) — cited everywhere as **[RPBC20]**

- The NTT falls to **single-trace** SASCA (Primas CHES'17, Pessl & Primas LATINCRYPT'19)
- Shuffling had been *proposed* as the defence — but **nobody had measured what it costs**
- This paper: **local masking** + **three shuffling variants** + the first real cost study on pqm4 / Cortex-M4

> **Headline:** 7–78% overhead for Kyber. **32–490% for Dilithium signing.**
> Affordable for a KEM, brutal for a signature scheme.

---

## Slide 2 — Why Standard Masking Doesn't Help ★ (75 s)

**Set this up before anything else, or the countermeasures make no sense.**

- SASCA recovers the secret from **within a single linear NTT operation**
- → splitting the NTT into two shares **does not meaningfully change attack efficiency**
- → Boolean/arithmetic masking — the default answer everywhere else — **does not apply here**

**Target attack:** Hamburg et al. (TCHES 2021) — chosen-ciphertext *k*-trace attack on **masked CCA2 Kyber**. BKZ-crafted compressible ciphertext gives a sparse INTT input (~25% non-zero), recovers the **long-term** key.

> Contrast with the other papers in this seminar: there, masking works and the debate is cost. Here it isn't even on the table. Hence *hiding* countermeasures.

---

## Slide 3 — Countermeasure 1: Local Masking (60 s)

**Multiplicative masking of the twiddle factors** — not share-splitting.

- Mask the twiddles → the intermediates flowing through the NTT are masked too
- Map `m : Z_q × W → Z_q` over the twiddle set `W = {ζⁱ mod q}`
- **The efficiency trick:** `ζᵃ·ζᵇ = ζᵃ⁺ᵇ` is itself a twiddle power → masked twiddles are **already in the precomputed table** → *no extra multiplication per butterfly*
- Extends Saarinen's blinding by **re-masking at every layer**, not just the input
- Configurable: **`u` = masks per layer**; four protection levels in the released code

> Authors' own phrasing: *"cheap and low-entropy masking."* Remember that phrase for Slide 7.

---

## Slide 4 — Countermeasure 2: Three Shuffling Granularities (60 s)

| Variant | What is permuted | Entropy per layer |
|---|---|---|
| **Fine** | load/store order **inside** a butterfly | 1 bit / butterfly |
| **Coarse block** | butterflies sharing the same ζ | ((n/2m)!)ᵐ |
| **Coarse full** | all butterflies in a layer | (n/2)! |

- Fine shuffling uses an **arithmetic conditional swap** (Hutter–Schwabe) — no secret branches, no secret lookups
- **Note the inversion in coarse block:** Kyber INTT, first layer has 64 blocks → only **2⁶⁴**; last layer 1 block → **128!**
  → **the weakest layers are the first ones — exactly where an attacker starts**

---

## Slide 5 — The Cost (60 s)

pqm4 implementations, ARM Cortex-M4:

| Scheme / procedure | Overhead |
|---|---|
| **Kyber** — all procedures | **7% – 78%** |
| **Dilithium** — key generation | **12% – 197%** |
| **Dilithium** — signing | **32% – 490%** |
| NTT alone (shuffling) | 181% – 356% |

**Why Dilithium is so much worse — not incidental:**
- Far more NTTs per operation — public matrix **A** alone needs k×l transforms
- **Rejection-sampling loop** repeats the whole polynomial body an unpredictable number of times, multiplying every per-NTT overhead

---

## Slide 6 — What Happened Next ★ (90 s — the payoff slide)

**The paper measured cost. It never evaluated security against an *adapted* attacker.** Three follow-ups did.

**Hermelink et al., TCHES 2023 — shuffling:**
| Variant | Outcome |
|---|---|
| Fine | **Broken.** "Mixing priors" works to σ=0.8; a learning **shuffle node** (updates ω by KL-divergence during BP) to σ=1.0 |
| Coarse block | **Broken** for a resourceful attacker — two-point matching + Sinkhorn–Knopp, σ=0.4 |
| Coarse full | **Survives** their techniques — needs extra twiddle-factor information |

Their verdict: the countermeasures are powerful, but *"could lead to a false security perception."*

**Carrera Rodriguez et al., IACR CiC 2025 — "Cracking the Mask":**
> At low `u`, attacking the **local-masked** NTT **outperforms** attacking the **unmasked** NTT.

The mask nodes add exploitable structure to the factor graph. The countermeasure, cheaply configured, is **worse than nothing**.

---

## Slide 7 — Takeaways (60 s)

- **"Cheap and low-entropy" was the tell.** The precomputation trick that makes local masking fast is exactly what constrains the mask space.
- **Only the most expensive variant survived.** Hiding buys security roughly in proportion to the entropy you pay for.
- **A security parameter that inverts at low settings is a dangerous default** — configurability without per-level security evidence.
- **Scope critique:** widely adopted and implemented on the strength of a paper that measured only performance. Legitimate scope; consequential gap.
- Local masking also turned out **not constant time** (runtime varies with `u` and butterfly type) — a countermeasure introducing a timing channel.

---

## Slide 8 — Relevance to Hardware / ML-DSA (60 s)

- **The Dilithium number is the one that matters** for signature IP — but the *software* cost model doesn't transfer. In hardware, shuffling costs **control logic + RNG**, not cycles (permutation network, randomised address generation); local masking costs **memory**, not time. The Kyber/Dilithium *asymmetry* does transfer — it comes from NTT count and the rejection loop, not the platform.
- **Two orthogonal threat models.** A masked polynomial multiplier answers CPA-class attacks. It does **not** answer single-trace SASCA on the NTT. "First-order masked" without naming the threat model is an incomplete claim.
- **No NTT → no factor graph.** A coefficient-domain sparse multiplier has no butterfly network and no twiddle constants for BP to run on. Genuine structural advantage — though ML-DSA still uses the NTT elsewhere, so the surface shrinks rather than vanishes.

---

## Cut List (in the deep dive, not the slides)

- Full SASCA / belief-propagation background — say "graph-based inference on load/store leakage" and move on
- MSISO / MSIDO / MDISO / MDIDO butterfly taxonomy — follow-up work's vocabulary, not this paper's
- The fine-shuffling weak points the authors themselves flag (butterfly multiplication, swap-mask leakage) — good verbal aside
- Hermelink's first-layer un-shuffling complexity (2¹⁶ → 2⁸ BP runs) — detail
- The 2024 hardware t-test paper — one clause on Slide 7 if time
- Per-level breakdown of the four protection levels — not in the sources anyway (flagged in the deep dive as to-verify)

## Timing Check
60 + 75 + 60 + 60 + 60 + 90 + 60 + 60 s ≈ **8.75 min**.

**Structural note:** unlike the attack papers, the money slide here is **Slide 6, not Slide 3**. The mechanisms are ordinary; the story is that all three were later weakened and one is worse than nothing. Protect Slides 2 and 6 — Slide 2 makes the problem legible, Slide 6 is the payoff. If you run long, compress Slide 4's table to the entropy column.
