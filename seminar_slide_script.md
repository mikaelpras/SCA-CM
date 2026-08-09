# Kyber SCA Seminar — Slide-by-Slide Script

49 slides · ~53 minutes · Paper 5 pending (PDF not obtained)

Each slide gives: **On slide** (what the audience reads) · **Visual** · **Say** (narration) · time.
Keep on-slide text short — the narration carries the detail.

---
---

# INTRO — Kyber / ML-KEM (5 slides, 5:00)

### I-1 — What ML-KEM is *(1:00)*

**On slide**
- ML-KEM (FIPS 203), from CRYSTALS-Kyber — the only KEM NIST standardized
- Security: **Module-LWE**
- `R_q = Z_q[X]/(X²⁵⁶ + 1)`, `q = 3329`

| | 512 | 768 | 1024 |
|---|---|---|---|
| k | 2 | 3 | 4 |
| η₁ | 3 | 2 | 2 |
| (d_u, d_v) | (10,4) | (10,4) | (11,5) |

**Visual:** parameter table only.

**Say:** "One prime, one ring, three parameter sets — only the module rank changes. One warning for anyone reading the older literature: Round-2 Kyber512 used η₁ = 2 and d_v = 3. That changes how many candidate values a secret coefficient has, which changes attack costs. Papers from 2020 assume the old numbers."

---

### I-2 — Two layers *(0:45)*

**On slide**
- Inner: **IND-CPA PKE** — the lattice math
- Outer: **IND-CCA KEM** — Fujisaki–Okamoto transform
- The inner scheme is **not** CCA-secure on its own

**Visual:** two nested boxes, CPA inside CCA.

**Say:** "Hold both layers in your head for the next hour. The inner PKE falls trivially to chosen-ciphertext attacks — that's not a flaw, it was never meant to resist them. The FO transform is what buys CCA security. Almost everything physically interesting about Kyber happens at the boundary between those two layers."

---

### I-3 — The IND-CPA PKE *(1:15)*

**On slide**
```
KeyGen    t̂ = Â∘ŝ + ê            pk = (t̂, ρ)   sk = ŝ
Encrypt   u = NTT⁻¹(Âᵀ∘r̂) + e₁
          v = NTT⁻¹(t̂ᵀ∘r̂) + e₂ + ⌈q/2⌋·m
Decrypt   m′ = v′ − NTT⁻¹(ŝᵀ ∘ NTT(u′))
```
> **m′ = ⌈q/2⌋·m + Δm**,  Δm = ⟨e,r⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv

**Visual:** the three algorithms, with the Δm equation boxed at the bottom.

**Say:** "Message bits are scaled to half the modulus, then buried in noise. Decryption subtracts the mask and rounds. Everything on this slide is standard — but look at the boxed equation. Two things matter later. First, Δm is **linear in the secret**. Second, because Δm is random, decryption can fail — with probability driven below 2⁻¹²⁸ by parameter choice. Both of those get exploited."

---

### I-4 — The FO transform *(1:00)*

**On slide**
```
Encaps:  (K̄, r) = G(m ‖ H(pk));  c = Enc(pk, m, r)

Decaps:  1. m′ = Dec(ŝ, c)
         2. (K̄′, r′) = G(m′ ‖ H(pk))
         3. c′ = Enc(pk, m′, r′)        ← re-encryption
         4. c = c′ ?  K = K̄′  :  K = J(z ‖ c)
```
- **Derandomization** → honest ciphertexts are reproducible
- **Implicit rejection** → failure returns a pseudorandom key, not ⊥

**Visual:** the pseudocode, steps 3 and 4 highlighted.

**Say:** "The randomness r isn't sampled at encryption — it's derived from the message. So an honest ciphertext can be recomputed: decrypt, re-derive, re-encrypt, compare. And on failure the device doesn't say 'invalid' — it returns a pseudorandom key. From the outside you cannot tell success from failure. On paper, that closes the chosen-ciphertext door completely."

---

### I-5 — Where the secret becomes observable *(1:00)*

**On slide**
- **A.** Direct — `NTT(u′)` → **`ŝᵀ ∘ û`** → reduce → `INTT`
- **B.** Indirect via `m′` — subtract, compress, **`G(m′‖H(pk))`**, re-encrypt, compare
- **C.** Integrity — `e` in KeyGen, the comparison result, the RNG seed

**Visual:** the decapsulation block diagram, both regions marked.

**Say:** "Three classes. The arithmetic touches the secret directly — that's the obvious target. But look at region B. To *check* a ciphertext, the device must decrypt it, hash the result, and re-encrypt — and only then decide to reject. Implicit rejection hides the output. It does nothing about the computation. And a third class that confidentiality thinking misses: values that break the scheme if *corrupted*. Some of those are public. Public doesn't mean safe to corrupt."

---
---

# PAPER 1 — Generic SCA on CCA-secure Lattice KEMs (8 slides, 8:00)

