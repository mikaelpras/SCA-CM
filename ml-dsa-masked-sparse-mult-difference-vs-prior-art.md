# Difference Between the Present Invention and the Primary Prior Art

**Invention:** Masked sparse polynomial multiplication in a minimal ring for ML-DSA (FIPS 204) signature generation

**Primary prior art:** US 12,362,931 B2 — *Masked infinity norm check for CRYSTALS-Dilithium signature generation* (NXP B.V.; inventors Bronchain, Renes, Schneider)

*Bracketed items marked [PLACEHOLDER] require your architecture details or measured figures.*

---

## 1. Full-Specification Review of the Primary Prior Art

Per the guideline, the analysis below covers the entire specification of US 12,362,931 B2 — description, drawings and claims — not the claims alone.

### 1.1 Disclosure of the primary prior art

**Objective of P1.** P1 addresses the cost of the masked rejection step of Dilithium signing, specifically the computation ‖ŷ + ĉ∘ŝ‖∞ ≤ β. Its stated aim is to reduce execution time and memory by keeping most arithmetic in the Boolean masking domain, thereby avoiding conversion of ŷ from Boolean to arithmetic masking.

**Structure disclosed.** FIG. 2 of P1 shows: a multiplier computing ĉ∘ŝ on arithmetic shares with modulus q′; a SecA2BMod q′ conversion; a Boolean SecAdd against the shares of ŷ; and two further SecAdd operations producing a carry bit that signals accept or reject. Algorithms 4 (SecGenZ) and 5 (SecSubAndCheck) give the detailed operation sequence. FIG. 3 is a generic processor / memory / bus / storage diagram; no dedicated datapath, accumulator or memory organization is disclosed.

**Sections of P1 that disclose part of the underlying concept of the present invention.** The following must be acknowledged explicitly:

| P1 location | Content disclosed |
|---|---|
| Detailed description, discussion of memory reduction | States that a smaller arithmetic masking modulus — possibly a power of two — may be used instead of the Dilithium modulus q, **because** the secret key is of small norm (coefficients in [−2,2] or [−4,4]) and is multiplied by a public **sparse** polynomial with coefficients in {−1,0,1}, so that no reduction modulo q can occur. |
| Detailed description, discussion of FIG. 2 | States the constraint on the modulus is ‖ĉ∘ŝ‖∞ < q′, and that q′ may be either a small prime permitting NTT-based multiplication or a small power of two permitting the cheaper SecA2B. |
| Detailed description, memory advantage | States that when masked modulo q′, secret-key coefficients require log₂q′ bits of storage rather than log₂q bits. |
| Detailed description, closing remark on applicability | States the approach applies to both software and hardware implementations. |
| Claim 3 / claim 13 | Recites ‖ĉ∘ŝ‖∞ < q′. |
| Claim 2 / claim 12 | Recites q′ as a power of two. |
| Claim 9 / claim 19 | Recites Boolean shares of k′ = ⌈log₂q′⌉ bits. |
| Description, SecGenZ line 1 | States that multiplication by ĉ is linear with respect to arithmetic masking and may be applied share-wise to each share. |

**Consequence for the present invention.** The narrow-ring rationale, the correctness condition governing the choice of modulus, the resulting storage reduction, and the identification of challenge sparsity and secret small-norm as the enabling properties are all disclosed by P1. For the s₁ and s₂ paths, the specific modulus width follows arithmetically from claims 2 and 3 applied to standard parameters (2^(M−1) ≥ τ·η gives M = 8 for ML-DSA-44 and ML-DSA-87, M = 9 for ML-DSA-65) and is therefore not a point of distinction.

**Accordingly, the distinction asserted below does not rest on the choice of a reduced modulus.** It rests on circuit structure, on operand-dependent ring selection, and on the realization of the multiplication itself, none of which P1 discloses.

### 1.2 What P1 does not disclose

