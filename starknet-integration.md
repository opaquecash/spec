# Starknet Integration — PSR × STRK20 Design

**Status:** Living design document — P0/P1 implemented and live on Starknet Sepolia; Tier-1 STRK20 prototype validated against the upstream pool built from source · **Date:** 2026-07-18, revised 2026-08-03 · **License:** CC0-1.0
**Depends on:** [CSAP.md], [PSR.md], [UAB.md], [privacy-pool.md], [conditional-disclosure.md], [relayer-market.md], [nullifier-registry.md]

> This document records the integration design for extending Opaque to Starknet as a third chain, with STRK20 (Starknet's shielded ERC-20 pool standard) as the primary integration target. Normative changes it implies (CSAP scheme constants, chain ids, account-class pinning) land in the referenced specs. Facts below were researched and adversarially cross-verified against primary sources on 2026-07-18 and re-verified against the upstream source tree (`starkware-libs/starknet-privacy` @ `c02eca0`, 2026-08-03); remaining single-source claims are marked **[unverified]**.

---

## 1. Abstract

STRK20 gives Starknet a single canonical shielded pool for every ERC-20, with mandatory auditor viewing-key escrow for after-the-fact disclosure. It has **no eligibility layer**: no association sets, no inclusion/exclusion proofs, no credential gating at entry or in-pool. PSR is exactly that missing layer — a schema-bound, ZK-proven, nullifier-scoped answer to "is this actor eligible?" that never identifies the actor. The integration ports only PSR's thin on-chain footprint to Cairo (a Garaga-generated Groth16-BN254 verifier plus a root/nullifier wrapper), consumes STRK20 through three permission tiers (permissionless in-pool gates first, StarkWare's screening seam second, an in-circuit eligibility SNIP third), and adds Starknet as a third CSAP chain via a counterfactual secp256k1 account class. The privacy pool, the Shamir viewing-key backstop, and the token-sweep forwarder are explicitly **not** ported: STRK20 and Starknet account abstraction subsume them.

## 2. Background: STRK20 as verified

