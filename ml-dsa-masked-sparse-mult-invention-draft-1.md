# Invention Description — Draft

**Title (working):** Masked sparse polynomial multiplication circuit with minimal-ring datapath and carry-sign modulus conversion for ML-DSA signature generation

*Bracketed items marked [ ] require your figures or confirmation. Items marked ⚠ require a decision before filing.*

---

## 1. Field of the Invention

A hardware circuit for computing the masked product of a public sparse challenge polynomial with a secret polynomial, as required by the ML-DSA (FIPS 204) signature generation algorithm, protected against side-channel analysis by arithmetic masking at configurable order.

---

## 2. Block Interface and Scope

The invention is a signing-datapath block with the following defined interface. Elements outside this interface are assumed present and are not claimed.

**Inputs**
- d arithmetic shares of the secret polynomial s (or t₀), held at ring width M
- The public challenge polynomial c, supplied as a table of entries each comprising an index field and a value field
- Public parameters: masking order d, ring width M, target modulus q

**Outputs**
- d arithmetic shares modulo q of the product c∘s

**Assumed present and not claimed:** the masking vector y and the commitment-derived w, each already held as arithmetic shares modulo q. Per FIPS 204 Algorithm 7, y is consumed by the NTT at line 12 and w is produced by NTT⁻¹ at line 12, so both are necessarily in arithmetic form modulo q at the point where the present block's output is consumed (line 20 for z, line 21 for r₀). The output interface of the present invention is therefore arithmetic shares modulo q by construction of the standard, not by design choice.

---

## 3. Architecture

### 3.1 Overall organization

The circuit comprises a plurality of masked sparse multiplication units and a plurality of accumulator units, arranged on two orthogonal axes of parallelism:

- **Share axis.** Each sparse multiplication unit is assigned to one masking share domain. The number of instantiated units scales with the masking order d.
- **Output-coefficient axis.** Each accumulator unit is assigned to a subset of output coefficients. The number of accumulator units is independent of the masking order.

The entire multiplication datapath operates modulo M. No arithmetic modulo q, and no modular multiplier, is instantiated within the multiplication datapath.

### 3.2 Table-driven addressing — no multiplier

The challenge polynomial is supplied as a table of (index, value) entries, one per non-zero coefficient. For each entry:

- The **index field** is used to generate a memory address into the share storage, selecting the secret coefficient to be accumulated.
- The **value field** is used as a sign control to the accumulator unit, selecting addition or subtraction.

Because the challenge coefficients are confined to {−1, 0, +1} and only the τ non-zero entries are represented, the product is formed entirely by addressed accumulation under sign control. No multiplication operator is instantiated anywhere in the protected datapath. This is the structural basis of the area reduction and is claimed as such.

### 3.3 Dataflow — tap-first accumulation

Accumulation proceeds tap-first: for a given tap, contributions are accumulated across the relevant output coefficients before the circuit advances to the next tap. [PLACEHOLDER — state the consequence you measured: accumulator count required, memory access pattern, and why tap-first was selected over coefficient-first. If the choice reduces address-generation logic or memory port pressure, say so quantitatively.]

### 3.4 Same-tap share separation

A one-cycle pipeline delay is imposed such that the shares belonging to a single tap are processed in different clock cycles. This is distinct from the register barriers conventionally placed between share domains in domain-oriented masking: the separation here is temporal and is applied *within* a tap, so that the share components of one secret coefficient are never combinationally present in the same cycle.

[PLACEHOLDER — state the specific leakage mechanism this addresses, e.g. glitch propagation through the shared address-generation or accumulator input path, and confirm by TVLA or formal check.]

### 3.5 Masking-order configurability

Masking order is changed by instantiating a corresponding number of sparse multiplication units along the share axis. The accumulator axis is unaffected. Order therefore scales by replication along one axis of an orthogonal structure, without modification of the accumulator organization, the addressing logic, or the dataflow schedule.

### 3.6 Operand-dependent ring width

