# Quantova Research

Research output of the Quantova Inc research department, cryptography and
distributed systems. Papers are published here for public cryptanalysis.
Every result is stated together with the model it is proven in, and nothing
is represented as unconditionally secure. Attacks, gaps in proofs, and
tightenings of bounds are welcome as issues.

## Repository structure

| Path | Contents |
| ---- | -------- |
| [QVRF/QVRF_Quantova_Inc.pdf](QVRF/QVRF_Quantova_Inc.pdf) | The QVRF working paper, version 0.1, July 2026. Read this first. |
| [QVRF/README.md](QVRF/README.md) | Paper summary, precise claims, file guide. |
| [QVRF/CITATION.cff](QVRF/CITATION.cff) | Citation metadata. |
| [.github/ISSUE_TEMPLATE/attack-report.md](.github/ISSUE_TEMPLATE/attack-report.md) | Template for submitting an attack or a correction. |

Future papers follow the same layout, one folder per paper with its own
citation file.

## QVRF

QVRF is a verifiable random function whose security rests exclusively on
primitives standardized by the United States National Institute of Standards
and Technology, the SHA-3 family of FIPS 202 and the module lattice
signature ML-DSA of FIPS 204, with a stateless hash based option from
SLH-DSA of FIPS 205. Quantova Inc is deploying it inside a production
ledger. To our knowledge this makes it the first verifiable random function
composed entirely from NIST standardized post quantum primitives to run in a
production ledger anywhere in the world, and the first to extract per block
randomness from Byzantine finality at zero marginal cost with bias
resistance that reduces to consensus safety.

### The problem it solves

The verifiable random function every ledger deploys today is RFC 9381,
built on elliptic curves. Shor's algorithm solves the discrete logarithm in
quantum polynomial time, so a cryptographically relevant quantum computer
recovers the secret key behind every RFC 9381 beacon outright and voids the
randomness on which leader election, committee sampling, and shard
assignment depend. Post quantum VRFs proposed from lattice assumptions have
stayed out of deployment for two reasons. Several rest on assumptions NIST
has not standardized, weakening the long term assurance institutional
adopters require. And the prevailing designs run as self contained gadgets
that add a computation, an argument, and a message to the consensus
critical path, where latency and bandwidth are scarce. QVRF removes both
obstacles.

### The Quantova QVRF flow

QVRF comprises two constructions sharing one primitive base.

The per block beacon reads the finality certificate the chain already
produces and hashes it once. The certificate is an object no minority can
predict or control, so its unpredictability is not a new assumption but a
restatement of the safety the chain already proves.

```
 per block beacon

 +---------------+     +------------------+     +------------------+     +----------+
 |   validator   |     |  BFT aggregate   |     |     SHAKE256     |     | seed[h]  |
 | ML-DSA attest.| --> | certificate C[h] | --> | seed[h]=XOF(..)  | --> | 32 bytes |
 +---------------+     +------------------+     +------------------+     +----------+

 cost, one hash per block, no new protocol message
```

The seed chain is recomputable by any verifier from public block data.

```
 seed[0] = H( genesis_nonce )
 seed[h] = XOF( seed[h-1] || H(C[h]) || LE64(h) , 256 )
```

No key, signature, argument, or message is introduced. The sole cost is one
XOF over roughly seventy bytes, independent of validator count.

The per input construction binds randomness to a specific key. The prover
signs the input with ML-DSA in deterministic mode, derives the output by
hashing the signature, and certifies the derivation with a transparent hash
based STARK. Verification touches nothing but ML-DSA verification and SHA-3.

```
 per input VRF

 +--------------+     +----------------+     +----------------+     +-------------+
 |   input x    |     |  ML-DSA det.   |     |    SHAKE256    |     |  output y   |
 | + secret key | --> |  σ=Sign(sk,x)  | --> |  y=XOF(σ||x)   | --> |  + proof π  |
 +--------------+     +----------------+     +----------------+     +-------------+
                              |                      |
                              +----------+-----------+
                                         v
                       +---------------------------------+
                       |  STARK proof   y = XOF(σ || x)  |
                       |  verified with SHA-3 and        |
                       |  ML-DSA operations only         |
                       +---------------------------------+
```

```
 primitive base

 ML-DSA (FIPS 204)   SLH-DSA (FIPS 205)   SHA-3 / SHAKE (FIPS 202)   hash STARK

 no elliptic curve, no pairing, no classical assumption anywhere
```

### Security results

Three theorems in the paper fix what is proven and against what adversary.

