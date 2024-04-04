# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Airdrop Exploitation by Waiting Unpaired Souls](#H-01)
    - ### [H-02. Staking Rewards Misaligned with Staking Period](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Users can become soulmates with themselves, can deplete protocol funds by generating a lot of self-paired soulmates](#M-01)
    - ### [M-02. Lack of Access Control in initVault Allows Front-Running](#M-02)
    - ### [M-03. Unpaired users can get divorced](#M-03)
- ## Low Risk Findings
    - ### [L-01. Staking Rewards Forfeited Upon Deposit Withdrawal](#L-01)
    - ### [L-02. Misleading Total Souls Count due to Unpaired and Self-Paired Users](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 3
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Airdrop Exploitation by Waiting Unpaired Souls            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L68

## Summary
`Airdrop::claim` allows unpaired participants, specifically those who have initiated the minting process but are not yet paired with a soulmate, to illegitimately access and claim love token rewards from the Airdrop Vault. This issue stems from a lack of adequate validation in the claim process, particularly failing to account for the state where `soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))` returns zero, indicative of a user in a waiting state for pairing. This loophole can lead to significant unauthorized claims, undermining the fairness and integrity of the reward distribution framework.

## Vulnerability Details
`Airdrop::claim` fails to account for users in a "waiting" state, identifiable when `soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))` returns zero. The protocol's logic intends for only paired users to access these rewards; however, due to this oversight, any user who has merely initiated a minting request can exploit this loophole to claim rewards, bypassing the intended eligibility criteria.

Proof of code:

```javascript

    function test_soulInWaitingCanStealFundsFromAirdropVault() public {
        uint256 balanceUser1;
        uint256 balanceAirdropVault;
        uint256 currentUnixEpoch = 1707922532;

        console2.log("timestamp: ", block.timestamp); // 1 by default on anvil
        vm.warp(block.timestamp + currentUnixEpoch);
        console2.log("timestamp: ", block.timestamp);

        vm.startPrank(user1);
        soulmateContract.mintSoulmateToken();
        airdropContract.claim();
        vm.stopPrank();

        balanceUser1 = loveToken.balanceOf(user1);
        balanceAirdropVault = loveToken.balanceOf(address(airdropVault));

        assertLt(0, balanceUser1);
        assertGt(500_000_000 ether, balanceAirdropVault);
        assertEq(balanceUser1 + balanceAirdropVault, 500_000_000 ether);
    }
```

## Impact
 Unauthorized claims by unpaired users can lead to the depletion of resources in the Airdrop Vault, intended for legitimately paired users, undermining trust in the protocol's governance and potentially destabilizing its economic model

## Tools Used
Manual review.

## Recommendations
Implement an appropriate check in `Airdrop::claim` as follows:

```diff
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
+      require( soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender)) != 0, "Unpaired souls cannot claim!");

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (
            block.timestamp - soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))
        ) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (amountAlreadyClaimed >= numberOfDaysInCouple * 10 ** loveToken.decimals()) {
            revert Airdrop__PreviousTokenAlreadyClaimed();
        }

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple * 10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (tokenAmountToDistribute >= loveToken.balanceOf(address(airdropVault))) {
            tokenAmountToDistribute = loveToken.balanceOf(address(airdropVault));
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);

        loveToken.transferFrom(address(airdropVault), msg.sender, tokenAmountToDistribute);
    }
```
## <a id='H-02'></a>H-02. Staking Rewards Misaligned with Staking Period            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L74

## Summary
The staking mechanism within `Staking.sol` incorrectly calculates rewards based on the duration since the soulmates were paired, rather than from when the users actually began staking their tokens. This misalignment can lead to users receiving rewards for a period during which their tokens were not actively staked, contrary to the intended functionality of rewarding users based on the time their capital is locked in the staking contract.

## Vulnerability Details
`Staking::claimRewards` determines the amount of rewards a user is eligible for based on the time elapsed since the soulmates were paired (as indicated by the timestamp of pairing). This approach does not account for the actual commencement of staking, potentially granting rewards for a period prior to the staking action. The provided test code demonstrates that rewards can be claimed immediately after depositing tokens into the staking contract, without a requisite waiting period reflective of actual staking duration:

