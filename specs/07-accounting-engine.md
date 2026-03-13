# LSC — Accounting Engine

## Overview

The Accounting Engine is the protocol's treasury. It manages the flow between:
- **System surplus** — LSC accumulated from stability fees (minted by TaxCollector via LSCEngine::UpdateAccumulatedRate)
- **Bad debt** — uncovered LSC debt from insolvent SAFEs pushed by LiquidationEngine

When surplus exceeds a threshold, surplus auctions are triggered: users bid LOGOS to receive protocol LSC; the LOGOS is burned (deflationary). When bad debt accumulates, debt auctions are triggered: users bid LSC to receive freshly minted LOGOS (inflationary).

**Program ID aliases:** `ACCOUNTING_ENGINE_ID`, `SURPLUS_AUCTION_ID`, `DEBT_AUCTION_ID`

---

## 1. AccountingEngine Program

### 1.1 Instruction Enum

```rust
enum AccountingEngineInstruction {
    /// One-time initialization
    Initialize {
        surplus_auction_params_id: [u8; 32],
        debt_auction_params_id: [u8; 32],
        lsc_engine_params_id: [u8; 32],
        lsc_token_def_id: [u8; 32],
        logos_token_def_id: [u8; 32],
        surplus_auction_amount: u128,           // Rad (surplus sold per auction)
        surplus_buffer: u128,                   // Rad (buffer kept before auctioning)
        debt_auction_lot_size: u128,            // Rad (debt covered per lot)
        initial_debt_auction_mint_amount: u128, // Wad (LOGOS offered per lot initially)
        max_debt_auction_size: u128,            // Rad (max debt per auction)
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

    /// Trigger a debt auction (permissionless, if conditions met)
    AuctionDebt,

    /// Update parameters (governance)
    UpdateParams {
        new_surplus_auction_amount: Option<u128>,
        new_surplus_buffer: Option<u128>,
        new_debt_auction_lot_size: Option<u128>,
        new_initial_debt_auction_mint_amount: Option<u128>,
        new_max_debt_auction_size: Option<u128>,
        new_surplus_auction_delay: Option<u64>,
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
accounting_params.debt_auction_nonce = 0

debt_queue.account_type = 62
debt_queue.queued_debt = 0
debt_queue.debt_on_auction = 0
debt_queue.entry_count = 0
```

**Chained Calls:**
```
// Initialize the system_surplus as a Token Program LSC holding account
TokenProgram::InitializeAccount {
    account: system_surplus_id,
    definition_id: lsc_token_def_id,
    holder_id: system_surplus_id,  // PDA is its own holder authority
}
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
- `liquidation_engine_params_id.id == accounting_params.debt_auction_params_id`
  (or: check against lsc_engine_params.liquidation_engine_params_id)

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
- `DebtQueueFull` — more than 256 entries (unlikely; settle first)

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
    // accounts: [system_surplus_id (PDA auth), lsc_token_def_id]
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
AND
debt_queue.debt_on_auction == 0  // no ongoing debt auctions
```

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | Yes | — | Update nonce + timestamp |
| 1 | `debt_queue_id` | No | — | Check debt_on_auction |
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
- `DebtAuctionOngoing` — cannot run surplus auction while debt auction active

---

### 1.6 `AuctionDebt`

**Description:** Initiates a debt auction when there is unhealed bad debt exceeding the surplus. LOGOS is minted and sold to market participants who pay LSC.

**Trigger conditions:**
```
queued_debt = debt_queue.queued_debt
surplus_balance_rad = system_surplus_lsc_balance * RAY

unhealed_debt_rad = queued_debt - surplus_balance_rad  // net debt after surplus offset

unhealed_debt_rad >= accounting_params.debt_auction_lot_size
AND
debt_queue.debt_on_auction == 0  // only one debt auction at a time (for simplicity)
```

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `accounting_params_id` | Yes | — | Update nonce |
| 1 | `debt_queue_id` | Yes | — | Update debt_on_auction |
| 2 | `new_debt_auction_id` (PDA) | Yes | PDA (new) | New debt auction account |
| 3 | `debt_auction_params_id` | No | — | Debt auction program |
| 4 | `lsc_market_oracle_id` | No | — | For timestamp |

**State Transition:**
```
let lot = min(
    accounting_params.debt_auction_lot_size,
    debt_queue.queued_debt - surplus_balance_rad,
);

debt_queue.debt_on_auction += lot
accounting_params.debt_auction_nonce += 1
```

**Chained Calls:**
```
DebtAuctionHouse::StartAuction {
    amount_to_raise: lot,  // Rad of LSC to raise
    initial_mint_amount: accounting_params.initial_debt_auction_mint_amount,  // Wad of LOGOS to offer
    // accounts: [debt_auction_params_id, new_debt_auction_id (auth)]
}
```

