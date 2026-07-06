# ONS — Opaque Name Service

**Status:** Draft v1 · **License:** CC0-1.0 · **Layer:** naming / UX (non-core)

> ONS gives a stealth meta-address a human-readable name that resolves on **both**
> Ethereum and Solana: `alice.opq.eth` returns the same CSAP meta-address to a MetaMask
> sender (on-chain ENS wildcard resolution) and to a Phantom sender (on-chain mirror PDA,
> no Ethereum RPC). One canonical record, one claim, no central server. ONS is a
> convenience layer over [CSAP.md]; it adds no privacy properties and no new trust beyond
> the Wormhole guardian set already used by the [UAB](./UAB.md).

## Abstract

A CSAP meta-address is 66 bytes (132 hex chars) — too long to share by hand. ENS resolves
`.eth` only from Ethereum; SNS resolves `.sol` only from Solana. ONS bridges the gap with a
**canonical registry on Ethereum** (an ENSIP-10 wildcard resolver serving `*.opq.eth`
subnames, sourcing meta-addresses from the deployed ERC-6538 registry) and a **read-only
mirror on Solana** (PDAs written exclusively by Wormhole VAA from the canonical registry).
Names can be claimed from either chain; Ethereum is always authoritative.

**Consistency model (normative): eventually consistent, canonical-chain-wins.** There is
**no atomic cross-chain claim** — cross-chain messaging is asynchronous and best-effort,
and the Ethereum leg finalizes in ~19 minutes. A claim made on Ethereum is immediately
authoritative. A claim made from Solana is *provisional* until the canonical registry
confirms it, and **loses to any concurrent direct Ethereum registration**. Implementations
MUST NOT present a provisional claim as owned (§6).

## 1. Names

### 1.1 Namespace

The production namespace is `*.opq.eth` (an ENS subname tree — a native `.opq` TLD is not
mintable on ENS). The parent name is a deployment parameter: testnet deployments develop
against a Sepolia-registered test name, and the parent in force is published in
`@opaquecash/deployments`. Everything below writes `opq.eth` for readability.

### 1.2 Labels

A v1 label (the `alice` in `alice.opq.eth`) MUST match `[a-z0-9-]{1,63}` and MUST NOT start
or end with `-`. Resolvers MUST lowercase input before hashing. Unicode normalization
(ENSIP-15) is out of scope for v1; non-LDH labels are rejected at registration.

### 1.3 Hashes

Two hashes of the same name are used, for the two chains' native keying:

| Hash | Definition | Used by |
|---|---|---|
| `node` | ENS namehash: `keccak256(parentNode ‖ keccak256(label))` | Ethereum registry storage; ENSIP-10 resolution |
| `name_hash` | `keccak256(utf8(fullName))`, e.g. `keccak256("alice.opq.eth")` | Solana mirror PDA seeds |

`fullName` is the lowercase dot-joined full name including the parent.

## 2. Canonical registry — Ethereum

`OpaqueNameRegistry.sol` is simultaneously the subname registrar and the ENSIP-10 wildcard
resolver for the parent name. The parent name's ENS resolver record is set to this contract.

### 2.1 Records

```
struct Record {
    address registrant;     // ETH controller; for Solana-originated claims, a
                            // deterministic surrogate (§4.2)
    bytes32 solAuthority;   // claimer's Solana pubkey (zero if none)
    bytes   spendPubKey;    // 33-byte compressed secp256k1 (empty => source ERC-6538)
    bytes   viewPubKey;     // 33-byte compressed secp256k1
    uint64  updatedAt;
}
```

Registration is first-come-first-served, free at the contract level (gas + Wormhole fee
only). Only the registrant may update or transfer. If `spendPubKey` is empty, the resolver
sources the meta-address live from the **ERC-6538 `StealthMetaAddressRegistry`**
(`stealthMetaAddressOf(registrant, schemeId=1)`) — names track the registrant's registry
entry with no second write. An explicit record overrides the registry-sourced one.

### 2.2 Resolution (ENSIP-10)

The contract implements `resolve(bytes name, bytes data)` (interface id `0x9061b923`) over
the DNS-encoded name, answering for any depth-1 subname of the parent:

- `text(node, "com.opaque.meta")` → the CSAP §2.9 record value: `st:opq:` + the
  `0x`-prefixed 132-hex-char `V‖S` serialisation;
- `addr(node)` → the registrant address (zero for Solana-surrogate registrants).

Resolvers MUST validate both 33-byte halves are valid compressed secp256k1 points before
use (CSAP §2.9). Unregistered names revert / return empty.

### 2.3 Wormhole emit

