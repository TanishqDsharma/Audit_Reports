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

****:

```solidity
 function borrowAllowed(address msgSender, address, uint) public view returns (bool) {
        if(msgSender == tx.origin) return true; //@audit EOA bypass
        return contractAllowlist[msgSender];
    }
```
