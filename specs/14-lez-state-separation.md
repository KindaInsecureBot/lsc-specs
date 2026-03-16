# LSC — LEZ State Separation: Feasibility Study

> **Relationship to spec 10:** `10-privacy-considerations.md` is a high-level survey of privacy properties. This document is a feasibility study analyzing how LEZ's public/private state model interacts with every account type in LSC, what ZK infrastructure would be required to support private variants, and what the concrete costs and constraints are.

---

## 1. LEZ State Separation Model

LEZ separates on-chain state into two distinct regimes:

### 1.1 Public Accounts

A public account's `data` field is stored and readable in plaintext on-chain. Any program or user can read it without special authorization. State transitions are verified by the LEZ runtime via replayed execution: validators re-run the program with the provided inputs and confirm the outputs match the committed state transition.

Characteristics:
- Data readable by anyone (full transparency)
- Composable: other programs can pass a public account as a read-only input and inspect its fields
- Authority: only the `program_owner` may write to the account, but any program may read it
- LSC's current design uses public accounts exclusively

### 1.2 Private Accounts

A private account stores only a cryptographic **commitment** (e.g., a hash or Pedersen commitment) on-chain. The actual data is known only to the account owner (or delegated parties). State transitions are verified via zero-knowledge proofs: the program receives private inputs off-chain and produces a ZK proof that the transition is valid, without revealing the underlying values.

Characteristics:
- Commitment is public; data is not
- Account existence is detectable (the account ID exists on-chain)
- Owner proves possession via knowledge of a nullifier or preimage
- Composability is constrained: a downstream program cannot read a private account's fields directly — it can only verify ZK proofs about them
- State transitions require ZK circuit execution (computationally expensive)

### 1.3 Nullifiers

For private accounts that track spent state (analogous to UTXOs), a **nullifier** is a deterministic value derived from the account secret. When a private state is consumed, its nullifier is published, preventing double-spend without revealing which commitment was nullified.

### 1.4 Hybrid Execution

A single transaction can mix public and private account accesses:
- Public accounts are read directly
- Private accounts are accompanied by ZK proofs attesting to specific properties of their hidden state
- The verifying program checks the proof against the public commitment and the public inputs (e.g., a global debt ceiling)

This hybrid model is the key mechanism that allows private SAFEs to interact with public protocol state (accumulated rates, debt ceilings, oracle prices) without full disclosure.

---

## 2. Account-by-Account Analysis

For each account type, we analyze: visibility requirements, fields that must be public, fields that could be hidden, and what ZK infrastructure would be required to support a private variant.

### 2.1 SystemParamsAccount

**PDA:** `LSC_ENGINE_ID.derive(b"system_params")`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `governance_id` | Yes | All programs verify governance authority |
| `lsc_token_def_id` | Yes | Required by Token Program interactions |
| `logos_token_def_id` | Yes | Required by all deposit/withdraw operations |
| `oracle_relayer_params_id` | Yes | Lookup pointer used by all programs |
| `tax_collector_params_id` | Yes | Lookup pointer |
| `liquidation_engine_params_id` | Yes | Lookup pointer |
| `accounting_engine_params_id` | Yes | Lookup pointer |
| `global_settlement_state_id` | Yes | Emergency lookup |
| `settlement_active` | Yes | Critical safety flag — all programs must be able to halt |
| `safe_nonce` | Yes | Determines next SAFE PDA — must be predictable |
| `vault_nonce` | Yes | Determines next vault PDA |

**Verdict: Unconditionally Public.**

`SystemParamsAccount` is a global configuration registry. Every program in the system reads from it to discover other program addresses, check settlement state, and derive PDAs. There is no meaningful subset of this account that could be hidden without breaking protocol composability. Hiding `safe_nonce` would make SAFE IDs non-deterministic from the outside, which would break the ability to verify PDA derivations.

---

### 2.2 CollateralTypeAccount

**PDA:** `LSC_ENGINE_ID.derive(b"collateral_type" || collateral_token_def_id)`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `token_def_id` | Yes | Identifies what collateral this type tracks |
| `vault_id` | Yes | Pointer to collateral vault; needed for deposits/withdrawals |
| `accumulated_rate` | **Yes** | Every SAFE operation computes effective debt as `normalized_debt * accumulated_rate`. Must be readable by all programs. |
| `last_stability_fee_update` | Yes | Needed to compute time-elapsed for fee accrual |
| `stability_fee` | Yes | Governs fee accrual; must be auditable |
| `safety_ratio` | Yes | Used in collateral ratio checks during `GenerateDebt` |
| `liquidation_ratio` | Yes | Used by `LiquidationEngine`; liquidators must know threshold |
| `liquidation_penalty` | Yes | Determines penalty at liquidation; affects auction sizing |
| `debt_ceiling` | Yes | Global cap; enforcement requires reading this |
| `debt_floor` | Yes | Dust protection; must be checkable by `GenerateDebt` |
| `global_normalized_debt` | **Yes** | Sum of all SAFEs' `normalized_debt`; required for ceiling enforcement |
| `active` | Yes | Gate for new operations |
| `settlement_processed` | Yes | Settlement state |
| `final_collateral_price` | Yes | Post-settlement redemption rate |

**Verdict: Unconditionally Public.**

`accumulated_rate` is the linchpin: it is the multiplier converting all `normalized_debt` values into effective debt. Every SAFE owner, liquidator, and auction participant must read it. `global_normalized_debt` aggregates contributions from every SAFE of this type — hiding any individual SAFE's contribution requires it to be subtracted from this public total or managed via a private accumulator (see §6.3). There is no viable path to making this account private.

---

### 2.3 SafeAccount

**PDA:** `LSC_ENGINE_ID.derive(b"safe" || owner_account_id || nonce_bytes)`

This is the central privacy question. See §3 for the full deep dive. Summary here:

**Fields:**
| Field | Must Be Public? | Can Be Hidden? | Notes |
|---|---|---|---|
| `owner_id` | Partial | Yes | Can use stealth addresses or commitment |
| `collateral_type_id` | Partial | Yes | Reveals which asset class the SAFE uses |
| `collateral` | **No** | **Yes** | Core privacy-sensitive field |
| `normalized_debt` | **No** | **Yes** | Core privacy-sensitive field |
| `opened_at` | No | Yes | Timing correlation risk |
| `last_modified_at` | No | Yes | Timing correlation risk |
| `liquidated` | Partial | No | A closed SAFE must be publicly observable to prevent re-use |
| `allowed_operators` | No | Yes | Delegation list — privacy-sensitive |

**Verdict: Conditionally Private.**

A `SafeAccount` can in principle be made private, but it requires substantial ZK infrastructure (see §3 and §6). Without that infrastructure, it must be public for the liquidation mechanism to function. In v1, all SAFEs are public.

---

### 2.4 CollateralVaultAccount

**Owner:** `TOKEN_PROGRAM_ID`
**Account ID:** PDA of `LSC_ENGINE_ID`

The vault is a Token Program holding account. Its balance (total collateral locked for a given collateral type) is public because:

1. The Token Program's `TokenHolding` account model stores balances in plaintext
2. The vault serves the entire collateral type, not individual SAFEs — it is a **pooled** account
3. `LiquidationEngine` must authorize transfers from the vault during auctions; the auction amount must be verifiable

**Fields that are visible:**
- Total collateral balance for each collateral type (e.g., total LOGOS locked system-wide)

**What is hidden:**
- How that balance is distributed across individual SAFEs (without reading the SAFEs themselves)

**Verdict: Unconditionally Public** (given current Token Program architecture).

The vault reveals aggregate collateral per type but not per-SAFE breakdown. This is already a partial privacy property: an observer sees "5M LOGOS in the vault" but not "Alice has 1M, Bob has 4M." If SAFEs become private (v2), the vault balance still leaks aggregate collateral — but not individual positions. For stronger privacy, a shielded pool model replaces the shared vault with a commitment tree (see §4.3).

---

### 2.5 GlobalDebtAccount

**PDA:** `LSC_ENGINE_ID.derive(b"global_debt")`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `total_debt` | Yes | System-wide LSC in circulation; auditable health metric |
| `total_surplus` | Yes | Protocol surplus; used in accounting engine decisions |

**Verdict: Unconditionally Public.**

`total_debt` is the sum of `global_normalized_debt * accumulated_rate` across all collateral types. It must be readable for system health assessment and settlement computations. Hiding it would undermine trust in the protocol's solvency.

---

### 2.6 OracleRelayerParamsAccount

**PDA:** `ORACLE_RELAYER_ID.derive(b"oracle_relayer_params")`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `redemption_price` | **Yes** | Every SAFE operator and liquidator reads this to compute debt values |
| `redemption_rate` | **Yes** | Governs how `redemption_price` evolves; must be auditable |
| `last_redemption_price_update` | Yes | Time reference for lazy price computation |
| `redemption_rate_lower_bound` | Yes | Governance parameter; must be auditable |
| `redemption_rate_upper_bound` | Yes | Governance parameter; must be auditable |
| `collateral_oracle_id` | Yes | Pointer to price feed |
| `lsc_market_oracle_id` | Yes | Pointer to price feed |
| `pi_controller_state_id` | Yes | Authorization pointer |
| `collateral_price_safety_margin` | Yes | Liquidation-critical parameter |

**Verdict: Unconditionally Public.**

The `redemption_price` is the protocol's internal unit of account. All debt denominated in LSC is valued against it. Every user who wants to know their real debt burden must read it. The `redemption_rate` governs future price trajectory and must be auditable for SAFE owners to make informed decisions. No field here can be hidden without breaking the protocol's fundamental pricing mechanism.

---

### 2.7 PIControllerStateAccount

**PDA:** `PI_CONTROLLER_ID.derive(b"pi_controller_state")`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `kp`, `ki` | Yes | Governance-set parameters; auditable |
| `error_integral` | Yes | Determines current rate output; must be auditable for rate predictability |
| `last_update_time` | Yes | Needed to enforce `update_delay` |
| `update_delay` | Yes | Governance parameter |
| `per_second_cumulative_leak` | Yes | Governance parameter |
| `oracle_relayer_params_id` | Yes | Pointer |
| `lsc_market_oracle_id` | Yes | Pointer |
| `noise_barrier` | Yes | Governance parameter |
| `last_proportional_term` | Partial | Diagnostic; could be omitted but useful for transparency |
| `last_integral_term` | Partial | Diagnostic; could be omitted |

**Verdict: Unconditionally Public.**

The PI controller directly sets `redemption_rate` in `OracleRelayerParamsAccount`. The rate affects every SAFE in the system. The controller state — especially `error_integral` — must be public for users to anticipate rate changes and for governance to audit controller behavior. A private PI controller would make the redemption rate opaque, which would be a significant trust regression.

---

### 2.8 TaxCollectorParamsAccount

**PDA:** `TAX_COLLECTOR_ID.derive(b"tax_collector_params")`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `accounting_engine_surplus_id` | Yes | Pointer |
| `lsc_engine_params_id` | Yes | Pointer |
| `global_stability_fee` | Yes | Base fee rate; all SAFE owners need to know their fee burden |

**Verdict: Unconditionally Public.**

The `global_stability_fee` affects every SAFE's debt cost. It must be publicly readable and auditable. The fee accrual mechanism (`AccrueStabilityFee`) updates `CollateralTypeAccount.accumulated_rate`, which is itself public. There is nothing here that can be hidden.

---

### 2.9 LiquidationEngineParamsAccount

**PDA:** `LIQUIDATION_ENGINE_ID.derive(b"liquidation_engine_params")`

**Fields:**
| Field | Must Be Public? | Rationale |
|---|---|---|
| `lsc_engine_params_id` | Yes | Pointer |
| `collateral_auction_params_id` | Yes | Pointer |
| `accounting_engine_params_id` | Yes | Pointer |
| `oracle_relayer_params_id` | Yes | Pointer |
| `on_auction_system_coin_limit` | Yes | Circuit breaker; must be auditable |
| `current_on_auction_system_coins` | Yes | Tracks in-flight auction debt; public for composability |

**Verdict: Unconditionally Public.**

