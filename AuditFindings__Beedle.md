# High

### [H-1] The `borrow` and `Refinance` function can be frontrun by the lender

**Description:** Whenever a user tries to borrow/refinance a loan, a Lender can frontrun the user and update the `auctionLength` or `interestRate` in his favour and the user might and up getting loan on unfavorable conditions such as very less `auctionLength` or very high `interestRate`.

The Lender is able to take the advantage of `auctionLength` because the pool's `auctionLength` is assigned to the loan without the borrower having the possibility to define a minimum value

**Proof Of Concept:**

1. If the the user/borrower calls `borrow` or `refinance` functions, the pool lender can frontrun and change the auctionLength to an unfavourable such as lender can change its value to 1 by using the `setPool` function. Since, the length of the auction is very less it will end in next block giving an oppurtunity to lender or literally anyone to seize the collateral.

2. Malicious lenders could also exploit this vulnerability to trick borrowers into accepting loans with artificially low interest rates. However, after the refinance transaction is sniffed on-chain, the loan's interest rate can be manipulated to a very high value of MAX\_INTEREST\_RATE (1000%), causing financial losses if the borrower is unaware of the situation and responds too late.

**Impact:** Users might accept the loan that are having unfavourable conditiond for them such as very less `auctionLength` and very high interest rate.

**Recommended Mitigations:**
* Define minimum `auctionLength` so that the lenders are not able to close the loan by updating the `auctionLength` by frontrunning th borrower.
* The `refinance` data should include an expected `auctionLength` and `interestRate` property to guard against front-running attacks. This would enable borrowers to verify the interest rate before finalizing the loan refinancing, helping them avoid accepting loans with unfavorable terms.

 
### [H-2] Weth Rewards are stuck in the `Staking` contract

**Description:** The issue arises when the WETH rewards are sent to the staking contract and noone has staked the tokens yet (i.e when tokenSupply==0), In such a scenario, the index that distributes WETH rewards to stakers is not updated, causing the newly added rewards to be stuck in the contract.

**Impact:** If there are not stakers and rewards are being sent to the contract these rewards are forever lost as these will not be tracked by the index.

**Proof Of Concept:**
* The total supply can be zero when the contract is recently deployed and no stakers have deposited any tokens after the deployment or can be zero if all the stakers have withdrawn there staking

* If the `totalSupply` the reward index will not be updated due to the check at line 63. If the index is not updated the rewards sent will not be recorded. Although the index will not be updated but the balance will be updated due the this line `balance = WETH.balanceOf(address(this));` in `claim` function. amounts. 

```solidity
    ....
        if (totalSupply > 0) {
    ....
```
* Fees contract will also send new rewards to the staking contract time to time but those rewards will also not be tracked as index will not update. 

* Example:
  * Initial State: The contract has 50 WETH and no stakers (totalSupply == 0). The balance is 50 WETH, and the index reflects past rewards based on this amount.
  * New Rewards Sent: The Fees contract sends 100 WETH to the contract. The WETH balance is now 150 WETH. The contract updates balance = 150 but does not update the index because totalSupply == 0.
  * A staker deposits tokens, making totalSupply > 0. They call update(), but because the contract has already updated balance = 150, the condition if (_balance > balance) (i.e., if (150 > 150)) fails. The contract does not detect the newly deposited 150 WETH as rewards and thus does not distribute any of those rewards to the staker.


**Recommended Mitigations:** 

* Consider subtracting the claimed rewards amount from `balance` in line 57 instead of storing the current WETH balance.
* Additionally, consider reverting if no rewards are claimable, i.e., `claimable[msg.sender] == 0` by the caller (`msg.sender`) in the `claim` function.

### [H-3] Hardcoded Router Address may cause Token LockUp in Non-Standard Networks

**Description:**  In `Fees` contract the `swapRouter` address is hardcoded. The router address is set to "0xE592427A0AEce92De3Edee1F18E0157C05861564" and refers to a specific instance of ISwapRouter. This can cause issues when deployed on networks where this address does not correspond to the appropriate Uniswap router. In such cases, tokens may become locked in the protocol indefinitely, preventing withdrawals and potentially leading to financial losses.

```solidity
  ISwapRouter public constant swapRouter =
        ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
```

