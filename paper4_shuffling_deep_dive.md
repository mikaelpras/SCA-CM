# Paper 4 Deep Dive — A Hardware-Friendly Shuffling Countermeasure for Kyber

Dejun Xu, Kai Wang, Jing Tian — School of Integrated Circuits, **Nanjing University**.
**IEEE TCAS-II: Express Briefs**, Vol. 72, No. 3, March 2025, pp. 504–508. Preprint: arXiv:2407.02452.

**Source note.** I verified every technical claim below against the published text (arXiv v3, which matches the TCAS-II version) rather than relying on the secondary study guide. Equations (3)–(5), Algorithm 1, Table I, Table II and all experimental numbers are confirmed verbatim. Material marked **[derived]** is my own computation and is not in the paper.

---

## 1. What this paper is about

**The problem.** Kyber's hardware implementations leak. Zhao et al. (TCAS-I 2023) identified three side-channel leakage points in the decryption datapath and broke them with CPA. The two existing defence families both have a cost problem: **masking** on hardware is enormous (Kamucheka et al.: ~1700× the area-time product), and existing **hiding** schemes either burn clock cycles or burn area.

**What they do.** Take an existing open-source unprotected Kyber FPGA design (Xing & Li, TCHES 2021) and bolt on a **shuffling** countermeasure. Rather than restructuring the datapath, they **permute the memory addresses** the datapath visits. Two new modules: a Random Permutation Generator (RPG) and an Address Controller (ADDR).

**What they find.** CPA that succeeded at 4×10³ traces on the unprotected design finds no peak at 10⁵ traces. TVLA that failed at 10⁴ traces passes at 10⁷. Cost: **+260 slices, +8.7 % ATP, zero extra clock cycles, zero extra DSPs or BRAMs.**

**Why it matters.** It is a genuinely cheap hardware SCA countermeasure, and the cost comes from one good architectural decision: **shuffle addresses, not arithmetic.**

> **One-sentence thesis:** if you permute *which memory location* each operation touches, you get shuffling for the price of a mux on the address bus — the datapath never knows it happened.

---

## 2. Protection point

**All of Kyber's decryption datapath**, via one mechanism.

The decryption core, equation (1):

```
m ← Compress_q( v − INTT( ŝᵀ ∘ NTT(u) ), 1 )
```

The three leakage points from Zhao et al., equation (2):

| Point | Expression | Datapath stage |
|---|---|---|
| 1 | `ŝᵀ ∘ û` | **PWM** — the multiplier |
| 2 | `(ŝᵀ ∘ û) mod q` | modular reduction after PWM |
| 3 | `v − (sᵀu mod q)` | subtraction after INTT |

Plus a **fourth** the authors add on their own initiative: **INTT**, protected by shuffling the butterfly units, because Hamburg et al. (TCHES 2021) attacked INTT in *software*.

**The structural point worth a slide:** all four are protected by the *same* mechanism — permuting memory addresses. That is why the marginal cost is one RPG plus some muxes, rather than four bespoke defences.

**Why decryption and not encryption/keygen:** the secret `ŝ` only appears in equation (1).

---

## 3. Key idea

Fisher–Yates produces a uniform random permutation:

```
for i = n−1 down to 1:
    j ← uniform random in [0, i]
    swap a[i], a[j]
```

**Two things make this awkward in hardware, and the paper names both:**

1. **Continuous random seed supply.** You need n−1 fresh random numbers. A TRNG fast enough to feed one per cycle is expensive, and TRNG bandwidth is itself an attack surface.
2. **Dynamically shrinking index range.** `j` must be uniform in `[0, i]` with `i` changing every iteration. Uniform reduction to an arbitrary range needs a divider/modulo (expensive) or rejection sampling (variable latency — poison for a fixed-schedule datapath).

The paper's answer to each is the contribution. Their own framing (Table I) is three criteria that a hardware shuffler should meet:

| Criterion | Why it matters | TECS [10] | TCAD [13] | INDOCRYPT [14] | **This work** |
|---|---|---|---|---|---|
| Huge permutation space | small sets can be enumerated | ✗ | ✗ | ✓ | ✓ |
| No dynamic input required | TRNG bandwidth is costly | ✗ | ✗ | ✗ | ✓ |
| No pre-storage required | storage dominates area | ✗ | ✗ | ✓ | ✓ |

