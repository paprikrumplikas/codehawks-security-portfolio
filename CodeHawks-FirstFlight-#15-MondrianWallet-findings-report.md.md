# First Flight #15: Mondrian Wallet - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `MondrianWallet::tokenURI` has incorrect logic for assigning `tokenURI`s to `tokenId`s](#H-01)
    - ### [H-02. Missing signature validation in `MondrianWallet::_validateSignature`](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Missing NFT distribution for Mondrian Wallets](#M-01)
    - ### [M-02. Accounts using non-standard signing methods won't work with `MondrianWallet`](#M-02)
- ## Low Risk Findings
    - ### [L-01. ZkSync Era does not have native support for transferring Ether](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #15

### Dates: May 9th, 2024 - May 16th, 2024

[See more contest details here](https://www.codehawks.com/contests/clvxt8idd00014zcc81dv6rde)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. `MondrianWallet::tokenURI` has incorrect logic for assigning `tokenURI`s to `tokenId`s            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L161-L175

## Summary
`MondrianWallet::tokenURI` has incorrect logic for assigning `tokenURI`s to `tokenId`s:
- the 4 `tokenURI`s are not randomly assigned to `tokenId`s, and
- the assignment does not result in uniform distribution of the 4 `tokenURI`s.

## Vulnerability Details

`MondrianWallet::tokenURI` is supposed to randomly assign 1 of the 4 `tokenURI`s to each `tokenId`. 
However, the current logic results in
- a deterministic assignment of `tokenURI`s to `tokenId`s, and
- a non-uniform distribution of the 4 `tokenURI`s.
```javascript
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        if (ownerOf(tokenId) == address(0)) {
            revert MondrainWallet__InvalidTokenId();
        }
        uint256 modNumber = tokenId % 10;
        if (modNumber == 0) {
            return ART_ONE;
        } else if (modNumber == 1) {
            return ART_TWO;
        } else if (modNumber == 2) {
            return ART_THREE;
        } else {
            return ART_FOUR;
        }
    }
```

Importantly, not only the `tokenURI` distribution/assignment is incorrect, but the `tokenId` distribution is also completely missing (i.e. the mint and distribution of the NFTs is not implemented, as described in a separate findings report). These 2 bugs are intertwined (and also are kind of worsened by the design). 


## Impact

`tokenURI` distribution will not be as desired. 

Since the `tokenId` distribution is also missing from the codebase (NFTs are not minted at all), the specific details of the impact cannot be accessed. There are three possible scenarios based on how NFT minting is supposed to be handled:

1. SCENARIO 1:
`MondrianWallet` handles the NFT functionality (as currently is the case, as it inherits `ERC721`), and minting is starting with a predefined `tokenID` (it is the common practice with NFT contracts), e.g. `tokenID=0`. In this case, since every instance of `MondrianWallet` is a separate NFT contract, the `tokenID`s of the NFTs minted by the different instance will be the same, i.e. `tokenID=0`. Consequently, each Mondrian Wallet will receive an NFT with the same `tokenURI`, `ART_ONE`.

2. SCENARIO 2: `MondrianWallet` handles the NFT functionality (as currently is the case, as it inherits `ERC721`), but minting is starting with a _random_ `tokenID`. In this case, the distribution share of the 4 different `tokenURI`s will be the same as in the 3rd scenario: check the point below for details.

3. SCENARIO 3: NFT functionality is separated from `MondrianWallet` (separation of concerns is among the most important best practices in smart contract development) as follows:
3. 1. `MondrianWallet` does not inherit `ERC721`,
3. 2. NFT functionality is handled in a separate contract, `MondrianNFT`,
3. 3. `MondrianNFT` has an external mint function to mint NFTs with incremental `tokenID`s, and this mint function is gated to `MondrianWallet` instances (via a check of the runtime bytecode),
3. 4.  Each `MondrianWallet` instance mints one NFT by calling `MondrianNFT::mint` from its constructor.

In this scenario, the distribution share of the 4 different `tokenURI`s will be as follows.
Consider the first 12 `tokenID`s and the `modNumber`s and `tokenURI`s corresponding to each:

| tokenId | modNumber | tokenURI  |
|---------|-----------|-----------|
|    0    |     0     | ART_ONE   |
|    1    |     1     | ART_TWO   |
|    2    |     2     | ART_THREE |
|    3    |     3     | ART_FOUR  |
|    4    |     4     | ART_FOUR  |
|    5    |     5     | ART_FOUR  |
|    6    |     6     | ART_FOUR  |
|    7    |     7     | ART_FOUR  |
|    8    |     8     | ART_FOUR  |
|    9    |     9     | ART_FOUR  |
|   10    |     0     | ART_ONE   |
|   11    |     1     | ART_TWO   |
|   12    |     2     | ART_THREE |
|   13    |     3     | ART_FOUR  |
|   14    |     4     | ART_FOUR  |
|   ...    |     ...     | ...  |

For a larger number of NFTs, the distribution of the four different `tokenURI`s in percentages will be as follows:

| tokenURI |  distribution share (%)  |
|---------|-----------|
|    ART_ONE    |     10%   |
|    ART_TWO    |     10%   |
|    ART_THREE    |    10% |
|    ART_FOUR    |     70%  |


## Tools Used
Manual review, Foundry.

## Recommendations

Before fixing this issue, first you need to make a design choice on how NFT minting (i.e. `tokenId` distribution) is supposed to be handled. Any of the 3 scenarios is a viable option if then the logic for assiging `tokenURI`s to `tokenId`s is appropriately modified. However, only SCENARIO 3 applies separation of concerns, so consider implementing that one as follows:

1. `MondrianWallet`:

```diff
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.24;

// AccountAbstraction Imports
import {IAccount} from "accountabstraction/contracts/interfaces/IAccount.sol";
import {IEntryPoint} from "accountabstraction/contracts/interfaces/IEntryPoint.sol";
import {UserOperationLib} from "accountabstraction/contracts/core/UserOperationLib.sol";
import {SIG_VALIDATION_FAILED, SIG_VALIDATION_SUCCESS} from "accountabstraction/contracts/core/Helpers.sol";
import {PackedUserOperation} from "accountabstraction/contracts/interfaces/PackedUserOperation.sol";
// OZ Imports
import {MessageHashUtils} from "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
- import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
+ import {MondrianNFT} from "./MondrianNFT.sol";  


/**
 * Our abstract art account abstraction... hehe
 */
- contract MondrianWallet is Ownable, ERC721, IAccount {
+ contract MondrianWallet is Ownable, IAccount {
    using UserOperationLib for PackedUserOperation;

    error MondrianWallet__NotFromEntryPoint();
    error MondrianWallet__NotFromEntryPointOrOwner();
-    error MondrainWallet__InvalidTokenId();

-    /*//////////////////////////////////////////////////////////////
-                                NFT URIS
-    //////////////////////////////////////////////////////////////*/
-    string constant ART_ONE = "ar://jMRC4pksxwYIgi6vIBsMKXh3Sq0dfFFghSEqrchd_nQ";
-    string constant ART_TWO = "ar://8NI8_fZSi2JyiqSTkIBDVWRGmHCwqHT0qn4QwF9hnPU";
-    string constant ART_THREE = "ar://AVwp_mWsxZO7yZ6Sf3nrsoJhVnJppN02-cbXbFpdOME";
-    string constant ART_FOUR = "ar://n17SzjtRkcbHWzcPnm0UU6w1Af5N1p0LAcRUMNP-LiM";

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
    IEntryPoint private immutable i_entryPoint;
+   MondiranNFT private immutable i_mondrianNFT;

    /*//////////////////////////////////////////////////////////////
                               MODIFIERS
    //////////////////////////////////////////////////////////////*/
    modifier requireFromEntryPoint() {
        if (msg.sender != address(i_entryPoint)) {
            revert MondrianWallet__NotFromEntryPoint();
        }
        _;
    }

    modifier requireFromEntryPointOrOwner() {
        if (msg.sender != address(i_entryPoint) && msg.sender != owner()) {
            revert MondrianWallet__NotFromEntryPointOrOwner();
        }
        _;
    }

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
-    constructor(address entryPoint) Ownable(msg.sender) ERC721("MondrianWallet", "MW") {
+    constructor(address entryPoint, address _mondrianNftAddress) Ownable(msg.sender) {

        i_entryPoint = IEntryPoint(entryPoint);
+       i_mondrianNFT = MondrianNFT(_mondrianNftAddress);
+       i_mondrianNFT.mint(address(this));
    }

    receive() external payable {}

    /*//////////////////////////////////////////////////////////////
                          FUNCTIONS - EXTERNAL
    //////////////////////////////////////////////////////////////*/
    /// @inheritdoc IAccount
    function validateUserOp(PackedUserOperation calldata userOp, bytes32 userOpHash, uint256 missingAccountFunds)
        external
        virtual
        override
        requireFromEntryPoint
        returns (uint256 validationData)
    {
        validationData = _validateSignature(userOp, userOpHash);
        _validateNonce(userOp.nonce);
        _payPrefund(missingAccountFunds);
    }

    /**
     * execute a transaction (called directly from owner, or by entryPoint)
     * @param dest destination address to call
     * @param value the value to pass in this call
     * @param func the calldata to pass in this call
     */
    function execute(address dest, uint256 value, bytes calldata func) external requireFromEntryPointOrOwner {
        (bool success, bytes memory result) = dest.call{value: value}(func);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }

    /*//////////////////////////////////////////////////////////////
                          FUNCTIONS - INTERNAL
    //////////////////////////////////////////////////////////////*/
    /**
     * Validate the signature is valid for this message.
     * @param userOp          - Validate the userOp.signature field.
     * @param userOpHash      - Convenient field: the hash of the request, to check the signature against.
     *                          (also hashes the entrypoint and chain id)
     * @return validationData - Signature and time-range of this operation.
     *                          <20-byte> aggregatorOrSigFail - 0 for valid signature, 1 to mark signature failure,
     *                                    otherwise, an address of an aggregator contract.
     *                          <6-byte> validUntil - last timestamp this operation is valid. 0 for "indefinite"
     *                          <6-byte> validAfter - first timestamp this operation is valid
     *                          If the account doesn't use time-range, it is enough to return
     *                          SIG_VALIDATION_FAILED value (1) for signature failure.
     *                          Note that the validation code cannot use block.timestamp (or block.number) directly.
     */
    function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
        internal
        pure
        returns (uint256 validationData)
    {
        bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
        ECDSA.recover(hash, userOp.signature);
        return SIG_VALIDATION_SUCCESS;
    }

    /**
     * Validate the nonce of the UserOperation.
     * This method may validate the nonce requirement of this account.
     * e.g.
     * To limit the nonce to use sequenced UserOps only (no "out of order" UserOps):
     *      `require(nonce < type(uint64).max)`
     * For a hypothetical account that *requires* the nonce to be out-of-order:
     *      `require(nonce & type(uint64).max == 0)`
     *
     * The actual nonce uniqueness is managed by the EntryPoint, and thus no other
     * action is needed by the account itself.
     *
     * @param nonce to validate
     *
     * solhint-disable-next-line no-empty-blocks
     */
    function _validateNonce(uint256 nonce) internal view virtual {}

    /**
     * Sends to the entrypoint (msg.sender) the missing funds for this transaction.
     * SubClass MAY override this method for better funds management
     * (e.g. send to the entryPoint more than the minimum required, so that in future transactions
     * it will not be required to send again).
     * @param missingAccountFunds - The minimum value this method should send the entrypoint.
     *                              This value MAY be zero, in case there is enough deposit,
     *                              or the userOp has a paymaster.
     */
    function _payPrefund(uint256 missingAccountFunds) internal virtual {
        if (missingAccountFunds != 0) {
            (bool success,) = payable(msg.sender).call{value: missingAccountFunds, gas: type(uint256).max}("");
            (success);
            //ignore failure (its EntryPoint's job to verify, not account.)
        }
    }

    /*//////////////////////////////////////////////////////////////
                             VIEW AND PURE
    //////////////////////////////////////////////////////////////*/
-    function tokenURI(uint256 tokenId) public view override returns (string memory) {
-        if (ownerOf(tokenId) == address(0)) {
-            revert MondrainWallet__InvalidTokenId();
-        }
-        uint256 modNumber = tokenId % 10;
-        if (modNumber == 0) {
-            return ART_ONE;
-        } else if (modNumber == 1) {
-            return ART_TWO;
-        } else if (modNumber == 2) {
-            return ART_THREE;
-        } else {
-            return ART_FOUR;
-        }
-    }

    function getEntryPoint() public view returns (IEntryPoint) {
        return i_entryPoint;
    }

    /**
     * Return the account nonce.
     * This method returns the next sequential nonce.
     * For a nonce of a specific key, use `entrypoint.getNonce(account, key)`
     */
    function getNonce() public view virtual returns (uint256) {
        return i_entryPoint.getNonce(address(this), 0);
    }

    /**
     * check current account deposit in the entryPoint
     */
    function getDeposit() public view returns (uint256) {
        return i_entryPoint.balanceOf(address(this));
    }

    /**
     * deposit more funds for this account in the entryPoint
     */
    function addDeposit() public payable {
        i_entryPoint.depositTo{value: msg.value}(address(this));
    }
}
```


1. `MondrianNFT`:

Note:
1. 1. the external `mint` function is gated to `MondrianWallet` instances (based on a check on runtime bytecode),
1. 2. minting is starting with `tokenId=0`, and `tokenId` is incremented,
1. 3. `tokenURI`s are assigned to `tokenId`s randomly. Randomness is generated via Chainlink VRF. 

**Note: this would not work on ZkSync.** Chainlink's VRF v2 service is currently not available on the zkSync. An alternative provider for random numbers on ZkSync could be Randomizer.AI ( https://randomizer.substack.com/p/introducing-randomizerai-random-numbers-22-06-25 ).

```diff
+ // SPDX-License-Identifier: MIT
+ pragma solidity 0.8.24;

+ import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
+ import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";
+ import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
+ import "@openzeppelin/contracts/access/Ownable.sol";
+ import "./MondrianWallet.sol";

+ contract MondrianNFT is ERC721, Ownable, VRFConsumerBaseV2 {
+    uint256 public tokenCounter;
+    mapping(uint256 => string) private _tokenURIs;
+    VRFCoordinatorV2Interface private vrfCoordinator;
+    bytes32 private keyHash;
+    uint64 private subscriptionId;
+    uint32 private callbackGasLimit;
+    uint16 private requestConfirmations;
+    uint32 private numWords;

+    // Store the hash of the MondrianWallet runtime bytecode
+    bytes32 public constant MONDRIAN_WALLET_CODEHASH = keccak256(type(MondrianWallet).runtimeCode);

+    string constant ART_ONE = "ar://jMRC4pksxwYIgi6vIBsMKXh3Sq0dfFFghSEqrchd_nQ";
+    string constant ART_TWO = "ar://8NI8_fZSi2JyiqSTkIBDVWRGmHCwqHT0qn4QwF9hnPU";
+    string constant ART_THREE = "ar://AVwp_mWsxZO7yZ6Sf3nrsoJhVnJppN02-cbXbFpdOME";
+    string constant ART_FOUR = "ar://n17SzjtRkcbHWzcPnm0UU6w1Af5N1p0LAcRUMNP-LiM";

+    string[] private artURIs = [ART_ONE, ART_TWO, ART_THREE, ART_FOUR];
+    uint256 private randomResult;

+    event RandomnessRequested(uint256 requestId);

+    constructor(
+        address _vrfCoordinator,
+        bytes32 _keyHash,
+        uint64 _subscriptionId,
+        uint32 _callbackGasLimit,
+        uint16 _requestConfirmations,
+        uint32 _numWords
+    )
+        ERC721("MondrianNFT", "MNFT")
+        VRFConsumerBaseV2(_vrfCoordinator)
+    {
+        vrfCoordinator = VRFCoordinatorV2Interface(_vrfCoordinator);
+        keyHash = _keyHash;
+        subscriptionId = _subscriptionId;
+        callbackGasLimit = _callbackGasLimit;
+        requestConfirmations = _requestConfirmations;
+        numWords = _numWords;
+    }

+    function mint(address to) external {
+        require(isMondrianWallet(msg.sender), "Not authorized to mint");
+
+        uint256 tokenId = tokenCounter;
+        tokenCounter++;

+        // Request randomness for tokenURI assignment
+        uint256 requestId = vrfCoordinator.requestRandomWords(
+            keyHash,
+            subscriptionId,
+            requestConfirmations,
+            callbackGasLimit,
+            numWords
+        );
        
+        emit RandomnessRequested(requestId);

+        // Mint the NFT
+        _safeMint(to, tokenId);
+    }

+    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
+        uint256 randomIndex = randomWords[0] % artURIs.length;
+        _tokenURIs[tokenCounter - 1] = artURIs[randomIndex];
+    }

+    function tokenURI(uint256 tokenId) public view override returns (string memory) {
+        require(_exists(tokenId), "ERC721Metadata: URI query for nonexistent token");
+        return _tokenURIs[tokenId];
+    }

+    function isMondrianWallet(address account) internal view returns (bool) {
+        return keccak256(account.code) == MONDRIAN_WALLET_CODEHASH;
+    }
+}
```
## <a id='H-02'></a>H-02. Missing signature validation in `MondrianWallet::_validateSignature`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L113-L121

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L120

## Summary
`MondrianWallet::_validateSignature` does not check if the address recovered by `ECDSA.recover(hash, userOp.signature);` matches the expected (accepted) signer's address. This oversight allows unauthorized users to execute transactions.

## Vulnerability Details
`MondrianWallet::_validateSignature` is supposed to 
- STEP1: recover the signer of the submitted `userOp` using the `EDSA.recover` method,
- STEP2: validate the signer's address matches the address (or addresses) that are authorized to sign  `UserOperation`s, and then
- STEP3: depending on the success or failure of the validation, return an approriate value.

In practice, however, the STEP2 (signature validation against accepted addresses) is omitted, while at STEP3 the return value is always `SIG_VALIDATION_SUCCESS`.

## Impact

Due to the missing signature validation, `MondrianWallet` will accept and execute any `userOp` that has been signed by any user, provided that the other fields of the `userOp` are valid (such ad `nonce`). This vulnerability allows any user to execute arbitrary transactions through the `MondrianWallet` by submitting a `userOp` via `iEntryPoint::handleOps`.

The following piece of test demonstrates that a non-owner user signs a `UserOp` for minting `ERC20` tokens via the `owner`'s `MondrianWallet`. Despite the user not having a signed transaction from the Mondrian Wallet Owner,  the transactions executes, 
Note: the test was written in Foundry.

<details>
<summary>Proof of Code</summary>

```javascript
    // add import: import {UserOperationLib, PackedUserOperation} from "../src/MondrianWallet.sol";
    // add using:     using UserOperationLib for PackedUserOperation;
   function testMissingSignatureValidation() public {
        address nonOwner = makeAddr("nonOwner");

        // user sets up a Mondrian Wallet
        address owner = makeAddr("owner");
        vm.prank(owner);
        MondrianWallet mondrianWallet;
        mondrianWallet = new MondrianWallet(address(entryPoint));

        // Prepare the data to call mint on the MockERC20 contract
        bytes memory data = abi.encodeWithSignature("mint()");

        // Pack gas limits
        bytes32 packedGasLimits = packGasLimits(1000000, 100000);
        // Pack gas fees
        bytes32 packedGasFees = packGasFees(tx.gasprice, tx.gasprice);

        // Prepare the UserOperation - mondrianWallet to mint ERC20
        PackedUserOperation memory userOp = PackedUserOperation({
            sender: address(mondrianWallet),
            nonce: 0, 
            initCode: "",
            callData: abi.encodeWithSelector(mondrianWallet.execute.selector, address(erc20), 0, data),
            accountGasLimits: packedGasLimits,
            preVerificationGas: 21000,
            gasFees: packedGasFees,
            paymasterAndData: "",
            signature: new bytes(65) // placeholder, will be updated at line userOp.signature = abi.encodePacked(r, s, v);
        });

        // Non-owner signs the UserOperation
        // signature field of UserOp (UserOp.signature) is populated here
        bytes32 userOpHash = keccak256(abi.encode(userOp));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(uint256(uint160(nonOwner)), userOpHash);
        userOp.signature = abi.encodePacked(r, s, v);

        // Prepare UserOperation array
        PackedUserOperation[] memory ops = new PackedUserOperation[](1);
        ops[0] = userOp;

        iEntryPoint = mondrianWallet.getEntryPoint();
        iEntryPoint.handleOps(ops, payable(owner));

        // Verify that the tokens were minted correctly
        uint256 balance = erc20.balanceOf(address(mondrianWallet));
        assertEq(balance, erc20.AMOUNT());
    }


    function packGasLimits(uint256 callGasLimit, uint256 verificationGasLimit) internal pure returns (bytes32) {
        return bytes32((callGasLimit << 128) | verificationGasLimit);
    }

    function packGasFees(uint256 maxFeePerGas, uint256 maxPriorityFeePerGas) internal pure returns (bytes32) {
        return bytes32((maxFeePerGas << 128) | maxPriorityFeePerGas);
    }
```
</details>

## Tools Used
Manual review, Foundry.

## Recommendations

implement the missing signature validation step in `_validateSignature`: ensure that the recovered address matches the expected signer's address (i.e., the wallet owner). The function should return `SIG_VALIDATION_FAILED` if the signature does not match the owner's address.

Here is the recommended code change:

```diff
    function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
        internal
        pure
        returns (uint256 validationData)
    {
        bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
-      ECDSA.recover(hash, userOp.signature);
+      address recoveredSigner = ECDSA.recover(hash, userOp.signature);

+      if(recoveredSigner != owner) {
+             return SIG_VALIDATION_FAILED;
+      }
        return SIG_VALIDATION_SUCCESS;
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing NFT distribution for Mondrian Wallets            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L59C1-L61C6

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L162

## Summary

No NFT is distributed to the created Mondrian Wallets.

## Vulnerability Details

As per the README, `MondrianWallet` instances should receive a Mondrian NFT. In practice, however, they do not receive one: `MondrianWallet` does not mint or distribute any NFTs.

The following test (written in Foundry) demonstrates that a user creates a Mondrian Wallet, but does not receive any NFTs.
For any `tokenId`, `MondrianWallet::tokenURI` will revert with custom error `MondrainWallet__InvalidTokenId()`.

<details>
<summary>Proof of Code</summary>

```javascript
    function testNoNftIsMinted() public {
        // user sets up a Mondrian Wallet
        address owner = makeAddr("owner");
        vm.prank(owner);
        MondrianWallet mondrianWallet;
        mondrianWallet = new MondrianWallet(address(entryPoint));

        bytes memory expectedRevertReason = abi.encodeWithSignature("ERC721NonexistentToken(uint256)", 0);

        vm.expectRevert(expectedRevertReason);
        string memory art = mondrianWallet.tokenURI(0);
        //console.log("Art: ", art);

        uint256 balanceMW = mondrianWallet.balanceOf(address(mondrianWallet));
        uint256 balanceOwner = mondrianWallet.balanceOf(owner);

        console.log("Mondrian wallet balance: ", balanceMW);
        console.log("Owner balance: ", balanceOwner);
    }
```
</details>

## Impact

Mondrian Wallets do not receive the Mondrian NFTs that they are entitled to.

## Tools Used
Manual review, Foundry.

## Recommendations

Modify `MondrianWallet` so that an NFT is minted when a `MondrianWallet` instance is created.

Regarding the code below, do note that it contains a suggestion only for fixing the absence of NFT minting and distribution. However, the distribution will not be fully as desired. For the desired outcome you have to make a design choice:
- either ensure that `tokenId` is a random `uint256` (by using ChainLink VRFv2 for deployment on Ethereum, and some other oracle for ZkSync), or
- decouple the account abstraction and NFT functionality by not inheriting ĘRC721` in `MondrianWallet`, but rather creating a separate MondrianNFT contract, and then handle all NFT-related logic therein.

```diff
    ...

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
    IEntryPoint private immutable i_entryPoint;
+  uint256 tokenId = 1;  // @note the value of tokenId should be random

    /*//////////////////////////////////////////////////////////////
                               MODIFIERS
    //////////////////////////////////////////////////////////////*/
    modifier requireFromEntryPoint() {
        if (msg.sender != address(i_entryPoint)) {
            revert MondrianWallet__NotFromEntryPoint();
        }
        _;
    }

    modifier requireFromEntryPointOrOwner() {
        if (msg.sender != address(i_entryPoint) && msg.sender != owner()) {
            revert MondrianWallet__NotFromEntryPointOrOwner();
        }
        _;
    }

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    constructor(address entryPoint) Ownable(msg.sender) ERC721("MondrianWallet", "MW") {
        i_entryPoint = IEntryPoint(entryPoint);
+     _mint(address(this), tokenId); // Mint to the new Mondrian Wallet

    }

...
```
## <a id='M-02'></a>M-02. Accounts using non-standard signing methods won't work with `MondrianWallet`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L119

## Summary
`MondrianWallet` expects ECDSA signatures, but ZkSync accounts might use non-standard signing methods. Any such accounts won't work with `MondrianWallet`.

## Vulnerability Details
zkSync's account abstraction allows accounts to use custom logic for signing transactions, not just ECDSA signatures. This means accounts using non-standard signing methods won't work with MondrianWallet as it currently relies on ECDSA.

## Tools Used
Manual review.

## Recommendations
Follow the recommendations in the ZkSync documentation:

1. https://docs.zksync.io/build/quick-start/best-practices.html#gasperpubdatabyte-should-be-taken-into-account-in-development 

_Use zkSync Era's native account abstraction support for signature validation instead of this [ecrecover] function.
We recommend not relying on the fact that an account has an ECDSA private key, since the account may be governed by multisig and use another signature scheme._

2. https://docs.zksync.io/build/developer-reference/account-abstraction.html

_The @openzeppelin/contracts/utils/cryptography/SignatureChecker.sol library provides a way to verify signatures for different account implementations. We strongly encourage you to use this library whenever you need to check that a signature of an account is correct_

# Low Risk Findings

## <a id='L-01'></a>L-01. ZkSync Era does not have native support for transferring Ether            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L88

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L152

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/78743748751efc269d4cfb6de087f771d950d7c8/contracts/MondrianWallet.sol#L201

## Summary
`MondiranWallet` has multiple functions with low-level calls that are supposed to transfer Ether. However, ZkSync Era does not have native support for transferring Ether, so deployment to ZkSync requires additional considerations.

## Vulnerability Details
`MondrianWallet` utilizes low-level calls (indicated with "@> below) to directly transfer Ether in the following three functions:

```javascript
    function execute(address dest, uint256 value, bytes calldata func) external requireFromEntryPointOrOwner {
@>        (bool success, bytes memory result) = dest.call{value: value}(func);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }
```

```javascript
    function _payPrefund(uint256 missingAccountFunds) internal virtual {
        if (missingAccountFunds != 0) {
@>            (bool success,) = payable(msg.sender).call{value: missingAccountFunds, gas: type(uint256).max}("");
            (success);
            //ignore failure (its EntryPoint's job to verify, not account.)
        }
    }
```

```javascript
    function addDeposit() public payable {
@>        i_entryPoint.depositTo{value: msg.value}(address(this));
    }
```

However, ZkSync does not have support for such direct Ether transfers, see the documentation at:
- https://docs.zksync.io/build/developer-reference/differences-with-ethereum.html#call-staticcall-delegatecall
- https://docs.zksync.io/zk-stack/components/smart-contracts/system-contracts.html#l2ethtoken-msgvaluesimulator
- https://docs.zksync.io/zk-stack/components/compiler/specification/system-contracts.html#ether-value-simulator

Instead, Ether transfers on ZkSync are handled by a special system contract called `MsgValueSimulator`. 

Further details on how this work is available here: https://www.rollup.codes/zksync-era

"`CALL`	Creates a new sub-context and executes the code of the given account.
The OPCODE does not support sending ether natively
If not compiled with zk-solc or zk-vyper, Ether must be send using a system contract `MsgValueSimulator` prior to executing the `CALL` opcode.
zk-solc and zk-vyper compilers inject the call to the system contract under the hood during compilation.
...
"


## Impact

The impact depends on the compilation:

- if the contract is compiled with `zk-solc` or `zk-vyper`, there is no impact, as Ether transfer through `CALL` is handled automatically by the compiler, which injects calls to the `MsgValueSimulator` system contract.
- if the contract is not compiled with `zk-solc` or `zk-vyper`, `MondrianWallet::execute`, `MondrianWallet::_payPrefund`, and `MondrianWallet::addDeposit` will not work on ZkSync and the contract will be esentially disfunctional, provided that the code is not adjusted.

## Tools Used
Manual review, Foundry.

## Recommendations
Either 

- compile the contract using zk-solc for zkSync (so that you can use the `CALL` opcode directly without manually invoking the `MsgValueSimulator`, or
- tod ensure compatibility across all environments, check the `chainID` and explicitly use the `MsgValueSimulator` for zkSync.


