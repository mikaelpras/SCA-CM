# Slide Outline — Will You Cross the Threshold for Me?
**Ravi, Ezerman, Bhasin, Chattopadhyay, Sinha Roy · TCHES 2022 · target: ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** Will You Cross the Threshold for Me? — Generic SCA-Assisted CCA on NTRU-based KEMs
Ravi et al., TCHES 2022 (NTU Singapore / TU Graz)

- Until this work, SCA-assisted chosen-ciphertext attacks were an **LWE/LWR-only** story (Kyber, Saber, Frodo)
- Adapts the **Jaulmes–Joux** CCA (CRYPTO 2000) on classical NTRU to the side-channel setting
- Instantiates **three oracles** — PC, DF, FD — on NTRU and NTRU Prime
- Full key recovery in a few thousand queries, all parameter sets

> **The headline:** *no considerable increase in attacker effort* vs. LWE/LWR schemes. NTRU's different arithmetic buys essentially nothing here.

---

## Slide 2 — Streamlined NTRU Prime, Just Enough (45 s)

Ring `(Z/qZ)[x]/(xᵖ − x − 1)` — **non-cyclotomic** by design. sntrup761: `(p,q,w) = (761, 4591, 286)`

```
Decrypt:  a  = 3f·c ∈ Rq   ← zero-centred in (−q/2, q/2]
          e  = a mod 3
          b' = e·ĝ ∈ R₃
          if Weight(b') == w: return b'
          else: return (1,1,…,1,0,…,0)
```

- `a = 3f·m + g·r`; params keep all coefficients inside (−q/2, q/2] → mod-q is a no-op → **perfectly correct, zero decryption failures by design**
- Message is **implicit**: it's the rounding noise in `c = Round(h·r)`

---

## Slide 3 — The Threshold Trick ★ (2 min — money slide)

**The title is literal.** Force exactly one coefficient of `a` across ±q/2.

- Centring subtracts **q** — a prime, **not** a multiple of 3
- → that one coefficient stops being ≡ 0 mod 3 → survives into `e = a mod 3`
- → every other coefficient stays a multiple of 3 and **vanishes**
- → whether it crosses depends on **a targeted secret coefficient**

**Getting there — "collisions":**
```
c = k₁·d₁ + k₂·d₂·h    →   a = 3k₁·(d₁·f) + k₂·(d₂·g)
```
- Naive `c = k + k·h`: single-collision probability ≈ **8 × 10⁻⁴³**. Useless.
- Fix: `d₁·f` is a **sum of rotations**; mult. by `xⁱ` mod (xᵖ−x−1) rotates **and adds carries** → coeffs in {−2,…,2}
- Collision needs *all* rotated coeffs at ±2 → probability tunable via `(m, n)`
- sntrup761: `(m,n) = (1,3)`, `(k₁,k₂) = (102,303)`

*If time: rounding adds key-dependent noise σ ≈ 57 vs q = 4591 → pick (k₁,k₂) maximising distance from the q/2 cliff.*

---

## Slide 4 — The Anchor Variable Shifts (60 s)

| | LWE/LWR (Kyber, Saber) | NTRU Prime |
|---|---|---|
| Anchor variable | decrypted message `m` | **internal variable `e`** |
| Attacker control | direct — any `m` | indirect — `e` not freely settable |
| Classes | `m = 0/1`, **key-independent** | `e = 0` / `±1·xⁱ`, **`i` depends on the key** |
| Search phase | none | **must hunt for a base ciphertext** |
| Template reuse | once per **device** | once per **secret key** |

> This is the real structural difference between attacking NTRU and attacking Kyber.

---

## Slide 5 — PC Oracle Attack (75 s)

**Phase 1 — find `c_base`:** draw `(d₁,d₂)` until `e` has one nonzero coefficient.
- Distinguisher: `e = 0` → `Weight(b') = 0`; `e = ±1·xⁱ` → `Weight(b') ≈ 500`
- Welch's t-test (±4.5), N = 10 traces → **≈61 trials ≈ 610 traces**

