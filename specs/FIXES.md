# LSC Specification Fixes

**Generated from:** specs/AUDIT.md
**Date:** 2026-03-13
**Scope:** All CRITICAL (🔴) and IMPORTANT (🟡) issues from the audit

This document lists the specific changes required, organized by file. Each fix references the corresponding audit finding.

---

## specs/01-system-architecture.md

### Fix 1.1 — PDA Derivation Format [CRITICAL — Audit §2.1, §7 C4]

**Problem:** All PDA derivation expressions use variable-length byte concatenation (e.g., `b"safe" || owner_id[32] || nonce[8]` = 44 bytes) but `PdaSeed` is exactly 32 bytes. The LEZ API is `PdaSeed([u8; 32])`.

**Change:** Replace the PDA derivation table in §3 (and all other sections that describe derivation) with the correct pattern. Each seed must be computed by SHA256-hashing the input material:

```
// Pattern (same as AMM program):
fn compute_pda_seed(input_bytes: &[u8]) -> PdaSeed {
    PdaSeed::new(Sha256::hash_bytes(input_bytes).as_bytes().try_into().unwrap())
}

// LSC PDA seeds:
safe_id          = LSC_ENGINE_ID.derive(compute_pda_seed(b"safe" || owner_id[32] || nonce[8]))
                   // input: 4 + 32 + 8 = 44 bytes → SHA256 → 32-byte PdaSeed

collateral_vault = LSC_ENGINE_ID.derive(compute_pda_seed(b"vault" || collateral_type_id[32]))
                   // input: 5 + 32 = 37 bytes → SHA256 → 32-byte PdaSeed

system_params    = LSC_ENGINE_ID.derive(compute_pda_seed(b"system_params"))
                   // input: 13 bytes → SHA256 → 32-byte PdaSeed

global_debt      = LSC_ENGINE_ID.derive(compute_pda_seed(b"global_debt"))

collateral_type  = LSC_ENGINE_ID.derive(compute_pda_seed(b"collateral_type" || collateral_symbol[32]))

// LSC token definition: MUST be a PDA of LSC_ENGINE_ID (see Fix 1.2)
lsc_token_def    = LSC_ENGINE_ID.derive(compute_pda_seed(b"lsc_token_definition"))

tax_collector    = TAX_COLLECTOR_ID.derive(compute_pda_seed(b"tax_collector_params"))

oracle_config    = ORACLE_PROGRAM_ID.derive(compute_pda_seed(b"oracle_config"))

feed             = ORACLE_PROGRAM_ID.derive(compute_pda_seed(b"feed" || feed_symbol[32]))

median           = ORACLE_PROGRAM_ID.derive(compute_pda_seed(b"median" || token_symbol[32]))

oracle_relayer   = ORACLE_RELAYER_ID.derive(compute_pda_seed(b"oracle_relayer_params"))

pi_controller    = PI_CONTROLLER_ID.derive(compute_pda_seed(b"pi_controller_state"))

liquidation_engine = LIQUIDATION_ENGINE_ID.derive(compute_pda_seed(b"liquidation_engine_params"))

auction_vault    = COLLATERAL_AUCTION_HOUSE_ID.derive(compute_pda_seed(b"auction_vault" || collateral_type_id[32]))

auction_state    = COLLATERAL_AUCTION_HOUSE_ID.derive(compute_pda_seed(b"auction" || auction_id[16]))

accounting_engine = ACCOUNTING_ENGINE_ID.derive(compute_pda_seed(b"accounting_engine_params"))

system_surplus   = ACCOUNTING_ENGINE_ID.derive(compute_pda_seed(b"system_surplus"))

system_debt_queue = ACCOUNTING_ENGINE_ID.derive(compute_pda_seed(b"system_debt_queue"))

surplus_auction  = SURPLUS_AUCTION_HOUSE_ID.derive(compute_pda_seed(b"surplus_auction_params"))

global_settlement = GLOBAL_SETTLEMENT_ID.derive(compute_pda_seed(b"global_settlement_state"))

settlement_collateral = GLOBAL_SETTLEMENT_ID.derive(compute_pda_seed(b"settlement_collateral" || collateral_type_id[32]))
```

Add a note: "Where input bytes include fixed-size fields (account IDs are 32 bytes, nonces are 8 bytes), the input is length-deterministic. For string prefixes, use the exact byte values shown."

Also clarify that `ProgramId` is `[u32; 8]`, not a string. The `LSC_ENGINE_ID` etc. are `ProgramId` constants.

