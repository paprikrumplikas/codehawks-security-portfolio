# First Flight #14: AirDropper - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unsafe use of Foundry's FFI capability in test scripts](#H-01)
    - ### [H-02. `MerkleAirdrop::claim` does not check whether an eligible user already claimed, allowing for repeated claims](#H-02)
    - ### [H-03. Anyone can submit a claim on behalf of any address, and anyone can precompute Merkle proofs](#H-03)
    - ### [H-04. Incorrect Merkle root `s_merkleRoot` in `Deploy.s.sol`](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Unclaimed USDC is impossible to recover from `MerkleAirdrop`](#M-01)
    - ### [M-02. Incorrect USDC token address `s_zkSyncUSDC` in `Deploy.s.sol`](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #14

### Dates: Apr 25th, 2024 - May 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 2
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Unsafe use of Foundry's FFI capability in test scripts            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/foundry.toml#L6

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/test/MerkleAirdropTest.t.sol#L41-L45

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/test/mocks/CheatCodes.t.sol#L1-L6

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/Makefile#L5

## Summary

The test script for `MerkleAirdrop` utilizes Foundry's Foreign Function Interface (FFI) capability to execute arbitrary code at the operating system level, which poses significant security risks. This feature, if misused or exploited in malicious or vulnerable scripts, can lead to severe implications including system compromise or data leakage.

## Vulnerability Details

The function `testPwned()` in `MerkleAirdrop.t.sol` uses FFI to execute system-level commands. Specifically, it uses the `touch` command to create or update a file named "youve-been-pwned" on the host machine. This demonstrates the potential for test scripts to perform unauthorized actions beyond the Ethereum Virtual Machine (EVM) environment, affecting the host system directly.

```javascript
function testPwned() public {
    string[] memory cmds = new string[](2);
    cmds[0] = "touch";
    cmds[1] = "youve-been-pwned";
    cheatCodes.ffi(cmds);
}
```

Importantly, the invocation of this function is triggered instantaneously as part of the automated processes defined in the Makefile's make command, significantly raising the potential for unintended consequences.

## Impact
By enabling FFI in test environments, developers expose their systems to a range of attacks that could lead to unauthorized access or manipulation of the filesystem, execution of arbitrary and potentially harmful commands, and other unintended actions that could compromise both the integrity and confidentiality of the system.

## Tools Used
Foundry, maual review.

## Recommendations

1. Disable the FFI capability in Foundry's configuration when not strictly necessary, particularly in shared or public codebases, to prevent the execution of arbitrary system commands.

2. Review and audit any use of FFI in smart contract development environments to ensure that it is used securely and only where absolutely necessary.
Implement strict access controls and review processes for any scripts that require FFI capabilities to mitigate potential security risks.

3. To disable FFI, modify the `foundry.toml` configuration file:

```diff
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
remappings = ['@openzeppelin/contracts=lib/openzeppelin-contracts/contracts']
- ffi = true
+ ffi = false
solc_version = "0.8.24"
solc = "0.8.24"

...
```
## <a id='H-02'></a>H-02. `MerkleAirdrop::claim` does not check whether an eligible user already claimed, allowing for repeated claims            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L30-L40

## Summary

`MerkleAirdrop::claim` does not track and check whether eligible users have already claimed their share of the airdrop. Consequently, eligible users can claim multiple times.

## Vulnerability Details

`MerkleAirdrop::claim` is supposed to enable airdrop-eligible users to claim their share of the airdrop. However, the function does not track and check whether eligible users have already claimed or not. As demonstrated by the test below, this is a vulnerability that can be exploited by eligible users to claim more than their share of the airdrop via submitting multiple claim transactions:

<details>
<summary> Proof of Code </summary>

```javascript

    function testSameUserCanClaimMoreWithMultipleClaims() public {
        uint256 noClaims = 3;
        uint256 startingBalance = token.balanceOf(collectorOne);
        vm.deal(collectorOne, noClaims * airdrop.getFee());

        vm.startPrank(collectorOne);
        for (uint256 i = 0; i < noClaims; i++) {
            airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        }
        vm.stopPrank();

        uint256 endingBalance = token.balanceOf(collectorOne);
        assertEq(endingBalance - startingBalance, amountToCollect * noClaims);
    }
```
</details>

## Impact
By repeatedly calling `MerkleAirdrop::claim`, airdrop-eligible users can claim more than they are eligible for. Essentially, a malicious airdrop-eligible user can
- claim the airdrop shares intended for other eligible users who have not claimed yet,
- drain the USDC balance of `MerkleAirdrop` until its balance has at least one share of airdrop amount left.

## Tools Used
Manual review, Foundry.

