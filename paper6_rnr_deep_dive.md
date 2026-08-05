# Paper 6 Deep Dive — Redundant Number Representation as an SPA Countermeasure

Nagpal, Hadžić, Primas, Mangard. *Efficient SPA Countermeasures Using Redundant Number Representation with Application to ML-KEM.*
**SAC 2025, LNCS 16207, pp. 753–780.** ePrint 2025/679. Code: `github.com/rishubn/rnr-kyber-spa`.
Affiliations: TU Graz (Nagpal, Mangard) and Intel Labs (Hadžić, Primas).

Derived material — my own computation, not stated in the paper — is marked **[derived]**.

---

## 1. Position in the seminar

This is the only **defensive** paper on your list, and it is the natural closer for two reasons.

**It explains, information-theoretically, the exact leakage that Paper 2 exploited.** Paper 2's Attack B hinges on the observation that a small negative number in two's complement has a very high Hamming weight while a small positive one has a very low weight. Paper 6 measures that phenomenon: for ML-KEM's q = 3329 in a 16-bit word, the signed representation leaks **3.561 bits** per coefficient through Hamming weight, versus **2.756 bits** unsigned — and the gap is essentially the sign bit. Tosun et al. (TMS24) independently spotted the same thing for DPA. Paper 2 and Paper 6 are describing the same physical fact from opposite sides.

**It is a direct competitor to Papers 4 and 5.** The paper explicitly positions RNR as "a low-cost alternative to shuffling," and benchmarks itself against the masking/shuffling NTT countermeasures of Ravi et al. (Rav+20, 7–78 % overhead) as evaluated by Hermelink et al. (Her+23b).

**Scope warning worth stating early:** RNR protects the NTT/INTT. It does **nothing** against Paper 1's PC-oracle attack, which targets the hash inside the FO transform, not polynomial arithmetic. Different attack surface, different defence.

---

## 2. Threat model — different from Papers 1 and 2

| | Papers 1 & 2 | Paper 6 |
|---|---|---|
| Class | Chosen-ciphertext, multi-trace | **SPA / SASCA, single-trace** |
| Traces for key recovery | 10² – 10⁴ | **1** |
| Attacker knowledge | oracle responses | full algorithmic structure + profiled templates |
| Inference | decision tables / inequality solver | **loopy belief propagation on a factor graph** |
| Target operation | FO hash, message polynomial | **NTT⁻¹** |

**SASCA** (Veyrat-Charvillon et al., VGS14) is the key concept. Rather than attacking one intermediate, you encode the whole algorithm as a factor graph — variables as circles, operations as factors — attach a leakage distribution to every intermediate you can profile, and run belief propagation until the marginals converge. For the NTT this is devastating, because seven butterfly layers give you seven views of every coefficient.

Two metrics you will need:

- **Mutual Information** `I[X;L] = H[X] + Σ Pr[x] Σ Pr[ℓ|x] log₂ Pr[x|ℓ]` — the true leakage
- **Perceived Information** — the same quantity with a *model* substituted for the true distribution. PI is a **lower bound** on MI and converges to it as the model improves. Used here to assess template quality.

---

## 3. The central observation

Hardware works in Z_{2^l}; cryptography works in Z_q. When **q ≪ 2^l**, that mismatch is a leak.

The Hamming weight of a machine word is not equally informative in all cases:

- `X ∈ {0,1}` stored in a whole word: `W(X) = X`, so `I = 1` bit — **perfect** leakage
- `X ∈ Z_{2^16}` filling the word: `I ≈ 3` bits, but `H[X] = 16`, so the *relative* leak is small
- `X ∈ Z_3329` in a 16-bit word: `H[X] = 11.701`, and the leak is 2.7–3.6 bits — a much worse ratio

And **the representation matters**. For q ≪ 2^l, two's complement makes the sign of X immediately visible in W(X), worth roughly one extra bit.

### Verified — [derived]

I recomputed the paper's headline figures directly (X uniform over Z_q, W = popcount of the 16-bit word):

