# First Flight #13: Baba Marta - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. NFT listing manipulation via reentrancy attack in the `MartenitsaMarketplace` contract](#H-01)
    - ### [H-02. Ether handling flaw in `MartenitsaMarketplace::buyMartenitsa` allows for unrecoverable overpayments](#H-02)
    - ### [H-03. `MartenitsaVoting::voteForMartenitsa` lets users to vote before voting period actually starts](#H-03)
    - ### [H-04. Incorrect rewards logic in `MartenitsaMarketplace::collectRewards`](#H-04)
    - ### [H-05. Missing access control in `MartenitsaToken::updateCountMartenitsaTokensOwner`](#H-05)
- ## Medium Risk Findings
    - ### [M-01. State management and access control issues due to inappropriate inheritance in `MartenitsaEvent`](#M-01)
    - ### [M-02. Improper handling of tied votes in `MartenitsaVoting::announceWinner` ](#M-02)
    - ### [M-03. `MartenitsaMarketplace::listMartenitsaForSale` allows owners to list their NFTs without providing approval for the marketplace to handle the token](#M-03)
    - ### [M-04. Internal accounting is not updated when NFTs are transferred via standard `ERC721` method `transferFrom`](#M-04)
    - ### [M-05. `MartenitsaVoting::announceWinner` does not check if the winner NFTs is still listed, risk of DoS](#M-05)
    - ### [M-06. `MartenitsaEvent::stopEvent` loops through an unbounded array, risk of eDoS/DoS](#M-06)
    - ### [M-07. `MartenitsaEvent::stopEvent` does not properly reset participant status after an event](#M-07)
- ## Low Risk Findings
    - ### [L-01. `MartenitsaMarketplace::makePresent` allows the transfer of listed NFTs (without clearing the listing)](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #13

### Dates: Apr 11th, 2024 - Apr 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 7
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. NFT listing manipulation via reentrancy attack in the `MartenitsaMarketplace` contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L78

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L39

## Summary

As a seller, a malicious smart contract can re-enter `MartentisaMarketplace::listMartenitsaForSale` from within `MartentisaMarketplace::buyMartenitsaMarketplace` when its `fallback` or `receive function` is triggered during the Ether transfer. The buyer will receive the NFT, but it will appear to be listed (at the re-list price set by the previous owner, the malicious smart contract).

## Vulnerability Details

`MartenitsaMarketplace::buyMartenitsa` allows users to purchase listed Martenitsa NFTs using Ether. This function transfers Ether to the seller, which will trigger the `fallback` or `receive` function if the seller is a smart contract. The protocol does not adequately mitigate the risk that the seller could be a malicious contract, exploiting potential reentrancy vulnerabilities. Malicious code within the `fallback` or `receive` functions can re-enter `MartenitsaMarketplace::listMartenitsaForSale`.

```javascript
    function buyMartenitsa(uint256 tokenId) external payable {
        Listing memory listing = tokenIdToListing[tokenId];
        require(listing.forSale, "Token is not listed for sale");
        require(msg.value >= listing.price, "Insufficient funds");

        address seller = listing.seller;
        address buyer = msg.sender;
        uint256 salePrice = listing.price;

        martenitsaToken.updateCountMartenitsaTokensOwner(buyer, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(seller, "sub");

        // Clear the listing
        delete tokenIdToListing[tokenId];

        emit MartenitsaSold(tokenId, buyer, salePrice);

        // Transfer funds to seller
@>        (bool sent,) = seller.call{value: salePrice}("");
        require(sent, "Failed to send Ether");

        // Transfer the token to the buyer
        martenitsaToken.safeTransferFrom(seller, buyer, tokenId);
    }
```

Conseqently, if the seller is a malicious smart contract, it can re-list an NFT that it is just in the process of being sold. The listing will be invalid (the NFT will not be able to be bought with those listing parameters), but merely the appearance of the listing can potentially create confusion and panic in the true real owner and potentially in other users.

The following test demonstrates how the vulnerability would be exploited. It also demonstrates the limits of the reentrancy attack, i.e. that the listing will actually be invalid:

<details>
<summary>Proof of Code</summary>
<details>
<summary>Test</summary>

Place this test in e.g. `MartenitsaToken.t.sol`:

```javascript
    function testReentrancy() public {
        uint256 price = 1 ether;
        uint256 relistPrice = 0.1 ether;
        uint256 attackedTokenId;
        ReentrancyAttacker attacker = new ReentrancyAttacker(marketplace, martenitsaToken);

        // attacker acquires an NFT: does not matter how, here Jack gives one to it
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        attackedTokenId = 0;
        martenitsaToken.approve(address(marketplace), attackedTokenId);
        marketplace.makePresent(address(attacker), attackedTokenId);
        vm.stopPrank();

        // attacker becomes a producer
        // in reality, it can become one only during an event
        // for simplicity, here the protocol owner assigns producer role to the attacker
        address[] memory newProducers = new address[](1);
        newProducers[0] = address(attacker);
        martenitsaToken.setProducers(newProducers);

        console.log("%e", address(attacker).balance);

        // attakcer lists the NFT
        attacker.list(attackedTokenId, price, relistPrice);

        // buyer buys NFT from the malicious contract
        address buyer = makeAddr("buyer");
        deal(buyer, price);
        vm.startPrank(buyer);
        marketplace.buyMartenitsa{value: price}(attackedTokenId);
        vm.stopPrank();

        console.log("%e", address(attacker).balance);

        // given the reentrancy attack, token 0 is relisted (although owned by buyer), now a lot cheaper
        assert(marketplace.getListing(attackedTokenId).forSale == true);
        assert(marketplace.getListing(attackedTokenId).price != price);
        assert(marketplace.getListing(attackedTokenId).price == relistPrice);

        // malicious contract acquires another NFT
        // this is needed otherwise underflow error
        // here the attacker gets one from Jack but it can also produce one while being a participant at an event
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 1);
        marketplace.makePresent(address(attacker), 1);
        vm.stopPrank();

        // suppose that the new owner of the NFT wants to flip it
        // (note that he buyer also needs to be a producer to be able to list (flip))
        // in preparation to sell, he approves the marketplace to spend the NFT
        vm.prank(buyer);
        martenitsaToken.approve(address(marketplace), attackedTokenId);

        // attacker notices the approval, jumps in to buy back the NFT cheap before the actual owner lists it with a proper price
        // but this fails as the protocol still thinks the owner of the NFT is the attacker and tries to trasnfer the NFT from it
        vm.expectRevert();
        attacker.buy(attackedTokenId, relistPrice);

        console.log("%e", address(attacker).balance);
        assert(martenitsaToken.ownerOf(0) != address(attacker));
        assert(martenitsaToken.ownerOf(0) == buyer);
    }
```
</details>
<details>
<summary>Malicious Contract</summary>

For this to work, place this contract in e.g. `MartenitsaToken.t.sol`, under the test contract. If done so, then 
- rename this test contract from `MartenitsaToken` to `MartenitsaTokenTest` to avoid conflicts with the `MartenitsaToken` contract from `MartenitsaToken.sol`, and
- ensure that all these imports are present in `MartenitsaToken.t.sol`:

<details>
<summary>Imports</summary>

```javascript
import {Test, console} from "forge-std/Test.sol";
import {BaseTest, MartenitsaMarketplace, MartenitsaToken} from "./BaseTest.t.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
```
</details>


```javascript
contract ReentrancyAttacker is IERC721Receiver {
    MartenitsaMarketplace marketplace;
    MartenitsaToken martenitsaToken;
    uint256 tokenId;
    uint256 relistPrice;
    bool attacked = false;

    constructor(MartenitsaMarketplace _marketplace, MartenitsaToken _martenitsaToken) {
        marketplace = _marketplace;
        martenitsaToken = _martenitsaToken;
    }

    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data)
        external
        override
        returns (bytes4)
    {
        return this.onERC721Received.selector;
    }

    function list(uint256 _tokenIdToList, uint256 _price, uint256 _relistPrice) external payable {
        tokenId = _tokenIdToList;
        martenitsaToken.approve(address(marketplace), tokenId);
        marketplace.listMartenitsaForSale(tokenId, _price);
        relistPrice = _relistPrice;
    }

    function buy(uint256 _tokenId, uint256 _rebuyPrice) external {
        marketplace.buyMartenitsa{value: _rebuyPrice}(_tokenId);
    }

    function _relistNftBeforeSell() internal {
        marketplace.listMartenitsaForSale(tokenId, relistPrice);
        attacked = true;
    }

    receive() external payable {
        if (!attacked) {
            _relistNftBeforeSell();
        }
    }

    fallback() external payable {
        if (!attacked) {
            _relistNftBeforeSell();
        }
    }
}

```
</details>
</details>