## Recommendations
Track and check which users have already claimed their share of the airdrop. Perform the following modifications in `MerkleAirdrop`:

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { MerkleProof } from "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { IERC20, SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract MerkleAirdrop is Ownable {
    using SafeERC20 for IERC20;

    error MerkleAirdrop__InvalidFeeAmount();
    error MerkleAirdrop__InvalidProof();
    error MerkleAirdrop__TransferFailed();

    uint256 private constant FEE = 1e9;
    IERC20 private immutable i_airdropToken;
    bytes32 private immutable i_merkleRoot;

+  mapping(address => bool) public hasClaimed;

    event Claimed(address account, uint256 amount);
    event MerkleRootUpdated(bytes32 newMerkleRoot);

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    constructor(bytes32 merkleRoot, IERC20 airdropToken) Ownable(msg.sender) {
        i_merkleRoot = merkleRoot;
        i_airdropToken = airdropToken;
    }

    function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
+      require(!hasClaimed[account], "You have already claimed.");

        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }

+     hasClaimed[account] = true;

        emit Claimed(account, amount);
        i_airdropToken.safeTransfer(account, amount);
    }

    ...

}
```


Note that due to another bug reported in another finding, the modified eligibility check above is this incomplete/incorrect. Taking into account both bugs, a full fix would look like as follows:

<details>
<summary>Fix for both bugs in eligibility check</summary>

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { MerkleProof } from "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { IERC20, SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract MerkleAirdrop is Ownable {
    using SafeERC20 for IERC20;

    error MerkleAirdrop__InvalidFeeAmount();
    error MerkleAirdrop__InvalidProof();
    error MerkleAirdrop__TransferFailed();

    uint256 private constant FEE = 1e9;
    IERC20 private immutable i_airdropToken;
    bytes32 private immutable i_merkleRoot;

+  mapping(address => bool) public hasClaimed;

    event Claimed(address account, uint256 amount);
    event MerkleRootUpdated(bytes32 newMerkleRoot);

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    constructor(bytes32 merkleRoot, IERC20 airdropToken) Ownable(msg.sender) {
        i_merkleRoot = merkleRoot;
        i_airdropToken = airdropToken;
    }

-    function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
+   function claim(uint256 amount, bytes32[] calldata merkleProof) external payable {
+      require(!hasClaimed[msg.sender], "You have already claimed.");

        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
-       bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
+       bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(msg.sender, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }

+     hasClaimed[msg.sender] = true;

-       emit Claimed(account, amount);
-       i_airdropToken.safeTransfer(account, amount);
+       emit Claimed(msg.sender, amount);
+       i_airdropToken.safeTransfer(msg.sender, amount);
    }

    ...

}
```
</details>
## <a id='H-03'></a>H-03. Anyone can submit a claim on behalf of any address, and anyone can precompute Merkle proofs            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/src/MerkleAirdrop.sol#L30

## Summary

`MerkleAirdrop::claim` does not check whether input parameter `account` is the same address as `msg.sender`. This flaw allows any caller to claim airdrops for any address if they have the corresponding Merkle proof.

The risk is exacerbated by the public knowledge of eligible addresses and amounts, enabling attackers to precompute Merkle proofs and claim airdrops fraudulently.

## Vulnerability Details

`MerkleAirdrop::claim` is supposed to enable airdrop-eligible users to claim their portion of the airdrop. However, the function verifies eligibility using the `account` parameter instead of the address `msg.sender`, and fails to ensure that these addresses match. While zkSync does not have a public mempool which makes traditional front-running less of a concern, the platform’s operators do possess the ability to see transactions before they are executed, allowing them to reorder transactions and claim before the eligible user.

To make things worse, the list of eligible addresses and their respective amounts are public data and, hence, an attacker can independently generate the necessary proofs without needing to intercept a specific transaction. This vulnerability could be exploited systematically, with bots programmed to claim all available airdrops before the rightful recipients.

The following test demonstrates that if a malicious, non-airdrop-eligible user acquires the proof and address of an airdrop-eligible user (either by precomputing the proofs from publicly available data, or by observing claim transactions in the mempool), then they can submit their own claim transaction with the these inputs, and can succesfully claim (by front-running the eligible user):

<details>
<summary>Proof of Code</summary>

```javascript
    function testAnyUserCanClaim() public {
        address user = makeAddr("user");
        uint256 startingBalance = token.balanceOf(user);

        vm.deal(user, airdrop.getFee());

        vm.startPrank(user);
        // Here, it is assumed that 'user' knows the address and proof of an eligible user.
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        vm.stopPrank();

        uint256 endingBalance = token.balanceOf(collectorOne);
        assertEq(endingBalance - startingBalance, amountToCollect);
    }
```
</details>

## Impact

Malicious, non-airdrop-eligible users can claim the airdrop reserved for airdrop-eligible users. To exploit this vulnerability, attackers can:

