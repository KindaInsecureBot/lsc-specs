# LSC — Core Engine (LSCEngine Program)

## Overview

The LSCEngine is the heart of the system. It manages:
- SAFE lifecycle (open, close)
- Collateral deposits and withdrawals
- LSC debt generation and repayment
- Global debt tracking
- Authorized access control for SAFEs
- SAFE confiscation (called by LiquidationEngine)

**Program ID alias:** `LSC_ENGINE_ID`

---

## Instruction Enum

```rust
enum Instruction {
    /// Initialize the LSC system (one-time)
    Initialize {
        governance_id: [u8; 32],
        lsc_token_def_id: [u8; 32],
        logos_token_def_id: [u8; 32],
        oracle_relayer_params_id: [u8; 32],
        tax_collector_params_id: [u8; 32],
        liquidation_engine_params_id: [u8; 32],
        accounting_engine_params_id: [u8; 32],
        global_settlement_state_id: [u8; 32],
    },

    /// Add a new collateral type (governance only)
    AddCollateralType {
        token_def_id: [u8; 32],
        safety_ratio: u128,          // Ray
        liquidation_ratio: u128,     // Ray
        liquidation_penalty: u128,   // Wad
        debt_ceiling: u128,          // Rad
        debt_floor: u128,            // Rad
        stability_fee: u128,         // Ray per second
    },

    /// Update parameters for a collateral type (governance only)
    UpdateCollateralType {
        new_safety_ratio: Option<u128>,
        new_liquidation_ratio: Option<u128>,
        new_liquidation_penalty: Option<u128>,
        new_debt_ceiling: Option<u128>,
        new_debt_floor: Option<u128>,
        new_stability_fee: Option<u128>,
        new_active: Option<bool>,
    },

    /// Open a new SAFE for the caller
    OpenSafe {
        /// Arbitrary nonce chosen by caller (must make PDA unique)
        nonce: u64,
    },

    /// Deposit LOGOS collateral into a SAFE
    DepositCollateral {
        amount: u128,   // Wad
    },

    /// Withdraw LOGOS collateral from a SAFE
    WithdrawCollateral {
        amount: u128,   // Wad
    },

    /// Generate LSC debt (mint LSC)
    GenerateDebt {
        amount: u128,   // Wad (normalized debt to add)
    },

    /// Repay LSC debt (burn LSC)
    RepayDebt {
        amount: u128,   // Wad (normalized debt to remove)
    },

    /// Modify collateral AND debt in one atomic instruction
    /// (combines DepositCollateral + GenerateDebt or WithdrawCollateral + RepayDebt)
    ModifySafe {
        collateral_delta: i128,  // Wad, signed (+ = deposit, - = withdraw)
        debt_delta: i128,        // Wad normalized, signed (+ = generate, - = repay)
    },

    /// Transfer SAFE ownership to a new owner
    TransferSafeOwnership {
        new_owner_id: [u8; 32],
    },

    /// Grant an operator permission to modify this SAFE
    AllowOperator {
        operator_id: [u8; 32],
        allowed: bool,
    },

    /// Called by LiquidationEngine to seize a SAFE's assets
    /// (This instruction is privileged: only LiquidationEngine can call it)
    ConfiscateSafe {
        /// Collateral to seize (moved to liquidation vault)
        collateral_amount: u128,   // Wad
        /// Debt to zero out (removed from SAFE)
        debt_amount: u128,         // Wad normalized
    },

    /// Called by GlobalSettlement to freeze a collateral type
    FreezeCollateralType {
        final_collateral_price: u128,   // Ray
    },

    /// Close an empty SAFE (zero collateral, zero debt)
    CloseSafe,

    /// Update redemption rate across a collateral type's accumulated_rate
    /// (Called by TaxCollector — privileged)
    UpdateAccumulatedRate {
        new_accumulated_rate: u128,  // Ray — the NEW absolute accumulated_rate (not a delta)
    },
}
```

---

## Instructions: Detailed Specification

### `Initialize`

