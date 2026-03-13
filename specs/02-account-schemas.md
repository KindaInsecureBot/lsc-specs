# LSC — Account Schemas

All account data is serialized using **Borsh** (Binary Object Representation Serializer for Hashing). All account `data` fields contain a Borsh-encoded struct. Account `balance` (u128) may be used for native LOGOS balance where applicable, but LSC uses the Token Program for token balances — so `balance` in most LSC accounts is 0.

Fixed-point conventions:
- `Ray` = `u128` representing a value × 10^27 (27 decimal places)
- `Wad` = `u128` representing a value × 10^18 (18 decimal places)
- `Rad` = `u128` representing a value × 10^45 (= ray × wad, used for debt units)
- Timestamps = `u64` (Unix seconds)
- Account IDs = `[u8; 32]`

---

## 1. LSC Engine Accounts

### 1.1 SystemParamsAccount

**PDA:** `LSC_ENGINE_ID.derive(b"system_params")`
**Owner:** `LSC_ENGINE_ID`

```rust
struct SystemParamsAccount {
    /// Account discriminator / version tag
    pub account_type: u8,               // = 1

    /// Governance multisig that can update params
    pub governance_id: [u8; 32],

    /// LSC token definition ID (for minting authority checks)
    pub lsc_token_def_id: [u8; 32],

    /// LOGOS token definition ID
    pub logos_token_def_id: [u8; 32],

    /// Oracle Relayer params account ID (for reading redemption price)
    pub oracle_relayer_params_id: [u8; 32],

    /// Tax Collector params account ID
    pub tax_collector_params_id: [u8; 32],

    /// Liquidation Engine params account ID
    pub liquidation_engine_params_id: [u8; 32],

    /// Accounting Engine params account ID
    pub accounting_engine_params_id: [u8; 32],

    /// Global settlement state account ID
    pub global_settlement_state_id: [u8; 32],

    /// Whether global settlement has been triggered
    pub settlement_active: bool,

    /// Nonce for SAFE ID derivation (increments per new SAFE)
    pub safe_nonce: u64,

    /// Nonce for collateral vault derivation
    pub vault_nonce: u64,

    /// Reserved for future use
    pub _reserved: [u8; 64],
}
```

---

### 1.2 CollateralTypeAccount

**PDA:** `LSC_ENGINE_ID.derive(b"collateral_type" || collateral_token_def_id)`
**Owner:** `LSC_ENGINE_ID`

```rust
struct CollateralTypeAccount {
    pub account_type: u8,               // = 2

    /// Token definition ID for this collateral (e.g. LOGOS_TOKEN_DEF_ID)
    pub token_def_id: [u8; 32],

    /// Vault account that holds collateral (PDA)
    pub vault_id: [u8; 32],

    /// Accumulated stability fee + redemption rate multiplier
    /// Starts at RAY (1.0). Increases over time.
    /// Unit: Ray
    pub accumulated_rate: u128,

    /// Timestamp of last accumulated_rate update (Unix seconds)
    pub last_stability_fee_update: u64,

    /// Per-second stability fee rate
    /// e.g. RAY + 315522921573 ≈ 1% APY
    /// Unit: Ray
    pub stability_fee: u128,

    /// Minimum collateralization ratio for debt generation
    /// e.g. 1_500_000_000_000_000_000_000_000_000 = 1.5 (150%)
    /// Unit: Ray
    pub safety_ratio: u128,

    /// Minimum collateralization ratio before liquidation
    /// e.g. 1_350_000_000_000_000_000_000_000_000 = 1.35 (135%)
    /// Unit: Ray
    pub liquidation_ratio: u128,

    /// Liquidation penalty (fraction of collateral seized as fee)
    /// e.g. 130_000_000_000_000_000 = 0.13 (13%)
    /// Unit: Wad
    pub liquidation_penalty: u128,

    /// Maximum total LSC debt for this collateral type
    /// Unit: Rad
    pub debt_ceiling: u128,

    /// Minimum debt per SAFE (dust protection)
    /// Unit: Rad
    pub debt_floor: u128,

    /// Total normalized debt across all SAFEs of this type
    /// Effective total debt = global_normalized_debt * accumulated_rate
    /// Unit: Wad
    pub global_normalized_debt: u128,

    /// Whether this collateral type is active
    pub active: bool,

    /// Whether global settlement has processed this collateral type
    pub settlement_processed: bool,

    /// Final collateral price at settlement (set by GlobalSettlement)
    /// Unit: Ray (collateral per LSC)
    pub final_collateral_price: u128,

    pub _reserved: [u8; 32],
}
```

