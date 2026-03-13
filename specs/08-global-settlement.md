# LSC — Global Settlement (Emergency Shutdown)

## Overview

Global Settlement (GS) is the last-resort emergency shutdown mechanism. When triggered:
1. The system **freezes** — no new debt can be generated, no new liquidations.
2. All outstanding collateral auctions are terminated.
3. Each collateral type is **processed** — a final collateral price is fixed.
4. SAFE owners can **redeem** their net collateral.
5. LSC holders can **redeem** LSC directly for collateral at the final price (after SAFE owners claim theirs).

Settlement is **irreversible**. It should only be triggered by governance in extreme circumstances: critical bug, oracle failure, governance attack, or system insolvency.

**Program ID alias:** `GLOBAL_SETTLEMENT_ID`

---

## 1. Settlement Phases

```
Phase 0: ACTIVE SYSTEM
         │
         │ Governance triggers TriggerSettlement
         ▼
Phase 1: SHUTDOWN INITIATED
         ├─ All new GenerateDebt calls blocked
         ├─ All new CollateralAuctions blocked
         ├─ Oracle Relayer rate updates blocked
         │
         │ Anyone calls FreezeCollateralType for each collateral type
         ▼
Phase 2: COLLATERAL TYPES FROZEN
         ├─ Final collateral price recorded for each type
         ├─ Collateral auctions can still complete (or be terminated)
         │
         │ Anyone calls ProcessSAFEs (removes surplus/deficit SAFEs)
         ▼
Phase 3: SAFERS CAN REDEEM
         │ SAFE owners call RedeemCollateral to reclaim excess collateral
         │
         │ (After SAFE redemption period, e.g., 3 days)
         ▼
Phase 4: LSC HOLDERS CAN REDEEM
         │ LSC holders call RedeemLSC to exchange LSC for collateral
         ▼
Phase 5: COMPLETE
```

---

## 2. Instruction Enum

```rust
enum GlobalSettlementInstruction {
    /// One-time initialization
    Initialize,

    /// Trigger global settlement (governance only)
    TriggerSettlement,

    /// Freeze a collateral type at the current oracle price
    FreezeCollateralType {
        collateral_type_id: [u8; 32],
    },

    /// Process SAFEs to compute net collateral available for LSC holders
    /// (Called once per collateral type after freezing)
    ProcessSAFEs {
        collateral_type_id: [u8; 32],
    },

    /// SAFE owner redeems their collateral (excess above their debt)
    RedeemCollateral {
        safe_id: [u8; 32],
    },

    /// Set the final collateral-per-LSC redemption rate
    /// (Called after all SAFEs processed, before LSC redemption opens)
    SetFinalRedemptionRate {
        collateral_type_id: [u8; 32],
    },

    /// LSC holder redeems LSC for collateral at final rate
    RedeemLSC {
        lsc_amount: u128,    // Wad
        collateral_type_id: [u8; 32],
    },
}
```

---

## 3. Instructions: Detailed Specification

### 3.1 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` (PDA) | Yes | PDA (new) | — |
| 1 | `governance_account` | No | Yes | Initial governance |

**State Transition:**
```
settlement_state.account_type = 100
settlement_state.active = false
settlement_state.triggered_at = 0
settlement_state.triggered_by = [0; 32]
settlement_state.final_redemption_price = 0
settlement_state.collateral_types = [[0; 32]; 16]
settlement_state.collateral_type_count = 0
settlement_state.processed_count = 0
settlement_state.total_outstanding_lsc = 0
```

---

### 3.2 `TriggerSettlement`

**Description:** Initiates global settlement. Governance-only, irreversible.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | Yes | — | — |
| 1 | `system_params_id` | Yes | — | Set settlement_active = true |
| 2 | `governance_account` | No | Yes | Must match system governance |
| 3 | `oracle_relayer_params_id` | No | — | Snapshot final redemption price |
| 4 | `lsc_market_oracle_id` | No | — | For timestamp |

**Validations:**
- `governance_account.id == system_params.governance_id`
- `governance_account.is_authorized == true`
- `settlement_state.active == false`

**State Transition:**
```
settlement_state.active = true
settlement_state.triggered_at = current_timestamp
settlement_state.triggered_by = governance_account.id
settlement_state.final_redemption_price = oracle_relayer_params.redemption_price

// Propagate to LSCEngine
system_params.settlement_active = true
```

