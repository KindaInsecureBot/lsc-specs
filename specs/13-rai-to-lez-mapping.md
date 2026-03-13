# RAI-to-LEZ Mapping Reference

This document maps every RAI/GEB Solidity construct to its LSC/LEZ equivalent. It is intended as a reference for implementors familiar with RAI/MakerDAO who want to understand how LSC adapts those designs to the Logos Execution Zone (LEZ).

**GEB naming note:** GEB renamed many MakerDAO (MCD) contracts. Both names appear below:
`Vat → SAFEEngine`, `Vow → AccountingEngine`, `Cat → LiquidationEngine`, `Jug → TaxCollector`, `Spot → OracleRelayer`, `End → GlobalSettlement`, `Flap → SurplusAuctionHouse`, `Flop → DebtAuctionHouse`, `Flip → CollateralAuctionHouse`.

---

## 1. Contract → Program Mapping

| RAI/GEB Contract | MCD Equivalent | LSC Program | Notes |
|---|---|---|---|
| `SAFEEngine` | `Vat` | `LSCEngine` | Core accounting; each SAFE is a standalone PDA account instead of a mapping entry |
| `TaxCollector` | `Jug` | `TaxCollector` | Identical stability-fee math (`rpow`); chains to `LSCEngine::UpdateAccumulatedRate` |
| `OracleRelayer` | `Spot` | `OracleRelayer` | Stores redemption price and rate; no Chainlink dependency — price sourced from `OracleProgram` |
| `DSValue` / `OSM` (Delayed Price Feed) | `PipLike` | `OracleProgram` | Keeper-submitted prices aggregated via median; provides canonical timestamps to all programs |
| `PIRateSetter` + `PIController` | *(no MCD equiv.)* | `PIController` | Same PI math with integral leak; reads time from oracle median timestamp, not `block.timestamp` |
| `LiquidationEngine` | `Cat` | `LiquidationEngine` | No `SAFESaviour`; full liquidation only; fixed-discount collateral auction |
| `EnglishCollateralAuctionHouse` / `FixedDiscountCollateralAuctionHouse` | `Flip` | `CollateralAuctionHouse` | LSC uses fixed-discount only (no English/increasing-bid variant in v1) |
| `AccountingEngine` | `Vow` | `AccountingEngine` | **No debt auction**; bad debt socialized or offset by governance recapitalization |
| `SurplusAuctionHouse` | `Flap` | `SurplusAuctionHouse` | Reverse auction; LOGOS burned instead of FLX/MKR |
| `DebtAuctionHouse` | `Flop` | **Removed** | No LOGOS minting; see §6 |
| `GlobalSettlement` | `End` | `GlobalSettlement` | Same 5-phase shutdown; adapted for stateless account model |
| `GebSAFEManager` | `DssCdpManager` | **Not specced** | Users interact with `LSCEngine` directly via deterministic PDAs; no proxy manager needed |
| `CoinSavingsAccount` | `Pot` | **Removed** | No coin savings rate in LSC v1 |
| `StabilityFeeTreasury` | *(no MCD equiv.)* | **Not specced** | Keeper automation handled off-chain in v1 |
| `SAFESaviour` | *(no MCD equiv.)* | **Not specced** | No SAFE insurance hook in v1 |

---

## 2. Function → Instruction Mapping

