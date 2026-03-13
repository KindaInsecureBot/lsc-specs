# LSC — Accounting Engine

## Overview

The Accounting Engine is the protocol's treasury. It manages the flow between:
- **System surplus** — LSC accumulated from stability fees (minted by TaxCollector via LSCEngine::UpdateAccumulatedRate)
- **Bad debt** — uncovered LSC debt from insolvent SAFEs pushed by LiquidationEngine

When surplus exceeds a threshold, surplus auctions are triggered: users bid LOGOS to receive protocol LSC; the LOGOS is burned (deflationary).

When bad debt accumulates, it is first offset against available surplus via `SettleDebt`. Any bad debt that cannot be offset remains as a **protocol liability** in the debt queue. It is socialized across LSC holders through the redemption price mechanism (the PI controller, adjusting redemption rates, implicitly accounts for the system's reduced backing). Governance can inject capital directly using `RecapitalizeWithLSC` (deposit LSC) or `RecapitalizeWithLOGOS` (deposit LOGOS, swapped for LSC on the AMM).

The system does **not** mint new LOGOS tokens to cover bad debt. There is no debt auction.

**Program ID aliases:** `ACCOUNTING_ENGINE_ID`, `SURPLUS_AUCTION_ID`

---

## 1. AccountingEngine Program

### 1.1 Instruction Enum

```rust
enum AccountingEngineInstruction {
    /// One-time initialization
    Initialize {
        surplus_auction_params_id: [u8; 32],
        lsc_engine_params_id: [u8; 32],
        liquidation_engine_params_id: [u8; 32],
        lsc_token_def_id: [u8; 32],
        surplus_auction_amount: u128,           // Rad (surplus sold per auction)
        surplus_buffer: u128,                   // Rad (buffer kept before auctioning)
        surplus_auction_delay: u64,             // seconds between surplus auctions
    },

    /// Receive bad debt from LiquidationEngine (privileged)
    PushDebt {
        debt_amount: u128,   // Rad
    },

    /// Cancel bad debt using available surplus (netting)
    SettleDebt {
        amount: u128,   // Rad — amount of debt to cancel against surplus
    },

    /// Trigger a surplus auction (permissionless, if conditions met)
    AuctionSurplus,

    /// Update parameters (governance)
    UpdateParams {
        new_surplus_auction_amount: Option<u128>,
        new_surplus_buffer: Option<u128>,
        new_surplus_auction_delay: Option<u64>,
    },

    /// Governance injects LSC directly to cancel bad debt (no LOGOS minting)
    RecapitalizeWithLSC {
        amount: u128,   // Wad (LSC to inject)
    },

    /// Governance injects LOGOS which is sold for LSC via AMM to cancel bad debt
    RecapitalizeWithLOGOS {
        logos_amount: u128,     // Wad (LOGOS to sell)
        min_lsc_out: u128,      // Wad (minimum LSC to receive; slippage protection)
    },
}
```

---

### 1.2 `Initialize`

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` (PDA) | Yes | PDA (new) | — |
| 1 | `system_surplus_id` (PDA) | Yes | PDA (new) | LSC holding account for surplus |
| 2 | `debt_queue_id` (PDA) | Yes | PDA (new) | Debt queue account |
| 3 | `governance_account` | No | Yes | — |
| 4 | `lsc_token_def_id` | No | — | LSC token definition |

**State Transition:**
```
accounting_params.account_type = 60
accounting_params.[all fields] = provided values
accounting_params.last_surplus_auction_time = 0
accounting_params.surplus_auction_nonce = 0

debt_queue.account_type = 62
debt_queue.queued_debt = 0
debt_queue.entry_count = 0
```

**Chained Calls:**
```
// Initialize the system_surplus as a Token Program LSC holding account
// Account order: [definition_account (read), account_to_initialize (writable, new)]
// InitializeAccount takes no instruction parameters — positions determine behavior.
TokenProgram::InitializeAccount
  accounts: [lsc_token_def_id (read), system_surplus_id (writable, new)]
  pda_seeds: [compute_pda_seed(b"system_surplus")]
  // After this call: system_surplus_id.program_owner = TOKEN_PROGRAM_ID
```

---

### 1.3 `PushDebt`

**Description:** Called by LiquidationEngine (via chained call) when a SAFE is liquidated and there is a debt deficit. The pushed debt represents LSC that was minted but the corresponding collateral was insufficient to cover it.

**Authorization:** Only the registered LiquidationEngine can call this (PDA authorization).

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | No | — | For params reference |
| 1 | `debt_queue_id` | Yes | — | Debt queue to update |
| 2 | `liquidation_engine_params_id` | No | Yes (PDA) | Must be registered |
| 3 | `oracle_id` | No | — | For current timestamp |

**Validations:**
- `liquidation_engine_params_id.is_authorized == true`
- `liquidation_engine_params_id.id == accounting_params.liquidation_engine_params_id`
  (the AccountingEngineParamsAccount stores the registered liquidation engine ID)

**State Transition:**
```
// Add entry to debt queue
let entry = DebtQueueEntry {
    debt: debt_amount,
    pushed_at: current_timestamp,
};
debt_queue.entries[debt_queue.entry_count] = entry;
debt_queue.entry_count += 1;
debt_queue.queued_debt += debt_amount;
```

**Errors:**
- `Unauthorized`
- `DebtQueueFull` — more than 256 entries (settle first)

---

### 1.4 `SettleDebt`

**Description:** Cancels bad debt against available surplus. The surplus LSC is burned; the corresponding debt is removed from the queue. This is the primary mechanism for keeping the books balanced.

**Anyone can call this.** It improves protocol health.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | No | — | — |
| 1 | `debt_queue_id` | Yes | — | Debt queue (reduce debt) |
| 2 | `system_surplus_id` | Yes | PDA (AccountingEngine) | LSC surplus holding |
| 3 | `lsc_token_def_id` | No | — | For burning |

**Validations:**
```
assert!(amount <= debt_queue.queued_debt, InsufficientDebt);
assert!(amount <= system_surplus_balance_rad, InsufficientSurplus);
// where system_surplus_balance_rad = system_surplus_holding.balance * RAY
```

**State Transition:**
```
// Reduce debt queue
debt_queue.queued_debt -= amount

// Burn surplus LSC
lsc_to_burn_wad = amount / RAY  // Rad → Wad

// Remove entries from front of queue (FIFO)
while amount > 0 && entry_count > 0:
    if debt_queue.entries[0].debt <= amount:
        amount -= debt_queue.entries[0].debt
        shift entries left
        entry_count -= 1
    else:
        debt_queue.entries[0].debt -= amount
        amount = 0
```

**Chained Calls:**
```
TokenProgram::Burn {
    amount_to_burn: lsc_to_burn_wad,
    // accounts: [lsc_token_def_id, system_surplus_id (PDA auth)]
    // Note: definition account first, then holder (authorized)
}
```

**Errors:**
- `InsufficientDebt` — amount > queued_debt
- `InsufficientSurplus` — not enough LSC surplus

---

### 1.5 `AuctionSurplus`

**Description:** Initiates a surplus auction when the system has more LSC than needed. The surplus is sold to the market for LOGOS, which is burned.

**Trigger conditions:**
```
surplus_balance_rad = system_surplus_lsc_balance * RAY
queued_debt = debt_queue.queued_debt

net_surplus_rad = surplus_balance_rad - queued_debt

net_surplus_rad > accounting_params.surplus_buffer + accounting_params.surplus_auction_amount
AND
current_timestamp >= accounting_params.last_surplus_auction_time + accounting_params.surplus_auction_delay
```

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | Yes | — | Update nonce + timestamp |
| 1 | `debt_queue_id` | No | — | Read queued_debt for net surplus calculation |
| 2 | `system_surplus_id` | Yes | PDA | Transfer LSC to auction |
| 3 | `new_surplus_auction_id` (PDA) | Yes | PDA (new) | New auction account |
| 4 | `surplus_auction_params_id` | No | — | Surplus auction program |
| 5 | `lsc_market_oracle_id` | No | — | For timestamp |

**Computation:**
```
// LSC to auction
lsc_to_auction_rad = accounting_params.surplus_auction_amount
lsc_to_auction_wad = lsc_to_auction_rad / RAY
```

**State Transition:**
```
accounting_params.last_surplus_auction_time = current_timestamp
accounting_params.surplus_auction_nonce += 1
```

**Chained Calls:**
```
// 1. Transfer LSC from system_surplus to new auction account
TokenProgram::Transfer {
    amount_to_transfer: lsc_to_auction_wad,
    // accounts: [system_surplus_id (PDA auth), new_surplus_auction_lsc_holding_id]
}

// 2. Create the surplus auction
SurplusAuctionHouse::StartAuction {
    amount_to_sell: lsc_to_auction_rad,
    // accounts: [surplus_auction_params_id, new_surplus_auction_id (auth)]
}
```

**Errors:**
- `SurplusNotSufficient` — net surplus below threshold
- `TooSoon` — delay not elapsed

---

### 1.6 `RecapitalizeWithLSC`

**Description:** Governance injects LSC directly into the system to cancel queued bad debt. The caller provides LSC tokens from their own holding account. No LOGOS is minted. If the deposited amount exceeds the queued bad debt, the remainder is added to the system surplus.

**Authorization:** Governance only.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | No | — | For governance check |
| 1 | `debt_queue_id` | Yes | — | Debt queue to reduce |
| 2 | `system_surplus_id` | Yes | PDA (AccountingEngine) | LSC surplus holding |
| 3 | `governance_lsc_holding_id` | Yes | Yes | Caller's LSC (source of injection) |
| 4 | `governance_account` | No | Yes | Must match accounting_params.governance_id |
| 5 | `lsc_engine_params_id` | No | — | For governance_id lookup |
| 6 | `lsc_token_def_id` | No | — | LSC token definition |

**Validations:**
```
assert!(governance_account.id == lsc_engine_params.governance_id, Unauthorized);
assert!(governance_account.is_authorized == true, Unauthorized);
assert!(amount > 0, ZeroAmount);
assert!(governance_lsc_holding.balance >= amount, InsufficientLSC);
```

**Computation:**
```
amount_rad = amount * RAY  // Wad → Rad

// How much cancels debt vs goes to surplus
let debt_to_cancel = min(amount_rad, debt_queue.queued_debt);  // Rad
let surplus_addition = amount_rad - debt_to_cancel;            // Rad
```

**State Transition:**
```
// Cancel as much queued debt as possible (FIFO)
if debt_to_cancel > 0:
    debt_queue.queued_debt -= debt_to_cancel
    // Remove entries from front of queue
    remaining = debt_to_cancel
    while remaining > 0 && entry_count > 0:
        if debt_queue.entries[0].debt <= remaining:
            remaining -= debt_queue.entries[0].debt
            shift entries left
            entry_count -= 1
        else:
            debt_queue.entries[0].debt -= remaining
            remaining = 0

// Any excess goes to system surplus (transferred in chained call below)
```

**Chained Calls:**
```
// Transfer LSC from governance into system_surplus
TokenProgram::Transfer {
    amount_to_transfer: amount,  // Wad
    // accounts: [governance_lsc_holding_id (auth), system_surplus_id]
}
```

**Errors:**
- `Unauthorized` — caller is not governance
- `ZeroAmount` — amount is 0
- `InsufficientLSC` — caller does not have enough LSC

---

### 1.7 `RecapitalizeWithLOGOS`

**Description:** Governance injects LOGOS which is immediately sold for LSC on the AMM. The acquired LSC is used to cancel queued bad debt. No LOGOS is minted by this instruction — the caller provides existing LOGOS tokens. If the acquired LSC exceeds the queued bad debt, the remainder goes to the system surplus.

**Authorization:** Governance only.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | No | — | For governance check |
| 1 | `debt_queue_id` | Yes | — | Debt queue to reduce |
| 2 | `system_surplus_id` | Yes | PDA (AccountingEngine) | Receives acquired LSC |
| 3 | `governance_logos_holding_id` | Yes | Yes | Caller's LOGOS (sold) |
| 4 | `governance_account` | No | Yes | Must match accounting_params.governance_id |
| 5 | `lsc_engine_params_id` | No | — | For governance_id lookup |
| 6 | `amm_pool_def_id` | No | — | LSC/LOGOS AMM pool definition |
| 7 | `amm_vault_logos_id` | Yes | — | AMM's LOGOS vault (receives LOGOS) |
| 8 | `amm_vault_lsc_id` | Yes | — | AMM's LSC vault (sends LSC) |
| 9 | `lsc_token_def_id` | No | — | LSC token definition |
| 10 | `logos_token_def_id` | No | — | LOGOS token definition |

**Validations:**
```
assert!(governance_account.id == lsc_engine_params.governance_id, Unauthorized);
assert!(governance_account.is_authorized == true, Unauthorized);
assert!(logos_amount > 0, ZeroAmount);
assert!(min_lsc_out > 0, ZeroAmount);
assert!(governance_logos_holding.balance >= logos_amount, InsufficientLOGOS);
```

**Computation:**

The LSC acquired from the AMM swap (`lsc_acquired`) is determined by the AMM's constant-product formula and is validated against `min_lsc_out` (slippage protection) by the AMM program itself. After the swap:

```
lsc_acquired_rad = lsc_acquired * RAY  // Wad → Rad

let debt_to_cancel = min(lsc_acquired_rad, debt_queue.queued_debt);
let surplus_addition = lsc_acquired_rad - debt_to_cancel;
```

**State Transition:**
```
// Cancel queued debt with acquired LSC (same FIFO logic as RecapitalizeWithLSC)
if debt_to_cancel > 0:
    debt_queue.queued_debt -= debt_to_cancel
    // Remove entries FIFO (same algorithm as SettleDebt / RecapitalizeWithLSC)

// Surplus addition: lsc goes to system_surplus via the AMM swap output
// (the AMM sends lsc_acquired directly to system_surplus_id as the swap recipient)
```

**Chained Calls:**

This instruction uses two chained calls (total: 2, within LEZ limit of 10):

```
// 1. Swap LOGOS for LSC on AMM — output goes directly to system_surplus_id
AMM::Swap {
    swap_amount_in: logos_amount,        // Wad (LOGOS in)
    min_amount_out: min_lsc_out,         // Wad (LSC out, slippage guard)
    token_definition_id_in: logos_token_def_id,
    // accounts: [
    //   governance_logos_holding_id (auth — caller's LOGOS),
    //   amm_vault_logos_id (writable),
    //   amm_vault_lsc_id (writable),
    //   system_surplus_id (writable — receives LSC output),
    //   amm_pool_def_id,
    // ]
}

// Note: The AMM swap sends LSC directly to system_surplus_id.
// The debt queue reduction happens in this instruction's state transition
// based on the lsc_acquired amount (computed from AMM pool state pre-swap).
// Because LEZ executes atomically, the post-swap surplus balance reflects
// lsc_acquired correctly.
```

**Note on chained call count:** `RecapitalizeWithLOGOS` issues 1 chained call (`AMM::Swap`). The AMM internally chains to `TokenProgram::Transfer` (2 calls), for a total depth of 3 nested calls — well within the LEZ limit of 10.

**Slippage:** The `min_lsc_out` parameter provides slippage protection. If the AMM cannot provide at least `min_lsc_out` LSC for `logos_amount` LOGOS (due to price movement or pool imbalance), the AMM will revert the entire transaction. Governance must set a reasonable `min_lsc_out` based on current pool prices.

**Errors:**
- `Unauthorized` — caller is not governance
- `ZeroAmount` — logos_amount or min_lsc_out is 0
- `InsufficientLOGOS` — caller does not have enough LOGOS
- AMM errors propagate: `SlippageExceeded` if `lsc_out < min_lsc_out`

---

## 2. SurplusAuctionHouse Program

**Program ID alias:** `SURPLUS_AUCTION_ID`

A **reverse auction**: protocol sells a fixed amount of LSC; buyers compete by offering increasing amounts of LOGOS. Highest bidder wins. LOGOS received is burned.

### 2.1 Instruction Enum

```rust
enum SurplusAuctionInstruction {
    /// Initialize (one-time)
    Initialize {
        accounting_engine_params_id: [u8; 32],
        logos_token_def_id: [u8; 32],
        bid_increase: u128,       // Wad (min bid increment, e.g. 5%)
        bid_duration: u64,        // seconds after last bid before auction ends
        total_auction_length: u64, // hard deadline from auction start
    },

    /// Start a new surplus auction (called by AccountingEngine — privileged)
    StartAuction {
        amount_to_sell: u128,   // Rad (LSC being auctioned)
    },

    /// Place a bid (pay LOGOS, get LSC)
    IncreaseBidSize {
        auction_id: u64,
        bid_amount: u128,   // Wad (LOGOS offered)
    },

    /// Settle an auction after it has ended
    SettleAuction {
        auction_id: u64,
    },

    /// Restart an expired auction (if no bids)
    RestartAuction {
        auction_id: u64,
    },
}
```

---

### 2.2 `StartAuction`

**Authorization:** `accounting_engine_params_id` must be `is_authorized`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `surplus_auction_params_id` | No | — | — |
| 1 | `new_auction_id` (PDA) | Yes | PDA (new) | — |
| 2 | `accounting_engine_params_id` | No | Yes (PDA) | Authorization |
| 3 | `lsc_market_oracle_id` | No | — | Timestamp |
| 4 | `logos_token_def_id` | No | — | For LOGOS holding initialization |
| 5 | `auction_logos_holding_id` (PDA) | Yes | PDA (new) | Receives LOGOS bids during auction |

**PDA derivation:**
```
new_auction_id = SURPLUS_AUCTION_ID.derive(b"surplus_auction" || auction_nonce.to_le_bytes())
```

**State Transition:**
```
new_auction.account_type = 70
new_auction.auction_id = nonce
new_auction.amount_to_sell = amount_to_sell
new_auction.bid_amount = 0         // no bid yet
new_auction.high_bidder = [0; 32]  // no bidder yet
new_auction.bid_time = current_timestamp
new_auction.auction_deadline = current_timestamp + surplus_auction_params.total_auction_length
new_auction.bid_duration = surplus_auction_params.bid_duration
new_auction.bid_increase = surplus_auction_params.bid_increase
new_auction.settled = false
```

**Chained Calls:**
```
// Initialize per-auction LOGOS holding account to receive bids
TokenProgram::InitializeAccount
  accounts: [logos_token_def_id (read), auction_logos_holding_id (writable, new)]
  pda_seeds: [compute_pda_seed(b"surplus_auction_logos" || nonce[8])]
  // After this call: auction_logos_holding_id.program_owner = TOKEN_PROGRAM_ID
  // This account receives LOGOS bids and is burned at auction settlement
```

---

### 2.3 `IncreaseBidSize`

**Description:** A bidder offers LOGOS in exchange for the fixed LSC amount. Each bid must be higher than the previous by at least `bid_increase`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `auction_id` | Yes | — | The auction |
| 1 | `bidder_account` | No | Yes | Bidder |
| 2 | `bidder_logos_holding_id` | Yes | Yes | Bidder's LOGOS (transfers from here) |
| 3 | `previous_bidder_logos_holding_id` | Yes | — | Previous bidder's LOGOS (refunded) |
| 4 | `auction_logos_holding_id` | Yes | PDA | Temp LOGOS holding during auction |
| 5 | `lsc_market_oracle_id` | No | — | Timestamp |

**Validations:**
```
assert!(!auction.settled, AuctionSettled);
assert!(current_timestamp < auction.auction_deadline, AuctionExpired);

// Must beat previous bid by at least bid_increase
let min_bid = if auction.bid_amount == 0 {
    1  // Any non-zero bid is acceptable as first bid
} else {
    auction.bid_amount + auction.bid_amount * auction.bid_increase / WAD
};
assert!(bid_amount >= min_bid, BidTooLow);
```

**State Transition:**
```
let prev_bidder = auction.high_bidder;
let prev_bid = auction.bid_amount;

auction.high_bidder = bidder_account.id
auction.bid_amount = bid_amount
auction.bid_time = current_timestamp
// Extend deadline if within bid_duration of end
if auction.auction_deadline - current_timestamp < auction.bid_duration {
    auction.auction_deadline = current_timestamp + auction.bid_duration
}
```

**Chained Calls:**
```
// 1. Refund previous bidder (if any)
if prev_bidder != [0; 32] {
    TokenProgram::Transfer {
        amount_to_transfer: prev_bid (Wad),
        // accounts: [auction_logos_holding_id (PDA auth), previous_bidder_logos_holding_id]
    }
}

// 2. New bidder pays LOGOS to auction holding
TokenProgram::Transfer {
    amount_to_transfer: bid_amount,
    // accounts: [bidder_logos_holding_id (auth), auction_logos_holding_id]
}
```

---

### 2.4 `SettleAuction`

**Description:** After auction deadline, winner receives LSC; LOGOS bids are burned.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `auction_id` | Yes | — | The auction |
| 1 | `winner_lsc_holding_id` | Yes | — | Winner receives LSC |
| 2 | `auction_lsc_holding_id` | Yes | PDA | LSC held during auction |
| 3 | `auction_logos_holding_id` | Yes | PDA | LOGOS to burn |
| 4 | `logos_token_def_id` | No | — | For burning LOGOS |
| 5 | `lsc_market_oracle_id` | No | — | Timestamp |

**Validations:**
```
assert!(current_timestamp >= auction.auction_deadline, AuctionNotEnded);
assert!(!auction.settled, AuctionSettled);
assert!(auction.bid_amount > 0, NoBids);
```

**State Transition:**
```
auction.settled = true
```

**Chained Calls:**
```
// 1. Send LSC to winner
TokenProgram::Transfer {
    amount_to_transfer: auction.amount_to_sell / RAY,  // Rad → Wad
    // accounts: [auction_lsc_holding_id (PDA auth), winner_lsc_holding_id]
}

// 2. Burn LOGOS bids (deflationary)
TokenProgram::Burn {
    amount_to_burn: auction.bid_amount,
    // accounts: [logos_token_def_id, auction_logos_holding_id (PDA auth)]
    // Note: definition account first, then holder (authorized)
}
```

---

## 3. System Surplus and Debt Flow

```
Stability Fees
     │
     │ (TaxCollector → LSCEngine::UpdateAccumulatedRate)
     │ (LSC minted)
     ▼
SystemSurplusAccount (LSC balance)
     │
     ├── if net_surplus > buffer + auction_lot:
     │        AccountingEngine::AuctionSurplus
     │        → SurplusAuction: sell LSC for LOGOS
     │        → LOGOS burned (deflationary)
     │
     └── AccountingEngine::SettleDebt (offset debt against surplus)
              → burns surplus LSC, reduces debt queue

Bad Debt (from liquidations)
     │
     │ (LiquidationEngine → AccountingEngine::PushDebt)
     ▼
SystemDebtQueueAccount
     │
     ├── AccountingEngine::SettleDebt (offset against surplus)
     │
     ├── AccountingEngine::RecapitalizeWithLSC (governance injects LSC)
     │        → LSC transferred from governance to system_surplus
     │        → debt queue reduced directly
     │
     ├── AccountingEngine::RecapitalizeWithLOGOS (governance sells LOGOS)
     │        → LOGOS swapped for LSC on AMM
     │        → acquired LSC deposited to system_surplus
     │        → debt queue reduced directly
     │
     └── If surplus insufficient: debt remains as protocol liability
              → socialized via redemption price mechanism
```

### Bad Debt Resolution Philosophy

The LSC system does not mint new LOGOS to cover bad debt. Instead:

1. **Surplus offsets debt first** (`SettleDebt`): any stability fee surplus is used to cancel queued bad debt before triggering a surplus auction. Net surplus is what remains after this offset.
2. **Governance recapitalization** (`RecapitalizeWithLSC`, `RecapitalizeWithLOGOS`): governance may inject external capital to cancel bad debt directly. This does not involve minting new LOGOS. `RecapitalizeWithLSC` injects LSC directly; `RecapitalizeWithLOGOS` sells existing LOGOS treasury holdings for LSC via the AMM.
3. **Unresolved debt is a protocol liability**: if `queued_debt > surplus_balance` and governance does not recapitalize, the difference represents LSC in circulation that is not fully backed by collateral. This is reflected in the system's backing ratio.
4. **PI controller adjusts**: the redemption rate mechanism implicitly accounts for reduced backing — a weaker-backed LSC will tend to trade below redemption price, causing the PI controller to raise the redemption rate, incentivizing SAFEs to be closed (reducing LSC supply) until balance is restored.

---

## 4. Error Codes

```rust
enum AccountingEngineError {
    AlreadyInitialized        = 5000,
    Unauthorized              = 5001,
    SurplusNotSufficient      = 5002,
    InsufficientDebt          = 5003,
    DebtQueueFull             = 5004,
    TooSoon                   = 5006,
    InsufficientSurplus       = 5007,
    InvalidPDA                = 5008,
    ZeroAmount                = 5009,
    InsufficientLSC           = 5010,
    InsufficientLOGOS         = 5011,
}

enum SurplusAuctionError {
    AlreadyInitialized        = 5100,
    Unauthorized              = 5101,
    AuctionSettled            = 5102,
    AuctionExpired            = 5103,
    AuctionNotEnded           = 5104,
    BidTooLow                 = 5105,
    NoBids                    = 5106,
    InvalidPDA                = 5107,
}
```
