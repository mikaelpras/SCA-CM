# Deep Dive: Pushing the Limits of Generic Side-Channel Attacks on LWE-based KEMs — Parallel PC Oracle Attacks on Kyber KEM and Beyond

**Authors:** Gokulnath Rajendran, Prasanna Ravi, Jan-Pieter D'Anvers, Shivam Bhasin, Anupam Chattopadhyay
**Affiliations:** Temasek Laboratories & SCSE, Nanyang Technological University, Singapore · imec-COSIC, KU Leuven
**Venue:** IACR TCHES 2023, Vol. 2023, No. 2, pp. 418–446
**DOI:** 10.46586/tches.v2023.i2.418-446 · ePrint 2022/931 · CC-BY 4.0
Received 2022-10-15 · Accepted 2022-12-15 · Published 2023-03-06

---

## 1. Summary — What the Paper Is About

Existing **Plaintext-Checking (PC) oracle** side-channel attacks on Kyber are attractive because they are *generic* — the attacker needs almost no knowledge of the target implementation. But they extract **one bit of secret information per query**, so full key recovery costs 1,300–2,400 queries.

This paper's observation: the leakage source is the **entire FO re-encryption procedure**, which contains *thousands* of leaky points of interest. Using all of that to distinguish `m = 0` from `m = 1` is enormous waste.

So they generalise to a **P-way parallel PC oracle** — craft chosen ciphertexts so that P message bits simultaneously depend on P different secret coefficients, then classify the trace among 2^P classes instead of 2. Three supporting contributions make it work:

1. **Ciphertext construction** that makes the P bits independently controllable
2. **New Binary Decision Trees** — minimum *depth*, not minimum *entropy* — because the optimal tree for the binary case is provably suboptimal in parallel
3. **Multi-class t-test classifier** validated at P = 10 (1024 classes) with 100% success rate

Result: **2.89×–7.65×** fewer queries than state-of-the-art binary PC oracle attacks, up to **~24.6×** for a well-resourced attacker. And it **completely ignores the shuffling countermeasure**.

The paper's framing is designer-facing: *what is the cheapest generic attack on an unprotected implementation, and therefore what key refresh rate is actually safe?* Their conclusion is that the answer is low enough that **masking is not optional**.

---

## 2. Existing SCA Method Used

| Element | Choice |
|---|---|
| Attack class | Chosen-ciphertext, side-channel-assisted PC oracle (TO_Indep) |
| Baseline | Binary PC oracle attack of Ravi et al. [RRCB20], TCHES 2020 |
| Side channel | Electromagnetic emanation |
| Distinguisher | Welch's t-test for PoI selection + sum-of-squared-difference template matching |
| Multi-class strategy | Pairwise binary classification in knockout-tournament structure |
| Query optimisation | Binary Decision Trees, extending Qin et al. [QCZ+21] |
| Partial-recovery estimation | Leaky LWE Estimator (Dachman-Soled et al.) |

Note there is **no neural network and no deep learning**. That is a deliberate design choice — see §9.

---

## 3. The Taxonomy They Introduce (worth internalising)

The paper splits Kyber SCA into two families, and the whole contribution is framed as bridging them:

| | **TO_Dep** (Target-Operation Dependent) | **TO_Indep** (Target-Operation Independent) |
|---|---|---|
| Targets | One specific operation (message encoding/decoding, NTT) | Leakage from anywhere in decapsulation |
| Traces | Very few (single-trace possible) | Few thousand |
| Leakage exploited | Fine, single-bit manipulations, few clock cycles | Coarse, thousands of PoIs |
| Fragility | High — sensitive to SNR, needs detailed target knowledge, unclear on pipelined/parallel hardware | Low — largely agnostic to implementation |
| Examples | Primas et al. NTT attacks, Xu et al. FD oracle, message-encoding attacks | Ravi et al. binary PC oracle, Guo et al. DF oracle |

**The trade-off this paper attacks:** ease of mounting vs. number of queries. Their answer is a TO_Indep attack with TO_Dep-competitive query counts.

---

## 4. Attack Point

**Not the polynomial multiplication.** This is the key structural difference from the CPA-style attacks.

The target is the **re-encryption inside CCA.Decaps**:

```
CCA.Decaps(sk, ct):
  m = CPA.Decrypt(sk, ct)
  (K̄, r) = G(m, H(pk))
  ct' = CPA.Encrypt(pk, m, r)     ← THE TARGET: Re_Encrypt(m, pk)
  if ct' == ct: return KDF(K̄ ‖ H(ct'))
  else:         return KDF(z ‖ H(ct'))
```

