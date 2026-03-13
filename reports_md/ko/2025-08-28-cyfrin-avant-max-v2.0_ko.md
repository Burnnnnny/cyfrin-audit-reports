**수석 감사**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**보조 감사**

---

# 발견 사항

## 낮은 위험 (Low Risk)

### 같은 블록에서 함수를 반복 호출하여 `PriceStorage::setPrice` 최대 하한 및 상한을 쉽게 우회 가능

**설명:** `PriceStorage::setPrice`는 갑작스러운 급격한 가격 변동을 방지하기 위해 현재 가격을 기준으로 최대 하한/상한 델타로 가격 변동을 제한합니다:
```solidity
uint256 lastPriceValue = lastPrice.price;
if (lastPriceValue != 0) {
  uint256 upperBound = lastPriceValue + ((lastPriceValue * upperBoundPercentage) / BOUND_PERCENTAGE_DENOMINATOR);
  uint256 lowerBound = lastPriceValue - ((lastPriceValue * lowerBoundPercentage) / BOUND_PERCENTAGE_DENOMINATOR);
  if (_price > upperBound || _price < lowerBound) {
    revert InvalidPriceRange(_price, lowerBound, upperBound);
  }
}
```

그러나 같은 블록에서 `setPrice` 함수를 여러 번 반복 호출하고, 매번 현재 허용된 최대 하한/상한만큼 `lastPrice`를 감소시키거나 증가시킴으로써 이를 쉽게 우회할 수 있습니다.

**파급력:** 급격한 가격 변동에 대한 제한은 사소하게 우회될 수 있어 효과적이지 않지만, `SERVICE_ROLE`을 가진 엔티티에 의해서만 가능합니다.

**권장 완화 방법:** `DEFAULT_ADMIN_ROLE`만 변경할 수 있는 구성 가능한 매개변수 `minPriceUpdateDelay`를 `PriceStorage` 계약에 추가하십시오. 그런 다음 `setPrice`에서:
```solidity
error PriceUpdateTooSoon(uint256 lastUpdate, uint256 minWaitTime);

// setPrice 내부:
if(lastPrice.timestamp != 0) {
    if(block.timestamp < lastPrice.timestamp + minPriceUpdateDelay) {
        revert PriceUpdateTooSoon(lastPrice.timestamp, minPriceUpdateDelay);
    }
}
```

**Avant:**
인지함: 제안은 타당하며, 향후 가격 설정을 자동화할 경우 변경을 고려할 수 있습니다. 현재 가격은 매주 수행되는 신중한 NAV 프로세스 후 수동으로 계산되며, 투명성을 높이기 위해 독립적인 제3자에게 아웃소싱될 가능성이 높습니다. 계산 후 가격 업데이트 트랜잭션도 수동으로 게시되며 체인에 제출되기 전에 정족수의 승인이 필요합니다. 이 설정은 반복 호출이나 잘못된 매개변수가 시도되지 않도록 보장하며, 현재로서는 현재 코드 제약 조건이 요구 사항에 적합합니다.

### 계약 관리자 및 소유자에 다중 서명 지갑 사용

