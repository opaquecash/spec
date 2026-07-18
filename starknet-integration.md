# Starknet Integration — PSR × STRK20 Design

**Status:** Draft (design document, pre-implementation) · **Date:** 2026-07-18 · **License:** CC0-1.0
**Depends on:** [CSAP.md], [PSR.md], [UAB.md], [privacy-pool.md], [conditional-disclosure.md], [relayer-market.md], [nullifier-registry.md]

> This document records the integration design for extending Opaque to Starknet as a third chain, with STRK20 (Starknet's shielded ERC-20 pool standard) as the primary integration target. It is a design/positioning document, not yet a normative protocol spec; normative changes it implies (CSAP scheme constants, chain ids, account-class pinning) land in the referenced specs when implementation starts. Facts below were researched and adversarially cross-verified against primary sources on 2026-07-18; claims that rest on a single secondary source are marked **[unverified]**.

---

## 1. Abstract

STRK20 gives Starknet a single canonical shielded pool for every ERC-20, with mandatory auditor viewing-key escrow for after-the-fact disclosure. It has **no eligibility layer**: no association sets, no inclusion/exclusion proofs, no credential gating at entry or in-pool. PSR is exactly that missing layer — a schema-bound, ZK-proven, nullifier-scoped answer to "is this actor eligible?" that never identifies the actor. The integration ports only PSR's thin on-chain footprint to Cairo (a Garaga-generated Groth16-BN254 verifier plus a root/nullifier wrapper), consumes STRK20 through three permission tiers (permissionless in-pool gates first, StarkWare's screening seam second, an in-circuit eligibility SNIP third), and adds Starknet as a third CSAP chain via a counterfactual secp256k1 account class. The privacy pool, the Shamir viewing-key backstop, and the token-sweep forwarder are explicitly **not** ported: STRK20 and Starknet account abstraction subsume them.

## 2. Background: STRK20 as verified

- **One pool, all tokens.** STRK20 is a single canonical shielded-pool Cairo contract for all ERC-20s, not a token-side interface. Tokens adopt it permissionlessly by being deposited; no wrappers or token changes. Reference: `starkware-libs/starknet-privacy`, `packages/privacy/src/interface.cairo`.
- **Note-based UTXO model.** Encrypted notes `(owner, token, u128 amount)` in write-once cells keyed by channel-derived hashes; Poseidon nullifiers incorporating the owner's private key; all-or-nothing spends with change notes; "open notes" (plaintext amount, filled at execution) for DeFi composition. The nullifier set grows linearly; sub-linear state growth is explicitly unsolved upstream.
- **Proving = SNIP-36** (Starknet v0.14.2 "Shinobi", mainnet 2026-04-21). The client executes against a virtual snapshot of a recent block and proves with Stwo (Circle STARK, ~29 s); the proof rides new optional `proof` / `proof_facts` fields on Invoke V3 and is verified by **Starknet consensus, not the Starknet OS**. StarkWare's own framing: degraded security versus native apps, privacy "de facto" rather than formally zero-knowledge, formal ZK deferred to later phases. A ~500 KB proof costs ≈ 75M L2 gas to propagate.
- **Compliance = two legs.** (1) Mandatory per-user registration of a viewing public key plus the viewing private key encrypted to a threshold-controlled third-party audit firm — whole-history, after-the-fact disclosure on lawful request. (2) An entry-screening seam: `IServer::apply_actions(actions: Span<ServerAction>, screening: Option<ScreeningAttestation>)` with an **admin-set, singular** screener public key. Screening's operator and mechanism are unspecified in all public sources; "screening at entry" is press paraphrase of Eli Ben-Sasson, not documentation.
- **No eligibility layer.** Confirmed negative: no association sets, no ASP role, no inclusion/exclusion proofs, no credential hooks on shield/transfer/unshield. KYC is expected "at the ramps," off-protocol.
- **Composition pattern.** Anonymizer helper contracts (Ekubo swap, Vesu lending, escrow) are invoked *from the pool* during a proven spend via the external-invocation action (max once per transaction bundle).
- **Ecosystem state (2026-07-18).** strkBTC live since 2026-05-12 (Atomiq bridge, Ready/Xverse wallets, 3 BTC cap at launch); all-ERC-20 rollout reported 2026-06-09 **[unverified]**; no published mainnet pool address found; SDK `@starkware-libs/starknet-privacy-sdk` v0.14.3-rc.3 exists but is GitHub-Packages-only; the recommended dapp path is the Privacy Wallet API (`WalletAccountV6` / `useStrk20` hooks).