**Impact:** The hardcoded address can lead to many issues when contract deployed on other networks. Tokens sent to the contract for swapping purposes may not be routed correctly, potentially resulting in funds being locked in the protocol forever.

**Recommended Mitigations:** Pass the address of the SwapRouter in the constructor and/or create functionality for changing it, accessible only by the governance.

### [H-4] Tokens with less decimals leads to losses 

**Description:** The protocol assumes that user will provide tokens that have 18 decimals as collateral and doesn't take into consideration the tokens which have less than 18 decimals. If the collateral is having more than 18 decimals then we are able to borrow the loan with very less collateral and if the collateral is having less than 18 decimals then we need to pay very high collateral to get the loan.

**Impact:**  Users wont be able to borrow loan if they are using collateral that having less decimals as the collateral amount needed would be very high to take the loan and if the users providing collateral with greater than 18 decimals than the users can take the loan without providing much collateral.

**Proof Of concept:**
Alice has a pool which lends DAI for USDC and max LTV is 75%, which means that if Bob wants to borrow 150 DAI, he should collateralize at least 200 USDC. But here is would be the result if we follow the current logic to calculate the LTV:

```solidity
uint256 loanRatio = (debt * 10 ** 18) / collateral;
```

Now, the above calculation will produce the below result:

```solidity
(150 * 10e18)/ 200 * 10e6;
```

We get `750 * 10^9` instead of the expected 750 LTV. This means that if Bob wants to borrow $150 of DAI, he should provide at least around $2,000,000 USDC.


**Recommended Mitigations:**

1. Make sure to scale debt and collateral based on the token decimals in order to calculate properly the `loanRatio`

### [H-6] `Fees::sellProfits` function lacks slippage protection

**Description:** This function can be easily exploited by the MEV as it fails to enforce the minimum token output.  Instead of ensuring that the user receives a minimum amount of tokens from the trade, the function can allow the user to receive zero tokens.

```solidity
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
```

The variable `amountOutMinimum` is having a harcoded value of zero which means that it allows for significant slippage in the token's value before executing the transaction. 

**Impact:** The transactions can be manipulated to get more tokens out of the user.

**Recommended Mitigations:** I recommend adding the `onlyOwner` modifier and setting the `amountOutMinimum` parameter to protect price slippage from MEV bots. If necessary, specify the `sqrtPriceLimitX96` parameter to set a stop price.

### [H-7] Fee on transfer tokens will cause users to lose funds

**Description:** Some ERC20 requires fees whenever `transfer` and `transferFrom` is called. If a contract does not allow for amounts to change after transfers, subsequent transfer operations based on the original amount will revert() due to the contract having an insufficient balance. However, a malicious user could also take advantage of this to steal funds from the pool.

**Impact:** If `Fee on transfer` tokens are not accounted by the protocol correctly it can lead to calculation mistakes and can make the protocol function in an unintended way

**Proof Of Concept:**

1. UserA sends 100 of beedleloan token to the contract when calling `addToPool()`.
2. beedleloan token contract takes 10% of the tokens (10 FEE).
3. 90 beedleloan tokens actually get deposit in contract.
4. `_updatePoolBalance(poolId, pools[poolId].poolBalance + amount);` will equal 100. We can see its taking the amount that we sent into account instead of taking the amount that is left after subtracting the fee.
5. Attacker then sends 100 beedleloan tokens to the contract
6. The contract now has 180 beedleloan tokens but each user has an accounting of 100 beedleloan.
7. The attacker then tries to redeem his collateral for the full amount 100 beedleloan tokens.
8. The contract will transfer 100 beedleloan tokens to attacker taking 10 of UserA's tokens with him.
9. Attacker can then deposit back into the pool and repeat this until he drains all of UserA's funds.
10. When UserA attempts to withdraw the transaction will revert due to insufficient funds.


**Recommended Mitigations:**
Consider protocol whitelist the token address or use balance before and after check to make sure the recipient receive the accurate amount of token when token transfer is performed.

### [H-8] UnsiwapRouter Cannot Swap the tokens without approval

**Description:** Tokens do not get approved to be spent by the Uniswap router, which will always make `sellProfits` revert and lock any tokens sent to this contract in the process.

```solidity
function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```

**Impact:** In `Fees.sol`, `sellProfits` does not approve tokens to be spent by the Uniswap router. This will cause any call to `sellProfits` to always revert upon calling and will results in all tokens sent to the contract to be locked forever.

