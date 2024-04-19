# First Flight #12: Kitty Connect - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Ownership information is not completely updated during NFT transfers via `KittyConnect::safeTransferFrom``](#H-01)
    - ### [H-02. Missing `LINK` token approval in `KittyBridge::bridgeNftWithData`](#H-02)
    - ### [H-03. Missing access control in `KittyBridge::bridgeNftWithData`](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Users can bypass `KittyConnect:.safeTransferFrom` and transfer Kitty NFTs themselves](#M-01)
    - ### [M-02. `KittyConnect` functions `tokenURI`, `getCatAge`, `getCatInfo` do not check whether `tokenId` exists](#M-02)
- ## Low Risk Findings
    - ### [L-01. `KittyConnect::safeTransferFrom` does not check whether the proposed new owner is a partner shop](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Ownership information is not completely updated during NFT transfers via `KittyConnect::safeTransferFrom``            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L181-L185

## Summary
When a token is transferred, it is not removed from the original owner's list of tokens (`s_ownerToCatsTokenId[currCatOwner]`). Consequently, the internal ownership accounting will be incorrect.

## Vulnerability Details
When a token is transferred via `KittyConnect::safeTransferFrom`, ownership info is supposed to be updated via a call to `KittyConnect::_updateOwnershipInfo`:

```javascript
    function _updateOwnershipInfo(address currCatOwner, address newOwner, uint256 tokenId) internal {
        s_catInfo[tokenId].prevOwner.push(currCatOwner);
        s_catInfo[tokenId].idx = s_ownerToCatsTokenId[newOwner].length;
        s_ownerToCatsTokenId[newOwner].push(tokenId);
    }
```
However, `KittyConnect::_updateOwnershipInfo` misses to remove the transferred token from the the original owner's list of tokens `s_ownerToCatsTokenId[currCatOwner]`.

<details>
<summary>Proof of Code</summary>

```javascript
   function test_ownershipInfoNotFullyUpdatedDuringTransfer() public {
        address newOwner = makeAddr("newOwner");
        uint256 tokenId = 0;

        string memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(user, catImageIpfsHash, "Meowdy", "Ragdoll", block.timestamp);

        vm.prank(user);
        kittyConnect.approve(newOwner, tokenId);

        vm.prank(partnerA);
        kittyConnect.safeTransferFrom(user, newOwner, tokenId);

        uint256 tokenIdOwnedByUser = kittyConnect.getCatsTokenIdOwnedBy(user)[tokenId]; // this does not aligns with real ownership
        uint256 tokenIdOwnedByNewOwner = kittyConnect.getCatsTokenIdOwnedBy(newOwner)[tokenId];

        console.log(tokenIdOwnedByUser);
        console.log(tokenIdOwnedByNewOwner);

        assert(tokenIdOwnedByNewOwner == tokenIdOwnedByUser);
        assert(kittyConnect.ownerOf(0) == newOwner);
    }
```
</details>


## Impact
Onwership info (internal ownership) accounting will be incorrect and unreliable. 

Note that this data can become even more entangled and cause more problems.  
The `idx` within the `CatInfo` structure is used as an index to track the position of each NFT within an owner's array of token IDs (`s_ownerToCatsTokenId[owner]`). This design aims to facilitate efficient management and lookup of NFTs owned by a particular user, especially for operations that involve modifying the ownership or characteristics of these NFTs.
`idx` is relied on in `KittyConnect::bridgeNftToAnotherChain` as follows:

```javascript
   function bridgeNftToAnotherChain(uint64 destChainSelector, address destChainBridge, uint256 tokenId) external {
        ...

        uint256[] memory userTokenIds = s_ownerToCatsTokenId[msg.sender];
        uint256 lastItem = userTokenIds[userTokenIds.length - 1];

        s_ownerToCatsTokenId[msg.sender].pop(); // e last one is removed

        if (idx < (userTokenIds.length - 1)) {
            s_ownerToCatsTokenId[msg.sender][idx] = lastItem;
        }
        ...
    }
```
However, after calling `KittyConnect::safeTransferFrom`, `idx` will be incorrect and unreliable. Consider the following scenario:

<details>
<summary> Detailed Scenario </summary>