- **Any structure for the multiplication.** P1 treats the multiplier as a functional block. It states only that multiplication by ĉ is linear under arithmetic masking and may be applied share-wise, and that q′ may be a small prime for NTT-based multiplication or a power of two. No sparse operand representation, addressing scheme, accumulator organization or multiplier-free realization is described.
- **Operand-dependent ring selection.** P1 contemplates a single modulus q′. Its rationale is expressly grounded in the small norm of the secret key (η-bounded coefficients), and therefore does not extend to the t₀ path, whose coefficients span 13 bits.
- **Any masking-order configuration mechanism.** The share count d is a parameter of the algorithms; no means of reconfiguring order within a fixed hardware realization is disclosed.
- **Any memory or register organization** exploiting narrow coefficient width — banking, packing or width-differentiated share storage.
- **Any glitch or recombination handling** in a physical circuit. P1's embodiment is instruction-sequence based; register placement and physical-implementation leakage are not addressed.
- **Any dedicated hardware datapath.** FIG. 3 is a generic computing platform.

---

## 2. Comparison Table

| | **Primary prior art — US 12,362,931 B2** | **Present invention** |
|---|---|---|
| **Objective** | Reduce execution time and memory consumption of the masked rejection / infinity-norm check in Dilithium signing, by performing most arithmetic in the Boolean masking domain and thereby avoiding Boolean-to-arithmetic conversion of the masking vector ŷ. | [PLACEHOLDER — state in structural terms. Suggested: *To provide a circuit organization for the masked challenge–secret product in which the modular multiplier is eliminated as a hardware element, share storage is width-differentiated per operand class, and masking order is reconfigurable without datapath redesign.* Do **not** state the objective as reducing the masking modulus — that is P1's objective.] |
| **Structure / composition** | Processor-executed instruction sequence. Share-wise multiplication ĉ∘ŝ over arithmetic shares modulo q′ (structure unspecified) → SecA2BMod q′ → SecSub against a randomly generated, γ-offset polynomial → SecAdd(β+γ) → SecAdd(2^(k+2) − 2β) → carry bit → unmask. Bitsliced secret representation. q′ a small prime or a power of two. Hardware embodiment is a generic processor, memory, storage and bus arrangement. | [PLACEHOLDER — **this cell carries the invention.** Recite concrete structure, e.g.: sparse (index, sign) table representation of ĉ and its decode logic; the addressing mechanism by which table entries drive accumulation; accumulator organization across share domains; width-differentiated share register banking (M-bit for the s-path, M′-bit for the t₀ path); the order-configuration mechanism; register placement between share-crossing operations. Every element P1 leaves unspecified is claimable structure here.] |
| **Effect** | Saves two SecA2BMod q conversions and one SecAdd at the cost of one additional SecA2BMod q′; secret-key coefficients stored in log₂q′ bits rather than log₂q bits; bitsliced additions consume only the exact bit width required. | [PLACEHOLDER — cite effects that follow from **structure**, not from the ring choice. Suggested categories: gate count with no modular multiplier instantiated in the protected datapath; area at masking order d = 2, 4, …; latency per signing operation; ability to reconfigure masking order without resynthesis or re-verification; share-storage reduction attributable to per-operand ring width rather than to a single reduced modulus. Effects attributable to the narrow modulus alone will be read as P1's effects and will not support the distinction.] |
| **Summary of the difference** | Directed to the **arithmetic and conversion sequence** of the norm check, executed as instructions on a general-purpose processor. The multiplication is treated as a linear share-wise operation whose realization is left unspecified. A single reduced modulus q′ is selected on the basis of the small norm of the secret key. | [PLACEHOLDER — directed to the **circuit realization of the multiplication itself**, and to per-operand-class ring selection across the signing datapath. The reduced modulus is a shared precondition, not the point of difference. Complete after finalizing the architecture.] |

---

## 3. Specific Points of Distinction

### 3.1 Structural distinction — circuit realization of the multiplication

P1 discloses no structure for the multiplication. The present invention is directed to that structure: [PLACEHOLDER — describe the sparse-table addressing, accumulator organization, and how shares are routed and stored. This is the primary asserted distinction and must be stated concretely, in terms of circuit elements and their interconnection, not in terms of an outcome.]

### 3.2 Operand-dependent ring selection across the signing datapath

P1 contemplates a single modulus q′, justified by the small norm of the secret key s. That justification does not extend to t₀, whose coefficients span 13 bits and for which ‖c∘t₀‖∞ ≤ τ·2¹², requiring a wider ring (M′ = 19 across all parameter sets) than the s-path (M = 8 or 9).

The present invention employs **heterogeneous ring widths within a single signing core**, with the ring selected per operand class from the applicable coefficient bound, together with a datapath and share-storage organization supporting mixed-width masked accumulation. P1 discloses neither the wider-ring analysis for t₀ nor any mechanism for supporting multiple ring widths in one datapath.

*Verification required:* confirm the τ, η and t₀ coefficient bounds against FIPS 204 directly, and confirm the t₀ bound is consistent with the γ₂ check. Note also US 12,388,657 B2 and its published application, which recite a bound check on c·t₀ against γ₂ and should be reviewed for any disclosure touching modulus width on that path.

### 3.3 Masking-order configurability

[PLACEHOLDER — state the mechanism. Must be distinguished from US 12,118,098 B1, which discloses a DOM multiplication gate of order M configured to perform order-N masking (N < M) by disabling cross-domain computations. If the present mechanism operates differently — e.g. by lane gating, storage repartitioning, or scheduling rather than by disabling cross-domain terms — state precisely how.]

### 3.4 Physical-implementation security

[PLACEHOLDER — P1's embodiment is instruction-sequence based and does not address glitch-induced recombination. State the register placement and any structural measure the present invention employs, and the resulting physical-leakage property.]

---

## 4. Compliance Check Against the Stated Screening Criteria

The disclosure form warns that inventions in the following categories may be merged with existing cases or not processed further. Each is addressed:

| Screening criterion | Assessment |
|---|---|
| **Merely raising a technical issue without a distinct idea** | Not applicable. The invention recites specific circuit structure — [PLACEHOLDER: name the two or three principal structural elements] — rather than identifying a problem alone. |
| **Emphasizing a different result without structural or methodological difference from conventional methods** | **This is the criterion of greatest concern and must be addressed directly.** The reduced-modulus effect and the storage reduction are disclosed by P1; asserting them as the advantage would fall squarely within this criterion. The distinction asserted here is therefore structural: [PLACEHOLDER — the circuit organization], together with operand-dependent ring selection, neither of which P1 discloses. Performance figures are cited only as consequences of that structure. |
| **Borrowing an existing idea and merely changing the application location or material type** | **Also directly relevant.** P1 expressly states that its approach applies to both software and hardware implementations. An argument resting on "P1 is software, the present invention is hardware" would fall within this criterion and must not be made. The asserted distinction is not the implementation medium but the circuit organization, which P1 leaves entirely unspecified in either medium. |

---

## 5. Candid Assessment for the Reviewer

It is acknowledged that US 12,362,931 B2 discloses the use of a reduced arithmetic masking modulus for the challenge–secret product, the correctness condition governing that choice, the resulting storage reduction, and the sparsity and small-norm properties that enable it. The present disclosure does not assert novelty over those elements.

The asserted contribution is the circuit organization by which the masked multiplication is realized — [PLACEHOLDER: one-sentence summary] — together with operand-dependent ring width across the signing datapath and a masking-order configuration mechanism. These elements are not disclosed in P1's specification, drawings or claims, nor in the further references listed in the Prior Art section.

> **[PLACEHOLDER — inventor's own assessment]**
> *Before submission, satisfy yourself that the architecture recited in Sections 2 and 3.1 constitutes a genuine circuit organization rather than a routine hardware implementation of P1's method. If the architecture reduces on inspection to "P1's algorithm, in RTL," the second and third screening criteria above will apply and the case should be withdrawn or merged rather than prosecuted. If it recites structure that a person implementing P1 would not arrive at as a matter of routine design, the distinction is sound.*