The ring width M is selected per operand class from the applicable coefficient bound:

| Operand | Bound on product | Required width |
|---|---|---|
| c∘s₁, c∘s₂ | ‖c∘s‖∞ ≤ τ·η | M = 8 (ML-DSA-44, ML-DSA-87), M = 9 (ML-DSA-65) |
| c∘t₀ | ‖c∘t₀‖∞ ≤ τ·2^12 | M′ = 19 (all parameter sets) |

[⚠ VERIFY against FIPS 204 directly — τ, η and the t₀ coefficient range — and confirm consistency with the γ₂ check at Algorithm 7 line 28. The mixed-width argument depends on these values.]

The circuit supports both widths within a single signing core. [PLACEHOLDER — state whether the two paths share accumulation hardware, share memory banked at two widths, or are separately instantiated, and how width is selected.]

### 3.7 Carry-sign modulus conversion

The output shares must be delivered modulo q. Conversion is performed as follows:

1. Convert the mod-M arithmetic sharing to a Boolean sharing (A2B over ⌈log₂M⌉ bits)
2. Perform a secure addition (SecAdd) over the Boolean sharing
3. Extract the sign bit and the carry bit positions
4. Convert back to an arithmetic sharing
5. Apply the correction v = u − (carry + sign)·M mod q

The correction at step 5 is a subtraction of a public constant multiple and is applied share-wise, hence linear in the masking and effectively free. The conversion exploits the fact that the magnitude of the product is bounded by construction, so the value modulo q is determined by the mod-M representative together with the extracted carry and sign information.

[PLACEHOLDER — gadget-call count for this conversion at order d and width M, to be compared against the prior-art figures in Section 5.3.]

⚠ **Security review required.** The extracted carry and sign bits are derived from shared values. Confirm that the extraction operates on shared carries throughout and that no combinational path allows shares to interact, and obtain a t-probing argument rather than TVLA evidence alone. This is the element of the design most likely to admit a first-order leak that a small-sample TVLA would not detect.

---

## 4. Effects

| Effect | Basis | Figure |
|---|---|---|
| No modular multiplier instantiated in the protected datapath | §3.2 — table-driven addressed accumulation | [gate count vs. Z_q masked baseline] |
| Share storage narrowed to M or M′ bits | §3.6 — operand-dependent ring width | [bits per polynomial at order d] |
| Order scaling by single-axis replication | §3.5 | [area at d = 1, 2, 4] |
| Reduced conversion cost at the mod-q interface | §3.7 — narrow A2B plus constant correction | [gadget calls vs. prior art] |
| Latency per signing operation | §3.3 — tap-first schedule | [cycles] |

Note: the reduction in share storage attributable to a narrow masking modulus per se is disclosed in the prior art (see Section 5). It is stated here as an effect of the architecture, not as a point of novelty.

---

## 5. Distinction from the Primary Prior Art

**Primary prior art: US 12,362,931 B2 (NXP) — masked infinity norm check for CRYSTALS-Dilithium.**

### 5.1 Acknowledged as disclosed in P1

Novelty is **not** asserted over the following, each of which appears in P1's specification or claims:

- Use of a reduced arithmetic masking modulus q′ for the challenge–secret product
- The correctness condition ‖ĉ∘ŝ‖∞ < q′ (claim 3)
- Selection of q′ as a power of two (claim 2)
- Share width ⌈log₂q′⌉ (claim 9)
- The rationale that challenge sparsity with coefficients in {−1,0,1} and secret small norm permit the reduced modulus
- Reduction of secret-key storage from log₂q to log₂q′ bits
- Applicability of the reduced-modulus approach to hardware as well as software

For the s₁/s₂ path, the specific value M = 8 or 9 follows arithmetically from P1 claims 2 and 3 applied to standard parameters and is not asserted as a distinction.

### 5.2 Not disclosed in P1 — asserted distinctions

