# LSC Specification Audit Report

**Auditor:** Claude Sonnet 4.6
**Date:** 2026-03-13
**Scope:** specs/00 through specs/12, cross-referenced against the LEZ codebase at `/tmp/logos-execution-zone`

---

## Executive Summary

The LSC specifications are well-structured and architecturally sound. The RAI model has been thoughtfully adapted to LEZ's stateless account model, and most concepts are correctly translated. The audit found **9 critical issues** and **14 important issues**. The most severe issues relate to incorrect Token Program chained call account ordering (which will panic at runtime), a fundamental misunderstanding of how minting authority works in the Token Program, an absent TaxCollector program specification, and a settlement accounting bug.

---

## 1. Completeness Check

### 1.1 Missing TaxCollector Program Specification

The TaxCollector is a core program (responsible for accruing stability fees and triggering `LSCEngine::UpdateAccumulatedRate`) but has **no instruction specification**. The specs show `TaxCollectorParamsAccount` (02), reference the program in flow diagrams (01, 09), and show `UpdateAccumulatedRate` as a privileged call from TaxCollector (03), but never define:
- The TaxCollector instruction enum
- The `AccrueStabilityFee` instruction's required accounts, validations, state transitions, or chained calls
- How TaxCollector computes the new accumulated_rate from stability_fee and elapsed time
- How TaxCollector reads the oracle timestamp

**An implementer cannot implement TaxCollector from these specs.**

### 1.2 ModifySafe Instruction Underspecified

`LSCEngine::ModifySafe` (03) says "see individual instructions" for required accounts and provides only a summary state transition. No complete account list is given. An implementer must union accounts from DepositCollateral/WithdrawCollateral and GenerateDebt/RepayDebt depending on delta signs, but the exact unioned list is not specified.

### 1.3 GlobalSettlement Initialize Missing Governance Parameter

`GlobalSettlement::Initialize` (08) takes no instruction parameters and sets no `governance_id` in `GlobalSettlementStateAccount`. But `TriggerSettlement` checks `governance_account.id == system_params.governance_id`. If governance is only stored in `SystemParamsAccount`, it cannot be checked independently by GlobalSettlement without reading it — but `SystemParamsAccount` is already a required account in `TriggerSettlement`, so this works. However, the GlobalSettlement state should store governance_id for standalone verification.

### 1.4 RestartAuction Discount Increment Undefined

`CollateralAuctionHouse::RestartAuction` (06) references `discount_increment` in its algorithm but this value is never defined anywhere — not in the params account, not as a constant, not as a parameter. This is a specification gap.

### 1.5 Surplus Auction House Params Account Missing

`SurplusAuctionHouse::Initialize` (07) takes parameters (`bid_increase`, `bid_duration`, `total_auction_length`), but there is no `SurplusAuctionHouseParamsAccount` schema in spec 02. This must exist — `IncreaseBidSize` references `auction.bid_duration` and `auction.bid_increase` which are per-auction copies, but `StartAuction` needs to read system-level defaults. A params account schema and PDA must be specified.

### 1.6 SAFE_REDEMPTION_PERIOD Not Parameterized

`GlobalSettlement::SetFinalRedemptionRate` (08) uses `SAFE_REDEMPTION_PERIOD` (hardcoded as "3 days = 259200 seconds" in comments) but this constant is never stored in any account. It should be a parameter in `GlobalSettlementStateAccount` or set at `Initialize` time. Spec 12 lists it as a tunable parameter, confirming it should be stored.

### 1.7 AuctionSurplus Missing Required Accounts

`AccountingEngine::AuctionSurplus` (07) needs `system_surplus_id` to compute `surplus_balance_rad`, and oracle accounts for timestamps, but these are not uniformly listed in the Required Accounts table.

---

## 2. LEZ Compatibility

### 2.1 PDA Derivation Format is Incompatible with Actual LEZ API 🔴

**File:** 01-system-architecture.md §3, 02-account-schemas.md headers

The specs describe PDA derivation using a variable-length byte concatenation notation:
```
safe_id = LSC_ENGINE_ID.derive(b"safe" || owner_account_id[0..32] || nonce[0..8])
```

The actual LEZ API uses `PdaSeed([u8; 32])` — a **fixed 32-byte** seed. The `AccountId::from((&program_id, &pda_seed))` function in `nssa/core/src/program.rs` takes exactly 32 bytes. The spec's seed `b"safe" || owner_id[32] || nonce[8]` = 44 bytes, which exceeds the 32-byte limit.

The AMM program demonstrates the correct pattern: hash the seed material to 32 bytes before constructing a `PdaSeed`:
```rust
// AMM pattern (amm/core/src/lib.rs):
let mut bytes = [0; 64];
bytes[0..32].copy_from_slice(&token_1.into_value());
bytes[32..].copy_from_slice(&token_2.into_value());
PdaSeed::new(Sha256::hash_bytes(&bytes).as_bytes().try_into().unwrap())
```

Every PDA derivation in the LSC specs must specify the exact bytes to hash (input format, length, prefix) to produce the 32-byte seed. Without this, two different implementers will produce different PDA addresses. **An implementer cannot deterministically compute LSC PDAs from these specs.**

Additionally, `ProgramId` is `[u32; 8]` (not a string), so `LSC_ENGINE_ID` is a numeric array, not a string constant. The specs should show the actual API call pattern.

### 2.2 Token Program Burn Account Order is Wrong Across All Specs 🔴

**Files:** 03-core-engine.md (RepayDebt), 07-accounting-engine.md (SettleDebt, SurplusAuction::SettleAuction), 08-global-settlement.md (RedeemLSC)

From `programs/token/src/burn.rs`:
```rust
pub fn burn(
    definition_account: AccountWithMetadata,   // ← FIRST
    user_holding_account: AccountWithMetadata,  // ← SECOND (must be is_authorized)
    amount_to_burn: u128,
) -> Vec<AccountPostState> {
    assert!(user_holding_account.is_authorized, "Authorization is missing");
```

