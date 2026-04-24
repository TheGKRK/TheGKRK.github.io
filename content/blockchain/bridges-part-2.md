+++
title = "Bridges Part 2: How Intent-Based Bridges Work"
date = 2026-04-23
description = "A deep dive into intent-based bridge architecture: intents, solvers, Dutch auctions, optimistic settlement, capital efficiency, and the economics that make it work."
draft = true
[taxonomies]
tags = ["blockchain", "bridges", "interoperability", "cross-chain", "intents", "defi"]
[extra]
toc = true
+++

In [Part 1](@/blockchain/bridges-intro.md) we built a trustless bridge using Ethereum sync committees. The user waits ~13 minutes for finality. The destination chain verifies a chain of cryptographic proofs. Security reduces entirely to Ethereum consensus.

Intent-based bridges make a different trade. No proofs at fill time. Settlement in seconds. Security backed by economics rather than cryptography. For most retail users bridging everyday amounts, this trade is worth it.

This post goes deep into how that works.

## The Core Idea

Traditional bridges move assets mechanically: lock on source, prove on destination, mint. The user is coupled to the bridge mechanism. They wait for finality, pay for proof verification, and the latency is dictated by the source chain's consensus timeline.

Intent-based bridges decouple the user's desired outcome from the mechanism that delivers it.

The user says: "I want X amount of token A on chain B. I will give Y amount of token B on chain A." This is the intent. The user does not care how it is filled - only that it is filled correctly, quickly, and at a fair price.

A network of solvers watches for intents and competes to fill them. The solver who fills first (or at the best price, depending on the auction mechanism) sends the output tokens directly to the user on the destination chain from their own pre-positioned balance. The user gets their tokens in seconds. The solver claims reimbursement from the locked funds on the source chain afterward.

> The user experience is a fast transfer. Under the hood, a solver is fronting capital and assuming the settlement risk.

## Anatomy of an Intent

An intent is not just "I want X for Y." It is a signed data structure with precise fields. ERC-7683 (the cross-chain intents standard) defines the canonical format:

```
struct CrossChainOrder {
    address originSettler;      // contract on source chain handling the order
    address user;               // who signed the intent
    uint256 nonce;              // replay protection
    uint256 originChainId;      // source chain
    uint32  openDeadline;       // latest block to open the order
    uint32  fillDeadline;       // latest block on destination to fill
    bytes32 orderDataType;      // type hash of the order data
    bytes   orderData;          // encoded input/output token specs
}
```

Inside `orderData`, the protocol specifies:

- **Input tokens**: what the user is giving up on the source chain (token address, amount).
- **Output tokens**: what the user wants on the destination chain (token address, minimum amount, recipient address).
- **Exclusivity**: optionally, which solver has exclusive rights to fill for an initial window.
- **Fill deadline**: after this block on the destination chain, the fill is invalid and the user can reclaim their locked funds.

The user signs this structure off-chain. The signature is submitted to the source chain's settler contract to open the order, locking the input tokens. From that point, solvers can race to fill it.

## Solvers

Solvers are off-chain agents that monitor intent protocols and fill orders profitably. Running a solver requires:

- **Monitoring infrastructure**: watching source chains for new intents in real time.
- **Pre-positioned liquidity**: tokens ready on every destination chain the solver wants to serve. A solver cannot fill an ETH->SUPRA intent if they have no tokens on Supra.
- **Execution speed**: in a competitive environment, the first valid fill wins. Latency matters.
- **Risk management**: solvers assess whether the fill is profitable before committing capital.

A solver's decision to fill comes down to one calculation:

```
profit = reimbursement - output_sent - gas_costs - capital_cost - settlement_risk
```

Where:
- `reimbursement` = input tokens locked by the user on the source chain (minus protocol fee)
- `output_sent` = tokens the solver sends to the user on the destination chain
- `gas_costs` = gas on both source and destination chains
- `capital_cost` = cost of locking up the output tokens during the settlement window
- `settlement_risk` = probability the solver is not reimbursed (dispute, bug, fraud)

If this is positive, the solver fills. If not, the intent goes unfilled until a solver finds it profitable or the deadline expires.

## Solver Competition: Dutch Auctions

The naive approach to solver competition is "first valid fill wins." This works but is inefficient - solvers race to submit transactions and most gas is wasted on failed attempts.

UniswapX uses a **Dutch auction** mechanism instead. The output amount the user receives starts high and decreases over time. Early in the auction window, the solver margin is thin (user gets most of the value). As the auction progresses, the output amount decreases and solver margin increases.

```
output_amount(t) = start_amount - (start_amount - end_amount) * (t / duration)
```

