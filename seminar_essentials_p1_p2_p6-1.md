# Seminar Essentials — Papers 1, 2, 6 (8 minutes each)

Each paper is treated as a standalone topic. Attack papers answer seven questions; the countermeasure paper answers five.
Every paper opens with a "what this paper is about" summary slide.

---
---

# PAPER 1 — Generic Side-Channel Attacks on CCA-secure Lattice-based PKE and KEMs

Ravi, Sinha Roy, Chattopadhyay, Bhasin — TCHES 2020(3), 307–335 · ePrint 2019/948

## SLIDE 1 — What this paper is about

**The problem.** IND-CCA security was believed to block chosen-ciphertext attacks on lattice KEMs. The Fujisaki–Okamoto (FO) transform re-encrypts the decrypted message and rejects anything that doesn't match, so the attacker learns nothing from the output.

**What they do.** They show the FO transform *must hash the decrypted message* in order to check it — and that hash leaks over EM. That gives them a plaintext-checking (PC) oracle, which resurrects the chosen-ciphertext attacks FO was supposed to prevent.

**What they find.** Full key recovery on six NIST Round-2 lattice KEMs. Kyber512 falls in ~11 minutes with 7.7k EM traces on a Cortex-M4.

**Why it matters.** The attack is **algorithm-level, not implementation-level**. It doesn't care how the NTT is written, whether the code is constant-time, or which compiler was used. Rewriting the arithmetic does nothing.

> **One-sentence thesis:** the hash that proves a ciphertext was honest leaks the secret before the rejection ever happens.

## 1. Existing SCA method it builds on

| Prior work | What it did | Limitation |
|---|---|---|
| Key-mismatch attacks (Fluhrer, Ding, Băetu, Qin) | CCA on **IND-CPA** schemes, assuming a PC oracle exists | FO transform removes the oracle |
| D'Anvers et al. 2019 | **Timing** leak in variable-time ECC decoding (LAC, RAMSTAKE) | Killed by constant-time ECC |
| **This paper** | Instantiates the PC oracle from **EM leakage** | Works on constant-time code |

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
2. **Silence the other 255 bits:** require `2·k_u ≤ 832` (i.e. `k_u ≤ 416`) so they decode to 0 for any secret. Only two messages ever reach G: `00…0` or `01 00…0`
3. **Profile on target** (`k_u = 0`), 50 traces per class → Welch t-test → PoIs → reduced templates
4. **Query** 5 chosen `(k_u, k_v)` pairs per coefficient; classify each trace O/X; look up the decision table
5. **Sweep** `u[p] = k_u·x^q` over `p ∈ [0,k)`, `q ∈ [0,n)` — anti-cyclic rotation reaches all 512 coefficients (**returns negated values for q ≥ 1**)
6. **Repeat 3×**, majority-vote to absorb oracle errors

## 5. Target device & results

STM32F407 Cortex-M4 @ 24 MHz · **pqm4** optimized code · near-field EM probe · LeCroy 610Zi @ 500 MSa/s

| Scheme | Traces | Time | Success/iter |
|---|---|---|---|
| **Kyber512** | 2560 queries → **7.7k traces** | **10.8 min** | 99 % |
| Round5 | 978 → 2.9k | 4.5 min | 99 % |
| LAC128 | 1024 → 3.0k | 25 min | 97 % |

**State honestly:** the title claims six schemes; **only three were measured**. Saber rides on structural similarity, NewHope is simulation with a perfect oracle, Frodo is an appendix sketch with no numbers.

## 6. What must be protected

**Everything downstream of decryption.** Masking only the NTT/pointwise multiply achieves nothing — the leak is after it.

1. The decrypted message `m′`, from `Poly_to_Msg` output onward
2. **`G` / Keccak**, operating on shared `m′`
3. The re-encryption
4. The ciphertext comparison — and it must not leak pass/fail early

## 7. Possible countermeasures for Kyber

