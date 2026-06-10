# PSR — Programmable Stealth Reputation

**Status:** Draft · **Canonical circuit version:** V2 (`stealth_reputation`, depth 20) · **License:** CC0-1.0
**Reference implementation:** `opaquecash/solana` (schema-registry, attestation-engine-v2, reputation-verifier, groth16-verifier). The Ethereum implementation runs the full V2 stack (`OpaqueSchemaRegistry`, `OpaqueAttestationRegistry`, `OpaqueReputationVerifierV2`, `Groth16VerifierV2`); Solana still pairs the V2 schema layer with a V1 verifier (see §5).

> PSR is Opaque's privacy-preserving reputation layer: a stealth identity can hold schema-bound attestations and prove possession of one — to a contract or a service — without revealing the stealth address, the wallet behind it, or any other attestation. This document specifies the **canonical V2 design**. Where the deployed code is still V1, that is called out explicitly; V2 is the target both chains converge on (project decision D3).

---

## Abstract

PSR composes three pieces over the CSAP stealth-address layer ([CSAP.md]): a **schema registry** that defines attestation types and who may issue them; an **attestation engine** that issues schema-bound, optionally-revocable attestations against a stealth identity; and a **Groth16 ZK circuit + on-chain verifier** that lets the holder prove "I own a stealth address carrying a valid attestation under schema X, for action Y" while revealing nothing else, with a nullifier that prevents reusing the proof for the same action. The scheme uses the BN254 curve and Poseidon hashing so the identical circuit verifies on Solana (via `alt_bn128` syscalls) and Ethereum (via the `ecPairing` precompile, EIP-197).

## Motivation

Reputation systems force a choice between public transparency (verifiable, not private) and private identity (private, not composable). PSR provides both: a verifier learns only "this party holds trait X for action Y," not the stealth address, the funding wallet, the transaction graph, or any unrelated trait. No comparable open primitive exists on either Ethereum or Solana — it is Opaque's primary differentiator and the basis for compliance-preserving privacy (a KYC provider can issue an attestation that a user later proves to a gate without doxxing).

V2 exists because V1 had two defects this spec fixes:
1. **Open issuance.** In V1 any wallet could attach any trait to any announcement — no notion of an authorized issuer. V2 introduces schemas with an authority and delegates.
2. **A circuit `is_valid` output.** V1 exposed a validity bit as a public output. A sound circuit must be *unsatisfiable* for an invalid witness, not emit a self-asserted validity flag. V2 removes it.

## Specification

Key words per RFC 2119. The PSR layer assumes CSAP key material and stealth identities.

### 1. Roles and identifiers

| Term | Definition |
|---|---|
| Authority | The wallet that registers a schema and permanently controls it. |
| Delegate | A pubkey the authority authorizes to issue under a schema (≤ 10 per schema). |
| Issuer | Authority or a delegate; the only parties that may `attest`. |
| `schema_id` | `SHA-256(authority_pubkey ‖ name_utf8 ‖ [version])`, 32 bytes. `version` is currently `1`. |
| `attestation uid` | `SHA-256(schema_id ‖ issuer_pubkey ‖ stealth_address_hash ‖ slot_le)`, 32 bytes. |
| `stealth_address_hash` | A 32-byte hash binding an attestation to a CSAP stealth identity without storing the address. |
| `external_nullifier` | A caller-chosen domain separator scoping a proof to one action context (e.g. "vote #42"). |
| `nullifier_hash` | `Poseidon(stealth_pk, external_nullifier)` — consumed once on-chain per action. |

### 2. Schema registry

A schema declares an attestation type and its issuance rules.

**State (`SchemaPDA`, seeds `["schema", authority, schema_id]`):** `authority`, optional `resolver` program (`default` = none), `revocable` (immutable after creation), `name` (≤ 64 chars), `field_definitions` (≤ 256 chars, ABI-style e.g. `"bool passed, u64 score"`), `version`, `delegates` (≤ 10), `created_at`, `schema_expiry_slot` (0 = none), `deprecated`.

**Instructions:**
- `register_schema(schema_id, name, field_definitions, revocable, resolver, schema_expiry_slot)` — the provided `schema_id` MUST equal `SHA-256(authority ‖ name ‖ [1])` (enforced on-chain).
- `add_delegate(delegate)` / `remove_delegate(delegate)` — authority only; ≤ 10 delegates.
- `update_resolver(new_resolver)` — authority only; `default` removes it.
- `deprecate_schema()` — authority only, irreversible; blocks new attestations, leaves existing ones valid.

