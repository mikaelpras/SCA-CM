# Deep Dive: Partial Number Theoretic Transform Masking in PQC Hardware — A Security Margin Analysis

**Authors:** Ray Iskander (Verdict Security), Khaled Kirah (Faculty of Engineering, Ain Shams University, Cairo)
**Venue:** arXiv:2604.03813 [cs.CR] — v1 4 Apr 2026, v2 17 Apr 2026 · 38 pages, 2 figures
**Archive:** Zenodo 10.5281/zenodo.19508454 · Code: github.com/rayiskander2406/ntt-security-margins-arXiv-2604.03813
**Status:** Preprint, **not peer reviewed**. Paper 2 of the seven-paper "QANARY" program.

> **Sourcing note.** The arXiv PDF has no machine-readable text layer, so the full body was not extractable. This deep dive is built from the complete abstract, the retrievable §1 and §3 (architecture and security claims), and detailed descriptions of this paper's findings in the authors' four sibling papers. §10 flags what to verify.

---

## 1. Summary — What the Paper Is About

**This is a red-team audit of a specific, real, open-source hardware accelerator.** Not a generic attack, not a generic countermeasure — an evaluation of whether one shipping design's published security claims hold up.

**The target:** **Adams Bridge**, the PQC accelerator built for the **Caliptra silicon root of trust** — an open-source hardware security framework backed by AMD, Google, Microsoft and NVIDIA. It implements ML-DSA (FIPS 204) and ML-KEM (FIPS 203).

**The design pattern under scrutiny — "partial masking":** fully masking every NTT operation costs too much silicon, so mask *some* operations with Boolean masking and argue that the *rest* are safe on search-space and shuffling grounds. Adams Bridge masks **the first INTT layer** and the point-wise multiplication, leaving layers 2–8 (ML-DSA) / 2–7 (ML-KEM) unmasked. The designers claim per-butterfly CPA complexities of **2⁴⁶ (ML-DSA)** and **2⁹⁶ (ML-KEM)**.

**The paper's verdict, in one line:** the 2⁴⁶ number is correct arithmetic for **the wrong threat model**, and the shuffling that was supposed to backstop it delivers **6 bits of entropy, not 296**.

**And the constructive payoff:** masking **3 consecutive mid-layers** — 43% of the cost of full masking — creates an information gap that soft-analytical attacks cannot bridge.

> The authors are explicit that this is **not** a demonstrated key recovery on silicon. See §9.

---

## 2. The Target Architecture (know this before anything else)

| Property | Detail |
|---|---|
| Scale | 30 synthesizable modules, **~1.17 million gate-equivalent cells** |
| Butterflies | **Gentleman–Sande** for INTT, **Cooley–Tukey** for NTT |
| ML-DSA | q = **8,380,417** (23-bit coefficients), **8** NTT layers, n = 256, Montgomery reduction |
| ML-KEM | q = **3,329** (12-bit coefficients), **7** NTT layers, Barrett reduction |
| Masking | First-order Boolean, **DOM-style, two shares**, on the **first INTT layer** + point-wise multiplication |
| Licence | Apache 2.0, RTL publicly available (CHIPS Alliance) |
| History | Caliptra 2.0 (Oct 2024): ML-DSA-87 only. Adams Bridge 2.0 / Caliptra 2.1 (Oct 2025): + ML-KEM-1024 |

**The critical structural detail:** after the masked first layer, **the two shares are recombined into a single unmasked intermediate value** before processing continues through the remaining layers.

The designers do not hide this. Their own documentation states that **"leakage naturally emerges in the unmasked layers."** The disagreement is about whether that leakage is exploitable, not whether it exists.

---

## 3. The Designers' Security Argument

