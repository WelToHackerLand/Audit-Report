# Sherlock / frankendao-nov-9
***************************
***************************

# [H-01] Users's nft can be locked because of the wrong implementation of function `Staking.delegate()`

## Lines of code 
* https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L307-L313
* https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L375-L383
* https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L453-L456

## Summary
The implementation of function `Staking._delegate()` forget to: 
* increase the total community power if `tokenVotingPower[_delegatee] == amount`
* decrease the total community power if `tokenVotingPower[currentDelegate] == 0`

## Vulnerability Detail
Following the logic of function `Staking.stake()`, the total community power will be increased when the delegate of `msg.sender` had no `tokenVoting` before. 
```solidity=
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L375-L383
function _stake(uint[] calldata _tokenIds, uint _unlockTime) internal {
    ...some_slice_of_codes...;

    // If the user had no tokenVotingPower before and doesn't delegate, they just unlocked their community voting power
    // If their tokenVotingPower == newVotingPower, that means (a) it was 0 before and (b) they don't delegate, 
    //                                                                                    or it'd be 0 now
    if (tokenVotingPower[msg.sender] == newVotingPower) {
      // The user's community voting power is reactivated, so we add it to the total community voting power
      _updateTotalCommunityVotingPower(msg.sender, true);
    
    // If their delegate had no tokenVotingPower before, then they just unlocked their community voting power
    } else if (tokenVotingPower[getDelegate(msg.sender)] == newVotingPower) {
      // The delegate's community voting power is reactivated, so we add it to the total community voting power
      _updateTotalCommunityVotingPower(getDelegate(msg.sender), true);
    }
}
```
Function `Staking.delegate()` should have the same logic as above when user A delegates to user B who had no voting power before, it should unlock the community power of B either. 

In the same sense, function `Staking.delegate()` should lock the voting power when tokenVotingPower of V reaches 0. 

This missing update can lead to a bad scenario which I will describe later by example in **Code Snippet section** 

## Impact
* Wrong calculation of voting power 
* User's nft will be locked 

