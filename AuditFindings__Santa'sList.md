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

**Recommended Mitigation: **

