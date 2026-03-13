# LSC Specification Fixes

**Generated from:** specs/AUDIT.md
**Date:** 2026-03-13
**Scope:** All CRITICAL (🔴) and IMPORTANT (🟡) issues from the audit
**Status:** All fixes applied as of 2026-03-13

This document lists the specific changes required, organized by file. Each fix references the corresponding audit finding.

---

## specs/01-system-architecture.md

### Fix 1.1 — PDA Derivation Format ✅ RESOLVED [CRITICAL — Audit §2.1, §7 C4]

**Problem:** All PDA derivation expressions used variable-length byte concatenation but `PdaSeed` is exactly 32 bytes.

**Resolution:** §3 now documents the correct `compute_pda_seed(input_bytes)` pattern (SHA256-hash the variable-length input → 32-byte PdaSeed). All PDA derivation expressions updated throughout. A note clarifies that `ProgramId` is `[u32; 8]`.

### Fix 1.2 — LSC Token Definition Must Be an LSCEngine PDA ✅ RESOLVED [CRITICAL — Audit §2.4, §7 C3]

**Resolution:** §7 (LSC Token Architecture) now documents that `lsc_token_def_id = LSC_ENGINE_ID.derive(compute_pda_seed(b"lsc_token_definition"))`. `minting_authority` field references removed. §3 PDA table includes `lsc_token_def_id`. Account ownership graph updated: CollateralVaultAccount and LscTokenDefinitionAccount are owned by TOKEN_PROGRAM_ID but their IDs are PDAs of LSC_ENGINE_ID.

### Fix 1.3 — Document ChainedCall PDA Seeds Mechanism ✅ RESOLVED [IMPORTANT — Audit §2.5]

**Resolution:** New §3b "ChainedCall Authorization" documents the `pda_seeds` mechanism with examples for minting, vault transfers, and cross-program delegation.

---

## specs/02-account-schemas.md

### Fix 2.1 — CollateralVaultAccount and SystemSurplusAccount Owner ✅ RESOLVED [IMPORTANT — Audit §2.6]

**Resolution:** CollateralVaultAccount owner updated to `TOKEN_PROGRAM_ID` with note that ID is PDA of `LSC_ENGINE_ID`. Same for SystemSurplusAccount (owner: TOKEN_PROGRAM_ID, ID: PDA of ACCOUNTING_ENGINE_ID).

### Fix 2.2 — Remove Spurious PIControllerStateAccount Field ✅ RESOLVED [IMPORTANT — Audit §3.4]

**Resolution:** `integral_period_size: u64` removed from `PIControllerStateAccount`. The `per_second_cumulative_leak` field comment updated to be self-explanatory.

### Fix 2.3 — Add max_timestamp_jump to OracleConfigAccount ✅ RESOLVED [IMPORTANT — Audit §1.6, §3.5]

**Resolution:** `max_timestamp_jump: u64` added to `OracleConfigAccount` with range [300, 86400] and initial value 7200.

### Fix 2.4 — Add SurplusAuctionHouseParamsAccount ✅ RESOLVED [IMPORTANT — Audit §1.5]

**Resolution:** `SurplusAuctionHouseParamsAccount` schema added as §8.0, including `governance_id`, `bid_increase`, `bid_duration`, `total_auction_length`, and `accounting_engine_params_id`.

### Fix 2.5 — Add safe_redemption_period to GlobalSettlementStateAccount ✅ RESOLVED [IMPORTANT — Audit §1.6]

**Resolution:** `safe_redemption_period: u64` added to `GlobalSettlementStateAccount`. `GlobalSettlement::Initialize` now accepts it as a parameter. `SetFinalRedemptionRate` reads `state.safe_redemption_period`.

---

## specs/03-core-engine.md

### Fix 3.1 — RepayDebt: Burn Chained Call Account Order ✅ RESOLVED [CRITICAL — Audit §2.2, §7 C1]

**Resolution:** `RepayDebt` chained call already shows correct order: `[lsc_token_def_id (writable), caller_lsc_holding_id (is_authorized, writable)]`. No change needed to order; `pda_seeds` comment added.

### Fix 3.2 — AddCollateralType: InitializeAccount Chained Call ✅ RESOLVED [CRITICAL — Audit §2.3, §7 C2]

**Resolution:** `AddCollateralType` chained call already shows correct account order: `[logos_token_def_account (read), collateral_vault_id (writable, new)]`. No named parameters.

### Fix 3.3 — RepayDebt: Unit Error in Debt Subtraction ✅ RESOLVED [CRITICAL — Audit §3.2, §7 C6]