## Impact

As a seller, a malicious smart contract can re-list NFTs that are just in the process of being sold. The listing will be invalid, but it mere existence have multpile considerable implications:

- Confusion and possibly panic among users:
-- the new owner (the real owner) of the NFT might see its NFT listed on the marketplace when they did not list it,


-- by setting a low re-list price (could be 1 wei), the malicious seller could effectively tank the floor price of the whole collection. If done with multiple NFTs, the effect will be even more serious.


-- since the NFT appears to be listed, it can be voted on in a Martenitsa voting event.

- Gas wastage:
-- Prospecting buyers who intend to buy the maliciously re-listed NFT will waste gas calling `MartenitsaMarketplace::buyMartenitsa`, as the transaction will fail at line
```javascript
        martenitsaToken.safeTransferFrom(seller, buyer, tokenId);
```
(i.e. when the marketplace contract attempts to transfer from the NFT from the malicious contract to the prospecting new buyer, but the malicious contract is not the real owner anymore.)

- Cheating in the Martenitsa voting event:
-- since the NFT appears to be listed, it can be voted on in a Martenitsa voting event,


-- Based on the invalid re-list data, `MartenitsaEvent` will continue to recognize the malicious seller as the owner of the NFT. Consequently, if this NFT wins the vote, the `healthToken` rewards will be sent to the malicious seller contract, not the new actual true owner of the NFT.

```javascript
    function announceWinner() external onlyOwner {
        require(block.timestamp >= startVoteTime + duration, "The voting is active");

        uint256 winnerTokenId;
        uint256 maxVotes = 0;

        for (uint256 i = 0; i < _tokenIds.length; i++) {
            if (voteCounts[_tokenIds[i]] > maxVotes) {
                maxVotes = voteCounts[_tokenIds[i]];
                // @audit does not handle ties
                winnerTokenId = _tokenIds[i];
            }
        }

@>        list = _martenitsaMarketplace.getListing(winnerTokenId);
@>        _healthToken.distributeHealthToken(list.seller, 1);

        emit WinnerAnnounced(winnerTokenId, list.seller);
    }
```



## Tools Used
Manual review, Foundry.

## Recommendations
Implement changes that prevent the re-listing of NFTs while they are in the process of being bought:

```diff


```diff
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.21;

import "@openzeppelin/contracts/access/Ownable.sol";
import {HealthToken} from "./HealthToken.sol";
import {MartenitsaToken} from "./MartenitsaToken.sol";