**Description:** One-time initialization. Creates the SystemParamsAccount and GlobalDebtAccount.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `system_params_id` (PDA) | Yes | PDA (new) | Created here |
| 1 | `global_debt_id` (PDA) | Yes | PDA (new) | Created here |
| 2 | `governance_account` | No | Yes (is_authorized) | Caller = initial governance |

**Validations:**
- `system_params_id` must be uninitialized (account_type == 0 / data empty)
- `global_debt_id` must be uninitialized
- PDA seeds must match expected derivation

**State Transition:**
```
system_params_id.data = SystemParamsAccount {
    account_type: 1,
    governance_id: governance_account.id,
    lsc_token_def_id,
    logos_token_def_id,
    oracle_relayer_params_id,
    tax_collector_params_id,
    liquidation_engine_params_id,
    accounting_engine_params_id,
    global_settlement_state_id,
    settlement_active: false,
    safe_nonce: 0,
    vault_nonce: 0,
    _reserved: [0; 64],
}

global_debt_id.data = GlobalDebtAccount {
    account_type: 4,
    total_debt: 0,
    total_surplus: 0,
    _reserved: [0; 32],
}
```

**Errors:**
- `AlreadyInitialized` — system_params already has data
- `InvalidPDA` — PDA seeds do not match

---

### `AddCollateralType`

**Description:** Adds a new collateral type and creates its vault. Governance-only.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `system_params_id` | Yes | — | For reading lsc_engine params |
| 1 | `governance_account` | No | Yes | Must match system_params.governance_id |
| 2 | `collateral_type_id` (PDA) | Yes | PDA (new) | Derived from token_def_id |
| 3 | `collateral_vault_id` (PDA) | Yes | PDA (new) | Derived from collateral_type_id |
| 4 | `logos_token_def_account` | No | — | Token definition for the collateral |

**Validations:**
- `governance_account.id == system_params.governance_id`
- `governance_account.is_authorized == true`
- `collateral_type_id` must be uninitialized
- `collateral_vault_id` must be uninitialized
- `safety_ratio >= liquidation_ratio`
- `liquidation_ratio > RAY` (must be > 100%)
- `liquidation_penalty < WAD` (< 100%)
- `debt_floor <= debt_ceiling`
- `stability_fee >= RAY` (rate >= 1.0, i.e. non-negative)

**State Transition:**
```
collateral_type_id.data = CollateralTypeAccount {
    account_type: 2,
    token_def_id: logos_token_def_account.id,
    vault_id: collateral_vault_id,
    accumulated_rate: RAY,
    last_stability_fee_update: current_timestamp,
    stability_fee,
    safety_ratio,
    liquidation_ratio,
    liquidation_penalty,
    debt_ceiling,
    debt_floor,
    global_normalized_debt: 0,
    active: true,
    settlement_processed: false,
    final_collateral_price: 0,
    _reserved: [0; 32],
}

// collateral_vault_id is initialized as a Token Program holding account
// with holder = collateral_vault_id (PDA), definition_id = logos_token_def_id
```

**Chained Calls:**
```
// InitializeAccount takes no instruction parameters — account positions determine behavior.
// Account order: [definition_account (read), account_to_initialize (writable, new)]
// The initialized account's holder is set to its own ID (the PDA).
TokenProgram::InitializeAccount
    Account list:
      logos_token_def_account (read)           // #0: token definition
      collateral_vault_id (writable, new)      // #1: account to initialize
```

**Errors:**
- `Unauthorized` — caller is not governance
- `CollateralTypeExists` — collateral_type_id already initialized
- `InvalidRatios` — safety_ratio < liquidation_ratio
- `InvalidPDA` — PDA derivation mismatch

---

### `OpenSafe`

**Description:** Opens a new SAFE for the caller. Creates a SafeAccount PDA.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `system_params_id` | No | — | For reading settlement_active |
| 1 | `collateral_type_id` | No | — | The collateral type for this SAFE |
| 2 | `safe_id` (PDA) | Yes | PDA (new) | Derived from caller + nonce |
| 3 | `caller_account` | No | Yes | SAFE owner |

**PDA Derivation:**
```
safe_id = LSC_ENGINE_ID.derive(b"safe" || caller_account.id || nonce.to_le_bytes())
```