### SAFEEngine → LSCEngine

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `initializeCollateralType` | `LSCEngine::AddCollateralType` | Takes `token_def_id` (PDA ref) instead of `bytes32`; includes `safety_ratio`, `liquidation_ratio`, `stability_fee` |
| `modifyParameters` (collateral) | `LSCEngine::UpdateCollateralType` | All fields optional (use `Option<u128>`); governance-only |
| `modifySAFECollateralization` | `LSCEngine::ModifySafe` | Single instruction combining collateral delta + debt delta; also decomposed into `DepositCollateral`, `WithdrawCollateral`, `GenerateDebt`, `RepayDebt` for ergonomics |
| *(no direct equiv.)* | `LSCEngine::OpenSafe` | Allocates a new `SafeAccount` PDA; RAI uses mapping entries allocated implicitly |
| `transferSAFECollateralAndDebt` | `LSCEngine::TransferSafeOwnership` | LSC v1 transfers whole SAFE; no partial collateral/debt splits |
| `confiscateSAFECollateralAndDebt` | `LSCEngine::ConfiscateSafe` | Privileged; only callable by `LiquidationEngine` via chained call |
| `updateAccumulatedRate` | `LSCEngine::UpdateAccumulatedRate` | Privileged; only callable by `TaxCollector` via chained call |
| `disableContract` | `LSCEngine::FreezeCollateralType` | Per-collateral freeze (not full engine disable); called by `GlobalSettlement` |
| `addAuthorization` / `removeAuthorization` | Governance multisig manages `SystemParamsAccount` | No `rely`/`deny` per-address model; governance is a single multisig key in v1 |
| `approveSAFEModification` | `LSCEngine::AllowOperator` | Adds `operator_id` to `safe.allowed_operators[4]`; max 4 operators per SAFE |
| *(no equiv.)* | `LSCEngine::CloseSafe` | Explicit close to reclaim account rent |

### TaxCollector → TaxCollector

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `taxSingle(collateralType)` | `TaxCollector::AccrueStabilityFee` | Permissionless keeper call; passes `median_timestamp` from oracle (no `block.timestamp`) |
| `modifyParameters` | `TaxCollector::UpdateParams` | Governance-only |
| *(no equiv.)* | `TaxCollector::Initialize` | One-time setup |

**Chained call:** `TaxCollector → LSCEngine::UpdateAccumulatedRate → TokenProgram::Mint`

### OracleRelayer → OracleRelayer + OracleProgram

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `updateCollateralPrice(collateralType)` | `OracleRelayer::UpdateRedemptionPrice` | Permissionless; uses oracle `median_timestamp` not `block.timestamp` |
| `redemptionPrice()` (view, lazy update) | `OracleRelayer::UpdateRedemptionPrice` | LSC requires explicit keeper call; no lazy state-write on read |
| `updateRedemptionRate(newRate)` | `OracleRelayer::UpdateRedemptionRate` | Privileged; only `PIController` via chained call |
| `modifyParameters` | `OracleRelayer::UpdateParams` | Governance-only |
| `priceFeedAddress` (OSM/DSValue) | `OracleProgram::SubmitPrice` + `UpdateMedian` | Keepers submit raw prices; median oracle aggregates; no single trusted feed |

### PIController → PIController

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `updateRate(feeReceiver)` | `PIController::UpdateRate` | Permissionless; respects `update_delay`; reads market price from `OracleProgram` |
| `modifyParameters` | `PIController::UpdateParams` | Governance-only |
| `resetIntegral` | `PIController::ResetIntegral` | Governance emergency reset |
| *(no equiv.)* | `PIController::Initialize` | One-time setup |

**Chained call:** `PIController → OracleRelayer::UpdateRedemptionRate`

### LiquidationEngine → LiquidationEngine

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `liquidateSAFE(collateralType, safe)` | `LiquidationEngine::LiquidateSafe` | Permissionless; full liquidation only (no partial); no `SAFESaviour` hook |
| `protectSAFE` | **Removed** | No SAFE insurance |
| `modifyParameters` | `LiquidationEngine::UpdateParams` | Governance-only |
| *(no equiv.)* | `LiquidationEngine::RemoveLiquidation` | Cleans up a completed `LiquidationAccount` PDA |

**Chained calls:** `LiquidationEngine → LSCEngine::ConfiscateSafe → TokenProgram::Transfer → AccountingEngine::PushDebt → CollateralAuctionHouse::StartAuction`

