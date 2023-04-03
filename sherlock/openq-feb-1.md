# Sherlock / openq-feb-1
***************************
***************************
# [H-01] Missing check of deposit's endTime lead to unable to refundDeposit 

## Summary
Since there is no check about the end time of the deposits when a user call `DepositManagerV1.fundBountyToken()`, a attacker can abuse this function by depositing into bounty a token with `expiration = type(uint256).max`. 
This will lead to the DDOS attack with function `DepositManagerV1.refundDeposit()`. When a user try to call refund, the function will loop through the entire array `deposits[]` to calculate the total locked fund. Then when the loop reachs to the deposit of the attacker, it will calculate `depositTime[depList[i]] + expiration[depList[i]]` and compare it with `block.timestamp`. Unfortunately `depositTime[depList[i]] + expiration[depList[i]] > 1 + 2^256 - 1 = 2^256` ==> overflow here

## Vulnerability Detail
You can place this test into `DepositManager.test.js` -> `describe('refundDeposits')` -> `describe('Event Emissions')` -> ``
```javascript=
it.only('should revert when expiration = type(uint256).max', async () => {
    // ARRANGE
    await openQProxy.mintBounty(Constants.bountyId, Constants.organization, atomicBountyInitOperation);

    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
    const bounty = await AtomicBountyV1.attach(bountyAddress);

    await mockLink.approve(bountyAddress, 10000000);
    const volume = 100;

    const depositedTimestamp = await setNextBlockTimestamp(10);

    /// user A deposit link with expiration = 1 
    const tokenDepositId = generateDepositId(Constants.bountyId, 0);
    await depositManager.fundBountyToken(bountyAddress, mockLink.address, volume, 1, Constants.funderUuid);

    /// user B deposit native token with expiration = 2^256 - 1
    const protocolDepositId = generateDepositId(Constants.bountyId, 1);
    await depositManager.fundBountyToken(bountyAddress, ethers.constants.AddressZero, volume, BigNumber.from(2).pow(256).sub(1), Constants.funderUuid, { value: volume });

    // revert because of overflow error 
    const expectedTimestamp = await setNextBlockTimestamp(2764800);
    await depositManager.refundDeposit(bountyAddress, tokenDepositId);
});
```

## Impact
Funders can't call refund when the deposit expired.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/Bounty/Implementations/BountyCore.sol#L342-L346

## Tool used
Hardhat 

## Recommendation
Check the endTime = startTime + expiration when funding the bounty or extending a deposit. 

***************************
***************************
# [H-02] Bounty doesn't work as expected with token revert transferring with amount = 0

## Summary
There are some type of ERC20 which revert the transferring when amount = 0

## Vulnerability Detail
`TieredPercentageBountyV1` is a contest which pays a percentage amount to multiple developers one time  based on each claimants' tier, e.g. in a hackathon with 1st, 2nd and 3rd place. 
When a winner claim their rewards, for each token the bounty will calculate the amount of tokens corresponding to the percentage of the tier can gain by formula: 
```solidity=
uint256 claimedBalance = (payoutSchedule[_tier] * fundingTotals[_tokenAddress]) / 100;
```
This `claimedBalance` can be equal to 0 when `payoutSchedule[_tier] * fundingTotals[_tokenAddress] < 100`. This will make the function `TieredPercentageBountyV1.claimTiered()` revert in case the `_tokenAddress` is a [weird ERC20](https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers) which revert when transferring 0 amount. Then it will incur the failure of function `ClaimManagerV1._claimTieredPercentageBounty()` when a winner are trying to claim their rewards. 
Note that this issue appear in `AtomicBounty` and `TieredFixedBounty` when the balance of `payoutAddress` is equal to 0 due to the refunding. 

## Impact
User can't claim the rewards

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/Bounty/Implementations/BountyCore.sol#L221-L228

## Tool used
Manual review 

## Recommendation
Check the amount > 0 before transferring 

***************************
***************************
# [H-03] Attacker can fund malicious tokens to make competitor unable to claim their reward. 

