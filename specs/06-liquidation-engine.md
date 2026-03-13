# LSC — Liquidation Engine

## Overview

The Liquidation Engine identifies and liquidates undercollateralized SAFEs. When a SAFE's collateral value falls below the `liquidation_ratio` threshold, any account can trigger liquidation. The seized collateral is sold via the **CollateralAuctionHouse** using a fixed-discount mechanism to repay the SAFE's debt plus a liquidation penalty.

**Program ID aliases:** `LIQUIDATION_ENGINE_ID`, `COLLATERAL_AUCTION_ID`

---

## 1. Liquidation Trigger Condition

A SAFE is liquidatable when:

```
effective_debt_rad = safe.normalized_debt * collateral_type.accumulated_rate

collateral_value_rad = (safe.collateral * collateral_price_wad * RAY) / redemption_price_ray
// where collateral_price is the SAFETY-MARGINED oracle price

liquidatable = collateral_value_rad * RAY < effective_debt_rad * liquidation_ratio
```

Equivalently: **collateral ratio < liquidation ratio**.

```
CR = (safe.collateral * collateral_price_wad) / (effective_debt_rad / RAY)
   < liquidation_ratio / RAY
```

Note: The **collateral price used for liquidation** is:
```
liquidation_price = median_price * collateral_price_safety_margin / RAY
```
Not the raw oracle price (reduced by safety margin).

---

## 2. LiquidationEngine Program

### 2.1 Instruction Enum

```rust
enum LiquidationEngineInstruction {
    /// One-time initialization
    Initialize {
        lsc_engine_params_id: [u8; 32],
        collateral_auction_params_id: [u8; 32],
        accounting_engine_params_id: [u8; 32],
        oracle_relayer_params_id: [u8; 32],
        on_auction_system_coin_limit: u128,   // Rad
    },

    /// Liquidate an undercollateralized SAFE
    LiquidateSafe,

    /// Callback: reduce current_on_auction_system_coins when auction ends
    RemoveLiquidation {
        auction_id: [u8; 32],
    },

    /// Update parameters (governance)
    UpdateParams {
        new_on_auction_system_coin_limit: Option<u128>,
    },
}
```

---

### 2.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `liquidation_engine_params_id` (PDA) | Yes | PDA (new) | — |
| 1 | `governance_account` | No | Yes | Initial governance |

**State Transition:**
```
liquidation_params.account_type = 40
liquidation_params.lsc_engine_params_id = lsc_engine_params_id
liquidation_params.collateral_auction_params_id = collateral_auction_params_id
liquidation_params.accounting_engine_params_id = accounting_engine_params_id
liquidation_params.oracle_relayer_params_id = oracle_relayer_params_id
liquidation_params.on_auction_system_coin_limit = on_auction_system_coin_limit
liquidation_params.current_on_auction_system_coins = 0
```

---

### 2.3 `LiquidateSafe`

**Description:** Called by any liquidator (permissionless) to liquidate an underwater SAFE. Seizes the SAFE's collateral, clears its debt, and starts a collateral auction.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `liquidation_engine_params_id` | Yes | — | Liquidation params (circuit breaker) |
| 1 | `safe_id` | Yes | — | The SAFE to liquidate |
| 2 | `collateral_type_id` | Yes | — | Collateral type |
| 3 | `oracle_relayer_params_id` | No | — | For redemption_price |
| 4 | `collateral_oracle_id` | No | — | LOGOS/USD median oracle |
| 5 | `system_params_id` | No | — | For settlement_active check |
| 6 | `global_debt_id` | Yes | — | Global debt tracking |
| 7 | `collateral_vault_id` | Yes | PDA (LSCEngine) | SAFE's collateral vault |
| 8 | `auction_collateral_vault_id` | Yes | PDA (CollateralAuction) | Auction vault |
| 9 | `accounting_engine_surplus_id` | No | — | Where surplus debt goes |
| 10 | `liquidation_account_id` (PDA) | Yes | PDA (new) | Liquidation record |
| 11 | `collateral_auction_params_id` | Yes | — | Auction house params |
| 12 | `new_auction_id` (PDA) | Yes | PDA (new) | The auction account to create |
| 13 | `liquidator_account` | No | Yes | Transaction signer (receives liquidation reward) |

