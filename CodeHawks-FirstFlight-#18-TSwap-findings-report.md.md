# First Flight #18: T-Swap - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to overtax users with a 90.3% fee](#H-01)
    - ### [H-02. Lack of slippage protection in `TswapPool::swapExactOutput` casues users to potentially receive way fewer tokens](#H-02)
    - ### [H-03. `TSwapPool::sellPoolTokens` mistakenly calls the incorrect swap function, causing users to receive the incorrect amount of tokens](#H-03)
    - ### [H-04. In `TswapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`](#H-04)
- ## Medium Risk Findings
    - ### [M-01. `TSwapPool::deposit` is missing deadline check, causing transactions to complete even after the deadline passed](#M-01)
    - ### [M-02. Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant](#M-02)
- ## Low Risk Findings
    - ### [L-01. `TSwapPool::getPriceOfOneWethInPoolTokens()` and `TSwapPool::getPriceOfOnePoolTokenInWeth()` return incorrect price values](#L-01)
    - ### [L-02. `TSwapPool::LiquidityAdded` event has parameters out of order, causing event to emit incorrect information](#L-02)
    - ### [L-03. In the function declaration of `TSwapPool::swapExactInput`, a return value is defined but never assigned a value, resulting in a default but incorrect return value](#L-03)
    - ### [L-04. Use `.symbol` instead of `.name` when assigning a value to `liquidityTokenSymbol` In `PoolFactory::createPool()`, ](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #18

### Dates: Jun 20th, 2024 - Jun 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-06-t-swap)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 4
- Medium: 2
- Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput` causes protocol to overtax users with a 90.3% fee            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L291-L293

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L349

## Summary

`TSwapPool::getInputAmountBasedOnOutput` overtaxes users with a 90.3% swap fee.

## Vulnerability Details

The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens the users should deposit given the amount of output tokens. However, the function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by 10_000 instead of 1_000, resulting in a 90.3% fee.

Consider the following scenario:

1. The user calls `TSwapPool::swapExactOutput` to buy a predefined amount of output tokens in exchange for an undefined amount of input tokens.
2. `TSwapPool::swapExactOutput` calls the `getInputAmountBasedOnOutput` function that is supposed to calculate the amount of input tokens required to result in the predefined amount of output tokens. However, the fee calculation in this function in incorrect.
3. The user gets overtaxed with a 90.4% fee.

Also consider the following test as proof of code:

<details>
<summary>Proof of Code </summary>

```javascript
function test_overTaxingUsersInSwapExactOutput() public {
        // providing liquidity to the pool
        uint256 initialLiquidity = 100e18;

        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidity);
        poolToken.approve(address(pool), initialLiquidity);

        pool.deposit({
            wethToDeposit: initialLiquidity,
            minimumLiquidityTokensToMint: 0,
            maximumPoolTokensToDeposit: initialLiquidity,
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        // @audit this function actually returns an incorrect value
        // uint256 priceOfOneWeth = pool.getPriceOfOneWethInPoolTokens();

        address someUser = makeAddr("someUser");
        uint256 userInitialPoolTokenBalance = 11e18;
        poolToken.mint(someUser, 11e18); // now the user has 11 pool tokens

        // user intends to buy 1 weth with pool tokens
        vm.startPrank(someUser);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, 1e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 userEndingPoolTokenBalance = poolToken.balanceOf(someUser);

        console.log("Initial user balance: ", userInitialPoolTokenBalance);
        // console.log("Price of 1 weth: ", priceOfOneWeth);
        console.log("Ending user balance: ", userEndingPoolTokenBalance);

        // Initial liquidity was 1:1, so user should have paid ~1 pool token
        // However, it spent much more than that. The user started with 11 tokens, and now only has less than 1.
        assert(userEndingPoolTokenBalance < 1 ether);
    }
```
</details>

## Impact

The protocol takes way more fees than expected by the users.

## Tools Used
Manual review, Foundry.

## Recommendations

Perform the following corrections in `TSwapPool::getInputAmountBasedOnOutput`:

```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return ((inputReserves * outputAmount) * 10_000) / ((outputReserves - outputAmount) * 997);
+        return ((inputReserves * outputAmount) * 1_000) / ((outputReserves - outputAmount) * 997);

    }