**Resolution:** `RepayDebt` state transition correctly uses `normalized_debt_to_repay` (Wad) for state updates and `lsc_to_burn` (Wad) only for the burn chained call.

### Fix 3.4 — UpdateAccumulatedRate: Parameter Naming Ambiguity ✅ RESOLVED [IMPORTANT — Audit §3.1]

**Resolution:** Parameter standardized as `new_accumulated_rate: u128 // Ray — the NEW absolute accumulated_rate`. Validation asserts `new_accumulated_rate >= collateral_type.accumulated_rate`.

### Fix 3.5 — GenerateDebt: Add pda_seeds to Mint ChainedCall ✅ RESOLVED [CRITICAL — Audit §2.4]

**Resolution:** `GenerateDebt` Mint chained call now specifies `pda_seeds: [compute_pda_seed(b"lsc_token_definition")]`. Same applied to `UpdateAccumulatedRate` Mint call and `ConfiscateSafe` Transfer call.

---

## specs/04-oracle-system.md

### Fix 4.1 — SubmitPrice: Read max_timestamp_jump from Config ✅ RESOLVED [IMPORTANT — Audit §1.6]

**Resolution:** `SubmitPrice` now validates `timestamp <= oracle_feed.timestamp + oracle_config.max_timestamp_jump`. `oracle_config_id` added to required accounts. `Initialize` and `UpdateParams` include `max_timestamp_jump`.

---

## specs/05-pi-controller.md

### Fix 5.1 — PIController: Fix i128 Overflow in error_ray Multiplication ✅ RESOLVED [CRITICAL — Audit §3.3, §7 C7]

**Resolution:** `UpdateRate` algorithm now uses `i256` intermediates for all Ray×Ray multiplications (proportional_term, integral_term, error_ray). Comment documents the overflow hazard.

### Fix 5.2 — PIController: Fix i128 Overflow in Integral Decay ✅ RESOLVED [CRITICAL — Audit §3.3]

**Resolution:** Integral decay step uses `i256` intermediates. Integral is clamped after each update to prevent runaway windup.

### Fix 5.3 — PIController: Clarify Kp Sign Convention ✅ RESOLVED [IMPORTANT — Audit §3.6]

**Resolution:** Sign convention locked in: `Kp > 0`, `error_ray = redemption_price - market_price`. Positive error (market below target) → rate increases → negative feedback (stabilizing). Example updated to use consistent positive Kp.

### Fix 5.4 — PIController UpdateRate: Check settlement_active ✅ RESOLVED [IMPORTANT — Audit §4.4]

**Resolution:** `UpdateRate` precondition: `assert!(!system_params.settlement_active, SettlementActive)`. `system_params_id` added to required accounts and `Initialize` parameters.

---

## specs/06-liquidation-engine.md

### Fix 6.1 — LiquidateSafe: Correct Chained Call Count ✅ RESOLVED [IMPORTANT — Audit §3.7]

**Resolution:** Note updated to "4 chained calls" (LiquidateSafe→ConfiscateSafe→Token::Transfer + PushDebt + StartAuction), within LEZ limit of 10.

### Fix 6.2 — RestartAuction: Define discount_increment ✅ RESOLVED [IMPORTANT — Audit §1.4]

**Resolution:** `discount_increment: u128` added to `CollateralAuctionHouseParamsAccount` and `Initialize` instruction. `RestartAuction` uses `params.discount_increment`.

### Fix 6.3 — StartAuction: Add collateral_type_id Parameter ✅ RESOLVED [IMPORTANT — Audit §1.7]

**Resolution:** `collateral_type_id: [u8; 32]` added to `StartAuction` instruction parameters. Passed by LiquidationEngine in `LiquidateSafe` chained call.

---

## specs/07-accounting-engine.md

### Fix 7.1 — UpdateAccumulatedRate: Update global_debt.total_debt ✅ RESOLVED [IMPORTANT — Audit §3.1]

**Resolution:** `UpdateAccumulatedRate` state transition explicitly updates `global_debt.total_debt += (new_accumulated_rate - old_rate) * collateral_type.global_normalized_debt`.

### Fix 7.2 — PushDebt: Correct Authorization Check ✅ RESOLVED

**Status:** Already resolved in prior commits. `AccountingEngineParamsAccount` stores `liquidation_engine_params_id: [u8; 32]`. `PushDebt` validation uses this field.

### Fix 7.3 — Initialize SurplusAuction LOGOS Holding Account ✅ RESOLVED [IMPORTANT]

