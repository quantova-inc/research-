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

QVRF comprises three constructions sharing one primitive base.

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

The sortition construction covers the setting the signature construction
cannot, the one in which the drawing party is itself the adversary. A FIPS
204 signature does not carry its randomizer, so no verifier can check that
a signer derandomized, and an adversary drawing its own committee seat
could hedge and re draw until the image fell under its threshold. Sortition
draws instead from one time preimages committed under a Merkle root that is
bound to the account's stake bond and registered before the beacon of any
slot it serves is determined. For a fixed account and slot exactly one
image exists, so there is nothing to re draw. The cost of a draw is one
hash and one Merkle opening, with no signature, no argument, and no
randomness supplied by the drawer.

```
 sortition draw

 +--------------------+     +----------------------+     +--------------------------+
 |  register          |     |  Draw(N)             |     |  verify                  |
 |  R=MerkleRoot(p_i) |     |  d=H(p_N || seed[h]) |     |  MerkleVfy(R,N,p_N,π)    |
 |  one time preimages| --> |  π=MerkleOpen(R,N)   | --> |  d=H(p_N || seed[h])     |
 |  bound to a stake  |     |                      |     |  member iff d < T(stake) |
 |  bond, in advance  |     |                      |     |                          |
 +--------------------+     +----------------------+     +--------------------------+
```

```
 primitive base

 ML-DSA (FIPS 204)   SLH-DSA (FIPS 205)   SHA-3 / SHAKE (FIPS 202)   hash STARK

 no elliptic curve, no pairing, no classical assumption anywhere
```

### Security results

Four theorems in the paper fix what is proven and against what adversary.

Theorem 1, unique provability. If ML-DSA is strongly existentially
unforgeable and the STARK has knowledge soundness error ε_STARK, then any
adversary producing two verifying outputs for one input under one public
key breaks one of the two, with Adv_Uniq(A) ≤ Adv_SUF-CMA(B) + ε_STARK.
Derandomized signing is load bearing here, and it is an assumption about
the signer rather than a rule a protocol can check, since a FIPS 204
signature does not carry its randomizer. The per input construction is
therefore deployed only where the signer is honest or has no incentive to
re draw. Wherever the drawing party is the adversary, the sortition
construction is used instead.

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

Theorem 4, structural uniqueness for sortition. For any registered root and
slot index, no adversary produces two accepted draws with distinct images
except with advantage at most q · 2^-128 in the random oracle model.
Uniqueness here is by construction rather than by assumption. There is no
randomizer to vary and nothing to re draw, and because the root is
committed before the beacon of any slot it serves is determined, the
preimages cannot have been selected for a favourable image. The
construction rests on SHA-3 alone.

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
- Theorem 1 assumes a derandomized signer. That assumption is not
  verifiable from a FIPS 204 signature, so the per input construction is
  unsuitable wherever the drawing party is the adversary, and the sortition
  construction is used there instead.
- Theorem 4 gives uniqueness per account and slot. Stake splitting remains
  available at the cost of a full bond per draw, and it buys draws, not
  expected seats. Committee membership is stake neutral under splitting,
  and a stake neutral leader weighting is a requirement on the deployed
  protocol verified by conformance testing rather than proven here.
- An account may draw, observe an unfavourable image, and reveal nothing.
  This forfeits a seat rather than obtaining one, and the paper records it
  as an accepted residual rather than a closed case.
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
