**수석 감사자**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

**보조 감사자**



---

# 발견 사항 (Findings)
## 중급 위험 (Medium Risk)


### `RiskOracle::_processUpdate`에서의 잘못된 상태 업데이트

**설명:** `RiskParameterUpdate` 구조체 내에는 매개변수의 이전 값을 저장하기 위한 [`bytes previousValue`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L15) 필드가 있습니다. 이 상태 업데이트는 `updatedById` 매핑에서 [얻은 값](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L155)으로 [`RiskOracle::_processUpdate`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L156-L158) 내에서 수행됩니다. 그러나 다른 업데이트 유형 및 시장에 대한 매개변수 간의 구분이 없으므로, 여러 업데이트 유형/시장이 있을 때 이 상태 업데이트는 압도적인 확률로 부정확할 것입니다. 따라서 이 계약의 소비자는 예상과 크게 다를 수 있는 이전 값을 가진 권장 사항을 받게 되며, 실제 변경을 나타내지 않는 델타를 기반으로 위험 매개변수 업데이트를 실행할 수도 있습니다.

**파급력:** 주어진 업데이트에 대한 `previousValue` 상태가 부정확할 가능성이 매우 높으며, 이로 인해 소비자가 부정확한 과거 데이터를 기반으로 위험 매개변수 업데이트를 수행할 수 있습니다.

**개념 증명 (PoC):** 다음 테스트는 이 발견 사항을 입증하기 위해 작성되었으며 이후 이 참여 기간 동안 리포지토리에 추가되었습니다.

```solidity
function test_PreviousValueIsCorrectForSpecificMarketAndType() public {
    bytes memory market1 = abi.encodePacked("market1");
    bytes memory market2 = abi.encodePacked("market2");
    bytes memory newValue1 = abi.encodePacked("value1");
    bytes memory newValue2 = abi.encodePacked("value2");
    bytes memory newValue3 = abi.encodePacked("value3");
    bytes memory newValue4 = abi.encodePacked("value4");
    string memory updateType = initialUpdateTypes[0];

    vm.startPrank(AUTHORIZED_SENDER);

    // Publish first update for market1 and type1
    riskOracle.publishRiskParameterUpdate(
        "ref1", newValue1, updateType, market1, abi.encodePacked("additionalData1")
    );

    // Publish second update for market1 and type1
    riskOracle.publishRiskParameterUpdate(
        "ref2", newValue2, updateType, market1, abi.encodePacked("additionalData2")
    );

    // Publish first update for market2 and type1
    riskOracle.publishRiskParameterUpdate(
        "ref3", newValue3, updateType, market2, abi.encodePacked("additionalData3")
    );

    // Publish first update for market1 and type1
    riskOracle.publishRiskParameterUpdate(
        "ref4", newValue4, updateType, market1, abi.encodePacked("additionalData4")
    );

    vm.stopPrank();

    // Fetch the latest update for market1 and type1
    RiskOracle.RiskParameterUpdate memory latestUpdateMarket1Type1 =
        riskOracle.getLatestUpdateByParameterAndMarket(updateType, market1);
    assertEq(latestUpdateMarket1Type1.previousValue, newValue2);

    // Fetch the latest update for market2 and type1
    RiskOracle.RiskParameterUpdate memory latestUpdateMarket2Type1 =
        riskOracle.getLatestUpdateByParameterAndMarket(updateType, market2);
    assertEq(latestUpdateMarket2Type1.previousValue, bytes(""));
}
```

**권장 완화 조치:** [`latestUpdateIdByMarketAndType`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L28) 매핑을 사용하여 올바른 과거 값을 검색하십시오.

