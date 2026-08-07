# Seminar Essentials — PQ-Hammer & Paper 4 (8 minutes each)

Same format as the P1 / P2 / P6 briefs. Each paper standalone, each opening with a summary slide.
Presented in seminar order: Paper 3 (attack) then Paper 4 (countermeasure).

---
---

# PAPER 3 — PQ-Hammer: End-to-end Key Recovery on PQC Using Rowhammer

Amer, Wang, Kippen, Dang, Genkin, Kwong, Nelson, Yerukhimovich — **IEEE S&P (Oakland) 2025**, pp. 3567–3582
Free copy: `par.nsf.gov/servlets/purl/10584474` · Code: `github.com/pqrowhammer/pqhammer`

## SLIDE 1 — What this paper is about

**The problem.** Every other physical attack on PQC assumes access to the board — probes, oscilloscopes, EM antennas. Rowhammer assumes none of that: an **unprivileged process on the same machine** can flip bits in another process's DRAM just by reading memory rapidly. Can that alone recover a PQC secret key?

**What they do.** Three attacks on three NIST schemes, all built from one primitive — bit flips:

| Scheme | Vector | Mechanism |
|---|---|---|
| **Kyber (ML-KEM)** | **data** flip in KeyGen | poison `e` → boost decryption failures → recover key from failures |
| **Dilithium, deterministic** | **data** flip on ρ | nonce reuse → algebraic key recovery |
| **Dilithium, randomized** | **instruction** flip | force deterministic keygen |
| **BIKE** | **instruction** flip | zero the RNG seed → deterministic keypair |

**What they find.** Full **long-term** key recovery on all of them, on commodity desktops, no elevated privileges, no supercomputer.

**Why it matters.** Prior PQC Rowhammer work got only *session* keys (FrodoFlips) or *partial* key info (SignatureCorrection). This is the first end-to-end long-term recovery. And it makes deterministic Dilithium a liability on any shared machine.

> **One-sentence thesis:** no probe on the board — a co-located process and a DDR4 stick are enough to take the whole key.

## 1. Existing method it builds on

