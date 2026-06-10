# CSAP — Cross-chain Stealth Address Protocol

**Status:** Draft · **Version:** 1 · **Scheme id:** 1 (secp256k1 + Keccak-256 view tags)
**Reference implementations:** `opaquecash/ethereum`, `opaquecash/solana` · **License:** CC0-1.0

> This document specifies the stealth-address payment layer that Opaque implements identically on Ethereum and Solana. It documents behaviour that is already deployed and live (Ethereum Sepolia, Solana Devnet), not a future design. The Programmable Stealth Reputation layer is specified separately in `PSR.md`; the cross-chain announcement transport in `UAB.md`.

---

## Abstract

CSAP defines a dual-key stealth address protocol (DKSAP) over the secp256k1 curve that lets a sender pay a recipient at a fresh, unlinkable on-chain address without any interaction, and lets the recipient discover those payments by scanning public announcements with a viewing key. The scheme is byte-compatible with [EIP-5564] announcements, [ERC-6538] registration, and the EIP-5564 view-tag optimisation, and it derives identical key material and identical stealth identifiers on both Ethereum and Solana — so a single scanner, a single meta-address, and a single key set work across both chains. Where Opaque deviates from EIP-5564 (notably the byte order of the serialised meta-address), this document states the deviation explicitly.

## Motivation

EIP-5564 standardised stealth addresses for Ethereum but stops at one chain, and no equivalent standard exists on Solana. A user who wants private receipt on both ecosystems must today run two unrelated systems with two key sets and two scanners. CSAP specifies one protocol that:

1. Produces the **same meta-address and the same per-payment stealth identifier on both chains**, so one scan covers both.
2. Requires **no server and no trusted party** — keys come from a wallet signature via HKDF, and proving/scanning are client-side.
3. Is **backwards compatible** with EIP-5564 / ERC-6538 tooling at the event, registry, and view-tag level, so existing Ethereum infrastructure (indexers, wallets) interoperates.
4. Gives wallets a single integration target for cross-chain private payments.

## Specification

The key words MUST, MUST NOT, SHOULD, and MAY are to be interpreted as in RFC 2119.

### Notation and constants

| Symbol | Meaning |
|---|---|
| `G`, `n` | secp256k1 generator and curve order |
| `v`, `s` | viewing / spending private keys (32-byte scalars in `[1, n-1]`) |
| `V = v·G`, `S = s·G` | viewing / spending public keys (compressed, 33 bytes) |
| `r`, `R = r·G` | ephemeral private / public key, fresh per payment (`R` compressed, 33 bytes) |
| `Keccak256` | Keccak-256 (the EVM hash), used for the shared-secret hash and address |
| `SHA-256` | used only inside HKDF for key derivation |
| `‖` | byte concatenation |
| `DOMAIN` | the ASCII string `"opaque-cash-v1"` |

Constants fixed by this version:
- HKDF info string: `DOMAIN = "opaque-cash-v1"`.
- Scheme id `1` = secp256k1 keys with Keccak-256 view tags. (On Ethereum the on-chain type is `uint256`; on Solana it is `u64`. The encoded value is identical.)
- Meta-address length: `66` bytes (`33 + 33`).
- Stealth identifier length: `20` bytes (an EVM-style address; see §2.3).

### 2.1 Meta-address format

A stealth meta-address is the concatenation of two compressed secp256k1 public keys:

```
metaAddress = compressed(V) ‖ compressed(S)        // 33 + 33 = 66 bytes
              ^ viewing pubkey   ^ spending pubkey
```

It is serialised as a `0x`-prefixed hex string of 66 bytes (132 hex chars). The **viewing key comes first**.

> **⚠ Deviation from EIP-5564.** EIP-5564 serialises the meta-address as `st:eth:0x<spendingPubKey><viewingPubKey>` — **spending key first**. CSAP places the **viewing key first** (`V‖S`). The two encodings are byte-reversed at the 33-byte boundary. Implementers interoperating with EIP-5564 tooling MUST convert. See §4 (Backwards Compatibility); whether to keep this deviation or align with EIP-5564 in a future scheme id is an open decision recorded there.

### 2.2 Key derivation

Keys are derived deterministically from a single wallet signature. There is no key server.

