# LSC — Tax Collector

## Overview

The **TaxCollector** is responsible for accruing per-second stability fees to active collateral types. It is called by permissionless keepers (off-chain bots) and chains to `LSCEngine::UpdateAccumulatedRate` to apply the fee.

When `AccrueStabilityFee` is called for a collateral type:
1. It computes elapsed time since the last fee update.
2. It applies the effective per-second fee rate (global base + per-collateral rate) via binary exponentiation (`rpow`).
3. It delegates the accumulated_rate update to `LSCEngine::UpdateAccumulatedRate` via a chained call, which also mints surplus LSC to the `system_surplus_id`.

**Program ID alias:** `TAX_COLLECTOR_ID`

---

## 1. Account Schema

### 1.1 TaxCollectorParamsAccount

**PDA:** `TAX_COLLECTOR_ID.derive(compute_pda_seed(b"tax_collector_params"))`
**Owner:** `TAX_COLLECTOR_ID`

```rust
struct TaxCollectorParamsAccount {
    pub account_type: u8,               // = 30

    /// Governance account (can update global_stability_fee)
    pub governance_id: [u8; 32],

    /// Accounting Engine's system surplus account ID
    /// (where accrued stability fees are minted as LSC)
    pub accounting_engine_surplus_id: [u8; 32],

    /// The LSC Engine's system params account (for privilege check)
    pub lsc_engine_params_id: [u8; 32],

    /// Global stability fee (base rate applied to ALL collateral types additively)
    /// Combined with per-collateral CollateralTypeAccount.stability_fee
    /// e.g. RAY (1.0 = 0% per second additional)
    /// Unit: Ray; must be >= RAY
    pub global_stability_fee: u128,

    pub _reserved: [u8; 32],
}
```

---

## 2. Instruction Enum

```rust
enum TaxCollectorInstruction {
    /// One-time initialization
    Initialize {
        governance_id: [u8; 32],
        accounting_engine_surplus_id: [u8; 32],
        lsc_engine_params_id: [u8; 32],
        global_stability_fee: u128,  // Ray; must be >= RAY
    },

    /// Accrue stability fee for one collateral type (permissionless keeper instruction)
    AccrueStabilityFee,

    /// Update global_stability_fee (governance only)
    UpdateParams {
        new_global_stability_fee: Option<u128>,  // Ray
    },
}
```

---

## 3. Instructions: Detailed Specification

### 3.1 `Initialize`

**Description:** One-time initialization. Creates `TaxCollectorParamsAccount`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `tax_collector_params_id` (PDA) | Yes | PDA (new) | Created here |
| 1 | `governance_account` | No | Yes | Initial governance |

**Validations:**
- `tax_collector_params_id` must be uninitialized
- `global_stability_fee >= RAY` (non-negative rate; RAY = 1.0 = 0% fee)

**State Transition:**
```
params.account_type = 30
params.governance_id = governance_account.id
params.accounting_engine_surplus_id = accounting_engine_surplus_id
params.lsc_engine_params_id = lsc_engine_params_id
params.global_stability_fee = global_stability_fee
```

**Errors:**
- `AlreadyInitialized`
- `InvalidRate` — global_stability_fee < RAY

---

### 3.2 `AccrueStabilityFee`

**Description:** Computes elapsed stability fee for one collateral type and delegates the `accumulated_rate` update to `LSCEngine::UpdateAccumulatedRate` via chained call.

This is **permissionless** — any keeper can call it at any time. It is a no-op if the oracle timestamp has not advanced (elapsed = 0).

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `tax_collector_params_id` | No | — | For global_stability_fee |
| 1 | `collateral_type_id` | No | — | For stability_fee, accumulated_rate, last_update |
| 2 | `global_debt_id` | No | — | Passed through to LSCEngine chained call |
| 3 | `system_params_id` | No | — | For LSCEngine authorization check |
| 4 | `accounting_engine_surplus_id` | No | — | Passed through to LSCEngine (mint target) |
| 5 | `lsc_token_def_id` | No | — | Passed through to LSCEngine (for minting) |
| 6 | `lsc_market_oracle_id` | No | — | For current timestamp |

**Preconditions:**
```
now = lsc_market_oracle.median_timestamp
elapsed = now - collateral_type.last_stability_fee_update
assert!(elapsed > 0, "No time elapsed since last fee update");
assert!(lsc_market_oracle.valid, "Oracle must be valid");
```

**Stability Fee Computation:**

The effective per-second rate is the combination of the global base rate and the per-collateral type rate:

```
// Both rates are Ray-valued per-second multipliers: (RAY + annual_fee_fraction / seconds_per_year)
// To combine two per-second multipliers, subtract one RAY to avoid double-counting the 1.0 base:
effective_rate_ray = params.global_stability_fee + collateral_type.stability_fee - RAY

// Compute rate^elapsed using binary exponentiation in Ray arithmetic:
// rpow(base, exp, precision) where base and result are in Ray units
rate_multiplier = rpow(effective_rate_ray, elapsed, RAY)

// New accumulated rate:
new_accumulated_rate = ray_mul(collateral_type.accumulated_rate, rate_multiplier)
// ray_mul(x, y) = x * y / RAY  (using u256 intermediate)
// accumulated_rate (Ray) * rate_multiplier (Ray) / RAY = new accumulated_rate (Ray)
```

