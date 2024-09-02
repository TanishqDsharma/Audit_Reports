# High

### [H-1] Incorrect Calcluation of timestamp in `Staking::lastClaim` mapping. The`Staking::claimRewards` calculates `lastClaim` of a user based on their soulmate nft creation timestamp instead of staking timestamp leading to incorrect staking rewards payout.

**Description:** The `staking::claimRewards` function rewards the users with token who deposit into the contract for a specific period of time. The token is rewarded on the below logic (number_of_weeks*number_of_tokens__deposited), so if you deposited 1 token for 2 weeks you will get 14 tokens as a reward. However, the contract does not follows the intended behaviour in some cases and the issue lies int `lastClaim` variable.

```solidity
       if (lastClaim[msg.sender] == 0) {
  @>          lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
                soulmateId );
```
The function first checks if `lastClaim[msg.sender]` is equal to zero. If the user is claiming the reward first time then it will call ` soulmateContract::idToCreationTimestamp` function and set `lastClaim[msg.sender]` to the timestamp their soulmate NFT was created. For a user who has no soulmate, this will set `lastClaim[msg.sender]` to the time the first soulmate NFT was created. Finally, For a user who is still awaiting a soulmate, it will set `lastClaim[msg.sender]` to 0 as the timestamp of the next soulmate NFT ID doesn't exist yet.

**Impact:** 
* A soulmate is rewarded tokens based on when their soulmate NFT was created regardless of the time they deposited.
* A user who has never minted a soulmate will be rewarded tokens based on when the first soulmate NFT was created regardless of the time they deposited.
* A user awaiting a soulmate but has deposited tokens can be rewarded tokens based on current `block.timestamp` divided by 604800 (`1 weeks`) regardless of the time they deposited.


**Proof Of Concept:**

```solidity

function test_IncorrectRewardCalculation()public{

        address newstaker = makeAddr("STAKER");
        vm.prank(soulmate1);
        soulmateContract.mintSoulmateToken();
        vm.prank(soulmate2);
        soulmateContract.mintSoulmateToken();

        vm.prank(soulmate1);
        vm.warp(2 weeks);

        airdropContract.claim();
        

        vm.prank(soulmate1);
        loveToken.transfer(newstaker, 2);
        vm.startPrank(newstaker);
        soulmateContract.mintSoulmateToken();
        loveToken.approve(address(stakingContract), 2);
        stakingContract.deposit(2);
        vm.warp(block.timestamp+1 weeks+1); // Total approx 3 weeks and some seconds
        stakingContract.claimRewards(); // User whos awaiting a soulmate initiated the claimRewards() function
        vm.stopPrank();
        assert(loveToken.balanceOf(newstaker)==6);
        }
```

**Recommended Mitigation:**
Modify the `claimRewards` function to use the deposit timestamp as the starting point for calculating rewards. Implement a mechanism to track the first deposit timestamp for each user and ensure that the lastClaim mapping is updated accordingly. Consider adding checks to prevent users from claiming rewards before making a deposit.

### [H-2] Lack of validations allows users without a Soulmate NFT to claim lovetokens and also divorced users can claim tokens 

**Decription:** According to the documentation only users having soulmate NFT can claim the loveTokens but since it lacks validations anyone can claim loveTokens without having the NFT. Additionally, `Soulmate::isDivorced` function, where the airdrop contract address is used
instead of the user's address. A divorced user can still claim a Soulbound NFT.

First, the `Airdrop::isDivorced` check send a contract address instead of users address to the function and it will always returns false and the check will fail

```solidity
 if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

```

```solidity
function isDivorced() public view returns (bool) {
        return divorced[msg.sender];
    }
```

**Impact:**
**Proof Of Concept:**
**Recommended Mitigation**

### [H-3]

**Decription:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation**

### [H-4]
**Decription:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation**


# Medium

### [M-1] Users Can Couple with themselves

**Description:** Instead of finding an address thats different from user so that he/she can couple with other address, the user can couple with themselves and claim the tokens.

**Impact:**
The documentation of the protocol states that there should be a couple to claim the `loveTokens`, but since the user can couple with himself and claim the `loveTokens` this defeats the whole logic of the protocol.

**Proof Of Concept:**

Add the below test to your testsuite:

```solidity
 function test_UserCanCoupleThemselvesAndClaim() public{
        console.log("Soulmate1 is having address:",soulmate1);
        vm.startPrank(soulmate1);
        soulmateContract.mintSoulmateToken();
        soulmateContract.mintSoulmateToken();
        vm.stopPrank();
        console.log("Soulmate of Soulmate1 is:",soulmateContract.soulmateOf(soulmate1));
        vm.warp(2); 

        vm.prank(soulmate1);
        airdropContract.claim();
        console.log("Balance of LoveToken is:",loveToken.balanceOf(soulmate1));}
```

