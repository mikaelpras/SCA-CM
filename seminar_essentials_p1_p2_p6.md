# Seminar Essentials — Papers 1, 2, 6 (8 minutes each)

Condensed from the full deep dives. Attack papers answer seven questions; the countermeasure paper answers five.
Hardware-IP relevance is called out throughout, since the source papers are all Cortex-M4 software.

---
---

# PAPER 1 — Generic SCA on CCA-secure Lattice PKE/KEMs

Ravi, Sinha Roy, Chattopadhyay, Bhasin — TCHES 2020(3) 307–335 · ePrint 2019/948

> **Thesis:** The FO transform must hash the decrypted message in order to check it. That hash happens *before* the rejection, so the value the rejection was supposed to hide has already leaked. IND-CCA security is defeated by observing, not by querying.

## 1. Existing SCA method it builds on

| Prior work | What it did | Limitation |
|---|---|---|
| Key-mismatch attacks (Fluhrer, Ding, Băetu, Qin) | CCA on **IND-CPA** schemes assuming a plaintext-checking (PC) oracle | FO transform removes the oracle |
| D'Anvers et al. 2019 | **Timing** leak in variable-time ECC decoding (LAC, RAMSTAKE) | Killed by constant-time ECC |
| **This paper** | Instantiates the PC oracle from **EM leakage** | Works on constant-time code; algorithm-level |

## 2. Attack model

- Physical access to a device decapsulating with a **long-term (static) secret key**
- Unlimited chosen-ciphertext queries; **passive** EM observation
- **No knowledge of the output** — never sees the shared key or the accept/reject decision
- **No clone device, no known-key profiling.** Templates are built on the target itself by setting `k_u = 0`, which forces `u·s = 0` regardless of the key, so `k_v` alone dictates the decrypted message

## 3. Attack points

| # | Leaky operation | Applies to |
|---|---|---|
| **A** | **ECC decoding** — XEf majority logic / BCH syndrome. Register set is zero for a valid codeword, non-zero otherwise | Round5(E), LAC |
| **B** | **`G(m′, pk)` inside the FO transform** — line 9 of `Decaps` | **Kyber**, Saber, NewHope, Frodo, Round5(NE) |

**For Kyber, point B is the whole attack.** G = SHA3-512 over 64 bytes = one Keccak-f[1600] permutation. Sample the *late* rounds: a 1-bit input difference becomes a ~50 % state difference by round 24. **The hash's own avalanche is the attacker's amplifier.**

## 4. Attack procedure (Kyber512)

1. **Craft** `u₀[0] = k_u`, `v[0] = k_v`, all else zero → `m′[0] = Poly_to_Msg(k_v − k_u·s₀[0])`
2. **Silence the rest:** require `2·k_u ≤ 832` (i.e. `k_u ≤ 416`) so all other message bits stay 0 for any secret. Only two possible messages ever reach G: `00…0` or `01 00…0`
3. **Profile on target** (`k_u = 0`), 50 traces per class, Welch t-test → PoIs → reduced templates
4. **Query** 5 chosen `(k_u, k_v)` pairs per coefficient; classify each trace O/X; look up the decision table
5. **Sweep** `u[p] = k_u·x^q` over `p ∈ [0,k)`, `q ∈ [0,n)` — anti-cyclic rotation walks all 512 coefficients (**note: returns negated values for q ≥ 1**)
6. **Repeat 3×** and majority-vote to absorb oracle errors

## 5. Target device & results

STM32F407 Cortex-M4 @ 24 MHz · **pqm4** optimized code · near-field EM probe · LeCroy 610Zi @ 500 MSa/s

| Scheme | Traces | Time | Success/iter |
|---|---|---|---|
| **Kyber512** | 2560 queries → **7.7k traces** | **10.8 min** | 99 % |
| Round5 | 978 → 2.9k | 4.5 min | 99 % |
| LAC128 | 1024 → 3.0k | 25 min | 97 % |

**Say this honestly:** the title claims six schemes; **only three were measured**. Saber rides on structural similarity, NewHope is simulation with a perfect oracle, Frodo is an appendix sketch with no numbers.