---

## 4. Protection method — the four moving parts

### 4.1 Solving problem 1 — LFSR + FIFO batching

- A **32-bit LFSR** emits 1 bit/cycle; a 6-bit buffer collects six bits into one index.
- At initialization the LFSR runs **6 × 64 = 384 cycles**, filling a **64-deep × 6-bit FIFO** with *every* index the permutation will need, up front.
- An external **TRNG** reseeds the LFSR via `rst_l` — a one-shot seed, not a stream. This is what earns the "No Dynamic Input Required" ✓.

**The elegant bit:** the FIFO is *reused*. As random indices shift out of `FIFO_out`, the selected permutation elements are pushed into `FIFO_in`. One out, one in — occupancy stays at 64, and a single 64×6 memory serves both roles. No second buffer.

### 4.2 Solving problem 2 — cheap range reduction

Equation (4), stage 1 (12 rounds):

```
idx_0 = idx           if idx ≤ 0x28
      = idx − 0x28    if idx > 0x28
```

Equation (5), stage 2 (52 rounds):

```
idx_1 = idx           if idx ≤ rest
      = idx & rest    if idx > rest      ← bitwise AND, not modulo
```

`rest` = (elements still to select) − 1, from a decrement counter. No divider, no rejection sampling, **fixed latency**. The paper's own worked example: `rest = 0x02`, `idx = 0x05` → `0x05 & 0x02 = 0x00`, so register 0 is selected and backfilled from register 2.

### 4.3 The address-conflict hazard and the two-stage fix

**The hazard.** The Xing & Li datapath is pipelined: a read at cycle *t* produces a write-back at *t*+12 (`addr_r12`) or *t*+1 (`addr_r1`). Apply a naive random permutation to the read sequence and certain adjacent addresses collide — read-after-write on the same RAM location — and **the computation is wrong**. Fig. 1(a) marks the first six and last six positions as error-prone.

**The fix.** Introduce a *designed* intermediate permutation split into two regions, equation (3):

```
per_a = {0x33, 0x32, …, 0x0c, 0x0b}                          41 elements
per_b = {0x34…0x38} ∪ {0x00…0x0a} ∪ {0x39…0x3f}              23 elements
```

The 12 risky slots must satisfy conditions of the form `> 0x00, > 0x02, > 0x04, > 0x06, > 0x08, > 0x0a` and `< 0x34, < 0x36, < 0x38, < 0x3a, < 0x3c, < 0x3e`. (The paper says "only needs to restrict those 11 elements" — `> 0x00` is trivially satisfied by everything but 0x00.)

**The insight:** `min(per_a) = 0x0b > 0x0a` and `max(per_a) = 0x33 < 0x34`, so **every element of per_a satisfies every constraint simultaneously.** Fill the risky slots exclusively from per_a and no conflict can occur — by construction, with no runtime check.

**The procedure:**
1. **Stage 1 (`shuffling_12`)** — select 12 addresses from the per_a region, place at the rightmost 12 positions; backfill vacated slots from per_b.
2. **Stage 2 (`shuffling_52`)** — select the remaining 52 from what's left.
3. **Cyclic shift right by 6** — so the 12 constrained elements land on the six leading and six trailing risky positions.

### 4.4 RPG microarchitecture (Fig. 2)

| Part | Contents | Role |
|---|---|---|
| (a) | 64-to-1 MUX0, 1-to-64 DEMUX, REG (6b × 64) | holds the working permutation |
| (b) | 3-to-1 MUX, FIFO (6 × 64) | dual-purpose buffer; `ena` = input mode, `enb` = cyclic mode |
| (c) | MUX2, two 2-to-1 MUXs, two comparators, counter, subtractor, AND gate | implements eq. (4)/(5), produces `idx′` |
| (d) | 32-bit LFSR + 6-bit buffer | serial-to-parallel index generation |
| (e) | CTRL | sequencer for `ena`, `sel_0`, `sel_1`, `finish` |

**Why it's small:** one register file, one FIFO, an LFSR, some muxes and comparators. No multiplier, no divider, no permutation storage.

**[derived] Cycle budget:** 384 (init) + 12 + 52 + 6 = **~454 cycles**, against 6,700 (Kyber512) or 10,000 (Kyber768) for a decapsulation. That, plus a shorter critical path, is why the paper's zero-overhead claim holds — and Table II confirms identical cycles and identical 206 MHz.