**Side Effects:**
- After this, `LSCEngine::GenerateDebt` will fail (`SettlementActive`).
- `PIController::UpdateRate` should also check and fail.
- `CollateralAuctionHouse::StartAuction` should fail.
- Existing auctions can still complete (or be terminated by governance).

---

### 3.3 `FreezeCollateralType`

**Description:** Locks in the final oracle price for a collateral type. Anyone can call this after settlement is triggered.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | Yes | — | Register collateral type |
| 1 | `collateral_type_id` | Yes | — | Freeze it |
| 2 | `collateral_oracle_id` | No | — | Get final collateral price |
| 3 | `lsc_market_oracle_id` | No | — | For timestamp |

**Validations:**
```
assert!(settlement_state.active, SettlementNotActive);
assert!(!collateral_type.settlement_processed, AlreadyFrozen);
assert!(collateral_oracle.valid, OracleInvalid);
assert!(settlement_state.collateral_type_count < 16, TooManyCollateralTypes);
```

**State Transition:**
```
let final_price = collateral_oracle.median_price;  // Wad (USD per LOGOS)

// Convert to Ray (USD per LOGOS in Ray units) for storage
collateral_type.final_collateral_price = final_price * RAY / WAD;  // Ray
collateral_type.active = false  // no new SAFEs can be opened

// Register in settlement state
settlement_state.collateral_types[settlement_state.collateral_type_count] = collateral_type_id
settlement_state.collateral_type_count += 1
```

---

### 3.4 `ProcessSAFEs`

**Description:** Computes the amount of collateral needed to cover all outstanding debt at the final price, and determines the surplus collateral available for LSC redemption.

This instruction must be called once per collateral type after freezing. Due to the individual SAFE account model, "processing" is done globally by reading `global_normalized_debt` and `accumulated_rate`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | Yes | — | Track processing |
| 1 | `collateral_type_id` | Yes | — | Read global debt |
| 2 | `collateral_redemption_id` (PDA) | Yes | PDA (new) | Create redemption record |
| 3 | `collateral_vault_id` | No | — | Read available collateral |
| 4 | `lsc_market_oracle_id` | No | — | For timestamp |

**Computation:**
```
// Total debt outstanding for this collateral type (in LSC Wad)
total_normalized_debt = collateral_type.global_normalized_debt  // Wad
total_effective_debt_rad = total_normalized_debt * collateral_type.accumulated_rate  // Rad
total_lsc_debt_wad = total_effective_debt_rad / RAY  // Wad

// Collateral needed to cover all debt at final price
// final_price is Ray (USD per LOGOS), redemption_price is Ray (USD per LSC)
// collateral_per_lsc = redemption_price / final_price  (LOGOS per LSC)
let p_col = collateral_type.final_collateral_price  // Ray (USD/LOGOS)
let p_red = settlement_state.final_redemption_price  // Ray (USD/LSC)

// Collateral needed (Wad of LOGOS) = LSC_debt (Wad) * (p_red/p_col)
// = LSC_debt * p_red / p_col
let collateral_needed_wad = total_lsc_debt_wad * p_red / p_col;  // Wad

// Available collateral in vault
let collateral_available = collateral_vault.balance  // Wad (LOGOS in vault)

// If available < needed: system is underwater (bad debt remains)
// If available >= needed: surplus collateral goes to LSC redemption
let surplus_collateral = if collateral_available >= collateral_needed_wad {
    collateral_available - collateral_needed_wad
} else {
    0  // underwater — LSC redemptions will receive less than par
};

// Collateral per LSC for final redemption
// = surplus_collateral / total LSC supply
// (total LSC supply must be tracked separately or computed from total global debt)
// We store collateral_available and total_lsc_debt; rate is set in SetFinalRedemptionRate
```

**State Transition:**
```
collateral_redemption.account_type = 101
collateral_redemption.collateral_type_id = collateral_type_id
collateral_redemption.final_collateral_price = collateral_type.final_collateral_price
collateral_redemption.collateral_available = collateral_available  // updated as SAFEs redeem
collateral_redemption.total_lsc_redeemed = 0
collateral_redemption.collateral_per_lsc = 0  // set later by SetFinalRedemptionRate

settlement_state.processed_count += 1
settlement_state.total_outstanding_lsc += total_lsc_debt_wad
```

---

### 3.5 `RedeemCollateral` (SAFE owners)

