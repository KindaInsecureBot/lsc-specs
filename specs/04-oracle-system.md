# LSC — Oracle System

## Overview

The Oracle System solves a fundamental constraint of LEZ: **programs cannot read block timestamps or external state**. All time and price data must be explicitly passed as accounts.

The Oracle System consists of:
1. **OracleProgram** — aggregates multiple external price feeds into a single authoritative median price
2. **OracleRelayer** — translates raw collateral price + redemption rate into the prices consumed by LSCEngine and LiquidationEngine

Price data flows:
```
External keepers
     │ (submit prices)
     ▼
OracleFeedAccounts (per source)
     │ (aggregated by OracleProgram)
     ▼
MedianOracleAccounts (LOGOS/USD, LSC/USD)
     │ (read by)
     ├──► LSCEngine (collateral price for CR checks)
     ├──► LiquidationEngine (liquidation price)
     ├──► PIController (LSC market price)
     └──► OracleRelayer (redemption price updates)
```

---

## 1. OracleProgram

**Program ID alias:** `ORACLE_PROGRAM_ID`

### 1.1 Instruction Enum

```rust
enum OracleInstruction {
    /// One-time initialization
    Initialize {
        governance_id: [u8; 32],
        max_feed_age: u64,
        min_feeds_for_median: u8,
    },

    /// Register a new price feed source
    AddFeed {
        feed_id: u64,
        symbol: [u8; 32],
        keeper_id: [u8; 32],
    },

    /// Remove a price feed (governance)
    RemoveFeed {
        feed_id: u64,
    },

    /// Change the authorized keeper for a feed
    UpdateFeedKeeper {
        feed_id: u64,
        new_keeper_id: [u8; 32],
    },

    /// Keeper submits a new price observation
    SubmitPrice {
        price: u128,       // Wad (USD price with 18 decimals)
        confidence: u128,  // Wad (uncertainty/spread)
        timestamp: u64,    // Unix seconds from external clock
    },

    /// Register a new median oracle (aggregator for a symbol)
    AddMedianOracle {
        symbol: [u8; 32],
        feed_ids: Vec<u64>,  // up to 8 feed IDs to aggregate
    },

    /// Recompute median from current feed values
    UpdateMedian {
        symbol: [u8; 32],
    },

    /// Update system parameters (governance)
    UpdateParams {
        new_max_feed_age: Option<u64>,
        new_min_feeds_for_median: Option<u8>,
    },
}
```

---

### 1.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_config_id` (PDA) | Yes | PDA (new) | Config account |
| 1 | `governance_account` | No | Yes | Initial governance |

**State Transition:**
```
oracle_config.account_type = 90
oracle_config.governance_id = governance_account.id
oracle_config.max_feed_age = max_feed_age
oracle_config.min_feeds_for_median = min_feeds_for_median
oracle_config.feed_count = 0
```

---

### 1.3 `AddFeed`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_config_id` | Yes | — | Config (feed_count incremented) |
| 1 | `oracle_feed_id` (PDA) | Yes | PDA (new) | Derived from feed_id |
| 2 | `governance_account` | No | Yes | Must match oracle_config.governance_id |

**PDA derivation:**
```
oracle_feed_id = ORACLE_PROGRAM_ID.derive(b"feed" || feed_id.to_le_bytes())
```

**Validations:**
- `governance_account.id == oracle_config.governance_id`
- `oracle_feed_id` uninitialized
- `feed_id` unique (governance responsibility to ensure no collision)

**State Transition:**
```
oracle_feed.account_type = 91
oracle_feed.feed_id = feed_id
oracle_feed.symbol = symbol
oracle_feed.keeper_id = keeper_id
oracle_feed.price = 0
oracle_feed.confidence = 0
oracle_feed.timestamp = 0
oracle_feed.active = true
oracle_config.feed_count += 1
```

---

### 1.4 `SubmitPrice`

**Description:** An authorized keeper submits a new price. This is the only instruction that brings external data on-chain.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_feed_id` | Yes | — | Feed to update |
| 1 | `keeper_account` | No | Yes | Must match oracle_feed.keeper_id |

**Validations:**
- `keeper_account.id == oracle_feed.keeper_id`
- `keeper_account.is_authorized == true`
- `oracle_feed.active == true`
- `timestamp > oracle_feed.timestamp` (monotonically increasing timestamps only)
- `timestamp <= oracle_feed.timestamp + 7200` (max 2-hour jumps per update, prevents fake future timestamps)
- `price > 0`
- `confidence < price` (sanity check: uncertainty < price)

**State Transition:**
```
oracle_feed.price = price
oracle_feed.confidence = confidence
oracle_feed.timestamp = timestamp
```

**Note:** After submitting a price, the keeper should call `UpdateMedian` to refresh the aggregated oracle.

**Errors:**
- `Unauthorized`
- `FeedInactive`
- `TimestampNotMonotonic`
- `TimestampJumpTooLarge`
- `InvalidPrice`

---

### 1.5 `AddMedianOracle`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_config_id` | No | — | For governance check |
| 1 | `median_oracle_id` (PDA) | Yes | PDA (new) | Derived from symbol |
| 2 | `governance_account` | No | Yes | — |
| 3..10 | `oracle_feed_ids` | No | — | The feeds to aggregate (1–8) |

