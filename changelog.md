# Specification Changelog

Version history of the Opaque protocol specifications, newest first. Each spec
carries its own status badge; this file records the cross-cutting decisions and
normative changes from CSAP v1 forward.

## 2026-07-01

- **relayer-market.md §9.4 added (sweep submission interface); §9.2 and §3.4
  amended.** The fee-in-token sweep gains a normative HTTP interface on the node
  gateway: `POST /v1/sweep` (chain-tagged EVM forwarder calldata or partially
  signed Solana transaction; synchronous, not re-gossiped) and
  `GET /v1/sweep/info` (per-chain operator + EVM forwarder). §9.2 now requires
  co-signing over the owner's original blockhash and recommends the fee-payer and
  token-program-whitelist guards; §3.4 lists the new routes. The previously
  optional Solana in-token fee transfer is implemented end-to-end (SDK builds the
  `amount - fee` / `fee` split; the reference node serves the endpoint).

## 2026-06-13

- **relayer-market.md §9 added (fee-in-token sweep).** Escrow-free, gasless
  withdrawal of an ERC-20/SPL balance from a one-time stealth address that holds
  the token but no native gas. Ethereum uses a stateless `StealthTokenSweep`
  forwarder: the owner signs an EIP-712 `Sweep` authorization
  (`token,owner,destination,value,fee,nonce,deadline`, domain
  `OpaqueStealthTokenSweep`/`1`) plus an EIP-2612 `permit`; the relayer submits,
  pays gas, and is reimbursed `fee` in-token while the destination receives
  `value - fee`. Solana needs no program: the relayer is the fee payer while the
  reconstructed stealth keypair signs the SPL transfer as authority. No bond or
  slashing (no escrow at risk); destination, amount, and fee are owner-signed, so
  a relayer can only execute the signed sweep or nothing. Complements, and is
  independent of, the native-fee job escrow (§5).

## 2026-06-12

- **conditional-disclosure.md Draft v1.** Threshold viewing keys: selective
  M-of-N disclosure without full viewing-key exposure. Active path is a
  pool-scoped Groth16 proof (`conditional_disclosure.circom`: state-tree
  membership + `value > threshold` qualification + disclosed `value`/`label`)
  gated by a **FROST(secp256k1) / BIP-340** custodian quorum signature verified
  on-chain (ecrecover trick / `secp256k1_recover`) over the proof's `context`;
  Shamir splitting of the viewing key is the documented recovery backstop
  (reconstruction reveals the full key — normative warning). Closes the
  conditional-disclosure nullifier reservation:
  `Poseidon(nullifier, context, DOMAIN_DISCLOSURE)` with
  `DOMAIN_DISCLOSURE = keccak256("opaque/disclosure/v1") mod r`.
- **nullifier-registry.md updated:** privacy-pool row marked live
  (`Poseidon(nullifier)`), disclosure row specified; arity-based layout
  separation noted.
- **privacy-pool.md Draft v1.** Amount privacy via the Privacy Pools
  (Buterin/Soleimani association-set) construction: `commitment =
  Poseidon(value, label, Poseidon(nullifier, secret))`, an append-only state
  tree of commitments and an ASP-curated association tree of labels, a
  partial-withdrawal Groth16 proof (`withdrawal.circom`) that folds
  state-membership + association-membership + nullifier + value-accounting +
  `context` binding into one proof, and a standalone `association.circom` ASP
  statement. **Verified ERC-8065 is the unrelated ZK Token Wrapper, not an
  association-set pool — no ERC-8065 compatibility is claimed.** Mainnet gated
  on circuit + contract audits, a production trusted-setup ceremony, and a
  legal opinion.

## 2026-06-11

- **relayer-market.md Draft v1.** Gas-private submission market: combined
  stake-registry + job-escrow contract per chain so submit-or-slash is
  on-chain-verifiable (create/accept-bond/submit/slash/cancel), GossipSub
  `opaque/jobs/v1` wire formats (advert, stake-backed bid, NaCl-box payload
  delivery), HTTP gateway intake, bond = fee, native-asset pre-funded fees
  (fee-in-proof deferred), and UAB/ONS VAA delivery as a node duty replacing
  the Phase-1 central relay.
- **ONS.md Draft v1.** Opaque Name Service: `*.opq.eth` subnames over an
  ENSIP-10 wildcard resolver on Ethereum (canonical), Wormhole-mirrored
  read-only PDAs on Solana, claims from either chain. Normative consistency
  model is **eventually consistent, canonical-chain-wins** — no atomic
  cross-chain claim exists or is claimed. Defines the 164-byte mirror payload
  (Ethereum → Solana), the variable-length claim payload (Solana → Ethereum),
  the provisional-claim reconciliation states
  (`pending`/`confirmed`/`lost`/`expired` with a 24 h pending window), and the
  four SDK resolution paths. Meta-addresses are sourced from the deployed
  ERC-6538 registry, not a parallel store.

## 2026-06-10 (V1 retirement + client V2)

- **V1 `stealth_attestation` retired.** Circuit removed from `circuits/`
  (history retains it), V1 entry point and verification key removed from the
  Solana `groth16-verifier` (redeployed), V1 artifacts removed from the app.
  PSR.md is V2-only; the V1↔V2 mismatch warnings are resolved.
- **TypeScript prover migrated to V2.** `@opaquecash/psr-prover` builds the V2
  leaf (`Poseidon(stealth_pk, schema_id, issuer_pk_x, trait_data_hash, nonce)`)
  with deterministic dev-mode defaults, parses the four V2 signals, and
  defaults to the `circuits/v2` artifacts. A freshly client-generated proof was
  verified on devnet through the SDK submit path — the full pipeline
  (prove → root registration → on-chain verify → nullifier consumption) is V2
  end to end.

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