```javascript
    function test_faultyStakingLogic() public {
        uint256 balanceUser1;
        uint256 balanceUser1_beforeRewardsClaim;
        uint256 balanceUser1_afterRewardsClaim;

        // pair up user 1 and 2
        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        vm.prank(user2);
        soulmateContract.mintSoulmateToken();

        vm.warp(block.timestamp + 4 weeks); // fast fwd time with 1 month
        vm.startPrank(user1);
        airdropContract.claim(); // claim airdrop
        balanceUser1 = loveToken.balanceOf(user1);
        loveToken.approve(address(stakingContract), type(uint256).max); // approve spending
        stakingContract.deposit(balanceUser1); // deposit all
        vm.stopPrank();

        // to claim rewards, we dont need to advance time after starting staking

        vm.startPrank(user1);
        balanceUser1_beforeRewardsClaim = loveToken.balanceOf(user1);
        stakingContract.claimRewards();
        balanceUser1_afterRewardsClaim = loveToken.balanceOf(user1);
        vm.stopPrank();

        assertLt(balanceUser1_beforeRewardsClaim, balanceUser1_afterRewardsClaim);
    }
```

## Impact
This flaw impacts the staking system in several ways:

1. Unwarranted Rewards: Users can claim rewards not duly earned through staking, undermining the fairness and integrity of the rewards distribution mechanism.
2. Resource Drain: The premature or unwarranted distribution of rewards can lead to a faster depletion of the rewards pool, affecting the sustainability of the staking program.
3. Incentive Structure Distortion: The misalignment between staking period and rewards calculation can distort the intended incentive structure, potentially rewarding short-term interactions over genuine long-term participation.

## Tools Used
Manual review.

## Recommendations
The Staking contract should be amended to calculate rewards based on the actual period tokens have been staked. This involves tracking the timestamp at which each user's tokens are deposited into the staking contract and using this timestamp to calculate rewards. 


```diff
+ mapping(address => uint256) private stakingStartTimes;

    function deposit(uint256 amount) public {
        if (loveToken.balanceOf(address(stakingVault)) == 0) {
            revert Staking__NoMoreRewards();
        }
+    if (stakingStartTimes[msg.sender] == 0) {
+        stakingStartTimes[msg.sender] = block.timestamp;
+    }
        // No require needed because of overflow protection
        userStakes[msg.sender] += amount;
        loveToken.transferFrom(msg.sender, address(this), amount);

        emit Deposited(msg.sender, amount);
    }


    function claimRewards() public {
        uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
        // first claim
        if (lastClaim[msg.sender] == 0) {
-            lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(soulmateId);
+            lastClaim[msg.sender] = stakingStartTimes[msg.sender];
        }

        // How many weeks passed since the last claim.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp - lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1) {
            revert Staking__StakingPeriodTooShort();
        }

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] * timeInWeeksSinceLastClaim;
        loveToken.transferFrom(address(stakingVault), msg.sender, amountToClaim);

        emit RewardsClaimed(msg.sender, amountToClaim);
    }

```

Note that the suggestion above has a limitation: it does not account for multiple deposits made by the same user at different times. To address this and accurately calculate rewards for each deposit, a more nuanced approach is required. This can involve tracking each deposit individually along with its timestamp. 
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can become soulmates with themselves, can deplete protocol funds by generating a lot of self-paired soulmates            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L62

## Summary
`Soulmate::mintSoulmateToken` is designed to facilitate the pairing of users with their soulmates. However, due to a flaw in the implementation, it is possible for a user to become their own soulmate, which contradicts the intended functionality and logic of creating interpersonal connections between different participants.

## Vulnerability Details
The vulnerability arises from the lack of checks within the `mintSoulmateToken` function to prevent a user from being paired with themselves. By calling mintSoulmateToken twice in succession without any intervening action, a user can bypass the intended pairing logic and end up being recorded as their own soulmate. This issue is demonstrated in the provided test code, where `user1` successfully becomes their own soulmate after calling `mintSoulmateToken` twice:

```javascript
    function test_usersCanBecomeTheirOwnSoulmate() public {
        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        vm.prank(user1);
        soulmateContract.mintSoulmateToken();

        assertEq(user1, soulmateContract.soulmateOf(user1));
    }
```

## Impact

Allowing users to become their own soulmates undermines the core purpose of the Soulmate contract and can have a number of undesirable effects:

1. Distorts Contract Logic: The fundamental logic and purpose of the contract are compromised, as the system is designed to foster connections between different users, not self-associations.
2. A malicious user can generate numerous self-paired soulmates. Creating self-paired soulmates can lead to unwarranted extraction of resources, such as claiming airdrops meant for genuinely paired users. This not only depletes the resources available for legitimate users but also inflates participation metrics, potentially draining the contract's assets.

## Tools Used
Manual review.

## Recommendations
Include a check that prevents a user from being paired with themselves:

