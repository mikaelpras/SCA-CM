# Kyber SCA Seminar — 7-Slide Deck (5-cluster format)

1 intro + 6 papers · ~53 min · keyword-only slides, narration carries the detail

**Convention:** `**bold**` = highlight visually (colored pill or box, not just bold weight). Nothing on a slide is a sentence.

**Layout — every slide, same skeleton:**

```
┌─────────────────────────────────────────────────┐
│  Paper title                                    │
│  Authors · Venue                                │
├─────────────────────────────────────────────────┤
│  ① SUMMARY — full width                         │
├──────────────────────┬──────────────────────────┤
│  ②                   │  ③                       │
├──────────────────────┼──────────────────────────┤
│  ④                   │  ⑤                       │
├──────────────────────┴──────────────────────────┤
│  KEY RESULT — one number, highlighted           │
└─────────────────────────────────────────────────┘
```

| | Attack papers (1–3) | Countermeasure papers (4–6) |
|---|---|---|
| ① | attack summary / idea | protection point + CM idea |
| ② | attack point & model | CM procedure |
| ③ | attack procedure | implementation |
| ④ | result & evaluation | advantage / disadvantage |
| ⑤ | what to protect & countermeasure | apply in our IP? |

---
---

# SLIDE 1 — Kyber / ML-KEM

**Header:** Kyber / ML-KEM (FIPS 203) — the standardized KEM

**① Foundation**
Module-LWE · `R_q = Z_q[X]/(X²⁵⁶+1)` · q = 3329 · n = 256 · k = 2/3/4
⚠ Round-2 Kyber512: η₁ = 2, d_v = 3

**② Two layers**
IND-CPA PKE ⊂ IND-CCA KEM
**FO transform** — the boundary is where everything happens

**③ IND-CPA PKE**
`t̂ = Â∘ŝ + ê`
`v = t̂ᵀ∘r̂ + e₂ + ⌈q/2⌋·m`
`m′ = v′ − ŝᵀ∘û`

**④ Decaps (FO)**
decrypt → **hash** → re-encrypt → compare
derandomization · **implicit rejection**

**⑤ Where the secret shows**
A. `ŝᵀ∘û` direct
B. everything after m′
C. integrity — e, comparison, seed

**KEY:** **m′ = ⌈q/2⌋·m + Δm** , Δm = ⟨e,r⟩ − ⟨s, e₁+Δu⟩ + e₂ + Δv — **linear in the secret**

### Script (5:00)

*① 45s.* "One prime, one ring, three parameter sets — only the module rank changes. A warning for the older literature: Round-2 Kyber512 used different parameters, which changes how many candidate values a secret coefficient has and therefore what attacks cost. Papers from 2020 assume the old numbers."

*② 45s.* "Hold both layers in your head for the next hour. The inner PKE falls trivially to chosen-ciphertext attacks — that was never its job. The FO transform buys CCA security. Almost everything physically interesting about Kyber happens at the boundary between them."

*③ 1:15.* "Message bits are scaled to half the modulus and buried in noise; decryption subtracts the mask and rounds. Look at the key line at the bottom. Two consequences. The noise term is **linear in the secret** — the message polynomial isn't just noisy, it's noisy in a way that depends on the key. And because that noise is random, decryption can fail. Parameters push the rate below two to the minus one twenty-eight, but it isn't zero, and failures leak. Both get exploited today."

*④ 1:15.* "Encapsulation derives its randomness from the message by a hash, so an honest ciphertext is reproducible — decrypt, re-derive, re-encrypt, compare. On mismatch the device doesn't return an error; it returns a pseudorandom key. From outside you cannot tell accept from reject. On paper that closes the door completely."

*⑤ 1:00.* "Three classes. The arithmetic touches the secret directly — obvious target. Region B is the interesting one: to *check* a ciphertext the device must decrypt it, hash the result, and re-encrypt, and only then reject. Implicit rejection hides the output. It does nothing about the computation. And a third class most people skip — values that break the scheme if *corrupted* rather than observed. Some of those are public. Public doesn't mean safe to corrupt."

---
---

# SLIDE 2 — Generic SCA on CCA-secure Lattice KEMs

**Header:** Ravi, Sinha Roy, Chattopadhyay, Bhasin · TCHES 2020(3)

**① Attack summary / idea**
FO must **hash the decrypted message** to verify it — and the hash runs **before** the rejection
EM leakage from that hash = a **plaintext-checking oracle**
Chosen-ciphertext attacks resurrected on CCA-secure KEMs · algorithm-level, not implementation-level