| Representation | H[W(X)] | H[W(X)\|X] | **I[X;W(X)]** | Paper |
|---|---|---|---|---|
| Signed, Z_q | 3.561 | 0 | **3.561** | 3.561 ✓ |
| Unsigned, Z_q | 2.756 | 0 | **2.756** | 2.756 ✓ |
| RNR±, η = 9, signed | 3.149 | 2.243 | **0.906** | 0.888 (≈) |
| RNR⁺, η = 5, unsigned | 2.958 | 1.739 | **1.219** | 1.219 ✓ |

Three of four match exactly. The RNR± value differs by ~2 %; I tried three plausible range conventions and got 0.906 each time, so it is likely a minor definitional difference, not an error in either direction.

**Note the mechanism in the middle columns.** For the non-redundant cases `H[W|X] = 0` — the Hamming weight is a *deterministic function* of the value, so all of H[W] leaks. RNR does not primarily reduce H[W]; it manufactures **conditional entropy**.

---

## 4. Why RNR works — the identity that drives everything

```
I[X ; W(X)]  =  H[W(X)]  −  H[W(X) | X]
```

To reduce the leak you must either

1. **flatten** the weight distribution (reduce H[W(X)]), or
2. **increase H[W(X) | X]** — make the weight uncertain *given* the value.

Option 2 is only possible if you break the constraint `q_h − q_l = q`. That is, you must allow **multiple machine encodings per residue class**. And once you do, you must redefine addition, multiplication and the modular reductions so results stay in the enlarged range.

That is RNR in one sentence: **compute in Z_{ηq} instead of Z_q.**

### The workflow (Section 3.3)

1. Start from an algorithm over Z_q with inputs X and outputs Y
2. Determine the largest η such that no operation overflows or breaks congruence in Z_{ηq}
3. Encode inputs as `X′ = X + Kq`, with K uniform in `[0, η)` (unsigned) or `[−η/2, η/2]` (signed)
4. Run the whole algorithm over Z_{ηq}
5. Decode outputs by reducing mod q

**The property that makes this cheap:** RNR needs **no fresh randomness during computation** and **no extra operations**. Only two things change — the rejection sampling in `GeneratePolyMatrix` / `SamplePolyCBD` must emit coefficients in Z_{ηq}, and the reduction constants must be adjusted. Contrast with masking (fresh shares, doubled arithmetic) or shuffling (permutation generation, indirection).

**Where the sweet spot is:** with η = ⌊2^{l−1}/q⌋, RNR drives I below 0.5 bits for **q ≤ 2^{l/2}**, and the benefit vanishes as q → 2^{l−1}. ML-KEM's q = 3329 ≈ 2^{11.7} sits in the intermediate regime — hence a residual ~0.9–1.2 bits rather than near-zero.

---

## 5. Choosing η for ML-KEM

The constraint is that the NTT butterfly must not overflow 16 bits. Signed CT butterfly: `a′ ≡ a + bζ`, `b′ ≡ a − bζ`, with `bζ` reduced by signed Montgomery.

**Signed** — first layer has `c = η±q/2`, `z = q/2`, Montgomery output in `[−u, u]` with `u = cz/2^16 + η±q/2`. Enforcing `c + u < 2^15`:

```
η± · (q + q²/2^18) < 2^15      ⟹      η± < 2^33 / (2^18·q + q²)
```

**Unsigned** — `c = η⁺q`, `z = q`, `u = cz/2^16 + η⁺q`. The subtraction must be computed as `b′ = a + ⌈u/q⌉q − t` to avoid underflow, giving the tighter constraint `c + u + q < 2^16`:

```
η⁺ · (2q + q²/2^16) < 2^16 − q      ⟹      η⁺ < (2^32 − 2^16·q) / (2^17·q + q²)
```

**Verified — [derived]:** signed bound = **9.720**, unsigned bound = **9.112**. Both < 10, as the paper states.

**Cost of a larger η:** both RNR variants need **two extra Barrett reductions** in the forward NTT to keep the invariant `a, b ∈ [−c, c]` across layers. In the conventional η = 1 implementation, the forward NTT reduces only after the *last* layer. For the inverse NTT (signed case), a fixpoint argument shows the Montgomery output `b′` needs no extra Barrett reduction — which is why **the INTT is free**.

---

## 6. Parameter forensics: η⁺ is 5, not 9 — [derived]

