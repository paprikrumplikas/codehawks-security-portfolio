# First Flight #16: Mafia Takedown - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. A call to `Shelf`'s `withdraw` function casues `MoneyVault::withdrawUSDC` to fail](#H-01)
    - ### [H-02. Front-running deposits in `Laundrette::depositTheCrimeMoneyInATM`](#H-02)
    - ### [H-03. Risk of blacklisting by the maganer of the `USDC` contract](#H-03)
    - ### [H-04. Omitted policy reconfig during module upgrade due to misassigned `dependencies` array in `Laundrette::configureDependencies`](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Funds are not transferred to `MoneyVault` during the migration](#M-01)
    - ### [M-02. `MoneyVault` lacks `moneyShelf` role, `MoneyVault::withdrawUSDC` will fail](#M-02)
    - ### [M-03. Any gangmember can kick out any other gangmember from the gang via `Laundrette::quitTheGang`](#M-03)
    - ### [M-04. `CrimeMoney` token is not a stablecoin](#M-04)
- ## Low Risk Findings
    - ### [L-01. Godfather cannot access functions reserved for gangmembers, as `Deployer` does not grant him the role](#L-01)
    - ### [L-02. Discrepancy in the decimal places of `CrimeMoney` and `USDC` - peg is not 1:1](#L-02)
    - ### [L-03. Incorrect script setup in `Deployer.s.sol`](#L-03)
    - ### [L-04. Godfather cannot reclaim `Kernel` admin rights via `Laundrette::retrieveAdmin`](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #16

### Dates: May 23rd, 2024 - May 30th, 2024