1.  `mintCatToNewOwner()` is called to mint an NFT to `userA`. For this, `Idx=0`, `tokenId=0`. 
2. `safeTransferFrom()` is called to transfer `tokenId=0` to `userB`.
3. `mintCatToNewOwner()` is called to mint a 2nd NFT to `userA`. For this, `Idx=1`, `tokenId=1` (due to the bug).
4. `mintCatToNewOwner()` is called to mint a 3rd NFT to `userA`. For this, `Idx=2`, `tokenId=2` (due to the bug).
5. `bridgeNftToAnotherChain()` is called by `userA` to bridge `tokenId=1` to another chain. 

As a result, the `s_ownerToCatsTokenId[userA]` array will incorrectly be `[0, 1]`, while it really should be `[2]`.

<details>
<summary> Even more detailed walk-through </summary>

- First Mint: userA receives an NFT (tokenId=0). The idx within CatInfo for this token is 0. The array s_ownerToCatsTokenId[userA] now contains [0].
- Transfer to userB: tokenId=0 is transferred to userB. Due to the bug, s_ownerToCatsTokenId[userA] mistakenly still contains [0], although it should be empty.
- Second Mint: Another NFT is minted to userA (tokenId=1). Given the bug, idx is incorrectly set to 1 for this new token because s_ownerToCatsTokenId[userA] was not updated properly earlier and still shows one token. After minting, s_ownerToCatsTokenId[userA] incorrectly shows [0, 1].
- Third Mint: A third NFT (tokenId=2) is minted to userA. The idx for this token is set to 2, following the same incorrect pattern. Now, s_ownerToCatsTokenId[userA] is [0, 1, 2], which is incorrect because tokenId=0 should not be there.
- Bridging tokenId=1: When bridgeNftToAnotherChain() is called for tokenId=1 by userA, the function attempts to remove tokenId=1 from userA's list. Here's what happens, step by step::

delete s_catInfo[tokenId]; removes the CatInfo for tokenId=1.
It then attempts to remove tokenId=1 from userA's list of token IDs by popping the last item and replacing tokenId=1's position if it's not the last item.
Given the array state [0, 1, 2]:

The last item (2) is stored in lastItem.
The last item is popped, so before any replacement, the array temporarily looks like [0, 1].
Since idx for tokenId=1 is 1, and it's not less than the current last index (1), no swap occurs. 

Values left in s_ownerToCatsTokenId[userA]:
The array will be [0, 1] immediately after popping 2, and since the removal logic does not correctly address the middle of the array without additional steps, it incorrectly remains [0, 1] (assuming no swap due to incorrect indexing logic). The intended outcome after correctly removing tokenId=1 should have been just [0] considering the original bug, but with proper management, it should actually end up being [2] after the transfer bug is fixed and transfers are properly managed.
</details>
</details>


## Tools Used
Manual review, Foundry.

## Recommendations
Modify the `_updateOwnershipInfo` function to include the removal of thet`tokenId` from the `currCatOwner`'s list. 

```diff
    function _updateOwnershipInfo(address currCatOwner, address newOwner, uint256 tokenId) internal {
+       _removeTokenFromOwnerTokens(currCatOwner, tokenId);
        s_catInfo[tokenId].prevOwner.push(currCatOwner);
        s_catInfo[tokenId].idx = s_ownerToCatsTokenId[newOwner].length;
        s_ownerToCatsTokenId[newOwner].push(tokenId);
    }

+ function _removeTokenFromOwnerTokens(address owner, uint256 tokenId) internal {
+    uint256 lastTokenIndex = s_ownerToCatsTokenId[owner].length - 1;
+    uint256 tokenIndex = s_catInfo[tokenId].idx; // Assumes idx is accurate up to this point
+
+    // If the token to remove isn't the last one, swap it with the last one
+    if (tokenIndex != lastTokenIndex) {
+        uint256 lastTokenId = s_ownerToCatsTokenId[owner][lastTokenIndex];
+        s_ownerToCatsTokenId[owner][tokenIndex] = lastTokenId;
+        s_catInfo[lastTokenId].idx = tokenIndex; // Update the moved token's idx
+    }

+    // Remove the last token (now either the one we're removing, or swapped)
+    s_ownerToCatsTokenId[owner].pop();
+}

```
## <a id='H-02'></a>H-02. Missing `LINK` token approval in `KittyBridge::bridgeNftWithData`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L68-L72

