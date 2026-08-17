# Deep Dive: Securing CRYSTALS-Kyber in FPGA Using Duplication and Clock Randomization

**Authors:** Michail Moraitis, Yanning Ji, Martin Brisfors, Elena Dubrova (KTH Royal Institute of Technology) · Niklas Lindskog, Håkan Englund (Ericsson Research)
**Venue:** IEEE Design & Test, vol. 41, no. 5, pp. 7–16, October 2024 (early access 2023)
**DOI:** 10.1109/MDAT.2023.3298805 · Preprint: KTH DiVA `diva2:1846186`
**Editor's note** (Swarup Bhunia, U. Florida): presented as a lightweight countermeasure targeting **both** side-channel and fault attacks.

> **Sourcing note.** The IEEE version is paywalled and the KTH DiVA full text blocks automated access. This deep dive is built from the published abstract, extensive retrievable passages of the DiVA preprint (architecture, clock generator, evaluation setup), and the surrounding literature — including two follow-up papers by the same group. §9 flags what to verify.

---

## 1. Summary — What the Paper Is About

A **hiding** countermeasure for Kyber in FPGA, imported wholesale from the block-cipher world: run **two cores** — a primary and a dummy — each driven by its **own randomized clock**.

The pitch is a direct attack on masking's weak points:

| Claimed advantage | Why it matters |
|---|---|
| **Universal coverage** | Protects the *whole* design, not selected operations. No "which layers do we mask?" question. |
| **Glitch immunity** | Hiding in the time domain isn't defeated by combinational glitches, which plague hardware masking. |
| **Zero clock cycle overhead** | The algorithm runs in exactly the same number of cycles. |
| **Resistance to repeated measurements** | Averaging across traces doesn't align, so multi-trace attacks degrade. |

The authors take the **most efficient publicly available Kyber FPGA implementation** (Xing & Li, TCHES 2021), protect Kyber768, and show the result **resists deep-learning-based side-channel analysis** that breaks the unprotected version.

**The interesting thing about this paper is its lineage** — the same group broke plain clock randomization two years earlier, added duplication to fix it, applied that to Kyber here, and then found new attacks on the AES version afterwards. See §7.

---

## 2. Why This Countermeasure Class At All

The paper sits in a specific gap in the hardware Kyber literature.

**The problem with masking in hardware:** glitches. Combinational logic transiently evaluates on intermediate values, recombining shares in ways the mathematical masking argument doesn't capture. Threshold implementations and DOM address this at real cost. And the partial-masking compromise — mask some operations, argue search space for the rest — is exactly the pattern the Adams Bridge analysis pulls apart.

**The problem with clock randomization alone:** the same authors killed it. Brisfors, Moraitis & Dubrova (FPS 2022), *"Do Not Rely on Clock Randomization,"* recovered an AES subkey from **fewer than 500 traces** by sampling at a frequency far above the encryption clock, resynchronising traces in pre-processing, and targeting the **beginning** of the encryption — where clock drift hasn't yet accumulated.

**The problem with duplication alone:** also known inadequate. Prior work extracted the secret key from a duplicated AES FPGA implementation. Duplication is a *fault-detection* technique; on its own it does little against SCA.

**The claim of this line of work:** the two together fix each other. Duplication supplies a second, uncorrelated power consumer; clock randomization desynchronises them so the attacker can't separate the two cores' contributions.

---

## 3. The Architecture

### Top level
- A **randomized clock generator**
- **Two CRYSTALS-Kyber cores** — a **primary** and a **dummy**
- Each core driven by a **different** randomized clock, via its own clock buffer

### The critical design decisions
- Both cores receive **identical input data**
- But they use **different secret and public key pairs**
- Kyber768 (k = 3) is the demonstrated target; other parameter sets said to follow similarly

### The randomized clock generator
Inherited from the authors' AES design (ISCAS 2023):
- An **MMCM** (Mixed-Mode Clock Manager) produces several base frequencies
- A **LUT-based 4-to-1 MUX** selects **asynchronously** among four base frequencies
- Asynchronous selection means the output has **varying frequency *and* varying duty cycle** — this matters, because pulse-width variation is part of what destroys trace alignment
- Simulation indicates **at least 403 distinct frequencies** can be generated this way
- The RNG produces a fresh random output **every clock cycle** of the base frequency

**Why the dummy core carries a different key:** if both cores ran the same key on the same data, their power traces would be correlated and duplication would add noise but no confusion. Different keys make the dummy's contribution genuinely independent of the target secret.

---

## 4. Threat Model

