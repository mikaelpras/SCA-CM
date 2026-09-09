# Problems with Background/Conventional Technology and the Purpose of the Invention

**Invention:** Masked minimal-ring sparse polynomial multiplication for ML-DSA (FIPS 204) signature generation

*Bracketed items marked [ ] are placeholders for your own measured figures.*

---

## 1. Background of the Invention

ML-DSA, standardized by NIST as FIPS 204 and mandated by CNSA 2.0 for national security systems, generates signatures by computing products of a challenge polynomial c with the secret polynomials s₁, s₂ and t₀. The challenge c has a distinctive structure: of its n = 256 coefficients, only τ are non-zero, and each non-zero coefficient is ±1. The polynomial is therefore sparse and ternary.

Conventional unprotected implementations exploit this structure directly. Rather than computing the products via the number-theoretic transform (NTT), the challenge is decoded into a compact table of (index, sign) pairs, and the product is evaluated as a sequence of indexed additions and subtractions. No wide modular multiplier is required. Published implementations of this approach report reductions in multiplication bit complexity exceeding 85% and in secret-key storage exceeding 68% relative to NTT-based designs, along with significant latency improvements on both embedded processors and dedicated hardware.

Independently of performance, ML-DSA signing is known to be vulnerable to side-channel analysis. The challenge–secret product is among the most sensitive operations in the signing procedure: soft-analytical side-channel attacks targeting this multiplication recover the long-term secret key. The exposure is aggravated by the fact that the challenge polynomial is public even for signature candidates that are subsequently rejected, so the attacker obtains known-input leakage on the secret at every loop iteration, whether or not a signature is emitted.

Masking is the accepted provable countermeasure. Each sensitive value is split into d arithmetic shares whose sum modulo the working modulus recovers the value, and the computation is restructured so that no set of fewer than d intermediate values reveals the secret. Security is argued in the ISW t-probing model.

---

## 2. Problems with Conventional Technology

### 2.1 Masking and sparse multiplication are mutually exclusive in conventional designs

The two lines of work described above cannot be combined, and this incompatibility is the central problem addressed by the invention.

Arithmetic masking over Z_q distributes each secret coefficient uniformly across the full range of the modulus q = 2²³ − 2¹³ + 1. The small-norm property of the secret and the sparsity of the challenge, which together made the unprotected implementation inexpensive, confer no benefit once the operands are shared: each share is a full-width, uniformly distributed 23-bit value. A masked implementation must therefore instantiate full-width modular multiplication hardware and replicate the associated storage per share, discarding every area, storage and latency advantage that sparse multiplication provides.

The cost of protection is consequently paid twice — once for the shares themselves, and again for abandoning the structural optimization that the algorithm's own parameters make available.

### 2.2 Existing approaches address the problem only partially

Prior attempts to recover this cost are each incomplete with respect to the requirements of this invention:

- **Techniques that retain the small-secret property under masking** operate within Z_q. They reduce the required multiplier width relative to naive masking, but a modular multiplication unit remains present in the protected datapath, and the arithmetic remains tied to the full 23-bit modulus.
- **Scheme variants that replace the prime modulus with a power of two** to make masking substantially cheaper require altering the signature scheme itself. Such variants are not compliant with standard ML-DSA parameters and cannot be used where FIPS 204 conformance is required — which is precisely the regulated market this invention targets.
- **Generic modulus-switching gadgets** permit an arithmetic sharing to be moved between moduli, but introduce a bounded additive error that must be accounted for in the correctness argument, and impose their own conversion cost and randomness consumption at every switch.
- **Masking-friendly signature schemes designed from scratch** address the problem at the scheme level rather than the implementation level and are not applicable where the deployed algorithm is fixed by standard or mandate.

### 2.3 Low-order masking is insufficient under the deployment conditions of interest

First- and second-order masking does not meet the security requirements of the target applications. Published attacks recover the ML-DSA secret key from second- and third-order masked implementations using on the order of 10³ traces. The scheme is structurally favourable to the attacker in this respect: each signing operation processes 256 coefficients per polynomial, yielding many exploitable samples per acquired trace, and soft-analytical attacks on the challenge–secret product further combine leakage across the multiplication's intermediate values.

High masking order therefore becomes a requirement, rather than a design preference, whenever any of the following conditions hold:

1. **Sustained adversarial physical possession.** Secure elements, smart cards, hardware wallets, smart meters and captured field equipment permit effectively unlimited trace acquisition against a fixed key.
2. **High signature volume over a long service life.** Per-signature leakage accumulates against a static long-term secret. Automotive, satellite and industrial deployments with 10–20 year lifetimes are the governing case.
3. **Certification mandate.** Common Criteria AVA_VAN.5 resistance for EAL5+/EAL6+, and FIPS 140-3 Level 3/4 physical security, require demonstrated resistance to high-order analysis.
4. **Low-noise platforms.** In a dedicated hardware signing core, the algorithmic noise that masking relies upon to amplify the attacker's estimation difficulty is weak, and additional shares must compensate for it.

It follows that the cost of *high-order* protection, not the cost of first-order protection, governs whether a design is feasible for these applications.

### 2.4 The cost of high-order masking with conventional NTT-based arithmetic

Because arithmetic masking is linear, the challenge–secret product may be computed share-wise: d forward NTTs, d pointwise multiplications and d inverse NTTs. The cost therefore scales linearly in the masking order d. The difficulty lies not in this scaling but in the constant factor per share, which is fixed by the full 23-bit modulus, and in the storage term, which grows as the product of order and coefficient width.

| Metric | Conventional Z_q, NTT-based, order d | This invention, order d |
|---|---|---|
| Multiplier in masked datapath | 23 × 23-bit modular multiplier, replicated per share | none — indexed addition/subtraction only |
| Multiplication work | d × (2 NTT + 1 INTT + n pointwise mults) | d × τ indexed accumulations over n coefficients |
| Coefficient share storage | d × n × 24 bit per polynomial | d × n × ⌈log₂ M⌉ bit per polynomial |
| Secret-key share storage (s₁, s₂, t₀) | full-width shares | [figure] |
| Twiddle-factor ROM | required, per parallel NTT unit | not required |
| Randomness for the multiplication | share refresh over Z_q | share refresh over Z_M |
| Measured area at order d = [ ] | [baseline] | [figure] |

The storage term deserves particular emphasis. Because share registers scale as d × ⌈log₂ q⌉ × n per polynomial, at order d = 4 and above the shares of s₁, s₂ and t₀ come to dominate the area of the signing core. Every bit of that width is a consequence of having masked in Z_q, not a requirement of the ML-DSA algorithm itself: the underlying secret coefficients lie in a range of at most a few bits. Narrowing the working ring from 23 bits to ⌈log₂ M⌉ bits scales this entire term by that ratio, and does so at every masking order simultaneously — the benefit compounds precisely where the conventional approach is most constrained.

A related constraint arises in randomness generation. Randomness consumption across a fully masked ML-DSA signing operation grows steeply with order, and in area- and power-constrained secure hardware the throughput of the on-chip TRNG, rather than combinational logic area, becomes the factor that caps the achievable masking order and therefore the attainable certification tier.

### 2.5 Structural limitations of conventional masked ML-DSA hardware

Beyond the arithmetic cost analysed above, the hardware organization of conventional masked ML-DSA signing cores imposes its own limitations. Existing protected designs are typically built by replicating an unprotected NTT-based datapath across share domains and inserting the register barriers required to prevent glitch-induced recombination. This organization exhibits the following structural weaknesses:

- **Replicated transform hardware.** Each share domain requires its own butterfly units and twiddle-factor storage, so transform hardware and its associated ROM scale with masking order even though the transform performs no share-crossing work.
- **Fixed-order datapaths.** The number of share lanes is generally fixed at design time. Changing the masking order to address a different certification tier requires modification and resynthesis of the datapath, and re-verification of the leakage properties, rather than reconfiguration of an existing core.
- **Register barriers on the critical path.** The register stages required between share-crossing operations to suppress glitch propagation add latency and area, and their placement must be revalidated whenever the datapath changes.
- **Control complexity across the rejection loop.** The variable iteration count of the signing rejection loop must be scheduled against a share-replicated datapath, and intermediate share state retained across iterations, increasing both control logic and storage.
- **Memory organization tied to full-width coefficients.** Share storage is banked and addressed on the assumption of full-width coefficients, forgoing the packing and banking opportunities that narrow coefficients would permit.