contract MartenitsaMarketplace is Ownable {

    HealthToken public healthToken;
    MartenitsaToken public martenitsaToken;

    uint256 public requiredMartenitsaTokens = 3;

    struct Listing {
        uint256 tokenId;
        address seller;
        uint256 price;
        string design;
        bool forSale;
+       bool beingSold;
    }

    mapping(address => uint256) private _collectedRewards;
    mapping(uint256 => Listing) public tokenIdToListing;

    event MartenitsaListed(uint256 indexed tokenId, address indexed seller, uint256 indexed price);
    event MartenitsaSold(uint256 indexed tokenId, address indexed buyer, uint256 indexed price);

    constructor(address _healthToken, address _martenitsaToken) Ownable(msg.sender) {
        healthToken = HealthToken(_healthToken);
        martenitsaToken = MartenitsaToken(_martenitsaToken);
    }

    /**
    * @notice Function to list the martenitsa for sale. Only producers can call this function.
    * @param tokenId The tokenId of martenitsa.
    * @param price The price of martenitsa.
    */
    function listMartenitsaForSale(uint256 tokenId, uint256 price) external {
        require(msg.sender == martenitsaToken.ownerOf(tokenId), "You do not own this token");
        require(martenitsaToken.isProducer(msg.sender), "You are not a producer!");
        require(price > 0, "Price must be greater than zero");
+       require(!tokenIdToListing[tokenId].beingSold, "Cant list an NFT that is in the process of being sold");

        Listing memory newListing = Listing({
            tokenId: tokenId,
            seller: msg.sender,
            price: price,
            design: martenitsaToken.getDesign(tokenId),
-           forSale: true
+           forSale: true,
+           beingSold: false
        });

        tokenIdToListing[tokenId] = newListing;
        emit MartenitsaListed(tokenId, msg.sender, price);
    }

    /**
    * @notice Function to buy a martenitsa.
    * @param tokenId The tokenId of martenitsa.
    */
    function buyMartenitsa(uint256 tokenId) external payable {
        Listing memory listing = tokenIdToListing[tokenId];
        require(listing.forSale, "Token is not listed for sale");
        require(msg.value >= listing.price, "Insufficient funds");
+       require(!listing.beingSold, "Cant buy an NFT that is in the process of being sold");

+       // @notice cannot use listing.beingSold = true here, as that only modifies the memory var, not the storage
+       tokenIdToListing[tokenId].beingSold = true;  // set reentrancy guard
        address seller = listing.seller;
        address buyer = msg.sender;
        uint256 salePrice = listing.price;

        martenitsaToken.updateCountMartenitsaTokensOwner(buyer, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(seller, "sub");

         // Clear the listing
        delete tokenIdToListing[tokenId];

        emit MartenitsaSold(tokenId, buyer, salePrice);

        // Transfer funds to seller
        (bool sent, ) = seller.call{value: salePrice}("");
        require(sent, "Failed to send Ether");

        // Transfer the token to the buyer
        martenitsaToken.safeTransferFrom(seller, buyer, tokenId);

+       tokenIdToListing[tokenId].beingSold = false;  // reset reentrancy guard
  
    }

...
}
```
```
## <a id='H-02'></a>H-02. Ether handling flaw in `MartenitsaMarketplace::buyMartenitsa` allows for unrecoverable overpayments            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L63

## Summary

`MartenitsaMarketplace::buyMartenitsa` lets users to send more `ETH` to the contract than the price of the Martenitsa they intend to buy. These extra funds are not returned to the buyer but are kept in the contract which has no method for withdrawing `ETH`.

## Vulnerability Details

`MartenitsaMarketplace::buyMartenitsa` is designed to enable users to buy listed Martenitsa NFTs. When calling the function, users are supposed to send an `ETH` amount with the transaction to cover the price of the selected NFT.

`MartenitsaMarketplace::buyMartenitsa` accepts amounts higher than the NFT price, but is not prepared to handle the extra funds: nor does it refund the buyer, neither does it have a mechanism for withdrawing such funds.

The code below demonstrates that the contract accepts amounts higher than the NFT price.

<details>
<summary>Proof of code</summary>

```javascript
    function testEtherStuckInContract() public {
        address user = makeAddr("user");
        uint256 userEthBalance = 2 ether;
        uint256 price = 1 ether;

        // jack produces an NFT, lists it
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 0);
        marketplace.listMartenitsaForSale(0, price);
        vm.stopPrank();

        // user sends twice as much ETH than the price of the NFT
        vm.deal(user, userEthBalance);
        vm.startPrank(user);
        healthToken.approve(address(marketplace), userEthBalance);
        marketplace.buyMartenitsa{value: userEthBalance}(0);
        vm.stopPrank();

        assert(user.balance == 0);
        assert(jack.balance == price);
        assert(address(marketplace).balance == userEthBalance - price);
    }
```
</details>

## Impact

- Financial loss: Users who overpay for an NFT will lose the excess Ether they sent, as there is no mechanism to refund it. Note that sending more Ether than the price of the NFT is not necessarily a sign of the buyer's mistake or unawareness than can be easily avoided. Consider the following scenario

-- buyer submits the transaction to buy an NFT. At this point, the transaction value matches the NFT price.


-- seller submits a transaction to lower the price of the NFT. This happens about the same time when the buyer submits its transaction.


-- if the seller's transaction is added the blockchain earlier than the buyer's, the buyer will unavoidably overpay compared to the reduced price.

- Unrecoverable Ether: extra funds send to the contract are stuck in the contract forever.

## Tools Used
Manual review, Foundry.

## Recommendations
There are a few possible solutions, including:

- modify `MartenitsaMarketplace::buyMartenitsa` to reject payments that exceed the NFT price, or
- send back the extra funds to the buyer, or
- implement a withdraw function that enables the contract owner or any other predefined, trusted address to withdraw Ether from the contract.

For simplicity and fairness, consider implementing the first option by introducing the following modification:

```diff
    function buyMartenitsa(uint256 tokenId) external payable {
        Listing memory listing = tokenIdToListing[tokenId];
        require(listing.forSale, "Token is not listed for sale");
-        require(msg.value >= listing.price, "Insufficient funds");
+       require(msg.value == listing.price, "Payment must exactly match the listing price");

     // rest of the logic
     ...
     }
```
## <a id='H-03'></a>H-03. `MartenitsaVoting::voteForMartenitsa` lets users to vote before voting period actually starts            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L43-L52

## Summary

`MartenitsaVoting::voteForMartenitsa` misses to enforce that users can only vote after the voting has actually started. Consequently, users can start voting before the protocol owner would officially kick of voting by calling `MartenitsaVoting::startVoting`.

## Vulnerability Details

`MartenitsaEvent::voteForMartenitsa` is supposed to enable users to vote for listed Martenitsa NFTs within a predefined voting period. Voting period starts when the protocol owner calls `MartenitsaVoting::startVoting`. In this same call, the duration and, hence, the end of the period is also defined.

`MartenitsaEvent::voteForMartenitsa` correctly check whether the voting period has already ended or not, and if ended, does not allow more votes. It is, however, misses to check whether the voting period has actually started and, hence, users can start voting any time right after the creation of the contract.

This is demonstarted in the proof of code below:

<details>
<summary>Proof of Code</summary>

```javascript
    function testCanVoteBeforeVotingStarts() public {
        address user = makeAddr("user");
        uint256 price = 1 ether;

        // jack produces an NFT, lists it
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 0);
        marketplace.listMartenitsaForSale(0, price);
        vm.stopPrank();

        // user votes before voting starts
        vm.prank(user);
        voting.voteForMartenitsa(0);

        assert(voting.getVoteCount(0) == 1);

        // voting starts
        voting.startVoting();

        // early vote still counts
        assert(voting.getVoteCount(0) == 1);
    }
```
</details>

## Impact

- Users can start voting before the start of the actual voting period.
- Martentisa NFT owners noticing and exploiting this bug can get an unfair advantage in the voting: they will have more time to get and/or generate votes for their own listed NFTs. With this unfair advantage they can get the `healthToken` rewards reserved for the winner of the vote.

## Tools Used
Manual review, Foundry.

## Recommendations
Implement a check in `MartenitsaVoting::voteForMartenitsa` so that users will not be able to vote before the voting actually starts:

```diff
    function voteForMartenitsa(uint256 tokenId) external {
+     require(block.timestamp >= startVoteTime, "Voting is not active yet");
        require(!hasVoted[msg.sender], "You have already voted");
        require(block.timestamp < startVoteTime + duration, "The voting is no longer active");
        list = _martenitsaMarketplace.getListing(tokenId);
        require(list.forSale, "You are unable to vote for this martenitsa");

        hasVoted[msg.sender] = true;
        voteCounts[tokenId] += 1; 
        _tokenIds.push(tokenId);
    }
```
## <a id='H-04'></a>H-04. Incorrect rewards logic in `MartenitsaMarketplace::collectRewards`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L101-L107

## Summary
`MartenitsaMarketplace::collectRewards` contains a logic error that allows users to collect more `healthToken` rewards than they are entitled to.

## Vulnerability Details

`MartentisaToken::collectReward` is supposed to allow non-producer users to collect `healthToken` rewards for every three different Martenitsa NFTs they own. However, the function logic does not follow this intention. The function does not properly account for rewards that have already been collected, nor does it adequately update the internal state to reflect the number of rewards that should be available based on current NFT ownership.

In the provided test, a user acquires five NFTs and collects their corresponding rewards (1 `healthToken`). However, after receiving a sixth NFT, they are able to repeatedly collect rewards for the same NFTs without any limitations.

<details>
<summary>Proof of Code</summary>

```javascript
    function testRewardsLogicIsIncorrect() public {
        address user = makeAddr("user");

        // jack gives 5 NFTs to user as a gift
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 0);
        martenitsaToken.approve(address(marketplace), 1);
        martenitsaToken.approve(address(marketplace), 2);
        martenitsaToken.approve(address(marketplace), 3);
        martenitsaToken.approve(address(marketplace), 4);
        marketplace.makePresent(user, 0);
        marketplace.makePresent(user, 1);
        marketplace.makePresent(user, 2);
        marketplace.makePresent(user, 3);
        marketplace.makePresent(user, 4);
        vm.stopPrank();

        assert(martenitsaToken.ownerOf(0) == user);

        // user collects rewards for 3 NFTs
        vm.prank(user);
        marketplace.collectReward();

        // jack gives a 6th NFT to the user as present
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 5);
        marketplace.makePresent(user, 5);
        vm.stopPrank();

        // user can collect more rewards than eligible for
        for (uint256 i = 0; i < 100; i++) {
            vm.prank(user);
            marketplace.collectReward();
        }

        assert(healthToken.balanceOf(user) > 2);
        console.log("User's health token balance: %e", healthToken.balanceOf(user));
    }
```

Note that there is an additional, lot less obvious flaw. Confirmed by the protocol owner, the real goal is to ensure that rewards are granted based on unique sets of three NFTs, where reusing any NFT from a previously rewarded set disqualifies the new set from earning another reward (for the same user). 
I.e. all of the following must be satisfied for new users (who has not claimed any rewards before):

- `userA` can claim 1 `healthToken` for any set of 3 unique NFTs, e.g. for `tokenIdSet_1=[1,2,3]`, and then

-- `userA` cannot claim another `healthToken` with any set of 3 unique NFTs that share any elements with `tokenIdSet_1`. E.g. cannot claim another reward with `tokenIdSet_2=[1,4,5]` or  `tokenIdSet_3=[4,5,100]`.


-- `userA` can claim another `healthToken` with any set of 3 unique NFTs that DO NOT share any elements with `tokenIdSet_1`. E.g. they can claim with `tokenIdSet_4=[4,5,6]` or `tokenIdSet_5=[x,y,z]` where `x!=1`, x!=2`, x!=3`, y!=1`, y!=2`, y!=3`, z!=1`, z!=2`, z!=3`.


-- `userB` or any other user can claim 1 `healthToken` with `tokenIdSet_1=[1,2,3]`.

</details>


## Impact

- The obious and more serious bug lets users to repeatedly claim rewards for the same NFTs without any limitation.
- The less obvious flaw results in users being able to claim less rewards (provided they do not take advantage of the more serious bug) than they are entitled to (e.g. if `userA` claims rewards for `tokenIdSet_1=[1,2,3]`, then sells or gives away these 3 NFTs and acquires 3 new ones and tries to claim once again).

## Tools Used
Manual review, Foundry.

## Recommendations

- To fix the obvious and more serious bug, update `MartenitsaMarketplace` by correcting the reward tracking logic to accurately count already paid rewards as follows:

```diff
    function collectReward() external {
        require(!martenitsaToken.isProducer(msg.sender), "You are producer and not eligible for a reward!");
        uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(msg.sender);
        uint256 amountRewards = (count / requiredMartenitsaTokens) - _collectedRewards[msg.sender];
        if (amountRewards > 0) {
-            _collectedRewards[msg.sender] = amountRewards;
+           _collectedRewards[msg.sender] += amountRewards;
            healthToken.distributeHealthToken(msg.sender, amountRewards);
        }
    }
```

With this partial fix users are rewarded based on the _net increase_ in the number of complete sets (3 NFTs) they hold. Rewards can only be claimed for new sets that increase the user's holdings beyond previously rewarded levels. It effectively prevents exploiting the rewards system by recalculating eligible sets from scratch each time, thus aligning reward claims with actual holdings growth.

- To adjust the logic completely according to the intention, a comprehensive logic overhaul is needed. In `MartenitsaMarketplace`, implement set-based tracking to ensure no NFT is reused in multiple reward claims:

```diff
+ import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";
...

contract MartenitsaMarketplace {
+    using EnumerableSet for EnumerableSet.UintSet;

+    mapping(address => EnumerableSet.UintSet) private claimedTokens;

+    function claimReward(uint256[] calldata tokenIds) external {
+        require(tokenIds.length == 3, "Must provide exactly three tokens");
+        require(areTokensUnique(tokenIds), "Tokens must be unique and unclaimed");

        // Mark tokens as claimed
+        for (uint256 i = 0; i < tokenIds.length; i++) {
+            claimedTokens[msg.sender].add(tokenIds[i]);
+        }

        // Logic to distribute rewards
+        distributeReward(msg.sender);
+    }

+    function areTokensUnique(uint256[] memory tokenIds) private view returns (bool) {
+        for (uint256 i = 0; i < tokenIds.length; i++) {
+            if (claimedTokens[msg.sender].contains(tokenIds[i])) {
+                return false;
+            }
+        }
+        return true;
+    }

+    function distributeReward(address recipient) private {
+                    healthToken.distributeHealthToken(msg.sender, 1);
+    }
+}

```
## <a id='H-05'></a>H-05. Missing access control in `MartenitsaToken::updateCountMartenitsaTokensOwner`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaToken.sol#L62

## Summary
`MartenitsaToken:: updateCountMartenitsaTokensOwner` does not have access control: anyone can call this function and can alter the internal accounting of token counts for any address.

## Vulnerability Details
`MartenitsaToken::updateCountMartenitsaTokensOwner` is supposed to update `mapping(address => uint256) public countMartenitsaTokensOwner` whenever a Marenitsa NFT is minted or changes owners. The mapping `countMartenitsaTokensOwner` is used for internal accounting and is supposed to reflect how many Martenitsa NFTs are owned by each address (kind of like the standard `ERC721` function `balanceOf()`). However, since `MartenitsaToken::updateCountMartenitsaTokensOwner` does not have any access control and anybody can call it, this internal accounting can be manipulated.

Consider the following test that demonstrates this vulnerability:

<details>
<summary> Proof of Code </summary>

```javascript
    function testAnyoneCanCallUpdateCountMartenitsaTokensOwner() public {
        address user = makeAddr("user");

        // jack mints 1 NFT
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        vm.stopPrank();

        uint256 userInternalBalance_before = martenitsaToken.getCountMartenitsaTokensOwner(user); // 0
        uint256 jackInternalBalance_before = martenitsaToken.getCountMartenitsaTokensOwner(jack); // 1

        vm.startPrank(user);
        martenitsaToken.updateCountMartenitsaTokensOwner(user, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(jack, "sub");
        vm.stopPrank();

        uint256 userInternalBalance_after = martenitsaToken.getCountMartenitsaTokensOwner(user); // 1
        uint256 jackInternalBalance_after = martenitsaToken.getCountMartenitsaTokensOwner(jack); // 0

        // internal accounting has been manipulated
        assert(userInternalBalance_after > userInternalBalance_before); // 1 > 0
        assert(jackInternalBalance_after < jackInternalBalance_before); // 0 < 1

        // internal accounting does not match reality
        assert(userInternalBalance_after != martenitsaToken.balanceOf(user)); // 1 != 0
        assert(jackInternalBalance_after != martenitsaToken.balanceOf(jack)); // 0 != 1

        // an earlier state of internal accounting matches reality
        assert(userInternalBalance_before == martenitsaToken.balanceOf(user)); // 0 = 0
        assert(jackInternalBalance_before == martenitsaToken.balanceOf(jack)); // 1 = 1
    }
```
</details>

## Impact
1. Internal accounting (`martenitsaToken.countMartenitsaTokensOwner()`) will not reflect true ownership status (`martenitsaToken.balanceOf()`).

2. A malicious user can alter the internal accounting in a way that the contract will think it has a lot more Martenitsa NFTs than it actually does, and then (after meeting all the other requirements for rewards collection) can collect more `healthToken` rewards from `MartenitsaMarketplace::collectRewards` than it is entitled to.

## Tools Used
Manual review, Foundry.

## Recommendations
Consider the following options and select whichever matches your current and future requirements more appropriately:

1. Replace the `countMartenitsaTokensOwner` with the the standard `ERC721` `balanceOf()`, as the former appears to duplicate the functionality already provided by the latter (i.e. tracking the number of tokens owned by each address). In this case you can completely remove not only the mapping but also the `updateCountMartenitsaTokensOwner` function.
(According to the protocol owner, this is the less favorable option, the protocol prefers to keep the internal accounting.)

2. Implement access control for `MartenitsaToken::updateCountMartenitsaTokensOwner` by modifying `MartenitsaToken.sol` as follows:

```diff
+   address marketplaceAddress;

+    function setMarketplaceAddress(address _marketplace) external onlyOwner {
+        require(marketplaceAddress == address(0), "Marketplace already set"); // Optional: Ensure only set once
+        marketplaceAddress = _marketplace;
+    }

...

    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
+        require(marketplaceAddress != address(0), "Marketplace address is not set yet.");
+        require(msg.sender == marketplaceAddress, "Unauthorized call");
        // existing logic
    }
```


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. State management and access control issues due to inappropriate inheritance in `MartenitsaEvent`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L7

## Summary

The `MartenitsaEvent` contract currently from `MartenitsaToken`, which unnecessarily exposes token management functionalities. This leads to shared state variables that can result in synchronization and access control issues across the codebase. Such an inheritance structure contradicts the principle of separation of concerns between event management and token management.

## Vulnerability Details

`MartenitsaEvent` is supposed to contain event management functionalities, which is a completely separate concern from the NFT token management and producer management functionalities provided by `MartenitsaToken`.

However, `MartenitsaEvent` extends `MartenitsaToken` via inheritance, inappropriately merging two disparate concerns. This inheritance structure gives `MartenitsaEvent` unnecessary access to the internal mechanisms (e.g. NFT minting method) and state variables (e.g. producer lists) of `MartenitsaToken`. 

This setup causes several problems, probably the biggest of which is that both contracts maintain separate and uncoordinated internal states regarding which addresses are recognized as producers. The test below demonstrates one such state inconcistency between the 2 contacts (one contract recognizes an address as producer, the other not).

<details>
<summary>Proof of Code</summary>

```javascript
    function testInconsistentProducerStates() public {
        // MartenitsaToken.sol recognizes Jack as producer
        assert(martenitsaToken.isProducer(jack) == true);
        // MartenitsaEvent.sol DOES NOT recognize Jack as producer
        assert(martenitsaEvent.isProducer(jack) == false);

        // accordingly, Jack can join an event even though as a producer he is not supposed to
        deal(address(healthToken), jack, 1 ether);
        martenitsaEvent.startEvent(1 days);
        vm.startPrank(jack);
        healthToken.approve(address(martenitsaEvent), 1 ether);
        martenitsaEvent.joinEvent();
        vm.stopPrank();
        assert(martenitsaEvent.getParticipant(jack) == true);
    }
```
</details>


## Impact

- State Confusion: both contracts will maintain separate and uncoordinated internal states regarding which addresses are recognized as producers. I.e. addresses denoted as producers by `MartenitsaToken` will not be recognized as producers by `MartenitsaEvent` and vice versa.

- Access Control Issues: state confusion causes access control issues, e.g.

-- Addresses recognized as producers by `MartenitsaToken` are able to join an event via `MartenitsaEvent::joinEvent`, even though as producers they are not supposed to. This is because `MartenitsaEvent::joinEvent` only checks its own internal state regarding producer lists:

```javascript
        require(!isProducer[msg.sender], "Producers are not allowed to participate");
```

-- Addresses recognized as producers by `MartenitsaEvent` (during the duration of an event) are able to collect rewards via `MartentisaMartkerplace::collectRewards` even during an event, although they are not supposed to. This is because `MartentisaMartkerplace::collectRewards` checks only `MartenitsaToken`'s internal state regarding producer lists:

```javascript
        require(!martenitsaToken.isProducer(msg.sender), "You are producer and not eligible for a reward!");
```

--  Addresses recognized as producers by `MartenitsaEvent` (during the duration of an event) are not able to create Martinetsa NFTs via `MartinetsaToken::createMartinetsa`, only via `MartinetsaEvent::createMartinetsa` which creates further state inconsistencies and accounting issues.

- Function Exposure: `MartenitsaEvent` gains access to NFT-specific functions that are irrelevant to its primary role of managing events. This not only adds unnecessary complexity to `MartenitsaEvent` but also increases the attack surface of the codebase.


## Tools Used
Manual review, Foundry.

## Recommendations

`MartenitsaEvent` should not inherit from `MartenitsaToken`. Instead, it should contain an instance of `MartenitsaToken` to allow explicit and controlled use of its functionalities. This change will limit `MartenitsaEvent` to necessary interactions with `MartenitsaToken` and prevent inadvertent exposure of token management features.

Consider the following modifications as a rough starting point for a fix. Note that the bug at hand is severe and fixing it requires rethinking the protocol design, thorough testing, and preferably another round of audit.

<details>
<summary>Implemented Fix Suggestion</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import {HealthToken} from "./HealthToken.sol";
import {MartenitsaToken} from "./MartenitsaToken.sol";

- contract MartenitsaEvent is MartenitsaToken {
+ contract MartenitsaEvent {
    
+    MartenitsaToken private _martenitsaToken;
    HealthToken private _healthToken;

    uint256 public eventStartTime;
    uint256 public eventDuration;
    uint256 public eventEndTime;
    uint256 public healthTokenRequirement = 10 ** 18;
    address[] public participants;

    mapping(address => bool) private _participants;

    event EventStarted(uint256 indexed startTime, uint256 indexed eventEndTime);
    event ParticipantJoined(address indexed participant);

-   constructor(address healthToken) onlyOwner {
+   constructor(address martenitsaToken, address healthToken) onlyOwner {
+       _martenitsaToken = MartenitsaToken(martenitsaToken);
        _healthToken = HealthToken(healthToken);
    }

    /**
    * @notice Function to start an event.
    * @param duration The duration of the event.
    */
    function startEvent(uint256 duration) external onlyOwner {
        eventStartTime = block.timestamp;
        eventDuration = duration;
        eventEndTime = eventStartTime + duration;
        emit EventStarted(eventStartTime, eventEndTime);
    }

    /**
    * @notice Function to join to event. Each participant is a producer during the event.
    * @notice The event should be active and the caller should not be joined already.
    * @notice Producers are not allowed to participate.
    */
    function joinEvent() external {
        require(block.timestamp < eventEndTime, "Event has ended");
        require(!_participants[msg.sender], "You have already joined the event");
-       require(!isProducer[msg.sender], "Producers are not allowed to participate");
+        require(!_martenitsaToken.isProducer[msg.sender], "Producers are not allowed to participate");
        require(_healthToken.balanceOf(msg.sender) >= healthTokenRequirement, "Insufficient HealthToken balance");
        
        _participants[msg.sender] = true;
        participants.push(msg.sender);
        emit ParticipantJoined(msg.sender);
        
        (bool success) = _healthToken.transferFrom(msg.sender, address(this), healthTokenRequirement);
        require(success, "The transfer is not successful");
        _addProducer(msg.sender);

    }

    /**
    * @notice Function to remove the producer role of the participants after the event is ended.
    */
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
        for (uint256 i = 0; i < participants.length; i++) {
-            isProducer[participants[i]] = false;
+            _martenitsaToken.removeProducer(participants[i]);
+            // note that `martenitsaToken.producers` also has to be updated, but that is considered a separate bug
        }
    }

    /**
    * @notice Function to get information if a given address is a participant in the event.
    */
    function getParticipant(address participant) external view returns (bool) {
        return _participants[participant];
    }

    /**
    * @notice Function to add a new producer.
    * @param _producer The address of the new producer.
    */
    function _addProducer(address _producer) internal {
-        isProducer[_producer] = true;
-        producers.push(_producer);
+        // note: creating an array every time just to add a single producer might be less efficient. 
+        // instead, consider adding a method in MartenitsaToken
+        address[] memory newProducers = new address[](1);
+        newProducers[0] = _producer;
+        _martenitsaToken.setProducers(newProducers);
    } 
}
```


```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MartenitsaToken is ERC721, Ownable {
    
    uint256 private _nextTokenId;
    address[] public producers;

+   address internal _eventContractAddress;


    mapping(address => uint256) public countMartenitsaTokensOwner;
    mapping(address => bool) public isProducer;
    mapping(uint256 => string) public tokenDesigns;

    event Created(address indexed owner, uint256 indexed tokenId, string indexed design);

    constructor() ERC721("MartenitsaToken", "MT") Ownable(msg.sender) {}



+    function setEventContractAddress(address _eventAddress) external onlyOwner {
+        require(_eventContractAddress == address(0), "Event contract address already set"); // Optional: Ensure only set once
+        _eventContractAddress = _eventAddress;
+    }

    /**
    * @notice Function to set producers.
    * @param _producersList The addresses of the producers.
    */
    function setProducers(address[] memory _producersList) public onlyOwner{
        for (uint256 i = 0; i < _producersList.length; i++) {
+           if(!isProducer[_producersList[i]]){
                isProducer[_producersList[i]] = true;
                producers.push(_producersList[i]);
+           }
        }
    }

+    // note: this is costly... this should be optimized
+    function removeProducer(address _producer) public {
+        require(msg.sender == _eventContractAddress);   
+        require(isProducer[_producer], "Not a producer");
+        isProducer[_producer] = false;
+        for (uint256 i = 0; i < producers.length; i++) {
+            if (producers[i] == _producer) {
+                producers[i] = producers[producers.length - 1];
+                producers.pop();
+                break;
+            }
+        }
+    }

    /**
    * @notice Function to create a new martenitsa. Only producers can call the function.
    * @param design The type (bracelet, necklace, Pizho and Penda and other) of martenitsa.
    */
    function createMartenitsa(string memory design) external {
        require(isProducer[msg.sender], "You are not a producer!");
        require(bytes(design).length > 0, "Design cannot be empty");
        
        uint256 tokenId = _nextTokenId++;
        tokenDesigns[tokenId] = design;
        countMartenitsaTokensOwner[msg.sender] += 1;

        emit Created(msg.sender, tokenId, design);

        _safeMint(msg.sender, tokenId);
    }

    /**
    * @notice Function to get the design of a martenitsa.
    * @param tokenId The Id of the MartenitsaToken.
    */
    function getDesign(uint256 tokenId) external view returns (string memory) {
        require(tokenId < _nextTokenId, "The tokenId doesn't exist!");
        return tokenDesigns[tokenId];
    }

    /**
    * @notice Function to update the count of martenitsaTokens for a specific address.
    * @param owner The address of the owner.
    * @param operation Operation for update: "add" for +1 and "sub" for -1.
    */
    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
        if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("add"))) {
            countMartenitsaTokensOwner[owner] += 1;
        } else if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("sub"))) {
            countMartenitsaTokensOwner[owner] -= 1;
        } else {
            revert("Wrong operation");
        }
    }

    /**
    * @notice Function to get the count of martenitsaTokens for a specific address.
    * @param owner The address of the owner.
    */
    function getCountMartenitsaTokensOwner(address owner) external view returns (uint256) {
        return countMartenitsaTokensOwner[owner];
    }

    /**
    * @notice Function to get the list of addresses of all producers.
    */
    function getAllProducers() external view returns (address[] memory) {
        return producers;
    }

    /**
    * @notice Function to get the next tokenId.
    */
    function getNextTokenId() external view returns (uint256) {
        return _nextTokenId;
    }
}
```

</details>


Note that a better, more elegant and simpler fix might be the following:

- remove the inheritance
- do not manipulate the state variables of `MartenitsaMarketplace` to grant producer privilages to event participants for the duration of an event, but instead:

--  completely separate participant and producer statuses, and use only the `participants` array and `_participants` mapping for participants (and producer state variables only for real producers set by `MartenitsaMarketplace`),


-- in all 5 contracts, whereever an action is gated for producers or non-producers, add a similar gate to the participants by referencing the particpants state variable of `MartenitsaEvent`. (For this to work, importing `MartenitsaEvent`will be needed in all affected contracts.)

This solution completely separates the handling of producers from participants. As such, it has the additional benefit that whenever an event ends, and the producer-like privilages of participants have to be revoked, it is enough to simply reset the `participants` array and `_participants` mapping.
## <a id='M-02'></a>M-02. Improper handling of tied votes in `MartenitsaVoting::announceWinner`             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L63-L71

## Summary
`MartenitsaVoting::announceWinner` fails to properly address scenarios where multiple NFTs receive the same highest number of votes. 

## Vulnerability Details

`MartenitsaVoting::announceWinner` is supposed to 
- identify the NFT(s) with the highest number of votes at the end of the voting period,  
- announce it/them as the winner(s), and 
- reward the the owner(s) of the NFT(s) with 1 `healthToken`. 

However, the current implementation does not handle cases where two or more NFTs receive the highest but equal number of votes. 

The test below simulates a voting scenario where:

- Two NFTs (token `ID 0` and token `ID 1`) are listed and receive an equal number of votes (1 each).
- The voting concludes, and the winner is announced.
- Despite the tie, only the owner of token `ID 0` receives the reward.

<details>
<summary>Proof of Code</summary>

```javascript
    function testProtocolCantHandleTiedVotes() public {
        uint256 price = 1 ether;
        address voter1 = makeAddr("voter1");
        address voter2 = makeAddr("voter2");

        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        marketplace.listMartenitsaForSale(0, price);
        vm.stopPrank();

        vm.startPrank(chasy);
        martenitsaToken.createMartenitsa("bracelet");
        marketplace.listMartenitsaForSale(1, price);
        vm.stopPrank();

        voting.startVoting();

        vm.prank(voter1);
        voting.voteForMartenitsa(0);

        vm.prank(voter2);
        voting.voteForMartenitsa(1);

        vm.warp(block.timestamp + 1 days + 1 seconds);

        voting.announceWinner();

        // NFTs with tokenId=0 and tokenID=1 both have 1 vote (highest vote count)
        assert(voting.getVoteCount(0) == 1);
        assert(voting.getVoteCount(1) == 1);

        console.log(healthToken.balanceOf(jack));
        console.log(healthToken.balanceOf(chasy));

        // only the owner of tokenId=0 is rewarded
        assert(healthToken.balanceOf(jack) == 1 ether);
        assert(healthToken.balanceOf(chasy) == 0);
    }
