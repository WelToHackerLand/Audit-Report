# Sherlock / volta-feb-15
***************************
***************************

# [H-01] Price of `voltGNS` share is calculated incorrectly
###### tags: `sherlock`, `2023-02-volta`, `high`

## Summary
Forget to deduct the `boostedGNS` when calculating price
 
## Vulnerability Detail
The price of voltGNS is updated whenever the function `voltGNS.swapToGNS()` is called. 
```solidity=
/// url = https://github.com/sherlock-audit/2023-02-volta-WelToHackerLand/blob/f5062bf50be95e2d49c3f68cc245997ef6266ac8/contracts/voltGNS.sol#L189-L205
function swapToGNS() internal {
    if(pendingDAI < 1e16) return; //0.01 DAI
    if(totalSupply() == 0) return;


    uint balanceBefore = gns.balanceOf(address(this));


    dai.approve(address(gnsRouter), pendingDAI);
    gnsRouter.uniswapV3Swap(pendingDAI, 0, routerPools);


    uint gnsBought = gns.balanceOf(address(this)) - balanceBefore;

    GNSVault.User memory u = gnsVault.users(address(this));
    uint totalStaked = gnsBought + u.stakedTokens;


    price = totalStaked * 1e18 / totalSupply();
    pendingDAI = 0;
}
```
The function firstly calculates the `totalStaked` which is amount of gns staking in `gnsVault` plus with amount of gns which was swapped from `pendingDai`. Then the price will be determined by dividing `totalStaked` with `totalSupply()` of the vault. 

At the first glance, this formula seem right when calculating the price using the total underlying assets and the total shares supply. So it means that if I withdraw all the `totalSupply` shares, I can receive all the `totalStaked`. Unfortunately this won't be happened because of the `boostedGNS`. The `boostedGNS` is amount of gns which is deposited by the owner and no share will be minted for the him. Furthermore this amount can be deposited / withdrew at anytime whenever the owner wants.

This `boostedGNS` mechanism can lead to the big issue for the `VoltaVault` contract which uses the `voltGNS` as collateral. When the owner calls `unboostGNS()`, it can make the `voltGNS.price()` lose a lot of value. That will make the big liquidation events occured for a lot of users without noticing them. 

The price of a token should depend on the fluctuation of the market, and should not depend on the single individual like owner. Malicious owner can abuse this to make profit. 


## Impact
When owner calls `unboostGNS()` can incur a immediate liquidation for users without their preparation.

## Code Snippet
https://github.com/sherlock-audit/2023-02-volta-WelToHackerLand/blob/f5062bf50be95e2d49c3f68cc245997ef6266ac8/contracts/voltGNS.sol#L201-L203

## Tool used
Manual review 

## Recommendation
Deducting the `boostedGNS` out of `totalStaked` when calculating the price. 


***************************
***************************
# [H-02] First voltGNS's depositor can manipulate price to steal other depositors's fund
###### tags: `sherlock`, `2023-02-volta`, `high`

## Summary
When a users A deposit an amount of gns into voltGNS, first depositor can transfer directly the same amount of gns into voltGNS to make the shares that the user A receive be equalt to 0.
 
## Vulnerability Detail
Function `voltGNS.totalAssets()` is implemented as follows:
```solidity=
function totalAssets() public view virtual override returns (uint256) {
    GNSVault.User memory u = gnsVault.users(address(this));
    return u.stakedTokens + gns.balanceOf(address(this)) - boostedGNS;
} 
```
As we can see that the totalAssets is comprised by 2 components: 
* `u.stakedTokens - boostedGNS`: amount of gns staking in gnsVault which was contributed by users (not owner)
* `gns.balanceOf(address(this))`: amount of gns gifted to the contract. 

The attackers can abuse the second component (`gns.balanceOf(address(this))`) to manipulate the value of `totalAssets()` by transferring gns directly into the vault. From that they can easily manipulate the price of voltGNS's share.

Here is the strategy for the first depositor to steal all the fund of other depositors. 

0. The voltGNS is empty right now (`totalSupply = totalAsset = 0`)
1. Attacker calls `voltGNS.deposit(1, addr)`, he gets 1 shares in return
    * `totalSupply = 1`
    * `totalAsset = 1`
2. Bob wants to deposit x gns into voltGNS and expect that he can get x shares 
3. Unfortunately, attacker is signaled about Bob's transaction. He front-run Bob's transaction by transferring x gns into voltGNS. 
    * `totalSupply = 1`
    * `totalAsset = 1 + x`
4. At the time Bob's tx is executed, he get: 
    * `shares = x * totalSupply / totalAsset = x * 1 / (x + 1) = 0`

After the step 4 we can see that Bob loses his x tokens and gains no shares in return. 

## Impact
First depositor can steal fund of other depositors

## Code Snippet
https://github.com/sherlock-audit/2023-02-volta-WelToHackerLand/blob/f5062bf50be95e2d49c3f68cc245997ef6266ac8/contracts/voltGNS.sol#L260-L263

## Tool used
Manual review 

## Recommendation
Consider to use the mechanism used in UniswapV2 which mints for address(0) first `MIN_LIQUIDITY` liquidity. 
https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L121
