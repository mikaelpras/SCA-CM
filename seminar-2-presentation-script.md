# Presentation Script — Seminar 2
**7 slides · ~50 minutes speaking + 10 minutes Q&A**

Written to be spoken aloud. Sentences are kept short on purpose — they are easier to deliver and easier to follow.
`[BRACKETS]` = stage direction, not spoken. **Bold** = land this phrase; slow down slightly.
Speaking rate assumed ≈ 140 words per minute. Times are cumulative.

---

# SLIDE 1 — OVERVIEW
**Target: 5 minutes · ends at 0:05**

Good morning, everyone. Thank you for coming.

Today I am reviewing six papers on side-channel attacks and countermeasures for post-quantum cryptography. All six are about lattice-based schemes — mostly Kyber, which is now standardised as ML-KEM, and some Dilithium, now ML-DSA.

Let me tell you the shape of the talk first, so you know where we are going.

The first three papers are **attacks**. They ask: how do these schemes actually break on real hardware?

The last three are **defences**. They ask a different question, and I think a more uncomfortable one: what does it actually cost to stop these attacks, and do the defences work?

`[PAUSE briefly, then walk the table]`

Paper one is from Queen's University Belfast, last year. It is a correlation power analysis attack on ML-KEM. What I like about this paper is that the contribution is not a new statistical method. The contribution is that they read the assembly code carefully. One instruction stores a value it should not store. From that single observation they recover the full key in **179 traces**.

Paper two is from NTU Singapore and KU Leuven. This is a chosen-ciphertext attack. The existing attacks in this family were spending thousands of leaky measurement points just to learn **one bit** of information. This paper says: that is wasteful. They parallelise it and get **seven and a half times** fewer queries.

Paper three is also from NTU, with TU Graz. This one is about NTRU, not Kyber. I will explain in detail why it belongs in this seminar. The short version: NTRU Prime was specifically designed to have a smaller attack surface than Kyber. This paper shows it falls to the same attack anyway. That tells us something important about **where** the vulnerability really lives.

Paper four is from 2020, also NTU. This is the first paper that actually measured what protecting the NTT costs. The number for Kyber is manageable. The number for Dilithium is not.

Paper five is the newest — a preprint from this April. It audits **Adams Bridge**, which is a real hardware accelerator, shipping today, backed by AMD, Google, Microsoft and NVIDIA. I think this is the most interesting paper in the set for anyone doing hardware.

Paper six is from KTH and Ericsson. A completely different style of defence — two cores running on randomised clocks. And it has an unusual history that I will come back to at the end.

`[PAUSE]`

One theme you will notice. In the attack papers, the defenders keep losing because of **implementation details**. In the defence papers, the defenders keep losing because they **evaluated cost but not security**. Keep that in mind as we go.

Let us start with the first paper.

---

# SLIDE 2 — Enhanced Two-Step CPA on ML-KEM
**Target: 7 minutes · ends at 0:12**

`[Section ①]`

This paper is from Kennaway and colleagues at Queen's University Belfast, published at SECRYPT last year.

It is a correlation power analysis attack. CPA is an old, well-understood technique — it goes back to 2004. You guess part of the secret, you predict the power consumption using a simple model, and you correlate that prediction against the measured traces. The correct guess correlates; the wrong guesses do not.

So the method here is not new. What is new is **where** they point it.

Their result: full recovery of a Kyber512 secret key in **179 traces**. And importantly — no chosen ciphertexts, no profiling phase, no clone device, no special preconditions. This is about as weak an attacker as you can have while still being able to measure power.

`[Section ②]`

Now, where does the attack point?

Kyber decapsulation computes the message from `v` minus the inverse NTT of `s` times `u`. The secret key `s` meets the attacker-known ciphertext `u` in that multiplication. That is the natural place to look, and many papers have looked there.

In the pqm4 implementation — which is the standard optimised ARM Cortex-M4 implementation — that multiplication ends up in a function called `doublebasemul_asm`. Inside it, each basic multiplication performs **five** modular multiplications.

`[Slow down here — this is the core of the paper]`

Here is what the authors noticed. Of those five multiplications, **four of them are fused into other operations**. The `SMLABB` instruction is a multiply-**accumulate** — it multiplies and adds in one step, so the product never exists on its own. `SMUADX` is a **dual** multiply — it computes two products and sums them inside a single instruction. Again, neither product exists alone.

