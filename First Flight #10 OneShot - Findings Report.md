# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Weak Randomess in `RapBattle::_battle` Allows Users to Influence / Predict Battle Winners](#H-01)
    - ### [H-02. `CRED` Whales Can Submit Excessively Large Bets in `RapBattle:goOnStageOrBattle`, Pricing Others Out, Resulting in DoS](#H-02)
    - ### [H-03. Missing `CRED` Balance Check in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Withouth `Credbility` to Cover the Bet](#H-03)
    - ### [H-04. Missing Challenger Input Validation in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Without an NFT and With Zero `CRED` Balance](#H-04)
    - ### [H-05. Missing NFT Ownership Check in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Without an NFT](#H-05)
- ## Medium Risk Findings
    - ### [M-01. `ICredToken::approve`, `ICredToken::transfer` And `ICredToken::transferFrom` Have Incorrect ERC20 Function Interfaces](#M-01)
- ## Low Risk Findings
    - ### [L-01. `battlesWon` statistics in the `OneShot::RapperStats` Struct is Not Updated After Winning a Battle](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 1
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Weak Randomess in `RapBattle::_battle` Allows Users to Influence / Predict Battle Winners            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L63

## Vulnerability Details
`RapBattle::_battle` hashes `msg.sender`, `block.timestamp`, and `block.prevrandao` to create a supposedly random number that is eventually used to determine the winner of a rap battle. However, hashing these values does not create a truly random number. Malicious users can manipulate these values or know them ahead of time to choose the winner of the battle. 