### 4.5 Address Controller (Fig. 3)

| Part | Role |
|---|---|
| (a) | 6 × 2-to-1 MUXs swap in shuffled addresses (4 read + 2 write). `repl_x = 0` passes the original through — shuffling is selectively disableable |
| (b) | 4 shift registers producing 1-cycle and 12-cycle delayed permutations, matching pipeline latency so writes use the same permuted address the read used |
| (c) | Mux tree over FSM `state` generating `repl0…repl5` |
| (d) | Concatenation logic building `addr0…addr5` |

**Five permutations from one.** The design needs permutations of `00–3f`, `40–7f`, `00–7f`, `80–bf`, `c0–ff`. Instead of five generators:
- `addr0 = {line, addr}` with `line ∈ {0..3}` → the four 6-bit ranges
- `addr2/4/5 = {addr, 1'b0}` or `{addr, 1'b1}` → the 7-bit `00–7f` range

`enb` is asserted every other cycle for PWM and subtraction, so one new address is consumed per 2 cycles.

---

## 5. Results

### Security (Fig. 4) — Kyber768 on Artix-7 XC7A100T, **EM** traces, Hamming-distance model on multiplier output registers (deliberately mirroring Zhao et al.'s methodology)

| Test | Unprotected | Protected | Ratio |
|---|---|---|---|
| CPA on PWM | key distinguishable at **4×10³** | no peak at **1×10⁵** | ≥25× |
| TVLA | \|t\| ≫ 4.5 at **1×10⁴** | within ±4.5 at **1×10⁷** | ≥1000× |

CPA/TVLA on the modular reduction and subtraction were "omitted for simplicity — same countermeasure, similar conclusions."

### Area and time (Table II) — all arithmetic verified **[derived]**

```
ENS = #Slices + 100×#DSPs + 200×#BRAMs
ATP = ENS × Cycles × (1/Frequency) × 10³
```

| Design | LUTs | FFs | Slices | DSP | BRAM | ENS | Cycles | MHz | ATP |
|---|---|---|---|---|---|---|---|---|---|
| Xing [11], unprotected, K512 | 7353 | 4633 | 2173 | 2 | 3 | 2973 | 6.7k | 206 | **96.7** |
| **This work, K512** | 8143 | 5151 | 2433 | 2 | 3 | 3233 | 6.7k | 206 | **105.2** |
| Kamucheka [8], *masked* | 143112 | – | 81746 | 60 | 294 | 146546 | 126.6k | 100 | **186×10³** |
| Jati [10], hiding | 7151 | 3730 | 2260 | 2 | 5.5 | 3560 | 57.2k | 258 | **789.3** |
| Xing [11], unprotected, K768 | 7353 | 4633 | 2173 | 2 | 3 | 2973 | 10.0k | 206 | **144.3** |
| Moraitis [9], hiding, K768 | 14341 | 9190 | 4734 | ≥2 | 6 | ≥6134 | 10.0k | ≤206 | **≥297.8** |
| **This work, K768** | 8143 | 5151 | 2433 | 2 | 3 | 3233 | 10.0k | 206 | **156.9** |

Checks: `2433 + 200 + 600 = 3233` ✓ · `3233 × 6700/206e6 × 10³ = 105.15` ✓ · `3233/2973 = 1.0874` → **+8.7 %** ✓ · `156.9/144.3 = 1.0873` ✓

**[derived] Overheads the paper doesn't break out:**

| Resource | Δ | % |
|---|---|---|
| LUTs | +790 | **+10.7 %** |
| FFs | +518 | **+11.2 %** |
| Slices | +260 | **+12.0 %** |
| DSP / BRAM | 0 | 0 % |
| ENS | +260 | **+8.7 %** |
| Cycles, f_max | 0 | 0 % |

**The 8.7 % headline is metric-dependent.** ENS adds a constant 800 (`100×2 + 200×3`) to both designs, and the countermeasure touches neither DSPs nor BRAMs — so the denominator is inflated and the percentage diluted. On **slices alone**, the resource actually consumed, it is **12 %**. Both are defensible; know which you're quoting.

**Also note:** Kyber512 and Kyber768 have *identical* area rows because Xing & Li's design is runtime-parameterised — k only changes the cycle count.

