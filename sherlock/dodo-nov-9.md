# Sherlock / dodo-nov-9
***************************
***************************

# [M-01] Rounding error when call function `dodoMultiswap()` can lead to revert of transaction or fund of user 

## Summary
The calculation of the proportion when do the split swap in function `_multiSwap` doesn't care about the rounding error 

## Vulnerability Detail
The amount of `midToken` will be transfered to the each adapter can be calculated by formula `curAmount = curTotalAmount * weight / totalWeight`
```solidity=
if (assetFrom[i - 1] == address(this)) {
    uint256 curAmount = curTotalAmount * curPoolInfo.weight / curTotalWeight;


    if (curPoolInfo.poolEdition == 1) {
        //For using transferFrom pool (like dodoV1, Curve), pool call transferFrom function to get tokens from adapter
        IERC20(midToken[i]).transfer(curPoolInfo.adapter, curAmount);
    } else {
        //For using transfer pool (like dodoV2), pool determine swapAmount through balanceOf(Token) - reserve
        IERC20(midToken[i]).transfer(curPoolInfo.pool, curAmount);
    }
}
```
It will lead to some scenarios when `curTotalAmount * curPoolInfo.weight` is not divisible by `curTotalWeight`, there will be some token left after the swap.

For some tx, if user set a `minReturnAmount` strictly, it may incur the reversion. 
For some token with small decimal and high value, it can make a big loss for the sender. 

## Impact
* Revert the transaction because not enough amount of `toToken`
* Sender can lose a small amount of tokens 

## Code Snippet
https://github.com/sherlock-audit/2022-11-dodo-WelToHackerLand/blob/24cf581717338942ea0f4b2176f063bc89c9520e/contracts/SmartRoute/DODORouteProxy.sol#L415-L425

## Tool used
Manual review 

## Recommendation
Add a accumulation variable to maintain the total amount is transfered after each split swap. In the last split swap, instead of calculating the `curAmount` by formula above, just take the remaining amount to swap. 