*Ravi, Sinha Roy, Chattopadhyay, Bhasin — TCHES 2020(3)*

### 1-1 — What this paper is about *(0:45)*

**On slide**
- **Problem:** FO transform was believed to block chosen-ciphertext attacks
- **Idea:** FO must *hash the decrypted message* to check it — and that hash leaks over EM
- **Result:** full key recovery on 6 lattice KEMs; Kyber512 in **~11 min, 7.7k traces**
- **Why it matters:** algorithm-level, not implementation-level

**Say:** "This is the paper that reopened chosen-ciphertext attacks on CCA-secure lattice KEMs. The insight is structural, not about any particular implementation — which is why rewriting your NTT, going constant-time, or changing compilers does nothing against it."

---

### 1-2 — The leak is upstream of the rejection *(1:00)*

**On slide**
```
Decaps:
  2.  m′ = Decrypt(sk, c)
  9.  r′ = G(m′, pk)          ← ★ THE LEAK
 10.  c′ = Encrypt(pk, m′, r′)
 11.  if c′ = c: return K
 15.  else:      return K = H(z‖c)
```

**Visual:** the pseudocode, line 9 boxed in red, line 15 boxed in gray.

**Say:** "Line 9 happens before line 15. The FO transform has to hash the decrypted message in order to verify it — so by the time the device decides to reject, it has already processed the thing the rejection was supposed to hide. For Kyber, G is SHA3-512 over 64 bytes: one Keccak permutation, 24 rounds. Sample the late rounds and a one-bit input difference has become a fifty-percent state difference. The hash's own avalanche is the attacker's amplifier."

---

### 1-3 — Crafting the ciphertext *(1:15)*

**On slide**
- Set `u₀[0] = k_u`, `v[0] = k_v`, everything else zero
- → `m′[0] = Poly_to_Msg(k_v − k_u·s₀[0])`
- Constraint: **`2·k_u ≤ 832` ⟹ `k_u ≤ 416`**
- ⟹ only **two** messages ever reach G: `00…0` or `01 00…0`

**Visual:** the message polynomial with one non-zero coefficient highlighted.

**Say:** "One knob, one secret coefficient. The constraint is what makes it work: keep k_u small enough that every *other* message bit decodes to zero no matter what the secret is. Then the device only ever hashes one of two fixed 32-byte inputs. That's why a single pair of templates classifies all 2560 queries."

---

### 1-4 — The decision table *(1:15)*

**On slide**

| s | (211,416) | (211,2913) | (101,832) | (101,2497) | (416,1248) |
|---|---|---|---|---|---|
| −2 | **X** | O | **X** | O | **X** |
| −1 | O | O | **X** | O | **X** |
| 0 | O | O | O | O | **X** |
| 1 | O | O | O | **X** | O |
| 2 | O | **X** | O | **X** | O |

- 5 candidates (η=2) → 5 queries → unique signature per row

**Visual:** the table; annotate the cells with margin 1.

**Say:** "This is the aha slide. Five queries, five candidate values, every row a unique pattern. Two things the paper doesn't say. Those constants aren't arbitrary — every one is a point on Kyber's ciphertext compression grid, because you can only inject values that survive decompression. And three of these decisions hinge on a margin of **one** out of 3329 — which is fine only because a chosen ciphertext carries no LWE error term. The arithmetic is exact."

---

### 1-5 — The distinguisher, and profiling on the target *(1:00)*

**On slide**
- Welch t-test locates points of interest → reduced templates → classify O/X
- **One trace per query**
- Profiling needs **no clone device**: set `k_u = 0` ⟹ `u·s = 0` for *any* key

**Visual:** the TVLA plot with the ±4.5 threshold line.

**Say:** "TVLA is normally an evaluation tool — a pass/fail leakage certification. Here it's repurposed as feature selection for an attack. And note the profiling trick: set k_u to zero and the secret drops out of the equation entirely, so v alone dictates the decrypted message. The attacker builds templates on the victim's own device with no known key."

---

### 1-6 — Reaching all 512 coefficients *(0:45)*

**On slide**
- `R_q = Z_q[X]/(X²⁵⁶+1)` is **anti-cyclic**
- `u[p] = k_u·x^q` → sweep `q ∈ [0,n)`, `p ∈ [0,k)`
- ⚠ For `q ≥ 1` the table returns the **negated** coefficient
- Total: **5 × 256 × 2 = 2560 queries**

**Visual:** a rotation arrow over a coefficient array.

**Say:** "Multiplying by x^q rotates the secret, so the same trick reaches every coefficient. Watch the sign — for any rotation you read back the negated value. It's a classic off-by-a-sign bug if you implement this. Frodo, by the way, has no rotation property at all, which is why it needs eight separate template sets."

---

### 1-7 — Results *(1:00)*

**On slide**

