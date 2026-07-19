# UAB — Universal Announcement Bus

**Status:** Draft · **License:** CC0-1.0 · **Transport:** Wormhole (generic messaging / Core Contract)

> The UAB makes a CSAP stealth announcement emitted on one chain visible to the other chain's
> scanner, with no central server. A sender publishes the 96-byte [payload-format.md] body
> through the Wormhole Core Contract; the guardian network signs it into a VAA; an off-chain
> relay submits the VAA to the destination chain's receiver, which re-emits it as a local
> announcement event. Privacy is unchanged — the body carries only what an EIP-5564
> announcement already exposes.

## Abstract

CSAP announcements are chain-local: an Ethereum scanner never sees a Solana announcement and
vice versa. The UAB bridges them over [Wormhole](https://wormhole.com), whose guardian network
(19 independent validators, 13/19 threshold) signs each published message into a **VAA**
(Verifiable Action Approval). The signed body is the fixed 96-byte cross-chain payload. Each
chain gains a *sender* (publishes to Wormhole, alongside the unchanged native announce) and a
*receiver* (verifies an incoming VAA's emitter, then re-emits the announcement locally).

```
Sender (EVM)                                 Sender (Solana)
  UABSender.announceWithRelay                  stealth-announcer::announce_with_relay
        │ emits Announcement (compat)                │ emits Announcement (compat)
        │ IWormhole.publishMessage(payload)          │ CPI core post_message(payload)
        ▼                                            ▼
  Wormhole Core (Sepolia 0x4a8b…)            Wormhole Core (devnet 3u8hJU…)
        └───────────────► Guardian Network ◄─────────┘
                          (signs → VAA)
                                │   off-chain relay fetches the VAA
                                │   and submits it to the destination
        ┌───────────────────────┴───────────────────┐
        ▼                                            ▼
  UABReceiver.receiveAnnouncement(vaa)        uab-receiver::receive_announcement(posted_vaa)
        │ emits CrossChainAnnouncement                │ emits CrossChainAnnouncement
        ▼                                            ▼
  Ethereum scanner (sees both chains)         Solana scanner (sees both chains)
```

## Components

| Role | Ethereum | Solana |
|---|---|---|
| Sender | `UABSender.sol` — emits the legacy `Announcement` (backwards compat) and calls `IWormhole.publishMessage(nonce, payload, consistencyLevel)` | `stealth-announcer::announce_with_relay` — emits `Announcement` and CPIs the Core Contract `post_message` |
| Receiver | `UABReceiver.sol` — `parseAndVerifyVM`, emitter check, emits `CrossChainAnnouncement(sourceChain, sourceEmitter, payload)` | `uab-receiver::receive_announcement` — reads a posted VAA account, emitter check, emits `CrossChainAnnouncement` |
| Relay | off-chain agent (Phase 1: a single trusted process; Phase 4: a permissionless market) | same agent, opposite direction |

The native `announce` / `announce_with_log` paths are **untouched**; `announceWithRelay` /
`announce_with_relay` are additive. Cross-chain visibility is opt-in per send.

## Wormhole constants (Testnet)

|  | Ethereum Sepolia | Solana devnet |
|---|---|---|
| Core Contract | `0x4a8bc80Ed5a4067f1CCf107057b8270E0cC11A78` | `3u8hJUVTA4jH1wYAyUur7FFZVQ8H635K3tSHHF4ssjQ5` |
| Wormhole chain id | `2` | `1` |

Re-verify at deploy time against `wormhole.com/docs/products/reference/contract-addresses`.

### Reference deployments (Testnet)

| Role | Ethereum Sepolia | Solana devnet |
|---|---|---|
| UAB sender | `0x872787c0BD1A0C71e6D1be5a144EB044e0CB2069` (UABSender) | `HGFn2fH7bVQ5cSuiG52NjzN9m11YrB3FZUfoN9b9A5jf` (stealth_announcer · `announce_with_relay`) |
| UAB receiver | `0x9eF189f7a263F870Cf80f9A89d1349A6AF7b15cF` (UABReceiver) | `7d4Sbmmpy954JwSNdjwf31pgbeWUQqwpgNdte5iy3vuM` (uab_receiver) |
| Wormhole emitter | `0x000…872787c0bd1a0c71e6d1be5a144eb044e0cb2069` | PDA `Ay5gspEYbCwKg2feCipJyrFWBp1W6EwScwDbnW6aTMKJ` |

Both directions are verified end-to-end through the live testnet guardian network (`opaquecash/relayer`).

## Emitter identity and authentication

A VAA carries `(emitterChainId, emitterAddress, sequence)`. `emitterAddress` is 32 bytes:

- **EVM emitter** = the 20-byte `UABSender` address, **left-padded** to 32 bytes.
- **Solana emitter** = the posting program's emitter PDA (`seeds = ["emitter"]`), 32 bytes.

Each receiver is configured with the expected `(chain, emitter)` of the opposite chain's
sender and **MUST reject** a VAA from any other emitter or chain. Replay is prevented by
recording consumed `(emitterChain, emitterAddress, sequence)` triples on the receiver (on
Solana, a posted VAA is also naturally consumed once).

## Message flow — Ethereum → Solana

1. A sender calls `UABSender.announceWithRelay(...)`. The contract emits `Announcement`
   (so local scanners still see it) and calls `publishMessage(payload)`. Wormhole logs
   `LogMessagePublished` with sequence `N`.
2. Guardians observe Sepolia at the requested consistency (**finalized**) and produce a
   signed VAA for `(chain=2, emitter=UABSender, sequence=N)`.
3. The relay fetches the VAA (by emitter + sequence, via the TS SDK `getVaa` or Wormholescan),
   posts it to the Solana Core Contract (`verify_signatures` + `post_vaa`), then calls
   `uab-receiver::receive_announcement(posted_vaa)`.
4. `uab-receiver` checks the emitter, decodes the 96-byte body, and emits
   `CrossChainAnnouncement`. The Solana scanner consumes it like a native announcement.

## Message flow — Solana → Ethereum

Symmetric. `announce_with_relay` CPIs `post_message`; the relay fetches the VAA and calls
`UABReceiver.receiveAnnouncement(vaaBytes)` on Sepolia, which `parseAndVerifyVM`s it, checks
the emitter is the Solana `uab` emitter PDA on chain 1, and emits `CrossChainAnnouncement`.

## Relaying

Wormhole's **automatic relayer is EVM-only**; any leg touching Solana is **manual** — an
off-chain agent must fetch the signed VAA and submit it to the destination. The relay is a
**liveness-only** role: it cannot forge a VAA (guardian signatures) nor alter the payload
(the body is signed), so a faulty or censoring relay can only delay delivery, and anyone may
run one. Phase 4 replaces the single Phase-1 relay with a permissionless, staked market.

## Consistency / finality

EVM publishes at `consistencyLevel = 200` (finalized) so guardians wait for Sepolia finality
(minutes on testnet); Solana posts at `Finalized` commitment. Lower levels (instant/safe)
reduce latency at the cost of reorg safety; UAB uses finalized.

## Fees

`publishMessage` may charge a Wormhole message fee (`0` on testnet today; query
`messageFee()`); the caller also pays normal gas. On Solana, `post_message` requires the
message-account rent plus any fee paid to the Core Contract fee collector, plus tx fees.

## Backwards compatibility

UAB builds on CSAP (it carries scheme id `1` and the stealth identity model) and adds no new
required behaviour to existing contracts/programs. A scanner that ignores
`CrossChainAnnouncement` keeps working unchanged.

## Security considerations

- **Emitter allowlist + replay protection** (above) are mandatory; without the emitter check
  any party could relay a forged-origin announcement.
- **No new privacy surface.** The body exposes only what a native announcement already does
  (view tag, ephemeral key, stealth address, small metadata). Unlinkability and ZK properties
  are unaffected.
- **Guardian trust.** Integrity rests on the 13/19 guardian threshold; this is Wormhole's
  trust model, inherited deliberately to avoid a single point of failure.
- **Relay liveness.** A single Phase-1 relay is a liveness dependency, not an integrity one:
  it can delay or drop relays but cannot forge them (integrity rests on the guardian threshold
  above). The relayer market decentralises it ([relayer-market.md](./relayer-market.md)). This
  is the canonical statement of the cross-chain relay liveness dependency.

## Copyright

Copyright and related rights waived via [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/).

[payload-format.md]: ./payload-format.md
