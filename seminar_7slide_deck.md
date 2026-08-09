# Kyber SCA Seminar — 7-Slide Deck

1 intro + 6 papers · ~53 min · sparse slides, dense narration

**Convention:** `**bold**` = a keyword to highlight visually on the slide (colored pill, box, or accent — not just bold). Everything else is plain keyword text. Nothing on a slide is a sentence.

**Layout for every paper slide:**

```
┌────────────────────────────────────────────────────────┐
│  Paper title                                           │
│  Authors · Venue                                       │
├──────────────────────────┬─────────────────────────────┤
│  cluster 1               │  cluster 2                  │
├──────────────────────────┼─────────────────────────────┤
│  cluster 3               │  cluster 4                  │
├──────────────────────────┼─────────────────────────────┤
│  cluster 5               │  cluster 6                  │
├──────────────────────────┴─────────────────────────────┤
│  RESULT strip — one number, highlighted                │
└────────────────────────────────────────────────────────┘
```

Six clusters, 2–4 keyword lines each, small gray label above. One result strip at the bottom. That's the whole slide.

---
---

# SLIDE 1 — Kyber / ML-KEM

**Header:** Kyber / ML-KEM (FIPS 203) — the standardized KEM

| Cluster | Keywords |
|---|---|
| **Foundation** | Module-LWE · `R_q = Z_q[X]/(X²⁵⁶+1)` · q = 3329 · k = 2/3/4 |
| **Two layers** | IND-CPA PKE ⊂ IND-CCA KEM · **FO transform** |
| **Encrypt** | `v = t̂ᵀ∘r̂ + e₂ + ⌈q/2⌋·m` · compress |
| **Decrypt** | `m′ = v′ − ŝᵀ∘û` → **m′ = ⌈q/2⌋m + Δm** |
| **Decaps (FO)** | decrypt → hash → re-encrypt → compare · **implicit rejection** |
| **Leak regions** | A. `ŝᵀ∘û` direct · B. everything after m′ · C. integrity |

**Result strip:** **Δm = ⟨e,r⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv — linear in the secret**

### Narration (5:00)

*Foundation, 45s.* "One prime, one ring, three parameter sets — only the module rank changes. A warning for the older literature: Round-2 Kyber512 used η₁ = 2 and d_v = 3, which changes how many candidate values a secret coefficient has and therefore what attacks cost. Papers from 2020 assume the old numbers."

*Two layers, 45s.* "Hold both layers in your head for the next hour. The inner PKE falls trivially to chosen-ciphertext attacks — that was never its job. The FO transform is what buys CCA security. Almost everything physically interesting about Kyber happens at the boundary between them."

*Encrypt / decrypt, 1:15.* "Message bits get scaled to half the modulus and buried in noise. Decryption subtracts the mask and rounds each coefficient to zero or q/2. Now look at the result strip. Two consequences. First, the noise term is **linear in the secret** — the message polynomial isn't just noisy, it's noisy in a way that depends on the key. Second, because that noise is random, decryption can fail. Parameters push the failure rate below 2⁻¹²⁸, but it isn't zero, and failures leak. Both of those get exploited later today."

*Decaps, 1:15.* "The randomness in encapsulation isn't sampled — it's derived from the message by a hash. So an honest ciphertext is reproducible: decrypt, re-derive, re-encrypt, compare. And on mismatch the device doesn't return an error — it returns a pseudorandom key derived from a secret stored in the private key. From outside you cannot distinguish accept from reject. On paper that closes the chosen-ciphertext door completely."

*Leak regions, 1:00.* "Three classes. The arithmetic touches the secret directly — obvious target. But region B is the interesting one. To *check* a ciphertext, the device must decrypt it, hash the result, and re-encrypt — and only then decide to reject. Implicit rejection hides the output. It does nothing about the computation. And a third class most people skip: values that break the scheme if *corrupted* rather than observed. Some of those are public. Public doesn't mean safe to corrupt."

---
---

