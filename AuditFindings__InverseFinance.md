# Low

### [L-1] Signature Malleablity in DBR.sol and Market.sol functions:

**Description:** In SECP256K1. each point on the elliptic curve has two valid `s` values (one in the lower half of the curve and one in the upper half) that can represent the same signature. Without, enforcing that `s` must fall within the lower half of the curve, a malicious actor could potentially provide an alternative version of a signature with same effect. Although Ethereum’s EIP-2 (introduced in the Homestead upgrade) requires that s be in the lower range for most signature checks, the ecrecover function does not enforce this check.

```solidity
 address recoveredAddress = ecrecover(
                keccak256(
                    abi.encodePacked(
                        "\x19\x01",
                        DOMAIN_SEPARATOR(),
                        keccak256(
                            abi.encode(
                                keccak256(
                                    "WithdrawOnBehalf(address caller,address from,uint256 amount,uint256 nonce,uint256 deadline)"
                                ),
                                msg.sender,
                                from,
                                amount,
                                nonces[from]++,
                                deadline
                            )
                        )
                    )
                ),
                v,
                r,
                s
```

**Impact:** Signature can be reused

**Recommended Mitigation:** 
* Use OpenZepplin Contracts:
  https://github.com/OpenZeppelin/openzeppelin-contracts/blob/7201e6707f6631d9499a569f492870ebdd4133cf/contracts/utils/cryptography/ECDSA.sol#L138-L149 or implement a check similar to this in the above code

### [L-2] Lack of `Zero Address` checks:

**Description:** There are many instances in the code where its lacking zero address check.

BorrowController.sol: LINE 26 

```solidity
    function setOperator(address _operator) public onlyOperator { operator = _operator; }
```

BorrowController.sol: LINE 14

```solidity
        operator = _operator;
```

SimpleERC20Escrow.sol: LINE 28

```solidity
        token = _token;
```
**Impact:** Some functions that cannot handle zero values can result in unexpected scenarios.

**Recommended Mitigations:** Implement Zero Address checks


### [L-3] Use of `tx.origin` in `BorrowController.sol:`

**Description:** `tx.origin` is a global variable in Solidity that returns the address of the account that sent the transaction.

```solidity
 function borrowAllowed(address msgSender, address, uint) public view returns (bool) {
        if(msgSender == tx.origin) return true; //@audit EOA bypass
        return contractAllowlist[msgSender];
    }
```

### [L-4] Solidity compiler is outdated and not specific

**Description:**
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. So, its recommended to user newer or latest versions:
The minimum required version must be 0.8.17; otherwise, contracts will be affected by the following important bug fixes.


Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of pragma solidity ^0.8.13;, use pragma solidity 0.8.13;

```solidity
pragma solidity ^0.8.13;
```

**Recommended Mitigations:** User newer or latest solc versions and use a specific compiler version also.

### [L-5] Ownership Transfer without any confirmation

**Description:**

```solidity
    function setGov(address _gov) public onlyGov { gov = _gov; }
```
The primary issue with this code snippet is that,it immediately transfers governance to the new address without any confirmation or acknowledgment from the new governance address (_gov). This approach has several risks:

1. Irreversible Ownership Loss: If an invalid or incorrect address is accidentally set as _gov (such as the zero address or an address controlled by mistake), there is no way to revert or recover the governance role. This could result in the contract becoming "orphaned," with no valid owner or governor to manage it.

2. Human Error: Even minor mistakes like typos can lead to setting an unintended address as the new owner. In a direct transfer approach, these mistakes can have critical, sometimes irreversible consequences for contract governance.


### [L-6] `Fed::contraction` function missing `Market Paused` check

**Description:** In the Fed contract, during the expansion method is checked that the market is not paused, this requirement is not done during the contraction.

**Recommended Mitigation:**