[See more contest details here](https://www.codehawks.com/contests/clwgiehgu00119zwn2xx92ay8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 4
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. A call to `Shelf`'s `withdraw` function casues `MoneyVault::withdrawUSDC` to fail            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/modules/MoneyVault.sol#L32

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/modules/Shelf.sol#L16-L17

## Summary
A call to `Shelf`'s `withdraw` function casues `MoneyVault::withdrawUSDC` to fail.

## Vulnerability Details

Consider the `MoneyVault::withdrawUSDC˙function:

```javascript
    function withdrawUSDC(address account, address to, uint256 amount) external {
        require(to == kernel.executor(), "MoneyVault: only GodFather can receive USDC");
        withdraw(account, amount);
        crimeMoney.burn(account, amount);
        usdc.transfer(to, amount);
    }
```

Within the function, `withdraw(account, amount);` would serve the purpose of updating the internal accounting regarding the depositied USDC balance of the caller of the function. However, according to the internal accounting of `MoneyVault`, everybody has a zero balance of USDC:

`MoneyVault` USDC balance does not and cannot come from deposits (that the internal accounting would track), but from a direct transfer (or transfers) from the godfather (that internal accounting does not track).

Given that as per the internal accounting of `MoneyVault` everybody has a zero balance of USDC, any calls to `MoneyVault::withdrawUSDC` will revert at line ``withdraw(account, amount);` with an arithmetic under-/overflow error.

This is demonstarted by the following test:

<details>
<summary>Proof of Code</summary>

```javascript
   function testCannotWithdrawFromMoneyVault() public {
        ////////////////////////////////////////////////
        /// Emergency migration ////////////////////////
        ////////////////////////////////////////////////

        EmergencyMigration migration;

        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyShelf));

        migration = new EmergencyMigration();
        MoneyVault moneyVault = migration.migrate(kernel, usdc, crimeMoney);

        // mitigating the effects of another bug: reconfiguring the policy
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
        vm.stopPrank();

        // mitigating the effects of another bug: granting the moneyShelf role
        vm.startPrank(godFather);
        kernel.executeAction(Actions.ChangeAdmin, godFather);
        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyVault));
        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
        vm.stopPrank();

        assertEq(kernel.hasRole(address(moneyVault), Role.wrap("moneyshelf")), true);

        //////////////////////////////////////////////////
        /////// SETUP OF BALANCES ////////////////////////
        //////////////////////////////////////////////////

        // suppose USDC funds have been transferred to moneyVault
        uint256 savedFunds = 1e6; // i.e. total supply of crimeMoney
        deal(address(usdc), address(moneyVault), savedFunds);

        // suppose godfather recovered crimeMoney for withdrawal
        uint256 crimeFunds = savedFunds;
        deal(address(crimeMoney), godFather, crimeFunds);

        //////////////////////////////////////////////////
        /////// GODFATHER TRIES TO WITHDRAW USDC /////////
        //////////////////////////////////////////////////

        // correcting another bug: giving the godfather gangmember role
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        vm.startPrank(godFather);
        vm.expectRevert(); // panic: arithmetic underflow or overflow || MoneyVault::withdrawUSDC
        laundrette.withdrawMoney(godFather, godFather, savedFunds);
        vm.stopPrank();
    }
}
```
</details>



## Impact
USDC will be stuck in `MoneyVault`, even the godfather cannot withdraw it.

## Tools Used
Manual review, Foundry.

## Recommendations
Modify `MoneyVault` as follows:

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";

import { Kernel, Keycode } from "src/Kernel.sol";

import { Shelf } from "src/modules/Shelf.sol";

// Emergency money vault to store USDC and

contract MoneyVault is Shelf {
    IERC20 private usdc;
    CrimeMoney private crimeMoney;

    constructor(Kernel kernel_, IERC20 _usdc, CrimeMoney _crimeMoney) Shelf(kernel_) {
        usdc = _usdc;
        crimeMoney = _crimeMoney;
    }

    function KEYCODE() public pure override returns (Keycode) {
        return Keycode.wrap("MONEY");
    }

    function depositUSDC(address, address, uint256) external pure {
        revert("MoneyVault: depositUSDC is disabled");
    }

    function withdrawUSDC(address account, address to, uint256 amount) external {
        require(to == kernel.executor(), "MoneyVault: only GodFather can receive USDC");
-      withdraw(account, amount);
        crimeMoney.burn(account, amount);
        usdc.transfer(to, amount);
    }
}
```
## <a id='H-02'></a>H-02. Front-running deposits in `Laundrette::depositTheCrimeMoneyInATM`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L60-L62

## Summary
`Laundrette::depositTheCrimeMoneyInATM`allows front-running attacks where malicious gangmembers can steal funds from honest depositors.

## Vulnerability Details

Depositing USDC funds in the protocol is a two-step process:
- user needs to approve the `MoneyShelf` contract to spend USDC on its behalf, and then
- user needs to call `Laundrette::depositTheCrimeMoneyInATM` to make the actual deposit.

However, the current implementation of `Laundrette::depositTheCrimeMoneyInATM` allows front-running attacks: 
seeing the approval transaction from another user, a malicious gangmember can quickly call the `Laundrette::depositTheCrimeMoneyInATM` to deposit the victim's funds into their own account, instead of the indended account.

The test below demonstartes that 
- `gangmember_1` tries to deposit USDC for himself, but after he approves the `MoneyShelf` contract, 
- `gangmember_2` intervenes and calls `Laundrette::depositTheCrimeMoneyInATM` with input parameters `account = gangmember_1` and `to = gangmember_2`, essentially stealing funds from `gangmember_1`.

<details>
<summary>Proof of Code</summary>

```javascript
    function testFrontRunDeposits() public {
        // grant Godfathar gangmember status - @note this method hides another error in the code
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        // Godfather adds 2 gangmembers
        address gangmember_1 = makeAddr("gangmember_1");
        address gangmember_2 = makeAddr("gangmember_2");
        vm.startPrank(godFather);
        laundrette.addToTheGang(gangmember_1);
        laundrette.addToTheGang(gangmember_2);
        vm.stopPrank();

        // Godfather sends bonus in USDC to gangmember_1
        uint256 bonus = 1e12;
        vm.prank(godFather);
        usdc.transfer(gangmember_1, bonus);

        // gandmember_1 prepares to deposit
        vm.prank(gangmember_1);
        usdc.approve(address(moneyShelf), bonus);

        // gangmember_2 sees the approval, frontruns gangmember_1 and deposits the bonus on his balance
        vm.prank(gangmember_2);
        laundrette.depositTheCrimeMoneyInATM(gangmember_1, gangmember_2, bonus);

        // now gangmember_2 has the bonus
        assertEq(moneyShelf.getAccountAmount(gangmember_1), 0);
        assertEq(moneyShelf.getAccountAmount(gangmember_2), bonus);
    }
```
</details>

## Impact
Malicious gangmembers can steal funds from honest depositors.


## Tools Used
Manual review, Foundry.

## Recommendations
Ensure that no front-running is possible by modifying the code as follows:

```diff
-    function depositTheCrimeMoneyInATM(address account, address to, uint256 amount) external {
+    function depositTheCrimeMoneyInATM(address to, uint256 amount) external {
        moneyShelf.depositUSDC(msg.sender, to, amount);
    }
```
## <a id='H-03'></a>H-03. Risk of blacklisting by the maganer of the `USDC` contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L68-L78

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L60-L62

## Summary
Any and all addresses related to the Mafia can be blacklisted in the `USDC` contract. Blacklisted Mafia addresses cannot transfer or receive funds in USDC.

## Vulnerability Details
The USDC contract is centralized and includes a mechanism for blacklisting addresses, primarily controlled by Centre, the consortium founded by Circle and Coinbase. This blacklisting capability allows Centre to comply with regulatory and law enforcement requirements to prevent illegal activities.

Centre can be ordered to blacklist any and all addresses related to the Mafia by
- regulatory agencies,
- law enforcement agencies,
- court orders.

Should an address be added to the blacklist, that address cannot receive or transfer USDC funds anymore (or until it is removed from the blacklist).

This is demonstrated by the following test:

<details>
<summary>Proof of Code </summary>

```javascript
    function testUsdcBlacklist() public {
        ///////////////////
        ///// SETUP ///////
        ///////////////////

        // STEP 1: deploy mock usdc contract with blacklist functionality
        address centre = makeAddr("centre");
        vm.prank(centre);
        RealMockUSDC rusdc = new RealMockUSDC();
        uint256 initialRealUsdcBalance = 1_000_000e6; // from the mock contract

        // STEP 2: redeploy MoneyShelf with the correct USDC == realUsdc
        MoneyShelf realMoneyShelf = new MoneyShelf(kernel, rusdc, crimeMoney);

        // STEP 3: replace MoneyShelf module with realMoneyShelf module
        vm.prank(godFather);
        kernel.executeAction(Actions.UpgradeModule, address(realMoneyShelf));

        // STEP 4: deactivate then re-activate the policy
        // needed so that the configureDependencies function is called again to set the new MoneyShelf address.
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
        vm.stopPrank();

        // STEP 5: grantRole to realMoneyShelf
        vm.prank(kernel.admin()); // @note supposed to be the godFather (?) but in reality it is the laundrette
        kernel.grantRole(Role.wrap("moneyshelf"), address(realMoneyShelf));

        // STEP 5: Godfather acquires USDC - in practice, not from Circle :)
        vm.prank(centre);
        rusdc.transfer(godFather, initialRealUsdcBalance);

        // STEP 6: initial balance checks
        assertEq(rusdc.balanceOf(godFather), initialRealUsdcBalance);
        assert(crimeMoney.balanceOf(godFather) == 0);

        // STEP 7: Godfather deposits hald of his USDC
        vm.startPrank(godFather);
        uint256 depositAmount = initialRealUsdcBalance / 2;
        rusdc.approve(address(realMoneyShelf), depositAmount);
        laundrette.depositTheCrimeMoneyInATM(godFather, godFather, depositAmount);

        assertEq(rusdc.balanceOf(godFather), depositAmount);
        assertEq(realMoneyShelf.getAccountAmount(godFather), depositAmount);

        ///////////////////
        //// EXECUTION ////
        ///////////////////

        address alternativeAddress = makeAddr("alternativeAddress");

        // STEP 1: Circle blacklists Godfather and RealMoneyShelf
        vm.startPrank(centre);
        rusdc.addToBlacklist(godFather);
        rusdc.addToBlacklist(address(realMoneyShelf));
        vm.stopPrank();

        ///////////////////
        /// ASSERTIONS ////
        ///////////////////

        bytes memory expectedRevertReason = abi.encodeWithSignature("AddressBlacklisted(address)", godFather);

        // Godfather cannot transfer his USDC
        vm.prank(godFather);
        vm.expectRevert(expectedRevertReason);
        rusdc.transfer(alternativeAddress, depositAmount); // assume Godfather controls alternativeAddress

        // others cannot transfer from Godfather even if approved
        vm.prank(godFather);
        rusdc.approve(alternativeAddress, depositAmount);
        vm.prank(alternativeAddress);
        vm.expectRevert(expectedRevertReason);
        rusdc.transferFrom(godFather, alternativeAddress, depositAmount);

        // no one can deposit to (or witdraw from) realMoneyShelf
        vm.startPrank(alternativeAddress);
        expectedRevertReason = abi.encodeWithSignature("AddressBlacklisted(address)", realMoneyShelf);
        vm.expectRevert(expectedRevertReason);
        laundrette.depositTheCrimeMoneyInATM(alternativeAddress, alternativeAddress, depositAmount);
        vm.stopPrank();
    }
```

</details>

<details>
<summary>Mock USDC contract with blacklisting functionality</summary>

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { console } from "lib/forge-std/src/console.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

error AddressBlacklisted(address account);

contract RealMockUSDC is ERC20 {
    address public immutable i_owner;
    mapping(address => bool) private _blacklist;

    ////////////////////////////////////////////
    /////////////// Modifiers //////////////////
    ////////////////////////////////////////////

    modifier onlyOwner() {
        require(msg.sender == i_owner, "Not the owner!");
        _;
    }

    modifier notBlacklisted(address account) {
        if (_blacklist[account]) {
            revert AddressBlacklisted(account);
        }
        _;
    }

    ////////////////////////////////////////////
    /////////////// Events /////////////////////
    ////////////////////////////////////////////

    event Blacklisted(address indexed account);
    event Whitelisted(address indexed account);

    ////////////////////////////////////////////
    /////////////// Functions //////////////////
    ////////////////////////////////////////////

    constructor() ERC20("RUSDC", "RUSDC") {
        i_owner = msg.sender;
        _mint(msg.sender, 1_000_000e6);
    }

    function decimals() public view override returns (uint8) {
        return 6;
    }

    ////////////////////////////////////////////
    /////// Blacklisting functionality /////////
    ////////////////////////////////////////////

    function addToBlacklist(address account) external onlyOwner {
        _blacklist[account] = true;
        emit Blacklisted(account);
    }

    function removeFromBlacklist(address account) external onlyOwner {
        _blacklist[account] = false;
        emit Whitelisted(account);
    }

    ////////////////////////////////////////////
    /////// Transfer functions /////////////////
    ////////////////////////////////////////////

    function transfer(
        address recipient,
        uint256 amount
    )
        public
        override
        notBlacklisted(msg.sender)
        notBlacklisted(recipient)
        returns (bool)
    {
        return super.transfer(recipient, amount);
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    )
        public
        override
        notBlacklisted(sender)
        notBlacklisted(recipient)
        returns (bool)
    {
        return super.transferFrom(sender, recipient, amount);
    }
}

```


</details>

## Impact
Any and all addresses related to the Mafia can be blacklisted on the `USDC` contract.
Blacklisted addresses cannot receive or transfer funds in USDC.
All funds the Mafia keeps in USDC can be frozen.

## Tools Used
Manual review, Foundry.

## Recommendations
If illegal activities are a part of your "business model", do not keep any funds in USDC. Consider using other, less centralied stablecoins instead, and rewrite your protocol accordingly. Some suggestions:

- DAI (DAI):

-- Decentralized Issuance: DAI is a decentralized stablecoin issued by the MakerDAO protocol. It is governed by a decentralized autonomous organization (DAO) rather than a central entity.

-- Collateral-backed: DAI maintains its peg to the USD through a system of collateralized debt positions (CDPs) backed by various cryptocurrencies.

-- Governance: While DAI has a governance body, decisions are made through community voting involving MKR token holders, reducing the risk of unilateral blacklisting.

- LUSD (LUSD):

-- Fully Decentralized: LUSD is issued by the Liquity protocol, which operates without a governance token, minimizing centralized control.

-- ETH-backed: LUSD is collateralized exclusively by ETH, enhancing its decentralization.

-- No Blacklisting: The protocol is designed to be fully decentralized, with no mechanisms for blacklisting or censorship.
## <a id='H-04'></a>H-04. Omitted policy reconfig during module upgrade due to misassigned `dependencies` array in `Laundrette::configureDependencies`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L23

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L26

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/Kernel.sol#L247

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/Kernel.sol#L329-L340

## Summary
`Laundrette::configureDependencies` contains a bug (misassigned `dependencies` array) that results in the omission of `Laundrette`'s policy reconfiguration when the module associated with the `MONEY` `keycode` is updated. `Laundrette` will keep working on the old module instead of the new one.

## Vulnerability Details
`Laundrette::configureDependencies` is a function called by the `Kernel` when the `Laundrette` policy is activated or reconfigured. This function is supposed to ensure that the policy's dependencies on specific modules are properly set up. 

However, these dependencies are not set correctly. `dependencies` in `Laundrette` is supposed to be a 2-element array containing the 2 different keycodes of the 2 modules the policy depends on, but in practice the first element (`dependencies[0]`) is overwritten by the `WEAPN` `KeyCode` and the second one is left empty:

```javascript
    function configureDependencies() external override onlyKernel returns (Keycode[] memory dependencies) {
        dependencies = new Keycode[](2);

@>        dependencies[0] = toKeycode("MONEY");
        moneyShelf = MoneyShelf(getModuleAddress(toKeycode("MONEY")));

@>        dependencies[0] = toKeycode("WEAPN");
        weaponShelf = WeaponShelf(getModuleAddress(toKeycode("WEAPN")));
    }
```



## Impact
In short, 
- the policy (`laundrette`) will not be reconfigured when `moneyShelf` is upgraded to `moneyVault`, and consequently
- `laundrette`'s internal references to `moneyShelf` be not be updated, it  will keep using the `moneyShelf` module that was supposed to be upgraded. 

Notably, the impact of all this can be contained if, after the migration, the godfather (as `kernel`'s `executor`) deactivates and then reactivates the policy:

```javacript
... module update

        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
```

<details>
<summary>Longer explanation</summary>

To determine the impact, we need to consider where and how the incorrectly set up `dependencies` array is used. It is passed down as a return value to `Kernel` which then uses it to define 2 mappings:

```javascript
    function _activatePolicy(Policy policy_) internal {
        if (policy_.isActive()) revert Kernel_PolicyAlreadyApproved(address(policy_));

        // Grant permissions for policy to access restricted module functions
        Permissions[] memory requests = policy_.requestPermissions();
        _setPolicyPermissions(policy_, requests, true);

        // Add policy to list of active policies
        activePolicies.push(policy_);
        getPolicyIndex[policy_] = activePolicies.length - 1;

        // Record module dependencies
        Keycode[] memory dependencies = policy_.configureDependencies();
        uint256 depLength = dependencies.length;

        for (uint256 i; i < depLength;) {
            Keycode keycode = dependencies[i];

@>            moduleDependents[keycode].push(policy_);
@>            getDependentIndex[keycode][policy_] = moduleDependents[keycode].length - 1;

            unchecked {
                ++i;
            }
        }
```

These mappings will obviously be incorrect (incomplete), but the real impact unfolds when `Kernel` tries to use these mappings:

<details>
<summary>Impact of the incorrect `moduleDependents` </summary>

- Impact - where & when:
apart from `Kernel::_activatePolicy` (which uses the incorrect `dependencies` array from `Laundrette` to set up the mappings), it is only used by `Kernel::_reconfigurePolicies` which, in turn, is only relied on by `Kernel::_upgradeModule`. 

- Impact - what:
Considering the code of `Kernel::_reconfigurePolicies` below, it can be seen that if the  `MONEY` keycode (that has never been added to `moduleDependents`, i.e. as a dependency to the policy) is provided as an input parameter (happens when the module associated with the `MONEY` `keycode` is being upgraded), then  

-- `depLength` will be `0` and `configureDependencies` will never be called on the policy, so

--  policy reconfiguration never happens, 

-- `Laundrette`s internal references to the dependencies are not updated and, hence, 

-- it will keep working with the old `moneyShelf` module.

```javascript
    function _reconfigurePolicies(Keycode keycode_) internal {
        Policy[] memory dependents = moduleDependents[keycode_];
        uint256 depLength = dependents.length;

        for (uint256 i; i < depLength;) {
            dependents[i].configureDependencies();

            unchecked {
                ++i;
            }
        }
    }
```

Notably, the impact of all this can be contained if, after the migration, the godfather (as `kernel`'s `executor`) deactivates and then reactivates the policy.

</details>



<details>
<summary>Impact of the incorrect `getDependentIndex` array</summary>

- Impact -  where & when: `getDependentIndex` is only used by `Kernel::_pruneFromDependents` which, in turn, can only be called by `Kernel::_deactivatePolicy`.

- Impact - what: none. 

</details>
</details>

Place the following test to `Laundrette.t.sol` to demonstrate that
- the policy is not reconfigured after an update from module `moneyShelf` to `moneyVault`,
- the policy keeps working on `moneyShelf` and not on `moneyVault` after the upgrade,
- the issue can be mitigated if the godfather deactivates the policy than activates it again.

<details>
<summary>Proof of Code</summary>

```javascript

    // @notice import: import { Policy } from "src/Kernel.sol";
    // @notice add: import { EmergencyMigration } from "script/EmergencyMigration.s.sol";
    // @notice add: import { MoneyVault } from "src/modules/MoneyVault.sol";
    // @notice add: import { Keycode } from "src/Kernel.sol";
    function testMissedPolicyReconfig() public {
        /////////////////////////////////////////////////////////
        /// SETUP - GODFATHER DEPOSITS TO MONEYSHELF  ///////////
        /////////////////////////////////////////////////////////

        uint256 depositAmount = 1e12;

        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        vm.startPrank(godFather);
        usdc.approve(address(moneyShelf), depositAmount);
        laundrette.depositTheCrimeMoneyInATM(godFather, godFather, depositAmount);
        vm.stopPrank();

        /////////////////////////////////////////////////////////
        /// CHECK THAT BOTH KEXCODES ARE RECOGNIZED BY KERNEL ///
        /////////////////////////////////////////////////////////

        Keycode keycode_0 = kernel.allKeycodes(0);
        string memory keycodeStr_0 = bytes5ToString(Keycode.unwrap(keycode_0));
        assertEq(keycodeStr_0, "WEAPN"); // because WEAPONSHELF was installed first

        Keycode keycode_1 = kernel.allKeycodes(1);
        string memory keycodeStr_1 = bytes5ToString(Keycode.unwrap(keycode_1));
        assertEq(keycodeStr_1, "MONEY"); // because MONEYSHELF was installed second

        /////////////////////////////////////////////////////////
        /// CHECK WHICH POLICY DEPEND ON KEYCODE "WEAPN" ////////
        /////////////////////////////////////////////////////////

        Policy policy = kernel.moduleDependents(keycode_0, 0);
        assertEq(address(policy), address(laundrette));

        /////////////////////////////////////////////////////////
        /// CHECK WHICH POLICY DEPEND ON KEYCODE "WEAPN" ////////
        /////////////////////////////////////////////////////////

        vm.expectRevert(); // laundrette policy appears to not depend on the MONEY keycode
        policy = kernel.moduleDependents(keycode_1, 0);

        /////////////////////////////////////////////////////////
        ///////////////// MODULE UPGRADE ////////////////////////
        /////////////////////////////////////////////////////////

        // initially, Kernel recognizes moneyShelf as the module associated with keycode MONEY - OK
        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyShelf));

        EmergencyMigration migration;
        migration = new EmergencyMigration();
        MoneyVault moneyVault = migration.migrate(kernel, usdc, crimeMoney); // migration

        // now Kernel recognizes moneyShelf as the module associated with keycode MONEY - OK
        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyVault));

        /////////////////////////////////////////////////////////
        ///////// PROOF THAT POLICY HAS NOT BEEN UPDATED ////////
        /////////////////////////////////////////////////////////

        uint256 withdrawAmount = depositAmount / 2;
        uint256 initialMoneyShelfBalance = usdc.balanceOf(address(moneyShelf));
        uint256 endingMoneyShelfBalance;

        vm.prank(godFather);
        laundrette.withdrawMoney(godFather, godFather, withdrawAmount);

        endingMoneyShelfBalance = usdc.balanceOf(address(moneyShelf));

        // initial and ending moneyShelf balances are not the same
        // --> laundrette.withdrawMoney worked on moneyShelf, not moneyVault
        assertLt(endingMoneyShelfBalance, initialMoneyShelfBalance);

        /////////////////////////////////////////////////////////
        ///////// THE ISSUE CAN BE MITIGATED ////////////////////
        /////////////////////////////////////////////////////////

        // setup a non-zero balance for moneyVault
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette)); // mitigation step1
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette)); // mitigation step2

        // expect laundrette.withdrawMoney to work on moneyVault, but it has no balance
        // so trx will revert with artihmetic under/overflow
        vm.expectRevert();
        laundrette.withdrawMoney(godFather, godFather, withdrawAmount);
        vm.stopPrank();

        // the remaining balance of moneyShelf did not change -> 2nd laundrette.withdrawMoney did not work on moneyShelf
        assertEq(usdc.balanceOf(address(moneyShelf)), depositAmount / 2);
    }
```
</details>

## Tools Used
Manual review, Foundry.

## Recommendations

Correct `Laundrette::configureDependencies` as follows:

```diff
    function configureDependencies() external override onlyKernel returns (Keycode[] memory dependencies) {
        dependencies = new Keycode[](2);

        dependencies[0] = toKeycode("MONEY");
        moneyShelf = MoneyShelf(getModuleAddress(toKeycode("MONEY")));

-      dependencies[0] = toKeycode("WEAPN");
+     dependencies[0] = toKeycode("WEAPN");
        weaponShelf = WeaponShelf(getModuleAddress(toKeycode("WEAPN")));
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Funds are not transferred to `MoneyVault` during the migration            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/script/EmergencyMigration.s.sol#L13-L39

## Summary
When the `MoneyShelf` module is upgraded from `MoneyShelf` to `MoneyVault`, the USDC funds are not moved to `MoneyVault`.
This should be handled by `EmergencyMigration::migrate`, but it misses to move the funds.

## Vulnerability Details

The following test demonstrates that after migration, the USDC funds are still in `MoneyShelf`, not `MoneyVault`:

<details>
<summary>Proof of Code</summary>

```javascript
    function testMoneyNotMigratedDuringEmergencyMigration() public {
        ////////////////////////////////////////////////
        /// Deposit in original setup, to MoneyShelf ///
        ////////////////////////////////////////////////

        uint256 depositAmount = 1e12;

        vm.startPrank(godFather);
        usdc.approve(address(moneyShelf), depositAmount);
        laundrette.depositTheCrimeMoneyInATM(godFather, godFather, depositAmount);
        vm.stopPrank();

        assertEq(moneyShelf.getAccountAmount(godFather), depositAmount);

        ////////////////////////////////////////////////
        /// Emergency migration ////////////////////////
        ////////////////////////////////////////////////

        EmergencyMigration migration;

        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyShelf));

        migration = new EmergencyMigration();
        MoneyVault moneyVault = migration.migrate(kernel, usdc, crimeMoney);

        // check upgade - OK
        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyVault));

        // correcting another bug
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
        vm.stopPrank();

        // correcting another bug
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        // correcting yet another bug
        vm.startPrank(godFather);
        kernel.executeAction(Actions.ChangeAdmin, godFather);
        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyVault));
        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
        vm.stopPrank();

        // check roles - OK
        assertEq(kernel.hasRole(address(moneyShelf), Role.wrap("moneyshelf")), true);

        ////////////////////////////////////////////////
        /// Post-migration checks and actions //////////
        ////////////////////////////////////////////////

        // check balance after migration
        assertEq(moneyShelf.getAccountAmount(godFather), depositAmount);
        assertEq(usdc.balanceOf(address(moneyShelf)), depositAmount); // moneyShelf still has the deposits
        assertEq(usdc.balanceOf(address(moneyVault)), 0); // moneyVault has no balance

        // cannot withdraw from MoneyVault
        vm.prank(godFather);
        vm.expectRevert();  // arithmetic underflow or overflow
        laundrette.withdrawMoney(godFather, godFather, depositAmount);
    }
