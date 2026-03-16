## Boss Bridge
[https://codehawks.cyfrin.io/c/2023-11-Boss-Bridge/s/143]

100
EXP

Nov 9th, 2023 → Nov 15th, 2023

## Submission Details
Severity: **High**

## Unprotected call to a function sending erc20 tokens to an arbitrary address.

firmanregar

## Summary
Unprotected call to a function sending erc20 tokens to an arbitrary address. When a smart contract allows for arbitrary sending of erc20 tokens, it means that anyone can trigger the transfer of funds to any recipient without proper controls or validation.

## Vulnerability Details
`L1BossBridge.depositTokensToL2(address,address,uint256)` (src/L1BossBridge.sol#70-78) uses arbitrary from in transferFrom: `token.safeTransferFrom(from,address(vault),amount)` (src/L1BossBridge.sol#74)

## Impact
Malicious actors could exploit the vulnerability to transfer ERC-20 tokens from the contract to arbitrary addresses without proper authorization. The arbitrary sending of ERC-20 tokens can result in financial loss for the contract and its users if tokens are moved without the intended permissions or conditions.
If the contract has a significant balance of ERC-20 tokens and arbitrary sending is not properly controlled, it could be targeted for repeated transfers, leading to a denial of service by consuming excessive gas. Arbitrary ERC-20 token transfers may allow attackers to manipulate the contract's state, potentially leading to unintended changes in variables or critical parameters.
The vulnerability could allow attackers to bypass security mechanisms, access controls, or conditions that are in place to prevent unauthorized token transfers.

## Tools Used
Manual code review

## Recommendations
Use access controls to ensure that only authorized users or contracts can trigger token transfers. Validate and sanitize inputs to the contract, especially if the token transfer is contingent on specific conditions or criteria. Implement limits on the amount of ERC-20 tokens that can be transferred in a single transaction to minimize the potential impact of an attack.
