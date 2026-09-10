# Prior Art

**Invention:** Masked minimal-ring sparse polynomial multiplication for ML-DSA (FIPS 204) signature generation

*Relevance key:* **H** = high, directly overlapping subject matter requiring distinction · **M** = medium, adjacent art establishing the state of the field · **L** = low, background context

---

## 1. Patents and Published Applications

| No. | Publication no. | Title | Assignee / inventors | Rel. | Relationship to this invention |
|---|---|---|---|---|---|
| P1 | US 12,362,931 B2 | Masked infinity norm check for CRYSTALS-Dilithium | NXP (Bronchain, Renes, Schneider) | **H — primary prior art** | **Full specification reviewed.** Discloses share-wise masked multiplication by the challenge polynomial on arithmetic shares modulo a reduced modulus q′; the correctness condition ‖ĉ∘ŝ‖∞ < q′ (claim 3); q′ as a power of two (claim 2); share width k′ = ⌈log₂q′⌉ (claim 9); and, in the description, the rationale that a smaller modulus is admissible **because** the secret key is of small norm and the challenge is public and sparse with coefficients in {−1,0,1}, so no reduction modulo q occurs. Also states the resulting reduction of secret-key storage from log₂q to log₂q′ bits, and that the approach applies to both software and hardware. **This anticipates the reduced-modulus element of the present invention; novelty is not asserted over it.** Does not disclose any structure for the multiplication, operand-dependent ring selection, order-configuration mechanism, memory organization, or physical-implementation leakage handling. See the separate "Difference from Primary Prior Art" document for the full analysis. |
| P2 | US 12,021,985 B2 | Masked decomposition of polynomials for lattice-based cryptography | NXP | M | Masked Decompose for Dilithium signing, adjacent to the challenge–secret product in the signing datapath. |
| P3 | US 11,924,346 B2 | Efficient and masked sampling of polynomials | NXP | M | Discloses arithmetic-to-arithmetic modulus conversion of shares enabling share-wise rejection sampling. Relevant to any claim involving a change of working modulus on shared values. |
| P4 | US 12,388,657 B2 | Low-memory masked Dilithium with alternative signing algorithm | NXP (Azouaoui, ElGhamrawy, Renes, Schneider) | M | Memory reduction in masked Dilithium signing; relevant to the share-storage argument in the background. Related application US 18/366,384 covers masked hint-vector computation. |
| P5 | US 12,118,098 B1 | Lower-order masking within a higher-order masked design | Abdulgadir, Elkhatib (PQSecure Technologies) | **H** | Discloses a domain-oriented masking multiplication gate of order M configured to perform order-N masking where N < M, by disabling cross-domain computations. **Directly relevant to any claim covering masking-order configurability of a single core.** |
| P6 | US 12,058,261 B2 | Low overhead side-channel protection for number-theoretic transform | Intel | M | Side-channel protection of the NTT, framed around multiplication by a secret polynomial in Dilithium. Represents the conventional protected-NTT approach the invention departs from. |
| P7 | US 11,792,004 B2 | Polynomial multiplication for side-channel protection | — | M | Side-channel-protected multiplication of a public polynomial by a private polynomial in lattice cryptography. |
| P8 | US 11,496,297 B1 | Low-footprint resource-sharing architecture for Dilithium and Kyber | — | L | Shared arithmetic unit covering polynomial multiplication and decomposition across both algorithms; architectural background relevant to the ML-KEM reuse argument. |
| P9 | US 12,368,581 B2 | Cryptographic processing system and method for CRYSTALS-Kyber and CRYSTALS-Dilithium using table-based A2B masked share conversion | — | **H** | Discloses use of a larger power-of-two modulus Q to avoid dedicated modular reduction, and expressly relaxes the requirement to Q > n·q·μ where a polynomial retains its smallness with coefficients in [−μ,+μ] — i.e. an operand-bound-dependent modulus selection. **Same family of reasoning as P1; review in full alongside it.** |

> **[PLACEHOLDER — additional patents]**
> *Add any further patents identified by formal search. A keyword search of the kind performed here will not surface pending applications or filings indexed under CPC classes not swept. Recommend a professional search covering at least CPC H04L 9/3247 (signature schemes), H04L 9/003 (side-channel countermeasures) and G06F 7/72 (modular arithmetic).*

**Note on verification:** the publication numbers above were obtained by public keyword search and should be verified against the official register before submission. Claim scope, not the abstract or description, governs the freedom-to-operate analysis — particularly for P1 and P5.

---

## 2. Academic Literature

### 2.1 Sparse and small-polynomial multiplication for ML-DSA (unmasked)

Establishes the sparse-representation technique as known art in unprotected implementations.