# SLIDE 2 — Generic SCA on CCA-secure Lattice KEMs

**Header:** Ravi, Sinha Roy, Chattopadhyay, Bhasin · TCHES 2020(3)

| Cluster | Keywords |
|---|---|
| **Existing method** | key-mismatch → CPA only · D'Anvers → timing, ECC |
| **Attack model** | static key · EM · blind to output · **no clone device** |
| **Attack point** | **G(m′ ‖ pk)** — FO hash · ECC decode |
| **Procedure** | `u₀[0]=k_u`, `v[0]=k_v` · **k_u ≤ 416** · 5 queries/coeff · anti-cyclic sweep |
| **Target device** | STM32F407 · pqm4 · EM · 500 MSa/s |
| **Protect / CM** | m′ → Keccak → re-encrypt → compare · mask all · prime q → A2B hard |

**Result strip:** 2560 queries → 7.7k traces → **10.8 min** · Kyber512 · 3 of 6 measured

### Narration (8:00)

*Framing, 1:00.* "This is the paper that reopened chosen-ciphertext attacks on CCA-secure lattice KEMs. Prior key-mismatch attacks only worked on CPA-secure schemes, because the FO transform removes the oracle they need. D'Anvers found a timing leak in error-correcting decoders, but constant-time code kills that. This paper's insight is structural: the FO transform has to hash the decrypted message in order to verify it — and that hash happens *before* the rejection. By the time the device decides to discard the result, it has already processed the thing the rejection was supposed to hide."

*Attack point, 1:00.* "For Kyber, G is SHA3-512 over 64 bytes — one Keccak permutation, twenty-four rounds. They sample the *late* rounds, where a one-bit input difference has become roughly a fifty-percent state difference. The hash's own avalanche is the attacker's amplifier. The cryptographic strength of the primitive is exactly what makes the side channel loud."

*Procedure, 2:00.* "One knob, one secret coefficient. Set a single coefficient of u and a single coefficient of v, everything else zero — then the first message bit becomes a function of one secret coefficient alone. The constraint is what makes it work: keep k_u small enough that every *other* message bit decodes to zero no matter what the secret is. That gives k_u at most 416, and it means the device only ever hashes one of two fixed thirty-two-byte inputs. A single pair of templates classifies all twenty-five hundred queries. Five well-chosen query pairs pin down a coefficient, and multiplying by a power of x rotates the secret so the same trick reaches all five hundred twelve. Watch the sign on the rotated ones — you read back the negated value."

*Distinguisher, 1:00.* "Welch's t-test locates the leaking sample points, then a reduced template classifies each trace. One trace per query. And note the profiling: set k_u to zero and the secret drops out of the equation entirely, so v alone dictates the message. The attacker builds templates on the victim's own device, with no known key and no clone. TVLA is normally an evaluation tool — pass or fail leakage certification. Here it's repurposed as feature selection for an attack."

*Results, 1:30.* "Eleven minutes for a full key on a Cortex-M4. The gap between twenty-five hundred queries and seven thousand seven hundred traces is three-times majority voting to absorb oracle errors. One honest caveat: the title says six schemes, but only three were actually measured. Saber rides on a structural-similarity argument, NewHope is simulation against a perfect oracle, and Frodo is an appendix sketch with no numbers at all."

*Countermeasures, 1:30.* "This paper diagnoses; it doesn't cure. The recommendation is masking everything from the decrypted message onward — through Keccak, through re-encryption, through the comparator. And Kyber's prime modulus makes the arithmetic-to-Boolean conversion considerably harder than a power-of-two modulus would. If you take one thing away: masking the polynomial multiply protects the wrong operation. The secret leaves through the hash that proves the ciphertext was honest."

---
---

# SLIDE 3 — Defeating Low-Cost Countermeasures

**Header:** Ravi, Paiva, Jap, D'Anvers, Bhasin · TCHES 2024(2)

