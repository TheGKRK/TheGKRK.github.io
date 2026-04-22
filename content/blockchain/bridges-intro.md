+++
title = "Cross-Chain Bridges: What They Are and How They Work"
date = 2026-04-23
description = "A ground-up introduction to cross-chain bridges - why they exist, the trust models that separate them, and a deep dive into sync committees and the SupraNova light-client bridge architecture."
[taxonomies]
tags = ["blockchain", "bridges", "interoperability", "cross-chain", "ethereum", "hypernova"]
[extra]
toc = true
+++

Blockchains don't talk to each other natively. Bitcoin doesn't know Ethereum exists. Ethereum doesn't know Supra exists. Each chain is a sovereign system with its own state, consensus, and finality rules. Bridges are the infrastructure that connects them.

In this post I am going to explain [SupraNova](https://supranova.ai/) - the first trustless bridge with MoveVM as the destination execution environment. Starting from first principles of how bridges work, and ending with how SupraNova is architected under the hood.

## The Interoperability Problem

Imagine a user holding USDC on Ethereum who wants to use a DeFi protocol on Solana. Without a bridge, they would need to sell their USDC for fiat, buy SOL, then rebuy USDC on Solana - paying exchange fees and slippage twice. Bridges exist to make this unnecessary.

But why can't chains just talk to each other directly?

Blockchain consensus is deterministic. Every validator must execute the same transactions and arrive at the same state. The moment a validator queries an external chain, that determinism breaks - different validators may see different external states, and consensus fails.

This is why chains are isolated by design. It's not a limitation to fix; it's a property of how Byzantine fault-tolerant systems work.

> Bridges work around this by moving information off-chain and anchoring it back on-chain in a verifiable way.

## Bridge Architecture

Every bridge, regardless of type, has three components:

**Source chain contracts** - Lock or burn assets, emit events that off-chain infrastructure listens to.

**Off-chain infrastructure** - Relayers, validators, or provers that observe source-chain events and produce some form of attestation (signatures, proofs, or messages) to be submitted on the destination.

**Destination chain contracts** - Verify the attestation, mint or release assets.

The trust model of a bridge is almost entirely determined by the off-chain component - who runs it, how many parties are involved, and what cryptographic guarantees they provide.

## Bridge Types

### Trustless Bridges (Light Client / ZK)

The destination chain verifies source-chain state directly. Either a light client tracks the source chain's consensus and verifies Merkle inclusion proofs, or a ZK circuit proves that a state transition was valid.

No external validator set. The cryptographic guarantees of the source chain *are* the trust.

**Cost:** 300-400k gas on the destination (proof verification is expensive).  
**Latency:** 13+ minutes for Ethereum (one finality period).  
**Trust:** Source chain consensus. If the source chain is sound, the bridge is sound.

Examples: IBC (Cosmos), HyperNova (ETH/BSC to SUPRA).

### Trusted / Multisig Bridges

A set of validators (often M-of-N threshold) observe source-chain events and co-sign messages to be submitted on the destination. The destination contract verifies the aggregate signature.

You trust the majority of validators won't collude or get compromised.

**Cost:** ~77k gas on the destination (signature verification is cheap).  
**Latency:** Seconds to minutes depending on validator response time.  
**Trust:** Validator set. If M validators are compromised, funds are at risk.

Examples: Wormhole (19-of-19 guardians), LayerZero (configurable oracle + relayer).

The $320M Wormhole hack and $600M Ronin hack both exploited validator key compromise. Large locked TVL and a small validator set is a high-value target.

### Intent-Based Bridges

Instead of moving assets through a bridge contract, a user expresses an intent - "I want X on chain B, I'll give Y on chain A." Solvers compete to fill the order, fronting the destination-side funds. Once the solver is confirmed on the source side, they're reimbursed.

**Cost:** Low. No proof verification on destination.  
**Latency:** Near-instant settlement for the user.  
**Trust:** Solver liveness. If no solver fills the order, the user is stuck until timeout.

Examples: Across Protocol, UniswapX.

### L2 Bridges

L2s derive security from L1 by posting state roots and (for optimistic rollups) allowing fraud proofs, or (for ZK rollups) posting validity proofs. The canonical bridge between L1 and its L2 inherits this security.

**Withdrawal latency:** 7 days for optimistic rollups (challenge window), near-instant for ZK rollups with proof aggregation.  
**Trust:** L1 consensus + the rollup's proof system.

Examples: Arbitrum bridge, Optimism bridge, zkSync bridge.

### Sidechain Bridges (Two-Way Peg)

A sidechain is an independent chain with its own consensus, pegged to a main chain via a bridge. Assets are locked on the main chain and unlocked on the sidechain, with a federation or validator set managing the peg.

Security is entirely dependent on the sidechain's validators - it doesn't inherit L1 security.

Examples: Polygon PoS bridge (pre-AggLayer), Ronin bridge.

## Use Cases

Bridges aren't just for moving tokens. There are three categories:

**Asset transfer** - Wrapped tokens, canonical bridges, liquidity networks. The most common use case. User wants native or wrapped asset on another chain.

**Message passing** - Arbitrary data cross-chain: governance votes, oracle updates, contract calls. The bridge carries a payload that triggers execution on the destination.

**Application-specific** - Bridges built for one protocol's needs - cross-chain lending, cross-chain NFT ownership, omnichain governance. Often use general-purpose messaging layers (LayerZero, Axelar, Wormhole) as the transport.

## Security vs. Cost vs. Latency

No universal solution exists. The right bridge depends on what you're optimizing for.

| Type | Gas Cost | Latency | Trust Assumption |
|---|---|---|---|
| Light Client / ZK | High (300-400k) | High (13+ min) | Source chain consensus |
| SupraNova (finality) | Low | ~12 min | Ethereum consensus |
| SupraNova (safe) | Low | ~4 min | Ethereum consensus |
| Multisig / MPC | Low (~77k) | Low (seconds) | Validator set majority |
| Intent-based | Low | Very low (instant) | Solver liveness |
| L2 canonical | Medium | High (optimistic: 7d) | L1 + proof system |
| Sidechain | Low | Low | Sidechain validators |

For high-value, security-critical transfers, the gas and latency overhead of trustless bridges is worth it. For frequent small transfers where speed matters and validator risk is acceptable, multisig or intent-based bridges make more sense.

The bridge landscape has shifted from convenience toward correctness. The $1B+ lost to bridge hacks in 2022 made this shift inevitable. Trustless bridges - light client or ZK-based - are harder to build but eliminate the attack surface of validator key compromise entirely.

---

The rest of this post goes deeper into *how* a trustless light-client bridge actually works - using SupraNova as the concrete example.

## Ethereum Sync Committees

The core question for any trustless ETH bridge: how does the destination chain verify Ethereum state without running a full node?

Full nodes process every block, every transaction, every attestation. That's gigabytes of data and significant compute. Impossible to do on-chain on another chain.

The answer is **sync committees** - Ethereum's light client primitive introduced in the Altair upgrade.

{% stats() %}
{{ stat(value="512", label="Validators per committee") }}
{{ stat(value="256", label="Epochs per rotation") }}
{{ stat(value="8,192", label="Slots per period") }}
{{ stat(value="~27.3h", label="Rotation period") }}
{% end %}

### How Sync Committees Work

A sync committee is a randomly selected group of 512 validators from the full validator set. Their job is to attest to the current chain head every slot (every 12 seconds), producing a compact signature that a light client can verify cheaply.

Here is what happens slot by slot:

**Step 1 - Signing.** Every slot, all 512 sync committee members are expected to sign the current beacon block header using their BLS12-381 private key.

**Step 2 - Aggregation.** These 512 individual BLS signatures are aggregated into a single compact aggregate signature. BLS signatures have a useful property: multiple signatures over the same message can be combined into one, and verified against the aggregate of the corresponding public keys. The aggregate signature is the same size as a single signature.

**Step 3 - Inclusion.** The aggregate signature and a participation bitfield (which of the 512 members actually signed) are packed into a `SyncAggregate` and included in the *next* block's body.

**Step 4 - Light client verification.** A light client holds the current committee's `aggregate_pubkey` - the BLS aggregate of all 512 members' public keys. To verify a block header, it only needs:

- The block header.
- The `SyncAggregate` (aggregate signature + participation bits).

It computes the aggregate pubkey of the *participating* members (using the bitfield to exclude absentees), then verifies the aggregate signature against it. One pairing operation. No need to track 500k+ attestations.

### Deterministic or Non-Deterministic?

This is a question worth answering precisely, because the answer is both.

**Committee selection is pseudo-random but deterministic.** Validators are selected using RANDAO - a verifiable randomness beacon built into Ethereum's consensus. Given the RANDAO value for an epoch, the selection of the 512 sync committee members is a deterministic computation. Any validator (or anyone observing the chain) can compute the next committee in advance. There is no unpredictability once the RANDAO value is known.

**Signature verification is fully deterministic.** Given a block header, a `SyncAggregate`, and the committee's public keys, the verification result is always the same. There is no ambiguity. This is what makes it usable on-chain: the Move module on Supra runs the same deterministic BLS verification and all nodes reach the same conclusion.

**Participation is non-deterministic in practice.** Which of the 512 members actually sign each slot is not predictable in advance. Validators can be offline, have network issues, or choose not to participate. The bridge must handle this - verification uses only the participating members' aggregate pubkey, derived from the bitfield.

{% card() %}
For a bridge, what matters is that *verification is deterministic*. Given the same inputs, every node on the destination chain will reach the same result. This is the property that makes on-chain light client verification possible.
{% end %}

### Committee Rotation

Committees rotate every 256 epochs (~27.3 hours). The next committee is announced one full period in advance - the Beacon API exposes it via the `next_sync_committee` field in `LightClientUpdate`.

This is critical for bridges: the on-chain light client must track rotations. When a period ends, it must update its stored committee. This is done via a `LightClientUpdate` that carries:

- The `next_sync_committee` and a Merkle branch proving its inclusion in the beacon state.
- A `sync_aggregate` from the current committee signing off on the update.

The on-chain module verifies the signature, verifies the Merkle branch, and stores the new committee. If the bridge misses a rotation by more than one period, it loses the ability to verify new headers - it no longer knows the current committee's keys.

### How Is Sync-Committee Consensus Secure?

The security of sync committee-based verification rests on one assumption: at most 1/3 of the 512-member committee is Byzantine (controlled by an attacker).

Given Ethereum's validator set of 500k+ validators, what is the probability that an attacker controlling a fraction *f* of the total stake ends up with more than 1/3 of a randomly selected 512-member committee?

This is a hypergeometric distribution problem. The math works strongly in the honest majority's favor: even an attacker with 33% of total stake (the threshold for disrupting full consensus) has a negligible probability of controlling 1/3 of a random 512-member committee. The committee is small enough to sample efficiently but large enough that the law of large numbers keeps it representative.

The HyperNova whitepaper formalizes this security argument and provides probability bounds. The key takeaway: sync committee-based verification inherits Ethereum's security guarantees at a fraction of the verification cost, with a statistically negligible degradation in the adversarial threshold.

{% card(type="purple") %}
Sync committee verification is slightly weaker than full consensus - it requires 2/3+ of 512 validators to be honest rather than 2/3 of all 500k+. But for a randomly selected committee drawn from a large honest-majority set, the practical security is equivalent. This is the fundamental insight behind Ethereum's light client design.
{% end %}

## Building a Bridge with Sync Committees

Knowing what sync committees are is one thing. Building a bridge with them is another. Here is the process.

**Step 1 - Bootstrap.** The on-chain light client needs a trusted starting point - an initial sync committee and a finalized header to anchor to. This comes from a trusted checkpoint: a known finalized block root at a known slot. The `LightClientBootstrap` API endpoint returns the current sync committee's pubkeys and an SSZ Merkle branch proving the committee is in the beacon state at that checkpoint. This is submitted once to initialize the light client on the destination chain.

**Step 2 - Track committee updates.** The committee updater continuously submits `LightClientUpdate` messages as new periods begin. Each update carries the next committee (with Merkle proof) and a sync aggregate from the current committee signing off on it. The on-chain light client verifies and rotates.

**Step 3 - Track headers via updates.** The committee updater continuously submits header updates to keep the on-chain light client's view of Ethereum current. SupraNova supports three update types:

- **Optimistic update** (`LightClientOptimisticUpdate`) - the latest attested header signed by the sync committee. Available every slot (~12 seconds). Not finalized, but useful for applications that prioritize speed.
- **Finality update** (`LightClientFinalityUpdate`) - a finalized checkpoint header with a Merkle proof of finality. Available every ~12 minutes. Cannot be reorged.
- **Safe update** - introduced by SupraNova. A header that has crossed a configurable justification threshold, sitting between optimistic and finality in both latency (~4 minutes) and security guarantees.

The on-chain light client stores the latest verified header for whichever update mode the operator has configured.

**Step 4 - Prove deposit events.** Once the on-chain light client holds a verified finalized header, the relayer can prove that a `MessagePosted` event occurred in a block at or before that header. It does this with a receipt Merkle proof: a path through the `receipts_root` of the finalized execution block proving that a specific transaction receipt (containing the deposit event) is included.

**Step 5 - Verify and mint.** The destination bridge contract checks the receipt proof against the `receipts_root` from the stored finalized header, decodes the deposit event, checks the fee and nonce, and mints the wrapped asset.

### Proof Verification Process

Settling a bridge transfer requires three distinct proofs, verified in sequence by the destination contracts. Each proof builds on the previous one - a chain of cryptographic trust from Ethereum's consensus layer down to a specific event log.

#### Signature Proof

The signature proof establishes that the submitted Ethereum block header is part of the canonical chain. It is a BLS12-381 aggregate signature produced by the sync committee.

**How BLS aggregation works.** BLS signatures have a special algebraic property: multiple signatures over the same message can be combined into a single signature of the same size. The verification equation for an aggregate signature is:

```
e(σ_agg, G2) == e(H(message), pk_agg)
```

Where `e` is a bilinear pairing on BLS12-381, `σ_agg` is the aggregate signature, `G2` is the generator of the G2 group, `H(message)` is the hash of the signed message mapped to G1, and `pk_agg` is the aggregate of the participating members' public keys.

The `SyncAggregate` in the Ethereum block carries two fields:

- `sync_committee_bits` - a 512-bit bitfield indicating which committee members participated.
- `sync_committee_signature` - the aggregate BLS signature of all participants over the beacon block header.

**Verification on MoveVM.** The light client module:

1. Reads the stored current committee's 512 individual public keys.
2. Uses `sync_committee_bits` to select only the participating members.
3. Aggregates their public keys into a single `pk_agg`.
4. Runs BLS12-381 signature verification: `verify(σ_agg, H(header), pk_agg)`.

Step 4 requires a BLS12-381 pairing operation - expensive elliptic curve math that has no native support on MoveVM. We implemented this as a native function in Supra's VM runtime: a Rust function exposed directly to the Move execution layer. The Move contract calls it like any other function; the VM dispatches it to the Rust implementation.

If verification passes, the submitted header is stored as the new trusted Ethereum head. Everything downstream depends on this.

{% card(type="purple") %}
A valid signature from 2/3+ of the sync committee is the only on-chain authority for what Ethereum's state looks like. There are no oracles, no multisig signers, no external validators. The math is the trust.
{% end %}

#### Ancestry Proof — Block Root and Historical Root

The signature proof gives us a trusted beacon block header with a verified `state_root`. The ancestry proof uses this `state_root` to establish the canonical block root of the target block — without walking a chain of parent hashes.

**How Ethereum commits to past block roots.**

The Ethereum beacon state is a large SSZ structure. Two of its fields are critical for ancestry proofs:

- `block_roots` — a fixed-size vector of the 8,192 most recent beacon block roots, one per slot. It acts as a rolling window of recent history.
- `historical_summaries` — for blocks older than 8,192 slots, Ethereum accumulates chunks of 8,192 block roots into a `HistoricalSummary`, each of which contains a `block_summary_root` (the Merkle root of a 8,192-slot chunk of block roots).

The key insight: **the beacon state is committed to in the beacon block header via `state_root`**, and the sync committee signs the beacon block header. This means the sync committee's BLS signature transitively commits to every block root stored in `block_roots` and `historical_summaries`.

**Proving the target block for recent slots.**

If the deposit occurred within the last 8,192 slots:

1. The relayer provides a Merkle branch from `state_root` into `block_roots[target_slot % 8192]`.
2. The contract verifies the branch using SSZ Merkle proof verification.
3. The result is `block_root` — the canonical beacon block root for the target slot, directly trusted because it is committed to in the committee-signed state.

**Proving the target block for older slots.**

If the deposit is older than 8,192 slots:

1. The relayer provides a Merkle branch from `state_root` into `historical_summaries[period]`, extracting the `block_summary_root` for the relevant period.
2. It then provides a second Merkle branch from `block_summary_root` into the specific `block_roots[slot % 8192]` within that period's chunk.
3. Both branches are verified in sequence, yielding the same canonical `block_root`.

In both cases, the output is identical: a `block_root` that is cryptographically tied to the sync committee's signature.

**From block root to receipts root.**

The `block_root` is the SSZ hash of the beacon block. The beacon block body contains an `execution_payload` — the execution layer block. A Merkle branch proves the `execution_payload_header` (which contains `receipts_root`) is included in the beacon block. The contract verifies this branch and extracts `receipts_root`.

This gives us a fully trusted `receipts_root` for the target block, anchored all the way back to the committee's signature.

**The manipulation resistance.**

This is the security property that makes the design tight. Suppose an attacker wants to prove a fake deposit - one that never happened on Ethereum. They would need to provide a fake execution block with a fabricated `MessagePosted` event. To pass the ancestry proof, their fake block's root must match `block_roots[slot]` in the beacon state.

But `block_roots[slot]` is committed to in `state_root`, which is committed to in the beacon block header, which is signed by the sync committee. To replace `block_roots[slot]` with a different value, the attacker would need to forge a BLS aggregate signature from 2/3+ of 512 validators — computationally infeasible.

{% card(type="purple") %}
The sync committee's signature does not just vouch for one header. It vouches for the entire beacon state — including every block root in `block_roots` and every historical summary in `historical_summaries`. An attacker cannot substitute a different target block without breaking the Merkle branch, and they cannot forge a new valid signature to cover a tampered state. The block root mismatch is detectable, and there is no way around it.
{% end %}

#### Receipt Proof

With a trusted `receipts_root` established through the ancestry proof, the receipt proof proves that a specific transaction receipt — containing the `MessagePosted` event — is included in that block.

**Ethereum's receipt trie.**

Ethereum stores all transaction receipts for a block in a Merkle Patricia Trie (MPT). The trie root is `receipts_root`, committed to in the execution block header and now trusted through the chain: signature → state root → block root → execution payload → receipts root.

Each receipt is keyed by its transaction index (RLP-encoded), and the leaf contains the full RLP-encoded receipt.

**Proof verification.**

The relayer constructs a Merkle Patricia Trie inclusion proof for the specific receipt — a sequence of trie nodes from `receipts_root` down to the leaf. The verifier:

1. Starts at the trusted `receipts_root`.
2. Follows the key path (derived from the transaction index) through branch and extension nodes, verifying each node's hash against its parent's reference.
3. Reaches the leaf containing the RLP-encoded receipt.
4. Decodes the receipt and locates the `MessagePosted` event in its log list.

**Event decoding.**

Each log in a receipt has three fields:

- `address` — must match the known bridge contract address on Ethereum.
- `topics[0]` — must equal `keccak256("MessagePosted(address,uint256,address,uint256)")`, the event signature fingerprint.
- `data` — ABI-encoded parameters: asset, amount, recipient, nonce, fee.

The bridge module validates the address and event signature, then ABI-decodes `data` to extract the deposit parameters used for minting on Supra.

**Why receipt root manipulation fails.**

If an attacker provides a fake receipt with a fabricated deposit amount, the MPT proof would not match the trusted `receipts_root` — the trie hash would differ. And since `receipts_root` is committed to inside the `block_root` that the sync committee signed, the attacker cannot produce a valid alternate `receipts_root` without breaking the committee signature. The two verification paths — ancestry and receipt — reinforce each other.

{% warning() %}
All three proofs must pass for a mint to execute. Signature proof fails — rejected at the light client, nothing downstream runs. Ancestry proof fails — the block root does not match the signed state, rejected. Receipt proof fails — the receipt is not in the trie or the event does not match, rejected. There is no partial success path and no override.
{% end %}

## SupraNova Verification Strategy

Most light-client bridges support only one update type and commit to a fixed security/latency tradeoff. SupraNova is designed to be **configurable** - operators can choose from three verification modes depending on the application's requirements.

### Optimistic Updates

Uses `LightClientOptimisticUpdate` - the latest block header attested by the sync committee, regardless of finality. The sync committee has signed it, but the block has not been finalized by the full beacon chain.

**Latency:** Near-instant (next slot, ~12 seconds).  
**Security:** Weaker - the block can still be reorged if Ethereum has a liveness failure. Suitable for low-value transfers or applications where speed is paramount.

### Finality Updates

Uses `LightClientFinalityUpdate` - a finalized checkpoint header with a Merkle proof of finality. The block cannot be reorged.

**Latency:** ~12.8 minutes (two epochs).  
**Security:** Strongest. This is the standard mode for production high-value transfers.

### Safe Updates

This is a mode we introduced in SupraNova - not part of the standard Ethereum light client spec.

A safe update is a `LightClientUpdate` that has crossed a configurable confirmation threshold short of full finality. Specifically, it requires the attested header to be included in a chain that has sufficient justification weight - more than the optimistic latest head but without waiting for the full two-epoch finalization window.

This gives bridge operators a tunable middle ground. A protocol that wants faster settlement than 12.8 minutes but stronger guarantees than optimistic can configure a safe update threshold that matches its risk model.

{% card(type="green") %}
The three-mode design means SupraNova can serve different applications without redeployment. A gaming application using low-value in-game assets can use optimistic mode for instant UX. A DeFi protocol moving large positions uses finality mode. A protocol that needs to balance both can configure safe mode with an appropriate threshold. The on-chain contracts handle all three; the operator configures which mode the committee updater submits.
{% end %}

## SupraNova Bridge Architecture

SupraNova (HyperNova) is a trustless bridge - an Ethereum light client running on Supra's MoveVM. There is no external validator set. Security reduces entirely to Ethereum's consensus.

The system has five components.

### Source Contracts

Solidity contracts deployed on Ethereum (and BSC for the BSC↔SUPRA direction). These are the entry point for users.

When a user wants to bridge, they call the deposit function with the asset and recipient address. The contract locks the asset and emits a `MessagePosted` event containing the amount, recipient, nonce, and fee. Nothing else - the contract is deliberately minimal. It doesn't know anything about Supra, sync committees, or proofs. Its only job is to custody assets and emit a verifiable record of deposits.

On the return path (Supra→ETH), the source contract handles withdrawals: it verifies a proof of a burn event on Supra and releases the locked asset.

### Relayer

The relayer is the off-chain workhorse. Written in Rust with Tokio, it watches finalized Ethereum blocks for `MessagePosted` events and drives them through to settlement on Supra.

For each finalized deposit event, the relayer:

1. Fetches the full transaction receipt from Ethereum.
2. Constructs a Merkle proof of the receipt's inclusion in the block's `receipts_root`.
3. Submits the proof and receipt to the destination bridge contract on Supra.

The relayer implements **pre-concurrent block fetching** - it prefetches upcoming blocks before they finalize, so proof construction starts immediately once finality lands. **Parallel submission** keeps multiple proofs in-flight simultaneously rather than waiting serially.

The relayer also handles **live configuration updates** - fee parameters, RPC endpoints, and relay intervals can be changed at runtime without restarting the process. This was important for production operations where downtime has a cost.

{% card() %}
A compromised relayer cannot steal funds or mint unauthorized tokens. The worst it can do is go offline and delay relaying. All security guarantees are enforced on-chain by the destination contracts - the relayer is untrusted infrastructure.
{% end %}

### Committee Updater

This is a separate process from the relayer, with a distinct responsibility: keeping the on-chain light client's sync committee up to date.

The committee updater continuously polls the Ethereum Beacon API for `LightClientFinalityUpdate` and `LightClientUpdate` messages and submits them to the light client module on Supra.

Critically, this is fully trustless and cryptographically verifiable. The committee updater is not a trusted party — it is untrusted infrastructure, exactly like the relayer. Every message it submits is verified on-chain by the Move light client module before being accepted:

- A `LightClientUpdate` (committee rotation) carries the `next_sync_committee` with an SSZ Merkle branch proving its inclusion in the beacon state, plus a `sync_aggregate` from the current committee signing off on the transition. The on-chain module verifies the BLS signature and the Merkle branch. If either is invalid, the update is rejected.
- A `LightClientFinalityUpdate` carries a finalized header with a Merkle branch proving finality, plus a `sync_aggregate`. Same verification: BLS check, Merkle check, reject on failure.

The committee updater cannot submit a false committee or a fabricated finalized header. Any tampered data produces an invalid signature or a Merkle branch mismatch — caught and rejected on-chain. The committee updater's only power is liveness: it can choose not to submit, but it cannot submit anything that passes verification unless Ethereum's consensus actually produced it.

Separating this from the relayer is a deliberate design choice. The light client must keep advancing regardless of whether any deposits are happening. If the committee updater falls more than one rotation period (~27h) behind, the on-chain light client loses the ability to verify signatures because it no longer knows the current committee's keys. Keeping it as an independent process with its own liveness guarantees makes this easier to reason about and operate.

### Destination Contracts

Two Move modules on Supra handle the receiving side.

**Light client module** - Stores the current sync committee's aggregate public key and the latest finalized Ethereum block header. When the committee updater submits a `LightClientFinalityUpdate`, this module:

1. Verifies the BLS12-381 aggregate signature from the sync committee over the attested header.
2. Updates the stored finalized header if the signature is valid.
3. On committee rotation, verifies the `next_sync_committee` Merkle branch against the beacon state root and updates the stored committee keys.

**Bridge module** - When the relayer submits a deposit proof, this module:

1. Verifies the receipt Merkle proof against the `receipts_root` from the stored finalized header.
2. Decodes the `MessagePosted` event from the receipt.
3. Checks the fee, verifies the nonce hasn't been replayed.
4. Mints the wrapped asset to the recipient.


### Indexer

The indexer is a read-only service that tracks the state of in-flight and completed bridge transactions. It watches both Ethereum and Supra for events, correlates deposits with mints, and exposes this data via an API that the frontend uses to show users the status of their bridge transactions.

It also serves as an operational tool - the team uses it to monitor bridge health, detect stuck transactions, and verify that the relayer and committee updater are keeping up.

{% warning() %}
The indexer is purely informational. It has no ability to influence on-chain state. Users can verify any bridge transaction independently by checking the source and destination contracts directly - the indexer is a convenience layer, not a trust assumption.
{% end %}

## End-to-End Flow

1. User calls `deposit()` on the Ethereum source contract. Asset is locked, `MessagePosted` event emitted.
2. Relayer detects the `MessagePosted` event in the finalized block. Constructs receipt Merkle proof.
3. Relayer submits proof to the Supra bridge module. Receipt is verified against the stored `receipts_root`, fee and nonce checked.
4. Wrapped asset minted to the recipient on Supra.
5. Indexer picks up both the source event and destination mint, marks the transaction complete.

End-to-end latency is dominated by Ethereum finality: ~13 minutes for a checkpoint. Post-finality, relay and proof submission on Supra takes seconds.

{% card(type="green") %}
**Trust model:** if you trust Ethereum's proof-of-stake consensus and the BLS12-381 implementation in MoveVM, the bridge is trustless. No external signers, no multisig, no oracle. The relayer and committee updater are liveness dependencies - not security dependencies.
{% end %}