Every state change (register, update, transfer, revoke) publishes an **ONS mirror payload**
(§5.1) through the Wormhole Core Contract at `consistencyLevel = 200` (finalized), exactly
as the UAB sender does. The registry contract address (left-padded to 32 bytes) is the ONS
Ethereum emitter.

## 3. Mirror — Solana

`OpaqueNameMirror` (Anchor) holds one PDA per name:

```
seeds = ["ons_mirror", name_hash]      // name_hash per §1.3

pub struct OnsRecord {
    pub name_hash: [u8; 32],
    pub spend_pubkey: [u8; 33],
    pub view_pubkey: [u8; 33],
    pub eth_owner: [u8; 20],
    pub sol_authority: Pubkey,         // zero if none
    pub wormhole_sequence: u64,        // last applied sequence
    pub updated_at: i64,
    pub bump: u8,
    pub revoked: bool,                 // true = tombstone (keys zeroed, floor retained)
}
```

Mirror PDAs are written **only** by `receive_record(posted_vaa)`, which MUST verify the
VAA emitter is the configured `(chainId = 2, OpaqueNameRegistry)` pair and MUST reject a
sequence ≤ the stored `wormhole_sequence` (stale/replayed update). There is no
direct-write path; the mirror is read-only from Solana's perspective.

**Revoke tombstones, never closes (OPQ-004).** A revoke MUST NOT close the PDA: closing
would destroy `wormhole_sequence`, letting a later-delivered but lower-sequence upsert VAA
re-create the record fresh and resurrect the name at stale keys (the monotonic floor is
only skipped for a genuinely new PDA, `name_hash == 0`). Instead a revoke updates the
record in place — advancing `wormhole_sequence`, keeping `name_hash`, zeroing
`spend_pubkey`/`view_pubkey`/`eth_owner`/`sol_authority`, and setting `revoked = true` — so
the floor survives. Resolvers MUST treat a `revoked` record as unresolved (equivalent to no
record).

**Resolution from Solana:** the SDK derives the PDA from `name_hash` client-side and reads
one account — no Ethereum RPC, no `getProgramAccounts` scan, no gateway.

**Staleness:** a mirror record lags the canonical record by Wormhole end-to-end latency
(Ethereum finality ~19 min + guardian signing + relay). Clients that can reach an Ethereum
RPC MAY prefer canonical resolution; mirror-only clients accept the lag (§7).

## 4. Claims

### 4.1 From Ethereum (authoritative immediately)

`register(label, spendPubKey, viewPubKey)` on the registry. The record is live for ENS
resolution in the same block; the mirror PDA follows after relay. One transaction.

### 4.2 From Solana (provisional)

`OpaqueNameRegistration` (Anchor) lets a Solana-only user claim without touching Ethereum:

1. `claim(label, spend_pubkey, view_pubkey)` creates a **provisional PDA**
   (`seeds = ["ons_claim", name_hash]`, payer = claimer) and CPIs the Wormhole Core
   `post_message` with an **ONS claim payload** (§5.2). Duplicate provisional claims for
   the same `name_hash` are rejected while one is open.
2. The relay delivers the VAA to `OpaqueNameRegistry.registerFromVAA(vaa)`, which verifies
   the emitter is the registration program's emitter PDA (`seeds = ["emitter"]`,
   chainId = 1), validates the label, and — **if the name is still free** — registers it.
   The registrant is the deterministic surrogate
   `address(uint160(uint256(keccak256(sol_authority))))`; `solAuthority` is stored, and
   subsequent updates for that name are accepted only via VAA from the same authority.
   If the name was taken in the interim, the VAA is consumed with **no effect** (the
   canonical chain wins) — the claim is lost.
3. Registration (like any registration) emits the mirror payload. **The mirror PDA is the
   confirmation signal**: a mirror record for `name_hash` with `sol_authority` equal to
   the claimer upgrades the claim to confirmed; one with a different owner proves the
   claim lost. `reconcile(name_hash)` on the registration program closes the provisional
   PDA against the mirror state (or unconditionally for the claimer after the pending
   window, §6) and refunds rent to the claimer.

## 5. Wormhole payloads

ONS payloads travel on dedicated emitters (the registry contract and the registration
program), so they are disambiguated from the 96-byte UAB payment payload by emitter, never
by sniffing. Both carry a leading version byte. Multi-byte integers are big-endian.

### 5.1 ONS mirror payload — Ethereum → Solana (164 bytes, fixed)