```
</details>

## Impact

In cases when mulitple NFTs have the highest but equal number of votes (i.e. tied winners), the contract arbitrarily rewards the NFT that appears first in the iteration that still has the maximum votes, while the other "winners" will not get any rewards.

## Tools Used
Manual review, Foundry.

## Recommendations

Adjust the code in `MartenitsaVoting` as follows:

```diff
-    function announceWinner() external onlyOwner {
-        require(block.timestamp >= startVoteTime + duration, "The voting is active");
-
-        uint256 winnerTokenId;
-        uint256 maxVotes = 0;
-
-        for (uint256 i = 0; i < _tokenIds.length; i++) {
-            if (voteCounts[_tokenIds[i]] > maxVotes) {
-                maxVotes = voteCounts[_tokenIds[i]];
-                winnerTokenId = _tokenIds[i];
-            }
-        }
-
-        list = _martenitsaMarketplace.getListing(winnerTokenId);
-        _healthToken.distributeHealthToken(list.seller, 1);

-        emit WinnerAnnounced(winnerTokenId, list.seller);
-    }

+ function announceWinner() external onlyOwner {
+    require(block.timestamp >= startVoteTime + duration, "The voting is active");

+
+    uint256 maxVotes = 0;

+    // First, determine the maximum number of votes
+    for (uint256 i = 0; i < _tokenIds.length; i++) {
+        if (voteCounts[_tokenIds[i]] > maxVotes) {
+            maxVotes = voteCounts[_tokenIds[i]];
+        }
+    }
+
+    // Now, distribute rewards to all listings with the maximum votes
+    for (uint256 i = 0; i < _tokenIds.length; i++) {
+        if (voteCounts[_tokenIds[i]] == maxVotes) {
+            list = _martenitsaMarketplace.getListing(_tokenIds[i]); 
+            _healthToken.distributeHealthToken(list.seller, 1);  
+            emit WinnerAnnounced(_tokenIds[i], list.seller);
+        }
+    }
+
+ }