| Scheme | Traces | Time | Success |
|---|---|---|---|
| **Kyber512** | 2560 queries → **7.7k** | **10.8 min** | 99 % |
| Round5 | 978 → 2.9k | 4.5 min | 99 % |
| LAC128 | 1024 → 3.0k | 25 min | 97 % |

STM32F407 Cortex-M4 @ 24 MHz · pqm4 · EM probe · 500 MSa/s

**Say:** "Eleven minutes for a full key. The gap between 2560 queries and 7.7k traces is three-times majority voting to absorb oracle errors. One honest caveat worth stating: the title says six schemes, but only three were actually measured. Saber rides on a structural-similarity argument, NewHope is simulation against a perfect oracle, and Frodo is an appendix sketch with no numbers."

---

### 1-8 — What must be protected *(1:00)*

**On slide**
- **Masking the NTT alone is useless** — the leak is downstream
- Must mask: `m′` → **Keccak** → re-encryption → comparison
- Masked A2B is **hard for Kyber**: `q = 3329` is prime
- The paper's only recommendation: mask the full decapsulation, and it concedes the cost

**Say:** "This paper diagnoses; it doesn't cure. The recommendation is to mask everything from the message onward, including a masked Keccak — and Kyber's prime modulus makes the arithmetic-to-Boolean conversion considerably harder than it would be for a power-of-two modulus. If you take one thing away: masking the polynomial multiply protects the wrong operation."

---
---

# PAPER 2 — Defeating Low-Cost Countermeasures (9 slides, 8:00)

*Ravi, Paiva, Jap, D'Anvers, Bhasin — TCHES 2024(2)*

### 2-1 — What this paper is about *(0:45)*

**On slide**
- **Problem:** masking full decapsulation is expensive → two cheap *detection* countermeasures proposed
- **What they do:** break both
- **Result:** Kyber768 key recovery in 325–7800 traces; plus a new inequality solver
- **Verdict:** detection gives **no standalone protection**

**Say:** "Two countermeasures that try to *detect* the attack rather than hide the data. Both fall. And the second one falls to something genuinely new — the first chosen-ciphertext side-channel attack that uses only valid ciphertexts."

---

### 2-2 — The two countermeasures *(0:45)*

**On slide**

| | Mechanism | Why it looked good |
|---|---|---|
| **Ciphertext sanity check** | reject low-variance ciphertexts | ~5 % overhead, rejects *before* decapsulation → no leakage at all |
| **Decapsulation failure check** | refresh key on **first** failure | free, protocol-level, caps attacker at **one trace** |

**Say:** "Both are genuinely attractive. Attack ciphertexts have one non-zero coefficient out of 768 — trivially detectable. And every known attack uses invalid ciphertexts, so refreshing the key on the first failure limits the attacker to a single trace. That second one is the stronger of the two."

---

### 2-3 — Attack A: mask the ciphertext with the public key *(1:00)*

**On slide**
Send `(u_atk + a*, v_atk + b*)` instead of `(u_atk, v_atk)`:
```
Δ = (v_atk + b*) − (u_atk + a*)·s
  = v_atk + A·s + e − u_atk·s − A·s
  = v_atk − u_atk·s + e
```
- Ciphertext now looks like a genuine LWE sample
- **Downstream attack completely unchanged**

**Say:** "Four lines of algebra. The A·s terms cancel and you're left with the original quantity plus a small error. The ciphertext passes the entropy test, and nothing downstream needs to change — any existing attack ports across unmodified. Patching this by rejecting ciphertexts *close to* the public key doesn't work either: use minus A, or two A, or a rotation."

---

### 2-4 — Why it only works 58 % of the time *(0:45)*

**On slide**

| z | s=−2 | s=−1 | s=0 | s=1 | s=2 |
|---|---|---|---|---|---|
| 624 | 1040 | **832** | 624 | 416 | 208 |
| 832 | 1248 | 1040 | **832** | 624 | 416 |
| 1040 | 1456 | 1248 | 1040 | **832** | 624 |

- Decode threshold = **833**
- Every column has one cell at **832** — margin **1**
- Success 57.8 % ⟹ expected restarts **0.73**

**Say:** "The paper reports 57.8 % without explaining it. Here's why: every column of the table contains exactly one cell whose value is 832, one unit below the decode threshold. For that candidate, the oracle's correctness is decided purely by the sign of the added error. It's a coin flip — which is also why the fix is simply to retry with a different mask."

---

### 2-5 — Attack B: the bind *(0:30)*

**On slide**
- Ciphertexts must be **valid** ⟹ re-encryption succeeds ⟹ no failure ever
- ⟹ the FO transform is off-limits
- ⟹ only the **decryption** procedure is exploitable
- But then `m` is not key-dependent… so where's the key?

**Say:** "If you can't use invalid ciphertexts, the whole FO transform closes to you. You're left with decryption only. And for a valid ciphertext the message isn't key-dependent at all. So where is the key?"