**② Attack point & model**
Point: **G(m′ ‖ H(pk))** · ECC decode (Round5, LAC)
Model: static key · unlimited chosen CT · passive EM
Blind to output · **no clone device** — self-profile with k_u = 0

**③ Attack procedure**
`u₀[0] = k_u`, `v[0] = k_v`, rest zero
**k_u ≤ 416** → only 2 messages reach G
t-test → PoI → reduced template → classify O/X
5 (k_u, k_v) pairs per coefficient
anti-cyclic sweep `u = k_u·x^q` → all 512

**④ Result & evaluation**
STM32F407 · pqm4 · EM · 500 MSa/s
Kyber512 **2560 → 7.7k traces**, 99 %
Round5 4.5 min · LAC128 25 min
⚠ only **3 of 6** schemes measured

**⑤ Protect & countermeasure**
Masking the NTT alone = **useless**
Protect m′ → Keccak → re-encrypt → compare
**q = 3329 prime** → masked A2B is hard
Verdict: mask the full decapsulation

**KEY:** **10.8 minutes to a full Kyber512 key**

### Script (8:00)

*① Summary — 1:15.* "This is the paper that reopened chosen-ciphertext attacks on CCA-secure lattice KEMs. Prior key-mismatch attacks only worked on CPA-secure schemes, because the FO transform removes the oracle they need. D'Anvers found a timing leak in error-correcting decoders, but constant-time code kills that. This paper's insight is structural. The FO transform has to hash the decrypted message in order to verify it — and that hash happens *before* the rejection. By the time the device decides to discard the result, it has already processed the thing the rejection was supposed to hide. That's why the attack is algorithm-level: it doesn't care how your NTT is written, whether the code is constant-time, or which compiler you used."

*② Point & model — 1:30.* "For Kyber the target is G, which is SHA3-512 over sixty-four bytes — one Keccak permutation, twenty-four rounds. They sample the *late* rounds, where a one-bit input difference has become roughly a fifty-percent state difference. The hash's own avalanche is the attacker's amplifier; the cryptographic strength of the primitive is exactly what makes the side channel loud. The model is a device decapsulating with a long-term key, unlimited chosen ciphertexts, passive EM, and — importantly — the attacker never sees the output. Nor do they need a clone device: set k_u to zero and the secret drops out of the equation, so the ciphertext decrypts to whatever you choose regardless of the key. Templates get built on the victim's own hardware."

*③ Procedure — 2:15.* "One knob, one secret coefficient. Set a single coefficient of u and a single coefficient of v, everything else zero. Then the first message bit becomes a function of one secret coefficient alone. The constraint is what makes it work: keep k_u small enough that every *other* message bit decodes to zero no matter what the secret is. That caps k_u at four hundred sixteen, and it means the device only ever hashes one of two fixed thirty-two-byte inputs. A single pair of templates classifies all twenty-five hundred queries. Then Welch's t-test locates the leaking sample points, a reduced template classifies each trace as one of two classes, and five well-chosen query pairs pin down a coefficient uniquely. Finally, multiplying by a power of x rotates the secret in the ring, so the same trick reaches all five hundred twelve coefficients. Watch the sign — on the rotated ones you read back the negated value, which is a classic implementation bug."

*④ Result — 1:30.* "Eleven minutes for a full key on a Cortex-M4. The gap between twenty-five hundred queries and seven thousand seven hundred traces is three-times majority voting to absorb oracle errors — the per-query accuracy is about ninety-nine percent, which isn't enough on its own across two and a half thousand queries. One honest caveat worth stating out loud: the title says six schemes, but only three were actually measured. Saber rides on a structural-similarity argument, NewHope is simulation against a perfect oracle, and Frodo is an appendix sketch with no numbers at all."

*⑤ Protect — 1:30.* "This paper diagnoses; it doesn't cure. The recommendation is masking everything from the decrypted message onward — through Keccak, through re-encryption, through the comparator. Note what that excludes: masking the polynomial arithmetic alone achieves nothing, because the leak is downstream of it. And Kyber's prime modulus makes the arithmetic-to-Boolean conversion considerably harder than a power-of-two modulus would, which is a large part of why the masking literature found Saber friendlier. If you take one thing away: masking the NTT protects the wrong operation. The secret leaves through the hash that proves the ciphertext was honest."

---
---