```

## <a id='M-03'></a>M-03. `MartenitsaMarketplace::listMartenitsaForSale` allows owners to list their NFTs without providing approval for the marketplace to handle the token            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L39

## Summary
`MartenitsaMarketplace::listMartenitsaForSale` lets Martenitsa NFT owners to list their NFTs on the marketplace, but it does not require the existance of owner approval for the marketplace to actually handle the NFT. If the approval is not made, the NFT will be impossible to buy despite the apparent listing.

## Vulnerability Details
`MartenitsaMarketplace::listMartenitsaForSale` is supposed to allow users to list their Martenitsa NFTs for sale on the marketplace. After successful execution of the call to `listMartenitsaForSale`, the NFT appears as listed and as such, is supposed to be able to be bought by another user who calls `MartenitsaMarketplace::buyMartenitsa`. However, this is not always the case. 

A preprequisite for successfully purchasing a listed NFT is that the seller approves the marketplace to move its listed token when it is purchased. Since `MartenitsaMarketplace::listMartenitsaForSale` does not check the existence of such an approval, unapproved NFTs can appear as listed but cannot be actually bought.

<details>
<summary>Proof of Code</summary>

```javascript
    function testNftListedWithoutApprovalCantBeBought() public {
        address user = makeAddr("user");
        uint256 price = 1 ether;

        // give user ETH
        vm.deal(user, 5 ether);

        // jack mints and NFT and then successfully lists it without approval
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        assert(martenitsaToken.ownerOf(0) == jack);
        assert(martenitsaToken.isProducer(jack) == true);
        marketplace.listMartenitsaForSale(0, price);
        assert(marketplace.getListing(0).forSale == true);
        vm.stopPrank();

        // @note
        bytes memory expectedRevertReason =
            abi.encodeWithSignature("ERC721InsufficientApproval(address,uint256)", address(marketplace), 0);

        // user sees the listing, attempts to buy the NFT but the trx will fail
        // because Jack did not do the approval
        vm.expectRevert(expectedRevertReason);
        vm.prank(user);
        marketplace.buyMartenitsa{value: 1 ether}(0);
    }