The `on_auction_system_coin_limit` and `current_on_auction_system_coins` form a circuit breaker preventing cascading liquidations. For this mechanism to function correctly and be auditable, both values must be public. Liquidators checking whether they can trigger a new liquidation must read `current_on_auction_system_coins`.

---

### 2.10 LiquidationAccount

**PDA:** `LIQUIDATION_ENGINE_ID.derive(b"liquidation" || safe_id)`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `safe_id` | Yes | Links to the SAFE being liquidated |
| `auction_id` | Yes | Links to the created auction |
| `collateral_seized` | Yes | Auction participants need to know what's being sold |
| `debt_to_cover` | Yes | Auction participants need to know the debt target |
| `liquidated_at` | Yes | Timestamp for audit |

**Verdict: Unconditionally Public.**

A `LiquidationAccount` is created as a side-effect of liquidating a public SAFE. Its contents are required for the `CollateralAuctionHouse` to function — auction buyers must know the collateral amount and debt target. Even in a hypothetical private-SAFE world, the act of liquidation necessarily forces partial disclosure: the collateral being auctioned must be visible so buyers can bid.

---

### 2.11 CollateralAuctionHouseParamsAccount

**PDA:** `COLLATERAL_AUCTION_ID.derive(b"collateral_auction_params")`

**Verdict: Unconditionally Public.**

All fields are governance parameters (discount rates, TTL, minimum bids) or program pointers. Auction parameters must be publicly known for participants to evaluate bids and for the system to be auditable. No field can be hidden.

---

### 2.12 CollateralAuctionAccount

**PDA:** `COLLATERAL_AUCTION_ID.derive(b"collateral_auction" || auction_nonce_bytes)`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `auction_id` | Yes | Identifier |
| `collateral_type_id` | Yes | What's being sold |
| `collateral_vault_id` | Yes | Pointer to collateral source |
| `amount_to_raise` | Yes | Buyers need to know the debt target |
| `collateral_to_sell` | Yes | Buyers need to know remaining supply |
| `amount_raised` | Yes | Tracks progress; buyers need this to avoid overbidding |
| `safe_id` | Yes | Provenance for audit |
| `created_at`, `expires_at` | Yes | Timing for restart logic |
| `settled` | Yes | Terminal state |
| `forgone_collateral` | Yes | Returned to SAFE owner; must be verifiable |

**Verdict: Unconditionally Public.**

Auctions are inherently a public-goods mechanism: competition among buyers requires full information. Hiding `collateral_to_sell` or `amount_to_raise` would prevent rational bidding. The current design uses a Dutch auction with a fixed discount — buyers must know the terms to participate. Commit-reveal bidding is theoretically possible to prevent front-running, but does not change the fact that final auction state must be public.

---

### 2.13 AccountingEngineParamsAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"accounting_engine_params")`

**Verdict: Unconditionally Public.**

Governance parameters controlling surplus auctions (`surplus_auction_amount`, `surplus_buffer`, timing fields) must be public. These parameters determine when system health events trigger, and must be auditable for governance accountability.

---

### 2.14 SystemSurplusAccount

**Owner:** `TOKEN_PROGRAM_ID`

This is a Token Program holding account storing LSC surplus. Its balance is visible to anyone:
- The `AccountingEngine` reads it to determine when to trigger surplus auctions
- Users can read it to assess protocol health

**Verdict: Unconditionally Public.**

Protocol surplus is a key health signal. Hiding it would undermine trust. Furthermore, the surplus auction mechanism requires the `AccountingEngine` to authorize transfers from it, which relies on the Token Program's public balance accounting.

---

### 2.15 SystemDebtQueueAccount

**PDA:** `ACCOUNTING_ENGINE_ID.derive(b"debt_queue")`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `queued_debt` | Yes | Total bad debt pending; protocol health signal |
| `entries[].debt` | Yes | Individual debt events; auditable |
| `entries[].pushed_at` | Yes | Ordering and timestamp for audit |

**Verdict: Unconditionally Public.**

The debt queue represents bad debt from failed liquidations. It must be publicly observable for governance to assess whether recapitalization is needed. The `AccountingEngine::SettleDebt` function offsets queued debt against surplus — this operation is auditable precisely because both sides are public.

---

### 2.16 SurplusAuctionAccount

**PDA:** `SURPLUS_AUCTION_ID.derive(b"surplus_auction" || auction_nonce_bytes)`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `auction_id` | Yes | Identifier |
| `amount_to_sell` | Yes | Buyers need to know what LSC they're bidding for |
| `bid_amount` | Yes | Current highest bid; competition requires visibility |
| `high_bidder` | Yes | Required for bid refund logic |
| `bid_time`, `auction_deadline` | Yes | Timing for deadline enforcement |
| `bid_increase` | Yes | Minimum increment must be known |
| `settled` | Yes | Terminal state |

**Verdict: Unconditionally Public.**

Same argument as `CollateralAuctionAccount`. Competitive auctions require public bid state.

---

### 2.17 OracleConfigAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"oracle_config")`

**Verdict: Unconditionally Public.**

Global oracle configuration (`max_feed_age`, `min_feeds_for_median`, `feed_count`) must be public for users to assess oracle quality and for programs to validate feed freshness. Hiding these parameters would make the oracle system's trust model opaque.

---

### 2.18 OracleFeedAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"feed" || feed_id_bytes)`

**Fields:**
| Field | Must Be Public? | Notes |
|---|---|---|
| `feed_id` | Yes | Identifier |
| `symbol` | Yes | What asset this feed tracks |
| `keeper_id` | Partial | Could use stealth keeper addresses for keeper privacy |
| `price` | **Yes** | The whole point of an oracle |
| `confidence` | Yes | Price quality signal; programs use it |
| `timestamp` | Yes | Freshness check |
| `active` | Yes | Feed validity flag |

**Verdict: Unconditionally Public** (with a note on keeper privacy).

Prices must be public — that is definitionally the purpose of an oracle. `keeper_id` could theoretically be replaced with a group key (threshold oracle) to provide keeper anonymity, but the price and timestamp must remain public. See `10-privacy-considerations.md §6` for keeper privacy options.

---

### 2.19 MedianOracleAccount

**PDA:** `ORACLE_PROGRAM_ID.derive(b"median" || symbol_bytes)`

