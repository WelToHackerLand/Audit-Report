# Sherlock / phuture
***************************
***************************
# [H-01] Should implement a periphery contract for user to mint indexToken  

###### tags: `c4`, `2022-04-phuture`

## Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/IndexLogic.sol#L31
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/IndexLogic.sol#L96
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/IndexLogic.sol#L48


## Impact
User can lose their fund 

## Proof of Concept 
When users want to mint an index token, users need to transfer their assets to address(vToken) first, then call the ```mint()``` function of [IndexLogic.sol](https://github.com/code-423n4/2022-04-phuture/blob/main/contracts/IndexLogic.sol). If users make it into 2 transaction, miner can manipulate it/ After users transfer their token to address(vToken), miner can front-run, and call ```mint()``` before users call ```mint()```, and of course the index token will be minted to miner instead of users 
==> User will lose their fund 

## Tools Used
manual review 

## Recommended Mitigation Steps
Implement a contract to help user execute 2 action (transfer to vToken + mint) into 1 transaction (like router in uniswapV2)

***************************
***************************
# [M-01] Wrong requirement in reweight function (ManagedIndexReweightingLogic.sol)

###### tags: `c4`, `2022-04-phuture`

## Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/ManagedIndexReweightingLogic.sol#L32
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/interfaces/IIndexRegistry.sol#L19


## Impact
The list of assets won't be changed after reweight because of reverted tx 

## Proof of Concept 
```require(_updatedAssets.length <= IIndexRegistry(registry).maxComponents())```
when [reweight](https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/ManagedIndexReweightingLogic.sol#L32) is not true, because as in the [doc](https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/interfaces/IIndexRegistry.sol#L19), 
```maxComponent``` is the maximum assets for an index, but ```_updatedAssets``` also contain the assets that you want to remove. So the comparision make no sense

## Tools Used
manual review 

## Recommended Mitigation Steps
Require ```assets.length() <= IIndexRegistry(registry).maxComponents()``` at the end of function instead 

***************************
***************************
# [M-02] Wrong shareChange formula (vToken.sol)

## Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/vToken.sol#L160

## Impact
Users can get wrong amount of vToken 
=> Make users lose their fund 

## Proof of Concept
Base on the code in function ```shareChange()``` in [vToken.sol](https://github.com/code-423n4/2022-04-phuture/blob/main/contracts/vToken.sol)
Assume that if ```oldShare = totalSupply > 0```, 
* ```newShares``` 
= ```(_amountInAsset * (_totalSupply - oldShares)) / (_assetBalance - availableAssets);```
= ```(_amountInAsset * (_totalSupply - _totalSupply)) / (_assetBalance - availableAssets);```
= ```0```

It make no sense, because if ```amountInAsset >> availableAssets```, ```newShares``` should be bigger than ```oldShares```, but in this case ```newShares = 0 < oldShares```

## Tools Used
manual review 

## Recommended Mitigation Steps
Modify the [line](https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/vToken.sol#L160) from ```if (_totalSupply > 0)``` to ```if (_totalSupply - oldShares) > 0```

***************************
***************************
# [LOW] QA report 

### typo comment 
* idd -> id 
    * https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashBase.sol#L97
* wether -> whether 
    * https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/index-coop-notional-trade-module/contracts/protocol/modules/v1/NotionalTradeModule.sol#L111
* updateable -> updatable 
    * https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/index-coop-notional-trade-module/contracts/protocol/modules/v1/NotionalTradeModule.sol#L114
    * https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/index-coop-notional-trade-module/contracts/protocol/modules/v1/NotionalTradeModule.sol#L117


### unused private function 
* https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L243-L248

### safeApprove is deprecated 
Affected code: 
* https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashBase.sol#L68
* https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashBase.sol#L73

Proof of concept
* https://github.com/OpenZeppelin/openzeppelin-contracts/blob/5a75065659a65e65bb04890192e3a4bcb7917fff/contracts/token/ERC20/utils/SafeERC20.sol#L39

***************************
***************************
# [GAS] Gas optimization

#### Use memory variable to store ```decimals()```
Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/IndexLogic.sol#L78
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/IndexLogic.sol#L79

#### Use ```newWeight != 0``` instead of ```newWeight > 0```
Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/ManagedIndexReweightingLogic.sol#L61

#### Use ```_totalSupply != 0``` instead of ```_totalSupply > 0``` 
Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/vToken.sol#L160

#### Use ```uint _balance = oldShares``` instead of ```uint _balance = _NAV.balanceOf[_account]```
Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/vToken.sol#L161

#### Add condition (i != 0) to if block 
Affected code 
* https://github.com/code-423n4/2022-04-phuture/blob/594459d0865fb6603ba388b53f3f01648f5bb6fb/contracts/TrackedIndex.sol#L38

Proof of concept
* Because ```maxCapitalization``` is initialized with value ```_capitalization[0]```, so we don't need to check if ```_capitalizations[i] > maxCapitalization``` if ```i == 0```