# Paper 3 Deep Dive — PQ-Hammer

Amer, Wang, Kippen, Dang, Genkin, Kwong, Nelson, Yerukhimovich.
*PQ-Hammer: End-to-end Key Recovery Attacks on Post-Quantum Cryptography Using Rowhammer.*
**IEEE Symposium on Security and Privacy (Oakland) 2025**, pp. 3567–3582. DOI 10.1109/SP61157.2025.00048.
Free copy: `par.nsf.gov/servlets/purl/10584474` · Code: `github.com/pqrowhammer/pqhammer`

Affiliations: Georgia Tech, UC Berkeley, UMD/Samsung, **NIST**, UNC Chapel Hill, Arkansas, GWU.

Derived material — my own computation, not stated in the paper — is marked **[derived]**.

---

## 1. What this paper is about

**The problem.** Everything else in the SCA literature assumes physical access to a device: probes, oscilloscopes, EM antennas. Rowhammer needs none of that. An **unprivileged process on the same machine** can flip bits in another process's DRAM just by reading memory rapidly. The question this paper asks is whether that primitive is enough to fully recover PQC secret keys.

**What they do.** Three attacks on three NIST schemes, all built from the single primitive of bit flips:

| Scheme | Attack vector | Mechanism |
|---|---|---|
| **Kyber (ML-KEM)** | **Data** flip during KeyGen | Poison `e` → boost decryption failure rate → recover key from failures |
| **BIKE** | **Instruction** flip | Zero the RNG seed → deterministic keypair |
| **Dilithium, deterministic** | **Data** flip on ρ | Nonce reuse → algebraic key recovery |
| **Dilithium, randomized** | **Instruction** flip | Force deterministic keygen |

**What they find.** Full long-term key recovery on all of them, on commodity desktops (Intel i7, Samsung DDR4), with no elevated privileges and no supercomputer.

**Why it matters.** Prior PQC Rowhammer work recovered only *session* keys (FrodoFlips) or *partial* key information (SignatureCorrection). This is the first **end-to-end long-term key recovery**. It also puts a stake in the ground on deterministic Dilithium: don't use it where an attacker can share your machine.

> **One-sentence thesis:** you don't need a probe on the board — a co-located process and a DDR4 stick are enough to take the whole key.

---

## 2. Threat model — and why it's a different animal

| | Power/EM SCA | **PQ-Hammer** |
|---|---|---|
| Physical access | required | **none** |
| Privileges | n/a | **unprivileged user** |
| Attacker location | at the device | **co-located process** (cloud, shared host, browser) |
| Nature | passive observation | **active fault injection** |
| Target | leakage of intermediates | **integrity of stored values** |
| Countermeasure class | masking, shuffling, hiding | redundancy, auditing, memory layout |

Stated assumptions (§3): unprivileged code execution on a machine co-located with the victim, and DRAM susceptible to Rowhammer. **All techniques used — performance degradation, cache flushing, memory profiling, memory massaging — require no elevated privileges.** ASLR and DRAM refresh timing were left at defaults.

Hardware setups (Table 2): Intel i7-6700 / i7-8700K / i7-10700K with Samsung DDR4 8 GiB DIMMs.

---

## 3. Rowhammer primitives you need on one slide

- **The bug.** Repeated activations of a DRAM row ("aggressor") flip bits in adjacent rows ("victim").
- **Double-sided hammering.** Victim row sandwiched between two aggressors. 1s in the victim flip more readily when aggressors hold 0.
- **TRR and why it fails.** DDR4 deploys in-DRAM Targeted Row Refresh, but it tracks only a limited number of rows — so **many-sided** hammering defeats it. This work uses **multi-bank hammering** (bank-level parallelism, from Sledgehammer), with **10 aggressors per bank across 2 banks**.
- **Performance degradation.** Flipping bits needs a long enough window. They flush the victim's hot instructions from cache (HyperDegrade technique) and induce pipeline stalls to slow it down.
- **Memory massaging.** The Linux kernel keeps a **first-in-last-out page frame cache** per core. Unmap the vulnerable page followed by *n* dummy pages, then trigger the victim, and the victim's target page lands on your vulnerable frame ("Frame Feng Shui").

---

## 4. Attack on Kyber — the main event

### 4.1 The target and the window