---

### 2-6 — The key is in the noise *(1:00)*

**On slide**
```
m′ = ⌈q/2⌋·m + Δm
Δm = ⟨r,e⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv
```
- **Linear in (s, e)**
- Attacker encrypted it themselves ⟹ knows `m, r, e₁, e₂, Δu, Δv`
- Everything is known **except s and e**

**Say:** "This is the pivot. The message doesn't carry the key, but the *noise* does — and it's linear in the secret. Because the attacker generated the ciphertext honestly, they know every term except the two they want. Exact knowledge of Δm for just six ciphertexts would give the key by linear algebra. They can't get exact values, only bounds — hence an inequality solver."

---

### 2-7 — Reading the sign off the Hamming weight *(1:15)*

**On slide**
```
Δm[i] = +5  →  0000 0000 0000 0101   HW = 2    ← low
Δm[i] = −5  →  1111 1111 1111 1011   HW = 14   ← high
```
- When `m[i] = 0`: **HW is a near-perfect sign detector**
- When `m[i] = 1`: `m′[i] ≈ 1664 ± Δm` → **no information**
- ⟹ only the ~128 zero bits per message are usable

**Visual:** the two bit patterns side by side, large.

**Say:** "The cleverest idea in the paper, and it isn't about Kyber at all — it's about how signed integers are stored. A small positive number has almost no bits set; a small negative number has almost all of them. So the Hamming weight tells you the sign of the noise. When the message bit is one, the value sits near 1664 and no sign boundary is crossed, so you learn nothing. That halves the usable data and drives everything downstream."

---

### 2-8 — Inequalities and the greedy solver *(1:00)*

**On slide**
- Sign → two-sided bounds via an empirical min/max-per-HW table
- Bounds must be **measured**: lazy reduction means HW=10 gives max **+253**, not −64
- Stack into `Hx + w ≥ 0`; solve greedily

| | Belief Propagation | **Greedy** |
|---|---|---|
| inequalities | baseline | **< half** |
| noise tolerance | dies past σ ≈ 0.5 | **σ ≈ 2.0** |
| runtime | > 10 min | **< 20 s** |

**Say:** "Instead of just a sign bit, use an empirical table of min and max per Hamming weight — two inequalities per observation. The bounds have to be measured rather than derived, because efficient Kyber implementations skip full modular reduction. Then the solver: start at zero, score every possible coordinate change by total distance from satisfying all inequalities, apply the best ones, repeat. No message passing, no probabilities — and it beats Belief Propagation on every axis."

---

### 2-9 — Results and what must be protected *(1:00)*

**On slide**

| Setting | HW acc. | Traces |
|---|---|---|
| Reference −O3 | 91 % | **5200 measured** (325 extrapolated) |
| Assembly-optimized | **32 %** | 7800 |
| Shuffled | assumed perfect | > 8 000 000 |

Protect: the `m′` store · **all shares** when masked · the *representation* of `m′`

**Say:** "Two caveats. The famous 325 figure is extrapolated from a measured 5200. And the masked evaluation replaced the library's assembly with C at -O0 — which is why masked accuracy beats unprotected-optimized. That's a compiler-flag artefact, not a masking result. The real lesson for hardware: one coefficient per store gives 91 % classifier accuracy, ten per store gives 32 %. A wide parallel datapath attenuates this for free."

---
---

# PAPER 3 — PQ-Hammer (10 slides, 8:00)

*Amer, Wang, Kippen, Dang, Genkin, Kwong, Nelson, Yerukhimovich — IEEE S&P 2025*

### 3-1 — What this paper is about *(0:45)*

**On slide**
- **No probe, no oscilloscope** — an unprivileged process on the same machine
- Rowhammer flips bits in the victim's DRAM
- **Three schemes, one primitive:** Kyber (data flip), BIKE (instruction flip), Dilithium (both)
- First **end-to-end long-term** key recovery on PQC

**Say:** "Everything else today assumes physical access to the board. This assumes you can run an unprivileged process on the same machine. Prior work on PQC Rowhammer got session keys or partial information — this gets the whole long-term key, on a desktop, with no supercomputer."

---

### 3-2 — Threat model *(0:45)*

**On slide**

| | Power/EM SCA | **Rowhammer** |
|---|---|---|
| physical access | required | **none** |
| privileges | — | **unprivileged user** |
| nature | passive observation | **active fault injection** |
| target property | confidentiality | **integrity** |

**Say:** "Note the last row especially. Everything else today is about information leaking out. This is about values being *changed*. That flips which countermeasures are relevant — masking does nothing here; redundancy and auditing do."

---

### 3-3 — Rowhammer in sixty seconds *(0:45)*

**On slide**
- Hammer an **aggressor** row → bits flip in adjacent **victim** rows
- DDR4's TRR tracks only a few rows → **many-sided / multi-bank** hammering defeats it
- **Performance degradation** widens the attack window
- **Memory massaging** places the victim's data on your vulnerable page