1. The wallet signs the **canonical derivation message** (a fixed UTF-8 string; see below). Let `sig` be the raw signature bytes (65 bytes for an Ethereum `personal_sign`/secp256k1 signature; 64 bytes for a Solana ed25519 `signMessage`).
2. Expand with HKDF:
   ```
   okm = HKDF(hash = SHA-256, ikm = sig, salt = ∅, info = DOMAIN, L = 64)
   v = okm[0:32]      // viewing private key
   s = okm[32:64]     // spending private key
   ```
3. `V = v·G`, `S = s·G`, both compressed; `metaAddress = V ‖ S` (§2.1).

`v` and `s` MUST be valid secp256k1 scalars in `[1, n-1]`. The probability that an HKDF output is `0` or `≥ n` is ≈ 2⁻¹²⁸ and is treated as a derivation failure (see §6).

**Canonical derivation message.** This version fixes the message to the chain-neutral string:

```
Sign this message to derive your Opaque Cash stealth keys. This does not approve any transaction.
```

All clients MUST sign exactly this string (no chain suffix, no trailing data) so that a given wallet always derives the same keys regardless of which Opaque frontend it uses.

> **Implementation note (known divergence to fix).** Some current Solana entry points sign a different string (`"… stealth keys on Solana. This is not a transaction and does not move funds."`). That produces a *different* key set for the same wallet depending on the onboarding path. Clients MUST converge on the canonical message above; a migration that scans both the canonical and the legacy-string-derived key sets is REQUIRED to avoid orphaning existing Solana users.

**Cross-chain identity.** Because the signature itself is the entropy, an Ethereum wallet and a Solana wallet sign *different bytes* over the same message and therefore derive *different* key sets. A single cross-chain identity is achieved by deriving once (from one wallet) and **registering that one meta-address on both chains** (§2.7), not by re-deriving per chain.

### 2.3 Stealth address derivation (sender)

Given a recipient meta-address `(V, S)`:

1. Generate a fresh ephemeral key pair `r` (random, per payment), `R = r·G` (compressed).
2. Shared secret (ECDH, raw point — no extra hashing inside ECDH): `s_point = r·V`; let `sec = compressed(s_point)` (33 bytes).
3. `s_h = Keccak256(sec)` (32 bytes).
4. **View tag** `t = s_h[0]` (the most significant / first byte).
5. Stealth public key: `P_stealth = S + (s_h mod n)·G`.
6. **Stealth identifier (scanner-matching address):**
   ```
   stealthAddress = Keccak256( uncompressed(P_stealth)[1:65] )[12:32]   // 20 bytes, EVM-style
   ```
   i.e. drop the `0x04` prefix byte, Keccak-256 the 64-byte `x‖y`, take the last 20 bytes.
7. Publish an announcement (§2.5) carrying `R`, `metadata` with `metadata[0] = t`, and `stealthAddress`, under scheme id `1`.

**Solana fund custody.** On Solana the 20-byte `stealthAddress` from step 6 is used **only as the scanner-matching identifier** (so one scanner matches both chains). The account that actually holds funds is a deterministic ed25519 keypair both parties can recompute from the stealth point:
```
solanaSeed   = SHA-256( "opaque-solana-stealth-v1" ‖ uncompressed(P_stealth) )[0:32]
solanaKeypair = ed25519_from_seed(solanaSeed)
```

The one-time **spending key** for `P_stealth` is `p_stealth = (s + (s_h mod n)) mod n` (recipient side; see §2.8 step 5).

### 2.4 View tag computation

The view tag is the single byte `s_h[0]` = the most significant byte of `Keccak256(sharedSecret)`. It is carried as the first byte of the announcement `metadata`. During scanning, a recipient computes its own `s_h[0]` and rejects any announcement whose `metadata[0]` differs without performing the elliptic-curve point addition of step 5/§2.8. A non-owned announcement passes the filter with probability `1/256`, so ≈ 99.6% of foreign announcements are rejected before any EC work.

### 2.5 Announcement format

A single announcement carries: scheme id, the ephemeral public key `R` (33-byte compressed), `metadata` (≥ 1 byte; `metadata[0]` = view tag; remaining bytes reserved for sender-defined data such as an encrypted memo), and the stealth identifier.

**Ethereum ([EIP-5564], exact match).** `StealthAddressAnnouncer` emits:
```solidity
event Announcement(
    uint256 indexed schemeId,
    address indexed stealthAddress,
    address indexed caller,
    bytes ephemeralPubKey,
    bytes metadata
);
// function announce(uint256 schemeId, address stealthAddress, bytes ephemeralPubKey, bytes metadata)
```