## Summary
In AtomicBounty, when paying the reward for closer, the bounty needs to loop through the entire `tokenAddress[]` array. So if there is a failed transfer in a iteration, the entire process will revert. 
```solidity=
// contract: ClaimManagerV1.sol 

function _claimAtomicBounty(
    IAtomicBounty _bounty,
    address _closer,
    bytes calldata _closerData
) internal {
    _eligibleToClaimAtomicBounty(_bounty, _closer);

    /// [#explain] claim ERC20  
    for (uint256 i = 0; i < _bounty.getTokenAddresses().length; i++) {
        uint256 volume = _bounty.claimBalance(
            _closer,
            _bounty.getTokenAddresses()[i] /// [$audit-med] out-of-gas here  
        );

        /// ...
    }

    /// [#explain] claim ERC721
    /// ... 
}
```
Since the bounty allows funders to deposit into the bounty unwhitelisted tokens if the token addresses is not reached, attacker can abuse this techinique by creating a malicious ERC20 token. This malicious token will revert when transferring from bounty to user, which will make a iteration in the loop of function `ClaimManagerV1._claimAtomicBounty` failed then make the whole function call revert. 

## Vulnerability Detail
Attacker can deploy a malicious ERC20 which reverts every `transfer()` call (not `transferFrom()`) like below: 
```solidity=
pragma solidity 0.8.17;

import '@openzeppelin/contracts/token/ERC20/ERC20.sol';

contract MockMaliciousERC20 is ERC20 {
    address public admin;

    constructor() ERC20('Mock MAL', 'mMAL') {
        _mint(msg.sender, 10000 * 10**18);
        admin = msg.sender;
    }
    
    /// [#explain] revert every transfer tx 
    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        require(0 == 1, "Iam a malicious ERC20");
    }
}
```
Here is the test scripts: 
```typescript=
it.only('should revert when there is malicious token in bounty rewards', async() => {
    // attacker deploy malicious tokens 
    const MockMal = await ethers.getContractFactory('MockMaliciousERC20');
    const mockMal = await MockMal.deploy();
    const attacker = notOwner;

    // ARRANGE
    const volume = 100;
    await openQProxy.mintBounty(Constants.bountyId, Constants.organization, atomicBountyInitOperation);

    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);

    await mockLink.approve(bountyAddress, 10000000);
    await mockDai.approve(bountyAddress, 10000000);
    await mockMal.approve(bountyAddress, 10000000);

    /// fund bounty with LINK, DAI and Malicious token
    await depositManager.fundBountyToken(bountyAddress, mockLink.address, volume, 1, Constants.funderUuid);
    await depositManager.fundBountyToken(bountyAddress, mockDai.address, volume, 1, Constants.funderUuid);
    await depositManager.fundBountyToken(bountyAddress, mockMal.address, volume, 1, Constants.funderUuid);

    await expect(
        claimManager.connect(oracle).claimBounty(bountyAddress, claimant.address, abiEncodedSingleCloserData)
    ).to.be.revertedWith("Iam a malicious ERC20");
});
```
You can place it in `ClaimManager.test.js` -> `describe('ClaimManager.sol')` -> `describe('claimBounty')` -> `describe('ATOMIC')` -> `describe('TRANSFER')`

## Impact
Competitors who join the bounty can't claim their rewards. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/DepositManager/Implementations/DepositManagerV1.sol#L45-L50
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L130-L148
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L230-L249

## Tool used
Hardhat 

## Recommendation
Consider to just deal with whitelisted token. 
Or you can define a new storage array `additionalTokens[]` which will be modified by the bounty's issuer and let these token permission to be deposited into bounty. 

***************************
***************************
# [H-04] Attacker can create a draft tokens and spam deposit to make function `DepositManagerV1.refundDeposit()` out-of-gas.
###### tags: `sherlock`, `2023-02-openq`, `high`

## Summary
Function `DepositManagerV1.refundDeposit()` loops through entire `bounty.deposit[]` array to get the total locked fund which can be abused to lead to DDOS
 