- **One pool, all tokens.** STRK20 is a single canonical shielded-pool Cairo contract for all ERC-20s, not a token-side interface. Tokens adopt it permissionlessly by being deposited; no wrappers or token changes. Reference: `starkware-libs/starknet-privacy`, `packages/privacy/src/interface.cairo`.
- **Note-based UTXO model.** Encrypted notes `(owner, token, u128 amount)` in write-once cells keyed by channel-derived hashes; Poseidon nullifiers incorporating the owner's private key; all-or-nothing spends with change notes; "open notes" (plaintext amount, filled at execution) for DeFi composition. The nullifier set grows linearly; sub-linear state growth is explicitly unsolved upstream.
- **Proving = SNIP-36** (Starknet v0.14.2 "Shinobi", mainnet 2026-04-21). The client executes against a virtual snapshot of a recent block and proves with Stwo (Circle STARK, ~29 s); the proof rides new optional `proof` / `proof_facts` fields on Invoke V3 and is verified by **Starknet consensus, not the Starknet OS**. StarkWare's own framing: degraded security versus native apps, privacy "de facto" rather than formally zero-knowledge, formal ZK deferred to later phases. A ~500 KB proof costs ≈ 75M L2 gas to propagate.
- **Compliance = two legs, both now documented in source and public writing (2026-08 re-verification).** (1) Mandatory per-user registration of a viewing public key plus the viewing private key encrypted to the audit firm — whole-history, after-the-fact disclosure on lawful request. The audit master key is held by **Financial Privacy Inc (FPI)** in a dual-enclave TEE, governed by a 3-of-4 StarkWare/FPI multisig with a 7-of-12 Starknet Security Council backup (StarkWare's "compliance layer" post, 2026-07-14). (2) Entry screening: `IServer::apply_actions(actions: Span<ServerAction>, screening: Option<ScreeningAttestation>)` — an attestation is **mandatory for every regular-pool deposit** and rejected otherwise. It is a STARK-curve ECDSA signature over the SNIP-12 `DepositorValidation { depositor, issued_at }` typed message (max age 300 s, 60 s future skew), verified against a **single admin-set `screener_public_key`** held by FPI, whose pipeline fronts an Elliptic screening service (`elliptic-proxy/` in the upstream repo). Third parties can be screening *clients* of that pipeline; there is no multi-screener registry — a second screener-of-record is a StarkWare protocol change.
- **Contract depositors are supported.** `assert_valid_signature` (`packages/privacy/src/utils.cairo`) authorizes the acting `user_addr` via any of three paths: (I) the account's SRC5-advertised `is_custom_signature_valid(calls, additional_data, signature)`, (II) SNIP-6 `is_valid_signature` over the tx hash, (III) `is_valid_signature` over the SNIP-12 `CallSet` hash. Path I means a **contract with arbitrary validation logic can be the depositor-of-record** — upstream's own `StarknetEth712Account` e2e (EVM-signature depositor, merged 2026-08-02) and Opaque's `strk20_gate` suite (§5) both exercise it. This resolves the former §8 blocking unknown affirmatively.
- **No eligibility layer.** Confirmed negative, unchanged at `c02eca0`: no association sets, no ASP role, no inclusion/exclusion proofs, no credential hooks. Screening is identity/sanctions-shaped (one FPI key over an Elliptic pipeline), not programmable eligibility.
- **Composition pattern.** Anonymizer helper contracts (`ekubo_swap_anonymizer`, `vesu_lending_anonymizer`, `sub_account_anonymizer` — now first-class packages in the upstream workspace) are invoked *from the pool* during a proven spend via the external-invocation action (max once per transaction bundle).
- **Ecosystem state (2026-08-03).** STRK20 and the pool are **live on Starknet mainnet** (v0.14.2 carried the protocol support; strkBTC live since 2026-05-12; "Privacy Is Now Live on Starknet" launch post; builder program opened with "Push to Private", 2026-07-15). Pool contract reports version `2.0`; V1 contracts were audited by OpenZeppelin (2026-05-29, `docs/audit/` upstream). AVNU private swaps are live; wallet support = Ready (live) and Xverse (rolling out) via the Privacy Wallet API v0.10.3 / starknet.js v10.4. The TS SDK lives in-repo (`client/`, Apache-2.0) with a self-hostable open-source prover crate (~29 s on a 12-core default build); it is still not on npm. The **mainnet pool address remains unpublished** — operator-env-injected in the upstream demo, recoverable from any shield transaction once one is captured.

## 3. Thesis

Viewing keys answer *"who did this?"* after the fact. Screening answers *"is this depositor sanctioned?"* at entry, against one vendor's list. PSR answers *"does this actor hold the required credential?"* before the fact, programmably and without identifying them. STRK20 now has the first two and still structurally lacks the third: the screening seam is filled by a **single FPI-held key over an Elliptic pipeline** — identity/sanctions screening, not schema-bound eligibility. Nothing in the pool can express "only KYB-attested treasuries", "only holders of credential X", or any issuer-defined predicate.

Positioning: Opaque occupies that gap as the **neutral, multi-chain, multi-issuer credential layer**, composed with (not against) the FPI screening leg. Timing signals (both to watch, neither verified beyond a single source): StarkWare demoed a first-party "Private KYC" flow (2026-06-24, passport NFC → attribute registry → ZK queries) **[unverified]**, and SilentSwap is publicly pitching a compliance layer at this exact seam — the same forum thread that articulates the attestation gap is also, therefore, competition.

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
- **Keeper.** The relayer TS keeper gains a Starknet root-publication duty. The 1 h TTL and the OPQ-018 rebuild-excluding-revoked obligation ([PSR.md](./PSR.md) §5) now apply on a third chain, so keeper liveness carries across all three.

## 5. How STRK20 consumes PSR proofs — three permission tiers

**Tier 1 — permissionless, PROVEN: credential-gated entry through a contract depositor.** The pool's `assert_valid_signature` path I (SRC5 `is_custom_signature_valid`) lets a contract account be the depositor-of-record with arbitrary validation logic. Opaque's `PsrGatedDepositor` (`opaquecash/starknet`, `contracts/strk20_gate`) answers `VALIDATED` only while the account holds a live PSR credential (the `verify_and_consume` adapter pattern proven by `PsrGate` on Sepolia) AND the owner signed the exact calls, with the standard fallback paths disabled — so **every pool action that account authorizes, including its deposits, is credential-gated at the boundary**. This is validated end-to-end against the real upstream pool built from source: the full fresh-account shielded deposit (`SetViewingKey → OpenChannel → OpenSubchannel → Deposit → CreateEncNote`) compiles through the production `__execute__` and applies through `apply_actions` with a screener attestation, and the rejection matrix holds (no credential / wrong owner key / missing or wrong-screener attestation all fail). The complementary anonymizer-style form — PSR-gated in-pool external actions (gated swap, gated lending, gated escrow), following upstream's own `*_anonymizer` package pattern — gates *flows* the same way and needs no StarkWare permission either.

> **Honest limit, revised.** The former blocking unknown ("can a contract be depositor-of-record?") is resolved YES and exercised. What Tier 1 still cannot do: force *other* depositors through a gate (a direct deposit by any account bypasses any external gate — pool-wide eligibility needs Tier 2/3), and the gated account's deposits still additionally require the FPI screening attestation like everyone else's (composition, not replacement).

**Tier 2 — negotiated: PSR inside the screening path.** The screening seam is now known to be a single FPI-held `screener_public_key` fronting an Elliptic pipeline, with third parties as screening *clients*. The realistic Tier 2 is therefore: (a) PSR proof verification as an **additional pre-condition inside the FPI/partner screening flow** before an attestation is signed (a policy integration with FPI/StarkWare, no protocol change), and (b) the standing ask for a **multi-screener registry** so an eligibility-scoped screener can exist as a peer key (a StarkWare protocol change to the pool). `strk20_gate` is the working artifact for that conversation.

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
  - **Salt rule (pinned):** `salt = sn_keccak(R)` over the 33-byte compressed ephemeral key `R`. Both parties compute it from the announcement, so the address is derivable without interaction. Note that per-payment unlinkability does **not** depend on the salt: because `P_stealth` (fresh per payment via CSAP's fresh `R`) is itself the constructor calldata, the address already varies per payment for any fixed salt — the `R`-derived salt only binds the address to its specific announcement. The address is `calculate_contract_address_from_hash(salt, class_hash, [x.low, x.high, y.low, y.high], deployer=0)`, cross-validated against `starknet.js` (`@opaquecash/stealth-chain-starknet` `computeStarknetStealthAccount`).
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

   > **Status: live end-to-end.** The Starknet consumer (`opaquecash/starknet`, `contracts/ons_mirror`, `OpaqueNameMirror`) is an `#[l1_handler]` that checks the sequencer-injected L1 sender against an admin-set emitter allowlist, enforces a monotonic sequence (floor bypassed only for a genuinely new name), and tombstones a revoke in place (keys zeroed, floor retained) per OPQ-004 — mirroring the Solana `ons_mirror` semantics ([ONS.md](./ONS.md) §3) over native messaging instead of Wormhole. The Ethereum sender (`opaquecash/ethereum`, `StarknetOnsMirrorSender`) owns a strictly increasing sequence and calls `sendMessageToL2`; a cross-check test asserts its felt payload matches the consumer's Serde layout exactly. Both are deployed on Sepolia (mirror `0x0355472f…508328`, sender `0xbdB9D13B…9131Ef`, allowlisted as the emitter), and a name upsert published from the sender was delivered across the bridge and read back from the mirror with its exact keys intact — the L1→L2 mirror is proven end-to-end, not just in tests.
2. **Hyperlane** where a direct or low-latency leg is needed: live on Starknet mainnet (audited Cairo Mailbox), production Solana↔Starknet routes since 2025-10. The interchain gas paymaster is undeployed — plan to run our own relayer, which fits the existing keeper pattern.
3. LayerZero: live on Starknet since 2026-01; arbitrary-payload path from Cairo unconfirmed. Wormhole and Axelar: not viable.

Starknet→outbound: L2→L1 finality is ~3–4 h — acceptable for registry sync, unusable for interactive UAB. **Launch Starknet without outbound UAB announcements**; revisit with Hyperlane if demanded.

## 8. Risks and blocking unknowns

1. **Felt252 vs BN254 r** (§4.1b) — soundness-critical limb handling.
2. **Ceremony/vkey lockstep redeploy** across three chains, with the Cairo verifier regenerated, not reconfigured.
3. **OPQ-018 revocation window inherited** (see [PSR.md](./PSR.md) §5): the Starknet keeper must rebuild-excluding-revoked within the 1 h TTL, same as the other chains.
4. **AA custody friction:** receiving to an undeployed counterfactual address works, but sweeping requires `DEPLOY_ACCOUNT` first — worse than an EVM stealth EOA — and needs a SNIP-9/paymaster flow that does not create fee-payer linkage.
5. **STRK20 governance gates Tiers 2–3:** the screener key is singular and FPI-held, the auditor leg is FPI-operated under StarkWare multisig governance, and the mainnet pool address is still unpublished. Tier 1 is the only zero-permission path — and it is now built and validated; sequence accordingly.
6. **Proof-system mismatch is permanent** unless Tier 3 lands: PSR stays Groth16-BN254/circomlib-Poseidon; STRK20 is Stwo/Cairo under SNIP-36 Phase 1 (consensus-verified, "de facto" privacy). Composition happens at the contract layer.
7. **Cost:** measured on Sepolia — ~43.1M L2 gas for a live `verify_reputation`, ~155.47M end-to-end for the PsrGate flow; workable, headroom re-checked per network upgrade.
8. **Toolchain fork:** the upstream pool tracks Cairo 2.17 / snforge 0.59 (v0.14.2 `proof_facts` in `TxInfo`) while this repo pins 2.16.1/0.57; `strk20_gate` carries its own toolchain and CI job until the repo-wide pin catches up.

**Resolved since 2026-07-18:** contract-as-depositor — YES, via `assert_valid_signature` path I (`ICustomSignatureValidation`, SRC5), exercised by upstream's Eth712Account e2e and Opaque's `strk20_gate` suite; ERC-20 rollout — live on mainnet; screener governance — documented (single FPI key, Elliptic pipeline, third parties are clients).

**Still open:** the mainnet pool address (operator-env only; recoverable from a captured shield tx — needed for the live-fork variant of the `strk20_gate` suite); SNIP-36 third-party circuit access (Tier 3); BIP-340 verification cost in Cairo (FROST port).

**Prior art:** StealthPay (ERC-5564-on-STARK-curve hackathon project) prefigures the announcer/registry/view-tag design on Starknet; differentiation is CSAP's cross-chain identity plus the PSR layer. Cite it in any public write-up.

## 9. Phased roadmap

**P0 — proof of concept. DONE (2026-07-19).** Garaga-generated verifier, reputation-verifier wrapper, proof encoder, and root publication all live-validated on Starknet Sepolia; fourth-verifier CI cross-check green; gas measured (§8.7).

**P1a — Starknet PSR + stealth primitives. DONE (2026-07-19), extended 2026-08-03.** Full stealth stack (announcer, SNIP-12 registry, pinned `stealth_account` class) plus `PsrGate` live on Sepolia. The Tier-1 STRK20 integration landed as `contracts/strk20_gate`: a PSR-credential-gated contract depositor completing a real deposit against the upstream pool built from source, with the rejection matrix tested (§5). Remaining from this phase: migrate the CSAP constants + test vectors into [CSAP.md] and fix its stale §2.6 chain-id table.

**P1b — three-chain client surface. DONE (2026-07-19 → 2026-08-03).** TS `StarknetAdapter` + three-chain facade/app refactor shipped and published; L1→L2 ONS mirror live end-to-end; Rust scanner adapter landed (crates.io publish pending). Facade Starknet write paths (`registerMetaAddress`, `submitReputationVerification` via a configured `starknet.account`, Garaga encoding behind a lazy import) landed 2026-08-03 with the app's proof-submission UI following the next SDK release.

**P2 — production (externally gated).** Post-ceremony vkey redeploy in lockstep (full Garaga regeneration, §4.1); Tier-2 FPI/StarkWare engagement using `strk20_gate` as the working artifact, plus the Tier-3 SNIP draft; live-pool fork variant of the `strk20_gate` suite once the mainnet pool address is captured; paymaster/SNIP-9 flow for the dust-sweep AA friction (§8.4); FROST disclosure port (BIP-340-in-Cairo spike first); relayer-market Starknet submitter and Cairo schema registry; Hyperlane outbound leg if demanded; third-party audit of all Cairo contracts and the account class.

## 10. References

- STRK20 announcement: https://www.starknet.io/blog/make-all-erc-20-tokens-private-with-strk20/
- Starknet v0.14.2 "The Privacy Engine Arrives": https://www.starknet.io/blog/starknet-v0-14-2-the-privacy-engine-arrives/
- Launch: "Privacy Is Now Live on Starknet": https://www.starknet.io/blog/privacy-live-on-starknet/
- Compliance architecture (FPI, TEE, multisig governance): https://www.starknet.io/blog/compliance-layer-onchain-privacy-strk20/
- Builder program: "Push to Private" (2026-07-15): https://www.starknet.io/blog/push-to-private/
- SNIP-36, in-protocol proof verification: https://community.starknet.io/t/snip-36-in-protocol-proof-verification/116123
- Reference implementation (re-verified @ `c02eca0`, 2026-08-03): https://github.com/starkware-libs/starknet-privacy
- Opaque Tier-1 prototype: https://github.com/opaquecash/starknet — `contracts/strk20_gate`
- STRK20 by Example (source mirror; strk20-by-example.org): https://github.com/Akashneelesh/strk20-by-example
- Garaga (Groth16-BN254 verification in Cairo): https://github.com/keep-starknet-strange/garaga
- Research basis: internal multi-agent research run 2026-07-18 with adversarial verification of load-bearing claims; source-tree re-verification 2026-08-03 (`assert_valid_signature`, screening SNIP-12 message, anonymizer packages, toolchain).

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
