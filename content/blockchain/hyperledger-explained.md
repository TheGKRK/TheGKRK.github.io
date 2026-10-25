+++
title = "Hyperledger Explained: Enterprise Blockchain, Fabric, Besu, and How It Differs from Public Chains"
date = 2026-10-25
draft = true
description = "A complete guide to Hyperledger in 2026: how it is governed, every major project, how permissioned enterprise blockchains differ from Bitcoin, Ethereum, Solana and Corda, deep dives into Hyperledger Fabric and Hyperledger Besu with code, and a security review checklist."
[taxonomies]
tags = ["blockchain", "hyperledger", "hyperledger-fabric", "hyperledger-besu", "enterprise-blockchain", "permissioned-blockchain", "dlt", "ethereum", "solidity", "consensus", "qbft", "smart-contracts"]
[extra]
toc = true
+++

The most common misunderstanding about Hyperledger is that it is a blockchain. It is not. Hyperledger is a family of open-source projects for building blockchains and distributed ledgers that organisations run for themselves, and those ledgers work very differently from Bitcoin, Ethereum or Solana.

I build cross-chain bridge infrastructure between public chains: Ethereum, BNB Smart Chain and Supra. This guide looks at Hyperledger from that vantage point. It covers what Hyperledger is and how it is governed, every major project in the family, how permissioned ledgers differ from public chains, how each project reaches consensus, deep dives into the two platforms that carry most enterprise workloads (Hyperledger Fabric and Hyperledger Besu), the rest of the family, how to choose between them, and a security checklist for reviewing a deployment.

## Key Takeaways

- Hyperledger is an umbrella of open-source projects under the Linux Foundation, not a single blockchain. There is no Hyperledger coin and no Hyperledger mainnet.
- Its ledgers are permissioned: participants are known organisations, validators are a fixed set, and there is usually no token and no gas fee.
- Hyperledger Fabric replaces the usual blockchain transaction model with execute-order-validate, and builds identity and confidentiality into the protocol.
- Hyperledger Besu is a full Ethereum client, so a private network gets Solidity and the whole Ethereum toolchain, with known validators and instant finality.
- Each ledger has its own consensus protocol: Raft or SmartBFT in Fabric, QBFT in Besu, Sumeragi in Iroha, RBFT in Indy and hashgraph in Hiero. They all trade open participation for fast, final agreement among known parties.
- Permissioned does not mean safe. Most of the risk sits in key custody, governance and integration, not in the cryptography.
- The hardest part of an enterprise blockchain is the consortium, not the code. Projects fail on governance and incentives far more often than on technology.

## What Hyperledger Is

### An Umbrella, Not a Blockchain

The best analogy is the Apache Software Foundation. Apache does not run a web server; it hosts open-source projects that thousands of organisations run on their own machines. Hyperledger works the same way. When a bank says its platform "runs on Hyperledger", it means the bank and its partners run their own network using one of the Hyperledger projects, usually Fabric or Besu. Two Hyperledger Fabric networks run by two different consortia share nothing except the software.

The projects have three things in common:

**Open source under Apache 2.0** - No licence fees and no lock-in to a single vendor. Several companies can support the same network.

**Built for permissioned use** - Participants are known and admitted, not anonymous.

**No native cryptocurrency required** - The ledgers record business data and digital assets without needing a coin to pay validators.