- reconstruct the Merkle tree using publicly available data, including the list of eligible addresses and their corresponding amounts, enabling them to generate the necessary proofs to submit fraudulent claims;
- leverage their position as operators within zkSync to observe and intercept valid claim transactions from eligible users, allowing them to submit these transactions on their behalf and claim the airdrops before the legitimate recipients."

## Tools Used
Manual review, Foundry.

## Recommendations

Perform the eligibility verification on address `msg.sender`, not on `MerkleAirdrop::claim` input parameter `account`. To do this, modify `MerkleAirdrop` as follows:

```diff
-    function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
+   function claim(uint256 amount, bytes32[] calldata merkleProof) external payable {

        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
-       bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
+       bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(msg.sender, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }
-      emit Claimed(account, amount);
+      emit Claimed(msg.sender, amount);
-       i_airdropToken.safeTransfer(account, amount);
+       i_airdropToken.safeTransfer(msg.sender, amount);

    }
```
## <a id='H-04'></a>H-04. Incorrect Merkle root `s_merkleRoot` in `Deploy.s.sol`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L9C20-L9C32

## Summary
An incorrect value is assigned to `Deploy::s_merkleRoot`.

## Vulnerability Details
`s_merkleRoot` in `Deploy.s.sol` is supposed to contain the Merkle root generated by `makeMerkle.js`. However, `makeMerkle.js` specifies the claimable amount incorrectly:

```javascript
@> const amount = (25 * 1e18).toString()
```

Given that the airdrop token is USDC, and the USDC token contract defines 6 decimals, the `amount` above esentially sets the claimable amount to `25 * 1e12` USDC, not to the intended `25` USDC. `makeMerkle.js` then encodes this incorrect `amount` value in the Merkle tree, resulting in an incorrect Merkle root - which in turn is used as an input parameter when deploying the `MerkleAirdrop` contract:

```javascript
contract Deploy is Script {
    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
@>    bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    // 4 users, 25 USDC each
    uint256 public s_amountToAirdrop = 4 * (25 * 1e6);

    // Deploy the airdropper
    function run() public {
        vm.startBroadcast();
@>        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }
```

## Impact

Due to the incorrect Merkle root used to deploy `MerkleAirdrop`, users are unable to claim the intended 25 USDC. Instead, the encoded data erroneously allows claims for 25 trillion USDC each—a claim that would fail as the contract does not and likely will never hold sufficient funds.

The following test demonstrates that users cannot claim 25 USDC (the trx will revert with `MerkleAirdrop::MerkleAirdrop__InvalidProof`), and neither they can claim `25 * 1e12` USDC, i.e. the airdrop amount that is encoded in the Merkle tree (the trx will revert with `ERC20InsufficientBalance`):

<details>
<summary> Proof of code </summary>

```javascript
   // import IERC20: import { MerkleAirdrop, IERC20 } from "../src/MerkleAirdrop.sol";
    function testIncorrectMerkleRoot() public {
        // Deploy script and test file use different setups.
        // To demonstrate the vulnerability, we use the same setup in this test as in the deploy script.

        // the test file had the correct Merkle root
        // to demonstrate the bug, re-set it to the incorrect value used in Deploy.s.sol
        bytes32 merkleRoot_bad = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;

        // the test file has the correct airdrop contract
        // to demonstrate the bug, we need an airdrop contract the is deployed with the wrong root
        // as done in Deploy.s.sol
        MerkleAirdrop airdrop_bad = new MerkleAirdrop(merkleRoot_bad, token);

        // the test file had the correct proof
        // to demonstrate the bug, set it to the incorrect value that the incorrect makeMerkle.js generates
        bytes32 proofOne_bad = 0x4fd31fee0e75780cd67704fbc43caee70fddcaa43631e2e1bc9fb233fada2394;
        bytes32 proofTwo_bad = 0xc88d18957ad6849229355580c1bde5de3ae3b78024db2e6c2a9ad674f7b59f84;
        bytes32[] memory proof_bad = new bytes32[](2);
        proof_bad[0] = proofOne_bad;
        proof_bad[1] = proofTwo_bad;

        // setting up balances
        vm.deal(collectorOne, airdrop_bad.getFee());
        token.mint(address(airdrop_bad), 4 * 25 * 1e6);
        assert(IERC20(token).balanceOf(address(airdrop_bad)) == 4 * 25 * 1e6);

        // @note SCENARIO 1: collector tries to collect 25 USDC
        // but the trx fails as a different value is encoded in the Merkle root
        vm.prank(collectorOne);
        // expectRevert would expect a revert for this call which is not we want, so
        // moving this out from the airdrop.claim call
        uint256 fee = airdrop_bad.getFee();
        vm.expectRevert(MerkleAirdrop.MerkleAirdrop__InvalidProof.selector);
        airdrop_bad.claim{ value: fee }(collectorOne, amountToCollect, proof_bad);

        // add more USDC to the airdrop contract
        //vm.deal(address(airdrop), 25*1e12)

        // @note SCENARIO 2: collector tries to collect the value that is actually encoded in the Markle root
        // but trx reverts because the airdrop contract does not have enough balance

        //setting up balances
        vm.deal(collectorOne, airdrop_bad.getFee());
        uint256 amountEncodedInRoot = 25 * 1e18;
        vm.prank(collectorOne);
        vm.deal(address(airdrop_bad), amountEncodedInRoot);

        // encode expected error
        address expectedAddress = 0x6914631e3e71Bc75A1664e3BaEE140CC05cAE18B;
        uint256 currentBalance = 100_000_000; // 1e8
        uint256 requiredBalance = 25_000_000_000_000_000_000; // 25e18
        // Encode the expected revert reason
        // note need import: import { IERC20Errors } from "@openzeppelin/contracts//interfaces/draft-IERC6093.sol";
        bytes memory encodedRevertReason = abi.encodeWithSelector(
            IERC20Errors.ERC20InsufficientBalance.selector, expectedAddress, currentBalance, requiredBalance
        );

        vm.expectRevert(encodedRevertReason);
        airdrop_bad.claim{ value: fee }(collectorOne, amountEncodedInRoot, proof_bad);
    }
```
</details>

