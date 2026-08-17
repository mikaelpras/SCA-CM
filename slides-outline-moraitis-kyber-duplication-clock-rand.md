# Slide Outline — Securing CRYSTALS-Kyber in FPGA: Duplication + Clock Randomization
**Moraitis, Ji, Brisfors, Dubrova (KTH) · Lindskog, Englund (Ericsson) · IEEE Design & Test 2024 · ~8 minutes · 8 slides**

Standalone presentation. Opens with a summary of the paper, per format.
**Note:** a *countermeasure* paper, hiding branch — structure follows the countermeasure template.

---

## Slide 1 — What This Paper Is About (60 s)

**Title:** Securing CRYSTALS-Kyber in FPGA Using Duplication and Clock Randomization
Moraitis, Ji, Brisfors, Dubrova (KTH) + Lindskog, Englund (Ericsson Research) — IEEE Design & Test, Oct 2024

- A **hiding** countermeasure imported from block ciphers: **two Kyber cores**, each on its **own randomized clock**
- Applied to **Kyber768**, built on the most efficient public FPGA implementation (Xing & Li, TCHES 2021)
- Claim: **resistant to deep-learning power analysis** that breaks the unprotected version

**Four claimed advantages over masking:**
**universal coverage · glitch immunity · zero clock-cycle overhead · resists repeated measurements**

---

## Slide 2 — Why This Class of Countermeasure (60 s)

Each piece is known-broken **on its own** — the claim is that together they fix each other.

| Approach | Status |
|---|---|
| **Masking in hardware** | Glitches recombine shares; partial masking leaves a boundary to audit |
| **Clock randomization alone** | **Broken by these same authors** (FPS 2022): AES subkey from **<500 traces** — oversample, resync in pre-processing, attack the *start* of encryption |
| **Duplication alone** | Known inadequate — secret key extracted from duplicated AES FPGA |

> The bet: duplication adds an uncorrelated power consumer; clock randomization stops the attacker separating the two.

---

## Slide 3 — The Architecture ★ (75 s)

- **Randomized clock generator** + **two Kyber cores**: a **primary** and a **dummy**
- Each core driven by a **different** randomized clock, through its own clock buffer
- Both cores get **identical input data** — but **different secret/public key pairs**

**Why different keys?** Same key + same data ⇒ correlated traces ⇒ duplication adds noise but no confusion. Different keys make the dummy's power genuinely independent of the target secret.

**The clock generator:**
- MMCM generates base frequencies; a LUT-based **4-to-1 MUX selects asynchronously** among four
- Asynchronous ⇒ varying **frequency *and* duty cycle** (pulse-width variation is part of what kills alignment)
- Simulation: **≥403 distinct frequencies**; RNG produces a fresh value every base clock cycle

---

## Slide 4 — Evaluation (45 s)

- **Platform:** Artix-7 XC7A100T on a **CW305** board
- **Three builds compared:** **Ku** (unprotected) · **Kr** (clock randomization only) · **Kdr** (duplication + randomization)
- **Baseline attack:** Ji & Dubrova DL power analysis recovers a message from the unprotected core in **5120 traces**
  → in Kyber, message recovery ⇒ **shared key recovery** (key is hashed from the message)
- Same FPGA for profiling and attack — the *attacker-favourable* choice, so a negative result carries weight

**Result:** Kdr resists the DL analysis that breaks Ku.

---

## Slide 5 — What It Costs (60 s)

| | |
|---|---|
| **Area** | **~2×** from duplication |
| **Cycles** | **Zero** overhead — the headline claim |
| **Wall-clock time** | **Not** zero — randomized clock has a lower *average* frequency |
| **Power / energy** | Roughly double — two cores doing real work |
| **Key management** | The dummy needs its own **real** key pair |

**Competing proposal for context:** Xu, Wang & Tian's Fisher–Yates shuffling architecture claims **8.7%** efficiency degradation — and explicitly criticises this line as *"high resource overhead."*

---

## Slide 6 — The Lineage ★★ (90 s — the payoff slide)

**Largely the same authors, four steps:**

| Year | What happened |
|---|---|
| **2022** | They **break** plain clock randomization on AES — <500 traces |
| **2023** | They **fix** it by adding duplication (AES, ISCAS) |
| **2023/24** | They **apply** the fix to Kyber768 → **this paper** |
| **2024** | They **break it again** — two DL attacks exploiting **sporadic synchronicity** |

**Sporadic synchronicity:** the two randomized clocks occasionally drift into momentary alignment. In those windows the trace looks single-core. A DL attacker who can *detect* those windows gets clean data.

> **Randomness does not guarantee non-coincidence.**

Three modular fixes were proposed to restore security — **demonstrated on AES**. Whether the Kyber design here is affected, and whether the fixes were applied to it, is **an open question**.

---

## Slide 7 — Critique (60 s)

- **"Resistant to DL-SCA" is an absence of a successful attack, not a proof.** Hiding has no probing-model equivalent — the noise floor is measured, not derived. And the sibling design fell within a year.
- **The fault-attack claim deserves scrutiny.** The editor's note frames this as covering fault attacks too — but duplication detects faults by **comparing identical computations**, and these two cores run **different keys**, so their outputs can't be compared.
- **Narrow threat model.** Profiled power analysis only. Says nothing about PC-oracle / chosen-ciphertext attacks or SASCA on the NTT.
- **The dummy core isn't free noise** — it's a real core running a real key. Either that key matters (second attack surface) or it doesn't (pure area cost).
- Kyber768 only, one FPGA family, one base implementation.

---

## Slide 8 — Takeaways for Hardware Design (60 s)

- **No masked/unmasked boundary to audit.** Universal coverage is a real architectural property — and the boundary is exactly where partial-masking arguments fail.
- **The area maths changes for ASIC IP.** 2× on an FPGA prototype ≠ 2× on a licensed block customers count gates for.
- **Randomized clocking is awkward to ship.** MMCM is FPGA-specific; in ASIC IP you hand your customer a timing-closure and integration problem.
- **Quantify accidental alignment, don't assert independence.** Any hiding scheme resting on two random processes staying out of phase needs a *rate* of coincidence, not just an independence claim.
- **For ML-DSA:** algorithm-agnostic so it ports — but the rejection-sampling loop makes runtime variable, so the two cores' relative timing becomes data-dependent in ways this analysis doesn't cover.

---

## Cut List (in the deep dive, not the slides)

- MMCM / clock-buffer implementation detail beyond the 4-to-1 MUX and 403 figure
- The Vivado place-and-route caveat on measuring duplication area
- Kr (randomization-only) intermediate results — mention only if the paper gives numbers
- Full threat-model table — Slide 4's baseline attack line covers it
- The Xing & Li base implementation's own architecture
- Detailed mechanics of the 2022 FPS attack (oversampling, resync) — one clause on Slide 2 is enough

## Timing Check
60 + 60 + 75 + 45 + 60 + 90 + 60 + 60 s ≈ **8.5 min**.

**Protect Slides 3 and 6.** Slide 3 makes the mechanism legible — especially *why different keys*, which is the non-obvious design decision. Slide 6 is the story: a countermeasure broken, fixed, deployed, and broken again by the same group. If you run long, fold Slide 7's last two bullets into Slide 6.