- Validators can know ahead of time the `block.timestamp`.
- `prevrandao` [suffers from biasibility](https://soliditydeveloper.com/prevrandao), miners can know its value ahead of time 
- User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner.

Blockchains are deterministic systems by nature, designed to achieve consensus across all nodes. Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.


## Impact
Users can influence / predict the winner of battles.

## Proof of Code

Insert the following piece of code to `OneShotTest.t.sol`:

```javascript
    function test_weakRandomness() public {
        address winner;
        address previousWinner;
        address defender = makeAddr("defender");

        vm.prank(defender);
        oneShot.mintRapper();
        assertEq(oneShot.ownerOf(0), defender);

        vm.prank(challenger);
        oneShot.mintRapper();
        assertEq(oneShot.ownerOf(1), challenger);

        // cheking that the rapper skills are the same, ensuring a theoretical chance of 50% for each party to win a battle
        assertEq(rapBattle.getRapperSkill(0), rapBattle.getRapperSkill(1));

        console.log("Block timestamp: ", block.timestamp);
        console.log(""); // empty log line
        console.log("Defender: ", defender);
        console.log("Challenger: ", challenger);
        console.log(""); // empty log line

        for (uint256 i = 0; i < 100; i++) {
            previousWinner = winner;

            vm.startPrank(defender);
            // approval is cleared each time the NFT moved, so we do need this in the for loop
            oneShot.approve(address(rapBattle), 0);
            rapBattle.goOnStageOrBattle(0, 0);
            vm.stopPrank();

            vm.startPrank(challenger);
            oneShot.approve(address(rapBattle), 1);
            vm.recordLogs();
            rapBattle.goOnStageOrBattle(1, 0);
            vm.stopPrank();

            Vm.Log[] memory entries = vm.getRecordedLogs();
            winner = address(uint160(uint256(entries[0].topics[2])));

            if (i > 0) {
                assertEq(winner, previousWinner);
            }
        }

        console.log("Winner: ", winner);
    }
```

and run it by executing `forge test --mt test_weakRandomness -vvv`.

This gives the following output:
```javascript
Running 1 test for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_weakRandomness() (gas: 10371401)
Logs:
  Block timestamp:  1
  
  Defender:  0x4fAD2c378d74800f0Be9eAbE53e1236dE790806F
  Challenger:  0x2c0169543641108F2d2c34489DBDab1A54cF59b2
  
  Winner:  0x2c0169543641108F2d2c34489DBDab1A54cF59b2

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 33.72ms
```

According to the test, when the same challenger and defender battle for 100 times, the challenger wins in all 100 cases. Since the `RapperSkill` of the challenger's and the defender's NFTs are the same, each party should have a 50% chance of winning a battle. Winning 100 battles after each other has as less chance as 7.8886091e-29 %, so by random chance we certainly cannot expect one party to win all 100 battles, indicating weakness in the PRNG used in the protocol.

(Note: The testing environment provided by Foundry's Anvil is static, and block properties are not advanced except if specifically manipulated. If the randomness generation mechanism were truly random, we would expect to see variation in outcomes even in a controlled environment like Anvil. A truly random process should yield different outcomes under identical conditions because it does not depend solely on deterministic inputs.)

## Tools Used
Manual review, Foundry.

## Recommendations
Consider using a cryptographically provable random number generator, such as Chainlink VRF.
## <a id='H-02'></a>H-02. `CRED` Whales Can Submit Excessively Large Bets in `RapBattle:goOnStageOrBattle`, Pricing Others Out, Resulting in DoS            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L38

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L41

## Summary
A `CRED` whale (an entity with a large amount of the `CRED` token) can either accidentally or intentionally cause a Denial of Service (DoS) scenario in the `RapBattle` contract by submitting such a large bet that nobody else (or only the wealthiest individuals) can (or want) to match.

## Vulnerability Details
To participate in a battle, users need to call `RapBattle::goOnStageOrBattle` and as arguments to this function, they need to provide 
- (1) the ID of one of their rapper NFTs and 
- (2) the amount of `CRED` tokens they intend to submit as a bet. 

Since there is no upper limit to the bet amount, a `CRED` whale can either accidentally or intentionally submit such a large bet that nobody else (or only the wealthiest individuals) can (or want) to match. Since there are no options and mechanisms for:

- defender users (in this case the whale) to go off the stage (pull back from the battle),
- lowering a submitted bet,
- dropping defender users (in this case the whale) from the stage if they do not find a challenger for a pre-defined amount of time,

this effectively results in a DoS scenario so that other users are essentially prevented from participating in battles, given that they cannot meet an excessively high bet.

An entitiy can become a `CRED` whale multiple ways:
- by buying `CRED` from other users,
- by minting and staking (then unstaking, re-staking, etc.) a lot of rapper NFTs,
- by winning a lot of rap battles with significant bet amounts.

## Impact
First, the general accessibility of the `RapBattle` contract is compromised in a way that only the wealthiest players can participate in a battle, others are priced out.
Eventually, a single whale can create a DoS-like scenario where no one else can match an excessively high bet (that is, at least for a significant amount of time) and, hence, `RapBattle::goOnStageOrBattle` will be rendered unusbable.

## Proof of Code

The following piece of code demonstrates how a user can acquire a large `CRED` balance by buying and staking lots of rapper NFTs. 
Insert this test in `OneShotTest.t.sol`:

```javascript
    function test_dosByCredWhale() public {
        address whale = makeAddr("whale");

        vm.startPrank(whale);
        for (uint256 i = 0; i < 1000; i++) {
            oneShot.mintRapper();
            oneShot.approve(address(streets), i);
            streets.stake(i);
        }

        vm.warp(4 days + 1);

        for (uint256 i = 0; i < 1000; i++) {
            streets.unstake(i);
        }

        uint256 whaleBalance = cred.balanceOf(whale);
        console.log("Whale balance: ", whaleBalance);
        cred.approve(address(rapBattle), whaleBalance);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, whaleBalance); // nobody will be able to match such a high bet
        vm.stopPrank();
    }
```

and test it by executing `forge test --mt test_dosByCredWhale`. 

Output:

```
Running 1 test for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_dosByCredWhales() (gas: 141504568)
Logs:
  Whale balance:  4000

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 213.13ms
```
 

## Tools Used
Manual review, Foundry.

## Recommendations
There are a couple of possible solutions you can consider:

1. Introduce maximum bet limits: Establish a cap on the amount of `CRED` that can be wagered in any single battle. This limit could be a static value determined by the platform or dynamically adjusted based on the average bet sizes over a certain period.

2. Bet matching system: Implement a matching system that pairs users based on their wager sizes. This could involve creating different "leagues" or "tiers" of battles, where users are matched with opponents who have placed similar bet amounts, ensuring that all players, regardless of their cred holdings, have the opportunity to participate in battles.

3. Tiered participation fees: For users placing higher bets, introduce scaled participation fees, where the fee percentage increases with the size of the bet. This approach could discourage the placement of excessively large bets for the sole purpose of excluding others from participation.

4. Dynamic bet adjustment mechanism: Consider a system where, if a bet remains unmatched for a certain period, it is automatically reduced to a more reasonable amount based on recent average bet sizes. This ensures that battles remain active and accessible to a broader range of players.

## <a id='H-03'></a>H-03. Missing `CRED` Balance Check in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Withouth `Credbility` to Cover the Bet            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L49

## Summary
When a user enters a battle as a challenger (2nd player on stage), `RapBattle::goOnStageOrBattle` does not check whether the challenger actually has the `CRED` balance to cover the bet. Conseqently, challengers can battle without risking any `CRED` tokens.

## Vulnerability Details
To participate in a battle, users need to call `RapBattle::goOnStageOrBattle` and as arguments to this function, they need to provide 
- (1) the ID of one of their rapper NFTs and 
- (2) the amount of `CRED` tokens they intend to wager. 

However,`goOnStageOrBattle` does not check whether the challenger actually has the `CRED` balance to cover the wager amount. 

## Impact
The challenger can cheat the system and is in an unfair advantage. Having a zero balance of `CRED`, the challenger does not actually risk any funds when entering a battle, hence cannot lose. Since the defender has no option to withdraw from the battle once entered on the stage, the challenger can keep the defender "hostage" and keep battling it until chance turns to the challenger's favor so that the challenger wins.

Consider the following scenario:
1. Defender calls `RapBattle::goOnStageOrBattle`, and submits (1) the ID of one of its (not staked) rapper NFTs and (2) some amount of its `CRED` balance as a bet.
2. Defender's NFT and amount of `CRED` tokens submitted to the battle are transferred to `RapBattle`.
3. Challenger calls `RapBattle::goOnStageOrBattle`, but submits the ID of one of its (not staked) rapper NFTs, and submits the same bet amount as the defender without actually having any balance of `CRED`.
3. Assuming that the defender wins the battle, the challenger cannot pay up since it has a zero balance of `CRED`. The transaction (where the challenger entered the battle) reverts. Defender's NFT and bet is stuck in the `RapBattle` contract.
4. Challenger enters the battle again, the same way as in point 3.
5. Assuming that this time the challenger wins the battle, the defender's bet amount is transferred to the challenger.
6. Defender's NFT is finally transferred back to the defender.

## Proof of Code

Insert the following piece of code in `OneShotTest.t.sol`:

```javascript
   function test_challengerCanBattleWithoutCred() public {
        address defender = makeAddr("defender");
        address challenger2 = makeAddr("challenger2");

        // defender mints, stakes for 4 days, unstakes, then goes on stage
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.warp(4 days + 1); // with this, defender wins if run on anvil
        streets.unstake(0);
        uint256 defenderInitialBalance = cred.balanceOf(defender);
        assert(defenderInitialBalance == 4);
        uint256 credBet = defenderInitialBalance; // bet his whole balance
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), credBet);
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        // challenger battles without having cred tokens
        vm.startPrank(challenger2);
        oneShot.mintRapper();
        cred.approve(address(rapBattle), credBet);
        assert(cred.balanceOf(challenger2) == 0);
        assert(oneShot.ownerOf(1) == challenger2);
        oneShot.approve(address(rapBattle), 1);
        vm.recordLogs();
        vm.expectRevert();
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        // assertions and logs after 1st round
        Vm.Log[] memory entries = vm.getRecordedLogs(); 
        address winner = address(uint160(uint256(entries[0].topics[2])));
        assertEq(winner, defender);
        console.log("Winner round 1: ", winner);
        // defender's credbet and NFT is stuck in the rapBattle contract...
        // ...until a valid challanger comes or it loses against challenger2
        assert(cred.balanceOf(defender) == 0);
        assert(oneShot.ownerOf(0) == address(rapBattle));

        // 2nd round, challenger2 enters battle again as challenger of defender
        vm.prank(challenger2);
        vm.warp(4 days + 6); // with this, challenger wins if run on local anvil
        vm.recordLogs();
        rapBattle.goOnStageOrBattle(1, credBet);

        // assertions and logs after 2nd round
        Vm.Log[] memory entries_2 = vm.getRecordedLogs(); 
        winner = address(uint160(uint256(entries_2[0].topics[2])));
        assertEq(winner, challenger2);
        console.log("Winner round 2: ", winner);
        assert(cred.balanceOf(defender) == 0); // defender lost 4 cred
        assert(cred.balanceOf(challenger2) == 4); // challenger2 won 4 cred but did not risk any
    }
```

and test it by executing `forge test --mt test_challengerCanBattleWithoutHavingAnNftOrCredTokens`. 

Output:

```
Running 1 test for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_challengerCanBattleWithoutCred() (gas: 574049)
Logs:
  Winner round 1:  0x4fAD2c378d74800f0Be9eAbE53e1236dE790806F
  Winner round 2:  0xd9230796eAB96bC0dDFB7A732aF97F5b482C5b71

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.62ms
```

## Tools Used
Manual review, Foundry.

## Recommendations
Before starting the battle, check that the challenger has enough `CRED` balance to cover the bet:

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
+          require(credToken.balanceOf(msg.sender) >= _credBet, "Not enough CRED balance to cover the bet!")
            _battle(_tokenId, _credBet);
        }
    }
