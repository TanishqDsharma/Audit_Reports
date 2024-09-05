# High

### [H-1] `RapBattle::_battle()` 
**Description:**
**Impact:**
**Proof Of Concept:**
**Recommended Mitigation:**

### [H-2] `RapBattle::goOnStageOrBattle()` doesn't checks for ownership of the Rapper NFT for the challenger which results in allowing the challengers to go on stage with othe Rappers NFT

**Description:** `RapBattle::goOnStageOrBattle()` allows users to put their rappers on stage and compete. For the defender, the Rapper NFT is transferred to `RapBattle` contract thus assuring that the caller i.e., `msg.sender` is the owner of that NFT, but for the challenger neither Rapper NFT is transferred nor ownership info is, allowing anyone to go on stage with anyone's rapper NFT. This results in if the challenger wons the battle using anothers Rapper NFT then we will also get the bet amount.

```solidity
 function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```


* Along, with that one can even pass tokenId which doesn't even exist and the rapper stats returned will be:
  as the value for struct returned will be:

```solidity
weakKnees: false
heavyArms: false
spaghettiSweater: false
calmAndReady: false
battlesWon: 0
```
As a result of which the rapper skill points returned by `RapBattle::getRapperSkill` will be 65 (even though a beginner rapper has 50 score but non existent rapper has 65), thus giving people unfair advantage for rap battle as it is used directly inside `RapBattle::_battle` function.

**Impact:**
This exploit allows users to battle with tokens they do not own, it even allows users to battle without having minted a token at all.

**Proof Of Concept:**

```solidity
function testBattleMyself() public mintRapper {
    vm.startPrank(user);
    oneShot.approve(address(streets), 0);
    streets.stake(0);
    vm.warp(1 days + 1);
    streets.unstake(0);
    cred.approve(address(rapBattle), 1);
    oneShot.approve(address(rapBattle), 0);
    rapBattle.goOnStageOrBattle(0, 0);
    vm.stopPrank();

       
    vm.startPrank(makeAddr("2PAC"));
    oneShot.mintRapper();
    vm.stopPrank();

    vm.startPrank(challenger);
    // Challenger is using NFT of user 2PAC to get on Stage
    rapBattle.goOnStageOrBattle(1, 0);
    vm.stopPrank();
}
```

**Recommended Mitigation:**

Add ownership check before allowing challenger to battle with rapper NFT token id.

```diff
 function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {

            // credToken.transferFrom(msg.sender, address(this), _credBet);
+           if (msg.sender != oneShotNft.ownerOf(_tokenId)) {
+               revert RapBattle__NotAuthorized();
+           }
            _battle(_tokenId, _credBet);;
        }
    }
```

### [H-3]  Weak Randomness in `RapBattle::_battle()` function that can allow attackers to predict the random number to influence or predict the winner

**Description:**
Hashing `block.timestamp`, `block.prevrandao` and `msg.sender`,  together created a predictable final number. A predictable number is not a good random number. Malicious users can manipulate these values or known these ahead of time to choose the winner themselves.

```solidity
uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;
```

**Impact:** An attacker can inflence the winner of the rap battle.

**Proof Of Concept:**

There are a few attack vectors here. 

1. Validators can know ahead of time the `block.timestamp` and `block.prevrandao` and use that knowledge to predict when / how to predict the random number to influence  the winner. See the [solidity blog on prevrando](https://soliditydeveloper.com/prevrandao) here.
2. Users can mine/manipulate the `msg.sender` value to result in their address being used to generate the winner.

Using on-chain values as a randomness seed is a [well-known attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.


**Recommended Mitigation:**
Consider using an oracle for your randomness like [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction).

# Medium

### [M-1] `OneShot::mintRapper()` is vulnerable to re-entrancy resulting in user minting NFT with high Rappers skills instead of starting from the default skills

**Description:** The `OneShot::mintRapper()` function presents a reentrancy vulnerability, `safeMint()` is executed prior to updating the `RapperStats`, violating the CEI.

```solidity
function mintRapper() public {
        uint256 tokenId = _nextTokenId++;
@>        _safeMint(msg.sender, tokenId);

        // Initialize metadata for the minted token
@>        rapperStats[tokenId] = RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});
    }
```

This flaw allows attackers to exploit the system by creating a contract specifically for minting new rappers. So, attackers can take advantage to mint high skill rapper NFTs.

**Impact:** Users can mint rapper NFTs that are having higher skills than default.

**Proof Of Concept:**

Attacker Contract: Add this to your test suite

```solidity
 function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        console.log("Current token Id is:",oneShot.getNextTokenId());
        uint256 challengerRapperSkill = rapBattle.getRapperSkill(1);
        // This is supposed to be 50, but it is 65 now
        require(challengerRapperSkill == 65, "PoC failed");
        console.log("challengerRapperSkill: ", challengerRapperSkill);        
        rapBattle.goOnStageOrBattle(1, rapBattle.defenderBet());
        return 0x150b7a02;
    }
```

Also add the below test to your test suite: 

```solidity
function testPoCReentrancys() public {
        // user staking for 4 days
        vm.startPrank(user);
        oneShot.mintRapper(); // _tokenId == 0
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.stopPrank();

        vm.warp(1 days + 1);
        vm.startPrank(user);
        streets.unstake(0);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(0, 1);
        vm.stopPrank();

        // challenger battle and steal token
        vm.startPrank(challenger);
        AttackContract attackContract = new AttackContract(address(rapBattle), address(oneShot));
        attackContract.pwn();
        vm.stopPrank();
    }
```
**Recommended Mitigation:** Follow the check, effects and interactions patterns

```diff
function mintRapper() public {
        uint256 tokenId = _nextTokenId++;
-        _safeMint(msg.sender, tokenId);

        // Initialize metadata for the minted token
        rapperStats[tokenId] =
            RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});
+       _safeMint(msg.sender, tokenId);

    }
```

### [M-2] `Streets::unstake()` mints incorrect amount of tokens for users as reward resulting in users reciving very negligble amount of tokens as Rewards.

**Description:** In `Streets::unstake()` , the conditionals statements mints only 1 unit of the token instead of one full token. The `CRED` token is an ERC20 contract with 18 decimals, therefore `1` CRED token is considered equivalent to 10^18 as additional 18 zeroes are used for representing the floating values. 

The protocol mentions to mint `1` CRED token per day staked for a maximum of 4 days, therefore the equivalent amount of CRED token to mint by considering the decimals in solidity will be 10^18 but it only mints `0.000000000000000001`. Therefore, users get very negligible amount of `CRED` token.

```solidity
  
if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
@>            credContract.mint(msg.sender, 1); 
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
@>              credContract.mint(msg.sender, 1); 
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
@>              credContract.mint(msg.sender, 1); 
        } 
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true; 
@>              credContract.mint(msg.sender, 1); 
        }

```

**Impact:** Users are reciveing very negligible amount of token.
 
**Recommended Mitigation:**

Tokens should be minted by taking the 18 decimals into consideration then only it will be equivalent to `1` CRED token.

```diff
    if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
```

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
