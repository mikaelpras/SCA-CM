# Paper 1 Deep Dive — The Kyber Attack (`Attack_FO`)

Ravi, Sinha Roy, Chattopadhyay, Bhasin. *Generic Side-channel attacks on CCA-secure lattice-based PKE and KEMs.* TCHES 2020(3), 307–335. Sections 5.1–5.3, Appendix E.

Everything below is either quoted from the paper or derived and numerically verified against Kyber's reference `poly_tomsg`. Derived material is marked **[derived]**.

---

## 1. Target parameters

Kyber512, **NIST Round 2** version — this matters, the parameters are not today's ML-KEM.

| Parameter | Value | Role in the attack |
|---|---|---|
| n | 256 | coefficients per polynomial |
| k | 2 | polynomials in the module (Kyber512) |
| q | 3329 | prime modulus |
| η | 2 | secret coefficients ∈ {−2,−1,0,1,2} → **5 candidates** |
| d_u | 10 | compression bits for **u** |
| d_v | 3 | compression bits for **v** |
| Total secret coefficients | n·k = 512 | |

The η = 2 is what makes the decision table have exactly 5 rows. **[derived]** In Round 3 / FIPS 203 ML-KEM-512, η₁ = 3, so a modern rerun would face **7** candidates per coefficient and need more queries.

---

## 2. Where exactly the leak sits

From `KEM.Decaps` (Alg. 1 in the paper):

```
2   c' = PKE.Decrypt(sk, ct)          <- secret used here
7   m' = c'                            (no ECC in Kyber)
9   r' = G(m', pk)                     <-- *** THE LEAK ***
10  ct' = PKE.Encrypt(pk, m', r')
11  if ct' == ct: return K = H(r' || ct')
15  else:         return K = H(z || ct')   <- attacker learns nothing here
```

The single most important structural point for your seminar:

> **The hash at line 9 is computed before the rejection at line 15.**
> The FO transform must hash the decrypted message in order to check it. By the time
> the device decides to reject, it has already leaked the thing the rejection was
> supposed to hide.

This is why the attack is called *generic*: it does not depend on the NTT, on the compiler, on constant-time coding, or on anything Kyber-specific. It depends only on the FO transform hashing `m'`.

---

## 3. Kyber decryption, reduced to what matters

From Alg. 5 in the paper:

```
KEM.Decrypt(ct, sk):
    (u, v) = DecodeCT(ct)        u ∈ R_q^k , v ∈ R_q
    s      = DecodeSK(sk)        s ∈ R_q^k
    w      = u × s               (in R_q = Z_q[x]/(x^n + 1))
    m'     = Poly_to_Msg(v − w)
```

### `Poly_to_Msg` written out

Kyber's reference implementation computes, per coefficient `a`:

```c
t = (((a << 1) + KYBER_Q/2) / KYBER_Q) & 1;
```

**[derived]** Working this out exactly for q = 3329:

> **bit = 1  ⟺  a ∈ [833, 2496]**

Equivalently `a` is nearer to q/2 than to 0. The boundaries q/4 = 832.25 and 3q/4 = 2496.75 fall between integers, so there is no ambiguity. Remember these two numbers — **833** and **2496** — the whole attack is built on them.

---

## 4. The chosen ciphertext

The attacker sets, in the *compressed* ciphertext:

- `u₀[0] = k_u` — first coefficient of the **first** polynomial of u; every other coefficient of every polynomial of u is 0
- `v[0]  = k_v` — every other coefficient of v is 0

Then `w = u × s = u₀ × s₀ + u₁ × s₁ = k_u · s₀` (since u₁ = 0 and u₀ is a constant polynomial), so `w[j] = k_u · s₀[j]`.

Subtracting:

```
m'[0] value  =  k_v − k_u · s₀[0]
m'[j] value  =     − k_u · s₀[j]        for j = 1 … n−1
```

which is the paper's Eqn. 12.