**Phase 2 — key recovery:** `c_att = c_base + ℓ₃·xᵘ` → `a[i] = δ + 3ℓ₃·β`, linear in `β = Rot_p(f,u)[i] ∈ {−2..2}`

| β | (93,276,48) | (93,276,−48) | (78,237,78) | (78,237,−78) |
|---|---|---|---|---|
| −2 | O | X | O | X |
| −1 | O | X | O | O |
| 0 | O | O | O | O |
| +1 | X | O | O | O |
| +2 | X | O | X | O |

**4 queries per coefficient**, single trace each, 100% success → **≈4.35k traces total**
(+ 2p = 1522 brute-force for the collision index and sign)

---

## Slide 6 — Why They Needed a Second Oracle (60 s)

**PC oracle limitation:** for *every* attack ciphertext, `Weight(b') ≠ w` (0 or ≈2n/3) → decryption **always** returns the same constant.

→ The O/X distinction **cannot propagate past decryption.** Attacker confined to `PKE.Decrypt`.
→ **Exact inverse of Kyber**, where re-encryption *is* the attack surface.

**DF oracle fix:** perturb a **valid** ciphertext — `c_pert = Round(h·r + c')`
- `e_valid` (no failure) vs `e_valid ± 1·xⁱ` (**manufactured** decryption failure)
- `r'` feeds re-encryption → distinction now propagates through the whole decapsulation

**Cost: ≈8.1k traces — *worse* than the PC oracle.** The gain is attack *surface*, not efficiency.
(Search phase alone: 4250 traces vs 610.)

---

## Slide 7 — Results & Comparison (60 s)

**PC oracle**
| Target | Traces |
|---|---|
| sntrup761 (this work) | 4.35k |
| Kyber512 (same setup, TCHES 2020) | ≈2.56k (single iteration) |

→ ~2× Kyber512. Authors: *"not very significant."*

**DF oracle**
| Target | Queries |
|---|---|
| sntrup761 (this work) | ≈9.6k |
| Kyber512, Bhasin et al. | 2¹⁷ → only reduces security to 2⁶⁵ |
| FrodoKEM, Guo et al. (timing) | 2³⁰ |

→ Orders of magnitude better than the LWE/LWR DF-oracle attacks.

---

## Slide 8 — Countermeasures & Takeaways (75 s)

**The scoped masking result — the most useful output:**

| Defending against | Required protection boundary |
|---|---|
| PC oracle | **decryption alone is sufficient** (leakage provably can't propagate) |
| DF oracle | **the entire IND-CCA decapsulation** |

Two hard problems they flag:
- Prior NTRU countermeasure work protects **only the polynomial multiplier** — nowhere near enough
- The **weight check** is non-linear and **not trivial to mask**; no complete masking scheme for NTRU-based KEMs existed

**Caveats worth stating:**
- The "not significant" claim rests on **one** PC-oracle comparison, one parameter set
- Internal inconsistency: DF cost given as 8.1k in results, 9.6k in the comparison
- Per-secret-key template regeneration isn't in the headline totals
- Noise analysis and `(m,n)` choice are empirical, not optimised
- No masked target was actually attacked

---

## Cut List (in the deep dive, not the slides)

- Full KeyGen / Encrypt / FO-transform pseudocode — Slide 2's decrypt block is enough
- The `Rot_p(d,i)` expansion and carry-term algebra — state the {−2..2} range, skip the derivation
- False-positive / false-negative analysis and the `(d_m1, d_m2)` distance optimisation — one line on Slide 3 if time
- The DF oracle's modified Eq. 33 constraint (deliberately keeping `a'[i]` *below* q/2) — elegant, but a verbal aside
- Second decision table for the DF oracle — structurally identical to Slide 5's
- Why non-cyclotomic (Bernstein et al. rationale) — one clause on Slide 2
- The 2p brute-force detail — one parenthesis on Slide 5

## Timing Check
60 + 45 + 120 + 60 + 75 + 60 + 60 + 75 s ≈ **9.1 min** — slightly over. Trim Slide 7 to the PC table only (−30 s), or drop Slide 4's last two rows.
Slide 3 is the one to protect. Slide 6 is the second priority — the PC→DF motivation is the paper's actual narrative arc.
