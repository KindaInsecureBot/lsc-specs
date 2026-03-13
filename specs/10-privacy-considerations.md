# LSC — Privacy Considerations

## Overview

LEZ supports both **public** accounts (state visible on-chain) and **private** accounts (state committed via ZK proof, only the owner knows the contents). Programs can run publicly (on-chain verifiable) or privately (ZK-proven execution).

This document analyzes which LSC operations can be private, which must be public, and what privacy properties the system provides.

---

## 1. Privacy Model in LEZ

**Public accounts:** Data is readable by anyone on-chain. State transitions are visible.

**Private accounts:** Data is ZK-committed. The account's existence is known, but its contents are hidden. Only the owner (who holds the secret key or nullifier) can prove ownership and modify.

**Hybrid execution:** A transaction can mix public and private account reads. A private account can be included in a public transaction — the program reads its committed hash and verifies a zero-knowledge proof of its contents.

---

## 2. Operations Analysis

### 2.1 Operations That MUST Be Public

The following operations involve system-wide state that must be visible for the protocol to function correctly:

| Operation | Reason Public Required |
|---|---|
| `OracleProgram::SubmitPrice` | Price feeds must be publicly auditable; all users need to read oracle state |
| `OracleProgram::UpdateMedian` | Median computation requires reading multiple public feeds |
| `PIController::UpdateRate` | Rate changes affect all SAFE owners; must be auditable |
| `OracleRelayer::UpdateRedemptionPrice` | Redemption price read by all programs; must be public |
| `TaxCollector::AccrueStabilityFee` | Updates accumulated_rate, which all SAFEs depend on |
| `LiquidationEngine::LiquidateSafe` | Liquidation must be publicly observable for auditing |
| `CollateralAuctionHouse::BuyCollateral` | Auction mechanics require public bid visibility |
| `AccountingEngine::AuctionSurplus/Debt` | System health auctions must be transparent |
| `GlobalSettlement::TriggerSettlement` | Emergency actions must be publicly visible |
| `CollateralTypeAccount` (all fields) | Safety ratio, debt ceiling, accumulated_rate — all programs read these |
| `SystemParamsAccount` | Global configuration; must be readable by all programs |

### 2.2 Operations That CAN Be Private

The following operations involve only the SAFE owner's individual position:

| Operation | Privacy Status | Notes |
|---|---|---|
| `OpenSafe` | Can be private | SAFE existence and owner hidden |
| `DepositCollateral` | Can be private | Collateral amount hidden |
| `WithdrawCollateral` | Can be private | Amount and recipient hidden |
| `GenerateDebt` | Can be private* | Debt amount hidden |
| `RepayDebt` | Can be private* | Payment amount hidden |
| `TransferSafeOwnership` | Can be private | New owner hidden |
| SAFE position (collateral, debt) | Can be private | Individual position confidential |

*With important caveats (see §3).

---

## 3. Privacy Constraints and Caveats

### 3.1 Collateral Vault Visibility

Even if a SAFE is private, **collateral deposited into the vault is public**: the vault account is a public Token Program holding, and its balance is observable. This reveals the total collateral locked per collateral type, but not how it's distributed across SAFEs.

**Mitigation:** Use a shielded pool model where individual deposits are not attributed to specific SAFEs. This requires additional ZK infrastructure (commitment tree, nullifiers) beyond v1 scope.

### 3.2 Token Transfers Are Public

Token Program transfers emit public state changes. When a user deposits LOGOS or receives LSC:
- The source and destination holding account balances change publicly
- If the user's LOGOS holding account is linked to their identity, their activity is visible

**Mitigation (v1):** Not mitigated. Users who want privacy should use intermediate accounts or a mixer (out of scope for v1).

### 3.3 Debt Generation and the Debt Ceiling

If SAFEs are private, the `GenerateDebt` instruction must **prove** that:
1. The collateral ratio is above the safety ratio (using private inputs)
2. The new debt does not exceed the debt ceiling (requires knowing global debt)

The global debt ceiling check requires reading `collateral_type.global_normalized_debt`, which is public. The ZK proof must show that the private debt addition keeps the total within bounds. This is feasible but requires a global accumulator (Merkle sum tree or similar).

**v1 recommendation:** Do not support private debt generation in v1. All `GenerateDebt` calls are public. Private SAFE positions can be explored in v2.

### 3.4 Liquidation and Private SAFEs

If a SAFE is private, liquidators cannot determine if it is undercollateralized without the owner's cooperation. This **breaks the liquidation mechanism**: liquidators need to see `safe.collateral` and `safe.normalized_debt` to check the collateral ratio.