But one instruction is different. `SMULTT` computes a single product of one secret coefficient and one ciphertext coefficient. Then Montgomery reduction is applied. And the result **sits alone in a register** — register `tmp2` — for at least one clock cycle.

That is the whole vulnerability. One value, one unknown, sitting by itself in a register.

`[PAUSE]`

Why does this matter so much? Because of the size of the guess. If a value depends on **one** secret coefficient, you have 3329 possibilities — that is `q`. You can enumerate that in a fraction of a second. If a value depends on **two** secret coefficients, you have `q` squared, which is about eleven million, and the leakage is also weaker because the algebra is more complicated.

The authors confirmed this experimentally. They tried correlating against the other three products. The correlation was flat everywhere. Those products genuinely do not leak on their own.

The setup was a ChipWhisperer-Lite with an STM32F3 target — a standard Cortex-M4 evaluation board. They captured 500 traces at four times the clock frequency.

`[Section ③]`

So how does the attack run? `[Walk the arrow chain]`

First they capture the traces. Then they cross-validate — they compute the same intermediate values on a laptop and check the device is producing what they expect. Then they identify points of interest by sweeping the correlation across every sample point.

Then the first CPA pass. They target the isolated product, and they recover all the **odd-indexed** coefficients of the secret key. Correlation of **0.87**. Measurements to disclosure — that is the number of traces needed before the correct guess separates from the wrong ones — as low as **ten traces**.

Ten traces. That is extremely strong.

But the even-indexed coefficients are not recoverable this way, because they only appear inside the fused instructions.

So here is the two-step part. Once you know the odd coefficients, they stop being unknowns. They become **known constants**. You can now write an attack function for the fused value where the only remaining unknown is the even coefficient. And that works — correlation of 0.32, and measurements to disclosure of 179.

That 179 is the worst case, and it is where the headline number comes from.

Then you simply replay both steps across all 512 coefficients, because the structure repeats identically.

`[Section ④]`

What does this tell us to protect?

The rule is simple and I think quite general: **never let a product of one secret value and one public value reach a storage element on its own.**

Now, here is something interesting the paper shows without emphasising it. The fused instructions — the accumulate and the dual multiply — are effectively an accidental countermeasure. They reduce the correlation from 0.87 down to 0.32. That is a factor of about **2.7**, and it costs nothing, because it is just how the code was written.

But look at the result. Step two still succeeded. 179 traces is still completely practical. So a factor of 2.7 is a **delay**, not a defence. This is the argument for real masking rather than clever structuring.

`[Optional if ahead of time — critique]`

Two small criticisms. First, the "179 traces" headline is the worst-case measurements-to-disclosure figure. They actually captured and used 500 traces throughout. Second, their comparison table puts their Kyber512 result against three Kyber768 results, which is not a fair comparison — different security levels have different key sizes.

`[TRANSITION]`
That was an attack on the arithmetic. The next paper attacks something completely different.

---

# SLIDE 3 — Parallel PC Oracle Attacks on Kyber KEM
**Target: 8 minutes · ends at 0:20**

`[Section ①]`

This is from Rajendran, Ravi, D'Anvers, Bhasin and Chattopadhyay — NTU Singapore and KU Leuven, published in TCHES 2023.

To understand this paper I need to explain what a plaintext-checking oracle attack is, because it is quite different from what we just saw.

`[Section ② — background first]`

Kyber is a CCA-secure KEM. That means it uses the Fujisaki-Okamoto transform. In decapsulation, the device decrypts the ciphertext to get a message `m`, and then it **re-encrypts** that message and checks whether the result matches the ciphertext it received. If not, it rejects.

Now here is the key observation that this whole family of attacks depends on. That re-encryption is **deterministic in the message**. Same message, same public key, same computation, every time.

So if an attacker can craft a ciphertext where the decrypted message is either zero or one, depending on one coefficient of the secret key — then measuring which computation happened tells them that coefficient.

That is the oracle. And because the message goes through hashing before re-encryption, a single bit flip changes **everything downstream**. Thousands of measurement points differ. It is very easy to distinguish.

