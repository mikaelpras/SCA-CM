# One-Slide Specs — The Three Attack Papers

Four sections per slide, as specified. Section ③ now shows an explicit **step pipeline** plus numbered detail.
Layout: **2×2 quadrants**, with ③ given the largest cell. Budget **3–4 min per slide**. *Italics* = speaking note, not slide text.

---

# SLIDE A — Enhanced Two-Step CPA on ML-KEM
### Kennaway, Hoang, Khalid, Rafferty & O'Neill · SECRYPT 2025 · Queen's University Belfast

**① ATTACK SUMMARY**
- Non-profiled CPA on Kyber512 **decapsulation polynomial multiplication**
- The contribution is an **assembly-scheduling observation**, not a new statistical method
- Full secret key in **179 traces** — no chosen ciphertexts, no profiling, no zero-value filtering

**② ATTACK POINT · MODEL · DEVICE**
- **Point:** `poly_frombytes_mul → doublebasemul_asm` in pqm4. `SMULTT` leaves an **isolated** `s_odd × u_odd` product alone in register `tmp2` — one unknown, q = 3329 hypotheses. `SMLABB` / `SMUADX` **fuse** their products → not directly attackable
- **Model:** non-profiled · **known** (not chosen) ciphertext · no manipulation · requires **static/semi-static key**
- **Device:** ChipWhisperer-Lite + CW308T-STM32F3 (Cortex-M4), pqm4 Kyber512, 500 traces @ 4× clock

**③ ATTACK METHOD — STEPS & RESULT**

> **Trace capture → Reference cross-validation → PoI identification → CPA #1 (odd coeffs) → Substitute as constants → CPA #2 (even coeffs) → Replay over 512 coeffs → Full key**
>
> *No profiling phase — the attacker never needs a clone device.*

1. **Capture** 500 traces over `doublebasemul_asm`, known ciphertexts, 4× clock
2. **Cross-validate** intermediates against a laptop reference implementation
3. **PoI identification** — sweep PCC per sample point → peaks at samples 153–164
4. **CPA #1** on `HW(fqmul(h₁,u₁))`, 3329 hypotheses → **odd** coefficients
5. **Substitute** the recovered odd coefficients as **known constants**
6. **CPA #2** on `HW(fqmul(fqmul(s₁,u₁),ζ) + fqmul(h₀,u₀))` → **even** coefficients
7. **Replay** steps 4–6 across all 512 coefficients (`doublebasemul` is structurally identical)

| CPA pass | Recovers | ρ | MtD |
|---|---|---|---|
| **#1** | odd coefficients | **0.87** | **10** |
| **#2** | even coefficients | 0.32 | **179** |

**④ WHAT MUST BE PROTECTED · COUNTERMEASURE NOTES**
- **Never let a single secret×public product reach a storage element unaccumulated** — that is the entire vulnerability
- Fusing into multiply-accumulate is worth ~**2.7×** (0.87 → 0.32) **for free** — but CPA #2 still succeeded. *A delay, not a countermeasure.*
- Symmetrise the odd/even paths; mask ŝ before the multiplier — masking carries the security argument
- Authors' own proposal: **refresh keys faster than 179 decapsulations**

---

# SLIDE B — Parallel PC Oracle Attacks on Kyber KEM
### Rajendran, Ravi, D'Anvers, Bhasin & Chattopadhyay · TCHES 2023 · NTU Singapore / KU Leuven

**① ATTACK SUMMARY**
- Generalises the plaintext-checking oracle from **1 bit per query → P bits per query**
- Motivation: the oracle spends **thousands of leaky points** to extract a **single bit**
- **2.89× – 7.65×** fewer queries than the binary baseline — and **shuffling provides no protection at all**

**② ATTACK POINT · MODEL · DEVICE**
- **Point:** the **FO re-encryption** `Re_Encrypt(m, pk)` — deterministic in m, so one bit flip changes everything downstream. Strongest observed leakage: **CBD sampling of the ephemeral secret r** (*attacker never needs to know this — the t-test finds it*)
- **Model:** **chosen** ciphertext · non-profiled · **no clone device required** · no implementation knowledge · static key · **pre-processing repeats per public key**
- **Device:** STM32F407VG (Cortex-M4) @ 24 MHz, pqm4 Kyber768, near-field EM

**③ ATTACK METHOD — STEPS & RESULT**

> **Choose P → Craft parallel ciphertexts → Build 2^P templates (t-test) → Query device → Multi-class classify → Descend P BDTs → Repeat ⌈256/P⌉·k times → Full key**
>
> *Steps 1–3 are pre-processing and repeat for every new public key. Steps 4–7 are the online attack.*

1. **Choose P** — trades template cost (2^P·T) against query cost (⌈256/P⌉·k·Q_set)
2. **Craft ciphertexts:** `u = 208·x⁰` (**k_u fixed**), `v = 208·t·Σᵢ₌₀^(P−1) xⁱ`
   → `mᵢ = F(s[i])` for i ∈ [0,P−1], `mᵢ = 0` otherwise. Fixed k_u keeps the **P traversals independent**
