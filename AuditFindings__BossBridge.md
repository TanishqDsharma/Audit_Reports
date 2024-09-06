# High


### [H-1] Users giving approvals of tokens to L1BossBridge may have those assest stolen

**Description:** The `L1BossBridge::depositTokensToL2` function allows anyone to call it with a from address of any account that has approved tokens to the bridge.

As a result any attacker can steal funds of the users that have given token allowance more than zero to `L1BossBridge`. This will move the tokens into the bridge vault, and assign them to the attacker's address in L2 (setting an attacker-controlled address in the l2Recipient parameter).

```solidity
 @> function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
        if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
            revert L1BossBridge__DepositLimitReached();
        }
        token.safeTransferFrom(from, address(vault), amount);

        // Our off-chain service picks up this event and mints the corresponding tokens on L2
        emit Deposit(from, l2Recipient, amount);
    }
```

**Impact:** Attacker can easily steal funds from the users.

**Proof Of Concept:** Add the below test to your test suite:

```solidity

function testAttackerCanStealUserDeposit() public {
        vm.prank(user);
        uint256 amount = 10e18;
        token.approve(address(tokenBridge), amount);

        address maliciousActor = makeAddr("ATTACKER");
        vm.prank(maliciousActor);
        tokenBridge.depositTokensToL2(user, maliciousActor, amount);

        assertEq(token.balanceOf(address(tokenBridge)), 0);
        assertEq(token.balanceOf(address(vault)), amount);
        }
```

**Recommended Mitigation:**

Modify the `L1BossBridge::depositTokensToL2` function so that the caller cannot specify a from address.

```diff

- function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
+ function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }
-   token.transferFrom(from, address(vault), amount);
+   token.transferFrom(msg.sender, address(vault), amount);

    // Our off-chain service picks up this event and mints the corresponding tokens on L2
-   emit Deposit(from, l2Recipient, amount);
+   emit Deposit(msg.sender, l2Recipient, amount);
}

```


### [H-2] `L1BossBridge::depositTokensToL2` can be exploited to a DOS attack by malicious actor. 

**Description:** There is a limit on how much a user can deposit to L2. The problem arises when the function uses the contract balance to validate the `DEPOSIT_LIMIT`. The contract sets a fixed DEPOSIT_LIMIT for the total amount of tokens that can be deposited. The contract does not account for withdrawals in adjusting this limit, potentially leading to a situation where no further deposits can be made once the limit is reached. 

```solidity
function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
 @>       if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
            revert L1BossBridge__DepositLimitReached();
        }
        token.safeTransferFrom(from, address(vault), amount);

        // Our off-chain service picks up this event and mints the corresponding tokens on L2
        emit Deposit(from, l2Recipient, amount);
    }
```

A Scenario like this can happen:
1. Deposit tokens repeatedly until the sum of tokens in the vault reaches DEPOSIT_LIMIT.
2. Attempt another deposit; it will fail due to the limit being reached.
3. Even if tokens are withdrawn, further deposits are not possible as the limit does not adjust.

**Impact:** This issue can lead to permanent halt of the deposit function, making a significant part of contracts functionality unusable.

**Proof Of Concept:**
```solidity

   function testDepositHalted() public{
        vm.startPrank(makeAddr("MaliciousActor"));
        deal(address(token), makeAddr("MaliciousActor"), tokenBridge.DEPOSIT_LIMIT());
        token.approve(address(tokenBridge), tokenBridge.DEPOSIT_LIMIT());
        token.transfer(address(vault), tokenBridge.DEPOSIT_LIMIT());
        vm.stopPrank();
        // Now, user wont be able to deposit
        vm.startPrank(user);
        uint256 amount = 1;
        deal(address(token), user, amount);
        token.approve(address(tokenBridge), amount);
        vm.expectRevert(L1BossBridge.L1BossBridge__DepositLimitReached.selector);
        tokenBridge.depositTokensToL2(user, userInL2, amount);
        vm.stopPrank();
    }
```

**Recommended Mitigation:**
Use a mapping to track the deposit limit of each use instead of using the contract balance

### [H-3] CREATE opcode does not work on zksync era

**Description:** zkSync Era chain has differences in the usage of the create opcode compared to the EVM. According to the protocol documentation is going to be deployed on zksync Era. 
The [zkSync Era docs](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions) explain how it differs from Ethereum. The description of CREATE and CREATE2 (zkSynce Era Docs) states that Create cannot be used for arbitrary code unknown to the compiler.

**Impact:**
No editions can be created in the zkSync Era chain.

**Recommended Mitigation:**
Follow the instructions given in (zksync era docs)[https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions#create-create2] related to CREATE,CREATE2.


To guarantee that create/create2 functions operate correctly, the compiler must be aware of the bytecode of the deployed contract in advance. The compiler interprets the calldata arguments as incomplete input for ContractDeployer, as the remaining part is filled in by the compiler internally. The Yul datasize and dataoffset instructions have been adjusted to return the constant size and bytecode hash rather than the bytecode itself.



### [H-4] Minted Tokens locked forever

**Description:** The `TokenFactory::deployToken` function calls `L1Token` contract for deploying purpose. The problem arises when the `L1Token` mints `INITIAL_SUPPLY` to `msg.sender` which is `TokenFactory` contract in this case. After deployment, there is no way to either transfer out these tokens or mint new ones, as the holder of the tokens, `TokenFactory`, has no functions for this, also not an upgradeable contract, so all token supply is locked forever.


**Impact:**
Using `TokenFactory` for deploying token contracts will result in unusable tokens as those tokens will stay locked forever.

**Recommended Mitigations:**

Modify the below code by adding a receiver address so that it have the intial minted tokens.

```diff
contract L1Token is ERC20 {
    uint256 private constant INITIAL_SUPPLY = 1_000_000;

-    constructor() ERC20("BossBridgeToken", "BBT") {
+    constructor(address receiver) ERC20("BossBridgeToken", "BBT") {
-         _mint(msg.sender, INITIAL_SUPPLY * 10 ** decimals());
+         _mint(receiver, INITIAL_SUPPLY * 10 ** decimals());
    }
}
```
