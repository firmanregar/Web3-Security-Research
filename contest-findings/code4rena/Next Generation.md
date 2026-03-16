# Replay Attacks via Externally Supplied Domain Separator
https://code4rena.com/audits/2025-01-next-generation/submissions?uid=QdVCMZNxq2x

**High**

## Finding description and impact
The Forwarder contract accepts an externally provided `domainSeparator` in `execute` and `verify`, making it vulnerable to cross-contract replay attacks. Attackers can reuse signatures from another contract/chain by supplying a different `domainSeparator`, bypassing signature validation if the same `requestTypeHash` is registered. This compromises the integrity of meta-transactions, allowing unauthorized execution.

## Proof of Concept
* Scenario: Two Forwarder contracts (A and B) share the same `requestTypeHash`.
* Attack: A user's valid signature for Contract A is submitted to Contract B with A's `domainSeparator`.
* Result: Signature validates on Contract B, executing unintended transactions.

## Recommended mitigation steps
* Compute `domainSeparator` internally using EIP-712 parameters (chainId, contract address).
* Remove `domainSeparator` as a user-provided input to prevent manipulation.

## Links to affected code
Forwarder.sol#L77-L118 [https://github.com/code-423n4/2025-01-next-generation/blob/499cfa50a56126c0c3c6caa30808d79f82d31e34/contracts/Forwarder.sol#L77-L118]