Where `t` is time elapsed since the auction started and `duration` is the total auction window.

This creates natural price discovery. A solver with low costs (cheap capital, efficient infrastructure) fills early when the user gets a great rate. A solver with higher costs waits until the margin is sufficient. The user always gets filled at the best rate a solver is willing to accept.

**Exclusive fill windows.** Some protocols add an exclusivity period at the start of the auction. A designated solver (often the one who quoted the user's rate off-chain) gets exclusive rights to fill for a short initial window. If they do not fill in time, the auction opens to all solvers. This reduces failed transaction spam while still ensuring competition as a fallback.

## The Fill and Settlement Flow

```
Source Chain          Off-Chain             Destination Chain
     |                    |                       |
     |<-- sign intent --- User                    |
     |                    |                       |
     |-- lock tokens ---->|                       |
     |   emit event       |                       |
     |                    |                       |
     |           Solver sees intent               |
     |           checks profitability             |
     |                    |                       |
     |                    |-- fill order -------->|
     |                    |   (send output        |
     |                    |    to recipient)      |
     |                    |                       |
     |<-- claim + proof --Solver                  |
     |                    |                       |
     |-- reimburse ------>Solver                  |
```

**Step 1 - Lock.** User signs the intent and calls the source chain settler. Input tokens are locked. An event is emitted with the full intent data.

**Step 2 - Fill.** A solver watching the source chain picks up the intent, verifies it is profitable, and sends the output tokens to the recipient on the destination chain. The fill transaction emits a fill event with the order hash.

**Step 3 - Settlement.** The solver needs to prove the fill happened and claim reimbursement from the source chain. This is the hardest part.

## Settlement Mechanisms

### Optimistic Settlement (Across Protocol)

Across uses an optimistic oracle (UMA's Optimistic Oracle v3) for settlement.

After filling on the destination chain, the solver submits a reimbursement claim to the Across hub contract on the source chain. The claim says: "I filled order X on destination chain Y at time Z." A challenge window opens.

During the window, anyone can dispute the claim by posting a bond. If a dispute is raised, UMA's oracle resolves it: both sides post bonds, UMA token holders vote, the loser's bond is slashed and given to the winner.

If no dispute is raised within the window, the claim is assumed valid and the solver is reimbursed.

```
Solver submits claim
         |
         v
Challenge window opens (typically 2 hours)
         |
    No dispute? --> Solver reimbursed
         |
    Dispute raised? --> UMA oracle votes
                              |
                    Solver correct? --> Solver reimbursed + disputer slashed
                    Disputer correct? --> Claim rejected + solver slashed
```

The security guarantee: as long as at least one honest party is watching and willing to dispute invalid claims, the system is secure. Across runs a network of watchers (relayer bots) that monitor fills and auto-dispute invalid claims.

**Speed vs security trade-off.** The challenge window adds latency for the solver (they wait hours before full reimbursement), but the user gets their tokens immediately when the fill happens. The solver absorbs the settlement latency in exchange for the fill fee.

### Hub and Spoke Architecture (Across)

Across uses a hub-and-spoke model. A hub contract sits on a canonical chain (Ethereum mainnet). Spoke contracts sit on each supported chain.

When a user bridges from chain A to chain B:
1. User locks tokens in the spoke on chain A.
2. Solver fills from their balance on chain B.
3. Solver submits claim to the hub on Ethereum.
4. After the challenge window, the hub reimburses the solver.

The hub holds a pool of liquidity (LP tokens) that backs reimbursements. LPs deposit into the hub pool and earn fees from bridge volume. The pool absorbs timing mismatches between fills and reimbursements.

**LP risk.** LPs earn fees but carry the risk of the optimistic oracle failing (a fraudulent claim passing unchallenged). In practice, the watcher network and bond economics make this extremely unlikely.

### ZK Settlement

Instead of an optimistic challenge window, the solver generates a ZK proof that the fill transaction is included in a valid destination chain block. This proof is verified on-chain on the source chain for immediate, trustless reimbursement.

No waiting window. No watcher dependency. But ZK proof generation is expensive and adds latency for the solver.

This is the direction the ecosystem is moving toward. Protocols like Polymer and Succinct are building ZK proof infrastructure specifically for cross-chain settlement. As proof generation costs drop, ZK settlement becomes more viable.

## The Rebalancing Problem

Solvers continuously drain liquidity on destination chains (from fills) and accumulate it on source chains (from reimbursements). Over time, a solver running many intents in one direction ends up with all their capital on the source chain and none on the destination.

This is the **inventory imbalance problem**. Solvers must continuously rebalance:

- Bridge tokens back from source to destination (using some bridge, potentially including the one they are solving for).
- Pay gas and bridge fees on every rebalancing transaction.
- Time rebalancing to minimize capital idle time.

The cost of rebalancing is factored into solver fees. High-volume directional flows (more users bridging A->B than B->A) result in higher fees as solvers' rebalancing costs increase.

**Why large solvers dominate.** A solver with deep capital can buffer large imbalances before needing to rebalance. A small solver hits their limit quickly and has to rebalance more frequently, paying proportionally more in fees. This creates natural centralisation pressure toward large, well-capitalised solvers.

## Capital Efficiency

Solvers tie up capital for the duration of the settlement window. A solver filling a 10,000 USDC intent has 10,000 USDC locked until reimbursement (hours for optimistic settlement).

The capital cost of a fill:

```
capital_cost = fill_amount * rate_of_return * settlement_duration
```

Where `rate_of_return` is what the solver could earn on that capital elsewhere (opportunity cost). For a 2-hour optimistic window and a 5% annual rate of return:

```
capital_cost = 10,000 * (0.05 / (365 * 24)) * 2 = ~$0.11
```

For large fills, this is negligible. For small fills with high gas costs, the math can break down and no solver will fill.

**Minimum viable fill size.** Every intent protocol has an effective floor below which fills are unprofitable: `gas_costs > solver_margin`. This floor rises on high-gas chains and during congestion. Users bridging very small amounts often cannot find a solver.

## Liveness Failure

If no solver fills an intent before the `fillDeadline`, the intent expires. The user did not get their tokens on the destination chain.

The source chain contract allows the user to reclaim their locked input tokens after the deadline. Nothing is lost - but the user's time and the gas spent opening the order are wasted.

Liveness failure happens when:
- No solver finds the intent profitable (amount too small, fees too high).
- All solvers are offline or have no liquidity on the destination chain.
- The fill deadline is too short to complete settlement.

This is the core trade-off vs trustless bridges. A trustless bridge will always settle (assuming Ethereum consensus holds). An intent-based bridge only settles if a solver is willing and able to fill. The user is dependent on solver liveness.

## Security Model

The security of intent-based bridges is economic, not cryptographic.

**What can go wrong:**

1. **Solver fills but claims reimbursement for a different amount.** Caught by the optimistic oracle if watchers are active.
2. **Solver claims reimbursement without filling.** Caught by the watcher network - the fill event is verifiable on the destination chain.
3. **All watchers are offline during the challenge window.** A fraudulent claim passes. The LP pool is drained. This is the catastrophic failure mode.
4. **Smart contract bug in settler or hub.** Locked funds at risk. Standard smart contract risk.

**What cannot go wrong (given honest watchers):**
- Solver cannot steal user funds. The user's locked tokens can only be released to the solver after a valid claim, or returned to the user after deadline.
- Solver cannot prevent the user from reclaiming. The deadline mechanism is enforced on-chain.

Compared to trustless bridges: an attacker breaking a trustless bridge needs to compromise Ethereum's consensus (infeasible). An attacker breaking an intent-based bridge needs to compromise the watcher network and post valid-looking bonds. The bar is lower, but still practically high given economic incentives.

## Intent-Based vs Trustless: When to Use Which

| | Trustless (Light Client) | Intent-Based |
|---|---|---|
| User latency | 13+ min | Seconds |
| Security model | Source chain consensus | Economic (watchers + bonds) |
| Proof cost | High | None at fill time |
| Solver required | No | Yes |
| Works for small amounts | Yes | Only above fill floor |
| Chain support | Requires light client per chain | Wherever solvers operate |
| Catastrophic failure | Source chain compromise | Watcher collusion + fraud |

For high-value transfers where security matters and latency is acceptable: trustless.

For frequent retail transfers where speed and UX matter and amounts are above the solver floor: intent-based.

Most real-world bridge volume is intent-based. Most high-value institutional transfers use trustless or multisig bridges.

## References

- [ERC-7683: Cross Chain Intents Standard](https://eips.ethereum.org/EIPS/eip-7683) - standardised intent format across protocols
- [Across Protocol V3 Documentation](https://docs.across.to/) - hub and spoke architecture, optimistic settlement, UBA fee model
- [UniswapX Whitepaper](https://uniswap.org/whitepaper-uniswapx.pdf) - Dutch auction mechanism and exclusive filler windows
- [UMA Optimistic Oracle V3](https://docs.uma.xyz/) - dispute mechanism used by Across for settlement
- [Bridges Part 1: Building Trustless Bridges with Sync Committees](@/blockchain/bridges-intro.md)