### CollateralAuctionHouse → CollateralAuctionHouse

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `startAuction` | `CollateralAuctionHouse::StartAuction` | Privileged; only `LiquidationEngine` via chained call |
| `buyCollateral(id, wad)` | `CollateralAuctionHouse::BuyCollateral` | Fixed-discount only; first-come-first-served; no bidding rounds |
| `settleAuction(id)` | `CollateralAuctionHouse::SettleAuction` | Permissionless cleanup |
| `restartAuction(id)` | `CollateralAuctionHouse::RestartAuction` | Increases discount after TTL expires |
| `terminateAuctionPrematurely(id)` | `CollateralAuctionHouse::TerminateAuction` | Called by `GlobalSettlement` |

### AccountingEngine → AccountingEngine

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `pushDebtToQueue(debt)` | `AccountingEngine::PushDebt` | Privileged; only `LiquidationEngine`; pushes to `SystemDebtQueueAccount` circular buffer |
| `settleDebt(rad)` | `AccountingEngine::SettleDebt` | Permissionless; burns surplus LSC to cover queued debt |
| `auctionSurplus()` | `AccountingEngine::AuctionSurplus` | Permissionless; requires `net_surplus > surplus_buffer + surplus_auction_amount` |
| `auctionDebt()` | **Removed** | No debt auctions; see `RecapitalizeWithLSC`, `RecapitalizeWithLOGOS` |
| `modifyParameters` | `AccountingEngine::UpdateParams` | Governance-only |
| *(no equiv.)* | `AccountingEngine::RecapitalizeWithLSC` | Governance injects external LSC to cover bad debt |
| *(no equiv.)* | `AccountingEngine::RecapitalizeWithLOGOS` | Governance swaps treasury LOGOS on AMM to obtain LSC |

### SurplusAuctionHouse → SurplusAuctionHouse

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `startAuction(amountToSell, initialBid)` | `SurplusAuctionHouse::StartAuction` | Privileged; only `AccountingEngine` via chained call |
| `increaseBidSize(id, amountToBuy, rad)` | `SurplusAuctionHouse::IncreaseBidSize` | Same reverse-auction mechanic; LOGOS bids burned instead of FLX |
| `settleAuction(id)` | `SurplusAuctionHouse::SettleAuction` | Permissionless; winner receives LSC; LOGOS burned |
| `restartAuction(id)` | `SurplusAuctionHouse::RestartAuction` | Permissionless after `total_auction_length` expires |

### GlobalSettlement → GlobalSettlement

| RAI Function | LSC Instruction | Differences |
|---|---|---|
| `freeze()` | `GlobalSettlement::TriggerSettlement` | Sets `settlement_active`; snapshots final redemption price; governance-only |
| `freezeCollateralType(collateralType)` | `GlobalSettlement::FreezeCollateralType` | Permissionless post-trigger; chains to `LSCEngine::FreezeCollateralType` |
| `processSAFE(collateralType, safe)` | `GlobalSettlement::ProcessSAFEs` | Computes surplus collateral; accounts for undercollateralized SAFEs |
| `freeCollateral(collateralType)` | `GlobalSettlement::RedeemCollateral` | SAFE owners withdraw net collateral after debt offset |
| `setOutstandingCoinSupply()` | `GlobalSettlement::SetFinalRedemptionRate` | After `safe_redemption_period` (3 days); sets `collateral_per_lsc` |
| `redeemCollateral(collateralType, coinsAmount)` | `GlobalSettlement::RedeemLSC` | LSC holders burn LSC for pro-rata collateral share |

---

## 3. Storage → Account Mapping

### SAFEEngine Storage → LSCEngine Accounts