**Verdict: Unconditionally Public.**

The median price and contributing feeds must be public. The PI controller, liquidation engine, and SAFE operators all read `median_price` and `median_timestamp`. Hiding any field would break the oracle aggregation logic.

---

### 2.20 GlobalSettlementStateAccount

**PDA:** `GLOBAL_SETTLEMENT_ID.derive(b"settlement_state")`

**Verdict: Unconditionally Public.**

Global settlement is an emergency action. Its state — whether active, when triggered, the final redemption price, which collateral types have been processed — must be immediately visible to all participants. SAFE owners must know if settlement is active to decide whether to redeem collateral or burn LSC. This is perhaps the most transparency-critical account in the system.

---

### 2.21 CollateralRedemptionAccount

**PDA:** `GLOBAL_SETTLEMENT_ID.derive(b"redemption" || collateral_type_id)`

**Verdict: Unconditionally Public.**

Post-settlement redemption rates (`collateral_per_lsc`) must be public for LSC holders to compute their redemption entitlements. The `total_lsc_redeemed` and `collateral_available` fields are needed for accounting verification. No field can be hidden.

---

## 3. The SAFE Privacy Problem (Deep Dive)

`SafeAccount` is the only account type where genuine privacy is architecturally feasible and user-desirable. This section analyzes the problem in depth.

### 3.1 What Must Be Known for Protocol Correctness

The LSC protocol requires these invariants to hold at all times:

**I1 (Solvency):** For every SAFE: `collateral * collateral_price / (normalized_debt * accumulated_rate) >= liquidation_ratio`

**I2 (Ceiling):** For every collateral type: `global_normalized_debt * accumulated_rate <= debt_ceiling` (in Rad units)

**I3 (Floor):** For every SAFE with non-zero debt: `normalized_debt * accumulated_rate >= debt_floor`

**I4 (Liquidatability):** Any SAFE violating I1 can be identified and liquidated by any participant.

In the current public design, these invariants are checkable by anyone reading on-chain state. In a private SAFE design, they must be enforced via ZK proofs without revealing the underlying `collateral` and `normalized_debt` values.

### 3.2 What Liquidators Need vs. What Can Be Hidden

**What liquidators need to do their job:**
1. Identify which SAFEs are undercollateralized (violate I1)
2. Know `collateral_seized` and `debt_to_cover` to start the auction (amounts must eventually become public)

**What can remain hidden:**
1. Exact `collateral` and `normalized_debt` values while the SAFE is healthy
2. SAFE owner identity (with stealth addresses or commitment-based ownership)
3. When the SAFE was opened and last modified
4. Delegation list (`allowed_operators`)

The fundamental tension: **liquidators need to discover undercollateralized SAFEs, but healthy SAFE owners want their positions hidden.** These goals are in direct conflict. The only resolution is a mechanism where:
- Healthy SAFEs publish proof of health (without revealing amounts), OR
- Unhealthy SAFEs are forced to disclose (triggering liquidation)

### 3.3 ZK Health Proof Design

A ZK health proof is a non-interactive zero-knowledge proof that asserts:

```
Statement: "I know (collateral, normalized_debt, salt) such that:
  1. commitment == Hash(collateral, normalized_debt, salt)
  2. collateral * collateral_price / (normalized_debt * accumulated_rate) >= liquidation_ratio"

Public inputs: commitment, collateral_price (from oracle), accumulated_rate, liquidation_ratio
Private inputs: collateral, normalized_debt, salt
```

This proof can be verified by anyone on-chain without learning `collateral` or `normalized_debt`. The prover is the SAFE owner.

**Arithmetic formulation** (avoiding division in ZK circuits, which is expensive):

```
Prove: collateral * collateral_price >= normalized_debt * accumulated_rate * liquidation_ratio

Where:
  - collateral_price: Wad (18 decimals)
  - accumulated_rate: Ray (27 decimals)
  - liquidation_ratio: Ray (27 decimals)
  - collateral: Wad
  - normalized_debt: Wad

Effective comparison (all in Rad = 45 decimals):
  collateral * collateral_price [Wad * Wad = 36 dec] * RAY [27 dec] = Rad-equivalent
  vs.
  normalized_debt * accumulated_rate * liquidation_ratio [Wad * Ray * Ray = 72 dec — must scale down]
```

The circuit must carefully manage fixed-point precision. A practical approach:
1. Scale `collateral * collateral_price` to Rad by multiplying by RAY (= 10^27)
2. Compare against `normalized_debt * accumulated_rate * liquidation_ratio` also scaled to Rad
3. Assert left-hand side >= right-hand side

### 3.4 The Forced Disclosure Problem

When a SAFE goes underwater (violates I1), the protocol must liquidate it. With private SAFEs, there is no liquidator that can discover the SAFE is unhealthy — unless the SAFE owner discloses, or the system has a mechanism to force disclosure.

**Option A: Liveness-Based Forced Disclosure**

The SAFE owner must submit a ZK health proof at most every `health_proof_period` seconds (e.g., 24 hours). If no proof is submitted:
- The SAFE is considered unhealthy by default
- A liquidator can trigger a forced liquidation
- The liquidator must reveal the SAFE's commitment preimage (the actual `collateral` and `normalized_debt`) to proceed — but only the owner has this information

This creates a problem: the owner will not cooperate with their own liquidation. Resolution requires a **ZK liquidator circuit**:

```
ZK Liquidation Proof:
  Statement: "I know (collateral, normalized_debt, salt) such that:
    1. commitment == Hash(collateral, normalized_debt, salt)
    2. collateral * collateral_price / (normalized_debt * accumulated_rate) < liquidation_ratio"

This proof reveals that the SAFE is unhealthy without requiring owner cooperation —
but it requires the liquidator to know the SAFE's private state.
```

How does a liquidator learn the SAFE's private state? Possible answers:
1. The owner shared it (cooperative liquidation — unlikely)
2. The owner is the liquidator (self-liquidation)
3. The state was leaked via a side-channel (e.g., historical disclosure)
4. The liquidator brute-forces small commitment spaces (not feasible for large amounts)

**This is the fundamental unsolvability of private liquidation:** without trusted hardware or cooperative disclosure, there is no mechanism for a third-party liquidator to prove a private SAFE is unhealthy without already knowing its contents.

