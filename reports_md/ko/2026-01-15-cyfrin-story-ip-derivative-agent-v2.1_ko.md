**Lead Auditors**

[Immeas](https://x.com/0ximmeas)

[JesJupyter](https://x.com/jesjupyter)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### 실제 민팅 수수료가 예측 수수료와 달라질 수 있음

**설명:** `IPDerivativeAgent::registerDerivativeViaAgent`는 `predictMintingLicenseFee(...)`에 의존해 `tokenAmount`를 산정하고, 해당 금액을 호출자로부터 전송받은 뒤 Royalty Module에 정확히 그 금액만큼 승인합니다. 그러나 `registerDerivative(...)` 수행 중 최종적으로 지불되는 수수료는 예측값과 달라질 수 있습니다. 예를 들어 훅 로직이나 실행 시점의 다른 조건 때문에 차이가 생길 수 있습니다. 현재 에이전트는 예측 금액과 실제 사용 금액을 정산하지 않습니다.

**영향:**
- 실제 수수료가 예측 금액보다 크면 Royalty Module이 에이전트가 승인하거나 보유한 것보다 많은 토큰을 가져가려 하면서 `registerDerivative(...)`가 되돌려져, 사용자가 충분히 높은 `maxMintingFee`를 제공했더라도 DoS가 발생할 수 있습니다.
- 실제 수수료가 예측 금액보다 작으면 초과 토큰이 자동 환불 경로 없이 에이전트 컨트랙트에 남게 되어 사용자 자금이 묶일 수 있습니다. 이 경우 권한 있는 관리자 출금이 있지 않는 한 회수가 어려울 수 있습니다.

**권장 완화 조치:**
`maxMintingFee`를 기준으로 승인 및 자금 조달을 수행한 뒤, 등록이 성공하면 남은 토큰 잔액을 호출자에게 환불하는 방안을 고려하십시오.

```diff

        // Handle token payment if required
        if (currencyToken != address(0) && tokenAmount > 0) {
            IERC20 token = IERC20(currencyToken);

            // Transfer tokens from licensee to this contract
            token.safeTransferFrom(msg.sender, address(this), tokenAmount);

            // Increase allowance for RoyaltyModule to pull tokens during registerDerivative
+           token.safeIncreaseAllowance(ROYALTY_MODULE, maxMintingFee);
-           token.safeIncreaseAllowance(ROYALTY_MODULE, tokenAmount);
        }

        // ...

        // Clean up any remaining allowance for RoyaltyModule
        if (currencyToken != address(0) && tokenAmount > 0) {
            IERC20 token = IERC20(currencyToken);
            uint256 remainingAllowance = token.allowance(address(this), ROYALTY_MODULE);
            if (remainingAllowance > 0) {
+               token.safeTransfer(msg.sender, token.balanceOf(address(this)));
                token.forceApprove(ROYALTY_MODULE, 0);
            }
        }

```

**Story:** [PR#5](https://github.com/piplabs/story-ecosystem/pull/5)에서 수정됨

**Cyfrin:** 확인함. 이제 에이전트로의 전송과 함께 `maxMintingFee` 기준 승인이 이뤄지며, 라이선스 모듈 호출 후 남은 토큰은 반환됩니다.

\clearpage
## 정보 (Informational)


### `IPDerivativeAgent`는 단일 부모 IP만 지원하여 `Multi-Parent Derivative` 사용 사례를 제한함

**설명:** `IPDerivativeAgent::registerDerivativeViaAgent`는 `parentIpIds`와 `licenseTermsIds`를 길이 `1`의 고정 배열로 구성하므로 단일 부모 IP만 지원하도록 하드코딩되어 있습니다.

```solidity
        // Prepare arrays for LicensingModule call (single parent)
        address[] memory parents = new address[](1);
        parents[0] = parentIpId;
        uint256[] memory licenseTermsIds = new uint256[](1);
        licenseTermsIds[0] = licenseTermsId;
```

하지만 하위 레벨의 `LicensingModule::registerDerivative`는 여러 부모 IP를 가진 파생작 등록을 명시적으로 지원합니다.

```solidity
    function registerDerivative(
        address childIpId,
        address[] calldata parentIpIds,
        uint256[] calldata licenseTermsIds,
```

프로토콜 [문서](https://docs.story.foundation/concepts/licensing-module/license-token#registering-a-derivative)에 따르면 IP Asset은 파생작으로 한 번만 등록할 수 있습니다. 부모가 여러 개인 경우 모든 부모 IP를 동일 호출에서 원자적으로 등록해야 하며, 한 번 등록되면 이후에 부모를 추가로 연결할 수 없습니다.
> IP Asset은 파생작으로 단 한 번만 등록할 수 있습니다. 여러 부모가 있다면 동시에 함께 등록해야 합니다.
> IP Asset이 한 번 파생작이 되면 더 이상 부모를 연결할 수 없습니다.

이 제약을 고려하면 에이전트의 단일 부모 설계는 나중에 복구 가능한 제한이 아닙니다. 오히려 코어 프로토콜 수준에서 명시적으로 지원되는 다중 부모 파생작을 `IPDerivativeAgent`를 통해서는 영구적으로 등록하지 못하게 만듭니다.

그 결과, 이 에이전트 추상화는 `LicensingModule`이 제공하는 기능과 보장으로부터 실질적으로 벗어나는 암묵적 제한을 도입합니다.

이 단일 부모 가정은 다음 영역에도 전파됩니다.
- 단일 부모 IP만 받는 `predictMintingLicenseFee()` 기반 수수료 추정은 다중 부모 파생작의 정확한 수수료 예측을 막습니다.
- 단일 `parentIpId`를 키로 사용하는 화이트리스트 메커니즘은 여러 부모 IP 조합에 대한 권한 규칙을 표현할 수 없게 만듭니다.

**영향:** 교차 부모 파생작 사용 사례에서 이 에이전트의 활용 가능성이 제한됩니다.

**권장 완화 조치:**
- 에이전트를 통해 다중 부모 파생작을 지원하려는 의도라면, `IPDerivativeAgent`가 부모 IP 배열과 라이선스 조건 배열을 입력받도록 확장하고, 수수료 예측 및 화이트리스트 로직도 이에 맞게 갱신하십시오.

- 단일 부모 지원이 의도된 설계라면, 이 제한을 명시적으로 문서화하고 다중 부모 파생작은 지원되지 않으며 직접 `LicensingModule`을 통해 등록해야 함을 분명히 하십시오.

**Story:** 인지함.


### 전송 수수료형 ERC20 토큰은 지원되지 않음

**설명:** `IPDerivativeAgent::registerDerivativeViaAgent`는 수수료 토큰이 1:1로 전송된다고 가정합니다. 호출자로부터 `transferFrom`으로 `tokenAmount`를 가져온 뒤, 이후 Royalty Module이 에이전트에서 다시 `transferFrom`으로 필요한 민팅 수수료를 가져갑니다.

그러나 fee-on-transfer 또는 디플레이션형 ERC20의 경우 초기 전송에서 에이전트가 `tokenAmount`보다 적은 수량을 받을 수 있습니다. 그러면 이후 Royalty Module의 인출 시 잔액이 부족해져 파생작 등록이 되돌려질 수 있습니다.

fee-on-transfer가 아닌 ERC20만 허용하거나, `amountIn` 같은 매개변수를 추가해 실제 수령 잔액 변화량을 측정한 뒤 필요한 수수료를 충당할 수 있을 때만 진행하고 초과분은 환불하는 방안을 고려하십시오.


**Story:** 인지함.


### `IPDerivativeAgent`를 통한 `CommercializerChecker` 접근 제어 우회

**설명:** `IPDerivativeAgent::registerDerivativeViaAgent`를 사용하면 에이전트 컨트랙트가 중개자로 동작하며 사용자를 대신해 `LicensingModule::registerDerivative`를 호출합니다. 이 과정에서 에이전트 컨트랙트 주소가 검증 흐름 전반에서 호출자 겸 라이선시로 전달됩니다.

LicensingModule 내에서는 다음과 같습니다.
```solidity
   if (
       !ILicenseTemplate(licenseTemplate).verifyRegisterDerivativeForAllParents(
           childIpId,
           parentIpIds,
           licenseTermsIds,
           msg.sender  // This is the agent contract address
       )
   ) {
       revert Errors.LicensingModule__LicenseNotCompatibleForDerivative(childIpId);
   }
```

`LicensingModule::verifyRegisterDerivativeForAllParents`에서는 호출자 파라미터가 `msg.sender`에서 파생되며, 이 경우 그 값은 에이전트 컨트랙트입니다. 이 값은 다시 `commercializerChecker` 훅으로 전달되어 검증 시 라이선시 파라미터로 사용됩니다.

```solidity
  // Check if the commercializerChecker allows the link
   if (terms.commercializerChecker != address(0)) {
       if (
           !IHookModule(terms.commercializerChecker).verify(
               parentIpId,
               licensee,  // This is the agent address, not the actual user
               terms.commercializerCheckerData
           )
       ) {
           return false;
       }
   }
```

결과적으로 `commercializerChecker` 구현체는 실제 최종 사용자가 아니라 에이전트 주소를 관찰하고 검증하게 됩니다. 따라서 라이선시를 최종 사용자라고 가정하는 훅 로직(예: 블랙리스트, 허용 목록, 컴플라이언스 검증)은 에이전트를 통해 호출될 때 직접 호출과 다른 방식으로 동작할 수 있습니다.

이 동작이 의도된 설계인지, 아니면 에이전트 추상화의 암묵적 결과인지 명확하지 않습니다.


**영향:** 이는 직접적인 취약점은 아니며 의도된 동작일 수 있습니다. 실제 영향은 에이전트를 통한 상호작용과 `LicensingModule` 직접 상호작용 사이의 의미론적 차이에 제한됩니다.

최악의 경우 제한된 최종 사용자가, 전역적으로는 허용된 부모-자식 IP 조합에 대해 에이전트를 경유해 파생작을 등록할 수 있습니다. 이는 부모 수준 권한 검사를 우회하는 것은 아니지만, 최종 사용자 수준의 강제를 전제로 한 훅 구현의 기대와는 달라질 수 있습니다.

**권장 완화 조치:** 예를 들어 "에이전트는 훅의 라이선시를 대체한다"는 식으로 관련 동작을 명시적으로 문서화할 수 있습니다.

**Story:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `IPDerivativeAgent::constructor`의 `owner` 검사는 중복됨

**설명:** `IPDerivativeAgent::constructor`는 초기화 목록에서 `Ownable(owner)`를 호출한 뒤 다시 `owner == address(0)`를 검사합니다. 하지만 OpenZeppelin의 `Ownable` 생성자가 이미 zero owner에 대해 되돌리므로, 이 커스텀 검사는 중복이며 실제로 도달할 수 없습니다. 제거를 고려하십시오.

**Story:** [PR#5](https://github.com/piplabs/story-ecosystem/pull/5)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