| RAI Storage (Solidity) | LSC Account / Field | Notes |
|---|---|---|
| `mapping(bytes32 => CollateralType) collateralTypes` | One `CollateralTypeAccount` PDA per collateral type | Seed: `b"collateral_type" \|\| token_def_id[32]`; owned by `LSC_ENGINE_ID` |
| `CollateralType.accumulatedRate` | `CollateralTypeAccount.accumulated_rate: u128` (Ray) | Same meaning; updated by `TaxCollector` |
| `CollateralType.safetyPrice` | Derived at instruction time from `OracleRelayerParamsAccount.redemption_price` + oracle feed | Not stored; computed inline |
| `CollateralType.liquidationPrice` | `CollateralTypeAccount.liquidation_ratio: u128` (Ray) + oracle feed | Separate from safety ratio, same as GEB dual-price model |
| `CollateralType.debtCeiling` | `CollateralTypeAccount.debt_ceiling: u128` (Rad) | |
| `CollateralType.debtFloor` | `CollateralTypeAccount.debt_floor: u128` (Rad) | Minimum debt to open/keep SAFE |
| `mapping(bytes32 => mapping(address => SAFE)) safes` | One `SafeAccount` PDA per (owner, nonce) | Seed: `b"safe" \|\| owner_id[32] \|\| nonce[8]` |
| `SAFE.lockedCollateral` | `SafeAccount.collateral: u128` (Wad) | |
| `SAFE.generatedDebt` | `SafeAccount.normalized_debt: u128` (Wad) | Normalized; actual debt = `normalized_debt * accumulated_rate` |
| `mapping(address => mapping(address => uint)) tokenCollateral` | Token Program `FungibleHoldingAccount` PDAs | Collateral held in `CollateralVaultAccount` (Token Program holding with PDA owned by LSCEngine) |
| `mapping(address => uint) coinBalance` | Token Program `FungibleHoldingAccount` for LSC | No internal coin ledger; real LSC token balances only |
| `mapping(address => uint) debtBalance` | `SystemDebtQueueAccount.queued_debt` + `GlobalDebtAccount.total_debt` | Aggregated, not per-address |
| `uint globalDebt` | `GlobalDebtAccount.total_debt: u128` (Rad) | |
| `uint globalUnbackedDebt` | `SystemDebtQueueAccount.queued_debt: u128` (Rad) | Pending bad debt awaiting settlement |
| `uint globalDebtCeiling` | `SystemParamsAccount.global_debt_ceiling: u128` (Rad) | |
| `bool contractEnabled` | `SystemParamsAccount.settlement_active: bool` | |
| `mapping(address => uint) approvedModifications` | `SafeAccount.allowed_operators: [[u8;32]; 4]` | Max 4; per-SAFE operator list |

### TaxCollector Storage → TaxCollectorParamsAccount

| RAI Storage | LSC Account / Field | Notes |
|---|---|---|
| `mapping(bytes32 => CollateralType) collateralTypes` | Per-collateral fields in `CollateralTypeAccount` | Stability fee stored in `CollateralTypeAccount.stability_fee` |
| `uint globalStabilityFee` | `TaxCollectorParamsAccount.global_stability_fee: u128` (Ray) | Base rate applied on top of per-collateral rate |
| `uint latestUpdateTime` (per collateral) | `CollateralTypeAccount.last_accrual_time: u64` | Unix timestamp from oracle; updated each `AccrueStabilityFee` call |

### OracleRelayer Storage → OracleRelayerParamsAccount

| RAI Storage | LSC Account / Field | Notes |
|---|---|---|
| `uint redemptionPrice` | `OracleRelayerParamsAccount.redemption_price: u128` (Ray) | |
| `uint redemptionRate` | `OracleRelayerParamsAccount.redemption_rate: u128` (Ray) | Set by `PIController`; applied by `UpdateRedemptionPrice` |
| `uint lastUpdateTime` | `OracleRelayerParamsAccount.last_redemption_price_update: u64` | Oracle timestamp, not `block.timestamp` |
| `mapping(bytes32 => Feed) orcls` | `OracleFeedAccount` PDAs (one per keeper per collateral) | Aggregated into `MedianOracleAccount` |

### PIController Storage → PIControllerStateAccount