| Aspect | Detail |
|---|---|
| Attack defended against | **Profiled, deep-learning-based power analysis** |
| Baseline attack | Ji & Dubrova's DL power analysis on the unprotected Xing–Li Kyber768 — message recovery from **5120 traces** with enumeration |
| Why message recovery matters | In Kyber KEM the shared key is derived from the message by hashing, so **message recovery ⇒ shared key recovery**. With chosen ciphertexts it extends toward secret key recovery. |
| Attacker capability assumed | Profiling access to an identical device, power measurement, DL training |
| Not addressed | Chosen-ciphertext oracle attacks, SASCA/BP, fault injection (see §8) |

---

## 5. Evaluation Setup

- **FPGA:** Xilinx Artix-7 XC7A100T-2FTG256 on a **CW305** target board
- **Three implementations compared:**
  - **Ku** — unprotected
  - **Kr** — randomized clock only
  - **Kdr** — duplication + randomized clock
- Bitstreams built with Vivado, default synthesis options
- **Same FPGA device used for profiling and attack.** Their justification: an attack that fails on the profiling board is unlikely to succeed on a different board — i.e. it's the *attacker-favourable* choice, so a negative result is meaningful.

**Note on area measurement:** Vivado may synthesize, place, and route the primary and dummy cores **differently**, so the area overhead of duplication is not cleanly 2×.

**Result:** Kdr is resistant to the deep-learning-based analysis that succeeds against Ku.

---

## 6. What This Buys and What It Costs

### Buys
- Coverage of the entire design — no per-operation masking decisions, no boundary to audit
- No glitch-based masking failures
- No added clock cycles
- Degraded multi-trace averaging

### Costs
- **Roughly 2× area** from duplication. A later paper (Xu, Wang & Tian, TCAS-II) explicitly criticises this line as **"high resource overhead"**, comparing it against their Fisher–Yates shuffling architecture at **8.7%** efficiency degradation.
- **Key management burden** — the dummy core needs its own real key pair, generated and stored.
- **Zero *cycle* overhead ≠ zero *time* overhead.** A randomized clock has a lower average frequency than a fixed clock at the same peak. Wall-clock throughput drops even though cycle count doesn't.
- **Power and energy roughly double** — two cores running real Kyber operations.

---

## 7. The Lineage — and the Follow-Up That Complicates It ★

The most interesting material for discussion. This countermeasure has an unusually well-documented arc, largely written by the same people:

| Year | Work | What happened |
|---|---|---|
| 2022 | Brisfors, Moraitis & Dubrova, FPS — *Do Not Rely on Clock Randomization* | **Broke** plain clock randomization on AES: <500 traces via high-frequency sampling + resync + attacking the start of encryption |
| 2023 | Moraitis et al., ISCAS | **Fixed** it by adding duplication — AES |
| 2023/24 | **This paper**, IEEE D&T | **Applied** the fixed version to Kyber768 |
| 2024 | Brisfors & Moraitis et al. (Springer) | **Broke it again** — two DL attacks on the clock-randomized *and* duplicated AES, exploiting **sporadic synchronicity** between the two cores' execution. Proposed **three modular additions** to restore security. |

> **The 2024 finding is the key discussion point.** The two randomized clocks occasionally drift into momentary alignment. When they do, the trace for that window looks like a single-core trace, and a DL attacker who can *detect* those windows gets clean data. Randomness doesn't guarantee non-coincidence.

**Open question for the seminar:** the sporadic-synchronicity attack was demonstrated on the **AES** instantiation. Whether the Kyber design here is vulnerable — and whether the three proposed fixes have been applied to it — is not something I could establish from available sources. That's a genuinely good question to raise rather than a gap to paper over.

---

## 8. Critical Points for Discussion

- **"Resistant to DL-based SCA" is an absence of a successful attack, not a proof.** Hiding countermeasures have no security argument comparable to masking's probing model — they raise the noise floor by an amount that is measured, not derived. And the same group's own follow-up found new attacks on the sibling design within a year.
- **The fault-attack claim deserves scrutiny.** The editor's note frames this as protecting against side-channel *and* fault attacks. But duplication detects faults by **comparing two identical computations** — and here the two cores run **different key pairs**, so their outputs cannot be compared. Worth checking whether the paper actually claims fault detection or whether that's the editor's framing.
- **~2× area is a serious price** in a field where the competing shuffling proposal claims 8.7%. Whether "universal coverage" justifies it is a real engineering argument, not a settled one.
- **Same-board profiling and attack.** Their justification is sound for a *negative* result, but it means the evaluation doesn't speak to cross-device portability of the attack, which is where DL SCA has been improving fastest.
- **The dummy core is not free noise — it's a real Kyber core doing real work on a real key.** If that key ever matters, it's a second attack surface. If it doesn't, it's pure area cost.
- **Threat model is narrow.** This addresses profiled power analysis. It says nothing about the PC-oracle / chosen-ciphertext class or SASCA on the NTT — both of which appear elsewhere in this seminar and neither of which is obviously stopped by desynchronisation.
- **Evaluated on Kyber768 only**, on one FPGA family, from one base implementation.

