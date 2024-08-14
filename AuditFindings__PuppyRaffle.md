
## [M-#] TITLE Looping through Players Array to check for duplicates in `PuppyRaffle::enterRaffle` is potential DOS(Denial Of Service), incrementing the gas cost for future entrance.

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is , the more checks new player will have to make. This means the gas cost for players who enter right when raffle starts will be drammatically lower than those who enters later.

Every additonal, address in the `players` array, is an additional check the loop will have to make.

This is due to the for loop in the PuppyRaffle::enterRaffle function.

```javascript
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

## I-1: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```

</details>

## I-2: Using an outdated version of Solidity is not recommended

Please user a newer version like `0.8.18`

**Description:**
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.
Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.