| RAI Storage | LSC Account / Field | Notes |
|---|---|---|
| `int kp` | `PIControllerStateAccount.kp: i128` | Proportional gain (signed Ray) |
| `int ki` | `PIControllerStateAccount.ki: i128` | Integral gain (signed Ray) |
| `int errorIntegral` | `PIControllerStateAccount.error_integral: i128` | Accumulated error with leak |
| `uint lastUpdateTime` | `PIControllerStateAccount.last_update_time: u64` | Oracle timestamp |
| `int perSecondCumulativeLeak` | `PIControllerStateAccount.per_second_cumulative_leak: u128` (Ray) | ≈ 0.99999992/s |

### AccountingEngine Storage → AccountingEngine Accounts

| RAI Storage | LSC Account / Field | Notes |
|---|---|---|
| `mapping(uint => uint) debtQueue` | `SystemDebtQueueAccount.entries: [DebtQueueEntry; 256]` | Circular buffer; max 256 entries |
| `uint totalQueuedDebt` | `SystemDebtQueueAccount.queued_debt: u128` (Rad) | |
| `uint totalOnAuctionDebt` | Tracked in `LiquidationEngineParamsAccount.on_auction_system_coin_limit` | Cap, not running total |
| `uint surplusBuffer` | `AccountingEngineParamsAccount.surplus_buffer: u128` (Rad) | |
| `uint surplusAuctionDelay` | `AccountingEngineParamsAccount.surplus_auction_delay: u64` | Seconds |
| `address surplusAuctionHouse` | `AccountingEngineParamsAccount.surplus_auction_house_id: [u8;32]` | Program ID reference |
| `address debtAuctionHouse` | **Removed** | |

### GlobalSettlement Storage → GlobalSettlementStateAccount

| RAI Storage | LSC Account / Field | Notes |
|---|---|---|
| `bool contractEnabled` | Derived from `SystemParamsAccount.settlement_active` | |
| `uint shutdownTime` | `GlobalSettlementStateAccount.shutdown_timestamp: u64` | Oracle timestamp at shutdown |
| `mapping(bytes32 => uint) finalCoinPerCollateral` | `CollateralRedemptionAccount.collateral_per_lsc: u128` | One PDA per collateral type |
| `mapping(bytes32 => uint) collateralTotalDebt` | Computed from collateral type accounts | |
| `mapping(bytes32 => uint) collateralShortfall` | Computed at `ProcessSAFEs` time | |
| `uint outstandingCoinSupply` | LSC token `total_supply` (from Token Program definition) | |
| `uint safeRedemptionPeriod` | `GlobalSettlementStateAccount.safe_redemption_period: u64` | Default 259200s (3 days) |

---

## 4. EVM Concept → LEZ Concept Mapping

