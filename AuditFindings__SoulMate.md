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

