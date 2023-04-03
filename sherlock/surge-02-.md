# Sherlock / volta-feb-15
***************************
***************************

# [H-01] User can borrow loan token without having enough collateral tokens 

## Summary
The value of mantissa (10^18) isn't big enough for some pairs of tokens which have the tremendous difference in value. 
 
## Vulnerability Detail
The collateralRatioMantissa is calculated by dividing the userDebt * 10^18 with collateral balance of the borrower. This calculation can't be applied for some pairs like BTC and SHIB, because this ratio will return the value 0 even the value of collateral is less than the debt value. 

For instance: 
* **LOAN_TOKEN:** BTC 
    * price: 20000$
    * decimals: 10^8
* **COLLATERAL_TOKEN:** SHIB 
    * price: 0.00001$ 
    * decimals: 10^18

Obviously when a user wants to borrow 10^8 BTC, they will need at least `20000 / 0.00001 * 10^18 = 2 * 10^27` SHIB as collateral. But applying this case into the current implementation, the borrowers can borrow 10^8 BTC with less than 2 * 10^27 SHIB. To prove the statement, I will assume that `collateralBalanceOf[borrower] = 10^27 < 2 * 10^27`
The function `Pool.borrow()` is as follows: 
```solidity=
/// url = https://github.com/sherlock-audit/2023-02-surge-WelToHackerLand/blob/c535f3c31aaeefa5e50b35bb92f85708a414d47b/surge-protocol-v1/src/Pool.sol#L455-L498
function borrow(uint amount) external {
    /// ... 
    
    uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalanceOf[msg.sender];
    require(userCollateralRatioMantissa <= _currentCollateralRatioMantissa, "Pool: user collateral ratio too high");

    /// ... 
}
```
As we can see that the loan is accepted if `userCollateralRatioMantissa <= _currentCollateralRatioMantissa`, in my example: 
* `userDebt = 10^8`
* `collateralBalanceOf[msg.sender] = 10^27`
* `userCollateralRatioMantissa = 10^8 * 10^18 / 10^27 = 0 <= _currentCollateralRatioMantissa`
==> The condition satisfied

Borrower can use 10^27 SHIB (`10000$`) to borrow 10^8 BTC (`20000$`) to make the `10000$` profit 

## Impact
Users can borrow the loan tokens without having enough collateral tokens 

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge-WelToHackerLand/blob/c535f3c31aaeefa5e50b35bb92f85708a414d47b/surge-protocol-v1/src/Pool.sol#L474-L475

## Tool used
Manual review 

## Recommendation
Consider to let pool's creator define the value of mantissa

***************************
***************************
# [H-02] First depositor of pool can abuse rounding error to steal tokens

## Summary
First depositor can transfer directly an amount of LOAN_TOKEN into pool to manipulate the price of shares.
 
## Vulnerability Detail
Function `Pool.deposit()` is defined as 
```solidity=
// url = https://github.com/sherlock-audit/2023-02-surge-WelToHackerLand/blob/c535f3c31aaeefa5e50b35bb92f85708a414d47b/surge-protocol-v1/src/Pool.sol#L307-L343
function deposit(uint amount) external {
    uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));

    /// ... 

    uint _shares = tokenToShares(amount, (_currentTotalDebt + _loanTokenBalance), _currentTotalSupply, false);
    require(_shares > 0, "Pool: 0 shares");
    
    /// ... 
}
```
which will call `tokenToShares()` to calculate the number of shares
```solidity=
function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
    if(_supplied == 0) return _tokenAmount;
    uint shares = _tokenAmount * _sharesTotalSupply / _supplied;

    if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) 
        shares++;
    return shares;
}
```
This function takes into account the `_supplied` which is comprised of the total debt and the balance of LOAN_TOKEN in pool (LOAN_TOKEN.balanceOf(address(this))). 

However, due to rounding errors, an attacker who deposits to a fresh vault can skew the ratio by transferring in a large quantity of LOAN_TOKEN by following the strategy 
1. Alice deposits 1 LOAN_TOKEN, receive 1 share in return
2. Alice then sends a large amount 1e18 wei of LOAN_TOKEN to the vault.
3. Bob comes and deposits 2e18 LOAN_TOKEN, he will receive `shares = 2e18 * 1 / (1e18+1) = 1`
4. Bob only gets minted 1 share, giving him claim to only half the vault, which has `3e18` tokens, instantly losing him `0.5e18` tokens which is profited by Alice.

## Impact
Exploit by first user of pool. Loss of tokens by other users

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge-WelToHackerLand/blob/c535f3c31aaeefa5e50b35bb92f85708a414d47b/surge-protocol-v1/src/Pool.sol#L308

## Tool used
Manual review 

## Recommendation
Burn the first (if `_supplied == 0`) 10000 wei of minted shares by sending them to address 0. This will make the cost to pull of this attack unfeasable.

***************************
***************************
# [M-01] Borrower can call `getCurrentState()` to make the interest can't be accrued 

## Summary
The `interest` is updated based on the duration between 2 consecutive function call `getCurrentState()`, which can be abused by attacker to bypass the interest 
 
## Vulnerability Detail
The `interest` is calculated in function `Pool.getCurrentState()` as follows: 
```solidity=
uint _interest = _totalDebt * _borrowRate * _timeDelta / (365 days * 1e18); // does the optimizer optimize this? or should it be a constant?
```
In case `totalDebt * _borrowRate * _timeDelta < 365 days * 1e18`, the `_interest` will get 0 value. Since the `lastAccrueInterestTime` is still set to `block.timestamp` even if the `_interest = 0`, the borrower can abuse this by calling function `Pool.getCurrentState()` in each block minted to make the `interest = 0` for the debt. 

With some tokens which has the big value and small decimal, it will be worth for the borrowers to pay the gas cost to call the function in each blocks rather than taking the interest. 

Note that, users can't directly call function `Pool.getCurrentState()` because it is a private function. But the user can call `Pool.removeCollateral(0)` instead to accrue the interest and no need to remove any their actual collateral 

## Impact
The borrower can bypass the interest

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge-WelToHackerLand/blob/c535f3c31aaeefa5e50b35bb92f85708a414d47b/surge-protocol-v1/src/Pool.sol#L154

## Tool used
Manual review 

## Recommendation
Use one more storage variable to store the timestamp which the `_interest != 0`. And use this variable to determine the deltaTime in interest calculation instead of `lastAccrueInterestTime`.

***************************
***************************

***************************
***************************