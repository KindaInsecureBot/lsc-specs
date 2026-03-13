# LSC — System Architecture

## 1. Programs Overview

LSC is composed of **10 programs**. Each is stateless — all state lives in accounts passed as inputs. Programs communicate via chained calls (tail calls) to the Token Program and AMM Program.

| Program ID Alias | Binary Name | Responsibility |
|---|---|---|
| `LSC_ENGINE_ID` | `lsc_engine` | Core SAFE management |
| `TAX_COLLECTOR_ID` | `tax_collector` | Stability fee accrual |
| `ORACLE_RELAYER_ID` | `oracle_relayer` | Redemption price + rate |
| `PI_CONTROLLER_ID` | `pi_controller` | Feedback rate computation |
| `LIQUIDATION_ENGINE_ID` | `liquidation_engine` | Undercollateralization detection + liquidation |
| `COLLATERAL_AUCTION_ID` | `collateral_auction_house` | Sell collateral for LSC |
| `ACCOUNTING_ENGINE_ID` | `accounting_engine` | Surplus routing; bad debt queuing and offsetting |
| `SURPLUS_AUCTION_ID` | `surplus_auction_house` | Surplus LSC auctions |
| `ORACLE_PROGRAM_ID` | `oracle_program` | External price feed aggregation |
| `GLOBAL_SETTLEMENT_ID` | `global_settlement` | Emergency shutdown |

Additionally, LSC relies on two **existing** programs:
- `TOKEN_PROGRAM_ID` — Token Program (mint, burn, transfer LSC and LOGOS)
- `AMM_PROGRAM_ID` — AMM Program (LOGOS/LSC pool for market price)

---

## 2. Account Ownership Graph

```
LSC_ENGINE_ID owns:
  ├── SystemParamsAccount (PDA: ["system_params"])
  ├── CollateralTypeAccount (PDA: ["collateral_type", collateral_token_def_id])
  ├── SafeAccount (PDA: ["safe", safe_id])          -- one per SAFE
  ├── CollateralVaultAccount (PDA: ["vault", collateral_type_id])
  └── GlobalDebtAccount (PDA: ["global_debt"])

ORACLE_RELAYER_ID owns:
  ├── OracleRelayerParamsAccount (PDA: ["oracle_relayer_params"])
  └── (reads CollateralOraclePriceAccount and LscMarketPriceAccount)

PI_CONTROLLER_ID owns:
  └── PIControllerStateAccount (PDA: ["pi_controller_state"])

TAX_COLLECTOR_ID owns:
  └── TaxCollectorParamsAccount (PDA: ["tax_collector_params"])
      -- accumulated_rate lives in CollateralTypeAccount (owned by LSC_ENGINE_ID)
      -- tax collector is authorized to update it

LIQUIDATION_ENGINE_ID owns:
  ├── LiquidationEngineParamsAccount (PDA: ["liquidation_engine_params"])
  └── LiquidationAccount (PDA: ["liquidation", safe_id])

COLLATERAL_AUCTION_ID owns:
  ├── CollateralAuctionHouseParamsAccount (PDA: ["collateral_auction_params"])
  └── CollateralAuctionAccount (PDA: ["collateral_auction", auction_id])

ACCOUNTING_ENGINE_ID owns:
  ├── AccountingEngineParamsAccount (PDA: ["accounting_engine_params"])
  ├── SystemSurplusAccount (PDA: ["system_surplus"])  -- holds protocol LSC balance
  └── SystemDebtQueueAccount (PDA: ["debt_queue"])

SURPLUS_AUCTION_ID owns:
  └── SurplusAuctionAccount (PDA: ["surplus_auction", auction_id])

ORACLE_PROGRAM_ID owns:
  ├── OracleFeedAccount (PDA: ["feed", feed_id])     -- one per price source
  ├── MedianOracleAccount (PDA: ["median", symbol])  -- aggregated price
  └── OracleConfigAccount (PDA: ["oracle_config"])

GLOBAL_SETTLEMENT_ID owns:
  ├── GlobalSettlementStateAccount (PDA: ["settlement_state"])
  └── CollateralRedemptionAccount (PDA: ["redemption", collateral_type_id])

TOKEN_PROGRAM_ID owns:
  ├── LscTokenDefinitionAccount (PDA of Token Program)
  ├── LogosTokenDefinitionAccount (pre-existing)
  ├── UserLscHoldingAccount (user-held)
  └── UserLogosHoldingAccount (user-held)
```

---

## 3. PDA Derivation

All PDAs are computed as: `program.derive_account(seeds)` where seeds are concatenated bytes.

### LSC Engine PDAs

```
system_params_id        = LSC_ENGINE_ID.derive(b"system_params")
collateral_type_id      = LSC_ENGINE_ID.derive(b"collateral_type" || collateral_token_def_id[0..32])
safe_id                 = LSC_ENGINE_ID.derive(b"safe" || owner_account_id[0..32] || nonce[0..8])
collateral_vault_id     = LSC_ENGINE_ID.derive(b"vault" || collateral_type_id[0..32])
global_debt_id          = LSC_ENGINE_ID.derive(b"global_debt")
```

