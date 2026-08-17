# Deep Dive: On Configurable SCA Countermeasures Against Single Trace Attacks for the NTT — A Performance Evaluation Study over Kyber and Dilithium on the ARM Cortex-M4

**Authors:** Prasanna Ravi, Romain Poussier, Shivam Bhasin, Anupam Chattopadhyay
**Affiliation:** Temasek Laboratories & SCSE, Nanyang Technological University, Singapore
**Venue:** SPACE 2020 (10th Int'l Conf. on Security, Privacy, and Applied Cryptography Engineering), Kolkata — LNCS 12586, pp. 123–146
**DOI:** 10.1007/978-3-030-66626-2_7 · ePrint 2020/1038
**Code:** github.com/PRASANNA-RAVI/SCA_protected_Kyber
**Cited as [RPBC20]** throughout the follow-up literature.

> **Sourcing note.** ePrint is IP-blocking automated fetches, so the primary PDF was not retrievable. This deep dive is assembled from: the paper's abstract (all performance figures), **Hermelink et al. TCHES 2023 §3**, which restates the three shuffling countermeasures precisely with their permutation entropies, the **authors' own GitHub README** (local masking mechanism), and three follow-up papers that analyse the countermeasures' actual security. §9 flags what to verify from the PDF.

---

## 1. Summary — What the Paper Is About

**This is a countermeasure and cost paper, not an attack paper.** It's the odd one out in this seminar and the template shifts accordingly: the question is not "how do you break it" but "what does defending actually cost, and does the defence work."

The setup: Primas et al. (CHES 2017) and Pessl & Primas (LATINCRYPT 2019) showed the NTT falls to **single-trace** soft-analytical side-channel attacks (SASCA) using belief propagation on load/store leakage. Pessl & Primas proposed shuffling as the only concrete defence but **never measured what it costs**. Nobody had studied randomisation-based alternatives at all.

The paper contributes:

1. **Local masking** — a *multiplicative* mask on the NTT twiddle constants. Not share-splitting. Exploits the arithmetic closure of the twiddle set so masked twiddles can be **precomputed**, making it cheap. Configurable via the number of masks per layer.
2. **Three shuffling variants** at different granularities — fine, coarse full, coarse block — with explicitly different entropy/cost trade-offs.
3. **The first real cost measurement** on pqm4 / ARM Cortex-M4 for both Kyber and Dilithium.

**The headline number, and the reason this paper matters:** overheads are **7%–78% for Kyber**, but **12%–197% for Dilithium keygen and 32%–490% for Dilithium signing**. Protecting the NTT is affordable for a KEM and brutal for a signature scheme.

---

## 2. The Threat Model — Why Standard Masking Doesn't Help

This is the single most important thing to understand before the countermeasures make sense.

| Attack | Venue | What it does |
|---|---|---|
| Primas, Pessl & Mangard | CHES 2017 | Single-trace SASCA on masked lattice encryption via the NTT; needs low noise and very many templates |
| Pessl & Primas | LATINCRYPT 2019 | More practical single-trace NTT attacks; targets ephemeral secrets |
| Hamburg et al. | TCHES 2021 | Chosen-ciphertext *k*-trace attack on **masked CCA2 Kyber**; sparse INTT input via a BKZ-crafted compressible ciphertext (~25% non-zero coefficients); recovers the **long-term** key |

> **The critical point:** these attacks are **unaffected by standard masking.** Because the secret is recovered from within a *single linear NTT operation*, splitting the NTT computation into two shares does not meaningfully change attack efficiency. Boolean/arithmetic masking of the secret — the default answer everywhere else in this literature — simply does not apply here.

That's why the countermeasures in this paper are *hiding* countermeasures (shuffling) and a *non-standard* multiplicative masking, rather than conventional share-based masking.

**Contrast worth drawing in the seminar:** the CPA and PC-oracle papers in this set are all defeated *in principle* by masking, and the debate is about cost. Here masking in the usual sense is not even on the table.

---

## 3. Countermeasure 1 — Local Masking

### Mechanism
- **Multiplicative masking of the twiddle factors** used in each butterfly. Masking the twiddles ensures the intermediates flowing through the NTT are also masked.
- Formally, over the twiddle set `W = {ζⁱ mod q | i ∈ [0, n−1]}`, the masking is a map `m : Z_q × W → Z_q`.
- **The efficiency trick:** the products of twiddle powers are **precomputed**. Because `ζᵃ · ζᵇ = ζᵃ⁺ᵇ` is itself a twiddle power, the masked twiddle is already in the table — so there is **no extra multiplication at butterfly time**. This is what makes the countermeasure cheap.
- **Low-entropy by design.** The authors describe it as "cheap and low-entropy masking." That phrasing is doing a lot of work — see §8.

### Relation to prior work
Close to **Saarinen's blinding** (J. Cryptogr. Eng. 2017). The augmentation here: Ravi et al. **change the mask at every layer** of the NTT, not just at the input, increasing entropy.

### Configurability
- Parameter **`u` = number of masks per layer** — the security/performance dial.
- The released code implements **four protection levels** for NTT and INTT, and the level can be set **independently per invocation** (e.g. `polyvec_invntt(&bp, 3)` selects level 3).
- Follow-up work classifies the resulting butterflies as **MSISO / MSIDO / MDISO / MDIDO** (mask same/different in/out), depending on whether inputs and outputs share masks. Unmasking and remasking is required whenever input masks differ.

### The catch nobody flagged at the time
**Local masking is not constant time.** Butterfly complexity depends on `u` and on the butterfly type, so runtime varies with the masking configuration. Identified as a vulnerability by later hardware work.

---

## 4. Countermeasure 2 — Three Shuffling Variants

### Fine shuffling
Randomises the **order of input loads and output stores within each butterfly**.

- Implementation: **arithmetic conditional swap** (Hutter–Schwabe technique) — a random bit decides load order, then a conditional swap via bitwise operations. Deliberately avoids secret branch conditions and secret lookups.
- Entropy: 1 bit per butterfly.
- **Weak points the authors themselves flag:** (a) attacking the multiplication step inside the butterfly, as Primas et al. do; (b) leakage of the swap mask itself, analogous to `cmov` side-channel attacks on ECC.

### Coarse full shuffling
Permutes the **execution order of all butterflies within a layer**. Coefficient pairs belonging to one butterfly still move together.

- Entropy: **(n/2)! permutations per layer.**

### Coarse shuffling in blocks
Butterflies sharing the same twiddle factor ζ form a **block**. Shuffling is restricted to within blocks.

- Entropy for a layer with *m* blocks: **((n/2m)!)ᵐ**
- **Note the inversion across layers** — for the Kyber INTT (n = 256): first layer has m = 64 blocks → only **2⁶⁴**; last layer has 1 block → **128!**. Early layers are the weak ones, and early layers are exactly where the attacker starts.

| Variant | Granularity | Entropy (per layer) | Cost |
|---|---|---|---|
| Fine | within a butterfly | 1 bit / butterfly | cheapest |
| Coarse block | within a ζ-block | ((n/2m)!)ᵐ | middle |
| Coarse full | across a whole layer | (n/2)! | most expensive |

---

## 5. Performance Results (the paper's actual deliverable)

**Platform:** pqm4 implementations of Kyber and Dilithium, ARM Cortex-M4.

| Scheme / procedure | Overhead range |
|---|---|
| **Kyber** — all procedures | **7% – 78%** |
| **Dilithium** — key generation | **12% – 197%** |
| **Dilithium** — signing | **32% – 490%** |
| NTT operation alone (shuffling) | 181% – 356% vs unprotected NTT |

### Why Dilithium is so much worse — worth a slide
The asymmetry is not incidental:
- Dilithium performs **far more NTTs per operation** — the public matrix **A** alone requires k×l transforms.
- Signing has a **rejection-sampling loop** that repeats the whole polynomial-arithmetic body an unpredictable number of times, multiplying any per-NTT overhead.
- Kyber's NTT count per operation is small and fixed by comparison.

**Interpretation:** the same countermeasure is a rounding error for a KEM and a redesign-forcing cost for a signature scheme. If your product is an ML-DSA implementation, this is your number, not the Kyber one.

---

## 6. Parts Needing Protection

1. **Load and store operations** in the NTT — the actual leakage source SASCA exploits (verified experimentally by Pessl & Primas)
2. **The twiddle multiplication** inside each butterfly
3. **The INTT in decryption/decapsulation specifically** — where the long-term secret meets attacker-influenced data (Hamburg et al.'s target)
4. **The first NTT layers** — lowest shuffling entropy under coarse block shuffling, and the attacker's entry point
5. **The conditional-swap mask** in fine shuffling — the countermeasure's own randomness becomes a target

---

## 7. What Happened Next — The Security Analyses

The paper measured cost but did **not** evaluate security against adapted attackers. Three follow-ups did, and the results are the most interesting material in this deep dive.

### 7.1 Hermelink, Streit, Strieder & Thieme — TCHES 2023, "Adapting Belief Propagation to Counter Shuffling of NTTs"

The definitive analysis of the shuffling variants.

**Fine shuffling — defeated.** Without adaptation, their attack failed at every noise level, so it *does* stop unadapted attacks. But two adaptations break it:
- **Mixing priors** — point-wise addition of the two potentially-swapped prior distributions. No BP modification needed, trivial to implement. Succeeds to **σ = 0.8**.
- **Shuffle node** — a new *stateful* factor node that **learns the permutation during the BP run**, updating a shuffle factor ω via Kullback–Leibler divergence between messages and priors. Succeeds to **σ = 1.0**. (To their knowledge, the first such factor node in SASCA.)

**Coarse block shuffling — defeated for a resourceful attacker.** Two-point matching (matching load/store distributions across layers, Sinkhorn–Knopp normalisation to a doubly stochastic mix matrix) works to **σ = 0.4**. Requires an **extended attacker model**: load *and* store leakage at every layer, not just Hamburg et al.'s subset. Un-shuffling the first layer needs up to 2¹⁶ BP runs, reducible to ~2⁸.

**Coarse full shuffling — not broken by their techniques alone.** Their matching relies on layer-permutation restrictions that coarse full shuffling removes. Breaking it needs additional information (assigning twiddle factors to measured butterflies), which they suggest deep learning might supply.

**Their verdict, quoted in spirit:** the countermeasures are *powerful* and do counter Hamburg et al. as published — **but could lead to a false security perception.** A resourceful adversary still gets through.

**They also expect local masking to counter their attack** when no further adaptations are made. Which sets up:

### 7.2 Carrera Rodriguez, Valea, Bruguier & Benoit — ePrint 2024/1194 (hardware)
Hardware implementation of the local-masked NTT; performance, area, and non-specific t-tests. Finding: **some local masking configurations leak more than others.** Plus the constant-time problem noted in §3.

### 7.3 Carrera Rodriguez, Bruguier, Valea & Benoit — IACR CiC 2025, "Cracking the Mask"
The sharpest result. They adapt SASCA to local-masked NTT by **adding masking nodes to the factor graph**.

> **At low values of `u`, the attack on the local-masked NTT *outperforms* the attack on the unmasked NTT.**

The countermeasure, cheaply configured, makes things **worse than doing nothing** — the mask nodes add exploitable structure to the graph. Security does increase gradually with `u`, but at significant performance cost, and they explicitly **question the practicality of local masking** relative to shuffling.

---

## 8. Critical Points for Discussion

- **The paper evaluates cost, not security.** That's a legitimate scope, and the title says "performance evaluation study" — but the countermeasures were adopted, cited, and implemented widely on the strength of a paper that never demonstrated they work against an adapted attacker. Every one of the three has since been weakened or broken.
- **"Cheap and low-entropy masking" was the tell.** Low entropy is exactly what "Cracking the Mask" exploits. The precomputation trick that makes local masking fast is also what constrains the mask space.
- **Local masking at low `u` is worse than no countermeasure.** This is the finding to lead with in discussion. A countermeasure with a security parameter that inverts at low settings is a dangerous default.
- **Coarse full shuffling survives best** — the most expensive variant is the only one still standing. Consistent with the general lesson that hiding countermeasures buy security roughly in proportion to the entropy you pay for.
- **The configurability framing cuts both ways.** "Configurable SCA resistance" sounds like a feature; in practice it means implementers pick a level with no security-per-level guidance, and the paper provides none.
- **Non-constant-time behaviour** of local masking was not identified until the 2024 hardware work — a countermeasure that introduces a timing channel.
- **The Dilithium numbers may understate current reality.** These are Round-3 Dilithium on pqm4 in 2020; ML-DSA (FIPS 204) and modern pqm4 differ.

---

## 9. What to Verify From the PDF

- [ ] The exact algebraic definition of the local masking map and the precomputed-table structure
- [ ] The four protection levels — what distinguishes them, and the per-level overhead breakdown
- [ ] Whether the paper gives **any** security argument or leakage evaluation (t-test, SASCA simulation) for the proposed levels, or purely performance
- [ ] The full per-procedure overhead table — which specific configuration produces the 7% floor and the 490% ceiling
- [ ] Whether combined masking + shuffling configurations are measured, and at what cost
- [ ] Their stated rationale for the twiddle-set choice of mask values

**Access:** SPACE 2020 proceedings (Springer LNCS 12586), ePrint 2020/1038, or the SCA_protected_Kyber repo — note the README says countermeasures are ported to Kyber's **first-round** submission.

---

## 10. Transfer to Hardware IP

This is the most directly relevant paper in the set to your work, for three reasons.

**1. The Dilithium overhead numbers are your problem, not Kyber's.**
32%–490% on signing, in software. In hardware the trade-off inverts in useful ways: shuffling costs **control logic and an RNG**, not cycles, if you have parallel butterfly units — a permutation network and a randomised address generator rather than serialised extra work. Local masking costs **memory** (precomputed masked twiddle tables) rather than time. The software cost model in this paper does not transfer, but the *relative* Kyber/Dilithium asymmetry does, because it comes from NTT count and the rejection loop, not the platform.

**2. Standard masking does not address this threat class.**
Worth being precise about in any security claim: a masked polynomial multiplier answers CPA-class attacks (the Kennaway paper). It does **not** answer single-trace SASCA on the NTT, because the secret is recovered inside one linear operation and share-splitting doesn't disrupt that. These are orthogonal threat models needing separate arguments. An IP datasheet claiming "first-order masked" without saying which threat model is making an incomplete claim.

**3. Avoiding the NTT sidesteps this attack surface entirely — for the operations that avoid it.**
A sparse polynomial multiplier that computes `c·s₁` directly in the coefficient domain has no butterfly network, no twiddle constants, and no factor graph for BP to run on. That's a genuine structural advantage against this specific attack class, and it's worth stating explicitly in a patent or security argument. The caveat: ML-DSA still uses the NTT elsewhere (matrix expansion, other products), so the surface shrinks rather than disappears.

**4. The cautionary lesson for any configurable countermeasure you design.**
"Cracking the Mask" shows a security parameter that, at low settings, makes the implementation *more* attackable than the unprotected baseline. If your IP exposes a configurable protection level, the low setting must be shown safe — not merely cheap. Configurability without per-level security evidence is how this paper's countermeasure ended up widely deployed and subsequently broken.

---

## 11. Key References to Follow Up

- Primas, Pessl & Mangard — *Single-Trace Side-Channel Attacks on Masked Lattice-Based Encryption*, CHES 2017 (the original threat)
- Pessl & Primas — *More Practical Single-Trace Attacks on the NTT*, LATINCRYPT 2019
- Hamburg, Hermelink, Primas, Samardjiska, Schamberger, Streit, Strieder & van Vredendaal — *Chosen Ciphertext k-Trace Attacks on Masked CCA2 Secure Kyber*, TCHES 2021 (the attack the countermeasures target)
- **Hermelink, Streit, Strieder & Thieme — *Adapting Belief Propagation to Counter Shuffling of NTTs*, TCHES 2023** (the shuffling security analysis — read this one)
- **Carrera Rodriguez, Bruguier, Valea & Benoit — *Cracking the Mask: SASCA Against Local-Masked NTT for CRYSTALS-Kyber*, IACR CiC 2025** (the local masking break)
- Carrera Rodriguez, Valea, Bruguier & Benoit — *Hardware Implementation and Security Analysis of Local-Masked NTT for CRYSTALS-Kyber*, ePrint 2024/1194 (the hardware angle)
- Saarinen — *Arithmetic Coding and Blinding Countermeasures for Lattice Signatures*, JCE 2017 (local masking's ancestor)
- Veyrat-Charvillon, Gérard & Standaert — *Soft Analytical Side-Channel Attacks*, ASIACRYPT 2014 (SASCA itself)
- Veyrat-Charvillon, Medwed, Kerckhof & Standaert — *Shuffling Against Side-Channel Attacks: A Comprehensive Study with Cautionary Note*, ASIACRYPT 2012 (the general warning, which this episode confirms)