| EVM / Solidity | LEZ / LSC | Notes |
|---|---|---|
| `msg.sender` | `is_authorized` flag on accounts | The runtime sets `is_authorized = true` on accounts the caller controls (owns or has PDA authority over) |
| `address` (20 bytes) | `AccountId` (32 bytes, base58) | LEZ uses 32-byte account identifiers throughout |
| `block.timestamp` | Oracle-provided `median_timestamp` | Programs have no clock access; the `OracleProgram::UpdateMedian` output carries a canonical timestamp consumed by all programs |
| `block.number` | Not available | No block number concept in LEZ; time is oracle-derived |
| `mapping(K => V)` | One PDA account per key | Each entry is a standalone account with a deterministic seed; passing the right accounts is the caller's responsibility |
| `require(condition, "msg")` | `assert!()` / return error code | Guest programs use Rust `assert!` or return typed error enums; no revert messages, only numeric error codes |
| `contract storage` | `account.data` (Borsh-serialized) | All state is in account data fields; programs are stateless binaries |
| `events` / `emit` | Account state changes observed by indexer | No event log; off-chain indexers track account diffs to reconstruct history |
| `ERC-20 transfer` | `TokenProgram::Transfer` (chained call) | LSC token ops are chained calls to the Token Program; no internal balance ledger |
| `ERC-20 mint` | `TokenProgram::Mint` (chained call, PDA authorized) | LSCEngine's PDA seed for `lsc_token_def_id` grants mint authority |
| `ERC-20 burn` | `TokenProgram::Burn` (chained call, holder `is_authorized`) | Holder must have `is_authorized = true` |
| `Uniswap swap` | `AMMProgram::Swap` (chained call) | Constant-product AMM; account ordering is strict: `[pool_def, vault_a, vault_b, user_a, user_b]` |
| `internal function call` | Intra-instruction logic in the same guest binary | No equivalent to `private`/`internal` cross-contract calls within one program |
| `external call` / `interface` | Chained call (`ChainedCall` struct) | Max 10 chained calls per top-level execution; caller includes `pda_seeds` for PDA authorization |
| `delegatecall` | Chained call with PDA authority | The callee receives the caller's PDA-derived authorization via `pda_seeds`; no code delegation |
| `reentrancy guard` | Not needed | LEZ execution is atomic and stateless; no shared mutable state between calls |
| `access control modifier` | `is_authorized` check + program ownership | Each instruction verifies the appropriate account has `is_authorized = true` or is owned by the expected program |
| `constructor` | `Initialize` instruction | One-time setup instruction; often guarded by "already initialized" check on `SystemParamsAccount` |
| `selfdestruct` / contract destruction | `CloseSafe` (account reclaim) | Individual accounts can be closed to recover storage; no program-level destruction |
| `try/catch` | Error code propagation | Chained call failures abort the entire execution; no partial recovery |
| `ABIEncode` / `ABIDecode` | Borsh serialization / deserialization | All account data and instruction arguments use Borsh |
| `keccak256` / `CREATE2` | `SHA256` + `compute_pda_seed` | PDA derivation hashes a 96-byte input: 32-byte prefix + 32-byte `program_id` + 32-byte seed |
| `uint256` arithmetic | `u256` intermediate type | All Ray/Wad/Rad multiplications must use 256-bit intermediates to avoid overflow |
| `int256` arithmetic | `i256` intermediate type | PI controller signed arithmetic requires 256-bit signed intermediates |
| `payable` / ETH value | LOGOS token (`FungibleHoldingAccount`) | No native value transfer; all value movement via Token Program |
| `onlyOwner` / `Ownable` | `governance_id` in `SystemParamsAccount` | v1 uses a single governance multisig key; checked against `is_authorized` on the governance account |
| `SafeMath` (overflow checks) | Rust's checked arithmetic (`checked_mul`, etc.) | Rust's default debug mode panics on overflow; release builds must use explicit checked or saturating ops |

---

## 5. Key Architectural Differences

### Stateless Programs vs. Stateful Contracts

RAI contracts hold state in Solidity storage slots. LEZ programs are **stateless binaries** — every program invocation receives all relevant accounts as explicit inputs and outputs. There is no "storage" inside a program. This means:

- Every SAFE is its own `SafeAccount` PDA that must be passed to instructions.
- Authorization cannot be looked up from internal state; it must be derivable from the accounts passed in.
- Programs cannot iterate over all SAFEs — off-chain indexers must enumerate accounts.

### Account-Passing vs. Contract Storage

In RAI, `SAFEEngine.safes[collateral][owner]` is a storage slot accessed directly. In LSC, the caller must pass the `SafeAccount` PDA as an explicit account. The runtime validates account ownership; the program validates the PDA seed to confirm it is the right account for the right owner.

### PDA Authorization vs. msg.sender

RAI uses `msg.sender` checks: `require(msg.sender == liquidationEngine, ...)`. LSC uses PDA-derived authorization:

1. `LiquidationEngine` includes `pda_seeds: [compute_pda_seed(b"liquidation_params")]` in its `ChainedCall`.
2. The LEZ runtime derives `AccountId::from((&LIQUIDATION_ENGINE_ID, &pda_seed))` and sets `is_authorized = true` on the matching account.
3. `LSCEngine::ConfiscateSafe` verifies that `liquidation_engine_params_id.is_authorized == true`.