**Correct order:** `[token_definition_account, holder_account (is_authorized)]`

Every `Burn` chained call in the specs has the order **reversed**:

- **03 RepayDebt:** `// accounts: [caller_lsc_holding_id (auth), lsc_token_def_id]` ❌
  Should be: `[lsc_token_def_id, caller_lsc_holding_id (auth)]`

- **07 SettleDebt:** `// accounts: [system_surplus_id (PDA auth), lsc_token_def_id]` ❌
  Should be: `[lsc_token_def_id, system_surplus_id (PDA auth)]`

- **07 SurplusAuction::SettleAuction:** `// accounts: [auction_logos_holding_id (PDA auth), logos_token_def_id]` ❌
  Should be: `[logos_token_def_id, auction_logos_holding_id (PDA auth)]`

- **08 RedeemLSC:** `// accounts: [holder_lsc_holding_id (auth), lsc_token_def_id]` ❌
  Should be: `[lsc_token_def_id, holder_lsc_holding_id (auth)]`

This will cause runtime panics (`"Token Definition account must be valid"` or deserialization failure) when attempting to deserialize a holding account as a definition.

### 2.3 Token Program InitializeAccount Account Order Wrong and Has Non-Existent Parameters 🔴

**Files:** 03-core-engine.md (AddCollateralType chained call), 09-token-integration.md §4

From `programs/token/src/initialize.rs`:
```rust
pub fn initialize_account(
    definition_account: AccountWithMetadata,   // ← FIRST
    account_to_initialize: AccountWithMetadata, // ← SECOND
) -> Vec<AccountPostState> {
```

The `InitializeAccount` instruction takes **no instruction parameters** — it only uses account positions. The holder of the new account is determined implicitly: the account's ID becomes the holder (set via `zeroized_from_definition`).

**Spec 03 (AddCollateralType):**
```
TokenProgram::InitializeAccount {
    account: collateral_vault_id,
    definition_id: logos_token_def_id,
    holder_id: collateral_vault_id,
}
```
This shows non-existent parameters. The actual call has no named params.

**Spec 09 §4:**
```
Account list:
  new_holding_account_id (PDA, writable, is_authorized=true)  // WRONG: should be definition first
  token_def_id (read)
```

Correct format:
```
Account list:
  token_def_id (read)
  new_holding_account_id (PDA, writable)
```

Note: `InitializeAccount` does not check any authorization flag — anyone can initialize a holding account for any definition.

### 2.4 LSC Minting Authority Mechanism is Fundamentally Wrong 🔴

**Files:** 01-system-architecture.md §7, 09-token-integration.md §1.3, §1.4

The spec states:
```
LscTokenDefinitionAccount:
  minting_authority: system_params_id (LSC Engine PDA)
```

And:
```
"Only the LSCEngine's system_params_id PDA can authorize minting of LSC."
```

**The Token Program's `TokenDefinition` has no `minting_authority` field:**
```rust
pub enum TokenDefinition {
    Fungible {
        name: String,
        total_supply: u128,
        metadata_id: Option<AccountId>,
    },
    // ...
}
```

The `Mint` instruction checks `definition_account.is_authorized`. This `is_authorized` flag is set in the `AccountWithMetadata` passed to the program. For a chained call, the calling program can mark its own PDAs as authorized by including their seeds in `ChainedCall.pda_seeds`.

**The correct mechanism:** The LSC token definition account ID must itself be a **PDA of LSC_ENGINE_ID**. Then, when LSCEngine issues a chained call to `TokenProgram::Mint`, it includes the PDA seed for the definition in `ChainedCall.pda_seeds`, which causes the ledger to set `definition_account.is_authorized = true` for that PDA.

The spec never specifies that the LSC token definition must be a PDA of LSCEngine, and never explains the `pda_seeds` mechanism for chained call authorization. Without this, there is no mechanism for LSCEngine to mint LSC. Note: the LSC system does not mint LOGOS — LSC programs only hold and transfer existing LOGOS tokens.

### 2.5 ChainedCall PDA Seeds Authorization Mechanism Never Explained 🟡

**Files:** 03-core-engine.md (all chained calls), 09-token-integration.md

The spec says PDAs are authorized "because the engine derives this PDA and can sign for it" and "via PDA authority" but never explains the actual mechanism: the `ChainedCall` struct has a `pda_seeds: Vec<PdaSeed>` field. When a chained call is executed, the runtime calls `compute_authorized_pdas(caller_program_id, pda_seeds)` to determine which account IDs are authorized for the callee. Any account ID in the resulting set has `is_authorized = true` in the callee's account list.

This needs to be documented, and every chained call spec must specify which `pda_seeds` the calling program must include.

### 2.6 Account Ownership Described Incorrectly for Token Holdings 🟡

**Files:** 02-account-schemas.md §1.4, §7.2

The `CollateralVaultAccount` spec says `Owner: LSC_ENGINE_ID` but this account is a Token Program holding. After `TokenProgram::InitializeAccount`, the account is `new_claimed` by TOKEN_PROGRAM_ID (as seen in `initialize.rs`: `AccountPostState::new_claimed(account_to_initialize)`). The LEZ `validate_execution` rules enforce that only the owning program can mutate account data.

`CollateralVaultAccount` owner = `TOKEN_PROGRAM_ID`. The PDA relationship is: the vault account's **ID** is derived from `LSC_ENGINE_ID`, which means LSCEngine can authorize it in chained calls (as a PDA). Similarly for `SystemSurplusAccount`.

This confusion between "owner" (program_owner in the Account struct) and "who can authorize transfers" must be clarified in the spec.

### 2.7 validate_execution Nonce Rule Not Addressed 🟢

**Files:** All program specs

