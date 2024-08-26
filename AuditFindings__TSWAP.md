# High

### [H-2] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users resulting in lost fees

**Description:**
The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of output tokens. However, the function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by 10_000 instead of 1_000.

**Impact:** Protocol takes more fees than expected from users.

**Recommended Mitigation:**

```diff
function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
        return
-            ((inputReserves * outputAmount) * 10000) /
+ 	     ((inputReserves * outputAmount) * 1000) /
            ((outputReserves - outputAmount) * 997);
    }
```

# Medium

### [M-1] `TSwapPool::Deposit` is missing deadline check causing transactions to complete even after the deadline

**Description:**
The deposit function accepts a deadline parameter which according to the documentation is "The deadline for the transaction to be completed by." However, this parameter is never used.
As a result, operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavourable

**Impact:** Transactions could be sent when the market conditions are unfavourable to deposit, even when adding when adding a deadline parameter.

**Proof Of Concept:** The `deadline` parameter is unused.

**Recommended Mitigations** Consider making following change to the function

```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        //@audit - this param is not being used in the function
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
+	revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
```
# Low

### [L-1] `TSwapPool::LiquidityAdded` event has parameters out of order

**Description:** When `Liquidity Added` event is emitted in the `TSwapPool::_addLiquidityMintAndTransfer` function, it logs values in an incorrect order.
The `poolTokensToDeposit` value should go in the third parameter position, whereas the `wethToDeposit` value should go second.

**Impact:** Event Emission is incorrect, leading to offchain fucntions potentially malfunctioning.

**Proof Of Concept:**

```diff
- emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+ emit LiquidityAdded(msg.sender, wethDeposit, poolTokensToDeposit);
```

### [L-2] Default Value returned by `TSwapPool::swapExactInput` results in incorrect return value given

**Description:** The `swapExactInput` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return `output` it is never assigned a value nor uses an explicit return statement

**Impact:** The return value will always be zero giving incorrect infromation to the caller.


**Recommended Mitigations:**
```diff
uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
+        output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);


-        if (outputAmount < minOutputAmount) {
-            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);}
+ 	 if (output < minOutputAmount) {
+            revert TSwapPool__OutputTooLow(output, minOutputAmount);}


-        _swap(inputToken, inputAmount, outputToken, outputAmount);
+        _swap(inputToken, inputAmount, outputToken, output);
```
# Informational

### [I-1] `PoolFactory::PoolFactory__PoolDoesNotExist` error is not used and should be removed

```diff
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking Zero Address checks 
```diff
constructor(address wethToken) {
+       if(wethToken==address(0)){
+           revert();
        i_wethToken = wethToken;
    }
```

### [I-3] `PoolFactory::createPool` should user `.symbol()` instead of `.name()`

```diff
-        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-4] Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>4 Found Instances</summary>


- Found in src/PoolFactory.sol [Line: 38](src/PoolFactory.sol#L38)

	```solidity
	    event PoolCreated(address tokenAddress, address poolAddress);
	```

- Found in src/TSwapPool.sol [Line: 53](src/TSwapPool.sol#L53)

	```solidity
	    event LiquidityAdded(
	```

- Found in src/TSwapPool.sol [Line: 58](src/TSwapPool.sol#L58)

	```solidity
	    event LiquidityRemoved(
	```

- Found in src/TSwapPool.sol [Line: 63](src/TSwapPool.sol#L63)

	```solidity
	    event Swap(
	```

</details>