| Cluster | Keywords |
|---|---|
| **Target countermeasures** | ciphertext sanity check · decapsulation failure check |
| **Attack A** | **+ public key mask** → A·s cancels · 57.8 % · retry |
| **Attack B** | **valid ciphertexts only** · decryption stage only |
| **Key insight** | Δm linear in (s,e) · **+5 → HW 2 · −5 → HW 14** |
| **Solver** | greedy > belief propagation · ½ inequalities · σ = 2.0 · 20 s |
| **Protect** | m′ store · all shares · **the representation** |

**Result strip:** 325 extrapolated / **5200 measured** / 7800 assembly · Kyber768 · STM32F407

### Narration (8:00)

*Framing, 1:00.* "Masking a full decapsulation is expensive, so people proposed two cheap alternatives that try to *detect* the attack instead of hiding the data. One rejects low-entropy ciphertexts — attack ciphertexts have one non-zero coefficient out of seven hundred sixty-eight, trivially detectable, and it rejects *before* decapsulation so the attacker sees no leakage at all. The other refreshes the key on the first decapsulation failure, which caps the attacker at a single trace. Both are attractive. This paper breaks both."

*Attack A, 1:30.* "Four lines of algebra. Add a row of the public key to the attack ciphertext. The A·s terms cancel and you're left with the original quantity plus a small error. The ciphertext now looks like a genuine LWE sample and passes the entropy test — and nothing downstream changes, so any existing attack ports across unmodified. It works fifty-eight percent of the time, and here's why the paper doesn't explain: every column of the decision table contains one cell whose value sits exactly one unit below the decode threshold. For that candidate the oracle is decided purely by the sign of the added error. A coin flip — which is also why the fix is just to retry with a different mask. Patching by rejecting ciphertexts *close to* the public key doesn't help either; use minus A, or two A, or a rotation."

*Attack B setup, 1:30.* "The failure check is the harder target, and it needs something genuinely new: the first chosen-ciphertext side-channel attack that uses only *valid* ciphertexts. If the ciphertext is valid, re-encryption succeeds, no failure ever occurs, and the countermeasure never fires. But that closes the entire FO transform to you — you're left with decryption alone. And for a valid ciphertext the message isn't key-dependent. So where is the key? It's in the noise. The message polynomial equals the scaled message plus a noise term, and that noise term is linear in the secret. Because the attacker generated the ciphertext honestly, they know every term except the two they want."

*The HW trick, 1:30.* "Now the cleverest idea in the paper, and it isn't about Kyber at all — it's about how signed integers sit in memory. Plus five has two bits set. Minus five has fourteen. So when the message bit is zero, the Hamming weight of the stored value is a near-perfect *sign detector* for the noise. When the message bit is one, the value sits up near sixteen sixty-four, no sign boundary is crossed, and you learn nothing. That means only about half the coefficients per message are usable, and it drives everything downstream."

*Solver, 1:30.* "Instead of a bare sign bit, they build an empirical table of minimum and maximum per Hamming weight — two inequalities per observation. The bounds have to be *measured* rather than derived, because efficient Kyber implementations skip full modular reduction, so the ranges aren't what theory predicts. Then a greedy solver: start at zero, score every possible coordinate change by total distance from satisfying all the inequalities, apply the best ones, decay, repeat. No message passing, no probability distributions. It beats belief propagation on every axis — half the inequalities, four times the noise tolerance, and twenty seconds against ten minutes."

*Results and caveats, 1:00.* "Two caveats worth stating. The famous three-hundred-twenty-five figure is extrapolated from a measured fifty-two hundred. And the masked evaluation replaced the library's assembly with C compiled at dash-O-zero — which is why masked accuracy comes out *better* than unprotected optimized. That's a compiler-flag artefact, not a masking result. The lesson for hardware is in the middle row: one coefficient per store gives ninety-one percent classifier accuracy, ten per store gives thirty-two. A wide parallel datapath attenuates this for free. And the headline verdict — detection assumes the attacker needs invalid ciphertexts. Once he doesn't, there's nothing to detect."