```
## <a id='H-04'></a>H-04. Missing Challenger Input Validation in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Without an NFT and With Zero `CRED` Balance            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L49

## Summary
When a user tries to enter a battle as challenger (2nd player), `RapBattle::goOnStageOrBattle` does check whether said user actually has the `CRED` balance to cover the bet amount, nor does it check whether said user has the NFT with the submitted ID. Conseqently, challengers can battle without having a rapper NFT, and without actually having any balance in `CRED` tokens. 

## Vulnerability Details
To participate in a battle, users need to call `RapBattle::goOnStageOrBattle` and as arguments to this function, they need to provide 
- (1) the ID of one of their rapper NFTs and 
- (2) the amount of `CRED` tokens they intend to wager. 

However,`goOnStageOrBattle` does not perform the neccessary input validation when a challenger (2nd player) tries to enter the battle. As a result, challengers can battle without having a rapper NFT, and without actually having any kind of balance in `CRED` tokens. This oversight undermines the fairness and integrity of battles, allowing challengers to participate risk-free,

## Impact

The challenger can cheat the system and is in an unfair advantage. Having a zero balance of `CRED`, the challenger does not actually risk any funds when entering a battle, hence cannot lose. Since the defender has no option to withdraw from the battle once entered on the stage, the challenger can keep the defender "hostage" and keep battling it until chance turns to the challenger's favor so that the challenger wins.

Consider the following scenario:
1. Defender calls `RapBattle::goOnStageOrBattle`, and submits (1) the ID of one of its (not staked) rapper NFTs and (2) some amount of its `CRED` balance as a bet.
2. Defender's NFT and amount of `CRED` tokens submitted to the battle are transferred to `RapBattle`.
3. Challenger calls `RapBattle::goOnStageOrBattle`, but submits the ID of an NFT that he does not actually own, and submits the same bet amount as the defender without actually having any balance of `CRED`.
3. Assuming that the defender wins the battle, the challenger cannot pay up since it has a zero balance of `CRED`. The transaction (where the challenger entered the battle) reverts. Defender's NFT and bet is stuck in the `RapBattle` contract.
4. Challenger enters the battle again, the same way as in point 3.
5. Assuming that this time the challenger wins the battle, the defender's bet amount is transferred to the challenger.
6. Defender's NFT is finally transferred back to the defender.

## Proof of Code

Insert the following piece of code in `OneShotTest.t.sol`:

```javascript

    function test_challengerCanBattleWithoutHavingAnNftOrCredTokens() public {
        address defender = makeAddr("defender");
        address challenger2 = makeAddr("challenger2");

        // defender mints, stakes for 4 days, unstakes, then goes on stage
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.warp(4 days + 1); // with this, defender wins if run on anvil
        streets.unstake(0);
        uint256 defenderInitialBalance = cred.balanceOf(defender);
        assert(defenderInitialBalance == 4);
        uint256 credBet = defenderInitialBalance; // bets his whole balance
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), credBet);
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        // challenger battles without having an NFT and cred tokens
        vm.startPrank(challenger2);
        assert(cred.balanceOf(challenger2) == 0);
        assert(oneShot.ownerOf(0) != challenger2);
        vm.recordLogs();
        vm.expectRevert();
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        // assertions and logs after 1st round
        Vm.Log[] memory entries = vm.getRecordedLogs(); 
        address winner = address(uint160(uint256(entries[0].topics[2])));
        assertEq(winner, defender);
        console.log("Winner round 1: ", winner);
        // defender's credbet and NFT is stuck in the rapBattle contract...
        // ...until a valid challanger comes or it loses against challenger2
        assert(cred.balanceOf(defender) == 0);
        assert(oneShot.ownerOf(0) == address(rapBattle));

        // 2nd round, challenger2 enters battle again as challenger of defender
        vm.prank(challenger2);
        vm.warp(4 days + 3); // with this, challenger wins if run on local anvil chain
        vm.recordLogs();
        rapBattle.goOnStageOrBattle(0, credBet);

        // assertions and logs after 2nd round
        Vm.Log[] memory entries_2 = vm.getRecordedLogs();
        winner = address(uint160(uint256(entries_2[0].topics[2])));
        assertEq(winner, challenger2);
        console.log("Winner round 2: ", winner);
        assert(cred.balanceOf(defender) == 0); // defender lost 4 cred
        assert(cred.balanceOf(challenger2) == 4); // challenger2 won 4 cred but did not risk any
    }
