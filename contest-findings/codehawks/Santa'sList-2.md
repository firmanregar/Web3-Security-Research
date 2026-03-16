## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/2]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## Default Enum Values Grant Universal Eligibility, Any Address Can Mint Free NFTs Due to Default NICE Status

firmanregar

## Root + Impact Description
Solidity mappings initialize to zero, and the Status enum defines `NICE = 0`. This means every address starts with `NICE` status by default. The `collectPresent` function checks both mappings for `NICE`, allowing any address to collect presents without ever being checked by Santa.
The enum ordering in `SantasList.sol`, combined with the eligibility check in `collectPresent`. Since both mappings default to `0` (NICE), the condition passes for unchecked addresses.
```solidity
enum Status {
    NICE,           // = 0 (default)
    EXTRA_NICE,     // = 1
    NAUGHTY,        // = 2
    NOT_CHECKED_TWICE // = 3
}
​
if (s_theListCheckedOnce[msg.sender] == Status.NICE && 
    s_theListCheckedTwice[msg.sender] == Status.NICE) {
    _mintAndIncrement();
    return;
}
```

## Risk
**Likelihood:**

* High. Deterministic behavior with no setup.

**Impact:**
Complete bypass of the verification system. Any address can:
* Mint NFTs without Santa's approval
* Create unlimited sybil addresses to drain the NFT supply
* Render the entire checking mechanism meaningless

## Proof of Concept
```solidity
function testUncheckedUserCanCollect() public {
    address attacker = makeAddr("attacker");
    
    // Warp past Christmas
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
    
    // Attacker was never checked by Santa
    assertEq(uint256(santasList.getNaughtyOrNiceOnce(attacker)), 0);
    assertEq(uint256(santasList.getNaughtyOrNiceTwice(attacker)), 0);
    
    // But can still collect
    vm.prank(attacker);
    santasList.collectPresent();
    
    assertEq(santasList.balanceOf(attacker), 1);
}
```

## Recommended Mitigation
Restructure the enum to use a non-eligible default state.
Alternatively, add explicit bool flags to track whether addresses have been checked.
```solidity
enum Status {
    NOT_CHECKED_TWICE,  // = 0 (default, ineligible)
    NICE,               // = 1
    EXTRA_NICE,         // = 2
    NAUGHTY            // = 3
}
```