| No. | Reference | Rel. | Relationship |
|---|---|---|---|
| A1 | Zhao, Zhao, Zhu, Yang, Wei, Liu, "Sparse Polynomial Multiplication-Based High-Performance Hardware Implementation for CRYSTALS-Dilithium," *IEEE HOST 2024*, pp. 150–159 | **H** | Sparse multiplication applied in the signing pipeline with a dedicated sparse computing core; reports reduction in multiplication bit complexity exceeding 85% and secret-key storage exceeding 68%. Closest unmasked hardware analogue to the sparse half of this invention. |
| A2 | Zheng et al., "ESPM-D: Efficient Sparse Polynomial Multiplication for Dilithium on ARM Cortex-M4," arXiv:2404.12675; ACISP 2024 | **H** | Index-based sparse multiplication exploiting the τ non-zero ±1 coefficients of c, decoded into signed integer index arrays. Establishes the (index, sign) representation. |
| A3 | Zhou, Chen et al., "Parallel Small Polynomial Multiplication for Dilithium" (PSPM), *ACSAC 2022* | M | Origin of the sparse multiplication technique for Dilithium. |
| A4 | "Parallel Sparse Ternary Polynomial Multiplication for ML-DSA," IACR ePrint 2023/1522 | M | Sparse ternary formulation stated directly for ML-DSA parameters. |
| A5 | "Parameter-Aware and Instruction-Driven Dilithium Optimization" (Fast-SPM), IACR ePrint 2026/1272 | M | Converts c·t₀ into index-shifted additions, bypassing the NTT. **Recent — confirm publication date against your priority date.** |

### 2.2 Masking that preserves small-secret or sparse structure

The most directly overlapping subject matter.

| No. | Reference | Rel. | Relationship |
|---|---|---|---|
| B1 | Aikata, Basso, Cassiers, Mert, Sinha Roy, "Kavach: Lightweight Masking Techniques for Polynomial Arithmetic in Lattice-Based Cryptography," *TCHES 2023(3)*, 366–390 (ePrint 2023/517) | **H** | States the same problem this invention addresses: compact multipliers exploit the small range of secret coefficients, but masking forces those coefficients across a large range. Proposes NTT-based and non-NTT techniques enabling masked polynomial multiplication while retaining the small-secret property, reusing unmasked compact multipliers; first-order TVLA validated. **The principal reference to distinguish against.** Distinction to assert: Kavach retains the small-secret property within Z_q; this invention relocates the computation into a narrow ring Z_M. |
| B2 | Land, Marotzke, Richter-Brockmann, Güneysu, *TCHES* | **H** | Identifies c·s₁ followed by addition to y as the most critical signing operation, and notes that c is public even for rejected candidates and is sparse and ternary. Closest published articulation of the threat model and operand structure motivating this invention. |

### 2.3 Reduced-modulus and modulus-switching approaches to masking

Relevant to the narrow-ring element of the invention.

| No. | Reference | Rel. | Relationship |
|---|---|---|---|
| C1 | Migliore, Gérard, Tibouchi, Fouque, "Masking Dilithium: Efficient Implementation and Side-Channel Evaluation," *ACNS 2019* (ePrint 2019/394) | **H** | Establishes that replacing the prime modulus with a power of two yields a considerably more efficient masked scheme, reporting 7.3–9× speedup on the most costly masking operations. **Distinction to assert: this approach modifies the scheme and is not FIPS 204 conformant; the present invention changes only the working ring of the multiplication while preserving standard parameters.** |
| C2 | Coron, Gérard, Trannoy, Zeitoun — modulus-switching lemma, as restated in *TCHES 2024(4)* | **H** | Formalizes conversion of an arithmetic sharing between moduli with bounded additive error. Governs any claim involving switching shares into a narrower ring; the correctness argument of this invention must be distinguished from, or shown to improve upon, this result. |

### 2.4 High-order masking gadgets and proof machinery

Cited as building blocks and as the conventional-cost baseline rather than as blocking art.

| No. | Reference | Rel. | Relationship |
|---|---|---|---|
| D1 | Coron, Gérard, Trannoy, Zeitoun, "Improved Gadgets for the High-Order Masking of Dilithium," *TCHES 2023(4)*, 110–145 | M | ShiftMod gadget, B2A mod arbitrary q with complexity independent of bit-width and modulus, improved masked Decompose; t-probing proofs. Reference point for high-order cost comparison. |
| D2 | Coron, Gérard, Lepoint, Trannoy, Zeitoun, "Improved High-Order Masked Generation of Masking Vector and Rejection Sampling in Dilithium," *TCHES 2024(4)*, 335–354 (ePrint 2024/1149) | M | Masked rejection sampling with complexity independent of modulus size; ML-DSA hedged derivation. |
| D3 | Azouaoui, Bronchain et al., "Protecting Dilithium against Leakage," *TCHES 2023* | M | Classifies signing intermediates by physical-security requirement. Supports the argument in the background regarding which operations require which masking order. |
| D4 | Barthe et al., "Masking the GLP Lattice-Based Signature Scheme at Any Order," *EUROCRYPT 2018* | L | Origin of gadget-based arbitrary-order masking for lattice signatures. |
| D5 | Belaïd et al., SUCRE (2026) | L | Unmask-with-permutation gadget preserving the infinity norm. |
| D6 | Raj, Ravi, Chia, Chattopadhyay, ePrint 2024/1817 (*J. Hardw. Syst. Secur.*, 2026) | M | Hardware realization of the D1 gadgets for first-order ML-DSA. Baseline for hardware cost comparison. |