**Comparison caveats:** Kamucheka is *masking* on a *Virtex-7* — a qualitatively stronger security claim, so the 1700× gap isn't apples-to-apples. Jati's slices are *estimated* via `0.25×LUTs + 0.125×FFs`, not measured. The authors re-synthesised the baseline on XC7A100T instead of the original XC7A12T — a genuinely good methodological choice.

---

## 6. Independent analysis — [derived]

Three things I checked that the paper doesn't quantify.

### 6.1 The permutation entropy budget

| Quantity | Value |
|---|---|
| `log₂(64!)` — a truly uniform 64-permutation | **296.0 bits** |
| Entropy actually injected by the 64 index draws, assuming *perfect* random input | **260.1 bits** |
| Ceiling imposed by the **32-bit LFSR** | **32 bits** |

Two independent losses. The reduction functions cost ~36 bits even with ideal randomness. But the binding constraint is the LFSR: all 384 index bits are a **linear function of 32 state bits**, so the entire permutation carries at most 32 bits of entropy regardless of how large 64! is. An adversary who recovers ~32 bits of LFSR output can predict every subsequent permutation and undo the shuffle completely. **The reseeding policy is never specified**, which is the crux — if the TRNG supplies a fresh 32-bit seed per decapsulation, 2³² is still a long way below 296 bits but the attack becomes a per-trace problem rather than a one-shot break.

### 6.2 Equation (5) is exactly uniform — but only on powers of two

This is sharper than "the AND is biased." Measured distribution of `idx_1` over uniform `idx ∈ [0,63]`:

| `rest` | distribution | entropy | uniform would be |
|---|---|---|---|
| 2 | {0: 31, 1: **1**, 2: 32} | 1.100 bits | 1.585 |
| 3 | {0:16, 1:16, 2:16, 3:16} | **2.000** bits | 2.000 |
| 7 | uniform, 8 each | **3.000** bits | 3.000 |

When `rest + 1` is a power of two, `idx & rest` **is** `idx mod (rest+1)`, which is perfectly uniform for uniform `idx`. Otherwise it collapses badly — at `rest = 2`, index 1 is reachable only through the `idx ≤ rest` branch, with probability **1/64** against 32/64 for index 2.

**[derived] Only 6 of the 52 stage-2 rounds** (`rest ∈ {0,1,3,7,15,31}`) are unbiased. The other 46 are skewed. A designer who restructured the shuffle to operate on power-of-two ranges would get uniformity **for free**, with no extra hardware.

### 6.3 The "by construction" guarantee does not hold as written

The paper's safety argument is that per_a elements satisfy all 11 constraints, so filling the 12 risky slots from per_a is safe. But **Algorithm 1 line 13** backfills the vacated slot with `REG[rest]`, where `rest` counts 63 → 52 during stage 1. Those registers hold **per_b** values.

I simulated Algorithm 1 directly (200,000 runs, uniform random indices):

```
REG[52..63] = 06 05 04 03 02 01 00 38 37 36 35 34     ← all per_b
```

| Result | Value |
|---|---|
| Runs where ≥1 stage-1 selection is **not** from per_a | **86.0 %** |
| Mean contaminants per run | **1.62** |
| Values that leak in | `0x00`–`0x05` and `0x34`–`0x38` |
| Most frequent | `0x34` (25.6 % of runs), `0x35` (23.8 %), `0x00` (15.1 %) |

The contaminating values are exactly those straddling the per_a boundary — and exactly the ones the constraints are designed to exclude. `0x34` violates `< 0x34`; `0x00` violates `> 0x00`.

**What this means.** Either the shipped HDL has a guard the brief omits, or the slot-to-constraint mapping happens to tolerate these particular values, or there is a real correctness hazard. The brief never formalises the constraints — they appear only as annotations in Fig. 1(c) — so it cannot be settled from the text. **If you were evaluating this design for adoption, this is the first thing to check against the RTL.** It is also a fair question to raise in the seminar: the security argument is empirical, but the *correctness* argument is claimed to be structural, and the structural argument has a gap.

---

## 7. Advantages / disadvantages