**Recommended Mitigations:** Approve the tokens so the router can work without any error

```diff
function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
+       IERC20(_profits).approve(address(swapRouter), IERC20(_profits).balanceOf(address(this)))
        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```
### [H-9] `Lender::buyLoan` function can be frontrun by the Pool Lender to drain another user's pool 

**Description:** When a user buys a loan, lender can only pass the `poolId` and no other parameter apart from this. This could lead to problems.

For example:
1. User create a pool with very high interest lets say `maximum interest`.
2. From another wallet, the same user takes a borrow for 1000 USDC and gives 3000 USDT as collateral
3. User A calls `startAuction` for this loan. User B sees the loan on auction and since it has high interest rate and is well-collateralized, decides to buy it. User B calls `buyLoan` and passes to corresponding `loanId`.
4. User sees the pending transaction and front-runs it and changes his `pool.maxLoanRatio` to `type(uint256).max` (basically allowing for loans not sufficiently backed up by collateral)
5. After changing the `maxLoanRatio`, user A calls `refinance` and sets the loan debt to user B's pool balance and changes the collateral to 1 USDT.
6. After doing so, user A changes `maxLoanRatio` back to its original value and once again calls `startAuction` with the same `loanId`.
7. Considering steps 6-8 all executed before user B's `buyLoan`, user B's call now executes and they've bought a borrow for all of their pool balance, backed up by pretty much no collateral.
8. User A successfully executed the attack and has now taken a huge borrow which they have no incentive to repay as there isn't any valuable collateral attached to it.
9. User A can then either withdraw their funds from their pool or repeat the attack.

**Impact:** Pool lender can drain other pools.

**Recommended Mitigations:**

Add the below check:

```solidity            
if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();
```

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

**Description:** Whenever a user tries to take loan, the lender can front-run the user and significantly increase the intrest rate lets say to the `MAX_INTEREST_RATE` and turn the loan into conditions which the user would see as unacceptable. Any user can take a loan by specifying the pool, the borrow amount they want and the collateral they are willing to provide but the user cant specify the interest hes willing to take loan on. So, a lender can monitor the mempool for transactions for taking loan and frontrun those transactions. In the current implementation of the contract, there is no mechanism to transfer funds to another address if the borrower or lender gets blacklisted. This leads to a situation where the funds—either the collateral or loan—get permanently frozen in the contract, as no one can move them from the contract balance.

**Impact:** Users will take laon at much higher interest rates.

**Recommended Mitigations:** Add a param to the Borrow struct which is the max interest rate the user is willing to pay. Upon taking a borrow, make sure this value >= the interest rate of the pool.

### [M-3] Funds will be remain stuck in the contract if Lenders/Borrowers are blacklisted by the ERC20's such as USDC

**Description:**  The vulnerability arises from the inability of a borrower or lender to transfer their withdrawable funds (either the loan or collateral assets) to another address in case they get blacklisted by the respective token contracts (loan or collateral token contracts). If an address gets blacklisted, it is essentially blocked from interacting with the token contract, meaning any attempt to transfer funds to or from that address will fail.

**Impact:** Funds remains stuck in the contract borrower/lender cannot withdraw the funds if they are blacklisted by tokens contracts such as USDC.

**Proof Of Concept:**
* repay(): This function allows the borrower to repay their debt and retrieve the collateral. If the borrower is blacklisted by the collateral asset contract, they cannot receive the collateral, causing it to remain locked in the contract.

* seizeLoan(): In cases of loan default, this function allows the lender to seize the borrower’s collateral. But if the lender is blacklisted by the collateral asset contract, the lender cannot receive the seized collateral, again causing the collateral to be locked in the contract.

* removeFromPool(): This function is supposed to transfer the loan tokens from the contract to the lender. However, if the lender is blacklisted by the loan token contract, the tokens cannot be transferred, and the loan funds remain stuck in the contract.

**Recommended Mitigations:** Add the `recipient` parameter to the functions `repay()`, `seizeLoan()` and `removeFromPool()` functions. This will help to specify whom should receive the fund in case the token contracts  blacklists lender/borrower.

### [M-4] No Proper deadline in `Fees::sellProfits` function