```
</details>

## Impact
USDC funds will remain in the `MoneyShelf`. This makes the whole migration pointless.

The only way to recover the funds is to upgrade the module `MoneyVault` back to the original `MoneyShelf` - provided that `MoneyShelf` is not already blacklisted by authorities or emptied/controlled by another gang.

## Tools Used
Manual reivew, Foundry.

## Recommendations
Ensure that USDC funds are moved from the `MoneyShelf` contract to `MoneyVault`. 
For this, the godfather needs to

1. withdraw every user's USDC deposit from `MoneyShelf` by repeatedly calling `Laundrette::withdrawMoney`;
2. transfer the USDC directly to `MoneyVault` (using the standard ERC20 function `transfer`)
## <a id='M-02'></a>M-02. `MoneyVault` lacks `moneyShelf` role, `MoneyVault::withdrawUSDC` will fail            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/script/EmergencyMigration.s.sol#L28-L38

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/CrimeMoney.sol#L23

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/src/modules/MoneyVault.sol#L33

## Summary
`EmergencyMigration` misses to assign the `moneyShelf` role to `MoneyVault`. 
Without the role, `MoneyVault` will not be able to call `CrimeMoney::burn` (or `CrimeMoney::mint`) and, conseqently, USDC withdrawal from `MoneyVault` will fail.

Consider `MoneyVault::withdrawUSDC` which requires `MoneyVault` to be able to call CrimeMoney::burn` for successful execution:

```javascript
    function withdrawUSDC(address account, address to, uint256 amount) external {
        require(to == kernel.executor(), "MoneyVault: only GodFather can receive USDC");
        withdraw(account, amount);
@>        crimeMoney.burn(account, amount);
        usdc.transfer(to, amount);
    }
```

The following test demonstrates that
- `MoneyVault` does not have the `moneyShelf` role after migration, and that
- without this role, a call to `MoneyVault::withdrawUSDC` will revert with `MoneyVault: only GodFather can receive USDC`:

_Note! For this test to properly work, uncomment the following line in `MoneyVault`, as that is another bug:_

```javascript
        withdraw(account, amount);
```

<details>
<summary>Proof of Code</summary>

```javascript
    function testMoneyVaultIsNotAssignedMoneyShelfRoleCantWithdraw() public {
        ////////////////////////////////////////////////
        /// Emergency migration ////////////////////////
        ////////////////////////////////////////////////

        EmergencyMigration migration;

        assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyShelf));

        migration = new EmergencyMigration();
        MoneyVault moneyVault = migration.migrate(kernel, usdc, crimeMoney);

        // mitigating the effects of another bug
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
        vm.stopPrank();

        // MoneyVault does not have moneyShelf role
        assertEq(kernel.hasRole(address(moneyVault), Role.wrap("moneyshelf")), false);

        //////////////////////////////////////////////////
        /////// SETUP OF BALANCES ////////////////////////
        //////////////////////////////////////////////////

        // suppose USDC funds have been transferred to moneyVault
        uint256 savedFunds = 1e6; // i.e. total supply of crimeMoney
        deal(address(usdc), address(moneyVault), savedFunds);

        // suppose godfather recovered crimeMoney for withdrawal
        uint256 crimeFunds = savedFunds;
        deal(address(crimeMoney), godFather, crimeFunds);

        //////////////////////////////////////////////////
        /////// GODFATHER TRIES TO WITHDRAW USDC /////////
        //////////////////////////////////////////////////

        // correcting another bug: giving the godfather gangmember role
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        vm.startPrank(godFather);
        vm.expectRevert("CrimeMoney: only MoneyShelf can mint");
        laundrette.withdrawMoney(godFather, godFather, savedFunds);
        vm.stopPrank();
    }
```
</details>