`validate_execution` in `program.rs` enforces: `pre.account.nonce != post.account.nonce → invalid`. The nonce is immutable. LSC specs never mention nonces; implementers might accidentally try to use nonces as counters. The spec should note this constraint.

### 2.8 pre_states and post_states Must Have Same Length 🟢

`validate_execution` checks `pre_states.len() != post_states.len() → invalid`. Implementations must include every account in pre_states in post_states, even if unchanged. The spec doesn't state this explicitly.

---

## 3. RAI Model Accuracy

### 3.1 UpdateAccumulatedRate Parameter Ambiguity Causes Formula Inconsistency 🔴

**File:** 03-core-engine.md §UpdateAccumulatedRate

The instruction parameter has a contradictory description:
```rust
UpdateAccumulatedRate {
    new_rate_multiplier: u128,   // Ray — the delta to multiply in
}
```
"The delta to multiply in" implies a multiplicative factor: `new_rate = old_rate * new_rate_multiplier / RAY`.

But the state transition treats it as the new absolute accumulated_rate:
```
collateral_type.accumulated_rate = new_rate_multiplier  // pre-computed by TaxCollector
```

And the surplus formula uses:
```
surplus_rad = (new_rate_multiplier - old_rate) * collateral_type.global_normalized_debt
```

If `new_rate_multiplier` is the new accumulated_rate, this surplus formula is correct (new - old gives the rate delta, times normalized debt gives Rad surplus). If it's a multiplicative delta, `surplus_rad = (old_rate * new_rate_multiplier / RAY - old_rate) * global_normalized_debt`.

**These are different values.** The spec must clearly define which it is. Based on RAI's `vat.fold()` (which takes the new rate directly), the correct interpretation is: `new_rate_multiplier` = the NEW accumulated_rate (absolute). The parameter name and comment must be corrected.

Additionally, the validation `new_rate_multiplier >= RAY` is wrong: it should be `new_rate_multiplier >= collateral_type.accumulated_rate` (the rate can only increase).

### 3.2 RepayDebt global_debt Update Has Unit Error 🔴

**File:** 03-core-engine.md §RepayDebt

```
global_debt.total_debt -= lsc_to_burn  // Rad approx
```

`lsc_to_burn` is in Wad (units of LSC with 18 decimal places). `global_debt.total_debt` is in Rad (45 decimal places). The comment "Rad approx" acknowledges this but doesn't fix it. The correct update is:

```
global_debt.total_debt -= lsc_to_burn * RAY  // Convert Wad → Rad
```

or equivalently, since `lsc_to_burn = amount * accumulated_rate / RAY`:
```
global_debt.total_debt -= amount * accumulated_rate  // already in Rad
```

This unit error would result in `total_debt` underflowing catastrophically on the first repayment.

### 3.3 GlobalDebtAccount.total_debt Not Updated in UpdateAccumulatedRate 🟡

**File:** 03-core-engine.md §UpdateAccumulatedRate

When `accumulated_rate` increases, the effective debt (normalized_debt × accumulated_rate) increases for all SAFEs. `GlobalDebtAccount.total_debt` (in Rad) is intended to track total effective debt but is never updated in `UpdateAccumulatedRate`. Over time it becomes stale.

In MakerDAO's `vat.fold()`, the equivalent `debt` variable is updated: `debt += utab * rate` where `utab` is the total normalized debt. LSC should do the same:
```
global_debt.total_debt += (new_rate - old_rate) * collateral_type.global_normalized_debt
```

### 3.4 PI Controller — Integral Multiplication Will Overflow i128 🟡

**File:** 05-pi-controller.md §3.3 UpdateRate

```rust
let integral_decayed: i128 = state.error_integral * leak_factor / RAY as i128;
```

`state.error_integral` can be up to ~`error_ray * dt = RAY * seconds = 10^27 * 86400 ≈ 8.6 × 10^31`. `leak_factor` from `rpow(RAY * 0.99999992, dt, RAY)` can be up to `RAY = 10^27`. Their product `8.6e31 × 10^27 = 8.6e58` massively overflows i128 (max ≈ 1.7 × 10^38).

Similarly, `error_ray * dt as i128` for large dt: `4.5e25 × 86400 = 3.9e30` is within i128 range, but when summed with the decayed integral over many updates this can grow.

All PI controller intermediate arithmetic must use 256-bit (i256 or u256) values. The spec's security checklist (11) correctly mentions 256-bit arithmetic for LSCEngine but misses the PI controller.

### 3.5 PI Controller — error_ray Computation Will Overflow i128 🟡

**File:** 05-pi-controller.md §3.3

```rust
let error_ray: i128 = ((p_r - p_m_ray) as i128 * (RAY as i128)) / p_r as i128;
```

`(p_r - p_m_ray) ≈ 10^26` × `RAY = 10^27` = `10^53`, which overflows i128 (max ~1.7×10^38). This needs u256 intermediate arithmetic. The spec mentions u256 for collateral ratio checks (spec 03) but not here.

### 3.6 PI Controller — Kp Sign Convention Contradicts Itself 🟢

**File:** 05-pi-controller.md §2

The numeric walkthrough first shows `Kp = -5e24` (negative convention from RAI), then immediately states "Using positive Kp convention" with `Kp = 5e22`. The parameter table in spec 12 uses `kp = 5e24` (positive). The contradictory example will confuse implementers. Only the positive convention should be documented.

### 3.7 Liquidation Chained Call Count Inconsistency 🟢

**File:** 06-liquidation-engine.md §2.3 vs 09-token-integration.md §3.3

Spec 06 states: "This sequence uses 3 chained calls." but the actual sequence is:
1. `LSCEngine::ConfiscateSafe`
2. ↳ `TokenProgram::Transfer` (nested from within ConfiscateSafe)
3. `AccountingEngine::PushDebt`
4. `CollateralAuctionHouse::StartAuction`

Spec 09 correctly states 4 chained calls. The "3" in spec 06 should be "4" (with a note that one is nested).

---

## 4. Consistency Check