| # | Distinction | Why P1 does not reach it |
|---|---|---|
| D1 | **Two-axis parallelism with order scaling on the share axis** (§3.1, §3.5) | P1 discloses no parallel structure of any kind. Its share count d is an algorithm parameter; no hardware organization is described. FIG. 3 is a generic processor/memory/bus diagram. |
| D2 | **Table-driven addressed accumulation with no multiplier instantiated** (§3.2) | P1 treats the multiplication as a functional block. Its description states only that multiplication by ĉ is linear under arithmetic masking and may be applied share-wise. No operand representation, addressing, or accumulator structure is disclosed. P1 does not teach NTT either — it names a small prime with NTT or a power of two as alternatives, and prefers the power of two, which precludes NTT. It is silent on realization in both cases. |
| D3 | **Same-tap temporal share separation** (§3.4) | P1's embodiment is an instruction sequence. It does not address physical-implementation leakage, register placement, or glitch-induced recombination. |
| D4 | **Operand-dependent ring width across the signing datapath** (§3.6) | P1 contemplates a single q′, justified expressly by the η-bounded norm of the secret key. That rationale does not extend to t₀, whose coefficients span 13 bits. No mechanism for supporting multiple ring widths in one datapath is disclosed. |
| D5 | **Carry-sign modulus conversion with constant correction** (§3.7) | P1 uses a generic SecA2BMod q′ and remains in the Boolean domain thereafter. It discloses no extraction of carry and sign information, and no correction identity of the form v = u − (carry+sign)·M mod q. |
| D6 | **Tap-first accumulation schedule** (§3.3) | Not disclosed. Weakest of the six — claim as dependent, tied to accumulator organization. |

### 5.3 On the cost comparison for D5

P1's saving rests on the assumption that the masking vector ŷ is regenerated by PRNG at the rejection step and is therefore Boolean-masked, permitting a Boolean SecAdd and avoiding conversion. That assumption reflects a memory-constrained software setting.

It does not hold for the present architecture. Per FIPS 204 Algorithm 7, y is consumed by the NTT at line 12 and w is produced by NTT⁻¹, so both are held as arithmetic shares modulo q. P1 expressly addresses this case and states that its method then costs one additional SecA2BMod q. The present invention therefore operates in the regime P1 identifies as its weaker case, and the mod-q output interface is a requirement of the standard rather than a design choice.

[⚠ REQUIRED BEFORE FILING — complete this comparison:]

| | P1, arithmetic-y variant | Present invention |
|---|---|---|
| A2B conversions | 1 × SecA2BMod q′ + 1 × SecA2BMod q | [1 × A2B over ⌈log₂M⌉ bits] |
| B2A conversions | none | [1] |
| SecAdd / SecSub | 3 | [ ] |
| Constant corrections | included | 1 (share-wise, linear) |
| **Total at order d, width M** | [ ] | [ ] |

If the total does not favour the present invention, D5 should be demoted to a dependent claim and the case rested on D1–D4.

### 5.4 Distinction from US 12,118,098 B1 (order configurability)

US 12,118,098 B1 discloses a domain-oriented masking multiplication gate of order M configured to perform order-N masking (N < M) by **disabling cross-domain computations** within a fixed-order gate. The present invention scales order by **instantiating units along one axis of an orthogonal two-axis structure**, leaving the accumulator axis and the dataflow schedule unchanged. The mechanisms are structurally different: the prior art gates computation within a fixed structure; the present invention replicates along a dimension of a structure that the prior art does not disclose.

### 5.5 Distinction from other references

- **Kavach (TCHES 2023(3)):** preserves the small-secret property *within* Z_q, retaining a modular multiplication unit. The present invention relocates the computation to a narrow ring and instantiates no multiplier.
- **Migliore et al. (ACNS 2019):** modifies the scheme's modulus. The present invention leaves FIPS 204 parameters unchanged.
- **Zhao et al. (HOST 2024), ESPM-D, PSPM:** disclose sparse multiplication in **unprotected** implementations only. None addresses masking, share domains, or the interaction of sparse addressing with share separation.
- **US 12,368,581 B2:** operand-bound-dependent power-of-two modulus selection. Same family of reasoning as P1; review in full. Does not disclose circuit structure.