`Re_Encrypt(m, pk)` is **deterministic in m** for a fixed public key. A single-bit difference in m propagates through hashing into completely different computation across the whole re-encryption. That is thousands of distinguishable leakage points, essentially for free.

**Where the actual leakage came from in their experiments:** the t-test showed two dominant peaks at the **CBD sampling of the ephemeral secret r** (Line 11 of CPA.Encrypt). Notably, the attacker doesn't need to know this — it falls out of the t-test. And they **deliberately did not use message-encoding leakage**, which is what makes the shuffling result work (§8).

---

## 5. Attack Model

| Aspect | Detail |
|---|---|
| Ciphertext control | **Chosen** ciphertext — the attacker submits malformed ct |
| Physical access | Query the decapsulation device and record EM |
| Implementation knowledge | **None required** — no source, no binary, no clock-cycle knowledge |
| Known-key profiling | **Not required** — templates built from known (m, pk) pairs, key never known |
| Clone device | **Optional** — improves the attack substantially but is not necessary |
| Key setting | Static / semi-static key with regular refresh |
| Per-public-key cost | Pre-processing must be **redone for every new public key**, since Re_Encrypt depends on pk |

This is a genuinely weak adversary model. The attacker needs a basic EM setup and the ability to send chosen ciphertexts — nothing else.

---

## 6. Attack Procedure

### 6.1 Baseline: the binary PC oracle

Chosen ciphertext `ct = (u, v)` with `u = k_u · x⁰`, `v = k_v · x⁰`:

```
m_i = Decode(k_v − k_u · s[0])    for i = 0
m_i = Decode(−k_u · s[i])         for i ∈ [1, n−1]
```

Choose (k_u, k_v) so that **only m₀ is secret-dependent** and all other bits are forced to 0. Then m ∈ {0, 1}, and which one depends solely on s[0]. That's the oracle.

**Rotation trick:** multiplying by x^p rotates anti-cyclically in R_q. Setting `u = k_u · x^p` makes m₀ depend on s[n−p] instead, so the same machinery walks the whole key.

**BDT optimisation:** Kyber coefficients follow a non-uniform CBD, so allocate fewer queries to more probable values. Query count is inversely proportional to probability of occurrence.

| Parameter set | Coeff range | Q_bin | Max depth |
|---|---|---|---|
| Kyber512 | [−3, 3] | 2.5625 | 4 |
| Kyber768 / 1024 | [−2, 2] | 2.3125 | 3 |

`Q_attack = 256 · k · Q_bin` → **1312 / 1776 / 2368** queries for Kyber512/768/1024.

### 6.2 The two observations that motivate the paper

1. Only **one** message bit is secret-dependent per query; the other 255 are wasted
2. Thousands of leaky PoIs are consumed to extract **one bit**

### 6.3 Parallel construction

```
u = 208 · x⁰                              (k_u fixed at 208)
v = 208 · t · Σᵢ₌₀^(P−1) xⁱ
```

giving

```
m_i = F(s[i])   for i ∈ [0, P−1]
m_i = 0         for i ∈ [P, n−1]
```

Now the **first P message bits each depend on a different secret coefficient**, and everything else is pinned to zero.

**Why this is not the FD oracle:** in Full-Decryption oracle attacks all 256 bits depend on s. Here the attacker **controls both the number and the position** of secret-dependent bits. That control is what lets the P BDT traversals stay independent.

**Why the traversals are independent:** k_u = 208 is fixed (found by exhaustive search) so that varying only v suffices to distinguish every possible coefficient value. Each BDT_i is then driven purely by its own message bit m_i.

### 6.4 The BDT insight — min-depth beats min-entropy

This is the subtlest contribution and worth a slow read.

For a *set* of P coefficients, the number of queries is governed by the **deepest** coefficient in the set, not the average. As P grows, the probability that at least one coefficient sits deep in the tree approaches 1. So minimum-*entropy* trees — optimal for P = 1 — become progressively worse.

They construct **BDT_min_depth** for Kyber512: depth 3, the lowest achievable for unique distinguishability over [−3, 3]. It beats BDT_min_ent for **P ≥ 3**.

For Kyber768/1024 the [−2, 2] range means the min-entropy tree *already* has minimum depth 3, so BDT_min_ent = BDT_min_depth and no change is needed.

Average queries to recover a set of P coefficients:

```
Q_set = Σ_{st_i ∈ V}  i · R_i
```

where R_i = probability that the P coefficients fall within subtree st_i with at least one in its last layer.

```
Q_attack = ⌈256/P⌉ · k · Q_set
```

They note a fully optimal tree would split into 2^P branches per node rather than 2, but that needs one template per node — usually not worth it.

### 6.5 Multi-class classifier