```
</details>

## Impact
NFTs that are listed by their owners without them providing their approval for the marketplace to move the listed token during a purchase attempt will appear as listed but will not be able to be actually bought. The implications are as follows:

- An owner who genuinly forgets to provide the approval will think that the listing is successful. Seeing that nobody buys the apparently listed NFT, the owner might list the NFT at a lower price:

-- If the owner provides the approval this time, the NFT might sell at a lower price than it actually would had. 


-- If the owner does not provide the approval this time either, the NFT will not be sold, so the owner might keep lowering the price further, eventually tanking the floor price of the whole collection.

- An owner might intentionally not provide the approval to

-- mislead other users with the apparent listing

-- apparently tank the FP of the collection

-- to acquire and keep eligibility for getting votes on the NFT from users calling `MartenitsaVoting::voteForMartenitsa`, without a real intention to sell it. In this scenario the final goal is of course to win the vote and get the rewards for the win.


- Unsuspecting users trying to buy the apparently listed but not actually purchaseable NFTs will waste gas on failed transactions.




## Tools Used
Manual review, Foundry.

## Recommendations
Check whether the approval for the marketplace to manage to token being listed actually exists. Make the following modifications in `MartenitsaMarketplace`:

```diff
    function listMartenitsaForSale(uint256 tokenId, uint256 price) external {
        require(msg.sender == martenitsaToken.ownerOf(tokenId), "You do not own this token");
+      require(martenitsaToken.getApproved(tokenId) == address(this), "Provide approval for the marketplace to manage your token");
        require(martenitsaToken.isProducer(msg.sender), "You are not a producer!");
        require(price > 0, "Price must be greater than zero");

        Listing memory newListing = Listing({
            tokenId: tokenId,
            seller: msg.sender,
            price: price,
            design: martenitsaToken.getDesign(tokenId),
            forSale: true
        });

        tokenIdToListing[tokenId] = newListing;
        emit MartenitsaListed(tokenId, msg.sender, price);
    }
```
## <a id='M-04'></a>M-04. Internal accounting is not updated when NFTs are transferred via standard `ERC721` method `transferFrom`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol#L137

## Summary
Using `MartenitsaToken::transferFrom` (a function `MartentisaToken` inherits from `ERC721.sol`) users can transfer NFTs between themselves. Since this function is not overriden, the logic responsible for updating internal accounting `mapping(address => uint256) public countMartenitsaTokensOwner` is not triggered.

## Vulnerability Details
`MartenitsaToken::updateCountMartenitsaTokensOwner` is supposed to update `mapping(address => uint256) public countMartenitsaTokensOwner` whenever a Marenitsa NFT is minted or changes owners. The mapping `countMartenitsaTokensOwner` is used for internal accounting and is supposed to reflect how many Martenitsa NFTs are owned by each address.

However, this internal accounting is not updated when users transfer NFTs via the standard `ERC721` method `transferFrom`.

Consider the following test that demonstrates this vulnerability:

<details>
<summary> Proof of Code </summary>

```javascript
function testInternalAccountingNotUpdatedWhenNftIsTransferredViaStandardErc721Method() public {
        address user = makeAddr("user");

        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        vm.stopPrank();

        uint256 userInternalBalance_before = martenitsaToken.getCountMartenitsaTokensOwner(user); // 0
        uint256 jackInternalBalance_before = martenitsaToken.getCountMartenitsaTokensOwner(jack); // 1

        vm.startPrank(jack);
        assert(martenitsaToken.ownerOf(0) == jack);
        martenitsaToken.approve(user, 0);
        vm.stopPrank();

        vm.prank(user);
        martenitsaToken.transferFrom(jack, user, 0);

        uint256 userInternalBalance_after = martenitsaToken.getCountMartenitsaTokensOwner(user); // 0
        uint256 jackInternalBalance_after = martenitsaToken.getCountMartenitsaTokensOwner(jack); // 1

        // internal accounting has not changed, has not tracked transfer via standard ERC721 function
        assert(userInternalBalance_after == userInternalBalance_before); // 0 = 0
        assert(jackInternalBalance_after == jackInternalBalance_before); // 1 = 1

        // internal accounting does not match reality
        assert(userInternalBalance_after != martenitsaToken.balanceOf(user)); // 0 != 1
        assert(jackInternalBalance_after != martenitsaToken.balanceOf(jack)); // 1 != 0
    }
