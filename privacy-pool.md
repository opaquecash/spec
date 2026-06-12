# Privacy Pool — amount privacy with association-set compliance

**Status:** Draft v1 (testnet) · **License:** CC0-1.0 · **Proof system:** Groth16 over BN254 (Poseidon)

> The privacy pool adds **amount privacy** on top of stealth addresses: a user deposits a
> value-bearing commitment and later withdraws to a fresh stealth address with no on-chain
> link between deposit and withdrawal. Withdrawal requires a zero-knowledge proof that the
> commitment is in the pool **and** that its deposit belongs to a curated **association
> set** (the "clean" set published by an Association Set Provider), so honest users can
> cryptographically dissociate from illicit deposits without revealing which deposit is
> theirs. This is the Privacy Pools construction (Buterin, Illum, Nadler, Schär,
> Soleimani, 2023), as implemented by 0xbow, not a mixer: see [DISCLAIMER.md](../ethereum/DISCLAIMER.md).

## 0. Standards note (ERC-8065)

The execution plan referenced "ERC-8065-compatible events." Verified 2026-06-12:
[ERC-8065](https://eips.ethereum.org/EIPS/eip-8065) is real but is the **Zero Knowledge
Token Wrapper** (an EIP-7503-style provable burn-and-remint primitive) — a different
privacy model from association-set pools. **This pool does not claim ERC-8065
compatibility.** It follows the Privacy Pools paper and the 0xbow reference
implementation. Event shapes below are Opaque's own, designed for the SDK scanner and an
ASP indexer.

## 1. Commitments

All values are BN254 field elements; `Poseidon` is the circomlib Poseidon used elsewhere
in Opaque (PSR V2). A deposit is identified by a **commitment**:

```
precommitment = Poseidon(nullifier, secret)
commitment    = Poseidon(value, label, precommitment)
```

| Field | Meaning |
|---|---|
| `value` | Deposited amount (wei / lamports), range-checked `< 2¹²⁸` |
| `nullifier`, `secret` | Random 254-bit field elements chosen by the depositor (private, never on-chain) |
| `label` | Per-deposit identifier the ASP curates: `label = Poseidon(scope, depositIndex)` |
| `scope` | Pool-binding constant: `scope = keccak256(poolAddress ‖ chainId) mod p` |
| `depositIndex` | The deposit's sequential index (= its state-tree leaf index) |

The depositor supplies only `precommitment` and `value` on deposit. The contract assigns
`label` (it owns `scope` and the deposit counter), computes `commitment`, inserts it into
the **state tree**, and emits `Deposit(commitment, label, value, leafIndex)`. The depositor
records `(value, label, nullifier, secret)` to spend later.

## 2. Trees

| Tree | Leaves | Maintained by | Depth |
|---|---|---|---|
| **State tree** | every `commitment` (deposits + withdrawal remainders) | the pool contract (incremental Merkle, append-only) | 20 |
| **Association tree (ASP)** | the `label`s of deposits in the "clean" set | the Association Set Provider, off-chain; root posted on-chain | 20 |

Both use Poseidon(2) hashing, depth 20 (~1.05M leaves), zero-subtree precomputation for
empty nodes. The pool stores a bounded history of recent state roots (so a proof against a
slightly stale root still verifies) and the current ASP root (updated by the ASP authority;
testnet: a single authorized address — production decentralises this).

## 3. Nullifier

```
nullifierHash = Poseidon(nullifier)
```

Consumed on withdrawal (recorded in the pool's nullifier set) to prevent double-spend. The
circuit proves the **same** `nullifier` appears in both `precommitment = Poseidon(nullifier,
secret)` and `nullifierHash = Poseidon(nullifier)`, without revealing `nullifier`. Domain:
the reserved `pool` nullifier domain of [nullifier-registry.md](./nullifier-registry.md).

## 4. Withdrawal (partial)

A withdrawal spends a commitment, pays out `withdrawnValue`, and re-deposits the remainder
as a fresh commitment **with the same label** (so the remainder stays in the association
set). Full withdrawal is the case `withdrawnValue == value` (remainder commitment of value 0
is still inserted and is unspendable by construction; the SDK omits recording it).

```
remainder       = value − withdrawnValue            (range-checked, no underflow)
newPrecommitment = Poseidon(newNullifier, newSecret)
newCommitment    = Poseidon(remainder, label, newPrecommitment)
```

### 4.1 `withdrawal.circom` — the on-chain proof

Proves, in one Groth16 proof:

1. `commitment = Poseidon(value, label, Poseidon(nullifier, secret))` is a member of the
   state tree at `stateRoot` (depth-20 Merkle inclusion).
2. `label` is a member of the association tree at `aspRoot` (depth-20 Merkle inclusion).
   **This is the compliance statement** — the withdrawer's deposit is in the clean set.
3. `nullifierHash = Poseidon(nullifier)` (bound to the same `nullifier` as the commitment).
4. Value accounting: `withdrawnValue` and `value` are `< 2¹²⁸`, `withdrawnValue ≤ value`,
   `remainder = value − withdrawnValue`, and `newCommitment = Poseidon(remainder, label,
   Poseidon(newNullifier, newSecret))`.
5. `context` is bound into the proof (a dummy constraint), committing the proof to the
   withdrawal's recipient / fee / processooor so a relayer cannot redirect funds
   (front-running / malleability protection).

| Signal | Visibility | Notes |
|---|---|---|
| `value`, `label`, `nullifier`, `secret` | private | the spent commitment's openings |
| `newNullifier`, `newSecret` | private | the remainder commitment's openings |
| `stateSiblings[20]`, `stateIndex[20]` | private | state-tree path |
| `aspSiblings[20]`, `aspIndex[20]` | private | association-tree path |
| `withdrawnValue` | **public** | amount paid out |
| `stateRoot` | **public** | a recent pool state root |
| `aspRoot` | **public** | the ASP root the proof attests against |
| `nullifierHash` | **public** | consumed on-chain |
| `newCommitment` | **public** | inserted into the state tree |
| `context` | **public** | `keccak256(abi.encode(withdrawal, scope)) mod p` |

Public-signal order (snarkjs): `withdrawnValue, stateRoot, aspRoot, nullifierHash,
newCommitment, context`.

### 4.2 `association.circom` — standalone ASP membership

A minimal circuit proving `label ∈ association tree(aspRoot)` (public: `aspRoot`, `label`).
It is **not** used on-chain — association membership is enforced inside `withdrawal.circom`
(single sound proof, no cross-proof binding gap). `association.circom` is the building block
ASP tooling uses to issue/verify set-membership statements off-chain and documents the
sub-statement in isolation.

## 5. Contract interface (`IOpaquePrivacyPool`)

```solidity
event Deposit(bytes32 indexed commitment, uint256 label, uint256 value, uint32 leafIndex);
event Withdrawal(bytes32 indexed nullifierHash, bytes32 newCommitment, uint256 withdrawnValue, address recipient);
event ASPRootUpdated(uint256 newRoot);

function deposit(uint256 precommitment) external payable;            // value = msg.value
function withdraw(WithdrawProof calldata proof, Withdrawal calldata w) external;
function latestRoot() external view returns (uint256);
function isKnownRoot(uint256 root) external view returns (bool);
function isSpent(bytes32 nullifierHash) external view returns (bool);
function aspRoot() external view returns (uint256);
```

`withdraw` recomputes `context` from `w` (recipient, fee, relayer/processooor) and `scope`,
verifies the Groth16 proof against `(withdrawnValue, stateRoot, aspRoot, nullifierHash,
newCommitment, context)`, requires `isKnownRoot(stateRoot)` and `aspRoot == currentAspRoot`,
rejects a spent `nullifierHash`, inserts `newCommitment`, records the nullifier, and pays
`withdrawnValue − fee` to `recipient` (`fee` to the relayer/processooor). Deposits are
fixed-asset native value in v1; ERC-20 pools are a v2 extension.

## 6. Flow

```
deposit:   user picks nullifier,secret → precommitment → deposit(precommitment){value}
           contract: label=Poseidon(scope,idx); commitment=Poseidon(value,label,precommitment)
                     → insert into state tree → emit Deposit
ASP:       provider reviews the deposit; if clean, adds `label` to the association tree,
           posts the new aspRoot on-chain
withdraw:  user proves commitment∈state ∧ label∈ASP ∧ nullifier ∧ value-accounting,
           to a FRESH stealth address (CSAP) → contract pays out, inserts remainder
```

The withdrawal recipient is a fresh stealth address (CSAP §2), so amount privacy composes
with recipient privacy: neither the payer↔payee link nor the deposit↔withdrawal link is
on-chain.

## 7. Security considerations

- **Soundness rests on the trusted setup.** v1 uses the repo's `pot16` Powers-of-Tau plus a
  circuit-specific Phase-2 contribution; a production deployment requires a multi-party
  ceremony (tracked in the audit/legal gates, §8).
- **Association membership is enforced in the withdrawal proof**, not a separate proof, to
  avoid a cross-proof binding gap.
- **`context` binding** prevents a relayer from redirecting the payout or altering the fee.
- **Nullifier consume-once** prevents double-spend; the same `nullifier` is bound across the
  commitment and `nullifierHash` so a depositor cannot withdraw twice with different hashes.
- **Range checks** (`< 2¹²⁸`) on `value`/`withdrawnValue` prevent field-wraparound in the
  remainder subtraction.
- **ASP trust (v1):** a single authorized address posts the ASP root on testnet. This is a
  liveness + curation trust point, decentralised before any mainnet step. The pool's
  *integrity* (no theft, no double-spend) does not depend on the ASP; only *which deposits
  can withdraw* does.
- **Regulatory surface.** This is the highest-regulatory-surface component; mainnet is gated
  on circuit + contract audits and a legal opinion (§8). Testnet only here.

## 8. Audit & legal gates (prerequisite for anything beyond testnet)

- [ ] Circom circuit audit (Veridise / zkSecurity / Hexens)
- [ ] Solidity + Anchor contract audit (OtterSec / Neodyme / Trail of Bits)
- [ ] Production trusted-setup ceremony (multi-party Phase 2)
- [ ] Legal opinion + `DISCLAIMER.md` distinguishing from a mixer
- [ ] EF Security grant application for audit funding

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).
