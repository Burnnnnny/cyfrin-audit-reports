**수석 감사 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

[0kage](https://twitter.com/0kage_eth)

**보조 감사 (Assisting Auditors)**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 공격자가 ETH를 수령할 때 의도적으로 트랜잭션을 되돌려(revert) 언스테이킹 중 DOS를 유발할 수 있음

**설명:** `fulfillUnstake()` 함수는 사용자의 언스테이크 요청을 처리하기 위해 내부적으로 사용됩니다. 이 함수는 ETH를 전송하기 위해 `userAddress`에 대한 저수준 호출(low-level call)을 수행하며 전송이 실패하면 트랜잭션을 되돌립니다. 또한 계약은 모든 언스테이크 요청을 선입선출(FIFO) 대기열에서 처리합니다. 즉, 나중 요청을 처리하기 전에 이전 요청을 처리해야 합니다.

공격자는 `receive()` 함수에서 의도적으로 되돌리기(revert)를 트리거하여 이를 악용할 수 있습니다. 이 조치는 `fulfillUnstake()`를 되돌리고 전체 언스테이크 대기열을 차단하게 됩니다.
```solidity
function fulfillUnstake(address userAddress, uint256 amount) private {
    (bool success,) = userAddress.call{value: amount}(""); // @audit DOS by reverting on `receive()`
    if (!success) {
        revert TransferFailed();
    }
    emit UnstakeFulfilled(userAddress, amount);
}
```

**영향:** 이로 인해 모든 언스테이크 요청에 대한 서비스 거부(DoS)가 발생하여 사용자의 자금이 잠길 수 있습니다.

**권장 완화 방법:** Pull-over-Push 패턴 사용을 고려하십시오.
참조: https://fravoll.github.io/solidity-patterns/pull_over_push.html

**Casimir:**
[cdbe7b1](https://github.com/casimirlabs/casimir-contracts/commit/cdbe7b1ed9e61a58d7971087e9b6e582eb36a55b)에서 수정됨

**Cyfrin:** 확인됨.


### `claimEffectiveBalance()` 함수가 지속적으로 되돌려져(revert) 대기열 인출을 완료할 수 없게 될 수 있음

**설명:** 이 함수는 인덱스 `0`의 인출을 제거하려고 시도하는 반면 인덱스 `i`의 인출을 사용하여 `completeQueuedWithdrawal()`을 호출합니다. 각 인출은 한 번만 완료할 수 있으므로 `delayedEffectiveBalanceQueue[]` 목록에는 결국 이미 완료된 인출이 포함됩니다. 함수가 이미 완료된 인출을 완료하려고 시도하면 변함없이 되돌려집니다.

```solidity
for (uint256 i; i < delayedEffectiveBalanceQueue.length; i++) {
    IDelegationManager.Withdrawal memory withdrawal = delayedEffectiveBalanceQueue[i];
    if (uint32(block.number) - withdrawal.startBlock > withdrawalDelay) {
        delayedEffectiveBalanceQueue.remove(0); // @audit Remove withdrawal at index 0
        claimedEffectiveBalance += withdrawal.shares[0];
        eigenDelegationManager.completeQueuedWithdrawal(withdrawal, tokens, 0, true); // @audit Complete withdrawal of index i
    } else {
        break;
    }
}
```

**영향:** `claimEffectiveBalance()` 함수가 지속적으로 되돌려져 대기열 인출을 완료할 수 없게 되어 ETH가 잠깁니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오.

1. 처음에 `delayedEffectiveBalanceQueue[]` 목록에는 5개의 인출 `[a, b, c, d, e]`가 포함됩니다.
2. `claimEffectiveBalance()` 함수가 호출됩니다.
    - 첫 번째 루프 반복 `i = 0`에서 인출 `a`가 제거되고 완료됩니다. 목록은 이제 `[b, c, d, e]`가 됩니다.
    - 두 번째 루프 반복 `i = 1`에서 인출 `b`가 제거되지만 인출 `c`가 완료됩니다. 목록은 이제 `[c, d, e]`가 됩니다.
    - 세 번째 루프 반복 `i = 2`에서 함수는 인출 `e`를 확인하고 인출 지연이 아직 도달하지 않았다고 가정합니다. 루프는 이 시점에서 중단되고 함수가 중지됩니다.
3. 다음에 `claimEffectiveBalance()` 함수가 호출됩니다.
    - 첫 번째 루프 반복 `i = 0`에서 함수는 인출 `c`를 제거하고 완료하려고 시도합니다. 그러나 인출 `c`가 이미 완료되었으므로 `completeQueuedWithdrawal()` 호출은 되돌려집니다.

**권장 완화 방법:** 인출 확인, 제거 및 완료에 일관된 인덱스를 사용하는 것을 고려하십시오.

**Casimir:**
[35fdf1e](https://github.com/casimirlabs/casimir-contracts/commit/35fdf1e42ad2a38f47028a8468efc0e78e6e7f67)에서 수정됨

**Cyfrin:** 확인됨.


### 내부 회계를 업데이트하지 않고 지연된 보상을 청구할 수 있음

**설명:** `claimRewards()` 함수는 EigenLayer 지연된 인출 라우터(Delayed Withdrawal Router)에서 지연된 인출을 청구하고 `delayedRewards` 및 `reservedFeeBalance`와 같은 회계 변수를 업데이트하도록 설계되었습니다.

```solidity
function claimRewards() external {
    onlyReporter();

    uint256 initialWithdrawalsBalance = address(eigenWithdrawals).balance;
    eigenWithdrawals.claimDelayedWithdrawals(
        eigenWithdrawals.getClaimableUserDelayedWithdrawals(address(this)).length
    );
    uint256 claimedAmount = initialWithdrawalsBalance - address(eigenWithdrawals).balance;
    delayedRewards -= claimedAmount;

    uint256 rewardsAfterFee = subtractRewardFee(claimedAmount);
    reservedFeeBalance += claimedAmount - rewardsAfterFee;
    distributeStake(rewardsAfterFee);

    emit RewardsClaimed(rewardsAfterFee);
}
```

그러나 이 함수는 `DelayedWithdrawalRouter::claimDelayedWithdrawals()` 함수를 통해 EigenLayer 측에서 직접 청구를 실행하여 우회할 수 있습니다. 이 함수를 사용하면 호출자가 지정된 수신자에 대한 인출을 청구할 수 있으며 수신자의 주소가 입력으로 제공됩니다. `CasimirManager` 계약 주소가 `recipient`로 사용되면 해당 주소를 대신하여 청구가 이루어집니다.

**영향:** 이 프로세스는 회계 변수를 업데이트하지 않아 계약 내에서 부정확한 회계로 이어집니다. 보상이 청구되었더라도 여전히 `delayedRewards`에 포함되어 잘못된 총 스테이크 값이 발생합니다.

**개념 증명 (Proof of Concept):** 지연된 인출 청구를 처리하는 EigenLayer 계약은 [여기](https://github.com/Layr-Labs/eigenlayer-contracts/blob/0139d6213927c0a7812578899ddd3dda58051928/src/contracts/pods/DelayedWithdrawalRouter.sol#L80)에서 찾을 수 있습니다.

**권장 완화 방법:** 계약이 보상 청구를 관리하는 방식을 변경하는 것을 고려하십시오. 청구된 보상 금액에 대한 회계를 `receive()` 함수로 이동하고 `eigenWithdrawals` 계약에서 받은 자금만 필터링하여 이를 달성할 수 있습니다.

**Casimir:**
[4adef64](https://github.com/casimirlabs/casimir-contracts/commit/4adef6482238c3d0926f72ffdff04e7a49886045)에서 수정됨

**Cyfrin:** 확인됨.


### 누구나 EigenPod `verifyAndProcessWithdrawals`를 통해 증명을 제출하여 `withdrawRewards`의 회계를 깨뜨릴 수 있음

**설명:** `CasimirManager::withdrawRewards`는 아래의 주요 작업을 수행하는 `onlyReporter` 작업입니다.

1. 주어진 인덱스에서 검증인의 부분 인출과 관련된 증명을 제출합니다.
2. `userDelayedWithdrawalByIndex` 배열의 마지막 요소를 기반으로 `delayedRewards`를 업데이트합니다.

파드(pod) 소유자뿐만 아니라 누구나 `EigenPod::verifyAndProcessWithdrawals`에 직접 증명을 제출할 수 있다는 점에 유의하십시오. 이 경우 `delayedRewards`가 업데이트되지 않으며 보고서 확정 중 후속 회계가 깨지게 됩니다.

`CasimirManager::withdrawRewards`를 호출하여 보상을 인출하려는 시도는 인출이 이미 처리되었기 때문에 되돌려집니다. 결과적으로 `delayedRewards`는 업데이트되지 않습니다.

동일한 문제가 전체 인출 처리를 위한 증명을 제출할 때도 적용됩니다. `CasimirManager::withdrawValidator`에서 업데이트되는 중요한 회계 매개변수는 증명이 EigenLayer에 직접 제출될 때 효과적으로 우회됩니다.

**영향:** `delayedRewards`가 업데이트되지 않으면 `rewardStakeRatioSum` 및 `latestActiveBalanceAfterFee` 회계가 깨집니다.

**권장 완화 방법:** EigenLayer는 파드 소유자에게만 인출 처리를 제한하지 않습니다. 그 정도까지 `CasimirManager::withdrawRewards`에 대한 액세스 제어는 항상 우회될 수 있습니다. 모든 인출이 리포터(reporter)를 통해서만 발생한다고 가정하고 EigenLayer의 `eigenWithdrawals.delayedWithdrawals` 및 `eigenWithdrawals.delayedWithdrawalsCompleted`를 직접 추적하여 지연된 보상을 계산하는 로직을 추가하는 것을 고려하십시오.

**Casimir:**
[eb31b43](https://github.com/casimirlabs/casimir-contracts/commit/eb31b4349e69eb401615e0eca253e9ab8cc0999d)에서 수정됨

**Cyfrin:** 확인됨.


### 증명을 제출하여 withdrawValidator를 선행 매매(front-run)하면 EigenLayer에서 검증인 언스테이킹을 영구적으로 DOS할 수 있음

**설명:** 공격자는 멤풀을 관찰하고 동일한 증명을 EigenLayer에 직접 제출하여 `CasimirManager::withdrawValidator` 트랜잭션을 선행 매매할 수 있습니다.

증명이 이미 검증되었으므로 `CasimirManager::withdrawValidator` 트랜잭션이 동일한 증명을 제출하려고 할 때 되돌려집니다. 빈 증명을 제출하여 증명 검증을 우회하는 것도 작동하지 않습니다. `finalEffectiveBalance`가 항상 0이 되어 인출 대기열에 추가하는 것을 방지하기 때문입니다.

````solidity
function withdrawValidator(
    uint256 stakedValidatorIndex,
    WithdrawalProofs memory proofs,
    ISSVClusters.Cluster memory cluster
) external {
    onlyReporter();

   // ..code...

    uint256 initialDelayedRewardsLength = eigenWithdrawals.userWithdrawalsLength(address(this)); //@note this holds the rewards
    uint64 initialDelayedEffectiveBalanceGwei = eigenPod.withdrawableRestakedExecutionLayerGwei(); //@note this has the current ETH balance

>   eigenPod.verifyAndProcessWithdrawals(
        proofs.oracleTimestamp,
        proofs.stateRootProof,
        proofs.withdrawalProofs,
        proofs.validatorFieldsProofs,
        proofs.validatorFields,
        proofs.withdrawalFields
    ); //@audit reverts if proof is already verified

    {
        uint256 updatedDelayedRewardsLength = eigenWithdrawals.userWithdrawalsLength(address(this));
        if (updatedDelayedRewardsLength > initialDelayedRewardsLength) {
            IDelayedWithdrawalRouter.DelayedWithdrawal memory withdrawal =
                eigenWithdrawals.userDelayedWithdrawalByIndex(address(this), updatedDelayedRewardsLength - 1);
            if (withdrawal.blockCreated == block.number) {
                delayedRewards += withdrawal.amount;
                emit RewardsDelayed(withdrawal.amount);
            }
        }
    }

    uint64 updatedDelayedEffectiveBalanceGwei = eigenPod.withdrawableRestakedExecutionLayerGwei();
>   uint256 finalEffectiveBalance =
        (updatedDelayedEffectiveBalanceGwei - initialDelayedEffectiveBalanceGwei) * GWEI_TO_WEI; //@audit if no proofs submitted, this will be 0
    delayedEffectiveBalance += finalEffectiveBalance;
    reportWithdrawnEffectiveBalance += finalEffectiveBalance;
}

//... code
````

**영향:** 공격자는 무시할 수 있는 비용으로 EigenLayer에서 검증인 인출을 방지하여 Casimir에 지급 불능 위험을 초래할 수 있습니다.

**권장 완화 방법:** `CasimirManager::withdrawValidators`에 대해 다음과 같은 변경을 고려하십시오.

- 함수를 두 개의 별도 함수, 즉 전체 인출 검증용 함수와 검증된 인출 대기열 추가용 함수로 분할합니다.
- `EigenPod::_processFullWithdrawal`에서 다음 이벤트 방출을 수신하고 수신자가 `CasimirManager`인 이벤트를 필터링합니다.
````solidity
event FullWithdrawalRedeemed(
    uint40 validatorIndex,
    uint64 withdrawalTimestamp,
    address indexed recipient,
    uint64 withdrawalAmountGwei
);
````
- 해당되는 각 검증인 인덱스에 대한 인출을 검증하는 동안 try-catch를 도입합니다. 증명이 이미 검증된 경우 `stakedValidatorIds` 배열을 업데이트하고 catch 섹션에서 `requestedExits`를 줄입니다.
- 모든 인출이 검증되면 이벤트 방출을 사용하여 `eigenDelegationManager.queueWithdrawals(params)`를 내부적으로 호출하는 두 번째 함수로 전송될 `QueuedWithdrawalParams` 배열을 생성합니다.
- 이 단계에서 `delayedEffectiveBalance` 및 `reportWithdrawnEffectiveBalance`를 업데이트합니다.

참고: 이것은 `reporter`가 프로토콜 제어라고 가정합니다.

**Casimir:**
[eb31b43](https://github.com/casimirlabs/casimir-contracts/commit/eb31b4349e69eb401615e0eca253e9ab8cc0999d)에서 수정됨

**Cyfrin:** 확인됨. 인출 증명은 EigenLayer의 대기열 인출과 분리됩니다. 이는 이 문제에서 보고된 서비스 거부 위험을 성공적으로 완화합니다. 그러나 유효 잔액이 32 이더로 하드코딩되어 있습니다.

EigenLayer에서 전체 인출 이벤트를 모니터링하여 유효 잔액을 매개변수로 전달하는 것이 좋습니다.


### `tipBalance`의 잘못된 회계로 인해 보고서 실행이 무기한 중단될 수 있음

**설명:** `CasimirManager`의 `receive` 폴백(fallback) 함수는 발신자가 `DelayedWithdrawalRouter`가 아닌 경우 `tip` 잔액을 증가시킵니다. 여기서 암시적 가정은 전체 또는 부분 인출이든 모든 인출이 `DelayedWithdrawalRouter`를 통해 라우팅된다는 것입니다. 이 가정은 부분 인출(보상)의 경우 사실이지만, 발신자가 `DelayedWithdrawalRouter`가 아니라 `EigenPod` 자체인 전체 인출의 경우에는 잘못된 가정입니다.

```solidity
    receive() external payable {
        if (msg.sender != address(eigenWithdrawals)) {
            tipBalance += msg.value; //@audit tip balance increases even incase of full withdrawals

            emit TipsReceived(msg.value);
        }
    }

```

소유자는 `CasimirManager:claimTips`를 통해 모든 팁을 완전히 인출할 수 있습니다.

```solidity
    function claimTips() external {
        onlyReporter();

        uint256 tipsAfterFee = subtractRewardFee(tipBalance);
        reservedFeeBalance += tipBalance - tipsAfterFee;
        tipBalance = 0;
        distributeStake(tipsAfterFee);
        emit TipsClaimed(tipsAfterFee);
    }

```

소유자가 팁을 청구할 때 `CasimirManager::claimEffectiveBalance`를 통해 전체 인출을 청구하는 동안 증가된 `withdrawnEffectiveBalance`는 실제 ETH 잔액과 동기화되지 않습니다. 사실상 `CasimirManager.balance >= withdrawnEffectiveBalance`라는 주요 불변성이 위반될 수 있습니다.

```solidity
    function claimEffectiveBalance() external {
        onlyReporter();

        IStrategy[] memory strategies = new IStrategy[](1);
        strategies[0] = IDelegationManagerViews(address(eigenDelegationManager)).beaconChainETHStrategy();
        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(address(0));

        uint256 withdrawalDelay = eigenDelegationManager.strategyWithdrawalDelayBlocks(strategies[0]);
        uint256 claimedEffectiveBalance;
        for (uint256 i; i < delayedEffectiveBalanceQueue.length; i++) {
            IDelegationManager.Withdrawal memory withdrawal = delayedEffectiveBalanceQueue[i];
            if (uint32(block.number) - withdrawal.startBlock > withdrawalDelay) {
                delayedEffectiveBalanceQueue.remove(0);
                claimedEffectiveBalance += withdrawal.shares[0];
                eigenDelegationManager.completeQueuedWithdrawal(withdrawal, tokens, 0, true);
            } else {
                break;
            }
        }

        delayedEffectiveBalance -= claimedEffectiveBalance;
        withdrawnEffectiveBalance += claimedEffectiveBalance; //@audit this accounting entry should be backed by ETH balance at all times
    }

```

**영향:** `tipBalance` 계산의 회계 오류는 `withdrawnEffectiveBalance`가 CasimirManager의 실제 ETH 잔액과 동기화되지 않으므로 `fulfilUnstakes` 실패를 유발할 수 있습니다. 이로 인해 보고서 실행이 때때로 중단될 수 있습니다.

**권장 완화 방법:** 발신자가 `DelayedWithdrawalRouter`도 아니고 `EigenPod`도 아닌 경우에만 `tipBalance`를 증가시키는 것을 고려하십시오.

**Casimir:**
[4adef64](https://github.com/casimirlabs/casimir-contracts/commit/4adef6482238c3d0926f72ffdff04e7a49886045)에서 수정됨

**Cyfrin:** 확인됨.

\clearpage
## 높은 위험 (High Risk)


### `getTotalStake()` 함수가 보류 중인 검증인을 설명하지 못해 부정확한 회계 발생

**설명:** `getTotalStake()` 함수는 `CasimirManager`의 총 스테이크를 계산하는 핵심 회계 함수입니다. `finalizeReport()` 함수 내에서 `rewardStakeRatioSum`의 변경 사항을 계산하는 데 사용됩니다.

```solidity
function getTotalStake() public view returns (uint256 totalStake) {
  // @audit Validators in pending state is not accounted for
  totalStake = unassignedBalance + readyValidatorIds.length * VALIDATOR_CAPACITY + latestActiveBalanceAfterFee
      + delayedEffectiveBalance + withdrawnEffectiveBalance + subtractRewardFee(delayedRewards) - unstakeQueueAmount;
}
```

이 함수는 각 "준비된 검증인"(`readyValidatorIds.length * VALIDATOR_CAPACITY`)의 `32 ETH`와 "스테이킹된 검증인"(`latestActiveBalanceAfterFee`)에 스테이킹된 ETH를 포함하여 다양한 소스의 스테이크를 집계합니다.

그러나 보류 중인 검증인의 ETH를 설명하지 못합니다.

검증인이 활성/스테이킹 상태가 되려면 세 단계를 거쳐야 합니다.

1. 사용자가 입금할 때마다 계약은 할당되지 않은 잔액이 `32 ETH`에 도달했는지 확인합니다. 도달하면 다음 검증인 ID가 `readyValidatorIds`에 추가됩니다.
2. 리포터는 `depositValidator()`를 호출하여 검증인으로부터 `32 ETH`를 비콘 예치 계약에 예치합니다. 이 단계에서 검증인 ID는 `readyValidatorIds`에서 `pendingValidatorIds`로 이동합니다.
3. 리포터는 `activateValidator()`를 호출하여 검증인 ID를 `pendingValidatorIds`에서 `stakedValidatorIds`로 이동하고 비콘 체인의 총 스테이크를 반영하도록 `latestActiveBalanceAfterFee`를 업데이트합니다.

보시다시피 `getTotalStake()` 함수는 1단계와 3단계의 검증인을 고려하지만 보류 상태(2단계)의 검증인 스테이크는 무시합니다. 현재 설계에서는 보류 중인 검증인 > 0인 경우 보고서 확정을 중지하는 것이 없습니다.

**영향:** `getTotalStake()` 함수는 총 스테이크 값을 실제보다 적게 반환합니다. 결과적으로 보고서 확정 전에 모든 보류 중인 검증인이 활성화되지 않은 시나리오에서 `rewardStakeRatioSum` 계산이 올바르지 않게 됩니다.

```solidity
uint256 totalStake = getTotalStake();

rewardStakeRatioSum += Math.mulDiv(rewardStakeRatioSum, gainAfterFee, totalStake);

rewardStakeRatioSum += Math.mulDiv(rewardStakeRatioSum, gain, totalStake);

rewardStakeRatioSum -= Math.mulDiv(rewardStakeRatioSum, loss, totalStake);
```

**권장 완화 방법:** `getTotalStake()` 함수에 `pendingValidatorIds.length * VALIDATOR_CAPACITY`를 추가하는 것을 고려하십시오.

**Casimir:**
[10fe228](https://github.com/casimirlabs/casimir-contracts/commit/10fe228406dc3f889db42e8850add94561a7325e)에서 수정됨

**Cyfrin:** 확인됨.


### `withdrawValidator`의 하드코딩된 클러스터 크기로 인해 운영자 또는 프로토콜에 손실이 발생할 수 있음

**설명:** `CasimirManager::withdrawValidator`는 인출 시 빚진 잔액, 즉 초기 32 이더에서 부족한 금액을 계산합니다. 그런 다음 검증인에 태그된 운영자로부터 빚진 금액을 회수하려고 시도합니다. 그러나 회수 금액을 계산할 때 하드코딩된 클러스터 크기 `4`가 사용됩니다.

```solidity
function withdrawValidator(
    uint256 stakedValidatorIndex,
    WithdrawalProofs memory proofs,
    ISSVClusters.Cluster memory cluster
) external {
    onlyReporter();

    // ... more code

    uint256 owedAmount = VALIDATOR_CAPACITY - finalEffectiveBalance;
    if (owedAmount > 0) {
        uint256 availableCollateral = registry.collateralUnit() * 4;
        owedAmount = owedAmount > availableCollateral ? availableCollateral : owedAmount;
>       uint256 recoverAmount = owedAmount / 4; //@audit hardcoded operator size
        for (uint256 i; i < validator.operatorIds.length; i++) {
            registry.removeOperatorValidator(validator.operatorIds[i], validatorId, recoverAmount);
        }
    }

    // .... more code

```

**영향:** 여기에는 2가지 부작용이 있습니다.
1. 운영자는 클러스터 크기가 큰 전략에서 담보 잔액의 더 높은 %를 잃습니다.
2. 반면에 `owedAmount`는 `4 * collateralUnit`으로 제한되므로 프로토콜이 필요한 것보다 적게 회수할 가능성도 있습니다.

**권장 완화 방법:** 하드코딩된 숫자 대신 전략의 `clusterSize`를 사용하는 것을 고려하십시오.

**Casimir:**
[7497e8c](https://github.com/casimirlabs/casimir-contracts/commit/7497e8cefae018a46606b4722c9bc20d03d0d23c)에서 수정됨.

**Cyfrin:** 확인됨.


### 악의적인 스테이커가 즉시 스테이킹 및 언스테이킹하여 검증인 인출을 강제할 수 있음

**설명:** 사용자가 `CasimirManager::requestUnstake`를 통해 언스테이킹할 때 필요한 검증인 퇴장 수는 다음과 같이 일반적인 예상 인출 가능 잔액을 사용하여 계산됩니다.

```solidity
function requestUnstake(uint256 amount) external nonReentrant {
    // code ....
    uint256 expectedWithdrawableBalance =
        getWithdrawableBalance() + requestedExits * VALIDATOR_CAPACITY + delayedEffectiveBalance;
    if (unstakeQueueAmount > expectedWithdrawableBalance) {
        uint256 requiredAmount = unstakeQueueAmount - expectedWithdrawableBalance;
>       uint256 requiredExits = requiredAmount / VALIDATOR_CAPACITY; //@audit required exits calculated here
        if (requiredAmount % VALIDATOR_CAPACITY > 0) {
            requiredExits++;
        }
        exitValidators(requiredExits);
    }

    emit UnstakeRequested(msg.sender, amount);
}
```

다음과 같은 단순화된 시나리오를 고려하십시오.

`unAssignedBalance = 31 ETH withdrawnBalance = 0 delayedEffectiveBalance = 0 requestedExits = 0`

또한 단순화를 위해 `deposit fees = 0%`라고 가정합니다.

악의적인 검증인인 Alice는 1 ETH를 스테이킹합니다. 이것은 `distributeStakes`를 통해 할당되지 않은 잔액을 새 검증인에게 할당합니다. 이 시점의 상태는 다음과 같습니다.

`unAssignedBalance = 0 ETH withdrawnBalance = 0 delayedEffectiveBalance = 0 requestedExits = 0`

Alice는 즉시 `requestUnstake`를 통해 1 ETH에 대한 언스테이크 요청을 합니다. 언스테이크를 이행하기에 충분한 잔액이 없기 때문에 기존 검증인은 비콘 체인에서 강제로 인출해야 합니다. 그 후 상태는 다음과 같습니다.

`unAssignedBalance = 0 ETH withdrawnBalance = 0 delayedEffectiveBalance = 0 requestedExits = 1`

이제 Alice는 공격을 반복할 수 있으며, 이번에는 즉시 64 ETH를 입금하고 인출합니다. 이것이 끝나면 상태는 다음과 같습니다.

`unAssignedBalance = 0 ETH withdrawnBalance = 0 delayedEffectiveBalance = 0 requestedExits = 2`

매번 Alice는 입금 수수료와 가스비만 잃으면 되지만 잠재적 보상을 잃는 진정한 스테이커와 검증인에서 강제로 쫓겨난 운영자를 괴롭힐 수 있습니다.

**영향:** 불필요한 검증인 인출 요청은 스테이커, 운영자 및 프로토콜 자체를 괴롭힙니다. 검증인 퇴장은 스테이커에게 수익 손실을 초래하고 프로토콜에 가스 비용이 많이 듭니다.

**권장 완화 방법:**
- 언스테이크 잠금 기간을 고려하십시오. 사용자는 입금 후 최소 시간/블록이 경과할 때까지 언스테이킹을 요청할 수 없습니다.
- 검증인을 먼저 퇴장시키는 대신 `readyValidators`에서 ETH를 제거하는 것을 고려하십시오. -> 활성 검증인은 이미 보상을 적립하고 있지만 준비된 검증인은 아직 프로세스를 시작하지 않았습니다. 그리고 운영자 제거, SSV 클러스터 등록 해제와 관련된 오버헤드는 준비된 검증인에서 ETH가 할당 해제되면 필요하지 않습니다.

**Casimir:**
[4a5cd14](https://github.com/casimirlabs/casimir-contracts/commit/4a5cd145c247d9274c3f21f9e9c1b5557a230a01)에서 수정됨

**Cyfrin:** 확인됨.


### 검증인의 `owedAmount == 0`일 때 레지스트리에서 운영자가 제거되지 않음

**설명:** `CasimirManager::withdrawValidator()` 함수는 전체 인출 후 검증인을 제거하도록 설계되었습니다. 제거된 검증인의 최종 유효 잔액이 초기 32 ETH 예치금을 충당하기에 충분한지 확인합니다. 슬래싱과 같은 이유로 최종 유효 잔액이 32 ETH 미만인 경우 운영자는 `registry.removeOperatorValidator()`를 호출하여 누락된 부분을 회수해야 합니다.

```solidity
uint256 owedAmount = VALIDATOR_CAPACITY - finalEffectiveBalance;
if (owedAmount > 0) {
    uint256 availableCollateral = registry.collateralUnit() * 4;
    owedAmount = owedAmount > availableCollateral ? availableCollateral : owedAmount;
    uint256 recoverAmount = owedAmount / 4;
    for (uint256 i; i < validator.operatorIds.length; i++) {
        // @audit if owedAmount == 0, this function is not called
        registry.removeOperatorValidator(validator.operatorIds[i], validatorId, recoverAmount);
    }
}
```

그러나 `removeOperatorValidator()` 함수는 `operator.validatorCount`와 같은 다른 운영자 상태를 업데이트할 책임도 있습니다. `owedAmount > 0`일 때만 이 함수가 호출되면 검증인이 32 ETH를 완전히 반환하는 경우 이러한 운영자의 상태가 업데이트되지 않습니다.

**영향:** 검증인이 제거될 때 `CasimirRegistry`에서 `operator.validatorCount`가 감소하지 않습니다. 결과적으로 운영자는 이 검증인에 대한 담보를 인출할 수 없으며 담보는 `CasimirRegistry` 계약에 잠긴 상태로 유지됩니다.

**권장 완화 방법:** `owedAmount == 0`일 때 `registry.removeOperatorValidator()` 함수도 `recoverAmount = 0`으로 호출되어야 합니다. 이렇게 하면 운영자의 담보가 확보됩니다.

**Casimir:**
[d7b35fc](https://github.com/casimirlabs/casimir-contracts/commit/d7b35fce9925bfa2133fd4e16ae11e483ab4daa4)에서 수정됨

**Cyfrin:** 확인됨.


### 지연된 잔액이나 보상이 청구되지 않은 경우 `rewardStakeRatioSum` 회계가 올바르지 않음

**설명:** 현재 회계는 지연된 유효 잔액과 지연된 보상이 새로운 보고서가 시작되기 전에 청구될 것이라고 잘못 가정합니다. 이러한 지연된 자금은 청구되기까지 며칠이 필요하므로 이는 부정확합니다. 이 자금이 청구되기 전에 새 보고서가 시작되면 `reportSweptBalance`는 이를 다시 계산합니다. 이 이중 회계는 `rewardStakeRatioSum` 계산에 영향을 미쳐 부정확성을 초래합니다.

**영향:** `rewardStakeRatioSum`에 대한 회계가 올바르지 않아 부정확한 사용자 스테이크가 발생합니다. 결과적으로 사용자는 언스테이킹 시 예상보다 많은 ETH를 받을 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오.

1. 처음에 하나의 검증인(32 ETH)이 스테이킹되고 비콘 체인 보상은 0.105 ETH라고 가정합니다. 보고서가 처리됩니다.
```solidity
// Before start
latestActiveBalanceAfterFee = 32 ETH
latestActiveRewards = 0

// startReport()
reportSweptBalance = 0 (rewards is in BeaconChain)

// syncValidators()
reportActiveBalance = 32.105 ETH

// finalizeReport()
rewards = 0.105 ETH
change = rewards - latestActiveRewards = 0.105 ETH
gainAfterFee = 0.1 ETH
=> rewardStakeRatioSum is increased
=> latestActiveBalanceAfterFee = 32.1

sweptRewards = 0
=> latestActiveRewards = 0.105
```

2. 비콘 체인은 0.105 ETH 보상을 휩씁니다(sweep). 그 다음 또 다른 보고서 처리가 이어집니다.
```solidity
// Before start
latestActiveBalanceAfterFee = 32.1 ETH
latestActiveRewards = 0.105

// startReport()
reportSweptBalance = 0.105 (rewards is in EigenPod)

// syncValidators()
reportActiveBalance = 32 ETH

// finalizeReport()
rewards = 0.105 ETH
change = rewards - latestActiveRewards = 0
=> No update to rewardStakeRatioSum and latestActiveBalanceAfterFee

sweptRewards = 0.105
=> latestActiveBalanceAfterFee = 32 ETH (subtracted sweptReward without fee)
=> latestActiveRewards = rewards - sweptRewards = 0
```

3. 작업이 수행되지 않는다고 가정합니다. 즉, 보상이 여전히 EigenPod에 있고 아직 청구되지 않았습니다. 다음 보고서가 처리됩니다.
```solidity
// Before start
latestActiveBalanceAfterFee = 32 ETH
latestActiveRewards = 0

// startReport()
reportSweptBalance = 0.105 (No action happens so rewards is still in EigenPod)

// syncValidators()
reportActiveBalance = 32 ETH

// finalizeReport()
rewards = 0.105 ETH
change = rewards - latestActiveRewards = 0.105
=> rewardStakeRatioSum is increased
=> latestActiveBalanceAfterFee = 32.1

sweptRewards = 0.105
=> latestActiveBalanceAfterFee = 32 ETH (subtracted sweptReward without fee)
=> latestActiveRewards = rewards - sweptRewards = 0
```

마지막 보고서와 현재 보고서 사이에 아무런 조치도 발생하지 않으므로 `latestActiveBalanceAfterFee` 및 `latestActiveReward` 값은 동일하게 유지됩니다. 그러나 `rewardStakeRatioSum` 값은 아무것도 없는 상태에서 증가했습니다. 이 보고 프로세스가 계속되면 `rewardStakeRatioSum`이 무한히 증가할 수 있습니다. 결과적으로 사용자 스테이크의 핵심 회계가 올바르지 않게 되고 사용자는 언스테이킹 시 예상보다 많은 ETH를 받을 수 있습니다.

**권장 완화 방법:** 회계 로직을 검토하여 지연된 유효 잔액과 지연된 보상이 보고서에서 한 번만 고려되도록 하십시오.

**Casimir**
[eb31b43](https://github.com/casimirlabs/casimir-contracts/commit/eb31b4349e69eb401615e0eca253e9ab8cc0999d)에서 수정됨

**Cyfrin**
확인됨.


### `reportRecoveredEffectiveBalance`의 잘못된 회계로 인해 검증인이 슬래싱될 때 보고서가 확정되지 않을 수 있음

**설명:** 검증인이 슬래싱되면 손실이 발생합니다. `finalizeReport()` 함수에서 `rewardStakeRatioSum` 및 `latestActiveBalanceAfterFee` 변수는 이 손실을 반영하여 감소합니다. 보상이 슬래싱된 금액보다 크면 변경 사항이 긍정적일 수 있지만 간단히 하기 위해 부정적인 경우에 집중하겠습니다. 여기서 손실이 회계 처리됩니다.

```solidity
} else if (change < 0) {
    uint256 loss = uint256(-change);
    rewardStakeRatioSum -= Math.mulDiv(rewardStakeRatioSum, loss, totalStake);
    latestActiveBalanceAfterFee -= loss;
}
```

그러나 모든 손실은 `CasimirRegistry`의 노드 운영자 담보로 회수됩니다. 사용자나 풀의 관점에서는 보장되는 경우 손실이 없으며 사용자는 전액 보상을 받게 됩니다. 여기서 누락된 회계는 `rewardStakeRatioSum` 및 `latestActiveBalanceAfterFee`가 `reportRecoveredEffectiveBalance` 변수를 사용하여 증가해야 한다는 것입니다.

**영향:** 사용자 또는 풀은 `reportRecoveredEffectiveBalance`로 보장되어야 할 손실을 입습니다. 잘못된 회계로 인해 `latestActiveBalanceAfterFee`가 예상보다 작아집니다. 특정 시나리오에서는 이로 인해 산술 언더플로가 발생하고 보고서 확정이 방지될 수 있습니다. 보고서 확정 없이는 새 검증인을 활성화할 수 없으며 새 보고서 기간을 시작할 수 없습니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오 - 마지막 검증인이 인출될 때 언더플로가 발생하여 보고서 확정이 방지됩니다.

_Report Period 0_
2 validators added

```State:
latestActiveBalanceAfterFee = 64
latestActiveRewards = 0
````

_Report Period 1_
Rewards: 0.1 per validator on BC.
Withdrawal: 32
Unstake request: 15

```
=> start
Eigenpod balance = 32.1
reportSweptBalance = 32.1

=> syncValidator
reportActiveBalance = 32.1
reportWithdrawableValidators = 1

=>withdrawValidator
delayedRewards = 0.1
Slashed = 0
Report Withdrawn Effective Balance = 32
Delayed Effective Balance = 32
Report Recovered Balance = 0

=> finalize
totalStake = 49
expectedWithdrawalEffectiveBalance = 32
expectedEffectiveBalance = 32

Rewards = 0.2

rewardStakeRatioSum = 1004.08
latestActiveBalanceAfterFee (reward adj.) = 64.2
swept rewards = 0.1

latestActiveBalanceAfterFee (swept reward adj) = 64.1
latestActiveBalanceAfterFee (withdrawals adj) = 32.1
latestActiveRewards = 0.1

```

_Report Period 2_
unstake request: 20
last validator exited with slashing of 0.2

```
=> start
Eigenpod balance = 63.9 (32.1 previous + 31.8 slashed)
Delayed effective balance = 32
Delayed rewards = 0.1

reportSweptBalance = 96

=> sync validator
reportActiveBalance = 0
reportWithdrawableValidators = 1

=> withdraw validator
Delayed Effective Balance = 63.8 (32+ 31.8)
Report Recovered Balance = 0.2
Report Withdrawn Effective Balance = 31.8 + 0.2 = 32
Delayed Rewards = 0.1



=> finalizeReport
Total Stake: 29.2
expectedWithdrawalEffectiveBalance = 32
expectedEffectiveBalance = 0
rewards = 64

Change = 63.9
rewardStakeRatioSum: 3201.369
latestActiveBalanceAfterFee (reward adj) = 96
Swept rewards = 64.2

latestActiveBalanceAfterFee (swept reward adj) = 31.8
latestActiveBalanceAfterFee (adj withdrawals) = -0.2 => underflow

latestActiveRewards = -0.2
````
여기서 산술 언더플로가 발생합니다. 올바른 조정은 `reportRecoveredBalance`를 보상에 포함하는 것입니다. 수정 시 다음 상태가 달성됩니다.

=> finalizeReport
````
Total Stake: 29.2
expectedWithdrawalEffectiveBalance = 32
expectedEffectiveBalance = 0
rewards = 64.2 (add 0.2 recoveredEffectiveBalance)

Change = 64.1
rewardStakeRatioSum: 3208.247
latestActiveBalanceAfterFee (reward adj) = 96.2
Swept rewards = 64.2

latestActiveBalanceAfterFee (swept reward adj) = 32
latestActiveBalanceAfterFee (adj withdrawals) = 0

latestActiveRewards = 0
````

**권장 완화 방법:** 회수된 ETH가 `rewardStakeRatioSum` 및 `latestActiveBalanceAfterFee` 계산에 포함되도록 `rewards` 계산에 `reportRecoveredEffectiveBalance`를 추가하는 것을 고려하십시오.

**Casimir:**
[eb31b43](https://github.com/casimirlabs/casimir-contracts/commit/eb31b4349e69eb401615e0eca253e9ab8cc0999d)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## 중간 위험 (Medium Risk)


### `exitValidators()`의 무한 루프로 인해 사용자가 `requestUnstake()`를 호출할 수 없음

**설명:** 사용자가 `requestUnstake()` 함수를 호출하여 ETH 언스테이킹을 요청하면 `CasimirManager` 계약은 현재 예상 인출 가능 잔액이 대기 중인 모든 언스테이킹 요청을 충당하기에 충분한지 계산합니다. 충분하지 않은 경우 `exitValidators()` 함수가 호출되어 모든 언스테이킹 요청을 이행하기에 충분한 ETH를 갖도록 일부 활성 검증인을 종료합니다.

`exitValidators()` 함수에서는 `stakedValidatorIds` 목록을 while 루프로 순회합니다. 활성 검증인을 찾으면 `ssvClusters`를 호출하여 이 검증인을 종료하고 상태를 `ACTIVE`에서 `EXITING`으로 변경합니다. 그러나 검증인 상태가 `ACTIVE`가 아닌 경우 루프 `index`도 업데이트되지 않아 루프가 무한히 계속 실행되지만 `stakedValidatorIds` 목록의 다음 검증인에 도달할 수 없습니다.

```solidity
function exitValidators(uint256 count) private {
    uint256 index = 0;
    while (count > 0) {
        uint32 validatorId = stakedValidatorIds[index];
        Validator storage validator = validators[validatorId];

        // @audit if status != ACTIVE, count and index won't be updated => Infinite loop
        if (validator.status == ValidatorStatus.ACTIVE) {
            count--;
            index++;
            requestedExits++;
            validator.status = ValidatorStatus.EXITING;
            ssvClusters.exitValidator(validator.publicKey, validator.operatorIds);
            emit ValidatorExited(validatorId);
        }
    }
}
```

**영향:**
- `stakedValidatorIds` 목록의 첫 번째 검증인 상태가 활성이 아닌 경우 `requestUnstake()` 함수는 호출자의 전체 가스 한도를 소비하고 되돌립니다(revert).

- 또한 소액 스테이커가 언스테이크 요청을 선행 매매하여 고래 스테이커의 언스테이크 요청을 지연시키는 슬픔(griefing) 공격으로 이어질 수 있습니다. `exitValidators`는 처음에는 성공적으로 실행되지만 고래 스테이커가 호출할 때 무한 루프로 인해 되돌려집니다.

**권장 완화 방법:** 모든 시나리오에서 루프가 중단되도록 `count` 및 `index` 변수를 업데이트하는 것을 고려하십시오.

**Casimir:**
[2945695](https://github.com/casimirlabs/casimir-contracts/commit/29456956e383e48277d604ca54d8fd43d6f31d10)에서 수정됨

**Cyfrin:** 확인됨.


### 인출된 잔액이 모든 언스테이크 요청이 이행된 후 조정되지 않기 때문에 다중 언스테이크 요청으로 인해 서비스 거부가 발생할 수 있음

**설명:** `while` 루프는 특정 수의 언스테이크 요청에 대해 실행되며, 모든 반복에서 언스테이크 요청이 이행 가능한지 확인합니다. 이행 가능한 경우 언스테이크된 금액은 언스테이킹을 요청한 스테이커에게 다시 전송됩니다. 모든 언스테이킹 요청을 존중하면 관리자의 유효 ETH 잔액이 줄어들지만, `getNextUnstake` 함수는 다음 언스테이크 요청이 이행 가능한지 확인하면서 오래된 `withdrawnEffectiveBalance`를 계속 사용합니다.

실제로 `withdrawnEffectiveBalance`는 `while` 루프가 완료된 후에만 조정됩니다.

```solidity
function fulfillUnstakes(uint256 count) external {
    //@note called when report status is in fulfilling unstakes
    onlyReporter();

    if (reportStatus != ReportStatus.FULFILLING_UNSTAKES) {
        revert ReportNotFulfilling();
    } //@note ok => report has to be in this state

    uint256 unstakedAmount;
    while (count > 0) {
        count--;

>       (Unstake memory unstake, bool fulfillable) = getNextUnstake(); //@audit uses the stale withdrawn balance
        if (!fulfillable) {
            break;
        }

        unstakeQueue.remove(0);
>       unstakedAmount += unstake.amount; //@audit unstakedAmount is increased here
>       fulfillUnstake(unstake.userAddress, unstake.amount); //@audit even after ETH is transferred, withdrawn balance is same
    }

    (, bool nextFulfillable) = getNextUnstake();
    if (!nextFulfillable) {
        reportStatus = ReportStatus.FINALIZING;
    }

>   if (unstakedAmount <= withdrawnEffectiveBalance) { //@audit withdrawn balance and unassigned balance adjustment happens here
        withdrawnEffectiveBalance -= unstakedAmount;
    } else {
        uint256 remainder = unstakedAmount - withdrawnEffectiveBalance;
        withdrawnEffectiveBalance = 0;
        unassignedBalance -= remainder;
    }

    unstakeQueueAmount -= unstakedAmount;
}
```
**영향:** `fulfillUnstakes()` 함수는 허용 가능한 인출 가능 잔액보다 더 많은 요청을 이행할 수 있습니다. 이로 인해 함수가 오버플로되어 마지막에 되돌릴 수 있습니다.

**권장 완화 방법:**
`fulfillUnstakes()` 함수에 의해 언스테이크 요청이 이행된 후 `withdrawnEffectiveBalance`를 업데이트하는 것을 고려하십시오.

**Casimir:**
[9f8920f](https://github.com/casimirlabs/casimir-contracts/commit/9f8920f483e5505726e3011132246bfcbea2e629)에서 수정됨

**Cyfrin:** 확인됨.


### `requestUnstake()`를 스팸하여 언스테이크 대기열에서 서비스 거부를 유발

**설명:**
- `requestUnstake()` 함수를 사용하면 사용자가 `amount = 0`인 경우에도 모든 금액을 요청할 수 있습니다.
- 계약은 모든 언스테이크 요청을 선입선출(FIFO) 대기열에서 처리하며, 나중 요청보다 이전 요청을 먼저 처리합니다.
- `remove()` 함수는 O(n)의 시간 복잡성을 가지며 가스를 소비합니다.

이는 공격자가 `requestUnstake()`를 반복적으로 호출하여 언스테이크 대기열을 확장할 수 있으며, 이로 인해 `fulfillUnstake()`의 가스 소비가 블록 가스 한도를 초과할 수 있음을 의미합니다.

**영향:** `fulfillUnstake()`를 호출할 때 과도한 가스 사용으로 인해 블록 가스 한도를 초과하여 DOS가 발생할 수 있습니다.

**권장 완화 방법:** 스팸을 비현실적으로 만들 만큼 충분히 큰 `requestUnstake()` 함수에 대한 최소 언스테이크 금액을 설정하는 것을 고려하십시오.

**Casimir:**
[4a5cd14](https://github.com/casimirlabs/casimir-contracts/commit/4a5cd145c247d9274c3f21f9e9c1b5557a230a01)에서 수정됨

**Cyfrin:** 확인됨.


### 사용자가 언스테이크 요청을 선행 매매(frontrun)하여 손실을 피할 수 있음

**설명:** 검증인이 슬래싱되면 손실이 발생할 수 있으며, 이는 `finalizeReport()` 함수에 반영됩니다. `change`가 0보다 작으면 손실이 발생했음을 나타냅니다. 결과적으로 회계는 `rewardStakeRatioSum`을 업데이트하여 `CasimirManager`의 모든 사용자 스테이크 가치를 줄입니다.

```solidity
} else if (change < 0) {
    uint256 loss = uint256(-change);
    rewardStakeRatioSum -= Math.mulDiv(rewardStakeRatioSum, loss, totalStake);
    latestActiveBalanceAfterFee -= loss;
}
```

그러나 사용자는 언스테이크 요청을 선행 매매하여 이 손실을 피할 수 있습니다. 동일한 `reportPeriod` 내에서 언스테이크 요청을 생성하고 이행할 수 있기 때문입니다. 사용자가 다음 보고서에서 잠재적 손실을 예상하는 경우(멤풀을 관찰하여) 언스테이크를 요청하여 이를 피할 수 있습니다. 계약은 모든 언스테이크 요청을 선입선출(FIFO) 대기열에서 처리하므로 리포터는 나중 요청보다 이전 요청을 먼저 이행해야 합니다.

```solidity
function getNextUnstake() public view returns (Unstake memory unstake, bool fulfillable) {
    // @audit Allow to create and fulfill unstake within the same `reportPeriod`
    if (unstakeQueue.length > 0) {
        unstake = unstakeQueue[0];
        fulfillable = unstake.period <= reportPeriod && unstake.amount <= getWithdrawableBalance();
    }
}
```

**영향:** 이는 불공정으로 이어질 수 있습니다. 선행 매매자는 모든 이익을 유지하면서 손실을 피할 수 있습니다.

**권장 완화 방법:** 언스테이크 요청을 이행하기 전에 대기 또는 지연 기간을 구현하는 것을 고려하십시오. 언스테이크 요청이 생성된 동일한 `reportPeriod`에 이행되도록 허용하지 마십시오. 또한 언스테이킹에 대해 소액의 사용자 수수료를 추가하는 것을 고려하십시오.

**Casimir:**
[28baa81](https://github.com/casimirlabs/casimir-contracts/commit/28baa8191a1b5a27d3ee495dee0d993177bf7e5f)에서 수정됨

**Cyfrin:** 확인됨.


### `Reporter` 역할에 많은 권한이 부여되어 중앙 집중화 위험 발생

**설명:** 현재 설계에서 프로토콜 제어 주소인 `Reporter`는 여러 미션 크리티컬 작업을 실행할 책임이 있습니다. `Reporter` 전용 작업에는 보고서 시작 및 확정, 운영자 선택 및 교체, 검증인 동기화/활성화/인출 및 입금, EigenLayer의 보상 검증 및 청구 등이 포함됩니다. 또한 이러한 작업의 타이밍과 순서가 Casimir 프로토콜의 적절한 기능에 중요하다는 사실도 주목할 만합니다.

**영향:** 단일 주소로 제어되는 작업이 너무 많고 그 중 상당 부분이 오프체인에서 시작되므로 프로토콜은 중앙 집중화와 관련된 모든 위험에 노출됩니다. 알려진 위험 중 일부는 다음과 같습니다.

- `Reporter` 주소를 제어하는 개인 키 손상/분실
- 불량 관리자
- 네트워크 다운타임
- 여러 작업 실행과 관련된 인적/자동화 오류

**권장 완화 방법:** 출시 단계의 프로토콜이 미션 크리티컬 매개변수에 대한 제어권을 유지하기를 원한다는 점을 이해하지만 출시 단계에서도 다음을 구현할 것을 강력히 권장합니다.

- 오프체인 프로세스의 지속적인 모니터링
- 다중 서명(multi-sig)을 통한 리포터 자동화

장기적으로 프로토콜은 분산화를 향한 명확한 경로를 고려해야 합니다.

**Casimir:**
인지함. 검증인 잔액을 동기화하는 동안 리포터의 개입을 크게 줄이는 예상되는 EigenLayer 체크포인트 업그레이드를 구현할 계획입니다.

**Cyfrin:** 인지함.

\clearpage
## 낮은 위험 (Low Risk)


### 운영자가 0 Wei를 전송하여 자신의 operatorID 상태를 활성으로 설정할 수 있음

**설명:** 담보를 인출할 때 로직은 담보 잔액이 0인지 확인하고 운영자 ID를 비활성으로 만듭니다.

```solidity
 function withdrawCollateral(uint64 operatorId, uint256 amount) external {
        onlyOperatorOwner(operatorId);

        Operator storage operator = operators[operatorId];
        uint256 availableCollateral = operator.collateralBalance - operator.validatorCount * collateralUnit; //@note can cause underflow here if validator count > 0
        if (availableCollateral < amount) {
            revert InvalidAmount();
        }

        operator.collateralBalance -= amount;
        if (operator.collateralBalance == 0) {
            operator.active = false;
        }

        (bool success,) = msg.sender.call{value: amount}("");
        if (!success) {
            revert TransferFailed();
        }

        emit CollateralWithdrawn(operatorId, amount);
    }

```

그러나 입금하는 동안 입금 금액에 대한 확인이 없습니다. 운영자는 0 Wei를 입금하고 운영자 ID를 활성으로 설정할 수 있습니다. 입금 및 인출 상태가 일치하지 않습니다.

```solidity
     function depositCollateral(uint64 operatorId) external payable {
        onlyOperatorOwner(operatorId);

        Operator storage operator = operators[operatorId];
        if (!operator.registered) {
            operatorIds.push(operatorId);
            operator.registered = true;
            emit OperatorRegistered(operatorId);
        }
        if (!operator.active) {
>            operator.active = true; //@audit -> can make operator active even with 0 wei
        }
        operator.collateralBalance += msg.value;

        emit CollateralDeposited(operatorId, msg.value);
    }**
```

**영향:** 검증인 추가 및 제거 시 일관성 없는 로직.

**권장 완화 방법:** `depositCollateral` 함수에서 담보 금액을 확인하는 것을 고려하십시오.

**Casimir:**
[109cf2a](https://github.com/casimirlabs/casimir-contracts/commit/109cf2af2c6009e4dfa483317f2f186c97ed9da3)에서 수정됨

**Cyfrin:** 확인됨.


### 리포터가 보류 중인 검증인을 재공유(reshare)하려고 하면 서비스 거부가 발생함

**설명:** 검증인은 `PENDING` 또는 `ACTIVE` 상태인 경우 운영자를 재공유할 수 있습니다. `PENDING` 상태의 검증인에 대해 재공유가 실행되면 기존 운영자가 SSV 클러스터에서 제거됩니다. -> 그러나 애초에 그러한 운영자는 등록되지 않았습니다. `depositStake`가 호출될 때 SSV 등록이 발생하지 않기 때문입니다.

```solidity
 function reshareValidator(
        uint32 validatorId,
        uint64[] memory operatorIds,
        uint64 newOperatorId,
        uint64 oldOperatorId,
        bytes memory shares,
        ISSVClusters.Cluster memory cluster,
        ISSVClusters.Cluster memory oldCluster,
        uint256 feeAmount,
        uint256 minTokenAmount,
        bool processed
    ) external {
        onlyReporter();

        Validator storage validator = validators[validatorId];
        if (validator.status != ValidatorStatus.ACTIVE && validator.status != ValidatorStatus.PENDING) {
            revert ValidatorNotActive();
        }

       // ... code

        uint256 ssvAmount = retrieveFees(feeAmount, minTokenAmount, address(ssvToken), processed);
        ssvToken.approve(address(ssvClusters), ssvAmount);
>        ssvClusters.removeValidator(validator.publicKey, validator.operatorIds, oldCluster); //@audit validtor key is not registered when the validator is in pending state
       ssvClusters.registerValidator(validator.publicKey, operatorIds, shares, ssvAmount, cluster); //@audit new operators registered

        validator.operatorIds = operatorIds;
        validator.reshares++;

        registry.removeOperatorValidator(oldOperatorId, validatorId, 0);
        registry.addOperatorValidator(newOperatorId, validatorId);

        emit ValidatorReshared(validatorId);
    }
```

`SSVCluster::removeValidator`는 제거할 검증인 데이터를 찾을 수 없을 때 되돌립니다(revert).

```solidity
    function removeValidator(
        bytes calldata publicKey,
        uint64[] memory operatorIds,
        Cluster memory cluster
    ) external override {
        StorageData storage s = SSVStorage.load();

        bytes32 hashedCluster = cluster.validateHashedCluster(msg.sender, operatorIds, s);
        bytes32 hashedOperatorIds = ValidatorLib.hashOperatorIds(operatorIds);

        bytes32 hashedValidator = keccak256(abi.encodePacked(publicKey, msg.sender));
        bytes32 validatorData = s.validatorPKs[hashedValidator];

        if (validatorData == bytes32(0)) {
>            revert ISSVNetworkCore.ValidatorDoesNotExist(); //@audit reverts when no key exists
        }

// ...
```

**영향:** 초기 입금 후 비활성화를 요청하는 운영자는 재공유될 수 없습니다.

**권장 완화 방법:** 다음 2가지 옵션 중 하나를 고려하십시오.
- PENDING 단계에서의 재공유가 지원되어야 하는 경우 `depositStake`에서 운영자를 등록하십시오.
- PENDING 단계에서의 재공유가 지원되지 않아야 하는 경우 `reshareValidator`에서 `Status.PENDING`인 검증인의 재공유를 허용하지 마십시오.

**Casimir:**
[cd03c74](https://github.com/casimirlabs/casimir-contracts/commit/cd03c740c457264e578945bb2a8cc8bcf2c875f8)에서 수정됨

**Cyfrin:** 확인됨.


### CasimirManager에 EigenPod `withdrawNonBeaconChainETHBalanceWei`에 대한 구현 누락

**설명:** EigenLayer에는 파드 소유자가 호출하여 EigenPod에 기부된 ETH를 쓸어담도록(sweep) 의도된 `EigenPod::withdrawNonBeaconChainETHBalanceWei` 함수가 있습니다. 현재 EigenPod에서 이 잔액을 인출할 방법이 없는 것 같습니다.

**영향:** EigenPod에 대한 기부금은 파드가 활성 상태인 동안 본질적으로 갇혀 있습니다.

**권장 완화 방법:** `CasimirManager::claimTips`와 유사하게 `nonBeaconChainETH` 잔액을 쓸어담고 `distributeStakes`로 보내는 함수를 `CasimirManager`에 추가하는 것을 고려하십시오.

**Casimir:**
[790817a](https://github.com/casimirlabs/casimir-contracts/commit/790817a9ba615dbcd7c85d449fe7aa19c02371b7)에서 수정됨

**Cyfrin:** 확인됨.


### 처리할 인출이 없는 경우 `withdrawRewards()` 함수로 인해 `delayedRewards`가 부정확해질 수 있음

**설명:** `CasimirManager`에서 리포터는 `withdrawRewards()` 함수를 사용하여 휩쓸린(swept) 검증인 보상을 처리할 수 있습니다. 리포터는 `WithdrawalProofs`를 제공해야 하며 함수는 이를 사용하여 `eigenPod.verifyAndProcessWithdrawals()`를 호출합니다.

```solidity
function withdrawRewards(WithdrawalProofs memory proofs) external {
    onlyReporter();

    eigenPod.verifyAndProcessWithdrawals(
        proofs.oracleTimestamp,
        proofs.stateRootProof,
        proofs.withdrawalProofs,
        proofs.validatorFieldsProofs,
        proofs.validatorFields,
        proofs.withdrawalFields
    );

    // @audit Not check if the delayed withdrawal length has changed or not
    uint256 delayedAmount = eigenWithdrawals.userDelayedWithdrawalByIndex(
        address(this), eigenWithdrawals.userWithdrawalsLength(address(this)) - 1
    ).amount;
    delayedRewards += delayedAmount;

    emit RewardsDelayed(delayedAmount);
}
```

`verifyAndProcessWithdrawals()` 함수는 인출 목록을 처리하고 하나의 인출로 지연된 인출 라우터에 보냅니다. 그러나 보낼 금액의 합계가 0이 아닌 경우에만 지연된 라우터에 새 인출을 생성합니다.

```solidity
if (withdrawalSummary.amountToSendGwei != 0) {
    _sendETH_AsDelayedWithdrawal(podOwner, withdrawalSummary.amountToSendGwei * GWEI_TO_WEI);
}
```

따라서 리포터가 인출 없이 `withdrawRewards()`를 호출하는 경우, 즉 빈 `withdrawalFields` 및 `validatorFields`를 사용하는 경우 지연된 인출 라우터는 새 항목을 생성하지 않습니다. 그러나 `withdrawRewards()`는 항상 지연된 인출 라우터의 최신 항목을 `delayedAmount`로 가져오므로 실제로는 이미 회계 처리된 오래된 금액을 검색합니다.

**영향:** 리포터가 실수로 인출 없이 `withdrawRewards()`를 호출하면 `delayedRewards`는 이전 지연 금액을 다시 계산하여 잘못된 회계로 이어집니다.

**권장 완화 방법:** `withdrawValidator()` 함수의 패턴을 따르는 것을 고려하십시오. `delayedRewards`에 금액을 추가하기 전에 `eigenWithdrawals.userWithdrawalsLength()` 길이가 변경되는지 확인합니다.

```solidity
uint256 initialDelayedRewardsLength = eigenWithdrawals.userWithdrawalsLength(address(this));
uint64 initialDelayedEffectiveBalanceGwei = eigenPod.withdrawableRestakedExecutionLayerGwei();

eigenPod.verifyAndProcessWithdrawals(
    ...
);

{
    uint256 updatedDelayedRewardsLength = eigenWithdrawals.userWithdrawalsLength(address(this));
    if (updatedDelayedRewardsLength > initialDelayedRewardsLength) {
        IDelayedWithdrawalRouter.DelayedWithdrawal memory withdrawal =
            eigenWithdrawals.userDelayedWithdrawalByIndex(address(this), updatedDelayedRewardsLength - 1);
        if (withdrawal.blockCreated == block.number) {
            delayedRewards += withdrawal.amount;

            emit RewardsDelayed(withdrawal.amount);
        }
    }
}
```

**Casimir:**
[81cb7f1](https://github.com/casimirlabs/casimir-contracts/commit/81cb7f19aaa0dfad5101bcfa8a233fe0fade9365)에서 수정됨

**Cyfrin:** 확인됨.


### 잘못된 테스트 설정으로 인해 잘못된 테스트 결과 발생

**설명:** `IntegrationTest.t.sol`에는 전체 스테이킹 수명 주기를 확인하는 통합 테스트가 포함되어 있습니다. 그러나 현재 테스트 설정은 여러 곳에서 파운드리(foundry)의 `vm.roll`을 사용하여 블록을 진행하지만 `vm.warp`를 사용하여 타임스탬프를 조정하는 것을 무시합니다.

이를 통해 테스트 설정은 시간 지연 없이 보상을 청구할 수 있습니다.

_IntegrationTest.t.sol Line 151_
```solidity
>       vm.roll(block.number + eigenWithdrawals.withdrawalDelayBlocks() + 1); //@audit changing block without changing timestamp
        vm.prank(reporterAddress);
>       manager.claimRewards(); //@audit claiming at the same timestamp

        // Reporter runs after the heartbeat duration
        vm.warp(block.timestamp + 24 hours);
        timeMachine.setProofGenStartTime(0.5 hours);
        beaconChain.setNextTimestamp(timeMachine.proofGenStartTime());
        vm.startPrank(reporterAddress);
        manager.startReport();
        manager.syncValidators(abi.encode(beaconChain.getActiveBalanceSum(), 0));
        manager.finalizeReport();
        vm.stopPrank();
```

타임스탬프를 업데이트하지 않고 블록을 이동하는 것은 블록체인의 비현실적인 시뮬레이션입니다. EigenLayer M2 현재 `WithdrawDelayBlocks`는 50400이며 이는 약 7일입니다. 타임스탬프를 변경하지 않고 50400 블록을 진행하면 테스트는 지연된 보상과 관련된 여러 회계 엣지 케이스를 간과합니다. 각 보고 기간이 24시간 동안 지속되므로 특히 그렇습니다. 즉, 보류 중인 보상을 실제로 청구하려면 7회의 보고 기간이 있어야 합니다.

**영향:** 잘못된 설정은 모든 엣지 케이스가 커버된다는 잘못된 확신을 프로토콜에 제공할 수 있습니다.

**권장 완화 방법:** 테스트 설정을 다음과 같이 수정하는 것을 고려하십시오.

- 보상을 즉시 청구하지 않고 보고서를 실행합니다. 이는 실제 블록체인의 이벤트를 정확하게 반영합니다.
- 블록이 진행될 때마다 시간을 조정하는 것을 고려하십시오.

**Casimir:**
[290d8e1](https://github.com/casimirlabs/casimir-contracts/commit/290d8e11846c5d20ed6a059e32864c8227fb582d)에서 수정됨

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### CasimirRegistry 초기화 시 누락된 검증

**설명:** Beacon Proxy 배포 중에 CasimirRegistry는 0 클러스터 크기 및 0 담보로 배포될 수 있습니다.

**권장 완화 방법:** 클러스터 크기 및 담보에 대한 입력 검증을 고려하십시오.

**Casimir:**
[37f3d34](https://github.com/casimirlabs/casimir-contracts/commit/37f3d34a9102478a85f6791774d86488bf5eb08e)에서 완화됨.

**Cyfrin:** 확인됨.


### CasimirFactory 및 CasimirRegistry에서 ReentrancyGuardUpgradeable이 사용되지 않음

**설명:** CasimirRegistry 및 CasimirFactory는 ReentrancyGuardUpgradeable을 상속하지만 nonRentrant 수정자는 두 계약 모두에서 사용되지 않습니다.

**권장 완화 방법:** `CasimirRegistry` 및 `CasimirFactory`에서 `ReentrancyGuardUpgradeable` 상속을 제거하는 것을 고려하십시오.

**Casimir**
[e403b8b](https://github.com/casimirlabs/casimir-contracts/commit/e403b8b86edbb96e02fd3f5e02e7f207890e1257)에서 수정됨

**Cyfrin**
확인됨.


### `getNextUnstake()`의 기간 확인이 항상 true를 반환함

**설명:** `getNextUnstake()` 함수는 대기열에서 다음 언스테이크 요청을 가져오는 동시에 요청이 이행 가능한지 확인하는 데 사용됩니다. 요청을 이행 가능하게 만드는 조건 중 하나는 `unstake.period <= reportPeriod`입니다.

```solidity
function getNextUnstake() public view returns (Unstake memory unstake, bool fulfillable) {
    if (unstakeQueue.length > 0) {
        unstake = unstakeQueue[0];
        fulfillable = unstake.period <= reportPeriod && unstake.amount <= getWithdrawableBalance();
    }
}
```

그러나 현재 코드베이스를 고려할 때 `unstake.period`는 항상 `reportPeriod`보다 작거나 같습니다. 이는 언스테이크 요청이 생성/대기열에 추가될 때 `unstake.period`에 `reportPeriod`가 할당되지만 `reportPeriod` 값은 시간이 지남에 따라 증가하기 때문입니다.

```solidity
unstakeQueue.push(Unstake({userAddress: msg.sender, amount: amount, period: reportPeriod}));
...
reportPeriod++;
```

**영향:** `unstake.period <= reportPeriod` 확인은 항상 true를 반환하므로 효과가 없습니다.

**권장 완화 방법:** `getNextUnstake()` 함수의 로직을 검토하고 불필요한 경우 기간 확인을 제거하는 것을 고려하십시오.

**Casimir:**
[28baa81](https://github.com/casimirlabs/casimir-contracts/commit/28baa8191a1b5a27d3ee495dee0d993177bf7e5f)에서 완화됨.

**Cyfrin:** 확인됨.


### 사용되지 않는 함수 `validateWithdrawalCredentials()`

**설명:** CasimirManager의 `validateWithdrawalCredentials()` 함수는 비공개(private)이며 계약 어디에서도 호출되지 않습니다.

```solidity
function validateWithdrawalCredentials(address withdrawalAddress, bytes memory withdrawalCredentials) // @audit never used
    private
    pure
{
    bytes memory computedWithdrawalCredentials = abi.encodePacked(bytes1(uint8(1)), bytes11(0), withdrawalAddress);
    if (keccak256(computedWithdrawalCredentials) != keccak256(withdrawalCredentials)) {
        revert InvalidWithdrawalCredentials();
    }
}
```

**권장 완화 방법:** 사용되지 않는 함수를 제거하는 것을 고려하십시오.

**Casimir:**
[d6bd8da](https://github.com/casimirlabs/casimir-contracts/commit/d6bd8dae6e8e2f927f34abc3fdb2db15899ae71b)에서 수정됨.

**Cyfrin:** 확인됨.


### 비 스테이커 계정에 대한 `getUserStake` 함수 실패

**설명:** 스마트 계약의 `getUserStake` 함수는 스테이크가 없는 주소에 대해 호출될 때 오류를 발생시킵니다. 구체적으로 함수는 0으로 나누기 오류로 인해 실패합니다. 이는 `userAddress`가 스테이킹한 적이 없는 경우 제수 `users[userAddress].rewardStakeRatioSum0`이 0일 수 있어 `Math.mulDiv` 연산에서 처리되지 않은 예외가 발생하기 때문입니다.

````solidity
function getUserStake(address userAddress) public view returns (uint256 userStake) {
    userStake = Math.mulDiv(users[userAddress].stake0, rewardStakeRatioSum, users[userAddress].rewardStakeRatioSum0);
}
````

**권장 완화 방법:** 나눗셈을 수행하기 전에 0 제수에 대한 확인을 포함하도록 `getUserStake` 함수를 수정하는 것을 고려하십시오. `users[userAddress].rewardStakeRatioSum0`이 0이면 함수는 0으로 나누기 오류를 피하기 위해 0의 스테이크를 반환해야 합니다.

**Casimir:**
[27c09f5](https://github.com/casimirlabs/casimir-contracts/commit/27c09f548d6d73222a087f2ef237335353cdfefa)에서 수정됨

**Cyfrin:** 확인됨.

\clearpage