---
---

# SLIDE 4 — PQ-Hammer

**Header:** Amer, Wang, Kippen, Dang, Genkin, Kwong, Nelson, Yerukhimovich · IEEE S&P 2025

| Cluster | Keywords |
|---|---|
| **Existing method** | FrodoFlips → session key · SignatureCorrection → partial |
| **Attack model** | **no physical access** · unprivileged co-located · active fault |
| **Attack point** | e in KeyGen window · ρ in Dilithium · instruction flips |
| **The enabler** | [−2,2] stored in int16 → **14 free bits** · bit 8 → 256 · bit 6 → 64 |
| **Procedure** | profile → massage → degrade → hammer → boost → average |
| **Countermeasure** | **bit-packing** · key auditing · redundant copy · seed checks |

**Result strip:** **8/100** ASLR off · 3/100 ASLR on · 142 s/attempt · i7-8700K · Samsung DDR4

### Narration (8:00)

*Framing, 1:00.* "Everything else today assumes physical access to the board. This assumes only that you can run an unprivileged process on the same machine. Rowhammer flips bits in a victim's DRAM by reading adjacent rows rapidly. Prior work on post-quantum Rowhammer got session keys — FrodoFlips needed eight exact bit flips, managed seven, and required six hundred sixty-five thousand failing ciphertexts and a supercomputer. Or it got partial key information, as in SignatureCorrection. This paper gets the whole long-term key on three schemes, on a desktop."

*Threat model, 1:00.* "Note what changes. This is *active fault injection*, not passive observation. The property under attack is integrity, not confidentiality. Which means masking does nothing here — the relevant defences are redundancy, auditing, and memory layout. Four ingredients, all borrowed from prior Rowhammer work: many-sided hammering to defeat DDR4's row-refresh mitigation, performance degradation to widen the window, contiguous-memory detection, and memory massaging to place the victim's data on your vulnerable page. The paper's own program committee noted that the individual techniques are well known; the contribution is stitching them together plus the cryptanalysis."

*The window, 1:15.* "In Kyber's key generation there's exactly one usable window: after the error vector is sampled, before it's transformed to the NTT domain. Before sampling, the array is contiguous zeros and flips get overwritten. After the transform, producing a *controlled* amount of noise in the original domain would need many coordinated flips. And that window is roughly fourteen hundred times shorter than a DRAM refresh cycle, which is why they need performance degradation. One counterintuitive detail: Kyber-1024 is the *easiest* target, because more NTT calls means a longer window."

*The enabler, 1:30.* "Here's the whole attack in one observation. The error coefficients live between minus two and plus two — five possible values. They're stored in sixteen-bit integers. Fourteen of those bits do nothing but wait for someone to flip one. Set bit eight and a coefficient of two becomes two hundred fifty-eight. That inflates the dot product enough to push decryption over the failure threshold often enough to be useful — but not so often that the keypair looks broken. Too few flips and the failure rate stays negligible; too many and it's obvious. Two bits is the sweet spot. It's a tuning problem, not a maximization problem."

*Recovery, 1:15.* "Then they generate failing ciphertexts, filtering the encryption randomness first so the coefficients multiplying the poisoned ones are at maximum magnitude with matching sign — discarding a hundred twenty-seven of every hundred twenty-eight candidates, all local computation. Each decryption failure is a *hint* that the attacker's known vector points roughly along the secret. Average enough of them and the secret emerges up to a scaling factor. The paper is honest that about two thirds of the observed failures come from the wrong coefficient, which is why they need an estimator rather than arithmetic — it picks the right one about three quarters of the time."

*Dilithium, 45s.* "Briefly, because it's short and devastating. Two signatures on the same message with the same nonce but different challenges give you the secret key by simple division. They target rho — it corrupts the challenge but not the nonce, and it's a static buffer. One successful fault, total break. And note rho is a *public* value. Public doesn't mean it doesn't need integrity protection."