**Visual:** aggressor/victim row stripe diagram.

**Say:** "Four ingredients, all from prior work. The paper's contribution is stitching them together plus the PQC-specific cryptanalysis — and its own program committee said as much in the meta-review."

---

### 3-4 — The window in Kyber KeyGen *(1:00)*

**On slide**
```c
for(i..) poly_getnoise_eta1(&skpv.vec[i], ...);
for(i..) poly_getnoise_eta1(&e.vec[i], ...);
polyvec_ntt(&skpv);      ← ★ THE WINDOW
polyvec_ntt(&e);
```
- Before sampling: arrays are zeros, flips get overwritten
- After NTT: controlled Euclidean noise would need many coordinated flips
- ~35,000 cycles per `poly_ntt`; **~1400× shorter** than a DRAM refresh cycle

**Say:** "Exactly one usable window: after e is sampled, before it's transformed. And it's roughly fourteen hundred times shorter than a refresh cycle, which is why they need performance degradation. One counterintuitive detail — Kyber-1024 is the *easiest* target, because more NTT calls means a longer window."

---

### 3-5 — Two bits *(1:00)*

**On slide**

| coefficient | int16 | set bit 8 | set bit 6 |
|---|---|---|---|
| 0 | `0x0000` | **256** | **64** |
| +2 | `0x0002` | **258** | **66** |

- `e` coefficients live in **[−2, 2]**, stored in **16-bit** integers
- Too few flips → failure rate stays negligible
- Too many → keypair visibly broken
- **Two is the sweet spot**

**Visual:** the bit pattern, large.

**Say:** "Five possible values stored in sixteen bits. Fourteen of those bits do nothing but wait for someone to flip one. Set bit eight and a coefficient of two becomes two hundred fifty-eight — which pushes the decryption error over the threshold often enough to be useful, but not so often that the key looks broken. It's a tuning problem, not a maximization problem."

---

### 3-6 — Failure boosting *(0:45)*

**On slide**
- Honest failure rate after poisoning: ~**2⁻¹⁰**
- Resample `r` until the two coefficients multiplying the poisoned ones are **±η₁, same sign** (prob. 1/128)
- ⚠ **~67 % of observed failures come from some *other* coefficient**

**Say:** "The attacker doesn't wait for random failures — they filter the encryption randomness first, discarding a hundred twenty-seven of every hundred twenty-eight candidates. All local computation, so it's cheap. And the paper is honest about the noise: two thirds of the failures you observe are the wrong ones. That's why the recovery needs an estimator rather than arithmetic."

---

### 3-7 — Failures are hints *(0:45)*

**On slide**
- A failure means `⟨S, W⟩ > t′` — the known vector points *along* the secret
- Decompose `W` into projection onto `S` plus orthogonal part
- **`E[W] = (E[α′]/ℓ)·S`** — the mean of failing ciphertexts is **parallel to the secret**
- Estimator picks the failing coefficient at ~75 % accuracy; James–Stein for scale

**Say:** "Each decryption failure tells you your known vector points roughly the same direction as the secret. Average enough of them and the secret emerges up to a scaling factor. The complication is that you can't tell *which* coefficient failed — their estimator gets it right about three quarters of the time, which turned out to be enough."

---

### 3-8 — Results *(0:45)*

**On slide**

| | ASLR off | ASLR on |
|---|---|---|
| page massaged | 91/100 | 44/100 |
| **correct bits flipped** | **8/100** | **3/100** |

- 49 s profiling + 93 s hammering ≈ 142 s per attempt
- ~40k failing ciphertexts, ~2 h · key extraction 4.5 min/iteration
- Geekbench: −8 % single-core, −71 % multi-core — *"snappy and reactive"*

**Say:** "Eight percent success with ASLR off, three with it on — and notice ASLR mostly hurts the *massaging*, not the flip. Quote these raw outcomes rather than the derived failure rates, incidentally: the paper states its decryption failure rate three different ways and they don't reconcile."

---

### 3-9 — Dilithium in one slide *(0:45)*

**On slide**
```
z  = y + c·s₁
z′ = y + c′·s₁     ⟹    s₁ = (z − z′)/(c − c′)
```
- Target **ρ**: contaminates `c` but **not** `y`, and is a static buffer
- Filter usable pairs by **HW(Δz)** — low ⟹ same loop iteration ⟹ nonce cancels
- **Deterministic Dilithium should not be used where Rowhammer is a concern**

**Say:** "One fault, total break. The target selection is elegant — they need something that corrupts the challenge but not the nonce, and is statically stored. Rho satisfies all three. And note that rho is a *public* value. Public doesn't mean it doesn't need integrity protection."

---

### 3-10 — Countermeasures *(0:45)*

