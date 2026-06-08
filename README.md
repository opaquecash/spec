# Opaque Protocol — Specifications

Canonical specifications for the Opaque privacy protocol. These documents describe the protocol as deployed on Ethereum (Sepolia) and Solana (Devnet); the reference implementations live in `opaquecash/ethereum` and `opaquecash/solana`.

| Spec | Scope | Status |
|---|---|---|
| [CSAP.md](./CSAP.md) | Cross-chain Stealth Address Protocol — the dual-key stealth payment layer (DKSAP) | Draft v1 |
| [PSR.md](./PSR.md) | Programmable Stealth Reputation — schema-bound attestations + ZK reputation proofs | Draft (canonical V2) |
| UAB.md | Universal Announcement Bus — cross-chain announcement transport (Wormhole) | Planned |
| payload-format.md | The 96-byte cross-chain payload encoding | Planned (see CSAP §2.6) |
| nullifier-registry.md | Cross-chain nullifier format | Planned |
| changelog.md | Version history | Planned |

## What Opaque is

An open, serverless, cross-chain privacy protocol. Stealth addresses hide who paid whom; Programmable Stealth Reputation lets a stealth identity carry verifiable, privacy-preserving credentials. Every primitive is specified before (or alongside) its code so any wallet or dApp can implement compatibility. The Opaque frontends are reference implementations, not the only ones.

## Reference deployments

| Chain | Contract / Program | Address / Program ID |
|---|---|---|
| Ethereum Sepolia | StealthMetaAddressRegistry (ERC-6538) | `0x77425e04163d608B876c7f50E34A378624A12067` |
| Ethereum Sepolia | StealthAddressAnnouncer (ERC-5564) | `0x840f72249A8bF6F10b0eB64412E315efBD730865` |
| Solana Devnet | stealth_registry | `E9LBRG5eP2kvuNfveouqQ9tA5P6nrpyLyWFjH9MFYVno` |
| Solana Devnet | stealth_announcer | `HGFn2fH7bVQ5cSuiG52NjzN9m11YrB3FZUfoN9b9A5jf` |

## License

CC0-1.0.
