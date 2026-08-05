# Paper 2 Deep Dive — Defeating Low-Cost Countermeasures

Ravi, Paiva, Jap, D'Anvers, Bhasin. *Defeating Low-Cost Countermeasures against Side-Channel Attacks in Lattice-based Encryption: A Case Study on Crystals-Kyber.*
**IACR TCHES 2024(2), 795–818.** ePrint 2023/1627. Code: `github.com/thalespaiva/sca_on_kyber_lcc`.

Target throughout: **Kyber768** (k = 3, n = 256, q = 3329, η₁ = η₂ = 2, d_u = 10, d_v = 4).

Derived material — my own computation, not in the paper — is marked **[derived]**.

---

## 1. Position in the seminar arc

Paper 1 (RRCB20) showed that the FO transform leaks the decrypted message, enabling PC-oracle key recovery. The field's response split in two:

- **Expensive answer:** mask the whole decapsulation (BGR+21, HKL+22). Several-fold runtime overhead.
- **Cheap answer:** don't mask — just *detect* the attack ciphertexts and react.

This paper is the demolition of the cheap answer. Note the authorship: **Prasanna Ravi is an author of Paper 1, of the survey that proposed the ciphertext sanity check (RCDB22), and of this paper breaking it.** Worth saying out loud in the talk — it is a healthy example of a research group attacking its own proposal.

The paper makes **three** contributions, and the third is easy to overlook but is arguably the most reusable:

1. Break the **ciphertext sanity check** (cheap, ~zero overhead for the attacker)
2. Break the **decapsulation failure check** (hard; requires a genuinely new attack)
3. A **new inequality solver** that beats Belief Propagation by >2× in required inequalities

---

## 2. The taxonomy you need on a slide first

| | Known-Ciphertext (KC) SCA | Chosen-Ciphertext (CC) SCA |
|---|---|---|
| Attacker control | knows ciphertext only | **chooses** ciphertext |
| Exploitable surface | decryption only (in practice: the polynomial multiply) | **entire decapsulation** incl. FO transform |
| Examples | PPM17, PP19, MWK+22 | RRCB20 (Paper 1), XPRO20, BDH+21 |
| Cost to defend | lower | higher — needs masking **and** shuffling |

CC attacks are the larger and more powerful family, which is why they are the focus.

The three CC sub-families:
- **PC oracle** — binary info about m (Paper 1, UXT+22, TUX+23)
- **FD oracle** — full decryption output (XPRO20)
- **DF oracle** — decryption-failure info (BDH+21)

---

## 3. The two countermeasures under attack

### Ciphertext Sanity Check (proposed in RCDB22)

**Observation it exploits:** PC- and FD-oracle attack ciphertexts are extremely sparse — one non-zero coefficient out of 768 in Paper 1's construction. Valid ciphertexts are LWE samples, uniform over [0, q].

**Mechanism:** compute mean µ(x) and standard deviation σ(x) of each polynomial of c₁ and c₂; reject if the variance is too low.

**Attractions:** ~5 % runtime overhead, and the rejection happens *before* decapsulation touches the ciphertext, so **the attacker observes no leakage at all**.

**Known gap even before this paper:** it does not stop DF-oracle attacks (BDH+21), which already use uniformly random invalid ciphertexts.

### Decapsulation Failure Check

**Observation it exploits:** every known CC attack uses **invalid** ciphertexts, which always fail re-encryption.

**Mechanism:** refresh the secret key on the *first* decapsulation failure. The attacker is then limited to a single trace per key.

**Why this one is genuinely strong:** it is protocol-level, essentially free, and it caps the attacker at one trace when all known attacks need 10²–10⁴. Breaking it requires abandoning invalid ciphertexts entirely — which is exactly what the paper does.

---

## 4. Attack A — Masking the ciphertext with the public key

### The idea in three lines

Take k = 1 and ignore compression. Public key (A, b = A·s + e), attack ciphertext (u_atk, v_atk) that would be rejected as too sparse. Send instead **(u_atk + A, v_atk + b)**:

```
Δ = (v_atk + b) − (u_atk + A)·s
  = v_atk + A·s + e − u_atk·s − A·s
  = v_atk − u_atk·s + e
```

The A·s terms cancel. You recover the original PC-oracle quantity **plus a small error e**. The ciphertext now looks like a genuine LWE sample and sails through the entropy test.

**The elegance is that nothing downstream changes.** The oracle responses are identical to the unprotected PC-oracle attack, so any existing CC attack can be ported unchanged. The paper explicitly notes they did *not* rerun the side-channel measurements, because the oracle is the same one already demonstrated in RRD+23.

