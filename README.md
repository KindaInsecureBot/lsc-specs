# LSC — Logos Stable Coin

A RAI-like reflexive stablecoin for the [Logos Execution Zone (LEZ)](https://logos.co/) blockchain.

This repository contains **specifications for an RFP (Request for Proposals)**. An external team will implement the code. No Rust implementation is included here — only specifications.

---

## What Is LSC?

LSC is a **non-pegged, collateralized stablecoin** backed by LOGOS tokens. It does not target a fixed USD value. Instead, an autonomous PI (Proportional-Integral) feedback controller continuously adjusts a *redemption rate*, which incentivizes market participants to keep the LSC market price close to an internal *redemption price*. The redemption price itself floats over time.

This design — pioneered by [RAI / Reflexer Finance](https://reflexer.finance/) — produces a stablecoin that is:
- **Minimally governed**: the stabilization mechanism is mathematical, not policy-driven
- **Non-pegged**: more resilient to peg-breaking events
- **Reflexive**: the system corrects itself via economic incentives, not interventions

---

## How to Read These Specs

The specs are written to be **implementation-ready**. An experienced Rust developer should be able to implement each program from its spec alone without needing to ask clarifying questions.

**Suggested reading order:**

1. Start with the [Overview](specs/00-overview.md) to understand the system conceptually.
2. Read [System Architecture](specs/01-system-architecture.md) to understand all programs and their relationships.
3. Read [Account Schemas](specs/02-account-schemas.md) to understand all data structures.
4. Read each program spec in order (03–08).
5. Read integration, privacy, and security specs last.

**Key conventions used:**

- `Ray` = `u128` with 27 decimal places (`10^27 = 1.0`). Used for rates and price multipliers.
- `Wad` = `u128` with 18 decimal places (`10^18 = 1.0`). Used for token amounts and USD prices.
- `Rad` = `u128` with 45 decimal places (`10^45 = Ray * Wad`). Used for debt accounting.
- All account data is Borsh-serialized.
- PDAs (Program-Derived Accounts) are deterministic from seeds.
- All programs are stateless; all state lives in accounts.
- Authorization uses the `is_authorized` flag on accounts passed to instructions.

---

## Table of Contents

### Core Specifications

| # | File | Contents |
|---|---|---|
| 00 | [specs/00-overview.md](specs/00-overview.md) | What LSC is, high-level architecture, comparison to RAI, glossary |
| 01 | [specs/01-system-architecture.md](specs/01-system-architecture.md) | All programs, account ownership graph, PDA derivations, initialization sequence |
| 02 | [specs/02-account-schemas.md](specs/02-account-schemas.md) | Every account type with Rust struct definitions and serialization notes |

### Program Specifications

| # | File | Contents |
|---|---|---|
| 03 | [specs/03-core-engine.md](specs/03-core-engine.md) | LSCEngine: SAFE lifecycle, collateral, debt generation/repayment, all instructions |
| 04 | [specs/04-oracle-system.md](specs/04-oracle-system.md) | OracleProgram + OracleRelayer: price feeds, median aggregation, timestamps, redemption price |
| 05 | [specs/05-pi-controller.md](specs/05-pi-controller.md) | PI Controller: mathematical model, error terms, integral windup, all instructions |
| 06 | [specs/06-liquidation-engine.md](specs/06-liquidation-engine.md) | LiquidationEngine + CollateralAuctionHouse: liquidation trigger, fixed-discount auctions |
| 07 | [specs/07-accounting-engine.md](specs/07-accounting-engine.md) | AccountingEngine + SurplusAuctionHouse + DebtAuctionHouse: surplus/debt management |
| 08 | [specs/08-global-settlement.md](specs/08-global-settlement.md) | GlobalSettlement: emergency shutdown, collateral processing, LSC redemption |

### Integration and Operations

| # | File | Contents |
|---|---|---|
| 09 | [specs/09-token-integration.md](specs/09-token-integration.md) | Token Program + AMM Program integration, chained call patterns |
| 10 | [specs/10-privacy-considerations.md](specs/10-privacy-considerations.md) | What can be private, privacy constraints, v2 recommendations |
| 11 | [specs/11-security-considerations.md](specs/11-security-considerations.md) | Attack vectors, mitigations, implementation checklist, known RAI issues |
| 12 | [specs/12-parameters.md](specs/12-parameters.md) | All system parameters with types, initial values, ranges, and governance |

---

## Program Summary

| Program | Role |
|---|---|
| `LSCEngine` | Core SAFE management: open/close SAFEs, deposit/withdraw collateral, mint/burn LSC |
| `TaxCollector` | Accrues stability fees on SAFE debt |
| `OracleRelayer` | Stores and computes redemption price from redemption rate |
| `PIController` | Computes new redemption rate from market/redemption price error |
| `LiquidationEngine` | Detects and liquidates undercollateralized SAFEs |
| `CollateralAuctionHouse` | Fixed-discount auctions for seized collateral |
| `AccountingEngine` | Routes surplus and bad debt; triggers auctions |
| `SurplusAuctionHouse` | Auctions protocol LSC surplus for LOGOS (burned) |
| `DebtAuctionHouse` | Auctions minted LOGOS for LSC to cover bad debt |
| `OracleProgram` | Aggregates price feeds via median oracle |
| `GlobalSettlement` | Emergency shutdown mechanism |

---

## Key References

- RAI Whitepaper: [github.com/reflexer-labs/whitepapers](https://github.com/reflexer-labs/whitepapers/blob/master/English/rai-english.pdf)
- RAI Code: [github.com/reflexer-labs/geb](https://github.com/reflexer-labs/geb)
- HAI (RAI fork with good documentation): [docs.letsgethai.com](https://docs.letsgethai.com/)
- LEZ Context: [CONTEXT.md](CONTEXT.md)

---

## RFP Notes for Implementors

1. **Language:** Rust (compiles to RISC-V via Risc0)
2. **No implementation files are included** — only specs. Create a fresh implementation from these docs.
3. **Arithmetic:** Use 256-bit intermediate arithmetic throughout. All ray/wad multiplications must use a `u256` type or equivalent.
4. **Testing:** Unit test all arithmetic edge cases. Integration test liquidation flows end-to-end with a simulated oracle.
5. **Audit:** Plan for a security audit before mainnet deployment. Key areas: oracle system, liquidation arithmetic, PI controller integral handling.
6. **Questions:** If any spec is ambiguous, document your interpretation decision and flag it for review.
