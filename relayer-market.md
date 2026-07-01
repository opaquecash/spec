# Relayer Market — gas-private submission

**Status:** Draft v1 (testnet) · **License:** CC0-1.0 · **Transport:** libp2p GossipSub + HTTP intake

> The relayer market solves gas-leak deanonymisation: submitting an on-chain action
> (a PSR reputation proof, a future pool withdrawal) from the user's own wallet links
> that wallet to the action. In the market, the user commits to a hidden payload,
> staked relayers compete to submit it, the winner executes it through an on-chain
> escrow that pays the fee, and a relayer that accepts a job and fails to deliver is
> slashed. No central coordinator; anyone can run a node.

## Abstract

Each chain hosts one `RelayerRegistry` contract/program combining the **stake
registry** and the **job escrow**, so the submit-or-slash condition is enforceable
on-chain with no out-of-band "failure proof". Off-chain, relayer nodes form a libp2p
GossipSub mesh on the topic `opaque/jobs/v1`. A user advertises a job (the payload
hash, never the payload), collects bids, picks a winner weighted by on-chain stake,
and sends the payload **encrypted to the winner's key**. The winner bonds part of its
stake by accepting the job on-chain, then submits; the escrow verifies the payload
against the commitment, executes it, and releases fee + bond. A missed deadline lets
the job creator claim the bond.

```
User                          Relayer mesh (GossipSub opaque/jobs/v1)         Chain
 │ createJob(fee, payloadHash, deadline) ────────────────────────────────────► escrow
 │ advert {jobId, chain, fee} ──► any node's HTTP gateway ──► gossip ──► all nodes
 │ ◄─ bids {jobId, operator, x25519Pk, sig} ◄─────────────────────────────────┤
 │ pick winner (stake-weighted, reads stakes on-chain)                        │
 │ payload encrypted to winner (NaCl box) ──► gateway ──► gossip ──► winner   │
 │                                            winner: acceptJob (bond) ──────► escrow
 │                                            winner: submitJob(payload) ────► escrow executes, pays
 │ (deadline passed, no submit?) slashJob ───────────────────────────────────► bond → user
```

## 1. Roles and trust

| Role | Trust |
|---|---|
| Relayer node | **Liveness + censorship only.** It cannot forge or alter the payload (the escrow checks the hash) and cannot take the fee without executing (payment is atomic with execution). Accepting then stalling costs the bond. |
| HTTP gateway (any node) | Intake convenience only; it re-gossips adverts/bids/payloads. A censoring gateway is bypassed by using any other node. The encrypted payload is opaque to it. |
| Job creator | Funds the escrow. The funding transaction is on-chain and SHOULD come from an address unlinked to the user's main identity (v1 limitation; see §8). |

## 2. On-chain: `RelayerRegistry`

One deployment per chain (Ethereum Sepolia contract; Solana devnet Anchor program).

### 2.1 Relayer registration

- `register(x25519PubKey, endpoint)` + stake deposit (native asset). `x25519PubKey`
  (32 bytes) is the encryption key bids advertise; `endpoint` is an optional HTTP
  gateway URL (UTF-8, MAY be empty).
- Stake is split into **free** and **bonded**. Bids are only credible up to free stake.
- `addStake()`, `requestUnstake(amount)` → `withdraw()` after the **unstake cooldown**
  (testnet: 1 hour). Cooldown prevents bid-then-unstake races.
- `MINIMUM_STAKE` (testnet: 0.01 ETH / 0.1 SOL) gates registration.

### 2.2 Job lifecycle