The existing attacks did exactly this, one bit at a time. Around 1776 queries for Kyber768.

`[Section ①-② bridge — the insight]`

The authors of this paper made two observations.

First: in each query, only **one** of the 256 message bits actually depends on the secret. The other 255 are forced to zero. Wasted.

Second: you are using **thousands** of leaky points to extract **one bit** of information. That is a very poor exchange rate.

`[PAUSE]`

So the idea is: can we make **P** message bits depend on **P** different secret coefficients, in a single query? And then instead of a binary classification, do a classification among 2-to-the-P classes?

`[Section ③]`

The construction is on the slide. They set `u` to a fixed constant times x-to-the-zero, and `v` to a sum of the first P powers of x. The effect is that the first P message bits each depend on a different secret coefficient, and everything else is pinned to zero.

Two details matter here.

The first is that `k_u` is **fixed**. They found the value 208 by exhaustive search. Because it is fixed, only `v` varies between queries, and that keeps the P decision tree traversals **independent** of each other. If they were coupled, this would not work.

The second is something I found genuinely surprising. These attacks use binary decision trees to minimise queries — because Kyber's secret coefficients follow a centred binomial distribution, so some values are much more likely than others. You assign fewer queries to the more probable values.

The optimal tree for the one-bit-at-a-time attack is the **minimum entropy** tree. But the authors show that in the parallel setting, that tree is **wrong**.

Why? Because when you attack a **set** of P coefficients at once, the number of queries you need is determined by the **deepest** coefficient in that set — not the average. And as P grows, the chance that at least one of your P coefficients sits deep in the tree goes to one.

So you want the **minimum depth** tree instead. For P of three or more, it wins.

`[PAUSE]`

Then the classifier. To distinguish 2-to-the-P classes they use Welch's t-test to find points of interest, build a template for each class, and classify by sum of squared differences. They do it pairwise, in a knockout tournament structure, which needs 2-to-the-P minus one comparisons — that is optimal.

The remarkable thing is how few traces they need per class. **Five.** Because the leakage difference between any two messages is so large.

`[Results table]`

The results. Baseline is 1776 queries. With a clone device for building templates offline, at P equal to ten — that is 1024 classes — they get down to **232 queries**. That is 7.65 times better. Without a clone device, where template building also costs queries, the optimum is P equal to four, and they get 613 queries — still 2.89 times better.

They validated P equal to ten with a **100 percent success rate** on real hardware, and verified the classification works at P equal to twelve — that is 4096 classes.

`[Section ④]`

Now the finding I think is most important for defenders.

**Shuffling does nothing against this attack.**

Shuffling was proposed to protect the message encoding operation, and against some attacks it works well. But this attack never touches message encoding. The strongest leakage they found was in the **sampling of the ephemeral secret** inside re-encryption — a completely different operation.

So on a shuffled implementation, they need **exactly the same number of traces** as on an unprotected one. That is currently the lowest known query count against shuffled Kyber.

The lesson: a countermeasure protects the operation it protects. If your attacker can get their information from somewhere else in the same procedure, you have bought nothing.

`[Critique — include if on time]`

One caveat on the headline. They also quote a figure of about 24 times improvement, at P equal to 32. But P equal to 32 means four **billion** template classes. That is not an attack, it is an asymptote. The validated numbers are 7.65 and 2.89.

And their conclusion is that masking is necessary — but masking is explicitly out of scope in the paper. They never attacked a masked implementation. So that conclusion is argued from outside.

`[TRANSITION]`
So Kyber falls to this. The natural question is — would a different kind of lattice scheme do better? That is exactly the next paper.

---

# SLIDE 4 — Attacking NTRU-based KEMs: What It Tells Us About Kyber
**Target: 8 minutes · ends at 0:28**

`[Section ① — address the obvious question immediately]`

You may be wondering why an NTRU paper is in a Kyber seminar. Let me answer that first.

This paper is fundamentally a **comparative** study, and its headline conclusion is a statement about Kyber. So it is directly relevant.

Here is the context. NTRU Prime was designed deliberately to reduce attack surface. It uses a **non-cyclotomic** ring, chosen specifically to avoid the algebraic structure that lattice attacks exploit. Its message is **implicit** — there is no separate message polynomial, the message is the rounding noise. And it is **perfectly correct** — by design, it has **zero** decryption failures.