**Solana.** `stealth_announcer` exposes:
```rust
// announce(scheme_id: u64, stealth_address: Vec<u8>, ephemeral_pub_key: Vec<u8> /*33*/, metadata: Vec<u8>)
#[event] pub struct Announcement {
    pub scheme_id: u64,
    pub stealth_address: Vec<u8>,   // 20-byte EVM-style identifier (scanner-matching)
    pub caller: Pubkey,
    pub ephemeral_pub_key: Vec<u8>, // 33-byte compressed secp256k1
    pub metadata: Vec<u8>,          // metadata[0] = view tag
}
```
`ephemeral_pub_key` MUST be 33 bytes and `metadata` MUST be non-empty (both enforced on-chain). An optional `announce_with_log(... , log_id: [u8;32])` additionally writes a PDA at seeds `["announcement", caller, log_id]` so indexers can use `getProgramAccounts` in addition to log parsing.

### 2.6 Cross-chain payload format (96 bytes) — UAB transport

For cross-chain relay (specified fully in `UAB.md`, implemented in Phase 1), an announcement is re-encoded into a fixed 96-byte payload so a view-tag filter works without parsing chain-specific event ABIs:

```
offset  size  field
0       1     view_tag                 (= metadata[0])
1       33    ephemeral_pubkey         (compressed secp256k1)
34      32    stealth_address          (Solana: 32-byte pubkey; Ethereum: 20-byte addr, left-padded with 12 zero bytes)
66      2     source_chain_id          (protocol id; Ethereum = 0x0001, Solana = 0x534F) 
68      4     scheme_id                (= 1)
72      24    metadata                 (reserved; metadata[0] = 0x00 standard, 0xB2 = PSR attestation marker)
```

This `source_chain_id` is a CSAP protocol identifier and is distinct from Wormhole's transport chain IDs (defined in `UAB.md`). The on-chain native formats of §2.5 remain canonical for single-chain operation; the 96-byte payload is the transport encoding only. Exact field values are finalised with Phase 1.

### 2.7 Registry interface

Registration binds a public account to a meta-address so senders can resolve a recipient by address rather than by sharing 66 raw bytes.

**Ethereum ([ERC-6538], exact match).** `StealthMetaAddressRegistry`:
- `mapping(address registrant => mapping(uint256 schemeId => bytes)) stealthMetaAddressOf`
- `registerKeys(uint256 schemeId, bytes stealthMetaAddress)` and alias `register(...)`
- `registerKeysOnBehalf(address registrant, uint256 schemeId, bytes signature, bytes stealthMetaAddress)` — verifies an EIP-712 signature (type hash `Erc6538RegistryEntry(uint256 schemeId,bytes stealthMetaAddress,uint256 nonce)`, domain name `"ERC6538Registry"`, version `"1.0"`) with an EIP-1271 fallback for contract accounts; consumes `nonceOf[registrant]`.
- `incrementNonce()` to invalidate outstanding signatures.
- Events: `StealthMetaAddressSet(address indexed registrant, uint256 indexed schemeId, bytes stealthMetaAddress)`, `NonceIncremented(address indexed registrant, uint256 newNonce)`.

**Solana.** `stealth_registry`:
- `register_keys(scheme_id: u64, stealth_meta_address: Vec<u8> /*MUST be 66*/)` → PDA at seeds `["stealth_meta", registrant, scheme_id.to_le_bytes()]`.
- `register_keys_on_behalf(...)` with an ed25519-authorised flow and a nonce PDA at seeds `["nonce", registrant]`.
- `increment_nonce()`, `resolve()`; clients MAY also read the PDA directly.
- Event `StealthMetaAddressSet { registrant, scheme_id, stealth_meta_address }`.

The stored `stealthMetaAddress` bytes use the §2.1 ordering (`V‖S`).

### 2.8 Scanner algorithm

A scanner holds the viewing private key `v` and the spending public key `S` (the spending *private* key is needed only to spend). For each announcement `(R, metadata, stealthAddress)`:

1. `sec = compressed(v·R)`.
2. `s_h = Keccak256(sec)`; `t' = s_h[0]`.
3. **View-tag filter:** if `t' ≠ metadata[0]`, skip (no EC addition).
4. Else compute `P_stealth = S + (s_h mod n)·G` and `addr' = Keccak256(uncompressed(P_stealth)[1:65])[12:32]`.
5. If `addr' == stealthAddress`, the payment is owned. To spend, reconstruct the one-time key `p_stealth = (s + (s_h mod n)) mod n` (requires the spending private key `s`).