**Option B: Escrow-Based Disclosure**

The SAFE owner deposits a disclosure key with a trusted enclave or multi-party committee. The committee can decrypt and reveal the SAFE state to liquidators if the liveness proof fails. This requires:
- A trust assumption in the committee
- A secure channel for key escrow
- A TEE (Trusted Execution Environment) or threshold MPC

This is not ZK-native and introduces new trust assumptions.

**Option C: Public Liquidation Threshold with Private Healthy State**

A hybrid model: SAFEs above a "healthy buffer" (e.g., CR >= 200%) can be private. SAFEs approaching the liquidation ratio must become public (e.g., when CR drops below 150%). This allows SAFE owners to voluntarily reveal when approaching the danger zone, or face forced revelation:

```
Transition rule:
  If CR >= healthy_threshold (200%): SAFE may be private
  If healthy_threshold > CR >= liquidation_ratio: SAFE must be public
  If CR < liquidation_ratio: SAFE is liquidatable (already public)
```

This requires:
1. A ZK proof of the CR range: `Prove: CR is in [healthy_threshold, ∞)`
2. The transition from private to public must be verifiable (the commitment must hash to the revealed values)
3. The forced-public rule creates a covert channel: observers can infer that a SAFE that recently became public was approaching liquidation

### 3.5 Interaction with accumulated_rate

`accumulated_rate` is public and changes over time as the `TaxCollector` accrues stability fees. A private SAFE stores `normalized_debt` (fixed at debt generation time), and its effective debt at any moment is:

```
effective_debt = normalized_debt * accumulated_rate
```

Since `accumulated_rate` is public and monotonically increasing, and `effective_debt` can be checked against the SAFE's commitment via ZK proof, the privacy of `normalized_debt` is maintained even as `accumulated_rate` changes — the proof is updated against the current `accumulated_rate`.

However, this creates a subtlety: a ZK health proof must use the **current** `accumulated_rate` as a public input. If the proof is computed with a stale `accumulated_rate`, it may attest to a healthier collateral ratio than actually exists. Therefore:

- Health proofs must include `accumulated_rate` as a public input with a freshness bound
- The verifier must check that the `accumulated_rate` used in the proof was the most recently accrued value (i.e., `AccrueStabilityFee` has been called recently)

### 3.6 Debt Ceiling Enforcement with Private Debt

Enforcing the debt ceiling (`I2`) with private SAFEs requires knowing the **aggregate** `normalized_debt` across all SAFEs of a given collateral type — without learning any individual SAFE's `normalized_debt`.

The current mechanism: `CollateralTypeAccount.global_normalized_debt` is a public running total. When a SAFE generates debt, it increments this value. When it repays, it decrements. This is trivially impossible to maintain privately if individual contributions are hidden.

**Solution: Merkle Sum Tree**

A Merkle sum tree commits to a set of values such that the root commits to both the set of elements and their sum. Each leaf is `Hash(normalized_debt_i, nullifier_i)`, and internal nodes aggregate sums. The root commits to `Σ normalized_debt_i` without revealing individual values.

To generate debt privately:
1. The SAFE owner computes a ZK proof that their new `normalized_debt` (post-generation) plus the existing Merkle sum does not exceed `debt_ceiling / accumulated_rate`
2. The Merkle tree root is updated (this requires a sequential update — concurrent debt generation is problematic)
3. The new global sum is committed in `CollateralTypeAccount.global_normalized_debt`

**Problem: Concurrent Updates**

If two SAFE owners simultaneously generate debt, both might generate proofs against the same Merkle root, and both proofs might individually satisfy the ceiling, but together they exceed it. This is a **double-spend-style race condition**. Solutions:
- Sequential mempool ordering (debt generation transactions are serialized)
- Pessimistic locking on `CollateralTypeAccount` (only one debt operation at a time)
- Epoch-based batching (debt requests are accumulated into batches, then a single circuit update is applied)

All of these have significant throughput implications. Private debt generation is inherently slower than public.

### 3.7 Global Normalized Debt Tracking

Even if individual SAFE debts are private, `CollateralTypeAccount.global_normalized_debt` must remain a publicly correct total. With a Merkle sum tree, the tree root serves as the commitment to the aggregate. The `global_normalized_debt` field in `CollateralTypeAccount` is replaced by a Merkle root:

```rust
// Hypothetical v2 private extension to CollateralTypeAccount
pub normalized_debt_tree_root: [u8; 32],   // Merkle root committing to all normalized_debt_i
pub normalized_debt_tree_sum: u128,         // Sum of all normalized_debt_i (public, committed by root)
```

This allows the ceiling check to remain public (`normalized_debt_tree_sum * accumulated_rate <= debt_ceiling`) while individual positions are private.

---

## 4. Token Transfer Privacy

### 4.1 Deposit/Withdrawal Privacy for Collateral

When a user deposits LOGOS into the collateral vault:
1. They transfer from their LOGOS holding account to `CollateralVaultAccount`
2. The Token Program records this as a balance change in both accounts
3. Both the source account and the vault balance are publicly observable

**What is leaked:**
- The source LOGOS account (linkable to the depositor)
- The amount deposited
- The timing of deposit (correlatable with SAFE creation)

**Mitigation: Intermediate Accounts**

A user can route through N intermediate accounts to break the deposit chain. This is pseudonymous, not private — a sufficiently motivated analyst can trace the chain. A true privacy solution requires a shielded pool (§4.3).

**Mitigation: Batch Deposits**

If multiple users deposit in the same block, the vault balance changes by a sum, reducing individual attribution — but individual transfer transactions are still separately visible.

### 4.2 LSC Minting Privacy

When a SAFE generates debt, the LSC Engine mints new LSC directly to the SAFE owner's LSC holding account. This reveals:
- That the recipient generated new LSC
- The amount minted
- The timing

**Can minting be hidden?**

In the current architecture, no. The Token Program records the mint as a public balance increase. A private minting scheme would require the Token Program itself to support private mints (commitment-based balances), which is a fundamental change to the Token Program — far beyond LSC's scope.

**Theoretical private minting:**