**Predicates:** `is_authorized_issuer(p) = (p == authority) ∨ (p ∈ delegates)`; `is_active(slot) = ¬deprecated ∧ (schema_expiry_slot == 0 ∨ slot < schema_expiry_slot)`.

### 3. Attestation engine

**Instruction `attest(stealth_address_hash, data, expiration_slot, ref_uid)`** (issuer signs):
1. REQUIRE `data.len ≤ 512`.
2. REQUIRE `schema.is_authorized_issuer(issuer)` — the core V2 invariant.
3. REQUIRE `schema.is_active(current_slot)`.
4. `uid = SHA-256(schema_id ‖ issuer ‖ stealth_address_hash ‖ slot_le)`.
5. Write `AttestationPDA` at seeds `["attestation_v2", schema_id, issuer, stealth_address_hash]` with: `uid`, `schema_pda`, `schema_id`, `issuer`, `stealth_address_hash`, `data` (ABI-encoded per `field_definitions`), `created_at`, `expiration_slot` (0 = none), `revocation_slot = 0`, `ref_uid` (chained-credential pointer; 0 = none).
6. If `schema.resolver ≠ default`: CPI into the resolver's `on_attest` hook (discriminator `0x01`, then `schema_id ‖ issuer ‖ stealth_address_hash ‖ uid ‖ len(data) ‖ data`); any resolver error reverts issuance. This is the programmable gate (payments, allowlists, NFT ownership, …).
7. Emit `AttestationCreated`.

**Instruction `revoke(attestation_uid)`:** authority only (not delegates); REQUIRE `schema.revocable`; sets `revocation_slot = current_slot` (data is preserved for auditability, not deleted); fires the resolver `on_revoke` hook (discriminator `0x02`) if configured; emits `AttestationRevoked`.

**Validity:** `is_valid(slot) = (revocation_slot == 0) ∧ (expiration_slot == 0 ∨ slot < expiration_slot)`.

The resolver interface uses fixed 8-byte discriminators so any program can implement it with no IDL dependency.

### 4. The ZK statement (V2 circuit `stealth_reputation`, depth 20)

Curve BN254, hash Poseidon. Tree depth 20 (~1M leaves).

**Private inputs:** `stealth_pk` (BN254 field element of the stealth private scalar), `schema_id` (the 32-byte id packed into a field element), `issuer_pk_x` (the issuer/authority BabyJubJub x-coordinate), `trait_data_hash` (Poseidon of the ABI-encoded attestation data), `nonce` (random, prevents leaf enumeration), `merkle_path[20]`, `merkle_path_indices[20]`.

**Public inputs (in this order):** `merkle_root`, `attestation_id`, `external_nullifier`, `nullifier_hash`.

**Constraints:**
1. `leaf = Poseidon(stealth_pk, schema_id, issuer_pk_x, trait_data_hash, nonce)`.
2. Merkle inclusion: hashing `leaf` up `merkle_path` with `merkle_path_indices` (each constrained binary) yields `merkle_root`.
3. **Schema binding:** `schema_id == attestation_id` (so a leaf built for schema A cannot satisfy a request for schema B).
4. **Nullifier binding:** `Poseidon(stealth_pk, external_nullifier) == nullifier_hash`.

Keeping `issuer_pk_x`, `trait_data_hash`, and `nonce` private prevents an observer from enumerating leaves or learning who attested to whom; the proof reveals only the schema (`attestation_id`), the action scope (`external_nullifier`), and the one-time `nullifier_hash`.

> **V2 vs V1 (normative difference).** V1's circuit committed a leaf of `Poseidon(address_commitment, attestation_id)` and exposed public signals `[nullifier, is_valid, merkle_root, attestation_id, external_nullifier]` — five signals, with `nullifier` and `is_valid` as outputs. V2 has four public signals with `nullifier_hash` as a public **input** and no `is_valid`. The two are not interchangeable.

### 5. On-chain verification

