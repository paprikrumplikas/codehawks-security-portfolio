# First Flight #19: Mondrian Wallet v2 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Upgrade machanism lacks access control: `_authorizedUpgrade` does not implement access resctriction](#H-01)
    - ### [H-02. Outcome of signature verification is not checked in `executeTransactionFromTheOutside`, any signed transactions will be executed](#H-02)
    - ### [H-03. Missing access control in `payForTransaction`](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Accounts using non-standard signing methods won't work with `MondrianWallet2`](#M-01)
    - ### [M-02. `_executeTransaction` uses the `CALL` opcode instead of an assembly block to make a call](#M-02)
- ## Low Risk Findings
    - ### [L-01. `_executeTransaction` is unprepared to call system contracts other then the Deployer System Contract](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #19

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-Mondrian-Wallet_v2)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 2
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Upgrade machanism lacks access control: `_authorizedUpgrade` does not implement access resctriction            



## Summary

`_authorizedUpgrade` does not implement any access resctriction. Consequently, anyone can upgrade the Mondrian wallet of any user to any new implementation contract.

## Vulnerability Details

The upgrade mechanism from a current implementation contract to a new one can be initiated by calling `MondrianWallet2::upgadeToAndCall` (a function defined in the `UUPSUpgradeable` contract). This function then calls `MondrianWallett::_authorizedUpgrade`, a function that has to be implemented in the implementation contract as per required by `UUPSUpgradeable`, which contract also states:

*" The {\_authorizeUpgrade} function must be overridden to include access restriction to the upgrade mechanism.*\
*...*\
*Function that should revert when `msg.sender` is not authorized to upgrade the contract. Called by {upgradeToAndCall}.*\
*Normally, this function will use an xref:access.adoc\[access control] modifier such as {Ownable-onlyOwner}.*\
*function authorizeUpgrade(address) internal onlyOwner {}"*

However, `MondiranWallet::_authorizeUpgrade` does not actually implement any access control.

The following test demonstrates if any non-owner user initiates the upgrade mechaism by calling `upgradeToAndCall`, the upgrade runs successfully, as signified by the emitted event.

```javascript
    event Upgraded(address indexed implementation);

    function testZkAnyoneCanUpgrade() public onlyZkSync {
        address notOwner = makeAddr("notOwner");

        // newImplementation has to be a comtract and need to implement _authorizeUpgrade(address newImplementation)
        MondrianWallet2 newImplementation = new MondrianWallet2();

        // Expect the Upgraded event
        vm.expectEmit(true, true, true, true);
        emit Upgraded(address(newImplementation)); // defining the expected event emission

        vm.prank(notOwner);
        mondrianWallet.upgradeToAndCall(address(newImplementation), "");
    }
```

## Impact

Malicious users can upgrade the Mondrian wallet of any user to any new implementation contract. They can implement whatever they want in the new implementation contract, including code that leads to loss of access and loss of funds for the true, original owner of the wallet.

## Tools Used

Manual review, Foundry.

## Recommendations

Restrict access to the owner as follows:

```diff
+  error MondrianWallet2__NotTheOwner();

    function _authorizeUpgrade(address newImplementation) internal override {
+          if (msg.sender != owner()) {
+            revert MondrianWallet2__NotTheOwner;
+        }
    }
```

You could consider granting access not only to the owner but also to the `Bootloader`, but in general it is best not to grant access to such critical functions to anyone but the owner.

## <a id='H-02'></a>H-02. Outcome of signature verification is not checked in `executeTransactionFromTheOutside`, any signed transactions will be executed            



## Summary

`executeTransactionFromTheOutside` does not check the outcome of signature verification. Consequently, any signed `transaction` can be executed through this function, not only those that were signed by the actual owner of the contract.

## Vulnerability Details

`executeTransactionFromTheOutside` is a function that anyone can call to submit `transaction`s for execution via the wallet. Importantly, only those submitted `transaction`s are supposed to be executed that have been properly validated and have been signed by the owner of the wallet.

Transaction validation happens via a call to `_validateTransaction` which, among other things, performs signature verification and returns a `bytes4 magic` value that signifies the outcome of the signature verification. `executeTransactionFromTheOutside`, however, ignores this return value and proceeds to execute the submitted `transaction` irrespectively of the validity of the signature.

Essentially, any signed transactions will be executed even if the signature is not valid - provided of course, that they do not revert at other verification steps, like a balance check.

The following test demonstrates that a `transaction` signed with a dummy private key (i.e. not a valid signature, not a signature from the owner) will get executed by `executeTransactionFromTheOutside`.