```diff
  function mintSoulmateToken() public returns (uint256) {
        // Check if people already have a soulmate, which means already have a token
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0)) {
            revert Soulmate__alreadyHaveASoulmate(soulmate);
        }
        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
        } else if (soulmate2 == address(0)) {
+          require(msg.sender != soulmate1, "Can't be your own soulmate!");
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

            _mint(msg.sender, nextID++);
        }

        return ownerToId[msg.sender];
    }
```
## <a id='M-02'></a>M-02. Lack of Access Control in initVault Allows Front-Running            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Vault.sol#L27

## Summary
`Vault::initVault` lacks adequate access control mechanisms. This function is critical for initializing the vault with the `LoveToken` contract and setting the corresponding manager contract. However, malicious actors can monitor pending transactions within the Ethereum mempool and execute a transaction calling `initVault` with unauthorized parameters before the original transaction is confirmed. This vulnerability arises because any user can call `initVault` without restrictions, leading to potential unauthorized initialization.

## Vulnerability Details
The Ethereum mempool is a holding area for transactions awaiting confirmation. Transactions in the mempool are publicly visible, allowing actors to observe and potentially exploit transactions that are not yet confirmed. The` initVault` function's current implementation does not restrict who can call it, nor does it ensure that the call comes from a trusted source. Consequently, an attacker can front-run the legitimate initialization transaction by submitting a similar call with a higher gas price, ensuring their transaction is confirmed first. This could result in the vault being initialized with an attacker-controlled manager contract, compromising the integrity and security of the vault and associated funds or operations.

## Impact
Successful front-running of the initVault function call can lead to several adverse outcomes, including but not limited to:

1. Unauthorized control over the vault, including redirection of funds or manipulation of token distributions.
2. Loss of confidence in the security and reliability of the protocol, potentially deterring user participation.
3. Financial losses for users and the protocol due to unauthorized access and potential exploitation.

The protocol is forced to redeploy its contracts.


## Tools Used
Manual review.

## Recommendations
Add access control to `Vault::initVault` as follows:

```diff
+    modifier onlyOwner() {
+        require(msg.sender == owner, "Caller is not the owner");
+        _;
+    }

+    constructor() {
+        owner = msg.sender;
+    }

-    function initVault(ILoveToken loveToken, address managerContract) public {
+    function initVault(ILoveToken loveToken, address managerContract) public onlyOwner {
        if (vaultInitialize) revert Vault__AlreadyInitialized();
        loveToken.initVault(managerContract);
        vaultInitialize = true;
    }
```
## <a id='M-03'></a>M-03. Unpaired users can get divorced            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L124

## Summary
The `getDivorced` function is supposed to be used by users who have already been paired up with another user, to initiate a divorce between the two. However, a flaw in `Soulmate.sol` allows users to invoke the `getDivorced` function regardless of their current pairing status. This includes users who have never been paired (minted a soulmate token) and users who have initiated the minting process but have not yet been paired with another user. The issue allows for a divorce state to be set for users outside the intended logic of requiring a pair to be formed first.

## Vulnerability Details
The `Soulmate::getDivorced` function does not check whether a user is currently in a paired state before allowing the divorce action to proceed. This results in the possibility for a user to:

- Become divorced without ever having a soulmate.
- Become divorced while in a waiting state for a soulmate.
- Impact the logical flow of the contract by allowing users in unintended states to become divorced.

The contract uses a boolean mapping to track divorced states, and the absence of checks allows any user to set their divorced state to true.

Place the following piece of code to `SoulmateTest.t.sol`:

```javascript
    function test_singleSoulsCanGetDivorced() public {
        // user who never called mint can get divorced
        // user1 divorced
        vm.startPrank(user1);
        soulmateContract.getDivorced();
        bool isUser1Divorced = soulmateContract.isDivorced();
        vm.stopPrank();
        assertEq(isUser1Divorced, true); // but user1 can still call mintSoulmateToken and paired up with somebody

        // user who called mint can get divorced before having been paired up with somebody
        // user2 divorced
        vm.startPrank(user2);
        soulmateContract.mintSoulmateToken();
        soulmateContract.getDivorced();
        bool isUser2Divorced = soulmateContract.isDivorced();
        vm.stopPrank();
        assertEq(isUser2Divorced, true); // but user2 remains in waiting state
        console2.log("User 2 is divorced: ", isUser2Divorced);

        // a user can get paired up with someone else who accidentally got divorced as a single
        // divorced user2 is paired up with non-divorced user3
        vm.prank(user3);
        soulmateContract.mintSoulmateToken();
        assertEq(user2, soulmateContract.soulmateOf(user3));
        assertEq(user3, soulmateContract.soulmateOf(user2));

        vm.prank(user3);
        bool isUser3Divorced = soulmateContract.isDivorced();
        assertEq(isUser3Divorced, false);
        vm.prank(user2);
        isUser2Divorced = soulmateContract.isDivorced();
        console2.log("User 3 is divorced: ", isUser3Divorced);
        console2.log("User 2 is divorced: ", isUser2Divorced);
        assertEq(isUser2Divorced, true);

        // non-divorced user3 wants to get divorced from divorced user2
        vm.startPrank(user3);
        soulmateContract.getDivorced();
        isUser3Divorced = soulmateContract.isDivorced();
        vm.stopPrank();
        vm.prank(user2);
        isUser2Divorced = soulmateContract.isDivorced();
        console2.log("User 3 is divorced: ", isUser3Divorced);
        console2.log("User 2 is divorced: ", isUser2Divorced);
        assertEq(isUser2Divorced, true);
        assertEq(isUser3Divorced, true);
    }
```

## Impact
This vulnerability disrupts the intended logic and flow of the Soulmate contract by allowing users to reach a divorced state without ever entering a proper pairing. This could lead to inconsistencies within the contract state, affecting the overall integrity of the system.
For example, consider the following scenario:
1. `UserA` calls `Soulmate::mintSoulmateToken`, and waits to be paired up with somebody else.
2. Before having been paired up with another user, `UserA` accidentally calls `Soulmate::getDivorced` which sets its divorced state to `true`.
3. `UserB` calls  `Soulmate::mintSoulmateToken` and, consequently, is paired up with `UserA`. Now we have a couple made up by `UserA` who has a divorced state `true`, and`UserB` who has a divorced state `false`. 
4. Even though`UserA` has not divorced `UserB`, `UserA` will not be able to collect its share of love tokens by calling `Airdrop::claim` due to its divorced flag being `true`.

## Tools Used
Manual review.

## Recommendations 
Implement checks within `getDivorced` to ensure that a user is in a valid state to request a divorce:

```diff
    function getDivorced() public {
+      address soulmate = soulmateOf[msg.sender];
+      require(soulmate != address(0), "Soulmate: Not currently paired");
+      // Further check to ensure the caller is not already divorced
+      require(!isDivorced[msg.sender], "Soulmate: Already divorced");
        address soulmate2 = soulmateOf[msg.sender];
        divorced[msg.sender] = true;
        divorced[soulmateOf[msg.sender]] = true;
        emit CoupleHasDivorced(msg.sender, soulmate2);
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Staking Rewards Forfeited Upon Deposit Withdrawal            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L90

## Summary
`Staking.sol` is designed to reward users for locking their love tokens over a period. However, due to a flaw in the implementation, users who withdraw their deposited funds before claiming their accrued rewards forfeit these rewards. This behavior deviates from the expected outcome where rewards accrued during the staking period should be claimable until the moment of withdrawal.

## Vulnerability Details
The vulnerability stems from the Staking contract's handling of reward claims and withdrawals. Specifically, the contract does not account for or preserve the rewards accrued by a user's stake when they perform a withdrawal. As demonstrated in the provided test code, after advancing time to allow for reward accrual and then withdrawing the initial deposit, the user's balance before and after attempting to claim rewards remains unchanged, indicating that the rewards were forfeited upon withdrawal.

```javascript
   function test_rewardsForfeitedUponDepositWithdrawal() public {
        uint256 balanceUser1;
        uint256 balanceUser1_beforeRewardsClaim;
        uint256 balanceUser1_afterRewardsClaim;
        uint256 totalAmountDeposited = 0;

        // pair up user 1 and 2
        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        vm.prank(user2);
        soulmateContract.mintSoulmateToken();

        vm.warp(block.timestamp + 4 weeks); // fast fwd time with 1 month
        vm.startPrank(user1);
        airdropContract.claim(); // claim airdrop
        balanceUser1 = loveToken.balanceOf(user1);
        totalAmountDeposited += balanceUser1;
        loveToken.approve(address(stakingContract), type(uint256).max); // approve spending
        stakingContract.deposit(balanceUser1); // deposit all
        vm.stopPrank();

        vm.warp(block.timestamp + 4 weeks); // fast fwd time with 1 month (due to another bug, not really neccesary)

        vm.startPrank(user1);
        stakingContract.withdraw(totalAmountDeposited);
        balanceUser1_beforeRewardsClaim = loveToken.balanceOf(user1); // no rewards to claim
        stakingContract.claimRewards();
        balanceUser1_afterRewardsClaim = loveToken.balanceOf(user1);
        vm.stopPrank();

        assertEq(balanceUser1_beforeRewardsClaim, balanceUser1_afterRewardsClaim);
    }