## Vulnerability Detail
Function `DepositManagerV1.refundDeposit()` is used to refund an individual deposit from bountyAddress to sender if expiration time has passed. It calculates the `availableFunds` from balance and the total locked funds of `_bountyAddress` then transfer this amount to the `msg.sender`. 
The calculation of total locked funds is implemented as follows: 
```solidity=
function getLockedFunds(address _depositId)
    public
    view
    virtual
    returns (uint256)
{
    uint256 lockedFunds;
    bytes32[] memory depList = this.getDeposits();
    for (uint256 i = 0; i < depList.length; i++) {
        if (
            block.timestamp <
            depositTime[depList[i]] + expiration[depList[i]] &&
            tokenAddress[depList[i]] == _depositId
        ) {
            lockedFunds += volume[depList[i]];
        }
    }

    return lockedFunds;
}
```
This implementation will loops through the entire array `depList[]` to sum up all the volume of unexpired deposit. The question here is can attackers make the `depList.length` too big to execute (out-of-gas). The answer is "yep they can" due to the `addressLimit` technique of function `DepositManagerV1.fundBountyToken()`. 
```solidity=
function fundBountyToken(
    address _bountyAddress,
    address _tokenAddress,
    uint256 _volume,
    uint256 _expiration,
    string memory funderUuid
) external payable onlyProxy {
    IBounty bounty = IBounty(payable(_bountyAddress));

    if (!isWhitelisted(_tokenAddress)) {
        require(
            !tokenAddressLimitReached(_bountyAddress),
            Errors.TOO_MANY_TOKEN_ADDRESSES
        );
    }
}
```
If the `_tokenAddress` is not whitelisted, it will check if the token addresses limit is reached or not. If it's not reached, the _tokenAddress will be accepted as a reward of the bounty. Based on this idea, attacker can deploy a new ERC20 token which doesn't have any value and deposit this token into the bounty as much as possible before the tokenAddresses limit reached. After executing a lot of depositing into the bounty (eg: 10M time), the `BountyCore.deposits[]` array will grow massively and make the loop in `BountyCore.getLockedFunds()` out of gas to call. 

Furthermore in case protocol doesn't accept unwhitelisted token (by setting token addresses limit = 0), attacker can spam by depositing 1 wei each deposit with some low value tokens and make other funders can't claim their expired deposits. 

## Impact
Users is unable to claim the rewards. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/DepositManager/Implementations/DepositManagerV1.sol#L171-L172
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/Bounty/Implementations/BountyCore.sol#L341

## Tool used
Manual review 

## Recommendation
Consider change the flow of bounty into 3 phases: 
**1. Phase 1: Before distribution**
During this phase, the fund hasn't claimed yet so when a user call refund, he can claim his entire deposit fund which was captured by `volume[depositId]`

**2. Phase 2: Distribution**
When this phase start, no1 can call refund 

**3. Phase 3: After distribution**
Since the bounty has finished the distribution, the locked time of deposit should be redundant to let funder claim their deposit back 

With this flow, we shouldn't need to calculate the total locked fund when call refund


***************************
***************************
# [H-05] Missing check whether the nft is refunded before transferring the reward to users
###### tags: `sherlock`, `2023-02-openq`, `high`

## Summary
Since there is no check about the nft is refunded or not before calling `_bounty.claimNft()` in function `ClaimManagerV1._claimAtomicBounty()`, it will make the user unable to claim their rewards. 
 
## Vulnerability Detail
Function `ClaimManagerV1._claimAtomicBounty()` is implemented as follows: 
```solidity=
 function _claimAtomicBounty(
        IAtomicBounty _bounty,
        address _closer,
        bytes calldata _closerData
    ) internal {
        /// [#explain] check if _close can claim the bounty 
        _eligibleToClaimAtomicBounty(_bounty, _closer);
        
        /// [#explain] claim ERC20  
        // ... 
        
        /// [#explain] claim ERC721
        for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
            _bounty.claimNft(_closer, _bounty.nftDeposits(i));

            emit NFTClaimed(
                _bounty.bountyId(),
                address(_bounty),
                _bounty.organization(),
                _closer,
                block.timestamp,
                _bounty.tokenAddress(_bounty.nftDeposits(i)),
                _bounty.tokenId(_bounty.nftDeposits(i)),
                _bounty.bountyType(),
                _closerData,
                VERSION_1
            );
        }
    }
```
Which `bounty.claimNft()` is 
```solidity=
function claimNft(address _payoutAddress, bytes32 _depositId)
        external
        virtual
        onlyClaimManager
        nonReentrant
    {
        _transferNft(
            tokenAddress[_depositId],
            _payoutAddress,
            tokenId[_depositId]
        );
    }

function _transferNft(
        address _tokenAddress,
        address _payoutAddress,
        uint256 _tokenId
    ) internal virtual {
        IERC721Upgradeable nft = IERC721Upgradeable(_tokenAddress);
        nft.safeTransferFrom(address(this), _payoutAddress, _tokenId);
    }
```
As we can see, there is no check if the bounty owns the nft or not. Then when a funder call refund his/her nft before the claim happen, the `bounty.claimNft()` will revert. 
Example: 
1. Alice deposits nft A into bounty 
2. Alice calls refund to get nft A back
3. `ClaimManagerV1._claimAtomicBounty()` was called --> Revert since the bounty isn't the owner of nft A anymore 

