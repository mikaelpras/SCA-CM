# Deep Dive: Will You Cross the Threshold for Me? — Generic Side-Channel Assisted Chosen-Ciphertext Attacks on NTRU-based KEMs

**Authors:** Prasanna Ravi, Martianus Frederic Ezerman, Shivam Bhasin, Anupam Chattopadhyay, Sujoy Sinha Roy
**Affiliations:** Temasek Laboratories & SCSE & SPMS, NTU Singapore · IAIK, TU Graz
**Venue:** IACR TCHES 2022, Vol. 2022, No. 1, pp. 722–761
**DOI:** 10.46586/tches.v2022.i1.722-761 · ePrint 2021/718 · CC-BY 4.0
**Code:** github.com/SCACCAONNTRU/SCACCAONNTRU

> **Sourcing note.** The TCHES 2022 full text was not retrievable (ePrint IP-blocked, TCHES PDF unreachable). This deep dive is built from the complete text of the authors' PQC 2021 companion paper — *"On Generic Side-Channel Assisted Chosen Ciphertext Attacks on Lattice-based PKE/KEMs: Towards key recovery attacks on NTRU-based PKE/KEMs"* (NIST 3rd PQC Standardization Conference) — which contains the full PC and DF oracle attacks on Streamlined NTRU Prime with all concrete parameters. **§11 lists what the TCHES version adds and what to verify from the PDF.**

---

## 1. Summary — What the Paper Is About

Side-channel-assisted chosen-ciphertext attacks on lattice KEMs were, until this work, an **LWE/LWR-only** story — Kyber, Saber, FrodoKEM. NTRU-based schemes had never been analysed this way, and it was an open question whether their very different arithmetic made them harder or easier targets.

This paper answers that. It adapts the **Jaulmes–Joux chosen-ciphertext attack** (CRYPTO 2000, against classical IND-CPA NTRU PKE) to the side-channel setting, and instantiates **three oracle types** — plaintext-checking, decryption-failure, and full-decryption — against IND-CCA secure NTRU and NTRU Prime.

The title is literal. NTRU Prime decryption computes `a = 3f·c ∈ Rq`, then zero-centres every coefficient in `(−q/2, q/2]`. The attack engineers a chosen ciphertext so that **exactly one coefficient of `a` crosses the ±q/2 threshold**. Because the reduction subtracts `q` — a prime, not a multiple of 3 — that one coefficient stops being ≡ 0 mod 3, so the subsequent `e = a mod 3` has exactly one nonzero entry. Every other coefficient stays a clean multiple of 3 and vanishes. Whether the threshold is crossed depends on a targeted secret coefficient. That's the oracle.

**Headline finding:** full key recovery in a few thousand queries on all parameter sets, and — the sentence that matters for the standardisation debate — **no considerable increase in attacker effort** compared to attacking LWE/LWR-based schemes. NTRU's structural difference buys essentially nothing against this attack class.

---

## 2. Existing SCA Method Used

| Element | Choice |
|---|---|
| Attack class | Chosen-ciphertext, side-channel-assisted oracle attacks (target-operation independent) |
| Classical ancestor | Jaulmes & Joux, CRYPTO 2000 — CCA on IND-CPA NTRU PKE |
| Side channel | Electromagnetic emanation |
| Leakage detection | Welch's t-test / TVLA (threshold ±4.5), used as a **feature-selection tool**, not an evaluation metric |
| Classification | Reduced templates from t-test PoIs + sum-of-squared-difference |
| Oracles instantiated | PC oracle, DF oracle, FD oracle |
| Traces per template | N = 10 |

---

## 3. Background: Why Streamlined NTRU Prime Is Different

Worth getting right, because every step of the attack depends on it.

**Ring:** `(Z/qZ)[x]/(xᵖ − x − 1)` — deliberately **non-cyclotomic**, chosen to reduce attack surface against structure-exploiting lattice attacks. Contrast with Kyber/Saber/NewHope, all cyclotomic.

**Parameters:** `(p, q, w)` with `2p ≥ 3w`, `q ≥ 16w+1`, `xᵖ − x − 1` irreducible. For **sntrup761**: `(761, 4591, 286)`.

**KeyGen:** `g ← GenSmall` (coeffs in {−1,0,1}), `ĝ = 1/g ∈ R₃`, `f ← GenShort` (small *and* weight-w). Public key `h = g/(3f) ∈ Rq`. Secret key `(ĝ, f)`.