## Impact
While `MoneyVault` lacks the `moneyShelf` role, all USDC funds of the Mafia will be stuck in `MoneyVault`.

Luckily, the godfather can mitigate the impact by performing the following steps:

- Reclaim the `admin` rights to `Kernel` by executing

```javascript
        kernel.executeAction(Actions.ChangeAdmin, godFather);
```

Note, however, that while he is the `admin`, `Laundrette::addToTheGang` and `Laundrette::quitTheGang` will not work.

- Grant the `moneyShelf` role to the deployed instance of `MoneyVault` by executing

```javascript
        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyVault));
```

- (and then he should give back the admin role to `Laundrette` by executing)

```javascript
        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
```

## Tools Used
Manual review, Foundry.

## Recommendations

The easiest solution is to deploy the `MoneyVault` contract together with the other contracts, in `Deploy.s.sol`, and grant `MoneyVault` the `moneyShelf` role while the godfather is still the `admin` of `Kernel`.

Alternatively, modify `EmergencyMigration` so that during the migration the `moneyShelf` role is granted to `MoneyVault`.


## <a id='M-03'></a>M-03. Any gangmember can kick out any other gangmember from the gang via `Laundrette::quitTheGang`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L87-L91

## Summary
Any gangmember can kick out any other gang member from the gang by specifying that other gangmember's address as an input parameter to `Laundrette::quitTheGang`.

## Vulnerability Details

`Laundrette::quitTheGang` is supposed to allow:
- gangmembers to quit the gang, and
- the godfather to kick out any gangmember if he sees fit.

However, an incorrect implementation allows not only the godfather but any gangmember to kick out any other gangmembers from the gang.

This is demonstrated by the following test:

<details>
<summary>Proof of Code</summary>

```javascript
    function testGangMemberCanKickOtherMemberOut() public {
        // grant Godfathar gangmember status - @note this method hides another error in the code
        vm.prank(kernel.admin());
        kernel.grantRole(Role.wrap("gangmember"), godFather);

        // Godfather adds 2 gangmembers
        address gangmember_1 = makeAddr("gangmember_1");
        address gangmember_2 = makeAddr("gangmember_2");
        vm.startPrank(godFather);
        laundrette.addToTheGang(gangmember_1);
        laundrette.addToTheGang(gangmember_2);
        vm.stopPrank();

        assertEq(kernel.hasRole(gangmember_1, Role.wrap("gangmember")), true);
        assertEq(kernel.hasRole(gangmember_2, Role.wrap("gangmember")), true);

        // gangmember_1 kicks out gangmember_2
        vm.prank(gangmember_1);
        laundrette.quitTheGang(gangmember_2);

        assertEq(kernel.hasRole(gangmember_2, Role.wrap("gangmember")), false);
    }
```
</details>


## Impact

Any gangmember can kick out any other gangmember (including the godfather) from the gang.

## Tools Used
Manual review, Foundry.

## Recommendations
To ensure that only the godfather can kick gang members out and that gangmembers can quit themselves, modify `Laundrette` as follows:

```diff
-    function quitTheGang(address account) external onlyRole("gangmember") {
-        kernel.revokeRole(Role.wrap("gangmember"), account);
-    }

+    function quitTheGang() external onlyRole("gangmember") {
+        kernel.revokeRole(Role.wrap("gangmember"), msg.sender);
+    }

+    function kickGangmemberOut(address member) external isGodFather {
+        kernel.revokeRole(Role.wrap("gangmember"), member);
+    }
```
## <a id='M-04'></a>M-04. `CrimeMoney` token is not a stablecoin            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/CrimeMoney.sol#L1C1-L36C2

## Summary

The `CrimeMoney` token is not a real stablecoin.

- Essentially, `CrimeMoney adds another layer of centralization on top of the USDC token that is already centralized in itself.
- A token issued and controlled by a crime organization is going to be targeted by law enforcement bodies.
- USDC is not always redeemable for `CrimeMoney`: 

-- external users can deposit on their USDC to get `CrimeMoney` tokens themselves, but cannot withdraw that USDC with their `CrimeMoney` tokens;

-- gangmembers can withdraw their depositied USDC with their `CrimeMoney` tokens as long as they do not quit the gang or the godfather does not kick them out from the gang;

-- if Centre, the governing body of the `USDC` contract blacklists the address holding the depositied USDC tokens or the addresses of gangmembers, withdrawal will not work.

With so much centralization, trust issues, and associated risks - especially without any advantage or value proposition over USDC -, the `CrimeMoney` token will have little to no adoption and abismal liquidity.

Considering these, it is apparent from the very beginning that the `CrimeMoney` token surely cannot fulfill the 3 functions of money:
- storage of value
- unit of account
- medium of exchange

## Vulnerability details

1. Centralization and Legal Risks

The centralization of `CrimeMoney` by a criminal organization makes it highly susceptible to legal action and regulatory scrutiny. Authorities can easily target and dismantle such centralized entities, freezing or seizing assets.

2. Trust and Adoption

Trust is a fundamental component of any currency, especially stablecoins. Given the dubious nature of `CrimeMoney` and its issuer, it is unlikely that users or other platforms will trust and adopt it. Without broad acceptance, its utility and value are severely limited.

3. Redemption Issues

The inability for all users to redeem USDC with `CrimeMoney` tokens undermines its claim as a stablecoin. True stablecoins must be easily redeemable for their pegged asset to maintain their value. The selective redemption rights create an inherent unfairness and reduce the token's reliability.

4. Blacklisting by Centre

The potential for addresses to be blacklisted by Centre adds another layer of risk. If critical addresses are blacklisted, it can freeze significant amounts of funds and disrupt the functionality of the entire system. This risk further erodes trust and reliability.

5. Lack of Clear Use Case

There is no clear advantage or unique value proposition for using `CrimeMoney` over directly using USDC. The additional complexity and risk introduced by `CrimeMoney` do not provide any benefits to justify its existence.

## Recommendations

Reconsider the need for the `CrimeMoney` token and its purpose.

# Low Risk Findings

## <a id='L-01'></a>L-01. Godfather cannot access functions reserved for gangmembers, as `Deployer` does not grant him the role            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L68-L78

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L80-L82

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L84-L86

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L88-L90

## Summary
The godfather lacks the role `gangmember` and, hence, cannot access functions reserved for gangmembers:
- `Laundrette::withdrawMoney`,
- `Laundrette::takeGuns`,
- `Laundrette::addToTheGang`,
- `Laundrette::quitTheGang`

## Vulnerability Details
`Deployer` misses to assign the `gangmember` role to the godfather, who cannot assign the role to himself for the following reasons:
- `Laundrette::addToTheGang` is gated by the `onlyRole("gangmember")` modifier,
- `kernel.grantRole()` is gated by the `onlyAdmin` modifier, and the `admin` rights for `Kernel` are possessed by `Laundrette`, not by the godfather. As described in another finding, these rights cannot be reclaimed by the godfather.


## Impact
While in theory godfather is supposed to have all rights and control the protocol, in practice, he cannot access most of the functions:
- `Laundrette::withdrawMoney`,
- `Laundrette::takeGuns`,
- `Laundrette::addToTheGang`,
- `Laundrette::quitTheGang`

## Tools Used
Manual review, Foundry.

## Recommendations
The clearest solution is to grant the `gangmember` role for the godfather from the get-go, i.e. in `Deployer`:

```diff
    function deploy() public returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {
        godFather = msg.sender;

        // Deploy USDC mock
        HelperConfig helperConfig = new HelperConfig();
        IERC20 usdc = IERC20(helperConfig.getActiveNetworkConfig().usdc);

        Kernel kernel = new Kernel();
        CrimeMoney crimeMoney = new CrimeMoney(kernel);

        WeaponShelf weaponShelf = new WeaponShelf(kernel);
        MoneyShelf moneyShelf = new MoneyShelf(kernel, usdc, crimeMoney);
        Laundrette laundrette = new Laundrette(kernel);

        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyShelf));

        kernel.executeAction(Actions.InstallModule, address(weaponShelf));
        kernel.executeAction(Actions.InstallModule, address(moneyShelf));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));

