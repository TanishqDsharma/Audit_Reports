# High

### [H-1] `preDepositPrice` can be manipulated resulting in stealing of user'd deposit

**Description:** The issue arises when the first user that stakes manipulates the total supply of `safEth Tokens` and by doing so create a rounding error for each subsequent user. In the worst case, an attacker can steal all the funds of the next user.

```solidity

 uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
        _mint(msg.sender, mintAmount);

```
To attack the `SafEth` the attackers first needs to become the first depositor with any amount of staked ETH, then withdraw the most of the stake so there is only 1 wei of SafEth shares left, and then inflate the underlying asset value by manually transferring a large amount of underlying assets to relevant derivative pools instead going through the `SafEth.stake()` function. 


**Impact:**  Loss of User's funds as any staker after the hacker's attack will receive zero shares for their stake. The attacker can redeem all underlying funds at any time by burning the only 1 wei share in the contract.

**Proof Of Code:**

**Recommended mitigations:** To add a mitigation against this attack protocol can implement the logic to mint the dead shares so that the cost to perform this attack increases considerably.

### [H-2] 