**A point the paper glosses over that is worth a slide:** because the ciphertext is entirely attacker-chosen and is *not* a valid LWE encryption, there is **no error term at all**. `m' = v − u·s` holds exactly, not approximately. This is why the attack tolerates decision margins of a single unit (see §6). Contrast this with decryption-failure attacks, where the noise is the whole point.

---

## 5. Constraint 1 — silencing the other 255 message bits

We need `m'[j] = 0` for all j ≥ 1, **whatever the secret is**. Since `s₀[j] ∈ {−2,…,2}`:

```
|−k_u · s₀[j]|  ≤  2·k_u        must stay outside [833, 2496]
```

**[derived]** The binding condition is `2·k_u ≤ 832`, giving

> **k_u ≤ 416**

The paper's three choices are k_u ∈ {101, 211, **416**} — so they use the maximum admissible value as one of them. Nothing larger is available.

The consequence is that exactly **one** message bit is ever non-zero. The device therefore hashes one of only two possible 32-byte inputs:

```
m = 00 00 00 … 00      (class O)
m = 01 00 00 … 00      (class X)
```

Two fixed inputs, forever. That is why a single pair of templates, built once, classifies all 2560 queries.

---

## 6. Constraint 2 — the decision table, fully verified

For each `(k_u, k_v)` pair, the response is `X` iff `(k_v − k_u·s) mod 3329 ∈ [833, 2496]`.

**[derived]** Reconstructing Table 4 of the paper from scratch. Every one of the 25 cells matches:

| s | (211, 416) | (211, 2913) | (101, 832) | (101, 2497) | (416, 1248) |
|---|---|---|---|---|---|
| **−2** | 838 → **X** | 6 → O | 1034 → **X** | 2699 → O | 2080 → **X** |
| **−1** | 627 → O | 3124 → O | 933 → **X** | 2598 → O | 1664 → **X** |
| **0** | 416 → O | 2913 → O | 832 → O | 2497 → O | 1248 → **X** |
| **1** | 205 → O | 2702 → O | 731 → O | 2396 → **X** | 832 → O |
| **2** | 3323 → O | 2491 → **X** | 630 → O | 2295 → **X** | 416 → O |

Each row is a unique 5-bit signature, so `RowCompare` recovers s unambiguously.

### The margins are razor-thin — deliberately

Look at the near-misses:

| Cell | Value | Boundary | Margin |
|---|---|---|---|
| (101, 832), s = 0 | 832 | 833 | **1** |
| (101, 2497), s = 0 | 2497 | 2496 | **1** |
| (416, 1248), s = 1 | 832 | 833 | **1** |
| (211, 416), s = −2 | 838 | 833 | 5 |
| (211, 2913), s = 2 | 2491 | 2496 | 5 |

Three decisions hinge on a margin of **one** in a modulus of 3329. This is not sloppiness — it is the only way to place a threshold between adjacent secret values, and it works precisely because §4's arithmetic is exact.

**Practical warning if you reproduce this:** any change to the rounding convention in `Decompress` or `Poly_to_Msg` flips these cells. Verify against your specific target binary, not against the spec.

---

## 7. Where the magic constants actually come from

The paper just states the numbers. **[derived]** They are not arbitrary — every one is forced by ciphertext compression.

Kyber transmits u and v compressed. The device decompresses with `Decompress_q(y,d) = round(q·y / 2^d)`. **An attacker can only inject values that survive this round-trip**, i.e. points on the compression grid.

| Constant | Grid | Index y | Check |
|---|---|---|---|
| k_u = 211 | d_u = 10 | y = 65 | round(3329·65/1024) = 211 ✓ |
| k_u = 101 | d_u = 10 | y = 31 | round(3329·31/1024) = 101 ✓ |
| k_u = 416 | d_u = 10 | y = 128 | round(3329·128/1024) = 416 ✓ |
| k_v = 416 | d_v = 3 | y = 1 | round(3329·1/8) = 416 ✓ |
| k_v = 832 | d_v = 3 | y = 2 | round(3329·2/8) = 832 ✓ |
| k_v = 1248 | d_v = 3 | y = 3 | round(3329·3/8) = 1248 ✓ |
| k_v = 2497 | d_v = 3 | y = 6 | round(3329·6/8) = 2497 ✓ |
| k_v = 2913 | d_v = 3 | y = 7 | round(3329·7/8) = 2913 ✓ |

