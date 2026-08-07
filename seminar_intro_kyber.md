# Seminar Intro — Kyber / ML-KEM in One Page

To open the session, before the six papers. Budget ~5 minutes.

---

## 1. What it is

**ML-KEM (FIPS 203)**, derived from CRYSTALS-Kyber — the **only** KEM NIST standardized. Security rests on **Module-LWE**. All arithmetic lives in

```
R_q = Z_q[X] / (X^n + 1)      n = 256,  q = 3329  (prime)
```

| | ML-KEM-512 | ML-KEM-768 | ML-KEM-1024 |
|---|---|---|---|
| module rank `k` | 2 | 3 | 4 |
| `η₁` (secret, error) | 3 | 2 | 2 |
| `η₂` (encryption noise) | 2 | 2 | 2 |
| `(d_u, d_v)` compression | (10, 4) | (10, 4) | (11, 5) |

> **Reading older papers:** Round-2 Kyber512 used `η₁ = 2` and `d_v = 3`. Papers from 2020 assume those. It changes secret-coefficient counts (5 vs 7 candidates) and which ciphertext values are injectable.

---

## 2. The two-layer structure — the thing to internalize first

```
            ┌─────────────────────────────────────┐
            │   IND-CCA KEM  (Encaps / Decaps)    │   ← Fujisaki–Okamoto transform
            │   ┌─────────────────────────────┐   │
            │   │  IND-CPA PKE                │   │   ← the lattice math
            │   │  KeyGen / Encrypt / Decrypt │   │
            │   └─────────────────────────────┘   │
            └─────────────────────────────────────┘
```

The inner PKE is **only CPA-secure** — it falls trivially to chosen-ciphertext attacks. The FO transform wraps it to get CCA security. **Almost everything interesting about Kyber's physical security happens at that boundary.**

---

## 3. The IND-CPA PKE

**KeyGen** — build an MLWE instance
```
Â ← U(R_q^{k×k})   from seed ρ
s, e ← β_η₁                          ← s is THE SECRET
t̂ = Â ∘ ŝ + ê
pk = (t̂, ρ)        sk = ŝ
```

**Encrypt(pk, m, r)** — hide the message in the noise
```
r, e₁ ← β_η₁ ,  e₂ ← β_η₂
u = NTT⁻¹(Âᵀ ∘ r̂) + e₁
v = NTT⁻¹(t̂ᵀ ∘ r̂) + e₂ + ⌈q/2⌋·m     ← message scaled to half the modulus
c = ( Compress(u, d_u), Compress(v, d_v) )
```

**Decrypt(sk, c)** — subtract the mask
```
u′ = Decompress(c₁, d_u) ,  v′ = Decompress(c₂, d_v)
m′ = v′ − NTT⁻¹( ŝᵀ ∘ NTT(u′) )
m  = Compress(m′, 1)                  ← 1 bit per coefficient
```

**Why it works, and the equation the whole seminar orbits:**

```
m′ = ⌈q/2⌋·m + Δm ,   Δm = ⟨e, r⟩ − ⟨s, e₁ + Δu⟩ + e₂ + Δv
```

Every term of `Δm` is small, so rounding each coefficient of `m′` to 0 or `q/2` recovers `m`. Two properties fall out and both matter:

- **`Δm` is linear in the secret `(s, e)`.** The message polynomial is not just noisy — it is noisy *in a way that depends on the key*.
- **Decryption failures are possible**, with probability driven below 2⁻¹²⁸ by parameter choice. Failures leak key information, which is why the failure rate is a security parameter and not just a usability one.

---

## 4. The FO transform — how CCA security is bought

**Encaps(pk)**
```
m ← random 32 bytes
(K̄, r) = G( m ‖ H(pk) )              ← randomness is DERIVED, not sampled
c = CPA.Encrypt(pk, m, r)
return (c, K = K̄)
```

**Decaps(sk, c)**
```
1.  m′  = CPA.Decrypt(ŝ, c)
2.  (K̄′, r′) = G( m′ ‖ H(pk) )        ← hashes the decrypted message
3.  c′  = CPA.Encrypt(pk, m′, r′)     ← RE-ENCRYPTION
4.  if c = c′ :  return K = K̄′
5.  else      :  return K = J(z ‖ c)   ← IMPLICIT REJECTION
```