---

## 6. Claim Skeleton

**Claim 1 (apparatus — datapath).** A circuit for computing a masked product of a public polynomial and a secret polynomial, comprising:
- a table store holding a plurality of entries, each entry comprising an index field and a value field, the entries corresponding to non-zero coefficients of the public polynomial;
- a share memory holding d arithmetic shares of the secret polynomial at a ring width M;
- address generation logic configured to produce an address into the share memory from the index field;
- a plurality of sparse multiplication units, each associated with one of the d share domains;
- a plurality of accumulator units, each associated with a respective subset of output coefficients, each configured to add or subtract an addressed operand in dependence on the value field;
- wherein the circuit contains no modular multiplier in the path of the shares of the secret polynomial;
- and pipeline registers configured such that shares associated with a single entry are processed in different clock cycles.

**Claim 2.** The circuit of claim 1, wherein the number of sparse multiplication units corresponds to the masking order d, and the number of accumulator units is independent of the masking order.

**Claim 3.** The circuit of claim 1, wherein accumulation proceeds for a first entry across the associated output coefficients before advancing to a second entry.

**Claim 4.** The circuit of claim 1, wherein the share memory holds shares of a first secret polynomial at a first ring width and shares of a second secret polynomial at a second, greater ring width.

**Claim 5.** The circuit of claim 4, wherein the first ring width is determined by a bound τ·η and the second by a bound τ·2^12.

**Claim 6 (independent — conversion circuit).** A modulus conversion circuit for a masked polynomial product, comprising: an arithmetic-to-Boolean converter operating over ⌈log₂M⌉ bits; a secure adder; extraction logic configured to extract a sign bit and at least one carry bit from the Boolean sharing; a Boolean-to-arithmetic converter; and correction logic configured to subtract a public constant multiple of M in dependence on the extracted sign and carry bits.

**Claim 7.** The circuit of claim 6, wherein the correction is applied share-wise.

**Claim 8 (independent — method).** [Mirror claim 1.]

**Claim 9 (independent — system).** A signature generation circuit for ML-DSA comprising the circuit of claim 1 and the circuit of claim 6, wherein the output of the circuit of claim 1 is supplied to the circuit of claim 6, and the output of the circuit of claim 6 is arithmetic shares modulo q.

**Deliberately not claimed:** a reduced masking modulus per se; the bound ‖c∘s‖∞ < 2^(M−1) standing alone; sparse multiplication of a ternary polynomial standing alone. All three are in the prior art. They appear in the specification as context.

---

## 7. Pre-Filing Checklist

| # | Item | Status |
|---|---|---|
| 1 | Verify τ, η, t₀ range against FIPS 204; confirm γ₂ consistency | ⚠ open |
| 2 | Read US 12,362,931 B2 and US 12,368,581 B2 in full | ⚠ open |
| 3 | Complete the gadget-count comparison in §5.3 | ⚠ open |
| 4 | t-probing argument for the carry-sign conversion (§3.7) | ⚠ open |
| 5 | Synthesis figures for §4 at order d = 1, 2, 4 | ⚠ open |
| 6 | Confirm §3.4 addresses a specific, identified leakage path | ⚠ open |
| 7 | Record any prior public disclosure of any element, with dates | ⚠ open |
| 8 | Professional prior-art search, CPC H04L 9/3247, H04L 9/003, G06F 7/72 | ⚠ open |

**Assessment.** D1, D2, D3 and D4 are structural, are not disclosed in P1's specification or drawings, and do not depend on the reduced-modulus concept. They constitute a filable case on their own. D5 depends on the outcome of item 3. D6 is a dependent claim only.

The case is narrower than originally conceived — the reduced-modulus concept is conceded to P1 — but it is directed to circuit structure that no identified reference discloses, and it does not rest on a change of implementation medium or on an effect without a corresponding structural difference.