Look at Kyber's `KeyGen` (Listing 1 in the paper):

```c
gen_a(a, publicseed);
for(i=0;i<KYBER_K;i++) poly_getnoise_eta1(&skpv.vec[i], noiseseed, nonce++);
for(i=0;i<KYBER_K;i++) poly_getnoise_eta1(&e.vec[i],   noiseseed, nonce++);
polyvec_ntt(&skpv);      // <-- THE WINDOW
polyvec_ntt(&e);
```

**Target: the error vector `e`, between sampling and NTT conversion.**

Why that window and no other:
- **Before sampling** the arrays are contiguous zeros — flips there are overwritten.
- **After NTT** you're in the NTT domain, where producing a *controlled* amount of noise in the Euclidean domain would require many precise, coordinated flips. Untenable.
- So the usable window is exactly `polyvec_ntt(&skpv)` — the time spent transforming `s` before `e` gets transformed.

The paper measures ~**35,000 cycles per `poly_ntt` call**; Kyber-1024 calls it 8 times, and they attack during the first seven. Even so, the window is roughly **1400× shorter than a full DRAM refresh cycle** — hence the need for performance degradation.

**Why Kyber-1024?** More `poly_ntt` calls ⟹ the longest available window. Attacking the *strongest* parameter set is *easier* here. Worth saying out loud.

### 4.2 Why exactly two bits, at 2⁸ and 2⁶

Kyber stores coefficients as **16-bit integers** but `e`'s coefficients come from B_η₁ and live in **[−2, 2]**. Setting a high bit turns a tiny number into a huge one — **[derived]**:

| Coefficient | int16 | set bit 8 | set bit 6 |
|---|---|---|---|
| 0 | `0x0000` | **256** | **64** |
| +1 | `0x0001` | **257** | **65** |
| +2 | `0x0002` | **258** | **66** |

The decoding error is `E = eᵀr + e₂ + c_v − sᵀe₁ − sᵀc_u`, and decryption fails when any coefficient of `E` exceeds ⌈q/4⌉. Inflating two coefficients of `e` by 256 and 64 makes the dot product `eᵀr` occasionally blow past the threshold.

**The engineering tension, and it's the heart of the paper:**
- Flip **too few** bits → failure rate stays negligible → no attack.
- Flip **too many** bits → failure rate becomes obvious, or the keypair stops working → detected.

Two bits is the sweet spot. The paper assumes the targeted bits are currently 0 (a high-probability event given the range) and that Rowhammer sets them to 1.

### 4.3 The failure-rate chain — verified [derived]

| Step | Value | Check |
|---|---|---|
| Per-coefficient failure prob. `p` | 2⁻¹⁸ | given |
| Honest user rate `1−(1−p)²⁵⁶` | **2⁻¹⁰·⁰⁰** | ✓ exact |
| Boosted coefficient `p′` | 2⁻¹¹ | given |
| Fraction of failures where `E_k` did **not** exceed (Eq. 5) | **66.6 %** | ✓ paper says ~67 % |
| Total boosted failure rate `p′ + 255p` | 1.46 × 10⁻³ ≈ **1/684** | derived |

The 67 % figure is the paper's honest admission that **two thirds of the observed failures are noise** — they come from some *other* coefficient crossing the threshold, not the poisoned one. That is why key recovery needs an estimator (§4.5 below) rather than plain arithmetic.

### 4.4 Failure boosting

The adversary doesn't wait for random failures. They **resample `r`** until the two coefficients of `r` that multiply the two poisoned coefficients of `e` both have **maximum magnitude (±η₁) and the same sign**.

**[derived] Cost of the filter:** P(a given coefficient of `r` = +2) = 1/16 and likewise for −2. So P(both at max magnitude, same sign) = 2/256 = **1/128**. You throw away 127 of every 128 candidate `r` values — cheap, since it's all local computation.

### 4.5 Recovering the key from failures

The clean idea: each decryption failure is a **hint** that the attacker's known vector points in roughly the same direction as the secret. Average enough hints and the secret emerges up to a scaling factor.

