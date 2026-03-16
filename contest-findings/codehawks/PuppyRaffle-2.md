## Puppy Raffle
[https://codehawks.cyfrin.io/c/2023-10-Puppy-Raffle/s/823]

100
EXP

Oct 25th, 2023 → Nov 1st, 2023

## Submission Details
Severity: **High**

# Potential reentrancy in `PuppyRaffle.refund(uint256)` would allow the miner to claim the refund

firmanregar

## Summary
The function does not check if playerIndex is within the bounds of the players array. If a caller provides an index that is greater than the length of the array, the function will revert due to an out-of-bounds error.

## Vulnerability Details
The function uses `sendValue` to send the refund to the player. The `sendValue` function only provides 2300 gas, which may not be enough if the recipient is a contract with a complex fallback function. If the recipient contract's fallback function requires more than 2300 gas, the `sendValue` call will fail. It's generally safer to use transfer instead of `sendValue`.

The function does not check if the contract has enough balance to refund the entranceFee to the player. If the contract's balance is less than entranceFee, the sendValue call will fail.

## Impact
Since the function uses the `msg.sender` to identify the player requesting a refund, a malicious miner could potentially front-run the transaction by inserting their own transaction before the player's transaction in the block. This would allow the miner to claim the refund instead of the player. On the other hand, although reentrancy is less likely to be an issue in this function because state changes occur after the external call, it's generally a good practice to use a reentrancy guard in functions that make external calls.

## Tools Used
Slither, Manual Review

## Recommendations
Apply the [check-effects-interactions pattern]