```

## Impact
1. Loss of Expected Rewards: Users lose potential rewards they have rightfully earned through staking, which can lead to dissatisfaction and reduced participation in the staking mechanism.
2. Misalignment with Staking Incentives: The fundamental incentive for staking — earning rewards over time — is undermined if users risk losing rewards by withdrawing their stake.

## Tools Used
Manual review.

## Recommendations
Automate the rewards claim process during withdrawal to eliminate the need for users to manually claim rewards in a separate transaction:

```diff
    function withdraw(uint256 amount) public {
+     claimRewards();
        // No require needed because of overflow protection
        userStakes[msg.sender] -= amount;
        loveToken.transfer(msg.sender, amount);
        emit Withdrew(msg.sender, amount);
    }
```


Additionally, do clearly document and communicate this change to users, explaining how the automatic reward claim process works and its benefits, to maintain transparency and trust.
## <a id='L-02'></a>L-02. Misleading Total Souls Count due to Unpaired and Self-Paired Users            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L139

## Summary
`Soulmate::totalSouls` inaccurately calculates the total number of paired souls. This function currently returns a value that assumes all minting actions result in successful pairings, thereby doubling the `nextID` to represent total souls. However, this calculation does not account for users who are still awaiting pairing or those who have been incorrectly paired with themselves, leading to a misrepresented count of active soulmate pairs within the system.

## Vulnerability Details
The `totalSouls` function's simplistic calculation method (return nextID * 2;) overlooks two critical scenarios:

1. Unpaired Souls: Users who have initiated the minting process but are not yet paired. These users should not contribute to the total count of souls until their pairing is confirmed.
2. Self-Paired Users: The current logic does not prevent a user from being paired with themselves.

The provided test case illustrates this issue by demonstrating that the `totalSouls` count can be inaccurate immediately after a minting request and can also reflect an incorrect increment when a user is allowed to pair with themselves.

Proof of code:

```javascript
    function test_numberOfSouls() public {
        uint256 totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertEq(totalSouls, 0);

        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertLt(totalSouls, 1);

        vm.prank(user1);
        soulmateContract.mintSoulmateToken();
        totalSouls = soulmateContract.totalSouls();
        console2.log(totalSouls);
        assertGt(totalSouls, 1);
    }
```

## Impact
The inaccurate reporting of total souls impacts the transparency and reliability of the protocol's metrics. It could mislead users and stakeholders about the platform's activity level and the actual number of successful pairings, potentially affecting user trust and engagement.

## Tools Used
Manual review.

## Recommendations
The following modifications are recommended:

1. Prevent Self-Pairing: Implement checks within the minting function to prevent users from being paired with themselves, ensuring that all pairings are between distinct users:


```diff
  function mintSoulmateToken() public returns (uint256) {
        // Check if people already have a soulmate, which means already have a token
        address soulmate = soulmateOf[msg.sender];
        if (soulmate != address(0)) {
            revert Soulmate__alreadyHaveASoulmate(soulmate);
        }
        address soulmate1 = idToOwners[nextID][0];
        address soulmate2 = idToOwners[nextID][1];
        if (soulmate1 == address(0)) {
            idToOwners[nextID][0] = msg.sender;
            ownerToId[msg.sender] = nextID;
            emit SoulmateIsWaiting(msg.sender);
        } else if (soulmate2 == address(0)) {
+          require(msg.sender != soulmate1, "Can't be your own soulmate!");
            idToOwners[nextID][1] = msg.sender;
            // Once 2 soulmates are reunited, the token is minted
            ownerToId[msg.sender] = nextID;
            soulmateOf[msg.sender] = soulmate1;
            soulmateOf[soulmate1] = msg.sender;
            idToCreationTimestamp[nextID] = block.timestamp;

            emit SoulmateAreReunited(soulmate1, soulmate2, nextID);

            _mint(msg.sender, nextID++);
        }

        return ownerToId[msg.sender];
    }
```

2. Rename the function and change its implementation so that it returns the number of pairs, not the number of souls:

```diff
-    function totalSouls() external view returns (uint256) {
-        return nextID * 2;
+    function totalPairs() external view returns (uint256) {
+        return nextID;
    }
```


