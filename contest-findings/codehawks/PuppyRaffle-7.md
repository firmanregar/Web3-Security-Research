## Fee Withdrawal Permanently Blocked by Strict Balance Check
Matched: [M-02] Slightly increasing puppyraffle's contract balance will render `withdrawFees` function useless

## Root + Impact Description
The `withdrawFees()` function requires the contract balance to exactly equal totalFees. Any ETH sent to the contract outside of the normal flow (e.g., via selfdestruct or while players are active) permanently breaks this equality, making fees unwithdrawable forever.
The withdrawal logic uses strict equality:
```solidity
require(address(this).balance == uint256(totalFees),
    "PuppyRaffle: There are currently players active!");
```

This assumes the contract balance only ever contains protocol fees and nothing else. In practice, this assumption is fragile:
* Forced ETH transfers. Anyone can force ETH into the contract via selfdestruct from another contract
* Active player funds. If players are active, their entrance fees inflate the balance above totalFees
* Rounding errors. Any accounting mismatch (like from the downcasting issue) permanently breaks the equality

## Risk
**Likelihood:**
High probability of occurrence and results in permanent fund lock.

**Impact:**
Once the balance diverges from totalFees, the owner can never withdraw accumulated fees. This is particularly problematic because:
* Forced ETH transfers are trivial and unavoidable on Ethereum
* There's no recovery mechanism
* All future fees become permanently locked

## Proof of Concept
An attacker can lock fees with a single transaction:
```solidity
contract ForceSend {
    constructor(address payable target) payable {
        selfdestruct(target);
    }
}
​
// Deploy with:
new ForceSend{value: 1 wei}(payable(address(puppyRaffle)));
```
This sends 1 wei to the raffle contract without triggering any functions. Now address(this).balance != totalFees, and the equality check fails forever.

## Recommended Mitigation
Track player funds separately from protocol fees, or simply allow withdrawing the known fee amount:
```solidity
function withdrawFees() external {
    require(totalFees > 0, "No fees to withdraw");
    require(address(this).balance >= totalFees, "Insufficient balance");
    
    uint256 feesToWithdraw = totalFees;
    totalFees = 0;
    
    (bool success,) = feeAddress.call{value: feesToWithdraw}("");
    require(success, "PuppyRaffle: Failed to withdraw fees");
}
```

This removes the strict equality requirement while still ensuring fees are available for withdrawal.
