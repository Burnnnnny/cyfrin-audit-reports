**수석 감사 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)


**보조 감사 (Assisting Auditors)**



---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### 악의적인 스테이커가 위임자(delegator)의 `voteWeight` 증가를 원하는 만큼 지연시킬 수 있음

**설명:** 스테이커가 자신의 잔액을 위임자에게 위임하면, 기간 변수인 `_weights[delegate].stakeTime`이 감소합니다. 그러나 위임을 취소할 때는 잔액만 감소합니다. 이를 통해 악의적인 사용자가 `_weights[delegate].stakeTime`을 감소시킬 수 있습니다. 그는 위임과 위임 취소를 반복하여 이 공격의 영향을 확대할 수 있습니다.

**영향:** 악의적인 사용자는 위임자의 `voteWeight` 증가를 원하는 만큼 지연시킬 수 있습니다.

**개념 증명 (Proof of Concept):** 시나리오 예시는 다음과 같습니다:

`HalfTime`이 `7`일이라고 가정합니다.
1. Alice가 `200`을 Bob(피해자)에게 위임합니다.
2. `14`일이 지났습니다. `t`는 `14`입니다.
                       Bob의 `votePower`는 `200*14/(14+7)=133`입니다.
4. Charlie(악의적인 사용자)가 `100`을 Bob에게 위임합니다. `t`는 `14`에서 `200*14*7/(300*(14+7)-200*14)=5.6`으로 감소합니다.
                       Bob의 `votePower`는 `300*5.6/(5.6+7)=133`입니다.
5. Charlie가 Bob에게서 `100`의 위임을 취소합니다. `t`는 여전히 `5.6`입니다. 하지만 `balance`는 `300`에서 `200`으로 감소합니다.
                       Bob의 `votePower`는 `200*5.6/(5.6+7)=88`입니다.

악의적인 사용자는 한 트랜잭션에서 3단계와 4단계를 반복하여 `t`를 원하는 만큼 줄일 수 있습니다.
이런 식으로 그는 `votePower`의 증가를 원하는 만큼 지연시킬 수 있습니다. 최악의 경우, 사용자의 `votePower`를 거의 0으로 영원히 제한하는 것이 가능합니다.

참고: 위임(3단계)과 위임 취소(4단계)를 결합하는 두 가지 방법이 가능합니다.
하나는 `setUserVoteDelegate()`와 `unsetUserVoteDelegate()`의 조합입니다.
다른 하나는 `stake()`와 `withdraw()`의 조합입니다.

**권장 완화 방법:** 위임과 위임 취소 사이의 스테이킹 시간을 조정하는 메커니즘을 개선해야 합니다.