### Oracle Relayer PDAs

```
oracle_relayer_params_id = ORACLE_RELAYER_ID.derive(b"oracle_relayer_params")
```

### PI Controller PDAs

```
pi_controller_state_id  = PI_CONTROLLER_ID.derive(b"pi_controller_state")
```

### Tax Collector PDAs

```
tax_collector_params_id = TAX_COLLECTOR_ID.derive(b"tax_collector_params")
```

### Liquidation Engine PDAs

```
liquidation_params_id   = LIQUIDATION_ENGINE_ID.derive(b"liquidation_engine_params")
liquidation_id          = LIQUIDATION_ENGINE_ID.derive(b"liquidation" || safe_id[0..32])
```

### Collateral Auction PDAs

```
collateral_auction_params_id = COLLATERAL_AUCTION_ID.derive(b"collateral_auction_params")
collateral_auction_id        = COLLATERAL_AUCTION_ID.derive(b"collateral_auction" || auction_nonce[0..8])
```

### Accounting Engine PDAs

```
accounting_params_id    = ACCOUNTING_ENGINE_ID.derive(b"accounting_engine_params")
system_surplus_id       = ACCOUNTING_ENGINE_ID.derive(b"system_surplus")
debt_queue_id           = ACCOUNTING_ENGINE_ID.derive(b"debt_queue")
```

### Surplus Auction PDAs

```
surplus_auction_id      = SURPLUS_AUCTION_ID.derive(b"surplus_auction" || auction_nonce[0..8])
```

### Oracle Program PDAs

```
oracle_feed_id          = ORACLE_PROGRAM_ID.derive(b"feed" || feed_id[0..8])
median_oracle_id        = ORACLE_PROGRAM_ID.derive(b"median" || symbol[0..16])
oracle_config_id        = ORACLE_PROGRAM_ID.derive(b"oracle_config")
```

### Global Settlement PDAs

```
settlement_state_id          = GLOBAL_SETTLEMENT_ID.derive(b"settlement_state")
collateral_redemption_id     = GLOBAL_SETTLEMENT_ID.derive(b"redemption" || collateral_type_id[0..32])
```

---

## 4. Program Interaction Diagram

### Normal Operation Flow

```
User
 │
 ├──GenerateDebt──► LSCEngine
 │                     │
 │                     ├── reads: CollateralTypeAccount (accumulated_rate, safety_ratio)
 │                     ├── reads: OracleRelayerParams (redemption_price)
 │                     ├── reads: MedianOracleAccount (collateral_price)
 │                     ├── mutates: SafeAccount (debt ↑)
 │                     ├── mutates: CollateralTypeAccount (global_debt ↑)
 │                     └── chained_call ──► TokenProgram::Mint(lsc_amount)
 │                                            └── mutates: UserLscHolding (balance ↑)
 │
 ├──AccrueStabilityFee──► TaxCollector
 │                            │
 │                            ├── reads: TaxCollectorParams (stability_fee, last_update)
 │                            ├── reads: OracleTimestampAccount (current_time)
 │                            └── mutates: CollateralTypeAccount (accumulated_rate ↑)
 │
 └──UpdateRedemptionRate──► PIController
                                │
                                ├── reads: MedianOracleAccount (lsc_market_price)
                                ├── reads: OracleRelayerParams (redemption_price)
                                ├── mutates: PIControllerState (cumulative_error ↑/↓)
                                └── mutates: OracleRelayerParams (redemption_rate, last_redemption_price_update)
```

### Liquidation Flow

```
Liquidator
 │
 └──LiquidateSAFE──► LiquidationEngine
                          │
                          ├── reads: SafeAccount (collateral, debt)
                          ├── reads: CollateralTypeAccount (accumulated_rate, liquidation_ratio)
                          ├── reads: MedianOracleAccount (collateral_price)
                          ├── reads: OracleRelayerParams (redemption_price)
                          ├── mutates: SafeAccount (collateral → 0, debt → 0)
                          ├── mutates: CollateralTypeAccount (global_debt ↓)
                          ├── chained_call ──► LSCEngine::ConfiscateSAFE(...)
                          │                       └── transfers collateral to auction vault
                          ├── chained_call ──► AccountingEngine::PushDebt(debt_amount)
                          │                       └── mutates: SystemDebtQueue
                          └── chained_call ──► CollateralAuctionHouse::StartAuction(...)
                                                  └── creates CollateralAuctionAccount
```

### Auction Flow (Collateral)

```
Buyer
 │
 └──BuyCollateral──► CollateralAuctionHouse
                         │
                         ├── reads: CollateralAuctionAccount (collateral_amount, debt_target)
                         ├── reads: MedianOracleAccount (collateral_price, lsc_market_price)
                         ├── reads: OracleRelayerParams (redemption_price)
                         ├── computes: discount_price
                         ├── mutates: CollateralAuctionAccount (remaining_collateral ↓)
                         ├── chained_call ──► TokenProgram::Transfer(lsc, buyer → accounting_engine)
                         └── chained_call ──► TokenProgram::Transfer(logos, vault → buyer)
```