### Two practical wrinkles

1. **Dimension mismatch.** u ∈ R_q^{k×1} but A ∈ R_q^{k×k}. Fix: use a single row **a\*** of A and the corresponding element **b\*** of b, with associated error **e\***.
2. **Compression.** You cannot inject arbitrary values; a\* + u_atk and b\* + v_atk get compressed. Result:

```
Δ = v_atk − u_atk·s + E ,    E = e − sᵀ·Δu + Δv
```

This is precisely the standard noisy-message equation with **r = 1, e₁ = 0, e₂ = 0**. Neat framing the paper offers: the *unmasked* attack is the same equation with **r = 0**.

### Why the success rate is only 57.8 % — [derived]

For a fixed mask, E is essentially a fixed unknown offset. The attack works iff E doesn't flip any decision. So: how much slack is there?

The paper reproduces RRD+23's Kyber768 attack table (its Table 1), y = 208, z ∈ {624, 832, 1040, 1248}. I verified all 20 cells against the reference `poly_tomsg` (bit = 1 ⟺ value ∈ [833, 2496]) — all match. Here are the values with **how much added error flips each cell**:

| z | s = −2 | s = −1 | s = 0 | s = 1 | s = 2 |
|---|---|---|---|---|---|
| 624 | 1040 → **1** (±208) | 832 → 0 (**+1**) | 624 → 0 (±209) | 416 → 0 (±417) | 208 → 0 (±625) |
| 832 | 1248 → **1** (±416) | 1040 → **1** (±208) | 832 → 0 (**+1**) | 624 → 0 (±209) | 416 → 0 (±417) |
| 1040 | 1456 → **1** (±624) | 1248 → **1** (±416) | 1040 → **1** (±208) | 832 → 0 (**+1**) | 624 → 0 (±209) |
| 1248 | 1664 → **1** (±832) | 1456 → **1** (±624) | 1248 → **1** (±416) | 1040 → **1** (±208) | 832 → 0 (**+1**) |

**Every single column contains exactly one cell with a flip margin of 1** — always the cell whose value is 832, sitting one unit below the decode threshold 833. So for one candidate value per query, the correctness of the oracle is decided by nothing more than **the sign of E**. A near-coin-flip, which is exactly why the measured success probability lands at 57.8 % rather than near 1.

This also explains the paper's restart strategy: rather than trying to control E, just draw a fresh mask. Expected restarts = (1 − p)/p = 0.422/0.578 = **0.73**, matching the paper.

**Fresh masks are cheap and plentiful:** a different row of A, a rotated version (X·a\*, X·b\*), or a scalar multiple (c·a\*, c·b\*).

### Why the obvious patch fails

"Reject if (u − A, v − b) is small" — defeated by adding **−A, −b** or **2A, 2b** or a rotated copy. There is no small set of forbidden offsets.

### Where the constants come from — [derived]

Cross-check with the Paper 1 deep dive: I showed there that with d_v = 3 (Kyber512), the k_v grid is confined to multiples of q/8 and every realizable query is a *threshold* test, forcing 5 queries where 4 would do.

Kyber768 has **d_v = 4**, so the v-grid is multiples of q/16 = 208.06 → {208, 416, 624, 832, 1040, 1248, …}. Verified: 624 = Decompress(3,4), 832 = Decompress(4,4), 1040 = Decompress(5,4), 1248 = Decompress(6,4), and y = 208 = Decompress(64,10). The finer grid lets RRD+23 hit **exactly the four optimal thresholds** (−2|−1, −1|0, 0|1, 1|2) — the 4-query optimum I derived for Paper 1. Nice continuity between the two talks.

---

## 5. Attack B — Key recovery from valid ciphertexts

This is the paper's centrepiece and the first CC-based SCA to use **only valid ciphertexts**.

### The bind

If ciphertexts must be valid, re-encryption succeeds, so no leakage from the FO transform is usable and the attacker is confined to the **decryption** procedure. Two obstacles:

1. **No divide-and-conquer.** Every coefficient of m′ depends on *all* coefficients of s, so classical CPA on individual key coefficients is impossible.
2. **Indirect information only.** The operations touching m′ don't manipulate s directly, so no leakage model gives you a key coefficient. A separate key-recovery step is mandatory.

### The way out: the noise carries the key

For a valid ciphertext, m′ = ⌊q/2⌉·m + Δm, with