**The enabling observation:** because of hash diffusion in re-encryption, *any two* values of m are distinguishable by t-test — not just 0 and 1. They demonstrate this on m = 330 vs m = 559.

**Pre-processing (per public key):**
1. Collect T traces of `Re_Encrypt(m, pk)` for each of the 2^P values of m
2. Normalise each trace (subtract mean, divide by std)
3. Welch's t-test between class pairs to find univariate leakage
4. Select points above threshold Th_PoI as the PoI set P
5. Reduced template for each class = mean of the reduced trace set

**Classification:** normalise the attack trace, compute sum-of-squared-difference against each candidate template, pick the minimum.

**Scaling to 2^P classes:** pairwise classification in a **knockout tournament** — 2^P − 1 matches, which is optimal. The alternative (all-pairs + majority vote) costs h² classifications.

---

## 7. Results

### Experimental setup
- **DUT:** STM32F407VG on STM32F4DISCOVERY, ARM Cortex-M4, clocked at 24 MHz
- **Target:** pqm4 `m4speed` Kyber768, unprotected
- **Channel:** near-field EM probe on chip, Lecroy HD6104, 250 MSam/s, 30 dB pre-amp
- **Templates:** T = 5 traces per class

### Achieved
- **P = 10** (1024 classes): full key recovery, **100% success rate**, 5520 template traces
- **P = 12** (4096 classes): pairwise classification verified at 100% accuracy
- No principled upper bound on P established — acknowledged as open

### Cost model

```
Q_total = Q_template + Q_attack = 2^P · T  +  ⌈256/P⌉ · k · Q_set
```

`Q_template` grows **exponentially** in P; `Q_attack` shrinks **inversely**. Hence a minimum exists.

**With clone device** (templates built offline, Q_template = 0 on the target):

| P | Queries to target (Kyber768) | Improvement |
|---|---|---|
| 1 (baseline) | 1776 | — |
| 10 | **232** | 7.65× |
| 12 | 197 | 9.0× |
| 32 (hypothetical) | 72 | ~24.6× |

**Without clone device** (both phases on the target):

| Config | Optimal P | Queries | Improvement |
|---|---|---|---|
| T = 5 (validated) | 4 | **613** | 2.89× |
| T = 1 (best case) | 6 | 437 | ~4× |

### Partial key recovery
Using the Leaky LWE Estimator, coefficients needed to drop security to a given offline work factor:

| | Full | 2³² offline | 2⁶⁴ offline |
|---|---|---|---|
| Kyber512 | 512 | 354 | 184 |
| Kyber768 | 768 | 667 | 463 |
| Kyber1024 | 1024 | 1010 | 782 |

Kyber768 with clone at P = 10 drops from 232 (full) → 202 (2³²) → **140** (2⁶⁴).

---

## 8. Parts Needing Protection & Countermeasures

### What is exposed
1. **The FO re-encryption procedure as a whole** — determinism in m is the vulnerability, not any single instruction
2. **CBD sampling of the ephemeral secret r** — the strongest observed leakage in their setup
3. **Any operation downstream of m** — hashing, encryption, comparison. Diffusion means every one of them carries information about m
4. **Key lifetime** — the attack is a query-count game; the refresh interval X must be below the attack cost Y

### Shuffling — defeated, and instructively so
Shuffling was proposed to stop single-trace message-encoding attacks, and it works for that: it prevents recovery of message-bit *order*, killing the FD oracle and forcing the attacker down to one bit per query.

**But this attack never touches message encoding.** They used only leakage up to the sampling of r. Their result: key recovery on the shuffled implementation with **exactly the same trace count as unprotected**, validated at P = 10 with single-trace class distinction among 1024 classes.

This is now the lowest known query count against shuffled Kyber.

### Masking
Explicitly declared **out of scope**. The paper's conclusion is a call for masking, framed from a designer's perspective: given how cheap the generic attack is, low-cost countermeasures don't buy enough, so the ~3.1× runtime penalty of masking is justified.

---

## 9. Comparison with Concurrent Work (Tanaka et al., TCHES 2023)

Independently developed, same core idea. Three differences the authors claim:

| | This work | Tanaka et al. [TUX+22] |
|---|---|---|
| BDT | New min-depth trees for the parallel setting | Reuses binary-optimal trees → more queries |
| Classifier | t-test, T < 10 traces per class | Neural network, 1000 train + 500 val per class |
| Target | Full Kyber implementation | SHAKE, SHA-3, AES components |

The classifier point is the substantive one: an NN needs ~1500 traces per class, which is fine on a clone device but ruinous without one, since pre-processing must be repeated per public key.

---

## 10. Generalisation Beyond Kyber