**설명:** 감사의 일환으로 배포된 계약의 온체인 상태를 조사하도록 요청받았습니다. 이더리움을 예로 들면:
* 주 배포자는 [0xA5Ab0683d4f4AD107766a9fc4dDd49B5a960e661](https://etherscan.io/address/0xA5Ab0683d4f4AD107766a9fc4dDd49B5a960e661)로 보입니다.
* 소유권과 관리자 권한을 [0xD47777Cf34305Dec4F1095F164792C1A4AFB327e](https://etherscan.io/address/0xd47777cf34305dec4f1095f164792c1a4afb327e)로 이전했습니다.

이들은 모두 [EOA 주소(외부 소유 계정)](https://academy.binance.com/en/glossary/externally-owned-account-eoa)입니다:
```
cast code 0xA5Ab0683d4f4AD107766a9fc4dDd49B5a960e661 --rpc-url $ETH_RPC_URL
cast code 0xD47777Cf34305Dec4F1095F164792C1A4AFB327e --rpc-url $ETH_RPC_URL
0x
0x
```

**파급력:** EOA는 심각한 보안 위험을 초래합니다:
* 단일 실패 지점 (하나의 개인 키 손상 = 전체 제어)
* 시간 지연, 지출 한도 또는 승인 요구 사항을 구현할 수 없음
* 키 분실 시 복구 메커니즘 없음
* 피싱, 기기 손상 또는 강요에 취약함

**권장 완화 방법:** 관리자 및 소유권 권한을 다음과 같은 기능을 갖춘 다중 서명 지갑(예: Gnosis Safe)으로 이전하십시오:
* 최소 3-of-5 또는 2-of-3 임계값
* 중요한 작업에 대한 시간 지연
* 시간 잠금된 관리자 작업을 거부/취소할 수 있는 신뢰할 수 있는 제3자 엔티티
* 하드웨어 지갑을 사용하는 서명자
* 문서화된 키 관리 절차

**Avant:**
인지함; Avant는 계약 소유권 및 관리자 권한을 프로토콜의 자금을 보관하는 데 사용되는 ForDeFi 커스터디 생태계 내의 EOA로 이전할 계획입니다. 플랫폼에서 제공하는 RBAC 구조와 관련된 MPC 지갑은 제안된 다중 서명 구성과 유사하게 작동합니다. Avant의 설정은 정족수의 관리자 수준 사용자만 관리자 계약 호출을 수행할 수 있도록 보장합니다.

\clearpage
## 정보성 (Informational)

### Solidity에서 기본값으로 초기화하지 마십시오

**설명:** Solidity에서 기본값으로 초기화하지 마십시오:
```solidity
RequestsManager.sol
72:    for (uint256 i = 0; i < _allowedTokenAddresses.length; i++) {
```

**Avant:**
커밋 [0b590fa](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/0b590fa60fa75d73396b9bb48543b52396c204ca)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 누락된 이벤트 방출

**설명:** 누락된 이벤트 방출:
* `RequestsManager::constructor`는 `AllowedTokenAdded`를 방출해야 합니다.

**Avant:**
커밋 [7a3587c](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/7a3587ca6d673d143565703d094b6f9526fd8020)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 명명된 반환 변수를 사용할 때 더 이상 사용되지 않는 `return` 문 제거

**설명:** 명명된 반환 변수를 사용할 때 더 이상 사용되지 않는 `return` 문 제거:
* `RequestManager::_addMintRequest, _addBurnRequest`

**Avant:**
커밋 [7a3587c](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/7a3587ca6d673d143565703d094b6f9526fd8020)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 키와 값의 목적을 명확히 하기 위해 명명된 매핑 매개변수 사용

**설명:** 명명된 매핑 매개변수는 코드베이스의 거의 모든 곳에서 이미 사용되고 있으며, 유일한 예외는 다음과 같습니다:
```solidity
SimpleToken.sol
13:  mapping(bytes32 => bool) private mintIds;
14:  mapping(bytes32 => bool) private burnIds;
```

**Avant:**
커밋 [1cc43e3](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/1cc43e3c59baa16b5b529ad06fee637bd6131ec1)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `RequestsManager`의 민트 및 번 요청에 `deadline` 매개변수 추가

**설명:** `RequestsManager::requestMint` 및 `requestBurn`을 사용하면 호출자가 최소 출력 금액을 지정할 수 있지만, 요청이 완료되어야 하는 기한(deadline)을 지정할 수는 없습니다.

최소 출력 금액은 시간이 지남에 따라 "오래된(stale)" 상태가 됩니다. 사용자가 오늘 최소값으로 기대하는 것이 내일은 다를 수 있고 다음 주에는 또 다를 수 있습니다.

`RequestsManager::requestMint` 및 `requestBurn` 호출자가 요청이 완료되어야 하는 기한을 지정할 수 있도록 하는 것이 이상적입니다. 기한이 지나면 완료는 되돌려져야(revert) 하지만 취소는 여전히 가능해야 합니다.

**Avant:**
사용자가 언제든지 민트 및 번 요청을 취소할 수 있고 이미 최소 예상 출력 금액을 지정했다는 점을 고려할 때, 제안된 추가 제약 조건은 UX의 복잡성을 증가시키는 반면 많은 가치를 추가하지 않을 것이라고 믿습니다.

### `RequestsManager::constructor`에서 화이트리스트 활성화

**설명:** 현재 `RequestsManager::constructor`에는 화이트리스트 활성화가 주석 처리되어 있습니다:
```solidity
  constructor(
    address _issueTokenAddress,
    address _treasuryAddress,
    address _providersWhitelistAddress,
    address[] memory _allowedTokenAddresses
  ) AccessControlDefaultAdminRules(1 days, msg.sender) {
    // *생략 : 관련 없는 내용* //

    // @audit 주석 처리됨, 무허가 상태에서 시작
    // isWhitelistEnabled = true;
  }
```

무허가 상태에서 시작하는 것보다 생성자에서 화이트리스트를 활성화하여 제한된 상태에서 시작하는 것이 더 방어적입니다.

**Avant:**
인지함: 화이트리스트 기능은 Avant의 단기 로드맵에 없었으므로 주석 처리되었습니다. 우리는 그것으로 시작하는 것이 약간의 방어를 추가한다는 것에 동의하지만, 발행(minting)과 상환(redeeming)은 우리가 제어하는 2단계 요청/완료 프로세스이므로 트레이드오프를 수락했습니다.

### 가격이 책정되는 토큰 또는 기타 엔티티를 나타내는 식별자 또는 설명자를 `PriceStorage`에 추가

**설명:** `PriceStorage` 계약에는 무엇의 가격이 책정되고 있는지를 쉽게 나타내는 식별자나 설명자가 없습니다.

가격이 책정되는 토큰 또는 엔티티의 이름이 있는 문자열과 같은 식별자 또는 설명자를 추가하는 것을 고려하십시오.

**Avant:**
인지함: Avant는 향후 `PriceStorage` 배포 시 토큰 식별자 추가를 고려할 것입니다.

\clearpage
## 가스 최적화 (Gas Optimization)

### 최적화 도구 활성화

**설명:** `foundry.toml`에서 [Foundry 최적화 도구 활성화](https://dacian.me/the-yieldoor-gas-optimizoor#heading-enabling-the-optimizer):
```diff
+ optimizer = true
+ optimizer_runs = 1_000
```

**Avant:**
커밋 [871140d](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/871140d5175f2f7ca7de7be960b834eb1f671206)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `PriceStorage` 스토리지 패킹 개선

**설명:** * `PriceStorage::upperBoundPercentage` 및 `lowerBoundPercentage`는 동일한 스토리지 슬롯에 패킹하기 위해 `uint128`로 안전하게 선언될 수 있습니다:
```diff
- uint256 public upperBoundPercentage;
- uint256 public lowerBoundPercentage;
+ uint128 public upperBoundPercentage;
+ uint128 public lowerBoundPercentage;
```

이는 `PriceStorage::setPrice`에 대한 모든 호출의 가스 비용을 줄입니다.

* `IPriceStorage::Price`는 `price` 및 `timestamp`에 `uint128`을 사용하여 각 `Price` 구조체를 동일한 스토리지 슬롯에 안전하게 패킹할 수 있습니다:
```diff
interface IPriceStorage {
  struct Price {
-   uint256 price;
-   uint256 timestamp;
+   uint128 price;
+   uint128 timestamp;
  }
```

이는 `PriceStorage::setPrice`에서 `prices[key]` 및 `lastPrice`에 쓸 때 두 번의 스토리지 쓰기를 절약합니다.

**Avant:**
커밋 [0325fcd](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/0325fcdfb7d78e311fb194845bb27ea541a301a0), [d40fc3d](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/d40fc3d103a502d2a3ab54939dd43ca8193ca176)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `RequestsManager::cancelMint, cancelBurn`에서 `safeTransfer` 호출 시 `msg.sender` 사용

**설명:** `RequestsManager::cancelMint`는 먼저 `request.provider == msg.sender`를 강제합니다:
```solidity
    _assertAddress(request.provider, msg.sender);
```

따라서 `safeTransfer`를 호출할 때 `request.provider` 대신 `msg.sender`를 직접 사용하여 1번의 스토리지 읽기를 절약할 수 있습니다:
```diff
-   depositedToken.safeTransfer(request.provider, request.amount);
+   depositedToken.safeTransfer(msg.sender, request.amount);
```

`RequestsManager::cancelBurn`에도 동일하게 적용됩니다.

**Avant:**
커밋 [7a3587c](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/7a3587ca6d673d143565703d094b6f9526fd8020)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 1개의 동일한 스토리지 읽기를 절약하기 위해 `RequestsManager::cancelMint, cancelBurn`에서 `mintRequestExist` 수정자를 안전하게 제거 가능

**설명:** `RequestsManager::cancelMint`는 `request.provider == msg.sender`인지 검증합니다:
```solidity
L152:    _assertAddress(request.provider, msg.sender);
```

따라서 `mintRequests[_id].provider`의 동일한 스토리지 읽기 1번을 절약하기 위해 수정자 `mintRequestExist(_id)`를 안전하게 제거할 수 있습니다.

`RequestsManager::cancelBurn`에도 동일하게 적용됩니다.

**Avant:**
커밋 [f74d1fc](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/f74d1fc334cf34f8751285e55cf44e41238859e1)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 동일한 스토리지 읽기 캐시

**설명:** 스토리지에서 읽는 것은 비용이 많이 듭니다; 동일한 스토리지 읽기를 캐시하십시오:
* `RequestsManager.sol`:
```solidity
// cache `request.amount` in `RequestsManager::completeBurn`
239:    issueToken.burn(_idempotencyKey, address(this), request.amount);
244:    emit BurnRequestCompleted(_id, request.amount, _withdrawalAmount);
```

**Avant:**
커밋 [9bf3b60](https://github.com/Avant-Protocol/Avant-Contracts-Max/commit/9bf3b6041bee7bfeb79f19a90a79e13cf6674afa)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