```
</details>

## Impact

1. Internal accounting (`martenitsaToken.countMartenitsaTokensOwner()`) will not refelct true ownership status (`martenitsaToken.balanceOf()`).

2. Users whose internal accounting data does not reflect true ownership status will be able to claim either less or more `healthToken` rewards from `MartenitsaMarketplace::collectRewards`  than they are entitled to.

## Tools Used
Manual review, Foundry.

## Recommendations
Consider the following options and select whichever matches your current and future requirements more appropriately:

1. Replace the `countMartenitsaTokensOwner` with the the standard `ERC721` `balanceOf()`, as the former appears to duplicate the functionality already provided by the latter (i.e. tracking the number of tokens owned by each address).  In this case you can completely remove not only the mapping but also the `updateCountMartenitsaTokensOwner` function.
(According to the protocol owner, this is the less favorable option, the protocol prefers to keep the internal accounting.)

2. Override the standard `ERC721` method `transferFrom` by modifying `MartenitsaToken.sol` as follows. Note that when doing so, other vulnerabilites need to be considered:

```diff
+   import {MartenitsaMarketplace} from "./MartenitsaMarketplace.sol";

+   address marketplaceAddress;

// part of a fix to another vulnerability
+    function setMarketplaceAddress(address _marketplace) external onlyOwner {
+        require(marketplaceAddress == address(0), "Marketplace already set"); // Optional: Ensure only set once
+        marketplaceAddress = _marketplace;
+    }


-    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
+   function updateCountMartenitsaTokensOwner(address owner, string memory operation) public {
          // part of a fix to another vulnerability
+        require(msg.sender == marketplaceAddress || msg.sender == address(this), "Unauthorized call");

        // existing logic
    }

+    function transferFrom(address currentOwner, address newOwner, uint256 tokenId) public override {
        // check that the NFT is not listed (fix for yet another vulnerability)
+      require(!MartenitsaMarketplace(marketplaceAddress).getListing(tokenId).forSale, "This NFT is currently listed on the marketplace.");

+      updateCountMartenitsaTokensOwner(newOwner, "add");
+      updateCountMartenitsaTokensOwner(currentOwner, "sub");
+      super.transferFrom(currentOwner, newOwner, tokenId); // Call the inherited method to handle the transfer
+    }
```
## <a id='M-05'></a>M-05. `MartenitsaVoting::announceWinner` does not check if the winner NFTs is still listed, risk of DoS            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L57-L74

## Summary

`MartenitsaVoting::announceWinner` does not check if the winner NFTs is still listed, and assumes that the owner of the NFT is `MartenitsaMarketplace.getListing(tokenId).seller`. If the winner NFT is not listed when `MartenitsaVoting::announceWinner` is called, then the function will effectively try to mint `healthToken` rewards to `address(0)` and, hence, `MartenitsaVoting::announceWinner` will be impossible to successfully execute.

## Vulnerability Details

Consider the following scenario: 
- `UserA` owns an NFT and lists it for sale on the Martenitsa marketplace;
- once listed, users can vote on it in the Martenitsa voting event;
- imagine this NFT receives the highest number of votes, positioning it as the sure winner of the event;
- in the meantime, another user purchases this NFT from `UserA`, based on the existing listing on the marketplace.
- following the sale, the NFT is automatically delisted
- soon after the voting event ends, `MartentisaVoting::announceWinner` is called,
- `MartentisaVoting::announceWinner` assumes that the owner of the NFT is `MartenitsaMarketplace.getListing(tokenId).seller`, which in the case of a delisted NFT is `address(0)`,
- MartenitsaVoting::announceWinner` will try to mint `healthToken` rewards to `address(0)` and, hence, the transaction will fail.
- it will be impossible to announce a winner until the new owner of the NFT relists it.


## Impact
It will be impossible to announce a winner until the new owner of the NFT relists it.


## Tools Used
Manual review, Foundry.

## Recommendations
Use onwerOf() to send the rewards to the true owner.

## <a id='M-06'></a>M-06. `MartenitsaEvent::stopEvent` loops through an unbounded array, risk of eDoS/DoS            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L62

## Summary

`MartenitsaEvent::stopEvent` loops through the unbounded array `participants`, making this function vulnerable to economic Denial of Service (eDoS) or even regular DoS attacks. 

## Vulnerability Details

`MartenitsaEvent::stopEvent` is designed to reset the producer status (tracked by mapping `isProducer`) of all participants at the end of an event. To perform this reset, the function iterates over the `participants` array that, however, is an unbounded array: there is no limit on the number of participants that can join an event. Consequently, this function is vulnerable to economic Denial of Service (eDoS) or even regular DoS attacks.

(eDoS: Executing the `stopEvent` function would not hit the gas limit, thus avoiding a complete Denial of Service (DoS), but it would consumes an inordinate amount of gas that would makes the transaction economically unfeasible for the protocol owner to perform.) 

The following test simulates a scenario where 40,000 participants join an event, which results in the stopEvent function consuming an amount of gas that would exceed typical block gas limits today (30 million):

<details>
<summary>Proof of Code</summary>

```javascript
    function testDOSdueToUnboundedLoop() public {
        uint256 numParticipants = 40000;
        address[] memory participants2 = new address[](numParticipants);

        // setting trx gas price to 1
        vm.txGasPrice(1);

        // event starts
        martenitsaEvent.startEvent(1 days);

        for (uint256 i = 0; i < numParticipants; i++) {
            address currentParticipant;
            participants2[i] = address(uint160(i + 1)); //skip zero address
            currentParticipant = participants2[i];

            // participants somehow acquire 1 HT each
            // here we deal them one, but they can get one from someone, buy on DEX, get from rewards, etc.
            deal(address(healthToken), currentParticipant, 1 ether);

            // participants enter the event
            vm.startPrank(currentParticipant);
            healthToken.approve(address(martenitsaEvent), 1 ether);
            martenitsaEvent.joinEvent();
            vm.stopPrank();
        }

        // event ends
        vm.warp(block.timestamp + 1 days + 1 seconds);
        uint256 gasStart = gasleft();
        martenitsaEvent.stopEvent();
        uint256 gasEnd = gasleft();

         // gas limit is currently 30 million, which is now exceeded
        console.log("Gas: ", gasStart - gasEnd);
    }
```
</details>

As seen, hitting the gas limit would require about 40,000 participants. While this number might seem big, consider that

- it is not inconceivable, and not without precedent, that a protocol gets so popular that even more people want to join its events;

- participants are economacially incentivized to perform a DoS to retain their producer status forever;

- performing an eDoS requires a lot less participants, but effectively gives the same results.


## Impact

Issues arise even without a DoS or eDoS:

- Increased Transaction Costs: Even without malicious intent, regular usage could lead to high gas costs. The more legitimate users join events, the more the cost of calling `stopEvent` will be.

- Contract Clogging: With the `participants` array not being cleared (another bug), repeated events can lead to unchecked growth in its size, further exacerbating the issue.

A DoS or eDoS would result in:

- Irreversible Temporary Statuses: Participants from the event will indefinitely maintain their producer status. 

- Economic Manipulation: Retaining their producer status allows participants to indefinitely create, list, and sell Martenitsa NFTs. This unlimited production and sale capability can lead to market manipulation and unfair economic advantages, as these users exploit their persistent producer privileges without temporal restrictions.


## Tools Used
Manual review, Foundry.

## Recommendations
Consider implementing one or more of the following changes:

- Limit the number of participants:

```diff
+ uint256 public constant MAX_PARTICIPANTS = 1000;

...

  function joinEvent() external {
+      require(participants.length < MAX_PARTICIPANTS, "Participant limit reached");
      ...
  }
```