**Two design ideas, both essential:**

**(a) Derandomization makes ciphertexts verifiable.** In Encaps the encryption randomness `r` is derived deterministically from `m`. So an honest ciphertext is *reproducible*: decrypt to `m′`, re-derive `r′`, re-encrypt, and you must get the same `c` back. If `c ≠ c′`, the ciphertext was not honestly generated.

**(b) Implicit rejection hides the verdict.** On failure the device does not return `⊥`. It returns a pseudorandom key derived from a secret `z` stored in `sk`. From the outside, a rejected ciphertext is indistinguishable from an accepted one — the attacker learns nothing from the output.

Together these are supposed to close the chosen-ciphertext door completely.

---

## 5. Where the secret becomes observable

Three distinct classes. This is the map for the rest of the session.

### A. The secret is touched directly — the decryption arithmetic

`NTT(u′)` → **`ŝᵀ ∘ û`** → modular reduction → `INTT`

The pointwise multiply manipulates `ŝ` itself. This is the obvious target, and the one designers instinctively protect.

**Note the scope limit:** `ŝ` appears *only* in Decaps. KeyGen and Encaps never touch it. That is why every attack in the literature targets decapsulation.

### B. The secret is touched indirectly — everything downstream of `m′`

`v′ − w` → `m′` → `Compress` → `m` → **`G(m′ ‖ H(pk))`** → re-encrypt → compare

Two separate reasons this region is sensitive:

- **For a chosen ciphertext**, `m′` becomes a direct function of individual secret coefficients. An attacker who can craft `c` makes the decrypted message a one-bit readout of the key.
- **For a valid ciphertext**, `m′ = ⌈q/2⌋·m + Δm`, and `Δm` is linear in `(s, e)`. The *noise* carries the key even when the message doesn't.

**The structural point worth stating slowly:**

> To *check* whether a ciphertext is honest, the device must first **decrypt** it, then **hash** the result, then **re-encrypt** it. The rejection decision comes last. So the key-dependent value `m′` has already been computed, hashed, and re-encrypted **before** the device decides to throw the result away.
>
> Implicit rejection hides the *output*. It does nothing about the *computation*.

This is why protecting the polynomial arithmetic alone is insufficient — the sensitive region extends through Keccak, through re-encryption, and through the comparator.

### C. Integrity, not confidentiality

Some values break the scheme if **corrupted**, regardless of whether they leak:

| Value | Where | Consequence if perturbed |
|---|---|---|
| `e` during KeyGen | KeyGen | inflates the decryption failure rate → failures leak the key |
| the comparison result | Decaps step 4 | forces acceptance of malformed ciphertexts |
| the RNG seed path | KeyGen | a deterministic keypair is a recoverable keypair |

Note that some of these are **public** values. Public does not mean safe to corrupt.

---

## 6. Slide split and timing (~5 min)

| # | Slide | Content | Time |
|---|---|---|---|
| 1 | **What ML-KEM is** | MLWE, ring, parameter table, the round-2 caveat | 1:00 |
| 2 | **The two-layer structure** | CPA PKE inside, FO wrapper outside — "the inner scheme is not CCA-secure on its own" | 0:45 |
| 3 | **IND-CPA PKE** | KeyGen / Encrypt / Decrypt, then `m′ = ⌈q/2⌋m + Δm` and the linearity of `Δm` | 1:15 |
| 4 | **The FO transform** | Encaps/Decaps pseudocode; derandomization + implicit rejection | 1:00 |
| 5 | **Block diagram** | the decapsulation pipeline with both sensitive regions marked | 1:00 |

**Land this line before moving to the papers:**

> *"The secret only appears in decapsulation — but decapsulation includes a hash and a full re-encryption of a value derived from the secret. The sensitive region is much larger than the multiply."*

---

## 7. Two questions to expect

**"If the ciphertext is rejected, why does leakage matter?"**
Because rejection happens at step 4, and the leak happens at steps 1–3. The device has already done the sensitive work by the time it decides to discard the result.

**"Doesn't a static vs. ephemeral key distinction matter here?"**
Yes, and it's worth stating up front. Most attacks on decapsulation need many adaptive queries against a **long-term** key — a smartcard, HSM, or IoT device. Ephemeral-key usage in a TLS handshake is a much harder target.