```

and test it by executing `forge test --mt test_challengerCanBattleWithoutHavingAnNftOrCredTokens`. 

Output:

```javascript
Running 1 test for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_challengerCanBattleWithoutHavingAnNftOrCredTokens() (gas: 475576)
Logs:
  Winner round 1:  0x4fAD2c378d74800f0Be9eAbE53e1236dE790806F
  Winner round 2:  0xd9230796eAB96bC0dDFB7A732aF97F5b482C5b71

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.50ms
```

## Tools Used
Manual review, Foundry

## Recommendations
Before starting the battle, check that the challenger
- actually has the NFT belonging to the ID he submitted to the battle.Before starting the battle, and
- has enough `CRED` balance to cover the bet.

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
+          require(oneShotNft.ownerOf(_tokenId) == msg.sender, "Not the owner of the NFT, or NFT is staked!");
+          require(credToken.balanceOf(msg.sender) >= _credBet, "Not enough CRED balance to cover the bet!")
            _battle(_tokenId, _credBet);
        }
    }
```


 
## <a id='H-05'></a>H-05. Missing NFT Ownership Check in `RapBattle::goOnStageOrBattle` Results in Challengers Being Able to Battle Without an NFT            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L49

## Summary
When a user enters a battle as a challenger (2nd player on stage), `RapBattle::goOnStageOrBattle` does not check whether the challenger actually has the NFT corresponding to the submitted ID. Conseqently, challengers can battle without having a rapper NFT, or can enter a battle with an NFT that they are staking in the `Streets` contract.

