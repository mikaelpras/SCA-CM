# Slide Outline — Enhanced Two-Step CPA on ML-KEM
**Kennaway et al., SECRYPT 2025 · target: ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** Enhanced Two-Step CPA on ML-KEM — Kennaway et al., SECRYPT 2025 (QUB / CSIT)

- Non-profiled, **known-ciphertext** CPA on Kyber512 decapsulation, ARM Cortex-M4 (pqm4)
- Target: the pointwise polynomial multiplication ŝᵀ ∘ NTT(u)
- Not a new statistical method — a **new attack point found by reading the assembly**
- Result: full secret key, **179 traces**, no chosen ciphertexts, no profiling, no zero-value filtering

> Say: "The contribution is an observation about instruction scheduling, not about statistics."

---

## Slide 2 — Where the Attack Sits (45 s)

**Kyber CPAPKE.Dec, line 4:**
```
m ← Encode(Compress(v − NTT⁻¹(ŝᵀ ∘ NTT(u))))
```

- Decryption runs on **every** decapsulation, valid ciphertext or not → free decryption oracle
- pqm4 path: `poly_frombytes_mul` → `doublebasemul_asm` → 2 × `basemul` → 5 × `fqmul`
- Attacker knows û (from the ciphertext); ŝ is the unknown

*Diagram: decap flow with the multiplication boxed.*

---

## Slide 3 — The Key Insight ★ (2 min — the money slide)

**Five `fqmul` per basemul, but only one leaves an isolated result:**

```asm
smultt  tmp,  poly0, poly1        ; s_odd × u_odd
montgomery q, qinv, tmp, tmp2     ; ← lands ALONE in tmp2
smultb  tmp2, tmp2, zeta
smlabb  tmp2, poly0, poly1, tmp2  ; s_even × u_even + accumulate
montgomery q, qinv, tmp2, tmp     ; → r0
smuadx  tmp2, poly0, poly1        ; two products, summed in one instruction
montgomery q, qinv, tmp2, tmp3    ; → r1
```

| Instruction | Structure | Attackable? |
|---|---|---|
| `SMULTT` | single product, stored | **Yes — one unknown, q hypotheses** |
| `SMLABB` | multiply-**accumulate** | No — fused |
| `SMUADX` | **dual** multiply, summed | No — fused |

→ **Odd coefficients leak directly. Even coefficients don't.** That asymmetry is the two steps.

Empirically confirmed: correlation against `fqmul(s0,u0)`, `(s1,u0)`, `(s0,u1)` is flat everywhere.

---

## Slide 4 — Attack Model & Setup (45 s)

- **Non-profiled** — no clone device
- **Known** ciphertext, not chosen — no manipulation, no preconditions
- Requires a **static / semi-static key** (≥179 decapsulations under one key)
- ChipWhisperer-Lite + CW308 UFO + CW308T-STM32F3 (Cortex-M4), pqm4 Kyber512
- 500 traces captured, sampled at 4× clock; intermediates cross-validated against a laptop reference

> One line on why the weak adversary model is the real selling point.

---

## Slide 5 — Step One: Odd Coefficients (1 min)

**Attack function:** `HW( fqmul(h₁, u₁) )`, h ∈ [0, 3328]

| Target | Value | ρ | MtD |
|---|---|---|---|
| s⁰₁ | 1683 | **0.869** | **10** |
| s⁰₃ | 2355 | 0.606 | 43 |

- PoI at samples 157–160 — four consecutive points = one clock cycle at 4× oversampling
- Top five correlations all point at the same hypothesis → unambiguous

*Figure: MtD plot, correct key in red diverging after 10 traces.*

---

## Slide 6 — Step Two: Even Coefficients (1 min)

Odd coefficients now become **known constants** inside a composite function:

```
HW( fqmul(fqmul(s₁,u₁), zeta) + fqmul(h₀, u₀) )
```

| Target | Value | ρ | MtD |
|---|---|---|---|
| s⁰₀ | 72 | **0.323** | **179** |
| s⁰₂ | 2841 | 0.402 | 43 |

- Correlation ~**2.7× lower** than Step One — richer algebraic structure dilutes the leakage
- Same two functions replay across all 512 coefficients (`doublebasemul` is structurally identical)

---

## Slide 7 — Comparison & Countermeasures (1 min)

| Work | Level | Traces | Needs |
|---|---|---|---|
| **This work** | Kyber512 | **179** | — |
| Tosun & Savaş '24 | Kyber768 | 160 | zero-value filtering |
| Mujdei et al. '24 | Kyber768 | 200 | q² search |
| Yang et al. '23 | Kyber512 | 25–500 | ciphertext manipulation |

**Authors' countermeasures:**
1. **Don't reuse keys** — refresh faster than the MtD; 179 is their proposed ceiling for key reuse
2. Masking / shuffling — acknowledged, hedged, **not evaluated** (deferred to future work)

---

## Slide 8 — Takeaways & Critique (1 min)

**The transferable principle:**
> Any storage element holding a value dependent on exactly **one** secret coefficient is a first-order CPA target with a q-sized search. Fusing two or more secret-dependent products before the storage boundary forces q² or a low-SNR composite.

**Caveats worth stating:**
- "179 traces" is the **worst-case MtD**, not the campaign size — 500 were captured and used
- Comparison table puts their Kyber512 against three Kyber768 results
- Only 8 of 512 coefficients demonstrated; full recovery argued from structural regularity
- Decreasing-MtD trend explained only as "noise settles" — under-supported
- Entirely dependent on one implementation's instruction schedule

**For hardware:** fusion bought 0.87 → 0.32. Real, but a delay — not a countermeasure. Masking still carries the security argument.

---

## Cut List (in the deep dive, not the slides)

- Kyber parameter table (n, k, q, η, du, dv) — assume audience knows it
- Full nine-step pqm4 decryption sub-operation table — only sub-op 3/4 matters
- Complete `doublebasemul_asm` listing from the appendix — Slide 3's seven lines are enough
- Related-works taxonomy (their Table 1, 19 rows) — one sentence on profiled vs non-profiled instead
- Incremental-PCC mechanics — just define MtD in one line when it first appears
- The alternative r₁ attack function via `tmp3` — good verbal aside if asked, not a slide

## Timing Check
60 + 45 + 120 + 45 + 60 + 60 + 60 + 60 s ≈ **8.5 min**. Slide 3 is the one to protect if you run long; trim Slide 7's table to two rows.