# SLIDE 3 — Defeating Low-Cost Countermeasures

**Header:** Ravi, Paiva, Jap, D'Anvers, Bhasin · TCHES 2024(2)

**① Attack summary / idea**
Two cheap **detection** countermeasures: ciphertext sanity check · decapsulation failure check
Break both — one by **masking the ciphertext with the public key**, one by using **only valid ciphertexts**
Bonus: a greedy inequality solver that beats belief propagation

**② Attack point & model**
A: same PC-oracle point in the FO transform
B: **the `m′` store in PolySub** — HW of a signed int16
Model: physical access · long-term key · **all ciphertexts valid** · clone device for profiling

**③ Attack procedure**
A: send `(u+a*, v+b*)` → A·s cancels → 57.8 %, retry (0.73 avg)
B1: leak HW(m′[i]) — CPA → PoI → random forest
B2: **+5 → HW 2 · −5 → HW 14** = sign detector
B3: empirical min/max per HW → 2 inequalities each
B4: greedy solve `Hx + w ≥ 0`

**④ Result & evaluation**
STM32F407 · Kyber768 · EM · 1.25 GSa/s
ref −O3 **5200 measured** (325 extrapolated) · asm 7800
shuffled > 8 M · masked ~10⁵
greedy: ½ inequalities · σ = 2.0 · 20 s vs 10 min
⚠ masked test used C at −O0

**⑤ Protect & countermeasure**
Detection = **no standalone protection**
Protect: the m′ store · **all shares** when masked · the **representation**
Wide parallel datapath: 1 coeff → 91 %, 10 coeff → 32 %

**KEY:** **valid ciphertexts never fail — so the countermeasure never fires**

### Script (8:00)

*① Summary — 1:15.* "Masking a full decapsulation is expensive, so people proposed two cheap alternatives that try to *detect* the attack instead of hiding the data. The first rejects low-entropy ciphertexts — attack ciphertexts have one non-zero coefficient out of seven hundred sixty-eight, trivially detectable, and it rejects *before* decapsulation so the attacker sees no leakage at all. The second refreshes the key on the first decapsulation failure, capping the attacker at a single trace. Both are attractive, and the second is genuinely strong. This paper breaks both, and the second break requires a completely new attack."

*② Point & model — 1:15.* "For the first attack, nothing changes about where the leak is — it's the same plaintext-checking oracle in the FO transform. The second attack is where the novelty is. Because the ciphertexts must be valid, re-encryption succeeds, no failure ever occurs, and the whole FO transform closes to the attacker. They're confined to the decryption procedure — specifically the store instruction that writes each coefficient of the message polynomial to memory. The leakage model is just the Hamming weight of a signed sixteen-bit value."

*③ Procedure — 2:15.* "Attack A first, and it's four lines of algebra. Add a row of the public key to the attack ciphertext. The A-times-s terms cancel and you're left with the original quantity plus a small error. The ciphertext now looks like a genuine LWE sample and passes the entropy test — and nothing downstream changes, so any existing attack ports across unmodified. It works fifty-eight percent of the time, and here's why, which the paper doesn't explain: every column of the decision table has one cell sitting exactly one unit below the decode threshold. For that candidate the oracle is decided purely by the sign of the added error. A coin flip — which is also why the fix is simply to retry with a different mask, and you expect less than one retry.

Attack B is the real contribution. For a valid ciphertext the message isn't key-dependent — but the *noise* is. The message polynomial equals the scaled message plus a noise term, and that noise term is linear in the secret. Because the attacker generated the ciphertext honestly, they know every term except the two they want. Now the cleverest idea in the paper, and it isn't about Kyber at all — it's about how signed integers sit in memory. Plus five has two bits set. Minus five has fourteen. So when the message bit is zero, the Hamming weight is a near-perfect *sign detector* for the noise. When the message bit is one, the value sits up near sixteen sixty-four, no sign boundary is crossed, and you learn nothing. That halves the usable data. They then upgrade the sign to two-sided bounds with an empirical table, and solve the resulting inequalities greedily."

*④ Result — 1:30.* "Two caveats worth stating. The famous three-hundred-twenty-five figure is extrapolated — they measured fifty-two hundred using eight zero-bits and scaled by sixteen. And the masked evaluation replaced the library's assembly with C compiled at dash-O-zero, which is why masked accuracy comes out *better* than unprotected optimized. That's a compiler-flag artefact, not a masking result. The solver comparison is the cleanest part of the evaluation: half the inequalities, four times the noise tolerance, twenty seconds against ten minutes. And it's the most portable piece of the paper — it works on any leakage that yields linear inequalities in the secret, including fault attacks."

