# Slide Outline — Parallel PC Oracle Attacks on Kyber KEM
**Rajendran, Ravi, D'Anvers, Bhasin, Chattopadhyay · TCHES 2023 · target: ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** Parallel PC Oracle Attacks on Kyber KEM — Rajendran et al., TCHES 2023 (NTU Singapore / KU Leuven)

- Generalises **Plaintext-Checking oracle** chosen-ciphertext attacks from **1 bit per query → P bits per query**
- Target is the **FO re-encryption**, not the polynomial multiplication
- Validated at **P = 10** (1024 classes), 100% success, pqm4 Kyber768 over EM
- **2.89×–7.65×** fewer queries than the binary baseline — and **shuffling doesn't help at all**

> Say: "The leakage was always there. The previous attacks just weren't using it."

---

## Slide 2 — The Trade-off This Paper Attacks (45 s)

| | **TO_Dep** (target-operation dependent) | **TO_Indep** (generic) |
|---|---|---|
| Targets | One operation (msg encoding, NTT) | Anywhere in decapsulation |
| Traces | Very few — single-trace possible | Few **thousand** |
| Needs | Detailed target knowledge, high SNR | Almost nothing |
| Survives complex HW? | Unclear | Yes |

**Gap:** easy-to-mount attacks are expensive; cheap attacks are fragile.
**Their claim:** get both.

---

## Slide 3 — Baseline: The Binary PC Oracle (60 s)

**Chosen ciphertext:** `u = k_u·x⁰`, `v = k_v·x⁰`

```
m₀ = F(s[0])        ← only this bit is secret-dependent
mᵢ = 0   for i ≥ 1
```

- So `m ∈ {0, 1}`, and which one **depends solely on s[0]**
- Oracle: `Re_Encrypt(m, pk)` is deterministic in m → one bit flip changes *everything* downstream → thousands of distinguishable PoIs
- Rotation trick (`u = k_u·x^p`) walks the rest of the key
- BDT optimisation exploits the non-uniform CBD distribution

**Cost:** 1312 / 1776 / 2368 queries for Kyber512 / 768 / 1024

---

## Slide 4 — The Core Idea ★ (2 min — money slide)

**Two observations:**
1. Only **1** of 256 message bits is secret-dependent per query
2. **Thousands** of leaky PoIs are spent to extract that **1 bit**

**Fix — new chosen ciphertext:**
```
u = 208 · x⁰                        (k_u FIXED)
v = 208 · t · Σᵢ₌₀^(P−1) xⁱ

→  mᵢ = F(s[i])   for i ∈ [0, P−1]
   mᵢ = 0         otherwise
```

- First **P** message bits now depend on **P different secret coefficients**
- Attacker controls both the **number and position** of secret-dependent bits (unlike the FD oracle, where all 256 depend on s)
- k_u fixed by exhaustive search → varying only **v** suffices → the **P BDT traversals stay independent**

→ Traverse **P decision trees in a single query.**

---

## Slide 5 — Why the Optimal Tree Changes (60 s)

**Non-obvious result:** the BDT that's optimal for the binary attack is *wrong* in parallel.

- Cost to recover a **set** of P coefficients = depth of the **deepest** member, not the average
- As P grows, probability of hitting a deep coefficient → 1
- So **minimum-depth** beats **minimum-entropy** for **P ≥ 3**

| | Kyber512 [−3,3] | Kyber768/1024 [−2,2] |
|---|---|---|
| Min-entropy tree | depth 4 | depth 3 |
| New min-depth tree | **depth 3** | already optimal — same tree |

`Q_attack = ⌈256/P⌉ · k · Q_set`

---

## Slide 6 — Building a 2^P-Class Oracle (60 s)

**Enabling observation:** hash diffusion in re-encryption means **any two** values of m are distinguishable by t-test — not just 0 vs 1. (Shown for m = 330 vs m = 559.)

**Pre-processing** (per public key): T traces per class → normalise → Welch's t-test → select PoIs → mean = template
**Classification:** sum-of-squared-difference, **knockout tournament** → 2^P − 1 pairwise matches (optimal)

- **T = 5** traces per class was enough
- No known-key profiling. No clone device required.
- Strongest leakage found at **CBD sampling of ephemeral r** — attacker never needs to know this

---

## Slide 7 — Cost & Results (60 s)

```
Q_total = 2^P · T   +   ⌈256/P⌉ · k · Q_set
          ↑ exponential      ↑ inverse in P     → a minimum exists
```

**Kyber768, STM32F407VG @ 24 MHz, EM:**

| Scenario | Optimal P | Queries | vs baseline |
|---|---|---|---|
| Baseline (binary) | 1 | 1776 | — |
| **With clone device** | 10 | **232** | **7.65×** |
| **Without clone** (T=5) | 4 | **613** | **2.89×** |
| *Hypothetical, 2³² templates* | *32* | *72* | *~24.6×* |

Partial recovery (2⁶⁴ offline work): 232 → **140** queries

---

## Slide 8 — Shuffling Defeated + Takeaways (75 s)

**Shuffling protects message encoding.** This attack never touches message encoding — it uses leakage only up to the sampling of r.

→ **Same trace count on shuffled as on unprotected.** Lowest known query count against shuffled Kyber.

**Takeaway:** the vulnerability is **algorithmic** — determinism of FO re-encryption in m. No amount of datapath restructuring, pipelining, or instruction rescheduling touches it. Protecting the polynomial multiplier alone does not protect an ML-KEM core.

**Caveats worth stating:**
- The 24.6× figure needs 2³² templates — an asymptote, not an attack. Validated numbers are 7.65× / 2.89×
- **Masking is explicitly out of scope** — no masked target was attacked, so "masking is necessary" is argued from outside
- Pre-processing must be **repeated for every new public key**
- No upper bound established on P; depends on SNR and target

---

## Cut List (in the deep dive, not the slides)

- Full CPA.KeyGen / Encrypt / Decrypt and CCA.Encaps / Decaps pseudocode — one line on re-encryption determinism is enough
- The Decryption-Failure (DF) oracle family — mention only if asked
- `Q_set = Σ i·Rᵢ` derivation and the subtree notation — state the result, skip the formalism
- Comparison with Tanaka et al. — good verbal aside (they used an NN needing 1500 traces/class), not a slide
- Saber / Frodo / LPR generalisation and the Saber BDT — one sentence on Slide 8 if time allows
- Partial-key-recovery table for all three parameter sets — one number on Slide 7 suffices
- Full experimental equipment list — the DUT and channel type are enough

## Timing Check
60 + 45 + 60 + 120 + 60 + 60 + 60 + 75 s ≈ **8.7 min**. Slides 4 and 5 are the pair to protect — 5 doesn't land without 4. If you run long, compress Slide 3 to just the chosen-ciphertext form and the 1776 figure.
