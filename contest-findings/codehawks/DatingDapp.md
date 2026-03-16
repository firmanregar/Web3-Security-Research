## DatingDapp
[https://codehawks.cyfrin.io/c/ai-datingdapp-clk83mi5b0004jp08axr82nq1/s/1]

100
EXP

Jan 18th, 2026 → Jan 18th, 2026

## Submission Details
Impact: **High**

## User Funds Lost Due to Missing Balance Accounting in `likeUser()`

firmanregar

## Root + Impact Description
The `likeUser()` function accepts ETH payment but fails to credit it to the sender's `userBalances` mapping. When users match, the reward distribution logic operates on zero balances, meaning users lose all deposited ETH with no way to recover it.

## Risk
Likelihood:
* The `likeUser()` function is marked payable and requires users to send at least 1 ETH:
```solidity
function likeUser(address liked) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    // ...
    
    likes[msg.sender][liked] = true;
    emit Liked(msg.sender, liked);
    
    if (likes[liked][msg.sender]) {
        matchRewards(liked, msg.sender);
    }
}
```

However, the function never updates `userBalances\[msg.sender]` with the received ETH.
When two users mutually like each other, `matchRewards()` executes:
```solidity
function matchRewards(address from, address to) internal {
    uint256 matchUserOne = userBalances[from];  // Returns 0
    uint256 matchUserTwo = userBalances[to];    // Returns 0
    userBalances[from] = 0;
    userBalances[to] = 0;
    
    uint256 totalRewards = matchUserOne + matchUserTwo;  // 0 + 0 = 0
    uint256 matchingFees = (totalRewards * FIXEDFEE) / 100;  // 0
    uint256 rewards = totalRewards - matchingFees;  // 0
    
    // Deploys multisig with 0 ETH
    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);
    (bool success,) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```

## Impact:

Direct User Fund Loss:
* Users pay 1+ ETH per like
* ETH is accepted by the contract
* ETH is never credited to their balance
* When matching occurs, rewards distribute 0 ETH
* Users cannot recover their deposited funds

Broken Core Functionality:
* The matching and reward system is non-functional
* Contract accumulates ETH with no distribution mechanism
* Users lose funds during normal, intended protocol usage

## Proof of Concept
```solidity
function test_UserETHLostDueToMissingAccounting() public {
    // Setup: Users mint profiles
    vm.prank(userA);
    nft.mintProfile("Alice", 30, "ipfs://alice");
    
    vm.prank(userB);
    nft.mintProfile("Bob", 32, "ipfs://bob");
    
    // Track initial balances
    uint256 registryBalanceBefore = address(registry).balance;
    uint256 userABalanceBefore = userA.balance;
    uint256 userBBalanceBefore = userB.balance;
    
    // User A likes User B (pays 1 ETH)
    vm.prank(userA);
    registry.likeUser{value: 1 ether}(userB);
    
    // User B likes User A (pays 1 ETH, triggers match)
    vm.prank(userB);
    registry.likeUser{value: 1 ether}(userA);
    
    // Verify contract received ETH
    assertEq(address(registry).balance, registryBalanceBefore + 2 ether);
    
    // Verify userBalances were never credited
    assertEq(registry.userBalances(userA), 0);
    assertEq(registry.userBalances(userB), 0);
    
    // Verify users lost ETH
    assertEq(userA.balance, userABalanceBefore - 1 ether);
    assertEq(userB.balance, userBBalanceBefore - 1 ether);
    
    // Multisig received 0 ETH (rewards were calculated as 0)
    address[] memory matches = registry.getMatches();
    // Users matched but received no rewards
}
```

Since `userBalances` is never populated, `totalRewards` is always zero, and the multisig wallet receives no ETH despite both users having paid.

Recommended Mitigation
Credit received ETH to the sender's balance in `likeUser()`:
```solidity
function likeUser(address liked) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");
    
    userBalances[msg.sender] += msg.value;  // ADD THIS
    
    likes[msg.sender][liked] = true;
    emit Liked(msg.sender, liked);
    
    if (likes[liked][msg.sender]) {
        matches[msg.sender].push(liked);
        matches[liked].push(msg.sender);
        emit Matched(msg.sender, liked);
        matchRewards(liked, msg.sender);
    }
}
```


This ensures that deposited ETH is properly tracked and can be distributed as rewards when users match.
