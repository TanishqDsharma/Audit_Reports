# High

### [H-1] `preDepositPrice` can be manipulated resulting in stealing of user'd deposit

**Description:** The issue arises when the first user that stakes manipulates the total supply of `safEth Tokens` and by doing so create a rounding error for each subsequent user. In the worst case, an attacker can steal all the funds of the next user.

```solidity

 uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
        _mint(msg.sender, mintAmount);

```
To attack the `SafEth` the attackers first needs to become the first depositor with any amount of staked ETH, then withdraw the most of the stake so there is only 1 wei of SafEth shares left, and then inflate the underlying asset value by manually transferring a large amount of underlying assets to relevant derivative pools instead going through the `SafEth.stake()` function. 


**Impact:**  Loss of User's funds as any staker after the hacker's attack will receive zero shares for their stake. The attacker can redeem all underlying funds at any time by burning the only 1 wei share in the contract.

**Proof Of Code:**

**Recommended mitigations:** To add a mitigation against this attack protocol can implement the logic to mint the dead shares so that the cost to perform this attack increases considerably.

### [H-2] `Reth::ethPerDerivation` function results in inconsitent results for rETH in `stake()`

**Description:** In the Reth contract, ethPerDerivative() is used to fetch the price of rETH in terms of ETH:

```solidity
/**
    @notice - Get price of derivative in terms of ETH
    @dev - we need to pass amount so that it gets price from the same source that it buys or mints the rEth
    @param _amount - amount to check for ETH price
*/
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
    if (poolCanDeposit(_amount))
        return
 @>           RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
 @>   else return (poolPrice() * 10 ** 18) / (10 ** 18);
}
```
For rETH specifically, ethPerDerivative() returns two possible prices: 
* If _amount can be deposited, price is based of getEthValue(10 ** 18), the price of rETH in the rocket deposit pool. 
* Otherwise, price is calculated using poolprice(), which is the price of rETH in the Uniswap liquidity pool.

As seen from above, `poolCanDeposit()` checks if _amount can be deposited into RocketDepositPool. Therefore, if _amount can be deposited into rocket pool, ethPerDerivative() returns its price. Otherwise, Uniswap's price is returned.


This becomes an issue in the SafEth contract. In the stake() function, ethPerDerivative() is first called when calculating the value held in each derivative contract:

```solidity
// Getting underlying value in terms of ETH for each derivative
for (uint i = 0; i < derivativeCount; i++)
    underlyingValue +=
        (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
            derivatives[i].balance()) /
        10 ** 18;
```

Afterwards, stake() uses the following logic to deposit ETH into each derivative contract:

```solidity
uint256 depositAmount = derivative.deposit{value: ethAmount}();
uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
    depositAmount
) * depositAmount) / 10 ** 18;
totalStakeValueEth += derivativeReceivedEthValue;
```

The code above does the following:

1. deposit() is called for each derivative. For rETH, ethAmount is deposited into rocket pool if possible.
2. ethPerDerivative() is called again to determine the amount of ETH deposited.

However, as the second call to ethPerDerivative() is after a deposit, it is possible for both calls to return different prices.

**Impact:** When staking user might get incorrect amounts `SafEth` tokens, causing them to loose or unfairly gain Eth.

**Proof Of Concept:**

Consider the following scenario:

1. Lets say, Rocket Pool starts with a following:
   * depositPoolBalance = 2000
   * maximumDepositPoolSize = 2004 ether
2. A user comes an wants to stake 10 ether:
   * As all three derivatives have equal weights, 3.33 ETH will be allocated to each derivative so 3.33 ETH is allocated to rETH derivative contract
   * In the firstcall to `ethPerDericative()`:
     * As `depositPoolBalance + 3.33 ETH` is less than `maximumDepositPoolSize`, so `poolCanDeposit()` returns `true`.
     * Thus, ethPerDerivative() returns rocket pool's rETH price.
   * 3.33 ETH is now deposited into the rocket pool:
     * depositPoolBalance = 5000 ether + 3.3 ether = 5003.3 ether
   * Now, in the second call to `ethPerDerivative` in the staking function:
     * Now, depositPoolBalance + 3.33 ETH is more than maximumDepositPoolSize, thus poolCanDeposit() returns false. As such, ethPerDerivative() returns Uniswap's price instead.