**TempleDAO:** [PR 1041](https://github.com/TempleDAO/temple/pull/1041)에서 수정됨

**Cyfrin:** 확인됨


### 악의적인 사용자가 자신의 투표권을 두 배로 늘릴 수 있음

**설명:** 피위임자(delegatee)의 `voteWeight`를 계산할 때, 자신의 `ownVoteWeight`가 추가됩니다. 이를 통해 일부 사용자는 순환을 구성하여 투표권을 두 배로 늘릴 수 있습니다. 예를 들어, Alice가 Bob에게 위임하고, Bob이 Alice에게 위임하는 경우입니다.

```solidity

    function getDelegatedVoteWeight(address _delegate) external view returns (uint256) {
        // check address is delegate
        uint256 ownVoteWeight = _voteWeight(_delegate, _balances[_delegate]);
        address callerDelegate = userDelegates[_delegate];
        if (!delegates[_delegate] || callerDelegate == msg.sender) { return ownVoteWeight; }
@>      return ownVoteWeight + _voteWeight(_delegate, _delegateBalances[_delegate]);
    }
```

**영향:** 악의적인 사용자가 자신의 투표권을 두 배로 늘릴 수 있습니다.

**개념 증명 (Proof of Concept):** 시나리오 예시는 다음과 같습니다:
    1. Alice가 자신의 투표 가중치 100을 Bob에게 위임했습니다.
    2. Bob이 자신의 투표 가중치 200을 Alice에게 위임했습니다.
    3. Alice와 Bob의 `_delegateBalances`의 총합은 (100 + 200) + (200 + 100) = 600입니다.

**권장 완화 방법:** 문제를 완화하는 몇 가지 방법이 있습니다.

- 첫 번째 방법

자신에게 위임하는 것을 허용하지 않아야 합니다.

```diff

-   function unsetUserVoteDelegate() external override {
+   function unsetUserVoteDelegate() public override {

    function setSelfAsDelegate(bool _approve) external override {
            [...]
+       if (_approve && !prevStatusTrue) unsetUserVoteDelegate();
    }
```

또한, 어떤 피위임자도 위임하는 것을 허용해서는 안 됩니다.

```diff

    function setUserVoteDelegate(address _delegate) external override {
        if (_delegate == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        if (!delegates[_delegate]) {  revert InvalidDelegate(); }
+       if (userDelegates[msg.sender] == msg.sender) { revert(); }
            [...]
    }

```

- 두 번째 방법

사용자가 자신에게 위임하지 않는 한 `ownVoteWeight`를 추가해서는 안 됩니다. 이것은 투표권을 계산하는 일반적인 방법입니다.

```diff

    function getDelegatedVoteWeight(address _delegate) external view returns (uint256) {
        // check address is delegate
-       uint256 ownVoteWeight = _voteWeight(_delegate, _balances[_delegate]);
-       address callerDelegate = userDelegates[_delegate];
-       if (!delegates[_delegate] || callerDelegate == msg.sender) { return ownVoteWeight; }
-       return ownVoteWeight + _voteWeight(_delegate, _delegateBalances[_delegate]);
        return _voteWeight(_delegate, _delegateBalances[_delegate]);
    }

```

**TempleDAO:** [PR 1040](https://github.com/TempleDAO/temple/pull/1040)에서 수정됨

**Cyfrin:** 확인됨


### 위임자가 자기 위임(self-delegation)을 재설정하면 프로토콜에 여러 문제가 발생함

**설명:** 위임자의 투표권이 변경될 때마다 위임자의 자기 위임 유효성이 확인되지 않아 프로토콜의 여러 부분에서 문제가 발생합니다.

- `unsetUserVoteDelegate`: 0에서 위임된 잔액을 빼려고 시도하기 때문에 스테이커가 함수를 통해 위임을 해제할 수 없습니다.
- `_withdrawFor`: 0에서 위임된 잔액을 빼려고 시도하기 때문에 스테이커가 자산을 인출할 수 없습니다.
- `_stakeFor`: 악의적인 스테이커가 스테이킹 & 위임자 수정 & 인출 과정을 반복하여 무한한 투표권을 얻을 수 있습니다.

**영향:** 공격자가 `_stakeFor`의 문제를 반복하여 투표권을 축적할 수 있으며, 인출 시 되돌리기(revert)가 발생합니다.

**개념 증명 (Proof of Concept):** 악의적인 공격자가 다음 과정을 반복하여 무한한 투표권을 얻을 수 있는 시나리오는 다음과 같습니다:

1. 스테이커가 Bob에게 위임합니다.
2. Bob이 자기 위임을 재설정합니다.
3. 스테이커가 토큰을 스테이킹합니다.
4. 스테이커가 위임을 다른 사람으로 변경합니다.

**권장 완화 방법:** 위의 3개 함수에서 함수 호출 시 위임자가 자기 위임을 활성화했는지 확인해야 합니다.

**TempleDAO:** [PR 1034](https://github.com/TempleDAO/temple/pull/1034)에서 수정됨

**Cyfrin:** 확인됨

\clearpage
## 중간 위험 (Medium Risk)


### TGLD 발행 설정 오류로 인해 최대 공급량이 너무 빨리 발행됨

**설명:** TempleGold 계약을 통해 TGLD를 발행할 때 발행량은 최소 금액인 10,000 TGLD보다 커야 합니다. 또한 발행이 처음 발생하는 경우 발행량은 항상 SUPPLY * n/d이며, 이는 1초 동안의 양입니다.

위의 계산에 따르면 초당 발행량은 10,000 TGLD보다 커야 하며, 이는 최대 공급량(1e9 TGLD)이 1e5초(약 1일 4시간) 만에 발행됨을 의미합니다.

**영향:** 최대 공급량 토큰이 1일 4시간 만에 발행되거나, 첫 번째 발행에 대해 토큰을 발행할 수 없습니다.

**권장 완화 방법:** TGLD 계약이 생성되거나 `setVestingFactor`가 처음 호출될 때 `_lastMintTimestamp`를 초기화해야 합니다.

**TempleDAO:** [PR 1026](https://github.com/TempleDAO/temple/pull/1026)에서 수정됨

**Cyfrin:** 확인됨


### DaiGoldAction 계약에서 recoverToken이 호출될 때 상태가 업데이트되지 않음

**설명:** `recoverToken` 함수는 경매 계약에서 특정 양의 TGLD를 다른 곳으로 이동하고 마지막 에포크(epoch)를 삭제하여 재설정 및 재시작할 수 있도록 합니다.
토큰은 이동되지만 `nextAuctionGoldAmount` 상태는 차감되지 않으며, 이로 인해 다음 에포크에 잘못된 양의 TGLD 토큰이 제공되어 결과적으로 입찰자가 TGLD를 청구할 수 없게 됩니다.

**영향:** 다음 경매의 경매 금액이 잘못되어 경매 종료 후 사용자가 토큰을 청구할 수 없게 됩니다.

**권장 완화 방법:** `recoverToken` 함수에서 토큰이 이동될 때 `nextAuctionGoldAmount`를 차감해야 합니다.

**TempleDAO:** [PR 1027](https://github.com/TempleDAO/temple/pull/1027)에서 수정됨

**Cyfrin:** 확인됨


### 스테이킹 계약에서 distributeGold 함수를 선행 매매(front-running)하여 보상 분배를 지연시킬 수 있음

**설명:** `TempleGoldStaking` 계약에서 보상 분배자는 토큰 계약에서 사용 가능한 TGLD를 발행하여 `distributeRewards`를 호출하고 발행된 양에 따라 보상률을 업데이트합니다.
그러나 분배는 분배 쿨다운 기간이 지난 후에만 작동합니다:
```Solidity
if (lastRewardNotificationTimestamp + rewardDistributionCoolDown > block.timestamp)
    { revert CannotDistribute(); }
```

최신 업데이트 타임스탬프로 `lastRewardNotificationTimestamp`를 사용하며, 이는 공개 함수인 `distributeGold`를 호출하는 누구나 업데이트할 수 있습니다.
이 함수를 선행 매매함으로써 분배자가 보상률을 업데이트하는 것을 방지합니다.

더 심각한 것은 분배 쿨다운 전에 `distributeGold`가 정기적으로 호출되면 보상 분배가 영원히 방지될 수 있다는 것입니다.

**영향:** 보상 분배가 지연될 수 있습니다.

**권장 완화 방법:** `distributeGold`를 허가된(permissioned) 상태로 만들거나 `distributeRewards` 함수에서만 업데이트되는 보상 분배의 최신 타임스탬프를 나타내는 다른 상태 변수를 도입하십시오.

**TempleDAO:** [PR 1028](https://github.com/TempleDAO/temple/pull/1028)에서 수정됨

**Cyfrin:** 확인됨


### compose 옵션을 구성하면 사용자가 대상 체인에서 TGLD를 받지 못하게 됨

**설명:** 사용자는 TempleGold 계약의 `send` 함수를 호출하여 TGLD를 보낼 수 있습니다.
`SendParam` 구조체가 사용자로부터 전달되므로 compose 데이터를 포함한 모든 필드를 메시지에 추가할 수 있습니다.
그러나 대상 체인의 `_lzReceive` 함수에서 compose 옵션이 있는 메시지를 차단합니다:
```Solidity
if (_message.isComposed()) { revert CannotCompose(); }
```

즉, compose 옵션과 함께 `send`를 호출하는 사용자는 대상 체인에서 TGLD 토큰을 받지 못합니다.

**영향:** 매개변수를 기반으로 토큰 전송이 작동하지 않아 사용자의 자금이 손실됩니다.

**권장 완화 방법:**
1. `send` 함수에서 `_sendParam`에 compose 옵션이 포함된 경우 되돌려야(revert) 합니다.
2. 더 좋은 방법은 `send` 함수가 사용자로부터 보낼 금액과 같은 필수 필드만 입력받고 사용자로부터 전체를 받는 대신 내부에서 `SendParam`을 구성하는 것입니다.

**TempleDAO:** [PR 1029](https://github.com/TempleDAO/temple/pull/1029)에서 수정됨

**Cyfrin:** 확인됨


### 스테이킹 계약에서 자기 위임을 재설정하면 투표 가중치가 0으로 재설정됨

**설명:** `TempleGoldStaking` 계약에서 피위임자가 자기 위임을 재설정하면 `_updateAccountWeight`를 호출하여 피위임자에게 위임된 투표 가중치가 제거됩니다.
새 잔액 매개변수로 0이 전달되므로 투표 가중치의 `stakingTime`이 0으로 재설정됩니다:
```solidity
// TempleGoldStaking.sol:L580
if (_newBalance == 0) {
    t = 0;
    _lastShares = 0;
}
```

이는 피위임자가 자신의 잔액을 가지고 있어도 투표 가중치가 0부터 누적되기 시작함을 의미하며, 이는 감소되어서는 안 됩니다.

**영향:** 피위임자는 투표권을 잃게 되며, 에포크 종료 직전에 자기 위임 재설정이 발생하면 0이 될 수도 있습니다.

**권장 완화 방법:** 자기 위임을 재설정할 때 `stakingTime`을 줄이지 않아야 피위임자의 투표권이 자신의 잔액에 따라 유지됩니다.

**TempleDAO:** [PR 1043](https://github.com/TempleDAO/temple/pull/1043)에서 수정됨

**Cyfrin:** 확인됨


### 스파이스(Spice) 경매의 종료 시간 설정이 잘못됨

**설명:** `SpiceAuction` 계약의 `startAuction` 함수에서 종료 시간이 잘못 설정되었습니다.
```solidity
uint128 startTime = info.startTime = uint128(block.timestamp) + config.startCooldown;
uint128 endTime = info.endTime = uint128(block.timestamp) + config.duration;
```
현재 `endTime`은 위에서 계산된 `startTime`이 아닌 `block.timestamp`를 사용하여 설정됩니다.

**영향:** 경매 종료 시간이 예상보다 짧아집니다.

**권장 완화 방법:** `endTime`은 `startTime`과 `duration`을 사용하여 계산되어야 합니다.
```solidity
uint128 endTime = info.endTime = startTime + config.duration;
```

**TempleDAO:** [PR 1032](https://github.com/TempleDAO/temple/pull/1032)에서 수정됨

**Cyfrin:** 확인됨


### 입찰 없이 경매가 종료될 때 토큰 회수 메커니즘이 없음

**설명:** `SpiceAuction` 계약에서 입찰 없이 경매가 종료되면 경매 토큰이 계약에 잠겨 회수할 수 없습니다. `recoverToken` 함수는 회수 가능한 금액을 `balance - totalAllocation`으로 제한하므로 경매 토큰을 회수하는 데 도움이 되지 않습니다.

**영향:** 경매 토큰이 계약에 갇혀 회수할 수 없게 될 수 있지만, 이는 드문 경우일 것입니다.

**권장 완화 방법:** 입찰 없이 종료될 때 경매에서 토큰을 회수하는 메커니즘을 구현해야 합니다.

**TempleDAO:** [PR 1033](https://github.com/TempleDAO/temple/pull/1033)에서 수정됨

**Cyfrin:** 확인됨


### `TempleGoldStaking.getVoteWeight()`가 계정의 투표 가중치를 잘못 계산함

**설명:** `TempleGoldStaking.sol` 계약에서 함수는 계정의 투표권을 반환할 것으로 예상되지만 때때로 잘못 계산됩니다.
L379에서 계정의 현재 잔액을 사용합니다.
```solidity
File: contracts\templegold\TempleGoldStaking.sol
378:     function getVoteWeight(address account) external view returns (uint256) {
379:         return _voteWeight(account, _balances[account]);
380:     }
```
그러나 `_voteWeight()` 함수에서는 L556의 `week > currentWeek`일 때 이전 투표 가중치를 사용합니다.
그런 다음 L563에서 현재 잔액과 이전 투표 가중치를 사용하여 투표권을 계산합니다.
```solidity
File: contracts\templegold\TempleGoldStaking.sol
549:     function _voteWeight(address _account, uint256 _balance) private view returns (uint256) {
550:         /// @dev Vote weights are always evaluated at the end of last week
551:         /// Borrowed from st-yETH staking contract (https://etherscan.io/address/0x583019fF0f430721aDa9cfb4fac8F06cA104d0B4#code)
552:         uint256 currentWeek = (block.timestamp / WEEK_LENGTH) - 1;
553:         AccountWeightParams storage weight = _weights[_account];
554:         uint256 week = uint256(weight.weekNumber);
555:         if (week > currentWeek) {
556:             weight = _prevWeights[_account];
557:         }
558:         uint256 t = weight.stakeTime;
559:         uint256 updated = weight.updateTime;
560:         if (week > 0) {
561:             t += (block.timestamp / WEEK_LENGTH * WEEK_LENGTH) - updated;
562:         }
563:         return _balance * t / (t + halfTime);
564:     }
```
결과적으로 투표권을 잘못 계산합니다.

**영향:** TempleGoldStaking.getVoteWeight()가 계정의 투표 가중치를 잘못 계산합니다.

**개념 증명 (Proof of Concept):** `halfTime`이 7일이라고 가정해 봅시다.
- Alice가 5월 3일에 `100 ether`를 스테이킹합니다.
- Alice가 5월 17일에 `400 ether`를 다시 스테이킹합니다.
- 그녀는 5월 18일에 `getVoteWeight()` 함수를 호출합니다.
  함수는 실제 투표권인 `100 * 13 / (13 + 7)` 대신 `500 * 13 / (13 + 7)`을 반환하며, 이는 5배 더 큽니다.

**권장 완화 방법:** 함수에서 이전 투표 가중치가 사용될 때 사용자의 이전 잔액을 사용하는 것이 좋습니다.

**TempleDAO:** [PR 1043](https://github.com/TempleDAO/temple/pull/1043)에서 수정됨

**Cyfrin:** 확인됨


### 스테이커가 없어도 보상이 분배된 것으로 계산되어 보상이 영원히 잠김

**설명:** `TempleGoldStaking.sol` 계약은 Synthetix 보상 분배 계약의 포크이며 약간의 수정이 있습니다. 코드는 `_totalSupply`가 0일 때 누적 비율을 업데이트하지 않음으로써 사용자가 없는 시나리오를 특별히 처리하지만, L476의 타임스탬프 추적에 대해서는 그러한 조건을 포함하지 않습니다.
```solidity
File: contracts\templegold\TempleGoldStaking.sol
475:     function _rewardPerToken() internal view returns (uint256) {
476:         if (totalSupply == 0) {
477:             return rewardData.rewardPerTokenStored;
478:         }
479:
480:         return
481:             rewardData.rewardPerTokenStored +
482:             (((_lastTimeRewardApplicable(rewardData.periodFinish) -
483:                 rewardData.lastUpdateTime) *
484:                 rewardData.rewardRate * 1e18)
485:                 / totalSupply);
486:     }
```
이 때문에 스테이킹하는 사용자가 없을 때에도 회계 로직은 해당 기간 동안 자금이 분산되었다고 생각합니다(시작 타임스탬프가 업데이트되기 때문).

결과적으로 스테이킹하는 사용자가 있기 전에 `distributeRewards()` 함수가 호출되면 첫 번째 스테이커에게 갔어야 할 자금은 대신 누구에게도 발생하지 않고 계약에 영원히 잠기게 됩니다.

**영향:** 분배되지 않은 보상이 계약에 갇힙니다.

**개념 증명 (Proof of Concept):** 시나리오 예시는 다음과 같습니다:
Alice는 `distributionStarter`이고 Bob은 `Temple`을 스테이킹하려는 사람입니다.
- Alice가 `distributeRewards()` 함수를 호출하여 이 계약에 대한 TGLD를 발행합니다.
  간단히 계산하기 위해 발행된 TGLD가 `7*86400 ether`라고 가정해 봅시다. 그러면 `rewardRate`는 `1 ether`가 됩니다.
- 24시간 후, Bob이 계약에 `10000` TGLD를 스테이킹합니다.
- 6일 후, Bob이 스테이킹된 모든 TGLD를 인출하고 보상을 청구합니다. 그러면 그는 `6*86400 ether`를 얻습니다.

결과적으로 `86400 ether`가 계약에 잠깁니다.

**권장 완화 방법:** `distributeRewards()` 함수에서 계약에 충분한 보상 토큰이 이미 있는지 확인하십시오.
```diff
File: contracts\templegold\TempleGoldStaking.sol
245:     function distributeRewards() updateReward(address(0)) external {
246:         if (distributionStarter != address(0) && msg.sender != distributionStarter)
247:             { revert CommonEventsAndErrors.InvalidAccess(); }
248:         // Mint and distribute TGLD if no cooldown set
249:         if (lastRewardNotificationTimestamp + rewardDistributionCoolDown > block.timestamp)
250:                 { revert CannotDistribute(); }
251:         _distributeGold();
252:         uint256 rewardAmount = nextRewardAmount;
253:         if (rewardAmount == 0 ) { revert CommonEventsAndErrors.ExpectedNonZero(); }
254:         nextRewardAmount = 0;
255:         _notifyReward(rewardAmount);
256:     }
```
```diff
File: contracts\common\CommonEventsAndErrors.sol
06: library CommonEventsAndErrors {
07:     error InsufficientBalance(address token, uint256 required, uint256 balance);
08:     error InvalidParam();
09:     error InvalidAddress();
10:     error InvalidAccess();
11:     error InvalidAmount(address token, uint256 amount);
12:     error ExpectedNonZero();
13:     error Unimplemented();
14:     event TokenRecovered(address indexed to, address indexed token, uint256 amount);
15: }
```
(Note: The recommended mitigation in the original text had a `+` line `if (totalSupply == 0) { revert CommonEventsAndErrors.NoStaker(); }` and added `NoStaker()` error. I'll translate based on the diff provided in the original text.)

**TempleDAO:** [PR 1037](https://github.com/TempleDAO/temple/pull/1037)에서 수정됨

**Cyfrin:** 확인됨


### 투표 위임자를 자신에서 다른 사용자로 변경할 때 `_delegateBalances`가 감소해서는 안 됨

**설명:** `setUserVoteDelegate` 함수에서 사용자가 투표 위임자를 자신으로 설정할 때 L149에서 `_delegateBalances` 값에 그의 `userBalance`가 포함되지 않습니다.
그가 투표 위임자를 자신에서 다른 사용자로 변경할 때 L145에서 `_delegateBalances`에서 `userBalance`를 뺍니다. 하지만 `userBalance`를 빼서는 안 됩니다.

```Solidity
File: contracts\templegold\TempleGoldStaking.sol
136:         if (userBalance > 0) {
137:             uint256 _prevBalance;
138:             uint256 _newDelegateBalance;
139:             if (removed) {
140:                 // update vote weight of old delegate
141:                 _prevBalance = _delegateBalances[_userDelegate];
142:                 /// @dev skip for a previous delegate with 0 users delegated and 0 own vote weight
143:                 if (_prevBalance > 0) {
144:                     /// @dev `_prevBalance > 0` because when a user sets delegate, vote weight and `_delegateBalance` are updated for delegate
145:                     _newDelegateBalance = _delegateBalances[_userDelegate] = _prevBalance - userBalance;
146:                     _updateAccountWeight(_userDelegate, _prevBalance, _newDelegateBalance, false);
147:                 }
148:             }
149:             if (msg.sender != _delegate) { // @audit msg.sender == _delegate ?
150:                 /// @dev Reuse variables
151:                 _prevBalance = _delegateBalances[_delegate];
152:                 _newDelegateBalance = _delegateBalances[_delegate] = _prevBalance + userBalance;
153:                 _updateAccountWeight(_delegate, _prevBalance, _newDelegateBalance, true);
154:             }
155:         }

```

**영향:** `_delegateBalances`가 올바르게 계산되지 않습니다.

**권장 완화 방법:** 이전에 위임된 사용자가 자신이 아닌지 확인하는 것이 좋습니다.

```diff
File: contracts\templegold\TempleGoldStaking.sol
        if (removed) {
            // update vote weight of old delegate
            _prevBalance = _delegateBalances[_userDelegate];
            /// @dev skip for a previous delegate with 0 users delegated and 0 own vote weight
-           if (_prevBalance > 0) {
+           if (_prevBalance > 0 && _userDelegate != msg.sender) {
                /// @dev `_prevBalance > 0` because when a user sets delegate, vote weight and `_delegateBalance` are updated for delegate
                _newDelegateBalance = _delegateBalances[_userDelegate] = _prevBalance - userBalance;
                _updateAccountWeight(_userDelegate, _prevBalance, _newDelegateBalance, false);
            }
        }
```

**TempleDAO:** [PR 1038](https://github.com/TempleDAO/temple/pull/1038)에서 수정됨

**Cyfrin:** 확인됨

\clearpage
## 낮은 위험 (Low Risk)


### 보상 분배 시 먼지(dust) 양이 남아 계약에 갇힘

**설명:** 보상 분배가 발생할 때 7일(604800초) 동안 분배되며 보상률은 초당 계산되므로 매번 604800 wei 미만의 토큰 나머지가 계약에 남습니다.

**권장 완화 방법:** 남은 보상 금액은 다음 분배 라운드에 추가되어야 합니다.

**Temple DAO:**
[PR 1046](https://github.com/TempleDAO/temple/pull/1046)에서 수정됨

**Cyfrin:** 확인됨


### 골드 토큰의 마지막 발행에 대해 DAI-GOLD 경매가 시작되지 않을 수 있음

**설명:** 일반적으로 골드 발행은 사용 가능한 양이 1e4보다 클 때 발생하며, 이는 DaiGoldAuction의 최소 분배 금액을 구성합니다. 그러나 최대 공급량에 도달하면 발행량이 최소 금액보다 작을 수 있으므로 DaiGoldAuction은 금액이 최소 분배 금액보다 작기 때문에 시작되지 않을 수 있습니다.

**권장 완화 방법:** 최대 공급량에 도달하면 남은 금액으로 경매를 시작할 수 있도록 허용합니다.

**Temple DAO:**
`config.auctionMinimumDistributedGold`가 반드시 최소 골드 발행량과 같을 필요는 없습니다. 또한 마지막 발행(최대 공급량)의 경우 `config.auctionMinimumDistributedGold`가 업데이트됩니다.

**Cyfrin:** 인지함


### 스파이스(Spice) 경매에서 전송 수수료 토큰(Fee-on-transfer tokens)이 지원되지 않음

**설명:** 전송 수수료 토큰은 스파이스 경매 계약에서 지원되지 않으며, 이로 인해 경매 토큰 계산이 잘못될 수 있습니다.

**권장 완화 방법:** 실제로 이동된 입찰 토큰 금액을 계산하려면 이전/이후 잔액 차이를 사용하십시오.

**Temple DAO:**
[PR 1046](https://github.com/TempleDAO/temple/pull/1046)에서 수정됨

**Cyfrin:** 확인됨


### TempleGold의 현재 설계는 토큰 분배를 방지함

**설명:** 현재 TempleGold 토큰 설계를 사용하면 토큰은 화이트리스트에 있는 주소로만/주소에서만 전송될 수 있습니다. 이는 TempleGold 토큰이 DEX에 상장되거나 담보로 대출 프로토콜에 배치되는 등의 추가 분배를 방지하며, 프로토콜이 사용자 요구에 따라 모든 주소를 화이트리스트에 올릴 수 없기 때문에 불가능합니다.

**권장 완화 방법:** 화이트리스트 플래그가 있어야 하며, true로 설정되면 화이트리스트에 있는 발신/수신 주소만 허용되고 false로 설정되면 개방되어야 합니다.

**Temple DAO:**
인지하지만 TempleGold는 거래되거나 대출 플랫폼에서 사용되지 않습니다.

**Cyfrin:** 인지함

\clearpage
## 정보성 (Informational)


### 계약 생성자(constructors)에서 0 주소가 확인되지 않음

**설명:** `DaiGoldAuction`, `TempleGold`, `TempleGoldStaking` 계약의 생성자에서 0 주소가 검증되지 않습니다.


### `TempleGold` 계약의 `send` 함수의 중복 로직

**설명:** `send` 함수는 `OFTCore`의 `send` 복사본이며 대상 주소만 검증됩니다. 이는 `super.send`를 사용하여 업데이트할 수 있습니다.



### 위임 설계가 지나치게 복잡함

**설명:** 스테이킹 계약에서 위임 관련 로직이 지나치게 복잡하며, `delegation`, `self-delegation`, `delegation to self` 개념이 혼합되어 있어 단순화할 수 있습니다.


### TempleGold가 일부 체인과 호환되지 않음

**설명:** 솔리디티 컴파일러 0.8.19 이하 버전에서 `PUSH0`이 지원되지 않기 때문에 TempleGold는 솔리디티 컴파일러 0.8.19 이하만 지원하는 Linea와 같은 체인과 호환되지 않습니다.

\clearpage