### 4.1 GlobalSettlement FreezeCollateralType Cannot Modify CollateralTypeAccount 🔴

**Files:** 08-global-settlement.md §3.3, 03-core-engine.md §FreezeCollateralType

`CollateralTypeAccount` is owned by `LSC_ENGINE_ID`. LEZ's `validate_execution` rule 6 states: data changes are only allowed if owned by the executing program. `GlobalSettlement` cannot directly write `collateral_type.active = false` or `collateral_type.final_collateral_price = ...` because it doesn't own the account.

The spec defines `LSCEngine::FreezeCollateralType` (spec 03) precisely for this purpose, but spec 08's `FreezeCollateralType` instruction never issues a chained call to `LSCEngine::FreezeCollateralType`. Without this chained call, GlobalSettlement's `FreezeCollateralType` will violate LEZ ownership rules and be rejected.

**Required chained call:**
```
GlobalSettlement::FreezeCollateralType
  └─ ChainedCall → LSCEngine::FreezeCollateralType {
       final_collateral_price: final_price,
     }
     accounts: [collateral_type_id, settlement_state_id (PDA auth), ...]
```

### 4.2 Settlement Accounting Bug: vault.balance vs collateral_available Diverge 🔴

**File:** 08-global-settlement.md §3.5, §3.6, §3.7

`ProcessSAFEs` initializes:
```
collateral_redemption.collateral_available = collateral_vault.balance  // V
```

`RedeemCollateral` updates:
```
collateral_redemption.collateral_available -= safe.collateral  // subtract full C
```

But the Transfer only sends `net_collateral = C - D_col` to the SAFE owner:
```
TokenProgram::Transfer { amount_to_transfer: net_collateral }  // only C - D_col
```

After one SAFE redemption:
- `collateral_vault.balance = V - (C - D_col) = V - C + D_col`
- `collateral_redemption.collateral_available = V - C`

These differ by `D_col` (the debt portion in collateral). `collateral_available < vault.balance`.

`SetFinalRedemptionRate` reads `collateral_vault.balance` to compute `collateral_per_lsc`:
```
collateral_per_lsc = remaining_collateral * WAD / total_lsc_outstanding
// where remaining_collateral = collateral_vault.balance  ← too HIGH
```

Then `RedeemLSC` caps redemptions at `collateral_redemption.collateral_available` ← too LOW.

The total LSC that can be redeemed = `collateral_available * WAD / collateral_per_lsc < total_lsc_outstanding`. LSC holders cannot fully redeem — there is leftover collateral in the vault that they can't reach because the cap prevents it.

**Fix:** Either:
- Change `RedeemCollateral` to: `collateral_redemption.collateral_available -= net_collateral` (track what stays for LSC), OR
- Change `SetFinalRedemptionRate` to use `collateral_redemption.collateral_available` instead of `collateral_vault.balance`

### 4.3 PushDebt Authorization Check (Resolved) 🟡

**File:** 07-accounting-engine.md §1.3 PushDebt

The spec has been updated. `AccountingEngineParamsAccount` now stores `liquidation_engine_params_id: [u8; 32]` (set at initialization). The `PushDebt` validation correctly checks:
```
liquidation_engine_params_id.id == accounting_params.liquidation_engine_params_id
```

### 4.4 GlobalSettlementError Has Duplicate Variant 🟡

**File:** 08-global-settlement.md §6

```rust
enum GlobalSettlementError {
    Unauthorized = 6001,
    // ...
    Unauthorized = 6014,  // ← duplicate!
}
```

Two `Unauthorized` variants with different discriminant values is a compilation error in Rust. The second entry is likely meant to be a different error (perhaps `AlreadyLiquidated` is missing a code, or `SafeRedemptionPeriodActive` was meant to replace one).

### 4.5 PIControllerStateAccount Has Spurious `integral_period_size` Field 🟡

**File:** 02-account-schemas.md §3.1

```rust
struct PIControllerStateAccount {
    // ...
    pub per_second_cumulative_leak: u128,
    /// Leak applied to integral each update (dampens windup)
    /// e.g. 999_999_920_000_000_000_000_000_000 (≈ 0.99999992 per second)
    /// Unit: Ray
    pub integral_period_size: u64,

    /// Oracle Relayer params account ID (to read redemption price and market price)
    pub oracle_relayer_params_id: [u8; 32],
```

The doc comment "Leak applied to integral…" belongs to `per_second_cumulative_leak` (the field above), but the `integral_period_size: u64` field has no description of its own — it appears the comment shifted. The `integral_period_size` field is:
1. Not in the `Initialize` instruction parameters
2. Not referenced in `UpdateRate` algorithm
3. Not in the parameter table in spec 12

This field appears to be a leftover from a previous draft. It should be removed, and the doc comments realigned.

### 4.6 SystemParamsAccount `safe_nonce` and `vault_nonce` Are Never Used 🟡

**File:** 02-account-schemas.md §1.1, 03-core-engine.md §OpenSafe

`SystemParamsAccount` has `safe_nonce: u64` and `vault_nonce: u64` fields, presumably for auto-incrementing nonces. But:
- `OpenSafe` uses a **caller-provided** nonce in the instruction: `OpenSafe { nonce: u64 }`, so `safe_nonce` is never incremented
- `AddCollateralType` creates the vault PDA using `collateral_type_id` as seed, not a counter

These fields are dead weight. Either switch to auto-increment (make `system_params_id` writable in `OpenSafe` and use `safe_nonce`) or remove the fields. The current design causes confusion about how SAFE IDs are derived.

### 4.7 ConfiscateSafe liquidated Flag Condition Ambiguous 🟢

**File:** 03-core-engine.md §ConfiscateSafe

```
safe.liquidated = true  // (if collateral_amount == safe.collateral && debt_amount == safe.normalized_debt)
```