In the scenario above, due to the difference in Rocket pool and Uniswap's prices, Bob will receive an incorrect amount of SafETH, causing him to unfairly gain or lose ETH.
    
 
**Recommended Mitigations:**

Consider making the following changes to stake():

1. Before the deposit, store the results of ethPerDerivative() for each derivative.
2. After the deposit, use the stored values instead of calling ethPerDerivative() again.


### [H-3] If the admin adds malfunctioning derivative it results in breaking of the protocol

**Description:** The problem is that if an incorrect address is used to add a new derivative, stake and unstake will revert every time and there's no logic to then remove that derivative. The only solution would be to upgrade the contract to include derivative removal functionality. To avoid this issue, derivative removal functionality should be included in the initial deployment.

**Impact:** Users wont be able to stake/unstake functions of the contract anymore

**Proof Of Code:** 

Both stake and unstake call .balance() on each derivative contract.

```solidity
// stake() - SafEth.sol:L71-75
for (uint i = 0; i < derivativeCount; i++)
	underlyingValue +=
		(derivatives[i].ethPerDerivative(derivatives[i].balance()) *
			derivatives[i].balance()) /
		10 ** 18;
```

```solidity
// unstake() - SafEth.sol:L113-116
for (uint256 i = 0; i < derivativeCount; i++) {
	// withdraw a percentage of each asset based on the amount of safETH
	uint256 derivativeAmount = (derivatives[i].balance() *
		_safEthAmount) / safEthTotalSupply;
```

If an incorrect address is added for the derivations the above functions will always revert and hence the protocol will stop working as expected.

**Recommended Mitigations:**
To mitigate the issues associated with adding corrupt, not supported, zero address, or duplicate derivatives, the following approaches can be taken:

1. Allow removal of derivatives: Modify the existing implementation by adding a function that enables the contract owner to remove derivatives from the protocol. This will allow for removing problematic derivatives that may have been accidentally added or are causing issues.

2. Implement positive validation and allow listing: Adopt a positive validation approach, where only whitelisted derivatives can be added to the protocol. Additionally, incorporate ERC165 standard for contract validation to ensure that only valid derivatives are added.


### [H-4] Price of `SfrxEth` derivative is calculated incorrectly

**Description:** The goal of the ethPerDerivative() function is to calculate the equivalent amount of ETH for one unit of sfrxETH (a derivative token representing ETH).

The process involves:
1. Converting 1 sfrxETH to frxETH using convertToAssets(10 ** 18). (Here, 10 ** 18 represents one unit of sfrxETH in its smallest unit.)
2. Adjusting the frxETH value by price_oracle, which gives the price of frxETH in ETH.

The price_oracle function in the Curve pool provides the exchange rate between frxETH and ETH. It is used to determine the value of frxETH in terms of ETH.

NOTE: The code assumes that price_oracle returns the value of 1 unit of frxETH in ETH (units: frxETH / ETH). But what actually happens is, price_oracle returns the reciprocal value, i.e., the value of 1 unit of ETH in frxETH (units: ETH / frxETH). The protocol will value the sfrxETH derivative token incorrectly. The valuation can either be too low or too high. This leads to accounting errors when calling the SafEth.stake function which is where the ethPerDerivative function is used.
This means that the amount of SafETH that the user is minted is either too low or too high.

**Impact:** `SfrxEth` derivative is calculated incorrectly which is being used in `stake` and `unstake` functions.

**Proof Of Code:**

1. Let's have a look at the value returned from the price_oracle function from the Curve pool. You can find the current value here: (https://etherscan.io/address/0xa1F8A6807c402E4A15ef4EBa36528A3FED24E577/#readContract) 

Check the value of `price_oracle` in the above webpage, that would be coming approx as 998607137138305022.

2. When looking at the Curve front end swap we can see that for 1 frxETH we get ~ 0.998 ETH:

So, the unit is 0.998 ETH / frxETH.

However, the ethPerDerivative function  performs the following calculation:

```solidity
return ((10 ** 18 * frxAmount) /
    IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle());
```

So say frxAmount is 1e18 then with the above price we get a result that is >1e18. Obviously this is wrong. For 1 frxETH we should get less than 1 ETH (i.e. <1e18).

We can also see this issue by looking at the units of the calculation: frxETH / (ETH / frxETH) = frxETH * (frxETH / ETH) = frxETH^2 / ETH. This is obviously wrong.

