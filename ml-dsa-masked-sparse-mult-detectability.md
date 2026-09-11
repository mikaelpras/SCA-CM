# Method to Verify Application of the Invention in Products

**Invention:** Masked sparse polynomial multiplication circuit with minimal-ring datapath and carry-sign modulus conversion for ML-DSA (FIPS 204) signature generation

---

## 0. Summary Assessment

Detectability for this invention is **moderate**. It is a masked cryptographic datapath internal to a secure IP block, so it is not exposed at any product interface and is not mandated or described by any public standard. However, three characteristics of the invention produce unusually clear observable signatures:

1. **The absence of a modular multiplier** is a physical fact visible in die analysis and in synthesis reports.
2. **Latency proportional to τ rather than to n log n** is measurable externally without any disassembly.
3. **Masked coefficient storage at 8, 9 or 19 bits rather than 23 or 24 bits** is visible in memory macro geometry.

Each is independently observable, and together they form a distinctive fingerprint. The detection routes below are ordered from lowest to highest cost.

---

## 1. Public Certification and Security Documentation — lowest cost, no disassembly

The strongest and cheapest route for a competitor product, because these documents are published and are often architecturally specific.

| Source | What to look for |
|---|---|
| **Common Criteria Security Target and Certification Report** (published on the Common Criteria portal and by the certifying scheme) | The countermeasure description for the signature engine. Security Targets for secure elements frequently name the masking order achieved, whether order is configurable, and the arithmetic domain of the countermeasure. A claim of configurable masking order on a fixed signature datapath is a direct indicator. |
| **CC Site / ETR excerpts and AVA_VAN.5 evaluation summaries** | Statements about resistance to higher-order analysis and the mechanism relied upon. |
| **FIPS 140-3 Non-Proprietary Security Policy** (published on the NIST CMVP validated modules list) | Module block diagrams and algorithm implementation descriptions. Security Policies are mandatory public documents and sometimes include datapath-level detail. |
| **Protection Profile conformance claims** (e.g. for secure elements, eUICC, TPM) | Indicates the attack-potential tier targeted, hence the masking order likely implemented. |
| **NIST CAVP / ACVP algorithm validation entries** | Confirms ML-DSA implementation and parameter sets supported; establishes the target for further analysis. |

**Action:** maintain a watch on CC and CMVP listings for competitor secure elements and automotive HSM cores claiming ML-DSA with high-order masking, and retrieve the Security Target for each.

## 2. Competitor Technical Publications and Patent Filings — low cost

| Source | What to look for |
|---|---|
| **Competitor patent applications and grants** | Claims and figures disclosing sparse indexed accumulation on shares, two-axis parallelism, per-operand ring width, or carry-sign conversion. Continuations of the NXP family (US 12,362,931; 12,368,581; 12,388,657) are the highest-priority watch. |
| **Conference papers and posters** by competitor design teams (CHES/TCHES, HOST, DATE, ISSCC, VLSI Symposium) | Architecture block diagrams, cycle counts, area breakdowns. Silicon teams routinely publish datapath diagrams of exactly this granularity. |
| **IP vendor product briefs and integration guides** | Licensable PQC IP datasheets often state cycle counts per signature, configurable masking order, and area at each order. |

**Action:** set standing alerts on CPC classes H04L 9/3247, H04L 9/003 and G06F 7/72 combined with ML-DSA/Dilithium terms, and on the inventor names in the NXP family.

## 3. External Timing and Cycle-Count Analysis — low cost, no disassembly

This is the most powerful non-invasive route, because the invention's dataflow determines externally measurable latency.

**Signature latency signature.** An NTT-based masked multiplication requires work proportional to d·n·log n per product, with a fixed cycle count independent of the challenge. Table-driven sparse accumulation requires work proportional to the number of non-zero challenge entries. Consequently:

- **Cycle count scales with τ, not with n log n.** Comparing measured signing latency across ML-DSA-44 (τ=39), ML-DSA-65 (τ=49) and ML-DSA-87 (τ=60) on the same device should show latency tracking τ if the invention is applied, and tracking n log n and the module dimensions if an NTT is used.
- **Absence of a fixed transform cost.** An NTT-based multiplier contributes a constant, parameter-set-independent block of cycles; sparse accumulation does not.
- **Latency scaling with masking order.** Where units are instantiated along the share axis, raising the order should leave latency approximately flat while area rises. Where a single unit is time-shared across share domains, latency rises linearly. Either behaviour distinguishes the invention from a design in which order and schedule are entangled.

**Measurement method:** drive the signature interface with inputs producing known challenges, and measure with a logic analyser or on-chip cycle counter. No decapsulation required. For a device exposing configurable masking order, repeat at each setting.

⚠ Note: the rejection loop makes total signing latency variable. Measurement must be per-iteration or averaged over many signatures with the iteration count recovered from the trace, or the τ relationship will be masked by loop-count variance.