Those are three real, deliberate structural differences from Kyber.

And this paper's conclusion is that attacking it requires — their words — **"no considerable increase in attacker effort."**

`[PAUSE]`

So the takeaway is: switching away from Kyber to escape this attack class is **not a mitigation**. The vulnerability is not in LWE. The vulnerability is in the **Fujisaki-Okamoto transform**, which all of these schemes use.

`[Section ② — the mechanism]`

Now, the mechanism. This is where the title comes from, and the title is literal.

NTRU Prime decryption computes `a` equals three `f` times `c`. Then every coefficient of `a` is **centred** into the range minus q-over-two to plus q-over-two. Then it takes `a` modulo 3.

Under normal operation, all the coefficients are already inside that range, so the centring does nothing, and everything stays a clean multiple of three. The mod-3 step removes it all.

But suppose you can force **one** coefficient across the threshold. Then the centring subtracts `q`. And here is the crucial arithmetic fact: `q` is a **prime**. It is **not** a multiple of three.

So that one coefficient is no longer divisible by three. It **survives** the mod-3 step. Every other coefficient vanishes.

And whether it crosses the threshold depends on a targeted coefficient of the secret key. That is the oracle.

`[Comparison table]`

Now let me map this against Kyber, because the differences are instructive.

In Kyber, the attacker controls the **message** directly. The oracle classes are "message equals zero" and "message equals one", and those are the same regardless of the key. So you build your templates once per **device**.

In NTRU Prime, the attacker cannot set the message. The anchor is an **internal** variable. And its non-zero position depends on the **secret key**. So the attacker needs a search phase, and templates must be rebuilt for **every new secret key**.

Every row of this table says NTRU Prime is harder. And it still falls, at roughly twice the cost — about 4350 traces versus about 2560 for Kyber512.

`[Section ③]`

Briefly on the method, because the detail is less important than the comparison.

The attacker crafts ciphertexts designed to produce a "collision" — a coefficient where everything lines up at maximum magnitude. The naive version has a probability of about eight times ten to the minus **43**. Useless. So they use sums of **rotations** instead, which makes the probability tunable.

Then phase one: search for a base ciphertext. The detection signal is beautiful — if the internal variable is zero, a weight check returns **zero**. If it is non-zero, the weight is around **500**. That is a huge difference in power consumption. About 61 trials, 610 traces.

Phase two: four queries per coefficient, each one a single trace, decoded through a decision table. About 3000 traces.

Phase three: a small brute force of 1522 candidates offline.

They also build a second oracle — a decryption-failure oracle — by perturbing a **valid** ciphertext. And this is worth noting: that version costs **more** traces, about 8100. The gain is not efficiency. The gain is **attack surface**, because a decryption failure propagates into re-encryption, whereas the plaintext-checking version stays trapped inside decryption.

`[Section ④ — this is the payoff, slow down]`

So what do we learn, for Kyber?

**First: perfect correctness is not a defence.** NTRU Prime has zero natural decryption failures. The attacker simply **manufactures** them. So Kyber's low failure rate is not what is protecting it.

**Second: which oracle is cheapest depends on the scheme.** For the plaintext-checking oracle, Kyber is the cheaper target. But for the decryption-failure oracle, Kyber is far **more expensive** — two-to-the-seventeen queries for only a partial security reduction, versus about 9600 here. Do not assume the ranking carries over.

**Third — and this is the one I want you to remember.** `[PAUSE]`

On NTRU Prime, the attack ciphertexts always produce the same decryption output. So the leakage **provably cannot escape the decryption procedure**. Which means masking decryption **alone** is sufficient against that oracle.

On Kyber, the opposite is true. As we saw in the previous paper, re-encryption **is** the attack surface.

So Kyber needs **full decapsulation masking** even against the cheapest oracle. NTRU gets a shortcut that Kyber does not.

That is a result about Kyber that comes from an NTRU paper. And it is why I included it.

`[TRANSITION]`
So far, three attacks, and all three end with "you need masking." The next three papers ask what that actually costs.

---

# SLIDE 5 — Configurable SCA Countermeasures for the NTT
**Target: 7 minutes · ends at 0:35**

`[Section ①]`