**Recommended Mitigation:**

Add the below line to conditional statement in ``Soulmate::mintSoulmateToken`` line 75:

```diff
else if (soulmate2 == address(0)) {
+          require(idToOwners[nextID][0] != msg.sender, "You cant couple with yourself!");	            
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;
```

### [M-2] `Soulmate::writeMessageInSharedSpace` can be used by anyone when NFT ID is `0`

**Description:** Since, `Soulmate::ownerToId` returns the soulmate NFT id that is associated with the soulmate participating in the contract but it return 0 as id if the user has not participated in the protocol. We know that  `Soulmate::nextID` starts from `0`, making the NFTId `0` and is assigned to soulmates, but `Soulmate::ownerToId` returning 0 for non-soulmates user lead to potential loss of soulmates whose id is actually 0 as well as allows the non-soulmates users to enjoy the same benefits (of Airdrop & Staking), soulmates with id = 0 are enjoying. This also maks the `Soulmate::writeMessageInSharedSpace` vulnerable to allow anybody to write a message to `soulmates` who own `nextID` `0`,


**Impact:** Non-Soulmate users can claim tokens from airdrop and stake tokens.

**Proof Of Concept:**

Add The below Test in  `Airdrop.t.sol`

```solidity
 function test_AnyoneCanClaimIfNFTId0() public {
        _mintOneTokenForBothSoulmates();
        
        address hater = makeAddr("HATER");
        console.log("Before the claim hater is having love tokens =",loveToken.balanceOf(hater));

        vm.prank(hater);
        vm.warp(10 weeks);
        airdropContract.claim();

        assert(loveToken.balanceOf(hater)>0);
        console.log("After the claim hater is having love tokens = ",loveToken.balanceOf(hater));
    }
```

Add The below Test in  `Soulmate.t.sol`

```solidity
    function test_AnyoneCanWriteMessageIfNFTID0() public {
            _mintOneTokenForBothSoulmates();
            address hater = makeAddr("HATER");
            vm.prank(hater);
            soulmateContract.writeMessageInSharedSpace("I dont Love U anymore .......");
             vm.prank(soulmate2);
            string memory message = soulmateContract.readMessageInSharedSpace();
            console2.log(message);}
```

**Recommended Mitigation:** Start the nextId from 1 instead of 0

# Low

### [L-1] `Soulmate::SoulmateAreReunited` event is emitting out incorrect values:

**Description:** The event `SoulmateAreReunited` is emitted with the parameter soulmate2, which is explicitly checked to be address(0) in the eles if conditional statement, causing soulmate2 to always be zero when triggering `emit SoulmateAreReunited(soulmate1, soulmate2, nextID)` .

```solidity
else if (soulmate2 == address(0)) {
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;
            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);
            _mint(msg.sender, nextID++);
```

**Impact:**
This emit being emitted with incorrect parameters can cause misinformation

**Proof Of Concept**:

Execute the below test and see the events emitted:

```solidity

    function test_MintNewToken() public {
        uint tokenIdMinted = 0;

        vm.prank(soulmate1);
        
        soulmateContract.mintSoulmateToken();

        assertTrue(soulmateContract.totalSupply() == 0);

        vm.prank(soulmate2);
       
        soulmateContract.mintSoulmateToken();

        assertTrue(soulmateContract.totalSupply() == 1);
        assertTrue(soulmateContract.soulmateOf(soulmate1) == soulmate2);
        assertTrue(soulmateContract.soulmateOf(soulmate2) == soulmate1);
        assertTrue(soulmateContract.ownerToId(soulmate1) == tokenIdMinted);
        assertTrue(soulmateContract.ownerToId(soulmate2) == tokenIdMinted);

    }
```

**Ouput Generated:**

```solidity
 [213016] SoulmateTest::test_MintNewToken()
    ├─ [0] VM::prank(soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f])
    │   └─ ← [Return] 
    ├─ [33015] Soulmate::mintSoulmateToken()
    │   ├─ emit SoulmateIsWaiting(soulmate: soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f])
    │   └─ ← [Return] 0
    ├─ [393] Soulmate::totalSupply() [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertTrue(true) [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::prank(soulmate2: [0xe93A5E9F20AF38E00a08b9109D20dEc1b965E891])
    │   └─ ← [Return] 
    ├─ [157343] Soulmate::mintSoulmateToken()
@>  │   ├─ emit SoulmateAreReunited(soulmate1: soulmate1: [0x65629adcc2F9C857Aeb285100Cc00Fb41E78DC2f], soulmate2: 0x0000000000000000000000000000000000000000, tokenId: 0)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: soulmate2: [0xe93A5E9F20AF38E00a08b9109D20dEc1b965E891], id: 0)
    │   └─ ← [Return] 0
```

