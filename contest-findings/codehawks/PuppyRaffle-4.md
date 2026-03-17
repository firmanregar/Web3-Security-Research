## Unchecked Arithmetic and Unsafe Downcasting Corrupt Fee Accounting
Matched: [H-05] Typecasting from uint256 to uint64 in PuppyRaffle.selectWinner() May Lead to Overflow and Incorrect Fee Calculation

## Root + Impact Description
The contract uses unchecked arithmetic and downcasts fees from `uint256` to `uint64` when accumulating protocol fees. In Solidity 0.7.6, arithmetic operations don't revert on overflow, and the downcast silently truncates values that exceed 64 bits. This can corrupt totalFees and eventually lock funds.

In `selectWinner()`, the fee calculation and accumulation looks like this:
```solidity
uint256 totalAmountCollected = players.length * entranceFee;
uint256 fee = (totalAmountCollected * 20) / 100;
totalFees = totalFees + uint64(fee);  // Unsafe cast from uint256 to uint64
```
If `fee` exceeds `type(uint64).max` (approximately 18.4 ETH), the high-order bits get discarded without warning.

## Risk
**Likelihood:**
High — Leads to permanent fund lock under certain conditions.

**Impact:**
When the cast truncates, totalFees no longer reflects the actual fees collected. This breaks the accounting invariant and has two serious consequences:
* Immediate fee loss — The protocol permanently loses the truncated portion of fees
* Withdrawal lockout — `withdrawFees()` requires `address(this).balance == uint256(totalFees)`, which will never be true once the accounting diverges
* The result is that the owner can't withdraw accumulated fees, even though the ETH is sitting in the contract.

## Proof of Concept
The vulnerability requires collecting enough fees to overflow uint64. With a 20% fee structure:
```solidity
fee > 2^64 - 1 when totalAmountCollected > 92 ETH
At entranceFee = 1 ETH, this requires 92+ players
At entranceFee = 0.1 ETH, this requires 920+ players
```
While large, these thresholds are reachable for popular raffles, especially those that run for extended periods or have low entry fees.

## Recommended Mitigation
Use Solidity 0.8.x (which has built-in overflow checks) or import SafeMath for 0.7.6. Store totalFees as uint256 instead of uint64:
```solidity
uint256 public totalFees = 0;  // Changed from uint64
```
This eliminates both the overflow risk and the unsafe downcast.
