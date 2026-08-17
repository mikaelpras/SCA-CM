# Deep Dive: An Enhanced Two-Step CPA Side-Channel Analysis Attack on ML-KEM

**Authors:** Mark Kennaway, Tuan Hoang, Ayesha Khalid, Ciara Rafferty, Máire O'Neill
**Affiliation:** Centre for Secure Information Technologies (CSIT), Queen's University Belfast
**Venue:** SECRYPT 2025 (22nd International Conference on Security and Cryptography), pp. 263–274
**DOI:** 10.5220/0013638600003979 · ISBN 978-989-758-760-3 · CC BY-NC-ND 4.0
**Funding:** Integrated Quantum Networks (IQN) Research Hub, EP/Z533208/1

---

## 1. Summary — What the Paper Is About

A **non-profiled, known-ciphertext Correlation Power Analysis attack** on ML-KEM (Kyber512) running on an ARM Cortex-M4, targeting the polynomial multiplication inside decapsulation.

The central contribution is not a new statistical technique — it is plain CPA with a Hamming Weight model. The contribution is **reading the pqm4 assembly closely enough to notice that one specific intermediate value gets parked alone in a register**, while every other intermediate is fused into a multiply-accumulate instruction. That single architectural accident splits the secret key into two classes:

- **Odd coefficients** leak *directly and strongly* (ρ ≈ 0.87, MtD as low as 10 traces)
- **Even coefficients** leak only *indirectly*, through a composite expression that requires the odd coefficients to already be known (ρ ≈ 0.32–0.40, MtD up to 179 traces)

Hence the "two-step" structure. Full key recovery in **179 traces**, with **no chosen-ciphertext manipulation, no zero-value filtering, and no profiling phase**.

---

## 2. Existing SCA Method Used

| Element | Choice |
|---|---|
| Attack class | Non-profiled CPA (Brier, Clavier & Olivier, CHES 2004) |
| Leakage model | Hamming Weight |
| Statistic | Pearson Correlation Coefficient |
| PoI selection | PCC sweep across sample points using known intermediates from a reference implementation |
| Trace-efficiency metric | Incremental PCC (Bottinelli & Bos, JCEN 2017) → Measurements to Disclosure (MtD), per Tiri et al. CHES 2005 |
| Hypothesis space | 3329 values per coefficient (full Z_q), i.e. q not q² |

Nothing exotic. The incremental PCC is worth noting as a methodological detail: it adds a "number of traces" axis to the correlation data, letting you read off the exact trace count at which the correct hypothesis diverges from the pack. That is what produces the MtD figures.

---

## 3. Attack Point

### 3.1 Algorithm level

`KYBER.CPAPKE.Dec(sk, c)`, **line 4**:

```
m ← Encode₁(Compress_q(v − NTT⁻¹(ŝᵀ ∘ NTT(u)), 1))
```

The `ŝᵀ ∘ NTT(u)` term is the target. Critically, **decryption is invoked on every decapsulation regardless of ciphertext validity**, and the ciphertext is always multiplied by the secret key. That is what gives the attacker a decryption oracle without needing to defeat the FO transform.

### 3.2 Implementation level (pqm4)

Decryption decomposes into nine sub-operations. The relevant one is `poly_frombytes_mul(&mp, sk)` and `poly_frombytes_mul(&bp, sk + i*KYBER_POLYBYTES)`, which implement `ŝᵀ ∘ NTT(u)`. For Kyber512 (k = 2), the first invocation processes ŝ⁰, the second ŝ¹.

Inside that sits `doublebasemul_asm` — two `basemul` operations, each performing pairwise pointwise multiplication of two 12-bit secret coefficients against two 12-bit ciphertext coefficients.

### 3.3 The actual leak — instruction level

Each `basemul` on inputs s = {s₀, s₁}, u = {u₀, u₁} consists of **five `fqmul` executions**:

```
fqmul(s1,u1), fqmul(r0,zeta), fqmul(s0,u0), fqmul(s0,u1), fqmul(s1,u0)
```

producing outputs r₀ and r₁. The assembly:

```asm
smultt  tmp,  poly0, poly1        ; s_odd × u_odd  (top × top)
montgomery q, qinv, tmp,  tmp2    ; → reduced product lands ALONE in tmp2
smultb  tmp2, tmp2,  zeta         ; × zeta
smlabb  tmp2, poly0, poly1, tmp2  ; s_even × u_even + accumulate
montgomery q, qinv, tmp2, tmp     ; → r0 in tmp
smuadx  tmp2, poly0, poly1        ; s_odd×u_even + s_even×u_odd (dual, summed)
montgomery q, qinv, tmp2, tmp3    ; → r1 in tmp3
```

**This is the whole paper in seven lines.**

- `SMULTT` produces an *isolated* product of one secret coefficient and one ciphertext coefficient. After Montgomery reduction it sits in `tmp2` for at least one clock cycle. → **directly attackable, single unknown.**
- `SMLABB` is multiply-**accumulate**: `s_even × u_even` is never materialised on its own; it is summed with `fqmul(fqmul(s₁,u₁), zeta)` inside the same instruction. → **not directly attackable.**
- `SMUADX` is a **dual** multiply: two products computed and summed within one instruction. → **not directly attackable.**

The authors confirm this empirically: correlation against `fqmul(s0,u0)`, `fqmul(s1,u0)` and `fqmul(s0,u1)` stays flat across all sample points. **s₀ cannot be recovered by a single-step attack.**

---

## 4. Attack Model

| Aspect | Detail |
|---|---|
| Class | Non-profiled — no clone device, no template building |
| Ciphertext control | **Known** ciphertext, not chosen. Ciphertexts generated by deterministic `CPAPKE.Enc` on random 32-byte messages |
| Preconditions | None. No zero-value filtering, no crafted ciphertexts, no ciphertext malleability |
| Oracle | Decapsulation with a reused key — attacker triggers decapsulation and records power |
| Key assumption | **Static or semi-static key.** The whole attack collapses if the key is refreshed more often than ~179 decapsulations |
| Adversary capability | Physical power measurement access + ability to invoke decapsulation ~180–500 times |
| Knowledge assumed | The implementation's assembly (pqm4 is open source), and û values (derivable from known ciphertext) |

This is a **weak adversary model**, which is the paper's real selling point. Most competitive results in this space need chosen ciphertexts or a profiling device.

---

## 5. Attack Procedure

### Preprocessing / PoI identification
1. Capture 500 power traces over `doublebasemul_asm` only, at 4× clock (4 × 7.37 MHz).
2. Independently compute ŝ, û and the raw `fqmul` outputs on a laptop reference implementation; cross-validate against device values (done — they matched).
3. Correlate each candidate intermediate against traces, sample point by sample point, to locate PoIs.
   - Global peak for `fqmul(s1,u1)` at **samples 153–164**
   - Global peaks for the composite r₀ / r₁ expressions at **samples 185–190** and **245–250**

### Step One — recover odd coefficients (s₁, s₃)

Attack function:
```
HW( fqmul(h₁, u₁) )     and     HW( fqmul(h₃, u₃) )
```
h = hypothetical key coefficient, 0…3328, in the NTT domain.

| Target | Recovered value | PoI | Peak ρ | MtD |
|---|---|---|---|---|
| s⁰₁ | 1683 | 158 | 0.869 | **10** |
| s¹₁ | 1920 | 158 | 0.876 | — |
| s⁰₃ | 2355 | 213 | 0.606 | **43** |
| s¹₃ | 2336 | 214 | 0.554 | — |

Note the top *five* correlations for h⁰₁ all point at the same hypothesis (1683) across consecutive sample points 157–161 — a clean, unambiguous result. Four consecutive strong points is consistent with 4× oversampling of a single clock cycle.

### Step Two — recover even coefficients (s₀, s₂)

Now that s₁, s₃ are known, they become constants inside a composite attack function targeting r₀ in `tmp`:

```
HW( fqmul(fqmul(s₁, u₁), zeta)  + fqmul(h₀, u₀) )
HW( fqmul(fqmul(s₃, u₃), −zeta) + fqmul(h₂, u₂) )
```