**Validations:**
- `system_params.settlement_active == false`
- `collateral_type.active == true`
- `safe_id` must be uninitialized
- `caller_account.is_authorized == true`
- PDA seeds must match

**State Transition:**
```
safe_id.data = SafeAccount {
    account_type: 3,
    owner_id: caller_account.id,
    collateral_type_id: collateral_type_id.id,
    collateral: 0,
    normalized_debt: 0,
    opened_at: current_timestamp,
    last_modified_at: current_timestamp,
    liquidated: false,
    allowed_operators: [[0u8; 32]; 4],
    allowed_operators_count: 0,
    _reserved: [0; 32],
}
```

**Errors:**
- `SettlementActive` — global settlement is ongoing
- `CollateralTypeInactive` — collateral type is disabled
- `SafeAlreadyExists` — safe_id already initialized
- `Unauthorized` — caller is not authorized

---

### `DepositCollateral`

**Description:** Transfers LOGOS from caller into the collateral vault, increasing `safe.collateral`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | The SAFE to deposit into |
| 1 | `collateral_type_id` | No | — | Collateral type |
| 2 | `collateral_vault_id` | Yes | — | Vault that receives collateral |
| 3 | `caller_logos_holding_id` | Yes | Yes | Caller's LOGOS token holding |
| 4 | `caller_account` | No | Yes | Must be SAFE owner or operator |

**Validations:**
- `safe.owner_id == caller_account.id` OR `caller_account.id in safe.allowed_operators`
- `caller_account.is_authorized == true`
- `safe.liquidated == false`
- `collateral_type.active == true`
- `amount > 0`
- `caller_logos_holding.balance >= amount`

**State Transition:**
```
safe.collateral += amount
safe.last_modified_at = current_timestamp
```

**Chained Calls:**
```
TokenProgram::Transfer {
    amount_to_transfer: amount,
    // accounts: [caller_logos_holding_id (writable, is_authorized), collateral_vault_id (writable)]
    // pda_seeds: []  (caller authorizes their own holding; no LSCEngine PDA needed here)
}
```

**Errors:**
- `Unauthorized` — caller is not owner or operator
- `SafeLiquidated` — SAFE has been liquidated
- `InsufficientBalance` — not enough LOGOS
- `ZeroAmount` — amount is 0

---

### `WithdrawCollateral`

**Description:** Withdraws LOGOS from the vault to the caller, decreasing `safe.collateral`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | The SAFE to withdraw from |
| 1 | `collateral_type_id` | No | — | Collateral type |
| 2 | `collateral_vault_id` | Yes | PDA (LSCEngine) | Vault to withdraw from |
| 3 | `oracle_relayer_params_id` | No | — | For reading redemption_price |
| 4 | `collateral_oracle_id` | No | — | For reading collateral price |
| 5 | `recipient_logos_holding_id` | Yes | — | Where LOGOS is sent |
| 6 | `caller_account` | No | Yes | SAFE owner or operator |

**Validations:**
- Ownership check (same as DepositCollateral)
- `safe.liquidated == false`
- `amount <= safe.collateral`
- After withdrawal, check collateralization:
  ```
  new_collateral = safe.collateral - amount
  effective_debt = safe.normalized_debt * accumulated_rate  // Rad
  collateral_value = new_collateral * collateral_price * redemption_price_ray  // in Rad units
  assert collateral_value >= effective_debt * safety_ratio
  ```
  (If `safe.normalized_debt == 0`, no collateralization check needed.)

**Collateralization Check Formula:**
```
Let:
  p_c = collateral_price (Wad, USD per LOGOS)
  p_r = redemption_price (Ray, USD per LSC)
  c   = new_collateral (Wad)
  d   = effective_debt = normalized_debt * accumulated_rate (Rad)
  s   = safety_ratio (Ray)

Collateral value in LSC units (Rad):
  cv = (c * p_c * RAY) / p_r   [all in Rad units]

Required:
  cv * RAY >= d * s
```

**State Transition:**
```
safe.collateral -= amount
safe.last_modified_at = current_timestamp
```