---

### 1.3 SafeAccount

**PDA:** `LSC_ENGINE_ID.derive(b"safe" || owner_account_id || nonce_bytes)`
**Owner:** `LSC_ENGINE_ID`

```rust
struct SafeAccount {
    pub account_type: u8,               // = 3

    /// Owner of this SAFE (must be is_authorized to modify)
    pub owner_id: [u8; 32],

    /// Collateral type this SAFE uses
    pub collateral_type_id: [u8; 32],

    /// Amount of collateral locked (in collateral token's native precision)
    /// Unit: Wad (18 decimals assumed for LOGOS)
    pub collateral: u128,

    /// Normalized debt (debt / accumulated_rate at time of last change)
    /// Effective debt = normalized_debt * accumulated_rate
    /// Unit: Wad
    pub normalized_debt: u128,

    /// Timestamp this SAFE was opened
    pub opened_at: u64,

    /// Timestamp of last modification
    pub last_modified_at: u64,

    /// Whether this SAFE has been liquidated and is closed
    pub liquidated: bool,

    /// Accounts that are allowed to modify this SAFE on behalf of owner
    /// (delegated operators, max 4)
    pub allowed_operators: [[u8; 32]; 4],
    pub allowed_operators_count: u8,

    pub _reserved: [u8; 32],
}
```

---

### 1.4 CollateralVaultAccount

**PDA:** `LSC_ENGINE_ID.derive(b"vault" || collateral_type_id)`
**Owner:** `LSC_ENGINE_ID`

This is a **Token Program holding account** (TokenHolding) owned by the LSC Engine's vault PDA. It stores LOGOS tokens deposited as collateral.

```rust
/// This account is a TokenHolding account owned by TOKEN_PROGRAM_ID,
/// but the holder authority is the CollateralVaultAccount PDA.
/// The LSC Engine uses its PDA authorization to transfer from it.
///
/// TokenHolding { Fungible { definition_id: LOGOS_TOKEN_DEF_ID, balance: u128 } }
/// Stored in the Token Program's account format.
```

The LSC Engine authorizes transfers from the vault by including the vault's PDA ID in the accounts list with `is_authorized = true` (the engine derives this PDA and can sign for it).

---

### 1.5 GlobalDebtAccount

**PDA:** `LSC_ENGINE_ID.derive(b"global_debt")`
**Owner:** `LSC_ENGINE_ID`

```rust
struct GlobalDebtAccount {
    pub account_type: u8,               // = 4

    /// Total LSC minted (sum across all collateral types)
    /// Unit: Rad
    pub total_debt: u128,

    /// Total LSC in surplus (held by AccountingEngine)
    /// Unit: Rad
    pub total_surplus: u128,

    pub _reserved: [u8; 32],
}
```

---

## 2. Oracle Relayer Accounts

### 2.1 OracleRelayerParamsAccount

**PDA:** `ORACLE_RELAYER_ID.derive(b"oracle_relayer_params")`
**Owner:** `ORACLE_RELAYER_ID`

