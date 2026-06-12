# Nullifier Registry — Cross-chain Nullifier Format

**Status:** Draft v1 · **Requires:** [PSR.md](./PSR.md), [CSAP.md](./CSAP.md)

## Abstract

A nullifier is a one-time value consumed on-chain to prevent a zero-knowledge
proof from being replayed for the same action. This document fixes the canonical
nullifier encoding shared by both chains, the registry semantics each chain
implements, the domain-separation rules that keep independent proof systems
(PSR today, the privacy pool in a later phase) from colliding, and the
cross-chain replay model.

## 1. Canonical encoding

A nullifier is a **BN254 scalar field element encoded as 32 big-endian bytes**.

```
nullifier ∈ [0, r)   where r = 21888242871839275222246405745257275088548364400416034343698204186575808495617
```

Verifiers MUST reject any candidate ≥ `r` *before* performing curve or storage
operations (both reference verifiers do; see `is_valid_scalar` in
`solana/programs/groth16-verifier` and the snarkjs-template field check in
`Groth16VerifierV2.sol`). The byte encoding is identical on both chains: no
endianness conversion is applied at any protocol boundary.

## 2. PSR V2 nullifiers

The only nullifier domain live today. As specified in [PSR.md](./PSR.md):

```
nullifier_hash = Poseidon(stealth_pk, external_nullifier)
```

- `stealth_pk` — the prover's stealth private-key scalar (private input).
- `external_nullifier` — a caller-chosen domain separator scoping the proof to
  one action context (vote id, campaign id, loan id, …).

The circuit constrains `nullifier_hash` to equal the public input, so the
on-chain registry can consume the value directly without recomputing Poseidon.

## 3. Registry semantics

A nullifier registry provides exactly one operation: **consume-once**. Marking
an already-consumed nullifier MUST fail atomically with the verification that
depends on it.

**Solana.** A `NullifierEntry` PDA at seeds `["nullifier", nullifier]` under the
`reputation-verifier` program. Anchor's `init` constraint fails if the account
exists — consumption and the existence check are the same instruction.

**Ethereum.** A `mapping(uint256 => bool)` in `OpaqueReputationVerifierV2`;
the verifying function reverts if the slot is already set, then sets it.

Reads are public on both chains: anyone can check whether a nullifier has been
consumed, and the consumed set is enumerable from chain history. This is by
design — unlinkability comes from the nullifier being meaningless without the
stealth key, not from hiding the set.

## 4. Domain separation

Distinct proof systems MUST NOT share a nullifier space, otherwise consuming a
value in one system could censor an unrelated proof in another. Separation is
achieved at two levels:

1. **Registry instance.** Each verifier owns its registry (separate program/
   contract storage). PSR V2 nullifiers live in the reputation verifier's
   registry; future pool nullifiers will live in the pool contract's registry.
   There is deliberately **no global shared registry**.
2. **Hash domain.** Within a proof system, the formula itself separates
   domains (`external_nullifier` for PSR). Future circuits MUST derive their
   nullifiers with a distinct Poseidon input layout so a collision across
   systems is cryptographically infeasible even if registries were merged.

### Reserved domains

| Domain | Formula | Registry | Status |
|---|---|---|---|
| PSR V2 reputation proof | `Poseidon(stealth_pk, external_nullifier)` | reputation verifier (per chain) | **Live (testnet)** |
| Privacy-pool withdrawal | `Poseidon(nullifier)` — see `privacy-pool.md` §3 | pool contract (per chain) | **Live (testnet)** |
| Conditional disclosure (threshold viewing keys) | `Poseidon(nullifier, context, DOMAIN_DISCLOSURE)` with `DOMAIN_DISCLOSURE = keccak256("opaque/disclosure/v1") mod r` — see `conditional-disclosure.md` §7 | disclosure verifier (per chain) | Specified |

The three formulas use distinct Poseidon arities (1, 2, 3 — the disclosure
domain additionally carries a constant tag), satisfying the layout rule above
even though each registry is already a separate contract/program.

## 5. Cross-chain replay model

Nullifier registries are **chain-local**. The same `nullifier_hash` can in
principle be consumed once on Ethereum and once on Solana, because:

- PSR Merkle roots are chain-local (an attestation is issued on one chain), so
  a proof built against chain A's root does not verify on chain B anyway;
- mirroring consumption through the UAB would make every private action depend
  on bridge liveness, which violates the protocol's serverless goal.

Applications that need **one action across both chains** MUST encode the
intended chain into the action scope, e.g.
`external_nullifier = Poseidon(action_id, chain_domain)` where `chain_domain`
is the Wormhole chain id used elsewhere in the protocol (Ethereum = 2,
Solana = 1; see [payload-format.md](./payload-format.md)). Applications that
treat each chain as an independent context need no extra step.

A future pool design with cross-chain unified liquidity would require nullifier
mirroring; that is explicitly out of scope until `privacy-pool.md` specifies it.

## 6. Security considerations

- **Front-running consumption.** A third party observing a pending proof
  transaction could attempt to submit it first; since the proof only consumes
  its own nullifier and executes the prover's intended action, the effect is a
  griefing replay at worst. Verifier entry points SHOULD bind proofs to
  `msg.sender`/payer where the action is beneficiary-specific.
- **Registry growth.** Consumed nullifiers are permanent state. On Solana the
  rent for each `NullifierEntry` PDA is paid by the consumer; on Ethereum the
  storage slot cost is borne in gas. Neither registry supports deletion.
- **Out-of-field injection.** Accepting a value ≥ `r` would allow two byte
  encodings of the same field element (`x` and `x + r`) to be consumed
  independently. The field check in §1 is therefore normative, not advisory.

## Copyright

CC0-1.0.
