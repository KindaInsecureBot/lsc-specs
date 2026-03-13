# LSC — Token Program and AMM Program Integration

## Overview

LSC integrates with two existing LEZ programs:
1. **Token Program** — manages all token operations (mint, burn, transfer) for both LSC and LOGOS
2. **AMM Program** — provides the LSC/LOGOS liquidity pool, used for market price discovery

All integration happens through **chained calls** (tail calls), which are sequences of program invocations within a single LEZ transaction. Each LSC program that needs to transfer, mint, or burn tokens issues a chained call to the Token Program.

---

## 1. Token Program Integration

### 1.1 Token Program Instruction Summary

```rust
// Existing Token Program instructions used by LSC:
enum TokenInstruction {
    Transfer { amount_to_transfer: u128 },
    Mint { amount_to_mint: u128 },
    Burn { amount_to_burn: u128 },
    InitializeAccount,
    NewFungibleDefinition { name: String, total_supply: u128 },
}
```

### 1.2 Account Authorization Model

The Token Program's authorization model:
- **Transfer:** The source holding account must have `is_authorized = true`
- **Mint:** The token definition account must have `is_authorized = true`
- **Burn:** The holder account must have `is_authorized = true`

**Critical account ordering (MUST follow exactly):**

```
// TokenProgram::Burn — definition FIRST, holder SECOND:
ChainedCall → TokenProgram::Burn { amount_to_burn: X }
  accounts[0] = token_definition_id  (writable)                   // ← FIRST
  accounts[1] = holder_account_id    (writable, is_authorized)    // ← SECOND
  pda_seeds: [<seed for holder if it's a PDA of the caller>]

// TokenProgram::Mint — definition FIRST (must be is_authorized), holder SECOND:
ChainedCall → TokenProgram::Mint { amount_to_mint: X }
  accounts[0] = token_definition_id  (writable, is_authorized)    // ← FIRST (minting auth)
  accounts[1] = holder_account_id    (writable)                   // ← SECOND
  pda_seeds: [<seed that derives token_definition_id>]

// TokenProgram::InitializeAccount — definition FIRST, new account SECOND:
ChainedCall → TokenProgram::InitializeAccount  (no instruction parameters)
  accounts[0] = token_definition_id  (read)                       // ← FIRST
  accounts[1] = new_holding_account  (writable, new)              // ← SECOND
  pda_seeds: [<seed for new_holding_account if it's a PDA of the caller>]
```

LSC programs authorize their PDAs via `ChainedCall.pda_seeds`. This is the LEZ equivalent of ERC-20's `transferFrom` with approval.

### 1.3 LSC Token Setup

**At deployment, the LSC token definition is a PDA of LSCEngine:**

```
// lsc_token_def_id = LSC_ENGINE_ID.derive(compute_pda_seed(b"lsc_token_definition"))
// LSCEngine initializes this account via TokenProgram::NewFungibleDefinition
// (or TokenProgram::InitializeAccount depending on Token Program API)
// The account ID is a PDA of LSC_ENGINE_ID — this grants minting authority.
```

**Authorization:** LSCEngine authorizes minting by including `compute_pda_seed(b"lsc_token_definition")` in `ChainedCall.pda_seeds`. The LEZ runtime derives `lsc_token_def_id = LSC_ENGINE_ID.derive(that_seed)` and marks it `is_authorized = true` in the Token Program callee. There is no separate `minting_authority` field.

### 1.4 Token Operations by Program

#### LSCEngine → Token Program

| Operation | Trigger | From | To | Amount |
|---|---|---|---|---|
| Transfer (deposit) | `DepositCollateral` | User LOGOS holding | Collateral vault PDA | `amount` |
| Transfer (withdraw) | `WithdrawCollateral` | Collateral vault PDA | User LOGOS holding | `amount` |
| Mint | `GenerateDebt` | (system_params_id auth) | User LSC holding | `lsc_amount` |
| Burn | `RepayDebt` | User LSC holding | (lsc_token_def_id) | `lsc_to_burn` |

**Chained call pattern for Mint:**
```
ChainedCall {
    program_id: TOKEN_PROGRAM_ID,
    instruction: TokenInstruction::Mint { amount_to_mint: lsc_amount },
    accounts: [
        AccountWithMetadata { id: lsc_token_def_id, is_authorized: true, is_writable: true },  // ← FIRST
        AccountWithMetadata { id: user_lsc_holding_id, is_authorized: false, is_writable: true }, // ← SECOND
    ],
    pda_seeds: [compute_pda_seed(b"lsc_token_definition")],  // ← authorizes lsc_token_def_id
}
```

The `lsc_token_def_id` gets `is_authorized: true` at runtime because LSCEngine includes `compute_pda_seed(b"lsc_token_definition")` in `pda_seeds`. The LEZ runtime derives `lsc_token_def_id = LSC_ENGINE_ID.derive(that_seed)` and marks it authorized.

