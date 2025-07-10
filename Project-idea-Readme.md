
# Move Smart Contract Product Requirement Documents (PRDs)

> Purpose: Provide five well‑scoped product ideas—each with a detailed PRD and on‑chain design—for developers who want to master the Move language on **Aptos** and **Sui**.  These documents are intended as build guides, learning road‑maps, and reference specs.

---

## Table of Contents

1. [FlowStream – Continuous Payment Streaming Protocol](#1-flowstream)
2. [Rentable – NFT Time‑Bound Rental Marketplace](#2-rentable)
3. [MoveDAO – Modular On‑Chain Governance & Treasury](#3-movedao)
4. [NodeStaker – DePIN Sensor Staking & Reward Module](#4-nodestaker)
5. [MoveBridge – Cross‑Network Liquidity & Message Bridge](#5-movebridge)

---

<a name="1-flowstream"></a>

## 1. **FlowStream – Continuous Payment Streaming Protocol**

#### Suggested GitHub Repo Name

`move-flowstream-payments`

### Vision & Goals

Create an on‑chain protocol that lets users stream fungible tokens (APT, SUI, or any Move‑based asset) to recipients in real‑time, unlocking use‑cases such as payroll, subscriptions, and grant disbursement. Emphasize efficient resource handling and time‑based calculations.

### Personas & Use Cases

| Persona         | Need                        | Example                |
| --------------- | --------------------------- | ---------------------- |
| Indie developer | Pay SaaS hosting per second | 0.01 APT/hr stream     |
| DAO treasurer   | Continuous payroll          | Pay contributors       |
| Investor        | Vest seed tokens linearly   | 1‑yr cliffless vesting |

### Functional Requirements

1. **CreateStream**: Sender defines `token`, `rate_per_second`, `start_time`, `end_time`, `recipient`.
2. **WithdrawStreamed**: Recipient may pull accrued amount at any time.
3. **CancelStream**: Sender cancels; remaining balance refunds appropriately.
4. **TopUp**: Anyone may add funds to an active stream.
5. **Pause/Resume** *(v2)*.

### Smart‑Contract Architecture

| Module                | Responsibility                                                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `flowstream::manager` | Entry points, authority checks                                                                                                       |
| `flowstream::stream`  | `struct Stream { id, sender, recipient, token, rate, start, end, deposited, withdrawn }` stored as a resource under sender’s account |
| `flowstream::utils`   | Time math, safe division                                                                                                             |

#### Key Resources & Capabilities

* **Resource** `Stream` – guarantees single writer (sender) via `key` ability.
* Use **event** streams: `StreamCreated`, `Withdraw`, `Cancelled`.
* Leverage **Aptos Timestamp** / **Sui Clock** oracle modules.

#### Entry Functions (Public Interfaces)

```move
public entry fun create_stream<Currency>(
    sender: &signer,
    recipient: address,
    rate: u64,
    duration_secs: u64,
    initial_deposit: u64
)
```

*(full signature list continues)*

### Aptos vs. Sui Divergence

| Aspect    | Aptos                              | Sui                               |
| --------- | ---------------------------------- | --------------------------------- |
| Clock     | `0x1::timestamp` seconds‑precision | `0x3::clock` ms‑precision         |
| Fee model | Gas units, parallel execution      | Object‑centric, gas after effects |
| Storage   | Table handles                      | Object mappings                   |

### Invariants & Security

* Total withdrawn ≤ deposited.
* One active stream ID per (sender, nonce).
* Reentrancy safe via Move’s single‑threaded execution.

### Gas & Performance Targets

* `create_stream` ≤ 70k gas.
* `withdraw_streamed` ≤ 40k gas on median path.

### Upgrade Path & Governance

* Owned by DAO; `upgrade_cap` stored in governance vault.

### Testing & Simulation Plan

* Unit tests for edge times.
* Property test: Sum of balances constant.
* Integration: Faucet tokens on devnet, CLI demo script.

### Future Extensions

* Stream NFTs (transferable rights).
* Streaming with cliff + linear vest.

---

<a name="2-rentable"></a>

## 2. **Rentable – NFT Time‑Bound Rental Marketplace**

#### Suggested GitHub Repo Name

`move-nft-rentable`

### Vision & Goals

Enable owners to lend NFTs for a fixed period while retaining ultimate ownership, unlocking in‑game asset rentals or paid access passes.

### Personas & Use Cases

* **Gamer**: Rents rare sword for weekend tournament.
* **Landlord**: Lends metaverse land plot for advertising.

### Core Features

1. **ListForRent**: Specify NFT object, daily price, max duration, collateral (optional).
2. **Rent**: Renter pays upfront; receives time‑limited usage right (wrapped NFT).
3. **Auto‑Return**: Contract‐enforced reclaim after expiry.
4. **Early Return & Refund**: Unused days refunded minus fee.

### Smart‑Contract Architecture

| Module              | Summary                                            |
| ------------------- | -------------------------------------------------- |
| `rentable::market`  | Listing registry, price oracle integration         |
| `rentable::wrapper` | Mints **RentalNFT** object with `expires_at` field |
| `rentable::admin`   | Pause, fee config                                  |

Resources:

* `Listing { owner, nft_id, price_per_day, max_duration }`
* `RentalNFT { original_id, expires_at }` – **object** in Sui world.

#### Events

`Listed`, `Rented`, `Returned`, `ExpiredAutoReturn`.

### Aptos vs. Sui Notes

* Use **object wrapping/unwrapping** on Sui for automatic recall.
* On Aptos, represent leases via `Table<address, Lease>` and withdraw rights via capability token.

### Security Considerations

* Prevent double‑wrap.
* Validate metadata immutability.

### Testing Matrix

* Time warp tests for expiry.
* Fuzz price calculations.

### Extensions

* Revenue‑sharing between lender & creator (royalties).
* Reputation scores for renters.

---

<a name="3-movedao"></a>

## 3. **MoveDAO – Modular On‑Chain Governance & Treasury**

#### Suggested GitHub Repo Name

`move-dao-governance-framework`

### Vision & Goals

Deliver a composable DAO framework (token‑weighted or NFT membership) with proposal modules, timelock, and asset treasury that can govern other Move modules via upgrade capabilities.

### Functional Modules

1. **Token**: Optional governance token mint.
2. **Treasury**: Holds multi‑asset vault.
3. **Governor**: Creates & executes proposals.
4. **Strategy**: Pluggable voting logic (Simple Majority, Quadratic, etc.).

### Proposal Lifecycle

`Draft → Active → Succeeded → Queued (Timelock) → Executed / Vetoed`.

### Move Design Highlights

* Each DAO is a **resource** `DAO { id, config, treasury, governor_cap }`.
* Delegation map `Table<address, u64>` for vote power.

### Differences Aptos vs. Sui

* Sui’s **dynamic fields** suit delegation maps.
* Aptos’ **multi‑agent txns** enable proposal batching.

### Security & Auditing Checklist

* Timelock cannot be bypassed.
* Voting period difference between chains.
* Upgrade authority stored inside DAO itself.

### Testnets & DevOps

* CLI script to spin‑up a local DAO in 30 seconds.
* End‑to‑end governance flow.

---

<a name="4-nodestaker"></a>

## 4. **NodeStaker – DePIN Sensor Staking & Reward Module**

#### Suggested GitHub Repo Name

`move-depin-node-staker`

### Vision & Goals

Power decentralized physical network (e.g., air‑quality sensors) by letting node operators stake tokens that unlock data submission rights and earn emission rewards based on data proofs.

### Actors & Flows

1. **Operator** stakes tokens → receives `ValidatorCap` resource.
2. Submits signed data proof with epoch nonce.
3. Epoch Rewards distributed proportionally to proof quality score.
4. Slashing if invalid data or downtime.

### Contract Components

* `staking::pool` – lock/unlock stake, epoch accounting.
* `oracle::submit` – verify data signatures.
* `rewards::distributor` – mint or transfer rewards.

### Move‑Specific Tricks

* Use `vector` of epoch records inside `Stake` resource.
* Implement slashing via burning stake resource.

### Aptos vs. Sui Specifics

* Aptos: Use on‑chain `ed25519` verify & event index.
* Sui: Leverage **Move events + off‑chain indexer** for scoring.

### KPIs & Metrics

* Stake TVL, data proofs per day, slash events.

### Educational Value

* Practice capability tokens, epoch accounting, burn/mint patterns.

---

<a name="5-movebridge"></a>

## 5. **MoveBridge – Cross‑Network Liquidity & Message Bridge**

#### Suggested GitHub Repo Name

`move-aptos-sui-bridge`

### Vision & Goals

Bridge fungible assets and arbitrary messages between Aptos and Sui testnets/mainnets using light‑client verification and relayer network.

### Core Mechanics

1. **Lock & Mint**: Lock tokens on source chain, mint wrapped tokens on destination.
2. **Burn & Release**: Burn wrapped tokens to unlock originals.
3. **Generic Message Passing**: Relay bytes payload with proof.

### Contract Breakdown

| Module                | Chain | Function                       |
| --------------------- | ----- | ------------------------------ |
| `bridge::vault`       | both  | Escrow & release               |
| `bridge::mint`        | dest  | Mint wrapped coin              |
| `bridge::lightclient` | both  | Verify headers & Merkle proofs |

### Proof System

* Use **Merkle Accumulator** from Move stdlib.
* Off‑chain relayers supply proof.

### Security Model

* 2‑of‑3 multisig of independent relayers initially.
* Upgrade to zk‑proof once prover available.

### Testing Plan

* Localnet docker compose for dual chains.
* Stress test with 10k tx batch.

### Learning Outcomes

* Cross‑chain data structures, signature verification, Merkle proofs.

---

---

<a name="6-moveetf"></a>

## 6. **MoveETF – On‑Chain Tokenized Index Fund**

#### Suggested GitHub Repo Name

`move-etf-index-fund`

### Vision & Goals

Enable anyone to mint and redeem a single fungible token that represents a diversified basket of underlying Move‑based assets on Aptos or Sui—similar to an ETF.  The protocol automatically rebalances holdings according to a chosen index methodology.

### Personas & Use Cases

| Persona           | Need                                | Example                      |
| ----------------- | ----------------------------------- | ---------------------------- |
| Retail user       | Simple diversified exposure         | Buy APT‑DeFi Index token     |
| DAO treasury      | Park funds in low‑maintenance index | 30% APT, 40% stable, 30% RWA |
| Portfolio manager | Offer branded index products        | Launch "AI Infra Index"      |

### Core Features

1. **MintShares**: Deposit the precise proportion of each constituent asset to receive ETF shares.
2. **RedeemShares**: Burn shares to withdraw underlying assets pro‑rata.
3. **Rebalance**: Anyone can call with new weights; executes swaps via integrated DEX; incentivized via small fee rebate.
4. **FeeCollection**: Streaming protocol fee (bps per second) flows to manager.
5. **Pause / Emergency Exit**: Safety switches for extreme events.

### Smart‑Contract Architecture

| Module             | Responsibility                                            |
| ------------------ | --------------------------------------------------------- |
| `etf::vault`       | Stores basket reserves, validates mint/redeem             |
| `etf::share_token` | Implements ETF share as fungible `Coin`/`Sui::coin::Coin` |
| `etf::rebalance`   | Weight oracle, DEX router calls                           |
| `etf::admin`       | Set fees, approve new constituents                        |

#### Key Resources & Types

* `Vault { id, constituents: vector<CoinType>, weights: vector<u64>, total_shares }`
* `struct RebalanceRequest { new_weights, deadline }` event‑driven.

### Aptos vs. Sui Specific Notes

| Aspect              | Aptos                                      | Sui                                            |
| ------------------- | ------------------------------------------ | ---------------------------------------------- |
| Share token         | Standard `Coin` implementing `store + key` | Native `Sui::coin::Coin<T>` object             |
| Rebalance execution | Multi‑agent txn calling `aptoswap`         | Object‑centric DEX call e.g. `DeepBook`        |
| Storage             | On‑chain table of constituent reserves     | Individual **objects** per asset held in vault |

### Invariants & Security

* Vault reserve balance equals aggregate deposits minus withdrawals.
* No share inflation beyond `total_shares`.
* Rebalance must conserve total value within slippage tolerance.

### Events

`Mint`, `Redeem`, `RebalanceExecuted`, `FeesCollected`.

### Testing & Audit Checklist

* Property test: NAV before = NAV after ± slippage.
* Fuzz mint/redeem with random weights.
* Simulate oracle manipulation attempts.

### Gas Targets

* `mint_shares` ≤ 80k gas.
* `rebalance` linear in `constituents`; ≤ 200k gas for 10 assets.

### Educational Value

* Practice complex resource accounting, DEX integration, oracle usage, and fee design.

---

## How to Use These PRDs

1. **Pick one project** that excites you.
2. Use the Smart‑Contract Architecture section as a coding road‑map.
3. Read the Aptos/Sui notes to understand chain‑specific tweaks.
4. Write unit tests first; reference the Testing Plan.
5. Iterate → Deploy to devnet → Seek feedback.

Happy building & mastering Move! 🚀