Theorem 1, unique provability. If ML-DSA is strongly existentially
unforgeable and the STARK has knowledge soundness error ε_STARK, then any
adversary producing two verifying outputs for one input under one public
key breaks one of the two, with Adv_Uniq(A) ≤ Adv_SUF-CMA(B) + ε_STARK.
Derandomized signing is load bearing here. Under hedged ML-DSA a message
admits many signatures and unique provability fails, so the deterministic
mode is enforced as a consensus rule checkable from the signature encoding.

Theorem 2, pseudorandomness. Modelling the XOF as a quantum random oracle,
for every adversary issuing q queries, Adv_PR(A) ≤ Adv_EUF-CMA(B) +
2q · 2^-128. The additive term is the quantum reprogramming advantage
against the 128 bit quantum preimage margin of a 256 bit output.

Theorem 3, bias resistance. Any adversary controlling strictly less than
one third of weighted stake that biases any bit of seed[h] with advantage ε
yields a consensus safety break with advantage at least ε − q · 2^-256.
Beacon unpredictability is therefore exactly as strong as the safety of the
ledger it runs on, and withholding forfeits the slot to the liveness path
rather than granting a resample.

### Work factors under quantum attack

Every QVRF component holds at or above the 128 bit quantum floor that a
256 bit hash establishes. The elliptic curve construction it replaces does
not survive at all.

| Component | Classical work | Quantum work |
| --------- | -------------- | ------------ |
| ECVRF key recovery (RFC 9381) | 2^128 | broken by Shor |
| ML-DSA-65 forgery | 2^207 | 2^192 |
| SHA-3 preimage | 2^256 | 2^128 |
| SHA-3 collision | 2^128 | 2^128 |
| Beacon bias | consensus safety bound | consensus safety bound |

There is no component whose defeat is cheaper under quantum attack than
under classical attack, which is the precise sense in which the
construction is post quantum by design rather than by retrofit.

### Deployment parameters

Sizes correspond to ML-DSA-65, NIST security category 3, the default
parameter set.

| Object | Size | Frequency |
| ------ | ---- | --------- |
| Stored secret (seed) | 32 bytes | per key |
| ML-DSA-65 public key | 1,952 bytes | revealed on first proof |
| ML-DSA-65 signature σ | 3,309 bytes | per input, off the finality path |
| Per input argument π_s | 45 to 200 KB | per input, off the finality path |
| Beacon output seed[h] | 32 bytes | per block |
| Beacon argument | none, C[h] is the witness | per block |

The per input signature and argument never enter consensus messages. They
accompany an application transaction, are verified off the finality path,
and may be pruned after verification, so their magnitude does not bear on
propagation or finality latency. Cryptographic agility is retained by a one
byte scheme identifier prefixing every key and signature, so ML-DSA-44,
ML-DSA-87, or a future standardized signature is adopted without altering
the beacon or the verification interface.

### What is claimed

The first verifiable random function composed entirely from NIST
standardized post quantum primitives. The first to extract per block
randomness from Byzantine finality at zero marginal cost with bias
resistance that reduces to consensus safety. Post quantum by design.
Nothing beyond that is claimed, and priority over the abstract notion of a
post quantum VRF, which precedes this work, is expressly not claimed.

### What is not claimed

- Theorems 2 and 3 are in the quantum random oracle model. Standard model
  analogues are open.
- Theorem 3 is conditional on the concrete safety advantage of the
  deployed protocol, which its own analysis supplies.
- The pseudorandomness bound is stated in standard, non tightened form.
- Unique provability requires strong unforgeability and derandomized
  ML-DSA. Hedged signing invalidates Theorem 1.
- STARK knowledge soundness is modular, with ε_STARK drawn from the
  prover's specification.

You will not find the words provably secure, unbiasable, or quantum proof
in this repository. A construction earns those the day everyone fails to
break it, not the day it ships.

### Submitting an attack

The point of publishing is to invite fire. If you can bias the beacon
without breaking consensus safety, forge a per input proof without breaking
ML-DSA, or find a gap in a proof, open an issue with the attack report
template. State the claim you are targeting, the paper section, the attack,
and the correction you would make. Credit goes to whoever lands the hit.

### Citation

Citation metadata lives at [QVRF/CITATION.cff](QVRF/CITATION.cff) and
GitHub renders it in the sidebar of this repository.

### Related repositories

The reference implementation lives at
[github.com/Quantova/QVRF](https://github.com/Quantova/QVRF). The
implementation specification will live in the Quantova-Specs repository as
SPEC-vrf.md.