**On slide**
- **Bit-pack the error terms** ← makes the attack *arithmetically impossible*
- Key auditing — test ciphertexts, reject on any failure
- Redundant copy or hash of key material
- Don't rely on a single RNG call; check seed Hamming weight
- ECC memory and TRR: raise the bar, don't close it

**Say:** "Lead with bit-packing. It doesn't detect the attack — it removes the space the fault needs to live in. Store a value from minus two to two in three bits and two hundred fifty-six is simply unrepresentable. Everything else on this list is detection or mitigation; that one is prevention."

---
---

# PAPER 4 — Hardware-Friendly Shuffling for Kyber (9 slides, 8:00)

*Xu, Wang, Tian — Nanjing University — IEEE TCAS-II 72(3), 2025*

### 4-1 — What this paper is about *(0:45)*

**On slide**
- **Problem:** Kyber hardware leaks; masking costs ~1700× the area-time product
- **Idea:** shuffle **memory addresses**, not the datapath
- **Result:** CPA 4×10³ → no peak at 10⁵; TVLA 10⁴ → passes at 10⁷
- **Cost:** +260 slices, **+8.7 % ATP, zero extra cycles**

**Say:** "A genuinely cheap hardware countermeasure, and the cheapness comes from one architectural decision: they never touch the arithmetic. They only change which memory location each operation reads."

---

### 4-2 — What leaks *(1:00)*

**On slide**
```
m ← Compress_q( v − INTT( ŝᵀ ∘ NTT(u) ), 1 )
```

| # | Operation |
|---|---|
| 1 | **PWM** `ŝᵀ ∘ û` |
| 2 | modular reduction after PWM |
| 3 | subtraction `v − (sᵀu mod q)` |
| +4 | **INTT** — added by the authors |

**Say:** "Three leakage points from prior work, plus a fourth the authors add on their own initiative because INTT had been attacked in software. The important structural point: all four get protected by the *same* mechanism. That's why the marginal cost is one module rather than four bespoke defences."

---

### 4-3 — Why hiding, not masking *(0:45)*

**On slide**

| Approach | ATP |
|---|---|
| unprotected baseline | 96.7 |
| **this work** | **105.2** |
| Jati (hiding) | 789.3 |
| Kamucheka (**masking**) | **186 × 10³** |

**Say:** "That last row is why people reach for hiding. Note it's not a fair comparison — masking is a qualitatively stronger security claim with a security order attached, and hiding is heuristic. But three orders of magnitude is three orders of magnitude."

---

### 4-4 — Fisher–Yates doesn't fit hardware *(0:45)*

**On slide**
```
for i = n−1 down to 1:
    j ← uniform random in [0, i]
    swap a[i], a[j]
```
1. Needs a **continuous** random supply — fast TRNG is expensive
2. **Shrinking range** — uniform `[0,i]` needs a divider, or rejection sampling with **variable latency**

**Say:** "Both problems are about hardware realities rather than cryptography. Variable latency is the killer — a fixed-schedule datapath can't tolerate a shuffler that sometimes takes longer."

---

### 4-5 — Fix 1: batch the randomness *(0:45)*

**On slide**
- 32-bit LFSR runs **384 cycles** at init → fills a **64 × 6 FIFO** with *every* index up front
- TRNG supplies a **one-shot seed**, not a stream
- **FIFO is reused:** indices shift out, selected elements push in — one memory, two roles

**Visual:** the FIFO with arrows out (indices) and in (permutation elements).

**Say:** "Generate all the randomness before you start, so nothing is needed during the shuffle. And the neat part — the same FIFO that empties of random indices refills with the permutation being built. Occupancy stays constant at sixty-four."

---

### 4-6 — Fix 2: cheap range reduction *(0:45)*

**On slide**
```
stage 1:  idx_0 = idx  if idx ≤ 0x28  else  idx − 0x28
stage 2:  idx_1 = idx  if idx ≤ rest  else  idx & rest
```
- No divider, no rejection sampling, **fixed latency**
- Example: `rest = 2`, `idx = 5` → `5 & 2 = 0`

**Say:** "A subtract and a bitwise AND. That's the whole reduction. It's biased — I'll come back to that — but it's constant-time and costs almost nothing."

---

### 4-7 — The address-conflict problem *(1:15)*

**On slide**
- Pipelined datapath: read at *t* → write-back at *t*+1 or *t*+12
- Naive permutation → collisions in the first six and last six slots → **wrong result**
- Fix: split the permutation
```
per_a = {0x0b … 0x33}      41 elements
per_b = rest                23 elements
```
- `min(per_a) = 0x0b > 0x0a` and `max(per_a) = 0x33 < 0x34`
- ⟹ **every per_a element satisfies all 11 constraints at once**