```
Private LSC mint:
1. SAFE owner provides a ZK proof of debt validity
2. Mint creates a "note commitment" rather than a public balance increment
3. The note commitment is: Hash(amount, recipient, salt)
4. The note can be spent by proving knowledge of the preimage + nullifier
```

This is essentially a ZK-based UTXO model for LSC. It would require an entirely new Token Program variant and is out of scope for the foreseeable future.

### 4.3 Shielded Pool Concepts for LOGOS Collateral

A **shielded pool** replaces the shared `CollateralVaultAccount` with a commitment tree. Users deposit LOGOS into the pool and receive a note commitment; they withdraw by proving ownership of a note and publishing its nullifier.

```
Shielded LOGOS Pool:
  State: commitment_tree: MerkleTree<Hash(amount, recipient, salt)>
         nullifier_set: Set<Hash(nullifier_key, commitment)>

  Deposit(amount):
    1. Transfer LOGOS from user holding → pool account (public)
    2. Compute commitment = Hash(amount, recipient, salt)
    3. Insert commitment into tree
    Public: amount, deposit timing
    Private: nothing (deposit is still public)

  Use as SAFE Collateral(commitment, ZK_proof):
    1. Prove: commitment is in the tree (Merkle inclusion proof)
    2. Prove: commitment.amount >= required_collateral
    3. Publish nullifier to prevent double-use
    Public: nullifier, that some note was consumed
    Private: amount, recipient, salt
```

This provides **collateral privacy at the SAFE level** — the amount of collateral in a specific SAFE is hidden — but the initial deposit into the pool is still public. For stronger privacy, deposits to the pool must be batched or routed through a mixer.

**Key constraint:** The current Token Program does not support shielded pools. This would require a separate Collateral Pool program with its own ZK verification logic.

### 4.4 UTXO vs. Account Model for Private Balances

The UTXO model (Unspent Transaction Output) is inherently more privacy-friendly than the account model because:
- There is no persistent account balance to track
- Each note is a fresh commitment; spent notes are replaced by new commitments
- The link between sender and recipient is broken by the note model

The account model (current LSC design) stores balances persistently and makes them trivially linkable. Converting LSC to a UTXO model would require:
1. A new Token Program variant supporting ZK-based note commitments
2. A ZK circuit for each Token operation (transfer, mint, burn)
3. A nullifier set to prevent double-spend
4. Changes to how the Accounting Engine, Liquidation Engine, and Auction House interact with LSC balances

This is a comprehensive redesign of the token layer, not an incremental privacy feature.

---

## 5. Classification Matrix

### 5.1 Unconditionally Public

State that cannot be hidden without breaking protocol invariants. No amount of ZK infrastructure can make these private while preserving protocol correctness.

| Account / Field | Why Unconditionally Public |
|---|---|
| `SystemParamsAccount` (all fields) | Global registry; all programs must read it |
| `CollateralTypeAccount.accumulated_rate` | Required multiplier for all debt computations |
| `CollateralTypeAccount.global_normalized_debt` | Debt ceiling enforcement |
| `CollateralTypeAccount.safety_ratio` | Required for debt generation checks |
| `CollateralTypeAccount.liquidation_ratio` | Required for liquidation trigger |
| `CollateralTypeAccount.debt_ceiling` | Required for ceiling enforcement |
| `CollateralTypeAccount.stability_fee` | Required for fee accrual |
| `OracleRelayerParamsAccount.redemption_price` | Unit of account for all debt |
| `OracleRelayerParamsAccount.redemption_rate` | Governs price trajectory |
| `MedianOracleAccount.median_price` | Collateral valuation for liquidations |
| `PIControllerStateAccount` (all fields) | Rate-setting mechanism; auditable |
| `TaxCollectorParamsAccount.global_stability_fee` | Fee policy; auditable |
| `LiquidationEngineParamsAccount` (all fields) | Liquidation circuit breaker |
| `CollateralAuctionAccount` (all fields) | Auction mechanics |
| `SurplusAuctionAccount` (all fields) | Auction mechanics |
| `GlobalDebtAccount` (all fields) | System health signal |
| `SystemSurplusAccount` (balance) | Protocol health; surplus auction trigger |
| `SystemDebtQueueAccount` (all fields) | Bad debt accounting |
| `GlobalSettlementStateAccount` (all fields) | Emergency action; must be immediately visible |
| `CollateralRedemptionAccount` (all fields) | Post-settlement redemption rates |
| `OracleFeedAccount.price`, `timestamp` | Oracle data is definitionally public |

### 5.2 Conditionally Private

State that can be hidden *if* specific ZK infrastructure is built and the associated trade-offs are accepted.

| Account / Field | Privacy Condition | Trade-off |
|---|---|---|
| `SafeAccount.collateral` | Requires ZK health proofs + shielded vault | Liquidators cannot directly discover underwater SAFEs |
| `SafeAccount.normalized_debt` | Requires Merkle sum tree for aggregate tracking | Sequential debt generation; throughput impact |
| `SafeAccount.owner_id` | Stealth addresses or commitment-based ownership | PDA derivation changes; SAFE recovery becomes harder |
| `SafeAccount.opened_at`, `last_modified_at` | Can be elided from commitment | Timing correlation attacks remain via deposit patterns |
| `SafeAccount.allowed_operators` | Can be committed | Operator delegation proofs needed |
| `LiquidationAccount.collateral_seized`, `debt_to_cover` | Only if SAFE was private | Liquidation necessarily forces partial disclosure |
| `OracleFeedAccount.keeper_id` | Threshold oracle with group key | Keeper accountability reduced |

### 5.3 Naturally Private

State that is already private or where privacy is structurally free.

| Item | Why Naturally Private |
|---|---|
| SAFE owner real-world identity | No KYC; account IDs are pseudonymous |
| Relationship between multiple SAFEs | No on-chain linkage unless owner reuses accounts |
| Internal SAFE accounting logic | Program execution is verified by output; inputs can vary |
| LSC destination (post-minting, if mixed) | LSC is fungible; post-issuance transfers can be mixed |

---

## 6. ZK Infrastructure Requirements

If LSC v2 were to implement private SAFEs, the following ZK circuits would be required. This section describes each circuit's statement, inputs, and on-chain verification requirements.