## Vulnerability Details
To participate in a battle, users need to call `RapBattle::goOnStageOrBattle` and as arguments to this function, they need to provide 
- (1) the ID of one of their rapper NFTs and 
- (2) the amount of `CRED` tokens they intend to wager. 

However,`goOnStageOrBattle` does not check whether the challenger actually holds the NFT. Conseqently, challengers can battle without having a rapper NFT, or with an NFT that they are actively staking.

## Impact

The challenger can cheat the system and is in an unfair advantage. For example, they can actively stake their NFT and enter the the battle with its ID.

Consider the following scenario:
1. Defender calls `RapBattle::goOnStageOrBattle`, and submits (1) the ID of one of its (not staked) rapper NFTs and (2) some amount of its `CRED` balance as a bet.
2. Defender's NFT and amount of `CRED` tokens submitted to the battle are transferred to `RapBattle`.
3. Challenger calls `RapBattle::goOnStageOrBattle`, but submits the ID of an NFT that he actively stakes (does not techically hold it), and submits the same bet amount as the defender.
4. The battle concludes without an issue. Assuming that the defender wins, the challenger's bet is transferred to the defender. All this time, however, the challenger was able to keep staking his NFT and accrue staking rewards.

## Proof of Code

Insert the following piece of code in `OneShotTest.t.sol`:

```javascript
   function test_challengerCanBattleWithoutNft() public {
        address defender = makeAddr("defender");
        address challenger2 = makeAddr("challenger2");

        // defender mints, stakes for 4 days, unstakes, then goes on stage
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.warp(4 days + 1);
        streets.unstake(0);
        uint256 defenderInitialBalance = cred.balanceOf(defender);
        assert(defenderInitialBalance == 4);
        uint256 credBet = defenderInitialBalance / 2; // bet only half his balance
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), credBet);
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        // challenger mints with ID = 1, stakes it for 4 days, unstakes it for rewards
        // then stakes it again, and battles with it while techically not having it
        vm.startPrank(challenger2);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 1);
        streets.stake(1);
        vm.warp(8 days + 6); // with this, defender wins if run on anvil local chain
        streets.unstake(1);
        oneShot.approve(address(streets), 1);
        streets.stake(1);
        assert(cred.balanceOf(challenger2) == 4); // has 4 cred
        assert(oneShot.ownerOf(1) != challenger2); // technically, it is locked in the streets contract
        cred.approve(address(rapBattle), credBet);
        vm.recordLogs();
        rapBattle.goOnStageOrBattle(0, credBet);
        vm.stopPrank();

        Vm.Log[] memory entries = vm.getRecordedLogs();
        address winner = address(uint160(uint256(entries[0].topics[2])));
        assertEq(winner, defender);
        console.log("Winner round 2: ", winner);
        assert(cred.balanceOf(defender) == defenderInitialBalance + credBet); // defender lost 2 cred
        assert(cred.balanceOf(challenger2) == defenderInitialBalance - credBet); // challenger2 won 2 cred but di
    }
```

and test it by executing `forge test --mt test_challengerCanBattleWithoutHavingAnNft`. 

Output:

```Running 2 tests for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_challengerCanBattleWithoutNft() (gas: 715298)
Logs:
  Winner round 2:  0x4fAD2c378d74800f0Be9eAbE53e1236dE790806F
```

## Tools Used
Manual review, Foundry.

## Recommendations
Before starting the battle, check whether the challenger actually has the NFT belonging to the ID he submitted to the battle:

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
+          require(oneShotNft.ownerOf(_tokenId) == msg.sender, "Not the owner of the NFT, or NFT is staked!");
            _battle(_tokenId, _credBet);
        }
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `ICredToken::approve`, `ICredToken::transfer` And `ICredToken::transferFrom` Have Incorrect ERC20 Function Interfaces            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/interfaces/ICredToken.sol#L7

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/interfaces/ICredToken.sol#L9

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/interfaces/ICredToken.sol#L11

## Vulnerability Details
`ICredToken::approve`, `ICredToken::transfer` and `ICredToken::transferFrom` have incorrect ERC20 function interfaces. These ERC20 functions should return a `bool` value to indicate success or failiure, but the `ICredToken` interface do not define these return values. A contract compiled with Solidity > 0.4.22 interacting with these functions will fail to execute them, as the return value is missing.

## Impact
Smart contracts interacting with `ICredToken` expecting boolean return values from these functions may revert or fail to execute as anticipated, leading to potential integration issues or loss of functionality. 

## Proof of Concept

Consider a DeFi platform that offers a staking feature, allowing users to stake ERC20 tokens in return for rewards:

```javascript
contract DeFiStakingContract {
    ICredToken public token;

    constructor(ICredToken _token) {
        token = _token;
    }

    function stakeTokens(uint256 _amount) public {
        // Expecting a boolean return value to ensure the transfer was successful
        require(token.transferFrom(msg.sender, address(this), _amount), "Transfer failed");
        
        // Logic to handle successful staking
    }
}

```
The `stakeTokens` function uses `require` to check the success of `transferFrom` by expecting a boolean return value. However, since `ICredToken`'s `transferFrom` does not return any value (due to the incorrect interface definition), the contract will revert and fail to execute, preventing users from staking their tokens. 

## Tools Used
Slither

## Recommendations
Update the `ICredToken` interface to align with the ERC20 standard by ensuring that `approve`, `transfer`, and `transferFrom` return a boolean value:

```diff
-    function approve(address to, uint256 amount) external;
-    function transfer(address to, uint256 amount) external;
-   function transferFrom(address from, address to, uint256 amount) external;
+    function approve(address to, uint256 amount) external returns (bool);
+    function transfer(address to, uint256 amount) external returns (bool);
+    function transferFrom(address from, address to, uint256 amount) external returns (bool);
```

# Low Risk Findings

## <a id='L-01'></a>L-01. `battlesWon` statistics in the `OneShot::RapperStats` Struct is Not Updated After Winning a Battle            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L81

## Summary
`battlesWon` is an element of the `OneShot::RapperStats` structure, a piece of the NFT's metadata, and is supposed to track how many battles has been won with a given rapper NFT. However, this element is never updated in the protocol, it remains 0 even if battles are won with an NFT.

## Impact
The metadata of the NFTs will be incorrect. Specifically, the `battlesWon` statistics will not correctly reflect the number of battles an NFT has previously won.

## Proof of Code

Insert the following piece of code in `OneShotTest.t.sol`:

```javascript
    function test_battlesWonNotUpdated() public twoSkilledRappers {
        vm.startPrank(user);
        uint256 initialuserBalance = cred.balanceOf(user);
        cred.approve(address(rapBattle), initialuserBalance);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, initialuserBalance);
        vm.stopPrank();

        vm.startPrank(challenger);
        cred.approve(address(rapBattle), initialuserBalance);
        // oneShot.approve(address(rapBattle), 1);
        vm.recordLogs();
        rapBattle.goOnStageOrBattle(1, initialuserBalance);
        vm.stopPrank();

        Vm.Log[] memory entries_2 = vm.getRecordedLogs();
        address winner = address(uint160(uint256(entries_2[0].topics[2])));

        assertEq(winner, challenger); // challenger won with NFT ID = 1
        assertEq(oneShot.ownerOf(1), challenger);

        stats = oneShot.getRapperStats(1);
        assertEq(stats.battlesWon, 0);
    }
```

and run it by executing `forge test --mt test_battlesWonNotUpdated`. 

Output:

```
Running 1 test for test/OneShotTest.t.sol:RapBattleTest
[PASS] test_battlesWonNotUpdated() (gas: 640173)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.50ms
```

## Tools Used
Manual reivew, Foundry.

## Recommendations
There are 2 possible solutions:

- remove this `battlesWon` element from the `OneShot::RapperStats` structure, and update the rest of the codebase accordingly. (Especially since it seems to be only a vanity metric that has no effect anywhere.), or
- update the `battlesWon` element after every battle as follows:

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
+           IOneShot.RapperStats memory stats = oneShotNft.getRapperStats(defenderTokenId);
+           uint256 updatedBattlesWon = stats.battlesWon + 1;
+           oneShotNft.updateRapperStats(defenderTokenId, stats.weakKnees, stats.heavyArms, stats.spaghettiSweater, stats.calmAndReady, updatedBattlesWon);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
+           IOneShot.RapperStats memory stats = oneShotNft.getRapperStats(_tokenId);
+           uint256 updatedBattlesWon = stats.battlesWon + 1;
+           oneShotNft.updateRapperStats(_tokenId, stats.weakKnees, stats.heavyArms, stats.spaghettiSweater, stats.calmAndReady, updatedBattlesWon);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }

```