| Advantages | Disadvantages |
|---|---|
| **+8.7 % ATP, +260 slices** — an order of magnitude cheaper than the hiding alternatives | **32-bit LFSR caps permutation entropy at 32 bits** regardless of the "huge space" claim; reseeding policy unspecified |
| **Zero clock-cycle overhead, zero DSP/BRAM overhead** — RPG runs in parallel with a shorter critical path | **Biased shuffle** — 46 of 52 stage-2 rounds are non-uniform; never analysed |
| **One mechanism covers all four leakage points** | **Only CPA and TVLA tested.** No second-order, integrated, or permutation-recovery attack — the attacks that actually matter against hiding |
| **Shuffles addresses, not the datapath** — the arithmetic is untouched | **"No peak at 10⁵" is where they stopped**, not a bound. Theory for N=64 predicts ~N² = 4096×; the experiment only demonstrates 25× |
| Aggressive reuse: one FIFO for two roles, one 6-bit permutation for five address ranges | **Derived permutations are not independent** — a 128-element shuffle built as `{π(i), b}` has 64!·structure, not 128! |
| Two-stage construction turns a runtime hazard into a static guarantee, nearly free | **The static guarantee has a gap** (§6.3) — 86 % of runs violate the stated invariant |
| Honest baseline re-synthesis on a matched FPGA part | **N = 64** is modest granularity; the paper never justifies it |
| | **No FIA protection** — acknowledged in §VI |
| | **Hiding, not masking** — heuristic, no security order, doesn't compose into a proof |

---

## 8. Can the idea be applied in our IP? — **Yes, with three substitutions**

This is the only paper of the set that is actually RTL, and the central architectural idea transfers directly.

**What transfers well**

1. **Shuffle addresses, not arithmetic.** The whole reason the cost is 8.7 % is that the datapath is never restructured — a mux on the address bus and a delay-matched copy for the write-back. In any memory-based PQC accelerator (BRAM/SRAM banked coefficient storage), this is the same two lines of RTL. It is the cheapest structural SCA lever in the literature for hardware.
2. **Parallel generation with a delay-matched write path.** The `addr_r1`/`addr_r12` shift registers that keep write-back aligned with the permuted read address are the non-obvious engineering detail, and it is exactly the detail you'd have to rediscover. Worth copying.
3. **Selective disable** (`repl_x = 0` passes the original address) is good design hygiene — lets you A/B the countermeasure on the same silicon during characterisation.
4. **Composability.** Shuffling is orthogonal to masking. Shuffling N slots multiplies the attack cost on a d-order masked design; the two are meant to be stacked, not chosen between.

**Three things I would change before adopting**

1. **Replace the LFSR.** A 32-bit linear generator is not acceptable for an IP — 384 output bits from 32 state bits, and linearity means an attacker who profiles once predicts forever. Substitute a lightweight DRBG. If a hash core is already on the die, a Keccak-based DRBG is close to free; otherwise Trivium is small. This changes the entropy story from "32 bits, linearly recoverable" to "as many bits as you reseed with."
2. **Restructure to power-of-two ranges.** Per §6.2, `idx & rest` is *exactly* uniform when `rest+1` is a power of two. Shuffling in blocks of 2^m gives an unbiased permutation with the same `AND`-gate cost and no divider — strictly better than what the paper does, for zero area.
3. **Resolve the backfill hazard** (§6.3) against real HDL before trusting the "no runtime check needed" claim, or add a cheap guard.

**Two caveats specific to ASIC**

- **Do not quote 8.7 % as an ASIC number.** A 64-to-1 mux plus 1-to-64 demux over 6-bit words plus a 64×6 register file is cheap in FPGA LUTs, where wide muxes map efficiently onto LUT trees and carry chains. In a standard-cell flow that same structure is a real mux tree (~380 2:1 mux-equivalents) plus 384 flops. Still small in absolute terms, but the FPGA slice ratio does not translate. Re-estimate against your own area budget.
- **The RPG's 454-cycle latency must be hidden.** On this design it's trivial (454 vs 6.7k). If your IP has a much shorter operation, or does back-to-back operations without gaps, you need either a second RPG to double-buffer permutations or a faster index path.

**Relevance beyond Kyber:** the authors' own §VI notes the method applies to any NTT-based scheme and names **Dilithium and Raccoon** as future targets. The address-permutation approach is agnostic to what the datapath computes — it only cares that coefficients live in addressable memory and that operations on distinct indices are independent.

---

## 9. Errata

