


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