**Chained Calls:**
```
TokenProgram::Transfer {
    amount_to_transfer: amount,
    // accounts: [collateral_vault_id (writable, is_authorized), recipient_logos_holding_id (writable)]
    // pda_seeds: [compute_pda_seed(b"vault" || collateral_type_id)]
}
```

**Errors:**
- `Unauthorized`
- `InsufficientCollateral` — amount > safe.collateral
- `UnsafeCR` — collateralization ratio would drop below safety_ratio
- `OracleStale` — oracle price is too old

---

### `GenerateDebt`

**Description:** Mints LSC to the caller by increasing the SAFE's debt.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | The SAFE generating debt |
| 1 | `collateral_type_id` | Yes | — | Collateral type (global_normalized_debt updated) |
| 2 | `global_debt_id` | Yes | — | Global debt tracking |
| 3 | `oracle_relayer_params_id` | No | — | For redemption_price |
| 4 | `collateral_oracle_id` | No | — | For collateral_price |
| 5 | `lsc_token_def_id` | No | — | LSC token definition |
| 6 | `recipient_lsc_holding_id` | Yes | — | Where minted LSC goes |
| 7 | `system_params_id` | No | — | For settlement check |
| 8 | `caller_account` | No | Yes | SAFE owner or operator |

**Validations:**
- Ownership check
- `system_params.settlement_active == false`
- `collateral_type.active == true`
- `safe.liquidated == false`
- `amount > 0`
- Collateralization check after adding debt:
  ```
  new_normalized_debt = safe.normalized_debt + amount
  new_effective_debt = new_normalized_debt * accumulated_rate  // Rad
  collateral_value_in_rad = (safe.collateral * collateral_price * RAY) / redemption_price
  assert collateral_value_in_rad * RAY >= new_effective_debt * safety_ratio
  ```
- Debt ceiling check:
  ```
  new_global_debt_rad = (collateral_type.global_normalized_debt + amount) * accumulated_rate
  assert new_global_debt_rad <= collateral_type.debt_ceiling
  ```
- Debt floor check (only if debt > 0 after):
  ```
  new_effective_debt_rad = new_normalized_debt * accumulated_rate
  assert new_effective_debt_rad >= collateral_type.debt_floor
  ```

**State Transition:**
```
safe.normalized_debt += amount
safe.last_modified_at = current_timestamp
collateral_type.global_normalized_debt += amount
global_debt.total_debt += amount * accumulated_rate  // Rad
```

**Chained Calls:**
```
// Mint LSC to recipient
// amount_in_lsc = amount * accumulated_rate / RAY  (convert normalized to actual LSC)
// Note: Token Program Mint uses Wad amounts
let lsc_to_mint = (amount * accumulated_rate) / RAY;  // Wad

TokenProgram::Mint {
    amount_to_mint: lsc_to_mint,
    // accounts: [lsc_token_def_id (writable, is_authorized), recipient_lsc_holding_id (writable)]
    // pda_seeds: [compute_pda_seed(b"lsc_token_definition")]
    // lsc_token_def_id is a PDA of LSCEngine derived from this seed.
    // The runtime marks it is_authorized because LSCEngine provides its seed.
}
```

**Errors:**
- `Unauthorized`
- `SettlementActive`
- `DebtCeilingExceeded`
- `BelowDebtFloor`
- `UnsafeCR`
- `ZeroAmount`
- `OracleStale`

---

### `RepayDebt`

**Description:** Burns LSC from caller to reduce SAFE debt.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | The SAFE repaying debt |
| 1 | `collateral_type_id` | Yes | — | Collateral type |
| 2 | `global_debt_id` | Yes | — | Global debt |
| 3 | `lsc_token_def_id` | No | — | LSC token definition |
| 4 | `caller_lsc_holding_id` | Yes | Yes | Caller's LSC (burned from here) |
| 5 | `caller_account` | No | Yes | SAFE owner or operator |

