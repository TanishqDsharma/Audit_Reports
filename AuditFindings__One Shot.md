# High

### [H-1] 
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

### [H-2]  Weak Randomness in `RapBattle::Battle` function
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

### [H-3]
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**


# Medium

### [M-1]
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

### [M-2]
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

# Low

### [L-1]  `RapBattle::_battle()` can emit wrong events
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

### [L-2] `battlesWon` property is never updated 

**Description:** `RapBattle` contract allows the rappers to get on stage and battle, and then the winner is selected but the `battlesWon` property is never updated. 

**Impact:** Since, `battlesWon` is never updated and its value always stays `0`, so there is no way to track that how many battle a rapper has won.

**Recommended Mitigation:**
Add logic to update the `battlesWon` property 

### [L-3] `RapBattle::_battle` function doesn't implements the check to select the winner correctly

**Description:** The `RapBattle::_battle` function selects the defender as the winner when random number is equal to defender rapper skill. This should lead to a tie between the defender and challenger but the way this is implemented selects the defender as the winner.

So, if the random number is less than or equal to defenders skill then defender wins, and only if random number is greater than defender rappers skill the challenger wins, which makes it quite unfair since the defender is having two possiblities for a win and challenger is only having one.

```solidity

function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;
    
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

@>        if (random <= defenderRapperSkill) {
            
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }

```

**Impact:** The lack of handling for equal skill could lead to confusion or undesired outcomes for users such as the bearing the reward of both bets even though there was a draw.

**Proof Of Concept:**
When the _battle function is called and the random numenr is equal to defender rapper skill,then the function fails to handle this tie scenario, unfairly awarding the total bet amount to the defender.

**Recommended Mitigation:**

Add the check to handle the situation when random number is equal to defender rapper skill.

### [L-4] Rappers can battle with themselves which results in avoiding a strong Opponent

**Description:** The `RapBattle::goOnStageOrBattle` function does not include checks to prevent rappers from battling themselves, which allows them to bypass facing a valid, potentially stronger opponent.

**Impact:**
An attacker can cause a problem for other users, as he can frontrun them and battle with himself before anyone else could battle the rapper. In this way he can stay the defending champion.

**Proof Of Concept:**
```solidity
function test_RappersCanBattleThemselves() public{
    vm.startPrank(user);
    oneShot.mintRapper();
    oneShot.approve(address(streets), 0);
    streets.stake(0);
    vm.warp(5 days+1);
    streets.unstake(0);
    cred.approve(address(rapBattle), 1);

    oneShot.approve(address(rapBattle), 0);
    rapBattle.goOnStageOrBattle(0,1);
    rapBattle.goOnStageOrBattle(0,1);
    vm.stopPrank(); 
}
```

**Recommended Mitigation:**

You can add the below check to `RapBattle::goOnStageOrBattle` that prevents rappers from battling themselves

```diff
function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
+       require(msg.sender != defender, "RapBattle: Cannot battle against yourself");
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
 ......
}
```

# Informational 

**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**