Two consequences worth stating out loud:

1. **v is only 3-bit compressed in Kyber512, so k_v is confined to 8 possible values** — multiples of q/8. That is a severe restriction, and it is the real reason the table needs five columns rather than a tidy binary search.
2. **The grid points y = 2 and y = 6 land almost exactly on the decode boundaries** q/4 = 832.25 and 3q/4 = 2496.75. Those are the maximum-sensitivity choices available, which is exactly why the margin-of-1 cells appear there.

This also tells you how the attack must be re-tuned for other parameter sets: ML-KEM-512 uses d_v = 4, so the k_v grid becomes multiples of q/16 — finer, and therefore *more* attacker freedom, not less.

---

## 8. Is 5 queries per coefficient optimal? No.

**[derived]** Reading off the X-sets from §6:

| Column | X at s = | Interpretation |
|---|---|---|
| (211, 416) | {−2} | s ≤ −2 |
| (211, 2913) | {2} | s ≥ 2 |
| (101, 832) | {−2, −1} | s ≤ −1 |
| (101, 2497) | {1, 2} | s ≥ 1 |
| (416, 1248) | {−2, −1, 0} | s ≤ 0 |

**Every realizable query is a threshold test.** Here is why: the accept window [833, 2496] has width 1664, and the map s ↦ k_v − k_u·s steps by k_u ≤ 416. So the accept set in s-space is an interval of width ≥ 1664/416 = 4 — wider than the 5-element candidate range minus one. Intersected with {−2,…,2}, every such interval degenerates into a half-line.

I brute-forced all admissible (k_u, k_v) on the compression grid. Only **10 distinct response patterns exist**, and all of them are thresholds:

```
X at s = { }  {2}  {1,2}  {0,1,2}  {-1,0,1,2}  {-2}  {-2,-1}  {-2,-1,0}  {-2,-1,0,1}  {-2,-1,0,1,2}
```

Consequences:

- **Non-adaptive minimum = 4 queries.** You need the four boundaries −2|−1, −1|0, 0|1, 1|2. The paper's fifth column is redundant: (101, 2497) and (416, 1248) test the *same* boundary from opposite sides. Columns {1,2,3,4} or {1,2,3,5} suffice.
- **The paper spends 5 where 4 would do — a 20 % overhead**, i.e. 2560 traces instead of 2048.
- **Adaptive minimum = 3 queries** (binary search over 5 ordered values, ⌈log₂5⌉ = 3). Algorithm 3 in the paper is explicitly non-adaptive: it fires all r ciphertexts, *then* calls `RowCompare`.

This 5 → 3 gap is precisely the slack that the follow-up literature attacked. When you get to the shuffling papers and they quote query-count improvements over "the binary PC oracle attack," this is the baseline they mean.

---

## 9. Sweeping to the other 511 coefficients

Kyber works in `R_q = Z_q[x]/(x^n + 1)` — **anti-cyclic**. Multiplying by x^q gives `Anti_Rotr(s, q)`, whose coefficient 0 is `−s[n−q]`.