```
## <a id='H-02'></a>H-02. Lack of slippage protection in `TswapPool::swapExactOutput` casues users to potentially receive way fewer tokens            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L335-L356

## Summary

`TSwapPool::swapExactOutput` does not include proper slippage protection.

## Vulnerability Details

The function `swapExactOutput` does not include any kind of slippage protection. This function is similar to what is done in `TSwapPool::swapExactInput`, where the function specifies the `minOutputAmount`. Similarly, `swapExactOutput` should specify a `maxInputAmount`.

## Impact

If the market conditions change before the transaction process, the user could get a much worse swap then expected.

## Tools Used

Foundry, manual review.

1. The price of WETH is 1_000 USDC.
2. User calls `swapExactOutput`, looking for 1 WETH with the following parameters:
   - inputToken: USDC
   - outputToken: WETH
   - outputAmount: 1
   - deadline: whatever
3. The function does not allow a `maxInputAmount`.
4. As the transaction is pending in the mempool, the market changes, and the price movement is huge: 1 WETH now costs 10_000 USDC, 10x more than the user expected!
5. The transaction completes, but the user got charged 10_000 USDC for 1 WETH.

## Recommendations

Include a `maxInputAmount` input parameter in the function declaration, so that the user could specify the maximum amount of tokens he would like to spend and, hence, could predict their spending when using this function of the protocol.

```diff
    function swapExactOutput(
        IERC20 inputToken,
+       uint256 maxInputAmount,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

+       if(inputAmount > maxInputAmount){
+           revert();
+       }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```
## <a id='H-03'></a>H-03. `TSwapPool::sellPoolTokens` mistakenly calls the incorrect swap function, causing users to receive the incorrect amount of tokens            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L367-L372

## Summary

`TSwapPool::sellPoolTokens` mistakenly calls the incorrect swap function.

## Vulnerability Details

The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive WETH in exchange. In the `poolTokenAmount` parameter, users indicate how many pool token they intend to sell. However, the function mistakenly calls `swapExactOutput` instead of `swapExactInput` to perform the swap, and therein assignes the value of `poolTokenAmount` to function input argument `outputAmount`, effectively mixing up the input and output tokens / amounts. 

Consider the following scenario:

1. A user has 100 pool tokens, and wants to sell 5 by calling the `sellPoolTokens` function.
2. Instead of the `swapExactInput` function, `sellPoolTokens`  calls `swapExactOutput`.
3. In `swapExactOutput`, `poolTokenAmount` is used as `outputAmount` while it is really the input amount.
4. As a result, user will swap more output tokens than originally intended.

Apart from this, the user will be overtaxed due to a bug in `getInputAmountBasedOnOutput()` called by `swapExactOutput`.


For a proof of code, add this piece of code to `TSwapPool.t.sol`:

<details>
<summary>Proof of Code</summary>

```javascript
function test_sellPoolTokensCallsTheIncorrectSwapFunction() public {
        // setting up the pool by providing liquidity
        uint256 initialLiquidity = 100e18;

        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidity);
        poolToken.approve(address(pool), initialLiquidity);

        pool.deposit({
            wethToDeposit: initialLiquidity,
            minimumLiquidityTokensToMint: 0,
            maximumPoolTokensToDeposit: initialLiquidity,
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        // setting up the user
        address someUser = makeAddr("someUser");
        uint256 userStartingPoolTokenBalance = 100 ether;
        poolToken.mint(someUser, userStartingPoolTokenBalance);
        vm.prank(someUser);
        poolToken.approve(address(pool), type(uint256).max);

        // user intends to sell 5 pool tokens
        vm.prank(someUser);
        uint256 poolTokensToSell = 5e18;
        // @note that sellPoolTokens() uses swapExactOutput() to perform the swap,
        // which in turn calls getInputAmountBasedOnOutput() to calculate the amount of input tokens to be
        // deducted from the user, and this function miscalculates the fee, so to make things worse,
        // the user becomes subject of overtaxing too
        pool.sellPoolTokens(poolTokensToSell);

        uint256 expectedEndingUserPoolTokenBalance = userStartingPoolTokenBalance - poolTokensToSell;
        uint256 realEndingUserPoolTokenBalance = poolToken.balanceOf(someUser);

        console.log("Expected pool token balance of the user: ", expectedEndingUserPoolTokenBalance);
        console.log("Real pool token balance of the user: ", realEndingUserPoolTokenBalance);

        assert(expectedEndingUserPoolTokenBalance > realEndingUserPoolTokenBalance);
    }
```
</details>

## Impact

Users will swap the incorrects amount of tokens, which severely discrupts the functionality of the protocol.

## Tools Used
Manual review, Foundry.

## Recommendations

Change the implementation to use `swapExactInput` instead of the `swapExactOutput` function. Note that this would require the `sellPoolTokens` function to accept an additional parameter (i.e. `minOutputAmount` to be passed to `swapExactInput`).

```diff

-    function sellPoolTokens(uint256 poolTokenAmount) external returns (uint256 wethAmount) {
+    function sellPoolTokens(uint256 poolTokenAmount, uint256 minWethToReceive) external returns (uint256 wethAmount) {

-        return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+        return swapExactInput(i_poolToken, i_wethToken, poolTokenAmount, minWethToReceive, uint64(block.timestamp));

    }
```

Additionally, it might be wise to add a deadline to the function, as currently there is no deadline.
## <a id='H-04'></a>H-04. In `TswapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L400

## Summary

In `TswapPool::_swap` the extra tokens given to users after every `swapCount` breaks the protocol invariant of `x * y = k`

## Vulnerability Details

The protocol follows a strict invariant of `x * y = k`, where 
- `x`: The balance of the pool token in the pool
- `y`: The balance of WETH in the pool
- `k`: The constant product of the 2 balances

This means that whenever the balances change in the protocol, the ratio between the two amounts should remain constant, hence the `k`. However, this is broken due to the extra incentive (a full extra token after every 10 swaps) in the `_swap` function, meaning that over time the protocol funds would be drained. 

The following block of code is responsible for the issue.

```javascript
        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
```

1. A user swaps 10 times and collects the extra incentive of 1 token (`1_000_000_000_000_000_000`)
2. The usercontinues to swap until all the protocol funds are drained.

Consider the following proof of code:

<details>
<summary>Proof of Code</summary>

```javascript
 function test_InvariantBreaks() public {
        // providing liquidity
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;

        // set up user, than perform 9 swaps
        vm.startPrank(user);
        poolToken.mint(user, 100e18);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        // from the invariant test (the handler).
        // We use these here because the interesting thing happens at the 10th swap
        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth); // this is deltaY

        // and then do a swap for the 10th time
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        // from the invariant test (the handler)
        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY); // this could be negative

        assertEq(expectedDeltaY, actualDeltaY);
    }
```
</details>

## Impact
A user could maliciously drain the protocol of funds by doing a lot of swaps and collecting the extra incentive (a full extra token after every 10 swaps) given out by the protocol. 

More simply put, the core invariant of the protocol is broken.

## Tools Used
Manual review, Foundry.

## Recommendations

Remove the extra incentive mechanism. If you want to keep this nonetheless, you should account for the change in the `x * y = k` invariant. Alternatively, you could set aside tokens the same way you did with fees.

```diff
-        swap_count++;
-        if (swap_count >= SWAP_COUNT_MAX) {
-            swap_count = 0;
-            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-        }
```
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `TSwapPool::deposit` is missing deadline check, causing transactions to complete even after the deadline passed            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L113-L182

## Vulnerability Details

The `deposit` function accepts a `deadline` as an input the parameter which, according to the documentation, "the deadline for the transaction to be completed by". However, this parameter is never actually used. As a consequence, operations that add liquidty to the pool might be executed at unexpected times, in market conditions when the deposit rate is unfavorable.

This also makes this part susceptible to MEV attacks.

Proof of concept: the `deadline` parameter is unused (this is highlighted by the compiler too).

## Impact
 Transactions can be sent when market conditions are unfavorable, even when the deadline is set.

## Tools Used
Manual review, Foundry.

## Recommendations

Perform the following changes to the function:

```diff
   function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {...}

```
## <a id='M-02'></a>M-02. Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L458-L465

## Summary
Weird ERC20 tokens with uncommon / malicious implementations can endanger the whole protocol. Examples include rebase, fee-on-transfer, and ERC777 tokens.



## Rebase Tokens: they automatically adjust their total supply periodically. This can disrupt the balance checks and liquidity calculations in the TSwapPool.

<details>
<summary>Rebase token implementation</summary>

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract RebaseToken is ERC20 {
    address public owner;

    constructor() ERC20("Rebase Token", "RBT") {
        owner = msg.sender;
        _mint(msg.sender, 1000 * 10 ** decimals());
    }

    function rebase(uint256 amount) external {
        require(msg.sender == owner, "Only owner can rebase");
        _mint(owner, amount);
    }
}
```
</details>

<details>
<summary>Test case</summary>

```javasscript
function testRebaseToken() public {
    RebaseToken rebaseToken = new RebaseToken();
    pool = new TSwapPool(address(rebaseToken), address(weth), "LTokenA", "LA");

    // Initial minting
    rebaseToken.mint(liquidityProvider, 200e18);
    weth.mint(liquidityProvider, 200e18);

    vm.startPrank(liquidityProvider);
    weth.approve(address(pool), 100e18);
    rebaseToken.approve(address(pool), 100e18);
    pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
    vm.stopPrank();

    // Rebase token supply
    rebaseToken.rebase(100e18);

    // Rebase affects the balance calculations
    vm.startPrank(user);
    rebaseToken.approve(address(pool), 10e18);
    uint256 expected = 9e18;
    pool.swapExactInput(rebaseToken, 10e18, weth, expected, uint64(block.timestamp));
    assert(weth.balanceOf(user) < expected); // The rebase disrupts the swap ratio
}
```
</details>





## Fee-on-Transfer Tokens: they deduct a fee on every transfer, causing discrepancies between sent and received amounts in the pool.

<details>
<summary>Fee-on-transfer token implementation</summary>

```javasscript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract FeeOnTransferToken is ERC20 {
    uint256 public feePercentage = 1; // 1%

    constructor() ERC20("Fee Token", "FEE") {
        _mint(msg.sender, 1000 * 10 ** decimals());
    }

    function _transfer(address sender, address recipient, uint256 amount) internal override {
        uint256 fee = (amount * feePercentage) / 100;
        uint256 amountAfterFee = amount - fee;
        super._transfer(sender, recipient, amountAfterFee);
        super._transfer(sender, address(this), fee);
    }
}
```

</details>

<details>
<summary>Test case</summary>

```javasscript
function testFeeOnTransferToken() public {
    FeeOnTransferToken feeToken = new FeeOnTransferToken();
    pool = new TSwapPool(address(feeToken), address(weth), "LTokenA", "LA");

    // Initial minting
    feeToken.mint(liquidityProvider, 200e18);
    weth.mint(liquidityProvider, 200e18);

    vm.startPrank(liquidityProvider);
    weth.approve(address(pool), 100e18);
    feeToken.approve(address(pool), 100e18);
    pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
    vm.stopPrank();

    // Perform a swap
    vm.startPrank(user);
    feeToken.approve(address(pool), 10e18);
    uint256 expected = 9e18;
    pool.swapExactInput(feeToken, 10e18, weth, expected, uint64(block.timestamp));
    assert(weth.balanceOf(user) < expected); // The fee on transfer disrupts the swap ratio
}
```

</details>




## ERC777 Tokens: they can execute arbitrary code during transfers, allowing reentrancy attacks.

<details>
<summary>ERC777 implementation</summary>

```javasscript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "@openzeppelin/contracts/token/ERC777/ERC777.sol";

contract MaliciousERC777 is ERC777 {
    address public pool;
    bool private inCallback = false;

    constructor(address[] memory defaultOperators) ERC777("Malicious Token", "MAL", defaultOperators) {
        _mint(msg.sender, 1000 * 10 ** decimals(), "", "");
    }

    function attack(address _pool) external {
        pool = _pool;
        _mint(address(this), 1 * 10 ** decimals(), "", "");
        IERC20(pool).approve(pool, 1 * 10 ** decimals());
        TSwapPool(pool).swapExactInput(IERC20(address(this)), 1 * 10 ** decimals(), IERC20(pool), 0, uint64(block.timestamp));
    }

    function tokensReceived(
        address,
        address from,
        address,
        uint256,
        bytes calldata,
        bytes calldata
    ) external override {
        if (!inCallback) {
            inCallback = true;
            TSwapPool(pool).withdraw(1 * 10 ** decimals(), 1, 1, uint64(block.timestamp));
            inCallback = false;
        }
    }
}
```

</details>


<details>
<summary>Test case</summary>

```javasscript
function testERC777Reentrancy() public {
    MaliciousERC777 malToken = new MaliciousERC777(new address );
    pool = new TSwapPool(address(malToken), address(weth), "LTokenA", "LA");

    // Initial minting
    malToken.mint(liquidityProvider, 200e18);
    weth.mint(liquidityProvider, 200e18);

    vm.startPrank(liquidityProvider);
    weth.approve(address(pool), 100e18);
    malToken.approve(address(pool), 100e18);
    pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
    vm.stopPrank();

    // Attack using ERC777 reentrancy
    vm.startPrank(user);
    malToken.attack(address(pool));
    // The attack should drain funds from the pool due to reentrancy
    assert(malToken.balanceOf(user) > 1 * 10 ** decimals());
}
```

</details>


## Impact
Protocol invariant is broken.

## Tools Used
Manual review, Foundry.

## Recommendations
Token Whitelist: Implement a whitelist for allowed ERC20 tokens to ensure only well-audited and known tokens can be used in the pool.

# Low Risk Findings

## <a id='L-01'></a>L-01. `TSwapPool::getPriceOfOneWethInPoolTokens()` and `TSwapPool::getPriceOfOnePoolTokenInWeth()` return incorrect price values            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L449-L456

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L458-L465

## Vulnerability Details

`getPriceOfOneWethInPoolTokens` is supposed to return the price of 1 WETH in terms of pool tokens, and `TSwapPool::getPriceOfOnePoolTokenInWeth` is supposed to return the price of 1 pool token in terms of WETH. However, the return values are incorrect. Both functions return the amount of output tokens, which is not the same as the price of 1 output token in input tokens. (Consider this: as compared to a fee-less protocol, if there are fees, the amount of output tokens should be lower, while the price should be not lower but higher.) 

Proof of concept: consider the following scenario:

1. A user has 1 WETH, and wants to swap it for pool tokens.
2. The user calls `getPriceOfOneWethInPoolTokens` and sees an incorrect price that is the inverse of the actual price.
3. User finds the price appealing and swaps his WETH.
4. User ends up with a lot less pool tokens than he expected.


Proof of code: Insert this piece of code to `TSwapPool.t.sol` (note that it demonstrates a different scenario than the one written under "Proof of Concept"):

<details>
<summary>Proof of Code</summary>

```javascript
 /**
     * @notice In scenarios where inputAmount is close to the minOutputAmount, even a small loss of precision can lead
     * to the outputAmount falling below minOutputAmount, triggering the TSwapPool__OutputTooLow error.
     */
    function test_incorrectPriceValueReturnedByGetPriceOfOneWethInPoolTokens() public {
        uint256 precision = 1 ether;

        // we need more liquidity in the pool, so granting additional money for the provider
        weth.mint(liquidityProvider, 800e18);
        poolToken.mint(liquidityProvider, 800e18);

        // providing liquidity to the pool
        uint256 initialLiquidity = 1000e18;

        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidity);
        poolToken.approve(address(pool), initialLiquidity);

        pool.deposit({
            wethToDeposit: initialLiquidity,
            minimumLiquidityTokensToMint: 0,
            maximumPoolTokensToDeposit: initialLiquidity,
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        uint256 incorrectPriceOfOneWeth = pool.getPriceOfOneWethInPoolTokens();
        uint256 correctPriceOfOneWeth = (1e18 * precision) / incorrectPriceOfOneWeth;

        console.log("Incorrect price: ", incorrectPriceOfOneWeth); // 987_158_034_397_061_298 = 9.87*10**17
        console.log("Correct price: ", correctPriceOfOneWeth);

        address userSuccess = makeAddr("userSuccess");
        address userFail = makeAddr("userFail");

        // userFail attempts to buy 1 weth with a balance of pool tokens that equals the incorrect price of 1 weth
        poolToken.mint(userFail, incorrectPriceOfOneWeth);
        vm.startPrank(userFail);
        poolToken.approve(address(pool), type(uint256).max);
        // expect a revert (specifically, TSwapPool__OutputTooLow)
        vm.expectRevert();
        // using swapExactOutput() would be more appropriate here, but that one has a huge bug
        pool.swapExactInput({
            inputToken: poolToken,
            inputAmount: incorrectPriceOfOneWeth,
            outputToken: weth,
            minOutputAmount: 99999e13, // due to precision loss, we cant really expect to get 1 full weth (1000e15), we
                // can only approximate
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        // userSuccess attempts to buy 1 weth with a balance of pool tokens that equals the correct price of 1 weth
        poolToken.mint(userSuccess, correctPriceOfOneWeth);
        vm.startPrank(userSuccess);
        poolToken.approve(address(pool), type(uint256).max);
        // using swapExactOutput() would be more appropriate here, but that one has a huge bug
        pool.swapExactInput({
            inputToken: poolToken,
            inputAmount: correctPriceOfOneWeth,
            outputToken: weth,
            minOutputAmount: 99999e13, // due to precision loss, we cant really expect to get 1 full weth (1000e15), we
                // can only approximate
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        assert(weth.balanceOf(userSuccess) > 999e15); // has nearly 1 full weth
        assertEq(poolToken.balanceOf(userSuccess), 0); // spent all his poolToken
    }
```

</details>

## Impact
User will think that the WETH / pool token is cheaper that it actually is, and they might make their trading decisions based on this incorrect price information. E.g. they might think the price of their token is falling, might panic and sell their tokens to avoid further losses by calling `sellPoolTokens()`.

## Tools Used
Manual review, Foundry.

## Recommendations
Perform the following changes:


```diff
    function getPriceOfOneWethInPoolTokens() external view returns (uint256) {
-        return getOutputAmountBasedOnInput(
-            1e18, i_wethToken.balanceOf(address(this)), i_poolToken.balanceOf(address(this))
-        );
+       uint256 precision = 1e18;
+       uint256 amountOfPoolTokensReceivedForOneWeth = getOutputAmountBasedOnInput(
+            1e18, i_wethToken.balanceOf(address(this)), i_poolToken.balanceOf(address(this)));
+            
+        uint256 priceOfOneWethInPoolTokens = (1e18 * precision) / amountOfPoolTokensReceivedForOneWeth;
+
+     return priceOfOneWethInPoolTokens;
    }

    function getPriceOfOnePoolTokenInWeth() external view returns (uint256) {
-        return getOutputAmountBasedOnInput(
-            1e18, i_poolToken.balanceOf(address(this)), i_wethToken.balanceOf(address(this))
-        );
+       uint256 precision = 1e18;
+       uint256 amountOfWethReceivedForOnePoolToken = getOutputAmountBasedOnInput(
+            1e18, i_poolToken.balanceOf(address(this)), i_wethToken.balanceOf(address(this)));
+            
+        uint256 priceOfOnePoolTokenInWeth = (1e18 * precision) / amountOfWethReceivedForOnePoolToke;
+
+     return priceOfOnePoolTokenInWeth;
    }
```
## <a id='L-02'></a>L-02. `TSwapPool::LiquidityAdded` event has parameters out of order, causing event to emit incorrect information            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L52-L56

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L194

## Vulnerability Details
When the `LiquidtyAdded` event is emitted in the `TSwapPool::_addLiquidityMintAndTransfer`, it logs values in the incorrect order. The `poolTokensToDeposit` value should go to the 3rd parameter position, whereas the `wethToDeposit` should go to 2nd.

## Impact
The emit emission is incorrect, causing off-chain functions potentially malfunctioning.

## Tools Used
Manual review, Foundry.

## Recommendations
Perform the following modifications:

```diff
-        emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+        emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```
## <a id='L-03'></a>L-03. In the function declaration of `TSwapPool::swapExactInput`, a return value is defined but never assigned a value, resulting in a default but incorrect return value            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L306

## Summary
The `swapExactInput` function is expected to return the actual amount of tokens bought by the user. However, while it declares the named return value `output`, it never assigns a value to it, nor uses an explicit return statement.

## Impact
The return value will always be 0, giving an incorrect information to the user.

## Tools Used
Manual review, Foundry.

## Recommendations

```diff
    function swapExactInput(
        IERC20 inputToken, // e input token to swap, e.g. sell DAI
        uint256 inputAmount, // e amount of DAI to sell
        IERC20 outputToken, // e token to buy, e.g. weth
        uint256 minOutputAmount, // mint weth to get
        uint64 deadline // deadline for the swap execution
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
-            uint256 output
+            uint256 outputAmount

        )
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
+        outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);



        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);


        }
        _swap(inputToken, inputAmount, outputToken, outputAmount);

+       return outputAmount;

    }

```

## <a id='L-04'></a>L-04. Use `.symbol` instead of `.name` when assigning a value to `liquidityTokenSymbol` In `PoolFactory::createPool()`,             

### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/PoolFactory.sol#L52

## Summary
`.symbol` is used instead of `.name` when assigning a value to `liquidityTokenSymbol` In `PoolFactory::createPool()`, 

## Impact
Confusion.

## Tools Used
Manual review.

## Recommendations
To be more in line with the name of the variable (and intended use thereof), use `.symbol` instead of `.name` when assigning a value to `liquidityTokenSymbol` In `PoolFactory::createPool()`

```diff
-         string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+         string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());

```