| Location | Issue |
|---|---|
| **Algorithm 1, lines 11 and 17** | Ternary branches are **inverted** relative to equations (4) and (5). Line 11 reads `idx_0 ← FIFO_out > 0x28 ? FIFO_out : FIFO_out − 0x28`, but eq. (4) puts the *subtraction* on the `> 0x28` branch. Same for line 17 vs eq. (5). The paper's own worked example (`rest = 0x02`, `idx = 0x05 > rest` → `0x05 & 0x02`) confirms the **equations** are correct. Trust eq. (4)/(5), not the pseudocode |
| §III-A | Says restrictions apply to "the first six and last six elements" (12 slots) but then "only needs to restrict those **11** elements". Reconcilable — `> 0x00` is trivial — but stated without explanation |
| Eq. (3) vs Alg. 1 line 1 | per_a is written descending `{33, 32, …, 0b}` in eq. (3) but ascending `{0b, 0c, …, 33}` in REG. Cosmetic |
| §III-A safety argument | The "filled exclusively from per_a" invariant is contradicted by the line-13 backfill in 86 % of runs (§6.3) |

---

## 10. Slide skeleton (~8 min)

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** — cheap hardware shuffling for Kyber | 0:45 |
| 2 | Eq. (1) decryption pipeline + the 3 leakage points, plus INTT | 1:00 |
| 3 | Hiding vs. masking, and the cost table that motivates hiding | 0:45 |
| 4 | Fisher–Yates and its two hardware problems | 0:45 |
| 5 | Fix 1: LFSR + batched FIFO, and the one-out-one-in reuse trick | 1:00 |
| 6 | Fix 2: eq. (4)/(5) — reduction without divider or rejection sampling | 0:45 |
| 7 | The address-conflict hazard and the two-stage per_a / per_b fix | 1:15 |
| 8 | Results: CPA 25×, TVLA 1000×, **+8.7 % ATP / 0 cycles** | 1:00 |
| 9 | Advantages / disadvantages + applicability verdict | 0:45 |

**Land this line:** *"The countermeasure never touches the arithmetic. It just lies to the memory about which coefficient you asked for."*

---

## 11. Self-check exercises

1. Why is Kyber's *decryption* the SCA target rather than encryption or key generation? *(The secret `ŝ` only appears in eq. (1).)*
2. Which fourth operation does this paper protect that Zhao et al. did not list, and why? *(INTT — because Hamburg et al. broke it in software, so the authors treat it as a potential hardware leakage point too.)*
3. State the two hardware problems with Fisher–Yates and the fix for each. *(Continuous random supply → batch all 384 bits into the FIFO up front from one TRNG seed. Shrinking index range → eq. (4)/(5) arithmetic reduction instead of modulo or rejection sampling.)*
4. Why does `per_a = {0x0b … 0x33}` satisfy all eleven constraints automatically? *(The tightest are `> 0x0a` and `< 0x34`; min(per_a) = 0x0b and max(per_a) = 0x33.)*
5. Why does the protection add zero clock cycles? *(RPG runs in parallel, needs ~454 cycles against 6.7k–10k for decapsulation, and has a shorter critical path — Table II confirms identical cycles and frequency.)*
6. Compute the ENS of the protected Kyber512 design. *(2433 + 100×2 + 200×3 = 3233.)*
7. Why is the ATP overhead (8.7 %) smaller than the slice overhead (12 %)? *(ENS adds a constant 800 for DSPs and BRAMs to both designs, and the countermeasure uses neither — inflating the denominator.)*
8. Theory says shuffling N slots raises the trace requirement by ~N². For N = 64 that's ~4096×. Why does the paper only demonstrate 25×? *(10⁵ traces is where they stopped collecting. It's a lower bound on effort, not a security level.)*
9. Give two reasons to treat the "huge permutation space" claim cautiously. *(All 384 index bits derive linearly from a 32-bit LFSR, capping entropy at 32 bits; and the reduction functions are biased, so even the reachable permutations are non-uniform. A third: the five address permutations are deterministic relabellings of one 64-element permutation.)*
10. Under what condition is `idx & rest` an *unbiased* reduction, and how could a designer exploit that? *(When `rest+1` is a power of two, the AND is exactly `mod (rest+1)`. Restructuring the shuffle to power-of-two block sizes gives uniformity at zero extra hardware cost.)*