The chosen-ciphertext construction relies on the **LPR encryption framework**, not on Kyber specifics. Applies to Saber, FrodoKEM, NewHope, Round5, LAC.

**Saber sketch** (validated by simulation): coefficients in [−4, 4]. With k_u = 57 and k_v ∈ [1, 7] they distinguish [−2, 4] in ≤4 queries, but **cannot separate −3 from −4** with that k_u — needs one extra query at (k_u, k_v) = (54, 1). Estimated ~390 queries at P = 10, vs 232 for Kyber768.

**NTRU-paradigm schemes** (NTRU, NTRU Prime) are explicitly **not** straightforward — left as future work.

---

## 11. Critical Points for Discussion

- **The masking conclusion is asserted, not demonstrated.** Masking is out of scope; no masked implementation was attacked. The argument is "the unprotected/lightly-protected attack is cheap, therefore masking is needed" — reasonable, but the paper cannot say how this attack fares against a masked target.
- **The 24.6× headline is a thought experiment.** P = 32 means 2³² classes. Building 2³² templates, even offline on a clone, is not a real attack — it is an asymptote. The honest validated numbers are 7.65× (with clone) and 2.89× (without).
- **Per-public-key pre-processing is a real constraint** that the "with clone" framing softens. Every new key pair means rebuilding all 2^P templates.
- **No upper bound on P.** They verified P = 12 pairwise and P = 10 end-to-end, and openly say the limit depends on SNR, device, and implementation, and can only be found empirically. Fair, but it means the "arbitrarily higher improvements" claim is untested.
- **100% success rate is setup-dependent.** They gesture at majority voting and error-correcting-code encodings of oracle responses for lower-SNR settings, but don't evaluate them.
- **Hardware applicability is stated as future work.** Their own conclusion lists "applicability in the hardware targets" as open — see §12.

---

## 12. Transfer to Hardware IP

The most important contrast with implementation-specific attacks: **this one does not care about your datapath.**

- The vulnerability is **algorithmic** — determinism of FO re-encryption in m. Re-scheduling instructions, changing multiplier architecture, pipelining, or fusing operations does nothing. A hardware ML-KEM decapsulator has exactly the same structure as the Cortex-M4 one.
- **Hardware may make it harder in practice but not in principle.** Parallelism, deep pipelining and jitter reduce SNR, which lowers the achievable P — the paper's own critique of TO_Dep attacks (fine leakages don't survive complex platforms) works partly in hardware's favour here. But P = 1 still gives the 1776-query baseline, and the attack still works.
- **Scope lesson for an IP block:** protecting the polynomial multiplier is not sufficient for a Kyber/ML-KEM core. The Keccak/SHA-3 path, the CBD sampler, and the ciphertext comparison all leak m. If the deliverable is "SCA-protected ML-KEM IP", the protected boundary has to enclose the whole decapsulation, not the arithmetic hot spot.
- **Direct relevance to ML-DSA work is limited** — signatures have no FO transform and no PC oracle in this form. The transferable point is methodological: a masked datapath answers CPA-class attacks, but oracle-class attacks are defeated at the protocol/algorithm level. Both threat models need separate arguments in any security claim.
- **Useful contrast to hold in mind:** where instruction-fusion bought a measurable but insufficient correlation reduction against CPA, **nothing at the microarchitectural level moves the needle here**. Different attack class, different defence layer.

---

## 13. Key References to Follow Up

- Ravi, Roy, Chattopadhyay & Bhasin — *Generic Side-Channel Attacks on CCA-secure Lattice-based PKE and KEMs*, TCHES 2020 (the binary baseline)
- Qin et al. — *A Systematic Approach and Analysis of Key Mismatch Attacks on Lattice-based NIST Candidate KEMs*, ASIACRYPT 2021 (BDT construction)
- Tanaka et al. — *Multiple-Valued Plaintext-Checking Side-Channel Attacks on Post-Quantum KEMs*, TCHES 2023 (concurrent work)
- D'Anvers, Tiepelt, Vercauteren & Verbauwhede — *Timing Attacks on Error Correcting Codes in Post-Quantum Schemes*, TIS 2019 (origin of the PC oracle)
- Guo, Johansson & Nilsson — *A Key-Recovery Timing Attack... FrodoKEM*, CRYPTO 2020 (the DF oracle sibling)
- Dachman-Soled, Ducas, Gong & Rossi — *LWE with Side Information*, CRYPTO 2020 (leaky LWE estimator, for the partial-recovery numbers)
- Bos, Gourjon, Renes, Schneider & van Vredendaal — *Masking Kyber: First- and Higher-Order Implementations*, TCHES 2021 (the countermeasure they advocate)
