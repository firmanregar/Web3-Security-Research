## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/8]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## Malicious ERC20 Implementation Skips Allowance Checks

firmanregar

## Root + Impact Description
`SantaToken` Allows Unauthorized Transfers Due to Compromised ERC20 Logic. The Solmate ERC20 implementation used allows a specific address to bypass allowance checks, enabling unrestricted token theft.

`transferFrom` bypasses allowance for a specific address.

## Risk
**Likelihood:**
High. Hardcoded behavior.

**Impact:**
* Arbitrary token theft
* ERC20 guarantees violated

## Proof of Concept
```solidity
santaToken.transferFrom(victim, attacker, 1e18);
```

## Recommended Mitigation
Replace with a verified ERC20 implementation (OpenZeppelin or clean Solmate).
