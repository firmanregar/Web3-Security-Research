## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/1]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## Any User Can Poison Eligibility and Block Santa From Finalizing Status

firmanregar

## Root + Impact Description
The checkList function lacks access control, allowing any address to arbitrarily set or overwrite the first-check status of any user. This enables attackers to sabotage eligibility, self-assign favorable statuses, or permanently block Santa from finalizing checks
In `SantasList.sol`, the checkList function is declared as external without access control. checkList is missing the `onlySanta` modifier.
Compare this to `checkTwice`, which correctly implements the `onlySanta` modifier. The inconsistency suggests this was an oversight rather than intentional design.
```solidity
function checkList(address person, Status status) external {
    s_theListCheckedOnce[person] = status;
}
```

## Risk
**Likelihood:**

* High, Single call, no privileges, trivial execution.

**Impact:**

* Attacker can mark victims as `NAUGHTY`, permanently preventing rewards.
* Attacker can self-assign `NICE`/`EXTRA_NICE`.
* Santa’s checkTwice can be griefed into perpetual revert.

## Proof of Concept
```solidity
function testAnyoneCanPoisonFirstCheck() public {
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");
    
    // Santa intends to mark victim as NICE
    vm.prank(santa);
    santasList.checkList(victim, SantasList.Status.NICE);
    
    // Attacker overwrites it
    vm.prank(attacker);
    santasList.checkList(victim, SantasList.Status.NAUGHTY);
    
    // Santa's second check now fails permanently
    vm.prank(santa);
    vm.expectRevert(SantasList.SantasList__SecondCheckDoesntMatchFirst.selector);
    santasList.checkTwice(victim, SantasList.Status.NICE);
}
```

Anyone can overwrite `s_theListCheckedOnce`, breaking the trust model.

## Recommended Mitigation
Add `onlySanta` to checkList.
```solidity
function checkList(address person, Status status) external onlySanta {
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}
```