**Description:** A SAFE owner redeems their SAFE's net collateral. They get back: `collateral - debt_in_collateral_units`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | No | — | Verify settlement active |
| 1 | `safe_id` | Yes | — | The SAFE to close |
| 2 | `collateral_type_id` | Yes | — | For accumulated_rate, final_price |
| 3 | `collateral_vault_id` | Yes | PDA (LSCEngine) | Source of collateral |
| 4 | `owner_logos_holding_id` | Yes | — | Receives net collateral |
| 5 | `owner_account` | No | Yes | SAFE owner |
| 6 | `collateral_redemption_id` | Yes | — | Reduce available collateral |

**Computation:**
```
// Effective debt in LSC
effective_debt_wad = safe.normalized_debt * collateral_type.accumulated_rate / RAY  // Wad

// Debt in collateral units at final price
// collateral_to_cover = effective_debt * (redemption_price / final_collateral_price)
let p_col = collateral_type.final_collateral_price  // Ray
let p_red = settlement_state.final_redemption_price  // Ray
let debt_in_collateral = effective_debt_wad * p_red / p_col  // Wad (LOGOS)

// Net collateral owner can reclaim
let net_collateral = if safe.collateral >= debt_in_collateral {
    safe.collateral - debt_in_collateral
} else {
    0  // insolvent SAFE; owner gets nothing
};
```

**Validations:**
```
assert!(settlement_state.active, SettlementNotActive);
assert!(collateral_type.settlement_processed == false || collateral_type.active == false);
// (frozen collateral types have active = false)
assert!(!safe.liquidated, AlreadyLiquidated);
assert!(owner_account.id == safe.owner_id, Unauthorized);
assert!(owner_account.is_authorized, Unauthorized);
```

**State Transition:**
```
// Reduce collateral available for LSC redemption
collateral_redemption.collateral_available -= debt_in_collateral
// (debt_in_collateral's worth is "consumed" to cover this SAFE's debt)
// (net_collateral goes back to owner, leaving less for LSC redemption)
// Actually:
// collateral_available_for_lsc = remaining vault - claimed by SAFE owners
// Each SAFE redemption claims safe.collateral (which includes the debt portion and net portion)
// The debt portion covers their LSC debt; the net goes to them
// So: collateral_redemption.collateral_available -= safe.collateral

collateral_redemption.collateral_available -= safe.collateral
safe.collateral = 0
safe.normalized_debt = 0
safe.liquidated = true  // closed
```

**Chained Calls:**
```
if net_collateral > 0 {
    TokenProgram::Transfer {
        amount_to_transfer: net_collateral,
        // accounts: [collateral_vault_id (PDA auth), owner_logos_holding_id]
    }
}
```

---

### 3.6 `SetFinalRedemptionRate`

**Description:** After all SAFEs have redeemed (or after a waiting period), compute the final collateral-per-LSC rate for LSC redemption. Anyone can call this.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | No | — | For timing check |
| 1 | `collateral_redemption_id` | Yes | — | Set collateral_per_lsc |
| 2 | `collateral_vault_id` | No | — | Read remaining collateral |
| 3 | `lsc_market_oracle_id` | No | — | Timestamp |

**Validations:**
```
assert!(settlement_state.active, SettlementNotActive);
assert!(collateral_redemption.collateral_per_lsc == 0, AlreadySet);
// Waiting period check:
assert!(
    current_timestamp >= settlement_state.triggered_at + SAFE_REDEMPTION_PERIOD,
    WaitingPeriodNotElapsed
);
// SAFE_REDEMPTION_PERIOD = 3 days = 259200 seconds (recommended)
```

**Computation:**
```
// Remaining collateral in vault (after SAFE owners redeemed)
remaining_collateral = collateral_vault.balance  // Wad

// Total LSC supply (outstanding LSC that can be redeemed)
total_lsc_outstanding = settlement_state.total_outstanding_lsc  // Wad

// Collateral per LSC
// = remaining_collateral / total_lsc_outstanding
// Unit: Wad (LOGOS per LSC)
let collateral_per_lsc = remaining_collateral * WAD / total_lsc_outstanding;
```

**State Transition:**
```
collateral_redemption.collateral_per_lsc = collateral_per_lsc
```

---

### 3.7 `RedeemLSC`

