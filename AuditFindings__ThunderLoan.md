# High

### [H-*] Erroneus `ThunderLoan::updateExchangeRate` function in the `deposit` function causes protocol to think it has more fees than, it really does which block redemptions and incorrectly sets the exchange rate

**Description:** In the Thunderloan System the exchange rate is responsible for calculating the exchange rate between assetTokens and underlying tokens. In a way, its responsibel for keeping track of how many fees to give to liqudity providers.

However, the deposit function updates this rate, without collecting any fees! 

```solidity
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetTReoken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
@>        uint256 calculatedFee = getCalculatedFee(token, amount);
@>        assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

**Impact:**
1. The `redeem` function is blocked because protocol thinks the owed tokens are more than it has
2. Rewards are incorrectly calculated, leading users potentially getting more or less than deserved.

**Proof Of Concept:**

1. LP deposits
2. User takes out a flash loan
3. It is now impossible for LP to Redeem

<details>

<summary>Proof Of Code:</summary>

Place the following into `ThunderLoanTest.t.sol`:

```solidity
  function testRedeemAfterLoan() public setAllowedTokens hasDeposits {
    uint256 amountToBorrow = AMOUNT * 10;
    uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

    vm.startPrank(user);
    tokenA.mint(address(mockFlashLoanReciver),calculatedFee);
    thunderLoan.flashloan(address(mockFlashLoanReciver),tokenA, amountToBorrow,"");
    vm.stopPrank():

    uint256 amountToRedeem = type(uint256).max;
    vm.startPrank(liqudityProvider);
    thunderloan.redeem(token, amountToRedeem);
}
```
</details>

**Recommended Mitigations:** Remove the incorrectly updated exchange rate lines from `deposit`.

```diff
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetTReoken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
-        uint256 calculatedFee = getCalculatedFee(token, amount);
-        assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```