This is Ravi, Poussier, Bhasin and Chattopadhyay, from SPACE 2020. It is a countermeasure paper, and it is a cost paper.

But I need to set up the threat first, because it is a different threat from everything we have seen so far.

There is a family of attacks called **soft-analytical side-channel attacks**, or SASCA. Instead of correlating one guess at a time, you build a **factor graph** of the whole computation and run belief propagation over it. Every load and store in the NTT gives you a probability distribution, and the algebra links them together. Then you let the inference run.

Primas and colleagues showed in 2017 that the NTT falls to this. And crucially — it works from a **single trace**.

`[Slow down — this is the key conceptual point]`

Here is why that matters so much. Standard masking splits a secret into two shares. But in these attacks, the secret is recovered from within a **single linear operation**. Splitting into shares does not meaningfully change the attack's efficiency.

So the usual answer — "just mask it" — **does not apply here**. That is why this paper reaches for a different toolbox.

The specific attack they target is from Hamburg and colleagues, TCHES 2021. It attacks a **masked** CCA-secure Kyber, uses a specially crafted sparse ciphertext, and recovers the **long-term** key. So this is not a theoretical worry.

`[Section ②]`

Two countermeasures are proposed.

The first is **local masking**. This is not share splitting. It is a **multiplicative** mask on the twiddle factors of the NTT. If the twiddles are masked, the intermediates flowing through are masked too.

And there is a nice efficiency trick. The twiddle factors are powers of a root of unity. So zeta-to-the-a times zeta-to-the-b is zeta-to-the-a-plus-b — which is **also** a twiddle factor, already in the precomputed table. So masking costs **no extra multiplication** at the butterfly. It is essentially free at runtime.

They also re-mask at every layer, not just at the input, which increases the entropy. And it is configurable — there is a parameter `u`, the number of masks per layer.

The authors describe it as "cheap and low-entropy masking." Remember that phrase. I will come back to it.

The second countermeasure is **shuffling**, at three different granularities. `[Walk the table]` Fine shuffling randomises the order inside each butterfly — one bit of entropy per butterfly. Coarse block shuffling permutes butterflies that share a twiddle factor. Coarse full shuffling permutes everything in a layer.

One detail worth pointing out. For coarse block shuffling on Kyber, the **first** layer has only 2-to-the-64 orderings, while the last layer has 128 factorial. So the entropy is lowest in the **early** layers — which is exactly where an attacker starts.

`[Section ③]`

Now the numbers, which are the paper's actual deliverable.

For Kyber, overheads range from **7 to 78 percent**, depending on configuration. That is manageable.

For Dilithium, key generation is 12 to 197 percent. And Dilithium **signing** is **32 to 490 percent**.

`[PAUSE]`

Why the difference? Two reasons, and neither is accidental. Dilithium performs far more NTTs per operation — the public matrix alone needs k times l transforms. And signing has a **rejection sampling loop**, which repeats the whole polynomial arithmetic an unpredictable number of times. So every per-NTT overhead gets multiplied.

The practical message: the same countermeasure is affordable for a KEM and close to prohibitive for a signature scheme.

`[Section ④ — the payoff]`

Now, what happened afterwards.

This paper measured **cost**. It did not evaluate **security** against an adapted attacker. And all three countermeasures have since been weakened.

Hermelink and colleagues, TCHES 2023, went after the shuffling. Fine shuffling: broken. They introduced something clever — a **shuffle node** in the factor graph, which **learns the permutation during the inference run**. Coarse block shuffling: also broken, for an attacker who can observe both loads and stores. Only **coarse full shuffling** — the most expensive variant — survived.

And then, last year, a paper called "Cracking the Mask" went after the local masking. Their result is the striking one.

`[Slow down]`

At **low** values of the parameter `u`, attacking the local-masked NTT is **easier** than attacking the **unmasked** NTT.

The countermeasure, cheaply configured, is **worse than doing nothing**. Because the mask nodes add structure to the factor graph that the attacker can exploit.

That is what "cheap and low-entropy" turned out to mean.

`[IP relevance]`

For hardware, two things carry over. The software cost model does not — in hardware, shuffling costs control logic and a random number generator rather than cycles, and local masking costs memory rather than time. But the Kyber-versus-Dilithium asymmetry **does** carry over, because it comes from the NTT count and the rejection loop, not from the platform.

