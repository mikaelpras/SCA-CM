# Applications and Use Cases

**Invention:** Masked minimal-ring sparse polynomial multiplication for ML-DSA (FIPS 204) signature generation

**Purpose of this document:** Supporting material for patent submission. Lists fields of use and deployment contexts in which the claimed invention provides a technical advantage.

---

## 1. Technical Advantages Being Mapped

Each application below is selected because at least one of the following properties of the invention addresses a binding constraint in that context:

| # | Advantage | Constraint it addresses |
|---|-----------|-------------------------|
| A1 | No large modular multiplier in the masked signing datapath | Silicon area, die cost, power |
| A2 | Reduced gate count for the challenge–secret product | Area-constrained secure IP integration |
| A3 | Reduced randomness (TRNG entropy) demand per masking order | TRNG throughput becomes the bottleneck at high order |
| A4 | Masking order scalable without datapath redesign | One IP core serving multiple certification tiers |
| A5 | Sparse representation reduces secret-key storage and bit complexity | On-chip memory footprint |

Applications are ordered by strength of fit. Sections 2–4 represent the primary claimed fields of use; sections 5–8 are secondary.

---

## 2. Constrained Secure Hardware — Primary Field of Use

The strongest fit. These devices combine a hostile physical-access threat model with severe area budgets, which is precisely the tension the invention resolves.

- **Secure element ICs.** Area is the dominant per-unit cost driver at high volume. A multiplier-free masked datapath (A1, A2) directly reduces die size. High-order masking is a prerequisite for Common Criteria EAL5+/EAL6+ certification with AVA_VAN.5 attack-potential resistance (A4).
- **SIM, eSIM, and iSIM.** Extreme area constraint, physically accessible for the full service life of the device.
- **Embedded secure enclaves within larger SoCs.** The cryptographic block competes directly against application logic for area; reduction in the signing datapath is realized as usable silicon elsewhere.
- **TPM 2.0 modules and discrete root-of-trust devices.** Attestation signing performed under assumed physical access to the package.
- **DICE and device-identity engines.** Per-boot key derivation and signing implemented in minimal silicon.
- **Hardware wallets and cold-storage signing devices.** Explicit physical side-channel threat model; consumer price point constrains area.

## 3. Automotive — Primary Field of Use

Vehicle service lifetimes of 15–20 years cross the post-quantum migration horizon, making PQC signing mandatory in designs being taped out today. Regulatory drivers include ISO/SAE 21434 and UNECE R155/R156.

- **ECU secure boot and authenticated firmware update.** Signature verification and, for gateway ECUs, signature generation, under strict cost-per-unit targets.
- **Automotive HSM cores embedded in MCUs** (EVITA Full/Medium class). Area-constrained by definition — the HSM must fit alongside the application core (A1, A2).
- **V2X message authentication.** High signature throughput per vehicle; the sparse representation additionally yields raw performance benefit beyond the masking advantage (A5).
- **EV charging authentication** (ISO 15118 Plug and Charge). Roadside equipment is physically exposed and unattended.
- **Component and sensor anti-counterfeiting.** Very low cost budget per authenticated part.

## 4. Government, Defense, and Aerospace — Primary Field of Use

- **CNSA 2.0 compliance.** The NSA Commercial National Security Algorithm Suite 2.0 mandates ML-DSA for national security systems, with firmware and software signing among the earliest required transitions. This is the strongest near-term regulatory driver for the invention.
- **Satellite and spacecraft command authentication.** Decade-plus missions with no field upgrade path. Radiation-hardened process nodes make silicon area disproportionately expensive, amplifying A1 and A2.
- **Tactical radios and deployed field equipment.** Capture-and-analyze is a realistic and planned-for threat model, requiring high-order masking (A4).
- **Platform, munition, and subsystem authentication.**

## 5. Payment and Identity Credentials

- **EMV chip cards and contactless payment.** EMVCo certification requires demonstrated side-channel resistance. Contactless operation imposes a hard power budget that constrains both logic area and TRNG throughput, engaging A1 and A3 simultaneously.
- **ePassports, national identity cards, and eIDAS-conformant signature creation devices.** Long credential validity periods; the adversary has unlimited physical access to the token.
- **Point-of-sale terminals and PIN entry devices** (PCI PTS).
- **Transit and access-control credentials.**

## 6. Infrastructure, Datacenter, and Cloud

- **Hardware security modules (FIPS 140-3 Level 3 and Level 4).** Level 4 requires robust physical protection. At the masking orders required for this tier, TRNG throughput rather than logic area frequently becomes the limiting factor — directly addressed by A3.
- **Code signing appliances and PKI certificate authority hardware.**
- **Cloud provider root-of-trust silicon** (Caliptra-class and equivalent open root-of-trust designs).
- **Confidential computing attestation.** Per-workload attestation signing at datacenter scale.
- **Network infrastructure identity.** MACsec and IPsec device authentication, line-card and pluggable module identity.

## 7. Industrial and IoT

- **Industrial controllers and PLCs** (IEC 62443 compliance).
- **Smart meters and grid-edge devices.** Multi-decade field deployment, physically accessible in uncontrolled locations, tight power budgets.
- **Medical implants and connected medical devices.** Extreme power constraint; authentication of commands is safety-critical.
- **Building access control and secure reader endpoints.**

## 8. Supply Chain and Content Protection

- **Semiconductor provenance and anti-counterfeit marking.**
- **FPGA bitstream authentication.**
- **Firmware signing and secure boot** across all categories above.
- **DRM and content protection endpoints.**

---

## 9. Cross-Cutting Arguments

These apply across multiple fields of use and are worth stating in the submission independently of any single application.

**Device longevity, not harvest-now-decrypt-later.** The retroactive-decryption argument does not apply to signatures. The correct argument for signing is service life: any device signing today with a 10+ year deployment horizon requires ML-DSA now, and requires it side-channel hardened because the device resides in the adversary's physical possession for that entire period.

**Randomness as the true high-order bottleneck.** Published high-order masked ML-DSA implementations report randomness consumption growing steeply with masking order. Where TRNG throughput rather than combinational area limits achievable order, a construction that reduces entropy demand per order (A3) raises the maximum certifiable security order for a fixed hardware budget. This is a distinct and separately claimable advantage from area reduction.

**Order-configurability as a licensing feature.** A single IP core parameterizable across masking orders (A4) can serve multiple certification tiers — from a first-order commercial IoT part to an EAL6+ secure element — without redesign or re-verification of the datapath. This materially broadens the addressable licensing market for the IP.

**Potential generalization to ML-KEM.** If the narrow-ring approach extends to Kyber's small secret coefficients, the invention supports unified masked ML-DSA/ML-KEM datapaths. Unified protected implementations of both algorithms are an active research and product direction, and this generalization broadens the claim scope. *To be confirmed by analysis before asserting.*

---

## 10. Notes for the Submission

- Applications do not establish novelty or non-obviousness and will not be weighed by an examiner for those purposes. Their function here is threefold: to support broad apparatus claim language ("a cryptographic device comprising..."), to demonstrate commercial value to the internal IP review committee, and to supply fallback dependent claims tied to specific deployment contexts.
- Recommend leading the submission with three applications only — **secure element ICs**, **automotive HSM cores**, and **CNSA 2.0 firmware signing** — which carry the area, longevity, and regulatory-mandate arguments respectively. The remainder can be compressed into a single "and other applications including..." sentence. Counsel will typically trim an exhaustive list to those where the technical advantage is causally connected to the application's binding constraint.
- Each application listed above should be checkable against at least one advantage in the Section 1 table. Any application that cannot be so mapped should be removed rather than retained as filler.