*Results and countermeasures, 1:15.* "Eight percent success with ASLR off, three percent with it on — and ASLR mostly hurts the *massaging*, not the flip. Incidentally, quote these raw outcomes rather than the paper's derived failure rates; it states them three different ways and they don't reconcile. For countermeasures, lead with bit-packing. It doesn't *detect* the attack — it removes the space the fault needs to live in. Store a value from minus two to two in three bits and two hundred fifty-six is simply unrepresentable. Everything else on the list is detection or mitigation. That one is prevention."

---
---

# SLIDE 5 — Hardware-Friendly Shuffling for Kyber

**Header:** Xu, Wang, Tian (Nanjing University) · IEEE TCAS-II 72(3), 2025

| Cluster | Keywords |
|---|---|
| **Protection point** | PWM · mod reduce · subtract · INTT — **all via addresses** |
| **Key idea** | **shuffle addresses, not arithmetic** |
| **Fisher–Yates problems** | continuous randomness · shrinking range · variable latency |
| **Fix 1 + 2** | LFSR 384 cycles → FIFO, reused · `idx − 0x28` · **`idx & rest`** |
| **Fix 3** | per_a / per_b · min > 0x0a, max < 0x34 · safe by construction |
| **Verdict / IP** | ✗ 32-bit LFSR · ✗ biased · ✗ CPA+TVLA only · **IP: yes, swap the LFSR** |

**Result strip:** CPA 4×10³ → none at 10⁵ · TVLA 10⁴ → 10⁷ · **+8.7 % ATP · 0 cycles**

### Narration (8:00)

*Framing, 1:00.* "Kyber hardware leaks — prior work identified three points in the decryption datapath and broke them with correlation power analysis. Both defence families have a cost problem. Hardware masking costs roughly seventeen hundred times the area-time product. Existing hiding schemes burn either cycles or area. This paper takes an open-source unprotected Kyber FPGA design and adds shuffling for eight point seven percent and zero extra clock cycles."

*Key idea, 1:00.* "And the cheapness comes from one architectural decision: they never touch the arithmetic. They permute the memory *addresses* the datapath visits. The datapath computes exactly what it computed before — it just doesn't know which coefficient it's holding. That's why one mechanism covers all four leakage points, including an inverse-NTT protection the authors add on their own initiative. The marginal cost is one module rather than four bespoke defences."

*Fisher–Yates, 1:00.* "Fisher–Yates gives you a uniform permutation, but it doesn't fit hardware. It needs a fresh random number per iteration — a true random generator that fast is expensive and is itself an attack surface. And the index range shrinks every iteration, so uniform reduction needs either a divider or rejection sampling. Rejection sampling has *variable latency*, which is poison for a fixed-schedule datapath. Both problems are about hardware realities rather than cryptography."

*Fixes 1 and 2, 1:30.* "First fix: generate all the randomness before you start. A thirty-two-bit LFSR runs three hundred eighty-four cycles at initialization and fills a FIFO with every index the permutation will need. The true random generator supplies one seed, not a stream. And the neat part — the same FIFO that empties of random indices refills with the permutation being built. One memory, two roles, occupancy constant. Second fix: replace the modulo with a subtract in stage one and a bitwise AND in stage two. That's the whole reduction. Constant latency, almost no area."

*Fix 3, 1:30.* "The third problem is the subtle one. The datapath is pipelined — a read at cycle t writes back at t plus one or t plus twelve. Apply a naive permutation and certain adjacent addresses collide, and the computation is simply *wrong*. Their fix is elegant: rather than checking constraints at runtime, design a permutation with two regions where one region satisfies every constraint by construction. The minimum of that region sits above the largest lower bound and its maximum sits below the smallest upper bound. Fill the twelve risky slots from that region and no conflict is possible. One caveat I'd flag — the algorithm backfills vacated slots from the *other* region, and in simulation about eighty-six percent of runs put at least one out-of-region value into those twelve slots. Either the shipped HDL has a guard the brief omits, or the guarantee is weaker than stated."