*⑤ Protect — 1:45.* "The headline verdict: detection assumes the attacker needs invalid ciphertexts. Once he doesn't, there's nothing to detect. So don't build these — you'd pay the area and gain nothing. What actually needs protecting is the message-polynomial store, every arithmetic share of it when masked, and — the interesting one — the *representation* itself, since the two's-complement asymmetry is the enabler. There's also a free lever hiding in their own numbers: one coefficient per store gives ninety-one percent classifier accuracy, ten coefficients per store gives thirty-two. A wide parallel datapath aggregates Hamming weights and attenuates this for nothing. It raises cost rather than closing the hole — thirty-two percent still recovered keys — but it's free."

---
---

# SLIDE 4 — PQ-Hammer

**Header:** Amer, Wang, Kippen, Dang, Genkin, Kwong, Nelson, Yerukhimovich · IEEE S&P 2025

**① Attack summary / idea**
Rowhammer: an **unprivileged co-located process** flips bits in the victim's DRAM
Kyber — poison `e` during KeyGen → boost the decryption failure rate → recover the key from failures
First **end-to-end long-term** key recovery on PQC (prior work: session keys or partial info)

**② Attack point & model**
Point: **`e` between sampling and NTT** · ρ in Dilithium · instruction flips (BIKE)
Model: **no physical access** · unprivileged · same machine · DDR4
**Active fault injection** — integrity, not confidentiality

**③ Attack procedure**
profile → massage (Frame Feng Shui) → degrade → multi-bank hammer
**[−2,2] in int16 → 14 free bits** · bit 8 → 256 · bit 6 → 64
failure boosting: resample `r` (keep 1 in 128)
average the failures → **E[W] ∥ S** · James–Stein for scale

**④ Result & evaluation**
i7-8700K · Samsung DDR4 · Kyber-1024 reference
**8/100** ASLR off · 3/100 on · 142 s per attempt
~40k failing CTs (~2 h) · 4.5 min per recovery iteration
Geekbench −8 % / −71 %, *"snappy and reactive"*
⚠ failure rate stated three different ways

**⑤ Protect & countermeasure**
**Bit-packing** → the attack becomes arithmetically impossible
key auditing · redundant copy or hash · seed HW check
ECC memory and TRR raise the bar only
HW accelerator removes the degradation lever

**KEY:** **two bit flips** · 8/100 attempts · no probe required

### Script (8:00)

*① Summary — 1:15.* "Everything else today assumes physical access to the board. This assumes only that you can run an unprivileged process on the same machine. Rowhammer flips bits in a victim's DRAM by hammering adjacent rows. Prior work on post-quantum Rowhammer fell short: FrodoFlips needed eight exact bit flips, managed seven, and required six hundred sixty-five thousand failing ciphertexts and a supercomputer — recovering only session keys. SignatureCorrection got partial key information on Dilithium. This paper gets the whole long-term key on three schemes, on a desktop, with two bit flips."

*② Point & model — 1:15.* "Note what changes about the threat model. This is *active fault injection*, not passive observation, and the property under attack is integrity rather than confidentiality. Masking does nothing here — the relevant defences are redundancy, auditing, and memory layout. For Kyber the target is the error vector during key generation, in a specific window I'll come to. For Dilithium it's the public seed rho. For BIKE it's an instruction in the binary, reached through the shared OS page cache. Same primitive, three completely different vectors."

*③ Procedure — 2:15.* "The window first. In Kyber's key generation there's exactly one usable moment: after the error vector is sampled, before it's transformed to the NTT domain. Before sampling, the array is contiguous zeros and flips get overwritten. After the transform, producing a *controlled* amount of noise in the original domain would need many coordinated flips. And that window is roughly fourteen hundred times shorter than a DRAM refresh cycle, which is why they need performance degradation — flushing the victim's hot instructions from cache and pinning stall processes to its sibling core.

Now the enabler, which is the whole attack in one observation. The error coefficients live between minus two and plus two — five possible values — stored in sixteen-bit integers. Fourteen of those bits do nothing but wait for someone to flip one. Set bit eight and a coefficient of two becomes two hundred fifty-eight. That inflates the dot product enough to push decryption over the failure threshold usefully often, but not so often the keypair looks broken. Too few flips and the rate stays negligible; too many and it's obvious. Two is the sweet spot — a tuning problem, not a maximization problem.

