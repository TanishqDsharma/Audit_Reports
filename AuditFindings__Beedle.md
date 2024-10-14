# High

# Medium

### [M-1] `Lender::_calculateInterest` Function is vulnerable to precision loss which allows makes `Lender::giveloan` to offer loans at less collateral

**Description:** The vulnerability stems from the multiple divisions happening within the _calculateInterest() function. Here's how the precision loss occurs:

```solidity
interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;
```
In the `Lender::giveLoan` function, `_calculateInterest()` is called to calculate the lender's interest and protocol fees. These values are then added to the loan's debt to compute the total debt that the pool must cover. However, due to the precision loss in `_calculateInterest()`, `lenderInterest` and `protocolInterest` could be much smaller than intended or even zero. This leads to an underestimation of totalDebt. 

```solidity
// calculate the interest
            (
                uint256 lenderInterest,
                uint256 protocolInterest
            ) = _calculateInterest(loan);
            uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
```

Incorrect calculation of totalDebt leads to incorrect calculation of loanRatio, allowing us to give loan to a pool that has a higher loan ratio than our current pool which is not the expected behavior of the protocol.

```solidity
            uint256 loanRatio = (totalDebt * 10 ** 18) / loan.collateral;
```

Example: 
Imagine you have a loan with a collateral value of 1000 tokens and a total debt of 600 tokens. The loan ratio would be 600/1000 = 0.6 or 60%. If the new pool has a maxLoanRatio of 70%, the loan can be transferred because 60% is lower than 70%.

However, due to precision loss, if the totalDebt is miscalculated as 400 tokens instead of 600, the loan ratio becomes 400/1000 = 0.4 or 40%. Now, even if the pool's actual maxLoanRatio is stricter, say 50%, the protocol might incorrectly allow the transfer because it thinks the loan ratio is 40%, even though it's actually riskier than it appears.

**Impact:** Loans can be given to other pools at less collateral

**Recommended Steps:** Therefore, verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary

### [M-2] `Lenders` can frontrun the users taking the loan and significantly increase the interest rate

**Description:** Whenever a user tries to take loan, the lender can front-run the user and significantly increase the intrest rate lets say to the `MAX_INTEREST_RATE` and turn the loan into conditions which the user would see as unacceptable. Any user can take a loan by specifying the pool, the borrow amount they want and the collateral they are willing to provide but the user cant specify the interest hes willing to take loan on. So, a lender can monitor the mempool for transactions for taking loan and frontrun those transactions.

**Impact:** Users will take laon at much higher interest rates.

**Recommended Mitigations:**

Add a param to the Borrow struct which is the max interest rate the user is willing to pay. Upon taking a borrow, make sure this value >= the interest rate of the pool.


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

### [L-15] `Lender::seizeLoan` function allows to seize the loan early

**Description:** Lenders cans seize the loan early before the auction time period ends. The problem lies in the below check:

```solidity
if (block.timestamp < loan.auctionStartTimestamp + loan.auctionLength) revert AuctionNotEnded();
``` 
This statement allows to end auction when the block.timestamp is equal to auctions end time but not greater than the auctions endtime. But we can see 'Lender::buyLoan' only considers end of the auction if blocktimestamp is greater than `loan.auctionStartTimestamp + loan.auctionLength` or we can auctions endtime.

**Impact:** Lenders can end the auction before it actually ends

**Recommended Mitigations:** 

Correct the check to:

```solidity
if (block.timestamp <= loan.auctionStartTimestamp + loan.auctionLength) revert AuctionNotEnded();
``` 

### [L-16] Event Based Rentrancy due to Callback Tokens:

**Description:** Callback functions that can reenter functions with events lead to Event Rentrancy. If a loan Token or collateral Token are callback tokens, when transferred out they may be sent to a contract that can callback into the same function before the first event is emitted. This can result in into missing data or incorrect events being emitted.
 
**Impact:**  This results in incorrect events and missed event emission information for offchain tooling, monitoring, analysis, front ends. Users may act on protocol on faulty information from these events

**Proof Of Code:**

1. After external call to transfer(), event borrowed is emitted 
<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L277>

2. After external call to transfer collateral tokens to borrower lines 329, event repaid 
<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L333>

3.After external call to transfer() to feeReceiver lines 403, event repaid is emitted
<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L405>

**Recommended Mitigations:**

* It is advised to follow Checks, Effects and Interaction (CEI) pattern
* Instead of allowing all tokens to be loan and collateral tokens whitelist some tokens and use those only for loans and collateral