## Summary
`KittyBridge::bridgeNftWithData` should approve the `router` to transfer `LINK` tokens on contract's behalf, but this approval is missing. Consequently, the 

## Vulnerability Details
In the second step of the bridging process, `KittyBridge::bridgeNftWithData` is supposed to use Chainlink's Cross-Chain Interoperability Protocol (CCIP) to send the encoded NFT data to the destination chain. Sending a cross-chain message via CCIP incurs an execution fee for the message, which is properly calculated here
```javascript
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);
```
and there is even a conditional to check that the `KittyBridge` contract does have enough balance to cover said fees:
```javascript
        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(s_linkToken.balanceOf(address(this)), fees);
        }
```

Before actually sending the message via
```javascript
        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);
```

In order to pay the fees, `KittyBridge` is supposed to approve the `router` to transfer `LINK` tokens on behalf of `KittyBridge`. However, this approval is missing. 

## Impact
The `router` will not be able to collect the execution fee from `KittyBridge`, and the bridging process will revert with `"ERC20: insufficient allowance"`.

## Tools Used
Manual review, Foundry.

## Recommendations
Approve the `router` to transfer `LINK` tokens on behalf of `KittyBridge` as follows:

```diff
    function bridgeNftWithData(uint64 _destinationChainSelector, address _receiver, bytes memory _data)
        external
        onlyAllowlistedDestinationChain(_destinationChainSelector)
        validateReceiver(_receiver)
        returns (bytes32 messageId)
    {
        // Create an EVM2AnyMessage struct in memory with necessary information for sending a cross-chain message
        Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(_receiver, _data, address(s_linkToken));

        // Initialize a router client instance to interact with cross-chain router
        IRouterClient router = IRouterClient(this.getRouter());

        // Get the fee required to send the CCIP message
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(s_linkToken.balanceOf(address(this)), fees);
        }

+      s_linkToken.approve(address(router), fees);
        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(messageId, _destinationChainSelector, _receiver, _data, address(s_linkToken), fees);

        return messageId;
    }
```
## <a id='H-03'></a>H-03. Missing access control in `KittyBridge::bridgeNftWithData`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyBridge.sol#L53C1-L57C36

## Summary
`KittyBridge::bridgeNftWithData` does not have adequate access control, and, consequently, anyone at any time can call this function.

## Vulnerability Details
`KittyBridge::bridgeNftWithData` is supposed to send the encoded NFT data for bridging an NFT from one chain to another. As such, it is supposed to be called only during the bridging process, and only from within `KittyConnect::bridgeNftToAnotherChain`. However, `bridgeNftWithData()` lacks the access control neccessary to enforce this and, as a result, anyone at anytime can call this function.

## Impact
Anyone, at any time, can call `KittyBridge::bridgeNftWithData` with an arbitrary (and arbitrarily large) payload. Since sending a message via
```javascript
        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);
```
entails paying execution fees for the message to Chainlink, an attacker could drain the LINK balance of the `KittyBridge` contract (that is, provided that another bug is fixed before this one, and `KittyBridge` approves the `router` to spend its LINK). 