And the design lesson: if you expose a **configurable** protection level, the **low** setting must be shown safe. Not merely cheap.

`[TRANSITION]`
That was a countermeasure evaluated in software, in 2020. The next paper looks at what happened when this general approach was built into real silicon.

---

# SLIDE 6 — Partial NTT Masking: A Security Margin Analysis
**Target: 8 minutes · ends at 0:43**

`[Section ①]`

This is a preprint from April this year, by Iskander and Kirah. I want to flag clearly at the start: it is **not peer reviewed**, and the design it criticises is actively defended by its authors. I will come back to that.

The target is **Adams Bridge**. This is not an academic prototype. It is the post-quantum accelerator for **Caliptra**, which is an open-source silicon root of trust backed by AMD, Google, Microsoft and NVIDIA. It implements ML-DSA and ML-KEM. About 1.17 million gate equivalents. And critically for this paper — the **RTL is public**, under Apache 2.0.

Now, the design decision under scrutiny. It is called **partial masking**, and the logic behind it is completely reasonable. Masking the entire NTT is expensive in silicon. So: mask the first inverse-NTT layer with proper first-order Boolean masking, mask the point-wise multiplication, and then argue that the remaining layers are safe on other grounds.

After that first masked layer, the two shares are **recombined into a single unmasked value**, and the remaining layers run unprotected.

The designers do not hide this. Their own documentation says that leakage "naturally emerges in the unmasked layers." Their argument is that it is not **exploitable**. Specifically: an attacker would have to guess two 23-bit coefficients simultaneously, giving a complexity of **two-to-the-46** — and on top of that, they apply a shuffling scheme.

So this paper is a **red team audit** of that specific claim. Seven analysis tracks, combining RTL verification with belief propagation simulation, and every finding carries a **confidence rating**.

`[Section ② — Finding 1, the strongest]`

Let me start with the finding that needs no attack model at all.

The shuffling is called **Random Start Index**. The RTL is public, so you can just read it. Lines 648 to 653 of the NTT control module.

What it does: it picks one of **16** starting positions within a 16-element chunk, and there is an additional factor of **4** from index selection. Sixteen times four is **64** orderings per layer.

Sixty-four orderings is **six bits** of entropy.

`[PAUSE]`

The security scaling argument assumed something approaching a full random permutation — around **296 bits**.

Why the gap? Because the processing is sequential **with wraparound**. That is a **rotation**, not a permutation. A rotation of sixteen elements has sixteen orderings. A permutation of sixteen elements has sixteen factorial — about two-to-the-44.

I want to be fair to the designers here. A rotation is the **sensible hardware choice** — a full permutation network is expensive. The problem is not the design decision. The problem is that the security argument was written as though they had bought the expensive one.

`[Finding 2]`

The second finding is conceptual, and it generalises well beyond this design.

The two-to-the-46 figure assumes each butterfly is attacked **independently**. Under that assumption, the number is correct.

But SASCA does not work that way. It processes the **entire factor graph simultaneously**, and it feeds specifically on the **dependencies between butterflies** — which is exactly what the independence assumption throws away. And as we saw in the previous paper, belief propagation can absorb shuffling by learning the permutation during inference.

So: **right arithmetic, wrong threat model.**

`[Finding 3 and 4]`

Experimentally, they run belief propagation over the complete inverse-NTT factor graph and achieve **100 percent coefficient recovery**. That resolves a genuine open question — previous BP results were small-scale or simulated, and it was not clear the gains would survive at production sizes. They do.

But — and they are honest about this — the soft-analytical pipeline demonstrates a 37-bit reduction **explicitly without achieving key recovery**.

Then the finding I think is the most useful in the whole paper. `[PAUSE]`

Four **evenly spread** observed layers give **100 percent** recovery.
Four **consecutive** observed layers give **zero percent**.

Same number of layers. Completely opposite outcome.

Security is determined by **where** the gaps are, not **how many** layers you protect.

`[Caveats — say these clearly]`

Now the caveats, and I want to be straight about them. This is a preprint. There is **no key recovery on actual silicon**. The lead author works at a commercial security company. And the Adams Bridge designers have published a **rebuttal**.