### Fix 1.2 — LSC Token Definition Must Be an LSCEngine PDA [CRITICAL — Audit §2.4, §7 C3]

**Problem:** The spec says the LSC token definition has a `minting_authority: system_params_id` field. `TokenDefinition` has no such field. Minting authorization relies on `definition_account.is_authorized`, which in a chained call is set by including the definition's PDA seed in `ChainedCall.pda_seeds`.

**Change:** Add to §7 (Initialization Sequence) and §3 (PDA table):

> **LSC Minting Authority:** The LSC token definition account must be a PDA of `LSC_ENGINE_ID`. Its account ID is derived as:
> ```
> lsc_token_def_id = LSC_ENGINE_ID.derive(compute_pda_seed(b"lsc_token_definition"))
> ```
> When LSCEngine issues a `TokenProgram::Mint` chained call, it includes `pda_seeds: [compute_pda_seed(b"lsc_token_definition")]` in the `ChainedCall`. The LEZ runtime then marks `lsc_token_def_id` as `is_authorized = true` in the callee's account list, satisfying the Token Program's mint authorization check. There is no `minting_authority` field in `TokenDefinition`.
>
> **LOGOS Minting Authority:** The LSC system does not mint LOGOS. The LOGOS token definition is owned by the LOGOS deployment and is not a PDA of any LSC program. LSC programs only hold and transfer existing LOGOS tokens (never mint them).
>
> Remove all references to `minting_authority` in token definition schemas.

### Fix 1.3 — Document ChainedCall PDA Seeds Mechanism [IMPORTANT — Audit §2.5]

**Problem:** The spec says PDAs are authorized "via PDA authority" but never documents the `pda_seeds` mechanism.

**Change:** Add a new subsection (e.g., §2.5 "ChainedCall Authorization") explaining:

> **ChainedCall Authorization:**
> ```rust
> struct ChainedCall {
>     program_id: ProgramId,
>     pre_states: Vec<AccountWithMetadata>,
>     instruction_data: Vec<u8>,
>     pda_seeds: Vec<PdaSeed>,  // ← seeds of PDAs this program authorizes
> }
> ```
> When a calling program issues a `ChainedCall`, the LEZ runtime calls `compute_authorized_pdas(caller_program_id, pda_seeds)` to derive account IDs. Any account whose ID matches a derived PDA gets `is_authorized = true` in the callee's account list. This is the ONLY mechanism by which a program can authorize its PDAs in downstream calls.
>
> Every chained call description in specs 03–09 must list the `pda_seeds` the calling program includes.

---

## specs/02-account-schemas.md

### Fix 2.1 — CollateralVaultAccount and SystemSurplusAccount Owner [IMPORTANT — Audit §2.6]

**Problem:** `CollateralVaultAccount` is shown with `Owner: LSC_ENGINE_ID`. After `TokenProgram::InitializeAccount`, the account is claimed by `TOKEN_PROGRAM_ID` (see `initialize.rs`: `AccountPostState::new_claimed`). The same applies to `SystemSurplusAccount`.

**Change:** Update account headers:

```
// Before:
CollateralVaultAccount
  Owner: LSC_ENGINE_ID

// After:
CollateralVaultAccount
  Owner: TOKEN_PROGRAM_ID
  Note: Account ID is a PDA of LSC_ENGINE_ID. LSCEngine authorizes token transfers
        from this vault by including its PDA seed in ChainedCall.pda_seeds.
```

Apply the same correction to `SystemSurplusAccount` (Owner: `TOKEN_PROGRAM_ID`, ID is PDA of `ACCOUNTING_ENGINE_ID`), and any auction vault holding accounts.

### Fix 2.2 — Remove Spurious PIControllerStateAccount Field [IMPORTANT — Audit §3.4]

**Problem:** `PIControllerStateAccount` includes `integral_period_size: u64` which appears nowhere in the PI controller math or any instruction spec. The RAI model does not use a discrete integral period.

**Change:** Remove `integral_period_size: u64` from `PIControllerStateAccount`. If it is intended as a future feature, it must not appear in the v1 account schema without any referencing logic.

### Fix 2.3 — Add max_timestamp_jump to OracleConfigAccount [IMPORTANT — Audit §1.6, §3.5]