```rust
struct OracleRelayerParamsAccount {
    pub account_type: u8,               // = 10

    /// Current redemption price (internal target price for LSC)
    /// Updated lazily: computed as redemption_price * redemption_rate^(now - last_update)
    /// Unit: Ray (price in USD, so RAY = $1.00)
    pub redemption_price: u128,

    /// Per-second redemption rate multiplier
    /// RAY = 1.0 (no change), RAY + x = positive rate (price rises)
    /// Unit: Ray
    pub redemption_rate: u128,

    /// Timestamp of last redemption_price update
    pub last_redemption_price_update: u64,

    /// Minimum allowed redemption rate (floor)
    /// e.g. RAY - 5e24 ≈ -99.99% per year
    pub redemption_rate_lower_bound: u128,

    /// Maximum allowed redemption rate (ceiling)
    pub redemption_rate_upper_bound: u128,

    /// Oracle account providing collateral price (logos/usd)
    pub collateral_oracle_id: [u8; 32],

    /// Oracle account providing LSC market price (lsc/usd)
    pub lsc_market_oracle_id: [u8; 32],

    /// PI Controller state account ID (authorized to update redemption_rate)
    pub pi_controller_state_id: [u8; 32],

    /// Safety price ratio: how much to discount collateral oracle price
    /// for liquidation purposes (e.g. 0.995 * RAY)
    /// Unit: Ray
    pub collateral_price_safety_margin: u128,

    pub _reserved: [u8; 32],
}
```

---

## 3. PI Controller Accounts

### 3.1 PIControllerStateAccount

**PDA:** `PI_CONTROLLER_ID.derive(b"pi_controller_state")`
**Owner:** `PI_CONTROLLER_ID`

```rust
struct PIControllerStateAccount {
    pub account_type: u8,               // = 20

    /// Proportional gain (Kp)
    /// Signed: positive values → rate increases when market < redemption
    /// Unit: Ray (but applied to an error term that is dimensionless)
    pub kp: i128,

    /// Integral gain (Ki)
    /// Unit: Ray per second
    pub ki: i128,

    /// Accumulated error integral (sum of error * dt over time)
    /// Signed
    /// Unit: Ray-seconds
    pub error_integral: i128,

    /// Timestamp of last PI controller update
    pub last_update_time: u64,

    /// Minimum seconds between updates
    /// e.g. 3600 (1 hour)
    pub update_delay: u64,

    /// Minimum redemption rate output from controller
    /// e.g. RAY - 5e24 (roughly -50% per year)
    pub per_second_cumulative_leak: u128,

    /// Leak applied to integral each update (dampens windup)
    /// e.g. 999_999_920_000_000_000_000_000_000 (≈ 0.99999992 per second)
    /// Unit: Ray
    pub integral_period_size: u64,

    /// Oracle Relayer params account ID (to read redemption price and market price)
    pub oracle_relayer_params_id: [u8; 32],

    /// Oracle account for LSC market price
    pub lsc_market_oracle_id: [u8; 32],

    /// Noise barrier: minimum error magnitude to act on
    /// e.g. 5_000_000_000_000_000 = 0.5% error threshold
    /// Unit: Wad
    pub noise_barrier: u128,

    /// Last computed proportional term (for inspection)
    pub last_proportional_term: i128,

    /// Last computed integral term (for inspection)
    pub last_integral_term: i128,

    pub _reserved: [u8; 32],
}
```

---

## 4. Tax Collector Accounts

### 4.1 TaxCollectorParamsAccount

**PDA:** `TAX_COLLECTOR_ID.derive(b"tax_collector_params")`
**Owner:** `TAX_COLLECTOR_ID`

```rust
struct TaxCollectorParamsAccount {
    pub account_type: u8,               // = 30

    /// Accounting Engine's system surplus account ID
    /// (where accrued stability fees are sent as LSC)
    pub accounting_engine_surplus_id: [u8; 32],

    /// The LSC Engine's system params account (for looking up collateral types)
    pub lsc_engine_params_id: [u8; 32],

    /// Global stability fee (base rate applied to all collateral types)
    /// Additional per-type fees are in CollateralTypeAccount.stability_fee
    /// Unit: Ray
    pub global_stability_fee: u128,

    pub _reserved: [u8; 32],
}
```

