## Boss Bridge
[https://codehawks.cyfrin.io/c/2023-11-Boss-Bridge/s/140]

100
EXP

Nov 9th, 2023 → Nov 15th, 2023

## Submission Details
Severity: **High**

## Manipulated arbitrary call found in `L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)`

firmanregar

## Summary
Manipulated arbitrary call found in `L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)` that possibly can execute arbitrary code or make arbitrary calls to external contracts, pose a significant security risk.

## Vulnerability Details
Manipulated call found: `(success) = target.call{value: value}(data) (src/L1BossBridge.sol#121)` in `L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)` (src/L1BossBridge.sol#112-125)
Both destination and calldata could be manipulated
The call could be fully manipulated (arbitrary call) through `L1BossBridge.constructor(IERC20)` (src/L1BossBridge.sol#42-47)
The call could be fully manipulated (arbitrary call) through `L1BossBridge.withdrawTokensToL1(address,uint256,uint8,bytes32,bytes32)` (src/L1BossBridge.sol#91-102)
The call could be fully manipulated (arbitrary call) through `L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)` (src/L1BossBridge.sol#112-125)

## Impact
An attacker could exploit arbitrary calls to gain unauthorized access to sensitive functions or data within the contract, potentially compromising the security and privacy of the system.

Manipulation of State: Arbitrary calls might allow an attacker to manipulate the contract's state, leading to unintended changes in variables or critical parameters. This can result in unexpected behavior and potential financial loss.

Reentrancy Attacks: If arbitrary calls are not handled carefully, they can open the door to reentrancy attacks, where an external contract repeatedly calls back into the vulnerable contract, potentially disrupting its intended functionality and leading to unexpected outcomes.

Execution of Malicious Code: An attacker could use arbitrary calls to execute malicious code, leading to a variety of exploits, such as stealing funds, corrupting data, or disrupting the normal operation of the contract.

## Tools Used
Slither, Manual code review

## Recommendations
Implement strict access controls, validate and sanitize inputs, and avoid allowing external contracts to execute arbitrary code within your contract. Use the checks-effects-interactions pattern to minimize the risk of reentrancy attacks.