Eventually, the contract will have to be redeployed. Supposing that the address of the USDC contract is correct in `Deploy.t.sol` (which is actually not, as described in another finding), the `100 USDC` sent to `MerkleAirdrop` will not be possible to recover.


## Tools Used
Manual review, Foundry.

## Recommendations
Perform the following corrective steps:

- Correct the airdrop amount in `makeMerkle.js`:

```diff
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree")
const fs = require("fs")

/*//////////////////////////////////////////////////////////////
                             INPUTS
//////////////////////////////////////////////////////////////*/
// @audit amount is incorrect, should be 25*1e6
- const amount = (25 * 1e18).toString()
+ const amount = (25 * 1e6).toString()

...
```

- Built the Merkle tree again and replace the incorrect Merkle root in `Deploy.s.sol` with the correct one:

```diff
...
-    bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
+    bytes32 public s_merkleRoot = 0x3b2e22da63ae414086bec9c9da6b685f790c6fab200c7918f2879f08793d77bd;
...
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unclaimed USDC is impossible to recover from `MerkleAirdrop`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L18

## Summary

`Deploy::run` sends `100 USDC` (the total airdrop amount) to `MerkleAirdrop`:

```javascript
    function run() public {
        vm.startBroadcast();
        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
@>     IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }
```

Since `MerkleAirdrop` does not have any methods for transferring USDC out of the contract, any unclaimed USDC will not be possible to recover.

## Vulnerability Details

Right at deployment, the total airdrop amount of 100 USDC is sent to `MerkleAirdrop`. 4 addresses are eligible to claim `25 USDC` each, but if any of them cannot claim, USDC funds will be irrecoverably stuck in the contract. Reasons why users might not (be able to) claim their share of the airdrop:

- missing the notification about the existence of the airdrop and about their eligibility,
- lost access to eligible address.

## Impact

Any unclaimed USDC funds are impossible to recover from the `MerkleAirdrop` contract.

## Tools Used
Manual review, Foundry.

## Recommendations
Add a method to `MerkleAirdrop` so that USDC can be transferred out from the contract:

```diff
+    function recoverUsdc(address _receiver) external onlyOwner {
+        i_airdropToken.safeTransfer(_receiver, i_airdropToken.balanceOf(address(this)));
+    }
```
## <a id='M-02'></a>M-02. Incorrect USDC token address `s_zkSyncUSDC` in `Deploy.s.sol`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L8

## Summary

An incorrect value is assigned to `Deploy::s_zkSyncUSDC` (i.e. the USDC token address on zkSync).

## Vulnerability Details

`s_zkSyncUSDC` is a state variable in `Deploy.s.sol` that is supposed to contain the address of the USDC token on the zkSync network. This address is then used as a parameter when deploying the `MerkleAirdrop` contract, which requires an `ERC20` token contract (the address of the airdrop token) for operation:

```javascript
    function run() public {
        vm.startBroadcast();
@>        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
        IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }
```

However, `Deploy::s_zkSyncUSDC` is assigned an incorrect value (the correct USDC address on zkSync is `0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4`, not `0x1D17CbCf0D6d143135be902365d2e5E2a16538d4`.


## Impact

`MerkleAirdrop` will be deployed with an incorrect constructor input value, and the contract will be disfunctional (the incorrect address is not only not the USDC token address, but it is not a token address at all).
The contract will have to be redeployed.

## Tools Used
Manual review, Foundry.

## Recommendations
Correct the token address in `Deploy.s.sol` as follows:

```diff
-    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
+    address public s_zkSyncUSDC = 0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4;
```