1. An attacker of the unmasked layers must hypothesise **multiple coefficients simultaneously** — 2 × 23-bit for ML-DSA, 8 × 12-bit for ML-KEM.
2. That gives per-butterfly CPA complexity of **2⁴⁶** and **2⁹⁶** respectively — computationally infeasible.
3. **RSI (Random Start Index) shuffling** of butterfly execution order provides an additional combinatorial barrier on top.

Their scaling argument treats the shuffling contribution as approaching that of a **full random permutation — 296 bits**.

---

## 4. Finding 1 — The Shuffling Entropy Gap ★ (RTL, high confidence)

The headline result, and the one that requires no attack at all — just reading the RTL.

**From `ntt_ctrl.sv`, lines 648–653:**
- A **chunk-level RSI** selects among **16 starting positions** within a 16-element chunk
- An **index-within-chunk** selection provides an additional factor of **4**
- Combined: **16 × 4 = 64 orderings per layer**
- **log₂(64) = 6 bits of entropy per layer**

**Not 296 bits. Six.**

**Why it's so low:** processing proceeds **sequentially with wraparound within each chunk**. This is a *rotation*, not a permutation. A rotation of 16 elements has 16 orderings; a permutation of 16 elements has 16! ≈ 2⁴⁴. The design chose the cheap one — which is the sensible hardware choice — but the security argument was written as though it had bought the expensive one.

→ **Effective margins fall below the designers' estimates.**

---

## 5. Finding 2 — The Attack-Model Gap (the conceptual core)

This is the part worth understanding properly, because it generalises far beyond Adams Bridge.

