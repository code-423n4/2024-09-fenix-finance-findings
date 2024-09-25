| Issue ID | Issue Name                                                                 |
|----------|----------------------------------------------------------------------------|
| [L-01](#l-01-users-can-not-create-the-permanent-lock-directly) | Users can't create the permanent lock directly |
| [L-02](#l-02-there-is-an-inconsistency-in-the-vote-and-poke-functions-regarding-the-handling-of-the-end-of-the-voting-window) | There is an inconsistency in the `vote` and `poke` functions regarding the handling of the end of the voting window |
| [L-03](#l-03-the-VoterUpgradeableV2distributeFees-function-always-reverts) | The `VoterUpgradeableV2.distributeFees()` function always reverts |
| [L-04](#l-04-unnecessary-statements-in-the-_checkpoint-function) | Unnecessary statements in the `_checkpoint` function |
| [L-05](#l-05-the-lastdistributiontimestamp-variable-should-be-always-updated-in-the-_distribute-function) | The `lastDistributionTimestamp` variable should be always updated in the `_distribute` function |
| [L-06](#l-06-the-poke-function-does-not-check-voting-delay) | The `poke` function does not check voting delay |
| [L-07](#l-07-voting-power-of-a-nft-is-not-used-completely) | Voting power of a NFT is not used completely |
| [L-08](#l-08-the-voting-power-is-not-calculated-correctly-according-to-the-weights-assigned-by-the-user-during-poking) | The voting power is not calculated correctly according to the weights assigned by the user during poking |

## [L-01] Users can not create the permanent lock directly

### Vulnerability Detail

In the `_createLock` function, it calls the `_processLockChange` function with `nftStates[newTokenId].locked`, which is an empty variable at L420. This means it only initializes with the `isPermanentLocked` parameter set to `false`.

When users try to lock their tokens permanently, they should first create the lock and then call `lockPermanent`. Additionally, to receive boosted FNX, they must set the end of the lock to be greater than `veBoostCached.getMinLockedTimeForBoost()` when creating the lock.

```solidity
File: contracts\core\VotingEscrowUpgradeableV2.sol
415:         _mint(to_, newTokenId);
416:         _proccessLockChange(
417:             newTokenId,
418:             amount_,
419:             unlockTimestamp,
420:             nftStates[newTokenId].locked,
421:             DepositType.CREATE_LOCK_TYPE,
422:             shouldBoosted_
423:         );
```

### Recommended Mitigation Steps

Add a mechanism that allows users to set permanent locking when they create the lock.

## [L-02] There is an inconsistency in the `vote` and `poke` functions regarding the handling of the end of the voting window

### Vulnerability Detail

The `VoterUpgradeableV2.vote` function allows voting at the end of the voting window, while the `VoterUpgradeableV2.poke` function does not accommodate this.

The `VoterUpgradeableV2.poke` function calls the `_checkVoteWindow` function to verify that the current time is within the allowed voting window.

```solidity
    function poke(uint256 tokenId_) external nonReentrant onlyNftApprovedOrOwner(tokenId_) {
@>      _checkVoteWindow();
        _poke(tokenId_);
    }
```

However, for whitelisted token in the `managedNFTManagerCache`, the `VoterUpgradeableV2.vote` function permits the current time to be at the end of the voting window.

```solidity
    function vote(
        uint256 tokenId_,
        address[] calldata poolsVotes_,
        uint256[] calldata weights_
    ) external nonReentrant onlyNftApprovedOrOwner(tokenId_) {
        [...]
        _checkStartVoteWindow();
        [...]
        if (!managedNFTManagerCache.isWhitelistedNFT(tokenId_)) {
@>          _checkEndVoteWindow();
        }
        [...]
    }
```

### Recommended Mitigation Steps

It is recommended to change the code as following:

```diff
    function poke(uint256 tokenId_) external nonReentrant onlyNftApprovedOrOwner(tokenId_) {
-       _checkVoteWindow();
+       _checkStartVoteWindow();
+       if (!managedNFTManagerCache.isWhitelistedNFT(tokenId_)) {
+           _checkEndVoteWindow();
+       }
        _poke(tokenId_);
    }
```

## [L-03] The `VoterUpgradeableV2.distributeFees()` function always reverts

### Vulnerability Detail

The `VoterUpgradeableV2.distributeFees()` function distributes fees to a list of gauges. It calls the `gauges.claimFees()` function at L400.

```solidity
File: core\VoterUpgradeableV2.sol
        function distributeFees(address[] calldata gauges_) external { 
            for (uint256 i; i < gauges_.length; i++) {
                GaugeState memory state = gaugesState[gauges_[i]];
                if (state.isGauge && state.isAlive) {
400: @>             IGauge(gauges_[i]).claimFees(); 
                }
            }
        }
```

The `guages.claimFees()` function calls the `feeVault.claimFees()` function at L394.

```solidity
File: contracts\gauges\GaugeUpgradeable.sol
         function claimFees() external nonReentrant returns (uint256 claimed0, uint256 claimed1) { 
389: @>      return _claimFees();
         }
     
         function _claimFees() internal returns (uint256 claimed0, uint256 claimed1) {
             address _token = address(TOKEN);
394: @>      (claimed0, claimed1) = IFeesVault(feeVault).claimFees();
```

In the `feeVault.claimFees()` function, it checks if gauge is registered in the Voter at L72.

```solidity
File: contracts\fees\FeesVaultUpgradeable.sol
        function claimFees() external virtual override returns (uint256, uint256) {
            IFeesVaultFactory factoryCache = IFeesVaultFactory(factory);
            (uint256 toGaugeRate, address[] memory recipients, uint256[] memory rates_) = factoryCache.getDistributionConfig(address(this));
    
            address poolCache = pool;
            if (toGaugeRate > 0) {
                address voterCache = IFeesVaultFactory(factory).voter();
72: @>          if (!IVoter(voterCache).isGauge(msg.sender)) {
73:                 revert AccessDenied();
74:             }
75: @>          if (poolCache != IVoter(voterCache).poolForGauge(msg.sender)) {
76:                 revert PoolMismatch();
77:             }
```

However, as the `VoterUpgradeableV2` contract does not have the `isGauge` function, this function call is reverted.

### Recommended Mitigation Steps

Add the `isGauge` function and `poolForGauge` function to the `VoterUpgradeableV2` contract.

## [L-04] Unnecessary statements in the `_checkpoint` function

### Vulnerability Detail

In the `VotingEscrowUpgradeableV2._checkpoint()` function, there are several unnecessary statements.

```solidity
File: core\VotingEscrowUpgradeableV2.sol
583:             if (last_point.bias < 0) {
584:                 // This can happen
585:                 last_point.bias;
586:             }
587:             if (last_point.slope < 0) {
588:                 // This cannot happen - just in case
589:                 last_point.slope;
590:             }

609:             if (last_point.slope < 0) {
610:                 last_point.slope;
611:             }
612:             if (last_point.bias < 0) {
613:                 last_point.bias;
614:             }
```

### Recommended Mitigation Steps

Remove these lines to reduce complexity.

## [L-05] The `lastDistributionTimestamp` variable should be always updated in the `_distribute` function

### Vulnerability Detail

In the `VoterUpgradeableV2._distribute()` function, the `lastDistributionTimestamp` variable is only updated when `claimable` is greater than 0 and gauge is alive.

```solidity
File: contracts\core\VoterUpgradeableV2.sol
644:             if (claimable > 0 && state.isAlive) {
645:                 gaugesState[gauge_].claimable = 0;
646:                 gaugesState[gauge_].lastDistributionTimestamp = currentTimestamp;
647:                 IGauge(gauge_).notifyRewardAmount(token, claimable);
648:                 emit DistributeReward(_msgSender(), gauge_, claimable);
649:             }
```

### Recommended Mitigation Steps

It is recommended to change the code as following:

```diff
        if (claimable > 0 && state.isAlive) {
            gaugesState[gauge_].claimable = 0;
-           gaugesState[gauge_].lastDistributionTimestamp = currentTimestamp;
            IGauge(gauge_).notifyRewardAmount(token, claimable);
            emit DistributeReward(_msgSender(), gauge_, claimable);
        }
+       gaugesState[gauge_].lastDistributionTimestamp = currentTimestamp;
```

## [L-06] The `poke` function does not check voting delay

### Vulnerability Detail

In the `VoterUpgradeableV2.poke()` function, it does not check voting delay.

```solidity
File: contracts\core\VoterUpgradeableV2.sol
460:     function poke(uint256 tokenId_) external nonReentrant onlyNftApprovedOrOwner(tokenId_) {
461:         _checkVoteWindow();
462:         _poke(tokenId_);
463:     }
```

### Recommended Mitigation Steps

It is recommended to change the code as following:

```diff
    function poke(uint256 tokenId_) external nonReentrant onlyNftApprovedOrOwner(tokenId_) {
+       _checkVoteDelay(tokenId_);
        _checkVoteWindow();
        _poke(tokenId_);
    }
```

## [L-07] Voting power of a NFT is not used completely

### Vulnerability Detail

In the `VoterUpgradeableV2._vote()` function, it calculates the `votePowerForPool` from weights. However, the actual voting power of a NFT(`nftVotePower`) is bigger than final voting power(`totalVoterPower`) due to precision loss.

```solidity
File: core\VoterUpgradeableV2.sol
725:         uint256 nftVotePower = IVotingEscrowV2(votingEscrow).balanceOfNFT(tokenId_); 

737:         for (uint256 i; i < pools_.length; i++) {
738:             address pool = pools_[i];
739:             address gauge = poolToGauge[pools_[i]];
740:             uint256 votePowerForPool = (weights_[i] * nftVotePower) / totalVotesWeight;

751:             totalVoterPower += votePowerForPool;      
755:         }
```

### Recommended Mitigation Steps

It is recommend to change code as following to reduce precision loss.
```diff
         for (uint256 i; i < pools_.length; i++) {
             address pool = pools_[i];
             address gauge = poolToGauge[pools_[i]];            
-            uint256 votePowerForPool = (weights_[i] * nftVotePower) / totalVotesWeight;
+            if (i == pools_.length - 1) {
+               uint256 votePowerForPool = nftVotePower - totalVoterPower;
+            }
+            else {
+               uint256 votePowerForPool = (weights_[i] * nftVotePower) / totalVotesWeight;
+            }
         }
```

## [L-08] The voting power is not calculated correctly according to the weights assigned by the user during poking

### Vulnerability Detail

In the `VoterUpgradeableV2._vote` function, there is precision loss in calculation of `votePowerForPool`.

```solidity
File: contracts\core\VoterUpgradeableV2.sol
740:             uint256 votePowerForPool = (weights_[i] * nftVotePower) / totalVotesWeight;
                 [...]
749:             votes[tokenId_][pool] = votePowerForPool;
```

And `votePowerForPool` is used recursively in `_poke` function.

```solidity
File: contracts\core\VoterUpgradeableV2.sol
615:         for (uint256 i; i < _poolVote.length; ) {
616:             _weights[i] = votes[tokenId_][_poolVote[i]];
617:             unchecked {
618:                 i++;
619:             }
620:         }
621:         _vote(tokenId_, _poolVote, _weights);
```

At that time, due to precision loss, `votePowerForPool` is not calculated correctly according to the weights assigned by the user.

### Recommended Mitigation Steps

Store the array of weights corresponding to the pools during voting and use it instead of `votes[tokenId_][_poolVote[i]]` in the poking process.
