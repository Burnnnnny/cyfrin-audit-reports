**수석 감사 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

**보조 감사 (Assisting Auditors)**



---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### `Goldilend.lock()`은 항상 되돌려짐 (revert)

**심각도:** 높음

**설명:** `lock()` 함수에서 사용자로부터 `iBGT`를 가져오기 전에 `_refreshiBGT()`를 호출하며, `iBGTVault(ibgtVault).stake()`를 호출하는 동안 되돌려집니다(revert).

```solidity
  function lock(uint256 amount) external {
    uint256 mintAmount = _GiBGTMintAmount(amount);
    poolSize += amount;
    _refreshiBGT(amount); //@audit 자금을 예치한 후 호출해야 함
    SafeTransferLib.safeTransferFrom(ibgt, msg.sender, address(this), amount);
    _mint(msg.sender, mintAmount);
    emit iBGTLock(msg.sender, amount);
  }
...
  function _refreshiBGT(uint256 ibgtAmount) internal {
    ERC20(ibgt).approve(ibgtVault, ibgtAmount);
    iBGTVault(ibgtVault).stake(ibgtAmount); //@audit 여기서 되돌려짐
  }
```

**영향:** `lock()`이 항상 되돌려지므로 사용자는 `iBGT`를 잠글 수 없습니다.

**권장 완화 방법:** `_refreshiBGT()`는 사용자로부터 자금을 가져온 후에 호출되어야 합니다.