**Encrypt:** `c = Round(h·r)` — round every coefficient to the nearest multiple of 3. **The rounding noise *is* the implicit message** `m ∈ R₃`; there's no explicitly added message polynomial. This matters enormously for the attack.

**Decrypt:**
```
a  = 3f·c ∈ Rq          , zero-centred in (−q/2, q/2]
e  = a mod 3
b' = e·ĝ ∈ R₃
if Weight(b') == w:  return r' = b'
else:                return r' = (1,1,…,1,0,0,…,0)
```

**Why it's correct:** `a = 3f·c = 3f·m + g·r`. Parameters guarantee every true coefficient already lies in `(−q/2, q/2]`, so mod-q reduction is a no-op. Then `a mod 3` kills the `3f·m` term, leaving `e = g·r`, and `e·ĝ = r`. **Perfectly correct — no decryption failures by design.**

That last point is the crux: a DF oracle attack on a scheme with *zero* natural decryption failures requires the attacker to manufacture them.

---

## 4. Attack Point & The Anchor Variable Concept

The paper introduces a useful piece of vocabulary: the **anchor variable** — the intermediate whose value is forced to depend on a targeted piece of the secret key.

| | LWE/LWR-based (Kyber, Saber, Frodo) | NTRU Prime (this work) |
|---|---|---|
| Anchor variable | the decrypted message `m` | the **internal variable `e`** |
| Attacker control | Direct — can build ciphertexts for any `m` | Indirect — `e` is internal, value not freely settable |
| Class values | Fixed: `m = 0` / `m = 1`, key-independent | `e = 0` / `e = ±1·xⁱ`, **and `i` depends on the secret key** |
| Search phase needed? | No | **Yes** — must hunt for a base ciphertext |
| Template reuse | Once per device | **Once per secret key** |

That bottom row is the sharpest practical difference and a good discussion point.

**Attack point in code:** line 4 of `PKE.Decrypt` (`e = a mod 3`) and the operations consuming `e` — specifically the **`Weight(b')` check**, which is where the EM leakage was harvested for the PC oracle.

---

## 5. Attack Model