| Countermeasure | Verdict |
|---|---|
| Constant-time ECC | Insufficient — this paper's starting point |
| Masking NTT only | **Useless** — wrong side of the leak |
| **Full masked decapsulation incl. masked Keccak** | Works. The paper's only recommendation; it concedes the cost |
| Masked A2B for `Poly_to_Msg` | Required, and **hard for Kyber because q = 3329 is prime** (power-of-two moduli are far friendlier) |
| Shuffling | Raises the bar, does not close the hole |
| Detection of malicious ciphertexts | Attractive on cost; subsequently shown breakable |

### Hardware-IP note

The leak is the **message register feeding the Keccak state**, not the polynomial arithmetic — so a Kyber IP must mask from `poly_tomsg` through SHA3 through the comparator. Don't expect hardware to help: a 1600-bit state register aggregates Hamming weight and lowers SNR, but the distinguisher is only a **2-class problem** (m = 0 vs m = 1), so low SNR barely matters. 2-share Keccak is tractable in RTL — χ is the only nonlinear step — at roughly 2× area.

## 8-minute slide plan

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** | 0:45 |
| 2 | FO `Decaps` pseudocode, line 9 boxed — hash happens before rejection | 1:00 |
| 3 | Chosen ciphertext + `k_u ≤ 416` → only two messages reach G | 1:15 |
| 4 | Decision table for Kyber512 (5 columns) | 1:15 |
| 5 | t-test + reduced template; the `k_u = 0` self-profiling trick | 1:00 |
| 6 | Anti-cyclic sweep → all 512 coefficients | 0:45 |
| 7 | Results + "3 of 6 measured" caveat | 1:00 |
| 8 | What must be protected / countermeasure verdict | 1:00 |

**Land this line:** *"Masking the NTT protects the wrong operation. The secret leaves through the hash that proves the ciphertext was honest."*

---
---

# PAPER 2 — Defeating Low-Cost Countermeasures against SCA in Lattice-based Encryption

Ravi, Paiva, Jap, D'Anvers, Bhasin — TCHES 2024(2), 795–818 · ePrint 2023/1627

## SLIDE 1 — What this paper is about

**The problem.** Masking a full Kyber decapsulation is expensive, so researchers proposed two cheap alternatives that *detect* attack ciphertexts instead of hiding data: a **ciphertext sanity check** (reject low-entropy ciphertexts) and a **decapsulation failure check** (refresh the key on the first failure). Nobody had evaluated how much protection they actually buy.

**What they do.** Break both. The sanity check falls to masking the attack ciphertext with the public key. The failure check falls to a genuinely new attack — the **first chosen-ciphertext SCA that uses only valid ciphertexts**, so no decapsulation failure is ever triggered.

**What they find.** Key recovery on Kyber768 in 325–7800 traces. They also contribute a **greedy inequality solver** that beats Belief Propagation with less than half the inequalities and ~30× faster.

**Why it matters.** Low-cost detection countermeasures give **no standalone protection**. If you build one, you pay the area and gain nothing.

> **One-sentence thesis:** detection assumes the attacker needs invalid ciphertexts — once he doesn't, there is nothing to detect.

## 1. Existing SCA method it builds on

- **PC-oracle chosen-ciphertext SCA** — the standard binary-oracle attack on Kyber decapsulation (Ravi et al. 2020; Rajendran et al. TCHES 2023 supplies the Kyber768 4-query table)
- **Known-ciphertext SCA** on the polynomial multiply (Primas/Pessl, Mujdei et al.)
- **Inequality solvers** imported from *fault* attacks — Belief Propagation of Pessl–Prokop, Hermelink et al., Delvaux

**The two countermeasures under attack:**

| Countermeasure | Mechanism | Why it looked good |
|---|---|---|
| **Ciphertext sanity check** | Reject low-variance ciphertexts (attack CTs have 1 non-zero coeff in 768) | ~5 % overhead, rejects **before** decapsulation so no leakage at all |
| **Decapsulation failure check** | Refresh the key on the **first** failure | Free, protocol-level, caps attacker at **one trace** |

## 2. Attack model