This replaces whitelist mappings (`rely`/`deny`) entirely — authorization is structural, derived from program identity.

### Chained Calls vs. Internal/External Calls

RAI uses Solidity-to-Solidity calls with no explicit cost limit beyond gas. LEZ enforces a **maximum of 10 chained calls** per top-level execution. This shapes the architecture:

- The deepest call chain in LSC is 4 calls (liquidation flow).
- Programs cannot be composed arbitrarily deeply.
- Each chained call must pass all required accounts explicitly.

### Balance Conservation Rule

LEZ enforces that the **total token supply is conserved** across a transaction. Programs cannot create or destroy tokens outside of explicit Mint/Burn calls to the Token Program. This is structurally enforced by the runtime, unlike RAI where the invariant is maintained by code conventions.

### Oracle-Provided Timestamps

RAI uses `block.timestamp` freely. LSC programs have **no clock access**. The canonical time source is `OracleProgram::MedianOracleAccount.median_timestamp`, which keepers must include when calling any time-sensitive instruction (TaxCollector, PIController, OracleRelayer, GlobalSettlement). This means:

- All time-dependent computations are gated on keeper activity.
- Stale oracles effectively pause the system (no rate accrual, no PI updates).
- Time is oracle-provided, so the oracle must be trusted for liveness.

### No Debt Auctions / No Protocol Token Minting

RAI (and MakerDAO) have a last-resort mechanism: mint protocol tokens (FLX/MKR) and auction them to cover bad debt. LSC **removes this mechanism entirely**. Rationale: minting LOGOS to cover debt dilutes all LOGOS holders and undermines the token's value proposition as collateral. Instead:

- Surplus offsets queued debt first (`AccountingEngine::SettleDebt`).
- Governance can inject LSC directly (`RecapitalizeWithLSC`).
- Governance can sell treasury LOGOS via the AMM (`RecapitalizeWithLOGOS`) — but this uses *existing* LOGOS, not newly minted tokens.
- Unresolved bad debt becomes a protocol liability socialized via redemption price dynamics.

### Dual Collateral Prices (Inherited from GEB, Preserved in LSC)

GEB (unlike MakerDAO) uses two separate prices per collateral type:
- **Safety price** (`safety_ratio`): used for debt generation checks (conservative).
- **Liquidation price** (`liquidation_ratio`): used for liquidation trigger (more lenient, giving owners time to react).

LSC preserves this dual-price model with `safety_ratio` and `liquidation_ratio` as separate fields in `CollateralTypeAccount`.

### Privacy (LEZ-Specific, Not in RAI)

LEZ supports optional privacy via ZK proofs. RAI has no privacy support. In LSC v1, all accounts are public, but the architecture is designed to accommodate private SAFEs in v2, where ZK health proofs would allow liquidation of private SAFEs without revealing their state.

---

## 6. What Was Removed from RAI

### Debt Auctions (`DebtAuctionHouse` / `Flop`)

**Removed.** RAI mints FLX (protocol token) when the surplus is insufficient to cover bad debt. LSC does not mint LOGOS for any reason. Bad debt is handled by:

1. Automatic surplus offset (`SettleDebt`).
2. Governance injection of external LSC (`RecapitalizeWithLSC`).
3. Governance AMM swap of existing treasury LOGOS (`RecapitalizeWithLOGOS`).
4. Socialization via redemption price drift (last resort, implicit).

**Rationale:** LOGOS is the collateral backing LSC. Minting LOGOS under stress dilutes collateral value and can trigger a death spiral (more minting → lower collateral value → more bad debt → more minting). Removing debt auctions eliminates this systemic risk at the cost of requiring adequate governance reserves.

### Coin Savings Rate (`CoinSavingsAccount` / `Pot`)