## Code Snippet
https://gist.github.com/huuducst/1a5215d765ed1107d62df283b6c4df2b
You can place this file into `/test` folder and run it using: 
```bash= 
forge test -v --match-test testDelegateIssue -v
```
Script explanation:
1. Alice, Bob, and Charlie stake their Franken NFTs.
2. Alice creates a proposal and votes for it. Bob voted for it too. Then Alice and Bob have the same community vote score (equal 1) and total community vote score = 2.
3. Alice's proposal was vetoed then Bob unstake his Franken token (Bob can do it because the proposal has not been active anymore). Then total community vote was updated to 1 (subtracted from Bob's).
4. Charlie delegate voting power to Bob, it will not add Bob's vote score to total community vote score. Then total community vote score still is 1 (because Charlie's vote score = 0).
5. Charlie unstake his Franken token, total community vote score will be updated to 0 (subtracted from Bob's vote score) because it made Bob's voting power decrease to 0.
6. Alice can not unstake her Franken token because it will revert with underflow bug (when subtracting the total community vote score from Alice's vote score).

## Tool used
Foundry 

## Recommendation
Should call `_updateTotalCommunityVotingPower` when delegate and undelegate if voting power of`delegatee` update to 0 or update from 0.

***************************
***************************

# [H-02] Malicious users can stake nft with high `unlockTime` to get a high voting power and prevent users from create a proposal 

## Lines of code 
* https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L358
* https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L390-L394

## Summary
There is no upper bound for the staking duration of an nft. It will let a user stake nft with a pretty high unlock time to get a high voting power. 

## Vulnerability Detail
Function `stake()` in contract `Staking.sol` let users stake an array of `tokenIds` with arbitrary `unlockTime`
```solidity=
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L341-L351
  function stake(uint[] calldata _tokenIds, uint _unlockTime) public {
    // Refunds gas if stakingRefund is true and hasn't been used by this user in the past 24 hours
    if (stakingRefund && lastStakingRefund[msg.sender] + refundCooldown <= block.timestamp) {
      uint256 startGas = gasleft();
      _stake(_tokenIds, _unlockTime);
      lastStakingRefund[msg.sender] = block.timestamp;
      _refundGas(startGas);
    } else {
      _stake(_tokenIds, _unlockTime);
    }
  }
```
Since the `unlockTime` also affects the calculation of voting power through the value of `stakedTimeBonus`
```solidity=
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L389-L394
function _stakeToken(uint _tokenId, uint _unlockTime) internal returns (uint) {
    if (_unlockTime > 0) {
      unlockTime[_tokenId] = _unlockTime;
      uint fullStakedTimeBonus = ((_unlockTime - block.timestamp) * stakingSettings.maxStakeBonusAmount) / stakingSettings.maxStakeBonusTime;
      stakedTimeBonus[_tokenId] = _tokenId < 10000 ? fullStakedTimeBonus : fullStakedTimeBonus / 2;
    }
    ... 
}

// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Staking.sol#L507-L515
function getTokenVotingPower(uint _tokenId) public override view returns (uint) {
  if (ownerOf(_tokenId) == address(0)) revert NonExistentToken();


  // If tokenId < 10000, it's a FrankenPunk, so 100/100 = a multiplier of 1
  uint multiplier = _tokenId < 10_000 ? PERCENT : monsterMultiplier;

  // evilBonus will return 0 for all FrankenMonsters, as they are not eligible for the evil bonus
  return ((baseVotes * multiplier) / PERCENT) + stakedTimeBonus[_tokenId] + evilBonus(_tokenId);
}
```
So the higher `unlockTime` a user set, the more power in voting they gain. For some users who didn't look for the benefit of nft, they can set the value of `unlockTime = HIGH_VALUE`, to get a lot of power to vote in the governance. 

Moreover, in contract `Governance.sol`, when someone call `propose`, function `_propose` will check that sender has enough voting power to propose or not (not smaller than `proposalThreshold`)
```solidity=
/// *** Governance.sol ***  
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L364-L373
function _propose(
        address[] memory _targets,
        uint256[] memory _values,
        string[] memory _signatures,
        bytes[] memory _calldatas,
        string memory _description
    ) internal returns (uint256) {
        // Confirm the proposer meets the proposalThreshold
        uint votesNeededToPropose = proposalThreshold();
        if (staking.getVotes(msg.sender) < votesNeededToPropose) revert NotEligible();
```
And `proposalThreshold` is calculated by total voting power which is too large because of above staking:
```solidity= 
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L316-L318
function proposalThreshold() public view returns (uint256) {
    return bps2Uint(proposalThresholdBPS, staking.getTotalVotingPower());
}
```
So a malicious user just needs to specify a big value of `unlockTime` to break the functionality of the Governance contract when it tries to create a proposal. 
(In case a malicious user try to create a proposal, (s)he can't, cause the value of `quorumVotes` is capped in `uint24`)

## Impact
* Users can have a large amount of voting power 
* Prevent users from creating a proposal

## Code Snippet
https://gist.github.com/huuducst/6854c00fd091f435c7c73522dc73f1dd

You can place this file into `/test` folder and run it using
```bash=
forge test -v --match-test testUnlockTimeIssue -v
```

## Tool used
Foundry 

## Recommendation
Provide a new variable to limit `unlockTime` when a user wants to stake their nfts. 

***************************
***************************
# [M-01] Users who didn't own any nfts still can increase their community power 
###### tags: `sherlock`, `2022-11-frankendao`, `high`

## Lines of code 
https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L607-L646

## Summary
There is no check if a user has 0 votes in the function `Governance.castVote()`. 

## Vulnerability Detail
Users can use the function `castVote()` to vote for a specified proposal. For each time user make a vote, the value of `totalCommunityScoreData.votes` and `userCommunityScoreData[_voter]` will increase by 1.
```solidity=
function _castVote(address _voter, uint256 _proposalId, uint8 _support) internal returns (uint) {
    ...some_slice_of_code...;
   
    uint24 votes = (staking.getVotes(_voter)).toUint24();
    
    ...some_slice_of_code...;
    
     /// [$audit-high] what if votes = 0 --> votes = 0, but still increase community power 
    ++totalCommunityScoreData.votes;
    ++userCommunityScoreData[_voter].votes;

    return votes;
}
```
But for some users who didn't own any nfts, this value still increases. This will make the community power of that users increase even if they don't join the market.

## Impact
* User who didn't own any nfts can increase their community power for future use (if they buy a nft in the future)
* Moreover, this issue can make the value of `proposalThreshold` higher. This will make users harder to propose a new proposal. 
    * Similar issue with `quorumVotes`.

## Code Snippet
```solidity= 
function testVoteWithZeroVotingPower() public { 
    address alice = vm.addr(0xA11CE);
    assertEq(staking.getVotes(alice), 0); //alice's voting power = 0

    uint proposalId = _createAndVerifyProposal();
    vm.warp(block.timestamp + gov.votingDelay());
    vm.stopPrank();

    vm.startPrank(alice);
    gov.castVote(proposalId, 1);

    //alice's vote scores increased
    (uint256 votes, , ) = gov.userCommunityScoreData(alice);
    assertEq(votes, 1);
}
```
Put this test function into `test/governance/CastVote.t.sol`
and run it using
```bash= 
forge test -v --match-test testVoteWithZeroVotingPower -v
```
## Tool used
Foundry 

## Recommendation
Revert the transaction when the voting power of `msg.sender` is 0 

***************************
***************************
# [M-02] Wrong update of community power when a proposal is queued
###### tags: `sherlock`, `2022-11-frankendao`, `high`

## Summary
When a proposal is queued, the value of `userCommunityScoreData[proposal.propser].proposalsCreated` is increased by 1, but it should update the value of `userCommunityScoreData[proposal.propser].proposalsPassed` instead. 

## Vulnerability Detail
The community power of a user is divided into 3 parameters as follows: 
```solidity=
struct CommunityScoreData {
    uint64 votes; // [#explain] number of time a user vote a proposal 
    uint64 proposalsCreated; // [#explain] number of time a user creates a proposal successfully 
    uint64 proposalsPassed; // [#explain] number of time a proposal of user is queued successfully 
}
```
Following the purpose of each parameter I explain in the code above, when a proposal is queued, it should increase the value of `proposalPassed` by 1. But in the function `Governance.queue()` it updates the value of `proposalCreated` which is a wrong implementation. 

## Impact
Wrong calculation of voting power for users (This can happen when `communityPowerMultipliers.proposalsCreated != communityPowerMultipliers.proposalsPassed`)

## Code Snippet
https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L483-L489

## Tool used
Manual review 

## Recommendation
Change function `Governance.queue()` as follow: 
```solidity=
function queue(uint256 _proposalId) external {
    // Succeeded means we're past the endTime, yes votes outweigh no votes, and quorum threshold is met
    if(state(_proposalId) != ProposalState.Succeeded) revert InvalidStatus();

    Proposal storage proposal = proposals[_proposalId];

    // Set the ETA (time for execution) to the soonest time based on the Executor's delay
    uint256 eta = block.timestamp + executor.DELAY();
    proposal.eta = eta.toUint32();

    // Queue separate transactions for each action in the proposal
    uint numTargets = proposal.targets.length;
    for (uint256 i = 0; i < numTargets; i++) {
        executor.queueTransaction(proposal.targets[i], proposal.values[i], proposal.signatures[i], proposal.calldatas[i], eta);
    }
 
    // If a proposal is queued, we are ready to award the community voting power bonuses to the proposer
    
    /// [!note] change here 
    ++userCommunityScoreData[proposal.proposer].proposalsPassed;

    // We don't need to check whether the proposer is accruing community voting power because
    // they needed that voting power to propose, and once they have an Active Proposal, their
    // tokens are locked from delegating and unstaking
    
    /// [!note] change here 
    ++totalCommunityScoreData.proposalsPassed;

    // Remove the proposal from the Active Proposals array
    _removeFromActiveProposals(_proposalId);

    emit ProposalQueued(_proposalId, eta);
}
```

***************************
***************************
# [M-03] Should decrease the community power of proposer when his/her proposal is vetoed/canceled 
###### tags: `sherlock`, `2022-11-frankendao`, `med`

## Summary
When a proposal is vetoed, the proposer of that proposal can still increase his/her community power. 

## Vulnerability Detail
When a proposal is created/queued, the corresponding community power of the proposer (proposalsCreated / proposalsPassed) is increased by 1. 
In some rare cases when a proposal has malicious behavior, the proposal will be vetoed by the admins. 
```solidity=
// url = https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L520-L534

/// @notice Vetoes a proposal 
/// @param _proposalId The id of the proposal to veto
/// @dev This allows the founder or council multisig to veto a malicious proposal
function veto(uint256 _proposalId) external cancelable(_proposalId) onlyAdmins {
    Proposal storage proposal = proposals[_proposalId];


    // If the proposal is queued or executed, remove it from the Executor's queuedTransactions mapping
    // Otherwise, remove it from the Active Proposals array
    _removeTransactionWithQueuedOrExpiredCheck(proposal);


    // Update the vetoed flag so the proposal's state is Vetoed
    proposal.vetoed = true;


    emit ProposalVetoed(_proposalId);
}
```
But even when a proposal is vetoed, the community power of that malicious proposer is still increased. It will support him/her in the future malicious proposal. 

The same issue happens in the function `Governance.cancel()`. 

## Impact
Malicious proposer has more power even when (s)he creates a malicious proposal 

## Code Snippet
https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L523-L534
https://github.com/sherlock-audit/2022-11-frankendao-Trumpero/blob/5055b04e2ba191e2acb15a6f750bf3e38c2e0657/src/Governance.sol#L538-L553

## Tool used
Manual review 

## Recommendation
Decrease the value of corresponding community power when a proposal is canceled/vetoed 