**PDA derivation:**
```
// symbol is a fixed 32-byte array (right-padded with 0s)
// e.g. b"LOGOS/USD" + [0; 23]
median_oracle_id = ORACLE_PROGRAM_ID.derive(b"median" || symbol)
```

**Validations:**
- Governance auth
- `feed_ids.len() >= 1 && feed_ids.len() <= 8`
- All feed accounts must be initialized OracleFeedAccounts
- `median_oracle_id` uninitialized

**State Transition:**
```
median_oracle.account_type = 92
median_oracle.symbol = symbol
median_oracle.feed_ids = [feed_id_accounts...; padded with zeros]
median_oracle.feed_count = feed_ids.len()
median_oracle.median_price = 0
median_oracle.last_update_timestamp = 0
median_oracle.median_timestamp = 0
median_oracle.valid = false
```

---

### 1.6 `UpdateMedian`

**Description:** Recomputes the median price from all contributing feeds. Anyone can call this; it's permissionless (keepers call it after `SubmitPrice`).

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_config_id` | No | — | For max_feed_age, min_feeds_for_median |
| 1 | `median_oracle_id` | Yes | — | Median to update |
| 2..9 | `oracle_feed_ids` | No | — | All contributing feeds (same order as registered) |

**Algorithm:**
```
1. Collect all active feeds with timestamp > (newest_timestamp - max_feed_age)
   - "newest_timestamp" = max(feed.timestamp for all active feeds)
2. Filter out feeds where feed.active == false
3. If valid_feed_count < min_feeds_for_median:
   median_oracle.valid = false
   return (do not update price — keep last valid price)
4. Sort valid feed prices ascending
5. median_price = prices[valid_count / 2]  (middle element; for even count, take lower middle)
6. median_timestamp = sorted_timestamps[valid_count / 2]
7. Update median_oracle:
   median_oracle.median_price = median_price
   median_oracle.last_update_timestamp = newest_timestamp (from step 1)
   median_oracle.median_timestamp = median_timestamp
   median_oracle.valid = true
```

**Validations:**
- All passed feed accounts must have account_type == 91
- Feed account IDs must match those registered in median_oracle.feed_ids

**Errors:**
- `InsufficientFeeds` — fewer than min_feeds_for_median valid feeds
- `AccountMismatch` — passed feeds don't match registered feeds

---

## 2. OracleRelayer

**Program ID alias:** `ORACLE_RELAYER_ID`

The OracleRelayer manages two key values:
1. **Redemption Price** — the internal target price for LSC (drifts over time based on redemption rate)
2. **Redemption Rate** — a per-second multiplier set by the PIController

It does **not** store the raw collateral price; that comes directly from the MedianOracleAccount.

### 2.1 Instruction Enum

```rust
enum OracleRelayerInstruction {
    /// One-time initialization
    Initialize {
        initial_redemption_price: u128,     // Ray
        initial_redemption_rate: u128,      // Ray (= RAY for neutral)
        redemption_rate_lower_bound: u128,  // Ray
        redemption_rate_upper_bound: u128,  // Ray
        collateral_oracle_id: [u8; 32],
        lsc_market_oracle_id: [u8; 32],
        pi_controller_state_id: [u8; 32],
        collateral_price_safety_margin: u128,  // Ray
    },

    /// Update the redemption rate (called by PIController — privileged)
    UpdateRedemptionRate {
        new_redemption_rate: u128,   // Ray
    },

    /// Refresh the redemption price (callable by anyone; applies accumulated rate drift)
    UpdateRedemptionPrice,

    /// Update system parameters (governance)
    UpdateParams {
        new_redemption_rate_lower_bound: Option<u128>,
        new_redemption_rate_upper_bound: Option<u128>,
        new_collateral_price_safety_margin: Option<u128>,
        new_collateral_oracle_id: Option<[u8; 32]>,
        new_lsc_market_oracle_id: Option<[u8; 32]>,
        new_pi_controller_state_id: Option<[u8; 32]>,
    },
}
```

---

### 2.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_relayer_params_id` (PDA) | Yes | PDA (new) | — |
| 1 | `governance_account` | No | Yes | Initial governance |