```
Δm = ⟨r, e⟩ − ⟨s, e₁ + Δu⟩ + e₂ + Δv          (Eqn. 2)
```

**Δm is linear in the secret (s, e).** And because the attacker *encrypted the ciphertext themselves*, they know r, e₁, e₂, Δu, Δv and m. Everything is known except s and e.

Baseline sanity check: (s, e) has 2kn unknowns. Each ciphertext gives n coefficients of Δm. So **exact** knowledge of Δm for just 2k = 6 ciphertexts would recover the key by linear algebra. The attack cannot get exact values — only bounds — hence the need for many more ciphertexts and an inequality solver.

### Step 1 — Leak HW(m′[i])

Target: the **store instruction** in `PolySub` (computing v − ⟨s,u⟩), which writes each m′ coefficient to memory. Leakage model: Hamming weight of the stored 16-bit signed value.

Two implementation regimes:

| | Reference (`-O3`) | Assembly-optimized (HZZ+22) |
|---|---|---|
| Code | `ldrh` / `sub` / `strh` per coefficient | `ldmia` ×2, `usub16` ×5, `stmia` |
| Coefficients per store | 1 | **10** (packed 2 per register, 5 registers) |
| Isolation | clean, individually distinguishable CPA peaks | 10 coefficients leak simultaneously |
| Attacker's trick | none needed | choose m so the other 9 bits are **1**, giving uniform noise on the target |
| **HW classifier accuracy** | **91.1 %** | **32.0 %** |

Classifier: points of interest from CPA above threshold, then **Random Forest** (1.5k trees) over 17 HW classes. Chosen over more complex models for robustness and lack of hyperparameter tuning; precision/recall checked to rule out class-imbalance artefacts.

### Step 2 — The two's-complement asymmetry (the key insight)

This is the single cleverest idea in the paper and deserves its own slide.

When **m[i] = 0**, we have m′[i] = Δm[i], a small signed number near zero, stored as int16:

```
Δm[i] = +5  ->  0000 0000 0000 0101   HW = 2   (very low)
Δm[i] = −5  ->  1111 1111 1111 1011   HW = 14  (very high)
```

So **HW(m′[i]) is a near-perfect sign detector for Δm[i]**. Low HW ⟹ positive; high HW ⟹ negative.

When **m[i] = 1**, m′[i] ≈ 1664 ± Δm[i]. There is no sign-change at the representation boundary, so the HW distributions for positive and negative noise sit on top of each other (Fig. 4b) — **no information**.

**Consequence:** only the ~128 zero bits of a random 256-bit message are exploitable. This constraint drives everything downstream, including the shuffling attack in §7.

### Step 2b — From sign to two-sided bounds

Instead of extracting only a sign, build an empirical table of min/max m′[i] per Hamming weight (Table 2, from 100 000 decryptions). Each observation then yields **two** inequalities, ω ≤ Δm[i] ≤ Ω, which is strictly more information than a sign bit.

**The lazy-reduction wrinkle** — verified: efficient Kyber implementations skip full modular reduction, so m′[i] is not guaranteed to lie in [−q/4, q/4). For HW = 10 one would predict max m′[i] = −64 (since −64 = `1111111111000000`, HW 10). The table instead shows **+253**, because 253 + q = 3582 = `110111111110`, also HW 10. Both confirmed numerically. **This is why the bounds must be measured empirically rather than derived.**

**The spread filter — [derived].** The paper says it simulated over the spreads in Table 2 and picked **317** as the cutoff for which inequalities to keep. Reading the spread row against that threshold:

| HW | 0 | 1 | 2–11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|
| spread | 0 | 255 | 572–676 | 308 | 297 | 317 | 255 | 0 |
| kept? | ✓ | ✓ | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |

So the filter is exactly: **keep only extreme Hamming weights**, HW ∈ {0, 1, 12, 13, 14, 15, 16}. Seven of seventeen classes. Note the asymmetry — only HW 0–1 on the low side but HW 12–16 on the high side, because negative int16 values near zero all carry very high Hamming weight.

### Step 3 — Inequalities in (s, e)

Using the negashift identity (Eqn. 1), write

```
h(i,j) = [ negashift_i(r_j)  ‖  −negashift_i(e₁_j + Δu_j) ] ∈ Z_q^{2kn}
```

so that ⟨h(i,j), [e ‖ s]⟩ + e₂_j[i] + Δv_j[i] = Δm_j[i]. For each observation with m_j[i] = 0:

```
 h(i,j)·x + e₂_j[i] + Δv_j[i] − ω(i,j)  ≥ 0
−h(i,j)·x − e₂_j[i] − Δv_j[i] + Ω(i,j)  ≥ 0
```

Stack γ of these into H ∈ Z^{2γ×2kn}, w ∈ Z^{2γ}. A small solution to **Hx + w ≥ 0** is the key.

### How hard is the problem really? — [derived]

Variance decomposition of Δm for Kyber768 (CBD(2) has variance 1; compression error variance ≈ (q/2^d)²/12):

| Term | Variance | Share |
|---|---|---|
| ⟨r, e⟩ | 768 | 13.2 % |
| ⟨s, e₁⟩ | 768 | 13.2 % |
| ⟨s, Δu⟩ | 676 | 11.6 % |
| e₂ | 1 | 0.0 % |
| **Δv** | **3608** | **62.0 %** |
| **Total** | 5821 | σ = **76.3** |

Consistent with the paper's stated σ(Δm) ≈ 79. Two observations worth making in the talk:

- **Δv dominates the noise**, contributing nearly two-thirds of the variance. But Δv is *known* to the attacker and is subtracted out in Eqn. 10. The genuinely unknown part has **σ ≈ 47**.
- Wider ciphertext compression on v (larger d_v ⇒ smaller Δv) would therefore make the *raw* noise smaller but would not change the attacker's effective problem much, since Δv was never the obstacle.

---

## 6. The greedy solver (Section 6.2.2)

Prior art: Belief Propagation (Pessl–Prokop PP21, Hermelink et al. HPP21, Delvaux Del22), developed for fault attacks. Its limits: poor scaling in the number of inequalities, heavy RAM use.

**Algorithm 7 (greedy):** start at x = 0 ∈ Z^{2kn}. Each iteration scores every action (j, v) — "add v to coordinate j", for v ∈ {−η,…,η} — and applies the α_it best ones. α_it starts large and decays exponentially for fast convergence.

**Algorithm 8 (score):** apply the candidate action, then for every unsatisfied inequality subtract the numerical distance from satisfaction:

```
t = Hx + w + v·H_j
score = − Σ_{i : t[i] < 0} |t[i]|
```

That is the whole heuristic. No message passing, no probability distributions, no graph.

**Why it wins:**

| | Belief Propagation | Greedy search |
|---|---|---|
| Inequalities needed | baseline | **< half** |
| Robustness to SCA noise | degrades fast beyond σ ≈ 0.5 | usable to σ ≈ 2.0 |
| Runtime at σ = 0.5 | > 10 minutes | **< 20 seconds** |
| Masked case | too many inequalities to be feasible | works |

The BP curve in Fig. 5 could not even be extended past σ = 0.5 because the simulations took too long.

**Presentation angle:** this is a nice reminder that a well-chosen simple heuristic can beat a principled probabilistic method when the problem is over-determined. It is also the most portable piece of the paper — the solver applies to any attack that yields linear inequalities in (s, e), including fault attacks.

---

## 7. Experimental results

| Setting | HW accuracy | Traces for full key recovery | Notes |
|---|---|---|---|
| Reference (`-O3`) | 91.1 % | **≈ 5200** measured (8 zero bits) | |
| Reference (`-O3`) | 91.1 % | **≈ 325** *extrapolated* (128 zero bits) | see caveat |
| Assembly-optimized | 32.0 % | **≈ 7800** (13 coefficients) | 24× the reference |
| Shuffled | assumed **perfect** | **> 8 000 000** ciphertexts | proof of concept only |
| First-order masked (mkm4) | 94 % | ~10⁵ (Fig. 7, σ = 0) | see caveat |

Target: STM32F407 @ 24 MHz; Langer RF-U 5-2 near-field EM probe; LeCroy 610Zi at **1.25 GSam/s**; 30 dB pre-amp; 48 MHz low-pass filter. Ten random secret keys recovered per configuration.

**Verified arithmetic:** 5200 × 8/128 = 325 exactly, and 7800/325 = 24.0 exactly.

### Caveats to state honestly in the talk