| Offset | Size | Field | Description |
|---:|---:|---|---|
| 0   | 1  | `version`       | `1` |
| 1   | 1  | `action`        | `1` = upsert, `2` = revoke |
| 2   | 32 | `name_hash`     | §1.3 |
| 34  | 33 | `spend_pubkey`  | compressed secp256k1 (zeroed on revoke) |
| 67  | 33 | `view_pubkey`   | compressed secp256k1 (zeroed on revoke) |
| 100 | 32 | `eth_owner`     | registrant, left-padded 20-byte address |
| 132 | 32 | `sol_authority` | claimer Solana pubkey; zero if none |

### 5.2 ONS claim payload — Solana → Ethereum (101 + L bytes)

| Offset | Size | Field | Description |
|---:|---:|---|---|
| 0   | 1  | `version`       | `1` |
| 1   | 1  | `action`        | `1` = claim |
| 2   | 32 | `sol_authority` | claimer Solana pubkey (must equal the claim signer) |
| 34  | 33 | `spend_pubkey`  | compressed secp256k1 |
| 67  | 33 | `view_pubkey`   | compressed secp256k1 |
| 100 | 1  | `label_len`     | `L`, 1–63 |
| 101 | L  | `label`         | UTF-8, §1.2 charset |

Receivers MUST enforce emitter allowlists and `(emitterChain, emitterAddress, sequence)`
replay protection exactly as specified in [UAB.md].

## 6. Reconciliation UX (normative for clients)

A Solana-originated claim moves through four states; clients MUST track and surface them:

| State | On-chain condition | Client behaviour |
|---|---|---|
| `pending` | provisional PDA exists; no mirror PDA for `name_hash` | Show "pending confirmation (~20–40 min)". The name MUST NOT resolve and MUST NOT be shown as owned. |
| `confirmed` | mirror PDA exists, `sol_authority` = claimer | Show owned. Offer `reconcile` to reclaim provisional rent. |
| `lost` | mirror PDA exists, owner ≠ claimer — a direct Ethereum registration won the race | Show "name taken on the canonical chain". Offer `reconcile` (rent refund) and a fresh-name retry. |
| `expired` | no mirror PDA after the **pending window of 24 h** | Treat as delivery failure (relay outage / VAA never submitted). Offer `reconcile`; the user may retry the claim. |

Resolution NEVER serves provisional claims: senders only ever see canonical (ENS) or
mirrored (PDA) records. The pending window exists purely so the claimer's rent is not
locked forever behind a dead relay; closing an expired claim does not release the name on
Ethereum if the VAA later arrives (the registry registers it regardless — the claimer can
re-create the provisional PDA and reconcile to `confirmed`).

## 7. Resolution paths (SDK)

`resolveOpaqueMetaAddress(name)` in `@opaquecash/opaque` tries, in order of input shape:

| Input | Path |
|---|---|
| `alice.opq.eth` (parent in force) | Mirror PDA first (cheap, no ETH RPC); ENS wildcard resolution as fallback / freshness override |
| any other `*.eth` | ENS `text(node, "com.opaque.meta")` per CSAP §2.9 |
| `*.sol` | SNS record `com.opaque.meta` (Records V2 / TXT) |
| raw meta-address | accepted directly |

All paths MUST point-validate both 33-byte halves (CSAP §2.9). Where canonical and mirror
disagree (staleness window), the canonical record wins; mirror-only clients accept the lag
bounded by Wormhole end-to-end finality.

## 8. Decentralisation and trust

- **Parent name custody:** `opq.eth` is held by the protocol multisig; registration and
  resolution logic is in the registry contract, not the key. (Testnet: deployer-held test
  name.)
- **Guardian trust:** mirror integrity rests on Wormhole's 13/19 threshold — the same
  assumption the UAB already makes. If Wormhole halts, each chain's existing records keep
  resolving; only sync is delayed.
- **No gateway:** both resolution paths are pure on-chain reads. CCIP Read is not used.
- **Squatting:** first-come-first-served, canonical-chain-wins. No atomicity is claimed;
  see §4.2/§6 for the race window and its honest handling.

## 9. Security considerations

- The registry MUST restrict updates to the registrant (or, for surrogate registrants,
  to VAAs from the recorded `sol_authority`).
- The mirror MUST reject non-allowlisted emitters and non-increasing sequences; the
  registry MUST consume each claim VAA at most once.
- ONS adds no privacy surface: a name maps to a *meta*-address, which is already public
  in the ERC-6538 registry; payments to it remain unlinkable per CSAP.
- A malicious relay can only delay (liveness), never forge or alter (guardian-signed
  payloads) — identical to the UAB trust model.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

[CSAP.md]: ./CSAP.md