**Validations:**
```
// 1. Settlement not active
assert!(!system_params.settlement_active, SettlementActive);

// 2. SAFE not already liquidated
assert!(!safe.liquidated, AlreadyLiquidated);

// 3. Oracle valid and fresh
assert!(collateral_oracle.valid, OracleInvalid);
let liquidation_price = collateral_oracle.median_price
    * oracle_relayer_params.collateral_price_safety_margin / RAY;

// 4. Check if SAFE is actually undercollateralized
let effective_debt_rad = safe.normalized_debt * collateral_type.accumulated_rate;
let collateral_value_rad = collateral_value_in_rad(
    safe.collateral, liquidation_price, oracle_relayer_params.redemption_price
);
assert!(
    collateral_value_rad * RAY < effective_debt_rad * collateral_type.liquidation_ratio,
    SafeNotLiquidatable
);

// 5. Circuit breaker: don't exceed total on-auction limit
let debt_to_cover_rad = effective_debt_rad
    + effective_debt_rad * collateral_type.liquidation_penalty / WAD;
assert!(
    liquidation_params.current_on_auction_system_coins + debt_to_cover_rad
    <= liquidation_params.on_auction_system_coin_limit,
    OnAuctionLimitReached
);
```

**Computation:**
```
// Amount of collateral to seize
// = collateral (all of it for full liquidation)
collateral_to_seize = safe.collateral

// Debt to cover (with penalty)
// actual_debt = normalized_debt * accumulated_rate (in Rad)
actual_debt_rad = safe.normalized_debt * collateral_type.accumulated_rate

// Amount to raise at auction = debt + liquidation_penalty * debt
amount_to_raise_rad = actual_debt_rad + (actual_debt_rad * collateral_type.liquidation_penalty / WAD)

// Normalize debt to remove from SAFE
debt_to_remove_normalized = safe.normalized_debt
```

**State Transitions:**
```
// Update circuit breaker
liquidation_params.current_on_auction_system_coins += debt_to_cover_rad

// Liquidation record
liquidation_account.account_type = 41
liquidation_account.safe_id = safe_id
liquidation_account.auction_id = new_auction_id
liquidation_account.collateral_seized = collateral_to_seize
liquidation_account.debt_to_cover = amount_to_raise_rad
liquidation_account.liquidated_at = current_timestamp
```

**Chained Calls (in order):**

```
// 1. Confiscate the SAFE (clear debt and collateral from SAFE)
LSCEngine::ConfiscateSafe {
    collateral_amount: collateral_to_seize,
    debt_amount: debt_to_remove_normalized,
    // accounts: [safe_id, collateral_type_id, global_debt_id, system_params_id,
    //            liquidation_engine_params_id (auth), collateral_vault_id, auction_collateral_vault_id]
}

// 2. Push debt to accounting engine (bad debt = amount_to_raise - debt_to_remove)
// (debt_to_remove already accounts for penalty; the "deficit" goes to accounting engine)
AccountingEngine::PushDebt {
    debt_amount: actual_debt_rad,
    // accounts: [accounting_engine_params_id, debt_queue_id (auth)]
}

// 3. Start the collateral auction
CollateralAuctionHouse::StartAuction {
    collateral_to_sell: collateral_to_seize,
    amount_to_raise: amount_to_raise_rad,
    safe_id: safe_id,
    collateral_type_id: collateral_type_id,
    // accounts: [collateral_auction_params_id, new_auction_id (auth), lsc_market_oracle_id]
}
```

**Note on chained call limit:** This sequence uses 4 chained calls (LiquidateSafe emits 3: ConfiscateSafe, PushDebt, StartAuction; ConfiscateSafe itself emits 1 more: Token::Transfer). Total tree depth = 4, well within LEZ limit of 10.

**Errors:**
- `SettlementActive`
- `AlreadyLiquidated`
- `OracleInvalid`
- `SafeNotLiquidatable` — SAFE is above liquidation ratio
- `OnAuctionLimitReached` — circuit breaker triggered
- `InvalidPDA`

---

## 3. CollateralAuctionHouse Program

The CollateralAuctionHouse implements a **fixed-discount auction**: buyers purchase collateral at a predetermined discount to the current oracle price. This is simpler and more capital-efficient than English auctions.

**Key property:** There is no bidding war. Buyers get collateral at `price * (1 - discount)`. First come, first served. The auction ends when all collateral is sold or the debt is fully raised.

### 3.1 Instruction Enum