**Solana — `reputation-verifier` (program `BSnkCDoTpgNVN5BbF3aN5L5EJPiaYUkqqj9MHp8kaqWM`):**
- `initialize()` → `VerifierConfig` (`admin`, `groth16_verifier` program id) at seeds `["verifier_config"]`.
- `update_merkle_root(root)` — admin only. Writes `MerkleRootEntry` (`root`, `timestamp`, seeds `["merkle_root", root]`) and appends to `RootHistory` (last `MAX_ROOT_HISTORY = 100`).
- `verify_reputation(proof_a[64], proof_b[128], proof_c[64], root, attestation_id, external_nullifier, nullifier)` — REQUIRE the root entry is ≤ `ROOT_EXPIRY_SECS = 3600` old; CPI into `groth16_verifier::verify_proof(A, B, C, pub_signals)`; on success initialize a `NullifierEntry` at seeds `["nullifier", nullifier]` (the `init` fails if it already exists → one-time use); emit `ReputationVerified`.
- `verify_reputation_view(...)` — same check without consuming the nullifier (returns `bool`).
- `transfer_admin(new_admin)` — `default` renounces.

**Solana — `groth16-verifier` (program `6mFaKyp7F4NqNeoiBLEWSqy5wJSk7rWf1EYumVXgHvhQ`):** verifies a Groth16 proof `(A[64], B[128], C[64])` against an array of 32-byte big-endian public signals using Solana's `alt_bn128` (BN254) syscalls; returns validity.

**Ethereum — `OpaqueReputationVerifierV2.sol` + `Groth16VerifierV2.sol`:** perform the V2 check against the four V2 public signals via the `ecPairing` precompile (`0x08`, EIP-197) and `ecAdd`/`ecMul` (`0x06`/`0x07`, EIP-196), with an equivalent root-history + nullifier registry. Schemas and attestations live in `OpaqueSchemaRegistry.sol` / `OpaqueAttestationRegistry.sol` (block numbers stand in for slots).

> **⚠ Solana's V2 upgrade is partially complete.** The `groth16-verifier` program source now implements `verify_proof_v2` with the real V2 verifying key (transcribed from the production trusted setup and pinned by the committed proof fixture in `circuits/test/fixtures/v2/`; its earlier swapped BN254 field constants are also fixed). Two gaps remain: (1) the **deployed** devnet program predates these fixes and requires an upgrade, and (2) `reputation-verifier::verify_reputation` still builds the **V1** public-signals array `[nullifier, is_valid=1, root, attestation_id, external_nullifier]` and CPIs the V1 entry point — it must be rewritten to the V2 layout (CPI `verify_proof_v2`, supply `nullifier_hash` as an input, drop `is_valid`, keep the `attestation_id`/`external_nullifier` field encoding consistent with the circuit). **Ethereum has completed this upgrade** (`OpaqueReputationVerifierV2` + `Groth16VerifierV2`, four V2 signals). Verifying V2 proofs against the V1 verifying key/signal layout MUST NOT be done — the layouts are incompatible and a mismatch either fails closed or, worse, verifies the wrong statement.

### 6. Nullifiers and Sybil resistance

`nullifier_hash = Poseidon(stealth_pk, external_nullifier)` is deterministic per (stealth identity, action). The on-chain `NullifierEntry` PDA keyed by the nullifier makes a second proof for the same action fail (the account already exists). Choosing a fresh `external_nullifier` per action scope (vote id, campaign id, loan id) is how an application controls anti-replay granularity.

### 7. What is public vs. private

**Public:** the schema being proven (`attestation_id`), the `external_nullifier` action scope, the `nullifier_hash`, proof success/failure, and the verifier event trail. **Also public, by storage:** a schema's `name`/`field_definitions` and each attestation's `data` and `issuer` live in readable on-chain accounts. **Private:** the stealth private key, the stealth address, the wallet behind it, the transaction graph, and any unrelated attestation.

> Privacy comes from stealth-address unlinkability + the ZK proof, **not** from hiding attestation contents. Issuers that place sensitive values in `data` MUST encrypt or hash them; only `stealth_address_hash` (never the address) is stored, but `data` is otherwise plaintext on-chain.

## Rationale

- **Schemas with authority/delegates** close V1's open-issuance gap, which otherwise lets anyone mint arbitrary reputation (spam + Sybil). The model deliberately mirrors the Ethereum Attestation Service vocabulary (schema → attestation → resolver) so the mental model and tooling transfer, while adding stealth + ZK privacy that EAS lacks.
- **BN254 + Poseidon** so one circuit verifies on Solana (`alt_bn128` syscalls) and Ethereum (`ecPairing`), and so Merkle hashing is cheap in-circuit.
- **`nullifier_hash` as a public input (V2)** lets the program handle the value directly while the circuit's constraint `Poseidon(stealth_pk, external_nullifier) == nullifier_hash` guarantees the proof actually binds it.
- **Resolver CPI hooks** make issuance programmable (payment-gated KYC, allowlists) without changing the engine.
- **`stealth_address_hash`, not the address** binds an attestation to a stealth identity without publishing it.