**Validations:**
- Ownership check
- `safe.liquidated == false`
- `amount <= safe.normalized_debt`
- After repayment, if `safe.normalized_debt - amount > 0`:
  ```
  new_effective_debt_rad = (safe.normalized_debt - amount) * accumulated_rate
  assert new_effective_debt_rad >= collateral_type.debt_floor
  ```
  (Partial repayment cannot leave SAFE below dust floor; must repay fully.)
- `lsc_to_burn = amount * accumulated_rate / RAY`
- Caller has enough LSC: checked implicitly by Token Program

**State Transition:**
```
safe.normalized_debt -= amount
safe.last_modified_at = current_timestamp
collateral_type.global_normalized_debt -= amount
global_debt.total_debt -= amount * accumulated_rate  // Rad (normalized_debt × rate = Rad)
```

**Chained Calls:**
```
let lsc_to_burn = (amount * accumulated_rate) / RAY;  // Wad

TokenProgram::Burn {
    amount_to_burn: lsc_to_burn,
    // accounts: [lsc_token_def_id, caller_lsc_holding_id (auth)]
    // Note: definition account first, then holder (authorized)
}
```

**Errors:**
- `Unauthorized`
- `DebtOverpayment` — amount > normalized_debt
- `BelowDebtFloor` — partial repayment leaves SAFE under dust floor
- `InsufficientLSC`

---

### `ModifySafe`

**Description:** Atomic combination of collateral change + debt change. Preferred for efficiency.

**Required Accounts:**
Union of accounts required by DepositCollateral/WithdrawCollateral + GenerateDebt/RepayDebt, depending on signs of deltas. See individual instructions.

Additional required accounts:
| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| All accounts from DepositCollateral or WithdrawCollateral | ... | ... | ... |
| All accounts from GenerateDebt or RepayDebt | ... | ... | ... |

**Validations:** Combination of validations from constituent instructions, applied atomically.

**State Transitions:**
```
if collateral_delta > 0: safe.collateral += collateral_delta
if collateral_delta < 0: safe.collateral -= abs(collateral_delta)
if debt_delta > 0: safe.normalized_debt += debt_delta
if debt_delta < 0: safe.normalized_debt -= abs(debt_delta)

// Collateralization check applied after both changes
// Debt ceiling / floor checks applied after both changes
```

**Chained Calls:** Token Program transfers/mints/burns as needed.

---

### `TransferSafeOwnership`

**Description:** Transfers a SAFE to a new owner.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | SAFE to transfer |
| 1 | `current_owner` | No | Yes | Current SAFE owner |

**Validations:**
- `current_owner.id == safe.owner_id`
- `current_owner.is_authorized == true`
- `new_owner_id != current_owner.id`
- `safe.liquidated == false`

**State Transition:**
```
safe.owner_id = new_owner_id
safe.allowed_operators = [[0; 32]; 4]  // clear operators on transfer
safe.allowed_operators_count = 0
safe.last_modified_at = current_timestamp
```

**Errors:**
- `Unauthorized`
- `SafeLiquidated`
- `SameOwner`

---

### `AllowOperator`

**Description:** Grants or revokes an operator's permission to modify a SAFE.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | The SAFE |
| 1 | `owner_account` | No | Yes | SAFE owner |

**Validations:**
- `owner_account.id == safe.owner_id`
- `owner_account.is_authorized == true`
- If `allowed == true`: `safe.allowed_operators_count < 4`
- If `allowed == false`: operator must be in allowed_operators list

**State Transition:**
```
if allowed:
    safe.allowed_operators[safe.allowed_operators_count] = operator_id
    safe.allowed_operators_count += 1
else:
    remove operator_id from safe.allowed_operators (shift remaining)
    safe.allowed_operators_count -= 1
```

**Errors:**
- `Unauthorized`
- `OperatorListFull` — already 4 operators
- `OperatorNotFound` — trying to remove non-existent operator

---

### `ConfiscateSafe`

**Description:** Called by LiquidationEngine to seize collateral and clear debt from a SAFE. This is a **privileged instruction** — only callable via chained call from LiquidationEngine.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | SAFE to confiscate |
| 1 | `collateral_type_id` | Yes | — | Collateral type |
| 2 | `global_debt_id` | Yes | — | Global debt |
| 3 | `system_params_id` | No | — | For liquidation_engine_id check |
| 4 | `liquidation_engine_params_id` | No | Yes (PDA) | Must be the registered liquidation engine |
| 5 | `collateral_vault_id` | Yes | — | SAFE's collateral vault |
| 6 | `liquidation_auction_vault_id` | Yes | — | Auction house vault (receives collateral) |

