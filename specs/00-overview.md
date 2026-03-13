# LSC — Logos Stable Coin: Overview

## 1. What Is LSC?

LSC (Logos Stable Coin) is a **reflexive, non-pegged stablecoin** native to the Logos Execution Zone (LEZ) blockchain. It is backed by LOGOS token collateral and stabilized by an autonomous proportional-integral (PI) feedback controller that continuously adjusts a *redemption rate*, incentivizing market participants to push the LSC market price toward an internal *redemption price*.

LSC is architecturally derived from [RAI / Reflexer Finance](https://reflexer.finance/) and adapted to LEZ's stateless, account-based programming model. Like RAI, LSC does **not** target a fixed USD peg. Instead, the redemption price floats, and the market price is kept close to it by economic incentives alone — without governance-driven peg adjustments.

---

## 2. Core Concepts

| Term | Definition |
|---|---|
| **SAFE** | A collateralized debt position (CDP). Users lock LOGOS collateral and draw LSC debt. |
| **Collateral** | LOGOS tokens locked inside a SAFE. |
| **LSC** | The stablecoin minted against SAFE collateral. The `Mint` instruction of the Token Program is called by the LSC Engine. |
| **Redemption Price** | The internal price at which the protocol values LSC for accounting purposes (debt generation, liquidation). Starts at a chosen initial value and drifts over time. |
| **Market Price** | The current trading price of LSC, derived from the AMM pool (LOGOS/LSC). |
| **Redemption Rate** | A per-second interest rate (positive or negative) applied to the redemption price. When market < redemption, the rate goes positive, making LSC debt more expensive and pushing price up. When market > redemption, the rate goes negative. |
| **Stability Fee** | An additional interest rate on SAFE debt, separate from (and additive to) the redemption rate. Accrues in the global `accumulated_rate` accumulator. |
| **Liquidation** | Forced closure of an undercollateralized SAFE. Collateral is sold via fixed-discount auction to cover debt + penalty. |
| **Surplus** | Protocol-owned LSC accumulated from stability fees. Auctioned off for LOGOS which is then burned. |
| **Bad Debt** | Uncovered LSC debt after a failed liquidation. Queued in the AccountingEngine and offset against future surplus. If unresolved, it remains as a protocol liability socialized across LSC holders via the redemption price mechanism. |
| **Global Settlement** | An emergency shutdown mechanism that freezes the system and allows all SAFE owners to redeem collateral. |
| **PDA** | Program-Derived Account — an account whose ID is deterministically derived from a program's seed space. |
| **Ray** | Fixed-point number with 27 decimal places (10^27 = 1.0). Used for rates and price accumulators. |
| **Wad** | Fixed-point number with 18 decimal places (10^18 = 1.0). Used for token amounts and prices. |

---

## 3. High-Level Architecture

```
                          ┌─────────────────────────────────────┐
                          │            LEZ Blockchain            │
                          └─────────────────────────────────────┘
                                         │
          ┌──────────────────────────────┼──────────────────────────────┐
          │                              │                              │
   ┌──────▼──────┐              ┌────────▼───────┐            ┌────────▼───────┐
   │ LSC Engine  │◄────────────►│ Oracle Program │            │  Token Program │
   │ (Core CDP)  │              │ (Price Feeds)  │            │  (LSC Token)   │
   └──────┬──────┘              └────────┬───────┘            └────────────────┘
          │                              │
          │  chained calls               │  reads
          │                              │
   ┌──────▼──────┐              ┌────────▼───────┐
   │ Tax         │              │  PI Controller  │
   │ Collector   │              │  Program        │
   └──────┬──────┘              └────────┬───────┘
          │                              │
          │  surplus/deficit             │  writes redemption rate
          │                              │
   ┌──────▼──────┐              ┌────────▼───────┐
   │ Accounting  │              │ Oracle Relayer │
   │ Engine      │              │ (Redemption    │
   └──────┬──────┘              │  Price/Rate)   │
          │                     └────────────────┘
          │
   ┌──────▼──────┐
   │   Surplus   │
   │   Auction   │
   └─────────────┘

   ┌──────────────┐       ┌──────────────┐
   │ Liquidation  │       │  AMM Program │
   │ Engine       │◄─────►│ (LSC/LOGOS   │
   └──────┬───────┘       │   Pool)      │
          │               └──────────────┘
   ┌──────▼──────┐
   │  Collateral │
   │   Auction   │
   └─────────────┘
                          ┌──────────────┐
                          │   Global     │
                          │  Settlement  │
                          └──────────────┘
```

### Program List

| Program | Role |
|---|---|
| `LSCEngine` | Core SAFE management: open/close SAFEs, deposit/withdraw collateral, mint/burn LSC |
| `TaxCollector` | Accrues stability fees on SAFE debt; updates `accumulated_rate` |
| `OracleRelayer` | Stores and updates redemption price and redemption rate |
| `PIController` | Computes new redemption rate from market/redemption price error |
| `LiquidationEngine` | Identifies and liquidates undercollateralized SAFEs |
| `CollateralAuctionHouse` | Sells confiscated collateral via fixed-discount auction |
| `AccountingEngine` | Routes surplus to surplus auction; queues and offsets bad debt |
| `SurplusAuctionHouse` | Accepts LSC bids; sells surplus LSC for LOGOS (burn) |
| `OracleProgram` | Aggregates external price feeds; publishes collateral and LSC market prices |
| `GlobalSettlement` | Emergency shutdown: freezes system, enables collateral redemption |

All programs interact with the existing **Token Program** (for LSC minting/burning/transfer) and **AMM Program** (for market price discovery).

---

## 4. Comparison to RAI

| Aspect | RAI (Ethereum) | LSC (LEZ) |
|---|---|---|
| Collateral | ETH | LOGOS token |
| Smart contract model | Stateful Solidity | Stateless LEZ programs |
| Storage | Contract storage (mappings) | Individual accounts (borsh-serialized) |
| Authorization | `msg.sender` | `is_authorized` flag on account |
| Token standard | ERC-20 (internal) | Token Program (chained calls) |
| Timestamps | `block.timestamp` | Oracle-provided timestamps |
| Price oracle | Uniswap v2 TWAP + Chainlink | AMM Program + external feed aggregator |
| Auctions | English auction / fixed-discount | Fixed-discount (adapted for LEZ) |
| Mappings | Solidity mapping | Per-SAFE PDA accounts |
| Events | Solidity events | Off-chain indexer reads account state |
| Governance | FLX token, timelocked | LOGOS governance (out of scope for v1) |
| Privacy | None (fully public) | Partial: SAFE state can be private |

The key innovation — the **PI controller** — is directly ported from RAI's mathematical model with no fundamental changes, only adaptation to LEZ's timestamp model.

---

## 5. User Lifecycle

```
1. User calls LSCEngine::OpenSafe
   → PDA SAFE account created, owned by user

2. User calls LSCEngine::DepositCollateral(amount)
   → Token Program::Transfer called (LOGOS → SAFE vault)
   → SAFE.collateral increases

3. User calls LSCEngine::GenerateDebt(amount)
   → Validates collateralization ratio
   → Token Program::Mint called (LSC → user's LSC account)
   → SAFE.debt increases, global debt increases

4. Over time: TaxCollector::AccrueStabilityFee
   → accumulated_rate increases
   → Effective debt (debt * accumulated_rate) grows

5a. If SAFE stays healthy: User repays debt + withdraws collateral

5b. If SAFE becomes undercollateralized:
   → LiquidationEngine::LiquidateSAFE
   → CollateralAuctionHouse auction created
   → Buyer bids LSC, receives LOGOS at discount
   → Remaining debt goes to AccountingEngine

6. AccountingEngine::AuctionSurplus (when protocol has LSC surplus)
   → SurplusAuctionHouse: users bid LOGOS to receive surplus LSC
   → LOGOS burned

6b. If bad debt remains after offsetting against surplus:
   → Bad debt persists as a protocol liability in the debt queue
   → Socialized across LSC holders via the redemption price mechanism
   → Governance may decide on external recapitalization (out of scope for v1)
```

---

## 6. Glossary

| Term | Definition |
|---|---|
| `accumulated_rate` | Global multiplier (ray) tracking compounded stability fees + redemption rate. SAFE effective debt = `debt * accumulated_rate`. |
| `collateral_type` | Identifier for a class of collateral (v1: only `LOGOS`). |
| `safety_ratio` | Minimum collateralization ratio for generating debt (e.g. 150%). |
| `liquidation_ratio` | Minimum collateralization ratio before liquidation (may equal safety_ratio). |
| `liquidation_penalty` | Extra collateral seized on liquidation (e.g. 13%). |
| `debt_ceiling` | Maximum total LSC that can be minted globally. |
| `debt_floor` | Minimum debt per SAFE (dust protection). |
| `stability_fee` | Per-second interest rate on SAFE debt. Expressed as a ray (e.g. 1.000000000315522921573 ≈ 1% APY). |
| `redemption_price` | Protocol's internal target price for LSC (ray, denominated in USD). |
| `redemption_rate` | Per-second multiplier applied to redemption_price (ray). |
| `market_price` | LSC market price from AMM oracle (wad, USD). |
| `Kp` | PI controller proportional gain. |
| `Ki` | PI controller integral gain. |
| `global_debt` | Total LSC debt outstanding across all SAFEs (wad). |
| `global_surplus` | Total LSC surplus held by AccountingEngine (wad). |
| `ray` | 10^27 fixed-point unit. |
| `wad` | 10^18 fixed-point unit. |
| `rad` | 10^45 fixed-point unit (ray * wad, used for debt accounting). |