**Removed.** RAI does not have a coin savings rate (unlike MakerDAO's DSR), but GEB includes a `CoinSavingsAccount` for protocol extensions. LSC omits this entirely in v1.

**Rationale:** The redemption rate already incentivizes LSC holding when positive. An additional savings rate would complicate accounting and introduce a second lever that can conflict with the PI controller.

### SAFE Saviour (`SAFESaviour`)

**Removed.** GEB allows SAFE owners to attach whitelisted "saviour" contracts that can top up collateral or repay debt during liquidation. LSC omits this in v1.

**Rationale:** SAFE saviours add significant complexity (whitelisting, integration surface, flash loan attack vectors). V1 prioritizes simplicity; SAFE owners must monitor their positions directly.

### GebSAFEManager / Proxy Manager

**Not specced.** GEB's `GebSAFEManager` is a convenience proxy allowing users to manage SAFEs without dealing with raw bytes32 identifiers. In LSC, SAFE PDAs are deterministic from `(owner_id, nonce)` — wallets/frontends can compute the SAFE address directly without a proxy contract.

### Stability Fee Treasury (`StabilityFeeTreasury`)

**Not specced.** GEB includes a stability fee treasury that routes a portion of fees to fund oracle keepers automatically. LSC v1 relies on off-chain keeper automation; fee routing to keepers is a v2 consideration.

### Oracle Security Module (OSM / Delayed Price Feed)

**Not specced for v1.** GEB uses an OSM that delays price updates by 1 hour, giving the system time to react to oracle manipulation. LSC v1 substitutes a **collateral price safety margin** (default 99.5%) and **noise barrier** on the PI controller. A full OSM-equivalent delay mechanism is a v2 enhancement.

### Post-Settlement Surplus Auction

**Implicitly removed.** GEB has a `PostSettlementSurplusAuctionHouse` to distribute remaining surplus after global settlement. LSC v1 does not specify this; surplus during settlement is an open implementation question.

### Multi-Address Stability Fee Distribution

**Not specced.** GEB's `TaxCollector` can split stability fees across multiple recipients (stability fee treasury, etc.). LSC v1 routes all fees to the `system_surplus` account only.

---

## Quick-Reference Summary

| RAI/GEB Feature | LSC Status | LSC Equivalent |
|---|---|---|
| SAFEs (CDPs) | ✅ Preserved | `SafeAccount` PDAs, `LSCEngine` |
| Stability fees | ✅ Preserved | `TaxCollector::AccrueStabilityFee` |
| PI controller / reflexive rate | ✅ Preserved | `PIController` |
| Liquidation engine | ✅ Preserved | `LiquidationEngine` |
| Fixed-discount collateral auction | ✅ Preserved | `CollateralAuctionHouse` |
| Surplus auction (LOGOS burned) | ✅ Preserved | `SurplusAuctionHouse` |
| Global settlement | ✅ Preserved | `GlobalSettlement` |
| Dual collateral prices (safety / liquidation) | ✅ Preserved | `safety_ratio` + `liquidation_ratio` in `CollateralTypeAccount` |
| Debt auction (mint protocol token) | ❌ Removed | Governance recapitalization instead |
| Coin savings rate | ❌ Removed | Redemption rate handles incentives |
| SAFE Saviour | ❌ Removed | v2 consideration |
| OSM (price delay) | ❌ Not in v1 | Safety margin + noise barrier |
| Proxy/CDP manager | ❌ Not needed | PDAs are deterministic |
| Stability fee treasury | ❌ Not in v1 | Off-chain keeper automation |
| Privacy | ✅ LEZ-native (v1: all public) | ZK proofs for private SAFEs in v2 |
| `block.timestamp` | ❌ Not available | Oracle `median_timestamp` |
| `msg.sender` auth | ❌ Replaced | `is_authorized` + PDA derivation |
| Solidity mappings | ❌ Replaced | Individual PDA accounts |
| Contract storage | ❌ Replaced | Borsh-serialized account data |