You can consider doing this in a more sophisticated manner: e.g. declaring the max number of participants as a non-constant variable, and add a function with which you could adjust its value within reasonable, safe limits. Additionally, if your events get super popular, you can just run more events after each other.

- Clear the `participants` array in `MartenitsaEvent::stopEvent`:

```diff
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");

        for (uint256 i = 0; i < participants.length; i++) {
            isProducer[participants[i]] = false;
+        _participants[participants[i]] = false;  // Reset participant status
        }

+      delete participants;  

    }
```
## <a id='M-07'></a>M-07. `MartenitsaEvent::stopEvent` does not properly reset participant status after an event            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L60-L65

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L44

## Summary
`MartenitsaEvent::stopEvent` does not properly reset participant status after an event concludes, leading to a situation where users who participated in a previous event are unable to join future events. 

## Vulnerability Details

The `MartenitsaEvent` contract manages event participation through an array that includes all participants (`participants` and a mapping that facilitates the lookup of participant status (`_participants`). Within this contract `MartenitsaEvent::stopEvent` is supposed to stop the event and clear all variables that reflect user statuses that are temporary during the event.  Although the `participants` and `_participants` mapping are exactly such variables, `MartenitsaEvent::stopEvent` does not adequately reset/clear them. At the same time, importantly, `MartenitsaEvent::stopEvent` does revoke participants's producer status:

```javascript
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
        for (uint256 i = 0; i < participants.length; i++) {
@>       // Current participant cleanup logic is missing
@>            isProducer[participants[i]] = false;
        }
    }
``` 

The persistent participant status prevents users from re-engaging in subsequent events, effectively barring them from participation after their initial event. This issue arises because the _participants mapping, which tracks whether an address has joined an event, is not cleared after an event concludes. At the same time

This is demonstrated in the following piece of code:

<details>
<summary>Proof of Code</summary>

```javascript
function testParticipantArrayNotUpdatedAfterEventCannotParticipateInMoreEvents() public {
        address user = makeAddr("user");

        // jack gives 3 NFTs to user as a gift
        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.createMartenitsa("bracelet");
        martenitsaToken.approve(address(marketplace), 0);
        martenitsaToken.approve(address(marketplace), 1);
        martenitsaToken.approve(address(marketplace), 2);
        marketplace.makePresent(user, 0);
        marketplace.makePresent(user, 1);
        marketplace.makePresent(user, 2);
        vm.stopPrank();

        // user collects HT reward after 3 NFTs
        vm.prank(user);
        marketplace.collectReward();

        assert(healthToken.balanceOf(user) == 1 ether);

        // protocol owner starts event
        martenitsaEvent.startEvent(1 days);

        // user enters the event as participant
        vm.startPrank(user);
        healthToken.approve(address(martenitsaEvent), 1 ether);
        martenitsaEvent.joinEvent();
        vm.stopPrank();

        // protocol owner ends event
        vm.warp(block.timestamp + 1 days + 1 seconds);
        martenitsaEvent.stopEvent();

        // user is still listed as participant
        assert(martenitsaEvent.getParticipant(user) == true);

        // protocol owner starts new event
        martenitsaEvent.startEvent(1 days);

        // user tries to join new event but cant
        vm.prank(user);
        vm.expectRevert("You have already joined the event");
        martenitsaEvent.joinEvent();
    }
```
</details>

## Impact

- Participant exclusion: Once a user joins an event, they are permanently marked as having participated and cannot join any subsequent events.

- Confusion about statuses: users who have the participant status are supposed to have also the producer status, but after an event `Martenitsa::stopEvent` revokes/clears only the latter, not the former, leading to a status inconsistent with the protocol's intentions.

- Extra computational overhead: the `participants` will keep growing from event to event. The piece of code in `MartenitsaEvent::stopEvent` that loops through this array will become increasingly computationally expensive to run:

```javascript
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
@>        for (uint256 i = 0; i < participants.length; i++) {
            isProducer[participants[i]] = false;
        }
    }
```

## Tools Used
Manual review, Foundry.

## Recommendations

Ensure that the `_participants` mapping and the `participants` array are cleared in `MartenitsaEvent::stopEvent`:

```diff
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");

        for (uint256 i = 0; i < participants.length; i++) {
            isProducer[participants[i]] = false;
+        _participants[participants[i]] = false;  // Reset participant status
        }

+      delete participants;  

    }
```



# Low Risk Findings

## <a id='L-01'></a>L-01. `MartenitsaMarketplace::makePresent` allows the transfer of listed NFTs (without clearing the listing)            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L60

## Summary
`MartenitsaMarketplace::makePresent` allows users to transfer their NFTs to another user, but it fails to ensure that either
- listed NFTs cannot be given away as a present, or
- if a listed NFT is given away as a present, the listing is cleared.

## Vulnerability Details
`MartenitsaMarketplace::makePresent` allows users to give away their NFTs to another user for free, as a present. Importantly, however, listed NFTs are not supposed to be able to be transferred between users unless bought. `MartenitsaMarketplace::makePresent` does not check whether an NFT is listed and, accordingly, a user can give away its listed NFT as a present to another user.

Consider the following test that demonstrates this vulnerability:

<details>
<summary>Proof of Code</summary>

```javascript
    function testListedNftCanBeGivenAwayAsPresent() public {
        address user = makeAddr("user");
        uint256 price = 1e18;

        vm.startPrank(jack);
        martenitsaToken.createMartenitsa("bracelet");
        uint256 tokenId = 0;
        assert(martenitsaToken.ownerOf(tokenId) == jack);
        assert(martenitsaToken.isProducer(jack) == true);
        marketplace.listMartenitsaForSale(tokenId, price);
        assert(marketplace.getListing(tokenId).forSale == true);
        martenitsaToken.approve(address(marketplace), tokenId);
        marketplace.makePresent(user, tokenId);
        vm.stopPrank();

        // listed nft changed owners
        assert(martenitsaToken.ownerOf(tokenId) == user);
        assert(martenitsaToken.balanceOf(jack) == 0);

        // nft is still listed
        assert(marketplace.getListing(tokenId).forSale == true);
    }
```
</details>

## Impact

If a listed NFT is given away to another user via `MartenitsaMarketplace::makePresent`, the NFT will remian incorrectly listed even after the ownership change. As a consequence:

-  `mapping(uint256 => Listing) public tokenIdToListing;` will be incorrect for the affected NFT,
- `MartenitsaMarketplace::getListing` will return the incorrect listing data for the affected NFT,
- although the affected NFT will continue to appear to be listed, it cannot be bought (as `safeTransferFrom` in `MartenitsaMarketplace::buyMartenitsa` will fail since it will try to move the NFT from the previous owner who made the present). Unsuspecting buyers will waste gas trying to buy the NFT. 

(Note that all these issues will become void if the new owner properly lists the NFT it got as a present, since previous listing details will be overriden and reflect reality and true ownership status once again.)


## Tools Used
Manual review, Foundry.

## Recommendations
Consider the following options and select whichever matches your current and future requirements more appropriately:

1. Implement a check in `MartenitsaMarketplace::makePresent` to ensure that listed tokens cannot be given away:

```diff
    function makePresent(address presentReceiver, uint256 tokenId) external {
        require(msg.sender == martenitsaToken.ownerOf(tokenId), "You do not own this token");
+      require(!getListing(tokenId).forSale, "Token is currently listed on the marketplace, you cannot give it away");
        martenitsaToken.updateCountMartenitsaTokensOwner(presentReceiver, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(msg.sender, "sub");
        martenitsaToken.safeTransferFrom(msg.sender, presentReceiver, tokenId);
    }
```

Note that additionally, you should overwrite the standard `ERC721` method `transferFrom` in the `MartenitsaToken` contract  and include a similar check therein.

2. Implement a logic that automatically cancels the listing when someone tries to give away thier listed NFTs as present:

```diff

+    event ListingDeleted(uint256 indexed tokenId);

    function makePresent(address presentReceiver, uint256 tokenId) external {
        require(msg.sender == martenitsaToken.ownerOf(tokenId), "You do not own this token");
+       if(getListing(tokenId).forSale) {    
+           delete tokenIdToListing[tokenId];
+           emit ListingDeleted(tokenId);  
+       }
        martenitsaToken.updateCountMartenitsaTokensOwner(presentReceiver, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(msg.sender, "sub");
        martenitsaToken.safeTransferFrom(msg.sender, presentReceiver, tokenId);
    }

```