| Prior work | What it did | Limitation |
|---|---|---|
| **FrodoFlips** (Fahr et al., CCS'22) | Rowhammer FrodoKEM KeyGen, failure attack | Needed **8 exact bits** (managed 7); **665,000** failing ciphertexts requiring a supercomputer; only **session** keys |
| **SignatureCorrection** (Islam et al., EuroS&P'22) | Fault attack on Dilithium | **Partial** key info only — lowered security, no recovery |
| **This paper** | End-to-end recovery on 3 schemes | **2 bits**, ~40k ciphertexts, personal computer |

Rowhammer toolchain assembled from prior art: **many-sided / multi-bank hammering** (TRRespass, Sledgehammer), **HyperDegrade** (performance degradation), **SPOILER** (contiguous memory), **DRAMA** (bank address bits), **Frame Feng Shui** (RAMBleed).

## 2. Attack model

- **No physical access.** Unprivileged user executing code on a machine **co-located** with the victim
- Machine's DRAM susceptible to Rowhammer-induced flips
- **No elevated privileges** for any step — degradation, cache flushing, profiling, massaging
- ASLR and DRAM refresh timing left at **defaults**
- Active **fault injection**, not passive observation

## 3. Attack points

| Scheme | Target | Where |
|---|---|---|
| **Kyber** | error vector **`e`** | between `poly_getnoise_eta1` and `polyvec_ntt(&e)` — i.e. *during* `polyvec_ntt(&skpv)` |
| **BIKE** | `mov` operand feeding `get_seeds` | **instruction** memory, via the shared OS page cache |
| **Dilithium (det.)** | public seed **ρ** | first 32 bytes of the sk buffer, page-aligned |
| **Dilithium (rand.)** | `shake256` length argument | instruction memory in `crypto_sign_keypair` |

**Why that window in Kyber, and no other:** before sampling the arrays are contiguous zeros (flips get overwritten); after NTT, producing a *controlled* amount of Euclidean-domain noise would need many coordinated flips in the NTT domain. Only the gap between them works.

**Why Kyber-1024 is the easiest target:** k = 4 means 8 `poly_ntt` calls (~35,000 cycles each) — the **longest** available window. Attacking the strongest parameter set is easier here.

## 4. Attack procedure (Kyber)

1. **Profile memory** — hammer your own address space until you find a page with flips at the right page offset and direction
2. **Massage** — Frame Feng Shui via the FILO page frame cache, so the victim's `e` lands on your vulnerable page
3. **Degrade** — flush 1 cacheline in `montgomery_reduce` and 2 in `ntt`; pin flushers to the victim's **sibling core** to induce machine clears
4. **Hammer** — multi-bank, 10 aggressors × 2 banks, striped 1-0-1, `clflushopt`
5. **Flip two bits** — at 2⁸ and 2⁶ in two coefficients of `e`
6. **Generate failing ciphertexts with boosting** — resample `r` until the two coefficients multiplying the poisoned ones are both **±η₁ with the same sign**
7. **Recover** — estimate which coefficient failed (~75 % accurate), rotate, average; **James–Stein** estimator for the scaling factor

**The key mechanism, on one slide.** Coefficients of `e` live in **[−2, 2]** but are stored in **16-bit integers**:

| Value | int16 | set bit 8 | set bit 6 |
|---|---|---|---|
| 0 | `0x0000` | **256** | **64** |
| +2 | `0x0002` | **258** | **66** |

Too few flips → failure rate stays negligible. Too many → the keypair looks broken. **Two is the sweet spot.**

**Why the recovery works:** each decryption failure is a *hint* that the attacker's known vector points roughly along the secret. Formally `E[W] = (E[α′]/ℓ)·S` — the mean of the failing ciphertexts' known vectors is **parallel to the secret**.

**Honest caveat the paper states:** with `p′ = 2⁻¹¹` and `p = 2⁻¹⁸`, **~67 % of observed failures come from some other coefficient**, not the poisoned one. That's why an estimator is needed instead of arithmetic.

## 5. Target device & results

Intel i7-8700K (Coffee Lake) · **Samsung DDR4 8 GiB** M378A1K43BB1-CPB · Ubuntu 23.04 · Kyber-1024 **reference** code

| Metric | ASLR **off** | ASLR **on** |
|---|---|---|
| Page massaged | 91/100 | 44/100 |
| Correct bits (and only those) flipped | **8/100** | **3/100** |

- Profiling **49 s**; massaging + hammering **93 s** ⟹ ~142 s per attempt
- ~**40,000** failing ciphertexts, ~2 h to generate
- Key extraction **4.5 min/iteration**; 1 iteration (ASLR off), avg 512 (ASLR on, known bit separation), up to **~60 core-months** (unknown separation)
- **Stealth:** Geekbench −8 % single-core, −71 % multi-core, but *"the machine was snappy and reactive"*

**Numbers to quote carefully.** The decryption failure rate appears three ways — 2⁻¹⁰ (analysis), 2⁻¹⁰·⁵ (experiment), 1/10,000 (§4.5, §7.1) — the last being 6.9× off. And §7.1's "over 99.9999 % detection" from 100k test ciphertexts is really **99.9955 %** at a 1/10,000 rate. **Quote the raw outcomes: 8/100, 3/100, 142 s.**

## 6. What must be protected

1. **Key material in a vulnerable representation** during KeyGen — minimize the window
2. **The storage layout of small-range values** — this is the real enabler
3. **ρ in Dilithium** — a *public* value that nonetheless needs integrity protection
4. **The RNG seed path** — a single call to system RNG is a single point of failure

## 7. Possible countermeasures

| Countermeasure | Verdict |
|---|---|
| **Bit-pack the error terms** | **Best.** Makes the attack *arithmetically impossible* — the poisoned value has nowhere to live. 68.75 % of coefficients are non-negative and fit in 2 bits, leaving 14 flippable bits each |
| **Key auditing** | Generate test ciphertexts; reject any key producing failures. Detects, doesn't prevent |
| **Redundant copy / hash of key material** | Detects tampering; good for long-lived material |
| **Reduce exposure window** | Mitigates; vectorized code shortens it |
| **Don't rely on a single RNG call**; check seed Hamming weight | Blocks the instruction-flip variants |
| Counter-based mitigations (Graphene, Hydra, START) | Effective but SRAM-tracker cost grows as thresholds fall |
| **TRR** (DDR4) | Largely ineffective against many-sided hammering |
| **ECC memory** | Raises the bar; does **not** guarantee protection |
| **Hardware-accelerated crypto** | Paper's own point — removes the instruction-cache degradation lever entirely |

### Hardware-IP note

**A hardware crypto IP with on-chip key storage is not a Rowhammer target** — the primitive is a DRAM disturbance effect; keys in registers, SRAM, or OTP are out of reach. The paper itself lists hardware acceleration as a defence.

Four things transfer anyway:
- **Bit-packing is free in RTL.** You choose the storage width. Store an [−2,2] coefficient in 3 bits and 256 is unrepresentable
- **Integrity protection on key registers** — parity/CRC checked before use; buys resistance to fault injection generally
- **Key auditing as a keygen self-test** — cheap when encapsulation is fast in hardware
- **For ML-DSA: prefer the hedged/randomized mode**, and treat ρ as integrity-critical storage even though it's public

**Where the surface returns:** if the IP fetches key material from **external DRAM**, or a soft-core runs keygen from DRAM. And immunity to Rowhammer is **not** immunity to fault injection — the cryptanalysis here works against any fault model that can perturb `e` during keygen or ρ during signing, including laser, glitch, and EM fault, which *do* reach an ASIC.

## 8-minute slide plan

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** — three schemes, one primitive, no probe | 0:45 |
| 2 | Threat model: co-located unprivileged process vs. physical probe | 0:45 |
| 3 | Rowhammer in 60 s: aggressors/victims, TRR, multi-bank, massaging | 0:45 |
| 4 | Kyber KeyGen listing, the window boxed; why nowhere else works | 1:00 |
| 5 | **Two bits: `0x0002` → 258.** [−2,2] in a 16-bit word | 1:00 |
| 6 | Failure boosting + the 67 % noise admission | 0:45 |
| 7 | Recovery: failures are hints; `E[W] ∥ S` | 0:45 |
| 8 | Results: 8/100, 3/100, 142 s, Geekbench stealth | 0:45 |
| 9 | Dilithium: `s₁ = (z−z′)/(c−c′)`, target ρ, HW(Δz) filter | 0:45 |
| 10 | Countermeasures — lead with **bit packing** | 0:45 |

**Land this line:** *"The secret is five possible values stored in sixteen bits. Fourteen of those bits do nothing but wait for someone to flip one."*

---
---

# PAPER 4 — A Hardware-Friendly Shuffling Countermeasure for Kyber

Dejun Xu, Kai Wang, Jing Tian — Nanjing University · **IEEE TCAS-II** Vol. 72 No. 3, March 2025, pp. 504–508 · arXiv:2407.02452

## SLIDE 1 — What this paper is about

**The problem.** Kyber hardware leaks: Zhao et al. (TCAS-I 2023) identified three side-channel leakage points in the decryption datapath and broke them with CPA. Both existing defence families have a cost problem — hardware **masking** is enormous (Kamucheka et al.: ~1700× the area-time product), and existing **hiding** schemes burn either clock cycles or area.

**What they do.** Take an open-source unprotected Kyber FPGA design (Xing & Li, TCHES 2021) and add a **shuffling** countermeasure. Crucially, they don't restructure the datapath — they **permute the memory addresses** it visits. Two new modules: a Random Permutation Generator (RPG) and an Address Controller (ADDR).

**What they find.** CPA that succeeded at 4×10³ traces finds no peak at 10⁵. TVLA that failed at 10⁴ passes at 10⁷. Cost: **+260 slices, +8.7 % ATP, zero extra clock cycles, zero extra DSPs or BRAMs.**

**Why it matters.** A genuinely cheap hardware SCA countermeasure, and the cheapness comes from one architectural decision.

> **One-sentence thesis:** permute *which memory location* each operation touches and you get shuffling for the price of a mux on the address bus — the datapath never knows.

## 1. Protection point

The **whole Kyber decryption datapath**, via one mechanism. Equation (1):

```
m ← Compress_q( v − INTT( ŝᵀ ∘ NTT(u) ), 1 )
```

| Point | Expression | Stage |
|---|---|---|
| 1 | `ŝᵀ ∘ û` | **PWM** — the multiplier |
| 2 | `(ŝᵀ ∘ û) mod q` | modular reduction after PWM |
| 3 | `v − (sᵀu mod q)` | subtraction after INTT |
| **+4** | **INTT** | added by the authors, since Hamburg et al. broke INTT in software |

All four protected by the *same* mechanism — permuting memory addresses. That's why the marginal cost is one RPG plus muxes rather than four bespoke defences.

*(Why decryption: the secret `ŝ` only appears in equation (1).)*

## 2. Key idea

Fisher–Yates gives a uniform permutation, but has **two hardware problems**:

1. **Continuous random seed supply** — n−1 fresh random numbers; a TRNG that fast is expensive and is itself an attack surface
2. **Dynamically shrinking index range** — uniform `j ∈ [0, i]` needs a divider/modulo (expensive) or rejection sampling (**variable latency**, poison for a fixed-schedule datapath)

The paper's own three criteria for a hardware shuffler (Table I):

| Criterion | TECS [10] | TCAD [13] | INDOCRYPT [14] | **This work** |
|---|---|---|---|---|
| Huge permutation space | ✗ | ✗ | ✓ | ✓ |
| No dynamic input required | ✗ | ✗ | ✗ | ✓ |
| No pre-storage required | ✗ | ✗ | ✓ | ✓ |

## 3. Protection method / steps

**Step 1 — batch the randomness.** A 32-bit LFSR runs **384 cycles** at init, filling a **64×6 FIFO** with every index the permutation needs, up front. External TRNG supplies a **one-shot** seed via `rst_l`, not a stream.
> *The elegant bit:* the FIFO is **reused**. Random indices shift out; selected permutation elements push back in. One out, one in — occupancy stays 64, one memory serves both roles.

**Step 2 — cheap fixed-latency range reduction.**
```
eq (4), stage 1:   idx_0 = idx           if idx ≤ 0x28
                         = idx − 0x28    if idx > 0x28
eq (5), stage 2:   idx_1 = idx           if idx ≤ rest
                         = idx & rest    if idx > rest     ← AND, not modulo
```
No divider, no rejection sampling, **constant latency**.

**Step 3 — dodge the address-conflict hazard.** The datapath is pipelined (read at *t* → write-back at *t*+1 or *t*+12). A naive permutation makes the first six and last six positions collide, and **the computation is wrong**. Fix: a *designed* intermediate permutation, equation (3):
```
per_a = {0x0b … 0x33}                                    41 elements
per_b = {0x34…0x38} ∪ {0x00…0x0a} ∪ {0x39…0x3f}          23 elements
```
Since `min(per_a) = 0x0b > 0x0a` and `max(per_a) = 0x33 < 0x34`, **every per_a element satisfies all 11 constraints at once**. Fill the 12 risky slots from per_a only → safe by construction, no runtime check. Then cyclic-shift right by 6 so those 12 land on the risky positions.

**Step 4 — swap the addresses (ADDR).** Six 2-to-1 muxes replace 4 read + 2 write addresses. **1-cycle and 12-cycle shift registers** keep write-back aligned with the permuted read address. One 6-bit permutation yields **five** address ranges by concatenation: `{line, addr}` for `00–3f`/`40–7f`/`80–bf`/`c0–ff`, and `{addr, 1'b0/1}` for `00–7f`.

## 4. Results

Kyber768, Artix-7 XC7A100T, **EM** traces, Hamming-distance model on multiplier output registers.

| Test | Unprotected | Protected | Ratio |
|---|---|---|---|
| CPA on PWM | distinguishable at **4×10³** | no peak at **1×10⁵** | ≥25× |
| TVLA | \|t\| ≫ 4.5 at **1×10⁴** | within ±4.5 at **1×10⁷** | ≥1000× |

| Resource | Δ | % |
|---|---|---|
| Slices | +260 | **+12.0 %** |
| ENS | +260 | **+8.7 %** |
| Cycles, f_max, DSP, BRAM | 0 | **0 %** |

vs. Kamucheka (masked) ~1700× worse ATP · vs. Jati 7.5× · vs. Moraitis ~1.9×

> **The 8.7 % headline is metric-dependent.** ENS adds a constant 800 for DSPs and BRAMs to both designs, and the countermeasure touches neither — inflating the denominator. On **slices alone** it is **12 %**. Both defensible; know which you're quoting.

## 5. Advantages / disadvantages

| Advantages | Disadvantages |
|---|---|
| **+8.7 % ATP, +260 slices** — an order of magnitude cheaper than hiding alternatives | **32-bit LFSR caps permutation entropy at 32 bits.** All 384 index bits are a *linear* function of 32 state bits; reseeding policy unspecified |
| **Zero cycle overhead, zero DSP/BRAM** — RPG runs in parallel (~454 cycles vs 6.7k–10k), shorter critical path | **Biased shuffle** — never analysed. *[derived]* only **6 of 52** stage-2 rounds are unbiased |
| **One mechanism covers all four leakage points** | **Only CPA and TVLA tested.** No second-order, integrated, or permutation-recovery attack — the ones that matter against hiding |
| **Shuffles addresses, not arithmetic** — datapath untouched | **"No peak at 10⁵" is where they stopped.** Theory for N=64 predicts ~N² = 4096×; only 25× demonstrated |
| Aggressive reuse: one FIFO two roles, one permutation five ranges | **Derived permutations are not independent** — `{π(i), b}` has 64!·structure, not 128! |
| Two-stage construction turns a runtime hazard into a static guarantee | *[derived]* **the static guarantee has a gap** — see below |
| Honest baseline re-synthesis on a matched FPGA part | **No FIA protection** (acknowledged); **hiding, not masking** — no security order |

**[derived] Two findings from my own analysis, worth a line each:**

- **`idx & rest` is *exactly* uniform when `rest+1` is a power of two** (it's then just `mod`). Only `rest ∈ {0,1,3,7,15,31}` qualify — 6 of 52 rounds. Restructuring to power-of-two blocks gives an unbiased shuffle at **zero extra hardware**.
- **The "filled exclusively from per_a" invariant fails.** Algorithm 1 line 13 backfills with `REG[rest]`, and for `rest = 63…52` those registers hold **per_b** values (`0x34–0x38`, `0x00–0x06`). Simulating Algorithm 1 over 200k runs: **86 % of runs put ≥1 non-per_a value into the 12 constrained slots**, averaging 1.62. The contaminants are exactly the boundary values the constraints exist to exclude — `0x34` in 26 % of runs violates `< 0x34`. Either the HDL has a guard the brief omits, or there's a correctness hazard; the constraints are only figure annotations, so it can't be settled from the text.

**Erratum:** Algorithm 1 lines 11 and 17 have their ternary branches **inverted** relative to equations (4) and (5). Trust the equations — the paper's own worked example confirms them.

## 6. Can the idea be applied in our IP? — **Yes, with three substitutions**

The only paper of the set that is actually RTL, and the central idea transfers directly.

**What transfers**

- **Shuffle addresses, not arithmetic.** The reason the cost is 8.7 % is that the datapath is never restructured — a mux on the address bus plus a delay-matched copy for write-back. In any memory-based PQC accelerator with banked coefficient storage, that's the same two lines of RTL. Cheapest structural SCA lever in the hardware literature.
- **The delay-matched write path** (`addr_r1` / `addr_r12`) is the non-obvious engineering detail you'd otherwise have to rediscover. Copy it.
- **Selective disable** (`repl_x = 0` passes the original address) — lets you A/B the countermeasure on the same silicon during characterisation.
- **Composability.** Shuffling is orthogonal to masking; shuffling N slots multiplies attack cost on a d-order masked design. Stack them, don't choose.

**Three things to change first**

1. **Replace the LFSR.** A 32-bit linear generator is not acceptable in an IP — profile once, predict forever. Use a lightweight DRBG; if a hash core is already on die, a Keccak-based DRBG is close to free.
2. **Restructure to power-of-two ranges** — unbiased permutation, same AND gate, no divider, zero extra area.
3. **Resolve the backfill hazard** against real HDL before trusting the "no runtime check needed" claim.

**Two ASIC caveats**

- **Do not quote 8.7 % as an ASIC number.** A 64-to-1 mux + 1-to-64 demux over 6-bit words + a 64×6 register file is cheap in FPGA LUTs, where wide muxes map onto LUT trees. In standard cells it's a real mux tree (~380 2:1 mux-equivalents) plus 384 flops. Small in absolute terms, but the slice ratio doesn't translate — re-estimate.
- **The 454-cycle RPG latency must stay hidden.** Trivial here (454 vs 6.7k). If your IP has shorter operations or runs back-to-back without gaps, you need a second RPG to double-buffer.

**Beyond Kyber:** §VI notes the method applies to any NTT-based scheme and names **Dilithium and Raccoon**. Address permutation is agnostic to what the datapath computes — it only needs coefficients in addressable memory and independence across indices.

## 8-minute slide plan

| # | Slide | Time |
|---|---|---|
| 1 | **What this paper is about** — cheap hardware shuffling for Kyber | 0:45 |
| 2 | Eq. (1) pipeline + the 3 leakage points, plus INTT | 1:00 |
| 3 | Hiding vs. masking — the cost table that motivates hiding | 0:45 |
| 4 | Fisher–Yates and its two hardware problems | 0:45 |
| 5 | Fix 1: LFSR + batched FIFO, and the one-out-one-in reuse | 0:45 |
| 6 | Fix 2: eq. (4)/(5) — reduction without divider or rejection sampling | 0:45 |
| 7 | The address-conflict hazard and the two-stage per_a/per_b fix | 1:15 |
| 8 | Results: CPA 25×, TVLA 1000×, **+8.7 % ATP / 0 cycles** | 1:00 |
| 9 | Advantages / disadvantages + applicability verdict | 1:00 |

**Land this line:** *"The countermeasure never touches the arithmetic. It just lies to the memory about which coefficient you asked for."*