The formal argument (§4.3 of the paper):
1. Write `E₀ = ⟨S, W⟩ + f₁h₁ + f₂h₂ + e₂⁽⁰⁾ + c_v⁽⁰⁾` where `S = (e′, s′)` is secret and `W` is known.
2. A failure means `⟨S, W⟩ > t′`, a known threshold (h₁, h₂ are guessable — the candidate set is tiny).
3. Decompose `W` into its projection onto `S` and an orthogonal part: `W = (α′/ℓ)·S + T`.
4. Taking expectation: **`E[W] = (E[α′]/ℓ)·S`** — the mean of the failing ciphertexts' known vectors is parallel to the secret.

Practical wrinkles the paper is candid about:
- You **can't tell which coefficient failed**. Their estimator guesses it with **≈75 % accuracy**, which was enough.
- Failing ciphertexts must be **rotated** to a common alignment before averaging.
- A **James–Stein estimator** supplies the final scaling factor.

### 4.6 The Rowhammer engineering (this is most of the work)

| Challenge | Solution |
|---|---|
| Window too short | **HyperDegrade**-style performance degradation: flush 1 cacheline in `montgomery_reduce` and 2 in `ntt`; pin flushing processes to the victim's **sibling core** to induce machine clears |
| Need contiguous physical memory | THPs **not** massageable here (target is a **stack** variable), so they use **SPOILER**; work around its inability to distinguish one 2 MB block from two 1 MB blocks by detecting multiple sets of 256 contiguous pages with a 2 MB-aligned probe address |
| Find same-bank addresses | **DRAMA** to recover bank address bits (all within the low 20 bits) |
| Get the flips | Multi-bank hammering, 10 aggressors × 2 banks, striped 1-0-1 pattern, `clflushopt` |
| Get the victim to use the page | **Frame Feng Shui** via the FILO page frame cache |

### 4.7 Results

| Metric | ASLR **off** | ASLR **on** |
|---|---|---|
| Page massaged successfully | 91/100 | 44/100 |
| Correct bits (and only those) flipped | **8/100** | **3/100** |
| Conditional flip success | 8.8 % | 6.8 % **[derived]** |

**[derived] Read this carefully:** ASLR barely affects the *flip* (8.8 % → 6.8 %); it halves the *massaging* success (91 → 44). ASLR's protection here comes from randomizing the stack variable's page offset, not from hindering the hammer.

**Timing.** Profiling **49 s**; massaging + hammering **93 s** (slowed by the degradation). ≈142 s per attempt.

**Stealth.** Geekbench 6 with degradation running: single-core 1674 → 1543 (**−7.8 %**), multi-core 5600 → 1648 (**−70.6 %**) — matching the paper's 8 % / 71 % ✓. Their own note: *"the machine was snappy and reactive, without any noticeable lag."*

**Key extraction.** 4.5 min per recovery iteration on a Xeon Gold 5420+.

| Case | Iterations |
|---|---|
| ASLR off, known flip locations | **1** |
| ASLR on, known bit separation | avg 512 (they observed 33 and 940) |
| ASLR on, **unknown** bit separation | up to 1024² → **≈54 core-months [derived]**, paper says ~60 |

---

## 5. Attack on BIKE — instruction flipping

Completely different vector, and simpler.

**Observation:** BIKE's reference implementation seeds its SHAKE256-based PRF from a single call to the system RNG. From then on, **the entire private key is a deterministic function of that seed**.

**Mechanism:** flip bits in the **binary itself**, not in data. The trick is the **OS page cache** — when several processes read the same file, they share one physical copy. Map the binary read-only, hammer it, and the victim executes your modified instructions.

**Target** (Listing 2): the `mov %rbx,%rdi` that sets up the argument to `get_seeds`. Flipping `89`→`88`, or `df`→`de` or `db`, yields **a seed of 0** and a fully deterministic keypair, with no error reported.

**Extra step:** run a memory-exhaustion process first, so the binary is evicted and gets re-loaded onto the chosen vulnerable page.

**Result:** 198 unique bit flips found before hitting the right page offset; hammering repeated 5× to replicate the flip. **No performance degradation needed** — the binary sits in memory indefinitely.

---

## 6. Attacks on Dilithium

### 6.1 Deterministic Dilithium — nonce reuse

Simplified signing: expand public seed ρ → matrix A; loop {sample nonce `y` from (M, s₁); sample challenge `c` from (A, y, M); `z := y + c·s₁`; reject if unsafe}.

**The algebra.** Two signatures on the same message with the **same nonce** but different challenges:

```
z  = y + c·s₁
z′ = y + c′·s₁      ⟹      s₁ = (z − z′)/(c − c′)
```

(`c − c′` is invertible with very high probability.)

**Target selection — a nice piece of reasoning.** They need a variable that (i) contaminates `c`, (ii) does **not** contaminate `y`, and (iii) is a static buffer that persists. **ρ** satisfies all three: it feeds `A` (hence `c`) but not `y`, and it's a static public buffer. Conveniently, the reference implementation packs ρ into the **first 32 bytes** of the secret key buffer, which the signing server maps page-aligned — putting ρ at offset 0 of a page.

**The complication and its fix.** A faulty ρ may cause the rejection loop to terminate at a *different* iteration, giving a different `y` that doesn't cancel. Their filter:

- Same iteration → `Δz = s₁(c′ − c)`. Since `s₁`, `c`, `c′` all have **low** Hamming weight, `Δz` does too.
- Different iteration → `Δz = s₁(c′ − c) + y′ − y`. Since `y`, `y′` have **high** Hamming weight, so does `Δz`.

So **HW(Δz) sorts the usable pairs from the useless ones.** Prior work puts the per-pair success rate at ~14 %.

**Result:** 396 unique bit flips before finding a suitable one at index 7; 10,000 faulty signatures collected; key recovered. No performance degradation needed.

### 6.2 Randomized Dilithium — instruction flip

Same technique as BIKE. In `crypto_sign_keypair`, flip the argument of `shake256` from `0x20` to `0x00` (Listing 4, line 5), making `rho`, `rhoprime`, and `key` deterministic — hence a deterministic keypair.

**The policy takeaway the paper draws:** Dilithium's documentation makes the **deterministic** variant the default except where side-channel attacks are a concern. This paper argues that deterministic Dilithium **should not be used anywhere Rowhammer is a concern** — which includes any shared or cloud machine.

---

## 7. Countermeasures (§7)

### Implementation defenses

| # | Recommendation | Detail |
|---|---|---|
| 1 | **Reduce key exposure time** | Minimize the window where key material sits in a vulnerable representation. For long-lived material, keep a **redundant copy or hash** and verify before use |
| 2 | **Key auditing** | The spec says `e ∈ [−2,2]`; the attack produces coefficients of 256 and 64. **The key owner can simply check.** Publicly, generate many test ciphertexts and reject any key that produces failures |
| 3 | **Memory layout unfriendly to Rowhammer** | **Bit-pack** the error terms. The reference code stores 5 possible values in 16-bit integers; **[derived] 68.75 % of coefficients are non-negative and fit in 2 bits**, leaving 14 zero bits per coefficient, each a candidate for a useful flip. Pack before NTT conversion and *the error term cannot exceed spec range at all* |
| 4 | **Code hardening** | Don't depend on a single call to the system RNG — a single point of failure. Check that seeds have Hamming weight above a threshold; audit generated keys for low-entropy seeds |

Item 3 is the strongest of the four: it doesn't detect the attack, it makes the attack **arithmetically impossible**, because the poisoned value has nowhere to live.

### System-level Rowhammer defenses

- **Counter-based mitigations** (Graphene, Hydra) — track activations per row. As DRAM densifies, thresholds fall and SRAM tracker cost becomes prohibitive; recent work (START) proposes reusing the LLC as the counter.
- **TRR** (DDR4) — largely ineffective against many-sided hammering.
- **ECC memory** — raises the bar but does not guarantee protection.
- **Software isolation** — proposed, not widely adopted.
- **Hardware-accelerated cryptography** — the paper's own closing observation: their Kyber attack **needed** performance degradation, which was possible only because `polyvec_ntt` has nested loops whose deepest level can be flushed from the instruction cache. A vectorized or hardware implementation removes that lever.

---

## 8. Numbers that don't reconcile — [derived]

Three things a careful reader will trip over. Worth knowing before you present.

**(a) The decryption failure rate is stated three different ways.**

| Location | Value | As a fraction |
|---|---|---|
| §4.2 analysis | 1 − (1−2⁻¹⁸)²⁵⁶ ≈ **2⁻¹⁰** | 1/1024 |
| §4.2 experiment | "just under **2⁻¹⁰·⁵**" | 1/1448 |
| §4.5 and §7.1 | "**~1/10,000**" | 1/10000 |