Then recovery. They filter the encryption randomness so the coefficients multiplying the poisoned ones are at maximum magnitude with matching sign, discarding a hundred twenty-seven of every hundred twenty-eight — all local computation, so it's cheap. Each decryption failure is a *hint* that the attacker's known vector points roughly along the secret. Average enough of them and the secret emerges up to a scaling factor."

*④ Result — 1:30.* "Eight percent success with ASLR off, three with it on — and notice ASLR mostly hurts the *massaging*, not the flip itself. Each attempt is about two and a half minutes, and attempts are cheap and repeatable. The stealth number is worth mentioning: with the degradation running, multi-core benchmark score drops seventy-one percent, but the authors report the machine stayed responsive with no noticeable lag. One thing to be careful about — the paper states its decryption failure rate three different ways and they don't reconcile, so quote the raw outcomes rather than the derived rates. The paper is also honest that about two thirds of observed failures come from the wrong coefficient, which is why the recovery needs an estimator."

*⑤ Protect — 1:45.* "Lead with bit-packing. It doesn't *detect* the attack — it removes the space the fault needs to live in. Store a value from minus two to two in three bits and two hundred fifty-six is simply unrepresentable. Everything else on the list is detection or mitigation; that one is prevention. Key auditing works too: the specification says the error coefficients live in a narrow range, and the attack produces values of two hundred fifty-six and sixty-four, so the key owner can simply check. At the system level, ECC memory and DDR4's row-refresh mitigation raise the bar without closing it. And the paper's own closing observation is relevant to anyone building hardware: their attack needed performance degradation, which was only possible because the reference NTT has nested loops whose innermost instructions can be flushed from cache. A hardware accelerator removes that lever entirely — though note that immunity to Rowhammer specifically is not immunity to fault injection. The cryptanalysis here works against any fault model that can perturb the error vector, including laser and glitch attacks that do reach silicon."

---
---

# SLIDE 5 — Hardware-Friendly Shuffling for Kyber

**Header:** Xu, Wang, Tian (Nanjing University) · IEEE TCAS-II 72(3), 2025

**① Protection point & CM idea**
Point: PWM · modular reduction · subtraction · **INTT** — all four
Idea: **shuffle addresses, not arithmetic** — the datapath is never restructured
Hardware-friendly Fisher–Yates via a Random Permutation Generator + Address Controller

**② CM procedure**
LFSR runs 384 cycles → fills a 64×6 FIFO up front · **one-shot TRNG seed**
FIFO **reused**: indices out, permutation elements in
Range reduce: `idx − 0x28` then **`idx & rest`** — fixed latency, no divider
Two-stage per_a / per_b → 12 risky slots **safe by construction**
Cyclic shift by 6

**③ Implementation**
RPG: 64×6 REG · MUX/DEMUX · FIFO · LFSR · CTRL
ADDR: 6 muxes · **1-cycle + 12-cycle shift registers** for write-back alignment
One 6-bit permutation → **5 address ranges** by concatenation
Artix-7 XC7A100T · +790 LUT / +518 FF / +260 slice

**④ Advantage / disadvantage**
✓ **+8.7 % ATP · 0 cycles · 0 DSP/BRAM** · one mechanism, four points
✗ **32-bit LFSR** caps entropy at 32 bits · biased AND reduction
✗ only CPA and TVLA tested · 25× shown vs ~N² = 4096× theory

**⑤ Apply in our IP?**
**Yes** — a mux on the address bus plus a delay-matched write path
Swap LFSR → real DRBG · power-of-2 ranges → **unbiased, free**
⚠ 8.7 % is an **FPGA slice ratio**, not an ASIC number

**KEY:** CPA 4×10³ → nothing at 10⁵ · **+8.7 % ATP, zero cycles**

### Script (8:00)

*① Summary — 1:15.* "Kyber hardware leaks — prior work identified three points in the decryption datapath and broke them with correlation power analysis. Both defence families have a cost problem. Hardware masking costs roughly seventeen hundred times the area-time product. Existing hiding schemes burn either cycles or area. This paper takes an open-source unprotected Kyber FPGA design and adds shuffling for eight point seven percent and zero extra clock cycles. The cheapness comes from one architectural decision: they never touch the arithmetic. They permute the memory *addresses* the datapath visits. The datapath computes exactly what it computed before — it just doesn't know which coefficient it's holding. That's why one mechanism covers all four leakage points, including an inverse-NTT protection the authors add on their own initiative."