**Errors:**
- `InsufficientDebt` — unhealed debt < lot_size
- `DebtAuctionOngoing`

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
    // accounts: [auction_logos_holding_id (PDA auth), logos_token_def_id]
}
```

---

## 3. DebtAuctionHouse Program

**Program ID alias:** `DEBT_AUCTION_ID`

A **decreasing-amount auction**: the protocol needs to raise a fixed amount of LSC. Bidders compete by accepting fewer and fewer LOGOS for providing the needed LSC. The winner is whoever accepts the fewest LOGOS. The LOGOS offered is then minted.

### 3.1 Instruction Enum

```rust
enum DebtAuctionInstruction {
    /// Initialize (one-time)
    Initialize {
        accounting_engine_params_id: [u8; 32],
        logos_token_def_id: [u8; 32],
        lsc_token_def_id: [u8; 32],
        bid_decrease: u128,            // Wad (min reduction per bid, e.g. 5%)
        bid_duration: u64,             // seconds
        total_auction_length: u64,     // hard deadline
    },

    /// Start a new debt auction (called by AccountingEngine — privileged)
    StartAuction {
        amount_to_raise: u128,         // Rad (LSC needed)
        initial_mint_amount: u128,     // Wad (LOGOS offered initially)
    },

    /// Place a bid (offer to receive fewer LOGOS for the fixed LSC)
    DecreaseSoldAmount {
        auction_id: u64,
        logos_amount: u128,   // Wad (bidder's offered LOGOS amount — must be LESS than current)
    },

    /// Settle the auction (mint LOGOS to winner, send LSC to accounting engine)
    SettleAuction {
        auction_id: u64,
    },

    /// Restart expired auction with increased mint amount
    RestartAuction {
        auction_id: u64,
    },
}
```

---

### 3.2 `StartAuction`

**Authorization:** `accounting_engine_params_id` must be `is_authorized`.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `debt_auction_params_id` | No | — | — |
| 1 | `new_auction_id` (PDA) | Yes | PDA (new) | — |
| 2 | `accounting_engine_params_id` | No | Yes (PDA) | Authorization |
| 3 | `lsc_market_oracle_id` | No | — | Timestamp |

**State Transition:**
```
new_auction.account_type = 80
new_auction.auction_id = nonce
new_auction.amount_to_raise = amount_to_raise
new_auction.amount_to_mint = initial_mint_amount
new_auction.high_bidder = [0; 32]
new_auction.bid_time = current_timestamp
new_auction.auction_deadline = current_timestamp + debt_auction_params.total_auction_length
new_auction.bid_duration = debt_auction_params.bid_duration
new_auction.bid_decrease = debt_auction_params.bid_decrease
new_auction.settled = false
```

---

### 3.3 `DecreaseSoldAmount`

**Description:** Bidder says: "I'll pay the fixed LSC in exchange for only X LOGOS (less than current offer)."

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `auction_id` | Yes | — | The auction |
| 1 | `bidder_account` | No | Yes | Bidder |
| 2 | `bidder_lsc_holding_id` | Yes | Yes | Bidder's LSC (escrow) |
| 3 | `prev_bidder_lsc_holding_id` | Yes | — | Previous bidder's LSC (refunded) |
| 4 | `auction_lsc_holding_id` | Yes | PDA | Temp LSC holding |
| 5 | `lsc_market_oracle_id` | No | — | Timestamp |

**Validations:**
```
assert!(!auction.settled, AuctionSettled);
assert!(current_timestamp < auction.auction_deadline, AuctionExpired);

// Must offer FEWER LOGOS than current (decreasing auction)
let max_logos = if auction.amount_to_mint == initial_mint_amount {
    auction.amount_to_mint  // First bid can be at most initial
} else {
    auction.amount_to_mint - auction.amount_to_mint * auction.bid_decrease / WAD
};
assert!(logos_amount <= max_logos, BidNotLowEnough);
assert!(logos_amount > 0, ZeroBid);
```

**State Transition:**
```
let prev_bidder = auction.high_bidder;
let lsc_to_escrow = auction.amount_to_raise / RAY;  // Rad → Wad

auction.high_bidder = bidder_account.id
auction.amount_to_mint = logos_amount
auction.bid_time = current_timestamp
if auction.auction_deadline - current_timestamp < auction.bid_duration:
    auction.auction_deadline = current_timestamp + auction.bid_duration
```

**Chained Calls:**
```
// 1. Refund previous bidder's LSC
if prev_bidder != [0; 32] {
    TokenProgram::Transfer {
        amount_to_transfer: lsc_to_escrow,
        // accounts: [auction_lsc_holding_id (PDA auth), prev_bidder_lsc_holding_id]
    }
}