**Say:** "This is the cleverest part. Rather than checking constraints at runtime, they design a permutation where one region satisfies every constraint by construction. Fill the twelve risky slots from that region and no conflict is possible. One caveat I'd flag: the algorithm backfills vacated slots from the *other* region, so in simulation about eighty-six percent of runs put at least one out-of-region value into those twelve slots. Either the shipped HDL has a guard the brief omits, or the guarantee is weaker than stated."

---

### 4-8 — Results *(1:00)*

**On slide**

| Test | Unprotected | Protected |
|---|---|---|
| CPA on PWM | key found at **4×10³** | **no peak at 10⁵** |
| TVLA | fails at **10⁴** | **passes at 10⁷** |

| Resource | Δ |
|---|---|
| Slices | **+260 (+12.0 %)** |
| ENS / ATP | **+8.7 %** |
| Cycles, f_max, DSP, BRAM | **0** |

**Say:** "Zero cycle overhead is real — the permutation generator runs in parallel and needs about four hundred fifty cycles against six to ten thousand for a decapsulation. One note on the headline number: ENS adds a constant for DSPs and BRAMs to both designs, and the countermeasure uses neither, so the percentage is diluted. On slices alone it's twelve percent. Both are defensible; just know which you're quoting."

---

### 4-9 — Verdict and applicability *(1:00)*

**On slide**

| ✓ | ✗ |
|---|---|
| +8.7 % ATP, 0 cycles | **32-bit LFSR caps entropy at 32 bits** |
| one mechanism, four leakage points | **biased shuffle**, never analysed |
| datapath untouched | only **CPA and TVLA** tested |
| composes with masking | 25× demonstrated, not the ~N²=4096× theory |

**For our IP:** yes — but replace the LFSR with a real DRBG, restructure to power-of-two ranges, and don't quote 8.7 % as an ASIC number

**Say:** "The core idea transfers directly to any memory-based PQC accelerator — it's a mux on the address bus plus a delay-matched write path. Three changes I'd make. The LFSR has to go: three hundred eighty-four output bits from thirty-two state bits, linearly related, means profile once and predict forever. Restructure to power-of-two block sizes and the AND becomes exactly a modulo — an unbiased shuffle for zero extra hardware. And the eight-point-seven percent is an FPGA slice ratio; a sixty-four-to-one mux tree doesn't map the same way in standard cells."

---
---

# PAPER 6 — RNR as an SPA Countermeasure (8 slides, 8:00)

*Nagpal, Hadžić, Primas, Mangard — SAC 2025*

### 6-1 — What this paper is about *(0:45)*

**On slide**
- **Problem:** ML-KEM computes in `Z_3329` but stores coefficients in **16-bit** words
- That mismatch leaks — and signed representation also hands over the sign
- **Idea:** compute in `Z_ηq` so each value has **η machine encodings**
- **Result:** leakage 3.56 → 0.9 bits/coefficient; SASCA rendered ineffective; **INTT costs 0 %**

**Say:** "This paper makes the case that your choice of numeric representation is itself a security parameter — an axis most implementers never consider."

---

### 6-2 — Threat model: single-trace SASCA *(0:45)*

**On slide**
- **Soft-Analytical Side-Channel Attack**: encode the algorithm as a **factor graph**, run **belief propagation**
- Seven NTT butterfly layers = seven views of every coefficient
- Key recovered from **one trace**
- Works against *masked* implementations too — shares fall divide-and-conquer

**Say:** "Different from a multi-trace differential attack. You profile every intermediate, connect them in a graph that encodes the algorithm, and let belief propagation propagate constraints. The NTT is the ideal target because every coefficient passes through seven layers."

---

### 6-3 — Why the representation leaks *(1:00)*

**On slide**
```
+5  →  0000 0000 0000 0101   HW = 2
−5  →  1111 1111 1111 1011   HW = 14
```
- `q = 3329 ≈ 2^11.7` stored in a **16-bit** word
- Hamming weight is a **deterministic function** of the value
- Signed representation additionally reveals the **sign**

**Say:** "Two separate problems. The Hamming weight is fully determined by the value — no uncertainty at all. And two's complement makes small negatives look completely different from small positives, which hands over one extra bit for free."

---

### 6-4 — Measuring it *(1:00)*

**On slide**

| Representation | H[W] | H[W\|X] | **I[X;W(X)]** |
|---|---|---|---|
| **signed** `Z_q` | 3.561 | 0 | **3.561** |
| unsigned `Z_q` | 2.756 | 0 | **2.756** |
| **RNR± (η=9)** | 3.149 | 2.243 | **0.906** |
| RNR⁺ (η=5) | 2.958 | 1.739 | **1.219** |

- Signed → unsigned alone saves **~206 bits per polynomial**, free

**Say:** "Four numbers, and the free one is worth noticing. Just switching from signed to unsigned storage costs the attacker two hundred six bits across a polynomial, at zero cost. The fast signed arithmetic everyone uses for performance is measurably less secure."

---

### 6-5 — Why redundancy is the only fix *(1:15)*