- Physical access, power/EM measurement, **long-term key**
- **All ciphertexts are VALID** — the attacker encrypts honestly and therefore knows `m, r, e₁, e₂, Δu, Δv`
- **Clone device required** for profiling

## 3. Attack points

**Attack A (vs sanity check):** unchanged — the standard PC-oracle point in the FO transform.

**Attack B (vs failure check):** the **store of `m′` coefficients** in `PolySub`, computing `m′ = v − ⟨s,u⟩`. Leakage model = **Hamming weight of the stored signed 16-bit value**.

Because the ciphertexts are valid, re-encryption succeeds and only the **decryption** procedure is exploitable.

## 4. Attack procedure

### Attack A — public-key masking

Send `(u_atk + a*, v_atk + b*)` instead of `(u_atk, v_atk)`. The `A·s` terms cancel:

```
Δ = (v_atk + b*) − (u_atk + a*)·s = v_atk − u_atk·s + e
```

The ciphertext now looks like a genuine LWE sample and passes the entropy test. **Downstream attack is completely unchanged.** Success 57.8 %; on failure retry with a different row of A, a rotation `(X·a*, X·b*)`, or a multiple `(c·a*, c·b*)`. Expected restarts **0.73**.

> **Why only 57.8 %?** *(my analysis — the paper doesn't explain it)* Every column of the Kyber768 decision table contains exactly one cell whose value is **832**, one unit below the decode threshold of 833. For that candidate the oracle is decided purely by **the sign of the added error E** — a coin flip. Hence ~58 %, and hence "just retry."

### Attack B — valid-ciphertext key recovery (the centrepiece)

**The pivot.** For a valid ciphertext, `m′ = ⌊q/2⌉·m + Δm` where

```
Δm = ⟨r,e⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv       ← LINEAR in (s, e); everything else is KNOWN
```

**Step 1 — leak `HW(m′[i])`.** CPA → PoIs → **Random Forest** classifier (1.5k trees, 17 HW classes).

**Step 2 — two's complement asymmetry (THE key idea):**

```
Δm[i] = +5  →  0000 0000 0000 0101   HW = 2    (low)
Δm[i] = −5  →  1111 1111 1111 1011   HW = 14   (high)
```

When `m[i] = 0`, **HW is a near-perfect sign detector for Δm[i]**. When `m[i] = 1`, `m′[i] ≈ 1664 ± Δm` — no sign boundary is crossed, so **no information**. Consequence: only the ~128 zero bits per message are usable.

**Step 2b — sign → two-sided bounds** using an empirical min/max-per-HW table. Bounds must be **measured, not derived**, because lazy modular reduction breaks the expected ranges: for HW = 10 you'd predict max = −64, but +253 is observed, since 253 + q also has HW 10. The filter keeps only **extreme** Hamming weights (HW ∈ {0,1,12,…,16}).

**Step 3 — solve.** Two inequalities per observation → `Hx + w ≥ 0`. **Greedy solver:** start at x = 0; each iteration, score every action (add v to coordinate j) by total distance from satisfying all inequalities; apply the best α_it; decay α exponentially.

## 5. Target device & results

STM32F407 Cortex-M4 @ 24 MHz · Langer RF-U 5-2 EM probe · LeCroy 610Zi @ **1.25 GSa/s** · 30 dB pre-amp, 48 MHz LPF · Kyber768 · 10 keys recovered

| Setting | HW accuracy | Traces |
|---|---|---|
| Reference `-O3` | 91 % | **5200 measured** (8 zero bits) |
| Reference `-O3` | 91 % | **325 extrapolated** (128 zero bits) |
| Assembly-optimized | **32 %** | **7800** — 24× worse |
| Shuffled | *assumed perfect* | **> 8 000 000** |
| First-order masked (mkm4) | 94 % | ~10⁵ |

**Greedy vs BP:** < half the inequalities · usable to σ ≈ 2.0 (BP dies past 0.5) · **< 20 s vs > 10 min**.

**Caveats to state:** 325 is extrapolated, not measured. The masked evaluation replaced mkm4's assembly with C at **`-O0`**, which is why masked accuracy (94 %) *exceeds* unprotected-optimized (32 %) — a compiler-flag artefact, not a masking result.

## 6. What must be protected

1. **The `m′` register/memory write** in the subtraction — the new attack point
2. **All arithmetic shares** of `m′` when masked — each share's HW leaks independently
3. **The representation of `m′`** — the two's-complement sign asymmetry is what enables Step 2

## 7. Possible countermeasures for Kyber

| Countermeasure | Verdict |
|---|---|
| Ciphertext sanity check | **BROKEN** — public-key masking, ~zero attacker cost |
| Decapsulation failure check | **BROKEN** — valid ciphertexts never fail |
| "Reject if close to the public key" | **BROKEN** — use −A, 2A, or X·A |
| Shuffling | Costs the attacker ~4.5 orders of magnitude (325 → 8 M). Real speed bump, not a wall |
| First-order masking | Broken as evaluated (see caveat); shares leak individually |
| Wider parallel datapath | Genuine partial attenuation — see below |
| Unsigned / offset representation of `m′` | Attacks the enabling mechanism directly |

### Hardware-IP note

**Concrete design lever:** the paper's own numbers show **1** coefficient per store → 91 % classifier accuracy, **10** coefficients per store → **32 %**. A datapath processing P coefficients per cycle aggregates their Hamming weights and drops per-coefficient SNR ≈ 1/P. **Parallelism is a free partial mitigation** — though 32 % still recovered keys in 7800 traces, so it raises cost rather than closing the hole.

**Don't implement detection countermeasures in an IP.** Both are broken at essentially zero attacker cost.

**The greedy solver is the portable threat.** It turns *any* leakage yielding linear inequalities in (s,e) into key recovery, tolerates 16 % wrong inequalities, and runs in seconds. Assume it exists when threat-modelling.

## 8-minute slide plan

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** | 0:45 |
| 2 | The two low-cost detection countermeasures | 0:45 |
| 3 | Public-key masking: 4 lines of algebra, `A·s` cancels | 1:00 |
| 4 | Margin table → "57.8 % is a coin flip, so retry" | 0:45 |
| 5 | The bind: valid ciphertexts ⟹ decryption only | 0:30 |
| 6 | **`Δm` is linear in (s,e); everything else is known** | 1:00 |
| 7 | **`+5` → HW 2, `−5` → HW 14** (the best slide in the paper) | 1:15 |
| 8 | Inequalities → greedy solver vs BP | 1:00 |
| 9 | Results + caveats + what must be protected | 1:00 |

**Land this line:** *"Every prior attack needed a ciphertext that fails. This one doesn't fail, so the countermeasure never fires."*

---
---

# PAPER 6 — Efficient SPA Countermeasures using Redundant Number Representation

Nagpal, Hadžić, Primas, Mangard — **SAC 2025**, LNCS 16207, 753–780 · ePrint 2025/679

## SLIDE 1 — What this paper is about

**The problem.** ML-KEM computes in Z_q with q = 3329 ≈ 2^11.7, but stores coefficients in **16-bit** machine words. That mismatch leaks: the Hamming weight of a stored coefficient is a *deterministic function* of its value, and in signed two's complement it also hands over the sign. Single-trace SASCA attacks on the NTT⁻¹ exploit this and recover the key from **one trace**.

**What they do.** Quantify the leak information-theoretically, then apply the **Redundant Number Representation (RNR)** countermeasure: compute in Z_{ηq} instead of Z_q, so each residue class has η different machine encodings and the Hamming weight no longer pins down the value.

**What they find.** RNR cuts leakage from 3.561 to ~0.9 bits per coefficient and renders state-of-the-art SASCA ineffective — in simulation and on a Cortex-M4. **The INTT costs 0 % overhead**; the forward NTT costs 60–90 %.

**Why it matters.** It shows the *choice of numeric representation* is itself a security parameter — an axis most implementers never consider — and offers a cheaper alternative to shuffling for SPA.

> **One-sentence thesis:** the leak isn't the value, it's that the value determines its own Hamming weight; give each value several encodings and that stops being true.

## 1. Protection point

**The NTT / NTT⁻¹ only** — specifically the machine representation of every polynomial coefficient and butterfly intermediate.

**Scope, stated up front:** this is an **SPA / SASCA** countermeasure. The paper's own discussion says masking "is generally required to protect against DPA," positioning RNR as an alternative to **shuffling**, not to masking.

## 2. Key idea

```
I[X ; W(X)]  =  H[W(X)]  −  H[W(X) | X]
```

Without redundancy, `H[W|X] = 0` — the Hamming weight is fully determined by the stored value, so *all* of H[W] leaks. You cannot fix this by reshuffling bits; you must **manufacture conditional entropy** by giving each residue multiple encodings.

**Verified figures** (X uniform in Z_3329, 16-bit word) — I recomputed all four:

| Representation | H[W] | H[W\|X] | **I[X;W(X)]** |
|---|---|---|---|
| **Signed** Z_q | 3.561 | 0 | **3.561** |
| Unsigned Z_q | 2.756 | 0 | **2.756** |
| **RNR± (η=9, signed)** | 3.149 | 2.243 | **0.906** |
| RNR⁺ (η=5, unsigned) | 2.958 | 1.739 | **1.219** |

Two effects both help the defender: RNR **raises** entropy per coefficient (11.70 → 14.87 bits) while **lowering** leakage (3.56 → 0.91 bits).

**A free result worth its own line:** switching signed → unsigned alone saves ~0.8 bits/coefficient ≈ **206 bits per polynomial**, at zero cost. The fast signed arithmetic everyone uses for performance is measurably less secure.

## 3. Protection method / steps

1. Start from an algorithm over Z_q
2. **Find the largest η** such that no operation overflows in Z_{ηq}. Kyber NTT butterfly: `η± < 2³³/(2¹⁸q + q²) = 9.72`, `η⁺ < 9.11` → **η± = 9, η⁺ = 5**
3. **Encode** inputs as `X′ = X + Kq`, with K sampled uniformly from `[0,η)` (unsigned) or `[−η/2, η/2]` (signed)
4. **Compute entirely over Z_{ηq}**, adding Barrett reductions where the range invariant would break
5. **Decode** outputs by reducing mod q

**Implementation delta is small:** rejection-sampling bounds in `SamplePolyCBD` / `GeneratePolyMatrix`, plus reduction constants. No extra operations, and no *fresh* randomness during the computation — K is drawn once at encode time.

*(Erratum: the text says "η⁺ := 9" but Table 1 says 5. Table 1 is correct — log₂(5q) = 14.023 matches the reported entropy, and 39987 = −(5q)⁻¹ mod 2¹⁶ matches the Montgomery constant.)*

## 4. Results

Simulated SASCA (7 GS-butterfly layers, 1792 factors, 25 BP iterations):

| Implementation | Behaviour |
|---|---|
| Signed | 100 % success until σ ≥ 3.0 |
| Unsigned | degrades from σ ≥ 1.6 |
| **RNR±** | **ineffective at every noise level** |
| RNR⁺ | ~25 % at σ ≤ 0.1 |

Practical: ChipWhisperer CW308 + **STM32F303**, Picoscope 6404C, 20 dB LNA, 10 MHz **synchronised** sampling, 100 k profiling traces, LDA classifiers via SCALib. RNR drops **SNR by two orders of magnitude**; PI ≈ 0 in shallow layers; BP cannot improve guessing entropy.

Performance (kcycles, −O3): **Signed-INTT 42.61 → RNR±-INTT 42.61 = 0 % overhead.** Forward NTT 26.48 → 50.70.

## 5. Advantages / disadvantages

| Advantages | Disadvantages |
|---|---|
| **INTT: 0 % overhead** (verified: 42.61 → 42.61 kcycles) | Forward NTT +54 % (−O0) / **+91 %** (−O3) from Table 2 — the text's "62.8 %" doesn't reconcile |
| No *fresh* randomness during computation | **SPA only.** No DPA/CPA protection — masking still required |
| Composes on top of masking | **RNR⁺ is not secure** (25 % success at low noise); only RNR± survives |
| Constants-only change; easy retrofit | **Reference C only** — optimized assembly left as future work |
| Cheaper than shuffling (Ravi et al.: 7–78 %) | Protects **only** the NTT/INTT — nothing else in decapsulation |
| Evaluated under attacker-favourable conditions | Requires spare word-width headroom; useless if q fills the word |
| | Heinz–Pöppelmann demasking concern at small η not fully closed |

**Open question worth raising:** the bound permits η⁺ = 9, which I compute would give **I = 0.835 bits — better than RNR±'s 0.906**, inverting the paper's recommendation. Is RNR± genuinely better, or just better-parameterised?

## 6. Can the idea be applied in our IP? — **No, not effectively**

Three blocking reasons.

**(a) Our datapath width is fixed.** RNR's entire benefit comes from spare headroom between q and 2^word: it needs `⌈log₂ ηq⌉` bits where q needs only `⌈log₂ q⌉` (Kyber 12 → 15 bits; ML-DSA 23 → 31 bits). In an ASIC IP the width is sized to the minimum and frozen; there is no free headroom to spend, and widening means re-doing the multiplier, whose area scales roughly quadratically with width. Software gets 16-bit words *for free* from the machine ABI and can exploit the slack. Hardware pays for every bit. **The condition RNR exploits is a software accident, not a hardware one.**

**(b) One random offset, never refreshed → weak against CPA/DPA.** RNR does draw K at encode time, but that is only **log₂(9) ≈ 3.17 bits** of entropy per coefficient, versus ~11.7 bits for a proper first-order arithmetic mask — about 27 %. Worse, because the NTT is **linear**, the same `Kq` offset propagates through all seven butterfly layers with **no re-randomisation**. That is the signature of a low-entropy masking scheme that never refreshes: adequate against a single trace, but averaged away by multi-trace CPA/DPA, and open to simply enumerating all η offsets. Our threat model has to include DPA, so RNR cannot be the protection — at best it sits on top of masking we would have to build anyway.

**(c) Coverage is too narrow for the cost.** RNR protects the NTT/INTT only — nothing for the message decode, the FO transform, the hash, or the comparison. Paying multiplier area for one block, while still needing masking on that same block against DPA, is a poor trade.

### What is still worth taking from the paper

Not the countermeasure — **the measurement**. `I[X;W(X)] = H[W(X)] − H[W(X)|X]` is a few lines of code, needs no traces and no silicon, and tells you whether a chosen representation leaks before the datapath is frozen. Two ways that's directly useful:

- **Sizing decision.** The paper's own numbers show the leak grows with the gap between q and the word width. Sizing a datapath to exactly `⌈log₂ q⌉` bits **minimises representation leakage for free** — the opposite conclusion from the paper's, and the right one for hardware.
- **Signed vs unsigned.** Signed leaks 3.561 bits/coefficient against unsigned's 2.756. That ~0.8-bit gap costs nothing in area to eliminate in RTL. Cheapest finding in the paper, and it *does* transfer.

## 8-minute slide plan

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** | 0:45 |
| 2 | Threat model: single-trace SASCA on the NTT⁻¹ | 0:45 |
| 3 | `+5` → HW 2, `−5` → HW 14; the q ≪ 2^word mismatch | 1:00 |
| 4 | MI table: 3.561 / 2.756 / 0.906 / 1.219 | 1:00 |
| 5 | `I = H[W] − H[W\|X]`, and why `H[W\|X] = 0` without redundancy | 1:15 |
| 6 | RNR: compute in Z_{ηq}, `X′ = X + Kq`; η bounds → 9 and 5 | 1:00 |
| 7 | Results (RNR± ineffective at all σ) + **42.61 vs 42.61 kcycles** | 1:00 |
| 8 | Advantages / disadvantages + **verdict: SPA-only, needs spare width, no DPA cover** | 1:15 |

**Land this line:** *"A clever result about representation, but it buys single-trace resistance with datapath width — and in an ASIC, width is the one thing we don't have spare."*