Two other independent analyses do agree with the general concern — one empirical attack on silicon from FAU, and an RTL review by Saarinen finding that the key is not arithmetically share-split. So the concern is not one group's opinion. But this is a **live technical dispute**, not a settled verdict.

`[Section ③ — IP adoption]`

For anyone designing this kind of hardware, four things carry over.

**One: placement beats coverage.** Their recommendation is to mask **three consecutive middle layers**, at **43 percent** of full masking cost. That creates a gap belief propagation cannot bridge. And note — masking the **first** layer, which is what Adams Bridge does, is the **worst** placement for a fixed budget, because it leaves every remaining layer contiguous.

**Two: count your shuffler's actual orderings.** A random start index with wraparound gives you log-n bits, not log-n-factorial. Enumerate it in RTL before you put a number in a datasheet.

**Three: publish the threat model with the complexity figure.** A bare number invites exactly this kind of audit.

**Four: audit the boundary.** Where masked meets unmasked, the shares get recombined. That seam is where the whole argument fails. It is the same principle as the first paper's isolated register — just one level of abstraction higher.

`[TRANSITION]`
The last paper takes a completely different approach to the same problem. Instead of masking selected operations, it hides everything.

---

# SLIDE 7 — Duplication + Clock Randomization for Kyber on FPGA
**Target: 7 minutes · ends at 0:50**

`[Section ①]`

This is from KTH and Ericsson Research, published in IEEE Design and Test in 2024.

The idea comes from the block cipher world. Run **two** Kyber cores — a primary and a dummy — and drive each one with its **own randomised clock**.

This is a **hiding** countermeasure, not a masking one. And the pitch is aimed squarely at masking's weak points. Four claims: it covers the **whole design**, so there are no per-operation decisions. It is immune to **glitches**, which are a persistent problem for hardware masking. It adds **zero clock cycles**. And it degrades attacks that rely on averaging many measurements.

`[Section ② — the honest framing]`

What I find interesting is that each ingredient is **known broken on its own**.

Clock randomisation alone was broken in 2022 — by **these same authors**. They recovered an AES subkey in under 500 traces, by sampling much faster than the encryption clock, resynchronising the traces afterwards, and attacking the **beginning** of the encryption before the clock drift accumulates.

And duplication alone was already known to be inadequate — someone had extracted a key from a duplicated AES implementation.

So the claim of this line of work is that the two **fix each other**.

`[Architecture]`

The clock generator uses an MMCM to produce base frequencies, and then a look-up-table based four-to-one multiplexer selects between them **asynchronously**. Because the selection is asynchronous, the output has varying **frequency and varying duty cycle** — the pulse widths change too, which is part of what destroys trace alignment. They report at least **403** distinct achievable frequencies.

Now, one design decision that looks strange until you think about it. Both cores receive **identical input data**, but they use **different key pairs**.

Why different keys? Because if both cores ran the same key on the same data, their power traces would be **correlated**. The dummy would add noise, but it would not add confusion. Different keys make the dummy's contribution genuinely independent of the target secret.

`[Section ③]`

The evaluation. The baseline attack is a deep-learning power analysis that recovers a message from the unprotected implementation in **5120 traces**. And in Kyber, recovering the message means recovering the shared key, because the key is derived from the message by hashing.

They compare three builds — unprotected, clock randomisation only, and both together — on an Artix-7 FPGA. The combined version resists the deep learning attack.

The cost. `[Walk the table]` Roughly **double the area**, from duplication. Zero clock-cycle overhead — that is the headline claim, and it is true. But **not** zero wall-clock time, because a randomised clock has a lower average frequency. Roughly double the power. And the dummy core needs its own real key pair, so there is a key management burden.

For context, a competing proposal based on shuffling claims **8.7 percent** overhead, and explicitly criticises this approach as high cost.

`[Section ④ — the story]`

Now the part I have been waiting to tell you. `[PAUSE]`

Look at the sequence. In **2022**, this group **breaks** clock randomisation. In **2023**, they **fix** it by adding duplication. In 2023 and 2024, they **apply** that fix to Kyber — this paper. And then in **2024**, they **break it again**.

