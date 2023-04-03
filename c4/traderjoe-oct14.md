# Sherlock / traderjoe-oct14
***************************
***************************
# [H-01] Wrong calculation in function `LBRouter._getAmountsIn` make user lose a lot of tokens when swap through JoePair (most of them will gifted to JoePair freely)

## Lines of code
* https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L725

## Vulnerable detail 
Function `LBRouter._getAmountsIn` is a helper function to return the amounts in with given `amountOut`. This function will check the pair of `_token` and `_tokenNext` is `JoePair` or `LBPair` using `_binStep`.
* If `_binStep == 0`, it will be a `JoePair` otherwise it will be an `LBPair`.
```solidity=
if (_binStep == 0) {
    (uint256 _reserveIn, uint256 _reserveOut, ) = IJoePair(_pair).getReserves();
    if (_token > _tokenPath[i]) {
        (_reserveIn, _reserveOut) = (_reserveOut, _reserveIn);
    }


    uint256 amountOut_ = amountsIn[i];
    // Legacy uniswap way of rounding
    amountsIn[i - 1] = (_reserveIn * amountOut_ * 1_000) / (_reserveOut - amountOut_ * 997) + 1;
} else {
    (amountsIn[i - 1], ) = getSwapIn(ILBPair(_pair), amountsIn[i], ILBPair(_pair).tokenX() == _token);
}
```
As we can see when `_binStep == 0` and `_token < _tokenPath[i]` (in another word  we swap through `JoePair` and pair's`token0` is `_token` and `token1` is `_tokenPath[i]`), it will 
1. Get the reserve of pair (`reserveIn`, `reserveOut`) 
2. Calculate the `_amountIn` by using the formula 
```
amountsIn[i - 1] = (_reserveIn * amountOut_ * 1_000) / (_reserveOut - amountOut_ * 997) + 1
```

But unfortunately the denominator `_reserveOut - amountOut_ * 997` seem incorrect. It should be `(_reserveOut - amountOut_) * 997`. 
We will do some math calculations here to prove the expression above is wrong. 

**Input:** 
* `_reserveIn (rIn)`: reserve of `_token` in pair 
* `_reserveOut (rOut)`: reserve of `_tokenPath[i]` in pair 
* `amountOut_`: the amount of `_tokenPath` the user wants to gain 
 
**Output:** 
* `rAmountIn`: the actual amount of `_token` we need to transfer to the pair. 

**Generate Formula** 
Cause `JoePair` [takes 0.3%](https://help.traderjoexyz.com/en/welcome/faq-and-help/general-faq#what-are-trader-swap-joe-fees) of `amountIn` as fee, we get 
* `amountInDeductFee = amountIn' * 0.997`

Following the [constant product formula](https://docs.uniswap.org/protocol/V2/concepts/protocol-overview/glossary#constant-product-formula), we have 
```
    rIn * rOut = (rIn + amountInDeductFee) * (rOut - amountOut_)
==> rIn + amountInDeductFee = rIn * rOut / (rOut - amountOut_) + 1
<=> amountInDeductFee = (rIn * rOut) / (rOut - amountOut_) - rIn + 1
<=> rAmountIn * 0.997 = rIn * amountOut / (rOut - amountOut_) + 1
<=> rAmountIn = (rIn * amountOut * 1000) / ((rOut - amountOut_) * 997) + 1
<=> 
```

As we can see `rAmountIn` is different from `amountsIn[i - 1]`, the denominator of `rAmountIn` is `(rOut - amountOut_) * 997` when the denominator of `amountsIn[i - 1]` is `_reserveOut - amountOut_ * 997` (Missing one bracket)

## Impact
**Loss of fund: User will send a lot of tokenIn (much more than expected) but just gain exact amountOut in return.** 

Let dive in the function `swapTokensForExactTokens()` to figure out why this scenario happens. I will assume I just swap through only one pool from `JoePair` and 0 pool from `LBPair`. 
* Firstly function will get the list `amountsIn` from function `_getAmountsIn`. So `amountsIn` will be [`incorrectAmountIn`, `userDesireAmount`]. 
    ```solidity=        
    // url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L440
    amountsIn = _getAmountsIn(_pairBinSteps, _pairs, _tokenPath, _amountOut);
    ``` 
* Then it transfers `incorrectAmountIn` to `_pairs[0]` to prepare for the swap. 
    ```solidity=
    // url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L444
    _tokenPath[0].safeTransferFrom(msg.sender, _pairs[0], amountsIn[0]);
    ```  
* Finally it calls function `_swapTokensForExactToken` to execute the swap. 
    ```solidity=
    // url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L446    
    uint256 _amountOutReal = _swapTokensForExactTokens(_pairs, _pairBinSteps, _tokenPath, amountsIn, _to);
    ```
    In this step it will reach to [line 841](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L841) which will set the expected `amountOut = amountsIn[i+1] = amountsIn[1] = userDesireAmount`.
    ```solidity=
    // url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L841
    amountOut = _amountsIn[i + 1];
    ```
    So after calling `IJoePair(_pair).swap()`, the user just gets exactly `amountOut` and wastes a lot of tokenIn that (s)he transfers to the pool. 


## Proof of concept 
Here is our test script to describe the impacts 
* https://gist.github.com/huuducst/6e34a7bdf37bb29f4b84d2faead94dc4

You can place this file into `/test` folder and run it using 
```bash=
forge test --match-test testBugSwapJoeV1PairWithLBRouter --fork-url https://rpc.ankr.com/avalanche --fork-block-number 21437560 -vv
```

Explanation of test script: (For more detail u can read the comments from test script above)
1. Firstly we get the Joe v1 pair WAVAX/USDC from JoeFactory.
2. At the forked block, price `WAVAX/USDC` was around 15.57. We try to use LBRouter function `swapTokensForExactTokens` to swap 10$ WAVAX (10e18 wei) to 1$ USDC (1e6 wei). But it reverts with the error `LBRouter__MaxAmountInExceeded`.
But when we swap directly to JoePair, it swap successfully 10$ AVAX (10e18 wei) to 155$ USDC (155e6 wei).
3. We use LBRouter function `swapTokensForExactTokens` again with very large `amountInMax` to swap 1$ USDC (1e6 wei). It swaps successfully but needs to pay a very large amount WAVAX (much more than price).

## Tools Used
Foundry 
 
## Recommended Mitigation Steps
Modify function `LBRouter._getAmountsIn` as follow
```solidity=
// url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L717-L728
if (_binStep == 0) {
    (uint256 _reserveIn, uint256 _reserveOut, ) = IJoePair(_pair).getReserves();
    if (_token > _tokenPath[i]) {
        (_reserveIn, _reserveOut) = (_reserveOut, _reserveIn);
    }


    uint256 amountOut_ = amountsIn[i];
    // Legacy uniswap way of rounding
    // Fix here 
    amountsIn[i - 1] = (_reserveIn * amountOut_ * 1_000) / ((_reserveOut - amountOut_) * 997) + 1;
} else {
    (amountsIn[i - 1], ) = getSwapIn(ILBPair(_pair), amountsIn[i], ILBPair(_pair).tokenX() == _token);
}
```

***************************
***************************
# [H-02] Wrong implementation of function `LBPair.setFeeParameter` can break the funcionality of LBPair and make user's tokens locked 

## Lines of code
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L905-L917

## Vulnerable detail 
Struct `FeeParameters` contains 12 fields as follows: 
```solidity=
struct FeeParameters {
    // 144 lowest bits in slot 
    uint16 binStep;
    uint16 baseFactor;
    uint16 filterPeriod; 
    uint16 decayPeriod; 
    uint16 reductionFactor; 
    uint24 variableFeeControl;
    uint16 protocolShare;
    uint24 maxVolatilityAccumulated; 
    
    // 112 highest bits in slot 
    uint24 volatilityAccumulated;
    uint24 volatilityReference;
    uint24 indexRef;
    uint40 time; 
}
```
Function [`LBPair.setFeeParamters(bytes _packedFeeParamters)`](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L788-L790) is used to set the first 8 fields which was stored in 144 lowest bits of `LBPair._feeParameter`'s slot to 144 lowest bits of `_packedFeeParameters` (The layout of `_packedFeeParameters` can be seen [here](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBFactory.sol#L572-L584)).
```solidity=
/// url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L905-L917

/// @notice Internal function to set the fee parameters of the pair
/// @param _packedFeeParameters The packed fee parameters
function _setFeesParameters(bytes32 _packedFeeParameters) internal {
    bytes32 _feeStorageSlot;
    assembly {
        _feeStorageSlot := sload(_feeParameters.slot)
    }

    /// [#explain]  it will get 112 highest bits of feeStorageSlot,
    ///             and stores it in the 112 lowest bits of _varParameters 
    uint256 _varParameters 
        = _feeStorageSlot.decode(type(uint112).max, _OFFSET_VARIABLE_FEE_PARAMETERS/*=144*/);

    /// [#explain]  get 144 lowest bits of packedFeeParameters 
    ///             and stores it in the 144 lowest bits of _newFeeParameters  
    uint256 _newFeeParameters = _packedFeeParameters.decode(type(uint144).max, 0);

    assembly {
        // [$audit-high] wrong operation `or` here 
        //              Mitigate: or(_newFeeParameters, _varParameters << 144)    
        sstore(_feeParameters.slot, or(_newFeeParameters, _varParameters))
    }
}
```
As we can see in the implementation of `LBPair._setFeesParametes` above, it gets the 112 highest bits of `_feeStorageSlot` and stores it in the 112 lowest bits of `_varParameter`. Then it gets the 144 lowest bits of `packedFeeParameter` and stores it in the 144 lowest bits of `_newFeeParameters`. 

Following the purpose of function `setFeeParameters`, the new `LBPair._feeParameters` should form as follow: 
```
// keep 112 highest bits remain unchanged 
// set 144 lowest bits to `_newFeeParameter`
[...112 bits...][....144 bits.....]
[_varParameters][_newFeeParameters]
```
It will make `feeParameters = _newFeeParameters | (_varParameters << 144)`. But current implementation just stores the `or` value of `_varParameters` and `_newFeeParameter` into `_feeParameters.slot`. It forgot to shift left the `_varParameters` 144 bits before executing `or` operation. 

This will make the value of `binStep`, ..., `maxVolatilityAccumulated` incorrect, and also remove the value (make the bit equal to 0) of `volatilityAccumulated`, ..., `time`.

## Impact
* Incorrect fee calculation when executing an action with LBPair (swap, flashLoan, mint)
* Break the functionality of LBPair. The user can't swap/mint/flashLoan
--> Make all the tokens stuck in the pools 

## Proof of concept 
Here is our test script to describe the impacts 
* https://gist.github.com/WelToHackerLand/012e44bb85420fb53eb0bbb7f0f13769

You can place this file into `/test` folder and run it using 
```bash=
forge test --match-contract High1Test -vv
```

Explanation of test script:
1. First we create a pair with `binStep = DEFAULT_BIN_STEP = 25`
2. We do some actions (add liquidity -> mint -> swap) to increase the value of `volatilityAccumulated` from `0` to `60000`
3. We call function `factory.setFeeParametersOnPair` to set new fee parameters. 
4. After that the value of `volatilityAccumulated` changed to value `0` (It should still be unchanged after `factory.setFeeParametersOnPair`) 
5. We check the value of `binStep` and it changed from`25` to `60025` 
    * `binStep` has that value because [line 915](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L915) set `binStep = uint16(volatilityAccumulated) | binStep = 60000 | 25 = 60025`. 
6. This change of `binStep` value will break all the functionality of `LBPair` cause `binStep > Constant.BASIS_POINT_MAX = 10000` 
--> `Error: BinStepOverflows` 


## Tools Used
Foundry 
 
## Recommended Mitigation Steps
Modify function `LBPair._setFeesParaters` as follow: 
```solidity=
/// url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L905-L917
function _setFeesParameters(bytes32 _packedFeeParameters) internal {
    bytes32 _feeStorageSlot;
    assembly {
        _feeStorageSlot := sload(_feeParameters.slot)
    }


    uint256 _varParameters = _feeStorageSlot.decode(type(uint112).max, _OFFSET_VARIABLE_FEE_PARAMETERS);
    uint256 _newFeeParameters = _packedFeeParameters.decode(type(uint144).max, 0);


    assembly {
        sstore(_feeParameters.slot, or(_newFeeParameters, shl(144, _varParameters)))
    }
}
```

***************************
***************************
# [H-03] Wrong calculation in function `LBRouter._swapSupportingFeeOnTransferTokens` make amountOut of swap less than expected 

## Lines of code
* https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L891
* https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L896

## Vulnerable detail 
Function `LBRouter._swapSupportingFeeOnTransferTokens` is a helper function to swap exact tokens supporting for a fee on transfer tokens. This function will check the pair of `_token` and `_tokenNext` is `JoePair` or `LBPair` using `_binStep`.
* If `_binStep == 0`, it will be a `JoePair` otherwise it will be an `LBPair`.
```solidity=
// url = https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L887-L903
if (_binStep == 0) {
    (uint256 _reserve0, uint256 _reserve1, ) = IJoePair(_pair).getReserves();
    if (_token < _tokenNext) {
        uint256 _balance = _token.balanceOf(_pair);
        uint256 _amountOut = (_reserve1 * (_balance - _reserve0) * 997) / (_balance * 1_000);


        IJoePair(_pair).swap(0, _amountOut, _recipient, "");
    } else {
        uint256 _balance = _token.balanceOf(_pair);
        uint256 _amountOut = (_reserve0 * (_balance - _reserve1) * 997) / (_balance * 1_000);


        IJoePair(_pair).swap(_amountOut, 0, _recipient, "");
    }
} else {
    ILBPair(_pair).swap(_tokenNext == ILBPair(_pair).tokenY(), _recipient);
}
```
As we can see when `_binStep == 0` and `_token < _tokenNext` (in another word  we swap through `JoePair` and pair's`token0` is `_token` and `token1` is `_tokenNext`), it will 
1. Get the reserve of pair (`reserve0`, `reserve1`)
2. Get the current `_token`'s balance 
3. Calculate the `_amountOut` by using the formula 
```
_amountOut = (_reserve1 * (_balance - _reserve0) * 997) / (_balance * 1_000)
```
4. Call `IJoePair(_pair).swap(0, _amountOut, _recipient, "")`

But the formula to calculate the `amountOut` seem weird. Is it true? 
Let's do the math here to confirm whether it's true or not. 

**Input:** 
* `_reserve0 (r0)`: reserve of token0 in pair 
* `_reserve1 (r1)`: reserve of token1 in pair 
* `_balance (b)`: balance of token0 after transfering `_token` to pair.
 
**Output:** 
* `amountOut'`: the amount of token1 we can get. 

**Generate Formula** 
Cause `JoePair` [takes 0.3%](https://help.traderjoexyz.com/en/welcome/faq-and-help/general-faq#what-are-trader-swap-joe-fees) of `amountIn` as fee, we get 
* `amountIn = b - r0`
* `amountDeductFee = 0.997 * amountIn`
* `balanceDeductFee = r0 + amountDeductFee`

Following the [constant product formula](https://docs.uniswap.org/protocol/V2/concepts/protocol-overview/glossary#constant-product-formula), we have 
```
    r0 * r1 = balanceDeductFee * (r1-amountOut)
<=> amountOut' = r1 - r0 * r1 / balanceDeductFee 
<=> amountOut' = (r1 * balanceDeductFee - r0 * r1) / balanceDeductFee
<=> amountOut' = (r1 * (r0 + amountDeductFee) - r0 * r1) / balanceDeductFee
<=> amountOut' = (r1 * amountDeductFee) / balanceDeductFee 
<=> amountOut' = r1 * 0.997 * (b - r0) / (r0 + 0.997 * amountIn) 
<=> amountOut' = (r1 * (b - r0) * 997) / (r0 * 1000 + 997 * amountIn)
<=> amountOut' = (r1 * (b - r0) * 997) / (b * 997 + 3 * r0)
<=> amountOut' = (_reserve1 * (_balance - _reserve0) * 997) / (_balance * 997 + 3 * _reserve0)
```

As we can see `amountOut'` is different from `amountOut`, the denominator of `amountOut` is `_balance * 1000` when the denominator of `amountOut'` is `_balance * 997 + 3 * _reserve0`

## Impact
As we can see above, `amountOut < amountOut'` (cause `_balance * 1000 > _balance * 997 + 3 * _reserve0`), so user will get a smaller amount out than expected.

## Proof of concept 
We can check function `_swapSupportingFeeOnTransferTokens` in `JoeRouter02.sol` following this link 
* https://github.com/traderjoe-xyz/joe-core/blob/e9a5757d87d60858eee2858afd2fb5a63498fc56/contracts/traderjoe/JoeRouter02.sol#L361-L383

```solidity=
# JoeRouter02.sol
// https://github.com/traderjoe-xyz/joe-core/blob/e9a5757d87d60858eee2858afd2fb5a63498fc56/contracts/traderjoe/JoeRouter02.sol#L374-L375
function _swapSupportingFeeOnTransferTokens(address[] memory path, address _to) internal virtual {
    ... 
    amountInput = IERC20Joe(input).balanceOf(address(pair)).sub(reserveInput);
    amountOutput = JoeLibrary.getAmountOut(amountInput, reserveInput, reserveOutput);
    ...
}

# JoeLibrary.sol
// https://github.com/traderjoe-xyz/joe-core/blob/e9a5757d87d60858eee2858afd2fb5a63498fc56/contracts/traderjoe/libraries/JoeLibrary.sol#L63-L74
function getAmountOut(
    uint256 amountIn,
    uint256 reserveIn,
    uint256 reserveOut
) internal pure returns (uint256 amountOut) {
    require(amountIn > 0, "JoeLibrary: INSUFFICIENT_INPUT_AMOUNT");
    require(reserveIn > 0 && reserveOut > 0, "JoeLibrary: INSUFFICIENT_LIQUIDITY");
    uint256 amountInWithFee = amountIn.mul(997);
    uint256 numerator = amountInWithFee.mul(reserveOut);
    uint256 denominator = reserveIn.mul(1000).add(amountInWithFee);
    amountOut = numerator / denominator;
}
```
As we can see from code above, the formula to calculate amountOut is the same as the calculation I described in **Vulnerable detail** section.

## Tools Used
Manual review 
 
## Recommended Mitigation Steps
Modify function `LBRouter._swapSupportingFeeOnTransferTokens` as follow
```solidtity=
if (_binStep == 0) {
    (uint256 _reserve0, uint256 _reserve1, ) = IJoePair(_pair).getReserves();
    if (_token < _tokenNext) {
        uint256 _balance = _token.balanceOf(_pair);
        
        // fix here 
        uint256 _amountOut = (_reserve1 * (_balance - _reserve0) * 997) / (_balance * 997 + 3 * _reserve0);
        IJoePair(_pair).swap(0, _amountOut, _recipient, "");
    } else {
        uint256 _balance = _token.balanceOf(_pair);
        // fix here 
        uint256 _amountOut = (_reserve0 * (_balance - _reserve1) * 997) / (_balance * 997 + 3 * _reserve1);
        IJoePair(_pair).swap(_amountOut, 0, _recipient, "");
    }
} else {
    ILBPair(_pair).swap(_tokenNext == ILBPair(_pair).tokenY(), _recipient);
}
```

***************************
***************************
# [LOW] QA report 

## Summary 
| Id | Issues |
| -------- | -------- | 
| [Q-01]    | Function `LBQuoter.findBestPathFromAmountIn` doesn't support `_route` contains 2 identical consecutive pair of tokens     |
| [Q-02]    | Forget to checking `ids.length` and `_amounts.length` when call function `LBPair.burn()`    |
| [Q-03]    | Should revert when `_tokenPath[0] != wavax` in function `LBRouter.swapExactAVAXForTokens`     |
| [Q-04]    | Should revert when `uint256(_id) > type(uint24).max` in function `BinHelper.getPriceFromId`     |
| [Q-05]    | Should revert when `tokenX == tokenY` in function `LBPair.initialize`     |


## QA report 

### [Q-01] Function `LBQuoter.findBestPathFromAmountIn` doesn't support `_route` contains 2 identical consecutive pair of tokens

**Lines of code** 
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBQuoter.sol#L54

**Detail**  
For each 2 consecutive tokens in `_route`, function `LBQuoter.findBestPathFromAmountIn` will check which type of pair (`JoePair` / `LBPair`) will return the best rate. Unfortunately for `_route` which contains 2 identical consecutive pair of tokens it may return the wrong `quote`. 
For example, a arbitrage bot need to do arbitrage with `LBPair` and `JoePair` using path `_route` = [avax, joe, avax]. It call to function `LBQuoter.findBestPathFromAmountIn` to get the best `amountOut` it can get. But function will return the same type of pair for pair [avax, joe] (it will return both `JoePair` or both `LBPair`) cause function didn't cache any changes of pair when executing a swap.
**Recommend mitigation** 
Add another function which support for these type of swap by caching the changes after swapping 


### [Q-02] Forget to checking `ids.length` and `_amounts.length` when call function `LBPair.burn()` 

**Lines of code** 
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L617-L618

**Detail** 
Function `LBPair.burn()` contains paramters 
* `_ids`: List of bin's id which the user want to remove its liquidity 
* `_amounts`: List of amount of token to burn for each bin 
Based on the purpose of these 2 arrays, it should have the same length. But in the function `burn` there is no requirement for this condition.

**Recommend Mitigation** 
Add new requirements for this condition with custom error.


### [Q-03] Should revert when `_tokenPath[0] != wavax` in function `LBRouter.swapExactAVAXForTokens`

**Lines of code**
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L407

**Detail** 
Function `LBRouter.swapExactAVAXForTokens` is used to swap exact input with tokenIn = wavax. So it should revert when `_tokenPath[0] != avax` like in function [`LBRouter.swapAVAXForExactTokens`](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L507) and function [`swapExactAVAXForTokensSupportingFeeOnTransferTokens`](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBRouter.sol#L601)


### [Q-04] Should revert when `uint256(_id) > type(uint24).max` in function `BinHelper.getPriceFromId` 

**Lines of code** 
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/BinHelper.sol#L40

**Detail** 
The value of `_id` is always < 2^24 follow the [whitepaper](https://github.com/traderjoe-xyz/LB-Whitepaper/blob/main/Joe%20v2%20Liquidity%20Book%20Whitepaper.pdf) so it should revert when `_id > type(uint24).max` like in function [`BinHelper.getIdFromPrice`](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/BinHelper.sol#L29).


### [Q-05] Should revert when `tokenX == tokenY` in function `LBPair.initialize`

**Lines of code** 
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L104-L110

**Detail** 
Pair of `tokenX` and `tokenY` is allowed to create/intialize when `tokenX != tokenY` and both of them are not `address(0)`, but function `LBPair.initialize()` haven't checked `tokenX != tokenY` or not 

**Mitigation recommendation** 
Revert with custom error when `tokenX == tokenY`