---

## 5. Liquidation Engine Accounts

### 5.1 LiquidationEngineParamsAccount

**PDA:** `LIQUIDATION_ENGINE_ID.derive(b"liquidation_engine_params")`
**Owner:** `LIQUIDATION_ENGINE_ID`

```rust
struct LiquidationEngineParamsAccount {
    pub account_type: u8,               // = 40

    /// LSC Engine params account ID
    pub lsc_engine_params_id: [u8; 32],

    /// Collateral Auction House params account ID
    pub collateral_auction_params_id: [u8; 32],

    /// Accounting Engine params account ID
    pub accounting_engine_params_id: [u8; 32],

    /// Oracle Relayer params account ID (for redemption price)
    pub oracle_relayer_params_id: [u8; 32],

    /// Maximum LSC that can be targeted for liquidation in a single call
    /// (circuit breaker: prevents flash liquidation of too much debt)
    /// Unit: Rad
    pub on_auction_system_coin_limit: u128,

    /// Current amount of LSC being auctioned
    /// Unit: Rad
    pub current_on_auction_system_coins: u128,

    pub _reserved: [u8; 32],
}
```

### 5.2 LiquidationAccount

**PDA:** `LIQUIDATION_ENGINE_ID.derive(b"liquidation" || safe_id)`
**Owner:** `LIQUIDATION_ENGINE_ID`
**Lifecycle:** Created when liquidation begins; closed (account data zeroed) when auction completes.

```rust
struct LiquidationAccount {
    pub account_type: u8,               // = 41

    /// The SAFE that was liquidated
    pub safe_id: [u8; 32],

    /// Collateral auction account created for this liquidation
    pub auction_id: [u8; 32],

    /// Collateral amount seized
    /// Unit: Wad
    pub collateral_seized: u128,

    /// Debt to cover (including penalty)
    /// Unit: Rad
    pub debt_to_cover: u128,

    /// Timestamp of liquidation
    pub liquidated_at: u64,

    pub _reserved: [u8; 16],
}
```

---

## 6. Collateral Auction House Accounts

### 6.1 CollateralAuctionHouseParamsAccount

**PDA:** `COLLATERAL_AUCTION_ID.derive(b"collateral_auction_params")`
**Owner:** `COLLATERAL_AUCTION_ID`

```rust
struct CollateralAuctionHouseParamsAccount {
    pub account_type: u8,               // = 50

    /// LSC Engine params account ID (for authorization checks)
    pub lsc_engine_params_id: [u8; 32],

    /// Liquidation Engine params account ID (authorized to start auctions)
    pub liquidation_engine_params_id: [u8; 32],

    /// Oracle Relayer params account ID (for redemption price)
    pub oracle_relayer_params_id: [u8; 32],

    /// Accounting Engine surplus account ID (LSC received goes here)
    pub accounting_engine_surplus_id: [u8; 32],

    /// Fixed discount on collateral price for auction buyers
    /// e.g. 50_000_000_000_000_000 = 5% discount
    /// Unit: Wad
    pub auction_discount: u128,

    /// Maximum discount (in case of stale oracle)
    /// Unit: Wad
    pub max_discount: u128,

    /// Minimum discount
    /// Unit: Wad
    pub min_discount: u128,

    /// Minimum amount of collateral in an auction (dust protection)
    /// Unit: Wad
    pub minimum_bid: u128,

    /// Auction duration (seconds); if not settled by then, anyone can restart
    pub auction_ttl: u64,

    /// Nonce counter for auction IDs
    pub auction_nonce: u64,

    pub _reserved: [u8; 32],
}
```

### 6.2 CollateralAuctionAccount

**PDA:** `COLLATERAL_AUCTION_ID.derive(b"collateral_auction" || auction_nonce_bytes)`
**Owner:** `COLLATERAL_AUCTION_ID`