## Backwards Compatibility

PSR is an Opaque-original primitive, not an existing EIP/SIMD. It builds on CSAP (so it inherits scheme id 1 and the stealth identity model) and is conceptually compatible with EAS-style schema/attestation/resolver tooling. The only internal compatibility concern is the V1↔V2 verifier mismatch in §5, which is an implementation task, not a standards conflict. Cross-chain: `schema_id`, `uid`, and the ZK statement are hash- and curve-defined and therefore chain-portable; the divergence is purely that Solana currently runs the V2 schema layer with a V1 verifier, while Ethereum runs the full V2 stack. Completing the Solana verifier upgrade unifies them.

## Test Vectors

Provide, in `opaquecash/circuits/test/`, at least:
1. **Schema id:** fixed `authority`, `name`, `version=1` → expected `SHA-256` `schema_id`.
2. **Attestation uid:** fixed `schema_id`, `issuer`, `stealth_address_hash`, `slot` → expected `uid`.
3. **Circuit:** a full witness (private inputs + path) and the four public signals, with a proof that verifies, plus negative cases — wrong `attestation_id` (schema-binding fails), tampered `merkle_root` (inclusion fails), and a second proof reusing the same `nullifier_hash` (on-chain rejection). Outputs marked `GENERATE` are produced by running the reference circuit + verifier and cross-validated (Task 0.5).

## Security Considerations

- **V1/V2 verifier mismatch (§5)** is the highest-priority correctness item: never verify V2 proofs against the V1 verifying key/signal layout.
- **Admin-controlled roots.** `update_merkle_root` is admin-only and roots expire after 1 hour. This is a liveness/censorship trust point. Mitigations: hold `admin` in a multisig (Squads / Gnosis Safe), and move toward permissionless/verifiable root computation. Proofs MUST be pinned to their generation root and resubmitted against that root on retry.
- **On-chain `data` is public.** See §7 — privacy is from unlinkability + ZK, not from the attestation payload. Encrypt sensitive fields.
- **Issuer trust.** A schema's reputation is only as good as its authority/delegates; a compromised delegate can mint attestations under that schema until removed. Keep delegate sets small; prefer revocable schemas for sensitive credentials.
- **Nullifier scope.** Reusing an `external_nullifier` across contexts links those actions for one identity; using a fresh scope per action preserves unlinkability.
- **Soundness.** Relies on the Groth16 trusted setup (use a published ceremony, e.g. Hermez Powers of Tau) and on circuit correctness — both REQUIRE a ZK-specific audit before mainnet (Veridise / zkSecurity / Hexens).
- **Field encoding.** `attestation_id`/`external_nullifier` cross the program boundary as `u64` today (`u64_to_be32`) but are field elements in-circuit; the V2 rewrite MUST keep the on-chain encoding and the circuit's field packing consistent, or proofs silently fail to bind.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

---

### Reference deployments (for reviewers)

| Chain | Program / Contract | Address / Program ID | Version |
|---|---|---|---|
| Solana Devnet | `schema_registry` | `FbgMJYGWnLKLcrKYS1NxM5uER1ihQkYLMTLs4STuDMWB` | V2 |
| Solana Devnet | `attestation_engine_v2` | `4T9kPCVCFGdEuLpEqRJihsPCbEEo2LWWDEPFvUESEqtM` | V2 |
| Solana Devnet | `reputation_verifier` | `BSnkCDoTpgNVN5BbF3aN5L5EJPiaYUkqqj9MHp8kaqWM` | V1 verifier (to upgrade) |
| Solana Devnet | `groth16_verifier` | `6mFaKyp7F4NqNeoiBLEWSqy5wJSk7rWf1EYumVXgHvhQ` | V1 vkey (to upgrade) |
| Ethereum Sepolia | `OpaqueSchemaRegistry` | `0xAA5F3942117bD48E7Cd81A500A8b7Bbb122ae80f` | V2 |
| Ethereum Sepolia | `OpaqueAttestationRegistry` | `0x049aF9CBB62387034CDd5403794a94E9c000ACCc` | V2 |
| Ethereum Sepolia | `OpaqueReputationVerifierV2` | `0x18cEc2812953c2E9bcADE20CbF6415BD36aEb44f` | V2 |
| Ethereum Sepolia | `Groth16VerifierV2` | `0x49A212bdbc52F1cb6C93623FC7814a61Fc71ddB5` | V2 vkey |

[CSAP.md]: ./CSAP.md