The projects are hosted by [LF Decentralized Trust](https://www.linuxfoundation.org/press/linux-foundation-decentralized-trust-launches-with-17-projects-100-founding-members), the Linux Foundation's home for ledger, identity and cryptography projects. The Hyperledger name is a brand on several of them, most prominently Hyperledger Fabric. Besu is officially just Besu, but the industry still widely calls it "Hyperledger Besu", and so does this post.

### Governance and Project Lifecycle

LF Decentralized Trust has two governing bodies. A **governing board** of member organisations sets budget and strategy. A **Technical Oversight Committee** makes technical decisions, including which projects enter the foundation and how they progress.

Projects move through a lifecycle:

**Labs** - A low-barrier sandbox where anyone can start an experimental project. Paladin and the Fabric Token SDK both began as labs.

**Incubating** - Accepted as a project, building a community and production track record.

**Graduated** - Mature, with an active and diverse set of maintainers and production users. Fabric, Besu, Iroha, Indy, Cacti, FireFly and Hiero are graduated.

**Archived** - No longer maintained.

This matters to an enterprise for a practical reason. A platform with maintainers from several organisations does not die if one vendor walks away, which is exactly the risk a ten-year banking system cannot accept.

### The Project Map in 2026

**Ledgers**

| Project | What it does | Origin |
|---|---|---|
| Hyperledger Fabric | Modular permissioned ledger; smart contracts ("chaincode") in Go, Java or JavaScript/TypeScript | IBM |
| Besu | Java Ethereum client for public and private networks | ConsenSys (as Pantheon) |
| Iroha | Permissioned ledger with built-in instructions for assets and accounts; Iroha 2 is a rewrite in Rust | Soramitsu |
| Indy | Ledger built specifically for decentralised identity | Sovrin Foundation |
| Hiero | Hashgraph-based ledger; the codebase behind the Hedera network | Hedera |
| Lineth | EVM-equivalent zero-knowledge rollup stack for Ethereum Layer 2 networks | Linea Consortium |

**Interoperability and integration**

| Project | What it does |
|---|---|
| Cacti | Connects ledgers to each other, permissioned and public (a merger of the earlier Cactus and Weaver projects) |
| FireFly | Middleware "supernode" that gives applications APIs, events, tokens and data exchange on top of Fabric and EVM chains |
| Web3j | Java and Android library for Ethereum and Besu |

**Decentralised identity**

| Project | What it does |
|---|---|
| AnonCreds | Verifiable credential format with selective disclosure and zero-knowledge predicates |
| Identus | Components for self-sovereign identity, contributed by IOG from its Atala PRISM work |
| CREDEBL | Platform for issuing and managing decentralised identities and credentials |
| Trust over IP | Standards and architecture for internet-scale digital trust |

**Privacy and cryptography**

| Project | What it does |
|---|---|
| Paladin | Programmable privacy on any EVM chain, from Kaleido |
| Panurus | Privacy-preserving, auditable token applications on permissioned ledgers; the evolution of the Fabric Token SDK, maintained by IBM Research |
| Lockness | Key management and signing with multi-party computation (MPC) and threshold cryptography, contributed by Dfns |
| Minokawa | Smart contract language for zero-knowledge execution, formerly Compact, contributed by Shielded Technologies |

Hyperledger Caliper, a benchmarking tool for measuring ledger throughput and latency, rounds out the tooling. Several early projects are archived: Sawtooth, Burrow, Composer, Grid and Avalon, and in 2025 Aries, whose agent frameworks (ACA-Py, Credo) moved to the [OpenWallet Foundation](https://www.lfdecentralizedtrust.org/blog/blockchain-pioneers-hyperledger-aries). A tutorial built on Composer or Sawtooth is a reliable sign that the tutorial is out of date.

### Hyperledger in Production

A few deployments show the range, including one that failed:

**Bakong (Iroha)** - The National Bank of Cambodia's payment system, built with Soramitsu on Hyperledger Iroha, described as the first retail payment system in the world built on blockchain technology. It has since been connected to payment networks in neighbouring countries.

**Hedera (Hiero and Besu)** - The public Hedera network runs on the Hiero codebase, and its smart contract service embeds Besu's EVM as its execution engine ([HIP-26](https://hips.hedera.com/HIP/hip-26)). Hyperledger code now runs inside public networks, not only private ones.

**Banking on Besu** - Besu's EVM compatibility makes it a common choice for bank tokenisation work. Hyperledger set up a [financial services working group for Besu](https://www.ledgerinsights.com/hyperledger-launches-financial-services-besu-blockchain-working-group-as-citi-joins/) when Citi joined it.

**TradeLens (Fabric), a failure worth studying** - IBM and Maersk built a global shipping platform on Hyperledger Fabric. It was [discontinued](https://siliconangle.com/2022/11/30/ibm-maersk-pull-plug-blockchain-based-tradelens-shipping-platform/) in 2023 because, in Maersk's words, "the need for full global industry collaboration has not been achieved". Competing carriers and freight forwarders were reluctant to join a platform run by one dominant player. The technology worked; the consortium did not.

> TradeLens is the most useful case study in enterprise blockchain. A shared ledger only works if the parties sharing it believe no single member controls it, and that is a governance problem before it is a technical one.

## How Hyperledger Differs from Other Blockchains

### Permissionless vs Permissioned

A public blockchain solves the problem of strangers agreeing on shared state. Anyone can join, nobody needs to know who anyone else is, and staked capital (or energy, for Bitcoin) makes cheating expensive.

A permissioned blockchain, often called distributed ledger technology (DLT), solves a different problem. A known group of organisations share a business process but do not fully trust each other: banks settling payments, suppliers and financiers in trade finance, manufacturers and logistics firms in a supply chain. They know each other's legal identities. They do not want competitors reading their data. They do not want a volatile native token. They want finality in seconds and a clear answer to the regulator's question, "who validated this?"

| | Public chain (Ethereum) | Permissioned network (Fabric, Besu) |
|---|---|---|
| Who can join | Anyone | Organisations admitted by the consortium |
| Identity | A key pair, pseudonymous | X.509 certificates (Fabric) or allowlisted nodes and accounts (Besu) |
| Validators | Open set, economic stake | Fixed set run by the members |
| Sybil resistance | Stake | Membership, with legal agreements behind the keys |
| Finality | About 13 minutes (two epochs) | Seconds, no reorganisations |
| Fees | Market-priced gas in a native token | Usually zero, no native token needed |
| Data visibility | Everything public | Members only, with finer partitions inside |

> If one organisation runs every node, it has built a slow replicated database. The value of a permissioned ledger comes from several organisations each running their own nodes, with none of them able to rewrite history alone.

### Seven Differences in Detail

**1. Membership and identity.** On a public chain an identity is a key pair anyone can generate for free, so the protocol cannot know who is behind an address. In Hyperledger networks every node, and often every user, is tied to a real organisation, through X.509 certificates in Fabric or allowlists in Besu. Access control can be written in terms of organisations and roles instead of addresses.

**2. Consensus and finality.** Bitcoin uses proof of work, where finality is probabilistic and six blocks (about an hour) is the usual convention. Ethereum uses proof of stake, where blocks become final after about two epochs. Hyperledger ledgers use voting protocols among a known set of validators, mostly Byzantine fault tolerant (BFT) ones. Finality is immediate, with no chain reorganisations, but these protocols only scale to tens of validators, not tens of thousands. [Consensus in Hyperledger](#consensus-in-hyperledger) below explains each protocol in detail.

**3. Economics.** Public chains pay validators with block rewards and fees, and fees also ration scarce block space and stop spam. Permissioned networks have no token economy. Members run nodes because the shared process benefits them, costs are infrastructure and governance, and spam is handled by identity, permissioning and rate limits.

**4. Privacy.** Public chains are transparent by design. Enterprise ledgers need confidentiality, and the Hyperledger projects take different approaches: Fabric builds it into the protocol with channels and private data collections, Besu moves it to the application layer with tools like Paladin, and identity ledgers like Indy keep personal data off the ledger entirely.

**5. Governance and upgrades.** Public chains upgrade through rough social consensus: improvement proposals, client releases, and node operators choosing to run them. In a Hyperledger network, upgrades follow a consortium agreement. Fabric makes this explicit: a smart contract cannot be deployed or upgraded until enough member organisations approve it. Besu validators vote validators in and out.

**6. Performance.** Without a global, open validator set, throughput is higher and latency lower. The original Fabric research paper ([Androulaki et al., EuroSys 2018](https://arxiv.org/abs/1801.10228)) measured over 3,500 transactions per second in its test setup. Real numbers depend on endorsement policies, the state database, network latency and hardware, so treat any headline TPS figure as a starting point for your own benchmark, which is exactly what Caliper is for.

**7. Smart contract model.** Public chains each have one execution environment (the EVM, Solana's SVM). Hyperledger offers several: the EVM in Besu, general-purpose languages in Fabric, built-in instructions plus WebAssembly in Iroha, and no general smart contracts at all in Indy.

### Side-by-Side Comparison

| | Bitcoin | Ethereum | Solana | Hyperledger Fabric | Hyperledger Besu (private) | R3 Corda |
|---|---|---|---|---|---|---|
| Access | Permissionless | Permissionless | Permissionless | Permissioned | Permissioned | Permissioned |
| Consensus | Proof of work | Proof of stake | Proof of stake | Endorsement policies plus Raft or SmartBFT ordering | QBFT | Notaries confirm uniqueness; no global ordering |
| Smart contracts | Limited script | Solidity, Vyper (EVM) | Rust programs (SVM) | Go, Java, JavaScript/TypeScript | Solidity, Vyper (EVM) | Kotlin, Java (JVM) |
| Native token | BTC | ETH | SOL | None | None needed | None |
| Finality | Probabilistic, about an hour | About 13 minutes | Seconds | On block commit | Every block | On notarisation |
| Data visibility | Public | Public | Public | Channel members; subsets via private data | All network members | Only parties to each transaction |
| Typical use | Payments, store of value | DeFi, tokens, settlement | High-throughput apps, trading | Consortium business processes | Tokenisation, enterprise EVM apps | Financial agreements |

### Hyperledger and Other Enterprise Platforms

**R3 Corda** is the main alternative outside Hyperledger. It is not a blockchain in the strict sense: there is no chain of blocks broadcast to every participant. Transactions are shared only with the parties involved, and notary services confirm that no state is spent twice. That gives strong privacy by default, but makes shared global state, the thing Besu and Fabric are good at, harder to model.

**Quorum** was J.P. Morgan's privacy-focused fork of Ethereum, acquired by ConsenSys in 2020. ConsenSys no longer maintains GoQuorum, Besu removed its GoQuorum compatibility features in 2023, and Quorum networks have been migrating to Besu.

**Hedera** is a public network with a governing council of large organisations running its nodes. Its software is now Hiero, an LF Decentralized Trust project, and its EVM is Besu's. It sits between the public and permissioned worlds: anyone can transact, but the node operators are known.

## Consensus in Hyperledger

Because Hyperledger is a family rather than a single chain, there is no single Hyperledger consensus algorithm. Each ledger uses a protocol that suits its trust model. This section starts with the ideas they all share, then walks through each one.

### The Basics: Safety, Liveness and Fault Types

A consensus protocol must provide two guarantees:

**Safety** - Honest nodes never finalise conflicting blocks. Once a transaction is final, it stays final.

**Liveness** - New transactions keep getting finalised. The network does not get stuck.

And it must survive two kinds of failure:

**Crash faults** - A node stops or becomes unreachable. It may go silent, but it never lies.

**Byzantine faults** - A node behaves arbitrarily: it lies, signs conflicting messages or colludes with others. The name comes from the Byzantine Generals Problem described by Lamport, Shostak and Pease in 1982.

Protocols that survive only crashes are **crash fault tolerant (CFT)**. Protocols that survive arbitrary behaviour are **Byzantine fault tolerant (BFT)**. CFT is cheaper and simpler. BFT is what you need when the nodes belong to organisations that do not fully trust each other.

### Why 2f + 1 and 3f + 1

The node counts come from one idea: **quorum intersection**. Any two quorums that make decisions must overlap in at least one node that will not lie, so two conflicting decisions can never both gather a quorum.

**Crash fault tolerance needs `2f + 1` nodes.** With a majority quorum of `f + 1`, any two majorities share at least one node. Crashed nodes never lie, so that shared node carries the latest decision forward. Five Raft nodes keep working with two of them down.

**Byzantine fault tolerance needs `3f + 1` nodes.** Now a node in the overlap might be lying, so the overlap must contain at least one honest node, which means at least `f + 1` nodes. With `3f + 1` nodes and quorums of `2f + 1`, two quorums overlap in at least `2(2f + 1) - (3f + 1) = f + 1` nodes. At least one of them is honest, and an honest node never votes for two conflicting blocks.

Why not simply wait for every node? Because the `f` faulty nodes can stay silent forever, and a protocol that waits for them can be stalled by them.

| Validators (n) | Faulty validators tolerated (f) | BFT quorum (signatures needed) |
|---|---|---|
| 4 | 1 | 3 |
| 5 | 1 | 4 |
| 7 | 2 | 5 |
| 10 | 3 | 7 |

The second row is a common design trap. A fifth validator adds cost and one more key to protect, but no extra fault tolerance. Size BFT validator sets as `3f + 1`.

### Finality, Accountability and Scale

**Finality is deterministic.** Once a BFT quorum has signed a block, it is final. There is no fork choice rule and no reorganisation. Compare Bitcoin, where finality is only ever probabilistic, and Ethereum, where Casper FFG finalises blocks after about two epochs.

**Liveness is the price.** The Fischer, Lynch and Paterson (FLP) result proves that in a fully asynchronous network, no deterministic protocol can guarantee both safety and liveness if even one node can fail. Practical BFT protocols such as QBFT and SmartBFT assume partial synchrony: they are always safe, and they make progress whenever messages arrive within some bound. When more than `f` nodes are down, or the network splits, they stop producing blocks rather than fork. For financial records, stopping is the right failure mode, but it makes validator uptime a business continuity concern.

**Accountability replaces slashing.** Public proof-of-stake chains punish misbehaving validators by destroying part of their stake. Permissioned BFT has no stake to slash, but every vote is signed by a key belonging to a known organisation. A validator that signs two conflicting blocks has produced cryptographic evidence against a legal entity, and the consortium agreement can act on it.

**Scale is limited.** Most protocols in this family have every validator send messages to every other validator, so message count grows with the square of the validator count. They work well with tens of validators, not tens of thousands. A permissioned network never needs tens of thousands.

### Hyperledger Fabric: Endorsement, Ordering and Validation

Consensus in Fabric is not one algorithm. It is three steps, each settling a different question:

1. **Endorsement: do the right organisations agree on the result?** Endorsing peers execute the transaction and sign the result. The endorsement policy decides which organisations must sign.
2. **Ordering: in what order do transactions happen?** The ordering service puts transactions into blocks, using Raft or SmartBFT.
3. **Validation: is each transaction still valid?** Every peer independently checks the endorsements and the read versions. The check is deterministic, so every honest peer reaches the same verdict without exchanging a single message.

The ordering service is where a classic consensus protocol runs, and Fabric offers two.

**Raft (crash fault tolerant).** Each channel runs its own Raft instance with its own leader. Any orderer that receives a transaction forwards it to the channel's leader. The leader appends transactions to its log and replicates the log to the followers. An entry is committed once a majority has stored it, and the leader cuts blocks from committed entries. Followers expect regular heartbeats from the leader, and if the leader goes quiet, a follower starts an election and becomes the new leader once a majority votes for it.

A three-node Raft cluster survives one failure and a five-node cluster survives two. If a channel loses its majority, ordering on that channel stops. Raft trusts every orderer to be honest. A malicious orderer cannot forge transactions, because clients and endorsers sign them, but a malicious leader can censor or reorder them. Raft can also drop transactions it had acknowledged during a leader change, so clients should watch for commit events and resubmit on timeout.

**SmartBFT (Byzantine fault tolerant, Fabric 3.0 and later).** The leader proposes a block, and the orderers run voting rounds on it. A block is committed once more than two thirds of the orderers agree. Three details make it resistant to a malicious leader:

- **Rotating leadership.** The leader is not elected; it rotates round-robin. Followers that suspect the leader, because heartbeats stop or transactions are ignored, push for a leader change.
- **Submission to every orderer.** Clients send each transaction to all ordering nodes, not just one. The Fabric Gateway does this automatically, so application code does not change.
- **Followers remember transactions.** If the leader ignores a transaction, the followers still hold it, and the leader must eventually include it or be replaced.

SmartBFT tolerates up to, but not including, a third of the orderers being malicious. If a third or more crash or become unreachable, no blocks are agreed. It is enabled per channel with the `V3_0` channel capability, and the same release removed the old Solo and Kafka orderers.

### Hyperledger Besu: QBFT, IBFT 2.0 and Proof of Stake

**QBFT** is the consensus protocol for Besu private networks. It was developed by the J.P. Morgan and ConsenSys protocol teams, is published as a specification by the [Enterprise Ethereum Alliance](https://entethalliance.org/specs/qbft), and belongs to the Istanbul BFT (IBFT) family. Every block height runs through one or more rounds:

```text
Round r at height h   (n = 3f + 1 validators, quorum = 2f + 1)

  proposer ---- PROPOSAL(block) ----> all validators
  each validator checks the block, then broadcasts PREPARE
  on a quorum of PREPAREs  -> broadcast COMMIT, including a commit seal
                              (a signature over the block)
  on a quorum of COMMITs   -> the block is final and imported,
                              with the commit seals stored in its header

  no quorum before the round timer expires
    -> broadcast ROUND-CHANGE, carrying any block this validator has
       seen reach a PREPARE quorum
    -> the next proposer starts round r + 1, and the timer doubles
```

The detail that keeps QBFT safe across rounds is the **prepared certificate**. If a block reached a quorum of PREPARE messages in an earlier round, some honest validators may already have committed it. So when the round changes, the new proposer must re-propose the prepared block from the highest round instead of a new one. A leader change therefore cannot produce two different final blocks at the same height.

The proposer role rotates among validators, so a single offline or malicious proposer only delays its own rounds. The doubling round timer means the network recovers on its own once enough validators are back. The commit seals in each block header give a permanent, verifiable record of exactly which validators approved it.

**IBFT 2.0** is QBFT's predecessor. Besu still supports it and documents a migration path to QBFT, which is the recommended protocol for new networks. The original IBFT 1.0 was deprecated in Besu 23.4.0.

**Proof of stake on Ethereum mainnet.** On public Ethereum, Besu does not run consensus at all. It acts as the execution client, and a separate consensus client (such as Teku, Lighthouse, Prysm, Nimbus or Lodestar) runs Ethereum's proof of stake. The two talk over the authenticated Engine API: the consensus client hands Besu blocks to execute and tells it which chain head to follow. The same Besu binary runs both modes; only the configuration differs.

### Iroha: Sumeragi

Sumeragi assigns roles in each round from an ordered list of peers, the network topology, which is derived from the hash of the last committed block:

**Leader** - Collects transactions and creates the block.

**Validating peers** - Check the block and sign it if it is valid.

**Proxy tail** - Collects the signatures and commits the block once it has `2f + 1` of them.

The network needs at least `3f + 1` peers to tolerate `f` faulty ones. If a block cannot gather enough signatures in time, a **view change** shifts every role by one position in the topology, so a different peer leads the next attempt.

### Indy: RBFT

Indy's consensus library, Plenum, implements **RBFT** (Redundant Byzantine Fault Tolerance), a variant of PBFT designed to fix one specific weakness. In classic PBFT, a malicious leader can stay just under every timeout and slow the network down dramatically without ever triggering a leader change.

RBFT runs `f + 1` instances of the ordering protocol in parallel, each with a different leader. Only the master instance's ordering is actually executed. The backup instances order the same requests purely as a benchmark. If the master instance is significantly slower than the backups, the nodes vote to replace its leader. A slow, malicious leader is caught by comparison, not by timeout.

### Hiero: Hashgraph

Hashgraph, the consensus algorithm behind Hiero and the Hedera network, works without a leader and without vote messages:

**Gossip about gossip** - Every node repeatedly sends everything it knows to a randomly chosen peer. Each gossip event records new transactions plus the hashes of two parent events: the sender's own previous event and the event it just received. These events form a directed acyclic graph, the hashgraph, which records who told what to whom, and when.

**Virtual voting** - Every node eventually holds the same graph, so each node can calculate how every other node would vote, without anyone sending a vote. The order of transactions follows from these virtual votes.

**Fair timestamps** - Each transaction's consensus timestamp is the median of the times at which the nodes first received it, which makes it hard for any single node to reorder transactions in its own favour.

Hashgraph is **asynchronous BFT**: it stays safe and live even with arbitrary network delays, as long as less than a third of the stake is malicious. It gets around the FLP result by using occasional randomised rounds.

### Lineth: Zero-Knowledge Rollup

Lineth works differently again, because it is an Ethereum Layer 2 rather than a standalone ledger. A sequencer orders and executes transactions quickly. A prover then generates a zero-knowledge validity proof for each batch, and a contract on Ethereum verifies it. Once the proof is verified and that Ethereum block is finalised, the batch is as final as Ethereum itself. Security comes from mathematics and from Ethereum's own consensus, not from voting among Layer 2 nodes.

### Consensus at a Glance

| Ledger | Protocol | Fault model | Nodes for `f` faults | Leader | Finality |
|---|---|---|---|---|---|
| Hyperledger Fabric | Endorsement, Raft ordering, validation | Crash (ordering) | `2f + 1` orderers | Elected per channel | On block commit |
| Hyperledger Fabric 3.x | Endorsement, SmartBFT ordering, validation | Byzantine | `3f + 1` orderers | Rotates round-robin | On block commit |
| Besu (private) | QBFT (IBFT 2.0 legacy) | Byzantine | `3f + 1` validators | Rotating proposer | Every block |
| Besu (Ethereum mainnet) | Ethereum proof of stake, via a consensus client | Economic (stake and slashing) | More than two thirds of stake honest | Proposer per slot | About two epochs |
| Iroha | Sumeragi | Byzantine | `3f + 1` peers | Leader and proxy tail, rotated by view change | On block commit |
| Indy | RBFT | Byzantine | `3f + 1` nodes | Master plus `f` backup leaders | On block commit |
| Hiero | Hashgraph | Asynchronous Byzantine | Less than a third of stake malicious | None | Seconds |
| Lineth | Zero-knowledge validity proofs on Ethereum | Inherits Ethereum | Not applicable | Sequencer | When the proof is verified and finalised on Ethereum |

## Hyperledger Fabric in Depth

### Architecture

A Fabric network has four kinds of component:

**Peers** - Hold the ledger and the world state (the current value of every key). For each chaincode, some peers act as endorsers and actually run the code. Every peer validates and commits blocks.

**Ordering service** - Puts transactions into blocks. It never executes chaincode and never sees private data.

**Clients** - Applications holding an X.509 certificate. They build transaction proposals and submit them through the **Fabric Gateway**, a service that runs inside peers since Fabric v2.4, with client APIs for Go, Node.js and Java.

**Certificate authorities** - Each organisation issues certificates to its own peers, orderers, administrators and users, using Fabric CA or existing enterprise PKI.

### Execute, Order, Validate

Ethereum, like most blockchains, is **order-execute**. A proposer orders transactions into a block, then every node executes every transaction in that order and must arrive at the same state. Execution is sequential, every node sees everything, and every node runs every contract.

Fabric flips this into **execute-order-validate**. Three consequences follow:

- Only the endorsers named in a chaincode's endorsement policy execute it, not every node.
- Endorsers can execute different transactions in parallel.
- Chaincode can be written in general-purpose languages, because a non-deterministic result shows up as mismatched endorsements rather than as a chain split.

### The Transaction Flow

```text
Client (signs with its X.509 certificate)
   |
   | 1. proposal: Transfer("asset1", "Org2MSP")
   v
Fabric Gateway (service on a peer)
   |
   | 2. forwards to the endorsers the endorsement policy requires
   v
Endorsing peers (Org1, Org2)
   |  simulate the chaincode against their current world state
   |  produce a read set  {asset1: version 4}
   |  and a write set     {asset1: new value}
   |  sign the result (an endorsement)
   v
Fabric Gateway
   |
   | 3. client signs the transaction envelope, gateway submits it
   v
Ordering service (Raft or SmartBFT)
   |  orders transactions and cuts blocks per channel
   |  never executes chaincode
   v
Every peer on the channel
   |  4. validate: endorsement policy satisfied? read versions still current?
   |  5. commit the block, apply write sets of valid transactions only
   v
Commit status event back to the client
```

**1. Proposal.** The client builds a proposal that invokes a chaincode function and signs it with its certificate.

**2. Endorsement.** Each endorsing peer checks the proposal is well formed, is not a replay, carries a valid signature, and comes from a member allowed to write on the channel. It then runs the chaincode against its current copy of the state. Nothing is written yet. The output is a response, a **read set** (the keys read and their versions) and a **write set** (the keys written and their new values), all signed by the peer.

**3. Ordering.** The gateway checks that the endorsements match, assembles the transaction, and the client signs the envelope. The ordering service sequences it into a block without looking inside.

**4. Validation.** Every peer on the channel checks every transaction in the block. Do the endorsement signatures satisfy the policy? Does every key in the read set still have the version it had during simulation?

**5. Commit.** The block is appended in full. Invalid transactions stay in the block, flagged as invalid, and their writes are skipped. The client receives a commit status event and must check it, because a transaction that was endorsed and ordered can still fail validation.

That last step is the Fabric version of a reverted Ethereum transaction: recorded in the block but changing nothing. The difference is that nobody paid gas for it, and the client is expected to retry.

### Read-Write Sets and MVCC Conflicts

Step 4 is optimistic concurrency control, also called MVCC (multi-version concurrency control). The [Fabric documentation](https://hyperledger-fabric.readthedocs.io/en/latest/readwrite.html) walks through an example worth internalising. Keys `k1` to `k5` start at version 1, and a block contains five transactions in this order:

| Transaction | Reads | Writes | Result |
|---|---|---|---|
| T1 | none | k1, k2 | Valid. k1 and k2 move to version 2 |
| T2 | k1 at version 1 | k3 | Invalid. k1 is now at version 2 |
| T3 | none | k2 | Valid. k2 moves to version 3 |
| T4 | k2 at version 1 | k2 | Invalid. k2 is now at version 3 |
| T5 | k5 at version 1 | k6 | Valid. Nothing touched k5 |

T2 and T4 were simulated against a state that no longer exists by the time they are validated, so they are thrown out. Three practical rules come out of this:

**Hot keys kill throughput.** If every transfer does a read-modify-write on one key (a total supply counter, a shared pool balance), concurrent transactions in the same block conflict and only the first survives. Ethereum serialises these for you; Fabric invalidates them. The standard fix is to write deltas under unique composite keys, such as `balance~alice~<txID>`, and sum them when reading, or to partition state so concurrent transactions touch different keys.

**You do not read your own writes.** Inside one transaction, `GetState` after `PutState` on the same key returns the committed value, not the pending write. The write set is only applied at commit.

**Phantom protection only covers range queries.** A range query (`GetStateByRange`) is recorded in the read set together with its results and re-run at validation. If new keys have appeared in the range, the transaction is invalid. Rich JSON queries against CouchDB (`GetQueryResult`) get no such check, which is why the documentation says not to combine a JSON query and a ledger update in the same transaction.

### Identity: MSPs, Certificates and Policies

A **Membership Service Provider** (MSP) maps certificates to organisations and roles: admin, peer, client, orderer. Proposals, endorsements and blocks are all signed with these keys, using ECDSA or, since Fabric 3.0, Ed25519.

On Ethereum, `msg.sender` is an address anyone can generate. In Fabric, chaincode asks who the caller is in terms of organisation and attributes:

```go
mspID, err := ctx.GetClientIdentity().GetMSPID()
role, found, err := ctx.GetClientIdentity().GetAttributeValue("role")
```

Policies sit on top of identities. The most important one is the **endorsement policy**, which says whose endorsements a chaincode's transactions need:

```text
AND('Org1MSP.peer', 'Org2MSP.peer')
OutOf(2, 'Org1MSP.peer', 'Org2MSP.peer', 'Org3MSP.peer')
```

The endorsement policy is the real security parameter of a Fabric application, the closest thing it has to a multisig threshold. During validation, peers check signatures and versions; they do **not** re-execute the chaincode. An organisation that can satisfy the policy on its own can therefore sign any write set it likes for that chaincode, whatever the code says. If the policy is `Org1MSP.peer` alone, Org1 is trusted absolutely.

### Channels and Private Data Collections

Fabric builds confidentiality into the protocol at two levels.

**Channels.** A channel is a separate ledger with its own members and its own chaincode deployments. Organisations outside a channel see nothing of it. Use one when whole transactions, and the whole ledger, must stay within a set of organisations. The cost is operational overhead, and atomic updates across channels are hard.

**Private data collections.** Inside a channel, a collection lets a subset of organisations share data that the rest of the channel cannot see. The data itself travels peer-to-peer over gossip, only to authorised organisations, and is stored in a private state database. Only a **hash** of it is endorsed, ordered and written to every peer's ledger, so the ordering service never sees the data.

The on-chain hash is useful evidence: a member can later reveal the data to an auditor, and anyone on the channel can check it against the hash with `GetPrivateDataHash`. Two details matter in practice:

- If the private value is predictable (a small amount, a yes/no flag), other channel members can brute-force its hash. Add a random salt to the private data.
- Collections can purge data automatically after a set number of blocks without modification. `PurgePrivateData` removes it from every authorised peer, history included, which matters for data-erasure obligations such as GDPR. `DelPrivateData` only removes it from the current state.

### Choosing the Ordering Service

How Raft and SmartBFT work is covered in [Consensus in Hyperledger](#consensus-in-hyperledger). The practical rule: Raft is enough when the orderers are run by a party every member trusts, or when the members trust each other's operations. When each organisation runs its own orderers and none of them trusts the others, choose SmartBFT. Fabric 2.5 is still the long-term support release, though, so many production networks still run Raft.

### Chaincode

A minimal asset contract using the Go contract API, where ownership is recorded by organisation (MSP ID) rather than by address:

```go
package main

import (
	"encoding/json"
	"fmt"

	"github.com/hyperledger/fabric-contract-api-go/v2/contractapi"
)

type Asset struct {
	ID    string `json:"ID"`
	Owner string `json:"Owner"`
	Value int    `json:"Value"`
}

type AssetContract struct {
	contractapi.Contract
}

func (c *AssetContract) Transfer(ctx contractapi.TransactionContextInterface, id string, newOwner string) error {
	data, err := ctx.GetStub().GetState(id) // recorded in the read set with its version
	if err != nil {
		return err
	}
	if data == nil {
		return fmt.Errorf("asset %s does not exist", id)
	}

	var asset Asset
	if err := json.Unmarshal(data, &asset); err != nil {
		return err
	}

	// the caller's organisation comes from its certificate, not from an address
	caller, err := ctx.GetClientIdentity().GetMSPID()
	if err != nil {
		return err
	}
	if asset.Owner != caller {
		return fmt.Errorf("%s does not own asset %s", caller, id)
	}

	asset.Owner = newOwner
	data, err = json.Marshal(asset)
	if err != nil {
		return err
	}
	return ctx.GetStub().PutState(id, data) // goes into the write set, applied only at commit
}

func main() {
	cc, err := contractapi.NewChaincode(&AssetContract{})
	if err != nil {
		panic(err)
	}
	if err := cc.Start(); err != nil {
		panic(err)
	}
}
```

Two things an Ethereum developer should notice. There is no gas, so an unbounded loop is a liveness problem for the endorser rather than an out-of-gas revert. And Go offers plenty of ways to be accidentally non-deterministic: iterating over a map, calling `time.Now()`, using random numbers, or calling an external API. Each one makes endorsers disagree, and the transaction fails its endorsement policy. Use the transaction timestamp from the stub (`GetTxTimestamp`) instead of the local clock, and sort keys before iterating.

### Chaincode Lifecycle: Governance in the Deploy Step

On Ethereum anyone can deploy a contract, and upgrades go through a proxy controlled by an admin key, a multisig or a timelock. In Fabric, deployment is itself a governance process:

```bash
peer lifecycle chaincode package asset.tar.gz --path ./chaincode --lang golang --label asset_1.0
peer lifecycle chaincode install asset.tar.gz                  # each org, on its own peers
peer lifecycle chaincode approveformyorg ... --sequence 1      # each org approves the definition
peer lifecycle chaincode checkcommitreadiness ... --sequence 1 # who has approved so far
peer lifecycle chaincode commit ... --sequence 1               # once enough orgs approve
```

Each organisation approves a **chaincode definition**: name, version, sequence number, endorsement policy and collection configuration. The commit succeeds once enough organisations have approved, as set by the channel's `LifecycleEndorsement` policy (a majority of organisations by default). An upgrade is the same process with the sequence number incremented.

Upgrade authority therefore belongs to the consortium, not to a single admin key. One subtlety: organisations approve the definition, not a shared code hash. Each organisation installs its own package, so two organisations can run different code as long as it produces the same read-write sets for the same inputs.

## Hyperledger Besu in Depth

### An Ethereum Client for Private Networks

Besu is an Ethereum client written in Java under the Apache 2.0 licence. It has two roles: an execution client on Ethereum mainnet, paired with a consensus client, and a node for private and consortium networks, where it runs its own consensus and needs nothing else.

Its biggest advantage is that it is Ethereum. Solidity, the EVM, the JSON-RPC API, Ethers.js, Web3j, Foundry and Hardhat all work unchanged. That has three practical consequences for an enterprise:

**Skills** - Any Ethereum developer is productive on day one. There is no new language or SDK to learn.

**Standards** - Token standards such as ERC-20, ERC-721 and ERC-3643 (permissioned security tokens), audit tooling and security knowledge all carry over.

**A path to public chains** - Assets issued on a private Besu network (tokenised deposits, bonds, invoices) can later be bridged to or settled on public Ethereum with the same contracts and tools.

What changes from public Ethereum is configuration: who validates, who can connect, who can transact, and what gas costs.

### Reference Architecture

```text
      Enterprise systems (ERP, core banking, partner portals)
                          |
                    REST API gateway
                          |
          Blockchain integration microservice
     (Ethers.js, signer service, nonce queue, event indexer)
                          |   JSON-RPC over TLS
                  Besu RPC nodes (non-validating)
                          |
    +------------+------------+------------+------------+
    | Validator  | Validator  | Validator  | Validator  |   QBFT consensus,
    |   Org A    |   Org B    |   Org C    |   Org D    |   one per member
    +------------+------------+------------+------------+
           node allowlist + account allowlist on every node
```

Four design rules shape this picture:

1. **One validator per member organisation, run on that organisation's own infrastructure.** If one team runs every validator, that team holds every validator key, and the network is decentralised on paper only.
2. **Applications never talk to validators.** Separate RPC nodes serve application traffic, so a traffic spike or a heavy query cannot slow down block production.
3. **Applications never hold keys.** A dedicated integration service signs transactions through a key management service or hardware security module, and exposes ordinary REST APIs to the rest of the enterprise.
4. **Reads come from an index, not the chain.** The integration service writes contract events into a database, and dashboards and reports read from there.

### Configuring QBFT

How QBFT reaches agreement is covered in [Consensus in Hyperledger](#consensus-in-hyperledger). For an application, the result is simple: with at least four validators the network tolerates one faulty validator, and one confirmation is final, so a transaction can be treated as settled as soon as it is in a block.

The [QBFT settings](https://docs.besu-eth.org/private-networks/how-to/configure/consensus/qbft) live in the genesis file (abridged here; generate the real one with `besu operator generate-blockchain-config`):

```json
{
  "config": {
    "chainId": 1981,
    "londonBlock": 0,
    "zeroBaseFee": true,
    "contractSizeLimit": 2147483647,
    "qbft": {
      "blockperiodseconds": 2,
      "epochlength": 30000,
      "requesttimeoutseconds": 4
    }
  },
  "gasLimit": "0x1fffffffffffff",
  "extraData": "0x..."
}
```

`blockperiodseconds` is the minimum block time and `requesttimeoutseconds` the round timeout. `extraData` is RLP-encoded and carries the initial validator list.

Validators can be added and removed in two ways:

**Block header voting** - Existing validators call `qbft_proposeValidatorVote` with an address and `true` (add) or `false` (remove). The change happens once more than half of the current validators have proposed it, and pending votes reset every `epochlength` blocks.

**Validator smart contract** - A contract holds the validator set, so membership changes can follow an on-chain governance process, such as a vote among member organisations. A `transitions` entry in the genesis file can switch between the two modes at a set block.

### Cryptography Under the Hood

A Besu network inherits Ethereum's cryptography, and it helps to know which key does what:

**Account keys** - Transactions are signed with ECDSA over the secp256k1 curve. An account address is the last 20 bytes of the Keccak-256 hash of the public key.

**Node keys** - Every node has its own secp256k1 key pair. Its public key is the node's identity in the `enode://` URL, and peer connections are encrypted with the RLPx protocol, which uses an ECIES handshake.

**Validator seals** - In QBFT the node key is also the validator's signing key. Validators sign each block, and the signatures (commit seals) are stored in the block header, so anyone holding the chain can verify who approved each block.

**Merkle Patricia tries** - Every block header commits to the state, the transactions and the receipts through Merkle roots. A member can prove a single transaction or receipt to an external auditor without handing over the whole ledger.

The node key is the most sensitive file in the deployment. Whoever holds it can sign blocks as that validator.

### Permissioning

Besu's built-in permissioning is local: each node reads a `permissions_config.toml` file.

```toml
nodes-allowlist=["enode://<node-public-key>@10.0.0.2:30303", "enode://<node-public-key>@10.0.0.3:30303"]
accounts-allowlist=["0xb9b81ee349c3807e46bc71aa2632203c5b462032"]
```

```bash
besu --permissions-nodes-config-file-enabled --permissions-accounts-config-file-enabled ...
```

The node allowlist controls which peers a node will connect to. The account allowlist is subtler. A node checks it when a transaction is submitted through `eth_sendRawTransaction`, when one arrives from another node, and when the node builds a block itself. It does **not** re-check blocks received from other nodes.

So account permissioning is only as strong as the validators that build blocks. If one validator's allowlist is out of date, it can include transactions that every other node would reject, and every node will still accept its block. Validator allowlists must stay identical, so manage them through configuration management, not manual edits.

Smart-contract-based (onchain) permissioning was [removed in Besu 25.6.0](https://lfdecentralizedtrust.org/blog/sunsetting-tessera-and-simplifying-hyperledger-besu). Rules more dynamic than a static list now belong either in a Besu plugin or in the application contracts themselves, which is the pattern ERC-3643 security tokens use: every transfer checks an on-chain identity registry.

### Gas on a Private Network

The EVM still meters every operation, but on a private network the price of gas can be zero. The genesis file above already has the [free gas settings](https://docs.besu-eth.org/private-networks/how-to/configure/free-gas): `zeroBaseFee` (needed on London or later forks), a very large `gasLimit` and a raised `contractSizeLimit`. Each node then runs with:

```bash
besu --min-gas-price=0 --tx-pool-enable-balance-check=false ...
```

Three things to know:

- **Every node needs the zero minimum gas price.** A node with a higher minimum silently drops zero-price transactions, which looks like transactions vanishing.
- **There is still a per-transaction limit.** On QBFT and IBFT 2.0 networks, EIP-7825 caps a single transaction at 16,777,216 gas unless `pertxgaslimit` is raised in the consensus config.
- **Free gas removes the economic brake.** Any account allowed to transact can fill blocks at no cost, so account permissioning and API rate limits take over the role that fees play on a public chain.

### Privacy After Tessera

For years, Besu offered private transactions through Tessera, a separate private transaction manager. Besu announced its [sunset](https://lfdecentralizedtrust.org/blog/sunsetting-tessera-and-simplifying-hyperledger-besu) in release 24.12.0 and removed it in 25.6.0, citing performance and a code base that had become, in the maintainers' words, "a monolithic swiss-army knife". Tutorials that set up Tessera, or use the `priv_` and `eea_` JSON-RPC methods, no longer apply to current Besu.

Confidentiality on Besu now lives at the application layer:

- **Hashes on-chain, data off-chain.** Every member can read every transaction, so documents stay in each organisation's own systems and only salted hashes go on-chain. This covers most enterprise cases.
- **[Paladin](https://www.lfdecentralizedtrust.org/blog/paladin-a-unified-programmable-privacy-framework-for-evm-becomes-newest-lf-decentralized-trust-project)** runs alongside any EVM chain and offers notary-backed confidential tokens (Noto), zero-knowledge confidential tokens (Zeto), and private EVM execution inside privacy groups (Pente).
- **Separate networks** for groups of members who must not see each other's activity at all.

This is the sharpest philosophical difference between the two big Hyperledger ledgers. Fabric puts confidentiality in the core protocol. Besu has moved it out of the client and into layers on top.

### Smart Contract Development, Testing and Deployment

Here is a contract that solves a real enterprise problem: double financing in trade finance, where a supplier presents the same invoice to several banks and gets paid more than once. A shared registry that every bank can check, but no single bank controls, stops it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

/// @notice Shared registry that stops the same invoice from being financed twice.
contract InvoiceRegistry is AccessControl {
    bytes32 public constant SUPPLIER_ROLE = keccak256("SUPPLIER_ROLE");
    bytes32 public constant FINANCIER_ROLE = keccak256("FINANCIER_ROLE");

    enum Status { None, Registered, Financed, Settled }

    struct Invoice {
        address supplier;
        address financier;
        uint64 dueDate;
        Status status;
    }

    /// @dev keyed by keccak256(invoice document, salt); the document itself stays off-chain
    mapping(bytes32 => Invoice) public invoices;

    event InvoiceRegistered(bytes32 indexed invoiceHash, address indexed supplier, uint64 dueDate);
    event InvoiceFinanced(bytes32 indexed invoiceHash, address indexed financier);
    event InvoiceSettled(bytes32 indexed invoiceHash);

    error AlreadyRegistered(bytes32 invoiceHash);
    error InvalidStatus(bytes32 invoiceHash, Status status);
    error NotFinancier(bytes32 invoiceHash, address caller);

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
    }

    function register(bytes32 invoiceHash, uint64 dueDate) external onlyRole(SUPPLIER_ROLE) {
        if (invoices[invoiceHash].status != Status.None) revert AlreadyRegistered(invoiceHash);
        invoices[invoiceHash] = Invoice(msg.sender, address(0), dueDate, Status.Registered);
        emit InvoiceRegistered(invoiceHash, msg.sender, dueDate);
    }

    function finance(bytes32 invoiceHash) external onlyRole(FINANCIER_ROLE) {
        Invoice storage invoice = invoices[invoiceHash];
        if (invoice.status != Status.Registered) revert InvalidStatus(invoiceHash, invoice.status);
        invoice.financier = msg.sender;
        invoice.status = Status.Financed;
        emit InvoiceFinanced(invoiceHash, msg.sender);
    }

    function settle(bytes32 invoiceHash) external {
        Invoice storage invoice = invoices[invoiceHash];
        if (invoice.status != Status.Financed) revert InvalidStatus(invoiceHash, invoice.status);
        if (msg.sender != invoice.financier) revert NotFinancier(invoiceHash, msg.sender);
        invoice.status = Status.Settled;
        emit InvoiceSettled(invoiceHash);
    }
}
```

The design decisions worth calling out:

**Only a salted hash goes on-chain.** Without the salt, a competitor could hash guessed invoice details and check whether they match.

**Roles, not owners.** OpenZeppelin `AccessControl` maps cleanly onto organisational roles. In production, the admin role belongs to a multisig controlled by several member organisations, not to one key.

**Events are the audit trail.** Every state change emits an event, which the integration layer indexes for reporting.

**Custom errors carry meaning to the API.** `AlreadyRegistered` becomes an HTTP 409 Conflict in the REST layer instead of a generic failure.

The core invariant gets a fuzz test in [Foundry](https://getfoundry.sh/), which tries hundreds of random invoice hashes:

```solidity
// setUp() deploys the registry and grants SUPPLIER_ROLE and FINANCIER_ROLE to the test accounts
function testFuzz_InvoiceCannotBeFinancedTwice(bytes32 invoiceHash) public {
    vm.prank(supplier);
    registry.register(invoiceHash, uint64(block.timestamp + 30 days));

    vm.prank(financierA);
    registry.finance(invoiceHash);

    vm.prank(financierB);
    vm.expectRevert(
        abi.encodeWithSelector(InvoiceRegistry.InvalidStatus.selector, invoiceHash, InvoiceRegistry.Status.Financed)
    );
    registry.finance(invoiceHash);
}
```

One deployment detail catches teams out. Recent Solidity compilers target recent Ethereum forks by default, and a contract compiled for Shanghai or later contains the `PUSH0` opcode. If the genesis file only enables London, deployment fails. Match the compiler to the network, or enable the later forks in the genesis file:

```toml
# foundry.toml
[profile.default]
solc_version = "0.8.28"
evm_version = "london"  # must match the forks enabled in the genesis file
```

Deployment is then the same as on any EVM chain, with the deployer key kept in an encrypted Foundry keystore rather than a shell variable:

```bash
forge create src/InvoiceRegistry.sol:InvoiceRegistry \
  --rpc-url "$BESU_RPC_URL" \
  --account deployer \
  --broadcast \
  --constructor-args "$ADMIN_MULTISIG"
```

### Integration with Enterprise Applications

Enterprise systems do not speak JSON-RPC. They speak REST, and they expect the usual guarantees: requests can be retried safely, errors mean something, and reads are fast. An integration microservice provides those guarantees on top of the chain. A minimal version with Express and [Ethers.js](https://docs.ethers.org/v6/) v6:

```js
import express from "express";
import { ethers } from "ethers";
import abi from "./InvoiceRegistry.abi.json" with { type: "json" };

const provider = new ethers.JsonRpcProvider(process.env.BESU_RPC_URL);
// demo only: in production the key sits in a KMS or HSM behind a signer service
const signer = new ethers.Wallet(process.env.SIGNER_KEY, provider);
const registry = new ethers.Contract(process.env.REGISTRY_ADDRESS, abi, signer);

const app = express();
app.use(express.json());

app.post("/invoices", async (req, res) => {
  const { invoiceHash, dueDate } = req.body;
  try {
    const tx = await registry.register(invoiceHash, dueDate);
    const receipt = await tx.wait(1); // QBFT finality: one block is final
    res.status(201).json({ txHash: receipt.hash, blockNumber: receipt.blockNumber });
  } catch (err) {
    if (err.code === "CALL_EXCEPTION" && err.revert?.name === "AlreadyRegistered") {
      return res.status(409).json({ error: "invoice already registered" });
    }
    res.status(502).json({ error: "ledger unavailable" });
  }
});

app.listen(8080);
```

The production concerns around this demo are where the engineering happens:

**Idempotency.** Networks time out, and clients retry. Without protection, a retry sends a second transaction, and on a ledger a duplicate is permanent. Clients send an `Idempotency-Key` header, the service records the key and the transaction hash before broadcasting, and a retry returns the stored result instead of sending again. This is the same discipline payment APIs use.

**Nonce management.** Every transaction from an account needs the next nonce in sequence. Two concurrent requests from the same signer can pick the same nonce, and one of them fails or replaces the other. Serialise sends per signing account through a queue, or wrap the signer in Ethers' `NonceManager`.

**Finality.** On QBFT, one confirmation is final, so `tx.wait(1)` is enough. That assumption must not be copied into code that also talks to public chains, where blocks can still be reorganised.

**Event indexing.** `contract.on(...)` is fine for a demo, but a production indexer records the last block it processed and, after a restart, catches up with `queryFilter` from that block before listening again.

A note on libraries: Web3.js was [archived by ChainSafe in March 2025](https://blog.chainsafe.io/web3-js-sunset/), with Ethers.js and Viem as the recommended replacements. Existing Web3.js integrations still run, but new services should start on Ethers.js or Viem, or on Web3j for Java teams.

### Deployment: Docker, Kubernetes and Cloud

Besu publishes an official Docker image, `hyperledger/besu`, so each node runs as a container. On Kubernetes, a sound layout looks like this:

- **Validators as StatefulSets** with persistent volumes. Chain data must survive pod restarts, and each validator needs a stable identity.
- **Node keys as secrets** backed by the cloud provider's secrets manager or KMS, never baked into images.
- **RPC nodes as a separate, horizontally scaled tier** behind an internal load balancer. Validators expose no JSON-RPC to applications at all.
- **Minimal RPC APIs.** RPC nodes enable only the API groups applications need (`ETH`, `NET`, `WEB3`) through `--rpc-http-api`. Administrative and debug APIs stay off, or are reachable only from an operations network. Besu also supports TLS and authentication on its JSON-RPC endpoints.
- **Monitoring.** With `--metrics-enabled`, Besu exposes Prometheus metrics. The alerts that matter most are no new block within a few block periods (consensus has stalled), falling peer counts, and validators missing from block signatures.
- **Rolling upgrades,** one validator at a time. With four validators only one can be offline at once, so each node must rejoin before the next one starts.

In a real consortium, each member runs its validator in its own cluster and cloud account. A network spread across AWS, Azure and GCP is not an exotic requirement. It is what naturally happens when four organisations each bring their own infrastructure, and it removes any single cloud provider or administrator as a point of failure.

## The Rest of the Family

### Iroha

Iroha takes a different approach to smart contracts. Instead of starting with a general-purpose virtual machine, it ships built-in instructions (Iroha Special Instructions) for the operations most asset systems need: create accounts, register and mint assets, transfer, grant permissions. Iroha 2 is a rewrite in Rust that adds WebAssembly smart contracts for custom logic, and uses Sumeragi, a BFT consensus protocol that tolerates up to a third of nodes being faulty. The built-in model suits payment and asset systems where the operations are well known in advance, which is why a central bank payment system like [Bakong](https://www.lfdecentralizedtrust.org/blog/2020/08/17/national-bank-of-cambodias-new-digital-payment-system-how-soramitsu-helped-modernize-retail-payments-using-hyperledger-iroha) could be built on it.

### Indy and Decentralised Identity

Indy is a ledger built for one purpose: decentralised identity. The key design decision is what it does **not** store. Personal data never goes on the ledger. Indy holds only public material: decentralised identifiers (DIDs) of credential issuers, credential schemas, credential definitions (the issuer's public keys), and revocation registries. The credentials themselves live in the holder's digital wallet.

With AnonCreds, a holder can prove facts about a credential without revealing the credential, for example proving they are over 18 without revealing their date of birth. A verifier checks the proof against the issuer's public keys on the ledger and never needs to contact the issuer. Indy reaches consensus among its validator nodes with RBFT, a redundant variant of PBFT.

The agent software that issued, held and verified credentials (Hyperledger Aries) was archived in 2025, and its main frameworks continue under the OpenWallet Foundation. Identus and CREDEBL carry on identity work inside LF Decentralized Trust.

### Cacti, FireFly and Web3j: Connecting Ledgers to Everything Else

A ledger is useless in isolation. Three projects handle the connections:

**Cacti** connects ledgers to each other: asset transfers, data sharing and atomic exchanges across Fabric, Besu and public chains. Bridging permissioned networks raises the same trust questions as bridging public ones, which I covered in [Part 1](@/blockchain/bridges-intro.md) and [Part 2](@/blockchain/bridges-part-2.md) of my bridges series.

**FireFly** sits between applications and the ledger. It provides REST APIs, event streams, token operations and off-chain data exchange, so application teams do not each rebuild the integration layer described in the Besu section.

**Web3j** is the standard Java library for Ethereum and Besu. For enterprises whose core systems run on the JVM, it generates type-safe Java wrappers from Solidity contracts.

### Paladin, Panurus, Lockness and Minokawa: The Privacy Layer

The newest projects show where enterprise blockchain is heading: privacy and key security, not new ledgers. Paladin brings confidential tokens and private execution to any EVM chain. Panurus does the same for token applications on permissioned ledgers, with auditability built in so a regulator can still see what it needs to. Lockness provides MPC and threshold signing, so no single machine ever holds a whole private key. Minokawa is a language for writing smart contracts that execute with zero-knowledge proofs.

## Choosing the Right Platform

### Do You Need a Blockchain at All?

Before choosing a Hyperledger project, answer four questions honestly:

1. **Do several organisations need to write to the same record?** If only one does, use a database.
2. **Do they distrust each other enough that none of them should run it alone?** If everyone trusts one operator, let that operator run a database with an audit log.
3. **Can the participants agree on governance:** who joins, who validates, who approves upgrades, who pays? If not, the project will fail regardless of technology, as TradeLens showed.
4. **Is the data allowed on a shared ledger at all?** Personal data, and anything subject to erasure rights such as GDPR or India's DPDP Act, belongs off-chain.

### Decision Guide

| If the main requirement is | Choose |
|---|---|
| Ethereum skills, Solidity, token standards, a future path to public Ethereum | Hyperledger Besu |
| Confidential data partitioned between different subsets of members | Hyperledger Fabric |
| Each contract encoding exactly which organisations must agree | Hyperledger Fabric |
| A payment or asset system with well-known operations | Iroha |
| Verifiable credentials and decentralised identity | Indy with AnonCreds, or Identus |
| Connecting a permissioned ledger to other ledgers | Cacti |
| Open participation, public liquidity, censorship resistance | A public chain, not Hyperledger |

## Security Review Checklist

These are the checks I would run when reviewing a Hyperledger deployment.

**Both platforms**
- Validator or orderer count is `3f + 1` for BFT, and each runs on a different member's infrastructure.
- Node keys, validator keys and CA keys are held in a secrets manager or HSM, with access logged.
- Admin and upgrade rights need several member organisations, never a single key.
- No personal data or plain documents on-chain. Hashes are salted.
- Every write endpoint in the integration layer is idempotent, nonces or retries are managed, and the event indexer can replay from a stored checkpoint.

**Hyperledger Fabric**
- No endorsement policy can be satisfied by a single organisation for chaincode that moves value.
- Chaincode is deterministic: no map iteration without sorting, no wall-clock time, no randomness, no external calls.
- No read-modify-write on hot keys.
- No JSON rich queries in update transactions.
- Certificate authorities are protected as roots of trust: whoever controls an organisation's CA can issue certificates as that organisation.

**Hyperledger Besu**
- Account and node allowlists are identical on every validator and managed as code.
- RPC nodes expose only the API groups applications need, behind TLS and authentication. Validators expose none.
- Free gas is paired with account permissioning and API rate limits.
- The compiler's `evm_version` matches the forks enabled in the genesis file.
- Standard Solidity review: access control on every state-changing function, reentrancy, unchecked external calls, event coverage, plus unit tests, fuzz tests for core invariants, and an external audit before go-live.

## Closing Thoughts

Hyperledger is easiest to understand as an answer to a specific question: how do organisations that do not fully trust each other keep one shared record, without handing control to any one of them and without the anonymity, tokens and fee markets of a public chain? Fabric answers it by redesigning the transaction model around organisations and policies. Besu answers it by taking Ethereum and closing the doors. Iroha, Indy and the newer privacy projects answer narrower versions of the same question.

Experience from public chains transfers well, because the public-chain threat model is a superset: BFT quorum maths, key custody, contract security and careful finality handling all apply directly. A permissioned network removes the anonymous attacker. It does not remove the compromised key, the misconfigured node, the weak endorsement policy or the integration service that sends the same transaction twice.

## References

- [Linux Foundation Decentralized Trust launch announcement](https://www.linuxfoundation.org/press/linux-foundation-decentralized-trust-launches-with-17-projects-100-founding-members) - the foundation that hosts the Hyperledger projects, and the Besu rename
- [LF Decentralized Trust projects](https://www.lfdecentralizedtrust.org/projects) - the current project list and lifecycle status
- [Hyperledger Fabric: A Distributed Operating System for Permissioned Blockchains](https://arxiv.org/abs/1801.10228) - the original Fabric research paper
- [Fabric: Transaction Flow](https://hyperledger-fabric.readthedocs.io/en/latest/txflow.html) - endorsement, ordering, validation and commit
- [Fabric: Read-Write Set Semantics](https://hyperledger-fabric.readthedocs.io/en/latest/readwrite.html) - MVCC validation and range query phantom protection
- [Fabric: Private Data](https://hyperledger-fabric.readthedocs.io/en/latest/private-data/private-data.html) - collections, gossip, on-chain hashes and purging
- [Fabric Gateway](https://hyperledger-fabric.readthedocs.io/en/latest/gateway.html) - the client-facing service introduced in Fabric v2.4
- [Fabric: The Ordering Service](https://hyperledger-fabric.readthedocs.io/en/latest/orderer/ordering_service.html) - Raft and SmartBFT ordering in Fabric
- [The Raft Consensus Algorithm](https://raft.github.io/) - leader election and log replication explained
- [EEA QBFT Specification](https://entethalliance.org/specs/qbft) - the QBFT protocol used by Besu
- [IBFT 2.0: A Safe and Live Variation of the IBFT Blockchain Consensus Protocol](https://arxiv.org/abs/1909.10194) - formal analysis of round changes and prepared certificates in the IBFT family
- [Indy Plenum](https://github.com/hyperledger/indy-plenum) - Indy's RBFT consensus implementation
- [Hedera is open source as Hiero](https://hedera.com/hiero-open-source/) - the hashgraph codebase under LF Decentralized Trust
- [Hyperledger Fabric v3: Smart Byzantine Fault Tolerant Consensus](https://lfdecentralizedtrust.org/blog/hyperledger-fabric-v3-delivering-smart-byzantine-fault-tolerant-consensus) - the SmartBFT orderer
- [Besu on GitHub](https://github.com/besu-eth/besu) - source code and release notes
- [Besu: Configure QBFT](https://docs.besu-eth.org/private-networks/how-to/configure/consensus/qbft) - genesis options, validator voting and fault tolerance
- [Besu: Local Permissioning](https://docs.besu-eth.org/private-networks/how-to/use-local-permissioning) - node and account allowlists
- [Besu: Free Gas Networks](https://docs.besu-eth.org/private-networks/how-to/configure/free-gas) - zero base fee configuration and caveats
- [Sunsetting Tessera and Simplifying Besu](https://lfdecentralizedtrust.org/blog/sunsetting-tessera-and-simplifying-hyperledger-besu) - why private transactions and onchain permissioning were removed
- [Paladin becomes an LF Decentralized Trust project](https://www.lfdecentralizedtrust.org/blog/paladin-a-unified-programmable-privacy-framework-for-evm-becomes-newest-lf-decentralized-trust-project) - Noto, Zeto and Pente
- [HIP-26: Hedera's migration to the Besu EVM](https://hips.hedera.com/HIP/hip-26) - Besu's EVM inside a public network
- [Announcing Hyperledger Iroha 2](https://www.lfdecentralizedtrust.org/blog/announcing-hyperledger-iroha-2) - the Rust rewrite, Sumeragi and WebAssembly contracts
- [Blockchain Pioneers: Hyperledger Aries](https://www.lfdecentralizedtrust.org/blog/blockchain-pioneers-hyperledger-aries) - why Aries was archived and where its frameworks went
- [IBM and Maersk discontinue TradeLens](https://siliconangle.com/2022/11/30/ibm-maersk-pull-plug-blockchain-based-tradelens-shipping-platform/) - the governance lesson
- [Ethers.js v6 documentation](https://docs.ethers.org/v6/) - providers, contracts, errors and `NonceManager`
- [Web3.js sunset announcement](https://blog.chainsafe.io/web3-js-sunset/) - ChainSafe's migration guidance
- [Bridges Part 1: Building Trustless Bridges with Sync Committees](@/blockchain/bridges-intro.md) - my post on light-client bridges between public chains
- [Bridges Part 2: Intent-Based Bridges and Atomic Swaps](@/blockchain/bridges-part-2.md)