```rust
enum CollateralAuctionInstruction {
    /// Initialize the auction house (one-time)
    Initialize {
        lsc_engine_params_id: [u8; 32],
        liquidation_engine_params_id: [u8; 32],
        oracle_relayer_params_id: [u8; 32],
        accounting_engine_surplus_id: [u8; 32],
        auction_discount: u128,       // Wad (e.g. 50_000_000_000_000_000 = 5%)
        max_discount: u128,           // Wad (hard maximum discount)
        min_discount: u128,           // Wad (hard minimum discount, usually 0)
        minimum_bid: u128,            // Wad (minimum collateral per purchase)
        auction_ttl: u64,             // seconds before auction can be restarted
        discount_increment: u128,     // Wad (added to discount per RestartAuction; e.g. 10_000_000_000_000_000 = 1%)
    },

    /// Start a new collateral auction (called by LiquidationEngine — privileged)
    StartAuction {
        collateral_to_sell: u128,     // Wad
        amount_to_raise: u128,        // Rad
        safe_id: [u8; 32],
        collateral_type_id: [u8; 32], // which collateral type is being auctioned
    },

    /// Buy collateral at the discounted price (anyone can call)
    BuyCollateral {
        /// Maximum amount of collateral to buy (Wad)
        /// (buyer can buy less if amount_to_raise is nearly met)
        collateral_amount: u128,
    },

    /// Settle an auction that has raised its target (cleanup)
    SettleAuction,

    /// Restart an expired auction with updated prices
    RestartAuction,

    /// Terminate an auction and return leftover collateral (governance, emergency)
    TerminateAuction,
}
```

---

### 3.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `collateral_auction_params_id` (PDA) | Yes | PDA (new) | — |
| 1 | `governance_account` | No | Yes | — |
| 2 | `auction_collateral_vault_id` (PDA) | Yes | PDA (new) | Vault for seized collateral |

**Validations:**
- `auction_discount <= max_discount`
- `min_discount <= auction_discount`
- `auction_ttl > 0`

**State Transition:**
```
auction_params.account_type = 50
auction_params.[all fields] = provided values  // including discount_increment
auction_params.auction_nonce = 0
```

**Chained Calls:**
```
// Initialize vault as Token Program holding account
// Account order: [definition_account (read), account_to_initialize (writable, new)]
// InitializeAccount takes no instruction parameters — positions determine behavior.
TokenProgram::InitializeAccount
  accounts: [logos_token_def_id (read), auction_collateral_vault_id (writable, new)]
  pda_seeds: [compute_pda_seed(b"auction_vault" || collateral_type_id)]
  // After this call: auction_collateral_vault_id.program_owner = TOKEN_PROGRAM_ID
```

---

### 3.3 `StartAuction`

**Description:** Creates a new auction account. Called by LiquidationEngine via chained call.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `collateral_auction_params_id` | Yes | — | Nonce incremented |
| 1 | `new_auction_id` (PDA) | Yes | PDA (new) | The new auction |
| 2 | `liquidation_engine_params_id` | No | Yes (PDA) | Must be registered LiquidationEngine |
| 3 | `lsc_market_oracle_id` | No | — | For timestamp |

**PDA derivation:**
```
new_auction_id = COLLATERAL_AUCTION_ID.derive(
    b"collateral_auction" || auction_params.auction_nonce.to_le_bytes()
)
```

**Authorization:** `liquidation_engine_params_id` must be `is_authorized`.

**Validations:**
- `liquidation_engine_params_id.is_authorized == true`
- `liquidation_engine_params_id.id == auction_params.liquidation_engine_params_id`
- `collateral_to_sell > 0`
- `amount_to_raise > 0`

**State Transition:**
```
let nonce = auction_params.auction_nonce;
auction_params.auction_nonce += 1;

new_auction.account_type = 51
new_auction.auction_id = nonce
new_auction.collateral_type_id = collateral_type_id  // from liquidation context
new_auction.collateral_vault_id = auction_params.collateral_vault_id
new_auction.amount_to_raise = amount_to_raise
new_auction.collateral_to_sell = collateral_to_sell
new_auction.amount_raised = 0
new_auction.safe_id = safe_id
new_auction.created_at = current_timestamp
new_auction.expires_at = current_timestamp + auction_params.auction_ttl
new_auction.settled = false
new_auction.forgone_collateral = 0
```

---

### 3.4 `BuyCollateral`

**Description:** A buyer purchases collateral at a discount. They pay LSC; they receive LOGOS.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `collateral_auction_params_id` | No | — | For discount params |
| 1 | `auction_id` | Yes | — | The auction to buy from |
| 2 | `oracle_relayer_params_id` | No | — | For redemption_price |
| 3 | `collateral_oracle_id` | No | — | For current collateral price |
| 4 | `buyer_account` | No | Yes | Buyer |
| 5 | `buyer_lsc_holding_id` | Yes | Yes | Buyer's LSC (pays here) |
| 6 | `buyer_logos_holding_id` | Yes | — | Buyer's LOGOS (receives here) |
| 7 | `auction_collateral_vault_id` | Yes | PDA | Vault (sends LOGOS) |
| 8 | `accounting_engine_surplus_id` | Yes | — | Receives LSC payment |
| 9 | `lsc_token_def_id` | No | — | For burn/transfer |