*Results, 1:00.* "Correlation analysis that found the key at four thousand traces finds no peak at a hundred thousand. TVLA that failed at ten thousand passes at ten million. Zero cycle overhead is real — the permutation generator runs in parallel and needs about four hundred fifty cycles against six to ten thousand for a decapsulation. One note on the headline: the area metric adds a constant for DSPs and block RAMs to both designs, and the countermeasure uses neither, so the percentage is diluted. On slices alone it's twelve percent. Both defensible; just know which you're quoting."

*Verdict, 1:00.* "The weaknesses are real. The thirty-two-bit LFSR caps the permutation entropy at thirty-two bits no matter how large sixty-four factorial is — profile once and predict forever. The reduction is biased and never analysed. And only CPA and TVLA were tested, not the attacks that actually matter against hiding. But for our purposes the core idea transfers directly to any memory-based accelerator: a mux on the address bus plus a delay-matched write path. Three changes I'd make. Replace the LFSR with a real deterministic random bit generator. Restructure to power-of-two block sizes — the AND then becomes exactly a modulo, giving an unbiased shuffle for zero extra hardware. And don't quote eight point seven percent as an ASIC number; that's an FPGA slice ratio, and a sixty-four-to-one mux tree doesn't map the same way in standard cells."

---
---

# SLIDE 6 — Low-Cost Shuffling for NTT-based PQC

**Header:** Chen, Ma, Jing (Chinese Academy of Sciences) · IEEE TCAD 42(1), 2023

⚠ *PDF not yet obtained — fill in once you have it. Structure below from the abstract and citing work.*

| Cluster | Keywords |
|---|---|
| **Protection point** | NTT · nested loops → **single-level loop** |
| **Key idea** | unified shuffling controller |
| **Method 1** | coefficient index randomization |
| **Method 2** | NTT network randomization |
| **Threat covered** | power analysis · template attacks |
| **Verdict / IP** | **9 % resource overhead** · negligible performance impact |

**Result strip:** FPGA · ~9 % overhead · ⚠ **verify: rotation by N/2, not full permutation?**

### What to check when you get it

A later ACM TECS paper describes their index scheme as using a **one-time rotation of the NTT index through modular addition of N/2**, rather than a full Fisher–Yates permutation. If that's accurate it's a much smaller permutation space than full shuffling — a random *starting offset* rather than a random *ordering* — and the security claim deserves scrutiny. That would be the single most important thing to establish, and it's a good question to raise even if you can't resolve it.

---
---

# SLIDE 7 — Redundant Number Representation

**Header:** Nagpal, Hadžić, Primas, Mangard · SAC 2025

| Cluster | Keywords |
|---|---|
| **Protection point** | NTT / INTT — **representation only** |
| **Threat** | single-trace SASCA · factor graph · belief propagation |
| **The problem** | q ≈ 2^11.7 in a **16-bit word** · +5 → HW 2 · −5 → HW 14 |
| **Key idea** | **I = H[W] − H[W\|X]** , and **H[W\|X] = 0** |
| **Method** | compute in **Z_ηq** · X′ = X + K·q · η± = 9 |
| **Verdict / IP** | ✗ SPA only · ✗ 3.17 bits, unrefreshed · **IP: no** |

**Result strip:** MI 3.561 → **0.906** · SASCA ineffective at all σ · **INTT 42.61 → 42.61 kcycles**

### Narration (8:00)

*Framing, 1:00.* "This paper makes a claim most implementers never consider: your choice of *numeric representation* is itself a security parameter. ML-KEM computes modulo three thousand three hundred twenty-nine — about two to the eleven point seven — but stores every coefficient in a sixteen-bit machine word. That mismatch leaks, and the paper quantifies exactly how much."