The text says "we can choose η⁺ := 9," but **Table 1 says η = 5 for RNR⁺ and 9 for RNR±**. This matters, so I resolved it three independent ways — all three say **Table 1 is right and the text is a typo**:

**(a) The entropy figures.** The paper reports RNR coefficient entropies of 14.871 and 14.023 bits.
- log₂(9q) = log₂(29961) = **14.871** ✓ → η± = 9
- log₂(5q) = log₂(16645) = **14.023** ✓ → η⁺ = 5 (log₂(9q) would be 14.871, not 14.023)

**(b) The joint butterfly entropies**, 29.741 and 28.046 bits, are exactly 2× the above. ✓

**(c) The Montgomery constants in Table 1.**
- η⁺ = 5 → m = 16645, and `−m⁻¹ mod 2^16 = ` **39987** — Table 1's RNR⁺ entry ✓
- η± = 9 → m = 29961, `m⁻¹ mod 2^16 = 43321`, which as a *signed* 16-bit value is **−22215** — Table 1's RNR± entry ✓

Both constants land exactly. So: **η± = 9, η⁺ = 5.**

**An open question worth raising in discussion.** The bound permits η⁺ = 9, and the paper never explains why 5 was chosen. My computation says η⁺ = 9 would give **I = 0.835 bits** — *lower* than RNR±'s 0.906. That would invert the paper's headline recommendation. The paper argues RNR± wins because signed storage leaves a smaller unused gap (`2^16 − η⁺q > 2^15 − η±q`), but that heuristic doesn't actually predict the MI ordering once η⁺ is raised. Good question for the room: **is RNR± really better, or is it just better-parameterised?**

---

## 7. The information budget for the ML-KEM NTT

The paper's framing of what an attacker must overcome. All arithmetic verified.

| Quantity | Signed | Unsigned | RNR± (η=9) | RNR⁺ (η=5) |
|---|---|---|---|---|
| H per coefficient | 11.701 | 11.701 | 14.871 | 14.023 |
| I[X; W(X)] per coefficient | **3.561** | 2.756 | 0.888 | 1.219 |
| Over 256 coefficients | 911.6 | 705.5 | — | — |
| Residual entropy (of 2995.4) | 2083.8 | 2289.9 | — | — |

So **switching signed → unsigned alone denies the attacker ≈ 206 bits** across a polynomial. Free, but not sufficient.

**The butterfly makes it worse.** With joint input entropy H[A,B] = 23.402 bits, knowing the four Hamming weights W(A), W(B), W(A′), W(B′) yields:

- **13.879 bits** (signed) — over half the joint entropy from one butterfly
- **10.877 bits** (unsigned)
- 12.549 bits (RNR±), 11.744 bits (RNR⁺)

The signed-vs-unsigned gap of 3.002 bits jointly is the "≈1.5 bits per coefficient" the paper mentions. Under RNR the same gap shrinks to 0.805 bits. *(Small inconsistency: the paper describes 1.5 as per-coefficient but 0.804 as joint. Compare like with like: 3.002 → 0.805 jointly.)*

Note that RNR **raises** entropy per coefficient (11.7 → 14.9) while **lowering** leakage (3.56 → 0.89). Both directions help the defender, and that dual effect is the core of the paper's argument.

---

## 8. The attack being defended against

**Hamburg et al.'s k-trace attack (Ham+21).** Craft ciphertexts so that a chosen number of NTT-domain coefficients are zero. Then the input to `NTT⁻¹(ŝ ∘ NTT(u))` is a **sparse** vector, and SASCA on the INTT recovers it, from which the key follows.

**Simplification used here:** assume recovering **32 contiguous non-zero coefficients** suffices — originally only applicable to Kyber1024 in Hamburg et al. The authors justify it as effective in high-noise regimes and as modelling a *more powerful* adversary needing less profiling effort. Entropy to eliminate: **32 × 11.701 = 374.4 bits**. The signed→unsigned change alone removes 32 × 0.805 ≈ **25.7 bits** of the attacker's information.

**Why this is the right worst case:** the paper notes k-trace attacks work against *masked* NTT⁻¹ implementations too, because shares can be attacked divide-and-conquer style with SPA. So masking alone does not close this hole — which is precisely the argument for a complementary countermeasure.

