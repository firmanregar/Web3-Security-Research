## Boss Bridge
[https://codehawks.cyfrin.io/c/2023-11-Boss-Bridge/s/142]

100
EXP

Nov 9th, 2023 → Nov 15th, 2023

## Submission Details
Severity: **High**

## Function `L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)` potentially sends eth to arbitrary user

firmanregar

## Summary
Unprotected call to a function sending Ether to an arbitrary address. When a smart contract allows for arbitrary sending of ETH, it means that anyone can trigger the transfer of funds to any recipient without proper controls or validation.

## Vulnerability Details
`L1BossBridge.sendToL1(uint8,bytes32,bytes32,bytes)` (src/L1BossBridge.sol#112-125) sends eth to arbitrary user
Dangerous calls:
```
(success) = target.call{value: value}(data) (src/L1BossBridge.sol#121)
```

## Impact
Malicious actors could exploit the vulnerability to transfer funds from the contract to arbitrary addresses without proper authorization. The arbitrary sending of ETH can result in financial loss for the contract and its users if funds are moved without the intended permissions or conditions.
If the contract has a significant balance and arbitrary sending is not properly controlled, it could be targeted for repeated transfers, leading to a denial of service by consuming excessive gas. Arbitrary ETH transfers may allow attackers to manipulate the contract's state, potentially leading to unintended changes in variables or critical parameters.
The vulnerability could allow attackers to bypass security mechanisms, access controls, or conditions that are in place to prevent unauthorized fund transfers.

## Tools Used
Manual code review

## Recommendations
Ensure that an arbitrary user cannot withdraw unauthorized funds. Use access controls to ensure that only authorized users or contracts can trigger fund transfers.