**Authorization:**
The `liquidation_engine_params_id` must be `is_authorized`. This happens because LiquidationEngine derives the correct PDA and includes it as authorized in the chained call. The LSCEngine verifies: `system_params.liquidation_engine_params_id == liquidation_engine_params_id.id`.

**Validations:**
- `system_params.liquidation_engine_params_id == liquidation_engine_params_id.id`
- `liquidation_engine_params_id.is_authorized == true`
- `collateral_amount <= safe.collateral`
- `debt_amount <= safe.normalized_debt`
- `safe.liquidated == false`

**State Transition:**
```
safe.collateral -= collateral_amount
safe.normalized_debt -= debt_amount
safe.liquidated = true  // (if collateral_amount == safe.collateral && debt_amount == safe.normalized_debt)
collateral_type.global_normalized_debt -= debt_amount
global_debt.total_debt -= debt_amount * accumulated_rate  // Rad
// Any resulting bad debt is pushed to AccountingEngine::PushDebt by the LiquidationEngine caller
```

**Chained Calls:**
```
// Transfer collateral from vault to auction vault
TokenProgram::Transfer {
    amount_to_transfer: collateral_amount,
    // accounts: [collateral_vault_id (writable, is_authorized), liquidation_auction_vault_id (writable)]
    // pda_seeds: [compute_pda_seed(b"vault" || collateral_type_id)]
}
```

**Errors:**
- `Unauthorized` — not called from registered LiquidationEngine
- `InsufficientCollateral`
- `InsufficientDebt`
- `SafeAlreadyLiquidated`

---

### `UpdateAccumulatedRate`

**Description:** Called by TaxCollector to update the `accumulated_rate` for a collateral type, effectively accruing stability fees. Privileged — only callable from TaxCollector.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `collateral_type_id` | Yes | — | Collateral type to update |
| 1 | `global_debt_id` | Yes | — | Global debt (updated for rate increase) |
| 2 | `system_params_id` | No | — | For tax_collector check |
| 3 | `tax_collector_params_id` | No | Yes (PDA) | Must be registered tax collector |
| 4 | `accounting_engine_surplus_id` | Yes | — | Where surplus LSC is sent |
| 5 | `lsc_token_def_id` | No | — | For minting surplus LSC |

**Validations:**
- `system_params.tax_collector_params_id == tax_collector_params_id.id`
- `tax_collector_params_id.is_authorized == true`
- `new_accumulated_rate >= collateral_type.accumulated_rate` (rate can only increase)
- `collateral_type.active == true` (or allow even on inactive for fee collection)

**State Transition:**
```
// new_accumulated_rate is the new ABSOLUTE accumulated_rate value,
// pre-computed by TaxCollector as: old_rate * rpow(stability_fee, dt, RAY) / RAY
old_rate = collateral_type.accumulated_rate
collateral_type.accumulated_rate = new_accumulated_rate
collateral_type.last_stability_fee_update = current_timestamp

// Surplus LSC minted = rate increase × total normalized debt (in Rad)
surplus_rad = (new_accumulated_rate - old_rate) * collateral_type.global_normalized_debt
surplus_wad = surplus_rad / RAY

// Also update global debt to reflect higher effective debt
global_debt.total_debt += (new_accumulated_rate - old_rate) * collateral_type.global_normalized_debt
```

**Chained Calls:**
```
// Mint surplus LSC to accounting engine
TokenProgram::Mint {
    amount_to_mint: surplus_wad,
    // accounts: [lsc_token_def_id (writable, is_authorized), accounting_engine_surplus_id (writable)]
    // pda_seeds: [compute_pda_seed(b"lsc_token_definition")]
}
```

**Errors:**
- `Unauthorized`
- `RateDecrease` — new_accumulated_rate < collateral_type.accumulated_rate