**Chaos Labs:** 커밋 [d16a227](https://github.com/ChaosLabsInc/risk-oracle/commit/d16a2277f7fb0efee0053389492aa116543a2bf7)에서 수정되었습니다.

**Cyfrin:** 확인됨, 이전 업데이트 값은 이제 올바른 식별자에서 검색됩니다.

\clearpage
## 정보성 (Informational)


### `RiskOracle::addUpdateType`과 계약 생성자 간의 유효성 검사 비대칭

**설명:** `RiskOracle::addUpdateType` 내에 다음 [유효성 검사](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L92)가 존재합니다:
```solidity
require(!validUpdateTypes[newUpdateType], "Update type already exists.");
```
그러나 이 함수에는 `onlyOwner` 수정자가 적용되어 있으므로 유효성 검사가 엄격하게 필요하지는 않습니다. 이는 소유자가 계약을 배포할 때 호출되는 생성자 내에서 관찰할 수 있는데, 여기에는 그러한 유효성 검사가 없으며 중복이 오프체인에서 확인될 것이라고 가정합니다. 따라서 이 두 인스턴스 간에 비대칭이 존재하며, 유효성 검사를 완전히 제거하거나 두 코드 경로 모두에 존재하도록 하여 일관성을 유지하는 것이 좋습니다.

**Chaos Labs:** 커밋 [9f7375a](https://github.com/ChaosLabsInc/risk-oracle/commit/9f7375a8291deb04719ec4ddbfff1eb638db55e1)에서 생성자에 중복 검사를 추가했습니다.

**Cyfrin:** 확인됨, 중복 검사가 생성자에 추가되었습니다.


### 계약 소유자가 승인된 발신자(authorized sender)가 되는 것에 대한 제한 없음

**설명:** [`RiskOracle::addAuthorizedSender`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L67-L75)에 적용된 `onlyOwner` 수정자로 인해 계약 소유자만이 승인된 발신자를 추가할 수 있습니다. 현재 존재하는 유일한 유효성 검사는 이미 승인된 발신자를 추가하는 것을 방지하는 것이므로, 소유자가 자신을 승인된 발신자로 추가하는 것이 가능합니다. 예를 들어 두 역할 간의 책임 분리를 엄격하게 시행하기 위해 이것이 바람직하지 않은 경우 이 제한을 추가해야 합니다.

**Chaos Labs:** 계약 소유자가 승인된 발신자가 되는 것에 제한이 없음을 인지했습니다.

**Cyfrin:** 인지함.


### 중복된 유효성 검사를 공유 내부 함수로 이동 가능

**설명:** 현재 `RiskOracle::publishRiskParameterUpdate`와 `RiskOracle::publishBulkRiskParameterUpdates` 모두 본질적으로 동일한 유효성 검사를 포함하고 있습니다:
```solidity
// `RiskOracle::publishRiskParameterUpdate`:
require(validUpdateTypes[updateType], "Unauthorized update type.");

// `RiskOracle::publishBulkRiskParameterUpdates`:
require(validUpdateTypes[updateTypes[i]], "Unauthorized update type at index");
```
두 함수 모두 내부 `_processUpdate()` 함수를 호출하므로, 이 유효성 검사를 그곳에 배치하여 중복을 제거할 수 있습니다.

**Chaos Labs:** 커밋 [6cf09fb](https://github.com/ChaosLabsInc/risk-oracle/commit/6cf09fbe31a2050d04b60c79eddfa15f5cd5ca15)에서 수정되었습니다.

**Cyfrin:** 확인됨, 유효성 검사가 이제 공유 내부 함수에 존재합니다.


### 병렬 데이터 구조가 필요하지 않음

**설명:** 단조 증가하는 [`updateCounter`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L29) 상태 변수에 의해 주어진 키를 사용하여 [`updatesById`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L25) 매핑을 사용하는 것은 사실상 인덱스 이동 1(0은 유효하지 않은 업데이트 ID를 위해 예약됨)을 사용하여 [`updateHistory`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L25) 배열을 사용하는 것과 동일합니다. 현재 설계에서는 이러한 병렬 데이터 구조를 유지할 필요가 없으므로 업데이트 ID 형식이 향후 변경될 가능성이 없다면 `updateHistory` 배열을 선호하여 `updatesById` 매핑을 제거할 수 있습니다. 이 수정은 `getLatestUpdateByType()`, `getLatestUpdateByParameterAndMarket()`, 및 `getUpdateById()` 함수에서 추가적인 리팩터링이 필요합니다.

**Chaos Labs:** 커밋 [6cf09fb](https://github.com/ChaosLabsInc/risk-oracle/commit/6cf09fbe31a2050d04b60c79eddfa15f5cd5ca15)에서 수정되었습니다.

**Cyfrin:** 확인됨, `RiskParameterUpdate[] updateHistory`가 제거되었습니다.


### 도달할 수 없는 코드 제거 가능

**설명:** [`RiskOracle::_processUpdate`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L155) 내에서 3항 연산자의 `else` 분기는 `updateCounter`가 0으로 초기화되고 이 라인 이전에 증가되므로 도달할 수 없습니다:

```solidity
updateCounter++;
bytes memory previousValue = updateCounter > 0 ? updatesById[updateCounter - 1].newValue : bytes("");
```

따라서 이 변수 할당 로직은 매핑에서 읽는 것으로 단순화될 수 있습니다 (하지만 M-01에서 보고된 바와 같이 `updatedById` 매핑의 사용이 잘못되었음을 유의하십시오).

**Chaos Labs:** 커밋 [6cf09fb](https://github.com/ChaosLabsInc/risk-oracle/commit/6cf09fbe31a2050d04b60c79eddfa15f5cd5ca15)에서 수정되었습니다.

**Cyfrin:** 확인됨, 코드 경로가 제거되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 불필요한 초기화 제거 가능

**설명:** `RiskOracle`의 [생성자 내](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L64)에서 `updateCounter` 상태 변수의 초기화는 불필요하며 제거할 수 있습니다. 이 상태는 기본적으로 `0`이 되기 때문입니다.

**Chaos Labs:** 커밋 [9f7375a](https://github.com/ChaosLabsInc/risk-oracle/commit/9f7375a8291deb04719ec4ddbfff1eb638db55e1)에서 수정되었습니다.

**Cyfrin:** 확인됨, 초기화가 더 이상 존재하지 않습니다.


### 배열 길이 유효성 검사가 필요하지 않음

**설명:** [`RiskOracle::publishBulkRiskParameterUpdates`](https://github.com/ChaosLabsInc/risk-oracle/blob/9449219174e3ee7da9a13a5db7fb566836fb4986/src/RiskOracle.sol#L117-L142)는 현재 모든 입력 배열의 길이가 같은지 검증합니다.

```solidity
function publishBulkRiskParameterUpdates(
    string[] memory referenceIds,
    bytes[] memory newValues,
    string[] memory updateTypes,
    bytes[] memory markets,
    bytes[] memory additionalData
) external onlyAuthorized {
    require(
        referenceIds.length == newValues.length && newValues.length == updateTypes.length
            && updateTypes.length == markets.length && markets.length == additionalData.length,
        "Mismatch between argument array lengths."
    );
    for (uint256 i = 0; i < referenceIds.length; i++) {
        require(validUpdateTypes[updateTypes[i]], "Unauthorized update type at index");
        _processUpdate(referenceIds[i], newValues[i], updateTypes[i], markets[i], additionalData[i]);
    }
}
```

`referenceIds`에 대한 루프로 인해 이 유효성 검사는 제거할 수 있습니다. 길이 불일치는 경계를 벗어난 액세스로 인해 revert 되거나 `referenceIds` 배열의 길이를 초과하는 추가 요소가 무시되는 결과를 초래하기 때문입니다.

**Chaos Labs:** 커밋 [6cf09fb](https://github.com/ChaosLabsInc/risk-oracle/commit/6cf09fbe31a2050d04b60c79eddfa15f5cd5ca15)에서 수정되었습니다.

**Cyfrin:** 확인됨, 유효성 검사가 제거되었습니다.

\clearpage