**Validations:**
- `oracle_relayer_params_id` uninitialized
- `initial_redemption_price > 0`
- `redemption_rate_lower_bound <= RAY <= redemption_rate_upper_bound`
- `collateral_price_safety_margin <= RAY`

**State Transition:**
```
oracle_relayer_params.account_type = 10
oracle_relayer_params.redemption_price = initial_redemption_price
oracle_relayer_params.redemption_rate = initial_redemption_rate  // = RAY
oracle_relayer_params.last_redemption_price_update = current_timestamp_from_oracle
oracle_relayer_params.redemption_rate_lower_bound = redemption_rate_lower_bound
oracle_relayer_params.redemption_rate_upper_bound = redemption_rate_upper_bound
oracle_relayer_params.collateral_oracle_id = collateral_oracle_id
oracle_relayer_params.lsc_market_oracle_id = lsc_market_oracle_id
oracle_relayer_params.pi_controller_state_id = pi_controller_state_id
oracle_relayer_params.collateral_price_safety_margin = collateral_price_safety_margin
```

---

### 2.3 `UpdateRedemptionPrice`

**Description:** Applies the accumulated redemption rate drift to update the stored redemption price. Must be called before any operation that reads redemption_price (LSCEngine GenerateDebt, LiquidationEngine, etc.).

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_relayer_params_id` | Yes | — | To update redemption_price |
| 1 | `lsc_market_oracle_id` | No | — | Provides current_timestamp |

**Formula:**
```
time_elapsed = lsc_market_oracle.median_timestamp - oracle_relayer_params.last_redemption_price_update

// Compute redemption_rate^time_elapsed using repeated squaring or approximation
// For small elapsed times, use: new_price ≈ old_price * (1 + rate_delta * time_elapsed)
// For precision, use rpow (ray power):
new_redemption_price = rpow(redemption_rate, time_elapsed, RAY) * redemption_price / RAY
```

**rpow Implementation:**
```rust
/// Computes base^exp where base and result are in Ray units (10^27)
/// Uses binary exponentiation
fn rpow(base: u128, exp: u64, one: u128) -> u128 {
    if exp == 0 { return one; }
    let mut result = one;
    let mut base = base;
    let mut exp = exp;
    while exp > 1 {
        if exp % 2 == 1 {
            result = rmul(result, base);
        }
        base = rmul(base, base);
        exp /= 2;
    }
    rmul(result, base)
}

fn rmul(x: u128, y: u128) -> u128 {
    // x * y / RAY, using u256 intermediate
    (x as u256 * y as u256 / RAY as u256) as u128
}
```

**Validations:**
- `lsc_market_oracle.valid == true`
- `lsc_market_oracle.median_timestamp >= oracle_relayer_params.last_redemption_price_update`
- `lsc_market_oracle.median_timestamp - last_update <= max_allowed_lag` (e.g. 86400 seconds = 1 day)

**State Transition:**
```
oracle_relayer_params.redemption_price = new_redemption_price
oracle_relayer_params.last_redemption_price_update = lsc_market_oracle.median_timestamp
```

**Errors:**
- `OracleInvalid` — median oracle not valid
- `TimestampRegression` — oracle timestamp < last update
- `TimestampTooFarAhead` — oracle timestamp exceeds max lag

---

### 2.4 `UpdateRedemptionRate`

**Description:** Called by PIController to set a new redemption rate. The rate is clamped to bounds.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `oracle_relayer_params_id` | Yes | — | To update rate |
| 1 | `pi_controller_state_id` | No | Yes (PDA) | Must be registered PI controller |

**Authorization:** The `pi_controller_state_id` account must have `is_authorized == true`. The PI Controller program calls this as a chained call, passing its PDA as authorized.

**Validations:**
- `pi_controller_state_id.is_authorized == true`
- `pi_controller_state.id == oracle_relayer_params.pi_controller_state_id`

**Clamping:**
```
clamped_rate = clamp(
    new_redemption_rate,
    oracle_relayer_params.redemption_rate_lower_bound,
    oracle_relayer_params.redemption_rate_upper_bound,
)
```

**State Transition:**
```
// First, update the redemption price with the OLD rate before changing it
// (caller must have already called UpdateRedemptionPrice, or we do it here)
oracle_relayer_params.redemption_rate = clamped_rate
```

**Errors:**
- `Unauthorized` — not called from registered PI controller
- `RateOutOfBounds` — new_rate violates hard bounds (before clamping — should not error, just clamp)

---

## 3. Timestamp Model

### 3.1 How Timestamps Flow

```
Real World Clock
      │
      │ (keeper observes)
      ▼