A scanner that has only `v` (not `s`) can *detect* all incoming payments but cannot spend — enabling delegated/watch-only scanning. This is the basis for the viewing-key disclosure features in `PSR.md` / Phase 5.

### 2.9 Name-service meta-address records (ONS seed)

A meta-address MAY additionally be published under an existing name service so senders can resolve a human-readable name instead of a registry lookup. The record key is the reverse-DNS string **`com.opaque.meta`** (ENSIP-5 style): on ENS it is a text record (`text(node, "com.opaque.meta")`), on SNS the equivalent record field of the `.sol` domain. The record value is the §2.1 serialisation — the `0x`-prefixed 132-hex-char `V‖S` meta-address — optionally prefixed with `st:opq:` for self-description; resolvers MUST accept both forms and MUST validate that both 33-byte halves are valid compressed secp256k1 points before use. The on-chain registry (§2.7) remains authoritative where both exist: a resolver finding a conflict SHOULD prefer the registry entry for the address the name resolves to. This record is the read-path seed for the Opaque Name Service (`alice.opq.eth`, specified separately); until ONS ships, writers set the record manually through their name-service tooling.

## Rationale

- **secp256k1 + Keccak-256.** Chosen so a derived stealth identifier is a valid Ethereum address with no extra mapping, and so the identical math runs on Solana (via the `k256` crate / `@noble/curves`). This is what makes one scanner cover both chains.
- **Raw-point ECDH (not RFC-5903 with an extra KDF).** Matches EIP-5564 exactly so announcements interoperate with EIP-5564 scanners.
- **View tags.** An 8-bit tag removes ~99.6% of EC work during scanning at the cost of leaking 8 bits of a per-payment value that is meaningless without the viewing key.
- **HKDF-from-signature.** Deterministic, serverless key recovery from any wallet; the viewing/spending split allows watch-only delegation.
- **20-byte identifier on Solana.** Using the EVM-style address as the *matching* key (while custody uses a derived ed25519 account) lets the universal scanner compare a single value type across chains.

## Backwards Compatibility

| Aspect | EIP-5564 / ERC-6538 | CSAP | Compatible? |
|---|---|---|---|
| `Announcement` event signature | as in EIP-5564 | identical | ✅ Yes |
| View tag (MSB of hashed secret, first metadata byte) | yes | identical | ✅ Yes |
| Registry storage + `registerKeys`/`registerKeysOnBehalf` + EIP-712/EIP-1271 | ERC-6538 | identical | ✅ Yes |
| Meta-address byte order | `spending ‖ viewing` | `viewing ‖ spending` | ⚠ **Reversed — convert** |
| `schemeId` integer type | `uint256` | `uint256` (EVM) / `u64` (Solana) | ✅ value-compatible |

Consequences of the meta-address deviation: a standard EIP-5564 client that parses an Opaque meta-address as `spending‖viewing` will swap the two keys, derive wrong shared secrets, and fail to match. Interop requires reversing the two 33-byte halves. **Open decision:** keep the deviation (document it as the CSAP scheme-1 ordering) or introduce an EIP-5564-aligned ordering under a new scheme id. Recorded in the project execution plan.

## Test Vectors

Vectors use the deterministic inputs already present in the reference Rust scanner tests (`opaquecash/ethereum/scanner/src/scanner.rs`):

```
viewing_private_key   = 0xaaaa…aa   (32 bytes, all 0xAA)
spending_private_key  = 0xbbbb…bb   (32 bytes, all 0xBB)
ephemeral_private_key = 0xcccc…cc   (32 bytes, all 0xCC)
```

Canonical vector 1 (all values verified to agree across three independent implementations — see below):