**Description:** There’s a vulnerability in the sellProfits transaction because it uses the current time (block.timestamp) as a deadline. A real deadline would give a limited time window for executing the transaction. For example, you might set the deadline to a few minutes or hours in the future. This ensures the transaction can only occur within that timeframe, protecting it from being executed much later than intended.Since the current code uses `block.timestamp`, it effectively allows the transaction to be executed immediately at any time without offering a meaningful time limit.

```solidity
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
@>                deadline: block.timestamp,// <== By using block.timestamp, you're saying: “The deadline is right now,” which means whenever the transaction is mined, it will pass.
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
```

**Impact:** Without an expiration deadline, a malicious miner/validator can hold a transaction until they favor it or they can make a profit. As a result, the `Fees` contract can lose a lot of funds from slippage.

**Recommended Mitigations:** Use a proper deadline in the future to prevent the transaction from being executed at any manipulated or unfavorable time.

### [M-5] Fixed Fees is used for Swapping Tokens in `Fees::sellProfits` function

**Description:** This issue is that fixed fee is being used in `Fees::sellProfits` function used to swap loan tokens for collateral tokens from liquidations.

```solidity
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
@>                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

```
But not all pools in uniswap have `0.3` fee. 

For example: 
1. fee level of XMON / ETH (0x59b4bb1f5d943cf71a10df63f6b743ee4a4489ee) on Mainnet is 10000 (1%),
2. fee level of WETH / BOB (0x1a54ae9f662b463f8d432482975c17e51518b50d) on Optimism is 500 (0.05%).

**Impact:** 
Using fixed fee level when swap tokens  may lead to some fee tokens being locked in contract.

**Recommended Mitigations:** Instead of using a fixedfee pass fee level to the function.

### [M-6] Pragma Non-Specification can lead to non-functional contracts when deployed to L2's

**Description:** In all the contracts pragma is `^0.8.19` which means it will use the compiler either `0.8.19` or `greater than this version`. By default the compiler will be using the latest version and some L2's wont be supporting this and it will produce broken code for some L2's

**Impact:** Wont be able to compile code on L2's that are not supporting 0.8.19 or higher solidity compiler

**Recommended Mitigations:**

Use earlier version of the compiler. While the project itself may or may not compile with 0.8.20, other projects with which it integrates, or which extend this project may, and those projects will have problems deploying these contracts/libraries.

### [M-7] Frontrunning the `Staking` can get full reward no need to deposit for a particular period of time

**Description:** The `Staking` contract is vulnerable to frontrunning that allows users to unfairly claim rewards by timing their deposits just before reward distribution, without having to stake for a long period. This issue arises because the contract does not account for staking duration when calculating and distributing rewards, leading to an imbalance where short-term stakers can claim the same rewards as long-term stakers.

For example:

1. A user (USERA) can observe that another user (USERB) has been staking for an extended period, accruing rewards.
2. Just before the rewards are distributed, USERA can stake tokens (which could even be for a single block or very few blocks).
3. Because the reward calculation only considers the current stake amount at the distribution time, USERA's newly staked tokens are counted equally to USERB’s long-staked tokens.
As a result, Alice is able to claim the same rewards as Bob, even though Bob staked for a much longer period (e.g., 100 blocks) while Alice staked for a minimal period.


**Impact:** Stakers dont need to stake tokens for longer periods to get rewards they can just stake before the reward distribution and can get the same rewards that any user gets on staking fro longer periods.

**Proof Of Concept:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "solady/src/tokens/ERC20.sol";
import "../src/Staking.sol";

contract SERC20 is ERC20 {
    function name() public pure override returns (string memory) {
        return "Test ERC20";
    }

    function symbol() public pure override returns (string memory) {
        return "TERC20";
    }

    function mint(address _to, uint256 _amount) public {
        _mint(_to, _amount);
    }
}