Note that this issue happens in `atomic`, `tieredFix`, `tieredPercentage`

## Impact
Users is unable to claim the rewards. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L150-L165
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L251-L269
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L320-L338

## Tool used
Manual review 

## Recommendation
Check whether the nft is refunded before transferring. 


***************************
***************************
# [H-06] Malicious funders can call `refundDeposit()` after a bounty is closed to make bounty doesn't have enough fund to pay for the winners. 
###### tags: `sherlock`, `2023-02-openq`, `high`

## Summary
Since there is no requirement about when the funder can call refund, they can call it right after the bounty close to make it not enough tokens to pay the competitor. 
 
## Vulnerability Detail
Function `DepositManagerV1.fundBountyToken()` just can be called when the bounty is open, so when a bounty is closed there won't be any deposit be accepted. 
```solidity=
function fundBountyToken(
        address _bountyAddress,
        address _tokenAddress,
        uint256 _volume,
        uint256 _expiration,
        string memory funderUuid
    ) external payable onlyProxy {
        /// ... 
        
        /// [#explain] check if a bounty is open or not ? 
        require(bountyIsOpen(_bountyAddress), Errors.CONTRACT_ALREADY_CLOSED);
        
        /// [#explain] process the deposit 
        (bytes32 depositId, uint256 volumeReceived) = bounty.receiveFunds{
            value: msg.value
        }(msg.sender, _tokenAddress, _volume, _expiration);

        /// ... 
    }
```
Different from the function above, [`DepositManagerV1.refundDeposit()`](https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152) can be called anytime as long as the deposit is expired. This will create a flaw for malicious funders to trick the bounty to ruin the organization's reputation. 

We will consider a scenario with a `TieredPercentageBounty`. 

0. A bounty need 1000 USDC to distribute rewards for competitors. First winner get 30% pool, second one get 70%.
1. Alice deposits 300 USDC into the bounty 
2. A Malicious funder deposits 700 USDC into bounty with `expiration = 1`. 
3. After at least 1 second, bounty's issuer see that there are enough fund to distribute the rewards to the winners, he calls claim reward for first winner with prize = 300 USDC 
4. Before the time the issuer call claim reward for the second winner, Malicious user (can front-run) called `DepositManagerV1.refundDeposit()` to claim his deposit back. 
5. Now when the issuer tries to call claim reward for the second winner, the tx will fail since there isn't enough fund to execute the claim (The balance of USDC in the contract now is 0 USDC < 1000 * 70% = 70 USDC)

Note that this issue doesn't happen just with `TieredPercentageBounty`, it also applies for the `Atomic` and `FixedTierBounty` by front-running the tx call `ClaimManagerV1.claimBounty()` to withdraw their deposit to reduce the rewards pool of the bounty.