| | **Per-butterfly CPA** (the designers' model) | **SASCA / Belief Propagation** (the actual threat) |
|---|---|---|
| Unit of attack | each butterfly, **independently** | the **entire NTT factor graph, simultaneously** |
| Hypothesis space | exponential **per butterfly pair** | joint inference across all layers |
| Exploits | one butterfly's leakage | **inter-butterfly dependencies** |
| Shuffling | multiplies the search space | **absorbed** — Hermelink et al.'s shuffle nodes resolve permutation *and* secret jointly |

So 2⁴⁶ is arithmetically correct **for a threat model that isn't the relevant one**. The whole point of BP-based attacks is that the NTT's algebraic structure links butterflies together, and that structure is exactly what a per-butterfly independence assumption throws away.

The designers' argument also implicitly assumes a **non-profiled** attacker. Profiled and algebraic attacks reduce the effective search space.

---

## 6. Findings 3–6 — The Experimental Tracks

Seven analysis tracks in total, with **confidence-rated evidence** (an unusual and welcome methodological choice — each claim is tagged with how strongly it's supported).

### Soft-analytical attack pipeline
Demonstrates a **37-bit enumeration reduction**, independent of any BP gains. Quantifies the attack-model gap concretely — **explicitly without achieving key recovery.**

### Full-scale BP on the complete INTT factor graph
Achieves **100% coefficient recovery**, versus the single-layer baseline. This **resolves an open question**: prior BP results were small-scale or simulated, and it was genuinely unclear whether BP gains survive at production NTT sizes. They do.

### Genie-aided information-theoretic bound
Observations contain **sufficient mutual information for full recovery at SNR×N as low as 15**. A possibility bound, not an achievable attack — see §9.

### Layer-ablation analysis
Identifies **four necessary conditions** governing BP convergence.

---

## 7. Finding 7 — Observation Topology, Not Count ★★

The most interesting and most useful result in the paper.

> **4 evenly spread layers → 100% recovery.**
> **4 consecutive layers → 0% recovery.**

**Same number of observed layers. Completely opposite outcomes.**

Security is determined by **where** the observable layers sit, not **how many** there are. BP propagates information across the graph; evenly spread observations let it stitch the whole thing together, while a contiguous observed block leaves an unbridgeable gap elsewhere.

### The constructive recommendation
**Mask 3 consecutive mid-layers.**

- Creates an **unrecoverable gap** that defeats soft-analytical attacks
- Costs **43% of full masking overhead**
- Contrast with Adams Bridge's current choice: masking layer 1 — the *edge* of the graph, the worst possible placement, because it leaves every remaining layer contiguous and observable

This is a genuine design rule, not just a criticism: *if you can only afford partial masking, put it in a contiguous mid-block, not at the boundary.*

---

## 8. Convergent Independent Analyses

The paper positions itself as the third of three independent analyses reaching the same architectural conclusion:

| Analysis | Method | Finding |
|---|---|---|
| **Karabulut & Azarderakhsh** (FAU) | Empirical CPA on silicon | Practical exploitability of the ML-DSA pipeline within their trace budget (Caliptra 2.0) |
| **Saarinen** | RTL review, pre-silicon | Partial masking on the ML-DSA signing path doesn't meet sufficient conditions — **the key is not arithmetically share-split** |
| **Iskander & Kirah** (Papers 1 & 2) | Structural dependency analysis + BP | **8,487 convergence points** across 30 modules; **165 INSECURE_CONSERVATIVE wires** in the Barrett module alone; absence of fresh inter-stage masking |

**Two distinct failure modes**, as characterised across their series:
- **Intra-stage:** within each stage, both shares interact through combinational logic **without fresh intermediate masking** — share₀ and share₁ paths converge.
- **Inter-stage:** `masking_en_ctrl = 1` only during `rounds_count == 0` (`ntt_ctrl.sv:264–272`). **Rounds 1–3 run with no fresh per-stage masks.** Worst-case max-multiplicity becomes the *product* of per-stage parameters — up to 2ʳ — rather than the max.

**The designers have responded** in a concurrent paper (Bisheh-Niasar, Karabulut, Upadhyayula, Norris & Pillilli, IACR ePrint 2026/256). Worth reading both sides before presenting conclusions as settled.

---

## 9. Critical Points for Discussion

- **No key recovery on real silicon.** The BP result runs on the factor graph; the SASCA pipeline explicitly stops short of key recovery. This is an *analysis of claims*, not a demonstrated break. The authors are honest about it — the confidence-rating methodology exists precisely for this — but the distinction should be stated plainly rather than glossed.
- **Preprint, not peer reviewed.** Part of a rapid seven-paper series by the same two authors, several listed as "manuscript under preparation," with heavy internal cross-citation. That's not disqualifying, but it means the findings haven't been through external review.
- **Declared interest.** The lead author is at **Verdict Security**, and QANARY appears to be their verification tooling. The analysis is technically substantive; note the commercial context.
- **Genie-aided bounds flatter the attacker.** SNR×N ≥ 15 says information is *present*, not that an attacker can extract it. It's an upper bound on the defender's security, not a lower bound on the attack.
- **The RSI finding is the most robust claim in the paper** — it's a direct RTL reading of publicly available source, independently checkable, and requires no attack model or simulation to accept. If you present one thing as settled, present this.
- **The 43% recommendation depends on the same simulation framework** as the attack claims. Strong engineering intuition, but not independently validated.
- **This is a live dispute.** Adams Bridge is deployed and its designers disagree. Present it as an open technical argument, not a verdict.

---

## 10. What to Verify From the PDF

- [ ] The seven analysis tracks named individually, with their confidence ratings
- [ ] The **four necessary conditions for BP convergence** from the layer-ablation study
- [ ] The exact noise/SNR model — simulated leakage or measured traces?
- [ ] How the 37-bit enumeration reduction is computed, and against which baseline
- [ ] The derivation of "43% overhead" — area, cycles, or randomness cost?
- [ ] Whether the 3-mid-layer recommendation is validated for both ML-DSA (8 layers) and ML-KEM (7 layers)
- [ ] The designers' counterarguments as characterised in §5.1

**Access:** arXiv:2604.03813 (v2 recommended), Zenodo 10.5281/zenodo.19508454, GitHub repo with artifacts.

---

## 11. Transfer to Hardware IP

**This is the most directly relevant paper in the entire seminar to your work**, and it's worth flagging why to the audience.

**1. Partial masking is the exact commercial temptation your IP faces.**
Full NTT masking costs area. Mask the first layer, argue search space for the rest, ship. Adams Bridge is the reference open-source ML-DSA accelerator, backed by four major vendors, and it made that call. This paper is the argument against it — and the reason "we mask the first layer" won't survive scrutiny in a security review.

**2. The layer-placement rule is directly usable.**
*Mask a contiguous mid-block, not the boundary.* Three consecutive mid-layers at 43% of full-masking cost beats one boundary layer at ~12% and beats full masking on cost. That's a defensible architecture decision with a stated rationale — exactly what an IP security argument needs. Masking layer 1 is the **worst** placement for a fixed budget, because it leaves everything else contiguous.

**3. Count your shuffler's actual orderings.**
The RSI finding generalises to any hardware shuffling scheme. A randomised start index with sequential wraparound is a **rotation**, and rotations give log₂(n) bits where permutations give log₂(n!). 6 bits versus 296 is a ~50× gap in the exponent. If you implement address randomisation and call it shuffling, enumerate the orderings in RTL before putting a number in a datasheet.

**4. State your threat model, or your complexity figure is meaningless.**
2⁴⁶ per-butterfly CPA is a real number under a per-butterfly independence assumption. Against SASCA it's not the relevant quantity at all. Any security claim your IP makes should name the attack class it holds against — a bare complexity figure invites exactly this kind of audit.

**5. Share recombination between masked and unmasked regions is the structural sin.**
Adams Bridge collapses two shares to one unmasked intermediate at the masked/unmasked boundary. That boundary is where the whole argument fails. Same principle as the Kennaway paper's `tmp2`, one abstraction level up: **the point where protection ends is the point to audit.**

**6. Sparse coefficient-domain multiplication sidesteps the factor graph.**
No butterfly network, no twiddle constants, nothing for BP to run inference over. Genuine structural advantage for `c·s₁` — but ML-DSA still needs the NTT for **A**·**y** and related products, so the placement rule above applies to whatever NTT you do instantiate.

**7. FIPS 140-3 relevance.** Paper 1 of this series is explicitly about producing pre-silicon SCA evidence for certification. If your IP needs certifiable side-channel evidence, this line of work is the tooling conversation you'll be having.

---

## 12. Key References to Follow Up

- **Bisheh-Niasar, Karabulut, Upadhyayula, Norris & Pillilli — *Adams Bridge Accelerator: Bridging the Post-Quantum Transition*, IACR ePrint 2026/256** (the designers' side — read this alongside)
- CHIPS Alliance — Adams Bridge RTL, github.com/chipsalliance/adams-bridge (Apache 2.0; the `ntt_ctrl.sv` claims are checkable)
- Iskander & Kirah — *Structural Dependency Analysis for Masked NTT Hardware*, arXiv:2604.15249 (**Paper 1** — the 8,487 convergence points)
- Iskander & Kirah — *Prime-Field PINI*, arXiv:2604.25878, and *Machine-Checked Cardinality Bounds for Masked Barrett Reduction* ("the 1-Bit Barrier"), arXiv:2604.24670 (the formal follow-ups)
- **Hermelink, Streit, Strieder & Thieme — *Adapting Belief Propagation to Counter Shuffling of NTTs*, TCHES 2023** (the shuffle-node technique this paper relies on — and the same paper that appears in your NTT countermeasures deep dive)
- Karabulut, M. & Azarderakhsh — empirical CPA on Adams Bridge (the silicon confirmation)
- Saarinen — RTL review of Adams Bridge partial masking
- Primas, Pessl & Mangard, CHES 2017 / Pessl & Primas, LATINCRYPT 2019 (the SASCA foundations)
- Gross, Mangard & Korak — Domain-Oriented Masking (the DOM style Adams Bridge uses)