## 4. Side-Channel Trace Structure Analysis — moderate cost

Power or electromagnetic traces reveal the dataflow structure even when the values are protected. Masking hides data, not structure.

| Observable | Indicates |
|---|---|
| **Repeating pattern with period corresponding to one tap**, repeated τ times per product | Tap-first accumulation over a sparse entry table (§3.3 of the invention description) |
| **Number of concurrent activity regions in the trace** | Degree of parallelism on each axis; counting them recovers the number of sparse multiplication units and accumulator units |
| **One-cycle offset between the activity signatures of successive share domains within a single tap** | Same-tap temporal share separation (§3.4) — a distinctive and deliberate timing artefact not present in DOM-style spatial separation |
| **Absence of a multiplier activation signature** | No modular multiplier in the datapath; multiplier arrays produce a characteristic high-amplitude, data-dependent power pattern |
| **A short, narrow-width conversion phase between accumulation and the mod-q output** | Carry-sign conversion at reduced width rather than a full-modulus A2B gadget |
| **Two distinct accumulation phase widths within one signing operation** | Per-operand ring width — the t₀ path at M′=19 versus the s-paths at M=8 or 9 |

**Method:** standard SCA acquisition setup (oscilloscope, near-field probe or shunt resistor). Structural analysis requires no key recovery and no leakage — trace alignment and pattern identification suffice.

## 5. Die Analysis, Delayering and Cross-Section — high cost, strongest evidence

| Observable | Method | Indicates |
|---|---|---|
| **Absence of a modular multiplier array** in the signature engine | Optical die photo of the delayered metal stack; multiplier arrays are visually distinctive regular structures | The central structural feature of claim 1 |
| **Memory macro bit width of 8, 9 or 19 bits** rather than 23 or 24 | Memory macro geometry from die photo; bitcell array dimensions and column count are directly countable | Per-operand ring width (§3.6) — the presence of **two** macros at different widths within one engine is highly specific |
| **Replicated identical accumulation blocks**, count matching the masking order | Layout floorplan; identical hard macros or synthesized blocks in a regular array | Share-axis parallelism (§3.1, §3.5) |
| **A second array of accumulator blocks orthogonal to the first** | Floorplan and interconnect routing between the two arrays | Two-axis organization — this is the signature of the invention and is not produced by any conventional masked design |
| **Address-generation logic adjacent to a small table store** | Block identification and net tracing | Table-driven (index, value) addressing (§3.2) |
| **Pipeline register placement between share-domain accumulation stages** | Netlist extraction | Same-tap separation (§3.4) |

**Method:** decapsulation, delayering, optical and SEM imaging, and for definitive evidence, netlist extraction from the imaged layers. Commercially available from reverse-engineering service providers. Cost is significant but standard for infringement assessment.

**Cross-sectional photography** contributes mainly by establishing memory macro bitcell dimensions and confirming the number of metal layers over each block; the plan-view delayered images carry most of the structural evidence.

## 6. Internal Verification — Samsung Application

For confirming that Samsung products implement the invention, the following are the documents of record and should be retained as evidence of practice:

- **RTL source** for the masked sparse multiplication and accumulator modules, with module and port names traceable to the claim elements
- **Microarchitecture specification** containing the block diagram of the two-axis structure, the entry table format, and the accumulation schedule
- **Parameter or configuration file** setting masking order and ring width per operand class — direct evidence of claims 2, 4 and 5
- **Synthesis reports** showing no modular multiplier instance in the signature datapath, and area figures at each masking order
- **Verification plan and coverage reports** for the masked multiplication block
- **TVLA and formal leakage-assessment reports**, evidencing the same-tap separation mechanism
- **Design review minutes** recording the architectural decisions

**Recommendation:** ensure module, signal and parameter names in the RTL correspond recognizably to the claim terminology. This materially reduces the cost of demonstrating practice later, and is a common omission.

---

## 7. Detection Difficulty — Candid Note for the Reviewer

It should be recorded that this invention is **harder to detect in a competitor product than a typical interface-level or protocol-level invention**, for three reasons:

1. It is internal to a secure IP block with no externally exposed interface.
2. No public standard mandates or describes the countermeasure architecture, so standard documents provide no evidence.
3. Secure element vendors deliberately limit architectural disclosure, and some certification documentation is withheld from publication.

Against this, the routes in Sections 1, 3 and 5 are practicable, and the two-axis floorplan signature in Section 5 is sufficiently unusual that it would constitute strong evidence if observed. The τ-proportional latency relationship in Section 3 is the most cost-effective screening test and can be applied to any competitor device without disassembly.

**Suggested detection sequence:** monitor CC/CMVP publications and competitor patent filings (Sections 1–2, ongoing, negligible cost) → apply the τ-scaling latency test to any candidate device (Section 3, low cost) → acquire structural side-channel traces if the latency test is positive (Section 4) → commission die analysis only where Sections 3 and 4 both indicate application (Section 5).