1. **The headline 325 is extrapolated, not measured.** They measured 5200 traces using 8 zero bits and scaled by 16×. It is a reasonable extrapolation — 128 is the *average* number of zero bits in a random message and Fig. 5's simulations agree — but it is not an experimental number.
2. **The assembly-optimized result is the honest one** and it is 24× worse. The gap comes from both fewer usable coefficients (1 in 10) and a much weaker classifier (32 % vs 91 %).
3. **The masked result used a rewritten implementation.** They replaced mkm4's assembly `PolySub` with C compiled at **`-O0`** "for simplicity." That is why the masked HW accuracy (94 %) *exceeds* the unprotected optimized one (32 %). The comparison is not apples-to-apples, and a real masked target using the shipped assembly would be considerably harder.
4. **The shuffled result assumes a perfect HW classifier** and still needs 8 million ciphertexts with 16 % wrong inequalities. The paper itself calls it a proof of concept and leaves improvement to future work.

---

## 8. Attacking the shuffled implementation (§7)

**The problem shuffling creates:** the attacker sees a permuted sequence of Hamming weights and no longer knows which observation corresponds to a zero message bit.

**The trick:** the attacker *chooses the message* (valid ciphertexts!), so choose m with only **θ** zero bits. If exactly θ extreme Hamming weights appear in the observed sequence (w ≤ 1 or w ≥ 15), they must correspond to the zero positions — even though you don't know which is which, and that's fine, because you can attribute the extreme HW value to each of them.

**Why θ = 2 fails:** you need intervals tight enough to exclude every HW that a m[i] = 1 coefficient could produce, which makes such observations rare. With θ = 2 there are only C(256,2) = 32 640 distinct ciphertexts — not enough to accumulate rare events.

**Relaxation:** use θ = 4 and accept an inequality whenever 2 extreme values appear, treating all m[i] = 0 indices as extreme. This **knowingly generates wrong inequalities** — 16 % of the 300 000 used were wrong — and the greedy solver absorbs them. Cost: > 8 million ciphertexts.

**Takeaway for the seminar:** shuffling is a real speed bump (325 → 8 000 000, roughly 4.5 orders of magnitude) but not a wall. This is the finding your Papers 4 and 5 have to be read against.

---

## 9. Attacking the masked implementation (§8)

First-order masked Kyber768 from **mkm4** (HKL+22). `MaskedPolySub` computes v − ⟨s,u⟩ over arithmetic shares w = w₀ + … + w_{d−1}: `poly_sub` computes v − w₀, and the other shares are simply negated. `poly_sub` is essentially the unprotected routine.

**Why it breaks:** the attacker recovers HW of *each individual arithmetic share* of m′[i], not the unmasked value. CPA (Fig. 6) shows clean leakage from all shares.

**Adapting the inequality construction:** replace the 1-D min/max table with **two 17 × 17 tables** indexed by (HW of share 0, HW of share 1), giving max and min of m′[i]. Built from 100 000 decryptions of the mkm4 code.

**A detail worth highlighting:** these tables need **no side-channel measurement at all** — they depend only on the implementation, not the device. The attacker builds them offline from public source code. That significantly lowers the profiling burden.

---

## 10. Errata in the paper

Flagging these because a close reader will hit them and get confused. (Figure numbering also differs between the ePrint and the published TCHES version — the CPA figure is Fig. 2 in ePrint, Fig. 3 in TCHES.)

| Location | Issue |
|---|---|
| §1, contribution 1(a) | "works with probability **0.57 %**" — should be **57.8 %**, per §3 |
| §2.2.1 | "key generation procedure shown in **Algorithm 3**" — should be Algorithm 1 |
| §2.4.2 | "for chosen-ciphertexts, the message m is **not** a key-dependent variable" — the "not" reverses the intended meaning and contradicts the preceding sentence |
| §4.2 | "Δm = **⟨e, e⟩** − ⟨s, e₁ + Δu⟩ …" — should be ⟨r, e⟩, cf. Eqn. 2 |
| §4.2 | The paragraph beginning "As we are limited to a KC attack…" appears **twice**, nearly verbatim |
| §4.2, Fig. 4 | Uses **Δn[i]** where the rest of the paper uses Δm[i] |
| Fig. 2 caption | Describes three panels (O0, O3, assembly); only two exist. §5.3 refers to "Figure 2(c)", which is panel (b) |
| Alg. 7 input | Declares H ∈ Z^{γ×2kn}; §6.2.1 defines H ∈ Z^{2γ×2kn} (two rows per observation) |

---

## 11. Critical assessment

**Strengths**

- The public-key masking trick is genuinely elegant and essentially free — it requires no change to the downstream attack at all.
- The valid-ciphertext attack is a real conceptual first and re-opens a category everyone had written off.
- The two's-complement HW asymmetry is a beautiful observation about *implementation representation*, not about Kyber.
- The greedy solver is the most reusable artefact: simpler, faster, and more noise-tolerant than BP, and applicable to fault attacks too.