| Target | Recovered value | PoI | Peak ρ | MtD |
|---|---|---|---|---|
| s⁰₀ | 72 | 186 | 0.323 | **179** |
| s¹₀ | 3015 | 189 | 0.376 | — |
| s⁰₂ | 2841 | 238 | 0.402 | **43** |
| s¹₂ | 780 | 237 | 0.369 | — |

Correlations are roughly **2.7× lower** than Step One, which the authors attribute to the more elaborate algebraic structure. The PoIs are also less tightly clustered.

**Alternative formulation** the authors note but don't pursue: target r₁ in `tmp3` via `fqmul(s_even, u_odd) + fqmul(s_odd, u_even)`. Worth a slide question — why did they prefer r₀?

### Generalisation to the full key
`doublebasemul` is structurally identical for every coefficient pair, so the same two functions replay across all 512 coefficients (k=2 × n=256). The 179-trace figure is the worst-case MtD observed.

### Reported leakage trend
MtD **decreases** as the attack progresses through successive `doublebasemul` iterations (shown for the first 16 coefficients of ŝ¹). Authors' explanation: noise decreases with device operation, so later traces correlate more strongly. They predict a gradually diminishing MtD toward some floor.

---

## 6. Parts Needing Protection

Ordered by exposure, as identified by this paper:

1. **Any register that holds an isolated single-coefficient product** post-reduction, even for one clock cycle. This is the primary sin. `tmp2` after `SMULTT`+`montgomery`.
2. **The output of Montgomery reduction generally** — reduction narrows the value range and makes HW modelling cleaner.
3. **Odd/even asymmetry in the pointwise multiplication schedule.** The pqm4 assembly treats the two halves of the packed 32-bit word differently, and that asymmetry is the entire attack surface.
4. **The `basemul` / `doublebasemul` inner loop** as a whole — this is where ŝ and the attacker-known û meet.
5. **Key lifetime.** Not an implementation surface, but the paper makes it the primary defensive lever.

Not implicated here (but attacked elsewhere in the literature per their Table 1): message encoding/decoding, NTT/INTT itself, Barrett reduction, polynomial subtraction, binary unpacking.

---

## 7. Countermeasures

### Proposed by the authors
1. **Avoid static/semi-static key configurations.** Refresh keys more often than the MtD. They explicitly position 179 traces as a practical upper bound on key reuse in an unprotected implementation. Trade-off: keygen cost.
2. **Masking and shuffling** — acknowledged, but hedged. They cite literature suggesting "limited effect" and defer to future work (they intend to attack the Heinz et al. 2022 first-order masked Kyber on Cortex-M4).

This is the weakest section of the paper. Countermeasures get roughly half a page.

### Countermeasures the attack structurally suggests (my reading, not the paper's)
- **Never let a single secret×public product land alone in a storage element.** Fuse the multiply into an accumulate, or delay the write-back. The paper's own evidence shows this is worth ~2.7× in correlation — `SMLABB` and `SMUADX` are, accidentally, mitigations.
- **Symmetrise the odd/even paths** so both halves have identical leakage structure. Removing the strong Step One would force the attacker into a q² joint search or a much lower-SNR composite.
- **First-order Boolean/arithmetic masking of ŝ** before entry to `poly_frombytes_mul` — the standard answer, and the only one with a security argument rather than an obscurity argument.
- **Shuffling coefficient order** raises the trace count by a factor related to the permutation entropy, but the paper's own observation that later iterations leak *more* suggests shuffling interacts with the noise trend in ways worth testing.

---

## 8. Comparison with Other Non-Profiled Attacks

| Work | Security level | Traces |
|---|---|---|
| **This work** | 1: Kyber512 | **179** |
| Tosun et al. 2024 | 3: Kyber768 | 250–400 |
| Mujdei et al. 2024 | 3: Kyber768 | 200 |
| Tosun & Savaş 2024 | 3: Kyber768 | 160 |
| Yang et al. 2023 | 1: Kyber512 | 25–500 |

Their claimed edge: Mujdei and Tosun et al. recover two coefficients at once, implying a q² search space; Tosun & Savaş reduce that to q via zero-value filtering but pay q traces to do it. Yang et al. use ciphertext manipulation, which adds overhead and complexity. This work needs neither — it gets a q-sized search for the odd coefficients for free from the instruction schedule.

