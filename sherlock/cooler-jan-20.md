# Sherlock / cooler-jan-20
***************************
***************************
# [H-01] Use `safeTransfer` instead of `transfer`

## Summary
When transferring token to user (lender, borrower), implementations use `transfer` instead of `safeTransfer`.
 
## Vulnerability Detail
Tokens not compliant with the ERC20 specification could return false from the transfer function call to indicate the transfer fails, while the calling contract would not notice the failure if the return value is not checked. Checking the return value is a requirement, as written in the [EIP-20 specification](https://eips.ethereum.org/EIPS/eip-20):
>Callers MUST handle `false` from `returns (bool success)`. Callers MUST NOT assume that `false` is never returned!


## Impact
User can't lose their tokens when the transferring return `false` 

## Code Snippet
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L85
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L102
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L122-L123
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L146
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L179
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L205
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/aux/ClearingHouse.sol#L111

## Tool used
Manual review 

## Recommendation
Use the SafeERC20 library [implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) from OpenZeppelin and call safeTransfer or safeTransferFrom when transferring ERC20 tokens.


***************************
***************************
# [H-02] Attackers can force borrower unable to repay the loan in case debt token is USDC 

## Summary
Function `Cooler.repay()` will transfer debtToken directly from the sender to the lender. It will create some risk when lender finds a way to block the transfer to him(her).
 
## Vulnerability Detail
[USDC](https://etherscan.io/address/0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf#code) is one of the popular stable coins which was widely used by many people. There is a list (mapping) in USDC called `blacklist` which block every transactions (transfer, mint) that contain at least 1 address belong to this list. 
A malicious user who has at least 1 blacklisted wallet can execute this strategy to block user repaying the loan. At first I assume the debt token we are using here is USDC. 
1. Attacker call `Cooler.clear()` with a non-blacklisted address (he can easily create a new one by using metamask) to fill a requested loan.  
2. Attacker transfer the `lender` role to `blacklisted` address by calling `Cooler.approve()` then `Cooler.transfer()`. 
==> Now the current lender of loan is `blacklisted` address
3. When borrower try to call `Cooler.roll()` to repay the debt, the transaction will be reverted because the loan.lender is USDC's blacklisted address. 

## Impact
Attacker can force user unable to repay the loan to get the collateral. 

## Code Snippet
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L122

## Tool used
Manual review 

## Recommendation
Create one more mapping to store amount repayed by the borrower then create one function for lender to call to claim the fund. 
```solidity=
mapping(address => uint) repayedAmount;

function claimRepayedAmount() external {
    debt.safeTransfer(msg.sender, repayedAmount);
    repayedAmount[msg.sender] = 0;
}
```


***************************
***************************
# [H-03] Collateral can be locked forever because borrower can `roll` the loan many times

## Summary
Function `roll` in contract Cooler.sol is used to extend the loan at origination terms. There is no limit to the number of times that borrower can roll. If the borrower rolls too many times, the collateral can be locked forever in the contract. In the cases of small interest (close to 0), borrower can roll with no cost of collateral. 
## Vulnerability Detail
Let's see function `roll` in contract Cooler.sol
```solidity=
function roll (uint256 loanID) external {
    Loan storage loan = loans[loanID];
    Request memory req = loan.request;

    if (block.timestamp > loan.expiry) 
        revert Default();

    if (!loan.rollable)
        revert NotRollable();

    uint256 newCollateral = collateralFor(loan.amount, req.loanToCollateral) - loan.collateral;
    uint256 newDebt = interestFor(loan.amount, req.interest, req.duration);

    loan.amount += newDebt;
    loan.expiry += req.duration;
    loan.collateral += newCollateral;

    collateral.transferFrom(msg.sender, address(this), newCollateral);
}
```
* The borrower can roll a loan many times because there is no limitation. 
* If the interest is too small (close to 0), `newCollateral` will be 0. Then borrower can roll with no cost of collateral, and extend the expiry to be very large.
* Contract `Cooler.sol` does not have any function to rescue the stuck tokens, so collateral will be locked forever in this case. It is very bad behavior that can not cover the accident cases from the lender.
## Impact
Collateral can be locked forever in some cases.

## Code Snippet
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L129-L147

## Tool used
Manual review 

## Recommendation
Should have a limit of the expiry when rolling a loan.

***************************
***************************
# [M-01] Because the default value of `rollable` is true, function `toggleRoll` can not block the rollover
###### tags: `sherlock`, `2023-01-cooler`, `high`

## Summary
Function `roll` in contract Cooler.sol is used to extend the loan at origination terms, and rollover can be blocked by the lender by calling the function `toggleRoll` (see [docs](https://ag0.gitbook.io/cooler-loans/servicing-and-default)). However, borrower can front-run and roll the loan before the lender calls `toggleRoll`.

## Vulnerability Detail
When lender clears a request by function `clear`, a new loan will be created as the following:
```solidity=
loanID = loans.length;
loans.push(
    Loan(req, req.amount + interest, collat, expiration, true, msg.sender)
);
```
The default value of the variable `rollable` in the new loan is true. If the lender doesn't allow the rollover, he/she must call function `toggleRoll`. However, the borrower can front-run to call function `roll` (can roll many times in 1 transaction) before, and the lender can not prevent it.

## Impact
The lender can not block the rollover.

## Code Snippet
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L177
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L191

## Tool used
Manual review 

## Recommendation
The variable `rollable` of a new loan should be false as default.
```solidity=
loans.push(
    Loan(req, req.amount + interest, collat, expiration, false, msg.sender)
);
```

***************************
***************************
# [M-02] Borrower can't roll a loan over when call `repay()` before 
###### tags: `sherlock`, `2023-01-cooler`, `high`

## Summary
Function `Cooler.roll()` is used to roll a loan over. It will calculate the new collateral that sender need to send to extend the loan. The formula to determine this value is as follow: 
```solidity=
uint256 newCollateral = collateralFor(loan.amount, req.loanToCollateral) - loan.collateral;
```
Which `collateralFor()` is implemented as: 
```solidity=
function collateralFor(uint256 amount, uint256 loanToCollateral) public pure returns (uint256) {
    return amount * decimals / loanToCollateral;
}
```
This formula works based on the assumption that the `loan.collateral` always less than `loan.amount * decimals / loanToCollateral` because the `loan.amount` is composed by the original loan amount and the interest. But this assumption won't be always true since the function `Cooler.repay()` use the rounding down when calculating variable `decollateralized`
```solidity=
uint256 decollateralized = loan.collateral * repaid / loan.amount;

if (repaid == loan.amount) delete loans[loanID];
else {
    loan.amount -= repaid;
    loan.collateral -= decollateralized;
}
```
Thus after some transactions calling `repay()`, users can't call the function `roll()` because the `collateralFor(loan.amount, req.loanToCollateral) < loan.collateral;` 

## Vulnerability Detail
To explain the vulnerability I will describe it through example below:
First I assume that the interest for the loan is 0 for easy to describe. 

* User A use 100 USDC as collateral to borrow 1000 KNC from --> `loanToCollateral = 1000 / 100 = 10` 
* User A call `Cooler.repay(loanID, repaid = 19)`
    * `decollateralized = loan.collateral * repaid / loan.amount = 100 * 19 / 1000 = 1`
    * `loan.amount = 1000 - 19 = 981`
    * `loan.collateral = 100 - 1 = 99`
* User A call `Cooler.roll(loanId)`
    * `newCollateral = collateralFor(981, 10) - 99 = 981 / 10 - 99 = 98 - 99 = -1 < 0`
==> **Revert because of underflow**

## Impact
Borrower can't roll the loan over 

## Code Snippet
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L139
https://github.com/sherlock-audit/2023-01-cooler-Trumpero/blob/55b6ca648a506a23eb4c1c919ef6ecf119ec67bf/src/Cooler.sol#L114

## Tool used
Manual review 

## Recommendation
Modify function `Cooler.roll()` as follow: 
```solidity=
function roll (uint256 loanID) external {
    Loan storage loan = loans[loanID];
    Request memory req = loan.request;

    if (block.timestamp > loan.expiry) revert Default();
    if (!loan.rollable) revert NotRollable();
    
    /// change here 
    uint256 newCollateral;
    if (collateralFor(loan.amount, req.loanToCollateral) < loan.collateral) {
        newCollateral = 0;
    } else {
        newCollateral = collateralFor(loan.amount, req.loanToCollateral) - loan.collateral;
    }
    uint256 newDebt = interestFor(loan.amount, req.interest, req.duration);

    loan.amount += newDebt;
    loan.expiry += req.duration;
    loan.collateral += newCollateral;

    collateral.transferFrom(msg.sender, address(this), newCollateral);
}
```