*② CM procedure — 2:00.* "Fisher–Yates gives a uniform permutation but doesn't fit hardware, for two reasons. It needs a fresh random number every iteration, and a true random generator that fast is expensive and is itself an attack surface. And the index range shrinks each iteration, so uniform reduction needs a divider or rejection sampling — and rejection sampling has *variable latency*, which is poison for a fixed-schedule datapath.

Their first fix: generate all the randomness before you start. A thirty-two-bit LFSR runs three hundred eighty-four cycles at initialization and fills a FIFO with every index the permutation will ever need. The true random generator supplies one seed, not a stream. And the neat part — the same FIFO that empties of random indices refills with the permutation being built. One memory, two roles, occupancy constant at sixty-four.

Second fix: replace the modulo with a subtract in stage one and a bitwise AND in stage two. That's the entire reduction — constant latency, almost no area.

Third fix, and this is the subtle one. The datapath is pipelined: a read at cycle t writes back at t plus one or t plus twelve. Apply a naive permutation and certain adjacent addresses collide, and the computation is simply *wrong*. Rather than checking constraints at runtime, they design a permutation with two regions where one region satisfies every constraint by construction — its minimum sits above the largest lower bound, its maximum below the smallest upper bound. Fill the twelve risky slots from that region and no conflict is possible."

*③ Implementation — 1:30.* "The permutation generator is five parts: a sixty-four by six register file with a mux and demux, the dual-purpose FIFO, the reduction arithmetic, the LFSR, and a small controller. No multiplier, no divider, no stored permutations — that's why it's two hundred sixty slices.

The address controller is where the non-obvious engineering lives. Six two-to-one muxes swap in the shuffled addresses for four reads and two writes. But because the pipeline writes back later than it reads, you need one-cycle and twelve-cycle delayed copies of the permutation so the write uses the *same* permuted address the read used. That's the detail you'd have to rediscover if you built this yourself. There's also a nice reuse: the design needs five different address ranges, and rather than five generators they derive all five from one six-bit permutation by concatenating bits."

*④ Advantage / disadvantage — 1:45.* "Zero cycle overhead is real — the permutation generator runs in parallel and needs about four hundred fifty cycles against six to ten thousand for a decapsulation, with a shorter critical path. One note on the headline number: the area metric adds a constant for DSPs and block RAMs to both designs, and the countermeasure uses neither, so the percentage is diluted. On slices alone it's twelve percent. Both defensible; just know which you're quoting.

The weaknesses are real too. The thirty-two-bit LFSR caps the permutation entropy at thirty-two bits, no matter how large sixty-four factorial is — three hundred eighty-four output bits from thirty-two state bits, linearly related, so profile once and predict forever. The reduction is biased and never analysed. And only CPA and TVLA were tested, not the second-order or permutation-recovery attacks that actually matter against hiding. The theory for sixty-four slots predicts about four thousand times more traces; they demonstrated twenty-five times, which is where they stopped collecting rather than a security bound."

*⑤ Apply in our IP — 1:30.* "Yes, and this is the paper of the six that transfers most directly, because it's the only one that's actually RTL. The core idea works in any memory-based accelerator with banked coefficient storage: a mux on the address bus plus a delay-matched write path. Three changes I'd make first. Replace the LFSR with a real deterministic random bit generator — if there's a hash core on the die already, a Keccak-based DRBG is nearly free. Restructure to power-of-two block sizes: the AND then becomes exactly a modulo, giving an unbiased shuffle for zero extra hardware, which is strictly better than what they do. And check that safe-by-construction claim against the actual HDL, because the algorithm as published backfills the constrained region from the *other* region.

One caveat for our estimating: don't quote eight point seven percent as an ASIC number. A sixty-four-to-one mux over six-bit words is cheap in FPGA LUTs where wide muxes map onto LUT trees. In standard cells it's a real mux tree plus three hundred eighty-four flops. Small in absolute terms, but re-estimate against our own budget."

---
---

# SLIDE 6 — Low-Cost Shuffling for NTT-based PQC

**Header:** Chen, Ma, Jing (Chinese Academy of Sciences) · IEEE TCAD 42(1), 2023

⚠ *PDF not yet obtained. Structure below from the abstract and citing work — fill in once you have it.*

**① Protection point & CM idea**
Point: the NTT · nested loops converted to a **single-level loop**
Idea: a **unified shuffling controller** driving two randomization schemes