```rust
struct CollateralAuctionAccount {
    pub account_type: u8,               // = 51

    /// Unique auction ID (= nonce at creation)
    pub auction_id: u64,

    /// Collateral type being auctioned
    pub collateral_type_id: [u8; 32],

    /// Vault holding the collateral to be sold
    pub collateral_vault_id: [u8; 32],

    /// LSC amount this auction needs to raise (debt + penalty)
    /// Unit: Rad
    pub amount_to_raise: u128,

    /// Remaining collateral available for purchase
    /// Unit: Wad
    pub collateral_to_sell: u128,

    /// Amount of LSC already raised
    /// Unit: Rad
    pub amount_raised: u128,

    /// The SAFE that was liquidated
    pub safe_id: [u8; 32],

    /// Timestamp when auction was created
    pub created_at: u64,

    /// Timestamp when auction expires (created_at + auction_ttl)
    pub expires_at: u64,

    /// Whether auction is complete
    pub settled: bool,

    /// Forgone (leftover) collateral returned to SAFE owner after debt cleared
    pub forgone_collateral: u128,

    pub _reserved: [u8; 16],
}
```

---

## 7. Accounting Engine Accounts

### 7.1 AccountingEngineParamsAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"accounting_engine_params")`
**Owner:** `ACCOUNTING_ENGINE_ID`

```rust
struct AccountingEngineParamsAccount {
    pub account_type: u8,               // = 60

    /// Surplus Auction House params account ID
    pub surplus_auction_params_id: [u8; 32],

    /// LSC Engine params account ID
    pub lsc_engine_params_id: [u8; 32],

    /// Liquidation Engine params account ID (authorized to call PushDebt)
    pub liquidation_engine_params_id: [u8; 32],

    /// Surplus threshold: start surplus auction when surplus > this
    /// Unit: Rad
    pub surplus_auction_amount: u128,

    /// Buffer: surplus buffer kept before auctioning
    /// Unit: Rad
    pub surplus_buffer: u128,

    /// Timestamp of last surplus auction
    pub last_surplus_auction_time: u64,

    /// Minimum time between surplus auctions
    pub surplus_auction_delay: u64,

    /// Nonce for surplus auction IDs
    pub surplus_auction_nonce: u64,

    pub _reserved: [u8; 32],
}
```

### 7.2 SystemSurplusAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"system_surplus")`
**Owner:** `ACCOUNTING_ENGINE_ID`

This is a **Token Program holding account** (TokenHolding, Fungible) that holds the protocol's accumulated LSC surplus.

```rust
/// TokenHolding { Fungible { definition_id: LSC_TOKEN_DEF_ID, balance: u128 } }
/// Holder authority = ACCOUNTING_ENGINE_ID PDA
/// The AccountingEngine authorizes transfers from this account via its PDA.
```

### 7.3 SystemLogosHoldingAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"system_logos_holding")`
**Owner:** `ACCOUNTING_ENGINE_ID`

This is a **Token Program holding account** (TokenHolding, Fungible) that temporarily stages LOGOS during `RecapitalizeWithLOGOS`. The AccountingEngine authorizes transfers from it by including its PDA seed in `ChainedCall.pda_seeds`.

```rust
/// TokenHolding { Fungible { definition_id: LOGOS_TOKEN_DEF_ID, balance: u128 } }
/// Holder authority = ACCOUNTING_ENGINE_ID PDA
/// Must be initialized (via TokenProgram::InitializeAccount) before
/// RecapitalizeWithLOGOS can be called. Typically initialized alongside
/// AccountingEngine::Initialize as an optional chained call.
```

---

### 7.4 SystemDebtQueueAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"debt_queue")`
**Owner:** `ACCOUNTING_ENGINE_ID`