```solidity
  function contraction(IMarket market, uint amount) public {
        require(msg.sender == chair, "ONLY CHAIR");
        require(dbr.markets(address(market)), "UNSUPPORTED MARKET");
+       require(!market.borrowPaused(), "CANNOT EXPAND PAUSED MARKETS");
        uint supply = supplies[market];
        require(amount <= supply, "AMOUNT TOO BIG"); // can't burn profits
        market.recall(amount);
        dola.burn(amount);
        supplies[market] -= amount;
        globalSupply -= amount;
        emit Contraction(market, amount);
    }
```

### [L-7] Avoid Using Magic Numbers:

**Description:** `10,000` its being used in many instances in the code, it should be assigned to some variable and that variable should be used instead of this number

```solidity
        require(_collateralFactorBps < 10000, "Invalid collateral factor");
        require(_liquidationIncentiveBps > 0 && _liquidationIncentiveBps < 10000, "Invalid liquidation incentive");
        require(_replenishmentIncentiveBps < 10000, "Replenishment incentive must be less than 100%");
```

**Recommended Mitigations:** Use well defined variable names instead of numbers directly

### [L-8] Avoid Using Harcoded Values:

**Description:** It is not good practice to hardcode values, but if you are dealing with addresses much less, these can change between implementations, networks or projects, so it is convenient to remove these values from the source code.

```solidity
    IERC20 public immutable dola = IERC20(0x865377367054516e17014CcdED1e7d814EDC9ce4);
```

**Recommended Mitigations:** Instead of hardcoding the value, some way should be implemented to take different chains into account.

### [L-9] Avoid use of same code in mulitple parts

**Description:** The `Oracle::getPrice` and `Oracle::viewPrice` methods of the Oracle contract are very similar, the only difference being the following piece of code:

```solidity if(todaysLow == 0 || normalizedPrice < todaysLow) {
                dailyLows[token][day] = normalizedPrice;
                todaysLow = normalizedPrice;
                emit RecordDailyLow(token, normalizedPrice);
            }
```

**Recommended Mitigations:** Use modifiers for re-using the code


### [L-10] Not all functionalites been implemented

**Description:** The code that contains “open todos” reflects that the development is not finished and that the code can change a prior release, with or without audit.

```solidity
constructor(IXINV _xINV) {
        xINV = _xINV; // TODO: Test whether an immutable variable will persist across proxies
    }
```

### [L-11] Integers values are not checked

**Description:**

In `DBR.sol`, when ` replenishmentPriceBps = _replenishmentPriceBps;` is assigned to a value its not being checked that is should not be equal to zero.

In `Market.sol`, `replenishmentIncentiveBps` is not checked to be > 0 during the constructor, nevertheless it’s checked in `setReplenismentIncentiveBps`

### [L-12] Events are not being emitted on every state change

**Description:** The Market.pauseBorrows, Market.setLiquidationFeeBps, Market.setLiquidationIncentiveBps, Market.setReplenismentIncentiveBps, Market.setLiquidationFactorBps, Market.setCollateralFactorBps, Market.setBorrowController, Market.setOracle methods do not emit an event when the state changes, something that it’s very important for dApps and users.

### [L-13] Oracle not compatible with tokens of 19 or more decimals

Keep in mind that the version of solidity used, despite being greater than 0.8, does not prevent integer overflows during casting, it only does so in mathematical operations.

In the case that feed.decimals() returns 18, and the token is more than 18 decimals, the following subtraction will cause an underflow, denying the oracle service.

```solidity
    uint8 feedDecimals = feeds[token].feed.decimals();  // 18 => [ETH/DAI] https://rinkeby.etherscan.io/address/0x74825dbc8bf76cc4e9494d0ecb210f676efa001d#readContract
    uint8 tokenDecimals = feeds[token].tokenDecimals;   // > 18
    uint8 decimals = 36 - feedDecimals - tokenDecimals; // overflow
```

All pairs have 8 decimals except the ETH pairs, so a token with 19 decimals in ETH, will fault.