## 3. Thesis

Viewing keys answer *"who did this?"* after the fact. PSR answers *"is this actor eligible?"* before the fact, without identifying them. STRK20 has the first and structurally lacks the second; the `Option<ScreeningAttestation>` parameter shows the seam was architected but never filled.

Positioning: Opaque should occupy that seam as the **neutral, multi-chain, multi-issuer credential layer** before a first-party alternative calcifies. Timing signals (both to watch, neither verified beyond a single source): StarkWare demoed a first-party "Private KYC" flow (2026-06-24, passport NFC → attribute registry → ZK queries) **[unverified]**, and SilentSwap is publicly pitching a compliance layer at this exact seam — the same forum thread that articulates the attestation gap is also, therefore, competition.

## 4. Integration architecture

### 4.1 On-chain (Cairo) — deploy only three things

**(a) Groth16-BN254 verifier, Garaga-generated.** Starknet has no pairing builtin; Garaga verifies BN254 Groth16 in Cairo via the `add_mod`/`mul_mod` ModBuiltin and is production-proven (World ID). `garaga gen` consumes the existing snarkjs `verification_key.json` (5 IC points, 4 public signals) — the same vkey pinned in `Groth16VerifierV2.sol` and the Solana `VK_*_V2` constants. Cost is order-of-magnitude ~34M Sierra gas per verification (Garaga's integration-test vkey; own-vkey gas **and calldata size** are a P0 exit criterion — Garaga calldata is thousands of felts and per-tx limits must be checked).

> **Ceremony coupling.** The Garaga verifier is *code-generated from the vkey*. The planned mainnet trusted-setup ceremony (revocation-aware leaf per OPQ-018, in-circuit issuer binding per OPQ-006) therefore forces a **full verifier redeploy on Starknet**, in lockstep with the EVM/Solana vkey swaps — not a parameter update. Launch Starknet on the current testnet vkey with an explicit redeploy plan, and add the Starknet verifier as the fourth implementation in the committed-fixture CI cross-check (snarkjs + EVM + Solana + Cairo).

**(b) Reputation-verifier wrapper**, mirroring `OpaqueReputationVerifierV2`:
- Admin-published Merkle roots, `ROOT_EXPIRY = 3600 s`, `MAX_ROOT_HISTORY = 100`. Adopt the **EVM delete-on-evict semantics**, not Solana's per-entry TTL (the OPQ-041 divergence must not propagate to a third chain).
- Nullifier set and all public signals stored as **u256 / two-felt limbs**: BN254's scalar field r ≈ 2^253.6 exceeds felt252 (max ≈ 2^251), so no signal, proof coordinate, or nullifier fits one felt. The reject-if-≥ r field check ([nullifier-registry.md], [PSR.md] field-encoding rule) is mandatory; a missed check or a double encoding (x vs x + r) is a soundness bug.
- **Schema-liveness binding (OPQ-006) from day one**: require the proof's `attestation_id` to reference a live registered schema before pairing, as the EVM wrapper does. The Solana deployment lacks this; the Starknet port must not inherit the asymmetry.
- **Chain-scoped nullifiers.** `external_nullifier` construction MUST embed a project-assigned Starknet chain-domain constant, and — because the verifier cannot introspect scope structure — **every consumer contract MUST construct and check its own domain**. Without the consumer-side check the mandate is vacuous and one (stealth_pk, scope) proof is consumable once per chain.

**(c) PSR-gate consumer contracts** — see §5.

**Deferred: Cairo schema registry.** Registries are off the proof path (they feed the off-chain indexer/tree builder), so ship verifier-first. When ported: keep `schema_id = SHA-256(authority ‖ name ‖ version)` for cross-chain parity, but derive off-chain and check equality on-chain — SHA-256 is an expensive syscall in Cairo, and schema ids are per-chain anyway because they embed the authority pubkey.

### 4.2 Off-chain — unchanged

- **Tree building and leaves.** Circomlib Poseidon over BN254, exactly as today. No deployed chain computes Poseidon on-chain — the indexer builds trees, the keeper publishes roots — so **no BN254 Poseidon in Cairo is needed**. (Garaga ships a circom-compatible Poseidon-BN254 since v0.15.4 if that ever changes.)
- **Prover.** `@opaquecash/psr-prover` (snarkjs fullProve, OPQ-030 artifact pinning, OPQ-038 random nonce) is reused byte-for-byte. Starknet needs only a third **proof encoder** — Garaga calldata with u256 limbs — as a new package sibling to the EVM uint256-array and Solana 64/128/64-BE encoders.
- **Keeper.** The relayer TS keeper gains a Starknet root-publication duty. With the 1 h TTL and the OPQ-018 rebuild-excluding-revoked obligation, keeper liveness becomes triple-critical.

## 5. How STRK20 consumes PSR proofs — three permission tiers

**Tier 1 — permissionless, ship first: PSR-gated in-pool external actions.** Follow STRK20's own composition pattern: deploy PSR-gated anonymizer-style contracts that the pool invokes during a proven spend — gated swap, gated lending ("only KYB-attested treasuries borrow privately"), gated escrow. The gate verifies the PSR proof, checks root freshness and the Starknet nullifier domain, consumes the nullifier, then executes the DeFi leg. No StarkWare permission required.

> **Honest limit.** Anonymizers gate *flows of funds already inside the pool*, not pool entry. Deposits are client-proven action bundles bound to the depositor's registered channel; whether a third-party contract can be depositor-of-record at all is a **blocking unknown** (§8), and any external entry gate is bypassable by direct deposit. Tier 1 delivers credentialed *flows*, not credentialed *entry*.

**Tier 2 — negotiated: PSR behind the screening seam.** Operate a screener service that signs `ScreeningAttestation`s only against a valid PSR proof, making PSR the eligibility engine behind StarkWare's own parameter. Requires the pool admin to accept the screener key; the interface exposes a **single** screener key (`get_screener_public_key`), so multi-screener support is a protocol change on StarkWare's side, not an admin configuration.

**Tier 3 — standard-track: eligibility inside the spend proof.** [privacy-pool.md] is explicit that eligibility enforced on a *different proof* than the spend can be split across actors; the sound end-state is an eligibility-root membership sub-statement inside STRK20's client circuit. That is a SNIP-scale proposal restated in STRK20's Stwo/Cairo proof system, not Groth16. Tiers 1–2 ship the weaker form meanwhile, with this gap documented. A related question worth pursuing with StarkWare: whether SNIP-36 will accept arbitrary third-party circuits — if yes, a future STARK-native PSR circuit sheds the Groth16 ceremony dependency entirely.

## 6. Non-goals — what Opaque does NOT port

1. **The privacy pool.** STRK20 already provides note commitments, nullifiers, shielded ERC-20 balances, change notes, and native client proving — ahead of Opaque's native-asset pool. PSR and the ASP model layer *on* STRK20; they do not compete with it.
2. **Shamir viewing-key escrow.** STRK20's mandatory auditor-encrypted viewing-key registration natively covers whole-key lawful escrow. The FROST threshold disclosure of [conditional-disclosure.md] remains **complementary** (quorum-based, per-case, (nullifier, context)-scoped versus single-auditor whole-history) but is P2 and requires a BIP-340-verification-in-Cairo cost spike first; if the cost is prohibitive the signature scheme changes and FROST tooling reuse breaks.
3. **StealthTokenSweep forwarder.** It exists because stealth EOAs hold tokens but no gas. On Starknet every stealth receiver is a contract account: SNIP-9 outside execution plus paymasters replace the *forwarder contract*. The deploy-account-before-sweep step and its fee-payer-linkage problem remain open (§8) — the forwarder is subsumed, the linkage problem is not.
4. **UAB-over-Wormhole.** Wormhole has no Starknet support on any product and no announced roadmap (verified). Transport changes per §7.4.

## 7. Third-chain CSAP and SDK plan

### 7.1 Custody: counterfactual secp256k1 account class

Keep the 98-byte meta-address of [CSAP.md] §2.1 **unchanged**. Pin one audited secp256k1-validating account class (OZ EthAccount-style; secp256k1 syscalls available since Cairo v2.1) whose signer is the existing one-time stealth key `P_stealth`. The Starknet stealth address is the counterfactual

```
starknet_stealth = compute_address(class_hash, salt, constructor_calldata)
```

derivable by the sender and any watch-only scanner from public announcement material (the Solana §2.3 precedent), and spendable only with `p_stealth = (s + (s_h mod n)) mod n` — the payer-sweep failure mode of the withdrawn OPQ-002 construction is avoided by construction, since spending requires the recipient's `s`.

Consequences, normative when implemented:
- `class_hash`, the constructor-calldata layout, and the salt rule are **consensus-critical CSAP constants** (a class upgrade changes all future stealth addresses); they migrate into [CSAP.md] with test vectors once Starknet custody is production (CSAP.md is scoped to already-deployed behaviour). The P1a reference class (`opaquecash/starknet`, `contracts/stealth_account`) fixes them as follows:
  - **Account class:** OpenZeppelin `EthAccountComponent` 2.0.0 (secp256k1 SRC-6 validation + deploy-account validator), non-upgradeable, no added storage. Declared on Starknet Sepolia at class hash `0x04794bab07198e0585d2d7951dbc5860fba47fea2a15d227ca3237b7b9e484ed`.
  - **Constructor calldata:** exactly the Serde serialisation of `P_stealth` as a `secp256k1::Secp256k1Point` (`[x.low, x.high, y.low, y.high]`), the single argument named `public_key` — matching OZ's `__validate_deploy__` so a real `deploy_account` validates against the same argument.
  - **Salt rule:** to be fixed alongside the SDK sender (§7.3); a per-payment salt derived from the ephemeral key `R` (candidate: low felt of `keccak(R)`) gives an announcement-derivable, unlinkable address. Pinned when the sender lands.
- The 20-byte EVM-style identifier remains the scanner-matching key; `starknet_stealth` is custody-only, mirroring the Solana split.
- **Key derivation:** default to deriving from the user's existing Ethereum or Solana wallet and registering the same meta-address on Starknet — CSAP's cross-chain identity model already works this way ([CSAP.md] §2.2), and it sidesteps Starknet wallet-signature determinism and account-key rotation. (HKDF being a prefix stream leaves `okm[96:128]` available if a Starknet-native slot is ever wanted.)

### 7.2 Registry and announcer (Cairo)

- **Announcer:** stateless event contract enforcing the Solana-style bounds (33-byte compressed ephemeral key, non-empty metadata, `metadata[0]` = view tag).
- **Registry:** `registrant → (scheme_id → meta_bytes)` with events; on-behalf registration authorized by **SNIP-12 typed data validated through the registrant account's SRC-6 `is_valid_signature`** (the EIP-1271 path becomes primary since every Starknet account is a contract), with a consumable nonce and a new domain string. Note: `is_valid_signature` requires a *deployed* registrant account; counterfactual accounts cannot register on-behalf until deployed.
- **Chain id:** allocate a project-assigned u16 (Wormhole's convention cannot cover Starknet) and register it in [payload-format.md] / [UAB.md].

### 7.3 SDK, scanner, app

A `StarknetAdapter` implements the existing `ChainAdapter` interface emitting the canonical announcement shape (20-byte id, 33-byte ephemeral, view tag), so the shared DKSAP scan filter runs unchanged; Rust twin in `scanner/` behind the existing adapter trait; indexing via `starknet_getEvents` (Apibara for streaming). The dominant cost is the **two-to-three-chain refactor**: the binary origin mapping in the SDK facade, the relayer-client job dispatch's non-Ethereum → Solana fallthrough, the `OpaqueScanChain` union and ~10 facade write branches, app-layer `ChainKey` unions and chain selectors, plus `generated/starknet.ts` in `@opaquecash/deployments`. This refactor is a P1 work item of its own (§9).

### 7.4 Registry/ONS mirroring and messaging (ranked)

1. **Native L1→L2 messaging** for Ethereum-origin data: `sendMessageToL2` from the Ethereum registry/ONS contracts; sequencer-automated, minutes-scale, no additional validator set. Ethereum acts as hub; Solana-origin data reaches Starknet transitively via the existing Solana→Ethereum Wormhole leg (trust is then Wormhole *plus* L1→L2 — unioned, not improved; the trust win is scoped to Ethereum-origin data such as the canonical ONS registry). Preserve the mirror invariants: emitter allowlist, monotonic sequence floor, revoke-as-tombstone (OPQ-004).
2. **Hyperlane** where a direct or low-latency leg is needed: live on Starknet mainnet (audited Cairo Mailbox), production Solana↔Starknet routes since 2025-10. The interchain gas paymaster is undeployed — plan to run our own relayer, which fits the existing keeper pattern.
3. LayerZero: live on Starknet since 2026-01; arbitrary-payload path from Cairo unconfirmed. Wormhole and Axelar: not viable.

Starknet→outbound: L2→L1 finality is ~3–4 h — acceptable for registry sync, unusable for interactive UAB. **Launch Starknet without outbound UAB announcements**; revisit with Hyperlane if demanded.

## 8. Risks and blocking unknowns

1. **Felt252 vs BN254 r** (§4.1b) — soundness-critical limb handling.
2. **Ceremony/vkey lockstep redeploy** across three chains, with the Cairo verifier regenerated, not reconfigured.
3. **OPQ-018 inherited:** revoked credentials prove valid until root rotation; the Starknet keeper must rebuild-excluding-revoked within the 1 h TTL.
4. **AA custody friction:** receiving to an undeployed counterfactual address works, but sweeping requires `DEPLOY_ACCOUNT` first — worse than an EVM stealth EOA — and needs a SNIP-9/paymaster flow that does not create fee-payer linkage.
5. **STRK20 governance gates Tiers 2–3:** screener/auditor keys are admin-set, the screener key is singular, screening operation is undocumented, no published mainnet pool address, SDK is rc on GitHub Packages. Tier 1 is the only zero-permission path; sequence accordingly.
6. **Proof-system mismatch is permanent** unless Tier 3 lands: PSR stays Groth16-BN254/circomlib-Poseidon; STRK20 is Stwo/Cairo under SNIP-36 Phase 1 (consensus-verified, "de facto" privacy). Composition happens at the contract layer.
7. **Cost:** ~34M Sierra gas per verification is workable but must be re-benchmarked with our vkey, our calldata, and current gas pricing in P0.

**Blocking unknowns to resolve first:** can a contract be depositor-of-record for a channel (shapes Tier 1's ceiling); STRK20 mainnet pool address and actual ERC-20 rollout state; screener-key governance and extensibility; SNIP-36 third-party circuit access; BIP-340 verification cost in Cairo.

**Prior art:** StealthPay (ERC-5564-on-STARK-curve hackathon project) prefigures the announcer/registry/view-tag design on Starknet; differentiation is CSAP's cross-chain identity plus the PSR layer. Cite it in any public write-up.

## 9. Phased roadmap

**P0 — proof of concept (3–5 weeks, ~1 engineer).** `garaga gen` verifier from the pinned vkey; minimal wrapper (u256 nullifier set, root registry + TTL, schema-liveness stub); Garaga-calldata proof encoder; keeper root publication to Starknet Sepolia; fourth-verifier CI cross-check. **Exit:** an existing testnet credential verifies on Starknet Sepolia; gas, calldata size, and tx-limit headroom measured.

**P1a — Starknet PSR + stealth primitives (~6–9 weeks, ~2 engineers).** Tier-1 gated contract plus one gated DeFi demo against the first reachable STRK20 deployment (tracked; blocked on pool address publication); Cairo announcer and SNIP-12 registry; stealth account class pinned and CSAP constants + test vectors written (also fix the stale §2.6 chain-id table).

**P1b — three-chain client surface (~6–9 weeks, ~2 engineers).** TS `StarknetAdapter`; deployments/facade/relayer-job/app refactor killing every binary chain dispatch; L1→L2 registry/ONS mirror. Rust scanner adapter and the Cairo schema registry move to P2.

**P2 — production (quarter-scale, externally gated).** Post-ceremony vkey redeploy in lockstep; Tier-2 screener negotiation and Tier-3 SNIP proposal; FROST disclosure port (BIP-340-in-Cairo spike first); relayer-market Starknet submitter and Cairo registry; Hyperlane outbound leg if demanded; third-party audit of all Cairo contracts and the account class.

## 10. References

- STRK20 announcement: https://www.starknet.io/blog/make-all-erc-20-tokens-private-with-strk20/
- Starknet v0.14.2 "The Privacy Engine Arrives": https://www.starknet.io/blog/starknet-v0-14-2-the-privacy-engine-arrives/
- SNIP-36, in-protocol proof verification: https://community.starknet.io/t/snip-36-in-protocol-proof-verification/116123
- Reference implementation: https://github.com/starkware-libs/starknet-privacy
- STRK20 by Example (source mirror; strk20-by-example.org): https://github.com/Akashneelesh/strk20-by-example
- Garaga (Groth16-BN254 verification in Cairo): https://github.com/keep-starknet-strange/garaga
- Research basis: internal multi-agent research run, 2026-07-18, with adversarial verification of load-bearing claims.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

[CSAP.md]: ./CSAP.md
[PSR.md]: ./PSR.md
[UAB.md]: ./UAB.md
[privacy-pool.md]: ./privacy-pool.md
[conditional-disclosure.md]: ./conditional-disclosure.md
[relayer-market.md]: ./relayer-market.md
[nullifier-registry.md]: ./nullifier-registry.md
[payload-format.md]: ./payload-format.md