The conditional is in a comment but the code shows an unconditional assignment. Since v1 uses full liquidation (per spec 06 §4), the condition is always true. But the code should be explicit, not conditional.

---

## 5. Gap Analysis

### 5.1 No TaxCollector AccrueStabilityFee Instruction (see §1.1) 🔴

### 5.2 No Chained Call pda_seeds Specification (see §2.5) 🔴

Every chained call must specify what PDA seeds the calling program includes. Currently chained calls only show account lists. For example, `LSCEngine::DepositCollateral` chains to `TokenProgram::Transfer` with `collateral_vault_id` as authorized — but how? LSCEngine must include the PDA seed for `collateral_vault_id` in the `ChainedCall.pda_seeds` list. This seed must be specified.

### 5.3 OracleConfigAccount Missing `max_timestamp_jump` Field 🟡

**File:** 02-account-schemas.md §10.1, 04-oracle-system.md §1.4 SubmitPrice

`SubmitPrice` validates `timestamp <= oracle_feed.timestamp + 7200` but `7200` is hardcoded. Spec 12 lists `max_timestamp_jump` as a governance-tunable parameter, but `OracleConfigAccount` does not contain this field:

```rust
struct OracleConfigAccount {
    pub governance_id: [u8; 32],
    pub max_feed_age: u64,
    pub min_feeds_for_median: u8,
    pub feed_count: u16,
    pub _reserved: [u8; 32],
    // MISSING: pub max_timestamp_jump: u64,
}
```

### 5.4 CollateralAuction StartAuction Missing collateral_type_id 🟡

**File:** 06-liquidation-engine.md §3.3 StartAuction

The state transition sets `new_auction.collateral_type_id = collateral_type_id // from liquidation context` but `collateral_type_id` is not in the `StartAuction` instruction parameters or required accounts for `CollateralAuctionHouse::StartAuction`. It is available in `LiquidateSafe`'s context but needs to be passed to `StartAuction` either as an instruction parameter or account.

### 5.5 Auction Holding Accounts Not Initialized 🟡

`SurplusAuction::IncreaseBidSize` transfers LOGOS to `auction_logos_holding_id` but this account is never initialized. `SurplusAuctionHouse::StartAuction` must include a `TokenProgram::InitializeAccount` chained call to create this LOGOS holding account before the auction accepts bids. Spec must specify when and how this is created.

### 5.6 Edge Cases for Empty System Not Covered 🟢

- `SettleDebt` when `debt_queue.entry_count == 0`: undefined behavior
- `UpdateMedian` when all feeds are stale: the spec says "do not update price — keep last valid price" but how is `median_oracle.valid = false` while keeping the old price useful?
- `AuctionSurplus` when `surplus_auction_amount > actual_surplus`: does it revert or auction less?

---

## 6. Security Review

### 6.1 Missing settlement_active Check in PIController::UpdateRate 🟡

**File:** 08-global-settlement.md §3.2

After `TriggerSettlement`, `PIController::UpdateRate` should be blocked. The spec notes this ("PIController::UpdateRate should also check and fail") but this check is not in the `UpdateRate` specification in spec 05. The `PIControllerStateAccount` doesn't store `oracle_relayer_params_id` in a way that would let it check settlement status without reading `system_params_id`. Either add `system_params_id` as a required account for `UpdateRate` or store `settlement_state_id` in `PIControllerStateAccount`.

### 6.2 ConfiscateSafe Authorization: Cross-Program PDA Verification 🟡

**File:** 03-core-engine.md §ConfiscateSafe

The authorization check is:
```
system_params.liquidation_engine_params_id == liquidation_engine_params_id.id
```

This checks that the passed account ID matches the stored one, but does NOT verify that the LiquidationEngine actually authorized the call (i.e., that `liquidation_engine_params_id.is_authorized == true` via pda_seeds). Both checks are needed. The spec mentions both but the implementation must verify that `liquidation_engine_params_id` is truly the LiquidationEngine's PDA being marked as authorized by the chained call chain.

### 6.3 Oracle Timestamp Manipulation via Coordinated Keepers 🟡

**File:** 04-oracle-system.md §3.2

The timestamp model says a single keeper can jump only 2 hours. But if a majority of keepers collude, they can all advance timestamps rapidly, causing programs to think more time has passed than actually has. This artificially accelerates `accumulated_rate` compounding (TaxCollector) and `redemption_price` decay (OracleRelayer). The `max_allowed_lag` in OracleRelayer (1 day) limits this but may not be tight enough. A **real-time clock oracle** (L1 timestamp commitment) would be more robust for high-value deployments.

### 6.4 Debt Queue: Full Queue Prevents Liquidation 🟡

**File:** 07-accounting-engine.md, 02-account-schemas.md §7.3

`SystemDebtQueueAccount.entries` has a fixed capacity of 256. If 256 liquidations have occurred without `SettleDebt` being called, the next `PushDebt` fails with `DebtQueueFull`. This can block new liquidations (since `LiquidateSafe` → `PushDebt` → fails). A griefing attack could pre-fill the queue with 256 dust-debt entries. The spec should either increase the limit, use a dynamic queue, or allow `PushDebt` to aggregate into existing entries.

### 6.5 Operator Clearing on SAFE Transfer May Break Workflows 🟢

**File:** 03-core-engine.md §TransferSafeOwnership

`safe.allowed_operators = [[0; 32]; 4]` on transfer. If a SAFE manager transfers a SAFE to a new owner who doesn't immediately set operators, the new owner must be online to manage the SAFE. This is a minor UX risk, not a security risk.

### 6.6 Collateral Auction TimeToLive (TTL) Security Assumption 🟢

**File:** 06-liquidation-engine.md §3.4

`RestartAuction` with an undefined `discount_increment` (see §1.4) means the discount cannot actually increase upon restart — the function just extends the TTL using the same `auction_params.auction_discount`. This means a persistently un-bid auction keeps the same discount forever, never becoming more attractive. With a slow oracle or declining collateral price, the auction may never settle.