**Problem:** Spec 04 (OracleProgram::SubmitPrice) validates that `price_timestamp - feed.last_updated <= max_timestamp_jump` using a hardcoded value of 7200, but `OracleConfigAccount` does not include `max_timestamp_jump`. Spec 12 lists it as a tunable governance parameter.

**Change:** Add to `OracleConfigAccount`:
```
max_timestamp_jump: u64  // seconds; initial value 7200; range [300, 86400]; set by governance
```

Update the `SubmitPrice` spec in 04 to read `max_timestamp_jump` from `OracleConfigAccount` instead of hardcoding 7200.

### Fix 2.4 — Add SurplusAuctionHouseParamsAccount [IMPORTANT — Audit §1.5]

**Problem:** `SurplusAuctionHouse::Initialize` accepts system parameters, but no params account schema is defined in spec 02.

**Change:** Add:

```
SurplusAuctionHouseParamsAccount {
    governance_id: AccountId,
    bid_increase: u128,         // Wad; min bid increment (LOGOS); initial 50_000_000_000_000_000 (5%)
    bid_duration: u64,          // seconds; extension per bid; initial 10800 (3h)
    total_auction_length: u64,  // seconds; hard deadline; initial 259200 (3d)
}
// PDA: SURPLUS_AUCTION_HOUSE_ID.derive(compute_pda_seed(b"surplus_auction_params"))
```

### Fix 2.5 — Add safe_redemption_period to GlobalSettlementStateAccount [IMPORTANT — Audit §1.6]

**Problem:** `GlobalSettlement::SetFinalRedemptionRate` uses `SAFE_REDEMPTION_PERIOD` as a hardcoded constant, but spec 12 lists it as a tunable parameter with range 1–7 days.

**Change:** Add to `GlobalSettlementStateAccount`:
```
safe_redemption_period: u64  // seconds; set at Initialize; default 259200 (3 days); range [86400, 604800]
```

Update `GlobalSettlement::Initialize` to accept `safe_redemption_period: u64` as an instruction parameter. Update `SetFinalRedemptionRate` to read from `state.safe_redemption_period`.

---

## specs/03-core-engine.md

### Fix 3.1 — RepayDebt: Burn Chained Call Account Order [CRITICAL — Audit §2.2, §7 C1]

**Problem:** The `RepayDebt` chained call for `TokenProgram::Burn` has accounts in the wrong order. The spec shows `[caller_lsc_holding_id (auth), lsc_token_def_id]` but `burn.rs` requires `[definition_account, holder_account (is_authorized)]`.

**Change:**
```
// Before (WRONG):
ChainedCall → TokenProgram::Burn { amount_to_burn: lsc_to_burn }
  accounts: [caller_lsc_holding_id (is_authorized), lsc_token_def_id (writable)]

// After (CORRECT):
ChainedCall → TokenProgram::Burn { amount_to_burn: lsc_to_burn }
  accounts: [lsc_token_def_id (writable), caller_lsc_holding_id (is_authorized, writable)]
```

Note: `lsc_token_def_id` does NOT need `is_authorized`; it is only writable. The holder account must be `is_authorized`.

### Fix 3.2 — AddCollateralType: InitializeAccount Chained Call [CRITICAL — Audit §2.3, §7 C2]

**Problem:** The `AddCollateralType` instruction shows `TokenProgram::InitializeAccount` with named parameters (`account`, `definition_id`, `holder_id`) that do not exist. The Token Program `InitializeAccount` instruction takes no parameters — account positions determine its behavior.

**Change:**
```
// Before (WRONG):
ChainedCall → TokenProgram::InitializeAccount {
    account: collateral_vault_id,
    definition_id: logos_token_def_id,
    holder_id: collateral_vault_id,
}

// After (CORRECT):
ChainedCall → TokenProgram::InitializeAccount
  accounts: [logos_token_def_id (read), collateral_vault_id (writable)]
  // No instruction parameters. The Token Program initializes the account with:
  //   program_owner = TOKEN_PROGRAM_ID
  //   data = TokenHolding::Fungible { definition_id: logos_token_def_id, balance: 0 }
  // pda_seeds: [compute_pda_seed(b"vault" || collateral_type_id)]
  //   (LSCEngine authorizes collateral_vault_id as its PDA)
```

### Fix 3.3 — RepayDebt: Unit Error in Debt Subtraction [CRITICAL — Audit §3.2, §7 C6]