> **[PLACEHOLDER — 2.5, specific architectural gap]**
> *Insert here the specific structural limitation of conventional hardware that your architecture addresses. State it as a deficiency of existing designs, not as a description of your solution. Suggested items to cover, if applicable:*
> - *[the operation or datapath element your architecture eliminates or restructures, and why conventional organizations require it]*
> - *[the addressing or sequencing limitation of conventional designs relative to a sparse operand representation]*
> - *[the reason conventional share-domain organization prevents order reconfiguration without redesign]*
> - *[any memory or accumulator organization deficiency specific to your approach]*
> - *[any glitch/recombination handling cost that your structure avoids]*

Consequently, even where the arithmetic cost of high-order masking could be reduced, a hardware organization inherited from unprotected NTT designs does not permit that reduction to be realized as an area, latency or configurability benefit. A circuit organization matched to sparse operands and narrow-ring arithmetic is required.

---

## 3. Task Requiring a Solution

A method and circuit for computing the masked challenge–secret product in ML-DSA signature generation that simultaneously satisfies the following:

**(a)** Preserves the sparse, small-norm structure of the operands under masking, so that no wide modular multiplier is required anywhere in the protected datapath.

**(b)** Reduces the width of masked coefficient storage, so that the share-storage term does not dominate area as the masking order is raised.

**(c)** Constrains randomness consumption for the multiplication such that high-order protection remains achievable within a practical on-chip TRNG budget.

**(d)** Remains fully compliant with standard FIPS 204 parameters, requiring no modification to the signature scheme, its modulus, or its security argument.

**(e)** Scales to arbitrary masking order d without redesign of the datapath, so that a single core can serve multiple certification tiers.

**(f)** Admits a security proof in the standard t-probing model, so that the construction can be relied upon in certification.

**(g)** Is realizable as a hardware circuit whose organization is matched to sparse operands and narrow-ring arithmetic, rather than inherited from an unprotected NTT datapath — including [PLACEHOLDER: the specific structural properties your architecture provides, e.g. accumulator organization, sparse-table addressing, share-domain arrangement, order reconfiguration mechanism, memory banking].

**(h)** Maintains security against glitch-induced recombination in a physical implementation, without [PLACEHOLDER: the specific register-barrier or timing cost your architecture avoids].

---

## 4. Motivation and Purpose of the Invention

The devices being designed today for secure elements, automotive HSM cores, satellite and defense platforms, and industrial infrastructure have service lives of ten to twenty years. They must adopt ML-DSA now, both because CNSA 2.0 and comparable mandates require it and because their deployment horizon extends well past the point at which classical signatures cease to be defensible. For their entire service life these devices reside in the physical possession of the adversary, so high-order side-channel protection is not an optional hardening measure but a condition of certification and of fitness for purpose.

Yet these are precisely the devices in which silicon area, power and on-chip entropy generation are most severely constrained. The conventional approach forces the designer into a direct trade between the masking order the threat model demands and the area and randomness budget the product allows — and, as shown in Section 2.4, that trade tightens rather than relaxes as the order is raised.

The motivation for this invention arose from the observation that this trade is an artefact of the choice of working ring, not a property of the algorithm. ML-DSA's challenge polynomial is sparse and ternary and its secret coefficients are of small norm; the only reason the masked computation requires 23-bit arithmetic is that the shares are taken over Z_q. If the challenge–secret product can instead be evaluated on arithmetic shares within a narrow ring Z_M, chosen so that the correctness of the signing operation is provably preserved, then the structural economy of sparse multiplication survives masking intact.

The purpose of this invention is to realize that construction: to represent the challenge polynomial as a sparse (index, sign) table and to compute the challenge–secret product on arithmetic shares in a minimal ring Z_M, thereby removing the modular multiplier from the masked datapath entirely, narrowing masked coefficient storage from 23 bits to ⌈log₂ M⌉ bits at every masking order, and [reducing area by X% and randomness consumption by Y% at order t relative to a Z_q-masked NTT baseline].

The invention further comprises a hardware architecture realizing this construction, in which [PLACEHOLDER — architecture summary. Two or three sentences naming the principal structural elements and what each achieves. Suggested shape: *"the sparse challenge table drives [addressing mechanism], accumulation across share domains is organized as [accumulator/datapath structure], and masking order is configured by [reconfiguration mechanism], such that [the resulting area, latency or configurability benefit]."* Keep this at the level of structure and effect; detailed embodiments belong in the architecture section of the disclosure, not in the background.]

The construction is parameterizable in masking order without datapath redesign, enabling a single hardware IP core to serve certification tiers from first-order commercial IoT parts through EAL6+ secure elements, while remaining fully conformant to standard FIPS 204 parameters.