| Aspect | Detail |
|---|---|
| Ciphertext control | **Chosen** — malformed ciphertexts of a specific algebraic structure |
| Physical access | EM probe on the decapsulation device |
| Implementation knowledge | Minimal — generic, target-agnostic |
| Known-key profiling | Not required |
| Template scope | **Per secret key** (unlike LWE/LWR, where it's per device) |
| Key setting | Static / reused key |
| Offline work | 2p brute-force at the end (1522 candidates for sntrup761) — negligible |

---

## 6. The Collision Mechanism (the technical core)

### 6.1 Naive version and why it fails

Take `c = k + k·h`. Then:
```
a = 3f·c = 3k·f + k·g
```
Coefficients of `f` and `g` are in {−1, 0, 1}. The maximum `|a[i]| = 4k` occurs only when `f[i] = g[i] = ±1` — the authors call this a **collision**.

Pick `k` with `3 | k`, `4k > q/2`, and `s·k < q/2` for `s ∈ [0,3]`. Then `a[i] > q/2` **only** on a collision → after centring, that coefficient alone loses its divisibility by 3 → `e[i] ≠ 0`, everything else zero.

**Problem:** probability of exactly one collision between `f` and `g` for sntrup761 is `≈ 8 × 10⁻⁴³`. Useless.

### 6.2 The fix — rotations

Generalise the ciphertext:
```
c = k₁·(x^i₁ + … + x^i_m) + k₂·(x^j₁ + … + x^j_n)·h  =  k₁·d₁ + k₂·d₂·h
```
Then `a = 3k₁·(d₁·f) + k₂·(d₂·g)`, where `d₁·f` is a **sum of rotations** of `f`.

Multiplication by `xⁱ` mod `(xᵖ − x − 1)` is not a clean rotation — it rotates *and* adds carry terms, producing coefficients in **{−2, −1, 0, 1, 2}**. Written `Rot_p(f, i)`.

A collision now requires *all* corresponding rotated coefficients to simultaneously hit ±2. Probability degrades rapidly as `(m, n)` grow — so `(m, n)` becomes a **tuning knob** for hitting "at most one collision, with high probability."

**Constraints on `(k₁, k₂)`:** `3 | k₁`, `3 | k₂`, and
```
3k₁·r + k₂·s  >  q/2   iff  r = 2m and s = 2n
              <  q/2   otherwise
```

**sntrup761 concrete values (PC oracle):** `(m, n) = (1, 3)`, `(k₁, k₂) = (102, 303)`, distances `(d_m1, d_m2) = (135, 168)`.

### 6.3 The rounding complication

Streamlined NTRU Prime ciphertexts have all coefficients divisible by 3, and only the quotient is transmitted. The attacker's crafted ciphertexts **aren't** multiples of 3, so they get rounded — introducing key-dependent noise `n = 3f·m'`.

For sntrup761 this noise is Gaussian, mean 0, **σ ≈ 57** against `q = 4591`. Small, but near the q/2 boundary it can produce:
- **False positive:** noise pushes a non-colliding coefficient over q/2
- **False negative:** noise suppresses a genuine collision

**Mitigation:** among all `(k₁, k₂)` satisfying the constraint, choose the tuple that **maximises the distances `d_m1`, `d_m2`** from the q/2 threshold — keep the signal values as far from the cliff edge as possible.

---

## 7. PC Oracle Attack

### Phase 1 — find the base ciphertext `c_base`

Randomly draw `(d₁, d₂)` until `e` has exactly one nonzero coefficient. Detecting this needs a side channel, because `e` is internal.

**The distinguishing signal:** if `e = 0` then `b' = 0` and `Weight(b') = 0`. If `e = ±1·xⁱ` then `b'` has uniformly random coefficients in {−1,0,1} and weight **≈ 500** for sntrup761. That is an enormous difference, trivially visible in EM from the weight-check operation.

**Procedure:** capture `T_O` (from `c = 0`, N = 10 traces) and `T_X` (candidate `c'`, N = 10). Welch's t-test between them. Peaks above ±4.5 → `e ≠ 0`.

**Cost:** ≈ **61 trials** on average × 10 traces = **≈ 610 traces** to find `c_base`.

### Phase 2 — key recovery

Build attack ciphertexts from `c_base`:
```
c_att = c_base + ℓ₃·x^u
```
Then at the colliding index `i`:
```
a[i] = δ + 3ℓ₃·Rot_p(f, u)[i]      (δ constant)
```
So `a[i]` is **linear in `β := Rot_p(f,u)[i] ∈ {−2,−1,0,1,2}`**. Choose `(ℓ₁, ℓ₂, ℓ₃)` so the threshold crossing happens iff `β = +2`; flip the sign of `ℓ₃` to test `β = −2`.

**Decision table for sntrup761** — four queries uniquely identify every candidate:

| β | (93,276,48) | (93,276,−48) | (78,237,78) | (78,237,−78) |
|---|---|---|---|---|
| −2 | O | X | O | X |
| −1 | O | X | O | O |
| 0 | O | O | O | O |
| +1 | X | O | O | O |
| +2 | X | O | X | O |

Vary the rotation index `u` to walk every coefficient of `f`.

**Classification:** t-test PoIs above a chosen threshold (they used ±7), reduced templates for classes O and X, sum-of-squared-difference match. **Single trace suffices, 100% success rate.**

### Recovering the exact key

Side channels give `Rot_p(f,u)[i]` for all `u`, but **not** the colliding index `i` nor the collision sign (+2/−2). Brute-force both: `2p = 1522` candidates for sntrup761. For each, check `f' ∈ R_sh` and verify a known ciphertext decrypts.

**Total cost:** 4 queries/coefficient × 761 = 3044 traces, plus the search and occasional re-runs → **≈ 4.35k traces**, ~100% success.

### The limitation that motivates the DF attack

For **every** PC-oracle attack ciphertext, `Weight(b') ≠ w` — it's 0 in class O and ≈2n/3 in class X. So decryption **always** returns the same constant `(1,1,…,1,0,…,0)`.

The O/X distinction therefore **does not propagate past decryption**. The attacker is confined to leakage from inside `PKE.Decrypt` and cannot touch the re-encryption procedure at all.

**This is the exact inverse of the Kyber situation**, where re-encryption is the entire attack surface.

---

## 8. DF Oracle Attack (the improvement)

**Idea:** instead of a fully malformed ciphertext, **perturb a valid one**:
```
c_pert = Round(h·r + c')
```
Now `e` is either `e_valid` (no failure → `r'_valid`) or `e_valid ± 1·xⁱ` (**single-coefficient error → decryption failure** → constant `r'_invalid`).

**Why this is better:** `r'` is the explicit input to re-encryption. The class distinction now propagates through the *entire* decapsulation — hashing, re-encryption, comparison. The attacker can harvest leakage from anywhere downstream.

**New noise term:** `n' = g·r + 3f·m'`, σ ≈ 53 — slightly wider than the PC case.

**Cleverer constraint (their Eq. 33):** choose `(k₁, k₂)` so that even *on* a collision, `a'[i]` sits **slightly below** q/2, in `[(q/2 − ε₁), (q/2 − ε₂)]`. Then crossing requires collision **AND** `n'[i] > ε₂`. This deliberately increases false negatives (cheap — just retry) while cutting false positives (expensive — wasted attack iterations). It also biases the search toward collisions where `g·r[i]` is large, which relaxes constraints in the key-recovery phase.

**sntrup761:** `(m, n) = (1, 3)`, `(k₁, k₂) = (93, 279)`, `(d_m1, d_m2) = (63, 342)`.

**Costs:**
- Finding `c_base`: ≈ **425 attempts** × 10 traces = **≈ 4250 traces** — seven times worse than the PC oracle's search
- Key recovery: same 4 queries/coefficient → 3044 traces
- **Total ≈ 8.1k traces**, ~100% success
- Target operation: the **encoding operation on the decrypted message inside re-encryption**

**Decision table** `(ℓ₁,ℓ₂,ℓ₃)`: (90,273,45), (90,273,−45), (78,231,78), (78,231,−78) — same O/X structure as the PC table.

---

## 9. Results Comparison

**PC oracle:**
| Target | Traces |
|---|---|
| sntrup761 (this work) | 4.35k |
| Kyber512 (Ravi et al. TCHES 2020, same setup) | 7.7k (3 iterations) / ≈2.56k single iteration |

sntrup761 costs roughly **2× Kyber512** — which the authors characterise as "not very significant."

**DF oracle:**
| Target | Queries |
|---|---|
| sntrup761 (this work) | ≈9.6k |
| Kyber512, Bhasin et al. (EM, ciphertext comparison) | 2¹⁷ queries → only reduces security to 2⁶⁵ |
| FrodoKEM, Guo et al. (timing) | 2³⁰ queries |

Their DF attack is **dramatically** more efficient than the LWE/LWR DF-oracle attacks — orders of magnitude.

---

## 10. Parts Needing Protection & Countermeasures

### Exposed
1. **The mod-q centring step** — the ±q/2 threshold is the entire mechanism
2. **The `Weight(b')` check** — the PC oracle's leakage source; a huge, easily-measured value difference (0 vs ≈500)
3. **`e = a mod 3` and everything consuming `e`** inside decryption
4. **The whole re-encryption path** — reachable once you upgrade to the DF oracle
5. **Static key reuse**

### Countermeasures (theirs)
**Masking** is the concrete answer, and the paper draws a sharp, actionable distinction:

- **Against the PC oracle:** masking the **decryption procedure alone is sufficient**, because leakage provably cannot propagate further.
- **Against the DF oracle:** the **entire IND-CCA decapsulation must be masked**.

Two hard problems they flag:
- Existing NTRU side-channel countermeasure literature targets **only the polynomial multiplier** in decryption. This work shows that's nowhere near enough.
- Streamlined NTRU Prime contains **non-linear operations — notably the weight check — that are not trivial to mask.**
- **No concrete, complete masking scheme for NTRU-based PKE/KEMs existed** at time of writing. They call this out as needing immediate community attention.

---

## 11. What the TCHES 2022 Version Adds — Verify From the PDF

The companion paper above covers **PC and DF oracles on Streamlined NTRU Prime (sntrup761) only**. Per the TCHES abstract, the journal version extends to:

- [ ] **NTRU** (the main finalist — HPS/HRSS), not just NTRU Prime. Low-level details differ; get the modified collision construction.
- [ ] **A third oracle: the FD (full-decryption) oracle** — completely absent from the companion. Get its construction and trace count.
- [ ] **All parameter sets** of both schemes, not just sntrup761. Get the full trace-count table.
- [ ] Whether the **countermeasure discussion** is expanded beyond the companion's one page.
- [ ] The `2447 traces` figure for **NTRU-HRSS n=701** — cited on pqc-forum by a follow-up work (Zhang et al., ICICS 2021) that improved it to ~1844 traces. Confirm that number and its context.

**Access:** tches.iacr.org article 9313 (open access), ePrint 2021/718, or the SCACCAONNTRU GitHub repo — the README confirms PC oracle code for NTRU, and PC + DF oracle code for NTRU Prime.

---

## 12. Critical Points for Discussion

- **The "not very significant" conclusion is the paper's real payload**, and it's a comparative claim resting on one PC-oracle data point (4.35k vs 2.56k) on one parameter set. Reasonable, but thinner than the confident phrasing suggests.
- **The DF oracle is worse, not better, on trace count** (8.1k vs 4.35k) — the search phase costs 4250 traces versus 610. Its advantage is *attack surface*, not efficiency. The paper's framing as an "improvement" is about what an attacker can target, not what it costs.
- **Internal trace-count inconsistency:** §3.2.1 reports ≈8.1k traces for the DF attack; the comparison paragraph says 9.6k. Worth flagging.
- **Per-secret-key template regeneration** is a genuine cost the paper mentions but doesn't quantify in the totals. Every new key means redoing the search.
- **Noise analysis is empirical.** The σ ≈ 57 / σ ≈ 53 figures and the `(m,n) = (1,3)` choice are "we empirically arrived at" — no closed-form optimisation.
- **The 2p brute-force is dismissed as negligible** (1522 for sntrup761), which is fair, but it does mean the SCA alone never yields the key.
- **No masked or protected target was attacked.** As with the parallel PC oracle paper, the masking conclusion is argued from outside.

---

## 13. Transfer to Hardware IP

- **Same class as the parallel PC oracle work: algorithmic, not microarchitectural.** The vulnerability is the q/2 centring threshold and the FO structure. Datapath restructuring, pipelining, or fusion does nothing.
- **But with a hardware-specific twist worth noting:** the PC oracle's leakage source is the **weight check** — a data-dependent, non-linear, hard-to-mask operation. In an ASIC that's a comparator tree or popcount whose activity scales with Hamming weight. The leakage is if anything *more* structural in hardware than in software.
- **The masking-scope result is directly actionable for IP design:** the required protection boundary depends on *which oracle* you're defending against. Mask decryption only → PC oracle blocked, DF oracle still open. That is a concrete, defensible architectural claim of the kind an IP security argument needs — and it generalises: **the protection boundary must enclose everything the class distinction can propagate to.**
- **Non-linear operations are the masking cost driver.** The weight check here is the analogue of any threshold, comparison, or rejection test in a masked design. For ML-DSA work, the parallel is the rejection-sampling bound check in signing — a non-linear, secret-dependent decision that masks poorly and is exactly the kind of operation this paper shows attackers gravitate toward.
- **Signatures have no PC/DF oracle in this form**, so the specific attacks don't port. The transferable finding is that *protecting the polynomial multiplier is the mistake this literature keeps documenting* — and the NTRU countermeasure literature made exactly that mistake for years before this paper.

---

## 14. Key References to Follow Up

- Jaulmes & Joux — *A Chosen-Ciphertext Attack against NTRU*, CRYPTO 2000 (the classical ancestor)
- Ravi, Roy, Chattopadhyay & Bhasin — *Generic Side-channel Attacks on CCA-secure Lattice-based PKE and KEMs*, TCHES 2020 (the LWE/LWR counterpart — the comparison baseline)
- Guo, Johansson & Nilsson — *A Key-Recovery Timing Attack... FrodoKEM*, CRYPTO 2020 (DF oracle origin)
- Bhasin, D'Anvers, Heinz, Pöppelmann & Van Beirendonck — *Attacking and Defending Masked Polynomial Comparison*, TCHES 2021 (the Kyber DF comparison point)
- Xu, Pemberton, Sinha Roy & Oswald — *Magnifying Side-Channel Leakage... Kyber*, IEEE TC 2022 (FD oracle)
- Bernstein et al. — *NTRU Prime: reducing attack surface at low cost*, SAC 2017 (why non-cyclotomic)
- Schamberger, Mischke & Sepulveda — *Practical Evaluation of Masking for NTRUEncrypt on ARM Cortex-M4*, COSADE 2019 (the insufficient prior countermeasure work)
- Goodwill, Jun, Jaffe & Rohatgi — TVLA methodology (the ±4.5 threshold)