```javascript
 // @note needs a modified helper, see below
    function testZkExecuteTransactionSignedByAnyone() public {
        // Arrange
        address anyUser = makeAddr("anyUser");
        address dest = address(usdc);
        uint256 value = 0;
        bytes memory functionData = abi.encodeWithSelector(ERC20Mock.mint.selector, anyUser, AMOUNT);

        Transaction memory transaction =
            _createUnsignedTransaction(mondrianWallet.owner(), 113, dest, value, functionData);

        // Act
        vm.startPrank(anyUser);
        transaction = _signTransaction(transaction); // if trx is not signed at all, next line would revert with [FAIL. Reason: ECDSAInvalidSignatureLength(0)]
        mondrianWallet.executeTransactionFromOutside(transaction);
        vm.stopPrank();

        // Assert
        assertEq(usdc.balanceOf(anyUser), AMOUNT);
    }

    function _signTransactionWithDummyKey(Transaction memory transaction) internal view returns (Transaction memory) {
        bytes32 unsignedTransactionHash = MemoryTransactionHelper.encodeHash(transaction);
        uint8 v;
        bytes32 r;
        bytes32 s;
        uint256 DUMMY_PRIVATE_KEY = 0x0;
        //uint256 ANVIL_DEFAULT_KEY = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
        (v, r, s) = vm.sign(DUMMY_PRIVATE_KEY, unsignedTransactionHash);
        Transaction memory signedTransaction = transaction;
        signedTransaction.signature = abi.encodePacked(r, s, v);
        return signedTransaction;
    }
```

## Impact
Any `transaction` that has been signed and is otherwise valid will be executed, even if the signature is invalid (is not coming from the owner). Accordingly, malicious users can drain any and all funds from the wallet.

## Tools Used

Manual review, Foundry.

## Recommendations

Ensure that the outcome of the signature verification is checked and that the transaction flow procceds only if the signature is valid. Perform the following modifications:

```diff
    function executeTransactionFromOutside(Transaction memory _transaction) external payable {
-       _validateTransaction(_transaction);
+      bytes4 magic =  _validateTransaction(_transaction);
+      if (magic != ACCOUNT_VALIDATION_SUCCESS_MAGIC) {
+             revert MondrianWallet2__InvalidSignature();
 +     }

        _executeTransaction(_transaction);
    }
```

## <a id='H-03'></a>H-03. Missing access control in `payForTransaction`            



## Summary

`payForTransaction` does not have any access control and, consequently, can be called by anyone any number of times.

## Vulnerability Details

`payForTransaction` is a method intended for paying the `Bootloader` for transactions. As such, it should be called only by the `Bootloader`, but this restriction is not implemented. Consequently, anyone can call this function with arbitrary transactions, as demonstrated by the following test:

```javascript
    function testZkAnyoneCanCallPayForTransaction() public {
        // Arrange
        address anyUser = makeAddr("anyUser");
        address dest = address(usdc);
        uint256 value = 0;
        bytes memory functionData = abi.encodeWithSelector(ERC20Mock.mint.selector, anyUser, AMOUNT);

        Transaction memory transaction =
            _createUnsignedTransaction(mondrianWallet.owner(), 113, dest, value, functionData);

        uint256 initialBalance = address(mondrianWallet).balance;

        // vm.txGasPrice(100); //setting gas price to 100 gwei - this has no effect on the amount paid to the bootloader

        vm.startPrank(anyUser);
        mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        vm.stopPrank();

        uint256 endingBalance = address(mondrianWallet).balance;

        assert(endingBalance < initialBalance);
    }
```

## Impact

A malicious user can keep calling `payForTransaction` until the wallet transfers all of its ether to the `Bootloader`.

## Tools Used

Manual review, Foundry.

## Recommendations

Restrict access to the function so that only the `Bootloader` can call it:

```diff
    function payForTransaction(bytes32, /*_txHash*/ bytes32, /*_suggestedSignedHash*/ Transaction memory _transaction)
        external
-      payable
+     payable requireFromBootLoader
    {
        bool success = _transaction.payToTheBootloader();
        if (!success) {
            revert MondrianWallet2__FailedToPay();
        }
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Accounts using non-standard signing methods won't work with `MondrianWallet2`            



## Summary
`MondrianWallet2` expects ECDSA signatures during signature verification, but ZkSync accounts might use non-standard signing methods. Any such accounts won't work with `MondrianWallet2`


## Vulnerability Details
zkSync's account abstraction allows accounts to use custom logic for signing transactions, not just ECDSA signatures. This means accounts using non-standard signing methods won't work with `MondrianWallet2` as it currently relies on ECDSA for signature verification.


## Tools Used
Manual review.

## Recommendations

Follow the recommendations in the ZkSync documentation:

1. <https://docs.zksync.io/build/quick-start/best-practices.html#gasperpubdatabyte-should-be-taken-into-account-in-development>

_Use zkSync Era's native account abstraction support for signature validation instead of this \[ecrecover] function.
We recommend not relying on the fact that an account has an ECDSA private key, since the account may be governed by multisig and use another signature scheme._

1. <https://docs.zksync.io/build/developer-reference/account-abstraction.html>

_The @openzeppelin/contracts/utils/cryptography/SignatureChecker.sol library provides a way to verify signatures for different account implementations. We strongly encourage you to use this library whenever you need to check that a signature of an account is correct_

1. <https://docs.zksync.io/build/developer-reference/account-abstraction/building-smart-accounts>

_For smart wallets, we highly encourage the implementation of the EIP1271 signature-validation scheme. This standard is endorsed by the ZKsync team and is integral to our signature-verification library._


## <a id='M-02'></a>M-02. `_executeTransaction` uses the `CALL` opcode instead of an assembly block to make a call            



## Summary
`_executeTransaction` uses the `CALL` opcode instead of an assembly block to make a call:

```javascript
    function _executeTransaction(Transaction memory _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint128 value = Utils.safeCastToU128(_transaction.value);
        bytes memory data = _transaction.data;

        if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
            uint32 gas = Utils.safeCastToU32(gasleft());
            SystemContractsCaller.systemCallWithPropagatedRevert(gas, to, value, data);
        } else {
            bool success;
@>            (success,) = to.call{value: value}(data);
            if (!success) {
                revert MondrianWallet2__ExecutionFailed();
            }
        }
    }
```

But the `CALL` opcode behaves differently on ZkSync than on Ethereum: https://www.rollup.codes/zksync-era 


## Impact
Unexpected behavior, failure to send ether.
If ether is to be sent via the `CALL` opcode, system contract `MsgValueSimulator` would have to be invoked but `_executeTransaction` is not prepared to interact with this system contract.

## Tools Used
Manual review, Foundry.

## Recommendations
Make the call in the ZkSync way:

```diff
    function _executeTransaction(Transaction memory _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint128 value = Utils.safeCastToU128(_transaction.value);
        bytes memory data = _transaction.data;

        if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
            uint32 gas = Utils.safeCastToU32(gasleft());
            SystemContractsCaller.systemCallWithPropagatedRevert(gas, to, value, data);
        } else {
            bool success;

-           (success,) = to.call{value: value}(data);
+           assembly {
+                success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
+           }

            if (!success) {
                revert MondrianWallet2__ExecutionFailed();
            }
        }
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. `_executeTransaction` is unprepared to call system contracts other then the Deployer System Contract            



## Summary

`_executeTransaction` is unprepared to call system contracts other then the Deployer System Contract, any calls specifying such system contracts in the `to` field of the `transaction` struct will fail.

## Vulnerability Details

ZkSync has a number of special contracts called system contracts. The ZkSync documentation (<https://docs.zksync.io/build/developer-reference/era-contracts/system-contracts>) introduces them as follows:

*While most of the primitive EVM opcodes can be supported out of the box (i.e. zero-value calls, addition/multiplication/memory/storage management, etc), some of the opcodes are not supported by the VM by default and they are implemented via “system contracts” — these contracts are located in a special kernel space, (...)*

Importantly, system contracts can be called only through the `SystemContractCaller` library. Accordingly, `_executeTransaction` should check whether the `to` field of the input parameter `transaction` corresponds to the address of a system contract and if yes, then should call that system contract via the `SystemContractCaller` library. 

`_executeTransaction` does check whether `to` is the address of the deployer system contract, but fails to account for cases when the any other system contract is specified in the `to` field.

```javascript
    function _executeTransaction(Transaction memory _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint128 value = Utils.safeCastToU128(_transaction.value);
        bytes memory data = _transaction.data;

@>        if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
            uint32 gas = Utils.safeCastToU32(gasleft());
@>           SystemContractsCaller.systemCallWithPropagatedRevert(gas, to, value, data);
        } else {
            bool success;
            (success,) = to.call{value: value}(data);
            if (!success) {
                revert MondrianWallet2__ExecutionFailed();
            }
        }
    }
```


## Impact
`_executeTransaction` will always fail if any other system contract than the deplyer system contract is specified in the `to` field of the input struct (`transaction`). MondrianWallet will not be able to interact with other system contracts in this function.

Such calls will revert with a general `MondrianWallet2__ExecutionFailed()` error.


## Tools Used

Manual review, Foundry.

## Recommendations
Consider adding conditionals in `_executeTransaction` for other system contracts like you did for the deployer system contract, so that transactions specifying those system contracts can be executed too.

To the very least, add conditionals that detect if `to` is a system contract, and revert such calls with a specific error message, like `MondrianWallet2__CannotCallThisSystemContract()`.