## 6. What must be protected

**Everything downstream of decryption.** Masking only the NTT/pointwise multiply achieves nothing — the leak is after it.

1. The decrypted message `m′` — from `Poly_to_Msg` output onward
2. **`G` / Keccak**, operating on shared `m′`
3. The re-encryption
4. The ciphertext comparison (and it must not leak pass/fail early)

## 7. Possible countermeasures

| Countermeasure | Verdict |
|---|---|
| Constant-time ECC | Insufficient — this paper's starting point |
| Masking NTT only | **Useless** — wrong side of the leak |
| **Full masked decapsulation incl. masked Keccak** | Works. The paper's only recommendation; it concedes the cost |
| Masked A2B for `Poly_to_Msg` | Required, and **hard for Kyber because q = 3329 is prime** (Saber's power-of-two moduli are much friendlier) |
| Shuffling | Raises the bar only → Papers 4/5 |
| Detection of malicious ciphertexts | → **Paper 2 breaks it** |

## → Relevance to hardware IP

- The leak is the **message register feeding the Keccak state**, not the polynomial arithmetic. A Kyber/ML-KEM hardware IP must mask from `poly_tomsg` through SHA3 through the comparator.
- **Don't expect hardware to save you.** A 1600-bit Keccak state register aggregates HW and lowers SNR — but the distinguisher is only a **2-class problem** (m=0 vs m=1), so low SNR barely matters.
- 2-share Keccak is tractable in RTL (χ is the only nonlinear step), roughly 2× area. This is where SHA-512/SHA-3 datapath experience transfers directly.

## 8-minute slide plan

1. FO `Decaps` pseudocode, line 9 boxed — "the hash happens before the rejection" *(1 min)*
2. Chosen ciphertext + the `k_u ≤ 416` constraint — only two messages reach G *(1.5 min)*
3. Decision table for Kyber512, 5 columns *(1.5 min)*
4. t-test + reduced template, and the `k_u = 0` self-profiling trick *(1 min)*
5. Anti-cyclic sweep → all 512 coefficients *(1 min)*
6. Results table + the "3 of 6 measured" caveat *(1 min)*
7. Countermeasure verdict → hand off to Paper 2 *(1 min)*

**Land this line:** *"Masking the NTT protects the wrong operation. The secret leaves through the hash that proves the ciphertext was honest."*

---
---

# PAPER 2 — Defeating Low-Cost Countermeasures

Ravi, Paiva, Jap, D'Anvers, Bhasin — TCHES 2024(2) 795–818 · ePrint 2023/1627

> **Thesis:** Both cheap detection countermeasures fail. One is bypassed by masking the attack ciphertext with the public key; the other by a new attack that uses **only valid ciphertexts**, so no decapsulation failure ever occurs.

*(Note the authorship: Ravi wrote Paper 1, co-proposed the sanity check, and now breaks it.)*

## 1. Existing SCA method it builds on

- **PC-oracle CC attacks** (Paper 1; Rajendran et al. TCHES 2023 for the Kyber768 4-query table)
- **Known-ciphertext SCA** on the polynomial multiply (Primas/Pessl, Mujdei et al.)
- **Inequality solvers** from *fault* attacks — Belief Propagation of Pessl–Prokop, Hermelink et al., Delvaux. This paper replaces BP with a greedy solver

**The countermeasures under attack:**

| Countermeasure | Mechanism | Why it looked good |
|---|---|---|
| **Ciphertext sanity check** | Reject low-variance ciphertexts (attack CTs have 1 non-zero coeff in 768) | ~5 % overhead, rejects **before** decapsulation so no leakage at all |
| **Decapsulation failure check** | Refresh the key on the **first** failure | Free, protocol-level, caps attacker at **one trace** |

## 2. Attack model

- Physical access, power/EM, **long-term key**
- **All ciphertexts are VALID** — the attacker encrypts honestly and therefore knows `m, r, e₁, e₂, Δu, Δv`
- **Clone device required** for profiling (weaker assumption than Paper 1 on this axis)

## 3. Attack points

**Attack A (vs sanity check):** unchanged — same PC-oracle point as Paper 1.

**Attack B (vs failure check):** the **store of `m′` coefficients** in `PolySub`, computing `m′ = v − ⟨s,u⟩`. Leakage model = **Hamming weight of the stored signed 16-bit value**.

Because ciphertexts are valid, only the *decryption* procedure is exploitable — the FO transform is off-limits.

## 4. Attack procedure

### Attack A — public-key masking (2 slides, no measurements needed)

Send `(u_atk + a*, v_atk + b*)` instead of `(u_atk, v_atk)`. The `A·s` terms cancel:

```
Δ = (v_atk + b*) − (u_atk + a*)·s = v_atk − u_atk·s + e
```

Ciphertext now looks like a genuine LWE sample. **Downstream attack is unchanged.** Success 57.8 %; on failure, retry with a different row of A, a rotation `(X·a*, X·b*)`, or a multiple `(c·a*, c·b*)`. Expected restarts **0.73**.

> **Why only 57.8 %?** *(my analysis — the paper doesn't explain it)* Every column of the Kyber768 decision table contains exactly one cell whose value is **832**, one unit below the decode threshold 833. For that candidate the oracle is decided purely by **the sign of the added error E** — a coin flip. Hence ~58 %, and hence "just retry."

### Attack B — valid-ciphertext key recovery (the centrepiece)

**The pivot:** for a valid ciphertext, `m′ = ⌊q/2⌉·m + Δm` where

```
Δm = ⟨r,e⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv       ← LINEAR in (s, e); everything else is KNOWN
```

**Step 1 — leak `HW(m′[i])`.** CPA → PoIs → **Random Forest** (1.5k trees, 17 HW classes).

**Step 2 — two's complement asymmetry (THE key idea):**

```
Δm[i] = +5  →  0000 0000 0000 0101   HW = 2    (low)
Δm[i] = −5  →  1111 1111 1111 1011   HW = 14   (high)
```

So when `m[i] = 0`, **HW is a near-perfect sign detector for Δm[i]**. When `m[i] = 1`, `m′[i] ≈ 1664 ± Δm` — no sign boundary crossed, **no information**. ⟹ only the ~128 zero bits per message are usable.

**Step 2b —** upgrade sign → two-sided bounds using an empirical min/max-per-HW table. Bounds must be **measured, not derived**, because lazy modular reduction breaks the expected ranges (for HW = 10 you'd predict max = −64, but +253 is observed, since 253 + q also has HW 10). Filter keeps only **extreme** Hamming weights (HW ∈ {0,1,12,…,16}).

**Step 3 — solve.** Two inequalities per observation → `Hx + w ≥ 0`. **Greedy solver:** start at x = 0; each iteration score every action (add v to coordinate j) by the total distance from satisfying all inequalities; apply the best α_it, decaying α exponentially.

## 5. Target device & results

STM32F407 Cortex-M4 @ 24 MHz · Langer RF-U 5-2 EM probe · LeCroy 610Zi @ **1.25 GSa/s** · 30 dB pre-amp, 48 MHz LPF · Kyber768 · 10 keys

| Setting | HW accuracy | Traces |
|---|---|---|
| Reference `-O3` | 91 % | **5200 measured** (8 zero bits) |
| Reference `-O3` | 91 % | **325 extrapolated** (128 zero bits) |
| Assembly-optimized | **32 %** | **7800** — 24× worse |
| Shuffled | *assumed perfect* | **> 8 000 000** |
| First-order masked (mkm4) | 94 % | ~10⁵ |

**Greedy vs BP:** < half the inequalities · usable to σ ≈ 2.0 (BP dies past 0.5) · **< 20 s vs > 10 min**.

**Caveats to state:** 325 is extrapolated, not measured. The masked evaluation replaced mkm4's assembly with C at **`-O0`**, which is why masked accuracy (94 %) *beats* unprotected-optimized (32 %) — a compiler-flag artefact, not a masking result.

## 6. What must be protected

1. **The `m′` register/memory write** in the subtraction — the new attack point
2. **All arithmetic shares** of `m′` when masked — each share's HW leaks independently
3. **Representation of `m′`** — the two's-complement sign asymmetry is the enabler → Paper 6

## 7. Possible countermeasures

| Countermeasure | Verdict |
|---|---|
| Ciphertext sanity check | **BROKEN** — public-key masking, ~zero cost |
| Decapsulation failure check | **BROKEN** — valid ciphertexts never fail |
| "Reject if close to the public key" | **BROKEN** — use −A, 2A, or X·A |
| Shuffling | Costs the attacker ~4.5 orders of magnitude (325 → 8 M). Real speed bump, not a wall |
| First-order masking | Broken as evaluated (caveat above); shares leak individually |
| **Wider parallel datapath** | *(see below)* — genuine attenuation |
| **Change the number representation** | → **Paper 6** |

## → Relevance to hardware IP

- **Concrete design lever:** the paper's own numbers show that storing **1** coefficient per write gives 91 % classifier accuracy, **10** coefficients per write gives **32 %**. A hardware datapath processing P coefficients per cycle aggregates their Hamming weights and drops per-coefficient SNR ≈ 1/P. **Parallelism is a partial, free mitigation.** But 32 % still worked, so it raises cost rather than closing the hole.
- **Do not implement detection countermeasures in an IP.** Both are broken at essentially zero attacker cost; you would pay area and gain nothing.
- **The greedy solver is the portable threat.** It converts *any* leakage yielding linear inequalities in (s,e) into key recovery, tolerates 16 % wrong inequalities, and is cheap. Assume it exists when threat-modelling.
- If your datapath stores coefficients in signed two's complement, **the same asymmetry is present in RTL.** This is the direct hook into Paper 6.

## 8-minute slide plan

1. Masking is expensive → the two cheap detection countermeasures *(1 min)*
2. Public-key masking: 4 lines of algebra, `A·s` cancels *(1 min)*
3. Margin table → "57.8 % is a coin flip, so just retry" *(1 min)*
4. The bind: valid ciphertexts ⟹ decryption only *(0.5 min)*
5. **`Δm` is linear in (s,e) and everything else is known** *(1 min)*
6. **`+5` → HW 2, `−5` → HW 14** *(1.5 min — the best slide in the paper)*
7. Inequalities → greedy solver vs BP *(1 min)*
8. Results + 4 caveats + verdict *(1 min)*

**Land this line:** *"Detection assumes the attacker needs invalid ciphertexts. Once he doesn't, the countermeasure has nothing to detect."*

---
---

# PAPER 6 — RNR as an SPA Countermeasure

Nagpal, Hadžić, Primas, Mangard — **SAC 2025**, LNCS 16207, 753–780 · ePrint 2025/679

> **Thesis:** When q ≪ 2^word, the *representation* leaks. Compute in Z_{ηq} instead of Z_q so each residue has η machine encodings; the Hamming weight then no longer determines the value. Zero overhead for the INTT.

**Different threat class from Papers 1–2:** single-trace profiled **SASCA** (factor graph + loopy belief propagation) on the **NTT⁻¹**, not multi-trace chosen-ciphertext.

## 1. Protection point

**The NTT / NTT⁻¹ only** — specifically the machine representation of every polynomial coefficient and butterfly intermediate.

**Scope limit to state clearly:** RNR does **nothing** against Paper 1 (FO-transform hash) and is untested against Paper 2 (the `m′` store). Different attack surface, different defence.

## 2. Key idea

```
I[X ; W(X)]  =  H[W(X)]  −  H[W(X) | X]
```

Without redundancy, `H[W|X] = 0` — the Hamming weight is a **deterministic function** of the stored value, so *all* of H[W] leaks. You cannot fix this by reshuffling bits; you must **manufacture conditional entropy** by giving each residue multiple encodings.

**Verified figures** (X uniform in Z_3329, 16-bit word) — I recomputed these:

| Representation | H[W] | H[W\|X] | **I[X;W(X)]** |
|---|---|---|---|
| **Signed** Z_q | 3.561 | 0 | **3.561** |
| Unsigned Z_q | 2.756 | 0 | **2.756** |
| **RNR± (η=9, signed)** | 3.149 | 2.243 | **0.906** |
| RNR⁺ (η=5, unsigned) | 2.958 | 1.739 | **1.219** |

Two effects, both helping the defender: RNR **raises** entropy per coefficient (11.70 → 14.87 bits) while **lowering** leakage (3.56 → 0.91 bits).

**Note the signed→unsigned gap alone is ~0.8 bits/coefficient ≈ 206 bits per polynomial — free.** The fast signed arithmetic everyone uses for performance is measurably less secure.

## 3. Protection steps

1. Start from an algorithm over Z_q
2. **Find the largest η** such that no operation overflows in Z_{ηq}. Kyber NTT butterfly: `η± < 2³³/(2¹⁸q + q²) = 9.72`, `η⁺ < 9.11` → **η± = 9, η⁺ = 5**
3. **Encode** inputs as `X′ = X + Kq`, K random in `[0,η)` (unsigned) or `[−η/2, η/2]` (signed)
4. **Compute entirely over Z_{ηq}**; add Barrett reductions where the range invariant would break
5. **Decode** outputs by reducing mod q

**Implementation delta is tiny:** rejection-sampling bounds in `SamplePolyCBD`/`GeneratePolyMatrix`, plus reduction constants. **No fresh randomness during computation, no extra operations.**

*(Erratum: the text says "η⁺ := 9" but Table 1 says 5. Table 1 is correct — log₂(5q) = 14.023 matches the reported entropy, and 39987 = −(5q)⁻¹ mod 2¹⁶ matches the Montgomery constant.)*

## 4. Results

Simulated SASCA (7 GS layers, 1792 factors, 25 BP iterations):

| | Behaviour |
|---|---|
| Signed | 100 % success until σ ≥ 3.0 |
| Unsigned | degrades from σ ≥ 1.6 |
| **RNR±** | **ineffective at every noise level** |
| RNR⁺ | ~25 % at σ ≤ 0.1 |

Practical: ChipWhisperer CW308 + **STM32F303**, Picoscope 6404C, 20 dB LNA, 10 MHz **synchronised** sampling, 100 k profiling traces, LDA via SCALib. RNR drops **SNR by two orders of magnitude**; PI ≈ 0 in shallow layers; BP cannot improve guessing entropy.

## 5. Advantages / disadvantages

| Advantages | Disadvantages |
|---|---|
| **INTT: 42.61 → 42.61 kcycles = 0 % overhead** (verified) | Forward NTT costs +54 % (−O0) / **+91 %** (−O3) from Table 2 — the text's "62.8 %" does not reconcile |
| **No randomness during compute** — no PRNG, no mask refresh | **RNR⁺ is not secure** (25 % at low noise); only RNR± survives |
| **Composes with masking** (masking→DPA, RNR→SPA) | **Reference C only** — optimized assembly left as future work |
| Constants-only change; trivial to retrofit | Protects **only** the NTT/INTT |
| Evaluated under attacker-favourable conditions (sync sampling) | Heinz–Pöppelmann demasking concern at small η not fully closed |
| Cheaper than shuffling (Ravi et al.: 7–78 %) | Nothing against DPA |

**Open question worth raising:** the bound permits η⁺ = 9, which I compute would give **I = 0.835 bits — better than RNR±'s 0.906**. That would invert the paper's recommendation. Is RNR± genuinely better, or just better-parameterised?

## 6. → Can the idea be applied in our IP?

**Yes, and the fit is better in hardware than in software.** Four points:

**(a) Hardware removes the paper's main cost.** The software overhead comes from extra Barrett reductions in the instruction stream. In RTL, a reduction is a pipeline stage, not a loop iteration — the cost moves from cycles to **area and width**:

| | Kyber | ML-DSA |
|---|---|---|
| ⌈log₂ q⌉ | 12 bits | 23 bits |
| ⌈log₂ ηq⌉ | **15 bits** (η=9) | **31 bits** (η≈256) |
| Multiplier area scaling (≈ quadratic) | ≈ 1.6× | ≈ 1.8× |

Registers and RAM grow linearly; the multiplier dominates. **RNR trades area for SPA resistance and buys zero randomness** — the opposite trade from masking, which costs PRNG bandwidth and mask-refresh routing.

**(b) A design decision the paper never raises: in hardware you choose the word width.** If you size the datapath to exactly 12 bits for Kyber, there is no redundant space, and the representation leak largely vanishes **for free** — but RNR becomes impossible. If you are already at 16 bits (bus alignment, IP reuse), RNR is nearly free in registers. **Decide this before freezing the datapath, not after.**

**(c) ML-DSA is a *better* RNR target than ML-KEM — [my computation]:**

| | ML-KEM (q=3329, 16-bit) | **ML-DSA (q=8380417, 32-bit)** |
|---|---|---|
| Signed leak I[X;W(X)] | 3.561 bits | **4.215 bits** (worse!) |
| Max η | 9 → 3.17 bits redundancy | **≈256 → 8.0 bits** |
| **RNR± residual leak** | 0.906 bits | **0.226 bits** |

ML-DSA's q sits far lower relative to its 32-bit word (2⁻⁹ vs 2⁻⁴·³), so there is much more redundant space. RNR should work **~4× better on ML-DSA than on ML-KEM**, and ML-DSA needs it more.

**(d) Tension with the narrow-ring sparse-multiplier concept.** RNR deliberately *widens* the ring; computing challenge–secret products in a narrow Z_M deliberately *narrows* it. These pull opposite ways on the same metric, and which wins depends on M relative to the datapath width — a narrow ring leaks fewer absolute bits but may leak a larger *fraction* of the secret's entropy.

**The genuinely transferable asset here is the evaluation method, not RNR itself:** compute `I[X;W(X)] = H[W(X)] − H[W(X)|X]` for your chosen M and register width. It is a few lines of code, needs no measurements, and tells you whether the representation is leaking before you tape out. RNR also **layers on top of masking**, so it is additive to the first-order masked design rather than an alternative.

## 8-minute slide plan

1. Threat model shift: single-trace SASCA on the NTT *(1 min)*
2. **`+5` → HW 2, `−5` → HW 14** — callback to Paper 2, now measured *(1 min)*
3. MI table: 3.561 / 2.756 / 0.906 / 1.219 *(1 min)*
4. `I = H[W] − H[W|X]`, and why `H[W|X] = 0` without redundancy *(1.5 min)*
5. RNR: compute in Z_{ηq}, `X′ = X + Kq`; η bounds → 9 and 5 *(1.5 min)*
6. Results: RNR± ineffective at all σ; SNR down 100× *(1 min)*
7. **42.61 vs 42.61 kcycles**, and the disadvantages column *(1 min)*
8. Applicability to hardware + "does this stop Paper 2?" *(1 min)*

**Land this line:** *"Masking hides the value. RNR hides the fact that the value determines its own Hamming weight — and it needs no randomness to do it."*

---
---

# Cross-paper closing slide (worth 1 minute at the end)

| Attack surface | Paper 1 | Paper 2 | Paper 6 |
|---|---|---|---|
| FO transform hash `G(m′)` | **attacked** | — | not covered |
| Message polynomial `m′` store | — | **attacked** | untested |
| NTT / NTT⁻¹ | — | — | **protected** |
| Ciphertext detection | — | **broken** | — |

**Three things no single countermeasure covers.** Masking the NTT misses Paper 1. Detection misses everything. RNR misses the FO transform. A real IP needs masking *and* representation hygiene *and* a masked hash — and Paper 2's greedy solver means partial leakage reduction may only raise the trace count, not close the hole.

**The open question to hand the audience:** Paper 2's attack works *because* of the signed representation Paper 6 fixes. Nobody has run that experiment.