### The "amplification effect" — a profiling gift

This is a subtle and important detail. Because 224 of the 256 INTT inputs are **zero and known with certainty**:

1. No templates are needed for those positions
2. When a zeroed intermediate is a GS-butterfly operand, the output is the **unchanged or inverted** other input (before ζ-multiplication)
3. So the *same value* reappears across several INTT layers, giving the attacker multiple independent looks at it

Result: templates needed drop from 1792 intermediates to **864**, and deeper-layer templates are far stronger than shallow ones. The attacker's own chosen-ciphertext structure is what creates the amplification.

---

## 9. Simulated SASCA results (Fig. 2)

Leakage model `ℓ = HW(x) + N(0, σ)`; factor graph over the input vector plus 7 GS-butterfly layers, butterflies merged into single factors (per PP19, to eliminate short loops); **1792 factors**; 25 BP iterations, no heuristics (per Nag+25); 25 experiments per noise level, σ from 0.1 to 3.2.

| Implementation | Behaviour |
|---|---|
| Signed | 100 % success until **σ ≥ 3.0** |
| Unsigned | degrades significantly from **σ ≥ 1.6** |
| **RNR±** | **completely ineffective at every noise level** |
| RNR⁺ | succeeds ~**25 %** when σ ≤ 0.1, ineffective above |

The signed-vs-unsigned gap is the money result for implementers: **the faster signed arithmetic that everyone uses for performance is measurably less secure.**

---

## 10. Practical results on Cortex-M4 (Section 5)

**Setup** — note this differs from Papers 1 and 2:

| | Paper 6 | Papers 1 & 2 |
|---|---|---|
| Board | ChipWhisperer CW308 UFO + **STM32F303** | STM32F4DISCOVERY / STM32F407 |
| Scope | Picoscope 6404C, 20 dB external LNA | LeCroy 610Zi |
| Clock | 10 MHz, **synchronised to sampling** | 24 MHz, asynchronous |
| Channel | Power | EM (near-field probe) |

Synchronous sampling on a ChipWhisperer is a deliberately *favourable* setup for the attacker — no jitter, perfect alignment. That strengthens the security claim: RNR holds even under near-ideal measurement conditions.

**Profiling:** 100 000 traces with the first 32 intermediates random and the rest zeroed. SNR computed per intermediate; POIs above a threshold, up to 1000 per intermediate; **LDA classifiers** via SCALib, subspace dimension tuned by parameter sweep.

**Results:**

- **SNR** (Fig. 3): signed shows peaks above 1; unsigned drops it by an order of magnitude; **both RNR variants drop it by two orders of magnitude**, RNR± lowest
- **PI** (Fig. 4): signed and unsigned reach near-certain prediction by **layer 4** thanks to amplification. RNR shallow layers have PI ≈ 0; maximum PI anywhere is **5.8 bits (RNR±)** and **7.8 bits (RNR⁺)**
- **Guessing entropy after SASCA** (Fig. 5): signed and unsigned inputs recovered from a **single trace**. Against RNR, BP "is unable to significantly improve the GE of the output of the templates" — it reduces GE in *deep* layers (many known zeros) but not in shallow layers, a known BP failure mode on high-entropy distributions (Nag+25)

One extra detail worth flagging: for the unsigned case, the **additional Barrett reduction with known quantities in layers 1–3 further improves the attacker's models.** Countermeasure-induced operations can themselves become leakage sources.

---

## 11. Performance (Table 2, thousands of cycles)

| Implementation | −O0 | −O3 |
|---|---|---|
| Signed-NTT | 127.02 | 26.48 |
| Unsigned-NTT | 158.00 | 36.75 |
| RNR±-NTT | 196.01 | 50.70 |
| RNR⁺-NTT | 260.52 | 84.74 |
| Signed-NTT⁻¹ | 202.04 | 42.61 |
| Unsigned-NTT⁻¹ | 270.39 | 64.91 |
| **RNR±-NTT⁻¹** | 203.19 | **42.61** |
| RNR⁺-NTT⁻¹ | 305.59 | 91.15 |