### 6.1 Collateral Ratio Proof Circuit

**Purpose:** Prove a SAFE is healthy without revealing collateral or debt amounts.

**Statement:**
```
Given public inputs:
  - commitment C = Hash(collateral, normalized_debt, salt)
  - collateral_price P  [Wad, from MedianOracleAccount]
  - accumulated_rate R  [Ray, from CollateralTypeAccount]
  - liquidation_ratio L [Ray, from CollateralTypeAccount]

Prove knowledge of private inputs (collateral, normalized_debt, salt) such that:
  1. C = Hash(collateral, normalized_debt, salt)
  2. collateral * P >= normalized_debt * R * L   [all scaled to Rad]
  3. collateral > 0
  4. normalized_debt >= 0
```

**On-chain verification:** The LSC Engine program verifies the proof against the SAFE's stored commitment and the current public parameters. A verified proof updates a `last_health_proof_at` timestamp in the SAFE's (public) metadata.

**Proof size:** Approximately 200-500 bytes for a Groth16 proof. Verification cost: O(1) pairing operations.

**Freshness requirement:** The proof must use current oracle values. Implementors should include a `proof_timestamp` as a public input and reject proofs older than `max_proof_age` (e.g., 1 hour).

### 6.2 Debt Ceiling Proof Circuit

**Purpose:** Prove that adding new debt to a private SAFE does not exceed the global debt ceiling.

**Statement:**
```
Given public inputs:
  - old Merkle root M_old (committing to all existing normalized_debt values)
  - new Merkle root M_new (after adding/modifying one leaf)
  - old tree sum S_old
  - new tree sum S_new = S_old + delta_normalized_debt
  - debt_ceiling D [from CollateralTypeAccount]
  - accumulated_rate R [from CollateralTypeAccount]

Prove:
  1. M_new is a valid Merkle tree obtained from M_old by updating leaf i
  2. S_new = S_old + delta_normalized_debt (sum is correctly updated)
  3. S_new * R <= D  (ceiling not exceeded, all in Rad)
  4. delta_normalized_debt is the difference between old and new normalized_debt for SAFE i
  5. new_commitment_i = Hash(new_collateral_i, new_normalized_debt_i, salt_i)
  6. New collateral ratio is >= safety_ratio (includes collateral ratio check)
```

**On-chain verification:** Updates `CollateralTypeAccount.normalized_debt_tree_root` and `normalized_debt_tree_sum`. This is the most complex circuit and the key bottleneck for private debt generation.

**Concurrency problem:** Two concurrent calls would both provide proofs against `M_old`, and both would be accepted by the runtime independently — but together they might exceed the ceiling. **Resolution required:** The LEZ runtime must serialize writes to `CollateralTypeAccount`. In practice, this means only one debt-modification transaction for a given collateral type can be in-flight at a time. This is a hard throughput constraint.

### 6.3 Nullifier-Based SAFE Lifecycle Circuit

**Purpose:** Allow SAFEs to be opened and closed using nullifiers, preventing SAFE replay and enabling ownership transfer without revealing SAFE contents.

**SAFE creation:**
```
Publish: commitment C = Hash(collateral=0, normalized_debt=0, salt, owner_nullifier_key)
Nullifier key: owner holds nullifier_key (secret)
SAFE ID: derived from commitment (not from owner_id as in v1)
```

**SAFE modification:**
```
Spend old commitment: publish nullifier = Hash(nullifier_key, C_old)
Create new commitment: C_new = Hash(new_collateral, new_normalized_debt, new_salt)
Prove: C_old → C_new transition is valid (amounts change correctly, ratio maintained)
```

**SAFE closure:**
```
Publish nullifier for C_current
Return remaining collateral to user
Burn outstanding LSC
```

This architecture means a SAFE is not "modified in place" — each modification consumes the old commitment and creates a new one. The nullifier set (a public set of spent nullifiers) prevents re-use.

**Impact on PDA derivation:** The current SAFE PDA includes `owner_account_id`. In a nullifier-based scheme, the SAFE ID would be derived from the commitment (or a sequence number), not from the owner. This breaks the ability to look up a user's SAFEs by their account ID — instead, users must track their own SAFE IDs off-chain.

### 6.4 Merkle Sum Tree for Aggregate Debt Tracking

**Data structure:** A binary Merkle tree where each leaf stores:
```
leaf_i = Hash(nullifier_key_i, normalized_debt_i)
```
Internal nodes aggregate: `node = Hash(left_child, right_child, left_sum + right_sum)`
Root commits to: all `normalized_debt_i` values and their sum.

**Operations supported:**
- Insert: Add new leaf (new SAFE created with zero debt)
- Update: Replace leaf (debt amount changes)
- Delete: Zero out leaf (SAFE closed, debt repaid)
- Sum proof: Prove the sum equals `S` without revealing individual values

**Proof size:** O(log N) for a tree with N SAFEs. For 10,000 SAFEs, depth ≈ 14, proof ≈ 14 × 32 bytes = 448 bytes (excluding ZK wrapper).

**Initialization:** With public SAFEs in v1, migration to a Merkle sum tree requires computing the initial root from all existing SAFEs' `normalized_debt` values — a one-time migration operation.

---

## 7. Feasibility Assessment

This section evaluates the feasibility of three levels of privacy for LSC, without prescribing implementation order or timelines. Each level is assessed independently on technical complexity, ZK infrastructure cost, and impact on protocol properties.

### 7.1 Baseline: Public SAFEs, Pseudonymous Operation

**What it means:** All accounts are public. Privacy is limited to pseudonymity (no real-world identity linkage).

**Privacy properties provided:**
- No KYC — account IDs are not linked to identities by default
- LSC fungibility — post-issuance, LSC cannot be traced to specific SAFEs
- Multiple accounts — users can operate multiple SAFEs from different accounts

**Privacy properties NOT provided:**
- Position confidentiality (anyone can see your collateral and debt)
- Deposit/withdrawal privacy (token transfers are public)
- Minting privacy (new LSC creation is visible)

**Feasibility:** Trivial — this is the default behavior of the system as currently specified. No additional infrastructure needed.