*Threat model, 1:00.* "Different from everything else today. A soft-analytical side-channel attack encodes the whole algorithm as a factor graph — variables as nodes, operations as factors — attaches a leakage distribution to every intermediate you can profile, and runs belief propagation until the marginals converge. The NTT is the ideal target because every coefficient passes through seven butterfly layers, giving you seven views of it. Key recovery from a *single trace*. And it works against masked implementations too, because shares can be attacked divide-and-conquer."

*The problem, 1:15.* "Two separate leaks. First, the Hamming weight is a *deterministic function* of the stored value — there's no uncertainty at all. Second, two's complement makes small negatives look completely different from small positives: plus five has two bits set, minus five has fourteen. So the signed representation hands over the sign for free. Measured: signed storage leaks three point five six bits per coefficient, unsigned two point seven six. That gap alone is about two hundred six bits across a polynomial — free to eliminate, and the fast signed arithmetic everyone uses for performance is measurably less secure."

*Key idea, 1:45.* "Here's the conceptual core, and it's worth going slowly. Mutual information is the entropy of the Hamming weight minus the conditional entropy of the weight *given* the value. If each value has exactly one machine encoding, that second term is zero — the weight is fully determined — and therefore *everything* leaks. No clever bit layout fixes that. You cannot reduce the first term much either. The only lever is manufacturing conditional entropy, and that means giving each value several possible encodings. Which means changing the ring you compute in."

*Method, 1:15.* "So: compute in Z sub eta-q instead of Z sub q. Find the largest eta that doesn't overflow the butterfly — for Kyber that's nine. Encode each input as the value plus a random multiple of q. Run the entire algorithm in the enlarged ring. Decode at the end by reducing mod q. The implementation delta is remarkably small: the rejection-sampling bounds and the reduction constants. No extra operations, no fresh randomness during the computation. Leakage drops from three point five six bits to nought point nine."

*Results, 1:00.* "Against simulated and real SASCA, the signed implementation succeeds until noise sigma of three; the signed RNR variant is ineffective at *every* noise level. Signal-to-noise drops two orders of magnitude. And the headline: forty-two point six one kilocycles against forty-two point six one. That's not rounding — the signed inverse NTT needs no additional reductions, so only the constants change. Since the inverse NTT is what touches the secret key in decryption, that's the number that matters most. The forward NTT does cost sixty to ninety percent."

*Verdict, 45s.* "For our IP, three blocking reasons. Our datapath width is fixed and sized to the minimum — the spare headroom this exploits is a software accident of the machine ABI, not something hardware has lying around. Second, it does draw randomness, but only about three bits per coefficient against roughly twelve for a real first-order mask, and because the NTT is linear that same offset propagates through all seven layers with no refresh. That's a low-entropy masking scheme that never re-randomizes — exactly what falls to averaging. And third, it covers one block while we'd still need masking on that same block for DPA. What *is* worth taking is the measurement: computing that mutual information for your own representation and register width is a few lines of code and needs no silicon."

---
---

# Timing

| Slide | Time | Cumulative |
|---|---|---|
| 1 · Kyber intro | 5:00 | 5:00 |
| 2 · Generic SCA | 8:00 | 13:00 |
| 3 · Defeating LCC | 8:00 | 21:00 |
| 4 · PQ-Hammer | 8:00 | 29:00 |
| 5 · HW shuffling | 8:00 | 37:00 |
| 6 · Low-cost shuffling | 8:00 | 45:00 |
| 7 · RNR | 8:00 | 53:00 |
| Q&A | 7:00 | **60:00** |

**Delivery note.** With one slide per paper there's no click to hide behind — the slide is fully visible from the first second, so the audience will read ahead. Two ways to manage that: build the six clusters with sequential appear animations so each lands as you reach it, or accept it and use the visible structure as a map ("we'll come back to the bottom-right"). The second is simpler and works fine for a technical audience.

**If you overrun,** the compressible beats are: Paper 2's solver detail, PQ-Hammer's Dilithium aside, Paper 5's method split. Each is worth 45–60 seconds and none is load-bearing.