**Chained call pattern for Transfer (from vault):**
```
ChainedCall {
    program_id: TOKEN_PROGRAM_ID,
    instruction: TokenInstruction::Transfer { amount_to_transfer: collateral_amount },
    accounts: [
        AccountWithMetadata { id: collateral_vault_id, is_authorized: true, is_writable: true },
        // collateral_vault_id is a PDA of LSCEngine; LSCEngine can authorize it
        AccountWithMetadata { id: user_logos_holding_id, is_authorized: false, is_writable: true },
    ],
}
```

#### TaxCollector → LSCEngine → Token Program

The TaxCollector calls `LSCEngine::UpdateAccumulatedRate`, which internally chains to `TokenProgram::Mint` to create surplus LSC:
```
TaxCollector::AccrueStabilityFee
  └─ chained call ──► LSCEngine::UpdateAccumulatedRate
                           └─ chained call ──► TokenProgram::Mint(surplus_lsc)
```

This is a **two-deep chained call chain.** LEZ supports up to 10 chained calls per execution, and each can themselves chain further (within the total limit of 10 per transaction). The implementation should confirm the exact nesting model with the LEZ team.

#### CollateralAuctionHouse → Token Program

| Operation | Trigger | From | To | Amount |
|---|---|---|---|---|
| Transfer (LOGOS out) | `BuyCollateral` | Auction vault PDA | Buyer LOGOS holding | `collateral_amount` |
| Transfer (LSC in) | `BuyCollateral` | Buyer LSC holding | Accounting engine surplus | `lsc_cost_wad` |

#### AccountingEngine → Token Program

| Operation | Trigger | From | To | Amount |
|---|---|---|---|---|
| Burn (LSC) | `SettleDebt` | System surplus PDA | (lsc_token_def_id) | `lsc_to_burn` |
| Burn (LOGOS) | `SurplusAuction::SettleAuction` | Auction LOGOS holding | (logos_token_def_id) | `bid_amount` |
| Transfer | Various auctions | Holding PDAs | Participants | Various |

---

## 2. AMM Program Integration

### 2.1 Purpose

The AMM Program's LOGOS/LSC pool provides the **market price feed** for LSC. This price is submitted by oracle keepers to the OracleProgram.

LSC itself does not directly call the AMM Program during normal operations. The AMM is accessed by:
1. **Oracle keepers** — off-chain, they read AMM pool state and submit prices
2. **Governance** — may interact with the pool for liquidity management (out of scope for v1)

### 2.2 AMM Pool Setup

At deployment, after LSC is created, an LOGOS/LSC liquidity pool must be created:

```
AMM_PROGRAM::NewDefinition {
    token_a_amount: INITIAL_LOGOS_AMOUNT,    // e.g., 100,000 LOGOS
    token_b_amount: INITIAL_LSC_AMOUNT,       // e.g., 314,000 LSC at $3.14 per LSC
    amm_program_id: AMM_PROGRAM_ID,
}
// Creates: PoolDefinition, VaultA (LOGOS), VaultB (LSC), LP token definition
// Pool represents: LOGOS/LSC pair with constant-product formula
```

The initial ratio determines the initial market price: `LOGOS_amount / LSC_amount` (in terms of token units).

### 2.3 Price Derivation

The LSC/USD price is derived by oracle keepers as follows:

**Step 1:** Read AMM pool state (off-chain, or via a read-only invocation):
```
pool = read(PoolDefinitionAccount)
logos_reserve = pool.reserve_a   // Wad (LOGOS in pool)
lsc_reserve = pool.reserve_b     // Wad (LSC in pool)
spot_price_logos_per_lsc = logos_reserve / lsc_reserve  // LOGOS per LSC
```

**Step 2:** Get LOGOS/USD price from an external source (Chainlink, Pyth, CEX feed):
```
logos_usd_price = external_price  // USD per LOGOS
```

**Step 3:** Compute LSC/USD:
```
lsc_usd_price = spot_price_logos_per_lsc * logos_usd_price
              = (logos_reserve / lsc_reserve) * logos_usd_price
```

**Step 4:** Submit to `OracleProgram::SubmitPrice` for the LSC_USD feed.

### 2.4 AMM Instruction Reference (for integration context)

The following AMM instructions are relevant to understand how the pool works:

```rust
// Creating the LSC/LOGOS pool (done at deployment)
AMM::NewDefinition {
    token_a_amount: u128,  // LOGOS deposited as initial liquidity
    token_b_amount: u128,  // LSC deposited as initial liquidity
    amm_program_id: [u8; 32],
}
// Required accounts:
// - pool_def (PDA of AMM)
// - vault_a, vault_b (PDAs of AMM, hold tokens)
// - lp_token_def (PDA of AMM, LP tokens)
// - user's LOGOS holding (is_authorized)
// - user's LSC holding (is_authorized)
// AMM internally calls TokenProgram::Transfer and TokenProgram::NewFungibleDefinition

// Swapping (used by arbitrageurs to maintain LSC price)
AMM::Swap {
    swap_amount_in: u128,
    min_amount_out: u128,
    token_definition_id_in: [u8; 32],
}
// This is called by market participants, not by LSC programs
```

### 2.5 Reading AMM Pool State