**Trade-off:** All user positions are fully transparent. Competitors, front-runners, and adversaries can observe SAFE health in real-time, enabling targeted liquidation strategies and position copying.

### 7.2 Partial Privacy: Private SAFE Positions with ZK Health Proofs

**What it means:** System-level accounts remain public. Individual SAFEs can optionally hide their `collateral` and `normalized_debt` behind ZK commitments.

**Required infrastructure:**
1. A `commitment` field on `SafeAccount` (or a parallel `PrivateSafeAccount` type)
2. Collateral Ratio Proof circuit (§6.1) — proves CR ≥ liquidation_ratio without revealing amounts
3. Debt Ceiling Proof circuit (§6.2) — proves private debt contribution fits within the ceiling via Merkle sum tree
4. Liveness enforcement: `last_health_proof_at` timestamp with mandatory periodic re-proof
5. Forced disclosure path for liquidation (unavoidable — collateral must be visible for auction buyers)

**What remains public even with this approach:**
- Total vault balance (aggregate collateral per type)
- Total `global_normalized_debt` (via Merkle sum root)
- Liquidation events (forced disclosure when a SAFE is liquidated)
- All system parameter, oracle, controller, and auction accounts

**What becomes private:**
- Individual SAFE `collateral` and `normalized_debt`
- SAFE `owner_id` (with stealth addresses)
- SAFE timing fields

**Key constraints and open issues:**
- **Liquidation discovery:** Third-party liquidators fundamentally cannot determine if a private SAFE is underwater without the owner's cooperation. This requires either liveness-based health proofs (§3.4 Option A), a trusted committee (Option B), or a public liquidation threshold buffer (Option C). None of these are zero-cost — each trades privacy for protocol safety.
- **Throughput:** Merkle sum tree updates for debt ceiling enforcement must be sequential. Under high load, debt generation transactions become a bottleneck.
- **Forced disclosure at liquidation:** When a private SAFE is liquidated, its contents are revealed. Privacy is conditional on solvency — users lose privacy precisely when they're most vulnerable.
- **Proof freshness:** Since `accumulated_rate` changes with every stability fee accrual, private SAFEs must re-prove health against the latest rate. Stale proofs could hide insolvency.

**Estimated complexity:** Moderate. The Collateral Ratio Proof circuit is straightforward (fixed-point arithmetic in a SNARK). The Merkle sum tree for debt ceiling enforcement is the harder component — it requires careful concurrency design and adds a global state dependency. Comparable to what Penumbra or Aztec Connect implemented for shielded DeFi, though narrower in scope.

**Assessment:** Technically feasible with known ZK primitives. The main risk is not the ZK circuits themselves but the interaction between private SAFEs and the liquidation mechanism. The forced disclosure requirement means privacy degrades under exactly the conditions where users most want it. Whether this trade-off is acceptable depends on the threat model — if the goal is hiding position sizes from competitors during normal operation, it works. If the goal is full privacy under all conditions, it doesn't.

### 7.3 Full Privacy: Shielded Pools and Private Minting

**What it means:** Collateral deposits, SAFE positions, and LSC minting are all hidden. Users interact with the protocol through ZK proofs at every step.

**Required infrastructure:**
1. Shielded pool for LOGOS collateral (replaces `CollateralVaultAccount` with a Pedersen commitment tree)
2. Private LSC minting (requires Token Program support for ZK-based note commitments)
3. UTXO-like model for LSC balances (fundamental change to the Token Program)
4. Nullifier-based SAFE lifecycle (§6.3)
5. Epoch-based batch processing for debt generation (to resolve Merkle sum tree concurrency)
6. Trusted liquidator mechanism or cooperative forced disclosure for private liquidations

**Estimated complexity:** Very high. This is comparable in scope to building Zcash's Sapling protocol or Aztec Protocol's rollup — systems that required years of dedicated cryptographic research and multiple security audits. Key differences from those projects:
- LSC has *stateful* positions (SAFEs with changing debt), not just transfers — this is harder than shielded transfers alone
- The liquidation mechanism requires third-party access to private state, which shielded transfer protocols don't face
- Epoch-based batching for debt generation may impose unacceptable latency for users

**Open research questions (no known production-ready solutions):**
- Can private liquidations be made trustless without trusted hardware or committees?
- How to handle sequential debt generation at scale without catastrophic throughput degradation?
- Can the Merkle sum tree root be updated in a parallelism-safe way under adversarial conditions?
- How to prevent timing-based deanonymization when only one SAFE is generating debt in a given epoch?

**Assessment:** Not feasible with current ZK infrastructure without a dedicated, multi-year research effort. The fundamental tension is between privacy and the liquidation mechanism — no existing DeFi protocol has solved trustless private liquidations at scale. This level of privacy should be treated as an open research direction, not an engineering project with predictable scope.

### 7.4 Summary of Feasibility

| Level | Technical Feasibility | ZK Infrastructure Cost | Liquidation Compatibility | Privacy Guarantee |
|---|---|---|---|---|
| Baseline (public + pseudonymous) | Trivial | None | Full | Pseudonymity only |
| Partial (private SAFEs + ZK proofs) | Feasible with known primitives | Moderate (2-4 circuits, Merkle sum tree) | Degraded (forced disclosure at liquidation) | Position sizes hidden during normal operation |
| Full (shielded pools + private minting) | Open research problem | Very high (comparable to Aztec/Zcash) | Unsolved (no trustless mechanism) | Strong, but breaks down at liquidation |

**Core finding:** The liquidation mechanism is the binding constraint on LSC privacy. Every level of privacy beyond pseudonymity must compromise on liquidation transparency. The more private the system, the harder it is for third parties to enforce solvency — and solvency enforcement is what makes a stablecoin stable.

---

## 8. References

| Spec | Relationship to This Document |
|---|---|
| `02-account-schemas.md` | Defines all account types analyzed in §2 |
| `10-privacy-considerations.md` | High-level privacy survey; this doc provides the mechanical depth |
| `00-overview.md` | System overview; glossary of terms used here |
| `01-system-architecture.md` | PDA derivation and ChainedCall model relevant to private SAFE PDA design |
| `13-rai-to-lez-mapping.md` | RAI's original design had no privacy; all LEZ privacy features are novel additions |
