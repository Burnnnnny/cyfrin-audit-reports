**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[Farouk](https://twitter.com/Ubermensh3dot0)

**Assisting Auditors**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# Findings
## Medium Risk


### `receive()` 함수가 누락되어 네이티브 토큰 상금을 펀딩할 수 없음 (Native token prizes cannot be funded due to missing `receive()` function)

**설명:** SpinGame은 ERC721, ERC20 및 네이티브 토큰을 포함한 여러 상금 유형을 지원하며, 여기서 네이티브 토큰은 `prize.tokenAddress = address(0)`으로 표시됩니다.

상금을 성공적으로 청구할 수 있도록 보장하기 위해, 프로토콜 팀은 Spin 계약에 필요한 자산을 전송하여 계약에 충분한 토큰 잔액을 유지할 책임이 있습니다.

그러나 네이티브 토큰 상금과 관련하여 특별한 문제가 있습니다. Spin 계약에는 `receive()` 또는 `fallback()` 함수가 없으며, `payable` 함수도 없습니다. 즉, 팀이 표준 전송을 사용하여 네이티브 토큰으로 계약을 펀딩할 방법이 없으므로 사용자가 네이티브 토큰 상금을 성공적으로 청구할 수 없습니다.

**영향:** 계약에 네이티브 토큰을 입금할 메커니즘이 없기 때문에 네이티브 토큰 상금을 청구할 수 없습니다. 네이티브 토큰 잔액을 제공하는 유일한 방법은 자금을 Spin 계약으로 보내는 계약을 self-destruct하는 것과 같은 난해한 해결 방법을 포함합니다.

**개념 증명 (Proof of Concept):** `Spin.t.sol`에 다음 테스트를 추가하세요:
```solidity
function testTransferNativeToken() public {
    vm.deal(admin,1e18);

    vm.prank(admin);
    (bool success, ) = address(spinGame).call{value: 1e18}("");

    // transfer failed as there is no `receive` or `fallback` function
    assertFalse(success);
}
```

**권장 완화 방법:** 네이티브 토큰 입금을 허용하기 위해 계약에 `receive()` 함수를 추가하는 것을 고려하세요:

```solidity
receive() external payable {}
```

**Linea:** 커밋 [`d1ab4bd`](https://github.com/Consensys/linea-hub/commit/d1ab4bdbaac3639a36d66440b9e6da95771e4b34)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Low Risk


### 부스트된 확률 계산의 반올림 오류로 인해 보장된 승리가 실패할 수 있음 (Rounding errors in boosted probability calculation can cause guaranteed wins to fail)

**설명:** Linea SpinGame에는 특정 사용자의 승리 확률을 높일 수 있는 부스팅 기능이 포함되어 있습니다. 그러나 이 메커니즘은 부스트된 확률의 합이 100%보다 큰 값이 될 수 있으므로 사용자의 총 승리 확률이 100%를 초과할 가능성을 도입합니다. 이를 해결하기 위해 계약은 [`Spin::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L538-L557)에서 총 부스트된 확률을 정규화합니다:

```solidity
// Apply boost on the sum of totalProbabilities.
uint256 boostedTotalProbabilities = totalProbabilities * userBoost / BASE_POINT;

// If boostedTotalProbabilities exceeds 100% we have to increase the winning threshold so it stays in bound.
//
// Example:
//   PrizeA probability: 50%
//   PrizeB probability: 30%
//   User boost: 1.5x
//   boostedPrizeAProbability: 75%
//   boostedPrizeBProbability: 45%
//
//   We now have a total of 120% totalBoostedProbability so we need to increase winning threshold by boostedTotalProbabilities to BASE_POINT ratio.
//
//   winningThreshold = winningThreshold * 12_000 / 10_000
if (boostedTotalProbabilities > BASE_POINT) {
    winningThreshold =
        (winningThreshold * boostedTotalProbabilities) /
        BASE_POINT;
}
```

나중에 [`Spin::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L569-L589)에서 사용자가 승리했는지 확인할 때 각 상금 확률이 독립적으로 스케일링됩니다:

```solidity
// Apply boost on a single prize probability.
uint256 boostedPrizeProbability = prizeProbability * userBoost / BASE_POINT;

unchecked {
    cumulativeProbability += boostedPrizeProbability;
}

if (winningThreshold < cumulativeProbability) {
    selectedPrizeId = localPrizeIds[i];

    // ... win
    break;
}
```

문제는 확률 계산에서 발생합니다:

```solidity
uint256 boostedPrizeProbability = prize.probability +
    ((prize.probability * userBoost) / BASE_POINT);
```

이 계산으로 인해 최종 `cumulativeProbability`가 `boostedTotalProbabilities`보다 낮을 수 있으며, 이는 승리가 보장되어야 하는 사용자가 반올림 오류로 인해 여전히 패배할 수 있는 시나리오로 이어집니다.

**영향:** 이론적으로 100% 승리 확률을 가진 사용자가 여전히 패배할 수 있습니다. 이는 발생 가능성이 낮은 엣지 케이스이지만, 수학적으로 승리가 보장됨에도 불구하고 수치 정밀도 문제로 인해 상금을 받지 못하는 불운한 사용자에게는 매우 문제가 될 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 예를 고려하세요:

- 각각 30%의 승리 확률을 가진 3개의 상금이 있습니다.
- 사용자는 133% 확률 부스트를 받습니다.

부스트된 확률 계산:

```solidity
boostedTotalProbabilities = 0.9e8*133_333_333/1e8 = 119_999_999
boostedPrizeProbability = 0.3e8*133_333_333/1e8 = 39_999_999
```
그리고
```
3*39_999_999 = 119_999_997
```

최악의 경우, 사용자는 다음을 얻을 수 있습니다:

```
winningThreshold = 99_999_999
```

임계값 조정 적용:

```solidity
if (boostedTotalProbabilities > BASE_POINT) {
    winningThreshold =
        (winningThreshold * boostedTotalProbabilities) /
        BASE_POINT;
}
```

결과는 다음과 같습니다:

```
winningThreshold = 99_999_999 * 119_999_999 / 1e8 = 119_999_997
```

`winningThreshold < cumulativeProbability`가 승리 조건이고:

```
119_999_997 < 119_999_997  // (false)
```

조건이 실패하므로 사용자는 승리가 보장되었음에도 불구하고 패배합니다. 이 문제는 확률 스케일링의 반올림 오류로 인해 발생합니다.

**권장 완화 방법:** 루프의 마지막 반복에서 `boostedTotalProbabilities >= BASE_POINT`인 경우 사용자에게 승리를 보장하도록 하는 것을 고려하세요:

```diff
- if (winningThreshold < cumulativeProbability) {
+ if (winningThreshold < cumulativeProbability ||
+     boostedTotalProbabilities >= BASE_POINT && i == prizeLen - 1 // last iteration and win is guaranteed
+ ) {
      selectedPrizeID = prizeIds[i];
```

이 변경은 매우 드문 경우에 목록의 마지막 상금에 약간 유리하지만, 이 상황이 이미 매우 드물다는 점을 감안할 때 이 트레이드오프는 합리적입니다.

**Linea:** 커밋 [`b32e038`](https://github.com/Consensys/linea-hub/commit/b32e038bba336d1ad6dddbdb972de8cafbbb2c1a)에서 수정됨.

**Cyfrin:** 검증됨.


### 사용자는 상금 수령을 지연시켜 더 높은 가치의 NFT를 선택할 수 있음 (Users can select higher-value NFTs by delaying prize claims)

**설명:** 사용자가 승리할 때, 계약은 [`Spin::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L576-L592)에서 특정 `prizeID`를 획득했다는 사실만 추적합니다:

```solidity
    if (winningThreshold < cumulativeProbability) {
        selectedPrizeId = localPrizeIds[i];

        // ...
        break;
    }
}

userToPrizesWon[user][selectedPrizeId] += 1;
```

그러나 사용자가 상금을 청구할 때, 상금이 NFT인 경우 계약은 [`Spin::_transferPrize`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L366-L368)에서 단순히 목록의 마지막 사용 가능한 NFT를 할당합니다:

```solidity
uint256 tokenId = prize.availableERC721Ids[
    prize.availableERC721Ids.length - 1
];
```

NFT는 대체 불가능하므로 각 `tokenId`는 고유한 항목을 나타냅니다. 즉, 승리한 사용자는 컬렉션에 가장 높은 가치의 NFT가 남을 때까지 상금 청구를 기다릴 수 있습니다. 이를 통해 그들은 최고의 사용 가능한 토큰을 전략적으로 청구할 수 있으며, 잠재적으로 즉시 상금을 청구하는 사용자를 희생시킬 수 있습니다.

**영향:** 사용자는 컬렉션에서 더 가치 있는 NFT를 확보하기 위해 청구를 지연시킬 수 있으며, 즉시 청구하는 다른 사용자는 자신도 모르게 더 낮은 가치의 토큰을 받을 수 있습니다. 이는 상금 배분 메커니즘을 이해하는 정보에 정통한 사용자에게 불공정한 이점을 줄 수 있습니다.

**권장 완화 방법:** 모든 잠재적 해결책에는 트레이드오프가 따르므로 완벽한 해결책은 없습니다. 한 가지 접근 방식은 `_fulfillRandomness`에서 승리 시점에 특정 NFT를 할당하는 것입니다. 그러나 이를 위해서는 각 사용자가 어떤 NFT를 획득했는지와 어떤 NFT가 남아 있는지 모두 추적해야 하므로 상태 복잡성과 가스 비용이 크게 증가합니다.

대신, 프로토콜은 이 문제를 인지하고 각 상금 카테고리 내의 NFT가 유사한 가치를 갖도록 해야 합니다. 컬렉션에 가치가 크게 다른 NFT가 포함된 경우 별도의 상금으로 추가하여 공정한 분배를 보장하고 사용자가 시스템을 악용하는 것을 방지해야 합니다.

**Linea:** 인지함. 더 높은 가치의 NFT는 별도의 상금으로 추가되어야 합니다.

**Cyfrin:** 인지함.


### 확률 오버플로우로 인해 `MaxProbabilityExceeded` 확인을 우회할 수 있음 (Probability overflow can bypass `MaxProbabilityExceeded` check)

**설명:** 새로운 상금을 추가할 때, 계약은 [`Spin::_addPrizes#L511-L513`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L511-L513)에서 총 확률이 100%를 초과하지 않도록 확인하는 검사를 포함합니다:

```solidity
if (totalProbabilities > BASE_POINT) {
    revert MaxProbabilityExceeded(totalProbabilities);
}
```

그러나 `totalProbabilities`가 계산되는 방식 때문에 이 확인을 우회할 수 있습니다. 확률의 누적은 다음 위치의 `unchecked` 블록에서 발생합니다:

- 개별 확률 값의 첫 번째 누적, [`Spin::_addPrizes#L503-L505`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L503-L505):

  ```solidity
  unchecked {
      totalProbIncrease += probability;
  }
  ```

- `totalProbabilities`의 최종 업데이트, [`Spin::_addPrizes#L508-L510`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L508-L510):

  ```solidity
  unchecked {
      totalProbabilities += totalProbIncrease;
  }
  ```

두 업데이트 모두 `unchecked` 블록 내에서 발생하기 때문에 매우 큰 확률 값은 오버플로우를 일으켜 `MaxProbabilityExceeded` 확인을 효과적으로 우회할 수 있습니다. 이를 통해 `totalProbabilities`가 래핑되어 `BASE_POINT`를 초과하더라도 유효한 것처럼 보일 수 있습니다.

**영향:** 이 함수는 신뢰할 수 있는 사용자(예: `CONTROLLER` 또는 `DEFAULT_ADMIN` 역할)만 호출할 수 있지만, 실수나 손상된 계정으로 인해 과도하게 큰 확률 값을 추가하여 이 문제가 발생할 수 있습니다. 이로 인해 오버플로우가 발생하여 `MaxProbabilityExceeded` 확인을 우회하고 상금 분배를 왜곡하여 게임의 무결성을 잠재적으로 손상시킬 수 있습니다.

**개념 증명 (Proof of Concept):** `Spin.t.sol`에 다음 테스트를 추가하세요:
```solidity
function testUpdateWithMoreThanMaxProba() external {
    MockERC721 nft = new MockERC721("Test NFT", "TNFT");
    nft.mint(address(spinGame), 10);
    nft.mint(address(spinGame), 21);

    ISpinGame.Prize[] memory prizesToUpdate = new ISpinGame.Prize[](2);
    uint256[] memory empty = new uint256[](0);

    uint256[] memory nftAvailable = new uint256[](2);
    nftAvailable[0] = 10;
    nftAvailable[1] = 21;

    prizesToUpdate[0] = ISpinGame.Prize({
        tokenAddress: address(nft),
        amount: 0,
        lotAmount: 2,
        probability: type(uint64).max - 1,
        availableERC721Ids: nftAvailable
    });

    prizesToUpdate[1] = ISpinGame.Prize({
        tokenAddress: address(0),
        amount: 1e18,
        lotAmount: 2,
        probability: 2,
        availableERC721Ids: empty
    });

    vm.prank(admin);
    spinGame.updatePrizes(prizesToUpdate);

    assertEq(spinGame.getPrize(1).probability, type(uint64).max - 1);
}
```

**권장 완화 방법:** 두 계산 모두에서 `unchecked` 블록을 제거하는 것을 고려하세요.

**Linea:** 커밋 [`e840e2f`](https://github.com/Consensys/linea-hub/commit/e840e2f04dca6006ac7b5782765c58e7a6869603)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Informational


### `winningThreshold` 조정이 무작위성 분포를 잘못 감소시킴 (Scaling `winningThreshold` incorrectly reduces randomness distribution)

**설명:** 사용자가 100%를 초과하는 승리 확률을 갖는 부스트를 가지고 있을 때, 계약은 [`Spin::_fulfillRandomness`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L534-L557)에서 `boostedTotalProbabilities`와 일치하도록 `winningThreshold`를 조정합니다:

```solidity
uint256 winningThreshold = _randomness % BASE_POINT;

// ...

if (boostedTotalProbabilities > BASE_POINT) {
    winningThreshold =
        (winningThreshold * boostedTotalProbabilities) /
        BASE_POINT;
}
```

여기서 문제는 `_randomness`가 `boostedTotalProbabilities`로 스케일링 업되기 전에 먼저 `BASE_POINT`로 스케일링 다운된다는 것입니다. 이 과정은 원래 `_randomness` 범위의 일부 값이 스케일링 후 최종 `winningThreshold`에 더 이상 표시되지 않기 때문에 유효 무작위성(엔트로피)을 감소시킵니다. 결과적으로 최종 임계값이 고르게 분포되지 않아 잠재적으로 편향이 발생할 수 있습니다.

승리 확률이 100%를 초과할 때 `_randomness`를 `boostedTotalProbabilities`에 직접 적용하여 엔트로피 손실이 없도록 하는 것을 고려하세요:

```diff
  if (boostedTotalProbabilities > BASE_POINT) {
-     winningThreshold =
-         (winningThreshold * boostedTotalProbabilities) /
-         BASE_POINT;
-
+     winningThreshold = _randomness % boostedTotalProbabilities;
  }
```

이것은 전체 무작위성 범위를 보존하고 가능한 승리 임계값의 보다 균일한 분포를 보장합니다.

**Linea:** 커밋 [`37a18ca`](https://github.com/Consensys/linea-hub/commit/37a18ca60b8e503643b5b6e996e9a0cd7c257ec2)에서 수정됨.

**Cyfrin:** 검증됨.


### 네이티브 토큰 전송에 명시적인 잔액 확인이 누락됨 (Native token transfers lack explicit balance check)

**설명:** 가능한 상금 유형 중 하나는 네이티브 토큰이며, `tokenAddress = address(0)`으로 표시됩니다. 네이티브 토큰을 획득한 사용자는 [`Spin::_transferPrize`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L347-L352)에서 상금을 청구할 수 있습니다:

```solidity
if (prize.tokenAddress == address(0)) {
    (bool success, ) = _winner.call{value: prize.amount}("");
    if (!success) {
        revert NativeTokenTransferFailed();
    }
} else {
```

ERC20 상금([여기서 처리됨](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L353-L362)) 및 ERC721 상금([여기서 처리됨](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L369-L371))의 경우, 계약은 전송을 진행하기 전에 충분한 잔액이나 토큰 소유권이 있는지 명시적으로 확인합니다.

현재 구현은 계약에 필요한 네이티브 토큰 잔액이 부족한 경우 여전히 revert되지만, 모든 상금 유형에 걸쳐 일관성을 제공하고 균일한 오류 메시지를 보장하여 사용성 및 디버깅을 개선하므로 네이티브 토큰에 대한 명시적인 잔액 확인을 추가하는 것을 고려하세요.

**Linea:** 커밋 [`7675766`](https://github.com/Consensys/linea-hub/commit/7675766ba45bd87888897ac130a587a45e47e96b)에서 수정됨.

**Cyfrin:** 검증됨.


### `updatePrizes`의 경쟁 조건(Race Condition)으로 인해 예상치 못한 상금이 발생할 수 있음 (Race Condition in `updatePrizes` Leading to Unexpected Prizes)

**설명:** `updatePrizes` 함수를 사용하면 사용 가능한 상금 목록을 수정할 수 있습니다. 그러나 무작위성 요청이 아직 충족되지 않은 진행 중인 참여를 고려하지 않아 일부 참가자에게 상금 ID가 할당되지 않은 상태로 남습니다. 즉, 참가자가 스핀을 시작하고 VRF가 난수를 제공하기 전에 `updatePrizes` 함수가 호출될 수 있습니다. 결과적으로 상금 분배가 업데이트되어 참가자는 원래 플레이했던 목록이 아닌 다른 목록의 상금을 받게 됩니다.

**영향:** 참가자는 처음에 참여했을 때 활성 상태였던 상금이 아닌 업데이트된 목록의 상금을 받을 수 있습니다.

**개념 증명 (Proof of Concept):**
1. 참가자가 스핀을 시작합니다.
2. VRF가 무작위성 요청을 충족하기 전에 updatePrizes가 호출되어 상금 분배가 수정됩니다.
3. 그런 다음 참가자는 예상되는 상금 대신 업데이트된 목록의 상금을 받습니다.

**권장 완화 방법:** 상금 목록에 대한 업데이트를 허용하기 전에 모든 보류 중인 VRF 요청이 충족되도록 하세요.

**Linea:** 인지함. 허용 가능한 동작임.

**Cyfrin:** 인지함.


### 어셈블리 블록에 `"memory-safe"` 주석을 추가하면 도움이 될 수 있음 (Assembly blocks could benefit from `"memory-safe"` annotation)

**설명:** [`Spin::_hashParticipation`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L383-L409) 및 [`Spin::_hashClaim`](https://github.com/Consensys/linea-hub/blob/295344925ec4321265f7cbac174fcf903b529a4e/contracts/src/Spin.sol#L411-L438)에서 요청 데이터를 해싱할 때, 인라인 어셈블리를 사용하여 해시를 효율적으로 계산합니다:
```solidity
assembly {
    let mPtr := mload(0x40)
    mstore(
        mPtr,
        0x4635ca970da82693e235d3cdaa3678d42c6824330c48b4135f080d655e54da78 // keccak256("ClaimRequest(address user,uint256 expirationTimestamp,uint64 nonce,uint32 prizeId)")
    )
    mstore(add(mPtr, 0x20), _user)
    mstore(add(mPtr, 0x40), _expirationTimestamp)
    mstore(add(mPtr, 0x60), _nonce)
    mstore(add(mPtr, 0x80), _prizeId)
    claimHash := keccak256(mPtr, 0xa0)
}
```

컴파일러 최적화를 개선하기 위해 어셈블리 블록에 [`memory-safe`](https://docs.soliditylang.org/en/latest/assembly.html#memory-safety) 주석을 추가하는 것을 고려하세요:

```diff
+ assembly ("memory-safe") {
```

어셈블리 블록은 여유 메모리 포인터(`0x40`) 이후의 메모리에만 액세스하므로 이 주석은 위험을 초래하지 않으며 Solidity 컴파일러가 추가 최적화를 적용하여 가스 효율성을 높일 수 있습니다.

**Linea:** 커밋 [`b4aaffc`](https://github.com/Consensys/linea-hub/commit/b4aaffc43e496b085e54ef2b08397fcb3c310e68)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