The first two are consistent; the third is **6.9× lower**. Also, §4.2's phrasing — *"slightly higher than 2⁻¹⁰, landing just under 2⁻¹⁰·⁵"* — is self-contradictory, since 2⁻¹⁰·⁵ < 2⁻¹⁰.

**(b) The auditing claim is overstated at the lower rate.** §7.1 says a 1/10,000 failure rate is detectable "with over 99.9999 % accuracy" from 100k test ciphertexts. At that rate you expect 10 failures, so P(zero failures) = e⁻¹⁰ = 4.5×10⁻⁵ and detection is **99.9955 %** — three nines short of the claim. (At 2⁻¹⁰·⁵ you'd expect 69 failures and detection really is ~100 %.) The recommendation is sound; the stated accuracy isn't, at the rate quoted alongside it.

**(c) The ciphertext accounting is loose.** §1.3 says ~40,000 failing ciphertexts obtained through "only 4MM encryption calls." At the boosted rate I compute (≈1/684), 40,000 failures needs ~2.7×10⁷ encapsulations; at 1/10,000 it needs 4×10⁸. Neither is 4×10⁶. The stated "about 2 hours to generate on a modern PC" is more consistent with 10⁷–10⁸ than with 4×10⁶.

None of this undermines the attack — the experiments demonstrably worked. But quote the raw experimental outcomes (8/100, 3/100, 142 s, 4.5 min/iteration) rather than the derived rates.

---

## 9. Critical assessment

**Strengths**

- **Genuinely end-to-end.** Prior work got session keys or partial information; this recovers long-term keys, verified experimentally 100 times per configuration.
- **Three distinct vectors from one primitive** — data flips, instruction flips, and control-flow manipulation — showing the breadth of what a single bit-flip capability buys.
- **The ρ target selection for Dilithium** is elegant: three requirements, one variable satisfying all three.
- **The HW(Δz) filter** is a clean, cheap way to separate usable signature pairs.
- **Honest about noise:** the 67 % figure and the 75 % estimator accuracy are stated plainly rather than buried.
- **Constructive countermeasures section**, including one (bit packing) that eliminates the attack rather than detecting it.

**Weaknesses — the program committee flagged two of these itself**

1. **"Challenging to determine if the environmental conditions are realistic"** (PC's own noted concern). Single known-vulnerable Samsung DIMM, default ASLR, reference implementations, debugging under `sudo` "for visibility only."
2. **"Some of the underlying attack techniques are well-known"** (also the PC). SPOILER, DRAMA, Frame Feng Shui, HyperDegrade, multi-bank hammering are all prior art; the contribution is the stitching plus the PQC-specific cryptanalysis.
3. **Reference implementations only.** Optimized/vectorized builds may not have the same instruction-cache lever — a point the paper concedes in §7.2.
4. **Low end-to-end success rate.** 8 % with ASLR off, 3 % with it on. Attempts are cheap and repeatable, but this is not a reliable one-shot.
5. **The unknown-bit-separation case is impractical** — ~60 core-months, which the authors leave to future work.
6. **Randomized Dilithium is not broken cryptanalytically**, only via keygen instruction flips. Extending the deterministic attack to the randomized variant is stated as an open question.

---

## 10. Relevance to hardware IP

**The headline: a hardware crypto IP with on-chip key storage is not a Rowhammer target.** The primitive is a DRAM disturbance effect. Keys in registers, SRAM, or OTP are outside its reach. The paper says as much in §7.2, listing **hardware-accelerated cryptography** as a defense — precisely because the performance-degradation lever depends on flushing nested-loop instructions from a CPU instruction cache, and there is no such cache in an RTL datapath.

That said, four things transfer directly:

**(a) Bit-packing is a free hardware design lever.** The attack exists because 5 possible values are stored in 16-bit words, leaving 14 flippable bits per coefficient. In RTL **you choose the storage width**. Storing an [−2,2] coefficient in 3 bits makes 256 unrepresentable — the fault has nowhere to land. This is the single cheapest structural defense in the paper and it costs nothing in hardware; it may even save area.

**(b) Integrity protection on key registers.** Recommendation 1 (redundant copy or hash) maps onto standard RTL practice: parity or CRC on key storage, checked before use. This buys resistance to the general class of fault injection, not just Rowhammer.

**(c) Key auditing as a self-test.** Recommendation 2 — generate test ciphertexts at keygen and reject a key that produces any failure — is implementable as a built-in self-test. Cheap in a hardware IP where encapsulation is fast. (Size the test using the actual failure rate, not the paper's; see §8(b).)

**(d) The Dilithium result is a live design decision.** For an ML-DSA signer, `s₁ = (z − z′)/(c − c′)` is a total break from a single successful fault, and the target ρ is a static public buffer. Two concrete implications: prefer the **hedged/randomized** signing mode, and treat **ρ as integrity-critical storage** even though it's public. Public does not mean untrusted-safe here — corrupting a public value is what breaks the scheme.

**Where the attack surface returns:** if the IP is a co-processor that fetches key material from **external DRAM**, or if a soft-core CPU runs any part of keygen from DRAM, the primitive applies again. Key transport and residency, not just key computation, belong in the threat model.

**And the honest caveat:** immunity to *Rowhammer specifically* is not immunity to fault injection. The cryptanalysis in this paper — failure boosting plus the averaging estimator, and the nonce-reuse algebra — works against **any** fault model that can perturb `e` during keygen or `ρ` during signing. Laser, clock glitch, and EM fault injection all qualify, and those *do* reach an ASIC.

---

## 11. Slide skeleton (~8 min)

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** — three schemes, one primitive, no probe needed | 0:45 |
| 2 | Threat model contrast: co-located unprivileged process vs. physical probe | 0:45 |
| 3 | Rowhammer in 60 seconds: aggressors/victims, TRR, multi-bank, massaging | 1:00 |
| 4 | Kyber KeyGen listing with the window boxed; why before-NTT is the only option | 1:00 |
| 5 | **Two bits: `0x0002` → 258.** The [−2,2] range in a 16-bit word | 1:00 |
| 6 | Failure boosting + the 67 % noise admission | 1:00 |
| 7 | Key recovery: failures are hints; average them; E[W] ∥ S | 0:45 |
| 8 | Results: 8/100, 3/100, 142 s, Geekbench stealth | 0:45 |
| 9 | Dilithium: `s₁ = (z−z′)/(c−c′)`, target ρ, HW(Δz) filter | 0:45 |
| 10 | Countermeasures — lead with **bit packing** | 1:00 |

**Land this line:** *"The secret is five possible values stored in sixteen bits. Fourteen of those bits do nothing but wait for someone to flip one."*

---

## 12. Self-check exercises

1. Why can't the attacker flip bits in `e` *after* the NTT conversion? *(Producing a controlled amount of noise in the Euclidean domain would require many precise, coordinated flips in the NTT domain — untenable, and more flips means more unintended ones.)*
2. Why does attacking Kyber-**1024** make the attack easier than Kyber-512? *(k = 4 means 8 `poly_ntt` calls, so the longest available hammering window.)*
3. Two bits are flipped, at 2⁸ and 2⁶. Why not one bit, or four? *(One doesn't raise the failure rate enough for practical recovery; too many makes the key visibly broken or inoperable. It's a tuning problem, not a maximization.)*
4. What fraction of observed decryption failures actually come from the poisoned coefficient? *(About 33 % — the other 67 % are from other coefficients crossing the threshold, which is why an estimator is needed.)*
5. What does failure boosting cost the attacker per usable ciphertext? *(Resampling `r` until two specific coefficients both hit ±η₁ with the same sign: probability 2/256, so ~128 samples discarded per usable one — all local computation.)*
6. Why is ρ the right Rowhammer target in Dilithium, and not `y` or `c` directly? *(It contaminates `c` but not `y`, so the nonce cancels; and it's a static persistent buffer rather than a value computed inside the signing loop.)*
7. How does the attacker tell which correct/faulty signature pairs are usable? *(Hamming weight of Δz: low ⟹ same loop iteration and the nonce cancels; high ⟹ different iteration, discard.)*
8. Which countermeasure makes the Kyber attack *impossible* rather than merely detectable, and why? *(Bit-packing the error coefficients. With 3 bits of storage the poisoned value 256 is unrepresentable, so the fault cannot produce an out-of-spec coefficient at all.)*
