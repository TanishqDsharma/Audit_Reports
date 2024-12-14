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