**Problem:** `RepayDebt` computes `lsc_to_burn` in Wad but the state update subtracts it from `safe.normalized_debt` (Wad) and `collateral_type.global_normalized_debt` (Wad) incorrectly. Tracing through the spec: `lsc_to_burn = normalized_debt_to_repay * accumulated_rate / RAY` is in Wad. The spec then writes `safe.normalized_debt -= lsc_to_burn` but it should subtract the normalized amount, not the LSC amount.

**Change:** Clarify the computation to maintain consistent units:
```
// normalized_debt_to_repay is in Wad (normalized debt units)
// accumulated_rate is in Ray
// lsc_to_burn = normalized_debt_to_repay * accumulated_rate / RAY  → result in Wad

// State updates (use normalized amount, not LSC amount):
safe.normalized_debt -= normalized_debt_to_repay           // Wad -= Wad ✓
collateral_type.global_normalized_debt -= normalized_debt_to_repay  // Wad -= Wad ✓

// The LSC amount burned is computed separately for the chained call only:
let lsc_to_burn: u128 = mul_wad(normalized_debt_to_repay, accumulated_rate);  // Wad
// chained call burns lsc_to_burn from the user's holding
```

Audit finding: The spec as written shows the debt state being decremented by `lsc_to_burn` (the Wad LSC amount) in one place and `normalized_debt_to_repay` elsewhere. Standardize on `normalized_debt_to_repay` for all state fields.

### Fix 3.4 — UpdateAccumulatedRate: Parameter Naming Ambiguity [IMPORTANT — Audit §3.1]

**Problem:** `LSCEngine::UpdateAccumulatedRate` takes a parameter called `new_rate` or `accumulated_rate_delta` in different parts of the spec. The description alternates between "the new accumulated rate" and "the delta to apply." This is a critical ambiguity.

**Change:** Standardize the parameter to:
```
// Instruction parameter:
new_accumulated_rate: u128  // Ray; the NEW absolute accumulated rate (not a delta)
                            // Caller (TaxCollector) computes this before calling

// Validation (in LSCEngine::UpdateAccumulatedRate):
assert!(new_accumulated_rate >= collateral_type.accumulated_rate, "Rate cannot decrease");

// State update:
let rate_delta_ray: u128 = new_accumulated_rate - collateral_type.accumulated_rate;
let normalized_debt_wad: u128 = collateral_type.global_normalized_debt;
// surplus_lsc_rad = normalized_debt_wad * rate_delta_ray  (Rad = Wad * Ray)
let surplus_lsc_rad: u128 = mul_ray_wad(normalized_debt_wad, rate_delta_ray);

collateral_type.accumulated_rate = new_accumulated_rate;
global_debt.total_debt += surplus_lsc_rad;  // Rad += Rad

// Chained call mints surplus_lsc_wad = surplus_lsc_rad / RAY (Wad) to system_surplus_id
```

Also confirm: `global_debt.total_debt` MUST be updated here (the spec is ambiguous about this; see Fix 7.1).

### Fix 3.5 — GenerateDebt: Add pda_seeds to Mint ChainedCall [CRITICAL — Audit §2.4]

**Problem:** The `GenerateDebt` chained call to `TokenProgram::Mint` does not specify `pda_seeds`. Without `pda_seeds: [compute_pda_seed(b"lsc_token_definition")]`, the `lsc_token_def_id` account will not be marked `is_authorized` in the callee, causing the mint to fail.

**Change:**
```
ChainedCall → TokenProgram::Mint { amount_to_mint: lsc_to_mint }
  accounts: [lsc_token_def_id (writable, is_authorized), user_lsc_holding_id (writable)]
  pda_seeds: [compute_pda_seed(b"lsc_token_definition")]
  // lsc_token_def_id is a PDA of LSCEngine derived from this seed.
  // The runtime marks it is_authorized because LSCEngine provides its seed.
```

Apply the same fix to all other `Mint` chained calls in the LSCEngine.

---

## specs/04-oracle-system.md

### Fix 4.1 — SubmitPrice: Read max_timestamp_jump from Config [IMPORTANT — Audit §1.6]

**Problem:** `OracleProgram::SubmitPrice` hardcodes `7200` for `max_timestamp_jump` instead of reading from `OracleConfigAccount`.

**Change:**
```
// Before:
assert!(
    price_timestamp - feed.last_updated <= 7200,
    "Timestamp jump too large"
);

// After:
assert!(
    price_timestamp <= feed.last_updated + oracle_config.max_timestamp_jump,
    "Timestamp jump exceeds max_timestamp_jump"
);
// oracle_config must be in the required accounts list for SubmitPrice
```

