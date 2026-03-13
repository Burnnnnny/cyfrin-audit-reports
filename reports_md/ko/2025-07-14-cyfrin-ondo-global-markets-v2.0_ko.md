**수석 감사자**

[Immeas](https://twitter.com/0ximmeas)

[Al-Qa'qa'](https://twitter.com/al_qa_qa)

**보조 감사자**



---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### `onUSD` 배포 시 `guardian`에게 `PAUSER_ROLE` 부여 누락

**설명:** `onUSD` 토큰의 배포는 `onUSDFactory`를 통해 처리되며, 이는 토큰을 투명 프록시 패턴(EIP-1967)을 사용하는 업그레이드 가능한 프록시로 설정합니다.

계약 주석에 문서화된 대로, `guardian` 주소는 `DEFAULT_ADMIN_ROLE`과 `PAUSER_ROLE`을 모두 부여받아야 합니다:

[globalMarkets/onUSDFactory.sol#L33-36](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSDFactory.sol#L33-L36)

```solidity
/**
...
 *         Following the above mentioned deployment, the address of the onUSD_Factory contract will:
 *         i) Grant the `DEFAULT_ADMIN_ROLE` & PAUSER_ROLE to the `guardian` address <<----------------
 *         ii) Revoke the `MINTER_ROLE`, `PAUSER_ROLE` & `DEFAULT_ADMIN_ROLE` from address(this).
 *         iii) Transfer ownership of the ProxyAdmin to that of the `guardian` address.
 */
```

그러나 실제 배포 로직에서는 `DEFAULT_ADMIN_ROLE`만 `guardian`에게 부여됩니다. `PAUSER_ROLE`은 누락되었습니다:

[globalMarkets/onUSDFactory.sol#L88](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSDFactory.sol#L88)

```solidity
  function deployonUSD( ... ) external onlyGuardian returns (address, address, address) {
    ...

    // @audit `PAUSER_ROLE` not granted to guardian
>>  onusdProxied.grantRole(DEFAULT_ADMIN_ROLE, guardian);

    onusdProxied.revokeRole(MINTER_ROLE, address(this));
    onusdProxied.revokeRole(PAUSER_ROLE, address(this));
    onusdProxied.revokeRole(DEFAULT_ADMIN_ROLE, address(this));

    onusdProxyAdmin.transferOwnership(guardian);
    assert(onusdProxyAdmin.owner() == guardian);
    initialized = true;
    emit onUSDDeployed( ... );

    return ( ... );
  }
```

결과적으로 배포가 완료되면 `onUSD` 토큰 계약에서 `guardian` 주소는 의도되고 문서화된 동작과 달리 `PAUSER_ROLE`을 갖지 않게 됩니다.


**영향:** `guardian`은 배포된 `onUSD` 토큰 계약에서 `PAUSER_ROLE`을 갖지 않습니다. 이는 배포 직후 토큰을 일시 정지하는 것을 방지하여 비상 상황에 대응하거나 규정 준수 제어를 시행하는 능력을 제한할 수 있습니다. 그러나 가디언이 `DEFAULT_ADMIN_ROLE`을 유지하므로 나중에 수동으로 자신에게 `PAUSER_ROLE`을 부여할 수 있습니다. 하지만 이는 의도된 원스텝 초기화 흐름에서 벗어나며 운영상의 실수 위험을 초래합니다.

**권장되는 완화 방법:** 계약의 의도된 동작 및 문서와 일치하도록 `DEFAULT_ADMIN_ROLE`을 할당한 직후 `guardian` 주소에 `PAUSER_ROLE`을 부여하십시오:

```diff
  onusdProxied.initialize(name, ticker, complianceView);

  onusdProxied.grantRole(DEFAULT_ADMIN_ROLE, guardian);
+ onusdProxied.grantRole(PAUSER_ROLE, guardian);

  onusdProxied.revokeRole(MINTER_ROLE, address(this));
  onusdProxied.revokeRole(PAUSER_ROLE, address(this));
```

**Ondo:** 커밋 [`b13a651`](https://github.com/ondoprotocol/rwa-internal/pull/472/commits/b13a651ae927e972a5c1478080fbe37e85409071)에서 수정되었습니다. 여기서 잘못된 것은 주석입니다. 우리는 기본 관리자 역할만 부여하기를 원합니다. 이는 배포 EOA가 계약을 올바르게 구성하기 위해 일시적으로 사용하기 때문입니다. 구성이 완료되면 기본 관리자 권한은 포기됩니다. 만약 배포 시 EOA에 일시 정지 권한도 부여된다면, 권한 포기를 위한 또 다른 호출이 필요할 것입니다.

**Cyfrin:** 확인되었습니다. 주석이 제거되었습니다.


### `onUSDManager`와 `onUSD` 이체 간의 규정 준수 확인 불일치

**설명:** `onUSDManager`를 통해 `onUSD`를 발행하거나 상환할 때, 계약은 `BaseRWAManager`를 확장하며, 이는 `onUSD` 토큰 주소(`address(onUSD)`)를 `rwaToken` 식별자로 사용하여 규정 준수 확인을 수행합니다. 이는 [`BaseRWAManager::_processSubscription`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/xManager/rwaManagers/BaseRWAManager.sol#L171-L172)에서 발생합니다:

```solidity
// Reverts if user address is not compliant
ondoCompliance.checkIsCompliant(rwaToken, _msgSender());
```

동일한 확인이 [`BaseRWAManager::_processRedemption`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/xManager/rwaManagers/BaseRWAManager.sol#L243-L244)을 통한 상환 중에도 발생합니다.

이와 별도로, `onUSD` 토큰 계약 자체는 이체, 발행 및 소각 중에 호출되는 [`onUSD::_beforeTokenTransfer`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSD.sol#L168-L180) 내부에서 규정 준수 확인을 수행합니다. 이 함수는 상속된 [`OndoComplianceGMClientUpgradeable::_checkIsCompliant`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/gmTokenCompliance/OndoComplianceGMClientUpgradeable.sol#L86-L88)를 호출하며, 이는 [`OndoComplianceGMView::checkIsCompliant`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/gmTokenCompliance/OndoComplianceGMView.sol#L75-L81)로 위임합니다:

```solidity
function checkIsCompliant(address user) external override {
  compliance.checkIsCompliant(gmIdentifier, user);
}
```

여기서 [`OndoComplianceGMViewgmIdentifier`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/gmTokenCompliance/OndoComplianceGMView.sol#L34-L36)는 문자열 `"global_markets"`에서 파생된 하드코딩된 주소이며 `rwaToken` 식별자로 사용됩니다:

```solidity
address public gmIdentifier =
  address(uint160(uint256(keccak256(abi.encodePacked("global_markets")))));
```

결과적으로 발행 및 상환 시 서로 다른 식별자를 사용하여 두 번의 규정 준수 확인이 트리거됩니다:

* 관리자 로직을 통한 `address(onUSD)`
* 토큰의 `_beforeTokenTransfer`를 통한 `gmIdentifier`

**영향:** `_beforeTokenTransfer`가 발행 및 소각 중에 실행되므로 두 규정 준수 확인이 모두 발생하지만, 두 개의 다른 `rwaToken` 식별자를 사용하는 것은 불필요한 불일치를 초래합니다. 두 규정 준수 목록이 일치하지 않는 경우, 사용자가 하나의 식별자 하에서는 규정을 준수하더라도 발행 또는 상환이 예상치 못하게 되돌려질(revert) 수 있습니다.

**권장되는 완화 방법:** `onUSD`의 정식(canonical) 규정 준수 식별자로 의도된 것이 무엇인지에 따라 두 가지 완화 접근 방식이 가능합니다.

1) `OnUSD::_beforeTokenTransfer`를 업데이트하여 모든 규정 준수 확인에서 `rwaToken`으로 `address(this)`를 명시적으로 사용하도록 합니다. 이는 이체/발행/소각 로직을 관리자의 발행/상환 흐름에서 사용되는 식별자와 일치시켜 일관성을 보장하고 두 개의 별도 규정 준수 목록을 유지할 필요성을 제거합니다.

   ```solidity
   if (from != msg.sender && to != msg.sender) {
     compliance.checkIsCompliant(address(this), msg.sender);
   }

   if (from != address(0)) {
     // If not minting
     compliance.checkIsCompliant(address(this), from);
   }

   if (to != address(0)) {
     // If not burning
     compliance.checkIsCompliant(address(this), to);
   }
   ```

2) `gmIdentifier`가 글로벌 시장 자산(`onUSD` 포함)에 대한 공유 규정 준수 ID로 사용될 의도라면, `onUSDManager` 발행/상환 흐름에서도 `gmIdentifier`를 사용하는 것을 고려하십시오. 이는 모든 규정 준수 확인을 단일 식별자 하에 통합하여 운영 파편화를 줄일 것입니다.


**Ondo:** 인지하였습니다. `USDonManager`의 `OndoCompliance` 확인은 `USDonManager`가 `BaseRWAManager`를 상속받기 때문에 존재하는 것입니다. `USDon` 이체 자체에 이미 확인이 존재하므로 이를 사용하면 완전히 중복됩니다. 이를 알고 있으므로, 우리는 `OndoCompliance`에서 `USDon`에 대한 제재 및 차단 목록을 설정하지 않은 상태로 두어 `USDonManager`에서 오는 확인이 효과적으로 우회되도록 하고, 대신 `USDon` 이체 자체에서 비롯되고 `gmIdentifier`를 키로 하는 확인에 의존할 것입니다.

\clearpage
## 정보성 (Informational)


### `OndoSanityCheckOracle::setAllowedDeviationBps`가 입력값으로 0을 확인하지 않아 사용 시 문제를 일으킬 수 있음

**설명:** `OndoSanityCheckOracle`에는 두 가지 유형의 편차(deviation) 값이 있습니다. 기본적으로 모든 토큰에 적용되는 기본 편차와 `setAllowedDeviationBps()`를 통해 자산별로 설정되는 토큰별 편차입니다.

기본 편차 값은 0이 아닌 값이어야 하지만, 토큰별 편차는 0으로 설정할 수 있습니다:

[OndoSanityCheckOracle.sol#L222-L245](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/sanityCheckOracle/OndoSanityCheckOracle.sol#L222-L245)

```solidity
function setAllowedDeviationBps(...) external onlyRole(CONFIGURER_ROLE) {
  if (bps >= BPS_DENOMINATOR) revert InvalidDeviationBps();
  prices[token].allowedDeviationBps = bps;
  emit AllowedDeviationSet(token, bps);
}

function setDefaultAllowedDeviationBps(...) public onlyRole(CONFIGURER_ROLE) {
  if (bps == 0) revert InvalidDeviationBps(); // enforced here
  if (bps >= BPS_DENOMINATOR) revert InvalidDeviationBps();
  emit DefaultAllowedDeviationSet(defaultDeviationBps, bps);
  defaultDeviationBps = bps;
}
```

그러나 토큰 편차를 0으로 설정하는 것은 기능적으로 의미가 없습니다. 가격 게시 중에 0은 "기본값 사용"으로 해석되기 때문입니다:

[OndoSanityCheckOracle.sol#L189-L192](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/sanityCheckOracle/OndoSanityCheckOracle.sol#L189-L192)

```solidity
if (priceData.allowedDeviationBps == 0) {
  priceData.allowedDeviationBps = defaultDeviationBps;
  emit AllowedDeviationSet(token, priceData.allowedDeviationBps);
}
```

이는 미묘한 불일치를 만듭니다. 계약은 토큰별 편차에 대해 `0`을 유효한 입력으로 받아들이지만, 가격을 게시할 때 이 값은 무시되고 재정의됩니다. 0 편차가 너무 엄격하거나 지원되지 않는 것으로 간주되면, `setDefaultAllowedDeviationBps()`의 검증을 반영하여 `setAllowedDeviationBps()`에서 `bps > 0` 확인을 시행하십시오.

또는 `0`이 "기본값 사용"을 나타내는 것으로 의도된 경우, `0`을 센티넬(sentinel) 값으로 의존하는 대신 토큰의 편차가 명시적으로 설정되었는지 여부를 추적하는 명시적인 불리언 필드를 도입하는 것을 고려하십시오.

**Ondo:** 커밋 [`6a33346`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/6a333464ee54fe04957331c270ce185a44e5e528)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `allowedDeviationBps`는 0이 될 수 없습니다.


### `onUSD`의 일관성 없는 unpause 역할

**설명:** [`onUSD::unpause`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSD.sol#L200-L202)는 전용 `UNPAUSER_ROLE`을 사용하는 시스템의 다른 계약과 달리 `DEFAULT_ADMIN_ROLE`로 제한됩니다. 이는 액세스 제어 설계의 일관성을 깨뜨리고 unpause 권한 위임의 유연성을 제한합니다:
```solidity
function unpause() public override onlyRole(DEFAULT_ADMIN_ROLE) {
  _unpause();
}
```

다른 계약에서 사용되는 패턴과 일치하도록 `onUSD::unpause`에 `UNPAUSER_ROLE`을 사용하는 것을 고려하십시오.

**Ondo:** 커밋 [`650c527`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/650c527100dbb655e9a485a5568c7796fdde3cc1)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `USDon::unpause` (이름 변경됨)에서 `UNPAUSER_ROLE`이 사용됩니다.


### `GMTokenManager::mintWithAttestation`이 Check-Effects-Interactions 패턴을 위반함

**설명:** [`GMTokenManager::mintWithAttestation`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L196-L231)에서, 함수는 속도 제한, 소각 및 발행과 같은 내부 회계 작업을 수행하기 전에 사용자로부터 토큰을 이체합니다. 이는 외부 호출(토큰 이체와 같은)이 일반적으로 모든 내부 상태 업데이트 후에 와야 위험을 줄일 수 있는 체크-이펙트-인터랙션 패턴을 위반합니다.

이체되는 토큰이 신뢰할 수 있는 스테이블코인이라고 가정하더라도, 이 순서는 통합된 토큰이 오작동하는 경우(예: 콜백 훅, 일시 중지 로직 또는 fee-on-transfer 동작을 통해) 예기치 않은 동작의 표면적을 증가시킵니다.

`mintWithAttestation`의 작업 순서를 변경하여 체크-이펙트-인터랙션 패턴을 따르도록 고려하십시오. 즉, `token.transferFrom()`을 호출하기 **전에** 속도 제한, 소각 및 발행을 수행하십시오.

**Ondo:** 커밋 [`29bdeb9`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/29bdeb92b8de97be3de6a60d78bf91449be90827)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 속도 제한이 이제 외부 호출 전에 수행됩니다.


### `GMTokenManager::setIssuanceHours`에 대한 일관성 없는 역할

**설명:** [`GMTokenManager::setIssuanceHours`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L400-L410) 함수는 `CONFIGURER_ROLE`로 제한되는 반면, 시스템 전반의 다른 구성 및 역할 할당 함수는 일반적으로 `DEFAULT_ADMIN_ROLE`로 제한됩니다. 이러한 불일치는 거버넌스 및 구성 조치를 담당하는 역할에 대한 혼란을 야기할 수 있습니다.

`setIssuanceHours`를 `DEFAULT_ADMIN_ROLE`로 제한하여 다른 유사한 구성 함수와 일관되게 액세스 제어를 조정하는 것을 고려하십시오.

**Ondo:** 커밋 [`3d18299`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/3d18299bab888ef204073581d92ffbc3de13ad30)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `GMTokenManager::setIssuanceHours`에 `DEFAULT_ADMIN_ROLE`이 사용됩니다.


### `GMTokenManager`에서 불필요한 불리언 비교

**설명:** [`GMTokenManager::_verifyQuote#L329`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L329) 및 [`GMTokenManager::adminProcessMint#L389`](http://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L389) 모두에서 불리언 비교가 있습니다:
```solidity
if (gmTokenAccepted[gmToken] == false) revert GMTokenNotRegistered();
```
이는 불필요합니다. 다음과 같이 단순화하는 것을 고려하십시오:
```solidity
if (!gmTokenAccepted[gmToken]) revert GMTokenNotRegistered();
```

**Ondo:** 커밋 [`1877211`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/1877211865727c6ae6e1587550266a43973d722c)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `IssuanceHours.HOUR_IN_SECONDS`에 대한 일관성 없는 타입 사용

**설명:** `IssuanceHours`에서 상수 [`IssuanceHours.HOUR_IN_SECONDS`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/issuanceHours/IssuanceHours.sol#L38) 필드는 `uint`로 선언된 반면, 코드베이스의 나머지 부분은 일관되게 `uint256`을 사용합니다:
```solidity
/// Constant for the number of seconds in an hour
uint constant HOUR_IN_SECONDS = 3_600;
```

프로젝트의 표준 타입 선언과 일치하도록 필드를 `uint256`을 사용하도록 업데이트하는 것을 고려하십시오.

**Ondo:** 커밋 [`fe452a1`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/fe452a120f8afde757d19736c44b26d4b07fbca3)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `HOUR_IN_SECONDS`는 `int256` 타입을 사용합니다 (`_validateTimezoneOffset`에서 캐스팅을 제거하기 위해).


### `PriceData` 구조체의 혼란스러운 필드 이름 `minimumLiveness`

**설명:** `OndoSanityCheckOracle`의 [`PriceData`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/sanityCheckOracle/OndoSanityCheckOracle.sol#L37-L49) 구조체에는 `minimumLiveness`라는 필드가 포함되어 있는데, 이는 실제로 가격이 오래된(stale) 것으로 간주되기 전까지의 최대 수명을 나타냅니다. 현재 이름은 오해의 소지가 있을 수 있는데, "minimum liveness(최소 활성도)"는 최신성에 대한 하한을 의미하는 반면 실제로는 진부화(staleness)에 대한 상한을 의미하기 때문입니다.

목적을 더 잘 반영하고 코드 가독성을 높이기 위해 필드 이름을 `maxPriceAge` 또는 `staleThreshold`와 같이 더 명확한 이름으로 변경하는 것을 고려하십시오.

**Ondo:** 커밋 [`b453b57`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/b453b57a8785ee7905d8dc46bee47694f43f152c) 및 [`9af9735`](https://github.com/ondoprotocol/rwa-internal/pull/470/commits/9af9735587341a0c97a04ce00ace905406b87e8c)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `maxTimeDelay`로 이름이 변경되었습니다.


### 테스트 개선 사항

**설명:** * `GMIntegrationTest_GM_ETH`: 두 테스트 [`test_hitRateLimits_onUSDInGMFlow_Subscribe`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/GM_IntegrationTest.t.sol#L1209-L1210) 및 [`test_hitRateLimits_onUSDInGMFlow_Redeem`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/GM_IntegrationTest.t.sol#L1254-L1255)는 빈 `expectReverts`를 가지고 있습니다:
   ```solidity
   // Should fail due to onUSD rate limit
   vm.expectRevert();
   gmTokenManager.mintWithAttestation(
     quote,
     signature,
     address(USDC),
     usdcAmount
   );
   ```
   모든 되돌리기(revert)를 수용하면 예상치 못한 오류를 숨겨 버그가 여전히 테스트를 통과할 수 있습니다. 예상되는 되돌리기를 잡는 것을 고려하십시오:
   ```diff
       // Should fail due to onUSD rate limit
   -   vm.expectRevert();
   +   vm.expectRevert(OndoRateLimiter.RateLimitExceeded.selector);
       gmTokenManager.mintWithAttestation(
         quote,
         signature,
         address(USDC),
         usdcAmount
       );
   ```

* `GmTokenManagerSanityCheckOracleTest`: 테스트 [`testPostPricesWithInvalidInput`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/tokenManager/GmTokenManagerSanityCheckOracleTest.t.sol#L563-L577) 또한 빈 `expectRevert()`를 가지고 있습니다. 이 테스트는 이상적으로 `...WithInvalidToken`, `...WithInvalidPrice`로 분할되어야 하며 올바른 오류인 `InvalidAddress` 및 `PriceNotSet`을 기대해야 합니다.

* `error TokenPauseManagerClientUpgradeable.TokenPauseManagerCantBeZero`에 대한 테스트가 부족합니다. 잘못된 `TokenPauseManager`를 할당하는 것에 대한 테스트를 추가하는 것을 고려하십시오.

* `GMTokenManagerTest_ETH`: 테스트 [`testMintFromNonKYCdSender`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/tokenManager/GmTokenManagerTest.t.sol#L587-L629)는 존재하지 않는 "KYC role"을 언급합니다. 또한 [L626](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/tokenManager/GmTokenManagerTest.t.sol#L626)에서 빈 되돌리기를 잡습니다. 이 catch는 올바른 오류를 잡지 못하며, 사용자가 속도 제한 설정이 없기 때문에 `OneRateLimiter.RateLimitExceeded` 오류를 잡습니다. 사용자는 [L601](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/forge-tests/globalMarkets/tokenManager/GmTokenManagerTest.t.sol#L601)에서 레지스트리에 추가되어 사실상 KYC가 완료되었다고 말하는 것과 같습니다. 따라서 KYC 확인을 통과합니다. KYC 역할에 대한 언급을 제거하고, 올바른 되돌리기(`IGMTokenManagerErrors.UserNotRegistered`)를 잡고, 사용자를 레지스트리에 추가하는 것을 제거하는 것을 고려하십시오.

**Cyfrin:** Cyfrin에 의해 커밋 [`d3155d0`](https://github.com/ondoprotocol/rwa-internal/pull/469/commits/d3155d09d8bb0ed48b7975d758830fc60c36e525)에서 수정되었습니다.


### Natspec 개선 사항

**설명:** * [`onUSD_Factory::deployonUSD`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSDFactory.sol#L54-L76)의 natspec에 `complianceView` 매개변수가 누락되었습니다.
* [`onUSD_Factory.onUSDDeployed`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/onUSDFactory.sol#L113-L126) 이벤트에 `name`, `ticker`, `complianceView` 매개변수가 누락되었습니다.
* [`GMTokenManager::constructor`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L140-L156)에 `_onUsd` 매개변수가 누락되었습니다.
* [`GMTokenManager::adminProcessMint`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L376-L398)에 `gmToken` 매개변수가 누락되었습니다.
* [`TokenPauseManager::unpauseAllTokens`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenPauseManager/TokenPauseManager.sol#L105-L113): `Only affects tokens paused by the pauseAllTokens function`라는 텍스트는 _모든_ 토큰에 해당하므로 더 나은 표현으로 수정될 수 있습니다.

**Ondo:** 커밋 [`d7dc414`](https://github.com/ondoprotocol/rwa-internal/pull/471/commits/d7dc4144d42a5edb04a25814f42a677c8b798723)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `GMTokenManager` 발행/상환에 `nonReentrant` 수정자 누락

**설명:** [`GMTokenManager::mintWithAttestation`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L170-L175) 및 [`GMTokenManager::redeemWithAttestation`](https://github.com/ondoprotocol/rwa-internal/blob/a74d03f4a71bd9cac09e8223377b47f7d64ca8d4/contracts/globalMarkets/tokenManager/GMTokenManager.sol#L248-L253) 함수는 외부 토큰 이체 및 내부 상태 업데이트를 수행하지만 `nonReentrant` 수정자를 사용하지 않습니다. `GMTokenManager`는 현재 사용되지 않는 OpenZeppelin의 `ReentrancyGuard`를 상속하지만, 이 수정자는 이러한 함수에 적용되지 않습니다.

`mintWithAttestation` 및 `redeemWithAttestation`에 `nonReentrant` 수정자를 추가하는 것을 고려하십시오.

**Ondo:** 커밋 [`d7dc414`](https://github.com/ondoprotocol/rwa-internal/pull/471/commits/d7dc4144d42a5edb04a25814f42a677c8b798723)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