The correct calculation would be (IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle() * frxAmount) / 1e18.

**Recommended Mitigations:**

Change these lines:

```solidity
        return ((10 ** 18 * frxAmount) /
            IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle());
```

to:

```solidity
        return (frxAmount * IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle() / 10 ** 18);
```


### [H-5] Reth poolPrice calculation may OverFlow

**Description:** The Reth derivative contract implements the `poolPrice` function to get the spot price of the derivative asset using a uniswapV3 pool. The function queries the pool to fetch the `sqrtPriceX96` and does the following calculation:

```solidity
function poolPrice() private view returns (uint256) {
    address rocketTokenRETHAddress = RocketStorageInterface(
        ROCKET_STORAGE_ADDRESS
    ).getAddress(
            keccak256(
                abi.encodePacked("contract.address", "rocketTokenRETH")
            )
        );
    IUniswapV3Factory factory = IUniswapV3Factory(UNI_V3_FACTORY);
    IUniswapV3Pool pool = IUniswapV3Pool(
        factory.getPool(rocketTokenRETHAddress, W_ETH_ADDRESS, 500)
    );
    (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
@>    return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);
}
```

The main issue here is that the multiplications in the expression `sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)` may eventually overflow.

The Process is as follows:

1. Square the sqrtPriceX96 value:

```solidity
	uint256 squaredPrice = sqrtPriceX96 * sqrtPriceX96;
```

* Squaring the 96-bit fixed-point value produces a 192-bit fixed-point number (in theory).
* In practice, this can easily exceed the uint256 limit (especially when sqrtPriceX96 approaches its maximum value).

2. Scale by 1e18:

```solidity
uint256 scaledPrice = squaredPrice * 1e18;
```

* Adding another scaling factor (1e18) multiplies the result further, increasing the likelihood of an overflow.

3. Shift down by 96 * 2:

```solidity
uint256 finalPrice = scaledPrice >> (96 * 2);
```
* The final adjustment scales the number back down. However, this occurs after the overflow, so it cannot fix the issue.



**Impact:** Overflow will result in reverting the function.

**Recommended Mitigations:** Use Chainlink to get price instead of poolPrice.


### [H-6] `WstEth::ethPerDerivative` calculates price in terms of StEth instead of Eth.

**Description:** `ethPerDerivative()` is used to determine the derivative's value in terms of ETH:

```solidity

    /**
        @notice - Get price of derivative in terms of ETH
     */
    function ethPerDerivative(uint256 _amount) public view returns (uint256) {
         //@audit amount is not used in the func
        return IWStETH(WST_ETH).getStETHByWstETH(10 ** 18); 
        //@audit not getting in terms of eth but instead wsteth
        
    }
```

As `getStETHByWstETH()` converts `WstETH` to `stETH`, `ethPerDerivative()` returns the price of `WstETH` in terms of `stETH` instead. This might cause loss of assets to users as `ethPerDerivative()` is used in staking calculations.

This assumption also occurs in `withdraw()`:

```solidity
uint256 stEthBal = IERC20(STETH_TOKEN).balanceOf(address(this));
IERC20(STETH_TOKEN).approve(LIDO_CRV_POOL, stEthBal);
uint256 minOut = (stEthBal * (10 ** 18 - maxSlippage)) / 10 ** 18;
IStEthEthPool(LIDO_CRV_POOL).exchange(1, 0, stEthBal, minOut);
```

minOut, which represents the minimum ETH received, is calculated by subtracting slippage from the contract's stETH balance. If the price of stETH is low, the amount of ETH received from the swap, and subsequently sent to the user, might be less than intended.

**Impact:** Incorrectly calculate the value in Eth and its being used by functions such as `withdraw`.

**Recommended Mitigations:** Use a price oracle to approximate the rate for converting stETH to ETH.


### [H-7] `Staking` and `Unstaking` DOS due to external conditions

**Description:** stake() and unstake() might permanently revert for a prolonged period of time.

In the SafEth contract, `stake()` calls `deposit()` for each dericative in a loop, as such if any deposit() or withdraw() reverts for any derivative, stake() and unstake() will fail. 

```solidity
for (uint i = 0; i < derivativeCount; i++) {
    // Some code here...
    
    uint256 depositAmount = derivative.deposit{value: ethAmount}();
    
    // Some code here...
}
```