Add `oracle_config_id (read)` to the required accounts for `SubmitPrice`.

---

## specs/05-pi-controller.md

### Fix 5.1 — PIController: Fix i128 Overflow in error_ray Multiplication [CRITICAL — Audit §3.3, §7 C7]

**Problem:** The spec computes:
```
proportional_component = kp * error_ray
```
where `kp: i128` (Ray, ≈ 5e24) and `error_ray: i128` (Ray, up to ≈ 1e27). The product can reach 5e24 × 1e27 = 5e51, which overflows i128 (max ≈ 1.7e38).

**Change:** Use 256-bit intermediate arithmetic:

```rust
// Use u256/i256 intermediate multiplication:
let proportional_component: i128 = i256::from(kp)
    .checked_mul(i256::from(error_ray))
    .expect("overflow")
    .checked_div(i256::from(RAY))  // normalize: (Ray * Ray) / Ray = Ray
    .expect("div by zero")
    .try_into()
    .expect("result too large for i128");
```

Similarly for `integral_component = ki * error_integral / RAY`.

The spec must document that all Ray×Ray multiplications must use 256-bit intermediates (or checked 128-bit with bounds enforcement on inputs).

### Fix 5.2 — PIController: Fix i128 Overflow in Integral Decay [CRITICAL — Audit §3.3]

**Problem:** `error_integral * per_second_cumulative_leak / RAY` — if `error_integral` is near i128::MAX and `per_second_cumulative_leak` is RAY (1e27), the multiplication overflows.

**Change:** Identical pattern — use i256 intermediate:
```rust
let new_integral: i128 = i256::from(error_integral)
    .checked_mul(i256::from(per_second_cumulative_leak))
    .expect("overflow")
    .checked_div(i256::from(RAY))
    .expect("div by zero")
    .try_into()
    .expect("integral too large after decay");
```

Document the valid range for `error_integral` and add a clamping step after each update.

### Fix 5.3 — PIController: Clarify Kp Sign Convention [IMPORTANT — Audit §3.6]

**Problem:** The spec says "positive Kp means: if LSC > target, raise redemption rate" but then shows an example where a positive error (LSC above target) produces a positive proportional term that is added to RAY, which would raise the rate. But separately the spec defines `error = target - market` which makes error negative when market > target, giving a negative proportional term. The two descriptions contradict.

**Change:** Lock in the convention explicitly:

```
error_ray = redemption_price - market_price  // positive when LSC is below target
                                              // negative when LSC is above target

proportional = Kp * error_ray / RAY
// With Kp > 0:
//   LSC above target → error negative → proportional negative → rate decreases
//   LSC below target → error positive → proportional positive → rate increases
// This is NEGATIVE feedback (stabilizing).

redemption_rate = clamp(RAY + proportional + integral, lower_bound, upper_bound)
```

Remove or correct any example that contradicts this convention.

### Fix 5.4 — PIController UpdateRate: Check settlement_active [IMPORTANT — Audit §4.4]

**Problem:** `PIController::UpdateRate` does not check `system_params.settlement_active`. During global settlement, the redemption rate should be frozen.

**Change:** Add to `UpdateRate` preconditions:
```
// Required accounts: add system_params_id (read)
assert!(!system_params.settlement_active, "Cannot update rate during settlement");
```

---

## specs/06-liquidation-engine.md

### Fix 6.1 — LiquidateSafe: Correct Chained Call Count [IMPORTANT — Audit §3.7]

**Problem:** The spec states the liquidation flow uses "3 chained calls" but the actual flow (per spec 09 §3.3) is 4 chained calls:
1. LSCEngine::ConfiscateSafe
2. TokenProgram::Transfer (collateral → auction vault)  ← nested inside #1
3. AccountingEngine::PushDebt
4. CollateralAuctionHouse::StartAuction

