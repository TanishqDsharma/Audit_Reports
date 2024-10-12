# High

# Medium

# Low

### [L-1] Missing Zero Address Check in `Lender::setFeeReceiver` function

**Description:** The function doesn't provide any checks for zero address.

```solidity
 function setFeeReceiver(address _feeReceiver) external onlyOwner {
        feeReceiver = _feeReceiver;
    }
```

**Impact:** If the `feeReceiver` is having zero address as the admin passed a zero address for feeReciever, then the functions such as `borrow`, `repay`, `giveLoan`, `buyLoan` etc will revert transactions.

**Recommended Mitigations:** Add a zero address check in the function

```diff
function setFeeReceiver(address _feeReceiver) external onlyOwner {
+   if (_feeReceiver == address(0)) revert ZeroAddress();
    feeReceiver = _feeReceiver;
    }
```

### [L-2] `Lender::giveLoan` has unchecked array lengths
**Description:** `Lender::giveLoan` is a function which takes  `loanIds` and `poolIds` arrays as input but doesn't checks length of these arrays. The function processes an array of loan IDs and corresponding pool IDs, assuming both arrays have the same length and that each index in loanIds maps to the correct poolId

For example: Lets take a case where lengths will be different such as: 

If the lender forgets to include a poolId, there will be an unmatched loanId, resulting in loanIds.length being greater than poolIds.length . This leads to an out-of-bounds error when accessing poolIds[i], causing the function to revert.


**Impact:**
Lender fails to give loan to another lender as the function breaks because of the `array out-of-bounds access` error.

**Proof Of Concept:**

<details>

<summary>Add the below test to `Lender.t.sol`</summary>

```solidity
function test_giveLoanDifferentLength() public {
    test_borrow();

    vm.startPrank(lender2);
    Pool memory p = Pool({
        lender: lender2,
        loanToken: address(loanToken),
        collateralToken: address(collateralToken),
        minLoanSize: 100 * 10 ** 18,
        poolBalance: 1000 * 10 ** 18,
        maxLoanRatio: 2 * 10 ** 18,
        auctionLength: 1 days,
        interestRate: 1000,
        outstandingLoans: 0
    });
    lender.setPool(p);

    uint256[] memory loanIds = new uint256[](1);
    loanIds[0] = 0;
    bytes32[] memory poolIds = new bytes32[](0);

    vm.startPrank(lender1);
    lender.giveLoan(loanIds, poolIds);
}
```
</details>

**Recommended Mitigations:**
Add a condition to checks if the lenghts of the passed arrays are equal or not, and if not equal revert the transaction.

### [L-3] Events are not being emitted when changes are made to state variables

**Description:** The problem lies in `Lender` contract where multiple functions do not emit events. 

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L84-L87>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L92-L95>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L100-L102>

**Impact:**
Emitting events in smart contracts is a fundamental practice that enhances communication between the blockchain and external applications, improves user interaction, and ensures that changes to state variables are efficiently and transparently communicated to all interested parties.

**Recommended Mitigations:**
Emits events where state changes are being done.


### [L-4] `Lender::_calculateInterest` function  in not correctly calculating the interest

**Description:** The _calculateInterest function is designed to calculate the interest accrued on a loan, as well as any associated fees. It takes a loan structure as input and returns both the calculated interest and fees. The main issue lies within the below formula that is being used for calculating interest:

```solidity
interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;
```
The main problem here is the divisin by 365 days.

**Impact:**

Can lead to wrong intrest calculations and losses to lenders for providing there assests and gaining almost negligble interest on it.

**Recommended Mitigations:**
Change the formula so that it can calculat interest correctly.



### [L-5] Missing Zero Amount Checks in `Beedle` and `Staking` contract

**Description:** Some functions in `Beedle` such as mint, _burn, _mint and some functions in `Staking` such deposit, withdraw are not implementin checks for passing zero value as amount.

**Impact:**
Users will be able to run the functions successfully and no there would be unnecessary gas cost. And also state update will cost gas.

**Recommended Mitigations:**
Consider checking if `_amount` is zero or not in the beginning of the function.


### [L-6] Using `block.timestamp` for setting timestamps
 
**Description:** Using `block.timestamp` for recording the timestamp in borrow function can introduce a vulnerablity. Malicious miners could potentially manipulate the `block.timestamp`, leading to inaccuracies in the recorded borrowing timestamps. This can have implications for time-sensitive operations such as interest calculations, time-based auctions, and loan durations.

```solidity
 Loan memory loan = Loan({
                lender: pool.lender,
                borrower: msg.sender,
                loanToken: pool.loanToken,
                collateralToken: pool.collateralToken,
                debt: debt,
                collateral: collateral,
                interestRate: pool.interestRate,
 @>               startTimestamp: block.timestamp,
                auctionStartTimestamp: type(uint256).max,
                auctionLength: pool.auctionLength
            });
```

**Impact:** Manipulation with block.timestamp can result in wrong interest calculation, can affect time based auctions etc

**Recommended Mitigations:**
Consider using `block.number` or other block-related variables, which are less susceptible to manipulation by miners, for any time-dependent functionalities or decision-making processes. 


### [L-7] `Staking` contract missing TKN!=Weth check

**Description:** The staking contract doesn't implement any checks to verify that the `TKN` and `WETH` contract addresses should not be the same.

**Impact:** Using the same address may introduce vulnerabilities, where a user could exploit the contract by tricking it into counting the same tokens multiple times or causing unintended behaviors in the logic. It also creates confusion

**Recommended Mitigations:**

Implement the below check inside the constructor:

```diff
constructor(address _token, address _weth) Ownable(msg.sender) {
+++   require(_token != _weth, "TKN and WETH must be different addresses");
    TKN = IERC20(_token);
    WETH = IERC20(_weth);
}

```

### [L-8] Staking Rewards Dilution

**Description:** In a staking contract, users are typically rewarded for depositing and holding tokens within the contract. However, if users can transfer tokens directly to the staking contract without using the designated deposit function, it can lead to unintended consequences. This vulnerability allows individuals to dilute the staking rewards pool without contributing to it properly, thereby negatively impacting legitimate stakers.

Example: Users can bypass the staking logic by directly sending staking tokens to the staking contract using the standard ERC20 transfer function, such as 

```solidity
token.transfer(address(staking), amount)
```
When this happens, the staking contract does not have a mechanism to recognize that these tokens have been deposited for staking purposes since the deposit function—which typically handles the updating of user balances, reward calculations, and other necessary bookkeeping—was not called.

**Impact:** When rewards are distributed, they are now shared among a larger number of tokens (150), reducing the rewards per token for the original stakers.

**Recommended Mitigations:**

Use Global Variable that keeps track of tokens that are deposit legitimately into the contract. This variable should be updated in both the deposit and withdraw functions to ensure accurate tracking of the actual staked balance. 


### [L-9] `Lender::refinance` emits incorrect parameters

**Description:** The `repaid` event  in the refinance function emits wrong parameters

```solidity
 emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
@>                debt,
@>                collateral,
                loan.interestRate,
                loan.startTimestamp
            );
```

**Impact:** Events with wrong parameter can be misinterpreted by the frontend.

**Recommended Mitigations:**

```diff
 emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
--              debt,
--              collateral,
++              loan.debt,
++              loan.collateral,
                loan.interestRate,
                loan.startTimestamp
            );
```

### [L-10] Rounding error risk in `Lender::borrow` function

**Description:** The formula that is used for calculating fees in borrow function is vulnerable to rounding errors. 

```solidity
uint256 fees = (debt * 50) / 10000;
```

**Impact:** If debt values are low enough, you could end up with a situation where fees are effectively zero, which may not be the intended outcome.

**Proof of Concept:**

Scenario 1:
* Let’s say debt is 19999. Then:
  * (19999 * 50) = 999950
  * 999950 / 10000 = 99.995 → rounded down to 99.

Scenario 2:
* If debt is 199:
  * (199 * 50) = 9950
  * 9950 / 10000 = 0.995 → rounded down to 0.

**Recommended Mitigation:** Import & use fixed-point arithmetic math libraries

### [L-11] Users can be prevented by Operator from borrowing from a pool

**Description:** When a user tries to borrow from a pool, another malicious user can frontrun the user by borrowing a large amount of tokens from the pool leaving it almost out of liquidity. Now,  when the user transaction goes for taking the loan it gets reverted as the malicious user already taken the loan almost leaving no funds resulting transaction being reverted.

**Impact:** This can be treated as DOS since customers are not able to take any loan and it also a problem for lenders as they wont be able to earn any interest on the loan as the malicious user immediately repays it.

**Recommended Mitigations:**
Include a minimum interest fee if the loan duration is less than a certain time to discourage this behavior.

### [L-12] Weird ERC20's that can pause the protocol

**Description:** If tokens used for collateral or Loan are ERC20's that are having pausible feature such as USDT. Now, if for any reason these gets paused, the protocol will not work as expected as users might be using ERC20's that are pausable for collateral and Loan purposes.

**Impact:**  If the token is paused then transfers of tokens into and out of the protocol are not possible, which impacts ability to deposit, ability to pay back, ability to move loans and all other such related functionality depending on transfer, transferFrom etc functions.

**Proof Of Concept:** 

Some of the few Instances where token pausing can affect the protocol there are more:

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L152>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L159>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L187>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L203>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L267>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L269>

**Recommended Mitigations:** It may be ideal to whitelist allowed tokens for loanToken and collateralTokens and not allow callback, hook, tokens such as ERC777, ERC1363 etc

### [L-13] Malicious Users can DOS lenders by not allowing them to withdraw

**Description:** A malicious user can frontrun the `Lender` when lender is trying to withdraw funds by taking all the liquidity as loan. Then later the user can repay the loan. Since, the duration of the loan is small the intrest will be ngeligible and the pool lender cannot withdraw there funds.

**Impact:** Malicious user can make Lenders not able to withdraw there funds, hence causing a dos for the withdrawal service.

**Recommended Mitigation**: Add a minimum interest fee, despite the length of the borrow, in order to make this attack costly and prevent it

### [L-14]  Users can prevent Pool Owners to change `minLoanSize` and `auctionLength`

**Description:** A malicious user can prevent the pool Owner or we can say the lender to change `minLoanSize` and `auctionLength`. Owners of the pools need to use the `Lender::setPool` function to change the `minLoanSize` and `auctionLength`.

The problem occurs during the below check:

```solidity
        if (p.outstandingLoans != pools[poolId].outstandingLoans)
            revert PoolConfig();
```

While this makes sure the accounting is correct, it opens up an attack vector. Since this is the only way to change the values of `minLoanSize` and `auctionLength`, a malicious user may be monitoring the mempool for such transactions and front-run them and borrow/ repay in order to change the value of `outstandingLoans`. As by changing it the transaction will revert

**Impact:** Lenders or Pool Owner cannot change the values of `minLoanSize` and `auctionLength`

**Recommended Mitigations:** Making separate functions that allows pool Owners to update the values of `minLoanSize` and `auctionLength`