3. **Template pre-processing:** T = 5 traces per class, all 2^P values of m → normalise → **Welch t-test** → select PoIs → mean = template
4. **Query** the decapsulation device, capture one trace
5. **Classify** among 2^P classes — pairwise SSD in a **knockout tournament** (2^P − 1 matches)
6. **Descend the P BDTs** — new **min-*depth*** trees, which beat min-*entropy* trees for **P ≥ 3** *(a set's cost is set by its deepest member, not its average)*
7. **Repeat** until all 256·k coefficients are recovered

| Scenario | P | Queries | Gain |
|---|---|---|---|
| Binary baseline | 1 | 1776 | — |
| **With clone device** | 10 | **232** | **7.65×** |
| **Without clone** | 4 | **613** | **2.89×** |

**④ WHAT MUST BE PROTECTED · COUNTERMEASURE NOTES**
- **The entire FO re-encryption** — hashing, CBD sampler, encryption, comparison. Every operation downstream of m carries information about m
- **Shuffling is defeated outright** — the attack never touches message encoding, so shuffled costs exactly the same as unprotected
- Masking is the authors' conclusion — but **explicitly out of scope**; no masked target was attacked
- Designer framing: the query count *is* your safe key-refresh interval

---

# SLIDE C — Will You Cross the Threshold for Me? (NTRU-based KEMs)
### Ravi, Ezerman, Bhasin, Chattopadhyay & Sinha Roy · TCHES 2022 · NTU Singapore / TU Graz

**① ATTACK SUMMARY**
- **First** SCA-assisted chosen-ciphertext attacks on **NTRU-based** KEMs — the class had only been done for LWE/LWR
- Adapts **Jaulmes–Joux** (CRYPTO 2000) to the side-channel setting; instantiates **three oracles: PC, DF, FD**
- Verdict: **no considerable increase in attacker effort** vs. attacking Kyber. NTRU's different arithmetic buys nothing here.

**② ATTACK POINT · MODEL · DEVICE**
- **Point:** `a = 3f·c` is zero-centred in **(−q/2, q/2]**. Force **exactly one** coefficient across ±q/2 → centring subtracts **q** (prime, **not** a multiple of 3) → that coefficient alone survives `e = a mod 3`
- **Anchor variable is the *internal* `e`, not the message m** → less attacker control, a **search phase is required**, and **templates must be rebuilt per secret key** (vs. per device for LWE/LWR)
- **Model:** chosen ciphertext · non-profiled · static key · **Device:** Cortex-M4, EM, sntrup761 `(p,q,w) = (761, 4591, 286)`

**③ ATTACK METHOD — STEPS & RESULT**

> **PHASE 1 — Search:** *Craft candidate → Query → t-test vs reference → single collision? → repeat ~61×* **→ `c_base`**
> **PHASE 2 — Recovery:** *Build 4 attack ciphertexts → Query → Template classify O/X → Decision-table lookup → next rotation index* **→ all coefficients**
> **PHASE 3 — Offline:** *Brute-force 2p candidates → verify decryption* **→ exact key**

**Phase 1 — find the base ciphertext (≈610 traces)**
1. Capture reference `T_O` from `c = 0` (10 traces)
2. Draw random `(d₁,d₂)`, build `c = k₁·d₁ + k₂·d₂·h`, query, capture `T_X` (10 traces)
3. **Welch t-test** `T_O` vs `T_X` at ±4.5 → distinguishes `Weight(b') = 0` from **≈500** *(huge signal)*
4. Repeat — **≈61 trials** — until exactly one collision → `c_base`

**Phase 2 — key recovery (3044 traces)**

5. Build 4 attack ciphertexts `c_att = c_base + ℓ₃·xᵘ` → `a[i]` is **linear in `β = Rot_p(f,u)[i] ∈ {−2..2}`**
6. Query each, capture a **single trace**; classify **O/X** by template SSD
7. **Decision-table lookup** on the 4 O/X responses → uniquely determines β
8. Increment rotation index `u`, repeat for every coefficient

| Oracle | Total | Note |
|---|---|---|
| **PC** | ≈**4.35k** traces | + 2p = 1522 offline brute force |
| **DF** | ≈**8.1k** traces | perturb a *valid* ciphertext → failure **propagates into re-encryption** |

*The DF oracle costs **more** traces (its search phase alone is 4250 vs 610). The gain is attack **surface**, not efficiency.*

**④ WHAT MUST BE PROTECTED · COUNTERMEASURE NOTES**
- The **q/2 centring step** and the **`Weight(b')` check** — the latter is **non-linear and not trivial to mask**
- **Scoped masking result** *(the most useful output of the paper)*:
  - vs. **PC oracle** → masking **decryption alone is sufficient** — leakage provably cannot propagate further
  - vs. **DF oracle** → **the entire IND-CCA decapsulation** must be masked
- Prior NTRU countermeasure work protected **only the polynomial multiplier** — demonstrably insufficient
- **No complete masking scheme for NTRU-based KEMs existed** at time of writing

---

## Layout Notes

- **2×2 quadrants**, reading left-to-right: ① top-left, ② top-right, ③ bottom-left (widest), ④ bottom-right
- Render each **pipeline line as a horizontal arrow chain graphic** across the top of quadrant ③ — the numbered steps sit underneath as supporting detail. If space is tight, **the arrow chain alone can carry the section** and the numbered list becomes speaker notes
- Slide C's three-phase pipeline is the densest; consider stacking the three phase-arrows vertically
- The **bolded phrase in each ①** is your opening sentence; the **bolded line in each ④** is your closing one

## Pipelines At a Glance

| | Step chain |
|---|---|
| **A** | capture → PoI → **CPA #1** (odd) → substitute → **CPA #2** (even) → replay |
| **B** | choose P → craft → **template** → query → **classify** → BDT descent → repeat |
| **C** | **search** for c_base → **query** ×4/coeff → classify O/X → **decision table** → brute-force |

## One-Sentence Versions
- **A:** one assembly instruction stores a product it shouldn't
- **B:** the oracle was wasting thousands of leaky points to learn one bit
- **C:** push one coefficient over the ±q/2 line and the secret falls out