OracleFeedAccount.timestamp  ←  keeper submits in SubmitPrice
      │
      │ (OracleProgram::UpdateMedian computes)
      ▼
MedianOracleAccount.median_timestamp  ← used as "block time" by all programs
```

### 3.2 Properties and Guarantees

1. **Monotonicity:** Each feed's timestamp is strictly increasing (enforced by `SubmitPrice`).
2. **Liveness:** If keepers stop submitting, `median_oracle.valid` eventually becomes false (oracle staleness), halting operations.
3. **Manipulation resistance:** A single keeper cannot push a fake timestamp far ahead because:
   - Max 2-hour jump per update per feed
   - Median of multiple feeds is taken (requires majority of keepers to collude)
4. **Precision:** Unix seconds. All programs treat time in integer seconds.
5. **Max lag:** Programs enforce a `max_allowed_lag` — the maximum seconds the oracle timestamp can trail the current real time. Recommended: 3600 seconds (1 hour).

### 3.3 Keeper Requirements

Two MedianOracles must be maintained:
1. `LOGOS_USD_MEDIAN` — LOGOS/USD price, used for collateral valuation
2. `LSC_USD_MEDIAN` — LSC/USD price, used by PIController

At least `min_feeds_for_median` (recommended: 3) independent keepers must update each feed. Keepers are expected to submit at least once per `max_feed_age / 2` seconds to maintain validity.

### 3.4 Oracle Security Model

**Delayed Oracle (for liquidations):** To prevent flash-loan-style oracle manipulation, the collateral price used for liquidation checks uses a **safety margin** (`collateral_price_safety_margin`). The effective liquidation price is:
```
liquidation_collateral_price = median_price * collateral_price_safety_margin / RAY
```

This means, e.g., at 99.5% safety margin, collateral is valued at 99.5% of market price for liquidation purposes. This reduces manipulation attack surface at the cost of slightly conservative liquidation thresholds.

**For high-value deployments**, consider implementing a `DelayedOracle` that stores the last two oracle prices and uses the older one (e.g., 1-hour delayed price) for liquidation checks. This provides stronger manipulation resistance.

```rust
// DelayedOracleFeedAccount (optional enhancement)
struct DelayedOracleFeedAccount {
    pub current_price: u128,       // Wad, most recent
    pub current_timestamp: u64,
    pub delayed_price: u128,       // Wad, one update delay old
    pub delayed_timestamp: u64,
    pub update_delay: u64,         // Minimum seconds between delayed price updates
}
// Liquidation uses delayed_price; debt generation uses current_price
```

---

## 4. AMM Program Integration for LSC Market Price

The LSC/USD market price is derived from the **AMM Program's LOGOS/LSC pool** combined with the LOGOS/USD oracle price.

```
LSC_market_price_usd = (LOGOS per LSC from AMM) * (LOGOS/USD from oracle)
                      = (AMM reserve_logos / AMM reserve_lsc) * logos_usd_price
```

However, because the AMM pool reserves change each block and can be manipulated, the keeper must:
1. Read the AMM pool reserves (PoolDefinition.reserve_a, reserve_b)
2. Compute the spot price
3. Optionally: compute a time-weighted average price (TWAP) using multiple historical samples
4. Submit this as a price feed to `OracleProgram::SubmitPrice` for the LSC_USD feed

**For v1:** Keepers compute and submit spot price. The median across 3+ keepers provides manipulation resistance.

**For v2:** Implement on-chain TWAP accumulator (AMM extension program) and have OracleProgram read from it directly.

---

## 5. Error Codes

```rust
enum OracleProgramError {
    AlreadyInitialized     = 2000,
    Unauthorized           = 2001,
    FeedNotFound           = 2002,
    FeedInactive           = 2003,
    FeedAlreadyExists      = 2004,
    TimestampNotMonotonic  = 2005,
    TimestampJumpTooLarge  = 2006,
    InvalidPrice           = 2007,
    InsufficientFeeds      = 2008,
    AccountMismatch        = 2009,
    MedianOracleExists     = 2010,
    InvalidPDA             = 2011,
}

enum OracleRelayerError {
    AlreadyInitialized     = 2100,
    Unauthorized           = 2101,
    OracleInvalid          = 2102,
    OracleStale            = 2103,
    TimestampRegression    = 2104,
    TimestampTooFarAhead   = 2105,
    InvalidRate            = 2106,
    InvalidPrice           = 2107,
    InvalidPDA             = 2108,
}
```
