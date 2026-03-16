## Santa's List
[https://codehawks.cyfrin.io/c/ai-santas-list-clk83mi5b0004jp08axr82nq1/s/4]

100
EXP

Dec 22nd, 2025 → Dec 22nd, 2025

## Submission Details
Severity: **High**

## Users Can Collect Presents Without Passing Santa’s Second Check

firmanregar

## Root + Impact Description
Even if Santa never calls `checkTwice`, users can still collect presents because the second-check mapping defaults to NICE.z// Root cause in the codebase with @> marks to highlight the relevant section. No requirement that `checkTwice` was ever executed.
```solidity
if (
    s_theListCheckedOnce[msg.sender] == Status.NICE &&
    s_theListCheckedTwice[msg.sender] == Status.NICE
)
```

Both mappings default to `NICE`.

## Risk
**Likelihood:**
High

**Impact:**
* `checkTwice` is meaningless
* Single or zero verification grants rewards

## Proof of Concept
```solidity
vm.prank(attacker);
santasList.checkList(attacker, Status.NICE);
​
vm.prank(attacker);
santasList.collectPresent();
```

## Recommended Mitigation
Require explicit `checkTwice` completion (e.g., boolean flag).