**Price Computation:**
```
// Current discounted price of collateral in LSC terms (Wad per collateral unit)
// Step 1: collateral price in USD (Wad)
p_collateral_usd = collateral_oracle.median_price  // Wad

// Step 2: redemption price in USD (Ray → convert to Wad)
p_redemption_usd = oracle_relayer_params.redemption_price / (RAY / WAD)  // Wad

// Step 3: collateral price in LSC (Wad)
p_collateral_lsc = p_collateral_usd * WAD / p_redemption_usd  // Wad

// Step 4: apply discount
discount_factor = WAD - auction_params.auction_discount  // e.g. WAD - 5% = 0.95 * WAD
p_discounted_lsc = p_collateral_lsc * discount_factor / WAD  // Wad
```

**Amount Computation:**
```
// Max collateral we can sell given remaining amount_to_raise
// amount_to_raise is in Rad (10^45), need to convert to LSC Wad then to collateral Wad
remaining_lsc_rad = auction.amount_to_raise - auction.amount_raised
remaining_lsc_wad = remaining_lsc_rad / RAY  // Wad

// Max collateral to cover remaining debt
max_collateral_for_debt = remaining_lsc_wad * WAD / p_discounted_lsc  // Wad

// Actual collateral to buy = min(requested, available, max_for_debt)
actual_collateral = min(
    collateral_amount,          // buyer's request
    auction.collateral_to_sell, // remaining in auction
    max_collateral_for_debt,    // limited by debt target
)

// Ensure minimum bid
assert!(actual_collateral >= auction_params.minimum_bid
    || actual_collateral == auction.collateral_to_sell,  // allow buying the last bit
    BelowMinimumBid
);

// LSC cost to buyer
lsc_cost_wad = actual_collateral * p_discounted_lsc / WAD  // Wad
lsc_cost_rad = lsc_cost_wad * RAY  // Rad
```

**Validations:**
```
assert!(!auction.settled, AuctionSettled);
assert!(auction.collateral_to_sell > 0, AuctionEmpty);
assert!(collateral_oracle.valid, OracleInvalid);
assert!(buyer_account.is_authorized, Unauthorized);
// Oracle freshness: timestamp within max_feed_age
```

**State Transition:**
```
auction.collateral_to_sell -= actual_collateral
auction.amount_raised += lsc_cost_rad

// If auction is done (all collateral sold or debt target hit):
if auction.collateral_to_sell == 0 || auction.amount_raised >= auction.amount_to_raise {
    auction.settled = true
    // Leftover collateral (if any) returned to SAFE owner
    auction.forgone_collateral = auction.collateral_to_sell
}
```

**Chained Calls:**
```
// 1. Buyer pays LSC to accounting engine surplus
TokenProgram::Transfer {
    amount_to_transfer: lsc_cost_wad,
    // accounts: [buyer_lsc_holding_id (auth), accounting_engine_surplus_id]
}

// 2. Auction vault sends LOGOS to buyer
TokenProgram::Transfer {
    amount_to_transfer: actual_collateral,
    // accounts: [auction_collateral_vault_id (PDA auth), buyer_logos_holding_id]
}

// 3. If settled: notify LiquidationEngine to update circuit breaker
// (Optional optimization: can be done by caller separately)
```

**Errors:**
- `AuctionSettled`
- `AuctionEmpty`
- `AuctionExpired`
- `OracleInvalid`
- `BelowMinimumBid`
- `Unauthorized`
- `InsufficientLSC`

---

### 3.5 `SettleAuction`

**Description:** Finalizes a completed auction. Returns leftover collateral to SAFE owner (if any) and notifies the LiquidationEngine to reduce `current_on_auction_system_coins`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `collateral_auction_params_id` | No | — | — |
| 1 | `auction_id` | Yes | — | The auction to settle |
| 2 | `safe_id` | No | — | For returning leftover collateral |
| 3 | `safe_owner_logos_holding_id` | Yes | — | Receives forgone collateral |
| 4 | `auction_collateral_vault_id` | Yes | PDA | Source of leftover collateral |
| 5 | `liquidation_engine_params_id` | Yes | — | Circuit breaker update |
| 6 | `liquidation_account_id` | Yes | — | Liquidation record to close |

**Validations:**
```
assert!(auction.settled, NotSettled);
assert!(auction.forgone_collateral == 0 || safe_owner_logos_holding has correct owner);
```

**State Transition:**
```
liquidation_engine_params.current_on_auction_system_coins -= auction.amount_to_raise
liquidation_account.data = [0; ...]  // close liquidation record
```