There is a mitigation for this issue is the issuer can send directly the amount of rewards missing to the bounty (don't need to use `DepositManagerV1.fundBountyToken()`), but in-case the reward is ETH (native token), it can't be success since the contract implements a auto revert fallback function. So the problem is still remained. 

## Impact
Bounty won't have enough fund to pay for the competitor. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152

## Tool used
Manual review 

## Recommendation
Consider to divide each bounty into 3 periods: 
1. Funding period: Let funders deposit their tokens into bounty, funders can call refund during this period. 
2. Distribute period: Distribute rewards to the competitors. No1 can call refund in this phase. 
3. Refund period: Let funders claim the remaining tokens left in bounty 

***************************
***************************
# [M-01] After the bounty completely distributed, the remaining tokens can be locked into the contracts
###### tags: `sherlock`, `2023-02-openq`, `med`

## Summary
There are some cases that the reward tokens were left in the ongoingBounty because total balance exceeded the `payoutVolume` of the bounty. But no one can't claim the remaining because of underflow error. 

## Vulnerability Detail
Since the ongoing bounty doesn't restrict who can deposit into it, there are some scenarios that there is an amount of reward tokens remaining after the payout is completely distributed. 
The question is what happen with the remaining payout, who can receive it ? The answer is anyone who has deposited into the bounty before can receive the portion of the bounty based on their deposit amount and deposit tokens. Simply a user with a bounty which has the expiry time less than block.timestamp can call `DepositManagerV1.refundDeposit()` to claim `min(volume, availableFunds)` tokenAddress back. 
But in some rare cases when there are some funders locked their fund forever, users can't claim the remaining. Here is a example of this scenario: 

0. Issuer call `OngoingBountyV1.setPayout()` with `payoutVolume = 1500 USDC` 
1. Alice deposits 1000 USDC into bounty with expiry time = 1
2. Bob who is doing charity deposits 1000 USDC into bounty with expiry time = inf 
3. Competitor claims their reward --> there are `1000 + 1000 - 1500 = 500 USDC` left in the contract
4. Alice try to call `DepositManagerV1.refundDeposit()` to claim the remaining reward in the bounty. But the transaction revert when she call it. Is there something happening with the tx ? 

The `DepositManagerV1.refundDeposit()` is implemented as follows: 
```solidity=
function refundDeposit(address _bountyAddress, bytes32 _depositId)
        external
        onlyProxy
    {
        /// ... 

        address depToken = bounty.tokenAddress(_depositId);
        
        uint256 availableFunds = bounty.getTokenBalance(depToken) -
            bounty.getLockedFunds(depToken);

        uint256 volume;
        if (bounty.volume(_depositId) <= availableFunds) {
            volume = bounty.volume(_depositId);
        } else {
            volume = availableFunds;
        }

        bounty.refundDeposit(_depositId, msg.sender, volume);

        /// ... 
    }
```
Based on the implementation, we get: 
* `bounty.getTokenBalance(depToken) = 500 USDC`
* `bounty.getLockedFunds(depToken) = 1000 USDC` since there is just a deposit from Bob is locked. 
* `availableFunds = bounty.getTokenBalance(depToken)  - bounty.getLockedFunds(depToken) = 500 - 1000 < 0`

Revert because of underflow error.

## Impact
User can't claim the remaining rewards 

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/DepositManager/Implementations/DepositManagerV1.sol#L171-L172

## Tool used
Manual review 

## Recommendation
Let the issuer mark the payout has completely. After that the expiry time of the deposits should be redundant. 

***************************
***************************
# [M-02] `setPayoutScheduleFixed()` can be executed with fewer tiers 
###### tags: `sherlock`, `2023-02-openq`, `medium`

## Summary
Function `TierFixedBountyV1.setPayoutScheduleFixed()` is implemented as follows: 
```solidity=
function setPayoutScheduleFixed(
    uint256[] calldata _payoutSchedule,
    address _payoutTokenAddress
) external onlyOpenQ {
    /// ... 
    payoutSchedule = _payoutSchedule;
    payoutTokenAddress = _payoutTokenAddress;

    string[] memory newTierWinners = new string[](payoutSchedule.length);
    bool[] memory newInvoiceComplete = new bool[](payoutSchedule.length);
    bool[] memory newSupportingDocumentsCompleted = new bool[](
        payoutSchedule.length
    );

    /// [$audit-med] resizing to fewer tiers --> out of bound ? 
    for (uint256 i = 0; i < tierWinners.length; i++) {
        newTierWinners[i] = tierWinners[i];
    }
    tierWinners = newTierWinners;
    
    /// ... 
}
```

I assume that the original `payoutSchedule.length = tierWinners.length = 2` and `_payoutSchedule.length = 1`, it will make: 
* `newTierWinners.length = 1`

Let take a look at the first loop in function, we can see that the value of `i` can be `2` since the `tierWinners.length = 2`. While the length of `newTierWinners` is just 1. It will incur the out-of-bound error when calling this function with the fewer tier. 

Note that this issue appear in both `TierFixedBountyV1.setPayoutScheduleFixed()` and `TierdPercentageBountyV1.setPayoutSchedule()`

## Vulnerability Detail
You can place this test into `TierdFixedBounty.test.js` -> `describe('setPayoutSchedule')`

```typescript=
it.only('sherlock should revert with fewer tier', async() => {
    // ASSUME
    let initialPayoutSchedule = await tieredFixedContract.getPayoutSchedule();
    let payoutToString = initialPayoutSchedule.map(thing => thing.toString());
    expect(payoutToString[0]).to.equal('80');
    expect(payoutToString[1]).to.equal('20');

    // REVERT 
    await expect(tieredFixedContract.setPayoutScheduleFixed([100], mockLink.address)).to.be.reverted;
});
```

## Impact
Can't resize the payoutSchedule to fewer tiers

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L165-L167
https://github.com/sherlock-audit/2023-02-openq-WelToHackerLand/blob/eacec290848a595e56d3ce9806a7b2f427a237e7/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L157-L159

## Tool used
Manual review 

## Recommendation
Break the loop when `i > newTierWinners.length`. Similar to `newInvoiceComplete` and `supportingDocumentationsComplete` 