+     kernel.grantRole(Role.wrap("gangmember"), godFather);


        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
        kernel.executeAction(Actions.ChangeExecutor, godFather);

        return (kernel, usdc, crimeMoney, weaponShelf, moneyShelf, laundrette);
    }
```


Additionally, consider removing the `onlyRole("gangmember")` modifier from `Laundrette::addToTheGang`, as the other modifier is stricter and does the job in itself:

```diff    
-    function addToTheGang(address account) external onlyRole("gangmember") isGodFather {
+    function addToTheGang(address account) external isGodFather {

        kernel.grantRole(Role.wrap("gangmember"), account);
    }
```
## <a id='L-02'></a>L-02. Discrepancy in the decimal places of `CrimeMoney` and `USDC` - peg is not 1:1            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/CrimeMoney.sol#L1-L26

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/modules/MoneyShelf.sol#L26-L27

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/modules/MoneyShelf.sol#L32-L33

## Summary
`CrimeMoney` and `USDC` tokens have different decimal precision which in not taken into account in the protocol.
Consequently, `CrimeMoney` is not pegged 1:1 to `USDC`.

## Vulnerability Details
Functions `MoneyShelf::depositUSDC`, `MoneyShelf::withdrawUSDC` and `MoneyVault::withdrawUSDC` mint and burn `CrimeMoney` like it had the same decimal precision as `USDC`, see e.g.

```javascript
    function depositUSDC(address account, address to, uint256 amount) external {
        deposit(to, amount);
        usdc.transferFrom(account, address(this), amount);
        crimeMoney.mint(to, amount);
    }
```

Here, the same amount (`amount`) is used in the `deposit` function and in the call to `crimeMoney.mint`.

However, in reality `USDC` has 6 decimal places, whereas `CrimeMoney` has 18.

The following test demonstrates that `CrimeMoney` is minted as it had the same decimal precision as the USDC token. Insert this test into `Laundrette.t.sol` and import the indicated dependencies:

<details>
<summary>Proof of Code</summary>

```javascript
    // @notice add: import { RealMockUSDC } from "./mocks/RealMockUSDC.sol";
    // @notice add: import { MoneyShelf } from "test/Base.t.sol";
    function testBrokenPeg() public {
        ///////////////////
        ///// SETUP ///////
        ///////////////////

        // STEP 1: deploy the decimal-correct usdc contract
        address centre = makeAddr("centre");
        vm.prank(centre);
        RealMockUSDC rusdc = new RealMockUSDC();
        uint256 initialRealUsdcBalance = 1_000_000e6; // from the mock contract

        // STEP 2: redeploy MoneyShelf with the correct USDC == realUsdc
        MoneyShelf realMoneyShelf = new MoneyShelf(kernel, rusdc, crimeMoney);

        // STEP 3: replace MoneyShelf module with realMoneyShelf module
        vm.prank(godFather);
        kernel.executeAction(Actions.UpgradeModule, address(realMoneyShelf));

        // STEP 4: deactivate then re-activate the policy
        // neccessary due to another bug, see
        // "Omitted policy reconfig during module upgrade due to misassigned dependencies..."
        vm.startPrank(godFather);
        kernel.executeAction(Actions.DeactivatePolicy, address(laundrette));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));
        vm.stopPrank();

        // STEP 5: grantRole to realMoneyShelf, necessary for mint and burn
        vm.prank(kernel.admin()); //
        kernel.grantRole(Role.wrap("moneyshelf"), address(realMoneyShelf));

        // STEP 6: Godfather acquires USDC
        vm.prank(centre);
        rusdc.transfer(godFather, initialRealUsdcBalance);

        // STEP 7: initial balance checks
        uint256 crimeMoneyBalance;
        assertEq(rusdc.balanceOf(godFather), initialRealUsdcBalance);
        assert(crimeMoney.balanceOf(godFather) == 0);

        ///////////////////
        //// EXECUTION ////
        ///////////////////

        vm.startPrank(godFather);
        // STEP 1: Godfather deposits all his USDC
        rusdc.approve(address(realMoneyShelf), initialRealUsdcBalance);
        laundrette.depositTheCrimeMoneyInATM(godFather, godFather, initialRealUsdcBalance);
        vm.stopPrank();

        assertEq(rusdc.balanceOf(godFather), 0);
        assertEq(realMoneyShelf.getAccountAmount(godFather), initialRealUsdcBalance);

        ///////////////////
        /// ASSERTIONS ////
        ///////////////////

        assert(rusdc.decimals() == 6); // 1 000 000 USDC == 1 USD
        assert(crimeMoney.decimals() == 18); // 1 000 000 crimeMoney == 1e-12 USD

        // total supply of the 2 tokens are the same
        // but considering the different decimal places, it should not be the same
        assert(rusdc.totalSupply() == crimeMoney.totalSupply());
    }