---

### `CloseSafe`

**Description:** Closes an empty SAFE (zeroes its data). Refunds account balance to owner.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `safe_id` | Yes | — | SAFE to close |
| 1 | `owner_account` | No | Yes | SAFE owner |

**Validations:**
- `owner_account.id == safe.owner_id`
- `owner_account.is_authorized == true`
- `safe.collateral == 0`
- `safe.normalized_debt == 0`

**State Transition:**
```
safe_id.data = [0u8; ...]  // zeroed
// Account balance returned to owner (LEZ account reclaim mechanism)
```

**Errors:**
- `Unauthorized`
- `SafeNotEmpty` — collateral or debt > 0

---

## Collateralization Ratio Check (Reference Implementation)

```rust
fn is_safe_collateralized(
    collateral: u128,         // Wad
    normalized_debt: u128,    // Wad
    accumulated_rate: u128,   // Ray
    collateral_price: u128,   // Wad (USD per LOGOS)
    redemption_price: u128,   // Ray (USD per LSC)
    safety_ratio: u128,       // Ray
) -> bool {
    if normalized_debt == 0 {
        return true;
    }
    // effective_debt in Rad = normalized_debt * accumulated_rate
    let effective_debt_rad = (normalized_debt as u256) * (accumulated_rate as u256); // Rad (45 dec)

    // collateral_value in LSC (Rad) = collateral * collateral_price / redemption_price
    // collateral: Wad (18 dec)
    // collateral_price: Wad (18 dec) → collateral * price = Rad (36 dec)
    // but we need Rad (45 dec) → multiply by RAY (10^27) and divide by redemption_price (Ray = 10^27)
    let collateral_value_rad =
        (collateral as u256) * (collateral_price as u256) * (RAY as u256) / (redemption_price as u256);
    // Result: Wad * Wad * Ray / Ray = Wad^2 ... need to reconcile units
    // Correct approach: express everything in Rad (10^45)
    // collateral (Wad) * price (Wad) = value in USD^2...
    //
    // Simpler: collateral_value_in_usd (Wad) = collateral * collateral_price / WAD
    // collateral_value_in_lsc (Wad) = collateral_value_in_usd * RAY / redemption_price
    // collateral_value_rad (Rad) = collateral_value_in_lsc * RAY (to get Rad)

    let collateral_value_usd_wad = (collateral as u256) * (collateral_price as u256) / WAD;
    let collateral_value_lsc_wad = collateral_value_usd_wad * RAY / (redemption_price as u256);
    let collateral_value_rad = collateral_value_lsc_wad * RAY;

    // Check: collateral_value_rad >= effective_debt_rad * safety_ratio / RAY
    collateral_value_rad * RAY >= effective_debt_rad * (safety_ratio as u256)
}
```

**Note:** All intermediate computations must use 256-bit arithmetic to avoid overflow. The implementation must use a `u256` type or equivalent big-integer arithmetic.

---

## Error Codes

```rust
enum LSCEngineError {
    AlreadyInitialized        = 1000,
    Unauthorized              = 1001,
    InvalidPDA                = 1002,
    SettlementActive          = 1003,
    CollateralTypeExists      = 1004,
    CollateralTypeInactive    = 1005,
    CollateralTypeNotFound    = 1006,
    SafeAlreadyExists         = 1007,
    SafeNotFound              = 1008,
    SafeLiquidated            = 1009,
    SafeNotEmpty              = 1010,
    InsufficientCollateral    = 1011,
    InsufficientBalance       = 1012,
    UnsafeCR                  = 1013,
    DebtCeilingExceeded       = 1014,
    BelowDebtFloor            = 1015,
    DebtOverpayment           = 1016,
    ZeroAmount                = 1017,
    OracleStale               = 1018,
    InvalidRatios             = 1019,
    OperatorListFull          = 1020,
    OperatorNotFound          = 1021,
    RateDecrease              = 1022,
    SameOwner                 = 1023,
    InvalidAccountType        = 1024,
    ArithmeticOverflow        = 1025,
    InsufficientLSC           = 1026,
}
```
