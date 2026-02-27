# HQC Cryptol Specification

A complete Cryptol specification of [HQC](https://pqc-hqc.org/) (Hamming
Quasi-Cyclic), the code-based post-quantum KEM selected by NIST for
standardization. Targets the **August 2025** version of the HQC specification.

## Files

```
Specification.cry              Parameterized spec (all algorithms)
Instantiations/
  HQC128.cry                   HQC-1 (128-bit security)
  HQC192.cry                   HQC-3 (192-bit security)
  HQC256.cry                   HQC-5 (256-bit security)
Tests/
  HQC128.cry                   Test vectors and properties for HQC-128
  AdaCrossValidation.cry       Cross-validation against Ada/SPARK implementation
  run_tests.icry               Full test suite batch file (~15-20 min)
  run_tests_fast.icry          Fast test suite, skips KEM-level tests (~1-2 min)
gen_test_vectors.sh            Generate reference vectors for Ada cross-validation
```

## What is implemented

The specification covers the full HQC-KEM algorithm:

- **GF(2^8) arithmetic** with irreducible polynomial 0x11D
- **Reed-Solomon** systematic encoding and decoding (Berlekamp-Massey,
  Chien search, Forney's formula)
- **Reed-Muller RM(1,7)** encoding and decoding (Hadamard transform)
- **Concatenated RS+RM** code encode/decode
- **GF(2)[X]/(X^n-1)** polynomial multiplication via Cryptol's `pmult`/`pmod`
- **SHAKE256/SHA3** hash functions with HQC domain separator bytes
- **Rejection sampling** (`SampleFixedWeightVect_s`) for key generation
- **Fisher-Yates sampling** (`SampleFixedWeightVect`) for encryption
- **HQC-PKE**: KeyGen, Encrypt, Decrypt
- **HQC-KEM**: KeyGen, Encaps, Decaps (FO transform with implicit rejection)

## Properties

The specification and test suite include properties across several categories:

### Specification properties (in `Specification.cry`)

| Property | Method | Status |
|---|---|---|
| `gf_mul_commutative` | `:prove` | Q.E.D. |
| `gf_mul_associative` | `:check` | Pass |
| `gf_inv_correct` | `:exhaust` | Q.E.D. (256/256) |
| `gf_alpha_primitive` | `:prove` | Q.E.D. |
| `RS_encode_decode_roundtrip` | `:check` | Pass |
| `RM_encode_decode_roundtrip` | `:exhaust` | Q.E.D. (256/256) |
| `C_encode_decode_roundtrip` | `:check` | Pass |
| `ring_mul_commutative` | `:check` | Pass |
| `kem_correctness` | `:check` | Pass (~10s/test) |

### KAT tests (in `Tests/HQC128.cry`)

Intermediate value tests against the C reference implementation:

| Property | Method | Checks |
|---|---|---|
| `RS_Encode_correct` | `:prove` | RS encoding matches reference |
| `RS_Decode_correct` | `:prove` | RS decoding matches reference |
| `C_roundtrip` | `:prove` | Concatenated code round-trip |
| `hash_i_correct` | `:prove` | Hash_I matches reference |
| `seed_derivation_correct` | `:prove` | seed_pke and sigma derivation |
| `hash_g_correct` | `:prove` | Hash_G produces correct K and theta |
| `keygen_correct` | `:check` | Full keygen matches reference |
| `encaps_correct` | `:check` | Encaps matches reference |
| `decaps_correct` | `:check` | Decaps round-trip matches reference |

### Noisy decoding tests (in `Tests/HQC128.cry`)

| Property | Method | Checks |
|---|---|---|
| `RM_roundtrip` | `:check` | RM byte-level encode/decode round-trip |
| `RM_Decode_1bit_noise` | `:exhaust` | RM decodes with 1 bit flip |
| `RM_Decode_heavy_noise` | `:exhaust` | RM decodes with 95 bit flips (near threshold) |
| `RS_Decode_1error` | `:check` | RS corrects 1 symbol error |
| `RS_Decode_max_errors` | `:check` | RS corrects 15 errors (max capacity) |
| `RS_Decode_beyond_capacity_fails` | `:check` | 16 errors exceeds correction bound |
| `C_Decode_1bit_noise` | `:check` | Concatenated code corrects 1 bit flip |

### Algebraic and structural properties (in `Tests/HQC128.cry`)

| Property | Method | Checks |
|---|---|---|
| `gf_mul_distributive` | `:check` | GF(2^8) distributivity |
| `gf_exp_matches_pow` | `:exhaust` | Exp table matches gf_pow |
| `rs_generator_roots` | `:prove` | alpha^1..alpha^30 are roots of g(x) |
| `RS_Syndromes_zero_for_valid_codeword` | `:check` | Valid codewords have zero syndromes |
| `C_Encode_injective` | `:check` | Distinct messages → distinct codewords |
| `ring_mul_identity` | `:check` | Polynomial 1 is multiplicative identity |
| `ring_mul_zero` | `:check` | Polynomial 0 annihilates |
| `serialization_roundtrip` | `:check` | bits↔bytes round-trip is lossless |
| `kem_implicit_rejection` | `:check` | Corrupted ciphertext triggers rejection |

## Cross-validation

The specification has been cross-validated against:

- **C reference implementation** — intermediate values match byte-for-byte
  for the deterministic test vector (PRNG seeded with `[0,1,...,47]`)
- **Ada/SPARK implementation** — bidirectional cross-validation:
  - `Tests/AdaCrossValidation.cry` verifies Cryptol against Ada-generated vectors
  - `gen_test_vectors.sh` generates Cryptol vectors for Ada to verify against

## Design choices

The following choices deviate from the spec but are traceably equivalent:

- **Chien search** for root finding instead of additive FFT
  Both produce identical results.
- **Computed tables** for GF(2^8) exp/log and RS generator polynomial,
  rather than hardcoded lookup tables.

## Usage

```
$ cryptol
:l Primitive/Asymmetric/KEM/HQC/Instantiations/HQC128.cry

// Run all provable properties
:prove gf_mul_commutative
:prove gf_alpha_primitive
:exhaust gf_inv_correct
:exhaust RM_encode_decode_roundtrip

// Run checked properties
:check RS_encode_decode_roundtrip
:check C_encode_decode_roundtrip
:check kem_correctness

// Evaluate KEM
let (ek, dk) = KeyGen (split 0x0102...1f20)
let (K, ct) = Encaps ek (split 0xdead...beef) (split 0xcafe...babe)
let ss = Decaps dk ct
ss == K  // True
```

### Batch test suites

```bash
# Fast suite (~1-2 min) — GF, RS, RM, code, hash, serialization
cryptol -b Primitive/Asymmetric/KEM/HQC/Tests/run_tests_fast.icry

# Full suite (~15-20 min) — includes vect_mul and KEM-level tests
cryptol -b Primitive/Asymmetric/KEM/HQC/Tests/run_tests.icry
```