contract StakingTest is Test {
    SERC20 st;
    SERC20 weth;
    Staking staking;

    function setUp() public {
        st = new SERC20();
        weth = new SERC20();
        staking  = new Staking(address(st), address(weth));
    }
    
    function testDeposit() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");

        deal(address(st), address(alice), 2 ether);
        deal(address(st), address(bob), 2 ether);

        vm.startPrank(bob);
        st.approve(address(staking), 2 ether);
        staking.deposit(2 ether);
        vm.stopPrank();

        vm.roll(100);

        vm.startPrank(alice);
        st.approve(address(staking), 2 ether);
        staking.deposit(2 ether);
        vm.stopPrank();

        deal(address(weth), address(staking), weth.balanceOf(address(staking)) + 1 ether);

        vm.startPrank(alice);
        staking.claim();
        vm.stopPrank();
        vm.startPrank(bob);
        staking.claim();
        vm.stopPrank();
        assertEq(weth.balanceOf(alice), weth.balanceOf(bob));
    }
}
```

**Recommended Mitigations:** Implement a mechanism where rewards are distributed based on both the amount staked and the duration of staking.

### [M-8] Transactions for some ERC20s will revert if value transferred is zero

**Description:** Some tokens such as LEND will revert if the value transferred is zero. So, in `repay` function `_calculateInterest` functions returns lenderInterest and protocolInterest. Later `protocolInterest` is used transfer the protocol fee to the fee receiver but the the transaction will revert if the `protocolInterest` is zero.

```solidity
 // transfer the protocol fee to the fee receiver
            IERC20(loan.loanToken).transferFrom(
                msg.sender,
                feeReceiver,
                protocolInterest
            );
```

**Impact:** If the value of fees that is being transferred is zero then transaction will revert as the transfer value will be zero and some tokens like LEND does not support this.

**Recommended Mitigations:** Upon sending fees, make sure it is a non-zero value.

### [M-9] If a token has low decimals then rounding errors will generate 

**Description:** While taking the loan if the user passes a token as collateral which has less decimals then the loan can be borrowed without paying any interest

```solidity
    function _calculateInterest(
        Loan memory l
    ) internal view returns (uint256 interest, uint256 fees) {
        uint256 timeElapsed = block.timestamp - l.startTimestamp;
@>      interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;
@>      fees = (lenderFee * interest) / 10000;
        interest -= fees;
    }
```

**Impact:** Borrowers will not be paying any interest or interest will be very negligible which not expected by the protocol

**Proof Of Concept:**

For example: Lets take GUSD which has 2 decimals
* `l.debt`: 100 GUSD
* `l.interestRate`: 100 (1%) per year
* `timeElapsed`: 86400 seconds (1 day)
* `lenderFee`: 100 (1%)

```solidity
interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days
         = (100 * (100 * 1e2) * 86400) / 10000 / 31536000
         = 8640 / 31536
         = 0 (rounding down; since Solidity has no fixed-point numbers)

fees = (lenderFee * interest) / 10000
     = (100 * 0) / 10000
     = 0
```

**Recommended Mitigations:**
Check the tokens that users are passing as collateral have enough decimals to avoid rounding errors. ADD `MIN_LOAN_TOKEN_DECIMALS` constant with an appropriate value.

### [M-10] Malicious User FrontRuns the User by creatingPool for WETH-Profits token pair with highly manipulated price

**Description:**  The issue arises from the way `sellProfits` function interacts with uniswapV3 pools to swap tokens.The function attempts to swap a specifies _profits token into WETH using `uniswapV3::exactInputSingle` function which requires an existing liquidity pool between the two tokens. If no such pool exists, the transaction will revert. 

However, this behaviour opens the door for a frontrunning attack where a malicious user can create a pool with manipulated liquidity ratios, enabling them to steal funds through unfavorable swaps.

**Impact** When swapping the _profits token for WETH most of the tokens will be lost due to highly manipulated prices.

**Proof Of Concept:**

This is a theoretical POC:

1. A liquidation event occurs, and the contract intends to swap _profits tokens for WETH.
2. The bad actor monitors pending transactions and sees the `sellProfits` function call about to execute.
3. Before the function executes, the bad actor frontruns the transaction by creating a Uniswap V3 pool with a highly manipulated price for the _profits token and WETH.
Step
4. The `sellProfits` function then proceeds to execute and swaps _profits for WETH, but due to the skewed liquidity ratio, the contract receives an unfavorable amount of WETH.

**Recommended Mitigations:** Consider using 1swapExactInputMultihop1 to first swap the less liquid token (such as the _profits token) into a more liquid intermediary token like USDC, and then swap USDC into WETH. This approach is preferable for the contract, as it handles a variety of tokens with different levels of liquidity. By routing the swap through a highly liquid token like USDC, you reduce the risk of price manipulation or unfavorable exchange rates that may occur with direct swaps involving less popular tokens. This ensures a more stable and reliable swap outcome.

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