**Client:** [PR #1](https://github.com/0xgeeb/goldilocks-core/pull/1)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldilend.repay()`에서 잘못된 `PoolSize` 증가

**심각도:** 높음

**설명:** 사용자가 `repay()`를 사용하여 대출을 상환할 때, 상환된 이자로 `poolSize`를 증가시킵니다. 증가시키는 동안 잘못된 금액을 사용합니다.

```solidity
  function repay(uint256 repayAmount, uint256 _userLoanId) external {
    Loan memory userLoan = loans[msg.sender][_userLoanId];
    if(userLoan.borrowedAmount < repayAmount) revert ExcessiveRepay();
    if(block.timestamp > userLoan.endDate) revert LoanExpired();
    uint256 interestLoanRatio = FixedPointMathLib.divWad(userLoan.interest, userLoan.borrowedAmount);
    uint256 interest = FixedPointMathLib.mulWadUp(repayAmount, interestLoanRatio);
    outstandingDebt -= repayAmount - interest > outstandingDebt ? outstandingDebt : repayAmount - interest;
    loans[msg.sender][_userLoanId].borrowedAmount -= repayAmount;
    loans[msg.sender][_userLoanId].interest -= interest;
    poolSize += userLoan.interest * (1000 - (multisigShare + apdaoShare)) / 1000; //@audit userLoan.interest 대신 interest를 사용해야 함
...
  }
```

사용자가 `interest`만 상환했으므로 `userLoan.interest` 대신 `interest`를 사용해야 합니다.

**영향:** `repay()` 호출 후 `poolSize`가 잘못 추적되어 여러 함수가 예상대로 작동하지 않을 수 있습니다.

**권장 완화 방법:** `poolSize`는 `interest`를 사용하여 업데이트되어야 합니다.

**Client:** [PR #2](https://github.com/0xgeeb/goldilocks-core/pull/2)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 사용자는 무효화된 NFT를 사용하여 만료된 부스트를 연장할 수 있음

**심각도:** 높음

**설명:** `Goldilend.sol#L251`에서 사용자는 무효화된 NFT로 부스트를 연장할 수 있습니다.
- 사용자가 유효한 NFT로 부스트를 생성했습니다.
- 그 후 `adjustBoosts()`를 사용하여 NFT가 무효화되었습니다.
- 원래 부스트가 만료된 후 사용자가 빈 배열로 `boost()`를 호출하면 원래 크기로 부스트가 다시 연장됩니다.

```solidity
  function _buildBoost(
    address[] calldata partnerNFTs,
    uint256[] calldata partnerNFTIds
  ) internal returns (Boost memory newUserBoost) {
    uint256 magnitude;
    Boost storage userBoost = boosts[msg.sender];
    if(userBoost.expiry == 0) {
...
    }
    else {
      address[] storage nfts = userBoost.partnerNFTs;
      uint256[] storage ids = userBoost.partnerNFTIds;
      magnitude = userBoost.boostMagnitude; //@audit 확인 없이 이전 크기 사용
      for (uint256 i = 0; i < partnerNFTs.length; i++) {
        magnitude += partnerNFTBoosts[partnerNFTs[i]];
        nfts.push(partnerNFTs[i]);
        ids.push(partnerNFTIds[i]);
      }
      newUserBoost = Boost({
        partnerNFTs: nfts,
        partnerNFTIds: ids,
        expiry: block.timestamp + boostLockDuration,
        boostMagnitude: magnitude
      });
    }
  }
```

**영향:** 악의적인 사용자는 무효화된 NFT를 사용하여 부스트를 영구적으로 연장할 수 있습니다.

**권장 완화 방법:** 사용자가 부스트를 연장할 때마다 NFT를 다시 평가해야 합니다.

**Client:** [PR #3](https://github.com/0xgeeb/goldilocks-core/pull/3)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 팀원은 초기 할당을 영원히 언스테이킹할 수 없음

**심각도:** 높음

**설명:** 사용자가 `unstake()`를 호출할 때 `_vestingCheck()`를 사용하여 베스팅된 금액을 계산합니다.

```solidity
  function _vestingCheck(address user, uint256 amount) internal view returns (uint256) {
    if(teamAllocations[user] > 0) return 0; //@audit 팀원에 대해 0 반환
    uint256 initialAllocation = seedAllocations[user];
    if(initialAllocation > 0) {
      if(block.timestamp < vestingStart) return 0;
      uint256 vestPortion = FixedPointMathLib.divWad(block.timestamp - vestingStart, vestingEnd - vestingStart);
      return FixedPointMathLib.mulWad(vestPortion, initialAllocation) - (initialAllocation - stakedLocks[user]);
    }
    else {
      return amount;
    }
  }
```

하지만 팀원에 대해 0을 반환하므로 그들은 영원히 언스테이킹할 수 없습니다.
또한 `stake()`에서는 팀원이 아닌 시드 투자자만 방지합니다. 따라서 팀원이 추가로 스테이킹한 경우에도 언스테이킹할 수 없습니다.

**영향:** 팀원은 영원히 언스테이킹할 수 없습니다.

**권장 완화 방법:** `_vestingCheck`는 팀원에 대해 초기 투자자와 동일한 로직을 사용해야 합니다.

**Client:** 팀이 토큰을 언스테이킹할 수 없는 것이 의도된 것임을 확인했습니다. [PR #4](https://github.com/0xgeeb/goldilocks-core/pull/4)는 `stake`가 팀원의 스테이킹을 막지 않는 문제를 수정합니다.

**Cyfrin:** 확인되었습니다.

### `GovLocks`에서 `deposits` 매핑을 사용해서는 안 됨

**심각도:** 높음

**설명:** `GovLocks`에서는 `deposits` 매핑을 사용하여 모든 사용자의 예금 금액을 추적합니다.
사용자는 `govLocks`를 자유롭게 전송할 수 있으므로 `deposits`이 `govLocks` 잔액보다 적을 수 있으며 원할 때 출금할 수 없을 수 있습니다.

```solidity
  function deposit(uint256 amount) external {
    deposits[msg.sender] += amount; //@audit 필요 없음
    _moveDelegates(address(0), delegates[msg.sender], amount);
    SafeTransferLib.safeTransferFrom(locks, msg.sender, address(this), amount);
    _mint(msg.sender, amount);
  }

  /// @notice Withdraws Locks to burn Govlocks
  /// @param amount Amount of Locks to withdraw
  function withdraw(uint256 amount) external {
    deposits[msg.sender] -= amount; //@audit 필요 없음
    _moveDelegates(delegates[msg.sender], address(0), amount);
    _burn(msg.sender, amount);
    SafeTransferLib.safeTransfer(locks, msg.sender, amount);
  }
```

다음은 가능한 시나리오입니다.
- Alice가 100 `LOCKS`를 입금하고 100 `govLOCKS`를 받았습니다. 또한 `deposits[Alice] = 100`입니다.
- Bob은 투표권을 얻기 위해 Alice로부터 50 `govLOCKS`를 샀습니다.
- Bob이 `withdraw()`를 호출하려고 하면 50 `govLOCKS`가 있음에도 불구하고 `deposits[Bob] = 0`이기 때문에 되돌려집니다(revert).

**영향:** 사용자는 `govLOCKS`로 `LOCKS`를 출금할 수 없습니다.

**권장 완화 방법:** `deposits` 매핑을 전혀 사용할 필요가 없으며 `govLocks` 잔액에만 의존할 수 있습니다.

**Client:** [PR #8](https://github.com/0xgeeb/goldilocks-core/pull/8)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldilend`의 일부 기능은 영원히 되돌려짐

**심각도:** 높음

**설명:** `Goldilend.multisigInterestClaim()/apdaoInterestClaim()/sunsetProtocol()`은 전송 전에 `ibgtVault`에서 `ibgt`를 인출하지 않기 때문에 영원히 되돌려집니다.

```solidity
  function multisigInterestClaim() external {
    if(msg.sender != multisig) revert NotMultisig();
    uint256 interestClaim = multisigClaims;
    multisigClaims = 0;
    SafeTransferLib.safeTransfer(ibgt, multisig, interestClaim);
  }

  /// @inheritdoc IGoldilend
  function apdaoInterestClaim() external {
    if(msg.sender != apdao) revert NotAPDAO();
    uint256 interestClaim = apdaoClaims;
    apdaoClaims = 0;
    SafeTransferLib.safeTransfer(ibgt, apdao, interestClaim);
  }

...

  function sunsetProtocol() external {
    if(msg.sender != timelock) revert NotTimelock();
    SafeTransferLib.safeTransfer(ibgt, multisig, poolSize - outstandingDebt);
  }
```

`ibgtVault`가 `Goldilend`의 모든 `ibgt`를 가지고 있으므로 먼저 `ibgtVault`에서 인출해야 합니다.

**영향:** `Goldilend.multisigInterestClaim()/apdaoInterestClaim()/sunsetProtocol()`은 영원히 되돌려집니다.

**권장 완화 방법:** 3개의 함수는 다음과 같이 변경되어야 합니다.

```diffclea
  function multisigInterestClaim() external {
    if(msg.sender != multisig) revert NotMultisig();
    uint256 interestClaim = multisigClaims;
    multisigClaims = 0;
+  iBGTVault(ibgtVault).withdraw(interestClaim);
    SafeTransferLib.safeTransfer(ibgt, multisig, interestClaim);
  }

  /// @inheritdoc IGoldilend
  function apdaoInterestClaim() external {
    if(msg.sender != apdao) revert NotAPDAO();
    uint256 interestClaim = apdaoClaims;
    apdaoClaims = 0;
+  iBGTVault(ibgtVault).withdraw(interestClaim);
    SafeTransferLib.safeTransfer(ibgt, apdao, interestClaim);
  }

...

  function sunsetProtocol() external {
    if(msg.sender != timelock) revert NotTimelock();
+  iBGTVault(ibgtVault).withdraw(poolSize - outstandingDebt);
    SafeTransferLib.safeTransfer(ibgt, multisig, poolSize - outstandingDebt);
  }
```

**Client:** [PR #9](https://github.com/0xgeeb/goldilocks-core/pull/9) 및 [PR #12](https://github.com/0xgeeb/goldilocks-core/pull/12)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

## 중간 위험 (Medium Risk)


### `Goldigovernor._getProposalState()`는 `totalSupply`를 사용해서는 안 됨

**심각도:** 중간

**설명:** `_getProposalState()`에서 비교 중에 `Goldiswap(goldiswap).totalSupply()`를 사용합니다.

```solidity
  function _getProposalState(uint256 proposalId) internal view returns (ProposalState) {
    Proposal storage proposal = proposals[proposalId];
    if (proposal.cancelled) return ProposalState.Canceled;
    else if (block.number <= proposal.startBlock) return ProposalState.Pending;
    else if (block.number <= proposal.endBlock) return ProposalState.Active;
    else if (proposal.eta == 0) return ProposalState.Succeeded;
    else if (proposal.executed) return ProposalState.Executed;
    else if (proposal.forVotes <= proposal.againstVotes || proposal.forVotes < Goldiswap(goldiswap).totalSupply() / 20) { //@audit totalSupply를 사용해서는 안 됨
      return ProposalState.Defeated;
    }
    else if (block.timestamp >= proposal.eta + Timelock(timelock).GRACE_PERIOD()) {
      return ProposalState.Expired;
    }
    else {
      return ProposalState.Queued;
    }
  }
```

`totalSupply`는 실시간으로 증가하고 있으므로, `Queued` 상태의 제안이 공급 증가로 인해 예기치 않게 `Defeated` 상태로 변경될 수 있습니다.

**영향:** 제안 상태가 예기치 않게 변경될 수 있습니다.

**권장 완화 방법:** `totalSupply`를 사용하는 대신 쿼럼(quorum) 확인을 위한 다른 메커니즘을 도입해야 합니다.

**Client:** [PR #5](https://github.com/0xgeeb/goldilocks-core/pull/5)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldivault.redeemYield()`에서 사용자는 재진입(reentrancy)을 사용하여 더 많은 수익 토큰을 상환할 수 있음

**심각도:** 중간

**설명:** `yieldToken`에 `beforeTokenTransfer` 훅이 있는 경우 `Goldivault.redeemYield()`에서 재진입이 가능합니다.

- `yt.totalSupply = 100, yieldToken.balance = 100`이고 사용자에게 20 yt가 있다고 가정합니다.
- 사용자가 10 yt로 `redeemYield()`를 호출합니다.
- 그러면 `yt.totalSupply`는 90으로 변경되고 사용자에게 `100 * 10 / 100 = 10 yieldToken`을 전송합니다.
- `beforeTokenTransfer` 훅 내부에서 사용자는 10 yt로 `redeemYield()`를 다시 호출합니다.
- `yieldToken.balance`는 여전히 100이므로 그는 `100 * 10 / 90 = 11 yieldToken`을 받게 됩니다.

```solidity
  function redeemYield(uint256 amount) external {
    if(amount == 0) revert InvalidRedemption();
    if(block.timestamp < concludeTime + delay || !concluded) revert NotConcluded();
    uint256 yieldShare = FixedPointMathLib.divWad(amount, ERC20(yt).totalSupply());
    YieldToken(yt).burnYT(msg.sender, amount);
    uint256 yieldTokensLength = yieldTokens.length;
    for(uint8 i; i < yieldTokensLength; ++i) {
      uint256 finalYield;
      if(yieldTokens[i] == depositToken) {
        finalYield = ERC20(yieldTokens[i]).balanceOf(address(this)) - depositTokenAmount;
      }
      else {
        finalYield = ERC20(yieldTokens[i]).balanceOf(address(this));
      }
      uint256 claimable = FixedPointMathLib.mulWad(finalYield, yieldShare);
      SafeTransferLib.safeTransfer(yieldTokens[i], msg.sender, claimable);
    }
    emit YieldTokenRedemption(msg.sender, amount);
  }
```

**영향:** 악의적인 사용자는 `redeemYield()`를 사용하여 `yieldToken`을 탈취할 수 있습니다.

**권장 완화 방법:** `redeemYield()`에 `nonReentrant` 수정자를 추가해야 합니다.

**Client:** [PR #13](https://github.com/0xgeeb/goldilocks-core/pull/13)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldigovernor.cancel()`의 잘못된 검증

**심각도:** 중간

**설명:** `Goldigovernor.cancel()`에서 제안자가 자신의 제안을 취소하려면 `proposalThreshold`보다 적은 투표권을 가져야 합니다.

```solidity
  function cancel(uint256 proposalId) external {
    if(_getProposalState(proposalId) == ProposalState.Executed) revert InvalidProposalState();
    Proposal storage proposal = proposals[proposalId];
    if(msg.sender != proposal.proposer) revert NotProposer();
    if(GovLocks(govlocks).getPriorVotes(proposal.proposer, block.number - 1) > proposalThreshold) revert AboveThreshold(); //@audit 부정확함
    proposal.cancelled = true;
    uint256 targetsLength = proposal.targets.length;
    for (uint256 i = 0; i < targetsLength; i++) {
      Timelock(timelock).cancelTransaction(proposal.targets[i], proposal.eta, proposal.values[i], proposal.calldatas[i], proposal.signatures[i]);
    }
    emit ProposalCanceled(proposalId);
  }
```

**영향:** 제안자는 자신의 투표권을 줄이지 않는 한 자신의 제안을 취소할 수 없습니다.

**권장 완화 방법:** 다음과 같이 수정해야 합니다.

```solidity
if(msg.sender != proposal.proposer && GovLocks(govlocks).getPriorVotes(proposal.proposer, block.number - 1) > proposalThreshold) revert Error;
```

**Client:** [PR #7](https://github.com/0xgeeb/goldilocks-core/pull/7)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 증가된 `proposalThreshold`로 인해 사용자가 제안을 취소할 수 없음

**심각도:** 중간

**설명:** 사용자가 `cancel()`을 호출할 때 `proposalThreshold`로 호출자의 투표권을 검증하는데, 이는 `setProposalThreshold()`를 사용하여 변경될 수 있습니다.

```solidity
  function setProposalThreshold(uint256 newProposalThreshold) external {
    if(msg.sender != multisig) revert NotMultisig();
    if(newProposalThreshold < MIN_PROPOSAL_THRESHOLD || newProposalThreshold > MAX_PROPOSAL_THRESHOLD) revert InvalidVotingParameter();
    uint256 oldProposalThreshold = proposalThreshold;
    proposalThreshold = newProposalThreshold;
    emit ProposalThresholdSet(oldProposalThreshold, proposalThreshold);
  }
```

다음은 가능한 시나리오입니다.
- `proposalThreshold = 100`이고 사용자가 100 투표권을 가지고 있다고 가정합니다.
- 사용자가 `propose()`를 사용하여 제안했습니다.
- 그 후 `multisig`에 의해 `proposalThreshold`가 150으로 증가했습니다.
- 사용자가 `cancel()`을 호출하면 투표권이 충분하지 않으므로 되돌려집니다(revert).

**영향:** 증가된 `proposalThreshold`로 인해 사용자가 자신의 제안을 취소할 수 없습니다.

**권장 완화 방법:** `proposalThreshold`를 제안 상태로 캐시하는 것이 좋습니다.

**Client:** 확인됨, 보류 중인 제안이 없을 때만 매개변수를 변경하도록 할 것입니다.

**Cyfrin:** 확인됨.

### `Goldilend.liquidate()`가 언더플로로 인해 되돌려질 수 있음

**심각도:** 중간

**설명:** `repay()`에서 `interest` 계산 중에 반올림이 발생할 수 있습니다.

```solidity
  function repay(uint256 repayAmount, uint256 _userLoanId) external {
      Loan memory userLoan = loans[msg.sender][_userLoanId];
      if(userLoan.borrowedAmount < repayAmount) revert ExcessiveRepay();
      if(block.timestamp > userLoan.endDate) revert LoanExpired();
      uint256 interestLoanRatio = FixedPointMathLib.divWad(userLoan.interest, userLoan.borrowedAmount);
L425  uint256 interest = FixedPointMathLib.mulWadUp(repayAmount, interestLoanRatio); //@audit 반올림 문제
      outstandingDebt -= repayAmount - interest > outstandingDebt ? outstandingDebt : repayAmount - interest;
      ...
  }
...
  function liquidate(address user, uint256 _userLoanId) external {
      Loan memory userLoan = loans[msg.sender][_userLoanId];
      if(block.timestamp < userLoan.endDate || userLoan.liquidated || userLoan.borrowedAmount == 0) revert Unliquidatable();
      loans[user][_userLoanId].liquidated = true;
      loans[user][_userLoanId].borrowedAmount = 0;
L448  outstandingDebt -= userLoan.borrowedAmount - userLoan.interest;
      ...
  }
```

다음은 가능한 시나리오입니다.
- `borrowedAmount = 100, interest = 10`인 2명의 차용인이 있습니다. 그리고 `outstandingDebt = 2 * (100 - 10) = 180`입니다.
- 첫 번째 차용인이 `repayAmount = 100`으로 `repay()`를 호출합니다.
- L425의 반올림 문제로 인해 `interest`는 10 대신 9입니다. 그리고 `outstandingDebt = 180 - (100 - 9) = 89`입니다.
- 두 번째 차용인에 대한 `liquidate()`에서 `outstandingDebt = 89 < borrowedAmount - interest = 90`이므로 L448에서 되돌려집니다.

**영향:** `liquidate()`가 언더플로로 인해 되돌려질 수 있습니다.

**권장 완화 방법:** `liquidate()`에서 `outstandingDebt`는 다음과 같이 업데이트되어야 합니다.

```diff
  /// @inheritdoc IGoldilend
  function liquidate(address user, uint256 _userLoanId) external {
    Loan memory userLoan = loans[msg.sender][_userLoanId];
    if(block.timestamp < userLoan.endDate || userLoan.liquidated || userLoan.borrowedAmount == 0) revert Unliquidatable();
    loans[user][_userLoanId].liquidated = true;
    loans[user][_userLoanId].borrowedAmount = 0;
+  uint256 debtToRepay = userLoan.borrowedAmount - userLoan.interest;
+  outstandingDebt -= debtToRepay > outstandingDebt ? outstandingDebt : debtToRepay;
   ...
  }
```

**Client:** [PR #10](https://github.com/0xgeeb/goldilocks-core/pull/10)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldigovernor`에서 블록 시간의 잘못된 가정

**심각도:** 중간

**설명:** `Goldigovernor.sol`에서 투표 기간/지연 제한은 15초 블록 시간으로 설정됩니다.

```solidity
  /// @notice Minimum voting period
  uint32 public constant MIN_VOTING_PERIOD = 5760; // About 24 hours

  /// @notice Maximum voting period
  uint32 public constant MAX_VOTING_PERIOD = 80640; // About 2 weeks

  /// @notice Minimum voting delay
  uint32 public constant MIN_VOTING_DELAY = 1;

  /// @notice Maximum voting delay
  uint32 public constant MAX_VOTING_DELAY = 40320; // About 1 week
```

하지만 Berachain은 [문서](https://docs.berachain.com/faq/#how-well-does-berachain-perform)에 따르면 5초의 블록 시간을 가집니다.

```
Berachain has the following properties:

- Block time: 5s
```

따라서 이러한 제한은 예상보다 짧게 설정됩니다.

**영향:** 투표 기간/지연 제한이 예상보다 짧게 설정됩니다.

**권장 완화 방법:** 5초 블록 시간으로 이러한 제한을 계산해야 합니다.

**Client:** [PR #14](https://github.com/0xgeeb/goldilocks-core/pull/14)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

## 낮은 위험 (Low Risk)


### `Goldivault.changeProtocolParameters()`는 `endTime`을 업데이트해서는 안 됨

**설명:** `Goldivault.changeProtocolParameters()`는 `startTime`을 변경하지 않고 `endTime`만 업데이트합니다.

```solidity
  function changeProtocolParameters(
    uint256 _earlyWithdrawalFee,
    uint256 _yieldFee,
    uint256 _delay,
    uint256 _duration
  ) external {
    if(msg.sender != timelock) revert NotTimelock();
    earlyWithdrawalFee = _earlyWithdrawalFee;
    yieldFee = _yieldFee;
    delay = _delay;
    duration = _duration;
    endTime = block.timestamp + _duration;
  }
```

이 함수는 단지 매개변수를 변경하기 위한 것이므로 `endTime`을 업데이트하지 않는 것이 더 적절합니다.

**Client:** [PR #11](https://github.com/0xgeeb/goldilocks-core/pull/11)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### `Goldilocked._lockedLocks()`는 올림해야 함

**설명:** 내림으로 인해 사용자가 1 wei를 더 빌릴 수 있습니다.
```solidity
  function _lockedLocks(address user) internal view returns (uint256) {
    return FixedPointMathLib.divWad(borrowedHoney[user], IGoldiswap(goldiswap).floorPrice());//@audit 올림 (round up)
  }
```
**Client:** [PR #15](https://github.com/0xgeeb/goldilocks-core/pull/15)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 상환 실패 가능성

**설명:** `Goldivault`에서 `changeProtocolParameters/initializeProtocol()`을 호출한 후 `endTime/duration`이 변경되기 때문에 `redeemOwnership()`이 예상대로 작동하지 않을 수 있습니다.

**Client:** 확인됨, 볼트가 종료되고 소유권 토큰 보유자가 상환할 기회를 가진 후에만 changeProtocolParameters를 호출할 계획입니다. 매개변수를 업데이트하고 갱신할 것이라고 공지할 것입니다.

**Cyfrin:** 확인됨.

### `eta` 확인 중 일관성 없는 비교

**설명:** `Goldigovernor`와 `Timelock`에서 일관성 없는 비교가 사용됩니다.

```solidity
File: Goldigovernor.sol
386:     else if (block.timestamp >= proposal.eta + Timelock(timelock).GRACE_PERIOD()) {
387:       return ProposalState.Expired;
388:     }

File: Timelock.sol
138:     if(block.timestamp > eta + GRACE_PERIOD) revert TxStale();
```
**Client:** [PR #6](https://github.com/0xgeeb/goldilocks-core/pull/6)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 잘못된 주석

**설명:**
```solidity
File: Timelock.sol
49:   /// @notice Delay for queueing a transaction in blocks //@audit 블록이 아닌 초 단위
60:   /// @param _delay Delay the timelock will use, in blocks //@audit 블록이 아닌 초 단위
103:   /// @param eta Duration of time until transaction can be executed, in blocks //@audit 블록이 아닌 초 단위
123:   /// @param eta Duration of time until transaction can be executed, in blocks //@audit 블록이 아닌 초 단위
154:   /// @param eta Duration of time until transaction can be executed, in blocks //@audit 블록이 아닌 초 단위

File: Goldilend.sol
496:   /// @notice Calculates claimable Porridge per GiBGT //@audit 청구 가능한 보상 토큰
608:   /// @dev Claims existing vault rewards and updates poolSize //@audit 이 함수는 poolSize를 업데이트하지 않음
```
**Client:** [PR #16](https://github.com/0xgeeb/goldilocks-core/pull/16)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

### 문서에 누락된 괄호

**설명:** [문서](https://goldilocks-1.gitbook.io/goldidocs/locks/the-price-function)에 괄호가 누락되었습니다.
```diff
- Market price = Floor Price + ((PSL/Supply)*(FSL+PSL/FSL)^6)
+ Market price = Floor Price + (PSL/Supply)*((FSL+PSL)/FSL)^6
```

**Client:** 확인됨, 수정되었습니다.

**Cyfrin:** 확인됨.

### 1 wei의 반올림 손실

**설명:** `Goldivault`에서 2개의 금액 합계가 반올림 손실로 인해 `amount`보다 작을 수 있습니다.

```solidity
File: Goldivault.sol
159:     if(remainingTime > 0) {
160:       SafeTransferLib.safeTransfer(depositToken, msg.sender, amount * (1000 - _fee) / 1000);
161:       SafeTransferLib.safeTransfer(depositToken, multisig, amount * _fee / 1000); //@audit 반올림 손실
162:       emit OwnershipTokenRedemption(msg.sender, amount * (1000 - _fee) / 1000);
163:     }

File: Goldilend.sol
430:     poolSize += userLoan.interest * (1000 - (multisigShare + apdaoShare)) / 1000;
431:     _updateInterestClaims(interest); //@audit 반올림 손실
...
608:   function _updateInterestClaims(uint256 interest) internal {
609:     multisigClaims += interest * multisigShare / 1000;
610:     apdaoClaims += interest * apdaoShare / 1000;
611:   }
```

**Client:** [PR #17](https://github.com/0xgeeb/goldilocks-core/pull/17)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

## 정보성 (Informational)


### 사용되지 않는 상수

```solidity
File: Goldigovernor.sol
102:   /// @notice Amount of votes to reach quorum
103:   uint256 public constant quorumVotes = 9_500_000e18; // 5% of LOCKS
```



### 사용자로부터 `HONEY`를 두 번 가져오는 나쁜 설계

사용자는 상호 작용 중에 전송을 두 번 승인해야 합니다. 사용자로부터 한 번에 전체 금액을 가져오고 계약에서 `multisig`로 전송하는 것이 더 좋습니다.

```solidity
File: Goldiswap.sol
149:     SafeTransferLib.safeTransferFrom(honey, msg.sender, address(this), price);
150:     SafeTransferLib.safeTransferFrom(honey, msg.sender, multisig, tax);
```

## 가스 최적화 (Gas Optimization)


### 생성자를 `payable`로 표시할 수 있음

Payable 함수는 컴파일러가 지불이 제공되지 않았는지 확인하는 추가 검사를 추가할 필요가 없으므로 실행 비용이 적게 듭니다. 배포자만 자금을 전달할 수 있고 프로젝트 자체는 자금을 전달하지 않으므로 생성자를 안전하게 `payable`로 표시할 수 있습니다. (9개 인스턴스)



### `if`-문 중첩은 `andand`를 사용하는 것보다 저렴함

`if`-문을 중첩하면 추가 jumpdest를 설정하고 사용하는 스택 작업을 피하고 6 가스를 절약할 수 있습니다.

```solidity
File: GovLocks.sol
214:     if (srcRep != dstRep && amt > 0) {
237:     if (nCheckpoints > 0 && checkpoints[delegatee][nCheckpoints - 1].fromBlock == block.number) {
266:     if(from != address(0) && to != address(0)) {

File: Goldilend.sol
536:     return _poolSize > 0 && supply > 0 ? FixedPointMathLib.mulWad(lockAmount, _GiBGTRatio(supply, _poolSize)) : lockAmount;

File: Goldiswap.sol
329:     if (elapsedIncrease >= 1 days && elapsedDecrease >= 1 days) {
```



### 변경 시 가스를 절약하기 위해 `true/false` 대신 `uint256(1)/uint256(2)` 사용

과거에 true였던 후 `false`에서 `true`로 변경할 때 20000 가스를 방지합니다.

```solidity
File: Goldilend.sol
107:   bool public borrowingActive;

File: Goldivault.sol
91:   bool public concluded;
```



### 증강 할당 연산자는 상태 변수에 대한 일반 덧셈보다 더 많은 가스 비용이 듬

상태 변수에 대해 일반 덧셈 연산(`x=x+y`)은 증강 할당 연산자(`x+=y`)보다 가스 비용이 적게 듭니다 (113 가스). (34개 인스턴스)


### 루프 외부에서 배열 길이를 캐시하고 unchecked 루프 증가 고려

```solidity
File: Goldilend.sol
251:   function boost(
252:     address[] calldata partnerNFTs,
253:     uint256[] calldata partnerNFTIds
254:   ) external {
255:     for(uint256 i; i < partnerNFTs.length; i++) {
256:       if(partnerNFTBoosts[partnerNFTs[i]] == 0) revert InvalidBoostNFT();
257:     }
258:     if(partnerNFTs.length != partnerNFTIds.length) revert ArrayMismatch();
259:     boosts[msg.sender] = _buildBoost(partnerNFTs, partnerNFTIds);
260:     for(uint8 i; i < partnerNFTs.length; i++) {
261:       IERC721(partnerNFTs[i]).safeTransferFrom(msg.sender, address(this), partnerNFTIds[i]);
262:     }
263:   }
```

\clearpage

