## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/5]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## `EXTRA_NICE` Addresses Can Inflate `SantaToken` Supply Infinitely

firmanregar

## Root + Impact Description
By combining the NFT transfer bypass with `EXTRA_NICE` rewards, users can repeatedly mint both NFTs and `SantaTokens`.
Root Cause :
* Same balance-based guard flaw
* Token minting tied to NFT mint without permanent state lock
```solidity
if (EXTRA_NICE) {
    i_santaToken.mint(msg.sender);
}
```

Executed on every successful collection.

## Risk
**Likelihood:**
High

**Impact:**
* Infinite SantaToken inflation
* Token value collapse

## Proof of Concept
```solidity
collect → transfer NFT → collect again
assertEq(santaToken.balanceOf(attacker), 2e18);
```

## Recommended Mitigation
Lock token rewards to a one-time claim flag.