**Change:** Update the comment in `LiquidateSafe` to "4 chained calls (within the LEZ limit of 10)." Verify that the nested call model (call #2 is nested within call #1) is counted correctly against the `MAX_NUMBER_CHAINED_CALLS = 10` limit. If LEZ counts total chained calls across the entire transaction tree (not just top-level), confirm the count remains within 10.

### Fix 6.2 — RestartAuction: Define discount_increment [IMPORTANT — Audit §1.4]

**Problem:** `CollateralAuctionHouse::RestartAuction` references `discount_increment` which is never defined.

**Change:** Add `discount_increment: u128 (Wad)` to `CollateralAuctionHouseParamsAccount`:
```
CollateralAuctionHouseParamsAccount {
    ...existing fields...
    discount_increment: u128,  // Wad; amount to increase discount per restart; initial 10_000_000_000_000_000 (1%)
}
```

Update `RestartAuction` to read `params.discount_increment`:
```
auction.current_discount = min(
    auction.current_discount + params.discount_increment,
    params.max_discount
);
```

### Fix 6.3 — StartAuction: Add collateral_type_id Parameter [IMPORTANT — Audit §1.7]

**Problem:** `CollateralAuctionHouse::StartAuction` (called by `LiquidationEngine`) does not list `collateral_type_id` in required accounts, but the auction must record which collateral type it's for.

**Change:** Add `collateral_type_id: AccountId` as an instruction parameter or required account in `StartAuction`. The created `CollateralAuctionStateAccount` must store `collateral_type_id` so `BuyCollateral` knows which vault to draw from.

---

## specs/07-accounting-engine.md

### Fix 7.1 — UpdateAccumulatedRate: Update global_debt.total_debt [IMPORTANT — Audit §3.1]

**Problem:** `LSCEngine::UpdateAccumulatedRate` (called by TaxCollector) computes new surplus LSC but the spec is inconsistent about whether `GlobalDebtAccount.total_debt` is updated. In some places the update is shown; in others it is omitted. Without updating `total_debt`, the accounting engine's surplus and debt logic will compute incorrect values.

**Change:** Add explicitly to `UpdateAccumulatedRate`'s state transitions:
```
// After computing surplus_lsc_rad:
global_debt.total_debt += surplus_lsc_rad;  // Rad += Rad
// This tracks the interest accrued as system debt until minted to surplus
```

The required accounts for `UpdateAccumulatedRate` must include `global_debt_id (writable)`.

### Fix 7.2 — PushDebt: Correct Authorization Check ✅ RESOLVED

**Status:** The spec has been updated. `AccountingEngineParamsAccount` now stores `liquidation_engine_params_id: [u8; 32]`. `PushDebt` validation correctly uses this field.

### Fix 7.3 — Initialize SurplusAuction LOGOS Holding Account [IMPORTANT]

**Problem:** `SurplusAuctionHouse::StartAuction` uses `auction_logos_holding_id` but never initializes it as a Token Program holding account.

**Change:** Add to `SurplusAuction::StartAuction` chained calls:
```
// During StartAuction, chain a call to create the per-auction LOGOS holding account:
ChainedCall → TokenProgram::InitializeAccount
  accounts: [logos_token_def_id (read), auction_logos_holding_id (writable, new)]
  pda_seeds: [compute_pda_seed(b"surplus_auction_logos" || auction_nonce[8])]
  // After this call, auction_logos_holding_id.program_owner = TOKEN_PROGRAM_ID
  // and is ready to hold LOGOS bids during the auction
```

---

## specs/08-global-settlement.md

### Fix 8.1 — FreezeCollateralType: Cross-Program Ownership [CRITICAL — Audit §4.1, §7 C9]

**Problem:** `GlobalSettlement::FreezeCollateralType` writes directly to `CollateralTypeAccount` (e.g., setting `frozen = true`). But `CollateralTypeAccount.program_owner = LSC_ENGINE_ID`. LEZ's `validate_execution` enforces that only the owning program can modify an account's data. `GlobalSettlement` cannot directly write to `CollateralTypeAccount`.

**Change:** `GlobalSettlement::FreezeCollateralType` must issue a chained call to `LSCEngine`:

```
// In GlobalSettlement::FreezeCollateralType:
ChainedCall → LSCEngine::FreezeCollateralType { collateral_type_id }
  accounts: [
    global_settlement_state_id (read, is_authorized),  // proves caller is GlobalSettlement
    collateral_type_id (writable),
    system_params_id (read),
  ]
  pda_seeds: [compute_pda_seed(b"global_settlement_state")]

// In LSCEngine::FreezeCollateralType (new instruction):
// - Verify caller has is_authorized global_settlement_state_id (a known GlobalSettlement PDA)
// - assert!(!system_params.settlement_active || collateral_type not already frozen)
// - collateral_type.frozen = true
// - collateral_type.final_collateral_price = settlement_price (from instruction param)
```

The spec must add `LSCEngine::FreezeCollateralType` as a new privileged instruction and update `GlobalSettlement::FreezeCollateralType` to use it.

### Fix 8.2 — Settlement Accounting Bug: collateral_available vs vault.balance [CRITICAL — Audit §4.2, §7 C8]

**Problem:** `GlobalSettlement::RedeemCollateral` decrements `settlement_collateral.collateral_available` by `safe.collateral` (the full collateral amount), but only transfers `net_collateral = safe.collateral - safe.normalized_debt * accumulated_rate / final_collateral_price` to the safe owner. This means `collateral_available` is decremented by more than what was sent out, causing the computed `collateral_per_lsc` in `SetFinalRedemptionRate` to be too low, and capping `RedeemLSC` incorrectly.

**Change:** Choose one of:

**Option A (Recommended):** Decrement `collateral_available` by `net_collateral` only (the amount actually sent to the user). The remaining (debt-coverage portion) stays available for LSC holders.
```
// In RedeemCollateral:
let net_collateral = safe.collateral - (safe.normalized_debt * accumulated_rate / final_collateral_price);
settlement_collateral.collateral_available -= net_collateral;  // ← only net, not full collateral
// Transfer net_collateral to safe owner
```

**Option B:** Track two counters: `collateral_reserved_for_debt` and `collateral_available_for_lsc`. Decrement only `collateral_available_for_lsc` by `net_collateral` during `RedeemCollateral`. During `SetFinalRedemptionRate`, compute `collateral_per_lsc` from `collateral_available_for_lsc` only.

### Fix 8.3 — GlobalSettlementError: Remove Duplicate Unauthorized [IMPORTANT — Audit §4.3]

**Problem:** `GlobalSettlementError` defines `Unauthorized = 6001` and `Unauthorized = 6014` — duplicate discriminant names.

**Change:** Remove one duplicate. Assign distinct error codes to all distinct error conditions. If there are two different authorization failures (e.g., non-governance and non-settlement-state), give them distinct names:
```
// Before:
Unauthorized = 6001
...
Unauthorized = 6014  // duplicate!

// After:
NotGovernance = 6001       // caller is not the governance account
NotGlobalSettlement = 6014 // caller is not the global settlement program
```

---

## specs/09-token-integration.md

### Fix 9.1 — InitializeAccount Pattern: Correct Account Order [CRITICAL — Audit §2.3]

**Problem:** §4 shows:
```
Account list:
  new_holding_account_id (PDA, writable, is_authorized=true)
  token_def_id (read)
```
The definition account must come first, and `InitializeAccount` does not check `is_authorized` on any account.

**Change:**
```
// Before (WRONG):
Account list:
  new_holding_account_id (PDA, writable, is_authorized=true)
  token_def_id (read)

// After (CORRECT):
Account list:
  token_def_id (read)
  new_holding_account_id (writable)
  // No is_authorized required. Anyone can initialize a holding account for any definition.
  // The new account is claimed by TOKEN_PROGRAM_ID after initialization.
  pda_seeds: [<seed for new_holding_account_id>]  // if new_holding_account_id is a PDA of the caller
```

### Fix 9.2 — All Burn Patterns: Correct Account Order [CRITICAL — Audit §2.2]

**Problem:** All `Burn` chained call examples in §1.4 show the holder account first and definition second. `burn.rs` requires definition first.

**Change:** Update table in §1.4 and all chained call patterns:
```
// Before (WRONG):
ChainedCall → TokenProgram::Burn
  accounts: [holder_account (is_authorized), token_def_id (writable)]

// After (CORRECT):
ChainedCall → TokenProgram::Burn { amount_to_burn: X }
  accounts: [token_def_id (writable), holder_account (is_authorized, writable)]
```

Apply to: RepayDebt, SettleDebt, SurplusAuction::SettleAuction, GlobalSettlement::RedeemLSC, and any other burn operations.

---

## NEW FILE: specs/03b-tax-collector.md (or expand specs/03-core-engine.md)

### Fix N.1 — Add TaxCollector Program Specification [CRITICAL — Audit §1.1, §7 C5]

**Problem:** TaxCollector has no instruction specification. An implementer cannot implement it.

**Change:** Create a new spec section (either a new file `03b-tax-collector.md` or a new major section in `03-core-engine.md`) with the following content:

---

### TaxCollector Program

The TaxCollector is responsible for accruing per-second stability fees to active collateral types. It is called by a keeper (off-chain bot) and chains to `LSCEngine::UpdateAccumulatedRate`.

#### TaxCollectorParamsAccount

```
TaxCollectorParamsAccount {
    governance_id: AccountId,
    global_stability_fee: u128,  // Ray, per-second; additive base fee
}
// PDA: TAX_COLLECTOR_ID.derive(compute_pda_seed(b"tax_collector_params"))
```

#### Instruction: Initialize

**Parameters:** `governance_id: AccountId, global_stability_fee: u128`
**Required accounts:**
- `tax_collector_params_id` (writable, new PDA of TAX_COLLECTOR_ID)
- `caller_account` (is_authorized)

**Preconditions:**
- `tax_collector_params_id` must not exist (first initialization)
- `global_stability_fee >= RAY` (non-negative rate)

**State transitions:**
```
params.governance_id = governance_id
params.global_stability_fee = global_stability_fee
```

#### Instruction: AccrueStabilityFee

**Purpose:** Compute and apply the elapsed stability fee to a collateral type, updating its accumulated_rate.

**Parameters:** *(none — all data from accounts)*
**Required accounts:**
- `tax_collector_params_id` (read)
- `collateral_type_id` (read — to get stability_fee and current accumulated_rate)
- `oracle_config_id` (read — for current timestamp)
- `caller_account` (is_authorized — any caller; this is a public keeper instruction)

**Preconditions:**
```
now = current_block_timestamp  // from ledger clock
elapsed = now - collateral_type.last_stability_fee_update
assert!(elapsed > 0, "No time elapsed")
```

**Stability fee computation:**
```
// Effective per-second rate (additive combination):
effective_rate_ray = params.global_stability_fee + collateral_type.stability_fee - RAY
// (subtracting RAY once because both are (RAY + per_second_delta))

// New accumulated rate:
// rate^elapsed using repeated squaring (binary exponentiation in Ray arithmetic):
rate_multiplier = rpow(effective_rate_ray, elapsed, RAY)
// rpow(base, exp, precision): Ray exponentiation
// base: effective_rate_ray in Ray; exp: seconds elapsed; result in Ray

new_accumulated_rate = ray_mul(collateral_type.accumulated_rate, rate_multiplier)
// Wad * Ray = Rad precision, then / RAY → Wad? No: accumulated_rate is Ray, multiplier is Ray
// new_accumulated_rate = accumulated_rate * rate_multiplier / RAY  (Ray * Ray / Ray = Ray)
```

**Chained calls:**
```
ChainedCall → LSCEngine::UpdateAccumulatedRate {
    collateral_type_id: collateral_type_id,
    new_accumulated_rate: new_accumulated_rate,
}
  accounts: [
    collateral_type_id (writable),
    global_debt_id (writable),
    system_surplus_id (writable),
    tax_collector_params_id (read, is_authorized),  // proves caller is TaxCollector
    lsc_token_def_id (writable),
    user_lsc_holding_id = system_surplus_id,
  ]
  pda_seeds: [compute_pda_seed(b"tax_collector_params")]
```

**Note:** `rpow(base, exp, precision)` must be implemented as binary exponentiation using 256-bit intermediate arithmetic to avoid overflow. Reference: MakerDAO's `_rpow` implementation.

---

## Summary of Required Changes by Severity

| File | Critical Fixes | Important Fixes |
|---|---|---|
| 01-system-architecture.md | PDA format (C4), LSC token def as PDA (C3) | ChainedCall pda_seeds doc |
| 02-account-schemas.md | — | Vault owner, spurious integral_period_size, max_timestamp_jump, surplus auction params, safe_redemption_period |
| 03-core-engine.md | Burn order (C1), InitializeAccount (C2), RepayDebt unit error (C6), GenerateDebt pda_seeds (C3) | UpdateAccumulatedRate parameter |
| 04-oracle-system.md | — | max_timestamp_jump from config |
| 05-pi-controller.md | i128 overflow (C7) | Kp sign convention, settlement check |
| 06-liquidation-engine.md | — | Chained call count, discount_increment, StartAuction params |
| 07-accounting-engine.md | — | global_debt.total_debt update, PushDebt auth (✅ resolved), SurplusAuction holding account init |
| 08-global-settlement.md | FreezeCollateralType cross-program (C9), settlement accounting (C8) | Duplicate error code |
| 09-token-integration.md | InitializeAccount order (C2), Burn order (C1) | — |
| NEW: 03b-tax-collector.md | TaxCollector specification (C5) | — |
