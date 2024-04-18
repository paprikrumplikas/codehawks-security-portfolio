# First Flight #11: Snek-Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `snek_raffle::fulfillRandomWords` Does Not Check the Success of the Prize Transfer, Allowing the Raffle to Conclude without the Winner Receiving the Prize Money](#H-01)
- ## Medium Risk Findings
    - ### [M-01. `snek_raffle::fulfillRandomWords` Distributes Common/Rare/Legend Rarities Equally, Causing a Mismatch with Intended Probabilities](#M-01)
    - ### [M-02. `snek_raffle::fulfillRandomWords` Does Not Check Whether the Winner is Capable of Interacting with ERC-721 Tokens, Tokens can Get Stuck If Recipient is Unprepared](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #11

### Dates: Mar 7th, 2024 - Mar 14th, 2024

[See more contest details here](https://www.codehawks.com/contests/cltd8lz860001y058i5ug72ko)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 2
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. `snek_raffle::fulfillRandomWords` Does Not Check the Success of the Prize Transfer, Allowing the Raffle to Conclude without the Winner Receiving the Prize Money            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-snek-raffle/blob/6b12b719f5d8587f9bbbf3b029954f23a5a38704/contracts/snek_raffle.vy#L158

## Summary

`snek_raffle::fulfillRandomWords` does not check the success of the money prize transfer. If the transfer does fail, the raffle can still conclude without the winner ever receiving the prize money.

## Vulnerability Details

`snek_raffle::fulfillRandomWords` is supposed to select the winner of a raffle, and then transfer the raffle prices (an NFT and the prize money, i.e. the sum of entrance fees). However, it does not check the success of the transfer of the prize money. The transfer can fail for multiple reasons, including

-  the recipient's inability to receive funds due to gas constraints,
- contract code execution failure.

Given that the success of the transfer is not checked, if the transfer does fail, the raffle can still conclude.

## Proof of Code

To simulate the transfer failure, this contract will intentionally fail when ether is sent to it:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract FailingRecipient {
    // Fallback function that fails
    receive() external payable {
        revert("Cannot receive ETH");
    }
}

```


## Impact

If the transfer fails, the raffle can still conclude without the winner receiving the prize money. In such a case, the prize money remains in the raffle contract, and will be given to the winner of the next raffle, provided that the transfer transaction to that winner is successful.

## Tools Used
ChatGPT.

## Recommendations

There are multiple potential solutions to this issue:

1. Use pull over push: implement a withdrawal pattern where winner withdraw their prizes themselves, or
2. Check the success of the ether transfer and revert the transaction if the transfer fails.

To implement the second option, change the code as follows:

```def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    ...
-   send(recent_winner, self.balance)
+   if not send(recent_winner, self.balance):
+       revert("Transfer failed")
    ...
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `snek_raffle::fulfillRandomWords` Distributes Common/Rare/Legend Rarities Equally, Causing a Mismatch with Intended Probabilities            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-snek-raffle/blob/6b12b719f5d8587f9bbbf3b029954f23a5a38704/contracts/snek_raffle.vy#L154

## Summary

`snek_raffle::fulfillRandomWords` allocates NFT rarities incorrectly. Instead of distributing rarities according to the specified chances (70% Common, 25% Rare, 5% Legendary), the contract equally distributes rarities.

## Vulnerability Details

snek_raffle::fulfillRandomWords is the function that - among other things - is supposed to allocate NFT rarites according to the specified probabilites (70% Common, 25% Rare, 5% Legendary). However, in the code rarity is assigned as `random_words[0] % 3`, which incorrectly maps the random number directly to one of the three rarities without considering the specified probabilities.

## Proof of Code

The following test focus analyzes the distribution of rarities after a significant number of entries have been processed, to statistically demonstrate that the distribution does not align with the intended 70% Common, 25% Rare, and 5% Legendary probabilities.

```
import boa
import pytest
from collections import defaultdict

# Assuming these are defined in your smart contract or test setup
COMMON = 0
RARE = 1
LEGEND = 2
NUM_SIMULATIONS = 1000  # Number of times to simulate raffle entry for statistical significance

@pytest.fixture
def setup_raffle_with_entries(raffle_boa_entered, vrf_coordinator_boa, entrance_fee):
    # Simulate multiple entries to the raffle
    for _ in range(NUM_SIMULATIONS):
        player = boa.env.generate_address("additional_player")
        boa.env.set_balance(player, STARTING_BALANCE)
        with boa.env.prank(player):
            raffle_boa_entered.enter_raffle(value=entrance_fee)

    # Advance time to allow for raffle winner request
    boa.env.time_travel(seconds=INTERVAL + 1)
    raffle_boa_entered.request_raffle_winner()

    # Mock fulfillment from VRF Coordinator (skipping requestId handling for simplicity)
    # This part should ideally be looped to simulate multiple raffle cycles
    # However, for a single cycle, this demonstrates the principle
    vrf_coordinator_boa.fulfillRandomWords(0, raffle_boa_entered.address)

    return raffle_boa_entered

def test_rarity_distribution_accuracy(setup_raffle_with_entries):
    raffle_boa = setup_raffle_with_entries
    rarity_counts = defaultdict(int)

    # Assuming a function exists to fetch the rarity of each NFT by index,
    # and a total_supply function exists to know how many NFTs to loop through.
    # These functionalities depend on the implementation details of your smart contract.
    for token_id in range(1, raffle_boa.total_supply() + 1):  # Assuming token IDs start at 1
        rarity = raffle_boa.tokenIdToRarity(token_id)
        rarity_counts[rarity] += 1

    total_nfts = raffle_boa.total_supply()
    common_percentage = (rarity_counts[COMMON] / total_nfts) * 100
    rare_percentage = (rarity_counts[RARE] / total_nfts) * 100
    legend_percentage = (rarity_counts[LEGEND] / total_nfts) * 100

    # Asserting the distribution matches expected probabilities
    # Note: These assertions may need a margin of error due to the statistical nature of probability
    assert common_percentage >= 65 and common_percentage <= 75, f"Common rarity distribution off: {common_percentage}%"
    assert rare_percentage >= 20 and rare_percentage <= 30, f"Rare rarity distribution off: {rare_percentage}%"
    assert legend_percentage >= 4 and legend_percentage <= 6, f"Legend rarity distribution off: {legend_percentage}%"
```

## Impact

"Common", "Rare", and "Legend" designations will be meaningless. In a larger collection, 33.33..% of the items will be "Common", 33.33..% "Rare", and 33.33..% "Legend".

## Tools Used
ChatGPT

## Recommendations

Update the rarity assignment logic to reflect the intended probabilities. Use ranges derived from the random number to allocate rarities according to the specified distribution probabilities:

```
def fulfillRandomWords(request_id: uint256, random_words: uint256[MAX_ARRAY_SIZE]):
    ...
-    rarity: uint256 = random_words[0] % 3
+    rarity_score: uint256 = random_words[0] % 100  # Map the random number to a 0-99 range
+    if rarity_score < 70:
+        rarity = COMMON  # 70% chance
+    elif rarity_score < 95:
+        rarity = RARE  # 25% chance
+    else:
+        rarity = LEGEND  # 5% chance
    self.tokenIdToRarity[ERC721._total_supply()] = rarity
    ...
```

## <a id='M-02'></a>M-02. `snek_raffle::fulfillRandomWords` Does Not Check Whether the Winner is Capable of Interacting with ERC-721 Tokens, Tokens can Get Stuck If Recipient is Unprepared            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-snek-raffle/blob/6b12b719f5d8587f9bbbf3b029954f23a5a38704/contracts/snek_raffle.vy#L157C1-L158C1

## Summary

`snek_raffle::fulfillRandomWords` mints and awards ERC-721 NFTs to raffle winners without verifying whether the recipient address (potentially a smart contract) is equipped to interact with ERC-721 tokens. This oversight can result in NFTs being permanently locked within contracts that lack the necessary interface to manage or transfer these tokens.

## Proof of Code

The following smart contract acts as an unprepared recipient for the ERC-721 token: it lacks the `onERC721Received` function, making it unable to acknowledge or interact with received ERC-721 tokens properly.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract UnpreparedRecipient {
    // Intentionally missing implementation of ERC721Receiver

    function tryTransferToken(address nftContract, uint256 tokenId) external {
        // Attempt to transfer the token, expecting this to fail
        // because this contract cannot handle ERC-721 tokens.
    }
}
```

and then use this test to demonstrate that this unprepared contract

```
from brownie import ERC721, UnpreparedRecipient, accounts, reverts

def test_minting_to_unprepared_contract():
    # Setup: Deploy an ERC721 contract and UnpreparedRecipient contract
    erc721 = ERC721.deploy("TestNFT", "TNFT", {'from': accounts[0]})
    unprepared = UnpreparedRecipient.deploy({'from': accounts[1]})

    # Act: Mint an NFT directly to the unprepared contract using mint
    token_id = 1
    erc721.mint(unprepared.address, token_id, {'from': accounts[0]})

    # Assertion: Try to transfer the NFT from the unprepared contract should fail
    # Since the unprepared contract lacks the means to interact with or transfer the NFT,
    # this action is expected to revert. The precise revert message or behavior
    # can depend on the ERC-721 implementation and the setup of the unprepared contract.
    # The following is a conceptual assertion expecting a revert due to the lack of transfer capability.
    with reverts():
        erc721.transferFrom(unprepared.address, accounts[2], token_id, {'from': unprepared.address})

```



## Impact

NFTs can be permanently locked within winner contracts that lack the necessary interface to manage or transfer ERC-721 tokens.

## Tools Used
Brownie, ChatGPT

## Recommendations

Use `safeMint` instead of `mint`:

```
-    # ERC721.safe_mint, not using this 
+   ERC721.safe_mint

function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) external {
    ...;

-    ERC721._mint(recent_winner, ERC721._total_supply())
+   ERC721_safeMint(recent_winner, ERC721._total_supply());

    // Continue with any additional logic following the minting process
    ...
}

```