---

## 5. Authorization Model

| Account Type | Who Can Authorize |
|---|---|
| SafeAccount | The SAFE owner (is_authorized on owner's account) |
| CollateralTypeAccount | LSC Engine (PDA authority) or Governance |
| SystemParamsAccount | Governance only |
| OracleRelayerParams | Oracle Relayer (PDA) for rate updates; Governance for param changes |
| PIControllerState | PI Controller program (PDA) |
| TaxCollectorParams | Tax Collector (PDA) for rate update; Governance for param changes |
| LiquidationEngineParams | Governance |
| CollateralAuctionHouseParams | Governance |
| AccountingEngineParams | Governance |
| OracleFeedAccount | Designated oracle keeper (authorized submitter) |
| MedianOracleAccount | Oracle Program (PDA) |
| GlobalSettlementState | Governance (one-time trigger) |

**Governance** in v1 is a single multisig account (governance_account_id stored in SystemParamsAccount). Governance can be upgraded to an on-chain voting program later.

---

## 6. Data Flow: Timestamps

LEZ programs cannot read block timestamps. All time-dependent operations receive a timestamp from an **Oracle-provided timestamp**:

1. `OracleFeedAccount` stores `timestamp: u64` (unix seconds) submitted by authorized keepers.
2. The `MedianOracleAccount` exposes the median timestamp from all feeds.
3. All programs that need time (TaxCollector, PIController, OracleRelayer, CollateralAuctionHouse) receive a `MedianOracleAccount` as input and read its `timestamp` field.
4. Programs validate that `timestamp > last_update_time` and `timestamp <= last_update_time + max_allowed_lag` before accepting it.

This creates a dependency: **price feeds must be updated before any time-sensitive operation can proceed**.

---

## 7. LSC Token Architecture

LSC is a **fungible token** managed by the Token Program:

```
LscTokenDefinitionAccount:
  - owned by: TOKEN_PROGRAM_ID
  - definition_id: LSC_TOKEN_DEF_ID (constant, set at deployment)
  - minting_authority: system_surplus_id (LSC Engine PDA)
  - total_supply: tracked by Token Program
```

The LSC Engine's `system_params_id` PDA is the authorized minting authority. Only LSCEngine can invoke `TokenProgram::Mint` for LSC (because the LSC token definition's `is_authorized` flag is set only when the calling chain includes LSCEngine as the minting authority).

LOGOS is a **pre-existing token** with its own token definition. No LSC program holds minting authority over LOGOS — the LSC system does not mint new LOGOS under any circumstances.

---

## 8. Initialization Sequence

Deployment must happen in this order:

```
1. Deploy all LSC programs (upload binaries)
2. TOKEN_PROGRAM::NewFungibleDefinition → creates LscTokenDefinitionAccount
   (minting authority = system_params_id PDA of LSCEngine)
3. LSCEngine::Initialize(params) → creates SystemParamsAccount, GlobalDebtAccount
4. LSCEngine::AddCollateralType(LOGOS_TOKEN_DEF_ID, params) → creates CollateralTypeAccount
5. TaxCollector::Initialize(params) → creates TaxCollectorParamsAccount
6. OracleProgram::Initialize → creates OracleConfigAccount
7. OracleProgram::AddFeed(collateral) → creates OracleFeedAccount for LOGOS/USD
8. OracleProgram::AddFeed(lsc_market) → creates OracleFeedAccount for LSC/USD
9. OracleRelayer::Initialize(params) → creates OracleRelayerParamsAccount
   (initial redemption_price = 3.14 * RAY, redemption_rate = RAY)
10. PIController::Initialize(params) → creates PIControllerStateAccount
11. LiquidationEngine::Initialize(params) → creates LiquidationEngineParamsAccount
12. CollateralAuctionHouse::Initialize(params) → creates CollateralAuctionHouseParamsAccount
13. AccountingEngine::Initialize(params) → creates AccountingEngineParamsAccount, SystemSurplusAccount, SystemDebtQueueAccount
14. SurplusAuctionHouse::Initialize(params) → creates (no state; auctions created on demand)
15. GlobalSettlement::Initialize → creates GlobalSettlementStateAccount (active = false)
16. AMM_PROGRAM::NewDefinition → creates LOGOS/LSC pool
```

---

## 9. Upgrade and Governance

In v1, governance is a multisig account (`governance_id`). Governance can:
- Change system parameters (safety ratio, debt ceiling, stability fee, PI gains, etc.)
- Add new collateral types
- Trigger global settlement
- Upgrade program binaries (LEZ native mechanism)
- Grant/revoke oracle keeper authorizations

Parameter changes take effect immediately (no timelock in v1; timelock recommended for v2).
