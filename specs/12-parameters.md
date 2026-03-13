# LSC — System Parameters

## Overview

This document catalogs all tunable parameters in the LSC system. Each parameter includes:
- Name (as used in code)
- Type and unit
- Description
- Recommended initial value
- Allowed range
- Who can modify

**Governance** in v1 = the governance multisig account stored in `SystemParamsAccount.governance_id`.

All ray values use 27 decimal places (RAY = 10^27 = 1.0).
All wad values use 18 decimal places (WAD = 10^18 = 1.0).

---

## 1. LSC Engine — System Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `governance_id` | AccountId | Governance multisig account | Set at deploy | Any valid AccountId | Current governance |
| `settlement_active` | bool | Whether global settlement is triggered | `false` | Set to `true` only once | GlobalSettlement only |

---

## 2. LSC Engine — Collateral Type Parameters

For v1, one collateral type (LOGOS):

### 2.1 Safety Ratio

| Field | `safety_ratio` |
|---|---|
| Type | `u128` (Ray) |
| Description | Minimum collateralization ratio required to generate debt. A SAFE must have `collateral_value >= debt * safety_ratio` to mint LSC. |
| Formula | `safety_ratio = 1.5` → `1_500_000_000_000_000_000_000_000_000` (Ray) |
| Initial Value | `1_500_000_000_000_000_000_000_000_000` (150%) |
| Allowed Range | `[1_100_000_000_000_000_000_000_000_000, 3_000_000_000_000_000_000_000_000_000]` (110% – 300%) |
| Who Can Modify | Governance |

### 2.2 Liquidation Ratio

| Field | `liquidation_ratio` |
|---|---|
| Type | `u128` (Ray) |
| Description | Minimum collateralization ratio before liquidation is allowed. Must be ≤ safety_ratio. |
| Formula | `liquidation_ratio = 1.35` → `1_350_000_000_000_000_000_000_000_000` |
| Initial Value | `1_350_000_000_000_000_000_000_000_000` (135%) |
| Allowed Range | `[1_050_000_000_000_000_000_000_000_000, safety_ratio]` (105% – safety_ratio) |
| Who Can Modify | Governance |

### 2.3 Liquidation Penalty

| Field | `liquidation_penalty` |
|---|---|
| Type | `u128` (Wad) |
| Description | Additional collateral seized as a liquidation fee, expressed as a fraction. E.g., 13% means the auction must raise debt + 13% of debt in LSC. |
| Formula | `0.13` → `130_000_000_000_000_000` (Wad) |
| Initial Value | `130_000_000_000_000_000` (13%) |
| Allowed Range | `[0, 300_000_000_000_000_000]` (0% – 30%) |
| Who Can Modify | Governance |

### 2.4 Debt Ceiling

| Field | `debt_ceiling` |
|---|---|
| Type | `u128` (Rad) |
| Description | Maximum total LSC that can be minted against this collateral type. Expressed in Rad (10^45). |
| Formula | 10 million LSC ceiling → `10_000_000 * RAY * WAD` = `10_000_000 * 10^45` |
| Initial Value | `10_000_000_000_000_000_000_000_000_000_000_000_000_000_000_000_000` (10M LSC) |
| Allowed Range | `[debt_floor, 10^60]` (effectively unlimited ceiling in Rad) |
| Who Can Modify | Governance |

### 2.5 Debt Floor

| Field | `debt_floor` |
|---|---|
| Type | `u128` (Rad) |
| Description | Minimum debt per SAFE. SAFEs below this amount (after generation) are rejected. Prevents dust SAFEs. |
| Formula | 100 LSC floor → `100 * RAY * WAD` |
| Initial Value | `100_000_000_000_000_000_000_000_000_000_000_000_000_000_000_000` (100 LSC) |
| Allowed Range | `[RAY, 1_000 * RAY * WAD]` (1 LSC – 1000 LSC) |
| Who Can Modify | Governance |

### 2.6 Stability Fee

| Field | `stability_fee` |
|---|---|
| Type | `u128` (Ray, per-second rate) |
| Description | Per-second interest rate on SAFE debt. Compounded with `accumulated_rate`. |
| Computation | `annual_rate = (1 + r)^(365*24*3600) - 1`, solve for `r` given annual rate |
| Formula | For 2% APY: `r = (1.02)^(1/31536000) - 1 ≈ 6.27e-10` per second → `RAY + 627_000_000_000_000_000` |
| Initial Value | `1_000_000_000_627_000_000_000_000_000` (≈ 2% APY) |
| Allowed Range | `[RAY, RAY + 21_979_553_151_200_000_000_000_000]` (0% – 100% APY) |
| Who Can Modify | Governance |

### Per-Second Rate Reference Table