They also note that Yang et al. do assembly-level analysis at a comparable depth but **overlook the `SMULTT` opportunity**, as does every other work in the table.

---

## 9. Critical Points for Discussion

Worth raising in the seminar — the paper is solid but the framing is generous in places:

- **"179 traces" is the worst-case MtD, not the campaign size.** They captured 500 traces and used 500 ciphertexts throughout. The honest claim is "the hardest coefficient we measured needed 179"; the abstract reads as though the whole attack cost 179 traces.
- **The comparison table mixes security levels.** 179 on Kyber512 (k=2) vs 160 on Kyber768 (k=3) is not a like-for-like win, and they acknowledge inter-group comparison is hard without fully discounting their own table.
- **Only 8 coefficients are demonstrated in detail** (s₀–s₃ across two polynomials). Full-key recovery is argued from the structural regularity of `doublebasemul`. Reasonable, but asserted rather than shown.
- **The decreasing-MtD trend is under-explained.** "Noise decreases with device operation" is hand-wavy. Alternative explanations they don't rule out: trace alignment improving away from the trigger, capture-hardware settling, or dependence on the particular û coefficient magnitudes. This is the most interesting empirical claim in the paper and the least rigorously supported.
- **Total dependence on one implementation's instruction scheduling.** Change the assembly, recompile with a different scheduler, or move to a different core, and Step One may vanish entirely. That is a strength (concrete, reproducible) and a weakness (narrow).
- **No masked or shuffled target was attacked.** The countermeasure discussion is speculative; the actual evaluation is entirely against an unprotected implementation.

---

## 10. Transfer to Hardware IP

The specific attack does **not** transfer — it is an ARM Cortex-M4 instruction-scheduling artifact, and a hardware datapath has no `SMULTT`/`SMLABB` distinction.

The **architectural principle does transfer**, and it is a sharp one:

> Any storage element (register, flop, pipeline stage) that holds a value functionally dependent on exactly one secret coefficient is a first-order univariate CPA target with a q-sized hypothesis space. Any storage element holding a value dependent on two or more secret coefficients forces the attacker into q² or a lower-SNR composite model.

Design implications worth carrying forward:

- **Audit every pipeline register for single-secret-coefficient dependence.** In an ASIC pointwise-multiplier, the natural place this happens is the output flop of a modular multiplier before accumulation. Same failure mode as `tmp2`.
- **Accumulate before you store, not after.** Combinational fusion of at least two secret-dependent products before the flop boundary buys measurable correlation reduction essentially for free in area. The paper quantifies that benefit at ~0.87 → ~0.32 on real silicon, which is a useful empirical anchor even though it comes from a microcontroller.
- **But: fusion is obfuscation, not security.** Step Two still succeeded, at 179 traces. Correlation reduction of 2.7× is not a countermeasure, it is a delay. This is the argument for masking as the load-bearing defence and datapath structure as defence-in-depth only.
- **Relevance to the masked sparse-multiplier work:** the analogous exposure is the accumulator holding a partial sum attributable to a single (index, sign) table entry. If the sparse traversal writes back after each challenge-coefficient contribution, the accumulator sequence is a `tmp2` equivalent. Worth checking whether the minimal-ring formulation forces or avoids that.

---

## 11. Key References to Follow Up

- Brier, Clavier & Olivier — *Correlation Power Analysis with a Leakage Model*, CHES 2004 (the CPA baseline)
- Bottinelli & Bos — *Computational Aspects of Correlation Power Analysis*, JCEN 2017 (incremental PCC)
- Heinz et al. — *First-Order Masked Kyber on ARM Cortex-M4*, ePrint 2022/058 (the authors' stated next target)
- Hoang et al. — *Deep Learning Enhanced Side Channel Analysis on CRYSTALS-Kyber*, ISQED 2024 (same group, fqmul **inputs** rather than outputs)
- Tosun & Savaş — *Zero-Value Filtering...*, IEEE TIFS 2024 (the closest competitor)
- Mujdei et al. — *Side-Channel Analysis of Lattice-Based PQC: Exploiting Polynomial Multiplication*, ACM TECS 2024
- Kannwischer et al. — pqm4 (the target implementation)