**Resolution:** `SurplusAuctionHouse::StartAuction` now chains a `TokenProgram::InitializeAccount` call to create the per-auction LOGOS holding account with `pda_seeds: [compute_pda_seed(b"surplus_auction_logos" || nonce[8])]`.

Also fixed: `AccountingEngine::Initialize` chained call for system_surplus uses correct `TokenProgram::InitializeAccount` account order.

---

## specs/08-global-settlement.md

### Fix 8.1 — FreezeCollateralType: Cross-Program Ownership ✅ RESOLVED [CRITICAL — Audit §4.1, §7 C9]

**Resolution:** Already resolved in prior work. `GlobalSettlement::FreezeCollateralType` delegates to `LSCEngine::FreezeCollateralType` via chained call with `pda_seeds: [compute_pda_seed(b"global_settlement_state")]`. LSCEngine verifies the call is from the registered GlobalSettlement.

### Fix 8.2 — Settlement Accounting Bug: collateral_available vs vault.balance ✅ RESOLVED [CRITICAL — Audit §4.2, §7 C8]

**Resolution:** `RedeemCollateral` only decrements `collateral_redemption.collateral_available` by `net_collateral` (the amount actually sent to the safe owner), not the full collateral. The debt-covering portion stays in the vault for LSC holders.

### Fix 8.3 — GlobalSettlementError: Remove Duplicate Unauthorized ✅ RESOLVED [IMPORTANT — Audit §4.3]

**Resolution:** Error codes renamed: `NotGovernance = 6001` (caller is not governance), `NotGlobalSettlement = 6014` (caller is not global settlement program). `CollateralTypeNotFrozen` moved to 6016.

---

## specs/09-token-integration.md

### Fix 9.1 — InitializeAccount Pattern: Correct Account Order ✅ RESOLVED [CRITICAL — Audit §2.3]

**Resolution:** §1.2 now includes explicit code blocks showing correct account ordering for Burn, Mint, and InitializeAccount. §4 pattern already had correct order.

### Fix 9.2 — All Burn Patterns: Correct Account Order ✅ RESOLVED [CRITICAL — Audit §2.2]

**Resolution:** §1.2 "Critical account ordering" section documents `[token_definition_id, holder_account_id]` order for Burn. LSC token setup updated to remove `minting_authority` reference and document PDA-based authorization.

---

## NEW FILE: specs/03b-tax-collector.md

### Fix N.1 — Add TaxCollector Program Specification ✅ RESOLVED [CRITICAL — Audit §1.1, §7 C5]

**Resolution:** Full TaxCollector specification created in `specs/03b-tax-collector.md`:
- `TaxCollectorParamsAccount` schema with PDA derivation
- `Initialize`, `AccrueStabilityFee`, `UpdateParams` instructions
- `AccrueStabilityFee` computes `rpow(effective_rate, elapsed, RAY)` with u256 intermediates
- Chains to `LSCEngine::UpdateAccumulatedRate` with `pda_seeds: [compute_pda_seed(b"tax_collector_params")]`
- Full integration diagram, stability fee rate reference, keeper requirements, error codes

---

## Summary of Required Changes by Severity

| File | Critical Fixes | Important Fixes | Status |
|---|---|---|---|
| 01-system-architecture.md | PDA format (C4), LSC token def as PDA (C3) | ChainedCall pda_seeds doc | ✅ All resolved |
| 02-account-schemas.md | — | Vault owner, spurious integral_period_size, max_timestamp_jump, surplus auction params, safe_redemption_period | ✅ All resolved |
| 03-core-engine.md | Burn order (C1), InitializeAccount (C2), RepayDebt unit error (C6), GenerateDebt pda_seeds (C3) | UpdateAccumulatedRate parameter | ✅ All resolved |
| 04-oracle-system.md | — | max_timestamp_jump from config | ✅ All resolved |
| 05-pi-controller.md | i128 overflow (C7) | Kp sign convention, settlement check | ✅ All resolved |
| 06-liquidation-engine.md | — | Chained call count, discount_increment, StartAuction params | ✅ All resolved |
| 07-accounting-engine.md | — | global_debt.total_debt update, PushDebt auth (✅ prior), SurplusAuction holding account init | ✅ All resolved |
| 08-global-settlement.md | FreezeCollateralType cross-program (C9), settlement accounting (C8) | Duplicate error code | ✅ All resolved |
| 09-token-integration.md | InitializeAccount order (C2), Burn order (C1) | — | ✅ All resolved |
| NEW: 03b-tax-collector.md | TaxCollector specification (C5) | — | ✅ All resolved |
