# LSC — PI Controller (Redemption Rate Controller)

## Overview

The PI Controller is the core innovation of LSC (and RAI). It is a **Proportional-Integral feedback controller** that continuously adjusts the **redemption rate** to push the LSC market price toward the redemption price.

The controller does **not** target a fixed USD value. Instead, it minimizes the relative error between:
- `p_r` — the redemption price (protocol's internal target)
- `p_m` — the market price (what LSC actually trades at)

**Program ID alias:** `PI_CONTROLLER_ID`

---

## 1. Mathematical Model

### 1.1 Error Term

At each update time `t`, the **proportional error** is:

```
e(t) = (p_r(t) - p_m(t)) / p_r(t)
```

Where:
- `p_r(t)` = current redemption price (Ray, USD per LSC)
- `p_m(t)` = current LSC market price (Wad, USD per LSC)
- `e(t)` is dimensionless, signed

When `p_m < p_r` (market below target): `e > 0` → rate increases → debt more expensive → holders rewarded → price rises.
When `p_m > p_r` (market above target): `e < 0` → rate decreases (goes negative) → debt cheaper → more minting → price falls.

### 1.2 Integral Term Accumulation

The error integral accumulates over time:

```
I(t) = I(t - Δt) * α^Δt + e(t) * Δt
```

Where:
- `Δt = t - t_last` (seconds since last update)
- `α` = **per-second leak factor** (dampens integral windup). `α < 1` causes the integral to decay exponentially if not fed. Stored as `per_second_cumulative_leak` (Ray).
- `α^Δt` is computed using `rpow(α, Δt, RAY)`

The leak prevents infinite integral windup: old errors become less relevant over time.

### 1.3 Redemption Rate Computation

The raw (unclamped) redemption rate output is:

```
raw_rate(t) = RAY + Kp * e(t) + Ki * I(t)
```

Where:
- `RAY` = 10^27 (the "1.0" baseline — neutral rate means price stays constant)
- `Kp` = proportional gain (signed i128, Ray units)
- `Ki` = integral gain (signed i128, Ray units per second)
- The result is a **per-second multiplier** for the redemption price

**Important:** `Kp * e(t)` and `Ki * I(t)` are both dimensionless corrections to the per-second multiplier, making `raw_rate` a Ray value.

### 1.4 Rate Clamping

The output is clamped to safe bounds:

```
redemption_rate(t) = clamp(raw_rate(t), lower_bound, upper_bound)
```

Where bounds are stored in `OracleRelayerParamsAccount`:
- `lower_bound` — minimum per-second rate (prevents catastrophic deflation)
- `upper_bound` — maximum per-second rate (prevents catastrophic inflation)

### 1.5 Redemption Price Evolution

After the new rate is set, the redemption price evolves according to:

```
p_r(t + Δt) = p_r(t) * redemption_rate(t)^Δt
```

This is applied lazily by `OracleRelayer::UpdateRedemptionPrice` each time the price is read.

### 1.6 Noise Barrier

If the absolute error is below the noise barrier, no update is performed:

```
if |e(t)| < noise_barrier:
    skip update (return without changing rate)
```

This prevents unnecessary thrashing when the market is already close to the redemption price.

---

## 2. Example Numeric Walkthrough

**Setup:**
- `p_r = 3.14 * RAY` (redemption price = $3.14)
- `p_m = 3.00 * WAD` (market price = $3.00)
- `Kp = 5_000_000_000_000_000_000_000_000` (= 5e24, **positive** gain in Ray)
  - Convention: `Kp > 0`, `error = redemption - market`, positive error → rate increases (stabilizing)

```
e(t) = (p_r - p_m_ray) / p_r
     = (3.14 * RAY - 3.00 * RAY) / (3.14 * RAY)
     = 0.14 / 3.14 ≈ 0.04459 (dimensionless)

// Convert to Ray units:
e_ray = 0.04459 * RAY ≈ 44_585_987_261_146_496_815_286_624

// With Kp = 5e24 (Ray), using 256-bit intermediate:
proportional_term = i256(Kp) * i256(e_ray) / i256(RAY)
                  ≈ 5e24 * 4.459e25 / 1e27
                  ≈ 2.23e23

// raw_rate = RAY + proportional_term ≈ 1.000000000223... per second
// Annualized: ~0.7% APY rate increase → incentivizes SAFEs to close → price rises
```

LSC market is below target → positive error → positive correction → rate rises above RAY → correct negative feedback ✓

---

## 3. Implementation Specification

### 3.1 Instruction Enum

```rust
enum PIControllerInstruction {
    /// One-time initialization
    Initialize {
        kp: i128,                           // Ray (proportional gain, signed; positive for negative feedback)
        ki: i128,                           // Ray (integral gain, signed)
        per_second_cumulative_leak: u128,   // Ray (leak factor ≤ RAY)
        update_delay: u64,                  // seconds between updates
        noise_barrier: u128,                // Wad (min error to act on)
        oracle_relayer_params_id: [u8; 32],
        lsc_market_oracle_id: [u8; 32],
        system_params_id: [u8; 32],         // for settlement_active check in UpdateRate
    },

    /// Update redemption rate — the main operation. Callable by anyone.
    UpdateRate,

    /// Update controller parameters (governance only)
    UpdateParams {
        new_kp: Option<i128>,
        new_ki: Option<i128>,
        new_per_second_cumulative_leak: Option<u128>,
        new_update_delay: Option<u64>,
        new_noise_barrier: Option<u128>,
    },

    /// Reset integral accumulator (governance, emergency use)
    ResetIntegral,
}
```

---

### 3.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `pi_controller_state_id` (PDA) | Yes | PDA (new) | State account |
| 1 | `governance_account` | No | Yes | Initial governance |
| 2 | `lsc_market_oracle_id` | No | — | For initial timestamp |

**Validations:**
- `pi_controller_state_id` uninitialized
- `per_second_cumulative_leak <= RAY` (leak must not amplify)
- `update_delay > 0`
- `noise_barrier < WAD` (< 100%)

**State Transition:**
```
pi_controller_state.account_type = 20
pi_controller_state.kp = kp
pi_controller_state.ki = ki
pi_controller_state.per_second_cumulative_leak = per_second_cumulative_leak
pi_controller_state.update_delay = update_delay
pi_controller_state.noise_barrier = noise_barrier
pi_controller_state.error_integral = 0
pi_controller_state.last_update_time = lsc_market_oracle.median_timestamp
pi_controller_state.oracle_relayer_params_id = oracle_relayer_params_id
pi_controller_state.lsc_market_oracle_id = lsc_market_oracle_id
pi_controller_state.system_params_id = system_params_id
pi_controller_state.last_proportional_term = 0
pi_controller_state.last_integral_term = 0
```

---

### 3.3 `UpdateRate`

**Description:** The main controller update. Reads market price and redemption price, computes new rate, stores it via OracleRelayer. Callable by anyone (permissionless), but only executes if enough time has passed and settlement is not active.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `pi_controller_state_id` | Yes | — | Controller state |
| 1 | `oracle_relayer_params_id` | Yes | — | Redemption price + rate |
| 2 | `lsc_market_oracle_id` | No | — | Current LSC market price + timestamp |
| 3 | `system_params_id` | No | — | For settlement_active check |

**Algorithm (pseudocode):**

```rust
fn update_rate(
    state: &mut PIControllerState,
    relayer: &mut OracleRelayerParams,
    market_oracle: &MedianOracle,
    system_params: &SystemParams,
) -> Result<()> {
    // 0. Settlement check: rate must not update during global settlement
    assert!(!system_params.settlement_active, SettlementActive);

    // 1. Validate oracle
    assert!(market_oracle.valid, OracleInvalid);
    let current_time = market_oracle.median_timestamp;

    // 2. Check update delay
    assert!(
        current_time >= state.last_update_time + state.update_delay,
        TooSoon
    );
    let dt = current_time - state.last_update_time;  // u64 seconds

    // 3. Get prices
    // Redemption price must be up-to-date (caller should have called UpdateRedemptionPrice first)
    let p_r: i128 = relayer.redemption_price as i128;  // Ray
    let p_m: i128 = market_oracle.median_price as i128;  // Wad

    // 4. Compute error (dimensionless, in Ray units)
    // e = (p_r - p_m_ray) / p_r
    // Sign convention: positive e means market is BELOW target → rate should INCREASE
    // Convert p_m from Wad to Ray: p_m_ray = p_m * RAY / WAD
    let p_m_ray: i128 = i256::from(p_m) * i256::from(RAY) / i256::from(WAD);
    let error_ray: i128 = (i256::from(p_r - p_m_ray) * i256::from(RAY) / i256::from(p_r))
        .try_into()
        .expect("error_ray overflow");
    // error_ray is in Ray units (dimensionless * RAY)
    // Positive when market < redemption (LSC below target)
    // Negative when market > redemption (LSC above target)

    // 5. Check noise barrier
    if error_ray.unsigned_abs() < (state.noise_barrier as u128 * RAY / WAD) {
        // Within noise — still update time but don't change rate
        state.last_update_time = current_time;
        return Ok(());
    }

    // 6. Update integral with leak (MUST use 256-bit intermediate to avoid overflow)
    // I(t) = I(t-dt) * leak^dt + e * dt
    // leak^dt can be near RAY; error_integral can be large → multiplication can exceed i128::MAX
    let leak_factor: i128 = rpow(state.per_second_cumulative_leak, dt, RAY) as i128;
    let integral_decayed: i128 = (i256::from(state.error_integral) * i256::from(leak_factor)
        / i256::from(RAY as i128))
        .try_into()
        .expect("integral_decayed overflow");
    let integral_new: i128 = integral_decayed
        .checked_add(error_ray.checked_mul(dt as i128).expect("error*dt overflow"))
        .expect("integral_new overflow");
    // Clamp integral to prevent runaway windup:
    let integral_new = integral_new.clamp(i128::MIN / 2, i128::MAX / 2);
    state.error_integral = integral_new;

    // 7. Compute proportional and integral terms (MUST use 256-bit intermediate)
    // kp (Ray) * error_ray (Ray) / RAY = Ray-dimensioned correction
    // Products can reach ~5e24 * 1e27 = 5e51, which overflows i128::MAX (~1.7e38)
    let proportional_term: i128 = (i256::from(state.kp) * i256::from(error_ray)
        / i256::from(RAY as i128))
        .try_into()
        .expect("proportional_term overflow");
    let integral_term: i128 = (i256::from(state.ki) * i256::from(integral_new)
        / i256::from(RAY as i128))
        .try_into()
        .expect("integral_term overflow");

    // 8. Compute raw rate
    // raw_rate = RAY + proportional_term + integral_term
    // With Kp > 0: positive error (market < target) → positive proportional → rate > RAY
    //              negative error (market > target) → negative proportional → rate < RAY
    let raw_rate: i128 = (RAY as i128)
        .checked_add(proportional_term)
        .and_then(|r| r.checked_add(integral_term))
        .expect("raw_rate overflow");

    // 9. Clamp to bounds
    let new_rate: u128 = clamp_i128_to_u128(
        raw_rate,
        relayer.redemption_rate_lower_bound,
        relayer.redemption_rate_upper_bound,
    );

    // 10. Update state
    state.last_update_time = current_time;
    state.last_proportional_term = proportional_term;
    state.last_integral_term = integral_term;

    // 11. Write new rate to OracleRelayer (via chained call)
    relayer.redemption_rate = new_rate;

    Ok(())
}
```

**Note on signs and Kp convention:**
- `error_ray = redemption_price - market_price` (in Ray-normalized units)
  - Positive when market < target (LSC below redemption price)
  - Negative when market > target (LSC above redemption price)
- `Kp > 0` (positive gain, **this is the correct/stable convention**):
  - Market below target → `error > 0` → `proportional > 0` → `rate > RAY` → debt more expensive → less minting → price rises. ✓ (negative feedback, stabilizing)
  - Market above target → `error < 0` → `proportional < 0` → `rate < RAY` → debt cheaper → more minting → price falls. ✓
- `Ki > 0`: integral of sustained positive error pushes rate up further.
- All Ray×Ray multiplications **must** use 256-bit intermediates. The implementation must use `i256`/`u256` for any product of two Ray-magnitude values before dividing by RAY.

**Chained Calls:**
```
OracleRelayer::UpdateRedemptionRate {
    new_redemption_rate: new_rate,
    // accounts: [oracle_relayer_params_id, pi_controller_state_id (PDA auth)]
}
```

**Errors:**
- `TooSoon` — update_delay not elapsed
- `OracleInvalid` — market oracle not valid
- `OracleStale` — market oracle timestamp too old

---

### 3.4 `UpdateParams`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `pi_controller_state_id` | Yes | — | State to update |
| 1 | `governance_account` | No | Yes | Must match system governance |
| 2 | `system_params_id` | No | — | For governance_id check |

**Validations:**
- `governance_account.id == system_params.governance_id`
- `governance_account.is_authorized == true`
- If `new_per_second_cumulative_leak.is_some()`: value <= RAY

**State Transition:**
Apply `Some(v)` fields; leave `None` fields unchanged.

---

### 3.5 `ResetIntegral`

**Description:** Emergency reset of the integral accumulator to zero. Used if the integral has wound up unreasonably (e.g. after oracle failure or extreme market dislocation).

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `pi_controller_state_id` | Yes | — | State to reset |
| 1 | `governance_account` | No | Yes | Governance |
| 2 | `system_params_id` | No | — | For governance check |

**State Transition:**
```
pi_controller_state.error_integral = 0
pi_controller_state.last_proportional_term = 0
pi_controller_state.last_integral_term = 0
```

---

## 4. Parameter Recommendations

| Parameter | Recommended Value | Notes |
|---|---|---|
| `kp` (Kp) | `5_000_000_000_000_000_000_000_000` (5e24, Ray) | Proportional gain (positive). Start conservative. All Ray×Ray products need i256 intermediates. |
| `ki` (Ki) | `1_000_000_000_000_000_000_000` (1e21, Ray/s) | Integral gain. Much smaller than Kp. |
| `per_second_cumulative_leak` | `999_999_920_000_000_000_000_000_000` (≈ RAY * 0.99999992) | ~68% decay per 1 year |
| `update_delay` | `3600` (1 hour) | Rate updated at most once per hour |
| `noise_barrier` | `5_000_000_000_000_000` (5e15, Wad = 0.5%) | Don't react to < 0.5% error |
| `redemption_rate_lower_bound` | `999_999_681_000_000_000_000_000_000` (≈ -1% APY) | Per-second lower bound |
| `redemption_rate_upper_bound` | `1_000_000_315_000_000_000_000_000_000` (≈ +1% APY) | Per-second upper bound |

**Rate bounds annualized:**
- `RAY + 315_522_921_573_372_800` ≈ 1% APY (one percent per year)
- `RAY - 317_097_919_837_645_900` ≈ -1% APY
- Upper bound: ~+100% APY → `RAY + 21_979_553_151_200_000_000_000_000`
- Lower bound: ~-50% APY → `RAY - 21_979_553_151_200_000_000_000_000`

For initial deployment, use tighter bounds (±1% APY) and widen as the system matures.

---

## 5. Gain Tuning Guidelines

The PI controller gains should be tuned based on:

1. **Market depth**: Thin markets require lower gains (less aggressive rate changes to avoid oscillation).
2. **Redemption price magnitude**: Error is relative (`e = (p_r - p_m) / p_r`), so gains are scale-independent.
3. **Update frequency**: With 1-hour updates, gains should be proportionally smaller than RAI's (which can update more frequently in theory).
4. **Stability criterion (Ziegler-Nichols heuristic)**: Find the critical gain Kc where the system oscillates, then set `Kp = 0.45 * Kc`, `Ti = period / 1.2`, `Ki = Kp / Ti`.

**Initial deployment:** Use proportional-only (`Ki = 0`) and observe system behavior before enabling the integral term.

---

## 6. Relationship to Stability Fee

The **redemption rate** and **stability fee** are independent but additive in effect:

- **Stability fee** (in `TaxCollector`): always positive, paid by SAFE owners regardless of market conditions. Compensates LPs and protocol treasury. Denominated in `accumulated_rate`.
- **Redemption rate** (in `OracleRelayer`): can be positive or negative, adjusts SAFE debt burden to push market price toward redemption price.

The effective per-second cost of holding debt is approximately:
```
effective_rate ≈ stability_fee + (redemption_rate - RAY)
```

When redemption_rate = RAY (neutral), only stability fee applies.

---

## 7. Error Codes

```rust
enum PIControllerError {
    AlreadyInitialized     = 3000,
    Unauthorized           = 3001,
    TooSoon                = 3002,
    OracleInvalid          = 3003,
    OracleStale            = 3004,
    InvalidParams          = 3005,
    ArithmeticOverflow     = 3006,
    InvalidPDA             = 3007,
}
```