### [L-17] L2 Sequencer dependent functions

**Description:** There are some function that are using some parameters such as `block.timestamp` which depends on the L2 sequencer and can cause protocol not working as expected. If L2 sequencer goes down (e.g., during upgrades, bugs, or maintenance), the protocol will be unable to properly calculate interest and other time-dependent operations like auctions.  This makes the system vulnerable during such downtimes, potentially causing errors in functions like `borrow`, `refinance`, and `buyLoan`.

**Impact:** It can make the protocol function in unexpected way such some functions might not work as expected when the sequencer goes down.

**Proof Of Concept:** L2 sequencer downtimes are not hypothetical; they have occurred in practice. Here are some real-world incidents:

1. Optimism Bedrock Upgrade: During this upgrade, the sequencer could not process transactions for several hours, causing downtime for all transactions dependent on block.timestamp on the network.

Impact: Users on Optimism could not interact with dApps, including those involving lending or auctions, because the sequencer was unavailable.

2. Arbitrum Sequencer Bug: A bug in the Arbitrum sequencer caused a temporary pause in transaction processing, which meant that no new blocks were generated, and smart contracts relying on block.timestamp were stuck.

Impact: dApps on Arbitrum were effectively frozen, with critical operations such as interest accrual, auctions, and liquidations being affected.

**Recommended Mitigations:**
1. Chainlink provides a Sequencer Uptime Feed that can be integrated into smart contracts to check if the L2 sequencer is currently up or down. This feed reports the status of sequencers on popular L2s like Arbitrum and Optimism.
2. Implementation of Extra Time Buffer

### [L-18] `Lender::refinance` function dos user for refinancing in the same pool

**Description:** When a user wants to refinance in the same pool they might not able to do so becuase of the below conditional check:

```solidity
            if (pool.poolBalance < debt) revert LoanTooLarge();
```
The above check checks the current poolBalance against the users debt which might be greater than than poolBalance mostly.

For example: Intially balance of the pool was 1000 USDC, one borrower came and took loan of 900 USDC now the Lender updated the loan terms and offering loan at lowers interest in this case the user might want to refinnace the loan but they cant do it as refinancing the loan in the same pool for 1000 USDC will revert the transaction as the current poolBalance is 100 USDC which is less than debt which will make the above check true and revert the transaction.

**Impact:** Users will not be able to refinnace the loans from the same pool.

**Recommended Mitigations:**

Restructure the refinance function

### [L-19] Fees generated can be sent to any address

**Description:** The `feeReceiver` is having value of `msg.sender` assigned to it inside the constructor instead of the `fees` contract. By not initializing the `feeReceiver` variable, the fees can be sent to the contract owner and not to the Fees.sol contract that allows the exchange of tokens and take it to the staking contract.

**Impact:** Wont be able to user `Fees.sol` and `Staking.sol` contracts, preventing the correct functioning of the `Lender` contract


**Recommended Mitigations:** Assign the address of fees contract to `feeReceiver` instead of passing the `msg.sender`value.

### [L-20] `Lender::buyLoan` reverts if the timelasped is very less from the start of the auction

**Description:** The issue in the `buyLoan` function stems from the calculation of `currentAuctionRate` during the refinance auction. The logic effectively prevents anyone from purchasing the loan at the start of the auction due to how the `currentAuctionRate` is derived.

```solidity
uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) / loan.auctionLength;
```

If timeElapsed is very small, such as at the beginning of the auction, the result of this calculation will be too low. 

**Impact:**
This leads to the `buyLoan` function being blocked at the beginning of an auction.

**Proof Of Concept:**

Add the below test to the testsuite:

```solidity

function testCantBuyLoanAtStartOfTheAuction() public{
        test_borrow();

        vm.startPrank(lender2);
        Pool memory p1 = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId2 = lender.setPool(p1);

        vm.stopPrank();
        

        vm.startPrank(lender1);

        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        
        lender.startAuction(loanIds);
        vm.stopPrank();

        vm.warp(block.timestamp+1);

        vm.startPrank(borrower);
        lender.buyLoan(0, poolId2);
        vm.stopPrank();
    }
```
The above test will fail giving out the error `[FAIL. Reason: RateTooHigh()]`. 

**Recommended Mitigations:**

Instead of doing this calculations just check if the `pools[poolId].interestRate > MAX_INTEREST_RATE`.