| Step | Who | Semantics |
|---|---|---|
| `createJob(jobId, payloadHash, deadline)` + fee | creator | Escrows the fee. `jobId` MUST be unique; `deadline` is a unix timestamp. |
| `acceptJob(jobId)` | registered relayer | Bonds `bond = fee` from free stake (the relayer's skin in the game). One relayer per job; first valid accept wins. MUST be rejected after `deadline`. |
| `submitJob(jobId, payload…)` | the accepting relayer | Verifies the payload against `payloadHash`, executes it, then atomically pays `fee` to the relayer and releases the bond. |
| `slashJob(jobId)` | creator, after `deadline` | Only if accepted but never submitted: bond → creator, fee refunded. |
| `cancelJob(jobId)` | creator, after `deadline` | Only if never accepted: fee refunded. |

### 2.3 Payload commitment

- **Ethereum:** `payloadHash = keccak256(abi.encode(target, calldata))`. `submitJob`
  performs `target.call(calldata)` (no value) with the escrow as `msg.sender` and
  reverts if the inner call reverts. Targets are therefore limited to functions that
  are permissionless w.r.t. caller (PSR `verifyReputation`, announcer `announce`,
  UAB receivers all qualify).
- **Solana:** `payload_hash = keccak256(program_id ‖ u32_le(n_accounts) ‖
  accounts[(pubkey ‖ is_signer ‖ is_writable)…] ‖ data)`. `submit_job` re-derives the
  hash from the instruction descriptor + remaining accounts and CPIs the inner
  instruction. `is_signer` MUST be false for every inner account (the escrow signs
  nothing on the inner instruction's behalf).

## 3. Wire formats (GossipSub topic `opaque/jobs/v1`)

All messages are JSON objects with a `t` tag. Chain ids use the Wormhole convention
(Ethereum = 2, Solana = 1) for consistency with [UAB.md](./UAB.md).

### 3.1 Job advert (user → mesh)

```json
{ "t": "advert", "v": 1, "jobId": "0x…32", "chain": 2,
  "fee": "1000000000000000", "deadline": 1781200000,
  "payloadHash": "0x…32" }
```

No payload bytes, no user identity. The advert is unauthenticated by design — the
escrow entry (`jobId` on-chain) is the source of truth; nodes MUST verify the job
exists, is unaccepted, and matches `fee`/`deadline`/`payloadHash` before bidding.

### 3.2 Bid (relayer → mesh)

```json
{ "t": "bid", "v": 1, "jobId": "0x…32", "chain": 2,
  "operator": "0x… | base58", "x25519Pk": "0x…32",
  "sig": "0x… | base58" }
```

`sig` is the operator key's signature over `keccak256("opaque-relayer-bid-v1" ‖ jobId
‖ x25519Pk)` (EVM: EIP-191 personal-sign by the registered operator address; Solana:
ed25519 by the registered operator pubkey). Users MUST check the operator is
registered, has free stake ≥ fee, and that `x25519Pk` matches the registry before
selecting. Selection policy is client-side; the reference implementation weights
uniformly at random among the top bids by free stake.

### 3.3 Payload delivery (user → winner)

```json
{ "t": "payload", "v": 1, "jobId": "0x…32",
  "to": "<winner x25519Pk>", "box": "<base64>" }
```

`box = epk(32) ‖ nonce(24) ‖ ciphertext` — NaCl `crypto_box`
(x25519-xsalsa20-poly1305) from a **fresh ephemeral sender key** to the winner's
registered x25519 key. The plaintext is the chain-specific payload:

- Ethereum: `abi.encode(target, calldata)` (exactly the `payloadHash` preimage)
- Solana: the §2.3 descriptor bytes

Only the winner can decrypt; other nodes and gateways see random bytes.

### 3.4 HTTP gateway intake

Every node SHOULD expose: `POST /v1/jobs` (advert), `GET /v1/jobs/{jobId}/bids`,
`POST /v1/jobs/{jobId}/payload`. The gateway publishes received messages to the
gossip topic and serves bids it has seen. Browsers and SDKs without a libp2p stack
use any gateway; nodes never trust gateway data beyond re-gossiping it.

Nodes that serve §9 fee-in-token sweeps additionally expose `POST /v1/sweep`
(§9.4) and `GET /v1/sweep/info`. Unlike the job routes these are not re-gossiped:
a sweep is a direct, synchronous request to *this* node to front gas.

## 4. Relayer node behaviour

1. Subscribe to `opaque/jobs/v1`; serve the HTTP gateway.
2. On advert: verify the on-chain job (exists, unaccepted, fee ≥ `--min-fee`,
   deadline sane, free stake ≥ fee) → publish bid.
3. On payload addressed to own x25519 key: decrypt, re-derive `payloadHash`, simulate
   the inner call; if it would succeed → `acceptJob` (bond) → `submitJob`.
   A node MUST NOT accept before it can decrypt and validate the payload — accepting
   blind risks the bond on an unsubmittable job.
4. **Delivery duty (UAB/ONS):** nodes poll Wormholescan for the Opaque emitters
   ([UAB.md](./UAB.md), [ONS.md](./ONS.md)) and deliver signed VAAs to the
   destination receivers directly (fee-less keeper work in v1). This replaces the
   Phase-1 central relay: the same binary, run by anyone, provides the liveness.

## 5. Fee model (v1)

Fees are native-asset, pre-funded into the escrow by `createJob`. The funding
transaction is a separate, earlier transaction and SHOULD be made from an address
not linked to the user's primary wallet. **Fee-in-proof** (the fee released by a
public signal inside the ZK proof itself) is the endgame and is deferred; the escrow
interface is designed so a future `submitJob` variant can release a proof-committed
fee without changing the market protocol.

**Fee-in-token sweep.** For the specific case of moving an ERC-20/SPL balance out of a
one-time stealth address that holds the token but no native gas, §9 defines a
complementary mechanism in which the relayer is paid its fee *in the swept token* and no
native-asset escrow is needed. It is escrow-free and does not use `createJob`/`acceptJob`;
it composes with the market (a registered relayer MAY offer it) but is independent of it.

## 6. Slashing

v1 slashes exactly one offence — *accepted but did not submit by the deadline* —
because it is the only offence the chain can verify without trusting anyone. The
bond equals the fee, so griefing a job costs the relayer what the user offered for
it. Stake-weighted selection makes Sybil bidding unattractive: free stake is checked
at bid time and bonded at accept time.

## 7. Reference deployments (testnet)

| Chain | Address / Program id |
|---|---|
| Ethereum Sepolia `RelayerRegistry` | see `@opaquecash/deployments` |
| Solana devnet `relayer-registry` | see `@opaquecash/deployments` |

Constants (testnet): `MINIMUM_STAKE` 0.01 ETH / 0.1 SOL · unstake cooldown 1 h ·
bond = fee.

## 8. Security considerations and v1 limitations

- **Funding linkage.** `createJob` is an on-chain transaction; an observer can link
  the funding address to the job (not to the payload content, which is hidden until
  submission). Users SHOULD fund from unlinked addresses. Fee-in-proof removes this
  (§5).
- **Payload privacy window.** The payload becomes public at `submitJob` (it executes
  on-chain). The market hides the *submitter identity*, not the action itself.
- **Front-running `acceptJob`.** A relayer that did not receive the payload gains
  nothing by accepting (it cannot submit and will be slashed); the bond makes the
  race self-defeating.
- **Gateway censorship** is a liveness nuisance only; use another node.
- **Unauthenticated adverts** cannot grief relayers: bidding is free, and nodes
  verify jobs on-chain before bonding anything.
- **Replay.** `jobId` uniqueness is enforced on-chain; bids bind to `jobId` and the
  registered operator key.

## 9. Fee-in-token sweep (gasless token withdrawal)

A one-time stealth address that received an ERC-20/SPL payment holds the token but no
native asset, so its owner cannot pay gas to move it. This section defines how a relayer
moves the token on the owner's behalf and is reimbursed *in that token* — no native-asset
escrow, and the relayer (not the owner) supplies gas. The owner authorizes everything with
the reconstructed one-time stealth key, entirely offline; the relayer cannot redirect funds
or inflate its cut because destination, amount, and fee are all signed.

### 9.1 Ethereum: `StealthTokenSweep` forwarder

A stateless forwarder verifies an owner-signed authorization and moves the token directly
from the owner to the destination and the relayer; it never custodies funds.

- **Authorization (EIP-712).** Domain `{name: "OpaqueStealthTokenSweep", version: "1",
  chainId, verifyingContract: forwarder}`. Type:
  ```
  Sweep(address token,address owner,address destination,uint256 value,uint256 fee,uint256 nonce,uint256 deadline)
  ```
  The owner is the stealth address; `nonce` is the forwarder's per-owner counter; `fee` MUST
  be `<= value`. The signature MUST recover to `owner`.
- **Allowance.** The forwarder pulls `value` via `transferFrom`. Allowance is granted in the
  same transaction by an EIP-2612 `permit` signed by the owner (`sweepWithPermit`), or by a
  prior `approve` (`sweep`). Tokens without EIP-2612 and without a prior allowance cannot use
  this path.
- **Settlement.** The submitter (any address; typically a relayer) receives `fee`; the
  destination receives `value - fee`. `msg.sender` earns the fee, so no fee recipient is
  signed. The per-owner `nonce` is consumed before any external call (replay-safe); `deadline`
  bounds validity.

### 9.2 Solana: relayer as fee payer

Solana lets the fee payer differ from the instruction's signing authority, so no forwarder
program is needed. The owner builds an SPL transfer (stealth ATA -> destination ATA, plus an
optional fee transfer stealth ATA -> relayer ATA, and an optional `closeAccount` reclaiming
rent to the fee payer), sets the relayer as `feePayer`, and partially signs as the token
authority with the reconstructed stealth keypair. The relayer adds its fee-payer signature,
pays the network fee, and broadcasts. The relayer's reimbursement is the fee-transfer
instruction; the owner signs the whole message, so the relayer cannot alter it.

The relayer MUST co-sign over the transaction's original `recent_blockhash`
(re-fetching one would invalidate the owner's partial signature), and SHOULD refuse
to front gas for a transaction that (a) does not name it as fee payer, or (b) touches
any program outside the token set (SPL Token, Token-2022, Associated Token Account,
Compute Budget) — the whitelist bounds the compute a stranger can bill to the node.

### 9.3 Trust and limitations

This mechanism is permissionless and trust-minimized for the *owner*: a relayer can only
execute exactly what was signed, or nothing. It has no bond or slashing (none is needed —
there is no escrow at risk and the relayer is paid only on success). Liveness is best-effort:
if no relayer submits before `deadline`, the authorization simply expires and may be
re-issued. The token transfer and its ATAs are visible on-chain (CSAP provides recipient
unlinkability, not amount privacy).

### 9.4 Submission interface

A node accepting sweeps exposes on its §3.4 gateway:

- `POST /v1/sweep` — body is tagged by chain:
  `{"chain": "ethereum", "to": <forwarder>, "data": <calldata>}` or
  `{"chain": "solana", "transactionBase64": <partially-signed tx>}`.
  The node validates its guards (EVM: `to` equals its configured forwarder;
  Solana: §9.2 fee-payer and program checks), fronts the gas, and answers
  `{"ok": true, "tx": <id>}` on confirmation or `{"ok": false, "error": …}`.
- `GET /v1/sweep/info` — `{"chains": [{"chain", "operator", "forwarder"?}]}`:
  per-chain operator identity (the Solana fee payer an owner MUST name when
  building the transaction) and, on EVM, the forwarder the node will call.

The interface is synchronous because the authorization is self-contained; there is
no advert, bid, or payload-delivery round. Whether the in-token `fee` covers the
fronted gas is the node's own pricing decision (§9.3); nodes SHOULD reject sweeps
whose fee they consider under-priced rather than subsidize them.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).