| Annual Rate | Per-Second Ray Value |
|---|---|
| 0% | `1_000_000_000_000_000_000_000_000_000` (RAY) |
| 0.5% | `1_000_000_000_158_000_000_000_000_000` |
| 1% | `1_000_000_000_315_522_921_573_372_800` |
| 2% | `1_000_000_000_627_000_000_000_000_000` |
| 5% | `1_000_000_001_547_125_957_863_212_600` |
| 10% | `1_000_000_003_022_265_980_097_387_650` |
| 20% | `1_000_000_005_781_378_656_804_591_700` |

---

## 3. Tax Collector Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `global_stability_fee` | `u128` (Ray) | Base per-second fee applied to all collateral types additively | `RAY` (0%) | `[RAY, RAY + 10% APY rate]` | Governance |

Note: The effective stability fee for a collateral type is `max(global_stability_fee, collateral_type.stability_fee)` or additive — implementation must specify. Recommended: additive (`effective = global + type_specific - RAY`).

---

## 4. Oracle Program Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `max_feed_age` | `u64` (seconds) | Maximum age of a feed price before it's considered stale | `3600` (1 hour) | `[60, 86400]` (1 min – 1 day) | Governance |
| `min_feeds_for_median` | `u8` | Minimum number of active feeds for the median to be valid | `3` | `[1, 8]` | Governance |
| `max_timestamp_jump` | `u64` (seconds) | Maximum jump per single price submission | `7200` (2 hours) | `[300, 86400]` | Governance |

---

## 5. Oracle Relayer Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `initial_redemption_price` | `u128` (Ray) | Starting redemption price for LSC | `3_140_000_000_000_000_000_000_000_000` ($3.14) | Set at deploy | Governance (one-time) |
| `redemption_rate` | `u128` (Ray) | Current per-second redemption rate (set by PI controller) | `RAY` | Set by PIController | PIController only |
| `redemption_rate_lower_bound` | `u128` (Ray) | Minimum allowed redemption rate | See PI Controller §4 | `[RAY - 50% APY, RAY]` | Governance |
| `redemption_rate_upper_bound` | `u128` (Ray) | Maximum allowed redemption rate | See PI Controller §4 | `[RAY, RAY + 50% APY]` | Governance |
| `collateral_price_safety_margin` | `u128` (Ray) | Discount applied to collateral price for liquidation | `995_000_000_000_000_000_000_000_000` (99.5%) | `[900_000_000_000_000_000_000_000_000, RAY]` (90% – 100%) | Governance |
| `max_allowed_lag` | `u64` (seconds) | Max seconds oracle timestamp can trail current time | `86400` (1 day) | `[3600, 604800]` (1 hour – 1 week) | Governance |

---

## 6. PI Controller Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `kp` | `i128` (Ray) | Proportional gain | `5_000_000_000_000_000_000_000_000` (5e24) | `[-10^27, 10^27]` | Governance |
| `ki` | `i128` (Ray) | Integral gain | `1_000_000_000_000_000_000_000` (1e21) | `[-10^25, 10^25]` | Governance |
| `per_second_cumulative_leak` | `u128` (Ray) | Integral decay factor per second | `999_999_920_000_000_000_000_000_000` | `[990_000_000_000_000_000_000_000_000, RAY]` (99% – 100% per second) | Governance |
| `update_delay` | `u64` (seconds) | Minimum seconds between PI updates | `3600` (1 hour) | `[60, 86400]` (1 min – 1 day) | Governance |
| `noise_barrier` | `u128` (Wad) | Minimum error to trigger rate change | `5_000_000_000_000_000` (0.5%) | `[0, 50_000_000_000_000_000]` (0% – 5%) | Governance |

### Redemption Rate Bounds (also set in OracleRelayer)

| Parameter | Initial Value | Notes |
|---|---|---|
| `redemption_rate_lower_bound` | `999_999_681_000_000_000_000_000_000` | ≈ -1% APY |
| `redemption_rate_upper_bound` | `1_000_000_317_000_000_000_000_000_000` | ≈ +1% APY |

Start with tight bounds; widen after observing system behavior.

---

## 7. Liquidation Engine Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `on_auction_system_coin_limit` | `u128` (Rad) | Circuit breaker: max LSC that can be in active auctions simultaneously | `1_000_000 * RAY * WAD` (1M LSC) | `[100 * RAY * WAD, ∞]` | Governance |

---