```
</details>

<details>
<summary>Mock USDC contract with 6 decimal places</summary>

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { console } from "lib/forge-std/src/console.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

error AddressBlacklisted(address account);

contract RealMockUSDC is ERC20 {
    address public immutable i_owner;
    mapping(address => bool) private _blacklist;

    ////////////////////////////////////////////
    /////////////// Modifiers //////////////////
    ////////////////////////////////////////////

    modifier onlyOwner() {
        require(msg.sender == i_owner, "Not the owner!");
        _;
    }

    modifier notBlacklisted(address account) {
        if (_blacklist[account]) {
            revert AddressBlacklisted(account);
        }
        _;
    }

    ////////////////////////////////////////////
    /////////////// Events /////////////////////
    ////////////////////////////////////////////

    event Blacklisted(address indexed account);
    event Whitelisted(address indexed account);

    ////////////////////////////////////////////
    /////////////// Functions //////////////////
    ////////////////////////////////////////////

    constructor() ERC20("RUSDC", "RUSDC") {
        i_owner = msg.sender;
        _mint(msg.sender, 1_000_000e6);
    }

    function decimals() public view override returns (uint8) {
        return 6;
    }

    ////////////////////////////////////////////
    /////// Blacklisting functionality /////////
    ////////////////////////////////////////////

    function addToBlacklist(address account) external onlyOwner {
        _blacklist[account] = true;
        emit Blacklisted(account);
    }

    function removeFromBlacklist(address account) external onlyOwner {
        _blacklist[account] = false;
        emit Whitelisted(account);
    }

    ////////////////////////////////////////////
    /////// Transfer functions /////////////////
    ////////////////////////////////////////////

    function transfer(
        address recipient,
        uint256 amount
    )
        public
        override
        notBlacklisted(msg.sender)
        notBlacklisted(recipient)
        returns (bool)
    {
        return super.transfer(recipient, amount);
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    )
        public
        override
        notBlacklisted(sender)
        notBlacklisted(recipient)
        returns (bool)
    {
        return super.transferFrom(sender, recipient, amount);
    }
}

```
</details>

## Impact
In the scope of the protocol, there is no real issue: gangmembers will be able to withdraw the same amount of USDC as they depositied.

However, protocols and businesses that integrate with the protocol expecting a stable 1:1 peg might find their integrations unreliable. For instance, a payment gateway that accepts `CrimeMoney` expecting it to be equivalent to `USDC` token might face significant issues in transaction processing and value reconciliation.


## Tools Used
Manual review, Foundry.

## Recommendations
Consider reserving the same amount of decimals for `MoneyShelf` as `USDC` does, i.e.:

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { Kernel, Actions, Role } from "./Kernel.sol";

contract CrimeMoney is ERC20 {
    Kernel public kernel;

    constructor(Kernel _kernel) ERC20("CrimeMoney", "CRIME") {
        kernel = _kernel;
    }

+    function decimals() public view override returns (uint8) {
+        return 6;
+    }

    modifier onlyMoneyShelf() {
        require(kernel.hasRole(msg.sender, Role.wrap("moneyshelf")), "CrimeMoney: only MoneyShelf can mint");
        _;
    }

    function mint(address to, uint256 amount) public onlyMoneyShelf {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) public onlyMoneyShelf {
        _burn(from, amount);
    }
}

```
## <a id='L-03'></a>L-03. Incorrect script setup in `Deployer.s.sol`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/script/Deployer.s.sol#L19

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/603ea5aa1b17c3c399502f5d8ba277396b9c70bd/script/Deployer.s.sol#L29

## Vulnerability Details

Deploy script `Deploy.s.sol` is set up incorrectly:

- ISSUE 1: the deployment of `HelperConfig` is broadcasted,
- ISSUE 2: `Deploy::run` does not return the addresses of the deployed contracts,
- ISSUE 3: It relies on a `HelperConfig` file that does not specify the correct USDC addresses on the different live chains

## Impact
- ISSUE 1: `HelperConfig` is also deployed on the blockchain, which is a waste of gas
- ISSUE 2: using `Deploy::run` will be impractical in tests,
- ISSUE 3: the protocol will not work if deployed to any of the specified live chains 

## Tools Used
Manual review, Foundry.

## Recommendations
Perform the following modifications:

- `Deployer.s.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import { Script, console2 } from "lib/forge-std/src/Script.sol";
import { IERC20 } from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import { HelperConfig } from "script/HelperConfig.s.sol";

import { Kernel, Actions, Role } from "src/Kernel.sol";
import { CrimeMoney } from "src/CrimeMoney.sol";

import { WeaponShelf } from "src/modules/WeaponShelf.sol";
import { MoneyShelf } from "src/modules/MoneyShelf.sol";

import { Laundrette } from "src/policies/Laundrette.sol";

