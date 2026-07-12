+++
title = "Bridges Part 2: Intent-Based Bridges and Atomic Swaps"
date = 2026-04-23
description = "How intent-based bridges work, using Across Protocol as the baseline and Eco Protocol's Routes system as a contrasting design, then a comparison with the older HTLC atomic swap primitive."
[taxonomies]
tags = ["blockchain", "bridges", "interoperability", "cross-chain", "intents", "atomic-swaps", "htlc", "defi"]
[extra]
toc = true
+++

In [Part 1](@/blockchain/bridges-intro.md) we built a trustless bridge using Ethereum sync committees. The user waits ~13 minutes for finality. The destination chain verifies a chain of cryptographic proofs. Security reduces entirely to Ethereum consensus.

Intent-based bridges make a different trade. No proofs at fill time. Settlement in seconds. Security backed by economics or by a delegated cryptographic proof, rather than by the destination chain verifying the source chain's own consensus. For most retail users bridging everyday amounts, this trade is worth it.

This post explains the intent-based model in two passes. First, how [Across Protocol](https://docs.across.to/) works: it is the dominant intent-based bridge by volume and the most mature design in the space, which makes it the right baseline. Then, how [Eco Protocol's Routes system](https://docs.eco.com/) takes the same ERC-7683 intent standard and settles it differently, using cryptographic messages carried by a cross-chain bridge instead of Across's economically bonded assertions. The order matters: the second design mostly makes sense as a reaction to the first, so it is worth understanding Across before contrasting anything against it.

## The Core Idea

Traditional bridges move assets mechanically: lock on source, prove on destination, mint. The user is coupled to the bridge mechanism. They wait for finality, pay for proof verification, and the latency is dictated by the source chain's consensus timeline.

Intent-based bridges decouple the user's desired outcome from the mechanism that delivers it.

The user says: "I want X amount of token A on chain B. I will give Y amount of token B on chain A." This is the intent. The user does not care how it is filled, only that it is filled correctly, quickly, and at a fair price.

A network of professional fillers, called relayers by Across and solvers by Eco, watches for intents and competes to fill them. The filler sends the output tokens directly to the user on the destination chain from their own pre-positioned balance. The user gets their tokens in seconds. The filler claims reimbursement from the funds locked on the source chain afterward, once some settlement mechanism confirms the fill happened.

> The user experience is a fast transfer. Under the hood, a filler is fronting capital and assuming the settlement risk in exchange for a fee.

The two designs in this post differ almost entirely in that last step: how the filler proves the fill and gets paid. Across uses an economic mechanism (a bond that goes unchallenged). Eco Routes uses a cryptographic one (a message delivered by a cross-chain bridge). Everything else, the intent, the racing fillers, the pre-positioned inventory, is common to both.

## How Across Works

Across is the baseline. It is worth reading this section in full even if Eco Routes is what you came for, because every contrast later is a contrast against the machinery described here.

### Anatomy of a Deposit

Across V3 uses the ERC-7683 cross-chain intents standard as its base format. A deposit is a signed data structure with precise fields:

```solidity
struct V3RelayData {
    address depositor;         // who locked funds on source chain
    address recipient;         // who receives on destination chain
    address exclusiveRelayer;  // relayer with exclusive fill rights (initial window)
    address inputToken;        // token depositor is giving
    address outputToken;       // token recipient wants
    uint256 inputAmount;       // amount depositor locks
    uint256 outputAmount;      // amount recipient receives (already fee-adjusted)
    uint256 originChainId;
    uint256 destinationChainId;
    uint32  fillDeadline;      // latest block on destination to fill
    uint32  exclusivityDeadline; // when exclusive window ends
    bytes   message;           // optional calldata to execute on destination
}
```

Key fields:

- **outputAmount**: the exact amount the recipient will receive. This is already adjusted for the bridge fee at deposit time. What the relayer earns is `inputAmount - outputAmount - protocol fee`.
- **exclusiveRelayer**: optionally, a specific relayer gets exclusive fill rights for a short window. After `exclusivityDeadline`, any relayer can fill.
- **fillDeadline**: after this block on the destination chain, the fill is invalid and the user can reclaim their locked funds.
- **message**: optional calldata. Across supports atomic cross-chain execution, the recipient address can be a contract and `message` is executed on fill. This enables cross-chain swaps, deposits into lending protocols, and other composed actions.

The user calls `SpokePool.deposit()` on the source chain. Input tokens are locked. An event is emitted. From that point, relayers race to fill it.

### Relayers

Relayers are off-chain agents that monitor Across spoke contracts and fill deposits profitably. Running a relayer requires:

- **Monitoring infrastructure**: watching all supported spoke chains for new deposits in real time. Across currently supports Ethereum, Arbitrum, Optimism, Base, Polygon, zkSync, Linea, and others.
- **Pre-positioned liquidity**: tokens ready on every destination chain the relayer wants to serve. A relayer cannot fill an ETH->Arbitrum deposit if they have no tokens on Arbitrum.
- **Execution speed**: in a competitive environment with exclusive windows, being the designated relayer matters. After the exclusivity window, the first valid fill wins.
- **Risk management**: relayers assess whether the fill is profitable before committing capital.

A relayer's decision to fill comes down to one calculation:

```text
profit = reimbursement - output_sent - gas_costs - capital_cost - settlement_risk
```

Where:
- `reimbursement` = inputAmount locked by the user (minus protocol fee), paid by the hub after settlement
- `output_sent` = outputAmount sent to the recipient on the destination chain
- `gas_costs` = gas on both source and destination chains
- `capital_cost` = cost of locking up output tokens during the settlement window
- `settlement_risk` = probability the relayer is not reimbursed (dispute, bug, fraud)

If this is positive, the relayer fills. If not, the deposit goes unfilled until a relayer finds it profitable or the deadline expires.

### The UBA Fee Model

Across uses a **Unified Bridge Architecture (UBA)** fee model. Fees are not fixed, they are dynamic based on the liquidity utilization of the hub pool.

The hub pool on Ethereum holds LP deposits. Relayers draw reimbursements from this pool. The pool tracks a running utilization curve: as more capital is tied up in in-flight fills awaiting settlement, utilization rises and fees increase. As relayers are reimbursed and capital returns, utilization falls and fees decrease.

This has two effects:

1. **Flow balancing**: high fees on heavily used routes discourage new deposits on those routes and attract deposits in the reverse direction (which restores balance).
2. **Relayer incentive alignment**: relayers earn higher fees precisely when they are providing scarce liquidity, which incentivizes capital deployment to constrained routes.

The fee a depositor pays is quoted at deposit time based on current utilization. The outputAmount in the deposit reflects this fee already baked in.

### Exclusive Relayer Windows

Across uses an exclusivity mechanism to reduce failed transaction spam. When a user gets a quote from Across, the quote names a specific relayer as the exclusive filler for a short initial window (typically a few seconds to a minute).

The exclusive relayer is typically the relayer that provided the best quote off-chain. They have a brief window to fill before the deposit opens to all relayers. If the exclusive relayer misses the window, any relayer can fill.

This reduces the gas wasted on failed fill attempts in competitive racing scenarios while still ensuring competition as a fallback. The exclusive relayer commits to a fill price at quote time and is held to it, they cannot fill at a worse rate than quoted.

### The Fill and Settlement Flow

```text
Source Chain          Off-Chain             Destination Chain
     |                    |                       |
     |<-- deposit() ------User                    |
     |   lock inputToken  |                       |
     |   emit Deposit     |                       |
     |                    |                       |
     |           Relayer sees deposit             |
     |           checks profitability             |
     |           (within exclusive window         |
     |            or after it opens)              |
     |                    |                       |
     |                    |-- fillRelay() ------->|
     |                    |   send outputToken    |
     |                    |   to recipient        |
     |                    |   emit FilledRelay    |
     |                    |                       |
     |<-- claim to hub ---Relayer                 |
     |   (submit fill proof)                      |
     |                    |                       |
     |-- reimburse ------>Relayer                 |
     |   (from hub pool)                          |
```

**Step 1 - Deposit.** User calls `SpokePool.deposit()` on the source chain. Input tokens are locked in the spoke. A `FundsDeposited` event is emitted with the full deposit data.

**Step 2 - Fill.** A relayer watching the source spoke picks up the deposit. If it is within the exclusive window, only the designated relayer can fill. After the window, any relayer can. The relayer calls `SpokePool.fillRelay()` on the destination chain, sending the outputAmount to the recipient. A `FilledRelay` event is emitted.

**Step 3 - Settlement.** The relayer needs to prove the fill happened and claim reimbursement from the hub on Ethereum mainnet. This is where Across's optimistic oracle comes in.

### Optimistic Settlement via UMA

Across uses UMA's **Optimistic Oracle V3** for settlement. This is the core mechanism that makes fast fills safe without on-chain proof verification.

After filling on the destination chain, the relayer submits a reimbursement claim to the Across hub contract on Ethereum. The claim contains a data assertion: "I, relayer address X, filled deposit ID Y on destination chain Z for outputAmount W." The relayer posts a bond with this assertion.

A challenge window opens (currently 2 hours on Ethereum mainnet).

During the window, anyone can dispute the claim by also posting a bond. If a dispute is raised, UMA's oracle resolves it: both sides post bonds, UMA token holders vote, the loser's bond is slashed and given to the winner.

If no dispute is raised within the window, the assertion is assumed valid and the relayer is reimbursed from the hub pool.

```text
Relayer submits claim + bond
         |
         v
Challenge window (2 hours)
         |
    No dispute? --> Relayer reimbursed from hub pool
         |
    Dispute raised? --> UMA token holders vote
                              |
                    Relayer correct? --> Relayer reimbursed + disputer bond slashed
                    Disputer correct? --> Claim rejected + relayer bond slashed
```

The security guarantee: as long as at least one honest party is watching and willing to dispute invalid claims, the system is secure. Across runs a network of bots (the **watcher network**) that monitor all fill events on destination chains and automatically dispute claims that do not match verified fills.

**Speed vs security trade-off.** The challenge window adds latency for the relayer (they wait 2 hours before full reimbursement), but the user gets their tokens immediately when the fill happens. The relayer absorbs the settlement latency in exchange for the fill fee.

This is the single most important thing to hold onto before reading about Eco Routes: **Across's settlement is a bond that is assumed correct if nobody disputes it within a fixed window.** No message crosses between chains to prove the fill. Correctness is enforced by economic incentive, the threat of a slashed bond, not by cryptography.

### Hub and Spoke Architecture

Across uses a hub-and-spoke model. A hub contract (`HubPool`) sits on Ethereum mainnet. Spoke contracts (`SpokePool`) sit on each supported chain.

When a user bridges from chain A to chain B:
1. User locks tokens in the SpokePool on chain A.
2. Relayer fills from their balance on chain B (SpokePool on B records the fill).
3. Relayer submits assertion to HubPool on Ethereum.
4. After the 2-hour challenge window, HubPool reimburses the relayer from the LP pool.

The hub holds a pool of liquidity that backs all reimbursements. LPs deposit into the hub pool and earn fees from bridge volume. The UBA fee model determines how fees are distributed between the protocol, LPs, and relayers.

**LP risk.** LPs earn fees but carry the risk of the optimistic oracle failing (a fraudulent claim passing unchallenged). In practice, the watcher network and bond economics make this extremely unlikely, a successful fraud requires the attacker to post a bond, wait 2 hours without being disputed by a single watcher, and extract an amount that must exceed the bond cost. The economic attack surface is narrow.

**Canonical tokens.** Across does not mint wrapped tokens. It uses canonical representations, USDC on Optimism is native USDC bridged via the Optimism canonical bridge. Across only bridges assets that have canonical liquidity on both ends. This eliminates the risk of synthetic tokens depegging.

That is Across in full: a signed deposit, racing relayers, a shared LP-funded hub pool, dynamic UBA fees, and optimistic bond-based settlement over a fixed 2-hour window. Everything below is a comparison against this.

## How Eco Routes Differs

Eco Routes implements the same ERC-7683 standard, but three design choices diverge from Across:

1. **The intent is a general call bundle, not a token transfer.** Across's deposit is fundamentally "send outputAmount of outputToken to recipient." Eco's intent is "execute this array of arbitrary calls," of which a token transfer is just the simplest case.
2. **Settlement is a cryptographic message, not an unchallenged bond.** Instead of Across's 2-hour optimistic window, a proof of fulfillment must physically arrive at the source chain via a cross-chain messaging bridge before the reward is claimable.
3. **Reward escrow is per-intent, not a shared pool.** Across pools LP capital into one HubPool. Eco deploys a fresh escrow vault for every single intent.

The rest of this section walks through each.

### Anatomy of an Intent

Where Across has one flat `V3RelayData` struct, Eco nests three:

```solidity
struct Intent {
    uint64 destination;  // target chain ID
    Route  route;         // what to execute, and where
    Reward reward;        // what the solver gets paid, and by when
}

struct Route {
    bytes32 salt;            // creator-supplied uniqueness / anti-replay
    uint64  deadline;        // latest timestamp the route can be executed
    address portal;          // Portal contract address on the destination chain
    uint256 nativeAmount;    // native token needed to execute the calls
    TokenAmount[] tokens;    // ERC20 tokens the solver must supply for the calls
    Call[] calls;            // arbitrary calls to execute on the destination chain
}

struct Reward {
    uint64  deadline;        // latest timestamp the reward can be claimed
    address creator;         // who funded the reward, can reclaim after expiry
    address prover;          // which prover contract must attest fulfillment
    uint256 nativeAmount;    // native token reward
    TokenAmount[] tokens;    // ERC20 token rewards
}

struct Call {
    address target;
    bytes   data;
    uint256 value;
}
```

The differences from Across's struct map directly onto the three design choices:

- **calls, not just outputToken.** `route.calls` is an arbitrary array of contract calls. Across bakes composition into an optional `message` field on top of a token transfer; Eco makes the call bundle the primitive, and a plain transfer is just a one-call route.
- **prover is chosen per intent.** The creator picks which prover contract must attest fulfillment before the reward unlocks. Across has no equivalent field because its settlement mechanism is fixed protocol-wide (always UMA). In Eco, the choice of prover *is* the choice of security model.
- **two independent deadlines.** Across has one `fillDeadline`. Eco splits it: `route.deadline` gates when a solver can still execute, and `reward.deadline` gates when the creator can reclaim an unfulfilled reward.

The intent's hash is computed on-chain as `keccak256(abi.encodePacked(chainId, keccak256(abi.encode(route)), keccak256(abi.encode(reward))))`. This binds the intent to a specific destination chain and Portal address, so a solver cannot fulfill it against the wrong contract.

Unlike Across, where the deposit call both locks funds and opens the order, Eco separates publishing from funding. The creator calls `publish(intent)`, which only emits an `IntentPublished` event. Funding happens separately via `fund()`, `publishAndFund()`, or a permit-based `fundFor()`, and deploys a fresh CREATE2-cloned `Vault` at a deterministic per-intent address holding the reward. Solvers check `isIntentFunded()` before committing, an unfunded intent is just data with nothing behind it.

### Solvers

Eco's solvers are the same role as Across's relayers, off-chain agents fronting pre-positioned inventory to fill intents. The monitoring, liquidity, and risk-management requirements carry over unchanged. Two things are new relative to Across:

- **Prover awareness.** A solver must check which prover each intent specifies and decide whether that prover's route back to the source chain is one they trust and can get proven on in acceptable time. Across relayers never make this choice, settlement is always UMA.
- **A different profit equation.** The message-bridge fee replaces the settlement bond as a cost line:

```text
profit = reward_value - route_execution_cost - gas_costs - capital_cost - proving_risk
```

Where `gas_costs` now includes the message-bridge fee charged at fulfillment time (Hyperlane and LayerZero both charge to send the proof), and `proving_risk` is the probability the chosen prover never delivers a valid attestation, an outcome with no clean analogue in Across's design.

### Solver Access: Whitelist Instead of Exclusivity

Across manages fill competition with a per-deposit exclusivity window. Eco solves a different problem, who is allowed to solve at all, with a deployment-level whitelist.

A Portal deployment can restrict `fulfill()` to approved solver addresses via `changeSolverWhitelist()`. The owner can later call `makeSolvingPublic()` to open it to anyone. This is a one-way switch: once public, it cannot be re-restricted.

This is coarser than Across's per-deposit exclusivity. It does not reduce gas wasted on fill races for any individual intent the way a short exclusive window does. What it buys instead is the ability to launch with vetted solvers only, then graduate to a permissionless set once the deployment has a track record. It is a bootstrapping lever, not a per-transaction quoting mechanism.

### The Fulfillment and Proving Flow

```text
Source Chain              Off-Chain              Destination Chain
     |                        |                        |
     |<-- publish() ---------Creator                   |
     |   emit IntentPublished |                        |
     |<-- fund() -------------Creator                  |
     |   deploy Vault, lock   |                        |
     |   reward, emit         |                        |
     |   IntentFunded         |                        |
     |                        |                        |
     |              Solver sees intent, checks         |
     |              isIntentFunded(), checks            |
     |              whitelist + prover trust            |
     |                        |                        |
     |                        |-- fulfill() ---------->|
     |                        |   execute route.calls  |
     |                        |   set claimant         |
     |                        |   emit IntentFulfilled |
     |                        |                        |
     |<-- prove() message ----------------------------|
     |   via chosen prover's message bridge             |
     |   (Hyperlane / LayerZero / Metalayer / storage) |
     |                        |                        |
     |-- withdrawRewards() -->Solver (or anyone,       |
     |   Vault pays claimant  paid to claimant)        |
```

**Step 1 - Publish.** Creator calls `publish(intent)`. `IntentPublished` is emitted. No funds move (contrast Across, where the deposit call locks funds immediately).

**Step 2 - Fund.** Creator or a third party funds the intent. A per-intent `Vault` is deployed via CREATE2 and funded. `IntentFunded` is emitted.

**Step 3 - Fulfill.** A solver calls `fulfill()` or `fulfillAndProve()` on the destination Portal, passing the route, the reward hash, and a `claimant` address to be paid on the source chain. The Portal recomputes the intent hash and reverts on mismatch, on wrong portal, or on a double-fill. An internal `Executor` pulls the required `route.tokens` from the solver and runs `route.calls`. `IntentFulfilled` is emitted. This is the analogue of Across's `fillRelay()`.

**Step 4 - Prove.** `fulfillAndProve()` calls `prove()`, packaging the intent hash and claimant and sending them through the reward's specified prover to the source chain. `fulfill()` alone skips this, proving can be batched via `sendBatch()` to amortize message-bridge fees across many intents. Across has no equivalent step, its "proof" is the unchallenged passage of time.

**Step 5 - Withdraw.** Once the source-chain prover records the intent as proven, anyone can call `withdrawRewards()`, but the Vault only ever pays the claimant the solver set in step 3.

### Cryptographic Settlement via Message Bridges

This is the core divergence from Across. Across posts a bonded assertion and waits out a challenge window: absence of dispute is treated as proof. Eco Routes instead requires a message to physically arrive at the source chain before a reward is claimable. There is no proof-by-silence, there is a delivered message or there is not.

The `reward.prover` field chooses which message-passing system carries that proof. The `eco-routes` contracts implement four:

- **HyperProver** - uses Hyperlane's Mailbox and Interchain Security Modules.
- **LayerZeroProver** - uses LayerZero's Endpoint and configured Decentralized Verifier Networks (DVNs).
- **MetaProver** - uses Caldera Metalayer's Router.
- **StorageProver** - proves fulfillment via storage proofs relayed by a system such as Polymer, closer in spirit to the light-client verification from [Part 1](@/blockchain/bridges-intro.md) than to a bonded assertion.

Each prover has its own trust assumptions and its own latency, set by the underlying bridge, not by Eco Routes. A HyperProver route is only as secure as Hyperlane's ISM for that chain pair; a LayerZeroProver route only as secure as its DVNs. There is no protocol-wide constant analogous to Across's fixed 2-hour UMA window, proof latency and trust level are properties of whichever prover the creator picked.

```text
Solver calls fulfillAndProve()
         |
         v
Message sent via chosen prover's bridge
(Hyperlane Mailbox / LayerZero Endpoint / Metalayer Router / storage proof relay)
         |
         v
Message verified by that bridge's own security (validator set / DVNs / light client)
         |
         v
Source-chain prover contract marks intent as proven for claimant
         |
         v
withdrawRewards() succeeds, Vault pays claimant
```

**The trade-off, restated against Across.** Across fixes the trade-off protocol-wide: always a 2-hour window, always bonds and watchers. Eco pushes the trade-off down to the intent. A creator who trusts Hyperlane or LayerZero can get proofs in minutes; a creator who wants light-client guarantees can specify a StorageProver and accept its latency. The cost is that the solver must independently evaluate the prover on every intent, rather than trusting one protocol-wide guarantee the way an Across relayer trusts UMA.

### Per-Intent Vaults, Not a Shared Pool

Across pools LP capital into one `HubPool`, and every relayer draws reimbursement from it. Eco has no hub. Every intent gets its own `Vault`, deployed fresh via CREATE2, holding exactly that intent's reward and nothing else.

This cuts both ways relative to Across:

**No LP risk, but no LP yield either.** There is no shared pool sitting behind every intent, so a compromised or misconfigured prover cannot drain third-party capital in bulk, the blast radius of any one bad intent is bounded by that intent's own reward. But there is also no passive LP role, and no LP fees. Reward capital comes directly from whoever creates the intent, not from a standing pool. Across's LPs earn yield and carry oracle-failure risk; Eco has neither party.

**No protocol-wide dynamic fee curve.** Across's UBA model adjusts fees by hub utilization. Eco has nothing to adjust, the reward is whatever the creator funds the Vault with. Competitive pricing is left to whatever quoting layer sits above the raw contracts, not enforced on-chain by a utilization curve.

## Rebalancing and Capital Efficiency

These problems apply to both designs, because both rely on fillers fronting capital on the destination chain and getting reimbursed on the source chain.

**The rebalancing problem.** A filler running many transfers from Ethereum to Arbitrum ends up with capital accumulating on Ethereum (from reimbursements) and draining on Arbitrum (from fills). They must continuously withdraw, bridge inventory back, pay gas and bridge fees, and time it to minimize idle capital. This is the **inventory imbalance problem**, and it is inherent to the model, not specific to hub-and-spoke.

The one difference: Across prices rebalancing cost into fees automatically through the UBA curve on high-volume directional routes. Eco has no such curve, so a solver must price rebalancing cost into their own per-intent decisions manually.

Either way, deep-capital fillers dominate: they buffer larger imbalances before needing to rebalance, so they pay proportionally less. This creates the same centralization pressure toward large, well-capitalized fillers in both systems.

**Capital cost per fill.** A filler ties up capital between fill and reimbursement:

```text
capital_cost = fill_amount * rate_of_return * settlement_duration
```

For Across, `settlement_duration` is the fixed 2-hour UMA window. A 10,000 USDC fill at a 5% annual opportunity cost:

```text
capital_cost = 10,000 * (0.05 / (365 * 24)) * 2  ~= $0.11
```

For Eco, `settlement_duration` is not fixed, it is whatever the chosen prover takes. Hyperlane or LayerZero routes commonly finalize in roughly 30 seconds to a few minutes, depending on the ISM or DVN configuration and source-chain finality (an optimistic ISM, by contrast, adds its own challenge window and behaves more like Across):

```text
capital_cost = 10,000 * (0.05 / (365 * 24 * 60)) * 10  ~= $0.01   (10-minute proving)
```

roughly 12x cheaper per fill than Across's window (just the ratio of the two durations, 120 minutes vs 10), when the prover cooperates. The catch is that this number has no protocol-guaranteed ceiling: a StorageProver route, or a degraded message bridge, can push Eco's latency well past Across's fixed 2 hours, whereas Across's 2-hour worst case is a constant.

**Minimum viable fill size.** Both have a floor where `gas_costs > filler_margin` makes small transfers unprofitable. Eco's floor is slightly higher on this axis: the message-bridge fee (Hyperlane, LayerZero) is an added per-fill cost that Across's optimistic model does not pay at fill time. Across's floor is driven mainly by L1 hub settlement gas, which it amortizes by routing most volume through cheap L2 spokes.

## Liveness Failure

Both designs can fail to deliver, and in both the user's funds are recoverable.

On **Across**, if no relayer fills before `fillDeadline`, the deposit expires and the user reclaims their locked input tokens from the SpokePool. Nothing is lost but gas and time.

On **Eco**, if no solver fulfills before `route.deadline`, or it is fulfilled but never proven before `reward.deadline`, the creator calls `refund()` to reclaim the reward from the Vault.

The failure triggers overlap: no profitable filler, all fillers offline or under-capitalized, deadline too short. Eco adds two of its own:

- Solving may be whitelisted with no whitelisted solver willing to act.
- The chosen prover may fail to deliver a message before `reward.deadline` even though the route executed correctly. This leaves the solver unpaid on a genuinely completed fill, a failure mode Across does not have, because Across reimbursement never depends on a third-party message bridge delivering. An Across relayer who fills correctly is always reimbursed (absent fraud); an Eco solver who fulfills correctly against a bad prover can eat the cost.

That last case is the sharpest liveness cost of the message-bridge model: a solver can do everything right and still not get paid if the creator picked an unreliable prover.

## Security Models Compared

Across is **economic**: a bond that goes unchallenged is treated as true, backed by a watcher network incentivized to dispute fraud. Eco is **cryptographic and delegated**: trust is handed wholesale to whichever message bridge the intent's `reward.prover` names.

**What can go wrong on Across:**
1. Relayer claims reimbursement without filling. Caught by watchers, bond slashed.
2. Relayer claims a wrong amount. Caught by watchers comparing claim to the on-chain fill.
3. All watchers offline for the full 2-hour window. A fraudulent claim passes and the hub pool is drained. The catastrophic case, mitigated by running multiple independent open-source watchers.
4. Contract bug in SpokePool or HubPool.

**What can go wrong on Eco:**
1. The creator-chosen prover is misconfigured or malicious. A solver can fulfill correctly and still never get a valid proof delivered, forfeiting the fill cost.
2. The underlying message bridge is compromised (Hyperlane ISM or LayerZero DVNs colluding), delivering a false proof and paying a claimant who never executed the route.
3. Message-bridge outage or extreme latency. No fixed window means no fixed worst case, capital sits fulfilled-but-unpaid for however long the bridge is degraded.
4. Contract bug in Portal, Vault, or a specific prover.

**What cannot go wrong on either (given honest infrastructure):**
- A filler cannot steal the user's destination tokens, once the fill executes, the recipient has them.
- A filler cannot block the user from reclaiming after the deadline.
- The same order cannot be filled twice, both contracts revert on a second fill.

The shape of the catastrophic case differs. Breaking Across means getting a fraudulent bond past every watcher during a fixed 2-hour window, one protocol-wide target, one protocol-wide guarantee. Breaking an Eco intent means compromising the specific bridge that intent's creator chose, a narrower, more concentrated per-intent target, but one whose strength varies intent by intent rather than being a single constant. And because Eco's escrow is per-intent, a single compromised prover drains one Vault, not a shared pool.

**A note on licensing.** Neither project's on-chain contracts are open source in the OSI sense, worth flagging since auditability underpins both security arguments. Across's contracts are BUSL-1.1 (source-available, with an eventual conversion to an open license contemplated). Eco's `eco-routes` is released under the Eco Source-Available License 1.0: source-visible and modifiable for non-production use only, production deployment requires a separate agreement with Eco Foundation, and the license has no scheduled conversion date. Individual `.sol` files carry `SPDX-License-Identifier: MIT` headers, but the repository-level LICENSE governs and explicitly states it is not open source. The one genuinely open component is Across's off-chain relayer and watcher bot (`across-protocol/relayer`), which is AGPL-3.0, exactly the software the watcher-network security argument leans on.

## Atomic Swaps: The Older Trustless Primitive

Both designs above still lean on off-chain infrastructure, relayers, solvers, provers, watchers, to make the user's experience fast. There is an older approach to trustless cross-chain exchange that needs none of it: the atomic swap, secured purely by a shared hash preimage checked locally on each chain. It predates intent-based bridges by close to a decade and is worth understanding both on its own terms and as the contrast case for why intent-based systems won on UX.

An atomic swap lets two parties exchange assets on different blockchains with no intermediary. Either both legs complete or neither does. The mechanism is a pair of Hash Time-Locked Contracts (HTLCs), one on each chain. The same preimage unlocks both, making the swap atomic: the party who reveals the preimage to claim their funds simultaneously allows the counterparty to claim theirs.

### The HTLC Primitive

A Hash Time-Locked Contract has two spending conditions:

1. **Hashlock:** provide the preimage `s` such that `sha256(s) == h`. This is the claim path.
2. **Timelock:** after block height `T`, the original funder can reclaim the funds. This is the refund path.

The hashlock forces the claimant to reveal the preimage on-chain. The timelock ensures funds are not locked indefinitely if the swap fails.

In Bitcoin Script:

```text
OP_IF
    OP_SHA256 <hash> OP_EQUALVERIFY
    <recipient_pubkey> OP_CHECKSIG
OP_ELSE
    <timelock_height> OP_CHECKLOCKTIMEVERIFY OP_DROP
    <sender_pubkey> OP_CHECKSIG
OP_ENDIF
```

To claim: push `<preimage> 1` onto the stack. The `OP_IF` branch executes, verifies the preimage against `<hash>`, then checks the recipient's signature.

To refund: push `<sender_sig> 0`. The `OP_ELSE` branch executes, checks the timelock has passed, then checks the sender's signature.

In Solidity:

```solidity
struct HTLC {
    address payable sender;
    address payable recipient;
    uint256 amount;
    bytes32 hashlock;   // sha256(preimage)
    uint256 timelock;   // block.timestamp deadline
    bool withdrawn;
    bool refunded;
}

function withdraw(bytes32 htlcId, bytes32 preimage) external {
    HTLC storage htlc = contracts[htlcId];
    require(!htlc.withdrawn && !htlc.refunded);
    require(sha256(abi.encodePacked(preimage)) == htlc.hashlock);
    require(block.timestamp < htlc.timelock);
    require(msg.sender == htlc.recipient);
    htlc.withdrawn = true;
    htlc.recipient.transfer(htlc.amount);
}

function refund(bytes32 htlcId) external {
    HTLC storage htlc = contracts[htlcId];
    require(!htlc.withdrawn && !htlc.refunded);
    require(block.timestamp >= htlc.timelock);
    require(msg.sender == htlc.sender);
    htlc.refunded = true;
    htlc.sender.transfer(htlc.amount);
}
```

The hashlock and timelock together define the security model: the preimage is secret until revealed, and once revealed on-chain it is public and anyone can observe it.

### The 2-Party Protocol

Say Alice wants to swap 1 BTC for 10 ETH with Bob. Alice controls chain A (Bitcoin), Bob controls chain B (Ethereum).

**Setup:** Alice generates a random secret `s` and computes `h = sha256(s)`. She keeps `s` private for now.

```text
Alice                               Bob
  |                                  |
  |-- generates s, h = sha256(s)     |
  |                                  |
  |   HTLC_A on Bitcoin              |
  |   hashlock=h, recipient=Bob      |
  |   timelock=T1 (+48h)             |
  |-- funds with 1 BTC ------------->|
  |                                  |
  |              HTLC_B on Ethereum  |
  |              hashlock=h (same)   |
  |              recipient=Alice     |
  |              timelock=T2 (+24h)  |
  |<-- funds with 10 ETH ------------|
  |                                  |
  |   Alice calls withdraw(s)        |
  |-- reveals preimage on ETH ------>|
  |   receives 10 ETH                |
  |                                  |
  |   Bob reads s from ETH calldata  |
  |   Bob calls Bitcoin HTLC with s  |
  |<-- receives 1 BTC ---------------|
```

**Step 1.** Alice creates HTLC_A on Bitcoin with `h`, Bob as recipient, timelock `T1` (48 hours from now). She funds it with 1 BTC.

**Step 2.** Bob sees HTLC_A on Bitcoin, reads `h`. Bob creates HTLC_B on Ethereum with the same `h`, Alice as recipient, timelock `T2` (24 hours from now, so `T2 < T1`). He funds it with 10 ETH.

**Step 3.** Alice calls `withdraw(s)` on HTLC_B. She receives 10 ETH. The preimage `s` is now public in the Ethereum transaction calldata.

**Step 4.** Bob reads `s` from the Ethereum chain and calls the Bitcoin HTLC with `s`. He receives 1 BTC.

The swap is complete. Neither party trusted the other at any point.

### Why T2 < T1 Matters

The timelock ordering is not arbitrary. `T2 < T1` defines who has the advantage if things go wrong.

If the gap between the two timelocks is too small, Alice can reveal `s` on Ethereum right before `T2` expires, and Bob may not have enough time to observe the reveal, extract `s`, and get a Bitcoin transaction confirmed before `T1`.

By setting `T2 < T1` with a sufficient gap, we ensure: after Alice reveals `s` on Ethereum, Bob has `T1 - T2` time to claim his BTC. The gap must account for:

- Bitcoin block time variance (average 10 minutes, but often 30+ minutes during low hashrate periods)
- Desired confirmation depth (typically 6 confirmations = ~60 minutes)
- Bob's monitoring latency and transaction submission time

A conservative gap for BTC/ETH: 24 hours between `T2` and `T1`. This is a long time to have capital locked.

### Failure Modes

**Alice never reveals `s`:**
Bob's HTLC expires at `T2`. Bob refunds his ETH. Alice's BTC is stuck until `T1`, then she refunds. No funds are lost. The swap does not happen. Cost: gas on both chains, and opportunity cost of locked capital for up to 48 hours.

**Bob never creates his HTLC:**
Alice's HTLC expires at `T1`. Alice refunds her BTC. No loss beyond the capital locked during `T1`.

**Network congestion race:**
If Alice reveals `s` near the end of the safe window, Bob might not get his Bitcoin claim confirmed before `T1`. This is a known failure mode on Bitcoin due to its high block time variance. The solution is conservative timelock gaps, which worsens capital efficiency.

**The free option problem:**
Alice holds a free option during the entire lockup window. If BTC price rises significantly relative to ETH between the time Alice locks and when `T1` expires, Alice may rationally prefer to abort the swap (let `T1` expire, reclaim BTC) rather than proceed. This makes long-lived HTLCs unattractive for counterparties on volatile assets: the initiating party can walk away at zero cost beyond gas fees.

### Cross-Chain Implementation Challenges

**Hash function compatibility:** Bitcoin uses SHA256 natively. Ethereum's default is keccak256, but SHA256 is available as a precompile at address `0x02`. Both sides of the swap must agree on SHA256. This is a convention, not automatic.

**Reading cross-chain state:** Bob needs to verify Alice's Bitcoin HTLC before committing his ETH. There is no native way to read Bitcoin state from Ethereum. Bob either runs a Bitcoin full node, trusts an SPV proof, or relies on an off-chain API. This is the same light-client problem as [trustless bridges](@/blockchain/bridges-intro.md), and it has no cheap solution.

**Block time vs wall clock:** Bitcoin timelocks use block height (`OP_CHECKLOCKTIMEVERIFY`). Ethereum uses `block.timestamp`. Converting between them requires estimating future block times, which varies. A timelock set to "6000 Bitcoin blocks from now" might expire in 40 hours or 80 hours depending on network hashrate.

**Address management:** Alice needs a Bitcoin address and an Ethereum address. So does Bob. Key management across multiple chains adds UX friction. Hardware wallets and most wallet software have limited cross-chain HTLC tooling.

### Practical Applications

**Lightning Network.** HTLCs are the core payment routing primitive in Lightning. When Alice routes a payment to Carol via Bob:

```text
Alice --[HTLC_1]--> Bob --[HTLC_2]--> Carol
     locks H             locks H
```

Carol generates the preimage and reveals it to Bob to claim HTLC_2. Bob uses it to claim HTLC_1 from Alice. This is a multi-hop HTLC chain within a single blockchain, not cross-chain, but the same locking mechanism. Lightning is the dominant production use case for HTLCs at scale, processing millions of payments.

**Cross-chain DEXes.** Komodo's AtomicDEX and Bisq use HTLCs for BTC/altcoin OTC-style swaps. They work correctly. Volume is a small fraction of AMM or intent-based bridges because of all the limitations above: long locktimes, both parties must be online, no liquidity pooling, free option problem.

**Privacy-preserving OTC.** For large bilateral trades where both parties want to avoid centralized custody, atomic swaps remain a viable option. A sovereign wealth fund swapping BTC for ETH with a bank might prefer an HTLC to a custodial bridge.

Atomic swaps solve trustless exchange between two willing, online counterparties with matched amounts. They do not solve liquidity aggregation. A user wanting to swap 0.5 BTC for ETH needs a counterparty willing to take the exact other side at an agreed rate. In practice, finding that counterparty requires a matching layer (an order book or RFQ system) that reintroduces centralized coordination.

Intent-based bridges sidestep this entirely: professional fillers hold pre-positioned inventory and fill any size in seconds, absorbing the inventory and settlement risk in exchange for a fee.

### The Hybrid: 1inch Fusion+

The two approaches are not strictly either/or. **1inch Fusion+** fuses them: it keeps the HTLC as the settlement rail but wraps it in an intent/solver front end.

The user signs a Fusion order (an intent) and does nothing else. A network of professional **resolvers** (1inch's word for solvers) competes to fill it through a **Dutch auction**, the price ratchets until a resolver takes the order. Underneath, settlement is a hashlock+timelock escrow on each chain: the resolver locks funds behind the same secret preimage, and the all-or-nothing HTLC guarantee still holds, with everything refunding if the timelock expires.

The division of labor is clean:
- The **HTLC** provides trustless settlement, no custodian, atomic, refund-on-failure, without a bond (Across) or a delegated message bridge (Eco).
- The **solver + Dutch-auction layer** provides the UX and liquidity: the user signs once, needs no matching counterparty, and gets filled in seconds.

This directly answers the two things plain atomic swaps could not do: counterparty discovery (a resolver is always willing to take the other side) and the user having to stay online and wait (the resolver absorbs that). The cost is that the resolver inherits the HTLC's capital-lock and timelock-race characteristics on the settlement leg, so its capital efficiency is worse than Across's optimistic window, in exchange for stronger settlement security (no bond assumption, no third-party bridge). It is the neatest illustration that "atomic swap" and "intent-based bridge" name a settlement primitive and a coordination model, not two mutually exclusive products.

### Why Atomic Swaps Did Not Win

The fundamental problems are capital efficiency and liveness.

A 24-48 hour timelock means capital is locked per-trade for up to two days. For a market maker providing cross-chain liquidity, the opportunity cost is enormous. An intent-based filler typically has capital tied up for minutes to a couple of hours, Eco Routes' message-bridge proving or Across's optimistic window. An atomic swap counterparty has it tied up one to two orders of magnitude longer.

Both parties must be online and responsive throughout. If Bob goes offline after Alice locks her BTC but before funding his HTLC, Alice waits up to 48 hours to get her funds back. Across and Eco both handle this passively: the user submits one transaction and a filler network handles the rest.

The free option problem is unsolved without additional infrastructure. Market makers in a cross-chain atomic swap book face adverse selection: the initiator has the option to abort if prices move favorably. This leads to wider spreads or reluctance to participate on the maker side. (1inch Fusion+ mitigates this by putting a resolver, not a peer, on the other side, and by keeping windows short.)

Atomic swaps remain elegant in theory and genuinely useful in specific contexts: Lightning routing, privacy-preserving bilateral OTC trades, the HTLC settlement rail inside Fusion+, and constructs where hash-preimage coordination maps naturally onto a larger protocol. For general-purpose retail bridging, the pure two-party form has been superseded by intent-based systems, whether secured economically like Across, cryptographically via message bridges like Eco Routes, or by HTLC-under-a-solver like Fusion+, that deliver dramatically better UX by moving the proof-waiting period off the user and onto a professional filler.

## Three Approaches Compared

| | Atomic Swap (HTLC) | Trustless Bridge (Light Client) | Intent-Based |
|---|---|---|---|
| Custodian | None | None | None (filler required) |
| Security model | Cryptographic hash preimage, verified locally | Chain consensus | Across: economic bond + watchers. Eco: delegated to chosen message bridge |
| User latency | Hours (timelock gap) | 13+ min | Seconds |
| Liquidity model | Per-counterparty | LP pool | Across: shared hub pool. Eco: solver inventory + per-intent vault |
| Requires counterparty | Yes | No | Filler (permissionless or whitelisted) |
| Capital locked duration | 24-48 hours | N/A for user | Across: ~2h fixed. Eco: minutes to hours, prover-dependent |
| Chain support | Any chain with HTLC | Requires light client per pair | Wherever contracts are deployed |
| Free option exposure | Yes | No | No |
| Works for small amounts | Yes (if counterparty exists) | Yes | Only above fill floor |
| Catastrophic failure | Counterparty griefing (bounded) | Source chain compromise | Across: watcher collusion + fraud. Eco: compromised prover (bounded per intent) |

For high-value bilateral trades where both parties are online and want zero counterparty risk: atomic swap.

For high-value transfers where security matters and latency is acceptable: trustless bridge.

For frequent retail transfers where speed and UX matter and amounts are above the fill floor: intent-based.

Most real-world bridge volume today is intent-based, led by Across, which alone has processed over 30 billion USD in cumulative volume (per DefiLlama, as of mid-2026). Eco Routes and other ERC-7683 implementations are newer and smaller by volume, but demonstrate that the same standard supports settlement models Across does not use, cryptographic message-bridge proving instead of economic bonding. Most high-value institutional transfers still use trustless or multisig bridges. Atomic swaps persist mainly inside Lightning Network routing, HTLC-settled hybrids like Fusion+, and niche OTC desks.

## References

- [Across Protocol V3 Documentation](https://docs.across.to/) - hub-and-spoke architecture, UBA fee model, optimistic settlement, the baseline design in this post
- [UMA Optimistic Oracle V3](https://docs.uma.xyz/) - dispute mechanism used by Across for settlement
- [across-protocol/relayer on GitHub](https://github.com/across-protocol/relayer) - AGPL-3.0 licensed relayer and watcher bot, contrast to Across's BUSL-1.1 contracts
- [DefiLlama: Across TVL, Fees and Revenue](https://defillama.com/protocol/across) - live cumulative bridge volume figures cited above
- [Eco Routes Documentation](https://docs.eco.com/) - intents, solvers, provers, and the Routes architecture
- [eco/eco-routes on GitHub](https://github.com/eco/eco-routes) - Portal, IntentSource, Inbox, Vault, and prover contract source
- [eco-routes LICENSE](https://github.com/eco/eco-routes/blob/main/LICENSE) - Eco Source-Available License 1.0 terms
- [Hyperlane Documentation](https://docs.hyperlane.xyz/) - Mailbox and Interchain Security Modules used by HyperProver
- [LayerZero V2 Documentation](https://docs.layerzero.network/) - Endpoint and DVN model used by LayerZeroProver
- [ERC-7683: Cross Chain Intents Standard](https://eips.ethereum.org/EIPS/eip-7683) - standardised intent format used by both Across and Eco Routes
- [1inch Fusion+ overview](https://help.1inch.com/en/articles/9842591-what-is-1inch-fusion-and-how-does-it-work) - HTLC settlement wrapped in an intent/resolver Dutch auction, the hybrid design
- [Tier Nolan, original atomic swap proposal (2013)](https://bitcointalk.org/index.php?topic=193281.msg2003765#msg2003765) - the BitcoinTalk post introducing cross-chain atomic swaps using HTLCs
- [BIP-199: Hashed Timelock Contracts](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki) - Bitcoin HTLC standard and script format
- [Lightning Network Whitepaper](https://lightning.network/lightning-network-paper.pdf) - HTLC-based payment channels and multi-hop routing
- [Bridges Part 1: Building Trustless Bridges with Sync Committees](@/blockchain/bridges-intro.md)