```rust
struct SystemDebtQueueAccount {
    pub account_type: u8,               // = 62

    /// Total bad debt queued (pending offset against surplus)
    /// Unit: Rad
    pub queued_debt: u128,

    /// Entries in the debt queue (up to 256)
    pub entries: [DebtQueueEntry; 256],
    pub entry_count: u16,

    pub _reserved: [u8; 32],
}

struct DebtQueueEntry {
    /// Amount of bad debt
    /// Unit: Rad
    pub debt: u128,

    /// Timestamp when this debt was pushed (for ordering)
    pub pushed_at: u64,
}
```

---

## 8. Surplus Auction Accounts

### 8.1 SurplusAuctionAccount

**PDA:** `SURPLUS_AUCTION_ID.derive(b"surplus_auction" || auction_nonce_bytes)`
**Owner:** `SURPLUS_AUCTION_ID`

```rust
struct SurplusAuctionAccount {
    pub account_type: u8,               // = 70

    pub auction_id: u64,

    /// Amount of LSC being auctioned (fixed)
    /// Unit: Rad
    pub amount_to_sell: u128,

    /// Current highest bid in LOGOS
    /// Unit: Wad
    pub bid_amount: u128,

    /// Current highest bidder
    pub high_bidder: [u8; 32],

    /// Timestamp of last bid
    pub bid_time: u64,

    /// Auction end time
    pub auction_deadline: u64,

    /// Min bid duration (restart clock if new bid)
    pub bid_duration: u64,

    /// Minimum bid increase (e.g. 5% = 50_000_000_000_000_000)
    /// Unit: Wad
    pub bid_increase: u128,

    /// Whether auction is settled
    pub settled: bool,

    pub _reserved: [u8; 16],
}
```

---

## 9. Oracle Program Accounts

### 9.1 OracleConfigAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"oracle_config")`
**Owner:** `ORACLE_PROGRAM_ID`

```rust
struct OracleConfigAccount {
    pub account_type: u8,               // = 90

    /// Governance account that can add/remove feeds and keepers
    pub governance_id: [u8; 32],

    /// Maximum allowed age of a price feed (seconds)
    /// e.g. 3600 (1 hour)
    pub max_feed_age: u64,

    /// Minimum number of feeds required for a valid median
    /// e.g. 3
    pub min_feeds_for_median: u8,

    /// Total number of feed accounts registered
    pub feed_count: u16,

    pub _reserved: [u8; 32],
}
```

### 9.2 OracleFeedAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"feed" || feed_id_bytes)`
**Owner:** `ORACLE_PROGRAM_ID`

```rust
struct OracleFeedAccount {
    pub account_type: u8,               // = 91

    /// Feed identifier (e.g. 0 = LOGOS/USD feed 1, 1 = LOGOS/USD feed 2)
    pub feed_id: u64,

    /// Human-readable symbol (e.g. "LOGOS/USD")
    pub symbol: [u8; 32],

    /// Account authorized to submit price updates
    pub keeper_id: [u8; 32],

    /// Current price
    /// Unit: Wad (USD price with 18 decimals)
    pub price: u128,

    /// Price confidence interval / uncertainty
    /// Unit: Wad
    pub confidence: u128,

    /// Timestamp of this price observation (Unix seconds)
    pub timestamp: u64,

    /// Whether this feed is active
    pub active: bool,

    pub _reserved: [u8; 16],
}
```

### 9.3 MedianOracleAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"median" || symbol_bytes)`
**Owner:** `ORACLE_PROGRAM_ID`

```rust
struct MedianOracleAccount {
    pub account_type: u8,               // = 92

    /// Symbol this median aggregates (e.g. "LOGOS/USD")
    pub symbol: [u8; 32],

    /// Feed account IDs contributing to this median (up to 8)
    pub feed_ids: [[u8; 32]; 8],
    pub feed_count: u8,

    /// Computed median price
    /// Unit: Wad
    pub median_price: u128,

    /// Timestamp of most recent contributing feed update
    pub last_update_timestamp: u64,

    /// Median of contributing feed timestamps
    /// Used as the "current time" for all programs
    pub median_timestamp: u64,

    /// Whether the median is valid (enough active feeds)
    pub valid: bool,

    pub _reserved: [u8; 16],
}
```