contract Deployer is Script {
    address public godFather;
+  Kernel public kernel;
+  IERC20 public usdc;
+  CrimeMoney public crimeMoney;
+  WeaponShelf public weaponShelf;
+  MoneyShelf public moneyShelf;
+  Laundrette public laundrette;

-  function run() public {
+    function run() public returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {
+      // Instantiate HelperConfig outside the broadcast block
+      HelperConfig helperConfig = new HelperConfig();
+      usdc = IERC20(helperConfig.getActiveNetworkConfig().usdc);

        vm.startBroadcast();
-      deploy();
+     _deploy(IERC20 usdc);
        vm.stopBroadcast();

+      return (kernel, usdc, crimeMoney, weaponShelf, moneyShelf, laundrette);

    }

-    function deploy() public returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {
+   function _deploy(IERC20 usdc) internal returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {

        godFather = msg.sender;

-        // Deploy USDC mock
-        HelperConfig helperConfig = new HelperConfig();
-        IERC20 usdc = IERC20(helperConfig.getActiveNetworkConfig().usdc);

        Kernel kernel = new Kernel();
        CrimeMoney crimeMoney = new CrimeMoney(kernel);

        WeaponShelf weaponShelf = new WeaponShelf(kernel);
        MoneyShelf moneyShelf = new MoneyShelf(kernel, usdc, crimeMoney);
        Laundrette laundrette = new Laundrette(kernel);

        kernel.grantRole(Role.wrap("moneyshelf"), address(moneyShelf));

        kernel.executeAction(Actions.InstallModule, address(weaponShelf));
        kernel.executeAction(Actions.InstallModule, address(moneyShelf));
        kernel.executeAction(Actions.ActivatePolicy, address(laundrette));

        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
        kernel.executeAction(Actions.ChangeExecutor, godFather);

        return (kernel, usdc, crimeMoney, weaponShelf, moneyShelf, laundrette);
    }
}
```

- `HelperConfig.s.sol`

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { Script } from "lib/forge-std/src/Script.sol";
import { console2 } from "lib/forge-std/src/console2.sol";

import { Deployer } from "script/Deployer.s.sol";

import { MockUSDC } from "test/mocks/MockUSDC.sol";

contract HelperConfig is Script {
    /*//////////////////////////////////////////////////////////////
                                 ERRORS
    //////////////////////////////////////////////////////////////*/
    error HelperConfig__InvalidChainId();

    /*//////////////////////////////////////////////////////////////
                                 TYPES
    //////////////////////////////////////////////////////////////*/
    struct NetworkConfig {
        address usdc;
    }

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
    uint256 constant ETH_MAINNET_CHAIN_ID = 1;
    uint256 constant ETH_SEPOLIA_CHAIN_ID = 11_155_111;

    uint256 constant ZKSYNC_MAINNET_CHAIN_ID = 324;
    uint256 constant ZKSYNC_SEPOLIA_CHAIN_ID = 300;

    uint256 constant POLYGON_MAINNET_CHAIN_ID = 137;
    uint256 constant POLYGON_MUMBAI_CHAIN_ID = 80_001;

    // Local network state variables
    NetworkConfig public localNetworkConfig;
    mapping(uint256 chainId => NetworkConfig) public networkConfigs;

    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    constructor() {
        networkConfigs[ETH_MAINNET_CHAIN_ID] = getEthMainnetConfig();
-      networkConfigs[ETH_SEPOLIA_CHAIN_ID] = getZkSyncSepoliaConfig();
+     networkConfigs[ETH_SEPOLIA_CHAIN_ID] = getEthSepoliaConfig();
        networkConfigs[ZKSYNC_MAINNET_CHAIN_ID] = getZkSyncConfig();
        networkConfigs[ZKSYNC_SEPOLIA_CHAIN_ID] = getZkSyncSepoliaConfig();
        networkConfigs[POLYGON_MAINNET_CHAIN_ID] = getPolygonMainnetConfig();
        networkConfigs[POLYGON_MUMBAI_CHAIN_ID] = getPolygonMumbaiConfig();
    }

    function getConfigByChainId(uint256 chainId) public returns (NetworkConfig memory) {
        if (networkConfigs[chainId].usdc != address(0)) {
            return networkConfigs[chainId];
        } else {
            return getOrCreateAnvilEthConfig();
        }
    }

    function getActiveNetworkConfig() public returns (NetworkConfig memory) {
        return getConfigByChainId(block.chainid);
    }

    /*//////////////////////////////////////////////////////////////
                                CONFIGS
    //////////////////////////////////////////////////////////////*/
    function getEthMainnetConfig() public pure returns (NetworkConfig memory) {
-        return NetworkConfig({ usdc: address(1) });
+       return NetworkConfig({ usdc: address(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48) });
    }

+    function getEthSepoliaConfig() public pure returns (NetworkConfig memory) {
+       return NetworkConfig({ usdc: address(0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238) });
+    }

    function getZkSyncConfig() public pure returns (NetworkConfig memory) {
-        return NetworkConfig({ usdc: address(1) });
+       return NetworkConfig({ usdc: address(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4) });
    }

    function getZkSyncSepoliaConfig() public pure returns (NetworkConfig memory) {
-      return NetworkConfig({ usdc: address(1) });
+     return NetworkConfig({ usdc: address(0xAe045DE5638162fa134807Cb558E15A3F5A7F853) });

    }

    function getPolygonMainnetConfig() public pure returns (NetworkConfig memory) {
-      return NetworkConfig({ usdc: address(1) });
+     return NetworkConfig({ usdc: address(0x2791bca1f2de4661ed88a30c99a7a9449aa84174) });
    }

    function getPolygonMumbaiConfig() public pure returns (NetworkConfig memory) {
-      return NetworkConfig({ usdc: address(1) });
+     return NetworkConfig({ usdc: address(0xe6b8a5CF854791412c1f6EFC7CAf629f5Df1c747) });
    }

    /*//////////////////////////////////////////////////////////////
                              LOCAL CONFIG
    //////////////////////////////////////////////////////////////*/
    function getOrCreateAnvilEthConfig() public returns (NetworkConfig memory) {
        if (localNetworkConfig.usdc != address(0)) {
            return localNetworkConfig;
        }
        console2.log(unicode"⚠️ You have deployed a mock conract!");
        console2.log("Make sure this was intentional");
        vm.prank(Deployer(msg.sender).godFather());
        address mockUsdc = _deployMocks();

        localNetworkConfig = NetworkConfig({ usdc: mockUsdc });
        return localNetworkConfig;
    }

    /*
     * Add your mocks, deploy and return them here for your local anvil network
     */
    function _deployMocks() internal returns (address) {
        MockUSDC usdc = new MockUSDC();

        return address(usdc);
    }
}
```
## <a id='L-04'></a>L-04. Godfather cannot reclaim `Kernel` admin rights via `Laundrette::retrieveAdmin`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/policies/Laundrette.sol#L92-L95

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/script/Deployer.s.sol#L45

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/Kernel.sol#L190

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/a907bf76019afd657530a8e476fb59a60e9566e6/src/Kernel.sol#L177-L180

## Summary
Godfather cannot reclaim admin rights for the `Kernel` contract, as a call to `Laundrette::retrieveAdmin` will always fail.

## Vulnerability Details
By default, the `Kernel` contract recognizes two distinguished roles: `admin` and `executor`, both of which are set to the address of the deployer of `Kernel`:

```javascript
    constructor() {
        executor = msg.sender;
        admin = msg.sender;
    }
```

`admin`is later changed to the address of `Laundrette` by `Deployer::deploy`:

```javascript
        kernel.executeAction(Actions.ChangeAdmin, address(laundrette));
```

With this, Godfather loses access to functions `Kernel::grantRole` and `Kernel::revokeRole` which are both gated by the `onlyAdmin` modifier.

`Laundrette::retrieveAdmin` should supposedly allow Godfather to reclaim `admin` rights :

```javascript
    function retrieveAdmin() external {
        kernel.executeAction(Actions.ChangeAdmin, kernel.executor());
    }
```

However, a call to this function will always fail. The call chain would be as follows:

1. Godfather calls `Laundrette::retrieveAdmin`.
2. Laundrette calls `kernel.executeAction(Actions.ChangeAdmin, kernel.executor());`
3. `Kernel::executeAction`'s `onlyExecutor` modifier restricts its use to `msg.sender == executor`. 
4. Even though Godfather _is_ the `executor` and the originator of the call flow is him, in `Kernel`'s context the `msg.sender` is not the Godfather but `Laundrette`. 
(`msg.sender` is a dynamic variable and is always the address that _directly_ initiates a given call to a contract). 

The following piece of test demonstrates that
- Godfather is not the `admin` of `Kernel`
- He cannot reclaim `admin` rights via `Laundrette::retrieveAdmin`
- He can reclaim `admin` rights via a direct call to `Kernel::executeAction`
- By doing so, he breaks other `Laundrette` functionality like `Laundrette:addToGang` and `Laundrette::quitTheGang`

<details>
<summary>Proof of Code</summary>

```javascript
    function testGodfatherCannotReclaimKernelRightsViaLaundrette() public {
        // Godfather is not the Kernel admin
        assert(godFather != kernel.admin());
        // Laundrette contract is the Kernel admin
        assert(address(laundrette) == kernel.admin());
        // Godfather is the kernel executor
        assert(godFather == kernel.executor());

        // Godfather fails to reclaim admin rights via `Laundrette`, as Kernel recognizes `Laundrette˛the msg.sender
        vm.prank(godFather);
        bytes memory expectedRevertReason = abi.encodeWithSignature("Kernel_OnlyExecutor(address)", laundrette);
        vm.expectRevert(expectedRevertReason);
        laundrette.retrieveAdmin();

        // Godfather can reclaim admin rights via kernel.executeAction
        vm.startPrank(godFather);
        kernel.executeAction(Actions.ChangeAdmin, kernel.executor());
        vm.stopPrank();

        assert(godFather == kernel.admin());

        // But this breaks other Laundrette functionality
        vm.startPrank(godFather);
        // first grants himself gangmember role directly via a call to kernel
        kernel.grantRole(Role.wrap("gangmember"), godFather);
        expectedRevertReason = abi.encodeWithSignature("Kernel_OnlyAdmin(address)", laundrette);
        // kernel recognizes laundrette as the msg.sender which is not the admin anymore -> revert
        vm.expectRevert(expectedRevertReason);
        laundrette.quitTheGang(godFather);
        vm.stopPrank();
    }
```
</details>

## Impact

The impact is limited:

- although Godfather cannot reclaim `admin` rights to `Kernel` via Laundrette::retrieveAdmin`, he can still do so by executing       `kernel.executeAction(Actions.ChangeAdmin, godFather);`;
- under normal circumstances, Godfather should not be the `admin` anyways, because that breaks other `Laundrette` functionality like `Laundrette:addToGang` and `Laundrette::quitTheGang`. Probably the only meaningful use of this function would be for circumventing another bug (where Godfather has no `gangmember` role and cannot access `Laundrette` functions reserved for this role.

## Tools Used
Manual reivew, Foundry.

## Recommendations
Revisit the need for need for `Laundrette::retrieveAdmin`. If all other bugs are fixed, this function might not be needed:

```diff
-    function retrieveAdmin() external {
-        kernel.executeAction(Actions.ChangeAdmin, kernel.executor());
-    }
```


