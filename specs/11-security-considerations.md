# LSC — Security Considerations

## Overview

This document catalogs known attack vectors against the LSC system, the mitigations implemented, and residual risks. Implementors must read this document carefully.

---

## 1. Oracle Manipulation

### 1.1 Price Feed Manipulation

**Attack:** An adversary compromises (or is) one or more oracle keepers and submits a false price.

**Impact:**
- False high LOGOS price → undercollateralized SAFEs appear healthy → system accrues bad debt when they're liquidated at real prices
- False low LOGOS price → overcollateralized SAFEs are falsely liquidated

**Mitigations:**
1. **Median oracle:** Requires a majority of keepers to collude. With 3 feeds and `min_feeds = 3`, all three must be compromised.
2. **Price safety margin:** Liquidation uses `price * safety_margin` (e.g., 99.5%), reducing the advantage of a marginally manipulated price.
3. **Timestamp monotonicity:** Keepers cannot submit old prices; price must be more recent than the last submission.
4. **Max jump per update:** Maximum 2-hour timestamp jump prevents teleporting timestamps far into the future.
5. **Keeper authorization:** Only governance-designated keepers can submit prices; permissioned set reduces attack surface.

**Residual risk:** If governance itself is compromised, keepers can be replaced with malicious ones. Mitigation: use a timelock on keeper changes (v2 recommendation).

### 1.2 AMM Flash Loan Price Manipulation

**Attack:** An adversary takes a large flash loan (or LEZ equivalent), manipulates the AMM pool's spot price, submits the manipulated price as an oracle update, then exploits the system.

**Impact:** If LSC market price is artificially inflated, the PI controller reduces the redemption rate (thinking LSC is overpriced), making debt cheaper, allowing cheap debt generation at the expense of long-term stability.

**Mitigations:**
1. **Keeper-submitted prices:** Prices are submitted by keepers, not read on-chain directly. Keepers can implement TWAP or sanity checks before submission.
2. **Median across keepers:** Multiple independent keepers must all submit manipulated prices simultaneously.
3. **PI controller noise barrier:** Small price deviations (< `noise_barrier`) are ignored.
4. **PI controller bounds:** Redemption rate is bounded; manipulation cannot move rate beyond `[lower_bound, upper_bound]`.
5. **Update delay:** PI controller updates at most once per `update_delay` (e.g., 1 hour); manipulation window is limited.

**LEZ Note:** LEZ may not have flash loans in the same form as Ethereum (no single-transaction borrow-use-repay without composability). Verify with LEZ team if atomic flash-loan-like patterns are possible.

**Residual risk:** If all keepers read the AMM at the same time and submit simultaneously, a coordinated manipulation could affect one update cycle.

### 1.3 Stale Oracle

**Attack:** Keepers go offline. Oracle becomes stale. Programs continue using outdated prices.

**Impact:** Liquidations at stale prices may over- or under-penalize SAFE owners.

**Mitigations:**
1. **`max_feed_age` parameter:** Oracle is marked invalid if feeds are too old.
2. **`UpdateMedian` validity check:** `median_oracle.valid = false` if `valid_feed_count < min_feeds_for_median`.
3. **Programs check `oracle.valid`:** All programs assert `oracle.valid == true` before accepting prices.

**Residual risk:** If keepers go offline simultaneously (e.g., outage), operations halt until they return. This is a liveness risk, not a safety risk.

---

## 2. Liquidation Attacks

### 2.1 Dust SAFEs

**Attack:** Create many SAFEs with debt at exactly the `debt_floor`. These SAFEs are expensive to liquidate (gas costs exceed profit) but technically valid, potentially clogging the system.

**Mitigation:** `debt_floor` is set high enough that liquidating any SAFE is profitable. The liquidation penalty (e.g., 13%) on even minimum-debt SAFEs must exceed the cost of the liquidation transaction.

**Parameter guidance:** `debt_floor` should be set such that `debt_floor * liquidation_penalty > 2 * transaction_cost` in LSC terms.

### 2.2 Liquidation Front-Running

**Attack:** A liquidator observes an undercollateralized SAFE and is outbid by another liquidator (or a bot) who submits the same transaction with higher fees.

**Impact:** Legitimate liquidators lose the liquidation opportunity; not a safety risk, just MEV competition.

**Mitigation (v1):** None. First-come, first-served. Liquidators compete on transaction ordering. This is acceptable — it ensures liquidations happen promptly.

### 2.3 Griefing via Circuit Breaker

**Attack:** An adversary triggers many small liquidations to fill the `on_auction_system_coin_limit` circuit breaker, preventing legitimate large liquidations.

**Mitigation:** Set `on_auction_system_coin_limit` large enough to handle realistic liquidation scenarios. Monitor the parameter and adjust via governance.

### 2.4 Auction Manipulation

**Attack in CollateralAuction:** An adversary repeatedly bids minimum amounts, keeping the auction alive without actually settling it, while oracle prices change against them.