---

## 7. Specific Issues

### 🔴 CRITICAL

---

**C-01: PDA Derivation Format Incompatible with LEZ API**
- **File:** 01-system-architecture.md §3, 02-account-schemas.md (all PDA headers)
- **Problem:** All PDA derivations use variable-length byte concatenation (e.g., `b"safe" || owner[32] || nonce[8]` = 44 bytes) but `PdaSeed` is exactly 32 bytes. The actual LEZ pattern requires hashing seed material (see AMM's `compute_pool_pda_seed` which SHA256-hashes 64 bytes into 32).
- **Fix:** For each PDA, specify the exact bytes to hash: `PdaSeed::new(Sha256::hash(prefix_32_bytes || input_bytes).into())` with explicit prefix selection to avoid cross-PDA collisions.

---

**C-02: Token Burn Account Order Wrong in All Chained Calls**
- **Files:** 03-core-engine.md (RepayDebt), 07-accounting-engine.md (SettleDebt, SurplusAuction::SettleAuction), 08-global-settlement.md (RedeemLSC)
- **Problem:** `TokenProgram::Burn` expects `[definition_account, holder_account (is_authorized)]` (confirmed from `programs/token/src/burn.rs`). All specs show the order reversed.
- **Fix:** Swap account order in all Burn chained calls. Definition first (no auth needed), holder second (auth required).

---

**C-03: Token InitializeAccount Has Wrong Account Order and Non-Existent Parameters**
- **Files:** 03-core-engine.md (AddCollateralType), 09-token-integration.md §4
- **Problem:** `InitializeAccount` takes `[definition_account, account_to_initialize]` with no instruction parameters. The specs show parameters that don't exist and reversed account order.
- **Fix:** Use `[definition_account (read), new_holding_account (writable, new)]` with no instruction parameters.

---

**C-04: LSC Minting Authority Mechanism Fundamentally Wrong**
- **Files:** 01-system-architecture.md §7, 09-token-integration.md §1.3
- **Problem:** `TokenDefinition` has no `minting_authority` field. The only way for LSCEngine to authorize `TokenProgram::Mint` is if the LSC token definition account ID is a PDA of `LSC_ENGINE_ID`, enabling it to be passed as an authorized PDA in `ChainedCall.pda_seeds`. The spec never specifies this and references a non-existent `minting_authority` concept.
- **Fix:** (1) Specify that the LSC token definition account is created as a PDA of `LSC_ENGINE_ID`. (2) Document the `pda_seeds` authorization mechanism. (3) Remove references to `minting_authority`. Note: the LSC system does not mint LOGOS.

---

**C-05: TaxCollector Program Instruction Spec Entirely Missing**
- **Files:** No dedicated spec exists
- **Problem:** The TaxCollector program has no instruction specification. `TaxCollectorParamsAccount` is defined (spec 02) but the instruction enum, `AccrueStabilityFee` accounts/validations/state-transitions, and how it computes the new accumulated_rate are all absent.
- **Fix:** Add a full program specification for TaxCollector (similar in structure to spec 03 for LSCEngine).

---

**C-06: UpdateAccumulatedRate Parameter Ambiguity (Delta vs. New Rate)**
- **File:** 03-core-engine.md §UpdateAccumulatedRate
- **Problem:** The parameter `new_rate_multiplier` is described as "the delta to multiply in" but used as the new absolute accumulated_rate. The surplus formula and the state transition assume different things.
- **Fix:** Rename to `new_accumulated_rate: u128` and clarify it is the absolute new value. Change validation to `new_accumulated_rate >= collateral_type.accumulated_rate`. Update the surplus formula to `surplus_rad = (new_accumulated_rate - old_accumulated_rate) * global_normalized_debt`.

---

**C-07: RepayDebt global_debt Update Has Critical Unit Error**
- **File:** 03-core-engine.md §RepayDebt
- **Problem:** `global_debt.total_debt -= lsc_to_burn` subtracts a Wad value from a Rad field. This would cause catastrophic underflow.
- **Fix:** `global_debt.total_debt -= amount * accumulated_rate` (where `amount` is the normalized debt being removed, result is Rad), or equivalently `global_debt.total_debt -= lsc_to_burn * RAY`.

---

**C-08: GlobalSettlement FreezeCollateralType Missing Required Chained Call**
- **File:** 08-global-settlement.md §3.3
- **Problem:** `GlobalSettlement::FreezeCollateralType` writes to `collateral_type_id` which is owned by `LSC_ENGINE_ID`. GlobalSettlement cannot modify it directly. A chained call to `LSCEngine::FreezeCollateralType` is required.
- **Fix:** Add chained call: `LSCEngine::FreezeCollateralType { final_collateral_price }` with `settlement_state_id (is_authorized via PDA)` and verify `system_params.global_settlement_state_id` matches the caller.

---

**C-09: Settlement Accounting Bug — vault.balance vs collateral_available Diverge**
- **File:** 08-global-settlement.md §3.5, §3.6, §3.7
- **Problem:** `RedeemCollateral` decrements `collateral_redemption.collateral_available` by `safe.collateral` but only transfers `net_collateral = safe.collateral - debt_in_collateral` out of the vault. After SAFE redemptions, `collateral_vault.balance > collateral_redemption.collateral_available`, causing `SetFinalRedemptionRate` (which reads vault.balance) to compute a different rate than what `RedeemLSC` (which caps on collateral_available) will allow. LSC holders cannot fully redeem.
- **Fix:** Change `RedeemCollateral` state transition to: `collateral_redemption.collateral_available -= net_collateral` (not `safe.collateral`). This keeps `collateral_available` in sync with the actual vault balance remaining for LSC holders.

---

### 🟡 IMPORTANT

---

**I-01: PI Controller Arithmetic Overflows i128**
- **File:** 05-pi-controller.md §3.3
- **Problem:** `error_integral * leak_factor` and `(p_r - p_m_ray) * RAY` overflow i128.
- **Fix:** Use i256/u256 throughout PI controller computations. Add explicit note to spec.

---

**I-02: PushDebt Authorization Check (Resolved)**
- **File:** 07-accounting-engine.md §1.3
- **Status:** ✅ Fixed. `AccountingEngineParamsAccount` now includes `liquidation_engine_params_id: [u8; 32]`. The `PushDebt` validation checks against this field.

---

**I-03: GlobalDebtAccount.total_debt Becomes Stale After Rate Updates**
- **File:** 03-core-engine.md §UpdateAccumulatedRate
- **Problem:** When accumulated_rate increases, effective debt grows but `global_debt.total_debt` isn't updated.
- **Fix:** Add to `UpdateAccumulatedRate` state transition: `global_debt.total_debt += (new_rate - old_rate) * collateral_type.global_normalized_debt`.

---

**I-04: safe_nonce and vault_nonce Fields in SystemParamsAccount Are Dead**
- **File:** 02-account-schemas.md §1.1, 03-core-engine.md §OpenSafe
- **Problem:** Fields never incremented/used. Creates confusion.
- **Fix:** Either (a) switch `OpenSafe` to auto-increment nonce from `system_params.safe_nonce` (remove the `nonce` parameter from `OpenSafe` instruction), or (b) remove `safe_nonce` and `vault_nonce` from `SystemParamsAccount`. Option (a) is safer (prevents nonce collisions).

---

**I-05: RestartAuction discount_increment Undefined**
- **File:** 06-liquidation-engine.md §3.6
- **Problem:** `discount_increment` used but never defined.
- **Fix:** Add `discount_increment: u128 (Wad)` to `CollateralAuctionHouseParamsAccount`. Add to `Initialize` instruction parameters. Add to spec 12 parameter table.

---

**I-06: SurplusAuctionHouseParamsAccount Missing**
- **File:** 02-account-schemas.md
- **Problem:** No account schema for the SurplusAuctionHouse system params.
- **Fix:** Add `SurplusAuctionHouseParamsAccount` schema with its PDA (`SURPLUS_AUCTION_ID.derive(b"surplus_auction_params")`) containing `bid_increase`, `bid_duration`, `total_auction_length` fields.

---

**I-07: OracleConfigAccount Missing max_timestamp_jump Field**
- **File:** 02-account-schemas.md §10.1, 04-oracle-system.md §1.4
- **Problem:** `max_timestamp_jump` is a governance parameter (spec 12) but absent from `OracleConfigAccount`.
- **Fix:** Add `pub max_timestamp_jump: u64` to `OracleConfigAccount`. Use in `SubmitPrice` validation. Add `new_max_timestamp_jump: Option<u64>` to `UpdateParams`.

---

**I-08: PIControllerStateAccount integral_period_size Field is Spurious**
- **File:** 02-account-schemas.md §3.1
- **Problem:** Field not initialized, not used, not in parameter table. Doc comments are misaligned.
- **Fix:** Remove `integral_period_size: u64`. Fix doc comment alignment for `per_second_cumulative_leak`.

---

**I-09: GlobalSettlementError Duplicate Unauthorized Variant**
- **File:** 08-global-settlement.md §6
- **Problem:** Two `Unauthorized = 6001` and `Unauthorized = 6014` entries. Compilation error.
- **Fix:** The second entry (6014) appears to be a duplicate. Identify what error was intended (likely related to an edge case in SAFE redemption) and rename accordingly. Renumber to avoid gaps.

---

**I-10: Chained Call pda_seeds Not Specified for Any Instruction**
- **File:** All specs with chained calls
- **Problem:** The mechanism by which programs authorize PDAs in chained calls (`ChainedCall.pda_seeds`) is never explained. Implementers cannot know which PDA seeds to include.
- **Fix:** In the spec introduction (or 09-token-integration.md), explain the `pda_seeds` mechanism. For each chained call, add a `pda_seeds` column specifying which seeds to include.

---

**I-11: CollateralAuction StartAuction Missing collateral_type_id Parameter**
- **File:** 06-liquidation-engine.md §3.3
- **Problem:** State transition references `collateral_type_id` but it's not in instruction parameters or required accounts.
- **Fix:** Add `collateral_type_id: [u8; 32]` to the `StartAuction` instruction parameters (or add the `collateral_type_id` account to required accounts).

---

**I-12: SurplusAuction LOGOS Holding Account Never Initialized**
- **File:** 07-accounting-engine.md §2.3 (IncreaseBidSize)
- **Problem:** `auction_logos_holding_id` (SurplusAuction) is used but never created.
- **Fix:** Add initialization of this holding account in `SurplusAuctionHouse::StartAuction` using a `TokenProgram::InitializeAccount` chained call.

---

**I-13: PIController UpdateRate Must Check settlement_active**
- **File:** 05-pi-controller.md §3.3
- **Problem:** After global settlement, PIController should stop updating rates. No check present.
- **Fix:** Add `system_params_id (read)` to `UpdateRate` required accounts. Add validation: `assert!(!system_params.settlement_active, SettlementActive)`.

---

**I-14: RedeemCollateral Authorization Condition is Misleading**
- **File:** 08-global-settlement.md §3.5
- **Problem:** `assert!(collateral_type.settlement_processed == false || collateral_type.active == false)` is always true (since `FreezeCollateralType` sets `active = false` and `settlement_processed` is only true after `ProcessSAFEs`). The intent seems to be "collateral type has been frozen" but the condition doesn't cleanly express this.
- **Fix:** Replace with `assert!(collateral_type.active == false, CollateralTypeNotFrozen)` or check `settlement_state.active && collateral_type.final_collateral_price > 0`.

---

### 🟢 MINOR

---

**M-01: Settlement Phase Typo**
- **File:** 08-global-settlement.md §1: "Phase 3: SAFERS CAN REDEEM" → "SAFE OWNERS CAN REDEEM"

**M-02: Kp Sign Convention Example Confusing**
- **File:** 05-pi-controller.md §2: Shows negative Kp first, then immediately switches to positive. Remove the negative example entirely.

**M-03: Liquidation Chained Call Count Mismatch**
- **File:** 06-liquidation-engine.md §2.3: States "3 chained calls" but the correct count is 4 (including the nested Token::Transfer from ConfiscateSafe).

**M-04: NewFungibleDefinition Requires Initial Holding Account**
- **File:** 01-system-architecture.md §8: `TOKEN_PROGRAM::NewFungibleDefinition` requires exactly 2 accounts: `[definition_account, holding_account]`. The holding account receives the `total_supply`. For LSC (total_supply=0), this holding can be any dummy account. The spec doesn't mention this second required account.

**M-05: ConfiscateSafe Conditional Assignment is Ambiguous**
- **File:** 03-core-engine.md §ConfiscateSafe: `safe.liquidated = true // (if ...)` — make unconditional since full liquidation is always used.

**M-06: UpdateMedian Fresh Feed Selection Algorithm Ambiguous**
- **File:** 04-oracle-system.md §1.6: "newest_timestamp = max(feed.timestamp for all active feeds)" then filters out feeds older than `newest_timestamp - max_feed_age`. This means the oldest feed could be excluded even if it's only 59 minutes old (if another feed is 2 hours newer). A simpler approach: filter feeds where `feed.timestamp >= current_time - max_feed_age`. Clarify which approach is intended.

**M-07: Debt Ceiling/Floor Formula in Spec 12 Inconsistent with Spec 03**
- **File:** 12-parameters.md §2.4, 2.5: Debt ceiling = `10_000_000 * RAY * WAD`. But `RAY * WAD = 10^45 = Rad`. So ceiling = `10M * 10^45`. This matches Rad units. Spec 03 says `debt_ceiling` is in Rad. Formula is correct but should be stated as `10_000_000 × RAD` for clarity.

**M-08: Stability Fee Combination Not Fully Specified**
- **File:** 12-parameters.md §3: "Recommended: additive (`effective = global + type_specific - RAY`)" — the subtraction of RAY is easily missed. The TaxCollector spec must define exactly how global and per-type stability fees are combined (additive RAY subtraction or multiplicative).

---

## Summary Table

| # | Issue | Severity | File |
|---|---|---|---|
| C-01 | PDA derivation format incompatible with LEZ API | 🔴 CRITICAL | 01, 02 |
| C-02 | Token Burn account order wrong across all specs | 🔴 CRITICAL | 03, 07, 08 |
| C-03 | Token InitializeAccount wrong order + nonexistent params | 🔴 CRITICAL | 03, 09 |
| C-04 | LSC minting authority mechanism fundamentally wrong | 🔴 CRITICAL | 01, 09 |
| C-05 | TaxCollector program specification entirely missing | 🔴 CRITICAL | — |
| C-06 | UpdateAccumulatedRate parameter ambiguity (delta vs new rate) | 🔴 CRITICAL | 03 |
| C-07 | RepayDebt global_debt has critical Wad/Rad unit error | 🔴 CRITICAL | 03 |
| C-08 | GlobalSettlement FreezeCollateralType missing chained call | 🔴 CRITICAL | 08 |
| C-09 | Settlement accounting bug (vault.balance vs collateral_available) | 🔴 CRITICAL | 08 |
| I-01 | PI Controller arithmetic overflows i128 | 🟡 IMPORTANT | 05 |
| I-02 | ~~PushDebt authorization check references wrong field~~ (Resolved — field updated) | ~~🟡 IMPORTANT~~ | 07 |
| I-03 | GlobalDebtAccount.total_debt not updated in UpdateAccumulatedRate | 🟡 IMPORTANT | 03 |
| I-04 | safe_nonce and vault_nonce fields dead/unused | 🟡 IMPORTANT | 02, 03 |
| I-05 | RestartAuction discount_increment undefined | 🟡 IMPORTANT | 06 |
| I-06 | SurplusAuctionHouseParamsAccount missing | 🟡 IMPORTANT | 02, 07 |
| I-07 | OracleConfigAccount missing max_timestamp_jump field | 🟡 IMPORTANT | 02, 04 |
| I-08 | PIControllerStateAccount spurious integral_period_size field | 🟡 IMPORTANT | 02 |
| I-09 | GlobalSettlementError duplicate Unauthorized variant | 🟡 IMPORTANT | 08 |
| I-10 | pda_seeds for chained call authorization never explained | 🟡 IMPORTANT | 03, 06, 07, 08, 09 |
| I-11 | CollateralAuction StartAuction missing collateral_type_id | 🟡 IMPORTANT | 06 |
| I-12 | SurplusAuction LOGOS holding account never initialized | 🟡 IMPORTANT | 07 |
| I-13 | PIController UpdateRate doesn't check settlement_active | 🟡 IMPORTANT | 05 |
| I-14 | RedeemCollateral authorization condition misleading | 🟡 IMPORTANT | 08 |
| M-01 | Settlement phase typo ("SAFERS") | 🟢 MINOR | 08 |
| M-02 | Kp sign convention example confusing | 🟢 MINOR | 05 |
| M-03 | Liquidation chained call count mismatch (3 vs 4) | 🟢 MINOR | 06 |
| M-04 | NewFungibleDefinition requires 2 accounts, spec doesn't show second | 🟢 MINOR | 01 |
| M-05 | ConfiscateSafe conditional assignment ambiguous | 🟢 MINOR | 03 |
| M-06 | UpdateMedian fresh feed selection algorithm ambiguous | 🟢 MINOR | 04 |
| M-07 | Debt ceiling formula notation inconsistent | 🟢 MINOR | 12 |
| M-08 | Stability fee combination not fully specified | 🟢 MINOR | 12 |
