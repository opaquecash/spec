# Opaque Protocol — Specifications

Canonical specifications for the Opaque privacy protocol. These documents describe the protocol as deployed on Ethereum (Sepolia) and Solana (Devnet); the reference implementations live in `opaquecash/ethereum` and `opaquecash/solana`.

| Spec | Scope | Status |
|---|---|---|
| [CSAP.md](./CSAP.md) | Cross-chain Stealth Address Protocol — the dual-key stealth payment layer (DKSAP) | Draft v1 |
| [PSR.md](./PSR.md) | Programmable Stealth Reputation — schema-bound attestations + ZK reputation proofs | Draft (canonical V2) |
| [UAB.md](./UAB.md) | Universal Announcement Bus — cross-chain announcement transport (Wormhole) | Draft |
| [payload-format.md](./payload-format.md) | The 96-byte cross-chain payload encoding | Draft v1 |
| [nullifier-registry.md](./nullifier-registry.md) | Cross-chain nullifier format and consume-once registry semantics | Draft v1 |
| [ONS.md](./ONS.md) | Opaque Name Service — cross-chain naming (`*.opq.eth`), canonical-chain-wins | Draft v1 |
| [relayer-market.md](./relayer-market.md) | Relayer market — gas-private submission (stake, blind jobs, submit-or-slash) | Draft v1 |
| [changelog.md](./changelog.md) | Version history from CSAP v1 forward | Living |

## What Opaque is

An open, serverless, cross-chain privacy protocol. Stealth addresses hide who paid whom; Programmable Stealth Reputation lets a stealth identity carry verifiable, privacy-preserving credentials. Every primitive is specified before (or alongside) its code so any wallet or dApp can implement compatibility. The Opaque frontends are reference implementations, not the only ones.

## Reference deployments

| Chain | Contract / Program | Address / Program ID |
|---|---|---|
| Ethereum Sepolia | StealthMetaAddressRegistry (ERC-6538) | `0x77425e04163d608B876c7f50E34A378624A12067` |
| Ethereum Sepolia | StealthAddressAnnouncer (ERC-5564) | `0x840f72249A8bF6F10b0eB64412E315efBD730865` |
| Solana Devnet | stealth_registry | `E9LBRG5eP2kvuNfveouqQ9tA5P6nrpyLyWFjH9MFYVno` |
| Solana Devnet | stealth_announcer | `HGFn2fH7bVQ5cSuiG52NjzN9m11YrB3FZUfoN9b9A5jf` |
| Ethereum Sepolia | UABSender / UABReceiver | `0x872787c0BD1A0C71e6D1be5a144EB044e0CB2069` / `0x9eF189f7a263F870Cf80f9A89d1349A6AF7b15cF` |
| Solana Devnet | uab_receiver | `7d4Sbmmpy954JwSNdjwf31pgbeWUQqwpgNdte5iy3vuM` |

## License

CC0-1.0.