**② CM procedure**
Scheme 1: **coefficient index randomization**
Scheme 2: **NTT network randomization**

**③ Implementation**
FPGA · unified controller across NTT stages
No full permutation storage

**④ Advantage / disadvantage**
✓ **~9 % resource overhead** · negligible performance impact
✓ covers power analysis and template attacks
✗ ⚠ see below

**⑤ Apply in our IP?**
TBD — depends on the permutation-space question

**KEY:** **~9 % resource overhead** · ⚠ verify: rotation by N/2, or full permutation?

### What to establish when you get the PDF

A later ACM TECS paper describes their index scheme as using a **one-time rotation of the NTT index via modular addition of N/2**, rather than a full Fisher–Yates permutation. If that's accurate, it's a random *starting offset* rather than a random *ordering* — a dramatically smaller permutation space, and the security claim needs scrutiny. That's the single most important thing to pin down, and it's worth raising as an open question even if you can't resolve it before the talk.

Second thing to check: what "NTT network randomization" actually randomizes — butterfly assignment, stage order, or datapath routing. The three have quite different security properties.

---
---

# SLIDE 7 — Redundant Number Representation

**Header:** Nagpal, Hadžić, Primas, Mangard · SAC 2025

**① Protection point & CM idea**
Point: **NTT / INTT — the representation only**, not the arithmetic
Problem: q ≈ 2^11.7 stored in a **16-bit word** → Hamming weight fully determines the value
Idea: compute in **Z_ηq** so each value has **η machine encodings**

**② CM procedure**
Find the largest η with no butterfly overflow → **η± = 9**, η⁺ = 5
Encode `X′ = X + K·q`, K random
Compute **entirely in Z_ηq**, adding Barrett reductions where the range invariant breaks
Decode: reduce mod q

**③ Implementation**
Delta = rejection-sampling bounds + reduction constants
**No extra operations, no fresh randomness mid-computation**
2 extra Barrett in the forward NTT · **INTT needs none**
Cortex-M4 · STM32F303 · ChipWhisperer · 100k profiling traces

**④ Advantage / disadvantage**
✓ **INTT 0 % overhead** · constants-only change · composes on top of masking
✗ **SPA only — no DPA/CPA protection**
✗ RNR⁺ still 25 % vulnerable at low noise · forward NTT +54–91 %
✗ protects the NTT/INTT and nothing else

**⑤ Apply in our IP?**
**No** — fixed datapath width, no spare headroom to spend
3.17 bits of entropy, **never refreshed** → averaged away by DPA
Coverage too narrow for the area cost
✓ **Do take the measurement:** `I = H[W] − H[W|X]`

**KEY:** MI 3.561 → **0.906 bits** · **INTT 42.61 → 42.61 kcycles**

### Script (8:00)

*① Summary — 1:30.* "This paper makes a claim most implementers never consider: your choice of *numeric representation* is itself a security parameter. ML-KEM computes modulo three thousand three hundred twenty-nine — about two to the eleven point seven — but stores every coefficient in a sixteen-bit machine word. Two separate leaks come out of that. First, the Hamming weight is a *deterministic function* of the stored value, so there's no uncertainty at all. Second, two's complement makes small negatives look completely different from small positives: plus five has two bits set, minus five has fourteen. Measured, signed storage leaks three point five six bits per coefficient against two point seven six unsigned — and that gap alone is about two hundred six bits across a polynomial, free to eliminate.

The threat is a soft-analytical side-channel attack: encode the whole algorithm as a factor graph, attach a leakage distribution to every profiled intermediate, run belief propagation. The NTT is the ideal target because every coefficient passes through seven butterfly layers, giving seven views of it. Key recovery from a *single trace*, and it works against masked implementations too because shares fall divide-and-conquer."

*② CM procedure — 1:45.* "Here's the conceptual core, and it's worth going slowly. Mutual information is the entropy of the Hamming weight minus the conditional entropy of the weight *given* the value. If each value has exactly one machine encoding, that second term is zero — the weight is fully determined — and therefore everything leaks. No clever bit layout fixes that, and you can't reduce the first term much either. The only lever is manufacturing conditional entropy, and that means giving each value several possible encodings. Which means changing the ring you compute in.

So: compute in Z sub eta-q instead of Z sub q. Find the largest eta that doesn't overflow the butterfly — for Kyber the bound works out just under ten, so they take nine. Encode each input as the value plus a random multiple of q. Run the entire algorithm in the enlarged ring, adding Barrett reductions where the range invariant would break. Decode at the end by reducing mod q. Leakage drops from three point five six bits to nought point nine."