---

## 10. Global Settlement Accounts

### 10.1 GlobalSettlementStateAccount

**PDA:** `GLOBAL_SETTLEMENT_ID.derive(b"settlement_state")`
**Owner:** `GLOBAL_SETTLEMENT_ID`

```rust
struct GlobalSettlementStateAccount {
    pub account_type: u8,               // = 100

    /// Whether settlement has been triggered
    pub active: bool,

    /// Timestamp when settlement was triggered
    pub triggered_at: u64,

    /// Governance account that triggered settlement
    pub triggered_by: [u8; 32],

    /// Final redemption price at settlement
    /// Unit: Ray
    pub final_redemption_price: u128,

    /// List of collateral types registered for processing
    pub collateral_types: [[u8; 32]; 16],
    pub collateral_type_count: u8,

    /// Number of collateral types fully processed
    pub processed_count: u8,

    /// Total outstanding LSC at settlement
    /// Unit: Rad
    pub total_outstanding_lsc: u128,

    pub _reserved: [u8; 32],
}
```

### 10.2 CollateralRedemptionAccount

**PDA:** `GLOBAL_SETTLEMENT_ID.derive(b"redemption" || collateral_type_id)`
**Owner:** `GLOBAL_SETTLEMENT_ID`

```rust
struct CollateralRedemptionAccount {
    pub account_type: u8,               // = 101

    pub collateral_type_id: [u8; 32],

    /// Collateral price at settlement (set by oracle at shutdown time)
    /// Unit: Wad (USD)
    pub final_collateral_price: u128,

    /// Amount of collateral available for redemption
    /// Unit: Wad
    pub collateral_available: u128,

    /// Total LSC redeemed against this collateral (for accounting)
    /// Unit: Rad
    pub total_lsc_redeemed: u128,

    /// Rate: collateral per LSC (computed post-processing)
    /// Unit: Wad (collateral per LSC)
    pub collateral_per_lsc: u128,

    pub _reserved: [u8; 16],
}
```

---

## 11. Serialization Notes

1. All structs must be serialized with **Borsh** with no padding (Borsh is a canonical schema).
2. The first byte of every account data is `account_type` — a discriminator. Programs must check this on all passed-in accounts to prevent account confusion attacks.
3. `i128` fields (signed) are serialized as two's-complement little-endian 16 bytes (Borsh default).
4. `[u8; 32]` is serialized as 32 raw bytes (no length prefix — Borsh fixed arrays).
5. `[[u8; 32]; N]` is serialized as N * 32 bytes (no length prefix).
6. `bool` is serialized as a single byte: 0 = false, 1 = true.

---

## 12. Account Size Reference

| Account | Approx Size (bytes) |
|---|---|
| SystemParamsAccount | ~320 |
| CollateralTypeAccount | ~256 |
| SafeAccount | ~256 |
| OracleRelayerParamsAccount | ~256 |
| PIControllerStateAccount | ~256 |
| TaxCollectorParamsAccount | ~128 |
| LiquidationEngineParamsAccount | ~192 |
| LiquidationAccount | ~128 |
| CollateralAuctionHouseParamsAccount | ~256 |
| CollateralAuctionAccount | ~192 |
| AccountingEngineParamsAccount | ~192 |
| SystemLogosHoldingAccount | ~64 |
| SystemDebtQueueAccount | ~4112 |
| SurplusAuctionAccount | ~192 |
| OracleConfigAccount | ~128 |
| OracleFeedAccount | ~128 |
| MedianOracleAccount | ~384 |
| GlobalSettlementStateAccount | ~512 |
| CollateralRedemptionAccount | ~128 |