## 8. Collateral Auction House Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `auction_discount` | `u128` (Wad) | Fixed discount buyers receive on collateral | `50_000_000_000_000_000` (5%) | `[10_000_000_000_000_000, 300_000_000_000_000_000]` (1% – 30%) | Governance |
| `max_discount` | `u128` (Wad) | Maximum discount (for restarted auctions) | `200_000_000_000_000_000` (20%) | `[auction_discount, 500_000_000_000_000_000]` (up to 50%) | Governance |
| `min_discount` | `u128` (Wad) | Minimum discount | `10_000_000_000_000_000` (1%) | `[0, auction_discount]` | Governance |
| `minimum_bid` | `u128` (Wad) | Minimum collateral per purchase | `10_000_000_000_000_000_000` (10 LOGOS) | `[0, ∞]` | Governance |
| `auction_ttl` | `u64` (seconds) | Auction duration before restart is allowed | `1800` (30 min) | `[300, 86400]` (5 min – 1 day) | Governance |

---

## 9. Accounting Engine Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `surplus_auction_amount` | `u128` (Rad) | LSC sold per surplus auction | `10_000 * RAY * WAD` (10k LSC) | `[1_000 * RAY * WAD, 1_000_000 * RAY * WAD]` | Governance |
| `surplus_buffer` | `u128` (Rad) | Surplus buffer kept before auctioning | `500_000 * RAY * WAD` (500k LSC) | `[0, ∞]` | Governance |
| `surplus_auction_delay` | `u64` (seconds) | Min time between surplus auctions | `86400` (1 day) | `[3600, 2592000]` (1 hour – 30 days) | Governance |

---

## 10. Surplus Auction House Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `bid_increase` | `u128` (Wad) | Minimum bid increment (LOGOS) | `50_000_000_000_000_000` (5%) | `[10_000_000_000_000_000, 300_000_000_000_000_000]` (1% – 30%) | Governance |
| `bid_duration` | `u64` (seconds) | Extension window after each bid | `10800` (3 hours) | `[600, 86400]` (10 min – 1 day) | Governance |
| `total_auction_length` | `u64` (seconds) | Hard deadline from auction start | `259200` (3 days) | `[3600, 604800]` (1 hour – 7 days) | Governance |

---

## 11. Global Settlement Parameters

| Parameter | Type | Description | Initial Value | Range | Who Can Modify |
|---|---|---|---|---|---|
| `safe_redemption_period` | `u64` (seconds) | Time SAFE owners have to redeem before LSC holders can | `259200` (3 days) | `[86400, 604800]` (1 day – 7 days) | Governance (pre-settlement) |

---

## 12. Constants (Not Tunable)

These are hardcoded constants that cannot be changed without a program upgrade:

| Constant | Value | Description |
|---|---|---|
| `RAY` | `1_000_000_000_000_000_000_000_000_000` (10^27) | Fixed-point ray unit |
| `WAD` | `1_000_000_000_000_000_000` (10^18) | Fixed-point wad unit |
| `RAD` | `RAY * WAD` = 10^45 | Fixed-point rad unit |
| `MAX_OPERATORS_PER_SAFE` | `4` | Maximum operator accounts per SAFE |
| `MAX_COLLATERAL_TYPES` | `16` | Maximum collateral types in GlobalSettlement |
| `MAX_FEEDS_PER_MEDIAN` | `8` | Maximum feeds per MedianOracle |
| `MAX_DEBT_QUEUE_ENTRIES` | `256` | Maximum entries in SystemDebtQueue |
| `MAX_CHAINED_CALLS` | `10` | LEZ limit on chained calls per transaction |
| `INITIAL_REDEMPTION_PRICE` | `3_140_000_000_000_000_000_000_000_000` | Starting LSC redemption price ($3.14 in Ray) |

---

## 13. Parameter Dependencies and Constraints

The following constraints must hold between parameters at all times. Governance must verify these after any parameter update:

```
safety_ratio >= liquidation_ratio >= RAY + 50_000_000_000_000_000_000_000_000

liquidation_penalty < WAD  (< 100%)

debt_floor <= debt_ceiling

stability_fee >= RAY  (non-negative fee)

collateral_price_safety_margin <= RAY

redemption_rate_lower_bound <= RAY <= redemption_rate_upper_bound

per_second_cumulative_leak <= RAY  (non-amplifying)

noise_barrier < WAD

auction_discount >= min_discount
max_discount >= auction_discount

surplus_auction_amount <= surplus_buffer + surplus_auction_amount  (buffer meaningful)
```

---

## 14. Recommended Parameter Review Schedule

| Frequency | Review |
|---|---|
| Daily (automated) | Oracle keeper health, price deviations, accumulated rate growth |
| Weekly | Safety ratio utilization (how close SAFEs are to liquidation_ratio on average) |
| Monthly | PI controller gains (are oscillations damping? is the integral growing unchecked?) |
| Quarterly | Debt ceiling (increase as TVL grows), stability fee (market rate comparison) |
| As needed | On any significant market event: oracle update, sharp LOGOS price move, governance proposal |
