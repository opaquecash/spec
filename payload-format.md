# Cross-Chain Announcement Payload (96 bytes)

**Status:** Draft · **Version:** 1 · **License:** CC0-1.0

> The fixed 96-byte body carried inside a Universal Announcement Bus (UAB) message. It is
> the chain-neutral, self-describing form of a CSAP stealth *payment* announcement, sized so
> a scanner can run the EIP-5564 view-tag pre-filter without parsing chain-specific event
> encodings. See [UAB.md] for how the body is transported (Wormhole) and re-emitted.

## Layout

All multi-byte integers are **big-endian**. The body is exactly **96 bytes**.

| Offset | Size | Field | Description |
|---:|---:|---|---|
| 0  | 1  | `view_tag`         | EIP-5564 view tag = MSB of `Keccak-256(shared_secret)`. The cheap pre-filter. |
| 1  | 33 | `ephemeral_pubkey` | Compressed secp256k1 ephemeral public key. |
| 34 | 32 | `stealth_address`  | Recipient one-time address. EVM 20-byte addresses are **left-padded** to 32 bytes; Solana pubkeys are 32 bytes as-is. |
| 66 | 2  | `source_chain_id`  | **Wormhole** chain id of the origin chain (Ethereum = 2, Solana = 1). |
| 68 | 4  | `scheme_id`        | CSAP / ERC-5564 stealth scheme. `1` = secp256k1 with view tags. |
| 72 | 24 | `metadata`         | Sender-defined bytes (e.g. an encrypted payment id). The view tag is **not** repeated here. |

`1 + 33 + 32 + 2 + 4 + 24 = 96`.

## Reconstructing a native announcement

A UAB receiver reconstructs the local EIP-5564 / CSAP announcement so existing scanners need
no change:

- `schemeId`       = `scheme_id`
- `stealthAddress` = `stealth_address` (EVM consumers take the low 20 bytes)
- `ephemeralPubKey` = `ephemeral_pubkey`
- `metadata`       = `view_tag ‖ metadata` (25 bytes, **view tag first** — the EIP-5564 convention)

## Scope and versioning

This v1 format carries CSAP **payment** announcements. The 24-byte `metadata` budget is
deliberately small so the body stays fixed-size for the scanner fast path; it does **not**
carry large PSR V2 attestation metadata (130 bytes: schema id / issuer / uid / nonce), which
remains chain-local in v1. A future length-delimited body is reserved for cross-chain
attestation relay; until then UAB messages are exactly 96 bytes and consumers **MUST** reject
any other length.

## Chain id note

`source_chain_id` is the **Wormhole** chain id of the emitting chain — not the EVM
`block.chainid`, and not an ASCII tag. (An earlier draft wrote `0x534F` for Solana; the
correct value is `1`.) The Wormhole VAA envelope independently carries `emitterChainId`;
receivers **MUST** treat the VAA's `emitterChainId` as authoritative and **MAY** cross-check
it against `source_chain_id`.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

[UAB.md]: ./UAB.md
