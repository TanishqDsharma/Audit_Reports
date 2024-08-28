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