**The headline result is real and verifiable:** RNR±-NTT⁻¹ at −O3 is **42.61 kcycles, identical to the unprotected signed baseline. Exactly 0 % overhead.** Since the INTT is the operation that touches the secret key in decryption, this is the number that matters most.

**But the forward-NTT percentages don't reconcile — [derived].** Computing (new/old − 1) from Table 2:

| Comparison | Paper's text | Table 2 gives |
|---|---|---|
| RNR± vs Signed NTT, −O3 | 62.8 % | **91.5 %** |
| RNR± vs Signed NTT, −O0 | 42.7 % | **54.3 %** |
| RNR⁺ vs Unsigned NTT, −O3 | 79.0 % | **130.6 %** |
| RNR⁺ vs Unsigned NTT, −O0 | 49.0 % | **64.9 %** |
| RNR⁺ vs Unsigned NTT⁻¹, −O3 | 27.7 % | **40.4 %** |

I could not find a definition of "overhead" that reproduces the quoted percentages. Possibly the ePrint and SAC proceedings versions carry different measurements. **Quote the raw cycle counts in your slides, not the percentages.**

RNR⁺ is also notably expensive because its Barrett reduction needs a **64-bit multiplication**, which is costly on Cortex-M4. Another point in RNR±'s favour.

---

## 12. Critical assessment

**Strengths**

- The information-theoretic framing is clean, correct, and reproducible — I verified three of four headline MI numbers exactly
- Zero-overhead protection for the INTT, which is the secret-touching operation in decryption
- No fresh randomness, no extra operations — only constants and rejection-sampling bounds change
- **Composable with masking.** RNR is not a masking substitute; it layers on top. Masking handles DPA, RNR handles SPA/SASCA
- Evaluated under favourable-to-attacker conditions (synchronous sampling, perfect trace alignment, 100k profiling traces)
- Open-sourced

**Weaknesses to raise**

1. **RNR⁺ is not fully secure** — 25 % success at σ ≤ 0.1. Only RNR± survives all noise levels. And per §6, that gap may be a parameter choice rather than a property of signedness.
2. **Reference implementation only.** The paper says applying RNR to *optimised* implementations "remain[s] as directions for future work." Real deployments use hand-tuned assembly with tight range invariants; adding 2–3 bits of headroom there is much harder than in C.
3. **Only the NTT/INTT is protected.** Message decoding, the FO transform, Keccak, and the ciphertext comparison are all untouched — i.e. everything Papers 1 and 2 attack.
4. **The 32-contiguous-coefficient assumption** is a simplification of Ham+21, originally Kyber1024-only.
5. **Small η.** Heinz–Pöppelmann (HP23) warned that RNR may demask coefficients when ζ is a multiple of η. The authors report this isn't an issue for SPA at η ∈ {5,9} and suggest masking ciphertext inputs with r ∈ [0,η) if you want belt and braces — but the concern isn't fully closed.
6. **Text/table inconsistency on η⁺** (§6) and **unreconcilable performance percentages** (§11).
7. **Nothing against DPA.** RNR is explicitly complementary, not a replacement for masking.

---

## 13. The synthesis: does RNR stop Paper 2?

This is the best closing question for your seminar, and it is genuinely open.

**The case that it should.** Paper 2's Attack B, Step 2, works entirely because `+5` stores as HW 2 and `−5` stores as HW 14. That asymmetry *is* the signed-representation leak Paper 6 quantifies. If m′[i] were stored as `m′[i] + Kq` with random K, the Hamming weight would no longer determine the sign, and the sign-detector at the heart of Attack B would break.

**The case that it might not.**

- Paper 6 applies RNR to the NTT/INTT, **not** to the `PolySub` that produces m′. Extending it there needs separate range analysis.
- The message decode must ultimately produce a canonical value, so a reduction has to happen *somewhere* — and that reduction is a new leakage point. Paper 6 already observed exactly this: the extra Barrett reductions in the unsigned NTT *improved* the attacker's models.
- Paper 2's greedy solver is remarkably noise-tolerant — it still works at simulated σ = 2.0, where BP has long collapsed. RNR reduces MI from 3.561 to ~0.9 bits, roughly a 4× cut. That would raise Paper 2's trace count substantially but might not be a wall.
- The two attacks make different inference demands. SASCA needs precise values and dies without them; Paper 2's inequality solver needs only a **sign** or a loose bound, and can tolerate 16 % wrong inequalities (as demonstrated in its shuffling attack).