(The impact is contained at this level, since the next step of the bridging process is receipt on the destination chain, and fortunately KittyBridge::_ccipReceive` accepts messages only from allowlisted senders.)

## Tools Used
Manual review, Foundry.

## Recommendations
Add access control to `KittyBridge::bridgeNftWithData`:

```diff
function bridgeNftWithData(uint64 _destinationChainSelector, address _receiver, bytes memory _data)
        external
+      onlyKittyConnect
        onlyAllowlistedDestinationChain(_destinationChainSelector)
        validateReceiver(_receiver)
        returns (bytes32 messageId) 
{
        ...
}
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can bypass `KittyConnect:.safeTransferFrom` and transfer Kitty NFTs themselves            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L5

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L118

## Summary
Using `KittyConnect::transferFrom` (a function `KittyConnect` inherits from `ERC721.sol), users can bypass `KittyConnect:.safeTransferFrom` and transfer Kitty NFTs themselves.

## Vulnerability Details
Transfer of Kitty NFTs is supposed to require the facilitation of the partner shops, users are not supposed to be able to transfer Kitty NFTs themselves. This is signified by the `onylShopOwner` modifier in `KittyConnect::safeTransferFrom`:

```javascript
    function safeTransferFrom(address currCatOwner, address newOwner, uint256 tokenId, bytes memory data) public override onlyShopPartner {
    ...
``` 
However, users can bypass `KittyConnect:safeTransferFrom` and transfer Kitty NFTs themselves if they call `KittyConnect::transferFrom` (a function `KittyConnect` inherits from `ERC721.sol).

<details>
<summary>Proof of Code</summary>

```javascript
    function test_usersCanTransferNFTsThemselves() public {
        address newOwner = makeAddr("newOwner");
        uint256 tokenId = 0;
        string memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";

        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(user, catImageIpfsHash, "Meowdy", "Ragdoll", block.timestamp);

        assert(kittyConnect.ownerOf(tokenId) == user);

        vm.prank(user);
        kittyConnect.approve(newOwner, tokenId);

        vm.prank(user);
        kittyConnect.transferFrom(user, newOwner, tokenId);

        // true ownership transferred to newOwner
        assert(kittyConnect.ownerOf(tokenId) == newOwner);

        // internal ownership accounting is not updated
        assertEq(kittyConnect.getCatsTokenIdOwnedBy(user).length, 1);
        assertEq(kittyConnect.getCatsTokenIdOwnedBy(newOwner).length, 0);
        assertEq(kittyConnect.getCatsTokenIdOwnedBy(user)[0], tokenId);
        assertEq(kittyConnect.getCatInfo(tokenId).prevOwner.length, 0);
    }
```
</details>

## Impact
Internal ownership accounting in `KittyConnect` will be messed up and not reflect true ownership status. For those NFTs that are transferred via `KittyConnect::transferFrom`, the following variables will have incorrect values:
- `s_ownerToCatsTokenId`
- `s_catInfo`.

Note that this data can become even more entangled and cause more problems.  
The `idx` within the `CatInfo` structure is used as an index to track the position of each NFT within an owner's array of token IDs (`s_ownerToCatsTokenId[owner]`). This design aims to facilitate efficient management and lookup of NFTs owned by a particular user, especially for operations that involve modifying the ownership or characteristics of these NFTs.
`idx` is relied on in `KittyConnect::bridgeNftToAnotherChain` as follows:

```javascript
   function bridgeNftToAnotherChain(uint64 destChainSelector, address destChainBridge, uint256 tokenId) external {
        ...

        uint256[] memory userTokenIds = s_ownerToCatsTokenId[msg.sender];
        uint256 lastItem = userTokenIds[userTokenIds.length - 1];

        s_ownerToCatsTokenId[msg.sender].pop(); // e last one is removed

        if (idx < (userTokenIds.length - 1)) {
            s_ownerToCatsTokenId[msg.sender][idx] = lastItem;
        }
        ...
    }
```
However, after calling `KittyConnect::safeTransferFrom`, `idx` will be incorrect and unreliable.

## Tools Used
Manual review, Foundry.

## Recommendations
Enforce that transfers can be made only via `KittyConnect:safeTransferFrom` by overwriting `ERC721::transferFrom` in `KittyConnect` as follows:

```diff
+    function transferFrom(address currCatOwner, address newOwner, uint256 tokenId) public override {
+        safeTransferFrom(currCatOwner, newOwner, tokenId);
+    }
```

## <a id='M-02'></a>M-02. `KittyConnect` functions `tokenURI`, `getCatAge`, `getCatInfo` do not check whether `tokenId` exists            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L196

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L217

https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L241

## Summary
`KittyConnect` functions `tokenURI`, `getCatAge`, `getCatInfo` do not check whether `tokenId` exists, they accept non-existent `tokenId`s as input.

## Impact
When providing non-existent `tokenId`s to these 3 functions, they will run and return values as if the `tokenId` existed, leading to confusion.

<details>
<summary>Proof of Code</summary>

```javascript
    function test_tokenUriDoesNotCheckIfTokenIdExists() public {
        uint256 nonExistentTokenId = 1000;

        vm.expectRevert("ERC721: invalid token ID");
        address owner = kittyConnect.ownerOf(nonExistentTokenId);
        //require(kittyConnect._exists(nonExistentTokenId) == false);   // this function is not exposed externally
        string memory tokenUri = kittyConnect.tokenURI(nonExistentTokenId);
        console.log(tokenUri);
    }

    function test_getCatAgeDoesNotCheckIfTokenExists() public {
        uint256 nonExistentTokenId = 1000;

        vm.warp(block.timestamp + 10 weeks);
        uint256 catAge = kittyConnect.getCatAge(nonExistentTokenId);
        console.log("Catage: ", catAge);
        assert(catAge > 0);
    }

    function test_getCatInfoDoesNotCheckIfTokenExists() public view {
        uint256 nonExistentTokenId = 1000;
        KittyConnect.CatInfo memory catInfo = kittyConnect.getCatInfo(nonExistentTokenId);
        console.log("CatName: ", catInfo.catName);
        console.log("Breed: ", catInfo.breed);
        console.log("Image: ", catInfo.image);
        console.log("DoB: ", catInfo.dob);
        //console.log("PrevOwner: ", catInfo.prevOwner[0]); // throws error [FAIL. Reason: panic: array out-of-bounds access (0x32)]
        console.log("ShopPartner: ", catInfo.shopPartner);
        console.log("Idx: ", catInfo.idx);
    }
```
</details>

## Tools Used
Manual review, Foundry.

## Recommendations
In all these functions, check whether the `tokenId` provided as input actually exist:

```diff
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
+        require(_ownerOf(tokenId) != address(0), "TokenID does not exist.");
    ...
    }

    function getCatAge(uint256 tokenId) external view returns (uint256) {
+        require(_ownerOf(tokenId) != address(0), "TokenID does not exist.");
          return block.timestamp - s_catInfo[tokenId].dob;
    }

    function getCatInfo(uint256 tokenId) external view returns (CatInfo memory) {
+        require(_ownerOf(tokenId) != address(0), "TokenID does not exist.");
          return s_catInfo[tokenId];
    }

```

# Low Risk Findings

## <a id='L-01'></a>L-01. `KittyConnect::safeTransferFrom` does not check whether the proposed new owner is a partner shop            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/c0a6f2bb5c853d7a470eb684e1954dba261fb167/src/KittyConnect.sol#L118

## Summary
`KittyConnect::safeTransferFrom` does not check whether the proposed new owner is a partner shop or not.

## Vulnerability Details
Partner shops are NOT supposed to own Kitty NFTs. This is signified by the `require` statement and custom error in `KittyConnect::mintCatToNewOwner`:

```javascript
        require(!s_isKittyShop[catOwner], "KittyConnect__CatOwnerCantBeShopPartner");
```

However, the restriction is not present in `KittyConnect::safeTransferFrom`, so partner shops can receive NFTs via calling this function.

<details>
<summary> Proof of Code </summary>

```javascript
    function test_NftCanBeTransferredToShop() public {
        string memory catImageIpfsHash = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";
        vm.prank(partnerA); // PartnerA mints a new kitty to User
        kittyConnect.mintCatToNewOwner(user, catImageIpfsHash, "Meowdy", "Ragdoll", block.timestamp);

        uint256 tokenId = 0;

        vm.prank(user);
        kittyConnect.approve(partnerB, tokenId);

        vm.prank(partnerA);
        kittyConnect.safeTransferFrom(user, partnerB, tokenId, "");

        assert(kittyConnect.ownerOf(0) == partnerB);
    }
```

</details>

## Impact
Parner shops can receive NFTs while they are not supposed to own any.

## Tools Used
Manual review, Foundry.

## Recommendations
To prevent shop partners from receiving kitty NFTs, add an additional `require` statement to `KittyConnect::safeTransferFrom` as follows:

```diff
    function safeTransferFrom(address currCatOwner, address newOwner, uint256 tokenId, bytes memory data)
        public
        override
        onlyShopPartner
    {
        require(_ownerOf(tokenId) == currCatOwner, "KittyConnect__NotKittyOwner");
+      require(!s_isKittyShop[newOwner], "KittyConnect__CatOwnerCantBeShopPartner");

        require(getApproved(tokenId) == newOwner, "KittyConnect__NewOwnerNotApproved");

        _updateOwnershipInfo(currCatOwner, newOwner, tokenId); // idx updated here

        emit CatTransferredToNewOwner(currCatOwner, newOwner, tokenId);
        _safeTransfer(currCatOwner, newOwner, tokenId, data);
    }
```