**Chained Calls:**
```
// Return forgone collateral to SAFE owner
if auction.forgone_collateral > 0 {
    TokenProgram::Transfer {
        amount_to_transfer: auction.forgone_collateral,
        // accounts: [auction_collateral_vault_id (PDA auth), safe_owner_logos_holding_id]
    }
}
```

---

### 3.6 `RestartAuction`

**Description:** If an auction expires (`current_time > auction.expires_at`) without being fully settled, anyone can restart it with updated oracle prices and a slightly higher discount.

**Required Accounts:** Same as `BuyCollateral` minus buyer accounts; plus auction_id.

**Algorithm:**
```
assert!(current_time > auction.expires_at, AuctionNotExpired);
assert!(!auction.settled, AuctionSettled);

// Increase discount by the configured increment
new_discount = min(
    auction_params.auction_discount + auction_params.discount_increment,
    auction_params.max_discount,
)
// Note: discount is re-applied at BuyCollateral time from params; store updated discount in auction account
// or update params.auction_discount temporarily — implementation must track per-auction discount

auction.expires_at = current_time + auction_params.auction_ttl
// Note: discount is recomputed dynamically at BuyCollateral time based on params,
// so the auction itself doesn't need to store it — just extend the TTL.
```

**Validations:**
- `current_time > auction.expires_at`
- `!auction.settled`

**State Transition:**
```
auction.expires_at = current_time + auction_params.auction_ttl
```

---

## 4. Partial vs Full Liquidation

In v1, LSC implements **full liquidation**: when a SAFE is liquidated, ALL of its collateral is seized and ALL of its debt is cleared. This is simpler and avoids the complexity of dust SAFEs.

**Rationale:** Partial liquidation (used in MakerDAO / RAI) allows liquidating only enough to restore the CR. However, it complicates auction accounting and can leave small SAFEs in limbo. Full liquidation is standard in newer DeFi protocols and simpler to implement correctly.

**Exception:** If the collateral value is insufficient to cover the full debt (i.e., the SAFE is insolvent — collateral < debt at current prices), all collateral is seized and the remaining debt becomes **bad debt**, pushed to the AccountingEngine.

---

## 5. Liquidation Sequence Diagram

```
Liquidator
    │
    ├─ LiquidateSafe ──────────────────────────────► LiquidationEngine
    │                                                       │
    │                         reads: safe, oracle, params ◄─┤
    │                         validates: SAFE is underwater  │
    │                                                        │
    │                         chained call #1 ──────────────►├─ LSCEngine::ConfiscateSafe
    │                                                         │     │
    │                                                         │     ├─ safe.collateral → 0
    │                                                         │     ├─ safe.debt → 0
    │                                                         │     └─ chained call: Token::Transfer(vault→auction_vault)
    │                         chained call #2 ──────────────►├─ AccountingEngine::PushDebt
    │                                                         │     └─ debt_queue += actual_debt
    │                         chained call #3 ──────────────►└─ CollateralAuction::StartAuction
    │                                                               └─ creates auction account
    │
    ├─ (later) BuyCollateral ─────────────────────► CollateralAuction
    │                                                       │
    │                         reads: price, discount ◄──────┤
    │                         computes: LSC cost             │
    │                         chained call: LSC transfer ────►├─ Token::Transfer(buyer_lsc → surplus)
    │                         chained call: LOGOS transfer ──►└─ Token::Transfer(vault → buyer)
    │
    └─ (when auction full) SettleAuction ──────────► CollateralAuction
                                                            │
                                                            ├─ circuit breaker ↓
                                                            └─ leftover collateral → SAFE owner
```

---

## 6. Error Codes

```rust
enum LiquidationEngineError {
    AlreadyInitialized         = 4000,
    Unauthorized               = 4001,
    SettlementActive           = 4002,
    SafeNotLiquidatable        = 4003,
    AlreadyLiquidated          = 4004,
    OracleInvalid              = 4005,
    OracleStale                = 4006,
    OnAuctionLimitReached      = 4007,
    InvalidPDA                 = 4008,
}

enum CollateralAuctionError {
    AlreadyInitialized         = 4100,
    Unauthorized               = 4101,
    AuctionNotFound            = 4102,
    AuctionSettled             = 4103,
    AuctionEmpty               = 4104,
    AuctionExpired             = 4105,
    AuctionNotExpired          = 4106,
    BelowMinimumBid            = 4107,
    OracleInvalid              = 4108,
    InsufficientLSC            = 4109,
    InvalidPDA                 = 4110,
    NotSettled                 = 4111,
}
```