---

## 9. What to Verify From the Paper

- [ ] Concrete **area/LUT/FF/DSP overhead** figures for Kdr vs Ku — the abstract doesn't give them
- [ ] The **actual throughput** impact — average effective frequency vs the unprotected fixed clock
- [ ] How many traces the DL attack was given against Kdr before being declared unsuccessful (5120? more?)
- [ ] Whether **TVLA / t-test** leakage assessment was performed, or only attack-based evaluation
- [ ] Whether the paper claims **fault** resistance, and if so how, given the different-key design
- [ ] Kr results — how much does clock randomization alone buy on Kyber, before duplication?
- [ ] Whether the three modular fixes from the 2024 AES follow-up apply here

**Access:** IEEE Xplore doc 10193851, or the KTH DiVA record `diva2:1846186` (preprint full text).

---

## 10. Transfer to Hardware IP

**1. This is the "hiding" branch of the design space, and it's worth knowing why you'd pick it.**
Everything else in this seminar's countermeasure discussion is masking or shuffling. Duplication + clock randomization occupies a different corner: it makes no per-operation decisions, so it has **no masked/unmasked boundary to audit**. Given that the boundary is exactly where the Adams Bridge argument fails, "no boundary" is a genuine architectural property, not just marketing.

**2. The area maths changes for ASIC IP.**
2× area on an FPGA prototype is one thing; 2× on a licensed ASIC block is a much harder sell, and customers count gates. Meanwhile the *shuffling* alternative at ~8.7% looks better on the datasheet but reintroduces the "how much entropy does your shuffler actually have?" question the Adams Bridge audit makes so sharp. Neither is free.

**3. Randomized clocking is awkward in an IP deliverable.**
An MMCM-based randomized clock generator is an FPGA-specific construct. In ASIC IP you'd need an equivalent (ring-oscillator-based, or a clock chopper), and — more importantly — a randomized clock domain is a **timing-closure and integration problem** you're handing to your customer. Soft IP that demands an unusual clocking scheme is harder to sell than IP that doesn't.

**4. "Randomness doesn't guarantee non-coincidence" is the transferable lesson.**
The 2024 sporadic-synchronicity attack is a general warning for any hiding scheme built on two independent random processes. If your protection relies on two things being out of phase, the attacker's job is to find the windows where they aren't — and with enough traces, those windows exist. Any hiding argument should quantify the *rate* of accidental alignment, not just assert independence.

**5. For ML-DSA specifically:** the countermeasure is algorithm-agnostic by construction, so it ports. But the cost profile is worse — Dilithium's rejection-sampling loop means variable-length operations, and duplicating a core whose runtime varies per signature makes the two cores' relative timing data-dependent in ways this design's analysis doesn't cover. Worth thinking through before assuming it transfers cleanly from a KEM.

---

## 11. Key References to Follow Up

- **Xing & Li — *A Compact Hardware Implementation of CCA-Secure Key Exchange Mechanism CRYSTALS-Kyber on FPGA*, TCHES 2021** (the base implementation being protected)
- **Brisfors, Moraitis & Dubrova — *Do Not Rely on Clock Randomization*, FPS 2022** (the attack that motivated adding duplication)
- Moraitis, Brisfors, Dubrova, Lindskog & Englund — *A Side-Channel Resistant Implementation of AES Combining Clock Randomization with Duplication*, ISCAS 2023 (the AES original)
- **Brisfors & Moraitis et al. (2024)** — the sporadic-synchronicity attacks on the combined countermeasure, plus three proposed fixes (**read this one**)
- Ji & Dubrova — *A Side-Channel Attack on a Hardware Implementation of CRYSTALS-Kyber*, 2023 (the 5120-trace baseline attack)
- Xu, Wang & Tian — *A Hardware-Friendly Shuffling Countermeasure Against Side-Channel Attacks for Kyber*, IEEE TCAS-II (the 8.7% competitor and explicit critique)
- Nikova, Rechberger & Rijmen — Threshold Implementations (the glitch-resistant masking alternative)
- Hettwer et al. — *Lightweight Side-Channel Protection Using Dynamic Clock Randomization*, FPL 2020