// 2. New bidder deposits LSC to escrow
TokenProgram::Transfer {
    amount_to_transfer: lsc_to_escrow,
    // accounts: [bidder_lsc_holding_id (auth), auction_lsc_holding_id]
}
```

---

### 3.4 `SettleAuction`

**Description:** Winner receives minted LOGOS; LSC goes to system surplus to cancel bad debt.

**Required Accounts:**

| # | Account | Writable | Auth | Description |
|---|---|---|---|---|
| 0 | `auction_id` | Yes | — | The auction |
| 1 | `debt_queue_id` | Yes | — | Reduce debt_on_auction |
| 2 | `auction_lsc_holding_id` | Yes | PDA | LSC to send to surplus |
| 3 | `accounting_engine_surplus_id` | Yes | — | Receives LSC |
| 4 | `winner_logos_holding_id` | Yes | — | Winner receives LOGOS |
| 5 | `logos_token_def_id` | No | — | For minting LOGOS |
| 6 | `lsc_market_oracle_id` | No | — | Timestamp |

**Validations:**
```
assert!(current_timestamp >= auction.auction_deadline, AuctionNotEnded);
assert!(!auction.settled, AuctionSettled);
assert!(auction.high_bidder != [0; 32], NoBids);
```

**State Transition:**
```
auction.settled = true
debt_queue.debt_on_auction -= auction.amount_to_raise
debt_queue.queued_debt -= auction.amount_to_raise
```

**Chained Calls:**
```
// 1. Send LSC from escrow to system surplus
TokenProgram::Transfer {
    amount_to_transfer: auction.amount_to_raise / RAY,  // Wad
    // accounts: [auction_lsc_holding_id (PDA auth), accounting_engine_surplus_id]
}

// 2. Mint LOGOS to winner (inflationary)
TokenProgram::Mint {
    amount_to_mint: auction.amount_to_mint,
    // accounts: [logos_token_def_id (minting auth = debt_auction PDA), winner_logos_holding_id]
}
```

**Note:** The DebtAuctionHouse must be granted minting authority over the LOGOS token definition at deployment time by LOGOS governance.

---

### 3.5 `RestartAuction` (DebtAuction)

**Description:** If a debt auction expires with no bids, increase the LOGOS offered (make it more attractive) and restart.

**Validations:**
```
assert!(current_timestamp > auction.auction_deadline, AuctionNotExpired);
assert!(!auction.settled, AuctionSettled);
```

**State Transition:**
```
// Offer more LOGOS (inflate the offer to attract bidders)
new_mint_amount = auction.amount_to_mint + auction.amount_to_mint * RESTART_MULTIPLIER / WAD
// RESTART_MULTIPLIER = 50_000_000_000_000_000 = 5%
auction.amount_to_mint = new_mint_amount
auction.auction_deadline = current_timestamp + debt_auction_params.total_auction_length
```

---

## 4. System Surplus and Debt Flow

```
Stability Fees
     │
     │ (TaxCollector → LSCEngine::UpdateAccumulatedRate)
     │ (LSC minted)
     ▼
SystemSurplusAccount (LSC balance)
     │
     ├── if surplus > buffer + auction_lot:
     │        AccountingEngine::AuctionSurplus
     │        → SurplusAuction: sell LSC for LOGOS
     │        → LOGOS burned (deflationary)
     │
     └── if bad debt > surplus:
              AccountingEngine::AuctionDebt
              → DebtAuction: mint LOGOS for LSC
              → LSC received cancels bad debt

Bad Debt (from liquidations)
     │
     │ (LiquidationEngine → AccountingEngine::PushDebt)
     ▼
SystemDebtQueueAccount
     │
     ├── AccountingEngine::SettleDebt (netting against surplus)
     └── AccountingEngine::AuctionDebt (when surplus insufficient)
```

---

## 5. Error Codes

```rust
enum AccountingEngineError {
    AlreadyInitialized        = 5000,
    Unauthorized              = 5001,
    SurplusNotSufficient      = 5002,
    InsufficientDebt          = 5003,
    DebtQueueFull             = 5004,
    DebtAuctionOngoing        = 5005,
    TooSoon                   = 5006,
    InsufficientSurplus       = 5007,
    InvalidPDA                = 5008,
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

enum DebtAuctionError {
    AlreadyInitialized        = 5200,
    Unauthorized              = 5201,
    AuctionSettled            = 5202,
    AuctionExpired            = 5203,
    AuctionNotEnded           = 5204,
    BidNotLowEnough           = 5205,
    ZeroBid                   = 5206,
    NoBids                    = 5207,
    AuctionNotExpired         = 5208,
    InvalidPDA                = 5209,
}
```
