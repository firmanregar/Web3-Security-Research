ULTRA-PREMIUM AUDIT PROMPT PACK - Move adapted version.md

# 1. Global System

```
You are a senior smart contract security researcher specializing in Sui Move DeFi protocols.

The protocol uses:

- Move resource-based state management
- Coin<T> token standard
- capability-based access control
- PTB (Programmable Transaction Blocks) atomic transactions
- flashloan-enabled strategies
- lending markets with interest index accounting

Assume attackers can compose multiple entrypoints inside a single PTB transaction.

Focus on identifying:

- Flashloan manipulation
- Borrow index accounting bugs
- Exchange rate manipulation
- Liquidation bypass
- eMode invariant violations
- Oracle price manipulation
- Decimal precision errors
- Resource misuse vulnerabilities
- Capability leakage

Verify protocol invariants and simulate attack flows.

When possible, construct realistic exploit scenarios step-by-step.
```

# 2. Core Move Security Prompts (Foundational)

## Prompt 1 - Reource Integrity
```
Analyze how resources are created, transferred, or destroyed.

Check for:

- unintended duplication of resources
- resource leakage
- missing ownership validation
- improper use of key/store/copy/drop abilities
- ability misuse enabling state manipulation

Identify if a malicious user can create, clone, or destroy critical resources.
```

## Prompt 2 - Capability Access Control
```
Analyze all capability-based access control patterns.

Check for:

- AdminCap leakage
- MarketCap misuse
- privilege escalation
- functions missing capability checks

Determine whether a user can gain administrative privileges or call restricted functions.
```

### Prompt 3 - Object Ownership Validation
```
Review object ownership and reference validation.

Check if:

- users can pass objects they do not own
- object IDs can be spoofed
- shared objects allow unintended mutation

Identify any attack where a malicious user manipulates objects belonging to other users.
```

# 3. DeFi Economic Exploit Prompts

## Prompt 4 - Flashloan PTB Manipulation
```
Analyze if a flashloan can be used within a single PTB transaction
to manipulate protocol state.

Simulate attack flows such as:

flashloan
deposit
enter_market
borrow
withdraw
repay_flashloan

Determine whether temporary state manipulation can break protocol invariants.
```

## Prompt - Lending Insolvency Attack
```
Analyze the lending system for insolvency risks.

Check:

- borrow limit enforcement
- collateral factor calculation
- interest index synchronization
- rounding errors
- stale accounting

Determine whether a borrower can withdraw more value than their collateral.
```

## Prompt 6 - Exchange Rate Manipulation
```
Analyze cToken-style exchange rate calculations.

Verify whether an attacker can manipulate:

exchange_rate = (cash + borrows - reserves) / supply

Check for:

- flashloan deposit manipulation
- zero supply edge cases
- rounding errors

Simulate attacks where the exchange rate becomes artificially inflated.
```

## Prompt 7 - Interest Index Desynchronization
```
Analyze borrow index and interest accrual logic.

Check whether interest indexes are updated consistently across:

- borrow
- repay
- deposit
- liquidation

Identify scenarios where a user can repay less than the borrowed amount.
```

# 4. Liquidation Exploit Prompts

## Prompt 8 - Liquidation Bypass
```
Analyze the liquidation logic.

Verify:

- correct health factor calculation
- collateral value normalization
- correct borrow value computation
- liquidation incentives

Identify scenarios where a position should be liquidatable but cannot be liquidated.
```

## Prompt 9 - Bad Debt Scenario
```
Assume liquidation bots fail to execute.

Analyze whether underwater positions can cause:

- protocol insolvency
- reserve depletion
- stuck debt positions

Determine how the protocol handles bad debt.
```

# 5. Oracle Attack Prompts

## Prompt 10 - Oracle Manipulation
```
Analyze the oracle integration.

Check for:

- stale price usage
- missing timestamp validation
- decimal precision mismatch
- price update race conditions

Determine whether flashloan attacks can exploit oracle updates.
```

# 6. Move-specifir Attack Prompts

## Prompt 11 - PTB Atomicity Abuse
```
Assume an attacker can call multiple entrypoints within a single PTB transaction.

Analyze whether state validation occurs:

- before state mutation
- after state mutation
- only at transaction boundaries

Identify if temporary state manipulation can bypass safety checks.
```

## Prompt 12 - Shared Object Race Conditions
```
Analyze the use of shared objects.

Check whether multiple users interacting with shared state can cause:

- inconsistent accounting
- race conditions
- unexpected state overwrites
```

# 7. Decimal Precision Attack Prompts
```
Review all decimal normalization logic.

Check:

- token decimal mismatches
- price feed precision
- interest calculation precision

Identify scenarios where incorrect rounding benefits attackers.
```

# 8. Exploit Flow Pattern Prompts (Move Version)
```
Construct possible exploit flows using PTB atomic transactions.

Try combinations such as:

flashloan -> deposit -> borrow -> withdraw -> repay
deposit -> borrow -> switch_emode -> borrow
borrow -> manipulate_exchange_rate -> withdraw
borrow -> manipulate_oracle -> liquidate

Identify flows that break protocol invariants.
```

# One More Trick
During audits, run this meta prompt:

```
Assume the protocol designers missed one critical vulnerability.

Identify the most likely catastrophic exploit in this codebase.

Construct a full attack scenario including:
- attacker steps
- affected functions
- protocol loss
```