Set `u[p] = k_u · x^q`, everything else zero. Then (paper's Eqn. 14):

```
m'[0] = Poly_to_Msg( k_v − k_u · s_p[0]     )     if q = 0
m'[0] = Poly_to_Msg( k_v + k_u · s_p[n−q]  )     if 1 ≤ q ≤ n−1
```

- **q sweeps 0 … n−1** → walks along one polynomial
- **p sweeps 0 … k−1** → moves to the next polynomial of the module

Recovery order: `s₀[0], −s₀[n−1], −s₀[n−2], …, −s₀[1]`, then the same for s₁.

**Don't miss the sign.** For q ≥ 1 the table returns the **negated** coefficient. Since the centred binomial distribution is symmetric this doesn't break anything statistically, but you must flip the sign when reassembling the key. This is a classic off-by-a-sign bug if you implement it.

**Contrast with Round5 and Frodo:**
- Round5 has `deg(γ) − deg(φ) = −1`, so rotation is by `i−1` and `s[2]` is unreachable → brute-forced.
- Frodo has **no rotation property at all** (matrices, not polynomials). The set message bit lands at position `1 << (B·f)` for column f, so Frodo needs `n̄ = 8` separate template sets.
- **Kyber is the clean case:** the anti-cyclic ring gives full coverage with one template pair.

Total: **5 · n · k = 5 · 256 · 2 = 2560 queries**.

---

## 10. Why the EM signal is so strong

The paper says only that they observe "somewhere towards the end of the hash computation so that the diffusion of the modified bit induces enough differential behaviour."

**[derived]** Unpacking that for Kyber: G is SHA3-512 applied to `m' || H(pk)`, i.e. 64 bytes. SHA3-512 has rate 72 bytes, so this is a **single Keccak-f[1600] permutation, 24 rounds**. The two classes differ in exactly **one bit** of the input. After a few rounds of θ/ρ/π/χ/ι, avalanche means roughly half of the 1600 state bits differ — so the Hamming weights of the intermediate states are wildly different by the late rounds.

The attacker is therefore using **the hash function's own diffusion as a free amplifier**. A 1-bit difference is impossible to see at the moment it is produced; 24 Keccak rounds later it is unmissable. Reported t-test peaks reach roughly ±30–40 against a ±4.5 pass/fail threshold; they chose ±7 for point-of-interest selection.

This is a lovely inversion to put in the talk: the cryptographic strength of the hash (perfect avalanche) is exactly what makes the side channel loud.

---

## 11. Experimental numbers (Section 5.3)

| Quantity | Value |
|---|---|
| Implementation | pqm4, **optimized** Kyber512 |
| DUT | STM32F4DISCOVERY / STM32F407, Cortex-M4 @ 24 MHz |
| Acquisition | near-field EM probe, LeCroy 610Zi, 500 MSample/s |
| Template traces | 2 × 50 = 100 (built on the target itself) |
| Traces per iteration | 5 · 256 · 2 = **2560** |
| Success rate per iteration | ≈ **99 %** |
| Repetitions (majority vote) | 3 |
| Total traces | ≈ **7 680** |
| Time per iteration | 230 s |
| **Total time** | **650 s ≈ 10.83 min** |

The 2560-vs-7680 discrepancy between citing papers is just this: 2560 *queries*, 3× majority voting, 7.7k *traces*.

**Self-profiling detail (Section 3.4.3, applies here too):** to build the templates the attacker sets `k_u = 0`, so `u·s = 0` regardless of the key, and `k_v` alone dictates the decrypted message. **No clone device, no known-key profiling.** For Kyber specifically, pick k_v = 0 for class O and any k_v ∈ [833, 2496] on the 3-bit grid — e.g. k_v = 1248 — for class X.

---

## 12. Countermeasure implications, Kyber-specific

Section 6 of the paper is short and offers only one real answer: **mask the decryption/decapsulation**, citing Oder et al. [OSPG18], while conceding it is costly. The paper does **not** discuss shuffling or ciphertext-detection countermeasures — those come later, and are what your Papers 2, 4, 5 are about.

Why masking Kyber against *this* attack is expensive:

1. **Masking must extend past decryption into the FO transform.** The leak is at line 9, downstream of all polynomial arithmetic. Masking only the NTT and the pointwise multiply achieves nothing here.
2. **Masked Keccak is required**, since G operates on the shared message. That is the dominant cost.
3. **`Poly_to_Msg` needs a masked arithmetic-to-Boolean conversion**, and q = 3329 is **prime**, not a power of two. Saber's power-of-two moduli make A2B far cheaper — this is a large part of why the masking literature found Saber friendlier than Kyber.
4. **The ciphertext comparison must also be masked**, and must not unmask partial results (a separate attack surface).

Talking point: this paper is a *diagnosis*, not a cure. It ends by calling efficient masking "an interesting research direction that warrants immediate attention." Everything else on your reading list is the field trying to find something cheaper than the expensive answer.

---

## 13. Questions your audience will probably ask

**"Doesn't the implicit rejection protect Kyber?"**
No. Implicit rejection hides the *output*. The attack never looks at the output — the attacker model explicitly states "has no knowledge about corresponding outputs."

**"Would a ciphertext sanity check stop this?"**
The chosen ciphertexts are wildly non-random — one non-zero coefficient out of 512 — so in principle yes. That is exactly the detection countermeasure, and Paper 2 is about why it fails.

**"Does this break ephemeral Kyber / TLS?"**
No. It needs ~2560 adaptive decapsulations against a **long-term** key plus physical proximity. The threat model is a smartcard, HSM, or IoT device with a static key.

**"Is Kyber worse than Saber here?"**
No — the paper says Saber is structurally similar and the methodology "directly applies." Saber was not separately measured. Kyber was chosen as the demonstrator because module lattices generalise to both standard (Frodo) and ideal (NewHope) lattices.

**"How many of the six schemes were actually measured?"**
Three: Round5, LAC, Kyber. Saber and Round5(NE) are argued by structural similarity; NewHope is *simulation with a perfect oracle*; Frodo is a methodology sketch in Appendix F with no numbers.

---

## 14. Self-check exercises

Useful if you want to be certain you can defend the slides.

1. Show that k_u = 500 is inadmissible. *(2·500 = 1000 > 832, so a secret coefficient of ±2 would set a message bit at position j ≥ 1 and break the single-bit invariant.)*
2. Given the responses `(O, O, X, O, X)` in column order, what is s? *(−1.)*
3. Which two of the five columns are redundant with one another, and why? *((101, 2497) and (416, 1248) — one tests s ≥ 1, the other s ≤ 0; complementary tests of the same boundary.)*
4. The attacker measures `X` for the query `u₀ = 416·x^{200}`, `v = 1248`. Which coefficient does this tell you about, and with what sign? *(−s₀[56], since n − q = 256 − 200 = 56; the table value read is for the negated coefficient.)*
5. Why is a margin of 1 acceptable here but not in a decryption-failure attack? *(Chosen ciphertexts carry no LWE error term; the arithmetic is exact.)*
6. ML-KEM-512 uses η₁ = 3 and d_v = 4. How does each change affect the query count? *(η₁ = 3 → 7 candidates → 6 non-adaptive threshold queries; d_v = 4 → finer k_v grid → more attacker freedom, so this partially offsets. Net: worse for the defender than a naive count suggests.)*

---

## 15. Slide skeleton for the Kyber segment

| # | Slide | Core message |
|---|---|---|
| 1 | `KEM.Decaps` with line 9 boxed | The hash happens *before* the rejection |
| 2 | Chosen ciphertext + Eqn. 12 | One knob, one coefficient |
| 3 | Constraint k_u ≤ 416 | Only two possible messages ever reach G |
| 4 | **Decision table with values shown** | The "aha" slide — annotate the margin-of-1 cells |
| 5 | Compression grid | The constants are *forced*, not chosen |
| 6 | Threshold-test analysis, 5 → 4 → 3 | Sets the baseline every later paper improves on |
| 7 | Anti-cyclic sweep + sign flip | 512 coefficients from one template pair |
| 8 | Keccak diffusion as amplifier | Hash strength = side-channel loudness |
| 9 | 2560 / 7.7k / 10.8 min + honest caveats | Only 3 of 6 schemes measured |
| 10 | Masking cost for Kyber (prime q) | Hand-off to Paper 2 |