**rpow Implementation Note:**
```rust
/// Ray exponentiation using binary (fast) exponentiation.
/// base: Ray value (e.g. 1_000_000_000_000_000_000_000_000_001 for tiny fee)
/// exp: integer exponent (seconds elapsed)
/// one: RAY (1e27)
///
/// MUST use u256 intermediates for all multiplications to avoid overflow.
fn rpow(base: u128, exp: u64, one: u128) -> u128 {
    if exp == 0 { return one; }
    let mut result: u256 = one as u256;
    let mut b: u256 = base as u256;
    let mut n = exp;
    while n > 1 {
        if n % 2 == 1 {
            result = result * b / (one as u256);
        }
        b = b * b / (one as u256);
        n /= 2;
    }
    (result * b / (one as u256)) as u128
}
```

**Chained Calls:**
```
// Delegate the accumulated_rate update to LSCEngine.
// LSCEngine will:
//   1. Update collateral_type.accumulated_rate = new_accumulated_rate
//   2. Update global_debt.total_debt += rate_increase * global_normalized_debt
//   3. Mint surplus LSC to accounting_engine_surplus_id
//
// Authorization: include tax_collector_params PDA seed so LSCEngine recognizes the caller.
ChainedCall → LSCEngine::UpdateAccumulatedRate {
    new_accumulated_rate: new_accumulated_rate,
}
  accounts: [
    collateral_type_id (writable),
    global_debt_id (writable),
    system_params_id (read),
    tax_collector_params_id (read, is_authorized via PDA),
    accounting_engine_surplus_id (writable),
    lsc_token_def_id (writable),
  ]
  pda_seeds: [compute_pda_seed(b"tax_collector_params")]
  // Runtime derives tax_collector_params_id = TAX_COLLECTOR_ID.derive(that_seed)
  // and marks it is_authorized = true in the LSCEngine callee.
  // LSCEngine verifies: system_params.tax_collector_params_id == tax_collector_params_id.id
```

**Errors:**
- `NoTimeElapsed` — elapsed == 0 (oracle timestamp unchanged)
- `OracleInvalid` — median oracle not valid
- `ArithmeticOverflow` — rpow or ray_mul overflow

---

### 3.3 `UpdateParams`

**Description:** Governance updates the global stability fee.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `tax_collector_params_id` | Yes | — | Params to update |
| 1 | `governance_account` | No | Yes | Must match params.governance_id |

**Validations:**
- `governance_account.id == params.governance_id`
- `governance_account.is_authorized == true`
- If `new_global_stability_fee.is_some()`: value >= RAY

**State Transition:**
Apply `Some(v)` fields; leave `None` fields unchanged.

**Errors:**
- `Unauthorized`
- `InvalidRate`

---

## 4. Integration with LSCEngine

The TaxCollector and LSCEngine interact as follows:

```
Keeper calls: TaxCollector::AccrueStabilityFee
  │
  ├─ reads: tax_collector_params (global_stability_fee)
  ├─ reads: collateral_type (stability_fee, accumulated_rate, last_update)
  ├─ reads: lsc_market_oracle (current timestamp)
  ├─ computes: new_accumulated_rate = old_rate * rpow(effective_fee, elapsed)
  │
  └─ ChainedCall → LSCEngine::UpdateAccumulatedRate { new_accumulated_rate }
       │
       ├─ verifies tax_collector_params_id is authorized (PDA check)
       ├─ updates collateral_type.accumulated_rate = new_accumulated_rate
       ├─ updates collateral_type.last_stability_fee_update = now
       ├─ computes surplus_rad = (new_rate - old_rate) * global_normalized_debt
       ├─ updates global_debt.total_debt += surplus_rad
       │
       └─ ChainedCall → TokenProgram::Mint { surplus_wad }
            accounts: [lsc_token_def_id (writable, is_authorized), system_surplus_id (writable)]
            pda_seeds: [compute_pda_seed(b"lsc_token_definition")]
```

**Call depth:** 2 chained calls deep from the user's perspective (TaxCollector → LSCEngine → TokenProgram), totaling 3 program invocations within the LEZ limit of 10.

---

## 5. Stability Fee Rates

The effective per-second stability fee is the multiplicative combination of:
1. `params.global_stability_fee` — applies to all collateral types
2. `collateral_type.stability_fee` — per-collateral additional fee

Both are Ray-valued per-second multipliers. Combining:
```
effective = global_fee + per_type_fee - RAY
```

This is equivalent to: `effective_APY ≈ global_APY + per_type_APY`.

**Example values:**

| Annual Fee | Per-second Ray value |
|---|---|
| 0% (neutral) | `RAY` = 1_000_000_000_000_000_000_000_000_000 |
| 1% APY | `RAY + 315_522_921_573_372_800` |
| 2% APY | `RAY + 627_937_192_004_702_900` |
| 5% APY | `RAY + 1_547_125_491_916_498_900` |

A collateral type with `global_stability_fee = RAY + 1% per-second` and `stability_fee = RAY + 2% per-second` has an effective annual rate of approximately 3%.

---

## 6. Keeper Requirements

Any off-chain keeper can call `AccrueStabilityFee`. Keepers should call this:
- At least once per oracle update cycle (after price feeds are refreshed)
- Before any significant operation that depends on up-to-date accumulated_rate (e.g., before checking collateralization ratios)

There is no minimum interval enforced on-chain (unlike the PI Controller's `update_delay`). However, very frequent calls with `elapsed = 0` are no-ops and waste gas.

---

## 7. Error Codes

```rust
enum TaxCollectorError {
    AlreadyInitialized    = 6500,
    Unauthorized          = 6501,
    NoTimeElapsed         = 6502,
    OracleInvalid         = 6503,
    InvalidRate           = 6504,
    ArithmeticOverflow    = 6505,
    InvalidPDA            = 6506,
}
```