**Recommended Mitigation:** Instead of emitting soulmate2 emit the msg.sender

```solidity
    emit SoulmateAreReunited(soulmate1, msg.sender, nextID);
```

### [L-2] The `Soulmate::totalSouls` functions inaccurately calculates the total paired souls

**Description:** he Soulmate::totalSouls function in your contract is supposed to count how many soul pairs (two people matched together) exist. However, the way it's currently calculating this number is incorrect. It assumes that every time a new soul is created, it automatically gets paired successfully with another soul. This assumption leads to scenario's such as:

1. Divorced Users: Users that got a divorce the function is still considering them as a pair
2. Self Paired Users: Users that formed a pair by themselves also considere a valid pair by this function
3. Unpaired Souls: Some users might start the process to find a soulmate but haven't been matched yet. These users shouldn't be counted as part of a pair, but the current function mistakenly includes them.

```solidity
function totalSouls() external view returns (uint256) {
        return nextID * 2;
    }
```

**Impact:**
The inaccurate calculation of total pairs in the impacts the metrics of the protocol. It could mislead users about the platform's activity level and the actual number of successful pairings, potentially affecting user trust and engagement.

**Proof Of Concept:**

```solidity
function testTotalSouls() public{
        vm.prank(makeAddr("IonlyPairwithmyself"));
        soulmateContract.mintSoulmateToken();
        vm.prank(makeAddr("IonlyPairwithmyself"));
        soulmateContract.mintSoulmateToken();
        
        //Fails to check for self paired soulmates
        assert(soulmateContract.totalSouls()==2);

        vm.prank(soulmate1);
        soulmateContract.mintSoulmateToken();
        vm.prank(soulmate2);
        soulmateContract.mintSoulmateToken();
        vm.prank(makeAddr("lover1"));
        soulmateContract.mintSoulmateToken();
        vm.prank(makeAddr("lover2"));
        soulmateContract.mintSoulmateToken();

        assert(soulmateContract.totalSouls()==6);

        //Lets a get a divorce 

        vm.prank(makeAddr("lover2"));
        soulmateContract.getDivorced();
        //After getting the divorce now there should only be one soulmate pair but the function doesn't takes 
        //that into account and still shows 2 pairs
        assert(soulmateContract.totalSouls()==4);
        //Fails to update pairs after getting the divorce
}
```

**Recommended Mitigations:**
1. Prevent Self-Pairing: Implement checks within the minting function to prevent users from being paired with themselves
2. Consider maintaining a separate counter for active couples.


### [L-3] Incorrect Data Emitted from events in `LoveToken`

**Description:**

`LoveToken::AirdropInitialized` and `LoveToken::StakingInitialized` events both emits incorrect data. The event data indicates that the parameters passed should be `airdropContract` and `stakingContract`, respectively. Still, the function emits both events with the parameter `managerContract`, resulting in inaccurate event logs.

```solidity
 function initVault(address managerContract) public {
        if (msg.sender == airdropVault) {
            _mint(airdropVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
@>            emit AirdropInitialized(managerContract);
        } else if (msg.sender == stakingVault) {
            _mint(stakingVault, 500_000_000 ether);
            approve(managerContract, 500_000_000 ether);
@>            emit StakingInitialized(managerContract);
        } else revert LoveToken__Unauthorized();
    }
```

**Impact:**
The incorrect event data poses a challenge for developers and external systems relying on event logs, as the information provided does not accurately reflect the initialized contracts. This discrepancy may result in confusion and hinder proper tracking of contract initialization events.

**Recommended Mitigations:**
Update the `initVault` function to emit both the `AirdropInitialized` and `StakingInitialized` events with the correct parameters, reflecting the addresses of the corresponding contracts (`airdropContract` and `stakingContract`).

### [L-4] `_mint` function is being used for minting NFT instead of `_safeMint`

**Description:** It's recommended to use `_safeMint` instead of `_mint` function when minting an NFT. If a soulmate is a smart contract address that doesn't support ERC721 standard, the NFT can be frozen in the contract.

**Impact:**

The Soulmate NFT can be frozen in the receiver contract.

**Recommended Mitigation:**

Use `_safeMint` instead of `_mint`, to check if receiver address supports ERC721's `onERC721Received` implementation.

