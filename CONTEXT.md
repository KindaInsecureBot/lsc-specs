# Context for LSC Spec Writing

## What This Repo Is

This repo contains **specifications for an RFP (Request for Proposals)**. An external team will implement the code. We only write specs here — no Rust implementation.

## The Goal

Create specs for **LSC (Logos Stable Coin)** — a RAI-like reflexive stablecoin for the **Logos Execution Zone (LEZ)** blockchain.

The model is **Reflexer Finance / RAI**:
- Whitepaper: https://github.com/reflexer-labs/whitepapers/blob/master/English/rai-english.pdf
- Repos: https://github.com/reflexer-labs (especially geb, geb-safe-manager, geb-deploy, geb-oracle-contracts)
- Sister project: https://www.letsgethai.com/ / https://docs.letsgethai.com/

## RAI Model Summary (for reference)

RAI is a non-pegged stablecoin backed by ETH collateral. Key components:

1. **SAFEs (collateralized debt positions)** — users lock ETH collateral and mint RAI against it
2. **Redemption Rate** — a floating interest rate that adjusts the redemption price to bring market price toward it (PI controller)
3. **Redemption Price** — the target price RAI should trade at (starts at some value, drifts based on redemption rate)
4. **Market Price** — actual market price from oracle
5. **Liquidation Engine** — liquidates undercollateralized SAFEs
6. **Oracle System** — feeds price data (collateral price, RAI market price)
7. **Stability Fee** — interest rate on debt
8. **Surplus/Debt Auctions** — handle protocol surplus and bad debt
9. **Tax Collector** — accrues stability fees on SAFEs
10. **Global Settlement** — emergency shutdown mechanism

The PI controller is the core innovation: when market price > redemption price, the redemption rate goes negative (making holding RAI less attractive), and vice versa. This creates a self-correcting mechanism without requiring a peg.

## LEZ Platform Overview

LEZ is a programmable blockchain with hybrid public/private state. Key characteristics:

### Account Model
- Accounts: `{ program_owner: ProgramId, balance: u128, data: Data, nonce: u128 }`
- AccountId: 32-byte identifier (base58 encoded)
- Accounts can be public (visible on-chain) or private (ZK-committed)
- Programs are **stateless** — all state lives in accounts passed as inputs

### Program Model
- Programs compile to RISC-V (Risc0 VM) binaries
- Every program follows: `read_nssa_inputs()` → logic → `write_nssa_outputs()`
- Programs receive `pre_states: Vec<AccountWithMetadata>` and an `instruction: T`
- Programs output `post_states: Vec<AccountPostState>`
- **Authorization**: accounts have `is_authorized` flag (signed tx for public, nullifier proof for private)
- **Ownership**: programs claim uninitialized accounts, only owner can mutate data/balance
- **Chained calls (tail calls)**: programs can invoke other programs via `ChainedCall`
- **PDAs**: program-derived accounts — each program has 2^256 unique account IDs it can authorize

### Existing Contracts

#### Token Program
```rust
enum Instruction {
    Transfer { amount_to_transfer: u128 },
    NewFungibleDefinition { name: String, total_supply: u128 },
    NewDefinitionWithMetadata { new_definition, metadata },
    InitializeAccount,
    Burn { amount_to_burn: u128 },
    Mint { amount_to_mint: u128 },
    PrintNft,
}
```
- Accounts: TokenDefinition, TokenHolding (Fungible { definition_id, balance } | NftMaster | NftPrintedCopy)
- Authorization required for: sender in Transfer, holder in Burn, definition in Mint

#### AMM Program
```rust
enum Instruction {
    NewDefinition { token_a_amount, token_b_amount, amm_program_id },
    AddLiquidity { min_amount_liquidity, max_amount_to_add_token_a, max_amount_to_add_token_b },
    RemoveLiquidity { remove_liquidity_amount, min_amount_to_remove_token_a, min_amount_to_remove_token_b },
    Swap { swap_amount_in, min_amount_out, token_definition_id_in },
}
```
- Uses constant-product (x*y=k) formula
- Pool state: PoolDefinition { definition_token_a/b_id, vault_a/b_id, liquidity_pool_id, liquidity_pool_supply, reserve_a/b, fees, active }
- Uses PDAs for pool accounts, vault accounts, and LP token definitions
- Uses chained calls to Token Program for transfers, mints, burns
- Accounts: Pool (PDA), Vault A/B (PDA), LP Token Definition (PDA), User Holdings

### Important LEZ Constraints
1. Programs are stateless — no internal storage, everything via accounts
2. No time/block-number access inside the VM (oracles must provide timestamps)
3. Account data is arbitrary bytes (borsh-serialized structs)
4. Total balance across pre/post states must be preserved (conservation rule)
5. Only program owner can decrease balance or modify data
6. Account IDs are unique per execution
7. Maximum 10 chained calls per execution
8. Programs can run publicly (on-chain) or privately (ZK proof)

## What the Specs Should Cover

### Document Structure
Create well-organized markdown specs covering:

1. **Overview** — What LSC is, how it works at a high level, comparison to RAI
2. **System Architecture** — All programs (contracts) needed, their relationships
3. **Account Schemas** — Every account type with exact data structures
4. **Program Specifications** — For each program:
   - Instructions (enum variants with parameters)
   - Required accounts per instruction
   - Authorization requirements
   - Validation rules
   - State transitions
   - Chained calls to other programs
5. **PDA Derivation** — How all program-derived accounts are computed
6. **Oracle System** — How price feeds work in LEZ's model
7. **PI Controller** — The redemption rate controller (this is the core RAI innovation)
8. **Liquidation Mechanism** — How undercollateralized positions are liquidated
9. **Global Settlement** — Emergency shutdown
10. **Parameter Recommendations** — Initial values for collateralization ratio, stability fee, PI controller gains, etc.
11. **Integration with Existing Contracts** — How LSC uses the Token Program and AMM Program
12. **Privacy Considerations** — What can/should be private vs public
13. **Security Considerations** — Attack vectors, mitigations

### Adaptation Decisions (RAI → LEZ)
- ETH collateral → LOGOS token collateral
- Solidity contracts → LEZ programs (stateless, account-based)
- Block.timestamp → Oracle-provided timestamps
- Mappings → Individual accounts (each SAFE is its own account)
- Events → Not applicable (use indexer)
- msg.sender → is_authorized flag
- Contract storage → Account data fields
- ERC-20 → Token Program integration via chained calls
- Uniswap oracle → AMM Program integration

### Quality Bar
- Specs must be detailed enough for an external team to implement without guessing
- Include exact data structure definitions (as if writing Rust structs)
- Include exact instruction enums
- Include all validation rules and edge cases
- Include sequence diagrams for complex flows (liquidation, rate updates)
- Reference the RAI whitepaper for mathematical formulas