```solidity
for (uint256 i = 0; i < derivativeCount; i++) {
    // Some code here...

    derivatives[i].withdraw(derivativeAmount);
}
```

External condition can cause stake() and unstake() to permanently revert for an prolonged period of time, as it is possible for deposit() and withdraw() to revert due to unchecked external conditions:

* Reth
  * [https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/deposit/RocketDepositPool.sol#L77](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/deposit/RocketDepositPool.sol#L77)
  * [Withdrawals will fail if rocket pool does not have sufficient ETH.](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/token/RocketTokenRETH.sol#L110-L114)

* WstETH
  * stETH has a daily [staking](https://docs.lido.fi/guides/steth-integration-guide/#staking-rate-limits) limit, which could cause deposits to fail.

If any of the external conditions above occurs, their respecitve function will be DOSed.


**Impact:** `stake` and `unstake` might permanently revert for a prolonged period of time.

**Recommended Mitigations:**
Check for the external conditions mentioned and handle them. For example, swap the staked ETH for derivatives through Uniswap in deposit(), and vice versa for withdraw().


# Medium 

### [M-1] Division before Multiplication is observes when calculating `minOut` 

**Description:** When Calcuting the minOut before doing trade, division before multiplication truncate minOut and incurs heavy precision loss, then very sub-optimal amount of the trade output can result in loss of fund from user because of the insufficient slippage protection.

**Impact:** Division before Multiplication leads to precision loss resulting minOut to be `Zero`.

**Proof Of Concept:**

Lets look at the below implementation of setting `slippage` in `Reth.sol`

```solidity
/**
	@notice - Owner only function to set max slippage for derivative
	@param _slippage - new slippage amount in wei
*/
function setMaxSlippage(uint256 _slippage) external onlyOwner {
	maxSlippage = _slippage;
}
```
Now, the above slippage setting affects this code:

```solidity
@>> uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
	((10 ** 18 - maxSlippage))) / 10 ** 18);
IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
uint256 amountSwapped = swapExactInputSingleHop(
	W_ETH_ADDRESS,
	rethAddress(),
	500,
	msg.value,
	minOut
);
```

Now, from above we can see that `Divsion befoe Multiplication` happens in below line of code:

```solidity
uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
	((10 ** 18 - maxSlippage))) / 10 ** 18);
```

For example, if maxSlippage is 10 ** 17

(10 ** 18 - 10 ** 17) / (10 ** 18) = 0

Then minOut is 0, slippage control is disabled because of the division before multipcation.


**Recommended Mitigations:** Don’t divide before multiply.


### [M-2] `sfrxEth` may revert on redeeming non-zero amount

**Description:**  The issue described relates to a revert condition that occurs during the unstaking process due to insufficient precision in asset calculations, potentially leading to a deadlock scenario where users cannot unstake their funds.

**Impact:** Unstake function can get blocked

**Proof Of Concept:**

During the unstake function, the contract withdraws a proportional amount of each derivative's underlying assets to convert the user's safETH into ETH.
For each derivative (e.g., sFrxEth), the withdraw function of the derivative is called. 

For the sFrxEth derivative, the withdraw function internally calls `IsFrxEth(SFRX_ETH_ADDRESS).redeem(_amount, address(this), address(this));`.
This function calculates the redeemable assets using `previewRedeem(shares)` before proceeding. If previewRedeem returns 0, the function reverts with the error "ZERO_ASSETS".

This function may revert if `_amount` is low due to the following line in redeem (where _amount is shares):
* `require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");`
* `previewRedeem(uint256 shares)` returns `convertToAssets(shares)` which is the shares scaled by the division of total assets by total supply:
   `shares.mulDivDown(totalAssets(), supply)`.

The edge case:

If the _amount i.e the shares, very small (e.g., 1), the result of convertToAssets(shares) can be 0 due to integer division.

For example:
* Total assets = 1,000,000
* Total supply = 2,000,000
  * _amount = 1

Result: 
convertToAssets(1)=1×1000000/2000000=0.5  which gets truncated to 0 in integer division. 

This causes the condition require(assets != 0) to fail, reverting the transaction. The revert from `sFrxEth.withdraw` propagates back to the `unstake` function in SafEth at line 118, causing the entire unstake operation to fail

**Recommended Mitigations:** In SfrxEth.withdraw check if IsFrxEth(SFRX_ETH_ADDRESS).previewRedeem(_amount) == 0 and simply return if that’s the case.

### [M-3] Stakes get DOSed if the first Staker removes all the tokens and leave every little (1 wei) in the contract

**Description:** A user who is the first staker will be the sole holder of 100% of totalSupply of safETH. They can then unstake (and therefore burn) totalSupply - 1 leaving a total of 1 wei of safETH in circulation.

```solidity
        uint256 totalSupply = totalSupply();
        uint256 preDepositPrice; // Price of safETH in regards to ETH
        if (totalSupply == 0)
            preDepositPrice = 10 ** 18; // initializes with a price of 1
        else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```

With totalSupply = 1, we see that the above code block will execute the else code path, and that if underlyingValue = 0, then preDepositPrice = 0.

For a simple case, assume there is 1 derivative with 100% weight. Let’s use rETH for this example since the derivative can get its ethPerDerivative price from an AMM. In this case:

Assume the ethPerDerivative() value has been manipulated in the underlying AMM pool such that 1 derivative ETH is worth less than 1 ETH. eg: 1 rETH = 9.99…9e17 ETH
In this case, also assume that since there is 1 wei of safETH circulating, there should be 1 wei of ETH staked through the protocol, and therefore derivatives[i].balance() = 1 wei.

This case will result in underlyingValue += (9.99...9e17 * 1) / 10 ** 18 = 0.

We can see that it is therefore possible to cause a divide by 0 revert and malfunction of the stake() function.

**Impact:** Potential inability to stake (ie: DoS) if sole safETH user (ie: this would also make them the sole safETH holder) unstakes totalSupply - 1. 

**Recommended Mitigations:** Assuming the deployment process will set up at least 1 derivative with a weight, simply adding a stake() operation of 0.5 ETH as the first depositor as part of the deployment process avoids the case where safETH totalSupply drops to 1 wei.


### [M-4] Lack of Deadline for uniswap AMM

**Description:**  The ISwapRouter.exactInputSingle params (used in the rocketpool derivative) does not include a deadline currently.

```solidity
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
    .ExactInputSingleParams({
        tokenIn: _tokenIn,
        tokenOut: _tokenOut,
        fee: _poolFee,
        recipient: address(this),
        amountIn: _amountIn,
        amountOutMinimum: _minOut,
        sqrtPriceLimitX96: 0
    });
```

**Proof Of Code:**

The following scenario can happen:

1. User is staking and some/all the weight is in Reth.
2. The pool can’t deposit eth, so uniswap will be used to convert eth to weth.
3. A validator holds the tx until it becomes advantageous due to some market condition (e.g. slippage or running his tx before and frontrun the original user stake).
4. This could potentially happen to a large amount of stakes, due to widespread usage of bots and MEV.

**Impact:**  Because Front-running is a key aspect of AMM design, deadline is a useful tool to ensure that your tx cannot be “saved for later”.

**Recommended Mitigations:**  The Reth.deposit() function should accept a user-input deadline param that should be passed along to Reth.swapExactInputSingleHop() and ISwapRouter.exactInputSingle().

### [M-5] De-Pegs of Derivations can cause high slippage scenarios

**Description:** 

The Purpose of `rebalanceToWeights()`:
* Withdrawing the full balance from each derivative.
* Reallocating ETH to the derivatives according to the new weights.

```solidity
function rebalanceToWeights() external onlyOwner {
    uint256 ethAmountBefore = address(this).balance; // Records ETH balance before rebalancing.
    for (uint i = 0; i < derivativeCount; i++) { // Loops through derivatives.
        if (derivatives[i].balance() > 0) // Checks if a derivative has funds.
            derivatives[i].withdraw(derivatives[i].balance()); // Fully withdraws from the derivative.
    }
    // Re-enter positions based on new weights (code not shown).
}
```

The Problem in De-peg Scenarios:

* When one derivative token (e.g., frax-ETH) begins to de-peg, users will likely exchange it for WETH (flight to safety). This triggers a cascading sell pressure across the pool. This lead to below issues:
  1. Non-Localized Exits:  Instead of exiting only from the de-pegged derivative, the protocol withdraws from all derivative positions, including those not affected by the de-peg. This causes unnecessary sell pressure on non-de-pegged assets, widening slippage and resulting in more losses for depositors.
  2. Immediate Re-entry into New Positions:After exiting, the protocol immediately reallocates funds to new positions based on updated weights. If the de-peg triggers further instability (e.g., USDT after UST collapse), re-entering the market too quickly may cause additional losses. he protocol exposes depositors to high slippage and gas costs while chasing volatile prices during market stress.
  3. Inefficient Rebalancing: The protocol fully exits all positions and then re-enters them based on new weights. Even derivatives that don't require rebalancing are unnecessarily exited and re-entered. This creates large, redundant transactions, increasing slippage, gas fees, and execution risks.


**Impact:**
1. Causes unnecessary sell pressure on non-de-pegged assets.
2. The protocol exposes depositors to high slippage and gas costs while chasing volatile prices during market stress.
3. This creates large, redundant transactions, increasing slippage, gas fees, and execution risks


**Proof Of Code:**

Lets consider an Example Scenario:

Scenario Setup:

The pool consists of three derivatives:
* frax-ETH: 10% of the portfolio.
* stETH: 70% of the portfolio.
* rETH: 20% of the portfolio.

Target weights after a frax-ETH de-peg:
* stETH: 80%.
* rETH: 20%.

Current Implementation:

The protocol fully withdraws:
* 70% stETH.
* 20% rETH.
The protocol reallocates ETH based on the new weights:
* 80% stETH.
* 20% rETH.

Now, let me explain what have happend above:
* Only 10% frax-ETH needed to be removed and reallocated.
  * Instead, the protocol:
    * Sells the entire pool (including unaffected derivatives).
    * Reallocates based on new weights.
  * This incurs significant slippage costs and gas fees, especially in volatile markets where prices fluctuate quickly.

**Recommeded Mitigations:** 

Consider following improvements to rebalanceToWeights:

* Separate exits & entries. Split the functionality to exit and re-enter. In stressed times or fast evolving de-peg scenarios, protocol owners should first ensure an orderly exit. And then wait for markets to settle down before re-entering new positions

* Localize exits, ie. if derivative A is de-pegging, first try to exit that derivative position before incurring exit costs on derivative B and C

* Implement a marginal re-balancing - for protocol's whose weights have increased, avoid exit and re-entry. Instead just increment/decrement based on marginal changes in net positions

### [M-6] High Slippage possiblity due to swaps from uniswapV3 `rETH/WETH` pool

**Description:** rETH is acquired using the Uniswap rETH/WETH pool. This solution has higher fees and lower liquidity than alternatives, which results in more lost user value than other solutions.

The Uniswap rETH/WETH pool that is used in Reth.sol to make swaps has a liquidity of $5 million. In comparison, the Balancer rETH/WETH pool has a liquidity of $80 million. Even the Curve rETH/WETH pool has a liquidity of $8 million. The greater liquidity should normally offer lower slippage to users. In addition, the fees to swap with the Balancer pool are only 0.04% compared to Uniswap’s 0.05%. Even the Curve pool offers a lower fee than Uniswap with just a 0.037% fee

One solution to finding the best swap path for rETH is to use RocketPool’s RocketSwapRouter.sol contract swapTo() function. When users visit the RocketPool frontend to swap ETH for rETH, this is the function that RocketPool calls for the user. RocketSwapRouter.sol automatically determines the best way to split the swap between Balancer and Uniswap pools.

```solidity
 amountOut = ISwapRouter(UNISWAP_ROUTER).exactInputSingle(params);
```

**Proof Of Code:**

Pools that can be used for rETH/WETH swapping:

* Uniswap rETH/WETH pool: $5 million in liquidity
* Balancer rETH/WETH pool: $80 million in liquidity
* Curve Finance rETH/ETH pool: $8 million in liquidity

**Recommended Mitigations:** The best solution is to use the same flow as RocketPool’s frontend UI and to call swapTo() in RocketSwapRouter.sol. An alternative is to modify Reth.sol to use the Balancer rETH/ETH pool for swapping instead of Uniswap’s rETH/WETH pool to better conserve user value by reducing swap fees and reducing slippage costs.

### [M-7] No Derivatives can result in freezing of ETH in stake contract

**Description:** If the `SafEth` contract is deployed and there are no derivatives added to the contract and a user tries to call the stake function, then this could result in a loss of funds for the user.

Proof of Concept
