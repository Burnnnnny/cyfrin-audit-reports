**수석 감사자**

[Immeas](https://twitter.com/0ximmeas)

[Hans](https://twitter.com/hansfriese)


---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### 경품 업데이트 시 검증 부족으로 무작위성 이행 시 `lotAmount` 언더플로우 발생 가능

**설명:** [`SpinGame::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/0af327319636960e9683897c5935aa1a78d1ded5/contracts/src/Spin.sol#L595-L603)에는 현재 값을 검증하지 않고 `prize.lotAmount`를 감소시키는 `unchecked` 블록이 있습니다:

```solidity
/// Should never underflow due to earlier check.
unchecked {
    prize.lotAmount -= 1;
}

if (prize.lotAmount == 0) {
    totalProbabilities -= prizeProbability;
    prize.probability = 0;
}
```

그러나 "이전의 확인"으로 인해 언더플로우가 방지된다는 주석은 커밋 [`db6ae3d`](https://github.com/Consensys/linea-hub/commit/db6ae3d7a68497ac1077297ab0a16b9c13bd9a73#diff-ca99e3568f81fcb74ee275bb22d15e8216159decd2a4cc2e7c4572f639bb27c3R536)에서 해당 확인이 제거되었으므로 더 이상 유효하지 않으며, 이 가정은 틀렸습니다. 결과적으로 `lotAmount`가 0인 경품이 추가되면, 확인되지 않은 감소 연산은 `type(uint32).max`로 언더플로우됩니다.

**영향:** 이 언더플로우로 인해 경품의 `lotAmount`가 매우 큰 것처럼 보이게 되어, 사용자가 의도된 수량을 훨씬 초과하여 해당 경품을 반복적으로 당첨되고 청구할 수 있게 됩니다. 이는 관리자의 잘못된 구성(예: `lotAmount = 0` 설정)을 필요로 하지만, 이러한 오류는 실제로 발생할 수 있으며 눈에 띄지 않을 수 있습니다.


**PoC (Proof of Concept):** 다음 테스트는 `lotAmount = 0`으로 경품이 등록될 때 `lotAmount`의 언더플로우를 보여줍니다:
```solidity
function testFulfillRandomnessWith0LotAmount() external {
    MockERC20 token = new MockERC20("Test Token", "TST");
    ISpinGame.Prize[] memory prizesToUpdate = new ISpinGame.Prize[](1);
    uint256[] memory empty = new uint256[](0);

    prizesToUpdate[0] = ISpinGame.Prize({
        tokenAddress: address(token),
        amount: 500 * 1e18,
        lotAmount: 0,
        probability: 1e8,
        availableERC721Ids: empty
    });

    vm.prank(controller);
    spinGame.updatePrizes(prizesToUpdate);

    uint64 nonce = 1;
    uint64 expirationTimestamp = uint64(block.timestamp + 1);
    uint64 boost = 1e8;

    ISpinGame.ParticipationRequest memory request = ISpinGame.ParticipationRequest({
        user: user,
        expirationTimestamp: expirationTimestamp,
        nonce: nonce,
        boost: boost
    });

    bytes32 messageHash = spinGame.hashParticipationExt(request);
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(signer, messageHash);
    ISpinGame.Signature memory signature = ISpinGame.Signature(r, s, v);

    assertFalse(spinGame.nonces(user, nonce));

    vm.prank(user);
    uint256 requestId = spinGame.participate(nonce, expirationTimestamp, boost, signature);
    assertTrue(spinGame.nonces(user, nonce));

    bytes memory extraData = new bytes(0);
    bytes memory data = abi.encode(requestId, extraData);

    uint256 round = 15608646;
    bytes memory dataWithRound = abi.encode(round, data);

    vm.prank(vrfOperator);
    spinGame.fulfillRandomness(2, dataWithRound);

    uint32[] memory prizeIds = new uint32[](1);
    prizeIds[0] = 0;
    uint256[] memory amounts = spinGame.getUserPrizesWon(user, prizeIds);
    assertEq(amounts[0], 1);
    assertEq(spinGame.hasWonPrize(user, 0), true);

    ISpinGame.Prize memory prize = spinGame.getPrize(0);
    assertEq(prize.lotAmount,type(uint32).max);
}
```

**권장되는 완화 방법:** 이 언더플로우를 방지하기 위해 `_addPrizes`에서 새 경품을 등록할 때 `lotAmount > 0`인지 확인하십시오:

```diff
    uint32 lotAmount = _prizes[i].lotAmount;

+   if (lotAmount == 0) {
+       revert InvalidLotAmount();
+   }

    if (prizeAmount == 0) {
        if (erc721IdsLen != lotAmount) {
            revert MismatchERC721PrizeAmount(erc721IdsLen, lotAmount);
        }
    } else {
        if (erc721IdsLen != 0) {
            revert ERC20PrizeWrongParam();
        }
    }
```

이는 배포될 경품이 0이 아닌 수량을 갖도록 보장하여 이행 중 언더플로우 가능성을 제거합니다.

**Linea:** 커밋 [`02d6d57`](https://github.com/Consensys/linea-hub/pull/551/commits/02d6d576cced6a6926ae12e2f187d8ef2fee771e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 새 경품이 추가될 때 `lotAmount`가 0이 아님을 확인합니다.


### 요청별 데이터 추적 부족으로 인해 비동기 VRF 요청 이행 시 오래되거나 잘못된 매개변수 사용

**설명:** `SpinGame` 계약은 비동기 VRF 요청을 처리하는 방식에 중대한 구조적 결함이 있습니다. 계약은 사용자 부스트(boost) 값과 경품 구성을 특정 요청에 연결하는 대신 전역적으로 저장하므로, 경품 구조가 변경되거나 사용자가 여러 참여 요청을 할 때 일관성 없는 게임 동작이 발생합니다.

두 가지 주요 문제가 있습니다:

첫째, 경품 구성에 버전 관리(versioning)가 없습니다. `SpinGame::updatePrizes` 함수는 기존 `prizeIds`를 삭제하고 새 경품을 추가하기 전에 `totalProbabilities`를 0으로 설정하여 경품 구조를 완전히 재설정합니다. 사용자가 `SpinGame::participate`를 호출하면 현재 경품 구조를 기반으로 서명을 받습니다. 그러나 요청과 VRF 이행 사이에 `SpinGame::updatePrizes`가 호출되면 `SpinGame::_fulfillRandomness` 함수는 사용자가 참여할 때 예상했던 구조 대신 업데이트된 경품 구조를 사용하게 됩니다. 이는 사용자가 참여했을 때 사용할 수 있었던 것과는 완전히 다른 경품을 받을 수 있음을 의미합니다.

둘째, 요청별 데이터가 제대로 추적되지 않습니다. `SpinGame::participate` 함수는 `userToBoost[user]`에 사용자의 부스트 값을 저장하지만, 첫 번째 요청이 이행되기 전에 동일한 사용자가 다른 부스트로 다시 참여하면 이 매핑이 덮어쓰여집니다. `SpinGame::_fulfillRandomness`가 실행될 때 `uint64 userBoost = userToBoost[user]`를 사용하여 부스트를 검색하는데, 이는 원래 요청의 부스트 값이 아닐 수 있습니다. 이는 150% 부스트로 참여한 사용자가 첫 번째 VRF 콜백 전에 더 높은 부스트로 두 번째 참여를 하면 500% 부스트로 요청이 이행될 수 있는 시나리오를 만듭니다.

계약은 `requestIdToUser` 및 `requestIdTimestamp` 매핑을 통해서만 최소한의 요청 데이터를 추적하며, 요청 시점의 특정 부스트 값 및 경품 구조 버전을 포함하여 적절한 요청 이행에 필요한 전체 컨텍스트를 캡처하지 못합니다.

**영향:** 사용자는 참여 시 예상했던 것과 다른 경품이나 당첨 확률을 받을 수 있으며, 이는 불공정한 게임 결과와 잠재적인 자금 손실로 이어질 수 있습니다.

```
// Scenario 1: Prize structure changes between request and fulfillment
1. User calls participate() when Prize A (50% chance) and Prize B (30% chance) are available
2. Controller calls updatePrizes() with Prize C (60% chance) and Prize D (20% chance)
3. VRF fulfills the request using new prize structure with C and D instead of A and B

// Scenario 2: Multiple participations with different boosts
1. User calls participate() with 150% boost, gets requestId1
2. User calls participate() with 500% boost, gets requestId2
3. VRF fulfills requestId1 but uses 500% boost instead of 150%
```

**권장되는 완화 방법:** 경품 버전 관리를 포함한 요청별 데이터 추적을 구현하십시오:

```diff
+ uint256 public prizeVersion;
+ mapping(uint256 requestId => uint64 boost) public requestIdToBoost;
+ mapping(uint256 requestId => uint256 prizeVersion) public requestIdToPrizeVersion;

function updatePrizes(Prize[] calldata _prizes) external onlyController {
    delete prizeIds;
    totalProbabilities = 0;
+   prizeVersion++;
    _addPrizes(_prizes);
}

function participate(
    uint64 _nonce,
    uint256 _expirationTimestamp,
    uint64 _boost,
    Signature calldata _signature
) external returns (uint256) {
    // ... existing validation ...

-   userToBoost[user] = _boost;
    uint256 requestId = _requestRandomness("");

    requestIdToUser[requestId] = user;
    requestIdTimestamp[requestId] = block.timestamp;
+   requestIdToBoost[requestId] = _boost;
+   requestIdToPrizeVersion[requestId] = prizeVersion;

    // ... rest of function ...
}

function _fulfillRandomness(
    uint256 _randomness,
    uint256 _requestId,
    bytes memory
) internal override {
    address user = requestIdToUser[_requestId];
    if (user == address(0)) {
        revert InvalidRequestId(_requestId);
    }

+   // Verify prize version hasn't changed
+   if (requestIdToPrizeVersion[_requestId] != prizeVersion) {
+       revert PrizeVersionMismatch();
+   }

-   uint64 userBoost = userToBoost[user];
+   uint64 userBoost = requestIdToBoost[_requestId];

    // ... rest of fulfillment logic ...

+   delete requestIdToBoost[_requestId];
+   delete requestIdToPrizeVersion[_requestId];
}
```

**Linea:** 인지하였습니다. 내부적으로 L2 문제에 대해 논의했습니다. 우리는 경품이 일반적으로 새로운 할당으로 동일할 것이기 때문에 이를 수정하지 않기로 결정했습니다. 다른 경품의 경우, 사용자가 다른 경품을 받을 수 있는 동작에 동의합니다.

\clearpage
## 정보성 (Informational)


### 증분 업데이트를 위해 설계되었으나 전체 경품 재설정에만 사용되는 복잡한 `_addPrizes` 함수

**설명:** `SpinGame::_addPrizes` 함수는 원래 기존 경품을 보존하면서 기존 경품 풀에 새 경품을 증분적으로 추가하도록 설계되었습니다. 그러나 이 함수는 `SpinGame::updatePrizes`에 의해서만 호출되며, 이 함수는 먼저 `delete prizeIds`를 호출하고 `totalProbabilities = 0`을 설정하여 경품 상태를 완전히 재설정합니다. 이는 `_addPrizes`의 보존 로직을 불필요하게 만들고 몇 가지 비효율성을 초래합니다:

1. 함수는 재설정 후 항상 비어 있는 `existingArray`로 기존 `prizeIds` 배열을 복사합니다.
2. `existingArrayLen`이 항상 0인 `len + existingArrayLen` 크기의 새 배열을 생성합니다.
3. 존재하지 않는 경품 ID를 복사하기 위해 빈 기존 배열을 루프합니다.

현재 구현은 전체 경품 풀을 효과적으로 대체하지만, 더 이상 의미 있는 목적을 수행하지 않는 [삭제된](https://github.com/Consensys/linea-hub/commit/21b48096edbd436311bb4ef688d1b5367c1121ff) `SpinGame::addBatchPrizes` 함수에서 남은 증분 추가 로직을 유지하고 있습니다.

**영향:** 과도하게 복잡한 구현은 증분 추가 로직이 사용되지 않으므로 기능적 이점을 제공하지 않으면서 가스 비용과 코드 복잡성을 증가시킵니다.

**권장되는 완화 방법:** `updatePrizes`는 항상 전체 재설정을 수행하므로 보존 로직을 제거하도록 `_addPrizes`를 단순화하고 이름을 `_setPrizes`로 변경할 수 있습니다.

**Linea:** [PR#558](https://github.com/Consensys/linea-hub/pull/558), 커밋 [`0d8611d`](https://github.com/Consensys/linea-hub/pull/558/commits/0d8611dd06a8c51fe34d1379a2ff7d0bfee1e503), [`f9b1bd2`](https://github.com/Consensys/linea-hub/pull/558/commits/f9b1bd2428dc2d592b182ad99cbf39e18deb6b41), [`e115331`](https://github.com/Consensys/linea-hub/pull/558/commits/e1153312dfdf4851feaf645c2f11efebd5381188), [`2d619e1`](https://github.com/Consensys/linea-hub/pull/558/commits/2d619e104839a017d25e00ba667d340c455f227b), [`7920d00`](https://github.com/Consensys/linea-hub/pull/558/commits/7920d00601ae2c3b31897e62b7faee20bebd271f)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.
* `updatePrizes`의 이름이 `setPrizes`로 변경되었습니다.
* `existingArray` 로직이 제거되었으며 이전 배열에 대한 반복이 없습니다.
* `prizeIds` 삭제가 제거되었으며 대신 `prizeIds`가 메모리 배열 `newPrizeIds`로 덮어쓰여집니다.


### 코드베이스 전반의 잘못되고 오해의 소지가 있는 주석이 혼란을 야기함

**설명:** SpinGame 계약 및 ISpin 인터페이스에는 코드가 문서화하는 기능을 정확하게 설명하지 않는 잘못되고 오해의 소지가 있으며 일관성이 없는 여러 주석이 포함되어 있습니다. 이러한 주석은 개발자, 감사자 및 코드베이스의 향후 유지 관리자에게 혼란을 줍니다.

1. **`SpinGame::_operator()`의 458행**: 주석은 "전용 _msgSender()의 주소를 반환합니다"라고 되어 있는데 이는 틀렸습니다. 함수는 `_msgSender()`와 관련된 어떤 것이 아니라 `vrfOperator` 주소를 반환합니다. 이 함수는 VRF 이행 콜백을 호출할 권한이 있는 주소를 지정합니다. 이 주석은 신중한 고려 없이 `GelatoVRFConsumerBase`에서 복사된 것으로 추측됩니다.

2. **`SpinGame::getPrize()`의 123행**: `@param _prizeId` 주석은 "청구할 경품의 ID"라고 되어 있는데, 이는 경품 정보를 반환하는 뷰(view) 함수이지 청구 함수가 아니므로 오해의 소지가 있습니다.

3. **`SpinGame::getUserPrizesWon()`의 132행**: `@return` 주석은 "당첨된 경품의 금액"이라고 되어 있지만, 실제로는 토큰 금액이 아니라 각 경품이 당첨된 횟수를 반환합니다.

4. **59행**: 매핑 주석은 "(userAddress => prizeId => amountOfPrizeIdWon)"이라고 되어 있지만 실제로는 경품이 당첨된 횟수를 추적할 때 "amount(금액/양)"라는 용어를 사용합니다.

5. **32행**: "마지막으로 사용된 prizeId"라는 주석은 오해의 소지가 있습니다. `latestPrizeId`는 마지막으로 사용된 것이 아니라 다음 사용 가능한 경품 ID를 나타내기 때문입니다.

6. **`SpinGame::getPrizesAmount()`의 157행**: `@return` 주석은 "PrizesAmount 사용 가능한 경품의 양"이라고 되어 있지만 변수 이름은 금액이 아니라 개수임을 암시합니다. 이는 수량을 나타내는지 가치를 나타내는지에 대한 모호성을 만듭니다.

7. **290, 300, 309행**: 관리자 출금 함수는 `@dev Controller function`으로 레이블이 지정되어 있지만 실제로는 `CONTROLLER_ROLE`이 아니라 `DEFAULT_ADMIN_ROLE`이 필요합니다.

8. **465행**: `@param _prizes` 주석은 "업데이트할 경품"이라고 되어 있어, 이 함수가 기존 경품을 업데이트하는 것이 아니라 모든 기존 경품을 완전히 대체한다는 것을 명확하게 나타내지 않습니다.

**영향:** 잘못된 문서는 개발자가 실제 코드 동작을 분석하는 대신 오해의 소지가 있는 주석에 의존할 수 있으므로 구현 오류, 보안 취약점 및 개발 시간 낭비로 이어질 수 있습니다.

**권장되는 완화 방법:** 실제 기능을 정확하게 반영하도록 오해의 소지가 있는 모든 주석을 업데이트하십시오:

```diff
- /// @notice Returns the address of the dedicated _msgSender().
+ /// @notice Returns the address of the VRF operator authorized to fulfill randomness requests.

- /// @param _prizeId Id of the prize to claim.
+ /// @param _prizeId Id of the prize to retrieve information for.

- /// @return Amounts of prize won.
+ /// @return Number of times each prize was won by the user.

- /// @notice Last used prizeId.
+ /// @notice Next available prize ID to be assigned.

- /// @dev Controller function to withdraw ERC20 tokens from the contract.
+ /// @dev Admin function to withdraw ERC20 tokens from the contract.

- /// @param _prizes Prizes to update.
+ /// @param _prizes Prizes to replace all existing prizes with.
```


**Linea:** 커밋 [`9f9d9fd`](https://github.com/Consensys/linea-hub/pull/554/commits/9f9d9fd76d2672f572e31079b5811bf6f0f48eed) 및 [`b407331`](https://github.com/Consensys/linea-hub/pull/554/commits/b407331d5dee4fa6e8d70855230d4f26ca0a9b11)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 주석이 수정되었습니다.


### 실패한 스핀에 대해 오래된 요청 ID 매핑이 지속됨

**설명:** `SpinGame`의 이전 버전과 다른 한 가지 변경 사항은 "제대로 된" 경품에 당첨되지 않은 사용자를 위한 위로 상품 역할을 했던 "LuckyNFT"가 제거된 것입니다.

이 변경의 일환으로 요청 매핑(`requestIdToUser`)의 삭제가 이동되어 이제 [`SpinGame::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/0af327319636960e9683897c5935aa1a78d1ded5/contracts/src/Spin.sol#L604-L612)에 표시된 대로 경품 당첨 경로에서만 **발생**합니다:

```solidity
            userToPrizesWon[user][selectedPrizeId] += 1;
            delete requestIdToUser[_requestId];
            emit PrizeWon(user, selectedPrizeId);

            return;
        }
    }
    emit NoPrizeWon(user);
}
```

그러나 사용자가 이기든 지든 관계없이 `requestIdToUser` 매핑을 지우는 것이 좋은 관리 방식으로 간주됩니다. 이는 일관된 상태 정리를 보장하고 잠재적으로 오래된 항목이 지속되는 것을 방지합니다.

다음과 같이 무조건적으로 요청 매핑을 지우는 것을 고려하십시오:
```diff
function _fulfillRandomness(
    uint256 _randomness,
    uint256 _requestId,
    bytes memory
) internal override {
    address user = requestIdToUser[_requestId];
    if (user == address(0)) {
        revert InvalidRequestId(_requestId);
    }
+   delete requestIdToUser[_requestId];

    ...

            userToPrizesWon[user][selectedPrizeId] += 1;
-           delete requestIdToUser[_requestId];
            emit PrizeWon(user, selectedPrizeId);


            return;
        }
    }
    emit NoPrizeWon(user);
}
```

**Linea:** 커밋 [`9f9d9fd`](https://github.com/Consensys/linea-hub/pull/554/commits/9f9d9fd76d2672f572e31079b5811bf6f0f48eed)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `_fulfillRandomness`의 시작 부분에서 `requestIdToUser`가 삭제됩니다.


### 생성자에서 `_disableInitializers()` 누락

**설명:** `SpinGame` 계약은 업그레이드 가능하지만 생성자에서 `_disableInitializers()`를 호출하지 **않습니다**. 업그레이드 가능한 계약 패턴에서 이 호출은 구현(로직) 계약이 직접 초기화되는 것을 방지하기 위한 모범 사례입니다.

이는 프록시의 동작에 영향을 미치지 않지만, 특히 블록 탐색기와 같이 프록시와 구현 계약이 모두 보이는 환경에서 구현 계약이 실수로 또는 악의적으로 단독으로 사용되는 것을 방지하는 데 도움이 됩니다.

`SpinGame` 계약의 생성자에 다음 줄을 추가하는 것을 고려하십시오:

```solidity
constructor(address _trustedForwarderAddress) ERC2771ContextUpgradeable(_trustedForwarderAddress) {
    _disableInitializers();
}
```

이는 구현 계약이 독립적으로 초기화될 수 없도록 보장합니다.

**Linea:** 커밋 [`9f9d9fd`](https://github.com/Consensys/linea-hub/pull/554/commits/9f9d9fd76d2672f572e31079b5811bf6f0f48eed)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 생성자가 이제 `_disableInitializers`를 호출합니다.


### `GelatoVRFConsumerBase`는 업그레이드에 안전하지 않음

**설명:** `SpinGame` 계약이 업그레이드 가능하도록 변경되었습니다. 그러나 계약은 향후 스토리지 충돌을 방지하기 위한 예약된 스토리지 갭(`uint256[x] private __gap`)이 없는 업그레이드에 안전하지 않은 `GelatoVRFConsumerBase`를 상속합니다. `GelatoVRFConsumerBase` 계약이 업스트림에서 수정되는 경우(예: 상태 변수 추가), 업그레이드 중 스토리지 레이아웃 손상으로 이어질 수 있는 위험이 있습니다.

`GelatoVRFConsumerBase`는 감사된 코드베이스의 제어 범위를 벗어난 외부 종속성이므로 이 위험은 해당 계약 내에서 직접 해결할 수 없습니다.

그러나 위험을 완화하기 위해 `GelatoVRFConsumerBase`의 잠재적인 향후 변경 사항을 설명하기 위해 스토리지 버퍼를 추가하는 것을 고려하십시오. 이는 다음과 같이 수행할 수 있습니다:

1. `SpinGame` 자체에 `_gap` 추가:

   ```solidity
   uint256[x] private __gelatoBuffer;
   ```

2. 대안으로, 스토리지 공간을 예약하기 위해서만 존재하는 중간 "버퍼" 계약을 상속 체인에 삽입:

   ```solidity
   contract GelatoVRFGap {
       uint256[x] private __gap;
   }

   contract SpinGame is ..., GelatoVRFConsumerBase, GelatoVRFGap
   ```

이는 `GelatoVRFConsumerBase`가 나중에 스토리지를 추가하더라도 `SpinGame`의 중요한 스토리지 슬롯을 덮어쓰지 않도록 보장합니다.

**Linea:** 커밋 [`9f9d9fd`](https://github.com/Consensys/linea-hub/pull/554/commits/9f9d9fd76d2672f572e31079b5811bf6f0f48eed)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 계약의 첫 번째 스토리지 슬롯은 이제 `uint256[50] private __gelatoBuffer`입니다.


### `latestPrizeId` 이름이 오해의 소지가 있음

**설명:** 이 감사의 변경 사항의 일환으로, `latestPrizeId` 변수는 이전 버전과 같이 가장 최근에 사용된 ID를 추적하는 대신, 이제 다음 사용 가능한 경품 ID에 대한 카운터로 기능합니다. `latestPrizeId`는 더 이상 실제 동작을 반영하지 않으므로 이는 명명 불일치를 만듭니다.

`_addPrizes` 함수는 심지어 이를 `nextPrizeId`로 캐시하는데, 이는 더 정확한 설명입니다. 이름을 `nextPrizeId`로 변경하는 것을 고려하십시오.

**Linea:** 커밋 [`8266a91`](https://github.com/Consensys/linea-hub/pull/555/commits/8266a9130db2e3ad9d66422c7c1c374dc806bada) 및 [`dd8de7b`](https://github.com/Consensys/linea-hub/pull/555/commits/dd8de7bd9626abca6c01bc24832551c8f50ef0a9)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 필드 이름이 `nextPrizeId`로 변경되었으며 `_addPrizes`의 스택 변수 이름이 `currentPrizeId`로 변경되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### `SpinGame::participate`에서 `signer` 캐시

**설명:** `SpinGame::participate`에서 `signer` 상태 변수는 여러 번 액세스됩니다:

```solidity
address recoveredSigner = ECDSA.recover(...);
if (recoveredSigner != signer || signer == address(0)) {
    revert SignerNotAllowed(recoveredSigner);
}
```

대조적으로, `SpinGame::claimPrize`는 비교 전에 `signer`를 로컬 변수(`cachedSigner`)에 캐시합니다. 이는 로컬 메모리 액세스보다 비용이 많이 드는 중복 스토리지 읽기를 방지합니다.

`SpinGame::participate`의 관련 확인 시작 부분에서도 `signer`를 로컬 변수에 캐시하는 것을 고려하십시오.

**Linea:** 커밋 [`0290123`](https://github.com/Consensys/linea-hub/pull/557/commits/02901233dbe9a184b80bffb67bf5d489bc015a10)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `signer`가 캐시되고 캐시된 값이 비교에 사용됩니다.


### 처음 사용하기 전에 `prize.probability` 캐시

**설명:** `SpinGame::_fulfillRandomness()`에서 `prize.probability`는 `if` 문에서 처음 읽힌 다음 즉시 캐시됩니다:

```solidity
if (prize.probability == 0) {
    continue;
}
uint64 prizeProbability = prize.probability;
```

이로 인해 캐싱 전에 불필요한 초기 스토리지 읽기가 발생합니다. `if` 문 위에서 캐시하는 것을 고려하십시오.

**Linea:** 커밋 [`0290123`](https://github.com/Consensys/linea-hub/pull/557/commits/02901233dbe9a184b80bffb67bf5d489bc015a10)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `prize.probability` 캐싱이 `if` 위로 이동되었으며 캐시된 값이 비교에 사용됩니다.


### `SpinGame::_transferPrize`에서 `prize.amount` 캐시

**설명:** `SpinGame::_transferPrize`에서 ERC20 또는 네이티브 토큰 경품을 이체할 때 `prize.amount` 필드는 세 번 읽힙니다. 한 번은 `if` 확인용, 한 번은 계약 잔고와 비교용, 그리고 이체 실행 시 다시 읽힙니다:

```solidity
if (prize.amount > 0) {
    ...
    if (contractBalance < prize.amount) {
        revert PrizeAmountExceedsBalance(..., prize.amount, contractBalance);
    }
    ...
    token.safeTransfer(_winner, prize.amount);
}
```

이로 인해 동일한 스토리지 슬롯을 세 번 별도로 읽게 됩니다. 실행 중에 값이 변경되지 않으므로 한 번 캐시할 수 있습니다.

첫 번째 if의 헤드에서 `prize.amount`를 캐시하는 것을 고려하십시오:

```solidity
uint256 amount = prize.amount;
if (amount > 0) {
    ...
    if (contractBalance < amount) {
        revert PrizeAmountExceedsBalance(..., amount, contractBalance);
    }
    ...
    token.safeTransfer(_winner, amount);
}
```

**Linea:** 커밋 [`0290123`](https://github.com/Consensys/linea-hub/pull/557/commits/02901233dbe9a184b80bffb67bf5d489bc015a10)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `prize.amount`는 이제 캐시되고 캐시된 값이 사용됩니다.

\clearpage
