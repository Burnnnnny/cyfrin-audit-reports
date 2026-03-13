**수석 감사관**

[Hans](https://twitter.com/hansfriese)

**보조 감사관**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### 초기화 중 기본 관리자 권한 취소에 대한 유효성 검사 누락

**설명:** `ShibuyaToken::initializeV3()`에서 계약은 제공된 `defaultAdmin` 매개변수로부터 `DEFAULT_ADMIN_ROLE`을 취소하려고 시도합니다. 테스트넷에 배포된 계약 및 팀 커뮤니케이션에 따르면, 의도는 업그레이드 프로세스 중에 원래 기본 관리자 주소에서 역할을 제거하는 것이었습니다.

그러나 현재 구현에는 두 가지 문제가 있습니다:
1. 주소가 이전 버전에서 실제로 역할을 보유하고 있는지 먼저 확인하지 않고 역할을 취소합니다.
2. 동반된 주석이 오해의 소지가 있어 "기본 관리자가 소유자임을 확인하고 배포자로부터 기본 관리자 역할을 제거하기 위한 것"이라고 명시하고 있습니다. 이는 배포자가 항상 기본 관리자 역할을 가지고 있음을 암시하지만, 반드시 그런 것은 아닙니다.

```solidity
Shibuya.sol
36:         // this is to make sure that the default admin is the owner and remove the default admin role from the deployer
37:         // so this means that the deployer will not have any role in the contract
38:         // the the DEFAULT_ADMIN_ROLE will not have any role in the contract
39:         // and the owner will be performing the role of the admin
40:         _revokeRole(DEFAULT_ADMIN_ROLE, defaultAdmin);
```

또한 38행의 주석에서 "the"가 반복되는 오타가 있습니다.

**권장 완화 방법:**
- 취소하기 전에 `defaultAdmin`이 `DEFAULT_ADMIN_ROLE`을 가지고 있는지 명시적으로 확인하는 기능을 추가하십시오.
- 업그레이드 가이드에 역할 요구 사항에 대해 명확하게 문서화하십시오.
- 주석의 사소한 오류를 수정하십시오.

**Startale:** 커밋 [01789b](https://github.com/StartaleLabs/ccip-contracts-registration/commit/01789b01cb654607c91f011a3ea768ebfc486a14)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### mint 함수에 대한 잘못된 문서

**설명:** `mint` 함수의 문서에는 "address(0)으로의 발행을 허용하지 않음(disallows minting to address(0))"이어야 하는데 "address(0)으로부터의 소각을 허용하지 않음(disallows burning from address(0))"이라고 잘못 명시되어 있습니다.

**권장 완화 방법:** 함수의 동작을 올바르게 반영하도록 문서를 업데이트하십시오.

**Startale:** PR [8](https://github.com/StartaleLabs/ccip-contracts-registration/pull/8)에서 수정됨.

**Cyfrin:** 확인됨.


### 불필요한 소유자 기반 액세스 제어 구현

**설명:** 계약은 `AccessControlUpgradeable`과 함께 사용자 지정 소유권 시스템을 구현하지만, 후자의 내장 액세스 제어 메커니즘을 효과적으로 활용하지 못합니다. 소유자에게 `DEFAULT_ADMIN_ROLE`이 부여되지 않기 때문에 계약은 적절한 액세스 제어를 유지하기 위해 `AccessControlUpgradeable`의 여러 공용 함수를 재정의(override)해야 합니다.

이러한 설계 선택은 잠재적인 문제를 의미합니다:
1. `AccessControlUpgradeable`에 이미 존재하는 액세스 제어 기능을 불필요하게 복제합니다.
2. `grantRole()` 및 `revokeRole()` 함수를 재정의해야 합니다.
3. `renounceRole()`에 대한 재정의를 놓쳐 역할 관리를 일관성 없게 처리합니다.
4. 코드 복잡성과 잠재적인 액세스 제어 혼란을 증가시킵니다.

현재 구현은 기능적이지만 불필요한 복잡성과 잠재적인 유지 관리 문제를 야기합니다.

**권장 완화 방법:** 별도의 소유권 시스템을 구현하는 대신 팀이 다음을 고려할 것을 권장합니다:
1. 초기화 중에 소유자에게 `DEFAULT_ADMIN_ROLE`을 부여하십시오.
2. 사용자 지정 소유권 구현을 제거하십시오.
3. `AccessControlUpgradeable`의 내장 역할 관리 함수를 활용하십시오.
4. 불필요한 함수 재정의를 제거하십시오.

**Startale:** 인지함.

**Cyfrin:** 인지함.


### 크로스체인 작업에서 0 금액 유효성 검사 누락

**설명:** `crosschainMint()` 및 `crosschainBurn()` 함수는 0 금액에 대한 유효성을 검사하지 않으므로 불필요한 이벤트가 방출될 수 있습니다.

**권장 완화 방법:** 0 금액 확인을 추가하고 금액이 0이면 되돌리십시오(revert).

**Startale:** 커밋 [fa77e9](https://github.com/StartaleLabs/ccip-contracts-registration/commit/fa77e9745ed943ba940a6523b441a67111355c1c)에서 수정됨.

**Cyfrin:** 확인됨.


### 불완전한 ERC20 인터페이스 지원

**설명:** `supportsInterface()` 함수는 ERC20 기능을 구현함에도 불구하고 `IERC20` 인터페이스 ID에 대한 지원을 선언하지 않습니다.
이로 인해 일부 통합 시나리오에서 인터페이스 감지 문제가 발생할 수 있습니다.

**권장 완화 방법:** `supportsInterface` 함수에 `type(IERC20).interfaceId`에 대한 지원을 추가하십시오.

**Startale:** 커밋 [cb5b05](https://github.com/StartaleLabs/ccip-contracts-registration/commit/cb5b05c4f09b449aa46b5e6290456f9f94cdb09f)에서 수정됨.

**Cyfrin:** 확인됨.



### 매개변수 명명 불일치

**설명:** `grantMintAndBurnRoles()` 함수의 매개변수 이름 'minAndBurner'에 오타가 있습니다.

```solidity
function grantMintAndBurnRoles(address minAndBurner)
```

**권장 완화 방법:** 매개변수 이름을 'mintAndBurner'로 수정하십시오.

**Startale:** 커밋 [669197](https://github.com/StartaleLabs/ccip-contracts-registration/commit/669197f945405f9805e90cc6fe49552c5f6e037a)에서 수정됨.

**Cyfrin:** 확인됨.


\clearpage

