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
| [privacy-pool.md](./privacy-pool.md) | Privacy pool — amount privacy + association-set compliance (Privacy Pools model) | Draft v1 |
| [conditional-disclosure.md](./conditional-disclosure.md) | Conditional disclosure — threshold viewing keys (FROST quorum + pool-scoped ZK disclosure) | Draft v1 |
| [changelog.md](./changelog.md) | Version history from CSAP v1 forward | Living |

## What Opaque is

An open, serverless, cross-chain privacy protocol. Stealth addresses hide who paid whom; Programmable Stealth Reputation lets a stealth identity carry verifiable, privacy-preserving credentials. Every primitive is specified before (or alongside) its code so any wallet or dApp can implement compatibility. The Opaque frontends are reference implementations, not the only ones.

## Reference deployments

Every spec'd component is live on Ethereum Sepolia + Solana devnet (testnet only).
Addresses and program ids are maintained in one place — the generated
`@opaquecash/deployments` package ([`opaquecash/sdk`](https://github.com/opaquecash/sdk))
and the [deployments page](https://docs.opaque.cash/protocol/deployments) — and in the
contract/program tables of the [`ethereum`](https://github.com/opaquecash/ethereum) and
[`solana`](https://github.com/opaquecash/solana) READMEs. This file deliberately does
not duplicate them.

## License

CC0-1.0.