*③ Implementation — 1:15.* "The implementation delta is remarkably small, and that's the selling point. You change the rejection-sampling bounds so the samplers emit coefficients in the enlarged ring, and you change the reduction constants. No new gadgets, no randomness consumed during the computation — the random multiple is drawn once at encoding.

The cost is two extra Barrett reductions per butterfly in the forward transform, needed to maintain the range invariant across layers. The inverse transform is the interesting case: a fixpoint argument shows the Montgomery output doesn't need an extra reduction, so only the constants change. Evaluation was on a Cortex-M4 with a ChipWhisperer setup and a hundred thousand profiling traces — synchronous sampling, which is deliberately favourable to the attacker, so the security claim holds under near-ideal measurement."

*④ Advantage / disadvantage — 1:45.* "The headline: forty-two point six one kilocycles against forty-two point six one. That's not rounding — zero overhead on the inverse NTT, which is the operation that touches the secret key in decryption, so it's the number that matters most. Against simulated and real attacks, the signed implementation succeeds up to noise sigma of three; the signed RNR variant is ineffective at *every* noise level, with signal-to-noise down two orders of magnitude.

The disadvantages are equally clear and the paper is honest about them. It's an SPA countermeasure — the paper's own discussion says masking is still required for differential attacks. The unsigned variant isn't actually secure, succeeding twenty-five percent of the time at low noise. The forward NTT costs fifty-four to ninety-one percent. And it protects the NTT and inverse NTT and nothing else — not the message decode, not the FO transform, not the comparison."

*⑤ Apply in our IP — 1:45.* "No, and for three reasons. First, our datapath width is fixed and sized to the minimum. The entire benefit comes from spare headroom between q and the word size — for Kyber that's twelve bits of data in a sixteen-bit word. But that headroom is a software accident of the machine ABI. In an ASIC you don't get sixteen-bit words for free; you size the datapath and freeze it, and widening it means redoing the multiplier, whose area goes roughly quadratic.

Second, on randomness. It does draw a random offset — but only about three point two bits of entropy per coefficient, against roughly eleven point seven for a proper first-order arithmetic mask. And because the NTT is linear, that same offset propagates through all seven layers with no re-randomization. That's the signature of a low-entropy masking scheme that never refreshes: fine against a single trace, averaged away by multi-trace analysis. Our threat model has to include DPA, so this can't be the protection — at best it sits on top of masking we'd have to build anyway.

Third, coverage. We'd pay multiplier area for one block while still needing masking on that same block. Poor trade. What *is* worth taking is the measurement rather than the countermeasure: computing that mutual information for your own representation and register width is a few lines of code, needs no traces and no silicon, and tells you whether your representation leaks before the datapath is frozen. And note their result runs the opposite way for us — narrowing the datapath to exactly the bits you need minimizes representation leakage for free."

---
---

# Timing

| Slide | Time | Cumulative |
|---|---|---|
| 1 · Kyber intro | 5:00 | 5:00 |
| 2 · Generic SCA (attack) | 8:00 | 13:00 |
| 3 · Defeating LCC (attack) | 8:00 | 21:00 |
| 4 · PQ-Hammer (attack) | 8:00 | 29:00 |
| 5 · HW shuffling (CM) | 8:00 | 37:00 |
| 6 · Low-cost shuffling (CM) | 8:00 | 45:00 |
| 7 · RNR (CM) | 8:00 | 53:00 |
| Q&A | 7:00 | **60:00** |

**Cluster timing within each paper (attack):** ① 1:15 · ② 1:15–1:30 · ③ 2:15 · ④ 1:30 · ⑤ 1:30–1:45

**Cluster timing within each paper (countermeasure):** ① 1:15–1:30 · ② 1:45–2:00 · ③ 1:15–1:30 · ④ 1:45 · ⑤ 1:30–1:45

**Delivery note.** With one slide per paper the whole thing is visible from the first second, so the audience reads ahead. Either build the five clusters with sequential appear animations, or accept it and use the visible structure as a map — "we'll come back to the bottom-right." The second is simpler and fine for a technical audience.

**If you overrun,** compress cluster ③ on papers 2 and 4 (procedure and implementation detail), and PQ-Hammer's stealth aside. Each is worth about a minute and none is load-bearing.