**Mitigation:** Auctions have a TTL (`auction_ttl`). After expiry, anyone can restart with updated prices. Fixed-discount auctions don't have a bidding war — first buyer wins; there's no incentive for adversarial bid behavior.

---

## 3. PI Controller Attacks

### 3.1 Integral Windup

**Attack:** Oracle failure or sustained market dislocation causes the error integral to wind up to extreme values, resulting in extreme redemption rates when the oracle recovers.

**Mitigation:**
1. **Integral leak:** The `per_second_cumulative_leak` factor continuously decays the integral. After oracle recovery, the integral resets naturally.
2. **Rate bounds:** Redemption rate is clamped to `[lower_bound, upper_bound]`; extreme integral values are absorbed by clamping.
3. **Emergency reset:** Governance can call `PIController::ResetIntegral` to zero the integral in extreme cases.

### 3.2 Rate Update Timing Attack

**Attack:** An adversary calls `PIController::UpdateRate` at a strategically unfavorable moment (e.g., when market price is transiently dislodged by a large trade).

**Mitigation:**
1. **`update_delay`:** Minimum time between updates; adversary cannot exploit momentary price spikes.
2. **`noise_barrier`:** Small errors are ignored.
3. **Multiple oracle keepers:** Momentary AMM price dislocations are averaged by multiple independent keepers.

---

## 4. Account Confusion Attacks

### 4.1 Wrong Account Type

**Attack:** An adversary passes a malicious account with the same ID as an expected account but with different `account_type` or data, tricking a program into misinterpreting it.

**Mitigation:** Every program checks `account.account_type` at the beginning of each instruction handler. If the type does not match the expected value, the instruction fails with `InvalidAccountType`.

### 4.2 PDA Derivation Collision

**Attack:** Two different inputs produce the same PDA. An adversary creates an account under one context and reuses it in another.

**Mitigation:** Each PDA uses a unique seed prefix (e.g., `b"safe"`, `b"vault"`, etc.) combined with unique per-account inputs (owner_id, nonce). The probability of collision is negligible for a 32-byte PDA space.

### 4.3 Account ID Spoofing

**Attack:** An adversary passes an account that looks like a legitimate system account (e.g., oracle_relayer_params_id) but is actually a crafted account with fake data.

**Mitigation:**
1. Programs check that the account ID matches the stored ID in the parent account (e.g., `system_params.oracle_relayer_params_id == passed_account.id`).
2. Programs check `account.account_type` discriminator.
3. PDA accounts: programs re-derive the expected PDA and verify it matches the passed account's ID.

---

## 5. Reentrancy and Chained Call Attacks

### 5.1 Chained Call Ordering

**Attack:** A malicious program in the chained call sequence reads stale intermediate state from a parent program (before the parent has committed its state changes).

**LEZ Model:** In LEZ, programs output `post_states` which are applied atomically. The chained call receives the updated pre-states. Chained calls are sequential; each sees the committed output of its parent. This is NOT like Ethereum reentrancy.

**Mitigation:** This is a non-issue by LEZ design, but implementors should verify with the LEZ team that post-state outputs from parent programs are visible as pre-states in chained calls.

### 5.2 Chained Call Limit Exhaustion

**Attack:** A complex user operation triggers many chained calls, hitting the LEZ limit of 10 per execution. This could cause a legitimate operation to fail.

**Mitigation:** Keep chained call chains as short as possible. The deepest chain in LSC is the liquidation flow (4 calls). Review new features against the limit before adding them.

---

## 6. Governance Attacks

### 6.1 Malicious Parameter Update

**Attack:** Governance (or a compromised governance account) sets malicious parameters:
- `safety_ratio = 1` (100%) → all SAFEs immediately liquidatable
- `debt_ceiling = 0` → all new debt blocked
- `stability_fee = max_value` → astronomical stability fees

**Mitigation (v1):** Governance is a single multisig. The security is as strong as the multisig.

**Mitigation (v2 recommendations):**
- Add a **timelock** on parameter changes (e.g., 48-hour delay between proposal and execution)
- Add **parameter bounds** checked in `UpdateParams` instructions (e.g., `safety_ratio <= 5 * RAY`)
- Move governance to an on-chain voting program with token-weighted votes

### 6.2 Global Settlement Abuse

**Attack:** Governance abuses `TriggerSettlement` to shut down the system and steal collateral at favorable prices.

**Mitigation:** Settlement freezes prices at oracle prices at the time of trigger. Governance cannot set arbitrary prices. Users can redeem at the frozen oracle price.

**Residual risk:** If governance triggers settlement when oracle price is manipulated, users may receive less than fair value.

---

## 7. Economic Attacks

### 7.1 SAFE Speculation

**Attack:** When LSC redemption rate is highly positive (market price below redemption price), SAFE owners can profitably open and close SAFEs to take advantage of the rate, without genuine economic need.

**Impact:** This is actually desired behavior — it's how the PI controller works. SAFE owners who generate LSC and sell it when rate is high are arbitrageurs that help correct the price.

