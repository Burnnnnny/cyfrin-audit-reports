**수석 감사자**

[Dacian](https://twitter.com/DevDacian)

[Carlitox477](https://twitter.com/carlitox477)

**보조 감사자**



---

# 발견 사항 (Findings)
## 중급 위험 (Medium Risk)


### 봇 메서드가 일시 중지될 때 `SwellLib.BOT`이 활성 검증자를 삭제할 수 있음

**설명:** `SwellLib.BOT`이 호출할 수 있는 거의 모든 함수는 봇 메서드가 일시 중지될 때 봇 기능이 작동하지 않도록 다음 검사를 포함합니다:

```solidity
if (AccessControlManager.botMethodsPaused()) {
  revert SwellLib.BotMethodsPaused();
}
```

유일한 예외는 [`NodeOperatorRegistry::deleteActiveValidators`](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/NodeOperatorRegistry.sol#L417-L423)로, 봇 메서드가 일시 중지된 경우에도 `SwellLib.BOT`에 의해 호출될 수 있습니다. 다음을 고려하십시오:
* 봇 메서드가 일시 중지된 경우 `SwellLib.BOT`이 호출할 수 없도록 이 함수에 유사한 검사 추가
* 또는 봇 메서드가 일시 중지된 경우에도 `SwellLib.BOT`이 호출할 수 있어야 함을 명시하는 주석을 이 함수에 추가.

첫 번째 솔루션에 대한 가능한 구현 중 하나:
```solidity
bool isBot = AccessControlManager.hasRole(SwellLib.BOT, msg.sender);

// prevent bot from calling this function when bot methods are paused
if(isBot && AccessControlManager.botMethodsPaused()) {
  revert SwellLib.BotMethodsPaused();
}

// function only callable by admin & bot
if (!AccessControlManager.hasRole(SwellLib.PLATFORM_ADMIN, msg.sender) && !isBot) {
  revert OnlyPlatformAdminOrBotCanDeleteActiveValidators();
}
```

**Swell:** 커밋 [1a105b7](https://github.com/SwellNetwork/v3-contracts-lst/commit/1a105b76899780e30b1fb88abdede11c0c0586ba)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `swEXIT::processWithdrawals` 호출 시 `_processedRate = 0`을 설정하여 `SwellLib.BOT`이 출금을 교묘하게 러그풀(rug-pull) 할 수 있음

**설명:** 사용자가 출금 요청을 생성하면 `swETH`가 [소각](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L202-L205)된 다음 현재 환율 `rateWhenCreated`를 `swETH::swETHToETHRate`에서 [가져옵니다](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L213):
```solidity
uint256 rateWhenCreated = AccessControlManager.swETH().swETHToETHRate();
```

그러나 `SwellLib.BOT`은 `swEXIT::processWithdrawals`를 호출할 때 `_processedRate`에 대해 [임의의 값을 전달](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L111)할 수 있습니다:
```solidity
function processWithdrawals(
  uint256 _lastTokenIdToProcess,
  uint256 _processedRate
) external override checkRole(SwellLib.BOT) {
```

사용되는 [최종 환율](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L150-L152)은 `rateWhenCreated`와 `_processedRate` 중 작은 값입니다:
```solidity
uint256 finalRate = _processedRate > rateWhenCreated
  ? rateWhenCreated
  : _processedRate;
```

이 최종 환율은 요청된 출금 금액에 [곱해져](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L158) 출금을 요청한 사용자에게 전송되는 실제 금액을 결정합니다:
```solidity
uint256 requestExitedETH = wrap(amount).mul(wrap(finalRate)).unwrap();
```

따라서 `SwellLib.BOT`은 `swEXIT::processWithdrawals`를 호출할 때 `_processedRate = 0`을 설정하여 모든 출금을 교묘하게 러그풀할 수 있습니다.

**권장 완화 조치:** 두 가지 가능한 완화 조치:
1) `swEXIT::processWithdrawals`가 항상 `swETH::swETHToETHRate`에서 현재 환율을 가져오도록 변경
2) `swEXIT::processWithdrawals`가 [올바르게 호출](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L130-L132)하는 `RepricingOracle` 계약에 의해서만 호출되도록 허용.

**Swell:** 커밋 [c6f8708](https://github.com/SwellNetwork/v3-contracts-lst/commit/c6f870847bdf276aee1bf9aeb1ed71771a2aba04), [64cfbdb](https://github.com/SwellNetwork/v3-contracts-lst/commit/64cfbdbf67e28d84f2a706982e28925ab51fd5e6)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 저위험 (Low Risk)


### 곱셈 전 불필요한 나눗셈으로 인한 `swETH::reprice`의 정밀도 손실

**설명:** `swETH::reprice` [L281-286](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L281-L286)은 노드 운영자 보상을 계산할 때 불필요한 [곱셈 전 나눗셈](https://dacian.me/precision-loss-errors#heading-division-before-multiplication)을 수행하여 정밀도 손실로 인해 노드 운영자 보상에 부정적인 영향을 미칩니다:

```solidity
UD60x18 nodeOperatorRewardPortion = wrap(nodeOperatorRewardPercentage)
  .div(wrap(rewardPercentageTotal));

nodeOperatorRewards = nodeOperatorRewardPortion
  .mul(rewardsInSwETH) // @audit mult after division
  .unwrap();
```

곱셈 후 나눗셈을 수행하도록 리팩터링:

```solidity
nodeOperatorRewards = wrap(nodeOperatorRewardPercentage)
  .mul(rewardsInSwETH)
  .div(wrap(rewardPercentageTotal))
  .unwrap();
```

운영자 보상 점유율을 계산할 때도 유사한 문제가 발생합니다 [L310-313](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L310-L313):

```solidity
uint256 operatorsRewardShare = wrap(operatorActiveValidators)
  .div(totalActiveValidators)
  .mul(wrap(nodeOperatorRewards)) // @audit mult after division
  .unwrap();
```

이것도 유사하게 곱셈을 먼저 수행하여 정밀도 손실을 방지하도록 리팩터링할 수 있습니다:

```solidity
uint256 operatorsRewardShare = wrap(operatorActiveValidators)
  .mul(wrap(nodeOperatorRewards))
  .div(totalActiveValidators)
  .unwrap();
```

이 문제는 새로운 변경 사항에서 도입된 것이 아니라 메인넷 코드에 있습니다 ([1](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/swETH.sol#L267-L272), [2](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/swETH.sol#L296-L299)).

`rewardsInSwETH`는 [나눗셈이 수행](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L239)된 후 [곱해지기](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L285) 때문에 잠재적인 정밀도 손실이 하나 남아 있지만, 이를 리팩터링하려고 시도하면 "stack too deep" 오류가 발생하여 피할 수 없을 수 있습니다.

**Swell:** 인지함.


### 유효성 검사가 부족한 `swEXIT::setWithdrawRequestMaximum` 및 `setWithdrawRequestMinimum`으로 인해 `withdrawRequestMinimum > withdrawRequestMaximum` 상태가 될 수 있음

**설명:** 불변 조건 `withdrawRequestMinimum <= withdrawRequestMaximum`은 항상 유지되어야 하지만, 새로운 최소/최대 출금 값이 설정될 때 이것이 확인되지 않습니다. 따라서 `withdrawRequestMinimum > withdrawRequestMaximum`인 비논리적인 상태로 진입할 수 있습니다.

**권장 완화 조치:**
```diff
  function setWithdrawRequestMaximum(
    uint256 _withdrawRequestMaximum
  ) external override checkRole(SwellLib.PLATFORM_ADMIN) {
+   require(withdrawRequestMinimum <= _withdrawRequestMaximum);

    emit WithdrawalRequestMaximumUpdated(
      withdrawRequestMaximum,
      _withdrawRequestMaximum
    );
    withdrawRequestMaximum = _withdrawRequestMaximum;
  }

  function setWithdrawRequestMinimum(
    uint256 _withdrawRequestMinimum
  ) external override checkRole(SwellLib.PLATFORM_ADMIN) {
+   require(_withdrawRequestMinimum <= withdrawRequestMaximum);

    emit WithdrawalRequestMinimumUpdated(
      withdrawRequestMinimum,
      _withdrawRequestMinimum
    );
    withdrawRequestMinimum = _withdrawRequestMinimum;
  }
```

**Swell:** 커밋 [a9dfe5c](https://github.com/SwellNetwork/v3-contracts-lst/commit/a9dfe5cef35404e4e957e8001d571b1cf43feb0a)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `swExit::getProcessedRateForTokenId`가 존재하지 않는 `tokenId` 입력에 대해 유효한 `processedRate`와 함께 `true`를 반환함

**설명:** `swExit::getProcessedRateForTokenId`는 존재하지 않는 `tokenId` 입력에 대해 유효한 `processedRate`와 함께 `true`를 반환합니다.

**영향:** 이 `public` 함수는 잘못된 입력에 대해 유효한 출력을 반환할 수 있습니다. 현재는 `finalizeWithdrawal`에서만 사용되는 것으로 보이며, 해당 함수는 `getProcessedRateForTokenId`를 호출하기 전에 존재하지 않는 토큰을 확인하므로 이 동작이 추가로 악용될 것 같지는 않습니다.

**개념 증명(Proof of Concept):** `getProcessedRateForTokenId.test.ts`에 다음 PoC를 추가하십시오:
```typescript
  it("Should return false for isProcessed when tokens have been processed but this token doesn't exist", async () => {
    await createWithdrawRequests(Deployer, 5);

    await swEXIT_Deployer.processWithdrawals(4, parseEther("1"));

    // @audit this test fails
    expect(await getProcessedRateForTokenId(0)).eql({
      isProcessed: false,               // @audit returns true
      processedRate: BigNumber.from(0), // @audit returns > 0
    });
  });
```

**권장 완화 조치:** `swExit::getProcessedRateForTokenId`는 `tokenId`가 존재하지 않을 때 `return(false, 0)`해야 합니다. 이 함수에서 현재 처리되지 않은 유일한 엣지 케이스는 `tokenId = 0`인 경우인 것으로 보입니다.

**Swell:** 커밋 [4c8cbfd](https://github.com/SwellNetwork/v3-contracts-lst/commit/4c8cbfde6fdb54385f8bab83c33f90409fd0a412), [262db73](https://github.com/SwellNetwork/v3-contracts-lst/commit/262db7361f543611237e889313b8022a47b77144)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### Chainlink `Swell ETH PoR` 오라클을 통해 준비금 증명을 가져올 때 데이터의 최신성(staleness) 확인

**설명:** `RepricingOracle::_assertRepricingSnapshotValidity`는 `Swell ETH PoR` Chainlink Proof Of Reserves 오라클을 [사용](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L329-L331)하여 Swell의 현재 준비금에 대한 오프체인 데이터 소스를 가져옵니다.

`Swell ETH PoR` 오라클은 Chainlink 웹사이트에 `86400`초의 하트비트를 갖는 것으로 [나열](https://docs.chain.link/data-feeds/proof-of-reserve/addresses?network=ethereum&page=1#networks)되어 있지만(테이블 오른쪽 상단의 "Show More Details" 상자 확인), `RepricingOracle`에 [최신성 검사가 구현되어 있지 않습니다](https://medium.com/SwellNetwork/chainlink-oracle-defi-attacks-93b6cb6541bf#99af):
```solidity
// @audit no staleness check
(, int256 externallyReportedV3Balance, , , ) = AggregatorV3Interface(
  ExternalV3ReservesPoROracle
).latestRoundData();
```

**영향:** `Swell ETH PoR` Chainlink Proof Of Reserves 오라클이 올바르게 작동하지 않는 경우, `RepricingOracle::_assertRepricingSnapshotValidity`는 오래된 준비금 데이터를 최신 데이터인 것처럼 계속 처리합니다.

**권장 완화 조치:** 최신성 검사를 구현하고 오라클이 오래된 경우 [오라클이 설정되지 않은 경우](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L325-L327) 현재 코드가 수행하는 것처럼 revert 하거나 사용을 건너뛰십시오.

멀티체인 배포의 경우 동일한 피드가 다른 체인에서 다른 하트비트를 가질 수 있으므로 [각 피드에 대해 올바른 최신성 검사가 사용되는지 확인하십시오](https://medium.com/SwellNetwork/chainlink-oracle-defi-attacks-93b6cb6541bf#fb78).

오라클이 오래되었는지 주기적으로 확인하고 그렇다면 팀이 조사할 수 있도록 내부 경고를 제기하는 오프체인 봇을 추가하는 것을 고려하십시오.

**Swell:** 커밋 [84a6517](https://github.com/SwellNetwork/v3-contracts-lst/commit/84a65178c31222d80559f6fd5f1b4c60f9249016)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 모든 검증자 운영자를 반복하기 때문에 `swETH::reprice`가 많은 수의 운영자로 확장될 때 가스가 부족하거나 엄청나게 비쌀 수 있음

**설명:** `swETH::reprice`는 모든 검증자 운영자를 [루프](https://github.com/SwellNetwork/v3-contracts-lst/tree/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L303-L321)하여 보상 점유율을 지급합니다:
```solidity
// @audit may run out of gas for larger number of validator operators
// or make repricing exorbitantly expensive
for (uint128 i = 1; i <= totalOperators; ) {
  (
    address rewardAddress,
    uint256 operatorActiveValidators
  ) = nodeOperatorRegistry.getRewardDetailsForOperatorId(i);

  if (operatorActiveValidators != 0) {
    uint256 operatorsRewardShare = wrap(operatorActiveValidators)
      .div(totalActiveValidators)
      .mul(wrap(nodeOperatorRewards))
      .unwrap();

    _transfer(address(this), rewardAddress, operatorsRewardShare);
  }

  // Will never overflow as the total operators are capped at uint128
  unchecked {
    ++i;
  }
}
```
Swell이 많은 수의 검증자로 확장되면 `swETH::reprice`가 가스 부족으로 인해 revert 되거나 리프라이싱(reprice) 작업이 엄청나게 비쌀 수 있습니다. `NodeOperatorRegistry::getNextValidatorDetails`도 유사하게 [영향](https://github.com/SwellNetwork/v3-contracts-lst/blob/c9a1e6c06d0f5b358f5c3d4b7644db7a33952444/contracts/implementations/NodeOperatorRegistry.sol#L117-L125)을 받을 수 있습니다.

현재 Swell은 ["전문 노드 운영자의 허가된 그룹"](https://docs.swellnetwork.io/swell/sweth-liquid-staking/sweth-v1.0-system-design/node-operators-set)의 작은 집합을 사용하므로 이는 낮은 위험을 나타냅니다.

그러나 Swell은 이 방식에서 [전환](https://docs.swellnetwork.io/swell/sweth-liquid-staking/sweth-v1.0-system-design/node-operators-set)할 의도가 있습니다: _"후속 반복에서는 운영자 집합이 **확장**되고 궁극적으로 무허가형이 될 것입니다.."_

Swell이 운영자 집합을 확장함에 따라 이 문제는 더 심각한 우려 사항이 될 것이며 완화가 필요할 수 있습니다.

**Swell:** 인지함.


### `NodeOperatorRegistry::updateOperatorControllingAddress`는 주소가 이미 운영자 ID에 할당된 경우 `_newOperatorAddress`를 덮어쓸 수 있도록 허용함

**설명:** 현재 구현에서는 새로 할당된 주소가 이미 운영자 ID에 할당되었는지 확인하지 않습니다. 결과적으로 `getOperatorIdForAddress` 매핑에서 현재 값을 덮어쓸 수 있으며, `getOperatorForOperatorId`는 동일한 운영자를 가리키는 2개의 운영자 ID를 갖게 됩니다.

이에 대한 직접적인 결과는 `_getOperatorSafe` 및 `_getOperatorIdSafe`에 미치며, 이는 새로 할당된 운영자 ID에 대한 데이터만 반환합니다.

따라서:
* `NodeOperatorRegistry::getOperatorsPendingValidatorDetails`는 이전 `_newOperatorAddress` 관련 검증자 세부 정보를 반환할 수 없습니다.
* `NodeOperatorRegistry::getOperatorsActiveValidatorDetails`는 이전 `_newOperatorAddress` 관련 활성 검증자 세부 정보를 반환할 수 없습니다.
* `enableOperator`는 이전 운영자 기록을 활성화할 수 없습니다.
* **__`disableOperator`는 이전 운영자 기록을 비활성화할 수 없습니다__**. 이는 `getOperatorForOperatorId[_newOperatorAddress].enabled` 저장소를 수정하고 [함수를 강제로 revert](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/NodeOperatorRegistry.sol#L194-L196) 시킬 방법이 없기 때문에, 프로토콜이 검증자 설정에 사용하기 위해 이미 활성화된 공개 키를 비활성화할 수 없음을 의미하여 `usePubKeysForValidatorSetup` 기능에 영향을 줄 수 있습니다. 이전에 `DepositManager::setupValidators`를 호출하여 함수를 호출할 수 있는 유일한 존재가 BOT이라는 점을 감안할 때 영향은 제한적입니다.
* `updateOperatorRewardAddress`는 이전 운영자 기록의 보상 주소를 수정할 수 없습니다.
* `updateOperatorName`은 이전 운영자 기록의 이름을 수정할 수 없습니다.

이 문제는 새로운 변경 사항에서 도입된 것이 아니라 메인넷 [코드](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/NodeOperatorRegistry.sol#L348-L364)에 있습니다.

**개념 증명(Proof Of Concept):**
`updateOperatorFields.test.ts`에 다음 테스트를 추가하십시오:
```typescript
    it("Should revert updating operator controlling address to existing address", async () => {
      // create another operator
      await NodeOperatorRegistry_Deployer.addOperator(
        "OPERATOR_2",
        NewOperator.address,
        NewOperator.address
      );

      // attempt to update first operator's controlling address to be
      // the same as the newly created operator - should revert but doesn't
      await NodeOperatorRegistry_Deployer.updateOperatorControllingAddress(
        Operator.address,
        NewOperator.address
      );
    });
```

**권장 완화 조치:**
`_newOperatorAddress`가 이미 운영자에 할당되지 않았는지 확인하십시오 ([이미 수행하는](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/NodeOperatorRegistry.sol#L300-L302) `addOperator`와 유사하게, 코드 재사용을 위해 새로운 private 또는 public 함수를 만들 수 있습니다):
```diff
  function updateOperatorControllingAddress(
    address _operatorAddress,
    address _newOperatorAddress
  )
    external
    override
    checkRole(SwellLib.PLATFORM_ADMIN)
    checkZeroAddress(_newOperatorAddress)
  {

    if (_operatorAddress == _newOperatorAddress) {
        revert CannotSetOperatorControllingAddressToSameAddress();
    }
+   if(getOperatorIdForAddress[_newOperatorAddress] != 0){
+       revert CannotUpdateOperatorControllingAddressToAlreadyAssignedAddress();
+   }

    uint128 operatorId = _getOperatorIdSafe(_operatorAddress);

    getOperatorIdForAddress[_newOperatorAddress] = operatorId;
    getOperatorForOperatorId[operatorId]
      .controllingAddress = _newOperatorAddress;

    delete getOperatorIdForAddress[_operatorAddress];
  }
```

**Swell:** 커밋 [55c7d5f](https://github.com/SwellNetwork/v3-contracts-lst/commit/55c7d5fba6d55c68558dcd15de016927e07e38fd)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 누구나 모든 출금을 완료할 수 있도록 허용하면 ETH를 받을 수 있는 스마트 계약에 통합 문제가 발생할 수 있음

**설명:** `swEXIT::finalizeWithdrawal`의 현재 구현은 이미 처리된 모든 출금 요청을 누구나 완료할 수 있도록 허용합니다. 그러나 이 설계 결정은 NFT 소유자가 항상 출금을 완료하기를 원한다는 강력한 가정을 하는데, 항상 그렇지는 않을 수 있습니다.

**영향:** 이미 처리된 출금 요청을 누구나 완료할 수 있도록 허용하면 일부 스마트 계약에 ETH가 갇힐 수 있습니다.

**POC:**
모든 토큰 또는 ETH를 받을 수 있는 경매를 진행하는 것이 목표인 프로토콜을 가정해 봅시다. 입찰자는 NFT에 대해 제공하는 토큰/ETH의 양에 대한 기록을 가지고 있으므로, 스마트 계약은 ETH를 수락하는 `receive` 함수를 구현합니다.

Eve는 출금 요청을 시작했지만, ETH가 급하게 필요하여 이 프로토콜을 사용하여 경매에서 NFT를 판매하기로 결정합니다. 이를 위해 그녀는 NFT를 경매 계약으로 전송해야 합니다.

Alice는 NFT에 입찰하기로 결정하고 경매가 끝날 때 낙찰받습니다. 이제 그녀는 NFT를 청구해야 합니다(현재 NFT의 소유자는 경매 계약입니다).

swEXIT NFT는 Alice가 청구하기 전에 처리됩니다. Eve는 경매 계약에 있는 NFT로 `finalizeWithdrawal`을 호출합니다. 이 계약은 ETH를 받을 수 있고 NFT 소유자이므로 트랜잭션이 revert 되지 않고, NFT와 관련된 ETH는 이제 경매 계약에 영원히 갇히게 되며 Alice는 아무것도 청구할 수 없습니다.

**권장 완화 조치:** NFT 소유자만 출금을 완료할 수 있도록 허용하십시오.

```diff
    function finalizeWithdrawal(uint256 tokenId) external override {
        if (AccessControlManager.withdrawalsPaused()) {
        revert WithdrawalsPaused();
        }

        address owner = _ownerOf(tokenId);

-       if (owner == address(0)) {
-           revert WithdrawalRequestDoesNotExist();
-       if (owner == msg.sender) {
+           revert WithdrawalRequestFinalizationOnlyAllowedForNFTOwner();
        }
```

**Swell:** 커밋 [b5d7a19](https://github.com/SwellNetwork/v3-contracts-lst/commit/b5d7a19e2f6de5c0ae086c8deaac5166767cd3fd)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `swETH::reprice`가 swETH 총 공급량을 늘리거나 줄임으로써 강제로 revert 되게 하는 다중 공격 경로

**설명:** 현재 총 swETH 공급량은 `swETH::reprice`에서 리프라이싱 중 최대 허용 총 swETH 공급량 차이를 시행하는 데 [사용](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L268-L273)됩니다. 총 공급량은 2가지 이유로 감소할 수 있습니다:
1. [출금이 완료됨](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L205)
2. 사용자가 [`swETH::burn`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L350-L356)을 호출하여 자신의 swETH를 소각함

총 공급량은 사용자가 `swETH::deposit`을 호출하여 증가할 수도 있습니다.

현재 공급량 차이가 최대 허용 차이 비율에 가까울수록 공격자가 다음을 통해 리프라이싱 트랜잭션을 프론트런하여 revert 시킬 가능성이 커집니다:
1. `swETH::deposit`을 통해 충분히 많은 양의 ETH를 예치하여 총 공급량 증가
2. 자신의 swETH를 소각하여 총 공급량 감소
3. 하나 이상의 출금을 완료(사용자가 다른 사람의 출금을 완료할 수 있음)하여 총 공급량 감소

**권장 완화 조치:**
가능한 완화 조치는 다음과 같습니다:
* 소각자(burner) 역할을 추가하고 `swEXIT`에만 할당한 다음, `swETH::burn`에 이 역할을 확인하는 해당 modifier를 추가합니다.
* NFT 소유자만 자신의 출금 요청을 완료하도록 허용합니다.

그러나 이러한 잠재적 완화 조치는 기능을 제한하는 동시에 공격자가 `swETH::deposit` 경로를 통해 리프라이싱을 revert 시킬 수 있게 합니다. 또 다른 옵션은 봇이 [flashbots](https://www.flashbots.net/)와 같은 서비스를 통해 리프라이싱 트랜잭션을 수행하여 트랜잭션이 프론트런되지 않도록 하는 것입니다; 이는 사용자가 자신의 swETH를 소각하고 다른 사람의 출금을 완료할 수 있는 기능을 유지하면서 모든 공격 경로를 방지합니다.

**Swell:** flashbots를 사용하여 리프라이싱 트랜잭션 수행.


### 리프라이싱 중 모든 활성 검증자가 삭제되면 보상을 분배할 수 없음

**설명:** 불변성 퍼징(Invariant fuzzing)은 다음과 같은 경우 리프라이싱 중 흥미로운 엣지 케이스를 발견했습니다:

1) 지난 기간에 누적된 분배할 보상이 있음,
2) 리프라이싱 작업에서 모든 현재 활성 검증자가 삭제됨

검증자가 [먼저 삭제](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L114-L122)되므로 리프라이싱 트랜잭션이 `NoActiveValidators` [오류](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L298-L300)와 함께 revert 됩니다.

새로운 활성 검증자가 추가될 때까지 리프라이싱이 불가능하며, 추가되면 삭제된 이전 검증자가 생성한 보상을 새 검증자가 받게 됩니다. 또한 Aaron은 TG에서 확인했습니다: _`DepositManager`로 전송된 모든 ETH는 보상으로 간주되고 수수료 대상이 되므로 활성 검증자 없이도 수수료가 생성되는 것은 이론적으로 가능합니다._

**권장 완화 조치:** 리프라이싱 중 활성 검증자는 없지만 분배할 보상이 있는 경우, revert 하는 대신 보상이 Swell 재무부(treasury)로 가야 합니다.

**Swell:** 커밋 [5594e20](https://github.com/SwellNetwork/v3-contracts-lst/commit/5594e204083a8507e69f0c28f4d1d7162f9a20fd)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 적은 보상으로 리프라이싱하면 `ETH` 준비금이 증가하고 `swETH 대 ETH` 환율이 증가하지만 운영자나 재무부에 보상이 지급되지 않는 유효하지 않은 상태가 됨

**설명:** 불변성 퍼징은 적은 보상으로 리프라이싱을 사용하여 `ETH` 준비금이 증가하고 `swETH : ETH` 환율이 증가하지만 운영자나 재무부에 보상이 지급되지 않는 유효하지 않은 상태에 도달했습니다.

**개념 증명(Proof of Concept):** 리프라이싱 중:
1) `_snapshot.rewardsPayableForFees`에 대한 [`RepricingOracle`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L227)이나 `_newETHRewards`에 대한 `swETH::reprice`에 의해 시행되는 최소값이 없음
2) `swETH::reprice`에서 `rewardsInSwETH`를 [계산](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L239-L241)할 때 0으로 내림되는 정밀도 손실에 대한 검사가 없음

이로 인해 퍼저는 다음과 같은 유효하지 않은 상태에 도달합니다:
1) `_snapshot.rewardsPayableForFees`에 대해 작은 값으로 `RepricingOracle::submitSnapshotV2`를 호출하면 작은 `_newETHRewards`로 `swETH::reprice`가 호출됨
2) `swETH::reprice` 내부에서 작은 `_newETHRewards`는 `rewardsInSwETH`의 보상 계산에서 0으로 내림되는 정밀도 손실을 트리거하여 [보상이 분배되지 않음](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L278)
3) 그러나 `swETH::reprice`는 작은 양의 `_newETHRewards` 값을 사용하여 `lastRepriceETHReserves`를 [업데이트](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L337)하고 트랜잭션이 성공적으로 완료됨.

이 결과로 다음과 같은 유효하지 않은 상태가 됩니다:

1) `swETH::lastRepriceETHReserves` 증가
2) `swETH : ETH` 환율 증가
3) 운영자/재무부에 지급되는 보상 없음

이 단순화된 PoC를 `reprice.test.ts`에 추가할 수 있습니다:
```typescript
    it("audit small rewards not distributed while reserves and exchange rate increasing", async () => {
      const swellTreasuryRewardPercentage = parseEther("0.1");

      await swETH_Deployer.setSwellTreasuryRewardPercentage(
        swellTreasuryRewardPercentage
      );

      await swETH_Deployer.deposit({
        value: parseEther("1000"),
      });
      const preRewardETHReserves = parseEther("1100");

      const swETHSupply = parseEther("1000");

      const ethRewards = parseUnits("1", "wei");

      const swellTreasuryPre = await swETH_Deployer.balanceOf(SwellTreasury.address);
      const ethReservesPre = await swETH_Deployer.lastRepriceETHReserves();
      const rateBefore = await swETH_Deployer.swETHToETHRate();

      swETH_Bot.reprice(
          preRewardETHReserves,
          ethRewards,
          swETH_Deployer.totalSupply());

      const swellTreasuryPost = await swETH_Deployer.balanceOf(SwellTreasury.address);
      const ethReservesPost = await swETH_Deployer.lastRepriceETHReserves();
      const rateAfter = await swETH_Deployer.swETHToETHRate();

      // no rewards distributed to treasury
      expect(swellTreasuryPre).eq(swellTreasuryPost);

      // exchange rate increases
      expect(rateBefore).lt(rateAfter);

      // reserves increase
      expect(ethReservesPre).lt(ethReservesPost);

      // repricing using small `_newETHRewards` can lead to increasing reserves
      // and increasing exchange rate without reward payouts
    });
```

이것은 새로운 변경 사항에서 도입된 것이 아니라 현재 메인넷 코드에 존재합니다 [[1](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/swETH.sol#L264), [2](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/swETH.sol#L323-L325)].

**Swell:** 인지함.


### 불필요한 숨겨진 곱셈 전 나눗셈으로 인한 `swETH::_deposit`의 정밀도 손실

**설명:** `swETH::_deposit` [L170](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L170)은 `_ethToSwETHRate` 호출이 나눗셈을 수행한 다음 `msg.value`에 곱해지기 때문에 불필요한 숨겨진 [곱셈 전 나눗셈](https://dacian.me/precision-loss-errors#heading-division-before-multiplication)을 포함합니다:

```solidity
uint256 swETHAmount = wrap(msg.value).mul(_ethToSwETHRate()).unwrap();
// @audit expanding this out
// wrap(msg.value).mul(_ethToSwETHRate()).unwrap();
// wrap(msg.value).mul(wrap(1 ether).div(_swETHToETHRate())).unwrap();
```

이 문제는 새로운 변경 사항에서 도입된 것이 아니라 메인넷 [코드](https://github.com/SwellNetwork/v3-core-public/blob/master/contracts/lst/contracts/implementations/swETH.sol#L170)에 있습니다.

**영향:** 예금자에게 `swETH`가 약간 덜 민팅됩니다. 개별 예금자가 손해보는 금액은 작지만, 그 효과는 누적되며 예금자와 예금 규모가 증가함에 따라 증가합니다.

**개념 증명(Proof of Concept):** 이 독립형 무상태 퍼즈 테스트는 제공된 하드코딩된 테스트 케이스뿐만 아니라 이를 증명하기 위해 Foundry 내부에서 실행할 수 있습니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {UD60x18, wrap} from "@prb/math/src/UD60x18.sol";

import "forge-std/Test.sol";

// run from base project directory with:
// (fuzz test) forge test --match-test FuzzMint -vvv
// (hardcoded) forge test --match-test HardcodedMint -vvv
contract MintTest is Test {

    uint256 private constant SWETH_ETH_RATE = 1050754209601187151; //as of 2024-02-15

    function _mintOriginal(uint256 inputAmount) private pure returns(uint256) {
        // hidden division before multiplication
        // wrap(inputAmount).mul(_ethToSwETHRate()).unwrap();
        // wrap(inputAmount).mul(wrap(1 ether).div(_swETHToETHRate())).unwrap()

        return wrap(inputAmount).mul(wrap(1 ether).div(wrap(SWETH_ETH_RATE))).unwrap();
    }

    function _mintFixed(uint256 inputAmount) private pure returns(uint256) {
        // refactor to perform multiplication before division
        // wrap(inputAmount).mul(wrap(1 ether)).div(_swETHToETHRate()).unwrap();

        return wrap(inputAmount).mul(wrap(1 ether)).div(wrap(SWETH_ETH_RATE)).unwrap();
    }

    function test_FuzzMint(uint256 inputAmount) public pure {
        uint256 resultOriginal = _mintOriginal(inputAmount);
        uint256 resultFixed    = _mintFixed(inputAmount);

        assert(resultOriginal == resultFixed);
    }

    function test_HardcodedMint() public {
        // found by fuzzer
        console.log(_mintFixed(3656923177187149889) - _mintOriginal(3656923177187149889)); // 1

        // 100 eth
        console.log(_mintFixed(100 ether) - _mintOriginal(100 ether)); // 21

        // 1000 eth
        console.log(_mintFixed(1000 ether) - _mintOriginal(1000 ether)); // 215

        // 10000 eth
        console.log(_mintFixed(10000 ether) - _mintOriginal(10000 ether)); // 2159
    }
}
```

**권장 완화 조치:** 나눗셈 전에 곱셈을 수행하도록 리팩터링하십시오:
```solidity
uint256 swETHAmount = wrap(msg.value).mul(wrap(1 ether)).div(_swETHToETHRate()).unwrap();
```

**Swell:** 커밋 [cb093ea](https://github.com/SwellNetwork/v3-contracts-lst/commit/cb093eac675e5248a3f736a01a3725d794dd177e)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 정보성 (Informational)


### eth 전송 시 `ETHSent` 이벤트 방출

**설명:** `DepositManager::receive`는 eth를 받을 때 `ETHReceived` 이벤트를 방출하지만, `transferETHForWithdrawRequests`는 eth를 보낼 때 이벤트를 방출하지 않습니다. eth를 보낼 때 `ETHSent` 이벤트도 방출하는 것을 고려하십시오.

**Swell:** 커밋 [c82dd3c](https://github.com/SwellNetwork/v3-contracts-lst/commit/c82dd3c8ca6853816dd9f1982ab0a5ef50d78cf2)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `swEXIT::createWithdrawRequest`에서 Checks-Effects-Interactions 패턴 사용

**설명:** 현재 구현은 상태 변수를 수정하기 전에 [`_safeMint`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L209)를 사용합니다:
* `withdrawalRequests[tokenId]`
* `exitingETH`
* `_lastTokenIdCreated`

이는 수신자가 업데이트되지 않은 상태 변수에 액세스할 수 있는 잠재적인 재진입(re-entrancy)을 허용합니다. 감사 중 이 문제와 관련된 의미 있는 무허가형 공격 벡터는 발견되지 않았지만, 보안 모범 사례를 따르기 위해 `_safeMint`를 함수 끝으로 이동하는 것이 좋습니다:

```diff
  function createWithdrawRequest(
    uint256 amount
  ) external override checkWhitelist(msg.sender) {
    if (AccessControlManager.withdrawalsPaused()) {
      revert WithdrawalsPaused();
    }

    if (amount < withdrawRequestMinimum) {
      revert WithdrawRequestTooSmall(amount, withdrawRequestMinimum);
    }

    if (amount > withdrawRequestMaximum) {
      revert WithdrawRequestTooLarge(amount, withdrawRequestMaximum);
    }

    IswETH swETH = AccessControlManager.swETH();
    swETH.transferFrom(msg.sender, address(this), amount);

    // Burn the tokens first to prevent reentrancy and to validate they own the requested amount of swETH
    swETH.burn(amount);

-   uint256 tokenId = _lastTokenIdCreated + 1; // Start off at 1
+   uint256 tokenId = ++_lastTokenIdCreated; // Starts off at 1

-   _safeMint(msg.sender, tokenId);

    uint256 lastTokenIdProcessed = getLastTokenIdProcessed();

    uint256 rateWhenCreated = AccessControlManager.swETH().swETHToETHRate();

    withdrawalRequests[tokenId] = WithdrawRequest({
      amount: amount,
      timestamp: block.timestamp,
      lastTokenIdProcessed: lastTokenIdProcessed,
      rateWhenCreated: rateWhenCreated
    });

    exitingETH += wrap(amount).mul(wrap(rateWhenCreated)).unwrap();
-   _lastTokenIdCreated = tokenId;
+   _safeMint(msg.sender, tokenId);

    emit WithdrawRequestCreated(
      tokenId,
      amount,
      block.timestamp,
      lastTokenIdProcessed,
      rateWhenCreated,
      msg.sender
    );

  }
```

**Swell:** 커밋 [d13aa43](https://github.com/SwellNetwork/v3-contracts-lst/commit/d13aa4390bb75c4e491b9cd92b7f7561cbe4ec15), [3f85df3](https://github.com/SwellNetwork/v3-contracts-lst/commit/3f85df3ba0e91b26e4234b15ad94f492fa6d46ec)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `NodeOperatorRegistry` 업데이트 메서드에 이벤트 누락

**설명:** `NodeOperatorRegistry`의 다음 함수는 여러 저장소 위치를 업데이트하지만 이벤트를 방출하지 않습니다:
* `updateOperatorControllingAddress`
* `updateOperatorRewardAddress`
* `updateOperatorName`

저장소에 대한 업데이트를 반영하기 위해 이 함수들에서 이벤트를 방출하는 것을 고려하십시오.

**Swell:** 커밋 [5849640](https://github.com/SwellNetwork/v3-contracts-lst/commit/584964072b1543128c02e3287fe7746a8a094226)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `NodeOperatorRegistry::getNextValidatorDetails`에서 동일한 코드 리팩터링

**설명:** 이 두 `else if` [분기](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/NodeOperatorRegistry.sol#L151-L162)의 본문은 동일합니다:

```solidity
} else if (foundOperatorId == 0) {
  // If no operator has been found yet set the smallest operator active keys to the current operator
  smallestOperatorActiveKeys = operatorActiveKeys;

  foundOperatorId = operatorId;

  // If the current operator has less keys than the smallest operator active keys, then we want to use this operator
} else if (smallestOperatorActiveKeys > operatorActiveKeys) {
  smallestOperatorActiveKeys = operatorActiveKeys;

  foundOperatorId = operatorId;
}
```

따라서 코드는 다음과 같이 단순화될 수 있습니다:
```solidity
// If no operator has been found yet set the smallest operator active keys to the current operator
// If the current operator has less keys than the smallest operator active keys, then we want to use this operator
} else if (foundOperatorId == 0 ||
           smallestOperatorActiveKeys > operatorActiveKeys) {
  smallestOperatorActiveKeys = operatorActiveKeys;
  foundOperatorId = operatorId;
}
```

**Swell:** 커밋 [d457d8d](https://github.com/SwellNetwork/v3-contracts-lst/commit/d457d8d109770f86b2b6ab3f785e1678ca341d6f)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### 변경되지 않고 여러 번 읽힐 때 저장소 변수를 메모리에 캐시

**설명:** 저장소에서 읽는 것이 메모리에서 읽는 것보다 훨씬 비싸므로, 변경되지 않고 여러 번 읽힐 때 저장소 변수를 메모리에 캐시하십시오:

File: `NodeOperatorRegistry.sol`
```solidity
// @audit 저장소에서 `numOperators`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
113:    uint128[] memory operatorAssignedDetails = new uint128[](numOperators + 1);
125:     for (uint128 operatorId = 1; operatorId <= numOperators; operatorId++) {

// @audit 증가된 값을 메모리에 저장하여
// 동일한 값을 여러 번 읽지 않도록 함, 예:
// uint128 newNumOperators = ++numOperators;
305:    numOperators += 1;
// 그런 다음 L314,315에서 `newNumOperators` 사용
314:    getOperatorIdForAddress[_operatorAddress] = numOperators;
315:    getOperatorForOperatorId[numOperators] = operator;
// @audit `Operator` 구조체도 다음과 같이 초기화할 수 있음:
// getOperatorForOperatorId[numOperators] = Operator(true, _rewardAddress, _operatorAddress, _name, 0);

// @audit `getOperatorForOperatorId[operatorId].activeValidators` 캐시
660:    if (getOperatorForOperatorId[operatorId].activeValidators == 0) {
666:      getOperatorForOperatorId[operatorId].activeValidators - 1
```

File: `RepricingOracle.sol`
```solidity
// @audit 리프라이싱 후 확인된 환율을 캐시하고
// 이미 발생한 리프라이싱 중에만 환율이 변경되므로
// 출금 처리 시 캐시된 버전 사용
125:    if (swETHToETHRate > AccessControlManager.swETH().swETHToETHRate()) {
132:        AccessControlManager.swETH().swETHToETHRate() // The rate to use for processing withdrawals

// @audit 저장소에서 `upgradeableRepriceSnapshot.meta.blockNumber`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
290:    bool useOldSnapshot = upgradeableRepriceSnapshot.meta.blockNumber == 0;
294:      : upgradeableRepriceSnapshot.meta.blockNumber;

// @audit 저장소에서 `maximumRepriceBlockAtSnapshotStaleness`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
317:    if (snapshotStalenessInBlocks > maximumRepriceBlockAtSnapshotStaleness) {
320:        maximumRepriceBlockAtSnapshotStaleness
```

File: `swETH.sol`
```solidity
// @audit 저장소에서 `lastRepriceUNIX`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
222:    uint256 timeSinceLastReprice = block.timestamp - lastRepriceUNIX;
249:    if (lastRepriceUNIX != 0) {

// @audit 저장소에서 `minimumRepriceTime`을 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
224:    if (timeSinceLastReprice < minimumRepriceTime) {
226:        minimumRepriceTime - timeSinceLastReprice

// @audit 저장소에서 `nodeOperatorRewardPercentage`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
233:      nodeOperatorRewardPercentage;
281:      UD60x18 nodeOperatorRewardPortion = wrap(nodeOperatorRewardPercentage)

// @audit 저장소에서 `swETHToETHRateFixed`를 메모리에 캐시하여
// 동일한 값을 여러 번 읽지 않도록 함
253:        swETHToETHRateFixed
256:      uint256 maximumRepriceDiff = wrap(swETHToETHRateFixed)

// @audit 저장소 값을 다시 읽을 필요 없이 방금 업데이트된 메모리 변수 사용
// 불필요하지만 비싼 저장소 읽기 제거
337:    lastRepriceETHReserves = totalReserves;
338:    lastRepriceUNIX = block.timestamp;
339:    swETHToETHRateFixed = updatedSwETHToETHRateFixed;

341:   emit Reprice(
342:      lastRepriceETHReserves, // @audit use `totalReserves` instead
343:       swETHToETHRateFixed,   // @audit use `updatedSwETHToETHRateFixed` instead
344:       nodeOperatorRewards,
345:       swellTreasuryRewards,
346:       totalETHDeposited

// @audit 첫 번째 검사는 정기적인 사용 중에 대부분 실패하므로
// `swETHToETHRateFixed`는 동일한 값으로 저장소에서 두 번 읽힘
374:    if (swETHToETHRateFixed == 0) {
375:    return wrap(swETHToETHRateFixed);
```

File: `swEXIT.sol`
```solidity
// @audit revert 케이스에서 추가 저장소 읽기를 피하기 위해
// `withdrawRequestMinimum` 및 `withdrawRequestMaximum`을 메모리에 캐시하는 것을 고려
193:    if (amount < withdrawRequestMinimum) {
194:       revert WithdrawRequestTooSmall(amount, withdrawRequestMinimum);
195:     }

197:     if (amount > withdrawRequestMaximum) {
198:      revert WithdrawRequestTooLarge(amount, withdrawRequestMaximum);
199:     }
```

**Swell:** 커밋 [23be897](https://github.com/SwellNetwork/v3-contracts-lst/commit/23be89740b7659ab4d98435d6a924364635fb9ca), [3f85df3](https://github.com/SwellNetwork/v3-contracts-lst/commit/3f85df3ba0e91b26e4234b15ad94f492fa6d46ec)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 루프 외부에서 배열 길이 캐시 및 unchecked 루프 증분 고려

**설명:** 루프 외부에서 배열 길이를 캐시하고 `solc --ir-optimized --optimize`로 컴파일하지 않는 경우 `unchecked {++i;}` 사용을 고려하십시오:

File: `DepositManager.sol`
```solidity
// @audit `validatorDetails.length` 캐시
116:    for (uint256 i; i < validatorDetails.length; i++) {
```

File: `NodeOperatorRegistry.sol`
```solidity
// @audit `numOperators` 캐시
133:    uint128[] memory operatorAssignedDetails = new uint128[](numOperators + 1);
125:      for (uint128 operatorId = 1; operatorId <= numOperators; operatorId++) {

// @audit `_pubKeys.length` 캐시
189:    validatorDetails = new ValidatorDetails[](_pubKeys.length);
191:    for (uint256 i; i < _pubKeys.length; i++) {
227:    numPendingValidators -= _pubKeys.length;

// @audit `_validatorDetails.length` 캐시
243:    if (_validatorDetails.length == 0) {
257:        _validatorDetails.length >
263:    for (uint128 i; i < _validatorDetails.length; i++) {
282:    numPendingValidators += _validatorDetails.length;

// @audit `_pubKeys.length` 캐시
396:    for (uint128 i; i < _pubKeys.length; i++) {
412:     numPendingValidators -= _pubKeys.length;

// @audit `_pubKeys.length` 캐시
425:     for (uint256 i; i < _pubKeys.length; i++) {

// @audit `operatorIdToValidatorDetails[operatorId].length()` 캐시
628:     if (operatorIdToValidatorDetails[operatorId].length() == 0) {
634:      operatorIdToValidatorDetails[operatorId].length() - 1
```

File: `swEXIT.sol`
```solidity
// @audit `requestsToProcess + 1` 캐시
143:    for (uint256 i = 1; i < requestsToProcess + 1; ) {
```

File: `Whitelist.sol`
```solidity
// @audit `_addresses.length` 캐시
 84:    for (uint256 i; i < _addresses.length; ) {
102:    for (uint256 i; i < _addresses.length; ) {
```

**Swell:** 커밋 [3c67e88](https://github.com/SwellNetwork/v3-contracts-lst/commit/3c67e88dbea1bb4cdf0bfeda27b40e71e494ef2c), [3f85df3](https://github.com/SwellNetwork/v3-contracts-lst/commit/3f85df3ba0e91b26e4234b15ad94f492fa6d46ec)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `NodeOperatorRegistry::_parsePubKeyToString`: 피제수/인수가 2의 거듭제곱일 때 나눗셈/곱셈 대신 시프트 연산 사용

**설명:** `DIV` 및 `MUL` opcode는 각각 5 가스 단위의 비용이 드는 반면 시프트 연산은 3 가스 단위의 비용이 듭니다. 따라서 `NodeOperatorRegistry::_parsePubKeyToString`은 가스를 절약하기 위해 이를 활용할 수 있습니다:

```diff
+   uint256 private SYMBOL_LENGTH = 16 // Because _SYMBOLS.length = 16
    function _parsePubKeyToString(
        bytes memory pubKey
    ) internal pure returns (string memory) {
        // Create the bytes that will hold the converted string
-      bytes memory buffer = new bytes(pubKey.length * 2);
+       // make sure that pubKey.length * 2 <= 2^256
+      bytes memory buffer = new bytes(pubKey.length << 1);

        bytes16 symbols = _SYMBOLS;
+       uint256 symbolLength = symbols.length;
+       uint256 index;
        for (uint256 i; i < pubKey.length; i++) {
-           buffer[i * 2] = symbols[uint8(pubKey[i]) / symbols.length];
-           buffer[i * 2 + 1] = symbols[uint8(pubKey[i]) % symbols.length];
+           index = i << 1; // i * 2
+           buffer[index] = symbols[uint8(pubKey[i]) >> 4]; // SYMBOL_LENGTH = 2^4
+           buffer[index + 1] = symbols[uint8(pubKey[i]) % SYMBOL_LENGTH];
        }

        return string(abi.encodePacked("0x", buffer));
    }
```

이 함수의 더 최적화된 버전은 다음과 같습니다:
```solidity
bytes16 private constant _SYMBOLS = "0123456789abcdef";
uint256 private constant SYMBOL_LENGTH = 16; // Because _SYMBOLS.length = 16

function _parsePubKeyToString(bytes memory pubKey) internal pure returns (string memory) {
    // Create the bytes that will hold the converted string
    // make sure that pubKey.length * 2 <= 2^256
    uint256 pubKeyLength  = pubKey.length;
    bytes memory buffer   = new bytes(pubKeyLength << 1);

    uint256 index;
    for (uint256 i; i < pubKeyLength;) {
        index             = i << 1; // i * 2
        buffer[index]     = _SYMBOLS[uint8(pubKey[i]) >> 4]; // SYMBOL_LENGTH = 2^4
        buffer[index + 1] = _SYMBOLS[uint8(pubKey[i]) % SYMBOL_LENGTH];

        unchecked {++i;}
    }

    return string(abi.encodePacked("0x", buffer));
}
```

Foundry & [Halmos](https://github.com/a16z/halmos/)를 사용하는 다음 독립 실행형 테스트는 최적화된 버전이 원본과 동일한 출력을 반환하는지 확인합니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "forge-std/Test.sol";

// run from base project directory with:
// halmos --function test --match-contract ParseTest
contract ParseTest is Test {

    bytes16 private constant _SYMBOLS = "0123456789abcdef";
    uint256 private constant SYMBOL_LENGTH = 16; // Because _SYMBOLS.length = 16

    function _parseOriginal(bytes memory pubKey) internal pure returns (string memory) {
        // Create the bytes that will hold the converted string
        bytes memory buffer = new bytes(pubKey.length * 2);

        bytes16 symbols = _SYMBOLS;

        // This conversion relies on taking the uint8 value of each byte, the first character in the byte is the uint8 value divided by 16 and the second character is modulo of the 16 division
        for (uint256 i; i < pubKey.length; i++) {
            buffer[i * 2] = symbols[uint8(pubKey[i]) / symbols.length];
            buffer[i * 2 + 1] = symbols[uint8(pubKey[i]) % symbols.length];
        }

        return string(abi.encodePacked("0x", buffer));
    }

    function _parseOptimized(bytes memory pubKey) internal pure returns (string memory) {
        // Create the bytes that will hold the converted string
        // make sure that pubKey.length * 2 <= 2^256
        uint256 pubKeyLength  = pubKey.length;
        bytes memory buffer   = new bytes(pubKeyLength << 1);

        uint256 index;
        for (uint256 i; i < pubKeyLength;) {
            index             = i << 1; // i * 2
            buffer[index]     = _SYMBOLS[uint8(pubKey[i]) >> 4]; // SYMBOL_LENGTH = 2^4
            buffer[index + 1] = _SYMBOLS[uint8(pubKey[i]) % SYMBOL_LENGTH];

            unchecked {++i;}
        }

        return string(abi.encodePacked("0x", buffer));
    }

    function test_HalmosParse(bytes memory pubKey) public {
        string memory resultOriginal  = _parseOriginal(pubKey);
        string memory resultOptimized = _parseOptimized(pubKey);

        assertEq(resultOriginal, resultOptimized);

    }
}
```

**Swell:** 커밋 [7db1874](https://github.com/SwellNetwork/v3-contracts-lst/commit/7db187409c7161d981b32d639e8b925fafc431a8), [3f85df3](https://github.com/SwellNetwork/v3-contracts-lst/commit/3f85df3ba0e91b26e4234b15ad94f492fa6d46ec)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `swETH::reprice`에서 `_preRewardETHReserves - rewardsInETH.unwrap() + _newETHRewards` 대신 `totalReserves - rewardsInETH.unwrap()` 사용

**설명:** 둘 다 동일한 결과를 낳지만 첫 번째 표현식은 `SUB` opcode를 절약합니다. 또한 제안된 수정은 불변 조건의 의도를 더 잘 반영하는 더 간단한 코드를 생성합니다.

```diff
    // swETH::reprice
    uint256 totalReserves = _preRewardETHReserves + _newETHRewards;

    uint256 rewardPercentageTotal = swellTreasuryRewardPercentage +
      nodeOperatorRewardPercentage;

    UD60x18 rewardsInETH = wrap(_newETHRewards).mul(
      wrap(rewardPercentageTotal)
    );

    UD60x18 rewardsInSwETH = wrap(_swETHTotalSupply).mul(rewardsInETH).div(
-       wrap(_preRewardETHReserves - rewardsInETH.unwrap() + _newETHRewards)
+       wrap(totalReserves - rewardsInETH.unwrap())
    );
```

**Swell:** 커밋 [7db1874](https://github.com/SwellNetwork/v3-contracts-lst/commit/7db187409c7161d981b32d639e8b925fafc431a8)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 중복된 일시 중지 검사 제거

**설명:** 1) `swETH::reprice`에서 중복된 `botMethodsPaused` [검사](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swETH.sol#L208-L210)를 제거하십시오:

* 이 함수는 [`RepricingOracle::handleReprice`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L233)에 의해서만 호출됩니다.
* `RepricingOracle::handleReprice`는 [`submitSnapshot`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L89-L94) 및 [`submitSnapshotV2`](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L105-L112)에 의해서만 호출될 수 있으며, 둘 다 이미 `botMethodsPaused` 검사를 포함하고 있습니다.

2) `swEXIT::processWithdrawals`에서 중복된 `withdrawalsPaused` [검사](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/swEXIT.sol#L117-L119)를 제거하십시오. 이 함수는 이미 검사를 [포함하는](https://github.com/SwellNetwork/v3-contracts-lst/blob/a95ea7942ba895ae84845ab7fec1163d667bee38/contracts/implementations/RepricingOracle.sol#L129) `RepricingOracle`에 의해서만 호출되어야 하기 때문입니다.

**Swell:** 커밋 [1fca965](https://github.com/SwellNetwork/v3-contracts-lst/commit/1fca965019facc4dcc79c35bfc45c8a711043196), [3f85df3](https://github.com/SwellNetwork/v3-contracts-lst/commit/3f85df3ba0e91b26e4234b15ad94f492fa6d46ec)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `RepricingOracle::handleReprice`, `_assertRepricingSnapshotValidity` 및 `_repricingPeriodDeltas` 리팩터링

**설명:** `RepricingOracle::_assertRepricingSnapshotValidity` 및 `_repricingPeriodDeltas`에는 `upgradeableRepriceSnapshot.meta.blockNumber == 0`인지 여부에 따라 이전 스냅샷을 사용할지 여부에 대한 많은 로직이 있습니다.

아이디어가 업그레이드 후 처음 리프라이싱이 발생할 때 실행 경로가 `useOldSnapshot = true`이지만 그 이후에는 매번 `useOldSnapshot = false`가 되는 것이라면, 한 번만 실행될 첫 번째 실행을 위한 함수를 만든 다음, 이후의 모든 일반적인 경우를 위한 함수를 갖는 것이 더 합리적일 수 있습니다. 이렇게 하면 추가 가스 비용을 피하고 첫 번째 호출 특수 케이스 이후의 모든 미래 일반적인 경우에 대한 코드도 단순화할 수 있습니다.

또한 `handleReprice`가 스냅샷 구조체를 로드하고, `upgradeableRepriceSnapshot.meta.blockNumber`를 캐시하고, `useOldSnapshot`을 한 번 계산한 다음 이를 `_assertRepricingSnapshotValidity` 및 `_repricingPeriodDeltas`에 입력으로 전달하도록 하여 가스 비용을 줄일 수 있습니다. 예:

```solidity

  function handleReprice(
    UpgradeableRepriceSnapshot calldata _snapshot
  ) internal {
    // only call getSnapshotStruct() once
    UpgradeableRepriceSnapshot
      storage upgradeableRepriceSnapshot = getSnapshotStruct();

    // only calculate these once and pass them as required
    uint256 ursMetaBlockNumber = upgradeableRepriceSnapshot.meta.blockNumber;
    bool useOldSnapshot = ursMetaBlockNumber == 0;

    // validation
    _assertRepricingSnapshotValidity(_snapshot, ursMetaBlockNumber, useOldSnapshot);

    _repricingPeriodDeltas(
          reserveAssets,
          _snapshot.state,
          _snapshot.withdrawState,
          upgradeableRepriceSnapshot,
          useOldSnapshot
        );

    // delete the call to getSnapshotStruct() near the end of handleReprice()
```


**Swell:** 인지함. 이전 스냅샷이 더 이상 관련이 없게 되는 향후 업그레이드에서 해결될 것입니다. Swell은 그동안 초과 가스 비용을 계속 지불할 것입니다.


### 변하지 않는 입금 금액에 상수 사용

**설명:** `DepositManager::setupValidators`에서 절대 변경되지 않는 이 변수를 선언하고 나중에 읽기 위해 가스를 지불할 필요가 없습니다:
```solidity
uint256 depositAmount = 32 ether;
```

대신 상수를 정의하십시오:
```solidity
uint256 private constant DEPOSIT_AMOUNT = 32 ether;
```
그리고 그 상수를 대신 사용하십시오.

**Swell:** 인지함.

\clearpage