**On slide**
```
I[X ; W(X)]  =  H[W(X)]  −  H[W(X) | X]
```
- Without redundancy **`H[W|X] = 0`** — the weight is fully determined by the value
- So *all* of `H[W]` leaks
- You cannot fix this by reshuffling bits
- You must **manufacture conditional entropy** → multiple encodings per value

**Visual:** the identity, with `H[W|X] = 0` circled.

**Say:** "This is the conceptual core. Mutual information is the entropy of the weight minus the conditional entropy given the value. If each value has exactly one encoding, that second term is zero and everything leaks. No clever bit layout fixes that. The only lever is giving each value several possible encodings — and that means changing the ring you compute in."

---

### 6-6 — RNR *(1:00)*

**On slide**
1. Find largest `η` with no overflow: `η± < 2³³/(2¹⁸q + q²) = 9.72` → **η± = 9**
2. Encode `X′ = X + K·q`, `K` random
3. Compute entirely over `Z_ηq`
4. Decode by reducing mod `q`

- Implementation delta: **rejection-sampling bounds + reduction constants**
- No extra operations, no fresh randomness during computation

**Say:** "Four steps, and the last line is the selling point. You don't add gadgets or randomness in the middle of the computation. You change two constants and the sampling bounds."

---

### 6-7 — Results *(1:00)*

**On slide**

| Implementation | SASCA outcome |
|---|---|
| signed | 100 % success until σ ≥ 3.0 |
| unsigned | degrades from σ ≥ 1.6 |
| **RNR±** | **ineffective at every noise level** |
| RNR⁺ | ~25 % at σ ≤ 0.1 |

- SNR down **two orders of magnitude**; PI ≈ 0 in shallow layers
- **Signed INTT 42.61 kcycles → RNR± INTT 42.61 kcycles — 0 % overhead**

**Say:** "Forty-two point six one against forty-two point six one. That's not rounding — the signed inverse NTT needs no additional reductions, so only the constants change. And since the inverse NTT is what touches the secret key in decryption, that's the number that matters most. The forward NTT does cost sixty to ninety percent."

---

### 6-8 — Verdict and applicability *(1:15)*

**On slide**

| ✓ | ✗ |
|---|---|
| INTT 0 % overhead | **SPA only — no DPA/CPA protection** |
| no fresh randomness mid-computation | RNR⁺ **not secure** (25 % at low noise) |
| composes on top of masking | reference C only, not optimized assembly |
| constants-only change | protects **only** the NTT/INTT |

**For our IP: no, not effectively**
- Fixed datapath width — no spare headroom to spend
- One unrefreshed offset, **3.17 bits** vs ~11.7 for a real mask → averaged away by DPA
- Coverage too narrow for the area cost

**Say:** "Three blocking reasons for us. Our datapath width is fixed and sized to the minimum — the spare headroom RNR exploits is a software accident of the machine ABI, not something hardware has lying around. Second, it does draw randomness, but only about three bits per coefficient, and because the NTT is linear that same offset propagates through all seven layers without refresh. That's a low-entropy masking scheme that never re-randomizes, which is exactly what falls to averaging. And third, it covers one block while we'd still need masking on that same block. What *is* worth taking is the measurement — computing that mutual information for your own representation and width is a few lines of code and needs no silicon."

---
---

# PAPER 5 — placeholder

*Chen, Ma, Jing — IEEE TCAD 42(1), 2023 — PDF not yet obtained*

Reserve 8 minutes. From the abstract and citing work, expect roughly:
1. What this paper is about
2. Protection point: NTT, converted to a single-level loop
3. Key idea: unified shuffling controller
4. Coefficient index randomization
5. NTT network randomization
6. Results: 9 % resource overhead, FPGA
7. Advantages / disadvantages
8. Applicability verdict

⚠ One thing to verify if you get the paper: a later ACM TECS paper describes their index scheme as using a **one-time rotation via modular addition of N/2**, not a full Fisher–Yates permutation. If accurate, that's a much smaller permutation space than full shuffling and deserves scrutiny.

---
---

# Running order and timing

| Segment | Slides | Time | Cumulative |
|---|---|---|---|
| Intro | 5 | 5:00 | 5:00 |
| Paper 1 | 8 | 8:00 | 13:00 |
| Paper 2 | 9 | 8:00 | 21:00 |
| Paper 3 | 10 | 8:00 | 29:00 |
| Paper 4 | 9 | 8:00 | 37:00 |
| Paper 5 | 8 | 8:00 | 45:00 |
| Paper 6 | 8 | 8:00 | 53:00 |
| Q&A | — | 7:00 | **60:00** |

**If you run long,** cut in this order: Paper 1 slide 6 (rotation sweep), Paper 3 slide 3 (Rowhammer background), Paper 2 slide 5 (the bind — fold into slide 6), Intro slide 3 (compress the PKE algorithms into one block).