### 7.2 Deflationary Spiral

**Attack / Risk:** If collateral (LOGOS) price drops rapidly:
1. Many SAFEs become undercollateralized simultaneously
2. Liquidation engine seizes collateral and sells it for LSC
3. Selling LOGOS depresses LOGOS price further
4. More SAFEs become undercollateralized (cascade)

**Mitigation:**
1. **Safety ratio buffer:** `safety_ratio > liquidation_ratio` gives SAFEs headroom before liquidation.
2. **Liquidation limit (`on_auction_system_coin_limit`):** Limits total value that can be auctioned at once, preventing liquidation flood.
3. **Collateral auction discount:** Buyers are incentivized to absorb collateral quickly due to the discount.
4. **PI controller:** Rising redemption rate makes LSC more attractive to hold, increasing demand and price, which helps the system recover.

**Residual risk:** A sufficiently large and rapid collateral drop (e.g., > 50% in 1 hour) could outpace these mechanisms. This is an existential risk for any collateralized stablecoin.

### 7.3 LSC Bank Run

**Attack / Risk:** If confidence in LSC falls, all holders simultaneously try to close SAFEs and sell LSC.

**Impact:** LSC price drops, PI controller raises redemption rate (making debt more expensive), which further incentivizes SAFE closure, potentially in a vicious cycle.

**Mitigation:** The reflexive mechanism is designed to be self-stabilizing: as price drops, rate rises, making it more expensive to exit SAFEs, which reduces selling pressure. This is the core innovation of the RAI model.

---

## 8. Implementation Security Checklist

For implementors, the following checks must be verified in code review:

### Arithmetic
- [ ] All intermediate calculations use 256-bit arithmetic (u256) to prevent overflow
- [ ] All divisions check for divisor != 0
- [ ] Ray/Wad/Rad conversions are correct at every boundary
- [ ] Signed i128 operations handle negative numbers correctly (integral, gains)

### Authorization
- [ ] Every `is_authorized` check is present and verified
- [ ] PDA re-derivation matches expected seeds
- [ ] `account_type` discriminator is checked on every passed account
- [ ] Governance-only instructions check governance_id match

### State Transitions
- [ ] Pre-state validation happens before any state mutation
- [ ] Chained calls are emitted only after all local state is finalized
- [ ] No partial state updates (all mutations or none, within the program's own accounts)

### Chained Calls
- [ ] Total chained call depth <= 10 per transaction
- [ ] Chained call account lists correctly propagate authorizations
- [ ] Chained calls cannot be replayed or double-executed

### Time
- [ ] No operation uses a hardcoded timestamp
- [ ] All time reads go through oracle accounts
- [ ] Timestamp monotonicity is enforced
- [ ] Stale oracle detection is present on all time-sensitive operations

### Settlement
- [ ] `settlement_active` check is in `GenerateDebt`, `OpenSafe`, `StartAuction` (collateral)
- [ ] Settlement is truly one-way (no un-settle function)

---

## 9. Known Issues from RAI (Apply to LSC)

The following known issues from the RAI/GEB codebase may apply to LSC and should be addressed:

### 9.1 Dusty SAFE Liquidation Issue

**Issue (from RAI):** SAFEs with very small debt (below dust threshold) that become undercollateralized may not be profitable to liquidate. If `debt_floor` is enforced at creation but not continuously, a SAFE can drift below floor via fee accrual.

**Mitigation for LSC:**
- `debt_floor` check applies at both `GenerateDebt` and after stability fee accrual
- Governance can lower `debt_floor` if needed
- Provide a way to liquidate dust SAFEs at reduced cost (or no liquidation penalty)

### 9.2 Oracle-Dependent Accumulated Rate

**Issue:** If the oracle is stale for a long time and then recovers, `TaxCollector::AccrueStabilityFee` is called with a large `time_elapsed`. The compounded fee can be very large.

**Mitigation for LSC:**
- `TaxCollector` should batch fee accrual if time_elapsed > max_accrual_period
- Or cap the single-call time_elapsed to `max_accrual_period` and require multiple calls for longer gaps
- Recommended: `max_accrual_period = 86400` seconds (1 day)

### 9.3 Redemption Price Precision

**Issue:** `rpow(redemption_rate, dt, RAY)` accumulates rounding errors over time.

**Mitigation:** Use the most precise 256-bit multiplication available. The `rpow` implementation must use proper rounding (round half-up) at each multiplication step.

---

## 10. Bug Bounty Recommendations

For the LSC deployment, the implementors should set up a bug bounty covering:

- Economic attacks that drain the system
- Oracle manipulation that causes incorrect liquidations
- Authorization bypass attacks
- Arithmetic overflow/underflow leading to incorrect state
- Account confusion attacks
- Any mechanism that allows minting LSC without collateral

Recommended bounty scope: all deployed LSC programs and their interactions with the Token Program and AMM Program.
