# Specification Changelog

Version history of the Opaque protocol specifications, newest first. Each spec
carries its own status badge; this file records the cross-cutting decisions and
normative changes from CSAP v1 forward.

## 2026-06-10

- **Solana on-chain PSR V2 complete.** Devnet `groth16-verifier` and
  `reputation-verifier` upgraded: real V2 verifying key, fixed field constants,
  `verify_reputation` rewritten to the V2 signal layout (`nullifier_hash` as
  input via `verify_proof_v2` CPI). First real proof verified on devnet with
  nullifier consumption and replay rejection. Client gap: the TypeScript
  prover still targets V1 and is the remaining migration.

### Earlier the same day


- **V2 verification key pinned.** A real Groth16 proof generated with the
  production V2 proving key is now committed as a fixture in
  `circuits/test/fixtures/v2/` and verified in CI against the snarkjs vkey, the
  Ethereum `Groth16VerifierV2`, and the Solana `groth16-verifier` program. The
  vkey transcribed in both on-chain verifiers is thereby locked to the circuit.
- **Solana verifier corrections (implementation, no spec change).** The
  `groth16-verifier` program had its BN254 scalar/base field constants swapped
  (negation used `r` instead of `q`, so no real proof could pass) and carried
  placeholder V2 IC points. Both fixed; the real V2 verification key installed.
  The deployed devnet program predates the fix and requires an upgrade.
- **`nullifier-registry.md` Draft v1.** Canonical 32-byte BE field-element
  encoding, consume-once registry semantics, domain separation, chain-local
  replay model with `external_nullifier` chain scoping.
- **CSAP §2.9 added (ONS seed).** `com.opaque.meta` ENSIP-5-style text-record
  key for resolving meta-addresses through existing name services; read path
  only until ONS ships.

## 2026-06-09

- **UAB live on testnet, both directions** (Ethereum Sepolia ⇄ Solana devnet):
  `UABSender`/`UABReceiver` on Sepolia, `announce_with_relay` + `uab-receiver`
  on devnet, central delivery relayer as a documented stop-gap.
  [UAB.md](./UAB.md) and [payload-format.md](./payload-format.md) describe the
  deployed behaviour.
- **`payload-format.md` Draft v1.** 96-byte chain-neutral announcement payload;
  corrected `source_chain_id` to Wormhole chain ids (earlier draft used an
  ASCII tag for Solana).

## 2026-06-08

- **Execution decisions D1–D5 resolved**, fixing spec-relevant ground rules:
  V2 (`stealth_reputation`) is the canonical circuit and V1
  (`stealth_attestation`) is deprecated and frozen (D3); the Stellar
  implementation is archived and out of spec scope (D5).
- **PSR.md draft (canonical V2).** Schema registry, attestation engine, V2
  circuit with public signals `[merkle_root, attestation_id,
  external_nullifier, nullifier_hash]`. The V1↔V2 signal-layout incompatibility
  is documented as normative.
- **CSAP.md Draft v1.** DKSAP over secp256k1 + Keccak-256 with scheme id `1`,
  HKDF domain `"opaque-cash-v1"`, 66-byte `V‖S` meta-address (EIP-5564
  byte-order deviation recorded as an open decision), 8-bit view tags,
  ERC-5564/6538-compatible announcement and registry interfaces, and the
  cross-validated test vectors in `circuits/test/test_vectors.json`.

## Versioning policy

Specs are individually versioned by their status line (Draft v1, …). A breaking
change to deployed behaviour — key derivation, payload layout, circuit public
signals, registry interfaces — requires either a new scheme id (CSAP), a new
payload version (UAB), or a new circuit version (PSR), plus an entry here.
Editorial changes are not recorded.