The new attack exploits something they call **sporadic synchronicity**. The two randomised clocks are independent — but occasionally, by chance, they drift into momentary alignment. In those windows, the trace looks like a single-core trace. And a deep learning attacker who can **detect** those windows gets clean data.

`[Slow down]`

**Randomness does not guarantee non-coincidence.**

I think that is the most transferable lesson here. If your protection depends on two random processes staying out of phase, the attacker's job is to find the moments when they are not. And with enough traces, those moments exist.

They proposed three fixes. Those were demonstrated on AES. Whether the Kyber design in this paper is affected, and whether the fixes have been applied to it — I could not establish. That is an honest open question.

`[One more critique]`

One other thing worth questioning. This paper is framed as protecting against **fault** attacks as well as side-channel attacks. But duplication detects faults by **comparing two identical computations**. Here the two cores run **different keys**. So their outputs cannot be compared. I would want to see how that claim is justified.

`[IP relevance and close]`

For hardware IP, the honest summary is a trade-off with no free option.

Universal coverage is a **genuine** architectural advantage — there is no masked-versus-unmasked boundary to audit, and we just saw in the previous paper that the boundary is exactly where those arguments fail.

But double the area is much harder to sell in an ASIC IP block than on an FPGA prototype. And a randomised clock domain is a **timing closure problem you hand to your customer**, which is a real obstacle for licensable IP.

`[CLOSING — 30 seconds]`

Let me close with what I take from all six papers together.

The attack papers keep finding the same thing: the vulnerability is almost never in the mathematics. It is in an instruction that stores a value it should not, or a procedure that is deterministic when it should not be.

And the defence papers keep finding something else: it is much easier to measure what a countermeasure **costs** than to establish what it **buys**. Every defence in the second half of this talk was published with a cost number and later weakened by an adapted attacker.

Thank you. I am happy to take questions.

---

# Q&A PREPARATION
**Reserve 10 minutes · ends at 1:00**

**"Is Kyber broken?"**
No. Every attack here needs physical access to the device and a static key. These are implementation attacks, not attacks on the underlying mathematics. What they change is what a *hardware* implementation has to do to be safe.

**"Which countermeasure would you actually use?"**
No single one. Masking carries the security argument, but only if the protected boundary covers the whole decapsulation — paper 4 shows why. Shuffling helps, but only at high entropy, and paper 2 shows it can be bypassed entirely. Partial anything needs its boundary audited.

**"Why is Dilithium so much more expensive to protect?"**
More NTTs per operation — the public matrix alone needs k times l transforms — plus a rejection sampling loop that repeats the whole polynomial body an unpredictable number of times, multiplying every overhead.

**"Is the Adams Bridge criticism fair?"**
Partly settled, partly not. The shuffling entropy number is a direct reading of public RTL, and I would treat it as fact. The exploitability claims rest on simulation, there is no key recovery on silicon, and the designers dispute the conclusions. Two other independent groups raised similar concerns.

**"Does any of this apply to hardware, or only to Cortex-M4?"**
Mixed. Paper 1 is entirely software-specific — it is an instruction scheduling artefact. Papers 2 and 3 are algorithmic and port directly. Papers 5 and 6 are hardware to begin with.

**"How practical is 179 traces / 232 queries in the field?"**
Both assume physical access and a key that is reused for at least that many operations. The mitigation both papers point to is the same one: refresh keys more often than the attack cost.

**If asked something you don't know:** *"That's not covered in the paper — I'd have to check the artifacts."* Better than guessing.

---

## Timing Summary

| Slide | Content | Duration | Ends |
|---|---|---|---|
| 1 | Overview | 5 min | 0:05 |
| 2 | Two-Step CPA | 7 min | 0:12 |
| 3 | Parallel PC Oracle | 8 min | 0:20 |
| 4 | NTRU / Lessons for Kyber | 8 min | 0:28 |
| 5 | NTT Countermeasures | 7 min | 0:35 |
| 6 | Adams Bridge Audit | 8 min | 0:43 |
| 7 | Duplication + Clock Rand. | 7 min | 0:50 |
| — | Q&A | 10 min | 1:00 |

**If you are running late:** cut the bracketed critique paragraphs in slides 2 and 3 first — about 90 seconds. Then compress slide 5's method section. **Do not** cut the section ④ payoffs; they are what people remember.