LSC programs do not read AMM state directly. Instead, keepers extract information off-chain and submit it via the oracle system. This is deliberate:
- Avoids tight coupling between LSC and AMM program versions
- Allows any price source to be added without changing LSC programs
- Oracle keepers can implement TWAP, outlier rejection, and other filters

**How oracle keepers should compute AMM-based prices:**

```
// TWAP (Time-Weighted Average Price) computation:
// keepers track (reserve_logos * reserve_lsc * timestamp) over multiple snapshots
// and compute the geometric mean price over a time window (e.g., 1 hour)

// Minimum viable approach (spot price):
// 1. Read PoolDefinition.reserve_a and reserve_b at submission time
// 2. Compute spot price
// 3. Apply sanity checks (price within X% of previous submission)
// 4. Submit if valid

// Recommended: collect 3+ independent keeper submissions and take the median
// This is exactly what MedianOracleAccount.UpdateMedian does
```

---

## 3. Chained Call Patterns

### 3.1 Deposit Collateral Flow

```
User transaction:
  LSCEngine::DepositCollateral { amount: 100_LOGOS }
    Account list:
      safe_id (writable)
      collateral_type_id (read)
      collateral_vault_id (writable)
      user_logos_holding_id (writable, is_authorized=true)  ← user signs this
      caller_account (is_authorized=true)

  LSCEngine logic:
    1. Validate safe, collateral type, authorization
    2. safe.collateral += 100_LOGOS
    3. Emit chained call:

  ChainedCall → TokenProgram::Transfer { amount_to_transfer: 100 }
    Account list:
      user_logos_holding_id (writable, is_authorized=true)  ← propagated from parent
      collateral_vault_id (writable)
```

### 3.2 Generate Debt Flow

```
User transaction:
  LSCEngine::GenerateDebt { amount: 50_NORMALIZED_DEBT }
    Account list:
      safe_id (writable)
      collateral_type_id (writable)
      global_debt_id (writable)
      oracle_relayer_params_id (read)
      collateral_oracle_id (read)
      lsc_token_def_id (read)
      user_lsc_holding_id (writable)
      system_params_id (read)
      caller_account (is_authorized=true)

  LSCEngine logic:
    1. Compute lsc_to_mint = 50 * accumulated_rate / RAY
    2. Update safe.normalized_debt, global_debt
    3. Emit chained call:

  ChainedCall → TokenProgram::Mint { amount_to_mint: lsc_to_mint }
    Account list:
      lsc_token_def_id (writable, is_authorized=true)  ← LSCEngine authorizes via PDA
      user_lsc_holding_id (writable)
```

### 3.3 Liquidation Flow (5 chained calls)

```
Liquidator transaction:
  LiquidationEngine::LiquidateSafe
    ├─ ChainedCall #1 → LSCEngine::ConfiscateSafe
    │     └─ ChainedCall #2 → TokenProgram::Transfer (collateral → auction vault)
    ├─ ChainedCall #3 → AccountingEngine::PushDebt
    └─ ChainedCall #4 → CollateralAuctionHouse::StartAuction

Total chained calls: 4 (within LEZ limit of 10)
```

---

## 4. Account Initialization with Token Program

When LSC programs create vault accounts, they initialize them as Token Program holding accounts:

```
// Pattern: Initialize a PDA as a Token Program holding account
// InitializeAccount takes NO instruction parameters — account positions determine behavior.
// Account order: [definition_account (read), account_to_initialize (writable, new)]
// The initialized account's holder is set to its own ID automatically.
// No is_authorized flag is required — anyone can initialize a holding account.
ChainedCall → TokenProgram::InitializeAccount
  Account list:
    token_def_id (read)                        // #0: token definition
    new_holding_account_id (PDA, writable)     // #1: account to initialize
    // Token Program sets: holding_account.holder_id = new_holding_account_id
    // Token Program sets: holding_account.definition_id = token_def_id
    // Token Program sets: holding_account.balance = 0
```

This is called by LSCEngine when adding a collateral type (for the vault) and by CollateralAuctionHouse in Initialize (for the auction vault).

---

## 5. Balance Conservation

LEZ enforces balance conservation: the sum of `account.balance` across all accounts in a transaction's pre-state and post-state must be equal. For LSC:

- **LOGOS balance** is managed entirely by the Token Program. LSC programs do not touch `account.balance` directly for LOGOS; all transfers go through Token Program chained calls.
- **LSC balance** is similarly managed by Token Program.
- **Native LEZ balance** (the `balance: u128` field in LSC program accounts) is always 0 for LSC PDAs. No native balance is used.

When an account is closed (e.g., `CloseSafe`), its balance must be transferred to another account (the owner's account). Since LSC PDA accounts have balance = 0, this is a no-op for balance purposes.

---

## 6. Token Program Error Handling

If a chained call to the Token Program fails (e.g., insufficient balance, unauthorized), the entire transaction is rolled back. LSC programs must not depend on partial success of chained calls.

Common Token Program errors that LSC must handle gracefully:
- `InsufficientBalance` — user doesn't have enough tokens
- `Unauthorized` — authorization propagation failed
- `InvalidAccountType` — wrong account type passed

In LEZ, chained call failures propagate upward and abort the whole transaction.