**Weaknesses to raise in discussion**

- The headline number (325 traces) is extrapolated; the measured figure is 5200.
- The masked evaluation swapped in unoptimized C at `-O0`, which materially favours the attacker.
- The shuffled attack assumes a perfect classifier and still needs 8 M ciphertexts — nowhere near practical.
- Attack A was never validated on hardware; only simulated, on the reasonable grounds that the oracle is identical to RRD+23's.
- The paper does not propose a fix. Its constructive contribution is negative: "don't rely on detection alone."

**A framing question worth putting to the room:** Attack B uses valid ciphertexts and exploits only the decryption procedure. Is it really a *chosen*-ciphertext attack, or is it a known-ciphertext attack with the added freedom to choose the plaintext? The paper itself slips and writes "as we are limited to a KC attack." The distinction matters, because if it is essentially KC, then the defender's job is narrower than the CC framing implies — you must protect m′ in decryption, but you may not need to protect the whole FO transform against *this* particular attack.

---

## 12. Links to the rest of your reading list

| Paper | Connection |
|---|---|
| **1 (RRCB20)** | The attack that made detection countermeasures necessary. Attack A restores it verbatim; the Kyber768 table here is the 4-query optimum I derived in the Paper 1 notes |
| **3 (PQ-Hammer)** | Also bypasses masking/shuffling, but by faulting the `fail` variable rather than by side channel. Complementary route to the same conclusion |
| **4, 5 (shuffling)** | §7 is the prior result they must beat. Shuffling costs the attacker ~4.5 orders of magnitude but does not stop them |
| **6 (redundant number representation)** | **The strongest link.** Attack B's Step 2 depends entirely on the *two's-complement* representation making small negatives high-weight. A redundant number representation changes the HW profile of stored values and could plausibly kill this asymmetry. Worth setting up as the closing question of the seminar |

---

## 13. Slide skeleton (~6 min)

| # | Slide | Message |
|---|---|---|
| 1 | Masking is expensive → detection countermeasures | Why cheap defences are attractive |
| 2 | The two countermeasures, side by side | Sanity check (pre-decapsulation) vs failure check (protocol-level) |
| 3 | Public-key masking, 4 lines of algebra | A·s cancels; entropy restored |
| 4 | Margin table + "57.8 % is a coin flip" | Why the success rate isn't ~100 %, and why restarting works |
| 5 | The bind: valid ciphertexts ⇒ decryption only | Sets up the pivot |
| 6 | **Δm is linear in (s, e) and everything else is known** | The pivot |
| 7 | **+5 vs −5 in int16: HW 2 vs HW 14** | The cleverest slide in the paper |
| 8 | Table 2 + lazy reduction (−64 vs +253) | Why bounds must be measured |
| 9 | Inequalities → greedy solver vs BP | 2× fewer inequalities, 30× faster |
| 10 | Results table with the four caveats | Be honest about 325 vs 5200 |
| 11 | Verdict: detection is not standalone protection | Hand off to Papers 4/5 on shuffling |

---

## 14. Self-check exercises

1. Why does the public-key mask cancel exactly? *(Both u and v are shifted by the same LWE relation; the A·s term appears in both and subtracts out, leaving only e.)*
2. In the Kyber768 table, which secret value is at risk for the query z = 1040, and why? *(s = 1: value 832, one unit below the 833 threshold. Any E ≥ +1 flips it.)*
3. Why does the attack extract nothing from message bits equal to 1? *(m′[i] ≈ 1664; adding ±Δm doesn't cross a two's-complement sign boundary, so the HW distributions for positive and negative noise coincide.)*
4. The secret (s, e) for Kyber768 has 2kn = 1536 unknowns. Why isn't 6 ciphertexts enough? *(Six would suffice for exact Δm values; HW leakage gives only bounds, so you need many loose inequalities instead of few exact equations.)*
5. Why is the spread filter set at 317, and what does it select? *(It admits only HW ∈ {0,1,12,13,14,15,16} — the extreme Hamming weights, where the min/max bounds are tightest.)*
6. Why is the masked HW classifier (94 %) more accurate than the unprotected optimized one (32 %)? *(Not a property of masking — they replaced the assembly PolySub with C at `-O0`. Methodological artefact.)*
7. What single implementation change would most directly undermine Step 2? *(Anything that removes the low-HW/high-HW asymmetry between small positive and small negative values — e.g. storing m′ in a redundant or offset representation. This is the bridge to Paper 6.)*
