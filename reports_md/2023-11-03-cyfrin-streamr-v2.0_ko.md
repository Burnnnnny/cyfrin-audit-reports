**Lead Auditors**

[Hans](https://twitter.com/hansfriese)

---

# 발견 사항
## 고위험

### `VoteKickPolicy._endVote()`가 언더플로우로 인해 영구적으로 revert될 수 있음

**심각도:** 높음

**설명:** `onFlag()`에서 반올림으로 인해 `targetStakeAtRiskWei[target]`가 플래거/검토자의 총 보상보다 적을 수 있습니다.

```solidity
File: contracts\OperatorTokenomics\StreamrConfig.sol
22:     /**
23:      * Minimum amount to pay reviewers+flagger
24:      * That is: minimumStakeWei >= (flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei) / slashingFraction
25:      */
26:     function minimumStakeWei() public view returns (uint) {
27:         return (flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei) * 1 ether / slashingFraction;
28:     }
```

- `flaggerRewardWei + flagReviewerCount * flagReviewerRewardWei = 100`, `StreamrConfig.slashingFraction = 0.03e18(3%)`, `minimumStakeWei() = 1000 * 1e18 / 0.03e18 = 10000 / 3 = 3333`이라고 가정해 봅시다.
- `stakedWei[target] = streamrConfig.minimumStakeWei()`라고 가정하면, `targetStakeAtRiskWei[target] = 3333 * 0.03e18 / 1e18 = 99.99 = 99`가 됩니다.
- 결과적으로 `targetStakeAtRiskWei[target]`가 총 보상(=100)보다 작아지며, 보상 분배 중 언더플로우로 인해 `_endVote()`가 revert됩니다.

위 시나리오는 `minimumStakeWei` 계산 중 반올림이 발생할 때만 가능합니다. 따라서 기본 `slashingFraction = 10%`에서는 정상적으로 작동합니다.

**파급력:** `VoteKickPolicy`가 예상대로 작동하지 않으며 악의적인 운영자가 영구적으로 추방되지 않을 수 있습니다.

**권장 완화 조치:** `minimumStakeWei()`를 항상 올림(ceil) 처리하십시오.

**클라이언트:** 커밋 [615b531](https://github.com/streamr-dev/network-contracts/commit/615b5311963082c79d05ec4072b1abeba4d1f9b4)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `_payOutFirstInQueue`에서 발생 가능한 오버플로우

**심각도:** 높음

**설명:** `_payOutFirstInQueue()`에서 `operatorTokenToDataInverse()` 호출 중 revert가 발생할 수 있습니다.

```solidity
uint amountOperatorTokens = moduleCall(address(exchangeRatePolicy), abi.encodeWithSelector(exchangeRatePolicy.operatorTokenToDataInverse.selector, amountDataWei));
```

위임자(delegator)가 `type(uint256).max`로 `undelegate()`를 호출하면, uint 오버플로우로 인해 `operatorTokenToDataInverse()`가 revert되고 큐 로직이 영구적으로 손상됩니다.

```solidity
   function operatorTokenToDataInverse(uint dataWei) external view returns (uint operatorTokenWei) {
       return dataWei * this.totalSupply() / valueWithoutEarnings();
   }
```

**파급력:** `_payOutFirstInQueue()`가 계속 revert되어 큐 로직이 영구적으로 손상됩니다.

**권장 완화 조치:** `operatorTokenToDataInverse()`를 호출하기 전에 `amountDataWei`에 상한을 두어야 합니다.

**클라이언트:** 커밋 [c62e5d9](https://github.com/streamr-dev/network-contracts/commit/c62e5d90ce8f8c084fe3917f499c967c85a3873b)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `DefaultUndelegationPolicy.onUndelegate()`의 잘못된 검증

**심각도:** 높음

**설명:** `onUndelegate()`에서 운영자 소유자가 최소 `minimumSelfDelegationFraction` 이상의 총 공급량을 보유하고 있는지 확인합니다.

```solidity
   function onUndelegate(address delegator, uint amount) external {
       // limitation only applies to the operator, others can always undelegate
       if (delegator != owner) { return; }

       uint actualAmount = amount < balanceOf(owner) ? amount : balanceOf(owner); //@audit amount:DATA, balanceOf:Operator
       uint balanceAfter = balanceOf(owner) - actualAmount;
       uint totalSupplyAfter = totalSupply() - actualAmount;
       require(1 ether * balanceAfter >= totalSupplyAfter * streamrConfig.minimumSelfDelegationFraction(), "error_selfDelegationTooLow");
   }
```

하지만 `amount`는 DATA 토큰 양을 의미하고 `balanceOf(owner)`는 `Operator` 토큰 잔액을 나타내므로 이들을 직접 비교하는 것은 불가능합니다.

**파급력:** `onUndelegate()`가 예상치 못하게 작동하여 운영자 소유자가 위임을 해제(undelegate)할 수 없게 됩니다.

**권장 완화 조치:** `onUndelegate()`는 동일한 토큰으로 변환한 후 양을 비교해야 합니다.

**클라이언트:** 커밋 [9b8c65e](https://github.com/streamr-dev/network-contracts/commit/9b8c65ea31b6bf15fe4ec913a975782f27c0c9a0)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 악의적인 타겟이 forceUnstaking/staking을 반복하여 `_endVote()`를 영구적으로 revert시킬 수 있음

**심각도:** 높음

**설명:** `_endVote()`에서 `target`의 스테이킹 상태에 따라 `forfeitedStakeWei` 또는 `lockedStakeWei[target]`를 업데이트합니다.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
179:     function _endVote(address target) internal {
180:         address flagger = flaggerAddress[target];
181:         bool flaggerIsGone = stakedWei[flagger] == 0;
182:         bool targetIsGone = stakedWei[target] == 0;
183:         uint reviewerCount = reviewers[target].length;
184:
185:         // release stake locks before vote resolution so that slashings and kickings during resolution aren't affected
186:         // if either the flagger or the target has forceUnstaked or been kicked, the lockedStakeWei was moved to forfeitedStakeWei
187:         if (flaggerIsGone) {
188:             forfeitedStakeWei -= flagStakeWei[target];
189:         } else {
190:             lockedStakeWei[flagger] -= flagStakeWei[target];
191:         }
192:         if (targetIsGone) {
193:             forfeitedStakeWei -= targetStakeAtRiskWei[target];
194:         } else {
195:             lockedStakeWei[target] -= targetStakeAtRiskWei[target]; //@audit revert after forceUnstake() => stake() again
196:         }
```

타겟이 양수의 스테이킹 금액을 가지고 있다면 여전히 활성 상태인 것으로 간주합니다. 하지만 타겟이 언스테이킹 후 다시 스테이킹했는지는 알 수 없으므로 아래 시나리오가 가능합니다.

- 타겟이 100을 스테이킹하고 플래거가 그를 신고했습니다.
- `onFlag()`에서 `lockedStakeWei[target] = targetStakeAtRiskWei[target] = 100`이 됩니다.
- 투표 기간 동안 타겟이 `forceUnstake()`를 호출했습니다. 그러면 `Sponsorship._removeOperator()`에서 `lockedStakeWei[target]`가 0으로 초기화됩니다.
- 그 후 그가 다시 스테이킹하면 `_endVote()`는 언더플로우로 인해 L195에서 영구적으로 revert됩니다.

결국, 현재의 플래깅이 완료되지 않으므로 그는 다시 플래깅되지 않을 것입니다.

게다가 악의적인 운영자는 위 상태를 스스로 조작하여 위험 없이 운영자 보상을 얻을 수 있습니다.

**파급력:** 악의적인 운영자가 `_endVote()`를 영구적으로 revert시켜 플래깅 시스템을 우회할 수 있습니다.

**권장 완화 조치:** `_endVote()`에서 현재 스테이킹 금액에 의존하지 않고 스테이크 잠금 해제를 수행하십시오.

**클라이언트:** 커밋 [8be1d7e](https://github.com/streamr-dev/network-contracts/commit/8be1d7e3ded2d595eaaa16ddd4474d3a8d31bfbe)에서 수정되었습니다.

**Cyfrin:** 확인됨.

## 중간 위험

### `VoteKickPolicy.onFlag()`에서 `targetStakeAtRiskWei[target]`가 `stakedWei[target]`보다 커서 `_endVote()`가 revert될 수 있음

**심각도:** 중간

**설명:** `onFlag()`에서 `targetStakeAtRiskWei[target]`가 `stakedWei[target]`보다 클 수 있습니다.

```solidity
targetStakeAtRiskWei[target] = max(stakedWei[target], streamrConfig.minimumStakeWei()) * streamrConfig.slashingFraction() / 1 ether;
```

예를 들어:
- 처음에 `streamrConfig.minimumStakeWei() = 100`이고 운영자(=타겟)가 100을 스테이킹했습니다.
- 재설정 후 `streamrConfig.minimumStakeWei()`가 2000으로 증가했습니다.
- 타겟에 대해 `onFlag()`가 호출되면 `targetStakeAtRiskWei[target]`는 `max(100, 2000) * 10% = 200`이 됩니다.
- `_endVote()`에서 타겟은 100만 스테이킹했으므로 `slashingWei = _kick(target, slashingWei)`는 100이 됩니다.
- 따라서 보상 분배 중 언더플로우로 인해 revert됩니다.

**파급력:** 적은 스테이킹 자금을 가진 운영자가 영구적으로 추방되지 않을 수 있습니다.

**권장 완화 조치:** `onFlag()`는 타겟이 보상을 위해 충분한 자금을 스테이킹했는지 확인하고 그렇지 않은 경우 별도로 처리해야 합니다.

**클라이언트:** 커밋 [05d9716](https://github.com/streamr-dev/network-contracts/commit/05d9716b8e19668fea70959327ab8e896ab0645d)에서 수정되었습니다. 충분한 지분(검토 비용 지불을 위한)이 없는 플래그 타겟은 검토 없이 추방됩니다. 이는 관리자가 최소 스테이크 요구 사항을 변경한 후(예: 검토자 보상 증가)에만 발생할 수 있으므로, 플래그 타겟의 잘못이 아니며 슬래싱되지 않습니다. 그들은 원한다면 새로운 최소 스테이크로 다시 스테이킹할 수 있습니다.

**Cyfrin:** 확인됨.

### `flag()`의 프론트 러닝 가능성

**심각도:** 중간

**설명:** `target`은 잠재적인 자금 손실을 피하기 위해 플래거가 `flag()`를 호출하기 전에 `unstake()/forceUnstake()`를 호출할 수 있습니다. 또한 `penaltyPeriodSeconds` 요건을 충족하면 언스테이킹 중 `target`에 대한 슬래싱이 발생하지 않습니다.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
65:     function onFlag(address target, address flagger) external {
66:         require(flagger != target, "error_cannotFlagSelf");
67:         require(voteStartTimestamp[target] == 0 && block.timestamp > protectionEndTimestamp[target], "error_cannotFlagAgain"); // solhint-disable-line not-rely-on-time
68:         require(stakedWei[flagger] >= minimumStakeOf(flagger), "error_notEnoughStake");
69:         require(stakedWei[target] > 0, "error_flagTargetNotStaked"); //@audit possible front run
70:
```

**파급력:** 악의적인 타겟이 프론트 러닝을 통해 킥(kick) 정책을 우회할 수 있습니다.

**권장 완화 조치:** 간단한 완화 방법은 없지만 스테이킹 자금의 일부에 대해 일종의 `지연된 언스테이킹(delayed unstaking)` 로직을 구현할 수 있습니다.

**클라이언트:** 현재 위협 모델은 Streamr 노드를 운영하지 않는 스테이커입니다. 그들은 UI를 통해 모든 스마트 계약 트랜잭션을 수행하기 위해 Metamask를 사용하는 사람이거나 복잡한 플래시봇 MEV 검색자일 수 있습니다. 그러나 그들이 Streamr 노드를 실행하지 않는다면, 정직한 노드들에 의해 발각되고 플래그가 지정되어 추방되어야 합니다.

고급 봇이 스테이킹하고 `Flagged` 이벤트를 수신할 수 있지만, 최소 유지 기간(`DefaultLeavePolicy.penaltyPeriodSeconds`)이 끝나기 전에 발각되어 플래그가 지정되면 플래깅을 프론트 러닝하더라도 스테이크가 슬래싱됩니다. 우리는 Streamr 노드를 실제로 실행하지 않고 스테이킹하는 사람이 해당 `penaltyPeriodSeconds` 동안 플래그될 가능성이 매우 높도록 네트워크 매개변수를 선택하는 것을 목표로 합니다. 그러면 플래깅을 프론트 러닝해도 슬래싱을 피할 수 없습니다.

**Cyfrin:** 인지함.

### 운영자가 `reduceStakeTo()`를 사용하여 `leavePenalty`를 우회할 수 있음

**심각도:** 중간

**설명:** 운영자는 예상보다 일찍 언스테이킹할 때 `leavePenalty`를 지불해야 합니다.
하지만 `reduceStakeTo()`에는 관련 요구 사항이 없으므로 스테이킹 금액을 최소값으로 줄일 수 있습니다.

- 운영자가 100을 스테이킹했고 더 일찍 언스테이킹하려고 합니다.
- `forceUnstake()`를 호출하면 `100 * 10% = 10`을 페널티로 지불해야 합니다.
- 그러나 먼저 `reduceStakeTo()`를 사용하여 스테이킹 금액을 최소값(예: 10)으로 줄인 다음 `forceUnstake()`를 호출하면 페널티는 `10 * 10% = 1`이 됩니다.

**파급력:** 운영자는 최소 금액에 대해서만 `leavePenalty`를 지불하게 됩니다.

**권장 완화 조치:** 운영자가 `forceUnstake`만 호출하든 먼저 `reduceStakeTo`를 호출하든 페널티는 동일해야 합니다.

**클라이언트:** 커밋 [72323d0](https://github.com/streamr-dev/network-contracts/commit/72323d0099a85c8c7a5d59335492eebcc9cc66bb)에서 수정되었습니다.

**Cyfrin:** 확인됨

### `Operator._transfer()`에서 `onDelegate()`는 토큰 잔액 업데이트 후 호출되어야 함

**심각도:** 중간

**설명:** `_transfer()`에서 소유자의 `minimumSelfDelegationFraction` 요건을 검증하기 위해 `onDelegate()`가 호출됩니다.

```solidity
File: contracts\OperatorTokenomics\Operator.sol
324:         // transfer creates a new delegator: check if the delegation policy allows this "delegation"
325:         if (balanceOf(to) == 0) {
326:             if (address(delegationPolicy) != address(0)) {
327:                 moduleCall(address(delegationPolicy), abi.encodeWithSelector(delegationPolicy.onDelegate.selector, to)); //@audit
328:should be called after _transfer()
329:             }
330:         }
331:
332:         super._transfer(from, to, amount);
```

하지만 `onDelegate()`가 토큰 잔액 업데이트 전에 호출되므로 아래 시나리오가 가능합니다.

- 운영자 소유자가 100주(필수 최소 비율)를 보유하고 있습니다. 그리고 위임 해제 정책은 없습니다.
- 논리적으로, 소유자는 `onDelegate()`의 최소 비율 요건으로 인해 자신의 100주를 새로운 위임자에게 양도할 수 없어야 합니다.
- 그러나 소유자가 `transfer(owner, to, 100)`를 호출하면 `onDelegation()`에서 `balanceOf(owner)`는 100이 되며, 이는 `super._transfer()` 전에 호출되기 때문에 요건을 통과합니다.

**파급력:** 운영자 소유자가 슬래싱을 피하기 위해 슬래싱이 예상될 때 자신의 지분을 다른 위임자에게 양도할 수 있습니다.

**권장 완화 조치:** `onDelegate()`는 `super._transfer()` 이후에 호출되어야 합니다.

**클라이언트:** 커밋 [93d6105](https://github.com/streamr-dev/network-contracts/commit/93d610561c109058c967d1d2f49ea91811f28579)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 중앙화 위험

**심각도:** 중간

**설명:** 프로토콜에는 사용자에게 영향을 미칠 수 있는 관리 작업을 수행할 수 있는 특권 권한을 가진 `DEFAULT_ADMIN_ROLE`이 있습니다. 특히 소유자는 수수료/보상 비율 설정 및 다양한 정책을 변경할 수 있습니다.

대부분의 관리자 기능은 현재 이벤트를 발생시키지 않습니다.

**파급력:** 프로토콜 소유자는 신뢰할 수 있는 당사자로 간주되지만, 소유자는 로깅 없이 많은 설정과 정책을 변경할 수 있습니다. 이는 예상치 못한 결과를 초래할 수 있으며 사용자에게 영향을 줄 수 있습니다.

**권장 완화 조치:** 문서에 소유자의 권한과 책임을 명시하십시오.
비율 설정의 최소값과 최대값으로 사용할 수 있는 상수 상태 변수를 추가하십시오.
중요한 상태 변수의 변경 사항을 이벤트를 통해 기록하십시오.

**클라이언트:** 커밋 [c530ec5](https://github.com/streamr-dev/network-contracts/commit/c530ec5092c1d6567357ef9993a0f029f113a0d8)에서 StreamrConfig에 로깅이 추가되었습니다. 커밋 [c343850](https://github.com/streamr-dev/network-contracts/commit/c343850136a97573349d7cb226733c9fd5729eb6)에서 더 나은 문서화와 다른 계약에 대한 로깅이 추가되었습니다. 이러한 커밋은 관리자 키 유출과 관련된 위험을 부분적으로 완화합니다.

StreamrConfig에서 구성 값을 변경하는 권한과 전체 계약을 교체하는 권한(업그레이드 가능)에는 큰 차이가 없습니다. 현재 일부 최대 및 최소 제한이 존재하지만, 그들의 주된 요점은 새로운 값, 특히 초기 값을 온전성 검사(sanity-check)하는 것입니다. 더 엄격한 제한으로 "손을 묶는 것"은 실제로 아무것도 바꾸지 않을 것이며, 기껏해야 의도를 나타낼 뿐입니다.

이러한 관리자 권한을 사용하여 구성 값을 변경하거나 업그레이드를 통해 계약을 수정하기 전에, 사용자에게 영향을 줄 수 있는 예상치 못한 부작용을 피하기 위해 더 광범위한 검토(커뮤니티, 감사자)가 필요할 것입니다. 일상적인 운영은 관리자의 개입이 필요하지 않도록 설계되었습니다. 관리자 권한은 예기치 않은 상황(예: 버그 핫픽스)이나 계획된 정책 변경에만 필요합니다. 현재로서는 그러한 변경이 필요할 것으로 예상되지 않습니다.

**Cyfrin:** 인지함.

### `onTokenTransfer`가 DATA 토큰 컨트랙트에서의 호출인지 검증하지 않음

**심각도:** 중간

**설명:** `SponsorshipFactory::onTokenTransfer` 및 `OperatorFactory::onTokenTransfer`는 단일 트랜잭션에서 토큰 전송 및 계약 배포를 처리하는 데 사용됩니다. 그러나 호출이 DATA 토큰 컨트랙트에서 온 것인지에 대한 검증이 없으며 누구나 이 함수들을 호출할 수 있습니다.

`Sponsorship` 배포의 경우 영향이 적지만, `Operator` 배포의 경우 운영자 토큰 이름과 운영자 주소를 기반으로 한 솔트(salt)와 함께 `ClonesUpgradeable.cloneDeterministic`이 사용됩니다. 공격자는 이를 악용하여 배포에 대한 DoS를 유발할 수 있습니다.

`Operator`와 같은 다른 컨트랙트에서는 이 검증이 올바르게 구현되어 있음을 확인했습니다.
```solidity
       if (msg.sender != address(token)) {
           revert AccessDeniedDATATokenOnly();
       }
```

**파급력:** 공격자가 `Operator` 컨트랙트의 배포를 방해할 수 있습니다.

**권장 완화 조치:** 호출자가 실제 DATA 컨트랙트인지 확인하는 검증을 추가하십시오.

**클라이언트:** 커밋 [8b13df4](https://github.com/streamr-dev/network-contracts/commit/8b13df49900c640df51e22f7c5a78fcad761f7cb)에서 수정되었습니다.

**Cyfrin:** 확인됨.

## 저위험

### `IERC20`와 함께 `transfer()/transferFrom()`의 안전하지 않은 사용

일부 토큰은 ERC20 표준을 제대로 구현하지 않지만 ERC20 토큰을 허용하는 대부분의 코드에서 여전히 허용됩니다. 예를 들어 Tether (USDT)의 L1에서의 `transfer()` 및 `transferFrom()` 함수는 사양에서 요구하는 대로 불리언을 반환하지 않고 반환 값이 없습니다. 대신 OpenZeppelin의 `SafeERC20`의 `safeTransfer()/safeTransferFrom()`을 사용하는 것을 고려하십시오.

```solidity
File: contracts\OperatorTokenomics\Operator.sol
264:         token.transferFrom(_msgSender(), address(this), amountWei);
442:         token.transfer(msgSender, rewardDataWei);

File: contracts\OperatorTokenomics\Sponsorship.sol
187:         token.transferFrom(_msgSender(), address(this), amountWei);
215:         token.transferFrom(_msgSender(), address(this), amountWei);
261:         token.transfer(streamrConfig.protocolFeeBeneficiary(), slashedWei);
273:         token.transfer(operator, payoutWei);

File: contracts\OperatorTokenomics\OperatorPolicies\QueueModule.sol
86:           token.transfer(delegator, amountDataWei);

File: contracts\OperatorTokenomics\OperatorPolicies\StakeModule.sol
128:         token.transfer(streamrConfig.protocolFeeBeneficiary(), protocolFee);
```

**클라이언트:** 우리는 시스템에서 DATA 토큰만 사용할 것입니다. DATA 토큰에는 위의 메서드가 없습니다. 따라서: `transfer`가 맞습니다.

**Cyfrin:** 인지함.

## 정보성 발견 사항

### 중복된 요구 사항
두 번째 요구 사항으로 충분하므로 첫 번째 요구 사항은 중복됩니다.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
156:         require(reviewerState[target][voter] != Reviewer.NOT_SELECTED, "error_reviewersOnly"); //@audit-issue redundant
157:         require(reviewerState[target][voter] == Reviewer.IS_SELECTED, "error_alreadyVoted");
```

**클라이언트:** 검토자가 아닌 사람이 투표하려고 할 때 유익한 오류 메시지를 제공하고 싶습니다. 따라서: 유지하는 것을 선호합니다.

**Cyfrin:** 인지함.

### 변수를 0으로 초기화할 필요 없음
변수의 기본값은 0이므로 0으로 초기화하는 것은 불필요합니다.

```solidity
File: contracts\OperatorTokenomics\SponsorshipPolicies\DefaultLeavePolicy.sol
10:           uint public penaltyPeriodSeconds = 0;

File: contracts\OperatorTokenomics\SponsorshipPolicies\VoteKickPolicy.sol
100:         uint sameSponsorshipPeerCount = 0;
226:         uint rewardsWei = 0;
```

**클라이언트:** 커밋 [c847fab](https://github.com/streamr-dev/network-contracts/commit/c847fab3e57f1f2dc3aa26e8cb79d44c8ed86f5e)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 이벤트가 적절하게 `indexed`되지 않음
이벤트 필드를 인덱싱하면 오프체인 도구가 이벤트를 파싱할 때 해당 필드에 더 빠르게 접근할 수 있습니다. 이는 특히 주소를 기반으로 필터링할 때 유용합니다. 그러나 인덱스 필드는 방출 시 추가 가스 비용이 발생하므로, 이벤트당 허용되는 최대값(3개 필드)을 인덱싱하는 것이 반드시 최선은 아닙니다. 해당되는 경우, 3개 이상의 필드가 있고 가스 사용량이 해당 이벤트에 대해 특별히 우려되지 않는다면 3개의 인덱스 필드를 사용해야 합니다. 해당 필드가 3개 미만인 경우 해당되는 모든 필드를 인덱싱해야 합니다.

**클라이언트:** 커밋 [92d4145](https://github.com/streamr-dev/network-contracts/commit/92d4145aaf6f56c3e9c72b4f9ef53a89e67aa36e)에서 수정되었습니다.

**Cyfrin:** 확인됨.