**Description:** LSC holders burn their LSC and receive collateral at the final rate.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `settlement_state_id` | No | — | Verify active |
| 1 | `collateral_redemption_id` | Yes | — | Track redeemed, reduce available |
| 2 | `holder_lsc_holding_id` | Yes | Yes | Burn LSC from here |
| 3 | `holder_logos_holding_id` | Yes | — | Receive LOGOS here |
| 4 | `collateral_vault_id` | Yes | PDA (GlobalSettlement) | Source of LOGOS |
| 5 | `lsc_token_def_id` | No | — | For burn |
| 6 | `holder_account` | No | Yes | LSC holder |

**Validations:**
```
assert!(settlement_state.active, SettlementNotActive);
assert!(collateral_redemption.collateral_per_lsc > 0, RedemptionRateNotSet);
assert!(lsc_amount > 0, ZeroAmount);
assert!(lsc_amount <= holder_lsc_holding.balance, InsufficientLSC);
```

**Computation:**
```
// LOGOS to receive
logos_to_receive = lsc_amount * collateral_redemption.collateral_per_lsc / WAD  // Wad

// Cap by remaining available
logos_to_receive = min(logos_to_receive, collateral_redemption.collateral_available)
```

**State Transition:**
```
collateral_redemption.collateral_available -= logos_to_receive
collateral_redemption.total_lsc_redeemed += lsc_amount * RAY  // track in Rad
```

**Chained Calls:**
```
// 1. Burn LSC
TokenProgram::Burn {
    amount_to_burn: lsc_amount,
    // accounts: [lsc_token_def_id, holder_lsc_holding_id (auth)]
    // Note: definition account first, then holder (authorized)
}

// 2. Send LOGOS to holder
TokenProgram::Transfer {
    amount_to_transfer: logos_to_receive,
    // accounts: [collateral_vault_id (PDA auth), holder_logos_holding_id]
}
```

---

## 4. Settlement Timing

| Phase | Action | Who | When |
|---|---|---|---|
| Trigger | `TriggerSettlement` | Governance | Emergency |
| Freeze | `FreezeCollateralType` | Anyone | Immediately after trigger |
| Process | `ProcessSAFEs` | Anyone | After all types frozen |
| SAFE Redemption | `RedeemCollateral` | SAFE owners | Up to SAFE_REDEMPTION_PERIOD (3 days) |
| Set Rate | `SetFinalRedemptionRate` | Anyone | After SAFE_REDEMPTION_PERIOD |
| LSC Redemption | `RedeemLSC` | LSC holders | After rate is set, indefinitely |

---

## 5. Edge Cases

### 5.1 Undercollateralized System

If the total collateral value < total LSC outstanding (system-wide insolvency), LSC holders receive less than the redemption price per token. The `collateral_per_lsc` rate reflects this. There is no recourse; this is the last-resort scenario.

### 5.2 Active Collateral Auctions at Settlement

When settlement is triggered:
- `CollateralAuctionHouse::StartAuction` is blocked.
- Existing auctions can be **terminated** by governance via `TerminateAuction`, which returns the collateral to the settlement vault.
- Or existing auctions can run to completion; the LSC raised goes to the AccountingEngine surplus.

Governance must decide which to do based on whether the existing auction prices are fair.

### 5.3 Bad Debt at Settlement

There are no debt auctions to terminate. Any queued bad debt in the `SystemDebtQueueAccount` at settlement time is absorbed by the settlement process — it reduces the collateral available to LSC holders proportionally. LSC holders may receive less than redemption price per token if bad debt exceeds surplus at the time of settlement (see §5.1).

### 5.4 Multiple Collateral Types

In v1, only LOGOS is a collateral type. The settlement process is applied once. Future versions with multiple collateral types must process each type independently. LSC holders can choose which collateral type to redeem against (in proportion to each type's availability).

---

## 6. Error Codes

```rust
enum GlobalSettlementError {
    AlreadyInitialized          = 6000,
    Unauthorized                = 6001,
    SettlementNotActive         = 6002,
    SettlementAlreadyActive     = 6003,
    AlreadyFrozen               = 6004,
    OracleInvalid               = 6005,
    TooManyCollateralTypes      = 6006,
    WaitingPeriodNotElapsed     = 6007,
    RedemptionRateNotSet        = 6008,
    AlreadySet                  = 6009,
    InsufficientLSC             = 6010,
    ZeroAmount                  = 6011,
    AlreadyLiquidated           = 6012,
    InvalidPDA                  = 6013,
    Unauthorized                = 6014,
    SafeRedemptionPeriodActive  = 6015,
}
```