```json
{
  "description": "scheme-1 secp256k1 DKSAP round trip (matches Rust scanner test inputs)",
  "scheme_id": 1,
  "viewing_private_key":  "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "spending_private_key": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "ephemeral_private_key":"0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
  "viewing_public_key":   "0x026a04ab98d9e4774ad806e302dddeb63bea16b5cb5f223ee77478e861bb583eb3",
  "spending_public_key":  "0x0268680737c76dabb801cb2204f57dbe4e4579e4f710cd67dc1b4227592c81e9b5",
  "meta_address":         "0x026a04ab98d9e4774ad806e302dddeb63bea16b5cb5f223ee77478e861bb583eb30268680737c76dabb801cb2204f57dbe4e4579e4f710cd67dc1b4227592c81e9b5",
  "ephemeral_public_key": "0x02b95c249d84f417e3e395a127425428b540671cc15881eb828c17b722a53fc599",
  "shared_secret":        "0x03bb26c34c778b763da72856a7b640f7402aac1b6bb41e5e880abd7d8f72f97067",
  "s_h":                  "0xe1641025b6abb6decc4d599b0f215df6c011542d8c0ffabd2187bb8d3620a3b4",
  "view_tag":             225,
  "stealth_address":      "0xa5847a467208cbcd5d238369865a90716310183a",
  "one_time_private_key": "0x9d1fcbe17267729a88091556cadd19b3c11e33029883163d1d7118bc21a61e2e"
}
```

The full set — one HKDF derivation vector (§2.2) plus three DKSAP round-trip vectors — lives in [`opaquecash/circuits/test/test_vectors.json`](https://github.com/opaquecash/circuits), generated by `circuits/test/generate_vectors.py`.

**Cross-validation (Task 0.5, done).** Every output above was produced by three independent implementations and confirmed byte-for-byte equal:

1. **Pure-Python** (`generate_vectors.py`) — its own secp256k1 / Keccak-256 / HKDF-SHA256, depending on neither of the below (replaces the `py_ecc` reference);
2. **Rust scanner** (`k256`) — pinned in `scanner.rs::matches_csap_test_vectors` (`cargo test`, green);
3. **TypeScript** (`@noble/curves`, `@noble/hashes`) — the exact primitives the `@opaquecash/opaque` SDK and both frontends use.

A vector with `s_h ≥ n` is intentionally avoided: the Rust scanner rejects rather than reduces (see §6, Security Considerations), so all three agree only when `s_h < n`.

## Security Considerations

- **Anonymity set.** CSAP hides *which* address a payment landed in, not the existence of payments. Privacy is proportional to the number of concurrent users; a small set is weak against timing analysis. See the execution plan §17 (anonymity-set growth).
- **No amount privacy.** Transfer amounts remain visible on-chain. Amount privacy is a separate layer (Privacy Pool / Phase 3).
- **Fee/gas linkage.** Sweeping or announcing from a funded public wallet links that wallet to stealth activity until a relayer market exists (Phase 4). Implementations SHOULD warn users.
- **Canonical message discipline.** Keys are only recoverable if the wallet signs exactly the canonical message (§2.2). The current Solana message inconsistency MUST be fixed with a scan-both-strings migration, or users lose access to funds at the legacy-derived key set.
- **Scalar edge cases.** `(s_h mod n)` and the derived private keys must be valid non-zero scalars `< n`. The TypeScript path reduces `s_h mod n`; the Rust scanner currently *rejects* `s_h ≥ n` rather than reducing. These agree except on a ≈ 2⁻¹²⁸ set; the canonical behaviour is **reduce mod n**, and the scanner SHOULD be reconciled to match. Point-at-infinity and zero-scalar results MUST be rejected.
- **Ephemeral key freshness.** `r` MUST be unique and unpredictable per payment. Reusing `r` across two recipients links them. The deterministic "announcer" ephemeral key (`HKDF(metaAddress‖"opaque-announcer-v1", info="opaque-announcer-ephemeral")`) is a special single-recipient ghost-receive construction and MUST NOT be used for normal sends.
- **Viewing-key delegation.** Sharing `v` discloses *all* incoming payments to the holder. Selective/threshold disclosure is addressed in Phase 5.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

---

### Reference deployments (for reviewers)

| Chain | Contract / Program | Address / Program ID |
|---|---|---|
| Ethereum Sepolia | `StealthMetaAddressRegistry` | `0x77425e04163d608B876c7f50E34A378624A12067` |
| Ethereum Sepolia | `StealthAddressAnnouncer` | `0x840f72249A8bF6F10b0eB64412E315efBD730865` |
| Solana Devnet | `stealth_registry` | `E9LBRG5eP2kvuNfveouqQ9tA5P6nrpyLyWFjH9MFYVno` |
| Solana Devnet | `stealth_announcer` | `HGFn2fH7bVQ5cSuiG52NjzN9m11YrB3FZUfoN9b9A5jf` |

[EIP-5564]: https://eips.ethereum.org/EIPS/eip-5564
[ERC-6538]: https://eips.ethereum.org/EIPS/eip-6538
