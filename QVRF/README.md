# QVRF working paper

Quantova Inc, Cryptography and Distributed Systems Research. Working paper,
version 0.1, July 2026. Released for public cryptanalysis. Proven results
and conjectures are distinguished throughout.

## Files

- `QVRF_Quantova_Inc.pdf`, the working paper.
- `CITATION.cff`, citation metadata for referencing the work.

## Summary

Shor's algorithm reduces integer factorization and the discrete logarithm
to polynomial time, and with them the elliptic curve verifiable random
function of RFC 9381, whose secret key a quantum adversary recovers
outright. QVRF is a verifiable random function whose security rests
exclusively on NIST standardized primitives, SHA-3 (FIPS 202) and ML-DSA
(FIPS 204), with a stateless hash based option from SLH-DSA (FIPS 205).

It comprises two constructions. A per input verifiable random function
derives its image from a derandomized ML-DSA signature and a transparent
hash based argument of correct evaluation, with unique provability under
strong existential unforgeability and pseudorandomness in the quantum
random oracle model. A per block beacon extracts protocol randomness from
the Byzantine fault tolerant finality certificate that consensus already
produces, with an explicit reduction showing that any adversary biasing the
beacon yields an adversary breaching consensus safety. The beacon costs one
hash evaluation per block and no additional protocol message.

The paper gives the security games, the reductions with concrete advantage
bounds, a quantitative analysis of classical and quantum work factors
against each component, and an explicit account of which results are proven
and which are conjectural pending review.

## What is claimed, precisely

- The first verifiable random function composed entirely from NIST
  standardized post quantum primitives, integrated into a production
  ledger.
- The first to extract per block randomness from Byzantine finality at zero
  marginal cost, with bias resistance that reduces to consensus safety.

Priority over the abstract notion of a post quantum verifiable random
function is not claimed. That idea precedes this work. The claim is the
composition, the deployment, and the finality extraction.

## What is not claimed

- Pseudorandomness is proven in the quantum random oracle model, not the
  standard model.
- The bias resistance theorem is only as formal as the safety bound of the
  BFT protocol it plugs into.
- The per input pseudorandomness bound is stated in standard, non tightened
  form.
- Everything assumes the deterministic ML-DSA mode. Hedged mode breaks
  uniqueness.
- The STARK is treated as a black box with assumed soundness.

No result is represented as unconditionally secure.

## Contact

Open an issue with an attack, a gap in a proof, or a tightening of a bound.
Use the attack report template in this repository.