### 2.5 Attacks establishing the need for high-order protection

Support the necessity argument in the background; not blocking art.

| No. | Reference | Rel. | Relationship |
|---|---|---|---|
| E1 | Bronchain, Azouaoui, ElGhamrawy, Renes, Schneider, "Exploiting Small-Norm Polynomial Multiplication with Physical Attacks: Application to CRYSTALS-Dilithium," *TCHES 2024* | M | Soft-analytical side-channel attack on the challenge–secret product. Establishes the specific operation this invention protects. |
| E2 | Mujdei, Wouters, Karmakar, Beckers, Mera, Verbauwhede, "Side-Channel Analysis of Lattice-Based Post-Quantum Cryptography: Exploiting Polynomial Multiplication," *ACM TECS 23(2)* | M | Reports key recovery against second- and third-order masked implementations. Supports the requirement for high masking order. |
| E3 | Steinegger, Primas et al., "Breaking and Protecting the Crystal," *PQCrypto 2023* | L | Further attack context. |

---

## 3. Undocumented and Internal Prior Art

> **[PLACEHOLDER — complete this section yourself. I have no visibility into internal sources.]**
>
> *List any of the following that exist. Omissions here are a common cause of later invalidity or ownership disputes, so err toward over-disclosure.*
>
> **Internal sources:**
> - *Prior internal designs, RTL, or IP cores implementing sparse or masked polynomial multiplication — include project name, approximate date, and whether the work was published or remained confidential*
> - *Internal technical reports, design reviews, or architecture proposals covering related subject matter*
> - *Earlier invention disclosures by yourself or colleagues touching masked lattice arithmetic, including any not filed*
> - *Prior art known from your own or colleagues' work at previous employers, where disclosable*
> - *Internal presentations or seminars in which any element of this concept was described, with dates and audience*
>
> **External non-patent sources:**
> - *Conference talks, workshop presentations, or rump sessions describing similar approaches*
> - *Open-source implementations (e.g. public repositories implementing masked ML-DSA or sparse multiplication)*
> - *Standards or working-group contributions (NIST PQC forum posts, ISO/IEC working documents)*
> - *Private communications or reviewer feedback in which the approach was described*
>
> **Your own prior disclosures — critical:**
> - *Any occasion on which you described this concept publicly: papers, preprints, talks, forum posts, or discussion with parties outside a confidentiality obligation. Record the date. Public disclosure by the inventor can start a grace period or bar patentability outright depending on jurisdiction, and counsel must know about it before filing.*

---

## 4. Summary of the Distinction to be Asserted

Each individual element of the invention appears in the art: sparse (index, sign) multiplication for ML-DSA is established by A1–A5; masking that preserves small-secret structure by B1; reduced-modulus masking by C1, P1 and P9; and order configurability by P5.

**Elements over which novelty is NOT asserted.** Following full review of P1's specification, the following are acknowledged as disclosed in the prior art and are not relied upon for the distinction:

- Selection of a reduced arithmetic masking modulus for the challenge–secret product (P1, P9)
- The correctness condition ‖ĉ∘ŝ‖∞ < q′ governing that selection (P1 claim 3)
- The consequent reduction of masked coefficient storage to ⌈log₂q′⌉ bits (P1)
- Identification of challenge sparsity and secret small-norm as the enabling properties (P1)
- Applicability of the reduced-modulus approach to hardware as well as software (P1)
- Sparse (index, sign) representation of the challenge in unprotected implementations (A1, A2)

**Distinctions to be asserted:**

1. **Circuit realization of the multiplication.** No reference discloses a structure for the masked multiplication itself. P1 treats it as a functional block; A1–A5 disclose sparse multiplication only in unprotected form. [PLACEHOLDER — state the structure.]
2. **Operand-dependent ring width across the signing datapath.** P1's single reduced modulus is justified by the η-bounded norm of s₁/s₂ and does not extend to t₀. Heterogeneous ring widths within one core, and the mixed-width accumulation and storage organization supporting them, are not disclosed.
3. **Versus B1 (Kavach):** the small-norm property is preserved by relocating the computation into a narrow ring rather than by retaining small-secret structure within Z_q, so no modular multiplication unit remains instantiated in the protected datapath.
4. **Versus C1 (Migliore et al.):** the signature scheme is unmodified and standard FIPS 204 parameters are retained.
5. **Versus P5 (PQSecure):** [PLACEHOLDER — state how the order-configuration mechanism differs from disabling cross-domain computation in a DOM gate.]

The hardware organization, the operand-dependent ring selection, and the order-configuration mechanism are the elements most likely to sustain claims. The reduced-modulus concept alone will not.