**The honest answer:** RNR attacks the right mechanism, but the attacks have very different noise tolerances, and nobody has run the experiment. That is a legitimate open problem to hand the audience.

---

## 14. Links to the rest of the reading list

| Paper | Connection |
|---|---|
| **1 (RRCB20)** | **Orthogonal.** PC-oracle attacks target the FO hash; RNR protects the NTT. RNR offers no defence here. Says something important: no single countermeasure covers the decapsulation |
| **2 (Defeating LCC)** | **Deepest link.** Paper 2 exploits the signed-representation HW asymmetry; Paper 6 quantifies and removes it. See §13 |
| **3 (PQ-Hammer)** | Orthogonal — fault injection on the `fail` variable, unaffected by any arithmetic-representation change |
| **4, 5 (shuffling)** | **Direct competitors.** RNR is pitched as the low-cost alternative. Comparison axes: overhead (RNR: 0 % INTT vs shuffling's 7–78 %), implementation effort (RNR: constants only), and whether shuffling defends anything RNR doesn't |

---

## 15. Slide skeleton (~6 min)

| # | Slide | Message |
|---|---|---|
| 1 | Threat model shift: single-trace SASCA on the NTT | Different attack class from Papers 1–5 |
| 2 | `+5` → HW 2, `−5` → HW 14 | Callback to Paper 2 — same phenomenon, now measured |
| 3 | The MI table: 3.561 / 2.756 / 0.906 / 1.219 | Representation determines leakage |
| 4 | `I = H[W] − H[W\|X]`, and H[W\|X] = 0 without redundancy | Why you must add encodings, not just reshuffle bits |
| 5 | RNR: compute in Z_{ηq}, `X′ = X + Kq` | No randomness during compute, no extra ops |
| 6 | η bounds: 9.72 signed, 9.11 unsigned → η± = 9, η⁺ = 5 | And note the text/table conflict |
| 7 | The amplification effect | The attacker's own sparsity creates stronger templates |
| 8 | Simulated + practical results | RNR± ineffective at all noise; PI ≈ 0 in shallow layers |
| 9 | **42.61 vs 42.61 kcycles — 0 % INTT overhead** | The headline; quote raw cycles, not percentages |
| 10 | Does RNR stop Paper 2? | Closing discussion question |

---

## 16. Self-check exercises

1. Why is `H[W(X) | X] = 0` for both signed and unsigned Z_q? *(W is a deterministic function of the stored value; each residue has exactly one encoding.)*
2. Why does signed leak more than unsigned when q ≪ 2^l? *(Two's complement sets the whole upper region of bits for negatives, so W(X) reveals the sign — roughly one extra bit — and spreads the weight distribution more evenly, raising H[W].)*
3. RNR raises per-coefficient entropy from 11.701 to 14.871 bits. Is that good or bad for the defender? *(Good, twice over: the attacker must eliminate more entropy AND learns less per observation.)*
4. Why is the INTT free while the forward NTT costs 60–90 %? *(The forward NTT needs two extra Barrett reductions per butterfly to maintain the range invariant across layers; for the signed INTT a fixpoint argument shows the Montgomery output needs no extra reduction, so only constants change.)*
5. Given the bound permits η⁺ < 9.11, why might the authors have picked 5? *(Unstated. Note that η⁺ = 9 would give I = 0.835 bits, better than RNR±'s 0.906 — which would flip the paper's recommendation.)*
6. Why does the k-trace attack's ciphertext sparsity help the *attacker's profiling*, not just the attack? *(Zeroed intermediates are known with certainty and propagate unchanged or inverted through GS butterflies, so the same value recurs across layers — the amplification effect. Templates drop from 1792 to 864.)*
7. Could RNR be the fix for Paper 2's Attack B? *(It attacks the right mechanism — the signed HW asymmetry — but Paper 6 only applies it to the NTT, the final reduction to canonical form is a new leakage point, and Paper 2's solver tolerates far more noise than BP does. Open problem.)*