**Options:**
1. **Public-only SAFEs (v1 recommendation):** All SAFEs are public. No privacy for individual positions.
2. **Voluntary disclosure:** SAFE owners can voluntarily publish a ZK proof revealing their collateral ratio exceeds a threshold. This proves they are not underwater without revealing exact amounts.
3. **Forced disclosure via oracle:** A separate "health oracle" program allows SAFE owners to submit periodic ZK proofs of solvency. If no proof is submitted within a period, the SAFE is assumed underwater and can be liquidated by anyone (with the SAFE being forced public at that point).

**v1 recommendation:** Implement option 1 (public SAFEs). Note option 3 as a future enhancement.

### 3.5 Oracle Privacy

Oracle keepers submit prices publicly. There is no privacy for oracle data. Oracle prices are:
- Submitted publicly by keepers
- Visible to all participants
- Required to be public for the PI controller and liquidation engine to function

### 3.6 Auction Privacy

Auctions (CollateralAuction, SurplusAuction, DebtAuction) must be public:
- Bidders need to see current bids to make competitive offers
- Auction outcomes must be publicly verifiable
- Anti-front-running measures (commit-reveal) are possible but add complexity; out of scope for v1

---

## 4. What LSC Conceals (Even in Public Mode)

Even with fully public SAFEs, LSC provides some privacy properties:

1. **No identity linkage by default:** Account IDs are not inherently linked to real-world identities. Users can create multiple accounts.
2. **Pseudonymous SAFE ownership:** SAFE ownership is tied to account IDs, not names.
3. **LSC fungibility:** All LSC tokens are identical; there is no "tainted" LSC traceable to specific liquidations.

---

## 5. Privacy Architecture Recommendations for v2

For a privacy-enhanced v2, consider:

### 5.1 Private SAFEs with ZK Health Proofs

```
PrivateSafeAccount {
    commitment: [u8; 32],  // Hash of (collateral, debt, salt)
    // Actual values hidden
}

// SAFE owner periodically submits:
ZKProof {
    // Proves: committed_collateral / committed_debt >= safety_ratio
    // Without revealing collateral or debt amounts
    statement: "CR >= 1.5",
    proof: [...],
}
```

If no health proof is submitted within `health_proof_period` (e.g., 24 hours), the SAFE can be forced-liquidated by revealing its commitment preimage (requires the SAFE owner's cooperation or a dedicated ZK liquidator circuit).

### 5.2 Shielded Collateral Pool

Instead of individual vault accounts per SAFE, use a single shielded pool:
- LOGOS deposits use a commitment scheme (Pedersen commitments or UTXOs)
- SAFEs are commitments to (collateral_note, debt_note)
- Debt generation creates a new note; repayment nullifies the debt note

This requires:
- Merkle tree for note commitments
- Nullifier set for spent notes
- ZK circuits for deposit, borrow, repay, liquidate

### 5.3 Private PI Controller Updates

The PI controller reads the LSC market price (public by nature) and updates the redemption rate (public by necessity). Privacy here is not meaningful — the rate must be public for SAFE owners to know their debt cost.

---

## 6. Oracle Keeper Privacy

Oracle keepers are **not** anonymous by default:
- Each feed has a designated `keeper_id`
- Keeper account IDs are public
- Price submissions are attributable to specific keepers

For keeper privacy, consider:
- Rotating keeper keys (each submission uses a different derived key)
- Threshold oracle (k-of-n keepers sign collectively, ZK proves threshold met without revealing who signed)

---

## 7. Summary Table

| Component | Public (v1) | Can Be Private | Notes |
|---|---|---|---|
| SAFE existence | Yes | Possible in v2 | |
| SAFE collateral amount | Yes | Possible in v2 | Requires ZK health proofs |
| SAFE debt amount | Yes | Possible in v2 | Requires ZK ceiling proofs |
| SAFE owner | Yes | Possible | Use stealth addresses |
| CollateralType params | Yes | No | Required for protocol operation |
| SystemParams | Yes | No | — |
| Oracle prices | Yes | No | Required for protocol operation |
| Redemption price/rate | Yes | No | Required for all SAFE ops |
| Accumulated rate | Yes | No | Required for all SAFE ops |
| Liquidations | Yes | No | Must be auditable |
| Auction bids | Yes | Partial | Commit-reveal possible in v2 |
| Token transfers | Yes | Partial | Mixer possible |
| PI controller state | Yes | No | Must be auditable |
| Global settlement | Yes | No | Emergency action, must be visible |
