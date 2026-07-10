# Specification Changelog

Version history of the Opaque protocol specifications, newest first. Each spec
carries its own status badge; this file records the cross-cutting decisions and
normative changes from CSAP v1 forward.

## 2026-07-10

- **PSR.md §4–§5 — PSR public signals are field-reduced and carried as 32-byte
  field elements (security fix OPQ-008).** `external_nullifier = keccak256(scope)`
  and any SHA-256-derived `attestation_id` are now reduced modulo the BN254 scalar
  field `r` at a single source of truth, so the witness signal, the on-chain public
  input, and the Poseidon preimage are the identical reduced value — the un-reduced
  256-bit value made the EVM verifier's `checkField` revert. The Solana
  `reputation-verifier` now takes `attestation_id`/`external_nullifier` as
  `[u8; 32]` big-endian (mirroring the EVM `uint256`) instead of `u64`, which had
  silently capped the domain at 2^64 and made spec/EVM-encoded proofs
  unrepresentable on Solana.
- **PSR.md §5 — reputation verifier schema binding + initializer authorization
  (security fixes OPQ-006, OPQ-007).** `OpaqueReputationVerifierV2` gains an
  admin-set schema-registry binding: when configured, a proof's `attestation_id`
  must reference a live registered schema, so a proof cannot claim reputation under
  a never-registered schema. (Binding the *issuer* still requires the in-circuit
  commitment to the schema authority — the tracked remediation.) The Solana
  privacy-pool and reputation-verifier `initialize` handlers are now gated to the
  program's upgrade authority, closing the unprotected-initializer front-run that
  could seize the ASP authority or record a rogue Groth16 verifier; the verifier's
  Groth16 program is additionally pinned by address.
- **Off-chain hardening (no spec change): OPQ-003/005/009/010/011/012/013.** Relayer
  nodes re-derive the payload hash and simulate the inner call before bonding (and
  submit atomically on Solana); the ASP never advances its Solana cursor past an
  undecoded deposit and recomputes each `label = Poseidon(scope, leafIndex)` rather
  than trusting the RPC; the Solana relayer validates the job account before
  slicing; the V2 attestation scanner drops rogue-issuer traits by default; the
  relayer client binds bids to the user's own jobId/chain; and the frontend caches
  the setup signature under a non-extractable IndexedDB key.

## 2026-07-06

- **ONS.md §3 — ons-mirror revoke tombstones instead of closing (security fix
  OPQ-004).** Closing the mirror PDA on revoke destroyed the stored
  `wormhole_sequence`, so a later-delivered but lower-sequence upsert VAA could
  re-create the record fresh (the monotonic floor is only skipped for a brand-new
  `name_hash == 0` PDA) and resurrect a revoked name at stale keys. A revoke now
  updates the record in place: it advances `wormhole_sequence`, keeps `name_hash`,
  zeroes the key material, and sets a new `revoked` flag. `OnsRecord` gains a
  trailing `revoked: bool` (account +1 byte); resolvers treat a revoked record as
  unresolved.

## 2026-07-05

- **CSAP.md §2.1–§2.3, §2.8 — native ed25519 Solana stealth accounts
  (security fix OPQ-002).** The Solana stealth account is now a native ed25519
  key derived by a DKSAP tweak on the ed25519 curve: the meta-address gains a
  third 32-byte half `S_ed = reduce(s_ed)·B` (so it is **98 bytes**,
  `V‖S‖S_ed`; `s_ed` comes from a third HKDF block, leaving `v`/`s`
  unchanged), the address is `P_ed = S_ed + h_ed·B` with
  `h_ed = SHA-512("opaque-solana-stealth-v2" ‖ sec) mod L`, and the one-time
  spend scalar `a = (reduce(s_ed) + h_ed) mod L` is reconstructable only with
  `s_ed`. The withdrawn construction seeded the account from
  `SHA-256("opaque-solana-stealth-v1" ‖ uncompressed(P_stealth))`, which the
  payer knows — letting the payer recompute the account key and sweep the
  funds. Sender funding and view-only balance scanning still work from public
  material; only the spend side is recipient-bound. Legacy 66-byte
  meta-addresses stay valid for Ethereum and fail closed on Solana. The Solana
  `stealth_registry` now stores 98-byte meta-addresses.
- **CSAP.md §2.7 (Solana registry) — `register_keys_on_behalf` now verifies an
  Ed25519 signature (security fix OPQ-001).** The instruction previously
  performed no authorization check, so anyone could overwrite any account's
  meta-address. It now requires an Ed25519 `SigVerify` instruction (introspected
  via the Instructions sysvar) proving the `registrant` signed a canonical
  message bound to the program id, registrant, scheme, current nonce, and
  meta-address; the nonce is consumed to prevent replay.

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
